---
read_when:
    - 集成需要 OpenAI Chat Completions 的工具
summary: 从 Gateway 网关公开兼容 OpenAI 的 `/v1/chat/completions` HTTP 端点
title: OpenAI 聊天补全
x-i18n:
    generated_at: "2026-07-26T06:44:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4cc5a1a56972bb9070da0f8f60d6efd673cc1d1d516b730c55bc9d171fc7a5b3
    source_path: gateway/openai-http-api.md
    workflow: 16
---

Gateway 网关可以提供一个小型的 OpenAI 兼容 Chat Completions 接口。它**默认禁用**。

启用后，它会在与 Gateway 网关相同的端口上提供以下所有接口（WS + HTTP 多路复用）：

| 方法 | 路径                   |
| ------ | ---------------------- |
| POST   | `/v1/chat/completions` |
| GET    | `/v1/models`           |
| GET    | `/v1/models/{id}`      |
| POST   | `/v1/embeddings`       |
| POST   | `/v1/responses`        |

请求会作为普通的 Gateway 网关智能体运行来执行（与 `openclaw agent` 使用相同的代码路径），因此路由、权限和配置均与你的 Gateway 网关一致。

## 启用端点

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: true },
      },
    },
  },
}
```

将 `enabled: false` 设为禁用值（或省略它）即可禁用。

## 安全边界（重要）

应将此端点视为对 Gateway 网关实例的**完整操作员访问权限**：

- 此端点的有效 Gateway 网关令牌/密码等同于所有者/操作员凭据，而非范围受限的单用户权限。
- 请求会通过与受信任操作员操作相同的控制平面智能体路径运行，因此，如果目标智能体的策略允许使用敏感工具，此端点也可以使用这些工具。
- 仅应将其部署在 local loopback、tailnet 或私有入口上。切勿将其暴露到公共互联网。

身份验证矩阵：

| 身份验证路径                                                                                            | 行为                                                                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway.auth.mode="token"` 或 `"password"` + `Authorization: Bearer ...`                            | 证明调用方持有共享的 Gateway 网关密钥。忽略任何 `x-openclaw-scopes` 请求头，并恢复完整的默认操作员权限范围集：`operator.admin`、`operator.approvals`、`operator.pairing`、`operator.read`、`operator.talk.secrets`、`operator.write`。将聊天轮次视为所有者发送方的轮次。 |
| 携带受信任身份的 HTTP（受信任代理身份验证，或私有入口上的 `gateway.auth.mode="none"`） | 存在 `x-openclaw-scopes` 时遵循其设置；不存在时回退到默认操作员权限范围集。仅当调用方明确缩小权限范围且省略 `operator.admin` 时，才会失去所有者语义。对于 `x-openclaw-model` 等所有者级控制，需要 `operator.admin`。                        |

请参阅[操作员权限范围](/zh-CN/gateway/operator-scopes)、[安全](/zh-CN/gateway/security)和[远程访问](/zh-CN/gateway/remote)。

## 身份验证

使用 Gateway 网关身份验证配置（有关该模式的详细信息，请参阅[受信任代理身份验证](/zh-CN/gateway/trusted-proxy-auth)）：

