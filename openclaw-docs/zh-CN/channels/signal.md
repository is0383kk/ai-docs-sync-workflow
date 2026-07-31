---
read_when:
    - 设置 Signal 支持
    - 调试 Signal 消息收发
summary: 通过 signal-cli（原生守护进程或 bbernhard 容器）支持 Signal，以及设置路径和号码模型
title: Signal
x-i18n:
    generated_at: "2026-07-26T05:39:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 744f817e425d378e9f3e160df534019a6fc865227eb3fc68959a12ad46c0b714
    source_path: channels/signal.md
    workflow: 16
---

Signal 是一个可下载的渠道插件（`@openclaw/signal`）。Gateway 网关通过 HTTP 与 `signal-cli` 通信：可以使用原生守护进程（JSON-RPC + SSE），也可以使用 [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) 容器（REST + WebSocket）。OpenClaw 不内嵌 libsignal。

## 号码模型（请先阅读）

- Gateway 网关连接到一个 **Signal 设备**：即 `signal-cli` 账户。
- 在**你的个人 Signal 账户**上运行机器人会使其忽略你自己的消息（循环保护）。
- 如需实现“我给机器人发短信，它会回复”，请使用一个**单独的机器人号码**。

## 安装

```bash
openclaw plugins install @openclaw/signal
```

仅指定插件名称时，会先尝试 ClawHub，再回退到 npm。使用 `openclaw plugins install clawhub:@openclaw/signal` 或 `npm:@openclaw/signal` 强制指定来源。`plugins install` 会注册并启用插件；无需另行执行 `enable` 步骤。有关通用安装规则，请参阅[插件](/zh-CN/tools/plugin)。

## 快速设置

