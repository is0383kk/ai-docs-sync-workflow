---
read_when:
    - 为插件导入选择合适的插件 SDK 子路径
    - 审计内置插件子路径和辅助功能接口
summary: 插件 SDK 子路径目录：按领域分组说明各导入项所在位置
title: 插件 SDK 子路径
x-i18n:
    generated_at: "2026-07-26T06:54:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 58df43436d0e26f1ffa1383be47fd108655e57d61cf5534d650a4fa2fb7b364c
    source_path: plugins/sdk-subpaths.md
    workflow: 16
---

插件 SDK 包含精简的公共子路径，以及 `openclaw/plugin-sdk/` 下仅供仓库内部使用的内置
辅助模块。本页列出两者，并明确标注
私有本地条目。以下三个文件定义了该边界：

- `scripts/lib/plugin-sdk-entrypoints.json`：由构建系统编译的
  维护中入口点清单。
- `scripts/lib/plugin-sdk-private-local-only-subpaths.json`：从有类型、有文档的 SDK 中
  排除的内部子路径。生产条目仍以仅含 JavaScript 的宿主运行时导出形式提供，
  供单独发布的官方插件使用；仅测试条目保持不导出。
- `src/plugin-sdk/entrypoints.ts`：弃用子路径、保留的内置辅助模块、
  受支持的内置门面，以及插件所有的公共表面的分类元数据。

维护者使用 `pnpm plugin-sdk:surface` 审核公共导出数量，并使用
`pnpm plugins:boundary-report:summary` 审核活跃的保留辅助模块子路径；
未使用的保留辅助模块导出会导致 CI 报告失败，而不是作为休眠的兼容性债务
保留在公共 SDK 中。

有关插件创作指南，请参阅[插件 SDK 概览](/zh-CN/plugins/sdk-overview)。

## 插件入口

