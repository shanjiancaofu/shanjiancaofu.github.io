---
title: V4L2 驱动框架：五个核心结构体与 UVC 注册流程
date: 2026-04-22 20:30:00 +0800
categories: [Linux]
tags: [Linux, V4L2, UVC, videobuf2, 摄像头驱动]
---

学习 Linux 摄像头驱动时，经常会同时遇到 `v4l2_device`、`v4l2_subdev`、`video_device`、`v4l2_fh` 和 `videobuf2`。这些名称看起来相似，实际分别解决设备聚合、硬件子模块抽象、字符设备节点、单次打开状态和视频缓冲区管理问题。

本文先梳理五个核心对象，再结合 UVC 驱动的 `probe` 路径说明一个 USB 摄像头如何最终出现在 `/dev/videoX`。

![V4L2 驱动核心对象关系](/assets/img/posts/v4l2-driver-architecture.svg)

## 先修正一个容易混淆的说法

这五部分可以看作典型视频采集或输出驱动的核心拼图，但不是每一个 V4L2 驱动都必须同时拥有它们。

- SoC camera bridge 或视频采集驱动通常会使用 `v4l2_device`、`video_device` 和 `vb2_queue`；
- sensor、decoder、CSI receiver 等子设备驱动通常内嵌 `v4l2_subdev`；
- 纯 subdev 驱动不一定创建 `/dev/videoX`；
- 不传输流式缓冲的 radio 或控制类节点不一定使用 videobuf2；
- 一个整体驱动也可能同时管理多个 `video_device` 和多个 `v4l2_subdev`。

因此应该根据驱动在媒体拓扑中的职责选择对象，而不是为了“结构完整”机械地创建全部五种结构体。

## 一、`struct v4l2_device`：V4L2 设备实例

`v4l2_device` 表示一个 V4L2 设备实例，用于聚合和管理相关的子设备，并为整个驱动提供统一名称、通知和控制入口。

它常被内嵌在驱动私有结构中：

```c
struct my_camera_dev {
    struct device *dev;
    struct v4l2_device v4l2_dev;
    struct video_device video_dev;
    struct vb2_queue queue;
    struct mutex lock;
};
```

注册方法：

```c
ret = v4l2_device_register(parent_dev, &cam->v4l2_dev);
if (ret)
    return ret;
```

注销时调用：

```c
v4l2_device_unregister(&cam->v4l2_dev);
```

`v4l2_device` 本身通常不会创建 `/dev/videoX`。它更像 V4L2 框架中的中间协调层：向下管理 `v4l2_subdev`，向上被一个或多个 `video_device` 引用。

如果存在 media controller，驱动还可能同时维护 `media_device`，用于描述 entity、pad 和 link 拓扑。`media_device` 与 `v4l2_device` 相关，但两者不是同一个对象。

## 二、`struct v4l2_subdev`：硬件子设备抽象

一条摄像头链路可能由多个硬件模块组成：

```text
Sensor → MIPI CSI-2 Receiver → ISP → DMA → Memory
```

Sensor、decoder、bridge、CSI receiver 和 ISP 等模块通常使用 `v4l2_subdev` 表示。每个子设备驱动应拥有一个 `v4l2_subdev`，一般将它嵌入自己的私有结构：

```c
struct my_sensor {
    struct i2c_client *client;
    struct v4l2_subdev sd;
    struct media_pad pad;
    struct v4l2_ctrl_handler ctrl_handler;
    struct mutex lock;
};
```

初始化时绑定 operations：

```c
v4l2_subdev_init(&sensor->sd, &sensor_subdev_ops);
sensor->sd.owner = THIS_MODULE;
strscpy(sensor->sd.name, "my-sensor", sizeof(sensor->sd.name));
```

I2C sensor 更常使用辅助函数：

```c
v4l2_i2c_subdev_init(&sensor->sd, client, &sensor_subdev_ops);
```

subdev operations 按职责分组，例如：

```c
static const struct v4l2_subdev_core_ops sensor_core_ops = {
    .s_power = sensor_s_power,
};

static const struct v4l2_subdev_video_ops sensor_video_ops = {
    .s_stream = sensor_s_stream,
};

static const struct v4l2_subdev_pad_ops sensor_pad_ops = {
    .enum_mbus_code = sensor_enum_mbus_code,
    .get_fmt = sensor_get_fmt,
    .set_fmt = sensor_set_fmt,
};

static const struct v4l2_subdev_ops sensor_subdev_ops = {
    .core = &sensor_core_ops,
    .video = &sensor_video_ops,
    .pad = &sensor_pad_ops,
};
```