<Steps>
  <Step title="选择号码">
    为机器人使用一个**单独的 Signal 号码**（推荐）。
  </Step>
  <Step title="安装插件">
    ```bash
    openclaw plugins install @openclaw/signal
    ```
  </Step>
  <Step title="运行引导式设置">
    ```bash
    openclaw channels add
    ```
    向导会检测 `signal-cli` 是否位于 `PATH` 中；如果缺失，则会提供安装选项：在 Linux x86-64 上下载官方原生 GraalVM 构建，或者在 macOS 和其他架构上通过 Homebrew 安装。随后，它会提示输入机器人号码和 `signal-cli` 路径。

    对于非交互式设置，`openclaw channels add --channel signal` 还接受用于指定机器人电话号码的 `--signal-number <e164>`，以及用于指定 Signal 守护进程端点的 `--http-host <host>` 和 `--http-port <port>`（默认为 `127.0.0.1:8080`）。

  </Step>
  <Step title="关联或注册账户">
    - **二维码关联（最快）：** `signal-cli link -n "OpenClaw"`，然后使用 Signal 扫描。请参阅[路径 A](#setup-path-a-link-existing-signal-account-qr)。
    - **短信注册：** 使用专用号码，通过验证码 + 短信验证完成注册。请参阅[路径 B](#setup-path-b-register-dedicated-bot-number-sms-linux)。

  </Step>
  <Step title="验证并配对">
    ```bash
    openclaw gateway call channels.status --params '{"probe":true}'
    ```
    发送第一条私信并批准配对：`openclaw pairing approve signal <CODE>`。
  </Step>
</Steps>

最小配置：

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "managed-native",
        cliPath: "signal-cli",
      },
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

| 字段        | 说明                                              |
| ----------- | ------------------------------------------------- |
| `account`   | E.164 格式的机器人电话号码（`+15551234567`） |
| `transport` | 账户拥有的 Signal 连接和进程模式                  |
| `dmPolicy`  | 私信访问策略（推荐 `pairing`）           |
| `allowFrom` | 允许发送私信的电话号码或 `uuid:<id>` 值   |

多账户支持：使用 `channels.signal.accounts` 配置各个账户，并可选择设置 `name`。每个命名账户拥有自己的 `transport`；它不会继承顶层传输配置。顶层传输配置仅属于隐式的 `default` 账户。有关通用模式，请参阅[多账户渠道](/zh-CN/gateway/config-channels#multi-account-all-channels)。

## 功能说明

- 确定性路由：回复始终返回 Signal。
- 私信共享智能体的主会话；群组相互隔离（`agent:<agentId>:signal:group:<groupId>`）。
- 默认情况下，Signal 可以写入由 `/config set|unset` 触发的配置更新（需要 `commands.config: true`）。使用 `channels.signal.configWrites: false` 可将其禁用。

## 设置路径 A：关联现有 Signal 账户（二维码）

1. 安装 `signal-cli`（JVM 或原生构建），或者让 `openclaw channels add` 为你安装。
2. 关联机器人账户：运行 `signal-cli link -n "OpenClaw"`，然后在 Signal 中扫描二维码。
3. 配置 Signal 并启动 Gateway 网关。

## 设置路径 B：注册专用机器人号码（短信，Linux）

使用此方法注册专用机器人号码，而不是关联现有的 Signal 应用账户。以下流程已在 Ubuntu 24 上测试。

1. 获取一个能够接收短信的号码（固定电话也可使用语音验证）。专用机器人号码可避免账户/会话冲突。
2. 在 Gateway 网关主机上安装 `signal-cli`：

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/
signal-cli --version
```

如果使用 JVM 构建（`signal-cli-${VERSION}.tar.gz`），请先安装 JRE。请保持 `signal-cli` 为最新版本；上游说明指出，随着 Signal 服务器 API 发生变化，旧版本可能会停止工作。

3. 注册并验证号码：

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register
```

如果需要验证码（完成此步骤需要访问浏览器）：

1. 打开 `https://signalcaptchas.org/registration/generate.html`。
2. 完成验证码，然后从 “Open Signal” 复制 `signalcaptcha://...` 链接目标。
3. 如果可能，请从与浏览器会话相同的外部 IP 运行（验证码令牌会很快过期）。
4. 立即注册并验证：

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register --captcha '<SIGNALCAPTCHA_URL>'
signal-cli -a +<BOT_PHONE_NUMBER> verify <VERIFICATION_CODE>
```

4. 配置 OpenClaw，重启 Gateway 网关，然后验证渠道：

```bash
# 如果以用户 systemd 服务的形式运行 Gateway 网关：
systemctl --user restart openclaw-gateway.service

# 然后验证：
openclaw doctor
openclaw channels status --probe
```

5. 配对你的私信发送方：
   - 向机器人号码发送任意消息。
   - 在服务器上批准：`openclaw pairing approve signal <PAIRING_CODE>`。
   - 将机器人号码保存为手机联系人，以避免出现 “Unknown contact”。

<Warning>
使用 `signal-cli` 注册电话号码账户可能会使该号码的 Signal 主应用会话失去身份验证。建议使用专用机器人号码，或者使用二维码关联模式以保留现有的手机应用设置。
</Warning>

上游参考资料：

- `signal-cli` README：`https://github.com/AsamK/signal-cli`
- 验证码流程：`https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha`
- 关联流程：`https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning)`

## 外部原生守护进程模式

如需自行管理 `signal-cli`（JVM 冷启动缓慢、容器初始化、共享 CPU），请单独运行守护进程，并将 OpenClaw 指向该进程：

对于非交互式设置，请在需要时显式选择端点类型：

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://127.0.0.1:8080 --signal-transport external-native
```

```json5
{
  channels: {
    signal: {
      transport: {
        kind: "external-native",
        url: "http://127.0.0.1:8080",
      },
    },
  },
}
```

这会跳过自动启动以及 OpenClaw 的启动等待。对于启动缓慢的托管守护进程，请设置 `channels.signal.transport.startupTimeoutMs`。

## 容器模式（bbernhard/signal-cli-rest-api）

你可以不以原生方式运行 `signal-cli`，而是使用 [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) Docker 容器；该容器通过 REST + WebSocket 接口封装 `signal-cli`。

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://signal-cli:8080 --signal-transport container
```

要求：

- 容器**必须**使用 `MODE=json-rpc` 运行，才能实时接收消息。
- 在连接 OpenClaw 之前，先在容器中注册或关联你的 Signal 账户。

`docker-compose.yml` 服务示例：

```yaml
signal-cli:
  image: bbernhard/signal-cli-rest-api:latest
  environment:
    MODE: json-rpc
  ports:
    - "8080:8080"
  volumes:
    - signal-cli-data:/home/.local/share/signal-cli
```

OpenClaw 配置：

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "container",
        url: "http://signal-cli:8080",
      },
    },
  },
}
```

`transport.kind` 控制 OpenClaw 使用的协议和进程生命周期：

| 值                  | 行为                                                                                                                                                         |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `"managed-native"`  | 启动原生 signal-cli，并在 `/api/v1/rpc` 使用 JSON-RPC、在 `/api/v1/events` 使用 SSE；`url` 可以选择一个不同于守护进程绑定地址的连接端点 |
| `"external-native"` | 连接到已在运行的原生 signal-cli 守护进程                                                                                                                     |
| `"container"`       | 连接到 `/v2/send` 上的 bbernhard REST 和 `/v1/receive/{account}` 上的 WebSocket                                                                            |

设置和 `openclaw doctor --fix` 可能会探测一次现有端点，以识别其具体类型。运行时操作不会自动检测或切换协议。

当容器公开相应 API 时，容器模式支持与原生模式相同的 Signal 操作：发送、接收、附件、输入状态指示器、已读/已查看回执、表情回应、群组和带样式文本。OpenClaw 会将原生 Signal RPC 调用转换为容器的 REST 载荷，包括 `group.{base64(internal_id)}` 群组 ID 和用于格式化文本的 `text_mode: "styled"`。

操作说明：

- 请使用 `MODE=json-rpc` 接收消息。`MODE=normal` 可能会让 `/v1/about` 看起来运行正常，但 `/v1/receive/{account}` 不会升级到 WebSocket，因此容器接收流的探测会失败。
- 将 `kind: "container"` 用于 bbernhard REST API，将 `kind: "external-native"` 用于原生 `signal-cli` JSON-RPC/SSE。
- 容器附件下载采用与原生模式相同的媒体字节限制。当服务器发送 `Content-Length` 时，过大的响应会在完全缓冲前被拒绝；否则会在流式传输期间拒绝。

## 访问控制（私信 + 群组）

私信：

- 默认值：`channels.signal.dmPolicy = "pairing"`。
- 未知发送方会收到配对码；在获得批准前，其消息会被忽略（配对码在 1 小时后过期）。
- 通过 `openclaw pairing list signal` 和 `openclaw pairing approve signal <CODE>` 批准。
- 配对是 Signal 私信的默认令牌交换机制。详情请参阅：[配对](/zh-CN/channels/pairing)
- 仅有 UUID 的发送方（来自 `sourceUuid`）会在 `channels.signal.allowFrom` 中存储为 `uuid:<id>`。

群组：

- `channels.signal.groupPolicy = open | allowlist | disabled`。
- `channels.signal.groupAllowFrom` 控制在设置 `allowlist` 时，哪些群组或发送者可以触发群组回复；条目可以是 Signal 群组 ID（原始格式、`group:<id>` 或 `signal:group:<id>`）、发送者电话号码、`uuid:<id>` 值或 `*`。
- `channels.signal.groups["<group-id>" | "*"]` 可以通过 `requireMention`、`tools` 和 `toolsBySender` 覆盖群组行为。
- 在多账户设置中，使用 `channels.signal.accounts.<id>.groups` 进行各账户覆盖。
- 通过 `groupAllowFrom` 将 Signal 群组加入允许列表，本身不会禁用提及门控。除非设置了 `requireMention=true`，否则专门配置的 `channels.signal.groups["<group-id>"]` 条目会处理每条群组消息。
- 使用 `requireMention=true` 时，系统会根据结构化提及元数据，将 Signal 原生 @提及与 Bot 账户电话号码或 `accountUuid` 进行匹配。配置的 `mentionPatterns` 仍作为纯文本回退方案。
- 运行时说明：如果完全缺少 `channels.signal`，运行时会回退到 `groupPolicy="allowlist"` 进行群组检查（即使设置了 `channels.defaults.groupPolicy`）。

具有有限上下文的提及门控群组：

```json5
{
  channels: {
    signal: {
      account: "+15551234567",
      accountUuid: "bot-signal-uuid",
      groupPolicy: "allowlist",
      groupAllowFrom: ["group:<signal-group-id>"],
      historyLimit: 8,
      groups: {
        "<signal-group-id>": { requireMention: true },
      },
    },
  },
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b"],
    },
  },
}
```

允许的群组消息如果未提及 Bot，将保持静默，并且仅保留在有限的待处理历史窗口中。当后续的原生 @提及或回退文本提及触发 Bot 时，OpenClaw 会包含这些近期上下文，并回复同一群组。被跳过的附件正文不会下载；它们可能仅以精简的媒体占位符形式出现在待处理上下文中。

## 工作原理（行为）

- 原生模式：`signal-cli` 作为守护进程运行；Gateway 网关通过 SSE 读取事件。
- 容器模式：Gateway 网关通过 REST API 发送，并通过 WebSocket 接收。
- 入站消息会规范化为共享渠道信封。
- 回复始终路由回相同的号码或群组。
- 当后端接受入站时间戳和作者时，对入站消息的回复会包含 Signal 原生引用元数据；如果引用元数据缺失或被拒绝，OpenClaw 会将回复作为普通消息发送。
- 使用 `channels.signal.replyToMode = off | first | all | batched` 配置原生引用，或使用 `channels.signal.replyToModeByChatType.direct/group` 进行各聊天类型覆盖。`channels.signal.accounts.<id>` 下的账户级值优先。

## 媒体和限制

- 出站文本会按 `channels.signal.textChunkLimit` 分块（默认值为 4000）。
- 可选的换行分块：设置 `channels.signal.streaming.chunkMode="newline"`，先按空行（段落边界）拆分，再按长度分块。
- 支持附件（从 `signal-cli` 获取 base64）。
- 当缺少 `contentType` 时，语音便笺附件使用 `signal-cli` 文件名作为 MIME 回退，使音频转写仍能识别 AAC 语音备忘录。
- 默认媒体上限：`channels.signal.mediaMaxMb`（默认值为 8）。
- 使用 `channels.signal.ignoreAttachments` 可跳过为任何传输协议下载媒体。
- 群组历史上下文使用 `channels.signal.historyLimit`（或 `channels.signal.accounts.*.historyLimit`），并回退到 `messages.groupChat.historyLimit`。设置 `0` 可禁用（默认值为 50）。

## 输入状态和已读回执

- **输入状态指示器**：OpenClaw 通过 `signal-cli sendTyping` 发送输入状态信号，并在回复运行期间持续刷新。
- **已读回执**：当 `channels.signal.sendReadReceipts` 为 true 时，OpenClaw 会转发已允许私信的已读回执。
- `signal-cli` 不会公开群组的已读回执。

## 生命周期状态回应

设置 `messages.statusReactions.enabled: true`，让 Signal 在入站轮次中显示共享的排队中/思考中/工具/压缩/完成/错误回应生命周期。Signal 使用入站消息时间戳作为回应目标；发送群组回应时，会使用 Signal 群组 ID，并将原始发送者作为目标作者。

状态回应还需要确认回应和匹配的 `messages.ackReactionScope`（`direct`、`group-all`、`group-mentions` 或 `all`）。设置 `channels.signal.reactionLevel: "off"` 可禁用 Signal 状态回应。

在最终完成/错误状态后，Signal 会恢复初始确认回应。

## 回应（消息工具）

将 `message action=react` 与 `channel=signal` 配合使用。

- 目标：发送者 E.164 或 UUID（使用配对输出中的 `uuid:<id>`；也可以使用不带前缀的 UUID）。
- `messageId` 是要回应的消息的 Signal 时间戳。
- 群组回应需要 `targetAuthor` 或 `targetAuthorUuid`。

```text
message action=react channel=signal target=uuid:123e4567-e89b-12d3-a456-426614174000 messageId=1737630212345 emoji=🔥
message action=react channel=signal target=+15551234567 messageId=1737630212345 emoji=🔥 remove=true
message action=react channel=signal target=signal:group:<groupId> targetAuthor=uuid:<sender-uuid> messageId=1737630212345 emoji=✅
```

配置：

- `channels.signal.actions.reactions`：启用/禁用回应操作（默认值为 true）。
- `channels.signal.reactionLevel`：`off | ack | minimal | extensive`（默认为 `minimal`）。
  - `off`/`ack` 会禁用智能体回应（消息工具 `react` 会报错）。
  - `minimal`/`extensive` 会启用智能体回应并设置指导级别。
- 各账户覆盖：`channels.signal.accounts.<id>.actions.reactions`、`channels.signal.accounts.<id>.reactionLevel`。

## 审批回应

Signal Exec 和插件审批提示使用顶层的 `approvals.exec` 和 `approvals.plugin` 路由块。Signal 没有 `channels.signal.execApprovals` 块。

- `👍` 仅批准一次。
- `👎` 拒绝。
- 当请求提供持久批准选项时，使用 `/approve <id> allow-always`。

审批回应解析需要通过 `channels.signal.allowFrom`、`channels.signal.defaultTo` 或匹配的账户级字段明确指定 Signal 审批者。即使没有明确指定审批者，直接在同一聊天中发送的 Exec 审批提示仍可抑制重复的本地 `/approve` 回退；没有审批者的群组审批会继续显示本地回退。

## 问题回应

对于包含一个非机密单选问题和一至四个选项的 `ask_user` 提示，Signal 会在选项标签旁显示 `1️⃣` 至 `4️⃣`。使用匹配的数字回应已送达的提示，即可作答。OpenClaw 会验证回应目标是否为 Bot 发出的消息，然后通过 Gateway 网关将数字映射到规范选项。过期或重复的点击会被忽略。多问题、多选和自由文本提示仍只能通过文本回复；常规 Signal 私信/群组准入规则用于授权发送者。

## 送达目标（CLI/定时任务）

- 私信：`signal:+15551234567`（或纯 E.164）。
- UUID 私信：`uuid:<id>`（或不带前缀的 UUID）。
- 群组：`signal:group:<groupId>`。
- 用户名：`username:<name>`（如果你的 Signal 账户支持）。

## 别名

为经常使用的 Signal 目标配置稳定名称的别名。别名仅是 OpenClaw 侧的配置；它们不会创建或编辑 Signal 联系人。

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
        jane: "uuid:123e4567-e89b-12d3-a456-426614174000",
        ops: "group:<groupId>",
      },
      defaultTo: "signal:me",
    },
  },
}
```

