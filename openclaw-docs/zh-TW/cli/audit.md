---
read_when:
    - 你需要回答是誰執行了代理程式或工具、何時執行，以及最終如何結束
    - 你需要不含內容的傳入或傳出訊息生命週期中繼資料
    - 你需要範圍明確且可安全遮蔽敏感資訊的活動匯出功能
summary: 僅中繼資料的執行、工具與訊息生命週期稽核記錄命令列介面參考資料
title: 稽核記錄
x-i18n:
    generated_at: "2026-07-26T07:45:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: da9df6f388b0a24c3b79d755fa59d047cce99262bc6d9c890be7a83da75693a8
    source_path: cli/audit.md
    workflow: 16
---

# `openclaw audit`

查詢閘道的僅中繼資料稽核分類帳，以取得代理程式執行、工具動作及
選擇啟用的訊息生命週期記錄。

執行與工具事件的分類帳預設為啟用。設定
[`audit.enabled: false`](/zh-TW/gateway/configuration-reference#audit) 並重新啟動
閘道，即可停止所有新的事件記錄。訊息記錄則預設為停用；
將 `audit.messages` 設為 `direct` 或 `all`，並重新啟動閘道以
記錄這些事件。現有記錄在到期前仍可查詢（30 天）。

分類帳與對話逐字記錄分開：它會記錄身分、順序、來源、動作、狀態及
正規化結果代碼，但絕不儲存內容；訊息識別碼也只會以安裝環境本機的
具金鑰假名形式出現。[稽核歷程](/zh-TW/gateway/audit) 定義完整的資料模型、
隱私語意、儲存／保留界限及涵蓋範圍限制；本頁說明命令介面。

```bash
openclaw audit
openclaw audit --agent main --status failed
openclaw audit --session "agent:main:main" --after 2026-07-01T00:00:00Z
openclaw audit --run 8c69f72e-8b11-4c54-98d5-1a3dd67450c3
openclaw audit --kind tool_action --limit 50 --json
openclaw audit --kind message --direction outbound --channel telegram --json
```

## 篩選條件

- `--agent <id>`：完全相符的代理程式 ID
- `--session <key>`：完全相符的工作階段金鑰
- `--run <id>`：完全相符的執行 ID
- `--kind <kind>`：`agent_run`、`tool_action` 或 `message`
- `--status <status>`：`started`、`succeeded`、`failed`、`cancelled`、
  `timed_out`、`blocked` 或 `unknown`
- `--direction <direction>`：訊息方向，`inbound` 或 `outbound`
- `--channel <channel>`：完全相符的訊息頻道
- `--after <timestamp>` / `--before <timestamp>`：含端點的 ISO 時間戳記或
  Unix 毫秒數
- `--limit <count>`：頁面大小，範圍為 1 至 500；預設為 `100`
- `--cursor <sequence>`：接續先前以最新優先排序的查詢
- `--json`：以 JSON 列印有界限的頁面

命令列介面會查詢具版本的活動 RPC，因此一個命令即可顯示完整的
已設定分類帳。文字輸出會顯示時間、種類、方向、頻道、狀態、
代理程式、執行及動作。缺少訊息來源資訊時會顯示為 `-`；OpenClaw
不會虛構代理程式或執行 ID。工具動作也會顯示工具名稱。JSON
輸出會在還有下一頁時包含 `nextCursor`。將該值傳給
`--cursor`，即可在分頁期間不重新排序新抵達記錄的情況下繼續查詢。

即使不含訊息本文與原始訊息身分欄位，這些匯出內容仍屬敏感的操作
中繼資料。代理程式、工作階段與執行 ID、時間、頻道、結果及穩定的
HMAC 參照可用於關聯活動。請使用與其他操作員記錄相同的存取控制與
保留作法來保護這些資料。

## 記錄的事件

閘道會將受信任的生命週期串流投影為六種動作：

- `agent.run.started`
- `agent.run.finished`
- `tool.action.started`
- `tool.action.finished`
- `message.inbound.processed`
- `message.outbound.finished`

每筆傳回的記錄都有穩定的事件 ID、單調遞增的分類帳
序號、生命週期時間戳記、動作者、動作、狀態、
`schemaVersion: 1` 標記、來源序號及 `redaction: "metadata_only"`。
只有受信任來源提供代理程式／工作階段／執行來源資訊及事件特有欄位時，
這些資料才會存在。訊息記錄刻意省略
`sessionKey` 與 `sessionId`，因此 `--session` 篩選條件僅適用於執行與工具記錄。

終止的執行與工具記錄會透過封閉狀態與錯誤代碼，區分成功、失敗、取消、
逾時及政策封鎖。當上游執行階段未公開具權威性的終止結果時，
`unknown` 是明確的非成功結果。工具呼叫 ID 僅會匯出為穩定的
指紋。工具名稱必須符合精簡且面向模型的名稱
合約；其他值會變成 `unknown`。

訊息記錄會加入方向、頻道、對話種類、結果，以及選用的傳遞種類、
失敗階段、持續時間、結果數量、正規化原因代碼，和具金鑰的
帳號／對話／訊息／目標假名。目前的輸入邊界涵蓋抵達核心分派的已接受
訊息，包括核心重複處理與終止處理結果。輸出
邊界會針對抵達共用持久傳遞流程的每個原始邏輯回覆承載內容，寫入一筆
終止資料列；分塊與轉接器扇出會彙總於
`resultCount`。可重試或結果不明確的佇列傳送，只有在
確認、寄送至無法處理佇列或協調程序使結果成為終止狀態後才會記錄。
繞過這些共用邊界的外掛本機與直接傳送路徑目前尚未涵蓋；
缺少資料列並不能證明訊息從未存在。

稽核分類帳不會取代逐字記錄、任務歷程、排程執行歷程或
日誌。它提供小型的跨執行索引，讓操作員能查詢相關問題，而不必將
對話內容複製到另一個儲存區。

對輸入資料列而言，`durationMs` 會測量核心分派，而 `resultCount` 會計算
已完成的佇列工具、封鎖及回覆承載內容數量。對輸出資料列而言，
`durationMs` 包含傳遞擁有權直到其終止為止（因此也包含
佇列等待時間），而 `resultCount` 會計算已識別的實體平台
傳送次數。若有 `deliveryKind`，它會描述鉤子處理後、
轉譯後的有效承載內容；遭抑制及當機結果不明確的資料列會省略此欄位。

## 閘道 RPC

`audit.activity.list` 需要 `operator.read`，並接受相同的篩選條件。它會
傳回具名的 V1 活動事件聯集，包括執行、工具、輸入訊息
及輸出訊息記錄。

```bash
openclaw gateway call audit.activity.list --params '{"channel":"telegram","limit":50}'
```

結果為 `{ "events": AuditActivityEventV1[], "nextCursor"?: string }`。
結果會以最新優先排序，每個要求最多 500 筆記錄。

已發布的 `audit.list` RPC 對較舊的執行／工具用戶端維持不變。當
較舊的閘道無法使用 `audit.activity.list` 時，只有在舊版方法支援所有
要求的篩選條件時，命令列介面才會重試 `audit.list`。在較舊的閘道上，
`--kind message`、`--direction` 及 `--channel` 會失敗並顯示升級訊息，
而不會遭到無聲捨棄。

## 相關內容

- [稽核歷程](/zh-TW/gateway/audit)
- [閘道通訊協定](/zh-TW/gateway/protocol#audit-ledger-rpc)
- [工作階段](/zh-TW/cli/sessions)
- [任務](/zh-TW/cli/tasks)
- [排程工作](/zh-TW/automation/cron-jobs)
