---
read_when:
    - 配对或重新连接 iOS 节点
    - 启用直接连接的 Apple Watch 节点或进行故障排查
    - 从源代码运行 iOS 应用
    - 调试 Gateway 网关发现或 Canvas 命令
summary: iOS 节点应用：连接到 Gateway 网关、配对、画布和故障排查
title: iOS 应用
x-i18n:
    generated_at: "2026-07-26T06:14:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2b01a63fa1e2c445f7fb35843536f7f5918e94bfe885dac19c852d7d52d86342
    source_path: platforms/ios.md
    workflow: 16
---

可用性：为某个版本启用后，iPhone 应用构建会通过 Apple 渠道分发。本地开发构建也可从源代码运行。

## 功能

- 通过 WebSocket（LAN 或 tailnet）连接到 Gateway 网关。
- 提供节点能力：Canvas、屏幕快照、相机拍摄、位置、Talk 模式、语音唤醒，以及选择启用的健康摘要。
- 接收 `node.invoke` 命令并报告节点状态事件。
- 可从智能体界面（文件）以只读方式浏览所选智能体的工作区：逐层查看目录、带语法高亮的文本预览、图像预览以及通过共享菜单导出。不支持写入操作；预览大小受 Gateway 网关限制。
- 为每个已配对的 Gateway 网关保留一份较小的只读离线缓存，存储最近的聊天会话和记录：冷启动时会立即显示最后已知的记录，并在 Gateway 网关响应后刷新；断开连接期间仍可浏览最近的聊天；重置或遗忘操作会清除受保护的本地缓存。
- 断开连接期间发送的文本消息会进入每个 Gateway 网关各自的持久发件箱队列（最多 50 条）：已排队的消息气泡会显示在记录中；重新连接后按顺序发送，并进行幂等重试；在规范历史记录确认已发送前始终持久保留；先采用退避策略重试，之后才显示重试/删除操作；离线超过 48 小时后消息会过期而不会发送；重置或遗忘操作会同时清除队列和缓存。
- 聊天是统一的文本和语音界面。聊天操作可以在不离开聊天的情况下打开完整的会话屏幕，还可以显示或隐藏助手推理和工具活动。轻点麦克风可进行草稿听写，打开其菜单可录制语音留言，也可使用内嵌的 Talk 控件进行实时语音交流；聆听或说话时，Talk 控件会根据实时麦克风或播放音量呈现动画。
- 当操作员连接具有 `operator.admin` 且 Gateway 网关支持 `openclaw.chat` 时，**设置 -> OpenClaw** 会打开专用的 Gateway 网关设置助手。其设置对话与普通聊天相互独立，会在本地隐去包含秘密信息的回复，并且只有在你轻点 **打开聊天** 后才会转到聊天。
- 可按需朗读助手消息：在聊天中长按一条消息，然后选择 **聆听**。应用会使用已配置的 TTS 提供商播放 Gateway 网关支持的 `tts.speak` 音频片段；如果 Gateway 网关音频不可用或无法播放，则回退到设备端语音。切换会话或应用进入后台时，播放会停止。

## 要求

- Gateway 网关运行在另一台设备上（macOS、Linux，或通过 WSL2 运行的 Windows）。
- 网络路径：
  - 通过 Bonjour 连接到同一 LAN，**或**
  - 通过单播 DNS-SD 连接到 tailnet（示例域名：`openclaw.internal.`），**或**
  - 手动指定主机/端口（回退方案）。

## 快速开始（配对并连接）

首次启动时，应用会展示简短的配对说明和一个权限页面
（通知、相机、麦克风、照片、通讯录、日历、提醒事项、
位置）。所有授权均为可选，并可稍后在 **设置** -> **权限**
或 iOS 的 Settings 应用中更改。

1. 启动一个经过身份验证且手机可访问其路由的 Gateway 网关。推荐使用 Tailscale
   Serve 作为远程访问路径：

```bash
openclaw gateway --port 18789 --tailscale serve
```

对于可信的同一 LAN 设置，请改用经过身份验证的 `gateway.bind: "lan"`。
默认的环回绑定无法从手机访问。如果尚未配置
Gateway 网关，请先运行 `openclaw onboard`，以便创建设置代码时
存在令牌或密码身份验证路径。

