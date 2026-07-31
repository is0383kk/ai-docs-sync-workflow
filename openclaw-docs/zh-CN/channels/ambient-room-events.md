---
read_when:
    - 配置始终开启的群组或频道房间
    - 你希望智能体关注房间内的聊天动态，但不要自动发布最终文本
    - 调试没有可见房间消息时的输入状态和 token 用量
sidebarTitle: Ambient room events
summary: 让受支持的群组房间提供静默上下文，除非智能体使用消息工具发送消息
title: 环境房间事件
x-i18n:
    generated_at: "2026-07-26T06:37:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 15c083c139058c9bd2c651794965bd8252d74691e536db2ad2a2ae0b4ac886e8
    source_path: channels/ambient-room-events.md
    workflow: 16
---

环境房间事件让 OpenClaw 能将群组或频道中未提及智能体的闲聊作为静默上下文处理。智能体可以更新记忆和会话状态，但除非智能体显式调用 `message` 工具，否则房间会保持静默。

对于始终开启的群聊，请将 `messages.groupChat.unmentionedInbound: "room_event"` 与 `messages.groupChat.visibleReplies: "message_tool"` 结合使用。智能体会监听并判断何时回复有帮助，且不再需要使用回复 `NO_REPLY` 的旧提示词模式。

目前支持：Discord 服务器频道、Slack 频道和私有频道、Slack 多人私信，以及 Telegram 群组或超级群组。其他群组渠道会保持其现有群组行为，除非其渠道页面注明支持环境房间事件。

## 推荐设置

设置全局群聊行为：

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
}
```

然后为该房间禁用提及门控，使其始终开启。该房间仍必须通过常规的 `groupPolicy`、房间允许列表和发送者允许列表检查。

保存配置后，Gateway 网关会热应用 `messages` 设置。仅当文件监视或配置重新加载被禁用（`gateway.reload.mode: "off"`）时才需要重启。

## 变化内容

使用 `messages.groupChat.unmentionedInbound: "room_event"` 时：

- 允许的群组或频道中未提及智能体的消息会变为静默房间事件
- 提及智能体的消息仍作为用户请求
- 文本控制命令和原生命令仍作为用户请求
- 中止或停止请求仍作为用户请求
- 私信仍作为用户请求

房间事件采用严格的可见传递方式。智能体的最终文本为私密内容。智能体必须调用 `message(action=send)` 才能在房间中发布消息。

对于房间事件，输入状态和生命周期状态表情回应仍会被抑制。唯一明确的回执例外是 `messages.ackReactionScope: "all"`，它会发送配置的确认表情回应；当房间必须完全保持静默时，请使用更窄的作用域或 `"off"`。

## Discord 示例

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "<DISCORD_SERVER_ID>": {
          requireMention: false,
          users: ["<YOUR_DISCORD_USER_ID>"],
        },
      },
    },
  },
}
```

当仅需将一个频道设为环境房间时，请使用 Discord 的按频道配置。在 `groupPolicy: "allowlist"` 下，列出频道即表示允许该频道（`enabled: false` 会禁用相应条目）：

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "<DISCORD_SERVER_ID>": {
          channels: {
            "<DISCORD_CHANNEL_ID_OR_NAME>": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

## Slack 示例

Slack 频道允许列表优先使用 ID。请使用 `C12345678` 之类的频道 ID，而不是 `#channel-name`。在 `channels.slack.channels` 下列出频道即表示允许该频道（`enabled: false` 会禁用相应条目）：

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    slack: {
      groupPolicy: "allowlist",
      channels: {
        "<SLACK_CHANNEL_ID>": {
          requireMention: false,
        },
      },
    },
  },
}
```

## Telegram 示例

对于 Telegram 群组，Bot 必须能够看到普通群组消息。如果 `requireMention: false`，请禁用 BotFather 隐私模式，或使用其他能将完整群组流量传递给 Bot 的 Telegram 设置。

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    telegram: {
      groups: {
        "<TELEGRAM_GROUP_CHAT_ID>": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

Telegram 群组 ID 通常是 `-1001234567890` 之类的负数。可以从 `openclaw logs --follow` 读取 `chat.id`，将群组消息转发给 ID 查询 Bot，或检查 Bot API 的 `getUpdates`。

## 智能体特定策略

当多个智能体共享同一房间，但只有一个智能体应将未提及它的闲聊视为环境上下文时，请使用智能体覆盖配置：

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "message_tool",
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          unmentionedInbound: "room_event",
          mentionPatterns: ["@openclaw", "openclaw"],
        },
      },
    ],
  },
}
```

