---
read_when:
    - 调整思考、快速模式或详细输出指令的解析或默认值
summary: /think、/fast、/verbose、/trace 和推理可见性的指令语法
title: 思考级别
x-i18n:
    generated_at: "2026-07-26T06:28:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80968ce58f642090ba0f807874e43eea1206cd31d919414c690b7537dc523658
    source_path: tools/thinking.md
    workflow: 16
---

## 功能说明

- 任何入站正文中的内联指令：`/t <level>`、`/think:<level>` 或 `/thinking <level>`。
- 级别（别名）：`off | minimal | low | medium | high | xhigh | adaptive | max | ultra`，大致对应 Anthropic 经典的 “think” < “think hard” < “think harder” < “ultrathink” 魔法词阶梯：
  - minimal ~ “think”
  - low ~ “think hard”
  - medium ~ “think harder”
  - high ~ “ultrathink”（最大预算）
  - xhigh ~ “ultrathink+”（GPT-5.2+ 和 Codex 模型，以及 Anthropic Claude Opus 4.7+ effort）
  - adaptive → 由提供商管理的自适应思考（Anthropic/Bedrock 上的 Claude 4.6、Anthropic Claude Opus 4.7+ 和 Google Gemini 动态思考支持）
  - max → 提供商最大推理强度（Anthropic Claude Opus 4.7+；Ollama 将其映射到最高原生 `think` effort）
  - ultra → 提供商最大推理强度，并在所选模型/运行时支持时主动编排子智能体
  - `x-high`、`x_high`、`extra-high`、`extra high` 和 `extra_high` 映射到 `xhigh`。
  - `highest` 映射到 `high`。
