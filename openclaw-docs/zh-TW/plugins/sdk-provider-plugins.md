---
read_when:
    - 你正在建立新的模型提供者外掛
    - 你想要將 OpenAI 相容的代理伺服器或自訂 LLM 新增至 OpenClaw
    - 你需要瞭解提供者驗證、目錄與執行階段掛鉤機制
sidebarTitle: Provider plugins
summary: 建立 OpenClaw 模型供應商外掛的逐步指南
title: 建置提供者外掛
x-i18n:
    generated_at: "2026-07-26T08:07:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d175fafc034bd52e996d47e047df104f079f2aba66662b22e8dbdf6c21e7e0
    source_path: plugins/sdk-provider-plugins.md
    workflow: 16
---

建置供應商外掛，為 OpenClaw 新增模型供應商（LLM）：模型
目錄、API 金鑰驗證，以及動態模型解析。

<Info>
  第一次接觸 OpenClaw 外掛嗎？請先閱讀[入門指南](/zh-TW/plugins/building-plugins)，
  以瞭解套件結構和資訊清單設定。
</Info>

<Tip>
  供應商外掛會將模型新增至 OpenClaw 的一般推論迴圈。如果模型
  必須透過原生代理程式常駐程式執行，且該常駐程式負責執行緒、壓縮
  或工具事件，請將供應商與[代理程式
  控制框架](/zh-TW/plugins/sdk-agent-harness)搭配使用，而不要將常駐程式通訊協定
  的詳細資訊放入核心。
</Tip>

## 操作說明