`pad_ops` 描述的是子设备之间的 media bus 格式，例如 Bayer RAW10 的 bus code；它不等同于用户空间 `/dev/videoX` 看到的 memory pixel format。

### subdev 如何加入父设备

同步注册可以调用：

```c
ret = v4l2_device_register_subdev(&cam->v4l2_dev, &sensor->sd);
```

真实 SoC 摄像头中，sensor 可能通过设备树描述，bridge 与 sensor 的 probe 顺序也不确定。这时通常使用 V4L2 async notifier，在所有远端 subdev 就绪后完成拓扑绑定。

subdev 默认不一定暴露字符设备节点。如果驱动需要用户空间直接访问 subdev API，可以注册 `/dev/v4l-subdevX`，但是否允许以及哪些操作可写必须由驱动明确控制。

## 三、`struct video_device`：创建 `/dev/videoX`

`video_device` 是 V4L2 视频节点在内核中的表示。调用 `video_register_device()` 后，V4L2 core 才会分配 minor 并创建类似 `/dev/video0` 的节点。

典型字段包括：

```c
strscpy(cam->video_dev.name, "my-camera", sizeof(cam->video_dev.name));
cam->video_dev.v4l2_dev = &cam->v4l2_dev;
cam->video_dev.fops = &camera_fops;
cam->video_dev.ioctl_ops = &camera_ioctl_ops;
cam->video_dev.lock = &cam->lock;
cam->video_dev.queue = &cam->queue;
cam->video_dev.release = video_device_release_empty;
cam->video_dev.vfl_dir = VFL_DIR_RX;

video_set_drvdata(&cam->video_dev, cam);

ret = video_register_device(&cam->video_dev, VFL_TYPE_VIDEO, -1);
```

几个字段的职责：

| 字段 | 作用 |
|---|---|
| `v4l2_dev` | 关联所属的 V4L2 设备实例 |
| `fops` | `open`、`release`、`poll`、`mmap` 等文件操作 |
| `ioctl_ops` | `VIDIOC_QUERYCAP`、格式协商、缓冲区等 ioctl |
| `queue` | 可选的 `vb2_queue`，供 V4L2 ioctl helper 使用 |
| `lock` | ioctl 序列化锁，可按驱动需求设置 |
| `release` | 最后一个引用释放时的回调，必须与分配方式匹配 |
| `vfl_dir` | capture 使用 `VFL_DIR_RX`，output 使用 `VFL_DIR_TX` |

动态调用 `video_device_alloc()` 分配的对象通常配合 `video_device_release()`。如果 `video_device` 内嵌在更大的私有对象中，则不能让默认 release 单独释放这块内存，常使用 `video_device_release_empty()`，再由外层对象生命周期负责回收。

注销节点：

```c
video_unregister_device(&cam->video_dev);
```

设备断开时，即使节点被注销，已经打开的文件仍可能持有引用。因此 remove/disconnect 不能立即释放所有私有数据，必须正确处理引用计数和正在进行的 I/O。

## 四、`struct v4l2_fh`：每一次 open 的状态

`video_device` 描述整个节点，而 `v4l2_fh` 描述某一次 `open()`。同一个 `/dev/video0` 被两个进程打开时，会拥有两个独立 file handle。

驱动通常把它嵌入自己的 per-file context：

```c
struct camera_fh {
    struct v4l2_fh fh;
    struct my_camera_dev *cam;
    bool is_owner;
};
```

open 路径：

```c
static int camera_open(struct file *file)
{
    struct video_device *vdev = video_devdata(file);
    struct my_camera_dev *cam = video_get_drvdata(vdev);
    struct camera_fh *ctx;

    ctx = kzalloc(sizeof(*ctx), GFP_KERNEL);
    if (!ctx)
        return -ENOMEM;

    ctx->cam = cam;
    v4l2_fh_init(&ctx->fh, vdev);
    file->private_data = &ctx->fh;
    v4l2_fh_add(&ctx->fh);
    return 0;
}
```

release 路径的顺序与之相反：

```c
static int camera_release(struct file *file)
{
    struct v4l2_fh *fh = file->private_data;
    struct camera_fh *ctx = container_of(fh, struct camera_fh, fh);

    v4l2_fh_del(fh);
    v4l2_fh_exit(fh);
    kfree(ctx);
    return 0;
}
```

简单驱动可以使用 `v4l2_fh_open()`、`v4l2_fh_release()` 等 helper。使用 vb2 时，也常直接组合 `vb2_fop_release()`。

