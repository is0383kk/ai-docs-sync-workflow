---
read_when:
    - 更改输入指示器的行为或默认设置
summary: OpenClaw 何时显示输入状态指示器以及如何调整它们
title: 正在输入指示器
x-i18n:
    generated_at: "2026-07-26T06:41:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3c66d61ea7e3e809b8e88ae2eabb9794f0886b629094753716ed02912843ffc
    source_path: concepts/typing-indicators.md
    workflow: 16
---

运行处于活动状态时，会向聊天渠道发送输入状态指示。使用 `agents.defaults.typingMode` 控制输入状态在**何时**开始，使用 `typingIntervalSeconds` 控制其**刷新频率**（保活间隔，默认为 6 秒）。

## 默认值

当 `agents.defaults.typingMode` **未设置**时：

- **私信**：模型循环开始后立即显示输入状态。
- **包含提及的群聊**：立即显示输入状态。
- **不包含提及的群聊**：当已准入的运行产生活跃的用户可见活动（例如 harness 执行活动或消息文本）时，开始显示输入状态。
- **Heartbeat 运行**：如果解析后的 Heartbeat 目标是支持输入状态的聊天，并且未禁用输入状态，则在 Heartbeat 运行开始时显示输入状态。

## 模式

将 `agents.defaults.typingMode` 设置为以下值之一：

- `never` - 永远不显示输入状态指示。
- `instant` - **模型循环一开始**就显示输入状态，即使运行最终只返回静默回复令牌。
- `thinking` - 在出现**第一个推理增量**时，或轮次被接受后 harness 开始活跃执行时，显示输入状态。
- `message` - 在出现**第一个用户可见的回复活动**时显示输入状态，例如活跃的 harness 执行或非静默文本增量。`NO_REPLY` 等静默回复令牌不计为文本活动。

按“触发时间从早到晚”排序：`never` -> `message`/`thinking` -> `instant`。

## 配置

设置 Agent 级别的默认值：

```json5
{
  agents: {
    defaults: {
      typingMode: "thinking",
      typingIntervalSeconds: 6,
    },
  },
}
```

为单个 Agent 覆盖该策略：

```json5
{
  agents: {
    entries: {
      support: {
        typingMode: "message",
      },
    },
  },
}
```

## 注意事项

- `message` 模式不会因静默回复令牌而开始显示输入状态，但在任何助手文本可用之前，活跃执行仍然可以显示输入状态。
- `thinking` 仍会响应流式推理（`reasoningLevel: "stream"`），也可以在推理增量到达之前因活跃执行而开始显示输入状态。
- Heartbeat 输入状态是解析后投递目标的活跃性信号。它在 Heartbeat 运行开始时启动，而不是遵循 `message` 或 `thinking` 的流式传输时序。将 `typingMode: "never"` 设置为禁用它。
- 当 Heartbeat 目标为 `"none"`、无法解析目标、已禁用 Heartbeat 的聊天投递，或渠道不支持输入状态时，Heartbeat 不会显示输入状态。
- `agents.defaults.typingIntervalSeconds` 控制每个 Agent 的**刷新间隔**，而非开始时间。默认值：6 秒。

## 相关内容

<CardGroup cols={2}>
  <Card title="在线状态" href="/zh-CN/concepts/presence" icon="signal">
    Gateway 网关如何跟踪已连接的客户端，以供 Control UI 的设备页面和 macOS 实例标签页使用。
  </Card>
  <Card title="流式传输和分块" href="/zh-CN/concepts/streaming" icon="bars-staggered">
    出站流式传输行为、分块边界和特定于渠道的投递。
  </Card>
</CardGroup>
