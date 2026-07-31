---
read_when:
    - 你想要建立一個只新增代理工具的簡易 OpenClaw 外掛
    - 你想要使用 defineToolPlugin，而不是手動編寫外掛資訊清單中繼資料
    - 你需要建構、產生、驗證、測試或發布僅含工具的外掛
sidebarTitle: Tool Plugins
summary: 使用 defineToolPlugin 與 openclaw plugins init/build/validate 建構簡單的型別化代理工具
title: 工具外掛
x-i18n:
    generated_at: "2026-07-26T07:52:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac23d15ba79cbdd1d8b8eab7c87007b44af16361b2866b14123e18f816bf4075
    source_path: plugins/tool-plugins.md
    workflow: 16
---

`defineToolPlugin` 會建置一個只新增可由代理呼叫工具的外掛：不含
頻道、模型供應商、掛鉤、服務或設定後端。它會產生 OpenClaw 所需的
資訊清單中繼資料，以便在不載入外掛執行階段程式碼的情況下探索工具。

若要建置供應商、頻道、掛鉤、服務或混合功能外掛，請改從
[建置外掛](/zh-TW/plugins/building-plugins)、[頻道外掛](/zh-TW/plugins/sdk-channel-plugins)
或[供應商外掛](/zh-TW/plugins/sdk-provider-plugins)開始。

## 需求

- Node 22.22.3+、Node 24.15+ 或 Node 25.9+。
- TypeScript ESM 套件輸出。
- `typebox` 位於 `dependencies`（而不只是 `devDependencies`——產生的
  外掛會在執行階段匯入它）。
- `openclaw >=2026.5.17`，即第一個匯出
  `openclaw/plugin-sdk/tool-plugin` 的版本。
- 套件根目錄須發布 `dist/`、`openclaw.plugin.json` 和
  `package.json`。

## 快速開始

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm install
npm run plugin:build
npm run plugin:validate
npm test
```

`plugins init` 會建立以下骨架：

| 檔案                   | 用途                                                           |
| ---------------------- | ----------------------------------------------------------------- |
| `src/index.ts`         | 包含一個 `echo` 工具的 `defineToolPlugin` 進入點                     |
| `src/index.test.ts`    | 斷言工具清單的中繼資料測試                             |
| `tsconfig.json`        | 輸出至 `dist/` 的 NodeNext TypeScript                             |
| `vitest.config.ts`     | `src/**/*.test.ts` 的 Vitest 設定                              |
| `package.json`         | 指令碼、執行階段相依套件、`openclaw.extensions: ["./dist/index.js"]` |
| `openclaw.plugin.json` | 初始工具的已產生資訊清單中繼資料                  |

`npm run plugin:build` 會執行 `npm run build`（tsc），然後執行
`openclaw plugins build --entry ./dist/index.js`。`npm run plugin:validate`
會重新建置並執行 `openclaw plugins validate --entry ./dist/index.js`。
驗證成功時會顯示：

```text
外掛 stock-quotes 有效。
```

`openclaw plugins init <id>` 選項：

| 旗標                 | 預設值            | 效果                                 |
| -------------------- | ------------------ | -------------------------------------- |
| `--directory <path>` | `<id>`             | 輸出目錄                       |
| `--name <name>`      | 採標題大小寫的 `<id>` | 顯示名稱                           |
| `--type <type>`      | `tool`             | 骨架類型：`tool` 或 `provider`    |
| `--force`            | 關閉                | 覆寫現有的輸出目錄 |

## 撰寫工具

`defineToolPlugin` 接受外掛身分、選用的設定結構描述，以及
靜態工具清單。參數和設定型別會從
TypeBox 結構描述推斷。

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "Fetch stock quote snapshots.",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "Quote API key." })),
    baseUrl: Type.Optional(Type.String({ description: "Quote API base URL." })),
  }),
  tools: (tool) => [
    tool({
      name: "stock_quote",
      label: "Stock Quote",
      description: "Fetch a stock quote snapshot.",
      parameters: Type.Object({
        symbol: Type.String({ description: "Ticker symbol, for example OPEN." }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          configured: Type.Boolean(),
          baseUrl: Type.String(),
        },
        { additionalProperties: false },
      ),
      async execute({ symbol }, config, context) {
        context.signal?.throwIfAborted();
        return {
          symbol: symbol.toUpperCase(),
          configured: Boolean(config.apiKey),
          baseUrl: config.baseUrl ?? "https://api.example.com",
        };
      },
    }),
  ],
});
```

