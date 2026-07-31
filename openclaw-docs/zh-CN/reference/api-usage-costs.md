---
read_when:
    - 你想了解哪些功能可能会调用付费 API
    - 你需要审计密钥、成本和使用情况的可见性
    - 你正在说明 `/status` 或 `/usage` 的费用报告功能
summary: 审计哪些功能可能产生费用、使用了哪些密钥，以及如何查看用量
title: API 使用与费用
x-i18n:
    generated_at: "2026-07-26T06:59:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 22caad8b8fa168739563223b3663a04adceeef7e83576a53dc9cdf885a35750d
    source_path: reference/api-usage-costs.md
    workflow: 16
---

OpenClaw 中可能调用付费提供商 API 的功能、各功能读取凭据的位置，以及相应费用的显示位置一览。

## 费用显示位置

**`/status`**（每会话快照）

- 显示当前会话的模型、上下文用量和上次响应的 token 数。
- 当 OpenClaw 具有当前模型的用量元数据和本地定价时，为上次回复添加**预估费用**，包括 Bedrock `aws-sdk` 模型等已明确设置价格且不使用 API key 的提供商。
- 如果实时会话快照中的数据不完整，`/status` 会从最新的会话记录用量条目中恢复 token/缓存计数器和当前模型标签。现有的非零实时值优先于会话记录数据；当存储的总量缺失或较小时，与提示词规模相当的会话记录总量仍可优先采用。

**`/usage`**（每消息页脚）

- `/usage full` 会为每条回复附加用量页脚；当已配置本地定价且用量元数据可用时，其中包括**预估费用**。
- `/usage tokens` 仅显示 token 数。订阅式 OAuth/token 和 CLI 运行时仅显示 token 数，除非它们提供兼容的用量元数据以及明确的本地价格。
- `/usage cost` 输出本地费用摘要；`/usage off` 禁用页脚。
- Gemini CLI 注意事项：`stream-json` 和旧版 `json` 输出均将用量放在 `stats` 下。OpenClaw 会将 `stats.cached` 规范化为 `cacheRead`，并在需要时根据 `stats.input_tokens - stats.cached` 推导输入 token 数。

**Control UI → Usage**（跨会话分析）

- 显示所选日期范围内根据会话记录推导出的 token 总量和预估费用总额，并按提供商、模型、智能体、渠道和 token 类型细分。
- 比较截至所选范围结束日期的较短日历时间窗口。缺失日期按零用量日历日计算；不会跳过这些日期来形成更密集的时间窗口。
- 直接标注每日图表的刻度。`√` 徽章表示正在使用平方根压缩，以保持低用量日期可见。
- 这些总量描述的是可用的本地会话历史记录，而不是提供商账单或终身计费账本。当部分条目缺少定价时，UI 会发出警告。

**CLI 用量窗口**（提供商配额，而非每消息费用）

- `openclaw status --usage` 和 `openclaw channels list` 将提供商的**用量窗口**显示为 `X% left`。
- 当前支持用量窗口的提供商包括：Anthropic、ClawRouter、DeepSeek、GitHub Copilot、Gemini CLI、MiniMax、OpenAI（涵盖 ChatGPT/Codex OAuth/token 身份验证）、Xiaomi 和 z.ai。完整的提供商/标志列表请参阅[模型 CLI](/zh-CN/cli/models)和[渠道 CLI](/zh-CN/cli/channels)。
- MiniMax 的原始 `usage_percent` / `usagePercent` 字段报告的是剩余配额，因此 OpenClaw 会将其反转；如果存在基于计数的字段，则优先使用这些字段。如果响应包含 `model_remains` 数组，OpenClaw 会选择聊天模型条目，在需要时根据时间戳推导窗口标签，并将模型名称包含在方案标签中。
- 如果提供商特定的钩子可用，用量身份验证将从这些钩子获取；否则，OpenClaw 会回退到从身份验证配置文件、环境变量或配置中匹配 OAuth/API key 凭据。

有关详细示例，请参阅 [Token 使用量和费用](/zh-CN/reference/token-use)。

<Note>
Anthropic 已确认，除非其发布新政策，否则复用 Claude CLI（包括 `claude -p`）属于获准的集成模式。Anthropic 不提供每消息的美元费用估算，因此 `/usage full` 无法显示 Claude CLI 用量的费用。
</Note>

## 密钥的发现方式

- **身份验证配置文件**：按智能体存储在 `auth-profiles.json` 中。
- **环境变量**：例如 `OPENAI_API_KEY`、`BRAVE_API_KEY`、`FIRECRAWL_API_KEY`。
- **配置**：`models.providers.*.apiKey`、`plugins.entries.*.config.webSearch.apiKey`、`plugins.entries.firecrawl.config.webFetch.apiKey`、`memory.search.*`、`talk.providers.*.apiKey`。
- **Skills**：`skills.entries.<name>.apiKey`，可能会将密钥导出到技能进程的环境变量中。

## 可能使用密钥并产生费用的功能

### 核心模型响应（聊天 + 工具）

每次回复或工具调用都在当前模型提供商上运行。这是用量和费用的主要来源，也包括在 OpenClaw 本地 UI 之外计费的订阅式托管方案：OpenAI Codex、Alibaba Cloud Model Studio Coding Plan、MiniMax Coding Plan、Z.AI/GLM Coding Plan，以及启用了 Extra Usage 的 Anthropic Claude 登录路径。

有关定价配置，请参阅[模型](/zh-CN/providers/models)；有关显示方式，请参阅 [Token 使用量和费用](/zh-CN/reference/token-use)。

### 媒体理解（音频/图像/视频）

