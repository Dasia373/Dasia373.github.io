---
date: 2026-06-09T20:00:00+08:00
draft: false
title: "阶段2 — 接入 person 模型"
tags: ["RK3588","RKNN","NPU","person检测"]
categories: ["嵌入式Linux"]
summary: "接入 person RKNN 模型绑定 NPU core0，验证同步推理掉帧，实现跳帧推理+缓存结果策略，帧率恢复到约 29.3fps。"
---

# 6月9日加入person模型检测

增加入口：

```c++
void runImx415StreamingLoop(rknn_lite *person_model)
```

核心部分：

```c++
detect_result_group_t person_results; // 定义 person 检测结果结构体，用来保存当前帧的检测框
memset(&person_results, 0, sizeof(detect_result_group_t)); // 先把检测结果清零，避免上一帧残留数据影响当前帧

auto infer_start = std::chrono::steady_clock::now(); // 记录 person 推理开始时间，用来观察同步推理耗时
if (person_model != nullptr) // 判断 person 模型是否创建成功，避免空指针导致程序崩溃
{
	person_model->ori_img = frame.clone(); // 把当前摄像头帧复制给 rknn_lite，interf() 内部会使用 
    person_model->interf(person_results); // 同步执行 person 模型推理，并把检测结果写入 person_results
}

auto infer_end = std::chrono::steady_clock::now(); // 记录 person 推理结束时间

double infer_ms = std::chrono::duration<double, std::milli>(infer_end - infer_start).count(); // 计算 person 推理耗时，单位是毫秒
```

现在可用打印（infer_ms ）person_infer_last_ms，来获得推理耗时也可以观测模型跑起来的情况。

```c++
stream_data.person_results = person_results;// person_results 不再清零，把当前帧 person 检测结果交给 StreamingManager，用于 RTSP 画框
```

在main()中创建person模型：

```c++
    rknn_lite *imx415_person_model = new rknn_lite(
        model_person.c_str(), // person 模型路径，复用 main() 里已经解析出来的 model_person
        0,                    // 使用 NPU core 0，先只接一个模型，避免多模型抢资源
        1,                    // person 模型类别数是 1
        0);                   // 模型 id 设置为 0，rknn_lite 内部会按 person 逻辑画绿色框
```

运行情况：

稳定后某个瞬间的运行情况：

```cmd
top - 19:33:31 up  2:21,  4 users,  load average: 5.43, 5.64, 5.05 
Threads:  10 total,   9 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s): 20.5 us,  6.4 sy,  0.0 ni, 73.0 id,  0.0 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem :   3899.9 total,   2733.0 free,    683.0 used,    483.9 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   3136.1 avail Mem

 进程号 USER      PR  NI    VIRT    RES    SHR    %CPU  %MEM     TIME+ COMMAND
   2982 quark      1 -19  785236  89796  43600 R  45.2   2.2   0:59.99 rtsp_stream
   2976 quark      1 -19  785236  89796  43600 R  43.2   2.2   0:58.06 imx415_loop
   2990 quark      1 -19  785236  89796  43600 R  18.3   2.2   0:22.74 imx415_loop
   2992 quark      1 -19  785236  90428  43600 R  17.3   2.3   0:20.96 imx415_loop
   2991 quark      1 -19  785236  90004  43600 R  16.9   2.3   0:21.80 imx415_loop
   2993 quark      1 -19  785236  90428  43600 R  15.9   2.3   0:19.17 imx415_loop
   2994 quark      1 -19  785236  90428  43600 R  13.6   2.3   0:17.09 imx415_loop
   2996 quark      1 -19  785236  90644  43600 R  13.6   2.3   0:15.47 imx415_loop
   2995 quark      1 -19  785236  90428  43600 R  10.3   2.3   0:16.22 imx415_loop
   2981 quark     20   0  785236  89796  43600 S   1.0   2.2   0:01.15 mpp_h264e_2976
```

