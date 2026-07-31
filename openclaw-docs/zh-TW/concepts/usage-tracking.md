---
read_when:
    - 你正在串接供應商用量／配額介面
    - 你需要說明使用量追蹤行為或驗證要求
summary: 用量追蹤介面與認證資訊需求
title: 使用量追蹤
x-i18n:
    generated_at: "2026-07-26T08:22:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5a1bc9aeb95cd80a48ab57a18fcd24894fdd6fb71e10e8bea8bae67a8688b78e
    source_path: concepts/usage-tracking.md
    workflow: 16
---

## 這是什麼

- 直接從各提供者的用量端點擷取提供者用量／配額。不估算提供者帳單；僅顯示提供者回報的方案名稱、配額週期、餘額、支出、預算、每日成本歷史記錄、權杖／模型歸屬或帳戶狀態摘要。
- 人類可讀的配額週期輸出會正規化為 `X% left`，即使提供者回報的是已用配額、剩餘配額或只有原始計數。沒有可重設配額週期的提供者則會顯示提供者摘要文字（例如餘額）。
- 當即時工作階段快照缺少權杖／模型資料時，工作階段層級的 `/status` 和 `session_status` 工具會回退至工作階段的文字記錄日誌。該回退機制會補齊缺少的權杖／快取計數、可復原目前執行階段的模型標籤，並在工作階段中繼資料缺失或較小時（`totalTokensFresh !== true`、零或低於從文字記錄推導出的值），優先採用較大的提示導向總數。非零的即時值一律優先於回退值。

## 顯示位置

- 聊天中的 `/status`：顯示工作階段權杖與估算成本的狀態卡（僅限 API 金鑰模型）。若可取得提供者用量，則會針對**目前的模型提供者**顯示正規化的 `X% left` 週期或提供者摘要文字。
- 聊天中的 `/usage off|tokens|full`：每則回應的用量頁尾。
- 聊天中的 `/usage cost`：從 OpenClaw 工作階段日誌彙總的本機成本摘要。
- 命令列介面：`openclaw status --usage` 會列印各提供者完整的用量／配額明細。
- 命令列介面：`openclaw models status` 會列出 OAuth／權杖驗證設定檔，並在每個具有用量週期的提供者旁顯示其摘要。
- Control UI：**Usage** 會在 OpenClaw 從工作階段推導的權杖與估算成本分析上方，顯示提供者方案與帳單卡片。Anthropic 和 OpenAI Admin API 認證資訊還會加入提供者回報的今日、7 天與 30 天支出、每日趨勢、權杖總數、熱門模型及成本類別。
- Control UI：聊天撰寫器的情境環形彈出視窗會顯示訂閱提供者的**方案用量**——各週期進度條（5 小時、每週、模型範圍）與重設時間、已知的提供者方案（例如 `Max (20x)`），以及額外用量點數。透過方案計費的工作階段會隱藏每權杖美元估算；透過 API 計費的工作階段則保留 `Est. cost` 和依類型劃分的成本明細。Claude Code 命令列介面（`claude-cli`）設定會重複使用相同的 Anthropic 訂閱用量。
- macOS 選單列：當可取得提供者用量快照時，根層級的 "Usage" 區段會顯示在 Context 下方。請參閱[選單列](/zh-TW/platforms/mac/menu-bar)。

`openclaw channels list` 不再列印提供者用量；它會改為引導使用者前往 `openclaw status` 或 `openclaw models list`。

## Anthropic 與 OpenAI 成本歷史記錄

訂閱配額與 API 帳單是不同的提供者介面：

- Anthropic 訂閱／設定認證資訊會繼續顯示 Claude 配額週期與選用的額外用量預算。設定 `ANTHROPIC_ADMIN_KEY` 或 `ANTHROPIC_ADMIN_API_KEY`，即可改為顯示組織的 Usage and Cost API 歷史記錄。系統會自動偵測以 `sk-ant-admin` 開頭的 Anthropic 提供者認證資訊。
- OpenAI ChatGPT／Codex OAuth 會繼續顯示方案、配額週期與點數餘額。設定 `OPENAI_ADMIN_KEY`，即可改為顯示組織的成本與補全用量歷史記錄；也可選擇設定 `OPENAI_PROJECT_ID`，將範圍限定為單一專案。OpenClaw 絕不會將 `OPENAI_API_KEY`、提供者設定或驗證設定檔中的推論認證資訊傳送至組織 API，因為這些金鑰可能屬於自訂端點。