`v4l2_fh` 还为 event subscription 和 priority 管理提供基础设施。它不是 Linux `struct file` 的替代，而是挂在 `file->private_data` 后的 V4L2 上下文。

## 五、videobuf2：流式视频缓冲区

老代码可能使用 videobuf，新驱动通常使用 videobuf2，简称 vb2。它统一处理用户空间缓冲区申请、排队、mmap/userptr/dmabuf 内存模型以及 stream on/off 状态。

驱动私有结构中通常包含：

```c
struct vb2_queue queue;
```

初始化示例：

```c
cam->queue.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
cam->queue.io_modes = VB2_MMAP | VB2_DMABUF;
cam->queue.drv_priv = cam;
cam->queue.buf_struct_size = sizeof(struct camera_buffer);
cam->queue.ops = &camera_vb2_ops;
cam->queue.mem_ops = &vb2_dma_contig_memops;
cam->queue.timestamp_flags = V4L2_BUF_FLAG_TIMESTAMP_MONOTONIC;
cam->queue.lock = &cam->lock;
cam->queue.dev = cam->dev;

ret = vb2_queue_init(&cam->queue);
```

不同硬件可选择不同 memory allocator，例如 `vb2_vmalloc_memops`、`vb2_dma_contig_memops` 或 `vb2_dma_sg_memops`。选择取决于设备 DMA 能力和平台内存结构，不能随意互换。

### 核心 `vb2_ops`

```c
static const struct vb2_ops camera_vb2_ops = {
    .queue_setup = camera_queue_setup,
    .buf_prepare = camera_buf_prepare,
    .buf_queue = camera_buf_queue,
    .start_streaming = camera_start_streaming,
    .stop_streaming = camera_stop_streaming,
    .wait_prepare = vb2_ops_wait_prepare,
    .wait_finish = vb2_ops_wait_finish,
};
```

- `queue_setup`：决定 plane 数量、每个 plane 大小和最少 buffer 数；
- `buf_prepare`：检查 buffer 是否足够大并设置 payload；
- `buf_queue`：把 buffer 加入驱动等待 DMA 的队列；
- `start_streaming`：启动硬件和 DMA；
- `stop_streaming`：停止硬件，并归还所有未完成 buffer。

### buffer 生命周期

用户空间常见调用顺序：

```text
VIDIOC_REQBUFS
  → VIDIOC_QUERYBUF
  → mmap
  → VIDIOC_QBUF × N
  → VIDIOC_STREAMON
  → poll
  → VIDIOC_DQBUF
  → 处理图像
  → VIDIOC_QBUF
  → VIDIOC_STREAMOFF
```

驱动侧对应：

```text
用户 QBUF
  → vb2 校验状态
  → driver .buf_queue
  → 硬件 DMA 写入
  → vb2_buffer_done(VB2_BUF_STATE_DONE)
  → 用户 DQBUF
```

`stop_streaming` 或错误路径必须把仍由驱动持有的 buffer 以 `DONE` 或 `ERROR` 状态归还。遗漏一个 buffer 就可能导致 `STREAMOFF` 卡住或用户空间永远等待。

## 六、五个对象如何连接

可以把它们理解成以下层次：

```text
设备实例：struct v4l2_device
├── 子设备：struct v4l2_subdev × N
└── 视频节点：struct video_device × N
    ├── 每次 open：struct v4l2_fh
    └── 流式缓冲：struct vb2_queue
```

但这不是强制的内存包含关系。例如 `vb2_queue` 可以放在驱动私有结构中，只由 `video_device->queue` 指向；`v4l2_fh` 则通常按每次 open 动态分配。

从用户空间看，只能直接操作注册出来的设备节点：

```text
/dev/videoX      视频采集或输出
/dev/vbiX        VBI 数据
/dev/radioX      Radio 设备
/dev/v4l-subdevX 可选的 subdev 用户接口
/dev/mediaX      Media Controller 拓扑
```

## 七、一个最小 V4L2 capture 驱动的注册顺序

典型 probe 可以按以下顺序组织：

```text
1. 分配驱动私有结构
2. 获取 clock / regulator / GPIO / IRQ / DMA 等资源
3. 初始化 mutex、spinlock、list 和硬件状态
4. 注册 v4l2_device
5. 发现并绑定 v4l2_subdev
6. 初始化 vb2_queue
7. 填充 video_device 的 fops / ioctl_ops / queue / release
8. video_set_drvdata
9. video_register_device 创建 /dev/videoX
10. 保存 bus drvdata，允许运行时访问
```

