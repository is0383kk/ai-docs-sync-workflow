---
read_when:
    - 你需要一份適合初學者的 OpenClaw 日誌記錄概覽
    - 你想要設定記錄層級、格式或遮蔽處理
    - 你正在進行疑難排解，需要快速找到日誌
summary: 檔案記錄、主控台輸出、命令列介面即時追蹤，以及控制介面的「記錄」分頁
title: 記錄日誌
x-i18n:
    generated_at: "2026-07-26T07:55:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c9303c4bc9c0797ca9c5775a281dce95229661b61d710425b2f7bec182b2e75
    source_path: logging.md
    workflow: 16
---

OpenClaw 有兩個主要的記錄介面：

- 由閘道寫入的**檔案記錄**（JSON 行）。
- 執行閘道之終端機中的**主控台輸出**。

控制介面的**記錄**分頁會持續追蹤閘道檔案記錄。本頁說明記錄的儲存位置、閱讀方式，以及如何設定記錄層級與格式。

## 記錄的儲存位置

依預設，閘道每天寫入一個滾動記錄檔。預設設定檔會保留歷史路徑：

`/tmp/openclaw/openclaw-YYYY-MM-DD.log`

具名設定檔會在相同目錄中使用包含設定檔識別資訊的檔名：

`/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`

檔名中的設定檔區段會轉為小寫，且僅限使用字母、數字與連字號。簡單的小寫名稱會保持易讀，因此 `--dev` 簡寫會寫入 `openclaw-dev-YYYY-MM-DD.log`。大小寫、底線與字面連字號會使用可逆的連字號逸出方式，確保不同的設定檔名稱絕不共用同一個記錄檔。直接透過環境設定的超長值會使用長度受限的雜湊後綴，以符合檔案系統的檔名長度限制。明確設定的 `logging.file` 會覆寫這些預設值。

日期採用閘道主機的本地時區。當 `/tmp/openclaw` 不安全或無法使用時（Windows 一律如此），OpenClaw 會改用作業系統暫存目錄下、使用者範圍的 `openclaw-<uid>` 目錄。含日期的記錄檔會在 24 小時後清除。

當下一次寫入會超過 `logging.maxFileBytes`（預設：100 MB）時，每個檔案都會輪替。OpenClaw 會在作用中檔案旁保留最多五個編號封存檔，例如 `openclaw-YYYY-MM-DD.1.log` 或 `openclaw-dev-YYYY-MM-DD.1.log`，並繼續寫入新的作用中記錄檔，而不會停止輸出診斷資訊。

你可以在 `~/.openclaw/openclaw.json` 中覆寫路徑：

```json
{
  "logging": {
    "file": "/path/to/openclaw.log"
  }
}
```

## 如何閱讀記錄

### 命令列介面：即時追蹤（建議）

透過 RPC 持續追蹤閘道記錄檔：

```bash
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
```

根層級設定檔選擇器會解析至閘道所使用的同一個設定檔專屬檔案，包括本機 RPC 無法使用時的命令列介面備援讀取。

選項：

| 旗標                | 預設值  | 行為                                                                              |
| ------------------- | -------- | ------------------------------------------------------------------------------------- |
| `--follow`          | 關閉      | 持續追蹤；中斷連線時使用退避機制重新連線                                   |
| `--limit <n>`       | `200`    | 每次擷取的最大行數                                                                   |
| `--max-bytes <n>`   | `250000` | 每次擷取的最大讀取位元組數                                                           |
| `--interval <ms>`   | `1000`   | 持續追蹤時的輪詢間隔                                                         |
| `--json`            | 關閉      | 以行分隔的 JSON（每行一個事件）                                              |
| `--plain`           | 關閉      | 在 TTY 工作階段中強制使用純文字                                                      |
| `--no-color`        | —        | 停用 ANSI 色彩                                                                   |
| `--utc`             | 關閉      | 以 UTC 顯示時間戳記（預設為本地時間）                                      |
| `--local-time`      | 關閉      | 接受的本地時間預設值相容拼法；除此之外沒有其他作用       |
| `--url` / `--token` | —        | 標準閘道 RPC 旗標                                                            |
| `--timeout <ms>`    | `30000`  | 閘道 RPC 逾時                                                                   |
| `--expect-final`    | 關閉      | 由代理程式支援的 RPC 最終回應等待旗標（此處透過共用用戶端層接受） |

