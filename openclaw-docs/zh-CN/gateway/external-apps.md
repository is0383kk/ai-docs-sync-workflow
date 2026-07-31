---
read_when:
    - 你正在构建一个与 OpenClaw 通信的外部应用、脚本、仪表板、CI 任务或 IDE 扩展
    - 你正在 Gateway RPC 与插件 SDK 之间进行选择
    - 你正在与 Gateway 网关的智能体运行、会话、事件、审批、模型或工具集成
    - 你正在将托管控制器与外部唤醒调度器配对
sidebarTitle: External apps
summary: 外部应用、脚本、仪表板、CI 任务和 IDE 扩展的当前集成路径
title: 外部应用的 Gateway 网关集成
x-i18n:
    generated_at: "2026-07-26T06:09:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 276c6f4173197683a60770327e131e6ab2fa4d33f416ba96c170539df7246f83
    source_path: gateway/external-apps.md
    workflow: 16
---

外部应用通过 Gateway 网关协议与 OpenClaw 通信：使用 WebSocket
传输和 RPC 方法。当脚本、仪表板、CI 作业、IDE
扩展或其他进程需要启动智能体运行、流式接收事件、等待
结果、取消工作或检查 Gateway 网关资源时，请使用此协议。

<Note>
  关于 npm 软件包、设备配对、重连恢复、历史记录、订阅
  和审批，请先阅读
  [构建 Gateway 客户端](https://docs.openclaw.ai/gateway/clients)。如果你的
  应用将 Gateway 网关作为子进程监管，还应阅读
  [嵌入 OpenClaw](https://docs.openclaw.ai/gateway/embedding)。在
  软件包初始发布期间，npm 可能会返回 `E404`，直到首个包含该软件包的
  OpenClaw 版本发布。
</Note>

<Note>
  本页面适用于 OpenClaw 进程之外的代码。在 OpenClaw
  内部运行的插件代码应改用已记录的 `openclaw/plugin-sdk/*` 子路径。
</Note>

## 当前可用功能

| 功能面                                                           | 状态          | 用途                                                                                          |
| ---------------------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------- |
| [Gateway 客户端指南](https://docs.openclaw.ai/gateway/clients)   | 发布序列      | npm 软件包、身份验证、重连、历史记录、事件、审批和版本策略。                                  |
| [嵌入指南](https://docs.openclaw.ai/gateway/embedding)           | 发布序列      | 子进程环境、就绪状态、生命周期、恢复、RPC 所有权和打包。                                      |
| [Gateway 网关协议](/zh-CN/gateway/protocol)                            | 就绪          | WebSocket 传输、连接握手、身份验证权限范围、协议版本控制和事件。                              |
| [Gateway RPC 参考](/zh-CN/reference/rpc)                               | 就绪          | 当前用于智能体、会话、任务、模型、工具、工件和审批的 Gateway 网关方法。                       |
| [`openclaw agent`](/zh-CN/cli/agent)                                 | 就绪          | 当通过 shell 调用 CLI 已足够时，用于一次性脚本集成。                                          |
| [`openclaw message`](/zh-CN/cli/message)                               | 就绪          | 从脚本发送消息或渠道操作。                                                                    |

## 推荐路径

1. 运行或发现 Gateway 网关。
2. 通过 [Gateway 网关协议](/zh-CN/gateway/protocol)连接。
3. 调用 [Gateway RPC 参考](/zh-CN/reference/rpc)中记录的 RPC 方法。
4. 固定你测试所针对的 OpenClaw 版本。
5. 升级 OpenClaw 时重新查看 RPC 参考。

对于智能体运行，请从 `agent` RPC 开始，并将其与 `agent.wait` 配合使用，以获取
终态结果。对于持久的对话状态，请使用 `sessions.*` 方法。
对于 UI 集成，请订阅 Gateway 网关事件，并且只呈现应用
能够理解的事件系列。

## 协作式主机挂起

冻结正在运行的进程或为其创建快照的托管控制器可以使用
与主机无关的挂起握手：

1. 停止接收由主机控制的外部入口流量。
2. 使用稳定且唯一的 `requestId` 调用 `gateway.suspend.prepare`。
3. 如果响应为 `busy`，请保持进程运行并稍后重试。
4. 如果响应为 `ready`，请保存返回的 `suspensionId`，然后在 `expiresAtMs`
   之前冻结进程或为其创建快照。
5. 解冻后，或者放弃挂起时，通过现有 WebSocket 或 Admin HTTP 控制
   路径，使用该 `suspensionId` 调用 `gateway.suspend.resume`。

已准备好的 Gateway 网关会拒绝新的 WebSocket 握手。WebSocket 控制器
必须在主机操作期间保持其已通过身份验证的连接处于打开状态。如果无法
保证这一点，请在准备之前启用并使用
[Admin HTTP RPC 插件](/zh-CN/plugins/admin-http-rpc)。如果
控制路径丢失，请等待两分钟租约到期后再
重新连接；租约到期会自动重新开放接入。

RPC 合约如下：

- `gateway.suspend.prepare` — `operator.admin`；参数
  `{ "requestId": "stable-host-operation-id" }`
- `gateway.suspend.status` — `operator.read`；参数
  `{ "suspensionId": "id-from-prepare" }`
- `gateway.suspend.resume` — `operator.admin`；参数
  `{ "suspensionId": "id-from-prepare" }`

ID 会去除首尾空白，必须包含一个非空白字符，并且上限为
128 个字符。繁忙的准备结果包含 `status: "busy"`、`reason`、
`retryAfterMs`、`activeCount` 和 `blockers`。就绪结果具有以下结构：

```json
{
  "status": "ready",
  "suspensionId": "2c3f...",
  "expiresAtMs": 1770000000000,
  "activeCount": 0,
  "blockers": []
}
```

状态返回 `{"status":"running"}`，或包含 `expiresAtMs` 的就绪结果。
恢复返回 `{"ok":true,"status":"running","resumed":true}`；成功恢复后
重复调用会返回 `resumed: false`。

相互冲突的请求 ID 或暂时性的调度器恢复失败会返回可重试的
`UNAVAILABLE`，其中包含 `retryAfterMs`。在调度器恢复期间，准备、状态
和恢复都会返回该错误，Gateway 网关保持未就绪并以故障关闭方式运行，
主机不得冻结它或为其创建快照。OpenClaw 会自动重试
调度器，并且只有在恢复成功后才会重新开放接入。
不匹配的恢复 ID 会返回 `INVALID_REQUEST`。准备操作与 Gateway 网关共享
每分钟三次尝试的控制平面写入预算；请遵守返回的
重试延迟。WebSocket 客户端按设备和 IP 分桶。Admin HTTP
控制器按解析出的客户端 IP 分桶，因此位于同一
代理后方的控制器可能共享一个预算。

准备操作仅会拒绝新工作：OpenClaw 关闭新的根级/会话/命令接入，
暂停自动定时任务触发，并同步检查工作。如果存在任何
活动工作，它会先恢复调度器并重新开放接入，然后再返回
`busy`；它不会中断或排空该工作。就绪租约持续两
分钟。使用相同的 `requestId` 重复调用 `prepare` 会续订租约；租约到期时，
系统会先恢复调度器，再重新开放接入。
在就绪租约期间到期应发出的重启会等待租约
恢复；正在进行的重启会使准备操作返回 `busy`。

处于就绪状态时，`/healthz` 仍保持可用，`/readyz` 返回 `503`。本地或
已通过身份验证的就绪响应包含 `gateway-draining`；未经身份验证的
远程探测仅会收到 `{ "ready": false }`。HTTP 健康探测、
现有 WebSocket 连接上的挂起方法，以及已启用的
Admin HTTP RPC 路由仍然可用。其他 RPC 返回可重试的
`UNAVAILABLE`。内置 HTTP 用户工作路由和普通插件 HTTP 路由，
包括与 OpenAI 兼容的 API、工具/会话操作、节点监视和
已配置的 Hooks，会返回包含 `error.code: "gateway_unavailable"` 的 `503`。新的
插件所有的 WebSocket 升级也会返回 `503`；这涵盖升级
所有权，而不涵盖稍后通过已建立的插件套接字执行的工作。

此握手不会持久化传入消息、停止第三方渠道
传输，也不会控制托管平台。主机必须在准备前隔离其入口
流量，并继续负责唤醒、创建快照/冻结和
停止。`activeCount` 是聚合后的受跟踪工作计数，而 `blockers`
包含非零类别计数和有界任务详情。这不是
通用的进程静止屏障。`background-exec` 阻塞项仅提供聚合信息：
命令文本、进程 ID、输出以及会话或权限范围标识符绝不会
通过协议传输。渠道健康检查、维护、缓存刷新、已建立的
插件 WebSocket 会话，以及未注册且归插件所有的后台工作可以
继续保持活动状态。
托管平台必须以一致方式冻结整个进程树及其
文件系统或为其创建快照；此初始合约无法证明未注册的工作
处于空闲状态。

<Tip>
  对于主机唤醒调度，请将面向 OpenClaw 的部分保留在进程内
  插件中，并将幂等的完整快照投影到外部主机适配器。
  托管控制器不应导入插件 SDK，也不应根据事件增量重建定时任务
  状态。请参阅[安全的外部定时任务
  投影](/zh-CN/plugins/hooks#safe-external-cron-projection)。
</Tip>

## 应用代码与插件代码

当代码位于 OpenClaw 外部时，使用 Gateway 网关 RPC：

- 启动或观察智能体运行的 Node 脚本
- 调用 Gateway 网关的 CI 作业
- 仪表板和管理面板
- IDE 扩展
- 无需成为渠道插件的外部桥接器
- 使用模拟或真实 Gateway 网关传输的集成测试

当代码在 OpenClaw 内部运行时，使用插件 SDK：

- 提供商插件
- 渠道插件
- 工具或生命周期 Hooks
- Agent harness plugins
- 受信任的运行时辅助程序

外部应用不应导入 `openclaw/plugin-sdk/*`；这些子路径供
OpenClaw 加载的插件使用。

## 相关内容

- [构建 Gateway 客户端](https://docs.openclaw.ai/gateway/clients)
- [嵌入 OpenClaw](https://docs.openclaw.ai/gateway/embedding)
- [Gateway 网关协议](/zh-CN/gateway/protocol)
- [Gateway RPC 参考](/zh-CN/reference/rpc)
- [CLI 智能体命令](/zh-CN/cli/agent)
- [CLI 消息命令](/zh-CN/cli/message)
- [Agent loop](/zh-CN/concepts/agent-loop)
- [Agent Runtimes](/zh-CN/concepts/agent-runtimes)
- [会话](/zh-CN/concepts/session)
- [后台任务](/zh-CN/automation/tasks)
- [ACP 智能体](/zh-CN/tools/acp-agents)
- [插件 SDK 概览](/zh-CN/plugins/sdk-overview)
