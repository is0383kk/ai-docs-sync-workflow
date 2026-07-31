---
read_when:
    - 將 OpenClaw 部署至 Render
    - 你想使用 Render Blueprints 進行宣告式雲端部署
summary: 使用基礎架構即程式碼在 Render 上部署 OpenClaw
title: Render
x-i18n:
    generated_at: "2026-07-26T08:36:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a5fbb3c6df04e186df958a62a6130da4e3e485acfeecc7e85fee0d5b69a0438f
    source_path: install/render.mdx
    workflow: 16
---

使用儲存庫的 `render.yaml` Blueprint，將 OpenClaw 部署至 [Render](https://render.com)。它會在單一檔案中宣告服務、磁碟和環境變數。

## 先決條件

- 一個 [Render 帳號](https://render.com)（提供免費方案）
- 來自你偏好的[模型提供者](/zh-TW/providers)之 API 金鑰

## 部署

[部署至 Render](https://render.com/deploy?repo=https://github.com/openclaw/openclaw)

這會根據 `render.yaml` 建立 Render 服務、建置 Docker 映像，並進行部署。你的服務 URL 會遵循 `https://<service-name>.onrender.com` 格式。

## Blueprint

```yaml
services:
  - type: web
    name: openclaw
    runtime: docker
    plan: starter
    healthCheckPath: /health
    envVars:
      - key: OPENCLAW_GATEWAY_PORT
        value: "8080"
      - key: OPENCLAW_STATE_DIR
        value: /data/.openclaw
      - key: OPENCLAW_WORKSPACE_DIR
        value: /data/workspace
      - key: OPENCLAW_GATEWAY_TOKEN
        generateValue: true # 自動產生安全權杖
    disk:
      name: openclaw-data
      mountPath: /data
      sizeGB: 1
```

| 功能                  | 用途                                                       |
| --------------------- | ---------------------------------------------------------- |
| `runtime: docker`     | 使用儲存庫的 Dockerfile 建置                               |
| `healthCheckPath`     | Render 監控 `/health`，並重新啟動狀況異常的執行個體 |
| `generateValue: true` | 自動產生密碼學安全值                                       |
| `disk`                | 重新部署後仍會保留的持久性儲存空間                         |

## 選擇方案

| 方案      | 縮減至停止狀態       | 磁碟          | 最適合                       |
| --------- | ------------------- | ------------- | ---------------------------- |
| Free      | 閒置 15 分鐘後      | 不提供        | 測試、示範                   |
| Starter   | 永不                | 1GB+          | 個人使用、小型團隊           |
| Standard+ | 永不                | 1GB+          | 正式環境、多個頻道           |

Blueprint 預設使用 `starter`。若要使用免費方案，請變更你分支中的 `render.yaml` 內之 `plan: free`。請注意，由於沒有持久性磁碟，OpenClaw 狀態會在每次部署時重設。

## 部署後

### 存取控制介面

網頁儀表板位於 `https://<your-service>.onrender.com/`。請使用共用密鑰連線：自動產生的 `OPENCLAW_GATEWAY_TOKEN`（可在 **Dashboard → your service → Environment** 中找到）；若已切換為密碼驗證，則使用你的密碼。

### 日誌

**Dashboard → your service → Logs** 會顯示建置日誌（建立 Docker 映像）、部署日誌（服務啟動）及執行階段日誌（應用程式輸出）。

### Shell 存取

**Dashboard → your service → Shell** 會開啟 Shell 工作階段。持久性磁碟掛載於 `/data`。

### 環境變數

在 **Dashboard → your service → Environment** 中編輯變數。變更會觸發自動重新部署。

### 自動部署

當已連線儲存庫的分支收到新提交時，Render 會自動重新部署。如果你是直接從 `openclaw/openclaw` 部署，而非從自己的分支部署，便沒有推送權限可觸發重新部署；因此，請從 Dashboard 手動執行 Blueprint 同步以更新，或將服務指向你自己的分支。

## 自訂網域

1. **Dashboard → your service → Settings → Custom Domains**
2. 新增你的網域
3. 依指示設定 DNS（將 CNAME 指向 `*.onrender.com`）
4. Render 會自動佈建 TLS 憑證

## 擴充規模

- **垂直擴充**：變更方案以取得更多 CPU/RAM。通常足以供 OpenClaw 使用。
- **水平擴充**：增加執行個體數量（Standard 及以上方案）。由於 OpenClaw 將執行階段狀態保存在本機磁碟上，因此需要黏性工作階段或外部狀態管理。

## 備份與移轉

你可以隨時從 Render Dashboard Shell 匯出狀態、設定、驗證設定檔和工作區：

```bash
openclaw backup create
```

這會建立可攜式備份封存檔。請參閱[備份](/zh-TW/cli/backup)。

## 疑難排解

### 服務無法啟動

請檢查 Render Dashboard 中的部署日誌。常見問題：

- 缺少 `OPENCLAW_GATEWAY_TOKEN` — 請確認已在 **Dashboard → Environment** 中設定
- 連接埠不符 — 請確保 `OPENCLAW_GATEWAY_PORT=8080`，以便閘道繫結至 Render 預期的連接埠

### 冷啟動緩慢（免費方案）

免費方案的服務會在閒置 15 分鐘後縮減至停止狀態；停止後的第一個要求會在容器啟動期間耗時數秒。升級至 Starter 即可持續運作。

### 重新部署後資料遺失

這會發生在免費方案中（沒有持久性磁碟）。請升級至付費方案，或定期從 Render Shell 使用 `openclaw backup create` 匯出備份。

### 健康狀態檢查失敗

如果建置成功但部署失敗，可能是服務啟動時間過長，或無法存取 `/health`。請檢查：

- 建置日誌中是否有錯誤
- 容器是否能使用 `docker build && docker run` 在本機執行

## 後續步驟

- 設定訊息頻道：[頻道](/zh-TW/channels)
- 設定閘道：[閘道設定](/zh-TW/gateway/configuration)
- 讓 OpenClaw 保持最新狀態：[更新](/zh-TW/install/updating)
