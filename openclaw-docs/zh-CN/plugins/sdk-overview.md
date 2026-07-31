---
read_when:
    - 你需要知道应从哪个 SDK 子路径导入
    - 你需要一份关于 `OpenClawPluginApi` 所有注册方法的参考文档
    - 你正在查找特定的 SDK 导出项
sidebarTitle: Plugin SDK overview
summary: 导入映射、注册 API 参考和 SDK 架构
title: 插件 SDK 概览
x-i18n:
    generated_at: "2026-07-26T06:19:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4f490aa8670c57cfc1a635fb1f5d9950fa1cabdb3d45abbc2295da796edcd52e
    source_path: plugins/sdk-overview.md
    workflow: 16
---

插件 SDK 是插件与核心之间的类型化契约。本页介绍**应导入什么**以及**可以注册什么**。

<Note>
  本页面向在 OpenClaw 内部使用 `openclaw/plugin-sdk/*` 的插件作者。对于希望通过 Gateway 网关运行智能体的外部应用、脚本、仪表板、CI 任务和 IDE 扩展，请改用
  [面向外部应用的 Gateway 网关集成](/zh-CN/gateway/external-apps)。
</Note>

<Tip>
想找操作指南？请从[构建插件](/zh-CN/plugins/building-plugins)开始。渠道请参阅[渠道插件](/zh-CN/plugins/sdk-channel-plugins)，模型提供商请参阅[提供商插件](/zh-CN/plugins/sdk-provider-plugins)，本地 AI CLI 后端请参阅[CLI 后端插件](/zh-CN/plugins/cli-backend-plugins)，原生智能体执行器请参阅[Agent harness plugins](/zh-CN/plugins/sdk-agent-harness)，工具或生命周期钩子请参阅[插件钩子](/zh-CN/plugins/hooks)。
</Tip>

## 导入约定

始终从具体的子路径导入：

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

每个子路径都是小型、自包含的模块。这样可以加快启动速度并避免循环依赖问题。对于渠道专用的入口/构建辅助函数，优先使用 `openclaw/plugin-sdk/channel-core`；将 `openclaw/plugin-sdk/core` 保留用于更广泛的总括接口以及 `buildChannelConfigSchema` 等共享辅助函数。

对于渠道配置，通过 `openclaw.plugin.json#channelConfigs` 发布渠道自有的 JSON Schema。`plugin-sdk/channel-config-schema` 子路径用于共享 schema 基元和通用构建器。OpenClaw 的内置插件使用 `plugin-sdk/bundled-channel-config-schema` 处理保留的内置渠道 schema。该内置 schema 子路径不应作为新插件的范式。

<Warning>
  不要导入带有提供商或渠道品牌的便捷接口（例如 `openclaw/plugin-sdk/slack`、`.../discord`、`.../signal`、`.../whatsapp`）。
  内置插件在其自己的 `api.ts` / `runtime-api.ts` barrel 中组合通用 SDK 子路径；核心使用方应使用这些插件本地 barrel，或者在确实存在跨渠道需求时添加范围明确的通用 SDK 契约。

少量内置插件辅助接口在有可追踪的所有者使用记录时，仍会出现在生成的导出映射中。它们仅用于维护内置插件，不建议新第三方插件采用这些导入路径。

`openclaw/plugin-sdk/discord` 和 `openclaw/plugin-sdk/telegram-account` 也作为弃用的兼容门面保留，以支持可追踪的所有者用法。不要在新插件中复制这些导入路径；请改用注入的运行时辅助函数和通用渠道 SDK 子路径。
</Warning>

## 子路径参考

插件 SDK 以一组按领域划分的窄子路径公开（插件入口、渠道、提供商、身份验证、运行时、能力、记忆以及保留的内置插件辅助函数）。完整目录及其分组链接请参阅[插件 SDK 子路径](/zh-CN/plugins/sdk-subpaths)。

编译器入口点清单位于 `scripts/lib/plugin-sdk-entrypoints.json`；类型化公共导出不包括 `scripts/lib/plugin-sdk-private-local-only-subpaths.json` 中列出的内部子路径。该列表中的生产入口会为单独发布的官方插件保留仅限 JavaScript 的宿主运行时导出，而仅用于测试的入口仍不导出。运行 `pnpm plugin-sdk:surface` 可审计公共导出数量。已足够陈旧且不再由内置扩展生产代码使用的弃用公共子路径记录在 `scripts/lib/plugin-sdk-deprecated-public-subpaths.json` 中；广泛的弃用重导出 barrel 记录在 `scripts/lib/plugin-sdk-deprecated-barrel-subpaths.json` 中。

## 注册 API

`register(api)` 回调会接收一个具有以下方法的 `OpenClawPluginApi` 对象：

为会话提供外部团队聊天界面的插件，可以注册由 `openclaw/plugin-sdk/session-discussion` 导出的单个进程级提供商。其 `info({ sessionKey })` 方法会报告讨论是不可用、可打开还是已打开；`open({ sessionKey })` 会创建或解析讨论，并返回其嵌入 URL 和外部 URL。注册另一个提供商会替换当前提供商。

### 能力注册

