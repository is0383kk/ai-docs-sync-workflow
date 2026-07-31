---
read_when:
    - 你需要为本地 OpenClaw 状态创建一流的备份归档文件
    - 你需要获取一个 OpenClaw SQLite 数据库的紧凑且经过验证的快照
    - 你想在重置或卸载之前预览将包含哪些路径
summary: '`openclaw backup` 的 CLI 参考（归档和 SQLite 快照）'
title: 备份
x-i18n:
    generated_at: "2026-07-26T06:37:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dfb5a118545589b181cede26dab72e9d029d98a1cac5cfccedd9d9cf2c56d3b5
    source_path: cli/backup.md
    workflow: 16
---

# `openclaw backup`

为 OpenClaw 状态、配置、身份验证配置文件、渠道/提供商凭据、会话以及可选的工作区创建本地备份归档。

```bash
openclaw backup create
openclaw backup create --output ~/Backups
openclaw backup create --dry-run --json
openclaw backup create --verify
openclaw backup create --no-include-workspace
openclaw backup create --only-config
openclaw backup verify ./2026-03-09T08-00-00.000+08-00-openclaw-backup.tar.gz
openclaw backup sqlite create --global --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite create --agent main --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite list --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite verify ~/Backups/openclaw-sqlite/<snapshot-id>
openclaw backup sqlite verify ~/Backups/openclaw-sqlite/<snapshot-id> --scratch ~/Private/openclaw-scratch
openclaw backup sqlite restore ~/Backups/openclaw-sqlite/<snapshot-id> --target ./restored/openclaw.sqlite
```

## 注意事项

- 归档中嵌入了一个 `manifest.json`，其中包含解析后的源路径和归档布局。
- 默认输出为当前工作目录中带时间戳的 `.tar.gz` 归档。带时间戳的文件名使用本机的本地时区，并包含 UTC 偏移量。如果当前工作目录位于要备份的源目录树内，OpenClaw 会改用主目录作为默认归档位置。
- 绝不会覆盖现有归档文件。为避免归档包含自身，系统会拒绝位于源状态/工作区目录树内的输出路径。
- `openclaw backup verify <archive>` 会检查归档是否仅包含一个根清单，拒绝遍历式归档路径和 SQLite 辅助文件，确认清单声明的每个有效负载均存在，验证每个 SQLite 快照的文件结构，并对规范 OpenClaw 数据库执行完整的完整性和角色检查。专用插件 schema 保持不透明，因为它们可能需要所有者定义的 SQLite 能力。`openclaw backup create --verify` 会在写入归档后立即运行该验证。
- `openclaw backup create --only-config` 仅备份当前使用的 JSON 配置文件。

## SQLite 快照

如果需要为单个 OpenClaw 所有的 SQLite 数据库创建便携式工件，而不是范围广泛的状态归档，请使用 `openclaw backup sqlite`。

创建快照时只接受一个具名源：

| 命令                                                         | 数据库               |
| --------------------------------------------------------------- | ---------------------- |
| `openclaw backup sqlite create --global --repository <dir>`     | OpenClaw 共享状态  |
| `openclaw backup sqlite create --agent <id> --repository <dir>` | 每个 Agent 一个数据库 |

存储库为每个已提交的快照包含一个目录。每个快照目录严格包含：

- `manifest.json`
- `database.sqlite`

创建快照时，会先验证实时数据库再读取它，使用 SQLite 在线备份 API 捕获已提交的 WAL 状态，同时避免长时间持有单个读取事务，然后关闭实时数据库，使用 `VACUUM` 压缩私有副本，再次验证生成的数据库，并在不覆盖现有路径的情况下发布已完成的目录。全局快照会在压缩前移除临时投递队列行，以免已删除的队列有效负载残留在空闲页中。

不要将实时的 `.sqlite`、`-wal`、`-shm` 或 `-journal` 文件复制为便携式工件。只复制已完成的快照目录。

SQLite 快照可能包含身份验证配置文件、会话状态、插件状态及其他敏感记录。保护存储库时，应采用与实时 OpenClaw 状态目录相同的权限、加密、保留策略和目标限制。

### 验证和恢复

```bash
openclaw backup sqlite verify <snapshot-directory>
openclaw backup sqlite restore <snapshot-directory> --target <new-database-path>
```

验证会检查严格的清单结构、工件大小和 SHA-256、SQLite 完整性、外键、schema 版本、数据库角色和所有者，以及 OpenClaw 所有的索引定义。

验证使用内容固定的私有副本，以防路径名竞争条件替换 SQLite 所检查的字节。默认情况下，该临时副本会在快照存储库旁创建，并在命令返回前删除。暂存根目录及其祖先目录链必须防止其他用户替换它。POSIX 根目录必须归当前用户所有，且不可由组或所有人写入；对于用户所有的子目录，可以接受 `/tmp` 等带粘滞位的祖先目录。会暴露暂存内容或允许替换暂存内容的 macOS ACL 授权会被拒绝。Windows 根目录及其祖先目录必须归当前用户或受信任的操作系统主体所有，并通过 ACL 拒绝不受信任的暂存访问。对于只读挂载或网络共享，请在具有同等加密和目标控制的存储上通过 `--scratch <existing-private-directory>`。

在暂存或发布数据库字节前，快照创建过程会对存储库执行相同的所有者、ACL、祖先目录和路径身份检查。

