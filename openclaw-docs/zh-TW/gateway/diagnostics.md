---
read_when:
    - 準備錯誤回報或支援請求
    - 偵錯閘道當機、重新啟動、記憶體壓力或過大的承載資料
    - 檢視哪些診斷資料會被記錄或遮蔽
summary: 建立可分享的閘道診斷套件，用於錯誤回報
title: 診斷資料匯出
x-i18n:
    generated_at: "2026-07-26T08:17:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97a805fed8d51de2e63e5c6a12ce03e91701d69654882cca7795c9f3553b1c55
    source_path: gateway/diagnostics.md
    workflow: 16
---

OpenClaw 可為錯誤報告建立本機診斷 `.zip`：經過清理的閘道
狀態、健康情況、日誌、設定結構，以及近期不含承載資料的穩定性事件。

在審查前，請將診斷套件視同機密資訊。承載資料與認證資訊
依設計會經過遮蔽，但套件仍會摘要本機閘道日誌與
主機層級的執行階段狀態。

## 快速開始

```bash
openclaw gateway diagnostics export
```

印出已寫入的 zip 路徑。選擇輸出路徑：

```bash
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
```

用於自動化：

```bash
openclaw gateway diagnostics export --json
```

## 聊天指令

擁有者可以在任何對話中執行 `/diagnostics [note]`，要求將本機
閘道匯出為一份可複製貼上的支援報告：

1. 傳送 `/diagnostics`，並可選擇附上簡短註記（`/diagnostics bad tool choice`）。
2. OpenClaw 會傳送前言並要求一次明確的執行核准，核准後會執行
   `openclaw gateway diagnostics export --json`。請勿透過
   全部允許規則核准診斷作業。
3. 核准後，OpenClaw 會回覆本機套件路徑、資訊清單
   摘要、隱私權注意事項，以及相關工作階段 ID。

在群組聊天中，擁有者仍可執行 `/diagnostics`，但 OpenClaw 會將
匯出結果、核准提示，以及 Codex 工作階段／執行緒明細
私下傳送給擁有者。群組只會看到一則簡短通知，告知診斷資料已
私下傳送。如果沒有可用的擁有者私人傳送路徑，該指令會以安全方式失敗，並要求
擁有者從私訊中執行。

當使用中的工作階段採用原生 OpenAI Codex 控制框架時，同一次執行
核准也涵蓋針對 OpenClaw 已知 Codex 執行緒的 OpenAI 意見回饋上傳。
此上傳與本機閘道 zip 分開，且只會
發生於 Codex 控制框架工作階段。核准提示會說明核准後
也會傳送 Codex 意見回饋，但不會列出 Codex 工作階段或執行緒 ID。核准
後，回覆會列出已傳送至 OpenAI 的執行緒所屬頻道、OpenClaw 工作階段 ID、Codex 執行緒 ID，以及
本機繼續執行指令。拒絕或
忽略核准會略過匯出、Codex 意見回饋上傳，以及
Codex ID 清單。

