---
read_when:
    - 你正在调试与对话记录结构相关的提供商请求拒绝问题
    - 你正在更改转录内容清理或工具调用修复逻辑
    - 你正在调查不同提供商之间的工具调用 ID 不匹配问题
summary: 参考：提供商特定的转录清理和修复规则
title: 记录清理
x-i18n:
    generated_at: "2026-07-26T06:23:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 33d978772062cb2a81eb358bb5c62bd1261b433ffdc8acdbaa6679b121fbbf62
    source_path: reference/transcript-hygiene.md
    workflow: 16
---

OpenClaw 会在运行前（构建模型上下文时）对转录记录应用**提供商特定的修复**。其中大多数是为满足提供商的严格要求而进行的**内存中**调整。单独的会话文件修复流程也可能在加载会话前重写已存储的 JSONL，但仅限于格式错误的行或不属于有效持久记录的已持久化轮次。已交付的智能体回复会保留在磁盘上；提供商特定的智能体预填充剥离仅在构建出站载荷时进行。

发生修复时，原始文件会在原子替换前写入临时的
`*.bak-<pid>-<ts>` 同级文件，并在替换成功后移除。仅当清理本身失败时才保留备份，在这种情况下会返回其路径。

范围包括：

- 仅限运行时的提示上下文不会进入用户可见的转录轮次
- 工具调用 ID 清理
- 工具调用输入验证
- 工具结果配对修复
- 轮次验证/排序
- 思维签名清理
- 思考签名清理
- 图像载荷清理
- 提供商重放前的空白文本块清理
- 提供商重放前的不完整纯推理长度轮次清理
- 用户输入来源标记（用于跨会话路由的提示）
- 用于 Bedrock Converse 重放的空智能体错误轮次修复

如需了解转录记录存储详情，请参阅
[会话管理深入解析](/zh-CN/reference/session-management-compaction)。

---

## 全局规则：运行时上下文不是用户转录记录

可以为某一轮次向模型提示添加运行时/系统上下文，但它不是最终用户编写的内容。OpenClaw 会为 Gateway 网关回复、排队的后续消息、ACP、CLI 和嵌入式 OpenClaw 运行单独保留一个面向转录记录的提示正文。存储的可见用户轮次使用该转录正文，而不是运行时增强后的提示。

对于已经持久化运行时包装器的旧版会话，Gateway 网关历史记录界面会在向 WebChat、TUI、REST 或 SSE 客户端返回消息前应用显示投影。

---

## 运行位置

所有转录记录清理均集中在嵌入式运行器中：

- 策略选择：`src/agents/transcript-policy.ts`
  （`resolveTranscriptPolicy`，以 `provider`、`modelApi` 和 `modelId` 为键）
- 清理/修复应用：`src/agents/embedded-agent-runner/replay-history.ts` 中的 `sanitizeSessionHistory`

会话文件修复独立于转录记录清理，在加载前按需执行：

- `src/agents/session-file-repair.ts` 中的 `repairSessionFileIfNeeded`
- 由 `src/agents/embedded-agent-runner/run/attempt.ts` 和
  `src/agents/embedded-agent-runner/compact.ts` 调用

---

## 全局规则：图像清理

始终清理图像载荷，以避免因大小限制导致提供商拒绝（对过大的 base64 图像进行缩小/重新压缩）。这也有助于控制支持视觉的模型中由图像产生的 token 压力：较低的最大尺寸可减少 token 用量，较高的尺寸则可保留细节。

实现：

- `src/agents/embedded-agent-helpers/images.ts` 中的
  `sanitizeSessionMessagesImages`
- `src/agents/tool-images.ts` 中的 `sanitizeContentBlocksImages`
- 图像最大边长可通过 `agents.defaults.imageMaxDimensionPx` 配置
  （默认值：`1200`）
- 此流程遍历重放内容时会移除空白文本块。
  因此变为空的智能体轮次会从重放副本中丢弃；变为空的用户轮次和工具结果轮次会收到一个非空的内容已省略占位符。

---

## 全局规则：格式错误的工具调用

在构建模型上下文前，会丢弃同时缺少 `input` 和 `arguments` 的智能体工具调用块。这样可避免提供商因部分持久化的工具调用而拒绝请求（例如在发生速率限制失败后）。

