---
read_when:
    - 你正在為外掛新增設定精靈
    - 你需要了解 setup-entry.ts 與 index.ts 之間的差異
    - 你正在定義外掛設定結構描述或 package.json 的 OpenClaw 中繼資料
sidebarTitle: Setup and config
summary: 設定精靈、setup-entry.ts、設定結構描述及 package.json 中繼資料
title: 外掛設定與組態
x-i18n:
    generated_at: "2026-07-26T08:30:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b07e3fa365939fa9c0885b31b7894f5e734313a7deef2297e316956063d97e45
    source_path: plugins/sdk-setup.md
    workflow: 16
---

OpenClaw 外掛封裝（`package.json` 中繼資料）、資訊清單（`openclaw.plugin.json`）、設定進入點及設定結構描述的參考資料。

<Tip>
**想查看逐步解說嗎？** 操作指南涵蓋實際情境中的封裝：[頻道外掛](/zh-TW/plugins/sdk-channel-plugins#step-1-package-and-manifest)和[供應商外掛](/zh-TW/plugins/sdk-provider-plugins#step-1-package-and-manifest)。
</Tip>

## 套件中繼資料

你的 `package.json` 需要一個 `openclaw` 欄位，告知外掛系統此外掛提供的功能：

<Tabs>
  <Tab title="頻道外掛">
    ```json
    {
      "name": "@myorg/openclaw-my-channel",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "my-channel",
          "label": "我的頻道",
          "blurb": "頻道的簡短說明。"
        }
      }
    }
    ```
  </Tab>
  <Tab title="供應商外掛／ClawHub 基準">
    ```json openclaw-clawhub-package.json
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
  </Tab>
</Tabs>

<Note>
在 ClawHub 對外發佈需要 `compat` 和 `build`。標準發佈片段位於 `docs/snippets/plugin-publish/`。
</Note>

### `openclaw` 欄位

<ParamField path="extensions" type="string[]">
  進入點檔案（相對於套件根目錄）。適用於工作區和 git 簽出開發的有效原始碼進入點。
</ParamField>
<ParamField path="runtimeExtensions" type="string[]">
  `extensions` 的已建置 JavaScript 對應檔案，OpenClaw 載入已安裝的 npm 套件時會優先使用。原始碼／建置成品的解析順序請參閱 [SDK 進入點](/zh-TW/plugins/sdk-entrypoints)。
</ParamField>
<ParamField path="setupEntry" type="string">
  輕量的僅設定進入點（選用）。
</ParamField>
<ParamField path="runtimeSetupEntry" type="string">
  `setupEntry` 的已建置 JavaScript 對應檔案。也必須設定 `setupEntry`。
</ParamField>
<ParamField path="plugin" type="object">
  `{ id, label }` 備援外掛身分，用於外掛沒有可據以衍生 id 或標籤的頻道／供應商中繼資料時。
</ParamField>
<ParamField path="channel" type="object">
  用於設定、選擇器、快速入門及狀態介面的頻道目錄中繼資料。
</ParamField>
<ParamField path="install" type="object">
  安裝提示：`npmSpec`、`localPath`、`defaultChoice`、`minHostVersion`、`expectedIntegrity`、`allowInvalidConfigRecovery`、`requiredPlatformPackages`。
</ParamField>
<ParamField path="startup" type="object">
  啟動行為旗標。
</ParamField>
<ParamField path="compat" type="object">
  此外掛支援的 `pluginApi` 版本範圍。對外發佈至 ClawHub 時為必填。
</ParamField>

<Note>
供應商 id（`providers: string[]`）是資訊清單中繼資料，而非套件中繼資料。請在 `openclaw.plugin.json` 中宣告，不要在此處宣告——請參閱[外掛資訊清單](/zh-TW/plugins/manifest)。
</Note>

### `openclaw.channel`

`openclaw.channel` 是低成本的套件中繼資料，可在執行階段載入前供頻道探索和設定介面使用。

### 頻道擁有的設定欄位

頻道外掛應在執行階段程式碼中使用 `defineChannelSetupContract(...)` 定義一次設定欄位，並在 `openclaw.channel.setup.fields` 下發佈相符且可序列化的投影。執行階段定義會推斷外掛本機輸入型別、解析引導式與非互動式值，並避免將頻道專屬鍵納入核心型別。套件中繼資料可讓 `openclaw channels add <channel-id> --help` 和 `openclaw channels add --channel <channel-id> --help` 僅探索所選頻道的選項，而不必載入外掛。

```ts
import { defineChannelSetupContract } from "openclaw/plugin-sdk/channel-setup";

export const setupContract = defineChannelSetupContract({
  fields: {
    endpoint: {
      kind: "string",
      cli: { flags: "--endpoint <url>", description: "服務端點" },
    },
    transport: {
      kind: "choice",
      choices: ["native", "container"],
      cli: { flags: "--transport <kind>", description: "傳輸擁有者" },
    },
  },
  adapter: {
    applyAccountConfig: ({ cfg, input }) => ({
      ...cfg,
      channels: { ...cfg.channels, example: input },
    }),
  },
});
```

```json
{
  "openclaw": {
    "channel": {
      "id": "example",
      "setup": {
        "fields": [
          {
            "key": "endpoint",
            "kind": "string",
            "cli": { "flags": "--endpoint <url>", "description": "服務端點" }
          },
          {
            "key": "transport",
            "kind": "choice",
            "choices": ["native", "container"],
            "cli": { "flags": "--transport <kind>", "description": "傳輸擁有者" }
          }
        ]
      }
    }
  }
}
```

支援的欄位種類為 `string`、`boolean`、`integer`、`string-list` 和 `choice`。認證資訊請使用 `sensitive: true`。每個欄位鍵都必須等於其長格式命令列介面旗標的駝峰式屬性名稱，包括任何否定形式，例如 `--api-token` 對應 `apiToken`。當同時需要肯定與 `--no-*` 形式時，布林欄位可以加入 `cli.negatedFlags`。`channel`、`account` 及帳號顯示用的 `name` 仍屬於共用控制封套。

已發佈的 `setup`/`ChannelSetupInput` 配接器仍可供現有的外部外掛使用。新外掛應公開 `setupContract`；兩者皆存在時，OpenClaw 一律優先使用它。

| 欄位                                  | 型別       | 意義                                                                 |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------------- |
| `id`                                   | `string`   | 標準頻道 id。                                                         |
| `label`                                | `string`   | 主要頻道標籤。                                                        |
| `selectionLabel`                       | `string`   | 需要與 `label` 不同時使用的選擇器／設定標籤。                        |
| `detailLabel`                          | `string`   | 用於資訊更豐富的頻道目錄和狀態介面的次要詳細資料標籤。       |
| `docsPath`                             | `string`   | 設定和選擇連結所使用的文件路徑。                                      |
| `docsLabel`                            | `string`   | 文件連結需要與頻道 id 不同時使用的覆寫標籤。 |
| `blurb`                                | `string`   | 簡短的初始設定／目錄說明。                                         |
| `order`                                | `number`   | 頻道目錄中的排序順序。                                               |
| `aliases`                              | `string[]` | 用於選擇頻道的額外查詢別名。                                   |
| `preferOver`                           | `string[]` | 此頻道應優先於其上的低優先級外掛／頻道 id。                |
| `systemImage`                          | `string`   | 頻道 UI 目錄的選用圖示／系統影像名稱。                      |
| `selectionDocsPrefix`                  | `string`   | 選擇介面中文件連結前的前置文字。                          |
| `selectionDocsOmitLabel`               | `boolean`  | 在選擇文案中直接顯示文件路徑，而非帶有標籤的文件連結。 |
| `selectionExtras`                      | `string[]` | 附加於選擇文案中的額外短字串。                               |
| `markdownCapable`                      | `boolean`  | 將頻道標記為支援 Markdown，以供傳出格式決策使用。      |
| `exposure`                             | `object`   | 控制頻道在設定、已設定清單和文件介面中的可見性。   |
| `quickstartAllowFrom`                  | `boolean`  | 讓此頻道加入標準快速入門 `allowFrom` 設定流程。         |
| `forceAccountBinding`                  | `boolean`  | 即使只有一個帳號，也要求明確繫結帳號。           |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | 解析此頻道的公告目標時，優先使用工作階段查詢。       |
| `setup`                                | `object`   | 用於延遲探索命令列介面選項、可序列化且由頻道擁有的設定欄位。   |

範例：

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "我的頻道",
      "selectionLabel": "我的頻道（自行託管）",
      "detailLabel": "我的頻道機器人",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "以網路鉤子為基礎的自行託管聊天整合。",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "指南：",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` 支援：

- `configured`：將頻道納入已設定／狀態類型的清單介面
- `setup`：將頻道納入互動式設定／配置選擇器
- `docs`：在文件／導覽介面中將頻道標記為公開

### `openclaw.install`

`openclaw.install` 是套件中繼資料，而非資訊清單中繼資料。

| 欄位                         | 類型                                | 意義                                                                              |
| ---------------------------- | ----------------------------------- | --------------------------------------------------------------------------------- |
| `clawhubSpec`                | `string`                            | 用於安裝／更新及新手引導隨選安裝流程的標準 ClawHub 規格。                         |
| `npmSpec`                    | `string`                            | 用於安裝／更新備援流程的標準 npm 規格。                                            |
| `localPath`                  | `string`                            | 本機開發或隨附安裝路徑。                                                           |
| `defaultChoice`              | `"clawhub"` \| `"npm"` \| `"local"` | 有多個來源可用時的偏好安裝來源。                                                   |
| `minHostVersion`             | `string`                            | 支援的最低 OpenClaw 版本，`>=x.y.z` 或 `>=x.y.z-prerelease`。              |
| `expectedIntegrity`          | `string`                            | 鎖定安裝預期的 npm dist 完整性字串，通常為 `sha512-...`。                   |
| `allowInvalidConfigRecovery` | `boolean`                           | 讓隨附外掛重新安裝流程能從特定的過時設定失敗中復原。                               |
| `requiredPlatformPackages`   | `string[]`                          | npm 安裝期間驗證的必要平台專用 npm 別名。                                          |

<AccordionGroup>
  <Accordion title="新手引導行為">
    互動式新手引導會將 `openclaw.install` 用於隨選安裝介面：如果你的外掛在執行階段載入前公開供應商驗證選項或頻道設定／目錄中繼資料，新手引導可以提示從 ClawHub、npm 或本機安裝、安裝或啟用外掛，然後繼續所選流程。ClawHub 選項使用 `clawhubSpec`，且存在時為優先選項；npm 選項需要可信任的目錄中繼資料，並包含登錄檔 `npmSpec`（確切版本與 `expectedIntegrity` 是選用的鎖定值，設定後會在安裝／更新時強制執行）。將「要顯示什麼」保留在 `openclaw.plugin.json`，並將「如何安裝」保留在 `package.json`。
  </Accordion>
  <Accordion title="minHostVersion 強制執行">
    如果設定了 `minHostVersion`，安裝與非隨附的資訊清單登錄檔載入都會強制執行此設定。較舊的主機會略過外部外掛；無效的版本字串會遭拒絕。隨附的原始碼外掛視為與主機簽出版本相同。
  </Accordion>
  <Accordion title="鎖定的 npm 安裝">
    對於鎖定的 npm 安裝，請將確切版本保留在 `npmSpec`，並加入預期的成品完整性：

    ```json
    {
      "openclaw": {
        "install": {
          "npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3",
          "expectedIntegrity": "sha512-REPLACE_WITH_NPM_DIST_INTEGRITY",
          "defaultChoice": "npm"
        }
      }
    }
    ```

  </Accordion>
  <Accordion title="allowInvalidConfigRecovery 範圍">
    `allowInvalidConfigRecovery` 並非損壞設定的一般性略過機制。它僅用於範圍狹窄的隨附外掛復原，讓重新安裝／設定能修復已知的升級殘留問題，例如缺少隨附外掛路徑，或同一外掛存在過時的 `channels.<id>` 項目。如果設定因不相關原因損壞，安裝仍會以封閉方式失敗，並告知操作人員執行 `openclaw doctor --fix`。
  </Accordion>
</AccordionGroup>

### 延後完整載入

頻道外掛可使用以下設定選擇延後載入：

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

啟用後，即使是已設定的頻道，OpenClaw 在開始監聽前的啟動階段也只會載入 `setupEntry`。完整進入點會在閘道開始監聽後載入。

<Warning>
只有在你的 `setupEntry` 會註冊閘道開始監聽前所需的一切項目（頻道註冊、HTTP 路由、閘道方法）時，才啟用延後載入。如果必要的啟動功能由完整進入點擁有，請保留預設行為。
</Warning>

如果你的設定／完整進入點會註冊閘道 RPC 方法，請將它們保留在外掛專用前綴下。保留的核心管理命名空間（`config.*`、`exec.approvals.*`、`wizard.*`、`update.*`）仍由核心擁有，且一律正規化為 `operator.admin`。

## 外掛資訊清單

每個原生外掛都必須在套件根目錄中提供 `openclaw.plugin.json`。OpenClaw 會使用它，在不執行外掛程式碼的情況下驗證設定。

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds My Plugin capabilities to OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook verification secret"
      }
    }
  }
}
```

對於頻道外掛，請加入 `channels`（供應商外掛則加入 `providers`）：

```json
{
  "id": "my-channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

即使外掛沒有設定，也必須提供結構描述。空的結構描述是有效的：

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

如需完整的結構描述參考，請參閱[外掛資訊清單](/zh-TW/plugins/manifest)。

## 發布至 ClawHub

Skills 與外掛套件使用不同的 ClawHub 發布命令。對於外掛套件，請使用套件專用命令：

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

<Note>
`clawhub skill publish <path>` 是用於發布 Skill 資料夾的不同命令，而非外掛套件。請參閱[在 ClawHub 上發布](/zh-TW/clawhub/publishing)。
</Note>

## 設定進入點

`setup-entry.ts` 是 `index.ts` 的輕量替代方案，OpenClaw 僅需要設定介面（新手引導、設定修復、停用頻道檢查）時會載入它：

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

這可避免在設定流程期間載入繁重的執行階段程式碼（加密程式庫、命令列介面註冊、背景服務）。

將設定安全匯出保留在附屬模組中的隨附工作區頻道，可以使用 `openclaw/plugin-sdk/channel-entry-contract` 的 `defineBundledChannelSetupEntry(...)`，而非 `defineSetupPluginEntry(...)`。該隨附合約也支援選用的 `runtime` 匯出，使設定期間的執行階段接線能保持輕量且明確。

<AccordionGroup>
  <Accordion title="OpenClaw 何時使用 setupEntry 而非完整進入點">
    - 頻道已停用，但需要設定／新手引導介面。
    - 頻道已啟用，但尚未設定。
    - 已啟用延後載入（`deferConfiguredChannelFullLoadUntilAfterListen`）。

  </Accordion>
  <Accordion title="setupEntry 必須註冊的項目">
    - 頻道外掛物件（透過 `defineSetupPluginEntry`）。
    - 閘道監聽前所需的任何 HTTP 路由。
    - 啟動期間所需的任何閘道方法。

    這些啟動閘道方法仍應避免使用保留的核心管理命名空間，例如 `config.*` 或 `update.*`。

  </Accordion>
  <Accordion title="setupEntry 不應包含的項目">
    - 命令列介面註冊。
    - 背景服務。
    - 繁重的執行階段匯入（加密、SDK）。
    - 僅在啟動後才需要的閘道方法。

  </Accordion>
</AccordionGroup>

### 窄範圍設定輔助程式匯入

對於高頻的純設定路徑，如果只需要設定介面的一部分，請優先使用窄範圍的設定輔助程式介面，而非較廣泛的 `plugin-sdk/setup` 統整介面：

| 匯入路徑                   | 用途                                                                                      | 主要匯出                                                                                                                                                                                                                                                                                                              |
| -------------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime` | 在 `setupEntry`／延後頻道啟動期間仍保持可用的設定階段執行階段輔助程式 | `createSetupTranslator`、`createPatchedAccountSetupAdapter`、`createEnvPatchedAccountSetupAdapter`、`createSetupInputPresenceValidator`、`noteChannelLookupFailure`、`noteChannelLookupSummary`、`promptResolvedAllowFrom`、`splitSetupEntries`、`createAllowlistSetupWizardProxy`、`createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-tools`   | 設定／安裝命令列介面／封存／文件輔助程式                                                  | `formatCliCommand`、`detectBinary`、`extractArchive`、`resolveBrewExecutable`、`formatDocsLink`、`CONFIG_DIR`                                                                                                                                                                                                    |

當你需要完整的共用設定工具箱（包括 `moveSingleAccountChannelSectionToDefaultAccount(...)` 等設定修補輔助程式）時，請使用較廣泛的 `plugin-sdk/setup` 介面。

將 `createSetupTranslator(...)` 用於固定的設定精靈文案。它會依序使用 `OPENCLAW_LOCALE`、`LC_ALL`、`LC_MESSAGES` 和 `LANG` 中第一個非空白值，然後回退至英文。設定 `OPENCLAW_LOCALE=en` 可明確覆寫英文。將外掛專用的設定文字保留在外掛擁有的程式碼中，並僅將共用目錄鍵用於一般設定標籤、狀態文字，以及官方隨附外掛的設定文案。

設定修補配接器在匯入時仍可安全用於高頻路徑。其隨附的單一帳號提升合約介面查詢採用延遲載入，因此匯入 `plugin-sdk/setup-runtime` 不會在實際使用配接器前就立即載入隨附合約介面的探索功能。

### 頻道擁有的設定輸入欄位

`ChannelSetupInput` 是設定呼叫端與頻道外掛共用的通用封裝。
其永久具型別的欄位為 `name`、`token`、`tokenFile`、
`useEnv`、`allowFrom` 和 `defaultTo`。執行階段輸入物件仍可包含
其他由外掛擁有的鍵，但共用型別不會宣告索引簽章。每個外掛都必須宣告並限縮自己的設定欄位，
或在配接器邊界使用外掛擁有的結構描述進行驗證：

```typescript
import type { ChannelSetupAdapter, ChannelSetupInput } from "openclaw/plugin-sdk/channel-setup";

type AcmeSetupInput = ChannelSetupInput & {
  workspaceId?: string;
  webhookUrl?: string;
};

export const acmeSetupAdapter: ChannelSetupAdapter = {
  applyAccountConfig: ({ cfg, input }) => {
    const setupInput = input as AcmeSetupInput;
    return {
      ...cfg,
      channels: {
        ...cfg.channels,
        acme: {
          token: setupInput.token,
          workspaceId: setupInput.workspaceId,
          webhookUrl: setupInput.webhookUrl,
        },
      },
    };
  },
};
```

先前直接宣告於
`ChannelSetupInput` 上的頻道特定欄位，暫時仍保留型別，以維持外部原始碼相容性。
這些欄位已淘汰。2026-07-22 對 426 個已發布、位於樹外的
頻道外掛進行登錄檔全面檢查後，移除了 21 個沒有讀取端的欄位，並保留 22 個已知有
讀取端的欄位。每個保留欄位會在沒有任何已發布外掛讀取時立即刪除；
不需要版本界線。新的和隨附的外掛不得依賴此
層級；請在本機宣告其擁有的欄位。

### 頻道擁有的單一帳號提升

當頻道從單一帳號的頂層設定升級至 `channels.<id>.accounts.*` 時，預設共用行為會將提升的帳號範圍值移至 `accounts.default`。

每個頻道外掛都可以透過其設定介面卡擴充或縮限該提升行為：

- `singleAccountKeysToMove`：應移入提升後帳號的額外頂層鍵
- `namedAccountPromotionKeys`：當具名帳號已存在時，只有這些鍵會移入提升後的帳號；共用原則／傳遞鍵會留在頻道根層級
- `resolveSingleAccountPromotionTarget(...)`：選擇要接收提升值的現有帳號

`singleAccountKeysToMove` 的存在表示提升合約已完整。即使該欄位是空陣列，也請宣告它，以選擇不採用舊版鍵提升。省略此欄位的介面卡，會為已發布的外掛保留一個由讀取端支持的預先宣告提升層級。2026-07-22 的登錄檔全面檢查移除了 23 個沒有已發布相依項目的鍵，並保留六個常用鍵以及僅供設定使用的 `rooms` 鍵。每個保留鍵會在其已發布的讀取端遷移至宣告後立即刪除；不需要版本界線。

當 doctor 必須從輕量隨附設定構件載入這些宣告時，請在外掛套件資訊清單中宣告 `openclaw.setupFeatures.configPromotion: true`。僅供設定使用的外掛介面與完整頻道外掛必須公開相同的宣告。

使用已解析的外掛呼叫 `moveSingleAccountChannelSectionToDefaultAccount(...)` 時，請將其設定介面卡以 `setupSurface` 傳入。呼叫端提供的設定介面優先於載入和隨附查詢，使具範圍限制或僅供設定使用的外掛能獨立於全域註冊。

<Note>
Matrix 是目前的隨附範例。如果已存在剛好一個具名 Matrix 帳號，或 `defaultAccount` 指向現有的非標準鍵（例如 `Ops`），提升作業會保留該帳號，而不會建立新的 `accounts.default` 項目。
</Note>

## 設定結構描述

外掛設定會依照資訊清單中的 JSON Schema 進行驗證。使用者可透過以下方式設定外掛：

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

外掛會在註冊期間以 `api.pluginConfig` 接收此設定。

對於頻道特定設定，請改用頻道設定區段：

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### 建構頻道設定結構描述

使用 `buildChannelConfigSchema` 將 Zod 結構描述轉換為外掛擁有的設定構件所使用的 `ChannelConfigSchema` 包裝函式：

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

如果你已使用 JSON Schema 或 TypeBox 編寫合約，請使用直接輔助函式，讓 OpenClaw 能在中繼資料路徑上略過 Zod 至 JSON Schema 的轉換：

```typescript
import { Type } from "typebox";
import { buildJsonChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const configSchema = buildJsonChannelConfigSchema(
  Type.Object({
    token: Type.Optional(Type.String()),
    allowFrom: Type.Optional(Type.Array(Type.String())),
  }),
);
```

對於第三方外掛，冷路徑合約仍是外掛資訊清單：請將產生的 JSON Schema 鏡像至 `openclaw.plugin.json#channelConfigs`，讓設定結構描述、設定和 UI 介面能在不載入執行階段程式碼的情況下檢查 `channels.<id>`。

## 設定精靈

頻道外掛可以為 `openclaw onboard` 提供互動式設定精靈。該精靈是 `ChannelPlugin` 上的 `ChannelSetupWizard` 物件：

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "已連線",
    unconfiguredLabel: "尚未設定",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "機器人權杖",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "要使用環境中的 MY_CHANNEL_BOT_TOKEN 嗎？",
      keepPrompt: "要保留目前的權杖嗎？",
      inputPrompt: "輸入你的機器人權杖：",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

`ChannelSetupWizard` 也支援 `textInputs`、`dmPolicy`、`allowFrom`、`groupAccess`、`prepare`、`finalize` 等項目。完整的隨附範例請參閱 Discord 外掛的 `src/setup-core.ts`。

<AccordionGroup>
  <Accordion title="共用 allowFrom 提示">
    對於只需要標準 `note -> prompt -> parse -> merge -> patch` 流程的 DM 允許清單提示，建議使用 `openclaw/plugin-sdk/setup` 中的共用設定輔助函式：`createPromptParsedAllowFromForAccount(...)` 和 `createTopLevelChannelParsedAllowFromPrompt(...)`。
  </Accordion>
  <Accordion title="標準頻道設定狀態">
    對於只有標籤、分數和選用額外行不同的頻道設定狀態區塊，建議使用 `openclaw/plugin-sdk/setup` 中的 `createStandardChannelSetupStatus(...)`，而不是在每個外掛中自行建立相同的 `status` 物件。
  </Accordion>
  <Accordion title="選用頻道設定介面">
    對於只應在特定情境中出現的選用設定介面，請使用 `openclaw/plugin-sdk/channel-setup` 中的 `createOptionalChannelSetupSurface`：

    ```typescript
    import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

    const setupSurface = createOptionalChannelSetupSurface({
      channel: "my-channel",
      label: "我的頻道",
      npmSpec: "@myorg/openclaw-my-channel",
      docsPath: "/channels/my-channel",
    });
    // 傳回 { setupAdapter, setupWizard }
    ```

    當你只需要該選用安裝介面的其中一半時，`plugin-sdk/channel-setup` 也會公開較低階的 `createOptionalChannelSetupAdapter(...)` 和 `createOptionalChannelSetupWizard(...)` 建構器。

    產生的選用介面卡／精靈在實際寫入設定時會採取失敗關閉。它們會在 `validateInput`、`applyAccountConfig` 和 `finalize` 中重複使用同一則需要安裝的訊息，並在設定 `docsPath` 時附加文件連結。

  </Accordion>
  <Accordion title="二進位檔支援的設定輔助函式">
    對於二進位檔支援的設定 UI，建議使用共用的委派輔助函式，而不是在每個頻道中複製相同的二進位檔／狀態黏合程式碼：

    - `createDetectedBinaryStatus(...)`：適用於只有標籤、提示、分數和二進位檔偵測不同的狀態區塊
    - `createCliPathTextInput(...)`：適用於路徑支援的文字輸入
    - `createDelegatedSetupWizardProxy(...)`：當 `setupEntry` 需要以延遲方式將狀態、準備或完成行為轉送至較完整、較大型的精靈時使用
    - `createDelegatedTextInputShouldPrompt(...)`：當 `setupEntry` 只需要委派 `textInputs[*].shouldPrompt` 決策時使用

  </Accordion>
</AccordionGroup>

## 發布與安裝

**外部外掛：**發布至 [ClawHub](/zh-TW/clawhub)，然後安裝：

<Tabs>
  <Tab title="npm">
    ```bash
    openclaw plugins install @myorg/openclaw-my-plugin
    ```

    在啟動切換期間，裸套件規格會從 npm 安裝，除非名稱符合隨附或官方外掛 ID；在此情況下，OpenClaw 會改用該本機／官方副本。使用 `clawhub:`、`npm:`、`git:` 或 `npm-pack:` 來確定性地選擇來源——請參閱[管理外掛](/zh-TW/plugins/manage-plugins)。

  </Tab>
  <Tab title="僅限 ClawHub">
    ```bash
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```
  </Tab>
  <Tab title="npm 套件規格">
    當套件尚未移至 ClawHub，或你在遷移期間需要
    直接的 npm 安裝路徑時，請使用 npm：

    ```bash
    openclaw plugins install npm:@myorg/openclaw-my-plugin
    ```

  </Tab>
</Tabs>

**存放庫內外掛：**放置於隨附外掛工作區樹狀結構下；建置期間會自動探索這些外掛。

<Info>
對於來源為 npm 的安裝，`openclaw plugins install` 會將套件安裝至 `~/.openclaw/npm/projects` 下的每個外掛專案，並停用生命週期指令碼（`--ignore-scripts`）。請讓外掛相依性樹維持純 JS/TS，並避免使用需要 `postinstall` 建置的套件。
</Info>

<Note>
閘道啟動時不會安裝外掛相依套件。npm/git/ClawHub 安裝流程負責使相依套件趨於一致；本機外掛必須已安裝其相依套件。
</Note>

隨附套件的中繼資料會明確指定，不會在閘道啟動時從建置完成的 JavaScript 推斷。執行階段相依套件應屬於擁有它們的外掛套件；封裝版 OpenClaw 啟動時絕不會修復或鏡像外掛相依套件。

## 相關內容

- [建置外掛](/zh-TW/plugins/building-plugins) — 逐步入門指南
- [外掛資訊清單](/zh-TW/plugins/manifest) — 完整資訊清單結構描述參考
- [SDK 進入點](/zh-TW/plugins/sdk-entrypoints) — `definePluginEntry` 和 `defineChannelPluginEntry`
