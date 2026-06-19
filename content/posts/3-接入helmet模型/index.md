---
date: 2026-06-10T10:00:00+08:00
draft: false
title: "阶段3 — 接入 helmet 模型"
tags: ["RK3588","RKNN","NPU","helmet检测"]
categories: ["嵌入式Linux"]
summary: "接入 helmet 模型绑定 NPU core1，双模型分核并行，通过错峰调度（offset=3、interval=12）避免同帧推理卡顿。"
---

# 接入第二个helmet模型

更改如下：

```c++
void runImx415StreamingLoop(rknn_lite *person_model, rknn_lite *helmet_model)
```

```c++
rknn_lite *imx415_helmet_model = new rknn_lite(model_helmet.c_str(), 1, 2, 1);
```

新增helmet调试参数：

```c++
const int HELMET_INFER_INTERVAL = 30; // 每 30 帧执行一次 helmet 推理，先做低频 smoke test
int helmet_frame_index = 0; // 记录 helmet 当前处理到第几帧
detect_result_group_t last_helmet_results; // 保存上一次 helmet 检测结果，非推理帧复用
memset(&last_helmet_results, 0, sizeof(detect_result_group_t)); // 初始化 helmet 缓存结果

int helmet_infer_count = 0; // 统计本轮 90 帧内真实执行了多少次 helmet 推理
int helmet_reuse_count = 0; // 统计本轮 90 帧内复用了多少次 helmet 检测结果
double stat_helmet_infer_ms_sum = 0.0; // 累加 helmet 推理耗时
double stat_helmet_infer_ms_max = 0.0; // 记录 helmet 推理最大耗时
double last_helmet_infer_ms = 0.0; // 记录最近一次 helmet 推理耗时
```

执行推理，同时打时间戳，设置好执行判定的帧数偏移量：

```c++
detect_result_group_t helmet_results; // 当前帧最终要送给 StreamingManager 的 helmet 检测结果
memset(&helmet_results, 0, sizeof(detect_result_group_t)); // 先清零，避免残留旧结果

helmet_frame_index++; // 每处理一帧，helmet 帧序号加 1

bool do_helmet_infer = false; // 标记当前帧是否执行 helmet 推理
if (helmet_model != nullptr && (helmet_frame_index % HELMET_INFER_INTERVAL) == 0) // 模型有效，并且命中 helmet 推理间隔
{
    do_helmet_infer = true; // 本帧执行 helmet 推理
}

double helmet_infer_ms = 0.0; // 当前帧 helmet 推理耗时；复用帧保持 0

if (do_helmet_infer) // 当前帧需要执行 helmet 推理
{
    auto helmet_infer_start = std::chrono::steady_clock::now(); // 记录 helmet 推理开始时间

    helmet_model->ori_img = frame.clone(); // 给 helmet 模型一份当前帧图像
    helmet_model->interf(helmet_results); // 执行 helmet 模型推理，结果写入 helmet_results

    auto helmet_infer_end = std::chrono::steady_clock::now(); // 记录 helmet 推理结束时间
    helmet_infer_ms = std::chrono::duration<double, std::milli>(
                          helmet_infer_end - helmet_infer_start)
                          .count(); // 计算 helmet 推理耗时，单位 ms

    last_helmet_infer_ms = helmet_infer_ms; // 保存最近一次 helmet 推理耗时
    last_helmet_results = helmet_results; // 缓存 helmet 检测结果，给后续帧复用

    helmet_infer_count++; // 本统计窗口 helmet 推理次数加 1
    stat_helmet_infer_ms_sum += helmet_infer_ms; // 累加 helmet 推理耗时

    if (helmet_infer_ms > stat_helmet_infer_ms_max) // 如果本次 helmet 推理更慢
    {
        stat_helmet_infer_ms_max = helmet_infer_ms; // 更新 helmet 最大推理耗时
    }
}
else // 当前帧不执行 helmet 推理
{
    helmet_results = last_helmet_results; // 复用上一次 helmet 检测结果
    helmet_reuse_count++; // helmet 复用次数加 1
}
```

将结果送到推流：

```c++
stream_data.helmet_results = helmet_results; // 使用 helmet 推理结果或缓存结果画框
```

==此时发现画面延迟大概上升了3-4s，然后推理的时候有明显卡顿，还有点发绿。==

注意到：每次 helmet 推理都和 person 推理撞在同一帧上。所以调整interval和求余的结果。

测试了三组：

interval = 30， offset=15，在第 15、45、75 帧推理。效果：延迟已减小，但是helmet推理密度太小。

interval = 10， offset=5，在第 5、15、25 帧推理。效果：延迟已减小，一开始还行就是1s左右延迟，但是后面就有掉帧和发绿的情况了，猜测可能是相邻帧连续推理。

interval = 12， offset=3，未改善。

猜测长期跑29.xfps，然后时间戳逐步堆积导致延迟到后面从1s左右的延迟到了5s以上。