| 子路径                         | 主要导出                                                                                                                                                                                                |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`      | `definePluginEntry`                                                                                                                                                                                     |
| `plugin-sdk/core`              | `defineChannelPluginEntry`、`createChatChannelPlugin`、`createChannelPluginBase`、`defineSetupPluginEntry`、`buildChannelConfigSchema`、`buildJsonChannelConfigSchema`、`resolveTailscalePublishedHost` |
| `plugin-sdk/provider-entry`    | 2026 年 7 月后为私有本地；`defineSingleProviderPluginEntry`                                                                                                                                        |
| `plugin-sdk/migration`         | 2026 年 7 月后为私有本地；迁移提供商条目辅助模块，例如 `createMigrationItem`、原因常量、条目状态标记、脱敏辅助模块和 `summarizeMigrationItems`                   |
| `plugin-sdk/migration-runtime` | 2026 年 7 月后为私有本地；运行时迁移辅助模块，例如 `copyMigrationFileItem`、`resolvePlannedMigrationTargets`、`withCachedMigrationConfigRuntime` 和 `writeMigrationReport`              |
| `plugin-sdk/health`            | 面向内置健康状态使用方的 Doctor 健康检查注册、检测、修复、选择、严重级别和发现类型                                                                                                                       |

### 兼容性和私有本地辅助模块

仅较晚窗口期的弃用子路径仍保持导出。2026 年 7 月的别名和
未使用的子路径已被删除，而仅供内置使用的辅助模块已从
公共软件包中移除，并在下方标记为私有本地。维护中的列表为
`scripts/lib/plugin-sdk-deprecated-public-subpaths.json`；CI 会拒绝内置
`plugin-sdk/text-runtime` 仅用于兼容，而 `plugin-sdk/zod` 是
兼容性再导出：请直接从 `zod` 导入 `zod`。宽泛的领域
桶文件 `plugin-sdk/agent-runtime`、`plugin-sdk/channel-lifecycle`、
`plugin-sdk/conversation-runtime`、`plugin-sdk/hook-runtime`、
`plugin-sdk/media-runtime`、`plugin-sdk/plugin-runtime` 和
`plugin-sdk/security-runtime` 同样已弃用，请改用聚焦的
子路径。

OpenClaw 基于 Vitest 的测试辅助模块子路径仅供仓库本地使用，不再是
软件包导出：`agent-runtime-test-contracts`、
`channel-contract-testing`、`channel-target-testing`、`channel-test-helpers`、
`plugin-state-test-runtime`、`plugin-test-api`、`plugin-test-contracts`、
`plugin-test-runtime`、`provider-http-test-mocks`、`provider-test-contracts`、
`reply-payload-testing`、`sqlite-runtime-testing`、`test-env`、`test-fixtures`、
`test-live`、`test-live-auth`、`test-media-generation`、
`test-media-understanding`、`test-node-mocks` 和 `testing`。私有内置辅助模块表面
`ssrf-runtime-internal` 和 `codex-native-task-runtime` 也仅供仓库本地
使用。

### 内置插件辅助模块子路径

经过 2026 年 7 月的清理后，仅供内置使用的辅助模块为私有本地模块。软件包契约护栏会阻止跨所有者导入。`src/plugin-sdk/entrypoints.ts` 单独跟踪仍保持公共的受支持内置门面，即由对应内置插件支持的 SDK
入口点，直至通用契约取代
`plugin-sdk/qa-runner-runtime`、`plugin-sdk/telegram-account`；
新代码不应再使用；请参阅下方各行的说明。

<AccordionGroup>
  <Accordion title="渠道子路径">
    | 子路径 | 主要导出 |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`、`defineSetupPluginEntry`、`createChatChannelPlugin`、`createChannelPluginBase`、`createChannelConfigUiHints` |
    | `plugin-sdk/json-schema-runtime` | 2026 年 7 月后为私有本地；用于插件自有 schema 的缓存式 JSON Schema 验证辅助模块 |
    | `plugin-sdk/channel-setup` | `defineChannelSetupContract`、渠道自有的设置字段/输入类型、`createOptionalChannelSetupSurface`、`createOptionalChannelSetupAdapter`、`createOptionalChannelSetupWizard`，以及 `DEFAULT_ACCOUNT_ID`、`createTopLevelChannelDmPolicy`、`setSetupChannelEnabled`、`splitSetupEntries` |
    | `plugin-sdk/setup` | 共享设置向导辅助模块、设置转换器、允许列表提示和设置状态构建器 |
    | `plugin-sdk/setup-runtime` | `defineChannelSetupContract`、`createSetupTranslator`、`createPatchedAccountSetupAdapter`、`createEnvPatchedAccountSetupAdapter`、`createSetupInputPresenceValidator`、`noteChannelLookupFailure`、`noteChannelLookupSummary`、`promptResolvedAllowFrom`、`splitSetupEntries`、`createAllowlistSetupWizardProxy`、`createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`、`detectBinary`、`extractArchive`、`resolveBrewExecutable`、`formatDocsLink`、`CONFIG_DIR` |
    | `plugin-sdk/account-core` | 多账号配置/操作门控辅助模块、默认账号回退辅助模块 |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`、账号 ID 规范化辅助模块 |
    | `plugin-sdk/account-resolution` | 账号查找 + 默认回退辅助模块 |
    | `plugin-sdk/account-helpers` | 精简的账号列表/账号操作辅助模块 |
    | `plugin-sdk/access-groups` | 2026 年 7 月后为私有本地；访问组允许列表解析和已脱敏的组诊断辅助模块 |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | 已弃用的兼容性门面。请使用 `plugin-sdk/channel-outbound`。 |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter`、`resolveChannelDmAccess`、`resolveChannelDmAllowFrom`、`resolveChannelDmPolicy`、`normalizeChannelDmPolicy`、`normalizeLegacyDmAliases` |
    | `plugin-sdk/channel-config-schema` | 共享渠道配置 schema 基元，以及 Zod 和直接 JSON/TypeBox 构建器 |
    | `plugin-sdk/bundled-channel-config-schema` | 2026 年 7 月后为私有本地；仅供维护中的内置插件使用的内置 OpenClaw 渠道配置 schema |
    | `plugin-sdk/chat-channel-ids` | 2026 年 7 月后为私有本地；`BUNDLED_CHAT_CHANNEL_IDS`、`BUNDLED_CHAT_CHANNEL_ENVELOPE_PREFIXES`、`ChatChannelId`。规范的内置/官方聊天渠道 ID，以及供需要识别带信封前缀文本而无需硬编码自身表格的插件使用的格式化器标签/别名。 |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-ingress-runtime` | 实验性的高级渠道入口运行时解析器、隐式提及策略解析器，以及面向已迁移渠道接收路径的路由事实构建器。应优先使用它，而不是在每个插件中分别组装生效的允许列表、命令允许列表和旧版投影。请参阅[频道入口 API](/zh-CN/plugins/sdk-channel-ingress)。 |
    | `plugin-sdk/channel-lifecycle` | 已弃用的兼容性门面。请使用 `plugin-sdk/channel-outbound`。 |
    | `plugin-sdk/channel-outbound` | 消息生命周期契约，以及回复管线选项、回执、实时预览/流式传输、生命周期辅助模块、出站身份、负载规划、持久发送和消息发送上下文辅助模块。请参阅[渠道出站 API](/zh-CN/plugins/sdk-channel-outbound)。 |
    | `plugin-sdk/channel-message` | `plugin-sdk/channel-outbound` 的已弃用兼容性别名。 |
    | `plugin-sdk/inbound-envelope` | 共享入站路由 + 信封构建器辅助模块 |
    | `plugin-sdk/inbound-reply-dispatch` | 已弃用的兼容性门面。入站运行器和分派谓词请使用 `plugin-sdk/channel-inbound`，消息投递辅助模块请使用 `plugin-sdk/channel-outbound`。 |
    | `plugin-sdk/messaging-targets` | 已弃用的目标解析别名；请使用 `plugin-sdk/channel-targets` |
    | `plugin-sdk/outbound-media` | 2026 年 7 月后为私有本地；共享出站媒体加载和托管媒体状态辅助模块 |
    | `plugin-sdk/poll-runtime` | 2026 年 7 月后为私有本地；精简的投票规范化辅助模块 |
    | `plugin-sdk/thread-bindings-runtime` | 2026 年 7 月后为私有本地；线程绑定生命周期和适配器辅助模块 |
    | `plugin-sdk/agent-media-payload` | 面向旧版 `Media*` 负载投影的已弃用兼容性门面。通过 `MsgContext.media` / `toInboundMediaFacts(...)` 传递有序事实；从 `plugin-sdk/media-local-roots` 导入本地根目录策略。 |
    | `plugin-sdk/conversation-runtime` | 已弃用的宽泛桶文件，涵盖对话/线程绑定、配对和已配置绑定辅助模块；请优先使用聚焦的绑定子路径，例如 `plugin-sdk/thread-bindings-runtime` 和 `plugin-sdk/session-binding-runtime` |
    | `plugin-sdk/runtime-group-policy` | 运行时组策略解析辅助模块 |
    | `plugin-sdk/channel-status` | 共享渠道状态快照/摘要辅助模块 |
    | `plugin-sdk/channel-config-primitives` | 精简的渠道配置 schema 基元 |
    | `plugin-sdk/channel-config-writes` | 2026 年 7 月后为私有本地；渠道配置写入授权辅助模块 |
    | `plugin-sdk/channel-plugin-common` | 共享渠道插件前导导出 |
    | `plugin-sdk/allowlist-config-edit` | 允许列表配置编辑/读取辅助模块 |
    | `plugin-sdk/group-access` | 已弃用的组访问决策辅助模块；请使用 `plugin-sdk/channel-ingress-runtime` 中的 `resolveChannelMessageIngress` |
    | `plugin-sdk/direct-dm-guard-policy` | 2026 年 7 月后为私有本地；精简的私信加密前防护策略辅助模块 |
    | `plugin-sdk/discord` | 面向已发布 `@openclaw/discord@2026.3.13` 和受跟踪所有者兼容性的已弃用 Discord 兼容性门面；新插件应使用通用渠道 SDK 子路径 |
    | `plugin-sdk/telegram-account` | 面向受跟踪所有者兼容性的已弃用 Telegram 账号解析兼容性门面；新插件应使用注入的运行时辅助模块或通用渠道 SDK 子路径 |
    | `plugin-sdk/interactive-runtime` | 语义化消息呈现、投递和旧版交互式回复辅助模块。请参阅[消息呈现](/zh-CN/plugins/message-presentation) |
    | `plugin-sdk/question-gateway-runtime` | 从渠道交互处理程序通过 Gateway 网关解析运行时生成的 `ask_user` 选项 |
    | `plugin-sdk/channel-inbound` | 用于事件分类、上下文构建、格式化、根目录、防抖、提及匹配、提及策略和入站日志的共享入站辅助模块 |
    | `plugin-sdk/channel-inbound-debounce` | 精简的入站防抖辅助模块 |
    | `plugin-sdk/channel-mention-gating` | 2026 年 7 月后为私有本地；不包含较宽泛入站运行时表面的精简提及策略、提及标记和提及文本辅助模块 |
    | `plugin-sdk/channel-streaming` | 已弃用的兼容性门面。请使用 `plugin-sdk/channel-outbound`。 |
    | `plugin-sdk/channel-send-result` | 回复结果类型 |
    | `plugin-sdk/channel-actions` | 渠道消息操作辅助模块，以及为保持插件兼容性而保留的已弃用原生 schema 辅助模块 |
    | `plugin-sdk/channel-route` | 2026 年 7 月后为私有本地；共享路由规范化、解析器驱动的目标解析、线程 ID 字符串化、去重/紧凑路由键、已解析目标类型，以及路由/目标比较辅助模块 |
    | `plugin-sdk/channel-targets` | 2026 年 7 月后为私有本地；目标解析辅助模块；路由比较调用方应使用 `plugin-sdk/channel-route` |
    | `plugin-sdk/channel-contract` | 渠道契约类型 |
    | `plugin-sdk/channel-feedback` | 反馈/表情回应接线 |
  </Accordion>

较晚窗口期的渠道兼容性子路径仅在其注册表日期内保持公共。
2026 年 7 月的别名（例如私信访问、回复选项、配对
路径和渠道运行时分支）已被移除；仅供内置使用的辅助模块
为私有本地模块。

  <Accordion title="提供商子路径">
    | 子路径 | 主要导出 |
    | --- | --- |
    | `plugin-sdk/provider-entry` | 2026 年 7 月后仅供私有本地使用；`defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | 2026 年 7 月后仅供私有本地使用；精选的本地/自托管提供商设置辅助函数 |
    | `plugin-sdk/cli-backend` | 2026 年 7 月后仅供私有本地使用；CLI 后端默认值 + 看门狗常量 |
    | `plugin-sdk/provider-auth-runtime` | 2026 年 7 月后仅供私有本地使用；提供商身份验证运行时辅助函数：OAuth 回环流程、令牌交换、身份验证持久化和 API 密钥解析 |
    | `plugin-sdk/provider-oauth-runtime` | 2026 年 7 月后仅供私有本地使用；通用提供商 OAuth 回调类型、回调页面渲染、PKCE/状态辅助函数、授权输入解析、令牌过期辅助函数和中止辅助函数 |
    | `plugin-sdk/provider-auth-api-key` | 2026 年 7 月后仅供私有本地使用；API 密钥新手引导/配置文件写入辅助函数，例如 `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | 2026 年 7 月后仅供私有本地使用；标准 OAuth 身份验证结果构建器 |
    | `plugin-sdk/provider-env-vars` | 2026 年 7 月后仅供私有本地使用；提供商身份验证环境变量查找辅助函数 |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`、`ensureApiKeyFromOptionEnvOrPrompt`、`upsertAuthProfile`、`upsertApiKeyProfile`、`writeOAuthCredentials`、OpenAI Codex 身份验证导入辅助函数、已弃用的 `resolveOpenClawAgentDir` 兼容性导出 |
    | `plugin-sdk/provider-model-shared` | 2026 年 7 月后仅供私有本地使用；`ProviderReplayFamily`、`buildProviderReplayFamilyHooks`、`selectPreferredLocalModelId`、`normalizeModelCompat`、共享重放策略构建器、提供商端点辅助函数和共享模型 ID 规范化辅助函数 |
    | `plugin-sdk/provider-catalog-live-runtime` | 2026 年 7 月后仅供私有本地使用；用于受保护的 `/models` 式发现的实时提供商模型目录辅助函数：`buildLiveModelProviderConfig`、`fetchLiveProviderModelRows`、`getCachedLiveProviderModelRows`、`fetchLiveProviderModelIds`、`LiveModelCatalogHttpError`、`clearLiveCatalogCacheForTests`、模型 ID 筛选、TTL 缓存和静态回退 |
    | `plugin-sdk/provider-catalog-runtime` | 提供商目录扩充运行时钩子，以及用于契约测试的插件提供商注册表接缝 |
    | `plugin-sdk/provider-catalog-shared` | 2026 年 7 月后仅供私有本地使用；`findCatalogTemplate`、`buildSingleProviderApiKeyCatalog`、`buildManifestModelProviderConfig`、`supportsNativeStreamingUsageCompat`、`applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | 2026 年 7 月后仅供私有本地使用；通用提供商 HTTP/端点能力辅助函数、提供商 HTTP 错误和音频转录多部分表单辅助函数 |
    | `plugin-sdk/provider-web-fetch-contract` | 2026 年 7 月后仅供私有本地使用；精简的 Web 获取配置/选择契约辅助函数，例如 `enablePluginInConfig` 和 `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | 2026 年 7 月后仅供私有本地使用；Web 获取提供商注册/缓存辅助函数 |
    | `plugin-sdk/provider-web-search-config-contract` | 2026 年 7 月后仅供私有本地使用；适用于无需插件启用接线的提供商的精简 Web 搜索配置/凭据辅助函数 |
    | `plugin-sdk/provider-web-search-contract` | 2026 年 7 月后仅供私有本地使用；精简的 Web 搜索配置/凭据契约辅助函数，例如 `createWebSearchProviderContractFields`、`enablePluginInConfig`、`resolveProviderWebSearchPluginConfig`，以及限定作用域的凭据设置器/获取器 |
    | `plugin-sdk/provider-web-search` | 2026 年 7 月后仅供私有本地使用；Web 搜索提供商注册/缓存/运行时辅助函数 |
    | `plugin-sdk/embedding-providers` | 2026 年 7 月后仅供私有本地使用；通用嵌入提供商类型和读取辅助函数，包括 `EmbeddingProviderAdapter`、`getEmbeddingProvider(...)` 和 `listEmbeddingProviders(...)`；插件通过 `api.registerEmbeddingProvider(...)` 注册提供商，以强制实施清单所有权 |
    | `plugin-sdk/provider-tools` | 2026 年 7 月后仅供私有本地使用；`ProviderToolCompatFamily`、`buildProviderToolCompatFamilyHooks`，以及 DeepSeek/Gemini/OpenAI 架构清理 + 诊断 |
    | `plugin-sdk/provider-usage` | 2026 年 7 月后仅供私有本地使用；提供商用量快照类型、共享用量获取辅助函数，以及 `fetchClaudeUsage` 等提供商获取器 |
    | `plugin-sdk/provider-stream` | 2026 年 7 月后仅供私有本地使用；`ProviderStreamFamily`、`buildProviderStreamFamilyHooks`、`composeProviderStreamWrappers`、流包装器类型、纯文本工具调用兼容层，以及共享的 Anthropic/Google/Kilocode/MiniMax/Moonshot/OpenAI/OpenRouter/Z.AI 包装器辅助函数 |
    | `plugin-sdk/provider-stream-shared` | 2026 年 7 月后仅供私有本地使用；公开的共享提供商流包装器辅助函数，包括 `composeProviderStreamWrappers`、`createOpenAICompatibleCompletionsThinkingOffWrapper`、`createPlainTextToolCallCompatWrapper`、`createPayloadPatchStreamWrapper`、`createToolStreamWrapper`、`normalizeOpenAICompatibleReasoningPayload`、`setQwenChatTemplateThinking`，以及与 Anthropic/DeepSeek/OpenAI 兼容的流工具函数 |
    | `plugin-sdk/provider-transport-runtime` | 2026 年 7 月后仅供私有本地使用；原生提供商传输辅助函数，例如受保护的获取、工具结果文本提取、传输消息转换和可写传输事件流 |
    | `plugin-sdk/provider-onboard` | 2026 年 7 月后仅供私有本地使用；新手引导配置补丁辅助函数 |
    | `plugin-sdk/global-singleton` | 2026 年 7 月后仅供私有本地使用；进程本地单例/映射/缓存辅助函数 |
    | `plugin-sdk/group-activation` | 2026 年 7 月后仅供私有本地使用；精简的群组激活模式和命令解析辅助函数 |
  </Accordion>

