---
read_when:
    - 你遇到连接或身份验证问题，并希望获得引导式修复帮助
    - 你已完成更新，想进行完整性检查
summary: '`openclaw doctor` 的 CLI 参考（健康检查 + 引导式修复）'
title: Doctor
x-i18n:
    generated_at: "2026-07-26T05:43:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e2b0aa9b51d7bccd4357d3ec747be514a0245b44a90e6e6c7ea789ab68420465
    source_path: cli/doctor.md
    workflow: 16
---

# `openclaw doctor`

对 Gateway 网关、渠道、插件、Skills、模型路由、本地状态和配置迁移执行健康检查和快速修复。每当某些功能未按预期运行，并且你希望通过一条命令了解问题所在时，请使用此命令。

当 Gateway 状态报告 SecretRef 所有者处于降级状态时，Doctor 会输出 **Secret 运行时降级**警告，其中包含每个冷启动或过期的所有者、受影响的配置路径、经隐去处理的原因，以及 `openclaw secrets reload` 重试命令。

当渠道入口事件被发送至死信队列时，Doctor 会列出每个受影响的渠道账户，并指向 [`openclaw channels dead-letters list`](/zh-CN/cli/channels#inbound-dead-letters)，以便检查和恢复。

相关内容：

- 故障排查：[故障排查](/zh-CN/gateway/troubleshooting)
- 安全审计：[安全](/zh-CN/gateway/security)

## 工作模式

Doctor 有五种工作模式：

| 工作模式                  | 命令                                      | 行为                                                                            |
| ------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------- |
| 检查                      | `openclaw doctor`                        | 面向人工操作的检查和引导式提示。                                                |
| 修复                      | `openclaw doctor --fix`                        | 应用支持的修复；除非非交互式修复是安全的，否则会使用提示。                      |
| Lint                      | `openclaw doctor --lint`                        | 为 CI、预检和审查门禁提供只读的结构化发现。                                     |
| 共享 SQLite 维护          | `openclaw doctor --state-sqlite compact`                        | 显式对规范共享状态数据库执行检查点、压缩和验证。                                |
| 会话 SQLite 迁移          | `openclaw doctor --session-sqlite <mode>`                        | 检查、导入、验证、压缩、恢复或还原会话状态。                                    |

当自动化需要稳定结果时，优先使用 `--lint`。当人工操作员希望 Doctor 编辑配置或状态时，优先使用 `--fix`。

## 示例

```bash
openclaw doctor
openclaw doctor --lint
openclaw doctor --lint --json
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --all
openclaw doctor --lint --allow-exec
openclaw doctor --deep
openclaw doctor --fix
openclaw doctor --fix --non-interactive
openclaw doctor --generate-gateway-token
openclaw doctor --post-upgrade
openclaw doctor --post-upgrade --json
openclaw doctor --state-sqlite compact
openclaw doctor --state-sqlite compact --json
openclaw doctor --session-sqlite inspect --session-sqlite-all-agents
openclaw doctor --session-sqlite dry-run --session-sqlite-agent main --json
openclaw doctor --session-sqlite import --session-sqlite-all-agents
openclaw doctor --session-sqlite validate --session-sqlite-all-agents --json
openclaw doctor --session-sqlite compact --session-sqlite-all-agents
openclaw doctor --session-sqlite recover --github-issue
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

对于特定渠道的权限，请使用渠道探测命令，而不是 `doctor`：

```bash
openclaw channels capabilities --channel discord --target channel:<channel-id>
openclaw channels status --probe
```

`channels capabilities` 报告机器人针对特定渠道目标的有效权限。`channels status --probe` 审计所有已配置的渠道和语音自动加入目标。

## 选项

| 选项                            | 效果                                                                                                                                                                                    |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--no-workspace-suggestions`              | 禁用工作区记忆/搜索建议。                                                                                                                                                               |
| `--yes`              | 接受默认值而不提示。                                                                                                                                                                    |
| `--repair` / `--fix` | 应用建议的非服务修复而不提示（`--fix` 是别名）。Gateway 网关服务的安装/重写仍需要交互式确认或显式的 `gateway` 命令。 |
| `--force`              | 应用激进修复，包括覆盖自定义服务配置。                                                                                                                                                  |
| `--non-interactive`              | 在无提示模式下运行；仅执行安全迁移和非服务修复。                                                                                                                                        |
| `--generate-gateway-token`              | 生成并配置 Gateway 网关令牌。                                                                                                                                                           |
| `--allow-exec`              | 允许 Doctor 在验证 Secret 时执行已配置的 `exec` SecretRef。                                                                                                                 |
| `--deep`              | 扫描系统服务以查找额外的 Gateway 网关安装；报告最近的 Gateway 网关监督程序重启移交情况。                                                                                                |
| `--lint`              | 以只读模式运行现代化健康检查并输出诊断发现。                                                                                                                                            |
| `--post-upgrade`              | 运行升级后的插件兼容性探测；发现输出到 stdout；如果存在任何错误级别的发现，则退出代码为 1。                                                                                              |
| `--state-sqlite <mode>`              | 运行显式的共享状态 SQLite 维护。唯一模式是 `compact`。                                                                                                                         |
| `--session-sqlite <mode>`              | 运行指定的会话 SQLite 迁移模式：`inspect`、`dry-run`、`import`、`validate`、`compact`、`recover` 或 `restore`。 |
| `--session-sqlite-store <path>`              | 与 `--session-sqlite` 一起使用：选择一个旧版 `sessions.json` 存储路径。                                                                                                              |
| `--session-sqlite-agent <id>`              | 与 `--session-sqlite` 一起使用：选择一个已配置的智能体。                                                                                                                                |
| `--session-sqlite-all-agents`              | 与 `--session-sqlite` 一起使用：选择已配置和已发现的智能体存储。                                                                                                                        |
| `--github-issue`              | 与 `--session-sqlite recover` 一起使用：准备一份经过净化处理的 openclaw/openclaw Issue 报告；在 `--yes` 或交互式确认后，Doctor 使用 `gh` 创建报告。 |
| `--json`              | 与 `--lint` 一起使用：JSON 格式的发现。与 `--post-upgrade` 一起使用：`{ probesRun, findings }`。与 `--state-sqlite` 或 `--session-sqlite` 一起使用：JSON 格式的维护报告。 |
| `--severity-min <level>`              | 与 `--lint` 一起使用：丢弃低于 `info`、`warning` 或 `error` 的发现。                                                                          |
| `--all`              | 与 `--lint` 一起使用：运行所有已注册的检查，包括默认集合中排除的选择加入型检查。                                                                                              |
| `--skip <id>`              | 与 `--lint` 一起使用：跳过一个检查 ID。可重复指定。                                                                                                                           |
| `--only <id>`              | 与 `--lint` 一起使用：仅运行给定的检查 ID。可重复指定。                                                                                                                       |

`--severity-min`、`--all`、`--only` 和 `--skip` 只能与 `--lint` 一起使用；`--json` 可与 `--lint`、`--post-upgrade`、`--state-sqlite` 和 `--session-sqlite` 一起使用。

## Lint 模式

`openclaw doctor --lint` 是只读的：不显示提示、不执行修复，也不重写配置/状态。

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --allow-exec
openclaw doctor --lint --only core/doctor/gateway-config --json
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

面向人工阅读的输出很简洁：

```text
doctor --lint：运行了 6 项检查，发现 1 个问题
  [warning] core/doctor/gateway-config gateway.mode - gateway.mode 未设置；Gateway 网关将无法启动。
    修复：运行 `openclaw configure` 并设置 Gateway 网关模式（local/remote），或运行 `openclaw config set gateway.mode local`。
```

JSON 输出是脚本接口：

```json
{
  "ok": false,
  "checksRun": 5,
  "checksSkipped": 0,
  "findings": [
    {
      "checkId": "core/doctor/gateway-config",
      "severity": "warning",
      "message": "gateway.mode 未设置；Gateway 网关将无法启动。",
      "path": "gateway.mode",
      "fixHint": "运行 `openclaw configure` 并设置 Gateway 网关模式（local/remote），或运行 `openclaw config set gateway.mode local`。"
    }
  ]
}
```

退出代码：

| 代码 | 含义                                                         |
| ---- | ------------------------------------------------------------ |
| `0` | 没有达到或超过所选严重性阈值的发现。                         |
| `1` | 至少有一个发现达到所选阈值。                                 |
| `2` | 在生成 Lint 发现之前发生命令/运行时故障。                     |

`--severity-min` 同时控制输出哪些发现以及退出阈值：即使存在严重性更低的 `info`/`warning` 发现，`openclaw doctor --lint --severity-min error` 也可能不输出任何内容并以 `0` 退出。

`--all` 控制在严重性筛选之前选择哪些检查。默认 Lint 运行会排除深度检查、历史检查，或更可能发现可修复旧版残留的检查；使用 `--all` 可运行完整清单。`--only <id>` 是最精确的选择器，可以按 ID 运行任意已注册的检查。

`core/doctor/local-audio-acceleration` 会报告自动选择的本地 STT 命令、相互独立的可用/请求/观测后端证据以及回退顺序，而无需加载语音模型。它会输出信息级别的发现，因此需要包含 `--severity-min info` 才能显示。

## 结构化健康检查

现代 Doctor 检查采用一个简洁的拆分式契约：

```ts
detect(ctx, scope?) -> HealthFinding[]
repair?(ctx, findings) -> HealthRepairResult
```

`detect()` 为 `doctor --lint` 提供支持。`repair()` 是可选的，并且仅在 `doctor --fix` / `doctor --repair` 下运行。尚未迁移到此结构的检查仍使用旧版 Doctor 贡献流程。

修复上下文可以携带 `dryRun`/`diff` 请求；修复结果可以返回结构化的 `diffs`（配置/文件编辑）和 `effects`（服务、进程、软件包、状态或其他副作用），因此转换后的检查可以逐步扩展到 `doctor --fix --dry-run`，而无需将变更规划移入 `detect()`。

`repair()` 报告 `status: "repaired" | "skipped" | "failed"`（省略状态表示 `repaired`）。当修复返回 `skipped` 或 `failed` 时，Doctor 会报告原因并跳过该检查的验证。成功修复后，Doctor 会重新运行限定于已修复发现项的 `detect()`；如果该发现项仍然存在，Doctor 会报告修复警告，而不会将该变更视为已完成。

发现项包括：

| 字段             | 用途                                                |
| ----------------- | ------------------------------------------------------ |
| `checkId`         | 用于 skip/only 筛选器和 CI 允许列表的稳定 ID。     |
| `severity`        | `info`、`warning` 或 `error`。                         |
| `message`         | 人类可读的问题陈述。                      |
| `path`            | 可用时的配置、文件或逻辑路径。          |
| `line` / `column` | 可用时的源位置。                        |
| `ocPath`          | 检查能够指向具体位置时的精确 `oc://` 地址。 |
| `fixHint`         | 建议的操作员操作或修复摘要。           |

现代化的核心 Doctor 检查仍附加到负责其面向用户的 `doctor` / `doctor --fix` 行为的有序 Doctor 贡献项。共享的结构化健康注册表是扩展点：内置检查和插件支持的检查由其所属软件包在活动命令路径中注册后，会在核心 Doctor 检查之后运行。`openclaw/plugin-sdk/health` 为插件作者公开相同的契约。

## 检查选择

```bash
openclaw doctor --lint --only core/doctor/gateway-config --json
openclaw doctor --lint --skip core/doctor/skills-readiness
openclaw doctor --lint --all --skip core/doctor/session-locks
```

`--only` 和 `--skip` 接受完整的检查 ID，并且可以重复指定。如果某个 `--only` ID 未注册，则不会为该 ID 运行任何检查；请使用输出中的 `checksRun`/`checksSkipped` 确认聚焦门禁选择了预期的检查。

## 升级后模式

`openclaw doctor --post-upgrade` 运行插件兼容性探测，适合串接在构建或升级之后。发现项输出到 stdout；如果任何发现项具有 `level: "error"`，退出码为 1。添加 `--json` 可获得机器可读的封装（`{ probesRun, findings }`），适用于 CI、社区 `fork-upgrade` skill 和其他升级后冒烟测试工具。如果已安装插件索引缺失或格式错误，JSON 模式仍会输出封装，其中包含一个 `plugin.index_unavailable` 错误发现项。

容器镜像启动是常规“更新后运行 Doctor”流程的例外。当 `openclaw gateway run` 在新版本 OpenClaw 上启动时，它会先运行安全的状态和插件修复，然后再报告就绪。如果无法安全完成修复，启动将退出，并提示你先针对相同的已挂载状态/配置，使用 `openclaw doctor --fix` 运行同一镜像一次，然后再正常重启容器。

## 旧版状态迁移

`openclaw doctor --fix` 是持久文件到 SQLite 迁移的唯一所有者。它会验证并认领每个已识别的来源，写入并验证规范行，记录迁移回执，然后删除已退役的来源。运行时代码不会执行延迟导入或回退读取。

这包括 `<state-dir>/mcp-oauth/*.json` 下已退役的 MCP OAuth 文件。修复前请停止 Gateway 网关。Doctor 会将有效凭据导入 `<state-dir>/state/openclaw.sqlite`；当两个存储同时存在时，保留现有的规范 SQLite 会话；删除过时的持久化 OAuth `state` 值；并使用迁移回执，防止重新创建的陈旧文件复活已注销的凭据。已退役的 `.lock` 边车会以失败关闭方式处理：如果 Doctor 报告陈旧所有者，请确认没有旧版 OpenClaw 进程正在运行，删除该边车，然后重新运行 Doctor。

## 共享状态 SQLite 压缩

有关架构版本控制、完整性检查和降级恢复，请参阅[数据库架构](/zh-CN/reference/database-schemas)。

`openclaw doctor --state-sqlite compact` 是对位于 `<state-dir>/state/openclaw.sqlite` 的规范共享状态数据库进行的显式离线维护。它不接受任意数据库路径，正常 Gateway 网关操作从不调用它，并且它不属于 `openclaw doctor --fix`。该命令会获取与 Gateway 网关启动相同的状态所有权锁，并在验证、检查点处理、`VACUUM` 和最终完整性检查期间持续持有该锁。当 Gateway 网关或另一个 SQLite 维护命令持有该锁时，它会拒绝运行。当 `OPENCLAW_ALLOW_MULTI_GATEWAY=1` 跳过每配置 Gateway 网关单例时，状态锁仍然有效，因此操作员 shell 无需继承 Gateway 网关服务的环境，维护操作也能检测到它。

请先停止 Gateway 网关并创建经过验证的备份：

```bash
openclaw gateway stop
openclaw backup create --verify
openclaw doctor --state-sqlite compact --json
openclaw gateway start
```

该命令：

1. 要求规范共享状态路径处存在常规文件。数据库缺失会报告为 `skipped`，并成功退出。
2. 在执行检查点处理或更改文件之前，验证当前支持的架构版本和 `schema_meta.role = "global"`。
3. 要求 `wal_checkpoint(TRUNCATE)` 不忙。如果检查点繁忙，请停止所有仍在运行的 OpenClaw 进程并重试。
4. 将 `auto_vacuum` 设置为 `INCREMENTAL`，运行完整的 `VACUUM`，然后再次执行检查点处理。
5. 运行 `quick_check`、`integrity_check` 和 `foreign_key_check`，然后对数据库和 SQLite 边车文件重新应用仅所有者权限。

JSON 输出会报告压缩前后的数据库和 WAL 大小、空闲列表页数、页面大小及 `auto_vacuum` 值，并报告回收的字节数以及 `quick_check` 和 `integrity_check` 结果。`foreign_key_check` 以失败关闭方式强制执行，并且没有单独的成功字段。SQLite 将 `auto_vacuum` 报告为：无时为 `0`，完整时为 `1`，增量时为 `2`。

如果架构过旧、比当前运行的 OpenClaw 构建更新，或属于 Agent 数据库，压缩会在不执行变更的情况下失败。对于较旧的共享状态架构，请先运行 `openclaw doctor --fix`。对于较新的架构，请恢复兼容的备份或升级 OpenClaw。

## 会话 SQLite 迁移

OpenClaw 会在 Gateway 网关启动期间以及运行 `openclaw doctor --fix` 时，自动将旧版会话行和转录历史导入每个 Agent 的 SQLite 数据库。`openclaw doctor --session-sqlite <mode>` 是用于该迁移的定向检查和验证工具。当前运行时会话行位于 `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`。旧版 `sessions.json` 文件是迁移来源。热转录 JSONL 文件在成功导入后会被导入并归档到活动会话目录之外；归档层 JSONL 文件仍是支持工件，而不是运行时回退来源。

模式：

| 模式       | 行为                                                                                                               |
| ---------- | ---------------------------------------------------------------------------------------------------------------------- |
| `inspect`  | 读取旧版和 SQLite 计数以及未引用的 JSONL 文件，但不导入。                                       |
| `dry-run`  | 解析旧版条目和转录 JSONL 文件，统计可导入行，并在不写入 SQLite 行的情况下报告问题。 |
| `import`   | 将旧版条目和转录事件导入所选目标的 SQLite。                                      |
| `validate` | 将所选旧版来源与 SQLite 行及转录事件计数进行比较。                                   |
| `compact`  | 对所选 Agent SQLite 数据库执行检查点处理和 VACUUM，以回收大量删除或归档清理后的空闲页面。    |
| `recover`  | 恢复最近一次失败的迁移运行、验证其目标，并准备经过清理的 GitHub issue 报告。            |
| `restore`  | 根据已记录的迁移清单恢复已归档的转录工件，而不删除 SQLite 数据。                  |

选择器：

- 默认：已配置的默认 Agent 存储（当该旧版存储文件存在时）。
- `--session-sqlite-agent <id>`：一个已配置的 Agent。
- `--session-sqlite-all-agents`：已配置的 Agent 存储以及已发现的 Agent 存储。
- `--session-sqlite-store <path>`：一个显式旧版 `sessions.json` 路径。

手动检查顺序：

```bash
openclaw doctor --session-sqlite inspect --session-sqlite-all-agents
openclaw doctor --session-sqlite dry-run --session-sqlite-all-agents --json
openclaw doctor --session-sqlite import --session-sqlite-all-agents
openclaw doctor --session-sqlite validate --session-sqlite-all-agents --json
openclaw doctor --session-sqlite compact --session-sqlite-all-agents
openclaw doctor --session-sqlite recover --github-issue
```

在包含重要历史记录的安装上运行 `import` 前，请备份 OpenClaw 状态目录。当所选旧版条目未出现在 SQLite 中、会话 ID 不同或转录事件计数不同时，`validate` 会以非零状态退出。使用 `--session-sqlite-store <path>` 时，请检查报告是否包含预期的目标数量；不存在的显式存储路径不会选择任何目标。

SQLite 删除操作会先回收数据库内部的页面；它们不一定会立即缩小数据库文件。删除或归档大型转录后，运行 `openclaw doctor --session-sqlite compact --session-sqlite-all-agents`，以对 WAL 文件执行检查点处理、运行 `VACUUM`，并报告数据库和 WAL 的前后大小。压缩要求目标为常规文件，使用当前 Agent 架构，具有所选 Agent 的持久所有者元数据，并且 Doctor 进程中没有打开的句柄。破坏性的 `import`、`compact`、`recover` 和 `restore` 模式会在整个操作期间持有与 Gateway 网关启动相同的状态所有权锁；`inspect`、`dry-run` 和 `validate` 保持只读且不会获取该锁。请先停止 Gateway 网关。破坏性模式会直接失败，而不会与实时写入或另一个维护命令发生竞争。破坏性的 `--session-sqlite-store` 目标必须位于活动状态目录内；在维护另一个安装之前，将 `OPENCLAW_STATE_DIR` 设置为该存储所属的状态目录。现有硬链接目标会被拒绝，因为锁定状态目录之外的另一路径可能共享同一个数据库 inode。相同的所有权检查也涵盖 SQLite WAL、共享内存和回滚日志边车。

每次导入都会先在 `~/.openclaw/session-sqlite-migration-runs/` 下写入清单，然后再将转录工件移入归档。如果工件移动后启动报告会话 SQLite 迁移失败，请运行恢复：

```bash
openclaw doctor --session-sqlite recover --github-issue
```

恢复操作会选择最新的失败迁移清单，仅还原清单中已归档的工件，验证受影响的目标，刷新经过净化处理的 `.failure.md` 和 `.failure.json` 报告，并准备一个 GitHub Issue 正文，其中不包含转录内容、原始环境信息、机密信息和无边界配置。当不存在失败的迁移清单，但所选 Agent 的 SQLite 数据库已损坏、不是数据库，或仅有日志边车文件而没有主数据库时，恢复操作会将完整文件集复制到临时检查目录。SQLite 可以在该一次性副本中回滚有效的热日志，然后再运行 `quick_check`、`integrity_check` 和 `foreign_key_check`，同时保持原始取证文件不变。完整性检查失败或存在孤立边车文件时，会通过使用同一个 `.corrupt-<timestamp>` 后缀重命名发现的整组文件，保留 DB、WAL、SHM 和回滚日志文件。如果捕获到重命名失败，则会先回滚已移动的文件，再报告失败，因此不会在不发出提示的情况下拆分可恢复的文件集。恢复前请停止 Gateway 网关；复制或重命名仍在动态变化的 SQLite 文件集并不安全，而且在不同操作系统上的行为也不同。使用 `--github-issue --yes` 时，Doctor 会通过 GitHub CLI 在 `openclaw/openclaw` 中创建 Issue；未经确认时，它会写入本地支持报告并输出预填充的 Issue URL。

`restore` 仍是更底层的撤销操作。它使用清单中的 `sourcePath -> archivePath` 记录，仅当原始路径缺失时才将已归档工件移回；当两个路径都存在时报告冲突，并将 SQLite 数据库保留在原位。

### 会话 SQLite 迁移后的降级

在启动较旧的文件存储型 OpenClaw 版本之前，请还原已归档的旧版转录工件：

```bash
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

旧版本会读取 `sessions.json` 条目以及这些条目中记录的 `sessionFile` 路径。完成 SQLite 迁移后，成功导入的热 JSONL 转录会被移入 `session-sqlite-import-archive/`，因此在还原操作将清单中记录的这些工件移回其原始路径之前，旧版运行时无法看到这段历史记录。

还原操作不会删除 SQLite 数据。在切换到 SQLite 后创建的会话仅存在于 SQLite 中，不会出现在旧版运行时中。如果之后再次升级，请运行上述正常迁移验证流程，以便 OpenClaw 在导入前比较已还原的旧版工件与 SQLite 行。

## 注意事项

- 在 Nix 模式（`OPENCLAW_NIX_MODE=1`）下，只读 Doctor 检查仍可运行，但 `doctor --fix`、`doctor --repair`、`doctor --yes` 和 `doctor --generate-gateway-token` 会被禁用，因为 `openclaw.json` 不可变。请改为编辑此安装的 Nix 源；对于 nix-openclaw，请使用 Agent 优先的[快速开始](https://github.com/openclaw/nix-openclaw#quick-start)。
- 交互式提示（钥匙串/OAuth 修复等）仅在 stdin 是 TTY 且**未**设置 `--non-interactive` 时运行。无头运行（cron、Telegram、无终端）会跳过提示。
- 非交互式 `doctor` 运行会跳过预先加载插件，以使无头健康检查保持快速。交互式会话仍会加载旧版健康检查/修复流程所需的插件界面。
- `--lint` 比 `--non-interactive` 更严格：始终只读、从不提示，也从不应用安全迁移。需要 Doctor 进行更改时，请使用 `doctor --fix` 或 `doctor --repair`。
- 默认情况下，Doctor 检查密钥时不会执行 `exec` SecretRef。仅当确实需要 Doctor 运行这些已配置的密钥解析器时，才使用 `--allow-exec`（可带或不带 `--lint`）。
- 任何配置写入（包括 `--fix` 修复）都会将备份轮换到 `~/.openclaw/openclaw.json.bak`（并使用编号为 `.bak.1`..`.bak.4` 的环形备份）。`--fix` 还会删除架构验证报告的未知配置键，并逐项列出删除内容；更新进行期间会跳过此操作，以免在迁移完成前清除部分写入的升级状态。
- 如果无法解析 `openclaw.json`，且无法恢复最后一个已知正常的配置，`doctor --fix` 会将原文件保留为 `openclaw.json.clobbered.<timestamp>`，保持当前文件不变，并报错退出，而不是写入不完整的替代文件。
- 当 Gateway 网关生命周期由其他监督程序管理时，请设置 `OPENCLAW_SERVICE_REPAIR_POLICY=external`。Doctor 仍会报告 Gateway 网关/服务健康状态并应用非服务修复，但会跳过服务安装、启动、重启、引导以及旧版服务清理。
- Doctor 会报告托管 Gateway 网关已应用的堆限制，以及根据当前主机或容器内存限制采用的自适应推导方式。在修复流程之外，可使用 `openclaw gateway status` 获取相同报告。
- 在 Linux 上，Doctor 会忽略未启用的额外类 Gateway 网关 systemd 单元，并且在修复期间不会重写正在运行的 systemd Gateway 网关服务的命令/入口点元数据。请先停止该服务，或使用 `openclaw gateway install --force` 替换活动启动器。
- `doctor --fix --non-interactive` 会报告缺失或过时的 Gateway 网关服务定义，但不会在更新修复模式之外安装或重写它们。若服务缺失，请运行 `openclaw gateway install`；若要替换启动器，请运行 `openclaw gateway install --force`。
- 状态完整性检查会检测会话目录中的孤立转录文件。将其归档为 `.deleted.<timestamp>` 需要交互式确认；`--fix`、`--yes` 和无头运行会将其保留在原处。
- Doctor 会扫描 `~/.openclaw/cron/jobs.json`（或 `cron.store`）中的旧版定时任务格式，在将规范行导入 SQLite 前重写它们。
- Doctor 会报告显式设置了 `payload.model` 覆盖项的定时任务，包括提供商命名空间计数及其与 `agents.defaults.model` 的不匹配情况，从而在调查身份验证或计费问题时显露未继承默认模型的定时任务。
- Doctor 会报告仍标记为执行中（`state.runningAtMs`）的定时任务，这可能导致 `openclaw cron list` 将其显示为 `running`。此检查为只读：如果当前没有 Gateway 网关正在执行被标记的任务，则定时任务服务下次启动时会记录被中断的运行并清除该标记。
- 在 Linux 上，如果用户的 crontab 仍在运行无人维护的旧版 `~/.openclaw/bin/ensure-whatsapp.sh`，Doctor 会发出警告；当 cron 缺少 systemd 用户总线环境时，它可能会错误报告 `Gateway inactive`。
- 启用 WhatsApp 后，Doctor 会检查 Gateway 网关事件循环是否已降级且本地 `openclaw-tui` 客户端仍在运行。`doctor --fix` 仅停止经过验证的本地 TUI 客户端，以免 WhatsApp 回复排在过时的 TUI 刷新循环之后。
- 当存在 HTTP(S) 代理环境变量但 `tools.web.fetch.useTrustedEnvProxy` 被禁用时，Doctor 会说明 `web_fetch` 仍使用直接路由，执行一次简短的直接 TLS 连接探测，并指出需要显式启用的选项。它绝不会自动启用代理信任。
- Doctor 会在主模型、回退模型、模型允许列表、图像/视频生成模型、Heartbeat/子智能体/压缩覆盖项、Hooks、渠道模型覆盖项、定时任务载荷以及过时的会话/转录路由固定项中，将旧版 `codex/*` 和 `openai-codex/*` 模型引用重写为规范的 `openai/*` 引用。在安全的情况下，`--fix` 还会合并旧版 `models.providers.codex` 和 `models.providers.openai-codex` 配置，将旧版 `openai-codex:*` 身份验证配置文件和 `auth.order.openai-codex` 条目迁移到 `openai:*`，将 Codex 意图移至提供商/模型范围的 `agentRuntime.id: "codex"` 条目，移除过时的完整智能体/会话运行时固定项，并让修复后的 OpenAI 智能体引用继续使用 Codex 身份验证路由，而不是直接使用 OpenAI API 密钥身份验证。
- 当非空的 `auth.order.<provider>` 列表所引用的配置文件已全部不存在，但仍有兼容的已存储凭据时，Doctor 会进行报告。`doctor --fix` 仅删除这些过时的覆盖项，以恢复自动的按智能体凭据选择；显式空顺序、部分仍有效的列表，以及不存在兼容已存储凭据的顺序均保持不变。如果活动 SQLite 身份验证存储无法读取或格式错误，Doctor 会说明跳过此修复的原因。如果正在运行的 Gateway 网关的配置重载模式不会自动应用写入，请在重新检查身份验证状态前重启该 Gateway 网关。
- Doctor 会清理旧版 OpenClaw 遗留的插件依赖暂存状态，并为将主机 `openclaw` 包声明为对等依赖的托管 npm 插件重新建立该包的链接。它还会修复配置引用的缺失可下载插件（`plugins.entries`、已配置的渠道、已配置的提供商/搜索设置、已配置的 Agent Runtimes）。在包更新期间，Doctor 会跳过包管理器插件修复，直至包替换完成；之后，如果已配置的插件仍需恢复，请重新运行 `openclaw doctor --fix`。如果下载失败，Doctor 会报告安装错误，并保留已配置的插件条目以供下次修复尝试。
- 当插件发现功能正常时，Doctor 会通过从 `plugins.allow`/`plugins.deny`/`plugins.entries` 中移除缺失的插件 ID，以及匹配的悬空渠道配置、Heartbeat 目标和渠道模型覆盖项，修复过时的插件配置。
- Doctor 会隔离无效的插件配置，方法是禁用受影响的 `plugins.entries.<id>` 条目并移除其无效的 `config` 载荷。Gateway 网关启动时本就只会跳过该问题插件，因此其他插件和渠道会继续运行。
- Doctor 会移除已停用的 `plugins.entries.codex.config.codexDynamicToolsProfile`；Codex app-server 始终将 Codex 原生工作区工具保留为原生工具。
- Doctor 会自动将旧版扁平 Talk 配置（`talk.voiceId`、`talk.modelId` 等）迁移到 `talk.provider` + `talk.providers.<provider>`。当唯一差异仅为对象键顺序时，重复运行 `doctor --fix` 不再报告/应用 Talk 规范化。
- Doctor 包含记忆搜索就绪情况检查，并可在缺少嵌入凭据时建议使用 `openclaw configure --section model`。
- 未配置命令所有者时，Doctor 会发出警告。命令所有者是获准运行仅限所有者的命令并批准危险操作的人类操作员账户。私信配对只能允许某人与 Bot 对话；如果在首次所有者引导功能出现之前批准过发送者，请显式设置 `commands.ownerAllowFrom`。
- 配置了 Codex 模式智能体且操作员的 Codex 主目录中存在个人 Codex CLI 资产时，Doctor 会报告一条信息提示。本地 Codex app-server 启动使用隔离的按智能体主目录；如有需要，请先安装 Codex 插件，然后使用 `openclaw migrate plan codex` 清点应有意提升的资产。
- 如果默认智能体允许使用的 Skills 在当前运行时环境中不可用（缺少二进制文件、环境变量、配置或操作系统要求），Doctor 会发出警告。`doctor --fix` 可通过 `skills.entries.<skill>.enabled=false` 禁用这些不可用的 Skills；如果需要保持 Skills 启用，请改为安装/配置缺失的要求。
- 如果已启用沙箱模式但 Docker 不可用，Doctor 会报告一条信号明确的警告，并提供修复方法（`install Docker` 或 `openclaw config set agents.defaults.sandbox.mode off`）。
- 如果存在旧版沙箱注册表文件或分片目录（`~/.openclaw/sandbox/containers.json`、`~/.openclaw/sandbox/browsers.json`、`~/.openclaw/sandbox/containers/` 或 `~/.openclaw/sandbox/browsers/`），Doctor 会报告它们；`--fix` 会将有效条目迁移到 SQLite，并隔离无效的旧版文件。
- 如果 `gateway.auth.token`/`gateway.auth.password` 由 SecretRef 管理，且在当前命令路径中不可用，Doctor 会报告只读警告，且不会写入明文回退凭据。对于由 Exec 支持的 SecretRef，除非存在 `--allow-exec`，否则 Doctor 会跳过执行。
- 如果在修复路径中检查渠道 SecretRef 失败，Doctor 会继续运行并报告警告，而不是提前退出。
- 完成状态目录迁移后，如果已启用的默认 Telegram 或 Discord 账户依赖环境变量回退，而 Doctor 进程无法使用 `TELEGRAM_BOT_TOKEN` 或 `DISCORD_BOT_TOKEN`，Doctor 会发出警告。
- Telegram `allowFrom` 用户名自动解析（`doctor --fix`）要求当前命令路径中存在可解析的 Telegram 令牌。如果无法检查令牌，Doctor 会报告警告，并在本次运行中跳过自动解析。

## macOS：`launchctl` 环境变量覆盖

如果之前运行过 `launchctl setenv OPENCLAW_GATEWAY_TOKEN ...`（或 `...PASSWORD`），该值会覆盖配置文件，并可能导致持续出现“unauthorized”错误。

```bash
launchctl getenv OPENCLAW_GATEWAY_TOKEN
launchctl getenv OPENCLAW_GATEWAY_PASSWORD

launchctl unsetenv OPENCLAW_GATEWAY_TOKEN
launchctl unsetenv OPENCLAW_GATEWAY_PASSWORD
```

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [Gateway 网关 Doctor](/zh-CN/gateway/doctor)
