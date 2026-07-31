---
read_when:
    - 變更日誌輸出或格式
    - 偵錯命令列介面或閘道輸出
summary: 日誌記錄介面、檔案日誌、WS 日誌樣式與主控台格式設定
title: 閘道日誌記錄
x-i18n:
    generated_at: "2026-07-26T07:41:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0b11a68611032c29c31091b2411982487e7f5df3ecf4f1e3b586e7d21e543d3
    source_path: gateway/logging.md
    workflow: 16
---

# 記錄

如需面向使用者的概覽（命令列介面 + 控制介面 + 設定），請參閱 [/logging](/zh-TW/logging)。

OpenClaw 有兩種記錄介面：

- **主控台輸出** - 你在終端機／偵錯介面中看到的內容。
- **檔案記錄** - 由閘道記錄器寫入的 JSON 行。

啟動時，閘道會記錄解析後的預設代理模型，以及影響新工作階段的模式預設值：

```text
代理模型：openai/gpt-5.6-sol（思考層級=medium，快速模式=開啟）
```

`thinking` 來自預設代理、模型參數或全域代理預設值；未設定時會顯示 `medium`。`fast` 來自預設代理或模型的 `fastMode` 參數。

## 檔案式記錄器

- 預設輪替記錄檔位於 `/tmp/openclaw/` 下（每天一個檔案），日期依閘道主機的本地時區判定。預設設定檔使用 `openclaw-YYYY-MM-DD.log`；具名設定檔使用 `openclaw-<profile>-YYYY-MM-DD.log`（例如 `openclaw-dev-YYYY-MM-DD.log`）。如果該目錄不安全或無法寫入（擁有者錯誤、所有人皆可寫入或為符號連結），OpenClaw 會改用使用者範圍的 `os.tmpdir()/openclaw-<uid>` 路徑；在 Windows 上則一律使用此作業系統暫存目錄備援路徑。
- 使用中的記錄檔會在 `logging.maxFileBytes` 時輪替（預設：100 MB），最多保留五個編號封存檔（`.1` 至 `.5`），並繼續寫入新的使用中檔案。
- 透過 `~/.openclaw/openclaw.json` 設定記錄檔路徑與層級：`logging.file`、`logging.level`。
- 檔案格式為每行一個 JSON 物件。

對話、即時語音及受管理房間的程式碼路徑會使用共用檔案記錄器，寫入有界限的生命週期記錄，供營運偵錯與 OTLP 記錄匯出使用。對話文字、音訊承載內容、輪次 ID、通話 ID 及供應商項目 ID 絕不會複製到記錄項目中。

控制介面的「記錄」分頁會透過閘道追蹤此檔案（`logs.tail`）。命令列介面也會執行相同操作：

```bash
openclaw logs --follow
```

### 詳細模式與記錄層級