管理員認證資訊具有優先權，因為它們提供實際的組織帳單。OpenClaw 不會將這些由提供者回報的總數與本機工作階段估算合併；這兩個區段刻意回答不同的問題。

## 預設用量頁尾模式

`/usage off|tokens|full` 會設定工作階段的頁尾，並在該工作階段中
記住此設定。`messages.responseUsage` 會為尚未選擇模式的工作階段提供初始模式，
因此無須每次輸入 `/usage`，頁尾也能預設開啟。

可為所有頻道設定單一模式，或使用具有 `default` 回退值的各頻道對應表：

```jsonc
{
  "messages": {
    "responseUsage": "tokens",
    // 或：{ "default": "off", "discord": "full" }
  },
}
```

接受的值：`"off"`、`"tokens"`、`"full"`，以及舊版別名 `"on"`（視為 `"tokens"`）。

### 三種不同的工作階段狀態

工作階段的 `responseUsage` 欄位有三種可表示的狀態，每種狀態具有
不同的語意：

| 狀態                | 儲存值                          | 有效模式                                                              |
| ------------------- | ------------------------------- | --------------------------------------------------------------------- |
| **未設定／繼承**    | `undefined`（不存在）    | 依序回退至 `messages.responseUsage` 設定預設值，再回退至 `off`。 |
| **明確關閉**        | `"off"`（已儲存）    | 一律關閉；非關閉的設定預設值無法重新啟用頁尾。                        |
| **明確開啟**        | `"tokens"` 或 `"full"`（已儲存） | 使用該模式，不受設定預設值影響。                                      |

### 優先順序

有效模式 = 工作階段覆寫 → 頻道設定項目 → `default` → `off`。

明確的 `/usage off` 會在工作階段中**持久儲存**為字面值 `"off"`，
與“未設定”不同。使用者明確停用頁尾後，非關閉的 `messages.responseUsage`
預設值無法再次將其開啟。

### 重設與關閉的差異

- `/usage off` 會強制關閉頁尾並持久儲存此選擇。已設定的
  非關閉預設值無法覆寫此設定。
- `/usage reset`（別名：`default`、`inherit`、`inherited`、`clear`、`unpin`）會清除工作階段
  覆寫。接著工作階段會**繼承**有效的設定預設值
  （`messages.responseUsage`）。若未設定預設值，頁尾會維持關閉。
- 完整重設工作階段（`/reset` 或 `/new`）或工作階段輪替會**保留**
  明確的用量模式偏好，讓使用者的顯示選擇能在工作階段輪替後繼續生效。
  只有 `/usage reset`（及其別名）會清除覆寫。

### 切換行為

不帶引數的 `/usage` 會依序循環：關閉 → 權杖 → 完整 → 關閉。循環的
起點是目前的**有效**模式（工作階段未設定覆寫時，會回退至設定預設值），
因此循環一律與使用者目前在頁尾看到的內容一致。

### 設定

若未設定任何設定，會維持先前的行為（頁尾保持關閉，直到使用 `/usage`）。
使用 `/usage reset` 可清除工作階段覆寫，並重新繼承已設定的預設值。

## 自訂 `/usage full` 頁尾

`/usage tokens` 一律呈現純文字的 `Usage: X in / Y out` 行（若可取得，
還會加上快取與估算成本後綴）。只有 `/usage full` 會呈現下述更豐富的
頁尾。

`/usage full` 會顯示內建的精簡頁尾，並在相關欄位可用時顯示模型、推理、快速／慢速、
情境視窗及成本。內建頁尾不需要範本檔案。

`messages.usageTemplate` 僅適用於進階自訂版面配置。其值可以是
JSON 檔案路徑（支援 `~`）或行內物件；若有效，將取代內建
頁尾。系統會監看檔案路徑，並在變更時即時重新載入。

```json
{
  "messages": {
    "usageTemplate": "~/.openclaw/usage-footer.json"
  }
}
```

缺少或空白的範本會安靜地回退至內建頁尾。無法讀取
或無效的已設定範本（JSON 錯誤，或結構中沒有可呈現的輸出
片段）也會回退至內建頁尾，並發出操作員警告。

