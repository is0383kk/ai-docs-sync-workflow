---
read_when:
    - 在 DigitalOcean 上设置 OpenClaw
    - 寻找适合 OpenClaw 的简单付费 VPS
summary: 在 DigitalOcean Droplet 上托管 OpenClaw
title: DigitalOcean
x-i18n:
    generated_at: "2026-07-26T06:49:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e124a59c079efda0c8e880018f2657fad784af1489ca3f98ed8ab609249e35bd
    source_path: install/digitalocean.md
    workflow: 16
---

在 DigitalOcean Droplet 上运行持久化的 OpenClaw Gateway 网关（1 GB Basic 方案每月约 $6）。

DigitalOcean 是一种简单直接的付费 VPS 方案。如需更便宜或免费的选项：

- [Hetzner](/zh-CN/install/hetzner)——每美元可获得更多核心数和 RAM。
- [Oracle Cloud](/zh-CN/install/oracle)——永久免费 ARM 层级（最高 4 OCPU、24 GB RAM），但注册过程可能不太顺利，而且仅支持 ARM。

## 前置条件

- DigitalOcean 账户（[注册](https://cloud.digitalocean.com/registrations/new)）
- SSH 密钥对（或愿意使用密码身份验证）
- 大约 20 分钟

## 设置

<Steps>
  <Step title="创建 Droplet">
    <Warning>
    使用干净的基础镜像（Ubuntu 24.04 LTS）。除非已审查第三方 Marketplace 一键镜像的启动脚本和防火墙默认设置，否则请勿使用。
    </Warning>

    1. 登录 [DigitalOcean](https://cloud.digitalocean.com/)。
    2. 点击 **Create > Droplets**。
    3. 选择：
       - **Region:** 距离你最近的区域
       - **Image:** Ubuntu 24.04 LTS
       - **Size:** Basic、Regular、1 vCPU / 1 GB RAM / 25 GB SSD
       - **Authentication:** SSH key（推荐）或 password
    4. 点击 **Create Droplet**，并记下 IP 地址。

  </Step>

  <Step title="连接并安装">
    ```bash
    ssh root@YOUR_DROPLET_IP

    apt update && apt upgrade -y

    # 安装 Node.js 24
    curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
    apt install -y nodejs

    # 安装 OpenClaw
    curl -fsSL https://openclaw.ai/install.sh | bash

    # 创建拥有 OpenClaw 状态和服务的非 root 用户。
    adduser openclaw
    usermod -aG sudo openclaw
    loginctl enable-linger openclaw

    su - openclaw
    openclaw --version
    ```

    仅使用 root shell 进行系统引导设置。以非 root `openclaw` 用户身份运行 OpenClaw 命令，以便状态存放在 `/home/openclaw/.openclaw/` 下，并将 Gateway 网关安装为该用户的 systemd `--user` 服务。

  </Step>

  <Step title="运行新手引导">
    ```bash
    openclaw onboard --install-daemon
    ```

    向导会引导你完成模型身份验证、渠道设置、Gateway 网关令牌生成和守护进程安装（systemd 用户服务）。

  </Step>

  <Step title="添加交换空间（建议用于 1 GB Droplet）">
    ```bash
    fallocate -l 2G /swapfile
    chmod 600 /swapfile
    mkswap /swapfile
    swapon /swapfile
    echo '/swapfile none swap sw 0 0' >> /etc/fstab
    ```
  </Step>

  <Step title="验证 Gateway 网关">
    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```
  </Step>

  <Step title="访问 Control UI">
    Gateway 网关默认绑定到环回地址。请选择以下选项之一。

    **选项 A：SSH 隧道（最简单）**

    ```bash
    # 在本地计算机上运行
    ssh -L 18789:localhost:18789 root@YOUR_DROPLET_IP
    ```

    然后打开 `http://localhost:18789`。

    **选项 B：Tailscale Serve**

    ```bash
    curl -fsSL https://tailscale.com/install.sh | sudo sh
    sudo tailscale up
    openclaw config set gateway.tailscale.mode serve
    openclaw gateway restart
    ```

    然后从 tailnet 中的任意设备打开 `https://<magicdns>/`。

    Tailscale Serve 通过 tailnet 身份标头对 Control UI 和 WebSocket 流量进行身份验证，这假定 Gateway 网关主机本身可信。无论如何，HTTP API 端点仍遵循 Gateway 网关的正常身份验证模式（令牌/密码）。若要在 Serve 上要求显式共享密钥凭据，请设置 `gateway.auth.allowTailscale: false`，并使用 `gateway.auth.mode: "token"` 或 `"password"`。

    **选项 C：绑定到 Tailnet（不使用 Serve）**

    ```bash
    openclaw config set gateway.bind tailnet
    openclaw gateway restart
    ```

    然后打开 `http://<tailscale-ip>:18789`（需要令牌）。

  </Step>
</Steps>

## 持久化和备份

OpenClaw 状态位于：

- `~/.openclaw/`——`openclaw.json`、渠道/提供商凭据、每个智能体的 `auth-profiles.json` 和会话数据。
- `~/.openclaw/workspace/`——Agent 工作区（SOUL.md、记忆和工件）。

这些内容在 Droplet 重启后仍会保留。要创建可移植的快照，请运行：

```bash
openclaw backup create
```

DigitalOcean 快照会备份整个 Droplet；`openclaw backup create` 可跨主机移植。

## 1 GB RAM 使用技巧

$6 的 Droplet 只有 1 GB RAM。为保持流畅运行：

- 确保将上面的交换空间步骤写入 `/etc/fstab`，使其在重启后仍然生效。
- 优先使用基于 API 的模型（Claude、GPT），而非本地模型——1 GB 内存无法容纳本地 LLM 推理。
- 如果处理大型提示词时遇到 OOM，请将 `agents.defaults.model.primary` 设置为更小的模型。
- 使用 `free -h` 和 `htop` 进行监控。

## 故障排查

**Gateway 网关无法启动**——运行 `openclaw doctor --non-interactive`，并使用 `journalctl --user -u openclaw-gateway.service -n 50` 检查日志。

**端口已被占用**——运行 `lsof -i :18789` 查找相应进程，然后将其停止。

**内存不足**——使用 `free -h` 验证交换空间是否已启用。如果仍然遇到 OOM，请改用基于 API 的模型（Claude、GPT），而非本地模型；或者升级到 2 GB Droplet。

## 后续步骤

- [渠道](/zh-CN/channels)——连接 Telegram、WhatsApp、Discord 等
- [Gateway 配置](/zh-CN/gateway/configuration)——所有配置选项
- [更新](/zh-CN/install/updating)——使 OpenClaw 保持最新状态

## 相关内容

- [安装概览](/zh-CN/install)
- [Fly.io](/zh-CN/install/fly)
- [Hetzner](/zh-CN/install/hetzner)
- [VPS 托管](/zh-CN/vps)
