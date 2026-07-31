---
read_when:
    - 集成使用 OpenResponses API 的客户端
    - 你需要基于条目的输入、客户端工具调用或 SSE 事件
summary: 从 Gateway 网关公开兼容 OpenResponses 的 /v1/responses HTTP 端点
title: OpenResponses API
x-i18n:
    generated_at: "2026-07-26T06:48:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bfd6ca3bf0cecd761fde865b41a95cff3fc5681f74f31b3adae5cd2e0b0be95
    source_path: gateway/openresponses-http-api.md
    workflow: 16
---

Gateway 网关可以提供与 OpenResponses 兼容的 `POST /v1/responses` 端点。它**默认禁用**，并与 Gateway 网关共用端口（WS + HTTP 多路复用）：`http://<gateway-host>:<port>/v1/responses`。

请求以普通的 Gateway 网关智能体运行方式执行（与 `openclaw agent` 使用相同的代码路径），因此路由、权限和配置与你的 Gateway 网关一致。

使用 `gateway.http.endpoints.responses.enabled` 启用或禁用。启用后，同一兼容接口还会提供 `GET /v1/models`、`GET /v1/models/{id}`、`POST /v1/embeddings` 和 `POST /v1/chat/completions`。

## 身份验证、安全和路由

运行行为与 [OpenAI Chat Completions](/zh-CN/gateway/openai-http-api) 一致：

- 身份验证路径与 `gateway.auth.mode` 一致：共享密钥（`token`/`password`）使用 `Authorization: Bearer <token-or-password>`；可信代理使用可识别身份的代理标头（同主机 local loopback 代理需要 `gateway.auth.trustedProxy.allowLoopback = true`；当不存在 `Forwarded`/`X-Forwarded-*`/`X-Real-IP` 标头时，可通过 `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` 使用同主机直接回退）；私有入口上的 `none` 不需要身份验证标头。请参阅[可信代理身份验证](/zh-CN/gateway/trusted-proxy-auth)。
- 应将该端点视为拥有对 Gateway 网关实例的完整操作员访问权限。
- 共享密钥身份验证模式会忽略由 Bearer 声明的更窄 `x-openclaw-scopes`，并恢复完整的默认操作员权限范围集：`operator.admin`、`operator.approvals`、`operator.pairing`、`operator.read`、`operator.talk.secrets`、`operator.write`。此端点上的聊天轮次会被视为所有者发送者轮次。
- 携带可信身份的 HTTP 模式（可信代理或 `gateway.auth.mode="none"`）会在存在 `x-openclaw-scopes` 时遵循其设置，否则回退到操作员默认权限范围集。仅当调用方显式缩小权限范围并省略 `operator.admin` 时，才会失去所有者语义。
- 使用 `model: "openclaw"`、`"openclaw/default"`、`"openclaw/<agentId>"` 或 `x-openclaw-agent-id` 标头选择智能体。
- 使用 `x-openclaw-model` 覆盖所选智能体的后端模型（在携带身份的身份验证路径上需要 `operator.admin`）。
- 使用 `x-openclaw-session-key` 进行显式会话路由（如果使用保留命名空间 `subagent:`、`cron:` 或 `acp:`，则会以 `400 invalid_request_error` 拒绝）。
- 使用 `x-openclaw-message-channel` 设置非默认的合成入口渠道上下文。