在回复流水线运行之前，可以通过提供商 API 对入站媒体进行摘要或转录。提供商支持按插件注册，并会随插件的增加而变化；有关当前列表和配置，请参阅[媒体理解](/zh-CN/nodes/media-understanding)。

### 图像和视频生成

`image_generate` 和 `video_generate` 会路由到任一可用的已身份验证提供商。当各自的 `agents.defaults.mediaModels` 条目未设置时，两者都可以推断一个由身份验证支持的默认提供商。

有关当前提供商列表，请参阅[图像生成](/zh-CN/tools/image-generation)和[视频生成](/zh-CN/tools/video-generation)。

### 记忆嵌入和语义搜索

当 `memory.search.provider` 指定远程适配器（例如 `openai`、`gemini`、`voyage`、`mistral`、`deepinfra`、`github-copilot`、`amazon-bedrock`）时，语义记忆搜索会使用嵌入 API。`memory.search.provider = "lmstudio"` 或 `"ollama"` 在本地/自托管服务器上运行，通常不会产生托管服务费用。`memory.search.provider = "local"` 将所有处理保留在设备上，不使用 API。可选的 `memory.search.fallback` 提供商可用于处理本地嵌入失败。

请参阅[记忆](/zh-CN/concepts/memory)。

### Web 搜索工具

根据所选提供商，`web_search` 可能产生用量费用。每个提供商会先从环境变量读取密钥，然后读取 `plugins.entries.<id>.config.webSearch.apiKey`：

| 提供商                 | 环境变量                                                                                                                                                               |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Brave Search           | `BRAVE_API_KEY`                                                                                                                                                     |
| DuckDuckGo             | 无需密钥；非官方、基于 HTML，不计费                                                                                                                                   |
| Exa                    | `EXA_API_KEY`                                                                                                                                                     |
| Firecrawl              | `FIRECRAWL_API_KEY`                                                                                                                                                     |
| Gemini (Google Search) | `GEMINI_API_KEY`                                                                                                                                                     |
| Grok (xAI)             | xAI OAuth 配置文件或 `XAI_API_KEY`                                                                                                                               |
| Kimi (Moonshot)        | `KIMI_API_KEY` 或 `MOONSHOT_API_KEY`                                                                                                                              |
| MiniMax Search         | `MINIMAX_CODE_PLAN_KEY`、`MINIMAX_CODING_API_KEY`、`MINIMAX_OAUTH_TOKEN` 或 `MINIMAX_API_KEY`                                                                                       |
| Ollama Web Search      | 对可访问且已登录的本地主机无需密钥；直接使用 `https://ollama.com` 搜索时使用 `OLLAMA_API_KEY`；受身份验证保护的主机会复用常规 Ollama 提供商的 bearer 身份验证 |
| Parallel               | `PARALLEL_API_KEY`                                                                                                                                                     |
| Perplexity Search API  | `PERPLEXITY_API_KEY` 或 `OPENROUTER_API_KEY`                                                                                                                              |
| SearXNG                | `SEARXNG_BASE_URL`；无需密钥/自托管，不产生托管服务费用                                                                                                               |
| Tavily                 | `TAVILY_API_KEY`                                                                                                                                                     |

旧版 `tools.web.search.*` 配置路径仍会通过兼容性垫片加载，但已不再是推荐的配置界面。

**Brave Search 免费额度**：每个方案每月包含续期的 $5 免费额度。Search 方案每 1,000 次请求收费 $5，因此该额度可免费覆盖每月 1,000 次请求。请在 Brave 控制面板中设置用量上限，以避免意外费用。

请参阅 [Web 工具](/zh-CN/tools/web)。

### Web 获取工具（Firecrawl）

`web_fetch` 可以使用无需密钥的入门访问调用 Firecrawl；添加 `FIRECRAWL_API_KEY`（或 `plugins.entries.firecrawl.config.webFetch.apiKey`）可获得更高限额。如果未配置 Firecrawl，该工具会回退到直接获取，并使用内置的 `web-readability` 插件（无付费 API）。禁用 `plugins.entries.web-readability.enabled` 可跳过本地 Readability 提取。

请参阅 [Web 工具](/zh-CN/tools/web)。

### 提供商用量快照（状态/健康）

`openclaw status --usage` 和 `openclaw models status --json` 会调用提供商用量端点，以显示配额窗口或身份验证健康状况。调用量较低，但仍会访问提供商 API。

请参阅[模型 CLI](/zh-CN/cli/models)。

### 压缩保护摘要

压缩保护机制可以使用当前模型汇总会话历史记录，运行时会调用提供商 API。

请参阅[会话管理和压缩](/zh-CN/reference/session-management-compaction)。

### 模型扫描/探测

启用探测后，`openclaw models scan` 可以探测 OpenRouter 模型，并使用 `OPENROUTER_API_KEY`。

请参阅[模型 CLI](/zh-CN/cli/models)。

### Talk（语音）

配置后，Talk 模式可以调用 ElevenLabs：`ELEVENLABS_API_KEY` 或 `talk.providers.elevenlabs.apiKey`。

请参阅 [Talk 模式](/zh-CN/nodes/talk)。

### Skills（第三方 API）

Skills 可以将 `apiKey` 存储在 `skills.entries.<name>.apiKey` 中。如果某个技能使用该密钥访问外部 API，费用将由该技能的提供商决定。

请参阅 [Skills](/zh-CN/tools/skills)。

## 相关内容

- [Token 使用量和费用](/zh-CN/reference/token-use)
- [提示词缓存](/zh-CN/reference/prompt-caching)
- [用量跟踪](/zh-CN/concepts/usage-tracking)
