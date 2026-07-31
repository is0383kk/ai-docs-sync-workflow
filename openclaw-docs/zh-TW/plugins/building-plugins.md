---
doc-schema-version: 1
read_when:
    - 你想建立新的 OpenClaw 外掛
    - 你需要一份外掛開發快速入門指南
    - 你正在選擇頻道、供應商、命令列介面後端、工具或掛鉤的文件
sidebarTitle: Getting Started
summary: 幾分鐘內建立你的第一個 OpenClaw 外掛
title: 建置外掛
x-i18n:
    generated_at: "2026-07-26T08:01:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d156ea305e46d3ca311a0b2cfc42e2c4522f6f10eb70cdd5526d9e9fcd7d4ef
    source_path: plugins/building-plugins.md
    workflow: 16
---

外掛可擴充 OpenClaw，而無須變更核心。外掛可新增訊息傳遞
頻道、模型供應商、本機命令列介面後端、代理工具、掛鉤、媒體供應商，
或其他由外掛擁有的功能。

你不需要將外部外掛新增至 OpenClaw 儲存庫。將
套件發布至 [ClawHub](/zh-TW/clawhub)，使用者可透過以下方式安裝：

```bash
openclaw plugins install clawhub:<package-name>
```

在啟動切換期間，未加前綴的套件規格仍會從 npm 安裝。需要透過 ClawHub
解析時，請使用 `clawhub:` 前綴。

## 需求

- Node 22.22.3+、Node 24.15+ 或 Node 25.9+，以及 `npm` 或 `pnpm`。
- TypeScript ESM 模組。
- 若要開發儲存庫內的內建外掛，請複製儲存庫並執行 `pnpm install`。
  原始碼簽出環境中的外掛開發僅支援 pnpm，因為 OpenClaw 會從
  `extensions/*` 工作區套件探索內建外掛。

## 選擇外掛形式

<CardGroup cols={2}>
  <Card title="頻道外掛" icon="messages-square" href="/zh-TW/plugins/sdk-channel-plugins">
    將 OpenClaw 連接至訊息傳遞平台。
  </Card>
  <Card title="供應商外掛" icon="cpu" href="/zh-TW/plugins/sdk-provider-plugins">
    新增模型、媒體、搜尋、擷取、語音或即時供應商。
  </Card>
  <Card title="命令列介面後端外掛" icon="terminal" href="/zh-TW/plugins/cli-backend-plugins">
    透過 OpenClaw 模型備援執行本機 AI 命令列介面。
  </Card>
  <Card title="工具外掛" icon="wrench" href="/zh-TW/plugins/tool-plugins">
    註冊代理工具。
  </Card>
</CardGroup>

## 快速入門

註冊一個必要的代理工具，即可建置最小工具外掛。這是
最精簡且實用的外掛形式，涵蓋套件、資訊清單、進入點及
本機驗證。

<Steps>
  <Step title="建立套件中繼資料">
    <CodeGroup>

```json package.json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "typebox": "1.1.39"
  },
  "peerDependencies": {
    "openclaw": ">=2026.3.24-beta.2"
  },
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2",
      "minGatewayVersion": "2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2",
      "pluginSdkVersion": "2026.3.24-beta.2"
    }
  }
}
```

