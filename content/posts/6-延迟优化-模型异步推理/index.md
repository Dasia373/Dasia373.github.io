---
date: 2026-06-11T16:00:00+08:00
draft: false
title: "阶段6 — 延迟优化：模型异步推理"
tags: ["RK3588","RKNN","NPU","异步推理","延迟优化"]
categories: ["嵌入式Linux"]
summary: "双 worker 异步推理架构，单槽覆盖策略，端到端延迟 500ms → 300–400ms，周期性顿挫消除。"
---

# AI模型异步推理

效果：

![优化效果](/posts/6-延迟优化-模型异步推理/assets/IMG_20260611_162653..jpg)

==可以看到这里优化到了从500mms以上优化到可以333ms左右==

异步推理优化思路：

创建一个状态共享结构体，然后往里面更新取得帧信息和推理帧信息，采集线程只需要把帧从里面发布最新帧，然后取得最近一次检测结果然后送到视频流，AI推理线程只需要对最新帧进行持续推理，然后把结果写入共享缓存中。

```c++
struct AiSharedState {
    std::mutex frame_mutex;              // 保护 latest_frame：采集线程写，两个 AI 线程读
    cv::Mat latest_frame;                // 最新一帧（只存 1 帧，新帧覆盖旧帧）
    bool has_frame = false;              // 是否已经有帧可用（防止启动初期读空）

    std::mutex person_mutex;             // 保护 person 结果
    detect_result_group_t person_results;// ai_person 写，采集线程读

    std::mutex helmet_mutex;             // 保护 helmet 结果
    detect_result_group_t helmet_results;// ai_helmet 写，采集线程读

    std::atomic<bool> stop{false};       // 退出标志，atomic 读写无需加锁

    AiSharedState() {
        memset(&person_results, 0, sizeof(person_results)); // 清零，避免首次读到脏数据
        memset(&helmet_results, 0, sizeof(helmet_results));
    }
};//三把独立锁（帧、person 结果、helmet 结果）分开，减少线程间争用，让采集/person/helmet 尽量并行。
```

推理线程函数：

```c++
static void aiModelWorker(rknn_lite *model,                 // 本线程负责的模型
                          AiSharedState *state,             // 共享状态（取帧用）
                          detect_result_group_t *out_results,// 结果写到哪
                          std::mutex *out_mutex,            // 保护该结果的锁
                          const char *thread_name) {        // 线程名，便于 top 观察
    pthread_setname_np(pthread_self(), thread_name);        // 设置线程名
    if (model == nullptr) {
        return;                                             // 模型无效直接退出
    }

    while (!state->stop.load()) {                           // 未收到停止信号就一直循环
        cv::Mat local;
        {
            std::lock_guard<std::mutex> lk(state->frame_mutex); // 加锁（仅拷贝瞬间持锁）
            if (state->has_frame) {
                local = state->latest_frame;                // 浅拷贝最新帧（引用计数，极快）
            }
        }                                                   // 出作用域自动解锁
        if (local.empty()) {                                // 还没有帧
            std::this_thread::sleep_for(std::chrono::milliseconds(5)); // 歇 5ms 再试，避免空转
            continue;
        }

        detect_result_group_t r;
        memset(&r, 0, sizeof(r));                           // 本轮结果先清零
        model->ori_img = local.clone();                     // 深拷贝给模型（interf 会画框改像素）
        model->interf(r);                                   // 真正推理，发生在锁外（约 33ms）

        {
            std::lock_guard<std::mutex> lk(*out_mutex);     // 加结果锁
            *out_results = r;                               // 值拷贝写回共享缓存
        }                                                   // 解锁
    }
}
```

启动线程：

```c++
AiSharedState ai_state;                                     // 三线程共享同一个实例
std::thread person_thread(aiModelWorker, person_model, &ai_state,
                          &ai_state.person_results, &ai_state.person_mutex, "ai_person");
std::thread helmet_thread(aiModelWorker, helmet_model, &ai_state,
                          &ai_state.helmet_results, &ai_state.helmet_mutex, "ai_helmet");
```