2. 打开 [Control UI](/zh-CN/web/control-ui)，选择 **节点**，然后在
   **设备** 页面点击 **配对移动设备**。建议使用完全访问权限，
   并且默认已选择该选项；仅当你希望排除
   Gateway 网关管理控制时才选择受限访问，然后点击 **创建设置代码**。

3. 在 iOS 应用中，打开 **设置** -> **Gateway 网关**，扫描二维码（或粘贴
   设置代码），然后连接。

   如果设置代码同时包含 LAN 和 Tailscale Serve 路由，应用会
   按顺序探测这些路由，并保存第一个可访问的端点。

   已配对的 Gateway 网关会保留在 **Gateway 网关** 列表中。对勾表示
   当前聚焦的 Gateway 网关；使用另一行的闪电控件可使其
   操作员会话同时保持连接。切换焦点不会
   断开其他已启用的 Gateway 网关。只有当前聚焦的 Gateway 网关会接收
   iPhone 上承载能力的节点会话，因此相机、屏幕、位置及
   其他设备命令始终只有一个明确的所有者。应用进入后台后，
   iOS 可能会暂停这些前台连接。

4. 官方应用会自动连接。如果显示 **待审批** 请求，
   请先检查其角色和权限范围，再予以批准。

   **设置 → Gateway 网关** 会显示已保存的操作员连接具有
   **完全** 还是 **受限** 访问权限。为确保不记名令牌安全，明文 LAN `ws://` 设置会自动
   限制访问权限。如果访问受限，请配置 `wss://` 或
   Tailscale Serve，从 Control UI 或 `openclaw qr` 扫描新的完全访问代码，
   然后重新连接以启用设置和升级功能。

Control UI 按钮要求已有一个具有 `operator.admin` 的配对会话。
作为终端回退方案，请在 iOS 应用中选择一个已发现的 Gateway 网关（或启用
手动主机并输入主机/端口），然后在 Gateway 网关主机上批准请求：

```bash
openclaw devices list
openclaw devices approve <requestId>
```

如果应用使用已更改的身份验证详细信息（角色/权限范围/公钥）重试配对，之前的待处理请求将被取代，并创建新的 `requestId`。批准前请再次运行 `openclaw devices list`。

可选：如果 iOS 节点始终从严格受控的子网连接，你可以使用明确的 CIDR 或精确 IP 地址，选择启用首次节点自动批准：

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

此功能默认禁用。它仅适用于未请求任何权限范围的全新 `role: node` 配对。操作员/浏览器配对以及任何角色、权限范围、元数据或公钥变更仍需手动批准。

5. 验证连接：

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

## HealthKit 摘要

iOS 节点可以返回选择启用的只读 HealthKit 聚合数据，范围为当前
日历日。iOS 设备同意和明确的 Gateway 网关命令授权是
彼此独立的门槛。有关设置、调用、负载字段、隐私行为和故障排查，
请参阅 [HealthKit 摘要](/zh-CN/platforms/ios-healthkit)。

默认情况下，Apple Watch 配套应用会继续使用现有的 iPhone 中继，
无需单独与 Gateway 网关配对。在 Apple 的 Watch 应用中将 Watch 与 iPhone 配对，
从 **Watch app -> My Watch -> Available
Apps** 安装 OpenClaw，然后分别在两台设备上打开一次 OpenClaw。

## 审查命令审批

具有 `operator.admin` 的操作员连接，或由 Gateway 网关明确指定的
已配对 `operator.approvals` 连接，可以在 iPhone 上审查
待处理的 Exec 请求。审批卡片会显示 Gateway 网关提供的
已净化命令预览、警告、主机上下文、到期时间，以及该请求提供的
所有可用决策。已配对的 Apple Watch 会通过现有 iPhone 中继接收同一份
对审查者安全的提示，并提供精简的仅允许一次/拒绝决策子集。直接 Watch
Gateway 网关模式不会传递审批提示。

