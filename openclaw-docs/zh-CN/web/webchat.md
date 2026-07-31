---
read_when:
    - 调试或配置 WebChat 访问权限
summary: 用于聊天 UI 的本地回环 WebChat 静态托管和 Gateway 网关 WebSocket 用法
title: WebChat
x-i18n:
    generated_at: "2026-07-26T07:06:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 19c301af1eb1b28650849cdd90924805dd0f5189516693505d9b75f62197007f
    source_path: web/webchat.md
    workflow: 16
---

状态：macOS/iOS SwiftUI 聊天 UI 直接与 Gateway 网关 WebSocket 通信。没有嵌入式浏览器，也没有本地静态服务器。

## 它是什么

- Gateway 网关的原生聊天 UI。
- 使用与其他渠道相同的会话和路由规则。
- 确定性路由：回复始终返回 WebChat。
- 历史记录始终从 Gateway 网关获取（不监视本地文件）。如果无法连接 Gateway 网关，WebChat 将处于只读状态。

## 快速开始

1. 启动 Gateway 网关。
2. 打开 WebChat UI（macOS/iOS 应用）或 Control UI 聊天标签页。
3. 确保已配置有效的 Gateway 网关身份验证路径（默认使用共享密钥，即使在 loopback 上也是如此）。

## 工作原理

- UI 连接到 Gateway 网关 WebSocket，并使用 `chat.history`、`chat.send`、`chat.inject` 和 `chat.message.get` RPC 方法。
- `chat.history` 设有上限以确保稳定性：Gateway 网关可能截断较长的文本字段、省略体量较大的元数据，并将过大的条目替换为 `[chat.history omitted: message too large]`。API 客户端可以发送每次请求专用的 `maxChars`，以便为单次调用覆盖默认限制。
- 当 `chat.history` 中可见的智能体消息被截断时，Control UI 可以打开侧边阅读器，并通过 `chat.message.get` 按需获取完整的显示规范化条目，而无需增加默认历史记录载荷。`chat.message.get` 使用与 `chat.history` 相同的转录分支和显示规则，但通过 `messageId` 定位单个条目；当无法再返回完整内容时，它会如实返回不可用原因。
- `chat.history` 会沿用仅追加会话文件的活跃转录分支，因此 WebChat 不会渲染已放弃的重写分支和已被取代的提示词副本。
- 压缩条目会渲染为“已压缩的历史记录”分隔线，说明压缩后的转录作为检查点保留，并提供打开会话检查点的操作（权限允许时可进行分支或恢复）。
- Control UI 会记住 `chat.history` 返回的底层 Gateway 网关 `sessionId`，并在后续 `chat.send` 调用中包含它，因此重新连接和刷新页面后仍会继续同一段已存储的对话，除非用户启动或重置会话。
- 前台发送还会将已渲染历史记录中所显示分支的叶节点作为 `expectedLeafEntryId` 包含在内；如果另一个客户端先切换了分支，Control UI 会暂存消息以供检查并刷新转录，而不是将其发布到新分支。重新连接和恢复发件箱后的重放会在协调当前历史记录后有意省略此前置条件。
- `chat.send` 接受一个幂等键（Control UI 使用运行 ID）；Gateway 网关会对重复使用同一键的请求进行去重，因此针对同一会话、消息和附件重试或重复提交正在处理的请求，不会创建第二次运行。
- 回复特定消息（右键单击 → Reply）时，会在 `chat.send` 上将目标消息的转录 ID 作为 `replyToId` 发送。Gateway 网关会从会话历史记录中解析该消息，并填充与渠道无关的回复上下文元数据，与 Discord 回复所用的相同：智能体会看到 `has_reply_context`，以及包含发送者标签和正文的不可信“当前用户消息的回复目标”块。（根据直接 WebChat 会话现有的字节稳定提示词策略，WebChat 提示词会继续抑制 `reply_to_id` 等易变对话 ID。）没有持久化转录 ID 的回复目标（例如待发送消息）会回退为消息正文中的内联引用。
- 工作区启动文件和待处理的 `BOOTSTRAP.md` 指令通过智能体系统提示词的 `# Project Context` 部分提供，而不会复制到 WebChat 用户消息中。如果引导内容被截断，系统提示词会改为获得一条简短的“引导上下文通知”；详细计数和配置选项仍保留在诊断界面中。
- `chat.history` 上的显示规范化会去除：仅供运行时使用的 OpenClaw 上下文、入站信封包装器、`[[reply_to_current]]`、`[[reply_to:<id>]]` 和 `[[audio_as_voice]]` 等内联投递指令标签、纯文本工具调用 XML 载荷（`<tool_call>`、`<function_call>`、`<tool_calls>`、`<function_calls>`，包括被截断的块），以及泄漏的 ASCII/全角模型控制令牌。如果智能体条目的全部可见文本仅为静默令牌 `NO_REPLY`（不区分大小写），则会省略该条目。
- 标记为推理的回复载荷（`isReasoning: true`）会从 WebChat 智能体内容、转录重放文本和音频内容块中排除，因此仅含思考过程的载荷不会显示为可见的智能体消息或可播放音频。
- `chat.inject` 会将一条智能体备注直接追加到转录，并广播到 UI（不运行智能体）。
- 已中止的运行可以在 UI 中保留可见的部分智能体输出。当存在缓冲输出时，Gateway 网关会将该部分文本持久化到转录历史记录，并使用中止元数据标记该条目。

