---
read_when:
    - 你想為代理程式執行啟用 OpenClaw 程式碼模式
    - 你需要說明為什麼 Code Mode 與 Codex Code Mode 不同
    - 你正在審查精簡工具合約、QuickJS-WASI 沙箱、TypeScript 轉換或隱藏的工具目錄橋接器
    - 你正在新增或審查內部程式碼模式命名空間登錄整合功能
sidebarTitle: Code Mode
summary: 使用 OpenClaw 程式碼模式，在精簡的 JavaScript 或 TypeScript 工作流程中探索、呼叫及組合大型工具目錄
title: 程式碼模式
x-i18n:
    generated_at: "2026-07-26T07:58:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a21df3bcfb11668da6dde1f7c69adcc284a28dc491c95f95097ce7f41e5c45bf
    source_path: tools/code-mode.md
    workflow: 16
---

Code mode 是一項實驗性、選擇啟用的 OpenClaw 代理執行階段功能。啟用後，模型不再看到每個已啟用工具的結構描述；它只會看到
`exec`、`wait`，以及任何結構化結果無法通過僅支援 JSON 的客體橋接器、只能直接使用的工具。模型會撰寫一小段 JavaScript 或 TypeScript
程式，以搜尋、描述及呼叫隱藏的工具目錄。

本頁說明的是 OpenClaw code mode，而非 Codex Code Mode。這兩項功能名稱相同，也使用相同的控制工具名稱（`exec`、`wait`），但它們是
各自獨立的實作：

- Codex Code Mode 在 Codex 程式設計框架內執行。其 `exec` 工具是
  自由形式文法工具：模型撰寫原始 JavaScript 原始碼（可選擇在開頭加上
  `// @exec: {...}` pragma 行以設定執行選項），並在 Codex 的處理程序內 V8 Code Mode 執行階段中執行。
- OpenClaw code mode 在通用 OpenClaw 代理執行階段中執行，且除非已設定
  `tools.codeMode.enabled: true`，否則維持停用。其 `exec`
  工具接受 JSON `{ code, language }` 承載資料，並在 QuickJS-WASI
  工作執行緒中執行。

兩者都是 JavaScript 執行介面，而非 Shell 命令介面。請將它們視為彼此獨立、實作方式不同，但碰巧公開同名
`exec`/`wait` 工具的功能。

## 功能

- 模型可見的工具清單會變成 `exec`、`wait`，以及任何只能直接使用的工具，
  例如 `computer`，或影像結果無法通過客體橋接器的原生視覺
  `image` 載入器。
- `exec` 會在隔離的 QuickJS-WASI 工作執行緒中評估模型產生的 JavaScript 或 TypeScript。
- 每個符合目錄資格的已啟用工具（OpenClaw 核心、外掛、MCP、用戶端）都會從模型的獨立工具清單中隱藏，並透過
  `ALL_TOOLS` 和 `tools` 在客體程式內公開。
- `exec` 說明包含受限的快速索引，其中列出確切的 OpenClaw／外掛
  目錄 ID、精簡的輸入提示，以及當受信任工具提供輸出結構描述時的精簡宣告輸出提示。它會省略說明、完整結構描述、
  MCP 項目和溢出的項目；客體端目錄查詢仍是備援機制。
- 客體程式碼會搜尋隱藏目錄、描述工具的結構描述，並透過一般代理回合所使用的相同執行路徑
  呼叫工具（政策、核准、掛鉤與遙測仍全數適用）。
- MCP 工具會歸入 `MCP` 命名空間；在 code mode 中，這是唯一支援的呼叫方式。
- 當巢狀工具呼叫仍在等待時，`wait` 會繼續已暫停的 code mode 執行。

Code mode 只會變更面向模型的協調介面。它不會取代工具、外掛工具、MCP 工具、驗證、核准政策、頻道行為或模型選擇。

## 使用理由

- 更小的提示介面：供應商會取得兩個控制工具、受限的原生工具索引，以及少數必要的直接工具，而非數十或數百個
  完整工具結構描述。
- 更佳的協調能力：模型可以在單一程式碼儲存格內使用迴圈、聯結、小型轉換、條件邏輯與平行巢狀工具呼叫。
- 減少模型往返次數：宣告的輸出合約可讓模型在單一 `exec` 中呼叫並轉換工具結果；
  未知輸出則仍會優先傳回原始結果。
- 不受供應商限制：適用於 OpenClaw、外掛、MCP 和用戶端工具，且不依賴供應商原生的程式碼執行功能。
- 故障時關閉：若已啟用 code mode，但 QuickJS-WASI 執行階段無法使用，該次執行會失敗，而不會無提示地退回廣泛公開直接工具。

這最適合具有大量已啟用工具目錄的代理，或模型必須在回答前搜尋、組合及呼叫多個工具的工作流程。

若工具目錄較小，或模型無法可靠地撰寫短程式，請繼續直接公開工具。若想使用精簡目錄，但偏好結構化的搜尋／描述／呼叫控制，而非
QuickJS-WASI 客體，請使用[工具搜尋](/zh-TW/tools/tool-search)。

## 快速開始

### 啟用 Code Mode

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

簡寫：

```json5
{
  tools: {
    codeMode: true,
  },
}
```

當省略 `tools.codeMode`、設定為 `false`，或物件中不含 `enabled: true` 時，code mode 會維持停用。

