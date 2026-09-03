---
title: Qt 车机项目重构（二）：过渡期的进程启动与日志管理
date: 2025-09-12 20:30:00 +0800
categories: [C++]
tags: [Qt, QProcess, 日志, 工程化]
---

统一车机外壳完成后，旧的音乐、视频和天气程序不可能在一天内全部改成新页面。比较实际的做法是保留独立程序，由主界面临时拉起它们，同时把进程启动封装成服务。

## 不在按钮槽函数中拼命令

最直接的写法是在按钮点击后调用 shell 命令，但这会带来路径、转义和注入问题。项目中的 `AppProcessService` 只接收程序名，并把它解析到主程序所在目录：

```cpp
QString AppProcessService::executablePath(const QString &name) const
{
    return QDir(QCoreApplication::applicationDirPath()).filePath(name);
}
```

启动前先确认目标存在并且是普通文件，然后使用 `QProcess::startDetached`：

```cpp
const QFileInfo info(path);
if (!info.exists() || !info.isFile()) {
    emit launchFailed(programName, "Executable not found");
    return false;
}

return QProcess::startDetached(
    path, {}, QCoreApplication::applicationDirPath());
```

这里没有经过 shell，也没有让页面提交任意参数。虽然它还是过渡方案，但边界比到处调用 `system()` 清晰得多。

## 为什么使用相对部署目录

顶层 qmake 工程将可执行文件统一输出到 `build/bin`。主程序通过 `applicationDirPath()` 寻找兄弟程序，不依赖开发者机器上的绝对路径。

```text
build/bin/
├── VehicleSystem
├── QMusicPlayer
├── QVideoPlayer
└── HttpGetWeather
```

这样复制整个运行目录后仍能启动，也便于未来制作安装包。资源目录同样应在部署阶段明确准备，而不是依赖源码树的当前位置。

## 用信号返回失败原因

启动服务不应该弹窗，因为它不知道当前使用的是 Widgets、QML 还是命令行测试。服务通过信号报告失败：

```cpp
emit launchFailed(programName, reason);
```

页面决定把错误显示在状态文字、对话框还是诊断页中。这个小设计使业务能力不依赖具体 UI。

## 日志为什么需要尽早统一

多程序过渡期间，终端会同时出现多个 Qt 应用的输出，很难判断消息来自哪里。项目安装全局 Qt 消息处理器，把日志按程序名写入文件：

```text
logs/VehicleSystem.log
logs/QMusicPlayer.log
logs/QVideoPlayer.log
logs/HttpGetWeather.log
```

每条日志包含时间、等级、源文件、行号和函数名。写入过程使用互斥锁保护，因为 Qt 日志可能来自不同线程。

## 控制日志体积与重复输出

嵌入式设备的存储空间有限，日志不能无限增长。当前实现采用简单的一代轮转：文件达到约 2 MiB 时，旧文件改名为 `.log.1`，再创建新文件。

连续相同的消息也不必全部落盘。记录器保存上一条消息的键，对重复内容只累计次数，并周期性写入摘要：

```text
previous message repeated 50 times
```

这对高频状态查询或设备异常尤其有效。不过生产系统还应补充总存储上限、断电安全以及敏感信息过滤。

## 环境变量只做部署配置

日志目录和是否同步输出到终端可由环境变量控制：

```bash
export VEHICLE_SYSTEM_LOG_DIR=/tmp/vehicle-system-logs
export VEHICLE_SYSTEM_LOG_TO_STDERR=1
```

代码不需要知道自己运行在开发机还是设备上。开发阶段可以同时看终端，部署后默认只写轮转文件。

过渡架构不是最终架构。独立程序以后仍应逐步收回主进程，由服务和数据模型管理。但在迁移完成前，受控的进程启动、明确的失败反馈和可追踪的日志，能让旧功能成为可管理的资产，而不是新的技术债入口。