| 方法                                           | 注册内容                                                                                                                         |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerProvider(...)`                      | 文本推理（LLM）                                                                                                                      |
| `api.registerWorkerProvider(...)`                | 云端工作节点生命周期租约                                                                                                             |
| `api.registerModelCatalogProvider(...)`          | 文本和媒体生成的模型目录行                                                                                          |
| `api.registerAgentHarness(...)`                  | [实验性](/zh-CN/plugins/sdk-agent-harness)原生智能体执行器（Codex、Copilot）                                                         |
| `api.registerCliBackend(...)`                    | 本地 CLI 推理后端                                                                                                               |
| `api.registerChannel(...)`                       | 消息渠道                                                                                                                         |
| `api.registerEmbeddingProvider(...)`             | 可复用的向量嵌入提供商                                                                                                        |
| `api.registerSpeechProvider(...)`                | 文本转语音 / STT 合成                                                                                                            |
| `api.registerRealtimeTranscriptionProvider(...)` | 流式实时转录                                                                                                          |
| `api.registerRealtimeVoiceProvider(...)`         | 双工实时语音会话                                                                                                            |
| `api.registerMediaUnderstandingProvider(...)`    | 图像/音频/视频分析                                                                                                                |
| `api.registerTranscriptSourceProvider(...)`      | 实时或导入的会议转录来源；会议插件可使用 `plugin-sdk/transcripts` 中的 `createMeetingTranscriptSourceProvider` |
| `api.registerImageGenerationProvider(...)`       | 图像生成                                                                                                                          |
| `api.registerMusicGenerationProvider(...)`       | 音乐生成                                                                                                                          |
| `api.registerVideoGenerationProvider(...)`       | 视频生成                                                                                                                          |
| `api.registerWebFetchProvider(...)`              | Web 获取/抓取提供商                                                                                                               |
| `api.registerWebSearchProvider(...)`             | Web 搜索                                                                                                                                |
| `api.registerCompactionProvider(...)`            | 可插拔的转录压缩后端                                                                                                   |

工作节点提供商还必须在 `contracts.workerProviders` 中声明其 ID。
核心会在 `provision(profile, operationId)` 之前持久化持久意图。提供商须在外部分配之前验证设置，并在配置文件被永久拒绝时抛出 `WorkerProviderError`。当操作 ID 重复时，`provision` 必须采用同一租约。
核心会将已验证的配置文件设置与租约一起持久化，并将该快照提供给必须具备幂等性的 `destroy({ leaseId, profile })`，以及返回 `active`、`destroyed` 或 `unknown` 的 `inspect({ leaseId, profile })`。这样，即使 Gateway 网关重启或命名配置文件被移除，提供商仍可路由生命周期调用。SSH 端点使用 `SecretRef` 表示 `keyRef`，绝不内联密钥材料；并包含来自可信预配输出的 `hostKey`，其内容必须恰好为 `algorithm base64`，不得包含主机名或注释。核心固定 `hostKey`，绝不信任首次连接提供的密钥。生成动态 `keyRef` 的提供商可以实现 `resolveSshIdentity({ leaseId, profile, keyRef })`；如果存在，该解析器即为权威来源；未实现它的提供商则使用已配置的通用密钥解析器。
具有可续期租约的提供商还可以实现 `renew(leaseId)`。
`inspect` 在发生暂时性或无法确定的故障时必须抛出异常；仅在权威确认不存在时返回 `unknown`。核心会将活跃的本地记录标记为孤立记录；如果此前已持久化销毁请求，则将该缺失状态视为拆除完成。

使用 `api.registerEmbeddingProvider(...)` 注册的嵌入提供商还必须列入插件清单的 `contracts.embeddingProviders` 中。这是用于可复用向量生成的通用嵌入接口。记忆搜索可以使用此通用提供商接口。较旧的 `api.registerMemoryEmbeddingProvider(...)` 和 `contracts.memoryEmbeddingProviders` 接口属于弃用的兼容机制，供现有的记忆专用提供商迁移使用。

仍公开运行时 `batchEmbed(...)` 的记忆专用提供商继续采用现有的逐文件批处理契约，除非其运行时明确设置 `sourceWideBatchEmbed: true`。启用该选项后，记忆宿主可在宿主批次限制范围内，通过一次 `batchEmbed(...)` 调用提交来自多个已修改记忆文件和已启用来源的分块。上传 JSONL 请求文件的批处理适配器必须在达到上传大小上限和请求数量上限之前拆分提供商任务。提供商必须按照与 `batch.chunks` 相同的顺序，为每个输入分块返回一个嵌入；如果提供商要求批次仅包含单个文件，或无法在更大的全来源任务中保持输入顺序，请省略该标志。

### 工具和命令

对于工具名称固定、仅包含简单工具的插件，请使用 [`defineToolPlugin`](/zh-CN/plugins/tool-plugins)。对于混合型插件或完全动态的工具注册，请直接使用 `api.registerTool(...)`。

| 方法                                 | 注册内容                                                                                                                        |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerTool(tool, opts?)`        | 智能体工具（必需或 `{ optional: true }`）                                                                                            |
| `api.registerCommand(def)`             | 自定义命令（绕过 LLM）                                                                                                        |
| `api.registerNodeHostCommand(command)` | 由 `openclaw node run` 处理的命令；可选的 `agentTool` 元数据可以在节点连接期间将其公开为智能体可见工具 |

当智能体需要简短且归命令所有的路由提示时，插件命令可以设置 `agentPromptGuidance`。该文本应仅描述命令本身；不要向核心提示词构建器添加特定于提供商或插件的策略。

指导条目可以是应用于每个提示词界面的旧版字符串，也可以是结构化条目：

```ts
agentPromptGuidance: [
  "全局命令提示。",
  { text: "仅在 OpenClaw 主提示词中显示此内容。", surfaces: ["openclaw_main"] },
];
```

结构化的 `surfaces` 可包含 `openclaw_main`、`codex_app_server`、
`cli_backend`、`acp_backend` 或 `subagent`。`pi_main` 仍是
`openclaw_main` 的已弃用别名。若要有意向所有表面提供指导，请省略 `surfaces`。不要
传递空的 `surfaces` 数组；系统会拒绝该数组，以免意外丢失作用域
导致提示文本变为全局文本。