rtsp_stream 对比没接模型推理之前，有所提高==34%--->45.2%==，imx415_loop 也有43%，有一定提高，说明压力正常。目前RES也有大概90mb，整体吃了194%大概2个CPU核，空闲有73% idle。其它几个imx415_loop是错误的，是给多次复制粘贴了pthread_setname_np。关键的几个线程也是PR=1, NI=-19高优先级，`mpp_h264e_5361`解码线程NI=0 普通优先级，编码在 NPU/VPU 上，CPU 只负责送帧/取包。

测试几分钟后在imx415_person_sync_test.log中可以得知：

```cmd
IMX415 fps 平均值     : 27.353 fps
person 推理耗时平均值 : 29.550 ms
RTSP_WORKER fps 平均值: 29.663 fps
queue size           : 一直是 1
reuse_last 平均       : 约 7 / 90 帧
```

RTSP 端还能接近 30fps,但是fps却掉到了27fps左右，尝试使用跳帧推理。

核心改动：

```c++
detect_result_group_t person_results; // 当前帧最终要送给 StreamingManager 的 person 检测结果
memset(&person_results, 0, sizeof(detect_result_group_t)); // 先清零，防止没有检测结果时残留旧数据

person_frame_index++; // 每处理一帧，帧序号加 1

bool do_person_infer = false; // 标记当前帧是否真的执行 person 推理
if (person_model != nullptr && (person_frame_index % PERSON_INFER_INTERVAL) == 0) // 模型有效，并且当前帧刚好命中跳帧间隔
{
    do_person_infer = true; // 本帧执行真实 person 推理
}

double infer_ms = 0.0; // 当前帧 person 推理耗时；如果本帧复用结果，则保持 0

if (do_person_infer) // 当前帧需要执行真实推理
{
    auto infer_start = std::chrono::steady_clock::now(); // 记录推理开始时间

    person_model->ori_img = frame.clone(); // 给模型一份当前帧图像，避免模型内部修改影响原始推流帧
    person_model->interf(person_results); // 执行 person 模型推理，结果写入 person_results

    auto infer_end = std::chrono::steady_clock::now(); // 记录推理结束时间
    infer_ms = std::chrono::duration<double, std::milli>(infer_end - infer_start).count(); // 计算本次推理耗时，单位 ms

    last_person_infer_ms = infer_ms; // 保存最近一次真实推理耗时
    last_person_results = person_results; // 缓存本次检测结果，给后续跳过推理的帧复用

    person_infer_count++; // 本统计窗口内真实推理次数加 1
    stat_person_infer_ms_sum += infer_ms; // 累加真实推理耗时

    if (infer_ms > stat_person_infer_ms_max) // 如果本次推理比之前记录的最大值还大
    {
        stat_person_infer_ms_max = infer_ms; // 更新最大推理耗时
    }
}
else // 当前帧不执行真实推理
{
    person_results = last_person_results; // 直接复用上一帧 person 检测结果
    person_reuse_count++; // 本统计窗口内复用次数加 1
}
```

新增打印参数：

```c++
<< ", person_interval=" << PERSON_INFER_INTERVAL
<< ", person_infer_count=" << person_infer_count
<< ", person_reuse_count=" << person_reuse_count
<< ", person_infer_avg_ms=" << avg_person_infer_ms
<< ", person_infer_max_ms=" << stat_person_infer_ms_max
<< ", person_infer_last_ms=" << last_person_infer_ms
```

10min测试中，观测到画面延迟很高大约4-5秒以上了，推理时画面会发绿，掉帧严重。立刻打印日志。

imx415_person_skip3_test.log

日志检测到：

```cmd
IMX415 fps 平均: 28.856
person_infer_avg_ms 平均: 32.068 ms
RTSP_WORKER fps 平均: 36.745
reuse_last 平均: 19 / 90 帧
queue size: 平均 1.06，最大 2
```

可以发现RTSP端超过了30fps，这里并不太正常，注意到原工程streamingworker里面有使用reuse复用帧，帧数复用可以避免画面卡住，估计是这里出现了一定问题。

