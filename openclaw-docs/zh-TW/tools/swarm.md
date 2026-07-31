---
read_when:
    - 你想要一個 Code Mode 指令碼，將工作分派給多個代理程式
    - 你需要結構化的子項結果、決策關卡或優先完成的流水線
    - 你正在啟用或調整 tools.swarm 限制
    - 你想在工作階段儀表板中觀察收集器子程序
sidebarTitle: Swarm
summary: 使用具備結構化結果、受限扇出及即時進度的 Code Mode 指令碼，協調並行子代理程式
title: 群集
x-i18n:
    generated_at: "2026-07-26T08:50:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0bec17da7a2e144df35189a65d9b35d829815b545a4bb89652e6a681ca971a9
    source_path: tools/swarm.md
    workflow: 16
---

Swarm 是一種實驗性、可選用的方式，可從 [Code Mode](/zh-TW/tools/code-mode) 指令碼協調多個子代理。使用一般的 JavaScript 或 TypeScript 控制流程，例如 `Promise.all`、`while` 和 `if`，來分派工作、收集結果並做出決策。

它沒有圖形 DSL，也沒有獨立的工作流程格式。程式本身就是協調流程。Swarm 為該程式加入可等待的收集器子項、結構化結果、受限並行處理，以及進度回報。

## 啟用 Swarm

建議的方式是在控制介面中前往 **Settings → Labs → Swarm**。此切換開關會立即生效，並將 `tools.swarm.enabled` 寫入你的設定。

你也可以直接在 `openclaw.json` 中啟用 Swarm：

```json5
{
  tools: {
    swarm: {
      enabled: true,
      maxConcurrent: 8,
      maxChildrenPerGroup: 50,
      maxTotalPerGroup: 200,
      waitTimeoutSecondsMax: 600,
      defaultAgentId: "",
    },
  },
}
```

布林值簡寫會啟用或停用此功能，其他所有值則使用預設值：

```json5
{
  tools: {
    swarm: true,
  },
}
```

| 欄位                    | 預設值 | 說明                                                                                                                           |
| ----------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `enabled`               | `false` | 提供收集器模式的產生選項、`agents_wait`，以及 Code Mode 的 `agents.*` 客體 API。                                   |
| `maxConcurrent`         | `8`     | 單一 Swarm 群組中可同時執行的收集器子項數量上限。額外接受的子項會依先進先出順序排入佇列。          |
| `maxChildrenPerGroup`   | `50`    | 單一群組中存活的收集器子項數量上限。                                                                                  |
| `maxTotalPerGroup`      | `200`   | 群組在其存續期間可產生的收集器子項數量上限。這是防止失控產生的最後防線。                            |
| `waitTimeoutSecondsMax` | `600`   | 單次 `agents_wait` 呼叫可接受的逾時上限。呼叫的預設值為 30 秒。                                            |
| `defaultAgentId`        | `""`    | 當產生作業省略 `agentId` 時使用的目標代理。空值會使用提出要求的代理。現有的子代理允許清單仍然適用。 |

數值必須是正整數。OpenClaw 將
`maxConcurrent` 限制為 `1`–`1000`、將 `maxChildrenPerGroup` 限制為 `1`–`10000`、
將 `maxTotalPerGroup` 限制為 `1`–`100000`，並將 `waitTimeoutSecondsMax` 限制為
`1`–`86400`。

你可以使用 `agents.entries.*.tools.swarm`，為單一已設定的代理覆寫 Swarm。每個代理的物件會合併至頂層 `tools.swarm` 物件之上。

## 需求

`agents.run`、`phase` 和 `log` 客體全域變數同時需要 Swarm 與 OpenClaw Code Mode：

```json5
{
  tools: {
    codeMode: true,
    swarm: true,
  },
}
```

