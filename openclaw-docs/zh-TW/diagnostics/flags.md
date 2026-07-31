---
read_when:
    - 你需要針對特定範圍取得偵錯日誌，而不提高全域日誌層級
    - 你需要擷取子系統特定的日誌以供支援使用
summary: 用於針對性偵錯日誌的診斷旗標
title: 診斷旗標
x-i18n:
    generated_at: "2026-07-26T07:40:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ad3bdab6ba1fd98ba58c99c93f9a12d31f57e2655cb0c1eb2de09e34b970f56c
    source_path: diagnostics/flags.md
    workflow: 16
---

診斷旗標可為單一子系統啟用額外記錄，而不會全域提高
`logging.level`。除非子系統會檢查旗標，否則旗標不會產生任何作用。

## 運作方式

- 旗標是不區分大小寫的字串，會從設定中的 `diagnostics.flags`
  加上 `OPENCLAW_DIAGNOSTICS` 環境變數覆寫值解析而來，並經過去重與轉為小寫處理。
- `name.*` 會比對 `name` 本身及 `name.` 下的任何項目（例如
  `telegram.*` 會比對 `telegram.http`）。
- `*` 或 `all` 會啟用所有旗標。
- 變更設定中的 `diagnostics.flags` 後，請重新啟動閘道；此設定
  不支援熱重新載入。

## 已知旗標

| 旗標                  | 啟用項目                                                   |
| --------------------- | --------------------------------------------------------- |
| `telegram.http`       | Telegram Bot API HTTP 錯誤記錄                       |
| `brave.http`          | Brave Search 請求／回應／快取記錄               |
| `profiler`            | 回覆階段分析器與 Codex app-server 分析器（兩者） |
| `reply.profiler`      | 僅限回覆階段分析器                                 |
| `codex.profiler`      | 僅限 Codex app-server 分析器                            |
| `health`              | 閘道健康狀態探測／帳號／繫結偵錯詳細資料        |
| `ingress.timing`      | 工作階段載入、模型選擇及模型目錄計時  |
| `plugin.load-profile` | 同步外掛模組載入計時                    |
| `timeline`            | 結構化 JSONL 時間軸成品（請見下文）            |

## 透過設定啟用

```json
{
  "diagnostics": {
    "flags": ["telegram.http"]
  }
}
```

多個旗標：

```json
{
  "diagnostics": {
    "flags": ["telegram.http", "brave.http", "gateway.*"]
  }
}
```

