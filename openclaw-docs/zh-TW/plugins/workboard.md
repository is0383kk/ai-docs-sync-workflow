---
read_when:
    - 你想在控制介面中使用看板式工作板
    - 你正在啟用或停用內建的 Workboard 外掛
    - 你想在不使用外部專案管理工具的情況下，追蹤規劃中的代理程式工作
summary: 供代理程式管理卡片與工作階段交接使用的選用儀表板工作看板
title: 工作看板外掛
x-i18n:
    generated_at: "2026-07-26T08:36:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ec05c990c3559015780d9cb80f3ceedd7cc79db89ccf1afd65906c8c7630331
    source_path: plugins/workboard.md
    workflow: 16
---

Workboard 外掛會在[控制介面](/zh-TW/web/control-ui)中新增選用的看板式工作區：包含適合代理程式處理的工作卡片、將工作指派給代理程式，以及返回卡片所屬任務、執行與儀表板工作階段的連結。

Workboard 刻意維持精簡：它追蹤單一 OpenClaw 閘道的本機作業工作，無意取代 GitHub Issues、Linear、Jira 或其他團隊專案管理系統。

## 啟用

Workboard 已內建，但預設停用：

1. 在控制介面中開啟 **Plugins**，或使用相對於已設定控制介面基礎路徑的 `/settings/plugins`。例如，基礎路徑為 `/openclaw` 時，會使用 `/openclaw/settings/plugins`。
2. 找到 **Workboard** 並選擇 **Enable**。由於 Workboard 已隨 OpenClaw 提供，因此不需要執行 **Install** 動作。
3. 如果介面顯示需要重新啟動，請重新啟動閘道。

外掛執行階段載入後，Workboard 分頁會出現在儀表板導覽列中。停用時，此分頁不會顯示在導覽列中。如果外掛已停用或遭 `plugins.allow`/`plugins.deny` 封鎖，直接開啟 `/workboard` 路由時，會顯示外掛無法使用的狀態，而非卡片資料。

等效的命令列介面工作流程如下：

```bash
openclaw plugins enable workboard
openclaw gateway restart
openclaw dashboard
```

## 設定

Workboard 沒有外掛專屬設定。請使用標準外掛項目啟用或停用：

```json5
{
  plugins: {
    entries: {
      workboard: {
        enabled: true,
        config: {},
      },
    },
  },
}
```

```bash
openclaw plugins disable workboard
openclaw gateway restart
```

## 卡片欄位

| 欄位        | 值                                                                                                            |
| ----------- | ------------------------------------------------------------------------------------------------------------- |
| `status`    | `triage`、`backlog`、`todo`、`scheduled`、`ready`、`running`、`review`、`blocked`、`done`                     |
| `priority`  | `low`、`normal`、`high`、`urgent`                                                                             |
| `labels`    | 自由格式字串                                                                                                |
| `agentId`   | 選用的受指派代理程式                                                                                       |
| 連結參照 | 選用的任務、執行、工作階段或來源 URL                                                                    |
| `execution` | 從卡片啟動之 Codex/Claude 執行的選用中繼資料（引擎、模式、模型、工作階段、執行 ID、狀態） |

卡片也會包含精簡的中繼資料，用於記錄嘗試、留言、連結、證明、成品、自動化設定、附件、工作程式記錄、工作程式通訊協定狀態、宣告、診斷、通知、範本 ID、封存狀態與過時工作階段偵測，以及近期事件清單（`created`、`edited`、`moved`、`linked`、`specified`、`decomposed`、`claimed`、`heartbeat`、`execution_updated`、`attempt_started`、`attempt_updated`、`comment_added`、`link_added`、`proof_added`、`artifact_added`、`attachment_added`、`diagnostic`、`notification`、`dispatch`、`orchestration`、`protocol_violation`、`archived`、`unarchived`、`stale`）。這些中繼資料可讓操作人員在不開啟連結工作階段的情況下，查看卡片如何在看板中移動；它們是本機作業情境資訊，不能取代工作階段逐字記錄或 GitHub 議題歷程。

此外掛與控制介面使用同一份 Workboard 卡片合約。因此，儀表板重新整理時會保留工作區來源與權限、宣告狀態、診斷動作及通知序號，而不是投影一份只供介面使用、內容較少的卡片副本。在兩個介面都支援未知的診斷種類、診斷嚴重性與通知種類之前，會忽略這些項目；絕不會將其改寫為另一個有效狀態。