审批状态与 Control UI 及受支持的聊天界面共享。第一个提交的答复生效。
当其他界面解决请求后、收到远程已解决通知后，以及每当解决确认可能
丢失时，iPhone 和 Watch 都会获取 Gateway 网关的规范终态记录。在该
回读确认请求是否仍处于待处理状态之前，操作始终不可用。

审批归属绑定到所选的 Gateway 网关。切换 Gateway 网关时，无法
将旧提示应用于替代连接。早于统一审批方法的 Gateway 网关会
回退到已发布的 Exec 专用方法；如需保留终态状态及获得更丰富的跨界面结果，
则必须更新 Gateway 网关。

## 回答智能体问题

对于具有 `operator.questions`（或 `operator.admin`）的操作员连接，
聊天会将待处理的 Gateway 网关问题显示为原生卡片。卡片支持单选和
多选选项、选项描述、自由文本 **其他** 答案以及到期倒计时。
重新连接后会从 Gateway 网关重新加载待处理的问题。当此设备回答问题、
其他界面先行回答问题，或问题到期或被取消时，卡片会锁定。

## 可选的直接 Apple Watch 节点

直接模式会为手表提供独立的已签名节点身份和 Gateway 网关连接。
当 OpenClaw 处于活动状态时，即使已配对的 iPhone 不可用，
受支持的节点命令仍可通过手表的 Wi-Fi 或蜂窝网络运行。

要求：

- iPhone 使用 `operator.admin` 权限范围连接到 Gateway 网关。
- 设置代码提供一个使用 watchOS 信任证书的 `wss://` Gateway 网关端点；
  手表会轮询对应的 `https://` 来源。不支持纯 HTTP，
  也不支持自签名或仅凭指纹建立信任。有关端点配置，请参阅 [Gateway 网关负责的
  配对](/zh-CN/gateway/pairing)。手表无法独立访问环回、仅限 iPhone
  以及仅限 tailnet 的路由。
- 使用蜂窝网络需要支持蜂窝网络且已开通服务的 Apple Watch。
- OpenClaw 在手表上处于活动状态。Apple 不允许普通 watchOS 应用
  保持通用 WebSocket/TCP 连接，因此直接节点使用短时 HTTPS
  轮询，并在应用返回前台时重新连接。请参阅 Apple 的
  [watchOS 低层网络指南](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchOS)。

设置：

1. 在 iPhone 上打开 **设置 -> Apple Watch**。
2. 轻点 **启用直接 Gateway 网关连接**。
3. 在短期有效的设置代码过期前，在手表上打开 OpenClaw。
4. 使用 `openclaw nodes status` 验证单独的 Apple Watch 行。

设置代码包含一个短期有效且仅供节点使用的引导凭据；在其过期前，
请像对待密码一样保护它。它绝不会包含 iPhone 已保存的 Gateway 网关
密码或令牌。配对后，手表会存储自己的设备令牌并
删除引导凭据。直接模式仅涵盖以下命令。
聊天、Talk、审批及现有的 `watch.*` 通知流程仍是
iPhone 中继功能，并且仍需要已配对的 iPhone。

直接 watchOS 节点命令：

| 界面          | 命令                           | 说明                                                    |
| ------------- | ------------------------------ | ------------------------------------------------------- |
| 设备          | `device.info`、`device.status` | Watch 身份、电池、温度、存储和网络。                    |
| 通知          | `system.notify`                | 应用处于活动状态时可用；需要手表权限。                  |

watchOS 不向第三方应用提供 WebKit，因此直接 Watch 节点
不会公布 Canvas 命令。

## 官方构建使用基于中继的推送

官方分发的 iOS 构建使用外部推送中继，而不会将原始 APNs 令牌发布给 Gateway 网关。来自公开发布通道的官方 App Store 构建使用托管中继 `https://ios-push-relay.openclaw.ai`；此基础 URL 已硬编码用于 App Store 分发，不会读取任何覆盖值。

自定义中继部署需要使用明确独立的 iOS 构建/部署路径，其中继 URL 必须与 Gateway 网关的中继 URL 匹配。App Store 发布通道绝不接受自定义中继 URL。如果你使用自定义中继构建，请设置匹配的 Gateway 网关中继 URL：

```json5
{
  gateway: {
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
        },
      },
    },
  },
}
```

