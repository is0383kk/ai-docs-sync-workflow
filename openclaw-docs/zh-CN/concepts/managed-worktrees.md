---
read_when:
    - 你希望为一项智能体任务创建隔离的分支和检出目录
    - 你正在为 Workboard 卡片配置 worktree 工作区
    - 你需要恢复或清理由 OpenClaw 管理的工作树
summary: 在隔离的 Git 检出目录中运行智能体任务，并自动创建快照和清理
title: 托管工作树
x-i18n:
    generated_at: "2026-07-26T06:39:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98ed2579b7243544dbdb550c4b8a292ccd4ab494fd4a45b2404256691c831401
    source_path: concepts/managed-worktrees.md
    workflow: 16
---

托管工作树为智能体任务提供独立的 git 分支和检出目录，而无需在源代码仓库内放置临时目录。OpenClaw 在其状态目录下创建工作树，将其记录在共享状态数据库中，并在移除前对其中已跟踪及未忽略的未跟踪内容创建快照。

## 布局和命名

每个工作树位于：

```text
<openclaw-state-dir>/worktrees/<repo-fingerprint>/<name>
```

仓库指纹是对规范 git 公共目录和源 URL 进行 SHA-256 哈希所得值的前 16 个十六进制字符。提供的名称必须匹配 `[a-z0-9][a-z0-9-]{0,63}`。如果未提供名称，OpenClaw 会生成 `wt-`，后跟八个随机十六进制字符。

OpenClaw 在请求的基础引用处创建分支 `openclaw/<name>`。如果未提供基础引用，它会获取 `origin`，在远程默认分支可用时使用该分支，并在仓库离线或没有可用远程仓库时回退到本地 `HEAD`。

## 配置忽略的文件

在源代码仓库根目录添加 `.worktreeinclude`，以将选定的已忽略未跟踪文件复制到新工作树中。该文件使用 gitignore 模式语法，每行一个模式，并使用 `#` 注释：

```gitignore
.env.local
fixtures/generated/**
```

只有被 git 报告为同时处于已忽略和未跟踪状态的文件才符合条件。已跟踪文件已通过 git 存在，此步骤绝不会复制它们。OpenClaw 不会覆盖或更改已存在的目标文件，不会跟随符号链接目录，并会保留所复制文件的模式。它仅记录实际创建的路径，因此后续对清单的编辑不会导致这些文件失去清理保护。

## 运行仓库设置

如果源代码仓库中存在 `.openclaw/worktree-setup.sh` 且其可执行，OpenClaw 会以新工作树作为当前目录运行它。该脚本接收：

```text
OPENCLAW_SOURCE_TREE_PATH=<source checkout>
OPENCLAW_WORKTREE_PATH=<managed worktree>
```

非零退出会中止创建，并移除新工作树和分支。这是仓库本地契约；没有对应的 OpenClaw 配置键。

## 会话工作树

通过工作树会话从 Git 支持的文件夹启动隔离聊天：在 Control UI 的 New session 页面上，使用 **Place** 选择器选择 Gateway 网关源文件夹，然后选择 **Worktree**（可选择性指定基础分支和工作树名称）。只有在 Gateway 网关确认所选文件夹是 Git 检出目录后，才会显示此选项；普通文件夹会直接运行，不显示 Git 隔离控件。当活动智能体工作区由 Git 支持时，iOS 会在 Chat actions 中提供相同选项，Android 则在 New Chat 旁提供此选项。

编码智能体在发现当前任务之外已确认的后续工作时，也可以调用 `spawn_task`。Control UI 会显示建议提示条，但不会启动任何操作；由 Gateway 网关支持的 TUI 则会显示带有相同操作的交互式提示。选择 **Start in worktree** 会从建议的项目创建新的会话所有工作树，并将自包含提示作为其首个轮次发送；关闭建议不会修改仓库。建议及其 ID 是临时的，Gateway 网关重启后不会保留。

OpenClaw 仅向具有可操作 Gateway 网关 UI 的操作员会话公开这些工具。渠道会话以及本地或嵌入式 TUI 会话在具备可移植的类型化任务操作契约之前，不会获得这些工具。

生成的托管工作树归会话所有，并且该会话中的每次智能体运行都使用其检出目录。当工作区是仓库子目录时，工作树以仓库根目录为锚点，而会话从其中对应的子目录运行。会话工作树创建使用该方法的 `operator.write` 权限范围，但仓库检出钩子和 `.openclaw/worktree-setup.sh` 步骤仅对 `operator.admin` 调用方运行，因为它们会执行仓库代码；`.worktreeinclude` 配置仍适用于所有调用方。仅当移除不会造成损失时，删除会话才会移除工作树。脏工作树或包含未推送提交的分支会继续保留；每小时清理会在会话工作树空闲 7 天后为其创建快照，并将最近的会话活动视为工作树活动。已移除的工作树仍可按下文所述从其快照恢复。

`sessions.create` 可以包含绝对 `cwd`，以便直接在另一个 Gateway 网关文件夹中运行；也可以结合 `worktree: true` 选择源检出目录，或设置已配对节点的工作目录。每个显式主机路径都需要 `operator.admin`；普通工作树聊天创建仍使用 `operator.write`，并继续锚定到已配置的工作区。

