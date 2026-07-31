---
read_when:
    - 设计 Codex 会话发现、续接或归档行为
    - 更改原生会话目录 UI 或 Gateway 网关 RPCs
    - 将 Codex 监督扩展到已配对节点
summary: 从 OpenClaw 监管原生 Codex 会话的架构与产品边界。
title: Codex 监督
x-i18n:
    generated_at: "2026-07-26T07:00:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5e259badc8f7fdec6fa093785a1dd04394e12287ae61f00474bcd45e7b95352d
    source_path: specs/codex-supervision.md
    workflow: 16
---

# Codex 监管

## 目标

Codex 监管使 OpenClaw 操作员能够发现原生 Codex 会话，并在安全时通过常规 OpenClaw Chat 界面创建本地分支。
Codex App Server 仍是线程和模型循环的所有者。OpenClaw 提供实例目录、经身份验证的操作员 UI、会话绑定和渠道交付。

此功能属于官方 `codex` 插件。不存在单独的 Supervisor 插件或第二套 Codex 协议实现。

## 产品边界

只要 Codex 插件处于活动状态，目录就会注册，除非使用以下配置明确禁用原生会话发现：

```text
plugins.entries.codex.config.sessionCatalog.enabled = false
```

使用以下配置启用面向智能体的监管工具：

```text
plugins.entries.codex.config.supervision.enabled = true
```

当前启用的初始产品有意小于长期实例计划：

- 仅列出未归档的 Codex 线程。
- 按稳定的主机标识对本地行和已选择加入的配对节点行进行分组。
- 从已存储或空闲的 Gateway 网关本地线程创建一个普通的、锁定模型的 Chat 分支，在第一次轮次时启动其完整的 Codex harness 线程，或打开为先前分支创建的 Chat。
- 仅在明确确认没有其他运行方后，归档已存储或空闲的 Gateway 网关本地线程。
- 显示活动的本地来源，不提供新建分支或归档控件，但仍允许打开现有的受监管 Chat。
- 在主侧边栏中显示每台主机的最新行，在会话页面上保留完整目录，并为本地和配对节点行提供有界的、基于游标分页的记录读取。
- 按主机隔离目录故障。

目录是未归档条目的集合。其中的某一行仍可具有空闲、活动、`notLoaded` 或错误轮次状态。

面向智能体的监管仍需选择加入。引导式新手引导会在成功检测到原生 Codex 安装，并且所选推理后端通过实时检查后，尝试安装并启用该功能，而不受用户选择哪个主要后端的影响。仅当这一机会性插件设置成功时，监管才会激活。明确禁用的插件、策略阻止或 `supervision.enabled: false` 对监管工具仍具有最高效力，但不会禁用操作员会话目录。`sessionCatalog.enabled: false` 会禁用操作员发现和配对节点目录命令；Codex 提供商和 harness 仍保持活动状态。

## 所有权

`codex` 插件拥有所有 Codex App Server 行为：

- 端点发现和连接生命周期
- 协议初始化和版本检查
- 线程列表、读取、恢复、归档和事件处理
- 审批和用户输入桥接
- 原生线程与 OpenClaw 会话的绑定
- 继续执行后的 Codex 专属模型和 harness 强制约束

Control UI 和 Gateway 网关使用该插件拥有的服务。它们不会直接读取 Codex rollout 文件，也不会实现另一个 App Server 客户端。

默认本地拓扑为：

```text
Codex Desktop -> 私有 stdio App Server -> 用户 Codex 主目录
                                             ^
OpenClaw Codex 插件 -> 监管 App Server 连接
  （默认为托管式用户主目录 stdio；遵循显式 appServer 设置）
  -> 被动来源目录和读取
  -> 快照固定 -> 规范 appServer 来源分支
  -> 可见历史注入以及之后每个受监管的 Chat 轮次

普通 OpenClaw Codex 会话 -> 默认使用托管式智能体主目录 stdio
  -> 普通完整 harness 线程 -> OpenClaw Chat 和渠道交付
```

