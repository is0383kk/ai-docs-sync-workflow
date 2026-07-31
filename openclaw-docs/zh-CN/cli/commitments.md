---
read_when:
    - 你想检查推断出的跟进承诺
    - 你想忽略待处理的签到请求
    - 你正在审核 Heartbeat 可能交付的内容
summary: '`openclaw commitments` 的 CLI 参考（检查并取消推断式跟进）'
title: '`openclaw commitments`'
x-i18n:
    generated_at: "2026-07-26T06:09:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4a7c573daad6a9bc6ce4532514c8cc22b3c510b4fc0cf9d1a79048413f08c1a2
    source_path: cli/commitments.md
    workflow: 16
---

检查并清除已停用的推断式跟进承诺实验所留下的记录。
OpenClaw 不再创建或交付新的跟进承诺，但仍保留维护命令，
以便升级时审计并清理现有的 SQLite 行。

不带子命令时，`openclaw commitments` 会列出待处理的跟进承诺。

## 用法

```bash
openclaw commitments [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments list [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments dismiss <id...> [--json]
```

## 选项

- `--all`：显示所有状态，而非仅显示待处理的跟进承诺。
- `--agent <id>`：按一个 Agent ID 筛选。
- `--status <status>`：按状态筛选。可选值：`pending`、`sent`、
  `dismissed`、`snoozed` 或 `expired`。未知值会导致程序报错退出。
- `--json`：输出机器可读的 JSON。

`dismiss` 将给定的跟进承诺 ID 标记为 `dismissed`。

## 示例

列出待处理的跟进承诺：

```bash
openclaw commitments
```

列出所有已存储的跟进承诺：

```bash
openclaw commitments --all
```

筛选到一个 Agent：

```bash
openclaw commitments --agent main
```

查找已延后的跟进承诺：

```bash
openclaw commitments --status snoozed
```

清除一个或多个跟进承诺：

```bash
openclaw commitments dismiss cm_abc123 cm_def456
```

导出为 JSON：

```bash
openclaw commitments --all --json
```

## 输出

文本输出会显示跟进承诺数量、共享 SQLite 数据库路径、所有生效的筛选条件，
并为每项跟进承诺输出一行：

- 跟进承诺 ID
- 状态
- 类型（`event_check_in`、`deadline_check`、`care_check_in` 或 `open_loop`）
- 最早到期时间
- 范围（Agent/渠道/目标）
- 建议的跟进文本

JSON 输出包含数量、生效的状态和 Agent 筛选条件、
共享 SQLite 数据库路径以及完整的存储记录。

## 相关内容

- [推断式跟进承诺](/zh-CN/concepts/commitments)
- [记忆概览](/zh-CN/concepts/memory)
- [Heartbeat](/zh-CN/gateway/heartbeat)
- [定时任务](/zh-CN/automation/cron-jobs)