- **檔案記錄**僅由 `logging.level` 控制。
- `--verbose` 僅影響**主控台詳細程度**（以及 WS 記錄樣式），**不會**提高檔案記錄層級。
- 若要在檔案記錄中擷取僅限詳細模式的資訊，請將 `logging.level` 設為 `debug` 或 `trace`。
- 追蹤記錄也會包含特定熱門路徑的診斷計時摘要，例如外掛工具工廠的準備作業。請參閱 [/tools/plugin#slow-plugin-tool-setup](/zh-TW/tools/plugin#slow-plugin-tool-setup)。

## 主控台擷取

命令列介面會擷取 `console.log/info/warn/error/debug/trace`，將其寫入檔案記錄，並仍輸出至 stdout/stderr。

可獨立調整主控台詳細程度：

- `logging.consoleLevel`（預設為 `info`）
- `logging.consoleStyle`（`pretty` | `compact` | `json`；在 TTY 上預設為 `pretty`，否則為 `compact`）

## 遮罩處理

OpenClaw 會在記錄或對話輸出離開程序之前遮罩敏感權杖。此遮罩政策適用於主控台、檔案記錄、OTLP 記錄項目及工作階段對話文字輸出端，因此相符的祕密值會在 JSONL 行或訊息寫入磁碟前遭到遮罩。

- 敏感值遮罩一律啟用。
- `logging.redactPatterns`：規則運算式字串陣列（覆寫預設值）
  - 使用原始規則運算式字串（自動 `gi`），或使用 `/pattern/flags` 指定自訂旗標。
  - 相符內容會被遮罩，但保留前 6 個及後 4 個字元（值長度 >= 18 個字元）；較短的值會變為 `***`。
  - 預設規則涵蓋常見的金鑰指派、命令列介面旗標、JSON 欄位、Bearer 標頭、PEM 區塊、常用供應商權杖前綴，以及付款認證資訊欄位名稱（卡號、CVC/CVV、共用付款權杖、付款認證資訊）。

控制介面工具呼叫事件、`sessions_history` 輸出、診斷匯出、供應商錯誤、執行核准顯示及閘道 WebSocket 記錄等安全邊界一律進行遮罩。`logging.redactPatterns` 可新增部署特定的模式。

## 閘道 WebSocket 記錄

閘道會以兩種模式輸出 WebSocket 通訊協定記錄：

- **一般模式（無 `--verbose`）**：僅輸出「值得注意」的 RPC 結果，包括錯誤（`ok=false`）、緩慢呼叫（預設門檻：`>= 50ms`）及剖析錯誤。
- **詳細模式（`--verbose`）**：輸出所有 WS 要求／回應流量。

### WS 記錄樣式

`openclaw gateway` 支援各閘道獨立切換樣式：

- `--ws-log auto`（預設）：一般模式使用最佳化輸出；詳細模式使用精簡輸出。
- `--ws-log compact`：詳細模式下使用精簡輸出（配對的要求／回應）。
- `--ws-log full`：詳細模式下輸出每個完整框架。
- `--compact`：`--ws-log compact` 的別名。

```bash
# 最佳化（僅顯示錯誤／緩慢呼叫）
openclaw gateway

# 顯示所有 WS 流量（配對）
openclaw gateway --verbose --ws-log compact

# 顯示所有 WS 流量（完整中繼資料）
openclaw gateway --verbose --ws-log full
```

## 主控台格式（子系統記錄）

主控台格式器可**感知 TTY**，並輸出一致且帶有前綴的行。子系統記錄器會讓輸出保持分組且易於瀏覽：

- 每一行都有**子系統前綴**（例如 `[gateway]`、`[canvas]`、`[tailscale]`）。
- **子系統色彩**（每個子系統使用穩定色彩，由名稱雜湊產生）以及層級色彩。
- 當輸出為 TTY，或環境看似支援豐富顯示的終端機（`TERM`/`COLORTERM`/`TERM_PROGRAM`）時會**使用色彩**；並遵循 `NO_COLOR` 與 `FORCE_COLOR`。
- **縮短的子系統前綴**：移除開頭的 `gateway/`、`channels/` 或 `providers/` 區段，然後最多保留其餘區段的最後 2 個（例如 `channels/turn/kernel` 會顯示為 `turn/kernel`）。已知的頻道子系統（`telegram`、`whatsapp`、`slack` 等）一律縮減為僅顯示頻道名稱。
- **依子系統建立子記錄器**（自動前綴 + 結構化欄位 `{ subsystem }`）。
- 用於 QR／使用者體驗輸出的 **`logRaw()`**（無前綴、無格式）。
- **主控台樣式**：`pretty` | `compact` | `json`。
- **主控台記錄層級**與檔案記錄層級分開（當 `logging.level` 為 `debug`/`trace` 時，檔案仍會保留完整詳細資訊）。
- **WhatsApp 訊息本文**以 `debug` 層級記錄（使用 `--verbose` 查看）。

如此可讓檔案記錄保持穩定，同時使互動式輸出易於瀏覽。

## 相關內容

- [記錄](/zh-TW/logging)
- [OpenTelemetry 匯出](/zh-TW/gateway/opentelemetry)
- [診斷匯出](/zh-TW/gateway/diagnostics)
