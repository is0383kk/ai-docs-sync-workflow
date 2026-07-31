---
read_when:
    - 你需要 `defineToolPlugin`、`definePluginEntry` 或 `defineChannelPluginEntry` 的確切型別簽章
    - 你想要瞭解註冊模式（完整、設定或命令列介面中繼資料）
    - 你正在查找進入點選項
sidebarTitle: Entry Points
summary: defineToolPlugin、definePluginEntry、defineChannelPluginEntry 與 defineSetupPluginEntry 參考資料
title: 外掛進入點
x-i18n:
    generated_at: "2026-07-26T08:44:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e64fe1d65531fea8f266aa23b73064daf2ed2c5c43af8bb08ea57e347fe566f4
    source_path: plugins/sdk-entrypoints.md
    workflow: 16
---

每個外掛都會匯出一個預設進入點物件。SDK 會為每種進入點形狀提供一個輔助函式：`defineToolPlugin`、`definePluginEntry`、`defineChannelPluginEntry`、`defineSetupPluginEntry`。

<Tip>
  **想查看逐步解說嗎？** 請參閱[工具外掛](/zh-TW/plugins/tool-plugins)、
  [頻道外掛](/zh-TW/plugins/sdk-channel-plugins)或
  [提供者外掛](/zh-TW/plugins/sdk-provider-plugins)的逐步指南。
</Tip>

## 套件進入點

已安裝的外掛會將 `package.json` `openclaw` 欄位同時指向原始碼與
建置後的進入點：

```json
{
  "openclaw": {
    "extensions": ["./src/index.ts"],
    "runtimeExtensions": ["./dist/index.js"],
    "setupEntry": "./src/setup-entry.ts",
    "runtimeSetupEntry": "./dist/setup-entry.js"
  }
}
```

- `extensions` 和 `setupEntry` 是原始碼進入點，用於工作區和 git
  checkout 開發。
- `runtimeExtensions` 和 `runtimeSetupEntry` 是已安裝
  套件的首選：它們可讓 npm 套件略過執行階段 TypeScript 編譯。
- 若有 `runtimeExtensions`，其陣列長度必須與 `extensions` 相同
  （進入點依位置配對）。`runtimeSetupEntry` 需要 `setupEntry`。
- 若宣告了 `runtimeExtensions`/`runtimeSetupEntry` 成品但
  該成品不存在，安裝／探索會因套件封裝錯誤而失敗；OpenClaw 不會
  靜默回退至原始碼。下述原始碼回退僅適用於完全未宣告
  執行階段進入點的情況。
- 若已安裝的套件僅宣告 TypeScript 原始碼進入點，OpenClaw
  會尋找相符的建置後 `dist/*.js`（或 `.mjs`/`.cjs`）對應項目並使用它；
  否則會回退至 TypeScript 原始碼。
- 所有進入點路徑都必須位於外掛套件目錄內。執行階段
  進入點和推斷出的建置後 JavaScript 對應項目，無法讓逸出目錄的 `extensions` 或
  `setupEntry` 原始碼路徑成為有效路徑。

## `defineToolPlugin`

**匯入：** `openclaw/plugin-sdk/tool-plugin`

適用於僅新增代理程式工具的外掛。它可保持原始碼精簡、從 TypeBox 結構描述推斷設定
與工具參數型別、將一般傳回值包裝成
OpenClaw 工具結果格式，並公開靜態中繼資料，供
`openclaw plugins build` 寫入外掛資訊清單（`contracts.tools`、
`configSchema`）。

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "Fetch stock quotes.",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "API key." })),
  }),
  tools: (tool) => [
    tool({
      name: "quote",
      label: "Quote",
      description: "Fetch a quote.",
      parameters: Type.Object({
        symbol: Type.String({ description: "Ticker symbol." }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          hasKey: Type.Boolean(),
        },
        { additionalProperties: false },
      ),
      execute: async ({ symbol }, config) => ({ symbol, hasKey: Boolean(config.apiKey) }),
    }),
  ],
});
```

- `configSchema` 為選用；省略時會使用嚴格的空物件結構描述
  （產生的資訊清單仍會包含 `configSchema`）。
- `execute` 會傳回一般字串或可序列化為 JSON 的值；輔助函式會
  將其包裝成文字工具結果，並將 `details` 設為原始
  （未字串化的）傳回值。
- `outputSchema` 可選擇描述該原始 `details` 值，以供 Code
  Mode 和 Tool Search 使用。目錄呼叫會在執行前拒絕無效的結構描述，
  並在傳回最終值前驗證該值。
- 若需自訂工具結果，`openclaw/plugin-sdk/tool-results` 會匯出
  `textResult` 和 `jsonResult`。
- 工具名稱是靜態的，因此 `openclaw plugins build` 會從已宣告的工具推導出
  `contracts.tools`，無須手動重複名稱。
- 執行階段載入仍採嚴格模式：已安裝的外掛仍需要
  `openclaw.plugin.json` 和 `package.json` `openclaw.extensions`。OpenClaw
  絕不會執行外掛程式碼來推斷缺少的資訊清單資料。

## `definePluginEntry`

**匯入：** `openclaw/plugin-sdk/plugin-entry`

適用於提供者外掛、進階工具外掛、鉤子外掛，以及任何
**不是**訊息頻道的外掛。

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Short summary",
  register(api) {
    api.registerProvider({/* ... */});
    api.registerTool({/* ... */});
  },
});
```

