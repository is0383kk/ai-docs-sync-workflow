---
read_when:
    - 你想要排程工作與喚醒功能
    - 你正在偵錯排程執行與記錄檔
summary: '`openclaw cron` 的命令列介面參考（排程並執行背景工作）'
title: 排程
x-i18n:
    generated_at: "2026-07-26T07:12:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5989a7558f4ae2f046480b6a52e3fa296c95d47b14b11c5bad709fea4af6af3e
    source_path: cli/cron.md
    workflow: 16
---

# `openclaw cron`

管理閘道排程器的排程工作。

<Tip>
執行 `openclaw cron --help` 以查看完整的命令介面。概念指南請參閱[排程工作](/zh-TW/automation/cron-jobs)。
</Tip>

<Note>
所有排程變更（`add`/`create`、`update`/`edit`、`remove`、`run`）都需要 `operator.admin`。命令酬載執行會直接在閘道程序中進行，而非作為代理程式的 `tools.exec` 工具呼叫；`tools.exec.*` 和執行核准仍會管控模型可見的執行工具。
</Note>

## 快速建立工作

`openclaw cron create` 是 `openclaw cron add` 的別名。建立新工作時，請先放排程，再放提示詞：

```bash
openclaw cron create "0 7 * * *" \
  "彙整夜間更新。" \
  --name "晨間摘要" \
  --agent ops
```

當工作應以 POST 傳送完成的酬載，而非傳送至聊天目標時，請使用 `--webhook <url>`：

```bash
openclaw cron create "0 18 * * 1-5" \
  "以 JSON 彙整今天的部署。" \
  --name "部署摘要" \
  --webhook "https://example.invalid/openclaw/cron"
```

若要執行確定性的 shell 風格工作，直接在 OpenClaw 排程中執行，而不啟動隔離的代理程式／模型執行，請使用 `--command`：

```bash
openclaw cron create "*/15 * * * *" \
  --name "佇列深度探測" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>` 會儲存 `argv: ["sh", "-lc", <shell>]`。若要精確執行 argv，請使用 `--command-argv '["node","scripts/report.mjs"]'`。命令工作會擷取 stdout/stderr、記錄一般排程歷程，並透過與隔離工作相同的 `announce`、`webhook` 或 `none` 傳送模式路由輸出。僅印出 `NO_REPLY` 的命令會被抑制。

## 工作階段

`--session` 接受 `main`、`isolated`、`current` 或 `session:<id>`。

<AccordionGroup>
  <Accordion title="工作階段金鑰">
    - `main` 會繫結至代理程式的主要工作階段。
    - `isolated` 會為每次執行建立新的逐字記錄和工作階段 ID。
    - `current` 會繫結至建立時的作用中工作階段。
    - `session:<id>` 會固定至明確的持久工作階段金鑰。

  </Accordion>
  <Accordion title="隔離工作階段語意">
    隔離執行會重設周遭對話內容。新執行會重設頻道與群組路由、傳送／佇列政策、權限提升、來源及 ACP 執行階段繫結。安全的偏好設定，以及使用者明確選取的模型或驗證覆寫，可以延續至後續執行。
  </Accordion>
</AccordionGroup>

## 傳送

`openclaw cron list` 和 `openclaw cron show <job-id>` 會預覽解析後的傳送路由。對於 `channel: "last"`，預覽會顯示路由是從主要或目前工作階段解析而來，或將以關閉方式失敗。

帶有提供者前綴的目標可釐清尚未解析的公告頻道。例如，當省略 `delivery.channel` 或其為 `last` 時，`to: "telegram:123"` 會選取 Telegram。只有已載入外掛所宣告的前綴才是提供者選擇器。如果明確指定 `delivery.channel`，前綴必須符合該頻道；搭配 `to: "telegram:123"` 的 `channel: "whatsapp"` 會被拒絕。`imessage:` 和 `sms:` 等服務前綴仍屬於頻道自行管理的目標語法。

<Note>
隔離的 `cron add` 工作預設使用 `--announce` 傳送。使用 `--no-deliver` 可將輸出保留在內部。`--deliver` 仍是 `--announce` 的已棄用別名。
</Note>

