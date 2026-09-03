---
title: 车载 Agent 实践（一）：用显式状态机管理一次对话
date: 2026-05-10 20:30:00 +0800
categories: [AI]
tags: [Agent, 状态机, C++, 对话系统]
---

语音 Agent 表面上是“输入一句话，输出一个回答”，但一次真实交互会经过唤醒、录音、识别、路由、执行、思考和播报等多个阶段。若只使用几个布尔变量表示状态，很快就会出现重复请求、错误打断和资源未释放的问题。

因此项目使用显式有限状态机描述一次对话的生命周期。

## 状态比布尔变量更能表达约束

如果使用 `isListening`、`isSpeaking`、`isThinking` 三个布尔量，理论上会产生八种组合，其中许多组合根本不应该出现。例如系统不能一边识别用户语音，一边播放自己的回答。

显式状态机在同一时间只允许一个状态：

```text
Disabled
Idle
Waking
Listening
Recognizing
Routing
Executing
Thinking
Speaking
FollowUp
Cancelled
ErrorRecovery
ShuttingDown
```

非法组合从数据结构上消失了，代码只需要判断哪些状态转换可以发生。

## 正常对话主路径

一次完整交互的主路径为：

```text
Idle
  → Waking
  → Listening
  → Recognizing
  → Routing
      ├→ Executing
      └→ Thinking
  → Speaking
  → FollowUp
  → Idle
```

`Routing` 是关键分叉点。确定性车控进入 `Executing`，开放问答进入 `Thinking`。两条路径最后都可以生成回复并进入 `Speaking`。

`FollowUp` 表示短时间内允许继续提问。窗口超时后回到 `Idle`；如果再次检测到有效输入，也能直接开始下一轮识别。

## 事件驱动转换

外部模块不直接设置状态，而是提交事件：

```cpp
enum class ConversationEvent {
  kWakeWordDetected,
  kWakePromptCompleted,
  kSpeechSegmentReady,
  kTranscriptReady,
  kActionSelected,
  kOpenRequestSelected,
  kResponseReady,
  kPlaybackCompleted,
  kCancelRequested,
  kFailure,
  kRecoveryCompleted,
  kShutdownRequested,
};
```

状态机根据“当前状态 + 事件”计算下一状态。比如 `TranscriptReady` 只允许从 `Recognizing` 进入 `Routing`；如果系统正在 `Speaking` 却收到这一事件，应拒绝转换，而不是猜测调用者想做什么。

## 记录被拒绝的转换

状态快照不仅包含当前状态，还维护成功和失败的转换计数，以及最近一次转换原因：

```cpp
struct ConversationStateSnapshot {
  InteractionState state;
  std::uint64_t accepted_transitions;
  std::uint64_t rejected_transitions;
  std::string last_reason;
};
```

这让状态机同时成为诊断工具。测试或现场日志看到 rejected 次数增加时，可以定位是否存在乱序回调、重复完成通知或竞态条件。

## 取消、失败和停机不是普通状态

活动会话可以进入 `Cancelled` 或 `ErrorRecovery`，完成清理后才能回到 `Idle`。这样取消不只是设置一个标志，而是一段需要停止模型请求、动作和音频输出的生命周期。

`ShuttingDown` 更严格：所有非停机状态都允许进入它，但进入后不允许再离开。这个规则可以防止模块停止过程中又接受新请求。

## 状态机也控制音频输入

状态机会映射到输入处理策略：

| Agent 状态 | 音频处理 |
|---|---|
| Idle | KWS 唤醒检测 |
| Listening、FollowUp | VAD 与语音分段 |
| Recognizing、Thinking、Speaking 等 | 暂停输入判定 |

第一阶段采用半双工，播报时暂停 KWS、VAD 和 ASR，避免扬声器内容再次触发 Agent。以后加入回声消除和语音打断时，也可以在状态表上明确扩展，而不是向各模块散落条件判断。

## 单会话是一种资源策略

当前 Agent 同一时间只处理一个会话。这并不是能力不足，而是 Jetson 8GB 环境下的明确选择：ASR、LLM、TTS 和相机同时运行时，资源峰值必须受控。

显式状态机把并发限制变成可以测试的合同。Agent 工程中，清楚知道“现在允许做什么”，往往比单独优化某个模型更重要。
