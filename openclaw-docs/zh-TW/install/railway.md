---
read_when:
    - 將 OpenClaw 部署至 Railway
    - 你想要一鍵式雲端部署，並搭配瀏覽器式控制介面
summary: 使用一鍵式範本在 Railway 上部署 OpenClaw
title: Railway
x-i18n:
    generated_at: "2026-07-26T08:22:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbef00b8de61545e9971b18164472c2f47fe607f69ec36f83a27a11b65ea863f
    source_path: install/railway.mdx
    workflow: 16
---

使用一鍵式範本將 OpenClaw 部署至 Railway，並透過網頁版控制介面存取。這是最簡單的「伺服器上不需要終端機」方式：Railway 會為你執行閘道。

## 一鍵部署

<a href="https://railway.com/deploy/clawdbot-railway-template" target="_blank" rel="noreferrer">
  Deploy on Railway
</a>

<Steps>
  <Step title="部署範本">
    按一下上方的 **Deploy on Railway**。
  </Step>

<Step title="新增磁碟區">
  掛接一個掛載於 `/data` 的磁碟區（持久保存狀態所必需）。
</Step>

  <Step title="設定變數">
    在服務上設定必要的 **Variables**：

    - `OPENCLAW_GATEWAY_PORT=8080`（必需——必須與 Public Networking 中的連接埠相符）
    - `OPENCLAW_GATEWAY_TOKEN`（必需；視為管理員密鑰）
    - `OPENCLAW_STATE_DIR=/data/.openclaw`（建議）
    - `OPENCLAW_WORKSPACE_DIR=/data/workspace`（建議）

  </Step>

<Step title="啟用公開網路">
  在 **Public Networking** 下，為服務的連接埠 `8080` 啟用 **HTTP Proxy**。
</Step>

  <Step title="連線">
    在 **Railway -> your service -> Settings -> Domains** 中尋找你的公開 URL——可能是產生的網域（通常為 `https://<something>.up.railway.app`），或是你掛接的自訂網域。

    開啟 `https://<your-railway-domain>/openclaw`，並使用設定的共用密鑰連線。範本預設使用 `OPENCLAW_GATEWAY_TOKEN`；如果你將其替換為密碼驗證，請改用該密碼。

  </Step>
</Steps>

## 你會獲得

- 託管的 OpenClaw 閘道與控制介面
- 透過 Railway Volume（`/data`）提供持久儲存空間，因此 `openclaw.json`、各代理程式的 `auth-profiles.json`、頻道／提供者狀態、工作階段與工作區都能在重新部署後保留

## 連接頻道

使用 `/openclaw` 的控制介面，或透過 Railway 的 shell 執行 `openclaw onboard`，以取得頻道設定說明：

- [Discord](/zh-TW/channels/discord)
- [Telegram](/zh-TW/channels/telegram)（最快——只需要機器人權杖）
- [所有頻道](/zh-TW/channels)

## 備份與移轉

匯出你的狀態、設定、驗證設定檔與工作區：

```bash
openclaw backup create
```

這會建立一個可攜式備份封存檔，其中包含 OpenClaw 狀態及所有已設定的工作區。詳情請參閱[備份](/zh-TW/cli/backup)。

## 後續步驟

- 設定訊息頻道：[頻道](/zh-TW/channels)
- 設定閘道：[閘道設定](/zh-TW/gateway/configuration)
- 讓 OpenClaw 保持最新版本：[更新](/zh-TW/install/updating)
