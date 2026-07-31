---
read_when:
    - 設定新機器
    - 你想要「最新、最強」，又不想破壞個人設定
summary: OpenClaw 的進階設定與開發工作流程
title: 設定
x-i18n:
    generated_at: "2026-07-26T07:57:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c40d6d2bf2814465f3cc49c65d4c1498671420af728ce8012d13af3fba67025a
    source_path: start/setup.md
    workflow: 16
---

<Note>
如果你是第一次設定，請先參閱[開始使用](/zh-TW/start/getting-started)。
如需初始設定的詳細資訊，請參閱[初始設定（命令列介面）](/zh-TW/start/wizard)。
</Note>

## 重點摘要

請根據你希望更新的頻率，以及是否要自行執行閘道來選擇設定工作流程：

- **客製化內容放在儲存庫之外：**將設定與工作區保存在 `~/.openclaw/openclaw.json` 和 `~/.openclaw/workspace/`，如此儲存庫更新就不會影響它們。
- **穩定版工作流程（建議大多數人使用）：**安裝 macOS App，並由它執行內建的閘道。
- **最前沿工作流程（開發）：**透過 `pnpm gateway:watch` 自行執行閘道，再讓 macOS App 以本機模式連接。

## 必要條件（從原始碼執行）

- 建議使用節點 24.15+（仍支援節點 22 LTS，目前為 `22.22.3+`）
- 從原始碼簽出時需要 `pnpm`。在開發模式下，OpenClaw 會從
  `extensions/*` pnpm 工作區套件載入內建外掛，因此根目錄的 `npm install`
  不會準備完整的原始碼樹。
- Docker（選用；僅用於容器化設定／端對端測試，請參閱 [Docker](/zh-TW/install/docker)）

## 客製化策略（避免更新造成破壞）

如果你想要「100% 為我量身打造」，_同時_又能輕鬆更新，請將客製化內容保存在：

- **設定：**`~/.openclaw/openclaw.json`（JSON／類 JSON5）
- **工作區：**`~/.openclaw/workspace`（Skills、提示詞、記憶；請將其設為私人 Git 儲存庫）

只需初始化設定／工作區資料夾一次，而不執行完整的初始設定精靈：

```bash
openclaw setup --baseline
```

尚未全域安裝？改從此儲存庫執行：

```bash
pnpm openclaw setup --baseline
```

（不含 `--baseline` 的純 `openclaw setup` 是 `openclaw onboard` 的別名，會執行完整的互動式精靈。）

## 從此儲存庫執行閘道

完成 `pnpm build` 後，即可直接執行封裝的命令列介面：

```bash
node openclaw.mjs gateway --port 18789 --verbose
```

## 穩定版工作流程（優先使用 macOS App）

1. 安裝並啟動 **OpenClaw.app**（選單列）。
2. 完成初始設定／權限檢查清單（TCC 提示）。
3. 確認閘道設為 **Local** 且正在執行（由 App 管理）。
4. 連結通訊管道（例如 WhatsApp）：

```bash
openclaw channels login
```

5. 基本檢查：

```bash
openclaw health
```

如果你的建置版本無法使用初始設定：

- 依序執行 `openclaw setup` 和 `openclaw channels login`，再手動啟動閘道（`openclaw gateway`）。

## 最前沿工作流程（在終端機中執行閘道）

目標：開發 TypeScript 閘道、使用熱重新載入，並讓 macOS App 使用者介面保持連線。

### 0)（選用）也從原始碼執行 macOS App

如果也想使用最前沿版本的 macOS App：

```bash
./scripts/restart-mac.sh
```

### 1) 啟動開發版閘道

```bash
pnpm install
# 僅限第一次執行（或重設本機 OpenClaw 設定／工作區後）
pnpm openclaw setup
pnpm gateway:watch
```