<Steps>
  <Step title="套件與資訊清單">
    ### 步驟 1：套件與資訊清單

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-ai",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "providers": ["acme-ai"],
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
      "id": "acme-ai",
      "name": "Acme AI",
      "description": "Acme AI model provider",
      "providers": ["acme-ai"],
      "modelSupport": {
        "modelPrefixes": ["acme-"]
      },
      "setup": {
        "providers": [
          {
            "id": "acme-ai",
            "envVars": ["ACME_AI_API_KEY"]
          }
        ]
      },
      "providerAuthAliases": {
        "acme-ai-coding": "acme-ai"
      },
      "providerAuthChoices": [
        {
          "provider": "acme-ai",
          "method": "api-key",
          "choiceId": "acme-ai-api-key",
          "choiceLabel": "Acme AI API key",
          "groupId": "acme-ai",
          "groupLabel": "Acme AI",
          "cliFlag": "--acme-ai-api-key",
          "cliOption": "--acme-ai-api-key <key>",
          "cliDescription": "Acme AI API key"
        }
      ],
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    `setup.providers[].envVars` 可讓 OpenClaw 在不載入外掛執行階段的情況下偵測認證資訊。
    當某個供應商變體應重複使用另一個供應商 ID 的驗證時，請新增 `providerAuthAliases`。
    `modelSupport` 為選用設定，可讓 OpenClaw 在執行階段掛鉤尚不存在前，從
    `acme-large` 之類的模型簡寫 ID 自動載入你的供應商外掛。ClawHub
    發布需要 `package.json` 中的 `openclaw.compat` 和 `openclaw.build`
    （`openclaw.compat.pluginApi` 和 `openclaw.build.openclawVersion`
    是兩個必填欄位；省略 `minGatewayVersion` 時會改用
    `openclaw.install.minHostVersion`）。

  </Step>

  <Step title="註冊供應商">
    最基本的文字供應商需要 `id`、`label`、`auth` 和 `catalog`。
    `catalog` 是供應商擁有的執行階段／設定掛鉤；它可以呼叫即時
    廠商 API，並傳回 `models.providers` 項目。

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth";

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          docsPath: "/providers/acme-ai",
          envVars: ["ACME_AI_API_KEY"],

          auth: [
            createProviderApiKeyAuthMethod({
              providerId: "acme-ai",
              methodId: "api-key",
              label: "Acme AI API key",
              hint: "API key from your Acme AI dashboard",
              optionKey: "acmeAiApiKey",
              flagName: "--acme-ai-api-key",
              envVar: "ACME_AI_API_KEY",
              promptMessage: "Enter your Acme AI API key",
              defaultModel: "acme-ai/acme-large",
            }),
          ],

          catalog: {
            order: "simple",
            run: async (ctx) => {
              const apiKey =
                ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: {
                  baseUrl: "https://api.acme-ai.com/v1",
                  apiKey,
                  api: "openai-completions",
                  models: [
                    {
                      id: "acme-large",
                      name: "Acme Large",
                      reasoning: true,
                      input: ["text", "image"],
                      cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
                      contextWindow: 200000,
                      maxTokens: 32768,
                    },
                    {
                      id: "acme-small",
                      name: "Acme Small",
                      reasoning: false,
                      input: ["text"],
                      cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
                      contextWindow: 128000,
                      maxTokens: 8192,
                    },
                  ],
                },
              };
            },
          },
        });

        api.registerModelCatalogProvider({
          provider: "acme-ai",
          kinds: ["text"],
          liveCatalog: async (ctx) => {
            const apiKey = ctx.resolveProviderApiKey("acme-ai").apiKey;
            if (!apiKey) return null;
            return [
              {
                kind: "text",
                provider: "acme-ai",
                model: "acme-large",
                label: "Acme Large",
                source: "live",
              },
            ];
          },
        });
      },
    });
    ```

    `registerModelCatalogProvider` 是較新的控制平面目錄介面，
    用於清單／說明／選擇器 UI，涵蓋 `text`、`voice`、`image_generation`、
    `video_generation` 和 `music_generation` 資料列。請將廠商端點
    呼叫及回應對應保留在外掛中；OpenClaw 負責共用資料列
    形狀、來源標籤和說明呈現。

    這樣就完成一個可運作的供應商。使用者現在可以執行
    `openclaw onboard --acme-ai-api-key <key>`，並選取
    `acme-ai/acme-large` 作為模型。

    ### 即時模型探索

    如果你的供應商提供與 OpenAI 相容的 `/models` API，請讓
    單一供應商輔助程式加入共用探索：

    ```typescript
    catalog: {
      buildProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      buildStaticProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      liveModelDiscovery: true,
    },
    ```

    `liveModelDiscovery: true` 是公開的外掛 SDK 契約，具有以下
    行為：

    | 範圍 | 契約 |
    | --- | --- |
    | 認證資訊 | 探索會使用目錄解析出的供應商認證資訊；若驗證提供 `discoveryApiKey`，則優先使用它。絕不會將機密參照標記作為權杖傳送。預設要求使用 `Authorization: Bearer <token>`；其他廠商驗證配置請使用 `buildRequestHeaders`。 |
    | 端點 | 預設 URL 是相對於有效供應商 `baseUrl` 的 `models`，包括啟用 `allowExplicitBaseUrl` 時的操作員覆寫值。其他相對路徑請使用 `endpointPath`。僅對固定的廠商 URL 使用 `endpointUrl: { url, requireBaseUrl }`；除非有效基底 URL 仍等於 `requireBaseUrl`，否則會略過探索，以免將自訂 Proxy 的認證資訊傳送給廠商。 |
    | 網路限制 | 擷取作業使用 OpenClaw 的 SSRF 防護機制，整個分頁共用一個 5 秒逾時額度，每頁回應上限為 4 MiB，且最多 50 頁。跨來源分頁連結會遭拒絕；跨來源重新導向後會移除認證資訊。 |
    | 快取 | 成功且非空的目錄會依供應商、端點和解析出的認證資訊快取 60 秒。空白或無法使用的結果不會快取。 |
    | 篩選 | 完全相符的即時 ID 會保留其可信任的靜態中繼資料。新資料列會保守地投射為文字／聊天模型。已停用、已封存、已淘汰、明確非聊天、嵌入、重新排序、內容審核、語音、僅影像及僅視訊的資料列會遭排除。只有在從非標準回應信封選取資料列時，才使用 `readRows`；供應商特定的模型語意仍應放在自訂目錄中。 |
    | 失敗 | 即時探索僅供參考。驗證、網路、逾時、分頁、剖析、空目錄和篩選失敗時，會傳回供應商擁有的靜態種子，而不會移除該供應商。 |

    對於非 Bearer 或非標準的清單端點，請傳遞選項，而不是
    `true`：

    ```typescript
    liveModelDiscovery: {
      endpointPath: "model-catalog",
      buildRequestHeaders: ({ apiKey, discoveryApiKey }) => ({
        "vendor-version": "2026-01-01",
        "x-api-key": discoveryApiKey ?? apiKey ?? "",
      }),
      readRows: (body) =>
        body && typeof body === "object" &&
        Array.isArray((body as { models?: unknown }).models)
          ? (body as { models: unknown[] }).models
          : [],
    },
    ```

    請勿將 `endpointUrl` 當作無條件使用的替代主機。其
    `requireBaseUrl` 檢查是認證資訊隔離邊界，適用於模型清單
    主機不同於推論主機的供應商。

    如果供應商需要自訂模型語意，而不是保守的
    OpenAI 相容投射，請將該投射保留在外掛中，並使用
    `openclaw/plugin-sdk/provider-catalog-live-runtime` 處理共用擷取
    生命週期。此輔助程式提供受保護的 HTTP 擷取、供應商驗證標頭、
    結構化 HTTP 錯誤、TTL 快取和靜態備援行為，且不會
    將供應商政策放入 OpenClaw 核心。

    當即時 API 只會告訴你目前有哪些
    供應商擁有的靜態目錄資料列可用時，請使用 `buildLiveModelProviderConfig`：

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      buildLiveModelProviderConfig,
      type LiveModelCatalogFetchGuard,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    const STATIC_MODELS = [
      {
        id: "acme-large",
        name: "Acme Large",
        reasoning: true,
        input: ["text", "image"],
        cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
        contextWindow: 200000,
        maxTokens: 32768,
      },
      {
        id: "acme-small",
        name: "Acme Small",
        reasoning: false,
        input: ["text"],
        cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
        contextWindow: 128000,
        maxTokens: 8192,
      },
    ] as const;

    async function buildAcmeLiveProvider(params: {
      apiKey: string;
      discoveryApiKey?: string;
      fetchGuard?: LiveModelCatalogFetchGuard;
    }) {
      return await buildLiveModelProviderConfig({
        providerId: "acme-ai",
        endpoint: "https://api.acme-ai.com/v1/models",
        providerConfig: {
          baseUrl: "https://api.acme-ai.com/v1",
          api: "openai-completions",
        },
        models: STATIC_MODELS,
        apiKey: params.apiKey,
        discoveryApiKey: params.discoveryApiKey,
        fetchGuard: params.fetchGuard,
        ttlMs: 60_000,
        auditContext: "acme-ai-model-discovery",
      });
    }

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          catalog: {
            order: "simple",
            run: async (ctx) => {
              const auth = ctx.resolveProviderAuth("acme-ai");
              const apiKey =
                auth.apiKey ?? ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: await buildAcmeLiveProvider({
                  apiKey,
                  discoveryApiKey: auth.discoveryApiKey,
                }),
              };
            },
          },
          staticCatalog: {
            order: "simple",
            run: async () => ({
              provider: {
                baseUrl: "https://api.acme-ai.com/v1",
                api: "openai-completions",
                models: [...STATIC_MODELS],
              },
            }),
          },
        });
      },
    });
    ```

    當供應商 API 傳回更豐富的中繼資料，且外掛需要自行將資料列投射為 OpenClaw 模型定義時，請使用 `getCachedLiveProviderModelRows`：

    ```typescript index.ts
    import {
      getCachedLiveProviderModelRows,
      LiveModelCatalogHttpError,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    async function discoverAcmeModels(apiKey: string) {
      try {
        const rows = await getCachedLiveProviderModelRows({
          providerId: "acme-ai",
          endpoint: "https://api.acme-ai.com/v1/models",
          apiKey,
          ttlMs: 60_000,
          auditContext: "acme-ai-model-discovery",
        });
        return rows
          .map((row) => projectAcmeModel(row))
          .filter((model) => model !== null);
      } catch (error) {
        if (error instanceof LiveModelCatalogHttpError) {
          return STATIC_MODELS;
        }
        throw error;
      }
    }
    ```

    `run` 應維持受驗證機制控管，且沒有可用的認證資訊時應傳回 `null`。請保留離線 `staticRun` 或靜態備援，讓設定、文件、測試和選擇器介面不需依賴即時網路存取。請使用適合模型清單時效性的 TTL，避免在請求期間輪詢檔案系統，並且僅在上游回應不是 OpenAI 相容的 `{ data: [{ id, object }] }` 形態時，才傳入供應商專屬的 `readRows` / `readModelId`。

    如果上游供應商使用與 OpenClaw 不同的控制權杖，請新增小型雙向文字轉換，而不是取代串流路徑：

    ```typescript
    api.registerTextTransforms({
      input: [
        { from: /red basket/g, to: "blue basket" },
        { from: /paper ticket/g, to: "digital ticket" },
        { from: /left shelf/g, to: "right shelf" },
      ],
      output: [
        { from: /blue basket/g, to: "red basket" },
        { from: /digital ticket/g, to: "paper ticket" },
        { from: /right shelf/g, to: "left shelf" },
      ],
    });
    ```

    `input` 會在傳輸前重寫最終系統提示和文字訊息內容。`output` 會在 OpenClaw 剖析自身的控制標記或傳遞至頻道前，重寫助理文字增量與最終文字。

    對於僅註冊一個採用 API 金鑰驗證的文字供應商，以及單一目錄支援執行階段的內建供應商，請優先使用範圍較窄的 `defineSingleProviderPluginEntry(...)` 輔助函式：

    ```typescript
    import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry";

    export default defineSingleProviderPluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      provider: {
        label: "Acme AI",
        docsPath: "/providers/acme-ai",
        auth: [
          {
            methodId: "api-key",
            label: "Acme AI API key",
            hint: "API key from your Acme AI dashboard",
            optionKey: "acmeAiApiKey",
            flagName: "--acme-ai-api-key",
            envVar: "ACME_AI_API_KEY",
            promptMessage: "Enter your Acme AI API key",
            defaultModel: "acme-ai/acme-large",
          },
        ],
        catalog: {
          buildProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
          buildStaticProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
        },
      },
    });
    ```

    `buildProvider` 是 OpenClaw 能解析實際供應商驗證資訊時使用的即時目錄路徑。它可執行供應商專屬的探索。`buildStaticProvider` 僅能用於設定驗證前可安全顯示的離線資料列；它不得要求認證資訊或發出網路請求。OpenClaw 的 `models list --all` 顯示目前僅會針對內建供應商外掛執行靜態目錄，並使用空白設定、空白環境，且不提供代理程式／工作區路徑。

    如果你的驗證流程也需要在上線引導期間修補 `models.providers.*`、別名和代理程式預設模型，請使用 `openclaw/plugin-sdk/provider-onboard` 中的預設輔助函式。範圍最窄的輔助函式為 `createDefaultModelPresetAppliers(...)`、`createDefaultModelsPresetAppliers(...)` 和 `createModelCatalogPresetAppliers(...)`。

    當供應商的原生端點在一般 `openai-completions` 傳輸上支援串流使用量區塊時，請優先使用 `openclaw/plugin-sdk/provider-catalog-shared` 中的共用目錄輔助函式，而不是將供應商 ID 檢查寫死。`supportsNativeStreamingUsageCompat(...)` 和 `applyProviderNativeStreamingUsageCompat(...)` 會從端點能力對應表偵測支援情況，因此即使外掛使用自訂供應商 ID，原生 Moonshot／DashScope 風格端點仍可選擇啟用。

    上述即時探索範例涵蓋 `/models` 風格的供應商 API。請將該探索保留在 `catalog.run` 內並限制於可用驗證資訊，且讓 `staticRun` 不使用網路，以便產生離線目錄。

  </Step>

  <Step title="新增動態模型解析">
    如果你的供應商接受任意模型 ID（例如 Proxy 或路由器），請新增 `resolveDynamicModel`：

    ```typescript
    api.registerProvider({
      // ... id, label, auth, catalog from above

      resolveDynamicModel: (ctx) => ({
        id: ctx.modelId,
        name: ctx.modelId,
        provider: "acme-ai",
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        reasoning: false,
        input: ["text"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 8192,
      }),
    });
    ```

    如果解析需要網路呼叫，請使用 `prepareDynamicModel` 進行非同步預熱；完成後，`resolveDynamicModel` 會再次執行。

  </Step>

  <Step title="新增執行階段掛鉤（視需要）">
    大多數供應商只需要 `catalog` + `resolveDynamicModel`。請依供應商需求逐步新增掛鉤。

    共用輔助建構器目前涵蓋最常見的重播／工具相容性系列，因此外掛通常不需要逐一手動連接每個掛鉤：

    ```typescript
    import { buildProviderReplayFamilyHooks } from "openclaw/plugin-sdk/provider-model-shared";
    import { buildProviderStreamFamilyHooks } from "openclaw/plugin-sdk/provider-stream";
    import { buildProviderToolCompatFamilyHooks } from "openclaw/plugin-sdk/provider-tools";

    const GOOGLE_FAMILY_HOOKS = {
      ...buildProviderReplayFamilyHooks({ family: "google-gemini" }),
      ...buildProviderStreamFamilyHooks("google-thinking"),
      ...buildProviderToolCompatFamilyHooks("gemini"),
    };

    api.registerProvider({
      id: "acme-gemini-compatible",
      // ...
      ...GOOGLE_FAMILY_HOOKS,
    });
    ```

    目前可用的重播系列：

    | 系列 | 連接的功能 | 內建範例 |
    | --- | --- | --- |
    | `openai-compatible` | OpenAI 相容傳輸的共用 OpenAI 風格重播原則，包括工具呼叫 ID 清理、助理優先排序修正，以及傳輸需要時的通用 Gemini 輪次驗證 | `moonshot`、`ollama`、`xai`、`zai` |
    | `anthropic-by-model` | 由 `modelId` 選擇的 Claude 感知重播原則，因此只有在解析出的模型確實是 Claude ID 時，Anthropic 訊息傳輸才會取得 Claude 專屬的思考區塊清理 | `amazon-bedrock` |
    | `native-anthropic-by-model` | 與 `anthropic-by-model` 相同的依模型套用 Claude 原則，另加工具呼叫 ID 清理，以及為必須保留供應商原生 ID 的傳輸保留原生 Anthropic 工具使用 ID | `anthropic-vertex`、`clawrouter` |
    | `google-gemini` | 原生 Gemini 重播原則與啟動重播清理。共用系列會讓文字輸出的 Gemini 命令列介面採用標記式推理；直接的 `google` 供應商會將 `resolveReasoningOutputMode` 覆寫為 `native`，因為 Gemini API 的思考內容會以原生思考部分送達。 | `google`、`google-gemini-cli` |
    | `passthrough-gemini` | 透過 OpenAI 相容 Proxy 傳輸執行 Gemini 模型時使用的 Gemini 思考簽章清理；不會啟用原生 Gemini 重播驗證或啟動重寫 | `openrouter`、`kilocode`、`opencode`、`opencode-go` |
    | `hybrid-anthropic-openai` | 適用於在單一外掛中混合 Anthropic 訊息與 OpenAI 相容模型介面的供應商之混合原則；選用的僅限 Claude 思考區塊捨棄功能仍限定於 Anthropic 端 | `minimax` |

    目前可用的串流系列：

    | 系列 | 接入內容 | 內建範例 |
    | --- | --- | --- |
    | `google-thinking` | 共用串流路徑上的 Gemini 思考承載資料正規化 | `google`、`google-gemini-cli` |
    | `kilocode-thinking` | 共用代理串流路徑上的 Kilo 推理包裝函式，且 `kilo-auto/balanced` 與不支援的代理推理 ID 會略過注入的思考內容 | `kilocode` |
    | `moonshot-thinking` | 從設定與 `/think` 層級對應 Moonshot 二元原生思考承載資料 | `moonshot` |
    | `minimax-fast-mode` | 共用串流路徑上的 MiniMax 快速模式模型重寫 | `minimax`、`minimax-portal` |
    | `openai-responses-defaults` | 共用的原生 OpenAI/Codex Responses 包裝函式：歸屬標頭、`/fast`/`serviceTier`、文字詳細程度、原生 Codex 網頁搜尋、推理相容性承載資料塑形，以及 Responses 上下文管理 | `openai` |
    | `openrouter-thinking` | 代理路由的 OpenRouter 推理包裝函式，集中處理不支援模型／`auto` 的略過情況 | `openrouter` |
    | `tool-stream-default-on` | 適用於 Z.AI 等供應商且預設啟用的 `tool_stream` 包裝函式，除非明確停用，否則會使用工具串流 | `zai` |

    <Accordion title="驅動系列建構器的 SDK 接合面">
      每個系列建構器都由同一套件匯出的較低階公開輔助函式組成；當供應商需要偏離常見模式時，可以使用這些函式：

      - `openclaw/plugin-sdk/provider-model-shared` - `ProviderReplayFamily`、`buildProviderReplayFamilyHooks(...)`，以及原始重播建構器（`buildOpenAICompatibleReplayPolicy`、`buildAnthropicReplayPolicyForModel`、`buildGoogleGeminiReplayPolicy`、`buildHybridAnthropicOrOpenAIReplayPolicy`）。另會匯出 Gemini 重播輔助函式（`sanitizeGoogleGeminiReplayHistory`、`resolveTaggedReasoningOutputMode`）與端點／模型輔助函式（`resolveProviderEndpoint`、`normalizeProviderId`、`normalizeGooglePreviewModelId`）。
      - `openclaw/plugin-sdk/provider-stream` - `ProviderStreamFamily`、`buildProviderStreamFamilyHooks(...)`、`composeProviderStreamWrappers(...)`，以及共用的 OpenAI/Codex 包裝函式（`createOpenAIAttributionHeadersWrapper`、`createOpenAIFastModeWrapper`、`createOpenAIServiceTierWrapper`、`createOpenAIResponsesContextManagementWrapper`、`createCodexNativeWebSearchWrapper`）、與 OpenAI 相容的 DeepSeek V4 包裝函式（`createDeepSeekV4OpenAICompatibleThinkingWrapper`）、Anthropic Messages 思考預填清理（`createAnthropicThinkingPrefillPayloadWrapper`）、純文字工具呼叫相容性（`createPlainTextToolCallCompatWrapper`），以及共用的代理／供應商包裝函式（`createOpenRouterWrapper`、`createToolStreamWrapper`、`createMinimaxFastModeWrapper`）。
      - `openclaw/plugin-sdk/provider-stream-shared` - 適用於供應商熱門路徑的輕量承載資料與事件包裝函式，包括 `createOpenAICompatibleCompletionsThinkingOffWrapper`、`createPayloadPatchStreamWrapper`、`createPlainTextToolCallCompatWrapper`、`normalizeOpenAICompatibleReasoningPayload(...)` 與 `setQwenChatTemplateThinking(...)`。
      - `openclaw/plugin-sdk/provider-tools` - `ProviderToolCompatFamily`、`buildProviderToolCompatFamilyHooks("deepseek" | "gemini" | "openai")`，以及底層供應商結構描述輔助函式。

      對於 Gemini 系列供應商，請讓推理輸出模式與
      傳輸方式保持一致。直接使用 Google Gemini API 的供應商應使用 `native`
      推理輸出，讓 OpenClaw 在不新增
      `<think>` / `<final>` 提示詞指令的情況下使用原生思考部分。僅文字、採 Gemini CLI 風格且
      解析最終 JSON／文字回應的後端，可以繼續使用共用的
      `google-gemini` 標記式合約。

      部分串流輔助函式會刻意保留在供應商本機。`@openclaw/anthropic-provider` 將 `wrapAnthropicProviderStream`、`resolveAnthropicBetas`、`resolveAnthropicFastMode`、`resolveAnthropicServiceTier` 與較低階的 Anthropic 包裝函式建構器保留在自己的公開 `api.ts` / `contract-api.ts` 接合面中，因為它們會編碼 Claude OAuth Beta 處理與 `context1m` 閘控。xAI 外掛同樣將原生 xAI Responses 塑形保留在自己的 `wrapStreamFn` 中（`/fast` 別名、預設 `tool_stream`、不支援的嚴格工具清理、xAI 專用推理承載資料移除）。

      相同的套件根目錄模式也支援 `@openclaw/openai-provider`（供應商建構器、預設模型輔助函式、即時供應商建構器）與 `@openclaw/openrouter-provider`（供應商建構器及上線引導／設定輔助函式）。
    </Accordion>

    <Tabs>
      <Tab title="權杖交換">
        適用於每次推論呼叫前都需要交換權杖的供應商：

        ```typescript
        prepareRuntimeAuth: async (ctx) => {
          const exchanged = await exchangeToken(ctx.apiKey);
          return {
            apiKey: exchanged.token,
            baseUrl: exchanged.baseUrl,
            expiresAt: exchanged.expiresAt,
          };
        },
        ```
      </Tab>
      <Tab title="自訂標頭">
        適用於需要自訂請求標頭或修改主體的供應商：

        ```typescript
        // wrapStreamFn 會傳回衍生自 ctx.streamFn 的 StreamFn
        wrapStreamFn: (ctx) => {
          if (!ctx.streamFn) return undefined;
          const inner = ctx.streamFn;
          return async (params) => {
            params.headers = {
              ...params.headers,
              "X-Acme-Version": "2",
            };
            return inner(params);
          };
        },
        ```
      </Tab>
      <Tab title="原生傳輸身分">
        適用於在
        通用 HTTP 或 WebSocket 傳輸上需要原生請求／工作階段標頭或中繼資料的供應商：

        ```typescript
        resolveTransportTurnState: (ctx) => ({
          headers: {
            "x-request-id": ctx.turnId,
          },
          metadata: {
            session_id: ctx.sessionId ?? "",
            turn_id: ctx.turnId,
          },
        }),
        resolveWebSocketSessionPolicy: (ctx) => ({
          headers: {
            "x-session-id": ctx.sessionId ?? "",
          },
          degradeCooldownMs: 60_000,
        }),
        ```
      </Tab>
      <Tab title="用量與計費">
        適用於提供用量／計費資料的供應商：

        ```typescript
        resolveUsageAuth: async (ctx) => {
          const auth = await ctx.resolveOAuthToken();
          return auth ? { token: auth.token } : null;
        },
        fetchUsageSnapshot: async (ctx) => {
          return await fetchAcmeUsage(ctx.token, ctx.timeoutMs);
        },
        ```

        `resolveUsageAuth` 有三種結果。當
        供應商具備用量／計費認證資訊時，傳回
        `{ token, accountId?, subscriptionType?, rateLimitTier? }`（選用欄位會將已解析設定檔中的
        非機密方案中繼資料帶入
        `fetchUsageSnapshot`）。只有在供應商已明確處理用量
        驗證，但沒有可用的用量權杖，且 OpenClaw 必須略過通用
        API 金鑰／OAuth 後援時，才傳回
        `{ handled: true }`。當供應商未
        處理請求，而 OpenClaw 應繼續使用通用後援時，傳回 `null` 或 `undefined`。

        在 `contracts.usageProviders` 中宣告供應商 ID。當該資訊清單
        合約與 **兩個** 鉤子都存在時，OpenClaw 會自動將
        供應商納入用量收集，而不載入不相關的供應商
        外掛。不需要更新核心允許清單。
        `fetchUsageSnapshot` 會傳回共用且不依賴特定供應商的結構：

        - `plan`：供應商回報的訂閱或金鑰標籤
        - `windows`：以已使用百分比表示的可重設配額時段
        - `billing`：具型別的 `balance`、`spend` 或 `budget` 項目；`unit` 可以是
          ISO 貨幣或 `credits` 等供應商單位
        - `summary`：無法納入這些
          結構化欄位的精簡供應商特定上下文

        請精確保留貨幣語意。除非
        上游合約如此定義，否則供應商點數並不等同於 USD。僅實作
        `fetchUsageSnapshot` 的外掛仍可供明確／合成呼叫端使用，但
        不會被自動探索，因為 OpenClaw 無法解析其用量認證資訊。
      </Tab>
    </Tabs>

    <Accordion title="常見供應商鉤子">
      對於模型／供應商外掛，OpenClaw 大致依照以下順序呼叫鉤子。
      多數供應商只會使用其中 2 至 3 個。這不是完整的 `ProviderPlugin`
      合約；如需完整且目前準確的鉤子清單與後援說明，請參閱[內部機制：供應商執行階段
      鉤子](/zh-TW/plugins/architecture-internals#provider-runtime-hooks)。
      OpenClaw 已不再呼叫、僅供相容性使用的供應商欄位，例如
      `ProviderPlugin.capabilities` 與 `suppressBuiltInModel`，不會列在
      此處。

      | 鉤子 | 使用時機 |
      | --- | --- |
      | `catalog` | 模型目錄或基礎 URL 預設值 |
      | `applyConfigDefaults` | 設定具體化期間由供應商擁有的全域預設值 |
      | `normalizeModelId` | 查詢前清理舊版／預覽模型 ID 別名 |
      | `normalizeTransport` | 通用模型組裝前清理供應商系列的 `api` / `baseUrl` |
      | `normalizeConfig` | 正規化 `models.providers.<id>` 設定 |
      | `applyNativeStreamingUsageCompat` | 設定型供應商的原生串流用量相容性重寫 |
      | `resolveConfigApiKey` | 解析供應商擁有的環境標記驗證 |
      | `resolveSyntheticAuth` | 本機／自行託管或由設定支援的合成驗證 |
      | `resolveExternalAuthProfiles` | 為命令列介面／應用程式管理的認證資訊疊加供應商擁有的外部驗證設定檔 |
      | `shouldDeferSyntheticProfileAuth` | 將合成的已儲存設定檔預留位置降至環境／設定驗證之後 |
      | `resolveDynamicModel` | 接受任意上游模型 ID |
      | `prepareDynamicModel` | 解析前非同步擷取中繼資料 |
      | `normalizeResolvedModel` | 執行器前的傳輸重寫 |
      | `normalizeToolSchemas` | 註冊前由供應商擁有的工具結構描述清理 |
      | `inspectToolSchemas` | 由供應商擁有的工具結構描述診斷 |
      | `resolveReasoningOutputMode` | 標記式與原生推理輸出合約 |
      | `prepareExtraParams` | 預設請求參數 |
      | `createStreamFn` | 完全自訂的 StreamFn 傳輸 |
      | `wrapStreamFn` | 一般串流路徑上的自訂標頭／主體包裝函式 |
      | `resolveTransportTurnState` | 原生的每回合標頭／中繼資料 |
      | `resolveWebSocketSessionPolicy` | 原生 WS 工作階段標頭／冷卻時間 |
      | `formatApiKey` | 自訂執行階段權杖形狀 |
      | `refreshOAuth` | 自訂 OAuth 重新整理 |
      | `buildAuthDoctorHint` | 驗證修復指引 |
      | `matchesContextOverflowError` | 由供應商擁有的溢位偵測 |
      | `classifyFailoverReason` | 由供應商擁有的速率限制／過載分類 |
      | `isCacheTtlEligible` | 提示詞快取 TTL 閘控 |
      | `buildMissingAuthMessage` | 自訂缺少驗證提示 |
      | `augmentModelCatalog` | 合成的向前相容資料列（已棄用——建議改用 `registerModelCatalogProvider`） |
      | `resolveThinkingProfile` | 模型特定的 `/think` 選項集 |
      | `isBinaryThinking` | 二元思考開啟／關閉相容性（已棄用——建議改用 `resolveThinkingProfile`） |
      | `supportsXHighThinking` | `xhigh` 推理支援相容性（已棄用——建議改用 `resolveThinkingProfile`） |
      | `resolveDefaultThinkingLevel` | 預設 `/think` 原則相容性（已棄用——建議改用 `resolveThinkingProfile`） |
      | `isModernModelRef` | 即時／冒煙測試模型比對 |
      | `prepareRuntimeAuth` | 推論前交換權杖 |
      | `resolveUsageAuth` | 自訂用量認證資訊剖析 |
      | `fetchUsageSnapshot` | 自訂用量端點 |
      | `createEmbeddingProvider` | 供記憶／搜尋使用、由供應商擁有的嵌入轉接器 |
      | `buildReplayPolicy` | 自訂逐字稿重播／壓縮原則 |
      | `sanitizeReplayHistory` | 通用清理後的供應商特定重播重寫 |
      | `validateReplayTurns` | 嵌入式執行器前的嚴格重播回合驗證 |
      | `onModelSelected` | 選取後回呼（例如遙測） |

      執行階段後援說明：

      - `normalizeConfig` 會針對每個供應商 id 解析出一個負責的外掛（先處理內建供應商，再處理相符的執行階段外掛），且只呼叫該掛鉤，不會掃描其他供應商。Google 自有的 `normalizeConfig` 掛鉤負責正規化 `google` / `google-vertex` / `google-antigravity` 設定項目；它並非獨立的核心後援機制。
      - `resolveConfigApiKey` 會在供應商掛鉤公開時使用它。Amazon Bedrock 將 AWS 環境標記解析保留在其供應商外掛中；使用 `auth: "aws-sdk"` 設定時，執行階段驗證本身仍會使用 AWS SDK 預設鏈。
      - `resolveThinkingProfile(ctx)` 會接收所選的 `provider`、`modelId`、選用的合併後 `reasoning` 目錄提示，以及選用的合併後模型 `compat` 資訊。僅使用 `compat` 選取供應商的思考 UI／設定檔。
      - `resolveSystemPromptContribution` 可讓供應商為某個模型系列注入具快取感知能力的系統提示詞指引。當行為屬於單一供應商／模型系列，且應保留穩定／動態快取的分割時，請優先使用它，而非舊版的外掛全域 `before_prompt_build` 掛鉤。

    </Accordion>

  </Step>

  <Step title="新增額外功能（選用）">
    ### 步驟 5：新增額外功能

    供應商外掛除了文字推論之外，還可註冊嵌入、語音、即時轉錄、
    即時語音、媒體理解、圖片生成、影片生成、
    網頁擷取及網頁搜尋。OpenClaw 將其歸類為
    **混合功能**外掛，這是公司外掛的建議模式
    （每個廠商一個外掛）。請參閱
    [內部原理：功能所有權](/zh-TW/plugins/architecture#capability-ownership-model)。

    請在 `register(api)` 內，與現有的
    `api.registerProvider(...)` 呼叫一併註冊各項功能。只選擇需要的分頁：

    <Tabs>
      <Tab title="語音（TTS）">
        ```typescript
        import {
          assertOkOrThrowProviderError,
          postJsonRequest,
        } from "openclaw/plugin-sdk/provider-http";

        api.registerSpeechProvider({
          id: "acme-ai",
          label: "Acme Speech",
          defaultTimeoutMs: 120_000,
          isConfigured: ({ config }) => Boolean(config.messages?.tts),
          synthesize: async (req) => {
            const { response, release } = await postJsonRequest({
              url: "https://api.example.com/v1/speech",
              headers: new Headers({ "Content-Type": "application/json" }),
              body: { text: req.text },
              timeoutMs: req.timeoutMs,
              fetchFn: fetch,
              auditContext: "acme speech",
            });
            try {
              await assertOkOrThrowProviderError(response, "Acme Speech API error");
              return {
                audioBuffer: Buffer.from(await response.arrayBuffer()),
                outputFormat: "mp3",
                fileExtension: ".mp3",
                voiceCompatible: false,
              };
            } finally {
              await release();
            }
          },
        });
        ```

        供應商發生 HTTP 失敗時，請使用 `assertOkOrThrowProviderError(...)`，讓
        外掛共用有限制的錯誤本文讀取、JSON 錯誤剖析及
        請求 id 後綴。
      </Tab>
      <Tab title="即時轉錄">
        建議使用 `createRealtimeTranscriptionWebSocketSession(...)`；這個共用
        輔助工具會處理 Proxy 擷取、重新連線退避、關閉時清空、就緒
        交握、音訊排隊及關閉事件診斷。你的外掛
        只需對應上游事件。

        ```typescript
        api.registerRealtimeTranscriptionProvider({
          id: "acme-ai",
          label: "Acme Realtime Transcription",
          isConfigured: () => true,
          createSession: (req) => {
            const apiKey = String(req.providerConfig.apiKey ?? "");
            return createRealtimeTranscriptionWebSocketSession({
              providerId: "acme-ai",
              callbacks: req,
              url: "wss://api.example.com/v1/realtime-transcription",
              headers: { Authorization: `Bearer ${apiKey}` },
              onMessage: (event, transport) => {
                if (event.type === "session.created") {
                  transport.sendJson({ type: "session.update" });
                  transport.markReady();
                  return;
                }
                if (event.type === "transcript.final") {
                  req.onTranscript?.(event.text);
                }
              },
              sendAudio: (audio, transport) => {
                transport.sendJson({
                  type: "audio.append",
                  audio: audio.toString("base64"),
                });
              },
              onClose: (transport) => {
                transport.sendJson({ type: "audio.end" });
              },
            });
          },
        });
        ```

        透過 POST 傳送多部分音訊的批次 STT 供應商應使用
        `openclaw/plugin-sdk/provider-http` 中的
        `buildAudioTranscriptionFormData(...)`。此輔助工具會正規化上傳
        檔名，包括需要使用 M4A 樣式檔名才能與
        相容轉錄 API 搭配的 AAC 上傳內容。
      </Tab>
      <Tab title="即時語音">
        ```typescript
        api.registerRealtimeVoiceProvider({
          id: "acme-ai",
          label: "Acme Realtime Voice",
          capabilities: {
            transports: ["gateway-relay"],
            inputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            outputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            supportsBargeIn: true,
            handlesInputAudioBargeIn: true,
            supportsToolCalls: true,
          },
          isConfigured: ({ providerConfig }) => Boolean(providerConfig.apiKey),
          createBridge: (req) => ({
            // 僅當供應商可接受單次工具呼叫的多個回應時，才設定此項，
            // 例如先立即回覆「處理中」，之後再回傳
            // 最終結果。
            supportsToolResultContinuation: false,
            connect: async () => {},
            sendAudio: () => {},
            setMediaTimestamp: () => {},
            handleBargeIn: () => {},
            submitToolResult: () => {},
            acknowledgeMark: () => {},
            close: () => {},
            isConnected: () => true,
          }),
        });
        ```

        宣告 `capabilities`，讓 `talk.catalog` 能向瀏覽器及原生 Talk
        用戶端公開有效的模式、傳輸方式、音訊格式及功能旗標。
        當傳輸層可以偵測到人類正在打斷助理播放，且供應商支援
        截短或清除作用中的音訊回應時，請實作 `handleBargeIn`。
        `submitToolResult` 可針對同步提交回傳 `void`，或回傳供應商
        橋接器能公開的非同步完成邊界
        `Promise<void>`。閘道轉送工作階段會等待該 Promise，之後才
        確認最終結果或清除連結的執行；提交失敗時應拒絕它。
        當供應商無法遵循 `options.suppressResponse` 時，請設定
        `supportsToolResultSuppression: false`。OpenClaw 隨後會避免抑制
        內部強制諮詢及取消結果，並拒絕直接
        要求抑制結果，而非無提示地開始回應。
        `createRealtimeVoiceBridgeSession` 的使用端同樣可從
        `onToolCall` 回傳 Promise；同步擲回及拒絕會路由至
        工作階段的 `onError` 回呼。
        僅當供應商 VAD 透過呼叫 `onClearAudio("barge-in")` 確認
        發生中斷時，才設定 `handlesInputAudioBargeIn`。未提供
        此旗標的供應商會使用 OpenClaw 的本機輸入音訊後援偵測。
      </Tab>
      <Tab title="媒體理解">
        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "acme-ai",
          capabilities: ["image", "audio"],
          describeImage: async (req) => ({ text: "一張……的照片" }),
          transcribeAudio: async (req) => ({ text: "逐字稿……" }),
        });
        ```

        刻意不要求認證資訊的本機或自行託管媒體供應商
        可公開 `resolveAuth` 並回傳 `kind: "none"`。
        對於未明確選擇加入的供應商，OpenClaw 仍會保留
        一般驗證閘門。現有供應商可繼續讀取 `req.apiKey`；
        新供應商應優先使用 `req.auth`。

        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "local-audio",
          capabilities: ["audio"],
          resolveAuth: () => ({
            kind: "none",
            source: "local-audio 外掛不需驗證",
          }),
          transcribeAudio: async (req) => ({ text: "逐字稿……" }),
        });
        ```
      </Tab>
      <Tab title="嵌入">
        ```typescript
        api.registerEmbeddingProvider({
          id: "acme-ai",
          defaultModel: "acme-embed",
          transport: "remote",
          authProviderId: "acme-ai",
          create: async ({ model }) => ({
            provider: {
              id: "acme-ai",
              model,
              dimensions: 1536,
              embed: async (input) => {
                const text = typeof input === "string" ? input : input.text;
                return fetchAcmeEmbedding(text);
              },
              embedBatch: async (inputs) =>
                Promise.all(
                  inputs.map((input) =>
                    fetchAcmeEmbedding(typeof input === "string" ? input : input.text),
                  ),
                ),
            },
          }),
        });
        ```

        請在 `contracts.embeddingProviders` 中宣告相同的 id。這是
        可重複使用向量生成的一般嵌入合約，包括
        記憶搜尋。`registerMemoryEmbeddingProvider(...)` 是為現有記憶體專用配接器提供的
        已棄用相容機制。
      </Tab>
      <Tab title="圖片與影片生成">
        圖片與影片功能使用**模式感知**結構。圖片
        供應商需宣告必要的 `generate` 與 `edit` 功能區塊；
        影片供應商需宣告 `generate`、`imageToVideo` 及
        `videoToVideo`。像 `maxInputImages` /
        `maxInputVideos` / `maxDurationSeconds` 這類扁平彙總欄位，不足以明確宣告
        轉換模式支援或停用的模式。音樂生成
        遵循相同的 `generate` / `edit` 模式。

        ```typescript
        api.registerImageGenerationProvider({
          id: "acme-ai",
          label: "Acme 圖片",
          capabilities: {
            generate: { maxCount: 4, supportsSize: true },
            edit: { enabled: false },
          },
          generateImage: async (req) => ({ images: [] }),
        });

        api.registerVideoGenerationProvider({
          id: "acme-ai",
          label: "Acme 影片",
          defaultTimeoutMs: 600_000,
          models: ["acme-video", "acme-image-video"],
          capabilities: {
            generate: { maxVideos: 1, maxDurationSeconds: 10, supportsResolution: true },
            imageToVideo: {
              enabled: true,
              maxVideos: 1,
              maxInputImages: 1,
              maxInputImagesByModel: { "acme/reference-to-video": 9 },
              maxDurationSeconds: 5,
            },
            videoToVideo: { enabled: false },
          },
          catalogByModel: {
            "acme-image-video": {
              modes: ["imageToVideo"],
              capabilities: {
                imageToVideo: {
                  enabled: true,
                  maxVideos: 1,
                  maxInputImages: 1,
                  resolutions: ["480P", "720P", "1080P"],
                  supportsResolution: true,
                },
                videoToVideo: { enabled: false },
              },
            },
          },
          generateVideo: async (req) => ({ videos: [] }),
        });
        ```

        兩種提供者類型都必須有 `capabilities`；`edit` 與
        影片轉換區塊（`imageToVideo`、`videoToVideo`）一律需要
        明確的 `enabled` 旗標。

        當列出的模型之靜態模式或功能
        與提供者預設值不同時，請使用 `catalogByModel`。此中繼資料可讓
        `video_generate action=list` 與模型目錄保持正確，而不必
        呼叫提供者程式碼。請求期間的功能查詢與強制執行
        仍應由 `resolveModelCapabilities` 與 `generateVideo` 負責；可行時，
        兩條路徑應重複使用相同的功能常數。
      </Tab>
      <Tab title="網頁擷取與搜尋">
        ```typescript
        api.registerWebFetchProvider({
          id: "acme-ai-fetch",
          label: "Acme 擷取",
          hint: "透過 Acme 的算繪後端擷取頁面。",
          envVars: ["ACME_FETCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/fetch",
          credentialPath: "plugins.entries.acme.config.webFetch.apiKey",
          getCredentialValue: (fetchConfig) => fetchConfig?.acme?.apiKey,
          setCredentialValue: (fetchConfigTarget, value) => {
            const acme = (fetchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "透過 Acme 擷取來擷取頁面。",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });

        api.registerWebSearchProvider({
          id: "acme-ai-search",
          label: "Acme 搜尋",
          hint: "透過 Acme 的搜尋後端搜尋網路。",
          envVars: ["ACME_SEARCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/search",
          credentialPath: "plugins.entries.acme.config.webSearch.apiKey",
          getCredentialValue: (searchConfig) => searchConfig?.acme?.apiKey,
          setCredentialValue: (searchConfigTarget, value) => {
            const acme = (searchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "透過 Acme 搜尋來搜尋網路。",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });
        ```

        兩種提供者類型共用相同的認證資訊接線結構：
        `hint`、`envVars`、`placeholder`、`signupUrl`、`credentialPath`、
        `getCredentialValue`、`setCredentialValue` 與 `createTool`
        全都是必要項目。
      </Tab>
    </Tabs>

  </Step>

  <Step title="測試">
    ### 步驟 6：測試

    ```typescript src/provider.test.ts
    import { describe, it, expect } from "vitest";
    // 從 index.ts 或專用檔案匯出你的提供者設定物件
    import { acmeProvider } from "./provider.js";

    describe("acme-ai 提供者", () => {
      it("解析動態模型", () => {
        const model = acmeProvider.resolveDynamicModel!({
          modelId: "acme-beta-v3",
        } as any);
        expect(model.id).toBe("acme-beta-v3");
        expect(model.provider).toBe("acme-ai");
      });

      it("有可用金鑰時傳回目錄", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: "test-key" }),
        } as any);
        expect(result?.provider?.models).toHaveLength(2);
      });

      it("沒有金鑰時傳回 null 目錄", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: undefined }),
        } as any);
        expect(result).toBeNull();
      });
    });
    ```

  </Step>
