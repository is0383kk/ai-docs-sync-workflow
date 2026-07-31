---
read_when:
    - 解释 Token 用量、费用或上下文窗口
    - 调试上下文增长或压缩行为
summary: OpenClaw 如何构建提示词上下文并报告 token 用量和成本
title: 令牌用量和成本
x-i18n:
    generated_at: "2026-07-26T06:01:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6624bceb0bcbca769c9d569389b73b82f1ea73133e09f0ae9859833196d85911
    source_path: reference/token-use.md
    workflow: 16
---

OpenClaw 跟踪的是 **token**，而不是字符。token 因模型而异，但对英文文本而言，大多数
OpenAI 风格的模型平均约为每个 token 4 个字符。

## 系统提示词的构建方式

OpenClaw 会在每次运行时组装自己的系统提示词。其中包括：

- 工具列表 + 简短说明
- Skills 列表（仅元数据；指令通过 `read` 按需加载）。原生
  Codex 轮次会将精简的 Skills 块作为仅限当前轮次的协作
  开发者指令；其他 harness 则会在常规提示词界面中接收它。
  受 `skills.limits.maxSkillsPromptChars` 限制，并可通过
  `agents.entries.*.skillsLimits.maxSkillsPromptChars` 按智能体覆盖。
- 自更新指令
- 工作区 + 引导文件（`AGENTS.md`、`SOUL.md`、`TOOLS.md`、
  `IDENTITY.md`、`USER.md`、`HEARTBEAT.md`，新建时还包括 `BOOTSTRAP.md`，
  存在时还包括 `MEMORY.md`）。注入的大型文件会由
  `agents.defaults.bootstrapMaxChars` 截断（默认值：`20000`）；引导内容的总
  注入量受 `agents.defaults.bootstrapTotalMaxChars` 限制（默认值：
  `60000`）。
  - 当相应工作区可使用记忆工具时，原生 Codex 轮次不会粘贴原始的
    `MEMORY.md`；它们会改为在仅限当前轮次的协作开发者指令中接收一个简短的记忆指针，
    并按需使用记忆工具。如果工具已禁用、记忆搜索不可用，或当前工作区不同于
    智能体记忆工作区，`MEMORY.md` 会回退到常规的有界轮次上下文路径。
  - 小写的根 `memory.md` 永远不会被注入。它是供
    `openclaw doctor --fix` 使用的旧版修复输入，后者会将其迁移到 `MEMORY.md`。
  - `memory/*.md` 每日文件不属于常规引导提示词；
    在普通轮次中，它们仍通过记忆工具按需使用。重置/启动时的模型运行可在首轮前置一个
    一次性启动上下文块，其中包含最近的每日记忆，
    由 `agents.defaults.startupContext` 控制。纯聊天的 `/new` 和
    `/reset` 会直接得到确认，而不会调用模型。
  - 压缩后的 `AGENTS.md` 摘录需要通过
    `agents.defaults.compaction.postCompactionSections` 显式选择启用；插件可以通过
    `before_prompt_build` 添加其他上下文。
- 时间（UTC + 用户时区）
- 回复标签 + Heartbeat 行为
- 运行时元数据（主机/操作系统/模型/思考）

完整说明请参阅[系统提示词](/zh-CN/concepts/system-prompt)。

记录凭据或身份验证代码片段时，请遵循
[密钥占位符约定](/zh-CN/reference/secret-placeholder-conventions)，以免仅修改文档时触发密钥扫描器误报。

## 上下文窗口中包含哪些内容

模型接收的所有内容都会计入上下文限制：

- 系统提示词（上述所有部分）
- 对话历史记录（用户 + 助手消息）
- 工具调用和工具结果
- 附件/转录内容（图片、音频、文件）
- 压缩摘要和修剪工件
- 提供商包装层或安全标头（不可见，但仍会计入）

运行时负载较重的界面在
`agents.defaults.contextLimits` 下有各自的明确上限（按智能体覆盖位于
`agents.entries.*.contextLimits` 下）：

| 键                       | 用途                                                                     |
| ------------------------ | ------------------------------------------------------------------------ |
| `memoryGetMaxChars`      | `memory_get` 在截断前返回的最大字符数。                            |
| `postCompactionMaxChars` | 压缩后刷新期间从 `AGENTS.md` 保留的最大字符数。 |

这些是有界的运行时摘录和由运行时拥有的注入块，
独立于引导限制、启动上下文限制和 Skills 提示词
限制。

OpenClaw 根据有效的模型上下文窗口推导实时工具结果上限：
低于 100K token 时为 `16000` 个字符，达到 100K+ token 时为 `32000` 个字符，达到 200K+ token 时为 `64000` 个字符。
运行时上下文占比防护还会将单个工具结果限制为上下文窗口的 30%。

