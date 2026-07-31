---
read_when:
    - 使用 OpenClaw 设置 Synology Chat
    - 调试 Synology Chat webhook 路由
summary: Synology Chat Webhook 设置和 OpenClaw 配置
title: Synology Chat
x-i18n:
    generated_at: "2026-07-26T06:09:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3c03379944ee4187260a7287f6d2aed1ad8fdd1c22b5581c8a5d55515bbb6ad5
    source_path: channels/synology-chat.md
    workflow: 16
---

Synology Chat 通过一对 webhook 连接到 OpenClaw：Synology Chat 出站 webhook 将收到的私信发送到 Gateway 网关，回复则通过 Synology Chat 入站 webhook 发回。

状态：官方插件，需单独安装。仅支持私信；支持文本和基于 URL 的文件发送。

## 安装

```bash
openclaw plugins install @openclaw/synology-chat
```

本地检出（从 Git 仓库运行时）：

```bash
openclaw plugins install ./path/to/local/synology-chat-plugin
```

详情：[插件](/zh-CN/tools/plugin)

## 快速设置

1. 安装插件（见上文）。
2. 在 Synology Chat 集成中：
   - 创建入站 webhook 并复制其 URL。
   - 使用你的秘密令牌创建出站 webhook。
3. 将出站 webhook URL 指向你的 OpenClaw Gateway 网关：
   - `https://gateway-host/webhook/synology`（默认）。
   - 或你的自定义 `channels.synology-chat.webhookPath`。
4. 在 OpenClaw 中完成设置。在这两种流程中，Synology Chat 都会出现在同一渠道设置列表中：
   - 引导式：`openclaw onboard` 或 `openclaw channels add`
   - 直接：`openclaw channels add --channel synology-chat --token <token> --url <incoming-webhook-url>`
5. 重启 Gateway 网关，然后向 Synology Chat Bot 发送私信。

Webhook 身份验证详情：

- OpenClaw 依次从 `body.token`、
  `?token=...` 和请求头中接受出站 webhook 令牌。
- 接受的请求头形式：
  - `x-synology-token`
  - `x-webhook-token`
  - `x-openclaw-token`
  - `Authorization: Bearer <token>`
- 令牌为空或缺失时将以拒绝方式安全失败。
- 载荷可以是 `application/x-www-form-urlencoded` 或 `application/json`；`token`、`user_id` 和 `text` 为必填项。

## 入站持久性

通过令牌、发送者策略和速率限制检查后，OpenClaw 会从存储的信封中移除 webhook 令牌，并在确认事件前将其持久化加入队列。只有追加成功后，路由才会返回 `204`；持久化失败则返回 `503`，以便 Synology Chat 重试，而不是静默丢失消息。

待处理或可重试的事件可在 Gateway 网关重启后继续保留。当对应的活动或保留完成记录存在时，Synology 的稳定 `post_id` 会抑制重复的队列条目。从队列移交给智能体的过程中仍采用至少一次交付，因此在该边界发生崩溃时，仍可能重放某个轮次。