`sessions.create` 除 `worktree: true` 外还接受 `worktreeBaseRef` 和 `worktreeName`，用于选择基础引用和工作树名称（分支会变为 `openclaw/<name>`）；两者均保持在 `operator.write`。创建的工作树会在创建结果中返回，并以 `worktree: { id, branch, repoRoot }` 持久化到会话行中，因此会话列表可以显示检出目录和分支。删除会话时，会将保留的脏检出目录报告为 `worktreePreserved`，而不是静默将其遗留。

## 快照、清理和恢复

移除操作首先创建一个包含已跟踪文件和未忽略的未跟踪文件的合成提交，然后将其固定在 `refs/openclaw/snapshots/<id>`。忽略的文件绝不会进入仓库对象数据库。OpenClaw 仅将其实际配置的忽略文件存储在分块共享状态数据库行中；即使 `.worktreeinclude` 后续发生更改或消失，所记录的路径集仍是权威依据。恢复操作从不可变快照中读取这些字节，并重新应用其完整模式。当记录的路径无法再安全创建快照时，自动清理会保留实时工作树。如果快照创建失败，移除操作会停止。显式强制删除可以在没有快照的情况下继续。

OpenClaw 应用以下清理规则：

- 运行结束时，仅当 `git status --porcelain` 为空且 `git log HEAD --not --remotes --oneline` 未发现未推送提交时，才会移除工作树。否则，它只会释放活动锁。
- 每小时清理会为已解锁且空闲超过 7 天的 Workboard 所有和会话所有工作树创建快照并将其移除，即使它们处于脏状态。手动工作树绝不会被自动移除。
- 快照记录可恢复 30 天。之后，清理操作会删除快照引用和注册表行。
- 实时 OpenClaw 进程锁以及任何外部或无法识别的 git 工作树锁都会保护工作树免遭垃圾回收。

恢复操作会在创建快照前的原始提交处重新创建 `openclaw/<name>`，然后将快照差异重建为未暂存的修改和未跟踪文件。这样可避免合成快照提交进入分支历史记录。快照引用会继续保留为来源记录。

## CLI

```bash
openclaw worktrees list [--json]
openclaw worktrees create <repo-root> [--name <name>] [--base-ref <ref>] [--json]
openclaw worktrees remove <id> [--force] [--json]
openclaw worktrees restore <id> [--json]
openclaw worktrees gc [--json]
```

Settings 下的 Control UI **Worktrees** 页面提供相同的操作，还支持通过基础分支选择器创建工作树；它会显示每个工作树的所有者（手动、Workboard 或拥有该工作树的会话，并提供进入其聊天的链接），并在移除操作报告快照失败时提供强制重试选项。

## Gateway 网关方法

| 方法               | 用途                                                                 |
| -------------------- | ----------------------------------------------------------------------- |
| `worktrees.list`     | 列出活动和可恢复的工作树记录。                            |
| `worktrees.branches` | 列出仓库的本地和远程分支，供基础引用选择器使用。    |
| `worktrees.create`   | 创建或复用命名的托管工作树。                               |
| `worktrees.remove`   | 为工作树创建快照并将其移除。强制移除会报告 `snapshotError`。 |
| `worktrees.restore`  | 从快照恢复已移除的工作树。                           |
| `worktrees.gc`       | 立即运行空闲、孤立和保留期清理。                            |

`worktrees.list` 需要 `operator.read`，而修改型方法需要 `operator.admin`。对于已配置的智能体工作区，`worktrees.branches` 需要 `operator.write`；其他任何主机路径则需要 `operator.admin`（与 `sessions.create` cwd 门槛一致）。它仅读取现有引用，从不执行获取操作；仅存在于远程的分支会以远程限定名称返回（`origin/feature-a`），因此每个返回的名称都能解析为基础引用。New Session 还可以从此方法请求类型化仓库状态；普通目录或不可用的检出目录不会返回分支，而不是迫使 UI 从错误字符串中推断 Git 能力。

## Workboard 工作区

内置的 [Workboard 插件](/zh-CN/plugins/workboard)可以将卡片工作区具体化为托管工作树：

```json
{
  "kind": "worktree",
  "path": "/absolute/path/to/source-checkout",
  "branch": "main"
}
```

`path` 标识源 git 检出目录。`branch` 是可选项，并会成为基础引用。对于完整主机调用方，Workboard 会创建或复用 `wb-<card-id>`，以托管检出目录作为工作目录运行子智能体，并将解析后的路径和分支写回卡片。Gateway 网关客户端需要 `operator.admin` 才能执行完整主机具体化。运行结束时，仅当可以证明移除不会造成损失时，Workboard 才会移除检出目录；脏工作或未推送提交会继续保留。

对于受工作区约束的调用方，`path` 和仓库根目录必须与目标智能体工作区完全匹配。随后，Workboard 会直接在该目录中运行，并记录目录工作区，而不是在主机上具体化托管工作树。目标必须为同一工作区使用可写、非共享的 Docker 沙箱，其实时容器哈希必须与请求的挂载和策略相匹配，并且不得公开提升权限的执行、主机控制、主机范围会话、持久化主机或节点执行，以及未分类的插件和 MCP 工具。如果目标策略或实时容器的权限范围更广，分派操作会使卡片保持未认领状态，并报告不兼容状态。
