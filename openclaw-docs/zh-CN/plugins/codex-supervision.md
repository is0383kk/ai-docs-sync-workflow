---
read_when:
    - 你希望 Codex Desktop 或 CLI 会话显示在 OpenClaw 中
    - 你需要从已存储或空闲的本地 Codex 会话创建分支，或将其归档
    - 你正在公开来自已配对节点的 Codex 会话和对话记录历史
sidebarTitle: Codex supervision
summary: 浏览 OpenClaw 节点中未归档的原生 Codex 会话和分页转录记录
title: 监督 Codex 会话
x-i18n:
    generated_at: "2026-07-26T06:20:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f365e3207dff092c3dfd8f7588d60d70a16f0cce484991eb4ab3fc0bd15f8051
    source_path: plugins/codex-supervision.md
    workflow: 16
---

Codex 监管是官方 `codex` 插件的一项可选能力。它会在常规会话侧边栏和聊天窗格中显示来自 Gateway 网关计算机及已选择加入的配对计算机上的未归档 Codex CLI、VS Code、Atlas 和 ChatGPT 来源会话。

初始版本刻意将所有权范围保持在较小范围内：

- 已存储或空闲的本地会话可基于其有限范围内持久化的用户和助手历史记录，创建一个锁定模型的 OpenClaw 聊天。第一条消息会启动原生快照分支，然后使用 Codex App Server 为该分支选择的确切模型和提供商，启动完整的 Codex harness 线程。后续轮次会恢复规范原生线程中持久化的模型与提供商组合，同时受监管绑定会阻止 OpenClaw 替换为其他运行时、模型或回退方案。独立的原生 Codex 控件仍可更改该持久化组合。已创建的分支会打开其现有聊天。
- 从另一个 Codex 进程发现的已存储会话，其实时活动状态未知。它可以创建分支；或者，只有在操作员确认没有其他 Codex 客户端正在使用它之后，才能将其归档。
- 活动来源会保持可见，但在其当前轮次结束前无法创建分支或归档。如果它已有受监管聊天，**打开聊天**仍然可用。
- 配对节点上的会话通过有界且基于游标分页的 App Server 读取来公开其持久化转录。远程继续需要未来的流式节点桥接；远程归档还需要运行器所有权租约或等效的隔离机制。
- 归档的会话不会列出。只有在操作员确认没有其他 Codex 客户端正在使用已存储或空闲的本地会话后，才能将其归档。

## 开始之前

- 在 Gateway 网关上安装官方 `@openclaw/codex` 插件。启用 Codex 功能时，OpenClaw macOS 应用可安装该插件；CLI 安装可以运行 `openclaw plugins install @openclaw/codex`。
- 在你希望列出其会话的每台计算机上安装 Codex Desktop 或 Codex CLI 并登录。
- 将远程计算机配对为 OpenClaw 节点。每台计算机都必须在本地选择加入；仅在 Gateway 网关上启用监管并不会授权其他节点。
- 使用由所有者控制的 Gateway 网关。会话标题、工作目录和 Git 分支可能泄露敏感的项目信息。

## 启用监管

引导式 `openclaw onboard` 和 macOS 首次运行设置会在检测到原生 Codex 安装并成功激活所选推理后端后，尝试安装并启用 Codex 监管。Codex 无需作为主要后端。该机会式插件激活成功后，监管即可使用。监管首次连接时会检查 App Server 可用性。显式禁用 Codex 插件或策略阻止会阻止机会式激活，而现有的显式 `supervision.enabled: false` 会禁用面向智能体的监管工具；只要 Codex 插件处于活动状态，操作员目录就会保持注册，除非 `sessionCatalog.enabled: false` 将其禁用。这个独立开关不会改变 Codex 提供商、harness 和面向智能体的监管策略，同时还会从此主机移除配对节点的目录列出/读取命令。现有安装可以手动启用相同能力：

