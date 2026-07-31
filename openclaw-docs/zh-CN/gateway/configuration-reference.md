---
read_when:
    - 你需要确切的字段级配置语义或默认值
    - 你正在验证渠道、模型、Gateway 网关或工具配置块
summary: 核心 OpenClaw 键、默认值及专用子系统参考链接的 Gateway 配置参考
title: 配置参考
x-i18n:
    generated_at: "2026-07-26T06:14:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7135554fda444fd1b8c072af5768c53a165f7be2dcd12a7909fc7fd4bd864428
    source_path: gateway/configuration-reference.md
    workflow: 16
---

`~/.openclaw/openclaw.json` 的字段级参考：键、默认值以及指向更深入子系统页面的链接。有关面向任务的设置指导，请参阅[配置](/zh-CN/gateway/configuration)。由渠道和插件所有的命令目录以及深层记忆/QMD 调节项位于各自的页面，而不在此处。

配置格式为 **JSON5**（允许注释和尾随逗号）。所有字段均为可选；省略时，OpenClaw 会使用安全的默认值。

代码事实优先于此页面：

- `openclaw config schema` 输出用于验证和 Control UI 的实时 JSON Schema，其中已合并内置插件和渠道的元数据。
- 编辑配置前，智能体应调用 `gateway` 工具操作 `config.schema.lookup`，获取一个精确的路径范围 Schema 节点。
- `pnpm config:docs:check` / `pnpm config:docs:gen` 根据当前 Schema 表面验证本文档的基线哈希。

Schema `uiHints` 还为每个路径携带解析后的 `advanced` 布尔值。
Control UI 使用该值优先显示常用字段，并按
章节折叠高级字段；搜索仍会覆盖两个层级。层级元数据仅用于呈现。
添加键时，请在叶节点上声明其层级，或让其继承最近
祖先的声明。没有已声明祖先的路径默认属于高级层级。

专门的深入参考：

- [记忆配置参考](/zh-CN/reference/memory-config)，涵盖 `memory.search.*`、`memory.qmd.*`、`memory.citations` 以及 `plugins.entries.memory-core.config.dreaming` 下的 Dreaming 配置。
- [斜杠命令](/zh-CN/tools/slash-commands)，涵盖当前的内置 + 捆绑命令目录。
- 有关渠道特定的命令表面，请参阅其所属的渠道/插件页面。

---

## 渠道

各渠道的配置键位于[配置 - 渠道](/zh-CN/gateway/config-channels)：Slack、Discord、Telegram、WhatsApp、Matrix、iMessage 及其他内置渠道的 `channels.*`（身份验证、访问控制、多账号和提及门控）。

## Agent 默认值、多 Agent、会话和消息

有关以下内容，请参阅[配置 - Agent](/zh-CN/gateway/config-agents)：

- `agents.defaults.*`（工作区、模型、思考、Heartbeat、记忆、媒体、技能、沙箱）
- `multiAgent.*`（多 Agent 路由和绑定）
- `session.*`（会话生命周期、压缩、修剪）
- `messages.*`（消息传递、TTS、Markdown 渲染）
- `talk.*`（Talk 模式）
  - `talk.consultThinkingLevel`：覆盖 Control UI Talk 实时咨询背后的完整 OpenClaw 智能体运行的思考级别
  - `talk.consultFastMode`：Control UI Talk 实时咨询的一次性快速模式覆盖
  - `talk.speechLocale`：用于 Android、iOS 和 macOS 上 Talk 语音识别的可选 BCP 47 区域设置 ID
  - `talk.silenceTimeoutMs`：未设置时，Talk 会在发送转录文本前保留平台的默认暂停窗口（`700 ms on macOS and Android, 900 ms on iOS`）
  - `talk.realtime.consultRouting`：用于跳过 `openclaw_agent_consult` 的已完成实时 Talk 转录文本的 Gateway 网关中继回退

## 工具和自定义提供商

工具策略、实验性开关、由提供商支持的工具配置以及自定义
提供商/base URL 设置位于
[配置 - 工具和自定义提供商](/zh-CN/gateway/config-tools)。

## 模型

