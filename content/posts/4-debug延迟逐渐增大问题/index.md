---
date: 2026-06-10T16:00:00+08:00
draft: false
title: "阶段4 — debug 延迟逐渐增大问题"
tags: ["RK3588","RTSP","PTS","FFmpeg"]
categories: ["嵌入式Linux"]
summary: "定位延迟从 1s 涨到 3s 的根因：PTS 用帧计数生成导致时间线漂移。改用 steady_clock 墙钟时间生成 PTS，延迟稳定不再增长。"
---

# 延迟越来越高分析

## 原因分析：

一开始猜测由于1/config.fps这种固定间距时基，可能会存在出帧慢的情况，就是不算启动时间的时间消耗，从0.000开始以33ms的间距把采集的第一帧，一帧一帧推出去，所以这就不会是现实同步的画面，而且真实fps并非完美的30fps，是29.xxxfps，导致帧数之间会存在抖动时间，所以会导致缓冲时间越来越长。意味着项目里面PTS其实就是指帧序号与现实时间是脱钩的。



## 解决方式：

这意味着我们需要考虑把PTS跟现实时间关联上，意味着pts不应该死板，应该基于时间可变，所以我们应该记录的是这一帧从推流开始到现在过去了多久，我们就需要一个变量不断记录这个时间。

效果应该如下：

| 情况       | pts_base_ | pts_now   | pts_ms |
| :--------- | :-------- | :-------- | :----- |
| 第一帧     | T0        | T0        | 0      |
| 约 33ms 后 | T0        | T0+33ms   | 33     |
| 约 1 秒后  | T0        | T0+1000ms | 1000   |

同时考虑边界情况，我们也需要避免一个如果两帧几乎在同一个毫秒内完成出帧的话，可能存在上一帧比下一帧还要晚的情况。需要保证让下一个pts严格递增。

这样我们就实现了与真实时间对齐的要求。

```c++
        auto pts_now = std::chrono::steady_clock::now();//获得当前时间。
        if (!pts_base_valid_) {
            pts_base_ = pts_now;
            pts_base_valid_ = true;
        }//获得第一帧的时间
        int64_t pts_ms = std::chrono::duration_cast<std::chrono::milliseconds>(
                             pts_now - pts_base_).count();//获得第一帧到现在的时间间隔。
        if (pts_ms <= last_pts_ms_) {//判断有没有存在时间“回溯”的情况，纠正
            pts_ms = last_pts_ms_ + 1; // 保证严格递增，避免 dts 非单调报错
        }
        last_pts_ms_ = pts_ms;//存一下上一帧对比时间
        current_pts_ms_ = pts_ms;//当前的时间戳PTS，可以用于传参进pkt。
```





## 结果：

可以发现测试15min后，延迟明显没有增大。问题暂时解决。