| 模式                                | 身份验证方式                                                                                                                                                                     |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway.auth.mode="token"`         | `Authorization: Bearer <token>`。通过 `gateway.auth.token` 或 `OPENCLAW_GATEWAY_TOKEN` 设置。                                                                                              |
| `gateway.auth.mode="password"`      | `Authorization: Bearer <password>`。通过 `gateway.auth.password` 或 `OPENCLAW_GATEWAY_PASSWORD` 设置。                                                                                     |
| `gateway.auth.mode="trusted-proxy"` | 通过已配置的身份感知代理进行路由；该代理会注入所需的身份请求头。同主机 local loopback 代理需要显式设置 `gateway.auth.trustedProxy.allowLoopback = true`。 |
| `gateway.auth.mode="none"`          | 无需身份验证请求头（仅限私有入口）。                                                                                                                                         |

注意：

- 在 `trusted-proxy` Gateway 网关上绕过代理的同主机调用方可以直接回退到 `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`。任何 `Forwarded`、`X-Forwarded-*` 或 `X-Real-IP` 请求头证据都会使请求继续使用受信任代理路径。
- 如果已配置 `gateway.auth.rateLimit`，且身份验证失败次数过多，端点将返回 `429`，并附带 `Retry-After` 请求头。

## 何时使用此端点

- 如果你的集成只是同一 Gateway 网关的另一个操作员/客户端接口，应优先使用此端点，而不是添加新的内置渠道。
- 对于直接连接远程 Gateway 网关的原生移动客户端，应优先使用 [WebChat](/zh-CN/web/webchat) 或采用配对设备引导/设备令牌流程的 [Gateway 协议](/zh-CN/gateway/protocol)，这样设备无需共享 HTTP 令牌/密码。
- 如果要集成拥有自身用户、房间、Webhook 投递或出站传输的外部消息网络，则应改为构建渠道插件。请参阅[构建插件](/zh-CN/plugins/building-plugins)。

## 智能体优先的模型契约

OpenClaw 将 OpenAI 的 `model` 字段视为**智能体目标**，而非原始提供商模型 ID。

| `model` 值                                | 路由到                                                                                                                |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `openclaw`                                   | 已配置的默认智能体                                                                                                 |
| `openclaw/default`                           | 已配置的默认智能体（稳定别名；即使实际默认智能体 ID 因环境而异，也可安全地硬编码） |
| `openclaw/<agentId>` 或 `openclaw:<agentId>` | 指定智能体                                                                                                           |
| `agent:<agentId>`                            | 指定智能体（兼容性别名）                                                                                     |

可选请求头：

| 请求头                                          | 效果                                                                                                                                                                                                                                                                      |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `x-openclaw-model: <provider/model-or-bare-id>` | 覆盖所选智能体的后端模型。使用共享密钥 Bearer 的调用方可以直接使用此请求头；携带身份的调用方（受信任代理，或具有 `x-openclaw-scopes` 的私有免身份验证入口）需要 `operator.admin`，否则为 `403 missing scope: operator.admin`。 |
| `x-openclaw-agent-id: <agentId>`                | 用于选择智能体的兼容性覆盖设置。                                                                                                                                                                                                                                 |
| `x-openclaw-session-key: <sessionKey>`          | 显式会话路由。如果使用保留的内部命名空间（`subagent:`、`cron:`、`acp:`），则以 `400 invalid_request_error` 拒绝。                                                                                                                                |
| `x-openclaw-message-channel: <channel>`         | 为支持渠道感知的提示词/策略设置合成入口渠道上下文。                                                                                                                                                                                              |

`/v1/models` 列出顶层智能体目标（`openclaw`、`openclaw/default`、`openclaw/<agentId>`），而非后端提供商模型或子智能体；子智能体仍属于内部执行拓扑。如果省略 `x-openclaw-model`，所选智能体将使用其正常配置的模型运行。

`/v1/embeddings` 使用相同的智能体目标 `model` ID。发送 `x-openclaw-model`（调用方须使用共享密钥，或为具有 `operator.admin` 的携带身份调用方）以选择特定的嵌入模型；否则，请求将使用所选智能体的正常嵌入设置。

## 会话行为

默认情况下，该端点**对每个请求无状态**（每次调用都会生成新的会话键）。

如果请求包含 OpenAI `user` 字符串，Gateway 网关会据此派生稳定的会话键，使重复调用可以共享智能体会话。对于自定义应用，应为每个对话线程复用相同的 `user` 值；除非你希望多个对话/设备共享同一个 OpenClaw 会话，否则应避免使用账户级标识符。仅当需要跨多个客户端/线程进行显式路由控制时，才使用 `x-openclaw-session-key`，并使用由应用拥有且避开上述保留命名空间的键。

## 请求限制

该端点使用以下内置限制：每个请求正文 20 MB、最新用户消息中包含 8 个 `image_url`
部分，以及累计 20 MB 的已解码图像
数据。图像来源策略仍可通过
`gateway.http.endpoints.chatCompletions.images` 进行配置：

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: {
          enabled: true,
          images: {
            allowUrl: false,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "image/jpeg",
              "image/png",
              "image/gif",
              "image/webp",
              "image/heic",
              "image/heif",
            ],
            maxBytes: 10485760,
            maxRedirects: 3,
            timeoutMs: 10000,
          },
        },
      },
    },
  },
}
```

