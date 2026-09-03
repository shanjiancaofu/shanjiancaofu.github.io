---
title: CAN 通信学习笔记：从总线基础到 Linux SocketCAN
date: 2026-03-14 20:30:00 +0800
categories: [嵌入式]
tags: [CAN, CAN FD, SocketCAN, Linux, STM32]
---

CAN 总线广泛用于汽车和工业控制。它与 UART 最大的不同是：CAN 是多主、基于消息 ID 仲裁的总线，节点发送的是“某类消息”，而不是向某个固定地址发送字节流。

学习 CAN 时，建议把物理层、数据链路层、应用协议和 Linux 程序分开理解。这样遇到“收不到数据”时，才能判断问题究竟在接线、位速率、过滤器还是字节解析。

## 总线与节点

一条典型 CAN 总线由 CAN_H 和 CAN_L 两根差分信号线组成，所有节点并联在总线上。总线两端各放置一个 120 Ω 终端电阻，断电后测量 CAN_H 与 CAN_L 之间通常接近 60 Ω。

```text
120Ω ── 节点 A ── 节点 B ── 节点 C ── 120Ω
          CAN_H / CAN_L
```

节点之间还应有可靠的参考地。CAN 控制器负责帧和仲裁，但 MCU 的 CAN/FDCAN 引脚不能直接接总线，必须经过 CAN 收发器完成电平转换。

## 显性位、隐性位与仲裁

CAN 使用显性 0 和隐性 1。多个节点同时发送时，显性位会覆盖隐性位。发送节点同时监听总线，一旦自己发送 1 却读到 0，就知道仲裁失败并停止发送。

因此数值更小的消息 ID 优先级更高。例如 `0x100` 会优先于 `0x200`。仲裁过程不会破坏获胜帧，失败节点会在总线空闲后重试。

ID 表示优先级和消息类型，不应该直接当作节点地址。车速状态可以拥有一个 ID，所有关心车速的节点都能接收它。

## 标准帧、扩展帧与 CAN FD

经典 CAN 标准帧使用 11 位 ID，扩展帧使用 29 位 ID，单帧最多携带 8 字节数据。CAN FD 将数据长度扩展到最多 64 字节，还可以在数据阶段启用更高位速率 BRS。

CAN FD 并不只是把结构体长度从 8 改成 64。控制器、收发器、Linux 接口配置和总线上其他节点都必须支持相同模式。

## 应用协议必须明确字节定义

CAN 只负责可靠传输帧，不知道两个字节代表车速还是温度。应用层需要明确：

```text
消息 ID
发送周期
数据长度
字节序
缩放系数与偏移
有效范围
sequence / rolling counter
CRC
超时行为
```

例如一个 16 位原始速度值采用小端序，缩放系数为 0.01 m/s：

```cpp
std::uint16_t raw = data[0] |
                    (static_cast<std::uint16_t>(data[1]) << 8);
double speed_mps = raw * 0.01;
```

不要把 C 结构体直接 `memcpy` 到 CAN 帧，因为填充、对齐、字节序和编译器设置可能变化。应逐字段编码，并对长度和范围进行检查。

## 使用 vcan0 学习 SocketCAN

Linux SocketCAN 将 CAN 设备抽象为网络接口。没有真实硬件时，可以先创建虚拟 CAN：

```bash
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set vcan0 up
ip -details link show vcan0
```

安装 `can-utils` 后，打开两个终端测试：

```bash
# 终端一
candump -tz vcan0

# 终端二
cansend vcan0 123#1122334455667788
```

使用 `cangen vcan0` 可以生成测试流量，但应限制范围和频率。真实 `can0` 上不要随意发送未知帧，尤其不能在车辆或执行机构上直接运行随机流量工具。

## 配置真实 CAN 接口

经典 CAN 示例：

```bash
sudo ip link set can0 down
sudo ip link set can0 up type can bitrate 500000 restart-ms 100
ip -details -statistics link show can0
```

CAN FD 示例：

```bash
sudo ip link set can0 down
sudo ip link set can0 up type can bitrate 500000 \
  dbitrate 2000000 fd on restart-ms 100
```

所有节点的仲裁位速率、采样点和 FD 数据位速率必须匹配。若接口错误计数持续增加，应先检查接线、终端电阻、位速率和收发器供电，而不是立即修改解析代码。

## C++ 中打开 SocketCAN

基本步骤与普通 socket 类似：

```cpp
int fd = socket(PF_CAN, SOCK_RAW, CAN_RAW);

ifreq ifr{};
std::strncpy(ifr.ifr_name, "can0", IFNAMSIZ - 1);
ioctl(fd, SIOCGIFINDEX, &ifr);

sockaddr_can addr{};
addr.can_family = AF_CAN;
addr.can_ifindex = ifr.ifr_ifindex;
bind(fd, reinterpret_cast<sockaddr *>(&addr), sizeof(addr));
```

之后可使用 `poll()` 设置接收超时，再读取 `can_frame` 或 `canfd_frame`。文件描述符应封装为 RAII 对象，禁止复制并支持移动，确保异常和提前返回时仍能关闭。

程序还应检查实际读取长度：经典 CAN 是 `CAN_MTU`，CAN FD 是 `CANFD_MTU`。不能把短读或未知长度当成有效帧。

## 过滤器与错误帧

SocketCAN 可以在内核中设置 ID 过滤器，只把关心的消息交给应用，减少无关唤醒。应用也应订阅并记录 CAN 错误帧，例如 bus-off、控制器错误和丢失仲裁。

```cpp
can_filter filter{};
filter.can_id = 0x200;
filter.can_mask = CAN_SFF_MASK;
setsockopt(fd, SOL_CAN_RAW, CAN_RAW_FILTER,
           &filter, sizeof(filter));
```

过滤器只是性能工具，不能代替协议层的 ID、长度、CRC 和 sequence 校验。

## 心跳、超时与旧帧

真实控制系统不能只处理“收到什么”，还要处理“长时间什么都没收到”。例如 STM32 每 100 ms 发送一次心跳，Linux 端设置 300 ms freshness 超时；超时后立即进入降级或安全状态。

接收循环即使 `poll()` 超时也要推进状态机，不能等下一帧到来才发现对端已经离线。

队列积压还会带来旧帧问题。Linux 可通过 `SO_TIMESTAMPNS` 和 `recvmsg()` 获取内核接收时间，再映射到单调时钟判断 freshness。不能在应用处理帧时调用 `now()`，把积压中的旧心跳重新变成“新数据”。

## 分层组织代码

推荐将代码分成三层：

```text
drivers/socketcan
  → socket、bind、poll、recvmsg、时间戳

modules/can
  → 通用 CanFrame、经典 CAN/CAN FD 适配

modules/vehicle
  → ID、字节布局、CRC、heartbeat、业务状态
```

这样更换协议或正式 DBC 时，不需要重写 Linux socket 生命周期；更换底层平台时，业务 codec 也可以继续复用。

## 建议的学习顺序

1. 使用 `vcan0` 跑通 `candump` 和 `cansend`；
2. 编写最小 C/C++ 收发程序；
3. 增加 ID 过滤、poll 超时和错误帧；
4. 编写独立 codec 与单元测试；
5. 加入 sequence、CRC、心跳和 freshness；
6. 连接两块真实开发板，验证位速率和终端电阻；
7. 最后进行拔线、bus-off、节点重启和队列积压测试。

CAN 通信的难点不只是成功收到一帧，而是让错误、丢帧、延迟和对端静默都有确定行为。先在虚拟总线上固定软件合同，再接真实硬件，可以显著降低联调复杂度。
