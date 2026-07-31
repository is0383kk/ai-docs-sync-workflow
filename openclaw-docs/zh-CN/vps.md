---
read_when:
    - 你想在 Linux 服务器或云 VPS 上运行 Gateway 网关
    - 你需要一份托管指南速览图
    - 你需要针对 OpenClaw 的通用 Linux 服务器调优方案
sidebarTitle: Linux Server
summary: 在 Linux 服务器或云 VPS 上运行 OpenClaw — 提供商选择、架构和调优
title: Linux 服务器
x-i18n:
    generated_at: "2026-07-26T06:32:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 634a246850ab8b854c2c799688fd368ebed3a02124baa85bf38d5ff6ef8cec64
    source_path: vps.md
    workflow: 16
---

在任何 Linux 服务器或云 VPS 上运行 OpenClaw Gateway 网关。本页帮助你
选择提供商，说明云端部署的工作方式，并介绍适用于各种环境的通用 Linux
调优方法。

## 选择提供商

<CardGroup cols={2}>
  <Card title="Azure" href="/zh-CN/install/azure">Linux 虚拟机</Card>
  <Card title="DigitalOcean" href="/zh-CN/install/digitalocean">简单的付费 VPS</Card>
  <Card title="exe.dev" href="/zh-CN/install/exe-dev">带 HTTPS 代理的虚拟机</Card>
  <Card title="Fly.io" href="/zh-CN/install/fly">Fly Machines</Card>
  <Card title="GCP" href="/zh-CN/install/gcp">Compute Engine</Card>
  <Card title="Hetzner" href="/zh-CN/install/hetzner">Hetzner VPS 上的 Docker</Card>
  <Card title="Hostinger" href="/zh-CN/install/hostinger">支持一键设置的 VPS</Card>
  <Card title="Northflank" href="/zh-CN/install/northflank">一键式浏览器设置</Card>
  <Card title="Oracle Cloud" href="/zh-CN/install/oracle">永久免费 ARM 层级</Card>
  <Card title="Railway" href="/zh-CN/install/railway">一键式浏览器设置</Card>
  <Card title="Raspberry Pi" href="/zh-CN/install/raspberry-pi">ARM 自托管</Card>
</CardGroup>

**AWS（EC2 / Lightsail / 免费层级）**也能良好运行。
社区视频演示可在
[x.com/techfrenAJ/status/2014934471095812547](https://x.com/techfrenAJ/status/2014934471095812547)
查看（社区资源——可能会失效）。

## 云端设置的工作方式

- **Gateway 网关在 VPS 上运行**，并管理状态和工作区。
- 你可以通过 **Control UI** 或 **Tailscale/SSH** 从笔记本电脑或手机连接。
- 将 VPS 视为唯一可信来源，并定期**备份**状态和工作区。
- 安全默认设置：让 Gateway 网关保持在回环地址上，并通过 SSH 隧道或 Tailscale Serve 访问。
  如果绑定到 `lan` 或 `tailnet`，Gateway 网关需要共享密钥
  （`gateway.auth.token` 或 `gateway.auth.password`），除非将身份验证委托给
  受信任的代理。

相关页面：[Gateway 网关远程访问](/zh-CN/gateway/remote)、[平台中心](/zh-CN/platforms)。

## 首先强化管理访问

在公共 VPS 上安装 OpenClaw 之前，请确定要如何管理
服务器本身。

- 对于仅限 Tailnet 的管理访问：先安装 Tailscale，将 VPS 加入你的
  tailnet，通过 Tailscale IP 或 MagicDNS 名称验证第二个 SSH 会话，
  然后限制公共 SSH。
- 不使用 Tailscale 时：在公开更多服务之前，对 SSH 访问路径应用
  等效的安全强化措施。
- 这与 Gateway 网关访问相互独立。你仍可让 OpenClaw 绑定到
  回环地址，并使用 SSH 隧道或 Tailscale Serve 访问仪表板。

Tailscale 专用的 Gateway 网关选项请参阅 [Tailscale](/zh-CN/gateway/tailscale)。

## VPS 上的公司共享智能体

当每个用户都处于相同的信任边界内，并且智能体仅用于业务时，
为团队运行单个智能体是一种可行的设置。

- 将其置于专用运行时中（VPS/虚拟机/容器 + 专用操作系统用户/账户）。
- 不要在该运行时中登录个人 Apple/Google 账户或个人浏览器/密码管理器配置文件。
- 如果用户之间存在对抗关系，请按 Gateway 网关/主机/操作系统用户进行隔离。

安全模型详情：[安全](/zh-CN/gateway/security)。

## 将节点与 VPS 配合使用

你可以将 Gateway 网关保留在云端，并在本地设备
（Mac/iOS/Android/无头设备）上配对**节点**。节点提供本地屏幕/摄像头/画布和 `system.run`
能力，而 Gateway 网关仍保留在云端。

文档：[节点](/zh-CN/nodes)、[节点 CLI](/zh-CN/cli/nodes)。

## 小型虚拟机和 ARM 主机的启动调优

如果 CLI 命令在低功耗虚拟机（或 ARM 主机）上运行缓慢，请启用 Node 的模块编译缓存：

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

- `NODE_COMPILE_CACHE` 可缩短重复执行命令时的启动时间；首次运行会预热缓存。
- `OPENCLAW_NO_RESPAWN=1` 使常规 Gateway 网关重启保持在进程内执行，从而避免额外的进程交接，并简化小型主机上的 PID 跟踪。
- 有关 Raspberry Pi 的具体信息，请参阅 [Raspberry Pi](/zh-CN/install/raspberry-pi)。

### systemd 调优清单（可选）

对于使用 `systemd` 的虚拟机主机，请考虑：

- 为稳定的启动路径设置服务环境：`OPENCLAW_NO_RESPAWN=1` 和
  `NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache`
- 明确的重启行为：`Restart=always`、`RestartSec=2`、`TimeoutStartSec=90`
- 为状态/缓存路径使用 SSD 后端磁盘，以减少随机 I/O 导致的冷启动性能损失。

标准 `openclaw onboard --install-daemon` 路径会安装 systemd 用户
单元；使用以下命令编辑：

```bash
systemctl --user edit openclaw-gateway.service
```

```ini
[Service]
Environment=OPENCLAW_NO_RESPAWN=1
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Restart=always
RestartSec=2
TimeoutStartSec=90
```

如果你有意安装的是系统单元，请通过
`sudo systemctl edit openclaw-gateway.service` 进行编辑。

`Restart=` 策略如何帮助自动恢复：
[systemd 可以自动恢复服务](https://www.redhat.com/en/blog/systemd-automate-recovery)。

有关 Linux OOM 行为、子进程终止目标选择以及 `exit 137`
诊断，请参阅 [Linux 内存压力和 OOM 终止](/zh-CN/platforms/linux#memory-pressure-and-oom-kills)。

## 相关内容

- [安装概览](/zh-CN/install)
- [DigitalOcean](/zh-CN/install/digitalocean)
- [Fly.io](/zh-CN/install/fly)
- [Hetzner](/zh-CN/install/hetzner)