請從內建結構開始建立自訂範本，再編輯你想
變更的部分：

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": {
    "braille": "⠐⡀⡄⡆⡇⣇⣧⣷⣿",
    "block": "░▏▎▍▌▋▊▉█",
    "shade": "░▒▓█",
    "moon": "🌑🌘🌗🌖🌕",
    "level": "▁▂▃▄▅▆▇█",
    "weather": ["🥶", "☁️", "🌥", "⛅️", "🌤", "☀️"],
    "plants": ["🪾", "🍂", "🌱", "☘️", "🍀", "🌿"],
    "moons6": ["🌑", "🌚", "🌘", "🌗", "🌖", "🌝"],
  },
  "aliases": {
    "models": {
      "claude-opus-4-6": "opus46",
      "claude-opus-4-8": "opus48",
      "claude-sonnet-4-6": "sonnet46",
      "claude-haiku-4-5": "haiku45",
      "gpt-5.5": "gpt5.5",
    },
    "reasoning": {
      "off": "🌑",
      "minimal": "🌚",
      "low": "🌘",
      "medium": "🌗",
      "high": "🌕",
      "xhigh": "🌝",
    },
  },
  "output": {
    "sep": "",
    "default": [
      { "text": "{model.provider}{identity.emoji|🤖}{model.display_name|alias:models}" },
      { "map": "model.is_fallback", "cases": { "true": "🔄" } },
      { "map": "model.is_override", "cases": { "true": "📌" } },
      { "when": "model.reasoning", "text": "{model.reasoning|alias:reasoning}" },
      { "map": "state.fast_mode", "cases": { "true": "⚡️", "false": "🐌" } },
      {
        "when": "context.max_tokens",
        "text": " | 📚[{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
      },
      { "when": "cost.turn_usd", "text": " 💰{cost.turn_usd|fixed:4}" },
    ],
    "surfaces": {
      "discord": [
        { "text": "-# -\n" },
        { "text": "-# {model.provider}{identity.emoji|🤖}{model.display_name|alias:models}" },
        { "map": "model.is_fallback", "cases": { "true": "🔄" } },
        { "map": "model.is_override", "cases": { "true": "📌" } },
        { "when": "model.reasoning", "text": "{model.reasoning|alias:reasoning}" },
        { "map": "state.fast_mode", "cases": { "true": "⚡️", "false": "🐌" } },
        {
          "when": "context.max_tokens",
          "text": " | 📚[{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
        },
        { "when": "cost.turn_usd", "text": " 💰{cost.turn_usd|fixed:4}" },
      ],
    },
  },
}
```

### 結構

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": { "<name>": "由低至高的字符" }, // 字串（每個字元 1 個字符）或陣列
  "aliases": { "<table>": { "<value>": "<label>" } },
  "output": {
    "sep": "", // 串接保留下來的片段
    "default": [/* pieces */], // 所有介面的回退值
    "surfaces": {
      "discord": [/* pieces */],
      "telegram": [/* pieces */],
    },
  },
}
```

每個介面都是一份依序排列的**片段**清單；引擎會呈現每個片段、捨棄
空白片段，並使用 `sep` 串接保留下來的片段。沒有項目的介面會使用
`output.default`。

### 合約路徑

片段會透過點號路徑從每回合合約讀取值。不存在的值視為
空白（因此 `when` 防護條件或 `|fallback` 能讓片段保持乾淨）。

| 路徑                                                                                | 意義                                                                                              |
| ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `surface`                                                                           | 頻道 ID（`discord`/`telegram`/等）                                                               |
| `agentId` / `chat_type`                                                             | 所屬代理程式 ID / 聊天介面類型                                                                  |
| `model.id` / `model.display_name` / `model.provider`                                | 模型 ID / 顯示名稱 / 提供者 ID                                                                |
| `model.actual`, `model.resolved_ref`                                                | 此輪實際使用的提供者/模型參照                                                        |
| `model.requested`                                                                   | 要求的提供者/模型參照（後援之前）                                                       |
| `model.reasoning`                                                                   | 推理強度（`off` 至 `xhigh`）                                                                       |
| `model.is_fallback` / `model.is_override`                                           | 布林值：是否使用後援 / 模型是否固定                                                                   |
| `model.override_source` / `model.auth_mode`                                         | 覆寫來源標籤 / 認證資訊模式（`oauth`, `api-key`, `token`, `mixed`, `aws-sdk`, `unknown`） |
| `state.fast_mode`                                                                   | 布林值：快速或慢速                                                                                   |
| `state.compactions`                                                                 | 工作階段的壓縮次數                                                                     |
| `context.max_tokens` / `context.used_tokens` / `context.pct_used`                   | 視窗額度 / 已占用權杖 / 已使用 0-100                                                         |
| `usage.input_tokens` / `usage.output_tokens` / `usage.total_tokens`                 | 此輪彙總                                                                                       |
| `usage.cache_read_tokens` / `usage.cache_write_tokens`                              | 此輪的快取讀取與快取寫入權杖                                                       |
| `usage.has_tokens` / `usage.has_split_tokens` / `usage.has_total_only_tokens`       | 權杖顯示防護條件                                                                                 |
| `usage.cache_hit_pct`                                                               | 快取讀取占提示詞權杖總數的比例                                                              |
| `usage.last.input_tokens` / `usage.last.output_tokens` / `usage.last.cache_hit_pct` | 僅限最後一次模型呼叫（另有 `cache_read_tokens`, `cache_write_tokens`, `total_tokens`）           |
| `cost.turn_usd` / `cost.available`                                                  | 此輪預估成本 / 是否解析出成本表                                                  |
| `timing.duration_ms`                                                                | 此輪實際經過時間                                                                             |
| `identity.name` / `identity.emoji` / `identity.avatar`                              | 代理程式身分名稱 / 表情符號 / 頭像                                                                 |
| `session.id`                                                                        | 工作階段 ID                                                                                           |