IMX415_LOOP采集部分：

```c++
// 1) 发布最新帧给 AI 线程（浅拷贝即可，cv::Mat 引用计数）
{
    std::lock_guard<std::mutex> lk(ai_state.frame_mutex);
    ai_state.latest_frame = frame;                          // 覆盖式：新帧盖旧帧，不排队
    ai_state.has_frame = true;
}

// 2) 读取最近一次检测结果（瞬间完成，不等推理）
detect_result_group_t person_results;
detect_result_group_t helmet_results;
{
    std::lock_guard<std::mutex> lk(ai_state.person_mutex);
    person_results = ai_state.person_results;               // 取走 ai_person 最近写的结果
}
{
    std::lock_guard<std::mutex> lk(ai_state.helmet_mutex);
    helmet_results = ai_state.helmet_results;               // 取走 ai_helmet 最近写的结果
}
```

退出收尾：

```c++
ai_state.stop.store(true);                                  // 通知 AI 线程退出循环
if (person_thread.joinable()) person_thread.join();         // 等线程真正结束
if (helmet_thread.joinable()) helmet_thread.join();         // 防止 ai_state 被提前销毁
```

效果如下：

| 指标       | 同步推理           | 异步推理         |
| :--------- | :----------------- | :--------------- |
| 采集 fps   | ~24.5（抖）        | ~29.8（稳）      |
| read_max   | 23–30ms 周期尖峰   | 7–19ms，尖峰消失 |
| 端到端延迟 | ~500ms             | 300–400ms        |
| 顿挫       | 推理命中帧明显卡顿 | 基本消除         |

---

## 推理耗时打点（rknn_run → 后处理结束）

原作者评估时关注 AI 推理耗时，要求打印「`rknn_run` 开始 → 后处理结束」的一段时间。在 `rknn_lite::interf()` 里用 `steady_clock` 打两个时间戳即可：

```c++
auto t_run_start = std::chrono::steady_clock::now();   // run 开始时刻
// ...
ret = rknn_run(rkModel, NULL);
// ...
post_process(...);

// 打印 rknn_run 开始 → 后处理结束 的耗时
auto t_post_end = std::chrono::steady_clock::now();
double run_post_ms = std::chrono::duration<double, std::milli>(t_post_end - t_run_start).count();
printf("[INFER id=%d] run->post = %.2f ms\n", id, run_post_ms);
```

实测结果（两个模型分别绑在不同 NPU 核上，异步并行）：

| 模型          | run→post 区间 | 大致均值 |
| :------------ | :------------ | :------- |
| id=0 (person) | 16~20 ms      | ~17.5 ms |
| id=1 (helmet) | 17~22 ms      | ~18.5 ms |

要点：

- 这段时间是 **NPU 推理 + CPU 后处理（NMS/坐标还原）之和**。
- 因为推理已经异步化（独立线程），这 17~20ms **完全在推流关键路径之外**，不计入端到端延迟。
- 两个模型分别跑在不同 NPU 核、并行执行，所以总时间 ≈ 单模型时间，而非相加。
- 若想进一步区分「NPU 慢还是后处理慢」，可再拆成 `run` 与 `post` 两段分别打点；YOLO 后处理在 CPU 上常占 5~8ms，往往是大头。

---

## 推流时间的可控延迟（直播式延迟缓冲）

直播「延迟 N 秒」的本质是一个**定时缓冲区**：帧先进队列、打上「应发出时刻」，到点才真正发出。

### 关键选择：在「编码后、写出前」做延迟

- 原始 1080p BGR 帧 ≈ 3MB/帧，缓存 10s@30fps ≈ 466MB，板子扛不住。
- 编码后的 H.264 包只有 5~8KB/帧，缓存 10s ≈ 2.4MB、30s ≈ 7MB，几乎零成本。
- 所以延迟缓冲放在编码之后、`writeEncodedFrame` 之前。

