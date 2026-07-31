---
read_when:
    - 你想在控制介面中查看 Dayflow 風格的每日時間軸
    - 你正在啟用或設定內建的 Logbook 外掛
    - 你想要以螢幕活動為依據的站立會議摘要或當日回顧
summary: 由定期螢幕快照建立的選用自動工作日誌
title: 日誌外掛
x-i18n:
    generated_at: "2026-07-26T08:33:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 19197e580421dfe81f82f8599578e4c68a15004813bb2b6c3de761c14f426b08
    source_path: plugins/logbook.md
    workflow: 16
---

Logbook 外掛會將螢幕活動轉換為自動工作日誌。它會定期從已配對的節點擷取螢幕快照、將其摘要為帶有時間戳記的觀察結果，並在
[控制介面](/zh-TW/web/control-ui)中建立時間軸卡片。它也能產生每日站立會議筆記，並回答有關所追蹤日期的問題。

OpenClaw 擁有的狀態會保留在閘道上的 `<state-dir>/logbook/` 下，但模型處理不一定在本機進行。取樣的螢幕截圖會傳送至已設定的視覺路由；觀察結果與時間軸文字則會傳送至預設的代理程式模型。如果螢幕內容與衍生的活動文字都必須保留在該機器上，請在兩個階段皆使用本機模型路由。

Logbook 已內建，但預設為停用。啟用此外掛即表示允許閘道擷取螢幕，因為 `captureEnabled` 的預設值為 `true`。

## 開始之前

你需要：

- 已連線且公開 `screen.snapshot` 或 `logbook.snapshot` 的節點。macOS 應用程式節點需要「螢幕錄製」權限。無頭 macOS 節點主機
  (`openclaw node host run`) 會取得由此外掛提供的 `logbook.snapshot`
  命令，該命令以系統的 `screencapture` 工具為基礎。
- 已啟用並完成驗證的內建 Codex 外掛。目前 Codex 提供 Logbook 所需的結構化影像擷取合約。使用
  `openclaw models auth login --provider openai` 登入；其他驗證途徑請參閱
  [Codex 工具環境](/zh-TW/plugins/codex-harness)。
- 可正常運作的預設代理程式模型。在視覺處理完成後，Logbook 會使用它合成卡片、站立會議筆記與當日問答。

## 快速開始

啟用 Codex 與 Logbook 外掛：

```bash
openclaw plugins enable codex
openclaw plugins enable logbook
```

設定明確的視覺模型，以確保啟動結果一致：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          visionModel: "codex/gpt-5.6-sol",
        },
      },
    },
  },
}
```

如果使用 `plugins.allow`，請同時包含 `codex` 與 `logbook`。變更外掛設定後，請重新啟動閘道，接著檢查註冊資訊並開啟儀表板：

```bash
openclaw gateway restart
openclaw plugins inspect logbook --runtime --json
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw dashboard
```

節點說明必須包含 `screen.snapshot` 或 `logbook.snapshot`。無頭節點只有在此外掛啟用後，才會公告 `logbook.snapshot`。如果缺少該命令，請參閱[節點疑難排解](/zh-TW/nodes/troubleshooting)。

只有在此外掛已啟用，且控制介面工作階段具有 `operator.write` 時，才會顯示 Logbook 分頁。狀態列應顯示 **正在擷取**，且沒有錯誤。分析時段結束時會顯示時間軸卡片；或者，你也可以在擷取到活動後選取 **立即分析**。

## 運作方式

1. **擷取**：每隔 `captureIntervalSeconds`（預設 30 秒），Logbook 會叫用所選節點的擷取命令，並儲存縮放後的 JPEG 影格。連續相同的影格會標記為閒置，並從分析中排除。
2. **觀察**：分析時段（預設 15 分鐘）經過後，此外掛會取樣最多 16 個活動影格，並將其傳送至視覺模型；模型會傳回帶有時間戳記的活動觀察結果（“VS Code：正在編輯
   store.ts、修正型別錯誤”）。超過兩分鐘的擷取中斷或本機午夜時間，也會結束目前的時段。
3. **合成**：觀察結果加上現有卡片中最近 45 分鐘的內容，會重新整理為時間軸卡片（每張涵蓋 10 至 60 分鐘），包含標題、摘要、類別、主要應用程式及任何短暫的分心活動。
4. **清除**：系統會刪除早於 `retentionDays`（預設 14）的影格。卡片、觀察結果與快取的站立會議筆記會保留。

日期界線與時間軸時鐘使用閘道的本機時區，而非瀏覽器的時區。影格與 SQLite 時間軸資料庫位於 `<state-dir>/logbook/` 下。

## 模型與資料流

Logbook 使用兩個獨立的模型路由：

| 階段             | 傳送的資料                                                  | 模型路由                                                          |
| ---------------- | --------------------------------------------------------- | ----------------------------------------------------------------- |
| 觀察             | 最多 16 個取樣的 JPEG 影格及其擷取時間                    | `visionModel`，或相容且借用的 `tools.media` Codex 項目 |
| 合成卡片         | 帶有時間戳記的觀察結果與近期時間軸卡片                    | 透過外掛 LLM 執行環境使用預設代理程式模型                         |
| 產生站立會議筆記 | 所選日期與前一天的卡片                                    | 透過外掛 LLM 執行環境使用預設代理程式模型                         |
| 詢問你的一天     | 問題、所選日期的卡片及近期觀察結果                        | 透過外掛 LLM 執行環境使用預設代理程式模型                         |

完整的 SQLite 資料庫不會傳送至任一模型。原始螢幕截圖只會傳送至觀察階段；卡片合成、站立會議筆記與問答只會接收衍生文字。

## 設定

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          captureEnabled: true,
          captureIntervalSeconds: 30,
          analysisIntervalMinutes: 15,
          nodeId: "my-mac",
          screenIndex: 0,
          maxWidth: 1440,
          visionModel: "codex/gpt-5.6-sol",
          retentionDays: 14,
        },
      },
    },
  },
}
```

