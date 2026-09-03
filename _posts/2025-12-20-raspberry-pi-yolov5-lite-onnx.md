---
title: 树莓派部署 YOLOv5-Lite：ONNX Runtime 实时检测与坐标输出
date: 2025-12-20 20:30:00 +0800
categories: [AI]
tags: [树莓派, YOLOv5-Lite, ONNX Runtime, OpenCV, 目标检测]
---

之前在电脑上完成了 YOLOv5s 模型训练，但将 `.pt` 权重直接放到树莓派运行时，PyTorch 环境较重，推理速度和安装兼容性也不理想。参考几篇树莓派部署记录后，我采用了更适合 ARM CPU 的路线：训练仍在电脑上完成，将轻量模型导出为 ONNX，树莓派只安装 OpenCV 与 ONNX Runtime，负责摄像头采集、推理和结果显示。

本文记录从树莓派环境准备、图片采集、模型训练与转换，到实时检测和输出目标坐标的完整过程。

## 整体流程

```text
树莓派采集图片 → 电脑端标注与训练 → best.pt 转 best.onnx
                                      ↓
树莓派 OpenCV 取帧 → ONNX Runtime 推理 → NMS → 显示类别、置信度和坐标
```

这种分工避免在树莓派上训练模型，也不要求部署端保留完整 PyTorch 环境。

## 一、准备树莓派系统

推荐使用 64 位 Raspberry Pi OS。烧录系统时可以提前设置 Wi-Fi、主机名、用户名和 SSH，启动后通过 SSH 或 VNC 远程操作。

首先检查系统架构、Python 和摄像头：

```bash
uname -m
python3 --version
rpicam-hello --list-cameras
```

较旧系统的相机命令可能叫 `libcamera-hello`。新系统使用 `rpicam-*`，不要为了复现旧教程而混装两代相机组件。

安装基础工具：

```bash
sudo apt update
sudo apt install -y git python3-pip python3-venv \
  python3-opencv python3-picamera2 rpicam-apps \
  libopenblas-dev libjpeg-dev v4l-utils
```

创建虚拟环境：

```bash
python3 -m venv --system-site-packages ~/venvs/yolov5-lite
source ~/venvs/yolov5-lite/bin/activate
python -m pip install --upgrade pip
```

使用 `--system-site-packages` 可以在虚拟环境中访问系统提供的 `cv2` 和 `picamera2`。Raspberry Pi OS Bookworm 启用了 PEP 668，不建议使用 `sudo pip install` 修改系统 Python。

## 二、安装并验证 OpenCV

先确认 OpenCV 能导入：

```bash
python -c "import cv2; print(cv2.__version__)"
```

USB 摄像头可以使用 V4L2 查看设备与格式：

```bash
v4l2-ctl --list-devices
v4l2-ctl --device /dev/video0 --list-formats-ext
```

最小测试程序：

```python
import cv2

camera = cv2.VideoCapture(0)
camera.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
camera.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)

while camera.isOpened():
    ok, frame = camera.read()
    if not ok:
        break
    cv2.imshow("camera", frame)
    if cv2.waitKey(1) & 0xFF == ord("q"):
        break

camera.release()
cv2.destroyAllWindows()
```

CSI 摄像头在新系统中由 libcamera 管理，`VideoCapture(0)` 不一定可用。此时使用 Picamera2：

```python
from picamera2 import Picamera2

camera = Picamera2()
config = camera.create_preview_configuration(
    main={"size": (640, 480), "format": "RGB888"}
)
camera.configure(config)
camera.start()
frame = camera.capture_array()
```

先让纯摄像头程序稳定运行，再接入模型。否则画面打不开时，很难判断是驱动问题还是推理代码问题。

## 三、采集和标注自己的数据

如果只使用 COCO 预训练权重，可以跳过训练步骤。需要识别自定义目标时，应在最终使用环境中采集图片，覆盖不同距离、角度、光照、背景和遮挡情况。

```python
import cv2
from pathlib import Path

output = Path("images")
output.mkdir(exist_ok=True)
camera = cv2.VideoCapture(0)
index = 0

while camera.isOpened():
    ok, frame = camera.read()
    if not ok:
        break
    cv2.imshow("collect", frame)
    key = cv2.waitKey(1) & 0xFF
    if key == ord("s"):
        path = output / f"image_{index:04d}.jpg"
        cv2.imwrite(str(path), frame)
        print("saved:", path)
        index += 1
    elif key == ord("q"):
        break

camera.release()
cv2.destroyAllWindows()
```

