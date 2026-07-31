---
read_when:
    - 你希望 OpenClaw 在雲端 VPS 上全天候執行（而不是在你的筆電上）
    - 你想在自己的 VPS 上執行可用於正式環境且持續運作的閘道
    - 你希望完整掌控持久性、二進位檔與重新啟動行為
    - 你正在 Hetzner 或類似供應商的 Docker 上執行 OpenClaw
summary: 在低成本的 Hetzner VPS 上使用 Docker 全天候執行 OpenClaw 閘道，並具備持久化狀態與內建二進位檔案
title: Hetzner
x-i18n:
    generated_at: "2026-07-26T08:20:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ffebc0ce725fd219d13d0a556940327e70dab810b8fbee0b365c4870dc7109b
    source_path: install/hetzner.md
    workflow: 16
---

使用 Docker 在 Hetzner VPS 上執行持久化的 OpenClaw 閘道，具備持久狀態、內建二進位檔，以及安全的重新啟動行為。

Hetzner 的定價會變動；請選擇符合需求的最小型 Debian/Ubuntu VPS，若遇到記憶體不足（OOM），再升級規格。

你可以從筆記型電腦透過 SSH 連接埠轉送存取閘道；若自行管理防火牆與權杖，也可以直接公開連接埠。

安全模型提醒：

- 若所有人都處於相同的信任邊界，且執行環境僅供業務用途，則公司共用代理程式沒有問題。
- 保持嚴格隔離：使用專用 VPS／執行環境與專用帳號；不要在該主機上使用個人的 Apple／Google／瀏覽器／密碼管理器設定檔。
- 若使用者彼此具有敵對性，請依閘道／主機／作業系統使用者進行隔離。

請參閱[安全性](/zh-TW/gateway/security)與 [VPS 託管](/zh-TW/vps)。

本指南假設 Hetzner 上執行的是 Ubuntu 或 Debian。若使用其他 Linux VPS，請相應調整套件。一般 Docker 流程請參閱 [Docker](/zh-TW/install/docker)。

## 所需項目

- 具有 root 存取權的 Hetzner VPS
- 可從筆記型電腦進行 SSH 存取
- Docker 與 Docker Compose
- 模型驗證認證資訊
- 選用的供應商認證資訊（WhatsApp QR、Telegram 機器人權杖、Gmail OAuth）
- 約 20 分鐘

## 快速流程

1. 佈建 Hetzner VPS
2. 安裝 Docker
3. 複製 OpenClaw 儲存庫
4. 建立持久化主機目錄
5. 設定 `.env` 與 `docker-compose.yml`
6. 將必要的二進位檔內建至映像檔
7. `docker compose up -d`
8. 驗證持久性與閘道存取

