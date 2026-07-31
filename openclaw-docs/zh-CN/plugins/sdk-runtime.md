---
read_when:
    - 你需要从插件调用核心辅助功能（TTS、STT、图像生成、Web 搜索、Gateway 网关、子智能体、节点）
    - 你想了解 `api.runtime` 公开了哪些内容
    - 你正在从插件代码访问配置、智能体或媒体辅助函数
sidebarTitle: Runtime helpers
summary: api.runtime -- 可供插件使用的注入式运行时辅助函数
title: 插件运行时辅助函数
x-i18n:
    generated_at: "2026-07-26T06:23:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ff1d901de8ec70011eeaafbab7b3cc30709fc95894c7ba4f4346c026de682cd0
    source_path: plugins/sdk-runtime.md
    workflow: 16
---

注入到每个插件注册过程中的 `api.runtime` 对象参考。请使用这些辅助函数，而不要直接导入宿主内部模块。

<CardGroup cols={2}>
  <Card title="渠道插件" href="/zh-CN/plugins/sdk-channel-plugins">
    在渠道插件的上下文中使用这些辅助函数的分步指南。
  </Card>
  <Card title="提供商插件" href="/zh-CN/plugins/sdk-provider-plugins">
    在提供商插件的上下文中使用这些辅助函数的分步指南。
  </Card>
</CardGroup>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

`api.runtime.version` 是当前的 OpenClaw 产品版本，来源于共享版本解析器，因此插件看到的值与 CLI 报告的值相同。

## 配置加载和写入

优先使用已传入当前调用路径的配置，例如注册期间的 `api.config`，或渠道/提供商回调中的 `cfg` 参数。这样可让同一个进程快照贯穿整个工作流程，而不是在热路径上重新解析配置。

仅当长生命周期处理程序需要当前进程快照，且该函数未收到配置时，才使用 `api.runtime.config.current()`。返回值为只读；编辑前请克隆它或使用变更辅助函数。

工具工厂会接收 `ctx.runtimeConfig` 和 `ctx.getRuntimeConfig()`。如果配置可能在工具定义创建后发生变化，请在长生命周期工具的 `execute` 回调中使用该 getter。

使用 `api.runtime.config.mutateConfigFile(...)` 或 `api.runtime.config.replaceConfigFile(...)` 持久化更改。每次写入都必须选择明确的 `afterWrite` 策略：

- `afterWrite: { mode: "auto" }` 让 Gateway 网关的重载规划器决定。
- `afterWrite: { mode: "restart", reason: "..." }` 在写入方确定热重载不安全时强制执行干净重启。
- `afterWrite: { mode: "none", reason: "..." }` 仅当调用方负责后续处理时，禁止自动重载/重启。

变更辅助函数会返回 `afterWrite` 以及类型化的 `followUp` 摘要，使调用方能够记录或测试是否请求了重启。实际何时重启仍由 Gateway 网关负责。

使用 `current()`、传入的 `cfg`、`mutateConfigFile(...)` 或
`replaceConfigFile(...)` 访问和写入运行时配置。

对于直接 SDK 导入，优先使用聚焦的配置子路径，而不是宽泛的 `openclaw/plugin-sdk/config-runtime` 兼容性 barrel：使用 `config-contracts` 获取类型，使用 `runtime-config-snapshot` 获取当前进程快照，使用 `config-mutation` 执行写入。从 `api.pluginConfig` 读取条目作用域的值；提供的工具上下文仅用于获取其运行时范围的配置快照，并将插件特定的合并保留在该边界。内置插件测试应直接模拟这些聚焦子路径，而不是模拟宽泛的兼容性 barrel。

OpenClaw 内部运行时代码遵循同一方向：在 CLI、Gateway 网关或进程边界加载一次配置，然后将该值继续传递。成功的变更写入会刷新进程运行时快照并推进其内部修订版本；长生命周期缓存应以运行时拥有的缓存键作为索引，而不是在本地序列化配置。长生命周期运行时模块对环境中的 `loadConfig()` 调用实施零容忍扫描；请使用传入的 `cfg`、请求的 `context.getRuntimeConfig()`，或在明确的进程边界使用 `getRuntimeConfig()`。

提供商和渠道执行路径必须使用当前运行时配置快照，而不是用于配置回读或编辑的文件快照。文件快照会保留 SecretRef 标记等源值，以供 UI 和写入使用；提供商回调需要的是解析后的运行时视图。当辅助函数可能接收到当前源快照或当前运行时快照中的任一种时，请先通过 `selectApplicableRuntimeConfig()` 路由，再读取凭据。

## 可复用的运行时实用工具

对于由机器人发出的入站消息，请使用入站 `botLoopProtection` 事实。核心会在会话记录和分发之前应用共享的内存滑动窗口防护，而不会将策略绑定到某个渠道。该防护会跟踪 `(scopeId, conversationId, participant pair)` 键，将一对通信双方的两个方向合并计数，在超过窗口预算后应用冷却时间，并择机清理不活跃条目。

向操作员公开此行为的渠道插件应优先使用共享的 `channels.defaults.botLoopProtection` 结构作为基准预算，然后在其上叠加渠道/提供商特定的覆盖。共享配置使用秒作为单位，因为它面向用户：

```typescript
type ChannelBotLoopProtectionConfig = {
  enabled?: boolean;
  maxEventsPerWindow?: number;
  windowSeconds?: number;
  cooldownSeconds?: number;
};
```

将规范化的机器人通信对事实与解析后的轮次一起传入。核心负责解析默认值、单位转换和 `enabled` 语义：

```typescript
return {
  channel: "example",
  routeSessionKey,
  storePath,
  ctxPayload,
  recordInboundSession,
  runDispatch,
  botLoopProtection: {
    scopeId: "account-1",
    conversationId: "channel-1",
    senderId: "bot-a",
    receiverId: "bot-b",
    config: channelConfig.botLoopProtection,
    defaultsConfig: runtimeConfig.channels?.defaults?.botLoopProtection,
    defaultEnabled: allowBotsMode !== "off",
  },
};
```

