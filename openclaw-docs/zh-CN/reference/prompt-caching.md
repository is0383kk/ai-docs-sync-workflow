---
read_when:
    - 你希望通过保留缓存来降低提示词的 token 成本
    - 在多 Agent 设置中，你需要按 Agent 配置缓存行为
    - 你正在同时调优 Heartbeat 和缓存 TTL 清理机制
summary: 提示词缓存选项、合并顺序、提供商行为和调优模式
title: 提示词缓存
x-i18n:
    generated_at: "2026-07-26T07:01:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 99dfd3d226d37014110adf16818051236114dcb0277e9b4d13eaced0f1fc03aa
    source_path: reference/prompt-caching.md
    workflow: 16
---

提示词缓存允许模型提供商跨轮次复用未更改的提示词前缀（系统/开发者指令、工具定义及其他稳定上下文），而不必在每次请求时重新处理。这可以降低包含重复上下文的长期会话的 token 成本和延迟。

只要上游 API 提供相应计数器，OpenClaw 就会将提供商的用量统一规范化为 `cacheRead` 和 `cacheWrite`。当实时会话快照缺少缓存计数器时，用量摘要（`/status` 及类似摘要）会回退到最后一条转录用量记录；非零的实时值始终优先于回退值。

提供商参考资料：

- [Anthropic 提示词缓存](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [OpenAI 提示词缓存](https://developers.openai.com/api/docs/guides/prompt-caching)

## 主要调节项

### `cacheRetention`

可选值：`"none" | "short" | "long"`。可配置为全局默认值、按模型配置以及按智能体配置。
`"standard"` 不是别名；要使用提供商的默认缓存窗口，请使用 `"short"`。无效值会被忽略并产生警告。

```yaml
agents:
  defaults:
    params:
      cacheRetention: "long" # none | short | long
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "short" # 覆盖此模型的全局默认值
  list:
    - id: "alerts"
      params:
        cacheRetention: "none" # 覆盖此智能体的两个默认值
```

合并顺序（后者优先）：

1. `agents.defaults.params` - 所有模型的全局默认值
2. `agents.defaults.models["provider/model"].params` - 按模型覆盖
3. `agents.entries.*.params` - 按智能体覆盖，通过智能体 ID 匹配

来源：`src/agents/embedded-agent-runner/extra-params.ts`（`resolveExtraParams`）。

### `contextPruning.mode: "cache-ttl"`

在缓存 TTL 窗口到期后剪除旧的工具结果上下文，使空闲后的请求不会重新缓存过大的历史记录。

```yaml
agents:
  defaults:
    contextPruning:
      mode: "cache-ttl"
      ttl: "1h"
```

完整行为请参阅[会话剪枝](/zh-CN/concepts/session-pruning)。

### Heartbeat 保温

Heartbeat 可以保持缓存窗口活跃，并减少空闲间隔后重复写入缓存的次数。可全局配置（`agents.defaults.heartbeat`）或按智能体配置（`agents.entries.*.heartbeat`）。

```yaml
agents:
  defaults:
    heartbeat:
      every: "55m"
```

## 提供商行为

### Anthropic（直接 API 和 Vertex AI）

- `cacheRetention` 支持 `anthropic` 和 `anthropic-vertex` 提供商；当显式设置 `cacheRetention` 时，也支持 `amazon-bedrock` 上的 Claude 模型以及自定义的 `anthropic-messages` 兼容端点。
- 未设置时，OpenClaw 会为直接 Anthropic 播种 `cacheRetention: "short"`（仅限 `anthropic` 和 `anthropic-vertex` 提供商；其他 Anthropic 系列路由需要显式值）。
- 原生 Anthropic Messages 响应会提供 `cache_read_input_tokens` 和 `cache_creation_input_tokens`，分别映射到 `cacheRead` 和 `cacheWrite`。
- `cacheRetention: "short"` 映射到默认的 5 分钟临时缓存。显式设置时，`cacheRetention: "long"` 会请求 1 小时 TTL（`cache_control: { type: "ephemeral", ttl: "1h" }`）。隐式/由环境驱动的长期保留（`OPENCLAW_CACHE_RETENTION=long`，且未显式设置 `cacheRetention`）仅会在 `api.anthropic.com` 或 Vertex AI（`aiplatform.googleapis.com` / `*-aiplatform.googleapis.com`）主机上升级为 1 小时 TTL；其他主机仍使用 5 分钟缓存。

来源：`packages/ai/src/transports/anthropic-payload-policy.ts`（`resolveAnthropicEphemeralCacheControl`、`isLongTtlEligibleEndpoint`）。

### OpenAI（直接 API）

- 受支持的较新模型会自动使用提示词缓存；OpenClaw 不会注入块级缓存标记。
- OpenClaw 会发送 `prompt_cache_key`，以保持跨轮次缓存路由稳定。直接的 `api.openai.com` 主机会自动获得此设置。OpenAI 兼容代理（oMLX、llama.cpp、自定义端点）需要在模型配置中设置 `compat.supportsPromptCacheKey: true` 才能选择启用——代理绝不会被自动检测。
- 仅当选择 `cacheRetention: "long"`，且解析后的端点同时支持缓存键和长期保留（`compat.supportsLongCacheRetention`，默认为 true；Together AI 和 Cloudflare 兼容配置文件会禁用它）时，才会添加 `prompt_cache_retention: "24h"`。`cacheRetention: "none"` 会抑制这两个字段。
- 缓存命中通过 `usage.prompt_tokens_details.cached_tokens`（Chat Completions）或 `input_tokens_details.cached_tokens`（Responses API）呈现，并映射到 `cacheRead`。
- Responses API 载荷还可能提供 `input_tokens_details.cache_write_tokens`，它会映射到 `cacheWrite`，并按模型的缓存写入费率计价；省略该字段的 Responses 载荷会将 `cacheWrite` 保持为 `0`。OpenAI 的 Chat Completions API 未记录也不会提供 `cache_write_tokens` 计数器，但 OpenClaw 仍会在其中读取 `prompt_tokens_details.cache_write_tokens`，以支持会报告独立写入计数的 OpenRouter 兼容代理和 DeepSeek 风格代理。
- 实际使用中，OpenAI 的行为更接近初始前缀缓存，而不是 Anthropic 对动态完整历史记录的复用——请参阅下文的 [OpenAI 实时预期](#openai-live-expectations)。

### Amazon Bedrock

- Anthropic Claude 模型引用（`amazon-bedrock/*anthropic.claude*`，以及 AWS 系统推理配置文件前缀 `us.`/`eu.`/`global.anthropic.claude*`）支持显式透传 `cacheRetention`。
- 非 Anthropic Bedrock 模型（例如 `amazon.nova-*`）在运行时会解析为不保留缓存，无论配置了何种 `cacheRetention` 值。
- 不透明的 Bedrock 应用程序推理配置文件 ARN（配置文件 ID 不含 `claude`）也会解析为不保留缓存，除非显式设置 `cacheRetention`，因为无法仅从 ARN 推断模型系列。

### OpenRouter

对于 `openrouter/anthropic/*` 模型引用，OpenClaw 会在系统/开发者提示词块上注入 Anthropic `cache_control` 标记，但仅限请求仍指向已验证的 OpenRouter 路由（默认端点上的 `openrouter`，或任何解析为 `openrouter.ai` 的提供商/基础 URL）。将模型重新指向任意 OpenAI 兼容代理 URL 会停止此注入。

`contextPruning.mode: "cache-ttl"` 可用于 `openrouter/anthropic/*`、`openrouter/deepseek/*`、`openrouter/moonshot/*`、`openrouter/moonshotai/*` 和 `openrouter/zai/*` 模型引用，因为这些路由会在提供商侧处理提示词缓存，无需 OpenClaw 注入标记。

来源：`extensions/openrouter/index.ts`（`OPENROUTER_CACHE_TTL_MODEL_PREFIXES`）。

OpenRouter 上的 DeepSeek 缓存构建采用尽力而为的方式，可能需要几秒钟；紧接着发出的后续请求仍可能显示 `cached_tokens: 0`。请短暂延迟后使用相同前缀重复请求，并以 `usage.prompt_tokens_details.cached_tokens` 作为缓存命中信号进行验证。

### Google Gemini（直接 API）

- 直接 Gemini 传输（`api: "google-generative-ai"`）通过上游 `cachedContentTokenCount` 报告缓存命中，并映射到 `cacheRead`。
- 符合条件的模型系列：`gemini-2.5*` 和 `gemini-3*`（不包括该前缀匹配范围之外的 Live/预览变体，例如 `gemini-live-2.5-flash-preview`）。
- 在符合条件的模型上设置 `cacheRetention` 后，OpenClaw 会自动为系统提示词创建、复用和刷新 `cachedContents` 资源——无需手动提供缓存内容句柄。`cacheRetention: "short"` 的 TTL 为 `300s`，`"long"` 的 TTL 为 `3600s`。
- 仍可通过 `params.cachedContent`（或旧版 `params.cached_content`）传入已有的 Gemini 缓存内容句柄；显式句柄会完全跳过自动缓存管理路径。
- 这与 Anthropic/OpenAI 的提示词前缀缓存不同：OpenClaw 会为 Gemini 管理提供商原生的 `cachedContents` 资源，而不是注入内联缓存标记。

来源：`src/agents/embedded-agent-runner/google-prompt-cache.ts`。

### CLI harness 提供商（Claude Code、Gemini CLI）

会生成 JSONL 用量事件（`jsonlDialect: "claude-stream-json"` 或 `"gemini-stream-json"`）的 CLI 后端会通过共享用量解析器处理；该解析器可识别多种字段名称变体，包括映射到 `cacheRead` 的普通 `cached` 计数器。当 CLI 的 JSON 载荷省略直接输入 token 字段时，OpenClaw 会将其推导为 `input_tokens - cached`。这仅是用量规范化——不会为这些由 CLI 驱动的模型创建 Anthropic/OpenAI 风格的提示词缓存标记。

来源：`src/agents/cli-output.ts`（`toCliUsage`）。

### 其他提供商

如果提供商不支持上述任何缓存模式，`cacheRetention` 不会产生任何效果。

## 系统提示词缓存边界

OpenClaw 会在内部缓存前缀边界处将系统提示词拆分为**稳定前缀**和**易变后缀**。边界上方的内容（工具定义、Skills 元数据、工作区文件）会按特定顺序排列，以保持跨轮次的字节完全一致。边界下方的内容（例如 `HEARTBEAT.md`、运行时时间戳及其他每轮元数据）可以发生变化，而不会使缓存前缀失效。

关键设计选择：

- 稳定的工作区项目上下文文件排在 `HEARTBEAT.md` 之前，因此 Heartbeat 的频繁变化不会破坏稳定前缀。
- 该边界适用于 Anthropic 系列、OpenAI 系列、Google 和 CLI 传输载荷塑形，因此所有受支持的提供商都能受益于相同的前缀稳定性。
- Codex Responses 和 Anthropic Vertex 请求会通过感知边界的缓存载荷塑形进行路由，使缓存复用与提供商实际收到的内容保持一致。
- 系统提示词指纹会进行规范化（空白字符、行尾、钩子添加的上下文、运行时能力排序），因此语义未变的提示词可以跨轮次共享缓存。

如果配置或工作区发生变化后出现意外的 `cacheWrite` 激增，请检查该变化位于缓存边界上方还是下方。将易变内容移到边界下方（或使其稳定）通常可以解决该问题。

## OpenClaw 缓存稳定性防护

- 内置 MCP 工具目录会在工具注册前按确定性顺序排序（先按服务器名称，再按工具名称），因此 `listTools()` 的顺序变化不会导致工具块频繁变化并破坏提示词缓存前缀。
- 包含持久化图像块的旧版会话会完整保留**最近 3 个已完成轮次**（计算所有已完成轮次，而不仅是包含图像的轮次）。更早且已经处理的图像块会替换为文本标记，使包含大量图像的后续请求不会持续重新发送庞大的陈旧载荷。

## 调优模式

### 混合流量（推荐默认设置）

在主智能体上保留长期基线，在突发式通知智能体上禁用缓存：

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m"
    - id: "alerts"
      params:
        cacheRetention: "none"
```

### 成本优先基线

- 将基线设置为 `cacheRetention: "short"`。
- 启用 `contextPruning.mode: "cache-ttl"`。
- 仅对能从热缓存中受益的智能体，将 Heartbeat 间隔保持在 TTL 以内。

## 实时回归测试

OpenClaw 运行一个组合式实时缓存回归门禁，涵盖重复前缀、工具轮次、图像轮次、MCP 风格工具转录，以及一个 Anthropic 无缓存对照。

- `src/agents/live-cache-regression.live.test.ts`
- `src/agents/live-cache-regression-runner.ts`
- `src/agents/live-cache-regression-baseline.ts`

使用以下命令运行：

```sh
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache
```

基线文件存储最近一次观测到的实时数值，以及测试用来检查回归的各提供商专属下限。每次运行都会使用全新的单次运行会话 ID 和提示词命名空间，因此之前的缓存状态不会污染当前样本。Anthropic 和 OpenAI 采用不同的执行方式：Anthropic 未达到下限属于硬回归（测试失败），而 OpenAI 未达到下限仅用于监测（记录为警告，但不会导致运行失败）。两者并不共用一个跨提供商阈值。

### Anthropic 实时预期

- 预期通过 `cacheWrite` 执行显式预热写入。
- 预期在重复轮次中复用几乎完整的历史记录，因为 Anthropic 的缓存控制会在对话过程中推进缓存断点。
- 稳定、工具、图像和 MCP 风格通道的基线下限均为硬回归门槛。

### OpenAI 实时预期

- 仅预期 `cacheRead`；在 Chat Completions 中，`cacheWrite` 保持为 `0`。
- 将重复轮次的缓存复用视为提供商特有的平台期，而不是 Anthropic 风格的移动式完整历史记录复用。
- 下限仅用于监测（未达到下限会记录为警告，而不会导致测试失败），基于在 `gpt-5.4-mini` 上观测到的实时行为得出：

| 场景                 | `cacheRead` 下限 | 命中率下限 |
| -------------------- | ----------------: | -------------: |
| 稳定前缀             |             4,608 |           0.90 |
| 工具转录记录         |             4,096 |           0.85 |
| 图像转录记录         |             3,840 |           0.82 |
| MCP 风格转录记录     |             4,096 |           0.85 |

最近一次观测到的基线数值（来自 `live-cache-regression-baseline.ts`）为：稳定前缀 `cacheRead=4864`，命中率 `0.966`；工具转录记录 `cacheRead=4608`，命中率 `0.896`；图像转录记录 `cacheRead=4864`，命中率 `0.954`；MCP 风格转录记录 `cacheRead=4608`，命中率 `0.891`。

断言不同的原因：Anthropic 会公开显式缓存断点，并以移动方式复用对话历史记录；而在实时流量中，OpenAI 的实际可复用前缀可能在达到完整提示词之前便进入平台期。使用单一的跨提供商百分比阈值比较这两个提供商会产生误报回归。

## `diagnostics.cacheTrace` 配置

```yaml
diagnostics:
  cacheTrace:
    enabled: true
    filePath: "~/.openclaw/logs/cache-trace.jsonl" # 可选
    includeMessages: false # 默认为 true
    includePrompt: false # 默认为 true
    includeSystem: false # 默认为 true
```

默认值：

| 键                | 默认值                                       |
| ----------------- | -------------------------------------------- |
| `filePath`        | `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl` |
| `includeMessages` | `true`                                       |
| `includePrompt`   | `true`                                       |
| `includeSystem`   | `true`                                       |

### 环境变量开关（一次性调试）

| 变量                                 | 效果                                 |
| ------------------------------------ | ------------------------------------ |
| `OPENCLAW_CACHE_TRACE=1`             | 启用缓存跟踪                         |
| `OPENCLAW_CACHE_TRACE_FILE=path`     | 覆盖输出路径                         |
| `OPENCLAW_CACHE_TRACE_MESSAGES=0\|1` | 切换完整消息载荷捕获                 |
| `OPENCLAW_CACHE_TRACE_PROMPT=0\|1`   | 切换提示词文本捕获                   |
| `OPENCLAW_CACHE_TRACE_SYSTEM=0\|1`   | 切换系统提示词捕获                   |

### 检查内容

- 缓存跟踪事件采用 JSONL 格式，并包含 `session:loaded`、`prompt:before`、`stream:context` 和 `session:after` 等分阶段快照。
- 每轮缓存 Token 的影响可在常规用量界面中查看：`cacheRead` 和 `cacheWrite` 会显示在 `/usage tokens`、`/status`、会话用量摘要以及自定义 `messages.usageTemplate` 布局中。
- 对于 Anthropic，启用缓存时应同时出现 `cacheRead` 和 `cacheWrite`。
- 对于 OpenAI，缓存命中时应出现 `cacheRead`；只有在 Responses API 载荷中包含 `cacheWrite` 时，该字段才会被填充（请参阅上方的 [OpenAI](#openai-direct-api)）。
- OpenAI 还会返回 `x-request-id`、`openai-processing-ms` 和 `x-ratelimit-*` 等跟踪及速率限制标头；使用这些标头跟踪请求，但缓存命中统计仍应来自用量载荷，而不是标头。

## 快速故障排除

- **大多数轮次中的 `cacheWrite` 较高**：检查系统提示词中是否存在易变输入；确认模型/提供商支持你的缓存设置。
- **Anthropic 上的 `cacheWrite` 较高**：通常意味着缓存断点落在每次请求都会变化的内容上。
- **OpenAI `cacheRead` 较低**：确认稳定前缀位于最前面、重复前缀至少为 1024 个 Token，并且本应共享缓存的轮次复用了同一个 `prompt_cache_key`。
- **`cacheRetention` 未生效**：确认模型键与 `agents.defaults.models["provider/model"]` 匹配。
- **带有缓存设置的 Bedrock Nova 请求**：这是预期行为——这些请求在运行时不会保留缓存。

相关文档：

- [Anthropic](/zh-CN/providers/anthropic)
- [Token 用量和成本](/zh-CN/reference/token-use)
- [会话裁剪](/zh-CN/concepts/session-pruning)
- [Gateway 配置参考](/zh-CN/gateway/configuration-reference)

## 相关内容

- [Token 用量和成本](/zh-CN/reference/token-use)
- [API 用量和成本](/zh-CN/reference/api-usage-costs)