工作流程如下：

- iOS 应用使用 App Attest 和 StoreKit 应用交易 JWS 向中继注册。
- 中继返回一个不透明的中继句柄和一个注册范围内的发送授权。
- iOS 应用获取已配对的 Gateway 网关身份（`gateway.identity.get`），并在中继注册时包含该身份，从而将中继支持的注册委托给该特定 Gateway 网关。
- 应用通过 `push.apns.register` 将该中继支持的注册转发到已配对的 Gateway 网关。
- Gateway 网关将存储的中继句柄用于 `push.test`、后台唤醒和唤醒提示。
- 如果应用之后连接到不同的 Gateway 网关，或连接到使用不同中继基础 URL 的构建版本，它会刷新中继注册，而不是复用旧绑定。

此路径中 Gateway 网关**不**需要：无需部署范围的中继令牌，也无需用于官方 App Store 中继发送的直接 APNs 密钥。

预期的操作员流程：

1. 安装官方 iOS 应用。
2. 可选：仅在有意使用单独的自定义中继构建版本时，才在 Gateway 网关上设置 `gateway.push.apns.relay.baseUrl`。
3. 将应用与 Gateway 网关配对，并等待其完成连接。
4. 当应用获得 APNs 令牌、操作员会话已连接且中继注册成功后，应用会发布 `push.apns.register`。
5. 此后，`push.test`、重新连接唤醒和唤醒提示都可以使用存储的中继支持注册。

## 后台存活信标

当 iOS 通过静默推送、后台刷新或重大位置变化事件唤醒应用时，应用会尝试短暂地重新连接节点，然后使用 `event: "node.presence.alive"` 调用 `node.event`。仅在获知经过身份验证的节点设备身份后，Gateway 网关才会将其记录为已配对节点/设备元数据中的 `lastSeenAtMs`/`lastSeenReason`。

仅当 Gateway 网关响应包含 `handled: true` 时，应用才会将后台唤醒视为已成功记录。旧版 Gateway 网关可能会使用 `{ "ok": true }` 确认 `node.event`；此响应兼容，但不算作持久的最后活动时间更新。

兼容性说明：

- `OPENCLAW_APNS_RELAY_BASE_URL` 仍可作为 Gateway 网关的临时环境变量覆盖项（`gateway.push.apns.relay.baseUrl` 是配置优先路径）。
- App Store 发布构建版本的推送模式硬编码了托管中继主机，并且绝不会读取中继 URL 覆盖项——构建时环境变量 `OPENCLAW_PUSH_RELAY_BASE_URL` 仅影响本地/沙箱 iOS 构建模式。

## 身份验证和信任流程

中继用于强制执行官方 iOS 构建版本在 Gateway 网关上直接使用 APNs 时无法提供的两个约束：

- 只有通过 Apple 分发的正版 OpenClaw iOS 构建版本才能使用托管中继。
- Gateway 网关只能为已与该特定 Gateway 网关配对的 iOS 设备发送中继支持的推送。

逐跳流程：

1. `iOS app -> gateway`：应用通过常规 Gateway 网关身份验证流程与 Gateway 网关配对，从而获得经过身份验证的节点会话和经过身份验证的操作员会话。操作员会话调用 `gateway.identity.get`。
2. `iOS app -> relay`：应用通过 HTTPS 调用中继注册端点，并提供 App Attest 证明和 StoreKit 应用交易 JWS。中继会验证 Bundle ID、App Attest 证明和 Apple 分发证明，并要求使用官方/生产分发路径——这可以阻止本地 Xcode/开发构建版本使用托管中继，因为本地构建版本无法满足官方 Apple 分发证明要求。
3. `gateway identity delegation`：在中继注册之前，应用从 `gateway.identity.get` 获取已配对的 Gateway 网关身份，并将其包含在中继注册载荷中。中继返回一个中继句柄和一个委托给该 Gateway 网关身份的注册范围内发送授权。
4. `gateway -> relay`：Gateway 网关存储来自 `push.apns.register` 的中继句柄和发送授权。在 `push.test`、重新连接唤醒和唤醒提示期间，Gateway 网关使用自己的设备身份对发送请求进行签名；中继会根据注册时委托的 Gateway 网关身份，验证存储的发送授权和 Gateway 网关签名。即使另一个 Gateway 网关通过某种方式获得该句柄，也无法复用该存储的注册。
5. `relay -> APNs`：中继拥有生产 APNs 凭据以及官方构建版本的原始 APNs 令牌。对于中继支持的官方构建版本，Gateway 网关绝不会存储原始 APNs 令牌；中继代表已配对的 Gateway 网关向 APNs 发送最终推送。

