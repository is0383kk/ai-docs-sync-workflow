---
read_when:
    - 你希望 OpenClaw 在 GCP 上全天候运行
    - 你希望在自己的虚拟机上运行一个生产级、始终在线的 Gateway 网关
    - 你希望完全控制持久化、二进制文件和重启行为
summary: 在 GCP Compute Engine VM（Docker）上全天候运行 OpenClaw Gateway 网关并持久保存状态
title: GCP
x-i18n:
    generated_at: "2026-07-26T06:17:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6ca46b2ee78731162261cae6ea5a26b718be6035b998fa92e4ee5c9ea2e7ae07
    source_path: install/gcp.md
    workflow: 16
---

使用 Docker 在 GCP Compute Engine 虚拟机上运行持久化的 OpenClaw Gateway 网关，并提供持久化状态、内置二进制文件和安全的重启行为。

价格因机器类型和区域而异；请选择能满足工作负载的最小虚拟机，如果遇到 OOM，再向上扩容。

可以通过笔记本电脑上的 SSH 端口转发访问 Gateway 网关；如果自行管理防火墙和令牌，也可以直接暴露端口。

本指南在 GCP Compute Engine 上使用 Debian。Ubuntu 也可使用；请相应调整软件包。有关通用 Docker 流程，请参阅 [Docker](/zh-CN/install/docker)。

## 所需条件

