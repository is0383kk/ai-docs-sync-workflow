---
read_when:
    - 编辑系统提示词文本、工具列表或时间/Heartbeat 部分
    - 更改工作区引导或 Skills 注入行为
summary: OpenClaw 系统提示词包含的内容及其组装方式
title: 系统提示词
x-i18n:
    generated_at: "2026-07-26T06:08:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 669fbc6f21a82a2c3c067d2ff3a6365acb3316460a85f2db165b7ad49ce79f70
    source_path: concepts/system-prompt.md
    workflow: 16
---

OpenClaw 会为每次智能体运行构建自己的系统提示词；不存在运行时默认提示词。

组装分为三层：

- `buildAgentSystemPrompt` 根据显式输入渲染提示词。它始终是纯渲染器，不会直接读取全局配置。
- `resolveAgentSystemPromptConfig` 为特定智能体解析由配置支持的提示词选项（所有者显示、TTS 提示、模型别名、记忆引用模式、子智能体委派模式）。
- 运行时适配器（嵌入式、CLI、命令/导出预览、压缩）收集实时信息（工具、沙箱状态、渠道能力、上下文文件、提供商提示词贡献），并调用已配置的提示词门面。

这样既能使导出/调试提示词界面与实时运行保持一致，又不会把每项运行时细节都塞进一个单体构建器。

提供商插件可以贡献支持缓存的指导，而无需替换 OpenClaw 所有的提示词。提供商运行时可以：

- 替换三个具名核心部分之一：`interaction_style`、`tool_call_style`、`execution_bias`
- 在提示词缓存边界上方注入**稳定前缀**
- 在提示词缓存边界下方注入**动态后缀**

使用提供商所有的贡献来进行特定模型系列的调优。仅将旧版 `before_prompt_build` 钩子用于兼容性或真正全局性的提示词更改。

内置的 OpenAI/Codex GPT-5 系列叠加层（`resolveGpt5SystemPromptContribution`）使用此机制：一个 `stablePrefix` 行为契约（执行策略、工具规范、输出契约、完成契约），以及可选的 `interaction_style` 覆盖项，用于提供更友好的语气。它适用于通过 OpenAI 或 Codex 插件路由的任何 `gpt-5*` 模型 ID，由 `agents.defaults.promptOverlays.gpt5.personality`（`"friendly"`/`"on"` 或 `"off"`）控制。

## 结构

提示词结构紧凑，包含以下固定部分：

- **工具**：结构化工具是真实来源的提醒，以及运行时工具使用指导。启用实验性 `update_plan` 工具（`tools.experimental.planTool`）时，其自身的工具描述会补充：仅将其用于非简单的多步骤工作，最多保持一个步骤处于 `in_progress` 状态，并对简单的单步骤工作跳过此工具。
- **执行倾向**：在当前轮次中处理可执行的请求，持续执行直至完成或受阻，从效果不佳的工具结果中恢复，实时检查可变状态，并在最终完成前进行验证。
- **安全**：简短的护栏提醒，防止追求权力的行为或绕过监督。
- **Skills**（可用时）：告知模型如何按需加载技能说明。
- **OpenClaw 控制**：配置/重启工作优先使用 `gateway` 工具；不要虚构 CLI 命令。
- **OpenClaw 自我更新**：使用 `config.schema.lookup` 安全检查配置，使用 `config.patch` 修补配置，使用 `config.apply` 替换完整配置，并且仅在用户明确请求时运行 `update.run`。面向智能体的 `gateway` 工具会拒绝重写 `tools.exec.mode`。
- **工作区**：工作目录（`agents.defaults.workspace`）。
- **文档**：本地文档/源代码路径，以及何时读取它们。
- **工作区文件（已注入）**：说明下方包含引导文件。
- **沙箱**（启用时）：沙箱隔离的运行时、沙箱路径、提升权限的 Exec 可用性。
- **当前日期和时间**：仅包含时区（缓存稳定；实时时钟来自 `session_status`）。
- **助手输出指令**：精简的附件、语音消息和回复标签语法。
- **Heartbeats**：为默认智能体启用 Heartbeat 时的 Heartbeat 提示词和确认行为。
- **运行时**：主机、操作系统、节点、模型、仓库根目录（检测到时）、思考级别（单行）。
- **推理**：当前可见性级别以及 `/reasoning` 切换提示。