## 環境變數覆寫（單次）

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,brave.http
```

值會依逗號或空白分割。特殊值：

| 值                       | 效果                                   |
| --------------------------- | ---------------------------------------- |
| `0`, `false`, `off`, `none` | 停用所有旗標，並一併覆寫設定 |
| `1`, `true`, `all`, `*`     | 啟用所有旗標                        |

`OPENCLAW_DIAGNOSTICS=0` 會針對該處理程序停用環境變數與設定中的旗標，
適合用來暫時關閉設定中仍啟用的分析器旗標，而不必編輯檔案。

## 分析器旗標

分析器旗標可管控輕量級計時範圍；關閉時不會增加任何負擔。

為單次閘道執行啟用所有受分析器管控的範圍：

```bash
OPENCLAW_DIAGNOSTICS=profiler openclaw gateway run
```

僅啟用回覆分派分析器範圍：

```bash
OPENCLAW_DIAGNOSTICS=reply.profiler openclaw gateway run
```

僅啟用 Codex app-server 啟動／工具／執行緒分析器範圍：

```bash
OPENCLAW_DIAGNOSTICS=codex.profiler openclaw gateway run
```

`profiler` 會同時啟用回覆分析器與 Codex 分析器；若只要啟用其中一個，
請使用具範圍限定的旗標名稱。

或在設定中指定：

```json
{
  "diagnostics": {
    "flags": ["reply.profiler", "codex.profiler"]
  }
}
```

變更設定旗標後，請重新啟動閘道。若要停用分析器旗標，
請將它從 `diagnostics.flags` 中移除並重新啟動，或使用
`OPENCLAW_DIAGNOSTICS=0` 啟動處理程序，以在該次執行中覆寫所有診斷旗標。

## 時間軸成品

`timeline` 旗標（別名：`diagnostics.timeline`）會將結構化的啟動
與執行階段計時事件寫入 JSONL，供外部 QA 測試框架使用：

```bash
OPENCLAW_DIAGNOSTICS=timeline \
OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=/tmp/openclaw-timeline.jsonl \
openclaw gateway run
```

或在設定中啟用：

```json
{
  "diagnostics": {
    "flags": ["timeline"]
  }
}
```

輸出路徑一律來自 `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH`，即使
旗標本身是在設定中指定也是如此；路徑沒有對應的設定鍵。
當 `timeline` 僅透過設定啟用時，最早期的設定載入範圍
不會記錄，因為 OpenClaw 尚未讀取設定；後續的啟動範圍則會正常擷取。

`OPENCLAW_DIAGNOSTICS=1`、`=all` 和 `=*` 也會啟用時間軸，因為它們
會啟用所有旗標。如果只需要 JSONL 成品，而不想啟用所有其他診斷旗標，
請優先使用具範圍限定的 `timeline` 旗標。

時間軸中的事件迴圈延遲樣本除了
`timeline` 外，還需要再明確啟用一項：在啟用時間軸之外，
另請設定 `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1`（或 `on`/`true`/`yes`）。

時間軸記錄使用 `openclaw.diagnostics.v1` 封裝格式，並可能包含
處理程序 ID、階段名稱、範圍名稱、持續時間、外掛 ID、相依項目
數量、事件迴圈延遲樣本、供應商操作名稱、子處理程序結束
狀態，以及啟動錯誤名稱／訊息。請將時間軸檔案視為本機
診斷成品；在分享至你的電腦以外之前，請先檢閱內容。

## 記錄檔位置

旗標會將記錄輸出至標準診斷記錄檔。預設為：

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

具名設定檔使用 `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`；例如，
`--dev` 使用 `openclaw-dev-YYYY-MM-DD.log`。

若設定了 `logging.file`，則改用該路徑。記錄採 JSONL 格式（每行一個 JSON
物件）。資料遮蔽仍會依據 `logging.redactSensitive` 套用。
如需完整的記錄檔路徑解析、輪替與資料遮蔽模型，請參閱[記錄](/zh-TW/logging)。

## 擷取記錄

讀取作用中設定檔的最新記錄檔：

```bash
openclaw logs --plain
# 具名設定檔範例：
openclaw --profile work logs --plain
```

篩選 Telegram HTTP 診斷資訊：

```bash
openclaw logs --plain --limit 5000 | rg "telegram http error"
```

篩選 Brave Search HTTP 診斷資訊：

```bash
openclaw logs --plain --limit 5000 | rg "brave http"
```

或在重現問題時持續追蹤：

```bash
openclaw logs --follow --plain | rg "telegram http error"
```

對於遠端閘道，請改用 `openclaw logs --follow`（請參閱
[/cli/logs](/zh-TW/cli/logs)）。

## 注意事項

- 如果 `logging.level` 設定得高於 `warn`，受旗標管控的記錄可能會
  遭到抑制。預設的 `info` 即可。
- `brave.http` 會記錄 Brave Search 請求 URL／查詢參數、回應
  狀態／計時，以及快取命中／未命中／寫入事件。它不會記錄 API 金鑰
  （透過請求標頭傳送）或回應本文，但搜尋查詢可能包含敏感資訊。
- 旗標可安全地保持啟用；它們只會影響特定子系統的
  記錄量。
- 使用 [/logging](/zh-TW/logging) 變更記錄目的地、層級與資料遮蔽設定。

## 相關內容

- [閘道診斷](/zh-TW/gateway/diagnostics)
- [閘道疑難排解](/zh-TW/gateway/troubleshooting)