所有 Logbook 設定鍵皆為選用。數值會四捨五入為整數，並限制在支援的範圍內。

| 鍵                        | 預設值  | 範圍或值                | 行為                                                                                           |
| ------------------------- | ------- | ----------------------- | ---------------------------------------------------------------------------------------------- |
| `captureEnabled`        | `true` | 布林值                  | 新快照的持久性總開關；當 `false` 時，時間軸仍可使用                                |
| `captureIntervalSeconds`        | `30` | `5`-`600` | 擷取嘗試之間的延遲                                                                             |
| `analysisIntervalMinutes`        | `15` | `3`-`120` | 目標觀察時段；中斷與午夜可能使其提前結束                                                       |
| `nodeId`        | 未設定  | 節點 ID 或顯示名稱      | 將擷取固定至一個已連線的節點；比對不區分大小寫                                                 |
| `screenIndex`        | `0` | `0`-`16` | 從零開始的顯示器索引                                                                           |
| `maxWidth`        | `1440` | `480`-`3840` | 要求的擷取尺寸上限；無頭 macOS 會將其套用至最大尺寸                                            |
| `visionModel`        | 未設定  | `provider/model`      | 明確的結構化路由；格式錯誤的參照會暫停分析，不支援的供應商會導致批次失敗                       |
| `retentionDays`        | `14` | `1`-`365` | 刪除舊影格；卡片、觀察結果與站立會議筆記會保留                                                 |

若未設定 `nodeId`，Logbook 會優先選擇公開 `screen.snapshot` 的已連線應用程式節點，若不可用則改用公開 `logbook.snapshot` 的無頭節點。在未固定節點的設定中，失敗的節點會輪替至其他符合資格的節點之後。儀表板的暫停切換開關僅適用於工作階段，且會在閘道重新啟動時重設；若要持久停止，請使用 `captureEnabled: false`。

### 視覺模型選擇

Logbook 依下列順序解析觀察模型：

1. `plugins.entries.logbook.config.visionModel`
2. `tools.media.models` 下第一個支援影像的 Codex 項目

系統會略過其他媒體供應商，因為它們目前未公開 Logbook 所需的結構化擷取合約。設定 `tools.media.image.enabled: false` 會停用借用的媒體預設值，但明確設定的 Logbook `visionModel` 仍會套用。

## 儀表板分頁

- **時間軸**：每項活動都有可展開的卡片，其中包含類別色彩、主要應用程式、分心活動標籤及快照關鍵影格。
- **一日概覽**：專注比例、類別分布、最常使用的應用程式。
- **每日站立會議筆記**：將昨天與今天的內容轉換為可直接貼上的更新。
- **詢問你的一天**：根據追蹤的時間軸回答自然語言問題（“我何時審查了閘道 PR？”）。
- **立即分析**：立即結束目前的擷取時段，而不等待分析間隔。

## 閘道方法

Logbook 會註冊下列閘道 RPC 方法：

