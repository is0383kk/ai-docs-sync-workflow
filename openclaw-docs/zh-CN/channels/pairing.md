---
read_when:
    - 设置私信访问控制
    - 配对新的 iOS/Android 节点
    - 审查 OpenClaw 的安全状况
summary: 配对概览：批准谁可以向你发送私信，以及哪些节点可以加入
title: 配对
x-i18n:
    generated_at: "2026-07-26T06:07:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dc874d660509f59bc26795c8b3ce13f5d238cd61154c717637f5d545b995fb08
    source_path: channels/pairing.md
    workflow: 16
---

“配对”是 OpenClaw 的显式访问审批步骤。
它用于两个方面：

1. **私信配对**（允许谁与 Bot 对话）
2. **节点配对**（允许哪些设备/节点加入 Gateway 网关网络）

安全上下文：[安全](/zh-CN/gateway/security)

## 1）私信配对（入站聊天访问）

当渠道配置了私信策略 `pairing` 时，未知发送者会收到一个短代码，在你批准之前，其消息**不会被处理**。

默认私信策略记录于：[安全](/zh-CN/gateway/security)

仅当生效的私信允许列表包含 `"*"` 时，`dmPolicy: "open"` 才是公开的。
设置和验证要求公开开放配置包含该通配符。如果现有
状态包含 `open` 和具体的 `allowFrom` 条目，运行时仍然仅允许
这些发送者，并且配对存储中的批准不会扩大 `open` 访问权限。

配对代码：

- 8 个字符、大写，不含易混淆字符（`0O1I`）。
- **1 小时后过期**。Bot 仅在创建新请求时发送配对消息（每位发送者大约每小时一次）。
- 待处理的私信配对请求上限为**每个渠道账号 3 个**；在其中一个请求过期或获批之前，其他请求将被忽略。

### 从 Control UI 批准

打开 **Settings → Channels → DM access requests**。该队列汇总了所有已配置渠道账号中，私信策略为 `pairing` 的待处理
请求。
按渠道或账号筛选，检查发送者 ID 和元数据，然后选择
**Approve**。

批准仅授予直接消息访问权限，不授予群组访问权限。在支持的情况下，
批准对话框还提供以下显式选项：

- **批准后通知请求者**
- **同时将此发送者设为首位命令所有者**，仅当不存在命令
  所有者且 Control UI 会话具有 `operator.admin` 时显示

选择 **Dismiss** 可移除待处理请求而不批准。移除
并非永久封禁；发送者稍后可以再次请求访问权限。

### 从 CLI 批准

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

添加 `--notify` 可通过同一渠道通知请求者。多账号渠道
使用 `--account <id>`。

与 Control UI 中的显式复选框不同，当未配置命令所有者时，CLI 会自动引导设置
`commands.ownerAllowFrom`，使用类似
`telegram:123456789` 的条目。这会为首次设置提供一位显式所有者，
用于特权命令和 Exec 审批提示。存在所有者后，后续
配对批准仅授予私信访问权限，不会添加更多所有者。

<Note>
WhatsApp 的登录二维码会将 WhatsApp 账号关联到 OpenClaw。私信访问请求
用于批准向该账号发送消息的人员。这是两个独立的流程。
</Note>

支持的渠道（任何声明配对功能的已安装渠道插件；`openclaw-weixin` 等外部插件可以添加更多渠道）：`discord`、`feishu`、`googlechat`、`imessage`、`irc`、`line`、`matrix`、`mattermost`、`msteams`、`nextcloud-talk`、`nostr`、`signal`、`slack`、`sms`、`synology-chat`、`telegram`、`twitch`、`whatsapp`、`zalo`、`zalouser`。

### 可复用的发送者组

当同一组受信任发送者应适用于
多个消息渠道，或同时适用于私信和群组允许列表时，请使用顶层 `accessGroups`。