创建此设计的原因是：将生产 APNs 凭据排除在用户 Gateway 网关之外，避免在 Gateway 网关上存储官方构建版本的原始 APNs 令牌，仅允许官方 OpenClaw iOS 构建版本使用托管中继，并防止一个 Gateway 网关向属于另一个 Gateway 网关的 iOS 设备发送唤醒推送。

本地/手动构建版本仍直接使用 APNs。如果在没有中继的情况下测试这些构建版本，Gateway 网关仍需要直接 APNs 凭据：

```bash
export OPENCLAW_APNS_TEAM_ID="TEAMID"
export OPENCLAW_APNS_KEY_ID="KEYID"
export OPENCLAW_APNS_PRIVATE_KEY_P8="$(cat /path/to/AuthKey_KEYID.p8)"
```

这些是 Gateway 网关主机运行时环境变量，而不是 Fastlane 设置。`apps/ios/fastlane/.env` 仅存储 App Store Connect 身份验证信息，例如 `APP_STORE_CONNECT_KEY_ID` 和 `APP_STORE_CONNECT_ISSUER_ID`；它不会为本地 iOS 构建版本配置直接 APNs 投递。

建议采用以下 Gateway 网关主机存储方式，与 `~/.openclaw/credentials/` 下的其他提供商凭据保持一致：

```bash
mkdir -p ~/.openclaw/credentials/apns
chmod 700 ~/.openclaw/credentials/apns
mv /path/to/AuthKey_KEYID.p8 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
chmod 600 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
export OPENCLAW_APNS_PRIVATE_KEY_PATH="$HOME/.openclaw/credentials/apns/AuthKey_KEYID.p8"
```

不要提交 `.p8` 文件，也不要将其放在仓库检出目录下。

## 设备发现路径

### Bonjour（局域网）

iOS 应用在 `local.` 上浏览 `_openclaw-gw._tcp`，并在配置后浏览相同的广域 DNS-SD 设备发现域。同一局域网内的 Gateway 网关会自动通过 `local.` 显示；跨网络设备发现可以使用已配置的广域域，而无需更改信标类型。

### Tailnet（跨网络）

如果 mDNS 被阻止，请使用单播 DNS-SD 区域（选择一个域；例如：`openclaw.internal.`）和 Tailscale 拆分 DNS。有关 CoreDNS 示例，请参阅 [Bonjour](/zh-CN/gateway/bonjour)。

### 手动主机/端口

在 Settings 中，启用 **Manual Host** 并输入 Gateway 网关主机和端口（默认值为 `18789`）。

## 多个 Gateway 网关

应用会保留所有已配对 Gateway 网关的注册表，因此可以在它们之间切换，而无需重新配对：

- **Settings -> Gateway** 会显示 **Paired Gateways** 列表，并标记当前活动的 Gateway 网关。点按某个条目即可切换；应用会断开当前会话并重新连接到所选 Gateway 网关。当配对了多个 Gateway 网关时，连接行旁会显示快速切换菜单。
- 凭据、TLS 信任决策、各 Gateway 网关的偏好设置和缓存的聊天历史记录会按 Gateway 网关分别存储。切换绝不会混用不同 Gateway 网关之间的状态，推送注册也会跟随活动 Gateway 网关。
- 轻扫已配对的 Gateway 网关（或使用其上下文菜单）以选择 **Forget**，这会移除其凭据、设备令牌、TLS 固定信息和缓存的聊天记录。
- 必须能在网络上发现 Gateway 网关，才能切换到它；手动添加的 Gateway 网关会使用保存的主机和端口重新连接。

## Canvas + A2UI