然后关闭了复用帧数的部分。

imx415_person_skip3_no_reuse_test.log

日志中可以看出

```cmd
IMX415 fps 平均: 28.980
RTSP_WORKER fps 平均: 28.979
reuse_last: 0
queue size 平均: 1.052，最大 2

person_interval: 3
person_infer_count: 30 / 90 帧
person_reuse_count: 60 / 90 帧
person_infer_avg_ms: 31.651 ms
person_infer_max_ms: 59.558 ms
```

然后提升帧数我们将跳帧改为6帧一次。

```cmd
IMX415 fps 平均: 29.315
RTSP fps 平均 29.302
person 推理 15 次 / 90 帧

person_interval: 6
person_infer_count: 15 / 90 帧
person_reuse_count: 75 / 90 帧
person_infer_avg_ms: 32.889 ms
```

发现6帧一次检测虽然降低了点实时性，但是画面相对来说是更加稳定的。

目前跑了一段时间的线程监控和温度如下：

```cmd
top - 22:34:32 up  5:22,  4 users,  load average: 6.16, 5.73, 5.29
Threads:  10 total,   1 running,   9 sleeping,   0 stopped,   0 zombie
%Cpu(s): 20.9 us,  7.8 sy,  0.0 ni, 70.8 id,  0.0 wa,  0.0 hi,  0.5 si,  0.0 st
MiB Mem :   3899.9 total,   2716.0 free,    683.9 used,    500.0 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   3135.2 avail Mem

 进程号 USER      PR  NI    VIRT    RES    SHR    %CPU  %MEM     TIME+ COMMAND
   5367 quark      1 -19  785204  91396  43760 S  42.0   2.3   9:04.07 rtsp_stream
   5361 quark      1 -19  785204  91396  43760 R  28.0   2.3   6:00.84 imx415_loop
   5374 quark      1 -19  785204  91660  43760 S  23.3   2.3   4:51.31 imx415_loop
   5375 quark      1 -19  785204  91660  43760 S  23.0   2.3   4:38.11 imx415_loop
   5376 quark      1 -19  785204  91660  43760 S  21.0   2.3   4:33.44 imx415_loop
   5377 quark      1 -19  785204  91660  43760 S  20.7   2.3   4:04.52 imx415_loop
   5378 quark      1 -19  785204  91924  43760 S  18.7   2.3   3:40.06 imx415_loop
   5380 quark      1 -19  785204  91924  43760 S  17.3   2.3   3:14.24 imx415_loop
   5379 quark      1 -19  785204  91924  43760 S  16.0   2.3   3:24.71 imx415_loop
   5366 quark     20   0  785204  91396  43760 S   1.0   2.3   0:10.60 mpp_h264e_5361
```

```cmd
quark@quarkpi-ca2:~$ cat /sys/class/thermal/thermal_zone*/temp
51769
51769
51769
51769
51769
50846
50846
quark@quarkpi-ca2:~$ cat /sys/class/thermal/thermal_zone*/temp
51769
51769
51769
51769
51769
50846
50846
quark@quarkpi-ca2:~$ cat /sys/class/thermal/thermal_zone*/temp
51769
51769
51769
51769
50846
49923
5176
```

可以看出：

```
1. 系统 CPU idle 仍有 70.8%，说明整机没有被打满。
2. myDemo 主要 CPU 消耗集中在 rtsp_stream、IMX415 采集/推理相关线程。
3. MPP H.264 编码线程 CPU 仅约 1%，说明 H.264 编码主要由硬件承担，不是 CPU 瓶颈。
4. rtsp_stream 约 42% CPU，主要可能来自 resize、drawDetections、BGR→YUV 转换、封包写出等流程。
5. 多个 imx415_loop 线程名称相同，是因为线程名可能被继承或未单独命名；它们并不一定都是采集线程，其中大概率包含 RKNN runtime / 线程池相关工作线程。
6. RES 约 91 MB，占用约 2.3%，内存压力较小。
7. 温度大概50度左右，属于正常工作温度。
```