当大型提供商窗口会显著改变成本或延迟时，不会自动启用。例如，OpenAI 直连的 GPT-5.5 和 GPT-5.6 模型
公布的总窗口为 `1050000` token，但 OpenClaw 默认将其有效
运行时预算设为 `272000` token。选择启用的 `922000` 输入预算会预留完整的
`128000` 输出配额；一旦输入超过 `272000` token，OpenAI 会对整个请求应用更高的长上下文定价。请参阅
[OpenAI 上下文窗口默认值](/zh-CN/providers/openai#context-window-defaults-and-long-context-opt-in)。

对于图片，OpenClaw 会在调用提供商之前缩小转录内容/工具中的图片载荷。
可通过 `agents.defaults.imageMaxDimensionPx` 调整（默认值：
`1200`）：

- 较低的值会减少视觉 token 用量和载荷大小。
- 较高的值会为依赖 OCR/UI 的屏幕截图保留更多视觉细节。

要查看实用的明细（按各个注入文件、工具、Skills 和系统
提示词大小划分），请使用 `/context list` 或 `/context detail`。请参阅
[上下文](/zh-CN/concepts/context)。

## 如何查看当前 token 用量

在聊天中：

- `/status` -> 显示包含会话模型、上下文用量、
  上次响应的输入/输出 token，以及在为当前模型配置了本地定价时显示预估成本的多表情符号状态卡片。
- `/usage off|tokens|full` -> 在每条回复后附加单次响应的用量页脚。
  设置会按会话持久化（存储为 `responseUsage`）。
  - `/usage reset`（别名：`inherit`、`clear`、`default`）会清除
    会话覆盖设置，使其重新继承已配置的默认值。
  - `/usage tokens` 显示轮次 token/缓存详情。
  - `/usage full` 显示精简的模型/上下文/成本详情；只有当 OpenClaw 拥有
    当前模型的用量元数据和本地定价时，才会显示预估成本。自定义 `messages.usageTemplate` 布局可以包含
    token/缓存字段。
- `/usage cost` -> 根据 OpenClaw 会话日志生成本地成本摘要。

其他界面：

- **TUI/Web TUI：**支持 `/status` 和 `/usage`。
- **CLI：**`openclaw status --usage` 和 `openclaw channels list` 显示
  标准化的提供商配额窗口（`X% left`，而非单次响应成本）。
  当前支持用量窗口的提供商包括：Claude（Anthropic）、ClawRouter、Copilot
  （GitHub）、DeepSeek、Gemini（Google Gemini CLI）、MiniMax、OpenAI、Xiaomi、
  Xiaomi Token Plan 和 z.ai。

用量界面会先标准化常见的提供商原生字段别名，然后再显示。对于 OpenAI 系列的 Responses 流量，这包括
`input_tokens`/`output_tokens` 和 `prompt_tokens`/`completion_tokens`，因此
特定于传输方式的字段名称不会改变 `/status`、`/usage` 或会话
摘要。Gemini CLI 用量也会进行标准化：默认的 `stream-json`
解析器会读取助手的 `message` 事件，`stats.cached` 会映射到
`cacheRead`；当 CLI 省略明确的 `stats.input` 字段时，则使用 `stats.input_tokens - stats.cached`。
旧版 JSON 覆盖设置仍会从 `response` 读取回复文本。

对于原生 OpenAI 系列的 Responses 流量，WebSocket/SSE 用量别名会以相同方式
标准化；如果 `total_tokens` 缺失或为 `0`，总量会回退为标准化后的输入量 + 输出量。

当当前会话快照信息不足时，`/status` 和 `session_status`
可以从最近的转录用量日志中恢复 token/缓存计数器和当前运行时模型标签。
现有的非零实时值仍优先于转录回退值；当存储的总量缺失或较小时，
面向提示词且数值更大的转录总量可以优先采用。

提供商配额窗口的用量身份验证会优先使用提供商专用钩子；
如果提供商没有钩子（或钩子无法解析出 token），OpenClaw
会回退到身份验证配置文件、环境变量或配置中匹配的 OAuth/API 密钥凭据。

助手转录条目会持久化相同的标准化用量结构，其中包括 `usage.cost`，
前提是当前模型配置了定价且提供商返回用量元数据。这样，即使实时
运行时状态已消失，`/usage cost` 和基于转录内容的会话状态仍有稳定的数据来源。

OpenClaw 会将提供商用量核算与当前上下文快照分开保存。提供商
`usage.total` 可能包括缓存输入、输出以及多次工具循环模型调用，
因此它适用于成本计算和遥测，但可能会高估实时上下文窗口。
上下文显示和诊断使用最新的提示词快照（`promptTokens`；若没有
提示词快照，则使用最后一次模型调用）来计算 `context.used`。

## 成本估算（显示时）

成本根据你的模型定价配置估算：

```text
models.providers.<provider>.models[].cost
```

这些值是 `input`、`output`、`cacheRead` 和
`cacheWrite` **每 100 万 token 的美元价格**。如果缺少定价，`/usage full` 会省略成本；
需要在每条回复中查看 token/缓存详情时，请使用
`/usage tokens` 或自定义 `messages.usageTemplate`。成本显示不限于 API 密钥
身份验证：当 `aws-sdk` 等非 API 密钥提供商的已配置模型条目中包含本地定价，
且提供商返回用量元数据时，也可以显示预估成本。

当 sidecar 和渠道进入 Gateway 网关就绪路径后，OpenClaw 会为尚未配置本地定价的
模型引用启动可选的后台定价引导流程。该流程会获取远程 OpenRouter 和
LiteLLM 定价目录。在离线或受限网络上，设置 `models.pricing.enabled: false` 可跳过这些
目录获取操作；显式的 `models.providers.*.models[].cost` 条目仍会用于本地成本估算。

## 缓存 TTL 和修剪的影响

提供商提示词缓存仅在缓存 TTL 窗口内有效。OpenClaw
可以选择运行 **缓存 TTL 修剪**：缓存 TTL 到期后，它会修剪会话，
然后重置缓存窗口，使后续请求重复使用新缓存的上下文，而不是重新缓存完整历史记录。
这可以在会话闲置时间超过 TTL 时降低缓存写入成本。

请在 [Gateway 配置](/zh-CN/gateway/configuration)中进行配置，并在
[会话修剪](/zh-CN/concepts/session-pruning)中查看行为详情。

Heartbeat 可以让缓存在闲置间隔期间保持**热状态**。如果模型缓存
TTL 为 `1h`，将 Heartbeat 间隔设置得略短于该值（例如 `55m`），可以
避免重新缓存完整提示词，从而降低缓存写入成本。

在多智能体设置中，可以共用一份模型配置，并通过
`agents.entries.*.params.cacheRetention` 按智能体调整缓存行为。

有关各项配置的完整指南，请参阅[提示词缓存](/zh-CN/reference/prompt-caching)。

对于 Anthropic API 定价，缓存读取的价格明显低于输入
token，而缓存写入则按更高的倍数计费。有关最新费率和 TTL 倍数，请参阅 Anthropic 的
提示词缓存定价：
[https://docs.anthropic.com/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/docs/build-with-claude/prompt-caching)

### 示例：使用 Heartbeat 保持 1h 缓存热状态

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
    heartbeat:
      every: "55m"
```

### 示例：采用按智能体缓存策略的混合流量

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long" # 大多数智能体的默认基准
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m" # 为深度会话保持长缓存预热
    - id: "alerts"
      params:
        cacheRetention: "none" # 避免为突发通知写入缓存
```

`agents.entries.*.params` 会合并到所选模型的 `params` 之上，因此你可以仅覆盖 `cacheRetention`，并原样继承其他模型默认值。

### Anthropic 1M 上下文

OpenClaw 为支持正式发布的 Claude 4.x 模型（例如 Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6）配置 Anthropic 的 1M 上下文窗口。这些模型不需要 `params.context1m: true`。

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        alias: opus
```

旧配置可以保留 `context1m: true`，但 OpenClaw 不再为此设置发送 Anthropic 已停用的 `context-1m-2025-08-07` beta 标头，也不会将不支持的旧版 Claude 模型扩展至 1M。

要求：凭据必须具备使用长上下文的资格。否则，Anthropic 会针对该请求返回提供商侧的速率限制错误。

如果使用 OAuth/订阅令牌（`sk-ant-oat-*`）进行 Anthropic 身份验证，OpenClaw 会保留 OAuth 所需的 Anthropic beta 标头，同时移除旧配置中仍然存在的已停用 `context-1m-*` beta。

## 减轻令牌压力的技巧

- 使用 `/compact` 汇总较长的会话。
- 在工作流中精简大型工具输出。
- 对于大量使用截图的会话，降低 `agents.defaults.imageMaxDimensionPx`。
- 保持技能描述简短（技能列表会注入提示词）。
- 对于输出冗长的探索性工作，优先使用较小的模型。

有关技能列表开销的精确计算公式，请参阅 [Skills](/zh-CN/tools/skills)。

## 相关内容

- [API 使用量和费用](/zh-CN/reference/api-usage-costs)
- [提示词缓存](/zh-CN/reference/prompt-caching)
- [用量跟踪](/zh-CN/concepts/usage-tracking)