若使用已設定 MCP 伺服器的沙箱代理，也請在沙箱工具政策中允許內建的 MCP 外掛，例如
`tools.sandbox.tools.alsoAllow: ["bundle-mcp"]`。請參閱
[設定－沙箱工具政策內的工具與自訂供應商](/zh-TW/gateway/config-tools#mcp-and-plugin-tools-inside-sandbox-tool-policy)。

設定明確限制以取得更嚴格的界限：

```json5
{
  tools: {
    codeMode: {
      enabled: true,
      timeoutMs: 10000,
      memoryLimitBytes: 67108864,
      maxOutputBytes: 65536,
      maxSnapshotBytes: 10485760,
      maxPendingToolCalls: 16,
      snapshotTtlSeconds: 900,
      searchDefaultLimit: 8,
      maxSearchLimit: 50,
    },
  },
}
```

### 模型的運作方式

對於具有已宣告輸出的工具，例如
`Array<{ id: string; paid: boolean; tons: number }>`，單一客體程式可以
選取、呼叫並轉換它：

```javascript
const [shipmentTool] = await tools.search("列出出貨項目");
const shipments = await tools.callValue(shipmentTool.id, {});
return shipments.filter((shipment) => !shipment.paid && shipment.tons > 10);
```

當快速索引行以 `-> ?` 結尾時，表示輸出形狀未知。第一次
`exec` 必須原封不動地傳回 `await tools.callValue(...)`。之後的 `exec` 可以
轉換觀察到的值。這會多耗用一個模型回合，但能防止模型猜測欄位名稱。

### 驗證作用中的介面

若要在偵錯時確認模型承載資料的形狀，請以針對性記錄功能執行閘道：

```bash
OPENCLAW_DEBUG_CODE_MODE=1 \
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 \
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools \
openclaw gateway
```

當 code mode 作用中時，記錄的面向模型工具名稱應為 `exec` 和
`wait`。若要取得完整且經過遮蔽處理的供應商承載資料，請在短暫的偵錯工作階段中加入
`OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`。

## 使用 Swarm 展開代理

[Swarm](/zh-TW/tools/swarm) 會新增 `agents.run()`、`phase()` 和 `log()` 客體全域變數，
以便從 Code Mode 指令碼協調並行子代理。同時啟用
`tools.codeMode` 和 `tools.swarm`，然後使用一般 JavaScript 控制流程進行
展開、決策閘門和結構化收集。Swarm 是另一個需選擇啟用的閘門；僅啟用 Code Mode 並不會公開 `agents.*` API。

## 技術導覽

本頁其餘部分涵蓋執行階段合約與實作細節，供維護者、正在偵錯工具公開狀況的外掛作者，以及驗證高風險部署的營運人員參考。

## 執行階段狀態

|                     |                                                                                             |
| ------------------- | ------------------------------------------------------------------------------------------- |
| 執行階段            | [`quickjs-wasi`](https://github.com/vercel-labs/quickjs-wasi)                               |
| 預設狀態            | 已停用                                                                                    |
| 穩定性              | 實驗性 OpenClaw 介面（Codex Code Mode 是獨立且穩定的 Codex 框架介面） |
| 目標介面            | 通用 OpenClaw 代理執行                                                                 |
| 安全態勢            | 模型程式碼具有敵意                                                                       |
| 對使用者的承諾      | 啟用 code mode 絕不會無提示地退回廣泛公開直接工具                  |

## 範圍

Code mode 負責已準備執行中面向模型的協調形狀。它不負責模型選擇、頻道行為、驗證、工具政策或工具實作。

範圍內：模型可見的控制／直接工具定義、隱藏工具目錄建構、JavaScript／TypeScript 客體執行、QuickJS-WASI 工作執行緒執行階段、搜尋／描述／呼叫的主機回呼、已暫停客體程式的可繼續狀態、輸出／逾時／記憶體／待處理呼叫／快照限制，以及巢狀工具呼叫的遙測／軌跡投影。

範圍外：供應商原生的遠端程式碼執行、Shell 執行語意、變更現有工具授權、使用者撰寫之指令碼的持久化、客體程式碼中的套件管理員／檔案／網路／模組存取，以及直接重用 Codex Code Mode 內部元件。

供應商擁有的工具（例如遠端 Python 沙箱）屬於獨立工具。請參閱
[程式碼執行](/zh-TW/tools/code-execution)。

## 詞彙

- **Code mode**：OpenClaw 執行階段模式，會隱藏與目錄相容的模型工具，並公開
  `exec`、`wait`，以及必要且只能直接使用的工具。
- **客體執行階段**：評估模型程式碼的 QuickJS-WASI JavaScript VM。
- **主機橋接器**：從客體程式碼回到 OpenClaw 的狹窄 JSON 相容回呼介面。
- **目錄**：套用一般工具政策、外掛、MCP 和用戶端工具解析後，僅限該次執行的有效工具清單。
- **巢狀工具呼叫**：客體程式碼透過主機橋接器進行的工具呼叫。
- **快照**：已序列化並儲存的 QuickJS-WASI VM 狀態，讓 `wait` 可以繼續已暫停的 code mode 執行。

## 設定

`tools.codeMode.enabled` 是啟用閘門；設定其他欄位本身不會啟用此功能。

| 欄位                 | 預設值                        | 限制                                           |
| --------------------- | ------------------------------ | ----------------------------------------------- |
| `enabled`             | `false`                        | 布林值；只有 `true` 會啟用 code mode          |
| `runtime`             | `"quickjs-wasi"`               | 唯一支援的值                            |
| `mode`                | `"only"`                       | 公開控制／直接工具，並將其餘工具納入目錄 |
| `languages`           | `["javascript", "typescript"]` | 可為兩者的任意子集                           |
| `timeoutMs`           | `10000`                        | `100`-`60000`                                   |
| `memoryLimitBytes`    | `67108864`                     | `1048576`-`1073741824`                          |
| `maxOutputBytes`      | `65536`                        | `1024`-`10485760`                               |
| `maxSnapshotBytes`    | `10485760`                     | `1024`-`268435456`                              |
| `maxPendingToolCalls` | `16`                           | `1`-`128`                                       |
| `snapshotTtlSeconds`  | `900`                          | `1`-`86400`                                     |
| `searchDefaultLimit`  | `8`                            | 限制為 `maxSearchLimit`                     |
| `maxSearchLimit`      | `50`                           | `1`-`50`                                        |

若已啟用 code mode，但無法載入 QuickJS-WASI，OpenClaw 會針對該次執行採取故障時關閉策略；它不會無提示地公開一般工具作為備援。

## 啟用

Code mode 會在有效工具政策確定後、最終模型要求組裝前進行評估：

1. 解析代理、模型、提供者、沙箱、頻道、傳送者與執行
   原則。
2. 建立有效的 OpenClaw 工具清單，加入符合資格的外掛、MCP 與
   用戶端工具。
3. 套用允許／拒絕原則。
4. 若 `tools.codeMode.enabled` 為 false，繼續採用一般工具公開方式。
5. 若已啟用，且此次執行的工具處於作用中，則保留必要的僅限直接呼叫
   工具，並將目錄中所有符合資格的有效工具註冊至程式碼模式
   目錄。
6. 從模型可見清單中移除已編入目錄的工具；將 `exec` 與
   `wait` 連同保留的僅限直接呼叫工具一起加入。

刻意不使用任何工具的執行（原始模型呼叫、`disableTools: true`，
或空的 `tools.allow` 清單），即使已設定 `tools.codeMode.enabled: true`，
也不會啟用程式碼模式介面。對單次執行而言，程式碼模式與 OpenClaw 工具
搜尋互斥；若程式碼模式啟用，工具搜尋不會進行
壓縮。

程式碼模式目錄的範圍限於單次執行，且不得洩漏來自其他
代理、工作階段、傳送者或執行的工具。

## 模型可見工具

程式碼模式啟用時，模型會看到 `exec`、`wait` 及任何必要的
僅限直接呼叫工具。所有其他已啟用的工具都會從面向模型的
工具清單中隱藏，並註冊至程式碼模式目錄。

使用 `exec` 進行工具協調、資料聯結、迴圈、平行巢狀呼叫
及結構化轉換。僅當 `exec` 傳回可繼續執行的
`waiting` 結果時，才使用 `wait`。

## `exec`

`exec` 會啟動程式碼模式單元並傳回一個結果。輸入程式碼由模型
產生，必須視為惡意內容。

輸入：

```typescript
type CodeModeExecInput = {
  code?: string;
  command?: string;
  language?: "javascript" | "typescript";
};
```

規則：

- `code` 或 `command` 其中之一不得為空。
- `code` 是文件記載的模型端欄位。
- `command` 可作為與 exec 相容的別名，用於掛鉤原則與
  受信任的重寫（一般 OpenClaw shell exec 工具也使用 `command`
  欄位）；若兩者皆存在，其值必須相符。
- `language` 預設為 `"javascript"`；結構描述將其公開為扁平的
  字串列舉（`"javascript" | "typescript"`），而不是 `oneOf`/`anyOf` 聯集，
  因為部分提供者會拒絕這些形狀。
- 若 `language` 為 `"typescript"`，OpenClaw 會先轉譯再求值。
- `exec` 會拒絕 `import`、`require`、動態 import 與模組載入器
  模式。
- `exec` 絕不會遞迴公開一般 shell 的 `exec` 實作。
- 外層程式碼模式的 `exec` 掛鉤事件會攜帶 `toolKind: "code_mode_exec"` 與
  `toolInputKind: "javascript" | "typescript"`（若已知），讓原則得以
  區分程式碼模式單元與共用相同工具名稱、類似 shell 的 `exec` 呼叫。

結果：

```typescript
type CodeModeResult = CodeModeCompletedResult | CodeModeWaitingResult | CodeModeFailedResult;

type CodeModeCompletedResult = {
  status: "completed";
  value: unknown;
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};

type CodeModeWaitingResult = {
  status: "waiting";
  runId: string;
  reason: "pending_tools" | "yield";
  pendingToolCalls?: CodeModePendingToolCall[];
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};

type CodeModeFailedResult = {
  status: "failed";
  error: string;
  code?: CodeModeErrorCode;
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};
```

當客體暫停並保留仍需模型可見接續操作的可恢復狀態時，
`exec` 會傳回 `waiting`——例如明確的 `yield_control(...)`，
或未能在 exec 截止時間內解析完成的橋接工具呼叫。結果會包含供
`wait` 使用的 `runId`。橋接工具呼叫——`tools.search`/`describe`/
`call` 與命名空間呼叫（包括 MCP 命名空間呼叫）——只要能在截止時間內解析完成，
就會在同一個 `exec`/`wait` 呼叫內自動排空，因此，等待多個工具的
精簡程式碼區塊可在一次模型
回合內執行完成，而不必每次 await 都強制進行一次模型工具呼叫。可安全重新啟動的執行絕不會
自動排空；其待處理工作仍會通過可安全重播檢查。

僅當客體 VM 沒有待處理工作，且經過 OpenClaw 的輸出配接器處理後，
最終值與 JSON 相容時，`exec` 才會傳回 `completed`。

## `wait`

`wait` 會繼續執行已暫停的程式碼模式 VM。

輸入：

```typescript
type CodeModeWaitInput = {
  runId: string;
};
```

輸出與 `exec` 傳回的 `CodeModeResult` 聯集相同。

之所以提供 `wait`，是因為巢狀 OpenClaw 工具可能速度緩慢、需要互動、
受核准機制管制，或串流部分更新；當主機等待外部工作時，模型不應需要持續開啟一個
長時間執行的 `exec` 呼叫。

QuickJS-WASI 的快照／還原是恢復執行機制：

1. `exec` 會執行程式碼求值，直到完成、失敗或暫停。
2. 暫停時，OpenClaw 會擷取 QuickJS VM 快照，並記錄待處理的主機
   工作。
3. 待處理工作完成後，`wait` 會還原 VM 快照，並
   使用穩定名稱重新註冊主機回呼。
4. OpenClaw 會將巢狀工具結果傳入已還原的 VM，並排空
   QuickJS 待處理工作。
5. `wait` 會傳回 `completed`、`failed` 或另一個 `waiting` 結果。

快照屬於執行階段狀態，而非使用者成品：它們只存在於
行程內對應表中（不會寫入資料庫或磁碟）、具有大小限制、會過期，且
範圍限於建立它們的執行與工作階段。

在下列情況下，`wait` 會失敗（傳回 `failed` 結果）：

- `runId` 未知，或其快照已過期。
- 呼叫者與已暫停的執行不在相同的執行／工作階段範圍內。
- 該 `runId` 已有一個 `wait` 正在進行。
- QuickJS-WASI 還原失敗。
- 恢復執行會超過 `maxOutputBytes` 或 `maxSnapshotBytes`。

## 客體執行階段 API

```typescript
declare const ALL_TOOLS: ToolCatalogEntry[];
declare const tools: ToolCatalog;
declare const MCP: Record<string, unknown>;
declare const namespaces: Record<string, unknown>;

declare function text(value: unknown): void;
declare function json(value: unknown): void;
declare function yield_control(reason?: string): Promise<void>;
```

`ALL_TOOLS` 是執行範圍目錄的精簡中繼資料；預設不包含
完整結構描述。模型可見的 `exec` 說明也包含
有限且具確定性的 OpenClaw／外掛精確 ID 子集、精簡輸入
提示，以及受信任的宣告輸出提示。說明仍採延遲載入，以免
惡意目錄文字引導模型。當該索引省略某個工具時，
請讀取 `ALL_TOOLS`，或在客體程式內呼叫 `tools.search(...)`。

每行快速索引中的箭頭描述 `tools.callValue(...)` 值。
`-> Array<{ id: string }>` 是宣告的輸出提示；`-> ?` 表示輸出未知。
對於未知輸出，仍應以原始值優先：原樣傳回該值、觀察其內容，然後
在後續的 `exec` 中篩選或對應，而不是猜測欄位名稱。當已宣告輸出的讀取操作
提供資料給最終的 `-> ?` 呼叫時，此原則同樣適用：請原樣傳回該
呼叫的原始值，不要將其包裝成要求的答案形狀。

```typescript
type ToolCatalogEntry = {
  id: string;
  name: string;
  label?: string;
  description: string;
  source: "openclaw" | "mcp" | "client";
  sourceName?: string;
  input: string;
  output?: string;
};
```

`input` 是適用於常見情況、長度受限且採 TypeScript 風格的簽章。若仍需要
精確的完整結構描述，請使用 `tools.describe(...)`。遠端 MCP
與用戶端項目使用 `input: "unknown"`，使其不受信任的結構描述維持
延遲載入，直到 `describe`。僅當完整的精簡提示衍生自受信任的 OpenClaw 核心
或外掛 `outputSchema` 時，才會提供
`output`。MCP 與用戶端的輸出結構描述宣告不會提升為
這項受信任的目錄提示。

外掛工具使用 `source: "openclaw"`，並將 `sourceName` 設為所屬的
外掛 ID；不存在獨立的 `"plugin"` 來源值。`source: "mcp"`
僅用於 `sourceName`/`mcp` 中 MCP 項目的中繼資料（且會從
`ALL_TOOLS`/`tools.*` 中濾除，請見下文）。

完整結構描述僅會依需求載入：

```typescript
type ToolCatalogEntryWithSchema = ToolCatalogEntry & {
  parameters: unknown;
  outputSchema?: unknown;
};
```

目錄輔助工具：

```typescript
type ToolCatalog = {
  search(query: string, options?: { limit?: number }): Promise<ToolCatalogEntry[]>;
  describe(id: string): Promise<ToolCatalogEntryWithSchema>;
  callValue(id: string, input?: unknown): Promise<unknown>;
  call(id: string, input?: unknown): Promise<unknown>;
  [safeToolName: string]: unknown;
};
```

僅會為無歧義的安全名稱安裝便利工具函式：

```typescript
const files = await tools.search("read local file");
const fileRead = await tools.describe(files[0].id);
const content = await tools.callValue(fileRead.id, { path: "README.md" });

// If the hidden catalog has an unambiguous `web_search` entry:
const hits = await tools.web_search({ query: "OpenClaw code mode" });
```

`tools.callValue(...)` 會直接傳回一般工具的 JSON `details` 值。
`tools.call(...)` 會保留原始的 `{ tool, result }` 封套，供需要
內容區塊或其他結果中繼資料的呼叫者使用。

## 宣告的輸出合約

OpenClaw 工具可針對置於 `AgentToolResult.details` 中的結構化值宣告
`outputSchema`。這對程式碼模式與工具搜尋很有用；
它並非提供者原生的工具回應結構描述，也不會改變直接工具
公開方式。

對於使用 `defineToolPlugin` 建立的工具，請在
`parameters` 旁宣告結構描述：

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

const Shipment = Type.Object(
  {
    id: Type.String(),
    paid: Type.Boolean(),
    tons: Type.Number(),
  },
  { additionalProperties: false },
);

export default defineToolPlugin({
  id: "shipping",
  name: "Shipping",
  description: "Shipment tools.",
  tools: (tool) => [
    tool({
      name: "shipping_list",
      description: "List shipments.",
      parameters: Type.Object({}),
      outputSchema: Type.Array(Shipment),
      execute: async () => loadShipments(),
    }),
  ],
});
```

對於 `api.registerTool(...)` 或工廠工具，請將相同的 `outputSchema`
屬性放在傳回的 `AnyAgentTool` 物件上。

目前的內建合約包括 `agents_list`、`apply_patch`、
`conversations_list`、`conversations_send`、`conversations_turn`、`edit`、
`openclaw`、`read`、`screen`、
`sessions_history`、`sessions_list`、`sessions_search`、`sessions_send`、
`session_status`、`spawn_task`、`terminal`、`web_fetch`，以及 `web_search`。
完全直通的項目可以重複使用其所屬的協定結構描述，而不必
複製僅供模型使用的合約。例如，對話工具會公開
`conversations.list`、`conversations.send` 與 `conversations.turn` 所使用的相同閘道結果結構描述；
`web_fetch` 擁有工具本機結構描述，其提示會公開穩定的中繼資料、文字、
快取狀態及巢狀溢出中繼資料；`web_search` 則將其精確的正規化結果／回答／錯誤／原始資料
聯集宣告為完整的快速索引提示。檔案系統合約會回傳結構化的
讀取文字、影像、截斷及選用的找不到項目結果；明確的編輯
變更狀態與差異／修補資料；以及套用修補的路徑摘要。當
快速索引宣告這些欄位時，一個單元即可組合探索與傳遞，
不需要額外的檢查回合：

```javascript
const listed = await tools.conversations_list({ query: "build bot" });
const target = listed.conversations.find((item) => item.label === "Build bot");
if (!target) throw new Error("找不到對話");
return await tools.conversations_send({
  conversationRef: target.conversationRef,
  message: "建置完成。",
});
```

巢狀呼叫仍會使用一般的工具政策、掛鉤與核准。如果完整
合約精確但對有界快速索引而言過大，仍可透過
`tools.describe(...)` 取得，而箭頭仍維持為 `-> ?`。

合約規則相當嚴格：

- 描述精確且與 JSON 相容的 `details` 值，而不是已呈現的 `content`
  區塊或供應商封套。
- 包含所有不會擲回例外的成功或錯誤變體。當
  工具沒有穩定的結構化結果時，請省略 `outputSchema`。
- 使用 `{ additionalProperties: false }` 封閉物件層，以形成完整的
  快速索引提示。開放、過大或以其他方式不完整的結構描述仍可
  透過 `tools.describe(...)` 取得，但不支援在單一回合中使用欄位。
- OpenClaw 會先編譯結構描述，再執行工具，然後在一般工具
  掛鉤之後、目錄呼叫回傳之前驗證最終的 `details`。無效的
  結構描述無法執行工具；不相符時會失敗，且不會列印該值。
- 精簡提示具確定性且有界。當精簡提示不足時，
  `tools.describe(...)` 會公開完整的受信任結構描述。
- 已安裝的外掛程式碼本來就是受信任的本機程式碼。遠端 MCP 與用戶端
  中繼資料仍不受信任，且無法選擇加入這些快速索引提示。

如需外掛撰寫的詳細資訊，請參閱[工具外掛](/zh-TW/plugins/tool-plugins#output-contracts)。

在程式碼模式下，無法透過 `tools.callValue(...)`、
`tools.call(...)` 或便利函式呼叫 MCP 目錄項目；這些項目
只能透過產生的 `MCP` 命名空間公開。唯讀的 `API`
虛擬檔案介面會提供 TypeScript 風格的宣告檔案，因此代理程式無須
將 MCP 結構描述加入提示，即可檢查 MCP 簽章：

```typescript
const files = await API.list("mcp");
const githubApi = await API.read("mcp/github.d.ts");

const issue = await MCP.github.createIssue({
  owner: "openclaw",
  repo: "openclaw",
  title: "調查閘道記錄",
});

const snapshot = await MCP.chromeDevtools.takeSnapshot({ output: "markdown" });
const resource = await MCP.docs.resources.read({ uri: "memo://one" });
const prompt = await MCP.docs.prompts.get({
  name: "brief",
  arguments: { topic: "release" },
});
```

`API.read("mcp/<server>.d.ts")` 會回傳從 MCP
工具中繼資料推導出的精簡宣告：

```typescript
type McpToolResult = {
  content?: unknown[];
  structuredContent?: unknown;
  isError?: boolean;
  [key: string]: unknown;
};

declare namespace MCP.github {
  /** 回傳此 TypeScript 風格的 API 標頭。 */
  function $api(toolName?: string, options?: { schema?: boolean }): Promise<McpApiHeader>;

  /**
   * 建立 GitHub 議題。
   * @param owner 儲存庫擁有者
   * @param repo 儲存庫名稱
   * @param title 議題標題
   */
  function createIssue(input: {
    owner: string;
    repo: string;
    title: string;
    body?: string;
  }): Promise<McpToolResult>;
}
```

宣告檔案是虛擬的，不會寫入工作區或狀態
目錄。對於每次程式碼模式的 `exec` 呼叫，OpenClaw 都會建立該次執行範圍的工具
目錄、保留可見的 MCP 項目、呈現 `mcp/index.d.ts`，並為每個
可見的伺服器呈現一個 `mcp/<server>.d.ts`，再將這份小型唯讀表格
注入 QuickJS 工作程式。客體程式碼只能看見 `API` 物件：
`API.list(prefix?)` 會回傳檔案中繼資料，而 `API.read(path)` 會回傳
所選的宣告內容。未知路徑以及 `.`/`..` 區段都會
遭到拒絕。

如此可避免大型 MCP 結構描述進入模型提示：代理程式會從
`exec` 工具說明得知虛擬 API 的存在，只讀取所需的
宣告檔案，然後使用一個物件引數呼叫 `MCP.<server>.<tool>()`。
`MCP.<server>.$api()` 仍可作為程式內單一工具結構描述回應的
行內備援方案。

客體執行階段絕不會直接看見主機物件。輸入與輸出會以
與 JSON 相容的值跨越橋接，並設有明確的大小上限。

## 內部命名空間

內部命名空間讓程式碼模式能使用精簡的領域 API，而不必新增更多
模型可見的工具。由載入器擁有的整合會註冊如
`Issues` 或 `Calendar` 的命名空間；客體程式碼接著會在
QuickJS 程式中呼叫該命名空間，而模型仍只會看見精簡的控制／直接介面。

命名空間目前僅供內部使用。尚無公開的外掛 SDK 命名空間 API：
外部外掛命名空間需要由載入器擁有的合約，確保外掛身分、
已安裝的資訊清單、驗證狀態及快取的目錄描述元不會與
支援該命名空間的外掛工具產生偏差。核心程式碼模式僅負責
沙箱、序列化、目錄閘控與橋接分派。

客體程式碼可以使用直接全域物件或 `namespaces` 對應：

```javascript
const open = await Issues.list({ state: "open" });
const alsoOpen = await namespaces.Issues.list({ state: "open" });
return { count: open.length, alsoCount: alsoOpen.length };
```

### 登錄生命週期

命名空間登錄是處理程序本機的，並以命名空間 ID 為索引鍵：

1. 受信任的載入器會呼叫 `registerCodeModeNamespaceForPlugin(pluginId, registration)`。
2. 程式碼模式會為該次執行建立隱藏的 `ToolSearchRuntime`，並讀取其
   執行範圍目錄。
3. `createCodeModeNamespaceRuntime(ctx, catalog)` 僅保留
   `requiredToolNames` 全部可見且由同一個 `pluginId` 擁有的登錄項目。
4. 每個可見的命名空間都會為目前執行呼叫 `createScope(ctx)`，
   並接收 `agentId`、`sessionKey`、`sessionId`、
   `runId`、設定及中止狀態等執行環境資訊。
5. 範圍資料會序列化為純描述元，並以直接全域物件及
   `namespaces.<globalName>` 的形式注入 QuickJS。
6. 客體呼叫會透過工作程式橋接暫停、在主機上解析命名空間路徑、
   將呼叫對應至已宣告且由外掛擁有的目錄工具，並
   透過 `ToolSearchRuntime.callExactId` 執行該工具。
7. 已就緒的命名空間橋接呼叫會在作用中的
   `exec`/`wait` 呼叫內自動清空；若逾時時命名空間工作仍在等待中，
   或客體明確讓出執行權，`wait` 會稍後恢復相同的命名空間執行階段。
8. 外掛回復或解除安裝時會呼叫
   `clearCodeModeNamespacesForPlugin(pluginId)`，避免過時的全域物件在外掛載入失敗後
   繼續存在。

命名空間呼叫就是目錄工具呼叫：它們使用與
`tools.call(...)` 相同的政策掛鉤、核准、中止處理、遙測、文字記錄投影及
暫停／恢復行為。

### 登錄格式

請從擁有後端工具的整合註冊命名空間。保持範圍精簡，並且只公開
會對應至已宣告目錄工具的領域動詞。

```typescript
import {
  createCodeModeNamespaceTool,
  registerCodeModeNamespaceForPlugin,
} from "../agents/code-mode-namespaces.js";

const pluginId = "github";

registerCodeModeNamespaceForPlugin(pluginId, {
  id: "github-issues",
  globalName: "Issues",
  description: "目前儲存庫的 GitHub 議題輔助工具。",
  requiredToolNames: ["github_list_issues", "github_update_issue"],
  prompt: "使用 Issues.list(params) 與 Issues.update(number, patch)。",
  createScope: (ctx) => ({
    repository: ctx.config,
    list: createCodeModeNamespaceTool("github_list_issues", ([params]) => params ?? {}),
    update: createCodeModeNamespaceTool("github_update_issue", ([number, patch]) => ({
      number,
      patch,
    })),
  }),
});
```

`createCodeModeNamespaceTool(toolName, inputMapper)` 會將範圍成員標記為
可呼叫的命名空間函式。選用的 `inputMapper` 會接收客體
引數，並回傳後端目錄工具的輸入物件；若未提供，
則使用第一個客體引數，省略時則使用 `{}`。

原始主機函式會在客體程式碼執行前遭到拒絕：

```typescript
createScope: () => ({
  // 錯誤：這會繞過目錄工具生命週期，因此將遭到拒絕。
  list: async () => githubClient.listIssues(),
});
```

### 所有權與可見性

命名空間所有權會繫結至登錄呼叫者的 `pluginId`。
`requiredToolNames` 同時是可見性閘門與所有權檢查：

- 每個必要工具都必須存在於執行目錄中
- 每個必要工具都必須具有 `sourceName === pluginId`
- 任何必要工具缺少或由其他外掛擁有時，
  都會隱藏該命名空間
- 每個可呼叫路徑都只能以 `requiredToolNames` 中所列的工具為目標

這可防止其他外掛透過註冊同名工具來公開命名空間，
並使命名空間與一般代理程式政策保持一致：若該次執行無法看見
後端工具，也就無法看見命名空間。

例如，GitHub 命名空間應位於 GitHub 所擁有的外掛之後，該外掛
負責 GitHub 驗證、REST/GraphQL 用戶端、速率限制、寫入核准及
測試。核心程式碼模式不應內嵌 GitHub 專用 API、權杖處理
或供應商政策。

### 範圍序列化規則

`createScope(ctx)` 可以回傳包含與 JSON 相容的
值、陣列、巢狀物件及 `createCodeModeNamespaceTool(...)` 呼叫
標記的純物件。主機物件絕不會直接進入 QuickJS。

序列化器會拒絕：

- 原始函式
- 循環物件圖
- 不安全的路徑區段：`__proto__`、`constructor`、`prototype`、空白索引鍵，
  或包含內部路徑分隔符號的索引鍵
- 不是 JavaScript 識別碼的 `globalName` 值
- `globalName` 與內建程式碼模式全域物件發生衝突，例如 `tools`、
  `namespaces`、`text`、`json`、`yield_control`、`MCP`、`API`、`ALL_TOOLS` 或
  `__openclaw*`

無法序列化為 JSON 的值，會在跨越橋接前轉換為 JSON 安全的備援
值。二進位資料、控制代碼、通訊端、用戶端及
類別執行個體應保留在一般目錄工具之後。

### 提示

只有在命名空間對該次執行可見時，命名空間的 `description` 與選用的 `prompt`
才會附加至模型可見的 `exec` 結構描述。請使用
它們來說明最小且實用的介面：

```typescript
{
  description: "虛構內容製作服務輔助函式。",
  prompt:
    "使用 Fictions.riskAudit()、Fictions.promoteIfReady(id, status) 和 Fictions.unpaidOver(amount)。",
}
```

提示應聚焦於命名空間合約，而非驗證設定、實作歷程或無關的外掛行為。

### 清理

命名空間是在處理程序本機註冊的。當所屬外掛遭停用、解除安裝或回復時，請移除這些命名空間：

```typescript
clearCodeModeNamespacesForPlugin(pluginId);
```

程式碼模式的清理由外掛負責；外掛生命週期結束時，請清除該外掛的命名空間註冊，而不是保留個別命名空間的拆除控制代碼。測試可以呼叫 `clearCodeModeNamespacesForTest()`，以避免註冊在測試案例之間外洩。

### 測試檢查清單

命名空間變更應涵蓋安全邊界與客體行為：

- 僅在後端工具可見時顯示命名空間提示文字
- 來自另一個 `sourceName` 的同名工具不會公開該命名空間
- 拒絕原始作用域函式
- 拒絕偽造的命名空間 ID 與偽造的路徑
- 可呼叫路徑不得指向未宣告的工具
- 巢狀物件與共用參照可正確序列化
- 命名空間呼叫會透過目錄工具執行，並傳回可安全轉換為 JSON 的詳細資料
- 客體程式碼可以攔截失敗
- 暫停的命名空間呼叫會透過 `wait` 繼續
- 外掛回復會清除其所屬的命名空間註冊

命名空間是通用 `tools.search`/`tools.call` 目錄的補充：任意已啟用的 OpenClaw、外掛與用戶端工具應使用該目錄；MCP 工具應使用 `MCP`；其他命名空間則用於由外掛擁有且有文件記載的領域 API，適合以精簡程式碼取代反覆查詢結構描述的情況。

## 輸出 API

- `text(value)` 會將人類可讀的輸出附加至 `output` 陣列。
- `json(value)` 會在與 JSON 相容的序列化後，附加一個結構化輸出項目。
- 客體程式碼最終傳回的值會成為 `completed` 結果中的 `value`。

```typescript
type CodeModeOutput = { type: "text"; text: string } | { type: "json"; value: unknown };
```

規則：輸出順序與客體呼叫順序一致；輸出上限由 `maxOutputBytes` 規定；無法序列化的值會轉換為純文字字串或錯誤；不支援二進位值。圖片與檔案會透過一般 OpenClaw 工具傳輸，而非程式碼模式橋接器。

## 工具目錄

隱藏目錄會依以下順序納入經有效政策篩選後的工具：OpenClaw 核心工具、隨附外掛工具、外部外掛工具、MCP 工具，最後是目前執行作業由用戶端提供的工具。

在單次執行作業內，目錄 ID 維持穩定；若情況允許，在等效工具集之間也具有確定性。實際格式：

```text
<source>:<owner>:<tool-name>
```

其中 `<source>` 為 `openclaw`、`mcp` 或 `client`（外掛工具使用 `openclaw`，並以外掛 ID 作為 `<owner>`；核心工具則使用 `openclaw:core:*`）。
範例：

```text
openclaw:core:message
openclaw:browser:browser_request
mcp:github:create_issue
client:app:select_file
```

目錄會省略程式碼模式控制工具（`exec`、`wait`、`tool_search_code`、`tool_search`、`tool_describe`、`tool_call`）以及僅限直接呼叫的工具。控制工具不得透過目錄遞迴呼叫；僅限直接呼叫的工具仍對模型可見，因為其結構化結果無法跨越 QuickJS 橋接器。

MCP 項目會保留在執行作業範圍的目錄中，讓政策、核准、掛鉤、遙測、逐字稿投影及確切工具 ID 與一般工具執行共用。面向客體的 `ALL_TOOLS`、`tools.search(...)`、`tools.describe(...)`、`tools.callValue(...)` 和 `tools.call(...)` 檢視會省略 MCP 項目。產生的 `MCP.<server>.<tool>({ ...input })` 命名空間會解析回確切的目錄 ID，並透過相同的執行器路徑分派。

## 與工具搜尋的互動

在程式碼模式啟用的執行作業中，程式碼模式會取代 OpenClaw 工具搜尋的模型介面。

當 `tools.codeMode.enabled` 為 true 且程式碼模式啟用時：

- OpenClaw 不會將 `tool_search_code`、`tool_search`、`tool_describe` 或 `tool_call` 公開為模型可見工具。
- 相同的目錄化概念會移至客體執行階段內。
- 客體執行階段會接收精簡的 `ALL_TOOLS` 中繼資料，以及供非 MCP 工具使用的搜尋／描述／呼叫輔助函式。
- MCP 呼叫會使用產生的 `MCP` 命名空間及其 `$api()` 標頭，而非 `tools.call(...)`。
- 巢狀呼叫會透過工具搜尋所使用的相同 OpenClaw 執行器路徑分派。

請參閱[工具搜尋](/zh-TW/tools/tool-search)，以瞭解程式碼模式在啟用的執行作業中所取代的 OpenClaw 精簡目錄橋接器。

## 工具名稱與衝突

模型可見的 `exec` 工具是程式碼模式工具。若已啟用一般 OpenClaw shell `exec` 工具，該工具會對模型隱藏，並像其他工具一樣納入目錄。

在客體執行階段內：

- 若政策允許，`tools.call("openclaw:core:exec", input)` 可以呼叫 shell exec 工具。
- 僅當 shell exec 目錄項目具有明確且安全的名稱時，才會安裝 `tools.exec(...)`。
- 程式碼模式的 `exec` 工具永遠無法透過 `tools` 遞迴使用。

若兩個工具正規化為相同的安全便利名稱，OpenClaw 會省略該便利函式，並要求使用 `tools.call(id, input)`。

## 巢狀工具執行

每次巢狀工具呼叫都會跨越主機橋接器並重新進入 OpenClaw，同時保留下列資訊：作用中的代理程式 ID、工作階段 ID 與金鑰、傳送者與頻道情境、沙箱政策、核准政策、外掛 `before_tool_call` 掛鉤、中止訊號、可用時的串流更新，以及軌跡／稽核事件。

巢狀呼叫會以真實工具呼叫投影至逐字稿中，讓支援套件顯示發生的情況；該投影也會識別上層程式碼模式工具呼叫與巢狀工具 ID。

平行巢狀呼叫最多允許 `maxPendingToolCalls` 個。

## 執行作業與快照生命週期

每個程式碼模式執行作業都會在處理程序內的對應表中追蹤，並以 `runId` 作為索引鍵（不會持久儲存至磁碟或資料庫）。`exec`/`wait` 會傳回三種結果狀態之一：`completed`、`waiting` 或 `failed`。

- `waiting` 結果會儲存 QuickJS 快照、待處理的橋接要求及作用域中繼資料（代理程式執行作業 ID、工作階段 ID／金鑰），直到 `wait` 繼續執行或其到期為止。
- 到期、工作階段不符、執行作業不符，以及未知／已在繼續執行的 `runId` 值不會產生不同的終止狀態；它們會顯示為帶有 `code: "invalid_input"` 的 `failed` 結果，並包含例如 `code mode
run is unavailable or expired.` 或 `code mode run belongs to a different
session.` 的訊息。
- 執行作業的快照一旦確定為 `completed` 或 `failed`，便會從對應表移除；閘道關閉時也會捨棄快照（重新啟動後不會保留任何內容：這是暫時性執行階段狀態）。
- 對於唯讀工作，`exec` 可以設定 `restartSafe: true`。OpenClaw 隨後會在執行前拒絕具有副作用的目錄呼叫與外掛命名空間，並將暫停的結果標記為可安全重播。若重新啟動中斷 `wait`，[重新啟動復原](/zh-TW/gateway/restart-recovery)會根據逐字稿重建該輪，而非還原處理程序本機快照。復原輪本身仍僅限於經稽核的唯讀核心工具，以及明確標示為可安全重播的外掛工具。
- OpenClaw 會限制每個處理程序同時暫停的執行作業數量（64），超過上限時會以 `too many suspended code mode
runs.` 拒絕新的暫停要求。

快照儲存空間受到每次執行作業的 `maxSnapshotBytes`、上述每個處理程序的暫停執行作業上限，以及 `snapshotTtlSeconds` 限制。

## QuickJS-WASI 執行階段

OpenClaw 會在所屬套件中將 `quickjs-wasi` 載入為直接相依套件；不會依賴為無關相依套件安裝的遞移副本。

執行階段職責：編譯／載入 QuickJS-WASI WebAssembly 模組；為每次程式碼模式執行作業或繼續執行建立一個隔離的 VM；以穩定名稱註冊主機回呼；設定記憶體與中斷限制；求值 JavaScript；清空待處理工作；建立已暫停 VM 狀態的快照；為 `wait` 還原快照；在終止狀態後釋放 VM 控制代碼與快照。

執行階段會在 Node.js 工作執行緒中執行，位於 OpenClaw 主事件迴圈之外。客體的無限迴圈不得無限期阻塞閘道處理程序；工作執行緒的中斷處理常式會強制執行實際經過時間逾時，且不依賴客體程式碼配合。

## TypeScript

TypeScript 支援僅為原始碼轉換：接受的輸入是一個 TypeScript 程式碼字串；輸出則是由 QuickJS-WASI 求值的 JavaScript 字串。不會進行型別檢查、模組解析，也沒有 `import`/`require`。診斷資訊會以 `failed` 結果傳回。

TypeScript 編譯器僅會針對 TypeScript 儲存格延遲載入；純 JavaScript 儲存格及停用的程式碼模式永遠不會載入它。

## 安全邊界

模型程式碼應視為惡意。執行階段採用縱深防禦：

- 在主事件迴圈之外的工作執行緒中執行 QuickJS-WASI
- 將 `quickjs-wasi` 載入為直接相依套件，而非透過 Codex 或遞移套件載入
- 客體中沒有檔案系統、網路、子處理程序、模組匯入、環境變數或主機全域物件
- 使用 QuickJS 記憶體與中斷限制，並搭配上層處理程序的實際經過時間逾時
- 強制執行輸出、快照、記錄及待處理呼叫上限
- 透過範圍受限的 JSON 轉接器序列化主機橋接值
- 將主機錯誤轉換為普通客體錯誤，絕不傳遞主機領域物件
- 在逾時、中止、工作階段結束或到期時捨棄快照
- 拒絕遞迴存取 `exec`、`wait` 及工具搜尋控制工具
- 防止便利名稱衝突遮蔽目錄輔助函式

沙箱只是其中一層安全措施；高風險部署可能仍需由操作人員進行作業系統層級的強化。

## 錯誤代碼

```typescript
type CodeModeErrorCode =
  | "invalid_input"
  | "runtime_unavailable"
  | "timeout"
  | "output_limit_exceeded"
  | "snapshot_limit_exceeded"
  | "internal_error";
```

`invalid_input` 涵蓋錯誤的 `exec`/`wait` 引數、已停用的語言、遭拒絕的模組存取、TypeScript 轉換失敗、未知／已到期／作用域不符的 `runId` 值，以及過多的暫停執行作業。`runtime_unavailable` 涵蓋無法啟動或以非零狀態結束的 QuickJS 工作執行緒。

傳回客體的錯誤是普通資料；主機 `Error` 執行個體、堆疊物件、原型及主機函式不會進入 QuickJS。

## 遙測

每個結果的 `telemetry` 欄位會報告：隱藏目錄大小及來源明細（`openclaw`/`mcp`/`client` 計數）、執行作業目錄的累計搜尋／描述／呼叫次數，以及模型可見的工具名稱（`exec`、`wait` 和保留的僅限直接呼叫工具）。

遙測不得包含機密資訊、原始環境值，或超出既有 OpenClaw 軌跡政策範圍的未遮蔽工具輸入。

## 偵錯

當程式碼模式的行為與一般工具執行不同時，請使用針對性的模型傳輸記錄：

```bash
OPENCLAW_DEBUG_CODE_MODE=1 \
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 \
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools \
OPENCLAW_DEBUG_SSE=events \
openclaw gateway
```

若要偵錯承載資料格式，請使用 `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`。
這會記錄經過大小限制與遮蔽處理的模型請求 JSON 快照；請僅在
偵錯時使用，因為提示詞和訊息文字仍可能出現。

若要偵錯串流，請使用 `OPENCLAW_DEBUG_SSE=peek` 記錄前五個
經過遮蔽處理的 SSE 事件。程式碼模式介面啟用後，如果最終供應商
承載資料未恰好包含一個 `exec`、一個 `wait`，以及僅有核准的
僅限直接呼叫工具，程式碼模式也會採取失敗關閉。

## 實作配置

- 設定契約：`tools.codeMode`
- 目錄建構器：將有效工具轉換為精簡項目與 ID 對應表
- 模型介面配接器：以控制工具／直接工具取代可見工具
- QuickJS-WASI 執行階段配接器：載入、求值、建立快照、還原、釋放
- 工作執行緒監督器：逾時、中止、當機隔離
- 橋接配接器：可安全轉換為 JSON 的主機回呼與結果傳遞
- TypeScript 轉換配接器
- 快照儲存區：TTL、大小上限、執行／工作階段範圍
- 巢狀工具呼叫的軌跡投影
- 遙測計數器與診斷資訊

此實作會重用工具搜尋的目錄與執行器概念，但
不會使用 `node:vm` 子項目作為沙箱。

## 驗證檢查清單

程式碼模式的涵蓋範圍應證明：

- 停用設定會維持現有工具公開方式不變
- 不含 `enabled: true` 的物件設定會讓程式碼模式保持停用
- 啟用設定會在該次執行啟用工具時，向
  模型公開 `exec`、`wait`，以及僅有必要的僅限直接呼叫工具
- 原始無工具執行、`disableTools` 和空白允許清單不會觸發
  程式碼模式承載資料強制檢查
- 所有符合目錄資格的有效非 MCP 工具都會出現在 `ALL_TOOLS` 中
- 僅限直接呼叫工具仍對模型可見，且不會出現在 `ALL_TOOLS` 中
- 遭拒絕的工具不會出現在 `ALL_TOOLS` 中
- `tools.search`、`tools.describe`、`tools.callValue` 和 `tools.call` 可用於 OpenClaw 工具
- `API.list("mcp")` 和 `API.read("mcp/<server>.d.ts")` 無須橋接／工具呼叫即可公開 TypeScript 樣式的
  MCP 宣告
- MCP 命名空間 `$api()` 仍可作為結構描述的行內備援
- MCP 命名空間呼叫可針對只有一個物件輸入的可見 MCP 工具運作，而
  直接 MCP 目錄項目不會出現在 `tools.*` 中
- 工具搜尋控制工具會同時從模型介面與
  隱藏目錄中隱藏
- 巢狀呼叫會保留核准與掛鉤行為
- Shell `exec` 對模型隱藏，但在允許時可透過目錄 ID 呼叫
- 遞迴程式碼模式的 `exec` 和 `wait` 無法從客體程式碼呼叫
- TypeScript 輸入會經過轉換與求值，且不會在
  停用或僅使用 JavaScript 的路徑中載入 TypeScript
- `import`、`require`、檔案系統、網路和環境存取皆會失敗
- 無限迴圈會逾時，且無法阻塞閘道
- 記憶體上限錯誤會終止客體 VM
- 已完成及暫停的呼叫都會強制套用輸出與快照上限
- `wait` 會繼續執行暫停的快照並傳回最終值
- 已過期、已中止、工作階段錯誤及未知的 `runId` 值會失敗
- 文字記錄重播與持久化會保留程式碼模式控制呼叫
- 文字記錄與遙測會清楚顯示巢狀工具呼叫

## 端對端測試計畫

變更執行階段時，請將以下項目作為整合或端對端測試執行：

1. 使用 `tools.codeMode.enabled: false` 啟動閘道。
2. 傳送一輪僅含少量直接工具集的代理程式回合。
3. 確認模型可見工具維持不變。
4. 使用 `tools.codeMode.enabled: true` 重新啟動。
5. 傳送一輪包含 OpenClaw、外掛、MCP 和用戶端測試工具的代理程式回合。
6. 確認模型可見工具清單為 `exec`、`wait`，再加上僅有已設定的
   僅限直接呼叫工具。
7. 在 `exec` 中讀取 `ALL_TOOLS`，並確認符合目錄資格的有效測試
   工具存在，而僅限直接呼叫工具不存在。
8. 在 `exec` 中，透過 `tools.search`、
   `tools.describe` 和 `tools.callValue`（或原始 `tools.call`）呼叫 OpenClaw／外掛／用戶端工具。
9. 在 `exec` 中呼叫 `API.list("mcp")` 和 `API.read("mcp/<server>.d.ts")`，並
   確認宣告檔描述可見的 MCP 工具。
10. 在 `exec` 中透過 `MCP.<server>.<tool>({ ...input })` 呼叫 MCP 工具，並
    確認直接 MCP 目錄項目不存在於 `ALL_TOOLS` 和
    `tools.*` 中。
11. 確認遭拒絕的工具不存在，且無法透過猜測的 ID 呼叫。
12. 啟動巢狀工具呼叫，使其在 `exec` 傳回 `waiting` 後完成。
13. 呼叫 `wait`，並確認還原後的 VM 會收到工具結果。
14. 確認最終答案包含還原後產生的輸出。
15. 確認逾時、中止和快照過期會清除執行階段狀態。
16. 匯出軌跡，並確認巢狀呼叫顯示於父層
    程式碼模式呼叫之下。

僅變更此頁面的文件時，仍應執行 `pnpm check:docs`。

## 相關內容

- [群集](/zh-TW/tools/swarm)：從程式碼模式指令碼進行扇出式代理程式協調
- [工具搜尋](/zh-TW/tools/tool-search)
- [代理程式執行階段](/zh-TW/concepts/agent-runtimes)
- [執行工具](/zh-TW/tools/exec)
- [程式碼執行](/zh-TW/tools/code-execution)