### 数据结构

```c++
struct DelayedPacket {
    std::vector<uint8_t> data;                        // 编码后字节（拷贝一份）
    int64_t pts_ms;                                   // 入队时确定的 PTS
    std::chrono::steady_clock::time_point release_at; // 应写出时刻
};

std::deque<DelayedPacket> delay_q_;
std::mutex delay_mtx_;
std::condition_variable delay_cv_;
std::atomic<int64_t> target_delay_ms_{0};             // 运行时可调
std::thread delay_writer_thread_;
```

### 入队（编码成功后，替换原来直接 write 的两行）

```c++
DelayedPacket dp;
dp.data.assign(encoded_frame.data(), encoded_frame.data() + packet_size);
dp.pts_ms     = current_pts_ms_;                      // 复用已有的墙钟 PTS
dp.release_at = std::chrono::steady_clock::now()
              + std::chrono::milliseconds(target_delay_ms_.load());
{
    std::lock_guard<std::mutex> lk(delay_mtx_);
    delay_q_.push_back(std::move(dp));
}
delay_cv_.notify_one();
```

### 出队写线程

```c++
void StreamingManager::delayWriter() {
    while (!should_stop_.load()) {
        DelayedPacket dp;
        {
            std::unique_lock<std::mutex> lk(delay_mtx_);
            delay_cv_.wait(lk, [&]{ return !delay_q_.empty() || should_stop_.load(); });
            if (should_stop_.load()) break;

            auto now = std::chrono::steady_clock::now();
            auto rel = delay_q_.front().release_at;
            if (now < rel) {
                delay_cv_.wait_until(lk, rel);        // 睡到该发的时刻
                continue;                             // 醒来重新判断
            }
            dp = std::move(delay_q_.front());
            delay_q_.pop_front();
        }
        // pts 用入队时确定的 dp.pts_ms，保证回放速度不被写出抖动扭曲
        writeEncodedFrameWithPts(rtmp_context_, dp.data.data(), dp.data.size(), dp.pts_ms, "RTMP");
        writeEncodedFrameWithPts(rtsp_context_, dp.data.data(), dp.data.size(), dp.pts_ms, "RTSP");
    }
}
```

把 `writeEncodedFrame` 里写死的 `pkt.pts = current_pts_ms_` 改成参数传入 `pts_ms` 即可。

### 为什么用「入队时刻的 PTS」

延迟恒定时，出队节奏 ≈ 入队节奏，播放速度自然正确。把 PTS 在入队（≈采集）时就钉死并随包带走，可让 PTS 间隔严格反映采集间隔，不受出队线程调度抖动影响——这是直播缓冲的标准做法。

### 运行时调延迟的边界

- **调大延迟**：新帧 `release_at` 更靠后，队列平滑变长，无副作用。
- **调小延迟**：已入队的帧会「过期」，若不处理会一次性突发涌出（快进感）。两种策略：
  1. 简单：接受短暂「快进追平」，几百毫秒内排空到新长度。
  2. 平滑：调小时不动已有帧，只让新帧用新延迟，队列自然漂移到新长度（更平滑但收敛慢）。
- 建议提供 `setStreamDelay(int ms)` 接口（写 `target_delay_ms_`），上层热更新。

### 注意点

- 延迟不改变 GOP 结构，新接入的 RTSP 播放器照常等下一个关键帧（GOP=fps/2≈15 帧 ≈0.5s），不受影响。
- 给 `delay_q_` 设保护上限（如 `target_delay_ms/帧周期 * 1.5`），异常时丢最旧帧，防止内存失控。
- 延迟统一作用于所有出口（RTMP/RTSP/GB28181），且与播放端缓冲解耦——这是相对「靠播放器缓冲凑延迟」的优势。