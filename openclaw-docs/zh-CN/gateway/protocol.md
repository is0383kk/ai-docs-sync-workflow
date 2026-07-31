---
read_when:
    - 实现或更新 Gateway 网关 WebSocket 客户端
    - 调试协议不匹配或连接失败
    - 重新生成协议架构/模型
summary: Gateway 网关 WebSocket 协议：握手、帧和版本控制
title: Gateway 网关协议
x-i18n:
    generated_at: "2026-07-26T06:45:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89d637a9070bc6512a182fea0fd890b56287e0080515ba4fba9b2591c6247e0d
    source_path: gateway/protocol.md
    workflow: 16
---

Gateway 网关 WS 协议是 OpenClaw 的唯一控制平面和节点传输协议。操作员和节点客户端（CLI、Web UI、macOS 应用、iOS/Android 节点、无头节点）通过 WebSocket 连接，并在握手时声明**角色**和**权限范围**。

## npm 软件包

这些软件包随 OpenClaw 发布系列一同交付。在初始推出期间，在首个包含软件包的版本发布之前，npm 可能返回 `E404`。

- [`@openclaw/gateway-protocol`](https://www.npmjs.com/package/@openclaw/gateway-protocol)
  发布架构、验证器、TypeScript 类型、轻量级帧和错误辅助工具以及版本常量。其 tarball 包含生成的
  [`protocol.schema.json`](https://unpkg.com/@openclaw/gateway-protocol/protocol.schema.json)
  机器可读契约。
- [`@openclaw/gateway-client`](https://www.npmjs.com/package/@openclaw/gateway-client)
  发布参考 Node 客户端，以及位于
  `@openclaw/gateway-client/browser` 的浏览器安全入口。

有关应用生命周期指导，请参阅
[构建 Gateway 客户端](https://docs.openclaw.ai/gateway/clients)。对于将 Gateway 网关作为子进程进行监管的应用，请参阅
[嵌入 OpenClaw](https://docs.openclaw.ai/gateway/embedding)。

## 传输和帧结构

- WebSocket、文本帧、JSON 载荷。
- 第一帧**必须**是 `connect` 请求。
- 连接前帧大小上限为 64 KiB（`MAX_PREAUTH_PAYLOAD_BYTES`）。握手后，
  遵循 `hello-ok.policy.maxPayload` 和
  `hello-ok.policy.maxBufferedBytes`。启用诊断后，过大的入站帧和缓慢的出站缓冲区会在
  Gateway 网关关闭或丢弃帧之前发出 `payload.large` 事件。这些事件携带 `surface`、字节
  大小、限制和安全原因代码，绝不包含消息正文、附件
  内容、原始帧字节、令牌、Cookie 或密钥。

帧结构：

- 请求：`{type:"req", id, method, params}`
- 响应：`{type:"res", id, ok, payload|error}`
- 事件：`{type:"event", event, payload, seq?, stateVersion?}`

响应错误使用 `{ code, message, details?, retryable?, retryAfterMs? }`。
客户端应根据 `code` 和 `details.code` 进行分支处理；`message` 保持人类可读，
除非兼容性说明另有规定，否则可能发生变化。方法级
授权失败使用顶层 `code: "FORBIDDEN"`，并包含结构化的
缺失权限范围详情：

- 缺失权限范围：`{ code: "MISSING_SCOPE", missingScope, requiredScopes }`。
  `requiredScopes` 是所请求操作的完整已知权限范围集合。
  为兼容旧版客户端，保留旧版 `missing scope: <scope>` 消息。

客户端应首先读取 `details`，仅将旧版消息用作兼容性
回退。`readMissingScopeError` 和 `readMissingScopeErrorDetails` 从
`@openclaw/gateway-protocol/gateway-error-details` 导出；浏览器安全的 Gateway 网关客户端
从 `@openclaw/gateway-client/browser` 重新导出它们。

架构从 `@openclaw/gateway-protocol/schema` 导出为 `GatewayErrorDetailsSchema`、
`MissingScopeErrorDetailsSchema`。
HTTP 权限范围失败会在 `error.details` 下镜像 `MISSING_SCOPE` 对象，并
使用 HTTP 状态码 `403`。

具有副作用的方法需要幂等键（参见架构）。

## 握手

Gateway 网关发送连接前质询：

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

客户端使用 `connect` 响应：

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Gateway 网关使用 `hello-ok` 响应：

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 4,
    "server": { "version": "…", "connId": "…" },
    "features": { "methods": ["…"], "events": ["…"] },
    "snapshot": { "…": "…" },
    "auth": {
      "role": "operator",
      "scopes": ["operator.read", "operator.write"]
    },
    "policy": {
      "maxPayload": 26214400,
      "maxBufferedBytes": 52428800,
      "tickIntervalMs": 15000
    }
  }
}
```

`server`、`features`、`snapshot`、`policy` 和 `auth` 均为
`HelloOkSchema`（`packages/gateway-protocol/src/schema/frames.ts`）的必填项。即使未签发设备令牌，`auth`
也会报告协商后的角色/权限范围（结构如上）。`pluginSurfaceUrls` 是可选项，用于将插件界面名称（例如
`canvas`）映射到有权限范围限制的托管 URL；它可能过期，因此节点使用 `{ "surface": "canvas" }`
调用 `node.pluginSurface.refresh` 以获取新的条目。
已弃用的 `canvasHostUrl` / `canvasCapability` / `node.canvas.capability.refresh`
路径不受支持；请使用插件界面。
快照中可选的 `appliedConfigHash` 是活动 Gateway 网关运行时接受的已解析源配置修订版本。客户端可将其与
`config.get.configRevisionHash` 比较，以确定较新的已保存配置是否仍
需要重启。`config.get.hash` 仍是配置写入冲突防护所使用的
原始根文件修订版本。

当 Gateway 网关仍在完成启动辅助进程时，`connect` 可能返回
可重试的 `UNAVAILABLE` 错误，其中包含 `details.reason: "startup-sidecars"` 和
`retryAfterMs`。应在连接预算内重试，而不是将其视为
终止性的握手失败。

签发设备令牌时，`hello-ok.auth` 会添加该令牌：

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

内置二维码/设置代码引导是移动端交接路径。成功的
基准设置代码连接会返回一个主节点令牌和一个受限的
操作员令牌：

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

此操作员交接是有意限制的：足以启动移动端
操作员循环和原生设置，包括用于读取 Talk 模式
配置的 `operator.talk.secrets`，但不包含配对变更权限范围，也不包含 `operator.admin`。更广泛的
配对/管理员访问需要单独批准的配对或令牌流程。仅当引导身份验证通过受信任的
传输（`wss://` 或环回/本地配对）运行时，才持久化
`hello-ok.auth.deviceTokens`。

受信任的同进程后端客户端（`client.id: "gateway-client"`、
`client.mode: "backend"`）在使用共享 Gateway 网关令牌/密码进行身份验证时，可在直接环回连接上省略 `device`。
此路径仅供内部控制平面 RPC 使用（例如子智能体会话更新），并避免
过时的 CLI/设备配对基准阻止本地后端工作。远程、
浏览器来源、节点以及显式设备令牌/设备身份客户端仍
需要经过常规配对和权限范围升级检查。

### 工作节点角色和封闭协议

云端工作节点通过 Gateway 网关所有、绑定主机密钥的 SSH 隧道使用专用环回入口。它仅接受工作节点身份，
绝不分派通用身份验证、节点事件、操作员 RPC 或插件方法。严格的 `connect`
会验证静态哈希存储的短期凭据，该凭据绑定到环境、软件包
哈希、所有者纪元、RPC 集版本、过期时间和一个可为空的会话；它还会
分别检查当前版本和功能集。成功时返回最小化的
`worker-hello-ok`；功能协商独立于通用协议
版本。帧大小保持在 64 KiB 以下，但经协商的 `worker.inference.start`
帧最大可达 25 MiB。封闭允许列表包含 `worker.heartbeat`、
`worker.transcript.commit`、`worker.live-event`、`worker.inference.start` 和
`worker.inference.cancel`。

转录提交使用所有者纪元隔离、Gateway 网关所有的会话绑定、
基叶比较并交换以及持久序列重放；Gateway 网关通过常规会话写入器生成
转录条目和父级 ID。每次 RPC 都会重新检查所有权和
过期时间。

### 客户端能力

操作员客户端可以在 `connect.params.caps` 中声明可选能力：

- `tool-events`：接受结构化工具生命周期事件。
- `inline-widgets`：可以渲染托管的内联小组件工具结果。

客户端能力描述的是已连接的客户端，而非授权。智能体工具可以声明所需能力；除非每项要求都出现在发起请求的客户端的 `caps` 中，否则 Gateway 网关会省略这些工具。由渠道发起的运行没有 Gateway 网关客户端能力，因此即使工具策略明确允许，能力受限的工具也不可用。

### 节点连接示例

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

节点在连接时声明能力：

- `caps`：高级类别，例如 `camera`、`canvas`、`screen`、
  `location`、`voice`、`talk`。
- `commands`：用于调用的命令允许列表。
- `permissions`：细粒度开关（例如 `screen.record`、`camera.capture`）。

Gateway 网关将这些视为声明，并在服务器端强制执行允许列表。

## 角色和权限范围

有关完整的操作员权限范围模型、批准时检查和共享密钥
语义，请参阅[操作员权限范围](/zh-CN/gateway/operator-scopes)。

角色：

- `operator`：控制平面客户端（CLI/UI/自动化）。
- `node`：能力宿主（摄像头/屏幕/画布/system.run）。
- `worker`：专用封闭工作节点协议上的云端执行宿主。

操作员权限范围（`src/gateway/operator-scopes.ts`），完整封闭集合：

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

带有 `includeSecrets: true` 的 `talk.config` 需要 `operator.talk.secrets`（或
`operator.admin`）。包含密钥时，从 `talk.resolved.config.apiKey` 读取活动 Talk 模式提供商
凭据；`talk.providers.<id>.apiKey`
保持源结构，可能是 SecretRef 对象或经过遮盖的字符串。

插件注册的 Gateway 网关 RPC 方法可以请求自己的操作员权限范围，
但以下保留的核心前缀始终解析为 `operator.admin`
（`src/shared/gateway-method-policy.ts`）：`config.*`、`exec.approvals.*`、
`wizard.*`、`update.*`。

方法权限范围只是第一道关卡。通过
`chat.send` 访问的部分斜杠命令会执行更严格的命令级检查：即使 Gateway 网关客户端
已持有较低的操作员权限范围，持久化 `/config set` 和
`/config unset` 写入仍需要 `operator.admin`。

除基础方法权限范围（`operator.pairing`）外，`node.pair.approve` 还会在批准时根据待处理请求声明的
`commands`（`src/infra/node-pairing-authz.ts`）执行额外的权限范围检查：

| 声明的命令                                                                                                                    | 所需权限范围                          |
| ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| 无                                                                                                                            | `operator.pairing`                    |
| 普通命令                                                                                                                      | `operator.pairing` + `operator.write` |
| 包含 `system.run`、`system.run.prepare`、`system.which`、`browser.proxy`、`fs.listDir` 或 `system.execApprovals.get/set` | `operator.pairing` + `operator.admin` |

### 能力/命令/权限（节点）

节点在连接时声明能力：

- `caps`：高级能力类别，例如 `camera`、`canvas`、`screen`、
  `location`、`voice` 和 `talk`。
- `commands`：调用命令的允许列表。
- `permissions`：细粒度开关（例如 `screen.record`、`camera.capture`）。

Gateway 网关将这些视为**声明**，并在服务器端强制执行允许列表。
成功连接或重新连接后，已连接的节点可以通过 `node.pluginTools.update` 发布可选的、对智能体可见的插件或 MCP 工具
描述符。无头节点主机需要重启才能应用声明式 MCP 清单
更改。此更新方法是唯一的发布路径；`connect` 参数不接受插件工具描述符。每个描述符必须使用提供商安全的工具 `name`，并指定
节点当前命令允许列表中的 `command`。Gateway 网关信任已配对节点提供的描述符
元数据，过滤超出已批准命令
范围的描述符，在节点断开连接时移除这些描述符，并拒绝操作员
尝试修改其他节点目录的操作。设置 `gateway.nodes.pluginTools.enabled: false`
可忽略节点发布的描述符。

已连接的节点主机通过
`node.skills.update` 发布其完整的技能替换目录。此节点角色方法是唯一的节点技能发布
路径；`connect` 参数不接受技能。每个描述符均包含安全的
名称、描述和有界的 `SKILL.md` 内容。Gateway 网关使用常规技能加载器解析该
内容，在节点连接期间将其纳入智能体技能快照，
并在断开连接时将其移除。设置
`gateway.nodes.allowSkills: false` 可忽略节点发布的技能。

## 在线状态

- `system-presence` 返回按设备身份标识的条目，其中包括
  `deviceId`、`roles` 和 `scopes`，因此即使设备同时以操作员和节点身份连接，
  UI 也能为每台设备显示一行。
- `node.list` 包含可选的 `lastSeenAtMs` 和 `lastSeenReason`。已连接的
  节点使用原因 `connect` 报告当前连接时间；已配对节点还可以
  通过受信任的节点事件报告持久的后台在线状态。

原生 macOS 节点还可以发送经过身份验证的 `node.presence.activity` 事件，
其中包含有界的输入空闲时间。Gateway 网关使用自身时钟推导活动时间戳，
通过 `node.list` 和
`node.describe` 公开最近活跃的已连接 Mac，并向具有读取权限范围的客户端广播 `node.presence` 更新。
用户禁用活动共享时，应用会发送 `{ "action": "clear" }`；
Gateway 网关仅清除该经过身份验证的确切节点连接的时间戳。
早于这一已确认操作的 Gateway 网关会将其作为未处理操作返回，因此 Mac
节点会重新连接一次，并让断开连接清理移除旧的连接状态。
有关选择、隐私、模型上下文和通知路由行为，请参阅[活动计算机在线状态](/zh-CN/nodes/presence)。

### 节点后台存活事件

节点调用带有 `event: "node.presence.alive"` 的 `node.event`，以记录
已配对节点在后台唤醒期间处于存活状态，但不将其标记为已连接：

```json
{
  "event": "node.presence.alive",
  "payloadJSON": "{\"trigger\":\"silent_push\",\"sentAtMs\":1737264000000,\"displayName\":\"Peter's iPhone\",\"version\":\"2026.4.28\",\"platform\":\"iOS 18.4.0\",\"deviceFamily\":\"iPhone\",\"modelIdentifier\":\"iPhone17,1\",\"pushTransport\":\"relay\"}"
}
```

`trigger` 是封闭枚举：`background`、`silent_push`、`bg_app_refresh`、
`significant_location`、`manual`、`connect`。未知值会规范化为
`background`（`src/shared/node-presence.ts`）。该事件仅对
经过身份验证的节点设备会话持久化；无设备或未配对的会话返回
`handled: false`。

成功的 Gateway 网关会返回结构化结果：

```json
{
  "ok": true,
  "event": "node.presence.alive",
  "handled": true,
  "reason": "persisted"
}
```

较旧的 Gateway 网关针对 `node.event` 可能仅返回 `{ "ok": true }`；应将其视为已确认的 RPC，而不是持久化在线状态。

## 广播事件作用域

服务器推送的广播事件受作用域限制，因此仅限配对作用域或仅限节点的会话不会被动接收会话内容（`src/gateway/server-broadcast.ts`）：

- 聊天、智能体和工具结果帧（流式 `agent` 事件、工具结果事件）至少需要 `operator.read`。没有该作用域的会话会完全跳过这些帧。
- 插件定义的 `plugin.*` 广播默认仅限 `operator.write` 或 `operator.admin`；`plugin.approval.requested` / `plugin.approval.resolved` 等显式条目则改用 `operator.approvals`。
- 状态/传输事件（`heartbeat`、`presence`、`tick`、连接/断开连接生命周期）不受限制，因此每个已通过身份验证的会话都可以观察传输健康状况。
- 未知的广播事件系列默认受作用域限制（失败时关闭），除非已注册的处理程序明确放宽限制。

每个客户端连接都维护各自的每客户端序列号，因此即使不同客户端看到事件流中经作用域过滤的不同子集，广播在相应套接字上仍保持单调有序。

## RPC 方法系列

`hello-ok.features.methods` 是一个保守的发现列表，由 `src/gateway/server-methods-list.ts` 加上已加载的插件/渠道方法导出构成——它不是每个方法的自动生成转储，并且某些方法（例如 `push.test`、`web.login.start`、`web.login.wait`、`sessions.usage`）即使是真实且可调用的方法，也会被有意排除在发现范围之外。应将其视为功能发现，而不是 `src/gateway/server-methods/*.ts` 的完整枚举。

<AccordionGroup>
  <Accordion title="系统和身份">
    - `health` 返回缓存的或新近探测的 Gateway 网关健康快照。
    - `diagnostics.stability` 返回最近的有界诊断稳定性记录：事件名称、计数、字节大小、内存读数、队列/会话状态、渠道/插件名称、会话 ID。不包含聊天文本、Webhook 正文、工具输出、原始请求/响应正文、令牌、Cookie 或密钥。需要 `operator.read`。
    - `status` 返回 `/status` 风格的 Gateway 网关摘要；敏感字段仅向具有管理员作用域的操作员客户端提供。
    - `gateway.identity.get` 返回中继和配对流程所使用的 Gateway 网关设备身份。
    - `system-presence` 返回已连接操作员/节点设备的当前在线状态快照。
    - `system-event` 追加系统事件，并可更新/广播在线状态上下文。
    - `last-heartbeat` 返回最近持久化的 Heartbeat 事件。
    - `set-heartbeats` 开启或关闭 Gateway 网关上的 Heartbeat 处理。
    - 仅当受跟踪的 Gateway 网关工作处于空闲状态时，`gateway.suspend.prepare` 才会创建一个短期协作式暂停租约。`gateway.suspend.status` 检查该租约，`gateway.suspend.resume` 则在恢复后或主机操作中止后释放该租约。

  </Accordion>

  <Accordion title="模型和用量">
    - `models.list` 返回运行时允许的模型目录。请参阅下文的“`models.list` 视图”。
    - `usage.status` 返回提供商用量窗口/剩余额度摘要。
    - `usage.cost` 返回某个日期范围内的汇总成本用量摘要。传入 `agentId` 可指定一个智能体，传入 `agentScope: "all"` 可汇总已配置的智能体。
    - `doctor.memory.status` 返回当前默认 Agent 工作区的向量记忆/缓存嵌入就绪状态。仅在需要显式实时探测嵌入提供商时传入 `{ "probe": true }` 或 `{ "deep": true }`。传入 `{ "agentId": "agent-id" }` 可将 Dreaming 存储统计限定到一个 Agent 工作区；省略时会汇总已配置的 Dreaming 工作区。
    - `doctor.memory.dreamDiary`、`doctor.memory.backfillDreamDiary`、`doctor.memory.resetDreamDiary`、`doctor.memory.resetGroundedShortTerm`、`doctor.memory.repairDreamingArtifacts` 和 `doctor.memory.dedupeDreamDiary` 接受可选的 `{ "agentId": "agent-id" }`；省略时，它们会对已配置的默认 Agent 工作区执行操作。
    - `doctor.memory.remHarness` 为远程控制平面客户端返回有界的只读 REM harness 预览，包括工作区路径、记忆片段、渲染后的有依据 Markdown，以及深度提升候选项。需要 `operator.read`。
    - `sessions.usage` 返回每个会话的用量摘要。传入 `agentId` 可指定一个智能体，传入 `agentScope: "all"` 可一并列出已配置的智能体。
      两种用量方法均接受带有 IANA `timeZone` 的 `mode: "specific"`，用于支持夏令时的日历日边界和分桶。较旧客户端仍支持 `utcOffset`，当 Gateway 网关运行时无法识别请求的时区时，也会将其用作后备方案。
    - `sessions.usage.timeseries` 返回一个会话的时序用量。
    - `sessions.usage.logs` 返回一个会话的用量日志条目。

  </Accordion>

  <Accordion title="渠道和登录辅助工具">
    - `channels.status` 返回内置 + 随附渠道/插件的状态摘要。
    - 如果渠道支持，`channels.logout` 会注销指定的渠道/账号。
    - `web.login.start` 为当前支持二维码的 Web 渠道提供商启动二维码/Web 登录流程。
    - `web.login.wait` 等待该流程完成，并在成功后启动渠道。
    - `push.test` 向已注册的 iOS 节点发送测试 APNs 推送。
    - `voicewake.get` 返回已存储的唤醒词触发项。
    - `voicewake.set` 更新唤醒词触发项并广播该变更。

  </Accordion>

  <Accordion title="插件管理">
    - `plugins.list`（`operator.read`）返回已安装的插件清单，以及本地精选的官方推荐项、诊断信息和当前安装模式是否允许修改。
    - `plugins.search`（`operator.read`）搜索可安装的 ClawHub 代码插件和插件包系列。传入非空的 `query`，以及可选的 `limit`，其取值范围为 1 到 100。
    - `plugins.install`（`operator.admin`）安装通过 `{ source: "official", pluginId }` 指定的官方目录条目，或通过 `{ source: "clawhub", packageName, version?, acknowledgeClawHubRisk? }` 指定的 ClawHub 软件包。ClawHub 安装会保留 Gateway 网关的信任、完整性和安装策略检查。安装成功后需要重启 Gateway 网关。
    - `plugins.setEnabled`（`operator.admin`）通过 `{ pluginId, enabled }` 更改一个已安装插件的启用策略。响应包含更新后的目录条目、重启元数据以及所有插槽选择警告。
    - `plugins.uninstall`（`operator.admin`）通过 `{ pluginId }` 移除一个外部安装的插件，包括配置引用、安装记录和托管文件。内置插件无法卸载，只能禁用。响应会列出移除操作，并且始终要求重启 Gateway 网关。

  </Accordion>

  <Accordion title="消息和日志">
    - `send` 是用于在聊天运行器之外按渠道、账户和线程目标直接发送出站消息的 RPC。
    - `logs.tail` 返回已配置的 Gateway 网关文件日志尾部内容，并提供游标、数量限制和最大字节数控制。

  </Accordion>

  <Accordion title="操作员终端">
    - `terminal.open` 为显式指定的 `agentId` 或默认智能体启动主机 PTY，并返回解析后的智能体、工作目录、shell 和隔离状态。
    - `terminal.input`、`terminal.resize` 和 `terminal.close` 仅能操作归调用连接所有的会话。
    - `terminal.upload` 接受一个不超过 16 MiB 的 base64 文件，将其暂存到会话所在 Gateway 网关或已配对节点主机上的私有 24 小时临时目录中，并返回绝对路径。调用方仍须粘贴或以其他方式使用该路径；此 RPC 绝不会写入终端输入或执行命令。
    - `terminal.data` 和 `terminal.exit` 事件仅流式传输到拥有该会话的连接。
    - 连接断开的会话会被分离，而不会被终止：它们在 `gateway.terminal.detachedSessionTimeoutSeconds` 内仍可重新附加（默认值为 300；`0` 会恢复断开连接时终止），同时近期输出会累积在有界的服务端缓冲区中。
    - `terminal.list` 返回可附加的会话；`terminal.attach` 将活动或已分离的会话重新绑定到调用连接，并返回回放缓冲区（类似 tmux 的接管——先前的活动所有者会收到 `terminal.exit`，原因为 `detached`）；`terminal.text` 在不附加的情况下以纯文本形式读取缓冲区。
    - 每个终端方法都需要 `operator.admin`；`gateway.terminal.enabled` 必须显式设为 true。完全沙箱隔离的智能体会被拒绝，并且智能体策略变更会关闭现有和进行中的 PTY，包括已分离的 PTY。

  </Accordion>

  <Accordion title="Talk 和 TTS">
    - `talk.catalog` 返回用于语音、流式转录和实时语音的只读 Talk 提供商目录：规范提供商 ID、注册表别名、标签、配置状态、可选的组级 `ready` 结果、公开的模型/语音 ID、规范模式、传输方式、智能策略以及实时音频/能力标志；不会返回提供商密钥，也不会修改全局配置。当前 Gateway 网关会在应用运行时提供商选择后设置 `ready`；在旧版 Gateway 网关上若缺少该值，应视为未经验证。
    - `talk.config` 返回生效的 Talk 配置载荷；`includeSecrets` 需要 `operator.talk.secrets`（或 `operator.admin`）。
    - `talk.session.create` 为 `realtime/gateway-relay`、`transcription/gateway-relay` 或 `stt-tts/managed-room` 创建由 Gateway 网关所有的 Talk 会话。对于 `stt-tts/managed-room`，传入 `sessionKey` 的 `operator.write` 调用方还必须传入 `spawnedBy`，以获得限定范围的会话密钥可见性；创建无范围的 `sessionKey` 以及 `brain: "direct-tools"` 需要 `operator.admin`。
    - `talk.session.join` 验证托管房间会话令牌，按需发出 `session.ready` 或 `session.replaced`，并返回房间/会话元数据及近期 Talk 事件，但绝不返回明文令牌或其哈希值。
    - `talk.session.appendAudio` 将 base64 PCM 输入音频追加到由 Gateway 网关所有的实时中继和转录会话。
    - `talk.session.startTurn`、`talk.session.endTurn` 和 `talk.session.cancelTurn` 驱动托管房间的轮次生命周期，并在清除状态前拒绝过期轮次。
    - `talk.session.cancelOutput` 停止助手音频输出，主要用于 Gateway 网关中继会话中受 VAD 控制的插话。
    - `talk.session.submitToolResult` 完成由 Gateway 网关所有的实时中继会话所发出的提供商工具调用。请求会等待提供商桥接层公开的任何异步完成信号；提交失败时，关联的运行会保持活动状态，并且不会发出成功的工具结果事件。传入 `options: { willContinue: true }` 可提供临时工具输出；当提供商桥接层声明支持抑制，且结果不应启动另一响应时，传入 `options: { suppressResponse: true }`。
    - `talk.session.steer` 将活动运行的语音控制发送到由 Gateway 网关所有、以智能体为后端的 Talk 会话：`{ sessionId, text, mode? }`，其中 `mode` 为 `status`、`steer`、`cancel` 或 `followup`；省略模式时，会根据口述文本进行分类。
    - `talk.session.close` 关闭由 Gateway 网关所有的中继、转录或托管房间会话，并发出终止 Talk 事件。
    - `talk.mode` 为 WebChat/Control UI 客户端设置并广播当前的 Talk 模式状态。
    - `talk.client.create` 使用 `webrtc` 或 `provider-websocket` 创建或恢复由客户端所有的实时提供商会话，同时由 Gateway 网关负责凭据、指令、工具策略以及返回的 `voiceSessionId`。客户端传入 `sessionKey`，并在一次通话期间替换提供商传输时复用 `voiceSessionId`。
    - `talk.client.transcript` 将一个已最终确定的 `{ role, text }` 项追加到普通智能体会话中。必需的 `entryId` 在 `voiceSessionId` 内具有幂等性；重试不会重复添加转录消息。
    - `talk.client.close` 在待处理的转录写入完成后关闭逻辑语音会话。关闭操作具有幂等性，并且可能向该会话最后使用的非 WebChat 渠道发送仅包含变更的通话摘要。
    - `talk.client.toolCall` 允许由客户端所有的实时传输将提供商工具调用转发给 Gateway 网关策略。首个受支持的工具是 `openclaw_agent_consult`；客户端获得运行 ID，并等待正常的聊天生命周期事件，然后再提交提供商专用的工具结果。与语音绑定的高影响操作会返回 `VOICE_CONFIRMATION_REQUIRED:<id>`，直至之后最终确定的用户话语明确确认该项确切操作，并且下一次咨询提供 `confirmationId`。
    - `talk.client.steer` 为由客户端所有的实时传输发送活动运行的语音控制。Gateway 网关通过 `sessionKey` 解析活动的嵌入式运行，并返回结构化的接受/拒绝结果，而不是静默丢弃引导操作。
    - `talk.event` 是实时、转录、STT/TTS、托管房间、电话和会议适配器的统一 Talk 事件渠道。
    - `talk.speak` 通过当前活动的 Talk 语音提供商合成语音。
    - `tts.status` 返回 TTS 启用状态、活动提供商、回退提供商和提供商配置状态。
    - `tts.providers` 返回可见的 TTS 提供商清单。
    - `tts.enable` 和 `tts.disable` 切换 TTS 偏好设置状态。
    - `tts.setProvider` 更新首选 TTS 提供商。
    - `tts.convert` 执行一次性文本转语音转换。
    - `tts.speak`（`operator.write`）使用已配置的通用 TTS 提供商链渲染非空的 `text`，并以内联的 `audioBase64` 形式返回一个完整音频片段，同时返回 `provider`，以及可选的 `outputFormat`、`mimeType` 和 `fileExtension` 元数据。与 `tts.convert` 不同，它不返回 Gateway 网关本地路径；与 `talk.speak` 不同，它不需要 Talk 提供商。超过 `tts.maxTextLength` 的文本会返回 `INVALID_REQUEST`；合成失败会返回 `UNAVAILABLE`。

  </Accordion>

  <Accordion title="密钥、配置、更新和向导">
    - `secrets.reload` 重新解析活动的 SecretRef，并以原子方式发布可感知所有者的运行时状态。符合条件的所有者故障可通过 `warningCount` 以冷降级或陈旧降级状态发布；严格模式下或未映射的故障会拒绝重新加载，并保留活动快照。
    - `secrets.resolve` 为特定的命令/目标集合解析命令目标密钥分配。
    - `config.get` 返回当前的磁盘配置快照、原始根文件 `hash`、已解析的 `configRevisionHash`，以及活动 Gateway 网关运行时所接受的已解析修订版本对应的可选 `appliedConfigHash`。
    - `config.set` 写入经过验证的配置载荷。
    - `config.patch` 合并部分配置更新。破坏性的数组替换要求受影响的路径位于 `replacePaths` 中；数组条目下的嵌套数组使用 `[]` 路径，例如 `agents.entries.*.skills`。
    - `config.apply` 验证并替换完整配置载荷。
    - `config.schema` 返回 Control UI 和 CLI 工具使用的实时配置架构载荷：架构、`uiHints`、版本、生成元数据，以及可加载时的插件和渠道架构元数据。它包含与 UI 相同的标签/帮助文本所生成的 `title` / `description` 元数据；存在匹配的字段文档时，还包括嵌套对象、通配符、数组项和 `anyOf` / `oneOf` / `allOf` 组合分支。
    - `config.schema.lookup` 返回一个配置路径的路径范围查询载荷：规范化路径、浅层架构节点、匹配的提示和 `hintPath`、可选的 `reloadKind`，以及供 UI/CLI 逐层深入查看的直接子项摘要。`reloadKind` 是 `restart`、`hot` 或 `none`（`src/config/schema.ts`）之一，并与请求路径对应的 Gateway 网关配置重新加载规划器保持一致。查询架构节点会保留面向用户的文档和常用验证字段（`title`、`description`、`type`、`enum`、`const`、`format`、`pattern`、数值/字符串/数组/对象边界、`additionalProperties`、`deprecated`、`readOnly`、`writeOnly`）。子项摘要会公开 `key`、规范化的 `path`、`type`、`required`、`hasChildren`、可选的 `reloadKind`，以及匹配的 `hint` / `hintPath`。
    - `update.run` 运行 Gateway 网关更新流程，并且仅在更新成功时安排重启；具有会话的调用方可包含 `continuationMessage`，以便启动后通过重启续接队列恢复一次后续智能体轮次。来自控制平面的包管理器更新和受监管的 Git 检出更新会使用分离式托管服务交接，而不是在实时 Gateway 网关内替换软件包树或更改检出内容/构建输出。已启动的交接会返回 `ok: true`，其中包含 `result.reason: "managed-service-handoff-started"` 和 `handoff.status: "started"`。由同一 Gateway 网关进程处理的第二个并发 `update.run` 会返回 `ok: false`，其中包含 `result.reason: "managed-service-handoff-already-running"` 和 `handoff.status: "already-running"`；其续接请求不会被接受，因此调用方可在活动更新完成后重试。独立的 CLI 更新程序和替代 Gateway 网关进程不受此进程本地保护机制约束。不可用或失败的交接会返回 `ok: false`，其中包含 `managed-service-handoff-unavailable` 或 `managed-service-handoff-failed`；需要手动通过 shell 更新时，还会包含 `handoff.command`。“不可用”表示 OpenClaw 缺少安全的监管程序边界或持久服务身份，例如 systemd 的 `OPENCLAW_SYSTEMD_UNIT`。在已启动的交接期间，重启哨兵可能会短暂报告 `stats.reason: "restart-health-pending"`；续接操作将延迟到 CLI 验证重启后的 Gateway 网关并写入最终的 `ok` 哨兵后执行。
    - `update.status` 刷新并返回最新的更新重启哨兵，包括可用时重启后的运行版本。
    - `wizard.start`、`wizard.next`、`wizard.status` 和 `wizard.cancel` 通过 WS RPC 公开新手引导向导。

  </Accordion>

  <Accordion title="智能体和工作区辅助功能">
    - `agents.list` 返回 Gateway 网关可见的智能体条目，包括有效的模型/运行时元数据，以及可选的语义 `kind`（`agent` 或 `system`）。客户端声明 `agent-kind` 握手能力后可接收完整的类型化名册；不具备该能力的客户端仍使用不含系统行、对旧版选择器安全的名册。可感知种类的客户端会从普通选择器中排除 `system` 行，同时在诊断视图中保留这些行。较旧的 v4 Gateway 网关可能返回不含 `kind` 的行。
    - `agents.create`、`agents.update` 和 `agents.delete` 管理智能体记录和工作区连接。
    - `agents.files.list`、`agents.files.get` 和 `agents.files.set` 管理向智能体公开的引导工作区文件。
    - `audit.activity.list` 返回带版本的纯元数据活动账本；`audit.list` 仍是兼容性安全的运行/工具 RPC。
    - `agents.workspace.list` 和 `agents.workspace.get`（`operator.read`）为 [操作员权限范围](/zh-CN/gateway/operator-scopes)中所述可信操作员域内的客户端提供对智能体工作区目录的只读分页浏览。请求仅接受工作区相对路径；读取操作始终限制在经过 realpath 解析的工作区根目录内（拒绝通过符号链接和硬链接逃逸），受大小上限约束，并且仅限 UTF-8 文本和常见图像类型（base64）。响应不会公开主机工作区路径。此命名空间中没有写入操作。
    - `tasks.list`、`tasks.get` 和 `tasks.cancel` 向 SDK 和操作员客户端公开 Gateway 网关任务账本。请参阅下方的[任务账本 RPC](#task-ledger-rpcs)。
    - `artifacts.list`、`artifacts.get` 和 `artifacts.download` 针对明确的 `sessionKey`、`runId` 或 `taskId` 范围，公开从转录记录派生的工件摘要和下载。运行和任务查询会在服务器端解析所属会话，并且仅返回来源匹配的转录媒体；对于不安全或本地 URL 来源，则返回不支持下载，而不会在服务器端获取。
    - `environments.list` 和 `environments.status` 保留 Gateway 网关本地环境和节点环境发现功能。已配置的云端工作节点以及早期配置文件留下的持久记录会添加 `worker` 元数据，其中包含 `providerId`、可选的 `leaseId`、`state`、`ageMs`、可选的 `idleMs` 和 `attachedSessionIds`。工作节点生命周期状态包括 `requested`、`provisioning`、`bootstrapping`、`ready`、`attached`、`idle`、`draining`、`destroying`、`destroyed`、`failed` 和 `orphaned`。
    - `environments.create`（`{ profileId, idempotencyKey }`）通过已配置的插件提供商配置文件预配工作节点；使用相同键重试时会复用持久操作。`environments.destroy`（`{ environmentId }`）请求以幂等方式拆除持久工作节点环境。两者都要求 `operator.admin`，都属于控制平面写入，并返回与状态响应所用结构相同的环境摘要。
    - `agent.identity.get` 返回智能体或会话的有效助手身份。
    - `agent.wait` 等待一次运行结束，并在可用时返回终止快照。

  </Accordion>

  <Accordion title="会话控制">
    - `sessions.list` 返回当前会话索引；配置 Agent Runtimes 后端时，其中包括每行的 `agentRuntime` 元数据。启用云端工作节点放置或存在持久恢复状态时，会话行还会包括已关闭的 `placement` 状态（`local`、`requested`、`provisioning`、`syncing`、`starting`、`active`、`draining`、`reconciling`、`reclaimed` 或 `failed`），以及特定于状态的环境、所有者纪元、工作区、捆绑包、ACK 游标或恢复字段。
    - `sessions.subscribe` 和 `sessions.unsubscribe` 为当前 WS 客户端开启或关闭会话变更事件订阅。
    - `sessions.messages.subscribe` 和 `sessions.messages.unsubscribe` 为一个会话开启或关闭转录/消息事件订阅。传递 `includeApprovals: true` 后，还会接收经过清理的 `session.approval` 生命周期事件，适用于其持久化受众包括该确切会话，且审查者绑定授权订阅客户端的审批。此时，订阅响应会包括有界的待处理 `approvalReplay`；当 `truncated` 为 false 时，它是权威数据。此选择加入按每次订阅调用生效，并非持久设置：在不带 `includeApprovals: true` 的情况下重新订阅同一会话，会移除现有的审批订阅。除正常的会话读取权限外，此选择加入还要求具备 `operator.admin`，或在已配对设备上具备 `operator.approvals`。
    - `sessions.preview` 返回特定会话键的有界转录预览。
    - `sessions.describe` 返回与确切会话键对应的一行 Gateway 网关会话数据。
    - `sessions.resolve` 解析会话目标或将其规范化。
    - `sessions.create` 创建新的会话条目。可选的 `model` 和 `thinkingLevel` 值以原子方式持久化初始模型和推理覆盖项。`worktree: true` 预配托管工作树；可选的 `worktreeBaseRef`/`worktreeName` 用于选择基础引用和分支名称，而 `execNode`（`operator.admin`）将会话 Exec 绑定到节点主机。创建的工作树会在结果中返回，并持久化到会话行（`worktree: { id, branch, repoRoot }`）。如果条目已创建，但其嵌套的初始 `chat.send` 被拒绝，成功结果会包括 `runStarted: false` 和 `runError`；客户端可以保留提示词，并使用返回的会话键重试。传递 `parentSessionKey` 和 `emitCommandHooks: true` 的调用方还应声明独立子会话的生命周期处置方式：`succeedsParent: true` 以 `session_end` 结束父会话，而 `false` 保持父会话处于活动状态，仅发出子会话的 `session_start`。省略 `succeedsParent` 可为现有客户端保留旧版父会话滚转行为。该处置要求同时存在父级关联和命令钩子；分叉不能使其父会话成功结束。主会话的原地重置行为保持不变，因为不会创建独立子会话。新行会带有来自可信创建接缝的只写一次创建来源信息（`createdVia`、`createdActor`、`createdAt`）；采用现有键时绝不会重新写入这些信息。对于人类用户资料参与者，投影会话行时会从当前用户资料解析 `createdActor.label`，且该值绝不会存储在会话条目中，因此用户资料重命名不会导致其发生偏移。会话行还包含 `parentSessionKey`（导航父级，持久化）、`controlOwnerSessionKey`（实时运行时控制器）、`forkSource`（分叉的确切源键 + 转录代次）以及 `previousSessionId`（同一键下的上一转录代次）。
    - `sessions.dispatch`（`operator.admin`）将拥有会话专属托管工作树的现有本地 OpenClaw 会话移动到已配置的云端工作节点配置文件。传递 `{ key, profileId, agentId? }`。未配置工作节点配置文件时，此方法不存在；它会在排空活动工作前关闭本地轮次准入，并且仅在放置达到 `active` 工作节点所有权后才返回。调度是单向的；此 RPC 不支持从工作节点拉回本地。
    - `sessions.groups.list`、`sessions.groups.put`、`sessions.groups.rename` 和 `sessions.groups.delete` 管理 Gateway 网关所有的自定义会话组目录（名称 + 显示顺序）。成员关系保留在每个会话的 `category` 字段中；重命名和删除操作会在服务器端更新成员会话。
    - `sessions.send` 向现有会话发送消息。
    - `sessions.steer` 是用于活动会话的中断并 Steer 变体。
    - `sessions.abort` 中止会话的活动工作。传递 `key` 以及可选的 `runId`，或者仅传递 `runId`，用于 Gateway 网关可以解析到会话的活动运行。提供 `runId` 会将取消操作限定在该次运行。对于仅含键的非全局请求，设置 `clearQueued: true` 后，还会丢弃该会话所有的后续队列和通道队列。省略 `clearQueued` 的现有调用方会保留这些队列。字面量 `global` 键保留现有的 Agent 限定 `chat.abort` 所有权规则，并且不会执行非全局后续队列或通道队列清理。
    - `sessions.patch` 更新会话元数据/覆盖项，并报告解析后的规范模型及有效的 `agentRuntime`。生成沿袭（`spawnedBy`、`spawnedWorkspaceDir`、`spawnedCwd`、`spawnDepth`、`subagentRole`、`subagentControlScope`）不再允许公开修补；这些事实由可信创建路径写入一次，仍发送这些字段的请求会被拒绝。
    - `sessions.reset`、`sessions.delete` 和 `sessions.compact` 执行会话维护。
    - `sessions.get` 返回完整的已存储会话行。
    - 聊天执行仍使用 `chat.history`、`chat.send`、`chat.abort` 和 `chat.inject`。`chat.history` 会针对 UI 客户端进行显示规范化：从可见文本中剥离内联指令标签；剥离纯文本工具调用 XML 载荷（`<tool_call>...</tool_call>`、`<function_call>...</function_call>`、`<tool_calls>...</tool_calls>`、`<function_calls>...</function_calls>` 以及被截断的工具调用块）和泄漏的 ASCII/全角模型控制令牌；省略仅包含静默令牌的助手行（与 `NO_REPLY` / `no_reply` 完全匹配）；超大行可替换为占位符。
    - `chat.message.get` 是针对单个可见转录条目的增量有界完整消息读取器。传递 `sessionKey`、会话选择限定于 Agent 时可选的 `agentId`，以及此前通过 `chat.history` 公开的转录 `messageId`；当已存储条目仍然可用且未超大时，Gateway 网关会返回相同的显示规范化投影，但不受轻量历史记录截断上限限制。
    - `chat.toolTitles` 返回在 Control UI 中呈现的工具调用简短用途标题（批量处理，最多 24 项，输入有界）。此功能通过 `gateway.controlUi.toolTitles` 选择加入（默认关闭）；已禁用此功能的 Gateway 网关会以 `{ titles: {}, disabled: true }` 响应，且不调用模型，以便客户端停止请求。启用后，标题使用标准实用模型路由：优先使用显式配置的 `utilityModel`（这是一项操作员决策，与所有实用任务一样，可能会将有界任务内容发送给选定的提供商）；否则使用会话提供商声明的默认小型模型，从而不会隐式出现新的出站目的地；空的 `utilityModel` 会将其完全禁用。标题绝不会回退到主模型。结果会按工具名称 + 输入作为键缓存在每 Agent 状态数据库中，因此重复查看绝不会对相同调用重复计费。
    - `chat.send` 接受单轮 `fastMode: "auto"`，对自动截止时间之前启动的模型调用使用快速模式，之后启动的重试、回退、工具结果或继续调用则不使用快速模式。截止时间默认为 60 秒（`DEFAULT_FAST_MODE_AUTO_ON_SECONDS`），并可通过 `agents.defaults.models["<provider>/<model>"].params.fastAutoOnSeconds` 按模型配置。`chat.send` 调用方可以传递单轮 `fastAutoOnSeconds`，为该请求覆盖截止时间。传递 `queueMode`（`steer`、`followup`、`collect` 或 `interrupt`）可仅为此请求覆盖已存储的队列模式；显式 Control UI Steer 操作使用 `queueMode: "steer"`。交互式客户端可以传递 `expectedLeafEntryId`，其中包含其显示的活动转录分支叶节点；也可以传递 `null`，表示权威的空转录。如果另一个客户端先切换了分支，Gateway 网关会以 `details.reason: "active-leaf-changed"` 拒绝发送。

  </Accordion>

  <Accordion title="设备配对和设备令牌">
    - `device.pair.list` 返回待处理和已批准的配对设备。
    - `device.pair.setupCode` 创建移动端设置代码，并默认创建 PNG QR 数据 URL。它要求 `operator.admin`，并且有意不在公布的设备发现信息中提供。结果包括 `setupCode`、可选的 `qrDataUrl`、`gatewayUrl`、非机密的 `auth` 标签以及 `urlSource`。
    - `device.pair.approve`、`device.pair.reject` 和 `device.pair.remove` 管理设备配对记录。
    - `device.pair.rename` 分配一个操作员标签（`{ deviceId, label }`）；该标签的优先级高于客户端报告的显示名称，并且在设备修复或重新批准后仍会保留。
    - `device.token.rotate` 在已配对设备获批的角色和调用方权限范围边界内轮换其令牌。
    - `device.token.revoke` 在已配对设备获批的角色和调用方权限范围边界内撤销其令牌。

    设置代码嵌入了一个短期有效的引导凭据。客户端不得在配对流程之外
    记录或持久化该凭据。

  </Accordion>

  <Accordion title="节点配对、调用和待处理工作">
    - `node.pair.list`、`node.pair.approve`、`node.pair.reject` 和 `node.pair.remove` 涵盖节点能力审批。`node.pair.request` 和 `node.pair.verify` 已于 2026.7 随独立节点配对存储一并移除；待处理请求由 Gateway 网关在节点连接期间创建。
    - `node.list` 和 `node.describe` 返回已知/已连接的节点状态。
    - `node.rename` 更新已配对节点的标签。
    - `node.invoke` 将命令转发到已连接的节点。
    - `node.invoke.result` 返回调用请求的结果。
    - `mcp.tools.call.v1` 是无头节点主机命令，用于调用已配置的节点本地 MCP 工具。它通过 `node.invoke` 传输，要求节点声明该命令，并且仍受配对审批和 `gateway.nodes.commands.deny` 约束。
    - `node.event` 将节点发起的事件带回 Gateway 网关。
    - `node.pluginTools.update` 是替换已连接节点中 Agent 可见的插件/MCP 工具描述符的唯一发布路径；`connect` 参数不携带这些描述符。
    - `node.pending.pull` 和 `node.pending.ack` 是已连接节点的队列 API。
    - `node.pending.enqueue` 和 `node.pending.drain` 管理离线/断开连接节点的持久待处理工作。

  </Accordion>

  <Accordion title="审批类别">
    - `approval.history` 返回按最新优先排列的终态审批记录，这些记录针对 Exec、插件和系统智能体请求保留 30 天（权限范围 `operator.approvals`）。它支持游标分页和可选的种类筛选器；待处理审批不是历史记录行。
    - `approval.get` 和 `approval.resolve` 是与种类无关的持久化审批方法（权限范围 `operator.approvals`）。`approval.get` 返回经过清理的待处理或已保留终态投影，其中包含稳定的 `urlPath`；`approval.resolve` 接受规范审批 ID、显式的 `kind` 和决定，采用首次答复优先的解析方式，并始终返回已记录的规范结果。
    - `exec.approval.request`、`exec.approval.get`、`exec.approval.list` 和 `exec.approval.resolve` 涵盖一次性 Exec 审批请求以及待处理审批的查询/重放。它们是同一持久化审批注册表之上的协议边界适配器。
    - `exec.approval.waitDecision` 等待一个待处理的 Exec 审批并返回最终决定（超时时返回 `null`）。
    - `exec.approvals.get` 和 `exec.approvals.set` 管理 Gateway 网关的 Exec 审批策略快照。
    - `exec.approvals.node.get` 和 `exec.approvals.node.set` 通过节点中继命令管理节点本地的 Exec 审批策略。
    - `plugin.approval.request`、`plugin.approval.list`、`plugin.approval.waitDecision` 和 `plugin.approval.resolve` 涵盖插件定义的审批流程。

  </Accordion>

  <Accordion title="Control UI 命令">
    - `ui.command` 允许 `operator.write` 调用方将类型化布局和导航命令发送给声明支持 `ui-commands` 能力的已连接 Control UI 客户端。
    - 命令涵盖窗格拆分/关闭/聚焦、侧边栏可见性、终端/浏览器面板的可见性和停靠，以及会话导航。
    - 协议 v1 有意将命令扇出到每个已连接且具备相应能力的 Control UI。如果没有此类客户端连接，请求将以 `UNAVAILABLE` 失败，而不是假装布局已更改。

  </Accordion>

  <Accordion title="自动化、Skills 和工具">
    - 自动化：`wake` 安排立即或在下一次 Heartbeat 时注入唤醒文本；`cron.get`、`cron.list`、`cron.status`、`cron.add`、`cron.update`、`cron.remove`、`cron.run`、`cron.runs` 管理定时工作。
    - `cron.run` 仍是用于手动运行的入队式 RPC。需要完成语义的客户端应读取返回的 `runId` 并轮询 `cron.runs`。
    - `cron.runs` 接受可选的非空 `runId` 筛选器，使客户端可以跟踪一次排队的手动运行，而不会与同一任务的其他历史记录发生竞态。
    - Skills 和工具：`commands.list`、`skills.*`、`tools.catalog`、`tools.effective`、`tools.invoke`。请参阅下文的[操作员辅助方法](#operator-helper-methods)。

  </Accordion>
</AccordionGroup>

### 常见事件类别

- `chat`：UI 聊天更新，例如 `chat.inject` 和其他仅限聊天记录的聊天
  事件。在协议 v4 中，增量载荷携带 `deltaText`；`message` 仍是
  累积的助手快照。非前缀替换会设置
  `replace=true`，并使用 `deltaText` 作为替换文本。
- `session.message`、`session.operation`、`session.tool`：已订阅会话的聊天记录、进行中的
  会话操作和事件流更新。
- `session.approval`：面向明确选择加入的精确会话订阅者，提供经过清理的待处理和终态审批事实。
  子级审批使用持久化的祖先受众；事件绝不会修改聊天记录或唤醒智能体。
- `sessions.changed`：会话索引或元数据已更改。
- `presence`：系统在线状态快照更新。
- `tick`：定期保活/存活事件。
- `health`：Gateway 健康快照更新。
- `heartbeat`：Heartbeat 事件流更新。
- `cron`：定时任务运行/任务变更事件。
- `shutdown`：Gateway 网关关闭通知。
- `node.pair.requested` / `node.pair.resolved`：节点配对生命周期。
- `node.invoke.request`：节点调用请求广播。
- `device.pair.requested` / `device.pair.resolved`：已配对设备生命周期。
- `voicewake.changed`：唤醒词触发器配置已更改。
- `config.changed`：配置写入已持久化（载荷携带配置路径、
  新快照哈希和时间戳，但绝不包含配置内容）。仅限操作员读取
  权限范围；客户端通过 `config.get` 刷新。
- `exec.approval.requested` / `exec.approval.resolved`：Exec 审批
  生命周期。
- `plugin.approval.requested` / `plugin.approval.resolved`：插件审批
  生命周期。

### 节点辅助方法

节点可以调用 `skills.bins`，以获取用于自动允许检查的当前技能可执行文件列表。

## 审计账本 RPC

`audit.activity.list` 为操作员客户端提供稳定的最新优先视图，其中包含智能体
运行、工具操作和选择加入的消息生命周期元数据。它要求
`operator.read`。查询会排除超过 30 天的记录，共享
SQLite 账本最多容纳 100,000 条记录。过期行会在
Gateway 网关启动、每小时维护以及后续写入期间删除。有关数据模型和隐私语义，请参阅
[审计历史](/zh-CN/gateway/audit)。

- 参数：可选的精确 `agentId`、`sessionKey` 或 `runId`；可选的 `kind`
  （`"agent_run"`、`"tool_action"` 或 `"message"`）；可选的 `status`
  （`"started"`、`"succeeded"`、`"failed"`、`"cancelled"`、`"timed_out"`、
  `"blocked"` 或 `"unknown"`）；可选的消息 `direction`（`"inbound"` 或
  `"outbound"`）和精确的 `channel`；可选的包含端点的 `after` / `before`
  Unix 毫秒边界；可选的 `limit`，范围为 `1` 到 `500`；以及可选的
  字符串 `cursor`，取自上一页。
- 结果：`{ "events": AuditActivityEventV1[], "nextCursor"?: string }`。

具名 V1 结果联合为智能体运行、工具操作、入站消息
和出站消息分别定义了独立的架构。`eventType` 判别字段分别为
`agent_run`、`tool_action`、`inbound_message` 或 `outbound_message`；`kind` 和
消息 `direction` 仍可用于筛选和显示。每个事件都有
整数 `schemaVersion: 1`。消息身份引用使用精确的
`hmac-sha256:v1:<32 hex key id>:<64 hex digest>` 格式；渠道发送者的参与者
ID 使用相同格式。

所有变体都要求 `eventType`、`schemaVersion`、`eventId`、`sequence`、
`sourceSequence`、`occurredAt`、`kind`、`action`、`status`、`actor` 和
`redaction`。变体字段如下：

| `eventType`        | 必填字段                                                   | 可选字段                                                                                                                 |
| ------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `agent_run`        | `agentId`、`runId`；`kind: "agent_run"`                           | `sessionKey`、`sessionId`、`errorCode`                                                                                          |
| `tool_action`      | `agentId`、`runId`；`kind: "tool_action"`                         | `sessionKey`、`sessionId`、`toolCallId`、`toolName`、`errorCode`                                                                |
| `inbound_message`  | `direction: "inbound"`、`channel`、`conversationKind`、`outcome`  | `agentId`、`runId`、`durationMs`、`resultCount`、身份引用、`reasonCode`、`errorCode`                                 |
| `outbound_message` | `direction: "outbound"`、`channel`、`conversationKind`、`outcome` | `agentId`、`runId`、`durationMs`、`resultCount`、身份引用、`reasonCode`、`deliveryKind`、`failureStage`、`errorCode` |

封闭的消息枚举如下：

- `conversationKind`：`direct`、`group`、`channel` 或 `unknown`。
- 入站 `outcome`：`completed`、`skipped` 或 `failed`；可选的
  `reasonCode`：`duplicate`、`reply_operation_active`、
  `reply_operation_aborted`、`fast_abort`、`plugin_bound_handled`、
  `plugin_bound_unavailable`、`plugin_bound_declined`、`plugin_bound_error`、
  `before_dispatch_handled`、`acp_dispatch_completed`、`acp_dispatch_failed`、
  `acp_dispatch_empty` 或 `acp_dispatch_aborted`。
- 出站 `outcome`：`sent`、`suppressed`、`failed` 或 `unknown`；可选的
  `reasonCode`：`cancelled_by_message_sending_hook`、
  `cancelled_by_reply_payload_sending_hook`、
  `empty_after_message_sending_hook`、`empty_after_reply_payload_sending_hook`
  或 `no_visible_payload`。如果适配器未返回平台身份，则为
  `unknown`，因为无法证明外部副作用未发生。
- `deliveryKind`：`text`、`media` 或 `other`；`failureStage`：
  `platform_send`、`queue` 或 `unknown`。

终态字段相互关联，而非各自独立可选：

| 变体          | 终态映射                                                                                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 智能体运行        | `started` 没有 `errorCode`；每个非成功的完成状态都要求与其对应的 `run_*` 代码。                                                                 |
| 工具操作      | `started` 和成功状态没有 `errorCode`；每个其他完成状态都要求与其对应的 `tool_*` 代码。                                                       |
| 入站消息  | 成功 = `completed`；已阻止 = `skipped`；失败 = `failed` 加 `message_processing_failed`。`reasonCode` 如存在，则必须属于该终态类别。 |
| 出站消息 | 成功 = `sent`；已阻止 = `suppressed` 加 `reasonCode`；失败 = `failed` 加 `errorCode` 和 `failureStage`；未知 = `unknown` 加 `failureStage`。      |

每个活动事件都包含稳定的事件 ID、单调递增的账本序列、
源事件序列、时间戳、参与者、操作、状态、整数
`schemaVersion: 1` 和 `redaction: "metadata_only"`。运行和工具记录
要求提供智能体和运行来源，并且可以包含会话来源。消息
记录可以包含智能体和运行 ID，但有意绝不包含
`sessionKey` 或 `sessionId`；因此，`sessionKey` 查询筛选器仅适用于
运行和工具行。工具事件可以包含工具调用 ID 和工具名称。

消息记录使用 `message.inbound.processed` 或
`message.outbound.finished`，并添加方向、渠道、会话类型、
规范化结果，以及可选的交付类型、失败阶段、持续时间、
结果数量、原因代码和安装本地加密键生成的
账户/会话/消息/目标伪标识符。这些伪标识符有助于
关联，但并非匿名化：状态数据库包含其密钥，
而 RPC 和 CLI 导出不包含。账本不存储提示词、消息
正文、工具参数、工具结果、命令输出或原始错误文本。
运行/工具的 `sessionKey` 值仍是原始关联元数据，其中可能嵌入
平台账户或对端 ID；消息记录不包含会话键。

对于入站行，`durationMs` 测量核心分派到其终止状态的耗时，
`resultCount` 统计最终确定的排队工具、分块和回复负载。对于
出站行，`durationMs` 涵盖从取得交付所有权到确认、
死信或协调完成的时间（包括排队等待时间），而 `resultCount`
统计已识别的物理平台发送次数。`deliveryKind`（如果存在）
描述经过钩子和渲染后的有效负载；被抑制或
崩溃状态不明确的行不包含该字段。

当前消息覆盖范围包括到达核心
分派的已接受入站消息，包括核心重复/终止结果。对于到达共享持久
交付的每个原始逻辑回复负载，出站覆盖范围会写入
一条终止行；分块和适配器扇出汇总在 `resultCount` 中。排队的
可重试或状态不明确的发送仅在确认、进入死信
或协调后记录。绕过这些
共享边界的插件本地和直接发送路径目前尚未覆盖。有界工作队列采用尽力而为方式，
在发生故障或饱和时可能丢弃记录，因此此表面并非
无损的合规性归档。

记录功能默认启用，并由
[`audit.enabled`](/zh-CN/gateway/configuration-reference#audit) 控制。消息记录
由 `audit.messages` 单独控制，默认值为 `"off"`。禁用
记录后，`audit.activity.list` 仍会提供此前写入的记录，
直到这些记录过期。

已发布的 `audit.list` 请求、结果和 `AuditEvent` 架构保持
不变，并且仅返回智能体运行和工具操作记录。新的操作员
客户端应在 Gateway 网关公布 `audit.activity.list` 时调用它。旧版
Gateway 网关可能报告 `unknown method: audit.activity.list`，也可能因为
已发布版本先执行授权、后查找方法，而对只读权限范围的请求报告 `missing scope:
operator.admin`。仅当该方法未被公布时，才将后者视为方法不存在。随后，只有当过滤器不需要消息类型、方向或渠道
支持时，客户端才能重试 `audit.list`。

使用 [`openclaw audit`](/zh-CN/cli/audit) 执行文本查询和有界 JSON 导出。

## 任务账本 RPC

操作员客户端通过
任务账本 RPC（`packages/gateway-protocol/src/schema/tasks.ts`）检查和取消 Gateway 网关后台任务记录。这些
RPC 返回经过清理的任务摘要，而非原始运行时状态。

- `tasks.list` 需要 `operator.read`。
  - 参数：可选的 `status`（`"queued"`、`"running"`、`"completed"`、
    `"failed"`、`"cancelled"` 或 `"timed_out"`）或由这些状态组成的数组，
    可选的 `agentId`、可选的 `sessionKey`、从 `1` 到
    `500` 的可选 `limit`，以及可选字符串 `cursor`。
  - 结果：`{ "tasks": TaskSummary[], "nextCursor"?: string }`。
- `tasks.get` 需要 `operator.read`。
  - 参数：`{ "taskId": string }`。
  - 结果：`{ "task": TaskSummary }`。
  - 任务 ID 不存在时，返回 Gateway 网关的未找到错误结构。
- `tasks.cancel` 需要 `operator.write`。
  - 参数：`{ "taskId": string, "reason"?: string }`。
  - 结果：`{ "found": boolean, "cancelled": boolean, "reason"?: string, "task"?: TaskSummary }`。
  - `found` 表示账本中是否存在匹配的任务。`cancelled`
    表示运行时是否接受或记录了取消操作。

`TaskSummary` 包含 `id`、`status` 以及可选元数据：`kind`、
`runtime`、`title`、`agentId`、`sessionKey`、`childSessionKey`、`ownerKey`、
`runId`、`taskId`、`flowId`、`parentTaskId`、`sourceId`、时间戳、进度、
终止摘要和经过清理的错误文本。`agentId` 标识执行任务的智能体；
`sessionKey` 和 `ownerKey` 保留请求者和控制
上下文。

## 操作员辅助方法

- `commands.list`（`operator.read`）获取智能体的运行时命令清单。
  - `agentId` 是可选的；省略它可读取默认的 Agent 工作区。
  - `scope` 控制主要 `name` 所针对的表面：`text` 返回
    不带前导 `/` 的主要文本命令令牌；`native` 和默认
    `both` 路径在可用时返回可感知提供商的原生命令名称。
  - `textAliases` 包含 `/model` 和 `/m` 等精确的斜杠别名。
  - `nativeName` 在存在时包含可感知提供商的原生命令名称。
  - `provider` 是可选的，仅影响原生命名和原生插件
    命令的可用性。
  - `includeArgs=false` 从响应中省略序列化参数元数据。
- `tools.catalog`（`operator.read`）获取智能体的运行时工具目录。
  响应包含分组工具和来源元数据：
  - `source`：`core` 或 `plugin`
  - `pluginId`：当值为 `source="plugin"` 时表示插件所有者
  - `optional`：插件工具是否为可选工具
- `tools.effective`（`operator.read`）获取会话的运行时有效工具
  清单。
  - `sessionKey` 为必填项。
  - Gateway 网关从服务器端会话中派生可信运行时上下文，
    而不接受调用方提供的身份验证或交付上下文。
  - 响应是服务器派生的、限定于会话范围的活跃
    清单投影，包括核心、插件、渠道以及已发现的 MCP
    服务器工具。
  - `tools.effective` 对 MCP 是只读的：它可以通过最终工具策略投影热会话的 MCP
    目录，但不会创建 MCP 运行时、
    连接传输协议或发出 `tools/list`。如果不存在匹配的热目录，
    响应可能包含 `mcp-not-yet-connected`、
    `mcp-not-yet-listed` 或 `mcp-stale-catalog` 等通知。
  - 有效工具条目使用 `source="core"`、`source="plugin"`、
    `source="channel"` 或 `source="mcp"`。
- `tools.invoke`（`operator.write`）通过与 `/tools/invoke`
  相同的 Gateway 网关策略路径调用一个可用工具。
  - `name` 为必填项。`args`、`sessionKey`、`agentId`、`confirm` 和
    `idempotencyKey` 为可选项。
  - 如果 `sessionKey` 和 `agentId` 同时存在，解析出的会话智能体
    必须与 `agentId` 匹配。
  - 仅限所有者使用的核心包装器（如 `cron`、`gateway` 和 `nodes`）需要
    所有者/管理员身份（`operator.admin`），即使 `tools.invoke` 本身
    是 `operator.write`。
  - 响应是面向 SDK 的封装，包含 `ok`、`toolName`、可选的
    `output` 和有类型的 `error` 字段。审批或策略拒绝会在
    负载中返回 `ok:false`，而不会绕过 Gateway 网关工具策略
    流水线。
- `skills.status`（`operator.read`）获取智能体可见的技能清单。
  - `agentId` 是可选的；省略它可读取默认的 Agent 工作区。
  - 响应包含资格信息、缺失的要求、配置检查
    和经过清理的安装选项，且不会暴露原始密钥值。
- `skills.search` 和 `skills.detail`（`operator.read`）返回 ClawHub
  发现元数据。
- `skills.upload.begin`、`skills.upload.chunk` 和 `skills.upload.commit`
  （`operator.admin`）会在安装私有技能归档前暂存该归档。这是供可信客户端使用的独立管理员上传路径，
  不是常规 ClawHub 技能安装流程，并且默认禁用，除非启用
  `skills.install.allowUploadedArchives`。
  - `skills.upload.begin({ kind: "skill-archive", slug, sizeBytes, sha256?, force?, idempotencyKey? })`
    创建绑定到该 slug 和 force 值的上传。
  - `skills.upload.chunk({ uploadId, offset, dataBase64 })` 在
    精确的解码后偏移量处追加字节。
  - `skills.upload.commit({ uploadId, sha256? })` 验证最终大小和
    SHA-256。提交只会完成上传；不会安装技能。
  - 上传的技能归档是包含 `SKILL.md` 根目录的 zip 归档。归档的内部目录名称绝不会选择安装目标。
- `skills.install`（`operator.admin`）有三种模式：
  - ClawHub 模式：`{ source: "clawhub", slug, version?, force? }` 将
    技能文件夹安装到默认 Agent 工作区的 `skills/` 目录中。
  - 上传模式：`{ source: "upload", uploadId, slug, force?, sha256?, timeoutMs? }`
    将已提交的上传内容安装到默认 Agent 工作区的
    `skills/<slug>` 目录中。slug 和 force 值必须与
    原始 `skills.upload.begin` 请求匹配。除非启用
    `skills.install.allowUploadedArchives`，否则请求会被拒绝；该设置不
    影响 ClawHub 安装。
  - Gateway 网关安装程序模式：`{ name, installId, timeoutMs? }` 在 Gateway 网关主机上运行声明的
    `metadata.openclaw.install` 操作。旧版客户端可能
    仍会发送 `dangerouslyForceUnsafeInstall`；此字段已弃用，
    仅为协议兼容性而接受，并会被忽略。对于
    操作员负责的安装决策，请使用 `security.installPolicy`。
- `skills.update`（`operator.admin`）有两种模式：
  - ClawHub 模式更新默认 Agent 工作区中的一个已跟踪 slug 或所有已跟踪的 ClawHub 安装。
  - 配置模式修补 `skills.entries.<skillKey>` 值，例如 `enabled`、
    `apiKey` 和 `env`。

### `models.list` 视图

`models.list` 接受可选的 `view` 参数
（`src/agents/model-catalog-visibility.ts`）：

- 省略或设为 `"default"`：如果已配置 `agents.defaults.modelPolicy.allow`，则
  响应为允许的目录，包括为 `provider/*` 条目动态发现的模型。否则，响应为完整的 Gateway 网关
  目录。
- `"configured"`：适合选择器规模的行为。如果已配置 `agents.defaults.modelPolicy.allow`，
  它仍然优先，包括为 `provider/*` 条目执行限定于提供商范围的发现。没有允许列表时，响应使用显式
  `models.providers.<provider>.models` 条目；仅当不存在已配置的模型行时，才回退到完整
  目录。
- `"provider-config"`：由来源定义的 `models.providers.*.models` 清单，
  独立于选择器允许列表。行中包含公开的模型能力和
  可感知路由的可用性，但不包含提供商端点、身份验证材料和
  运行时请求配置。
- `"all"`：完整的 Gateway 网关目录，绕过 `agents.defaults.modelPolicy.allow`。用于
  诊断/发现 UI，而不是常规模型选择器。

## Exec 审批

- 当 Exec 请求需要审批时，Gateway 网关会广播
  `exec.approval.requested`。
- 操作员客户端通过调用 `exec.approval.resolve` 来处理（需要
  `operator.approvals`）。
- 对于 `host=node`，`exec.approval.request` 必须包含 `systemRunPlan`
  （规范的 `argv`/`cwd`/`rawCommand`/会话元数据）。缺少
  `systemRunPlan` 的请求会被拒绝。
- 审批后，转发的 `node.invoke system.run` 调用会复用该规范
  `systemRunPlan`，将其作为权威的命令/cwd/会话上下文。
- 如果调用方在准备阶段与最终获批的 `system.run` 转发之间修改了
  `command`、`rawCommand`、`cwd`、`agentId` 或
  `sessionKey`，Gateway 网关会拒绝运行，而不是信任修改后的载荷。

## Agent 投递回退

- `agent` 请求可以包含 `deliver=true`，以请求出站投递。
- `bestEffortDeliver=false`（默认值）保持严格行为：无法解析或
  仅限内部的投递目标会返回 `INVALID_REQUEST`。
- `bestEffortDeliver=true` 允许在无法解析出任何外部可投递路由时，
  回退为仅在会话中执行（例如内部/webchat
  会话或存在歧义的多渠道配置）。
- 请求投递时，最终的 `agent` 结果可能包含 `result.deliveryStatus`，
  使用与 [`openclaw agent --json --deliver`](/zh-CN/cli/agent#json-delivery-status)
  中记录的相同 `sent`、`suppressed`、`partial_failed` 和
  `failed` 状态。

## 版本控制

- `PROTOCOL_VERSION`、`MIN_CLIENT_PROTOCOL_VERSION`、
  `MIN_NODE_PROTOCOL_VERSION` 和 `MIN_PROBE_PROTOCOL_VERSION` 位于
  `packages/gateway-protocol/src/version.ts` 中。
- 客户端发送 `minProtocol` + `maxProtocol`。操作员和 UI 客户端必须
  在该范围内包含当前协议；当前客户端和服务器运行
  v4 协议。
- 同时具有 `role: "node"` 和 `client.mode: "node"`
  的已认证客户端可以使用 N-1 节点协议（当前为 v3）。轻量级重启探测使用
  相同的 N-1 窗口。设备身份验证、配对、权限范围、命令策略和 Exec
  审批不受此兼容性窗口影响。插件所有的节点
  能力和命令会被禁用，直到节点升级到当前
  协议，因为其托管接口不属于 N-1 合约的一部分。
- 架构和模型由 TypeBox 定义生成：
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

### 客户端常量

参考客户端实现位于 `packages/gateway-client/src/`
（OpenClaw 通过轻量级 `src/gateway/client.ts` 门面封装它）。这些
默认值在 v4 协议中保持稳定，是第三方客户端的预期基线。

| 常量                                      | 默认值                                                | 来源                                                                                                                      |
| ----------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `PROTOCOL_VERSION`                        | `4`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_CLIENT_PROTOCOL_VERSION`             | `4`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_NODE_PROTOCOL_VERSION`               | `3`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_PROBE_PROTOCOL_VERSION`              | `3`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| 请求超时（每个 RPC）                      | `30_000` ms                                           | `packages/gateway-client/src/client.ts`（`requestTimeoutMs`）                                                              |
| 预认证/连接质询超时                       | `15_000` ms                                           | `packages/gateway-client/src/timeouts.ts`（`OPENCLAW_HANDSHAKE_TIMEOUT_MS` 环境变量可提高已配对服务器/客户端的时间预算） |
| 初始重连退避                              | `1_000` ms                                            | `packages/gateway-client/src/client.ts`（`GATEWAY_RECONNECT_POLICY`）                                                      |
| 最大重连退避                              | `30_000` ms                                           | `packages/gateway-client/src/client.ts`（`GATEWAY_RECONNECT_POLICY`）                                                      |
| 设备令牌关闭后的快速重试限制              | `250` ms                                              | `packages/gateway-client/src/client.ts`                                                                                   |
| `terminate()` 前的强制停止宽限期     | `250` ms                                              | `FORCE_STOP_TERMINATE_GRACE_MS`                                                                                           |
| `stopAndWait()` 默认超时               | `1_000` ms                                            | `STOP_AND_WAIT_TIMEOUT_MS`                                                                                                |
| 默认 tick 间隔（`hello-ok` 前）   | `30_000` ms                                           | `packages/gateway-client/src/client.ts`                                                                                   |
| tick 超时关闭                             | 静默时间超过 `tickIntervalMs * 2` 时使用代码 `4000` | `packages/gateway-client/src/client.ts`                                                                                   |
| `MAX_PAYLOAD_BYTES`                       | `25 * 1024 * 1024`（25 MB）                            | `src/gateway/server-constants.ts`                                                                                         |

服务器在 `hello-ok` 中公布实际生效的 `policy.tickIntervalMs`、
`policy.maxPayload` 和 `policy.maxBufferedBytes`；客户端
应遵循这些值，而不是握手前的默认值。

当每个待处理请求都有截止时间时，参考客户端允许有限时请求使用各自配置的截止时间。
没有有限 `timeoutMs` 的 `expectFinal` 请求、
任何带有 `timeoutMs: null` 的请求，或有限时请求与
无界请求的混合，都会使 tick 看门狗保持活动状态。如果入站事件和
响应持续静默并超过 tick 超时阈值，客户端会使用代码 `4000` 关闭
套接字、拒绝所有待处理请求并重新连接。重新连接后，它不会
重放被拒绝的请求。

## 身份验证

- 共享密钥 Gateway 网关身份验证使用 `connect.params.auth.token` 或
  `connect.params.auth.password`，具体取决于所配置的
  `gateway.auth.mode`（`"none" | "token" | "password" | "trusted-proxy"`）。
- Tailscale Serve（`gateway.auth.allowTailscale: true`）等携带身份信息的模式
  或非环回 `gateway.auth.mode: "trusted-proxy"` 会根据请求标头通过连接
  身份验证检查，而不是使用 `connect.params.auth.*`。
- 私有入口 `gateway.auth.mode: "none"` 会完全跳过共享密钥连接身份验证；
  请勿将该模式暴露在公共或不受信任的入口上。
- 配对后，Gateway 网关会签发一个作用域限定为连接
  角色 + 权限范围的设备令牌，并在 `hello-ok.auth.deviceToken` 中返回。客户端应在
  每次成功连接后持久保存该令牌。
- 使用已存储的设备令牌重新连接时，也应复用该令牌已获批准并存储的
  权限范围集。这样可以保留已授予的读取、探测和状态访问权限，
  并避免重新连接时无提示地收窄为隐式的仅管理员
  权限范围。
- 客户端连接身份验证组装（`packages/gateway-client/src/client.ts` 中的
  `selectConnectAuth`）：
  - `auth.password` 与其他身份验证方式相互独立，设置后始终会被转发。
  - `auth.token` 按以下优先顺序填充：首先是显式共享令牌，
    然后是显式 `deviceToken`，最后是已存储的单设备令牌（以
    `deviceId` + `role` 为键）。
  - 仅当上述方式均未解析出
    `auth.token` 时，才会发送 `auth.bootstrapToken`。共享令牌或任何已解析的设备令牌都会抑制它。
  - 在一次性
    `AUTH_TOKEN_MISMATCH` 重试时自动提升已存储设备令牌，仅限受信任端点：环回，
    或使用固定 `tlsFingerprint` 的 `wss://`。未固定指纹的公共 `wss://`
    不符合条件。
- 内置设置代码引导会返回主节点
  `hello-ok.auth.deviceToken`，以及 `hello-ok.auth.deviceTokens` 中一个权限受限的操作员令牌，
  供受信任的移动端交接使用。该操作员令牌包含用于原生 Talk 配置读取的
  `operator.talk.secrets`，但不包含配对变更权限范围和 `operator.admin`。
- 当非基线设置代码引导等待批准时，
  `PAIRING_REQUIRED` 详细信息包含 `recommendedNextStep: "wait_then_retry"`、
  `retryable: true` 和 `pauseReconnect: false`。继续使用同一
  引导令牌重新连接，直到请求获得批准或令牌失效。
- 仅当连接通过 `wss://` 或环回/本地配对等受信任传输
  使用引导身份验证时，才持久保存 `hello-ok.auth.deviceTokens`。
- 如果客户端提供显式 `deviceToken` 或显式 `scopes`，则
  调用方请求的权限范围集仍具有决定权；仅当客户端复用已存储的单设备令牌时，
  才会复用缓存的权限范围。
- 可以通过 `device.token.rotate` 和
  `device.token.revoke` 轮换或撤销设备令牌（需要 `operator.pairing`）。轮换或撤销
  节点或其他非操作员角色还需要 `operator.admin`。
- `device.token.rotate` 返回轮换元数据。仅当同一设备的调用已经使用该
  设备令牌完成身份验证时，它才会回传替换后的不记名令牌，以便仅使用令牌的客户端
  在重新连接前持久保存替换令牌。共享/管理员轮换不会回传不记名令牌。
- 令牌签发、轮换和撤销始终受限于该设备配对条目中记录的已批准角色
  集；令牌变更无法扩展到或指定配对批准从未授予的设备角色。
- 对于已配对设备的令牌会话，设备管理仅限自身，除非
  调用方还具有 `operator.admin`：非管理员调用方只能管理
  自己设备条目的操作员令牌。节点和其他非操作员令牌
  仅限管理员管理，即使是调用方自己的设备也不例外。
- `device.token.rotate` 和 `device.token.revoke` 还会根据调用方当前会话的权限范围，
  检查目标操作员令牌的权限范围集。
  非管理员调用方无法轮换或撤销权限范围比其现有权限更广的操作员令牌。
- 身份验证失败包括 `error.details.code` 以及恢复提示：
  - `error.details.canRetryWithDeviceToken`（布尔值）
  - `error.details.recommendedNextStep`：以下之一：`retry_with_device_token`、
    `update_auth_configuration`、`update_auth_credentials`、
    `wait_then_retry`、`review_auth_configuration`
    （`packages/gateway-protocol/src/connect-error-details.ts`）。
- 针对 `AUTH_TOKEN_MISMATCH` 的客户端行为：
  - 受信任客户端可以尝试一次有界重试，并使用缓存的单设备
    令牌。
  - 如果该重试失败，请停止自动重新连接循环，并显示操作员
    操作指引。
- `AUTH_SCOPE_MISMATCH` 表示设备令牌已被识别，但未涵盖
  所请求的角色/权限范围。不要将其显示为令牌错误；应提示
  操作员重新配对，或批准更窄/更宽的权限范围约定。

## 设备身份与配对

- 节点应包含由密钥对指纹派生的稳定设备身份（`device.id`）。
- Gateway 网关按设备 + 角色签发令牌。
- 除非启用了本地自动批准，否则新的设备 ID 需要获得
  配对批准。
- 配对自动批准以直接 local loopback 连接为核心。
- OpenClaw 还为受信任的共享密钥辅助流程提供一条范围狭窄的
  后端/容器本地自连接路径。
- 同一主机上的 tailnet 或 LAN 连接在配对时仍视为远程连接，
  并且需要批准。
- WS 客户端通常会在 `connect` 期间包含 `device` 身份（操作员 +
  节点）。唯一不带设备身份的操作员例外是以下显式信任路径：
  - 成功通过 `gateway.auth.mode: "trusted-proxy"` 完成操作员 Control UI 身份验证。
  - 保留内部辅助路径上通过直接环回 `gateway-client` 发起的后端 RPC。
- 省略设备身份会影响权限范围。当不带设备身份的
  操作员连接通过显式信任路径被允许时，除非该路径具有
  指定的权限范围保留例外，否则 OpenClaw 仍会将自行声明的权限范围清空。
  此后，受权限范围限制的方法会以 `missing scope` 失败。
- 保留的直接环回 `gateway-client` 后端辅助路径仅为内部本地控制平面 RPC
  保留权限范围；自定义后端 ID
  不享有此例外。
- 所有连接都必须签署服务器提供的 `connect.challenge` nonce。

### 设备身份验证迁移诊断

对于仍使用质询前签名行为的旧版客户端，`connect`
会在 `error.details.code` 下返回 `DEVICE_AUTH_*` 详细代码，并带有稳定的
`error.details.reason`。

常见迁移失败：

| 消息                     | details.code                     | details.reason           | 含义                                            |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | 客户端省略了 `device.nonce`（或发送了空值）。     |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | 客户端使用过期/错误的 nonce 进行了签名。            |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | 签名载荷与 v2 载荷不匹配。       |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | 签名时间戳超出允许的偏差范围。          |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` 与公钥指纹不匹配。 |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | 公钥格式处理/规范化失败。         |

迁移目标：

- 始终等待 `connect.challenge`。
- 签署包含服务器 nonce 的 v2 载荷。
- 在 `connect.params.device.nonce` 中发送同一 nonce。
- 首选签名载荷为 `v3`
  （`packages/gateway-client/src/device-auth.ts` 中的 `buildDeviceAuthPayloadV3`），
  除设备/客户端/角色/权限范围/令牌/nonce 字段外，它还绑定
  `platform` 和 `deviceFamily`。
- 为保持兼容性，仍接受旧版 `v2` 签名，但已配对设备
  元数据固定仍会控制重新连接时的命令策略。

## TLS 与固定

- WS 连接支持 TLS（`gateway.tls` 配置）。
- 客户端可以选择通过
  `gateway.remote.tlsFingerprint` 或 CLI `--tls-fingerprint` 固定 Gateway 网关证书指纹。

## 范围

此协议公开完整的 Gateway 网关 API：状态、渠道、模型、聊天、
智能体、会话、节点、审批等。确切的接口范围由从
`packages/gateway-protocol/src/schema.ts` 重新导出的 TypeBox 模式定义。

## 相关内容

- [构建 Gateway 客户端](https://docs.openclaw.ai/gateway/clients)
- [嵌入 OpenClaw](https://docs.openclaw.ai/gateway/embedding)
- [Bridge protocol](/zh-CN/gateway/bridge-protocol)
- [Gateway 运行手册](/zh-CN/gateway)
