---
read_when:
    - 配置由 Bot 撰写的渠道消息
    - 调整 Bot 间循环保护机制
sidebarTitle: Bot loop protection
summary: Bot 间循环保护的默认设置与渠道覆盖项
title: 机器人循环保护
x-i18n:
    generated_at: "2026-07-26T06:40:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d59d3b48dd5506e774282b880334df8970b05c4d001261ff7107e8e1678894db
    source_path: channels/bot-loop-protection.md
    workflow: 16
---

OpenClaw 可以接收支持 `allowBots` 的渠道中其他 Bot 发送的消息。启用该路径后，Bot 对循环保护可防止两个 Bot 身份无限地相互回复。

该保护机制由核心入站回复运行器执行。每个支持的渠道都会将其入站事件映射为通用事实：账号或作用域、对话 ID、发送方 Bot ID 和接收方 Bot ID。核心会双向跟踪参与方对（A 到 B 和 B 到 A 视为同一对），应用滑动窗口限额，并在超出限额后于冷却期内抑制该参与方对。

## 默认值

只要渠道允许 Bot 发送的消息进入分发流程，Bot 对循环保护就会启用。内置默认值：

| 键                   | 默认值 | 含义                                                |
| -------------------- | ------- | --------------------------------------------------- |
| `enabled`            | `true`  | 对支持该保护的渠道启用保护。                        |
| `maxEventsPerWindow` | `20`    | Bot 对在窗口内可以交换的事件数。                    |
| `windowSeconds`      | `60`    | 滑动窗口长度。                                      |
| `cooldownSeconds`    | `60`    | Bot 对超出限额后的抑制时间。                        |

该保护机制不会影响人类发送的消息、单 Bot 部署、自身消息过滤，也不会影响未超出限额的 Bot 回复。

## 配置共享默认值

设置一次 `channels.defaults.botLoopProtection`，即可为所有支持的渠道提供相同的基准。渠道还可以提供范围更窄的覆盖配置；Feishu 特意仅使用此共享基准。

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
        windowSeconds: 60,
        cooldownSeconds: 60,
      },
    },
  },
}
```

仅当你的渠道策略有意允许 Bot 之间进行对话且不进行自动抑制时，才设置 `enabled: false`。

## 按渠道、账号或房间覆盖

支持的渠道会逐个键地将自身配置叠加在共享默认值之上。优先级从范围最窄到最宽如下：

1. `channels.<channel>.<room-or-space>.botLoopProtection`，当渠道支持按对话覆盖时
2. `channels.<channel>.accounts.<account>.botLoopProtection`，当渠道支持账号时
3. `channels.<channel>.botLoopProtection`，当渠道支持顶层默认值时
4. `channels.defaults.botLoopProtection`
5. 内置默认值

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
      },
    },
    discord: {
      botLoopProtection: {
        maxEventsPerWindow: 8,
      },
      accounts: {
        secondary: {
          allowBots: true,
          botLoopProtection: {
            maxEventsPerWindow: 5,
            cooldownSeconds: 90,
          },
        },
      },
    },
    googlechat: {
      allowBots: true,
      groups: {
        "spaces/AAAA": {
          botLoopProtection: {
            maxEventsPerWindow: 5,
          },
        },
      },
    },
    matrix: {
      allowBots: "mentions",
      groups: {
        "!roomid:example.org": {
          botLoopProtection: {
            maxEventsPerWindow: 5,
          },
        },
      },
    },
    slack: {
      allowBots: "mentions",
      botLoopProtection: {
        maxEventsPerWindow: 8,
      },
    },
  },
}
```

## 渠道支持

- Discord：原生 `author.bot` 事实，按 Discord 账号、频道和 Bot 对确定键。
- Feishu：针对获准进入的 Bot 发送的群组消息提供原生 `sender_type=bot` 事实，按 Feishu 账号、聊天和 Bot 对确定键。Feishu 仅使用 `channels.defaults.botLoopProtection`。
- Google Chat：针对已接受的 Bot 发送消息提供原生 `sender.type=BOT` 事实，按账号、空间和 Bot 对确定键。
- Matrix：已配置的 Matrix Bot 账号，按 Matrix 账号、房间和已配置的 Bot 对确定键。
- Slack：针对已接受的 Bot 发送消息提供原生 `bot_id` 事实，按 Slack 账号、频道和 Bot 对确定键。

无法提供可靠入站 Bot 身份的渠道会继续使用其常规的自身消息过滤器和访问策略过滤器。在能够识别 Bot 对中的双方之前，它们不应启用此保护机制。

有关插件实现的详细信息，请参阅 [SDK 运行时](/zh-CN/plugins/sdk-runtime#reusable-runtime-utilities)。
