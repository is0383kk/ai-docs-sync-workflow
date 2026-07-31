---
read_when:
    - 你想了解 TaskFlow 與背景任務之間的關係
    - 你在版本資訊或文件中看到 TaskFlow 或 OpenClaw 任務流程
    - 你想要檢查或管理持久的流程狀態
summary: 位於背景任務之上的 Task Flow 協調層
title: 任務流程
x-i18n:
    generated_at: "2026-07-26T08:23:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5ccc6acf58b4b44c2989e3061bff08dabce8ef385706102360c756a1286ddd1b
    source_path: automation/taskflow.md
    workflow: 16
---

Task Flow 是位於[背景任務](/zh-TW/automation/tasks)之上的協調層。流程是多步驟工作的持久記錄，具有自己的狀態、JSON 狀態資料、修訂計數器及連結的任務記錄。流程可在閘道重新啟動後繼續存在；個別任務仍是分離式工作的基本單位。

## 使用 Task Flow 的時機

| 情境                                      | 使用方式                                    |
| ----------------------------------------- | ------------------------------------------- |
| 單一背景作業                              | 一般任務                                    |
| 由外掛程式碼驅動的多步驟流水線            | Task Flow（受管理）                         |
| 分離式 ACP 或子代理啟動                   | Task Flow（鏡像，自動建立）                 |
| 一次性提醒                                | 排程作業                                    |

## 同步模式

### 受管理模式

受管理流程具有控制器：外掛程式碼透過外掛執行階段的 Task Flow API 建立流程，提供目標及必要的控制器 ID，然後明確驅動該流程。

- 每個步驟都會以流程下建立的背景任務執行；流程的擁有者金鑰與請求者來源會沿用至子任務。
- 控制器會在 `running`、`waiting` 與終止狀態之間推進流程，並將任意 JSON 步驟狀態儲存在流程記錄中。
- 每次變更都會傳入流程的預期修訂版。過時的寫入會因修訂衝突而遭拒，而不會覆寫較新的狀態。
- 一旦請求取消，系統便會拒絕新的子任務；當沒有任何子任務仍處於活動狀態時，流程會以 `cancelled` 結束。

範例：每週報告流程會 (1) 收集資料、(2) 產生報告，以及 (3) 傳送報告，每個步驟各使用一個背景任務：

```
流程：weekly-report
  步驟 1：gather-data     → 已建立任務 → 已成功
  步驟 2：generate-report → 已建立任務 → 已成功
  步驟 3：deliver         → 已建立任務 → 執行中
```

### 鏡像模式

當分離式 ACP 或子代理執行開始時（具有可傳遞完成結果的工作階段範圍任務），OpenClaw 會自動建立鏡像的單任務流程。流程記錄會鏡像其唯一的支援任務，包括狀態、目標與時間資訊，因此分離式啟動不需要控制器，也能取得穩定的流程控制代碼，用於狀態與重試介面。鏡像流程在命令列介面中會顯示同步模式 `task_mirrored`。

## 流程狀態

| 狀態      | 意義                                                                       |
| ----------- | -------------------------------------------------------------------------- |
| `queued`    | 已建立，尚未開始推進                                                       |
| `running`   | 流程正在積極推進                                                           |
| `waiting`   | 受管理流程依據等待中繼資料暫停（計時器、外部事件）                         |
| `blocked`   | 某個步驟已完成但沒有可用結果；`blockedTaskId`/摘要會指出是哪個步驟      |
| `succeeded` | 已成功完成                                                                 |
| `failed`    | 因錯誤而完成                                                               |
| `cancelled` | 已請求取消，且所有子任務均已結束                                           |
| `lost`      | 流程失去其具權威性的支援狀態                                               |

## 持久狀態與修訂追蹤

流程記錄會與任務記錄一起持久儲存在共用 SQLite 狀態資料庫（`~/.openclaw/state/openclaw.sqlite`、`flow_runs` 資料表）中，因此進度可在閘道重新啟動後繼續保留。每次寫入都會遞增流程的 `revision`；並行寫入者若傳入過時的預期修訂版，將收到衝突並必須重新讀取。WAL 的成長會由 SQLite 自動檢查點及定期的被動檢查點限制，關閉時則會執行截斷檢查點。舊版安裝中的舊式 `flows/registry.sqlite` 附屬檔案會由 `openclaw doctor` 匯入。

## 取消行為

`openclaw tasks flow cancel` 會在流程上設定持續有效的取消意圖、取消其活動中的子任務，並拒絕新的受管理子任務。當沒有任何子任務仍處於活動狀態時，流程會以 `cancelled` 結束；可能立即結束，也可能在子任務需要較長時間才能結束時，由維護掃描完成。此意圖會持久儲存，因此即使閘道在所有子任務終止前重新啟動，已取消的流程仍會維持取消狀態。

## 命令列介面命令

