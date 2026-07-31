---
read_when:
    - 开发 Nextcloud Talk 渠道功能
summary: Nextcloud Talk 支持状态、功能和配置
title: Nextcloud Talk
x-i18n:
    generated_at: "2026-07-26T05:39:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 59f4fe51555bcb13d630140866307b1a49ba077059818ec116ee50ef0c877b2b
    source_path: channels/nextcloud-talk.md
    workflow: 16
---

Nextcloud Talk 是一个可下载的渠道插件（`@openclaw/nextcloud-talk`），通过 Talk webhook Bot 将 OpenClaw 连接到自托管的 Nextcloud 实例。支持私信、房间、表情回应和 Markdown 消息；媒体以 URL 形式发出。

## 安装

```bash
openclaw plugins install @openclaw/nextcloud-talk
```

使用不带版本的包说明符可跟随当前官方发布标签。仅在需要可复现安装时固定确切版本。

从本地检出安装（开发工作流）：

```bash
openclaw plugins install ./path/to/local/nextcloud-talk-plugin
```

安装后重启 Gateway 网关。详情：[插件](/zh-CN/tools/plugin)

## 快速设置（初学者）

1. 安装插件（见上文）。
2. 在你的 Nextcloud 服务器上创建一个 Bot：

   ```bash
   ./occ talk:bot:install "OpenClaw" "<shared-secret>" "<webhook-url>" --feature webhook --feature response --feature reaction
   ```

   保留 `--feature response`：如果没有它，出站回复会因 401 而失败。使用 `./occ talk:bot:state --feature webhook --feature response --feature reaction <botId> 1` 修复现有 Bot。

