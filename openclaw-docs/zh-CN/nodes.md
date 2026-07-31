---
read_when:
    - 将 iOS/watchOS/Android 节点配对到 Gateway 网关
    - 使用节点画布/摄像头获取智能体上下文
    - 添加新的节点命令或 CLI 辅助工具
summary: 节点：配对、能力、权限，以及用于画布/摄像头/屏幕/设备/通知/系统的 CLI 辅助工具
title: 节点
x-i18n:
    generated_at: "2026-07-26T06:46:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b4f7c80491d713777e1ba5b8f55c88bd9fa48be602b504e6ac6ba00cd12a4313
    source_path: nodes/index.md
    workflow: 16
---

**节点**是连接到 Gateway 网关的配套设备（macOS/iOS/watchOS/Android/无头设备），它使用 `role: "node"` 进行连接，并通过 `node.invoke` 提供命令接口（例如 `canvas.*`、`camera.*`、`device.*`、`notifications.*`、`system.*`）。大多数节点使用操作员端口上的 Gateway WebSocket。可选的直连 Apple Watch 节点在同一端口上使用签名 HTTPS 轮询，因为 watchOS 会阻止普通应用进行通用的底层网络通信。协议详情：[Gateway 协议](/zh-CN/gateway/protocol)。

旧版传输协议：[Bridge protocol](/zh-CN/gateway/bridge-protocol)（TCP JSONL；仅供当前节点的历史参考）。

macOS 也可以在**节点模式**下运行：菜单栏应用作为一个节点连接到 Gateway 网关的
WS 服务器（因此 `openclaw nodes …` 可作用于这台 Mac）。该应用
将原生 Canvas、摄像头、屏幕、通知和计算机控制命令
添加到 `openclaw node run` 所使用的同一节点主机命令接口中。不要在该 Mac 上启动
第二个 CLI 节点；该应用会将对应的 CLI 节点主机运行时作为
内部工作进程运行，并保持为唯一的 Gateway 网关连接和节点身份。

节点是**外围设备**，而不是网关：它们不运行网关服务，渠道消息（Telegram、WhatsApp 等）会送达网关，而不是节点。

故障排查运行手册：[/nodes/troubleshooting](/zh-CN/nodes/troubleshooting)

## 配对 + 状态

节点使用**设备配对**。节点在连接期间提供签名的设备身份；Gateway 网关为 `role: node` 创建设备配对请求。通过设备 CLI（或 UI）批准。直连 Apple Watch 的设置使用由管理员签发、短期有效且仅限节点使用的设置代码，批准其固定的低风险命令接口；后续扩展能力仍需正常批准。

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
```

待处理的配对请求会在设备最后一次重试 5 分钟后过期——持续重新连接的设备会让其唯一的待处理请求（以及 `requestId`）保持有效，而不是每隔几分钟生成一次新提示；完整的请求/批准生命周期请参阅[节点配对](/zh-CN/gateway/pairing)。如果节点使用已更改的身份验证详情（角色/权限范围/公钥）重试，先前的待处理请求会被取代，并创建新的 `requestId`——客户端会收到针对被取代请求的 `device.pair.resolved` 事件，批准前应重新运行 `openclaw devices list`。

- `nodes status` 在节点的设备配对角色包含 `node` 时，将节点标记为**已配对**。
- 已连接的原生 Mac 可以通过
  **Settings -> Permissions -> Active computer detection** 选择启用合并后的物理输入活动检测。还需要
  辅助功能权限。Gateway 网关会将最新的符合条件的 Mac 标记为
  `active`，为智能体提供稳定的节点 ID 提示，并优先将节点连接
  提醒路由到该设备，随后才进行延迟回退。有关设置、隐私、时序和
  故障排查，请参阅
  [活动计算机在线状态](/zh-CN/nodes/presence)。
- 设备配对记录是持久的已批准角色契约。令牌轮换始终限制在该契约内；它无法将已配对节点升级为配对批准从未授予的角色。
- `node.pair.*`（CLI：`openclaw nodes pending/approve/reject/remove/rename`）是由网关所有的独立节点配对存储，用于跟踪节点在多次重新连接之间获准使用的命令/能力接口。它**不**控制传输身份验证——设备配对负责该功能。
- `openclaw nodes remove --node <id|name|ip>` 会移除节点配对。对于由设备支持的节点，它会在已配对设备存储中撤销该设备的 `node` 角色，并断开该设备的节点角色会话：混合角色设备会保留其记录，仅失去 `node` 角色，而仅限节点的设备记录会被删除。它还会从独立的节点配对存储中清除所有匹配条目。`operator.pairing` 可以移除其他设备上的非操作员节点记录；使用设备令牌的调用方若要在混合角色设备上撤销自身节点角色，还需要 `operator.admin`。
- 批准范围遵循待处理请求声明的命令：
  - 无命令请求：`operator.pairing`
  - 非 exec 节点命令：`operator.pairing` + `operator.write`
  - `system.run` / `system.run.prepare` / `system.which`：`operator.pairing` + `operator.admin`

## 版本偏差和升级顺序

Gateway WebSocket 在 N-1 协议窗口内接受经过身份验证的节点客户端。
因此，当前的 v4 Gateway 网关会接受同时声明
`role: "node"` 和 `client.mode: "node"` 的 v3 节点连接。操作员和 UI 会话仍
必须使用当前协议。

对于分阶段的设备群升级，请先升级 Gateway 网关，再升级各个节点。
N-1 节点在升级期间仍然可见且可管理；Gateway 网关会记录
`legacy node protocol accepted` 并提供升级建议。配对、
设备身份验证、命令允许列表和 exec 审批仍然适用。
插件拥有的能力和命令将保持隐藏，直到节点升级到
当前协议。早于 N-1 的节点必须先通过带外方式升级，然后才能
重新连接。

直连 watchOS HTTPS 传输要求使用当前协议版本；启用直连模式前，
请同时更新 watch 应用和 Gateway 网关。

## 远程节点主机（system.run）

当 Gateway 网关在一台计算机上运行，而你希望在另一台计算机上执行命令时，请使用**节点主机**。模型仍与**网关**通信；选择 `host=node` 后，网关会将 `exec` 调用转发到**节点主机**。

| 角色         | 职责                                                   |
| ------------ | ---------------------------------------------------------------- |
| Gateway 网关主机 | 接收消息、运行模型并路由工具调用。            |
| 节点主机    | 在节点计算机上执行 `system.run`/`system.which`。        |
| 审批    | 通过 `~/.openclaw/exec-approvals.json` 在节点主机上强制执行。 |

审批注意事项：

- 由审批支持的节点运行会绑定确切的请求上下文。exec 路径会在审批前准备规范的 `systemRunPlan`；批准后，网关会转发该已存储计划，而不是调用方后来编辑的任何命令/cwd/会话字段，并在运行前重新验证工作目录。
- 对于直接执行的 shell/运行时文件，OpenClaw 还会尽最大努力绑定一个具体的本地文件操作数；如果该文件在执行前发生变化，则拒绝运行。
- 如果 OpenClaw 无法为解释器/运行时命令准确识别一个具体的本地文件，则会拒绝由审批支持的执行，而不是假装已完整覆盖运行时。对于更广泛的解释器语义，请使用沙箱隔离、独立主机，或明确受信任的允许列表/完整工作流。

### 启动节点主机（前台）

在节点计算机上：

```bash
openclaw node run --host <gateway-host> --port 18789 --display-name "Build Node"
```

`node run` 还接受 `--context-path`（Gateway WS 上下文路径）、`--tls`、`--tls-fingerprint <sha256>` 和 `--node-id`（覆盖旧版客户端实例 ID；这不会重置配对）。在 macOS 上，传入 `--share-installed-apps` 以公布 `device.apps`；共享功能默认关闭。使用 `--no-share-installed-apps` 可禁用之前保存的启用设置。

### 通过 SSH 隧道连接远程网关（回环绑定）

如果 Gateway 网关绑定到回环地址（`gateway.bind=loopback`，本地模式下的默认值），远程节点主机便无法直接连接。请创建 SSH 隧道，并将节点主机指向隧道的本地端。

示例（节点主机 -> 网关主机）：

```bash
# 终端 A（保持运行）：将本地 18790 转发到网关 127.0.0.1:18789
ssh -N -L 18790:127.0.0.1:18789 user@gateway-host