静态组使用 `type: "message.senders"`，并通过渠道允许列表中的
`accessGroup:<name>` 引用：

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
        whatsapp: ["+15551234567"],
      },
    },
  },
  channels: {
    telegram: { dmPolicy: "allowlist", allowFrom: ["accessGroup:operators"] },
    whatsapp: { groupPolicy: "allowlist", groupAllowFrom: ["accessGroup:operators"] },
  },
}
```

访问组的详细文档参见：[访问组](/zh-CN/channels/access-groups)

### 状态的存储位置

存储在共享 SQLite 状态数据库
`~/.openclaw/state/openclaw.sqlite` 中：

- `channel_pairing_requests` 中的待处理请求
- `channel_pairing_allow_entries` 中的已批准发送者

账号作用域行为：

- 每个请求和已批准发送者均按渠道和账号作为键
- 运行时仅读取规范 SQLite 行，不会合并旧版文件

旧版 Gateway 网关曾在 `~/.openclaw/credentials/` 下写入 `<channel>-pairing.json` 和
`<channel>-<accountId>-allowFrom.json`。
启动迁移和 `openclaw doctor --fix` 会将这些文件导入 SQLite，并在
成功导入后移除各个源文件。请将 SQLite 数据库视为
敏感数据，因为这些行控制着对你的助手的访问权限。

<Note>
配对允许列表存储用于私信访问。群组授权与其相互独立。
批准私信配对代码不会自动允许该发送者在群组中运行
命令或控制 Bot。首位所有者引导设置是 `commands.ownerAllowFrom` 中单独的配置
状态，群聊消息传递仍遵循
渠道的群组允许列表（例如 `groupAllowFrom`、`groups`，或由渠道决定的按群组
或按话题覆盖配置）。
</Note>

## 2）节点设备配对（iOS/Android/macOS/无界面节点）

节点以具有 `role: node` 的**设备**身份连接到 Gateway 网关。Gateway 网关
会创建设备配对请求，且必须获得批准。

### 从 Control UI 配对（推荐）

使用已连接且具有 `operator.admin` 访问权限的 Control UI 会话：

1. 打开 Control UI 并前往 **Settings → Devices**。
2. 在 **Devices** 页面上，点击 **Pair mobile device**。
3. 保留 **Full access (recommended)**，或选择 **Limited access** 以省略
   Gateway 网关管理控制权限。
4. 点击 **Create setup code**。
5. 在手机上打开 OpenClaw 应用 → **Settings** → **Gateway**。
6. 扫描二维码或粘贴设置代码，然后连接。

当官方 OpenClaw iOS 和 Android 应用的
设置代码元数据匹配时，会自动获得批准。如果 **Pending approval** 显示请求（
例如来自非官方客户端或元数据不匹配），请在批准前检查其角色和
权限范围。

当当前 Control UI 会话没有
管理员访问权限时，该按钮会被禁用。在这种情况下，请从 Gateway 网关主机使用下方的 CLI 批准流程。

### 通过 Telegram 配对

如果使用 `device-pair` 插件，可以完全通过 Telegram 完成首次设备配对：

1. 在 Telegram 中向你的 Bot 发送消息：`/pair`
2. Bot 会回复两条消息：一条说明消息和一条单独的**设置代码**消息（便于在 Telegram 中复制/粘贴）。
3. 在手机上打开 OpenClaw iOS 应用 → Settings → Gateway。
4. 扫描二维码（`/pair qr`）或粘贴设置代码，然后连接。
5. 官方移动应用会自动连接。如果 `/pair pending` 显示
   请求，请在批准前检查其角色和权限范围。

设置代码是一个经过 base64 编码的 JSON 有效负载，其中包含：

- `url`：Gateway 网关 WebSocket URL（`ws://...` 或 `wss://...`）
- `urls`：可用时，移动应用可以按顺序尝试的 LAN/Tailnet 路由
- `bootstrapToken`：用于初始配对握手的一次性引导令牌；Gateway 网关会在 10 分钟后使其过期

配对完成后运行 `/pair cleanup`，使未使用的设置代码失效。

该引导令牌携带内置的配对引导配置：

- 安全的 `wss://` 设置（或同主机环回）默认使用 `node`，并具有完整的
  原生移动端 `operator` 访问权限
- 移交的 `node` 令牌保持为 `scopes: []`
- 默认移交的 `operator` 令牌包含 `operator.admin`、
  `operator.approvals`、`operator.read`、`operator.talk.secrets` 和
  `operator.write`
- Control UI **Limited access** 和 `openclaw qr --limited` 会省略
  `operator.admin`，同时保留其他操作员权限范围
- 明文 LAN `ws://` 设置会自动使用相同的受限配置；
  配置 `wss://` 或 Tailscale Serve，并生成新代码以获得完整访问权限