### 傳送責任歸屬

隔離排程的聊天傳送由代理程式與執行器共同負責：

- 當聊天路由可用時，代理程式可使用 `message` 工具直接傳送。
- 僅當代理程式未直接傳送至解析後的目標時，`announce` 才會以備援方式傳送最終回覆。
- `webhook` 會將完成的酬載 POST 至 URL。
- `none` 會停用執行器的備援傳送。

使用 `cron add|create --webhook <url>` 或 `cron edit <job-id> --webhook <url>` 設定網路鉤子傳送。請勿將 `--webhook` 與 `--announce`、`--no-deliver`、`--channel`、`--to`、`--thread-id` 或 `--account` 等聊天傳送旗標搭配使用。

`cron edit <job-id>` 可透過 `--clear-channel`、`--clear-to`、`--clear-thread-id` 和 `--clear-account` 取消設定個別傳送路由欄位（每一項若與其對應的設定旗標搭配使用，都會被拒絕）。`--no-deliver` 只會停用執行器備援傳送；不同的是，這些選項會移除已儲存的欄位，使工作再次從預設值解析該部分路由。

`--announce` 是最終回覆的執行器備援傳送。`--no-deliver` 會停用該備援，但當聊天路由可用時，不會移除代理程式的 `message` 工具。

從作用中聊天建立的提醒會保留即時聊天傳送目標，供備援公告傳送使用。內部工作階段金鑰可能為小寫；請勿將其作為 Matrix 房間 ID 等區分大小寫之提供者 ID 的真實來源。

### 失敗傳送

失敗通知依下列順序解析：

1. 工作上的 `delivery.failureDestination`。
2. 全域 `cron.failureDestination`。
3. 工作的主要公告目標（前兩者皆未解析至具體目的地時）。

<Note>
只有當主要傳送模式為 `webhook` 時，主要工作階段工作才能使用 `delivery.failureDestination`。隔離工作在所有模式下都可使用。
</Note>

即使未產生回覆酬載，隔離排程執行也會將執行層級的代理程式失敗視為工作錯誤，因此模型／提供者失敗仍會增加錯誤計數器並觸發失敗通知。

命令排程工作不會啟動隔離的代理程式回合。結束代碼為零時會記錄 `ok`；非零結束代碼、訊號、逾時或無輸出逾時則會記錄 `error`，並可能觸發相同的失敗通知路徑。

如果隔離執行在首次模型請求前逾時，`openclaw cron show` 和 `openclaw cron runs` 會包含階段特定錯誤，例如 `setup timed out before runner start`，或指出最後已知啟動階段的停滯訊息（例如 `context-engine`）。對於命令列介面支援的提供者，模型前監控程序會持續運作，直到外部命令列介面回合開始，因此工作階段查詢、鉤子、驗證、提示詞及命令列介面設定的停滯都會回報為模型前排程失敗。

## 排程

### 單次工作

`--at <datetime>` 會排定單次執行。未含時差的日期時間會視為 UTC，除非同時傳入 `--tz <iana>`，此時會依指定時區解讀當地鐘面時間。

<Note>
單次工作預設會在成功後刪除。使用 `--keep-after-run` 可予以保留。
</Note>

### 週期性工作

週期性工作在連續發生錯誤後，會使用指數重試退避：30s、1m、5m、15m、60m。下一次成功執行後，排程會恢復正常。

略過的執行會與執行錯誤分開追蹤。它們不會影響重試退避，但 `openclaw cron edit <job-id> --failure-alert-include-skipped` 可讓失敗警示包含重複的略過執行通知。

對於以本機已設定模型提供者為目標的隔離工作（基底 URL 位於迴路介面、私人網路或 `.local`），排程會在啟動代理程式回合前執行輕量的提供者預檢：在 `/api/tags` 探測 `api: "ollama"` 提供者；在 `/models` 探測其他本機 OpenAI 相容提供者（`api: "openai-completions"`，例如 vLLM、SGLang、LM Studio）。如果端點無法連線，該次執行會記錄為 `skipped`，並在之後的排程重試；每個端點的可連線性結果會快取 5 分鐘，以免許多使用相同本機伺服器的工作反覆探測並造成負荷。

