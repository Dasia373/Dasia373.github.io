---
date: 2026-06-07T21:33:00+08:00
draft: false
title: "阶段1 — IMX415 冒烟测试 + debug 卡顿问题"
tags: ["RK3588", "IMX415", "RTSP", "V4L2", "ftrace"]
categories: ["嵌入式Linux"]
summary: "打通 IMX415 V4L2 采集 → MPP H.264 编码 → FFmpeg RTSP 推流链路，定位并修复 streamingWorker 中 sleep_for(33ms) 导致的队列积压与卡顿。"
---


# 6月7日调试日志

## 调试环境：

正点原子QuarkPi（RK3588s）+ IMX415摄像头（MIPI@CAM1），SSH（PowerShell，Xshell8，XFTP8），Neovim, Codex CLI，mediamtx。

推流参数：1920x1080（@30fps）从V4L2采集，走RTSP协议*（不使用RTMP和GB28181平台）*，相关代码改动已经同步至github仓库分支[feat/imx415-camera-smoke-test](https://github.com/Dasia373/IMX415Optim/commit/5ce80da91afd47614223ad1cec1f997921b4c10e)。

## 问题描述：

观测时发现有2s到3s左右的延迟，伴随有较为明显的掉帧。于是开始观测行为。

【2026.6.7晚上21点33找到问题，是sleep33ms导致了，queue爆满，同步不及时，原本这个sleep用于视频流同步刻意等待了1000 / config.fps ms】

## 问题寻找：

### 程序运行时：

#### 1.初始化：

```cmd
./run_streams.sh rtsp 4

Streaming config:
  RTMP    : 0 (rtmp://192.168.137.1/live/livestream)
  RTSP    : 1 (rtsp://192.168.137.1:8554/mystream)
  GB28181 : 0 (/home/quark/My_Projects/IMX415optimizer/gb28181.conf)

StreamLoaderManager created
H.264 extradata from MPP, size: 37 bytes
Streaming started
VIDIOC_S_PARM failed, errno=25
IMX415 opened: /dev/video33, 1920x1080, format=0x3231564e, plane=1
rga_api version 1.10.1_[0]
```

可以看出摄像头被成功打开且成功推流，format=0x3231564e说明为NV12，但是__VIDIOC_S_PARM failed__。

#### 2.运行日志：

```cmd
Added frame to streaming queue, queue size: 11
[IMX415] total=1440, fps=29.74, read_avg_ms=5.73231 ms, read_max_ms=24.312 ms
Sent frame 900 to RTSP (MPP encoded, size=10465 bytes)
Added frame to streaming queue, queue size: 11
[IMX415] total=1530, fps=29.6997, read_avg_ms=5.02038 ms, read_max_ms=21.0212 ms
Added frame to streaming queue, queue size: 11
Sent frame 1000 to RTSP (MPP encoded, size=591 bytes)
[IMX415] total=1620, fps=29.736, read_avg_ms=5.29003 ms, read_max_ms=24.5967 ms
Added frame to streaming queue, queue size: 11
[IMX415] total=1710, fps=29.6997, read_avg_ms=6.18894 ms, read_max_ms=23.8422 ms
Sent frame 1100 to RTSP (MPP encoded, size=703 bytes)
[IMX415] total=1800, fps=29.692, read_avg_ms=5.70099 ms, read_max_ms=25.3068 ms
Added frame to streaming queue, queue size: 11
[IMX415] total=1890, fps=29.5616, read_avg_ms=5.6416 ms, read_max_ms=21.8569 ms
Added frame to streaming queue, queue size: 11
Sent frame 1200 to RTSP (MPP encoded, size=1059 bytes)
[IMX415] total=1980, fps=29.8682, read_avg_ms=5.40226 ms, read_max_ms=21.758 ms
Added frame to streaming queue, queue size: 11
[IMX415] total=2070, fps=29.7288, read_avg_ms=5.83811 ms, read_max_ms=25.9905 ms
```

启动阶段存在一个较大的延迟，后续通过打印的时间戳可以看出推流帧数可以稳定在29.5帧以上。

【为啥不会达到30fps？】

注意到queue size:10，__说明推流队列经常能满到11，怀疑消费端存在瓶颈？__。

```cmd
quark@quarkpi-ca2:~$ cat /sys/class/thermal/thermal_zone*/temp
49923
49923
49923
49923
49923
49000
49923
quark@quarkpi-ca2:~$ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq
1800000
1800000
1800000
1800000
816000
816000
2256000
2256000
```

注意到：温度目前维持在49°C左右。

4 个核：1.8 GHz
2 个核：816 MHz
2 个核：2.256 GHz

### 系统日志检查：

#### TOP-P检查：

```cmd
quark@quarkpi-ca2:~$ cat /proc/2431/status | grep -E "VmRSS|VmSize|Threads"
VmSize:	  813208 kB
VmRSS:	  126864 kB
Threads:	10
```

- VmSize ≈ 794 MB：虚拟地址空间。包含 mmap(V4L2)、动态库映射、预留地址等。
- VmRSS ≈ 124 MB：真实常驻内存。。
- Threads = 10：10个线程。

指令：

```cmd
pidof myDemo #编译程序名称，pid为2431
top -p 2431
```

瞬时结果：

```cmd
top-p :"top - 16:12:47 up 54 min,  5 users,  load average: 5.69, 4.76, 3.24 
任务:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu(s): 19.5 us,  6.2 sy,  0.0 ni, 74.0 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
MiB Mem :   3899.9 total,   2739.7 free,    746.0 used,    414.2 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   3080.1 avail Mem 

 进程号 USER      PR  NI    VIRT    RES    SHR    %CPU  %MEM     TIME+ COMMAND                                           
   2431 quark     20   0  813208 126800  24344 R 182.3   3.2  11:10.34 myDemo"
```

myDemo %CPU = 182.3 注意到这里吃了1.8个CPU核（RK3588s有4大4小），74% idle说明CPU仍然存有余量，RES = 126800 KB ≈ 124 MB  %MEM = 3.2% 内存目前占用不高（RK3588s设备为4G+32G版本）。

<img src="/posts/1-imx415冒烟测试-debug卡顿/assets/RK3588S.png" alt="RK3588架构" style="zoom:75%;" />

#### TOP-H检查：

线程监控：

```cmd
top -H -p 2431
```

瞬时结果：

```cmd
quark@quarkpi-ca2:~$ top -H -p 2431
top - 16:13:22 up 54 min,  5 users,  load average: 5.47, 4.80, 3.31
Threads:  10 total,   0 running,  10 sleeping,   0 stopped,   0 zombie
%Cpu(s): 19.9 us,  5.7 sy,  0.0 ni, 74.4 id,  0.0 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem :   3899.9 total,   2739.9 free,    745.7 used,    414.2 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   3080.3 avail Mem 

 进程号 USER      PR  NI    VIRT    RES    SHR    %CPU  %MEM     TIME+ COMMAND                                           
   2437 quark     20   0  813208 126812  24344 S  31.0   3.2   2:01.47 myDemo                                            
   2444 quark     20   0  813208 126812  24344 S  21.0   3.2   1:27.10 myDemo                                            
   2445 quark     20   0  813208 126812  24344 S  21.0   3.2   1:24.12 myDemo                                            
   2446 quark     20   0  813208 126812  24344 S  20.3   3.2   1:21.75 myDemo                                            
   2447 quark     20   0  813208 126812  24344 S  19.3   3.2   1:17.42 myDemo                                            
   2449 quark     20   0  813208 126812  24344 S  18.3   3.2   1:11.89 myDemo                                            
   2431 quark     20   0  813208 126812  24344 S  18.0   3.2   1:02.84 myDemo                                            
   2448 quark     20   0  813208 126812  24344 S  17.7   3.2   1:14.02 myDemo                                            
   2450 quark     20   0  813208 126812  24344 S  17.3   3.2   1:10.23 myDemo                                            
   2436 quark     20   0  813208 126812  24344 S   0.7   3.2   0:02.30 mpp_h264e_2431 
```

注意到：mpp_h264e_2431 0.7%  这里MPP编码线程没有怎么吃CPU，推测CPU占用主要来源为用户业务线程上，包括不仅限于：

1. IMX415 V4L2 DQBUF + NV12->BGR；
2. StreamingManager 里的 BGR resize / BGR->MPP 输入处理；
3. 队列拷贝 cv::Mat；
4. ThreadPool 空转线程；
5. 项目全局线程池创建了 8 个线程。

注意到原工程上有创建用于RKNN推理线程的行为：

```c++
dpool::ThreadPool global_pool(8);
```

猜测ThreadPool的worker在忙等/轮询，这里尽管我们在冒烟测试中仅仅在执行RKNN推理函数前把函数写死了跑IMX415推流，但是已经创建的推理线程的worker依旧在执行，导致这一部分也会吃CPU。

#### 确认线程：

```cmd
ps -T -p 2431 -o pid,tid,comm,pcpu,stat
```

```cmd
quark@quarkpi-ca2:~$ ps -T -p 2431 -o pid,tid,comm,pcpu,stat
    PID     TID COMMAND         %CPU STAT
   2431    2431 myDemo          15.8 Sl+
   2431    2436 mpp_h264e_2431   0.5 Sl+
   2431    2437 myDemo          30.5 Sl+
   2431    2444 myDemo          21.9 Sl+
   2431    2445 myDemo          21.1 Sl+
   2431    2446 myDemo          20.5 Sl+
   2431    2447 myDemo          19.4 Sl+
   2431    2448 myDemo          18.5 Sl+
   2431    2449 myDemo          18.1 Sl+
   2431    2450 myDemo          17.6 Sl+
```

注意到：2437 myDemo 30.5%，查看下线程stack

```cmd
sudo cat /proc/2431/task/2437/stack
```

```
__switch_to
0xffffff81f3ba4ec0
```

没看出什么东西。

值得注意的是：

```cmd
quark@quarkpi-ca2:~$ sudo cat /proc/2431/task/2444/stack
[<0>] __switch_to+0xe4/0x140
[<0>] futex_wait_queue_me+0xb8/0x140
[<0>] futex_wait+0xe0/0x250
[<0>] do_futex+0x134/0xe10
[<0>] __arm64_sys_futex+0x11c/0x190
[<0>] el0_svc_common.constprop.0+0x6c/0x1c0
[<0>] do_el0_svc+0x20/0x30
[<0>] el0_svc+0x1c/0x2c
[<0>] el0_sync_handler+0xa8/0xac
[<0>] el0_sync+0x158/0x180
quark@quarkpi-ca2:~$ sudo cat /proc/2431/task/2445/stack
[<0>] __switch_to+0xe4/0x140
[<0>] futex_wait_queue_me+0xb8/0x140
[<0>] futex_wait+0xe0/0x250
[<0>] do_futex+0x134/0xe10
[<0>] __arm64_sys_futex+0x11c/0x190
[<0>] el0_svc_common.constprop.0+0x6c/0x1c0
[<0>] do_el0_svc+0x20/0x30
[<0>] el0_svc+0x1c/0x2c
[<0>] el0_sync_handler+0xa8/0xac
[<0>] el0_sync+0x158/0x180
```

futex_wait这里可以注意到，在工程中StreamLoader，StreamingManager，image[]里面都有使用到std::mutex，由于访问是瞬时的，猜测是某个mutex在等待被释放，也可能是在等待条件变量暂时还没被唤醒。正常来说推理线程这里一般应该是在等待下一帧，如果没有堆积在一块的话。

【这里到时候补充下抓到行为下的日志】

### ftrace抓取V4L2事件日志：

#### 抓取过程1：

```cmd
#进入 root shell：
sudo -s
cd /sys/kernel/tracing
#清理旧状态：
echo 0 > tracing_on
echo nop > current_tracer
echo > trace
#先关闭可能残留的事件：
echo 0 > events/enable
#打开 V4L2/vb2 相关事件：
echo 1 > events/v4l2/v4l2_qbuf/enable
echo 1 > events/v4l2/v4l2_dqbuf/enable
echo 1 > events/v4l2/vb2_v4l2_qbuf/enable
echo 1 > events/v4l2/vb2_v4l2_dqbuf/enable
echo 1 > events/v4l2/vb2_v4l2_buf_queue/enable
echo 1 > events/v4l2/vb2_v4l2_buf_done/enable

echo 1 > events/vb2/vb2_qbuf/enable
echo 1 > events/vb2/vb2_dqbuf/enable
echo 1 > events/vb2/vb2_buf_queue/enable
echo 1 > events/vb2/vb2_buf_done/enable

#开始抓 10 秒：
echo 1 > tracing_on
sleep 10
echo 0 > tracing_on

#保存结果并关闭事件：
cat trace > /home/quark/imx415_v4l2_10s_trace.txt
cd /sys/kernel/tracing
echo 0 > events/enable
echo > trace
exit
```

imx415_v4l2_10s_trace.txt目前已经在assets文件夹。

#### 透过Codex分析1：

确认：**IMX415 V4L2 采集链路基本正常，当前主要瓶颈更像是在采集之后的处理/推流队列，而不是摄像头取帧本身。**

**关键结果**

/dev/video33 对应 trace 里的：

```cmd
minor = 33 type = VIDEO_CAPTURE_MPLANE
```

抓到的数据统计是：

```cmd
dq_count = 297 
q_count  = 297 
duration = 9.968 s 
fps      = 29.696
```

也就是说，10 秒内应用实际从 V4L2 拿到了约 **29.7 FPS**，这已经非常接近 30FPS。

每帧 DQBUF 间隔：

```cmd
min = 33.090 ms 
avg = 33.674 ms 
p50 = 33.635 ms 
p90 = 34.040 ms 
max = 39.765 ms
```

这说明摄像头/ISP/V4L2 产帧节奏是稳定的，基本就是 30FPS。

每帧从应用 DQBUF 拿到 buffer，到 QBUF 还回 buffer 的时间：

```cmd
avg = 4.871 ms
p50 = 3.072 ms 
p90 = 13.457 ms
max = 23.670 ms
```

这段时间主要包含你现在 readImx415Frame() 里面的用户态处理，比如 NV12 -> BGR 的 cv::cvtColor()，以及少量调度开销。

还有一个小现象：

```cmd
sequence gap_count = 3
```

10 秒内出现了 3 次 sequence 跳号，相当于约 300 帧里少了 3 帧，比例约 1%。现在还不算严重，但值得后面继续观察 5-10 分钟。

**结论**

目前可以先这样判断：

1. **V4L2 摄像头取流基本健康。**
2. bytesused = 3110400，刚好是 1920 * 1080 * 1.5，说明 NV12 buffer 大小正确。
3. 你之前日志里 Streaming queue size: 11 经常满，这比 V4L2 更值得怀疑。
4. 后续瓶颈优先看：NV12->BGR 转换、缩放、MPP 编码前处理、RTSP 发送线程、队列消费速度。

#### 抓取过程2：

本次抓取V4L2 + sched调度trace，判断存在的偶发13-23ms是否为线程被调度走了。

```cmd
cd /sys/kernel/tracing
echo 0 > tracing_on
echo > trace
echo 0 > events/enable
echo 1 > events/v4l2/v4l2_dqbuf/enable
echo 1 > events/v4l2/v4l2_qbuf/enable
echo 1 > events/vb2/vb2_dqbuf/enable
echo 1 > events/vb2/vb2_qbuf/enable
echo 1 > events/sched/sched_switch/enable
echo 1 > events/sched/sched_wakeup/enable
echo 1 > tracing_on
sleep 10
echo 0 > tracing_on
cat trace > /home/quark/imx415_sched_v4l2_10s_trace.txt
```

imx415_sched_v4l2_10s_trace.txt目前已经在assets文件夹。

#### 透过Codex分析2：

我分析了 imx415_sched_v4l2_10s_trace.txt，这份 trace 有两个关键信号：

1. **trace buffer 溢出了**

```cmd
entries-in-buffer/entries-written: 175140/689611
```

所以这份文件不是完整 10 秒全量，只保留了最后一段；里面的丢帧/sequence gap 只能作为参考，不能直接当最终 FPS 结论。

1. **调度切换非常异常地多**

```cmd
sched_switch = 157965 
sched_wakeup = 15823
```

保留下来的窗口里，/dev/video33 仍然是：

```cmd
minor = 33 
bytesused = 3110400
```

说明 NV12 1920x1080 buffer 仍然正确。

**V4L2 统计**

```cmd
dq_count = 113 
q_count  = 113 
DQ interval p50 = 33.732 ms 
DQ-to-Q avg = 4.960 ms 
DQ-to-Q p50 = 2.807 ms 
DQ-to-Q max = 32.099 ms
```

中位数还是 33ms，说明正常情况下摄像头节奏还是 30FPS。但因为 sched trace 开销很大，加上 buffer 溢出，这次统计出来的整体 FPS 不如上一份可靠。

最有价值的是这个：

```cmd
最慢 DQ-to-Q: 32.099 ms 
offcpu_ms = 0
```

也就是说这帧不是因为采集线程 2431 被调度走了，而是它在 CPU 上自己跑了很久。结合代码，DQBUF -> QBUF 中间主要就是：

```
cv::cvtColor(nv12, frame, cv::COLOR_YUV2BGR_NV12);
```

所以第一阶段可以判断：**V4L2 本身不是主要瓶颈，NV12 转 BGR 和后面的推流处理更值得看。**

另一个明显问题是多个 myDemo 线程高频 R 状态切换：

```cmd
tid=2448 R_out=28531 
tid=2450 R_out=27168 
tid=2449 R_out=25379 
tid=2447 R_out=14981 
...
```

这说明你的程序里有多个线程处于可运行状态并频繁抢 CPU。后面接两个模型推理时，这个问题会被放大。

下一步建议先不要继续扩大 ftrace，而是在代码里做用户态分段计时，优先看这几段：

```cmd
readImx415Frame:
  DQBUF wait
  cvtColor
  QBUF

runImx415StreamingLoop:
  addStreamingData

StreamingManager::streamingWorker:
  queue wait/pop
  frame.clone
  drawDetections
  cv::resize
  encodeFrame
  writeEncodedFrame
  sleep_for
```

尤其是 StreamingManager 里现在每帧有：

```c++
cv::Mat frame = data.frame.clone();
cv::resize(frame, frame, cv::Size(config_.width, config_.height));
last_frame = frame.clone();
encodeFrame(...);
writeEncodedFrame(...);
sleep_for(1000 / fps);
```

这里很可能是队列长期 queue size: 11 的原因。

## 当前问题排查方向：

目前怀疑队列长期爆满这个问题可能是在NV12->BGR转换和StreamingManager这两个区域排查。

1. 给关键进程做命名处理；
2. readImx415Frame 针对加入队列（QBUF）取出队列（DQBUF）以及格式转换（NV12 转 BGR）分段计时；
3. 再看 StreamingManager；
4. 最后用 perf/ftrace 辅助确认。

## 加入时间戳和分段计时后：

日志：

```cmd
[IMX415] total=90, fps=28.0503, read_avg_ms=9.2076 ms, read_max_ms=209.894 ms, dqbuf_avg_ms=4.1421, dqbuf_max_ms=190.688, cvt_avg_ms=4.82991, cvt_max_ms=23.6, qbuf_avg_ms=0.225096, qbuf_max_ms=2.66417
Added frame to streaming queue, queue size: 11
Sent frame 100 to RTSP (MPP encoded, size=209 bytes)
[IMX415] total=180, fps=29.7118, read_avg_ms=5.28045 ms, read_max_ms=26.1239 ms, dqbuf_avg_ms=0.0162676, dqbuf_max_ms=0.058627, cvt_avg_ms=5.05092, cvt_max_ms=25.8524, qbuf_avg_ms=0.203348, qbuf_max_ms=0.43868
Added frame to streaming queue, queue size: 11
[IMX415] total=270, fps=29.6173, read_avg_ms=5.05914 ms, read_max_ms=20.0485 ms, dqbuf_avg_ms=0.0157828, dqbuf_max_ms=0.033251, cvt_avg_ms=4.82015, cvt_max_ms=19.7399, qbuf_avg_ms=0.212846, qbuf_max_ms=0.490015
Added frame to streaming queue, queue size: 11
Sent frame 200 to RTSP (MPP encoded, size=140 bytes)
[IMX415] total=360, fps=29.8387, read_avg_ms=5.86182 ms, read_max_ms=24.5143 ms, dqbuf_avg_ms=0.0155074, dqbuf_max_ms=0.032667, cvt_avg_ms=5.62736, cvt_max_ms=24.1882, qbuf_avg_ms=0.209089, qbuf_max_ms=0.592977
Added frame to streaming queue, queue size: 11
[IMX415] total=450, fps=29.6566, read_avg_ms=5.71902 ms, read_max_ms=26.9073 ms, dqbuf_avg_ms=0.0148755, dqbuf_max_ms=0.030917, cvt_avg_ms=5.47597, cvt_max_ms=26.6866, qbuf_avg_ms=0.214045, qbuf_max_ms=0.481848
Sent frame 300 to RTSP (MPP encoded, size=540 bytes)
Added frame to streaming queue, queue size: 11
[IMX415] total=540, fps=29.7998, read_avg_ms=5.41633 ms, read_max_ms=24.8421 ms, dqbuf_avg_ms=0.015219, dqbuf_max_ms=0.032084, cvt_avg_ms=5.17853, cvt_max_ms=24.6962, qbuf_avg_ms=0.212673, qbuf_max_ms=0.607852
Added frame to streaming queue, queue size: 11
[IMX415] total=630, fps=29.6584, read_avg_ms=4.84044 ms, read_max_ms=24.7496 ms, dqbuf_avg_ms=0.0157051, dqbuf_max_ms=0.063586, cvt_avg_ms=4.61732, cvt_max_ms=24.5827, qbuf_avg_ms=0.198296, qbuf_max_ms=0.587435
Sent frame 400 to RTSP (MPP encoded, size=214 bytes)
Added frame to streaming queue, queue size: 11
[IMX415] total=720, fps=29.7299, read_avg_ms=6.14123 ms, read_max_ms=35.5034 ms, dqbuf_avg_ms=0.0149694, dqbuf_max_ms=0.034709, cvt_avg_ms=5.92194, cvt_max_ms=35.3085, qbuf_avg_ms=0.193326, qbuf_max_ms=0.413597
Sent frame 500 to RTSP (MPP encoded, size=180 bytes)
Added frame to streaming queue, queue size: 11
[IMX415] total=810, fps=29.7232, read_avg_ms=5.11551 ms, read_max_ms=25.511 ms, dqbuf_avg_ms=0.0187691, dqbuf_max_ms=0.246175, cvt_avg_ms=4.86741, cvt_max_ms=25.2841, qbuf_avg_ms=0.218975, qbuf_max_ms=0.596186
[IMX415] total=900, fps=29.7511, read_avg_ms=5.37939 ms, read_max_ms=24.1718 ms, dqbuf_avg_ms=0.0159403, dqbuf_max_ms=0.05746, cvt_avg_ms=5.1357, cvt_max_ms=23.8725, qbuf_avg_ms=0.217796, qbuf_max_ms=0.482724
Added frame to streaming queue, queue size: 11
Sent frame 600 to RTSP (MPP encoded, size=646 bytes)
[IMX415] total=990, fps=29.7035, read_avg_ms=4.50473 ms, read_max_ms=23.1041 ms, dqbuf_avg_ms=0.0152903, dqbuf_max_ms=0.030334, cvt_avg_ms=4.27736, cvt_max_ms=22.8608, qbuf_avg_ms=0.20228, qbuf_max_ms=0.59181
Added frame to streaming queue, queue size: 11
[IMX415] total=1080, fps=29.6103, read_avg_ms=4.72055 ms, read_max_ms=26.9322 ms, dqbuf_avg_ms=0.0162042, dqbuf_max_ms=0.053376, cvt_avg_ms=4.45374, cvt_max_ms=26.555, qbuf_avg_ms=0.240402, qbuf_max_ms=1.12674
Added frame to streaming queue, queue size: 11
Sent frame 700 to RTSP (MPP encoded, size=132 bytes)
```

可以看出：

DQBUF   ≈ 0.016 ms
cvtColor≈ 5.05 ms
QBUF    ≈ 0.20 ms

这说明V4L2的取帧速度是很快的，这里就不是主要的瓶颈，采集线程稳定30FPS左右出帧，推流端消费不是很快。那现在需要找到推流线程为什么消费不过来，导致 queue size 长期 11的原因在哪？接下来需要针对StreamingLoader进行分段处理。

对于**streamingWorker()**而言里面涉及到的操作主要是：clone，drawDetections，resize，encodeFrame，writeEncodedFrame，sleep_for。下一步是给这些操作进行计时处理。

```cmd
Streaming config:
  RTMP    : 0 (rtmp://192.168.137.1/live/livestream)
  RTSP    : 1 (rtsp://192.168.137.1:8554/mystream)
  GB28181 : 0 (/home/quark/My_Projects/IMX415optimizer/gb28181.conf)

StreamLoaderManager created
H.264 extradata from MPP, size: 37 bytes
Streaming started
VIDIOC_S_PARM failed, errno=25
IMX415 opened: /dev/video33, 1920x1080, format=0x3231564e, plane=1
rga_api version 1.10.1_[0]
[IMX415] total=90, fps=27.9931, read_avg_ms=11.8855 ms, read_max_ms=211.742 ms, dqbuf_avg_ms=4.74175, dqbuf_max_ms=195.158, cvt_avg_ms=6.92835, cvt_max_ms=28.2782, qbuf_avg_ms=0.206027, qbuf_max_ms=0.599394
Added frame to streaming queue, queue size: 11
[RTSP_WORKER] total=90, fps=17.2745, wait_avg_ms=0.103652, wait_max_ms=7.93212, clone_avg_ms=1.7472, clone_max_ms=9.86535, draw_avg_ms=0.73281, draw_max_ms=1.4785, resize_avg_ms=13.595, resize_max_ms=53.1161, last_clone_avg_ms=0.602262, last_clone_max_ms=1.36796, encode_avg_ms=5.13969, encode_max_ms=7.76529, write_avg_ms=0.121182, write_max_ms=1.36242, total_avg_ms=22.0558, total_max_ms=62.5214, reuse_last=0, encode_ok=90, write_ok=90
Sent frame 100 to RTSP (MPP encoded, size=633 bytes)
[IMX415] total=180, fps=29.7769, read_avg_ms=5.63893 ms, read_max_ms=23.435 ms, dqbuf_avg_ms=0.0156435, dqbuf_max_ms=0.03996, cvt_avg_ms=5.40186, cvt_max_ms=23.2997, qbuf_avg_ms=0.212449, qbuf_max_ms=0.640228
Added frame to streaming queue, queue size: 11
[IMX415] total=270, fps=29.7583, read_avg_ms=5.52293 ms, read_max_ms=24.3765 ms, dqbuf_avg_ms=0.016337, dqbuf_max_ms=0.040543, cvt_avg_ms=5.27342, cvt_max_ms=24.0501, qbuf_avg_ms=0.222259, qbuf_max_ms=0.998407
[RTSP_WORKER] total=180, fps=18.8976, wait_avg_ms=0.0151315, wait_max_ms=0.034126, clone_avg_ms=1.65715, clone_max_ms=4.73302, draw_avg_ms=0.711273, draw_max_ms=2.09715, resize_avg_ms=11.3561, resize_max_ms=51.7128, last_clone_avg_ms=0.650193, last_clone_max_ms=1.5978, encode_avg_ms=5.21004, encode_max_ms=8.66219, write_avg_ms=0.111699, write_max_ms=0.552851, total_avg_ms=19.7256, total_max_ms=59.9795, reuse_last=0, encode_ok=90, write_ok=90
Added frame to streaming queue, queue size: 11
Sent frame 200 to RTSP (MPP encoded, size=290 bytes)
[IMX415] total=360, fps=29.6972, read_avg_ms=5.11649 ms, read_max_ms=22.5613 ms, dqbuf_avg_ms=0.0161427, dqbuf_max_ms=0.097128, cvt_avg_ms=4.88649, cvt_max_ms=22.3257, qbuf_avg_ms=0.205343, qbuf_max_ms=0.631478
Added frame to streaming queue, queue size: 11
[RTSP_WORKER] total=270, fps=18.6379, wait_avg_ms=0.0147254, wait_max_ms=0.029751, clone_avg_ms=1.57686, clone_max_ms=3.05851, draw_avg_ms=0.707702, draw_max_ms=1.49455, resize_avg_ms=11.9874, resize_max_ms=52.3631, last_clone_avg_ms=0.676745, last_clone_max_ms=1.45167, encode_avg_ms=5.33928, encode_max_ms=8.84128, write_avg_ms=0.103439, write_max_ms=0.36722, total_avg_ms=20.4195, total_max_ms=59.3458, reuse_last=0, encode_ok=90, write_ok=90
[IMX415] total=450, fps=29.7165, read_avg_ms=6.20872 ms, read_max_ms=27.4603 ms, dqbuf_avg_ms=0.0162237, dqbuf_max_ms=0.079627, cvt_avg_ms=5.9663, cvt_max_ms=27.1409, qbuf_avg_ms=0.217263, qbuf_max_ms=1.28221
Sent frame 300 to RTSP (MPP encoded, size=367 bytes)
Added frame to streaming queue, queue size: 11
[IMX415] total=540, fps=29.7995, read_avg_ms=5.0103 ms, read_max_ms=21.8718 ms, dqbuf_avg_ms=0.0152676, dqbuf_max_ms=0.040251, cvt_avg_ms=4.77864, cvt_max_ms=21.6974, qbuf_avg_ms=0.207297, qbuf_max_ms=0.982072
[RTSP_WORKER] total=360, fps=18.4483, wait_avg_ms=0.0153162, wait_max_ms=0.025376, clone_avg_ms=1.66183, clone_max_ms=3.16877, draw_avg_ms=0.780144, draw_max_ms=1.59138, resize_avg_ms=12.4909, resize_max_ms=64.0562, last_clone_avg_ms=0.636076, last_clone_max_ms=1.54151, encode_avg_ms=5.27957, encode_max_ms=8.32093, write_avg_ms=0.0997516, write_max_ms=0.319386, total_avg_ms=20.9772, total_max_ms=76.1173, reuse_last=0, encode_ok=90, write_ok=90
Added frame to streaming queue, queue size: 11
[IMX415] total=630, fps=29.6709, read_avg_ms=4.72756 ms, read_max_ms=23.9504 ms, dqbuf_avg_ms=0.0152838, dqbuf_max_ms=0.031792, cvt_avg_ms=4.50139, cvt_max_ms=23.6479, qbuf_avg_ms=0.202105, qbuf_max_ms=0.43168
Sent frame 400 to RTSP (MPP encoded, size=318 bytes)
Added frame to streaming queue, queue size: 11
[IMX415] total=720, fps=29.7072, read_avg_ms=5.66792 ms, read_max_ms=22.7641 ms, dqbuf_avg_ms=0.0155625, dqbuf_max_ms=0.044334, cvt_avg_ms=5.44245, cvt_max_ms=22.4875, qbuf_avg_ms=0.200903, qbuf_max_ms=0.483307
[RTSP_WORKER] total=450, fps=18.0825, wait_avg_ms=0.015582, wait_max_ms=0.036459, clone_avg_ms=1.55058, clone_max_ms=3.83233, draw_avg_ms=0.75164, draw_max_ms=1.93235, resize_avg_ms=14.0799, resize_max_ms=56.0716, last_clone_avg_ms=0.586257, last_clone_max_ms=1.40471, encode_avg_ms=5.03437, encode_max_ms=7.83879, write_avg_ms=0.0980883, write_max_ms=0.382388, total_avg_ms=22.1305, total_max_ms=64.2992, reuse_last=0, encode_ok=90, write_ok=90
Added frame to streaming queue, queue size: 11
Sent frame 500 to RTSP (MPP encoded, size=343 bytes)
[IMX415] total=810, fps=29.7018, read_avg_ms=5.32689 ms, read_max_ms=21.7256 ms, dqbuf_avg_ms=0.0154038, dqbuf_max_ms=0.033542, cvt_avg_ms=5.09908, cvt_max_ms=21.3945, qbuf_avg_ms=0.20375, qbuf_max_ms=0.413888
[RTSP_WORKER] total=540, fps=18.3404, wait_avg_ms=0.0155918, wait_max_ms=0.066211, clone_avg_ms=1.68831, clone_max_ms=3.13085, draw_avg_ms=0.756726, draw_max_ms=1.61151, resize_avg_ms=12.7339, resize_max_ms=53.4505, last_clone_avg_ms=0.690599, last_clone_max_ms=1.36125, encode_avg_ms=5.30207, encode_max_ms=7.19389, write_avg_ms=0.110381, write_max_ms=0.489723, total_avg_ms=21.312, total_max_ms=62.0978, reuse_last=0, encode_ok=90, write_ok=90
[IMX415] total=900, fps=29.6784, read_avg_ms=5.17114 ms, read_max_ms=21.531 ms, dqbuf_avg_ms=0.0161491, dqbuf_max_ms=0.069418, cvt_avg_ms=4.93488, cvt_max_ms=21.3653, qbuf_avg_ms=0.210789, qbuf_max_ms=0.521224
Added frame to streaming queue, queue size: 11
Sent frame 600 to RTSP (MPP encoded, size=291 bytes)
[IMX415] total=990, fps=29.7913, read_avg_ms=5.7838 ms, read_max_ms=24.5446 ms, dqbuf_avg_ms=0.0140296, dqbuf_max_ms=0.041127, cvt_avg_ms=5.55574, cvt_max_ms=24.3293, qbuf_avg_ms=0.204863, qbuf_max_ms=0.418554
Added frame to streaming queue, queue size: 11
[RTSP_WORKER] total=630, fps=18.5754, wait_avg_ms=0.0145644, wait_max_ms=0.025376, clone_avg_ms=1.56437, clone_max_ms=3.22652, draw_avg_ms=0.689194, draw_max_ms=1.51846, resize_avg_ms=12.4895, resize_max_ms=57.1545, last_clone_avg_ms=0.645622, last_clone_max_ms=1.32975, encode_avg_ms=5.1604, encode_max_ms=7.15947, write_avg_ms=0.099805, write_max_ms=0.499057, total_avg_ms=20.677, total_max_ms=67.3751, reuse_last=0, encode_ok=90, write_ok=90
[IMX415] total=1080, fps=29.7743, read_avg_ms=4.66445 ms, read_max_ms=22.9991 ms, dqbuf_avg_ms=0.0152514, dqbuf_max_ms=0.069711, cvt_avg_ms=4.42867, cvt_max_ms=22.7813, qbuf_avg_ms=0.211662, qbuf_max_ms=0.595894
Added frame to streaming queue, queue size: 11
Sent frame 700 to RTSP (MPP encoded, size=516 bytes)
[RTSP_WORKER] total=720, fps=18.437, wait_avg_ms=0.0160292, wait_max_ms=0.039084, clone_avg_ms=1.68332, clone_max_ms=3.84925, draw_avg_ms=0.794054, draw_max_ms=1.78243, resize_avg_ms=12.5226, resize_max_ms=56.5258, last_clone_avg_ms=0.66011, last_clone_max_ms=1.41784, encode_avg_ms=5.26103, encode_max_ms=6.76255, write_avg_ms=0.11503, write_max_ms=0.417971, total_avg_ms=21.0663, total_max_ms=64.4734, reuse_last=0, encode_ok=90, write_ok=90
[IMX415] total=1170, fps=29.6885, read_avg_ms=5.40176 ms, read_max_ms=23.2057 ms, dqbuf_avg_ms=0.0176659, dqbuf_max_ms=0.180255, cvt_avg_ms=5.16091, cvt_max_ms=22.9062, qbuf_avg_ms=0.213989, qbuf_max_ms=0.909445
Added frame to streaming queue, queue size: 11
[IMX415] total=1260, fps=29.7213, read_avg_ms=5.75183 ms, read_max_ms=22.6545 ms, dqbuf_avg_ms=0.0174519, dqbuf_max_ms=0.256966, cvt_avg_ms=5.53078, cvt_max_ms=22.3669, qbuf_avg_ms=0.195267, qbuf_max_ms=0.410097
Sent frame 800 to RTSP (MPP encoded, size=263 bytes)
Added frame to streaming queue, queue size: 11
[RTSP_WORKER] total=810, fps=18.4162, wait_avg_ms=0.0440195, wait_max_ms=2.52446, clone_avg_ms=1.45424, clone_max_ms=3.30848, draw_avg_ms=0.657479, draw_max_ms=1.61909, resize_avg_ms=13.1129, resize_max_ms=54.7547, last_clone_avg_ms=0.592295, last_clone_max_ms=1.37613, encode_avg_ms=5.11605, encode_max_ms=7.07197, write_avg_ms=0.0955077, write_max_ms=0.338052, total_avg_ms=21.0851, total_max_ms=63.3754, reuse_last=0, encode_ok=90, write_ok=90
[IMX415] total=1350, fps=29.7159, read_avg_ms=5.64293 ms, read_max_ms=23.4347 ms, dqbuf_avg_ms=0.0163144, dqbuf_max_ms=0.04521, cvt_avg_ms=5.41087, cvt_max_ms=23.2166, qbuf_avg_ms=0.206778, qbuf_max_ms=0.416222
Added frame to streaming queue, queue size: 11
[IMX415] total=1440, fps=29.7675, read_avg_ms=5.08064 ms, read_max_ms=25.7875 ms, dqbuf_avg_ms=0.0158542, dqbuf_max_ms=0.069711, cvt_avg_ms=4.84186, cvt_max_ms=25.5649, qbuf_avg_ms=0.213724, qbuf_max_ms=0.599978
Sent frame 900 to RTSP (MPP encoded, size=297 bytes)
[RTSP_WORKER] total=900, fps=18.4923, wait_avg_ms=0.0149995, wait_max_ms=0.025085, clone_avg_ms=1.63118, clone_max_ms=3.14864, draw_avg_ms=0.770667, draw_max_ms=1.57359, resize_avg_ms=12.5225, resize_max_ms=54.1734, last_clone_avg_ms=0.651918, last_clone_max_ms=1.37525, encode_avg_ms=5.24377, encode_max_ms=7.50044, write_avg_ms=0.0958998, write_max_ms=0.290801, total_avg_ms=20.9443, total_max_ms=63.9454, reuse_last=0, encode_ok=90, write_ok=90
```

这里日志总体看下来可以知道：

采集端：

```cmd
IMX415 fps ≈ 29.7
dqbuf_avg_ms ≈ 0.015 ms
cvt_avg_ms ≈ 4.5-5.9 ms
qbuf_avg_ms ≈ 0.20 ms
```

同时推流端

```cmd
RTSP_WORKER fps ≈ 18.36
total_avg_ms ≈ 21.04 ms
resize_avg_ms ≈ 12.69 ms
encode_avg_ms ≈ 5.21 ms
clone_avg_ms ≈ 1.62 ms
write_avg_ms ≈ 0.10 ms
```

刚刚看代码的时候发现一段：

```c++
std::this_thread::sleep_for(std::chrono::milliseconds(1000 / config_.fps));
```

那这里意味着回固定睡33ms。将这里注释掉再测试看看：

#### 找到问题：

取出后，卡顿明显减少（忘了这玩意前面用来解码视频处理后进行速度控制的）汗颜。

### 稳定性测试：

对于程序运行观测以下指标：

```cmd
IMX415 fps       29-30
RTSP_WORKER fps 29-30
queue size      0-2 #左右
reuse_last      0
encode_ok       90
write_ok        90
temperature		about 50
```

跑了个10min的log，目前在assets文件夹下面。

TOP -H -P:

```cmd
rtsp_stream     34%
多个 imx415_loop 22%-31%
mpp_h264e       0.7%
```

检索STACK的时候发现多数imx415_loop基本上都是保持在等锁/条件变量，后续这边刻意考虑下优化命名。

log总结：

采集端：

```
IMX415 fps avg = 29.752 
read_avg_ms avg = 2.717 ms 
dqbuf_avg_ms avg = 0.095 ms 
cvt_avg_ms avg = 2.404 ms 
qbuf_avg_ms avg = 0.209 ms
```

推流端：

```
RTSP_WORKER fps avg = 29.750 
wait_avg_ms avg = 18.235 ms 
resize_avg_ms avg = 8.115 ms 
encode_avg_ms avg = 4.844 ms 
write_avg_ms avg = 0.074 ms 
total_avg_ms avg = 33.594 ms
```

队列：

```
queue size: avg = 1, min = 1, max = 1
```

温度：

```
约 50°C
```

内存：

```
RES ≈ 47.8 MB MEM ≈ 1.2%
```

#### 总结：

去除 StreamingManager::streamingWorker() 末尾固定 sleep_for 后，10 分钟 RTSP 推流稳定运行。IMX415 采集端平均约 29.75FPS，RTSP_WORKER 平均约 29.75FPS，streaming queue 长期保持 size=1。说明推流消费端已经能跟上采集端，原先 queue size=11 的问题由 “处理耗时 + 固定 sleep_for” 叠加造成。

当前主要 CPU 开销仍集中在推流端 resize，平均约 8.1ms；MPP encode 平均约 4.8ms；RTSP write 平均约 0.07ms，不是瓶颈。系统温度约 50°C，内存占用约 48MB，稳定性良好。