# 终端 B：导出网关令牌并通过隧道连接
export OPENCLAW_GATEWAY_TOKEN="<gateway-token>"
openclaw node run --host 127.0.0.1 --port 18790 --display-name "Build Node"
```

注意：

- `openclaw node run` 支持令牌或密码身份验证。
- 首选环境变量：`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`。
- 配置回退项为 `gateway.auth.token` / `gateway.auth.password`。
- 在本地模式下，节点主机会有意忽略 `gateway.remote.token` / `gateway.remote.password`。
- 在远程模式下，`gateway.remote.token` / `gateway.remote.password` 可按远程优先级规则使用。
- 如果配置了有效的本地 `gateway.auth.*` SecretRefs 但无法解析，节点主机身份验证会以安全关闭方式失败。
- 节点主机身份验证解析仅接受 `OPENCLAW_GATEWAY_*` 环境变量。

### 启动节点主机（服务）

```bash
openclaw node install --host <gateway-host> --port 18789 --display-name "Build Node"
openclaw node start
openclaw node restart
```

`node install` 还接受 `--context-path`、`--tls`、`--tls-fingerprint`、`--node-id`（仅限旧版客户端实例 ID）、`--share-installed-apps` / `--no-share-installed-apps`、`--runtime <node>`（默认值：node），以及用于重新安装的 `--force`。还可以使用 `node status`、`node stop` 和 `node uninstall`。

### 配对 + 命名

在网关主机上：

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

如果节点使用已更改的身份验证详情重试，请重新运行 `openclaw devices list` 并批准当前的 `requestId`。

命名选项：

- `--display-name`，用于 `openclaw node run` / `openclaw node install`（与客户端实例 ID 和 Gateway 网关连接元数据一起持久保存在共享的 `node_host_config` SQLite 记录中）。
- `openclaw nodes rename --node <id|name|ip> --name "Build Node"`（网关覆盖值）。

### 节点托管的 MCP 服务器

请在节点计算机上的 `openclaw.json` 中配置 MCP 服务器，而不是在
Gateway 网关上配置：

```json5
{
  nodeHost: {
    mcp: {
      servers: {
        localDocs: {
          command: "npx",
          args: ["-y", "@modelcontextprotocol/server-filesystem", "/srv/docs"],
          toolFilter: {
            include: ["read_*", "search"],
          },
        },
        internalApi: {
          url: "https://mcp.internal.example/mcp",
          transport: "streamable-http",
          headers: {
            Authorization: "Bearer ${INTERNAL_MCP_TOKEN}",
          },
        },
      },
    },
  },
}
```

无头节点主机会启动这些服务器，列出其工具，并在连接后发布
描述符。工具调用通过 `mcp.tools.call.v1` 返回该节点；Gateway 网关不需要
匹配的 MCP 配置或 JS 插件。此节点托管的 v1 路径不支持 OAuth MCP
服务器。

当前节点主机会在初次配对期间声明内置的 `mcp.tools.call.v1` 命令族，
即使未配置任何 MCP 服务器也是如此。在较旧 OpenClaw 版本上配对的节点，
可能会在节点主机更新后请求一次性命令接口升级。此后添加、移除或筛选
服务器不需要重新配对，因为已批准的命令族保持不变。重启
`openclaw node run` 或 `openclaw node restart` 以应用节点 MCP 配置更改；
节点主机不会监视此配置。

Gateway 网关操作员可以使用
`gateway.nodes.pluginTools.enabled: false` 忽略由已配对节点发布的所有智能体可见工具，
包括节点托管的 MCP 工具。精确的命令拒绝规则（例如
`gateway.nodes.commands.deny: ["mcp.tools.call.v1"]`）也会阻止执行。

### 节点托管的 Skills

在节点计算机的 OpenClaw 当前 Skills 目录下安装 Skills，默认目录为
`~/.openclaw/skills`。`OPENCLAW_HOME`、`OPENCLAW_STATE_DIR` 和
`OPENCLAW_CONFIG_PATH` 会移动此当前配置文件。对于 Skills，`OPENCLAW_STATE_DIR`
优先；否则，`skills/` 位于
`openclaw config file` 所输出路径的旁边。无头节点主机连接后会发布有效的
`SKILL.md` 文件，并且仅当该节点保持连接时，Gateway 网关才会将其添加到
智能体的 Skills 快照中。每个 Skills 目录名称必须与 `name`
frontmatter 字段匹配，以便抽象节点定位器映射到单个条目，而无需添加
另一个协议字段。

初始节点角色配对会批准 Skills 发布。添加、移除或
更改 Skills 无需再次配对或更改 Gateway 配置。
更改节点 Skills 文件后，请重启 `openclaw node run` 或
`openclaw node restart`；节点主机不会监视 Skills 目录。

节点托管的 Skills 条目会标识其节点并携带其执行
位置。Skills 文件、引用的相对路径和二进制文件均保留在该
节点上。智能体使用常规 `read` 工具读取公布的
`node://.../SKILL.md` 位置。`file_fetch` 接受经操作员批准的节点绝对路径，
而非节点 Skills 定位器；没有常规读取工具的运行时可以改为通过
`exec host=node node=<node-id>` 运行 `cat SKILL.md`，并将公布的
`node://.../skills/<name>` 目录用作 `workdir`。引用的文件和二进制文件
使用相同的执行目标和工作目录。节点主机会根据
其当前 OpenClaw 状态目录解析该定位器，因此相对路径是在节点上解析，
而不是在 Gateway 网关计算机上解析。发布节点必须已批准 `system.run`，
且智能体的 Exec 策略必须允许 `host=node`；否则，该 Skills 不会
出现在该智能体的快照中。