将图片传到电脑后，使用 LabelImg 等工具按 YOLO 格式标注：

```text
dataset/
├── images/
│   ├── train/
│   └── val/
└── labels/
    ├── train/
    └── val/
```

每张图片对应一个同名 `.txt` 文件，每行格式为：

```text
class_id x_center y_center width height
```

坐标按图像宽高归一化到 0～1。连续视频帧不要随机分散到训练集和验证集，否则相邻画面过于相似，会让验证结果虚高。

## 四、在电脑上训练模型

准备数据集配置 `dataset.yaml`：

```yaml
path: /path/to/dataset
train: images/train
val: images/val
names:
  0: target
```

如果使用官方 YOLOv5s：

```bash
python train.py \
  --weights yolov5s.pt \
  --data dataset.yaml \
  --imgsz 640 \
  --epochs 100 \
  --batch-size 16
```

如果使用 YOLOv5-Lite，则按对应仓库提供的轻量网络配置和预训练权重训练。不同仓库的网络文件、输出层和导出脚本可能不同，不能只把权重文件改名为 `yolov5s.pt`。

训练完成后保存验证效果最好的 `best.pt`，同时记录代码 commit、数据集版本、类别顺序和训练参数。

## 五、将 PT 权重转换为 ONNX

在训练电脑中导出：

```bash
python export.py \
  --weights runs/train/exp/weights/best.pt \
  --include onnx \
  --imgsz 640 \
  --opset 12
```

YOLOv5-Lite 仓库可能提供自己的导出脚本，应优先使用与训练代码同一版本的脚本。导出后先用固定图片对比 PT 与 ONNX 结果。

检查模型输入输出：

```python
import onnxruntime as ort

session = ort.InferenceSession("best.onnx")
for item in session.get_inputs():
    print("input:", item.name, item.shape, item.type)
for item in session.get_outputs():
    print("output:", item.name, item.shape, item.type)
```

不要假设所有模型输出都是 `[1, 25200, 85]`。自定义类别数、Lite 分支版本以及是否把 NMS 导入模型，都会改变输出形状。

记录模型哈希并传到树莓派：

```bash
sha256sum best.onnx
scp best.onnx pi@raspberrypi.local:~/models/
```

## 六、安装 ONNX Runtime

```bash
source ~/venvs/yolov5-lite/bin/activate
python -m pip install numpy onnxruntime
python -c "import onnxruntime as ort; print(ort.__version__, ort.get_available_providers())"
```

ARM64 wheel 是否可用取决于系统、Python 和 ONNX Runtime 版本。如果没有匹配版本，先核对 `uname -m` 与 Python ABI，不要直接下载来源不明的 wheel。CPU 部署应至少看到 `CPUExecutionProvider`。

## 七、图像预处理

摄像头画面通常不是正方形，直接拉伸会改变目标比例，因此使用 letterbox：

```python
import cv2
import numpy as np

def letterbox(image, size=640):
    height, width = image.shape[:2]
    scale = min(size / width, size / height)
    new_width = round(width * scale)
    new_height = round(height * scale)
    resized = cv2.resize(image, (new_width, new_height))

    canvas = np.full((size, size, 3), 114, dtype=np.uint8)
    left = (size - new_width) // 2
    top = (size - new_height) // 2
    canvas[top:top + new_height, left:left + new_width] = resized
    return canvas, scale, left, top
```

ONNX Runtime 常见输入为 NCHW、RGB、`float32`，像素归一化到 0～1：

```python
input_image = cv2.cvtColor(input_image, cv2.COLOR_BGR2RGB)
input_tensor = input_image.astype(np.float32) / 255.0
input_tensor = np.transpose(input_tensor, (2, 0, 1))
input_tensor = np.expand_dims(input_tensor, axis=0)
input_tensor = np.ascontiguousarray(input_tensor)
```

Picamera2 配置为 `RGB888` 时要核对实际数组颜色顺序，避免重复转换造成红蓝通道颠倒。

## 八、推理、NMS 与坐标还原

```python
input_name = session.get_inputs()[0].name
outputs = session.run(None, {input_name: input_tensor})
predictions = outputs[0][0]
```

对未包含 NMS 的常见 YOLOv5 输出，每个候选框包括中心坐标、宽高、objectness 和各类别概率：

```python
objectness = row[4]
class_id = int(np.argmax(row[5:]))
confidence = float(objectness * row[5 + class_id])
```

过滤低置信度候选后，使用 `cv2.dnn.NMSBoxes` 去除高度重叠的框。如果模型已包含 NMS，则不要重复处理。

