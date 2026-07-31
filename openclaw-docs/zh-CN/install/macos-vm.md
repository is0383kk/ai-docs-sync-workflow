---
read_when:
    - 你希望将 OpenClaw 与主要的 macOS 环境隔离开来
    - 你想在沙箱中集成 iMessage
    - 你需要一个可重置且可克隆的 macOS 环境
    - 你想比较本地与托管的 macOS 虚拟机方案
summary: 当你需要隔离环境或使用 iMessage 时，请在沙箱隔离的 macOS 虚拟机（本地或托管）中运行 OpenClaw
title: macOS 虚拟机
x-i18n:
    generated_at: "2026-07-26T06:20:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7e6b963faaf40f65adce1081715bc295059b8bed278a8c71a05a86e04ad7a7a5
    source_path: install/macos-vm.md
    workflow: 16
---

## 推荐默认方案（适合大多数用户）

- 使用**小型 Linux VPS**，以低成本保持 Gateway 网关始终在线。请参阅 [VPS 托管](/zh-CN/vps)。
- 如果需要完全控制，并希望使用**住宅 IP**进行浏览器自动化，请选择**专用硬件**（Mac mini 或 Linux 设备）。许多网站会屏蔽数据中心 IP，因此本地浏览通常效果更好。
- **混合方案**：将 Gateway 网关部署在廉价 VPS 上，需要进行浏览器/UI 自动化时，将 Mac 作为**节点**连接。请参阅[节点](/zh-CN/nodes)和[远程 Gateway 网关](/zh-CN/gateway/remote)。

仅当确实需要 iMessage 等 macOS 独有功能，或希望与日常使用的 Mac 严格隔离时，才使用 macOS VM。

## macOS VM 选项

### Apple Silicon Mac 上的本地 VM（Lume）

使用 [Lume](https://cua.ai/docs/lume)，在现有 Apple Silicon Mac 上的沙箱隔离 macOS VM 中运行 OpenClaw。这样可以获得：

- 隔离的完整 macOS 环境（保持宿主机整洁）
- 通过 `imsg` 支持 iMessage；默认本地路径无法在 Linux/Windows 上使用
- 通过克隆 VM 即时重置
- 无需额外硬件或云服务费用

### 托管式 Mac 提供商（云端）

如果希望在云端使用 macOS，也可以选择托管式 Mac 提供商：

- [MacStadium](https://www.macstadium.com/)（托管式 Mac）
- 其他托管式 Mac 供应商也可以使用；请遵循其 VM + SSH 文档

获得 macOS VM 的 SSH 访问权限后，继续执行下方的[安装 OpenClaw](#6-install-openclaw)。

## 快速路径（Lume，适合有经验的用户）

1. 安装 Lume。
2. `lume create openclaw --os macos --ipsw latest`
3. 完成 Setup Assistant，启用 Remote Login（SSH）。
4. `lume run openclaw --no-display`
5. 通过 SSH 登录，安装 OpenClaw，并配置渠道。
6. 完成。

## 所需条件（Lume）

- Apple Silicon Mac（M1/M2/M3/M4）
- 宿主机运行 macOS Sequoia 或更高版本
- 每个 VM 约需 60 GB 可用磁盘空间
- 约 20 分钟

## 1) 安装 Lume

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/trycua/cua/main/libs/lume/scripts/install.sh)"
```

如果 `~/.local/bin` 不在 PATH 中：

```bash
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc && source ~/.zshrc
```

验证：

```bash
lume --version
```

文档：[Lume 安装](https://cua.ai/docs/lume/guide/getting-started/installation)

## 2) 创建 macOS VM

```bash
lume create openclaw --os macos --ipsw latest
```

这会下载 macOS 并创建 VM。VNC 窗口将自动打开。

<Note>
下载可能需要一些时间，具体取决于网络连接。
</Note>

## 3) 完成 Setup Assistant

在 VNC 窗口中：

1. 选择语言和地区。
2. 跳过 Apple ID（如果之后想使用 iMessage，也可以登录）。
3. 创建用户账户（记住用户名和密码）。
4. 跳过所有可选功能。

设置完成后：

1. 启用 SSH：System Settings -> General -> Sharing，启用 "Remote Login"。
2. 如需以无头模式使用 VM，请启用自动登录：System Settings -> Users & Groups，选择 "Automatically log in as:"，然后选择 VM 用户。

## 4) 获取 VM IP 地址

```bash
lume get openclaw
```

查找 IP 地址（通常为 `192.168.64.x`）。

## 5) 通过 SSH 登录 VM

```bash
ssh youruser@192.168.64.X
```

将 `youruser` 替换为创建的账户，并将 IP 替换为 VM 的 IP。

## 6) 安装 OpenClaw

在 VM 内：

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

按照新手引导提示设置模型提供商（Anthropic、OpenAI 等）。

## 7) 配置渠道

编辑配置文件：

```bash
nano ~/.openclaw/openclaw.json
```

添加渠道：

```json5
{
  channels: {
    telegram: {
      botToken: "YOUR_BOT_TOKEN",
    },
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],
    },
  },
}
```

然后登录 WhatsApp（扫描二维码）：

```bash
openclaw channels login
```

## 8) 以无头模式运行 VM

停止 VM，然后在不显示界面的情况下重新启动：

```bash
lume stop openclaw
lume run openclaw --no-display
```

VM 会在后台运行；OpenClaw 守护进程会让 Gateway 网关持续运行。检查状态：

```bash
ssh youruser@192.168.64.X "openclaw status"
```

## 额外功能：iMessage 集成

这是在 macOS 上运行的杀手级功能。使用 [iMessage](/zh-CN/channels/imessage) 和 `imsg` 将 Messages 添加到 OpenClaw。

在 VM 内：

1. 登录 Messages。
2. 安装 `imsg`。
3. 为运行 OpenClaw/`imsg` 的进程授予 Full Disk Access 和 Automation 权限。
4. 使用 `imsg rpc --help` 验证 RPC 支持。

添加到 OpenClaw 配置：

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
    },
  },
}
```