```bash
# 列出活動中與近期的流程
openclaw tasks flow list [--status <status>] [--json]

# 顯示特定流程的詳細資料
openclaw tasks flow show <lookup> [--json]

# 取消執行中的流程及其活動任務
openclaw tasks flow cancel <lookup>
```

| 命令                              | 說明                                                                  |
| --------------------------------- | --------------------------------------------------------------------- |
| `openclaw tasks flow list`        | 顯示所追蹤流程的同步模式、狀態、修訂版、控制器及任務數量              |
| `openclaw tasks flow show <id>`   | 依流程 ID 或擁有者金鑰檢查單一流程，包括連結的任務                    |
| `openclaw tasks flow cancel <id>` | 取消執行中的流程及其活動任務                                          |

`openclaw tasks audit`（過時或損壞的流程問題）與 `openclaw tasks maintenance`（完成卡住的取消作業，並在 7 天後清除終止流程）也會涵蓋流程。

## 可靠的排程工作流程模式

對於市場情報摘要等週期性工作流程，應將排程、協調與可靠性檢查視為不同層級：

1. 使用[排程任務](/zh-TW/automation/cron-jobs)控制執行時間。
2. 當工作流程應延續先前的內容脈絡時，使用持久的排程工作階段。
3. 使用 [Lobster](/zh-TW/tools/lobster) 處理確定性步驟、核准關卡及繼續權杖。
4. 使用 Task Flow 跨子任務、等待、重試及閘道重新啟動來追蹤多步驟執行。

排程形式範例：

```bash
openclaw cron add \
  --name "Market intelligence brief" \
  --cron "0 7 * * 1-5" \
  --tz "America/New_York" \
  --session session:market-intel \
  --message "Run the market-intel Lobster workflow. Verify source freshness before summarizing." \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

當週期性工作流程需要刻意保留歷史記錄、先前執行摘要或長期內容脈絡時，請使用 `--session session:<id>`，而非 `isolated`。當每次執行都應從全新狀態開始，且工作流程中已明確提供所有必要狀態時，請使用 `isolated`。

在工作流程內，請將可靠性檢查置於 LLM 摘要步驟之前：

```yaml
name: market-intel-brief
steps:
  - id: preflight
    command: market-intel check --json
  - id: collect
    command: market-intel collect --json
    stdin: $preflight.json
  - id: summarize
    command: market-intel summarize --json
    stdin: $collect.json
  - id: approve
    command: market-intel deliver --preview
    stdin: $summarize.json
    approval: required
  - id: deliver
    command: market-intel deliver --execute
    stdin: $summarize.json
    condition: $approve.approved
```

建議的預先檢查：

- 瀏覽器可用性與設定檔選擇，例如使用 `openclaw` 處理受管理狀態，或在需要已登入的 Chrome 工作階段時使用 `user`。請參閱[瀏覽器](/zh-TW/tools/browser)。
- 每個來源的 API 認證資訊與配額。
- 必要端點的網路連線能力。
- 代理已啟用的必要工具，例如 `lobster`、`browser` 及 `llm-task`。
- 已為排程設定失敗目的地，以便查看預先檢查失敗。請參閱[排程任務](/zh-TW/automation/cron-jobs#delivery-and-output)。

每個收集項目的建議資料來源欄位：

```json
{
  "sourceUrl": "https://example.com/report",
  "retrievedAt": "2026-04-24T12:00:00Z",
  "asOf": "2026-04-24",
  "title": "Example report",
  "content": "..."
}
```

讓工作流程在摘要前拒絕過時項目或將其標示為過時。LLM 步驟應只接收結構化 JSON，並應要求其在輸出中保留 `sourceUrl`、`retrievedAt` 及 `asOf`。當工作流程內需要經結構描述驗證的模型步驟時，請使用 [LLM 任務](/zh-TW/tools/llm-task)。

對於可供團隊或社群重複使用的工作流程，請將命令列介面、`.lobster` 檔案及任何設定說明封裝為 Skill 或外掛，並透過 [ClawHub](/zh-TW/clawhub) 發布。除非外掛 API 缺少必要的通用能力，否則請將工作流程特定的防護措施保留在該套件中。

## 流程與任務的關係

流程負責協調任務，而非取代任務。單一流程在其生命週期中可以驅動多個背景任務。使用 `openclaw tasks` 檢查個別任務記錄，並使用 `openclaw tasks flow` 檢查負責協調的流程。

## 相關內容

- [背景任務](/zh-TW/automation/tasks) - 流程所協調的分離式工作記錄
- [命令列介面：任務](/zh-TW/cli/tasks) - `openclaw tasks flow` 的命令列介面命令參考
- [自動化概覽](/zh-TW/automation) - 一覽所有自動化機制
- [排程作業](/zh-TW/automation/cron-jobs) - 可輸入流程的排程作業
