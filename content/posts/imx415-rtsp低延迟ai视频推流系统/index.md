---
date: 2026-06-12T10:00:00+08:00
draft: false
title: "基于 RK3588 的 IMX415 低延迟 AI 视频推流系统"
tags: ["RK3588", "IMX415", "RTSP", "RKNN", "NPU", "视频推流", "嵌入式Linux"]
categories: ["嵌入式Linux"]
summary: "基于 RK3588 / IMX415 摄像头，实现 1080p@30fps RTSP 推流，并集成 person + helmet 双模型实时检测。从零调试到「稳定 30fps + 300–400ms 端到端延迟 + AI 异步推理无顿挫」的完整记录。"
---

基于 RK3588 / IMX415 摄像头，实现 1080p@30fps RTSP 推流，并集成 person + helmet 双模型实时检测。
从零调试到「稳定 30fps + 300–400ms 端到端延迟 + AI 异步推理无顿挫」。

---

## 项目成果

| 指标       | 起点                   | 最终                               |
| ---------- | ---------------------- | ---------------------------------- |
| 端到端延迟 | ~1s（且持续增长到 3s） | **300–400ms，稳定不增长**          |
| 推流帧率   | 24.5fps                | **~29.8fps**                       |
| 推理顿挫   | 推理命中帧卡 33–67ms   | **基本消除**                       |
| AI 模型    | 无                     | **person + helmet 双模型并行推理** |

---

## 阶段工作记录

### 阶段 1 — IMX415 冒烟测试 + debug 卡顿问题
📄 [日志笔记](/posts/1-imx415冒烟测试-debug卡顿/) | [测试日志](/posts/1-imx415冒烟测试-debug卡顿/assets/imx415_rtsp_10min_no_sleep.log)

- 打通 IMX415 V4L2 采集 → MPP H.264 编码 → FFmpeg RTSP 推流完整链路
- 发现推流卡顿、延迟约 1s、`streaming_queue_` 长期积压到 11
- 通过日志定位根因：`streamingWorker` 中 `sleep_for(33ms)` 把消费速率人为限制到 ~18fps，导致队列堆积
- 移除该 `sleep` 后，推流稳定 30fps，队列降到 1，延迟恢复正常
- 使用 `ftrace` 采集 V4L2 内核调度事件，验证采集端本身健康

---

### 阶段 2 — 接入 person 模型
📄 [日志笔记](/posts/2-接入person模型/)

测试日志：
[同步推理](/posts/2-接入person模型/assets/imx415_person_sync_test.log) |
[跳帧3复用](/posts/2-接入person模型/assets/imx415_person_skip3_test.log) |
[跳帧6复用](/posts/2-接入person模型/assets/imx415_person_skip6_test.log) |
[跳帧3无复用](/posts/2-接入person模型/assets/imx415_person_skip3_no_reuse_test.log) |
[跳帧6无复用](/posts/2-接入person模型/assets/imx415_person_skip6_no_reuse_test.log)

- 接入 person RKNN 模型，绑定 NPU core0
- 验证同步推理导致帧率从 30fps 下降到约 23fps
- 实现「跳帧推理 + 缓存上一次结果」策略（interval=6）
- 确认 `reuse_last=0`（RTSP worker 禁用上一帧复用），消除画面异常
- 稳定帧率恢复到约 29.3fps，队列 size=1

---

### 阶段 3 — 接入 helmet 模型
📄 [日志笔记](/posts/3-接入helmet模型/)

测试日志：
[加载smoke](/posts/3-接入helmet模型/assets/imx415_helmet_model_load_test.log) |
[smoke推理](/posts/3-接入helmet模型/assets/imx415_person_helmet_smoke_test.log) |
[interval=10](/posts/3-接入helmet模型/assets/imx415_person_helmet_interval10_test.log) |
[stagger错峰](/posts/3-接入helmet模型/assets/imx415_person_helmet_stagger_test.log) |
[interval=12](/posts/3-接入helmet模型/assets/imx415_person_helmet_interval12_test.log) |
[25fps验证](/posts/3-接入helmet模型/assets/imx415_person_helmet_interval12_fps25_test.log)

- helmet 模型绑定 NPU core1，两模型分核并行
- 发现 person + helmet 同帧触发时单帧卡顿 ~67ms（双模型叠加）
- 通过错峰调度（`offset=3`、`interval=12`）避免同帧推理
- 分析 interval=10/12 在 30fps 下的稳定性边界
- 确认同步双模型在 30fps 目标下偏紧，引出下一阶段 PTS 问题

---