### 转录和投递模型

WebChat 有两条独立的数据路径：

- SQLite 转录行是持久的模型/运行时转录。对于普通智能体运行，嵌入式 OpenClaw 运行时通过会话访问器持久化模型可见的 `user`、`assistant` 和 `toolResult` 消息。WebChat 不会将任意投递、状态或辅助文本写入该转录。
- Gateway 网关 `ReplyPayload` 事件是实时投递投影：已针对 WebChat/渠道显示、分块流式传输、指令标签、媒体嵌入、TTS/音频标志和 UI 回退行为进行规范化。它们本身并不是规范会话日志。
- 需要通过 `tools.message` 提供可见回复的运行框架，仍会将 WebChat 用作当前运行的内部源回复接收端。来自该活跃 WebChat 运行且没有目标的 `message.send` 会投影到同一聊天中，并镜像到会话转录；WebChat 不会成为可复用的出站渠道，也绝不会继承 `lastChannel`。
- 仅当 Gateway 网关在普通嵌入式智能体轮次之外拥有一条已显示消息时，WebChat 才会注入智能体转录条目：`chat.inject`、非智能体命令回复、已中止的部分输出，以及由 WebChat 管理的媒体转录补充内容。
- 如果运行期间出现实时智能体文本，但在重新加载历史记录后消失，请依次检查：SQLite 转录是否包含该智能体文本、`chat.history` 显示投影是否将其去除，然后检查 Control UI 的乐观尾部合并是否用持久化快照替换了本地投递状态。

普通智能体运行的最终答案应当是持久的，因为嵌入式运行时会写入智能体 `message_end`。任何将已投递的最终载荷镜像到转录中的回退机制，都必须首先避免重复写入嵌入式运行时已写入的智能体轮次。

## Control UI 智能体工具面板

- Control UI 的 `/agents` Tools 面板包含一个由 `tools.effective(sessionKey=...)` 支持的“当前可用”视图：这是服务器派生的当前会话工具清单只读投影，其中包括核心工具、插件工具、渠道所有的工具，以及已发现的 MCP 服务器工具。
- 另一个配置编辑视图（由 `tools.catalog` 支持）涵盖配置文件、按智能体覆盖和目录语义。
- 运行时可用性以会话为范围。在同一智能体上切换会话可能会改变“当前可用”列表。如果已配置的 MCP 服务器自上次发现以来尚未连接或已发生更改，面板会显示通知，而不是从读取路径静默启动 MCP 传输。
- 配置编辑器并不表示运行时可用；有效访问权限仍遵循策略优先级（`allow`/`deny`、按智能体以及提供商/渠道覆盖）。

## 远程使用

- 远程模式通过 SSH/Tailscale 为 Gateway 网关 WebSocket 建立隧道。
- 无需运行单独的 WebChat 服务器。

## 配置参考（WebChat）

完整配置：[配置](/zh-CN/gateway/configuration)

WebChat 没有持久化配置部分。Gateway 网关使用内置的 `chat.history` 显示限制；API 客户端可以发送每次请求专用的 `maxChars`，以便为单次调用覆盖该限制。旧版 `channels.webchat` 和 `gateway.webchat` 配置已停用；运行 `openclaw doctor --fix` 将其移除。

相关全局选项：

- `gateway.port`、`gateway.bind`：WebSocket 主机/端口。
- `gateway.auth.mode`、`gateway.auth.token`、`gateway.auth.password`：
  共享密钥 WebSocket 身份验证。
- `gateway.auth.allowTailscale`：启用后，浏览器 Control UI 聊天标签页可以使用 Tailscale
  Serve 身份标头。
- `gateway.auth.mode: "trusted-proxy"`：适用于位于可感知身份的 **非 loopback** 代理源之后的浏览器客户端的反向代理身份验证（参见[可信代理身份验证](/zh-CN/gateway/trusted-proxy-auth)）。
- `gateway.remote.url`、`gateway.remote.token`、`gateway.remote.password`：远程 Gateway 网关目标。
- `session.*`：会话存储和主键默认值。

## 相关内容

- [Control UI](/zh-CN/web/control-ui)
- [仪表板](/zh-CN/web/dashboard)