原生 Codex app-server 的开发者指令比其他提示
表面更严格：只有明确限定到 `codex_app_server` 的指导才会提升到
这个更高优先级的通道。为保持兼容性，旧版字符串指导和未限定作用域的结构化
指导仍可供非 Codex 提示表面使用。

节点主机命令在已连接的节点主机上运行，而不是在 Gateway 网关
进程内运行。如果存在 `agentTool`，节点会在成功连接
Gateway 网关后发布描述符；仅当该节点保持连接，并且描述符的
`command` 位于节点已批准的命令表面中时，Gateway 网关才会将其提供给智能体
运行。将 `agentTool.defaultPlatforms` 设置为把
非危险命令加入默认节点命令允许列表；否则需要显式
`gateway.nodes.commands.allow` 或节点调用策略。`agentTool.name`
必须符合提供商安全要求：以字母开头，只能使用字母、数字、
下划线或连字符，并且长度不得超过 64 个字符。由 MCP 支持的节点工具
可以设置 `agentTool.mcp` 元数据，以便目录和工具搜索表面显示
远程 MCP 服务器/工具身份，但执行仍通过
公布的节点命令进行。

### 基础设施

| 方法                                            | 注册内容                                                               |
| ----------------------------------------------- | ---------------------------------------------------------------------- |
| `api.registerHook(events, handler, opts?)`      | 事件钩子                                                               |
| `api.registerHttpRoute(params)`                 | Gateway 网关 HTTP 端点                                                 |
| `api.registerGatewayMethod(name, handler)`      | Gateway 网关 RPC 方法                                                  |
| `api.registerGatewayDiscoveryService(service)`  | 本地 Gateway 网关设备发现广播器                                        |
| `api.registerCli(registrar, opts?)`             | CLI 子命令                                                             |
| `api.registerNodeCliFeature(registrar, opts?)`  | `openclaw nodes` 下的节点功能 CLI                                    |
| `api.registerService(service)`                  | 后台服务                                                               |
| `api.registerInteractiveHandler(registration)`  | 交互式处理程序                                                         |
| `api.registerAgentToolResultMiddleware(...)`    | 运行时工具结果中间件                                                   |
| `api.registerMemoryPromptSupplement(builder)`   | 附加的记忆相关提示部分                                                 |
| `api.registerMemoryPromptPreparation(prepare)`  | 为记忆相关提示部分执行异步准备                                         |
| `api.registerMemoryCorpusSupplement(adapter)`   | 附加的记忆搜索/读取语料库                                              |
| `api.registerHostedMediaResolver(resolver)`     | 浏览器式托管媒体 URL 的解析器                                          |
| `api.registerMcpServerConnectionResolver(...)`  | 静态服务器名称对应的按请求者 MCP 传输（`url`/`headers`） |
| `api.registerTextTransforms(transforms)`        | 插件自有的提示/消息兼容性文本重写                                     |
| `api.registerConfigMigration(migrate)`          | 在插件运行时加载前执行的轻量级配置迁移                                |
| `api.registerMigrationProvider(provider)`       | `openclaw migrate` 的导入器                                             |
| `api.registerAutoEnableProbe(probe)`            | 可自动启用此插件的配置探测器                                          |
| `api.registerReload(registration)`              | 用于重新加载处理的重启/热重载/无操作配置前缀策略                      |
| `api.registerNodeHostCommand(command)`          | 向已配对节点公开的命令处理程序                                        |
| `api.registerNodeInvokePolicy(policy)`          | 节点调用命令的允许列表/审批策略                                       |
| `api.registerSecurityAuditCollector(collector)` | `openclaw security audit` 的发现项收集器                                      |

#### 确认响应后的 Webhook 工作

在处理完成前确认请求的 Webhook 路由必须将
这项分离的工作转移到其自身的受跟踪准入根上：

```typescript
import { runDetachedWebhookWork } from "openclaw/plugin-sdk/webhook-request-guards";

void runDetachedWebhookWork(() => processWebhookEvent(event)).catch((error) => {
  runtime.error?.(`Webhook 分派失败：${String(error)}`);
});
```

必须在 HTTP 请求仍处于准入状态时同步调用 `runDetachedWebhookWork(...)`。
该辅助函数会立即预留一个独立根，然后在下一个微任务中启动
回调，以便请求处理程序先写入
确认响应。返回的 Promise 会采用回调结果；调用方
仍负责处理拒绝。这样可确保确认响应后的队列工作被接受，并让
重启或暂停时的排空操作等待该工作完成。返回前会等待所有处理
完成的处理程序不需要此辅助函数。

#### 按请求者限定作用域的 MCP 连接

在 `mcp.servers`、原生插件的 `mcpServers` 清单字段或捆绑包清单中，将 MCP 服务器的**身份**
（名称、工具筛选器）保持为静态。可以选择注册连接解析器，使每个受信任的
消息请求者都获得自己的传输：

```ts
api.registerMcpServerConnectionResolver({
  serverName: "user-email",
  resolve: async (ctx) => {
    // ctx.requesterSenderId 由主机信任；绝不要在这里虚构发送者身份。
    const token = await lookupUserToken(ctx.requesterSenderId);
    if (!token) {
      return null; // 为当前运行省略此服务器
    }
    return {
      url: "https://mcp.example.com/email",
      headers: { Authorization: `Bearer ${token}` },
    };
  },
});
```

契约说明：

- 解析器上下文仅携带受信任的主机身份（`requesterSenderId`，
  以及可选的 `agentAccountId` / `messageChannel`）。未来的受信任字段（例如
  定时任务/子智能体用户上下文）可以通过附加方式添加。
