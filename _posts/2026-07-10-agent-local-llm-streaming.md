---
title: 车载 Agent 实践（三）：本地 LLM 的流式响应、超时与取消
date: 2026-07-10 20:30:00 +0800
categories: [AI]
tags: [Agent, LLM, SSE, llama.cpp, C++]
---

确定性路由没有命中时，Agent 会把普通问答交给本地 LLM。项目通过 llama.cpp 的 `llama-server` 提供 OpenAI-compatible HTTP 接口，并在 C++ 中实现一个只返回文本的 `LocalLlmClient`。

真正的难点不在发送一段 JSON，而在流式协议、首内容延迟、总超时和可取消性。

## 用最小接口隔离推理实现

业务层只依赖下面的抽象：

```cpp
class LocalLlmClient {
public:
  virtual LocalLlmResult GenerateResponse(
      const SpeechTranscript &transcript,
      std::chrono::steady_clock::time_point deadline) = 0;
  virtual void Cancel() = 0;
};
```

返回值包含成功状态、回复正文、provider、错误信息、首内容延迟和总延迟。接口中没有工具调用、动作类型、shell 或任意 RPC，因此模型能力被限制在文本回复通道。

## 请求只发送 system 与 user 消息

请求启用流式输出并限制生成长度：

```json
{
  "model": "Qwen3.5-2B",
  "stream": true,
  "max_tokens": 128,
  "messages": [
    {"role": "system", "content": "You are a cockpit assistant."},
    {"role": "user", "content": "..."}
  ]
}
```

Qwen3.5 请求还显式关闭 thinking，推理字段不会进入最终播报正文。车机问答强调短响应和稳定延迟，不需要把长推理过程读给用户。

## HTTP 流与 SSE 是两层协议

服务端可能使用 `Transfer-Encoding: chunked`。客户端要先解析 HTTP chunk，把分块还原成连续 body，再按 SSE 行消费 `data:` 事件。

```text
TCP 字节流
  → HTTP headers
  → chunked decoder
  → SSE line buffer
  → JSON delta
  → response_text
```

一次 `recv()` 不保证对应一个 HTTP chunk，更不保证对应一条完整 SSE 消息。客户端必须保留 pending buffer，等待分隔符完整后再解析。

JSON 字符串还要正确处理转义和 Unicode，包括代理对。否则中英文混合回答可能在分片边界处产生乱码。

## 首内容超时与总超时应分开

用户体验主要受两种延迟影响：多久听到第一个有效内容，以及完整回答多久结束。

`first_token_timeout` 用于发现模型未加载、服务卡住或排队过久；总 deadline 则防止回答无限生成。客户端使用 `steady_clock` 计算超时，系统时间调整不会改变请求期限。

结果分别记录：

```text
first_content_latency
total_latency
```

只有把两者分开，性能测试才能判断问题发生在提示处理、首 token 生成还是后续解码速度。

## 取消需要主动关闭阻塞点

只设置 `cancelled=true` 不足以取消正在等待网络数据的线程。客户端记录当前活动 socket，`Cancel()` 主动关闭连接，使阻塞中的 poll/recv 尽快返回。

实现还要处理竞争：请求正常结束时不能关闭后来建立的新 socket，取消线程也不能访问已经释放的描述符。因此活动 socket 的注册和清理需要互斥保护，并确认描述符仍属于当前请求。

## 流式输出不等于直接逐字播报

LLM token 可以持续到达，但 TTS 更适合处理完整短句。项目先按中英文句末标点、换行或有界 UTF-8 长度切分文本，再逐段合成和播放。

```text
LLM token stream
  → 文本聚合
  → 句子切分
  → TTS PCM
  → 播放完成回执
```

前一段播放完成后再提交下一段，可以限制内存和音频队列，也便于取消剩余回答。

本地 LLM 接入不是一次普通 HTTP 调用。只有明确消息边界、超时、取消和输出节流，生成式能力才能成为长期运行 Agent 中可管理的一环。