- 提供商说明：
  - 思考菜单和选择器由提供商配置文件驱动。提供商插件会声明所选模型的确切级别集合，包括二元 `on` 等标签。
  - 仅支持 `adaptive`、`xhigh`、`max` 和 `ultra` 的提供商/模型/运行时配置文件才会提供这些级别。对于不支持的级别，输入的指令会被拒绝，并显示该模型的有效选项。
  - 现有已存储但不受支持的级别会按提供商配置文件的等级重新映射。在非自适应模型上，`adaptive` 回退到 `medium`；`xhigh` 和 `max` 则回退到所选模型支持的最高非关闭级别。
  - 未明确设置思考级别时，Anthropic Claude 4.6 模型默认为 `adaptive`。
  - 除非明确设置思考级别，否则 Anthropic Claude Opus 4.8 和 Opus 4.7 会保持关闭思考。启用自适应思考后，Opus 4.8 由提供商定义的默认 effort 为 `high`。
  - Anthropic Claude Opus 4.7+ 将 `/think xhigh` 映射为自适应思考加 `output_config.effort: "xhigh"`，因为 `/think` 是思考指令，而 `xhigh` 是 Opus effort 设置。
  - Anthropic Claude Opus 4.7+ 还提供 `/think max`；它映射到相同的提供商最大 effort 路径。
  - 直连 DeepSeek V4 模型提供 `/think xhigh|max`；两者都映射到 DeepSeek `reasoning_effort: "max"`，较低的非关闭级别则映射到 `high`。
  - 通过 OpenRouter 路由的 DeepSeek V4 模型提供 `/think xhigh`，并发送 OpenRouter 支持的 `reasoning.effort` 值，而不是 DeepSeek 原生的顶层 `reasoning_effort`。较低的非关闭级别映射到 `high`，已存储的 `max` 覆盖值则回退到 `xhigh`。
  - 支持思考的 Ollama 模型提供 `/think low|medium|high|max`；`max` 映射到原生 `think: "high"`，因为 Ollama 的原生 API 接受 `low`、`medium` 和 `high` effort 字符串。
  - OpenAI GPT 模型通过特定于模型的 Responses API effort 支持来映射 `/think`。仅当目标模型支持时，`/think off` 才发送 `reasoning.effort: "none"`；否则，OpenClaw 会省略已禁用的推理载荷，而不是发送不受支持的值。
  - GPT-5.6 Sol 和 Terra 通过 Codex 运行时提供原生 `/think ultra`。GPT-5.6 Luna 通过 `max` 提供级别，因为其 Codex 目录未声明 Ultra。
  - 嵌入式 OpenClaw 运行时为 GPT-5.6 Sol、Terra 和 Luna 提供逻辑 `/think ultra`。它会发送提供商最大 effort，并添加仅作用于本次运行的主动子智能体编排指导。
  - 自定义 OpenAI 兼容目录条目可以通过将 `models.providers.<provider>.models[].compat.supportedReasoningEfforts` 设置为包含 `"xhigh"` 来选择启用 `/think xhigh`。这使用与出站 OpenAI 推理 effort 载荷映射相同的兼容性元数据，因此菜单、会话验证、智能体 CLI 和 `llm-task` 与传输行为保持一致。
  - 已配置但过时的 OpenRouter Hunter Alpha 引用会跳过代理推理注入，因为该已停用的路由可能通过推理字段返回最终答案文本。
  - Google Gemini 将 `/think adaptive` 映射到由 Gemini 提供商定义的动态思考。Gemini 3 请求会省略固定的 `thinkingLevel`，而 Gemini 2.5 请求会发送 `thinkingBudget: -1`；固定级别仍会映射到该模型系列最接近的 Gemini `thinkingLevel` 或预算。
  - Anthropic 兼容流式路径上的 MiniMax M2.x（`minimax/MiniMax-M2*`）默认为 `thinking: { type: "disabled" }`，除非你在模型参数或请求参数中明确设置思考。这样可避免 M2.x 的非原生 Anthropic 流格式泄漏 `reasoning_content` 增量。MiniMax-M3（以及 M3.x）不受此限制：M3 会发出正确的 Anthropic 思考块，并在禁用思考时返回空内容，因此 OpenClaw 会让 M3 保持在提供商的省略/自适应思考路径上。
  - 对于大多数 GLM 模型，Z.AI（`zai/*`）是二元模式（`on`/`off`）。GLM-5.2 是例外：它提供 `/think off|low|high|max`，将 `low` 和 `high` 映射到 Z.AI `reasoning_effort: "high"`，并将 `max` 映射到 `reasoning_effort: "max"`。
  - Moonshot API Kimi K3（`moonshot/kimi-k3`）始终以 `max` 思考，发送 `reasoning_effort: "max"`，省略 K2 的 `thinking` 字段和固定采样覆盖值，并保留 K3 支持的工具选择。Kimi Code K3（`kimi/k3` 和 `kimi/k3[1m]`）提供 `/think off|max`：关闭时发送 `thinking.type: "disabled"`，最大时发送采用最大 effort 的自适应思考。当前 Kimi Code 引用还包括 `kimi/kimi-for-coding` 和 `kimi/kimi-for-coding-highspeed`。Kimi K2.7 Code（`moonshot/kimi-k2.7-code` 和 `moonshot/kimi-k2.7-code-highspeed`）始终思考，仅提供 `on`，并省略出站的 `thinking` 和 `reasoning_effort`。其他 `moonshot/*` 模型将 `/think off` 映射到 `thinking: { type: "disabled" }`，并将任何非 `off` 级别映射到 `thinking: { type: "enabled" }`。启用 K2 思考时，Moonshot 仅接受 `tool_choice` `auto|none`；OpenClaw 会将不兼容的值规范化为 `auto`。

## 解析顺序

1. 消息中的内联指令（仅应用于该消息）。
2. 会话覆盖值（通过发送仅含指令的消息设置）。
3. 每个智能体的默认值（配置中的 `agents.entries.*.thinkingDefault`）。
4. 全局默认值（配置中的 `agents.defaults.thinkingDefault`）。
5. 回退：有提供商声明的默认值时使用该值；否则，支持推理的模型解析为 `medium` 或该模型支持的最接近的非 `off` 级别，不支持推理的模型则保持 `off`。

