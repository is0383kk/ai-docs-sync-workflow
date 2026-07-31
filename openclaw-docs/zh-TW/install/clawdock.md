---
read_when:
    - 你經常使用 Docker 執行 OpenClaw，並希望縮短日常使用的命令。
    - 你需要一個用於儀表板、日誌、權杖設定和配對流程的輔助層
summary: 用於 Docker 型 OpenClaw 安裝的 ClawDock shell 輔助工具
title: ClawDock
x-i18n:
    generated_at: "2026-07-26T07:54:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bb829a3301178503f910931e86a39f7befeaf186044f4088a25dc80ea99130d
    source_path: install/clawdock.md
    workflow: 16
---

ClawDock 是用於 Docker 型 OpenClaw 安裝的小型 Shell 輔助工具層。

它讓你使用 `clawdock-start`、`clawdock-dashboard` 和 `clawdock-fix-token` 等簡短命令，而不必使用較長的 `docker compose ...` 呼叫。

如果尚未設定 Docker，請先參閱 [Docker](/zh-TW/install/docker)。

## 安裝

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

如果你先前從 `scripts/shell-helpers/clawdock-helpers.sh` 安裝 ClawDock，請從目前的 `scripts/clawdock/clawdock-helpers.sh` 路徑重新安裝；舊的 GitHub 原始內容路徑已移除。

這些輔助工具會在首次使用時自動偵測你的 OpenClaw 原始碼簽出目錄（檢查 `~/openclaw`、`~/projects/openclaw` 等常見路徑），並將結果快取於 `~/.clawdock/config`。如果你的原始碼簽出目錄位於其他位置，請自行設定 `CLAWDOCK_DIR`。

## 提供的功能

### 基本操作

| 命令               | 說明                |
| ------------------ | ------------------- |
| `clawdock-start` | 啟動閘道            |
| `clawdock-stop` | 停止閘道            |
| `clawdock-restart` | 重新啟動閘道        |
| `clawdock-status` | 檢查容器狀態        |
| `clawdock-logs` | 持續顯示閘道日誌    |

### 存取容器

| 命令               | 說明                                  |
| ------------------ | ------------------------------------- |
| `clawdock-shell` | 在閘道容器內開啟 Shell               |
| `clawdock-cli <command>` | 在 Docker 中執行 OpenClaw 命令列介面命令 |
| `clawdock-exec <command>` | 在容器中執行任意命令                  |

### Web UI 與配對

| 命令               | 說明                   |
| ------------------ | ---------------------- |
| `clawdock-dashboard` | 開啟控制介面 URL       |
| `clawdock-devices` | 列出待處理的裝置配對   |
| `clawdock-approve <id>` | 核准配對要求           |

### 設定與維護

| 命令               | 說明                             |
| ------------------ | -------------------------------- |
| `clawdock-fix-token` | 將閘道權杖寫入容器設定           |
| `clawdock-update` | 拉取、重新建置並重新啟動         |
| `clawdock-rebuild` | 僅重新建置 Docker 映像檔         |
| `clawdock-clean` | 移除容器和磁碟區                 |

### 公用工具

| 命令               | 說明                         |
| ------------------ | ---------------------------- |
| `clawdock-health` | 執行閘道健康狀態檢查         |
| `clawdock-token` | 輸出閘道權杖                 |
| `clawdock-cd` | 前往 OpenClaw 專案目錄       |
| `clawdock-config` | 開啟 `~/.openclaw`      |
| `clawdock-show-config` | 輸出已遮蔽值的設定檔         |
| `clawdock-workspace` | 開啟工作區目錄               |
| `clawdock-help` | 列出所有 ClawDock 命令       |

## 首次使用流程

```bash
clawdock-start
clawdock-fix-token
clawdock-dashboard
```

如果瀏覽器顯示需要配對：

```bash
clawdock-devices
clawdock-approve <request-id>
```

## 設定與密鑰

ClawDock 會讀取兩個不同的 `.env` 檔案，與 [Docker](/zh-TW/install/docker) 中所述的拆分方式相符：

- 位於 `docker-compose.yml` 旁的專案 `.env`：包含映像檔名稱、連接埠和 `OPENCLAW_GATEWAY_TOKEN` 等 Docker 專用值。`clawdock-token` 會從此處讀取權杖。
- `~/.openclaw/.env`（掛載至容器中）：由 OpenClaw 本身管理、以環境變數為基礎的密鑰，與 `openclaw.json` 和 `agents/<agentId>/agent/auth-profiles.json` 並列。

`clawdock-fix-token` 會將權杖從專案 `.env` 複製至容器的 `gateway.remote.token` 和 `gateway.auth.token` 設定值，然後重新啟動閘道。

使用 `clawdock-show-config` 可快速檢查 `openclaw.json` 和兩個 `.env` 檔案；它會在輸出中遮蔽 `.env` 的值。

## 相關內容

<CardGroup cols={2}>
  <Card title="Docker" href="/zh-TW/install/docker" icon="docker">
    OpenClaw 的標準 Docker 安裝方式。
  </Card>
  <Card title="Docker VM 執行階段" href="/zh-TW/install/docker-vm-runtime" icon="cube">
    由 Docker 管理的 VM 執行階段，提供強化隔離。
  </Card>
  <Card title="更新" href="/zh-TW/install/updating" icon="arrow-up-right-from-square">
    更新 OpenClaw 套件和受管理的服務。
  </Card>
</CardGroup>
