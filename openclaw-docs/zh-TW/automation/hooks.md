---
read_when:
    - 你想要針對 /new、/reset、/stop 和代理程式生命週期事件進行事件驅動的自動化作業
    - 你想要建置、安裝或偵錯鉤子
summary: 鉤子：針對命令與生命週期事件的事件驅動自動化
title: 鉤子
x-i18n:
    generated_at: "2026-07-26T07:42:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 039a55cca60e0005d7b9c4d950a86aceb6e7c29d5768108b34011bfc21c85be6
    source_path: automation/hooks.md
    workflow: 16
---

鉤子是當代理程式事件觸發時，在閘道內執行的小型指令碼：例如 `/new`、`/reset`、`/stop` 等命令、工作階段壓縮、閘道生命週期及訊息流程。系統會從目錄中探索這些鉤子，並透過 `openclaw hooks` 管理。只有在你啟用鉤子，或設定至少一個鉤子項目、鉤子套件、舊版處理常式或額外鉤子目錄後，閘道才會載入內部鉤子。

OpenClaw 中有兩種鉤子：

- **內部鉤子**（本頁）：當代理程式事件觸發時，在閘道內執行。
- **網路鉤子**：外部 HTTP 端點，可讓其他系統觸發 OpenClaw 中的工作。請參閱[網路鉤子](/zh-TW/automation/cron-jobs#webhooks)。

鉤子也可以封裝在外掛內。`openclaw hooks list` 會同時顯示獨立鉤子與由外掛管理的鉤子（顯示為 `plugin:<id>`）。

## 選擇正確的介面

OpenClaw 有數種看似相似、但用於解決不同問題的擴充介面：

| 如果你想要……                                                                                                     | 使用……                                | 原因                                                                                           |
| --------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------- |
| 在 `/new` 時儲存快照、記錄 `/reset`、在 `message:sent` 後呼叫外部 API，或新增粗粒度的營運自動化 | 內部鉤子（`HOOK.md`，本頁） | 檔案式鉤子適用於由營運人員管理的副作用，以及命令／生命週期自動化 |
| 重寫提示詞、封鎖工具、取消外送訊息，或新增有順序的中介軟體／政策                              | 透過 `api.on(...)` 使用具型別的外掛鉤子  | 具型別的鉤子具有明確的合約、優先順序、合併規則，以及封鎖／取消語意      |
| 新增僅限遙測的匯出或可觀測性                                                                            | 診斷事件                     | 可觀測性使用獨立的事件匯流排，而非政策鉤子介面                              |

若你想要行為類似小型已安裝整合的自動化，請使用內部鉤子。若你需要控制執行階段生命週期，請使用具型別的外掛鉤子。

## 快速開始

```bash
# 列出可用的鉤子
openclaw hooks list

# 啟用鉤子
openclaw hooks enable session-memory

# 檢查鉤子狀態
openclaw hooks check

# 取得詳細資訊
openclaw hooks info session-memory
```

## 事件類型

鉤子會訂閱此表中的特定鍵，或訂閱不含動作的系列名稱
（`command`、`session`、`agent`、`gateway`、`message`），以接收
該系列中的每個動作。OpenClaw 核心不會發出其他事件，因此任何其他名稱幾乎
一定是拼字錯誤，會讓鉤子悄無聲息地無法運作（只有發出
自訂事件的外掛可能觸發它）。鉤子載入器會針對這類名稱記錄警告
（例如 `command:nwe`），而 `openclaw hooks info <name>` 也會標示它們，因此
從未執行的鉤子可以被診斷。

| 事件                    | 觸發時機                                              |
| ------------------------ | ---------------------------------------------------------- |
| `command:new`            | 發出 `/new` 命令時                                      |
| `command:reset`          | 發出 `/reset` 命令時                                    |
| `command:stop`           | 發出 `/stop` 命令時                                     |
| `command`                | 任何命令事件（通用監聽器）                       |
| `session:compact:before` | 壓縮摘要歷史記錄之前                       |
| `session:compact:after`  | 壓縮完成之後                                 |
| `session:patch`          | 修改工作階段屬性時                       |
| `agent:bootstrap`        | 注入工作區啟動程序檔案之前              |
| `gateway:startup`        | 頻道啟動且鉤子載入後                  |
| `gateway:shutdown`       | 閘道開始關閉時                               |
| `gateway:pre-restart`    | 預期重新啟動閘道之前                         |
| `message:received`       | 來自任何頻道的傳入訊息                           |
| `message:transcribed`    | 音訊轉錄完成後                        |
| `message:preprocessed`   | 媒體與連結預處理完成或遭略過後 |
| `message:sent`           | 嘗試外送時（`context.success` 包含結果） |

## 撰寫鉤子

### 鉤子結構

每個鉤子都是包含兩個檔案的目錄：

```text
my-hook/
├── HOOK.md          # 中繼資料 + 文件
└── handler.ts       # 處理常式實作
```

處理常式檔案可以是 `handler.ts`、`handler.js`、`index.ts` 或 `index.js`。

### HOOK.md 格式

```markdown
---
name: my-hook
description: "此鉤子功能的簡短說明"
metadata:
  { "openclaw": { "emoji": "🔗", "events": ["command:new"], "requires": { "bins": ["node"] } } }
---

# 我的鉤子

詳細文件寫在這裡。
```

**中繼資料欄位**（`metadata.openclaw`）：

| 欄位      | 說明                                          |
| ---------- | ---------------------------------------------------- |
| `emoji`    | 命令列介面的顯示表情符號                                |
| `events`   | 要監聽的事件陣列                        |
| `export`   | 要使用的具名匯出（預設為 `"default"`）        |
| `os`       | 必要平台（例如 `["darwin", "linux"]`）     |
| `requires` | 必要的 `bins`、`anyBins`、`env` 或 `config` 路徑 |
| `always`   | 略過資格檢查（布林值）                  |
| `hookKey`  | 設定鍵覆寫（預設為鉤子名稱）      |
| `homepage` | `openclaw hooks info` 顯示的文件 URL              |
| `install`  | 安裝方式                                 |

### 處理常式實作

```typescript
const handler = async (event) => {
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  console.log(`[my-hook] 已觸發新命令`);
  // 你的邏輯寫在這裡

  // 可選擇在可回覆的介面上傳送回覆
  event.messages.push("鉤子已執行！");
};

export default handler;
```

每個事件都包含：`type`、`action`、`sessionKey`、`timestamp`、`messages` 與 `context`（事件特定資料）。代理程式與工具鉤子的具型別外掛鉤子內容也可以包含 `trace`，這是唯讀且與 W3C 相容的診斷追蹤內容，外掛可將其傳入結構化記錄，以進行 OTEL 關聯。

推送至 `event.messages` 的字串只會針對
`command:new` 和 `command:reset` 傳回聊天（路由為原始
對話的回覆），以及針對 `session:compact:before`／`session:compact:after`
傳送（作為壓縮狀態通知）。所有其他事件，包括
`command:stop`、`message:*`、`agent:bootstrap`、`session:patch` 與
`gateway:*`，都會忽略推送的訊息。

### 事件內容重點

**命令事件**（`command:new`、`command:reset`）：`context.sessionEntry`、`context.previousSessionEntry`、`context.commandSource`、`context.senderId`、`context.workspaceDir`、`context.cfg`。

**命令事件**（`command:stop`）：`context.sessionEntry`、`context.sessionId`、`context.commandSource`、`context.senderId`。

**訊息事件**（`message:received`）：`context.from`、`context.content`、`context.channelId`、`context.media`（依序排列的已暫存附件事實）、`context.originalMedia`，以及遠端媒體尚未在本機暫存時的 `context.mediaStagingPending`，還有 `context.metadata`（供應商特定資料，包括 `senderId`、`senderName`、`guildId`）。`context.content` 會優先採用類命令訊息中的非空白命令本文，接著退回原始傳入本文與通用本文；它不包含僅供代理程式使用的增補資料，例如討論串歷史記錄或連結摘要。`metadata` 內的舊版媒體別名已棄用。

**訊息事件**（`message:sent`）：`context.to`、`context.content`、`context.success`、`context.channelId`，以及傳送失敗時的 `context.error`。

**訊息事件**（`message:transcribed`）：`context.transcript`、`context.from`、`context.channelId` 與 `context.media`。`context.mediaPath` 與 `context.mediaType` 仍是第一項事實的已棄用別名。

**訊息事件**（`message:preprocessed`）：`context.bodyForAgent`（最終增補本文）、`context.from`、`context.channelId`。

**啟動程序事件**（`agent:bootstrap`）：`context.bootstrapFiles`（可變陣列）、`context.agentId`。

**工作階段修補事件**（`session:patch`）：`context.sessionEntry`、`context.patch`（僅變更的欄位）、`context.cfg`。只有具特殊權限的用戶端能觸發修補事件；內容是複本，因此處理常式無法改變即時工作階段項目。

**壓縮事件**：`session:compact:before` 包含 `messageCount`、`tokenCount`。`session:compact:after` 另加入 `compactedCount`、`summaryLength`、`tokensBefore`、`tokensAfter`。

`command:stop` 會觀察使用者發出 `/stop`；這屬於取消／命令
生命週期，而非代理程式定稿閘門。需要檢查
自然產生的最終答案，並要求代理程式再執行一次的外掛，應改用具型別的
外掛鉤子 `before_agent_finalize`。請參閱[外掛鉤子](/plugins/hooks)。

**閘道生命週期事件**：`gateway:shutdown` 包含 `reason` 與 `restartExpectedMs`，並在閘道開始關閉時觸發。`gateway:pre-restart` 包含相同內容，但只有在關閉屬於預期重新啟動的一部分，且提供有限的 `restartExpectedMs` 值時才會觸發。關閉期間，每個生命週期鉤子的等待都會盡力執行並受到時間限制，因此即使處理常式停滯，關閉程序仍會繼續。`gateway:shutdown` 的預設等待預算為 5 秒，`gateway:pre-restart` 則為 10 秒。

頻道仍可使用時，請使用 `gateway:pre-restart` 傳送簡短的重新啟動通知：

```typescript
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const execFileAsync = promisify(execFile);

export default async function handler(event) {
  if (event.type !== "gateway" || event.action !== "pre-restart") {
    return;
  }

  const restartInSeconds = Math.ceil(event.context.restartExpectedMs / 1000);
  await execFileAsync("openclaw", [
    "system",
    "event",
    "--mode",
    "now",
    "--text",
    `閘道將在約 ${restartInSeconds} 秒後重新啟動（${event.context.reason}）。請立即建立檢查點。`,
  ]);
}
```

在 `gateway:shutdown`（或 `gateway:pre-restart`）事件與關閉程序的其餘部分之間，閘道也會針對程序停止時仍處於作用中的每個工作階段，觸發具型別的 `session_end` 外掛鉤子。一般 SIGTERM／SIGINT 停止時，事件的 `reason` 為 `shutdown`；若關閉是預期重新啟動的一部分，則為 `restart`。此排空作業受到時間限制，因此緩慢的 `session_end` 處理常式無法阻止程序結束；已透過取代／重設／刪除／壓縮完成定稿的工作階段則會略過，以避免重複觸發。

## 鉤子探索

鉤子會從四個來源探索：

1. **內建掛鉤**：隨 OpenClaw 一併提供
2. **外掛掛鉤**：內建於已安裝的外掛中；可覆寫同名的內建掛鉤
3. **受管理的掛鉤**：`~/.openclaw/hooks/`（由使用者安裝，跨工作區共用）；可覆寫內建和外掛掛鉤。來自 `hooks.internal.load.extraDirs` 的額外目錄具有相同的優先順序。
4. **工作區掛鉤**：`<workspace>/hooks/`（各代理程式獨立，預設停用，直到明確啟用）

工作區掛鉤可以新增掛鉤名稱，但無法覆寫同名的內建、受管理或外掛提供的掛鉤。

在設定內部掛鉤之前，閘道啟動時會略過內部掛鉤探索。使用 `openclaw hooks enable <name>` 啟用內建或受管理的掛鉤、安裝掛鉤套件，或設定 `hooks.internal.enabled=true` 以選擇啟用。啟用一個指定掛鉤時，閘道只會載入該掛鉤的處理常式；`hooks.internal.enabled=true`、額外掛鉤目錄和舊版處理常式則會選擇啟用廣泛探索。

### 掛鉤套件

掛鉤套件是透過 `package.json` 中的 `openclaw.hooks` 匯出掛鉤的 npm 套件。使用以下命令安裝：

```bash
openclaw plugins install <path-or-spec>
```

Npm 規格僅限登錄檔（套件名稱 + 選用的確切版本或 dist-tag）。Git／URL／檔案規格和 semver 範圍會遭拒絕。舊版的 `openclaw hooks install` 和 `openclaw hooks update` 命令已棄用，分別是 `openclaw plugins install`／`openclaw plugins update` 的別名。

## 內建掛鉤

| 掛鉤                  | 事件                                            | 功能                                                   |
| --------------------- | ------------------------------------------------- | -------------------------------------------------------------- |
| session-memory        | `command:new`, `command:reset`                    | 將工作階段內容儲存至 `<workspace>/memory/`                 |
| bootstrap-extra-files | `agent:bootstrap`                                 | 從 glob 模式注入額外的啟動檔案          |
| command-logger        | `command`                                         | 將所有命令記錄至 `~/.openclaw/logs/commands.log`           |
| compaction-notifier   | `session:compact:before`, `session:compact:after` | 在工作階段壓縮開始／結束時傳送可見的聊天通知 |
| boot-md               | `gateway:startup`                                 | 在閘道啟動時執行 `BOOT.md`                         |

啟用任一內建掛鉤：

```bash
openclaw hooks enable <hook-name>
```

<a id="session-memory"></a>

### session-memory 詳細資訊

擷取最後幾則使用者／助理訊息（預設 15 則，可使用 `hooks.internal.entries.session-memory.messages` 設定），並使用主機本機日期將它們儲存至 `<workspace>/memory/YYYY-MM-DD-HHMM.md`。記憶擷取會在背景執行，因此 `/new` 和 `/reset` 確認訊息不會因讀取逐字稿或選用的 slug 產生作業而延遲。設定 `hooks.internal.entries.session-memory.llmSlug: true` 以產生描述性的檔名 slug，並可選擇將 `hooks.internal.entries.session-memory.model` 設為已設定的別名（例如 `sonnet`）、代理程式預設提供者上的純模型 ID，或 `provider/model` 參照。省略 `model` 時，slug 產生會使用代理程式的預設模型；若無法使用，則改用時間戳記 slug。必須設定 `workspace.dir`。

<a id="bootstrap-extra-files"></a>

### bootstrap-extra-files 設定

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "bootstrap-extra-files": {
          "enabled": true,
          "paths": ["packages/*/AGENTS.md", "packages/*/TOOLS.md"]
        }
      }
    }
  }
}
```

`patterns` 和 `files` 可作為 `paths` 的別名。路徑會相對於工作區解析，且必須位於工作區內。只會載入可辨識的啟動基本檔名（`AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md`、`BOOTSTRAP.md`、`MEMORY.md`）。

<a id="command-logger"></a>

### command-logger 詳細資訊

將每個斜線命令以一行 JSON（時間戳記、動作、工作階段金鑰、傳送者 ID、來源）記錄至 `~/.openclaw/logs/commands.log`。

<a id="compaction-notifier"></a>

### compaction-notifier 詳細資訊

當 OpenClaw 開始和完成壓縮工作階段逐字稿時，會將簡短的狀態訊息傳送至目前的對話。這可減少聊天介面上長回合造成的困惑，因為使用者可以看到助理正在摘要內容，並會在壓縮後繼續。

<a id="boot-md"></a>

### boot-md 詳細資訊

閘道啟動時，若各個已設定代理程式範圍的已解析工作區中存在 `BOOT.md`，則執行該檔案。

## 外掛掛鉤

外掛可以透過外掛 SDK 註冊具型別的掛鉤，以進行更深入的整合：
攔截工具呼叫、修改提示詞、控制訊息流程等。
需要 `before_tool_call`、`before_agent_reply`、
`before_install` 或其他程序內生命週期掛鉤時，請使用外掛掛鉤。

由外掛管理的內部掛鉤則不同：它們會參與本頁所述的
粗粒度命令／生命週期事件系統，並在 `openclaw hooks list` 中顯示為
`plugin:<id>`。請將它們用於副作用及與掛鉤套件的相容性，而非
有序中介軟體或政策閘門。

如需完整的外掛掛鉤參考資料，請參閱[外掛掛鉤](/plugins/hooks)。

## 設定

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "session-memory": { "enabled": true },
        "command-logger": { "enabled": false }
      }
    }
  }
}
```

