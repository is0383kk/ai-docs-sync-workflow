---
read_when:
    - 添加或修改消息 CLI 操作
    - 更改渠道出站行为
summary: '`openclaw message` 的 CLI 参考（发送 + 渠道操作）'
title: 消息
x-i18n:
    generated_at: "2026-07-26T06:38:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e2d1cca9be7cfa7625cac3e440ecb5847d9fab9c545c9267a41a2f99c26c514b
    source_path: cli/message.md
    workflow: 16
---

# `openclaw message`

用于跨 Discord、Google Chat、iMessage、Matrix、Mattermost（插件）、Microsoft Teams、Signal、Slack、Telegram 和 WhatsApp 发送消息及执行渠道操作的统一出站命令。

```bash
openclaw message <subcommand> [flags]
```

## 渠道选择

- 如果配置了多个渠道，则必须指定 `--channel <name>`；如果只配置了一个渠道，则默认使用该渠道。
- 可选值：`discord|googlechat|imessage|matrix|mattermost|msteams|signal|slack|telegram|whatsapp`
  （Mattermost 需要相应插件）。
- 带渠道前缀的目标（例如 `discord:channel:123`）无需显式指定 `--channel` 即可解析到所属插件。

## 目标格式（`-t, --target`）

| 渠道                | 格式                                                                                                       |
| ------------------- | ---------------------------------------------------------------------------------------------------------- |
| Discord             | `channel:<id>`、`user:<id>`、`<@id>` 提及，或纯数字 ID（视为渠道 ID）               |
| Google Chat         | `spaces/<spaceId>` 或 `users/<userId>`                                                                  |
| iMessage            | 句柄、`chat_id:<id>`、`chat_guid:<guid>` 或 `chat_identifier:<id>`                                       |
| Mattermost（插件）  | `channel:<id>`、`user:<id>`、`@username`，或纯 ID（视为渠道）                           |
| Matrix              | `@user:server`、`!room:server` 或 `#alias:server`                                             |
| Microsoft Teams     | `conversation:<id>`（`19:...@thread.tacv2`）、纯会话 ID 或 `user:<aad-object-id>`                                |
| Signal              | `+E.164`、`group:<id>`、`uuid:<id>`、`username:<name>`/`u:<name>`，或以上任一格式加上 `signal:` 前缀 |
| Slack               | `channel:<id>` 或 `user:<id>`（纯 ID 视为渠道）                                                |
| Telegram            | 聊天 ID、`@username`，或论坛主题目标：`<chatId>:topic:<topicId>`（或 `--thread-id <topicId>`）                  |
| WhatsApp            | E.164、群组 JID（`...@g.us`）或频道/Newsletter JID（`...@newsletter`）                           |

渠道名称查找：对于具有目录的提供商（Discord/Slack 等），`Help` 或 `#help` 等名称会通过目录缓存解析；如果缓存未命中且提供商支持实时目录查找，则回退到实时查找。

## 通用标志

每个操作都接受：`--channel <name>`、`--account <id>`、`--json`、`--dry-run`、`--verbose`。需要目标位置的操作还接受 `-t, --target <dest>`。

## SecretRef 解析

`openclaw message` 会在运行操作前解析渠道 SecretRef，并将范围尽可能缩小：

- 设置 `--channel` 时（或从带前缀的目标推断出渠道时）限定到渠道范围
- 同时设置 `--account` 时限定到账户范围
- 两者均未设置时涵盖所有已配置渠道

不相关渠道中未解析的 SecretRef 绝不会阻止定向操作；所选渠道或账户中存在未解析的 SecretRef 时，操作会以关闭方式失败。

## 操作

### 核心