启用监管不会改变普通 Codex harness：默认情况下，它仍限定于智能体范围。单独的监管连接默认为托管式用户主目录 stdio，因此其目录和快照操作能够看到原生存储的线程。遵循显式 `appServer` 连接设置。未设置 `homeScope` 时，监管连接会将其解析为 stdio 或 Unix 的 `"user"`，或 WebSocket 的 `"agent"`。仅当普通 harness 也应共享原生 Codex 主目录时，才显式设置 `appServer.homeScope: "user"`。从 Codex 侧边栏组接管的 Chat 是例外：其私有监管绑定会让来源读取、规范分支创建和后续轮次继续使用监管连接。实时状态和所有权仍限定于进程本地；对于 OpenClaw 监管进程未知的线程，即使 Codex Desktop 正在主动运行它，该线程也是 `notLoaded`。

Codex 有一个实验性的规范本地守护进程，具有独立的安装程序托管引导契约。此功能不得隐式引导、声明使用或假定该守护进程存在。

## 目录流程

通用 Gateway 网关方法 `sessions.catalog.list` 分派到 `codex` 目录提供商，该提供商始终请求 `archived: false`，并让 App Server 应用其交互式来源默认值：`cli`、`vscode`、Atlas 和 ChatGPT。它会合并：

1. 来自监管 App Server 的 Gateway 网关本地 `thread/list` 结果；该服务器默认使用托管式用户主目录 stdio。
2. 来自每个已连接且已选择加入的节点的 `codex.appServer.threads.list.v1` 结果。

记录选择在本地使用带 `itemsView: "full"` 的 `thread/turns/list`，或在所选节点上使用有版本控制的 `codex.appServer.thread.turns.list.v1` 命令。每个响应最多包含 20 个持久化轮次以及不透明的向前/向后游标。Control UI 请求从最新开始的页面，按时间顺序渲染每一页，并将较旧页面添加到开头。它绝不会回退到无界的 `thread/read`。OpenClaw 还会在任何序列化项目页面跨越节点或 Gateway 网关传输之前，拒绝超过 20 MiB 的页面。

原生 macOS 配对节点实现仅支持未设置/默认值或显式 `appServer.transport: "stdio"`，其监管范围需为未设置/默认值或显式 `appServer.homeScope: "user"`。它会将已配置的 `command`、`args` 和规范化的 `clearEnv` 传递给子进程。使用 `"unix"`、`"websocket"` 或显式 `homeScope: "agent"` 时，它既不公布目录能力，也不公布相应命令；直接调用同样会以关闭状态失败。它绝不能为限定于智能体范围的配置公开用户 Codex 主目录，也不能用本地 stdio 替代显式端点。

目录投影会规范化标识符、标题、cwd、状态、活动等待标志、时间戳、来源、模型提供商、Codex 版本和 Git 分支。它不会返回记录预览、轮次、rollout 路径、Codex 主目录路径、Git 远程仓库、提交 SHA、原始端点或原始 App Server 错误。记录响应仅包含明确请求的 App Server 项目页面及其不透明游标。

主机故障仅影响各自的主机结果。离线节点或不可用的本地 App Server 不会从页面中移除健康的主机。连接状态是主机属性，而不是线程状态：失败的主机结果不包含新的会话行，也不会将 `offline` 投影到原生线程上。

Control UI 请求渐进式目录更新。每个本地或配对主机会在其自身 App Server 列表请求完成时出现；聚合响应仍作为兼容性和恢复快照。可见页面会在连接状态变化后、获得焦点时以及最多每 30 秒进行协调，发生变化后还会更快执行一次。因此，在其他客户端中创建的原生 Codex 会话最终会被发现，而无需将其导入 OpenClaw 存储。

目录发现是被动的。列出或读取元数据不得调用 `thread/resume`、让 OpenClaw 客户端订阅实时线程请求或回应审批。

