---
read_when:
    - 排程背景工作或喚醒作業
    - 將外部觸發條件（網路鉤子、Gmail）接入 OpenClaw
    - 為排程任務選擇心跳偵測或排程
sidebarTitle: Scheduled tasks
summary: 閘道排程器的排程工作、網路鉤子與 Gmail PubSub 觸發程序
title: 排程任務
x-i18n:
    generated_at: "2026-07-26T08:23:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dd889cf8e45196eda3ec7c2af930abcb2cc2bae8bad2dbdcaf3cd521a9e884b2
    source_path: automation/cron-jobs.md
    workflow: 16
---

Cron 是閘道內建的排程器。它會保存工作、在正確時間喚醒代理程式，並可將輸出傳送至聊天頻道、網路鉤子，或不傳送至任何位置。

## 快速開始

<Steps>
  <Step title="新增一次性提醒">
    ```bash
    openclaw cron create "2027-02-01T16:00:00Z" \
      --name "Reminder" \
      --session main \
      --system-event "提醒：檢查 Cron 文件草稿" \
      --wake now \
      --delete-after-run
    ```
  </Step>
  <Step title="檢查你的工作">
    ```bash
    openclaw cron list
    openclaw cron get <job-id>
    openclaw cron show <job-id>
    ```
  </Step>
  <Step title="查看執行記錄">
    ```bash
    openclaw cron runs --id <job-id>
    ```
  </Step>
</Steps>

## Cron 的運作方式

- Cron 在**閘道程序內部**執行，而非模型內部。閘道必須保持執行，排程才會觸發。
- 工作定義、執行階段狀態與執行記錄會保存於 OpenClaw 的共用 SQLite 狀態資料庫，因此重新啟動不會遺失排程。
- 每次 Cron 執行都會建立一筆[背景任務](/zh-TW/automation/tasks)記錄。
- 一次性工作（`--at`）預設會在成功後自動刪除；傳入 `--keep-after-run` 可保留工作。
- 每次執行的實際經過時間預算：設定時使用 `--timeout-seconds`。否則，隔離／卸離的代理程式回合工作會受 Cron 自身的 60 分鐘監控逾時限制，早於底層代理程式回合逾時（`agents.defaults.timeoutSeconds`，預設 48 小時）生效；命令工作預設為 10 分鐘，指令碼承載內容預設為 5 分鐘。
- 閘道啟動時，逾期的隔離代理程式回合工作會重新排程，而不會立即重播，以避免在頻道連線期間執行模型／工具啟動工作。
- 如果你透過系統 Cron 或其他外部排程器驅動 `openclaw agent`，即使命令列介面已處理 `SIGTERM`/`SIGINT`，仍應使用強制終止升級機制包裝它。由閘道支援的執行會要求閘道中止已接受的執行；`--local` 執行也會收到相同的中止訊號。使用 GNU `timeout` 時，應優先使用 `timeout -k 60 600 openclaw agent ...`，而非單獨使用 `timeout 600 ...` — 如果程序無法及時完成收尾，`-k` 值便是最後保障。對於 systemd 單元，請使用 `SIGTERM` 停止訊號，並在最終強制終止前保留寬限期（`TimeoutStopSec`）。若原始閘道執行仍在進行時重複使用 `--run-id`，系統會將重複項目回報為執行中，而不會啟動第二次執行。

<AccordionGroup>
  <Accordion title="隔離執行強化">
    - 隔離執行完成時，會盡力關閉其 `cron:<jobId>` 工作階段所追蹤的瀏覽器分頁／程序，並透過主要工作階段與自訂工作階段執行所使用的相同共用拆卸路徑，釋放為該工作建立的任何隨附 MCP 執行階段執行個體。清理失敗會被忽略，確保 Cron 結果仍具優先權。
    - 具有有限 Cron 自我清理授權的隔離執行，可讀取排程器狀態、僅包含自身工作的自我篩選清單，以及該工作的執行記錄，且只能移除自身工作。
    - 隔離執行會防範過時的確認回覆：如果第一個結果只是一則暫時狀態更新（`on it`、`pulling everything together` 及類似提示），且沒有任何後代子代理程式仍負責最終答案，OpenClaw 會再提示一次，以取得實際結果後再傳送。
    - 系統會辨識結構化的執行拒絕中繼資料（包括巢狀錯誤以 `SYSTEM_RUN_DENIED` 或 `INVALID_REQUEST` 開頭的節點主機 `UNAVAILABLE` 包裝器），因此受阻的命令不會被回報為成功執行，同時也不會將一般助理文字誤認為拒絕。
    - 即使沒有回覆承載內容，執行層級的代理程式失敗也會計為工作錯誤，因此模型／提供者失敗會增加錯誤計數並觸發失敗通知，而不會將工作標記為成功。
    - 工作達到 `timeoutSeconds` 時，Cron 會中止該次執行，並提供短暫的清理時間。如果仍未完成收尾，在 Cron 記錄逾時前，由閘道擁有的清理程序會強制清除該次執行的工作階段擁有權，避免佇列中的聊天工作卡在過時的處理中工作階段之後。
    - 設定／啟動停滯會套用階段專屬逾時（例如 `cron: isolated agent setup timed out before runner start` 或 `cron: isolated agent run stalled before execution start (last phase: context-engine)`）。這些監控逾時涵蓋嵌入式與命令列介面支援的提供者，即使其外部命令列介面程序尚未啟動亦然；其上限不受較長的 `timeoutSeconds` 值影響，因此冷啟動／驗證／情境失敗能快速浮現。

  </Accordion>
  <Accordion title="任務核對">
    Cron 任務核對首先以執行階段為準，其次才以永久記錄為依據：只要 Cron 執行階段仍將該工作追蹤為執行中，即使舊的子工作階段資料列仍然存在，作用中的 Cron 任務也會維持有效。執行階段不再擁有該工作且 5 分鐘寬限期屆滿後，維護檢查會從持久化執行記錄與工作狀態中尋找相符的 `cron:<jobId>:<startedAt>` 執行。其中的終止結果會完成任務帳本；否則，由閘道擁有的維護程序可將任務標記為 `lost`。離線命令列介面稽核可從永久記錄復原，但其自身程序內的作用中工作集合為空，並不能證明由閘道擁有的執行已消失。
  </Accordion>