工具名稱是穩定的 API。請選擇唯一、全小寫，且
足夠明確的名稱，以免與核心工具或其他外掛衝突。

## 選用工具與工廠工具

當使用者應先明確將工具加入允許清單，工具才會傳送給模型時，請設定 `optional: true`。
`openclaw plugins build` 會寫入相符的
`toolMetadata.<tool>.optional` 資訊清單項目，讓 OpenClaw 無須載入外掛執行階段程式碼，
也能得知該工具為選用工具。

```typescript
tool({
  name: "workflow_run",
  description: "Run an external workflow.",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  execute: ({ goal }) => ({ queued: true, goal }),
});
```

當工具必須先取得執行階段工具情境才能建立時，請使用 `factory`——例如針對特定執行選擇退出、檢查沙箱狀態，或繫結
執行階段輔助函式。即使具體工具是在
執行階段建置，中繼資料仍維持靜態。

```typescript
tool({
  name: "local_workflow",
  description: "Run a local workflow outside sandboxed sessions.",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  factory({ api, toolContext }) {
    if (toolContext.sandboxed) {
      return null;
    }
    return createLocalWorkflowTool(api);
  },
});
```

工廠仍須預先宣告固定的工具名稱。當外掛會動態計算工具名稱，或將工具
與掛鉤、服務、供應商或命令結合時，請直接使用 `definePluginEntry`。

## 傳回值

`defineToolPlugin` 會將純值傳回結果包裝成 OpenClaw 工具結果
格式：

- 當模型應看到該段完全相同的文字時，傳回字串。
- 當你希望模型看到格式化的 JSON，且 OpenClaw 將原始值保留在 `details` 中時，
  傳回與 JSON 相容的值。

```typescript
tool({
  name: "echo_text",
  description: "Echo input text.",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => input,
});
```

```typescript
tool({
  name: "echo_json",
  description: "Echo input as structured JSON.",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => ({ input, length: input.length }),
});
```

需要自訂 `AgentToolResult`，或想重複使用現有的
`api.registerTool` 實作時，請使用工廠工具。

## 輸出合約

當工具會傳回穩定且與 JSON 相容的資料時，請加入 `outputSchema`。它描述的是
儲存在 `AgentToolResult.details` 中的原始值，而不是
`content` 中的格式化文字：

```typescript
tool({
  name: "shipment_list",
  description: "List shipments.",
  parameters: Type.Object({
    buyer: Type.Optional(Type.String()),
  }),
  outputSchema: Type.Array(
    Type.Object(
      {
        id: Type.String(),
        buyer: Type.String(),
        paid: Type.Boolean(),
        tons: Type.Number(),
      },
      { additionalProperties: false },
    ),
  ),
  execute: ({ buyer }) => listShipments(buyer),
});
```

[程式碼模式](/zh-TW/tools/code-mode)和[工具搜尋](/zh-TW/tools/tool-search)會將此
結構描述轉換為有界限的 TypeScript 風格輸出提示。如此一來，模型便能在單一程式中呼叫並
轉換已知結果，而不必再耗費另一輪模型互動來
觀察其形狀。