模型坐标基于 letterbox 后的输入，需要减去 padding，再除以缩放比例：

```python
x1 = int((x1_model - left) / scale)
y1 = int((y1_model - top) / scale)
x2 = int((x2_model - left) / scale)
y2 = int((y2_model - top) / scale)

x1 = max(0, min(x1, frame_width - 1))
y1 = max(0, min(y1, frame_height - 1))
x2 = max(0, min(x2, frame_width - 1))
y2 = max(0, min(y2, frame_height - 1))
```

如果忘记还原 padding，绘制出来的检测框会整体偏移。

## 九、输出目标坐标

NMS 后输出左上角、右下角、中心点、类别和置信度：

```python
center_x = (x1 + x2) // 2
center_y = (y1 + y2) // 2

print(
    f"class={class_names[class_id]} "
    f"confidence={confidence:.3f} "
    f"xyxy=({x1}, {y1}, {x2}, {y2}) "
    f"center=({center_x}, {center_y})"
)

cv2.rectangle(frame, (x1, y1), (x2, y2), (0, 255, 0), 2)
cv2.circle(frame, (center_x, center_y), 4, (0, 0, 255), -1)
cv2.putText(frame, f"{class_names[class_id]} {confidence:.2f}",
            (x1, max(20, y1 - 8)), cv2.FONT_HERSHEY_SIMPLEX,
            0.5, (0, 255, 0), 1)
```

如果坐标还要发送给机械臂、云台或其他模块，应携带帧序号、图像尺寸和采集时间戳。像素中心点只是二维图像坐标，不能直接当作真实世界距离或转角。

## 十、性能优化

先分别统计采集、预处理、推理、NMS 和显示耗时：

```python
from time import perf_counter

started = perf_counter()
outputs = session.run(None, {input_name: input_tensor})
inference_ms = (perf_counter() - started) * 1000
```

建议按以下顺序优化：

1. 将模型输入从 640 调整为 416 或 320；
2. 使用 YOLOv5-Lite，而不是在板端加载更大的模型；
3. 采集线程与推理解耦，只保留最新帧；
4. 降低显示和日志刷新频率；
5. 不在循环中重复创建 `InferenceSession`；
6. 给树莓派增加散热，检查持续运行是否降频；
7. 根据实测调整 ONNX Runtime 线程数。

摄像头以 30 FPS 采集、模型只能处理 5 FPS 时，应丢弃旧帧并处理最新画面，否则检测结果会越来越滞后。

## 十一、常见问题

### 找不到 `cv2`

如果 OpenCV 通过 apt 安装，而虚拟环境看不到它，应确认环境是否使用了 `--system-site-packages`。

### ONNX Runtime 无法安装

检查是否为 64 位系统、Python 版本是否存在对应 ARM64 wheel，不能混用 32 位用户态与 64 位包。

### 模型运行但没有检测框

依次核对 RGB/BGR、归一化、NCHW/NHWC、类别数量、输出结构、置信度计算和 NMS 输入格式。

### 检测框位置不正确

检查 letterbox 的 `scale`、`left`、`top` 是否用于坐标还原，以及坐标格式究竟是 `xywh` 还是 `xyxy`。

### 摄像头延迟越来越大

减小 V4L2 buffer，或让采集线程覆盖最新帧，不要让推理线程消费无界 FIFO。

## 十二、记录部署结果

一次可复现的测试至少记录：

```text
树莓派型号、内存与散热方式
Raspberry Pi OS 和 Python 版本
OpenCV、ONNX Runtime 版本
模型仓库 commit、权重哈希和类别表
输入尺寸、置信度与 NMS 阈值
摄像头分辨率和像素格式
预热后的平均/P95 推理延迟
端到端 FPS、CPU、内存、温度和连续运行时间
```

不同文章中的 FPS 只能作为参考，因为模型、输入尺寸、树莓派型号和散热条件可能完全不同。自己的项目应使用固定图片集做正确性对比，再进行真机摄像头长时间测试。

## 参考资料

- [YOLOv5lite + ONNX Runtime 树莓派实时识别检测](https://blog.csdn.net/CadyCady/article/details/141218126)
- [树莓派 YOLOv5 部署参考（一）](https://blog.csdn.net/black_sneak/article/details/131374492)
- [树莓派 OpenCV 环境参考（二）](https://blog.csdn.net/black_sneak/article/details/131343797)
- [Ultralytics YOLOv5](https://github.com/ultralytics/yolov5)