</AccordionGroup>

## 排程類型

| 種類      | 命令列介面旗標           | 說明                                                                                              |
| --------- | ------------------ | -------------------------------------------------------------------------------------------------------- |
| `at`      | `--at`             | 一次性時間戳記（ISO 8601 或如 `20m` 的相對時間）                                                     |
| `every`   | `--every`          | 固定間隔（`10m`、`1h`、`1d`）                                                                       |
| `cron`    | `--cron`           | 5 欄位或 6 欄位 Cron 運算式，可選用 `--tz`                                                  |
| `on-exit` | `--on-exit`        | 監看的命令結束時觸發一次（事件觸發器；回合拆卸後仍會保留；可選用 `--on-exit-cwd`） |
| `stream`  | `--stream-command` | 由受監督的長時間執行命令所產生的批次行觸發                                      |

沒有時區的時間戳記會視為 UTC。加入 `--tz America/New_York`，即可在該 IANA 時區中解讀不含時差的 `--at` 日期時間，或評估 Cron 運算式。未含 `--tz` 的 Cron 運算式會使用閘道主機時區。`--tz` 不適用於 `--every` 或 `--on-exit`。

每小時整點循環的運算式（分鐘欄位為 `0`，小時欄位使用萬用字元）會自動錯開最多 5 分鐘，以降低負載尖峰。使用 `--exact` 可強制精準計時，或使用 `--stagger 30s` 設定明確時間範圍（僅限 Cron 排程）。

### 心跳偵測任務移轉

舊版心跳偵測暫存支援結構化的 `tasks:` 區塊。升級後執行 `openclaw doctor --fix`，將每個項目轉換為一般且可編輯的主要工作階段 Cron 工作。Doctor 會保留間隔與先前的上次執行時間，先建立工作再移除區塊，並在重新執行時安全地收斂相同的宣告鍵。

這些已移轉的工作帶有公用 `systemEvent` 承載內容，因此 `openclaw cron list`、`get`、`edit`、`remove` 與 Cron 工具可如同其他工作一樣管理它們。其執行使用受防護的心跳偵測任務喚醒機制：作用時段、最小間隔、防洪控制與忙碌重試仍然適用，而 Cron 則負責各任務的獨立節奏。在同一合併時間範圍到期的工作可共用一次心跳偵測回合。若排定的執行發生於心跳偵測作用時段之外，系統會略過該次執行，並在工作的下一次執行時間重試。

心跳偵測暫存現在僅用於監控文字。執行階段心跳偵測不會將 `tasks:` 文字解析為排程；請使用 Cron 建立新的循環工作。

### 串流來源

串流排程會在閘道下持續執行由操作人員編寫的 argv 命令，並根據其 stdout 與 stderr 行觸發工作。串流排程由事件驅動，絕不會因時間到期而執行，且必須使用 `cron.triggers.enabled: true`，因為長時間執行命令與觸發器指令碼屬於相同的無人值守信任等級。停用或移除工作會停止該程序；閘道關閉時會等待程序樹完成拆卸。快速失敗時會使用 Cron 內建的錯誤退避機制重新啟動。若連續五次執行均短於 60 秒，工作會維持錯誤狀態並使用一般失敗警示路徑；請手動重新啟用工作以清除重新啟動上限。

```bash
openclaw cron add \
  --name "Build event stream" \
  --stream-command '["node","scripts/build-events.mjs"]' \
  --stream-mode match \
  --stream-match '^(failed|recovered):' \
  --stream-batch-ms 250 \
  --session isolated \
  --message "調查這些建置事件。"
```

`mode: "line"`（預設值）接受每一行。`mode: "match"` 僅接受符合已編譯 `match` 規則運算式的行。批次會在靜默 `batchMs` 後關閉（預設 250 毫秒，限制在 50–5000），或在達到 `maxBatchBytes` 時關閉（預設 16384，限制在 1024–65536）。達到位元組上限時，批次會以 `[truncated]` 結尾。比對模式一律使用完整文字評估完整行，即使超過 `maxBatchBytes` 亦然（只有傳送的批次會被截斷）；在有界原始輸入上限處截斷的行只是一段前綴，因此會視為不相符，避免以結尾為錨點的模式在截斷處觸發。批次會附加至系統事件文字或代理程式回合訊息。串流排程會拒絕命令承載內容，因為來源命令與承載內容命令的程序擁有權會產生歧義。

每個工作只會保留一次承載內容觸發與一個有界的待處理批次。承載內容執行期間，或內建的 30 秒觸發間隔尚未經過時抵達的行，會合併至該待處理批次，而不會建立無界佇列。單一序列化擁有者會在 `streamDroppedBatches` 中記錄閘門捨棄、承載內容錯誤，以及未執行狀態下的分派；有界合併會遞增 `streamCoalescedBatches`。失敗的承載內容不會重試，因為其操作可能不是等冪的。邏輯來源身分在受監督的子程序重新啟動時會保持穩定，但來源被停用、移除或取代時則會輪替，因此即使進行 A 到 B 再到 A 的編輯，已淘汰來源的佇列批次也無法觸發。停止完成後，舊子程序的延遲回呼不會產生作用。V1 不包含原生 WebSocket 來源；可透過 argv 命令（例如 `websocat wss://example.invalid/events`）進行橋接。

當串流工作也具有 `trigger.script` 時，閘門會對每個已關閉批次執行一次。目前批次會以深度凍結的 `trigger.streamBatch` 字串形式，與 `trigger.state` 一同提供。`fire: false` 會在保存閘門狀態後捨棄該批次。`fire: true` 會保留既有的觸發訊息語意，然後將批次附加至產生的承載內容。串流工作也可改用不含條件閘門的指令碼承載內容；該指令碼會透過相同的 `trigger.streamBatch` 值接收批次。系統會拒絕將指令碼承載內容與條件閘門結合，因為兩者都會擁有持久化的 `trigger.state` 槽位。

### 動態節奏（步調控制）

循環工作可將 `pacing.min` 和／或 `pacing.max` 設為 `15m` 或 `4h` 之類的持續時間字串；至少需要設定一個界限。將 `--pacing-min` 和 `--pacing-max` 與 `cron add|edit` 搭配使用（`--clear-pacing` 會移除兩個界限）。