最小配置：

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      token: "synology-outgoing-token",
      incomingUrl: "https://nas.example.com/webapi/entry.cgi?api=SYNO.Chat.External&method=incoming&version=2&token=...",
      webhookPath: "/webhook/synology",
      dmPolicy: "allowlist",
      allowedUserIds: ["123456"],
      rateLimitPerMinute: 30,
      allowInsecureSsl: false,
    },
  },
}
```

## 环境变量

对于默认账户，可以使用环境变量：

- `SYNOLOGY_CHAT_TOKEN`
- `SYNOLOGY_CHAT_INCOMING_URL`
- `SYNOLOGY_NAS_HOST`
- `SYNOLOGY_ALLOWED_USER_IDS`（以逗号分隔）
- `SYNOLOGY_RATE_LIMIT`
- `OPENCLAW_BOT_NAME`

配置值会覆盖环境变量。

无法通过工作区 `.env` 设置 `SYNOLOGY_CHAT_INCOMING_URL` 和 `SYNOLOGY_NAS_HOST`；请参阅[工作区 `.env` 文件](/zh-CN/gateway/security#workspace-env-files)。

## 私信策略和访问控制

- 支持的 `dmPolicy` 值：`allowlist`（默认）、`open` 和 `disabled`。Synology Chat 没有配对流程；将发送者的 Synology 数字用户 ID 添加到 `allowedUserIds` 以批准发送者。
- `allowedUserIds` 接受 Synology 用户 ID 列表（或以逗号分隔的字符串）。
- 在 `allowlist` 模式下，空的 `allowedUserIds` 列表会被视为配置错误，webhook 路由将不会启动。
- 仅当 `allowedUserIds` 包含 `"*"` 时，`dmPolicy: "open"` 才允许公开私信；如果包含限制性条目，则只有匹配的用户可以聊天。`open` 搭配空的 `allowedUserIds` 列表时，也会拒绝启动路由。
- `dmPolicy: "disabled"` 会阻止私信。
- 默认情况下，回复收件人绑定始终使用稳定的数字 `user_id`。`channels.synology-chat.dangerouslyAllowNameMatching: true` 是紧急兼容模式，会重新启用可变的用户名/昵称查找来投递回复。

## 出站投递

使用 Synology Chat 数字用户 ID 作为目标。接受 `synology-chat:`、`synology_chat:` 和 `synology:` 前缀。

示例：

```bash
openclaw message send --channel synology-chat --target 123456 --message "Hello from OpenClaw"
openclaw message send --channel synology-chat --target synology-chat:123456 --message "Hello again"
openclaw message send --channel synology-chat --target synology:123456 --message "Short prefix"
```

出站文本按 2000 个字符分块。支持通过基于 URL 的文件投递发送媒体：NAS 会下载并附加该文件（最大 32 MB）。出站文件 URL 必须使用 `http` 或 `https`，OpenClaw 在将 URL 转发给 NAS webhook 前，会拒绝私有网络目标或其他被阻止的网络目标。

## 多账户

`channels.synology-chat.accounts` 下支持多个 Synology Chat 账户。
每个账户都可以覆盖令牌、入站 URL、webhook 路径、私信策略和限制。
私信会话按账户和用户隔离，因此两个不同 Synology 账户中相同的数字 `user_id`
不会共享对话记录状态。
为每个已启用的账户指定不同的 `webhookPath`。OpenClaw 会拒绝完全重复的路径，
并且在多账户设置中，拒绝启动仅继承共享 webhook 路径的命名账户。
如果确实需要命名账户使用旧版继承，请在该账户或 `channels.synology-chat`
中设置 `dangerouslyAllowInheritedWebhookPath: true`，
但完全重复的路径仍会以拒绝方式安全失败。应优先为每个账户明确指定路径。

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      accounts: {
        default: {
          token: "token-a",
          incomingUrl: "https://nas-a.example.com/...token=...",
        },
        alerts: {
          token: "token-b",
          incomingUrl: "https://nas-b.example.com/...token=...",
          webhookPath: "/webhook/synology-alerts",
          dmPolicy: "allowlist",
          allowedUserIds: ["987654"],
        },
      },
    },
  },
}
```

## 安全说明

- 对 `token` 保密，并在泄露后进行轮换。
- 除非你明确信任本地 NAS 的自签名证书，否则请保持 `allowInsecureSsl: false`。
- 入站 webhook 请求会验证令牌，并按发送者实施速率限制（`rateLimitPerMinute`，默认值为 30）。
- 无效令牌检查使用恒定时间的秘密值比较并以拒绝方式安全失败；反复尝试无效令牌会暂时锁定来源 IP。
- 入站消息文本会针对已知的提示词注入模式进行清理，并截断至 4000 个字符。
- 生产环境应优先使用 `dmPolicy: "allowlist"`。
- 除非明确需要基于旧版用户名的回复投递，否则请保持关闭 `dangerouslyAllowNameMatching`。
- 除非在多账户设置中明确接受共享路径路由风险，否则请保持关闭 `dangerouslyAllowInheritedWebhookPath`。

## 故障排查

- `Missing required fields (token, user_id, text)`：
  - 出站 webhook 载荷缺少一个必填字段
  - 如果 Synology 在请求头中发送令牌，请确保 Gateway 网关/代理保留这些请求头
- `Invalid token`：
  - 出站 webhook 密钥与 `channels.synology-chat.token` 不匹配
  - 请求到达了错误的账户/webhook 路径
  - 反向代理在请求到达 OpenClaw 前剥离了令牌请求头
- `Rate limit exceeded`：
  - 来自同一来源的无效令牌尝试次数过多，可能会暂时锁定该来源
  - 已通过身份验证的发送者还受到单独的每用户消息速率限制
- `Allowlist is empty. Configure allowedUserIds or use dmPolicy=open with allowedUserIds=["*"].`：
  - 已启用 `dmPolicy="allowlist"`，但未配置任何用户
- `User not authorized`：
  - 发送者的数字 `user_id` 不在 `allowedUserIds` 中

## 相关内容

- [渠道概览](/zh-CN/channels) — 所有受支持的渠道
- [群组](/zh-CN/channels/groups) — 群聊行为和提及门控
- [频道路由](/zh-CN/channels/channel-routing) — 消息的会话路由
- [安全性](/zh-CN/gateway/security) — 访问模型和安全加固