輸出模式：

- **TTY 工作階段**：美化、彩色且結構化的記錄行。
- **非 TTY 工作階段**：純文字。

當你傳入明確的 `--url` 時，命令列介面不會自動套用設定或環境認證資訊；請自行加入 `--token`，否則呼叫會因 `gateway url override requires explicit credentials` 而失敗。

在 JSON 模式中，命令列介面會輸出帶有 `type` 標記的物件：

- `meta`：串流中繼資料（檔案、來源、來源種類、服務、游標、大小）
- `log`：已剖析的記錄項目
- `notice`：截斷／輪替提示
- `raw`：未剖析的記錄行
- `error`：閘道連線失敗（寫入 stderr）

若隱含的本機回送閘道要求配對、在連線期間關閉，或在 `logs.tail` 回應前逾時，`openclaw logs` 會自動改為讀取已設定的閘道檔案記錄。明確的 `--url` 目標不會使用此備援機制。`openclaw logs --follow` 更為嚴格：在 Linux 上，若可用，會依 PID 使用作用中的使用者 systemd 閘道日誌；否則會使用退避機制重試即時閘道，而不是持續追蹤可能已過時的並列檔案。

若無法連線至閘道，命令列介面會顯示簡短提示，要求執行：

```bash
openclaw doctor
```

### 控制介面（網頁）

控制介面的**記錄**分頁會使用 `logs.tail` 持續追蹤相同的檔案。請參閱[控制介面](/zh-TW/web/control-ui)以瞭解如何開啟。

### 僅限頻道的記錄

若要篩選頻道活動（WhatsApp／Telegram 等），請使用：

```bash
openclaw channels logs --channel whatsapp
```

`--channel` 預設為 `all`；也可使用 `--lines <n>`（預設為 200）與 `--json`。

## 記錄格式

### 檔案記錄（JSONL）

記錄檔中的每一行都是一個 JSON 物件。命令列介面與控制介面會剖析這些項目，以顯示結構化輸出（時間、層級、子系統、訊息）。

檔案記錄的 JSONL 紀錄在可用時也會包含可供機器篩選的頂層欄位：

- `hostname`：閘道主機名稱。
- `message`：用於全文搜尋的扁平化記錄訊息文字。
- `agent_id`：記錄呼叫帶有代理程式上下文時的作用中代理程式 ID。
- `session_id`：記錄呼叫帶有工作階段上下文時的作用中工作階段 ID／金鑰。
- `channel`：記錄呼叫帶有頻道上下文時的作用中頻道。

OpenClaw 會在這些欄位旁保留原始的結構化記錄引數，因此讀取編號 tslog 引數鍵的既有剖析器仍可繼續運作。

Talk、即時語音與受管理房間活動會透過同一個檔案記錄管線輸出有界限的生命週期記錄。這些紀錄會在可用時包含事件類型、模式、傳輸方式、供應商，以及大小／計時測量值，但不包含逐字稿文字、音訊承載資料、輪次 ID、通話 ID 與供應商項目 ID。

### 主控台輸出

主控台記錄會**感知 TTY**，並以易讀格式呈現：

- 子系統前綴（例如 `gateway/channels/whatsapp`）
- 層級色彩（info／warn／error）
- 選用的精簡或 JSON 模式

主控台格式由 `logging.consoleStyle` 控制。

### 閘道 WebSocket 記錄

`openclaw gateway` 也提供 RPC 流量的 WebSocket 通訊協定記錄：

- 一般模式：僅顯示值得注意的結果（錯誤、剖析錯誤、緩慢呼叫）
- `--verbose`：所有要求／回應流量
- `--ws-log auto|compact|full`：選擇詳細輸出的呈現樣式
- `--compact`：`--ws-log compact` 的別名

範例：

```bash
openclaw gateway
openclaw gateway --verbose --ws-log compact
openclaw gateway --verbose --ws-log full
```

## 設定記錄

所有記錄設定都位於 `~/.openclaw/openclaw.json` 的 `logging` 下。

```json
{
  "logging": {
    "level": "info",
    "file": "/path/to/openclaw.log",
    "consoleLevel": "info",
    "consoleStyle": "pretty",
    "redactSensitive": "tools",
    "redactPatterns": ["sk-.*"]
  }
}
```

### 記錄層級

層級：`silent`、`fatal`、`error`、`warn`、`info`、`debug`、`trace`。