## 设置会话默认值

- 发送一条**仅包含**指令的消息（允许空白字符），例如 `/think:medium` 或 `/t high`。
- 该设置会在当前会话中持续生效（默认按发送者区分）。使用 `/think default` 清除会话覆盖值，并继承已配置的默认值或提供商默认值；别名包括 `inherit`、`clear`、`reset` 和 `unpin`。
- `/think off` 会存储一个明确的关闭覆盖值。在更改或清除会话覆盖值之前，它会一直禁用思考。
- 系统会发送确认回复（`Thinking level set to high.` / `Thinking disabled.`）。如果级别无效（例如 `/thinking big`），命令会被拒绝并显示提示，同时保持会话状态不变。
- 发送不带参数的 `/think`（或 `/think:`）可查看当前思考级别。

## 智能体的应用方式

- **嵌入式 OpenClaw**：解析后的级别会传递给进程内 OpenClaw 智能体运行时。
- **Claude CLI 后端**：使用 `claude-cli` 时，具体的非关闭级别会作为 `--effort` 传递给 Claude Code；`adaptive` 会移除已配置的 effort 标志，并将实际 effort 交由 Claude Code 的环境、设置和模型默认值决定。请参阅 [CLI 后端](/zh-CN/gateway/cli-backends)。

## 快速模式（/fast）

- 级别：`auto|on|off|default`。
- 仅含指令的消息会切换会话快速模式覆盖值，并回复 `Fast mode set to auto.`、`Fast mode enabled.` 或 `Fast mode disabled.`。使用 `/fast default` 清除会话覆盖值，并继承已配置的默认值；别名包括 `inherit`、`clear`、`reset` 和 `unpin`。
- 发送不带模式的 `/fast`（或 `/fast status`）可查看当前生效的快速模式状态。
- OpenClaw 按以下顺序解析快速模式：
  1. 内联/仅含指令的 `/fast auto|on|off` 覆盖值（`/fast default` 清除此层）
  2. 会话覆盖值
  3. 每个智能体的默认值（`agents.entries.*.fastModeDefault`）
  4. 每个模型的配置：`agents.defaults.models["<provider>/<model>"].params.fastMode`
  5. 回退：`off`
- `auto` 会将会话/配置模式保持为自动，但会独立解析每次新的模型调用。在自动截止点之前开始的调用会启用快速模式；之后开始的重试、回退、工具结果或续接调用会禁用快速模式。截止时间默认为 60 秒；在当前模型上设置 `agents.defaults.models["<provider>/<model>"].params.fastAutoOnSeconds` 可更改此时间。
- 对于 `openai/*`，快速模式会通过在支持的 Responses 请求中发送 `service_tier=priority`，映射到 OpenAI 优先处理。
- 对于由 Codex 支持的 `openai/*` / `openai-codex/*` 模型，快速模式会在 Codex Responses 中发送相同的 `service_tier=priority` 标志。原生 Codex app-server 轮次仅在 `turn/start` 或线程启动/恢复时接收该层级，因此 `auto` 无法更改已经运行中的 app-server 轮次层级；它会应用于 OpenClaw 启动的下一个模型轮次。
- 对于直接公开的 `anthropic/*` 请求（包括发送到 `api.anthropic.com` 的 OAuth 身份验证流量），快速模式会映射到 Anthropic 服务层级：`/fast on` 设置 `service_tier=auto`，`/fast off` 设置 `service_tier=standard_only`。
- 对于 Anthropic 兼容路径上的 `minimax/*`，`/fast on`（或 `params.fastMode: true`）会将 `MiniMax-M2.7` 重写为 `MiniMax-M2.7-highspeed`。
- 同时设置时，明确的 Anthropic `serviceTier` / `service_tier` 模型参数会覆盖快速模式默认值。对于非 Anthropic 代理基础 URL，OpenClaw 仍会跳过 Anthropic 服务层级注入。
- 启用快速模式时，`/status` 会显示 `Fast`；配置模式为自动时，则显示 `Fast:auto`。