在节点上设置 `nodeHost.skills.enabled: false` 可停止发布。Gateway 网关
操作员可以使用 `gateway.nodes.allowSkills: false`
忽略所有已配对节点中的 Skills。

### 无头身份状态

无头节点在共享 SQLite 中保存三条独立的状态记录：

- `~/.openclaw/state/openclaw.sqlite`（`node_host_config`）：客户端实例 ID、显示名称和 Gateway 网关连接元数据。
- `~/.openclaw/state/openclaw.sqlite`（`device_identities`，键 `primary`）：已签名的设备密钥对和派生的加密设备 ID。
- `~/.openclaw/state/openclaw.sqlite`（`device_auth_tokens`）：按加密设备 ID 和角色索引的已配对设备身份验证令牌。

对于已签名节点，Gateway 网关使用加密设备 ID 进行配对和
节点路由。客户端实例 ID 仅是连接元数据。因此，更改
`--node-id` 或迁移已弃用的 `node.json` 不会重置配对。有关
受支持的撤销并重新配对流程和升级说明，请参阅
[身份和配对状态](/zh-CN/cli/node#identity-and-pairing-state)。

已弃用的 `identity/device.json` 和 `identity/device-auth.json` 文件是
由 Doctor 管理的迁移输入。停止节点主机并运行
`openclaw doctor --fix`；Doctor 会先将其行导入 SQLite 并进行验证，
然后再移除旧文件。

### 将命令加入允许列表

Exec 审批是**按节点主机**进行的。从 Gateway 网关添加允许列表条目：

```bash
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/uname"
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/sw_vers"
```

审批信息存储在节点主机的 `~/.openclaw/exec-approvals.json` 中。

### 将 Exec 指向节点

配置默认值（Gateway 配置）：

```bash
openclaw config set tools.exec.host node
openclaw config set tools.exec.mode allowlist
openclaw config set tools.exec.node "<id-or-name>"
```

或按会话配置：

```text
/exec host=node security=allowlist node=<id-or-name>
```

设置后，任何带有 `host=node` 的 `exec` 调用都会在节点主机上运行（受节点允许列表/审批约束）。

`host=auto` 不会自行隐式选择节点，但允许从 `auto` 发出显式的单次调用 `host=node` 请求。如果要将节点 Exec 设为会话默认值，请显式设置 `tools.exec.host=node` 或 `/exec host=node ...`。

相关内容：

- [节点主机 CLI](/zh-CN/cli/node)
- [Exec 工具](/zh-CN/tools/exec)
- [Exec 审批](/zh-CN/tools/exec-approvals)

### 本地模型推理

桌面或服务器节点可以公开该节点上运行的 Ollama 服务器中的聊天模型。智能体使用 Ollama 插件的 `node_inference` 工具发现已安装的模型，并远程运行有界提示词；Gateway 网关无需直接通过网络访问 Ollama。有关设置、模型筛选和直接验证命令，请参阅 [Ollama 节点本地推理](/zh-CN/providers/ollama#node-local-inference)。

### Codex 会话和转录记录

官方 `codex` 插件可以公开无头节点主机或原生 macOS 节点上
未归档的 Codex 会话。目录注册不再依赖
`supervision.enabled`；该选项用于控制面向智能体的监督工具。
在 Codex 插件配置中设置 `sessionCatalog.enabled: false`，可以禁用
操作员目录和已配对节点目录命令，而不禁用
提供商或 harness。
该插件仍必须在两台计算机上均处于启用状态，并且节点设置仍表示
本地同意：仅在 Gateway 网关上启用并不能读取另一台计算机的 Codex
状态。

节点会公布带版本的只读
`codex.appServer.threads.list.v1` 和
`codex.appServer.thread.turns.list.v1` 命令。安装了
Codex CLI 的原生节点主机还会公布 `codex.terminal.resume.v1`。这些命令首次出现时，请批准节点配对
升级。Gateway 网关会通过常规插件节点策略调用它们，并按主机隔离故障。

已配对节点的行会在常规会话侧边栏中显示为 **Codex** 组。
默认情况下，每台主机中的行按项目文件夹分组；位于
`.claude/worktrees/<name>` 下的工作目录会归入其源代码仓库，项目
组可像其他侧边栏分区一样折叠。使用目录标题中的文件夹图标
展平或恢复项目组。同样的分组方式也适用于
Claude 会话目录。
默认情况下，选择一行会打开常规聊天窗格，并通过有界、基于游标分页且采用完整条目投影的
`thread/turns/list` 调用读取其持久化转录记录。使用行菜单、查看器标题或 **Open Codex/Claude sessions in** 偏好设置，可在会话所属计算机的操作员终端中启动 `codex resume <thread-id>`。已配对节点的终端路径是由 Codex 插件管理的允许列表 PTY 中继，而非任意节点命令执行。

该中继不提供完整的 OpenClaw harness 继续执行和归档所有权契约。因此，远程行无法使用**继续**和**归档**。在 Gateway 网关计算机上，已存储且空闲的
行可以启动单独的模型锁定聊天分支。只有在
操作员确认没有其他 Codex 客户端正在使用后，才能归档其中任意一种；
已存储行的实时活动状态仍然未知。活动行无法创建分支或归档。

有关设置、分页、本地继续执行和元数据安全边界，请参阅
[监督 Codex 会话](/zh-CN/plugins/codex-supervision)。

### Claude 会话和转录记录

内置 `anthropic` 插件默认会发现 Gateway 网关和已配对节点上未归档的 Claude CLI 与 Claude
Desktop 会话。设置
`plugins.entries.anthropic.config.sessionCatalog.enabled: false` 可禁用
操作员目录和已配对节点目录命令，而不禁用 Anthropic
模型或 Claude CLI 后端。
启用了 Anthropic 插件且 `~/.claude/projects/` 存在时，
远程 macOS 应用节点会公布
`anthropic.claude.sessions.list.v1` 和 `anthropic.claude.sessions.read.v1`。
这些命令首次出现时，请批准节点配对升级。

安装了 Claude CLI 的原生节点主机还会公布
`anthropic.claude.terminal.resume.v1`。符合条件的 CLI 和 Desktop 行可以在其所属主机的操作员终端中打开
`claude --resume <session-id>`。
这是对原生会话的接管；与 OpenClaw 接管不同，它不会
先为 Claude 会话创建分支。

该目录会将有效的 Claude CLI 项目索引记录与针对未索引 JSONL 转录记录的有界
元数据回退相结合。该回退可以识别
并发的非 sidechain 交互式（`cli`）和无头 Agent SDK CLI
（`sdk-cli`）会话。Claude Desktop 的本地元数据提供 Desktop 标题和归档
状态。当两个来源指向同一个 Claude Code
会话 ID 时，Desktop 元数据优先；仅 CLI 的转录记录仍然可见，因为 CLI 没有归档
标志。转录记录读取使用不透明的
字节偏移游标和有界的反向文件读取，因此选择大型
会话或加载更早的页面时，不会将整个 JSONL 历史记录读入单个
Gateway 网关响应。

列表和读取命令均为只读。它们仅通过通用的 `sessions.catalog.list` 和
`sessions.catalog.read` 方法，向具有
`operator.write` 的已验证操作员连接公开目录元数据和转录记录
内容。Gateway 网关本地的 Claude CLI 行可以从常规
聊天编辑器接管：OpenClaw 会导入有界的可见历史记录，在第一轮使用
`--fork-session` 恢复，并保持源转录记录不变。

无头节点主机可以选择启用相同的继续执行流程：

```json5
{
  nodeHost: {
    agentRuns: {
      claude: { enabled: true },
    },
  },
}
```

仅当此节点本地设置
已启用且 `claude` 可执行文件可在该节点上解析时，节点才会公布 `agent.cli.claude.run.v1`。Gateway 网关无法
远程启用它。该命令也会经过节点现有的 Exec
审批策略。当全部三个 Claude 命令均已公布，并且
Gateway 网关的节点命令策略允许它们时，该节点上的 Claude CLI
行即可继续执行：OpenClaw 会导入有界历史记录，将
接管的会话绑定到该节点及其目录报告的工作目录，并
在该处运行每个单次 `claude -p` 轮次。第一轮仍使用
`--fork-session`，从而保持源转录记录不变。

放置在节点上的轮次使用节点的 Claude 默认值。在 v1 中，它们不会接收
Gateway 网关 local loopback MCP 配置或 Gateway 网关 Skills 插件，无法从
Gateway 网关转录记录重新植入上下文，并且会拒绝附件和图像。Claude Desktop 行以及
未公布运行命令的节点仍仅供查看。macOS 应用
节点目前尚未公布此命令，因此其行仍仅供查看。

有关 Control UI 行为和存储来源，请参阅
[Anthropic：跨计算机的 Claude 会话](/zh-CN/providers/anthropic#claude-sessions-across-computers)。

### OpenCode 和 Pi 会话

内置 OpenCode 和 ACPX 插件也会发现 Gateway 网关和已配对节点上的只读原生会话
目录。安装了 `opencode`
CLI 时，节点会公布 `opencode.sessions.list.v1` / `opencode.sessions.read.v1`；
存在 Pi 会话目录时，则会公布 `acpx.pi.sessions.list.v1` / `acpx.pi.sessions.read.v1`。
新命令首次出现时，请批准节点配对升级。如果相应 CLI 也可用，节点会添加
`opencode.terminal.resume.v1` 或 `acpx.pi.terminal.resume.v1`；随后可以使用现有的行
菜单和查看器标题，通过 `opencode --session <id>` 或 `pi --session <id>`
在所属终端中重新打开所选会话。

OpenCode 通过其官方 CLI JSON/导出接口读取。Pi 读取其
文档中说明的 JSONL 会话存储，包括项目和全局 `settings.json`
会话目录，以及 `PI_CODING_AGENT_DIR` 和
`PI_CODING_AGENT_SESSION_DIR` 覆盖项。两个目录默认均已启用；
可在 Web UI 的 **Config > Plugins** 下将其关闭。

终端恢复使用已存储的会话工作目录，以及与 Codex 和 Claude 相同的
允许列表双工 PTY 中继。它不会公开任意
节点命令执行。

### 终端文件上传

Control UI 可以将文件拖入已打开的配对节点终端。原生节点主机会公布仅限管理员使用的 `terminal.upload` 命令；首次出现时，请批准配对升级。每个文件限制为 16 MiB，会暂存到该节点上的私有临时目录中，并以经 shell 引用的路径形式返回终端，而不会执行该文件。

路径插入支持 PowerShell、`cmd.exe` 和可识别的 POSIX shell（`sh`、Bash、Dash、Ash、Ksh、Zsh 和 Fish），包括 Windows 上的 Git Bash。其他 shell 覆盖会被拒绝，因为无法安全推断其引用规则；如需原生 WSL 路径，请在 WSL 内运行节点主机。包含 `%` 或 `!` 的 `cmd.exe` 路径也会被拒绝，因为该 shell 即使在双引号内也会展开这些字符。

## 调用命令

底层调用（原始 RPC）：

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command canvas.eval --params '{"javaScript":"location.href"}'
```

`nodes invoke` 会阻止 `system.run` 和 `system.run.prepare`；这些命令只能通过带有 `host=node` 的 `exec` 工具运行（见上文）。针对常见的“向智能体提供 MEDIA 附件”工作流，提供了更高层级的辅助命令（Canvas、相机、屏幕、位置，见下文）。

长时间运行的流式节点命令使用附加的 `node.invoke.progress`
事件。每个事件都包含调用 ID、从零开始的序列号和一个
大小受限的 UTF-8 文本块；Gateway 网关会对文本块排序，然后再将其传递给
调用方。现有的 `node.invoke.result` 仍然是唯一的终结
响应。流式调用方可以设置非活动截止时间，该计时从
第一个进度事件开始，并在后续进度到达时重置，同时在审批和执行期间保留
该调用单独的硬超时。结果、硬
超时、非活动超时和节点断开连接都会丢弃待处理的流
状态。调用方取消会发出 `node.invoke.cancel`；随后节点主机
会终止匹配的进程树。现有的请求/响应命令保持不变。

## 命令策略

节点命令必须通过两个关卡才能调用：

1. 节点必须在其经过身份验证的连接元数据（`connect.commands`）中声明该命令。
2. Gateway 网关根据平台和审批派生的允许列表必须包含所声明的命令。

各平台的默认允许列表（应用插件默认值以及 `commands.allow`/`commands.deny` 覆盖之前）：

| 平台 | 默认允许的命令                                                                                                                                                                                                                                                                                           |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| iOS      | `camera.list`, `location.get`, `device.info`, `device.status`, `contacts.search`, `calendar.events`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer`, `system.notify`                                                                                                                        |
| watchOS  | `device.info`, `device.status`, `system.notify`                                                                                                                                                                                                                                                                       |
| Android  | `camera.list`, `location.get`, `notifications.list`, `notifications.actions`, `system.notify`, `device.info`, `device.status`, `device.permissions`, `device.health`, `device.apps`, `contacts.search`, `calendar.events`, `callLog.search`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer` |
| macOS    | `camera.list`, `location.get`, `device.info`, `device.status`, `device.apps`, `contacts.search`, `calendar.events`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer`, `system.notify`                                                                                                         |
| Windows  | `camera.list`, `location.get`, `device.info`, `device.status`, `system.notify`                                                                                                                                                                                                                                        |
| Linux    | `system.notify`（节点主机命令（如 `system.run`）受审批限制，见下文）                                                                                                                                                                                                                                  |

这些行描述的是 Gateway 网关策略的上限，而不是每个节点应用都已实现的命令。只有连接的节点也声明了某个命令，该命令才可用。特别是，当前的 macOS 应用不会声明 macOS 策略行中列出的设备和个人数据命令族。

`canvas.*` 命令（`canvas.present`、`canvas.hide`、`canvas.navigate`、`canvas.eval`、`canvas.snapshot`、`canvas.a2ui.*`）是 iOS、Android、macOS、Windows、Linux 和未知平台上的插件默认命令。Linux 节点仅在桌面应用的本地 Canvas 套接字存在时声明这些命令。在 iOS 上，所有 Canvas 命令都仅限前台运行。

对于任何公布 `talk` 能力或声明 `talk.*` 命令的节点，无论平台标签为何，默认均允许 `talk.ptt.start`、`talk.ptt.stop`、`talk.ptt.cancel` 和 `talk.ptt.once`。

桌面主机命令（macOS/Windows/Linux 上的 `system.run`、`system.run.prepare`、`system.which`、`browser.proxy`、`mcp.tools.call.v1` 和 `screen.snapshot`）不属于上面的静态平台默认表。操作员批准声明这些命令的配对请求后，这些命令才会变得可用；此后，节点已获批准的命令集会在重新连接时继续保留这些命令。

危险或高度涉及隐私的命令即使已由节点声明，仍然需要通过 `gateway.nodes.commands.allow` 显式选择启用：`camera.snap`、`camera.clip`、`screen.record`、`computer.act`、`contacts.add`、`calendar.add`、`reminders.add`、`health.summary`、`sms.send`、`sms.search`。`gateway.nodes.commands.deny` 的优先级始终高于默认值和额外的允许列表条目。有关 iPhone 的同意关卡，请参阅 [HealthKit 摘要](/zh-CN/platforms/ios-healthkit)；有关桌面输入的额外能力、工具策略、启用和平台执行器关卡，请参阅[计算机使用](/zh-CN/nodes/computer-use)。

插件拥有的节点命令可以添加 Gateway 网关节点调用策略。该策略在允许列表检查之后、转发到节点之前运行，因此原始 `node.invoke`、CLI 辅助命令和专用智能体工具共用同一插件权限边界。危险的插件节点命令仍需通过 `gateway.nodes.commands.allow` 显式选择启用。

节点更改其声明的命令列表后，请拒绝旧的设备配对并批准新请求，以便 Gateway 网关存储更新后的命令快照。

## 配置（`openclaw.json`）

节点相关设置位于 `gateway.nodes` 和 `tools.exec` 下：

```json5
{
  gateway: {
    nodes: {
      // 自动批准来自可信网络（CIDR 列表）的首次节点配对。
      // 未设置时禁用。仅适用于未请求权限范围的首次 role:node 请求；
      // 不会自动批准升级。
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
        // 通过 SSH 验证的自动批准（默认：启用）。当通过 SSH 回读的
        // 设备密钥完全匹配时，批准首次节点配对。
        sshVerify: true,
      },
      // 信任已配对节点发布的智能体可见插件工具（默认：true）。
      pluginTools: {
        enabled: true,
      },
      // 选择启用危险或高度涉及隐私的节点命令（camera.snap 等）。
      commands: {
        allow: ["camera.snap", "screen.record"],
        // 即使默认值或 commands.allow 包含命令，也阻止名称完全匹配的命令。
        deny: ["camera.clip"],
      },
    },
  },
  tools: {
    exec: {
      // 默认 Exec 主机："node" 会将所有 Exec 调用路由到已配对节点。
      host: "node",
      // 节点 Exec 的安全模式：仅允许已批准或已加入允许列表的命令。
      security: "allowlist",
      // 将 Exec 固定到特定节点（ID 或名称）。省略则允许任意节点。
      node: "build-node",
    },
  },
}
```

请使用准确的节点命令名称。即使平台默认值或 `commands.allow` 条目原本会允许某个命令，`commands.deny` 也会将其移除。默认情况下，已配对节点可以发布智能体可见的插件工具描述符，但每个描述符的命令仍必须位于节点已批准的命令范围内。设置 `gateway.nodes.pluginTools.enabled: false` 可忽略所有此类描述符。有关 Gateway 网关节点配对和命令策略字段的详细信息，请参阅 [Gateway 配置参考](/zh-CN/gateway/configuration-reference#gateway)。

按智能体覆盖 Exec 节点：

```json5
{
  agents: {
    list: [
      {
        id: "main",
        tools: { exec: { node: "build-node" } },
      },
    ],
  },
}
```

## 屏幕截图（Canvas 快照）

如果节点正在显示 Canvas（WebView），`canvas.snapshot` 会返回 `{ format, base64 }`。

CLI 辅助命令（写入临时文件并输出保存路径）：

```bash
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format png
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format jpg --max-width 1200 --quality 0.9
```

### Canvas 控制

```bash
openclaw nodes canvas present --node <idOrNameOrIp> --target https://example.com
openclaw nodes canvas hide --node <idOrNameOrIp>
openclaw nodes canvas navigate https://example.com --node <idOrNameOrIp>
openclaw nodes canvas eval --node <idOrNameOrIp> --js "document.title"
```

注意：

- `canvas present` 在支持本地路径的节点上接受 URL 或本地文件路径（`--target`），还可使用可选的 `--x/--y/--width/--height` 进行定位。Linux Canvas 接受 HTTP(S) URL 或其内置的 A2UI 渲染器。
- `canvas eval` 接受内联 JS（`--js`）或位置参数。

### A2UI（Canvas）

```bash
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --text "Hello"
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --jsonl ./payload.jsonl
openclaw nodes canvas a2ui reset --node <idOrNameOrIp>
```

注意：

- 移动端和 Linux 桌面节点使用由应用拥有的内置 A2UI 页面进行支持操作的渲染。
- 仅支持 A2UI v0.8 JSONL（会拒绝 v0.9/createSurface）。
- iOS 和 Android 会渲染远程 Gateway 网关 Canvas 页面，但 A2UI 按钮操作仅从由应用拥有的内置 A2UI 页面分派。在这些移动客户端上，由 Gateway 网关托管的 HTTP/HTTPS A2UI 页面仅供渲染。
- macOS 可以从应用选择的、能力范围完全匹配的 Gateway 网关 A2UI 页面分派操作。其他 HTTP/HTTPS 页面仍仅供渲染。
- Linux 仅从内置 A2UI 页面分派操作。其他 HTTP/HTTPS 页面仍仅供渲染，并且未安装桌面应用的无头 Linux 节点不会公布 Canvas。

## 照片和视频（节点相机）

照片（`jpg`）：

```bash
openclaw nodes camera list --node <idOrNameOrIp>
openclaw nodes camera snap --node <idOrNameOrIp>            # 默认：前后摄像头（2 行 MEDIA）
openclaw nodes camera snap --node <idOrNameOrIp> --facing front
openclaw nodes camera snap --node <idOrNameOrIp> --device-id <id> --max-width 1200 --quality 0.9 --delay-ms 2000
```

视频片段（`mp4`）：

```bash
openclaw nodes camera clip --node <idOrNameOrIp> --duration 10s
openclaw nodes camera clip --node <idOrNameOrIp> --duration 3000 --no-audio
```

注意事项：

- 使用 `canvas.*` 和 `camera.*` 时，节点必须处于**前台**（后台调用会返回 `NODE_BACKGROUND_UNAVAILABLE`）。
- 节点会限制视频片段时长，以确保 base64 载荷易于处理（各平台的确切限制请参阅[摄像头捕获](/zh-CN/nodes/camera)）。`nodes` 智能体工具还会在转发调用前，将请求的 `durationMs` 上限设为 300000（5 分钟）；节点本身会执行更严格的限制。
- Android 会尽可能提示授予 `CAMERA`/`RECORD_AUDIO` 权限；如果权限被拒绝，则调用会失败并返回 `*_PERMISSION_REQUIRED`。

## 屏幕录制（节点）

受支持的节点会公开 `screen.record`（mp4）。示例：

```bash
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10 --no-audio
```

注意事项：

- `screen.record` 是否可用取决于节点平台。
- `nodes` 智能体工具会将请求的 `durationMs` 上限设为 300000（5 分钟）；节点可能会实施更严格的限制，以约束返回载荷的大小。
- `--no-audio` 会在受支持的平台上禁用麦克风捕获。
- 当有多个屏幕可用时，使用 `--screen <index>` 选择显示器（0 = 主显示器）。

## 位置（节点）

在设置中启用“位置”后，节点会公开 `location.get`。

CLI 辅助命令：

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

注意事项：

- 位置功能**默认关闭**。
- “Always”需要系统权限；后台获取仅尽力而为。
- 响应包含纬度/经度、精度（米）和时间戳。
- 完整的参数/响应结构和错误代码：[位置命令](/zh-CN/nodes/location-command)。

## SMS（Android 节点）

当用户授予 **SMS** 权限且设备支持电话功能时，Android 节点可以公开 `sms.send` 和 `sms.search`。这两个命令默认都被视为危险命令：Gateway 网关操作员还必须将它们添加到 `gateway.nodes.commands.allow`，之后才能调用（请参阅[命令策略](#command-policy)）。

对于只读 SMS 搜索，请在 `openclaw.json` 中明确选择启用：

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["sms.search"] },
    },
  },
}
```

仅当节点还应能够发送消息时，才单独添加 `sms.send`。Android 权限与 Gateway 网关命令授权彼此独立；授予手机权限不会修改 Gateway 网关策略。

底层调用：

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command sms.send --params '{"to":"+15555550123","message":"来自 OpenClaw 的问候"}'
```

注意事项：

- 可以在授予 `READ_SMS` 之前声明 `sms.search`，这样调用便可返回权限诊断信息；读取消息仍需要该 Android 权限。
- 不具备电话功能的纯 Wi-Fi 设备不会通告 `sms.send`。
- `requires explicit gateway.nodes.commands.allow opt-in` 错误表示手机已声明该命令，但 Gateway 网关操作员尚未授权。

## 设备和个人数据命令

iOS 和 Android 节点默认通告若干只读数据命令（请参阅[命令策略](#command-policy)表）；Android 还会公开一组范围更大的命令，并由其应用内设置控制。仅当操作员通过 `--share-installed-apps` 启用已安装应用共享后，macOS 或无头 Mac TypeScript 节点主机才会通告 `device.apps`。

可用命令系列：

- `device.status`、`device.info` — iOS、Android、Windows。
- `device.permissions`、`device.health` — 仅限 Android。
- `device.apps` — Android、macOS 和无头 Mac 节点。Android 需要在 Settings 中启用 Installed Apps 共享，并默认返回启动器中可见的应用。TypeScript 节点主机默认关闭共享，并接受 `query`、`limit` 和 `includeSystem`；macOS 结果包含 `label`、`bundleId`、`path` 和 `system`。
- `notifications.list`、`notifications.actions` — 仅限 Android。
- `photos.latest` — iOS、Android。
- `contacts.search` — iOS、Android（默认为只读）；`contacts.add` 属于危险命令，需要 `gateway.nodes.commands.allow`。
- `calendar.events` — iOS、Android（默认为只读）；`calendar.add` 属于危险命令，需要 `gateway.nodes.commands.allow`。
- `reminders.list` — iOS、Android（默认为只读）；`reminders.add` 属于危险命令，需要 `gateway.nodes.commands.allow`。
- `callLog.search` — 仅限 Android。
- `motion.activity`、`motion.pedometer` — iOS、Android；是否可用取决于可用的传感器能力。

调用示例：

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command device.status --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command device.apps --params '{"limit":10}'
openclaw nodes invoke --node <idOrNameOrIp> --command notifications.list --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command photos.latest --params '{"limit":1}'
```

## 系统命令（节点主机/Mac 节点）

macOS 节点公开 `system.run`、`system.which`、`system.notify` 和 `system.execApprovals.get/set`。无头节点主机公开 `system.run.prepare`、`system.run`、`system.which` 和 `system.execApprovals.get/set`。

示例：

```bash
openclaw nodes notify --node <idOrNameOrIp> --title "连通性测试" --body "Gateway 网关已就绪"
openclaw nodes invoke --node <idOrNameOrIp> --command system.which --params '{"bins":["git"]}'
```

注意事项：

- `system.run` 会在载荷中返回标准输出、标准错误和退出代码。
- Shell 执行现在通过带有 `host=node` 的 `exec` 工具进行；`nodes` 仍是显式节点命令的直接 RPC 接口。
- `nodes invoke` 不公开 `system.run` 或 `system.run.prepare`；这些功能仍仅保留在 exec 路径上。
- exec 路径会在审批前准备规范的 `systemRunPlan`。审批获准后，Gateway 网关会转发该已存储的计划，而不是调用方之后编辑过的命令、cwd 或会话字段。
- `system.notify` 遵循 macOS 应用中的通知权限状态；支持 `--priority <passive|active|timeSensitive>` 和 `--delivery <system|overlay|auto>`。
- 无法识别的节点 `platform`/`deviceFamily` 元数据会使用保守的默认允许列表，其中不包含 `system.run` 和 `system.which`。如果确实需要在未知平台上使用这些命令，请通过 `gateway.nodes.commands.allow` 明确添加。
- `system.run` 支持 `--cwd`、`--env KEY=VAL`、`--command-timeout` 和 `--needs-screen-recording`。
- 对于 Shell 包装器（`bash|sh|zsh ... -c/-lc`），请求范围的 `--env` 值会缩减为明确的允许列表（`TERM`、`LANG`、`LC_*`、`COLORTERM`、`NO_COLOR`、`FORCE_COLOR`）。
- 对于允许列表模式下的始终允许决定，已知的分派包装器（`env`、`flock`、`nice`、`nohup`、`stdbuf`、`timeout`）会持久化内部可执行文件路径，而不是包装器路径。如果无法安全地解包，则不会自动持久化任何允许列表条目。
- 在允许列表模式下的 Windows 节点主机上，通过 `cmd.exe /c` 运行 Shell 包装器需要审批（仅有允许列表条目不会自动允许包装器形式）。
- 节点主机会忽略 `--env` 中的 `PATH` 覆盖，并在运行命令前移除一组数量庞大且持续维护的解释器/Shell 启动变量（例如 `NODE_OPTIONS`、`PYTHONPATH`、`BASH_ENV`、`DYLD_*`、`LD_*`）。如果需要额外的 PATH 条目，请配置节点主机服务环境（或将工具安装到标准位置），而不要通过 `--env` 传递 `PATH`。
- 在 macOS 节点模式下，`system.run` 受 macOS 应用中的 Exec 审批控制（Settings → Exec approvals）。询问/允许列表/完全模式的行为与无头节点主机相同；被拒绝的提示会返回 `SYSTEM_RUN_DENIED`。
- 在无头节点主机上，`system.run` 受 Exec 审批（`~/.openclaw/exec-approvals.json`）控制；特别是在 macOS 上，请参阅下方[无头节点主机](#headless-node-host-cross-platform)中的 exec 主机路由环境变量。

## Exec 节点绑定

当有多个节点可用时，可以将 exec 绑定到特定节点。这会为 `exec host=node` 设置默认节点（并且可以按智能体覆盖）。

全局默认值：

```bash
openclaw config set tools.exec.node "node-id-or-name"
```

按智能体覆盖：

```bash
openclaw config get agents.entries
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
```

取消设置以允许使用任意节点：

```bash
openclaw config unset tools.exec.node
openclaw config unset 'agents.entries.main.tools.exec.node'
```

## 权限映射

节点可以在 `node.list`/`node.describe` 中包含 `permissions` 映射，以权限名称（例如 `screenRecording`、`accessibility`、`location`）作为键，值为布尔值（`true` = 已授予）。

## 无头节点主机（跨平台）

OpenClaw 可以运行连接到 Gateway 网关 WebSocket 并公开 `system.run`/`system.which` 的**无头节点主机**（无 UI）。这适用于 Linux/Windows，或在服务器旁运行精简节点。

启动命令：

```bash
openclaw node run --host <gateway-host> --port 18789
```

注意事项：

- 仍然需要配对（Gateway 网关会显示设备配对提示）。
- 客户端实例元数据、已签名的设备身份和配对身份验证使用独立的状态记录；请参阅[无头身份状态](#headless-identity-state)。
- Exec 审批通过 `~/.openclaw/exec-approvals.json` 在本地执行（请参阅 [Exec 审批](/zh-CN/tools/exec-approvals)）。
- 在 macOS 上，无头节点主机默认在本地执行 `system.run`。设置 `OPENCLAW_NODE_EXEC_HOST=app` 可通过配套应用的 exec 主机路由 `system.run`；添加 `OPENCLAW_NODE_EXEC_FALLBACK=0` 可要求必须使用应用主机，并在其不可用时以失败关闭。
- 当 Gateway 网关 WS 使用 TLS 时，请添加 `--tls`/`--tls-fingerprint`。

## Mac 节点模式

- macOS 菜单栏应用会以节点身份连接到 Gateway 网关 WS 服务器（因此 `openclaw nodes …` 可以在这台 Mac 上使用）。
- 在远程模式下，应用会为 Gateway 网关端口打开 SSH 隧道，并连接到 `localhost`。