- 一个服务器名称仅归一个插件所有：如果另一个
  插件为同一 `serverName` 重复注册
  `registerMcpServerConnectionResolver`，系统会拒绝并生成错误诊断（首次注册生效），因此
  连接所有权绝不会取决于插件加载顺序。
- 工具名称根据完整的已声明服务器集合派生，因此部分解析
  绝不会导致不同请求者或轮次之间的安全服务器名称发生变化。核心不会
  验证不同请求者端点是否提供相同的工具架构；解析器
  必须让每个请求者都指向同一逻辑服务，否则工具
  架构（以及提示缓存稳定性）会因请求者而异。
- 没有受信任 `requesterSenderId` 的运行（定时任务、子智能体、Heartbeat、公共
  Gateway 网关）绝不会实例化按请求者限定作用域的服务器。不存在共享的
  后备连接。
- 每台服务器的 `resolve` 时限为 10 秒；超时或抛出异常会在该次
  运行中省略相应服务器，但不会导致静态 MCP 失败。
- 每个请求者的已解析连接最多每 5 分钟重新验证一次：
  轮换会使用新凭据重建传输，而 `null` 结果
  会撤销连接（即使会话正在进行，也会释放缓存的运行时）。因此，已撤销或
  已轮换的凭据最多可能继续使用 5 分钟。
- 已解析的 `headers` 绝不会被记录或持久化；核心只保留临时的
  内存键控摘要（进程本地 HMAC）来检测凭据轮换，并将
  已解析的标头/URL 凭据值注册到日志/调试捕获
  脱敏注册表中。
- 按请求者限定作用域的服务器不会创建 MCP App 视图：视图的生命周期长于
  经请求者身份验证的运行，而 Gateway 网关视图边界没有请求者
  身份，因此这些服务器的应用预览保持故障时关闭。工具结果
  不受影响。
- 没有解析器的静态服务器会保持现有的会话作用域生命周期。
- **Harness 交付规则：**按请求者限定作用域的服务器绝不会进入 harness 原生
  MCP 客户端配置（Codex 线程 `mcp_servers`、CLI `-c mcp_servers=…` 或任何
  其他会话共享的 MCP 投影）。Harness 会改为将其作为运行作用域
  工具交付：
  - 嵌入式运行器：会话 MCP 运行时 + 捆绑包工具（静态 + 限定作用域）。
  - Codex app-server：通过
    `materializeRequesterScopedMcpToolsForHarnessRun` 提供动态工具（仅限限定作用域的工具；静态
    服务器仍使用 Codex 的原生 MCP 客户端）。
- 限定作用域的工具**规格**在该会话中首次成功解析后保持会话稳定，
  因此共享线程的 harness（Codex）不会在
  发送者变化时轮换线程。在任何请求者完成解析前，不会公布限定作用域的规格。
- 共享线程 harness 上未经身份验证的请求者仍能看到已公布的
  限定作用域工具；调用此类工具会为该请求者返回明确的未连接工具错误。
  OpenClaw 绝不会退回使用其他请求者的凭据。

记忆提示补充构建器会接收可选的 `agentId`、
`agentSessionKey` 和 `sandboxed` 上下文。记忆语料库补充的 `search`
和 `get` 调用会接收可选的 `agentId` 和 `sandboxed` 上下文。使用
智能体自有存储的插件应在每次调用时解析该存储，而不是
在注册期间捕获一个全局路径。如果多智能体操作需要智能体 ID 但
该 ID 缺失，应采取故障时关闭，而不是选择任意
智能体。

当提示文本依赖异步
插件状态时，使用 `registerMemoryPromptPreparation(...)`。该回调会在每次完整智能体提示前运行一次，并接收
与同步记忆提示构建器相同的工具、智能体、会话和沙箱
上下文。加载持久化状态前验证当前的存储所有者实例，
然后仅返回该次运行所需的行。OpenClaw 会冻结这些行，并将
不可变结果交给同步提示组装。持久化、
原子替换和移除所有者时的删除操作应保留在所属插件内；不要
从提示构建器轮询或读取文件。

Telegram 交互式处理程序可以返回 `{ submitText }`，在处理程序成功后通过
Telegram 的常规入站智能体路径路由文本。如果入站策略跳过该文本或
处理失败，OpenClaw 会保留回调按钮，以便
用户在阻塞条件发生变化后重试。此结果字段
仅适用于 Telegram；其他渠道保留各自的交互结果契约。

### 工作流插件的主机钩子

主机钩子是供需要参与主机
生命周期，而不只是添加提供商、渠道或工具的插件使用的 SDK 接口。它们是
通用契约；Plan Mode 可以使用它们，审批工作流、
工作区策略门控、后台监控器、设置向导和 UI 配套
插件也同样可以使用。

