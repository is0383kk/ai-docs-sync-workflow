---
read_when:
    - 你想要在 Linux 伺服器或雲端 VPS 上執行閘道
    - 你需要一份託管指南的快速導覽圖
    - 你想要針對 OpenClaw 進行通用的 Linux 伺服器調校
sidebarTitle: Linux Server
summary: 在 Linux 伺服器或雲端 VPS 上執行 OpenClaw — 提供者選擇器、架構與調校
title: Linux 伺服器
x-i18n:
    generated_at: "2026-07-26T08:11:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 634a246850ab8b854c2c799688fd368ebed3a02124baa85bf38d5ff6ef8cec64
    source_path: vps.md
    workflow: 16
---

在任何 Linux 伺服器或雲端 VPS 上執行 OpenClaw 閘道。本頁協助你
選擇供應商、說明雲端部署的運作方式，並涵蓋適用於各種環境的通用 Linux
調校。

## 選擇供應商

<CardGroup cols={2}>
  <Card title="Azure" href="/zh-TW/install/azure">Linux 虛擬機器</Card>
  <Card title="DigitalOcean" href="/zh-TW/install/digitalocean">簡易付費 VPS</Card>
  <Card title="exe.dev" href="/zh-TW/install/exe-dev">具備 HTTPS Proxy 的虛擬機器</Card>
  <Card title="Fly.io" href="/zh-TW/install/fly">Fly Machines</Card>
  <Card title="GCP" href="/zh-TW/install/gcp">Compute Engine</Card>
  <Card title="Hetzner" href="/zh-TW/install/hetzner">Hetzner VPS 上的 Docker</Card>
  <Card title="Hostinger" href="/zh-TW/install/hostinger">支援一鍵設定的 VPS</Card>
  <Card title="Northflank" href="/zh-TW/install/northflank">一鍵式瀏覽器設定</Card>
  <Card title="Oracle Cloud" href="/zh-TW/install/oracle">永久免費 ARM 方案</Card>
  <Card title="Railway" href="/zh-TW/install/railway">一鍵式瀏覽器設定</Card>
  <Card title="Raspberry Pi" href="/zh-TW/install/raspberry-pi">ARM 自行託管</Card>
</CardGroup>

**AWS（EC2 / Lightsail / 免費方案）**也相當適用。
社群影片逐步教學位於
[x.com/techfrenAJ/status/2014934471095812547](https://x.com/techfrenAJ/status/2014934471095812547)
（社群資源——日後可能無法使用）。

## 雲端設定的運作方式

- **閘道在 VPS 上執行**，並管理狀態與工作區。
- 你可以從筆記型電腦或手機，透過**控制介面**或 **Tailscale/SSH** 連線。
- 將 VPS 視為單一真實來源，並定期**備份**狀態與工作區。
- 安全的預設做法：讓閘道維持在迴路介面上，並透過 SSH 通道或 Tailscale Serve 存取。
  如果繫結至 `lan` 或 `tailnet`，除非將驗證委派給
  受信任的 Proxy，否則閘道需要共用密鑰
  （`gateway.auth.token` 或 `gateway.auth.password`）。

相關頁面：[閘道遠端存取](/zh-TW/gateway/remote)、[平台中心](/zh-TW/platforms)。

## 先強化管理員存取安全性

在公用 VPS 上安裝 OpenClaw 之前，請先決定要如何管理
該主機本身。

- 若要僅透過 Tailnet 進行管理員存取：先安裝 Tailscale、將 VPS 加入你的
  Tailnet、透過 Tailscale IP 或 MagicDNS 名稱確認第二個 SSH 工作階段，
  然後限制公用 SSH。
- 不使用 Tailscale 時：在公開更多服務之前，先對你的 SSH 路徑套用同等的安全強化措施。
- 這與閘道存取無關。你仍可讓 OpenClaw 繫結至
  迴路介面，並使用 SSH 通道或 Tailscale Serve 存取儀表板。

Tailscale 專用的閘道選項請參閱 [Tailscale](/zh-TW/gateway/tailscale)。

## VPS 上的公司共用代理程式

如果每位使用者都處於相同的信任邊界，而且代理程式僅供業務使用，
讓團隊共用單一代理程式是可行的設定。

- 將其置於專用執行環境中（VPS/虛擬機器/容器，以及專用的作業系統使用者/帳號）。
- 不要讓該執行環境登入個人 Apple/Google 帳號，或個人瀏覽器/密碼管理器設定檔。
- 如果使用者彼此具有敵對性，請依閘道/主機/作業系統使用者分隔。

安全模型詳細資訊：[安全性](/zh-TW/gateway/security)。

## 搭配 VPS 使用節點

你可以將閘道保留在雲端，並在本機裝置
（Mac/iOS/Android/無頭裝置）上配對**節點**。節點提供本機螢幕/相機/畫布與 `system.run`
功能，而閘道則留在雲端。

文件：[節點](/zh-TW/nodes)、[節點命令列介面](/zh-TW/cli/nodes)。

## 小型虛擬機器與 ARM 主機的啟動調校

如果命令列介面命令在低效能虛擬機器（或 ARM 主機）上執行緩慢，請啟用 Node 的模組編譯快取：

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

- `NODE_COMPILE_CACHE` 可縮短重複執行命令的啟動時間；第一次執行會預熱快取。
- `OPENCLAW_NO_RESPAWN=1` 會讓例行閘道重新啟動維持在同一處理程序內，從而避免額外的處理程序交接，並簡化小型主機上的 PID 追蹤。
- 如需 Raspberry Pi 的特定資訊，請參閱 [Raspberry Pi](/zh-TW/install/raspberry-pi)。

### systemd 調校檢查清單（選用）

對於使用 `systemd` 的虛擬機器主機，請考慮：

- 用於穩定啟動路徑的服務環境變數：`OPENCLAW_NO_RESPAWN=1` 和
  `NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache`
- 明確的重新啟動行為：`Restart=always`、`RestartSec=2`、`TimeoutStartSec=90`
- 為狀態/快取路徑使用 SSD 儲存磁碟，以減少隨機 I/O 造成的冷啟動延遲。

標準 `openclaw onboard --install-daemon` 路徑會安裝 systemd 使用者
單元；使用下列命令編輯：

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

如果你刻意安裝的是系統單元，請改用
`sudo systemctl edit openclaw-gateway.service` 進行編輯。

`Restart=` 原則如何協助自動復原：
[systemd 可自動復原服務](https://www.redhat.com/en/blog/systemd-automate-recovery)。

如需 Linux OOM 行為、子處理程序終止目標選擇，以及 `exit 137`
診斷的相關資訊，請參閱 [Linux 記憶體壓力與 OOM 終止](/zh-TW/platforms/linux#memory-pressure-and-oom-kills)。

## 相關內容

- [安裝概覽](/zh-TW/install)
- [DigitalOcean](/zh-TW/install/digitalocean)
- [Fly.io](/zh-TW/install/fly)
- [Hetzner](/zh-TW/install/hetzner)