（提供者的速率限制視窗**不在**此契約中；目前沒有陣列值路徑，因此 `each` 片段沒有可供迭代的內容。）

### 動詞

由左至右將值傳入各動詞；非動詞區段為後援值。

| 動詞            | 效果                                | 範例                           |
| --------------- | ------------------------------------- | --------------------------------- |
| `num`           | 縮寫計數                         | `272000 -> 272k`                  |
| `fixed:N`       | N 位小數（`0..100`，預設為 2）      | `0.0377`                          |
| `dur`           | 將秒數轉為持續時間                   | `14820 -> 4h07m`                  |
| `pct`           | 附加 `%`                            | `96 -> 96%`                       |
| `inv`           | `100 - x`                             | 用於將已使用轉為剩餘             |
| `alias:TABLE`   | 在 `aliases` 中查找，未列出時原樣回傳 | `medium -> 🌗`                    |
| `meter:W:SCALE` | 以 W 個儲存格的字符長條表示 0-100 的值   | `[⣿⣿⠐⠐⠐]`（`meter:1` = 一個字符） |

`fixed:N` 僅接受 0 至 100 的完整十進位整數。無效的
精度引數會使該插值結果為空。

`meter:W:SCALE` 僅接受 1 至 100 的完整十進位整數寬度。將寬度留空即可使用預設值 5（`meter::braille`）；無效的
寬度會使該插值結果為空。

### 片段形式

- `{ "text": "📚 {context.max_tokens|num}" }`：常值 + 插值。
- `{ "when": "<path>", "text": "..." }`：僅在路徑值為真時呈現。
- `{ "map": "<path>", "cases": { "true": "⚡", "false": "🐌" } }`：將值對應至字符（`_default` 案例涵蓋未比對的值）。
- `{ "each": "<array-path>", "item": "{label}" }`：迭代陣列值路徑（目前契約中沒有任何路徑是陣列）。

