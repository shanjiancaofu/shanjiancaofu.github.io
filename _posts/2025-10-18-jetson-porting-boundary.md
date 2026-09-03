---
title: Qt 车机项目重构（三）：从 i.MX6U 迁移到 Jetson 的边界设计
date: 2025-10-18 20:30:00 +0800
categories: [嵌入式]
tags: [Jetson, i.MX6U, Qt, 硬件抽象]
---

将旧 i.MX6U 车机项目搬到 Jetson Orin Nano，最大的误区是只修改几个设备节点，然后期待原程序继续运行。两块平台的内核版本、摄像头链路、GPIO 接口、驱动加载和部署方式都不同，真正需要迁移的是边界。

## 先给旧代码确定身份

旧项目中包含内核驱动、设备树片段，以及 `/dev/sr04`、`/dev/dht11_device`、`/dev/sr501_device` 等板级接口。这些代码很有学习价值，但不能直接作为 Jetson 主线代码。

因此仓库将其放在 `legacy/imx6u` 中，并明确为参考材料。主线只保留与新平台匹配的应用、服务和平台适配代码。这样可以避免构建脚本误编译旧驱动，也防止 UI 继续依赖历史设备名。

## UI 不应该认识硬件细节

理想的依赖方向是：

```text
Qt Widgets / QML
        ↓
service / model
        ↓
hardware abstraction
        ↓
platform/jetson
```

页面只表达“开始预览”“读取传感器状态”或“设置亮度”。设备路径、GStreamer pipeline、GPIO 编号和权限配置属于平台层。

例如摄像头页面不应直接写：

```cpp
open("/dev/video0", O_RDWR);
```

更合适的是依赖一个接口：

```cpp
class CameraService {
public:
    virtual bool startPreview() = 0;
    virtual void stopPreview() = 0;
};
```

桌面环境可以提供 mock 或 USB 摄像头实现，Jetson 再提供 CSI、V4L2 或 GStreamer 实现。

## `/dev/video0` 并不是产品合同

摄像头设备号可能随模块、驱动加载顺序和启动顺序变化。开发时写死 `/dev/video0` 很方便，但设备部署后并不可靠。

更稳妥的办法包括：

- 使用 udev 规则创建稳定软链接；
- 根据设备属性枚举并选择；
- 将产品允许的设备写入部署配置；
- 启动时验证分辨率、像素格式和帧率能力。

同样，不能因为设备节点存在就认为摄像头可用。格式协商、取帧和超时都应进入健康检查。

## 驱动加载不属于页面职责

旧程序可能在按钮槽函数中执行 `insmod`，这在单板实验中容易观察，但在产品程序中会让权限和生命周期失控。Jetson 上的模块加载、udev 权限和开机启动应放到部署目录：

```text
deploy/jetson/
├── scripts/
├── systemd/
└── udev-rules/
```

UI 只消费准备好的能力。如果驱动不可用，服务返回明确错误，页面展示降级状态，而不是临时提权修改系统。

## 将平台差异限制在最小范围

一个常见的目录规划是：

```text
src/platform/
├── desktop/
└── jetson/
    ├── JetsonCamera.cpp
    └── JetsonGpio.cpp
```

上层只包含公共接口头文件。这样在 WSL 或桌面 Linux 上仍能编译大部分 UI 和业务逻辑，必须依赖真机的部分则单独验证。

## 真机验证不能被虚拟环境替代

WSL 适合编译、单元测试和页面开发，却不能证明 CSI 摄像头、硬件 ISP、GPIO 时序或音频设备已经正确工作。平台层的验收应记录设备型号、驱动版本、输入格式、分辨率、帧率和资源占用。

跨平台迁移的核心不是增加大量 `#ifdef`，而是让大多数代码根本不需要知道平台发生了变化。边界清晰后，旧平台代码可以留下供学习，新平台实现也能独立演进，UI 则保持稳定。