OpenClaw 會在執行目錄呼叫前編譯結構描述，然後在工具掛鉤之後驗證
最終的 `details` 值，再透過橋接層傳回。
無效的結構描述無法執行工具；結果不符會使已完成的
呼叫失敗。請包含所有不會擲回例外的結果變體，包括結構化錯誤
變體；若結果不穩定，則省略結構描述。請勿在結構描述的說明中放入祕密
或敏感值，因為受信任的輸出中繼資料可能會
對模型可見。
若要取得完整且
精簡的輸出提示，請在物件層使用 `{ additionalProperties: false }`；開放或截斷的結構描述仍可透過
`tools.describe(...)` 使用，但不會被標示為完整的快速索引合約。

工廠工具會在其傳回的具體 `AnyAgentTool` 上宣告 `outputSchema`。
靜態 `tool({ factory })` 宣告不接受獨立的
輸出結構描述，因為它可能與執行階段工具產生差異。

## 設定

`configSchema` 為選用。省略時，OpenClaw 會套用嚴格的空物件
結構描述；產生的資訊清單仍會包含 `configSchema`。

```typescript
export default defineToolPlugin({
  id: "no-config-tools",
  name: "No Config Tools",
  description: "Adds tools that do not need configuration.",
  tools: () => [],
});
```

若有 `configSchema`，第二個 `execute` 引數的型別會從中推斷：

```typescript
const configSchema = Type.Object({
  apiKey: Type.String(),
});

export default defineToolPlugin({
  id: "configured-tools",
  name: "Configured Tools",
  description: "Adds configured tools.",
  configSchema,
  tools: (tool) => [
    tool({
      name: "configured_ping",
      description: "Check whether configuration is available.",
      parameters: Type.Object({}),
      execute: (_params, config) => ({ hasKey: config.apiKey.length > 0 }),
    }),
  ],
});
```

OpenClaw 會從閘道設定中的外掛項目讀取外掛設定。請勿
在原始碼或文件範例中寫死祕密；請依外掛的安全模型使用設定、環境
變數或 SecretRefs。

## 已產生的中繼資料

OpenClaw 必須先讀取外掛資訊清單，才能匯入外掛執行階段程式碼。
`defineToolPlugin` 會為此公開靜態中繼資料，而
`openclaw plugins build` 會將它寫入套件。變更外掛 ID、名稱、說明、設定結構描述、啟用方式或工具
名稱後，請重新執行產生器：

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

單一工具外掛所產生的資訊清單：

```json
{
  "id": "stock-quotes",
  "name": "Stock Quotes",
  "description": "Fetch stock quote snapshots.",
  "version": "0.1.0",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  },
  "activation": {
    "onStartup": true
  },
  "contracts": {
    "tools": ["stock_quote"]
  }
}
```

`contracts.tools` 是重要的探索合約：它會告訴 OpenClaw 每個工具由哪個
外掛擁有，而不必載入所有已安裝外掛的執行階段。過時的
資訊清單可能導致探索時遺漏工具，或將註冊
錯誤歸咎於錯誤的外掛。

## 套件中繼資料

`openclaw plugins build` 也會將 `package.json` 與選定的執行階段
進入點對齊：