提供商用量快照通常会报告一个或多个配额 `windows`，每个都包含
标签、已用百分比和可选的重置时间。对于公开余额或
账户状态文本而非可重置配额窗口的提供商，应返回
`summary` 并将 `windows` 数组置空，而不是编造百分比。
OpenClaw 会在状态输出中显示该摘要文本；仅当用量端点
失败或未返回可用的用量数据时，才使用 `error`。

  <Accordion title="身份验证和安全子路径">
    | 子路径 | 主要导出 |
    | --- | --- |
    | `plugin-sdk/command-auth` | 已弃用的宽泛命令授权表面（`resolveControlCommandGate`、命令注册表辅助函数，包括动态参数菜单格式化和发送者授权辅助函数）；请使用频道入口/运行时授权或命令状态辅助函数 |
    | `plugin-sdk/command-status` | 命令/帮助消息构建器，例如 `buildCommandsMessagePaginated` 和 `buildHelpMessage` |
    | `plugin-sdk/approval-auth-runtime` | 审批人解析和同一聊天内的操作身份验证辅助函数 |
    | `plugin-sdk/approval-client-runtime` | 原生 Exec 审批配置文件/筛选辅助函数 |
    | `plugin-sdk/approval-delivery-runtime` | 原生审批能力/交付适配器 |
    | `plugin-sdk/approval-gateway-runtime` | 共享审批 Gateway 网关解析器 |
    | `plugin-sdk/approval-reference-runtime` | 2026 年 7 月后仅供私有本地使用；用于受传输限制的审批回调的确定性持久定位器辅助函数 |
    | `plugin-sdk/approval-handler-adapter-runtime` | 用于高频渠道入口点的轻量级原生审批适配器加载辅助函数 |
    | `plugin-sdk/approval-handler-runtime` | 更宽泛的审批处理程序运行时辅助函数；当更精简的适配器/Gateway 网关接缝已足够时，应优先使用它们 |
    | `plugin-sdk/approval-native-runtime` | 原生审批目标、账户绑定、路由门控、转发回退和本地原生 Exec 提示抑制辅助函数 |
    | `plugin-sdk/approval-reaction-runtime` | 2026 年 7 月后仅供私有本地使用；硬编码的审批表情回应绑定、表情回应提示负载、表情回应目标存储、表情回应提示文本辅助函数，以及本地原生 Exec 提示抑制的兼容性导出 |
    | `plugin-sdk/approval-reply-runtime` | Exec/插件审批回复负载辅助函数 |
    | `plugin-sdk/approval-runtime` | Exec/插件审批负载辅助函数、审批能力构建器、审批身份验证/配置文件辅助函数、原生审批路由/运行时辅助函数，以及 `formatApprovalDisplayPath` 等结构化审批显示辅助函数 |
    | `plugin-sdk/command-auth-native` | 原生命令身份验证、动态参数菜单格式化和原生会话目标辅助函数 |
    | `plugin-sdk/command-detection` | 共享命令检测辅助函数 |
    | `plugin-sdk/command-primitives-runtime` | 用于高频渠道路径的轻量级命令文本谓词 |
    | `plugin-sdk/command-surface` | 2026 年 7 月后仅供私有本地使用；命令正文规范化和命令表面辅助函数 |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/provider-auth-login-flow-runtime` | 2026 年 7 月后仅供私有本地使用；用于私有渠道和 Web UI 设备代码配对的延迟提供商身份验证登录流程辅助函数 |
    | `plugin-sdk/channel-secret-runtime` | 已弃用的宽泛密钥契约表面（`collectSimpleChannelFieldAssignments`、`getChannelSurface`、`pushAssignment`、密钥目标类型）；请优先使用下方聚焦的子路径 |
    | `plugin-sdk/channel-secret-basic-runtime` | 用于非 TTS 渠道/插件密钥表面的精简密钥契约导出和目标注册表构建器 |
    | `plugin-sdk/channel-secret-tts-runtime` | 2026 年 7 月后仅供私有本地使用；精简的嵌套渠道 TTS 密钥赋值辅助函数 |
    | `plugin-sdk/secret-ref-runtime` | 用于密钥契约/配置解析的精简 SecretRef 类型定义、解析和计划目标路径查找 |
    | `plugin-sdk/security-runtime` | 已弃用的宽泛桶文件，涵盖信任、私信门控、根目录边界内的文件/路径辅助函数（包括仅创建写入、同步/异步原子文件替换、同级临时文件写入、跨设备移动回退、私有文件存储辅助函数、符号链接父目录防护）、外部内容、敏感文本编校、恒定时间密钥比较和密钥收集辅助函数；请优先使用聚焦的安全/SSRF/密钥子路径 |
    | `plugin-sdk/ssrf-policy` | 主机允许列表和私有网络 SSRF 策略辅助函数 |
    | `plugin-sdk/ssrf-dispatcher` | 2026 年 7 月后仅供私有本地使用；不含宽泛基础设施运行时表面的精简固定调度器辅助函数 |
    | `plugin-sdk/ssrf-runtime` | 固定调度器、受 SSRF 防护的获取、SSRF 错误和 SSRF 策略辅助函数 |
    | `plugin-sdk/secret-input` | 密钥输入解析辅助函数 |
    | `plugin-sdk/webhook-ingress` | Webhook 请求/目标辅助函数，以及原始 WebSocket/正文强制转换 |
    | `plugin-sdk/webhook-request-guards` | 请求正文大小/超时辅助函数，以及用于跟踪确认后处理的 `runDetachedWebhookWork` |
  </Accordion>

  <Accordion title="Runtime and storage subpaths">
    | 子路径 | 主要导出 |
    | --- | --- |
    | `plugin-sdk/runtime` | 运行时、日志、备份辅助函数，插件安装路径警告，以及进程辅助函数 |
    | `plugin-sdk/runtime-env` | 精简的运行时环境、日志记录器、超时、重试和退避辅助函数 |
    | `plugin-sdk/browser-config` | 2026 年 7 月后仅限私有本地使用；受支持的浏览器配置门面，用于规范化配置文件/默认值、CDP URL 解析和浏览器控制身份验证辅助函数 |
    | `plugin-sdk/agent-harness-task-runtime` | 2026 年 7 月后仅限私有本地使用；用于基于 harness、使用主机签发任务范围的智能体的通用任务生命周期和完成结果交付辅助函数 |
    | `plugin-sdk/codex-mcp-projection` | 2026 年 7 月后仅限私有本地使用；预留的内置 Codex 辅助函数，用于将用户 MCP 服务器配置映射到 Codex 线程配置；不适用于第三方插件 |
    | `plugin-sdk/codex-native-task-runtime` | 仓库本地的内置 Codex 辅助函数，用于原生任务镜像/运行时接线；不是包导出 |
    | `plugin-sdk/channel-runtime-context` | 通用渠道运行时上下文注册和查找辅助函数 |
    | `plugin-sdk/matrix` | 面向旧版第三方渠道包的已弃用 Matrix 兼容性门面；新插件应直接导入 `plugin-sdk/run-command` |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | 面向插件命令、钩子、HTTP 和交互辅助函数的已弃用宽泛 barrel；优先使用聚焦的插件运行时子路径 |
    | `plugin-sdk/hook-runtime` | 面向 webhook/内部钩子流水线辅助函数的已弃用宽泛 barrel；优先使用聚焦的钩子/插件运行时子路径 |
    | `plugin-sdk/lazy-runtime` | 延迟运行时导入/绑定辅助函数，例如 `createLazyRuntimeModule`、`createLazyRuntimeMethod` 和 `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | 2026 年 7 月后仅限私有本地使用；进程 Exec 辅助函数 |
    | `plugin-sdk/node-host` | 2026 年 7 月后仅限私有本地使用；Node 主机可执行文件解析和 PTY 恢复辅助函数 |
    | `plugin-sdk/cli-runtime` | 2026 年 7 月后仅限私有本地使用；面向 CLI 格式化、等待、版本、参数调用和延迟命令组辅助函数的已弃用宽泛 barrel；优先使用聚焦的 CLI/运行时子路径 |
    | `plugin-sdk/qa-runner-runtime` | 2026 年 7 月后仅限私有本地使用；受支持的门面，通过 CLI 命令表面公开插件 QA 场景 |
    | `plugin-sdk/tts-runtime` | 2026 年 7 月后仅限私有本地使用；用于文本转语音配置架构和运行时辅助函数的受支持门面 |
    | `plugin-sdk/gateway-method-runtime` | 预留的 Gateway 网关方法分派辅助函数，用于声明 `contracts.gatewayMethodDispatch: ["authenticated-request"]` 的插件 HTTP 路由 |
    | `plugin-sdk/gateway-runtime` | Gateway 网关客户端、事件循环就绪的客户端启动辅助函数、Gateway CLI RPC、Gateway 网关协议错误、广播的 LAN 主机解析，以及渠道状态修补辅助函数 |
    | `plugin-sdk/config-contracts` | 聚焦的纯类型配置表面，用于 `OpenClawConfig` 等插件配置形状以及渠道/提供商配置类型 |
    | `plugin-sdk/plugin-config-runtime` | 面向运行时插件配置辅助函数的已弃用兼容性门面；新插件使用 `api.pluginConfig`，以及聚焦的配置契约、快照和变更辅助函数 |
    | `plugin-sdk/config-mutation` | 事务性配置变更辅助函数，例如 `mutateConfigFile`、`replaceConfigFile` 和 `logConfigUpdated` |
    | `plugin-sdk/message-tool-delivery-hints` | 2026 年 7 月后仅限私有本地使用；共享消息工具交付元数据提示字符串 |
    | `plugin-sdk/runtime-config-snapshot` | 当前进程配置快照辅助函数，例如 `getRuntimeConfig`、`getRuntimeConfigSnapshot` 和测试快照设置器 |
    | `plugin-sdk/text-autolink-runtime` | 2026 年 7 月后仅限私有本地使用；无需宽泛文本 barrel 的文件引用自动链接检测 |
    | `plugin-sdk/reply-runtime` | 共享入站/回复运行时辅助函数、分块、分派、Heartbeat、回复规划器 |
    | `plugin-sdk/reply-dispatch-runtime` | 精简的回复分派/最终确定和对话标签辅助函数 |
    | `plugin-sdk/reply-history` | 共享的短时间窗口回复历史辅助函数。新的消息轮次代码应使用 `createChannelHistoryWindow`；较底层的映射辅助函数仍仅作为已弃用的兼容性导出 |
    | `plugin-sdk/reply-reference` | 2026 年 7 月后仅限私有本地使用；`createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | 精简的文本/Markdown 分块辅助函数 |
    | `plugin-sdk/session-store-runtime` | 会话工作流辅助函数（`getSessionEntry`、`listSessionEntries`、`patchSessionEntry`、`upsertSessionEntry`）、修复/生命周期辅助函数（`deleteSessionEntry`、`cleanupSessionLifecycleArtifacts`、`resolveSessionStoreBackupPaths`）、用于过渡性 `sessionFile` 值的标记辅助函数、按会话身份进行的有界近期用户/助手转录文本读取、会话存储路径/会话键辅助函数和更新时间读取，且不含宽泛配置写入/维护导入 |
    | `plugin-sdk/session-transcript-runtime` | 2026 年 7 月后仅限私有本地使用；转录身份、有界原始及可见游标、限定范围的目标/读取/写入辅助函数、可见消息条目映射、更新发布、写入锁和转录内存命中键 |
    | `plugin-sdk/sqlite-runtime` | 2026 年 7 月后仅限私有本地使用；面向第一方运行时的聚焦 SQLite 智能体架构、路径和事务辅助函数，不含数据库生命周期控制 |
    | `plugin-sdk/cron-store-runtime` | 2026 年 7 月后仅限私有本地使用；定时任务存储路径/加载/保存辅助函数 |
    | `plugin-sdk/state-paths` | 状态/OAuth 目录路径辅助函数 |
    | `plugin-sdk/plugin-state-runtime` | 2026 年 7 月后仅限私有本地使用；插件范围内的键控状态、BLOB 和协作式 SQLite 租约契约，以及连接 pragma、经验证的 WAL 维护和原子 STRICT 架构迁移辅助函数。租约回调接收中止信号，类型化错误用于区分超时、取消、所有权丢失、无效输入和存储故障 |
    | `plugin-sdk/routing` | 路由/会话键/账号绑定辅助函数，例如 `resolveAgentRoute`、`buildAgentSessionKey` 和 `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | 共享渠道/账号状态摘要辅助函数、运行时状态默认值和问题元数据辅助函数 |
    | `plugin-sdk/target-resolver-runtime` | 2026 年 7 月后仅限私有本地使用；共享目标解析辅助函数 |
    | `plugin-sdk/string-normalization-runtime` | 2026 年 7 月后仅限私有本地使用；Slug/字符串规范化辅助函数 |
    | `plugin-sdk/request-url` | 2026 年 7 月后仅限私有本地使用；从类似 fetch/request 的输入中提取字符串 URL |
    | `plugin-sdk/run-command` | 带有规范化 stdout/stderr 结果的定时命令运行器 |
    | `plugin-sdk/param-readers` | 通用工具/CLI 参数读取器 |
    | `plugin-sdk/tool-plugin` | 定义简单的类型化智能体工具插件，并公开用于生成清单的静态元数据 |
    | `plugin-sdk/tool-payload` | 2026 年 7 月后仅限私有本地使用；从工具结果对象中提取规范化载荷 |
    | `plugin-sdk/tool-send` | 从工具参数中提取规范发送目标字段 |
    | `plugin-sdk/sandbox` | 2026 年 7 月后仅限私有本地使用；沙箱后端类型和 SSH/OpenShell 命令辅助函数，包括快速失败的 Exec 命令预检 |
    | `plugin-sdk/temp-path` | 共享临时下载路径辅助函数和私有安全临时工作区 |
    | `plugin-sdk/logging-core` | 子系统日志记录器和脱敏辅助函数 |
    | `plugin-sdk/markdown-table-runtime` | 2026 年 7 月后仅限私有本地使用；Markdown 表格模式和转换辅助函数 |
    | `plugin-sdk/model-session-runtime` | 模型/会话覆盖辅助函数，例如 `applyModelOverrideToSessionEntry` 和 `resolveAgentMaxConcurrent` |
    | `plugin-sdk/talk-config-runtime` | 2026 年 7 月后仅限私有本地使用；Talk 提供商配置解析辅助函数 |
    | `plugin-sdk/json-store` | 小型 JSON 状态读写辅助函数 |
    | `plugin-sdk/json-unsafe-integers` | 2026 年 7 月后仅限私有本地使用；将不安全的整数字面量保留为字符串的 JSON 解析辅助函数 |
    | `plugin-sdk/file-lock` | 2026 年 7 月后仅限私有本地使用；可重入文件锁辅助函数，以及 Doctor 安全回收明确已过期、未更改且已停用的锁 sidecar |
    | `plugin-sdk/persistent-dedupe` | 磁盘支持的去重缓存辅助函数 |
    | `plugin-sdk/ingress-effect-once` | 用于非幂等入口副作用的持久声明/提交防护 |
    | `plugin-sdk/acp-runtime` | 2026 年 7 月后仅限私有本地使用；ACP 运行时/会话和回复分派辅助函数 |
    | `plugin-sdk/acp-runtime-backend` | 2026 年 7 月后仅限私有本地使用；用于启动时加载插件的轻量级 ACP 后端注册和回复分派辅助函数 |
    | `plugin-sdk/acp-binding-resolve-runtime` | 2026 年 7 月后仅限私有本地使用；不导入生命周期启动逻辑的只读 ACP 绑定解析 |
    | `plugin-sdk/agent-config-primitives` | 已弃用的智能体运行时配置架构基元；请从维护中的插件自有表面导入架构基元 |
    | `plugin-sdk/boolean-param` | 宽松的布尔参数读取器 |
    | `plugin-sdk/dangerous-name-runtime` | 2026 年 7 月后仅限私有本地使用；危险名称匹配解析辅助函数 |
    | `plugin-sdk/device-bootstrap` | 设备引导和配对令牌辅助函数，包括 `BOOTSTRAP_HANDOFF_OPERATOR_SCOPES` |
    | `plugin-sdk/extension-shared` | 共享被动渠道、状态和环境代理辅助基元 |
    | `plugin-sdk/models-provider-runtime` | `/models` 命令/提供商回复辅助函数 |
    | `plugin-sdk/skill-commands-runtime` | 技能命令列表辅助函数 |
    | `plugin-sdk/native-command-registry` | 原生命令注册表/构建/序列化辅助函数 |
    | `plugin-sdk/agent-harness` | 面向底层智能体 harness 的实验性可信插件表面：harness 类型、活动运行 Steer/中止辅助函数、OpenClaw 工具桥接辅助函数、运行时计划工具策略辅助函数、终端结果分类、工具进度格式化/详情辅助函数和尝试结果实用工具 |
    | `plugin-sdk/async-lock-runtime` | 2026 年 7 月后仅限私有本地使用；用于小型运行时状态文件的进程本地异步锁辅助函数 |
    | `plugin-sdk/channel-activity-runtime` | 2026 年 7 月后仅限私有本地使用；渠道活动遥测辅助函数 |
    | `plugin-sdk/concurrency-runtime` | 2026 年 7 月后仅限私有本地使用；有界异步任务并发辅助函数 |
    | `plugin-sdk/dedupe-runtime` | 内存和持久化后端支持的去重缓存辅助函数 |
    | `plugin-sdk/delivery-queue-runtime` | 2026 年 7 月后仅限私有本地使用；出站待处理交付排空辅助函数 |
    | `plugin-sdk/file-access-runtime` | 2026 年 7 月后仅限私有本地使用；安全的本地文件和媒体源路径辅助函数 |
    | `plugin-sdk/heartbeat-runtime` | 2026 年 7 月后仅限私有本地使用；Heartbeat 唤醒、事件和可见性辅助函数 |
    | `plugin-sdk/expect-runtime` | 2026 年 7 月后仅限私有本地使用；用于可证明运行时不变量的必需值断言辅助函数 |
    | `plugin-sdk/number-runtime` | 2026 年 7 月后仅限私有本地使用；数值强制转换辅助函数 |
    | `plugin-sdk/secure-random-runtime` | 2026 年 7 月后仅限私有本地使用；安全令牌/UUID 辅助函数 |
    | `plugin-sdk/system-event-runtime` | 2026 年 7 月后仅限私有本地使用；系统事件队列辅助函数 |
    | `plugin-sdk/transport-ready-runtime` | 2026 年 7 月后仅限私有本地使用；传输就绪等待辅助函数 |
    | `plugin-sdk/exec-approvals-runtime` | 2026 年 7 月后仅限私有本地使用；不含宽泛基础设施运行时 barrel 的 Exec 审批策略文件辅助函数 |
    | `plugin-sdk/infra-runtime` | 已弃用的兼容性 shim；请使用上面的聚焦运行时子路径 |
    | `plugin-sdk/collection-runtime` | 小型有界缓存辅助函数 |
    | `plugin-sdk/diagnostic-runtime` | 诊断标志、事件和跟踪上下文辅助函数 |
    | `plugin-sdk/error-runtime` | 错误图、格式化、未知值强制转换、共享错误分类辅助函数、`PlatformMessageNotDispatchedError`、`isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | 2026 年 7 月后仅限私有本地使用；封装的 fetch、代理、EnvHttpProxyAgent 选项和固定查找辅助函数 |
    | `plugin-sdk/runtime-fetch` | 2026 年 7 月后仅限私有本地使用；可感知 dispatcher 的运行时 fetch，不导入代理/受保护 fetch |
    | `plugin-sdk/inline-image-data-url-runtime` | 2026 年 7 月后仅限私有本地使用；不含宽泛媒体运行时表面的内联图像数据 URL 清理和签名嗅探辅助函数 |
    | `plugin-sdk/response-limit-runtime` | 2026 年 7 月后仅限私有本地使用；不含宽泛媒体运行时表面的按字节、空闲时间和截止时间设限的响应正文读取器 |
    | `plugin-sdk/session-binding-runtime` | 2026 年 7 月后仅限私有本地使用；当前对话绑定状态，不含已配置的绑定路由或配对存储 |
    | `plugin-sdk/context-visibility-runtime` | 2026 年 7 月后仅限私有本地使用；上下文可见性解析和补充上下文过滤，不导入宽泛配置/安全模块 |
    | `plugin-sdk/string-coerce-runtime` | 精简的基础记录/字符串强制转换和规范化辅助函数，不导入 Markdown/日志模块 |
    | `plugin-sdk/html-entity-runtime` | 2026 年 7 月后仅限私有本地使用；无需宽泛文本实用工具的单遍、分号终止 HTML5 实体解码 |
    | `plugin-sdk/text-utility-runtime` | 2026 年 7 月后为私有本地使用；底层文本和路径辅助函数，包括五实体 HTML 转义 |
    | `plugin-sdk/widget-html` | 自包含 HTML 小组件的完整文档检测、大小验证和工具输入错误 |
    | `plugin-sdk/host-runtime` | 2026 年 7 月后为私有本地使用；主机名和 SCP 主机规范化辅助函数 |
    | `plugin-sdk/retry-runtime` | 2026 年 7 月后为私有本地使用；重试配置和重试运行器辅助函数 |
    | `plugin-sdk/agent-runtime` | 已弃用的宽泛 barrel，用于 Agent 目录、身份和工作区辅助函数，包括 `resolveAgentDir`、`resolveDefaultAgentDir` 和已弃用的 `resolveOpenClawAgentDir` 兼容性导出；应优先使用聚焦的 Agent/运行时子路径 |
    | `plugin-sdk/directory-runtime` | 基于配置的目录查询/去重 |
    | `plugin-sdk/keyed-async-queue` | 2026 年 7 月后为私有本地使用；`KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="能力和测试子路径">
    | 子路径 | 主要导出 |
    | --- | --- |
    | `plugin-sdk/media-runtime` | 已弃用的宽泛媒体桶形导出，包括 `saveRemoteMedia`、`saveResponseMedia`、`readRemoteMediaBuffer` 和已弃用的 `fetchRemoteMedia`；应优先使用 `plugin-sdk/media-store`、`plugin-sdk/media-mime`、`plugin-sdk/outbound-media` 和能力运行时子路径；当 URL 应转换为 OpenClaw 媒体时，应先使用存储辅助函数，再读取缓冲区 |
    | `plugin-sdk/media-local-roots` | 用于插件自有本地媒体读取的专用 `getAgentScopedMediaLocalRoots(...)` 和可感知策略的 `getAgentScopedMediaLocalRootsForSources(...)` 辅助函数 |
    | `plugin-sdk/media-mime` | 精简的 MIME 规范化、文件扩展名映射、MIME 检测和媒体类型辅助函数 |
    | `plugin-sdk/media-store` | 精简的媒体存储辅助函数，例如 `saveMediaBuffer` 和 `saveMediaStream` |
    | `plugin-sdk/media-generation-runtime` | 2026 年 7 月后仅限私有本地使用；共享的媒体生成故障转移辅助函数、候选项选择和模型缺失消息 |
    | `plugin-sdk/media-understanding` | 用于媒体理解提供商类型和辅助函数的已弃用兼容门面；新提供商通过注入的插件 API 注册，并将请求辅助函数保留在插件自身范围内 |
    | `plugin-sdk/text-chunking` | 出站文本和保留偏移量的范围分块、Markdown 分块/渲染辅助函数、可感知引号的 HTML 标签词元化、Markdown 表格转换、指令标签剥离和安全文本实用工具 |
    | `plugin-sdk/speech` | 2026 年 7 月后仅限私有本地使用；语音提供商类型，以及面向提供商的指令、注册表、验证、OpenAI 兼容 TTS 构建器和语音辅助函数导出 |
    | `plugin-sdk/speech-core` | 2026 年 7 月后仅限私有本地使用；共享语音提供商类型、注册表、指令、规范化和语音辅助函数导出 |
    | `plugin-sdk/speech-settings` | 不含提供商注册表或合成运行时的轻量级 TTS 配置解析和规范化原语 |
    | `plugin-sdk/realtime-transcription` | 2026 年 7 月后仅限私有本地使用；实时转录提供商类型、注册表辅助函数和共享 WebSocket 会话辅助函数 |
    | `plugin-sdk/realtime-bootstrap-context` | 2026 年 7 月后仅限私有本地使用；用于有限注入 `IDENTITY.md`、`USER.md` 和 `SOUL.md` 上下文的实时配置文件引导辅助函数 |
    | `plugin-sdk/realtime-voice` | 2026 年 7 月后仅限私有本地使用；实时语音提供商类型、注册表辅助函数、共享音频能量/语音起始门控和实时语音行为辅助函数，包括与传输无关的会话框架和输出活动跟踪 |
    | `plugin-sdk/meeting-runtime` | 浏览器会议会话运行时、实时音频引擎/传输、`MeetingPlatformAdapter`、浏览器/节点控制、智能体咨询、语音通话委派、设置检查和 SoX 命令辅助函数 |
    | `plugin-sdk/image-generation` | 2026 年 7 月后仅限私有本地使用；图像生成提供商类型，以及图像资产/数据 URL 辅助函数和 OpenAI 兼容图像提供商构建器 |
    | `plugin-sdk/image-generation-core` | 2026 年 7 月后仅限私有本地使用；共享图像生成类型、故障转移、身份验证和注册表辅助函数 |
    | `plugin-sdk/music-generation` | 2026 年 7 月后仅限私有本地使用；音乐生成提供商/请求/结果类型 |
    | `plugin-sdk/video-generation` | 2026 年 7 月后仅限私有本地使用；视频生成提供商/请求/结果类型 |
    | `plugin-sdk/video-generation-core` | 2026 年 7 月后仅限私有本地使用；共享视频生成类型、故障转移辅助函数、提供商查找和模型引用解析 |
    | `plugin-sdk/transcripts` | 2026 年 7 月后仅限私有本地使用；共享转录源提供商类型、注册表辅助函数、会议提供商桥接工厂、会话描述符和话语元数据 |
    | `plugin-sdk/webhook-targets` | 2026 年 7 月后仅限私有本地使用；Webhook 目标注册表和路由安装辅助函数 |
    | `plugin-sdk/web-media` | 共享远程/本地媒体加载辅助函数 |
    | `plugin-sdk/zod` | 已弃用的兼容性重新导出；请直接从 `zod` 导入 `zod` |
    | `plugin-sdk/plugin-test-api` | 仓库本地的精简 `createTestPluginApi` 辅助函数，用于直接插件注册单元测试，无需导入仓库测试辅助桥接 |
    | `plugin-sdk/agent-runtime-test-contracts` | 仓库本地的原生智能体运行时适配器契约夹具，用于身份验证、交付、故障转移、工具钩子、提示词叠加层、模式和转录投影测试 |
    | `plugin-sdk/channel-test-helpers` | 仓库本地面向渠道的测试辅助函数，用于通用操作/设置/状态契约、目录断言、账户启动生命周期、发送配置传递、运行时模拟、状态问题、出站交付和钩子注册 |
    | `plugin-sdk/channel-target-testing` | 用于渠道测试的仓库本地共享目标解析错误用例套件 |
    | `plugin-sdk/channel-contract-testing` | 不使用宽泛测试桶形导出的仓库本地精简渠道契约测试辅助函数 |
    | `plugin-sdk/plugin-test-contracts` | 仓库本地插件包、注册、公开工件、直接导入、运行时 API 和导入副作用契约辅助函数 |
    | `plugin-sdk/plugin-state-test-runtime` | 仓库本地插件状态存储、入口队列和状态数据库测试辅助函数 |
    | `plugin-sdk/provider-test-contracts` | 仓库本地提供商运行时、身份验证、发现、新手引导、目录、向导、媒体能力、重放策略、实时 STT 实况音频、Web 搜索/获取和流契约辅助函数 |
    | `plugin-sdk/provider-http-test-mocks` | 2026 年 7 月后仅限私有本地使用；仓库本地的可选 Vitest HTTP/身份验证模拟，用于测试 `plugin-sdk/provider-http` 的提供商测试 |
    | `plugin-sdk/reply-payload-testing` | 用于向回复载荷夹具附加元数据的仓库本地辅助函数 |
    | `plugin-sdk/sqlite-runtime-testing` | 用于第一方测试的仓库本地 SQLite 生命周期辅助函数 |
    | `plugin-sdk/test-fixtures` | 仓库本地的通用 CLI 运行时捕获、沙箱上下文、技能编写器、智能体消息、系统事件、模块重载、内置插件路径、终端文本、分块、身份验证令牌和类型化用例夹具 |
    | `plugin-sdk/test-node-mocks` | 仓库本地专用 Node 内置模块模拟辅助函数，用于 Vitest `vi.mock("node:*")` 工厂内部 |
  </Accordion>

  <Accordion title="记忆子路径">
    | 子路径 | 主要导出 |
    | --- | --- |
    | `plugin-sdk/memory-core-host-embedding-registry` | 2026 年 7 月后仅限私有本地使用；轻量级记忆嵌入提供商注册表辅助函数 |
    | `plugin-sdk/memory-core-host-engine-foundation` | 记忆宿主基础引擎导出 |
    | `plugin-sdk/memory-core-host-engine-embeddings` | 2026 年 7 月后仅限私有本地使用；记忆宿主嵌入契约、注册表访问、本地提供商以及通用批处理/远程辅助函数。此表面上的 `registerMemoryEmbeddingProvider` 已弃用；新提供商请使用通用嵌入提供商 API。 |
    | `plugin-sdk/memory-core-host-engine-qmd` | 2026 年 7 月后仅限私有本地使用；记忆宿主 QMD 引擎导出 |
    | `plugin-sdk/memory-core-host-engine-storage` | 2026 年 7 月后仅限私有本地使用；记忆宿主存储引擎导出 |
    | `plugin-sdk/memory-core-host-secret` | 2026 年 7 月后仅限私有本地使用；记忆宿主密钥辅助函数 |
    | `plugin-sdk/memory-core-host-status` | 2026 年 7 月后仅限私有本地使用；记忆宿主状态辅助函数 |
    | `plugin-sdk/memory-core-host-runtime-cli` | 2026 年 7 月后仅限私有本地使用；记忆宿主 CLI 运行时辅助函数 |
    | `plugin-sdk/memory-core-host-runtime-core` | 2026 年 7 月后仅限私有本地使用；记忆宿主核心运行时辅助函数 |
    | `plugin-sdk/memory-core-host-runtime-files` | 2026 年 7 月后仅限私有本地使用；记忆宿主文件/运行时辅助函数 |
    | `plugin-sdk/memory-host-core` | 用于供应商中立的记忆宿主辅助函数的已弃用兼容门面。新记忆插件使用注入的记忆能力和宿主预备的提示词；在专用读取接缝可用之前，配套插件仍使用保留的门面发现公开工件。 |
    | `plugin-sdk/memory-host-events` | 2026 年 7 月后仅限私有本地使用；记忆宿主事件日志辅助函数的供应商中立别名 |
    | `plugin-sdk/memory-host-markdown` | 2026 年 7 月后仅限私有本地使用；用于记忆相关插件的共享托管 Markdown 辅助函数 |
    | `plugin-sdk/memory-host-search` | 2026 年 7 月后仅限私有本地使用；用于访问搜索管理器的主动记忆运行时门面 |
  </Accordion>

  <Accordion title="保留的内置辅助子路径">
    保留的内置辅助 SDK 子路径是供内置插件代码使用的精简所有者专用表面。SDK 清单会跟踪这些子路径，以确保软件包构建和别名映射保持确定性，但它们并不是通用的插件开发 API。新的可复用宿主契约应使用通用 SDK 子路径，例如 `plugin-sdk/gateway-runtime` 和 `plugin-sdk/ssrf-runtime`。

    | 子路径 | 所有者和用途 |
    | --- | --- |
    | `plugin-sdk/codex-mcp-projection` | 2026 年 7 月后仅限私有本地使用；内置 Codex 插件辅助函数，用于将用户 MCP 服务器配置投影到 Codex 应用服务器线程配置中（保留的软件包导出） |
    | `plugin-sdk/codex-native-task-runtime` | 内置 Codex 插件辅助函数，用于将 Codex 应用服务器原生子智能体镜像到 OpenClaw 任务状态中（仅限仓库本地使用，不是软件包导出） |

  </Accordion>
</AccordionGroup>

## 相关内容

- [插件 SDK 概览](/zh-CN/plugins/sdk-overview)
- [插件 SDK 设置](/zh-CN/plugins/sdk-setup)
- [构建插件](/zh-CN/plugins/building-plugins)
