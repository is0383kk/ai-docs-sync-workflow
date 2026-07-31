---
read_when:
    - 处理 Gateway 网关协议、客户端或传输协议相关工作
summary: WebSocket Gateway 网关架构、组件和客户端流程
title: Gateway 网关架构
x-i18n:
    generated_at: "2026-07-26T05:45:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8054bd87f738b957c24f8d6965d55365de2293d44902530a9ba778afa597cc7
    source_path: concepts/architecture.md
    workflow: 16
---

## 概览

- 一个长期运行的 **Gateway 网关** 负责所有消息传递界面（通过 Baileys 接入 WhatsApp、通过 grammY 接入 Telegram，以及 Slack、Discord、Signal、iMessage、WebChat）。
- 控制平面客户端（macOS 应用、CLI、Web UI、自动化）通过配置的绑定主机上的 **WebSocket** 连接到 Gateway 网关（默认值为 `127.0.0.1:18789`）。
- **节点**（macOS/iOS/Android/无头设备）也通过 **WebSocket** 连接，但会使用明确的能力/命令声明 `role: node`。
- 每台主机一个 Gateway 网关；它是唯一打开 WhatsApp 会话的位置。
- **画布主机**由 Gateway 网关 HTTP 服务器通过以下路径提供：
  - `/__openclaw__/canvas/`（智能体可编辑的 HTML/CSS/JS）
  - `/__openclaw__/a2ui/`（A2UI 主机）

  它与 Gateway 网关使用相同的端口（默认值为 `18789`）。

## 组件和流程

### Gateway 网关（守护进程）

- 维护提供商连接。
- 公开类型化的 WS API（请求、响应和服务器推送事件）。
- 根据 JSON Schema 验证入站帧。
- 发出 `agent`、`chat`、`presence`、`health`、`heartbeat`、`cron` 等事件。

### 客户端（Mac 应用 / CLI / Web 管理界面）

- 每个客户端使用一个 WS 连接。
- 发送请求（`health`、`status`、`send`、`agent`、`system-presence`）。
- 订阅事件（`tick`、`agent`、`presence`、`shutdown`）。

### 节点（macOS / iOS / Android / 无头设备）

- 使用 `role: node` 连接到**同一台 WS 服务器**。
- 在 `connect` 中提供设备身份；配对**基于设备**（角色为 `node`），审批记录存储在设备配对存储中。
- 公开 `canvas.*`、`camera.*`、`screen.record`、`location.get` 等命令。

协议详情：[Gateway 网关协议](/zh-CN/gateway/protocol)

### WebChat

- 使用 Gateway 网关 WS API 获取聊天记录并发送消息的静态 UI。
- 在远程设置中，通过与其他客户端相同的 SSH/Tailscale 隧道连接。

## 连接生命周期（单个客户端）

```mermaid
sequenceDiagram
    participant Client
    participant Gateway

    Client->>Gateway: 请求：连接
    Gateway-->>Client: 响应（成功）
    Note right of Gateway: 或响应错误并关闭
    Note left of Client: 载荷=hello-ok<br>快照：在线状态 + 健康状态

    Gateway-->>Client: 事件：在线状态
    Gateway-->>Client: 事件：时钟周期

    Client->>Gateway: 请求：智能体
    Gateway-->>Client: 响应：智能体<br>确认 {runId, status:"accepted"}
    Gateway-->>Client: 事件：智能体<br>（流式传输）
    Gateway-->>Client: 响应：智能体<br>最终结果 {runId, status, summary}
```

## 线路协议（摘要）

- 传输方式：WebSocket，使用 JSON 载荷的文本帧。
- 第一帧**必须**是 `connect`。
- 握手后：
  - 请求：`{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
  - 事件：`{type:"event", event, payload, seq?, stateVersion?}`
- `hello-ok.features.methods` / `events` 是设备发现元数据，并非每个可调用辅助路由的生成式完整转储。
- 共享密钥身份验证使用 `connect.params.auth.token` 或 `connect.params.auth.password`，具体取决于配置的 Gateway 网关身份验证模式。
- Tailscale Serve（`gateway.auth.allowTailscale: true`）或非 local loopback `gateway.auth.mode: "trusted-proxy"` 等携带身份的模式通过请求标头完成身份验证，而不使用 `connect.params.auth.*`。
- 私有入口 `gateway.auth.mode: "none"` 会完全禁用共享密钥身份验证；不要对公共/不受信任的入口启用该模式。
- 具有副作用的方法（`send`、`agent`）需要幂等键才能安全重试；服务器会维护一个短期去重缓存。
- 节点必须在 `connect` 中包含 `role: "node"` 以及能力/命令/权限。

## 配对和本地信任

- 所有 WS 客户端（操作员 + 节点）都会在 `connect` 中包含**设备身份**。
- 新的设备 ID 需要配对审批；Gateway 网关会签发一个**设备令牌**，供后续连接使用。
- 可以自动批准直接的本地 local loopback 连接，以保持同主机用户体验流畅。
- OpenClaw 还为受信任的共享密钥辅助流程提供了一条受限的后端/容器本地自连接路径。
- Tailnet 和 LAN 连接（包括同主机的 Tailnet 绑定）仍需明确的配对审批。
- 所有连接都必须对 `connect.challenge` nonce 进行签名。签名载荷 `v3` 还会绑定 `platform` 和 `deviceFamily`；重新连接时，Gateway 网关会固定已配对的元数据，元数据发生变化时需要修复配对。
- **非本地**连接仍需明确批准。
- Gateway 网关身份验证（`gateway.auth.*`）仍适用于**所有**连接，无论本地还是远程。

详情：[Gateway 网关协议](/zh-CN/gateway/protocol)、[配对](/zh-CN/channels/pairing)、
[安全性](/zh-CN/gateway/security)。

## 协议类型和代码生成

- TypeBox schema 定义协议。
- JSON Schema 由这些 schema 生成。
- Swift 模型由 JSON Schema 生成。

## 远程访问

- 首选：Tailscale 或 VPN。
- 替代方案：SSH 隧道

  ```bash
  ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
  ```

- 通过隧道时使用相同的握手和身份验证令牌。
- 远程设置中的 WS 可以启用 TLS 和可选的固定证书。

## 运维快照

- 启动：`openclaw gateway`（前台运行，将日志写入 stdout）。
- 健康状态：通过 WS 使用 `health`（也包含在 `hello-ok` 中）。
- 进程监管：使用 launchd/systemd 自动重启。

## 不变量

- 每台主机只能由一个 Gateway 网关控制一个 Baileys 会话。
- 必须执行握手；如果第一帧不是 JSON 或不是连接帧，将直接关闭连接。
- 事件不会重放；出现缺口时，客户端必须刷新。

## 相关内容

- [Agent loop](/zh-CN/concepts/agent-loop) — 详细的智能体执行周期
- [Gateway 网关协议](/zh-CN/gateway/protocol) — WebSocket 协议约定
- [队列](/zh-CN/concepts/queue) — 命令队列和并发
- [安全性](/zh-CN/gateway/security) — 信任模型和安全加固
