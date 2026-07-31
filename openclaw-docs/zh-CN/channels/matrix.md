---
read_when:
    - 在 OpenClaw 中设置 Matrix
    - 配置 Matrix 端到端加密和验证
summary: Matrix 支持状态、设置和配置示例
title: Matrix
x-i18n:
    generated_at: "2026-07-26T06:33:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa84c7d9d9019040a3fec3cfaabb78590006a4a2dd4bb95836f2cf37072777c5
    source_path: channels/matrix.md
    workflow: 16
---

Matrix 是一个可下载的渠道插件（`@openclaw/matrix`），基于官方 `matrix-js-sdk` 构建。它支持私信、房间、话题串、媒体、表情回应、投票、位置和端到端加密（E2EE）。

## 安装

```bash
openclaw plugins install @openclaw/matrix
```

仅含插件名称的规格会先尝试 ClawHub，然后回退到 npm。使用 `openclaw plugins install clawhub:@openclaw/matrix` 或 `npm:@openclaw/matrix` 强制指定来源。从本地检出安装：`openclaw plugins install ./path/to/local/matrix-plugin`。

`plugins install` 会注册并启用插件；无需单独执行 `enable` 步骤。在完成下方配置之前，该渠道仍不会执行任何操作。常规安装规则请参阅[插件](/zh-CN/tools/plugin)。

## 设置