</Steps>

## 發布至 ClawHub

提供者外掛的發布方式與其他外部程式碼外掛相同：

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

`clawhub skill publish <path>` 是用來發布 skill
資料夾的另一個命令，而非外掛套件，因此請勿在此使用。

## 檔案結構

```
<bundled-plugin-root>/acme-ai/
├── package.json              # openclaw.providers 中繼資料
├── openclaw.plugin.json      # 包含提供者驗證中繼資料的資訊清單
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # 測試
    └── usage.ts              # 用量端點（選用）
```

## 目錄順序參考

`catalog.order` 控制你的目錄相對於內建
提供者的合併時機：

| 順序     | 時機          | 使用案例                                        |
| --------- | ------------- | ----------------------------------------------- |
| `simple`  | 第一輪    | 單純使用 API 金鑰的提供者                         |
| `profile` | 簡易項目之後  | 受驗證設定檔限制的提供者                |
| `paired`  | 設定檔之後 | 合成多個相關項目             |
| `late`    | 最後一輪     | 覆寫現有提供者（發生衝突時勝出） |

## 後續步驟

- [頻道外掛](/zh-TW/plugins/sdk-channel-plugins) - 如果你的外掛也提供頻道
- [SDK 執行階段](/zh-TW/plugins/sdk-runtime) - `api.runtime` 輔助工具（TTS、搜尋、子代理程式）
- [SDK 概覽](/zh-TW/plugins/sdk-overview) - 完整的子路徑匯入參考
- [外掛內部架構](/zh-TW/plugins/architecture-internals#provider-runtime-hooks) - 鉤子詳細資訊與內附範例

## 相關內容

- [外掛 SDK 設定](/zh-TW/plugins/sdk-setup)
- [建置外掛](/zh-TW/plugins/building-plugins)
- [建置頻道外掛](/zh-TW/plugins/sdk-channel-plugins)