- 后续令牌轮换/撤销仍同时受设备已批准的
  角色契约和调用者会话的操作员权限范围约束

设置代码有效期间，请像对待密码一样对待它。

iOS 和 Android 的 **Settings → Gateway** 页面会显示 **Full** 或 **Limited**
访问权限。要升级访问受限的手机，请先配置安全的 `wss://` 或
Tailscale Serve 路由，然后生成新的完整访问设置代码，在该设置页面中扫描或粘贴
该代码，并重新连接。

对于 Tailscale、公共或其他远程移动设备配对，请使用 Tailscale Serve/Funnel
或其他 `wss://` Gateway 网关 URL。明文 `ws://` 设置代码仅接受
环回、私有 LAN 地址、`.local` Bonjour 主机和 Android
模拟器主机。非环回明文路由会获得受限访问权限。在签发
二维码/设置代码之前，Tailnet CGNAT 地址、`.ts.net` 名称和公共主机仍会以失败关闭方式处理。

对于 `gateway.bind=lan` 设置 URL，OpenClaw 会检测代理当前 Gateway 网关
环回端口的持久 Tailscale Serve HTTPS 根地址，并将其与 LAN 路由
一起公布。设置命令仅为 `lan` 添加此回退；
`custom` 和 `tailnet` 保留其显式公布的路由。
iOS 应用会按顺序探测公布的路由，并保存第一个可访问的
端点。

### 批准节点设备

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

如果显式批准因执行批准操作的已配对设备会话
仅以配对权限范围打开而被拒绝，CLI 会使用
`operator.admin` 重试同一请求。这使现有具备管理员能力的已配对设备能够恢复新的
Control UI/浏览器配对，而无需手动编辑配对存储。Gateway 网关仍会验证
重试的连接；无法使用 `operator.admin` 完成身份验证的令牌仍会被阻止。

如果同一设备使用不同的身份验证详情重试（例如不同的
角色/权限范围/公钥），先前的待处理请求会被取代，并创建新的
`requestId`。

<Note>
已配对设备不会悄然获得更广泛的访问权限。如果它重新连接并请求更多权限范围或更广泛的角色，OpenClaw 会保持现有批准不变，并创建新的待处理升级请求。在批准前，请使用 `openclaw devices list` 比较当前已批准的访问权限与新请求的访问权限。
</Note>

### 可选的受信任 CIDR 节点自动批准

设备配对默认仍需手动完成。对于受到严格控制的节点网络，
可以通过显式 CIDR 或确切 IP 选择启用首次节点自动批准：

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

这仅适用于未请求权限范围的新 `role: node` 配对请求。
操作员、浏览器、Control UI 和 WebChat 客户端仍需手动
批准。角色、权限范围、元数据和公钥变更仍需手动
批准。

### 节点配对状态存储

存储在 `~/.openclaw/state/openclaw.sqlite` 的共享 SQLite 状态数据库中：

- 待处理的设备配对请求（短期有效；5 分钟后过期）
- 已配对设备和令牌

旧版 Gateway 网关将此状态保存在 `~/.openclaw/devices/*.json` 中；这些文件会在 Gateway 网关启动时导入 SQLite，并以 `.migrated` 后缀归档。

### 注意事项

- `node.pair.*` API（CLI：`openclaw nodes pending|approve|reject|remove|rename`）管理存储在同一已配对设备记录中的节点能力审批。WS 节点仍需进行设备配对；请参阅[节点配对](/zh-CN/gateway/pairing)。
- 配对记录是已批准角色的持久事实来源。有效的设备令牌始终受限于该已批准角色集合；已批准角色之外的孤立令牌条目不会创建新的访问权限。

## 相关文档

- 安全模型 + 提示词注入：[安全](/zh-CN/gateway/security)
- 安全更新（运行 Doctor）：[更新](/zh-CN/install/updating)
- 渠道配置：
  - Telegram：[Telegram](/zh-CN/channels/telegram)
  - WhatsApp：[WhatsApp](/zh-CN/channels/whatsapp)
  - Signal：[Signal](/zh-CN/channels/signal)
  - iMessage：[iMessage](/zh-CN/channels/imessage)
  - Discord：[Discord](/zh-CN/channels/discord)
  - Slack：[Slack](/zh-CN/channels/slack)
