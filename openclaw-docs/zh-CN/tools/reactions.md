---
read_when:
    - 在任何渠道中处理表情回应
    - 了解各平台上表情回应的差异
summary: 所有受支持渠道中的表情回应工具语义
title: 表情回应
x-i18n:
    generated_at: "2026-07-26T06:37:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e148a93edbcfbe997075f6e9e191667ec257f76fa48162688fd1f333479661f0
    source_path: tools/reactions.md
    workflow: 16
---

智能体使用 `message` 工具的 `react`
操作添加和移除表情回应。具体行为因渠道而异。

## 工作原理

```json
{
  "action": "react",
  "messageId": "msg-123",
  "emoji": "thumbsup"
}
```

- 添加表情回应时必须提供 `emoji`。
- 在支持此功能的渠道上，将 `emoji` 设置为空字符串（`""`），即可移除 Bot 的表情回应。
- 设置 `remove: true` 可移除一个特定表情（要求
  `emoji` 非空）。
- 在支持状态表情回应的渠道上，对表情回应设置 `trackToolCalls: true` 后，
  运行时可在同一轮次中复用该已添加回应的消息，以显示后续工具进度
  表情回应。

## 渠道行为

<AccordionGroup>
  <Accordion title="Discord 和 Slack">
    - 将 `emoji` 留空会移除该消息上 Bot 的所有表情回应。
    - `remove: true` 仅移除指定的表情。

  </Accordion>

  <Accordion title="Nextcloud Talk">
    - 仅支持添加表情回应：必须提供非空的 `emoji`。
    - 表情回应移除功能尚未接入删除调用；使用 `remove: true` 时会返回明确错误，而不会静默地不执行任何操作。
    - 要求注册 Talk Bot 时启用 `reaction` 功能（参阅 [Nextcloud Talk 渠道文档](/zh-CN/channels/nextcloud-talk)）。

  </Accordion>

  <Accordion title="Telegram">
    - 将 `emoji` 留空会移除 Bot 的表情回应。
    - `remove: true` 也会移除表情回应，但工具验证仍要求 `emoji` 非空。

  </Accordion>

  <Accordion title="WhatsApp">
    - 将 `emoji` 留空会移除 Bot 的表情回应。
    - `remove: true` 会在内部映射为空表情（但工具调用中仍需提供 `emoji`）。
    - WhatsApp 中每条消息只有一个 Bot 表情回应槽位；发送新的表情回应会替换原有回应，而不会叠加多个表情。

  </Accordion>

  <Accordion title="Zalo Personal（zalouser）">
    - 添加和移除操作均要求 `emoji` 非空。
    - `remove: true` 会移除该特定表情回应。

  </Accordion>

  <Accordion title="Feishu/Lark">
    - 与其他渠道使用相同的 `react` 操作（通过消息表情回应 ID 添加、移除和列出），而不是使用单独的工具。
    - 添加时要求 `emoji` 非空（映射为 Feishu `emoji_type`，例如 `SMILE`、`THUMBSUP`、`HEART`）。
    - `remove: true` 要求 `emoji` 非空，并移除 Bot 自己添加的、与该表情类型匹配的回应。
    - 将 `emoji` 留空并使用 `clearAll: true`，会移除该消息上 Bot 的所有表情回应。

  </Accordion>

  <Accordion title="Signal">
    - 入站表情回应通知由 `channels.signal.reactionNotifications` 控制：`"off"` 会禁用通知；`"own"`（默认值）会在用户回应 Bot 消息时发出事件；`"all"` 会为所有表情回应发出事件；`"allowlist"` 则仅为 `channels.signal.reactionAllowlist` 中的发送者发出事件。

  </Accordion>

  <Accordion title="iMessage">
    - 出站表情回应使用 iMessage 点回（`love`、`like`、`dislike`、`laugh`、`emphasize` 和 `question`）；添加表情回应时，`emoji` 必须映射为其中一种类型。
    - 使用 `remove: true` 时，如果未指定可识别的点回类型，则会移除所有点回类型；如果指定了可识别的类型，则仅移除该类型。

  </Accordion>
</AccordionGroup>

## 表情回应级别

每个渠道的 `reactionLevel` 会限制智能体发送自身
表情回应的频率。可选值：`off`、`ack`、`minimal` 或 `extensive`。

- [Telegram 表情回应通知](/zh-CN/channels/telegram#feature-reference) - `channels.telegram.reactionLevel`（默认值为 `minimal`）
- [WhatsApp 表情回应级别](/zh-CN/channels/whatsapp#reaction-level) - `channels.whatsapp.reactionLevel`（默认值为 `minimal`）
- [Signal 表情回应](/zh-CN/channels/signal#reactions-message-tool) - `channels.signal.reactionLevel`（默认值为 `minimal`）

## 相关内容

- [Agent Send](/zh-CN/tools/agent-send) - 包含 `react` 的 `message` 工具
- [渠道](/zh-CN/channels) - 渠道特定配置
