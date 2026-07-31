---
read_when:
    - 啟動新的 OpenClaw 代理程式工作階段
    - 啟用或稽核預設 Skills
summary: 個人助理設定的預設 OpenClaw 代理程式指示與 Skills 清單
title: 預設 AGENTS.md
x-i18n:
    generated_at: "2026-07-26T08:04:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 645342f8c6e2805135817cf4bbc2c8bd1d57066054ed671eda93876b2762ffb1
    source_path: reference/AGENTS.default.md
    workflow: 16
---

## 第一次執行（建議）

OpenClaw 代理程式使用工作區目錄。預設值：`~/.openclaw/workspace`（可透過 `agents.defaults.workspace` 設定，支援 `~`）。

1. 建立工作區：

```bash
mkdir -p ~/.openclaw/workspace
```

2. 將預設工作區範本複製到其中：

```bash
cp docs/reference/templates/AGENTS.md ~/.openclaw/workspace/AGENTS.md
cp docs/reference/templates/SOUL.md ~/.openclaw/workspace/SOUL.md
cp docs/reference/templates/TOOLS.md ~/.openclaw/workspace/TOOLS.md
```

3. 選用：使用此檔案的個人助理 Skills 清單，而非通用範本：

```bash
cp docs/reference/AGENTS.default.md ~/.openclaw/workspace/AGENTS.md
```

4. 選用：指向不同的工作區：

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

## 安全預設值

- 不要將目錄內容或祕密傾印到聊天中。
- 除非明確要求，否則不要執行破壞性命令。
- 變更設定或排程器（crontab、systemd 單元、nginx 設定、shell rc 檔案）之前，請先檢查現有狀態，並預設保留或合併現有內容。
- 不要向外部訊息介面傳送部分或串流回覆（僅傳送最終回覆）。

## 現有解決方案預檢

在提議或建置自訂系統、功能、工作流程、工具、整合或自動化之前，先檢查是否已有開放原始碼專案、持續維護的函式庫、現有 OpenClaw 外掛或免費平台，足以解決需求。若適用，應優先採用。僅當現有選項不適用、成本過高、缺乏維護、不安全、不符規範，或使用者明確要求自訂方案時，才進行自訂建置。除非使用者明確同意付費，否則避免推薦付費服務。此步驟應保持輕量，作為預檢關卡，而非研究任務。

## 工作階段開始（必要）

- 回覆前，請先閱讀 `SOUL.md`、`USER.md`，以及 `memory/` 中今天與昨天的內容。
- 若 `MEMORY.md` 存在，請閱讀該檔案。

## 靈魂（必要）

- `SOUL.md` 定義身分、語氣與界線。請保持其內容為最新狀態。
- 如果變更 `SOUL.md`，請告知使用者。
- 每個工作階段都是全新的執行個體；連續性保存在這些檔案中。

## 共用空間（建議）

- 你不代表使用者發言；在群組聊天或公開頻道中務必謹慎。
- 不要分享私人資料、聯絡資訊或內部筆記。

## 記憶系統（建議）

- 每日紀錄：`memory/YYYY-MM-DD.md`（如有需要，請建立 `memory/`）。
- 長期記憶：使用 `MEMORY.md` 保存持久性事實、偏好與決策。
- 小寫的 `memory.md` 僅供舊版修復輸入使用；請勿刻意同時保留兩個根目錄檔案。
- 工作階段開始時，請閱讀今天、昨天，以及存在時的 `MEMORY.md`。
- 寫入記憶檔案前，請先讀取內容；僅寫入具體更新，切勿寫入空白預留內容。
- 記錄：決策、偏好、限制條件、未完成事項。
- 除非明確要求，否則避免記錄祕密。

## 工具與 Skills

- 工具位於 Skills 中；需要使用時，請遵循各 Skill 的 `SKILL.md`。
- 將環境特定的筆記保存在 `TOOLS.md`（Skills 的筆記）中。

## 備份提示（建議）

將此工作區視為助理的記憶：把它建立為 git 儲存庫（最好是私人儲存庫），以便備份 `AGENTS.md` 與記憶檔案。

```bash
cd ~/.openclaw/workspace
git init
git add AGENTS.md
git commit -m "Add workspace"
# 選用：新增私人遠端儲存庫並推送
```

## OpenClaw 的功能

