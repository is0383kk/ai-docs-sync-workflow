---
read_when:
    - 你正在建置 OpenClaw 外掛
    - 你需要發布外掛設定結構描述或偵錯外掛驗證錯誤
summary: 外掛資訊清單與 JSON 綱要需求（嚴格設定驗證）
title: 外掛資訊清單
x-i18n:
    generated_at: "2026-07-26T07:26:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 244e5c8265ff79b0ff6e8f4b60c9635cccc3ba66093cecab458676beb9578264
    source_path: plugins/manifest.md
    workflow: 16
---

本頁說明 **OpenClaw 原生外掛資訊清單** `openclaw.plugin.json`。如需相容套件配置（Codex、Claude、Cursor），請參閱[外掛套件](/zh-TW/plugins/bundles)。

相容的套件格式會改用各自的資訊清單檔案：

- Codex 套件：`.codex-plugin/plugin.json`
- Claude 套件：`.claude-plugin/plugin.json`，或不含資訊清單的預設 Claude 元件配置
- Cursor 套件：`.cursor-plugin/plugin.json`

OpenClaw 會自動偵測這些配置，但不會根據下方的 `openclaw.plugin.json` 結構描述進行驗證。對於相容套件，當其配置符合 OpenClaw 的執行階段預期時，OpenClaw 會讀取套件中繼資料、宣告的 Skill 根目錄、Claude 命令根目錄、Claude `settings.json` 預設值、Claude LSP 預設值，以及支援的鉤子套件。

每個 OpenClaw 原生外掛都**必須**在**外掛根目錄**中提供 `openclaw.plugin.json`。OpenClaw 會讀取此檔案，以便在**不執行外掛程式碼**的情況下驗證設定。資訊清單遺失或無效會阻止設定驗證，並視為外掛錯誤。

