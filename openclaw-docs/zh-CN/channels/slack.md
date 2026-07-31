---
read_when:
    - 设置 Slack 或调试 Slack 套接字、HTTP 或中继模式
summary: Slack 设置与运行时行为（Socket Mode、HTTP Request URLs 和中继模式）
title: Slack
x-i18n:
    generated_at: "2026-07-26T06:02:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0f974ddf8e6965b09cede6a16f171434915a994fa3c1fc744d2350399941bee
    source_path: channels/slack.md
    workflow: 16
---

OpenClaw 的 Slack 支持通过 Slack 应用集成覆盖私信和频道。默认传输方式为 Socket Mode；也支持 HTTP Request URLs。Relay 模式适用于由可信路由器负责 Slack 入口的托管式部署。

<CardGroup cols={3}>
  <Card title="配对" icon="link" href="/zh-CN/channels/pairing">
    Slack 私信默认使用配对模式。
  </Card>
  <Card title="斜杠命令" icon="terminal" href="/zh-CN/tools/slash-commands">
    原生命令行为和命令目录。
  </Card>
  <Card title="频道故障排除" icon="wrench" href="/zh-CN/channels/troubleshooting">
    跨频道诊断和修复操作手册。
  </Card>
</CardGroup>

## 选择传输方式

Socket Mode 和 HTTP Request URLs 在消息传递、斜杠命令、App Home 和交互功能方面具有同等能力。应根据部署形态而非功能进行选择。

| 考量事项                     | Socket Mode（默认）                                                                                                                                  | HTTP Request URLs                                                                                              |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| 公共 Gateway 网关 URL        | 不需要                                                                                                                                               | 需要（DNS、TLS、反向代理或隧道）                                                                               |
| 出站网络                     | 必须能够通过出站 WSS 访问 `wss-primary.slack.com`                                                                                                         | 无出站 WS；仅使用入站 HTTPS                                                                                    |
| 所需令牌                     | Bot 身份：bot token + 具有 `connections:write` 的 App-Level Token；用户身份：user token + App-Level Token                                             | Bot 身份：bot token + Signing Secret；用户身份：user token + Signing Secret                                    |
| 开发笔记本电脑 / 防火墙之后  | 可直接使用                                                                                                                                           | 需要公共隧道（ngrok、Cloudflare Tunnel、Tailscale Funnel）或预发布 Gateway 网关                                 |
| 水平扩展                     | 每台主机上的每个应用只能有一个 Socket Mode 会话；多个 Gateway 网关需要使用不同的 Slack 应用                                                          | 无状态 POST 处理程序；多个 Gateway 网关副本可通过负载均衡器共用一个应用                                        |
| 单个 Gateway 网关上的多账户  | 支持；每个账户会打开自己的 WS                                                                                                                        | 支持；每个账户需要唯一的 `webhookPath`（默认为 `/slack/events`），以免注册发生冲突                      |
| 斜杠命令传输                 | 通过 WS 连接传递；`slash_commands[].url` 会被忽略                                                                                                        | Slack 向 `slash_commands[].url` 发送 POST；命令分派要求提供此字段                                                  |
| 请求签名                     | 不使用（身份验证使用 App-Level Token）                                                                                                               | Slack 对每个请求进行签名；OpenClaw 使用 `signingSecret` 验证                                               |
| 连接断开后的恢复             | 已启用 Slack SDK 自动重连；OpenClaw 还会使用有界退避重启失败的 Socket Mode 会话。适用 Pong 超时传输调优。                                            | 没有可能断开的持久连接；由 Slack 对每个请求进行重试                                                            |

<Note>
  **单 Gateway 网关主机、开发笔记本电脑，以及能够通过出站连接访问 `*.slack.com` 但无法接受入站 HTTPS 的本地网络，应选择 Socket Mode**。

**在负载均衡器之后运行多个 Gateway 网关副本、出站 WSS 被阻止但允许入站 HTTPS，或已经通过反向代理终止 Slack Webhook 时，应选择 HTTP Request URLs**。
</Note>