`gateway:watch` 會在具名的 tmux 工作階段
（`openclaw-gateway-watch-main`）中啟動或重新啟動閘道監看程序，並從互動式
終端機自動連接。非互動式 Shell 會保持中斷連接並輸出
`tmux attach -t openclaw-gateway-watch-main`；使用
`OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch` 可讓互動式執行保持中斷連接，
或使用 `pnpm gateway:watch:raw` 啟用前景監看模式。監看器接管使用中設定檔的
已設定／預設連接埠之前，會停止該設定檔已安裝的閘道服務，以免服務監督程式
取代原始碼程序。該服務仍會保持安裝狀態；完成監看後請執行
`pnpm openclaw gateway start`。啟動失敗後，tmux 窗格仍可使用，
讓另一個終端機或代理程式能夠連接或擷取其日誌。監看器會在相關原始碼、
設定與內建外掛中繼資料變更時重新載入。如果受監看的閘道在啟動期間結束，
`gateway:watch` 會執行一次 `openclaw doctor --fix --non-interactive` 並重試；
設定 `OPENCLAW_GATEWAY_WATCH_AUTO_DOCTOR=0` 可停用這個僅供開發使用的修復階段。
`pnpm gateway:watch` 不會重新建置 `dist/control-ui`，因此在 `ui/` 變更後請重新執行 `pnpm ui:build`，或在開發控制介面時使用 `pnpm ui:dev`。

### 2) 將 macOS App 指向執行中的閘道

在 **OpenClaw.app** 中：

- Connection Mode：**Local**
  App 會連接至設定連接埠上執行中的閘道。

### 3) 驗證

- App 內的閘道狀態應顯示 **"Using existing gateway …"**
- 或透過命令列介面：

```bash
openclaw health
```

### 常見陷阱

- **連接埠錯誤：**閘道 WS 預設使用 `ws://127.0.0.1:18789`；App 與命令列介面必須使用相同的連接埠。
- **狀態儲存位置：**
  - 通訊管道／提供者狀態：`~/.openclaw/credentials/`
  - 模型驗證設定檔：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
  - 工作階段與文字記錄：`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
  - 舊版／封存的工作階段成品：`~/.openclaw/agents/<agentId>/sessions/`
  - 日誌：`/tmp/openclaw/`

## 認證資訊儲存對照表

偵錯驗證問題或決定備份項目時，請參考此對照表：

- **WhatsApp**：`~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- **Telegram 機器人權杖**：設定／環境變數或 `channels.telegram.tokenFile`（僅限一般檔案；不接受符號連結）
- **Discord 機器人權杖**：設定／環境變數或 SecretRef（環境變數／檔案／執行提供者）
- **Slack 權杖**：設定／環境變數（`channels.slack.*`）
- **配對允許清單**：
  - `~/.openclaw/credentials/<channel>-allowFrom.json`（預設帳號）
  - `~/.openclaw/credentials/<channel>-<accountId>-allowFrom.json`（非預設帳號）
- **模型驗證設定檔**：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- **檔案型祕密承載資料（選用）**：`~/.openclaw/secrets.json`
- **舊版 OAuth 匯入**：`~/.openclaw/credentials/oauth.json`
  更多詳細資訊：[安全性](/zh-TW/gateway/security#credential-storage-map)。

## 更新（不破壞你的設定）

- 將 `~/.openclaw/workspace` 和 `~/.openclaw/` 視為「你的內容」；不要將個人提示詞／設定放入 `openclaw` 儲存庫。
- 更新原始碼：`git pull` + `pnpm install`，並繼續使用 `pnpm gateway:watch`。

## Linux（systemd 使用者服務）

Linux 安裝會使用 systemd **使用者**服務。systemd 預設會在使用者
登出／閒置時停止使用者服務，因而終止閘道。初始設定會嘗試為你啟用
登入後持續執行（可能會提示輸入 sudo）。如果仍未啟用，請執行：

```bash
sudo loginctl enable-linger $USER
```

對於持續運作或多使用者伺服器，請考慮使用 **系統**服務，而非
使用者服務（不需要啟用登入後持續執行）。systemd 相關注意事項請參閱[閘道操作手冊](/zh-TW/gateway)。

## 相關文件

- [閘道操作手冊](/zh-TW/gateway)（旗標、監督、連接埠）
- [閘道設定](/zh-TW/gateway/configuration)（設定結構描述與範例）
- [Discord](/zh-TW/channels/discord) 和 [Telegram](/zh-TW/channels/telegram)（回覆標籤與 replyToMode 設定）
- [OpenClaw 助理設定](/zh-TW/start/openclaw)
- [macOS App](/zh-TW/platforms/macos)（閘道生命週期）
