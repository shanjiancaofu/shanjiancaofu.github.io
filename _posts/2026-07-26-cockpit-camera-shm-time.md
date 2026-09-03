---
title: 智能座舱开发（二）：相机共享内存与时间戳合同
date: 2026-07-26 20:30:00 +0800
categories: [嵌入式]
tags: [智能座舱, 相机, 共享内存, 时间戳, V4L2]
---

相机预览最容易出现两类隐蔽问题：为了跨进程传图像产生大量复制，以及把不同来源的时间戳当成同一种时间。智能座舱项目用共享内存传输最新帧，并为 source time、receive time 和 timeout time 分别规定用途。

## 为什么不把图像放进 gRPC

一帧 1920×1080 的 BGRx 图像约为：

```text
1920 × 1080 × 4 ≈ 7.9 MiB
```

30 FPS 时，仅原始数据就接近 237 MiB/s。如果每帧再经过 protobuf 序列化和多次复制，会挤占本来应该留给语音、UI 和本地模型的 CPU 与内存带宽。

因此相机 Driver 将像素写入 POSIX shared memory，控制接口只提供共享内存名称、格式、尺寸、generation 和健康状态。

## 双缓冲如何避免读到半帧

共享区域准备两个数据槽。Writer 总是在非活动槽完成整帧写入，最后再发布新的 generation 和活动槽编号。Reader 先读取元数据，复制对应槽的数据，再确认 generation 没有变化。

```text
时刻 A：slot 0 对外可读，writer 写 slot 1
时刻 B：slot 1 写完，原子发布 generation=N+1
时刻 C：slot 1 对外可读，writer 可复用 slot 0
```

这个设计不保存历史队列，慢 Reader 可能跳过中间帧，但总能追上最新画面。对车机预览来说，显示当前帧比积压旧帧更重要。

## generation 不只是帧序号

generation 可以帮助 Reader 判断数据是否更新，也可用于检测读期间发生的槽切换。实现时还需要定义进程重启、共享内存重新创建和序号回绕的行为，不能只把它当作一个普通计数器。

元数据中还应包含像素格式和步长。仅传宽高不够，因为 BGRx、RGB、NV12 等格式的每像素字节数和布局不同。

## 四种时间不能混用

系统为时间建立了明确分类：

| 时间 | 主要用途 |
|---|---|
| source/sample time | 传感器采样、跨传感器匹配、ROS `header.stamp` |
| received time | 接收记录、设备时钟映射锚点、日志关联 |
| steady time | watchdog、freshness、deadline、耗时 |
| wall/log time | 人工阅读的日志和文件时间线 |

超时必须使用单调时钟。系统时间被 NTP 校准或人工修改时，`steady_clock` 不会突然跳变，因此不会把一个设备误判为刚恢复或超时很久。

## source timestamp 必须带有效性

相机帧只有在 `source_timestamp_valid=true` 时才能使用 source timestamp。GStreamer 时间需要结合 pipeline clock 与 base time 映射到统一时钟域；V4L2 时间戳则要保留驱动给出的 clock domain 和 flags。

不能简单地在 UI 收到帧时调用 `now()` 并称其为采样时间。这样得到的是接收或处理时间，队列延迟会被悄悄算进传感器时间。

## 跨传感器匹配应 fail closed

相机与 LiDAR 数据做投影时，必须比较同一时间域中的 sample timestamp，并设置明确的最大时间差。超过阈值就拒绝匹配，不能因为两份数据“差不多同时收到”而放宽条件。

标定参数也要验证：旋转矩阵应接近正交且行列式约为 +1；位于相机后方、图像外或包含 NaN/Inf 的点均应拒绝。

共享内存解决的是高带宽传输问题，时间戳合同解决的是数据含义问题。两者结合，预览链路才不仅是“画面能出来”，而是具备可测量、可同步和可诊断的基础。