| 方法                  | 參數                     | 範圍               | 結果                                                                       |
| --------------------- | ------------------------ | ------------------ | -------------------------------------------------------------------------- |
| `logbook.status`    | 無                       | `operator.read` | 擷取、分析、模型、節點、閘道日期及閘道時區狀態                             |
| `logbook.days`    | 無                       | `operator.read` | 具有時間軸卡片數量與卡片時間界線的日期                                     |
| `logbook.timeline`    | `{ day?: "YYYY-MM-DD" }`       | `operator.read` | 衍生卡片與當日統計資料；預設為閘道的目前日期                               |
| `logbook.frames`    | `{ startMs, endMs }`       | `operator.write` | 所要求的 Unix 紀元毫秒範圍內的影格中繼資料                                 |
| `logbook.frame`    | `{ frameId }`       | `operator.write` | 一個以 base64 編碼的原始 JPEG 影格                                         |
| `logbook.standup`    | `{ day?, refresh? }`       | `operator.write` | 某日快取或重新產生的站立會議筆記文字                                       |
| `logbook.ask`    | `{ day?, question }`       | `operator.write` | 以時間軸為依據的當日回答                                                    |
| `logbook.capture.set`    | `{ paused }`       | `operator.write` | 僅限工作階段的暫停狀態與更新後的狀態                                       |
| `logbook.analyze.now`    | 無                       | `operator.write` | 開始待處理的分析，或傳回無法開始的原因                                     |

讀取方法會傳回作業狀態或衍生文字。原始螢幕截圖像素、會產生模型費用的動作，以及執行階段異動，都需要 `operator.write`。控制介面分頁也需要 `operator.write`，因為它會公開這些動作與原始影格預覽；唯讀用戶端仍可直接呼叫衍生文字方法。

## 隱私權注意事項

- 快照可能包含螢幕上的任何內容，包括機密資訊。影格絕不會
  離開本機，唯一例外是作為取樣輸入傳送至已設定的觀察
  模型。
- 在卡片合成、產生站立會議摘要或問答期間，觀察結果、近期卡片與問題可能會透過
  預設代理模型離開本機。請將供應商的資料處理政策套用至這兩條模型
  路由。
- 需要完全在本機執行的流水線時，請為結構化觀察模型與預設代理
  模型使用本機路由。
- 影格、時間軸資料庫與暫存擷取內容皆會以
  僅擁有者可存取的檔案權限寫入。
- 將 `screen.snapshot` 新增至 `gateway.nodes.commands.deny`，即可啟用
  螢幕擷取終止開關：它會同時封鎖應用程式節點擷取與 Logbook 本身的
  `logbook.snapshot` 命令。
- 設定 `tools.media.image.enabled: false` 也會阻止 Logbook 借用
  媒體影像模型進行分析；此時只會使用外掛設定中明確指定的 `visionModel`。

## 疑難排解

### Logbook 分頁未顯示

請檢查以下三道關卡：

1. `openclaw plugins list --enabled` 包含 `logbook`。
2. 閘道已在外掛或允許清單變更後重新啟動。
3. Control UI 連線具有 `operator.write`；唯讀工作階段不會
   收到互動式分頁描述元。

如果已設定 `plugins.allow`，建議的設定必須同時包含 `logbook` 與 `codex`。

### 擷取作業回報錯誤

```bash
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw logs --follow
```

- 確認節點公開 `screen.snapshot` 或 `logbook.snapshot`。
- 在執行擷取的 Mac 上授予「螢幕錄製」權限。
- 如果已設定 `nodeId`，請確認其符合節點 ID 或顯示名稱。
- 確認 `gateway.nodes.commands.deny` 不包含
  `screen.snapshot`。

連續失敗三次後，Logbook 會暫停十個擷取週期，
然後重試。未固定節點的設定可輪替至另一個符合資格的節點。

### 擷取成功但未顯示卡片

- **缺少模型**狀態表示找不到相容的結構化視覺路由。
  請啟用 Codex 外掛並完成驗證，或設定有效且明確的
  `visionModel`。缺少模型時，已擷取的影格會維持待處理狀態，
  並可在修正設定後進行分析。
- 等待 `analysisIntervalMinutes`，或在擷取到活動後選取 **立即分析**。
- 連續且完全相同的影格會被視為閒置證據，不會進入分析
  批次。測試前請變更螢幕上的可見內容。
- 如果最新批次顯示錯誤，請修正模型或驗證問題，然後選取
  **立即分析**。為避免重複產生模型費用，失敗的批次只會在執行該明確操作時
  重試。

## 相關內容

- [管理外掛](/zh-TW/plugins/manage-plugins)
- [Codex 測試框架](/zh-TW/plugins/codex-harness)
- [媒體理解](/zh-TW/nodes/media-understanding)
- [節點](/zh-TW/nodes)
- [節點疑難排解](/zh-TW/nodes/troubleshooting)
- [Control UI](/zh-TW/web/control-ui)