可在任何接受 Signal 送达目标的位置使用别名：

```bash
openclaw message send --channel signal --target signal:ops --message "部署已完成"
```

各账户别名会继承顶层别名，并且可以添加或覆盖名称：

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
      },
      accounts: {
        work: {
          aliases: {
            ops: "group:<workGroupId>",
          },
        },
      },
    },
  },
}
```

`openclaw directory peers list --channel signal` 和 `openclaw directory groups list --channel signal` 会列出已配置的别名。Signal 目录由配置提供支持；它不会实时查询 Signal 联系人，也不会修改 Signal 账户。

## 故障排查

首先运行以下排查步骤：

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

然后根据需要确认私信配对状态：

```bash
openclaw pairing list signal
```

常见故障：

- 守护进程可访问但无回复：验证 `account`、`transport.kind`、传输 URL 和接收模式。
- 私信被忽略：发送者正在等待配对批准。
- 群组消息被忽略：群组发送者/提及门控阻止了送达。
- 编辑后出现配置验证错误：运行 `openclaw doctor --fix`。
- 诊断中缺少 Signal：确认 `channels.signal.enabled: true`。

其他检查：

```bash
openclaw pairing list signal
pgrep -af signal-cli
openclaw logs --plain --limit 500 | grep -i "signal" | tail -20
```

有关分类排查流程，请参阅[渠道故障排查](/zh-CN/channels/troubleshooting)。

## 安全说明

- `signal-cli` 在本地存储账户密钥（通常位于 `~/.local/share/signal-cli/data/`）。
- 迁移或重建服务器之前，请备份 Signal 账户状态。
- 除非明确需要更广泛的私信访问权限，否则请保留 `channels.signal.dmPolicy: "pairing"`。
- 仅注册或恢复流程需要 SMS 验证，但失去对号码/账户的控制可能会使重新注册变得复杂。

## 配置参考（Signal）

完整配置：[配置](/zh-CN/gateway/configuration)

提供商选项：

- `channels.signal.enabled`：启用/禁用渠道启动。
- `channels.signal.account`：Bot 账号的 E.164 号码。
- `channels.signal.accountUuid`：可选的 Bot 账号 UUID，用于原生 @提及检测和循环保护。
- `channels.signal.transport`：账号自有的传输方式。使用托管式原生默认设置时请省略。
- `channels.signal.transport.kind`：`managed-native | external-native | container`。
- `channels.signal.transport.url`：`external-native` 和 `container` 必需；当 `managed-native` 的连接端点与守护进程绑定地址不同时，该项为可选。
- `channels.signal.transport.cliPath`：指向 `signal-cli` 的托管式原生路径。
- `channels.signal.transport.configPath`：可选的托管式原生 `signal-cli --config` 目录。
- `channels.signal.transport.httpHost`、`channels.signal.transport.httpPort`：托管式原生守护进程绑定地址（默认值为 `127.0.0.1:8080`）。
- `channels.signal.transport.startupTimeoutMs`：托管式原生启动等待时间，以毫秒为单位（最小值 1000，上限 120000；默认值 30000）。
- `channels.signal.transport.receiveMode`：托管式原生 `on-start | manual`。
- `channels.signal.ignoreAttachments`：跳过此账号的入站附件下载。
- `channels.signal.transport.ignoreStories`：托管式原生动态开关。
- `channels.signal.sendReadReceipts`：转发已读回执。
- `channels.signal.dmPolicy`：`pairing | allowlist | open | disabled`（默认值：配对）。
- `channels.signal.allowFrom`：私信允许列表（E.164 或 `uuid:<id>`）。`open` 要求提供 `"*"`。Signal 没有用户名；请使用电话号码/UUID ID。
- `channels.signal.aliases`：OpenClaw 端的私信或群组投递目标别名。
- `channels.signal.groupPolicy`：`open | allowlist | disabled`（默认值：允许列表）。
- `channels.signal.groupAllowFrom`：群组允许列表；接受 Signal 群组 ID（原始格式、`group:<id>` 或 `signal:group:<id>`）、发送者的 E.164 号码或 `uuid:<id>` 值。
- `channels.signal.groups`：以 Signal 群组 ID（或 `"*"`）为键的每群组覆盖设置。支持的字段：`requireMention`、`tools`、`toolsBySender`。
- `channels.signal.accounts.<id>.groups`：用于多账号设置的 `channels.signal.groups` 每账号版本。
- `channels.signal.accounts.<id>.aliases`：每账号别名，与顶层别名合并。
- `channels.signal.replyToMode`：原生回复引用模式，`off | first | all | batched`（默认值：`all`）。
- `channels.signal.replyToModeByChatType.direct`、`channels.signal.replyToModeByChatType.group`：按聊天类型设置的原生回复引用覆盖项。
- `channels.signal.accounts.<id>.replyToMode`、`channels.signal.accounts.<id>.replyToModeByChatType.direct`、`channels.signal.accounts.<id>.replyToModeByChatType.group`：每账号回复引用覆盖项。
- `channels.signal.historyLimit`：作为上下文包含的最大群组消息数（0 表示禁用）。
- `channels.signal.dmHistoryLimit`：以用户轮次计的私信历史记录上限。每用户覆盖项：`channels.signal.dms["<phone_or_uuid>"].historyLimit`。
- `channels.signal.textChunkLimit`：出站分块大小，以字符数计（默认值 4000）。
- `channels.signal.streaming.chunkMode`：使用 `length`（默认值），或使用 `newline` 在按长度分块前先按空行（段落边界）拆分。
- `channels.signal.mediaMaxMb`：入站/出站媒体大小上限，以 MB 为单位（默认值 8）。
- `channels.signal.reactionLevel`：`off | ack | minimal | extensive`（默认值为 `minimal`）。请参阅[表情回应](#reactions-message-tool)。
- `channels.signal.reactionNotifications`：`off | own | all | allowlist`（默认值为 `own`）——何时向智能体通知其他人的传入表情回应。
- `channels.signal.reactionAllowlist`：当 `reactionNotifications: "allowlist"` 时，其表情回应会通知智能体的发送者。
- `channels.signal.streaming.block.enabled`、`channels.signal.streaming.block.coalesce`：各渠道共享的分块模式流式传输控制项。请参阅[流式传输](/zh-CN/concepts/streaming)。

相关全局选项：

- `agents.entries.*.groupChat.mentionPatterns`（纯文本回退；配置了 Bot 账号身份后，会从结构化元数据中检测 Signal 原生 @提及）。
- `messages.groupChat.mentionPatterns`（全局回退）。
- `channels.signal.responsePrefix` 或账号级 `responsePrefix`。

## 相关内容

- [渠道概览](/zh-CN/channels) - 所有支持的渠道
- [配对](/zh-CN/channels/pairing) - 私信身份验证和配对流程
- [群组](/zh-CN/channels/groups) - 群聊行为和提及门控
- [频道路由](/zh-CN/channels/channel-routing) - 消息的会话路由
- [安全性](/zh-CN/gateway/security) - 访问模型和安全强化
