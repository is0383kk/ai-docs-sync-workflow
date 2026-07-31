---
read_when:
    - 构建无法使用 Gateway 网关 WebSocket RPC 客户端的主机工具
    - 通过私有可信入口开放 Gateway 网关管理自动化功能
    - 审计通过 HTTP 访问 Gateway 网关方法的安全模型
summary: 通过内置且需选择启用的 admin-http-rpc 插件公开选定的 Gateway 网关控制平面方法
title: 管理员 HTTP RPC 插件
x-i18n:
    generated_at: "2026-07-26T06:49:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0709081efd0ce65cef7edac54df9a71978cbad17e2b25df83ac9075de938376c
    source_path: plugins/admin-http-rpc.md
    workflow: 16
---

内置的 `admin-http-rpc` 插件通过 HTTP 公开一组列入允许列表的 Gateway 网关控制平面方法，供无法保持 Gateway 网关 WebSocket 连接的可信主机自动化使用。

它随 OpenClaw 一起提供，但默认禁用；禁用时，不会注册该路由。启用后，它会在与 Gateway 网关相同的监听器（`http://<gateway-host>:<port>/api/v1/admin/rpc`）上添加 `POST /api/v1/admin/rpc`。

仅将其用于私有主机工具、tailnet 自动化或可信内部入口。切勿将此路由直接暴露到公共互联网。

## 启用前

管理员 HTTP RPC 是完整的操作员控制平面接口：任何通过 Gateway 网关 HTTP 身份验证的调用方都可以调用下方列入允许列表的方法。仅当以下条件全部满足时才启用它：

- 调用方受信任，可以操作 Gateway 网关。
- 调用方无法使用 WebSocket RPC 客户端。
- 仅可通过环回地址、tailnet 或经过身份验证的私有入口访问该路由。
- 你已审查允许的方法，并确认它们与计划运行的自动化相符。

对于能够保持 Gateway 网关 WebSocket 连接的 OpenClaw 客户端和交互式工具，请改用 WebSocket RPC。

## 启用

启用内置插件：

<Tabs>
  <Tab title="CLI">
    ```bash
    openclaw plugins enable admin-http-rpc
    openclaw gateway restart
    ```
  </Tab>
  <Tab title="配置">
    ```json5
    {
      plugins: {
        entries: {
          "admin-http-rpc": { enabled: true },
        },
      },
    }
    ```
  </Tab>
</Tabs>

该路由会在插件启动期间注册，因此更改插件配置后请重启 Gateway 网关。

不再需要此 HTTP 接口时，请将其禁用：

```bash
openclaw plugins disable admin-http-rpc
openclaw gateway restart
```

## 验证路由

使用 `health` 作为最小的安全请求：

```bash
curl -sS http://<gateway-host>:<port>/api/v1/admin/rpc \
  -H 'Authorization: Bearer <gateway-token>' \
  -H 'Content-Type: application/json' \
  -d '{"method":"health","params":{}}'
```

成功响应包含 `ok: true`：

```json
{
  "id": "generated-request-id",
  "ok": true,
  "payload": {
    "status": "ok"
  }
}
```

插件禁用时，由于该路由未注册，因此返回 `404`。

## 身份验证

插件路由使用 Gateway 网关 HTTP 身份验证。

常见身份验证方式：

- 共享密钥身份验证（`gateway.auth.mode="token"` 或 `"password"`）：`Authorization: Bearer <token-or-password>`
- 携带可信身份的 HTTP 身份验证（`gateway.auth.mode="trusted-proxy"`）：通过已配置的身份感知代理转发路由，并由其注入所需的身份标头
- 私有入口开放身份验证（`gateway.auth.mode="none"`）：无需身份验证标头

## 安全模型

将此插件视为完整的 Gateway 网关操作员接口。

- 启用插件会有意在 `/api/v1/admin/rpc` 提供对允许列表中管理员 RPC 方法的访问权限。
- 该插件声明了保留的 `contracts.gatewayMethodDispatch: ["authenticated-request"]` 清单契约，因此其通过 Gateway 网关身份验证的 HTTP 路由能够在进程内分派控制平面方法。这并非沙箱：该契约可防止意外使用保留的 SDK 辅助函数，但可信插件仍在 Gateway 网关进程中运行。
- 共享密钥持有者身份验证（`token`/`password` 模式）可证明调用方持有 Gateway 网关操作员密钥；在该路径中，范围更窄的 `x-openclaw-scopes` 标头会被忽略，并恢复正常的完整操作员默认权限。
- 携带可信身份的 HTTP 身份验证（`trusted-proxy` 模式）会在存在 `x-openclaw-scopes` 时遵循其设置。
- `gateway.auth.mode="none"` 表示启用插件后此路由不进行身份验证。仅应在你完全信任的私有入口之后使用此设置。
- 插件路由身份验证通过后，请求会经由与 WebSocket RPC 相同的 Gateway 网关方法处理程序和权限范围检查进行分派。
- 在已准备的暂停租约期间，该路由仍然可访问。有界请求验证和本地 `commands.list` 发现响应仍然可用。在分派到 Gateway 网关的方法中，准入关闭期间仅 `gateway.suspend.prepare`、`gateway.suspend.status` 和 `gateway.suspend.resume` 可以运行；其他列入允许列表的方法会返回正常的可重试 Gateway 网关 `UNAVAILABLE` 响应。
- 请将此路由限制在环回地址、tailnet 或可信私有入口上。不要将其直接暴露到公共互联网。当调用方跨越信任边界时，请使用不同的 Gateway 网关。