图像设置的默认值为：

| 键                   | 默认值                                                             |
| --------------------- | ------------------------------------------------------------------- |
| `images.allowUrl`     | `false`（除非启用，否则拒绝源自 URL 的 `image_url` 部分） |
| `images.maxBytes`     | 每张图像 10MB                                                      |
| `images.maxRedirects` | 3                                                                   |
| `images.timeoutMs`    | 10s                                                                 |

系统接受 HEIC/HEIF `image_url` 来源，并在通过共享的 OpenClaw 图像处理器（Rastermill）传递给提供商之前将其标准化为 JPEG；对于需要外部编解码器支持的格式，该处理器会回退到系统转换器（`sips`、ImageMagick、GraphicsMagick 或 ffmpeg）。

安全说明：将主机名加入允许列表不会绕过私有/内部 IP 屏蔽。对于暴露在互联网中的 Gateway 网关，除应用级防护外，还应实施网络出口控制。请参阅[安全](/zh-CN/gateway/security)。

## 聊天工具契约

`/v1/chat/completions` 支持与常见 OpenAI Chat 客户端兼容的函数工具子集。

### 支持的请求字段

| 字段                       | 说明                                                                                                                                          |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `tools`                    | `{ "type": "function", "function": { ... } }` 数组                                                                                         |
| `tool_choice`              | `"auto"`、`"none"`、`"required"` 或 `{ "type": "function", "function": { "name": "..." } }`                                                  |
| `messages[*].role: "tool"` | 后续轮次                                                                                                                               |
| `messages[*].tool_call_id` | 将工具结果绑定回先前的工具调用                                                                                                 |
| `max_completion_tokens`    | 数字；每次调用的完成 token 总数上限（包括推理 token）。当前字段名；当它与 `max_tokens` 同时发送时使用此字段。 |
| `max_tokens`               | 数字；旧版别名，当 `max_completion_tokens` 也存在时忽略。                                                                   |
| `temperature`              | 0-2 的数字；尽力处理，转发给上游提供商。超出范围时为 `400 invalid_request_error`。                                     |
| `top_p`                    | 0-1 的数字；尽力处理。超出范围时为 `400 invalid_request_error`。                                                                         |
| `frequency_penalty`        | -2.0 到 2.0 的数字；尽力处理。超出范围时为 `400 invalid_request_error`。                                                                 |
| `presence_penalty`         | -2.0 到 2.0 的数字；尽力处理。超出范围时为 `400 invalid_request_error`。                                                                 |
| `seed`                     | 整数；尽力处理。对于非整数值为 `400 invalid_request_error`。                                                                     |
| `stop`                     | 字符串或最多包含 4 个字符串的数组；尽力处理。对于超过 4 个序列或非字符串/空条目为 `400 invalid_request_error`。           |

所有采样和 token 上限字段都通过同一个智能体流参数渠道传递，并以尽力而为的方式转发：

- Token 上限：传输字段名由提供商传输层选择：OpenAI 系列端点使用 `max_completion_tokens`，仅接受旧版名称的提供商（Mistral、Chutes）使用 `max_tokens`。
- `stop` 映射到传输层的停止字段：Chat Completions 后端使用 `stop`，Anthropic 使用 `stop_sequences`。OpenAI Responses API 没有停止参数，因此 `stop` 不会应用于基于 Responses 的模型。
- 基于 ChatGPT 的 Codex Responses 后端使用固定的服务端采样，并在请求到达该后端前移除 `temperature`/`top_p`（以及 `max_output_tokens`、`metadata`、`prompt_cache_retention`、`service_tier`）。