| 方法                                                                               | 所有的契约                                                                                                                                           |
| ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.session.state.registerSessionExtension(...)`                                    | 插件所有、与 JSON 兼容的会话状态，通过 Gateway 网关会话投影                                                                             |
| `api.session.workflow.enqueueNextTurnInjection(...)`                                 | 注入一个会话的下一智能体轮次、持久且恰好一次的上下文                                                                             |
| `api.registerTrustedToolPolicy(...)`                                                 | 由清单控制、受信任且先于插件执行的工具策略，可阻止或重写工具参数                                                                        |
| `api.registerToolMetadata(...)`                                                      | 不更改工具实现的工具目录显示元数据                                                                                     |
| `api.registerCommand(...)`                                                           | 有作用域的插件命令；命令结果可设置 `continueAgent: true` 或 `suppressReply: true`；Discord 原生命令支持 `descriptionLocalizations` |
| `api.session.controls.registerControlUiDescriptor(...)`                              | 用于会话、工具、运行、设置或标签页界面的 Control UI 贡献描述符                                                                      |
| `api.lifecycle.registerRuntimeLifecycle(...)`                                        | 在重置、删除或重新加载路径上清理插件所有运行时资源的回调                                                                          |
| `api.agent.events.registerAgentEventSubscription(...)`                               | 用于工作流状态和监视器的净化事件订阅                                                                                              |
| `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`  | 每次运行的插件暂存状态，在运行终止生命周期时清除                                                                                             |
| `api.session.workflow.registerSessionSchedulerJob(...)`                              | 插件所有调度器任务的清理元数据；不调度工作，也不创建任务记录                                                            |
| `api.session.workflow.sendSessionAttachment(...)`                                    | 仅限内置插件，由主机中介将文件附件传送到活动的直接出站会话路由                                                            |
| `api.session.workflow.scheduleSessionTurn(...)` / `unscheduleSessionTurnsByTag(...)` | 仅限内置插件，由 Cron 支持的定时会话轮次及基于标签的清理                                                                                    |
| `api.session.controls.registerSessionAction(...)`                                    | 客户端可通过 Gateway 网关分派的类型化会话操作                                                                                             |

`surface: "tab"` 描述符会向 Control UI 添加侧边栏标签页。活动
插件的标签页描述符会通过 Gateway 网关
hello（`controlUiTabs`）通告给仪表板客户端，因此该标签页仅在插件启用时显示。
内置插件可以为其标签页提供一流的仪表板视图；其他
插件可将 `path` 设置为插件 HTTP 路由（参见
`api.registerHttpRoute(...)`），仪表板会在沙箱隔离的框架中渲染该路由。
`icon` 是仪表板图标名称提示，`group` 选择侧边栏分区
（`control` 或 `agent`），`order` 对插件标签页排序，而 `requiredScopes`
会对缺少这些操作员权限范围的连接隐藏该标签页：

对于受 Gateway 网关保护的外部标签页，请在同一插件的
`auth: "gateway"` HTTP 路由下注册描述符 `path`。完成身份验证引导后，浏览器会获得一个
作用域限定于该插件和路由根目录、短期有效且采用 HttpOnly 的授权，使
沙箱隔离的框架无需将 Gateway 网关不记名令牌复制到其 URL
或 JavaScript 中即可加载。经过身份验证的父页面会在外部标签页
处于活动状态时续期该授权，并在导航或浏览器恢复后挂载标签页前续期。它还会
在挂载前从同一不透明沙箱中探测该授权，因此
阻止 Cookie 的浏览器隐私模式会以安全关闭方式显示不可用面板。
框架授权仅接受 `GET` 和 `HEAD`，并且始终携带
`operator.read`；`requiredScopes` 控制标签页可见性，但绝不会扩大
Cookie 授权范围。变更操作仍需通过显式的、经 Gateway 网关身份验证的父页面或
不记名令牌界面进行。外部标签页要求使用 HTTPS/Tailscale Serve 或
浏览器信任的 local loopback 源；LAN 主机上的纯 HTTP 会显示
安全上下文错误，而不会挂载无法进行身份验证的面板。
完全阻止第三方 Cookie 也会导致受 Gateway 网关保护的标签页不可用。
与所有原生插件界面一样，该框架仍处于已安装
插件的信任边界内；OpenClaw 不会将已安装插件视为彼此
隔离的浏览器安全主体。
Cookie 授权使用浏览器的主机名边界，而不是端口边界。即使使用其他
端口，也不要在 Gateway 网关主机名上共同托管互不信任的服务。
由插件管理身份验证支持的标签页会保留其直接 iframe 行为，且不会
请求或要求此 Gateway 网关授权。

```typescript
api.session.controls.registerControlUiDescriptor({
  surface: "tab",
  id: "logbook",
  label: "日志簿",
  description: "以屏幕快照构建时间线，记录你的一天。",
  icon: "sun",
  group: "control",
  requiredScopes: ["operator.write"],
});
```

新插件代码请使用分组命名空间：

- `api.session.state.registerSessionExtension(...)`
- `api.session.workflow.enqueueNextTurnInjection(...)`
- `api.session.workflow.registerSessionSchedulerJob(...)`
- `api.session.workflow.sendSessionAttachment(...)`
- `api.session.workflow.scheduleSessionTurn(...)`
- `api.session.workflow.unscheduleSessionTurnsByTag(...)`
- `api.session.controls.registerSessionAction(...)`
- `api.session.controls.registerControlUiDescriptor(...)`
- `api.agent.events.registerAgentEventSubscription(...)`
- `api.agent.events.emitAgentEvent(...)`
- `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`
- `api.lifecycle.registerRuntimeLifecycle(...)`

等效的扁平方法仍作为弃用的兼容性
别名供现有插件使用。不要添加直接调用
`api.registerSessionExtension`、`api.enqueueNextTurnInjection`、
`api.registerControlUiDescriptor`、`api.registerRuntimeLifecycle`、
`api.registerAgentEventSubscription`、`api.emitAgentEvent`、
`api.setRunContext`、`api.getRunContext`、`api.clearRunContext`、
`api.registerSessionSchedulerJob`、`api.registerSessionAction`、
`api.sendSessionAttachment`、`api.scheduleSessionTurn` 或
`api.unscheduleSessionTurnsByTag` 的新插件代码。

`scheduleSessionTurn(...)` 是基于 Gateway 网关
Cron 调度器的会话作用域便捷封装。Cron 负责计时，并在
轮次运行时创建后台任务记录；插件 SDK 仅约束目标会话、插件所有的
命名和清理。当工作本身需要持久的多步骤 Task Flow 状态时，请在定时
轮次中使用 `api.runtime.tasks.managedFlows`。

这些契约有意拆分权限：

- 外部插件可以拥有会话扩展、UI 描述符、命令、工具
  元数据、下一轮注入和常规钩子。
- 受信任的工具策略先于普通 `before_tool_call` 钩子运行，并且
  受主机信任。内置策略首先运行；已安装插件的策略需要
  显式启用，并且其本地 ID 必须列在
  `contracts.trustedToolPolicies` 中，随后按插件加载顺序运行。策略 ID
  的作用域限定于注册该策略的插件。
- 保留命令的所有权仅限内置插件。外部插件应使用自己的
  命令名称或别名。
- `allowPromptInjection=false` 会禁用修改提示词的钩子，包括
  `agent_turn_prepare`、`before_prompt_build`、`heartbeat_prompt_contribution`
  和 `enqueueNextTurnInjection`。

非 Plan 使用方示例：

| 插件原型             | 使用的钩子                                                                                                                             |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| 审批工作流            | 会话扩展、命令延续、下一轮注入、UI 描述符                                                            |
| 预算/工作区策略门禁 | 受信任的工具策略、工具元数据、会话投影                                                                                 |
| 后台生命周期监视器 | 运行时生命周期清理、智能体事件订阅、会话调度器所有权/清理、Heartbeat 提示词贡献、UI 描述符 |
| 设置或新手引导向导   | 会话扩展、有作用域的命令、Control UI 描述符                                                                              |

<Note>
  保留的核心管理命名空间（`config.*`、`exec.approvals.*`、`wizard.*`、
  `update.*`）始终保持 `operator.admin`，即使插件尝试分配
  更窄的 Gateway 网关方法作用域也是如此。对于
  插件所有的方法，优先使用插件专属前缀。
</Note>

<Accordion title="何时使用工具结果中间件">
  当内置插件和已显式启用且具有匹配
  清单契约的已安装插件需要在工具执行后、运行时将结果
  反馈给模型前重写工具结果时，可以使用 `api.registerAgentToolResultMiddleware(...)`。
  这是用于 tokenjuice 等异步输出归约器的、受信任且与运行时无关的
  接缝。

插件必须为每个目标
运行时声明 `contracts.agentToolResultMiddleware`，例如 `["openclaw", "codex"]`。没有该
契约或未显式启用的已安装插件无法注册此中间件；对于不需要在模型处理前
介入工具结果时序的工作，请继续使用普通 OpenClaw 插件钩子。旧的
仅限嵌入式运行器的扩展工厂注册路径已被移除。
</Accordion>

### Gateway 网关设备发现注册

`api.registerGatewayDiscoveryService(...)` 允许插件通过 mDNS/Bonjour 等本地设备发现传输协议通告活动的
Gateway 网关。当本地设备发现已启用时，OpenClaw 会在
Gateway 网关启动期间调用该服务，传入当前 Gateway 网关端口和非机密的 TXT 提示数据，并在
Gateway 网关关闭期间调用返回的 `stop` 处理程序。

```typescript
api.registerGatewayDiscoveryService({
  id: "my-discovery",
  async advertise(ctx) {
    const handle = await startMyAdvertiser({
      gatewayPort: ctx.gatewayPort,
      tls: ctx.gatewayTlsEnabled,
      displayName: ctx.machineDisplayName,
    });
    return { stop: () => handle.stop() };
  },
});
```

Gateway 网关设备发现插件不得将通告的 TXT 值视为机密或
身份验证信息。设备发现只是路由提示；信任仍由 Gateway 网关身份验证和 TLS 固定
负责。

### CLI 注册元数据

`api.registerCli(registrar, opts?)` 接受两类命令元数据：

- `commands`：注册方拥有的显式命令名称
- `descriptors`：用于 CLI 帮助、
  路由和延迟插件 CLI 注册的解析时命令描述符
- `parentPath`：嵌套命令组的可选父命令路径，例如
  `["nodes"]`

对于已配对节点功能，优先使用
`api.registerNodeCliFeature(registrar, opts?)`。它是
`api.registerCli(..., { parentPath: ["nodes"] })` 的轻量封装，可将
`openclaw nodes canvas` 等命令明确标记为插件所有的节点功能。

如果希望插件命令在常规根 CLI 路径中保持延迟加载，
请提供覆盖该注册方公开的每个顶层命令根的 `descriptors`。

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "管理 Matrix 账户、验证、设备和个人资料状态",
        hasSubcommands: true,
      },
    ],
  },
);
```

