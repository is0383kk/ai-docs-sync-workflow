---
read_when:
    - 你想要檢查、稽核或取消背景工作記錄
    - 你正在記錄 `openclaw tasks flow` 下的 TaskFlow 命令
summary: '`openclaw tasks` 的命令列介面參考（背景任務帳本與 Task Flow 狀態）'
title: '`openclaw tasks`'
x-i18n:
    generated_at: "2026-07-26T08:29:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b03a4aa9fab12b6e5773259a76a1e89fd6e6398c73e5b0533a31e5e3a3894f9c
    source_path: cli/tasks.md
    workflow: 16
---

檢查持久化背景工作與 Task Flow 狀態。未指定子命令時，
`openclaw tasks` 等同於 `openclaw tasks list`。

如需瞭解生命週期與傳遞模型，請參閱[背景工作](/zh-TW/automation/tasks)；如需完整的發現項目說明，請參閱其中的 `tasks audit` 章節。

## 用法

```bash
openclaw tasks
openclaw tasks list
openclaw tasks list --runtime acp
openclaw tasks list --status running
openclaw tasks show <lookup>
openclaw tasks notify <lookup> state_changes
openclaw tasks cancel <lookup>
openclaw tasks audit
openclaw tasks maintenance
openclaw tasks maintenance --apply
openclaw tasks flow list
openclaw tasks flow show <lookup>
openclaw tasks flow cancel <lookup>
```

## 根層級選項

| 旗標               | 說明                                                                                        |
| ------------------ | -------------------------------------------------------------------------------------------------- |
| `--json`           | 輸出 JSON。                                                                                       |
| `--runtime <name>` | 依種類篩選：`subagent`、`acp`、`cron` 或 `cli`。                                               |
| `--status <name>`  | 依狀態篩選：`queued`、`running`、`succeeded`、`failed`、`timed_out`、`cancelled` 或 `lost`。 |

## 子命令

### `list`

```bash
openclaw tasks list [--runtime <name>] [--status <name>] [--json]
```

依最新到最舊列出追蹤中的背景工作。

### `show`

```bash
openclaw tasks show <lookup> [--json]
```

依工作 ID、執行 ID 或工作階段金鑰顯示一項工作。

### `notify`

```bash
openclaw tasks notify <lookup> <done_only|state_changes|silent>
```

變更執行中工作的通知原則。

### `cancel`

```bash
openclaw tasks cancel <lookup>
```

取消執行中的背景工作。

### `audit`

```bash
openclaw tasks audit [--severity <warn|error>] [--code <name>] [--limit <n>] [--json]
```

顯示過時、遺失、傳遞失敗或其他不一致的工作與
Task Flow 記錄。保留至 `cleanupAfter` 的遺失工作會列為警告；
已到期或未加上時間戳記的遺失工作則會列為錯誤。

`--code` 接受工作代碼（`stale_queued`、`stale_running`、`lost`、
`delivery_failed`、`missing_cleanup`、`inconsistent_timestamps`）以及 Task
Flow 代碼（`restore_failed`、`stale_waiting`、`stale_blocked`、
`cancel_stuck`、`missing_linked_tasks`、`blocked_task_missing`）。如需瞭解各代碼的嚴重性與觸發條件詳細資訊，請參閱
[背景工作](/zh-TW/automation/tasks)。

### `maintenance`

```bash
openclaw tasks maintenance [--apply] [--json]
```

預覽或套用工作與 Task Flow 的調整、清理標記、
修剪，以及過時的排程執行工作階段登錄檔清理。

對於排程工作，調整程序會先使用持久化的執行日誌／作業狀態，再將舊的作用中工作標記為
`lost`，因此已完成的排程執行不會只因記憶體中的閘道執行階段狀態已消失，
就變成錯誤的稽核錯誤。離線命令列介面稽核對閘道程序本機的排程作用中作業集合
不具權威性。若命令列介面工作具有執行 ID／來源 ID，當其即時閘道執行內容已消失時，
即使仍有舊的子工作階段資料列，也會標記為 `lost`。

套用後，維護也會修剪超過 7 天的 `cron:<jobId>:run:<uuid>` 工作階段
登錄資料列，同時保留目前執行中的排程作業，且不變更非排程工作階段資料列。

### `flow`

```bash
openclaw tasks flow list [--status <name>] [--json]
openclaw tasks flow show <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

檢查或取消工作帳本下的持久化 Task Flow 狀態。
`flow list --status` 接受 `queued`、`running`、`waiting`、`blocked`、
`succeeded`、`failed`、`cancelled` 或 `lost`。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [背景工作](/zh-TW/automation/tasks)
