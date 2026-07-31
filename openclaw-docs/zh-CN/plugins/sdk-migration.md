---
read_when:
    - 你看到 OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED 警告
    - 你看到 OPENCLAW_EXTENSION_API_DEPRECATED 警告
    - 你在 OpenClaw 2026.4.25 之前使用了 api.registerEmbeddedExtensionFactory
    - 你正在将插件更新为现代插件架构
    - 你维护一个外部 OpenClaw 插件
sidebarTitle: Migrate to SDK
summary: 从旧版向后兼容层迁移到现代插件 SDK
title: 插件 SDK 迁移
x-i18n:
    generated_at: "2026-07-26T05:57:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a483f9c0f8409505fc2688872995382944e002520ceb651214dbc5ad8e3554fb
    source_path: plugins/sdk-migration.md
    workflow: 16
---

OpenClaw 已将宽泛的向后兼容层替换为由小型、聚焦导入构成的现代插件
架构。如果你的插件早于这一变更，本指南可帮助它迁移到当前契约。

## 变更内容

过去，多个完全开放的导入表面允许插件从单一入口点访问几乎所有内容：

- **`openclaw/plugin-sdk`** 和 **`openclaw/plugin-sdk/compat`** - 在构建聚焦式 SDK
  期间重新导出了数十个辅助程序。现在这两个根入口均已移除；请改为导入有文档说明的子路径。
- **`openclaw/plugin-sdk/infra-runtime`** - 一个宽泛的桶式导出，混合了系统
  事件、Heartbeat 状态、投递队列、fetch/代理辅助程序、文件辅助程序、
  审批类型和不相关的实用程序。
- **`openclaw/plugin-sdk/config-runtime`** - 一个宽泛的配置桶式导出，仅为其
  后续兼容窗口而保留；直接运行时加载/写入辅助程序已移除。
- **`openclaw/extension-api`** - 一个已移除的桥接层，曾允许插件直接
  访问主机端辅助程序，例如嵌入式智能体运行器。