排程工作、待處理的執行階段狀態及執行歷程都位於共用的 SQLite 狀態資料庫中。舊版 `jobs.json`、`<name>-state.json` 和 `runs/*.jsonl` 檔案會匯入一次，並以 `.migrated` 後綴重新命名。匯入後，請使用 `openclaw cron add|edit|remove` 編輯排程，而非編輯 JSON 檔案。

### 手動執行

`openclaw cron run <job-id>` 預設會強制執行，並在手動執行排入佇列後立即返回。成功回應包含 `{ ok: true, enqueued: true, runId }`。使用傳回的 `runId` 查看之後的結果：

```bash
openclaw cron run <job-id>
openclaw cron runs --id <job-id> --run-id <run-id>
```

如果指令碼應阻塞至該次排入佇列的確切執行記錄終止狀態，請加入 `--wait`：

```bash
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
```

搭配 `--wait` 時，命令列介面仍會先呼叫 `cron.run`，再針對傳回的 `runId` 輪詢 `cron.runs`。只有當執行以 `ok` 狀態完成時，命令才會以 `0` 結束。當執行以 `error` 或 `skipped` 完成、閘道回應不包含 `runId`，或 `--wait-timeout` 到期時（預設為 `10m`，預設每隔 `2s` 輪詢），命令會以非零代碼結束。`--poll-interval` 必須大於零。

<Note>
若只想在工作目前已到期時執行手動命令，請使用 `--due`。如果 `--due --wait` 未將執行排入佇列，命令會傳回一般的未執行回應，而不進行輪詢。
</Note>

## 模型

`cron add|edit --model <ref>` 會為工作選取允許的模型。`cron add|edit --fallbacks <list>` 會設定每個工作的備援模型，例如 `--fallbacks openrouter/gpt-4.1-mini,openai/gpt-5`；傳入 `--fallbacks ""` 可執行不使用備援的嚴格執行。`cron edit <job-id> --clear-fallbacks` 會移除每個工作的備援覆寫。`cron edit <job-id> --clear-model` 會移除每個工作的模型覆寫，使工作遵循一般排程模型選取優先順序（若有已儲存的排程工作階段覆寫則使用它，否則使用代理程式／預設模型）；此選項不能與 `--model` 搭配使用。`cron add|edit --thinking <level>` 會設定每個工作的思考覆寫；`cron edit <job-id> --clear-thinking` 會移除該覆寫，使工作遵循一般排程思考優先順序，且不能與 `--thinking` 搭配使用。

<Warning>
如果模型不受允許或無法解析，排程會以明確的驗證錯誤使該次執行失敗，而不會退回使用工作的代理程式或預設模型選取。
</Warning>

排程 `--model` 是**工作的主要模型**，而非聊天工作階段的 `/model` 覆寫。這表示：

- 當選取的工作模型失敗時，已設定的模型備援仍會套用。
- 如果存在每個工作的酬載 `fallbacks`，它會取代已設定的備援清單。
- 空白的每個工作備援清單（工作酬載／API 中的 `--fallbacks ""` 或 `fallbacks: []`）會使排程執行採用嚴格模式。
- 當工作有 `--model`，但未設定備援清單時，OpenClaw 會傳入明確的空白備援覆寫，使代理程式主要模型不會被附加為隱藏的重試目標。
- 本機提供者預檢會依序檢查已設定的備援，之後才會將排程執行標記為 `skipped`。

`openclaw doctor` 會回報已設定 `payload.model` 的工作，包括提供者命名空間計數，以及與 `agents.defaults.model` 不符的項目。當即時聊天與排程工作之間的驗證、提供者或計費行為有所不同時，請使用此檢查。

### 隔離排程模型優先順序

隔離排程會依下列順序解析作用中模型：

