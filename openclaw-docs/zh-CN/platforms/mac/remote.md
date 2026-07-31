---
read_when:
    - 设置或调试远程 Mac 控制
summary: 用于控制远程 OpenClaw Gateway 网关的 macOS 应用流程
title: 远程控制
x-i18n:
    generated_at: "2026-07-26T06:53:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7e558c39fa173a77bf11270a8961c14c6e2350dfc4f458da3633532513b98bf6
    source_path: platforms/mac/remote.md
    workflow: 16
---

此流程可让 macOS 应用作为完整的远程控制端，控制运行在另一台主机（台式机/服务器）上的 OpenClaw Gateway 网关。应用可直接连接受信任的 LAN/Tailnet Gateway 网关 URL；如果远程 Gateway 网关仅限回环访问，应用也可以管理 SSH 隧道。健康检查、语音唤醒转发和 Web Chat 会复用 _Settings -> General_ 中的同一远程配置。

## 模式

- **本地（此 Mac）**：所有内容都在笔记本电脑上运行；不涉及 SSH。
- **通过 SSH 远程连接（默认）**：OpenClaw 命令在远程主机上运行。应用使用 `-o BatchMode`、你选择的身份/密钥以及本地端口转发来建立 SSH 连接。
- **远程直连（ws/wss）**：不使用 SSH 隧道；应用直接连接 Gateway 网关 URL（LAN、Tailscale、Tailscale Serve 或公共 HTTPS 反向代理）。

## 远程传输方式

- **SSH 隧道**（默认）：使用 `ssh -N -L ...` 将 Gateway 网关端口转发到 localhost。由于隧道使用回环连接，Gateway 网关看到的节点 IP 为 `127.0.0.1`。
- **直连（ws/wss）**：直接连接 Gateway 网关 URL。Gateway 网关可以看到真实的客户端 IP。

应用会为自身的 SSH 进程禁用 SSH 连接复用和身份验证后的后台运行，以便监控并重启确切的进程，即使所选别名启用了 `ControlMaster` 或 `ForkAfterAuthentication`。

默认严格验证 SSH 主机密钥，因为 Gateway 网关凭据会通过此隧道传输。若要选择采用托管 SSH 别名自身的信任行为，请通过 `openclaw-mac configure-remote` 设置 `--ssh-host-key-policy openssh`，或直接将 `gateway.remote.sshHostKeyPolicy` 设置为 `"openssh"`。选择采用此行为之前，请检查该别名以及任何匹配的 `Host *` 或系统配置。在应用中或通过 `configure-remote` 更改 SSH 目标时，策略会重置为 `strict`，除非你再次明确选择对新目标采用该行为。

在 SSH 隧道模式下，发现的 LAN/Tailnet 主机名会保存为 `gateway.remote.sshTarget`。应用会将 `gateway.remote.url` 保持为本地隧道端点（例如 `ws://127.0.0.1:18789`），使 CLI、Web Chat 和本地节点主机服务都使用同一个回环传输连接。当设备发现同时返回原始 Tailnet IP 和稳定主机名时，应用会优先使用 Tailscale MagicDNS 或 LAN 名称，以便在地址变化后更可靠地维持连接。如果本地隧道端口与远程 Gateway 网关端口不同，请将 `gateway.remote.remotePort` 设置为远程主机上的端口。

远程模式下的浏览器自动化由 CLI 节点主机负责，而不是由原生 macOS 应用节点负责。应用会尽可能启动已安装的节点主机服务；若要从该 Mac 启用浏览器控制，请使用 `openclaw node install ...` 和 `openclaw node start` 安装并启动该服务（或在前台运行 `openclaw node run ...`），然后将目标设为支持浏览器的节点。

## 远程主机上的前提条件

1. 安装 Node + pnpm，并构建/安装 OpenClaw CLI（`pnpm install && pnpm build && pnpm link --global`）。
2. 确保非交互式 shell 的 PATH 中包含 `openclaw`（如有需要，可将其符号链接到 `/usr/local/bin` 或 `/opt/homebrew/bin`）。
3. 对于 SSH 传输：设置基于密钥的 SSH 身份验证。建议使用 Tailscale IP，以便在 LAN 外也能稳定访问。

## macOS 应用设置

若要跳过欢迎流程并通过 SSH 预配置应用：

```bash
openclaw-mac configure-remote \
  --ssh-target user@gateway-host \
  --local-port 18789 \
  --remote-port 18789 \
  --token "$OPENCLAW_GATEWAY_TOKEN"
```

或者，如果受信任的 LAN 或 Tailnet 已经可以访问 Gateway 网关，则完全跳过 SSH：

```bash
openclaw-mac configure-remote \
  --direct-url ws://192.168.0.202:18789 \
  --token "$OPENCLAW_GATEWAY_TOKEN"
```

`openclaw-mac connect`、`wizard` 和 `configure-remote` 按以下顺序解析活动配置：`OPENCLAW_CONFIG_PATH`，然后是 `$OPENCLAW_STATE_DIR/openclaw.json`，最后是 `~/.openclaw/openclaw.json`。两种配置形式都会写入该活动文件、将新手引导标记为已完成，并让应用在下次启动时管理所选传输方式。`--local-port`/`--remote-port` 默认为 `18789`。其他标志包括：`--password`、`--identity <path>`、`--ssh-host-key-policy <strict|openssh>`、`--project-root <path>`、`--cli-path <path>`、`--json`。运行 `openclaw-mac configure-remote --help` 查看完整参考。