智能体特定的 `agents.entries.*.groupChat.unmentionedInbound` 值会为该智能体覆盖 `messages.groupChat.unmentionedInbound`。

## 可见回复模式

对于普通群组或频道用户请求，`messages.groupChat.visibleReplies` 默认为 `"automatic"`。当智能体的最终文本应在无需显式调用消息工具的情况下公开发布时，请保留此默认值。

对于始终开启的环境房间，仍建议使用 `messages.groupChat.visibleReplies: "message_tool"`，尤其是在使用 GPT-5.6 Sol 等最新一代、工具调用可靠的模型时。它允许智能体通过调用消息工具来决定何时发言。如果模型在未调用工具的情况下返回最终文本，OpenClaw 会将该最终文本保留为私密内容，并记录传递被抑制的元数据。

即使其他群组请求使用自动回复，房间事件仍保持严格模式。未提及智能体的环境房间事件始终需要 `message(action=send)` 才能产生可见输出。

## 历史记录

`messages.groupChat.historyLimit` 设置全局群组历史记录默认值（未设置时为 50；必须为正整数）。渠道可以使用 `channels.<channel>.historyLimit` 覆盖此值，部分渠道还支持按账户设置历史记录限制。将渠道级 `historyLimit: 0` 设为禁用，可关闭该渠道的群组历史记录上下文。

支持房间事件的渠道会保留近期环境房间消息作为上下文。Telegram 会维护一个始终开启、按群组滚动的窗口，其大小受 `historyLimit` 限制；用户请求轮次会选择 Bot 上次记录的回复之后的条目，而房间事件轮次会接收完整的近期窗口，使模型能够看到自己最近发布的内容。已弃用的 Telegram `includeGroupHistoryContext` 模式键会由 `openclaw doctor --fix` 移除。

## 故障排查

如果房间显示输入状态或 Token 用量，但没有可见消息：

1. 确认渠道允许列表和发送者允许列表允许该房间。
2. 确认 `requireMention: false` 已在预期的房间层级设置。
3. 检查 `messages.groupChat.unmentionedInbound` 或智能体覆盖配置是否为 `"room_event"`。
4. 检查日志中是否存在被抑制的最终有效负载元数据或 `didSendViaMessagingTool: false`。
5. 对于普通群组请求，如果希望自动发布最终回复，请保留或恢复 `messages.groupChat.visibleReplies: "automatic"`。对于使用 `message_tool` 的环境房间，请使用能可靠调用工具的模型或运行时。

如果 Telegram 环境房间完全不触发，请检查 BotFather 隐私模式，并验证 Gateway 网关是否正在接收普通群组消息。

如果 Slack 环境房间不触发，请验证频道键是否为 Slack 频道 ID，并确认应用具有对应房间类型的历史记录权限范围：`channels:history`（公开）、`groups:history`（私有）或 `mpim:history`（多人私信）。

## 相关内容

- [群组](/zh-CN/channels/groups)
- [Discord](/zh-CN/channels/discord)
- [Slack](/zh-CN/channels/slack)
- [Telegram](/zh-CN/channels/telegram)
- [渠道故障排查](/zh-CN/channels/troubleshooting)
- [频道配置参考](/zh-CN/gateway/config-channels)