1. 在你的主服务器上创建一个 Matrix 账户。
2. 使用 `homeserver` + `accessToken`，或 `homeserver` + `userId` + `password` 配置 `channels.matrix`。
3. 重启 Gateway 网关。
4. 与 Bot 发起私信，或邀请它加入房间。只有 [`autoJoin`](#auto-join) 允许时，新邀请才会生效。

### 交互式设置

```bash
openclaw channels add
openclaw configure --section channels
```

向导会询问主服务器 URL、身份验证方式（令牌或密码）、用户 ID（仅密码身份验证）、可选的设备名称、是否启用 E2EE，以及房间访问和自动加入设置。如果匹配的 `MATRIX_*` 环境变量已存在，且该账户没有已保存的身份验证信息，向导会提供使用环境变量的快捷方式。使用 `openclaw channels resolve --channel matrix "Project Room"` 解析房间名称后再保存允许列表。在向导中启用 E2EE 会运行与[`openclaw matrix encryption setup`](#encryption-and-verification)相同的引导流程。

### 最小配置

基于令牌：

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

基于密码（首次登录后会缓存令牌）：

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

### 自动加入

`channels.matrix.autoJoin` 默认为 `"off"`：在你手动加入之前，Bot 不会出现在新邀请的房间或私信中。OpenClaw 无法在收到邀请时判断它是私信还是群组，因此每个邀请都会先经过 `autoJoin`；`dm.policy` 仅在之后生效，即 Bot 已加入且房间已完成分类后。

<Warning>
设置 `autoJoin: "allowlist"` 和 `autoJoinAllowlist` 以限制接受的邀请，或设置 `autoJoin: "always"` 以接受所有邀请。

`autoJoinAllowlist` 仅接受 `!roomId:server`、`#alias:server` 或 `*`。普通房间名称会被拒绝；别名根据主服务器解析，而不是根据受邀房间声明的状态解析。
</Warning>

```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": { requireMention: true },
      },
    },
  },
}
```

### 允许列表目标格式

- 私信（`dm.allowFrom`、`groupAllowFrom`、`groups.<room>.users`）：使用 `@user:server`。默认忽略显示名称（显示名称可变）；仅在明确需要兼容显示名称时设置 `dangerouslyAllowNameMatching: true`。
- 房间允许列表键（`groups`，旧版别名为 `rooms`）：使用 `!room:server` 或 `#alias:server`。除非设置 `dangerouslyAllowNameMatching: true`，否则会忽略普通名称。
- 邀请允许列表（`autoJoinAllowlist`）：使用 `!room:server`、`#alias:server` 或 `*`。普通名称始终会被拒绝。

### 账户 ID 规范化

向导会将易读名称转换为规范化的账户 ID（`Ops Bot` -> `ops-bot`）。在限定作用域的环境变量名称中，标点符号会进行十六进制转义，以防账户发生冲突：`-`（0x2D）会变为 `_X2D_`，因此 `ops-prod` 会映射到环境变量前缀 `MATRIX_OPS_X2D_PROD_`。

### 缓存的凭据

Matrix 会将账户凭据缓存在共享的 `state/openclaw.sqlite` 插件状态中。存在缓存凭据时，即使配置文件中没有 `accessToken`，OpenClaw 也会将 Matrix 视为已配置——这适用于设置、`openclaw doctor` 和渠道状态探测。升级时会通过 `openclaw doctor --fix` 导入已弃用的 `~/.openclaw/credentials/matrix/credentials*.json` 文件，验证 SQLite 行，然后归档这些文件。

### 环境变量

由配置键支持的环境变量会在对应配置键未设置时使用。默认账户使用无前缀的名称；命名账户会在后缀前插入账户令牌（请参阅[规范化](#account-id-normalization)）。

| 默认账户       | 命名账户（`<ID>` = 账户令牌） |
| --------------------- | -------------------------------------- |
| `MATRIX_HOMESERVER`   | `MATRIX_<ID>_HOMESERVER`               |
| `MATRIX_ACCESS_TOKEN` | `MATRIX_<ID>_ACCESS_TOKEN`             |
| `MATRIX_USER_ID`      | `MATRIX_<ID>_USER_ID`                  |
| `MATRIX_PASSWORD`     | `MATRIX_<ID>_PASSWORD`                 |
| `MATRIX_DEVICE_ID`    | `MATRIX_<ID>_DEVICE_ID`                |
| `MATRIX_DEVICE_NAME`  | `MATRIX_<ID>_DEVICE_NAME`              |

对于账户 `ops`，名称会变为 `MATRIX_OPS_HOMESERVER`、`MATRIX_OPS_ACCESS_TOKEN`，依此类推。无法通过工作区 `.env` 设置 `MATRIX_HOMESERVER`（以及任何限定 `*_HOMESERVER` 作用域的变体）；请参阅[工作区 `.env` 文件](/zh-CN/gateway/security)。

<Note>
恢复密钥不是由配置支持的环境变量：OpenClaw 本身绝不会从环境中读取它。CLI 指引文本建议，对于默认账户，通过名为 `MATRIX_RECOVERY_KEY` 的 shell 变量传入恢复密钥；对于命名账户，则通过 `MATRIX_RECOVERY_KEY_<ID>`（直接将账户 ID 转为大写，不进行十六进制转义）传入——请参阅[使用恢复密钥验证此设备](#verify-this-device-with-a-recovery-key)。
</Note>

## 配置示例

包含私信配对、房间允许列表和 E2EE 的实用基线配置：

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": { requireMention: true },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: { mode: "partial" },
    },
  },
}
```

## 流式预览

Matrix 回复流式传输需主动启用。`streaming.mode` 控制 OpenClaw 如何传送生成中的智能体回复；`streaming.block.enabled` 控制是否将每个已完成的分块保留为单独的 Matrix 消息。

```json5
{
  channels: {
    matrix: {
      streaming: { mode: "partial" },
    },
  },
}
```

若要保留实时回答预览，但隐藏中间的工具/进度行：

```json5
{
  channels: {
    matrix: {
      streaming: {
        mode: "partial",
        preview: {
          toolProgress: false,
        },
      },
    },
  },
}
```

完整配置接受 `{ mode, chunkMode, block, preview, progress }`：

```json5
{
  channels: {
    matrix: {
      streaming: {
        mode: "progress",
        progress: {
          label: "auto", // 从配置或内置标签中选取（设为 false 可隐藏）
          labels: ["思考中", "撰写中", "搜索中"], // label: "auto" 的候选项
          maxLines: 8, // 滚动显示的最大进度行数（默认：8）
          maxLineChars: 120, // 截断前每行的最大字符数（默认：120）
          toolProgress: true, // 显示工具/进度活动（默认：true）
        },
      },
    },
  },
}
```

- `progress.label`：自定义标签；设置为 `"auto"` 或不设置时，从已配置或内置标签中选取；设置为 `false` 时隐藏标签。
- `progress.labels`：仅当 `label` 为 `"auto"` 或未设置时使用的候选项。
- `progress.maxLines`：草稿中保留的最大滚动进度行数；超过后会移除较早的行。
- `progress.maxLineChars`：每条紧凑进度行在截断前允许的最大字符数。
- `progress.toolProgress`：当设为 `true`（默认）时，实时工具/进度活动会显示在草稿中。

| `streaming.mode`  | 行为                                                                                                                                                 |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `"off"`（默认） | 等待完整回复，然后一次性发送。                                                                                                                      |
| `"partial"`       | 模型撰写当前分块时，就地编辑一条普通文本消息。标准客户端可能在首次预览时发出通知，而不是在最终编辑时。          |
| `"quiet"`         | 与 `"partial"` 相同，但消息是不触发通知的通知类消息。当每用户推送规则与最终编辑匹配时，接收者会收到一次通知（见下文）。 |
| `"progress"`      | 使用进度草稿发送各条紧凑的进度行。                                                                                          |

`streaming.block.enabled`（默认值为 `false`）独立于 `streaming.mode`：

| `streaming.mode`        | `block.enabled: true`                                               | `block.enabled: false`（默认）                     |
| ----------------------- | ------------------------------------------------------------------- | ---------------------------------------------------- |
| `"partial"` / `"quiet"` | 当前分块使用实时草稿，已完成的分块保留为消息 | 当前分块使用实时草稿，并就地完成最终定稿 |
| `"off"`                 | 每个完成的分块发送一条触发通知的 Matrix 消息                     | 整个回复发送一条触发通知的 Matrix 消息      |

注意：

- 如果预览内容超过 Matrix 的单事件大小限制，OpenClaw 会停止预览流式传输，并回退到仅传送最终内容。
- 媒体回复始终按正常方式发送附件；如果无法安全地复用过期预览，OpenClaw 会先将其撤回，再发送最终媒体回复。
- 启用预览流式传输时，默认会更新工具进度预览。设置 `streaming.preview.toolProgress: false` 可保留回答文本的预览编辑，但让工具进度继续使用正常传送路径。
- 预览编辑会产生额外的 Matrix API 调用。为获得最保守的速率限制配置，请保留 `streaming.mode: "off"`。
- 旧版标量/布尔值 `streaming` 以及扁平的 `blockStreaming` / `chunkMode` 键会由 `openclaw doctor --fix` 重写为这种嵌套结构。

## 语音消息

入站 Matrix 语音消息会在房间提及检查之前转录，因此在 `requireMention: true` 房间中，一条说出 Bot 名称的语音消息可以触发智能体，并且智能体会收到转录文本，而不只是音频附件占位符。

Matrix 使用 `tools.media.audio` 下的共享音频媒体提供商，例如 OpenAI `gpt-4o-mini-transcribe`。有关提供商设置和限制，请参阅[媒体工具概览](/zh-CN/tools/media-overview)。

- `m.audio` 事件和 MIME 类型为 `audio/*` 的 `m.file` 事件符合条件。
- 在加密房间中，OpenClaw 会先通过现有的 Matrix 媒体路径解密附件，然后再进行转录。
- 在智能体提示词中，转录文本会被标记为由机器生成且不受信任。
- 附件会被标记为已转录，以免下游媒体工具再次进行转录。
- 设置 `tools.media.audio.enabled: false` 可全局禁用音频转录。

## 审批元数据

Matrix 原生审批提示是普通的 `m.room.message` 事件，其 OpenClaw 专用内容位于 `com.openclaw.approval` 键下。标准客户端仍会呈现文本正文；支持 OpenClaw 的客户端可以读取结构化的审批 ID、类型、状态、决定以及 Exec/插件详情。

当提示过长，无法放入单个 Matrix 事件时，OpenClaw 会对可见文本进行分块，并且只在第一个块上附加 `com.openclaw.approval`。允许/拒绝表情回应会绑定到该第一个事件，因此长提示与单事件提示使用相同的审批目标。

### 用于安静的最终预览的自托管推送规则

`streaming.mode: "quiet"` 仅在块或轮次最终确定后通知接收者——必须使用按用户配置的推送规则来匹配最终预览标记。完整配置方法请参阅 [用于安静预览的 Matrix 推送规则](/zh-CN/channels/matrix-push-rules)。

## Bot 间通信房间

默认情况下，来自其他已配置 OpenClaw Matrix 账号的 Matrix 消息会被忽略。使用 `allowBots` 可有意允许智能体间通信：

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true` 接受允许的房间和私信中来自其他已配置 Matrix Bot 账号的消息。
- `allowBots: "mentions"` 仅当这些消息在房间中明确提及此 Bot 时才接受；无论是否提及，私信仍会被接受。
- `groups.<room>.allowBots` 覆盖单个房间的账号级设置。
- 接受的已配置 Bot 消息使用共享的 [Bot 循环保护](/zh-CN/channels/bot-loop-protection)。配置 `channels.defaults.botLoopProtection`，然后使用 `channels.matrix.botLoopProtection` 按账号覆盖，或使用 `channels.matrix.groups.<room>.botLoopProtection` 按房间覆盖。
- OpenClaw 仍会忽略来自同一 Matrix 用户 ID 的消息，以避免自我回复循环。
- Matrix 没有原生 Bot 标志；OpenClaw 将“由 Bot 发送”视为“由此 OpenClaw Gateway 网关上的另一个已配置 Matrix 账号发送”。

在共享房间中启用 Bot 间通信时，请使用严格的房间允许列表和提及要求。

## 加密和验证

在加密（E2EE）房间中，出站图像事件使用 `thumbnail_file`，因此图像预览会与完整附件一起加密；未加密房间使用普通的 `thumbnail_url`。无需配置——插件会自动检测 E2EE 状态。

所有 `openclaw matrix` 命令都接受 `--verbose`（完整诊断）、`--json`（机器可读输出）和 `--account <id>`（多账号设置）。默认输出简洁。

### 启用加密

```bash
openclaw matrix encryption setup
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix encryption setup --recovery-key-stdin
```

引导设置机密存储和交叉签名，在需要时创建房间密钥备份，然后输出状态和后续步骤。实用标志：

- `--recovery-key-stdin` 从标准输入读取恢复密钥，而不会在进程参数中暴露它；`--recovery-key <key>` 仍保留用于兼容
- `--force-reset-cross-signing` 丢弃当前交叉签名身份并创建新身份（仅限有意使用）

对于新账号，请在创建时启用 E2EE：

```bash
openclaw matrix account add \
  --homeserver https://matrix.example.org \
  --access-token syt_xxx \
  --enable-e2ee
```

`--encryption` 是 `--enable-e2ee` 的别名。等效的手动配置：

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

### 状态和信任信号

```bash
openclaw matrix verify status
openclaw matrix verify status --include-recovery-key --json
```

`verify status` 报告三个相互独立的信任信号（`--verbose` 会显示全部信号）：

- `Locally trusted`：仅受此客户端信任
- `Cross-signing verified`：SDK 报告已通过交叉签名验证
- `Signed by owner`：已由你自己的自签名密钥签名（仅用于诊断）

仅当 `Cross-signing verified` 为 `yes` 时，`Verified by owner` 才是 `yes`；仅有本地信任或所有者签名并不足够。

`--allow-degraded-local-state` 无需先准备 Matrix 账号即可返回尽力而为的诊断信息；适用于离线探测或配置不完整的探测。

### 使用恢复密钥验证此设备

通过标准输入传递恢复密钥，而不是在命令行中传递：

```bash
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin
```

该命令报告三种状态：

- `Recovery key accepted`：Matrix 已接受该密钥，用于机密存储或设备信任。
- `Backup usable`：可以使用受信任的恢复材料加载房间密钥备份。
- `Device verified by owner`：此设备具有完整的 Matrix 交叉签名身份信任。

如果完整身份信任尚未完成，即使恢复密钥已解锁备份材料，该命令仍会以非零状态退出。在这种情况下，请从另一个 Matrix 客户端完成自我验证：

```bash
openclaw matrix verify self
```

`verify self` 会等待 `Cross-signing verified: yes`，然后成功退出。使用 `--timeout-ms <ms>` 调整等待时间。

字面密钥形式 `openclaw matrix verify device "<recovery-key>"` 也可以使用，但密钥会留在 shell 历史记录中。

### 引导设置或修复交叉签名

```bash
openclaw matrix verify bootstrap
```

这是用于加密账号的修复/设置命令。它会按以下顺序执行：

- 引导设置机密存储，并尽可能复用现有恢复密钥
- 引导设置交叉签名并上传缺失的公钥
- 标记当前设备并对其进行交叉签名
- 如果服务器端房间密钥备份尚不存在，则创建一个

如果主服务器要求通过 UIA 上传交叉签名密钥，OpenClaw 会先尝试无身份验证方式，然后尝试 `m.login.dummy`，再尝试 `m.login.password`（需要 `channels.matrix.password`）。

实用标志：

- `--recovery-key-stdin`（与 `printf '%s\n' "$MATRIX_RECOVERY_KEY" | ...` 搭配）或 `--recovery-key <key>`
- `--force-reset-cross-signing` 用于丢弃当前交叉签名身份（仅限有意操作；要求活动恢复密钥已存储，或通过 `--recovery-key-stdin` 提供）

### 房间密钥备份

```bash
openclaw matrix verify backup status
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin
```

`backup status` 显示服务器端备份是否存在，以及此设备能否将其解密。`backup restore` 将已备份的房间密钥导入本地加密存储；如果恢复密钥已存储在磁盘上，则省略 `--recovery-key-stdin`。

若要使用全新基线替换损坏的备份（接受丢失无法恢复的旧历史记录；如果当前备份机密无法加载，也可以重新创建机密存储）：

```bash
openclaw matrix verify backup reset --yes
```

仅当需要有意使先前的恢复密钥无法再解锁全新备份基线时，才添加 `--rotate-recovery-key`。

### 列出、请求和响应验证

```bash
openclaw matrix verify list
```

列出所选账号的待处理验证请求。

```bash
openclaw matrix verify request --own-user
openclaw matrix verify request --user-id @ops:example.org --device-id ABCDEF
```

从此账号发送验证请求。`--own-user` 请求自我验证（在同一用户的另一个 Matrix 客户端中接受提示）；`--user-id`/`--device-id`/`--room-id` 以其他人为目标。`--own-user` 不能与其他目标标志结合使用。

对于更底层的生命周期处理——通常用于跟踪来自另一个客户端的入站请求——以下命令对特定请求 `<id>` 执行操作（由 `verify list` 和 `verify request` 输出）：

| 命令                                       | 用途                                                                |
| ------------------------------------------ | ------------------------------------------------------------------- |
| `openclaw matrix verify accept <id>`       | 接受入站请求                                                        |
| `openclaw matrix verify start <id>`        | 启动 SAS 流程                                                       |
| `openclaw matrix verify sas <id>`          | 输出 SAS 表情符号或十进制数字                                      |
| `openclaw matrix verify confirm-sas <id>`  | 确认 SAS 与另一客户端显示的内容匹配                                 |
| `openclaw matrix verify mismatch-sas <id>` | 当表情符号或十进制数字不匹配时拒绝 SAS                              |
| `openclaw matrix verify cancel <id>`       | 取消；接受可选的 `--reason <text>` 和 `--code <matrix-code>` |

当验证锚定到特定私信房间时，`accept`、`start`、`sas`、`confirm-sas`、`mismatch-sas` 和 `cancel` 都接受 `--user-id` 和 `--room-id` 作为私信后续提示。

### 多账号说明

如果没有 `--account <id>`，Matrix CLI 命令会使用隐式默认账号。如果存在多个命名账号但未指定 `channels.matrix.defaultAccount`，命令将拒绝猜测并要求你选择。当命名账号的 E2EE 被禁用或不可用时，错误会指向该账号的配置键，例如 `channels.matrix.accounts.assistant.encryption`。

<AccordionGroup>
  <Accordion title="启动行为">
    使用 `encryption: true` 时，`startupVerification` 默认为 `"if-unverified"`。启动时，未验证设备会在另一个 Matrix 客户端中请求自我验证，同时跳过重复请求并应用冷却时间（默认为 24 小时）。使用 `startupVerificationCooldownHours` 调整，或使用 `startupVerification: "off"` 禁用。

    启动时还会运行一次保守的加密引导流程，复用当前的机密存储和交叉签名身份。如果引导状态损坏，即使没有 `channels.matrix.password`，OpenClaw 也会尝试受保护的修复；如果主服务器要求密码 UIA，启动过程会记录警告，但不会因此失败。已具有所有者签名的设备会被保留。

    完整升级流程请参阅 [Matrix 迁移](/zh-CN/channels/matrix-migration)。

  </Accordion>

  <Accordion title="验证通知">
    Matrix 会将验证生命周期通知作为 `m.notice` 消息发送到严格的私信验证房间中：请求、就绪（包含“通过表情符号验证”指引）、开始/完成，以及可用时的 SAS（表情符号/十进制数字）详情。

    来自另一个 Matrix 客户端的入站请求会被跟踪并自动接受。对于自我验证，OpenClaw 会自动启动 SAS 流程，并在表情符号验证可用后确认自己这一端——你仍需在 Matrix 客户端中进行比较并确认“They match”。

    验证系统通知不会转发到智能体聊天管道。

  </Accordion>

  <Accordion title="已删除或无效的 Matrix 设备">
    如果 `verify status` 表明当前设备已不在主服务器列表中，请创建新的 OpenClaw Matrix 设备。对于密码登录：

```bash
openclaw matrix account add \
  --account assistant \
  --homeserver https://matrix.example.org \
  --user-id '@assistant:example.org' \
  --password '<password>' \
  --device-name OpenClaw-Gateway
```

    对于令牌身份验证，请在 Matrix 客户端或管理员 UI 中创建新的访问令牌，然后更新 OpenClaw：

```bash
openclaw matrix account add \
  --account assistant \
  --homeserver https://matrix.example.org \
  --access-token '<token>'
```

    将 `assistant` 替换为失败命令中的账户 ID，或省略 `--account` 以使用默认账户。

  </Accordion>

  <Accordion title="设备维护">
    由 OpenClaw 管理的旧设备可能会不断累积。列出并清理这些设备：

```bash
openclaw matrix devices list
openclaw matrix devices prune-stale
```

  </Accordion>

  <Accordion title="加密存储">
    Matrix E2EE 使用官方 `matrix-js-sdk` Rust 加密路径，并将 `fake-indexeddb` 用作 IndexedDB 适配层。加密状态会持久化到 `crypto-idb-snapshot.json`（采用严格的文件权限）。

    加密的运行时状态位于 `~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/` 下，包括同步存储、加密存储、恢复密钥、IDB 快照、线程绑定和启动验证状态。当令牌发生变化但账户身份保持不变时，OpenClaw 会复用最佳的现有根目录，以便先前的状态仍然可见。

    单个较旧的令牌哈希根目录可能是正常的令牌轮换连续性路径。如果 OpenClaw 记录了 `matrix: multiple populated token-hash storage roots detected`，请检查账户目录，并且仅在确认所选活动根目录正常后归档过时的同级根目录。最好将过时的根目录移入 `_archive/` 目录，而不是立即删除。

  </Accordion>
</AccordionGroup>

## 资料管理

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

在一次调用中同时传入两个选项。Matrix 直接接受 `mxc://` 头像 URL；传入 `http://`/`https://` 时，会先上传文件，然后将解析后的 `mxc://` URL 存入 `channels.matrix.avatarUrl`（或对应账户的覆盖项）。

## 线程

Matrix 对自动回复和消息工具发送均支持原生线程。两个相互独立的选项控制其行为：

### 会话路由（`sessionScope`）

`dm.sessionScope` 决定如何将 Matrix 私信房间映射到 OpenClaw 会话：

- `"per-user"`（默认）：路由到同一对端的所有私信房间共享一个会话。
- `"per-room"`：每个 Matrix 私信房间都有自己的会话键，即使对端相同也是如此。

显式对话绑定始终优先于 `sessionScope`；已绑定的房间和线程会保留其选定的目标会话。

### 回复线程（`threadReplies`）

`threadReplies` 决定机器人在何处发布回复：

- `"off"`：回复位于顶层。入站线程消息仍使用父会话。
- `"inbound"`：仅当入站消息已位于某个线程中时，才在线程内回复。
- `"always"`：在线程内回复，线程根为触发消息；从首次触发开始，该对话通过匹配的线程范围会话进行路由。

`dm.threadReplies` 仅针对私信覆盖此设置——例如，在保持私信扁平化的同时隔离房间线程。

### 线程继承和斜杠命令

- 入站线程消息会将线程根消息作为额外的智能体上下文。
- 消息工具发送以同一房间（或同一私信用户目标）为目标时，会自动继承当前 Matrix 线程，除非显式提供 `threadId`。
- 仅当当前会话元数据能够证明其为同一 Matrix 账户上的同一私信对端时，才会复用私信用户目标；否则，OpenClaw 会回退到常规的用户范围路由。
- `/focus`、`/unfocus`、`/agents`、`/session idle`、`/session max-age` 和线程绑定的 `/acp spawn` 均可在 Matrix 房间和私信中使用。
- 启用 `threadBindings.spawnSessions` 后，顶层 `/focus` 会创建新的 Matrix 线程，并将其绑定到目标会话。
- 在现有 Matrix 线程内运行 `/focus` 或 `/acp spawn --thread here`，会就地绑定该线程。

当 OpenClaw 检测到某个 Matrix 私信房间与同一共享会话中的另一个私信房间冲突时，会发布一次性 `m.notice`，指向 `/focus` 这一规避方法，并建议更改 `dm.sessionScope`。仅在线程绑定已启用时才会显示此通知。

## ACP 对话绑定

Matrix 房间、私信和现有 Matrix 线程可以成为持久的 ACP 工作区，而无需更改聊天界面。

操作员快速流程：

- 在 Matrix 私信、房间或现有线程内运行 `/acp spawn codex --bind here` 以继续使用。
- 在顶层私信或房间中，当前私信/房间会保留为聊天界面，后续消息将路由到已生成的 ACP 会话。
- 在现有线程内，`--bind here` 会就地绑定当前线程。
- `/new` 和 `/reset` 会就地重置同一个已绑定 ACP 会话。
- `/acp close` 会关闭 ACP 会话并移除绑定。

`--bind here` 不会创建子 Matrix 线程。`threadBindings.spawnSessions` 控制 `/acp spawn --thread auto|here`，在该流程中 OpenClaw 需要创建或绑定子线程。

### 线程绑定配置

Matrix 从 `session.threadBindings` 继承全局默认值，并支持按渠道覆盖：

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSessions`：同时控制子智能体和 ACP 线程的生成。
- 已弃用的 `threadBindings.spawnSubagentSessions` / `threadBindings.spawnAcpSessions` 键会由 `openclaw doctor --fix` 迁移到 `spawnSessions`。
- `threadBindings.defaultSpawnContext`

Matrix 线程绑定会话的生成默认启用。将 `threadBindings.spawnSessions: false` 设置为阻止顶层 `/focus` 和 `/acp spawn --thread auto|here` 创建或绑定 Matrix 线程。如果原生子智能体线程生成不应派生父转录，请设置 `threadBindings.defaultSpawnContext: "isolated"`。

## 表情回应

Matrix 支持出站表情回应、入站表情回应通知和确认表情回应。

出站表情回应工具由 `channels.matrix.actions.reactions` 控制：

- `react` 为 Matrix 事件添加表情回应。
- `reactions` 列出 Matrix 事件当前的表情回应摘要。
- `emoji=""` 移除机器人自己在该事件上的表情回应。
- `remove: true` 仅移除机器人指定的表情符号回应。

**解析顺序**（第一个已定义的值优先）：

| 设置                 | 顺序                                                                               |
| ----------------------- | ----------------------------------------------------------------------------------- |
| `ackReaction`           | 按账户 -> 渠道 -> `messages.ackReaction` -> 智能体身份表情符号回退   |
| `ackReactionScope`      | 按账户 -> 渠道 -> `messages.ackReactionScope` -> 默认 `"group-mentions"` |
| `reactionNotifications` | 按账户 -> 渠道 -> 默认 `"own"`                                           |

当新增的 `m.reaction` 事件以机器人编写的 Matrix 消息为目标时，`reactionNotifications: "own"` 会转发这些事件；`"off"` 会禁用表情回应系统事件。表情回应移除不会被合成为系统事件——Matrix 将其呈现为删改，而不是独立的 `m.reaction` 移除事件。

## 历史上下文

- `channels.matrix.historyLimit` 控制当房间消息触发智能体时，将多少条近期房间消息作为 `InboundHistory` 包含在内。回退到 `messages.groupChat.historyLimit`；如果二者均未设置，则实际默认值为 `0`（禁用）。
- Matrix 房间历史记录仅限房间；私信继续使用常规会话历史记录。
- 房间历史记录仅包含待处理消息：OpenClaw 会缓冲尚未触发回复的房间消息，然后在提及或其他触发条件到达时创建该窗口的快照。
- 当前触发消息不包含在 `InboundHistory` 中；它会保留在该轮次的主入站正文中。
- 重试同一 Matrix 事件时，会复用原始历史记录快照，而不会向前漂移到更新的房间消息。

## 上下文可见性

Matrix 支持共享的 `contextVisibility` 控制项，用于控制补充房间上下文，例如获取的回复文本、线程根和待处理历史记录。

- `contextVisibility: "all"` 为默认值。补充上下文会按收到时的内容保留。
- `contextVisibility: "allowlist"` 会将补充上下文筛选为活动房间/用户允许列表检查所允许的发送者。
- `contextVisibility: "allowlist_quote"` 的行为类似 `allowlist`，但仍会保留一条显式引用回复。

这仅影响补充上下文的可见性，不影响入站消息本身能否触发回复。触发授权仍来自 `groupPolicy`、`groups`、`groupAllowFrom` 和私信策略设置。

## 私信和房间策略

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": { requireMention: true },
      },
    },
  },
}
```

要在保持房间正常工作的同时完全停用私信，请设置 `dm.enabled: false`：

```json5
{
  channels: {
    matrix: {
      dm: { enabled: false },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
    },
  },
}
```

有关提及触发和允许列表行为，请参阅[群组](/zh-CN/channels/groups)。

Matrix 私信的配对示例：

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

如果未经批准的 Matrix 用户在获批前持续发送消息，OpenClaw 会复用同一个待处理配对码，并可能在短暂冷却后发送提醒回复，而不是生成新配对码。

有关共享私信配对流程和存储布局，请参阅[配对](/zh-CN/channels/pairing)。

## 私信房间修复

如果私信状态发生偏移，OpenClaw 最终可能会出现过时的 `m.direct` 映射，指向旧的单人房间而不是当前有效的私信。检查某个对端的当前映射：

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

修复映射：

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

对于多账户设置，这两个命令均接受 `--account <id>`。修复流程：

- 优先使用已在 `m.direct` 中映射的严格 1:1 私信
- 否则使用当前已加入的、与该用户建立的任意严格 1:1 私信
- 如果不存在正常的私信，则创建新的私信房间并重写 `m.direct`

该流程不会自动删除旧房间。它会选择正常的私信并更新映射，以便未来的 Matrix 发送、验证通知和其他私信流程以正确的房间为目标。

## Exec 审批

Matrix 可以充当原生审批客户端。在 `channels.matrix.execApprovals` 下配置（或使用 `channels.matrix.accounts.<account>.execApprovals` 进行按账户覆盖）：

- `enabled`：通过 Matrix 原生提示传递审批。未设置或设为 `"auto"` 时，只要能够解析出至少一名审批者，就会自动启用；设置为 `false` 可显式禁用。
- `approvers`：允许审批 Exec 请求的 Matrix 用户 ID（`@owner:example.org`）。回退到 `channels.matrix.dm.allowFrom`。
- `target`：提示的发送位置。`"dm"`（默认）发送到审批者的私信；`"channel"` 发送到发起请求的房间或私信；`"both"` 同时发送到两者。
- `agentFilter` / `sessionFilter`：用于限定哪些智能体/会话触发 Matrix 传递的可选允许列表。

不同审批类型的授权略有不同：

- **Exec 审批**使用 `execApprovals.approvers`，并回退到 `dm.allowFrom`。
- **插件审批**仅通过 `dm.allowFrom` 授权。

两种审批共用 Matrix 表情回应快捷方式和消息更新。审批者会在主要审批消息上看到以下表情回应快捷方式：

- ✅ 允许一次
- ❌ 拒绝
- ♾️ 始终允许（当有效的 Exec 策略允许时）

备用斜杠命令：`/approve <id> allow-once`、`/approve <id> allow-always`、`/approve <id> deny`。

只有已解析的审批者才能批准或拒绝。Exec 审批的渠道投递内容包含命令文本——仅在受信任的房间中启用 `channel` 或 `both`。

相关内容：[Exec 审批](/zh-CN/tools/exec-approvals)。

## 斜杠命令

斜杠命令（`/new`、`/reset`、`/model`、`/focus`、`/unfocus`、`/agents`、`/session`、`/acp`、`/approve` 等）可直接在私信中使用。在房间中，OpenClaw 也会识别以机器人自身 Matrix 提及开头的命令，因此 `@bot:server /new` 无需自定义提及正则表达式即可触发命令路径——这样，当用户先通过 Tab 补全机器人再输入命令时，机器人仍能响应 Element 和类似客户端发送的房间样式 `@mention /command` 帖子。

授权规则仍然适用：命令发送者必须满足与普通消息相同的私信或房间允许列表/所有者策略。

## 多账户

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

**继承：**

- 除非账户覆盖，否则顶层 `channels.matrix` 值会作为命名账户的默认值。
- 使用 `groups.<room>.account` 将继承的房间条目限定到特定账户。不含 `account` 的条目由所有账户共享；在顶层配置默认账户时，`account: "default"` 仍然有效。

**默认账户选择：**

- 设置 `defaultAccount`，选择隐式路由、探测和 CLI 命令优先使用的命名账户。
- 如果你有多个账户，且其中一个名称正好是 `default`，即使未设置 `defaultAccount`，OpenClaw 也会隐式使用该账户。
- 如果存在多个命名账户但未选择默认账户，CLI 命令会拒绝猜测——请设置 `defaultAccount` 或传入 `--account <id>`。
- 仅当顶层 `channels.matrix.*` 块的身份验证信息完整时（`homeserver` + `accessToken`，或 `homeserver` + `userId` + `password`），才会将其视为隐式 `default` 账户。缓存的凭据足以完成身份验证后，仍可通过 `homeserver` + `userId` 发现命名账户。

**提升：**

- 当 OpenClaw 在修复或设置期间将单账户配置提升为多账户配置时，如果现有命名账户存在，或 `defaultAccount` 已指向某个账户，则会保留该账户。只有 Matrix 身份验证/引导键会移入提升后的账户；共享投递策略键仍保留在顶层。

有关共享多账户模式，请参阅[配置参考](/zh-CN/gateway/config-channels#multi-account-all-channels)。

## 私有/LAN 主服务器

默认情况下，为防止 SSRF，OpenClaw 会阻止私有/内部 Matrix 主服务器，除非你为各账户明确选择启用。

如果你的主服务器运行在 localhost、LAN/Tailscale IP 或内部主机名上，请为该账户启用 `network.dangerouslyAllowPrivateNetwork`：

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
      accessToken: "syt_internal_xxx",
    },
  },
}
```

CLI 设置示例：

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

此选择启用仅允许受信任的私有/内部目标。`http://matrix.example.org:8008` 等公共明文主服务器仍会被阻止。应尽可能优先使用 `https://`。

## 代理 Matrix 流量

如果你的 Matrix 部署需要显式出站 HTTP(S) 代理，请设置 `channels.matrix.proxy`：

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

命名账户可以使用 `channels.matrix.accounts.<id>.proxy` 覆盖顶层默认值。OpenClaw 对运行时 Matrix 流量和账户状态探测使用相同的代理设置。

## 目标解析

在 OpenClaw 要求提供房间或用户目标的任何位置，Matrix 都接受以下目标形式：

- 用户：`@user:server`、`user:@user:server` 或 `matrix:user:@user:server`
- 房间：`!room:server`、`room:!room:server` 或 `matrix:room:!room:server`
- 别名：`#alias:server`、`channel:#alias:server` 或 `matrix:channel:#alias:server`

Matrix 房间 ID 区分大小写。配置显式投递目标、定时任务、绑定或允许列表时，请使用 Matrix 中房间 ID 的确切大小写。OpenClaw 会规范化内部会话键以供存储，因此这些小写键不能作为 Matrix 投递 ID 的可靠来源。

实时目录查找使用已登录的 Matrix 账户：

- 用户查找会查询该主服务器上的 Matrix 用户目录。
- 房间查找直接接受显式房间 ID 和别名。已加入房间的名称查找采用尽力而为方式，并且仅在设置 `dangerouslyAllowNameMatching: true` 时适用于运行时房间允许列表。
- 如果无法将房间名称解析为 ID 或别名，运行时允许列表解析会忽略该名称。

## 配置参考

允许列表样式的用户字段（`groupAllowFrom`、`dm.allowFrom`、`groups.<room>.users`）接受完整的 Matrix 用户 ID（最安全）。默认情况下会忽略非 ID 条目。如果设置了 `dangerouslyAllowNameMatching: true`，则会在启动时，以及监视器运行期间允许列表发生变化时，解析 Matrix 目录中显示名称完全匹配的条目；运行时会忽略无法解析的条目。

房间允许列表键（`groups`、旧版 `rooms`）应为房间 ID 或别名。默认情况下会忽略纯房间名称键；`dangerouslyAllowNameMatching: true` 会恢复针对已加入房间名称的尽力而为查找。

### 账户和连接

- `enabled`：启用或禁用该渠道。
- `name`：账户的可选显示标签。
- `defaultAccount`：配置多个 Matrix 账户时的首选账户 ID。
- `accounts`：按命名账户进行的覆盖。顶层 `channels.matrix` 值会作为默认值继承。
- `homeserver`：主服务器 URL，例如 `https://matrix.example.org`。
- `network.dangerouslyAllowPrivateNetwork`：允许此账户连接到 `localhost`、LAN/Tailscale IP 或内部主机名。
- `proxy`：Matrix 流量的可选 HTTP(S) 代理 URL。支持按账户覆盖。
- `userId`：完整的 Matrix 用户 ID（`@bot:example.org`）。
- `accessToken`：用于基于令牌的身份验证的访问令牌。env/file/exec 提供商均支持明文和 SecretRef 值（[机密管理](/zh-CN/gateway/secrets)）。
- `password`：用于基于密码登录的密码。支持明文和 SecretRef 值。
- `deviceId`：显式 Matrix 设备 ID。
- `deviceName`：密码登录时使用的设备显示名称。
- `avatarUrl`：用于个人资料同步和 `profile set` 更新的已存储个人头像 URL。
- `initialSyncLimit`：启动同步期间获取的最大事件数。

### 加密

- `encryption`：启用 E2EE。默认值：`false`。
- `startupVerification`：`"if-unverified"`（启用 E2EE 时的默认值）或 `"off"`。当此设备未经验证时，启动时自动请求自我验证。
- `startupVerificationCooldownHours`：下次启动时自动请求前的冷却时间。默认值：`24`。

### 访问和策略

- `groupPolicy`：`"open"`、`"allowlist"` 或 `"disabled"`。默认值：`"allowlist"`。
- `groupAllowFrom`：房间流量的用户 ID 允许列表。
- `mentionPatterns`：房间提及的限定范围正则表达式模式。包含 `{ mode: "allow"|"deny", allowIn: [roomId, ...], denyIn: [roomId, ...] }` 的对象。控制配置的 `agents.entries.*.groupChat.mentionPatterns` 是否按房间应用。
- `dm.enabled`：当为 `false` 时，忽略所有私信。默认值：`true`。
- `dm.policy`：`"pairing"`（默认）、`"allowlist"`、`"open"` 或 `"disabled"`。在机器人加入房间并将其归类为私信后应用；不影响邀请处理。
- `dm.allowFrom`：私信流量的用户 ID 允许列表。
- `dm.sessionScope`：`"per-user"`（默认）或 `"per-room"`。
- `dm.threadReplies`：仅用于私信的回复线程覆盖（`"off"`、`"inbound"`、`"always"`）。
- `allowBots`：接受来自其他已配置 Matrix 机器人账户的消息（`true` 或 `"mentions"`）。
- `allowlistOnly`：当为 `true` 时，强制所有活动私信策略（`"disabled"` 除外）以及 `"open"` 群组策略采用 `"allowlist"`。不会更改 `"disabled"` 策略。
- `dangerouslyAllowNameMatching`：当为 `true` 时，允许为用户允许列表条目进行 Matrix 显示名称目录查找，并为房间允许列表键查找已加入的房间名称。应优先使用完整的 `@user:server` ID，以及房间 ID 或别名。
- `autoJoin`：`"always"`、`"allowlist"` 或 `"off"`。默认值：`"off"`。适用于每个 Matrix 邀请，包括私信样式的邀请。
- `autoJoinAllowlist`：当 `autoJoin` 为 `"allowlist"` 时允许的房间/别名。别名条目会针对主服务器进行解析，而不是依据受邀房间声称的状态进行解析。
- `contextVisibility`：补充上下文可见性（`"all"` 为默认值，`"allowlist"`、`"allowlist_quote"`）。

### 回复行为

- `replyToMode`: `"off"`（默认）、`"first"`、`"all"` 或 `"batched"`。
- `threadReplies`: `"off"`（除非显式设置，否则顶层默认值解析为 `"inbound"`）、`"inbound"` 或 `"always"`。
- `threadBindings`: 用于线程绑定会话路由和生命周期的各渠道覆盖设置。
- `streaming`: 嵌套对象 `{ mode, chunkMode, block: { enabled, coalesce }, preview: { toolProgress }, progress: { label, labels, maxLines, maxLineChars, toolProgress } }`。`mode` 为 `"off"`（默认）、`"partial"`、`"quiet"` 或 `"progress"`。旧版标量/布尔值写法通过 `openclaw doctor --fix` 迁移。
- `streaming.block.enabled`: 当 `true` 时，已完成的助手内容块将保留为独立的进度消息。默认值：`false`。
- `markdown`: 用于出站文本的可选 Markdown 渲染配置。
- `responsePrefix`: 添加到出站回复开头的可选字符串。
- `textChunkLimit`: 当 `streaming.chunkMode: "length"` 时，以字符数计的出站分块大小。默认值：`4000`。
- `streaming.chunkMode`: `"length"`（默认，按字符数拆分）或 `"newline"`（在行边界处拆分）。
- `historyLimit`: 当房间消息触发智能体时，作为 `InboundHistory` 包含的近期房间消息数量。回退到 `messages.groupChat.historyLimit`；实际默认值为 `0`（已禁用）。
- `mediaMaxMb`: 出站发送和入站处理的媒体大小上限（以 MB 为单位）。默认值：`20`。

### 表情回应设置

- `ackReaction`: 此渠道/账户的确认表情回应覆盖设置。
- `ackReactionScope`: 范围覆盖设置（默认 `"group-mentions"`、`"group-all"`、`"direct"`、`"all"`、`"none"`、`"off"`）。
- `reactionNotifications`: 入站表情回应通知模式（默认 `"own"`、`"off"`）。

### 工具和各房间覆盖设置

- `actions`: 按操作控制工具使用权限（`messages`、`reactions`、`pins`、`profile`、`memberInfo`、`channelInfo`、`verification`）。
- `groups`: 各房间策略映射。解析后，会话标识使用稳定的房间 ID。（`rooms` 是旧版别名。）
  - `groups.<room>.account`: 将一个继承的房间条目限制到特定账户。
  - `groups.<room>.enabled`: 各房间开关。当 `false` 时，该房间将被忽略，如同它不在映射中一样。
  - `groups.<room>.requireMention`: 对渠道级提及要求的各房间覆盖设置。
  - `groups.<room>.allowBots`: 对渠道级设置的各房间覆盖（`true` 或 `"mentions"`）。
  - `groups.<room>.botLoopProtection`: 对机器人间循环防护预算的各房间覆盖设置。
  - `groups.<room>.users`: 各房间发送者允许列表。
  - `groups.<room>.tools`: 各房间工具允许/拒绝覆盖设置。
  - `groups.<room>.autoReply`: 各房间提及门控覆盖设置。`true` 禁用该房间的提及要求；`false` 强制重新启用。
  - `groups.<room>.skills`: 各房间 Skills 筛选器。
  - `groups.<room>.systemPrompt`: 各房间系统提示词片段。

### Exec 审批设置

- `execApprovals.enabled`: 通过 Matrix 原生提示传递 Exec 审批。
- `execApprovals.approvers`: 允许进行审批的 Matrix 用户 ID。回退到 `dm.allowFrom`。
- `execApprovals.target`: `"dm"`（默认）、`"channel"` 或 `"both"`。
- `execApprovals.agentFilter` / `execApprovals.sessionFilter`: 用于传递的可选智能体/会话允许列表。

## 相关内容

- [渠道概览](/zh-CN/channels) - 所有支持的渠道
- [配对](/zh-CN/channels/pairing) - 私信身份验证和配对流程
- [群组](/zh-CN/channels/groups) - 群聊行为和提及门控
- [频道路由](/zh-CN/channels/channel-routing) - 消息的会话路由
- [安全性](/zh-CN/gateway/security) - 访问模型和安全加固