- GCP 账户（`e2-micro` 符合免费层资格）
- `gcloud` CLI，或 [Cloud Console](https://console.cloud.google.com)
- 从笔记本电脑进行 SSH 访问
- Docker 和 Docker Compose
- 模型身份验证凭据
- 可选的提供商凭据（WhatsApp 二维码、Telegram Bot 令牌、Gmail OAuth）
- 约 20-30 分钟

## 快速路径

1. 创建 GCP 项目，启用结算和 Compute Engine API
2. 创建 Compute Engine 虚拟机（`e2-small`、Debian 12、20GB）
3. 通过 SSH 登录虚拟机并安装 Docker
4. 克隆 OpenClaw 仓库
5. 创建持久化主机目录
6. 配置 `.env` 和 `docker-compose.yml`
7. 内置所需二进制文件、构建并启动

<Steps>
  <Step title="安装 gcloud CLI（或使用 Console）">
    从 [cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install) 安装，然后运行：

    ```bash
    gcloud init
    gcloud auth login
    ```

    也可以改为通过 [Cloud Console](https://console.cloud.google.com) Web UI 完成以下所有步骤。

  </Step>

  <Step title="创建 GCP 项目">
    ```bash
    gcloud projects create my-openclaw-project --name="OpenClaw Gateway"
    gcloud config set project my-openclaw-project
    gcloud services enable compute.googleapis.com
    ```

    在 [console.cloud.google.com/billing](https://console.cloud.google.com/billing) 启用结算（Compute Engine 必需）。

    Console 中的等效操作：IAM & Admin > Create Project，启用结算，然后依次选择 APIs & Services > Enable APIs > "Compute Engine API" > Enable。

  </Step>

  <Step title="创建虚拟机">
    | 类型      | 规格                     | 费用             | 说明                                      |
    | --------- | ------------------------ | ---------------- | ----------------------------------------- |
    | e2-medium | 2 vCPU、4GB RAM          | 约 $25/月        | 用于本地 Docker 构建最可靠                |
    | e2-small  | 2 vCPU、2GB RAM          | 约 $12/月        | Docker 构建的最低推荐配置                 |
    | e2-micro  | 2 vCPU（共享）、1GB RAM  | 符合免费层资格   | Docker 构建经常因 OOM 失败（退出码 137）  |

    ```bash
    gcloud compute instances create openclaw-gateway \
      --zone=us-central1-a \
      --machine-type=e2-small \
      --boot-disk-size=20GB \
      --image-family=debian-12 \
      --image-project=debian-cloud
    ```

  </Step>

  <Step title="通过 SSH 登录虚拟机">
    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    Console：在 Compute Engine 信息中心中，点击虚拟机旁边的 "SSH"。

    创建虚拟机后，SSH 密钥传播可能需要 1-2 分钟；如果连接被拒绝，请等待后重试。

  </Step>

  <Step title="安装 Docker（在虚拟机上）">
    ```bash
    sudo apt-get update
    sudo apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sudo sh
    sudo usermod -aG docker $USER
    ```

    注销并重新登录，使组更改生效，然后再次通过 SSH 登录：

    ```bash
    exit
    ```

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    验证：

    ```bash
    docker --version
    docker compose version
    ```

  </Step>

  <Step title="克隆 OpenClaw 仓库">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    本指南会构建自定义镜像，因此内置的所有二进制文件在重启后仍会保留。

  </Step>

  <Step title="创建持久化主机目录">
    Docker 容器是临时的；所有长期状态都必须存放在主机上。

    ```bash
    mkdir -p ~/.openclaw
    mkdir -p ~/.openclaw/workspace
    ```

  </Step>

  <Step title="配置环境变量">
    在仓库根目录中创建 `.env`：

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/home/$USER/.openclaw
    OPENCLAW_WORKSPACE_DIR=/home/$USER/.openclaw/workspace

    GOG_KEYRING_PASSWORD=
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    设置 `OPENCLAW_GATEWAY_TOKEN`，通过 `.env` 管理稳定的 Gateway 网关令牌；否则，在依赖客户端跨重启连接之前配置 `gateway.auth.token`。如果两者均未设置，OpenClaw 会为该次启动使用仅限运行时的令牌。为 `GOG_KEYRING_PASSWORD` 生成密钥环密码：

    ```bash
    openssl rand -hex 32
    ```

    **不要提交此文件。**其中包含 `OPENCLAW_GATEWAY_TOKEN` 等容器/运行时环境变量。存储的提供商 OAuth/API 密钥身份验证信息位于挂载的 `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` 中。

  </Step>

  <Step title="Docker Compose 配置">
    创建或更新 `docker-compose.yml`：

    ```yaml
    services:
      openclaw-gateway:
        image: ${OPENCLAW_IMAGE}
        build: .
        restart: unless-stopped
        env_file:
          - .env
        environment:
          - HOME=/home/node
          - NODE_ENV=production
          - TERM=xterm-256color
          - OPENCLAW_GATEWAY_BIND=${OPENCLAW_GATEWAY_BIND}
          - OPENCLAW_GATEWAY_PORT=${OPENCLAW_GATEWAY_PORT}
          - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
          - GOG_KEYRING_PASSWORD=${GOG_KEYRING_PASSWORD}
          - XDG_CONFIG_HOME=${XDG_CONFIG_HOME}
          - PATH=/home/linuxbrew/.linuxbrew/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        volumes:
          - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
          - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
        ports:
          # 推荐：让 Gateway 网关仅在虚拟机的回环地址上监听；通过 SSH 隧道访问。
          # 要公开暴露，请移除 `127.0.0.1:` 前缀，并相应配置防火墙。
          - "127.0.0.1:${OPENCLAW_GATEWAY_PORT}:18789"
        command:
          [
            "node",
            "dist/index.js",
            "gateway",
            "--bind",
            "${OPENCLAW_GATEWAY_BIND}",
            "--port",
            "${OPENCLAW_GATEWAY_PORT}",
            "--allow-unconfigured",
          ]
    ```

    `--allow-unconfigured` 仅用于方便引导启动，不能替代真正的 Gateway 网关配置。仍需为部署设置身份验证（`gateway.auth.token` 或密码）以及安全的绑定模式。

  </Step>

  <Step title="共享 Docker 虚拟机运行时步骤">
    按照共享运行时指南完成通用 Docker 主机流程：

    - [将所需二进制文件内置到镜像中](/zh-CN/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [构建并启动](/zh-CN/install/docker-vm-runtime#build-and-launch)
    - [各类数据的持久化位置](/zh-CN/install/docker-vm-runtime#what-persists-where)
    - [更新](/zh-CN/install/docker-vm-runtime#updates)

  </Step>

  <Step title="GCP 特定启动说明">
    如果在 `pnpm install --frozen-lockfile` 期间构建因 `Killed` 或 `exit code 137` 失败，则虚拟机内存不足。至少使用 `e2-small`，或者使用 `e2-medium` 以提高首次构建的可靠性。

    绑定到 LAN（`OPENCLAW_GATEWAY_BIND=lan`）时，请先配置受信任的浏览器来源再继续：

    ```bash
    docker compose run --rm openclaw-cli config set gateway.controlUi.allowedOrigins '["http://127.0.0.1:18789"]' --strict-json
    ```

    如果更改了端口，请将 `18789` 替换为所配置的端口。

  </Step>

  <Step title="从笔记本电脑访问">
    创建 SSH 隧道以转发 Gateway 网关端口：

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a -- -L 18789:127.0.0.1:18789
    ```

    在浏览器中打开 `http://127.0.0.1:18789/`。

    重新输出一个简洁的信息中心链接：

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

    如果 UI 提示进行共享密钥身份验证，请将配置的令牌或密码粘贴到 Control UI 设置中（此 Docker 流程默认写入令牌；如果已切换为密码身份验证，请改用配置的密码）。

    如果 Control UI 显示 `unauthorized` 或 `disconnected (1008): pairing required`，请批准浏览器设备：

    ```bash
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    有关共享持久化映射，请参阅 [Docker 虚拟机运行时](/zh-CN/install/docker-vm-runtime#what-persists-where)；有关更新方式，请参阅[更新流程](/zh-CN/install/docker-vm-runtime#updates)。

  </Step>
</Steps>

## 故障排查

**SSH 连接被拒绝**

创建虚拟机后，SSH 密钥传播可能需要 1-2 分钟。请等待后重试。

**OS Login 问题**

检查 OS Login 配置文件：

```bash
gcloud compute os-login describe-profile
```

确保账户具有所需的 IAM 权限（Compute OS Login 或 Compute OS Admin Login）。

**内存不足（OOM）**

如果 Docker 构建因 `Killed` 和 `exit code 137` 失败，则虚拟机进程已被 OOM 终止：

```bash
# 先停止虚拟机
gcloud compute instances stop openclaw-gateway --zone=us-central1-a

# 更改机器类型
gcloud compute instances set-machine-type openclaw-gateway \
  --zone=us-central1-a \
  --machine-type=e2-small

# 启动虚拟机
gcloud compute instances start openclaw-gateway --zone=us-central1-a
```

## 服务账号（安全最佳实践）

对于个人使用，默认用户账号即可。对于自动化或 CI/CD，请创建权限最小化的专用服务账号：

```bash
gcloud iam service-accounts create openclaw-deploy \
  --display-name="OpenClaw Deployment"

gcloud projects add-iam-policy-binding my-openclaw-project \
  --member="serviceAccount:openclaw-deploy@my-openclaw-project.iam.gserviceaccount.com" \
  --role="roles/compute.instanceAdmin.v1"
```

避免为自动化使用 Owner 角色；应使用能够正常工作的最小权限角色。请参阅[了解角色](https://cloud.google.com/iam/docs/understanding-roles)。

## 后续步骤

- 设置消息渠道：[渠道](/zh-CN/channels)
- 将本地设备配对为节点：[节点](/zh-CN/nodes)
- 配置 Gateway 网关：[Gateway 配置](/zh-CN/gateway/configuration)

## 相关内容

- [安装概览](/zh-CN/install)
- [Azure](/zh-CN/install/azure)
- [VPS 托管](/zh-CN/vps)
