---
read_when:
    - 诊断数据库架构版本过新的错误
    - 更新或降级前检查数据库兼容性
    - 为旧版 OpenClaw 恢复数据库
summary: OpenClaw SQLite 数据库位置、架构版本、完整性检查和降级恢复
title: 数据库架构
x-i18n:
    generated_at: "2026-07-26T06:00:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 73993e2c593ba460784108aedef70bbfb499e525c709d6d6bdd956ccf93e0ddc
    source_path: reference/database-schemas.md
    workflow: 16
---

OpenClaw 将控制平面状态存储在全局 SQLite 数据库中，并将智能体数据存储在每个智能体各自的 SQLite 数据库中。数据库打开时会执行前向模式迁移。旧版 OpenClaw 构建会拒绝由较新模式写入的数据库。

## 数据库布局

| 范围           | 默认路径                           | 内容                                                                           |
| -------------- | ---------------------------------- | ------------------------------------------------------------------------------ |
| 全局控制平面   | `~/.openclaw/state/openclaw.sqlite`                 | 共享配置状态、注册表、审批、插件状态和共享运行时状态                           |
| 每智能体数据平面 | `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`                 | 会话、对话记录、记忆索引、身份验证状态、对话状态和智能体范围内的运行时状态     |

少数数据量较大或具有特定生命周期的功能使用专用 SQLite 存储，包括任务注册表和轨迹数据。

## 版本控制契约

每个数据库在两个位置记录其模式：

- `PRAGMA user_version` 是 SQLite 模式版本。
- 主 `schema_meta` 行记录 `role`、`agent_id`、`schema_version` 和 `app_version`。`app_version` 是最后写入模式元数据的 OpenClaw 构建。

OpenClaw 打开受支持的旧数据库时，会应用仅向前迁移。对于 `user_version` 高于当前运行构建所支持版本的数据库，它会拒绝打开并报告 `newer schema version` 错误。Gateway 网关在启动前会检查所有已注册的数据库。`openclaw update` 也会拒绝声明的模式支持版本低于磁盘数据库版本的软件包或源码目标。在添加模式元数据之前发布的目标软件包无法进行预检。

通过 npm 手动安装 OpenClaw 会绕过更新程序的防护。数据库打开检查仍会拒绝不兼容的构建。

## 智能体模式历史

