---
read_when:
    - 在没有 macOS UI 的情况下实现节点配对审批
    - 添加用于审批远程节点的 CLI 流程
    - 使用节点管理扩展 Gateway 网关协议
summary: 节点能力审批：设备配对后节点如何获得命令访问权限
title: 节点配对
x-i18n:
    generated_at: "2026-07-26T06:16:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 25e4016657379573ddb7e9027899afd8b97b16709da6e73ed44d4016b99e715a
    source_path: gateway/pairing.md
    workflow: 16
---

节点配对分为两层，两者都存储在 Gateway 网关的 SQLite 状态数据库中的已配对设备记录上：

- **设备配对**（角色 `node`）负责控制 `connect` 握手。请参阅下文的
  [受信任 CIDR 设备自动批准](#trusted-cidr-device-auto-approval)
  和[渠道配对](/zh-CN/channels/pairing)。
- **节点能力批准**（`node.pair.*`）负责控制已连接节点可以公开哪些已声明的
  能力/命令。Gateway 网关是事实来源；UI（macOS 应用、Control UI）是用于批准或
  拒绝待处理请求的前端。

以前独立的节点配对存储（`nodes/paired.json`，包含每节点
令牌，已于 2026 年 1 月从连接路径中停用）现已移除：Gateway 网关会在启动时将
所有剩余行一次性合并到设备记录中，并使用 `.migrated` 后缀归档
旧文件。旧版 TCP 桥接支持已移除。

## 能力批准的工作原理

1. 节点连接到 Gateway 网关 WS（设备配对负责控制此步骤）。
2. Gateway 网关将已声明的能力/命令范围与已批准的范围进行比较；新增或扩大的范围会在
   设备记录上存储一个**待处理请求**，并发出 `node.pair.requested`。
3. 你可以批准或拒绝该请求（通过 CLI 或 UI）。
4. 在获得批准前，节点命令会保持过滤状态；批准后将公开已声明的
   范围，但仍受常规命令策略约束。

待处理请求会在**节点最后一次重试后 5 分钟**自动过期——持续主动重新连接的节点会让其唯一的待处理请求保持有效，
而不是每次尝试都生成新请求（和批准提示）。

## CLI 工作流（适合无头环境）

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes status
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name "Living Room iPad"
```

`nodes status` 显示已配对/已连接的节点及其能力。

## API 表面（Gateway 网关协议）

事件：

- `node.pair.requested` - 创建新的待处理请求时发出。
- `node.pair.resolved` - 请求获得批准、被拒绝或
  过期时发出。

方法：

- `node.pair.list` - 列出待处理和已配对的节点（`operator.pairing`）。
- `node.pair.approve` - 批准待处理请求。
- `node.pair.reject` - 拒绝待处理请求。
- `node.pair.remove` - 移除已配对节点。此操作会撤销已配对设备存储中该设备的 `node`
  角色，同时移除已批准的节点范围，并
  使该设备的节点角色会话失效并断开连接。**混合角色**
  设备（例如还拥有 `operator` 的设备）会保留其记录，仅
  失去 `node` 角色；仅有节点角色的设备记录则会被删除。授权：
  `operator.pairing` 可以移除非操作员节点记录；使用设备令牌的调用者若要在混合角色设备上
  撤销其**自身的**节点角色，还需要
  `operator.admin`。
- `node.rename` - 重命名已配对节点面向操作员的显示名称。

已在 2026.7 中移除：`node.pair.request` 和 `node.pair.verify`。待处理
请求由 Gateway 网关自身在节点连接期间创建，而它们所服务的
独立每节点令牌已不复存在；节点身份验证使用
设备配对令牌。

注意：

- 以未变化的范围重新连接时会复用待处理请求；重复
请求会刷新已存储的节点元数据和最新的允许列表内
已声明命令快照，以供操作员查看。
- 操作员权限范围级别和批准时检查汇总于
  [操作员权限范围](/zh-CN/gateway/operator-scopes)。
- `node.pair.approve` 使用待处理请求中声明的命令来强制执行
  额外的批准权限范围：
  - 无命令请求：`operator.pairing`
  - 普通命令请求：`operator.pairing` + `operator.write`
  - 包含 `system.run`、`system.run.prepare`、
    `system.which`、`browser.proxy`、`fs.listDir` 或
    `system.execApprovals.get/set` 的管理员敏感请求：`operator.pairing` + `operator.admin`

<Warning>
节点配对批准会记录受信任的能力范围。它**不会**为每个节点固定实时节点命令范围。

- 实时节点命令来自节点连接时声明的内容，并由
  Gateway 网关的全局节点命令策略（`gateway.nodes.commands.allow` 和
  `gateway.nodes.commands.deny`）进行过滤。
- 每节点 `system.run` 的允许和询问策略存放在
  `exec.approvals.node.*` 中的节点上，而不在配对记录中。

</Warning>

## 节点命令门控（2026.3.31+）

<Warning>
**重大变更：**从 `2026.3.31` 开始，在节点配对获得批准前，节点命令将被禁用。仅完成设备配对已不足以公开声明的节点命令。
</Warning>

节点首次连接时，会自动请求配对。
在该请求获得批准前，来自该节点的所有待处理节点命令都会被
过滤且不会执行。配对获得批准后，节点声明的
命令将变为可用，但仍受常规命令策略约束。

这意味着：

- 以前仅依赖设备配对来公开命令的节点，现在
  还必须完成节点配对。
- 在配对批准前排队的命令会被丢弃，而不会延迟执行。

## 节点事件信任边界（2026.3.31+）

<Warning>
**重大变更：**节点发起的运行现在仅限于缩减后的受信任范围。
</Warning>

节点发起的摘要及相关会话事件仅限于
预期的受信任范围。以前依赖更广泛主机或会话工具访问权限的通知驱动或节点触发流程
可能需要调整。
此强化措施可防止节点事件升级并获得超出节点信任边界所允许范围的
主机级工具访问权限。

持久节点在线状态更新遵循相同的身份边界：
`node.presence.alive` 事件仅接受来自已经过身份验证的节点设备
会话，并且只有在设备/节点身份已经配对时才更新配对元数据。
自行声明的 `client.id` 值不足以写入
最后上线状态。

## 经 SSH 验证的设备自动批准（默认）

首次从私有/CGNAT 地址进行 `role: node` 设备配对时，如果 Gateway 网关能够**通过 SSH 证明机器所有权**，
则会自动批准：它会反向连接到配对主机（`BatchMode`、`StrictHostKeyChecking=yes`），
在该主机上运行 `openclaw node identity --json`，并且仅当远程
设备 ID 和公钥与待处理请求完全匹配时才批准。密钥匹配
确保了此过程的安全性：仅能连接绝不会触发批准，因此 NAT 共同租户、
共享主机上的其他用户和局域网欺骗都会转入正常的
提示流程。

默认启用。触发要求：

- Gateway 网关进程用户（或 `sshVerify.user`）可以通过 SSH 非交互式连接到节点主机
  （使用密钥/智能体；Tailscale SSH 也可），并且该主机密钥
  已受信任。
- `openclaw` 能在远程 `PATH` 上解析，以供非交互式 `sh -lc` 使用。
- 连接 IP 是直接的（未经代理且非环回）私有、ULA、
  链路本地或 CGNAT 地址，或者在设置 `sshVerify.cidrs` 时与之匹配。
- 适用门槛与受信任 CIDR 批准相同：仅限没有权限范围的新节点
  配对；升级、浏览器、Control UI 和 WebChat 始终提示。

探测运行期间，节点客户端会收到继续重试的指示
（`wait_then_retry`），而不是暂停并等待手动批准；如果探测
失败，下一次尝试会回退到正常提示流程。失败的目标
会进入短暂冷却期（密钥不匹配后 5 分钟）。

获批准的设备会记录 `approvedVia: "ssh-verified"`，其首次声明的
能力范围也会在同一步骤中获得批准——密钥匹配已经证明
节点在操作员拥有的机器上以操作员账户运行，这与
手动能力批准所确认的声明相同。后续范围升级仍会
提示。

强化或禁用：

```json5
{
  gateway: {
    nodes: {
      pairing: {
        // 完全禁用：
        sshVerify: false,
        // ...或限定/调整探测：
        // sshVerify: { user: "me", identity: "~/.ssh/probe", timeoutMs: 7000, cidrs: ["10.0.0.0/8"] },
      },
    },
  },
}
```

## 自动批准（macOS 应用）

在以下情况下，macOS 应用可以尝试**静默批准**节点能力请求：

- 请求被标记为 `silent`（当设备配对以非交互方式获得批准时，Gateway 网关会将首个能力
  范围标记为静默），并且
- 应用可以使用相同
  用户验证与 Gateway 网关主机的 SSH 连接。

如果静默批准失败，则会回退到正常的 Approve/Reject 提示。

## 受信任 CIDR 设备自动批准

`role: node` 的 WS 设备配对默认保持手动模式。对于 Gateway 网关已经信任网络路径的私有节点
网络，操作员可以通过显式 CIDR 或精确 IP 选择启用：

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

安全边界：

- 未设置 `gateway.nodes.pairing.autoApproveCidrs` 时禁用。
- 不存在一概适用于局域网或私有网络的自动批准模式；经 SSH 验证的
  自动批准（见上文）需要加密设备密钥匹配，绝不会
  仅依赖网络位置。
- 只有不请求任何权限范围的新 `role: node` 设备配对请求
  符合条件。
- 操作员、浏览器、Control UI 和 WebChat 客户端保持手动模式。
- 角色、权限范围、元数据和公钥升级保持手动模式。
- 同主机环回受信任代理标头路径不符合条件，因为该
  路径可能被本地调用者欺骗。

## 静默配对取代清理

非交互式批准会在已配对设备记录中记下其来源：
同主机本地策略批准记为 `silent`，受信任 CIDR 节点批准记为
`trusted-cidr`，经 SSH 验证的节点批准记为 `ssh-verified`。状态目录为临时目录（临时主目录、
容器、每次运行独立沙箱）的客户端会在每次运行时生成新的设备密钥对，并且每次
运行都会以全新设备静默重新配对——如果不清理，已配对列表
每次运行都会增加一条陈旧记录。

当 Gateway 网关静默批准**本地**设备配对时，它会停用
属于同一客户端集群的旧 `silent` 批准记录
（`clientId`、`clientMode` 和显示名称均匹配），且这些记录当前未
连接。本地客户端运行在 Gateway 网关主机本身，因此集群键
不可能匹配其他机器。停用的记录会立即失去其令牌；
所有匹配的旧版节点配对条目都会被清除，并广播 `node.pair.resolved`
移除事件。

边界：

- 只有最新一次批准属于同一主机本地批准（`silent`）的记录才符合条件，
  无论作为触发方还是目标。受信任 CIDR 和经 SSH 验证的配对会跨越主机，
  而显示元数据并不能代表机器身份，因此绝不会自动移除这些配对——请使用
  Control UI 清理功能或 `openclaw nodes remove` 进行清理。
- 所有者批准的配对以及通过二维码/设置代码完成的（引导）配对绝不会
  自动移除。在来源信息功能引入之前批准的记录会继续受到保护，
  即使之后对同一设备 ID 再次进行了静默批准，也是如此。
- 当前已连接的设备会被跳过，因此使用不同状态目录的并发本地会话
  在连接存续期间会保留其令牌。最近一分钟内批准的记录也会被跳过，
  因此同时进行的配对握手不会在各自连接完成注册前将对方移除。
- 受影响的客户端在设计上均为本地客户端，因此会在下次连接时
  静默重新配对。

## 元数据升级自动批准

当已配对的设备重新连接且只有非敏感元数据发生变化
（例如显示名称或客户端平台提示）时，OpenClaw 会将其视为
`metadata-upgrade`。静默自动批准的适用范围很窄：它仅适用于受信任的非浏览器
本地重连，且客户端必须已证明其持有本地或共享凭据；这包括操作系统版本
元数据发生变化后，同一主机上的原生应用重新连接。浏览器/Control UI 客户端和远程客户端
仍使用显式重新批准流程。从读取权限升级到写入/管理员权限等权限范围升级，以及
公钥变更，**不**符合元数据升级自动批准条件；它们仍会作为显式重新批准请求处理。

## 二维码配对辅助功能

`/pair qr` 会将配对载荷呈现为结构化媒体，以便移动端和
浏览器客户端直接扫描。

删除设备时，也会清理该设备 ID 对应的所有过期待处理配对请求，
因此撤销后 `nodes pending` 不会显示孤立记录。

## 本地性和转发标头

仅当原始套接字和所有上游代理证据均一致时，Gateway 网关配对才会将连接视为 local loopback。
如果请求通过 local loopback 到达，但携带 `Forwarded`、任何
`X-Forwarded-*` 或 `X-Real-IP` 标头证据，则这些转发标头证据会使
local loopback 本地性声明失效，配对路径将要求显式批准，而不是静默地将请求
视为同一主机连接。有关操作员身份验证的等效规则，请参阅
[受信任代理身份验证](/zh-CN/gateway/trusted-proxy-auth)。

## 存储（本地、私有）

配对状态存储在共享 SQLite 状态数据库的已配对设备记录中，
该数据库位于 Gateway 网关状态目录下（默认值为 `~/.openclaw`）：

- `~/.openclaw/state/openclaw.sqlite`（已配对设备及其设备身份验证信息、
  已批准的节点接口、待处理的接口请求、待处理的设备配对请求和引导令牌）

如果覆盖 `OPENCLAW_STATE_DIR`，数据库也会随之迁移。从使用 JSON 存储的版本升级而来的
Gateway 网关会在启动时导入这些数据，并留下 `devices/*.json.migrated` 和
`nodes/*.json.migrated` 归档。

安全注意事项：

- 设备令牌属于机密信息；应将状态数据库视为敏感数据。
- 轮换设备令牌使用 `openclaw devices rotate` /
  `device.token.rotate`。

## 传输行为

- 传输层是**无状态的**；它不存储成员关系。
- 如果 Gateway 网关离线或配对已禁用，节点将无法配对。
- 在远程模式下，配对针对远程 Gateway 网关的存储进行。

## 相关内容

- [渠道配对](/zh-CN/channels/pairing)
- [节点 CLI](/zh-CN/cli/nodes)
- [设备 CLI](/zh-CN/cli/devices)