- 執行訊息頻道閘道（WhatsApp、Telegram、Discord、Signal、iMessage、Slack 等）及內嵌代理程式，讓助理能讀寫聊天訊息、擷取脈絡，並透過主機執行 Skills。
- macOS 應用程式管理權限（螢幕錄製、通知、麥克風），並透過其內附的二進位檔提供 `openclaw` 命令列介面。
- 直接聊天預設會合併至代理程式的 `main` 工作階段；群組及頻道／聊天室則各自取得工作階段金鑰。確切的金鑰格式請參閱[頻道路由](/zh-TW/channels/channel-routing)。心跳偵測會讓背景工作持續運作。

## 核心 Skills（在 Settings → Skills 中啟用）

以下是個人助理工作區的範例清單；請依你的設定替換為適合的 Skills。

- **mcporter** - 用於管理外部 Skill 後端的工具伺服器執行環境／命令列介面。
- **Peekaboo** - 快速擷取 macOS 螢幕截圖，並可選擇使用 AI 視覺分析。
- **camsnap** - 從 RTSP／ONVIF 安全攝影機擷取影格、短片或移動警示。
- **oracle** - 支援 OpenAI 的代理程式命令列介面，具備工作階段重播與瀏覽器控制功能。
- **eightctl** - 從終端機控制你的睡眠。
- **imsg** - 傳送、讀取及串流 iMessage 與 SMS。
- **wacli** - WhatsApp 命令列介面：同步、搜尋、傳送。
- **discord** - Discord 動作：回應、貼圖、投票。使用 `user:<id>` 或 `channel:<id>` 目標（單獨的數字 ID 語意不明確）。
- **gog** - Google Suite 命令列介面：Gmail、Calendar、Drive、Contacts。
- **spotify-player** - 用於搜尋、排入佇列及控制播放的終端機 Spotify 用戶端。
- **sag** - 採用 macOS 風格 say 使用體驗的 ElevenLabs 語音工具；預設串流至揚聲器。
- **Sonos CLI** - 從指令碼控制 Sonos 揚聲器（探索／狀態／播放／音量／群組）。
- **blucli** - 從指令碼播放、群組及自動控制 BluOS 播放器。
- **OpenHue CLI** - 控制 Philips Hue 照明的情境與自動化。
- **OpenAI Whisper** - 在本機將語音轉換為文字，適合快速聽寫及語音信箱轉錄。
- **Gemini CLI** - 從終端機使用 Google Gemini 模型進行快速問答。
- **agent-tools** - 用於自動化與輔助指令碼的公用工具組。

## 使用注意事項

- 編寫指令碼時，優先使用 `openclaw` 命令列介面；桌面應用程式負責處理權限。
- 從 Skills 分頁執行安裝；必要的二進位檔已存在時，安裝按鈕會隱藏。
- 保持啟用心跳偵測，讓助理能排程提醒、監控收件匣及觸發攝影機擷取。
- Canvas 使用者介面以全螢幕搭配原生覆疊層執行。避免將重要控制項放在左上角、右上角或底部邊緣；請加入明確的版面邊距，而非依賴安全區域內距。
- 進行瀏覽器驅動的驗證時，請使用 `openclaw browser` 命令列介面（內附的 `browser` 外掛），並搭配由 OpenClaw 管理的 Chrome／Brave／Edge／Chromium 設定檔。
- 管理：`status`、`doctor [--deep]`、`start [--headless]`、`stop`、`tabs`、`tab [new|select|close]`、`open <url>`、`focus <id>`、`close <id>`。
- 檢查：`screenshot [--full-page|--ref|--labels]`、`snapshot [--format ai|aria|--interactive|--efficient]`、`console`、`errors`、`requests`、`pdf`、`responsebody`。
- 操作：`navigate`、`click <ref>`、`type <ref> <text>`、`press`、`hover`、`drag`、`select`、`upload`、`download`、`fill`、`dialog`、`wait`、`evaluate --fn <js>`、`highlight`。動作需要來自 `snapshot` 的 `ref`（動作不接受 CSS 選擇器）；需要 `document.querySelector` 風格的目標指定時，請使用 `evaluate`。
- 在任何檢查命令中加入 `--json`，即可取得機器可讀的輸出。

## 相關內容

- [代理程式工作區](/zh-TW/concepts/agent-workspace)
- [代理程式執行環境](/zh-TW/concepts/agent)
- [頻道路由](/zh-TW/channels/channel-routing)
