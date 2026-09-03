---
title: Software ISP 学习笔记：从 IMX219 RAW10 到 BGRx 图像
date: 2026-09-02 20:30:00 +0800
categories: [嵌入式]
tags: [ISP, IMX219, RAW10, V4L2, NEON, Jetson]
---

摄像头传感器输出的 RAW Bayer 数据还不是可以直接显示的彩色图像。它需要经过黑电平校正、白平衡、去马赛克和 Gamma 等处理，才能转换为 RGB 或 BGR 图像。这条处理链通常由硬件 ISP 完成，但自己实现一条 Software ISP，可以更直观地理解每个阶段的输入、输出和性能代价。

本文结合 Jetson Orin Nano、IMX219 和 Direct V4L2 实现一条最小 Software ISP：从 RG10/MMAP 采集开始，经过 RAW10 unpack、BLC、白平衡、demosaic 和 Gamma，最终输出 BGRx 图像。

## 一、什么是 ISP

ISP 是 Image Signal Processor，即图像信号处理器。传感器像素记录的是光强，RAW 数据通常具有以下特点：

- 每个像素只记录一个颜色分量；
- 存在黑电平偏置；
- 不同颜色通道的响应强度不同；
- 亮度仍接近线性空间，不适合直接显示；
- 可能包含坏点、噪声、镜头阴影等问题。

完整的商用 ISP 往往包含很多阶段：

```text
RAW 输入
  → 黑电平校正 BLC
  → 坏点校正 DPC
  → 镜头阴影校正 LSC
  → 降噪
  → 白平衡 WB
  → 去马赛克 Demosaic
  → 色彩校正 CCM
  → Gamma / Tone Mapping
  → 锐化
  → RGB / YUV 输出
```

当前学习实现只保留最小可显示链路，并不等同于 Jetson 硬件 ISP 的完整图像质量。

## 二、三条相机路径的边界

项目把相机输入分成三条固定路径：

| Pipeline | 输入 | ISP 所在位置 | 用途 |
|---|---|---|---|
| `argus_isp` | Jetson CSI | NVIDIA Camera Stack 与硬件 ISP | 生产预览、低 CPU 占用 |
| `uvc` | USB MJPEG/YUYV | 摄像头内部或解码链路 | 普通 USB 摄像头 |
| `software_isp` | IMX219 RG10 | 用户态 C++ | RAW 学习、调试和算法实验 |

USB 摄像头通常已经在设备内部完成 Bayer 处理，输出 MJPEG 或 YUYV，不应该再送入 Software ISP。`software_isp` 只接受 `tegra-video + RG10`，`argus_isp` 只接受 Argus URI。明确限制组合，可以避免把错误格式处理成“看起来有颜色”的乱码图像。

Argus 和 Direct V4L2 可以访问同一颗 IMX219，但不能同时占用同一个 sensor session。

## 三、Direct V4L2 获取 RAW10

Software ISP 绕过 Jetson Argus 硬件处理，通过 V4L2 直接获取 Bayer 数据：

```text
IMX219
  → tegra-video
  → /dev/video0
  → VIDIOC_REQBUFS / MMAP
  → poll
  → VIDIOC_DQBUF
  → RawBayerFrame
  → VIDIOC_QBUF
```

Jetson 使用该路径前，需要确认驱动处于允许 Bayer bypass 的模式，并验证设备实际支持 `SRGGB10/RG10`。设备号、分辨率、stride 和 `bytesused` 都应以驱动协商结果为准。

```bash
v4l2-ctl --device /dev/video0 --all
v4l2-ctl --device /dev/video0 --list-formats-ext
```

不能只根据文件名判断设备。生产代码还应检查 driver 名称是否为 `tegra-video`、总线信息是否属于 CSI，以及当前 fourcc 是否与 Software ISP 匹配。

## 四、RAW10 的内存布局

“RAW10”只说明有效位深是 10 位，不代表内存一定按同一种方式排列。常见的 MIPI packed RAW10 使用 5 字节保存 4 个像素；当前 Jetson V4L2 协商得到的 RG10 则以 16 位容器保存每个像素，有效 10 位位于高位。

项目中的标量解包为：

```cpp
std::uint16_t container =
    static_cast<std::uint16_t>(source[offset]) |
    static_cast<std::uint16_t>(source[offset + 1]) << 8;

destination[x] = container >> 6;
```

因此一行最少需要 `width × 2` 字节。处理时仍要使用驱动返回的 `bytes_per_line` 跳到下一行，因为 stride 可能大于有效像素宽度。

