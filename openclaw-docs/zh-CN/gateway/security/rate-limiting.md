---
read_when:
    - 客户端遇到 `rate limit exceeded for <method>`、`AUTH_RATE_LIMITED` 或锁定错误
    - 你想要调优 `gateway.auth.rateLimit`
    - 你正在分析暴露的 Gateway 网关所需的暴力破解防护
    - 你需要了解哪些 Gateway 网关接口会受到限流，以及具体的限流值。
summary: 每项 Gateway 网关速率限制的参考：身份验证前锁定、浏览器和 Webhook 限流、控制平面写入兜底限制、ACP 会话上限以及重启冷却时间
title: 速率限制
x-i18n:
    generated_at: "2026-07-26T06:16:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7aa37b65347610bedfb1db8f661e7ba75ef3cdfed0ba73c4ce53d80acace1e48
    source_path: gateway/security/rate-limiting.md
    workflow: 16
---

Gateway 网关实施多项彼此独立的速率限制。它们保护不同的
边界、基于不同的身份确定键值，并以不同的错误形式失败。
本页是所有这些限制的参考。

概览：

| 范围                                | 限制（默认）                     | 键值依据                          | 可配置                   |
| ----------------------------------- | -------------------------------- | -------------------------------- | ------------------------ |
| 身份验证失败（令牌/密码/设备）      | 60 秒内失败 10 次，锁定 5 分钟   | IP + 凭据范围                    | `gateway.auth.rateLimit` |
| 浏览器来源的 WS 身份验证失败        | 相同，local loopback **不**豁免  | IP，或来自回环地址的页面来源      | `gateway.auth.rateLimit` |
| Webhook（`/hooks`）身份验证失败 | 60 秒内失败 20 次，锁定 60 秒    | IP                               | 否                       |
| 控制平面写入 RPC                    | 每种方法 60 秒内 30 个请求       | 方法 + 设备 + IP                 | 否                       |
| ACP 会话创建                        | 10 秒内 120 个会话               | 转换器实例                       | 内部                     |
| Gateway 网关重启周期                | 两次重启之间冷却 30 秒           | 进程                             | 否                       |

## 身份验证尝试（身份验证前）

在处理任何请求之前，系统会按客户端 IP 限制失败的身份验证尝试。
这是针对暴露的 Gateway 网关的暴力破解防护机制。

- 只有_错误的_凭据才会计数。缺少凭据（客户端从未
  发送令牌）和身份验证成功不会消耗配额；身份验证成功会重置该 IP
  的计数器。
- 默认值：每 60 秒允许失败 10 次，之后该 IP 将被锁定 5 分钟。
- 默认豁免 local loopback（`127.0.0.1` / `::1`），因此本地 CLI 会话
  不会被锁定。
- 计数器按凭据类别划分范围，因此针对一个表面的请求洪泛
  不会挤占另一个表面的配额。范围包括共享的 Gateway 网关
  令牌/密码、设备令牌、节点配对、已配对节点重新审批、
  设备引导令牌以及 watchOS 质询签发。

锁定期间，连接尝试将失败并返回：

```json
{
  "code": "INVALID_REQUEST",
  "message": "unauthorized: too many failed authentication attempts (retry later)",
  "retryable": true,
  "retryAfterMs": 297000,
  "details": {
    "code": "AUTH_RATE_LIMITED",
    "authReason": "rate_limited",
    "recommendedNextStep": "wait_then_retry"
  }
}
```

锁定期间，来自其他 IP（包括 local loopback）的尝试不受影响。

在 `openclaw.json` 的 `gateway.auth.rateLimit` 下调整此限制：

```json
{
  "gateway": {
    "auth": {
      "rateLimit": {
        "maxAttempts": 10,
        "windowMs": 60000,
        "lockoutMs": 300000,
        "exemptLoopback": true
      }
    }
  }
}
```

Gateway 网关日志中反复出现 `AUTH_RATE_LIMITED` 条目，意味着有人正在
猜测凭据；请参阅[暴露运行手册](/zh-CN/gateway/security/exposure-runbook)。

### 浏览器来源的连接

