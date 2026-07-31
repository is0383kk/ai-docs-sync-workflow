---
read_when:
    - 你想快速了解时区处理的基本思路
    - 你正在决定在哪里设置或覆盖时区
summary: 时区在 OpenClaw 中的显示位置——消息封装、工具载荷、系统提示词
title: 时区
x-i18n:
    generated_at: "2026-07-26T06:12:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d1620b4b2cedba89bd6ab4392018cd48d0ef92a6abc1744011d482557e2c4fc
    source_path: concepts/timezone.md
    workflow: 16
---

OpenClaw 对时间戳进行标准化，使模型看到**单一参考时间**，而不是混合使用各提供商的本地时钟。以下三个位置会显示时区，各有不同用途：

## 三个时区位置

| 位置 | 显示内容 | 默认值 | 配置方式 |
| ----------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------- | ------------------------------------------------------ |
| 消息信封 | 包装传入的渠道消息：`[Signal +1555 Sun 2026-01-18 00:19:42 PST] hello` | 主机本地时区 | `agents.defaults.envelopeTimezone` |
| 工具载荷 | 渠道 `readMessages` 类工具返回原始提供商时间，以及标准化的 `timestampMs` / `timestampUtc` | 始终包含 UTC 字段 | 不可配置；保留提供商原生时间戳 |
| 系统提示词 | 一个简短的 `Current Date & Time` 块，仅包含**时区**（不含时钟值，以保持缓存稳定） | 未设置 `userTimezone` 时使用主机时区 | `agents.defaults.userTimezone` |

系统提示词特意省略实时时钟，以保持各轮次间的提示词缓存稳定。当智能体需要当前时间时，它会调用 `session_status`。

## 设置用户时区

```json5
{
  agents: {
    defaults: {
      userTimezone: "America/Chicago",
    },
  },
}
```

如果未设置 `userTimezone`，OpenClaw 会在运行时通过 `Intl.DateTimeFormat().resolvedOptions().timeZone` 解析主机时区（不会写入配置）。`agents.defaults.timeFormat`（`auto` | `12` | `24`）控制消息信封和下游位置采用 12 小时制还是 24 小时制，不影响系统提示词部分。

## 消息信封时区值

`agents.defaults.envelopeTimezone` 接受：

- `"local"`（默认）或 `"host"`——主机的时区。
- `"utc"` 或 `"gmt"`——UTC。
- `"user"`——解析后的 `agents.defaults.userTimezone`（未设置时回退到主机时区）。
- 任意明确的 IANA 时区字符串，例如 `"Europe/Vienna"`。

## 何时覆盖

- 当不同地区的主机需要使用一致的时间戳，或需要与采用 UTC 的诊断/日志输出保持一致时，**使用 `"utc"`**。
- 若要使消息信封始终与已配置的用户时区保持一致，而不受 Gateway 网关主机所在时区影响，**使用 `"user"`**。
- 当 Gateway 网关主机位于一个时区，但无论主机如何迁移，消息信封都应始终显示另一个时区时，**使用固定的 IANA 时区**。
- 当时间戳上下文对对话没有帮助时，**设置 `envelopeTimestamp: "off"`**。这会从消息信封、直接智能体提示词前缀和嵌入的模型输入前缀中移除绝对时间戳。

有关完整的行为参考、各提供商示例和已用时间格式，请参阅[日期与时间](/zh-CN/date-time)。

## 相关内容

- [日期与时间](/zh-CN/date-time)——完整的消息信封、工具和提示词行为及示例。
- [Heartbeat](/zh-CN/gateway/heartbeat)——活跃时段使用时区进行调度。
- [定时任务](/zh-CN/automation/cron-jobs)——cron 表达式使用时区进行调度。