iOS 节点会呈现 WKWebView 画布。使用 `node.invoke` 驱动它：

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.navigate --params '{"url":"http://<gateway-host>:18789/__openclaw__/canvas/"}'
```

说明：

- Gateway 网关画布主机通过 Gateway 网关 HTTP 服务器（与 `gateway.port` 使用相同端口，默认值为 `18789`）提供 `/__openclaw__/canvas/` 和 `/__openclaw__/a2ui/`。
- iOS 节点将内置框架保留为连接后的默认视图。`canvas.a2ui.push` 和 `canvas.a2ui.reset` 使用应用自带的内置 A2UI 页面。
- 远程 Gateway 网关 A2UI 页面在 iOS 上仅供呈现；仅接受来自应用自带内置页面的原生 A2UI 按钮操作。
- 使用 `canvas.navigate` 和 `{"url":""}` 返回内置框架。

## 与计算机使用的关系

iOS 应用是移动节点界面，并非 Codex Computer Use 后端。Codex Computer Use 和 `cua-driver mcp` 通过 MCP 工具控制本地 macOS 桌面；iOS 应用通过 OpenClaw 节点命令公开 iPhone 功能，例如 `canvas.*`、`camera.*`、`screen.*`、`location.*` 和 `talk.*`。

智能体仍可通过调用节点命令，经由 OpenClaw 操作 iOS 应用，但这些调用会经过 Gateway 网关节点协议，并受 iOS 前台/后台限制。使用 [Codex Computer Use](/zh-CN/plugins/codex-computer-use) 控制本地桌面，使用本页面了解 iOS 节点功能。

### Canvas 求值/快照

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.eval --params '{"javaScript":"(() => { const {ctx} = window.__openclaw; ctx.clearRect(0,0,innerWidth,innerHeight); ctx.lineWidth=6; ctx.strokeStyle=\"#ff2d55\"; ctx.beginPath(); ctx.moveTo(40,40); ctx.lineTo(innerWidth-40, innerHeight-40); ctx.stroke(); return \"ok\"; })()"}'
```

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.snapshot --params '{"maxWidth":900,"format":"jpeg"}'
```

## 语音唤醒 + Talk 模式

- 语音唤醒和 Talk 模式可在 Settings 中使用。
- 当 `talk.realtime.transport` 为 `webrtc` 时，OpenAI 实时 Talk 使用客户端自有的 WebRTC；显式的 `gateway-relay` 配置仍由 Gateway 网关所有。请参阅 [Talk 模式](/zh-CN/nodes/talk)。
- 支持 Talk 的 iOS 节点会公布 `talk` 能力，并可声明 `talk.ptt.start`、`talk.ptt.stop`、`talk.ptt.cancel` 和 `talk.ptt.once`；对于可信且支持 Talk 的节点，Gateway 网关默认允许这些按住说话命令。
- iOS 可能会暂停后台音频；应用未处于活动状态时，应将语音功能视为尽力而为。

## 常见错误

- `NODE_BACKGROUND_UNAVAILABLE`：将 iOS 应用切换到前台（画布/相机/屏幕命令要求应用位于前台）。
- `A2UI_HOST_UNAVAILABLE`：应用 WebView 无法访问内置 A2UI 页面；让应用保持在前台的 Screen 标签页，然后重试。
- 配对提示始终不出现：运行 `openclaw devices list` 并手动批准。
- Watch 未显示 iPhone 状态：确认 iPhone 在 `watch.status` 中报告 `watchPaired: true`
  和 `watchAppInstalled: true`。如果配对状态为 false，请在 Apple 的 Watch 应用中配对
  Watch。如果安装状态为 false，请从 **My Watch -> Available Apps** 安装配套应用。
  完成任一更改后，在 Watch 上打开一次 OpenClaw；要实现即时可达，两个应用仍必须同时运行，
  而排队的更新可以稍后在后台到达。
- 重新安装后无法重新连接：钥匙串中的配对令牌已清除；请重新配对节点。

## 相关文档

- [配对](/zh-CN/channels/pairing)
- [设备发现](/zh-CN/gateway/discovery)
- [Bonjour](/zh-CN/gateway/bonjour)