這可縮短 Codex 偵錯循環：在頻道中發現異常行為、
執行 `/diagnostics`、核准一次、分享報告，接著如果你想自行檢查執行緒，
可在本機執行印出的
`codex resume <thread-id>` 指令。請參閱 [Codex 控制框架](/zh-TW/plugins/codex-harness#inspect-codex-threads-locally)。

## 匯出內容

- `summary.md`：供支援人員使用的易讀概覽。
- `diagnostics.json`：設定、日誌、狀態、健康情況
  與穩定性資料的機器可讀摘要。
- `manifest.json`：匯出中繼資料與檔案清單。
- 經過清理的設定結構與非機密設定詳細資料。
- 經過清理的日誌摘要與近期已遮蔽的日誌行。
- 以最大努力取得的閘道狀態與健康情況快照。
- `stability/latest.json`：可用時，包含最新的持久化穩定性套件。

即使閘道狀況不佳，匯出仍然有用：如果狀態／健康情況
要求失敗，在可用時仍會收集本機日誌、設定結構與最新的穩定性套件。

## 隱私權模型

保留：子系統名稱、外掛 ID、供應商 ID、頻道 ID、已設定的
模式、狀態碼、持續時間、位元組數、佇列狀態、記憶體讀數、
經過清理的日誌中繼資料、已遮蔽的作業訊息、設定結構，以及
非機密功能設定。

省略或遮蔽：聊天文字、提示、指示、網路鉤子主體、工具
輸出、認證資訊、API 金鑰、權杖、Cookie、機密值、原始
要求／回應主體、帳號 ID、訊息 ID、原始工作階段 ID、
主機名稱，以及本機使用者名稱。

當日誌訊息看起來像使用者、聊天、提示或工具承載資料文字時，
匯出只會保留某則訊息已省略的事實及其位元組數。

## 穩定性記錄器

啟用診斷時，閘道預設會記錄有界且不含承載資料的穩定性事件串流。
它擷取的是作業事實，而非內容。

當事件迴圈或 CPU 看似飽和時，相同的心跳偵測也會取樣存活情況，
並發出包含事件迴圈延遲、
事件迴圈使用率、CPU 核心比例、使用中／等待中／已排入佇列的工作階段數、
目前啟動／執行階段階段（若已知）、近期階段時段，以及
有界工作標籤的 `diagnostic.liveness.warning` 事件。只有在
工作正在等待或已排入佇列，或使用中工作與持續的事件迴圈
延遲重疊時，這些事件才會成為閘道 `warn` 層級的日誌行；
否則會以 `debug` 記錄。閒置時的存活取樣仍會記錄
為診斷事件，但本身絕不會升級為警告。

啟動階段會發出包含實際經過時間與
CPU 計時的 `diagnostic.phase.completed` 事件。停滯的內嵌執行診斷會在最後一次橋接進度看似已終止
（例如原始回應
項目或回應完成事件），但閘道仍將
內嵌執行視為使用中時，標記 `terminalProgressStale=true`。

檢查即時記錄器：

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --json
```

在嚴重錯誤結束、關閉逾時或
重新啟動失敗後，檢查最新的持久化套件：

```bash
openclaw gateway stability --bundle latest
```

從最新的持久化套件建立診斷 zip：

```bash
openclaw gateway stability --bundle latest --export
```

存在事件時，持久化套件會存放於 `~/.openclaw/logs/stability/` 下。

## 實用選項

```bash
openclaw gateway diagnostics export \
  --output openclaw-diagnostics.zip \
  --log-lines 5000 \
  --log-bytes 1000000
```

| 旗標                    | 預設值                                                                       | 說明                                        |
| ----------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------- |
| `--output <path>`       | `$OPENCLAW_STATE_DIR/logs/support/openclaw-diagnostics-<timestamp>-<pid>.zip` | 寫入指定的 zip 路徑（或目錄）。       |
| `--log-lines <count>`   | `5000`                                                                        | 要納入的清理後日誌行數上限。            |
| `--log-bytes <bytes>`   | `1000000`                                                                     | 要檢查的日誌位元組數上限。                      |
| `--url <url>`           | -                                                                             | 用於狀態／健康情況快照的閘道 WebSocket URL。 |
| `--token <token>`       | -                                                                             | 用於狀態／健康情況快照的閘道權杖。         |
| `--password <password>` | -                                                                             | 用於狀態／健康情況快照的閘道密碼。      |
| `--timeout <ms>`        | `3000`                                                                        | 狀態／健康情況快照逾時。                    |
| `--no-stability-bundle` | 關閉                                                                           | 略過持久化穩定性套件查詢。            |
| `--json`                | 關閉                                                                           | 印出機器可讀的匯出中繼資料。            |

## 停用診斷

診斷預設為啟用。若要停用穩定性記錄器與
診斷事件收集：

```json5
{
  diagnostics: {
    enabled: false,
  },
}
```

停用診斷會減少錯誤報告的詳細程度；不會影響一般
閘道日誌記錄。

記憶體壓力事件會記錄 RSS、堆積、臨界值與成長資訊
（`rss_threshold`、`heap_threshold`、`rss_growth`），而不會執行
檔案系統掃描或寫入 OOM 前快照。

## 相關內容

- [健康情況檢查](/zh-TW/gateway/health)
- [閘道命令列介面](/zh-TW/cli/gateway#gateway-diagnostics-export)
- [閘道通訊協定](/zh-TW/gateway/protocol#rpc-method-families)
- [日誌記錄](/zh-TW/logging)
- [OpenTelemetry 匯出](/zh-TW/gateway/opentelemetry) - 將診斷資料串流至收集器的獨立流程