### 範例

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": { "braille": "⠐⡀⡄⡆⡇⣇⣧⣷⣿" },
  "aliases": { "reasoning": { "medium": "🌗", "high": "🌕" } },
  "output": {
    "surfaces": {
      "discord": [
        { "text": "{model.display_name}" },
        { "when": "model.reasoning", "text": " {model.reasoning|alias:reasoning}" },
        { "map": "state.fast_mode", "cases": { "true": " ⚡", "false": " 🐌" } },
        {
          "when": "context.max_tokens",
          "text": " | 📚 [{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
        },
      ],
    },
  },
}
```

例如會呈現為 `claude-sonnet-4-6 🌗 🐌 | 📚 [⣿⣿⣿⣿⣧]272k`。

## 提供者與認證資訊

若無法解析出可用的提供者用量驗證，便會隱藏用量。OpenClaw
會自動探索已啟用、宣告
`contracts.usageProviders` 並同時實作 `resolveUsageAuth` 與
`fetchUsageSnapshot` 的提供者外掛；核心沒有獨立的提供者允許清單。靜態
契約可在不匯入每個提供者外掛的情況下限制探索範圍。每個
外掛都擁有其上游端點與回應對應。共用快照以提供者中立的方式保留方案名稱、配額視窗、餘額、支出與預算，
供命令列介面、應用程式與控制介面的使用端使用。

- **Anthropic (Claude)**：驗證設定檔中的 OAuth 權杖。若 OAuth 權杖缺少
  `user:profile` 範圍，則在已設定時，後援至 `claude.ai` 網頁工作階段（`CLAUDE_AI_SESSION_KEY`、
  `CLAUDE_WEB_SESSION_KEY`，或 `CLAUDE_WEB_COOKIE` 中的 `sessionKey=` Cookie）。
  若 Anthropic 回報依模型劃分的限制，以及已啟用額外用量的每月支出/預算，
  也會一併包含。明確指定的 Anthropic Admin API 金鑰，或
  自動偵測到的 `sk-ant-admin...` 提供者設定檔，則會改為顯示 30 天
  組織成本與 Messages API 歷程。
- **ClawRouter**：API 金鑰（`CLAWROUTER_API_KEY`）。設定後會顯示每月預算視窗
  與具類型的 USD 預算；否則顯示彙總支出，以及
  請求/權杖/成本摘要。
- **DeepSeek**：透過環境變數/設定/驗證儲存區取得 API 金鑰（`DEEPSEEK_API_KEY`）。
  顯示提供者回報的各幣別餘額。
- **GitHub Copilot**：驗證設定檔中的 OAuth 權杖。
- **Gemini CLI**：驗證設定檔中的 OAuth 權杖。
- **MiniMax**：API 金鑰或 MiniMax OAuth 驗證設定檔。OpenClaw 將
  `minimax`、`minimax-cn` 與 `minimax-portal` 視為相同的 MiniMax 配額
  介面；若存在已儲存的 MiniMax OAuth，會優先使用，否則後援至
  `MINIMAX_CODE_PLAN_KEY`、`MINIMAX_CODING_API_KEY` 或 `MINIMAX_API_KEY`。
  用量輪詢會在已設定時，從 `models.providers.minimax-portal.baseUrl`
  或 `models.providers.minimax.baseUrl` 推導 Coding Plan 主機，否則使用
  MiniMax 中國區主機。
  MiniMax 的原始 `usage_percent` / `usagePercent` 欄位表示**剩餘**
  配額，因此 OpenClaw 會在顯示前將其反轉；若存在依計數的欄位，
  則以該欄位為優先。
  - 若提供者有小時/分鐘欄位，視窗標籤會取自這些欄位，否則
    後援至 `start_time` / `end_time` 時間範圍。
  - 若 coding-plan 端點傳回 `model_remains`，OpenClaw 會優先採用
    聊天模型項目；當明確的
    `window_hours` / `window_minutes` 欄位不存在時，會從時間戳記推導視窗標籤，並在方案標籤中包含模型
    名稱。
- **OpenAI (Codex/ChatGPT plan)**：驗證設定檔中的 OAuth 權杖（若有帳戶 ID，
  則傳送 `ChatGPT-Account-Id` 標頭）。顯示 ChatGPT 方案、可重設的
  Codex 視窗，以及提供者回報的點數餘額。點數仍為提供者
  點數；OpenClaw 不會將其標示為美元。當金鑰具有 Usage
  Dashboard 存取權時，`OPENAI_ADMIN_KEY` 會加入 30 天組織成本與 completions 用量歷程。
  推論認證資訊絕不會轉送至組織 API。
- **OpenRouter**：API 金鑰或由 OAuth 支援的 API 金鑰（`OPENROUTER_API_KEY` 或驗證
  設定檔）。結合帳戶點數端點與金鑰配額端點，
  因此當認證資訊可存取這些資料時，會顯示帳戶餘額/支出、金鑰預算，
  以及每日/每週/每月用量。任一端點都能
  獨立補充快照。
- **Venice**：透過環境變數/設定/驗證儲存區取得 API 金鑰（`VENICE_API_KEY`）。顯示 USD 與
  DIEM 餘額，以及提供者回報的 DIEM epoch 配額用量。
- **Xiaomi MiMo**：兩個獨立的用量介面。隨用隨付使用 API 金鑰
  （`XIAOMI_API_KEY`）；Token Plan 使用另一把獨立金鑰（`XIAOMI_TOKEN_PLAN_API_KEY`）。
  兩者目前都不回報配額視窗。
- **z.ai**：透過環境變數/設定/驗證儲存區取得 API 金鑰（`ZAI_API_KEY` 或 `Z_AI_API_KEY`）。

## 相關內容

- [權杖用量與成本](/zh-TW/reference/token-use)
- [API 用量與成本](/zh-TW/reference/api-usage-costs)
- [提示詞快取](/zh-TW/reference/prompt-caching)
- [選單列](/zh-TW/platforms/mac/menu-bar)