## 详细输出指令（/verbose 或 /v）

- 级别：`on`（最低）| `full` | `off`（默认）。
- 仅含指令的消息会切换会话详细输出，并回复 `Verbose logging enabled.` / `Verbose logging disabled.`；无效级别会返回提示，但不会更改状态。
- `/verbose off` 会存储显式的会话覆盖设置；可在会话 UI 中选择 `inherit` 将其清除。
- 经过授权的外部渠道发送者可以持久保存会话详细输出覆盖设置。内部 Gateway 网关/WebChat 客户端需要 `operator.admin` 才能将其持久保存。
- 内联指令仅影响当前消息；其他情况下应用会话/全局默认值。
- 发送不带参数的 `/verbose`（或 `/verbose:`）可查看当前详细输出级别。
- 开启详细输出时，会生成结构化工具结果的智能体会将每次工具调用作为单独的纯元数据消息发回，并在可用时添加 `<emoji> <tool-name>: <arg>` 前缀。这些工具摘要会在每个工具启动后立即发送（显示为独立气泡），而不是作为流式增量发送。
- 在普通模式下，工具失败摘要仍然可见，但除非详细输出为 `full`，否则原始错误详情后缀会被隐藏。
- 当详细输出为 `full` 时，工具输出也会在完成后转发（显示为独立气泡，并截断至安全长度）。如果在运行过程中切换 `/verbose on|full|off`，后续工具气泡会采用新设置。
- `agents.defaults.toolProgressDetail` 控制 `/verbose` 工具摘要和进度草稿工具行的形式。使用 `"explain"`（默认）可显示简洁易懂的标签，例如 `🛠️ Exec: checking JS syntax`；如果还需要附加原始命令/详情以便调试，请使用 `"raw"`。每个智能体的 `agents.entries.*.toolProgressDetail` 会覆盖默认值。
  - `explain`：`🛠️ Exec: check JS syntax for /tmp/app.js`
  - `raw`：`🛠️ Exec: check JS syntax for /tmp/app.js, node --check /tmp/app.js`

## 插件跟踪指令（/trace）

- 级别：`on` | `off`（默认）。
- 仅含指令的消息会切换会话插件跟踪输出，并回复 `Plugin trace enabled.` / `Plugin trace disabled.`。
- 内联指令仅影响当前消息；其他情况下应用会话/全局默认值。
- 发送不带参数的 `/trace`（或 `/trace:`）可查看当前跟踪级别。
- `/trace` 的范围比 `/verbose` 更窄：它只会公开插件自身生成的跟踪/调试行，例如主动记忆调试摘要。
- 跟踪行可能出现在 `/status` 中，也可能在智能体正常回复后作为后续诊断消息出现。

## 推理可见性（/reasoning）

- 级别：`on|off|stream`。
- 仅含指令的消息会切换是否在回复中显示思考块。
- 启用后，推理会作为带有 `Thinking` 前缀的**单独消息**发送。
- `stream`：当当前渠道支持推理预览时，会在回复生成期间流式传输推理，随后发送不含推理的最终答案。
- 别名：`/reason`。
- 发送不带参数的 `/reasoning`（或 `/reasoning:`）可查看当前推理级别。
- 解析顺序：内联指令、会话覆盖设置、每个智能体的默认值（`agents.entries.*.reasoningDefault`）、全局默认值（`agents.defaults.reasoningDefault`），最后是回退值（`off`）。

系统会谨慎处理本地模型格式错误的推理标签。在普通回复中，已闭合的 `<think>...</think>` 块会保持隐藏，已显示文本之后未闭合的推理内容也会被隐藏。如果回复完全包裹在单个未闭合的开始标签中，并且原本会以空文本形式送达，OpenClaw 会移除格式错误的开始标签并送达剩余文本。

