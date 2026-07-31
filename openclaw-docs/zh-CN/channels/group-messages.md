---
read_when:
    - 专门配置 WhatsApp 群组
    - 更改 WhatsApp 激活模式（`mention` 与 `always`）
    - 调整 WhatsApp 群组会话键或待处理消息上下文
sidebarTitle: WhatsApp groups
summary: WhatsApp 群组消息处理——激活、允许列表、会话和上下文注入
title: WhatsApp 群组消息
x-i18n:
    generated_at: "2026-07-26T05:38:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7325dd3ae64d7abca8c1de0504f294ae280394fa5dd336d2532c5eaefcb03828
    source_path: channels/group-messages.md
    workflow: 16
---

有关跨渠道群组模型（Discord、iMessage、Matrix、Microsoft Teams、QQ Bot、Signal、Slack、Telegram、WhatsApp、Zalo），请参阅[群组](/zh-CN/channels/groups)。本页介绍该模型之上的 WhatsApp 特有行为：激活、群组允许列表、每群组会话键以及待处理消息上下文注入。

目标：让 OpenClaw 加入 WhatsApp 群组，仅在被提及时唤醒，并使该对话与个人私信会话相互独立。

<Note>
`agents.entries.*.groupChat.mentionPatterns` 与其他渠道的提及门控共用。对于多 Agent 设置，请为每个 Agent 分别设置，或使用 `messages.groupChat.mentionPatterns` 作为全局回退。如果两者均未设置，则根据 Agent 身份的名称/表情符号派生模式。
</Note>

## 行为

- 激活模式：`mention`（默认）或 `always`。`mention` 要求提及：真实的 WhatsApp @ 提及（`mentionedJids`）、已配置的正则表达式模式、文本中任意位置出现机器人的 E.164 数字，或引用回复机器人的某条消息（共享号码自聊设置除外）。`always` 会在每条消息到达时唤醒 Agent，但注入的群组提示词会要求它仅在能够提供价值时回复，否则返回完全一致的静默令牌 `NO_REPLY`（不区分大小写）。默认值来自配置（`channels.whatsapp.groups` `requireMention`），并可通过 `/activation` 按群组覆盖。
- 群组允许列表：设置 `channels.whatsapp.groups` 后，仅允许列表中的群组 JID（包含 `"*"` 可允许所有群组）；未列出的群组所发消息会被丢弃，并记录日志提示。
- 群组策略：`channels.whatsapp.groupPolicy` 控制是否接受群组消息（`open|disabled|allowlist`）。`allowlist` 使用 `channels.whatsapp.groupAllowFrom`（回退：显式设置的 `channels.whatsapp.allowFrom`）。默认值为 `allowlist`（在添加发送者前保持阻止状态）。
- 每群组会话：会话键的形式类似 `agent:<agentId>:whatsapp:group:<jid>`（非默认账户会追加 `:thread:whatsapp-account-<accountId>`），因此 `/verbose on`、`/trace on` 或 `/think high` 等指令（作为独立消息发送）仅作用于该群组；个人私信状态不受影响。
- 上下文注入：**仅待处理的**群组消息（默认 50 条）中，未触发运行的消息会添加在 `[Chat messages since your last reply - for context]` 下，触发消息行则添加在 `[Current message - respond to this]` 下。运行后会清空待处理窗口；会话中已有的消息不会再次注入。
- 发送者归属：每条群组消息行都会在消息信封中包含发送者标签，例如 `[WhatsApp <groupJid> <timestamp>] Alice (+447700900123): text`；发送者身份以及群组主题/成员信息会一同包含在不可信的对话元数据块中。
- 临时/阅后即焚消息：提取文本/提及之前会先解开包装，因此其中的提及仍可触发运行。
- 群组系统提示词：群组会话的第一个轮次（以及 `/activation` 更改模式后的任意轮次）会将激活指南注入系统提示词（`Activation: trigger-only ...` 或 `Activation: always-on ...`，外加“回应特定的发送者”）。系统始终会包含持久的群聊发送指南（“你正在 WhatsApp 群聊中……”）。

## 配置示例（WhatsApp）

即使 WhatsApp 从文本正文中移除了可见的 `@`，也能让按显示名称提及生效：

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
      },
      historyLimit: 50, // 待处理群组上下文窗口（默认 50）
    },
  },
  agents: {
    entries: {
      main: {
        groupChat: {
          mentionPatterns: ["@?openclaw", "\\+?15555550123"],
        },
      },
    },
  },
}
```

注意：

- 正则表达式不区分大小写，并采用与其他配置正则表达式表面相同的安全正则保护机制；无效模式和不安全的嵌套重复会被忽略。
- 当有人点按联系人时，WhatsApp 仍会通过 `mentionedJids` 发送规范的提及，因此很少需要号码回退，但它是一个实用的安全保障。
- 待处理上下文窗口按 `channels.whatsapp.accounts.<id>.historyLimit` → `channels.whatsapp.historyLimit` → `messages.groupChat.historyLimit` → 50 的顺序解析。

### 激活命令（仅限所有者）

使用群聊命令：

- `/activation mention`
- `/activation always`

只有所有者号码（来自 `channels.whatsapp.allowFrom`；未设置时为机器人自身的 E.164 号码）才能更改此设置；其他任何人发送的 `/activation` 都会被忽略，仅作为上下文存储。在群组中将 `/status` 作为独立消息发送，可查看当前激活模式。

## 使用方法

1. 将你的 WhatsApp 账户（运行 OpenClaw 的账户）添加到群组。
2. 发送 `@openclaw ...`（或包含该号码）。除非设置 `groupPolicy: "open"`，否则只有允许列表中的发送者才能触发它。
3. Agent 提示词包含待处理的群组上下文及带发送者标签的消息行，使其能够回应正确的人。
4. 会话指令（`/verbose on`、`/trace on`、`/think high`、`/new` 或 `/reset`、`/compact`）仅应用于该群组的会话；请将其作为独立消息发送，以便系统识别。你的个人私信会话保持独立。

## 测试/验证

- 手动冒烟测试：
  - 在群组中发送 `@openclaw` 提及，并确认回复中引用了发送者姓名。
  - 再次发送提及，验证其中包含历史记录块，然后确认该块在下一个轮次中已被清除。
- 检查 Gateway 网关日志（使用 `--verbose` 运行），查看显示 `from: <groupJid>` 和带发送者标签正文的 `inbound web message` 条目。

## 已知注意事项

- Heartbeat 在 Agent 的主会话中运行；群组会话永远不会运行 Heartbeat。
- 回显抑制会按会话记住组合提示词（历史记录 + 当前消息），以防机器人自己发出的消息再次触发运行；内容完全相同的重复批次可能会被判定为回显并跳过。
- 会话存储条目在每 Agent 的 SQLite 会话存储中显示为 `agent:<agentId>:whatsapp:group:<jid>`；缺少条目仅表示该群组尚未触发运行。
- 输入状态指示器遵循 `agents.entries.*.typingMode` / `agents.defaults.typingMode`。当可见回复选择仅使用消息工具的模式时，默认会立即显示输入状态，使群组成员即使在未自动发布最终回复的情况下也能看到 Agent 正在处理。显式的输入模式配置仍具有优先权。

## 相关内容

- [群组](/zh-CN/channels/groups)
- [渠道路由](/zh-CN/channels/channel-routing)
- [广播群组](/zh-CN/channels/broadcast-groups)