搜索仅按标题进行，且不区分大小写。对于返回的每个目录页面，Gateway 网关和配对 Mac 会扫描数量有界的原生页面，而不会将查询传递给 App Server，因为原生搜索还可能匹配记录预览。返回的原生游标允许调用方继续扫描。

## 操作员 CLI 边界

该插件注册三个由 Gateway 网关支持的 shell 命令：

```text
openclaw codex sessions [--search <text>] [--host <id>] [--limit <count>] [--cursor <cursor>] [--json] [gateway-options]
openclaw codex continue <thread-id> [--json] [gateway-options]
openclaw codex archive <thread-id> --confirm-no-other-runner [--json] [gateway-options]
```

`[gateway-options]` 是 `--url <url>`、`--token <token>`、`--timeout <ms>` 以及继承的 `--expect-final` 开关。会话列出的默认超时时间为 75,000 ms；继续和归档的默认超时时间为 30,000 ms；`--expect-final` 对这些一元 RPC 没有额外效果。会话搜索仅按标题进行，且不区分大小写；每个响应都会扫描有界的原生页面链，`--cursor` 用于继续获取更早的结果。每台主机的限制默认为 50，可接受 1 到 100；使用游标时必须指定一个稳定的 `--host` 目标。任何命令都不接受 archived/include-archived 选项。只有 `sessions` 可以指向配对主机；`continue` 和 `archive` 始终发送 `hostId: "gateway:local"`，而归档需要显式确认标志。

shell 命名空间并非 Chat 内的 `/codex` 运行时命名空间。具体而言，`/codex sessions --host <node>` 会列出一个节点上的 Codex CLI 会话文件，`/codex threads` 会列出当前对话连接的 App Server 线程，而 `/codex resume` 或 `/codex bind` 会修改该对话的绑定。这些命令不能替代 `sessions.catalog.continue`，并且不存在 `/codex continue` 或 `/codex archive` 运行时命令。

## 本地继续

对于已存储或空闲的 Gateway 网关本地行，UI 会使用 `catalogId: "codex"` 加上主机和线程 ID 调用 `sessions.catalog.continue`。插件会：

1. 当来源已有受监管 Chat 时，复用现有 Chat。
2. 否则，将截至来源最后一个终止持久化轮次（已完成、已中断或失败）的有界用户和助手历史投影到新的 OpenClaw Chat 中，并记录待处理的 harness 分支。
3. 存储待处理的 Codex 专属模型锁定策略，而不是具体的模型或提供商选择，同时存储私有监管连接范围，并返回 OpenClaw `sessionKey`。

历史投影会选择可见用户和助手消息的最新尾部，硬性限制为 200 条消息、UTF-8 文本总计 512 KiB，以及每条消息 64 KiB。它会将图像和本地图像输入替换为 `[Image attachment]`，绝不复制图像载荷或路径，并省略推理、工具调用和工具结果。

UI 使用该会话键导航到普通 Chat。此时尚不存在规范 harness 线程。在第一个普通 Chat 轮次中，harness 会安装真正的 Codex 审批、信息获取、事件和交付处理程序，然后：

1. 使用监管连接调用原生 `thread/fork`，且不覆盖模型或提供商，并固定持久化的来源快照。Codex 当前的 `ConfigManager` 状态会选择模型和提供商，而 fork 响应会报告实际组合。如果模型不同于来源中最后记录的模型，Codex 会发出其常规模型差异警告。
2. 在同一连接上，使用 `threadSource: "appServer"`、OpenClaw 的 cwd、策略、配置、环境、完整的 OpenClaw harness 工具界面，以及 fork 为此次初始启动返回的确切模型和提供商，启动规范的完整 Codex harness 线程。
3. 通过该连接注入有界的可见用户和助手历史，在不丢弃其监管范围的情况下提交规范绑定，运行该轮次，并归档临时 fork。