伪代码：

```c
static int camera_probe(struct platform_device *pdev)
{
    struct my_camera_dev *cam;
    int ret;

    cam = devm_kzalloc(&pdev->dev, sizeof(*cam), GFP_KERNEL);
    if (!cam)
        return -ENOMEM;

    cam->dev = &pdev->dev;
    mutex_init(&cam->lock);
    spin_lock_init(&cam->qlock);

    ret = v4l2_device_register(&pdev->dev, &cam->v4l2_dev);
    if (ret)
        return ret;

    ret = camera_init_vb2(cam);
    if (ret)
        goto err_v4l2;

    camera_init_video_device(cam);
    ret = video_register_device(&cam->video_dev, VFL_TYPE_VIDEO, -1);
    if (ret)
        goto err_v4l2;

    platform_set_drvdata(pdev, cam);
    return 0;

err_v4l2:
    v4l2_device_unregister(&cam->v4l2_dev);
    return ret;
}
```

实际 SoC camera 驱动还会涉及 media entity pads、async notifier、runtime PM 和 pipeline validate，不能只靠这段骨架直接驱动硬件。

## 八、UVC 驱动的 `probe` 流程

UVC 是 USB Video Class。符合标准的 USB 摄像头通常由内核 `uvcvideo` 驱动识别，不需要为每一款摄像头重新写一个 V4L2 驱动。

当前上游 Linux 的 `uvc_probe()` 主路径可以概括为：

```text
USB interface 匹配
  → 分配 struct uvc_device
  → 初始化 entities / chains / streams / kref
  → 保存 usb_device、usb_interface、quirks
  → 可选：初始化 media_device
  → uvc_parse_control
  → uvc_gpio_parse
  → v4l2_device_register
  → uvc_scan_device
  → uvc_ctrl_init_device
  → uvc_register_chains
  → 可选：media_device_register
  → usb_set_intfdata
  → uvc_status_init
  → uvc_gpio_init_irq
  → uvc_meta_init
```

不同内核版本的细节可能变化，阅读代码时应以目标设备正在运行的内核源码为准。

### 1. 分配并初始化 `uvc_device`

`uvc_device` 是 UVC 驱动的总私有对象，保存 USB 设备、接口、实体链、视频流、quirks、V4L2 和 media controller 状态。

源码先初始化链表、引用计数和映射计数，再取得 `usb_device` 与 `usb_interface` 的引用。任一步失败，都要沿错误路径释放已经取得的资源。

### 2. `uvc_parse_control()` 解析控制描述符

UVC 设备在 VideoControl interface 中描述 input terminal、processing unit、extension unit、output terminal 等实体。`uvc_parse_control()` 读取这些 class-specific descriptors，建立内核中的 entity 和连接关系。

这里的 “control” 不只是亮度、对比度等按钮，而是理解整个 UVC 功能拓扑的基础。

### 3. 注册 `v4l2_device`

```c
ret = v4l2_device_register(&intf->dev, &dev->vdev);
```

需要注意 UVC 私有结构中的成员名 `dev->vdev` 在这里实际是 `struct v4l2_device`。成员名不能代替类型判断，阅读源码应跳转到结构定义确认。

### 4. 扫描 chain 并初始化 controls

`uvc_scan_device()` 根据解析出的实体连接寻找有效视频 chain。随后 `uvc_ctrl_init_device()` 为摄像头支持的控制项建立映射和状态，例如曝光、增益、亮度或厂商扩展控制。

不是每个 UVC 摄像头都实现相同控制项。用户空间应通过 `VIDIOC_QUERYCTRL`、`VIDIOC_QUERY_EXT_CTRL` 或 `v4l2-ctl --list-ctrls` 枚举，不能假设某个 control 必然存在。

### 5. `uvc_register_chains()` 创建设备节点

每条有效 chain 可能关联一个或多个 streaming interface。UVC 驱动为视频流初始化队列和 `video_device`，最终调用：

```c
video_register_device(vdev, VFL_TYPE_VIDEO, -1);
```

此时才会出现 `/dev/videoX`。因此 `v4l2_device_register()` 成功不等于视频节点已经创建，真正创建设备节点的是 `video_register_device()`。

### 6. `uvc_status_init()` 初始化状态端点

UVC status interrupt endpoint 用于异步通知控制值变化、按钮事件等状态。当前源码中，状态端点初始化失败通常只表示不支持 status interrupt，并不一定让整个摄像头 probe 失败。