Code Mode 也必須具有對 `sessions_spawn` 的有效存取權。工具設定檔、允許／拒絕政策、供應商規則及沙箱政策都可能移除該工具。
如果指令碼回報 `sessions_spawn` 無法使用，請參閱 [Code Mode 啟用方式](/zh-TW/tools/code-mode#activation)和[子代理](/zh-TW/tools/subagents)。

`defaultAgentId` 和每次執行的 `agentId` 值，都必須指定提出要求者的 `subagents.allowAgents` 政策所允許的已設定目標。OpenClaw 會拒絕未知或不允許的目標，而不會改用其他代理。

## 撰寫 Swarm 指令碼

啟用 Swarm 後，Code Mode 會提供以下客體 API：

```typescript
type AgentRunOptions = {
  label?: string;
  model?: string;
  thinking?: string;
  fastMode?: boolean | "auto";
  agentId?: string;
  schema?: Record<string, unknown>;
  phase?: string;
};

agents.run(prompt: string, options?: AgentRunOptions & { schema?: undefined }): Promise<string>;
agents.run<T>(prompt: string, options: AgentRunOptions & { schema: Record<string, unknown> }): Promise<T>;
phase(title: string): void;
log(message: string): void;
```

若未提供 `schema`，`agents.run()` 會解析為子項的最終文字。若提供 JSON Schema，則會解析為子項透過 `structured_output` 工具提交的值。失敗、遭終止、逾時或結構描述無效的子項，會以 `SwarmAgentError` 拒絕該 Promise。請在 Code Mode 內從 `API.read("agents.d.ts")` 讀取確切產生的宣告與簡短協調慣用法。

使用 `label`，可在儀表板與側邊欄中顯示容易辨識的子項名稱。在選項中使用 `phase`，可在該子項啟動前立即發布階段；若多個子項屬於同一階段，也可呼叫 `phase()`。
`log()` 會發布簡短的進度備註。進度呼叫採射後不理方式；若介面無法使用，也不會延遲指令碼。

### 並行分派並取得結構化結果

此範例會為每個主題啟動一個研究代理，等待所有代理完成，然後要求最後一個子項綜整其結構化報告：

```javascript
const reportSchema = {
  type: "object",
  properties: {
    finding: { type: "string" },
    evidence: { type: "array", items: { type: "string" } },
    confidence: { type: "number" },
  },
  required: ["finding", "evidence", "confidence"],
  additionalProperties: false,
};

const topics = ["authentication", "storage", "recovery"];
phase("獨立審查");

const reports = await Promise.all(
  topics.map((topic) =>
    agents.run(`審查 ${topic} 路徑。傳回一項發現及其證據。`, {
      label: `review-${topic}`,
      thinking: "high",
      fastMode: "auto",
      schema: reportSchema,
    }),
  ),
);

phase("綜整");
log(`已收集 ${reports.length} 份獨立報告。`);

return await agents.run(
  `協調這些報告並說明彼此的歧異：\n${JSON.stringify(reports)}`,
  { label: "synthesis" },
);
```

`Promise.all` 是分派與彙整的邊界。OpenClaw 最多會為群組啟動 `maxConcurrent` 個子項，其餘則依提交順序排入佇列。

Code Mode 另外透過 `tools.codeMode.maxPendingToolCalls` 限制同時執行的客體橋接呼叫（預設為 `16`，上限為 `128`）。對於非常大的群組，請在該限制以下分批啟動，並為 `phase()`、`log()` 和子項等待狀態轉換保留空間。`maxConcurrent` 會限制執行中的子項數量；它不會提高客體橋接呼叫限制。

### 在決策關卡中迴圈執行

當每次執行都會決定是否需要再執行一次時，請使用有界的 `while` 迴圈：

```javascript
const gateSchema = {
  type: "object",
  properties: {
    ready: { type: "boolean" },
    reason: { type: "string" },
    nextAction: { type: "string" },
  },
  required: ["ready", "reason", "nextAction"],
  additionalProperties: false,
};

let pass = 0;
let decision = { ready: false, reason: "尚未檢查", nextAction: "審查" };

while (!decision.ready && pass < 4) {
  pass += 1;
  phase(`決策輪次 ${pass}`);
  decision = await agents.run(
    `檢查發布證據是否完整。上一次的決策：${JSON.stringify(decision)}`,
    {
      label: `release-gate-${pass}`,
      schema: gateSchema,
    },
  );
  log(decision.reason);
}

if (!decision.ready) {
  throw new Error(`經過 ${pass} 個輪次後關卡仍未開啟：${decision.nextAction}`);
}

return decision;
```

決策迴圈一律必須設有限制。`maxTotalPerGroup` 是最終安全防線，不能取代明確的停止條件。

### 處理第一個完成的子項

`agents.run()` 會傳回一般 Promise，因此 `Promise.race` 可對第一個完成的 Code Mode 子項做出反應。對於呼叫較低階工具的測試框架，`agents_wait` 提供相同的首次完成邊界：只要至少一個要求的執行完成，或有界逾時期限到期，它就會傳回。完整的排空迴圈請參閱[從其他測試框架使用 Swarm](#use-swarm-from-other-harnesses)。

## 收集器子項的行為方式

收集器子項是一般的隔離子代理工作階段，但具有不同的完成路徑。它們會寫入持久的收集器結果供父項等待，而不是宣告結果或將回覆引導回父工作階段。

目標代理依下列順序解析：

1. `agentId`，位於產生作業或 `agents.run()` 呼叫上。
2. `tools.swarm.defaultAgentId`。
3. 提出要求的代理。

當 Swarm 子項需要更小的工具介面、成本較低的模型，或更嚴格的沙箱政策時，專用且精簡的工作代理會很有用。OpenClaw 不會內建 `worker` 代理 ID；將其指定為預設值前，請先設定該代理。
請在其個別代理設定中使用 `tools.swarm: false` 強化該工作代理，使其可被產生，但無法從自己的頂層工作階段啟動 Swarm：

```json5
{
  tools: { swarm: { enabled: true, defaultAgentId: "worker" } },
  agents: {
    list: [
      {
        id: "main",
        default: true,
        subagents: { allowAgents: ["worker"] },
      },
      { id: "worker", tools: { swarm: false } },
    ],
  },
}
```

收集器核准採失敗關閉。子項絕不會開啟操作員核准提示。需要核准的工具動作會遭拒絕，而子項可在結果中回報此拒絕，讓指令碼決定下一步。

對於結構化輸出，OpenClaw 會將合成的 `structured_output` 工具加入子項，並根據提供的 JSON Schema 驗證其承載資料。無效或缺少的承載資料會收到一次修正提示。如果重試後仍未通過驗證，收集器完成結果會保留子項的原始文字、讓 `structured` 維持未設定，並包含 `schemaError`。低階 `agents_wait` 結果會公開這些欄位，供明確的復原邏輯使用。

### 子項是葉節點

Swarm 子項預設為葉節點。通用的 `agents.defaults.subagents.maxSpawnDepth` 防護機制會在預設深度 `1` 下，防止子項產生自己的子項。一般的協調慣用法是將工作傳回父項，而不是從子項產生更多工作：

```javascript
const plan = await agents.run("將此工作規劃為彼此獨立的任務。", {
  schema: {
    type: "object",
    properties: { tasks: { type: "array", items: { type: "string" } } },
    required: ["tasks"],
    additionalProperties: false,
  },
});
return await Promise.all(plan.tasks.map((task) => agents.run(task)));
```

巢狀子代理必須由操作員透過 `agents.defaults.subagents.maxSpawnDepth` 選擇啟用，不建議在 Swarm 中使用。群組上限、預算和可觀測性皆以扁平的收集器群組為前提。

每個子項都有一個准入擁有者。宣告型與互動式子項使用 `agents.defaults.subagents.maxChildrenPerAgent`（預設為 `5`），且不計入收集器子項。收集器子項僅使用 `maxChildrenPerGroup` 和 `maxTotalPerGroup`；它們不會耗用每個工作階段的子項預算。產生深度防護機制仍同時適用於兩種模式。

准入後，超出 `maxConcurrent` 的子項會在其 Swarm 群組內依先進先出順序排入佇列，並巢狀位於全域子代理通道中。這些並行處理層會將工作排入佇列，而非拒絕工作。超出任一群組上限的收集器產生作業會遭拒絕，且錯誤中會包含相關的設定鍵。

## 觀察 Swarm

Swarm 運作時，在控制介面中開啟父工作階段的儀表板。Swarm 小工具會呈現每個作用中的收集器群組，每個子項各以一個圓點表示，並顯示已排入佇列、執行中、已完成或失敗狀態。標籤會顯示在圓點的工具提示中，因此簡短且穩定的標籤可讓較大型的 Swarm 更容易閱讀。

工作階段側邊欄會保留一般的父項／子項樹狀結構。展開父項列，即可檢查收集器子項或開啟其逐字記錄，而不會失去 Swarm 階層結構。

收集器結果在其群組封存前仍可等待取得。每個成員都達到保留期限後，OpenClaw 會將該群組的子項目整批封存，使已完成的群集不會留在即時工作階段樹狀結構中。

## 從其他執行框架使用 Swarm

你可以在不使用 OpenClaw Code Mode 的情況下使用 Swarm。其核心工具不依賴執行框架：使用 `sessions_spawn({ collect: true })` 啟動收集器子項目，並透過有界的 `agents_wait` 呼叫取出結果。

Codex Code Mode 會自動在 `tools.*` 下公開符合資格的動態 OpenClaw 工具。它不使用 OpenClaw 的 QuickJS 客體 API，也不需要 `tools.codeMode`，但仍必須啟用 `tools.swarm`。Codex 執行框架的 `agents_wait` 呼叫支援完整的 600 秒逾時。

在目前支援的 Codex 執行階段中，動態 OpenClaw 工具結果會以 JSON 文字傳送至 Code Mode。讀取欄位前，請先剖析每個結果。Codex 也會將動態工具呼叫序列化，因此 `Promise.all` 不會同時提交多個 `sessions_spawn` 呼叫。請在有界迴圈中啟動收集器；已接受的子項目在後續啟動請求提交時仍可繼續執行。

```javascript
function parseToolResult(value) {
  if (typeof value !== "string") return value;
  return JSON.parse(value);
}

const tasks = [
  "檢查驗證路徑。",
  "檢查儲存路徑。",
  "檢查復原路徑。",
];
const launches = [];

for (const [index, task] of tasks.entries()) {
  const launch = parseToolResult(
    await tools.sessions_spawn({
      task,
      collect: true,
      label: `review-${index + 1}`,
    }),
  );
  if (launch.status !== "accepted") {
    throw new Error(launch.error ?? "收集器產生請求未被接受。");
  }
  launches.push(launch);
}

const pending = new Set(launches.map((launch) => launch.runId));
const completed = [];

while (pending.size > 0) {
  const ids = [...pending].slice(0, 1000);
  const batch = parseToolResult(
    await tools.agents_wait({
      ids,
      timeoutSeconds: 30,
    }),
  );

  // 將這個有界視窗輪替至尚未檢查的 ID 之後。
  for (const runId of ids) {
    if (pending.delete(runId)) pending.add(runId);
  }

  for (const item of batch.completed) {
    pending.delete(item.runId);
    if (item.status !== "done") {
      throw new Error(item.schemaError ?? item.result ?? `${item.runId}: ${item.status}`);
    }
    completed.push(item); // 每個結果完成後立即處理。
  }

  for (const failure of batch.errors ?? []) {
    pending.delete(failure.runId);
    throw new Error(`${failure.runId}: ${failure.error}`);
  }
}

return completed;
```

每個 `agents_wait` 呼叫接受 1–1000 個執行 ID。它會傳回：

```typescript
type AgentsWaitResult = {
  completed: Array<{
    runId: string;
    status: "done" | "failed" | "killed" | "timeout";
    result: string;
    structured?: unknown;
    schemaError?: string;
    sessionKey: string;
    label?: string;
    usage?: { inputTokens: number; outputTokens: number };
  }>;
  pending: string[];
  errors?: Array<{
    runId: string;
    error: "not_found" | "not_owner";
  }>;
};
```

當任何要求的子項目已完成、至少一個待處理子項目完成、不再有有效的待處理 ID，或呼叫逾時時，呼叫會立即傳回。已完成的記錄具冪等性，因此傳入已完成的執行 ID 會再次傳回其結果。只有產生該收集器的工作階段或其已授權的父層鏈可以等待該收集器。

這是有界長輪詢，而非忙碌狀態迴圈。請持續只傳入剩餘的執行 ID，直到 `pending` 為空。收集器模式支援原生 OpenClaw 子代理程式；不支援 ACP 執行階段、執行緒綁定、可見工作階段或持續性工作階段模式。

## 限制與藍圖

Swarm v1 執行單次收集器子項目；規劃中的 `agents.session()` API 將新增具狀態的多輪工作程式。子項目目前在本機閘道的子代理程式通道上執行；雲端配置規劃為明確的產生選項。儲存的工作流程定義和圖形 DSL 並不屬於 Swarm 目前的發展方向。

## 相關內容

- [Code Mode](/zh-TW/tools/code-mode)，瞭解 QuickJS 客體執行階段和啟用規則
- [子代理程式](/zh-TW/tools/subagents)，瞭解子項目原則、隔離和工作階段行為
- [多代理程式沙箱工具](/zh-TW/tools/multi-agent-sandbox-tools)，瞭解各代理程式的限制
- [工具概覽](/zh-TW/tools)，瞭解工具設定檔和原則路由