如果把 16 位容器错误地按 5-byte packed RAW10 解析，画面会出现周期性条纹；反过来也一样。实现前必须通过 fourcc、stride 和少量原始字节确认实际布局。

## 五、使用 NEON 加速 RAW 解包

标量版本一次处理一个像素。ARM64 NEON 版本一次加载 8 个 `uint16_t`，同时右移 6 位：

```cpp
const auto samples =
    vld1q_u16(reinterpret_cast<const std::uint16_t *>(source) + x);
vst1q_u16(destination + x, vshrq_n_u16(samples, 6));
```

循环尾部不足 8 个像素的部分仍由标量代码处理。构建支持 NEON 时默认选择向量化函数，也可以通过环境变量临时关闭，以便进行 A/B 测试：

```bash
export COCKPIT_CAMERA_DISABLE_NEON=1
```

NEON 优化前要先保证标量实现正确，并用相同 RAW fixture 比较两者输出。SIMD 版本更快，但不能改变任意像素值。

## 六、黑电平校正与归一化

传感器即使完全遮光，RAW 值也不一定为 0。这个偏置称为 black level。当前配置的黑电平为 64，10 位最大值为 1023。

转换到 8 位可以写成：

```text
output = clamp((raw - black_level) × 255 / (1023 - black_level), 0, 255)
```

代码使用 OpenCV `convertTo` 同时完成缩放和偏移：

```cpp
const double scale = 255.0 / (1023.0 - black_level);
const double offset = -black_level * scale;
raw16.convertTo(raw8, CV_8UC1, scale, offset);
```

black level 设置太小，暗部会发灰；设置太大，暗部细节会被截断。固定值适合最小基线，进一步完善时应结合传感器模式、曝光、温度和暗场图进行标定。

## 七、Bayer Pattern 与去马赛克

IMX219 当前输入按 RGGB Bayer 排列：

```text
R G R G ...
G B G B ...
R G R G ...
G B G B ...
```

每个位置只有一个颜色值，demosaic 根据相邻像素估算缺失的两个颜色通道：

```cpp
cv::cvtColor(raw8, bgr, cv::COLOR_BayerRG2BGR);
```

Bayer pattern 选错时，最常见现象是红蓝通道交换或颜色出现网格。不同库对 pattern 名称的描述角度可能不同，不能只看枚举名字下结论。应使用色卡、纯色目标或与可信硬件 ISP 输出对比，确认最终通道语义。

当前实现只接受 RGGB，收到其他 pattern 时直接失败，不尝试自动猜测。

## 八、白平衡与 Gamma LUT

不同颜色通道对光源和传感器的响应不同，需要分别乘以 gain。当前基线参数为：

```text
red_gain   = 1.8
green_gain = 1.0
blue_gain  = 1.5
gamma      = 2.2
```

为了避免对每个像素反复执行浮点幂运算，初始化时为每个通道生成 256 项 LUT：

```text
normalized = clamp(value / 255 × gain, 0, 1)
output     = pow(normalized, 1 / gamma) × 255
```

运行时拆分 BGR 通道，分别执行 `cv::LUT`，最后再合并。这一实现简单且速度稳定，但它把 gain 和 Gamma 合并在 8 位域中，容易在高亮区域提前饱和。更高质量的 ISP 应尽量在高位深线性域完成白平衡与色彩校正，再在靠后的阶段压缩到 8 位。

## 九、统一输出为 BGRx

完成颜色处理后，ISP 将三通道 BGR 转成四通道 BGRx：

```cpp
output.format = CameraPixelFormat::kBgrx;
output.stride_bytes = width * 4;

cv::cvtColor(bgr, output_bgra, cv::COLOR_BGR2BGRA);
```

固定输出格式可以简化共享内存和 Qt UI 消费端。对外公开的是已经处理的 `CameraFrame`，底层 RG10 只保留在诊断工具和 ISP 内部。

帧的 sequence、source timestamp、receive timestamp 和时钟域信息也必须沿流水线保留。像素格式转换不能顺便把采样时间替换为处理完成时间。

## 十、采集与 ISP 双线程

V4L2 采集和 Software ISP 在两个线程中运行，中间使用容量为 2 的有界队列：

```text
Capture Thread                       ISP Thread
poll / DQBUF                         pop newest RAW
复制或接管 RAW buffer   → queue →   SoftwareIsp::Process
QBUF                                 输出 BGRx
```

队列满时丢弃最旧帧，并增加 `dropped_queue_frames`。实时预览不能使用无界队列，否则 ISP 偶尔变慢后，UI 会持续显示历史画面。