在首次轮次之前，Chat 是一个锁定的待处理分支，并带有可见的历史记录镜像；之后，每个模型轮次都通过监督连接上的规范 Codex harness 线程运行。该分支并非完整的原生 rollout 克隆：源推理、工具调用和工具结果会被有意省略。如果快照固定或规范线程创建失败，待处理分支仍可重试。如果发生绑定竞争、监督被禁用，或者监督连接不可用或不匹配，则会在轮次运行前以关闭方式失败，而不会回退到普通的 Agent 主目录 harness。

这保证由 Codex 负责选择，而不是保留源的历史模型。分叉返回的模型与提供商组合用于启动规范线程，而 Codex 会持久保存该线程的原生模型和提供商。后续恢复时会省略 OpenClaw 的模型和提供商覆盖，因此 Codex 会恢复持久保存的组合。如果单独的原生 Codex 控制更改了规范线程，OpenClaw 会接受该原生持久化选择。外层 OpenClaw 模型和回退链绝不会替代它。

对于受监督且模型锁定的 Chat，模型更改、会话删除以及会话重置/新建操作都会以关闭方式失败。修改 `/codex model <model>`、`/codex
bind`、`/codex resume`（包括节点 `--bind here`）以及 `/codex detach` 或
`/codex unbind` 也会以关闭方式失败，因为这些操作会替换或清除绑定。`/codex model` 查询以及 `/codex fast`、`/codex permissions` 和 `/codex
threads` 仍然可用。`codex_threads` Agent 工具无法附加新的分叉，也无法归档已绑定的原生线程。列表和仅元数据读取仍然可用；脚本字段需要 `supervision.allowRawTranscripts`，而重命名、取消归档、分离式分叉以及归档无关线程需要 `supervision.allowWriteControls`。这两个选项都不能替换锁定的绑定。
删除或重置 OpenClaw 条目原本会丢弃原生绑定，并在看似 Codex 的会话背后创建或允许一个通用线程。因此，即使模型锁定条目超过普通的存续时间、数量或磁盘预算限制，保留维护也会保留这些条目。禁用或卸载所属插件时，也会保留锁定状态和插件所有权标记。在重新启用同一插件之前，Chat 会一直不可用并以关闭方式失败；清理操作绝不会将其转换为普通模型会话。

此操作绝不会恢复或修改源。临时分叉会固定一个快照；它不是持久的延续线程。在首次轮次中启动一个独立的规范 harness 线程，可防止 OpenClaw 仅仅因为进程本地状态未发现 Desktop 所拥有的轮次，就成为竞争性的源写入方。可见历史记录镜像和固定快照可能会省略活动源中尚未完成的工作。原始 CLI、VS Code、Atlas 或 ChatGPT 源仍然有资格出现在原生目录和 OpenClaw 目录中。规范分支在监督存储中仍是原生 Codex 线程，但原生客户端可能会筛除其 `appServer` 源类型，因此 Codex Desktop 中的可见性并非契约。

## 归档行为

对于已存储或空闲的 Gateway 网关本地行，带有 `catalogId: "codex"` 的 `sessions.catalog.archive` 需要明确的 `confirmNoOtherRunner: true`，并会重新读取当前进程本地状态；仅当状态为 `idle` 或 `notLoaded` 时才会继续，调用原生 `thread/archive`，且仅在 Codex 接受该操作后才返回成功。随后，该行会从未归档目录中移除。

重新读取后若状态为活动或错误，则会拒绝归档。源中的正在初始化或待处理的受监督分支也会如此：必须由首次 Chat 轮次将其规范分支实体化，之后才能归档源。若精确目标存在已知的活动 OpenClaw 绑定所有者，或存在任何未归档的衍生后代，也会拒绝归档。OpenClaw 会对 Codex 的实验性 `thread/list ancestorThreadId` 关系进行分页，并在请求或响应错误、游标或线程循环以及安全限制耗尽时以关闭方式失败。原生归档可能会关闭已加载的父级及后代工作，因此归档不是中断的快捷方式。读取、后代枚举和归档调用并非原子操作。
独立客户端仍可能拥有或在本地显示为空闲或 `notLoaded` 的行上启动工作。在 Codex 提供条件归档或跨进程租约之前，“无其他运行方”确认会涵盖未知客户端及该竞争情况。禁止对已配对节点执行归档。

