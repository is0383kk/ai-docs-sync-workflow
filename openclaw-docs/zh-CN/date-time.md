---
read_when:
    - 你正在更改向模型或用户显示时间戳的方式
    - 你正在调试消息或系统提示词输出中的时间格式问题
summary: 信封、提示词、工具和连接器中的日期与时间处理
title: 日期和时间
x-i18n:
    generated_at: "2026-07-26T06:47:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e6f923022c021c1cf18ba306cd7b9a4873f5df947bb9a8fae9c737a89f64cbf2
    source_path: date-time.md
    workflow: 16
---

OpenClaw 对传输时间戳使用**主机本地时间**，并且在系统提示词中**仅加入时区**。
提供商时间戳会被保留，使工具维持其原生语义。当智能体需要当前
时间时，它会运行 `session_status` 工具。

## 消息信封（默认为本地时间）

入站消息会封装为包含星期和精确到秒的时间戳：

```
[WhatsApp +1555 Mon 2026-01-05 16:26:34 PST] 消息文本
```

无论提供商采用哪个时区，信封时间戳**默认都使用主机本地时间**。
可在 `agents.defaults` 下覆盖：

```json5
{
  agents: {
    defaults: {
      envelopeTimezone: "local", // "utc" | "local" | "user" | IANA 时区
      envelopeTimestamp: "on", // "on" | "off"
      envelopeElapsed: "on", // "on" | "off"
    },
  },
}
```

| 键                  | 值                                                   | 行为                                                                                                                                                                            |
| ------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `envelopeTimezone`  | `local`（默认）、`utc`、`user`、显式 IANA 名称 | `user` 使用 `agents.defaults.userTimezone`（未设置时为主机时区）。显式 IANA 名称（例如 `"America/Chicago"`）会固定使用指定时区；无法识别的名称会回退到 UTC。 |
| `envelopeTimestamp` | `on`（默认）、`off`                                | `off` 会从信封标头、直接智能体提示词前缀和嵌入式模型输入前缀中移除绝对时间戳。                                                       |
| `envelopeElapsed`   | `on`（默认）、`off`                                | `off` 会移除自会话中上一条消息以来显示的已用时间后缀（`+30s` / `+2m` 样式）。                                                               |

### 示例

**本地时间（默认）：**

```
[WhatsApp +1555 Sun 2026-01-18 00:19:42 PST] 你好
```

**用户时区：**

```
[WhatsApp +1555 Sun 2026-01-18 00:19:42 CST] 你好
```

**使用 `envelopeTimezone: "utc"` 的已用时间：**

```
[WhatsApp +1555 +30s Sun 2026-01-18T05:19:00Z] 后续消息
```

## 系统提示词：当前日期和时间

系统提示词包含一个**当前日期和时间**部分，其中**仅包含时区**
（不包含时钟或时间格式），以保持提示词缓存稳定：

```
时区：America/Chicago
```

配置后，时区为 `agents.defaults.userTimezone`；否则为主机时区。
提示词还会指示智能体在需要当前日期、时间或星期时运行
`session_status` 工具。

## 系统事件行（默认为本地时间）

插入智能体上下文的排队系统事件会添加时间戳前缀，并使用与
消息信封相同的 `envelopeTimezone` 选择（默认：主机本地时间）。

```
系统：[2026-01-12 12:19:17 PST] 模型已切换。
```

### 配置用户时区和格式

```json5
{
  agents: {
    defaults: {
      userTimezone: "America/Chicago",
      timeFormat: "auto", // auto | 12 | 24
    },
  },
}
```

- `userTimezone` 设置提示词上下文（以及 `envelopeTimezone: "user"`）的**用户本地时区**。
- `timeFormat` 控制面向提示词的时间采用 **12 小时制还是 24 小时制**显示。`auto` 遵循操作系统偏好设置。

## 时间格式检测（自动）

当 `timeFormat: "auto"` 时，OpenClaw 会检查操作系统偏好设置（macOS 和 Windows），
并回退到区域设置格式。检测到的值会**按进程缓存**，
以避免重复进行系统调用。

## 工具载荷和连接器（原始提供商时间和规范化字段）

渠道工具会返回**提供商原生时间戳**，并添加规范化字段以保持一致性：

- `timestampMs`：纪元毫秒数（UTC）
- `timestampUtc`：ISO 8601 UTC 字符串

原始提供商字段会被保留，确保信息不会丢失。

- Discord：UTC ISO 时间戳
- Slack：来自 API、格式类似纪元时间的字符串
- Telegram/WhatsApp：提供商特定的数字或 ISO 时间戳

如果需要本地时间，请在下游使用已知时区进行转换。

## 相关文档

- [系统提示词](/zh-CN/concepts/system-prompt)
- [时区](/zh-CN/concepts/timezone)
- [消息](/zh-CN/concepts/messages)