在 `openclaw.json` 中启用 `codex` 插件及其监管能力：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          supervision: {
            enabled: true,
          },
        },
      },
    },
  },
}
```

如果存在 `plugins.allow`，请包含 `codex`。更改插件激活状态后，重新启动 Gateway 网关。

如果没有显式的 `appServer` 连接设置，监管会针对原生用户 Codex 主目录使用单独的托管 stdio 监管连接。普通 Codex harness 默认仍限定在智能体范围内。这样既能让原生会话在两个应用中可见，又不会让普通 OpenClaw 轮次共享原生 Codex 状态。如果 harness 也应共享该状态，请显式设置 `appServer.homeScope: "user"`。监管会遵循显式的 `appServer` 连接设置，而不会将其替换为本地用户主目录默认设置。

从侧边栏的 **Codex** 分组接管的聊天不是普通的 harness 会话。其私有监管绑定会使用监管连接执行来源读取、规范分支创建、历史记录注入以及之后的每个轮次。使用默认本地连接时，这会保留原生用户 Codex 主目录、身份验证和提供商配置，同时不会更改其他会话的默认设置。受监视的已接管聊天也会参与[会话状态感知](/zh-CN/concepts/session-state)。

对于默认本地监管连接，存储与原生 Codex 客户端共享。OpenClaw 不会假设另一个客户端共享相同的实时 App Server 进程，而且原生状态所有权仅限于进程本地。因此，对于监管 App Server 报告为 `notLoaded` 的线程，它会将其视为**已存储/活动状态未知**，而不是空闲。

在每个应显示其会话的无头节点主机上应用相同的选择加入设置。原生 OpenClaw macOS 应用在向配对的 Gateway 网关公布其 Codex 目录时，会读取相同的本地设置。该配对原生 Mac 目录仅支持默认或显式的 `appServer.transport: "stdio"`，且 `appServer.homeScope: "user"` 未设置或已显式设置。该 stdio 进程会遵循 `command`、`args` 和 `clearEnv`。如果 Mac 配置选择了 `"unix"`、`"websocket"` 或 `homeScope: "agent"`，应用不会公布目录能力或命令，并且过期的直接调用会失败，而不是公开用户 Codex 主目录或生成另一个本地 stdio App Server。

新公布的节点命令会更改节点已获批准的命令界面。请从 Gateway 网关主机批准更新：

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

未归档的 Codex 会话也会显示在 Control UI 主侧边栏中，并按主机分组。选择一个会话即可读取其持久化转录。查看器使用最新的 Codex `thread/turns/list` API 和 `itemsView: "full"`，每次请求最多加载 20 个轮次；**加载更早的转录项目**会沿用最新页面中的不透明 App Server 游标。已加载的页面按时间顺序呈现。查看器绝不会加载无界的 `thread/read` 历史记录。超过 20 MiB 传输安全上限的页面会以失败关闭方式处理，而不会让节点或 Gateway 网关连接面临风险。

打开常规会话侧边栏中的 **Codex** 分组。它会列出按主机分组的相同会话。**加载更多会话**会从每个仍有更早行的主机追加下一页，这些追加行会在侧边栏的定期刷新后继续保留。每个主机都会在其自身原生列表查询完成后立即显示。可见页面会在节点连接状态变化后、重新获得焦点时以及最多每 30 秒进行一次协调；结果发生变化时，会更快地进行一次后续检查。因此，在 Codex Desktop、CLI 或其他原生客户端中创建的会话无需完整重新加载页面即可显示。第一页遵循 Codex 自身按最近更新时间排序的顺序，因此新创建的原生会话可立即进入结果。
由于原生搜索还可匹配转录预览，因此每个返回的搜索页面会扫描每个主机上数量有限的原生页面，而不是将查询发送到 App Server。

主机可用性和线程状态相互独立。**离线**或**不可用**描述的是主机刷新状态；不可用的主机不会返回新的会话行，也不会将线程的原生状态更改为 `offline`。会话行使用 Codex 状态，例如 `idle`、`active`、`notLoaded` 或错误。某个主机失败不会隐藏健康主机的结果。

侧边栏警告包含目录错误代码和安全的底层 Gateway 网关错误。打开**设置 > 自动化 > 插件 > Codex > 原生会话发现**，可在不禁用 Codex 的情况下禁用发现。对于 `NODE_LIST_FAILED`，请比较 `openclaw nodes list` 和**设置 > 设备**；详细原因会指出需要修复的是配对存储、节点注册表、权限还是 Gateway 网关生命周期故障。

## 使用操作员 CLI

终端 CLI 提供相同的未归档目录，以及 Gateway 网关本地分支和归档操作：

```bash
openclaw codex sessions [--search <text>] [--host <id>] [--limit <count>] [--cursor <cursor>] [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
openclaw codex continue <thread-id> [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
openclaw codex archive <thread-id> --confirm-no-other-runner [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
```

`openclaw codex sessions` 选项：

- `--search <text>`以不区分大小写的方式搜索会话标题。
- `--host <id>`将响应限制为一个稳定的目录主机，例如 `gateway:local` 或 `node:<node-id>`。
- `--limit <count>`将每个主机的行数设置为 1 到 100；默认值为 50。
- `--cursor <cursor>`继续获取一个主机的页面，因此需要 `--host`。
- `--json`输出结构化的 Gateway 网关响应。

这三个命令都从 Gateway 网关客户端继承 `--url`、`--token` 和 `--timeout <ms>`。会话列出的默认超时时间为 75,000 ms，以便冷启动的配对节点目录能够完成；继续和归档的默认超时时间为 30,000 ms。它们还提供共享的 `--expect-final` 开关，但该开关不会更改这些一元监管 RPC。每个命令都需要 `operator.write` Gateway 网关权限范围。
每个子命令都提供标准的 `-h, --help` 输出。
不存在已归档或包含已归档选项。`sessions` 可以列出配对主机，但 `continue` 和 `archive` 始终以 `gateway:local` 为目标；配对行仅供列出。归档始终需要 `--confirm-no-other-runner`。

这些 shell 命令不同于聊天内的 `/codex` 运行时命令。
`/codex threads [filter]` 列出当前会话连接可用的 App Server 线程。`/codex sessions --host <node>` 列出一个节点上可恢复的 Codex CLI 会话文件，而不是监管机群目录。`/codex
resume` 和 `/codex bind` 会附加当前会话，而不是创建安全的受监管分支，并且锁定模型的受监管聊天会拒绝这些绑定变更。不存在 `/codex continue` 或 `/codex archive` 运行时命令。

## 从本地会话创建分支

在 Gateway 网关计算机上的已存储或空闲行中选择**作为分支继续**。OpenClaw 会创建普通聊天条目，镜像截至来源最后一个持久化终止轮次（已完成、已中断或失败）的有限用户和助手历史记录，记录一个待处理的 harness 分支，然后打开聊天。通用模型选择器会被锁定，但此时尚未选择具体模型或提供商。来源不会恢复，规范 harness 线程也尚未启动。重复执行此操作会打开现有聊天，而不是创建另一个分支。

镜像会保留同时满足以下三个限制的最新可见尾部：最多 200 条用户或助手消息、UTF-8 文本总计 512 KiB，以及每条消息 64 KiB。过大的消息会使用标记截断，达到上限时会省略更早的消息。图像或本地图像输入会变为字面量 `[Image attachment]` 占位符；不会复制图像数据和本地路径。

发送第一条普通聊天消息即可开始工作。Codex harness 会安装真实的
审批、信息征询、事件和交付处理程序。它会在监督连接上使用一个临时的
原生分支固定源快照，而不提供模型或提供商覆盖。Codex App Server 会从其
当前原生配置中选择这两者，并返回实际选择。在同一连接上，
OpenClaw 会在其 cwd 和运行时策略下，使用返回的确切组合启动规范的 `appServer` 源完整 harness 线程，
注入有限的可见历史记录，并归档临时分支。规范线程
拥有完整的 OpenClaw harness 工具界面。这是一个可见历史记录分支，而不是
完整的原生 rollout 克隆：源推理、工具调用和工具结果均被
省略。此轮及之后的每一轮都保留在受监督的 Codex 连接上，
而不会转移到其他 OpenClaw 模型运行时或普通的 Agent 主目录 harness。

返回的选择不能证明源在历史上使用的模型。如果
当前原生配置与源最后一轮所记录的模型不同，
Codex 会发出常规模型差异警告。OpenClaw 会使用
返回的组合启动规范线程。Codex 会持久化该规范
线程的原生模型和提供商；由于 OpenClaw 不提供模型和提供商覆盖，
后续恢复会保留它们。如果通过单独的原生 Codex 控制更改规范线程，
OpenClaw 会接受 Codex 持久化的
选择。OpenClaw 绝不会替换为其外层模型或回退链。

受监督且模型锁定的聊天无法删除、切换模型、使用 `/new`
或 `/reset`、调用 Gateway 网关会话重置操作，也无法使用通用的
**派生会话**操作。修改 `/codex model <model>`、`/codex
bind`、`/codex resume`（包括使用 `--bind here` 的节点会话）以及
`/codex detach` 或 `/codex unbind` 也会被拒绝，因为这些操作会替换
或清除锁定的原生绑定。`/codex model` 查询以及 `/codex fast`、
`/codex permissions` 和 `/codex threads` 仍然可用。如需不同模型或全新线程，
请启动另一个普通会话。

请为此聊天保持启用监督。如果监督被禁用，或者其
存储的连接绑定不可用或不一致，该轮次会
以故障关闭方式失败，而不会转移到普通的 Agent 主目录会话。

禁用或卸载 `codex` 插件不会释放该所有权，
也不会使该聊天可以使用其他模型。锁定的聊天会保留，但
无法使用；请重新安装或重新启用同一插件并重启 Gateway 网关以
恢复它。这种有意采用的故障关闭行为可防止保留清理或
临时插件中断悄然使原生绑定失去归属。

`codex_threads` Agent 工具遵循相同的边界。它无法附加
其他分支，也无法归档该聊天绑定的原生线程。列表和仅元数据
读取仍然可用。原始记录读取需要 `allowRawTranscripts`。
禁用原始访问后，`codex_threads` 也会拒绝列表搜索，因为
原生搜索包含记录预览；Control UI 和操作员 CLI
仍提供有限的仅标题搜索。重命名、取消归档、分离式派生，以及
归档不相关且无所有者的线程，需要
`allowWriteControls`。这两个选项都无法绕过锁定的绑定。

OpenClaw 仅列出源线程或显示待处理聊天时，
不会订阅或回应审批请求。第一轮启动一个独立的规范
harness 线程，可让另一个 Codex 进程继续拥有
源，而不会产生相互竞争的 rollout 写入器。

原始 CLI、VS Code、Atlas 或 ChatGPT 源仍对原生
客户端和 OpenClaw 目录可见。规范分支存储为原生
Codex 线程，但其源类型为 `appServer`；Codex Desktop 或其他
原生客户端可能会过滤该源类型，因此无法保证分支本身
出现在每个原生历史记录视图中。

OpenClaw 的 App Server 报告为活动状态的行无法启动新分支。请等待
当前轮次完成并刷新目录。Codex App Server
会在单个进程中串行执行变更操作，但不会提供独占的
跨进程运行器或审批所有者租约。

对于**已存储 / 活动状态未知**的行，聊天镜像和第一轮快照
固定使用 Codex 截至最后一个已持久化终止轮次的状态。源
线程不会被恢复、中断或归档。如果另一个进程存在
正在进行的轮次，其最新的进行中工作可能不会出现在分支中。

## 归档本地会话

在已存储或空闲的 Gateway 网关本地行上选择**归档**，然后确认没有
其他 Codex 客户端或 OpenClaw 运行器正在使用该线程或其派生的
后代线程。OpenClaw 会重新读取进程本地状态，仅在状态为
`idle` 或 `notLoaded` 时继续，调用原生 Codex 归档操作，并从
未归档列表中移除该会话。原生 Codex 还会尝试归档该
线程派生的后代线程。

如果重新读取后报告会话处于活动或错误状态、会话属于已配对节点，
或者新创建的受监督聊天仍有从该源派生的待处理分支，
则无法归档。请发送聊天的第一条消息以实体化其规范分支，
然后再归档源。如果 OpenClaw 已知某个活动绑定拥有
确切的目标线程或任何未归档的派生后代线程，归档也会被阻止。
OpenClaw 会逐页执行实验性 Codex 后代查询；无效响应、
请求失败、重复的游标或线程，或者安全限制耗尽，都会导致
归档被拒绝。

读取、后代枚举和归档请求不是一个条件式
操作，因此轮次仍可能在这些操作之间启动。App Server 状态也
不会在独立进程之间共享。因此，对于未知客户端和该竞态，
确认操作就是安全边界：请在确认前退出或以其他方式核实
所有其他客户端。可使用 Codex Desktop、Codex CLI 或经所有者授权的
原生线程管理流程恢复已归档线程；取消归档后，它会重新出现。

```bash
codex unarchive <thread-id>
```

## 了解已配对节点的限制

已配对节点会公开带版本的只读
`codex.appServer.threads.list.v1` 和
`codex.appServer.thread.turns.list.v1` 命令。安装了
Codex CLI 的原生节点主机还会公开列入允许列表的 `codex.terminal.resume.v1`
命令。Gateway 网关接收规范化的
元数据和明确请求的有限记录页面，而不会接收原始 App Server
端点。在操作员终端中打开某行时，会在所属主机上运行 `codex resume <thread-id>`
并中继该命令的 PTY；这不会公开通用
shell 或由 Gateway 网关提供的 argv。

终端中继不提供 harness 继续执行或归档所有权
契约。因此，即使远程线程处于空闲状态，远程行仍然可见，但不会提供**继续**或
**归档**。请通过**在终端中打开**在该计算机上使用 Codex，
或使用未来具备安全运行器所有权边界的继续执行流程。

## 元数据和权限

目录行可能包含：

- 线程和会话标识符
- 标题和工作目录
- 当前状态和活动等待标志
- 创建、更新和活动时间戳
- 来源、模型提供商、Codex CLI 版本和 Git 分支

目录投影不包含记录预览、轮次、rollout 路径、
Codex 主目录路径、Git 远程仓库、提交 SHA 和原始 App Server 错误。目录
访问和 Control UI 记录读取需要 `operator.write` Gateway 网关
权限范围，因为集群聚合使用标准 `node.invoke` 路径，即使
两个节点命令都是只读的。

`supervision.allowRawTranscripts` 和 `supervision.allowWriteControls` 用于管理
自主智能体和独立 MCP 工具。两者默认均为 `false`。启用
监督后，除非允许原始记录，否则 `codex_threads` 会从
列表和仅元数据读取结果中移除记录预览和轮次；
包含轮次的读取会以故障关闭方式失败。每次派生、重命名、归档和取消归档
都需要写入控制。这些选项不限制经过身份验证的 Control UI
记录查看，也不会绕过绑定、主机、状态或确认检查。

### 兼容性工具

官方 `codex` 插件会为现有智能体和独立 MCP 客户端
保留已发布的五个 Supervisor 工具名称：

- `codex_endpoint_probe`
- `codex_sessions_list`
- `codex_session_read`
- `codex_session_send`
- `codex_session_interrupt`

`codex_sessions_list` 默认仅包含已加载项；不存在 `loaded_only`
参数。设置 `include_stored: true` 后，还会从 Codex 的状态数据库读取
未归档的已存储行。可选的 `max_stored_sessions` 上限默认为 200，
每个端点接受 1 到 1,000 行。它不限制已加载行。
如果没有原始记录权限，列表结果会省略从记录派生的名称、
预览和详细端点错误。
`codex_session_read` 需要 `allowRawTranscripts`；`include_turns: true`
还会要求 Codex 返回轮次。

`codex_session_send` 和 `codex_session_interrupt` 需要
`allowWriteControls`。发送操作接受 `mode: "auto" | "start" | "steer"`，但
`"start"` 始终会被拒绝，并且 `"auto"` 和 `"steer"` 都只能 Steer
可读取的活动轮次。空闲线程会被拒绝，并提示使用 **Codex
会话**；在该位置，完整 harness 会先安装审批和工具处理程序，
然后再继续执行。中断操作同样需要可读取的活动轮次。这些工具
不会恢复或启动空闲的源线程。

`openclaw doctor --fix` 会将已停用的 `codex-supervisor` 条目、其端点
和权限字段，以及插件允许/拒绝策略引用迁移到官方
`codex` 插件中，而不会覆盖明确设置的规范配置。独立的
兼容性 MCP 适配器会继续从该插件加载相同的五个工具；
旧版策略环境变量仅在该受信任适配器内部生效。

有关所有监督配置字段，请参阅
[Codex harness reference](/zh-CN/plugins/codex-harness-reference#supervision)。

## 故障排查

**未显示任何会话：**请确认已安装 `@openclaw/codex`，插件和
`supervision.enabled` 均为 true，当前插件允许列表允许
`codex`，并且会话未被归档。更改激活状态后，请重启 Gateway 网关或节点。

**继续功能已禁用：**未映射的行处于活动状态、属于已配对节点、
其主机离线，或另一操作正在等待处理。Gateway 网关本地已存储和空闲
行会提供**以分支方式继续**，而不是不安全地接管确切线程。已经
拥有受监督聊天的行会提供**打开聊天**。

**归档功能已禁用：**对于已存储/活动状态未知和
空闲的 Gateway 网关本地行，在确认没有其他运行器后可以归档。活动、错误、
离线、已配对节点、待处理分支，以及已知有确切绑定所有者的行，在归档方面仍为
只读。

**已归档会话消失：**这是预期行为。监督页面
没有归档视图。运行 `codex unarchive <thread-id>` 或使用 Codex Desktop
重新显示它。

**仍存在旧的 `codex-supervisor` 配置：**运行 `openclaw doctor --fix`。Doctor
会将已停用的插件条目及相关插件策略引用迁移到
`plugins.entries.codex.config.supervision`，而不会覆盖明确设置的 Codex
配置。

## 相关内容

- [Codex harness](/zh-CN/plugins/codex-harness)
- [Codex harness reference](/zh-CN/plugins/codex-harness-reference)
- [Codex harness runtime](/zh-CN/plugins/codex-harness-runtime)
- [Codex 监督架构](/zh-CN/specs/codex-supervision)
- [节点](/zh-CN/nodes)
- [Gateway 安全](/zh-CN/gateway/security)