提供商定义、模型允许列表和自定义提供商设置位于
[配置 - 工具和自定义提供商](/zh-CN/gateway/config-tools#custom-providers-and-base-urls)。
`models` 根节点还负责全局模型目录行为。

```json5
{
  models: {
    // 可选。默认值：true。更改后需要重启 Gateway 网关。
    pricing: { enabled: false },
  },
}
```

- `models.mode`：提供商目录行为（`merge` 或 `replace`）。
- `models.providers`：以提供商 ID 为键的自定义提供商映射。
- `models.providers.*.localService`：用于
  本地模型服务器的可选按需进程管理器。OpenClaw 会探测配置的健康端点，在需要时启动
  绝对路径 `command`，等待其就绪，然后发送模型
  请求。请参阅[本地模型服务](/zh-CN/gateway/local-model-services)。
- `models.pricing.enabled`：控制后台定价引导，该过程
  会在边车和渠道进入 Gateway 网关就绪路径后启动。当 `false` 时，
  Gateway 网关会跳过获取 OpenRouter 和 LiteLLM 定价目录；配置的
  `models.providers.*.models[].cost` 值仍可用于本地成本估算。

## MCP

由 OpenClaw 管理的 MCP 服务器定义位于 `mcp.servers` 下，并由
嵌入式 OpenClaw 和其他运行时适配器使用。`openclaw mcp list`、
`show`、`set` 和 `unset` 命令可管理此配置块，而无需在编辑配置时连接到
目标服务器。

```json5
{
  mcp: {
    servers: {
      docs: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-fetch"],
      },
      remote: {
        url: "https://example.com/mcp",
        transport: "streamable-http", // streamable-http | sse
        requestTimeoutMs: 20000,
        connectionTimeoutMs: 5000,
        supportsParallelToolCalls: true,
        headers: {
          Authorization: "Bearer ${MCP_REMOTE_TOKEN}",
        },
        auth: "oauth",
        oauth: {
          scope: "docs.read",
        },
        sslVerify: true,
        clientCert: "/path/to/client.crt",
        clientKey: "/path/to/client.key",
        toolFilter: {
          include: ["search_*"],
          exclude: ["admin_*"],
        },
        // 可选的 Codex 应用服务器投影控制项。
        codex: {
          agents: ["main"],
          defaultToolsApprovalMode: "approve", // auto | prompt | approve
        },
      },
    },
  },
}
```

- `mcp.servers`：用于公开已配置 MCP 工具的运行时的具名 stdio 或远程 MCP 服务器定义。
  远程条目使用 `transport: "streamable-http"` 或 `transport: "sse"`；
  `type: "http"` 是 CLI 原生别名，`openclaw mcp set` 和
  `openclaw doctor --fix` 会将其规范化为标准 `transport` 字段。
- `mcp.servers.<name>.enabled`：设置 `false` 可保留已保存的服务器定义，
  同时将其从嵌入式 OpenClaw MCP 设备发现和工具投影中排除。
- `mcp.servers.<name>.requestTimeoutMs`：每个服务器的 MCP 请求超时，以毫秒为单位。
- `mcp.servers.<name>.connectionTimeoutMs`：每个服务器的连接超时，以毫秒为单位。
- `mcp.servers.<name>.supportsParallelToolCalls`：可选并发提示，供
  能够选择是否发出并行 MCP 工具调用的适配器使用。
- `mcp.servers.<name>.auth`：对于需要 OAuth 的 HTTP MCP 服务器，设置 `"oauth"`。
  运行 `openclaw mcp login <name>`，将令牌存储在 OpenClaw 状态中。
- `mcp.servers.<name>.oauth`：可选的 OAuth 权限范围、重定向 URL 和客户端
  元数据 URL 覆盖。
- `mcp.servers.<name>.sslVerify`、`clientCert`、`clientKey`：用于
  私有端点和双向 TLS 的 HTTP TLS 控制项。
- `mcp.servers.<name>.toolFilter`：可选的按服务器工具选择。`include`
  将发现的 MCP 工具限制为名称匹配的工具；`exclude` 会隐藏名称匹配的
  工具。条目可以是准确的 MCP 工具名称，也可以是简单的 `*` glob。具有
  资源或提示词的服务器还会生成实用工具名称（`resources_list`、
  `resources_read`、`prompts_list`、`prompts_get`），这些名称使用
  相同的过滤器。
- `mcp.servers.<name>.codex`：可选的 Codex 应用服务器投影控制项。
  此配置块仅是面向 Codex 应用服务器线程的 OpenClaw 元数据；它不会
  影响 ACP 会话、通用 Codex harness 配置或其他运行时适配器。
  非空的 `codex.agents` 会将服务器限制为列出的 OpenClaw 智能体 ID。
  空白、空值或无效的限定范围智能体列表会被配置验证拒绝，
  并由运行时投影路径忽略，而不会变为全局设置。
  `codex.defaultToolsApprovalMode` 会为该服务器输出 Codex 原生的
  `default_tools_approval_mode`。OpenClaw 会先移除 `codex`
  配置块，再将原生 `mcp_servers` 配置传递给 Codex。省略此配置块可
  使服务器继续投影到每个 Codex 应用服务器智能体，并使用 Codex
  默认的 MCP 审批行为。
- 会话范围的内置 MCP 运行时使用内置的 10 分钟空闲 TTL。
  一次性嵌入式运行会请求在运行结束时清理；TTL 是针对长生命周期会话和未来调用方的兜底机制。
- `mcp.*` 下的更改会通过释放缓存的会话 MCP 运行时来热应用。
  下一次工具发现/使用时，会根据新配置重新创建这些运行时，因此已移除的
  `mcp.servers` 条目会立即回收，而不是等待空闲 TTL。
- 运行时发现还会通过丢弃
  该会话的缓存目录来响应 MCP 工具列表变更通知。公布资源或
  提示词的服务器会获得用于列出/读取资源以及列出/获取
  提示词的实用工具。工具调用反复失败时，受影响的服务器会暂停片刻，然后
  才会再次尝试调用。

有关运行时行为，请参阅 [MCP](/zh-CN/cli/mcp#openclaw-as-an-mcp-client-registry) 和
[CLI 后端](/zh-CN/gateway/cli-backends#bundle-mcp-overlays)。

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
      allowUploadedArchives: false,
    },
    workshop: {
      allowSymlinkTargetWrites: false,
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // 或纯文本字符串
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`：仅适用于内置技能的可选允许列表（不影响托管/工作区技能）。
- `load.extraDirs`：额外的共享技能根目录（优先级最低）。
- `load.allowSymlinkTargets`：当技能符号链接位于其配置的源根目录之外时，
  允许其解析到的受信任真实目标根目录。
- `workshop.allowSymlinkTargetWrites`：允许 Skill Workshop 应用操作通过
  已受信任的符号链接目标进行写入（默认值：false）。
- `install.preferBrew`：为 true 时，如果 `brew`
  可用，则优先使用 Homebrew 安装程序，然后才回退到其他类型的安装程序。
- `install.nodeManager`：`metadata.openclaw.install`
  规范的 Node 安装程序偏好（`npm` | `pnpm` | `yarn` | `bun`）。
- `install.allowUploadedArchives`：允许受信任的 `operator.admin` Gateway 网关
  客户端安装通过 `skills.upload.*` 暂存的私有 ZIP 归档
  （默认值：false）。这只会启用上传归档路径；常规 ClawHub
  安装不需要此设置。
- `entries.<skillKey>.enabled: false` 会禁用技能，即使它已内置或已安装。
- `entries.<skillKey>.apiKey`：为声明主环境变量的技能提供的便捷设置（纯文本字符串或 SecretRef 对象）。
- `limits.maxCandidatesPerRoot`、`limits.maxSkillsLoadedPerSource`、`limits.maxSkillsInPrompt`、`limits.maxSkillsPromptChars`、`limits.maxSkillFileBytes`：限制技能发现和面向模型的技能提示词。
- Skill Workshop 自主性/审批设置（`workshop.autonomous.enabled`、`workshop.approvalPolicy`、`workshop.maxPending`、`workshop.maxSkillBytes`）记录在 [Skills 配置](/zh-CN/tools/skills-config)中。

---

## 插件

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-plugin"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- 从 `~/.openclaw/extensions` 和 `<workspace>/.openclaw/extensions` 下的软件包或捆绑包目录，以及 `plugins.load.paths` 中列出的文件或目录加载。
- 将独立插件文件放入 `plugins.load.paths`；自动发现的扩展根目录会忽略顶层的 `.js`、`.mjs` 和 `.ts` 文件，因此这些根目录中的辅助脚本不会阻止启动。
- 设备发现支持原生 OpenClaw 插件、兼容的 Codex 捆绑包和 Claude 捆绑包，包括没有清单且采用 Claude 默认布局的捆绑包。
- **配置更改需要重启 Gateway 网关。**
- `allow`：可选允许列表（仅加载其中列出的插件）。`deny` 优先。
- `plugins.entries.<id>.apiKey`：插件级 API key 便捷字段（插件支持时）。
- `plugins.entries.<id>.env`：插件作用域的环境变量映射。
- `plugins.entries.<id>.hooks.allowPromptInjection`：当 `false` 时，核心会阻止修改提示词的钩子，例如 `before_prompt_build`。适用于原生插件钩子以及受支持的捆绑包所提供的钩子目录。
- `plugins.entries.<id>.hooks.allowConversationAccess`：当 `true` 时，受信任的非内置插件可以通过 `llm_input`、`llm_output`、`before_model_resolve`、`before_agent_reply`、`before_agent_run`、`before_agent_finalize` 和 `agent_end` 等类型化钩子读取原始对话内容。
- `plugins.entries.<id>.subagent.allowModelOverride`：明确允许此插件为后台子智能体运行请求每次运行的 `provider` 和 `model` 覆盖。
- `plugins.entries.<id>.subagent.allowedModels`：受信任子智能体覆盖可使用的规范 `provider/model` 目标的可选允许列表。仅当你明确希望允许任意模型时才使用 `"*"`。
- `plugins.entries.<id>.llm.allowModelOverride`：明确允许此插件为 `api.runtime.llm.complete` 请求模型覆盖。
- `plugins.entries.<id>.llm.allowedModels`：受信任插件 LLM 补全覆盖可使用的规范 `provider/model` 目标的可选允许列表。仅当你明确希望允许任意模型时才使用 `"*"`。
- `plugins.entries.<id>.llm.allowAgentIdOverride`：明确允许此插件针对非默认 Agent ID 运行 `api.runtime.llm.complete`。
- `plugins.entries.<id>.config`：插件定义的配置对象（在可用时由原生 OpenClaw 插件 schema 验证）。
- 渠道插件的账户/运行时设置位于 `channels.<id>` 下，应由所属插件清单中的 `channelConfigs` 元数据描述，而不是由 OpenClaw 中央选项注册表描述。

### Codex harness 插件配置

内置的 `codex` 插件负责
`plugins.entries.codex.config` 下的原生 Codex app-server harness 设置。完整配置
范围请参阅 [Codex harness reference](/zh-CN/plugins/codex-harness-reference)，运行时模型请参阅 [Codex harness](/zh-CN/plugins/codex-harness)。

`codexPlugins` 仅适用于选择原生 Codex harness 的会话。
它不会为 OpenClaw 提供商运行、ACP
对话绑定或任何非 Codex harness 启用 Codex 插件。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
                allow_destructive_actions: false,
              },
            },
          },
        },
      },
    },
  },
}
```

- `plugins.entries.codex.config.codexPlugins.enabled`：为 Codex harness 启用原生 Codex
  插件/应用支持。默认值：`false`。
- `plugins.entries.codex.config.codexPlugins.allow_all_plugins`：在每个新的原生 Codex 线程中，公开当前已通过身份验证的 Codex 账户所连接的所有可访问应用。默认值：`false`。
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions`：
  已配置插件应用请求的默认破坏性操作策略。
  使用 `true` 可在不提示的情况下接受安全的 Codex 审批 schema，使用 `false`
  可拒绝这些请求，使用 `"auto"` 可通过 OpenClaw
  插件审批转发 Codex 要求的审批，使用 `"ask"` 则会针对每个插件写入/破坏性
  操作进行提示，且不会持久保存审批。`"ask"` 模式会清除受影响应用的持久化 Codex
  单工具审批覆盖，并在 Codex 线程启动前为该应用选择人工
  审批审查者。
  默认值：`true`。
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.enabled`：当全局 `codexPlugins.enabled` 也为 true 时，启用
  已配置的插件条目。
  显式条目的默认值：`true`。
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.marketplaceName`：
  稳定的 marketplace 标识；每个已解析条目都必须同时提供此项和 `pluginName`。
  支持 `"openai-curated"` 和 `"workspace-directory"`。缺少任一标识字段的条目
  将被忽略。
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.pluginName`：稳定的
  Codex 插件标识，必须与 `marketplaceName` 一起提供。`workspace-directory`
  条目必须使用 `plugin/list` 返回的、带确切 marketplace 限定的
  `summary.id`，例如
  `"example-plugin@workspace-directory"`。
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.allow_destructive_actions`：
  每插件破坏性操作覆盖。省略时使用全局
  `allow_destructive_actions` 值。每插件值支持相同的
  `true`、`false`、`"auto"` 或 `"ask"` 策略。

每个使用 `"ask"` 的已准入插件应用都会将该应用的审批请求
转发给人工审查者。其他应用和非应用线程审批仍使用其
已配置的审查者，因此混合插件策略不会继承 `"ask"` 行为。

`codexPlugins.enabled` 是全局启用指令。由迁移写入的显式插件
条目是持久化的精选安装和修复
资格集合。手动配置的 `workspace-directory` 条目必须已经
安装并启用，且其所属应用必须可访问；OpenClaw
不会安装这些条目或为其进行身份验证。如果 Codex 拒绝显式工作区
目录请求，已启用的工作区条目将通过
`marketplace_missing` 以故障关闭方式失败，而默认目录中的精选条目仍然
可用。不支持 `plugins["*"]`，不存在 `install` 开关，并且
本地 `marketplacePath` 值因特定于主机而有意不设为配置字段。有关 app-server 版本和
就绪要求，请参阅
[Native Codex plugins](/zh-CN/plugins/codex-native-plugins)。

`app/list` 就绪检查会缓存一小时，并在过期后
异步刷新。Codex 线程应用配置在 Codex harness
会话建立时计算，而不是在每个轮次中计算；更改原生插件配置后，请使用 `/new`、`/reset` 或重启 Gateway
网关。

`codexPlugins.allow_all_plugins` 会将当前所有可访问的账户
应用快照到每个新的原生 Codex 线程中。它不会安装插件或应用，
无法访问的应用仍会被排除。账户应用使用全局
`codexPlugins.allow_destructive_actions` 策略。当同一应用同时存在于两条路径中时，显式插件条目
优先。如果无法读取 `app/list`，
账户范围的公开将以故障关闭方式失败。

- `plugins.entries.firecrawl.config.webFetch`：Firecrawl 网页抓取提供商设置。
  - `apiKey`：用于提高限额的可选 Firecrawl API key（接受 SecretRef）。回退到 `plugins.entries.firecrawl.config.webSearch.apiKey` 或 `FIRECRAWL_API_KEY` 环境变量。
  - `baseUrl`：Firecrawl API 基础 URL（默认值：`https://api.firecrawl.dev`；自托管覆盖必须指向私有/内部端点）。
  - `onlyMainContent`：仅从页面中提取主要内容（默认值：`true`）。
  - `maxAgeMs`：最大缓存时长，以毫秒为单位（默认值：`172800000` / 2 天）。
  - `timeoutSeconds`：抓取请求超时时间，以秒为单位（默认值：`60`）。
- `plugins.entries.xai.config.xSearch`：xAI X Search（Grok Web 搜索）设置。
  - `enabled`：启用 X Search 提供商。
  - `model`：用于搜索的 Grok 模型（例如 `"grok-4.3"`）。
- `plugins.entries.memory-core.config.dreaming`：记忆 Dreaming 设置。有关阶段和阈值，请参阅 [Dreaming](/zh-CN/concepts/dreaming)。
  - `enabled`：Dreaming 总开关（默认值：`false`）。
  - `frequency`：每次完整 Dreaming 扫描的 cron 周期（默认值为 `"0 3 * * *"`）。
  - `model`：可选的 Dream Diary 子智能体模型覆盖。需要 `plugins.entries.memory-core.subagent.allowModelOverride: true`；可与 `allowedModels` 配合使用以限制目标。模型不可用错误会使用会话默认模型重试一次；信任或允许列表失败不会静默回退。
  - 阶段策略和阈值属于实现细节（不是面向用户的配置键）。
- 完整记忆配置位于[记忆配置参考](/zh-CN/reference/memory-config)：
  - `memory.search.*`
  - `agents.entries.*.memory.search.*` 用于按 Agent 覆盖
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- 已启用的 Claude 捆绑包插件还可以从 `settings.json` 提供嵌入式 OpenClaw 默认值；OpenClaw 会将其作为经过清理的 Agent 设置应用，而不是作为原始 OpenClaw 配置补丁。
- `plugins.slots.memory`：选择活跃的记忆插件 ID，或使用 `"none"` 禁用记忆插件。
- `plugins.slots.contextEngine`：选择活跃的上下文引擎插件 ID；除非安装并选择其他引擎，否则默认为 `"legacy"`。

请参阅[插件](/zh-CN/tools/plugin)。

---

## 浏览器

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // 仅在需要可信私有网络访问时选择启用
      // allowPrivateNetwork: true, // 旧版别名
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    tabCleanup: {
      enabled: true,
      idleMinutes: 120,
      maxTabsPerSession: 8,
      sweepMinutes: 5,
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    // attachOnly: false,
  },
}
```

- `evaluateEnabled: false` 会禁用 `act:evaluate` 和 `wait --fn`。
- `tabCleanup` 控制尽力而为的定期清理：当跟踪的主智能体
  标签页空闲一段时间后，或会话超过其上限时执行清理。仅跟踪由浏览器工具
  `action: "open"` 创建的标签页；用户打开或归属未知的标签页绝不会被接管。
  禁用 `tabCleanup` 不会禁用显式的会话生命周期清理。
- 使用稳定原生 CDP 目标和浏览器身份的主机本地打开记录
  存储在共享 SQLite 状态中，并在 Gateway 网关重启后仍可用于
  `/new` 和会话生命周期清理。面向原生工具的 CDP 目标在重启后
  也仍可进行空闲和上限清理。Chrome MCP 使用进程本地目标句柄，因此冷启动的
  现有会话记录会等待生命周期清理，而不会冒险对重启后无法归属的活动执行
  空闲清理。OpenClaw 会在关闭前验证配置文件和浏览器实例。
  Chrome MCP 自动连接、缺少 `/json/version` 浏览器身份以及无法解析的原生目标
  仍完全由进程本地管理，因此重启后不会自动关闭。较早的未跟踪标签页需要
  手动关闭。暂时性失败会保持待处理状态，以便稍后重试。请参阅
  [标签页清理所有权](/zh-CN/tools/browser#tab-cleanup-ownership)。
- 未设置 `ssrfPolicy.dangerouslyAllowPrivateNetwork` 时，该功能处于禁用状态，因此浏览器导航默认保持严格模式。
- 仅当你有意信任专用网络浏览器导航时，才设置 `ssrfPolicy.dangerouslyAllowPrivateNetwork: true`。
- 在严格模式下，远程 CDP 配置文件端点（`profiles.*.cdpUrl`）在可达性/发现检查期间也受相同的专用网络阻止规则约束。
- `ssrfPolicy.allowPrivateNetwork` 仍作为旧版别名受到支持。
- 在严格模式下，使用 `ssrfPolicy.hostnameAllowlist` 和 `ssrfPolicy.allowedHostnames` 设置显式例外。
- 远程配置文件只能附加（启动/停止/重置均已禁用）。
- `profiles.*.cdpUrl` 接受 `http://`、`https://`、`ws://` 和 `wss://`。
  如果希望 OpenClaw 发现 `/json/version`，请使用 HTTP(S)；如果提供商给出的是
  直接 DevTools WebSocket URL，请使用 WS(S)。
- 如果可通过 loopback 访问外部管理的 CDP 服务，请设置该配置文件的
  `attachOnly: true`；否则，OpenClaw 会将 loopback 端口视为本地托管的浏览器配置文件，
  并可能报告本地端口所有权错误。
- `existing-session` 配置文件使用 Chrome MCP 而非 CDP，并且可在
  所选主机上或通过已连接的浏览器节点进行附加。
- `existing-session` 配置文件可设置 `userDataDir`，以指定特定的
  Chromium 浏览器配置文件，例如 Brave 或 Edge。
- 当 Chrome 已在 DevTools HTTP(S) 发现端点或直接 WS(S) 端点后运行时，
  `existing-session` 配置文件可设置 `cdpUrl`。在该模式下，OpenClaw 会将端点
  传递给 Chrome MCP，而不是使用自动连接；Chrome MCP 启动参数会忽略
  `userDataDir`。
- `existing-session` 配置文件保留当前 Chrome MCP 路由限制：
  使用快照/引用驱动的操作，而不是 CSS 选择器定位；仅支持单文件上传
  钩子；不支持对话框超时覆盖；不支持 `wait --load networkidle`；也不支持
  `responsebody`、PDF 导出、下载拦截或批量操作。
- 本地托管的 `openclaw` 配置文件会自动分配 `cdpPort` 和 `cdpUrl`；
  仅对远程 CDP 配置文件或现有会话端点附加显式设置 `cdpUrl`。
- 本地托管的配置文件可设置 `executablePath`，以覆盖该配置文件的全局
  `browser.executablePath`。可用此方式让一个配置文件在 Chrome 中运行，另一个在 Brave 中运行。
- 自动检测顺序：默认浏览器（如果基于 Chromium）→ Chrome → Brave → Edge → Chromium → Chrome Canary。
- `browser.executablePath` 和 `browser.profiles.<name>.executablePath`
  都接受 `~` 和 `~/...`，并在启动 Chromium 前将其解析为操作系统主目录。
  `existing-session` 配置文件中按配置文件设置的 `userDataDir` 也会展开波浪号。
- 控制服务：仅限 loopback（端口由 `gateway.port` 派生，默认值为 `18791`）。
- `extraArgs` 会向本地 Chromium 启动过程追加额外的启动标志（例如
  `--disable-gpu`、窗口大小设置或调试标志）。

---

## 用户界面

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // 表情符号、短文本、图片 URL 或数据 URI
    },
    prefs: {
      theme: "claw", // claw | knot | dash | custom
      themeMode: "system", // light | dark | system
      locale: "en",
      chatShowThinking: true,
      chatShowToolCalls: true,
      chatPersistCommentary: true, // 运行结束后在 Control UI 中保留评注；不会将其发送到渠道
      chatSendShortcut: "enter", // enter | modifier-enter
      chatFollowUpMode: "steer", // steer | queue；省略时使用服务器队列模式
      showAdvancedSettings: false, // 展开设置中的所有高级组
    },
  },
}
```

- `seamColor`：原生应用界面装饰的强调色（Talk 模式气泡色调等）。
- `assistant`：Control UI 身份覆盖。未设置时回退到活跃智能体身份。
- `prefs`：跨设备操作员偏好设置。这是规范的存储位置，使智能体能够
  通过审批门控更改这些设置，并让每个 Control UI 客户端保持同步；浏览器会将这些值镜像到
  本地存储以实现即时启动，并在无法写入配置时（仅查看者权限范围、离线）保留设备本地副本。
  `chatPersistCommentary` 默认为 `true`。将其设置为 `false` 会在运行期间
  保持实时评注可见，但会在完成时将其移除，并阻止新的 Codex 评注进入持久化转录镜像。
  消息渠道的发送仍独立处理且保持不变。
  `showAdvancedSettings` 默认为 `false`；设置搜索可能会临时打开一个匹配的高级组，
  而不会更改此偏好设置。
  纯呈现偏好设置（例如文本缩放、聊天宽度和实时侧边栏活动）仍保留在浏览器本地，
  并在设置中进行配置。
  已连接的客户端会实时应用服务器端变更：每次持久化配置写入后，Gateway 网关都会广播
  一个仅含哈希的 `config.changed` 事件，客户端随后刷新其快照（当本地设置草稿存在
  未保存的编辑时跳过）。重新连接的客户端会在连接时进行协调。

---

## Gateway 网关

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // 或 OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // 用于 mode=trusted-proxy；请参阅 /gateway/trusted-proxy-auth
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },
    tailscale: {
      mode: "off", // off | serve | funnel
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // toolTitles: false, // 选择启用由 AI 为工具调用生成的用途标题（会消耗实用模型令牌）
      // embedSandbox: "scripts", // strict | scripts | trusted
      // allowExternalEmbedUrls: false, // 危险：允许绝对外部 http(s) 嵌入 URL
      // allowedOrigins: ["https://control.example.com"], // 非 loopback Control UI 必须设置
      // dangerouslyAllowHostHeaderOriginFallback: false, // 危险的 Host 标头来源回退模式
    },
    terminal: {
      enabled: false,
      // shell: "/bin/zsh",
    },
    remote: {
      url: "ws://127.0.0.1:18789",
      transport: "ssh", // ssh | direct
      token: "your-token",
      // password: "your-password",
    },
    trustedProxies: ["10.0.0.1"],
    // 可选。默认为 false。
    allowRealIpFallback: false,
    nodes: {
      pairing: {
        // 可选。默认未设置/禁用。
        autoApproveCidrs: ["192.168.1.0/24", "fd00:1234:5678::/64"],
        // 通过 SSH 验证的自动审批。默认：启用（true）。
        // 设置为 false 只会禁用 SSH 验证；不会影响
        // 上面的 autoApproveCidrs。若仅允许手动节点配对，请设置为 false，并且
        // 不设置 autoApproveCidrs。可传入对象进行调整：{ user, identity,
        // timeoutMs, cidrs }。
        sshVerify: true,
      },
      commands: {
        allow: ["canvas.navigate"],
        deny: ["system.run"],
      },
    },
    tools: {
      // 额外拒绝通过 /tools/invoke HTTP 调用的工具
      deny: ["browser"],
      // 对所有者/管理员调用方，从默认 HTTP 拒绝列表中移除工具
      allow: ["gateway"],
    },
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },
  },
}
```