| 欄位                      | 型別                                                             | 必要 | 預設值              |
| ------------------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`                      | `string`                                                         | 是       | -                   |
| `name`                    | `string`                                                         | 是       | -                   |
| `description`             | `string`                                                         | 是       | -                   |
| `kind`                    | `string`（已棄用，請見下文）                                 | 否       | -                   |
| `configSchema`            | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | 否       | 空物件結構描述 |
| `reload`                  | `OpenClawPluginReloadRegistration`                               | 否       | -                   |
| `nodeHostCommands`        | `OpenClawPluginNodeHostCommand[]`                                | 否       | -                   |
| `securityAuditCollectors` | `OpenClawPluginSecurityAuditCollector[]`                         | 否       | -                   |
| `register`                | `(api: OpenClawPluginApi) => void`                               | 是       | -                   |

- `id` 必須與你的 `openclaw.plugin.json` 資訊清單相符。
- 外部工作階段目錄使用
  `openclaw/plugin-sdk/session-catalog` 和
  `api.registerSessionCatalog({ id, label, list, read, continueSession?, archive? })`。
  核心擁有 `sessions.catalog.*` 閘道方法；提供者會傳回主機、
  工作階段和正規化的轉錄內容投影，而不會註冊 RPC。清單提供者應在每個主機
  完成時呼叫選用的 `onHost(host)` 回呼；傳回的主機陣列仍是必要的最終相容性
  快照。
- `kind` 已棄用：請改為在 `openclaw.plugin.json` 資訊清單的 `kind` 欄位中
  宣告互斥插槽（`"memory"` 或
  `"context-engine"`）。執行階段進入點的 `kind` 僅保留作為
  舊版外掛的相容性回退。
- `configSchema` 可以是用於延遲求值的函式。OpenClaw 會在
  第一次存取時解析並記憶該結構描述，因此昂貴的結構描述建構器只會執行
  一次。
- `nodeHostCommands` 描述項可以定義 `isAvailable({ config, env })`。
  傳回 `false` 會從無頭節點的閘道宣告中省略該命令及其功能。
  OpenClaw 會根據節點本機的啟動設定進行評估；命令處理常式在
  叫用時仍應驗證可用性。

## `defineChannelPluginEntry`

**匯入：** `openclaw/plugin-sdk/channel-core`

使用頻道專用接線包裝 `definePluginEntry`：它會自動
呼叫 `api.registerChannel({ plugin })`、公開選用的根說明命令列介面
中繼資料接縫，並根據註冊模式限制 `registerFull`。

```typescript
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineChannelPluginEntry({
  id: "my-channel",
  name: "My Channel",
  description: "Short summary",
  plugin: myChannelPlugin,
  setRuntime: setMyRuntime,
  registerCliMetadata(api) {
    api.registerCli(/* ... */);
  },
  registerFull(api) {
    api.registerGatewayMethod(/* ... */);
  },
});
```

| 欄位                  | 型別                                                             | 必要 | 預設值              |
| --------------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`                  | `string`                                                         | 是       | -                   |
| `name`                | `string`                                                         | 是       | -                   |
| `description`         | `string`                                                         | 是       | -                   |
| `plugin`              | `ChannelPlugin`                                                  | 是       | -                   |
| `configSchema`        | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | 否       | 空物件結構描述 |
| `setRuntime`          | `(runtime: PluginRuntime) => void`                               | 否       | -                   |
| `registerCliMetadata` | `(api: OpenClawPluginApi) => void`                               | 否       | -                   |
| `registerFull`        | `(api: OpenClawPluginApi) => void`                               | 否       | -                   |