```json
{
  "type": "module",
  "files": ["dist", "openclaw.plugin.json", "README.md"],
  "dependencies": {
    "typebox": "^1.1.38"
  },
  "peerDependencies": {
    "openclaw": ">=2026.5.17"
  },
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

請發布建置完成的 JavaScript（`./dist/index.js`），而非 TypeScript 原始碼進入點。
原始碼進入點只適用於工作區本機開發。

## 在 CI 中驗證

當產生的中繼資料過時時，`plugins build --check` 會失敗，但不會重寫檔案：

```bash
npm run build
openclaw plugins build --entry ./dist/index.js --check
openclaw plugins validate --entry ./dist/index.js
npm test
```

OpenClaw SDK 相容性欄位帶有 TypeScript `@deprecated` 註解，
編輯器會將其顯示為遷移警告。若要在 CI 中強制執行，請啟用
具型別感知能力的規則，例如
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/)。
Oxlint 不具型別感知能力，因此無法強制執行這些註解。因此，產生的
`plugins init` 骨架不會新增淘汰項目 lint 設定。

`plugins validate` 會檢查：

- `openclaw.plugin.json` 存在，且可通過一般資訊清單載入器。
- 目前的進入點會匯出 `defineToolPlugin` 中繼資料。
- 產生的資訊清單欄位與進入點中繼資料相符。
- `contracts.tools` 與宣告的工具名稱相符。
- `package.json` 會將 `openclaw.extensions` 指向所選的執行階段進入點。

## 在本機安裝並檢查

從另一個 OpenClaw 簽出目錄或已安裝的命令列介面安裝套件路徑：

```bash
openclaw plugins install ./stock-quotes
openclaw plugins inspect stock-quotes --runtime
```

若要進行封裝後的冒煙測試，請先封裝，再安裝 tarball：

```bash
npm pack
openclaw plugins install npm-pack:./openclaw-plugin-stock-quotes-0.1.0.tgz
openclaw plugins inspect stock-quotes --runtime --json
```

安裝後，請重新啟動或重新載入閘道，並要求代理程式使用該
工具。如果看不到工具，請先檢查外掛執行階段和有效的
工具目錄，再變更程式碼（請參閱[疑難排解](#troubleshooting)）。

## 發布

套件準備就緒後，透過 ClawHub 發布。`clawhub package publish`
接受一個來源：本機資料夾、GitHub 儲存庫（`owner/repo[@ref]`）或
tarball URL。

```bash
clawhub package publish ./stock-quotes --dry-run
clawhub package publish ./stock-quotes
```

使用明確的 ClawHub 定位字串安裝：

```bash
openclaw plugins install clawhub:your-org/stock-quotes
```

在啟動切換期間，單純的 npm 套件規格仍會從 npm 安裝，但
ClawHub 是 OpenClaw 外掛的首選探索與發佈介面。關於擁有者範圍和
版本審查，請參閱 [ClawHub 發布](/zh-TW/clawhub/publishing)。

## 疑難排解

### `plugin entry not found: ./dist/index.js`

所選的進入點檔案不存在。請執行 `npm run build`，然後重新執行
`openclaw plugins build --entry ./dist/index.js` 或
`openclaw plugins validate --entry ./dist/index.js`。

### `plugin entry does not expose defineToolPlugin metadata`

該進入點未匯出由 `defineToolPlugin` 建立的值。請確認
模組的預設匯出是 `defineToolPlugin(...)` 的結果，或透過
`--entry` 傳入正確的進入點。

### `openclaw.plugin.json generated metadata is stale`

資訊清單已不再與進入點中繼資料相符。請執行：

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

請提交 `openclaw.plugin.json` 和 `package.json` 的變更。

### `package.json openclaw.extensions must include ./dist/index.js`

套件中繼資料指向不同的執行階段進入點。請執行
`openclaw plugins build --entry ./dist/index.js`，讓產生器將
套件中繼資料與你打算發佈的進入點對齊。

### `Cannot find package 'typebox'`

建置後的外掛會在執行階段匯入 `typebox`。請將它保留在 `dependencies` 中，
重新安裝、重新建置，並再次執行驗證。

### 安裝後未顯示工具

請依序檢查下列項目：

1. `openclaw plugins inspect <plugin-id> --runtime`
2. `openclaw plugins validate --root <plugin-root> --entry ./dist/index.js`
3. `openclaw.plugin.json` 具有 `contracts.tools`，且其中包含預期的工具名稱。
4. `package.json` 具有 `openclaw.extensions: ["./dist/index.js"]`。
5. 安裝外掛後，已重新啟動或重新載入閘道。

## 另請參閱

- [建置外掛](/zh-TW/plugins/building-plugins)
- [外掛進入點](/zh-TW/plugins/sdk-entrypoints)
- [外掛 SDK 子路徑](/zh-TW/plugins/sdk-subpaths)
- [外掛資訊清單](/zh-TW/plugins/manifest)
- [外掛命令列介面](/zh-TW/cli/plugins)
- [ClawHub 發布](/zh-TW/clawhub/publishing)