嵌套命令接收解析后的父命令作为 `program`：

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerNodesCanvasCommands } = await import("./src/cli.js");
    registerNodesCanvasCommands(program);
  },
  {
    parentPath: ["nodes"],
    descriptors: [
      {
        name: "canvas",
        description: "从已配对节点捕获或渲染画布内容",
        hasSubcommands: true,
      },
    ],
  },
);
```

仅当不需要延迟注册根 CLI 时，才单独使用 `commands`。
该即时兼容路径仍受支持，但它不会安装
由描述符支持的占位符来实现解析时延迟加载。

### CLI 后端注册

`api.registerCliBackend(...)` 允许插件拥有本地
AI CLI 后端（例如 `claude-cli` 或 `my-cli`）的默认配置。

- 后端 `id` 会成为模型引用（例如 `my-cli/gpt-5`）中的提供商前缀。
- 后端 `config` 是权威命令适配器：argv、环境、
  解析器、会话、图像和可靠性行为均位于插件代码中。
- 用户通过模型引用或模型范围的 `agentRuntime.id` 选择后端；
  `openclaw.json` 不会重写适配器。
- 当已注册的静态字段需要运行时感知的
  规范化处理时，请使用 `normalizeConfig`。
- 对于属于 CLI 方言的请求范围 argv 重写，请使用 `resolveExecutionArgs`，
  例如将 OpenClaw 思考级别映射到原生工作量
  标志。该钩子接收 `ctx.executionMode`；使用 `"side-question"` 为临时 `/btw` 调用添加
  后端原生隔离标志。如果这些标志
  能可靠地为原本始终启用原生工具的 CLI 禁用这些工具，还应声明
  `sideQuestionToolMode: "disabled"`。
- 对于后端拥有的启动环境或临时
  身份验证/配置桥接，请使用 `prepareExecution`。其 `ctx.contextTokenBudget` 是为本次运行选择的有效令牌
  限制，因此支持原生压缩的后端可以对齐自身
  阈值，而无需在核心中添加提供商专用分支。它还会接收
  核心准备的 `ctx.env`，以供后端暂存需要扩展内置 MCP 设置时使用。
- 能够为特定运行禁用所有原生工具的后端可以声明
  `nativeToolMode: "selectable"`。受限调用会传入精确的
  `ctx.toolAvailability.native` 列表以及规范的
  `ctx.toolAvailability.openClaw` 名称。请声明
  `toolAvailabilityEnforcement: "execution-args"` 并在最终的新建/恢复 argv 中强制执行该契约，或者声明 `"prepare-execution"`，
  在暂存策略中强制执行该契约，并返回 `toolAvailabilityEnforced: true`。OpenClaw 会针对定时任务 `toolsAllow` 等运行时能力限制
  禁用原生工具，并在声明的执行路径不完整时以关闭方式失败。

有关端到端编写指南，请参阅
[CLI 后端插件](/zh-CN/plugins/cli-backend-plugins)。

### 独占槽位

| 方法                                     | 注册内容                                                                                                                                                                                  |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | 上下文引擎（一次只能有一个处于活动状态）。当宿主可以提供模型/提供商/模式诊断信息时，生命周期回调会接收 `runtimeSettings`；对于旧版严格引擎，将在不带该键的情况下重试。 |
| `api.registerMemoryCapability(capability)` | 统一记忆能力                                                                                                                                                                          |

### 已弃用的记忆嵌入适配器

| 方法                                         | 注册内容                              |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | 活动插件的记忆嵌入适配器 |

- `registerMemoryCapability` 是独占的记忆插件 API。
- `registerMemoryCapability` 还可以公开 `publicArtifacts.listArtifacts(...)`
  以用于宿主管理的导出。枚举这些已声明
  工件的配套插件仍使用保留的
  `openclaw/plugin-sdk/memory-host-core` 外观中的 `listActiveMemoryPublicArtifacts(...)`，直至出现专门的公共使用方
  API；它们不得访问其他插件的私有布局。
- `MemoryFlushPlan.model` 可以将刷新轮次固定到精确的 `provider/model`
  引用（例如 `ollama/qwen3:8b`），而不继承活动的回退
  链。
- `registerMemoryEmbeddingProvider` 已弃用。新的嵌入提供商
  应使用 `api.registerEmbeddingProvider(...)` 和
  `contracts.embeddingProviders`。
- 现有的记忆专用提供商在迁移
  窗口期间将继续工作，但对于非内置插件，插件检查会将其报告为兼容性债务。

### 事件和生命周期

| 方法                                       | 作用                  |
| -------------------------------------------- | ----------------------------- |
| `api.on(hookName, handler, opts?)`           | 类型化生命周期钩子          |
| `api.onConversationBindingResolved(handler)` | 对话绑定回调 |

有关示例、常用钩子名称和守卫
语义，请参阅[插件钩子](/zh-CN/plugins/hooks)。

### 钩子决策语义

`before_install` 是插件运行时生命周期钩子，而不是操作员安装
策略界面。当允许/阻止决策必须
覆盖 CLI 和 Gateway 网关支持的安装或更新路径时，请使用 `security.installPolicy`。

- `before_tool_call`：返回 `{ block: true }` 即为终止。一旦任何处理程序设置它，便会跳过优先级更低的处理程序。
- `before_tool_call`：返回 `{ block: false }` 会被视为未作决定（与省略 `block` 相同），而不是覆盖。
- `before_install`：返回 `{ block: true }` 即为终止。一旦任何处理程序设置它，便会跳过优先级更低的处理程序。
- `before_install`：返回 `{ block: false }` 会被视为未作决定（与省略 `block` 相同），而不是覆盖。
- `reply_dispatch`：返回 `{ handled: true, ... }` 即为终止。一旦任何处理程序声明接管分发，便会跳过优先级更低的处理程序和默认模型分发路径。
- `message_sending`：返回 `{ cancel: true }` 即为终止。一旦任何处理程序设置它，便会跳过优先级更低的处理程序。
- `message_sending`：返回 `{ cancel: false }` 会被视为未作决定（与省略 `cancel` 相同），而不是覆盖。
- `message_received`：需要入站线程/主题路由时，请使用类型化的 `threadId` 字段。将 `metadata` 保留用于渠道专用的额外内容。
- `message_sending`：回退到渠道专用的 `metadata` 之前，请先使用类型化的 `replyToId` / `threadId` 路由字段。
- `gateway_start`：对于 Gateway 网关拥有的启动状态，请使用 `ctx.config`、`ctx.workspaceDir` 和 `ctx.getCron?.()`，而不是依赖内部 `gateway:startup` 钩子。此时定时任务可能仍在加载。
- `cron_reconciled`：在启动或调度器重新加载后重建完整的外部定时任务投影。它包含 `reason` 和有效的 `enabled` 状态（包括 `enabled: false`），而 `ctx.getCron?.()` 返回准确的已协调调度器。将 `ctx.abortSignal` 传入持久投影工作；当该调度器快照被取代或 Gateway 网关关闭时，它会中止。
- `cron_changed`：观察 Gateway 网关拥有的定时任务生命周期变化。`scheduled` 和 `removed` 事件是提交后的协调提示，而不是有序的增量日志。当作业没有下次唤醒时间时，计划事件的 `event.nextRunAtMs` 不存在；移除事件仍携带已删除作业的快照。

外部唤醒调度器应对 `cron_changed` 事件进行防抖或合并，
然后从 `cron_reconciled` 最后捕获的调度器中重新读取完整的持久视图。
不要采用 `cron_changed` 上下文中的调度器：
来自旧调度器的分离提示可能与后续重新加载重叠。

对于 Gateway 网关启动或调度器替换时加载的持久状态，请使用 `cron_reconciled`
作为完整快照触发器。仅插件热重载时不会重放该触发器。
观察处理程序并行运行，触发后不等待结果的
分发可能重叠，因此使用方不得依赖事件完成顺序。
应始终将 OpenClaw 作为到期检查和执行的事实来源。

有关具备持久替换、重试/退避和干净
关闭机制的单次执行适配器，请参阅[安全的外部定时任务投影](/zh-CN/plugins/hooks#safe-external-cron-projection)。

### API 对象字段

| 字段                    | 类型                      | 描述                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | 插件 ID                                                                                   |
| `api.name`               | `string`                  | 显示名称                                                                                |
| `api.version`            | `string?`                 | 插件版本（可选）                                                                   |
| `api.description`        | `string?`                 | 插件描述（可选）                                                               |
| `api.source`             | `string`                  | 插件源路径                                                                          |
| `api.rootDir`            | `string?`                 | 插件根目录（可选）                                                            |
| `api.config`             | `OpenClawConfig`          | 当前配置快照（如果可用，则为活动的内存中运行时快照）                  |
| `api.pluginConfig`       | `Record<string, unknown>` | 来自 `plugins.entries.<id>.config` 的插件专用配置                                   |
| `api.runtime`            | `PluginRuntime`           | [运行时辅助工具](/zh-CN/plugins/sdk-runtime)                                                     |
| `api.logger`             | `PluginLogger`            | 作用域限定的日志记录器（`debug`、`info`、`warn`、`error`）                                            |
| `api.registrationMode`   | `PluginRegistrationMode`  | 当前加载模式；`"setup-runtime"` 是进入完整入口之前的轻量级启动/设置窗口 |
| `api.resolvePath(input)` | `(string) => string`      | 解析相对于插件根目录的路径                                                        |

## 内部模块约定

在插件内部，对内部导入使用本地桶文件：

```text
my-plugin/
  api.ts            # 面向外部使用方的公共导出
  runtime-api.ts    # 仅供内部使用的运行时导出
  index.ts          # 插件入口点
  setup-entry.ts    # 仅用于设置的轻量级入口（可选）
