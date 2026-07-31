---
read_when:
    - 你正在构建一个新的模型提供商插件
    - 你想要向 OpenClaw 添加兼容 OpenAI 的代理或自定义 LLM
    - 你需要了解提供商身份验证、目录和运行时钩子
sidebarTitle: Provider plugins
summary: 为 OpenClaw 构建模型提供商插件的分步指南
title: 构建提供商插件
x-i18n:
    generated_at: "2026-07-26T06:28:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d175fafc034bd52e996d47e047df104f079f2aba66662b22e8dbdf6c21e7e0
    source_path: plugins/sdk-provider-plugins.md
    workflow: 16
---

构建一个提供商插件，为 OpenClaw 添加模型提供商（LLM）：模型目录、API 密钥身份验证和动态模型解析。

<Info>
  初次使用 OpenClaw 插件？请先阅读[入门指南](/zh-CN/plugins/building-plugins)，
  了解包结构和清单设置。
</Info>

<Tip>
  提供商插件会将模型添加到 OpenClaw 的常规推理循环中。如果模型必须通过原生智能体守护进程运行，且该守护进程负责线程、压缩或工具事件，请将提供商与 [Agent harness](/zh-CN/plugins/sdk-agent-harness) 配合使用，而不要将守护进程协议的详细信息放入核心。
</Tip>

## 操作步骤