有关智能体目标模型、`openclaw/default`、嵌入直通和后端模型覆盖的规范说明，请参阅 [OpenAI Chat Completions](/zh-CN/gateway/openai-http-api#agent-first-model-contract)。

请参阅[操作员权限范围](/zh-CN/gateway/operator-scopes)和[安全](/zh-CN/gateway/security)。

## 会话行为

默认情况下，该端点**每个请求均无状态**（每次调用都会生成新的会话键）。

如果请求包含 OpenResponses `user` 字符串，Gateway 网关会从中派生稳定的会话键，使重复调用可以共享一个智能体会话。

当请求仍处于同一智能体/用户/请求会话范围内（按身份验证主体、智能体 ID 和 `x-openclaw-session-key` 匹配）时，`previous_response_id` 会复用先前响应的会话。

## 请求结构

| 字段                                                            | 支持情况                                                                                                                        |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `input`                                                          | 字符串或项目对象数组。                                                                                               |
| `instructions`                                                   | 合并到系统提示词中。                                                                                                 |
| `tools`                                                          | 客户端工具定义（函数工具）。                                                                                      |
| `tool_choice`                                                    | 使用 `"auto"`、`"none"`、`"required"` 或 `{ "type": "function", "name": "..." }` 筛选或要求使用客户端工具。                |
| `stream`                                                         | 启用 SSE 流式传输。                                                                                                         |
| `max_output_tokens`                                              | 尽力而为的输出限制（取决于提供商）。                                                                                 |
| `temperature`                                                    | 尽力而为的采样温度。基于 ChatGPT 的 Codex Responses 后端使用固定的服务器端采样，因此会忽略此项。 |
| `top_p`                                                          | 尽力而为的核采样。存在与 `temperature` 相同的 Codex Responses 注意事项。                                                    |
| `user`                                                           | 稳定会话路由。                                                                                                        |
| `previous_response_id`                                           | 会话连续性（见上文）。                                                                                                |
| `max_tool_calls`、`reasoning`、`metadata`、`store`、`truncation` | 接受，但目前忽略。                                                                                                |

## 项目（输入）

### `message`

角色：`system`、`developer`、`user`、`assistant`。

- `system` 和 `developer` 会追加到系统提示词中。
- 最近的 `user` 或 `function_call_output` 项目会成为“当前消息”。
- 更早的用户/助手消息会作为历史记录纳入上下文。

### `function_call_output`（基于轮次的工具）

将工具结果发送回模型：

```json
{
  "type": "function_call_output",
  "call_id": "call_123",
  "output": "{\"temperature\": \"72F\"}"
}
```

### `reasoning` 和 `item_reference`

为保持架构兼容性而接受，但构建提示词时会忽略。

## 工具（客户端函数工具）

通过 `tools: [{ type: "function", name, description?, parameters? }]` 提供工具。

如果智能体调用工具，响应会返回一个 `function_call` 输出项目。发送包含 `function_call_output` 的后续请求以继续该轮次。

对于 `tool_choice: "required"` 和固定函数的 `tool_choice`，该端点会缩小公开的客户端函数工具集，指示运行时在响应前调用客户端工具；如果该轮次未包含匹配的结构化客户端工具调用，则会按照 `/v1/chat/completions` 契约拒绝该轮次。非流式请求返回带有 `api_error` 的 `502`；流式请求会发出 `response.failed` 事件。

## 图像（`input_image`）

支持 base64 或 URL 来源：

```json
{
  "type": "input_image",
  "source": { "type": "url", "url": "https://example.com/image.png" }
}
```

允许的 MIME 类型（默认）：`image/jpeg`、`image/png`、`image/gif`、`image/webp`、`image/heic`、`image/heif`。最大大小（默认）：10MB。

## 文件（`input_file`）

支持 base64 或 URL 来源：

```json
{
  "type": "input_file",
  "source": {
    "type": "base64",
    "media_type": "text/plain",
    "data": "SGVsbG8gV29ybGQh",
    "filename": "hello.txt"
  }
}
```

允许的 MIME 类型（默认）：`text/plain`、`text/markdown`、`text/html`、`text/csv`、`application/json`、`application/pdf`。最大大小（默认）：5MB。

当前行为：

- 文件内容会被解码并添加到**系统提示词**而非用户消息中，因此它保持临时状态（不会持久化到会话历史记录）。
- 解码后的文件文本在添加前会被包装为**不受信任的外部内容**，因此文件字节会被视为数据，而非可信指令。注入的块使用显式边界标记（`<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>`）和一行 `Source: External` 元数据。为保留提示词预算，它有意省略较长的 `SECURITY NOTICE:` 横幅；边界标记和元数据仍然适用。
- PDF 会首先进行文本解析。如果找到的文本很少，则会将前几页栅格化为图像并传递给模型，同时注入的文件块会使用占位符 `[PDF content rendered to images]`。

PDF 解析由内置的 `document-extract` 插件提供，该插件使用 `clawpdf` 及其打包的 PDFium WebAssembly 运行时进行文本提取和页面渲染。

URL 获取默认值：

- `files.allowUrl`：`true`
- `images.allowUrl`：`true`
- `maxUrlParts`：`8`（每个请求中基于 URL 的 `input_file` + `input_image` 部分总数）
- 请求受到防护（DNS 解析、私有 IP 阻止、重定向次数上限、超时）。
- 每种输入类型均支持可选的主机名允许列表（`files.urlAllowlist`、`images.urlAllowlist`）：精确主机（`"cdn.example.com"`）或通配符子域名（`"*.assets.example.com"`，不匹配根域名）。允许列表为空或省略时，表示不施加主机名允许列表限制。
- 要完全禁用基于 URL 的获取，请设置 `files.allowUrl: false` 和/或 `images.allowUrl: false`。

## 文件和图像限制

该端点使用内置的 20 MB 请求正文限制。文件和图像来源
策略仍可在 `gateway.http.endpoints.responses` 下配置：

```json5
{
  gateway: {
    http: {
      endpoints: {
        responses: {
          enabled: true,
          maxUrlParts: 8,
          files: {
            allowUrl: true,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "text/plain",
              "text/markdown",
              "text/html",
              "text/csv",
              "application/json",
              "application/pdf",
            ],
            maxBytes: 5242880,
            maxChars: 60000,
            maxRedirects: 3,
            timeoutMs: 10000,
            pdf: {
              maxPages: 4,
              maxPixels: 4000000,
              minTextChars: 200,
            },
          },
          images: {
            allowUrl: true,
            urlAllowlist: ["images.example.com"],
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

省略时的默认值：

| 键                      | 默认值   |
| ------------------------ | --------- |
| `maxUrlParts`            | 8         |
| `files.maxBytes`         | 5MB       |
| `files.maxChars`         | 60k       |
| `files.maxRedirects`     | 3         |
| `files.timeoutMs`        | 10s       |
| `files.pdf.maxPages`     | 4         |
| `files.pdf.maxPixels`    | 4,000,000 |
| `files.pdf.minTextChars` | 200       |
| `images.maxBytes`        | 10MB      |
| `images.maxRedirects`    | 3         |
| `images.timeoutMs`       | 10s       |

HEIC/HEIF `input_image` 源在通过共享的 OpenClaw 图像处理器（Rastermill）交付给提供商之前，会被规范化为 JPEG；对于需要外部编解码器支持的格式，该处理器会回退到系统转换器（`sips`、ImageMagick、GraphicsMagick 或 ffmpeg）。

安全说明：在获取 URL 之前以及重定向的每一跳，都会强制执行 URL 允许列表。将主机名列入允许列表不会绕过对私有/内部 IP 的阻止。对于暴露于互联网的 Gateway 网关，除应用级防护外，还应实施网络出站控制。请参阅[安全](/zh-CN/gateway/security)。

## 流式传输（SSE）

设置 `stream: true` 以接收服务器发送事件：

- `Content-Type: text/event-stream`
- 每个事件行均为 `event: <type>` 和 `data: <json>`
- 流以 `data: [DONE]` 结束

当前发出的事件类型：`response.created`、`response.in_progress`、`response.output_item.added`、`response.content_part.added`、`response.output_text.delta`、`response.output_text.done`、`response.content_part.done`、`response.output_item.done`、`response.completed`、`response.failed`（发生错误时）。

## 使用量

当底层提供商报告 token 数量时，会填充 `usage`。在这些计数器传递到下游状态/会话界面之前，OpenClaw 会规范化常见的 OpenAI 风格别名，包括 `input_tokens` / `output_tokens` 和 `prompt_tokens` / `completion_tokens`。

## 错误

错误使用如下 JSON 对象：

```json
{ "error": { "message": "...", "type": "invalid_request_error" } }
```

常见情况：`400` 请求正文无效、`401` 身份验证缺失/无效、`403` 缺少操作员权限范围、`405` 方法错误、`429` 身份验证失败次数过多（附带 `Retry-After`）。

## 示例

非流式传输：

```bash
curl -sS http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "input": "hi"
  }'
```

流式传输：

```bash
curl -N http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "stream": true,
    "input": "hi"
  }'
```

## 相关内容

- [OpenAI 聊天补全](/zh-CN/gateway/openai-http-api)
- [操作员权限范围](/zh-CN/gateway/operator-scopes)
- [OpenAI](/zh-CN/providers/openai)