恢复过程会重复验证，并且只写入全新的目标。它会拒绝现有目标、`-wal`、`-shm` 或 `-journal` 辅助文件，且绝不会就地替换实时 OpenClaw 数据库。目标父目录的路径安全要求与验证暂存目录相同。启用已恢复的数据库仍需操作员显式执行离线步骤。

快照存储库是本地目录。调度、上传、保留、增量 WAL 包、故障转移和启动时恢复行为有意不包含在此命令的范围内。

## 备份内容

`openclaw backup create` 根据本地 OpenClaw 安装规划源：

- 状态目录（通常为 `~/.openclaw`）
- 当前使用的配置文件路径
- 解析后的 `credentials/` 目录（当其存在于状态目录之外时）
- 从当前配置中发现的工作区目录，除非传入 `--no-include-workspace`

身份验证配置文件及其他每 Agent 运行时状态存储在状态目录下的 SQLite 中（`agents/<agentId>/agent/openclaw-agent.sqlite`），因此状态备份条目会自动涵盖这些内容。

`--only-config` 会跳过状态、凭据目录和工作区发现，仅归档当前使用的配置文件路径。

OpenClaw 会先规范化路径再构建归档：如果配置、凭据目录或工作区已位于状态目录内，则不会将其重复添加为独立的顶层备份源。缺失的路径会被跳过。

创建归档期间，OpenClaw 会在 `tar` 读取已知的实时变更路径前将其排除。这样可避免文件记录的大小与并发写入之间出现竞争条件。该过滤器会在每个备份的状态目录下应用以下状态相对路径规则：

| 状态相对范围                         | 跳过的文件后缀         |
| -------------------------------------------- | ----------------------------- |
| `sessions/**`                                | `.jsonl`、`.log`              |
| `agents/<agentId>/sessions/**`               | `.jsonl`、`.log`              |
| `cron/runs/**`                               | `.jsonl`、`.log`              |
| `logs/**`                                    | `.jsonl`、`.log`              |
| `delivery-queue/**`                          | `.json`、`.delivered`、`.tmp` |
| `session-delivery-queue/**`                  | `.json`、`.delivered`、`.tmp` |
| 备份状态目录下的任何路径 | `.sock`、`.pid`、`.tmp`       |

这些规则不会过滤状态目录之外的工作区文件。它们还会忽略符合表中规则的已完成转录和日志文件，因此需要时应单独保留这些记录。JSON 结果中的 `skippedVolatileCount` 会报告有多少文件被有意忽略。

状态目录下的 SQLite 数据库会使用 SQLite 在线备份 API 捕获，并通过 `VACUUM` 离线压缩，以防已删除页面的残留内容进入归档，同时不会复制实时 WAL/SHM 文件。如果插件所有的数据库需要当前不可用的所有者定义 SQLite 能力，操作会以安全失败方式终止，而不会回退到直接复制文件。通过工作区备份包含的 SQLite 文件会作为工作区文件复制，不受压缩保证覆盖。

状态目录的 `extensions/` 目录树下已安装插件的源文件和清单文件会被包含，但其嵌套的 `node_modules/` 依赖目录树会作为可重新构建的安装工件被跳过。恢复归档后，如果恢复的插件报告缺少依赖项，请使用 `openclaw plugins update <id>`，或通过 `openclaw plugins install <spec> --force` 重新安装。

状态目录下由安装程序管理且可重新构建的运行时根目录也会被跳过：`dev/`、`git/`、`npm/`、旧版 `npm-runtime/` 和 `tools/`。这些目录包含托管检出、软件包目录树和下载的运行时，而非权威用户状态；恢复后请重新安装或更新相应的运行时或插件。显式配置且位于这些根目录之一的配置文件、凭据目录或工作区仍会包含在内。

## 配置无效时的行为

`openclaw backup` 会绕过常规配置预检，因此在恢复期间仍可提供帮助。工作区发现依赖有效配置，因此当配置文件存在但无效且工作区备份仍处于启用状态时，`openclaw backup create` 会快速失败。

在这种情况下，如需执行部分备份，请使用 `--no-include-workspace` 重新运行：它会继续涵盖状态、配置和外部凭据目录，同时完全跳过工作区发现。

当配置格式错误时，`--only-config` 也可以使用，因为它不会为工作区发现解析配置。

## 大小和性能

OpenClaw 不强制设置内置的最大备份大小或单文件大小限制。如果归档写入连续五分钟未产生任何数据，操作会失败并删除其未完成的临时文件，而不会无限期挂起。除此之外，实际限制来自：

- 临时归档写入和最终归档所需的可用空间
- 遍历大型工作区目录树并将其压缩为 `.tar.gz` 所需的时间
- 使用 `--verify` 或 `openclaw backup verify` 重新扫描归档所需的时间
- 目标文件系统的行为：OpenClaw 要求通过禁止覆盖的硬链接发布，以确保最终归档路径绝不会暴露仍在进行中的副本；不支持的文件系统会返回可操作的错误

如果发布后无法确认最终目录的持久性，命令会报告失败，但会保留完整的最终条目，以免删除并发替换的文件。

大型工作区通常是决定归档大小的主要因素。使用 `--no-include-workspace` 可获得更小、更快的备份，或使用 `--only-config` 创建最小的归档。

## 相关内容

- [CLI 参考](/zh-CN/cli)