1. Gmail 鉤子覆寫。
2. 每個工作的 `--model`。
3. 已儲存的排程工作階段模型覆寫（使用者選取模型時）。
4. 代理程式或預設模型選取。

### 快速模式

隔離排程快速模式會遵循解析後的即時模型選擇。模型設定 `params.fastMode` 預設會套用，但已儲存工作階段的 `fastMode` 覆寫仍優先於設定。當解析後的模式為 `auto` 時，截止時間會使用所選模型的 `params.fastAutoOnSeconds` 值，預設為 60 秒。

### 即時模型切換重試

如果隔離執行擲回 `LiveSessionModelSwitchError`，排程會在重試前，為目前執行保存切換後的供應商和模型（若有切換後的驗證設定檔覆寫，也會一併保存）。外層重試迴圈在初次嘗試後最多允許兩次切換重試，之後便會中止，而非無限循環。

## 執行輸出與拒絕

### 抑制過時確認訊息

隔離排程回合會抑制僅含過時確認訊息的回覆。如果第一個結果只是暫時狀態更新，且沒有任何後代子代理程式執行負責提供最終答案，排程會在傳送前重新提示一次，以取得真正的結果。

### 抑制靜默權杖

如果隔離排程執行只傳回靜默權杖（`NO_REPLY` 或 `no_reply`），排程會同時抑制直接對外傳送與備援的佇列摘要路徑，因此不會有任何內容回傳至聊天。

### 結構化拒絕

隔離排程執行會將內嵌執行所提供的結構化執行拒絕中繼資料（編碼為 `SYSTEM_RUN_DENIED` 或 `INVALID_REQUEST` 的致命執行工具錯誤）視為權威拒絕訊號。它們也會辨識節點主機的 `UNAVAILABLE` 包裝，其中包含帶有上述任一代碼的巢狀結構化錯誤。

除非內嵌執行也提供結構化拒絕中繼資料，否則排程不會將最終輸出中的文字或看似要求核准的拒絕措辭分類為拒絕，因此一般的助理文字不會被視為遭封鎖的命令。

`cron list` 和執行歷程會顯示拒絕原因，而不會將遭封鎖的命令回報為 `ok`。

## 保留

保留行為：

- `cron.sessionRetention`（預設為 `24h`，或設為 `false` 以停用）會清除已完成的隔離執行工作階段。
- 執行歷程會為每個排程工作保留最新的 2000 個終止狀態資料列。遺失的資料列仍採用標準的 24 小時遺失工作清理期限。

## 移轉較舊的工作

<Note>
如果你的排程工作建立於目前的傳送與儲存格式之前，請執行 `openclaw doctor --fix`。Doctor 會正規化舊版排程欄位（`jobId`、`schedule.cron`、包括舊版 `threadId` 在內的頂層傳送欄位，以及酬載中的 `provider` 傳送別名），並在移除該設定鍵之前，將 `notify: true` 網路鉤子備援工作從已淘汰的原始 `cron.webhook` 值移轉為明確的網路鉤子傳送。已向聊天發送公告的工作會保留該傳送方式，並取得完成網路鉤子的目的地。如果沒有舊版網路鉤子，對於沒有移轉目標的工作，將移除無作用的頂層 `notify` 標記（既有傳送方式會原封不動地保留），因此 `doctor --fix` 不再持續對這些工作發出警告。
</Note>

## 常見編輯

在不變更訊息的情況下更新傳送設定：

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "123456789"
```

停用隔離工作的傳送：

```bash
openclaw cron edit <job-id> --no-deliver
```

為隔離工作啟用輕量啟動內容：

```bash
openclaw cron edit <job-id> --light-context
```

向特定頻道發送公告：

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
```

向 Telegram 論壇主題發送公告：

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "-1001234567890" --thread-id 42
```

建立使用輕量啟動內容的隔離工作：

```bash
openclaw cron create "0 7 * * *" \
  "摘要整理夜間更新。" \
  --name "輕量晨間摘要" \
  --session isolated \
  --light-context \
  --no-deliver