实现：

- `src/agents/session-transcript-repair.ts` 中的 `sanitizeToolCallInputs`
- 应用于 `sanitizeSessionHistory`
  （`src/agents/embedded-agent-runner/replay-history.ts`）

---

## 全局规则：工具结果配对

在重写提供商特定的调用 ID 前，会将工具结果与每个智能体轮次中的工具调用实例配对。提供商生成的 ID 可能在后续轮次中重复，因此与重复调用相邻的结果会保留给该调用实例。仅当恰好有一个尚未解析的调用实例可接收错位结果时，才会移动该结果；存在歧义的多余结果会被丢弃，缺少结果的调用实例则会收到合成错误结果。

实现：`src/agents/session-transcript-repair.ts` 中的
`sanitizeToolUseResultPairing`

---

## 全局规则：不完整或静默的纯推理轮次

发生以下任一事件后，如果智能体轮次仅包含思考或已隐去的思考内容，则会从内存中的重放副本中省略：

- 提供商输出限制使轮次以不完整的推理状态结束。
- 静默回复清理移除了该轮次唯一可见的 `NO_REPLY` 文本。

静默回复清理可防止严格的提供商重建对话时，将隐藏推理合并到后续智能体工具使用轮次中。

空的长度轮次保持不变，包含可见文本、工具调用或未知内容块的长度轮次也保持不变。包含工具调用或未知内容块的静默回复轮次同样保持不变。不会重写已存储的转录记录。

实现：`src/agents/embedded-agent-runner/replay-history.ts` 中的
`normalizeAssistantReplayContent`

---

## 全局规则：跨会话输入来源

当智能体通过 `sessions_send` 向另一个会话发送提示时（包括智能体到智能体的回复/通知步骤），OpenClaw 会使用 `message.provenance.kind = "inter_session"` 持久化所创建的用户轮次。

OpenClaw 还会在路由后的提示文本前添加同轮次的 `[Inter-session message] ... isUser=false`
标记，以便当前模型调用区分来自其他会话的输出与外部最终用户指令。该标记包含源会话、渠道和工具（如可用）。为兼容提供商，转录记录仍使用 `role: "user"`，但可见文本和来源元数据都会将该轮次标记为跨会话数据。

重建上下文期间，OpenClaw 会对仅包含来源元数据的旧版已持久化跨会话用户轮次应用相同标记。

---

## 提供商矩阵（当前行为）

**OpenAI / OpenAI Codex**

- 仅清理图像。
- 对于 OpenAI Responses/Codex 转录记录，丢弃孤立的推理签名（后面没有内容块的独立推理项）；切换模型路由后，丢弃可重放的 OpenAI 推理。
- 保留可重放的 OpenAI Responses 推理项载荷，包括加密的空摘要项，以便手动/WebSocket 重放使所需的 `rs_*` 状态与智能体输出项保持配对。
- 原生 ChatGPT Codex Responses 会重放先前的 Responses 推理/消息/函数载荷而不包含先前的项 ID，同时保留会话 `prompt_cache_key`，从而与 Codex 线路保持一致。
- OpenAI Responses 系列重放会保留规范的 `call_*|fc_*` 同模型推理对，但会在转换为 pi-ai 载荷前，以确定性方式规范化格式错误或过长的 `call_id`/函数调用项 ID。
- 工具结果配对修复可能会移动真实的匹配输出，并为缺少结果的工具调用合成 Codex 风格的 `aborted` 输出。
- 不进行轮次验证或重新排序；不剥离思维签名。

**OpenAI 兼容的 Chat Completions**

- 重放前会剥离历史智能体思考/推理块，以免本地及代理式 OpenAI 兼容服务器收到 `reasoning` 或 `reasoning_content` 等先前轮次的推理字段。
- 当前同轮次工具调用的延续内容会使智能体推理块保持附加到工具调用，直至工具结果完成重放。
- 带有 `reasoning: true` 的自定义/自托管模型条目会保留重放的推理元数据。
- 当线路协议要求重放推理元数据时，提供商自有的例外情况可以选择退出此行为。

**Google（Generative AI / Gemini CLI / Antigravity）**