<Steps>
  <Step title="包和清单">
    ### 第 1 步：包和清单

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
      "description": "Acme AI 模型提供商",
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
          "choiceLabel": "Acme AI API 密钥",
          "groupId": "acme-ai",
          "groupLabel": "Acme AI",
          "cliFlag": "--acme-ai-api-key",
          "cliOption": "--acme-ai-api-key <key>",
          "cliDescription": "Acme AI API 密钥"
        }
      ],
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    `setup.providers[].envVars` 让 OpenClaw 无需加载插件运行时即可检测凭据。当某个提供商变体应复用另一个提供商 ID 的身份验证时，请添加 `providerAuthAliases`。`modelSupport` 是可选项，可让 OpenClaw 在运行时钩子存在之前，根据 `acme-large` 等简写模型 ID 自动加载你的提供商插件。`package.json` 中的 `openclaw.compat` 和 `openclaw.build` 是发布到 ClawHub 所必需的（`openclaw.compat.pluginApi` 和 `openclaw.build.openclawVersion` 是两个必填字段；省略 `minGatewayVersion` 时，将回退到 `openclaw.install.minHostVersion`）。

  </Step>

  <Step title="注册提供商">
    最小文本提供商需要 `id`、`label`、`auth` 和 `catalog`。
    `catalog` 是提供商拥有的运行时/配置钩子；它可以调用实时供应商 API，并返回 `models.providers` 条目。

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth";

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI 模型提供商",
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
              label: "Acme AI API 密钥",
              hint: "来自 Acme AI 控制面板的 API 密钥",
              optionKey: "acmeAiApiKey",
              flagName: "--acme-ai-api-key",
              envVar: "ACME_AI_API_KEY",
              promptMessage: "输入你的 Acme AI API 密钥",
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

    `registerModelCatalogProvider` 是较新的控制平面目录接口，用于列表、帮助和选择器 UI，涵盖 `text`、`voice`、`image_generation`、`video_generation` 和 `music_generation` 行。将供应商端点调用和响应映射保留在插件中；OpenClaw 负责共享的行结构、来源标签和帮助内容呈现。

    至此，一个可用的提供商就完成了。用户现在可以运行 `openclaw onboard --acme-ai-api-key <key>`，并选择 `acme-ai/acme-large` 作为其模型。

    ### 实时模型发现

    如果你的提供商公开了兼容 OpenAI 的 `/models` API，请让单提供商辅助函数启用共享发现：

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

    `liveModelDiscovery: true` 是公开的插件 SDK 契约，具有以下行为：

    | 范畴 | 契约 |
    | --- | --- |
    | 凭据 | 发现功能使用目录解析出的提供商凭据；如果身份验证提供了 `discoveryApiKey`，则优先使用它。绝不会将密钥引用标记作为令牌发送。默认请求使用 `Authorization: Bearer <token>`；如需使用其他供应商身份验证方案，请使用 `buildRequestHeaders`。 |
    | 端点 | 默认 URL 是相对于有效提供商 `baseUrl` 的 `models`，包括启用 `allowExplicitBaseUrl` 时的操作员覆盖。如需使用其他相对路径，请使用 `endpointPath`。仅对固定供应商 URL 使用 `endpointUrl: { url, requireBaseUrl }`；除非有效基础 URL 仍等于 `requireBaseUrl`，否则将跳过发现，以免将自定义代理凭据发送给供应商。 |
    | 网络限制 | 获取操作使用 OpenClaw 的 SSRF 防护；分页共享一个 5 秒超时预算，每页响应上限为 4 MiB，最多 50 页。跨源分页链接将被拒绝；发生跨源重定向后会移除凭据。 |
    | 缓存 | 成功且非空的目录会按提供商、端点和已解析凭据缓存 60 秒。空结果或不可用结果不会缓存。 |
    | 筛选 | 精确匹配的实时 ID 会保留其可信的静态元数据。新行会被保守地投影为文本/聊天模型。已禁用、已归档、已弃用、明确非聊天、嵌入、重排序、审核、语音、仅图像和仅视频的行会被排除。仅在需要从非标准响应封装中选择行时使用 `readRows`；提供商特定的模型语义仍应放在自定义目录中。 |
    | 失败 | 实时发现仅提供辅助信息。身份验证、网络、超时、分页、解析、空目录和筛选失败时，将返回提供商拥有的静态种子，而不是移除该提供商。 |

    对于非 Bearer 或非标准列表端点，请传递选项，而不是 `true`：

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

    不要将 `endpointUrl` 用作无条件的备用主机。它的 `requireBaseUrl` 检查是凭据隔离边界，适用于模型列表主机与推理主机不同的提供商。

    如果提供商需要自定义模型语义，而不是保守的 OpenAI 兼容投影，请将该投影保留在插件中，并使用 `openclaw/plugin-sdk/provider-catalog-live-runtime` 来处理共享获取生命周期。该辅助函数为你提供受保护的 HTTP 获取、提供商身份验证请求头、结构化 HTTP 错误、TTL 缓存和静态回退行为，而无需将提供商策略放入 OpenClaw 核心。

    当实时 API 只能告知你提供商拥有的静态目录中哪些行当前可用时，请使用 `buildLiveModelProviderConfig`：

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

    当提供商 API 返回更丰富的元数据，并且插件需要自行将各行映射为 OpenClaw 模型定义时，请使用 `getCachedLiveProviderModelRows`：

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

    `run` 应保持身份验证门控，并在没有可用凭据时返回 `null`。保留离线 `staticRun` 或静态回退，以免设置、文档、测试和选择器界面依赖实时网络访问。使用适合模型列表新鲜度的 TTL，避免在请求时轮询文件系统，并且仅当上游响应不是兼容 OpenAI 的 `{ data: [{ id, object }] }` 结构时，才传入提供商专用的 `readRows` / `readModelId`。

    如果上游提供商使用的控制令牌与 OpenClaw 不同，请添加小型双向文本转换，而不是替换流式传输路径：

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

    `input` 会在传输前重写最终系统提示词和文本消息内容。`output` 会在 OpenClaw 解析自身的控制标记或进行渠道投递之前，重写助手文本增量和最终文本。

    对于仅注册一个使用 API 密钥身份验证且仅有一个由目录支持的运行时的文本提供商的内置提供商，优先使用范围更窄的 `defineSingleProviderPluginEntry(...)` 辅助函数：

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

    `buildProvider` 是 OpenClaw 能够解析真实提供商身份验证时使用的实时目录路径。它可以执行提供商专用的发现。仅将 `buildStaticProvider` 用于在配置身份验证之前即可安全显示的离线行；它不得要求凭据或发出网络请求。OpenClaw 的 `models list --all` 显示目前仅对内置提供商插件执行静态目录，并使用空配置、空环境变量，且不提供 Agent/工作区路径。

    如果身份验证流程还需要在新手引导期间修补 `models.providers.*`、别名和 Agent 默认模型，请使用 `openclaw/plugin-sdk/provider-onboard` 中的预设辅助函数。范围最窄的辅助函数是 `createDefaultModelPresetAppliers(...)`、`createDefaultModelsPresetAppliers(...)` 和 `createModelCatalogPresetAppliers(...)`。

    当提供商的原生端点在常规 `openai-completions` 传输上支持流式用量块时，请优先使用 `openclaw/plugin-sdk/provider-catalog-shared` 中的共享目录辅助函数，而不是硬编码提供商 ID 检查。`supportsNativeStreamingUsageCompat(...)` 和 `applyProviderNativeStreamingUsageCompat(...)` 会根据端点能力映射检测支持情况，因此，即使插件使用自定义提供商 ID，原生 Moonshot/DashScope 风格的端点仍可选择启用。

    上述实时发现示例涵盖 `/models` 风格的提供商 API。请将该发现保留在 `catalog.run` 内，由可用的身份验证进行门控，并确保 `staticRun` 不访问网络，以便生成离线目录。

  </Step>

  <Step title="添加动态模型解析">
    如果提供商接受任意模型 ID（例如代理或路由器），请添加 `resolveDynamicModel`：

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

    如果解析需要网络调用，请使用 `prepareDynamicModel` 进行异步预热——完成后会再次运行 `resolveDynamicModel`。

  </Step>

  <Step title="添加运行时钩子（按需）">
    大多数提供商仅需要 `catalog` + `resolveDynamicModel`。随着提供商产生需求，逐步添加钩子。

    共享辅助构建器现已涵盖最常见的重放/工具兼容系列，因此插件通常无需逐个手动连接每个钩子：

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

    当前可用的重放系列：

    | 系列 | 接入的内容 | 内置示例 |
    | --- | --- | --- |
    | `openai-compatible` | 用于兼容 OpenAI 的传输的共享 OpenAI 风格重放策略，包括工具调用 ID 清理、助手优先排序修复，以及传输需要时的通用 Gemini 轮次验证 | `moonshot`、`ollama`、`xai`、`zai` |
    | `anthropic-by-model` | 由 `modelId` 选择的 Claude 感知重放策略，因此只有在解析出的模型实际为 Claude ID 时，Anthropic 消息传输才会进行 Claude 专用的思考块清理 | `amazon-bedrock` |
    | `native-anthropic-by-model` | 与 `anthropic-by-model` 相同的按模型选择 Claude 策略，另外包含工具调用 ID 清理，并为必须保留供应商原生 ID 的传输保留原生 Anthropic 工具使用 ID | `anthropic-vertex`、`clawrouter` |
    | `google-gemini` | 原生 Gemini 重放策略以及引导重放清理。共享系列会使输出文本的 Gemini CLI 继续使用带标签的推理；直接的 `google` 提供商会将 `resolveReasoningOutputMode` 覆盖为 `native`，因为 Gemini API 的思考内容以原生思考部分的形式到达。 | `google`、`google-gemini-cli` |
    | `passthrough-gemini` | 对通过兼容 OpenAI 的代理传输运行的 Gemini 模型执行 Gemini 思考签名清理；不会启用原生 Gemini 重放验证或引导重写 | `openrouter`、`kilocode`、`opencode`、`opencode-go` |
    | `hybrid-anthropic-openai` | 适用于在一个插件中混合使用 Anthropic 消息和兼容 OpenAI 的模型界面的提供商的混合策略；可选的仅限 Claude 的思考块丢弃仍限定在 Anthropic 一侧 | `minimax` |

    当前可用的流式传输系列：

    | 系列 | 接入的功能 | 内置示例 |
    | --- | --- | --- |
    | `google-thinking` | 共享流路径上的 Gemini 思考负载规范化 | `google`、`google-gemini-cli` |
    | `kilocode-thinking` | 共享代理流路径上的 Kilo 推理包装器，其中 `kilo-auto/balanced` 和不受支持的代理推理 ID 会跳过注入思考内容 | `kilocode` |
    | `moonshot-thinking` | 根据配置和 `/think` 级别映射 Moonshot 二进制原生思考负载 | `moonshot` |
    | `minimax-fast-mode` | 共享流路径上的 MiniMax 快速模式模型重写 | `minimax`、`minimax-portal` |
    | `openai-responses-defaults` | 共享的原生 OpenAI/Codex Responses 包装器：归属标头、`/fast`/`serviceTier`、文本详细程度、原生 Codex Web 搜索、推理兼容负载塑形，以及 Responses 上下文管理 | `openai` |
    | `openrouter-thinking` | 用于代理路由的 OpenRouter 推理包装器，集中处理不受支持的模型/`auto` 跳过逻辑 | `openrouter` |
    | `tool-stream-default-on` | 默认启用的 `tool_stream` 包装器，供 Z.AI 等希望启用工具流式传输、除非显式禁用的提供商使用 | `zai` |

    <Accordion title="为系列构建器提供支持的 SDK 接口">
      每个系列构建器都由同一软件包导出的较低层级公共辅助函数组合而成。当提供商需要偏离通用模式时，可以使用这些函数：

      - `openclaw/plugin-sdk/provider-model-shared` - `ProviderReplayFamily`、`buildProviderReplayFamilyHooks(...)`，以及原始重放构建器（`buildOpenAICompatibleReplayPolicy`、`buildAnthropicReplayPolicyForModel`、`buildGoogleGeminiReplayPolicy`、`buildHybridAnthropicOrOpenAIReplayPolicy`）。还导出 Gemini 重放辅助函数（`sanitizeGoogleGeminiReplayHistory`、`resolveTaggedReasoningOutputMode`）以及端点/模型辅助函数（`resolveProviderEndpoint`、`normalizeProviderId`、`normalizeGooglePreviewModelId`）。
      - `openclaw/plugin-sdk/provider-stream` - `ProviderStreamFamily`、`buildProviderStreamFamilyHooks(...)`、`composeProviderStreamWrappers(...)`，以及共享的 OpenAI/Codex 包装器（`createOpenAIAttributionHeadersWrapper`、`createOpenAIFastModeWrapper`、`createOpenAIServiceTierWrapper`、`createOpenAIResponsesContextManagementWrapper`、`createCodexNativeWebSearchWrapper`）、兼容 OpenAI 的 DeepSeek V4 包装器（`createDeepSeekV4OpenAICompatibleThinkingWrapper`）、Anthropic Messages 思考预填充清理（`createAnthropicThinkingPrefillPayloadWrapper`）、纯文本工具调用兼容功能（`createPlainTextToolCallCompatWrapper`）和共享代理/提供商包装器（`createOpenRouterWrapper`、`createToolStreamWrapper`、`createMinimaxFastModeWrapper`）。
      - `openclaw/plugin-sdk/provider-stream-shared` - 用于提供商热路径的轻量级负载和事件包装器，包括 `createOpenAICompatibleCompletionsThinkingOffWrapper`、`createPayloadPatchStreamWrapper`、`createPlainTextToolCallCompatWrapper`、`normalizeOpenAICompatibleReasoningPayload(...)` 和 `setQwenChatTemplateThinking(...)`。
      - `openclaw/plugin-sdk/provider-tools` - `ProviderToolCompatFamily`、`buildProviderToolCompatFamilyHooks("deepseek" | "gemini" | "openai")`，以及底层提供商架构辅助函数。

      对于 Gemini 系列提供商，应确保推理输出模式与传输方式保持一致。
      直接使用 Google Gemini API 的提供商应采用 `native`
      推理输出，以便 OpenClaw 使用原生思考部分，而无需添加
      `<think>` / `<final>` 提示词指令。仅处理文本、采用 Gemini CLI 风格并解析最终 JSON/文本响应的
      后端可以继续使用共享的
      `google-gemini` 标签化契约。

      部分流辅助函数会有意保留在提供商本地。`@openclaw/anthropic-provider` 将 `wrapAnthropicProviderStream`、`resolveAnthropicBetas`、`resolveAnthropicFastMode`、`resolveAnthropicServiceTier` 和较低层级的 Anthropic 包装器构建器保留在其自身的公共 `api.ts` / `contract-api.ts` 接口中，因为它们编码了 Claude OAuth Beta 处理和 `context1m` 门控。类似地，xAI 插件也将原生 xAI Responses 塑形保留在其自身的 `wrapStreamFn` 中（`/fast` 别名、默认 `tool_stream`、不受支持的严格工具清理、xAI 特有的推理负载移除）。

      相同的软件包根目录模式还支持 `@openclaw/openai-provider`（提供商构建器、默认模型辅助函数、实时提供商构建器）和 `@openclaw/openrouter-provider`（提供商构建器以及新手引导/配置辅助函数）。
    </Accordion>

    <Tabs>
      <Tab title="令牌交换">
        对于需要在每次推理调用前交换令牌的提供商：

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
      <Tab title="自定义标头">
        对于需要自定义请求标头或修改请求正文的提供商：

        ```typescript
        // wrapStreamFn 返回一个从 ctx.streamFn 派生的 StreamFn
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
      <Tab title="原生传输身份">
        对于需要在通用 HTTP 或 WebSocket 传输中使用原生请求/会话标头或元数据的提供商：

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
      <Tab title="用量和计费">
        对于公开用量/计费数据的提供商：

        ```typescript
        resolveUsageAuth: async (ctx) => {
          const auth = await ctx.resolveOAuthToken();
          return auth ? { token: auth.token } : null;
        },
        fetchUsageSnapshot: async (ctx) => {
          return await fetchAcmeUsage(ctx.token, ctx.timeoutMs);
        },
        ```

        `resolveUsageAuth` 有三种结果。当
        提供商具有用量/计费凭据时，返回
        `{ token, accountId?, subscriptionType?, rateLimitTier? }`（可选字段会将已解析配置文件中的非机密套餐元数据传递给
        `fetchUsageSnapshot`）。仅当提供商已明确处理用量
        身份验证，但没有可用的用量令牌，且 OpenClaw 必须跳过通用
        API 密钥/OAuth 回退时，才返回
        `{ handled: true }`。当提供商未处理该请求且 OpenClaw 应继续使用通用回退时，返回 `null` 或 `undefined`。

        在 `contracts.usageProviders` 中声明提供商 ID。当该清单
        契约和**两个**钩子都存在时，OpenClaw 会自动将
        该提供商纳入用量收集，而无需加载无关的提供商
        插件。不需要更新核心允许列表。
        `fetchUsageSnapshot` 返回共享的提供商中立结构：

        - `plan`：提供商报告的订阅或密钥标签
        - `windows`：以已用百分比表示的可重置配额窗口
        - `billing`：类型化的 `balance`、`spend` 或 `budget` 条目；`unit` 可以是
          ISO 货币，也可以是 `credits` 等提供商单位
        - `summary`：无法放入上述结构化字段的紧凑提供商特定上下文

        货币语义必须准确。除非上游契约如此规定，否则提供商额度并不等同于 USD。仅实现
        `fetchUsageSnapshot` 的插件仍可供显式/合成调用方使用，但
        不会被自动发现，因为 OpenClaw 无法解析其用量凭据。
      </Tab>
    </Tabs>

    <Accordion title="常用提供商钩子">
      对于模型/提供商插件，OpenClaw 大致按以下顺序调用钩子。
      大多数提供商只使用其中 2-3 个。这不是完整的 `ProviderPlugin`
      契约——有关完整且当前准确的钩子列表和回退说明，请参阅[内部机制：提供商运行时
      钩子](/zh-CN/plugins/architecture-internals#provider-runtime-hooks)。
      此处未列出 OpenClaw 不再调用、仅用于兼容性的提供商字段，例如
      `ProviderPlugin.capabilities` 和 `suppressBuiltInModel`。

      | 钩子 | 使用时机 |
      | --- | --- |
      | `catalog` | 模型目录或基础 URL 默认值 |
      | `applyConfigDefaults` | 配置具体化期间由提供商拥有的全局默认值 |
      | `normalizeModelId` | 查找前清理旧版/预览版模型 ID 别名 |
      | `normalizeTransport` | 通用模型组装前清理提供商系列的 `api` / `baseUrl` |
      | `normalizeConfig` | 规范化 `models.providers.<id>` 配置 |
      | `applyNativeStreamingUsageCompat` | 配置提供商的原生流式用量兼容重写 |
      | `resolveConfigApiKey` | 解析由提供商拥有的环境标记身份验证 |
      | `resolveSyntheticAuth` | 本地/自托管或配置支持的合成身份验证 |
      | `resolveExternalAuthProfiles` | 为 CLI/应用管理的凭据叠加由提供商拥有的外部身份验证配置文件 |
      | `shouldDeferSyntheticProfileAuth` | 将合成的已存储配置文件占位符置于环境/配置身份验证之后 |
      | `resolveDynamicModel` | 接受任意上游模型 ID |
      | `prepareDynamicModel` | 解析前异步获取元数据 |
      | `normalizeResolvedModel` | 运行器执行前重写传输 |
      | `normalizeToolSchemas` | 注册前清理由提供商拥有的工具架构 |
      | `inspectToolSchemas` | 由提供商拥有的工具架构诊断 |
      | `resolveReasoningOutputMode` | 标签化与原生推理输出契约 |
      | `prepareExtraParams` | 默认请求参数 |
      | `createStreamFn` | 完全自定义的 StreamFn 传输 |
      | `wrapStreamFn` | 正常流路径上的自定义标头/正文包装器 |
      | `resolveTransportTurnState` | 原生的每轮标头/元数据 |
      | `resolveWebSocketSessionPolicy` | 原生 WS 会话标头/冷却时间 |
      | `formatApiKey` | 自定义运行时令牌结构 |
      | `refreshOAuth` | 自定义 OAuth 刷新 |
      | `buildAuthDoctorHint` | 身份验证修复指导 |
      | `matchesContextOverflowError` | 由提供商拥有的溢出检测 |
      | `classifyFailoverReason` | 由提供商拥有的速率限制/过载分类 |
      | `isCacheTtlEligible` | 提示缓存 TTL 门控 |
      | `buildMissingAuthMessage` | 自定义缺少身份验证提示 |
      | `augmentModelCatalog` | 合成的前向兼容行（已弃用——建议使用 `registerModelCatalogProvider`） |
      | `resolveThinkingProfile` | 特定于模型的 `/think` 选项集 |
      | `isBinaryThinking` | 二进制思考开关兼容性（已弃用——建议使用 `resolveThinkingProfile`） |
      | `supportsXHighThinking` | `xhigh` 推理支持兼容性（已弃用——建议使用 `resolveThinkingProfile`） |
      | `resolveDefaultThinkingLevel` | 默认 `/think` 策略兼容性（已弃用——建议使用 `resolveThinkingProfile`） |
      | `isModernModelRef` | 实时/冒烟模型匹配 |
      | `prepareRuntimeAuth` | 推理前交换令牌 |
      | `resolveUsageAuth` | 自定义用量凭据解析 |
      | `fetchUsageSnapshot` | 自定义用量端点 |
      | `createEmbeddingProvider` | 用于记忆/搜索、由提供商拥有的嵌入适配器 |
      | `buildReplayPolicy` | 自定义转录重放/压缩策略 |
      | `sanitizeReplayHistory` | 通用清理后特定于提供商的重放重写 |
      | `validateReplayTurns` | 嵌入式运行器执行前的严格重放轮次验证 |
      | `onModelSelected` | 选择后回调（例如遥测） |

      运行时回退说明：

      - `normalizeConfig` 会为每个提供商 ID 解析出一个所属插件（先解析内置提供商，再解析匹配的运行时插件），并且只调用该钩子——不会扫描其他提供商。Google 自己的 `normalizeConfig` 钩子负责规范化 `google` / `google-vertex` / `google-antigravity` 配置条目；它不是独立的核心回退机制。
      - `resolveConfigApiKey` 会在提供商暴露钩子时使用该钩子。Amazon Bedrock 将 AWS 环境标记解析保留在其提供商插件中；使用 `auth: "aws-sdk"` 配置时，运行时身份验证本身仍使用 AWS SDK 默认链。
      - `resolveThinkingProfile(ctx)` 接收选定的 `provider`、`modelId`、可选的合并后 `reasoning` 目录提示，以及可选的合并后模型 `compat` 信息。仅使用 `compat` 选择提供商的思考 UI/配置文件。
      - `resolveSystemPromptContribution` 允许提供商为某个模型系列注入支持缓存感知的系统提示词指导。当行为属于单个提供商/模型系列，并且应保留稳定/动态缓存拆分时，应优先使用它，而不是旧版插件级 `before_prompt_build` 钩子。

    </Accordion>

  </Step>

  <Step title="添加额外能力（可选）">
    ### 步骤 5：添加额外能力

    提供商插件可以在文本推理之外注册嵌入、语音、实时转录、
    实时语音、媒体理解、图像生成、视频生成、
    Web 获取和 Web 搜索。OpenClaw 将其归类为
    **混合能力**插件——这是公司插件的推荐模式
    （每个供应商一个插件）。请参阅
    [内部机制：能力所有权](/zh-CN/plugins/architecture#capability-ownership-model)。

    在 `register(api)` 中将每项能力与现有的
    `api.registerProvider(...)` 调用一起注册。只选择需要的标签页：

    <Tabs>
      <Tab title="语音（TTS）">
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

        对提供商 HTTP 失败使用 `assertOkOrThrowProviderError(...)`，以便
        插件共享有大小上限的错误正文读取、JSON 错误解析和
        请求 ID 后缀。
      </Tab>
      <Tab title="实时转录">
        优先使用 `createRealtimeTranscriptionWebSocketSession(...)`——这个共享
        辅助函数会处理代理捕获、重连退避、关闭时刷新、就绪
        握手、音频排队和关闭事件诊断。你的插件
        只需映射上游事件。

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

        通过 POST 上传 multipart 音频的批量 STT 提供商应使用
        `openclaw/plugin-sdk/provider-http` 中的
        `buildAudioTranscriptionFormData(...)`。该辅助函数会规范化上传
        文件名，包括为需要 M4A 风格文件名的 AAC 上传生成
        与转录 API 兼容的文件名。
      </Tab>
      <Tab title="实时语音">
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
            // 仅当提供商接受一次工具调用对应多个工具响应时才设置此项，
            // 例如，先立即返回“正在处理”响应，随后返回
            // 最终结果。
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

        声明 `capabilities`，以便 `talk.catalog` 可以向浏览器和原生 Talk
        客户端公开有效的模式、传输协议、音频格式和功能标志。
        当传输协议能够检测到人类正在打断助手播放，并且提供商支持
        截断或清除活动音频响应时，实现 `handleBargeIn`。
        `submitToolResult` 可以为同步提交返回 `void`，也可以返回
        `Promise<void>`，作为提供商桥接可公开的异步完成边界。
        Gateway 网关中继会话会等待该 Promise，然后才确认最终结果或
        清除关联的运行；提交失败时应拒绝该 Promise。
        当提供商无法遵循 `options.suppressResponse` 时，设置
        `supportsToolResultSuppression: false`。这样，OpenClaw 就不会对内部强制咨询和取消结果
        应用抑制，并且会拒绝直接发出的结果抑制请求，而不是静默启动响应。
        `createRealtimeVoiceBridgeSession` 的使用方同样可以从
        `onToolCall` 返回 Promise；同步抛出和拒绝会被路由到
        会话的 `onError` 回调。
        仅当提供商 VAD 通过调用 `onClearAudio("barge-in")` 确认中断时，
        才设置 `handlesInputAudioBargeIn`。未设置该标志的提供商将使用
        OpenClaw 的本地输入音频回退检测。
      </Tab>
      <Tab title="媒体理解">
        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "acme-ai",
          capabilities: ["image", "audio"],
          describeImage: async (req) => ({ text: "一张……的照片" }),
          transcribeAudio: async (req) => ({ text: "转录文本……" }),
        });
        ```

        有意不要求凭据的本地或自托管媒体提供商可以暴露
        `resolveAuth` 并返回 `kind: "none"`。
        对于未明确选择启用该行为的提供商，OpenClaw 仍会保留常规身份验证关卡。
        现有提供商可以继续读取 `req.apiKey`；
        新提供商应优先使用 `req.auth`。

        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "local-audio",
          capabilities: ["audio"],
          resolveAuth: () => ({
            kind: "none",
            source: "local-audio 插件无需身份验证",
          }),
          transcribeAudio: async (req) => ({ text: "转录文本……" }),
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

        在 `contracts.embeddingProviders` 中声明相同的 ID。这是
        用于可复用向量生成（包括记忆搜索）的通用嵌入契约。
        `registerMemoryEmbeddingProvider(...)` 是为现有记忆专用适配器保留的
        已弃用兼容机制。
      </Tab>
      <Tab title="图像和视频生成">
        图像和视频能力使用**模式感知**结构。图像
        提供商声明必需的 `generate` 和 `edit` 能力块；
        视频提供商声明 `generate`、`imageToVideo` 和
        `videoToVideo`。像 `maxInputImages` /
        `maxInputVideos` / `maxDurationSeconds` 这样的扁平聚合字段不足以清晰地声明
        转换模式支持或已禁用的模式。音乐生成
        遵循相同的 `generate` / `edit` 模式。

        ```typescript
        api.registerImageGenerationProvider({
          id: "acme-ai",
          label: "Acme Images",
          capabilities: {
            generate: { maxCount: 4, supportsSize: true },
            edit: { enabled: false },
          },
          generateImage: async (req) => ({ images: [] }),
        });

        api.registerVideoGenerationProvider({
          id: "acme-ai",
          label: "Acme Video",
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

        两种提供商类型都需要 `capabilities`；`edit` 和
        视频转换块（`imageToVideo`、`videoToVideo`）始终需要
        显式的 `enabled` 标志。

        当所列模型的静态模式或能力与提供商默认值不同时，
        使用 `catalogByModel`。此元数据无需调用提供商代码，
        即可确保 `video_generate action=list` 和模型目录准确。
        请求时的能力查找和强制执行仍应位于 `resolveModelCapabilities` 和
        `generateVideo` 中；如果可能，请为这两条路径复用
        同一个能力常量。
      </Tab>
      <Tab title="网页获取和搜索">
        ```typescript
        api.registerWebFetchProvider({
          id: "acme-ai-fetch",
          label: "Acme Fetch",
          hint: "Fetch pages through Acme's rendering backend.",
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
            description: "Fetch a page through Acme Fetch.",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });

        api.registerWebSearchProvider({
          id: "acme-ai-search",
          label: "Acme Search",
          hint: "Search the web through Acme's search backend.",
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
            description: "Search the web through Acme Search.",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });
        ```

        两种提供商类型使用相同的凭据连接结构：
        `hint`、`envVars`、`placeholder`、`signupUrl`、`credentialPath`、
        `getCredentialValue`、`setCredentialValue` 和 `createTool`
        均为必需项。
      </Tab>
    </Tabs>

  </Step>

  <Step title="测试">
    ### 第 6 步：测试

    ```typescript src/provider.test.ts
    import { describe, it, expect } from "vitest";
    // 从 index.ts 或专用文件导出提供商配置对象
    import { acmeProvider } from "./provider.js";

    describe("acme-ai provider", () => {
      it("resolves dynamic models", () => {
        const model = acmeProvider.resolveDynamicModel!({
          modelId: "acme-beta-v3",
        } as any);
        expect(model.id).toBe("acme-beta-v3");
        expect(model.provider).toBe("acme-ai");
      });

      it("returns catalog when key is available", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: "test-key" }),
        } as any);
        expect(result?.provider?.models).toHaveLength(2);
      });

      it("returns null catalog when no key", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: undefined }),
        } as any);
        expect(result).toBeNull();
      });
    });
    ```

  </Step>
</Steps>

## 发布到 ClawHub

提供商插件的发布方式与其他任何外部代码插件相同：

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

`clawhub skill publish <path>` 是用于发布技能文件夹的另一条命令，
而不是用于发布插件包——请勿在此处使用。

## 文件结构

```
<bundled-plugin-root>/acme-ai/
├── package.json              # openclaw.providers 元数据
├── openclaw.plugin.json      # 包含提供商身份验证元数据的清单
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # 测试
    └── usage.ts              # 用量端点（可选）