如需完整的外掛系統指南，請參閱[外掛](/zh-TW/tools/plugin)；如需原生能力模型及目前的外部相容性指引，請參閱[能力模型](/zh-TW/plugins/architecture#public-capability-model)。

## 此檔案的用途

`openclaw.plugin.json` 是 OpenClaw 在**載入你的外掛程式碼之前**讀取的中繼資料。其中所有內容都必須能以足夠低的成本進行檢查，而無須啟動外掛執行階段。

**適用於：**

- 外掛識別、設定驗證及設定 UI 提示
- 驗證、初始設定及安裝中繼資料（別名、自動啟用、供應商環境變數、驗證選項）
- 控制平面介面的啟用提示
- 模型家族簡寫的所有權
- 靜態能力所有權快照（`contracts`）
- 儀表板小工具的資料繫結與動作動詞
- 外掛啟用期間應存在的靜態 MCP 伺服器
- 共用 `openclaw qa` 主機可檢查的 QA 執行器中繼資料
- 合併至目錄及驗證介面的頻道專屬設定中繼資料

**請勿用於：**註冊原生執行階段鉤子、宣告外掛程式碼進入點或 npm 安裝中繼資料。這些項目應放在你的外掛程式碼及 `package.json` 中。

## 最小範例

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## 完整範例

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "OpenRouter 供應商外掛",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "modelIdNormalization": {
    "providers": {
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  },
  "providerEndpoints": [
    {
      "endpointClass": "openrouter",
      "hostSuffixes": ["openrouter.ai"]
    }
  ],
  "providerRequest": {
    "providers": {
      "openrouter": {
        "family": "openrouter"
      }
    }
  },
  "cliBackends": ["openrouter-cli"],
  "syntheticAuthRefs": ["openrouter-cli"],
  "setup": {
    "providers": [
      {
        "id": "openrouter",
        "envVars": ["OPENROUTER_API_KEY"]
      }
    ]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "OpenRouter API 金鑰",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "OpenRouter API 金鑰",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "API 金鑰",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## 頂層欄位參考

| 欄位                                 | 必填     | 類型                         | 說明                                                                                                                                                                                                                                                                                           |
| ------------------------------------ | -------- | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                 | 是       | `string`                     | 標準外掛 ID。這是 `plugins.entries.<id>` 中使用的 ID。                                                                                                                                                                                                                                      |
| `configSchema`                       | 是       | `object`                     | 此外掛設定的內嵌 JSON Schema。                                                                                                                                                                                                                                                          |
| `requiresPlugins`                    | 否       | `string[]`                   | 必須一併安裝，此外掛才能生效的外掛 ID。探索機制會讓此外掛保持可載入狀態，但若缺少任何必要外掛，則會發出警告。                                                                                                                                                                           |
| `enabledByDefault`                   | 否       | `true`                       | 將隨附外掛標記為預設啟用。省略此項或設定任何非 `true` 值，即可讓此外掛保持預設停用。                                                                                                                                                                                                 |
| `enabledByDefaultOnPlatforms`        | 否       | `string[]`                   | 將隨附外掛標記為僅在所列 Node.js 平台上預設啟用，例如 `["darwin"]`。明確設定仍具有優先權。                                                                                                                                                                                          |
| `legacyPluginIds`                    | 否       | `string[]`                   | 會正規化為此標準外掛 ID 的舊版 ID。                                                                                                                                                                                                                                                      |
| `autoEnableWhenConfiguredProviders`  | 否       | `string[]`                   | 當驗證、設定或模型參照提及這些提供者 ID 時，應自動啟用此外掛。                                                                                                                                                                                                                         |
| `kind`                               | 否       | `PluginKind \| PluginKind[]` | 宣告 `plugins.slots.*` 使用的一或多個互斥外掛種類（`"memory"`、`"context-engine"`）。同時擁有兩個位置的外掛會在同一個陣列中宣告兩種種類。                                                                                                                                               |
| `channels`                           | 否       | `string[]`                   | 此外掛擁有的頻道 ID。用於探索和設定驗證。                                                                                                                                                                                                                                               |
| `providers`                          | 否       | `string[]`                   | 此外掛擁有的提供者 ID。                                                                                                                                                                                                                                                                 |
| `providerCatalogEntry`               | 否       | `string`                     | 輕量型提供者目錄模組路徑，相對於外掛根目錄，用於限定於資訊清單範圍的提供者目錄中繼資料；無須啟用完整外掛執行階段即可載入。                                                                                                                                                              |
| `modelSupport`                       | 否       | `object`                     | 資訊清單擁有的模型系列簡寫中繼資料，用於在執行階段之前自動載入外掛。                                                                                                                                                                                                                   |
| `modelCatalog`                       | 否       | `object`                     | 此外掛擁有之提供者的宣告式模型目錄中繼資料。這是未來在不載入外掛執行階段的情況下，進行唯讀列出、初始設定、模型選擇器、別名及隱藏的控制平面合約。                                                                                                                                         |
| `modelPricing`                       | 否       | `object`                     | 提供者擁有的外部定價查詢原則。可用它讓本機／自行託管的提供者不使用遠端定價目錄，或將提供者參照對應至 OpenRouter/LiteLLM 目錄 ID，而無須在核心中硬編碼提供者 ID。                                                                                                                         |
| `modelIdNormalization`               | 否       | `object`                     | 必須在提供者執行階段載入前執行，由提供者擁有的模型 ID 別名／前綴清理。                                                                                                                                                                                                                  |
| `providerEndpoints`                  | 否       | `object[]`                   | 提供者路由的資訊清單所擁有端點主機／baseUrl 中繼資料，核心必須在提供者執行階段載入前將其分類。                                                                                                                                                                                          |
| `providerRequest`                    | 否       | `object`                     | 通用要求原則在提供者執行階段載入前使用的輕量型提供者系列及要求相容性中繼資料。                                                                                                                                                                                                         |
| `secretProviderIntegrations`         | 否       | `Record<string, object>`     | 宣告式 SecretRef exec 提供者預設集，讓設定或安裝介面可以提供這些選項，而無須在核心中硬編碼提供者特定的整合。                                                                                                                                                                          |
| `cliBackends`                        | 否       | `string[]`                   | 此外掛擁有的命令列介面推論後端 ID。用於根據明確設定參照在啟動時自動啟用。                                                                                                                                                                                                               |
| `syntheticAuthRefs`                  | 否       | `string[]`                   | 在執行階段載入前進行冷模型探索時，應探測其外掛所擁有之合成驗證鉤子的提供者或命令列介面後端參照。                                                                                                                                                                                       |
| `nonSecretAuthMarkers`               | 否       | `string[]`                   | 隨附外掛擁有的預留位置 API 金鑰值，代表非機密的本機、OAuth 或環境認證資訊狀態。                                                                                                                                                                                                         |
| `commandAliases`                     | 否       | `object[]`                   | 此外掛擁有的命令名稱，應在執行階段載入前產生外掛感知的設定與命令列介面診斷。                                                                                                                                                                                                            |
| `providerUsageAuthEnvVars`           | 否       | `Record<string, string[]>`   | 僅用於用量／計費的提供者認證資訊。OpenClaw 使用這些名稱進行用量探索和機密資訊清理，但絕不將其用於推論驗證。                                                                                                                                                                             |
| `providerAuthAliases`                | 否       | `Record<string, string>`     | 應重複使用另一個提供者 ID 進行驗證查詢的提供者 ID，例如共用基礎提供者 API 金鑰和驗證設定檔的程式設計提供者。                                                                                                                                                                           |
| `providerAuthChoices`                | 否       | `object[]`                   | 用於初始設定選擇器、偏好提供者解析及簡易命令列介面旗標連接的輕量型驗證選項中繼資料。                                                                                                                                                                                                    |
| `activation`                         | 否       | `object`                     | 用於啟動、提供者、命令、頻道、路由及能力觸發載入的輕量型啟用規劃器中繼資料。僅限中繼資料；實際行為仍由外掛執行階段擁有。                                                                                                                                                               |
| `setup`                              | 否       | `object`                     | 探索和設定介面無須載入外掛執行階段即可檢查的輕量型設定／初始設定描述元。                                                                                                                                                                                                                |
| `qaRunners`                          | 否       | `object[]`                   | 共用 `openclaw qa` 主機在外掛執行階段載入前使用的輕量型 QA 執行器描述元。                                                                                                                                                                                                          |
| `dashboard`                          | 否       | `object`                     | 儀表板小工具的資料繫結與動作動詞。每個項目都會根據此外掛註冊的閘道方法進行驗證，並須具備必要的讀取或寫入範圍。請參閱[儀表板參考](#dashboard-reference)。                                                                                                                                |
| `mcpServers`                         | 否       | `Record<string, object>`     | 啟用此外掛時提供的靜態 MCP 伺服器定義。相對命令引數與工作目錄會從外掛根目錄解析。操作者的 `mcp.servers` 項目會覆寫或停用同名定義。請參閱 [MCP 伺服器參考](#mcp-server-reference)。 |
| `contracts`                          | 否       | `object`                     | 外部驗證掛鉤、嵌入、語音、即時轉錄、即時語音、媒體理解、影像／影片／音樂生成、網頁擷取、網頁搜尋、工作程式提供者、文件／網頁內容擷取及工具所有權的靜態功能所有權快照。                     |
| `configContracts`                    | 否       | `object`                     | 由資訊清單擁有、供通用核心輔助程式使用的設定行為：危險旗標偵測、SecretRef 遷移目標，以及舊版設定路徑限縮。請參閱 [configContracts 參考](#configcontracts-reference)。                                                                         |
| `mediaUnderstandingProviderMetadata` | 否       | `Record<string, object>`     | 為 `contracts.mediaUnderstandingProviders` 中宣告的提供者 ID 設定低成本的媒體理解預設值。                                                                                                                                                                                       |
| `imageGenerationProviderMetadata`    | 否       | `Record<string, object>`     | 為 `contracts.imageGenerationProviders` 中宣告的提供者 ID 設定低成本的影像生成驗證中繼資料，包括提供者擁有的驗證別名與基礎 URL 防護。                                                                                                                             |
| `videoGenerationProviderMetadata`    | 否       | `Record<string, object>`     | 為 `contracts.videoGenerationProviders` 中宣告的提供者 ID 設定低成本的影片生成驗證中繼資料，包括提供者擁有的驗證別名與基礎 URL 防護。                                                                                                                             |
| `musicGenerationProviderMetadata`    | 否       | `Record<string, object>`     | 為 `contracts.musicGenerationProviders` 中宣告的提供者 ID 設定低成本的音樂生成驗證中繼資料，包括提供者擁有的驗證別名與基礎 URL 防護。                                                                                                                             |
| `toolMetadata`                       | 否       | `Record<string, object>`     | 為 `contracts.tools` 中宣告、由外掛擁有的工具設定低成本的可用性中繼資料。當工具在缺乏設定、環境或驗證依據時不應載入執行階段，請使用此項目。                                                                                                                      |
| `channelConfigs`                     | 否       | `Record<string, object>`     | 由資訊清單擁有的頻道設定中繼資料，會在載入執行階段前合併至探索與驗證介面。                                                                                                                                                                                     |
| `skills`                             | 否       | `string[]`                   | 要載入的 Skill 目錄，路徑相對於外掛根目錄。                                                                                                                                                                                                                                        |
| `name`                               | 否       | `string`                     | 方便使用者閱讀的外掛名稱。                                                                                                                                                                                                                                                                    |
| `description`                        | 否       | `string`                     | 顯示於外掛介面中的簡短摘要。                                                                                                                                                                                                                                                        |
| `catalog`                            | 否       | `object`                     | 外掛目錄介面的選用呈現提示。此中繼資料不會安裝或啟用外掛，也不會授予外掛信任。                                                                                                                                                                   |
| `icon`                               | 否       | `string`                     | 市集／目錄卡片使用的 HTTPS 圖片 URL。ClawHub 接受任何有效的 `https://` URL；若省略此項目或其無效，則會改用預設外掛圖示。                                                                                                                             |
| `version`                            | 否       | `string`                     | 資訊用途的外掛版本。                                                                                                                                                                                                                                                                  |
| `uiHints`                            | 否       | `Record<string, object>`     | 設定欄位的 UI 標籤、預留位置文字與敏感性提示。                                                                                                                                                                                                                              |

## MCP 伺服器參考

`mcpServers` 可讓原生外掛隨附 MCP 伺服器（包括 MCP App），而不需要操作人員在 `openclaw.json` 中重複其靜態處理程序定義：

```json
{
  "mcpServers": {
    "example": {
      "transport": "stdio",
      "command": "node",
      "args": ["./mcp-server.js"]
    }
  }
}
```

OpenClaw 只會在所屬外掛啟用時納入這些伺服器。相對的 `command`、`args`、`cwd` 和 `workingDirectory` 路徑會從外掛根目錄解析。使用者設定仍具有最高優先權：`mcp.servers.<name>` 可以取代外掛預設值，或將 `enabled: false` 設為略過該伺服器。MCP App 算繪和伺服器工具呼叫仍需要一般的 MCP Apps 設定與有效的工具原則；宣告伺服器不會繞過任一邊界。

## 儀表板參考

`dashboard` 可讓已啟用的外掛向已獲授權的儀表板小工具公開現有的閘道 RPC，而不需要將外掛原則加入核心。資料繫結必須指定同一外掛透過 `operator.read` 註冊的方法；動作動詞必須指定其透過 `operator.write` 註冊的方法。若不相符，外掛會在註冊期間遭拒。

```json
{
  "dashboard": {
    "dataBindings": [
      {
        "id": "items.list",
        "method": "example.items.list",
        "description": "列出範例項目。"
      }
    ],
    "actionVerbs": [
      {
        "id": "refresh",
        "method": "example.items.refresh",
        "description": "重新整理範例項目。",
        "paramShape": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "force": { "type": "boolean" }
          }
        }
      }
    ]
  }
}
```

資訊清單 ID 的作用域僅限外掛本身。小工具授權使用 `<plugin-id>.<id>`，例如 `example.items.list` 和 `example.refresh`。為確保持久化授權的命名空間明確無歧義，OpenClaw 會將外掛 ID 區段中的 `%` 和 `.` 分別逸出為 `%25` 和 `%2E`；一般外掛 ID 則維持自然形式。`paramShape` 是選用的 JSON Schema，OpenClaw 會在叫用外掛 RPC 前將其套用至動作參數物件。

## 目錄參考

`catalog` 為外掛瀏覽器提供選用的顯示提示。主機可以忽略這些提示。這些提示絕不會安裝或啟用外掛，也不會變更其執行階段行為或信任層級。

```json
{
  "catalog": {
    "featured": true,
    "order": 10
  }
}
```

| 欄位      | 類型      | 含義                                                                       |
| ---------- | --------- | -------------------------------------------------------------------------- |
| `featured` | `boolean` | 目錄介面是否應重點推薦此外掛。                                             |
| `order`    | `number`  | 精選外掛之間的遞增顯示提示；數值較小者會較早顯示。                         |

## 生成提供者中繼資料參考

生成提供者中繼資料欄位描述相符 `contracts.*GenerationProviders` 清單中所宣告提供者的靜態驗證訊號。OpenClaw 會在提供者執行階段載入前讀取這些欄位，讓核心工具無須匯入每個提供者外掛，即可判斷生成提供者是否可用。

這些欄位僅用於成本低廉的宣告式事實。傳輸、要求轉換、權杖重新整理、認證資訊驗證及實際生成行為仍由外掛執行階段負責。

```json
{
  "contracts": {
    "imageGenerationProviders": ["example-image"]
  },
  "imageGenerationProviderMetadata": {
    "example-image": {
      "aliases": ["example-image-oauth"],
      "authProviders": ["example-image"],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example-image.config",
          "overlayPath": "image",
          "mode": {
            "path": "mode",
            "default": "local",
            "allowed": ["local"]
          },
          "requiredAny": ["workflow", "workflowPath"],
          "required": ["promptNodeId"]
        }
      ],
      "authSignals": [
        {
          "provider": "example-image"
        },
        {
          "provider": "example-image-oauth",
          "providerBaseUrl": {
            "provider": "example-image",
            "defaultBaseUrl": "https://api.example.com/v1",
            "allowedBaseUrls": ["https://api.example.com/v1"]
          }
        }
      ]
    }
  }
}
```

每個中繼資料項目支援：

| 欄位                   | 必要 | 類型       | 含義                                                                                                                                                 |
| ---------------------- | -------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `aliases`              | 否       | `string[]` | 應視為此生成提供者靜態驗證別名的其他提供者 ID。                                                                                                     |
| `authProviders`        | 否       | `string[]` | 其已設定驗證設定檔應視為此生成提供者驗證依據的提供者 ID。                                                                                           |
| `configSignals`        | 否       | `object[]` | 適用於無須驗證設定檔或環境變數即可設定之本機或自行託管提供者的低成本、僅設定可用性訊號。                                                             |
| `authSignals`          | 否       | `object[]` | 明確的驗證訊號。若存在，這些訊號會取代由提供者 ID、`aliases` 和 `authProviders` 組成的預設訊號集。                                      |
| `referenceAudioInputs` | 否       | `boolean`  | 僅限影片生成。當提供者接受參考音訊資產時設為 `true`；否則 `video_generate` 會隱藏音訊參考參數。                                       |

每個 `configSignals` 項目支援：

| 欄位             | 必要 | 類型       | 含義                                                                                                                                                                                      |
| ---------------- | -------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `rootPath`       | 是      | `string`   | 要檢查之外掛所屬設定物件的點號路徑，例如 `plugins.entries.example.config`。                                                                                                                |
| `overlayPath`    | 否       | `string`   | 根設定內的點號路徑；評估訊號前，該路徑的物件應覆疊根物件。適用於 `image`、`video` 或 `music` 等功能專屬設定。     |
| `overlayMapPath` | 否       | `string`   | 根設定內的點號路徑；該路徑物件中的每個值都應各自覆疊根物件。適用於 `accounts` 等具名帳號對應，其中任何已設定帳號皆應符合資格。              |
| `required`       | 否       | `string[]` | 有效設定內必須具有已設定值的點號路徑。字串不得為空；物件與陣列不得為空。                                                                            |
| `requiredAny`    | 否       | `string[]` | 有效設定內至少一個必須具有已設定值的點號路徑。                                                                                                      |
| `mode`           | 否       | `object`   | 有效設定內的選用字串模式防護條件。僅當僅設定可用性只適用於某一模式時使用。                                                                          |

每個 `mode` 防護條件支援：

| 欄位         | 必要 | 類型       | 含義                                                                               |
| ------------ | -------- | ---------- | ---------------------------------------------------------------------------------- |
| `path`       | 否       | `string`   | 有效設定內的點號路徑。預設為 `mode`。                                  |
| `default`    | 否       | `string`   | 設定省略該路徑時要使用的模式值。                                                   |
| `allowed`    | 否       | `string[]` | 若存在，只有在有效模式為其中一個值時，訊號才會通過。                               |
| `disallowed` | 否       | `string[]` | 若存在，當有效模式為其中一個值時，訊號會失敗。                                     |

每個 `authSignals` 項目支援：

| 欄位              | 必要 | 類型     | 含義                                                                                                                                                                  |
| ----------------- | -------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | 是      | `string` | 要在已設定驗證設定檔中檢查的提供者 ID。                                                                                                                               |
| `providerBaseUrl` | 否       | `object` | 選用的防護條件，讓訊號只有在所參照的已設定提供者使用允許的基礎 URL 時才列入計算。適用於驗證別名僅對特定 API 有效的情況。                                               |

每個 `providerBaseUrl` 防護條件支援：

| 欄位              | 必要 | 類型       | 含義                                                                                                                                                 |
| ----------------- | -------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | 是      | `string`   | 應檢查其 `baseUrl` 的提供者設定 ID。                                                                                                       |
| `defaultBaseUrl`  | 否       | `string`   | 提供者設定省略 `baseUrl` 時所採用的基礎 URL。                                                                                              |
| `allowedBaseUrls` | 是      | `string[]` | 此驗證訊號允許的基礎 URL。當已設定或預設的基礎 URL 不符合這些正規化值的任一項時，會忽略該訊號。                                                     |

## 工具中繼資料參考

`toolMetadata` 使用與生成提供者中繼資料相同的 `configSignals` 和 `authSignals` 結構，並以工具名稱作為索引鍵。`contracts.tools` 宣告所有權。`toolMetadata` 宣告低成本的可用性依據，讓 OpenClaw 不必僅為了讓工具工廠傳回 `null` 而匯入外掛執行階段。

```json
{
  "setup": {
    "providers": [
      {
        "id": "example",
        "envVars": ["EXAMPLE_API_KEY"]
      }
    ]
  },
  "contracts": {
    "tools": ["example_search"]
  },
  "toolMetadata": {
    "example_search": {
      "authSignals": [
        {
          "provider": "example"
        }
      ],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example.config",
          "overlayPath": "search",
          "required": ["apiKey"]
        }
      ]
    }
  }
}
```

`toolMetadata` 項目除了接受上述共用的 `configSignals`/`authSignals` 欄位外，也接受 `optional`（將工具標記為啟用外掛時非必要）與 `replaySafe`（將工具執行標記為可在模型回合未完成後安全地重複執行）。

如果工具沒有 `toolMetadata`，OpenClaw 會保留現有行為，並在工具合約符合政策時載入擁有該工具的外掛。對於其工廠函式依賴驗證／設定的熱門路徑工具，外掛作者應宣告 `toolMetadata`，而不是讓核心匯入執行階段來詢問。

## providerAuthChoices 參考資料

每個 `providerAuthChoices` 項目描述一個初始設定或驗證選項。OpenClaw 會在提供者執行階段載入前讀取此項目。提供者設定清單會使用這些資訊清單選項、從描述元衍生的設定選項，以及安裝目錄中繼資料，而不載入提供者執行階段。

| 欄位                 | 必填 | 類型                                                                  | 意義                                                                                             |
| --------------------- | -------- | --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `provider`            | 是      | `string`                                                              | 此選項所屬的提供者 ID。                                                                       |
| `method`              | 是      | `string`                                                              | 要分派至的驗證方法 ID。                                                                            |
| `choiceId`            | 是      | `string`                                                              | 初始設定與命令列介面流程使用的穩定驗證選項 ID。                                                   |
| `choiceLabel`         | 否       | `string`                                                              | 使用者可見的標籤。若省略，OpenClaw 會退回使用 `choiceId`。                                         |
| `choiceHint`          | 否       | `string`                                                              | 選擇器的簡短輔助文字。                                                                         |
| `icon`                | 否       | HTTPS URL                                                             | 支援的初始設定用戶端在此選項旁顯示的圖像。                                         |
| `website`             | 否       | HTTPS URL                                                             | 支援的初始設定用戶端顯示的產品、登入或安裝頁面。                             |
| `assistantPriority`   | 否       | `number`                                                              | 在助理驅動的互動式選擇器中，值越小排序越前面。                                        |
| `assistantVisibility` | 否       | `"visible"` \| `"manual-only"`                                        | 在仍允許手動透過命令列介面選取的同時，從助理選擇器隱藏此選項。                         |
| `deprecatedChoiceIds` | 否       | `string[]`                                                            | 應將使用者重新導向此替代選項的舊版選項 ID。                                  |
| `groupId`             | 否       | `string`                                                              | 用於將相關選項分組的選用群組 ID。                                                           |
| `groupLabel`          | 否       | `string`                                                              | 該群組的使用者可見標籤。                                                                         |
| `groupHint`           | 否       | `string`                                                              | 群組的簡短輔助文字。                                                                          |
| `onboardingFeatured`  | 否       | `boolean`                                                             | 在互動式初始設定選擇器的精選層級中，於 "More..." 項目之前顯示此群組。 |
| `optionKey`           | 否       | `string`                                                              | 簡單單旗標驗證流程的內部選項鍵。                                                       |
| `cliFlag`             | 否       | `string`                                                              | 命令列介面旗標名稱，例如 `--openrouter-api-key`。                                                            |
| `cliOption`           | 否       | `string`                                                              | 完整的命令列介面選項形式，例如 `--openrouter-api-key <key>`。                                              |
| `cliDescription`      | 否       | `string`                                                              | 命令列介面說明中使用的描述。                                                                             |
| `appGuidedSecret`     | 否       | `boolean`                                                             | 一個貼上的祕密加上提供者預設值，即足以完成應用程式引導的設定。                              |
| `appGuidedDiscovery`  | 否       | `boolean`                                                             | 相符的執行階段驗證方法透過 `appGuidedSetup` 負責唯讀的本機探索。                 |
| `appGuidedAuth`       | 否       | `"oauth"` \| `"device-code"`                                          | 由提供者擁有、原生設定用戶端可通用呈現的互動式登入。                        |
| `onboardingScopes`    | 否       | `Array<"text-inference" \| "image-generation" \| "music-generation">` | 此選項應出現在哪些初始設定介面中。若省略，預設為 `["text-inference"]`。  |

當 `appGuidedDiscovery` 為 true 時，相符的提供者驗證方法必須公開
`appGuidedSetup.detect` 與 `appGuidedSetup.prepare`。偵測必須是
唯讀的：不得登入、拉取模型、下載或寫入設定。準備階段會重新檢查
確切選取的模型並傳回設定提案；OpenClaw 會隔離地即時測試該
提案，且僅在成功後才提交。

## commandAliases 參考資料

當外掛擁有一個使用者可能誤放入 `plugins.allow`，或嘗試當作根層級命令列介面命令執行的執行階段命令名稱時，請使用 `commandAliases`。OpenClaw 會使用此中繼資料進行診斷，而不匯入外掛執行階段程式碼。

```json
{
  "commandAliases": [
    {
      "name": "dreaming",
      "kind": "runtime-slash",
      "cliCommand": "memory"
    }
  ]
}
```

| 欄位        | 必填 | 類型              | 意義                                                           |
| ------------ | -------- | ----------------- | ----------------------------------------------------------------------- |
| `name`       | 是      | `string`          | 屬於此外掛的命令名稱。                               |
| `kind`       | 否       | `"runtime-slash"` | 將此別名標記為聊天斜線命令，而非根層級命令列介面命令。 |
| `cliCommand` | 否       | `string`          | 若存在，供命令列介面操作建議使用的相關根層級命令列介面命令。  |

## activation 參考資料

當外掛可以低成本地宣告哪些控制平面事件應將其納入啟用／載入計畫時，請使用 `activation`。

此區塊是規劃器中繼資料，而非生命週期 API。它不會註冊執行階段行為、不會取代 `register(...)`，也不保證外掛程式碼已經執行。啟用規劃器會使用這些欄位縮小候選外掛的範圍，之後才退回使用現有的資訊清單擁有權中繼資料，例如 `providers`、`channels`、`commandAliases`、`setup.providers`、`contracts.tools` 與鉤子。

應優先使用已能描述擁有權的最精確中繼資料。當 `providers`、`channels`、`commandAliases`、設定描述元或 `contracts` 能表達該關係時，請使用這些欄位。無法由這些擁有權欄位表示的額外規劃器提示，請使用 `activation`。對於命令列介面執行階段別名（例如 `claude-cli`、`my-cli` 或 `google-gemini-cli`），請使用頂層 `cliBackends`；`activation.onAgentHarnesses` 僅適用於尚無擁有權欄位的嵌入式代理程式框架 ID。

每個外掛都應有意識地設定 `activation.onStartup`。只有當外掛必須在閘道啟動期間執行時，才將其設為 `true`。當外掛在啟動時處於非作用狀態，且只應由更精確的觸發條件載入時，請將其設為 `false`。省略 `onStartup` 不再會隱含地於啟動時載入外掛；對於啟動、頻道、設定、代理程式框架、記憶體或其他更精確的啟用觸發條件，請使用明確的啟用中繼資料。

```json
{
  "activation": {
    "onStartup": false,
    "onProviders": ["openai"],
    "onCommands": ["models"],
    "onChannels": ["web"],
    "onRoutes": ["gateway-webhook"],
    "onConfigPaths": ["browser"],
    "onCapabilities": ["provider", "tool"]
  }
}
```

| 欄位               | 必填 | 類型                                                 | 意義                                                                                                                                                                                        |
| ------------------ | ---- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onStartup`        | 否   | `boolean`                                            | 明確啟用閘道啟動。每個外掛都應設定此項。`true` 會在啟動期間匯入外掛；`false` 則會讓它保持延遲至啟動時載入，除非另一個相符的觸發條件要求載入。 |
| `onProviders`      | 否   | `string[]`                                           | 應將此外掛納入啟用／載入計畫的提供者 ID。                                                                                                                      |
| `onAgentHarnesses` | 否   | `string[]`                                           | 應將此外掛納入啟用／載入計畫的內嵌代理程式框架執行階段 ID。命令列介面後端別名請使用頂層 `cliBackends`。                                           |
| `onCommands`       | 否   | `string[]`                                           | 應將此外掛納入啟用／載入計畫的命令 ID。                                                                                                                       |
| `onChannels`       | 否   | `string[]`                                           | 應將此外掛納入啟用／載入計畫的頻道 ID。                                                                                                                       |
| `onRoutes`         | 否   | `string[]`                                           | 應將此外掛納入啟用／載入計畫的路由種類。                                                                                                                       |
| `onConfigPaths`    | 否   | `string[]`                                           | 當路徑存在且未明確停用時，應將此外掛納入啟動／載入計畫的相對於根目錄的設定路徑。                                                      |
| `onCapabilities`   | 否   | `Array<"provider" \| "channel" \| "tool" \| "hook">` | 控制平面啟用規劃所使用的廣泛功能提示。可行時請優先使用範圍更精確的欄位。                                                                                     |

目前的實際使用端：

- 閘道啟動規劃使用 `activation.onStartup` 進行明確的啟動匯入。
- 由命令觸發的命令列介面規劃會回退至舊版 `commandAliases[].cliCommand` 或 `commandAliases[].name`。
- 代理程式執行階段啟動規劃對內嵌框架使用 `activation.onAgentHarnesses`，對命令列介面執行階段別名則使用頂層 `cliBackends[]`。
- 由頻道觸發的設定／頻道規劃，會在缺少明確的頻道啟用中繼資料時回退至舊版 `channels[]` 擁有權。
- 啟動外掛規劃會對非頻道的根設定介面使用 `activation.onConfigPaths`，例如內建瀏覽器外掛的 `browser` 區塊。
- 由提供者觸發的設定／執行階段規劃，會在缺少明確的提供者啟用中繼資料時回退至舊版 `providers[]` 和頂層 `cliBackends[]` 擁有權。

規劃器診斷可區分明確的啟用提示與資訊清單擁有權回退。例如，`activation-command-hint` 表示 `activation.onCommands` 相符，而 `manifest-command-alias` 表示規劃器改用了 `commandAliases` 擁有權。這些原因標籤供主機診斷與測試使用；外掛作者應持續宣告最能描述擁有權的中繼資料。

## qaRunners 參考

當外掛在共用 `openclaw qa` 根目錄下提供一或多個傳輸執行器時，
請使用 `qaRunners`。此中繼資料應保持輕量且靜態；外掛
執行階段仍透過輕量的 `runtime-api.ts` 介面負責實際的命令列介面註冊，
該介面會匯出相符的 `qaRunnerCliRegistrations`。選用的
`adapterFactory` 可將傳輸方式公開給共用 QA 情境，而不會
變更已註冊命令的執行器。

```json
{
  "qaRunners": [
    {
      "commandName": "matrix",
      "description": "對可拋棄式家用伺服器執行由 Docker 支援的 Matrix 即時 QA 流程"
    }
  ]
}
```

| 欄位          | 必填 | 類型     | 意義                                                               |
| ------------- | ---- | -------- | ------------------------------------------------------------------ |
| `commandName` | 是   | `string` | 掛載於 `openclaw qa` 下的子命令，例如 `matrix`。    |
| `description` | 否   | `string` | 共用主機需要預留命令時使用的備援說明文字。 |

`adapterFactory` ID 必須與 `commandName` 相符。請勿為
資訊清單中不存在的命令匯出註冊項目。

## setup 參考

當設定與初始設定介面需要在載入執行階段之前取得外掛擁有的輕量中繼資料時，請使用 `setup`。

```json
{
  "setup": {
    "providers": [
      {
        "id": "openai",
        "authMethods": ["api-key"],
        "envVars": ["OPENAI_API_KEY"],
        "authEvidence": [
          {
            "type": "local-file-with-env",
            "fileEnvVar": "OPENAI_CREDENTIALS_FILE",
            "requiresAllEnv": ["OPENAI_PROJECT"],
            "credentialMarker": "openai-local-credentials",
            "source": "openai 本機認證資訊"
          }
        ]
      }
    ],
    "cliBackends": ["openai-cli"],
    "configMigrations": ["legacy-openai-auth"],
    "requiresRuntime": false
  }
}
```

頂層 `cliBackends` 仍然有效，並繼續描述命令列介面推論後端。`setup.cliBackends` 是供控制平面／設定流程使用的設定專用描述項介面，應僅包含中繼資料。

若存在，`setup.providers` 與 `setup.cliBackends` 是設定探索時優先使用的描述項優先查詢介面。若描述項只能縮小候選外掛範圍，而設定仍需要更豐富的設定階段執行階段掛鉤，請設定 `requiresRuntime: true`，並保留 `setup-api` 作為備援執行路徑。

OpenClaw 會將 `setup.providers[].envVars` 納入一般提供者驗證與環境變數查詢。請將設定與狀態環境中繼資料放在此處。

當計費或組織層級認證資訊必須啟用 `resolveUsageAuth`，但不能成為推論認證資訊時，請使用 `providerUsageAuthEnvVars`。這些名稱會納入工作區 dotenv 封鎖、ACP 子程序移除、沙箱機密資訊篩選，以及廣泛的機密資訊清理。提供者執行階段仍會在 `resolveUsageAuth` 中讀取並分類該值。

當沒有可用的設定項目，或 `setup.requiresRuntime: false` 宣告不需要設定執行階段時，OpenClaw 也可從 `setup.providers[].authMethods` 衍生簡單的設定選項。對於自訂標籤、命令列介面旗標、初始設定範圍與助理中繼資料，仍優先使用明確的 `providerAuthChoices` 項目。

只有當這些描述項足以支援設定介面時，才設定 `requiresRuntime: false`。OpenClaw 會將明確的 `false` 視為僅使用描述項的契約，且不會執行 `setup-api` 或 `openclaw.setupEntry` 來查詢設定。若僅使用描述項的外掛仍提供其中一個設定執行階段項目，OpenClaw 會回報附加診斷並繼續忽略該項目。省略 `requiresRuntime` 會保留舊版回退行為，讓已新增描述項但未新增此旗標的現有外掛不致中斷。

由於設定查詢可能執行外掛擁有的 `setup-api` 程式碼，正規化後的 `setup.providers[].id` 與 `setup.cliBackends[]` 值在探索到的外掛之間必須保持唯一。若擁有權不明確，系統會採取封閉式失敗，而非依探索順序選出一方。

當設定執行階段確實執行時，若 `setup-api` 註冊了資訊清單描述項未宣告的提供者或命令列介面後端，或描述項沒有相符的執行階段註冊項目，設定登錄診斷會回報描述項偏移。這些診斷屬於附加資訊，不會拒絕舊版外掛。

### setup.providers 參考

| 欄位           | 必填 | 類型       | 意義                                                                                             |
| -------------- | ---- | ---------- | ------------------------------------------------------------------------------------------------ |
| `id`           | 是   | `string`   | 在設定或初始設定期間公開的提供者 ID。正規化後的 ID 應在全域保持唯一。             |
| `authMethods`  | 否   | `string[]` | 此提供者無須載入完整執行階段即可支援的設定／驗證方法 ID。                       |
| `envVars`      | 否   | `string[]` | 一般設定／狀態介面可在載入外掛執行階段之前檢查的環境變數。               |
| `authEvidence` | 否   | `object[]` | 可透過非機密標記進行驗證的提供者所使用的輕量本機驗證證據檢查。 |

`authEvidence` 用於提供者擁有的本機認證資訊標記，這些標記可在不載入執行階段程式碼的情況下驗證。這些檢查必須保持輕量且僅在本機進行：不得進行網路呼叫、不得讀取鑰匙圈或機密資訊管理員、不得執行 Shell 命令，也不得探查提供者 API。

支援的證據項目：

| 欄位               | 必填 | 類型       | 意義                                                                                                           |
| ------------------ | ---- | ---------- | -------------------------------------------------------------------------------------------------------------- |
| `type`             | 是   | `string`   | 目前為 `local-file-with-env`。                                                                               |
| `fileEnvVar`       | 否   | `string`   | 包含明確認證資訊檔案路徑的環境變數。                                                           |
| `fallbackPaths`    | 否   | `string[]` | 當 `fileEnvVar` 不存在或為空時檢查的本機認證資訊檔案路徑。支援 `${HOME}` 與 `${APPDATA}`。 |
| `requiresAnyEnv`   | 否   | `string[]` | 證據有效前，列出的環境變數中至少一個必須為非空值。                                    |
| `requiresAllEnv`   | 否   | `string[]` | 證據有效前，列出的每個環境變數都必須為非空值。                                           |
| `credentialMarker` | 是   | `string`   | 證據存在時傳回的非機密標記。                                                       |
| `source`           | 否   | `string`   | 驗證／狀態輸出中向使用者顯示的來源標籤。                                                               |

### setup 欄位

| 欄位               | 必填 | 類型       | 含義                                                                                                 |
| ------------------ | -------- | ---------- | --------------------------------------------------------------------------------------------------- |
| `providers`        | 否       | `object[]` | 在設定與初始設定期間公開的提供者設定描述元。                                                         |
| `cliBackends`      | 否       | `string[]` | 設定期間用於優先依描述元查找設定的後端 ID。請確保正規化後的 ID 在全域中唯一。                         |
| `configMigrations` | 否       | `string[]` | 由此外掛設定介面擁有的設定遷移 ID。                                                                  |
| `requiresRuntime`  | 否       | `boolean`  | 依描述元查找後，設定是否仍需執行 `setup-api`。                                                |

## uiHints 參考資料

`uiHints` 是從設定欄位名稱對應至簡要呈現提示的映射。索引鍵可使用點號表示巢狀設定欄位，但任何路徑區段都不得為 `__proto__`、`constructor` 或 `prototype`；設定程序會拒絕這些名稱。

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API 金鑰",
      "help": "用於 OpenRouter 請求",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

每個欄位提示可包含：

| 欄位           | 類型             | 含義                                                                                                              |
| -------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------- |
| `label`        | `string`         | 顯示給使用者的欄位標籤。                                                                                          |
| `help`         | `string`         | 簡短的輔助文字。                                                                                                  |
| `tags`         | `string[]`       | 選用的 UI 標籤。                                                                                                  |
| `advanced`     | `boolean`        | 將欄位標示為進階欄位。                                                                                            |
| `sensitive`    | `boolean`        | 將欄位標示為機密或敏感欄位。                                                                                      |
| `placeholder`  | `string`         | 表單輸入欄位的預留位置文字。                                                                                      |
| `presentation` | `"phone-number"` | 僅供顯示的本地化電話號碼格式，適用於可剖析的國際（`+...`）值；原始值維持不變。                         |

## contracts 參考資料

僅將 `contracts` 用於 OpenClaw 無須匯入外掛執行階段即可讀取的靜態功能擁有權中繼資料。

```json
{
  "contracts": {
    "agentToolResultMiddleware": ["openclaw", "codex"],
    "trustedToolPolicies": ["workflow-budget"],
    "externalAuthProviders": ["acme-ai"],
    "embeddingProviders": ["openai-compatible"],
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "memoryEmbeddingProviders": ["local"],
    "mediaUnderstandingProviders": ["openai"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "musicGenerationProviders": ["stability-audio"],
    "documentExtractors": ["example-docs"],
    "webContentExtractors": ["firecrawl"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "workerProviders": ["example-worker"],
    "usageProviders": ["acme-ai"],
    "migrationProviders": ["hermes"],
    "gatewayMethodDispatch": ["authenticated-request"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

每個清單皆為選用：

| 欄位                             | 類型       | 含義                                                                                                                                 |
| -------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `embeddedExtensionFactories`     | `string[]` | Codex 應用程式伺服器擴充功能工廠 ID，目前為 `codex-app-server`。                                                                     |
| `agentToolResultMiddleware`      | `string[]` | 此外掛可為其註冊工具結果中介軟體的執行階段 ID。                                                                                       |
| `trustedToolPolicies`            | `string[]` | 已安裝外掛可註冊的外掛本機可信任工具執行前政策 ID。隨附外掛無須此欄位即可註冊政策。                                                    |
| `externalAuthProviders`          | `string[]` | 此外掛擁有其外部驗證設定檔掛鉤的提供者 ID。                                                                                           |
| `embeddingProviders`             | `string[]` | 此外掛擁有的一般嵌入提供者 ID，用於可重複使用的向量嵌入，包括記憶體。                                                                 |
| `speechProviders`                | `string[]` | 此外掛擁有的語音提供者 ID。                                                                                                          |
| `realtimeTranscriptionProviders` | `string[]` | 此外掛擁有的即時轉錄提供者 ID。                                                                                                      |
| `realtimeVoiceProviders`         | `string[]` | 此外掛擁有的即時語音提供者 ID。                                                                                                      |
| `memoryEmbeddingProviders`       | `string[]` | 此外掛擁有且已淘汰的記憶體專用嵌入提供者 ID。                                                                                         |
| `mediaUnderstandingProviders`    | `string[]` | 此外掛擁有的媒體理解提供者 ID。                                                                                                      |
| `transcriptSourceProviders`      | `string[]` | 此外掛擁有的逐字稿來源提供者 ID。                                                                                                    |
| `documentExtractors`             | `string[]` | 此外掛擁有的文件（例如 PDF）擷取器提供者 ID。                                                                                         |
| `imageGenerationProviders`       | `string[]` | 此外掛擁有的影像生成提供者 ID。                                                                                                      |
| `videoGenerationProviders`       | `string[]` | 此外掛擁有的影片生成提供者 ID。                                                                                                      |
| `musicGenerationProviders`       | `string[]` | 此外掛擁有的音樂生成提供者 ID。                                                                                                      |
| `webContentExtractors`           | `string[]` | 此外掛擁有的網頁內容擷取提供者 ID。                                                                                                  |
| `webFetchProviders`              | `string[]` | 此外掛擁有的網頁擷取提供者 ID。                                                                                                      |
| `webSearchProviders`             | `string[]` | 此外掛擁有的網頁搜尋提供者 ID。                                                                                                      |
| `workerProviders`                | `string[]` | 此外掛擁有的雲端工作者提供者 ID，用於佈建及由設定檔支援的租約生命週期。                                                               |
| `usageProviders`                 | `string[]` | 此外掛擁有其用量驗證及用量快照掛鉤的提供者 ID。                                                                                       |
| `migrationProviders`             | `string[]` | 此外掛為 `openclaw migrate` 擁有的匯入提供者 ID。                                                                                     |
| `gatewayMethodDispatch`          | `string[]` | 保留權限，用於已驗證且在程序內分派閘道方法的外掛 HTTP 路由。                                                                          |
| `tools`                          | `string[]` | 此外掛擁有的代理程式工具名稱。                                                                                                       |

`contracts.embeddedExtensionFactories` 保留給隨附且僅限 Codex 應用程式伺服器使用的擴充功能工廠。隨附的工具結果轉換應改為宣告 `contracts.agentToolResultMiddleware`，並使用 `api.registerAgentToolResultMiddleware(...)` 註冊。已安裝外掛僅可在明確啟用時使用相同的中介軟體接合面，而且僅限於其在 `contracts.agentToolResultMiddleware` 中宣告的執行階段。

需要主機可信任工具執行前政策層級的已安裝外掛，必須在 `contracts.trustedToolPolicies` 中宣告每個已註冊的本機 ID，並明確啟用。隨附外掛會保留現有的可信任政策路徑，但具有未宣告政策 ID 的已安裝外掛會在註冊前遭到拒絕。政策 ID 的範圍僅限於註冊該 ID 的外掛，因此兩個外掛可以同時宣告並註冊 `workflow-budget`；單一外掛不得重複註冊相同的本機 ID。

執行階段的 `api.registerTool(...)` 註冊必須與 `contracts.tools` 相符。工具探索會使用此清單，僅載入可擁有所要求工具的外掛執行階段。

實作 `resolveExternalAuthProfiles` 的提供者外掛應宣告 `contracts.externalAuthProviders`；未宣告的外部驗證掛鉤會被忽略。

同時實作 `resolveUsageAuth` 與 `fetchUsageSnapshot` 的提供者外掛，應在 `contracts.usageProviders` 中宣告每個自動探索的提供者 ID。用量探索會在載入執行階段程式碼之前讀取此合約，接著僅載入已宣告的擁有者，再驗證兩個掛鉤。

一般嵌入提供者應針對使用 `api.registerEmbeddingProvider(...)` 註冊的每個配接器宣告 `contracts.embeddingProviders`。一般合約用於可重複使用的向量生成，包括記憶體搜尋所使用的提供者。`contracts.memoryEmbeddingProviders` 是已淘汰的記憶體專用相容性機制，僅在現有提供者遷移至通用嵌入提供者接合面期間保留。

工作者提供者必須在 `contracts.workerProviders` 中宣告每個 `api.registerWorkerProvider(...)` ID。核心會在呼叫 `provision` 前保存持久意圖；提供者會在進行外部分配前驗證其設定，使用相同操作 ID 的重複呼叫必須採用相同租約。核心也會保存該已驗證的設定快照，並透過 `leaseId` 將其傳遞給 `inspect({ leaseId, profile })` 與 `destroy({ leaseId, profile })`，即使具名設定檔已變更或移除亦然。銷毀作業具冪等性，檢查會傳回封閉的 `active` / `destroyed` / `unknown` 狀態聯集，而 SSH 私密金鑰資料僅能透過 `SecretRef` 參照。佈建的 SSH 端點還必須包含來自可信任佈建輸出的公開 `hostKey`，其格式必須恰為 `algorithm base64`，不得包含主機名稱或註解，以便核心在連線前固定主機。產生動態身分參照的提供者可實作權威的 `resolveSshIdentity({ leaseId, profile, keyRef })`；未實作的提供者則使用核心的通用機密解析器。權威的 `unknown` 會將作用中的本機記錄標示為孤立；保存銷毀請求後，它會確認拆除完成。

`contracts.gatewayMethodDispatch` 目前接受 `"authenticated-request"`。這是針對原生外掛 HTTP 路由的 API 衛生閘門；這些路由會刻意在程序內分派閘道控制平面方法，而它並不是防範惡意原生外掛的沙箱。僅將它用於已經過嚴格審查、且已要求閘道 HTTP 驗證的內建／操作人員介面。只有當具有權限的路由也宣告 `auth: "gateway"` 與該路由專屬的 `gatewayRuntimeScopeSurface: "trusted-operator"` 時，在閘道根工作接納關閉期間仍可存取該路由；同一外掛的一般同層路由仍會受接納邊界限制。如此可在不授予整個外掛略過接納限制的情況下，讓暫停狀態與恢復功能保持可用。請將剖析與回應塑形限制在分派之外的明確範圍內；實質性或會改變狀態的工作必須透過閘道方法分派執行，由其負責接納與範圍強制執行。

## configContracts 參考

當通用核心輔助程式需要由資訊清單擁有的設定行為，但不應匯入外掛執行階段時，請使用 `configContracts`：危險旗標偵測、SecretRef 遷移目標，以及舊版設定路徑縮限。

```json
{
  "configContracts": {
    "compatibilityMigrationPaths": ["legacyProvider"],
    "compatibilityRuntimePaths": ["legacyProvider.webhook"],
    "dangerousFlags": [
      {
        "path": "accounts.*.allowUnverifiedSenders",
        "equals": true
      }
    ],
    "secretInputs": {
      "bundledDefaultEnabled": false,
      "paths": [
        {
          "path": "routes.*.secret",
          "expected": "string",
          "ownerKind": "route"
        }
      ]
    }
  }
}
```

| 欄位                         | 必填 | 類型       | 意義                                                                                                                                                                                                                          |
| ----------------------------- | -------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `compatibilityMigrationPaths` | 否       | `string[]` | 相對於根的設定路徑，表示此外掛的設定階段相容性遷移可能適用。當設定從未參照該外掛時，通用執行階段設定讀取作業可藉此略過該外掛的所有設定介面。                 |
| `compatibilityRuntimePaths`   | 否       | `string[]` | 此外掛能在外掛程式碼完全啟用前，於執行階段處理的相對於根的相容性路徑。對於應縮小內建候選項集合、但不應匯入每個相容外掛執行階段的舊版介面，請使用此欄位。 |
| `dangerousFlags`              | 否       | `object[]` | 啟用時，`openclaw doctor` 應標示為不安全或危險的設定常值。請參閱下文。                                                                                                                                   |
| `secretInputs`                | 否       | `object`   | `plugins.entries.<id>.config` 下的設定路徑，用於 SecretRef 遷移、稽核、啟動實體化，以及選用的執行階段擁有者隔離。請參閱下文。                                                                             |

每個 `dangerousFlags` 項目支援：

| 欄位    | 必填 | 類型                                  | 意義                                                                                                       |
| -------- | -------- | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `path`   | 是      | `string`                              | 相對於 `plugins.entries.<id>.config`、以點號分隔的設定路徑。支援對應 map／陣列區段的 `*` 萬用字元。 |
| `equals` | 是      | `string \| number \| boolean \| null` | 將此設定值標示為危險的精確常值。                                                            |

`secretInputs` 支援：

| 欄位                   | 必填 | 類型       | 意義                                                                                                                                                                                                                                                                                                                                              |
| ----------------------- | -------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bundledDefaultEnabled` | 否       | `boolean`  | 決定此 SecretRef 介面是否生效時，覆寫內建外掛的預設啟用狀態。當外掛為內建，但此介面應在設定中明確啟用後才生效時，請使用此欄位。                                                                                                                                            |
| `paths`                 | 是      | `object[]` | 具秘密資料形態的設定路徑；每個路徑都有 `path`（以點號分隔、相對於 `plugins.entries.<id>.config`，並支援 `*` 萬用字元）、選用的 `expected`（目前僅支援 `"string"`），以及選用的 `ownerKind`（目前僅支援 `"route"`）。解析失敗時，已宣告的擁有者只會隔離該確切相符的路徑；其擁有者 ID 為完整設定路徑。 |

## mediaUnderstandingProviderMetadata 參考

當媒體理解供應商具有預設模型、自動驗證備援優先順序，或通用核心輔助程式在執行階段載入前需要的原生文件支援時，請使用 `mediaUnderstandingProviderMetadata`。鍵也必須在 `contracts.mediaUnderstandingProviders` 中宣告。

```json
{
  "contracts": {
    "mediaUnderstandingProviders": ["example"]
  },
  "mediaUnderstandingProviderMetadata": {
    "example": {
      "capabilities": ["image", "audio"],
      "defaultModels": {
        "image": "example-vision-latest",
        "audio": "example-transcribe-latest"
      },
      "autoPriority": {
        "image": 40
      },
      "nativeDocumentInputs": ["pdf"],
      "documentModels": {
        "pdf": {
          "textExtraction": "example-doc-text-latest",
          "image": "example-doc-vision-latest"
        }
      }
    }
  }
}
```

每個供應商項目可包含：

| 欄位                  | 類型                                                             | 意義                                                                                                   |
| ---------------------- | ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `capabilities`         | `("image" \| "audio" \| "video")[]`                              | 此供應商公開的媒體能力。                                                                    |
| `defaultModels`        | `Record<string, string>`                                         | 設定未指定模型時使用的能力對模型預設值。                                         |
| `autoPriority`         | `Record<string, number>`                                         | 進行自動、以認證資訊為基礎的供應商備援時，數字越小排序越前。                                    |
| `nativeDocumentInputs` | `"pdf"[]`                                                        | 供應商支援的原生文件輸入。                                                               |
| `documentModels`       | `{ pdf?: { textExtraction?: string; image?: string \| false } }` | 各文件類型的模型覆寫。將 `image: false` 設定為停用該文件類型以影像為基礎的擷取。 |

## channelConfigs 參考

當頻道外掛在執行階段載入前需要低成本的設定中繼資料時，請使用 `channelConfigs`。如果沒有可用的設定項目，或 `setup.requiresRuntime: false` 宣告不需要設定執行階段，唯讀的頻道設定／狀態探索可直接對已設定的外部頻道使用此中繼資料。

`channelConfigs` 是外掛資訊清單中繼資料，而不是新的頂層使用者設定區段。使用者仍在 `channels.<channel-id>` 下設定頻道執行個體。OpenClaw 會讀取資訊清單中繼資料，以便在執行外掛執行階段程式碼前，判斷哪個外掛擁有已設定的頻道。

對於頻道外掛，`configSchema` 與 `channelConfigs` 描述不同的路徑：

- `configSchema` 驗證 `plugins.entries.<plugin-id>.config`
- `channelConfigs.<channel-id>.schema` 驗證 `channels.<channel-id>`

宣告 `channels[]` 的非內建外掛，也應宣告相符的 `channelConfigs` 項目。若未宣告，OpenClaw 仍可載入此外掛，但在外掛執行階段執行前，冷路徑設定結構描述、設定流程及控制介面無法得知該頻道擁有的選項形態或僅供顯示的介面提示。

`channelConfigs.<channel-id>.commands.nativeCommandsAutoEnabled` 與 `nativeSkillsAutoEnabled` 可為在頻道執行階段載入前執行的命令設定檢查，宣告靜態 `auto` 預設值。內建頻道也可透過 `package.json#openclaw.channel.commands`，連同其套件擁有的其他頻道目錄中繼資料一起發布相同的預設值。

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "Homeserver URL",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Matrix homeserver connection",
      "commands": {
        "nativeCommandsAutoEnabled": true,
        "nativeSkillsAutoEnabled": true
      },
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

每個頻道項目可包含：

| 欄位         | 類型                     | 意義                                                                                                    |
| ------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | `channels.<id>` 的 JSON Schema。每個已宣告的頻道設定項目都必須提供。                                |
| `uiHints`     | `Record<string, object>` | 該頻道設定區段的選用標籤、預留位置文字、敏感性，以及僅供顯示的呈現提示。 |
| `label`       | `string`                 | 當執行階段中繼資料尚未就緒時，合併至選擇器與檢查介面的頻道標籤。                        |
| `description` | `string`                 | 用於檢查與目錄介面的簡短頻道說明。                                                      |
| `commands`    | `object`                 | 用於執行階段前設定檢查的原生命令與原生 Skills 自動預設值。                              |
| `preferOver`  | `string[]`               | 此頻道應在選擇介面中優先於其排序的舊版或較低優先順序外掛 ID。                           |

### 取代另一個頻道外掛

當你的外掛是某個頻道 ID 的偏好擁有者，而另一個外掛也能提供該 ID 時，請使用 `preferOver`。常見情況包括重新命名的外掛 ID、取代內建外掛的獨立外掛，或為了設定相容性而保留相同頻道 ID 的持續維護分支。

```json
{
  "id": "acme-chat",
  "channels": ["chat"],
  "channelConfigs": {
    "chat": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "webhookUrl": { "type": "string" }
        }
      },
      "preferOver": ["chat"]
    }
  }
}
```

設定 `channels.chat` 時，OpenClaw 會同時考量頻道 ID 與偏好的外掛 ID。若較低優先順序的外掛僅因其為內建或預設啟用而被選取，OpenClaw 會在有效的執行階段設定中停用該外掛，讓單一外掛擁有該頻道及其工具。使用者明確選擇仍具有優先權：若使用者明確啟用兩個外掛（透過 `plugins.allow` 或實質的 `plugins.entries` 設定），OpenClaw 會保留該選擇並回報頻道／工具重複的診斷資訊，而不是默默變更要求的外掛集合。

請將 `preferOver` 限定於確實能提供相同頻道的外掛 ID。它不是一般用途的優先順序欄位，也不會重新命名使用者設定鍵。

## modelSupport 參考

若 OpenClaw 應在載入外掛執行階段前，從 `gpt-5.6-sol` 或 `claude-sonnet-4.6` 等模型簡寫 ID 推斷你的供應商外掛，請使用 `modelSupport`。

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw 會套用以下優先順序：

- 明確的 `provider/model` 參照會使用擁有者的 `providers` 資訊清單中繼資料
- `modelPatterns` 優先於 `modelPrefixes`
- 若一個非內建外掛和一個內建外掛皆相符，非內建外掛優先
- 其餘歧義會被忽略，直到使用者或設定指定供應商

欄位：

| 欄位            | 類型       | 意義                                                                            |
| --------------- | ---------- | ------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | 使用 `startsWith` 與模型簡寫 ID 比對的前綴。                 |
| `modelPatterns` | `string[]` | 移除設定檔後綴後，與模型簡寫 ID 比對的正規表示式來源。 |

`modelPatterns` 項目會透過 `compileSafeRegex` 編譯；此機制會拒絕包含巢狀重複的模式（例如 `(a+)+$`）。未通過安全性檢查的模式會被默默略過，處理方式與語法無效的正規表示式相同。請保持模式簡潔，並避免使用巢狀量詞。

## modelCatalog 參考

若 OpenClaw 應在載入外掛執行階段前得知供應商模型中繼資料，請使用 `modelCatalog`。這是由資訊清單擁有的固定目錄列、供應商別名、抑制規則及探索模式來源。執行階段重新整理仍屬於供應商執行階段程式碼，但資訊清單會告知核心何時需要執行階段。

```json
{
  "providers": ["openai"],
  "modelCatalog": {
    "providers": {
      "openai": {
        "baseUrl": "https://api.openai.com/v1",
        "api": "openai-responses",
        "models": [
          {
            "id": "gpt-5.4",
            "name": "GPT-5.4",
            "input": ["text", "image"],
            "reasoning": true,
            "contextWindow": 256000,
            "maxTokens": 128000,
            "cost": {
              "input": 1.25,
              "output": 10,
              "cacheRead": 0.125
            },
            "status": "available",
            "tags": ["default"]
          }
        ]
      }
    },
    "aliases": {
      "azure-openai-responses": {
        "provider": "openai",
        "api": "azure-openai-responses"
      }
    },
    "suppressions": [
      {
        "provider": "azure-openai-responses",
        "model": "gpt-5.3-codex-spark",
        "reason": "not available on Azure OpenAI Responses"
      }
    ],
    "discovery": {
      "openai": "static"
    }
  }
}
```

頂層欄位：

| 欄位             | 類型                                                     | 意義                                                                                                        |
| ---------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `providers`      | `Record<string, object>`                                 | 此外掛所擁有之供應商 ID 的目錄列。鍵也應出現在頂層 `providers` 中。       |
| `aliases`        | `Record<string, object>`                                 | 在目錄或抑制規劃中，應解析為所擁有供應商的供應商別名。              |
| `suppressions`   | `object[]`                                               | 此外掛基於供應商特定原因而抑制的其他來源模型列。                  |
| `discovery`      | `Record<string, "static" \| "refreshable" \| "runtime">` | 供應商目錄是否可從資訊清單中繼資料讀取、重新整理至快取，或需要執行階段。 |
| `runtimeAugment` | `boolean`                                                | 僅當供應商執行階段必須在資訊清單／設定規劃後附加目錄列時，才設為 `true`。       |

`aliases` 會參與模型目錄規劃的供應商擁有權查詢。別名目標必須是由同一外掛擁有的頂層供應商。當依供應商篩選的清單使用別名時，OpenClaw 可讀取擁有者的資訊清單，並套用別名的 API／基礎 URL 覆寫，而無須載入供應商執行階段。別名不會展開未篩選的目錄清單；廣泛清單只會輸出擁有者的標準供應商列。

`suppressions` 會取代舊的供應商執行階段 `suppressBuiltInModel` 掛鉤。只有在供應商由此外掛擁有，或宣告為以所擁有供應商為目標的 `modelCatalog.aliases` 鍵時，抑制項目才會生效。模型解析期間不再呼叫執行階段抑制掛鉤。

供應商欄位：

| 欄位                  | 類型                     | 意義                                                                                                                                                                                                              |
| --------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `baseUrl`             | `string`                 | 此供應商目錄中模型的選用預設基礎 URL。                                                                                                                                                    |
| `api`                 | `ModelApi`               | 此供應商目錄中模型的選用預設 API 配接器。                                                                                                                                                 |
| `headers`             | `Record<string, string>` | 套用至此供應商目錄的選用靜態標頭。                                                                                                                                                      |
| `defaultUtilityModel` | `string`                 | 供應商建議用於簡短內部工具工作（標題、進度敘述）的選用小型模型 ID。當未設定 `agents.defaults.utilityModel`，且此供應商提供代理程式的主要模型時使用。 |
| `models`              | `object[]`               | 必要的模型列。缺少 `id` 的列會被忽略。                                                                                                                                                            |

模型欄位：

| 欄位               | 類型                                                           | 意義                                                                        |
| ------------------ | -------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `id`               | `string`                                                       | 供應商本機模型 ID，不含 `provider/` 前綴。                    |
| `name`             | `string`                                                       | 選用的顯示名稱。                                                      |
| `api`              | `ModelApi`                                                     | 選用的個別模型 API 覆寫。                                            |
| `baseUrl`          | `string`                                                       | 選用的個別模型基礎 URL 覆寫。                                       |
| `headers`          | `Record<string, string>`                                       | 選用的個別模型靜態標頭。                                          |
| `input`            | `Array<"text" \| "image" \| "document">`                       | 模型接受的模態。其他值會被默默捨棄。            |
| `reasoning`        | `boolean`                                                      | 模型是否公開推理行為。                               |
| `contextWindow`    | `number`                                                       | 供應商原生的上下文視窗。                                             |
| `contextTokens`    | `number`                                                       | 與 `contextWindow` 不同時，選用的有效執行階段上下文上限。 |
| `maxTokens`        | `number`                                                       | 已知時的最大輸出權杖數。                                           |
| `thinkingLevelMap` | `Record<string, string \| null>`                               | 選用的各思考層級模型 ID 或參數覆寫。                    |
| `cost`             | `object`                                                       | 選用的每百萬權杖美元計價，包括選用的 `tieredPricing`。 |
| `compat`           | `object`                                                       | 與 OpenClaw 模型設定相容性相符的選用相容性旗標。  |
| `mediaInput`       | `object`                                                       | 選用的各模態輸入設定，目前僅支援圖片。                   |
| `status`           | `"available"` \| `"preview"` \| `"deprecated"` \| `"disabled"` | 清單狀態。僅當該列完全不應出現時才加以抑制。          |
| `statusReason`     | `string`                                                       | 與非可用狀態一併顯示的選用原因。                            |
| `replaces`         | `string[]`                                                     | 此模型取代的舊版供應商本機模型 ID。                       |
| `replacedBy`       | `string`                                                       | 已淘汰列的替代供應商本機模型 ID。                    |
| `tags`             | `string[]`                                                     | 選擇器與篩選器使用的穩定標籤。                                    |

抑制欄位：

| 欄位                      | 類型       | 含義                                                                                             |
| -------------------------- | ---------- | --------------------------------------------------------------------------------------------------------- |
| `provider`                 | `string`   | 要隱藏的上游資料列之提供者 ID。必須由此外掛擁有，或宣告為其擁有的別名。 |
| `model`                    | `string`   | 要隱藏的提供者本機模型 ID。                                                                      |
| `reason`                   | `string`   | 直接要求已隱藏資料列時顯示的選用訊息。                                     |
| `when.baseUrlHosts`        | `string[]` | 套用隱藏前所需的有效提供者基底 URL 主機選用清單。               |
| `when.providerConfigApiIn` | `string[]` | 套用隱藏前所需的確切提供者設定 `api` 值選用清單。              |

請勿將僅限執行階段的資料放入 `modelCatalog`。只有當資訊清單資料列足夠完整，可讓依提供者篩選的清單與選擇器介面略過登錄檔／執行階段探索時，才使用 `static`。當資訊清單資料列可作為實用的可列出種子或補充資料，但稍後重新整理／快取可新增更多資料列時，請使用 `refreshable`；可重新整理的資料列本身不具權威性。當 OpenClaw 必須載入提供者執行階段才能得知清單時，請使用 `runtime`。

## modelIdNormalization 參考資料

針對必須在提供者執行階段載入前執行、成本低廉且由提供者擁有的模型 ID 清理，請使用 `modelIdNormalization`。這可將短模型名稱、提供者本機舊版 ID，以及代理前綴規則等別名保留在其所屬外掛的資訊清單中，而非核心模型選擇表內。

```json
{
  "providers": ["anthropic", "openrouter"],
  "modelIdNormalization": {
    "providers": {
      "anthropic": {
        "aliases": {
          "sonnet-4.6": "claude-sonnet-4-6"
        }
      },
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  }
}
```

提供者欄位：

| 欄位                                | 類型                    | 含義                                                                             |
| ------------------------------------ | ----------------------- | ----------------------------------------------------------------------------------------- |
| `aliases`                            | `Record<string,string>` | 不區分大小寫的確切模型 ID 別名。傳回值時會保留原始寫法。                  |
| `stripPrefixes`                      | `string[]`              | 在查詢別名前移除的前綴，適用於舊版提供者／模型重複情況。     |
| `prefixWhenBare`                     | `string`                | 當正規化後的模型 ID 尚未包含 `/` 時要新增的前綴。                  |
| `prefixWhenBareAfterAliasStartsWith` | `object[]`              | 查詢別名後的條件式裸 ID 前綴規則，以 `modelPrefix` 和 `prefix` 為鍵。 |

## providerEndpoints 參考資料

針對通用要求原則必須在提供者執行階段載入前得知的端點分類，請使用 `providerEndpoints`。核心仍負責定義各 `endpointClass` 的含義；外掛資訊清單則擁有主機和基底 URL 中繼資料。

正式外部化的提供者外掛會從核心發行內容中排除，因此在安裝前無法看見其資訊清單。其 `providerEndpoints` 也必須同步至 `scripts/lib/official-external-provider-catalog.json`，如此即使沒有外掛，端點分類仍可正常運作；契約測試會強制兩者保持同步。

端點欄位：

| 欄位                          | 類型       | 含義                                                                                  |
| ------------------------------ | ---------- | ---------------------------------------------------------------------------------------------- |
| `endpointClass`                | `string`   | 已知的核心端點類別，例如 `openrouter`、`moonshot-native` 或 `google-vertex`。        |
| `hosts`                        | `string[]` | 對應至端點類別的確切主機名稱。                                                |
| `hostSuffixes`                 | `string[]` | 對應至端點類別的主機尾碼。加上 `.` 前綴可僅比對網域尾碼。 |
| `baseUrls`                     | `string[]` | 對應至端點類別的確切正規化 HTTP(S) 基底 URL。                             |
| `googleVertexRegion`           | `string`   | 確切全域主機的靜態 Google Vertex 區域。                                            |
| `googleVertexRegionHostSuffix` | `string`   | 從相符主機移除的尾碼，以顯示 Google Vertex 區域前綴。                 |

## providerRequest 參考資料

針對通用要求原則在不載入提供者執行階段的情況下所需、成本低廉的要求相容性中繼資料，請使用 `providerRequest`。請將與行為相關的承載資料重寫保留在提供者執行階段掛鉤或共用的提供者系列輔助工具中。

```json
{
  "providerRequest": {
    "providers": {
      "vllm": {
        "family": "vllm",
        "openAICompletions": {
          "supportsStreamingUsage": true
        }
      }
    }
  }
}
```

提供者欄位：

| 欄位                 | 類型         | 含義                                                                          |
| --------------------- | ------------ | -------------------------------------------------------------------------------------- |
| `family`              | `string`     | 通用要求相容性決策與診斷所使用的提供者系列標籤。 |
| `compatibilityFamily` | `"moonshot"` | 共用要求輔助工具的選用提供者系列相容性分類。              |
| `openAICompletions`   | `object`     | OpenAI 相容的補全要求旗標，目前為 `supportsStreamingUsage`。       |

## secretProviderIntegrations 參考資料

當外掛可以發布可重複使用的 SecretRef exec 提供者預設組態時，請使用 `secretProviderIntegrations`。OpenClaw 會在外掛執行階段載入前讀取此中繼資料，將外掛擁有權儲存在 `secrets.providers.<alias>.pluginIntegration`，並將實際的祕密解析交由 SecretRef 執行階段處理。預設組態只會針對內建外掛，以及從受管理的外掛安裝根目錄中探索到的已安裝外掛（例如透過 git 和 ClawHub 安裝者）公開。

```json
{
  "secretProviderIntegrations": {
    "secret-store": {
      "providerAlias": "team-secrets",
      "displayName": "Team secrets",
      "source": "exec",
      "command": "${node}",
      "args": ["./bin/resolve-secrets.mjs"]
    }
  }
}
```

對應表的鍵是整合 ID。如果省略 `providerAlias`，OpenClaw 會使用整合 ID 作為 SecretRef 提供者別名。提供者別名必須符合一般 SecretRef 提供者別名模式，例如 `team-secrets` 或 `onepassword-work`。

當操作人員選取預設組態時，OpenClaw 會寫入如下的提供者參照：

```json
{
  "secrets": {
    "providers": {
      "team-secrets": {
        "source": "exec",
        "pluginIntegration": {
          "pluginId": "acme-secrets",
          "integrationId": "secret-store"
        }
      }
    }
  }
}
```

啟動／重新載入時，OpenClaw 會載入目前的外掛資訊清單中繼資料、檢查所屬外掛是否已安裝並啟用，以及從資訊清單具體化 exec 命令，藉此解析該提供者。停用或移除外掛會撤銷使用中 SecretRef 的提供者。希望使用獨立 exec 設定的操作人員，仍可直接手動寫入 `command`/`args` 提供者。

目前僅支援 `source: "exec"` 預設組態。`command` 必須是 `${node}`，而 `args[0]` 必須是相對於外掛根目錄的 `./` 解析器指令碼。OpenClaw 會在啟動／重新載入時，將其具體化為目前的 Node 執行檔和外掛內指令碼的絕對路徑。`--require`、`--import`、`--loader`、`--env-file`、`--eval` 和 `--print` 等 Node 選項不屬於資訊清單預設組態契約。需要非 Node 命令的操作人員，可直接設定獨立的手動 exec 提供者。

對於資訊清單預設組態，OpenClaw 會從外掛根目錄推導 `trustedDirs`，而對於 `${node}` 預設組態，則也會從目前 Node 執行檔的目錄推導。資訊清單撰寫的 `trustedDirs` 會被忽略。`timeoutMs`、`noOutputTimeoutMs`、`maxOutputBytes`、`jsonOnly`、`env`、`passEnv` 和 `allowInsecurePath` 等其他 exec 提供者選項，會直接傳遞至一般 SecretRef exec 提供者設定。

## modelPricing 參考資料

當提供者需要在執行階段載入前控制控制平面的定價行為時，請使用 `modelPricing`。閘道定價快取會讀取此中繼資料，而不匯入提供者執行階段程式碼。

```json
{
  "providers": ["ollama", "openrouter"],
  "modelPricing": {
    "providers": {
      "ollama": {
        "external": false
      },
      "openrouter": {
        "openRouter": {
          "passthroughProviderModel": true
        },
        "liteLLM": false
      }
    }
  }
}
```

提供者欄位：

| 欄位        | 類型              | 含義                                                                                      |
| ------------ | ----------------- | -------------------------------------------------------------------------------------------------- |
| `external`   | `boolean`         | 對於絕不應擷取 OpenRouter 或 LiteLLM 定價的本機／自行託管提供者，請設為 `false`。 |
| `openRouter` | `false \| object` | OpenRouter 定價查詢對應。`false` 會停用此提供者的 OpenRouter 查詢。           |
| `liteLLM`    | `false \| object` | LiteLLM 定價查詢對應。`false` 會停用此提供者的 LiteLLM 查詢。                 |

來源欄位：

| 欄位                      | 類型               | 含義                                                                                                        |
| -------------------------- | ------------------ | -------------------------------------------------------------------------------------------------------------------- |
| `provider`                 | `string`           | 當外部目錄提供者 ID 與 OpenClaw 提供者 ID 不同時使用，例如 `zai` 提供者使用 `z-ai`。 |
| `passthroughProviderModel` | `boolean`          | 將包含斜線的模型 ID 視為巢狀提供者／模型參照，適用於 OpenRouter 等代理提供者。       |
| `modelIdTransforms`        | `"version-dots"[]` | 額外的外部目錄模型 ID 變體。`version-dots` 會嘗試如 `claude-opus-4.6` 的點分版本 ID。            |

### OpenClaw 提供者索引

OpenClaw 提供者索引是由 OpenClaw 擁有的預覽中繼資料，適用於外掛可能尚未安裝的提供者。它不屬於外掛資訊清單。外掛資訊清單仍是已安裝外掛的權威來源。提供者索引是內部備援契約；當提供者外掛尚未安裝時，未來的可安裝提供者與安裝前模型選擇器介面將使用此契約。

目錄權威性順序：

1. 使用者設定。
2. 已安裝外掛資訊清單 `modelCatalog`。
3. 透過明確重新整理產生的模型目錄快取。
4. OpenClaw 提供者索引預覽資料列。

Provider Index 不得包含密鑰、啟用狀態、執行階段掛鉤或即時的帳號特定模型資料。其預覽目錄使用與外掛資訊清單相同的 `modelCatalog` 提供者資料列形式，但除非刻意與已安裝的外掛資訊清單保持一致，否則應僅限於穩定的顯示中繼資料，不應包含 `api`、`baseUrl`、定價或相容性旗標等執行階段轉接器欄位。具有即時 `/models` 探索功能的提供者，應透過明確的模型目錄快取路徑寫入重新整理後的資料列，而不是讓一般列表或新手引導呼叫提供者 API。

對於外掛已移出核心或尚未安裝的提供者，Provider Index 項目也可包含可安裝外掛的中繼資料。此中繼資料遵循頻道目錄模式：套件名稱、npm 安裝規格、預期完整性，以及簡單的驗證方式標籤，就足以顯示可安裝的設定選項。外掛安裝完成後，以其資訊清單為準，並忽略該提供者的 Provider Index 項目。

`openclaw doctor --fix` 會將一組小型且封閉的舊版頂層資訊清單能力鍵遷移至 `contracts.*`：`speechProviders`、`mediaUnderstandingProviders`、`imageGenerationProviders` 和 `tools`。這些鍵（或任何其他能力清單）都不再作為頂層資訊清單欄位讀取；一般資訊清單載入僅會辨識 `contracts` 下的這些鍵。

## 資訊清單與 package.json 的比較

這兩個檔案用途不同：

| 檔案                   | 用途                                                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | 在外掛程式碼執行前即必須存在的探索、設定驗證、驗證方式中繼資料及 UI 提示                         |
| `package.json`         | npm 中繼資料、相依套件安裝，以及用於進入點、安裝管控、設定或目錄中繼資料的 `openclaw` 區塊 |

如果不確定某項中繼資料應放在哪裡，請遵循以下規則：

- 如果 OpenClaw 必須在載入外掛程式碼前得知該資訊，請將其放入 `openclaw.plugin.json`
- 如果該資訊與封裝、進入檔案或 npm 安裝行為有關，請將其放入 `package.json`

### 影響探索的 package.json 欄位

部分執行階段前的外掛中繼資料會刻意存放在 `package.json` 的 `openclaw` 區塊下，而非 `openclaw.plugin.json` 中。`openclaw.bundle` 和 `openclaw.bundle.json` 並非 OpenClaw 外掛合約；原生外掛必須使用 `openclaw.plugin.json`，以及下列受支援的 `package.json#openclaw` 欄位。

重要範例：

| 欄位                                                                                      | 意義                                                                                                                                                                             |
| ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                                                      | 宣告原生外掛進入點。必須位於外掛套件目錄內。                                                                                                        |
| `openclaw.runtimeExtensions`                                                               | 宣告已安裝套件的建置後 JavaScript 執行階段進入點。必須位於外掛套件目錄內。                                                                      |
| `openclaw.setupEntry`                                                                      | 僅供設定使用的輕量進入點，用於新手引導、延後的頻道啟動，以及唯讀的頻道狀態／SecretRef 探索。必須位於外掛套件目錄內。      |
| `openclaw.runtimeSetupEntry`                                                               | 宣告已安裝套件的建置後 JavaScript 設定進入點。需要 `setupEntry`、必須存在，且必須位於外掛套件目錄內。                              |
| `openclaw.channel`                                                                         | 簡單的頻道目錄中繼資料，例如標籤、文件路徑、別名及選取文案。                                                                                                      |
| `openclaw.channel.approvalFlags`                                                           | 執行階段載入前可用的封閉式核准行為旗標。`native` 表示頻道擁有原生核准 UI 及同一回合的解析能力。                                                |
| `openclaw.channel.commands`                                                                | 在頻道執行階段載入前，由設定、稽核及命令清單介面使用的靜態原生命令與原生 Skill 自動預設中繼資料。                                               |
| `openclaw.channel.cliAddOptions`                                                           | 由外掛擁有的 `openclaw channels add` 選項。每個項目會宣告 `flags`、`description`、選用的 `defaultValue`，以及用於通用輸入強制轉型的選用 `valueType`（`int` 或 `list`）。 |
| `openclaw.channel.configuredState`                                                         | 輕量的已設定狀態檢查器中繼資料，可在不載入完整頻道執行階段的情況下回答“是否已存在僅使用環境變數的設定？”。                                              |
| `openclaw.channel.persistedAuthState`                                                      | 輕量的持久化驗證檢查器中繼資料，可在不載入完整頻道執行階段的情況下回答“是否已有任何登入狀態？”。                                                    |
| `openclaw.install.clawhubSpec` / `openclaw.install.npmSpec` / `openclaw.install.localPath` | 內建及外部發布外掛的安裝／更新提示。                                                                                                                        |
| `openclaw.install.defaultChoice`                                                           | 有多個安裝來源可用時的偏好安裝路徑。                                                                                                                       |
| `openclaw.install.minHostVersion`                                                          | 支援的最低 OpenClaw 主機版本，使用 `>=2026.3.22` 或 `>=2026.5.1-beta.1` 等 semver 下限。                                                                                  |
| `openclaw.compat.pluginApi`                                                                | 此套件所需的最低 OpenClaw 外掛 API 範圍，使用 `>=2026.5.27` 等 semver 下限。                                                                                      |
| `openclaw.install.expectedIntegrity`                                                       | 預期的 npm dist 完整性字串，例如 `sha512-...`；安裝及更新流程會據此驗證擷取的成品。                                                                 |
| `openclaw.install.allowInvalidConfigRecovery`                                              | 設定無效時，允許使用範圍有限的內建外掛重新安裝復原路徑。                                                                                                            |
| `openclaw.install.requiredPlatformPackages`                                                | 當其鎖定檔平台限制符合目前主機時，必須具體化的 npm 套件別名。                                                                                |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`                          | 允許設定執行階段頻道介面在開始監聽前載入，然後將完整的已設定頻道外掛延後至監聽後啟用。                                                      |

資訊清單中繼資料決定執行階段載入前，新手引導會顯示哪些提供者／頻道／設定選項。`package.json#openclaw.install` 會告知新手引導，當使用者選取其中一個選項時，該如何擷取或啟用該外掛。請勿將安裝提示移至 `openclaw.plugin.json`。

對於 `openclaw.channel.cliAddOptions`，請使用 Commander 的長選項語法，例如 `--initial-sync-limit <n>`。將 `valueType: "int"` 設為解析非負整數，或將 `valueType: "list"` 設為在外掛設定轉接器收到輸入前，依逗號、分號或換行符號分割成字串。省略 `valueType`，即可將已解析的 Commander 值原樣傳遞。

對於非內建外掛來源，`openclaw.install.minHostVersion` 會在安裝及資訊清單登錄載入期間強制執行。無效值會遭拒絕；較新但有效的值，會使較舊的主機略過外部外掛。內建來源外掛視為與主機簽出版本共同進行版本控管。

`openclaw.install.requiredPlatformPackages` 適用於透過選用且針對特定平台的別名提供必要原生二進位檔的 npm 套件。請列出每個受支援平台別名的 npm 純套件名稱。在 npm 安裝期間，OpenClaw 只會驗證鎖定檔限制符合目前主機的已宣告別名。如果 npm 回報成功但省略該別名，OpenClaw 會使用全新快取重試一次；若別名仍然缺少，便會復原安裝。

對於非內建外掛來源，`openclaw.compat.pluginApi` 會在套件安裝期間強制執行。請將它用於套件建置時所依據的 OpenClaw 外掛 SDK／執行階段 API 下限。當外掛套件需要較新的 API，但仍要為其他流程保留較低的安裝提示時，其限制可以比 `minHostVersion` 更嚴格。依預設，OpenClaw 官方版本同步會將現有官方外掛的 API 下限提升至 OpenClaw 發布版本，但若套件刻意支援較舊的主機，僅發布外掛的版本仍可保留較低的下限。請勿只使用套件版本作為相容性合約。`peerDependencies.openclaw` 仍是 npm 套件中繼資料；OpenClaw 使用 `openclaw.compat.pluginApi` 合約進行安裝相容性判斷。

當外掛發布於 ClawHub 時，官方隨選安裝中繼資料應使用 `clawhubSpec`；新手引導會將其視為偏好的遠端來源，並在安裝後記錄 ClawHub 成品資訊。`npmSpec` 則仍作為尚未移至 ClawHub 之套件的相容性備援。

精確的 npm 版本鎖定已存放於 `npmSpec`，例如 `"npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3"`。官方外部目錄項目應將精確規格與 `expectedIntegrity` 配對，確保擷取的 npm 成品不再符合鎖定版本時，更新流程會採取封閉式失敗。為了相容性，互動式新手引導仍會提供受信任的登錄 npm 規格，包括純套件名稱和 dist-tag。目錄診斷可區分精確、浮動、完整性鎖定、缺少完整性、套件名稱不相符及無效預設選項來源。當 `expectedIntegrity` 存在，但沒有可供其鎖定的有效 npm 來源時，也會發出警告。若有 `expectedIntegrity`，安裝／更新流程會強制執行；若省略，則會記錄登錄解析結果，但不進行完整性鎖定。

當狀態、頻道清單或 SecretRef 掃描需要在不載入完整執行階段的情況下識別已設定的帳號時，頻道外掛應提供 `openclaw.setupEntry`。設定進入點應公開頻道中繼資料，以及可安全用於設定的組態、狀態和密鑰轉接器；網路用戶端、閘道監聽器及傳輸執行階段則應保留在主要擴充功能進入點中。

執行階段進入點欄位不會覆寫來源進入點欄位的套件邊界檢查。例如，`openclaw.runtimeExtensions` 無法讓逸出邊界的 `openclaw.extensions` 路徑變得可載入。

`openclaw.install.allowInvalidConfigRecovery` 的適用範圍刻意設得很窄。它不會讓任意損壞的設定變得可安裝。目前，它只允許安裝流程從特定的過時內建外掛升級失敗中復原，例如缺少內建外掛路徑，或同一個內建外掛存在過時的 `channels.<id>` 項目。無關的設定錯誤仍會阻止安裝，並將操作人員導向 `openclaw doctor --fix`。

`openclaw.channel.persistedAuthState` 是小型檢查器模組的套件中繼資料：

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

當設定、doctor、狀態或唯讀存在性流程需要在載入完整頻道外掛前，先進行低成本的是／否驗證探測時，請使用它。持久化驗證狀態並不是已設定的頻道狀態：請勿使用此中繼資料自動啟用外掛、修復執行階段相依性，或判斷是否應載入頻道執行階段。目標匯出應是一個只讀取持久化狀態的小型函式；請勿讓它經過完整的頻道執行階段 barrel。

`openclaw.channel.configuredState` 支援低成本的已設定檢查。當環境變數已足夠時，請優先使用宣告式環境中繼資料：

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "env": {
          "allOf": ["TELEGRAM_BOT_TOKEN"]
        }
      }
    }
  }
}
```

當列出的每個變數都為必要時，請使用 `env.allOf`；只要任一個非空白變數就足夠時，請使用 `env.anyOf`。如果小型非執行階段檢查需要的資訊超出環境中繼資料，請依 `persistedAuthState` 的示範使用 `specifier` 加上 `exportName`；當 `env` 存在時，OpenClaw 會直接使用它而不載入該模組。如果檢查需要完整的設定解析或實際頻道執行階段，請改將該邏輯保留在外掛的 `config.hasConfiguredState` 掛鉤中。

## 探索優先順序（重複的外掛 ID）

OpenClaw 會從三個根目錄探索外掛，並依此順序檢查：隨 OpenClaw 提供的內建外掛、全域安裝根目錄（`~/.openclaw/extensions`）及目前工作區根目錄（`<workspace>/.openclaw/extensions`），再加上任何明確的 `plugins.load.paths` 項目。

如果兩個探索結果具有相同的 `id`，只會保留**優先順序最高**的資訊清單；優先順序較低的重複項目會被捨棄，而不會並列載入。優先順序由高至低如下：

1. **由設定選取** — 在 `plugins.entries.<id>` 中明確釘選的路徑
2. **符合追蹤安裝記錄的全域安裝** — 透過 `openclaw plugin install`/`openclaw plugin update` 安裝，且 OpenClaw 的安裝追蹤可辨識為相同 ID 的外掛，即使該 ID 也屬於內建外掛
3. **內建** — 隨 OpenClaw 提供的外掛
4. **工作區** — 相對於目前工作區探索到的外掛
5. 任何其他探索到的候選項目

影響如下：

- 位於工作區或全域根目錄中、未受追蹤的內建外掛分支版本或過時副本，不會遮蔽內建版本。
- 若要覆寫內建外掛，請針對該 ID 執行 `openclaw plugin install`，使受追蹤的全域安裝優先於內建副本；或透過 `plugins.entries.<id>` 釘選特定路徑，使其憑藉由設定選取的優先順序勝出。
- 捨棄重複項目時會記錄日誌，讓 Doctor 和啟動診斷能指出遭捨棄的副本。
- 在診斷中，由設定選取的重複覆寫會表述為明確覆寫，但仍會顯示警告，讓過時分支版本和意外遮蔽保持可見。

## JSON Schema 要求

- **每個外掛都必須提供 JSON Schema**，即使它不接受任何設定。
- 空白 schema 也可以接受（例如 `{ "type": "object", "additionalProperties": false }`）。
- Schema 會在讀寫設定時驗證，而不是在執行階段驗證。
- 使用新的設定鍵擴充或分支內建外掛時，請同時更新該外掛的 `openclaw.plugin.json` `configSchema`。內建外掛的 schema 採嚴格模式，因此如果在使用者設定中新增 `plugins.entries.<id>.config.myNewKey`，但未將 `myNewKey` 新增至 `configSchema.properties`，就會在外掛執行階段載入前遭到拒絕。

Schema 擴充範例：

```json
{
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "myNewKey": {
        "type": "string"
      }
    }
  }
}
```

## 驗證行為

- 未知的 `channels.*` 鍵屬於**錯誤**，除非該頻道 ID 已由外掛資訊清單宣告。如果相同 ID 也出現在 `plugins.allow`、`plugins.entries` 或 `plugins.installs`（已被參照但目前無法探索的外掛）中，OpenClaw 會改將其降級為**警告**。
- 參照未知外掛 ID 的 `plugins.entries.<id>`、`plugins.allow` 和 `plugins.deny` 屬於**警告**（「已忽略過時的設定項目」），而非錯誤，因此升級以及已移除／重新命名的外掛不會阻止閘道啟動。
- 參照未知外掛 ID 的 `plugins.slots.memory` 屬於**錯誤**，但已知的官方外部外掛 `memory-lancedb` 除外；對它只會顯示警告。
- 如果外掛已安裝，但其資訊清單或 schema 損壞或遺失，驗證就會失敗，且 Doctor 會回報外掛錯誤。
- 如果外掛設定存在，但外掛已**停用**，設定會予以保留，且 Doctor 與日誌中會顯示**警告**。

如需完整的 `plugins.*` schema，請參閱[設定參考](/zh-TW/gateway/configuration)。

## 注意事項

- **原生 OpenClaw 外掛必須具備資訊清單**，包括從本機檔案系統載入的外掛。執行階段仍會另外載入外掛模組；資訊清單只用於探索與驗證。
- 原生資訊清單使用 JSON5 解析，因此只要最終值仍為物件，就可以使用註解、尾端逗號及未加引號的鍵。
- 資訊清單載入器只會讀取有文件記載的資訊清單欄位。請避免使用自訂的頂層鍵。
- 如果外掛不需要 `channels`、`providers`、`cliBackends` 和 `skills`，可以全部省略。
- `providerCatalogEntry` 必須保持輕量，且不應匯入廣泛的執行階段程式碼；請將它用於靜態提供者目錄中繼資料或範圍明確的探索描述元，而非請求期間的執行。
- 互斥外掛種類透過 `plugins.slots.*` 選取：`kind: "memory"` 透過 `plugins.slots.memory`（預設為 `memory-core`），`kind: "context-engine"` 透過 `plugins.slots.contextEngine`（預設為 `legacy`）。
- 請在此資訊清單中宣告互斥外掛種類。執行階段進入點的 `OpenClawPluginDefinition.kind` 已被淘汰，僅保留作為舊版外掛的相容性後備機制。
- `setup.providers[].envVars` 中的環境變數中繼資料僅供宣告使用。狀態、稽核、排程傳遞驗證及其他唯讀介面，在將環境變數視為已設定之前，仍會套用外掛信任與有效啟用原則。
- 如需需要提供者程式碼的執行階段精靈中繼資料，請參閱[提供者執行階段掛鉤](/zh-TW/plugins/architecture-internals#provider-runtime-hooks)。
- 如果你的外掛依賴原生模組，請記載建置步驟及任何套件管理器允許清單要求（例如 pnpm `allow-build-scripts` + `pnpm rebuild <package>`）。

## 相關內容

<CardGroup cols={3}>
  <Card title="建置外掛" href="/zh-TW/plugins/building-plugins" icon="rocket">
    開始使用外掛。
  </Card>
  <Card title="外掛架構" href="/zh-TW/plugins/architecture" icon="diagram-project">
    內部架構與能力模型。
  </Card>
  <Card title="SDK 概覽" href="/zh-TW/plugins/sdk-overview" icon="book">
    外掛 SDK 參考與子路徑匯入。
  </Card>
</CardGroup>
