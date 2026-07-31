---
read_when:
    - 实现或更改 Bonjour 设备发现/广播
    - 调整远程连接模式（直连与 SSH）
    - 远程节点的设备发现与配对设计
summary: 用于查找 Gateway 网关的节点设备发现和传输协议（Bonjour、Tailscale、SSH）
title: 设备发现和传输协议
x-i18n:
    generated_at: "2026-07-26T06:44:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3a3f1a6a1212ab0bc7021e77c88de059edcb8e09eff90d3e1e59451b9b20876b
    source_path: gateway/discovery.md
    workflow: 16
---

OpenClaw 有两个相关但不同的设备发现问题：

1. **操作员远程控制**：macOS 菜单栏应用控制在其他位置运行的 Gateway 网关。
2. **节点配对**：iOS/Android（及未来的节点）查找 Gateway 网关并进行安全配对。

所有网络设备发现/广播功能都位于 **Node Gateway 网关**
（`openclaw gateway`）中；客户端（Mac 应用、iOS）仅作为使用方。

## 术语

- **Gateway 网关**：一个长期运行的进程，负责管理状态（会话、
  配对、节点注册表）并运行渠道。大多数设置中每台主机使用一个；
  也可以设置相互隔离的多个 Gateway 网关。
- **Gateway 网关 WS（控制平面）**：默认位于 `127.0.0.1:18789`
  的 WebSocket 端点；通过 `gateway.bind` 将其绑定到 LAN/tailnet。
- **直接 WS 传输**：面向 LAN/tailnet 的 Gateway 网关 WS 端点（不使用 SSH）。
- **SSH 传输（回退方案）**：通过 SSH 转发
  `127.0.0.1:18789` 来实现远程控制。
- **旧版 TCP 桥接（已移除）**：较旧的节点传输方式（参见
  [Bridge protocol](/zh-CN/gateway/bridge-protocol)）；已不再通过设备发现进行广播，
  也不再包含于当前构建中。

协议详情：[Gateway 网关协议](/zh-CN/gateway/protocol)、
[Bridge protocol（旧版）](/zh-CN/gateway/bridge-protocol)。

## 为何直接连接与 SSH 并存

- **直接 WS** 可在同一网络和 tailnet 内提供最佳用户体验：通过 Bonjour
  在 LAN 中自动发现，由 Gateway 网关管理配对令牌和 ACL，
  并且无需 shell 访问权限。
- **SSH** 是通用的回退方案：只要拥有 SSH 访问权限即可在任何位置使用，
  即使网络互不关联也能工作，不受组播/mDNS 问题影响，并且除 SSH 外
  无需开放新的入站端口。

## 设备发现输入

### 1) Bonjour / DNS-SD

组播 Bonjour 仅尽力而为，且无法跨越网络。OpenClaw 还支持通过配置的广域 DNS-SD
域浏览同一 Gateway 网关信标，因此设备发现既可以覆盖同一 LAN 上的
`local.`，也可以覆盖用于跨网络发现的已配置单播 DNS-SD 域。

启用内置 `bonjour` 插件后，**Gateway 网关**会通过 Bonjour
广播其 WS 端点；客户端浏览并显示“选择 Gateway 网关”列表，
然后存储所选端点。

故障排查和信标详情：[Bonjour](/zh-CN/gateway/bonjour)。

#### 服务信标详情

- 服务类型：`_openclaw-gw._tcp`（Gateway 网关传输信标）。
- TXT 键（非机密）：

  | 键                          | 说明                                                                                                                                                             |
  | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `role=gateway`              | 始终存在。                                                                                                                                                       |
  | `transport=gateway`         | 始终存在。                                                                                                                                                       |
  | `displayName=<name>`        | 操作员配置的显示名称。                                                                                                                                           |
  | `lanHost=<hostname>.local`  | 仅用于 LAN mDNS 广播器；广域 DNS-SD 不会写入此项。                                                                                                               |
  | `gatewayPort=18789`         | Gateway 网关 WS + HTTP 端口。                                                                                                                                    |
  | `gatewayTls=1`              | 仅在启用 TLS 时存在。                                                                                                                                            |
  | `gatewayTlsSha256=<sha256>` | 仅在启用 TLS 且指纹可用时存在。                                                                                                                                  |
  | `tailnetDns=<magicdns>`     | 可选提示；Tailscale 可用时自动检测。                                                                                                                             |
  | `sshPort=<port>`            | 仅当 `discovery.mdns.mode="full"` 时存在；在默认 `"minimal"` 模式下省略（SSH 默认为 `22`），LAN 广播器和广域 DNS-SD 均如此。 |
  | `cliPath=<path>`            | 与 `sshPort` 使用相同的 `discovery.mdns.mode="full"` 条件；用于 CLI 路径的远程安装提示。                                                                          |

  插件设备发现契约中定义了一个 `canvasPort` TXT 键，用于未来的
  canvas 主机端口，但当前没有任何代码路径为其设置值，因此目前
  从不会发送该键。

安全说明：

- Bonjour/mDNS TXT 记录**未经身份验证**。客户端必须仅将 TXT
  值视为用户体验提示。