## 相关内容

- 提升权限模式的文档位于[提升权限模式](/zh-CN/tools/elevated)。

## Heartbeat

- Heartbeat 探测正文为配置的 Heartbeat 提示词（默认：`Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`）。Heartbeat 消息中的内联指令会照常生效（但应避免通过 Heartbeat 更改会话默认值）。
- Heartbeat 默认仅送达最终载荷。若还要发送单独的 `Thinking` 消息（可用时），请设置 `agents.defaults.heartbeat.includeReasoning: true` 或每个智能体的 `agents.entries.*.heartbeat.includeReasoning: true`。

## Web 聊天 UI

- 页面加载时，Web 聊天的思考选择器会同步入站会话存储/配置中保存的会话级别。
- 选择其他级别会通过 `sessions.patch` 立即写入会话覆盖设置；它不会等待下一次发送，也不是一次性的 `thinkingOnce` 覆盖设置。
- 如果在模型、推理或速度选择器的更改仍在应用时发送消息，系统会等待所有待处理的选择器补丁；如果有更改失败，消息会保持未发送状态，以便检查。
- 第一个选项始终用于清除覆盖设置。它会显示 `Inherited: <resolved level>`，包括继承的思考功能已禁用时的 `Inherited: Off`。
- 显式选择器选项使用各自的直接级别标签，并在存在提供商标签时予以保留（例如，提供商标记的 `max` 选项会显示为 `Maximum`）。
- 选择器使用 Gateway 网关会话行/默认值返回的 `thinkingLevels`，同时保留 `thinkingOptions` 作为旧版标签列表。浏览器 UI 不维护自己的提供商正则表达式列表；模型特定的级别集合由插件所有。
- `/think:<level>` 仍然有效，并会更新同一项已保存的会话级别，因此聊天指令和选择器会保持同步。

## 提供商配置文件

- 提供商插件可以公开 `resolveThinkingProfile(ctx)`，以定义模型支持的级别和默认值。
- 代理 Claude 模型的提供商插件应复用 `openclaw/plugin-sdk/provider-model-shared` 中的 `resolveClaudeThinkingProfile(modelId)`，以确保直接 Anthropic 目录和代理目录保持一致。
- 每个配置文件级别都有一个已存储的规范 `id`（`off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`adaptive`、`max` 或 `ultra`），并且可以包含显示用的 `label`。二元提供商使用 `{ id: "low", label: "on" }`。
- 配置文件钩子会在可用时接收合并后的目录事实，包括 `reasoning`、`compat.thinkingFormat` 和 `compat.supportedReasoningEfforts`。仅当配置的请求契约支持相应载荷时，才使用这些事实公开二元或自定义配置文件。
- 需要验证显式思考覆盖设置的工具插件应使用 `api.runtime.agent.resolveThinkingPolicy({ provider, model, agentRuntime })` 和 `api.runtime.agent.normalizeThinkingLevel(...)`；不应自行维护提供商/模型级别列表。当工具拥有执行路径时（例如始终嵌入式运行），请传入 `agentRuntime`。
- 能够访问已配置的自定义模型元数据的工具插件，可以将 `catalog` 传入 `resolveThinkingPolicy`，使 `compat.supportedReasoningEfforts` 的显式启用设置能够反映在插件端验证中。
- 已发布的旧版钩子（`supportsXHighThinking`、`isBinaryThinking` 和 `resolveDefaultThinkingLevel`）仍作为兼容性适配器保留，但新的自定义级别集合应使用 `resolveThinkingProfile`。
- Gateway 网关行/默认值会公开 `thinkingLevels`、`thinkingOptions` 和 `thinkingDefault`，以便 ACP/聊天客户端呈现与运行时验证所用内容相同的配置文件 ID 和标签。
