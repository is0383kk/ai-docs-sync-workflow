---
read_when:
    - 在 OpenClaw 仓库之外构建操作员工具、仪表板或 WebChat 客户端
    - 实现 Gateway 网关重连、历史记录、审批或设备配对
    - 为新的 Gateway 网关线路版本更新第三方客户端
summary: 为 Gateway 网关 WebSocket 协议构建第三方操作员或 WebChat 客户端
title: 构建 Gateway 客户端
x-i18n:
    generated_at: "2026-07-26T06:15:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fa24b196ff1fa28fb3b64d49ac25597f22cf1945aea56029e78e4375f1bdddb7
    source_path: gateway/clients.md
    workflow: 16
---

使用已发布的 Gateway 网关软件包构建操作员仪表板、WebChat 客户端
及其他第三方应用。本指南介绍围绕线协议的客户端生命周期：
身份验证、能力、重连恢复、历史记录、订阅和版本升级。

有关帧结构、握手、错误和完整方法列表，请阅读
[Gateway 网关协议规范](https://docs.openclaw.ai/gateway/protocol)。

## 安装软件包

```bash
npm install @openclaw/gateway-client @openclaw/gateway-protocol
```

<Note>
这些软件包随 OpenClaw 发布系列一同提供。在初始发布期间，首次包含这些软件包的
OpenClaw 版本发布前，npm 可能返回 `E404`；
请仅在下方注册表页面可访问后安装。
</Note>

- [`@openclaw/gateway-protocol`](https://www.npmjs.com/package/@openclaw/gateway-protocol)
  提供 schema、运行时验证器、TypeScript 类型、客户端身份和
  能力注册表、结构化错误读取器以及协议版本常量。
  其 npm tarball 还包含生成的
  [`protocol.schema.json`](https://unpkg.com/@openclaw/gateway-protocol/protocol.schema.json)
  机器可读协议。
- [`@openclaw/gateway-client`](https://www.npmjs.com/package/@openclaw/gateway-client)
  是参考连接实现。导入软件包根入口以使用 Node
  客户端，并导入 `@openclaw/gateway-client/browser` 以使用浏览器安全的协议、
  设备身份验证和重连辅助程序。

Node 入口自行管理 WebSocket 传输。浏览器宿主需提供 WebSocket
适配器，以及用于设备身份和设备令牌的持久化存储与签名回调。

## 选择权限范围并配对设备

同时呈现审批提示的完整交互式聊天客户端应请求
`role: "operator"`，并包含以下权限范围：

| 权限范围                | 用途                                                                                |
| -------------------- | ----------------------------------------------------------------------------------------- |
| `operator.read`      | `chat.history`、`sessions.list`、`sessions.subscribe`、模型状态和只读事件 |
| `operator.write`     | `chat.send` 和普通会话变更                                                |
| `operator.approvals` | 列出、显示和处理 Exec 或插件审批                               |

仅当客户端处理交互式问题时添加 `operator.questions`；
仅当客户端管理已配对设备或节点时添加 `operator.pairing`；
仅针对 `config.patch` 等管理操作添加 `operator.admin`。
[操作员权限范围参考](https://docs.openclaw.ai/gateway/operator-scopes)
定义了完整的方法和审批时规则。

不要通过手动编辑 `openclaw.json` 来创建每客户端 bearer 令牌。使用 `openclaw configure --section
gateway` 或 `openclaw onboard --gateway-auth ...` 选项配置
Gateway 网关的共享引导身份验证，然后让设备
配对签发客户端令牌：

1. 在客户端中持久化 Ed25519 设备身份。
2. 等待 `connect.challenge`，签署与质询绑定的设备负载，然后发送
   `connect`，其中包含请求的操作员角色、权限范围，以及用于引导身份验证的共享 Gateway 网关令牌
   或密码。
3. 如果 Gateway 网关返回结构化的 `PAIRING_REQUIRED` 详细信息，请显示请求
   ID，并根据 `error.details.recommendedNextStep` 暂停或重试。
4. 在 Gateway 网关主机上，使用 `openclaw devices list` 审查请求，然后
   使用 `openclaw devices approve <requestId>` 批准该确切的当前请求。
5. 重新连接，并将 `hello-ok.auth.deviceToken` 与协商后的角色和
   权限范围一起持久化。后续连接使用该设备令牌。

升级权限范围或角色会创建新的待处理配对请求。令牌轮换无法
扩大已批准的配对协议。有关批准、轮换和
撤销命令，请参阅[设备 CLI](https://docs.openclaw.ai/cli/devices)。

## 声明客户端能力

`connect.params.caps` 描述客户端可以使用的可选行为。它
不授予授权。应从 `GATEWAY_CLIENT_CAPS` 导入名称，而不要
重复字符串字面量：

```ts
import { GATEWAY_CLIENT_CAPS } from "@openclaw/gateway-protocol/client-info";

const caps = [GATEWAY_CLIENT_CAPS.TOOL_EVENTS];
```

当前注册表包含 `approvals`、`exec-approvals`、`inline-widgets`、
`run-tool-bindings`、`session-scoped-events`、`plugin-approvals`、
`task-suggestions`、`terminal-offset-seq`、`tool-events` 和 `ui-commands`。
仅声明客户端实际实现的能力。

<Warning>
`tool-events` 控制实时工具执行流式传输。Gateway 网关仅将
声明此能力的连接注册为某次运行的结构化工具事件接收者。
如果未声明此能力，该连接不会收到实时工具事件，且
握手不会报告错误。
</Warning>

受能力限制的智能体工具是同一声明的另一种用途。如果某个
智能体工具需要客户端能力，除非发起请求的客户端声明了所有必需能力，
否则 Gateway 网关会省略该工具。

## 重连后恢复状态

将每次成功重连视为基于持久历史记录和
当前内存中运行状态的新投影：

1. 重新建立 `sessions.subscribe` 和所选会话的
   `sessions.messages.subscribe` 订阅。
2. 针对所选 `sessionKey` 调用 `chat.history`，并用返回的
   `messages` 投影替换本地持久化行。
3. 如果存在 `inFlightRun`，采用其 `runId`、缓冲的 `text` 和可选的
   `plan`。即使 `text` 为空，也要采用该运行。
4. 读取 `sessionInfo.hasActiveRun` 和 `sessionInfo.activeRunIds`。判断保留的运行是否仍拥有
   流式 UI 时，优先采用 `activeRunIds` 中的精确成员关系。
   `hasActiveRun` 为 true 但未列出 ID 时，可能表示另一个
   活跃的运行时投影。
5. 按 `payload.runId` 和 `payload.seq` 协调后续 `agent` 事件。
   为每次运行分别维护已接受的最高序列，忽略
   已见过或更低的序列，并将向前的序列缺口视为重新加载
   权威历史记录的理由。

外层事件帧还有可选的 `seq`，用于对
当前 WebSocket 连接上的事件排序。建立新连接后它会重置。
`agent` 事件负载中的 `seq` 按运行分配，用于对该运行的生命周期、
助手、计划、工具和其他流式事件排序。

## 使用历史元数据和稳定锚点

`chat.history` 返回的行可以携带 `__openclaw` 元数据封装：

- `id` 是转录条目的身份标识。将其用于锚定历史记录请求，
  但不要将其用作唯一的显示行键。
- `seq` 是正数的转录记录序列。一个存储记录可以投影
  为多个显示行，因此应将具有相同 `id` 和序列的同级行
  保持在一起。
- `kind` 标识合成行。压缩边界使用
  `kind: "compaction"`；如果匹配的检查点记录了相关指标，还可能包含 `tokensBefore` 和 `tokensAfter`。

使用响应中的 `hasMore` 和 `nextOffset` 值向后翻页。数字
偏移量描述当前转录投影，因此不要将其作为
跨重置或压缩的长期书签持久化。应改为持久化 `__openclaw.id`。
要恢复到已知行附近，请使用 `messageId` 和返回该值的
`sessionId` 调用 `chat.history`。Gateway 网关可以从重置
归档历史记录中解析该锚点；锚定响应会有意省略数字分页元数据。

## 订阅而非轮询用量

使用 `sessions.list` 加载初始目录，然后每个连接调用一次
`sessions.subscribe`。按 `sessionKey` 合并 `sessions.changed` 事件。会话变更
负载可以携带实时 `inputTokens`、`outputTokens`、`totalTokens`、
`totalTokensFresh`、`contextTokens`、`estimatedCostUsd`、响应用量设置
和活跃运行状态。

某些变更通知只是失效信号。如果事件省略了视图所需的
行字段，请刷新 `sessions.list`。不要通过轮询 `usage.cost` 或
`sessions.usage` 来保持实时会话列表为最新状态；这些方法仅用于
按需聚合报告或详细报告。

## 回填 Exec 审批

具有 `operator.approvals` 的客户端应在
`hello-ok` 完成后立即安装事件监听器，然后调用 `exec.approval.list`
回填连接建立前的请求。按审批 ID 协调列表与实时
`exec.approval.requested` / `exec.approval.resolved` 事件，确保与列表请求并发的
状态转换既不会丢失，也不会被重新恢复。

## 跟踪协议版本

当前线协议版本为 `4`。通用操作员和 WebChat 客户端必须
使用 `minProtocol: 4` 和 `maxProtocol: 4` 协商完全匹配的当前版本。
只有经过身份验证的节点客户端和轻量级探针具有 N-1 接受
窗口，目前支持协议 `3` 到 `4`。

协议变更优先采用增量方式。`protocol.schema.json` 包含 `since`
发布版本元数据和核心方法所需的权限范围元数据，但线协议
版本提升对第三方客户端而言仍是明确的破坏性事件。固定
测试过的软件包版本，在线协议版本变化时同步升级客户端和 Gateway 网关，并在
每次升级前查看
[OpenClaw 变更日志](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)。

## 相关内容

- [Gateway 网关协议](https://docs.openclaw.ai/gateway/protocol)
- [嵌入 OpenClaw](https://docs.openclaw.ai/gateway/embedding)
- [Gateway RPC 参考](https://docs.openclaw.ai/reference/rpc)
- [面向外部应用的 Gateway 网关集成](https://docs.openclaw.ai/gateway/external-apps)