| 版本 | 变更                                                                                                                                                                                                                                                           | 首个发布版本                                    |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| 1    | 初始每智能体存储（[#88349](https://github.com/openclaw/openclaw/pull/88349)）                                                                                                                                                                                  | `v2026.5.30-beta.1`，稳定延续至 `v2026.7.1` |
| 2    | 记忆索引标识（[#104449](https://github.com/openclaw/openclaw/pull/104449)）                                                                                                                                                                                     | `v2026.7.2-beta.1`                              |
| 4    | 会话和对话记录迁移至 SQLite（[#98236](https://github.com/openclaw/openclaw/pull/98236)）                                                                                                                                                                       | `v2026.7.2-beta.1`                              |
| 5-6  | 终端新鲜度和状态生命周期（[#104859](https://github.com/openclaw/openclaw/pull/104859)）                                                                                                                                                                        | `v2026.7.2-beta.1`                              |
| 7    | 每条目生命周期状态投影（[#106151](https://github.com/openclaw/openclaw/pull/106151)）                                                                                                                                                                          | `v2026.7.2-beta.1`                              |
| 8    | 每份对话记录的会话来源（[#106766](https://github.com/openclaw/openclaw/pull/106766)）                                                                                                                                                                          | `v2026.7.2-beta.2`                              |
| 9    | `STRICT` 表（[#108663](https://github.com/openclaw/openclaw/pull/108663)）                                                                                                                                                                           | `v2026.7.2-beta.2`                              |
| 10   | 物化的活跃对话记录路径（[#108851](https://github.com/openclaw/openclaw/pull/108851)）                                                                                                                                                                          | 未发布                                          |
| 11   | 租约、持久化投递、对话地址和 Heartbeat 结果（[#109636](https://github.com/openclaw/openclaw/pull/109636)、[#95838](https://github.com/openclaw/openclaw/pull/95838)、[#109999](https://github.com/openclaw/openclaw/pull/109999)） | 未发布                                          |

版本 3 是未发布的开发步骤，已并入版本 4。

## 状态模式历史

| 版本 | 变更                                                                                                         | 首个发布版本      |
| ---- | ------------------------------------------------------------------------------------------------------------ | ----------------- |
| 1    | 初始共享状态数据库                                                                                           | `v2026.5.30-beta.1` |
| 2    | 仅含元数据的消息审计事件（[#103903](https://github.com/openclaw/openclaw/pull/103903)）                       | `v2026.7.2-beta.1` |
| 3    | `STRICT` 表和模式漂移强化（[#108663](https://github.com/openclaw/openclaw/pull/108663)）            | `v2026.7.2-beta.2` |
| 4    | 使用会话监视来源替代编码的哨兵行                                                                             | 未发布            |

## 完整性检查

| 时机                           | 检查                                                               |
| ------------------------------ | ------------------------------------------------------------------ |
| 每次打开                       | 验证 `schema_meta` 表和主元数据行                             |
| 执行待处理迁移之前             | 运行完整的完整性、外键、角色、模式和索引扫描                       |
| Gateway 网关后台验证程序       | 大约每天运行一次完整扫描并记录结果                                 |
| Doctor、备份验证和压缩         | 接受或重写数据库之前运行完整扫描                                   |

Gateway 网关预检仅读取模式头。对于不需要迁移的数据库，由后台验证程序负责较慢的完整扫描。
隔离决策仅存放在专用的 `openclaw-quarantine.sqlite` 存储中，因此即使被隔离的数据库损坏，这些决策仍会保留。验证结果会记录到日志中。

## 故障排查

### 为什么更新至 2026.7.2 后无法回退

截至 `v2026.7.1` 的每个版本均使用智能体模式 1 和状态模式 1。2026.7.2 发布系列（从 `v2026.7.2-beta.1` 开始）会在首次启动时向前迁移数据库。此迁移是单向的：数据会被重写为较新的模式，之后安装旧版 OpenClaw 并不会撤销迁移。旧版构建会拒绝启动，并报告 `newer schema version` 错误，其中会指出拥有该数据库的构建。

降级二进制文件绝不会降级数据。如果更新后必须运行早于 2026.7.2 的版本，有以下三种选择：

1. 恢复更新前创建的备份。在重大更新前[创建并验证备份](/zh-CN/cli/backup)。
2. 让旧版构建使用单独的状态目录（`OPENCLAW_STATE_DIR`）。它会从全新状态启动；迁移后的数据保持不变，以便返回较新构建时继续使用。
3. 按照下方的手动降级步骤操作。此操作不受支持，且在没有经过验证的备份时可能导致数据丢失。

自 2026.7.2 起，`openclaw update` 会拒绝安装无法打开当前数据库的版本，因此更新程序不会使系统陷入这种情况。通过 npm 手动安装旧版本会绕过此防护；数据库仍会拒绝旧版二进制文件，但只会在其安装完成后才拒绝。

### Gateway 网关因较新的模式版本错误而拒绝启动

较新的 OpenClaw 构建写入了数据库，而当前运行的构建较旧。错误和 Gateway 网关启动日志会指出拥有该数据库的构建（`app_version`）。安装该版本或更新版本，或者使用上方的选项之一。不要通过编辑数据库来消除该错误。

### 完整性验证失败后数据库被隔离

后台验证程序已证实该文件损坏，此后每次打开都会立即失败，而不再重新扫描。请从备份恢复或修复数据库，然后运行 `openclaw doctor --fix` 清除隔离记录。如果无法清除隔离记录本身，Doctor 会报告明确错误；请反复运行，直到其报告状态正常。

## 不支持降级

手动降级模式仅适用于愿意承担风险的智能体和操作员。编辑任何数据库之前，请先[创建并验证备份](/zh-CN/cli/backup)。停止 Gateway 网关以及所有可能打开该数据库的进程。

一般步骤如下：

1. 阅读目标版本的模式和迁移。
2. 在单个事务中，删除目标版本之后引入的所有表、索引、触发器和列。
3. 将 `PRAGMA user_version` 和 `schema_meta.schema_version` 设置为目标版本。
4. 启动 Gateway 网关之前，运行目标版本的完整数据库验证。

### 示例：将智能体模式 11 降级至 9

模式 10 添加了活跃对话记录投影。模式 11 添加了租约、持久化投递、对话地址状态和 Heartbeat 结果。QMD 协调使用 `state_leases` 中的行，不存在需要保留的独立 QMD 表。

检查写入数据库的确切模式后，对每个受影响的每智能体数据库运行等效 SQL：

```sql
BEGIN IMMEDIATE;

DROP TABLE IF EXISTS heartbeat_outcomes;
DROP TABLE IF EXISTS conversation_deliveries;
DROP TABLE IF EXISTS state_leases;
DROP TABLE IF EXISTS session_transcript_active_events;

ALTER TABLE session_transcript_index_state DROP COLUMN active_event_count;
ALTER TABLE session_transcript_index_state DROP COLUMN active_message_count;
ALTER TABLE conversations DROP COLUMN delivery_target;

PRAGMA user_version = 9;
UPDATE schema_meta
SET schema_version = 9,
    updated_at = unixepoch('now') * 1000
WHERE meta_key = 'primary';

COMMIT;
```

这会丢弃版本 10-11 的状态，包括正在进行的投递操作、租约、Heartbeat 结果以及派生的活跃对话记录投影。降级操作一旦出错，就需要从经过验证的备份恢复。