<Accordion title="Gateway 网关字段详情">

- `mode`：`local`（运行 Gateway 网关）或 `remote`（连接到远程 Gateway 网关）。除非 `local`，否则 Gateway 网关会拒绝启动。
- `port`：用于 WS + HTTP 的单一多路复用端口。优先级：`--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`。
- `bind`：`auto`、`loopback`（默认）、`lan`（`0.0.0.0`）、`tailnet`（可用时使用 Tailscale IPv4，否则使用环回地址），或 `custom`（一个 IPv4 地址）。对于同一主机上的客户端，解析后的 `tailnet` 地址，以及除 `127.0.0.1` 或 `0.0.0.0` 之外的任何 `custom` 地址，都要求在同一端口上使用 `127.0.0.1`；如果任一监听器无法绑定，启动将失败。非环回暴露仍仅限于所选接口。
- **旧版绑定别名**：在 `gateway.bind` 中使用绑定模式值（`auto`、`loopback`、`lan`、`tailnet`、`custom`），而不是主机别名（`0.0.0.0`、`127.0.0.1`、`localhost`、`::`、`::1`）。
- **Docker 注意事项**：默认的 `loopback` 绑定会在容器内监听 `127.0.0.1`。使用 Docker 桥接网络（`-p 18789:18789`）时，流量会到达 `eth0`，因此无法访问 Gateway 网关。请使用 `--network host`，或设置 `bind: "lan"`（或者将 `bind: "custom"` 与 `customBindHost: "0.0.0.0"` 配合使用）以监听所有接口。
- **身份验证**：默认必需。非环回绑定需要 Gateway 网关身份验证。实际而言，这意味着需要共享令牌/密码，或配合 `gateway.auth.mode: "trusted-proxy"` 使用可感知身份的反向代理。新手引导向导默认会生成令牌。
- 如果同时配置了 `gateway.auth.token` 和 `gateway.auth.password`（包括 SecretRef），请将 `gateway.auth.mode` 显式设置为 `token` 或 `password`。如果两者均已配置但未设置模式，启动以及服务安装/修复流程将失败。
- `gateway.auth.mode: "none"`：显式无身份验证模式。仅用于可信的本地 local loopback 设置；新手引导提示特意不提供此选项。
- `gateway.auth.mode: "trusted-proxy"`：将浏览器/用户身份验证委托给可感知身份的反向代理，并信任来自 `gateway.trustedProxies` 的身份标头（参阅[可信代理身份验证](/zh-CN/gateway/trusted-proxy-auth)）。默认情况下，此模式要求代理来源为**非环回地址**；同一主机上的环回反向代理需要显式设置 `gateway.auth.trustedProxy.allowLoopback = true`。同一主机上的内部调用方可以使用 `gateway.auth.password` 作为本地直接回退；`gateway.auth.token` 仍与可信代理模式互斥。
- `gateway.auth.allowTailscale`：当 `true` 时，Tailscale Serve 身份标头可满足 Control UI/WebSocket 身份验证（通过 `tailscale whois` 验证）。HTTP API 端点**不会**使用该 Tailscale 标头身份验证；它们改为遵循 Gateway 网关的常规 HTTP 身份验证模式。此无令牌流程假定 Gateway 网关主机可信。当 `tailscale.mode = "serve"` 时，默认为 `true`。
- `gateway.auth.rateLimit`：可选的身份验证失败限流器。按客户端 IP 和身份验证范围应用（共享密钥和设备令牌分别跟踪）。被阻止的尝试会返回 `429` + `Retry-After`。
  - 在异步 Tailscale Serve Control UI 路径上，同一 `{scope, clientIp}` 的失败尝试会在写入失败结果前串行处理。因此，来自同一客户端的并发错误尝试可能会在第二个请求时触发限流器，而不是两个请求都在竞态中仅作为普通不匹配通过。
  - `gateway.auth.rateLimit.exemptLoopback` 默认为 `true`；仅当确实希望 localhost 流量也受到速率限制时（用于测试设置或严格的代理部署），才设置 `false`。