| 操作            | 渠道                                                                                                            | 必需项                                                         | 备注                                                                                                                                                                                                                                                                                                  |
| --------------- | --------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `send`          | Discord、Google Chat、iMessage、Matrix、Mattermost（插件）、Microsoft Teams、Signal、Slack、Telegram、WhatsApp | `--target`，以及 `--message`/`--media`/`--presentation` 之一 | 请参阅下方的[发送](#send)。                                                                                                                                                                                                                                                                             |
| `poll`          | Discord、Matrix、Microsoft Teams、Telegram、WhatsApp                                                            | `--target`、`--poll-question`、`--poll-option`（可重复） | 请参阅下方的[投票](#poll)。                                                                                                                                                                                                                                                                             |
| `react`         | Discord、Matrix、Nextcloud Talk、Signal、Slack、Telegram、WhatsApp                                              | `--message-id`、`--target`                         | `--emoji`、`--remove`（需要 `--emoji`；省略该项可在支持的渠道中清除自己的表情回应，请参阅[表情回应](/zh-CN/tools/reactions)）。WhatsApp：`--participant`、`--from-me`。Signal 群组表情回应需要 `--target-author` 或 `--target-author-uuid`。Nextcloud Talk 仅支持添加表情回应；`--remove` 会报错。 |
| `reactions`     | Discord、Matrix、Microsoft Teams、Slack                                                                         | `--message-id`、`--target`                         | `--limit`。                                                                                                                                                                                                                                                                                   |
| `read`          | Discord、Matrix、Microsoft Teams、Slack                                                                         | `--target`                                             | `--limit`、`--message-id`、`--before`、`--after`。Discord：`--around`、`--include-thread`。Slack：`--message-id` 读取特定时间戳；与 `--thread-id` 结合可读取准确的线程回复。 |
| `edit`          | Discord、Matrix、Microsoft Teams、Slack、Telegram                                                               | `--message-id`、`--message`、`--target`    | Telegram 论坛线程使用 `--thread-id`。                                                                                                                                                                                                                                                            |
| `delete`        | Discord、Matrix、Microsoft Teams、Slack、Telegram                                                               | `--message-id`、`--target`                         |                                                                                                                                                                                                                                                                                                        |
| `pin` / `unpin` | Discord、Matrix、Microsoft Teams、Slack                                                                         | `--message-id`、`--target`                         | `unpin` 也接受 `--pinned-message-id`（Microsoft Teams：固定项/固定项列表的资源 ID，而非聊天消息 ID）。                                                                                                                                                                                          |
| `pins`（列表） | Discord、Matrix、Microsoft Teams、Slack                                                                         | `--target`                                             | `--limit`。                                                                                                                                                                                                                                                                                   |
| `permissions`   | Discord、Matrix                                                                                                 | `--target`                                             | Matrix：仅在启用加密并允许验证操作时可用。                                                                                                                                                                                                                                                             |
| `search`        | Discord                                                                                                         | `--guild-id`、`--query`                         | `--channel-id`、`--channel-ids`（可重复）、`--author-id`、`--author-ids`（可重复）、`--limit`。                                                                                                                                                                               |
| `member info`   | Discord、Matrix、Microsoft Teams、Slack                                                                         | `--user-id`                                             | `--guild-id`（Discord）。                                                                                                                                                                                                                                                                        |

### 发送

```bash
openclaw message send --channel discord \
  --target channel:123 --message "hi" --reply-to 456
```

- `--media <path-or-url>`：附加图片/音频/视频/文档（本地路径或 URL）。
- `--presentation <json>`：包含 `text`、`context`、`divider`、`chart`、`table`、`buttons` 和 `select` 块的共享有效载荷，根据各渠道的能力进行渲染。请参阅[消息呈现](/zh-CN/plugins/message-presentation)。
- `--delivery <json>`：通用投递偏好，例如 `{"pin":
true}`。当渠道支持固定投递时，`--pin` 是其简写形式。
- `--reply-to <id>`、`--thread-id <id>`（Telegram 论坛主题；Slack 线程时间戳，与 `--reply-to` 使用相同字段）。
- `--force-document`（Telegram、WhatsApp）：将图片/GIF/视频作为文档发送，以避免渠道压缩。
- `--silent`（Telegram、Discord）：发送时不触发通知。
- `--gif-playback`（仅限 WhatsApp）：将视频媒体作为 GIF 播放。

```bash
openclaw message send --channel discord \
  --target channel:123 --message "选择：" \
  --presentation '{"blocks":[{"type":"buttons","buttons":[{"label":"批准","value":"approve","style":"success"},{"label":"拒绝","value":"decline","style":"danger"}]}]}'
```

```bash
openclaw message send --channel telegram --target @mychat --message "选择：" \
  --presentation '{"blocks":[{"type":"buttons","buttons":[{"label":"是","value":"cmd:yes"},{"label":"否","value":"cmd:no"}]}]}'
```

Slack 会原生渲染支持的图表块；其他渠道会以可读文本形式接收相同
数据：

```bash
openclaw message send --channel slack --target channel:C123 \
  --presentation '{"blocks":[{"type":"chart","chartType":"bar","title":"季度收入","categories":["第一季度","第二季度"],"series":[{"name":"收入","values":[120,145]}],"xLabel":"季度"}]}'
```

Slack 也会原生渲染显式表格块。其他渠道会以确定性文本形式接收
说明文字和每一行：

```bash
openclaw message send --channel slack --target channel:C123 \
  --presentation '{"title":"销售管线报告","blocks":[{"type":"table","caption":"未结销售管线","headers":["客户","阶段","ARR"],"rows":[["Acme","Won",125000],["Globex","Review",82000]],"rowHeaderColumnIndex":0}]}'
```

Telegram Mini App 按钮使用 `webApp`（仍会解析 `web_app` 以兼容旧版
JSON），并且仅在用户与 Bot 之间的私聊中渲染：

```bash
openclaw message send --channel telegram --target 123456789 --message "打开应用：" \
  --presentation '{"blocks":[{"type":"buttons","buttons":[{"label":"启动","webApp":{"url":"https://example.com/app"}}]}]}'
```

```bash
openclaw message send --channel telegram --target @mychat \
  --media ./diagram.png --force-document
```

```bash
openclaw message send --channel msteams \
  --target conversation:19:abc@thread.tacv2 \
  --presentation '{"title":"状态更新","blocks":[{"type":"text","text":"构建已完成"}]}'
```

### 投票

```bash
openclaw message poll --channel discord \
  --target channel:123 \
  --poll-question "零食？" \
  --poll-option Pizza --poll-option Sushi \
  --poll-multi --poll-duration-hours 48
```

- `--poll-option <choice>`：重复 2-12 次。
- `--poll-multi`：允许多选。
- Discord：`--poll-duration-hours`、`--silent`、`--message`。
- Telegram：`--poll-duration-seconds <n>`（5-600）、`--silent`、
  `--poll-anonymous` / `--poll-public`、`--thread-id`。

```bash
openclaw message poll --channel telegram \
  --target @mychat \
  --poll-question "午餐？" \
  --poll-option Pizza --poll-option Sushi \
  --poll-duration-seconds 120 --silent
```

```bash
openclaw message poll --channel msteams \
  --target conversation:19:abc@thread.tacv2 \
  --poll-question "午餐？" \
  --poll-option Pizza --poll-option Sushi
```

### 话题串

- `thread create`：渠道 Discord。必需：`--thread-name`、`--target`
  （频道 ID）。可选：`--message-id`、`--message`、`--auto-archive-min`。
- `thread list`：渠道 Discord。必需：`--guild-id`。可选：
  `--channel-id`、`--include-archived`、`--before`、`--limit`。
- `thread reply`：渠道 Discord。必需：`--target`（话题串 ID）、
  `--message`。可选：`--media`、`--reply-to`。

### 表情符号

- `emoji list`：Discord（`--guild-id`）、Slack（无额外标志）。
- `emoji upload`：Discord。必需：`--guild-id`、`--emoji-name`、`--media`。
  可选：`--role-ids`（可重复）。

### 贴纸

- `sticker send`：Discord。必需：`--target`、`--sticker-id`（可重复）。
  可选：`--message`。
- `sticker upload`：Discord。必需：`--guild-id`、`--sticker-name`、
  `--sticker-desc`、`--sticker-tags`、`--media`。

### 身份组、频道、语音和事件（Discord）

- `role info`：`--guild-id`。
- `role add` / `role remove`：`--guild-id`、`--user-id`、`--role-id`。
- `channel info`：`--target`。
- `channel list`：`--guild-id`。
- `voice status`：`--guild-id`、`--user-id`。
- `event list`：`--guild-id`。
- `event create`：必需 `--guild-id`、`--event-name`、`--start-time`；
  可选 `--end-time`、`--desc`、`--channel-id`、`--location`、
  `--event-type`、`--image <url-or-path>`。

### 管理操作（Discord）

- `timeout`：`--guild-id`、`--user-id`；可选 `--duration-min` 或
  `--until`（两者均省略可清除超时）、`--reason`。
- `kick`：`--guild-id`、`--user-id`、`--reason`。
- `ban`：`--guild-id`、`--user-id`、`--delete-days`、`--reason`。

### 广播

```bash
openclaw message broadcast --targets <target...> [--channel all] [--message <text>] [--media <url>] [--dry-run]
```

将一个有效负载发送到多个目标。`--targets` 接受以空格分隔的
列表。使用 `--channel all` 可将所有已配置的提供商设为目标。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [Agent 发送](/zh-CN/tools/agent-send)
- [消息呈现](/zh-CN/plugins/message-presentation)