同时记录 V4L2 sequence。如果当前 sequence 比上一帧大 1 以上，就增加 `source_sequence_gaps`。两项指标含义不同：

- queue drop 表示应用内部来不及处理；
- source sequence gap 表示采集源或驱动层已经出现缺口。

区分两者才能定位瓶颈发生在采集前还是 ISP 队列中。

## 十一、预分配减少抖动

ISP 每帧需要 RAW16、RAW8、BGR、通道数组和 BGRx 输出。实现将 `cv::Mat` 和通道容器保存为成员，并使用 `create()` 复用尺寸相同的内存。

如果在 30 FPS 循环中频繁分配和释放数 MiB buffer，不仅增加 CPU 开销，还会产生延迟抖动和内存碎片。实时链路不能只看平均耗时，P95 和最大值同样重要。

## 十二、分阶段性能测量

每帧分别记录：

```text
raw_unpack
normalize
demosaic
gain_gamma
output
total
```

端到端还记录 RAW 入队到 BGRx 输出的 P50、P95 和最大延迟。这样发现总耗时升高时，可以判断是 unpack、OpenCV demosaic、颜色 LUT，还是采集队列等待造成的。

在同一台 Jetson Orin Nano 8GB、同一颗 IMX219、1920×1080@30、300 帧测试中得到：

| 路径 | 平均处理耗时 | P95 | 进程 CPU | 丢帧/sequence gap |
|---|---:|---:|---:|---:|
| CPU scalar ISP | 23.108 ms | 24.188 ms | 116.993% | 0 / 0 |
| CPU + NEON ISP | 16.007 ms | 16.858 ms | 110.467% | 0 / 0 |
| CUDA prototype | 6.991 ms | 8.909 ms | 38.402% | 不适用 |
| Argus Hardware ISP | 未公开 | 未公开 | 28.284% | 未公开 |

NEON 相比 scalar 的平均处理耗时降低约 30.7%，两条 live V4L2 路径都维持约 30 FPS。

## 十三、为什么 CUDA 更快却没有成为默认路径

CUDA prototype 对同一张离线 RAW 图重复执行 H2D、kernel 和 D2H，平均约 6.991 ms。这证明计算有加速潜力，但还不能说明 live camera pipeline 已经完成。

当前仍存在三个边界：

1. CUDA 测试没有接入实时 V4L2 队列，因此没有 queue drop 和 sequence gap 数据；
2. 每帧仍包含 Host-to-Device 与 Device-to-Host 复制；
3. CUDA 输出与 CPU/OpenCV demosaic 的红蓝通道质量尚未完全对齐。

图像算法不仅要快，还要输出正确。质量对齐、DMABUF/zero-copy 评估和长时间稳定性完成之前，CUDA 只能保留为性能实验。

## 十四、Argus 数据应该如何解读

Argus 使用 Jetson 硬件 ISP，适合与 ASR、TTS、LLM、UI 和 ROS2 长期并发。短测中应用层统计约 32 FPS，但 sensor mode 目标仍为 30 FPS，不能把短时计时误写成相机稳定超频。

Argus 当前没有提供与 Software ISP 相同的 source-to-output 分阶段时间，因此不能拿应用帧间隔冒充 ISP latency。`tegrastats` 的 `GR3D_FREQ` 也不是独立硬件 ISP 利用率。

Argus 的进程 RSS 包含 GStreamer、Argus runtime 和内部 buffer pool，同样不能解释成 ISP 算法本身占用的内存。

## 十五、还可以继续完善什么

当前最小 ISP 能输出稳定画面，但图像质量仍有较大改进空间：

- 使用暗场数据标定 black level；
- 根据灰卡或统计方法实现自动白平衡；
- 在高位深线性域执行 WB 和 CCM；
- 增加坏点校正、降噪和镜头阴影校正；
- 比较不同 demosaic 算法的边缘和彩色伪影；
- 建立色卡、PSNR、SSIM 和主观图像质量测试；
- 为不同曝光、增益和 sensor mode 保存独立配置；
- 评估 DMABUF、CUDA zero-copy 和硬件编解码链路。

## 总结

这条 Software ISP 的完整数据流为：

```text
IMX219 RG10
  → V4L2 MMAP
  → 16-bit container RAW10 unpack
  → black level normalize
  → RGGB demosaic
  → per-channel gain + Gamma LUT
  → BGRx CameraFrame
  → shared memory / Qt UI
```

自己实现 ISP 后，RAW 格式、stride、颜色通道、时钟和队列背压不再是抽象名词。硬件 ISP 仍是生产预览的合理选择，Software ISP 则提供了一条可观察、可修改和可测量的学习与诊断路径。