- 浏览器来源的 WS 身份验证尝试始终会受到限流，且禁用环回豁免（作为针对基于浏览器的 localhost 暴力破解的纵深防御）。
- 在环回地址上，这些浏览器来源的锁定会按规范化后的 `Origin`
  值隔离，因此来自某个 localhost 来源的重复失败不会自动
  锁定另一个来源。
- `tailscale.mode`：`serve`（仅限 tailnet，环回绑定）或 `funnel`（公开，需要身份验证）。
- `tailscale.serviceName`：Serve 模式的可选 Tailscale Service 名称，例如
  `svc:openclaw`。设置后，OpenClaw 会将其传递给 `tailscale serve
--service`，以便通过命名 Service 暴露 Control UI，而不是
  使用设备主机名。该值必须采用 Tailscale 的 `svc:<dns-label>`
  Service 名称格式；启动时会报告派生的 Service URL。
- `tailscale.preserveFunnel`：当 `true` 且 `tailscale.mode = "serve"` 时，OpenClaw
  会在启动时重新应用 Serve 之前检查 `tailscale funnel status`，如果外部配置的 Funnel 路由已覆盖 Gateway 网关端口，
  则会跳过重新应用。默认为 `false`。
- `controlUi.allowedOrigins`：Gateway 网关 WebSocket 连接的显式浏览器来源允许列表。公开的非环回浏览器来源必须配置此项。来自环回地址、RFC1918/链路本地地址、`.local`、`.ts.net` 或 Tailscale CGNAT 主机的私有同源 LAN/Tailnet UI 加载，无需启用 Host 标头回退即可接受。
- `controlUi.toolTitles`：选择启用由 AI 为 Control UI 聊天中的工具调用生成用途标题。默认值：`false`（工具渲染保持完全确定性，不会进行后台模型调用）。启用后，`chat.toolTitles` 方法会通过标准实用模型路由为复杂调用添加标签——使用智能体的 `utilityModel`（这是操作员决策，与所有实用任务一样，可能会将有限的工具参数发送给所选提供商），或会话提供商声明的小模型默认值（OpenAI → `gpt-5.6-luna`，Anthropic → `claude-haiku-4-5`）——并将结果缓存在每个 Agent 的状态数据库中，因此重复查看绝不会重复计费。与所有其他实用任务一样，`utilityModel: \"\"` 会禁用标题；标题绝不会回退到主模型。
- `controlUi.dangerouslyAllowHostHeaderOriginFallback`：一种危险模式，会为有意依赖 Host 标头来源策略的部署启用 Host 标头来源回退。
- `terminal.enabled`：选择启用管理员范围的操作员终端。默认值：`false`。该终端会在所选 Agent 工作区中启动主机 PTY，继承 Gateway 网关进程环境，并拒绝为配置了 `sandbox.mode: "all"` 的 Agent 启动。仅在可信的操作员部署中启用；更改此项会重启 Gateway 网关并更新 Control UI 内容安全策略。
- `terminal.shell`：可选的 shell 可执行文件。未设置时，OpenClaw 在 Unix 上使用 `$SHELL`，在 Windows 上使用 `%ComSpec%`。
- `terminal.detachedSessionTimeoutSeconds`：终端会话在连接断开（页面重新加载、笔记本电脑休眠）后继续存活的时长；在此期间，可通过 `terminal.attach` 重新连接并重放其近期输出。默认值：`300`。设置 `0` 可在连接断开时立即终止会话。已分离的会话会继续运行其命令，因此在共享或暴露的主机上应缩短此时长。
- `remote.transport`：`ssh`（默认）或 `direct`（ws/wss）。对于 `direct`，在公开主机上，`remote.url` 必须为 `wss://`；明文 `ws://` 仅对环回地址、LAN、链路本地地址、`.local`、`.ts.net` 和 Tailscale CGNAT 主机接受。
- `remote.remotePort`：远程 SSH 主机上的 Gateway 网关端口。默认为 `18789`；当本地隧道端口与远程 Gateway 网关端口不同时使用此项。
- `remote.tlsFingerprint`：远程 `wss://` Gateway 网关的预期 SHA-256 证书指纹。macOS 应用会将其同时应用于操作员/控制连接和配套节点连接。若未显式设置值，macOS 仅会在常规系统信任验证成功后记录首次使用固定值。
- `remote.sshHostKeyPolicy`：macOS SSH 隧道主机密钥策略。`strict` 是默认值，要求密钥已受信任。`openssh` 表示明确选择对托管别名使用有效的 OpenSSH 配置；使用前请检查匹配的用户和系统 SSH 设置。更改目标时，macOS 应用和 `configure-remote` 会将此策略重置为 `strict`，除非再次明确选择启用。
- `gateway.remote.token` / `.password` 是远程客户端凭据字段。它们本身不会配置 Gateway 网关身份验证。
- `gateway.push.apns.relay.baseUrl`：外部 APNs 中继的基础 HTTPS URL，在采用中继的 iOS 构建将注册信息发布到 Gateway 网关后使用。公开的 App Store 构建使用托管的 OpenClaw 中继。自定义中继 URL 必须与特意独立的 iOS 构建/部署路径匹配，并且该路径的中继 URL 指向该中继。
- `gateway.push.apns.relay.timeoutMs`：Gateway 网关向中继发送请求的超时时间（毫秒）。默认为 `10000`。
- 采用中继的注册会委托给特定的 Gateway 网关身份。已配对的 iOS 应用会获取 `gateway.identity.get`，在中继注册中包含该身份，并将注册范围的发送授权转发给 Gateway 网关。其他 Gateway 网关无法复用该已存储的注册。
- `OPENCLAW_APNS_RELAY_BASE_URL` / `OPENCLAW_APNS_RELAY_TIMEOUT_MS`：上述中继配置的临时环境变量覆盖值。
- `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true`：仅供开发使用的逃生机制，用于允许环回 HTTP 中继 URL。生产环境的中继 URL 应保持使用 HTTPS。
- `OPENCLAW_HANDSHAKE_TIMEOUT_MS`：内置的身份验证前 Gateway 网关 WebSocket 握手超时时间的可选环境变量覆盖值。
- `channels.<provider>.healthMonitor.enabled`：在保持全局监视器启用的同时，按渠道选择停用健康监视器重启。
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`：多账户渠道的按账户覆盖值。设置后，其优先级高于渠道级覆盖值。
- 仅当未设置 `gateway.auth.*` 时，本地 Gateway 网关调用路径才能使用 `gateway.remote.*` 作为回退。
- 如果通过 SecretRef 显式配置了 `gateway.auth.token` / `gateway.auth.password` 但无法解析，则解析会以关闭方式失败（不会用远程回退掩盖问题）。
- `trustedProxies`：终止 TLS 或注入转发客户端标头的反向代理 IP。仅列出你控制的代理。对于同一主机上的代理/本地检测设置（例如 Tailscale Serve 或本地反向代理），环回条目仍然有效，但它们**不会**使环回请求具备使用 `gateway.auth.mode: "trusted-proxy"` 的资格。
- `allowRealIpFallback`：当 `true` 时，如果缺少 `X-Forwarded-For`，Gateway 网关会接受 `X-Real-IP`。默认为 `false`，以实现故障时关闭行为。
- `gateway.nodes.pairing.autoApproveCidrs`：可选的 CIDR/IP 允许列表，用于自动批准未请求权限范围的首次节点设备配对。未设置时禁用。此项不会自动批准操作员/浏览器/Control UI/WebChat 配对，也不会自动批准角色、权限范围、元数据或公钥升级。
- `gateway.nodes.pairing.sshVerify`：通过 SSH 验证的首次节点设备配对自动批准（默认：启用）。Gateway 网关会通过 SSH 回连配对主机（BatchMode，严格主机密钥检查），并且仅在 `openclaw node identity` 设备密钥完全匹配时才批准。资格下限与 `autoApproveCidrs` 相同；除非 `cidrs` 覆盖，否则探测仅限于私有/CGNAT 来源地址。设置 `false` 可禁用，或设置 `{ user, identity, timeoutMs, cidrs }` 进行调整。参阅[节点配对](/zh-CN/gateway/pairing#ssh-verified-device-auto-approval-default)。
- `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny`：在完成配对和平台允许列表评估后，对已声明的节点命令进行全局允许/拒绝控制。使用 `commands.allow` 选择启用危险的节点命令，例如 `camera.snap`、`camera.clip`、`screen.record`、`health.summary`、`sms.search` 和 `sms.send`；即使平台默认设置或显式允许原本会包含某个命令，`commands.deny` 也会将其移除。iOS Health 权限、Android SMS 权限和 Gateway 网关命令授权彼此独立。节点更改其声明的命令列表后，请拒绝该设备配对并重新批准，以便 Gateway 网关存储更新后的命令快照。
- `gateway.tools.deny`：针对 HTTP `POST /tools/invoke` 额外阻止的工具名称（扩展默认拒绝列表）。
- `gateway.tools.allow`：从面向所有者/管理员调用方的默认 HTTP 拒绝列表中移除工具名称。这不会将携带身份信息的 `operator.write` 调用方提升为所有者/管理员访问权限；即使加入允许列表，非所有者调用方仍无法使用 `cron`、`gateway` 和 `nodes`。

</Accordion>

### OpenAI 兼容端点

- 管理 HTTP RPC：默认关闭，与 `admin-http-rpc` 插件相同。启用该插件以注册 `POST /api/v1/admin/rpc`。请参阅[管理 HTTP RPC](/zh-CN/plugins/admin-http-rpc)。
- Chat Completions：默认禁用。使用 `gateway.http.endpoints.chatCompletions.enabled: true` 启用。
- Responses API：`gateway.http.endpoints.responses.enabled`。
- Responses URL 输入加固：
  - `gateway.http.endpoints.responses.maxUrlParts`
  - `gateway.http.endpoints.responses.files.urlAllowlist`
  - `gateway.http.endpoints.responses.images.urlAllowlist`
    空允许列表会被视为未设置；使用 `gateway.http.endpoints.responses.files.allowUrl=false`
    和/或 `gateway.http.endpoints.responses.images.allowUrl=false` 禁用 URL 获取。
- 可选的响应加固标头：
  - `gateway.http.securityHeaders.strictTransportSecurity`（仅为你控制的 HTTPS 来源设置；请参阅[受信任代理身份验证](/zh-CN/gateway/trusted-proxy-auth#tls-termination-and-hsts)）

### 多实例隔离

使用唯一端口和状态目录在一台主机上运行多个 Gateway 网关：

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

便捷标志：`--dev`（使用 `~/.openclaw-dev` + 端口 `19001`）、`--profile <name>`（使用 `~/.openclaw-<name>`）。

请参阅[多个 Gateway 网关](/zh-CN/gateway/multiple-gateways)。

### `gateway.tls`

```json5
{
  gateway: {
    tls: {
      enabled: false,
      autoGenerate: false,
      certPath: "/etc/openclaw/tls/server.crt",
      keyPath: "/etc/openclaw/tls/server.key",
      caPath: "/etc/openclaw/tls/ca-bundle.crt",
    },
  },
}
```

- `enabled`：在 Gateway 网关监听器处启用 TLS 终止（HTTPS/WSS）（默认值：`false`）。
- `autoGenerate`：未配置显式文件时，自动生成本地自签名证书/密钥对；仅供本地/开发使用。
- `certPath`：TLS 证书文件的文件系统路径。
- `keyPath`：TLS 私钥文件的文件系统路径；请严格限制权限。
- `caPath`：用于客户端验证或自定义信任链的可选 CA 证书包路径。

### `gateway.reload`

```json5
{
  gateway: {
    reload: {
      mode: "hybrid", // off | restart | hot | hybrid
      debounceMs: 500,
      deferralTimeoutMs: 300000,
    },
  },
}
```

- `mode`：控制如何在运行时应用配置编辑。
  - `"off"`：忽略实时编辑；更改需要显式重启。
  - `"restart"`：配置更改时始终重启 Gateway 网关进程。
  - `"hot"`：在进程内应用更改，无需重启。
  - `"hybrid"`（默认值）：先尝试热重载；必要时回退到重启。
- `debounceMs`：应用配置更改前的防抖窗口，以 ms 为单位（非负整数；默认值：`300`）。
- `deferralTimeoutMs`：在强制重启或渠道热重载之前，等待进行中操作完成的可选最长时间，以 ms 为单位。省略时使用默认的有限等待时间（`300000`）；设为 `0` 可无限期等待，并定期记录仍在等待的警告。

---

## 云端工作节点环境

云端工作节点为选择启用。如果缺少 `cloudWorkers`，或 `profiles` 为空，OpenClaw 不接受创建任何新工作节点。此前创建的持久记录仍会进行协调并保持可见；现有 Gateway 网关/节点投影保持不变。

每个工作节点提供商都必须从受信任的预配输出中返回 SSH `hostKey`，其格式必须恰好为 `algorithm base64`，且不含主机名或注释。引导程序会将该密钥写入隔离的 `known_hosts` 文件，使用 `StrictHostKeyChecking=yes`，并在提供商未提供密钥时于建立连接前失败。不提供首次使用时信任的回退机制。

隧道按需设置，而不是预配的一部分。启动后，Gateway 网关会将工作节点本地的 Unix 套接字反向转发到其 loopback WebSocket 端点。该套接字位于随机分配且仅所有者可访问的远程目录中；与 loopback TCP 端口不同，多用户工作节点上的其他账户无法访问它，并且它不会与其他环境的端口冲突。仅当隧道所有者仍为当前所有者时，SSH 保活和有上限的重连退避才会运行。停止隧道时，会先阻止重新连接，再关闭 SSH 进程。

控制流量和工作区传输使用独立的 SSH 连接。两者复用同一已解析身份和隔离、固定的 `known_hosts` 文件，但工作区传输不会与长期运行的隧道共享 SSH 连接复用，因此 rsync 无法阻塞控制流量。

### Crabbox 配置文件

内置的 `crabbox` 提供商通过本地 Crabbox CLI 预配支持 SSH 的租约。内部的 `settings.provider` 用于选择 Crabbox 后端；它与外部 OpenClaw 提供商 ID 相互独立。

```json5
{
  cloudWorkers: {
    profiles: {
      production: {
        provider: "crabbox",
        install: "bundle", // Default; use "npm" only for a released gateway version.
        settings: {
          provider: "aws",
          class: "standard",
          ttl: "24h",
          idleTimeout: "60m",
          // Optional absolute path. Default: sibling ../crabbox/bin/crabbox, then PATH.
          binary: "/usr/local/bin/crabbox",
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `settings.provider`（必需）：通过 `--provider` 传递的 Crabbox 后端。请使用其检查输出包含 SSH 端点的后端；`aws` 选择直接 AWS 后端。
- `settings.class`（必需）：传递给 `--class` 的 Crabbox 机器类别。
- `settings.ttl` 和 `settings.idleTimeout`（必需）：传递给 `--ttl` 和 `--idle-timeout` 的正值 Go 时长字符串。这些提供商侧故障保护措施与下方存储的 OpenClaw `lifetime` 策略不同。
- `settings.binary`：可选的 Crabbox 可执行文件绝对路径。如果未设置，OpenClaw 会依次检查同级 Crabbox 检出、`PATH` 中的可执行条目，最后调用 `crabbox`，从而使缺失 CLI 仍表现为可见的提供商错误。

未知设置将被拒绝。Crabbox 凭据和后端特定账户配置仍由 Crabbox 管理；不要将它们放入 `settings`。OpenClaw 仅调用本地 CLI，此插件不会发起提供商网络调用。预配时始终传递 `--keep=true`；OpenClaw 管理外部生命周期，并使用 `crabbox stop` 销毁租约。

<Note>
  OpenClaw 通过提供商拥有的密钥解析器解析 Crabbox 租约本地的 `sshKey` 路径，并固定 `crabbox inspect --json` 返回的权威 `sshHostKey`。AWS 准入还要求 `providerMetadata.instanceProfileAttached`。请安装 Crabbox 0.38.1 或更高版本，以使用此封闭式检查契约。
</Note>

### 静态 SSH 开发配置文件

```json5
{
  cloudWorkers: {
    profiles: {
      development: {
        provider: "static-ssh",
        settings: {
          host: "worker.example.test",
          port: 22,
          user: "openclaw",
          hostKey: "ssh-ed25519 <base64-public-host-key>",
          keyRef: {
            source: "env",
            provider: "default",
            id: "OPENCLAW_WORKER_SSH_KEY",
          },
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `profiles`：具有非空且已去除首尾空白字符 ID 的命名工作节点配置文件。每个配置文件选择一个由插件注册的提供商。
- `provider`：非空工作节点提供商 ID。示例使用内置的 `crabbox` 提供商和 QA Lab `static-ssh` 提供商。
- `install`：工作节点安装方式。`"bundle"`（默认值）传输 Gateway 网关已安装构建的内容哈希包，并支持已发布、开发中和尚未发布的版本。`"npm"` 是针对未经修改的打包发行版的选择启用优化；它从公共 npm 注册表安装 `openclaw@<exact gateway version>`，且绝不会安装 `latest`。
- 配置后会自动选择内置提供商插件，但显式禁用和 `plugins.allow` 仍然生效。配置允许列表时，请包含提供商 ID（例如 `crabbox`）。外部提供商插件还必须安装并显式启用。
- `settings`：由提供商管理且有界的 JSON。所选插件定义并验证其键；对于包含密钥的值，请使用 [SecretRef 对象](/zh-CN/gateway/secrets)。静态 SSH 提供商要求 `host`、`user`、`hostKey` 和 `keyRef`；`port` 默认为 `22`。`hostKey` 必须是从已知主机或其他受信任渠道获取的一行 OpenSSH 公共主机密钥（`algorithm base64`），且不含选项前缀。
- `lifetime.idleTimeoutMinutes`：为后续空闲回收策略存储的正整数分钟数。
- `lifetime.maxLifetimeMinutes`：为后续生命周期策略存储的正整数分钟数。

工作节点上必须已安装受支持的 Node 运行时（22.22.3+、24.15+ 或 25.9+）以及可安全重置 WAL 的 SQLite。选择启用的 `"npm"` 方法还要求 `npm`，并且能够通过出站 HTTPS 访问公共 npm 注册表。联网工具链设置属于提供商策略；引导程序会报告可操作的错误，而不是自行安装工具链。

此基础功能会安装并验证 Gateway 网关构建，并提供隧道启动/停止生命周期，但不会启动通用 OpenClaw CLI。自包含的工作节点入口和循环将在下一个云端工作节点里程碑中实现。

每条持久环境记录都会在创建时的配置文件快照中保留其已验证的提供商设置、已解析的安装方式和生命周期策略。更改或删除命名配置文件会影响新创建操作；只要归属插件仍然可用，现有记录将继续使用该快照进行生命周期协调。

在首个云端工作节点版本中，生命周期值仅作为数据使用；自动执行将在后续生命周期工作中实现。配置文件更改需要重启 Gateway 网关。

<Warning>
  `static-ssh` 提供商是源代码树中的 QA Lab 开发测试工具，不包含在打包发行版中。运行在其共享主机上的工作节点可以读取不相关的主机数据，因此不要将此提供商用作生产隔离边界。
  其操作员必须提供预期的 `hostKey`；OpenClaw 不会在首次连接时获取或接受密钥。
  销毁其租约只会释放 OpenClaw 的逻辑记录；不会停止或清理主机。
</Warning>

---

## Hooks

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: true,
    allowedSessionKeyPrefixes: ["hook:", "hook:gmail:"],
    allowedAgentIds: ["hooks", "main"],
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        agentId: "hooks",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "From: {{messages[0].from}}\nSubject: {{messages[0].subject}}\n{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

身份验证：`Authorization: Bearer <token>` 或 `x-openclaw-token: <token>`。
查询字符串中的 Hook 令牌会被拒绝。

验证和安全说明：

- `hooks.enabled=true` 要求 `hooks.token` 非空。
- `hooks.token` 应与当前 Gateway 网关共享密钥身份验证（`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` 或 `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`）区分开；启动时若检测到重复使用，会记录一条非致命安全警告。
- `openclaw security audit` 会将钩子/Gateway 网关身份验证重复使用标记为严重发现，包括仅在审计时提供的 Gateway 网关密码身份验证（`--auth password --password <password>`）。运行 `openclaw doctor --fix` 轮换已持久化且被重复使用的 `hooks.token`，然后更新外部钩子发送方以使用新的钩子令牌。
- `hooks.path` 不能是 `/`；请使用专用子路径，例如 `/hooks`。
- 如果 `hooks.allowRequestSessionKey=true`，请限制 `hooks.allowedSessionKeyPrefixes`（例如 `["hook:"]`）。
- 如果映射或预设使用模板化的 `sessionKey`，请设置 `hooks.allowedSessionKeyPrefixes` 和 `hooks.allowRequestSessionKey=true`。静态映射键不需要此项显式启用。

**端点：**

- `POST /hooks/wake` → `{ text, mode?: "now"|"next-heartbeat" }`
- `POST /hooks/agent` → `{ message, name?, agentId?, sessionKey?, wakeMode?, deliver?, channel?, to?, model?, thinking?, timeoutSeconds? }`
  - 仅当 `hooks.allowRequestSessionKey=true` 时，才接受请求负载中的 `sessionKey`（默认值：`false`）。
- `POST /hooks/<name>` → 通过 `hooks.mappings` 解析
  - 模板渲染的映射 `sessionKey` 值被视为由外部提供，因此也需要 `hooks.allowRequestSessionKey=true`。

<Accordion title="映射详情">

- `match.path` 匹配 `/hooks` 后的子路径（例如 `/hooks/gmail` → `gmail`）。
- `match.source` 匹配通用路径的负载字段。
- 类似 `{{messages[0].subject}}` 的模板从负载中读取数据。
- `transform` 可以指向返回钩子操作的 JS/TS 模块。
  - `transform.module` 必须是相对路径，并且不能超出 `hooks.transformsDir`（绝对路径和路径遍历会被拒绝）。
  - 请将 `hooks.transformsDir` 保存在 `~/.openclaw/hooks/transforms` 下；工作区 Skills 目录会被拒绝。如果 `openclaw doctor` 报告此路径无效，请将转换模块移入钩子转换目录，或移除 `hooks.transformsDir`。
- `agentId` 将请求路由至特定智能体；未知 ID 会回退到默认智能体。
- `allowedAgentIds`：限制有效的智能体路由，包括省略 `agentId` 时的默认智能体路径（`*` 或省略 = 全部允许，`[]` = 全部拒绝）。
- `defaultSessionKey`：用于未显式设置 `sessionKey` 的钩子智能体运行的可选固定会话键。
- `allowRequestSessionKey`：允许 `/hooks/agent` 调用方和模板驱动的映射会话键设置 `sessionKey`（默认值：`false`）。
- `allowedSessionKeyPrefixes`：显式 `sessionKey` 值（请求 + 映射）的可选前缀允许列表，例如 `["hook:"]`。当任一映射或预设使用模板化的 `sessionKey` 时，此项为必需。
- `deliver: true` 将最终回复发送至渠道；`channel` 默认为 `last`。
- `model` 为此次钩子运行覆盖 LLM（如果已设置模型目录，则必须允许该模型）。

</Accordion>

### Gmail 集成

- 内置 Gmail 预设使用 `sessionKey: "hook:gmail:{{messages[0].id}}"`。
- 此按消息划分的键用于隔离对话上下文，而非工具或工作区访问权限。如果没有设置 `agentId` 的自定义映射，预设会使用默认智能体。
- 对于不受信任的收件箱，请将 Gmail 路由至专用读取智能体，并通过[按 Agent 配置的沙箱和工具策略](/zh-CN/tools/multi-agent-sandbox-tools)限制该智能体。如果读取智能体必须通知主智能体，请使用 [`tools.agentToAgent`](/zh-CN/gateway/config-tools#toolsagenttoagent) 限制交接。有关推荐的威胁模型和模型层级，请参阅[提示词注入](/zh-CN/gateway/security#prompt-injection)。
- 如果保留这种按消息路由，请设置 `hooks.allowRequestSessionKey: true`，并限制 `hooks.allowedSessionKeyPrefixes` 以匹配 Gmail 命名空间，例如 `["hook:", "hook:gmail:"]`。
- 如果需要 `hooks.allowRequestSessionKey: false`，请使用静态 `sessionKey` 覆盖预设，而不是使用模板化默认值。

```json5
{
  hooks: {
    gmail: {
      account: "openclaw@gmail.com",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

- 配置后，Gateway 网关会在启动时自动启动 `gog gmail watch serve`。设置 `OPENCLAW_SKIP_GMAIL_WATCHER=1` 可禁用。
- 不要在 Gateway 网关旁另行运行 `gog gmail watch serve`。

---

## Canvas 插件主机

```json5
{
  plugins: {
    entries: {
      canvas: {
        config: {
          host: {
            root: "~/.openclaw/workspace/canvas",
            liveReload: true,
            // enabled: false, // 或 OPENCLAW_SKIP_CANVAS_HOST=1
          },
        },
      },
    },
  },
}
```

- 通过 Gateway 网关端口下的 HTTP 提供智能体可编辑的 HTML/CSS/JS 和 A2UI：
  - `http://<gateway-host>:<gateway.port>/__openclaw__/canvas/`
  - `http://<gateway-host>:<gateway.port>/__openclaw__/a2ui/`
- 仅限本地：保留 `gateway.bind: "loopback"`（默认值）。
- 非 local loopback 绑定：与其他 Gateway 网关 HTTP 接口一样，Canvas 路由需要 Gateway 网关身份验证（令牌/密码/可信代理）。
- 节点 WebView 通常不会发送身份验证标头；节点配对并连接后，Gateway 网关会通告用于访问 Canvas/A2UI 的节点范围能力 URL。
- 能力 URL 绑定到活跃的节点 WS 会话，并会迅速过期。不使用基于 IP 的回退。
- 将实时重载客户端注入所提供的 HTML。
- 为空时自动创建初始 `index.html`。
- 还会在 `/__openclaw__/a2ui/` 提供 A2UI。
- 更改需要重启 Gateway 网关。
- 对于大型目录或出现 `EMFILE` 错误时，请禁用实时重载。

---

## 设备发现

### mDNS (Bonjour)

```json5
{
  discovery: {
    mdns: {
      mode: "minimal", // minimal | full | off
    },
  },
}
```

- `minimal`（默认值）：从 TXT 记录中省略 `cliPath` + `sshPort`。
- `full`：包含 `cliPath` + `sshPort`；LAN 多播通告仍要求启用内置的 `bonjour` 插件。
- `off`：在不更改插件启用状态的情况下，禁止 LAN 多播通告。
- 内置的 `bonjour` 插件会在 macOS 主机上自动启动，而在 Linux、Windows 和容器化 Gateway 网关部署中需显式启用。
- 当系统主机名是有效的 DNS 标签时，主机名默认使用系统主机名，否则回退到 `openclaw`。可通过 `OPENCLAW_MDNS_HOSTNAME` 覆盖。
- `OPENCLAW_DISABLE_BONJOUR=1` 会完全禁用 mDNS 通告，并覆盖 `discovery.mdns.mode`。

### 广域（DNS-SD）

```json5
{
  discovery: {
    wideArea: { enabled: true },
  },
}
```

在 `~/.openclaw/dns/` 下写入单播 DNS-SD 区域。对于跨网络设备发现，请将其与 DNS 服务器（推荐 CoreDNS）和 Tailscale 分离 DNS 配合使用。

设置：`openclaw dns setup --apply`。

---

## 环境

### `env`（内联环境变量）

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

- 仅当进程环境中缺少相应键时，才会应用内联环境变量。
- `.env` 文件：CWD `.env` + `~/.openclaw/.env`（两者都不会覆盖现有变量）。
- `shellEnv`：从登录 shell 配置文件导入缺失的预期键名。
- 有关完整优先级，请参阅[环境](/zh-CN/help/environment)。

### 环境变量替换

在任何配置字符串中使用 `${VAR_NAME}` 引用环境变量：

```json5
{
  gateway: {
    auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" },
  },
}
```

- 仅匹配大写名称：`[A-Z_][A-Z0-9_]*`。
- 变量缺失或为空时，会在加载配置时报错。
- 使用 `$${VAR}` 转义，以表示字面量 `${VAR}`。
- 适用于 `$include`。

---

## 密钥

密钥引用是增量功能：纯文本值仍然有效。

### `SecretRef`

使用以下一种对象结构：

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

验证：

- `provider` 模式：`^[a-z][a-z0-9_-]{0,63}$`
- `source: "env"` ID 模式：`^[A-Z][A-Z0-9_]{0,127}$`
- `source: "file"` ID：绝对 JSON 指针（例如 `"/providers/openai/apiKey"`）
- `source: "exec"` ID 模式：`^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$`（支持 AWS 风格的 `secret#json_key` 选择器）
- `source: "exec"` ID 不得包含 `.` 或 `..` 斜杠分隔路径段（例如 `a/../b` 会被拒绝）

### 支持的凭据范围

- 规范矩阵：[SecretRef 凭据范围](/zh-CN/reference/secretref-credential-surface)
- `secrets apply` 面向受支持的 `openclaw.json` 凭据路径。
- `auth-profiles.json` 引用包含在运行时解析和审计覆盖范围内。

### 密钥提供商配置

```json5
{
  secrets: {
    providers: {
      default: { source: "env" }, // 可选的显式环境提供商
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
        timeoutMs: 5000,
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        passEnv: ["PATH", "VAULT_ADDR"],
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

注意：

- `file` 提供商支持 `mode: "json"` 和 `mode: "singleValue"`（在 singleValue 模式下，`id` 必须为 `"value"`）。
- 当 Windows ACL 验证不可用时，文件和 exec 提供商路径会以失败关闭方式处理。仅对无法验证的可信路径设置 `allowInsecurePath: true`。
- `exec` 提供商要求使用绝对 `command` 路径，并通过 stdin/stdout 使用协议负载。
- 默认拒绝符号链接命令路径。设置 `allowSymlinkCommand: true` 可允许符号链接路径，同时验证解析后的目标路径。
- 如果配置了 `trustedDirs`，可信目录检查将应用于解析后的目标路径。
- `exec` 子进程环境默认保持最小化；请使用 `passEnv` 显式传递所需变量。
- 密钥引用会在激活时解析为内存快照，随后请求路径只读取该快照。
- 激活期间会应用活动接口筛选：已启用接口上未解析的引用会导致启动/重新加载失败，而非活动接口会被跳过并生成诊断信息。

---

## 身份验证存储

```json5
{
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:personal": { provider: "openai", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      openai: ["openai:personal"],
    },
  },
}
```

- 每个智能体的配置文件存储在 `<agentDir>/auth-profiles.json`。
- `auth-profiles.json` 支持静态凭据模式的值级引用（`api_key` 使用 `keyRef`，`token` 使用 `tokenRef`）。
- 旧版扁平 `auth-profiles.json` 映射（如 `{ "provider": { "apiKey": "..." } }`）不是运行时格式；`openclaw doctor --fix` 会将它们重写为规范的 `provider:default` API 密钥配置文件，并创建 `.legacy-flat.*.bak` 备份。
- OAuth 模式配置文件（`auth.profiles.<id>.mode = "oauth"`）不支持由 SecretRef 支持的身份验证配置文件凭据。
- 静态运行时凭据来自内存中已解析的快照；发现旧版静态 `auth.json` 条目时会将其清除。
- 旧版 OAuth 从 `~/.openclaw/credentials/oauth.json` 导入。
- 请参阅 [OAuth](/zh-CN/concepts/oauth)。
- 关于密钥运行时行为和 `audit/configure/apply` 工具，请参阅[密钥管理](/zh-CN/gateway/secrets)。

---

## 审计

```json5
{
  audit: {
    enabled: true,
    messages: "off", // 关闭 | 直接对话 | 全部
  },
}
```

Gateway 网关会将智能体运行和工具操作的**仅元数据**审计事件记录到共享状态数据库中。消息生命周期元数据需要单独选择启用。账本会存储身份、时间、工具名称和规范化结果，但绝不会存储提示词、消息正文、工具参数、结果或原始错误文本。消息行不会存储原始的平台账户、对话、消息和目标 ID。运行/工具会话键仍可用于关联，其本身可能包含平台账户或对等方 ID。记录将在 30 天后过期，账本上限为 100,000 行。可使用 [`openclaw audit`](/zh-CN/cli/audit) 或 [`audit.activity.list`](/zh-CN/gateway/protocol#audit-ledger-rpc) Gateway RPC 查询。有关完整的数据模型、隐私语义和覆盖范围限制，请参阅[审计历史记录](/zh-CN/gateway/audit)。

- `enabled`：记录新的审计事件（默认值：`true`）。账本默认启用，因为只有在事件发生后才启用的审计跟踪无法解释该事件。设置 `false` 后，Gateway 网关重启时会停止插入新事件；现有记录在过期前仍可读取。重新启用后会从该时间点恢复记录，不会回填中间的空缺。
- `messages`：消息元数据范围（默认值：`"off"`）。`"direct"` 仅记录已知的直接对话。`"all"` 还会记录群组、频道和未知类型的对话。两种模式均不包含内容，并在能够进行关联时使用安装实例本地的带密钥假名替换原始标识符。这些是关联辅助信息，并非匿名化；状态数据库会存储派生密钥，但 RPC 和 CLI 导出中不会包含该密钥。

运行中的 Gateway 网关会在启动时捕获 `audit.enabled` 和 `audit.messages`；更改任一设置后请重启。目前的消息覆盖范围包括到达核心分派的已接受入站消息，以及到达共享持久传递边界的每个原始逻辑出站回复负载对应的一条终止记录。绕过这些共享边界的插件本地路径和直接发送路径尚未覆盖。这个有界后台写入器采用尽力而为方式，并不是无损的合规归档。

---

## 日志

```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty", // 美观 | 紧凑 | json
    redactSensitive: "tools", // 关闭 | 工具
    redactPatterns: ["\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1"],
  },
}
```

- 默认日志文件：`/tmp/openclaw/openclaw-YYYY-MM-DD.log`；命名配置文件使用 `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`。
- 设置 `logging.file` 可使用稳定路径。
- 当 `--verbose` 时，`consoleLevel` 会提升至 `debug`。
- `maxFileBytes`：轮转前活动日志文件的最大字节数（正整数；默认值：`104857600` = 100 MB）。OpenClaw 会在活动文件旁保留最多五个带编号的归档文件。
- `redactSensitive` / `redactPatterns`：尽力对控制台输出、文件日志、OTLP 日志记录和持久化会话转录文本进行掩码处理。`redactSensitive: "off"` 仅禁用此通用日志/转录策略；UI、工具和诊断安全界面仍会在输出前隐去密钥。

---

## 诊断

```json5
{
  diagnostics: {
    enabled: true,
    flags: ["telegram.*"],

    otel: {
      enabled: false,
      endpoint: "https://otel-collector.example.com:4318",
      tracesEndpoint: "https://traces.example.com/v1/traces",
      metricsEndpoint: "https://metrics.example.com/v1/metrics",
      logsEndpoint: "https://logs.example.com/v1/logs",
      protocol: "http/protobuf", // http/protobuf | grpc
      headers: { "x-tenant-id": "my-org" },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: false,
      logsExporter: "otlp",
      sampleRate: 1.0,
      flushIntervalMs: 5000,
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },

    cacheTrace: {
      enabled: false,
      filePath: "~/.openclaw/logs/cache-trace.jsonl",
      includeMessages: true,
      includePrompt: true,
      includeSystem: true,
    },
  },
}
```

- `enabled`：插桩输出的总开关（默认值：`true`）。
- `flags`：用于启用目标日志输出的标志字符串数组（支持 `"telegram.*"` 或 `"*"` 等通配符）。
- `otel.enabled`：启用 OpenTelemetry 导出管道（默认值：`false`）。有关完整配置、信号目录和隐私模型，请参阅 [OpenTelemetry 导出](/zh-CN/gateway/opentelemetry)。
- `otel.endpoint`：用于 OTel 导出的收集器 URL。
- `otel.tracesEndpoint` / `otel.metricsEndpoint` / `otel.logsEndpoint`：可选的信号专用 OTLP 端点。设置后，它们仅针对相应信号覆盖 `otel.endpoint`。
- `otel.protocol`：`"http/protobuf"`（默认值）或 `"grpc"`。
- `otel.headers`：随 OTel 导出请求发送的额外 HTTP/gRPC 元数据标头。
- `otel.serviceName`：资源属性的服务名称。
- `otel.traces` / `otel.metrics` / `otel.logs`：启用跟踪、指标或日志导出。
- `otel.logsExporter`：日志导出接收端：`"otlp"`（默认值）、每行标准输出一个 JSON 对象的 `"stdout"`，或 `"both"`。
- `otel.sampleRate`：跟踪采样率 `0`-`1`。
- `otel.flushIntervalMs`：周期性遥测刷新间隔，单位为 ms。
- `otel.captureContent`：选择启用将原始内容捕获到 OTEL span 属性中。默认关闭。布尔值 `true` 会捕获非系统消息/工具内容；对象形式允许显式启用 `inputMessages`、`outputMessages`、`toolInputs`、`toolOutputs`、`systemPrompt` 和 `toolDefinitions`。
- `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`：最新实验性 GenAI 推理 span 结构的环境开关，包括 `{gen_ai.operation.name} {gen_ai.request.model}` span 名称、`CLIENT` span 类型，以及使用 `gen_ai.provider.name` 而非旧版 `gen_ai.system`。默认情况下，span 会保留 `openclaw.model.call` 和 `gen_ai.system` 以保持兼容性；GenAI 指标使用有界语义属性。
- `OPENCLAW_OTEL_PRELOADED=1`：适用于已注册全局 OpenTelemetry SDK 的主机的环境开关。此时 OpenClaw 会跳过插件自有 SDK 的启动/关闭，同时保持诊断监听器处于活动状态。
- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`、`OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` 和 `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`：在对应配置键未设置时使用的信号专用端点环境变量。
- `cacheTrace.enabled`：记录嵌入式运行的缓存跟踪快照（默认值：`false`）。
- `cacheTrace.filePath`：缓存跟踪 JSONL 的输出路径（默认值：`$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`）。
- `cacheTrace.includeMessages` / `includePrompt` / `includeSystem`：控制缓存跟踪输出中包含的内容（默认值均为 `true`）。

---

## 更新

```json5
{
  update: {
    channel: "stable", // 稳定版 | 扩展稳定版 | 测试版 | 开发版
    checkOnStart: true,

    auto: {
      enabled: false,
    },
  },
}
```

- `channel`：发布渠道——`"stable"`、`"extended-stable"`、`"beta"` 或 `"dev"`。扩展稳定版仅以软件包形式提供：前台命令负责安装，而 Gateway 网关可以发出只读更新提示。
- `checkOnStart`：Gateway 网关启动时检查 npm 更新（默认值：`true`）。已存储的扩展稳定版选择使用相同的只读提示和 24 小时提示计划。
- `auto.enabled`：为稳定版和测试版软件包安装启用后台自动更新（默认值：`false`）。扩展稳定版绝不会自动应用更新。

---

## ACP

```json5
{
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    fallbacks: ["acpx-secondary"],
    defaultAgent: "main",
    allowedAgents: ["main", "ops"],
    stream: {
      repeatSuppression: true,
      deliveryMode: "live", // 实时 | 仅最终结果
    },
  },
}
```

- `enabled`：全局 ACP 功能开关（默认值：`true`；设置 `false` 可隐藏 ACP 分派和创建入口）。
- `dispatch.enabled`：ACP 会话轮次分派的独立开关（默认值：`true`）。设置 `false` 可在阻止执行的同时保留 ACP 命令。
- `backend`：默认 ACP 运行时后端 ID（必须与已注册的 ACP 运行时插件匹配）。
  请先安装后端插件；如果已设置 `plugins.allow`，请包含后端插件 ID（例如 `acpx`），否则 ACP 后端将无法加载。
- `fallbacks`：备用 ACP 后端 ID 的有序列表。当主后端在产生任何输出前因疑似瞬时错误（不可用、达到速率限制、配额耗尽或过载）而提前失败时，会依次尝试这些后端。每个条目都必须与已注册 ACP 运行时插件的后端匹配。
- `defaultAgent`：创建时未指定显式目标时使用的备用 ACP 目标智能体 ID。
- `allowedAgents`：允许用于 ACP 运行时会话的智能体 ID 白名单；为空表示没有额外限制。
- `stream.repeatSuppression`：抑制每个轮次中重复的状态/工具行（默认值：`true`）。
- `stream.deliveryMode`：`"live"` 采用增量流式传输；`"final_only"` 会缓冲到轮次终止事件发生。
- `stream.tagVisibility`：标签名称到流式事件布尔可见性覆盖值的记录。
- `runtime.installCommand`：引导初始化 ACP 运行时环境时要运行的可选安装命令。

---

## 向导

CLI 引导设置流程（`onboard`、`configure`、`doctor`）的行为和元数据：

```json5
{
  wizard: {
    accessMode: "full",
    appRecommendations: true,
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
    securityAcknowledgedAt: "2026-01-01T00:00:00.000Z",
  },
}
```

- `wizard.accessMode`：在引导式新手引导开始时选择的设备发现同意选项。`"full"`（推荐）允许设置过程自动查找 AI 应用、密钥和本地运行时；`"guarded"` 会让设置过程在查找前询问一次，并提供手动配置作为替代方案。

- `wizard.appRecommendations` 默认为 `true`。将其设置为 `false`，可在引导式或经典新手引导期间禁用已安装应用推荐，并阻止 Gateway 网关的 `device.apps` 访问。节点主机仍需启用其单独的、默认关闭的已安装应用共享标志，才会公布该命令。

---

## 身份

请参阅 [Agent 默认值](/zh-CN/gateway/config-agents#agent-defaults)下的 `agents.entries` 身份字段。

---

## Bridge（旧版，已移除）

当前版本已不再包含 TCP bridge。节点通过 Gateway 网关 WebSocket 连接。`bridge.*` 键已不再属于配置模式（移除前验证会失败；`openclaw doctor --fix` 可剔除未知键）。

<Accordion title="旧版 bridge 配置（历史参考）">

```json
{
  "bridge": {
    "enabled": true,
    "port": 18790,
    "bind": "tailnet",
    "tls": {
      "enabled": true,
      "autoGenerate": true
    }
  }
}
```

</Accordion>

---

## 定时任务

```json5
{
  cron: {
    enabled: true,
    webhook: "https://example.invalid/legacy", // 已存储 notify:true 任务的已弃用回退项
    webhookToken: "replace-with-dedicated-token", // 用于出站 webhook 身份验证的可选 bearer token
    sessionRetention: "24h", // 时长字符串或 false
  },
}
```

- `sessionRetention`：在清理 SQLite 会话行之前，保留已完成的隔离定时任务运行会话的时长。还控制已归档且已删除的定时任务记录的清理。默认值：`24h`；设置为 `false` 可禁用。
- 运行历史记录会自动为每个任务保留最新的 2000 个终止状态行。丢失的行仍保留其 24 小时清理窗口。
- `webhookToken`：用于定时任务 webhook POST 递送的 bearer token（`delivery.mode = "webhook"`）；如果省略，则不发送身份验证标头。
- `webhook`：已弃用的旧版回退 webhook URL（http/https），由 `openclaw doctor --fix` 用于迁移仍含有 `notify: true` 的已存储任务；运行时递送使用各任务的 `delivery.mode="webhook"` 加 `delivery.to`，或在保留公告递送时使用 `delivery.completionDestination`。

### `cron.failureAlert`

```json5
{
  cron: {
    failureAlert: {
      enabled: false,
      after: 3,
      cooldownMs: 3600000,
      includeSkipped: false,
      mode: "announce",
      accountId: "main",
    },
  },
}
```

- `enabled`：启用定时任务失败警报（默认值：`false`）。
- `after`：触发警报前允许的连续失败次数（正整数，最小值：`1`）。
- `cooldownMs`：同一任务重复发出警报之间的最短毫秒数（非负整数）。
- `includeSkipped`：将连续跳过的运行计入警报阈值（默认值：`false`）。跳过的运行会单独跟踪，不影响执行错误退避。
- `mode`：递送模式——`"announce"` 通过渠道消息发送；`"webhook"` 发布到已配置的 webhook。
- `accountId`：用于限定警报递送范围的可选账户或渠道 ID。

### `cron.failureDestination`

```json5
{
  cron: {
    failureDestination: {
      mode: "announce",
      channel: "last",
      to: "channel:C1234567890",
      accountId: "main",
    },
  },
}
```

- 所有任务的定时任务失败通知默认目的地。
- `mode`：`"announce"` 或 `"webhook"`；当目标数据充足时，默认为 `"announce"`。
- `channel`：公告递送的渠道覆盖项。`"last"` 会复用最后已知的递送渠道。
- `to`：明确的公告目标或 webhook URL。webhook 模式下必填。
- `accountId`：可选的递送账户覆盖项。
- 每个任务的 `delivery.failureDestination` 会覆盖此全局默认值。
- 如果既未设置全局失败目的地，也未设置每任务失败目的地，已通过 `announce` 递送的任务会在失败时回退到该主要公告目标。
- `delivery.failureDestination` 仅支持 `sessionTarget="isolated"` 任务，除非任务的主要 `delivery.mode` 是 `"webhook"`。

请参阅[定时任务](/zh-CN/automation/cron-jobs)。隔离的定时任务执行会作为[后台任务](/zh-CN/automation/tasks)进行跟踪。

## 媒体模型模板变量

在 `tools.media.models[].args` 中展开的模板占位符：

| 变量                        | 描述                                              |
| --------------------------- | ------------------------------------------------- |
| `{{Body}}`                  | 完整的入站消息正文                                |
| `{{RawBody}}`               | 原始正文（不含历史记录/发送者包装）               |
| `{{BodyStripped}}`          | 已移除群组提及的正文                              |
| `{{From}}`                  | 发送者标识符                                      |
| `{{To}}`                    | 目的地标识符                                      |
| `{{MessageSid}}`            | 渠道消息 ID                                       |
| `{{SessionId}}`             | 当前会话 UUID                                     |
| `{{IsNewSession}}`          | 创建新会话时为 `"true"`                 |
| `{{AttachmentUrl}}`         | 当前附件 URL 或提供商引用                         |
| `{{AttachmentPath}}`        | 当前附件的本地路径                                |
| `{{AttachmentContentType}}` | 当前附件的 MIME 内容类型                          |
| `{{AttachmentDir}}`         | 包含 `AttachmentPath` 的目录                    |
| `{{AttachmentIndex}}`       | 从零开始的源事实索引                              |
| `{{Transcript}}`            | 音频转录文本                                      |
| `{{Prompt}}`                | 为 CLI 条目解析后的媒体提示词                     |
| `{{MaxChars}}`              | 为 CLI 条目解析后的最大输出字符数                 |
| `{{ChatType}}`              | `"direct"` 或 `"group"`          |
| `{{GroupSubject}}`          | 群组主题（尽力获取）                              |
| `{{GroupMembers}}`          | 群组成员预览（尽力获取）                          |
| `{{SenderName}}`            | 发送者显示名称（尽力获取）                        |
| `{{SenderE164}}`            | 发送者电话号码（尽力获取）                        |
| `{{Provider}}`              | 提供商提示（whatsapp、telegram、discord 等）      |

旧版 `{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}` 和 `{{MediaDir}}`
名称在插件 SDK 兼容窗口期间仍然可用，但已弃用。新配置应使用
`Attachment*` 变量。

---

## 配置包含（`$include`）

将配置拆分到多个文件中：

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

**合并行为：**

- 单个文件：替换包含它的对象。
- 文件数组：按顺序深度合并（后面的覆盖前面的）。
- 同级键：在包含项之后合并（覆盖包含的值）。
- 嵌套包含：最多可嵌套 10 层。
- 路径：相对于包含它的文件解析，但必须位于顶层配置目录内（`openclaw.json` 的 `dirname`）。只有在解析后仍位于该边界内时，才允许绝对路径/`../` 形式。设置 `OPENCLAW_INCLUDE_ROOTS`（绝对路径）可允许配置目录之外的其他根目录。
- 限制：路径不得包含空字节，并且解析前后的长度都必须严格小于 4096 个字符；每个包含的文件上限为 2 MB。
- 对于仅更改由单文件包含支持的一个顶层节的 OpenClaw 自有写入操作，内容会直接写入该被包含的文件。例如，`plugins install` 会更新 `plugins.json5` 中的 `plugins: { $include: "./plugins.json5" }`，并保持 `openclaw.json` 不变。
- 对于根包含、包含数组以及带同级覆盖项的包含，OpenClaw 自有写入操作只能读取；这些写入会以失败关闭方式终止，而不会将配置扁平化。
- 错误：针对文件缺失、解析错误、循环包含、无效路径格式和长度过长提供清晰的消息。

---

## 相关内容

- [配置](/zh-CN/gateway/configuration)
- [配置示例](/zh-CN/gateway/configuration-examples)
- [Doctor](/zh-CN/gateway/doctor)