Codex 目录中没有已归档视图。在另一个经所有者授权的 Codex 界面中通过 `thread/unarchive` 恢复的线程，会再次有资格进入未归档目录。

## 活动线程安全

Codex 会在同一 App Server 的客户端之间串行处理某个线程的修改，但不会公开跨进程独占运行方租约或审批所有者租约。独立的 stdio App Server 可以追加到同一个 rollout，而每个服务器只能看到自身的内存状态。审批请求也可能发送给同一服务器的所有订阅者，并由第一个有效响应完成该请求。

因此：

- 被动目录客户端不会订阅或自动拒绝审批
- 当前报告为活动的行既不提供新分支，也不提供 Archive
- 未映射的源会成为可见历史记录分支，其规范 harness
  线程绝不会恢复源
- `notLoaded` 会显示为活动状态未知，且只有在知情确认无其他运行方后才能归档
- 本地归档需要该确认以及重新读取到 `idle` 或 `notLoaded`，
  同时承认读取与归档之间存在协议竞争

中断和多客户端交接属于未来的产品决策。显示活动行并不隐含支持这些功能。

## 已配对节点边界

节点调用当前仅支持请求/响应。它可以安全地返回有界的目录元数据和脚本轮次页面，但无法承载 Codex harness 运行所需的长生命周期事件流、审批请求、工具调用、取消和助手增量内容。

因此，节点契约支持列表和脚本轮次页面。远程行仍可读取，但无论是否空闲，**Continue** 和 **Archive** 均不可用。真正的远程延续需要节点侧运行方和流式传输桥接，以维护与本地 harness 相同的审批与绑定不变量。

## 权限

每台计算机都需要在本地选择启用。启用 Gateway 网关并不会授权另一个节点读取其 Codex 元数据。节点能力必须通过常规配对和命令策略审批。

设备群列表和脚本查看使用 `operator.write` Gateway 网关权限范围，因为它们会调用已配对节点。本地延续和归档属于经过身份验证的操作员操作，并且仍受主机和状态检查约束。

自主 Agent 和独立 MCP 访问是另一套机制。已发布的 `codex_endpoint_probe`、`codex_sessions_list`、`codex_session_read`、`codex_session_send` 和 `codex_session_interrupt` 工具契约仍归 `codex` 插件所有。启用监督后，原始 `codex_threads` 脚本读取和源自脚本的列表字段也需要 `supervision.allowRawTranscripts`；每次 `codex_threads` 分叉、重命名、归档或取消归档都需要 `supervision.allowWriteControls`。这两项策略默认均为禁用。

## 兼容性

`openclaw doctor --fix` 会迁移已发布的 `plugins.entries.codex-supervisor` 配置，包括端点、脚本/写入策略以及插件允许/拒绝引用，并将其迁移到 `plugins.entries.codex.config.supervision`。发生冲突时，显式的规范目标值优先。迁移后，运行时代码仅使用规范的 `codex` 插件形态。

官方插件仅保留五个 Supervisor 兼容性工具：
`codex_endpoint_probe`、`codex_sessions_list`、`codex_session_read`、`codex_session_send` 和 `codex_session_interrupt`。默认情况下，会话列表仅包含已加载项；不存在 `loaded_only` 参数。`include_stored: true` 会添加未归档的状态数据库行，每个端点受 `max_stored_sessions` 限制（默认值为 200，接受范围为 1 到 1,000）；已加载行不受该设置限制。源自脚本的字段和读取仍受 `allowRawTranscripts` 控制；发送和中断仍受 `allowWriteControls` 控制。