- `logging.level`：**檔案記錄**（JSONL）層級（預設：`info`）。
- `logging.consoleLevel`：**主控台**詳細程度層級。

你可以透過 **`OPENCLAW_LOG_LEVEL`** 環境變數覆寫兩者（例如 `OPENCLAW_LOG_LEVEL=debug`）。環境變數的優先順序高於設定檔，因此你可以在不編輯 `openclaw.json` 的情況下，僅針對單次執行提高詳細程度。你也可以傳入全域命令列介面選項 **`--log-level <level>`**（例如 `openclaw --log-level debug gateway run`），它會針對該命令覆寫環境變數。

`--verbose` 僅影響主控台輸出與 WS 記錄的詳細程度；不會變更檔案記錄層級。

### 針對性的模型傳輸診斷

偵錯供應商呼叫時，請使用針對性的環境旗標，而不是將所有記錄提高至 `debug`：

```bash
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 openclaw gateway
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools OPENCLAW_DEBUG_SSE=events openclaw gateway
```

可用旗標：

- `OPENCLAW_DEBUG_MODEL_TRANSPORT=1`：以 `info` 層級輸出要求開始、fetch 回應、SDK 標頭、第一個串流事件、串流完成與傳輸錯誤。
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=summary`：在模型要求記錄中包含有界限的要求承載資料摘要。
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=tools`：在承載資料摘要中包含所有面向模型的工具名稱。
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`：包含經遮蔽且大小受限的 JSON 承載資料快照。僅在偵錯時使用；祕密會被遮蔽，但提示詞與訊息文字仍可能存在。
- `OPENCLAW_DEBUG_SSE=events`：輸出第一個事件與串流完成的計時資訊。
- `OPENCLAW_DEBUG_SSE=peek`：另外輸出前五個經遮蔽的 SSE 事件承載資料，且每個事件的大小皆有限制。
- `OPENCLAW_DEBUG_CODE_MODE=1`：輸出程式碼模式的模型介面診斷，包括因程式碼模式擁有工具介面而隱藏原生供應商工具時的資訊。

這些旗標會透過一般 OpenClaw 記錄輸出，因此 `openclaw logs --follow` 與控制介面的記錄分頁都會顯示它們。若未使用這些旗標，相同診斷仍可在 `debug` 層級取得。

無論 `OPENCLAW_DEBUG_MODEL_TRANSPORT` 為何，`[model-fetch]` 的開始與回應中繼資料（供應商、API、模型、狀態、延遲，以及方法、URL、逾時、Proxy 與原則等要求欄位）一律會以 `info` 層級輸出，因此無須偵錯旗標也能看見基本的模型傳輸健康狀況。

### 追蹤關聯

檔案記錄採用 JSONL。當記錄呼叫帶有有效的診斷追蹤上下文時，OpenClaw 會將追蹤欄位寫為頂層 JSON 鍵（`traceId`、`spanId`、`parentSpanId`、`traceFlags`），讓外部記錄處理器能將該行與 OTEL span 及供應商 `traceparent` 傳播建立關聯。

閘道 HTTP 要求與閘道 WebSocket 框架會建立內部要求追蹤範圍。若在該非同步範圍內輸出的記錄與診斷事件未傳入明確的追蹤上下文，就會繼承要求追蹤。代理程式執行與模型呼叫追蹤會成為作用中要求追蹤的子項，因此本機記錄、診斷快照、OTEL span 與受信任的供應商 `traceparent` 標頭可透過 `traceId` 連結，且無須記錄原始要求或模型內容。

啟用 OpenTelemetry 記錄匯出時，Talk 生命週期記錄也會流向 diagnostics-otel 記錄匯出，並使用與檔案記錄相同的有界限屬性。設定 `diagnostics.otel.logsExporter` 以選擇 OTLP、stdout JSONL 或兩種接收端。

### 模型呼叫大小與計時

模型呼叫診斷會記錄有界限的要求／回應測量值，而不擷取原始提示詞或回應內容：

- `requestPayloadBytes`：最終模型請求承載內容的 UTF-8 位元組大小
- `responseStreamBytes`：串流模型回應區塊承載內容的 UTF-8 位元組大小。高頻率文字、思考及工具呼叫差異事件僅計算增量的 `delta` 位元組，而非完整的 `partial` 快照。
- `timeToFirstByteMs`：第一個串流回應事件前的經過時間
- `durationMs`：模型呼叫總持續時間

啟用診斷匯出後，診斷快照、模型呼叫外掛鉤子及
OTEL 模型呼叫範圍／指標皆可使用這些欄位。

### 主控台樣式

`logging.consoleStyle`：

- `pretty`：易於閱讀、具色彩並附有時間戳記。
- `compact`：更緊湊的輸出（最適合長時間工作階段）。
- `json`：每行一個 JSON（供日誌處理器使用）。

### 遮蔽

OpenClaw 可在敏感權杖抵達主控台輸出、檔案日誌、
OTLP 日誌記錄、持久化工作階段逐字稿文字或 Control UI 工具
事件承載內容（工具啟動引數、部分／最終結果承載內容、衍生的
執行輸出及修補摘要）前將其遮蔽：

- 一律啟用敏感值遮蔽。
- `logging.redactPatterns`：正規表示式字串清單，用於取代日誌／逐字稿輸出的預設集合。對於 Control UI 工具承載內容，自訂模式會疊加於內建預設模式之上，因此新增模式絕不會削弱對預設模式已捕捉值的遮蔽。

檔案日誌及工作階段逐字稿仍採用 JSONL，但相符的機密值會在
該行或訊息寫入磁碟前遭到遮罩。遮蔽採盡力而為：
它適用於包含文字的訊息內容及日誌字串，而非每個
識別碼或二進位承載內容欄位。

內建預設模式涵蓋常見 API 認證資訊及付款認證資訊欄位
名稱，例如卡號、CVC/CVV、共用付款權杖及付款認證資訊；
這些名稱以 JSON 欄位、URL 參數、命令列介面旗標或指派形式出現時皆適用。

OpenClaw 也會遮蔽顯示給 UI 用戶端、支援套件、
診斷觀察器、核准提示或代理程式工具的安全邊界承載內容。自訂
`logging.redactPatterns` 可在這些介面加入專案特定模式。

## 診斷與 OpenTelemetry

診斷是模型執行及訊息流程遙測（網路鉤子、佇列、
工作階段狀態）的結構化機器可讀事件。它們**不會**
取代日誌，而是提供資料給指標、追蹤及匯出器。事件預設會在
處理程序內發出（將 `diagnostics.enabled: false` 設定為關閉即可停用）；
匯出事件則是另一項獨立設定。

有兩個相鄰介面：

- **OpenTelemetry 匯出** — 透過 OTLP/HTTP 將指標、追蹤及日誌傳送至
  任何相容 OpenTelemetry 的收集器或後端（Datadog、Grafana、
  Honeycomb、New Relic、Tempo 等）。完整設定、訊號目錄、
  指標／範圍名稱、環境變數及隱私權模型位於專屬頁面：
  [OpenTelemetry 匯出](/zh-TW/gateway/opentelemetry)。
- **診斷旗標** — 將額外日誌路由至
  `logging.file` 的特定偵錯日誌旗標，而不提高 `logging.level`。旗標不區分大小寫，
  且支援萬用字元（`telegram.*`、`*`）。請在 `diagnostics.flags` 下設定，
  或透過 `OPENCLAW_DIAGNOSTICS=...` 環境變數覆寫。完整指南：
  [診斷旗標](/zh-TW/diagnostics/flags)。

若要將 OTLP 匯出至收集器，請參閱 [OpenTelemetry 匯出](/zh-TW/gateway/opentelemetry)。

## 疑難排解提示

- **無法連線至閘道？** 請先執行 `openclaw doctor`。
- **日誌是空的？** 請確認閘道正在執行，且正寫入
  `logging.file` 中的檔案路徑。
- **需要更多詳細資訊？** 請將 `logging.level` 設定為 `debug` 或 `trace`，然後重試。

## 相關內容

- [OpenTelemetry 匯出](/zh-TW/gateway/opentelemetry) — OTLP/HTTP 匯出、指標／範圍目錄、隱私權模型
- [診斷旗標](/zh-TW/diagnostics/flags) — 特定偵錯日誌旗標
- [閘道日誌記錄內部機制](/zh-TW/gateway/logging) — WS 日誌樣式、子系統前置詞及主控台擷取
- [設定參考](/zh-TW/gateway/configuration-reference#diagnostics) — 完整的 `diagnostics.*` 欄位參考
