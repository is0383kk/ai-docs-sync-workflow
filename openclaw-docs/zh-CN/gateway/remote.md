---
read_when:
    - 运行远程 Gateway 网关设置或对其进行故障排查
summary: 使用 Gateway 网关 WebSocket、SSH 隧道和 tailnet 进行远程访问
title: 远程访问
x-i18n:
    generated_at: "2026-07-26T05:49:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8f05e32fcfa16d5ddfcd684d0550c9af311914e2b4d91c95edad3490dc2e56d9
    source_path: gateway/remote.md
    workflow: 16
---

OpenClaw 在一台主机上运行一个 Gateway 网关（主网关），并将所有客户端连接到该网关。Gateway 网关负责管理会话、身份验证配置文件、渠道和状态；除此之外的一切都是客户端。

- **操作员**（你或 macOS 应用）：如果 Gateway 网关可访问，直接使用 LAN/Tailnet WebSocket 最简单；SSH 隧道是通用的后备方案。
- **节点**（iOS/Android 和其他设备）：连接到 Gateway 网关的 **WebSocket**（LAN/tailnet 或 SSH 隧道）。

## 核心理念

Gateway 网关的 WebSocket 默认绑定到 **loopback**，端口为 `18789`（`gateway.port`）。如需远程使用，可通过 Tailscale Serve / 受信任的 LAN-Tailnet 绑定将其公开，或通过 SSH 转发 loopback 端口。

## 拓扑选项

| 设置                             | Gateway 网关运行位置                                                                                    | 最适合                                                                                                                                          |
| --------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| tailnet 中始终在线的 Gateway 网关 | 持久运行的主机（VPS 或家庭服务器），通过 Tailscale 或 SSH 访问                                        | 经常休眠但需要智能体始终在线的笔记本电脑。参见 [exe.dev](/zh-CN/install/exe-dev)（简单的虚拟机）或 [Hetzner](/zh-CN/install/hetzner)（生产环境 VPS）。 |
| 家庭台式机                      | 台式机；笔记本电脑通过 macOS 应用的远程模式连接（Settings → Connection → OpenClaw runs） | 将智能体运行在持续通电的硬件上。运行手册：[macOS 远程访问](/zh-CN/platforms/mac/remote)。                                       |
| 笔记本电脑                            | 笔记本电脑，通过 SSH 隧道或 Tailscale Serve 安全公开（保留 `gateway.bind: "loopback"`）                | 单机设置。参见 [Tailscale](/zh-CN/gateway/tailscale) 和 [Web](/zh-CN/web)。                                                                       |

对于始终在线和笔记本电脑设置，建议保留 `gateway.bind: "loopback"`，并对 Control UI 使用 **Tailscale Serve**，或使用带有 `gateway.remote.transport: "direct"` 的受信任 LAN/Tailnet 绑定。SSH 隧道是可从任何机器使用的后备方案。

## 命令流程（各组件在哪里运行）

一个 Gateway 网关负责管理状态和渠道；节点是外围设备。示例（将 Telegram 消息路由到节点工具）：

1. Telegram 消息到达 **Gateway 网关**。
2. Gateway 网关运行**智能体**，由智能体决定是否调用节点工具。
3. Gateway 网关通过 Gateway 网关 WebSocket（`node.invoke` RPC）调用**节点**。
4. 节点返回结果；Gateway 网关向 Telegram 回复。

节点不运行 Gateway 网关服务。除非有意运行相互隔离的配置文件，否则每台主机只应运行一个 Gateway 网关（参见[多个 Gateway 网关](/zh-CN/gateway/multiple-gateways)）。macOS 应用的“节点模式”只是通过 Gateway 网关 WebSocket 连接的节点客户端。

## SSH 隧道（CLI + 工具）

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

隧道建立后，`openclaw health` 和 `openclaw status --deep` 会通过 `ws://127.0.0.1:18789` 访问远程 Gateway 网关。`openclaw gateway status`、`openclaw gateway health`、`openclaw gateway probe` 和 `openclaw gateway call` 也可以通过 `--url` 指向转发后的 URL。

<Note>
将 `18789` 替换为已配置的 `gateway.port`（或 `--port` / `OPENCLAW_GATEWAY_PORT`）。
</Note>