```json openclaw.plugin.json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds a custom tool to OpenClaw",
  "contracts": {
    "tools": ["my_tool"]
  },
  "activation": {
    "onStartup": true
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

    </CodeGroup>

    已發布的外部外掛應將執行階段進入點指向建置後的 JavaScript
    檔案。完整進入點合約請參閱 [SDK 進入點](/zh-TW/plugins/sdk-entrypoints)。

    每個外掛都需要資訊清單，即使沒有設定也一樣。執行階段工具必須
    出現在 `contracts.tools` 中，讓 OpenClaw 無須
    預先載入每個外掛執行階段即可探索擁有權。請謹慎設定 `activation.onStartup`；
    此範例會在閘道啟動時載入。

    主機信任的外掛介面也受資訊清單管控，且已安裝的外掛必須明確
    宣告：`api.registerAgentToolResultMiddleware(...)`
    需要在 `contracts.agentToolResultMiddleware` 中列出每個目標執行階段，
    而 `api.registerTrustedToolPolicy(...)` 需要在
    `contracts.trustedToolPolicies` 中列出每個原則 ID。這些宣告可讓安裝階段的
    檢查與執行階段註冊保持一致。

    每個資訊清單欄位的說明請參閱[外掛資訊清單](/zh-TW/plugins/manifest)。

  </Step>

  <Step title="註冊工具">
    ```typescript index.ts
    import { Type } from "typebox";
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

    export default definePluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Adds a custom tool to OpenClaw",
      register(api) {
        api.registerTool({
          name: "my_tool",
          description: "Echo one input value",
          parameters: Type.Object({ input: Type.String() }),
          outputSchema: Type.Object(
            { input: Type.String() },
            { additionalProperties: false },
          ),
          async execute(_id, params) {
            const details = { input: params.input };
            return {
              content: [{ type: "text", text: `Got: ${params.input}` }],
              details,
            };
          },
        });
      },
    });
    ```

    非頻道外掛請使用 `definePluginEntry`。頻道外掛則改用
    `openclaw/plugin-sdk/core` 中的 `defineChannelPluginEntry`。

  </Step>

  <Step title="測試執行階段">
    對於已安裝或外部外掛，請檢查已載入的執行階段：

    ```bash
    openclaw plugins inspect my-plugin --runtime --json
    ```

    如果外掛註冊了命令列介面命令，也請執行該命令並確認
    輸出，例如 `openclaw demo-plugin ping`。

    對於此儲存庫中的內建外掛，OpenClaw 會從 `extensions/*` 工作區
    探索原始碼簽出的外掛套件。請執行最接近的針對性
    測試：

    ```bash
    pnpm test extensions/my-plugin/
    pnpm check
    ```

  </Step>

  <Step title="測試套件安裝">
    發布可封裝的外掛前，請測試使用者實際取得的相同安裝形式。
    首先新增建置步驟，將 `openclaw.extensions` 等執行階段進入點
    指向 `./dist/index.js` 之類的建置後 JavaScript，並確保
    `npm pack` 包含該 `dist/` 輸出。TypeScript 原始碼進入點
    僅適用於原始碼簽出及本機開發路徑。

    接著封裝外掛，並使用 `npm-pack:` 安裝 tarball：

    ```bash
    npm pack --pack-destination /tmp
    openclaw plugins install npm-pack:/tmp/<plugin-package>.tgz --force
    openclaw plugins inspect my-plugin --runtime --json
    ```

    `npm-pack:` 使用 OpenClaw 管理的每外掛 npm 專案，因此能找出
    原始碼簽出測試可能掩蓋的執行階段相依性錯誤。它能證明
    套件及相依性形式，但無法證明與目錄連結的官方信任狀態。
    執行階段匯入項目必須位於 `dependencies` 或 `optionalDependencies`；
    僅留在 `devDependencies` 中的相依性不會安裝至
    受管理的執行階段專案。

    請勿將原始封存檔／路徑安裝作為官方或具特殊權限外掛行為的最終
    驗證。原始碼適合用於本機偵錯，但無法證明與 npm 或 ClawHub 安裝
    相同的相依性路徑。如果你的外掛依賴受信任的官方外掛狀態，請透過
    目錄支援的官方安裝，或可記錄官方信任狀態的已發布套件路徑，
    加入第二項驗證。安裝根目錄與相依性擁有權的詳細資訊請參閱
    [外掛相依性解析](/zh-TW/plugins/dependency-resolution)。

  </Step>

  <Step title="發布">
    發布前請驗證套件：

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    ```

    標準 ClawHub 套件片段位於 `docs/snippets/plugin-publish/`。

  </Step>

  <Step title="安裝">
    透過 ClawHub 安裝已發布的套件：

    ```bash
    openclaw plugins install clawhub:your-org/your-plugin
    ```

  </Step>
</Steps>

<a id="registering-agent-tools"></a>

## 註冊工具

工具可以是必要或選用。啟用外掛時，必要工具一律可用。選用工具需要
使用者明確選擇加入，OpenClaw 才會載入擁有該工具的外掛執行階段。

工具工廠會收到受信任的執行階段內容，其中包括 `deliveryContext`、
可用時目前平台對話的 `nativeChannelId`，以及
`requesterSenderId`。

```typescript
register(api) {
  api.registerTool(
    {
      name: "workflow_tool",
      description: "Run a workflow",
      parameters: Type.Object({ pipeline: Type.String() }),
      outputSchema: Type.Object(
        { pipeline: Type.String() },
        { additionalProperties: false },
      ),
      async execute(_id, params) {
        return {
          content: [{ type: "text", text: params.pipeline }],
          details: { pipeline: params.pipeline },
        };
      },
    },
    { optional: true },
  );
}
```

`outputSchema` 為選用。它描述 [Code Mode](/zh-TW/tools/code-mode) 與
[工具搜尋](/zh-TW/tools/tool-search)所使用的結構化 `details` 值。目錄
呼叫會在執行前拒絕無效的結構描述，並在工具掛鉤後驗證最終值。
對於沒有穩定 JSON 結果的工具，請省略此項。完整合約請參閱
[工具外掛](/zh-TW/plugins/tool-plugins#output-contracts)。

每個使用 `api.registerTool(...)` 註冊的工具也必須在
外掛資訊清單中宣告：

```json
{
  "contracts": {
    "tools": ["workflow_tool"]
  },
  "toolMetadata": {
    "workflow_tool": {
      "optional": true
    }
  }
}
```

使用者可透過 `tools.allow` 選擇加入：

```json5
{
  tools: { allow: ["workflow_tool"] }, // or ["my-plugin"] for every tool from one plugin
}
```

選用工具控制是否將工具公開給模型。當工具或掛鉤應在模型選取後、
動作執行前要求核准時，請使用
[外掛權限要求](/zh-TW/plugins/plugin-permission-requests)。

對於具有副作用、使用不常見二進位檔，或不應預設公開的功能，
請使用選用工具。工具名稱不得與核心工具名稱衝突；衝突項目會被略過，
並在外掛診斷中回報。格式錯誤的註冊也會以相同方式略過並回報：
缺少非空白的 `name`、`execute` 不是函式，或工具描述元缺少
`parameters` 物件。

工具工廠會收到執行階段提供的內容物件。當工具需要記錄、顯示目前
回合的作用中模型，或根據該模型調整行為時，請使用 `ctx.activeModel`；
其中可能包含 `provider`、`modelId` 和 `modelRef`。請將其視為
資訊性執行階段中繼資料，而不是防範本機操作者、已安裝外掛程式碼或
修改版 OpenClaw 執行階段的安全邊界。敏感的本機工具仍應要求明確的
外掛或操作者選擇加入，且在作用中模型中繼資料缺失或不適用時，
應採取拒絕執行的安全預設。

資訊清單負責宣告擁有權與探索；執行時仍會呼叫即時註冊的工具實作。
請讓 `toolMetadata.<tool>.optional: true` 與 `api.registerTool(..., { optional: true })` 保持一致，
讓 OpenClaw 在該工具明確列入允許清單前，無須載入
該外掛執行階段。

## 匯入慣例

從聚焦的 SDK 子路徑匯入：

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
```

在你的外掛套件中，內部匯入請使用 `api.ts` 和
`runtime-api.ts` 等本機彙整檔。請勿透過 SDK 路徑匯入自己的外掛。
除非介面確實通用，否則供應商特定的輔助函式應留在供應商套件中。

自訂閘道 RPC 方法屬於進階進入點。請使用外掛專屬前綴；像是
`config.*`、`exec.approvals.*`、`operator.admin.*`、`wizard.*` 和 `update.*`
等核心管理命名空間維持保留，並解析為 `operator.admin`。
`openclaw/plugin-sdk/gateway-method-runtime` 橋接器保留給宣告 `contracts.gatewayMethodDispatch: ["authenticated-request"]` 的外掛 HTTP
路由使用。

完整匯入對應請參閱[外掛 SDK 概觀](/zh-TW/plugins/sdk-overview)。

OpenClaw SDK 相容性欄位帶有 TypeScript `@deprecated` 註解，
編輯器會將其顯示為遷移警告。若要在建置時強制執行，請啟用具備型別感知能力的規則，例如
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/)。
Oxlint 不具備型別感知能力，因此無法強制執行這些註解。

## 提交前檢查清單

<Check>**package.json** 具有正確的 `openclaw` 中繼資料</Check>
<Check>**openclaw.plugin.json** 資訊清單存在且有效</Check>
<Check>進入點使用 `defineChannelPluginEntry` 或 `definePluginEntry`</Check>
<Check>所有匯入均使用明確的 `plugin-sdk/<subpath>` 路徑</Check>
<Check>內部匯入使用本機模組，而非 SDK 自我匯入</Check>
<Check>測試通過（`pnpm test <bundled-plugin-root>/my-plugin/`）</Check>
<Check>`pnpm check` 通過（存放於儲存庫內的外掛）</Check>

## 針對 Beta 版本進行測試

1. 關注 [openclaw/openclaw](https://github.com/openclaw/openclaw/releases) 發布版本（`Watch` > `Releases`）。Beta 標籤的格式如 `v2026.3.N-beta.1`。你也可以在 X 上追蹤 [@openclaw](https://x.com/openclaw)，以取得發布公告。
2. Beta 標籤出現後，請立即針對該標籤測試你的外掛。距離穩定版發布通常只有幾個小時的時間。
3. 測試後，請在 `plugin-forum` Discord 頻道（[discord.gg/clawd](https://discord.gg/clawd)）中你的外掛討論串內發文，註明 `all good` 或說明發生的問題。如果尚無討論串，請建立一個。
4. 如果發生問題，請建立或更新標題為 `Beta blocker: <plugin-name> - <summary>` 的議題，並套用 `beta-blocker` 標籤。在你的討論串中附上該議題的連結。
5. 向 `main` 開啟標題為 `fix(<plugin-id>): beta blocker - <summary>` 的 PR，並在 PR 和 Discord 討論串中附上該議題的連結。貢獻者無法為 PR 加上標籤，因此標題是向維護者與自動化系統傳達 PR 狀態的訊號。有 PR 的阻斷問題會被合併；沒有 PR 的阻斷問題仍可能隨版本發布。
6. 未收到回應即表示一切正常。錯過此時間窗口通常表示你的修正會在下一個週期合併。

## 後續步驟

<CardGroup cols={2}>
  <Card title="頻道外掛" icon="messages-square" href="/zh-TW/plugins/sdk-channel-plugins">
    建立訊息頻道外掛
  </Card>
  <Card title="供應商外掛" icon="cpu" href="/zh-TW/plugins/sdk-provider-plugins">
    建立模型供應商外掛
  </Card>
  <Card title="命令列介面後端外掛" icon="terminal" href="/zh-TW/plugins/cli-backend-plugins">
    註冊本機 AI 命令列介面後端
  </Card>
  <Card title="SDK 概覽" icon="book-open" href="/zh-TW/plugins/sdk-overview">
    匯入對應表與註冊 API 參考資料
  </Card>
  <Card title="執行階段輔助工具" icon="settings" href="/zh-TW/plugins/sdk-runtime">
    透過 api.runtime 使用 TTS、搜尋與子代理
  </Card>
  <Card title="測試" icon="test-tubes" href="/zh-TW/plugins/sdk-testing">
    測試公用程式與模式
  </Card>
  <Card title="外掛資訊清單" icon="file-json" href="/zh-TW/plugins/manifest">
    完整的資訊清單結構描述參考資料
  </Card>
</CardGroup>

## 相關內容

- [外掛鉤子](/plugins/hooks)
- [外掛架構](/zh-TW/plugins/architecture)