回呼會依每種註冊模式執行（完整表格請見
[註冊模式](#registration-mode)）：

- `setRuntime` 會在除 `"cli-metadata"` 和
  `"tool-discovery"` 以外的所有模式下執行。請在此儲存執行階段參照，通常透過
  `createPluginRuntimeStore`。
- `registerCliMetadata` 會針對 `"cli-metadata"`、`"discovery"` 和
  `"full"` 執行。請將其用作頻道所擁有之命令列介面描述項的標準位置，
  讓根說明保持非啟用狀態、探索快照包含靜態
  命令中繼資料，且一般命令列介面註冊仍與完整
  外掛載入相容。
- `registerFull` 僅針對 `"full"` 和 `"tool-discovery"` 執行。對於
  `"tool-discovery"`，它會_取代_頻道註冊執行：OpenClaw
  會完全略過 `registerChannel`/`setRuntime`，並且只呼叫
  `registerFull`，因此頻道在獨立工具探索或執行時所需的任何提供者／工具註冊，
  都必須放在該處，而不能置於一般頻道設定之後。
- 探索註冊不會啟用外掛，但並非不會匯入：OpenClaw 可能會
  對受信任的外掛進入點和頻道外掛模組進行求值，以建構
  快照。頂層匯入不得產生副作用，並應將通訊端、
  用戶端、背景工作程式和服務置於僅限 `"full"` 的路徑之後。
- 與 `definePluginEntry` 相同，`configSchema` 可以是延遲處理的工廠函式；OpenClaw
  會在第一次存取時記憶解析後的結構描述。

命令列介面註冊：

- 對於你想要延遲載入、但不希望從根命令列介面
  剖析樹中消失的外掛自有根命令列介面命令，請使用 `api.registerCli(..., { descriptors: [...] })`。
  描述元名稱必須符合字母、數字、連字號及底線，且以字母或數字開頭；
  OpenClaw 會拒絕其他格式，並在呈現說明前移除描述中的終端控制序列。
  涵蓋註冊器公開的每個頂層命令根。
  單獨使用 `commands` 時，會維持在積極載入的相容性路徑上。
- 對於配對節點功能命令，請使用 `api.registerNodeCliFeature(...)`，使其
  位於 `openclaw nodes` 之下（等同於
  `registerCli(registrar, { parentPath: ["nodes"], ... })`）。
- 對於其他巢狀外掛命令，請新增 `parentPath`，並在傳給註冊器的
  `program` 物件上註冊命令；OpenClaw 會先將其解析為父命令，
  再呼叫外掛。
- 對於頻道外掛，請從 `registerCliMetadata` 註冊命令列介面描述元，
  並讓 `registerFull` 專注於僅限執行階段的工作。
- 如果 `registerFull` 也註冊閘道 RPC 方法，請將其置於
  外掛專用前綴下。保留的核心管理命名空間（`config.*`、
  `exec.approvals.*`、`wizard.*`、`update.*`）一律會強制轉換為
  `operator.admin`。

## `defineSetupPluginEntry`

**匯入：** `openclaw/plugin-sdk/channel-core`

用於輕量的 `setup-entry.ts` 檔案。僅傳回 `{ plugin }`，
不包含執行階段或命令列介面接線。

```typescript
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineSetupPluginEntry(myChannelPlugin);
```

當頻道停用、尚未設定，或啟用延遲載入時，OpenClaw 會載入此項目，
而非完整進入點。請參閱[設定與組態](/zh-TW/plugins/sdk-setup#setup-entry)，
以瞭解這在何種情況下很重要。

請將 `defineSetupPluginEntry(...)` 與範圍精簡的設定輔助函式系列搭配使用：

| 匯入                                | 用途                                                                                                                                                                              |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw/plugin-sdk/setup-runtime` | 執行階段安全的設定輔助函式：`createSetupTranslator`、匯入安全的設定修補配接器、查閱備註輸出、`promptResolvedAllowFrom`、`splitSetupEntries`、委派設定代理 |
| `openclaw/plugin-sdk/channel-setup` | 選用安裝的設定介面                                                                                                                                                                 |
| `openclaw/plugin-sdk/setup-tools`   | 設定／安裝命令列介面、封存及文件輔助函式                                                                                                                                           |

請將大型 SDK、命令列介面註冊及長期運作的執行階段服務保留在
完整進入點中。

將設定與執行階段介面分離的內建工作區頻道，可以改用
`openclaw/plugin-sdk/channel-entry-contract` 中的
`defineBundledChannelSetupEntry(...)`。它能讓設定進入點保留設定安全的外掛／密鑰匯出，
同時仍公開執行階段 setter：

```typescript
import { defineBundledChannelSetupEntry } from "openclaw/plugin-sdk/channel-entry-contract";

export default defineBundledChannelSetupEntry({
  importMetaUrl: import.meta.url,
  plugin: {
    specifier: "./channel-plugin-api.js",
    exportName: "myChannelPlugin",
  },
  runtime: {
    specifier: "./runtime-api.js",
    exportName: "setMyChannelRuntime",
  },
  registerSetupRuntime(api) {
    api.registerHttpRoute({
      path: "/my-channel/events",
      auth: "plugin",
      handler: async (req, res) => {
        /* 設定安全的路由 */
      },
    });
  },
});
```

只有在設定流程確實需要於完整頻道進入點載入前，提供輕量的執行階段
setter 或設定安全的閘道介面時，才使用此方式。
`registerSetupRuntime` 僅會針對 `"setup-runtime"` 載入執行；請將其
限制為僅限組態的路由，或必須在延遲完整啟用前存在的方法。

## 註冊模式

`api.registrationMode` 會告知外掛其載入方式：

| 模式               | 時機                                               | 要註冊的內容                                                                                                              |
| ------------------ | -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `"full"`           | 一般閘道啟動                                     | 所有內容                                                                                                                  |
| `"discovery"`      | 唯讀能力探索                                     | 頻道註冊加上靜態命令列介面描述元；進入點程式碼可以載入，但需略過通訊端、工作處理器、用戶端及服務                                  |
| `"tool-discovery"` | 用於列出或執行特定外掛工具的範圍限定載入           | 僅註冊能力／工具；不啟用頻道                                                                                              |
| `"setup-only"`     | 已停用／未設定的頻道                             | 僅註冊頻道                                                                                                                |
| `"setup-runtime"`  | 可使用執行階段的設定流程                           | 註冊頻道，並僅加入完整進入點載入前所需的輕量執行階段                                                                        |
| `"cli-metadata"`   | 根說明／命令列介面中繼資料擷取                   | 僅命令列介面描述元                                                                                                        |

`defineChannelPluginEntry` 會自動處理此分流。如果你直接對頻道使用
`definePluginEntry`，請自行檢查模式，並記得
`"tool-discovery"` 會略過頻道註冊：

```typescript
register(api) {
  if (
    api.registrationMode === "cli-metadata" ||
    api.registrationMode === "discovery" ||
    api.registrationMode === "full"
  ) {
    api.registerCli(/* ... */);
    if (api.registrationMode === "cli-metadata") return;
  }

  if (api.registrationMode === "tool-discovery") {
    // 僅註冊能力介面（提供者／工具），不註冊頻道。
    return;
  }

  api.registerChannel({ plugin: myPlugin });
  if (api.registrationMode !== "full") return;

  // 僅限執行階段的大型註冊項目
  api.registerService(/* ... */);
}
```

長期運作的服務可透過其服務情境發出小型失效或生命週期事件：

```typescript
api.registerService({
  id: "index-events",
  start(ctx) {
    ctx.gatewayEvents?.emit("changed", { revision: 1 }, { scope: "operator.read" });
  },
});
```

OpenClaw 會將其命名為 `plugin.<plugin-id>.changed`。事件名稱須為單一
小寫片段，承載資料必須是有界的 JSON，而範圍必須為
`operator.read`、`operator.write` 或 `operator.admin`。發射器僅在
服務存續期間存在，並會在停止或啟動失敗後撤銷。請優先使用版本或
失效承載資料，而非完整記錄，讓經授權的用戶端透過外掛的範圍限定
閘道方法重新讀取標準狀態。

探索模式會建立不啟用功能的登錄快照。它仍可能評估外掛進入點及
頻道外掛物件，讓 OpenClaw 能註冊頻道能力及靜態命令列介面描述元。
請將探索期間的模組評估視為可信任但應保持輕量：頂層不得建立網路
用戶端、子程序、監聽器、資料庫連線、背景工作處理器、讀取認證資訊，
或產生其他即時執行階段副作用。

請將 `"setup-runtime"` 視為設定專用的啟動介面必須存在、但不得重新進入
完整內建頻道執行階段的時段。適合的項目包括頻道註冊、設定安全的 HTTP
路由、設定安全的閘道方法，以及委派設定輔助函式。大型背景服務、
命令列介面註冊器及提供者／用戶端 SDK 啟動程序仍應置於
`"full"`。

## 外掛形式

OpenClaw 會依照載入外掛的註冊行為進行分類：

| 形式                  | 說明                                         |
| --------------------- | -------------------------------------------- |
| **plain-capability**  | 一種能力類型（例如僅提供者）                 |
| **hybrid-capability** | 多種能力類型（例如提供者 + 語音）            |
| **hook-only**         | 僅有掛鉤，沒有能力                           |
| **non-capability**    | 有工具／命令／服務，但沒有能力               |

使用 `openclaw plugins inspect <id>` 查看外掛的形式。

## 相關內容

- [SDK 概觀](/zh-TW/plugins/sdk-overview) - 註冊 API 與子路徑參考
- [執行階段輔助函式](/zh-TW/plugins/sdk-runtime) - `api.runtime` 與 `createPluginRuntimeStore`
- [設定與組態](/zh-TW/plugins/sdk-setup) - 資訊清單、設定進入點、延遲載入
- [頻道外掛](/zh-TW/plugins/sdk-channel-plugins) - 建立 `ChannelPlugin` 物件
- [提供者外掛](/zh-TW/plugins/sdk-provider-plugins) - 提供者註冊與掛鉤