<Warning>
`--url` 绝不会回退使用配置或环境中的凭据。请显式传入 `--token` 或 `--password`；如果未传入，客户端不会发送任何凭据，而当目标 Gateway 网关要求身份验证时，连接将失败。
</Warning>

## CLI 远程默认值

持久保存远程目标，使 CLI 命令默认使用该目标：

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://127.0.0.1:18789",
      token: "your-token",
    },
  },
}
```

当 Gateway 网关仅限 loopback 时，将 URL 保持为 `ws://127.0.0.1:18789`，并先打开 SSH 隧道。在 macOS 应用的 SSH 隧道传输方式中，将发现的 Gateway 网关主机名填入 `gateway.remote.sshTarget`（`user@host` 或 `user@host:port`）；`gateway.remote.url` 仍为本地隧道 URL。如果远程端口与本地端口不同，请设置 `gateway.remote.remotePort`。

默认严格验证主机密钥（`gateway.remote.sshHostKeyPolicy: "strict"`）。将其设为 `"openssh"` 可改为委托给实际生效的 OpenSSH 配置；启用前请检查用户级和系统级 SSH 设置。

对于已经可通过受信任 LAN 或 Tailnet 访问的 Gateway 网关，请使用直接模式：

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      transport: "direct",
      url: "ws://192.168.0.202:18789",
      token: "your-token",
    },
  },
}
```

## 凭据优先级

Gateway 网关凭据解析在调用、探测、状态路径以及 Discord Exec 审批监控中遵循同一份共享约定。节点主机也使用相同约定，但本地模式有一个例外（会忽略 `gateway.remote.*`）。

- 在接受显式身份验证的调用路径中，显式凭据（`--token`、`--password` 或工具的 `gatewayToken`）始终优先。
- URL 覆盖安全规则：
  - CLI `--url` 绝不会复用隐式配置/环境凭据。
  - 环境变量 `OPENCLAW_GATEWAY_URL` 只能使用环境凭据（`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`）。
- 本地模式默认值：
  - 令牌：`OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token` -> `gateway.remote.token`（仅当本地令牌未设置时才使用远程后备值）
  - 密码：`OPENCLAW_GATEWAY_PASSWORD` -> `gateway.auth.password` -> `gateway.remote.password`（仅当本地密码未设置时才使用远程后备值）
- 远程模式默认值：
  - 令牌：`gateway.remote.token` -> `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token`
  - 密码：`OPENCLAW_GATEWAY_PASSWORD` -> `gateway.remote.password` -> `gateway.auth.password`
- 节点主机的本地模式例外：忽略 `gateway.remote.token` / `gateway.remote.password`。
- 默认严格执行远程探测/状态令牌检查：以远程模式为目标时，仅使用 `gateway.remote.token`（不回退使用本地令牌）。
- Gateway 网关环境覆盖仅使用 `OPENCLAW_GATEWAY_*`。

## 聊天 UI 远程访问

WebChat 没有单独的 HTTP 端口；SwiftUI 聊天 UI 直接连接到 Gateway 网关 WebSocket。

- 通过 SSH 转发 `18789`（参见上文），然后将客户端连接到 `ws://127.0.0.1:18789`。
- 对于 LAN/Tailnet 直接模式，将客户端连接到已配置的私有 `ws://` 或安全 `wss://` URL。
- 在 macOS 上，应用的远程模式会自动管理所选传输方式。

## macOS 应用远程模式

macOS 菜单栏应用会端到端管理同一套设置：远程状态检查、WebChat 和 Voice Wake 转发。运行手册：[macOS 远程访问](/zh-CN/platforms/mac/remote)。

## 安全规则（远程/VPN）

除非确定需要绑定，否则请让 Gateway 网关保持**仅限 loopback**。