仅对于不经过共享入站回复运行器的自定义
双参与方事件循环，才直接使用 `openclaw/plugin-sdk/pair-loop-guard-runtime`。

## 运行时命名空间

<AccordionGroup>
  <Accordion title="api.runtime.agent">
    Agent 身份、目录和会话管理。

    ```typescript
    // 解析 Agent 的工作目录（必须提供 agentId）
    const agentDir = api.runtime.agent.resolveAgentDir(cfg, agentId);

    // 解析 Agent 工作区
    const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId);

    // 获取 Agent 身份
    const identity = api.runtime.agent.resolveAgentIdentity(cfg);

    // 获取默认思考级别
    const thinking = api.runtime.agent.resolveThinkingDefault({
      cfg,
      provider,
      model,
    });

    // 根据当前提供商配置验证用户提供的思考级别
    const policy = api.runtime.agent.resolveThinkingPolicy({ provider, model });
    const level = api.runtime.agent.normalizeThinkingLevel("extra high");
    if (level && policy.levels.some((entry) => entry.id === level)) {
      // 将级别传给嵌入式运行
    }

    // 获取 Agent 超时时间
    const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

    // 确保工作区存在
    await api.runtime.agent.ensureAgentWorkspace(cfg);

    // 运行嵌入式 Agent 轮次
    const result = await api.runtime.agent.runEmbeddedAgent({
      sessionId: "my-plugin:task-1",
      runId: crypto.randomUUID(),
      workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId),
      prompt: "总结最新更改",
      timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
    });
    ```

    `runEmbeddedAgent(...)` 是从插件代码启动常规 OpenClaw Agent 轮次的中性辅助函数。它使用与渠道触发的回复相同的提供商/模型解析和 Agent harness 选择机制。

    `runEmbeddedPiAgent(...)` 仍作为现有插件的已弃用兼容性别名保留。新代码应使用 `runEmbeddedAgent(...)`。

    `resolveCliBackendDispatchEligibility({ provider, model, agentId, authProfileId, config, agentDir, workspaceDir })` 与选择加入 `cliBackendDispatch: "subscription-auth"` 的嵌入式运行调用方共享嵌入式运行器的 CLI 后端分发决策（路由、后端声明的 `subscriptionAuthDispatch` 能力、已存储的凭据模式——遵循显式固定的 `authProfileId`）。当运行将通过 CLI 后端执行时，它返回 `{ provider }`；当运行继续采用直接透传时，它返回 `undefined`，从而让调用方能够为实际执行的运行分配超时预算。

    `resolveThinkingPolicy(...)` 返回提供商/模型支持的思考级别以及可选的默认值。提供商插件通过其思考钩子拥有模型特定的配置，因此工具插件应调用此运行时辅助函数，而不是导入或复制提供商列表。

    `normalizeThinkingLevel(...)` 会先将 `on`、`x-high` 或 `extra high` 等用户文本转换为规范的存储级别，再根据解析后的策略进行检查。

    **会话存储辅助函数**位于 `api.runtime.agent.session` 下：

    ```typescript
    const entry = api.runtime.agent.session.getSessionEntry({ agentId, sessionKey });
    for (const { sessionKey, entry } of api.runtime.agent.session.listSessionEntries({ agentId })) {
      // 遍历会话行，而不依赖旧版 sessions.json 结构。
    }
    await api.runtime.agent.session.patchSessionEntry({
      agentId,
      sessionKey,
      update: (entry) => ({ thinkingLevel: "high" }),
    });

    const created = await api.runtime.agent.session.createSessionEntry({
      cfg,
      key: "agent:main:my-plugin:task-1",
      initialEntry: {
        agentHarnessId: "my-harness",
        modelSelectionLocked: true,
        pluginExtensions: { "my-plugin": { phase: "initializing" } },
      },
      afterCreate: async () => ({
        pluginExtensions: { "my-plugin": { phase: "ready" } },
      }),
    });

    const storePath = api.runtime.agent.session.resolveStorePath(cfg.session?.store, { agentId });
    await api.runtime.agent.session.runWithWorkAdmission(
      { storePath, sessionKey },
      async (signal) => {
        // 创建或更新会话，然后将 signal 传给已准入的 Agent 运行。
      },
    );
    ```

    会话工作流优先使用 `getSessionEntry(...)`、`listSessionEntries(...)`、`patchSessionEntry(...)` 或 `upsertSessionEntry(...)`。这些辅助函数通过 Agent/会话身份定位会话，使插件无需依赖旧版 `sessions.json` 存储结构。对于不应刷新会话活动时间的仅元数据补丁，请使用 `preserveActivity: true`；仅当回调返回完整条目且已删除字段必须保持删除状态时，才使用 `replaceEntry: true`。Doctor 和迁移路径可以组合使用 `fallbackEntry`、`skipMaintenance` 和 `requireWriteSuccess`，以执行一次原子化的规范存储修复。

    `createSessionEntry(...)` 会创建新的规范会话行和转录记录。其受信任的 `initialEntry` 接口被刻意限制为：非空的 `agentHarnessId`、可选的 `modelSelectionLocked: true` 和可选的 `pluginExtensions`。注入的运行时仅接受由调用插件通过 `registerAgentHarness(...)` 拥有的 harness ID；这是所有权不变量，而不是进程内插件之间的沙箱。它会拒绝已存在的行；`label` 和 `spawnedCwd` 是独立的创建字段，而不是受信任的条目补丁。

    创建过程通过 `afterCreate` 持有会话生命周期变更栅栏，因此新工作会等待插件拥有的初始化完成，而已有的已准入工作会导致创建失败。回调接收已创建状态的克隆。如果它返回补丁，则该补丁只能包含 `pluginExtensions`，且其值是完整的最终 `pluginExtensions` 字段。回调或最终持久化失败会回滚未发生变化的新行和转录记录；受保护的回滚会保留被并发更改或认领的行。`recoverMatchingInitialEntry: true` 仅用于在持久化的受信任字段完全匹配时重试中断的初始化，并且恢复要求 `afterCreate` 返回最终补丁。

    当插件开始处理持久化会话时，请使用 `runWithWorkAdmission(...)`。该回调会拒绝已归档或被并发替换的会话，使归档/重置/删除变更在完成前保持协调，并接收必须转发给 Agent 运行的 `AbortSignal`。harness 可以通过其实验性的 `delegatedExecutionPluginIds` 注册字段显式指定受信任的执行委托方。委托方只能准入和运行完全匹配且已锁定模型的现有会话；所有会话变更仍仅限 harness 所有者执行。请参阅 [Agent harness plugins](/zh-CN/plugins/sdk-agent-harness#delegated-execution)。

    维护和修复插件可以将 `deleteSessionEntry(...)` 用于一个限定范围的会话条目，将 `cleanupSessionLifecycleArtifacts(...)` 用于由生命周期管理的临时会话，并在变更存储之前使用 `resolveSessionStoreBackupPaths(...)`。当删除操作不得与并发会话更新发生竞争时，请传入 `expectedSessionId` 和 `expectedUpdatedAt`；当较早的快照没有会话 ID 时，请使用 `expectedSessionId: null`。这些辅助函数是范围有限的修复/生命周期接口，并非通用的存储删除 API。

    `resolveStorePath(...)` 和 `updateSessionStoreEntry(...)` 完善了会话辅助函数：`resolveStorePath` 解析给定作用域的会话存储路径，而当调用方已知该路径时，`updateSessionStoreEntry({ storePath, sessionKey, update })` 会按存储路径直接修补一个条目。

    `loadTranscriptEventsSync(...)` 可用于无法使用异步转录运行时的同步 Doctor 和修复路径。它返回原始 `SessionStoreTranscriptEvent` 记录。常规插件运行时代码应优先使用 `openclaw/plugin-sdk/session-transcript-runtime`。

    `formatSqliteSessionFileMarker(...)`、`parseSqliteSessionFileMarker(...)` 和 `sqliteSessionFileMarkerMatchesSession(...)` 是过渡期辅助函数，供仍接收名为 `sessionFile` 的旧字段的代码使用。解析后的 SQLite 标记用于标识实时 SQLite 转录目标，而不是文件系统路径。新 API 应携带类型化的会话身份，而不是标记字符串。

    对于转录读取和写入，请导入 `openclaw/plugin-sdk/session-transcript-runtime`，并将 `resolveSessionTranscriptIdentity(...)`、`resolveSessionTranscriptTarget(...)`、`readSessionTranscriptEvents(...)`、`readSessionTranscriptRawDelta(...)`、`readSessionTranscriptVisibleMessageDelta(...)`、`readVisibleSessionTranscriptMessageEntries(...)`、`appendSessionTranscriptMessageByIdentity(...)`、`publishSessionTranscriptUpdateByIdentity(...)` 或 `withSessionTranscriptWriteLock(...)` 与 `{ agentId, sessionKey, sessionId }` 配合使用。这些 API 允许插件标识转录、读取原始事件或分支安全的可见消息条目、追加消息、发布更新，以及在同一个转录写锁下执行相关操作，而无需依赖活动转录文件路径。`readVisibleSessionTranscriptMessageEntries(...)` 返回有序的读取元数据；其 `seq` 字段不是可恢复的游标。

    `appendSessionTranscriptMessageByIdentity(...)` 是对已规范化消息进行追加的底层接口。插件不得使用顶层 `MediaPath`、`MediaPaths`、`MediaUrl`、`MediaUrls`、`MediaType` 或 `MediaTypes` 构造包含媒体的用户行。渠道入口应通过 `MsgContext.media` 传递有序事实，并由宿主负责持久化用户轮次。由宿主准备并持久化的用户消息会在 `message.__openclaw.media` 下携带规范化的有序事实；通用追加 API 不会推断或修复旧版并行数组。

    `readSessionTranscriptRawDelta(...)` 返回有界的 `page`、`reset` 或 `missing` 结果。请将不透明的 `page.cursor` 传入下一次调用。纯追加操作会保留游标，而替换转录会返回带有新引导游标的 `reset`。页面默认最多包含 1,000 个事件和 1,000,000 个序列化字节；调用方最多可请求 10,000 个事件和 64 MiB。当仅下一个事件就超过 `maxBytes` 时，页面为空并报告 `requiredBytes`；如果所需字节限制不超过 64 MiB，请至少使用该限制重试。更大的单个事件需要使用完整读取 API。游标仅标识位置，绝不会授予对其他会话的访问权限。

    `readSessionTranscriptVisibleMessageDelta(...)` 针对由宿主管理的活动消息投影提供相同的有界引导和恢复形式。它按从最旧到最新的顺序返回消息，因此上下文引擎可以依次读取初始历史记录，并将不透明游标持久化为水位标记。请原样存储并返回游标；它是续接提示，而不是授权凭据。线性追加会从最后返回的消息之后恢复。转录替换、锚点已离开活动分支或在其中移动的游标、格式错误的游标以及跨会话游标，都会返回带有新引导游标的 `reset`。计数和字节的默认值及上限与原始增量 API 相同。当分支变更后活动投影正在重建时，结果为 `unavailable`，原因为 `projection_rebuilding`；请稍后重试，而不要回退到活动转录文件。

    插件 SDK 不再导出旧版的全存储和活动转录文件辅助函数。会话元数据请使用限定范围的条目辅助函数，活动转录操作请使用转录身份辅助函数。需要文件工件的归档/支持工作流应使用其专用归档接口，而不是活动会话运行时 API。

  </Accordion>
  <Accordion title="api.runtime.agent.defaults">
    默认模型和提供商常量：

    ```typescript
    const model = api.runtime.agent.defaults.model; // 例如 "gpt-5.6-sol"
    const provider = api.runtime.agent.defaults.provider; // 例如 "openai"
    ```

  </Accordion>

  <Accordion title="api.runtime.llm">
    运行由宿主管理的文本补全，无需导入提供商内部实现，也无需重复 OpenClaw 的模型/身份验证/基础 URL 准备逻辑。

    ```typescript
    const result = await api.runtime.llm.complete({
      messages: [{ role: "user", content: "总结此转录。" }],
      purpose: "my-plugin.summary",
      maxTokens: 512,
      temperature: 0.2,
      reasoning: "high",
    });
    ```

    提供商编排还可以在发出 HTTP 请求之前获取已配置本地服务的生命周期：

    ```typescript
    const lease = await api.runtime.llm.acquireLocalService(
      {
        providerId,
        baseUrl,
        headers,
      },
      signal,
    );
    try {
      // 发送并完整读取提供商请求。
    } finally {
      await lease?.release();
    }
    ```

    `acquireLocalService(...)` 是稳定的通用提供商服务 SDK 契约。宿主从 `models.providers.<providerId>.localService` 解析进程配置；调用方不能提供命令、参数、环境或生命周期策略。进程生成、就绪检查、诊断和空闲停止策略仍由宿主内部管理。

    请传入配置中确切的提供商 ID 和解析后的请求基础 URL。不要用适配器 ID 替换别名：不同别名可能指向不同的本地 GPU 主机。除 Ollama 和 LM Studio 适配器使用的 `/v1` 规范化外，宿主会拒绝与所配置提供商基础 URL 不匹配的端点。宿主负责启动串行化、就绪探测、请求租约、中止处理和空闲关闭。

    此辅助函数使用与 OpenClaw 内置运行时相同的简单补全准备路径，以及由宿主管理的运行时配置快照。上下文引擎会收到绑定到会话的 `llm.complete` 能力，因此模型调用会使用活动会话的智能体，不会静默回退到默认智能体。结果包含提供商/模型/智能体归属信息，以及可用时经过规范化的 token、缓存和估算成本用量。

    设置 `reasoning` 可为所选模型请求推理强度。宿主会在分派补全之前，针对所选提供商和模型规范化标准思考级别（`off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`adaptive`、`max` 和 `ultra`）。`adaptive` 会变为 `medium`；`max` 和 `ultra` 在受支持时会变为 `max`，否则变为 `xhigh`。

    <Warning>
    模型覆盖需要操作员通过配置中的 `plugins.entries.<id>.llm.allowModelOverride: true` 明确选择启用。使用 `plugins.entries.<id>.llm.allowedModels` 将受信任插件限制为特定的规范化 `provider/model` 目标。跨智能体补全需要 `plugins.entries.<id>.llm.allowAgentIdOverride: true`。
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.gateway">
    在进程内调用另一个 Gateway 网关方法，同时保留当前插件的受信任运行时身份。此功能适用于内置或受信任的官方插件，让它们能够组合由插件拥有的 Gateway 网关能力，而无需建立 local loopback WebSocket 连接。

    ```typescript
    if (await api.runtime.gateway.isAvailable()) {
      const result = await api.runtime.gateway.request<{ callId: string }>(
        "voicecall.start",
        { to: "+15550001234", mode: "conversation" },
        { timeoutMs: 60_000 },
      );
    }
    ```

    请求使用 `operator.write` 作用域，不授予管理员作用域。来自任意外部插件的调用会被拒绝。失败的方法会抛出 `GatewayClientRequestError`，并保留结构化的 `details`、重试元数据和 Gateway 网关错误代码，以供恢复流程使用。对于也可在独立智能体进程中运行的工具，请在选择此路径之前使用 `isAvailable()`。

  </Accordion>
  <Accordion title="api.runtime.subagent">
    启动和管理后台子智能体运行。

    ```typescript
    // 启动子智能体运行
    const { runId } = await api.runtime.subagent.run({
      sessionKey: "agent:main:subagent:search-helper",
      message: "将此查询扩展为重点明确的后续搜索。",
      toolsAlsoAllow: ["my_plugin_progress"],
      provider: "openai", // 可选覆盖
      model: "gpt-5.6-sol", // 可选覆盖
      deliver: false,
    });

    // 等待完成
    const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

    // 读取会话消息
    const { messages } = await api.runtime.subagent.getSessionMessages({
      sessionKey: "agent:main:subagent:search-helper",
      limit: 10,
    });

    // 删除会话
    await api.runtime.subagent.deleteSession({
      sessionKey: "agent:main:subagent:search-helper",
    });
    ```

    <Warning>
    模型覆盖（`provider`/`model`）需要操作员通过配置中的 `plugins.entries.<id>.subagent.allowModelOverride: true` 明确选择启用。不受信任的插件仍可运行子智能体，但覆盖请求会被拒绝。
    </Warning>

    `toolsAlsoAllow` 会将调用插件注册且由其唯一拥有的精确工具添加到工作进程的常规工具接口中。运行时会拒绝核心工具以及与其他插件共享名称的工具。配置文件和操作员工具策略仍然适用，包括显式允许列表和拒绝规则。

    `deleteSession(...)` 可以删除同一插件通过 `api.runtime.subagent.run(...)` 创建的会话。删除任意用户或操作员会话仍需要具有管理员作用域的 Gateway 网关请求。

  </Accordion>
  <Accordion title="api.runtime.sandbox">
    检查智能体会话的有效沙箱工作区权限。

    ```typescript
    const authority = api.runtime.sandbox.resolveWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
    });

    const liveAuthority = await api.runtime.sandbox.prepareWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
      workspaceDir,
      confinedToolNames: ["my_plugin_safe_tool"],
    });
    ```

    结果会报告此会话是否经过沙箱隔离、其工作区是否不可用、只读或可写，以及当有效的 Docker、工具、会话、浏览器或提升权限策略可能逃逸该工作区时提供可选的 `confinementError`。请将其用于由宿主管理的委派决策，确保不会向工作进程授予超出调用方的权限。它是证明辅助函数，不能替代对调用方自身授权的检查。

    `prepareWorkspaceAuthority(...)` 执行相同的策略检查，并且还会为 `workspaceDir` 准备 Docker 沙箱。如果正在运行的容器的实时配置哈希与请求的挂载或策略不匹配，它会拒绝该容器。仅传入由调用插件限制其注册实现的确切工具名称；通配符前缀无法证明工具所有权。

  </Accordion>
  <Accordion title="api.runtime.nodes">
    从 Gateway 网关加载的插件代码或插件 CLI 命令中列出已连接的节点并调用节点主机命令。当插件负责在已配对设备上执行本地工作时，请使用此功能，例如另一台 Mac 上的浏览器或音频桥接。

    ```typescript
    const { nodes } = await api.runtime.nodes.list({ connected: true });

    const result = await api.runtime.nodes.invoke({
      nodeId: "mac-studio",
      command: "my-plugin.command",
      params: { action: "start" },
      timeoutMs: 30000,
    });
    ```

    当节点向智能体公开由插件或 MCP 支持的工具时，`nodes.list(...)` 会包含每个已连接节点所通告的
    `nodePluginTools` 描述符。这些描述符属于实时连接状态：节点断开连接时，Gateway 网关
    会丢弃它们；本地插件/MCP 清单发生变化后，节点可以用
    `node.pluginTools.update` 替换它们。

    在 Gateway 网关内部，此运行时在进程内运行。在插件 CLI 命令中，它通过 RPC 调用已配置的 Gateway 网关，因此 `openclaw googlemeet recover-tab` 等命令可以从终端检查已配对的节点。节点命令仍会经过常规的 Gateway 网关节点配对、命令允许列表、插件节点调用策略和节点本地命令处理。

    公开节点托管型智能体工具的插件可以为应默认加入允许列表的非危险命令设置 `agentTool.defaultPlatforms`。如果必须由操作员通过 `gateway.nodes.commands.allow` 选择启用，请省略该字段。危险的节点宿主命令应使用 `api.registerNodeInvokePolicy(...)` 注册节点调用策略；该策略会在 Gateway 网关中运行，执行时机是在命令允许列表检查之后、命令转发到节点之前，因此直接的 `node.invoke` 调用、节点托管型插件工具和更高级别的插件工具会共享同一执行约束路径。

    <Warning>
    可选的 `scopes` 字段为调用请求 Gateway 网关操作员权限范围。OpenClaw 仅对内置插件和受信任的官方插件安装支持此字段；其他插件的请求不会提升调用权限。仅当受信任的插件必须使用更严格的 Gateway 网关权限范围（例如 `operator.admin`）调用节点命令时，才使用此字段。
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.tasks">
    将 Task Flow 和 Task Run 状态绑定到现有的 OpenClaw 会话键或受信任的工具上下文。

    - `api.runtime.tasks.managedFlows` 支持变更操作：创建、推进和取消 Task Flow。
    - `api.runtime.tasks.flows` 和 `api.runtime.tasks.runs` 是用于列表和状态查询的只读 DTO 视图；二者均公开 `bindSession(...)` / `fromToolContext(...)`，以及 `get`、`list`、`findLatest` 和 `resolve`。

    Task Flow 跟踪持久的多步骤工作流状态。它不是调度器：
    对于未来的唤醒，请使用 Cron 或 `api.session.workflow.scheduleSessionTurn(...)`；
    如果相应工作需要流程状态、子任务、等待或取消，请在计划轮次中使用
    `managedFlows`。

    ```typescript
    const taskFlow = api.runtime.tasks.managedFlows.fromToolContext(ctx);

    const created = taskFlow.createManaged({
      controllerId: "my-plugin/review-batch",
      goal: "审查新的拉取请求",
    });

    const child = taskFlow.runTask({
      flowId: created.flowId,
      runtime: "acp",
      childSessionKey: "agent:main:subagent:reviewer",
      task: "审查 PR #123",
      status: "running",
      startedAt: Date.now(),
    });

    const waiting = taskFlow.setWaiting({
      flowId: created.flowId,
      expectedRevision: created.revision,
      currentStep: "await-human-reply",
      waitJson: { kind: "reply", channel: "telegram" },
    });
    ```

    如果你已经从自己的绑定层获得受信任的 OpenClaw 会话键，请使用 `bindSession({ sessionKey, requesterOrigin })`。不要根据原始用户输入进行绑定。

  </Accordion>
  <Accordion title="api.runtime.tts">
    文本转语音合成。

    ```typescript
    // 标准 TTS
    const clip = await api.runtime.tts.textToSpeech({
      text: "来自 OpenClaw 的问候",
      cfg: api.config,
    });

    // 针对电话通信优化的 TTS
    const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
      text: "来自 OpenClaw 的问候",
      cfg: api.config,
    });

    // 列出可用语音
    const voices = await api.runtime.tts.listVoices({
      provider: "elevenlabs",
      cfg: api.config,
    });
    ```

    使用核心 `tts` 配置和提供商选择。返回 PCM 音频缓冲区和采样率。`textToSpeechStream` 也可用于流式合成。

  </Accordion>
  <Accordion title="api.runtime.mediaUnderstanding">
    图像、音频和视频分析。

    ```typescript
    // 描述图像
    const image = await api.runtime.mediaUnderstanding.describeImageFile({
      filePath: "/tmp/inbound-photo.jpg",
      cfg: api.config,
      agentDir: "/tmp/agent",
    });

    // 转写音频
    const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
      filePath: "/tmp/inbound-audio.ogg",
      cfg: api.config,
      mime: "audio/ogg", // 可选，适用于无法推断 MIME 的情况
    });

    // 描述视频
    const video = await api.runtime.mediaUnderstanding.describeVideoFile({
      filePath: "/tmp/inbound-video.mp4",
      cfg: api.config,
    });

    // 通用文件分析
    const result = await api.runtime.mediaUnderstanding.runFile({
      filePath: "/tmp/inbound-file.pdf",
      cfg: api.config,
    });

    // 通过指定的提供商/模型执行结构化图像提取。
    // 至少包含一张图像；文本输入用于补充上下文。
    const evidence = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
      provider: "codex",
      model: "gpt-5.6-sol",
      input: [
        {
          type: "image",
          buffer: receiptImageBuffer,
          fileName: "receipt.png",
          mime: "image/png",
        },
        { type: "text", text: "优先采用印刷的总金额，而不是手写备注。" },
      ],
      instructions: "提取商家、总金额和可搜索的标签。",
      schemaName: "receipt.evidence",
      jsonSchema: {
        type: "object",
        properties: {
          vendor: { type: "string" },
          total: { type: "number" },
          tags: { type: "array", items: { type: "string" } },
        },
        required: ["vendor", "total"],
      },
      cfg: api.config,
    });
    ```

    未产生输出时（例如跳过了输入），返回 `{ text: undefined }`。

    `describeImageFileWithModel(...)` 通过指定的提供商/模型描述已知图像，并绕过 `describeImageFile(...)` 使用的默认活动模型解析。

  </Accordion>
  <Accordion title="api.runtime.imageGeneration">
    图像生成。

    ```typescript
    const result = await api.runtime.imageGeneration.generate({
      prompt: "一个正在绘制日落的机器人",
      cfg: api.config,
    });

    const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.videoGeneration">
    视频生成，其结构与图像生成相同。

    ```typescript
    const result = await api.runtime.videoGeneration.generate({
      prompt: "日出时飞越海岸线的无人机航拍镜头",
      cfg: api.config,
    });

    const providers = api.runtime.videoGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.musicGeneration">
    音乐生成，其结构与图像生成相同。

    ```typescript
    const result = await api.runtime.musicGeneration.generate({
      prompt: "适合编程时聆听的欢快 lo-fi 曲目",
      cfg: api.config,
    });

    const providers = api.runtime.musicGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.webSearch">
    Web 搜索。

    ```typescript
    const providers = api.runtime.webSearch.listProviders({ config: api.config });

    const result = await api.runtime.webSearch.search({
      config: api.config,
      args: { query: "OpenClaw 插件 SDK", count: 5 },
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.media">
    底层媒体实用工具。

    ```typescript
    const webMedia = await api.runtime.media.loadWebMedia(url);
    const mime = await api.runtime.media.detectMime(buffer);
    const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "image"
    const isVoice = api.runtime.media.isVoiceCompatibleAudio(filePath);
    const metadata = await api.runtime.media.getImageMetadata(filePath);
    const resized = await api.runtime.media.resizeToJpeg(buffer, { maxWidth: 800 });
    const terminalQr = await api.runtime.media.renderQrTerminal("https://openclaw.ai");
    const pngQr = await api.runtime.media.renderQrPngBase64("https://openclaw.ai", {
      scale: 6, // 1-12
      marginModules: 4, // 0-16
    });
    const pngQrDataUrl = await api.runtime.media.renderQrPngDataUrl("https://openclaw.ai");
    const tmpRoot = resolvePreferredOpenClawTmpDir();
    const pngQrFile = await api.runtime.media.writeQrPngTempFile("https://openclaw.ai", {
      tmpRoot,
      dirPrefix: "my-plugin-qr-",
      fileName: "qr.png",
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.config">
    当前运行时配置快照和事务式配置写入。优先使用
    已传入活动调用路径的配置；仅当处理程序需要直接获取进程快照时，
    才使用 `current()`。

    ```typescript
    const cfg = api.runtime.config.current();
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    `mutateConfigFile(...)` 和 `replaceConfigFile(...)` 返回一个 `followUp`
    值，例如 `{ mode: "restart", requiresRestart: true, reason }`；
    该值记录写入者的意图，同时不会从 Gateway 网关手中夺取重启控制权。

  </Accordion>
  <Accordion title="api.runtime.system">
    系统级实用工具。

    ```typescript
    await api.runtime.system.enqueueSystemEvent(event);
    api.runtime.system.requestHeartbeat({
      source: "other",
      intent: "event",
      reason: "plugin-event",
    });
    api.runtime.system.requestHeartbeatNow({ reason: "plugin-event" }); // 已弃用的兼容性别名。
    const heartbeatResult = await api.runtime.system.runHeartbeatOnce({
      reason: "plugin-triggered-check",
    });
    const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
    const hint = api.runtime.system.formatNativeDependencyHint(pkg);
    ```

    `runHeartbeatOnce(...)` 会立即运行单次 Heartbeat 周期，绕过常规的合并计时器。传入 `{ heartbeat: { target: "last" } }` 可强制将内容发送到最后活跃的渠道，而不是执行默认的 `target: "none"` 抑制。

    `runCommandWithTimeout(...)` 返回捕获的 `stdout` 和 `stderr`、可选的
    截断计数、`code`、`signal`、`killed`、`termination` 和
    `noOutputTimedOut`。如果子进程未提供非零退出代码，超时和无输出超时结果会报告 `code: 124`。
    非超时的信号退出仍可能返回 `code: null`，因此请使用 `termination` 和
    `noOutputTimedOut` 区分超时原因。

  </Accordion>
  <Accordion title="api.runtime.events">
    事件订阅。

    ```typescript
    api.runtime.events.onAgentEvent((event) => {
      /* ... */
    });
    api.runtime.events.onSessionTranscriptUpdate((update) => {
      /* ... */
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.logging">
    日志。

    ```typescript
    const verbose = api.runtime.logging.shouldLogVerbose();
    const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
    ```

  </Accordion>
  <Accordion title="api.runtime.modelAuth">
    模型和提供商身份验证解析。

    ```typescript
    const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });

    // 可直接用于请求的身份验证，包括提供商运行时交换（例如 OAuth 刷新）
    const runtimeAuth = await api.runtime.modelAuth.getRuntimeAuthForModel({ model, cfg });

    const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
      provider: "openai",
      cfg,
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.state">
    状态目录解析和由 SQLite 支持的键控存储。

    ```typescript
    const stateDir = api.runtime.state.resolveStateDir(process.env);
    const store = api.runtime.state.openKeyedStore<MyRecord>({
      namespace: "my-feature",
      maxEntries: 200,
      defaultTtlMs: 15 * 60_000,
    });

    await store.register("key-1", { value: "hello" });
    const claimed = await store.registerIfAbsent("dedupe-key", { value: "first" });
    const value = await store.lookup("key-1");
    await store.deleteIf?.("key-1", (current) => current.value === "hello");
    await store.consume("key-1");
    await store.clear();

    const blobs = api.runtime.state.openBlobStore<MyBlobMetadata>({
      namespace: "rendered-artifacts",
      maxEntries: 100,
      maxBytesPerEntry: 4 * 1024 * 1024,
      maxBytesPerNamespace: 64 * 1024 * 1024,
      defaultTtlMs: 15 * 60_000,
    });
    await blobs.register(
      "artifact-1",
      new TextEncoder().encode("binary or text payload"),
      { contentType: "text/plain" },
    );
    const blob = await blobs.lookup("artifact-1");

    await api.runtime.state.withLease(
      {
        namespace: "my-feature",
        key: "writer",
        database: { scope: "agent", agentId },
        leaseMs: 5 * 60_000,
        waitMs: 30_000,
      },
      async ({ signal, assertOwned }) => {
        await runExternalWriter({ signal });
        assertOwned();
      },
    );
    ```

    键控存储可在重启后继续保留，并按运行时绑定的插件 ID 隔离。使用 `registerIfAbsent(...)` 进行原子去重声明：当键不存在或已过期并完成注册时，它返回 `true`；当仍有效的值已存在时，它返回 `false`，且不会覆盖其值、创建时间或 TTL。当清理操作只能移除此前观察到的值时，使用 `deleteIf(...)`；其同步谓词和删除操作在同一个 SQLite 事务中运行。限制：每个命名空间 `maxEntries`，每个插件 50,000 个有效行，JSON 值小于 64KB，并可选 TTL 过期机制。默认情况下，在达到任一行数限制时执行写入，会从当前写入的命名空间中清除最旧的有效行；不会为该次写入驱逐同级命名空间中的行，如果命名空间无法释放足够的行，写入仍会失败。对于绝不能被驱逐的持久所有权记录，请设置 `overflowPolicy: "reject-new"`：新键在达到任一限制时会失败，而现有键仍可更新。

    `openSyncKeyedStore<T>(...)` 返回相同结构的存储，但使用同步方法（`register`、`registerIfAbsent`、`deleteIf`、`lookup`、`consume`、`clear` 均直接返回值而非 Promise），供无法使用 await 的调用方使用。

    `openBlobStore<TMetadata>(...)` 在共享 SQLite 中存储有界二进制载荷，无需 base64 或文件伴随存储。它要求设置单条目、单命名空间字节数和行数限制；在 API 边界复制字节数组；并在不加载每个 BLOB 的情况下列出元数据。`register(...)` 是显式 upsert，也适用于已过期的键。`registerIfAbsent(...)` 提供无冲突创建：已过期的键仍视为被占用，直到其所有者使用 `deleteExpiredKey(key)` 或 `deleteExpired()` 声明该键，从而保留在 SQLite 提交后移除相关具名工件所需的元数据。任何带 TTL 的行都是临时数据，即使尚未过期，也不会包含在备份/恢复中；持久且可恢复的状态应省略 TTL。主机熔断限制为每个 BLOB 100 MiB、每个插件实际存储的 BLOB 总量 512 MiB，以及每个插件实际存储的行数 50,000，其中包括等待所有者清理的已过期行。当外部实体化内容不得因替换或驱逐而被静默遗弃时，请将 `registerIfAbsent(...)` 与 `overflowPolicy: "reject-new"` 配合使用。

    `openChannelIngressQueue<TPayload>(...)` 打开一个限定于调用插件的持久化入口队列，用于缓冲需要跨重启进行至少一次处理的入站事件。当陈旧声明恢复使用 `shouldRecover` 时，如果损坏的已声明载荷应被隔离，还需提供 `shouldRecoverCorrupt`：其与载荷无关的声明标识使插件能够在队列将该行标记为墓碑前，保留当前所有者和通道策略。

    `withLease(...)` 对 OpenClaw 进程间的协作式插件工作进行串行化。选择 `database: { scope: "shared" }` 可使用一个全局所有者，选择 `{ scope: "agent", agentId }` 可实现按 Agent 独立所有权。将回调的 `AbortSignal` 传递给每个可能失败的操作。`assertOwned()` 是开始另一个重要步骤前的时间点检查；主机也会在回调后验证所有权。租约丢失或调用方取消会中止该信号。获取等待和心跳发生在短时同步 SQLite 事务之外；插件永远不会收到数据库路径或句柄。这是协作式取消，而不是隔离令牌，也不构成执行未隔离外部写入的授权。

    `openChannelIngressDrain(...)` 在该队列之上打开与核心渠道无关的工作器（如果未提供队列，则创建一个）。清空操作负责陈旧声明恢复、按通道串行声明、采用时完成或分发返回时完成、重试/死信处置、可选的采用前取代，以及声明→采用停滞超时。使用 `turnAdoptionLifecycle` 将声明所有权接入回复生成（通过 `plugin-sdk/channel-outbound` 中的 `bindIngressLifecycleToReplyOptions`）。渠道插件保留接收侧入队、通道派生、不可重试分类，以及任何取代授权策略。

    <Warning>
    在此版本中，`openBlobStore`、`openKeyedStore`、`openSyncKeyedStore`、`withLease`、`openChannelIngressQueue` 和 `openChannelIngressDrain` 仅可用于内置插件和受信任的官方插件安装。
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.channel">
    渠道特定的运行时辅助方法（加载渠道插件时可用）。按关注点分组：

    | 分组 | 用途 |
    | --- | --- |
    | `text` | 分块（`chunkText`、`chunkMarkdownText`、`resolveChunkMode`）、控制命令检测、Markdown 表格转换。 |
    | `reply` | 缓冲块回复分发、信封格式化、有效消息/人工延迟配置解析。 |
    | `routing` | `buildAgentSessionKey`、`resolveAgentRoute`。 |
    | `pairing` | `buildPairingReply`、允许列表读取/移除、配对请求 upsert，以及由请求派生的审批条目。 |
    | `media` | 远程媒体下载/保存（见下文）。 |
    | `activity` | 记录/读取最后一次渠道活动。 |
    | `session` | 来自入站事件的会话元数据、最后路由更新。 |
    | `mentions` | 提及策略辅助方法（见下文）。 |
    | `reactions` | 用于处理中指示器的确认表情回应句柄。 |
    | `groups` | 群组策略和必须提及解析。 |
    | `debounce` | 入站消息防抖。 |
    | `commands` | 命令授权和文本命令门控。 |
    | `outbound` | 加载渠道的出站适配器。 |
    | `inbound` | 构建入站事件上下文并运行共享的入站事件/回复内核。 |
    | `threadBindings` | 调整已绑定会话线程的空闲超时/最大时长。 |
    | `runtimeContexts` | 注册、读取和监视进程本地的每渠道/账户/能力上下文。 |

    `api.runtime.channel.media` 是渠道媒体下载和存储的首选接口：

    ```typescript
    const saved = await api.runtime.channel.media.saveRemoteMedia({
      url,
      subdir: "inbound",
      maxBytes,
      filePathHint: fileName,
    });
    ```

    当远程 URL 应转换为 OpenClaw 媒体时，使用 `saveRemoteMedia(...)`。当插件已使用插件自有的身份验证、重定向或允许列表处理获取 `Response` 时，使用 `saveResponseMedia(...)`。仅当插件需要原始字节进行检查、转换、解密或重新上传时，才使用 `readRemoteMediaBuffer(...)`。`fetchRemoteMedia(...)` 仍是 `readRemoteMediaBuffer(...)` 的已弃用兼容别名。

    `api.runtime.channel.mentions` 是供使用运行时注入的内置渠道插件使用的共享入站提及策略接口：

    ```typescript
    const mentionMatch = api.runtime.channel.mentions.matchesMentionWithExplicit(text, {
      mentionRegexes,
      mentionPatterns,
    });

    const decision = api.runtime.channel.mentions.resolveInboundMentionDecision({
      facts: {
        canDetectMention: true,
        wasMentioned: mentionMatch.matched,
        implicitMentionKinds: api.runtime.channel.mentions.implicitMentionKindWhen(
          "reply_to_bot",
          isReplyToBot,
        ),
      },
      policy: {
        isGroup,
        requireMention,
        allowTextCommands,
        hasControlCommand,
        commandAuthorized,
      },
    });
    ```

    可用的提及辅助方法：

    - `buildMentionRegexes`
    - `matchesMentionPatterns`
    - `matchesMentionWithExplicit`
    - `implicitMentionKindWhen`
    - `resolveInboundMentionDecision`

    使用规范化的 `{ facts, policy }` 路径进行提及决策。

    `reply`、`session` 和 `inbound` 下的多个字段带有逐字段的 `@deprecated` 注释，指向当前的渠道轮次内核或渠道出站适配器；在基于特定辅助方法构建新代码前，请查看其内联 JSDoc。

  </Accordion>
</AccordionGroup>

## 存储运行时引用

使用 `createPluginRuntimeStore` 存储运行时引用，以便在 `register` 回调之外使用：

<Steps>
  <Step title="创建存储">
    ```typescript
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

    const store = createPluginRuntimeStore<PluginRuntime>({
      pluginId: "my-plugin",
      errorMessage: "my-plugin runtime not initialized",
    });
    ```

  </Step>
  <Step title="接入入口点">
    ```typescript
    export default defineChannelPluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Example",
      plugin: myPlugin,
      setRuntime: store.setRuntime,
    });
    ```
  </Step>
  <Step title="从其他文件访问">
    ```typescript
    export function getRuntime() {
      return store.getRuntime(); // throws if not initialized
    }

    export function tryGetRuntime() {
      return store.tryGetRuntime(); // returns null if not initialized
    }
    ```

  </Step>
</Steps>

<Note>
运行时存储标识应优先使用 `pluginId`。较底层的 `key` 形式适用于一个插件有意需要多个运行时槽位的少见情况。
</Note>

## 其他顶层 `api` 字段

除 `api.runtime` 外，API 对象还提供：

<ParamField path="api.id" type="string">
  插件 ID。
</ParamField>
<ParamField path="api.name" type="string">
  插件显示名称。
</ParamField>
<ParamField path="api.config" type="OpenClawConfig">
  当前配置快照（可用时为内存中有效的运行时快照）。
</ParamField>
<ParamField path="api.pluginConfig" type="Record<string, unknown>">
  来自 `plugins.entries.<id>.config` 的插件专用配置。
</ParamField>
<ParamField path="api.logger" type="PluginLogger">
  限定作用域的日志记录器（`debug`、`info`、`warn`、`error`）。
</ParamField>
<ParamField path="api.registrationMode" type="PluginRegistrationMode">
  当前加载模式：`"full"`（实时激活）、`"discovery"` / `"tool-discovery"`（只读能力发现）、`"setup-only"`（轻量级设置入口）、`"setup-runtime"`（还需要运行时渠道入口的设置流程），或 `"cli-metadata"`（CLI 命令元数据收集）。
</ParamField>
<ParamField path="api.resolvePath(input)" type="(string) => string">
  解析相对于插件根目录的路径。
</ParamField>

## 相关内容

- [插件内部机制](/zh-CN/plugins/architecture) — 能力模型和注册表
- [SDK 入口点](/zh-CN/plugins/sdk-entrypoints) — `definePluginEntry` 选项
- [SDK 概览](/zh-CN/plugins/sdk-overview) — 子路径参考