### 阶段 4 — debug 延迟逐渐增大问题
📄 [日志笔记](/posts/4-debug延迟逐渐增大问题/) | [修复后测试日志](/posts/4-debug延迟逐渐增大问题/assets/imx415_person_helmet_interval12_fps30_PTS_Modified_test.log)

- 定位"延迟从 1s 涨到 3s"根因：推流 PTS 用帧计数（`frame_index_`）生成，声明 30fps 而真实只有约 29fps，时间线与真实时钟每秒偏差 ~0.03s，播放器缓冲持续垫高
- **修复方案**：改用 `steady_clock` 真实墙钟时间生成 PTS，同步修改时基为 `{1,90000}` 90kHz
- 修复后长跑 12 分钟（2 万帧），fps 始终 ~29 无衰减，延迟稳定不再增长

---

### 阶段 5 — 延迟问题探索（P0–P3 优化）
📄 [日志笔记](/posts/5-延迟问题探索/)

测试日志：
[P0基线](/posts/5-延迟问题探索/assets/IMX415_P0.log) |
[P1 muxdelay](/posts/5-延迟问题探索/assets/IMX415_P1.log) |
[P2 1080p去resize](/posts/5-延迟问题探索/assets/IMX415_P2.log) |
[综合测试](/posts/5-延迟问题探索/assets/IMX415_test.log)

- **P0**：低缓冲 ffplay 验证，确认约 500ms 大头在播放器缓冲，推流端已健康
- **P1**：`muxdelay` 0.1→0，`muxpreload=0`，减少 muxer 固定缓冲 ~100ms
- **P2**：GOP 30→15，加快起播和抖动恢复
- **P3**：输出分辨率 720p→1080p，`stream_config.width/height` 改原生尺寸，`cv::resize` 自动跳过，消除 `resize_avg≈11ms`、`resize_max≈53ms` 尖峰
- 综合延迟从约 1s 压到约 500ms，`resize_avg=0` 确认跳过生效

---

### 阶段 6 — 延迟优化：模型异步推理
📄 [日志笔记](/posts/6-延迟优化-模型异步推理/) | [测试日志](/posts/6-延迟优化-模型异步推理/assets/IMX415_model_async.log)

- 分析 500ms 剩余延迟分布：推理 inline 造成采集端 fps 抖动 + V4L2 缓冲堆积 ~130ms
- `buffer_count` 4→2，削减采集队列堆积
- **核心改动**：双 worker 异步推理架构
  - `AiSharedState` 共享结构体，三把独立锁分别保护"最新帧/person 结果/helmet 结果"
  - `aiModelWorker` 通用线程函数，person/helmet 各启一个线程，分别绑 NPU core0/core1
  - 采集线程只做"发布最新帧 + 取最近缓存结果 + 送流"，从不等待推理
  - 「单槽覆盖」策略：新帧直接覆盖旧帧，AI 处理不过来时丢旧帧，延迟不累积
- 效果：fps 从 24.5（抖）→ 29.8（稳），`read_max` 尖峰基本消失，端到端延迟 500ms → **300–400ms**，周期性顿挫消除

---

## 关键技术点

- **V4L2 多平面采集（MPLANE）**：NV12 格式采集，RGA/OpenCV 转 BGR
- **Rockchip MPP 硬编码**：H.264 硬件编码，GOP/bitrate 配置
- **FFmpeg RTSP 推流**：`av_packet_rescale_ts`、时基配置、`muxdelay` 调优
- **墙钟 PTS**：`steady_clock` 生成真实时间戳，防止媒体时间线漂移
- **RKNN NPU 推理**：多模型分核绑定（`rknn_set_core_mask`），RGA 预处理加速
- **异步推理架构**：生产者-消费者「单槽覆盖」模式，`std::mutex` 细粒度锁，`std::atomic<bool>` 退出控制

---

## 项目结构

```text
IMX415optimizer/
├── main.cpp                    # 主逻辑：采集循环 + 异步推理 + 推流
├── lib/
│   ├── streaming_manager.cpp   # RTSP/RTMP 推流，PTS 生成，MPP 编码调用
│   ├── imx415_camera.cpp       # V4L2 摄像头采集
│   ├── mpp_encoder.cpp         # Rockchip MPP H.264 编码器
│   └── ...
├── include/
│   ├── rknnPool.hpp            # RKNN 模型推理封装，NPU 核绑定
│   ├── streaming_manager.h
│   └── ...
├── assets/models/              # RKNN 模型文件
└── run_streams.sh              # 启动脚本（设置环境变量）
```