重启 Gateway 网关。智能体现在可以发送和接收 iMessage。完整设置详情：[iMessage 渠道](/zh-CN/channels/imessage)。

## 保存黄金镜像

进一步自定义之前，请为干净状态创建快照：

```bash
lume stop openclaw
lume clone openclaw openclaw-golden
```

随时重置：

```bash
lume stop openclaw && lume delete openclaw
lume clone openclaw-golden openclaw
lume run openclaw --no-display
```

## 运行 24/7

通过以下方式保持 VM 运行：

- 保持 Mac 接通电源
- 在 System Settings -> Energy Saver 中禁用睡眠
- 需要时使用 `caffeinate`

如需真正保持始终在线，请考虑使用专用 Mac mini 或小型 VPS。请参阅 [VPS 托管](/zh-CN/vps)。

## 故障排查

| 问题                     | 解决方案                                                                            |
| ------------------------ | ----------------------------------------------------------------------------------- |
| 无法通过 SSH 登录 VM     | 检查 VM 的 System Settings 中是否已启用 "Remote Login"                              |
| 未显示 VM IP             | 等待 VM 完全启动，然后再次运行 `lume get openclaw`                                  |
| 找不到 Lume 命令         | 将 `~/.local/bin` 添加到 PATH                                                   |
| WhatsApp 二维码无法扫描  | 运行 `openclaw channels login` 时，确保登录的是 VM（而非宿主机）                           |

## 相关文档

- [VPS 托管](/zh-CN/vps)
- [节点](/zh-CN/nodes)
- [远程 Gateway 网关](/zh-CN/gateway/remote)
- [iMessage 渠道](/zh-CN/channels/imessage)
- [Lume 快速开始](https://cua.ai/docs/lume/guide/getting-started/quickstart)
- [Lume CLI 参考](https://cua.ai/docs/lume/reference/cli-reference)
- [无人值守 VM 设置](https://cua.ai/docs/lume/guide/fundamentals/unattended-setup)（高级）
- [Docker 沙箱隔离](/zh-CN/install/docker)（替代隔离方案）