開啟的儀表板會透過 `plugin.workboard.changed` 失效事件進行更新。每個事件僅包含儲存區紀元與修訂版；接著，介面會透過一般的 `operator.read` RPC 重新讀取標準卡片。多個修訂版會合併為一次後續讀取。拖曳、編輯或寫入卡片時，Workboard 會延後該次讀取，並在本機互動完成後繼續。每次重新連線都會執行標準重新載入。系統不會例行輪詢完整卡片，而 **Refresh** 仍可用於手動復原。

存在多個看板時，工具列會包含由持久化看板中繼資料支援的 **Board** 篩選器，而非只依據目前可見的卡片。因此，空白及已封存的看板仍可供選取。沒有明確看板 ID 的卡片屬於標準 `default` 看板。每個看板都有標準的 `/workboard/<boardId>` 頁面，可以加入書籤、分享或釘選至側邊欄。先前發布的 `/workboard?board=<boardId>` 形式仍會保留為相容性別名，並重新導向至該頁面，同時保留其他查詢參數。選擇 **All boards** 會返回 `/workboard`。

卡片儲存在外掛自己的閘道狀態中，並會隨該閘道的其餘 OpenClaw 狀態一併移動（請參閱[儲存空間](#storage)）。

## 從卡片開始工作

未連結的卡片可以直接開始工作：

- **Run Codex** / **Run Claude** 會使用明確指定的引擎啟動由任務追蹤的代理程式執行、傳送卡片提示，並將卡片標記為 `running`。Codex 執行會使用 `openai/gpt-5.6-sol`；Claude 執行則使用 `anthropic/claude-sonnet-4-6`。
- **Open Codex** / **Open Claude** 會建立已連結的儀表板工作階段，但不會傳送卡片提示或移動卡片，適用於仍附加於看板的手動工作。

自主啟動會使用閘道由任務追蹤的代理程式執行路徑（除非明確選擇 Codex/Claude，否則使用預設代理程式與模型）；接著，Workboard 會將產生的任務、執行 ID 與工作階段金鑰連結回卡片。每次連結的執行也會記錄嘗試摘要（引擎、模式、模型、執行 ID、時間戳記、狀態、累計失敗次數），讓重複失敗持續保持可見。

儀表板會從閘道任務分類帳重新整理任務狀態，並依任務 ID、執行 ID 或已連結的工作階段金鑰，將任務與卡片進行比對。已排入佇列或正在執行的任務會使卡片的生命週期保持作用中；已完成、失敗、逾時或取消的任務，則會使用與已連結工作階段相同的同步規則，將卡片移向 `review` 或 `blocked`（請參閱[工作階段生命週期同步](#session-lifecycle-sync)）。

## 代理程式工具

| 工具                                                                                                                                             | 用途                                                                                                                                                                                   |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `workboard_list`                                                                                                                                 | 列出包含認領／診斷狀態的精簡卡片；可選擇依看板篩選。                                                                                                                    |
| `workboard_read`                                                                                                                                 | 傳回一張卡片及受限的工作者情境（備註、嘗試、留言、連結、證明、成品、父項結果、受指派者近期工作、作用中診斷）。                               |
| `workboard_create`                                                                                                                               | 建立卡片，可選擇設定父項、租戶、Skills、看板、工作區中繼資料、冪等鍵、執行時間限制及重試額度。                                                             |
| `workboard_link`                                                                                                                                 | 將父卡片連結至子卡片。子卡片會維持 `todo`，直到所有父項皆達到 `done`，之後分派提升會將其移至 `ready`。                                                     |
| `workboard_claim`                                                                                                                                | 為呼叫的代理程式認領卡片；將 `backlog`/`todo`/`ready` 移至 `running`。                                                                                                        |
| `workboard_heartbeat`                                                                                                                            | 在較長的執行期間重新整理認領心跳偵測。                                                                                                                                          |
| `workboard_release`                                                                                                                              | 在完成、暫停或交接後釋放認領；可將卡片移至下一個狀態。                                                                                                |
| `workboard_complete` / `workboard_block`                                                                                                         | 用於最終摘要、證明、成品和已建立卡片資訊清單（必須參照連結回已完成卡片的卡片）或阻礙原因的結構化生命週期工具。                 |
| `workboard_attachment_add` / `workboard_attachment_read` / `workboard_attachment_delete`                                                         | 將小型卡片附件儲存在外掛 SQLite 狀態中、在卡片上建立索引，並顯示於工作者情境中。                                                                                         |
| `workboard_worker_log` / `workboard_protocol_violation`                                                                                          | 記錄工作者日誌行，並在自動化工作者停止而未呼叫 `workboard_complete`/`workboard_block` 時封鎖卡片。                                                           |
| `workboard_board_create` / `workboard_board_archive` / `workboard_board_delete`                                                                  | 管理持久化的看板中繼資料（顯示名稱、說明、封存狀態、預設工作區）。                                                                                            |
| `workboard_runs`                                                                                                                                 | 傳回卡片的持久化執行嘗試歷程。                                                                                                                                      |
| `workboard_specify`                                                                                                                              | 將粗略的分流／待辦卡片轉換為已釐清的 `todo` 卡片；在卡片上記錄規格摘要。                                                                                      |
| `workboard_decompose`                                                                                                                            | 將父協調卡片展開為相互連結的子卡片，並繼承看板／租戶中繼資料；可使用已建立卡片資訊清單完成父卡片。                                             |
| `workboard_notify_subscribe` / `workboard_notify_list` / `workboard_notify_events` / `workboard_notify_advance` / `workboard_notify_unsubscribe` | 管理通知訂閱。事件讀取可安全重播；`advance` 會移動持久游標，讓呼叫端恢復時不會遺失或重複讀取已完成／失敗／過期的卡片事件。 |
| `workboard_boards` / `workboard_stats`                                                                                                           | 檢查看板命名空間與佇列統計資料。                                                                                                                                                 |
| `workboard_promote` / `workboard_reassign` / `workboard_reclaim`                                                                                 | 復原或交接卡住的工作。                                                                                                                                                           |
| `workboard_comment` / `workboard_proof`                                                                                                          | 新增交接備註或附加證明／成品參照。                                                                                                                                    |
| `workboard_unblock`                                                                                                                              | 將遭封鎖的工作移回 `todo`。                                                                                                                                                         |
| `workboard_move`                                                                                                                                 | 將卡片移至其他狀態；已認領的卡片要求呼叫端具備代理程式認領範圍。                                                                                                      |
| `workboard_dispatch`                                                                                                                             | 在不啟動工作者的情況下觸發相依項目提升或過期認領清理；工作者啟動使用閘道或斜線命令分派。                                                        |

證明狀態是工作者回報的結果，而非獨立驗證。`passed`
項目表示工作者回報其命令或檢查已成功；需要
獨立品質關卡的使用端應檢查附加的命令、URL 或成品，並
執行自己的驗證器。`workboard_proof` 會傳回新記錄的 `proofId`。當
`workboard_complete` 回報同一證明的終止狀態時，請傳入 `proofId`，使
待處理記錄在原處獲得解析，而不會遺失其身分或時間戳記。已有
相同終止狀態的證明會原封不動地重複使用。沒有
`proofId` 的完成證明維持僅附加，因此之後的重試不能只因
命令或備註相同，就改寫較舊的歷程。

除非呼叫端持有 `workboard_claim` 傳回的認領權杖，
否則已認領的卡片會拒絕其他代理程式透過代理程式工具進行的變更。代理程式工具或
閘道 RPC 呼叫傳回的每張卡片都會將 `metadata.claim.token` 遮蔽為 `[redacted]`
（權杖本身只會由 `workboard_claim` 在頂層傳回一次），
因此儀表板操作員和其他代理程式可檢查認領狀態，卻絕不會
看到可用的權杖。復原會透過
`workboard_promote`/`workboard_reassign`/`workboard_reclaim` 進行，這些操作
不需要權杖。

## 分派

分派僅在閘道本機進行：它不會產生任意的作業系統處理程序。一般
OpenClaw 子代理程式工作階段仍負責執行。一次分派流程會：

1. 提升相依項目已就緒的卡片。
2. 在就緒卡片上記錄分派中繼資料。
3. 封鎖認領已過期或執行已逾時的卡片。
4. 將看板設定的分流卡片標示為協調候選項目。
5. 認領一小批就緒卡片，並透過
   閘道子代理程式執行階段啟動工作者執行。

工作者會取得受限的卡片情境，以及透過 Workboard 工具進行心跳偵測、
完成或封鎖卡片所需的認領權杖。

工作區路徑沿用呼叫端現有的檔案系統權限。具備
`operator.write` 的閘道用戶端可使用已設定的代理程式工作區；
`operator.admin` 用戶端可使用其他主機簽出。沙箱化代理程式工具使用
其沙箱工作區存取權，而未沙箱化且僅限工作區的工具則使用
其設定的工作區根目錄。Workboard 會在指派工作區時記錄該權限，
並在分派時再次與目前呼叫端的權限取交集，
因此持久化卡片無法擴大之後呼叫端的存取權。具有明確
主機工作區但未記錄權限的舊卡片，必須先重新儲存該工作區，
才能進行完整主機分派；沒有主機路徑的卡片會在第一次
分派時採用目前呼叫端的權限。

僅當工作區繫結分派的目錄或 Git 簽出之
儲存庫根目錄與目標代理程式工作區完全相符時，才會接受該目錄或簽出。工作樹要求
會縮限至該目錄，並持久化為目錄工作區，因此
主機不會具現化簽出或執行儲存庫設定程式碼。
目標工作者必須針對該確切工作區使用可寫入且非共用的 Docker 沙箱，
不得使用提升權限的執行、持久化的主機／節點執行覆寫，或
未分類的外掛與 MCP 工具。Workboard 會列舉其已註冊工具，
而不是信任 `workboard_*` 前綴；若熱 Docker
容器的即時掛載／設定雜湊已過期，分派便會遭拒。分派會回報
不相容的目標原則，而不是啟動限制較寬鬆的工作者。
完整主機分派可將其他本機簽出設為目標，並保留一般的受管理
工作樹設定。

工作區權限不會建立第二套卡片生命週期權限模型。
可變更 Workboard 卡片的呼叫端，能在每個介面上手動將其移經相同
狀態；唯讀工作區存取權只會阻止需要寫入權限的工作者
分派。

### 工作者選擇

每次執行**預設最多啟動 3 個工作程式**。就緒卡片依
優先順序、位置、建立時間排序。每次執行只會為每位
擁有者／代理程式啟動一張卡片，並略過看板上已有執行中或審查中工作的
擁有者。封存的卡片、有有效宣告的卡片，以及不處於 `ready`
狀態的卡片，絕不會被選中來啟動工作程式（但仍可能受派送的
資料處理部分影響：清除過期宣告、提升相依項目、逾時
清理）。

工作階段金鑰會依看板／卡片以確定性方式產生，因此重複派送會導回
同一個工作程式通道，而不會建立不相關的工作階段：

- 已指派的卡片：`agent:<agentId>:subagent:workboard-<boardId>-<cardId>`
- 未指派的卡片：`subagent:workboard-<boardId>-<cardId>`（閘道會解析
  已設定的預設代理程式）

如果卡片被宣告後仍無法啟動工作程式，Workboard 會封鎖該
卡片、清除宣告、記錄執行啟動失敗，並附加一行工作程式
日誌，可在儀表板、命令列介面 JSON、代理程式工具和卡片
診斷中看到。

### 進入點

- 儀表板派送動作
- `openclaw workboard dispatch`
- 支援命令的頻道上的 `/workboard dispatch`

閘道可用時，三者都會使用閘道子代理程式執行階段。命令列介面有一種操作員備援機制：如果閘道呼叫因
連線／不可用錯誤（或舊版
閘道的 `unknown method` 錯誤）而失敗，且沒有明確的 `--url`/`--token` 目標，也沒有適用的已設定遠端
閘道（`OPENCLAW_GATEWAY_URL` 或 `gateway.mode: remote`），命令列介面會針對本機 SQLite 狀態執行
僅資料派送；它可以提升相依項目、
清除過期宣告及封鎖逾時執行，但無法啟動工作程式。可連線閘道傳回的認證、
權限及驗證失敗不會被視為不可用；
它們會顯示為命令錯誤；明確指定 `--url`/`--token` 目標時發生的任何閘道
失敗也同樣如此。

看板中繼資料可以設定 `autoDecompose`、`autoDecomposePerDispatch`、
`defaultAssignee` 和 `orchestratorProfile`。OpenClaw 會記錄此意圖並
在工作程式情境中公開；實際的規格制定／分解仍透過
一般 Workboard 工具執行。

## 命令列介面與斜線命令

```bash
openclaw workboard list [--board <id>] [--status <status>] [--include-archived] [--json]
openclaw workboard create "修正過期卡片的生命週期" --priority high --labels bug,workboard
openclaw workboard show <card-id> [--json]
openclaw workboard move <card-id> --status <status> [--json]
openclaw workboard dispatch [--board <id>] [--json]
```

`list` 文字輸出預設會隱藏封存的卡片（`--include-archived`
可覆寫）；`--json` 一律包含封存的卡片，符合現有指令碼使用的完整卡片
合約。`show` 和 `move` 接受無歧義的 ID
前綴。`list`、`create`、`show` 和 `move` 一律直接讀寫本機外掛
狀態。只有 `dispatch` 會呼叫執行中的閘道，並採用上述
備援行為。

完整旗標、JSON 輸出、閘道備援行為、ID 前綴處理、派送選取規則及
疑難排解，請參閱 [Workboard 命令列介面](/zh-TW/cli/workboard)。

`/workboard list`、`/workboard show <card-id>`、`/workboard create <title>`、
`/workboard move <card-id> --status <status>` 和 `/workboard dispatch` 對應
命令列介面。任何獲授權的命令傳送者都可執行清單和顯示讀取操作。
建立、移動和派送需要聊天介面上的擁有者身分，或具有
`operator.write`/`operator.admin` 的閘道用戶端。操作員手動移動所採用的
宣告覆寫行為與儀表板拖放相同。其工作樹存取仍遵循
上述相同的工作區界線。

## 工作階段生命週期同步

卡片可以連結至現有的儀表板工作階段，或連結至你從卡片
開始工作時建立的工作階段。已連結的卡片會內嵌顯示工作階段生命週期：
執行中、過期、已連結但閒置、已完成、失敗或遺失。你也可以在 Sessions 分頁中使用 **Add to Workboard** 擷取
現有工作階段；卡片會連結至該工作階段，使用工作階段標籤或近期使用者提示作為標題，
並在可用時，以近期使用者提示加上最新的助理回應
預填備註。

如果已連結的工作階段遺失，卡片會保留連結以供參考，並
仍提供啟動控制項，以重新啟動至新的工作階段。如果有效的
已連結工作階段停止回報近期活動，Workboard 會將卡片標記為
`stale`，並將其儲存為中繼資料，直到生命週期將其清除。

卡片處於有效工作狀態時，Workboard 會跟隨已連結的工作階段：

| 已連結的工作階段狀態                  | 卡片狀態 |
| ------------------------------------- | ----------- |
| 使用中                                | `running`   |
| 已完成                                | `review`    |
| 失敗、遭終止、逾時或中止              | `blocked`   |

**手動審查狀態優先。** 將卡片移至 `review`、`blocked` 或 `done`
會停止該卡片的自動同步，直到你將它移回 `todo` 或 `running`。

啟動卡片時會使用一般閘道工作階段；Workboard 只儲存卡片
中繼資料和連結。對話逐字稿、模型選擇和執行
生命週期仍由一般工作階段系統管理。在即時
連結卡片上使用 **Stop** 可中止有效執行；Workboard 會將該卡片標記為 `blocked`，使其
保持可見以供後續處理。

新卡片可以從 Workboard 範本（`bugfix`、`docs`、`release`、
`pr_review`、`plugin`）開始建立。範本會預填標題、備註、標籤和優先順序；
範本 ID 會儲存為卡片中繼資料。

## 儀表板工作流程

1. 在 Control UI 中開啟 Workboard 分頁。
2. 使用標題、備註、優先順序、標籤、選用的代理程式及
   選用的已連結工作階段建立卡片；或開啟 Sessions，為現有工作階段選擇 **Add to Workboard**。
3. 在各欄之間拖曳卡片，或聚焦其精簡狀態控制項並使用
   選單或 ArrowLeft/ArrowRight。拖曳期間，來源卡片會變暗，
   可用的放置欄會顯示外框。
4. 從卡片開始工作，以建立或重複使用儀表板工作階段。
5. 代理程式工作時，從卡片開啟已連結的工作階段。
6. 讓生命週期同步將執行中的工作移至 `review`/`blocked`，接著在接受後手動
   將卡片移至 `done`。

### 工作階段看板小工具

Workboard 隨附兩個用於工作階段儀表板的原生小工具（請參閱
[儀表板](/zh-TW/web/dashboards)）。代理程式使用其 `dashboard` 工具
搭配 `content: { kind: "plugin", pluginKind, props }` 釘選這些小工具；它們會以
第一方 UI 即時呈現資料，不需要沙箱框架或能力授權：

- `workboard:card` 搭配 `props: { cardId }`，顯示單張卡片及其狀態
  控制項、優先順序和已指派的代理程式。
- `workboard:mini` 搭配選用的 `props: { boardId, limit }`，顯示各狀態的
  計數以及排名最前的就緒／執行中卡片，並連結至完整看板頁面。
  若無 `boardId`，會彙總所有看板；若有 `boardId`，則範圍限於該
  看板（未明確指定看板 ID 而建立的卡片位於 `default`）。

## 診斷

診斷依本機卡片中繼資料計算。內建檢查會標示：

| 類型                        | 條件                                                                      |
| --------------------------- | ------------------------------------------------------------------------------ |
| `stranded_ready`            | 已指派且超過 1 小時未更新的 `todo`/`backlog`/`ready` 卡片。             |
| `running_without_heartbeat` | 超過 20 分鐘沒有宣告心跳偵測或執行更新的 `running` 卡片。 |
| `blocked_too_long`          | 超過 24 小時未更新的 `blocked` 卡片。                                   |
| `repeated_failures`         | 卡片所追蹤的失敗次數達到 2 次以上。                                |
| `missing_proof`             | 沒有證明、成品或附件的 `done` 卡片。                          |
| `orphaned_session`          | 具有 `sessionKey` 但沒有 `execution` 中繼資料的 `running` 卡片。                |

## 權限

閘道 RPC 方法位於 `workboard.*` 下：

| 範圍            | 方法                                                                                                                                                                                                                                                                                                                                                                            |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operator.read`  | `cards.list`、`cards.export`、`cards.diagnostics`、附件清單／取得、通知事件讀取、`boards.list`、`cards.stats`、`cards.runs`                                                                                                                                                                                                                                       |
| `operator.write` | `cards.diagnostics.refresh`、建立／更新／移動／刪除／留言／連結／連結相依項目／證明／成品、附件新增／刪除、工作程式日誌、通訊協定違規、宣告／心跳偵測／釋放／提升／重新指派／重新宣告／完成／封鎖／解除封鎖、`cards.dispatch`、`cards.bulk`、封存、`boards.upsert`/`archive`/`delete`、`cards.specify`/`decompose`、通知訂閱／刪除／推進 |

沒有任何 RPC 方法需要 `operator.admin`。以唯讀
操作員存取權連線的瀏覽器可以檢視看板，但無法修改卡片。管理員範圍
會擴大可接受的 Workboard 主機路徑，但不會變更可用的方法。

## 儲存

Workboard 將持久資料儲存在 OpenClaw 狀態目錄下、由外掛擁有的關聯式 SQLite 資料庫中：
看板、卡片、標籤、生命週期事件、
執行嘗試、留言、相依性連結、證明、成品參照、
附件中繼資料與二進位大型物件、診斷、通知、工作程式日誌、
通訊協定狀態和訂閱，全都位於 Workboard 資料表中（而非
外掛鍵值項目）。卡片匯出會保留看板脈絡，
但不會內嵌附件二進位大型物件內容。

曾在 `.28` 版本中使用 Workboard 的安裝環境，可以執行
`openclaw doctor --fix`，將已發布的舊版外掛狀態命名空間
（`workboard.cards`、`workboard.boards`、`workboard.notify`，以及存在時的
`workboard.attachments`）遷移至關聯式資料庫。

## 疑難排解

**分頁顯示 Workboard 無法使用**

```bash
openclaw plugins inspect workboard --runtime --json
```

如果已設定 `plugins.allow`，請將 `workboard` 加入其中。如果 `plugins.deny`
包含 `workboard`，請先將其移除再啟用外掛。

**卡片無法儲存**

確認瀏覽器連線具有 `operator.write` 存取權。唯讀操作員
工作階段可以列出卡片，但無法建立、編輯、移動或刪除卡片。

**啟動卡片時未開啟預期的工作階段**

檢查卡片的代理程式 ID 和已連結工作階段，接著開啟 Sessions 或 Chat
以檢查實際執行狀態。

**派送未啟動工作程式**

確認至少有一張沒有有效宣告的 `ready` 卡片：

```bash
openclaw workboard list --status ready
```

如果命令列介面回報僅資料分派，請啟動或重新啟動閘道後重試——僅資料分派會更新本機工作看板狀態，但無法啟動子代理工作程序執行。若同一負責人或代理的另一張卡片已在執行中或等待審查，卡片也可能被略過；請先完成、封鎖或釋放該進行中的工作，再為同一負責人分派更多工作。

## 相關內容

- [控制介面](/zh-TW/web/control-ui)
- [工作看板命令列介面](/zh-TW/cli/workboard)
- [外掛](/zh-TW/tools/plugin)
- [管理外掛](/zh-TW/plugins/manage-plugins)
- [工作階段](/zh-TW/concepts/session)