携带浏览器 `Origin` 标头的 WebSocket 连接使用相同的
限制，但 local loopback 豁免**始终关闭**——本地浏览器中的恶意页面
仍是不受信任的客户端，因此 localhost 在该路径上不会获得豁免。
当此类连接_来自_回环地址时，其失败次数按规范化后的页面来源确定键值
（例如 `browser-origin:https://evil.example`），而不是按共享的回环 IP，
因此每个来源都有自己的桶；来自非回环地址时，键值仍为客户端 IP。
此行为不可配置。

### Webhooks

HTTP `/hooks` 入口有自己的失败限制器：每个客户端 IP
每 60 秒允许 20 次身份验证失败，之后锁定 60 秒。
local loopback 不豁免。Hook 身份验证成功会重置计数器。受到限流的
请求会收到纯 HTTP `429 Too Many Requests` 响应，其中包含
`Retry-After` 标头（秒）。限制固定；如果合法集成触发了此限制，
应修复其凭据，而不是更频繁地重试。

## 控制平面写入（身份验证后的后备保护）

写入侧管理 RPC（`config.apply`、`config.patch`、`plugins.install`、
`plugins.setEnabled`、`plugins.uninstall`、`update.run`、`worktrees.*`、
`gateway.restart.request`、……）在授权**之后**还会受到额外的速率限制：
按每个 `deviceId+clientIp`、每种方法计算，每 60 秒 30 个请求。

这并非安全边界——调用方已持有 `operator.admin`——而是一项
后备保护，用于限制失控的客户端或智能体循环反复调用高成本操作。
交互式使用永远不会触发此限制；每种方法都有自己的桶，因此切换插件
不会消耗配置写入的配额。

超过限制时，请求将失败并返回可重试错误：

```json
{
  "code": "UNAVAILABLE",
  "message": "rate limit exceeded for config.patch; retry after 35s",
  "retryable": true,
  "retryAfterMs": 34539,
  "details": { "method": "config.patch", "limit": "30 per 60s" }
}
```

客户端应遵循 `retryAfterMs`。此限制固定（不可配置）；
桶会自行过期，并由 Gateway 网关维护任务清理。

## ACP 会话创建

ACP 转换器将每个转换器实例的会话创建限制为每 10 秒最多 120 个
新会话。超过限制时，请求将失败，错误消息中会包含等待时间
（此路径上没有结构化的 `retryAfterMs` 字段）：

```
超过 <method> 的 ACP 会话创建速率限制；请在 <n> 秒后重试。
```

这可以限制循环创建会话的失控客户端；正常的 IDE 和
智能体使用量远低于此限制。

## 重启冷却

Gateway 网关重启请求会先合并，然后在重启周期之间实施 30 秒的冷却。
在冷却期间发出的重启请求不会被拒绝，而是安排在冷却结束后执行。
这与上述控制平面限制器相互独立：`gateway.restart.request` 会消耗一个
控制平面配额槽，_并且_由此产生的重启也必须遵守冷却时间。

## 运维说明

- 所有限制器均位于内存中且按进程隔离，多个 Gateway 网关不会
  共享状态。替换 Gateway 网关进程会清除由 Gateway 网关拥有的
  计数器（身份验证锁定、Webhook 限流和控制平面桶）。重启冷却会
  特意跨进程内重启周期保留——这正是它要限制的对象——并且仅随进程
  重置。ACP 会话上限属于其转换器实例，并在该实例重新创建时重置，
  而不是在 Gateway 网关重启时重置。
- 桶映射有界（硬性条目上限加定期清理），因此
  唯一键洪泛不会导致内存无限增长。
- 当客户端位于反向代理之后时，有效 IP 是解析出的
  客户端 IP；请参阅[受信任代理身份验证](/zh-CN/gateway/trusted-proxy-auth)，了解
  代理标头在能够影响该 IP 之前如何经过验证。
- 不同表面的重试信号有所不同：Gateway 网关 RPC 限制器返回
  `retryable: true` 和 `retryAfterMs`，Webhook 入口使用 HTTP 429
  并附带 `Retry-After` 标头，ACP 则将等待时间嵌入错误消息。
  在所有情况下，都应按指示的时长退避，而不是立即重试。
