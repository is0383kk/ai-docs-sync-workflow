---
read_when:
    - 调试缺少操作员权限范围的错误
    - 审查设备或节点配对审批
    - 添加或分类 Gateway 网关 RPC 方法
summary: Gateway 客户端的操作员角色、权限范围和审批时检查
title: 操作员权限范围
x-i18n:
    generated_at: "2026-07-26T06:15:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 40053793bb5a80afab28fdfcdcac6565abde6bca988389b03a407272c70043e2
    source_path: gateway/operator-scopes.md
    workflow: 16
---

操作员权限范围控制 Gateway 网关客户端在通过身份验证后可以执行的操作。
它们是单个可信 Gateway 网关操作员域内的控制平面防护机制，
并非针对恶意多租户环境的隔离机制。若要在人员、团队或机器之间实现强隔离，
请在不同的操作系统用户或主机下运行独立的 Gateway 网关。

相关内容：[安全](/zh-CN/gateway/security)、[Gateway 网关协议](/zh-CN/gateway/protocol)、
[Gateway 网关配对](/zh-CN/gateway/pairing)、[设备 CLI](/zh-CN/cli/devices)。

## 角色

每个 Gateway 网关 WebSocket 客户端都使用以下一种角色连接：

- `operator`：控制平面客户端，例如 CLI、Control UI、自动化和
  可信辅助进程。
- `node`：通过 `node.invoke` 公开命令的能力宿主
  （macOS、iOS、Android、无头设备）。

操作员 RPC 方法要求 `operator` 角色；源自节点的方法
要求 `node` 角色。

## 权限范围级别

| 权限范围                   | 含义                                                                                                                                                       |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operator.read`         | 只读状态、列表、目录、日志、会话读取及其他非修改型调用。                                                                          |
| `operator.write`        | 修改型操作员操作：发送消息、调用工具、更新 Talk/语音设置、转发节点命令。同时满足 `operator.read`。                |
| `operator.admin`        | 管理访问权限。满足所有 `operator.*` 权限范围。修改配置、执行更新、使用原生钩子、访问保留命名空间和进行高风险审批时必须具备此权限。 |
| `operator.pairing`      | 设备和节点配对管理：列出、批准、拒绝、移除、轮换、撤销。                                                                            |
| `operator.approvals`    | Exec 和插件审批 API。                                                                                                                                |
| `operator.questions`    | 列出、读取、回答和解决交互式问题。                                                                                             |
| `operator.talk.secrets` | 读取包含密钥的 Talk 配置。                                                                                                             |

对于未来未知的 `operator.*` 权限范围，除非调用方已持有
`operator.admin`，否则必须精确匹配。

## 方法权限范围只是第一道关卡

每个 Gateway 网关 RPC 都有一个最小权限方法权限范围，用于决定请求是否能到达其处理程序。
参数感知型方法会在分发前推导该权限范围，
以便授权失败统一返回规范的结构化响应：

- `agent` 对普通轮次需要 `operator.write`，对
  `/new` 或 `/reset` 会话生命周期命令则需要 `operator.admin`。
- `node.invoke` 对普通转发命令需要 `operator.write`，对
  `browser.proxy`、`fs.listDir` 和 `terminal.upload` 则需要
  `operator.admin`。
- `talk.config` 需要 `operator.read`；`includeSecrets: true` 还需要
  `operator.talk.secrets`。

随后，某些处理程序会根据具体审批或修改的对象执行更严格的检查：

- `device.pair.approve` 可凭 `operator.pairing` 访问，但批准操作员设备时，
  只能签发或保留调用方已经持有的权限范围。
- `node.pair.approve` 可凭 `operator.pairing` 访问，随后会根据
  待处理节点声明的命令列表推导额外的审批权限范围。
- `chat.send` 是写入权限方法，但 `/config set` 和
  `/config unset` 聊天命令还需要额外具备 `operator.admin`，
  无论调用方是否具备聊天发送权限范围。

这样，权限范围较低的操作员可以执行低风险的配对操作，
而无需将所有配对审批都限制为仅管理员可执行。

会话修改 RPC 根据协商确定的操作员权限范围进行授权，
与连接客户端的 `client.id` 或 `client.mode` 无关。客户端身份
仍可能影响连接和设备身份验证策略，但它既不会授予也不会移除
会话修改权限。

## 设备配对审批

设备配对记录是已批准角色和权限范围的持久化权威来源。
已配对设备不会在不作提示的情况下获得更广泛的访问权限：如果重新连接时
请求更广泛的角色或权限范围，将创建新的待处理升级请求。

批准设备请求：

- 不包含操作员角色的请求无需操作员权限范围审批。
- 请求非操作员设备角色（例如 `node`）时，需要
  `operator.admin`，即使 `device.pair.approve` 本身只需要
  `operator.pairing`。
- 请求 `operator.read`、`operator.write`、`operator.approvals`、
  `operator.questions`、`operator.pairing` 或 `operator.talk.secrets` 时，
  调用方必须已经持有相应权限范围或 `operator.admin`。
- 请求 `operator.admin` 时，需要 `operator.admin`。
- 不包含显式权限范围的修复请求可以继承现有操作员
  令牌的权限范围；如果该令牌具有管理员权限范围，批准操作仍需要
  `operator.admin`。

非管理员共享密钥会话和可信代理会话只能在其自身声明的操作员权限范围内
批准操作员设备请求；即使这些会话在其他情况下可以使用
`operator.pairing`，批准非操作员角色仍仅限管理员。

对于使用已配对设备令牌的会话，除非调用方具有
`operator.admin`，否则管理权限仅限自身：非管理员调用方只能看到自己的配对条目，
并且只能批准、拒绝、轮换、撤销或移除自己的设备条目。

## 节点配对审批

旧版 `node.pair.*` 方法使用由 Gateway 网关单独管理的节点配对存储。
WebSocket 节点则使用设备配对（`role: node`），但适用相同的审批
术语。有关这两个存储之间的关系，请参阅 [Gateway 网关配对](/zh-CN/gateway/pairing)。

`node.pair.approve` 根据待处理请求的命令列表推导额外的必需权限范围：

| 声明的命令                                                                                                    | 必需权限范围                       |
| -------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| 无                                                                                                                 | `operator.pairing`                    |
| 普通节点命令                                                                                               | `operator.pairing` + `operator.write` |
| `system.run`、`system.run.prepare`、`system.which`、`browser.proxy`、`fs.listDir` 或 `system.execApprovals.get/set` | `operator.pairing` + `operator.admin` |

批准节点声明不会启用受独立运行时允许列表关卡限制的命令。
例如，批准声明了 `computer.act` 的节点需要配对权限和写入权限，
但只会记录该能力表面。管理员或所有者仍必须启用 `computer.act`。
在其保持启用期间，通过 `node.invoke` 调用它需要写入权限，
但每次操作不需要管理员权限。

节点配对用于建立身份和信任；它不能取代节点自身的
`system.run` Exec 审批策略。

## 共享密钥身份验证

共享 Gateway 网关令牌/密码身份验证被视为对该 Gateway 网关的可信操作员访问。
对于共享密钥持有者身份验证，OpenAI 兼容 HTTP 接口、`/tools/invoke` 和 HTTP
会话历史记录端点会恢复完整的默认操作员权限范围集，即使调用方发送了更窄的声明权限范围。

携带身份的模式（例如可信代理身份验证或专用入口 `none`）
仍可遵循显式声明的权限范围。若要实现真正的信任边界隔离，请使用独立的 Gateway 网关。