这与视频数据使用的 isochronous 或 bulk endpoint 是不同通道。

## 九、UVC 中的 videobuf2

UVC streaming queue 使用 `vb2_queue`，并根据 capture/output/meta 类型选择相应 `vb2_ops`。初始化时会设置：

```text
queue type
I/O modes
driver private pointer
buffer struct size
vb2 operations
memory operations
timestamp flags
queue lock
```

之后调用 `vb2_queue_init()`。开始流时，UVC 的 `start_streaming` 获取 runtime PM 引用并提交 USB URB；停止时取消视频传输，将仍在队列中的 buffer 归还为 error 状态。

数据链可以简化为：

```text
USB isochronous/bulk URB
  → UVC payload 解析
  → 填充 vb2 buffer
  → vb2_buffer_done
  → 用户空间 VIDIOC_DQBUF
```

UVC 摄像头通过 USB 描述符动态建立拓扑，不能简单地把图中的每个 UVC entity 都等同于独立的内核 `v4l2_subdev`。图中的 subdev 关系更适合描述 SoC sensor/CSI/ISP 这类组件化媒体 pipeline。

## 十、从用户空间反向观察驱动

驱动加载后，可以使用以下命令逐层确认：

```bash
# 查看设备节点
v4l2-ctl --list-devices

# 查看 driver、card、bus_info 和 capabilities
v4l2-ctl --device /dev/video0 --all

# 枚举像素格式、分辨率和帧率
v4l2-ctl --device /dev/video0 --list-formats-ext

# 查看 controls
v4l2-ctl --device /dev/video0 --list-ctrls-menus

# 测试 MMAP streaming
v4l2-ctl --device /dev/video0 \
  --stream-mmap=4 --stream-count=100 --stream-to=/dev/null

# 查看 media topology
media-ctl --print-topology
```

`VIDIOC_QUERYCAP` 返回的 `driver`、`card`、`bus_info` 和 `device_caps` 应作为设备识别依据。只判断路径是不是 `/dev/video0` 不可靠，因为设备编号会随 probe 顺序变化。

## 十一、常见错误

### 把 `v4l2_device` 当成字符设备

它负责聚合和管理，不负责直接创建 `/dev/videoX`。字符设备节点由 `video_device` 注册产生。

### 把 `v4l2_subdev` 当成视频 buffer 节点

subdev 主要描述 pipeline 中的硬件模块和 pad 格式。真正向用户空间排队视频帧的通常是 video node 与 vb2。

### 在 `video_register_device()` 后继续修改关键字段

节点一旦注册，用户空间就可能立即打开它。`fops`、`ioctl_ops`、锁、队列和 drvdata 应在注册前准备完毕。

### `stop_streaming` 没有归还 buffer

停止或错误路径必须把所有驱动持有的 buffer 交还 vb2，否则应用可能永远阻塞。

### release 回调与分配方式不匹配

动态分配的 `video_device` 和内嵌对象的释放策略不同。错误释放会导致 double free、use-after-free 或内存泄漏。

### remove 时立即释放私有对象

USB 拔出或 platform remove 后，已有 file handle 可能还没关闭。需要停止硬件、让新操作失败，并等待最后引用释放。

## 总结

五个核心对象可以用一句话记忆：

```text
v4l2_device：管理一个 V4L2 设备实例
v4l2_subdev：抽象 sensor、CSI、ISP 等子模块
video_device：注册 /dev/videoX 等用户节点
v4l2_fh：保存每一次 open 的 V4L2 上下文
videobuf2：管理流式视频 buffer 的生命周期
```

理解它们之后，再看一个具体驱动的 probe、open、ioctl、streaming 和 disconnect 路径，就能知道每段代码属于设备管理、用户接口还是数据流。

## 参考资料

- [Linux Kernel：V4L2 device instance](https://docs.kernel.org/driver-api/media/v4l2-device.html)
- [Linux Kernel：V4L2 sub-devices](https://docs.kernel.org/driver-api/media/v4l2-subdev.html)
- [Linux Kernel：Video device internal representation](https://docs.kernel.org/driver-api/media/v4l2-dev.html)
- [Linux Kernel：V4L2 file handles](https://docs.kernel.org/driver-api/media/v4l2-fh.html)
- [Linux Kernel：videobuf2 framework](https://docs.kernel.org/driver-api/media/v4l2-videobuf2.html)
- [Linux Kernel：UVC driver source](https://github.com/torvalds/linux/blob/master/drivers/media/usb/uvc/uvc_driver.c)