- **`api.registerEmbeddedExtensionFactory(...)`** - 一个已移除的、仅供嵌入式运行器使用的
  钩子，曾观察 `tool_result` 等嵌入式运行器事件。请改用智能体
  工具结果中间件（参阅[将嵌入式工具结果扩展迁移到
  中间件](#how-to-migrate)）。

根 SDK、兼容桶式导出、扩展桥接层和嵌入式扩展工厂
均已移除。`infra-runtime` 和 `config-runtime` 仅在各自单独记录的
后续窗口中保留；新插件应使用聚焦式子路径。

<Warning>
  导入已移除的根入口、兼容表面或扩展表面的插件将无法再
  加载。升级前请遵循下方的映射关系。
</Warning>

OpenClaw 不会在引入替代方案的同一变更中移除或重新解释已有文档说明的
插件行为。破坏性契约变更会先经过兼容适配器、诊断、文档和弃用窗口。
这适用于 SDK 导入、清单字段、设置 API、钩子和运行时
注册行为。

### 原因

- **启动缓慢** - 导入一个辅助程序会加载数十个不相关的模块。
- **循环依赖** - 宽泛的重新导出很容易产生导入循环。
- **API 表面不明确** - 无法区分稳定导出与内部导出。

现在，每个 `openclaw/plugin-sdk/<subpath>` 都是一个小型、自包含且
具有明确文档契约的模块。

面向内置渠道的旧提供商便捷接口也已移除 -
以渠道命名的辅助程序快捷方式只是私有单体仓库中的便利设施，并非
稳定的插件契约。请改用精简的通用 SDK 子路径。在
内置插件工作区内，将提供商拥有的辅助程序保留在该插件自身的
`api.ts` 或 `runtime-api.ts` 中：

- Anthropic 将 Claude 专用的流式辅助程序保留在自身的 `api.ts` /
  `contract-api.ts` 接口中。
- OpenAI 将提供商构建器、默认模型辅助程序和实时提供商
  构建器保留在自身的 `api.ts` 中。
- OpenRouter 将提供商构建器及新手引导/配置辅助程序保留在自身的
  `api.ts` 中。

## 兼容策略

外部插件兼容性工作遵循以下顺序：

1. 添加新契约。
2. 通过兼容适配器继续接入旧行为。
3. 发出诊断或警告，指出旧路径和替代路径。
4. 在测试中覆盖两条路径。
5. 记录弃用和迁移路径。
6. 仅在公布的迁移窗口结束后移除，通常是在一个主
   版本中。

如果某个清单字段仍被接受，请继续使用它，直到文档和
诊断另有说明。新代码应优先采用有文档说明的替代方案；
现有插件不应在普通次版本发布期间中断。

### 已发布渠道设置兼容性

通过 `2026.7.1` 发布的 Slack、Discord、Signal 和 Microsoft Teams
软件包会从 `openclaw/plugin-sdk/bundled-channel-config-schema` 导入
渠道专用配置架构。已发布的 Slack 和
Discord 软件包还会从
`openclaw/plugin-sdk/setup-runtime` 导入 `createLegacyCompatChannelDmPolicy` 和
`promptLegacyChannelAllowFromForAccount`。

这些导出仍作为已弃用的运行时兼容适配器提供。
新插件和重新发布的插件应在本地拥有自身的配置架构和设置策略，
并使用 `channel-config-schema` 和
`setup-runtime` 中的通用原语。只有在最低支持的已发布软件包版本
不再导入这些兼容导出后，才能将其移除。

### 渠道设置输入字段兼容性

`ChannelSetupInput` 现在仅永久保留跨渠道设置封装的类型定义。
渠道专用字段仍在已弃用的兼容层级中保留类型定义，
以便现有外部插件在插件作者将这些字段迁移到插件本地设置输入类型期间
仍可编译。

OpenClaw 不发布主版本。2026-07-22 的注册表扫描检查了
426 个已发布的树外渠道插件，并移除了 21 个没有读取方的字段。
保留的 22 个字段各自都有已知的已发布读取方。只要某个字段不再被
任何已发布插件读取，就会立即将其删除；随着插件作者迁移到插件本地
设置输入类型，保留集合会不断缩小。

同一次扫描还移除了 23 个没有已发布依赖方的旧版未声明适配器提升键。
仍保留六个常用键以及仅用于设置的 `rooms` 键。
随着已发布插件声明 `singleAccountKeysToMove`，该集合也会不断缩小。

共享类型没有索引签名。插件拥有的键仍可存在于
运行时输入对象上；请在插件本地交叉类型中声明它们，或通过
所属插件的设置架构收窄其类型。

| `code`                                  | `owner`   | `replacement`                                                                                    | 移除条件                                                     |
| --------------------------------------- | --------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| `plugin-sdk-channel-setup-input-fields` | `channel` | 将 `ChannelSetupInput` 与声明所属渠道字段的插件本地类型进行交叉 | 当已发布插件注册表扫描中没有读取方时删除该字段 |

旧版未声明适配器提升层级遵循相同的读取方驱动
策略。请声明 `singleAccountKeysToMove`；当插件不需要额外提升键时，也应包含空数组，
以便逐个键停用共享回退机制。

#### 验证读取方

1. 使用每个 `nextCursor` 对 `https://clawhub.ai/api/v1/packages?family=code-plugin&limit=100` 进行分页查询，并保留其 `categories` 包含 `channels` 的软件包。
2. 添加来自 `npm search --json --searchlimit=1000 "openclaw channel plugin"` 的 npm 候选项。通过在 GitHub 中搜索 `openclaw/plugin-sdk/channel-setup`、`openclaw/plugin-sdk/setup` 和 `openclaw/plugin-sdk/core` 的代码，添加仅有源代码的候选项。
3. 解析每个候选项的最新已发布版本。运行 `npm pack <package>@<version> --json --pack-destination <temp-dir>`，将其解包，并检查已发布的 `dist` JavaScript 和声明中是否存在直接或解构字段读取。如果软件包没有 npm 版本，请下载 ClawHub 工件。
4. 记录软件包、版本、字段或提升键以及匹配文件。只有在没有任何已发布插件工件读取某个字段或键时，才能删除它。确保保留字段和键列表旁代码注释中的读取方名称与扫描结果保持同步。

这只是源代码/类型兼容性记录。它没有运行时适配器或
兼容性注册表条目，因为运行时设置输入对象和设置
行为均未改变。

使用 `pnpm plugins:boundary-report` 审计当前迁移队列：

| 标志                                                    | 效果                                                                         |
| ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `--summary`（或 `pnpm plugins:boundary-report:summary`） | 显示精简计数，而非完整详情。                                         |
| `--json`                                                | 生成机器可读报告。                                                       |
| `--owner <id>`                                          | 筛选到单个插件或兼容性所有者。                                   |
| `--fail-on-cross-owner`                                 | 遇到跨所有者的保留 SDK 导入时以非零状态退出。                             |
| `--fail-on-eligible-compat`                             | 当已弃用兼容记录的 `removeAfter` 日期已过时，以非零状态退出。 |
| `--fail-on-unclassified-unused-reserved`                | 遇到未使用的保留 SDK 垫片时以非零状态退出。                                    |

`pnpm plugins:boundary-report:ci` 会启用全部三个失败标志。已弃用的
记录通常会设置明确的 `removeAfter` 日期，而不是含糊的“下一个
主版本”。如果记录的所有者尚未批准日期，则不设置
`removeAfter`，该记录会显示为 `no-date`，并且永远不符合移除条件。
报告按日期对已弃用记录进行分组，统计本地代码/文档引用，
显示跨所有者的保留 SDK 导入，并汇总私有
内存主机 SDK 桥接层。保留的 SDK 子路径必须有可追踪的所有者使用记录；
未使用的保留导出应从公共 SDK 中移除。

### 旧版媒体投影

`media-legacy-projection` 兼容性记录涵盖旧的并行
媒体字段、有效负载构建器、钩子元数据别名和媒体模板
名称。其批准的 `removeAfter` 日期为 **2026-10-01**（在事实优先的
替代方案发布两个发布周期之后）。届时还必须完成一次无异常的已发布插件
工件扫描才能移除；请在该日期前完成迁移。

对于渠道入口，请将单数/复数形式的 `MediaPath`、`MediaUrl`、
`MediaType`、`MediaPaths`、`MediaUrls`、`MediaTypes`、
`MediaTranscribedIndexes`、`MediaWorkspaceDir` 和 `MediaStaged` 替换为有序
事实：

```ts
import { toInboundMediaFacts } from "openclaw/plugin-sdk/channel-inbound";

const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

在 `inbound_claim` 和 `message_received` 钩子中使用 `event.media`。如果远程
媒体尚未在本地暂存，请使用 `event.originalMedia` 进行身份标识/诊断，
并等待 `event.media`；`event.mediaStagingPending` 可区分该
状态。不要从 `event.metadata` 读取已弃用的单数/复数属性。

对于 CLI 媒体模型，请将 `{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}`
和 `{{MediaDir}}` 替换为 `{{AttachmentPath}}`、`{{AttachmentUrl}}`、
`{{AttachmentContentType}}` 和 `{{AttachmentDir}}`。当附件位置很重要时，请使用
`{{AttachmentIndex}}`。

对于本地媒体读取策略，请从
`openclaw/plugin-sdk/media-local-roots` 导入 `getAgentScopedMediaLocalRoots(...)` 或
`getAgentScopedMediaLocalRootsForSources(...)`。
`openclaw/plugin-sdk/agent-media-payload` 门面及其
`buildAgentMediaPayload(...)` 投影已弃用。

## 如何迁移

<Steps>
  <Step title="迁移运行时配置加载/写入辅助程序">
    内置插件应停止直接调用 `api.runtime.config.loadConfig()` 和
    `api.runtime.config.writeConfigFile(...)`。应优先使用已传入当前有效
    调用路径的配置。需要当前进程快照的长期运行处理程序
    可以使用 `api.runtime.config.current()`。长期运行的
    智能体工具应在 `execute` 内读取 `ctx.getRuntimeConfig()`，这样在配置写入前
    创建的工具仍能看到刷新后的配置。

    配置写入应通过事务辅助程序执行，并明确指定
    写入后策略：

    ```typescript
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    当更改需要彻底重启 Gateway 网关时，使用 `afterWrite: { mode: "restart", reason: "..." }`；仅当调用方负责后续处理并有意抑制重新加载规划器时，才使用 `afterWrite: { mode: "none", reason: "..." }`。变更结果包含用于测试和日志记录的类型化 `followUp` 摘要；Gateway 网关仍负责执行或安排重启。

    `loadConfig` 和 `writeConfigFile` 已从插件运行时中移除。内置插件和仓库运行时代码受
    `pnpm check:deprecated-api-usage` 和
    `pnpm check:no-runtime-action-load-config` 保护：新的生产插件用法会直接失败，直接写入配置会失败，Gateway 网关服务器方法必须使用请求运行时快照，运行时渠道发送/操作/客户端辅助函数必须从其边界接收配置，并且长生命周期运行时模块不允许调用任何环境中的 `loadConfig()`。

    新插件代码应避免使用宽泛的 `openclaw/plugin-sdk/config-runtime`
    聚合入口。请根据任务使用相应的精确子路径：

    | 需求 | 导入 |
    | --- | --- |
    | `OpenClawConfig` 等配置类型 | `openclaw/plugin-sdk/config-contracts` |
    | 插件入口配置查找 | `api.pluginConfig` |
    | 配置合并 | 配置边界处的插件本地逻辑 |
    | 读取当前运行时快照 | `openclaw/plugin-sdk/runtime-config-snapshot` |
    | 配置写入 | `openclaw/plugin-sdk/config-mutation` |
    | 会话存储辅助函数 | `openclaw/plugin-sdk/session-store-runtime` |
    | Markdown 表格配置 | `openclaw/plugin-sdk/markdown-table-runtime` |
    | 群组策略运行时辅助函数 | `openclaw/plugin-sdk/runtime-group-policy` |
    | 机密输入解析 | `openclaw/plugin-sdk/secret-input-runtime` |
    | 模型/会话覆盖项 | `openclaw/plugin-sdk/model-session-runtime` |

    内置插件及其测试受到扫描器保护，禁止使用宽泛的聚合入口，以便导入和模拟仅限于所需行为。该聚合入口仍为外部兼容性而保留，但新代码不应依赖它。

  </Step>

  <Step title="将嵌入式工具结果扩展迁移到中间件">
    内置插件必须将仅适用于嵌入式运行器的
    `api.registerEmbeddedExtensionFactory(...)` 工具结果处理程序替换为
    不依赖运行时的中间件：

    ```typescript
    // OpenClaw 运行时工具和 Codex 运行时动态工具（结果可能会被
    // 转换）。Codex 原生工具结果也会被转发以供观察，
    // 但其转换后的输出绝不会传递给模型：Codex
    // PostToolUse 钩子契约无法替换原生工具响应。
    api.registerAgentToolResultMiddleware(async (event) => {
      return compactToolResult(event);
    }, {
      runtimes: ["openclaw", "codex"],
    });
    ```

    同时更新插件清单：

    ```json
    {
      "contracts": {
        "agentToolResultMiddleware": ["openclaw", "codex"]
      }
    }
    ```

    已安装的插件在明确启用并且所有目标运行时均已在
    `contracts.agentToolResultMiddleware` 中声明时，也可以注册工具结果中间件。未声明的已安装中间件注册会被拒绝。

  </Step>

  <Step title="将原生审批处理程序迁移到能力事实">
    支持审批的渠道插件通过
    `approvalCapability.nativeRuntime` 和共享运行时上下文注册表公开原生审批行为：

    - 将 `approvalCapability.handler.loadRuntime(...)` 替换为
      `approvalCapability.nativeRuntime`。
    - 将审批专用的身份验证/投递从旧版 `plugin.auth` /
      `plugin.approvals` 接线迁移到 `approvalCapability`。
    - `ChannelPlugin.approvals` 已从公共渠道插件契约中移除；请将投递/原生/渲染字段迁移到
      `approvalCapability`。
    - `plugin.auth` 仅保留用于渠道登录/退出流程；核心不再从中读取审批身份验证钩子。
    - 通过 `openclaw/plugin-sdk/channel-runtime-context` 注册渠道拥有的运行时对象（客户端、令牌、Bolt 应用）。
    - 不要从原生审批处理程序发送插件拥有的重新路由通知；核心根据实际投递结果负责发送“已路由至其他位置”通知。
    - 将 `channelRuntime` 传入 `createChannelManager(...)` 时，请提供真实的 `createPluginRuntime().channel` 接口——不完整的存根会被拒绝。

    有关当前审批能力布局，请参阅[渠道插件](/zh-CN/plugins/sdk-channel-plugins)。

  </Step>

  <Step title="审核 Windows 包装器的回退行为">
    如果你的插件使用 `openclaw/plugin-sdk/windows-spawn`，则无法解析的 Windows
    `.cmd`/`.bat` 包装器现在会以关闭方式失败，除非你明确传入
    `allowShellFallback: true`：

    ```typescript
    // 之前
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // 之后
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // 仅为有意接受经由 shell 回退的可信兼容性调用方
      // 设置此项。
      allowShellFallback: true,
    });
    ```

    如果调用方并非有意依赖 shell 回退，请勿设置
    `allowShellFallback`，而应处理抛出的错误。

  </Step>

  <Step title="查找已弃用的导入">
    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "plugin-sdk/infra-runtime" my-plugin/
    grep -r "plugin-sdk/config-runtime" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```
  </Step>

  <Step title="替换为精确导入">
    旧接口中的每个导出均映射到特定的现代导入路径：

    ```typescript
    // 之前（已弃用的向后兼容层）
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // 之后（现代精确导入）
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    对于主机端辅助函数，请使用注入的插件运行时，而非直接导入：

    ```typescript
    // 之前（已弃用的 extension-api 桥接）
    import { runEmbeddedAgent } from "openclaw/extension-api";
    const result = await runEmbeddedAgent({ sessionId, prompt });

    // 之后（注入的运行时）
    const result = await api.runtime.agent.runEmbeddedAgent({ sessionId, prompt });
    ```

    其他旧版桥接辅助函数也采用相同模式：

    | 旧导入 | 现代等效项 |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | 会话存储辅助函数 | `api.runtime.agent.session.*` |

  </Step>

  <Step title="替换宽泛的 infra-runtime 导入">
    `openclaw/plugin-sdk/infra-runtime` 仍为外部兼容性而保留，但新代码应导入实际需要的精确接口：

    | 需求 | 导入 |
    | --- | --- |
    | 系统事件队列辅助函数 | `openclaw/plugin-sdk/system-event-runtime` |
    | Heartbeat 唤醒、事件和可见性辅助函数 | `openclaw/plugin-sdk/heartbeat-runtime` |
    | 排空待处理投递队列 | `openclaw/plugin-sdk/delivery-queue-runtime` |
    | 渠道活动遥测 | `openclaw/plugin-sdk/channel-activity-runtime` |
    | 内存和持久化后端去重缓存 | `openclaw/plugin-sdk/dedupe-runtime` |
    | 安全的本地文件/媒体路径辅助函数 | `openclaw/plugin-sdk/file-access-runtime` |
    | 感知调度器的获取操作 | `openclaw/plugin-sdk/runtime-fetch` |
    | 代理和受保护的获取辅助函数 | `openclaw/plugin-sdk/fetch-runtime` |
    | SSRF 调度器策略类型 | `openclaw/plugin-sdk/ssrf-dispatcher` |
    | 审批请求/解决类型 | `openclaw/plugin-sdk/approval-runtime` |
    | 审批回复载荷和命令辅助函数 | `openclaw/plugin-sdk/approval-reply-runtime` |
    | 错误格式化辅助函数 | `openclaw/plugin-sdk/error-runtime` |
    | 等待传输就绪 | `openclaw/plugin-sdk/transport-ready-runtime` |
    | 安全令牌辅助函数 | `openclaw/plugin-sdk/secure-random-runtime` |
    | 有界异步任务并发 | `openclaw/plugin-sdk/concurrency-runtime` |
    | 用于可证明不变量的必需值断言 | `openclaw/plugin-sdk/expect-runtime` |
    | 数值强制转换 | `openclaw/plugin-sdk/number-runtime` |
    | 进程本地异步锁 | `openclaw/plugin-sdk/async-lock-runtime` |
    | 文件锁 | `openclaw/plugin-sdk/file-lock` |

    内置插件受到扫描器保护，禁止使用 `infra-runtime`，因此仓库代码无法退回到宽泛的聚合入口。

  </Step>

  <Step title="迁移渠道路由辅助函数">
    新渠道路由代码使用 `openclaw/plugin-sdk/channel-route`。较旧的路由键名称仍作为兼容性别名保留：

    | 旧辅助函数 | 现代辅助函数 |
    | --- | --- |
    | `channelRouteIdentityKey(...)` | `channelRouteDedupeKey(...)` |
    | `channelRouteKey(...)` | `channelRouteCompactKey(...)` |

    现代路由辅助函数会在原生审批、回复抑制、入站去重、定时任务投递和会话路由中以一致方式规范化
    `{ channel, to, accountId, threadId }`。

    不要新增对 `plugin-sdk/channel-route` 中
    `ChannelMessagingAdapter.parseExplicitTarget` 或
    `resolveChannelRouteTargetWithParser(...)` 的使用——它们已弃用，仅为旧插件保留。新的渠道插件应使用
    `messaging.targetResolver.resolveTarget(...)` 进行目标 ID 规范化和目录未命中回退，
    在核心需要提前确定对端类型时使用 `messaging.inferTargetChatType(...)`，
    并使用 `messaging.resolveOutboundSessionRoute(...)` 表示提供商原生的会话和线程标识。

  </Step>

  <Step title="构建和测试">
    ```bash
    pnpm build
    pnpm test my-plugin/
    ```
  </Step>
</Steps>

## 导入路径参考

公共软件包导出映射是可导入 SDK 子路径的权威来源。请使用 [SDK 概览](/zh-CN/plugins/sdk-overview)中链接的专题 SDK 指南，并优先选择文档中最精确的公共子路径。`scripts/lib/plugin-sdk-entrypoints.json` 中的编译器清单还包含用于构建内置插件的私有本地条目；这些条目出现在其中并不意味着它们是公共软件包导出项。

此表仅列出常用迁移子集，并非完整的 SDK 接口。编译器入口点清单位于
`scripts/lib/plugin-sdk-entrypoints.json`；软件包导出项从公共子集生成。

为内置插件保留的辅助接口已从公共 SDK 导出映射中移除，但明确记录的兼容性外观除外，例如为仍直接导入已发布 `@openclaw/discord` 软件包的外部插件保留的已弃用
`plugin-sdk/discord` 垫片。所有者专用辅助函数位于其所属插件软件包内；共享主机行为通过
`plugin-sdk/gateway-runtime`、`plugin-sdk/security-runtime` 和注入的插件 API 等通用 SDK 契约提供。

请使用与任务匹配的最精确导入。如果找不到某个导出项，请检查
`src/plugin-sdk/` 中的源代码，或询问维护者应由哪个通用契约负责。

## 已移除的兼容性接口

2026 年 7 月的清理移除了根 SDK 和兼容性聚合入口、扩展 API 桥接、已过期的 SDK 子路径别名、未使用的 SDK 子路径，以及仅供内置使用的 SDK 模块的公共导出。仅供内置使用的模块仍可由其仓库所有者通过私有本地构建映射使用；无法从已发布的软件包中导入这些模块。

### 进程全局 API 提供商发布

`registerApiProvider(...)` 和 `unregisterApiProviders(...)` 已从
`openclaw/plugin-sdk/llm` 中移除。它们将 API 传输发布到进程全局状态中，之后由生命周期所有的模型运行时复制到每个已准备的注册表中。

提供商插件应通过 `api.registerProvider(...)` 注册文本推理提供商。构造
`ApiRegistry` 的主机所有代码和测试应直接在该注册表上注册，以便将提供商所有权和拆卸范围限定在已准备的运行时中。

### 私有测试聚合入口

`openclaw/plugin-sdk/testing` 仅供仓库本地使用，且未包含在已发布的软件包工件中，因此已在其 2026-07-28
`removeAfter` 日期之前移除。仓库测试使用精确子路径，例如
`plugin-sdk/plugin-test-runtime`、`plugin-sdk/channel-test-helpers`、`plugin-sdk/channel-target-testing`、
`plugin-sdk/test-env` 和 `plugin-sdk/test-fixtures`。

## 迁移参考

  这些映射涵盖已于 2026 年 7 月移除的表面，以及后续时间窗口内仍处于活跃状态的弃用项。映射是迁移指导，并不表示旧表面仍然可用；请查阅兼容性注册表和移除时间线以了解当前状态。

  <AccordionGroup>
  <Accordion title="command-auth 帮助构建器 -> command-status">
    **旧版（`openclaw/plugin-sdk/command-auth`）**：`buildCommandsMessage`、
    `buildCommandsMessagePaginated`、`buildHelpMessage`。

    **新版（`openclaw/plugin-sdk/command-status`）**：签名相同，从范围更窄的子路径导入。
    `command-auth` 兼容性再导出已被移除。

    ```typescript
    // 之前
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-auth";

    // 之后
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-status";
    ```

  </Accordion>

  <Accordion title="提及门控辅助函数 -> resolveInboundMentionDecision">
    **旧版**：来自 `openclaw/plugin-sdk/channel-inbound` 或
    `openclaw/plugin-sdk/channel-mention-gating` 的 `resolveMentionGating(params)` 和
    `resolveMentionGatingWithBypass(params)`。

    **新版**：`resolveInboundMentionDecision({ facts, policy })`——使用一个决策对象，
    而不是两个分离的调用形式。

    Discord、iMessage、Matrix、MS Teams、QQ Bot、Signal、Telegram、WhatsApp
    和 Zalo 均已采用。Slack 自身的 `app_mention` 事件模型不使用此辅助函数。

  </Accordion>

  <Accordion title="渠道运行时垫片和渠道操作辅助函数">
    `openclaw/plugin-sdk/channel-runtime` 已被移除。请使用
    `openclaw/plugin-sdk/channel-runtime-context` 注册运行时对象。

    `openclaw/plugin-sdk/channel-actions` 中的原生消息架构辅助函数已与原始的 “actions”
    渠道导出一同移除。请改为通过语义化的 `presentation` 表面公开能力——渠道插件声明其渲染的内容（卡片、按钮、选择器），而不是其接受哪些原始操作名称。

  </Accordion>

  <Accordion title="Web 搜索提供商 tool() 辅助函数 -> 插件上的 createTool()">
    **旧版**：来自 `openclaw/plugin-sdk/provider-web-search` 的 `tool()` 工厂。

    **新版**：直接在提供商插件上实现 `createTool(...)`。
    OpenClaw 不再需要通过 SDK 辅助函数注册工具包装器。

  </Accordion>

  <Accordion title="纯文本渠道信封 -> BodyForAgent">
    **旧版**：使用 `api.runtime.channel.reply.formatInboundEnvelope(...)`（以及入站消息对象上的
    `channelEnvelope` 字段），根据入站渠道消息构建扁平的纯文本提示词信封。

    **新版**：使用 `BodyForAgent` 加结构化用户上下文块。渠道插件将路由元数据（线程、主题、回复目标、表情回应）作为类型化字段附加，而不是将其拼接到提示词字符串中。`formatAgentEnvelope(...)` 辅助函数仍支持用于合成面向助手的信封，但入站纯文本信封正在逐步淘汰。

    受影响的区域：`inbound_claim`、`message_received`，以及任何对旧信封文本进行后处理的自定义渠道插件。

  </Accordion>

  <Accordion title="deactivate 钩子 -> gateway_stop">
    **旧版**：`api.on("deactivate", handler)`。

    **新版**：`api.on("gateway_stop", handler)`。关闭清理契约相同；仅钩子名称发生变化。

    ```typescript
    // 之前
    api.on("deactivate", async (event, ctx) => {
      await stopPluginService(ctx);
    });

    // 之后
    api.on("gateway_stop", async (event, ctx) => {
      await stopPluginService(ctx);
    });
    ```

    `deactivate` 仍作为已弃用的兼容性别名保持连接，直至在 2026-08-16 之后移除。

  </Accordion>

  <Accordion title="subagent_spawning 钩子 -> 核心线程绑定">
    **旧版**：`api.on("subagent_spawning", handler)` 返回
    `threadBindingReady` 或 `deliveryOrigin`。

    **新版**：让核心通过渠道会话绑定适配器准备 `thread: true` 子智能体绑定。
    仅将 `api.on("subagent_spawned", handler)` 用于启动后的观察。

    ```typescript
    // 之前
    api.on("subagent_spawning", async () => ({
      status: "ok",
      threadBindingReady: true,
      deliveryOrigin: { channel: "discord", to: "channel:123", threadId: "456" },
    }));

    // 之后
    api.on("subagent_spawned", async (event) => {
      await observeSubagentLaunch(event);
    });
    ```

    在外部插件迁移期间，`subagent_spawning`、`PluginHookSubagentSpawningEvent`、
    `PluginHookSubagentSpawningResult` 和 `SubagentLifecycleHookRunner.runSubagentSpawning(...)`
    仅作为已弃用的兼容性表面保留，并将在 2026-08-30 之后移除。

  </Accordion>

  <Accordion title="提供商发现类型 -> 提供商目录类型">
    四个发现类型别名现在是目录时代类型的轻量包装器：

    | 旧别名                    | 新类型                    |
    | ------------------------- | ------------------------- |
    | `ProviderDiscoveryOrder`  | `ProviderCatalogOrder`    |
    | `ProviderDiscoveryContext`| `ProviderCatalogContext`  |
    | `ProviderDiscoveryResult` | `ProviderCatalogResult`   |
    | `ProviderPluginDiscovery` | `ProviderPluginCatalog`   |

    这些别名和旧版 `ProviderCapabilities` 静态包已被移除。提供商插件应使用显式提供商钩子，例如 `buildReplayPolicy`、`normalizeToolSchemas` 和 `wrapStreamFn`，而不是静态对象。

  </Accordion>

  <Accordion title="思考策略钩子 -> resolveThinkingProfile">
    **旧版**（`ProviderThinkingPolicy` 上的三个独立钩子）：
    `isBinaryThinking(ctx)`、`supportsXHighThinking(ctx)` 和
    `resolveDefaultThinkingLevel(ctx)`。

    **新版**：单个 `resolveThinkingProfile(ctx)`，返回一个
    `ProviderThinkingProfile`，其中包含规范的 `id`、可选的 `label` 和按等级排序的级别列表。OpenClaw 会自动按照配置档案等级降级过时的已存储值。

    上下文包含 `provider`、`modelId`、可选的合并后 `reasoning`，以及可选的合并后模型 `compat` 事实。仅当已配置的请求契约支持时，提供商插件才能使用这些目录事实公开特定于模型的配置档案。

    只需实现一个钩子，而不是三个。旧版钩子已被移除。

  </Accordion>

  <Accordion title="外部身份验证提供商 -> contracts.externalAuthProviders">
    **旧版**：实现外部身份验证钩子，但未在插件清单中声明提供商。

    **新版**：在插件清单中声明 `contracts.externalAuthProviders`，**并且**实现 `resolveExternalAuthProfiles(...)`。

    ```json
    {
      "contracts": {
        "externalAuthProviders": ["anthropic", "openai"]
      }
    }
    ```

  </Accordion>

  <Accordion title="提供商环境变量查找 -> setup.providers[].envVars">
    **旧版**清单字段：`providerAuthEnvVars: { anthropic: ["ANTHROPIC_API_KEY"] }`。

    **新版**：将同一环境变量查找镜像到清单上的 `setup.providers[].envVars`。
    这会将设置/状态环境元数据整合到一个位置，并避免仅为响应环境变量查找而启动插件运行时。

    不再接受 `providerAuthEnvVars`。

  </Accordion>

  <Accordion title="记忆插件注册 -> registerMemoryCapability">
    **旧版**：三个独立调用——`api.registerMemoryPromptSection(...)`、
    `api.registerMemoryFlushPlan(...)`、`api.registerMemoryRuntime(...)`。

    **新版**：记忆状态 API 上的一个调用——
    `registerMemoryCapability(pluginId, { promptBuilder, flushPlanResolver, runtime })`。

    插槽相同，使用单次注册调用。增量式提示词和语料库辅助函数
    （`registerMemoryPromptSupplement`、`registerMemoryCorpusSupplement`）不受影响。

  </Accordion>

  <Accordion title="记忆嵌入提供商 API">
    **旧版**：`api.registerMemoryEmbeddingProvider(...)` 加
    `contracts.memoryEmbeddingProviders`。

    **新版**：`api.registerEmbeddingProvider(...)` 加
    `contracts.embeddingProviders`。

    通用嵌入提供商契约可在记忆功能之外复用，并且是新提供商支持的路径。现有提供商迁移期间，记忆专用注册 API 仍作为已弃用的兼容性表面保持连接。插件检查会将非内置用法报告为兼容性债务。

  </Accordion>

  <Accordion title="原始渠道发送结果 -> OutboundDeliveryResult">
    **旧版**：通过 `ChannelSendRawResult` 返回 `{ ok, messageId, error }`，
    并使用 `createRawChannelSendResultAdapter(...)` 对其进行规范化。

    **新版**：返回 `OutboundDeliveryResult` 字段，并使用
    `createAttachedChannelResultAdapter(...)` 附加渠道。发送失败时应抛出异常，而不是返回错误字符串。原始结果类型会继续保留，直至下一个插件 SDK 主版本发布。

  </Accordion>

  <Accordion title="子智能体会话消息类型已重命名">
    `src/plugins/runtime/types.ts` 仍导出两个旧版类型别名：

    | 旧版                          | 新版                            |
    | ----------------------------- | ------------------------------- |
    | `SubagentReadSessionParams`   | `SubagentGetSessionMessagesParams` |
    | `SubagentReadSessionResult`   | `SubagentGetSessionMessagesResult` |

    运行时方法 `readSession` 已弃用，建议改用
    `getSessionMessages`。签名相同；旧方法会转调新方法。

  </Accordion>

  <Accordion title="已移除的会话和转录文件 API">
    切换到 SQLite 会话/转录后，将移除或弃用那些向插件公开活跃 `sessions.json` 存储、JSONL 转录路径或会话文件列表的 API。运行时插件应使用会话身份和 SDK 运行时辅助函数，而不是解析或修改活跃文件。

    | 迁移表面 | 替代方案 |
    | ----------------- | ----------- |
    | 已弃用的 `loadSessionStore(...)`、`updateSessionStore(...)` 和 `resolveSessionStoreEntry(...)` | `getSessionEntry(...)`、`listSessionEntries(...)` 和行级会话修改。 |
    | 已弃用的 `resolveSessionFilePath(...)` | 会话身份（`sessionKey`、`sessionId` 和 SDK 运行时目标辅助函数），以及操作当前会话的 Gateway 网关方法。 |
    | 已移除的 `saveSessionStore(...)` | Gateway 网关拥有的会话运行时 API；插件代码应通过已记录的运行时/上下文辅助函数请求或修改会话状态，而不是写入活跃存储文件。 |
    | 已移除的 `resolveSessionTranscriptPathInDir(...)` 和 `resolveAndPersistSessionFile(...)` | 会话身份和操作当前会话的 Gateway 网关方法。 |
    | `readLatestAssistantTextFromSessionTranscript(...)` | 当前运行时上下文公开的基于身份的转录读取器；当插件位于转录所有者路径之外时，则使用 Gateway 网关历史记录/会话方法。 |
    | `SessionTranscriptUpdate.sessionFile` | 带有 `agentId`、`sessionKey` 和 `sessionId` 的 `SessionTranscriptUpdate.target`。 |
    | 记忆同步输入，例如 `sessionFiles` | 主机提供的基于身份的转录/会话源；不要为实时会话遍历活跃 JSONL 文件。 |
    | 活跃会话中名为 `transcriptPath` 或 `sessionFile` 的运行时选项 | 携带存储中立会话身份的 `sessionTarget`/运行时目标对象。 |

    旧版 JSONL 转录文件作为导入、归档、导出和支持工件仍然有效。它们不再是活跃会话的稳态运行时契约。

    使用 `v2026.7.1-beta.5` 发布的官方插件导入了上述四个已弃用辅助函数。`openclaw/plugin-sdk/session-store-runtime` 会将这一精确桥接保留至 2026-10-12；新插件必须使用替代方案。
    `resolveStorePath(...)` 仍是受支持的 SDK 辅助函数，不属于此次弃用范围。

    `openclaw plugins inspect --all --runtime` 会报告加载错误或诊断信息仍引用这些已移除文件 API 的非内置插件。`@openclaw/plugin-inspector` 咨询扫描必须使用 `0.3.17` 或更高版本，以便外部软件包扫描也能在发布前标记整个存储会话辅助函数、会话文件路径辅助函数、旧版转录文件目标和低级转录辅助函数。

  </Accordion>

  <Accordion title="runtime.tasks.flow -> runtime.tasks.managedFlows">
    **旧版**：`runtime.tasks.flow`（单数）返回实时任务流访问器。

    **新版**：`runtime.tasks.managedFlows` 为从流程中创建、更新、取消或运行子任务的插件保留托管 TaskFlow 修改运行时。当插件仅需要基于 DTO 的读取时，请使用 `runtime.tasks.flows`。

    ```typescript
    // 之前
    const flow = api.runtime.tasks.flow.fromToolContext(ctx);
    // 之后
    const flow = api.runtime.tasks.managedFlows.fromToolContext(ctx);
    ```

    旧版别名已于 2026 年 7 月移除。

  </Accordion>

  <Accordion title="嵌入式扩展工厂 -> Agent 工具结果中间件">
    上文的[如何迁移](#how-to-migrate)中已介绍。为完整起见，这里再次说明：
    已移除的仅限嵌入式运行器的 `api.registerEmbeddedExtensionFactory(...)` 路径由
    `api.registerAgentToolResultMiddleware(...)` 替代，并在
    `contracts.agentToolResultMiddleware` 中提供显式运行时列表。
  </Accordion>

  <Accordion title="OpenClawSchemaType 别名 -> OpenClawConfig">
    `OpenClawSchemaType` 根 SDK 别名已移除。请使用规范名称
    `OpenClawConfig`。

    ```typescript
    // 之前
    import type { OpenClawSchemaType } from "openclaw/plugin-sdk";
    // 之后
    import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
    ```

  </Accordion>
</AccordionGroup>

<Note>
扩展级弃用项（位于 `extensions/` 下的内置渠道/提供商插件中）
会在各自的 `api.ts` 和 `runtime-api.ts` 导出入口中跟踪。
它们不会影响第三方插件契约，因此未在此列出。如果你直接使用内置插件的本地导出入口，
请在升级前阅读该导出入口中的弃用注释。
</Note>

## Talk 和实时语音迁移

实时语音、电话、会议和浏览器 Talk 代码共享一个由 `openclaw/plugin-sdk/realtime-voice`
导出的 Talk 会话控制器。该控制器负责通用 Talk 事件封装、活动轮次状态、采集状态、
音频输出状态、最近事件历史记录和过期轮次拒绝。提供商插件负责供应商特定的实时会话。
浏览器会议插件使用 `openclaw/plugin-sdk/meeting-runtime` 处理会话、浏览器、音频、节点主机、
Agent 咨询和语音通话机制，然后实现 `MeetingPlatformAdapter`，
以处理 URL 规则、DOM 脚本、手动操作映射、字幕、创建和拨入计划。
平台 REST API、OAuth、工件、选择器和传输字段名称仍保留在插件中。
浏览器权限计划会接收请求的会议 URL，以便每个平台仅授予其确切支持的来源。
会话运行时还必须在确认浏览器离开后规范化平台特定的实时健康状态；
历史转录字段可以保留，但离开后字幕和音频就绪状态不得继续处于活动状态。

所有内置界面都在共享控制器上运行：浏览器中继、托管房间交接、
语音通话实时会话、语音通话流式 STT、Google Meet 实时会话以及原生按键通话。
Gateway 网关在 `hello-ok.features.events` 中公布一个实时 Talk 事件渠道：
`talk.event`。

除非实现低级适配器或测试夹具，否则新代码不应直接调用
`createTalkEventSequencer(...)`。请使用共享控制器，以确保无法在没有轮次 ID 的情况下发出
轮次范围事件，过期的 `turnEnd` / `turnCancel` 调用无法清除
较新的活动轮次，并确保音频输出生命周期事件在电话、会议、浏览器中继、
托管房间交接和原生 Talk 客户端之间保持一致。

公共 API 结构：

```typescript
// Gateway 网关负责的 Talk 会话 API。
await gateway.request("talk.session.create", {
  mode: "realtime",
  transport: "gateway-relay",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.session.appendAudio", { sessionId, audioBase64 });
await gateway.request("talk.session.cancelOutput", { sessionId, reason: "barge-in" });
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "working" },
  options: { willContinue: true },
});
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "already_delivered" },
  options: { suppressResponse: true },
});
await gateway.request("talk.session.submitToolResult", { sessionId, callId, result });
await gateway.request("talk.session.close", { sessionId });

// 客户端负责的提供商会话 API。
await gateway.request("talk.client.create", {
  mode: "realtime",
  transport: "webrtc",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.client.toolCall", { sessionKey, callId, name, args });
await gateway.request("talk.client.steer", { sessionKey, text, mode: "steer" });
```

浏览器负责的 WebRTC/提供商 WebSocket 会话使用 `talk.client.create`，
因为浏览器负责提供商协商和媒体传输，而 Gateway 网关负责凭据、指令和工具策略。
`talk.session.*` 是 Gateway 网关管理的通用界面，用于 Gateway 网关中继实时会话、
Gateway 网关中继转录以及托管房间原生 STT/TTS 会话。

对于将实时选择器放在 `talk.provider` / `talk.providers`
旁边的旧版配置，应使用 `openclaw doctor --fix` 修复；Talk 运行时不会将语音/TTS
提供商配置重新解释为实时提供商配置。

受支持的 `talk.session.create` 组合被有意限制在较小范围内：

| 模式            | 传输方式       | 核心           | 负责方              | 说明                                                                                                              |
| --------------- | --------------- | --------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `realtime`      | `gateway-relay` | `agent-consult` | Gateway 网关            | 通过 Gateway 网关桥接全双工提供商音频；工具调用通过 Agent 咨询工具路由。           |
| `transcription` | `gateway-relay` | `none`          | Gateway 网关            | 仅限流式 STT；调用方发送输入音频并接收转录事件。                                        |
| `stt-tts`       | `managed-room`  | `agent-consult` | 原生/客户端房间 | 采用按键通话和对讲机模式的房间，其中客户端负责采集/播放，Gateway 网关负责轮次状态。 |
| `stt-tts`       | `managed-room`  | `direct-tools`  | 原生/客户端房间 | 仅限管理员的房间模式，适用于直接执行 Gateway 网关工具操作的可信第一方界面。                  |

供从旧版 `talk.realtime.*` / `talk.transcription.*` /
`talk.handoff.*` 系列迁移的读者参考的方法映射（均已移除）：

| 旧方法                              | 新方法                                                      |
| -------------------------------- | -------------------------------------------------------- |
| `talk.realtime.session`          | `talk.client.create`                                     |
| `talk.realtime.toolCall`         | `talk.client.toolCall`                                   |
| `talk.realtime.relayAudio`       | `talk.session.appendAudio`                               |
| `talk.realtime.relayCancel`      | `talk.session.cancelOutput` 或 `talk.session.cancelTurn` |
| `talk.realtime.relayToolResult`  | `talk.session.submitToolResult`                          |
| `talk.realtime.relayStop`        | `talk.session.close`                                     |
| `talk.transcription.session`     | `talk.session.create({ mode: "transcription" })`         |
| `talk.transcription.relayAudio`  | `talk.session.appendAudio`                               |
| `talk.transcription.relayCancel` | `talk.session.cancelTurn`                                |
| `talk.transcription.relayStop`   | `talk.session.close`                                     |
| `talk.handoff.create`            | `talk.session.create({ transport: "managed-room" })`     |
| `talk.handoff.join`              | `talk.session.join`                                      |
| `talk.handoff.revoke`            | `talk.session.close`                                     |

统一控制词汇也被刻意限制在较小范围内：

| 方法                          | 适用范围                                              | 契约                                                                                                                                                                                                                  |
| ------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `talk.session.appendAudio`      | `realtime/gateway-relay`、`transcription/gateway-relay` | 将 base64 PCM 音频块追加到由同一 Gateway 网关连接负责的提供商会话。                                                                                                                             |
| `talk.session.startTurn`        | `stt-tts/managed-room`                                  | 启动托管房间用户轮次。                                                                                                                                                                                           |
| `talk.session.endTurn`          | `stt-tts/managed-room`                                  | 在完成过期轮次验证后结束活动轮次。                                                                                                                                                                          |
| `talk.session.cancelTurn`       | 所有由 Gateway 网关负责的会话                              | 取消某个轮次中正在进行的采集/提供商/Agent/TTS 工作。                                                                                                                                                                 |
| `talk.session.cancelOutput`     | `realtime/gateway-relay`                                | 停止助手音频输出，但不一定结束用户轮次。                                                                                                                                                     |
| `talk.session.submitToolResult` | `realtime/gateway-relay`                                | 在其桥接层公开的任何异步完成操作结束后完成提供商工具调用；为中间输出传入 `options.willContinue`，或在受支持时传入 `options.suppressResponse`，以避免再次生成助手响应。 |
| `talk.session.steer`            | 由 Agent 支持的 Talk 会话                              | 将口述的 `status`、`steer`、`cancel` 或 `followup` 控制发送到根据 Talk 会话解析出的活动嵌入式运行。                                                                                                 |
| `talk.session.close`            | 所有统一会话                                    | 停止中继会话或撤销托管房间状态，然后忘记统一会话 ID。                                                                                                                                     |

不要在核心中引入提供商或平台特例来实现此功能。
核心负责 Talk 会话语义。提供商插件负责供应商会话设置。
语音通话和 Google Meet 负责电话/会议适配器。浏览器和原生应用
负责设备采集/播放用户体验。

## 移除时间线

| 时间                                        | 发生的情况                                                                                                                              |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **现在**                                     | 可发出警告的已弃用接口会生成运行时警告；仓库防护规则会拒绝核心和内置插件从已弃用的 SDK 路径导入。 |
| **等待所有者决定**                  | 无日期的记录会保持弃用状态，且不能移除，直到其所有者发布 `removeAfter` 日期。                          |
| **每条兼容性记录的 `removeAfter` 日期** | 对应的特定接口从此可移除；日期过后，`pnpm plugins:boundary-report --fail-on-eligible-compat` 会使 CI 失败。    |
| **下一个主要版本**                      | 有日期的接口只能在其 `removeAfter` 日期之后移除；无日期的记录仍需所有者批准并发布日期。   |

以下剩余的公共 SDK 子路径均有由注册表支持的移除窗口。
7 月 30 日对应的条目已在维护者提前授权的清理后移除：
未使用的子路径已删除，早期的兼容性别名已删除，并且
仅供内置组件使用的模块已降级为私有本地构建映射。

| `removeAfter` | 层级                               | SDK 子路径                                                                                                                                                                        |
| ------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `2026-08-15`  | 早期兼容性弃用项 | `agent-config-primitives`、`channel-logging`、`channel-secret-runtime`、`channel-streaming`、`group-access`、`inbound-reply-dispatch`、`matrix`、`text-runtime`、`zod`              |
| `2026-09-01`  | 早期兼容性弃用项 | `channel-lifecycle`、`channel-message`、`channel-reply-pipeline`、`config-runtime`、`infra-runtime`                                                                                 |
| `2026-10-01`  | 媒体旧版投影            | `agent-media-payload`，以及非子路径的 `MsgContext Media*` 字段、渠道入站媒体有效载荷构建器、`buildMediaPayload`、钩子媒体别名和 `{{Media*}}` 模板 |

所有核心插件都已完成迁移。外部插件应在
下一个主要版本发布前迁移。运行 `pnpm plugins:boundary-report`，查看插件所用接口中
哪些兼容性记录最早到期。

## 暂时禁止显示警告

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

这只是临时的应急手段，并非永久解决方案。

## 相关内容

- [入门指南](/zh-CN/plugins/building-plugins) - 构建你的第一个插件
- [SDK 概览](/zh-CN/plugins/sdk-overview) - 完整的子路径导入参考
- [渠道插件](/zh-CN/plugins/sdk-channel-plugins) - 构建渠道插件
- [提供商插件](/zh-CN/plugins/sdk-provider-plugins) - 构建提供商插件
- [插件内部机制](/zh-CN/plugins/architecture) - 深入解析架构
- [Plugin Manifest](/zh-CN/plugins/manifest) - 清单架构参考