若要改为从 UI 配置：

1. 打开 _Settings -> General_。
2. 在 **OpenClaw runs** 下，选择 **Remote** 并设置：
   - **Transport**：选择 **SSH tunnel** 或 **Direct (ws/wss)**。
   - **SSH target**：`user@host`（可选 `:port`）。如果 Gateway 网关位于同一 LAN 中并通过 Bonjour 广播，请从发现列表中选择它，以自动填充此字段。
   - **Gateway URL**（仅限 Direct）：`wss://gateway.example.ts.net`（本地/LAN 使用 `ws://...`）。
   - **Identity file**（高级）：你的密钥路径。
   - **Project root**（高级）：用于运行命令的远程检出路径。
   - **CLI path**（高级）：可选的可运行 `openclaw` 入口点/二进制文件路径（广播时自动填充）。
3. 点击 **Test remote**。成功表示远程 `openclaw status --json` 已正确运行。失败通常表示 PATH/CLI 存在问题；退出码 127 表示在远程主机上找不到 CLI。
4. 健康检查和 Web Chat 现在会自动通过所选传输方式运行。

## Web Chat

- **SSH 隧道**：通过转发后的 WebSocket 控制端口（默认 18789）连接 Gateway 网关。
- **直连（ws/wss）**：直接连接已配置的 Gateway 网关 URL。
- 没有单独的 Web Chat HTTP 服务器。

## 权限

- 远程主机需要与本地主机相同的 TCC 批准（Automation、Accessibility、Screen Recording、Microphone、Speech Recognition、Notifications）。在该计算机上运行一次新手引导以授予这些权限。
- 节点通过 `node.list` / `node.describe` 广播其权限状态，以便智能体了解哪些功能可用。

## 安全说明

- 远程主机应优先绑定到回环接口，并通过 SSH、Tailscale Serve 或受信任的 Tailnet/LAN 直连 URL 进行连接。
- 默认情况下，SSH 隧道要求使用已受信任的主机密钥。请先信任主机密钥（将其添加到配置的 known-hosts 文件），或针对其 OpenSSH 信任策略已被你接受的托管别名，明确设置 `gateway.remote.sshHostKeyPolicy: "openssh"`。
- 如果将 Gateway 网关绑定到非回环接口，则必须要求有效的 Gateway 网关身份验证：令牌、密码或带有 `gateway.auth.mode: "trusted-proxy"` 的身份感知型反向代理。
- 直连 `wss://` 会对操作员/控制流量和 Mac 配套节点应用同一证书策略。设置 `gateway.remote.tlsFingerprint` 以明确固定证书。如果未设置，应用只有在常规 macOS 信任验证成功后，才会记录首次使用时的固定证书。
- 请参阅[安全](/zh-CN/gateway/security)和 [Tailscale](/zh-CN/gateway/tailscale)。

## WhatsApp 登录流程（远程）

- **在远程主机上**运行 `openclaw channels login --channel whatsapp --verbose`。使用手机上的 WhatsApp 扫描二维码。
- 如果身份验证过期，请在该主机上重新运行登录。健康检查会显示关联问题。

## 故障排除

| 症状                                          | 原因 / 修复方法                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `exit 127` / 未找到                           | 非登录 shell 的 PATH 中没有 `openclaw`。请将其添加到 `/etc/paths`、你的 shell rc 文件中，或创建符号链接到 `/usr/local/bin`/`/opt/homebrew/bin`。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| 健康探测失败                              | 检查 SSH 是否可达、PATH 是否正确，以及 Baileys (WhatsApp) 是否已登录（`openclaw status --json`）。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Web Chat 卡住                                   | 确认 Gateway 网关正在远程主机上运行，且转发端口与 Gateway 网关的 WS 端口一致；UI 需要健康的 WS 连接。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| 节点 IP 显示为 `127.0.0.1`                        | 使用 SSH 隧道时，这是预期行为。如果希望 Gateway 网关看到真实客户端 IP，请将 **Transport** 切换为 **Direct (ws/wss)**。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| 仪表板正常，但 Mac 功能处于离线状态 | 操作员/控制连接正常，但配套节点连接尚未连接或缺少其命令接口。打开菜单栏中的设备部分，检查 Mac 是否为 `paired · disconnected`。直接的 `wss://` 操作员连接和节点连接使用相同的已配置或已存储证书策略。对于受信任的 `wss://*.ts.net` Tailscale Serve 端点，证书轮换后，过期的已存储叶证书固定值会被替换，并自动重试。已配置的固定值绝不会自动轮换；请在检查新证书后更新 `gateway.remote.tlsFingerprint`，或切换到 **Remote over SSH**。 |
| 语音唤醒                                       | 在远程模式下，触发短语会自动转发，无需单独的转发器。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

## 通知声音

通过脚本使用 `openclaw nodes notify` 为每条通知选择声音，例如：

```bash
openclaw nodes notify --node <id> --title "Ping" --body "远程 Gateway 网关已就绪" --sound Glass
```

应用中没有全局默认声音开关；调用方为每个请求选择声音（或不使用声音）。

## 相关内容

- [macOS 应用](/zh-CN/platforms/macos)
- [远程访问](/zh-CN/gateway/remote)