兼容性发送绝不会启动或恢复空闲线程。`mode: "start"` 始终会被拒绝；`"auto"` 和 `"steer"` 仅能引导可读取的活动轮次。中断同样要求存在可读取的活动轮次。空闲延续会路由到原生 Codex 目录，从而由完整 harness 负责审批、工具和绑定。独立的旧版 MCP 适配器会从官方插件解析这些相同工具，并且是唯一遵循所保留旧版策略环境变量的路径。

七月的目录 UI、Gateway 网关方法、节点能力和 CLI 注册尚未以旧插件 ID 发布。它们会直接移交给 `codex` 所有，而不会增加第二个运行时 facade。

## 未来工作

- 用于远程延续的节点侧流式运行方和事件桥接
- 用于并发客户端交接的显式运行方和审批所有者租约
- 在具备运行方所有权租约或等效隔离机制后支持远程归档
- 中断和更丰富的活动会话观察
- 在 Codex Desktop、CLI 和 OpenClaw 之间进行经审计的交接

浏览已归档内容不属于计划中的监督侧边栏功能。原生 Codex 界面仍是已归档线程的恢复路径。

## 验收测试

- 启用监管后，会列出未归档的本地会话。
- 已归档的会话绝不会出现在目录响应或 UI 中。
- 当另一台主机发生故障时，健康的主机仍保持可见；不可用的主机
  不会虚构离线会话状态，而是不返回任何新行。
- 已存储或空闲的本地行会创建一个带有仅限 Codex
  模型/运行时锁定的 Chat 镜像；首个轮次会固定一个临时快照并启动
  规范的完整 harness 线程，而重复执行 Continue 会打开现有 Chat。
- 首个轮次在快照分支上省略模型/提供商覆盖，并将
  规范启动固定到 Codex 返回的确切组合，即使 Codex 警告
  其当前模型与源最后记录的模型不同。
- 待处理和已提交的受监管绑定使用监管连接进行
  源访问、规范分支创建以及后续每个轮次；普通
  Codex 会话仍限定在智能体范围内。
- 后续恢复会省略 OpenClaw 模型/提供商覆盖，保留 Codex 的
  规范持久化选择，接受对该线程进行的独立原生更改，
  并且绝不会替换为外层 OpenClaw 模型或回退链。
- 禁用监管或丢失绑定/连接生命周期时会以关闭方式失败，
  而不会将 Chat 移至普通的智能体主目录 harness。
- 受监管且模型锁定的 Chat 在保护原生
  绑定期间无法删除。
- Chat 最多镜像 200 条用户和助手消息，总计 512 KiB，
  每条消息 64 KiB。图像会变为占位符；源推理、工具调用、
  工具结果、图像负载和本地路径不会被克隆。
- 分支流程绝不会恢复源线程。
- 原始源仍可出现在两个目录中。规范原生
  分支使用 `appServer` 源类型，且不保证会出现在
  Codex Desktop 中。
- 活跃的本地源无法创建分支或归档；现有的
  受监管 Chat 仍可打开。
- 活动状态未知的行无需确认即可创建分支；归档则需要
  明确确认没有其他运行器。
- 具有正在初始化或待处理受监管分支的源无法归档，
  直到首个 Chat 轮次具体化规范分支。
- 确切目标或任何未归档的已生成后代若存在已知的活跃绑定所有者，
  都会阻止归档；后代枚举失败时会以关闭方式失败，而
  显式确认仍需负责处理未知客户端以及
  从状态检查到归档之间的竞态条件。
- 经确认的已存储或空闲本地归档会在原生操作成功后移除该行。
- 已配对节点的行仍保持可见，但不提供 Continue 或 Archive。
- 被动列举绝不会订阅或响应线程审批。
- 旧版 Supervisor 配置会迁移到规范的 Codex 配置结构。
- 默认情况下，旧版列表仅会被加载，已存储项的枚举遵守其每个端点的
  上限，并且兼容性发送绝不会启动或恢复空闲线程。