```

`--light-context` 僅適用於隔離的代理程式回合工作。對於排程執行，輕量模式會讓啟動內容保持空白，而非注入完整的工作區啟動集合。

建立具有精確 argv、cwd、env、stdin 和輸出限制的命令工作：

```bash
openclaw cron create "*/30 * * * *" \
  --name "部位匯出" \
  --command-argv '["node","scripts/export-position.mjs"]' \
  --command-cwd "/srv/app" \
  --command-env "NODE_ENV=production" \
  --command-input '{"mode":"summary"}' \
  --timeout-seconds 120 \
  --no-output-timeout-seconds 30 \
  --output-max-bytes 65536 \
  --webhook "https://example.invalid/openclaw/cron"
```

## 常見管理命令

手動執行與檢查：

```bash
openclaw cron list
openclaw cron list --agent ops
openclaw cron get <job-id>
openclaw cron show <job-id>
openclaw cron run <job-id>
openclaw cron run <job-id> --due
openclaw cron run <job-id> --wait --wait-timeout 10m
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
openclaw cron runs --id <job-id> --limit 50
openclaw cron runs --id <job-id> --run-id <run-id>
```

`openclaw cron list` 預設會顯示已啟用的工作。傳入 `--all` 可納入已停用的工作，或傳入 `--agent <id>`，僅顯示其有效正規化代理程式 ID 相符的工作；未儲存代理程式 ID 的工作會視為使用已設定的預設代理程式。

`openclaw cron get <job-id>` 會直接傳回已儲存工作的 JSON。若需要包含傳送路由預覽的人類可讀檢視，請使用 `cron show <job-id>`。

`cron list --json` 和 `cron show <job-id> --json` 會在每個工作的頂層加入 `status` 欄位，此欄位根據 `enabled`、`state.runningAtMs` 和 `state.lastRunStatus` 計算。值包括：`disabled`、`running`、`ok`、`error`、`skipped` 或 `idle`。JSON 狀態會保持標準且不加裝飾，使外部工具無須重新推導即可讀取工作狀態；人類可讀輸出可能會用失敗次數標示重複的 `error` 狀態。

`cron runs` 項目包含傳送診斷資訊，其中包括預期的排程目標、解析後的目標、訊息工具傳送、備援使用情況和已傳送狀態。

每個工作的私有暫存內容（心跳偵測檢查清單和類似的監控內容）：

```bash
openclaw cron scratch <job-id>                  # 顯示目前的暫存內容
openclaw cron scratch <job-id> --json           # 暫存內容及修訂中繼資料
openclaw cron scratch <job-id> --set "text"     # 以指定文字取代暫存內容
openclaw cron scratch <job-id> --file notes.md  # 使用檔案內容取代暫存內容（- 代表 stdin）
openclaw cron scratch <job-id> --unset          # 移除暫存資料列
```

暫存內容會儲存在共用狀態資料庫中，上限為 256 KiB，而且絕不會包含在 `cron list`/`cron get`/`cron runs` 輸出中。寫入會使用命令啟動時讀取的修訂版本，以比較後交換方式防止衝突；也可傳入 `--expected-revision <n>`，改為固定使用明確的修訂版本。如需瞭解心跳偵測監控器如何使用暫存內容，請參閱[心跳偵測](/zh-TW/gateway/heartbeat#monitor-scratch-optional)。

重新指定代理程式和工作階段：

```bash
openclaw cron edit <job-id> --agent ops
openclaw cron edit <job-id> --clear-agent
openclaw cron edit <job-id> --session current
openclaw cron edit <job-id> --session "session:daily-brief"
```

當代理程式回合工作省略 `--agent` 時，`openclaw cron add` 會發出警告，並改用預設代理程式（`main`）。建立工作時傳入 `--agent <id>`，即可固定使用特定代理程式。

傳送調整：

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
openclaw cron edit <job-id> --webhook "https://example.invalid/openclaw/cron"
openclaw cron edit <job-id> --best-effort-deliver
openclaw cron edit <job-id> --no-best-effort-deliver
openclaw cron edit <job-id> --no-deliver
```

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [排程工作](/zh-TW/automation/cron-jobs)