在隔離執行期間，具節奏控制的工作可使用 `action: "next_check"` 和 `in: "30m"` 呼叫 `cron` 工具。此提議僅套用至目前執行中的工作，並從成功完成執行時起計算。OpenClaw 會以靜默方式將其限制在設定的界限內。

未提出提議的節奏控制不會變更正常排程。失敗、逾時及跳過的執行會捨棄提議，因此現有的重試與錯誤退避行為具有優先權。手動強制執行週期性工作屬於頻帶外操作，並會保留其待處理的自然或節奏控制時段。對於條件觸發的工作，即使提議要求更早檢查，內建的最小間隔仍是下限。

### 月中日期與星期幾採用 OR 邏輯

Cron 運算式由 [croner](https://github.com/Hexagon/croner) 剖析。當月中日期與星期幾欄位都不是萬用字元時，只要**任一**欄位相符，croner 就會判定相符，而非要求兩者皆相符。這是標準的 Vixie cron 行為。

```bash
# 預期：「僅當 15 日是星期一時，在上午 9 點執行」
# 實際：「每月 15 日上午 9 點，以及每個星期一上午 9 點」
0 9 15 * 1
```

這會使工作每月觸發約 5-6 次，而非每月 0-1 次。若要同時要求兩個條件，請使用 croner 的 `+` 星期幾修飾符（`0 9 15 * +1`），或依其中一個欄位排程，並在工作的提示詞或命令中檢查另一個條件。

## 事件觸發條件（條件監看器）

事件觸發條件會將無介面條件指令碼加入 `every`、`cron` 或 `stream` 排程。時間排程會在到期時評估該指令碼；串流排程則會針對每個已關閉批次進行評估。只有當指令碼傳回 `fire: true` 時，排程才會執行正常承載內容：

```json5
{
  schedule: { kind: "every", everyMs: 30000 },
  trigger: {
    // 僅在觀察到的狀態與上次評估不同時觸發。
    script: "const res = await tools.call('exec', { command: 'gh pr checks 123 --json state -q \\'.[].state\\' | sort -u' }); const status = String(res?.result?.details?.aggregated ?? '').trim(); json({ fire: status !== trigger.state?.status, message: `PR 123 CI: ${trigger.state?.status ?? 'unknown'} -> ${status}`, state: { status } });",
    once: false,
  },
  payload: { kind: "agentTurn", message: "調查 CI 狀態變更。" },
}
```

指令碼必須傳回 `{ fire, message?, state? }`。先前的 JSON 狀態可透過深度凍結的 `trigger.state` 取得；串流閘門也會透過 `trigger.streamBatch` 接收目前批次。傳回新的 `state` 值即可將其持久保存。狀態上限為 16 KB。當觸發結果包含 `message` 時，排程會先將其附加到系統事件文字或代理程式回合訊息，再執行工作。`once: true` 會在第一次成功執行已觸發的承載內容後停用工作。

`fire: false` 會持久保存評估狀態與計數器，然後重新排程，而不建立執行歷程記錄。如果已觸發的承載內容執行失敗，傳回的 `state` **不會**持久保存——下一次評估會看到先前的狀態，並可再次觸發，因此請將指令碼撰寫為唯讀檢查，並將動作保留在承載內容中。觸發條件排程的內建最小間隔為 30 秒。每次評估有 30 秒的實際經過時間預算，最多可呼叫工具 5 次。

監看器應以**可採取行動的狀態**為核心設計，而不應只監看成功：如果監看器在檢查失敗或逾時時沉默，看起來會像是正常運作，實際上卻已故障。將觀察結果與 `trigger.state` 比較，並傳回最新狀態以進行去重；不要依賴模型或程序記憶體。觸發時，請確保 `message` 自成一體，因為它會成為該次已觸發執行的完整事件內容。

<Warning>
啟用 `cron.triggers.enabled` 會允許條件觸發指令碼與 `script` 承載內容，在無介面模式下使用所屬代理程式的**完整工具政策，包括 `exec`** 執行。請將此視為以該代理程式權限進行的無人值守程式碼執行；除非所有獲准建立排程工作的代理程式都同樣可信，否則請保持停用。
</Warning>

從本機指令碼檔案建立監看器（`-` 會從標準輸入讀取指令碼）：

```bash
openclaw cron add \
  --name "PR CI watcher" \
  --every 30s \
  --trigger-script ./watch-pr-ci.js \
  --message "Respond to the CI status change" \
  --session isolated
```

## 承載內容

每個工作恰好包含一種承載內容類型，並透過旗標選擇：

| 承載內容      | 旗標                                           | 執行方式                                                   |
| ------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| 系統事件      | `--system-event <text>`                        | 加入主工作階段佇列，本身不呼叫模型                         |
| 代理程式訊息  | `--message <text>`                             | 由模型支援的代理程式回合                                   |
| 命令          | `--command <shell>` 或 `--command-argv <json>` | 在閘道主機上執行殼層／程序，不呼叫模型                     |
| 指令碼        | `--script <file\|->`                           | 使用所屬代理程式工具的無介面程式碼模式指令碼               |

另有一種承載內容類型 `heartbeat` 由系統擁有：閘道會為每個已啟用心跳偵測的代理程式，收斂至一個心跳偵測監控工作（請參閱[心跳偵測](/zh-TW/gateway/heartbeat)）。它會出現在 `cron list --all` 中，但無法透過命令列介面或 API 建立或編輯。啟動、重新載入設定或執行 `openclaw doctor --fix` 時，心跳偵測設定會直接寫入持久保存的監控排程。停用排程時，監控工作不會計時，也不會執行備援心跳偵測計時器。

### 代理程式回合選項

<ParamField path="--message" type="string" required>
  提示詞文字（隔離／目前／自訂工作階段工作必填）。
</ParamField>
<ParamField path="--model" type="string">
  模型覆寫；必須解析為允許的模型，否則執行會因驗證錯誤而失敗。
</ParamField>
<ParamField path="--fallbacks" type="string">
  各工作專用的備援模型清單，例如 `--fallbacks openai/gpt-5.6-sol,openrouter/meta-llama/llama-3.3-70b-instruct:free`。傳入 `--fallbacks ""` 可進行不使用備援模型的嚴格執行。
</ParamField>
<ParamField path="--clear-fallbacks" type="boolean">
  在 `cron edit` 上，移除各工作專用的備援模型覆寫，使工作遵循已設定的備援優先順序。無法與 `--fallbacks` 合併使用。
</ParamField>
<ParamField path="--clear-model" type="boolean">
  在 `cron edit` 上，移除各工作專用的模型覆寫，使工作遵循正常的排程模型優先順序（已儲存的排程工作階段覆寫，否則使用代理程式／預設模型）。無法與 `--model` 合併使用。
</ParamField>
<ParamField path="--thinking" type="string">
  思考層級覆寫（`off|minimal|low|medium|high|xhigh|adaptive|max|ultra`）。可用層級仍取決於所選模型與代理程式執行階段。
</ParamField>
<ParamField path="--clear-thinking" type="boolean">
  在 `cron edit` 上，移除各工作專用的思考層級覆寫。無法與 `--thinking` 合併使用。
</ParamField>
<ParamField path="--light-context" type="boolean">
  略過工作區啟動檔案注入。
</ParamField>
<ParamField path="--tools" type="string">
  限制工作可使用的工具，例如 `--tools exec,read`。
</ParamField>

可執行工具的新工作一律會儲存明確的工具政策。由代理程式建立的工作
會受限於建立該工作的回合所能使用的工具，且代理程式無法擴大
已儲存的清單。由已驗證身分且未使用 `--tools` 的操作員建立的工作，會儲存
不受限制的 `*` 政策；`cron edit --clear-tools` 會還原該明確的不受限制
政策。在明確工具政策推出前建立的現有工作會保留目前行為，
直到其工具政策被明確編輯或重新建立工作為止。

`--model` 會設定工作的主要模型；它不會取代工作階段的 `/model` 覆寫，因此已設定的備援鏈仍會套用於其上。無法解析或不允許的模型會使執行因明確的驗證錯誤而失敗，而不會以靜默方式退回預設值。如果工作具有 `--model`，但沒有明確或已設定的備援清單，OpenClaw 會傳入空白備援覆寫，而不會以靜默方式將代理程式主要模型附加為隱藏的重試目標。

隔離工作的模型選擇優先順序，由高至低：

1. 各工作承載內容 `model`（明確設定；不允許的模型會使執行失敗）
2. Gmail 鉤子的模型覆寫（僅限執行來自 Gmail 且允許該覆寫時）
3. 使用者選取且已儲存的排程工作階段模型覆寫
4. 代理程式／預設模型選擇

快速模式會遵循已解析的即時選擇。如果所選模型設定具有 `params.fastMode`，隔離排程預設會使用它；已儲存的工作階段 `fastMode` 覆寫（其次為代理程式 `fastModeDefault`）在任一方向上仍優先於模型設定。自動模式使用模型的 `params.fastAutoOnSeconds` 臨界值，預設為 60 秒。

如果執行遇到即時模型切換交接，排程會使用切換後的提供者／模型重試，並為目前執行持久保存該選擇（以及任何新的驗證設定檔）。重試次數有限：首次嘗試加上 2 次切換重試後，排程會中止，而不會持續循環。

在隔離執行開始前，OpenClaw 會檢查已設定之 `api: "ollama"` 與 `api: "openai-completions"` 提供者的可連線本機端點，這些提供者的 `baseUrl` 為迴路、私人網路或 `.local`。此前置檢查會逐一檢查工作的已設定備援鏈，只有在所有候選項目都無法連線時，才將執行標記為 `skipped`；`--fallbacks ""` 會將該檢查嚴格限制為僅檢查主要模型。端點停機時，會以清楚的錯誤將執行記錄為 `skipped`，而不會開始模型呼叫。結果會依端點快取 5 分鐘（不是依工作或模型），因此許多到期工作若共用已停機的本機 Ollama/vLLM/SGLang/LM Studio 伺服器，只需進行一次探測，而不會引發請求風暴。跳過的前置檢查執行不會增加執行錯誤退避；設定 `failureAlert.includeSkipped` 可選擇接收重複的跳過警示。

### 命令承載內容

命令承載內容會在閘道排程器內執行確定性的指令碼，而不會啟動由模型支援的回合。它們會在閘道主機上執行、擷取 stdout/stderr、將執行記錄於排程歷程，並重複使用與代理程式回合工作相同的 `announce`、`webhook` 及 `none` 傳遞模式。

<Note>
命令排程是供操作員管理的閘道自動化介面，而不是代理程式的 `tools.exec` 呼叫。建立、更新、移除或手動執行排程工作需要 `operator.admin`；之後排定的命令執行，會在閘道程序內以該管理員撰寫的自動化內容執行。代理程式執行政策（`tools.exec.mode`、核准提示、各代理程式工具允許清單）管理模型可見的執行工具，而非命令排程承載內容。
</Note>

```bash
openclaw cron create "*/15 * * * *" \
  --name "Queue depth probe" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>` 會儲存 `argv: ["sh", "-lc", <shell>]`。使用 `--command-argv '["node","scripts/report.mjs"]'` 可執行精確的 argv，而不經過殼層剖析。選用的 `--command-env KEY=VALUE`（可重複）、`--command-input`、`--timeout-seconds`（預設 10 分鐘）、`--no-output-timeout-seconds` 及 `--output-max-bytes` 可控制程序環境、標準輸入與輸出界限。

傳遞的文字取自程序輸出：非空白的 stdout 優先；如果 stdout 為空白而 stderr 非空白，則傳遞 stderr；如果兩者都有內容，排程會傳送小型的 `stdout:`／`stderr:` 區塊。結束代碼 `0` 會將執行記錄為 `ok`；非零結束、訊號、逾時或無輸出逾時會記錄為 `error`，並可能觸發失敗警示。只輸出 `NO_REPLY` 的命令會使用正常的排程靜默權杖抑制機制，不會將任何內容回傳至聊天。

### 指令碼承載內容

指令碼酬載會在與觸發指令碼相同的程式碼模式執行器中以無介面方式執行，而不會啟動對話式代理程式回合。建立或執行前，請啟用 `cron.triggers.enabled`；這個危險自動化閘門同時涵蓋觸發指令碼與指令碼酬載。指令碼工作僅支援 `main` 和 `isolated` 工作階段目標。

```bash
openclaw cron create "0 * * * *" \
  --name "Hourly queue check" \
  --script ./automation/check-queue.js \
  --script-timeout-seconds 300 \
  --script-tool-budget 50 \
  --session isolated \
  --announce
```

使用 `--script <file|->` 從檔案或標準輸入讀取 JavaScript。逾時預設為 300 秒，上限為 900 秒；工具預算預設為 50 次呼叫，上限為 200 次。這些酬載預算與較小的觸發閘門評估預算分開計算。

指令碼可以傳回包含下列選用欄位的物件：

- `notify`：透過工作的 `announce`、`webhook` 或 `none` 傳遞模式送出的文字。若省略，則不會傳遞任何內容。若為 `main` 工作，該文字會成為系統事件。
- `wake`：`"now"` 會要求在將 `notify`（或精簡的完成事件）加入佇列後立即進行心跳偵測；`"next-heartbeat"` 則會將事件加入佇列，留待下一次心跳偵測。
- `state`：JSON 狀態，上限為 16 KB，且僅在成功執行後保存。下一次執行會以 `trigger.state` 接收其凍結副本，行為與觸發指令碼一致。由於該命名空間只有一個持久化擁有者，因此同一個工作不能同時使用指令碼酬載與條件觸發器。
- `nextCheck`：例如 `"15m"` 的持續時間。它僅適用於已啟用步調控制的工作，並使用與代理程式回合提案相同的步調限制。

擲回例外、逾時、耗盡工具預算、無效結果，以及未啟用步調控制時使用 `nextCheck`，都屬於一般的排程執行錯誤：這些錯誤會進入執行歷程、退避與失敗警示處理流程，且不會保存傳回的狀態。

## 執行方式

| 方式           | `--session` 值   | 執行位置                  | 最適合                        |
| --------------- | ------------------- | ------------------------ | ------------------------------- |
| 主要工作階段    | `main`              | 專用排程喚醒通道 | 提醒、系統事件        |
| 隔離            | `isolated`          | 專用 `cron:<jobId>` | 報告、背景例行工作      |
| 目前工作階段 | `current`           | 建立時繫結   | 可感知內容脈絡的週期性工作    |
| 自訂工作階段  | `session:custom-id` | 持久命名工作階段 | 以歷程為基礎延續的工作流程 |

<AccordionGroup>
  <Accordion title="主要工作階段、隔離與自訂工作階段的比較">
    **主要工作階段**工作會將系統事件加入排程擁有的執行通道，並可選擇喚醒心跳偵測（`--wake now` 或 `--wake next-heartbeat`）。它們可以使用目標主要工作階段上次的傳遞內容脈絡來回覆，但不會將例行排程回合附加至人類聊天通道，也不會延長目標工作階段每日／閒置重設的新鮮度。**隔離**工作會以全新工作階段執行專用代理程式回合。**自訂工作階段**（`session:xxx`）會跨執行保留內容脈絡，從而支援每日站立會議等以先前摘要為基礎延續的工作流程。

    主要工作階段的排程事件是獨立完整的系統事件提醒。它們不會自動包含預設的心跳偵測提示或心跳偵測監控暫存內容；若提醒應查閱該內容脈絡，請在排程事件文字中明確說明。

  </Accordion>
  <Accordion title="隔離工作中的「全新工作階段」代表什麼">
    每次執行都會使用新的逐字稿／工作階段 ID。OpenClaw 會沿用安全的偏好設定（思考／快速／詳細程度設定、標籤、使用者明確選取的模型／驗證覆寫），但不會從較舊的排程資料列繼承環境對話內容脈絡：頻道／群組路由、傳送或佇列原則、權限提升、來源或 ACP 執行階段繫結。若週期性工作應刻意以相同對話內容脈絡為基礎延續，請使用 `current` 或 `session:<id>`。
  </Accordion>
  <Accordion title="無人值守執行合約">
    隔離排程與掛鉤代理程式回合明確屬於無人值守：沒有人能在場釐清或核准。最終回覆必須是交付成果，而不是計畫、確認或輸入要求。若無須執行任何動作，代理程式會傳回 `HEARTBEAT_OK`，並清楚說明失敗；排程負責重試與失敗警示原則。

    對於受信任的排程工作，若工作本身的指示刻意要求提出問題或計畫，則以該指示為準；代理程式也可以移除已不再需要的工作。外部掛鉤回合只會收到共用的無人值守合約；跨越外部內容邊界後，它們不會收到該覆寫或自行移除指引。

  </Accordion>
  <Accordion title="子代理程式與 Discord 傳遞">
    當隔離排程執行協調子代理程式時，傳遞會優先採用最終後代輸出，而不是過時的父項中間文字。若後代仍在執行，OpenClaw 會抑制該部分父項更新，而不會發出通知。

    對於純文字 Discord 公告目標，OpenClaw 只會傳送一次標準最終助理文字，而不會同時重播串流／中間文字與最終答案。媒體與結構化 Discord 酬載仍會分開傳遞，以免遺漏附件與元件。

  </Accordion>
</AccordionGroup>

## 傳遞與輸出

| 模式       | 行為                                                        |
| ---------- | ------------------------------------------------------------------- |
| `announce` | 若代理程式未傳送，則將最終文字備援傳遞至目標 |
| `webhook`  | 將完成事件酬載以 POST 傳送至 URL                                |
| `none`     | 執行器不進行備援傳遞                                         |

使用 `--announce --channel telegram --to "-1001234567890"` 進行頻道傳遞。對於 Telegram 論壇主題，請使用 `-1001234567890:topic:123`；OpenClaw 也接受 Telegram 擁有的 `-1001234567890:123` 簡寫。直接 RPC／設定呼叫端可以字串或數字形式傳入 `delivery.threadId`。Slack／Discord／Mattermost 目標使用明確前綴（`channel:<id>`、`user:<id>`）。Matrix 聊天室 ID 區分大小寫；請使用確切的聊天室 ID 或 Matrix 提供的 `room:!room:server` 形式。

當公告傳遞使用 `channel: "last"` 或省略 `channel` 時，像 `telegram:123` 這類帶有供應者前綴的目標，可以在排程改用工作階段歷程或單一已設定頻道之前選取頻道。只有已載入外掛所公告的前綴才是供應者選擇器。若明確指定 `delivery.channel`，目標前綴必須指名相同的供應者；使用 `channel: "whatsapp"` 搭配 `to: "telegram:123"` 會遭拒絕，而不會讓 WhatsApp 將 Telegram ID 解讀為電話號碼。目標種類與服務前綴（`channel:<id>`、`user:<id>`、`imessage:<handle>`、`sms:<number>`）仍是頻道所擁有的目標語法，而不是供應者選擇器。

對於隔離工作，聊天傳遞是共用的：若有可用的聊天路由，即使使用 `--no-deliver`，代理程式仍可使用 `message` 工具。若代理程式傳送至已設定／目前的目標，OpenClaw 會略過備援公告。除此之外，`announce`、`webhook` 和 `none` 僅控制代理程式回合結束後，執行器如何處理最終回覆。

當代理程式從進行中的聊天建立隔離提醒時，OpenClaw 會保存現行傳遞目標，供備援公告路由使用。內部工作階段索引鍵可能為小寫；若可取得目前聊天內容脈絡，則不會從這些索引鍵重建供應者傳遞目標。

隱含公告傳遞會使用已設定的頻道允許清單來驗證並重新路由過時的目標。私訊配對存放區核准項目不是備援自動化收件者；若排程工作應主動傳送至私訊，請設定 `delivery.to` 或設定頻道的 `allowFrom` 項目。

### 失敗通知

失敗通知遵循獨立的目的地路徑：

- `cron.failureDestination` 設定全域失敗通知預設值。
- `job.delivery.failureDestination` 會針對個別工作覆寫該設定。
- 若兩者皆未設定，且工作已透過 `announce` 傳遞，失敗通知會改用該主要公告目標。
- `delivery.failureDestination` 僅支援 `sessionTarget="isolated"` 工作，除非主要傳遞模式為 `webhook`。
- `failureAlert.includeSkipped: true` 讓個別工作或全域排程警示原則選擇啟用重複的略過執行警示。略過的執行會保有獨立的連續略過計數器，因此不會影響執行錯誤退避。
- `openclaw cron edit` 提供個別工作的警示調整：`--failure-alert`/`--no-failure-alert`、`--failure-alert-after <n>`、`--failure-alert-channel`、`--failure-alert-to`、`--failure-alert-cooldown`、`--failure-alert-include-skipped`/`--failure-alert-exclude-skipped`、`--failure-alert-mode` 和 `--failure-alert-account-id`。

### 輸出語言

排程工作不會根據頻道、地區設定或先前訊息推斷回覆語言。請將語言規則放入排程訊息或範本：

```bash
openclaw cron edit <jobId> \
  --message "摘要更新內容。請以中文回覆；網址、程式碼和產品名稱保持不變。"
```

對於範本檔案，請在轉譯後的提示中保留語言指示，並在工作執行前確認 `{{language}}` 等預留位置已填入。若輸出混合多種語言，請明確指定規則，例如：“敘述文字使用中文，技術術語保留英文。”

## 命令列介面範例

<Tabs>
  <Tab title="單次提醒">
    ```bash
    openclaw cron add \
      --name "Calendar check" \
      --at "20m" \
      --session main \
      --system-event "Next heartbeat: check calendar." \
      --wake now
    ```
  </Tab>
  <Tab title="週期性隔離工作">
    ```bash
    openclaw cron create "0 7 * * *" \
      "Summarize overnight updates." \
      --name "Morning brief" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --announce \
      --channel slack \
      --to "channel:C1234567890"
    ```
  </Tab>
  <Tab title="模型與思考覆寫">
    ```bash
    openclaw cron add \
      --name "Deep analysis" \
      --cron "0 6 * * 1" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --message "Weekly deep analysis of project progress." \
      --model "opus" \
      --thinking high \
      --announce
    ```
  </Tab>
  <Tab title="網路鉤子輸出">
    ```bash
    openclaw cron create "0 18 * * 1-5" \
      "Summarize today's deploys as JSON." \
      --name "Deploy digest" \
      --webhook "https://example.invalid/openclaw/cron"
    ```
  </Tab>
  <Tab title="命令輸出">
    ```bash
    openclaw cron create "*/15 * * * *" \
      --name "Queue depth probe" \
      --command "scripts/check-queue.sh" \
      --command-cwd "/srv/app" \
      --announce \
      --channel telegram \
      --to "-1001234567890"
    ```
  </Tab>
</Tabs>

## 管理工作

```bash
# 列出已啟用的作業
openclaw cron list

# 包含已停用的作業
openclaw cron list --all

# 以 JSON 取得一個已儲存的作業
openclaw cron get <jobId>

# 顯示一個作業，包括解析後的傳遞路由
openclaw cron show <jobId>

# 啟用／停用而不刪除
openclaw cron enable <jobId>
openclaw cron disable <jobId>

# 編輯作業
openclaw cron edit <jobId> --message "更新後的提示詞" --model "opus"

# 立即強制執行作業
openclaw cron run <jobId>

# 立即強制執行作業並等待其終止狀態
openclaw cron run <jobId> --wait --wait-timeout 10m --poll-interval 2s

# 僅在到期時執行
openclaw cron run <jobId> --due

# 檢視執行歷程
openclaw cron runs --id <jobId> --limit 50

# 檢視一次確切的執行
openclaw cron runs --id <jobId> --run-id <runId>

# 刪除作業
openclaw cron remove <jobId>

# 代理程式選擇（多代理程式設定）
openclaw cron create "0 6 * * *" "檢查維運佇列" --name "維運巡查" --session isolated --agent ops
openclaw cron edit <jobId> --clear-agent
```

封存工作階段（透過控制介面，或由 operator-admin 呼叫者使用 `sessions.patch { archived: true }`）會停用綁定至該工作階段的所有已啟用排程作業：其隔離的 `cron:<jobId>` 工作階段、`session:<key>` 目標，或傳遞／喚醒 `sessionKey` 通道。還原工作階段不會重新啟用這些作業；請使用 `openclaw cron enable <jobId>`。具有已啟用綁定作業的工作階段會在控制介面側邊欄顯示時鐘徽章。

`openclaw cron run <jobId>` 會在手動執行排入佇列後返回。對於必須阻塞直到佇列中的執行完成的關閉掛鉤、維護指令碼或其他自動化，請使用 `--wait`；它會輪詢返回的 `runId`（預設逾時 `10m`，輪詢間隔 `2s`），狀態為 `ok` 時以 `0` 結束，狀態為 `error`、`skipped` 或等待逾時時則以非零值結束。

代理程式 `cron` 工具會從 `cron(action: "list")` 返回精簡的作業摘要（`id`、`name`、`enabled`、`nextRunAtMs`、`scheduleKind`、`lastRunStatus`）；使用 `cron(action: "get", jobId: "...")` 取得一個完整的作業定義。直接呼叫閘道的呼叫者可將 `compact: true` 傳給 `cron.list`；省略它會保留包含傳遞預覽的完整回應。

`openclaw cron create` 是 `openclaw cron add` 的別名。新作業可使用位置式排程（`"0 9 * * 1"`、`"every 1h"`、`"20m"` 或 ISO 時間戳記），後接位置式代理程式提示詞。在 `cron add|create` 或 `cron edit` 上使用 `--webhook <url>`，將完成的執行承載資料以 POST 傳送至 HTTP 端點；網路鉤子傳遞無法與聊天傳遞旗標（`--announce`、`--channel`、`--to`、`--thread-id`、`--account`）搭配使用。在 `cron edit`、`--clear-channel`、`--clear-to`、`--clear-thread-id` 和 `--clear-account` 上，分別取消設定這些路由欄位（每個旗標與其對應的設定旗標並用時都會遭拒）——這與 `--no-deliver` 不同，後者僅停用執行器的備援傳遞。

<Note>
模型覆寫注意事項：

- `openclaw cron add|edit --model ...` 會變更作業選取的模型。
- 如果模型獲准使用，該確切的供應商／模型會用於隔離的代理程式執行。
- 如果模型未獲准使用或無法解析，排程會以明確的驗證錯誤使該次執行失敗。
- API `cron.update` 承載資料修補可將 `model: null` 設為清除已儲存作業的模型覆寫。
- `openclaw cron edit <job-id> --clear-model` 會從命令列介面清除該覆寫（效果與 `model: null` 修補相同），且無法與 `--model` 搭配使用。
- 已設定的備援鏈仍會套用，因為排程 `--model` 是作業的主要模型，而不是工作階段的 `/model` 覆寫。
- `openclaw cron add|edit --fallbacks ...` 會設定承載資料 `fallbacks`，取代該作業已設定的備援；`--fallbacks ""` 會停用備援並使執行採用嚴格模式。`openclaw cron edit <job-id> --clear-fallbacks` 會清除每個作業的覆寫。
- 若單純的 `--model` 沒有明確或已設定的備援清單，就不會無提示地轉而將代理程式的主要模型作為額外重試目標。

</Note>

## 網路鉤子

閘道可公開 HTTP 網路鉤子端點，供外部觸發器使用。在設定中啟用：

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
  },
}
```

### 驗證

每個請求都必須透過標頭包含掛鉤權杖：

- `Authorization: Bearer <token>`（建議）
- `x-openclaw-token: <token>`

查詢字串權杖會遭拒。

<AccordionGroup>
  <Accordion title="POST /hooks/wake">
    將系統事件排入主要工作階段的佇列：

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/wake \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"text":"收到新電子郵件","mode":"now"}'
    ```

    <ParamField path="text" type="string" required>
      事件描述。
    </ParamField>
    <ParamField path="mode" type="string" default="now">
      `now` 或 `next-heartbeat`。
    </ParamField>

  </Accordion>
  <Accordion title="POST /hooks/agent">
    執行一次隔離的代理程式輪次：

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/agent \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"message":"摘要收件匣","name":"電子郵件","model":"openai/gpt-5.6-sol"}'
    ```

    欄位：`message`（必填）、`name`、`agentId`、`sessionKey`（需要 `hooks.allowRequestSessionKey=true`）、`idempotencyKey`、`wakeMode`、`deliver`、`channel`、`to`、`model`、`thinking`、`timeoutSeconds`。

  </Accordion>
  <Accordion title="對應掛鉤（POST /hooks/<name>）">
    自訂掛鉤名稱會透過設定中的 `hooks.mappings` 解析。對應可使用範本或程式碼轉換，將任意承載資料轉換為 `wake` 或 `agent` 動作。
  </Accordion>
</AccordionGroup>

<Warning>
請將掛鉤端點置於回送介面、tailnet 或受信任的反向 Proxy 之後。

- 使用專用的掛鉤權杖；不要重複使用閘道驗證權杖。
- 將 `hooks.path` 保留在專用的子路徑；`/` 會遭拒。
- 設定 `hooks.allowedAgentIds`，以限制掛鉤可指定的有效代理程式，包括省略 `agentId` 時的預設代理程式。
- 除非需要由呼叫者選取工作階段，否則請保留 `hooks.allowRequestSessionKey=false`。
- 如果啟用 `hooks.allowRequestSessionKey`，也請設定 `hooks.allowedSessionKeyPrefixes`，以限制允許的工作階段金鑰格式。
- 預設會以安全邊界包裝掛鉤承載資料。

</Warning>

## Gmail PubSub 整合

透過 Google PubSub 將 Gmail 收件匣觸發器連接至 OpenClaw。

<Note>
**先決條件：** `gcloud` 命令列介面、`gog`（gogcli）、已啟用 OpenClaw 掛鉤，以及用於公開 HTTPS 端點的 Tailscale。
</Note>

### 精靈設定（建議）

```bash
openclaw webhooks gmail setup --account openclaw@gmail.com
```

這會寫入 `hooks.gmail` 設定、啟用 Gmail 預設集，並將推送端點預設為 Tailscale Funnel（`--tailscale funnel|serve|off`）。

<Warning>
Gmail 預設集的每封郵件工作階段會分隔對話內容；它不會限制目標代理程式的工具或工作區。若沒有設定 `agentId` 的自訂對應，Gmail 掛鉤會以預設代理程式執行。

對於不受信任的收件匣，請將掛鉤路由至專用的讀取代理程式，僅授予該代理程式唯讀或不授予工作區存取權，並拒絕檔案系統寫入、Shell、瀏覽器及其他不必要的工具。如果它需要通知主要代理程式，僅允許必要的代理程式間交接。請參閱[提示詞注入](/zh-TW/gateway/security#prompt-injection)、[多代理程式沙箱與工具](/zh-TW/tools/multi-agent-sandbox-tools)及 [`tools.agentToAgent`](/zh-TW/gateway/config-tools#toolsagenttoagent)。
</Warning>

### 閘道自動啟動

設定 `hooks.enabled=true` 和 `hooks.gmail.account` 後，閘道會在開機時啟動 `gog gmail watch serve`，並自動續訂監看。設定 `OPENCLAW_SKIP_GMAIL_WATCHER=1` 可選擇停用。

### 手動一次性設定

<Steps>
  <Step title="選取 GCP 專案">
    選取擁有 `gog` 所用 OAuth 用戶端的 GCP 專案：

    ```bash
    gcloud auth login
    gcloud config set project <project-id>
    gcloud services enable gmail.googleapis.com pubsub.googleapis.com
    ```

  </Step>
  <Step title="建立主題並授予 Gmail 推送存取權">
    ```bash
    gcloud pubsub topics create gog-gmail-watch
    gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
      --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
      --role=roles/pubsub.publisher
    ```
  </Step>
  <Step title="啟動監看">
    ```bash
    gog gmail watch start \
      --account openclaw@gmail.com \
      --label INBOX \
      --topic projects/<project-id>/topics/gog-gmail-watch
    ```
  </Step>
</Steps>

### Gmail 模型覆寫

```json5
{
  hooks: {
    gmail: {
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

對於不受信任的收件匣，請使用供應商提供的最新一代最高階模型。上述值僅為範例；該模型必須存在於已設定的目錄與允許清單中。

## 設定

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    triggers: {
      enabled: false,
    },
    webhookToken: "replace-with-dedicated-webhook-token",
    sessionRetention: "24h",
  },
}
```

`webhookToken` 會在排程網路鉤子 POST 中以 `Authorization: Bearer <token>` 傳送。

`cron.store` 是邏輯儲存金鑰與 doctor 移轉路徑，不是可手動編輯的即時 JSON 檔案。作業資料位於 SQLite 中；請使用命令列介面或閘道 API 進行變更。

停用排程：`cron.enabled: false` 或 `OPENCLAW_SKIP_CRON=1`。

<AccordionGroup>
  <Accordion title="重試行為">
    **單次作業重試**：暫時性錯誤（速率限制、過載、網路、逾時、伺服器錯誤）會使用內建的重試排程。永久性錯誤會立即停用作業。

    **週期性作業重試**：連續執行錯誤會依延長的排程進行指數退避（30s、60s、5m、15m、60m）。下次成功執行後，退避會重設。

  </Accordion>
  <Accordion title="維護">
    `cron.sessionRetention`（預設 `24h`，`false` 會停用）會清除隔離的執行工作階段項目。執行歷程會為每個作業保留最新的 2000 筆終止記錄；遺失的記錄仍保留其 24 小時清理期限。
  </Accordion>
  <Accordion title="舊版儲存移轉">
    升級時，執行 `openclaw doctor --fix`，將舊版 `~/.openclaw/cron/jobs.json`、`jobs-state.json` 和 `runs/*.jsonl` 檔案匯入 SQLite，並以 `.migrated` 後綴重新命名。格式錯誤的作業記錄會在執行階段略過，並複製到 `jobs-quarantine.json`，供日後修復或檢閱。
  </Accordion>
</AccordionGroup>

## 疑難排解

### 指令階梯

```bash
openclaw status
openclaw gateway status
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
openclaw doctor
```

<AccordionGroup>
  <Accordion title="排程未觸發">
    - 檢查 `cron.enabled` 和 `OPENCLAW_SKIP_CRON` 環境變數。
    - 確認閘道持續運行。
    - 對於 `cron` 排程，請確認時區（`--tz`）與主機時區。
    - 執行輸出中的 `reason: not-due` 表示手動執行使用 `openclaw cron run <jobId> --due` 進行檢查，而作業尚未到期。

  </Accordion>
  <Accordion title="排程已觸發但未傳遞">
    - 傳遞模式 `none` 表示預期不會由執行器進行備援傳送。當有可用的聊天路由時，代理程式仍可使用 `message` 工具直接傳送。
    - 傳遞目標缺失或無效（`channel`/`to`）表示已略過對外傳送。
    - 對於 Matrix，複製的工作或舊版工作若使用小寫的 `delivery.to` 聊天室 ID，可能會失敗，因為 Matrix 聊天室 ID 區分大小寫。請編輯工作，使用 Matrix 中確切的 `!room:server` 或 `room:!room:server` 值。
    - 頻道驗證錯誤（`unauthorized`、`Forbidden`）表示傳遞因認證資訊而遭阻擋。
    - 如果隔離執行僅傳回靜默權杖（`NO_REPLY` / `no_reply`），OpenClaw 會抑制直接對外傳遞與備援的佇列摘要路徑，因此不會將任何內容傳回聊天。
    - 如果代理程式應自行傳訊息給使用者，請確認工作有可用的路由（`channel: "last"` 搭配先前的聊天，或明確指定頻道／目標）。

  </Accordion>
  <Accordion title="排程或心跳偵測似乎會阻止 /new-style 輪替">
    - 每日與閒置重設的新鮮度並非以 `updatedAt` 為依據；請參閱[工作階段管理](/zh-TW/concepts/session#session-lifecycle)。
    - 排程喚醒、心跳偵測執行、exec 通知與閘道簿記可能會更新工作階段資料列以供路由／狀態使用，但不會延長 `sessionStartedAt` 或 `lastInteractionAt`。
    - 對於在這些欄位存在前建立的舊版資料列，只要檔案仍可用，OpenClaw 就能從逐字記錄 JSONL 的工作階段標頭復原 `sessionStartedAt`。沒有 `lastInteractionAt` 的舊版閒置資料列會使用該復原的開始時間作為閒置基準。

  </Accordion>
  <Accordion title="時區注意事項">
    - 未設定 `--tz` 的排程會使用閘道主機的時區。
    - 未設定時區的 `at` 排程會視為 UTC。
    - 心跳偵測 `activeHours` 會使用設定的時區解析方式。

  </Accordion>
</AccordionGroup>

## 相關內容

- [自動化](/zh-TW/automation) — 一覽所有自動化機制
- [背景任務](/zh-TW/automation/tasks) — 排程執行的任務帳本
- [心跳偵測](/zh-TW/gateway/heartbeat) — 定期執行的主要工作階段輪次
- [時區](/zh-TW/concepts/timezone) — 時區設定