大型稳定内容（包括**项目上下文**）保留在内部提示词缓存边界上方。每轮易变部分（Control UI 嵌入指导、**消息传递**、**语音**、**群聊上下文**、**表情回应**、**Heartbeats**、**运行时**）附加在该边界下方，以便支持前缀缓存的本地后端能够在不同渠道轮次间复用稳定的工作区前缀。如果接受的架构已包含当前渠道名称这一运行时细节，工具描述应避免嵌入该名称。

工具部分还包含长时间运行工作指导：

- 对未来的跟进工作（`check back later`、提醒、重复性工作）使用 cron，而不是 `exec` 休眠循环、`yieldMs` 延迟技巧或重复的 `process` 轮询
- 仅对立即启动并在后台继续运行的命令使用 `exec` / `process`
- 启用自动完成唤醒时，只启动命令一次，并依赖基于推送的唤醒路径
- 使用 `process` 获取运行中命令的日志、状态、输入，或对其进行干预
- 对于较大型任务，优先使用 `sessions_spawn`；子智能体完成通知基于推送，并会自动向请求者发布
- 不要仅为等待完成而循环轮询 `subagents list` / `sessions_list`

`agents.defaults.subagents.delegationMode`（默认值为 `"suggest"`）可以强化此行为。`"prefer"` 会添加专门的**子智能体委派**部分，要求主智能体作为响应迅速的协调者，并将比直接回复更复杂的任何工作通过 `sessions_spawn` 推送出去。这仅影响提示词；工具策略仍然控制 `sessions_spawn` 是否可用。

系统提示词中的安全护栏属于建议，而非强制执行。硬性执行应使用工具策略、Exec 审批、沙箱隔离和渠道允许列表；按照设计，操作员可以禁用提示词护栏。

对于提供原生审批卡片/按钮的渠道，提示词会要求智能体优先依赖该 UI，并且仅在工具结果表明聊天审批不可用或手动审批是唯一路径时，才包含手动 `/approve` 命令。

## 提示词模式

OpenClaw 会为子智能体渲染更小的系统提示词。运行时为每次运行设置一个 `promptMode`（不是面向用户的配置）：

- `full`（默认）：包含上述所有部分。
- `minimal`：用于子智能体；省略记忆提示词部分（以**记忆回顾**形式内置）、**OpenClaw 自我更新**、**模型别名**、**用户身份**、**助手输出指令**、**消息传递**、**静默回复**和 **Heartbeats**。工具、**安全**、**Skills**（提供时）、工作区、沙箱、当前日期和时间（已知时）、运行时以及注入的上下文仍然可用。
- `none`：仅返回基础身份行。

在 `promptMode=minimal` 下，额外注入的提示词标记为**子智能体上下文**，而不是**群聊上下文**。

对于渠道自动回复运行，当直接、群组或仅消息工具上下文已经负责可见回复契约时，OpenClaw 会省略通用的**静默回复**部分。只有旧版自动群组/渠道模式会显示 `NO_REPLY`；直接聊天和仅消息工具回复会跳过静默令牌指导。

## 提示词快照

OpenClaw 在 `test/fixtures/agents/prompt-snapshots/codex-runtime-happy-path/` 下为 Codex runtime 的正常路径保留已提交的提示词快照。它们会渲染选定的应用服务器线程/轮次参数，以及重建的模型绑定提示词分层栈，用于 Telegram 私聊、Discord 群组和 Heartbeat 轮次：固定的 Codex `gpt-5.5` 模型提示词夹具、Codex 正常路径权限开发者文本、OpenClaw 开发者指令、由 OpenClaw 提供时的轮次范围协作模式指令、用户轮次输入，以及动态工具规范的引用。

使用 `pnpm prompt:snapshots:sync-codex-model` 刷新固定的 Codex 模型提示词夹具。默认情况下，它依次查找 `$CODEX_HOME/models_cache.json`、`~/.codex/models_cache.json`，然后查找维护者检出目录约定的 `~/code/codex/codex-rs/models-manager/models.json`；如果都不存在，它会退出且不更改已提交的夹具。传入 `--catalog <path>` 可从特定的 `models_cache.json` 或 `models.json` 文件刷新。

这些快照不是逐字节的原始 OpenAI 请求捕获。OpenClaw 发送线程和轮次参数后，Codex 可以添加运行时所有的工作区上下文（`AGENTS.md`、环境上下文、记忆、应用/插件指令、内置的默认协作模式指令）。