- 路由（主机/端口）应优先使用**解析后的服务端点**
  （SRV + A/AAAA），而不是 TXT 提供的 `lanHost`、`tailnetDns` 或 `gatewayPort`。
- TLS 固定绝不能允许广播的 `gatewayTlsSha256` 覆盖
  先前存储的固定值。
- 只要所选路由基于安全连接/TLS，iOS/Android 节点在首次存储固定值前
  应要求用户明确确认“信任此指纹”（带外验证）。

启用、禁用和覆盖：

- `openclaw plugins enable bonjour` 启用 LAN 组播广播。
- `openclaw.json` 中的 `discovery.mdns.mode` 控制 mDNS 广播：
  `"minimal"`（默认）、`"full"`（将 `cliPath`/`sshPort` 添加到 LAN
  信标和所有广域 DNS-SD 区域），或 `"off"`（禁用 mDNS）。
- `OPENCLAW_DISABLE_BONJOUR=1` 强制禁用广播；`discovery.mdns.mode="off"`
  可单独将其禁用。`OPENCLAW_DISABLE_BONJOUR=0` 是一个显式选择加入项，
  可覆盖插件在检测到容器（Docker、containerd、Kubernetes、LXC）时的自动禁用行为；
  但不会覆盖 `discovery.mdns.mode="off"`。内置 `bonjour` 插件在
  macOS 主机（`enabledByDefaultOnPlatforms: ["darwin"]`）上自动启动，并在检测到容器时
  自动禁用；Linux、Windows 和其他容器化部署需要显式设置
  `plugins enable bonjour`。
- `~/.openclaw/openclaw.json` 中的 `gateway.bind` 控制 Gateway 网关绑定模式。
- `OPENCLAW_SSH_PORT` 覆盖广播的 SSH 端口（仅当
  `discovery.mdns.mode="full"` 时生效）。
- `OPENCLAW_TAILNET_DNS` 发布 `tailnetDns` 提示（MagicDNS）。
- `OPENCLAW_CLI_PATH` 覆盖广播的 CLI 路径。

### 2) Tailnet（跨网络）

对于位于不同物理网络上的 Gateway 网关，Bonjour 无法提供帮助。
建议的直接连接目标是 Tailscale MagicDNS 名称（首选）或
稳定的 tailnet IP。

如果 Gateway 网关检测到自身运行于 Tailscale 下，它会发布
`tailnetDns` 作为客户端的可选提示（包括广域信标）。
macOS 应用在发现 Gateway 网关时优先使用 MagicDNS 名称，而不是原始 Tailscale IP；
由于 MagicDNS 会自动解析到当前 IP，因此即使 tailnet IP 发生变化
（节点重启、CGNAT 重新分配），连接仍保持可靠。

对于移动节点配对，设备发现提示绝不会放宽 tailnet/公共路由上的传输安全要求：

- iOS/Android 仍要求使用安全的首次 tailnet/公共连接路径
  （`wss://` 或 Tailscale Serve/Funnel）。
- 发现的原始 tailnet IP 只是路由提示，并不表示允许使用
  明文远程 `ws://`。
- 仍支持私有 LAN 直接连接 `ws://`。
- 如需在移动节点上使用最简单的 Tailscale 路径，请使用 Tailscale Serve，
  以便设备发现和设置均解析到同一个安全的 MagicDNS 端点。

### 3) 手动 / SSH 目标

当没有直接路由（或直接连接已禁用）时，客户端始终可以通过 SSH
转发 local loopback Gateway 网关端口来连接。参见
[远程访问](/zh-CN/gateway/remote)。

## 传输方式选择（客户端策略）

1. 如果已配置且可访问已配对的直接端点，则使用该端点。
2. 否则，如果设备发现在 `local.` 或配置的广域域中找到 Gateway 网关，
   则提供一键“使用此 Gateway 网关”选项，并将其保存为
   直接端点。
3. 否则，如果已配置 tailnet DNS/IP，则尝试直接连接。对于使用
   tailnet/公共路由的移动节点，直接连接意味着安全端点，而不是
   明文远程 `ws://`。
4. 否则，回退到 SSH。

## 配对和身份验证（直接传输）

Gateway 网关是节点/客户端准入的事实来源：

- 配对请求在 Gateway 网关中创建/批准/拒绝（参见
  [Gateway 网关配对](/zh-CN/gateway/pairing)）。
- Gateway 网关强制执行身份验证（令牌/密钥对）、权限范围/ACL
  （并非将所有方法直接代理出去）和速率限制。

## 各组件的职责

- **Gateway 网关**：广播设备发现信标、负责配对决策并托管
  WS 端点。
- **macOS 应用**：帮助你选择 Gateway 网关、显示配对提示，仅将 SSH
  用作回退方案。
- **iOS/Android 节点**：为方便起见浏览 Bonjour，连接到
  已配对的 Gateway 网关 WS。

## 相关内容

- [远程访问](/zh-CN/gateway/remote)
- [Tailscale](/zh-CN/gateway/tailscale)
- [Bonjour 设备发现](/zh-CN/gateway/bonjour)