<Warning>
  Slack 可以为一个应用维护多个 Socket Mode 连接，并可能将每个负载发送到任意连接。因此，共用一个 Slack 应用的不同 OpenClaw Gateway 网关需要采用等效的路由和授权配置。否则，请为每个 Gateway 网关使用不同的 Slack 应用、使用单一 Relay 入口，或在负载均衡器之后使用 HTTP Request URLs。请参阅[使用 Socket Mode](https://docs.slack.dev/apis/events-api/using-socket-mode#using-multiple-connections)。
</Warning>

### Relay 模式

Relay 模式将 Slack 入口与 OpenClaw Gateway 网关分离。可信路由器负责唯一的 Slack Socket Mode 连接、选择目标 Gateway 网关，并通过经过身份验证的 WebSocket 转发类型化事件。Gateway 网关仍使用自己的 bot token 发起出站 Slack Web API 调用。

```json5
{
  channels: {
    slack: {
      mode: "relay",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      relay: {
        url: "wss://router.example.com/gateway/ws",
        authToken: { source: "env", provider: "default", id: "SLACK_RELAY_AUTH_TOKEN" },
        gatewayId: "team-gateway",
      },
    },
  },
}
```

除非目标为 localhost，否则 Relay URL 必须使用 `wss://`。应将 bearer token 和路由器路由表视为 Slack 授权边界的一部分：路由的事件将作为已授权激活进入常规 Slack 消息处理程序。路由器在 WebSocket `hello` 帧中提供的 `slack_identity` 可以设置默认出站用户名和图标；调用方显式提供的身份仍然优先。Relay 连接采用与 Socket Mode 相同的有界退避时间进行重连，并在每次断开连接时清除路由器提供的身份。

### Enterprise Grid 全组织安装

一个 Slack 账户可以接收 Enterprise Grid 全组织安装所覆盖的每个工作区中的消息。请选择直接使用 Socket Mode 或 HTTP Request URLs；企业账户不支持 Relay 模式。以下两个最小权限清单都仅启用 V1 `message` 和 `app_mention` 事件路径、即时回复以及由监听器负责的状态表情回应。

#### Socket Mode

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "适用于 OpenClaw 的 Slack 连接器"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

请让 Enterprise Grid Org Admin 或 Org Owner 批准该应用、在组织级别安装该应用，并选择此安装覆盖的工作区。启动 OpenClaw 之前，请确认该应用在每个目标工作区中均可用。为 Socket Mode 生成具有 `connections:write` 的 app-level token，然后从组织安装中复制 bot token。配置使用组织安装 bot token 的账户：

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      enterpriseOrgInstall: true,
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

#### HTTP Request URLs

当 Gateway 网关具有公共 HTTPS 端点且不打开 Socket Mode 连接时，请使用 HTTP 模式。将示例 URL 替换为 Gateway 网关的公共 `webhookPath` URL（默认为 `/slack/events`）：

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "适用于 OpenClaw 的 Slack 连接器"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

请让 Enterprise Grid Org Admin 或 Org Owner 批准该应用、在组织级别安装该应用，并选择此安装覆盖的工作区。Slack 验证 Request URL 后，请复制组织安装的 bot token 以及应用的 **Basic Information -> App Credentials -> Signing Secret**。使用相同的 Request URL 路径配置企业账户：

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      enterpriseOrgInstall: true,
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: {
        source: "env",
        provider: "default",
        id: "SLACK_SIGNING_SECRET",
      },
      webhookPath: "/slack/events",
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

启动时，OpenClaw 会通过 Slack `auth.test` 验证 `enterpriseOrgInstall`。不带该标志的组织安装令牌或带有该标志的工作区令牌都会导致启动失败。Slack 仍是哪些工作区已授予此安装权限的事实来源；随后，OpenClaw 会将配置的频道、用户、私信和提及策略应用于每个传递的事件。无论 `allowBots` 如何配置，Enterprise V1 都会在分派前拒绝所有由 Bot 创建的 `message` 和 `app_mention` 事件，因为组织安装不会提供用于防止循环的稳定、工作区限定 Bot 身份。

企业支持有意限制为直接使用 Socket Mode 或 HTTP 的 `message` 和 `app_mention` 事件及其即时回复。企业账户无法使用 Relay 模式、斜杠命令、交互、App Home、表情回应事件监听器、置顶、Slack 操作工具、Slack 原生审批、绑定、排队或定时传递以及主动发送。通过监听器负责的 Slack 客户端支持出站确认、输入状态和状态表情回应，并且需要 `reactions:write`；入站表情回应通知和表情回应操作工具仍不可用。

即时回复会复用 Slack 的标准交付行为来处理分块、
媒体、元数据、身份回退、链接展开和回执，但仅限经过验证且由监听器拥有的客户端仍处于活跃事件轮次期间。内存中的发送队列和话题参与记录按该事件所属的工作区分区；客户端本身绝不会被序列化或持久化。

频道策略键和 `dm.groupChannels` 条目必须使用原始且稳定的 Slack 频道 ID，或
`channel:<id>` 形式。OpenClaw 会将任一形式规范化为原始频道 ID，以便在
运行时进行匹配；使用 `slack:`、`group:` 和 `mpim:` 前缀会导致启动失败。
用户策略条目必须使用稳定的 Slack 用户 ID；姓名、slug、显示名称和
电子邮件地址会导致启动失败。ID 必须使用 Slack 规范的大写
前缀和主体（例如 `C0123456789` 或 `U0123456789`）；小写形式和
简短的相似形式会导致启动失败。企业账户无法启用
`dangerouslyAllowNameMatching`。企业账户可以设置全局
`mentionPatterns.mode`，但 `mentionPatterns.allowIn` 和
`mentionPatterns.denyIn` 会导致启动失败，因为不带限定信息的 Slack 频道 ID 并未
限定到工作区，并且可能在不同工作区中重复使用。工作区安装
会保留现有的作用域内提及模式行为。每个获准的工作区
都会获得独立的路由、会话、转录记录、去重、历史记录和缓存标识，
即使 Slack ID 重叠也是如此。在 `message` 流中，支持普通用户消息
和用户发起的 `file_share` 事件；其他消息子类型会在
授权或系统事件处理之前被拒绝。

企业私信必须禁用（`dm.enabled=false` 或
`dmPolicy="disabled"`），或者通过 `dmPolicy="open"` 显式开放，并且
有效的账户 `allowFrom` 必须包含字面值 `"*"`。空的
允许列表或不含 `"*"` 的用户特定 ID 会导致启动失败。配对和
按用户设置的私信允许列表会被拒绝，因为这些授权存储中的 Slack 用户 ID
未限定到工作区。频道和发送者策略仍会应用于频道消息。

## 安装

```bash
openclaw plugins install @openclaw/slack
```

`plugins install` 会注册并启用该插件。在配置下方的 Slack 应用和频道设置之前，它不会执行任何操作。有关通用插件安装规则，请参阅[插件](/zh-CN/tools/plugin)。

## 快速设置

本节中的清单会创建作用于工作区范围的安装。对于
Enterprise Grid 组织级安装，请改用专门的
[组织级清单和工作流](#enterprise-grid-org-wide-installs)。

<Tabs>
  <Tab title="Socket 模式（默认）">
    <Steps>
      <Step title="创建新的 Slack 应用">
        打开 [api.slack.com/apps](https://api.slack.com/apps/new) → **Create New App** → **From a manifest** → 选择你的工作区 → 粘贴下方任一清单 → **Next** → **Create**。

        <CodeGroup>

```json Recommended
{
  "display_information": {
    "name": "OpenClaw",
    "description": "用于 OpenClaw 的 Slack 连接器"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw 将 Slack Agent View 对话连接到 OpenClaw 智能体。",
      "suggested_prompts": [
        { "title": "你能做什么？", "message": "你能为我提供什么帮助？" },
        {
          "title": "总结此频道",
          "message": "总结此频道最近的活动。"
        },
        { "title": "起草回复", "message": "帮我起草一条回复。" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "向 OpenClaw 发送消息",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "用于 OpenClaw 的 Slack 连接器"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw 将 Slack Agent View 对话连接到 OpenClaw 智能体。",
      "suggested_prompts": [
        { "title": "你能做什么？", "message": "你能为我提供什么帮助？" },
        {
          "title": "总结此频道",
          "message": "总结此频道最近的活动。"
        },
        { "title": "起草回复", "message": "帮我起草一条回复。" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "向 OpenClaw 发送消息",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    }
  }
}
```

        </CodeGroup>

        <Note>
          **推荐配置**与 Slack 插件的完整功能集相匹配：App Home、斜杠命令、文件、表情回应、置顶、群组私信以及表情符号/用户组读取。当工作区策略限制权限范围时，请选择**最小配置**——它涵盖私信、频道/群组历史记录、提及和斜杠命令，但不包括文件、表情回应、置顶、群组私信（`mpim:*`）、`emoji:read` 和 `usergroups:read`。有关各权限范围的理由以及额外斜杠命令等增量选项，请参阅[清单和权限范围检查清单](#manifest-and-scope-checklist)。
        </Note>

        Slack 创建应用后：

        - **Basic Information -> App-Level Tokens -> Generate Token and Scopes**：添加 `connections:write`，保存，然后复制 App-Level Token。
        - **Install App -> Install to Workspace**：复制 Bot User OAuth Token。

      </Step>

      <Step title="配置 OpenClaw">

        推荐的 SecretRef 设置：

```bash
export SLACK_APP_TOKEN=slack-app-token-example
export SLACK_BOT_TOKEN=slack-bot-token-example
cat > slack.socket.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./slack.socket.patch.json5 --dry-run
openclaw config patch --file ./slack.socket.patch.json5
```

        环境变量回退（仅限默认账户）：

```bash
SLACK_APP_TOKEN=slack-app-token-example
SLACK_BOT_TOKEN=slack-bot-token-example
```

      </Step>

      <Step title="启动 Gateway 网关">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="HTTP 请求 URL">
    <Steps>
      <Step title="创建新的 Slack 应用">
        打开 [api.slack.com/apps](https://api.slack.com/apps/new) → **Create New App** → **From a manifest** → 选择你的工作区 → 粘贴下方任一清单 → 将 `https://gateway-host.example.com/slack/events` 替换为你的公共 Gateway 网关 URL → **Next** → **Create**。

        <CodeGroup>

```json Recommended
{
  "display_information": {
    "name": "OpenClaw",
    "description": "用于 OpenClaw 的 Slack 连接器"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw 将 Slack Agent View 对话连接到 OpenClaw 智能体。",
      "suggested_prompts": [
        { "title": "你能做什么？", "message": "你能为我提供什么帮助？" },
        {
          "title": "总结此频道",
          "message": "总结此频道最近的活动。"
        },
        { "title": "起草回复", "message": "帮我起草一条回复。" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "向 OpenClaw 发送消息",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw 的 Slack 连接器"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw 将 Slack Agent View 对话连接到 OpenClaw 智能体。",
      "suggested_prompts": [
        { "title": "你能做什么？", "message": "你能帮我做什么？" },
        {
          "title": "总结此频道",
          "message": "总结此频道最近的活动。"
        },
        { "title": "起草回复", "message": "帮我起草一条回复。" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "向 OpenClaw 发送消息",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

        </CodeGroup>

        <Note>
          **推荐配置**与 Slack 插件的完整功能集一致；**最小配置**会为限制严格的工作区移除文件、表情回应、置顶、群组私信（`mpim:*`）、`emoji:read` 和 `usergroups:read`。有关各权限范围的理由，请参阅[清单和权限范围检查表](#manifest-and-scope-checklist)。
        </Note>

        <Info>
          这三个 URL 字段（`slash_commands[].url`、`event_subscriptions.request_url` 以及 `interactivity.request_url` / `message_menu_options_url`）都指向同一个 OpenClaw 端点。Slack 的清单架构要求分别命名这些字段，但 OpenClaw 会按负载类型路由，因此只需一个 `webhookPath`（默认值为 `/slack/events`）。在 HTTP 模式下，未设置 `slash_commands[].url` 的斜杠命令会静默地不执行任何操作。
        </Info>

        Slack 创建应用后：

        - **Basic Information → App Credentials**：复制用于验证请求的 **Signing Secret**。
        - **Install App -> Install to Workspace**：复制 Bot User OAuth Token。

      </Step>

      <Step title="配置 OpenClaw">

        推荐的 SecretRef 设置：

```bash
export SLACK_BOT_TOKEN=slack-bot-token-example
export SLACK_SIGNING_SECRET=...
cat > slack.http.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: { source: "env", provider: "default", id: "SLACK_SIGNING_SECRET" },
      webhookPath: "/slack/events",
    },
  },
}
JSON5
openclaw config patch --file ./slack.http.patch.json5 --dry-run
openclaw config patch --file ./slack.http.patch.json5
```

        <Note>
        为多账户 HTTP 使用唯一的 webhook 路径

        为每个账户分配不同的 `webhookPath`（默认值为 `/slack/events`），以免注册发生冲突。
        </Note>

      </Step>

      <Step title="启动 Gateway 网关">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## 用户身份（以真人身份发帖）

用户身份允许 OpenClaw 以授权 Slack 应用的用户身份读取和发帖。`userToken` 是执行操作的身份；配套的 Slack 应用通过 Socket Mode 或 HTTP Request URL 承载 Events API 流量。配套应用不需要机器人用户或机器人令牌。

按如下方式设置配套应用：

1. 在 **OAuth & Permissions -> User Token Scopes** 下，添加以下用户范围权限：

   - 历史记录：`channels:history`、`groups:history`、`im:history`、`mpim:history`
   - 对话查找：`channels:read`、`groups:read`、`im:read`、`mpim:read`
   - 人员：`users:read`
   - 发帖：`chat:write`（消息将以授权用户的身份发布）
   - 打开私信：`im:write`、`mpim:write`

2. 在 **Event Subscriptions -> Subscribe to events on behalf of users** 下，添加以下用户事件。不要仅将其添加到机器人事件列表：

   - `message.channels`
   - `message.groups`
   - `message.im`
   - `message.mpim`

3. 选择一种事件传输方式：

   - **Socket Mode：**启用 Socket Mode，并创建具有 `connections:write` 的应用级令牌。将其配置为 `appToken`。
   - **HTTP Request URL：**将 Event Subscriptions 指向公开的 OpenClaw Slack 端点，并复制 **Basic Information -> App Credentials -> Signing Secret**。将其配置为 `signingSecret`。

4. 安装或重新安装应用，以预期的用户身份授权，并将生成的用户 OAuth 令牌复制到 `userToken`。

Socket Mode 配置：

```json5
{
  channels: {
    slack: {
      identity: "user",
      userToken: "<xoxp>",
      appToken: "<xapp>",
    },
  },
}
```

HTTP Request URL 配置：

```json5
{
  channels: {
    slack: {
      identity: "user",
      mode: "http",
      userToken: "<xoxp>",
      signingSecret: "<signing-secret>",
      webhookPath: "/slack/events",
    },
  },
}
```

<Warning>
  私信和群组私信仅能通过上述用户范围事件订阅工作。机器人无法加入用户的 1:1 私信，也无法被添加到现有群组私信中。配套应用是不可见的底层设施：其他 Slack 成员看到的是授权用户发送的消息，而不是 OpenClaw 机器人发送的消息。
</Warning>

OpenClaw 会自动丢弃由解析出的用户身份所发送的用户范围消息事件，因此它发送的消息不会触发自我回复。

## Socket Mode 传输调优

对于 Socket Mode，OpenClaw 默认将 Slack SDK 客户端的 pong 超时设为 15 秒。仅在需要针对工作区或主机进行特定调优时覆盖传输设置：

```json5
{
  channels: {
    slack: {
      mode: "socket",
      socketMode: {
        clientPingTimeout: 20000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
    },
  },
}
```

仅在 Socket Mode 工作区记录 Slack websocket pong/server-ping 超时，或运行于已知存在事件循环饥饿问题的主机时使用此设置。`clientPingTimeout` 是 SDK 发送客户端 ping 后等待 pong 的时间；`serverPingTimeout` 是等待 Slack 服务器 ping 的时间。应用消息和事件仍属于应用状态，而非传输活跃性信号。

注意：

- `socketMode` 在 HTTP Request URL 模式下会被忽略。
- 除非被覆盖，否则基础 `channels.slack.socketMode` 设置适用于所有 Slack 账户。每个账户的覆盖设置使用 `channels.slack.accounts.<accountId>.socketMode`；由于这是对象覆盖，请包含该账户所需的所有 socket 调优字段。
- 只有 `clientPingTimeout` 具有 OpenClaw 默认值（`15000`）。仅在配置后，`serverPingTimeout` 和 `pingPongLoggingEnabled` 才会传递给 Slack SDK。
- Socket Mode 重启退避从约 2 秒开始，上限约为 30 秒。可恢复的启动、启动等待和断开连接故障会持续重试，直到渠道停止。无效身份验证、令牌被撤销或缺少权限范围等永久性账户和凭据错误会快速失败，而不会无限重试。

## 清单和权限范围检查表

Socket Mode 和 HTTP Request URL 使用相同的基础 Slack 应用清单。只有 `settings` 块（以及斜杠命令的 `url`）不同。

基础清单（默认为 Socket Mode）：

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw 的 Slack 连接器"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw 将 Slack Agent View 对话连接到 OpenClaw 智能体。",
      "suggested_prompts": [
        { "title": "你能做什么？", "message": "你能帮我做什么？" },
        {
          "title": "总结此频道",
          "message": "总结此频道最近的活动。"
        },
        { "title": "起草回复", "message": "帮我起草一条回复。" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "向 OpenClaw 发送消息",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

对于 **HTTP Request URLs 模式**，将 `settings` 替换为 HTTP 变体，并为每个斜杠命令添加 `url`。需要公开 URL：

```json
{
  "features": {
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "向 OpenClaw 发送消息",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

### 其他清单设置

提供不同的功能，以扩展上述默认设置。

默认清单会启用 Slack App Home 的 **Home** 标签页，并订阅 `app_home_opened`。当工作区成员打开 Home 标签页时，OpenClaw 会发布一个安全的默认 Home 视图，其中包含 `views.publish`；不会包含任何对话载荷或私有配置。启用单一斜杠命令模式时，命令提示使用 `channels.slack.slashCommand.name`；使用原生命令或不使用斜杠命令的安装会省略该提示。**Messages** 标签页仍会为 Slack 私信启用。新应用通过 `features.agent_view`、`assistant:write` 和 `app_context_changed` 使用 Slack Agent View。每个可见的 Agent View 根节点都会路由到其各自的 OpenClaw 线程会话，而 Slack 的有序活动视图实体只会作为不受信任的上下文传递给智能体。

已经使用 `features.assistant_view` 的现有应用可以保留当前清单。OpenClaw 会继续为这些安装处理 `assistant_thread_started` 和 `assistant_thread_context_changed`。Slack 不允许撤销从 Assistant View 到 Agent View 的迁移，并要求用户在迁移后执行硬刷新，因此在你打算迁移整个工作区之前，不要替换现有应用中的 `assistant_view`。

<AccordionGroup>
  <Accordion title="可选的原生斜杠命令">

    可以使用多个[原生斜杠命令](#commands-and-slash-behavior)来替代单个已配置的命令，但需注意：

    - 请使用 `/agentstatus` 而不是 `/status`，因为 `/status` 命令已被保留。
    - 一个 Slack 应用一次最多只能注册 25 个斜杠命令（Slack 平台限制）。

    OpenClaw 会为已启用的原生命令注册处理程序，但 Slack 清单条目仍由管理员管理，并且不会在运行时同步。请手动将 `/login` 添加到清单；为将命令数保持在 25 个，下面的示例包含该命令，而未包含可选的 `/side` 别名。`/login` 可以显示在任何位置，但仅会在私密聊天或 Web UI 中生成配对码。

    请将现有的 `features.slash_commands` 部分替换为[可用命令](/zh-CN/tools/slash-commands#command-list)的一个子集：

    <Tabs>
      <Tab title="Socket Mode（默认）">

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "开始新会话",
      "usage_hint": "[model]"
    },
    {
      "command": "/reset",
      "description": "重置当前会话"
    },
    {
      "command": "/compact",
      "description": "压缩会话上下文",
      "usage_hint": "[instructions]"
    },
    {
      "command": "/stop",
      "description": "停止当前运行"
    },
    {
      "command": "/session",
      "description": "管理线程绑定的过期时间",
      "usage_hint": "空闲 <duration|off> 或最大时长 <duration|off>"
    },
    {
      "command": "/think",
      "description": "设置思考级别",
      "usage_hint": "<level>"
    },
    {
      "command": "/verbose",
      "description": "切换详细输出",
      "usage_hint": "on|off|full"
    },
    {
      "command": "/fast",
      "description": "显示或设置快速模式",
      "usage_hint": "[status|on|off]"
    },
    {
      "command": "/reasoning",
      "description": "切换推理可见性",
      "usage_hint": "[on|off|stream]"
    },
    {
      "command": "/elevated",
      "description": "切换提升权限模式",
      "usage_hint": "[on|off|ask|full]"
    },
    {
      "command": "/exec",
      "description": "显示或设置 Exec 默认值",
      "usage_hint": "host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>"
    },
    {
      "command": "/approve",
      "description": "批准或拒绝待处理的审批请求",
      "usage_hint": "<id> <decision>"
    },
    {
      "command": "/model",
      "description": "显示或设置模型",
      "usage_hint": "[name|#|status]"
    },
    {
      "command": "/models",
      "description": "列出提供商/模型",
      "usage_hint": "[provider] [page] [limit=<n>|size=<n>|all]"
    },
    {
      "command": "/help",
      "description": "显示简短帮助摘要"
    },
    {
      "command": "/commands",
      "description": "显示生成的命令目录"
    },
    {
      "command": "/tools",
      "description": "显示当前智能体此刻可使用的内容",
      "usage_hint": "[compact|verbose]"
    },
    {
      "command": "/agentstatus",
      "description": "显示运行时状态，包括可用时的提供商用量/配额"
    },
    {
      "command": "/tasks",
      "description": "列出当前会话的活动/近期后台任务"
    },
    {
      "command": "/context",
      "description": "说明上下文的组装方式",
      "usage_hint": "[list|detail|json]"
    },
    {
      "command": "/whoami",
      "description": "显示你的发送者身份"
    },
    {
      "command": "/skill",
      "description": "按名称运行技能",
      "usage_hint": "<name> [input]"
    },
    {
      "command": "/btw",
      "description": "在不更改会话上下文的情况下询问旁支问题",
      "usage_hint": "<question>"
    },
    {
      "command": "/login",
      "description": "配对 Codex 登录",
      "usage_hint": "[codex|openai]"
    },
    {
      "command": "/usage",
      "description": "控制用量页脚或显示成本摘要",
      "usage_hint": "off|tokens|full|cost"
    }
  ]
}
```

      </Tab>
      <Tab title="HTTP 请求 URL">
        使用与上方 Socket Mode 相同的 `slash_commands` 列表，并向每个条目添加 `"url": "https://gateway-host.example.com/slack/events"`。示例：

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "开始新会话",
      "usage_hint": "[model]",
      "url": "https://gateway-host.example.com/slack/events"
    },
    {
      "command": "/help",
      "description": "显示简短帮助摘要",
      "url": "https://gateway-host.example.com/slack/events"
    }
  ]
}
```

        为列表中的每个命令重复使用该 `url` 值。

      </Tab>
    </Tabs>

  </Accordion>
  <Accordion title="可选的作者身份权限范围（写入操作）">
    如果你希望出站消息使用活动智能体身份（自定义用户名和图标），而不是默认的 Slack 应用身份，请添加 `chat:write.customize` Bot 权限范围。

    如果使用表情符号图标，Slack 要求采用 `:emoji_name:` 语法。

  </Accordion>
  <Accordion title="可选的用户令牌权限范围（读取操作）">
    如果配置了 `channels.slack.userToken`，典型的读取权限范围包括：

    - `channels:history`、`groups:history`、`im:history`、`mpim:history`
    - `channels:read`、`groups:read`、`im:read`、`mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read`（如果你依赖 Slack 搜索读取）

  </Accordion>
</AccordionGroup>

## 令牌模型

- Bot 身份（默认）在 Socket Mode 下需要 `botToken` + `appToken`，在 HTTP 模式下需要 `botToken` + `signingSecret`。
- 用户身份在 Socket Mode 下需要 `userToken` + `appToken`，在 HTTP 模式下需要 `userToken` + `signingSecret`。它不使用 Bot 令牌。
- 中继模式需要 `botToken`，以及 `relay.url`、`relay.authToken` 和 `relay.gatewayId`；它不使用应用令牌或签名密钥。
- `botToken`、`appToken`、`signingSecret`、`relay.authToken` 和 `userToken` 接受明文
  字符串或 SecretRef 对象。
- 配置中的令牌会覆盖环境变量回退值。
- `SLACK_BOT_TOKEN`、`SLACK_APP_TOKEN` 和 `SLACK_USER_TOKEN` 的环境变量回退值均仅适用于默认账户。
- `userToken` 默认为只读行为（`userTokenReadOnly: true`）。

状态快照行为：

- Slack 账户检查会按凭据跟踪 `*Source` 和 `*Status`
  字段（`botToken`、`appToken`、`signingSecret`、`userToken`）。
- 状态为 `available`、`configured_unavailable` 或 `missing`。
- `configured_unavailable` 表示账户通过 SecretRef
  或其他非内联密钥来源配置，但当前命令/运行时路径
  无法解析实际值。
- 在 HTTP 模式下，会包含 `signingSecretStatus`。Socket Mode 对 Bot 身份使用
  `botTokenStatus` + `appTokenStatus`，对用户身份使用
  `userTokenStatus` + `appTokenStatus`。

<Tip>
对于 Bot 身份，操作和目录读取可以优先使用可选的用户令牌；除非 `userTokenReadOnly: false` 允许回退，否则写入操作仍使用 Bot 令牌。对于 `identity: "user"`，读取和写入始终使用 `userToken`。
</Tip>

## 操作和门控

Slack 操作由 `channels.slack.actions.*` 控制。

当前 Slack 工具中可用的操作组：

| 组         | 默认值 |
| ---------- | ------- |
| messages   | 已启用 |
| reactions  | 已启用 |
| pins       | 已启用 |
| memberInfo | 已启用 |
| emojiList  | 已启用 |

当前 Slack 消息操作包括 `send`、`upload-file`、`download-file`、`read`、`edit`、`delete`、`pin`、`unpin`、`list-pins`、`member-info` 和 `emoji-list`。`download-file` 接受入站文件占位符中显示的 Slack 文件 ID，并为图像返回图像预览，为其他文件类型返回本地文件元数据。

## 访问控制和路由

<Tabs>
  <Tab title="私信策略">
    `channels.slack.dmPolicy` 控制私信访问。`channels.slack.allowFrom` 是规范的私信允许列表。

    - `pairing`（默认）
    - `allowlist`
    - `open`（要求 `channels.slack.allowFrom` 包含 `"*"`）
    - `disabled`

    私信标志：

    - `dm.enabled`（默认为 true）
    - `channels.slack.allowFrom`
    - `dm.allowFrom`（旧版）
    - `dm.groupEnabled`（群组私信默认为 false）
    - `dm.groupChannels`（可选的 MPIM 允许列表）

    多账户优先级：

    - `channels.slack.accounts.default.allowFrom` 仅适用于 `default` 账户。
    - 当命名账户自身未设置 `allowFrom` 时，会继承 `channels.slack.allowFrom`。
    - 命名账户不会继承 `channels.slack.accounts.default.allowFrom`。

    为了兼容，仍会读取旧版 `channels.slack.dm.policy` 和 `channels.slack.dm.allowFrom`。如果能够在不改变访问权限的情况下执行迁移，`openclaw doctor --fix` 会将它们迁移到 `dmPolicy` 和 `allowFrom`。

    私信中的配对使用 `openclaw pairing approve slack <code>`。

  </Tab>

  <Tab title="频道策略">
    `channels.slack.groupPolicy` 控制频道处理：

    - `open`
    - `allowlist`
    - `disabled`

    频道允许列表位于 `channels.slack.channels` 下，并且配置键**必须使用稳定的 Slack 频道 ID**（例如 `C12345678`）。

    运行时说明：如果完全缺少 `channels.slack`（仅使用环境变量的设置），运行时会回退到 `groupPolicy="allowlist"` 并记录警告（即使已设置 `channels.defaults.groupPolicy`）。

    名称/ID 解析：

    - 当令牌访问权限允许时，会在启动时解析频道允许列表条目和私信允许列表条目
    - 无法解析的频道名称条目会按配置保留，但默认在路由时忽略
    - 入站授权和频道路由默认优先使用 ID；直接按用户名/短名称匹配需要 `channels.slack.dangerouslyAllowNameMatching: true`

    <Warning>
    基于名称的键（`#channel-name` 或 `channel-name`）在 `groupPolicy: "allowlist"` 下**无法**匹配。默认情况下，渠道查找优先使用 ID，因此基于名称的键永远无法成功路由，该渠道中的所有消息都会被静默阻止。这与 `groupPolicy: "open"` 不同；在后者中，路由不要求渠道键，因此基于名称的键看起来可以正常工作。

    始终使用 Slack 渠道 ID 作为键。查找方法：在 Slack 中右键单击渠道 → **Copy link** — ID（`C...`）位于 URL 末尾。

    正确：

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            C12345678: { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```

    错误（在 `groupPolicy: "allowlist"` 下会被静默阻止）：

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            "#eng-my-channel": { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```
    </Warning>

  </Tab>

  <Tab title="提及和渠道用户">
    默认情况下，渠道消息受提及门控限制。

    提及来源：

    - 显式应用提及（`<@botId>`）
    - Slack 用户组提及（`<!subteam^S...>`），前提是 Bot 用户是该用户组的成员；需要 `usergroups:read`
    - 提及正则表达式模式（`agents.entries.*.groupChat.mentionPatterns`，回退为 `messages.groupChat.mentionPatterns`）
    - 对 Bot 自己的 Slack 消息的回复（`implicitMentions.replyToBot`）
    - Bot 参与过的线程中的后续消息（`implicitMentions.threadParticipation`）

    每渠道控制项（`channels.slack.channels.<id>`；名称仅可通过启动时解析或 `dangerouslyAllowNameMatching` 使用）：

    - `requireMention`
    - `ignoreOtherMentions`
    - `replyToMode`（`off|first|all|batched`；覆盖此渠道的账户/聊天类型回复模式）
    - `users`（允许列表）
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`、`toolsBySender`
    - `toolsBySender` 键格式：`channel:`、`id:`、`e164:`、`username:`、`name:`，或 `"*"` 通配符
      （旧版无前缀键仍仅映射到 `id:`）

    `ignoreOtherMentions`（默认值为 `false`）会丢弃提及其他用户或用户组、但未提及此 Bot 的渠道消息。私信和群组私信（MPIM）不受影响。该过滤器需要从 `auth.test` 解析出 Bot 用户 ID；如果无法获取该身份（例如只有用户令牌的身份），门控会以开放方式失败，消息将保持不变并继续传递。

    `allowBots` 对渠道和私有渠道采取保守策略：仅当发送消息的 Bot 明确列在该房间的 `users` 允许列表中，或者 `channels.slack.allowFrom` 中至少一个显式 Slack 所有者 ID 当前是房间成员时，才接受由 Bot 编写的房间消息。通配符和显示名称形式的所有者条目不能满足所有者在场条件。所有者在场检查使用 Slack `conversations.members`；请确保应用具有与房间类型匹配的读取权限范围（公共渠道使用 `channels:read`，私有渠道使用 `groups:read`）。如果成员查找失败，OpenClaw 会丢弃由 Bot 编写的房间消息。

    已接受的由 Bot 编写的 Slack 消息使用共享的 [Bot 循环保护](/zh-CN/channels/bot-loop-protection)。通过 `channels.defaults.botLoopProtection` 配置默认预算；当某个工作区或渠道需要不同限制时，再使用 `channels.slack.botLoopProtection` 或 `channels.slack.channels.<id>.botLoopProtection` 覆盖。

  </Tab>
</Tabs>

## 线程、会话和回复标签

- 私信路由为 `direct`；渠道路由为 `channel`；MPIM 路由为 `group`。
- Slack 路由绑定接受原始对等方 ID，以及 `channel:C12345678`、`user:U12345678` 和 `<@U12345678>` 等 Slack 目标形式。
- 使用默认的 `session.dmScope=main` 时，普通 Slack 私信会归并到智能体主会话。Agent View 根消息和现有 Assistant View 线程仍作为 `:thread:<threadTs>` 会话保持隔离。
- 渠道会话：`agent:<agentId>:slack:channel:<channelId>`。
- 普通的顶层渠道消息会保留在每渠道会话中，即使 `replyToMode` 为非 `off` 值。
- Slack 渠道、MPIM、Agent View 和 Assistant View 的线程回复使用父 Slack `thread_ts` 作为会话后缀（`:thread:<threadTs>`）。普通私信回复线程仍只是基础私信会话上的 UI 辅助功能。
- 当某个符合条件的顶层渠道根消息预计会启动可见的 Slack 线程时，OpenClaw 会将该根消息植入 `agent:<agentId>:slack:channel:<channelId>:thread:<rootTs>`，从而使根消息与后续线程回复共享一个 OpenClaw 会话。这适用于 `app_mention` 事件、显式 Bot 提及或已配置的提及模式匹配，以及 `replyToMode` 为非 `off` 值的 `requireMention: false` 渠道。
- `channels.slack.thread.historyScope` 的默认值为 `thread`；`thread.inheritParent` 的默认值为 `false`。
- `channels.slack.thread.initialHistoryLimit` 控制新线程会话启动时获取多少条现有线程消息（默认值为 `20`；设为 `0` 可禁用）。
- `channels.slack.implicitMentions.replyToBot` 控制对 Bot 自己消息的回复是否绕过提及门控（默认值为 `true`）。
- `channels.slack.implicitMentions.threadParticipation` 控制 Bot 已回复过的线程中的后续消息是否绕过提及门控（默认值为 `true`）。将其设为 `false`，可要求这些后续消息中出现新的显式提及。`openclaw doctor --fix` 会将原来的 `channels.slack.thread.requireExplicitMention` 键迁移为这个正向规范标志。
- 账户级覆盖项位于 `channels.slack.accounts.<id>.implicitMentions`；共享默认值位于 `channels.defaults.implicitMentions`。

回复线程控制项：

- `channels.slack.channels.<id>.replyToMode`：Slack 渠道/私有渠道消息的每渠道覆盖项
- `channels.slack.replyToMode`：`off|first|all|batched`（默认值为 `off`）
- `channels.slack.replyToModeByChatType`：按 `direct|group|channel` 配置
- 直接聊天的旧版回退项：`channels.slack.dm.replyToMode`

支持手动回复标签：

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

对于来自 `message` 工具的显式 Slack 线程回复，请设置 `replyBroadcast: true`，并搭配 `action: "send"` 和 `threadId` 或 `replyTo`，要求 Slack 同时将线程回复广播到父渠道。这会映射到 Slack 的 `chat.postMessage` `reply_broadcast` 标志，并且仅支持文本或 Block Kit 发送，不支持媒体上传。

当 `message` 工具调用在 Slack 线程内运行并以同一渠道为目标时，OpenClaw 通常会根据有效的账户级、聊天类型级或每渠道 `replyToMode` 继承当前 Slack 线程。自动回复以及同渠道的 `send` 或 `upload-file` 调用使用相同的每渠道覆盖项。在 `action: "send"` 或 `action: "upload-file"` 上设置 `topLevel: true`，可强制发送新的父渠道消息。`threadId: null` 也被视为相同的顶层退出选项。

<Note>
`replyToMode="off"` 会禁用可选的 Slack 出站回复线程功能，包括显式 `[[reply_to_*]]` 标签。Agent View 和 Assistant View 是由 Slack 管理的线程式体验，因此无论此设置如何，它们的回复和状态都会保留在可见的根消息上。此设置不会将其他入站 Slack 线程会话扁平化。这与 Telegram 不同；在 Telegram 的 `"off"` 模式下，显式标签仍会生效。Slack 线程会向渠道隐藏消息，而 Telegram 回复仍以内联方式显示。
</Note>

## 确认表情回应

`ackReaction` 会在 OpenClaw 处理入站消息时发送一个确认表情符号。`ackReactionScope` 决定该表情符号实际发送的_时机_。

默认情况下，确认表情回应保持不变，而 Slack 原生智能体/助手线程状态会通过轮换的加载消息显示进度。设置 `messages.statusReactions.enabled: true` 可启用排队/思考/工具/完成/错误的表情回应生命周期。

### 表情符号（`ackReaction`）

解析顺序：

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- 智能体身份表情符号回退项（`agents.entries.*.identity.emoji`，否则为 `"eyes"` / 👀）

注意：

- Slack 要求使用短代码（例如 `"eyes"`）。
- 使用 `""` 可为 Slack 账户或全局禁用该表情回应。

### 范围（`messages.ackReactionScope`）

Slack provider 从 `messages.ackReactionScope` 读取范围（默认值为 `"group-mentions"`）。目前没有 Slack 账户级或 Slack 渠道级覆盖项；该值对 Gateway 网关全局生效。

值：

- `"all"`：在私信和群组中添加表情回应，包括环境房间事件。
- `"direct"`：仅在私信中添加表情回应。
- `"group-all"`：对除环境房间事件以外的每条群组消息添加表情回应（不包括私信）。
- `"group-mentions"`（默认值）：在群组中添加表情回应，但仅限 Bot 被提及时（或在已选择启用的群组可提及对象中）。**不包括私信。**
- `"off"` / `"none"`：从不添加表情回应。

<Note>
默认范围（`"group-mentions"`）不会在私信或环境房间事件中触发确认表情回应。若要在入站 Slack 私信和安静的房间事件中看到已配置的 `ackReaction`（例如 `"eyes"`），请将 `messages.ackReactionScope` 设置为 `"all"`。`messages.ackReactionScope` 会在 Slack provider 启动时读取，因此需要重启 Gateway 网关才能使更改生效。
</Note>

```json5
{
  messages: {
    ackReaction: "eyes",
    ackReactionScope: "all", // 在私信和群组中添加表情回应
  },
}
```

## 文本流式传输

`channels.slack.streaming` 控制实时预览行为：

- `off`：禁用实时预览流式传输。
- `partial`（默认值）：使用最新的部分输出替换预览文本。
- `block`：追加分块预览更新。
- `progress`：生成期间显示进度状态文本，然后发送最终文本。
- `streaming.preview.toolProgress`：启用草稿预览时，将工具/进度更新路由到同一条经过编辑的预览消息中（默认值：`true`）。设置 `false` 可保留单独的工具/进度消息。
- `streaming.preview.commandText` / `streaming.progress.commandText`：设为 `status` 可在隐藏原始命令/Exec 文本的同时保留紧凑的工具进度行（默认值：`raw`）。

隐藏原始命令/Exec 文本，同时保留紧凑的进度行：

```json
{
  "channels": {
    "slack": {
      "streaming": {
        "mode": "progress",
        "progress": {
          "toolProgress": true,
          "commandText": "status"
        }
      }
    }
  }
}
```

当 `channels.slack.streaming.mode` 为 `partial` 时，`channels.slack.streaming.nativeTransport` 控制 Slack 原生文本流式传输（默认值：`true`）。

Slack 原生进度任务卡在进度模式下需选择启用。将 `channels.slack.streaming.progress.nativeTaskCards` 设置为 `true` 并搭配 `channels.slack.streaming.mode="progress"`，可在工作运行期间发送 Slack 原生计划/任务卡，并在完成时更新同一张任务卡。如果未设置此标志，进度模式会继续使用可移植的草稿预览行为。

- 必须有回复线程，原生文本流式传输和 Slack 助手线程状态才会显示。线程选择仍遵循 `replyToMode`。
- 当原生流式传输不可用或不存在回复线程时，频道、群聊和顶层私信根消息仍可使用常规草稿预览。
- 顶层 Slack 私信默认不使用线程，因此不会显示 Slack 的线程式原生流式传输/状态预览；OpenClaw 会改为在私信中发布并编辑草稿预览。
- 媒体和非文本载荷会回退到常规投递。
- 媒体/错误最终载荷会取消待处理的预览编辑；符合条件的文本/分块最终载荷仅在可以原地编辑预览时才会刷新。
- 如果流式传输在回复过程中失败，OpenClaw 会对剩余载荷回退到常规投递。

使用草稿预览，而不是 Slack 原生文本流式传输：

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "partial",
        nativeTransport: false,
      },
    },
  },
}
```

选择启用 Slack 原生进度任务卡片：

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "progress",
        progress: {
          nativeTaskCards: true,
          render: "rich",
        },
      },
    },
  },
}
```

旧版键：

- `channels.slack.streamMode`（`replace | status_final | append`）是 `channels.slack.streaming.mode` 的旧版别名。
- 布尔值 `channels.slack.streaming` 是 `channels.slack.streaming.mode` 和 `channels.slack.streaming.nativeTransport` 的旧版别名。
- 顶层 `channels.slack.chunkMode` 和 `channels.slack.nativeStreaming` 是 `channels.slack.streaming.chunkMode` 和 `channels.slack.streaming.nativeTransport` 的旧版别名。
- 运行时不会读取旧版别名；请运行 `openclaw doctor --fix`，将持久化的 Slack 流式传输配置重写为规范键。

## 输入状态表情回应回退

`typingReaction` 会在 OpenClaw 处理回复期间，为入站 Slack 消息添加临时表情回应，并在运行结束时将其移除。这最适合在线程回复之外使用；线程回复默认使用“正在输入...”状态指示器。

解析顺序：

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

注意：

- Slack 要求使用短代码（例如 `"hourglass_flowing_sand"`）。
- 表情回应采用尽力而为的方式，并会在回复或失败路径完成后自动尝试清理。

## 语音输入

目前要在 Slack 中通过语音与 OpenClaw 交互，请向 OpenClaw 应用发送 Slack 音频剪辑。Slackbot 的听写麦克风是 Slack 自有的独立功能，并非应用 API。

- **[Slackbot 语音听写](https://slack.com/help/articles/202026038-How-to-use-Slackbot)**位于用户的私有 Slackbot 对话中。Slack 会将录音转换为 Slackbot 提示词，但不会通过 Events API 向第三方 Slack 应用发送音频文件、听写事件、提示词或输入来源标记。OpenClaw Slack 插件无法启用或接收该功能。
- **[Slack 音频剪辑](https://slack.com/help/articles/4406235165587-Record-audio-and-video-clips-in-Slack)**是存储在 Slack 中的文件，可以发布到 OpenClaw 私信、频道或线程中。OpenClaw 使用 Bot 令牌下载可访问的剪辑，规范化 Slack 的剪辑 MIME 元数据，然后将其发送至共享的[音频转录流水线](/zh-CN/nodes/audio)。推荐的应用清单包含必需的 `files:read` 权限范围。

音频剪辑与 Slackbot 听写具有不同的隐私语义：剪辑遵循 Slack 文件保留策略，OpenClaw 会下载剪辑以进行转录；而 Slack 表示不会存储听写音频。

在启用了 `requireMention: true` 的频道中，无字幕的音频剪辑可以通过说出已配置的提及模式（`agents.entries.*.groupChat.mentionPatterns`，回退到 `messages.groupChat.mentionPatterns`）来满足门控条件。OpenClaw 会先授权发送者，再下载或转录剪辑，并且仅当转录文本匹配时才接纳该消息。失败或不匹配的推测性转录文本会连同下载的剪辑一起丢弃；不会将其保留在频道历史记录中。无法根据语音推断原生 Slack `@bot` 身份，因此请配置口述名称模式或添加文本提及。如果启用了转录文本回显，则仅在消息被接纳后发送回显。

## 媒体、分块和投递

<AccordionGroup>
  <Accordion title="入站附件">
    Slack 文件附件会从 Slack 托管的私有 URL 下载（使用令牌身份验证的请求流程），并在获取成功且大小限制允许时写入媒体存储。文件占位符包含 Slack `fileId`，以便智能体使用 `download-file` 获取原始文件。

    下载使用有界的空闲超时和总超时。如果 Slack 文件检索停滞或失败，OpenClaw 会继续处理消息，并回退到文件占位符。

    运行时入站大小上限默认为 `20MB`，除非被 `channels.slack.mediaMaxMb` 覆盖。

  </Accordion>

  <Accordion title="出站文本和文件">
    - 文本块使用 `channels.slack.textChunkLimit`（默认值为 `8000`，上限为 Slack 自身的消息长度限制）
    - `channels.slack.streaming.chunkMode="newline"` 启用段落优先拆分
    - 文件发送使用 Slack 上传 API，并且可以包含线程回复（`thread_ts`）
    - 较长的文件说明使用第一个符合 Slack 安全限制的文本块作为上传评论，并将其余文本块作为后续消息发送
    - 配置后，出站媒体上限遵循 `channels.slack.mediaMaxMb`；否则，频道发送使用媒体流水线基于 MIME 类型的默认值

  </Accordion>

  <Accordion title="投递目标">
    首选的显式目标：

    - `user:<id>` 用于私信
    - `channel:<id>` 用于频道

    仅含文本/分块的 Slack 私信可以直接发布到用户 ID；文件上传和线程发送会先通过 Slack 对话 API 打开私信，因为这些路径需要具体的对话 ID。

  </Accordion>
</AccordionGroup>

## 命令和斜杠行为

斜杠命令在 Slack 中显示为单个已配置命令或多个原生命令。配置 `channels.slack.slashCommand` 可更改命令默认值：

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

```txt
/openclaw /help
```

原生命令要求在 Slack 应用中配置[额外的清单设置](#additional-manifest-settings)，并改为通过全局配置中的 `channels.slack.commands.native: true` 或 `commands.native: true` 启用。

- Slack 的原生命令自动模式为**关闭**，因此 `commands.native: "auto"` 不会启用 Slack 原生命令。

```txt
/help
```

原生参数菜单按以下优先级顺序呈现为其中一种形式：

- 3-5 个足够短的选项：溢出（“...”）菜单
- 超过 100 个选项，并且支持异步选项筛选：外部选择器
- 1-2 个选项，或者任一选项的编码值过长、无法用于选择器：按钮块
- 其他情况（6-100 个选项，或超过 100 个选项但不支持异步筛选）：静态选择菜单，每个菜单按 100 个选项分块

```txt
/think
```

斜杠命令会话使用类似 `agent:<agentId>:slack:slash:<userId>` 的隔离键，并仍使用 `CommandTargetSessionKey` 将命令执行路由到目标对话会话。

## 原生图表

Slack 的公共 [`data_visualization` Block Kit 块](https://docs.slack.dev/reference/block-kit/blocks/data-visualization-block/)
可在消息中呈现折线图、柱状图、面积图和饼图。OpenClaw 将可移植的
`presentation` `chart` 块映射到该原生结构；除正常的
`chat:write` 消息访问权限外，无需额外的 OAuth 权限范围、
文件上传、图像渲染器或 Slack 配置。

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "bar",
      "title": "季度收入",
      "categories": ["第一季度", "第二季度"],
      "series": [{ "name": "收入", "values": [120, 145] }],
      "xLabel": "季度"
    }
  ]
}
```

原生呈现前会强制执行 Slack 的限制：

- 标题和可选的坐标轴标签：50 个字符
- 饼图：1-12 个正值扇区
- 折线图/柱状图/面积图：1-12 个名称唯一的系列，以及 1-20 个共享类别
- 扇区、类别和系列标签：20 个字符
- 每个系列必须为每个类别包含一个有限数值；非饼图数值
  可以为负数

每个原生图表还包含顶层文本表示，供屏幕阅读器、通知、会话镜像以及无法呈现该块的客户端使用。发送到其他 OpenClaw 频道的标准呈现内容也会以文本形式接收相同的确定性图表数据，除非这些频道声明支持原生图表。如果 Slack 在分阶段推出期间以 `invalid_blocks` 拒绝图表，OpenClaw 会移除被拒绝的原生数据块、保留所有同级控件，并将完整的图表表示作为可见文本发送。

Slack 目前每条消息最多接受两个 `data_visualization` 块。当呈现内容包含两个以上有效图表时，OpenClaw 会保持其顺序，并在后续消息中继续进行原生呈现，每条消息最多包含两个图表。

Slack 的[开发者发布说明](https://docs.slack.dev/changelog/2026/06/16/block-kit-data-visualization-block/)
将该块记录为面向应用的 Block Kit 功能，并且未公布任何付费套餐限制。Business+/Enterprise 资格说明适用于 Slackbot 的自动 AI 图表生成，与应用发送已经结构化的 Block Kit 图表相互独立。图表是仅限消息使用的块，不适用于 App Home、模态窗口或 Canvas 内容。

## 原生表格

Slack 当前的 [`data_table` Block Kit 块](https://docs.slack.dev/reference/block-kit/blocks/data-table-block/)
可在消息中呈现结构化行和列。OpenClaw 将显式的
可移植 `presentation` `table` 块映射到 `data_table`；它不使用 Slack 的
旧版 [`table` 块](https://docs.slack.dev/reference/block-kit/blocks/table-block/)。
除正常的 `chat:write` 消息访问权限外，无需额外的 OAuth 权限范围或 Slack 配置。

```json
{
  "blocks": [
    {
      "type": "table",
      "caption": "进行中的销售管线",
      "headers": ["客户", "阶段", "ARR"],
      "rows": [
        ["Acme", "已赢单", 125000],
        ["Globex", "审核", 82000]
      ],
      "rowHeaderColumnIndex": 0
    }
  ]
}
```

OpenClaw 将表头单元格和字符串单元格映射为 Slack `raw_text` 单元格。数值单元格
映射为 `raw_number`，并保留有限数值，以支持原生排序
和筛选。存在 `rowHeaderColumnIndex` 时，它会将该从零开始编号的
列标记为 Slack 行标题。

原生呈现前会强制执行 Slack 公布的 `data_table` 限制：

- 1-20 列
- 1-100 个数据行，另加表头行
- 每行的单元格数量相同
- 一条消息中所有表格单元格的字符总数最多为 10,000

只要消息仍在字符总数限制内，多个有效表格块即可进行原生呈现。无法在原生范围内呈现的表格会转换为完整的确定性文本，而不会丢失行或单元格。如果该文本超过一条 Slack 消息的容量，发送操作和斜杠命令响应将使用有序文本块。表格编辑会返回明确的大小错误，而不会从现有消息中静默截断行。

从可移植呈现生成的每个原生表格还会携带顶层文本表示，供屏幕阅读器、通知、会话镜像以及无法渲染该块的客户端使用。原始图表和表格值在后备内容中保持字面形式，因此像 `<@U123>` 这样的单元格数据不会变成 Slack 提及。如果 Slack 因 `invalid_blocks` 拒绝原生图表或表格块，OpenClaw 会在一次有界恢复步骤中移除所有原生数据块，保留按钮和选择器等有效的同级块，并在禁用 Slack 格式的情况下发送完整可见的图表和表格文本。斜杠命令投递会在整个命令执行期间跟踪 Slack 的五次调用 `response_url` 预算。每批回复前，它会选择一个适合剩余调用次数的完整方案，或者在发布该批回复前失败。

只有显式的 `presentation` 表格块会被提升为原生表格。Markdown 管道表格仍为编写的文本；OpenClaw 不会猜测表格结构或单元格类型。现有受信任的 Slack 原生内容生成方可以继续通过 `channelData.slack.blocks` 传递原始块；OpenClaw 会从有效的原始 `data_table` 单元格派生后备文本，而格式错误的自定义块可能会降级为其标题或通用 Block Kit 后备内容。可移植的智能体、CLI 和插件输出应使用 `presentation`。

## 交互式回复

Slack 可以渲染由智能体编写的交互式回复控件，但默认禁用此功能。
对于新的智能体、CLI 和插件输出，优先使用共享的
`presentation` 按钮或选择块。它们使用相同的 Slack 交互路径，同时也能在其他渠道上降级。

全局启用：

```json5
{
  channels: {
    slack: {
      capabilities: {
        interactiveReplies: true,
      },
    },
  },
}
```

或者仅为一个 Slack 账户启用：

```json5
{
  channels: {
    slack: {
      accounts: {
        ops: {
          capabilities: {
            interactiveReplies: true,
          },
        },
      },
    },
  },
}
```

启用后，智能体仍可发出已弃用的 Slack 专用回复指令：

- `[[slack_buttons: Approve:approve, Reject:reject]]`
- `[[slack_select: Choose a target | Canary:canary, Production:production]]`

这些指令会编译为 Slack Block Kit，并通过现有的 Slack 交互事件路径将点击或选择路由回来。为旧提示词和 Slack 专用的应急通道保留它们；新建可移植控件时请使用共享呈现方式。

对于新的内容生成方代码，指令编译器 API 也已弃用：

- `compileSlackInteractiveReplies(...)`
- `parseSlackOptionsLine(...)`
- `isSlackInteractiveRepliesEnabled(...)`
- `buildSlackInteractiveBlocks(...)`

新的 Slack 渲染控件请使用 `presentation` 载荷和 `buildSlackPresentationBlocks(...)`。

注意：

- 这是 Slack 专用的旧版 UI。其他渠道不会将 Slack Block
  Kit 指令转换为各自的按钮系统。
- 交互回调值是由 OpenClaw 生成的不透明令牌，而不是智能体编写的原始值。
- 如果生成的交互块会超出 Slack Block Kit 限制，OpenClaw 会回退到原始文本回复，而不会发送无效的块载荷。

### 插件所有的模态框提交

注册了交互处理程序的 Slack 插件还可以在 OpenClaw 为智能体可见的系统事件压缩载荷之前，接收模态框
`view_submission` 和 `view_closed` 生命周期事件。打开 Slack 模态框时，请使用以下路由模式之一：

- 将 `callback_id` 设置为 `openclaw:<namespace>:<payload>`。
- 或者保留现有的 `callback_id`，并将 `pluginInteractiveData:
"<namespace>:<payload>"` 放入模态框的 `private_metadata` 中。

处理程序会接收作为 `view_submission` 或
`view_closed` 的 `ctx.interaction.kind`、规范化的 `inputs`，以及来自 Slack 的完整原始 `stateValues` 对象。仅按回调 ID 路由就足以调用插件处理程序；当模态框还应生成智能体可见的系统事件时，请包含现有模态框的 `private_metadata` 用户/会话路由字段。智能体会收到经过压缩和脱敏的 `Slack interaction: ...` 系统事件。如果处理程序返回
`systemEvent.summary`、`systemEvent.reference` 或 `systemEvent.data`，这些字段将包含在该压缩事件中，使智能体无需查看完整表单载荷即可引用插件所有的存储。

## Slack 中的原生审批

Slack 可以通过交互式按钮和交互操作充当原生审批客户端，而无需回退到 Web UI 或终端。

- Exec 和插件审批可以渲染为 Slack 原生 Block Kit 提示。
- `channels.slack.execApprovals.*` 仍是原生 Exec 审批客户端的启用及私信/频道路由配置。
- Exec 审批私信使用 `channels.slack.execApprovals.approvers` 或 `commands.ownerAllowFrom`。
- 当 Slack 被启用为发起会话的原生审批客户端，或者 `approvals.plugin` 路由到发起的 Slack 会话或 Slack 目标时，插件审批会使用 Slack 原生按钮。
- 插件审批私信使用 `channels.slack.allowFrom` 中的 Slack 插件审批者、具名账户的 `allowFrom`，或账户默认路由。
- 仍会强制执行审批者授权：仅有 Exec 审批权限的审批者无法批准插件请求，除非他们同时也是插件审批者。

此功能使用与其他渠道相同的共享审批按钮界面。在 Slack 应用设置中启用 `interactivity` 后，审批提示会直接以 Block Kit 按钮呈现在对话中。
当这些按钮存在时，它们是主要审批体验；仅当工具结果表明聊天审批不可用或手动审批是唯一路径时，OpenClaw 才应包含手动 `/approve` 命令。

配置路径：

- `channels.slack.execApprovals.enabled`
- `channels.slack.execApprovals.approvers`（可选；可能时回退到 `commands.ownerAllowFrom`）
- `channels.slack.execApprovals.target`（`dm` | `channel` | `both`，默认值：`dm`）
- `agentFilter`、`sessionFilter`

当 `enabled` 未设置或为 `"auto"`，且至少解析出一名 Exec 审批者时，Slack 会自动启用原生 Exec 审批。当 Slack 插件审批者解析成功且请求符合原生客户端筛选条件时，Slack 也可以通过此原生客户端路径处理原生插件审批。将 `enabled: false` 设置为显式禁用 Slack 作为原生审批客户端。将 `enabled: true` 设置为在解析出审批者时强制启用原生审批。禁用 Slack Exec 审批不会禁用通过 `approvals.plugin` 启用的原生 Slack 插件审批投递；插件审批投递改用 Slack 插件审批者。

未显式配置 Slack Exec 审批时的默认行为：

```json5
{
  commands: {
    ownerAllowFrom: ["slack:U12345678"],
  },
}
```

仅当需要覆盖审批者、添加筛选条件或选择加入原始聊天投递时，才需要显式配置 Slack 原生功能：

```json5
{
  channels: {
    slack: {
      execApprovals: {
        enabled: true,
        approvers: ["U12345678"],
        target: "both",
      },
    },
  },
}
```

共享的 `approvals.exec` 转发是独立功能。仅当 Exec 审批提示还必须路由到其他聊天或显式的带外目标时才使用它。共享的 `approvals.plugin` 转发也是独立功能；仅当 Slack 可以原生处理插件审批请求时，Slack 原生投递才会抑制该后备方式。

同一聊天中的 `/approve` 也适用于已经支持命令的 Slack 频道和私信。有关完整的审批转发模型，请参阅 [Exec 审批](/zh-CN/tools/exec-approvals)。

## 事件和运行行为

- 消息编辑/删除会映射为系统事件。
- 线程广播（“Also send to channel”线程回复）会作为普通用户消息处理。
- 添加/移除表情回应事件会映射为系统事件。
- 成员加入/离开、频道创建/重命名以及添加/移除置顶事件会映射为系统事件。
- 可选的在线状态轮询可以将观察到的人类参与者从 `away` 到 `active` 的转换，映射到该参与者最近活跃且符合条件的 Slack 会话。默认关闭。
- 启用 `configWrites` 后，`channel_id_changed` 可以迁移频道配置键。
- 频道主题/用途元数据会被视为不受信任的上下文，并可注入路由上下文。
- Agent View 的 `app_context` 实体按 Slack 相关性顺序进行验证，并且仅作为结构化的不受信任上下文公开；省略的上下文会清除当前轮次，而不会复用过时实体。
- 适用时，线程起始消息和初始线程历史上下文播种会按配置的发送者允许列表进行筛选。
- 块操作、快捷方式和模态框交互会发出结构化的 `Slack interaction: ...` 系统事件，其中包含丰富的载荷字段：
  - 块操作：选定值、标签、选择器值和 `workflow_*` 元数据
  - 全局快捷方式：回调和操作者元数据，路由到操作者的直接会话
  - 消息快捷方式：回调、操作者、频道、线程和所选消息上下文
  - 模态框 `view_submission` 和 `view_closed` 事件，包含已路由的频道元数据和表单输入

在 Slack 应用配置中定义全局或消息快捷方式，并使用任意非空回调 ID。OpenClaw 会确认匹配的快捷方式载荷，应用与其他 Slack 交互相同的私信/频道发送者策略，并将经过清理的事件排入已路由智能体会话的队列。触发器 ID 和响应 URL 会从智能体上下文中脱敏。

### 在线状态事件

Slack 不会通过 Events API 或 Socket Mode 发送在线状态变化。OpenClaw 可以改为针对消息已通过常规 Slack 访问和路由检查的人类参与者轮询 [`users.getPresence`](https://docs.slack.dev/reference/methods/users.getPresence/)。

```json5
{
  channels: {
    slack: {
      presenceEvents: { mode: "auto" },
      channels: {
        C0123456789: { presenceEvents: { mode: "on" } },
        C0987654321: { presenceEvents: { mode: "off" } },
      },
    },
  },
}
```

- `off`（默认值）：无在线状态计时器，也不调用 Slack API。
- `auto`：监控过去 24 小时内活跃的私信、MPIM 和 Slack 线程，最多观察 8 名人类参与者。不包括顶层频道会话。
- `on`：监控相同的对话，但不限制参与者数量，并包括顶层频道会话。使用按频道覆盖配置来强制启用或抑制某个频道。

OpenClaw 每个 Slack 账户每分钟最多轮询 45 个不同用户，首次结果仅用于初始化，不会唤醒智能体，并且仅在观察到从 `away` 到 `active` 的转换时唤醒。每个 Slack 账户和用户都适用持久的 8 小时冷却时间，即使该用户参与了多个线程也是如此。事件只会路由到该用户最近活跃且符合条件的对话，并告知智能体在决定是否发送一句简短问候语之前查阅记忆/wiki 和已知的时区上下文。智能体可以保持沉默。

Bot 令牌需要 `users:read`，推荐的清单中已包含它。Enterprise Grid 组织范围安装不支持在线状态事件。

## 配置参考

主要参考：[Configuration reference - Slack](/zh-CN/gateway/config-channels#slack)。

<Accordion title="重要 Slack 字段">

- 模式/身份验证：`identity`、`mode`、`enterpriseOrgInstall`、`botToken`、`appToken`、`userToken`、`signingSecret`、`webhookPath`、`accounts.*`
- 私信访问：`dm.enabled`、`dmPolicy`、`allowFrom`（旧版：`dm.policy`、`dm.allowFrom`）、`dm.groupEnabled`、`dm.groupChannels`
- 兼容性开关：`dangerouslyAllowNameMatching`（紧急解锁；除非需要，否则保持关闭）
- 频道访问：`groupPolicy`、`channels.*`、`channels.*.users`、`channels.*.requireMention`、`implicitMentions.*`
- 线程/历史记录：`replyToMode`、`replyToModeByChatType`、`thread.*`、`historyLimit`、`dmHistoryLimit`、`dms.*.historyLimit`
- 在线状态唤醒：`presenceEvents.mode`、`channels.*.presenceEvents.mode`（`off|auto|on`；默认值为 `off`）
- 投递：`textChunkLimit`、`streaming.chunkMode`、`mediaMaxMb`、`streaming`、`streaming.nativeTransport`、`streaming.preview.toolProgress`
- 展开预览：`unfurlLinks`（默认值：`false`），使用 `unfurlMedia` 控制 `chat.postMessage` 链接/媒体预览；将 `unfurlLinks: true` 设为相应值可重新启用链接预览
- 运维/功能：`configWrites`、`commands.native`、`slashCommand.*`、`actions.*`、`userToken`、`userTokenReadOnly`

</Accordion>

## 故障排查

<AccordionGroup>
  <Accordion title="频道中没有回复">
    按顺序检查：

    - `groupPolicy`
    - 频道允许列表（`channels.slack.channels`）— **键必须是频道 ID**（`C12345678`），不能是名称（`#channel-name`）。由于频道路由默认优先使用 ID，基于名称的键在 `groupPolicy: "allowlist"` 下会静默失效。要查找 ID：在 Slack 中右键单击频道 → **Copy link** — URL 末尾的 `C...` 值就是频道 ID。
    - `requireMention`
    - 每频道 `users` 允许列表
    - `messages.groupChat.visibleReplies`：普通群组/频道请求默认为 `"automatic"`。如果你选择启用 `"message_tool"`，且日志显示有助手文本但没有 `message(action=send)` 调用，则表示模型未使用可见的消息工具路径。在此模式下，最终文本会保持私密；请检查 Gateway 网关详细日志中的已抑制载荷元数据，或者如果希望通过旧版路径发布每条普通助手最终回复，请将其设为 `"automatic"`。
    - `messages.groupChat.unmentionedInbound`：如果其值为 `"room_event"`，允许频道中未提及智能体的聊天内容会作为环境上下文，并保持静默，除非智能体调用 `message` 工具。请参阅[环境房间事件](/zh-CN/channels/ambient-room-events)。

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "automatic",
    },
  },
}
```

    常用命令：

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="私信消息被忽略">
    检查：

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy`（或旧版 `channels.slack.dm.policy`）
    - 配对审批/允许列表条目（`dmPolicy: "open"` 仍需要 `channels.slack.allowFrom: ["*"]`）
    - 群组私信使用 MPIM 处理；启用 `channels.slack.dm.groupEnabled`，并且如果已配置，请将该 MPIM 加入 `channels.slack.dm.groupChannels`
    - Slack Assistant 私信事件：详细日志中提到 `drop message_changed`
      通常意味着 Slack 发送了经过编辑的 Assistant 线程事件，但消息元数据中
      没有可恢复的人类发送者

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="Socket 模式无法连接">
    在 Slack 应用设置中验证机器人令牌、应用令牌以及 Socket Mode 是否已启用。
    App-Level Token 需要 `connections:write`，而作为机器人令牌的 Bot User OAuth Token
    必须与应用令牌属于同一个 Slack 应用/工作区。

    如果 `openclaw channels status --probe --json` 显示 `botTokenStatus` 或
    `appTokenStatus: "configured_unavailable"`，则表示 Slack 账户已配置，
    但当前运行时无法解析由 SecretRef 支持的值。

    `slack socket mode failed to start; retry ...` 之类的日志表示可恢复的
    启动失败。缺少权限范围、令牌被撤销和身份验证无效则会快速失败。
    `slack token mismatch ...` 日志表示机器人令牌和应用令牌
    似乎属于不同的 Slack 应用；请修正 Slack 应用凭据。

  </Accordion>

  <Accordion title="HTTP 模式未接收事件">
    验证：

    - 签名密钥
    - webhook 路径
    - Slack Request URLs（Events + Interactivity + Slash Commands）
    - 每个 HTTP 账户使用唯一的 `webhookPath`
    - 公共 URL 终止 TLS，并将请求转发到 Gateway 网关路径
    - Slack 应用的 `request_url` 路径与 `channels.slack.webhookPath` 完全匹配（默认值为 `/slack/events`）

    如果账户快照中出现 `signingSecretStatus: "configured_unavailable"`，
    则表示 HTTP 账户已配置，但当前运行时无法
    解析由 SecretRef 支持的签名密钥。

    如果反复出现 `slack: webhook path ... already registered` 日志，则表示两个 HTTP
    账户正在使用同一个 `webhookPath`；请为每个账户指定不同的路径。

  </Accordion>

  <Accordion title="原生/斜杠命令未触发">
    验证你期望使用的是：

    - 原生命令模式（`channels.slack.commands.native: true`），并在 Slack 中注册了匹配的斜杠命令
    - 或单斜杠命令模式（`channels.slack.slashCommand.enabled: true`）

    Slack 不会自动创建或移除斜杠命令。`commands.native: "auto"` 不会启用 Slack 原生命令；请使用 `true`，并在 Slack 应用中创建匹配的命令。在 HTTP 模式下，每个 Slack 斜杠命令都必须包含 Gateway 网关 URL。在 Socket Mode 下，命令载荷通过 websocket 到达，Slack 会忽略 `slash_commands[].url`。

    还应检查 `commands.useAccessGroups`、私信授权、频道允许列表
    以及每频道 `users` 允许列表。对于被阻止的斜杠命令发送者，Slack 会返回
    仅发送者可见的错误，包括：

    - `This channel is not allowed.`
    - `You are not authorized to use this command here.`

  </Accordion>
</AccordionGroup>

## 附件媒体参考

当 Slack 文件下载成功且大小限制允许时，Slack 可以将下载的媒体附加到智能体轮次。音频剪辑可以转录，图像文件可以通过媒体理解路径处理或直接传递给支持视觉的回复模型，其他文件仍可作为可下载的文件上下文使用。

### 支持的媒体类型

| 媒体类型                     | 来源               | 当前行为                                                                  | 备注                                                                     |
| ------------------------------ | -------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Slack 音频剪辑              | Slack 文件 URL       | 下载并通过共享音频转录流程处理                          | 需要 `files:read` 以及可用的 `tools.media.audio` 模型或 CLI      |
| JPEG / PNG / GIF / WebP 图像 | Slack 文件 URL       | 下载并附加到轮次，以供支持视觉的处理路径使用                   | 单文件上限：`channels.slack.mediaMaxMb`（默认 20 MB）                 |
| PDF 文件                      | Slack 文件 URL       | 下载并作为文件上下文提供给 `download-file` 或 `pdf` 等工具 | Slack 入站流程不会自动将 PDF 转换为图像视觉输入 |
| 其他文件                    | Slack 文件 URL       | 尽可能下载并作为文件上下文提供                              | 二进制文件不会被视为图像输入                               |
| 线程回复                 | 线程起始消息文件 | 当回复没有直接媒体时，可将根消息文件载入为上下文  | 仅包含文件的起始消息使用附件占位符                          |
| 多文件消息            | 多个 Slack 文件 | 分别评估每个文件                                              | Slack 每条消息最多处理八个文件                     |

### 入站流程

当带有文件附件的 Slack 消息到达时：

1. OpenClaw 使用机器人令牌从 Slack 的私有 URL 下载文件。
2. 下载成功后，文件会写入媒体存储。
3. 下载媒体的路径和内容类型会添加到入站上下文。
4. 音频剪辑会路由到共享转录流程；支持图像的模型/工具路径可以使用同一上下文中的图像附件。
5. 其他文件仍以文件元数据或媒体引用的形式供能够处理它们的工具使用。

### 线程根附件继承

当消息在线程中到达（具有 `thread_ts` 父消息）时：

- 如果回复本身没有直接媒体，而包含的根消息带有文件，Slack 可以将根文件载入为线程起始消息上下文。
- 只有在初始化新的线程会话或重置后的线程会话时，才会载入根文件。后续纯文本回复会复用现有会话上下文，不会将根文件作为新媒体重新附加。
- 直接回复附件的优先级高于根消息附件。
- 如果根消息只有文件而没有文本，则使用附件占位符表示，以便回退流程仍可包含其文件。

### 多附件处理

当一条 Slack 消息包含多个文件附件时：

- 每个附件都会通过媒体流程独立处理。
- 下载的媒体引用会汇总到消息上下文中。
- 处理顺序遵循事件载荷中的 Slack 文件顺序。
- 一个附件下载失败不会阻塞其他附件。

### 大小、下载和模型限制

- **大小上限**：默认每个文件 20 MB。可通过 `channels.slack.mediaMaxMb` 配置。
- **音频转录上限**：将下载的文件发送给转录提供商或 CLI 时，所选支持音频的 `tools.media.models[]` 条目的 `maxBytes` 同样适用。
- **下载失败**：Slack 无法提供的文件、已过期的 URL、无法访问的文件、超出大小限制的文件，以及 Slack 身份验证/登录 HTML 响应都会被跳过，而不会被报告为不支持的格式。
- **视觉模型**：当活动回复模型支持视觉时，图像分析会使用该模型；否则使用在 `agents.defaults.imageModel` 中配置的图像模型。

### 已知限制

| 场景                                      | 当前行为                                                                   | 解决方法                                                                    |
| --------------------------------------------- | ---------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| Slack 文件 URL 已过期                        | 跳过文件；不显示错误                                                       | 在 Slack 中重新上传文件                                                   |
| 音频转录不可用               | 剪辑仍保持附加状态，但不会生成转录文本                                | 配置 `tools.media.audio` 或安装受支持的本地转录 CLI  |
| 无字幕剪辑未通过提及门控 | 在私下推测性转录后丢弃；转录文本和下载内容均被丢弃 | 配置口述名称提及模式、添加键入的 Bot 提及，或使用私信 |
| 未配置视觉模型                   | 图像附件会存储为媒体引用，但不会作为图像进行分析       | 配置 `agents.defaults.imageModel` 或使用支持视觉功能的回复模型    |
| 超大图像（默认 > 20 MB）        | 根据大小上限跳过                                                               | 如果 Slack 允许，请增大 `channels.slack.mediaMaxMb`                          |
| 转发/共享的附件                  | 尽力处理文本以及由 Slack 托管的图像/文件媒体                             | 直接在 OpenClaw 话题串中重新共享                                      |
| PDF 附件                               | 存储为文件/媒体上下文，不会自动通过图像视觉功能处理        | 使用 `download-file` 获取文件元数据，或使用 `pdf` 工具分析 PDF      |

### 相关文档

- [媒体理解流水线](/zh-CN/nodes/media-understanding)
- [音频和语音留言](/zh-CN/nodes/audio)
- [PDF 工具](/zh-CN/tools/pdf)

## 相关内容

<CardGroup cols={2}>
  <Card title="配对" icon="link" href="/zh-CN/channels/pairing">
    将 Slack 用户与 Gateway 网关配对。
  </Card>
  <Card title="群组" icon="users" href="/zh-CN/channels/groups">
    频道和群组私信行为。
  </Card>
  <Card title="频道路由" icon="route" href="/zh-CN/channels/channel-routing">
    将入站消息路由至智能体。
  </Card>
  <Card title="安全性" icon="shield" href="/zh-CN/gateway/security">
    威胁模型和安全加固。
  </Card>
  <Card title="配置" icon="sliders" href="/zh-CN/gateway/configuration">
    配置布局和优先级。
  </Card>
  <Card title="斜杠命令" icon="terminal" href="/zh-CN/tools/slash-commands">
    命令目录和行为。
  </Card>
</CardGroup>