## 请求

```http
POST /api/v1/admin/rpc
Authorization: Bearer <gateway-token>
Content-Type: application/json
```

```json
{
  "id": "optional-request-id",
  "method": "health",
  "params": {}
}
```

字段：

- `id`（字符串，可选）：复制到响应中。省略时会生成 UUID。
- `method`（字符串，必填）：允许的 Gateway 网关方法名称。
- `params`（任意类型，可选）：特定于方法的参数。

默认最大请求正文大小为 1 MB。

## 响应

成功响应使用 Gateway 网关 RPC 格式：

```json
{
  "id": "optional-request-id",
  "ok": true,
  "payload": {}
}
```

Gateway 网关方法错误使用以下格式：

```json
{
  "id": "optional-request-id",
  "ok": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "bad params"
  }
}
```

HTTP 状态码取决于错误代码：

| 错误代码                 | HTTP 状态码 |
| -------------------------- | ----------- |
| `INVALID_REQUEST`          | 400         |
| `APPROVAL_NOT_FOUND`       | 404         |
| `NOT_LINKED`, `NOT_PAIRED` | 409         |
| `UNAVAILABLE`              | 503         |
| `AGENT_TIMEOUT`            | 504         |
| 任何其他代码             | 500         |

## 允许的方法

- 发现：`commands.list`
  返回此插件允许的 HTTP RPC 方法名称。
- Gateway 网关：`health`、`status`、`logs.tail`、`usage.status`、`usage.cost`、`gateway.restart.request`、`gateway.suspend.prepare`、`gateway.suspend.status`、`gateway.suspend.resume`
- 配置：`config.get`、`config.schema`、`config.schema.lookup`、`config.set`、`config.patch`、`config.apply`
- 渠道：`channels.status`、`channels.start`、`channels.stop`、`channels.logout`
- Web：`web.login.start`、`web.login.wait`
- 模型：`models.list`、`models.authStatus`
- 智能体：`agents.list`、`agents.create`、`agents.update`、`agents.delete`
- 审批：`exec.approvals.get`、`exec.approvals.set`、`exec.approvals.node.get`、`exec.approvals.node.set`
- 定时任务：`cron.status`、`cron.list`、`cron.get`、`cron.runs`、`cron.add`、`cron.update`、`cron.remove`、`cron.run`
- 设备：`device.pair.list`、`device.pair.approve`、`device.pair.reject`、`device.pair.remove`
- 节点：`node.list`、`node.describe`、`node.pair.list`、`node.pair.approve`、`node.pair.reject`、`node.pair.remove`、`node.rename`
- 任务：`tasks.list`、`tasks.get`、`tasks.cancel`
- 诊断：`doctor.memory.status`、`update.status`

其他 Gateway 网关方法均会被阻止，直到有意将其添加。

## WebSocket 对比

对于 OpenClaw 客户端，常规 Gateway 网关 WebSocket RPC 路径仍是首选的控制平面 API。仅当主机工具需要请求/响应式 HTTP 接口时，才使用管理员 HTTP RPC。

没有可信设备身份的共享令牌 WebSocket 客户端无法在连接期间自行声明管理员权限范围。管理员 HTTP RPC 有意遵循现有的可信 HTTP 操作员模型：启用插件后，共享密钥持有者身份验证会被视为拥有此管理员接口的完整操作员访问权限。

## 故障排查

`404 Not Found`

：插件已禁用、启用插件后尚未重启 Gateway 网关，或请求被发送到了另一个 Gateway 网关进程。

`401 Unauthorized`

：请求未满足 Gateway 网关 HTTP 身份验证要求。请检查持有者令牌或可信代理身份标头。

`405 Method Not Allowed`

：请求使用了 `POST` 以外的内容。

`413 Payload Too Large`

：请求正文超过 1 MB 限制。

`400 INVALID_REQUEST`

：请求正文不是有效的 JSON、缺少 `method` 字段、方法不在插件允许列表中，或暂停恢复 ID 与活动租约不匹配。

`503 UNAVAILABLE`

：Gateway 网关方法正在启动、受到速率限制、处于暂停状态，或正在等待与之竞争的暂停/恢复操作。存在 `error.details` 时请检查它，并在重试前遵循 `error.retryAfterMs`。

## 相关内容

- [操作员权限范围](/zh-CN/gateway/operator-scopes)
- [Gateway 安全](/zh-CN/gateway/security)
- [远程访问](/zh-CN/gateway/remote)
- [插件清单](/zh-CN/plugins/manifest#contracts-reference)
- [SDK 子路径](/zh-CN/plugins/sdk-subpaths)