- **Loopback + SSH/Tailscale Serve** 是最安全的默认方案（不会公开暴露）。
- 明文 `ws://` 可用于 loopback、私有/LAN（RFC 1918）、链路本地、CGNAT、`.local` 和 `.ts.net` 主机。公共远程主机必须使用 `wss://`。
- **非 loopback 绑定**（`lan`/`tailnet`/`custom`，或 loopback 不可用时的 `auto`）必须使用 Gateway 网关身份验证：令牌、密码，或带有 `gateway.auth.mode: "trusted-proxy"` 的身份感知反向代理。
- `gateway.remote.token` / `.password` 是客户端凭据来源；它们本身不会配置服务器身份验证。
- 仅当 `gateway.auth.*` 未设置时，本地调用路径才能使用 `gateway.remote.*` 作为后备值。
- 如果通过 SecretRef 显式配置的 `gateway.auth.token` / `gateway.auth.password` 无法解析，解析将以关闭方式失败（不会使用远程后备值掩盖问题）。
- `gateway.remote.tlsFingerprint` 会固定 `wss://` 的远程 TLS 证书，包括操作员/控制流量以及 macOS 直接模式中的配套节点。如果没有存储的固定值，macOS 仅会在正常系统信任检查通过后于首次使用时固定证书；使用自签名证书或私有 CA 的 Gateway 网关需要显式指纹或通过 SSH 远程连接。
- 当 `gateway.auth.allowTailscale: true` 时，**Tailscale Serve** 可通过身份标头对 Control UI/WebSocket 流量进行身份验证。HTTP API 端点不使用该标头身份验证，而是遵循 Gateway 网关的常规 HTTP 身份验证模式。此无令牌流程假定 Gateway 网关主机可信；将其设为 `false` 可在所有位置使用共享密钥身份验证。
- 默认情况下，**受信任代理**身份验证要求使用非 loopback 的身份感知代理。同一主机上的 loopback 反向代理需要显式设置 `gateway.auth.trustedProxy.allowLoopback = true`。
- 将浏览器控制视同操作员访问：仅限 tailnet，并有意识地进行节点配对。

深入了解：[安全](/zh-CN/gateway/security)。

### macOS：通过 LaunchAgent 建立持久 SSH 隧道

对于 macOS 客户端，最简单的持久化设置是使用 SSH `LocalForward` 配置条目，并配合 LaunchAgent，使隧道在重启和崩溃后继续保持运行。

#### 第 1 步：添加 SSH 配置

编辑 `~/.ssh/config`：

```ssh
Host remote-gateway
    HostName <REMOTE_IP>
    User <REMOTE_USER>
    LocalForward 18789 127.0.0.1:18789
    IdentityFile ~/.ssh/id_rsa
```

将 `<REMOTE_IP>` 和 `<REMOTE_USER>` 替换为你的值。

#### 第 2 步：复制 SSH 密钥（一次性）

```bash
ssh-copy-id -i ~/.ssh/id_rsa <REMOTE_USER>@<REMOTE_IP>
```

#### 第 3 步：配置 Gateway 网关令牌

```bash
openclaw config set gateway.remote.token "<your-token>"
```

如果远程 Gateway 网关使用密码身份验证，请改用 `gateway.remote.password`。`OPENCLAW_GATEWAY_TOKEN` 作为 Shell 级覆盖仍然有效，但持久的远程客户端设置应使用 `gateway.remote.token` / `gateway.remote.password`。

#### 第 4 步：创建 LaunchAgent

保存为 `~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.ssh-tunnel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/ssh</string>
        <string>-N</string>
        <string>remote-gateway</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

#### 第 5 步：加载 LaunchAgent

```bash
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist
```

隧道会在登录时自动启动，在崩溃后重新启动，并保持转发端口可用。

<Note>
如果旧设置遗留了 `com.openclaw.ssh-tunnel` LaunchAgent，请将其卸载并删除。
</Note>

#### 故障排查

```bash
# 检查隧道是否正在运行
ps aux | grep "ssh -N remote-gateway" | grep -v grep
lsof -i :18789

# 重启隧道
launchctl kickstart -k gui/$UID/ai.openclaw.ssh-tunnel

# 停止隧道
launchctl bootout gui/$UID/ai.openclaw.ssh-tunnel
```

| 配置项                               | 作用                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| `LocalForward 18789 127.0.0.1:18789` | 将本地端口 18789 转发到远程端口 18789                        |
| `ssh -N`                             | 通过 SSH 连接但不执行远程命令（仅用于端口转发）              |
| `KeepAlive`                          | 隧道崩溃时自动重启                                           |
| `RunAtLoad`                          | 登录时 LaunchAgent 加载后启动隧道                            |

## 相关内容

- [Tailscale](/zh-CN/gateway/tailscale)
- [身份验证](/zh-CN/gateway/authentication)
- [远程 Gateway 网关设置](/zh-CN/gateway/remote-gateway-readme)