```

## 目录顺序参考

`catalog.order` 控制你的目录相对于内置提供商
进行合并的时机：

| 顺序     | 时机          | 使用场景                                        |
| --------- | ------------- | ----------------------------------------------- |
| `simple`  | 第一轮    | 普通 API 密钥提供商                         |
| `profile` | 简单提供商之后  | 受身份验证配置文件约束的提供商                |
| `paired`  | 配置文件提供商之后 | 合成多个相关条目             |
| `late`    | 最后一轮     | 覆盖现有提供商（发生冲突时胜出） |

## 后续步骤

- [渠道插件](/zh-CN/plugins/sdk-channel-plugins) - 如果你的插件还提供渠道
- [SDK 运行时](/zh-CN/plugins/sdk-runtime) - `api.runtime` 辅助函数（TTS、搜索、子智能体）
- [SDK 概览](/zh-CN/plugins/sdk-overview) - 完整的子路径导入参考
- [插件内部机制](/zh-CN/plugins/architecture-internals#provider-runtime-hooks) - 钩子详情和内置示例

## 相关内容

- [插件 SDK 设置](/zh-CN/plugins/sdk-setup)
- [构建插件](/zh-CN/plugins/building-plugins)
- [构建渠道插件](/zh-CN/plugins/sdk-channel-plugins)