各掛鉤的環境值（連同程序環境）可滿足掛鉤的 `requires.env` 資格檢查，而處理常式可以從其掛鉤設定項目讀取這些值：

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "my-hook": {
          "enabled": true,
          "env": { "MY_CUSTOM_VAR": "value" }
        }
      }
    }
  }
}
```

額外掛鉤目錄：

```json
{
  "hooks": {
    "internal": {
      "load": {
        "extraDirs": ["/path/to/more/hooks"]
      }
    }
  }
}
```

<Note>
為了向後相容，仍支援舊版 `hooks.internal.handlers` 陣列設定格式，但新掛鉤應使用以探索為基礎的系統。
</Note>

## 命令列介面參考資料

```bash
# 列出所有掛鉤（加入 --eligible、--verbose 或 --json）
openclaw hooks list

# 顯示掛鉤的詳細資訊
openclaw hooks info <hook-name>

# 顯示資格摘要
openclaw hooks check

# 啟用／停用
openclaw hooks enable <hook-name>
openclaw hooks disable <hook-name>
```

## 最佳實務

- **保持處理常式快速。** 掛鉤會在命令處理期間執行。使用 `void processInBackground(event)` 以射後不理的方式執行繁重工作。
- **妥善處理錯誤。** 將有風險的操作包在 try/catch 中；不要擲出錯誤，以便其他處理常式繼續執行。
- **儘早篩選事件。** 如果事件類型／動作不相關，請立即傳回。
- **使用特定事件索引鍵。** 優先使用 `"events": ["command:new"]`，而非 `"events": ["command"]`，以減少額外負擔。

## 疑難排解

### 未探索到掛鉤

```bash
# 驗證目錄結構
ls -la ~/.openclaw/hooks/my-hook/
# 應顯示：HOOK.md、handler.ts

# 列出所有已探索到的掛鉤
openclaw hooks list
```

### 掛鉤不符合資格

```bash
openclaw hooks info my-hook
```

檢查是否缺少二進位檔（PATH）、環境變數、設定值，或作業系統相容性。

### 掛鉤未執行

1. 確認掛鉤已啟用：`openclaw hooks list`
2. 重新啟動閘道程序，讓掛鉤重新載入。
3. 檢查閘道日誌：`openclaw logs --follow | grep -i hook`

## 相關內容

- [命令列介面參考資料：掛鉤](/zh-TW/cli/hooks)
- [網路鉤子](/zh-TW/automation/cron-jobs#webhooks)
- [外掛掛鉤](/plugins/hooks) — 程序內外掛生命週期掛鉤
- [設定](/zh-TW/gateway/configuration-reference#hooks)