```

<Warning>
  切勿在生产代码中通过 `openclaw/plugin-sdk/<your-plugin>`
  导入你自己的插件。内部导入应通过 `./api.ts` 或
  `./runtime-api.ts`。SDK 路径仅作为外部契约。
</Warning>

通过 facade 加载的内置插件公共接口（`api.ts`、`runtime-api.ts`、
`index.ts`、`setup-entry.ts` 及类似的公共入口文件）在 OpenClaw 已运行时优先使用
当前运行时配置快照。如果尚无运行时快照，则回退到磁盘上解析后的配置文件。
打包的内置插件 facade 应通过 OpenClaw 的插件 facade 加载器加载；直接从
`dist/extensions/...` 导入会绕过打包安装在处理插件自有代码时使用的清单和运行时 sidecar 检查。

当某个辅助函数明确特定于提供商，且尚不适合放入通用 SDK 子路径时，
提供商插件可以公开范围有限的插件本地契约 barrel。内置示例：

- **Anthropic**：面向 Claude beta-header 和 `service_tier`
  流辅助函数的公共 `api.ts` / `contract-api.ts` 接口。
- **`@openclaw/openai-provider`**：`api.ts` 导出提供商构建器、
  默认模型辅助函数和实时提供商构建器。
- **`@openclaw/openrouter-provider`**：`api.ts` 导出提供商构建器
  以及新手引导/配置辅助函数。

<Warning>
  扩展的生产代码也应避免从 `openclaw/plugin-sdk/<other-plugin>`
  导入。如果某个辅助函数确实是共享的，应将其提升到中立的 SDK 子路径，
  例如 `openclaw/plugin-sdk/speech`、`.../provider-model-shared` 或其他
  面向能力的接口，而不是将两个插件耦合在一起。
</Warning>

## 相关内容

<CardGroup cols={2}>
  <Card title="入口点" icon="door-open" href="/zh-CN/plugins/sdk-entrypoints">
    `definePluginEntry` 和 `defineChannelPluginEntry` 选项。
  </Card>
  <Card title="运行时辅助函数" icon="gears" href="/zh-CN/plugins/sdk-runtime">
    完整的 `api.runtime` 命名空间参考。
  </Card>
  <Card title="设置和配置" icon="sliders" href="/zh-CN/plugins/sdk-setup">
    打包、清单和配置 schema。
  </Card>
  <Card title="测试" icon="vial" href="/zh-CN/plugins/sdk-testing">
    测试工具和 lint 规则。
  </Card>
  <Card title="SDK 迁移" icon="arrows-turn-right" href="/zh-CN/plugins/sdk-migration">
    从已弃用的接口迁移。
  </Card>
  <Card title="插件内部机制" icon="diagram-project" href="/zh-CN/plugins/architecture">
    深入了解架构和能力模型。
  </Card>
</CardGroup>