使用 `pnpm prompt:snapshots:gen` 重新生成；使用 `pnpm prompt:snapshots:check` 验证偏差。CI 会将偏差检查与额外边界分片一同运行，因此提示词更改和快照更新会在同一个 PR 中落地。

## 工作区引导注入

引导文件从活动工作区解析，并路由到与其生命周期匹配的提示词界面：

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`（仅用于全新工作区）
- 存在 `MEMORY.md` 时

在原生 Codex harness 上，OpenClaw 会避免在每个用户轮次中重复稳定的工作区文件。Codex 通过自己的项目文档发现机制加载 `AGENTS.md`。`TOOLS.md` 作为继承的 Codex 开发者指令转发。`SOUL.md`、`IDENTITY.md` 和 `USER.md` 作为轮次范围的协作开发者指令转发，因此原生 Codex 子智能体不会继承它们。`HEARTBEAT.md` 内容不会直接注入；当该文件存在且非空时，Heartbeat 轮次会获得一条指向该文件的协作模式说明。`MEMORY.md` 内容也不会粘贴到每个原生 Codex 轮次中：当工作区可使用记忆工具时，Codex 轮次会获得一条简短的工作区记忆说明，引导模型前往 `memory_search` 或 `memory_get`。如果工具被禁用、记忆搜索不可用，或者活动工作区与智能体记忆工作区不同，`MEMORY.md` 会回退到正常的有界轮次上下文路径。`BOOTSTRAP.md` 保持正常的轮次上下文角色。

在非 Codex harness 上，引导文件会按照其现有门控条件组合到 OpenClaw 提示词中。当默认智能体禁用 Heartbeat 或 `agents.defaults.heartbeat.includeSystemPromptSection` 为 false 时，正常运行会省略 `HEARTBEAT.md`。应保持注入文件简洁，尤其是非 Codex 的 `MEMORY.md`：它应当是一份精选的长期摘要，而详细的每日笔记应放在 `memory/*.md` 中，并可通过 `memory_search` / `memory_get` 按需检索。过大的非 Codex `MEMORY.md` 文件会增加提示词用量，并且可能会根据下方的引导文件限制仅注入部分内容。

<Note>
`memory/*.md` 每日文件**不**属于正常引导项目上下文。在普通轮次中，它们通过 `memory_search` / `memory_get` 按需访问，因此除非模型显式读取，否则不会占用上下文窗口。纯 `/new` 和 `/reset` 轮次属于例外：运行时可以为第一个轮次预置近期每日记忆，作为一次性启动上下文块。
</Note>

大型文件会被截断，并附带标记：

| 限制                                         | 配置键                                             | 默认值   |
| -------------------------------------------- | -------------------------------------------------- | -------- |
| 每个文件的最大字符数                         | `agents.defaults.bootstrapMaxChars`                | 20000    |
| 所有文件的总字符数                           | `agents.defaults.bootstrapTotalMaxChars`           | 60000    |
| 截断警告（`off`\|`once`\|`always`） | `agents.defaults.bootstrapPromptTruncationWarning` | `always` |

缺失的文件会注入一个简短的文件缺失标记。详细的原始/注入计数仍保留在诊断信息中，例如 `/context`、`/status`、doctor 和日志。

对于记忆文件，截断并不意味着数据丢失：磁盘上的文件保持完整。在 Native Codex 上，当记忆工具可用时，会按需通过记忆工具读取 `MEMORY.md`；否则使用有界的提示词回退。在其他 harness 上，模型只能看到缩短后的注入副本，直到它直接读取或搜索记忆。如果 `MEMORY.md` 被反复截断，请将其提炼为更短的持久摘要，把详细历史移至 `memory/*.md`，或有意提高引导文件限制。

子智能体会话仅注入 `AGENTS.md` 和 `TOOLS.md`（其他引导文件会被过滤掉，以保持较小的子智能体上下文）。

内部钩子可以通过 `agent:bootstrap` 事件拦截此步骤，以修改或替换注入的引导文件（例如，将 `SOUL.md` 替换为其他角色设定）。

若要让表达听起来不那么泛化，请从 [SOUL.md 个性指南](/zh-CN/concepts/soul)开始。

若要检查每个注入文件的贡献量（原始与注入、截断、工具架构开销），请使用 `/context list` 或 `/context detail`。请参阅[上下文](/zh-CN/concepts/context)。

## 时间处理

仅当用户时区已知时，才会显示**当前日期与时间**部分，并且只包含**时区**（不包含动态时钟或时间格式），以保持提示词缓存稳定。

当智能体需要当前时间时，请使用 `session_status`；其状态卡片包含时间戳行。该工具还可以选择设置每个会话的模型覆盖（`model=default` 会将其清除）。

配置方式：

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat`（`auto` | `12` | `24`）

有关完整行为详情，请参阅[时区](/zh-CN/concepts/timezone)和[日期与时间](/zh-CN/date-time)。

## Skills

存在符合条件的 Skills 时，OpenClaw 会注入一个紧凑的 `<available_skills>` 列表（`formatSkillsForPrompt`），其中包含每项 Skills 的**文件路径**和基于内容生成的 `<version>sha256:...</version>` 标记。提示词会指示模型使用 `read` 加载所列位置（工作区、托管或内置）的 SKILL.md，并在某项 Skills 的 `<version>` 与上一轮不同时重新读取该 Skills。如果没有符合条件的 Skills，则省略 Skills 部分。

Native Codex 轮次会以轮次范围的协作开发者指令接收此列表，而不是将其作为每轮用户输入，但保留确切定时提示词的轻量级定时任务轮次除外。其他 harness 保留常规提示词部分。

该位置可以指向嵌套的 Skills，例如 `skills/personal/foo/SKILL.md`。嵌套仅用于组织；提示词使用 `SKILL.md` frontmatter 中的扁平 Skills 名称。

资格判定包括 Skills 元数据门控、运行时环境/配置检查，以及配置了 `agents.defaults.skills` 或 `agents.entries.*.skills` 时生效的智能体 Skills 允许列表。仅当所属插件启用时，插件内置的 Skills 才符合条件，从而让工具插件可以提供更深入的操作指南，而不必将所有这些指导嵌入每个工具说明中。

```xml
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
    <version>sha256:...</version>
  </skill>
</available_skills>
```

这样既能保持基础提示词精简，又能支持有针对性地使用 Skills。大小限制由 Skills 子系统负责，与通用运行时读取/注入大小限制分开：

| 范围     | Skills 提示词预算                                 | 运行时摘录预算             |
| --------- | ---------------------------------------------------- | ---------------------------------- |
| 全局    | `skills.limits.maxSkillsPromptChars`                 | `agents.defaults.contextLimits.*`  |
| 每个智能体 | `agents.entries.*.skillsLimits.maxSkillsPromptChars` | `agents.entries.*.contextLimits.*` |

运行时摘录预算涵盖 `memory_get`、实时工具结果和压缩后的 `AGENTS.md` 刷新。

## 文档

**文档**部分会在本地文档可用时指向本地文档（Git 检出中的 `docs/` 或内置 npm 包文档），否则回退到 [https://docs.openclaw.ai](https://docs.openclaw.ai)。它还会列出 OpenClaw 源代码的位置：Git 检出会公开本地源代码根目录；软件包安装则提供 GitHub 源代码 URL，并指示在文档不完整或过时时前往该处审查源代码。

在模型了解 OpenClaw 的工作方式（记忆/每日笔记、会话、工具、Gateway 网关、配置、命令、项目上下文）之前，提示词将文档定位为 OpenClaw 自身知识的权威来源，并指示模型将 `AGENTS.md`、项目上下文、工作区/配置文件/记忆笔记和 `memory_search` 视为指令上下文或用户记忆，而不是 OpenClaw 的设计/实现知识。如果文档没有相关说明或已经过时，模型应明确指出并检查源代码。提示词还指示模型尽可能自行运行 `openclaw status`，仅在无法访问时才询问用户。

对于配置，提示词会特别指引智能体使用 `gateway` 工具操作 `config.schema.lookup`，获取精确到字段级别的文档和约束，然后查阅 `docs/gateway/configuration.md` 和 `docs/gateway/configuration-reference.md` 以获取更广泛的指导。

## 相关内容

- [Agent 运行时](/zh-CN/concepts/agent)
- [Agent 工作区](/zh-CN/concepts/agent-workspace)
- [上下文引擎](/zh-CN/concepts/context-engine)
