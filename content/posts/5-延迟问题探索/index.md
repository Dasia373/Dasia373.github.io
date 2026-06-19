---
date: 2026-06-11T11:00:00+08:00
draft: false
title: "阶段5 — 延迟问题探索（P0–P3 优化）"
tags: ["RK3588","RTSP","FFmpeg","延迟优化"]
categories: ["嵌入式Linux"]
summary: "从播放器缓冲、muxdelay、GOP、resize 多个维度探索延迟优化，综合延迟从约 1s 压到约 500ms。"
---

# 1.尝试使用低缓冲的ffplay（效果明显）

```cmd
ffplay -rtsp_transport tcp -i rtsp://192.168.137.1:8554/mystream
```

正常情况下，默认缓存下的播放缓冲有接近800~1200ms左右的延迟。

```cmd
ffplay -fflags nobuffer -flags low_delay -probesize 32 -analyzeduration 0 -rtsp_transport tcp -i rtsp://192.168.137.1:8554/mystream #低缓冲
```

使用低缓冲后，默认缓存下播放端有接近500~600ms的延迟表现。

![播放端低缓存](/posts/5-延迟问题探索/assets/IMG_20260611_111820.jpg)

# 2.降低muxdelay等待时间（不明显，这个项目里面应该不是瓶颈）

[FFmpeg Formats Documentation](https://ffmpeg.org/ffmpeg-formats.html)

```
Muxers are configured elements in FFmpeg which allow writing multimedia streams to a particular type of file.

When you configure your FFmpeg build, all the supported muxers are enabled by default. You can list all available muxers using the configure option --list-muxers.

You can disable all the muxers with the configure option --disable-muxers and selectively enable / disable single muxers with the options --enable-muxer=MUXER / --disable-muxer=MUXER.

The option -muxers of the ff* tools will display the list of enabled muxers. Use -formats to view a combined list of enabled demuxers and muxers.

A description of some of the currently available muxers follows.
```

Muxer可以在多路视频流和视频音频流下通过等待实现同步，那我这边有个望文生义的思考就是，Muxer的MuxDelay是否存在阻塞行为呢？然后我从AI中询问了下：Muxdelay属于是在Muxer内部最多攒包多久，这种就不属于那种非阻塞IO和Sleep这两种情况了。项目里面是调用av_interleaved_write_frame()后，包就进入Muxer中，在interleave队列中，然后由于当前项目只有一路流，就很快能处理好，几乎是可以不用等的。所以我们可以把delay改为0。

```c++
av_dict_set(&options, "muxdelay", "0", 0);//0.1→0：单路流不需要 100ms muxer 缓冲，直接发包降延迟
av_dict_set(&options, "muxpreload", "0", 0);//同步去掉 muxer 预加载缓冲
```

坏处：因为现在只有一路H.264视频流，如果后面多路视频流的话可能要再调整，不过0.1到0的修改其实差异也不大（==测试后没有很明显变化==）。

![改MuxDelay测试](/posts/5-延迟问题探索/assets/IMG_20260611_131843..jpg)

# 3.修改GOP从fps=30到15，（不明显，应该也不是瓶颈）

图片组的知识我直接一个记不清了，看眼百科[图片组 - 维基百科 --- Group of pictures - Wikipedia](https://en.wikipedia.org/wiki/Group_of_pictures)

![GOP](/posts/5-延迟问题探索/assets/GOP_2.svg)

> 在[MPEG](https://zh.wikipedia.org/wiki/MPEG) [视讯编码](https://zh.wikipedia.org/wiki/視訊編碼)中，**图像群组**（**G**roup **o**f **p**ictures，GOP）即I画格和I画格之间的画格排列。
>
> 图像群组就是一组以MPEG编码的影片或视讯串流内部的连续图像。每一个以MPEG编码的影片或视讯串流都由连续的图像群组组成。
>
> 图像群组可包含下列图像类型：
>
> - I-图像/画格（节点编码图像，intra coded picture）参考图像，相当于一个固定影像，且独立于其它的图像类型。每个图像群组由此类型的图像开始。
> - P-图像/画格（预测编码图像，predictive coded picture）包含来自先前的I或P-画格的差异资讯。
> - B-图像/画格（前后预测编码图像，bidirectionally predictive coded pictures）包含来自先前和/或之后的I或P-画格的差异资讯。
> - D-图像/画格（指示编码图像，DC direct coded picture）用于快速进带。
>
> 图像群组总是以I画格为起始点。后面有若干P画格。其它的则是B画格。下一个I画格即为新的图像群组的起始点。以上图为例，显示的顺序是：I0、B1、B2、B3、P4、B5、B6、B7、P8、B9、B10、B11、I12；则编解码顺序为：I0、P4、B1、B2、B3、P8、B5、B6、B7、I12、B9、B10、B11。

一般情况下解码是从从第一个I帧解码出来开始才能出画面，那么可以通过缩短GOP间距，让I帧变多，就是让这个解码到画面最大时间差变小，反正I帧也变密集了，尽管可能传输过程中出现数据包损坏导致花屏也可以更快透过下一个I帧解码到，代价就是由于I帧比P帧变密，且I帧本身比P帧要大而且恒定码率下，画质要略微下降，码率压力由于全去解码I帧了，压力会有点大。

![改GOP](/posts/5-延迟问题探索/assets/IMG_20260611_132300..jpg)

# 4.每次取帧排空队列，只保留最新帧（一堆段错误，代码没写出来，操）

readImx415Frame现在取得的DQBUF的帧数相对V4L2传回的buffer里面的帧而言要旧，可以不断把DQBUFF把队列里面所有填充完的帧数取出来只去取最新的帧，每次到最新的帧旧QBUF还回去，然后把那一帧做cvtColor，每次只采集最新的帧。



没写出来，空悲切。

思路，等后面哪天填坑：

```yacas
latest = 无

loop:
    ret = DQBUF(&buf)
    if ret == EAGAIN:   // 没有更多就绪帧了
        break
    if 之前有 latest:
        QBUF(上一帧)     // 旧帧不要了，直接还回去
    latest = buf

if latest 不存在:
    return false

cvtColor(latest 对应的 mmap 内存)
QBUF(latest)
return true
```

ISP--了解

MIPI--熟悉

智驾，卓yu，机器人