- 工具调用 ID 清理：严格限制为字母数字字符。
- 工具结果配对修复和合成工具结果。
- 轮次验证（Gemini 风格的轮次交替）。
- Google 轮次排序修复（如果历史记录以智能体开始，则在前面添加一条极短的用户引导消息）。
- Antigravity Claude：规范化思考签名；丢弃未签名的思考块。

**Anthropic / Minimax（兼容 Anthropic）**

- 工具结果配对修复和合成工具结果。
- 轮次验证（合并连续的用户轮次以满足严格交替要求）。
- 启用思考时，会从出站 Anthropic Messages 载荷中剥离末尾的智能体预填充轮次，包括 Cloudflare AI Gateway 网关路由。
- 会话经过压缩后，会在提供商重放前剥离压缩前的智能体思考签名。思考签名在生成时以加密方式绑定到对话前缀；压缩后，前缀会发生变化（摘要内容取代原始内容），因此重放原始签名会导致 Anthropic 以“Invalid signature in thinking block”为由拒绝请求。思考文本会保留为未签名块，然后由下方规则处理。
- 在转换为提供商格式前，会剥离重放签名缺失、为空或仅含空白字符的思考块。如果这使智能体轮次变为空，OpenClaw 会使用非空的推理已省略文本来保持轮次结构。
- 必须剥离的旧版纯思考智能体轮次会替换为非空的推理已省略文本，以免提供商适配器丢弃该重放轮次。

**Amazon Bedrock（Converse API）**

- 重放前，会将空的智能体流错误轮次修复为非空的回退文本块。Bedrock Converse 会拒绝包含 `content: []` 的智能体消息，因此带有 `stopReason:
"error"` 且内容为空的已持久化智能体轮次也会在加载前于磁盘上修复。
- 仅包含空白文本块的智能体流错误轮次会从内存中的重放副本中丢弃，而不是重放无效的空白块。
- 会话经过压缩后，会在 Converse 重放前剥离压缩前的智能体思考签名，原因与上文 Anthropic 相同。
- 在 Converse 重放前，会剥离重放签名缺失、为空或仅含空白字符的 Claude 思考块。如果这使智能体轮次变为空，OpenClaw 会使用非空的推理已省略文本来保持轮次结构。
- 必须剥离的旧版纯思考智能体轮次会替换为非空的推理已省略文本，使 Converse 重放保持严格的轮次结构。
- 重放会过滤 OpenClaw 交付镜像和 Gateway 网关注入的智能体轮次。
- 按照全局规则应用图像清理。

**Mistral（包括基于模型 ID 的检测）**

- 工具调用 ID 清理：strict9（字母数字字符，长度为 9）。

**OpenRouter Gemini**

- 思维签名清理：剥离非 base64 的 `thought_signature` 值（保留 base64）。

**OpenRouter Anthropic**

- 启用推理时，会从经过验证的 OpenRouter OpenAI 兼容 Anthropic 模型载荷中剥离末尾的智能体预填充轮次，与直接使用 Anthropic 和 Cloudflare Anthropic 时的重放行为保持一致。

**其他所有情况**

- 仅清理图像。

---

## 历史行为（2026.1.22 之前）

在 2026.1.22 版本之前，OpenClaw 会应用多层转录记录清理：

- 一个 **transcript-sanitize 扩展**会在每次构建上下文时运行，并且可以：
  - 修复工具使用与结果的配对。
  - 清理工具调用 ID（包括一种保留
    `_`/`-` 的非严格模式）。
- 运行器还会执行特定于提供商的清理，从而
  造成重复处理。
- 提供商策略之外还会发生其他变更，包括
  在持久化之前从助手文本中移除 `<final>` 标签、丢弃
  空的助手错误轮次，以及截断工具
  调用之后的助手内容。

这种复杂性导致了跨提供商回归（尤其是
`openai-responses` `call_id|fc_id` 配对问题）。2026.1.22 的清理移除了
该扩展，将逻辑集中到运行器中，并使 OpenAI 除图像清理外保持**不作改动**。

## 相关内容

- [会话管理](/zh-CN/concepts/session)
- [会话修剪](/zh-CN/concepts/session-pruning)