3. 在目标房间设置中启用该 Bot。
4. 配置 OpenClaw：
   - 配置：`channels.nextcloud-talk.baseUrl` + `channels.nextcloud-talk.botSecret`
   - 或使用环境变量：`NEXTCLOUD_TALK_BOT_SECRET`（仅限默认账户）

   CLI 设置（`--url`/`--token` 是显式字段的别名；`nc-talk` 和 `nc` 可用作渠道别名）：

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --url https://cloud.example.com \
     --token "<shared-secret>"
   ```

   等效的显式字段：

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --base-url https://cloud.example.com \
     --secret "<shared-secret>"
   ```

   基于文件的密钥：

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --base-url https://cloud.example.com \
     --secret-file /path/to/nextcloud-talk-secret
   ```

5. 重启 Gateway 网关（或完成设置）。

最小配置：

```json5
{
  channels: {
    "nextcloud-talk": {
      enabled: true,
      baseUrl: "https://cloud.example.com",
      botSecret: "shared-secret",
      dmPolicy: "pairing",
    },
  },
}
```

## 注意事项

- Bot 无法主动发起私信。用户必须先向 Bot 发送消息。
- Nextcloud 服务器必须能够访问 webhook URL；当 Gateway 网关位于代理之后时，请设置 `webhookPublicUrl`。Webhook 请求使用 Bot 密钥进行 HMAC-SHA256 签名；签名无效的请求会被拒绝并受到速率限制。
- Bot API 不支持媒体上传；出站媒体会作为 `Attachment: <url>` 行附加。
- Webhook 载荷无法区分私信和房间；设置 `apiUser` + `apiPassword` 可启用房间类型查询（缓存约 5 分钟）。如果不设置，所有对话都会被视为房间。
- 出站请求会经过 SSRF 防护。对于可信专用/内部网络上的 Nextcloud 主机，请通过 `channels.nextcloud-talk.network.dangerouslyAllowPrivateNetwork: true` 明确启用。
- 设置 `apiUser`/`apiPassword` 和 `webhookPublicUrl` 后，`openclaw channels status` 会探测 Bot，并在缺少 `response` 功能时发出警告。

## 访问控制（私信）

- 默认值：`channels.nextcloud-talk.dmPolicy = "pairing"`。未知发送者会收到配对码。
- 通过以下方式批准：
  - `openclaw pairing list nextcloud-talk`
  - `openclaw pairing approve nextcloud-talk <CODE>`
- 公开私信：`channels.nextcloud-talk.dmPolicy="open"` 加 `channels.nextcloud-talk.allowFrom=["*"]`。
- `allowFrom` 仅匹配 Nextcloud 用户 ID（转换为小写）；显示名称会被忽略。

## 房间（群组）

- 默认值：`channels.nextcloud-talk.groupPolicy = "allowlist"`（需要提及）。
- 使用 `channels.nextcloud-talk.rooms` 将房间加入允许列表，以房间令牌为键；`"*"` 设置通配符默认值：

```json5
{
  channels: {
    "nextcloud-talk": {
      rooms: {
        "room-token": { requireMention: true },
      },
    },
  },
}
```

- 每个房间的键：`requireMention`（默认为 true）、`enabled`（false 会禁用该房间）、`allowFrom`（每个房间的发送者允许列表）、`tools`（允许/拒绝工具覆盖）、`skills`（限制加载的 Skills）、`systemPrompt`。
- 要禁止所有房间，请将允许列表留空或设置 `channels.nextcloud-talk.groupPolicy="disabled"`。

## 功能

| 功能           | 状态       |
| -------------- | ---------- |
| 私信           | 支持       |
| 房间           | 支持       |
| 话题串         | 不支持     |
| 媒体           | 仅 URL     |
| 表情回应       | 支持       |
| 原生命令       | 不支持     |

## 配置参考（Nextcloud Talk）

完整配置：[配置](/zh-CN/gateway/configuration)

提供商选项：

- `channels.nextcloud-talk.enabled`：启用/禁用渠道启动。
- `channels.nextcloud-talk.baseUrl`：Nextcloud 实例 URL。
- `channels.nextcloud-talk.botSecret`：Bot 共享密钥（字符串或密钥引用）。
- `channels.nextcloud-talk.botSecretFile`：常规文件密钥路径。符号链接会被拒绝。
- `channels.nextcloud-talk.apiUser`：用于房间查询（私信检测）和状态探测的 API 用户。
- `channels.nextcloud-talk.apiPassword`：用于房间查询的 API/应用密码。
- `channels.nextcloud-talk.apiPasswordFile`：API 密码文件路径。
- `channels.nextcloud-talk.webhookPort`：Webhook 监听端口（默认值：8788）。
- `channels.nextcloud-talk.webhookHost`：Webhook 主机（默认值：0.0.0.0）。
- `channels.nextcloud-talk.webhookPath`：Webhook 路径（默认值：/nextcloud-talk-webhook）。
- `channels.nextcloud-talk.webhookPublicUrl`：可从外部访问的 webhook URL。
- `channels.nextcloud-talk.dmPolicy`：`pairing | allowlist | open | disabled`（默认值：配对）。`open` 需要 `allowFrom=["*"]`。
- `channels.nextcloud-talk.allowFrom`：私信允许列表（用户 ID）。
- `channels.nextcloud-talk.groupPolicy`：`allowlist | open | disabled`（默认值：允许列表）。
- `channels.nextcloud-talk.groupAllowFrom`：房间发送者允许列表（用户 ID）；未设置时回退到 `allowFrom`。
- `channels.nextcloud-talk.rooms`：每个房间的设置和允许列表（见上文）。
- 静态发送者访问组可以通过 `accessGroup:<name>` 从 `allowFrom` 和 `groupAllowFrom` 中引用。
- `channels.nextcloud-talk.historyLimit`：群组历史记录上限（0 表示禁用）。
- `channels.nextcloud-talk.dmHistoryLimit`：私信历史记录上限（0 表示禁用）。
- `channels.nextcloud-talk.dms`：以用户 ID 为键的每私信覆盖设置（`historyLimit`）。
- `channels.nextcloud-talk.textChunkLimit`：出站文本分块大小，以字符数计（默认值：4000）。
- `channels.nextcloud-talk.streaming.chunkMode`：`length`（默认值），或使用 `newline` 在按长度分块前按空行（段落边界）拆分。
- `channels.nextcloud-talk.streaming.block.enabled`：为此渠道启用或禁用分块流式传输。
- `channels.nextcloud-talk.streaming.block.coalesce`：分块流式传输的合并调优。
- `channels.nextcloud-talk.responsePrefix`：出站回复前缀。
- `channels.nextcloud-talk.markdown.tables`：Markdown 表格渲染模式（`off | bullets | code | block`）。
- `channels.nextcloud-talk.mediaMaxMb`：入站媒体上限（MB）。
- `channels.nextcloud-talk.network.dangerouslyAllowPrivateNetwork`：允许专用/内部 Nextcloud 主机通过 SSRF 防护。
- `channels.nextcloud-talk.accounts.<id>`：每账户覆盖设置（相同的键）；`defaultAccount` 选择默认账户。环境变量 `NEXTCLOUD_TALK_BOT_SECRET` / `NEXTCLOUD_TALK_API_PASSWORD` 仅应用于默认账户。

## 相关内容

- [渠道概览](/zh-CN/channels) — 所有受支持的渠道
- [配对](/zh-CN/channels/pairing) — 私信身份验证和配对流程
- [群组](/zh-CN/channels/groups) — 群聊行为和提及门控
- [频道路由](/zh-CN/channels/channel-routing) — 消息的会话路由
- [安全性](/zh-CN/gateway/security) — 访问模型和安全加固