### 不支持的变体

以下情况返回 `400 invalid_request_error`：

- 非数组 `tools`、非函数工具条目或缺少 `tool.function.name`
- `tool_choice` 变体，例如 `allowed_tools` 和 `custom`
- 与所提供工具不匹配的 `tool_choice.function.name` 值

对于 `tool_choice: "required"` 和固定到函数的 `tool_choice`，该端点会缩小向客户端公开的函数工具集，指示运行时在响应前调用客户端工具，并在智能体响应中没有匹配的结构化客户端工具调用时报错。这适用于调用方提供的 HTTP `tools` 列表，而不是 OpenClaw 智能体的所有内部工具。

### 非流式工具响应结构

当智能体调用工具时，响应使用：

- `choices[0].finish_reason = "tool_calls"`
- 包含 `id`、`type: "function"`、`function.name`、`function.arguments`（JSON 字符串）的 `choices[0].message.tool_calls[]` 条目
- 工具调用前的助手注释，位于 `choices[0].message.content` 中（可能为空）

### 流式工具响应结构

当 `stream: true` 时，工具调用以增量 SSE 块到达：先是初始助手角色增量，然后是可选的助手注释增量，再是一个或多个携带工具标识和参数片段的 `delta.tool_calls` 块，最后是包含 `finish_reason: "tool_calls"` 和 `data: [DONE]` 的结束块。

如果 `stream_options.include_usage=true`，则会在 `[DONE]` 前发送一个尾部用量块。

### 工具后续循环

收到 `tool_calls` 后，执行所请求的函数，并发送后续请求，其中包含先前的助手工具调用消息以及一条或多条具有匹配 `tool_call_id` 的 `role: "tool"` 消息。这会继续同一个智能体推理循环，以生成最终答案。

## 流式传输（SSE）

设置 `stream: true` 以接收服务器发送事件：

- `Content-Type: text/event-stream`
- 每个事件行都是 `data: <json>`
- 流以 `data: [DONE]` 结束

## Open WebUI 快速设置

- 基础 URL：`http://127.0.0.1:18789/v1`
- macOS 上 Docker 的基础 URL：`http://host.docker.internal:18789/v1`
- API 密钥：你的 Gateway 网关 bearer token
- 模型：`openclaw/default`

预期行为：`GET /v1/models` 会列出 `openclaw/default`，Open WebUI 会将其用作聊天模型 ID。对于特定的后端提供商/模型，请设置智能体的常规默认模型，或发送 `x-openclaw-model`（使用共享密钥的调用方，或具有 `operator.admin` 的身份调用方）。

快速冒烟测试：

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

如果返回 `openclaw/default`，大多数 Open WebUI 设置都可以使用相同的基础 URL 和 token 进行连接。

## 示例

为一个应用对话使用稳定会话：

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "user": "conv:YOUR_CONVERSATION_ID",
    "messages": [{"role":"user","content":"总结我今天的任务"}]
  }'
```

在该对话的后续调用中复用相同的 `user` 值，以继续同一个智能体会话。

非流式：

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "messages": [{"role":"user","content":"你好"}]
  }'
```

流式：

```bash
curl -N http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/gpt-5.4' \
  -d '{
    "model": "openclaw/research",
    "stream": true,
    "messages": [{"role":"user","content":"你好"}]
  }'
```

列出模型：

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

获取单个模型：

```bash
curl -sS http://127.0.0.1:18789/v1/models/openclaw%2Fdefault \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

创建嵌入：

```bash
curl -sS http://127.0.0.1:18789/v1/embeddings \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/text-embedding-3-small' \
  -d '{
    "model": "openclaw/default",
    "input": ["alpha", "beta"]
  }'
```

`/v1/embeddings` 支持将 `input` 设为字符串或字符串数组。

## 相关内容

- [配置参考](/zh-CN/gateway/configuration-reference)
- [操作员权限范围](/zh-CN/gateway/operator-scopes)
- [OpenAI](/zh-CN/providers/openai)