<Steps>
  <Step title="佈建 VPS">
    在 Hetzner 建立 Ubuntu 或 Debian VPS，然後以 root 身分連線：

    ```bash
    ssh root@YOUR_VPS_IP
    ```

    將 VPS 視為具狀態的基礎架構，而不是可隨時棄置的基礎架構。

  </Step>

  <Step title="安裝 Docker（在 VPS 上）">
    ```bash
    apt-get update
    apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sh
    ```

    驗證：

    ```bash
    docker --version
    docker compose version
    ```

  </Step>

  <Step title="複製 OpenClaw 儲存庫">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    本指南會建置自訂映像檔，確保內建的所有二進位檔在重新啟動後仍然存在。

  </Step>

  <Step title="建立持久化主機目錄">
    Docker 容器是暫時性的；所有長期狀態都必須存放在主機上。

    ```bash
    mkdir -p /root/.openclaw/workspace

    # 將擁有者設為容器使用者（uid 1000）：
    chown -R 1000:1000 /root/.openclaw
    ```

  </Step>

  <Step title="設定環境變數">
    在儲存庫根目錄建立 `.env`：

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/root/.openclaw
    OPENCLAW_WORKSPACE_DIR=/root/.openclaw/workspace

    GOG_KEYRING_PASSWORD=
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    設定 `OPENCLAW_GATEWAY_TOKEN`，以透過
    `.env` 管理穩定的閘道權杖；否則，在重新啟動後繼續依賴用戶端之前，請先設定 `gateway.auth.token`。若兩者皆未設定，OpenClaw 會為
    該次啟動使用僅限執行階段的權杖。為 `GOG_KEYRING_PASSWORD` 產生鑰匙圈密碼：

    ```bash
    openssl rand -hex 32
    ```

    **請勿提交此檔案。**其中包含容器／執行環境的環境變數，例如
    `OPENCLAW_GATEWAY_TOKEN`。已儲存的供應商 OAuth／API 金鑰驗證資訊位於
    掛載的 `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` 中。

  </Step>

  <Step title="Docker Compose 設定">
    建立或更新 `docker-compose.yml`：

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
          # 建議：讓 VPS 上的閘道僅限回送介面存取；透過 SSH 通道連線。
          # 若要公開存取，請移除 `127.0.0.1:` 前綴，並相應設定防火牆。
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

    `--allow-unconfigured` 僅用於方便初始啟動，無法取代正式的閘道設定。部署時仍須設定驗證（`gateway.auth.token` 或密碼）與安全的繫結模式。

  </Step>

  <Step title="共用 Docker VM 執行環境步驟">
    依照共用執行環境指南完成一般 Docker 主機流程：

    - [將必要的二進位檔內建至映像檔](/zh-TW/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [建置並啟動](/zh-TW/install/docker-vm-runtime#build-and-launch)
    - [各項資料的持久化位置](/zh-TW/install/docker-vm-runtime#what-persists-where)
    - [更新](/zh-TW/install/docker-vm-runtime#updates)

  </Step>

  <Step title="Hetzner 特定存取方式">
    完成共用的建置與啟動步驟後，開啟通道。

    **必要條件：**確認 VPS 的 sshd 設定允許 TCP 轉送。若你
    強化過 SSH 設定，請檢查 `/etc/ssh/sshd_config` 並設定：

    ```text
    AllowTcpForwarding local
    ```

    `local` 允許從筆記型電腦使用 `ssh -L` 本機轉送，同時阻擋
    從伺服器發起的遠端轉送。將其設為 `no` 會導致通道失敗，並顯示：
    `channel 3: open failed: administratively prohibited: open failed`

    確認已啟用 TCP 轉送後，重新啟動 SSH 服務
    （`systemctl restart ssh`），然後從筆記型電腦執行通道：

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP
    ```

    開啟 `http://127.0.0.1:18789/`，並貼上已設定的共用密鑰。
    本指南預設使用閘道權杖；若已切換為密碼驗證，
    請改用設定的密碼。

  </Step>
</Steps>

共用持久化對照表位於 [Docker VM 執行環境](/zh-TW/install/docker-vm-runtime#what-persists-where)。

## 基礎架構即程式碼（Terraform）

對於偏好基礎架構即程式碼工作流程的團隊，社群維護的 Terraform 設定提供：

- 具備遠端狀態管理的模組化 Terraform 設定
- 透過 cloud-init 自動佈建
- 部署指令碼（初始啟動、部署、備份／還原）
- 安全強化（防火牆、UFW、僅限 SSH 存取）
- 用於存取閘道的 SSH 通道設定

**儲存庫：**

- 基礎架構：[openclaw-terraform-hetzner](https://github.com/andreesg/openclaw-terraform-hetzner)
- Docker 設定：[openclaw-docker-config](https://github.com/andreesg/openclaw-docker-config)

此方式為上述 Docker 設定補充可重現的部署、版本控制的基礎架構，以及自動化災難復原功能。

<Note>
由社群維護。如需回報問題或參與貢獻，請參閱上方的儲存庫連結。
</Note>

## 後續步驟

- 設定訊息通道：[通道](/zh-TW/channels)
- 設定閘道：[閘道設定](/zh-TW/gateway/configuration)
- 讓 OpenClaw 保持最新版本：[更新](/zh-TW/install/updating)

## 相關內容

- [安裝概覽](/zh-TW/install)
- [Fly.io](/zh-TW/install/fly)
- [Docker](/zh-TW/install/docker)
- [VPS 託管](/zh-TW/vps)
