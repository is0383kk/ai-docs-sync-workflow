---
read_when:
    - バックグラウンドタスクの記録を確認、監査、またはキャンセルしたい場合
    - '`openclaw tasks flow` 配下の Task Flow コマンドについて説明しています'
summary: '`openclaw tasks` の CLI リファレンス（バックグラウンドタスク台帳と Task Flow の状態）'
title: '`openclaw tasks`'
x-i18n:
    generated_at: "2026-07-26T10:09:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b03a4aa9fab12b6e5773259a76a1e89fd6e6398c73e5b0533a31e5e3a3894f9c
    source_path: cli/tasks.md
    workflow: 16
---

永続的なバックグラウンドタスクと Task Flow の状態を確認します。サブコマンドを指定しない場合、
`openclaw tasks` は `openclaw tasks list` と同等です。

ライフサイクルと配信モデルについては [バックグラウンドタスク](/ja-JP/automation/tasks)を、すべての検出結果の説明についてはその `tasks audit` セクションを参照してください。

## 使用方法

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

## ルートオプション

| フラグ               | 説明                                                                                        |
| ------------------ | -------------------------------------------------------------------------------------------------- |
| `--json`           | JSON を出力します。                                                                                       |
| `--runtime <name>` | 種類で絞り込みます：`subagent`、`acp`、`cron`、または `cli`。                                               |
| `--status <name>`  | ステータスで絞り込みます：`queued`、`running`、`succeeded`、`failed`、`timed_out`、`cancelled`、または `lost`。 |

## サブコマンド

### `list`

```bash
openclaw tasks list [--runtime <name>] [--status <name>] [--json]
```

追跡中のバックグラウンドタスクを新しい順に一覧表示します。

### `show`

```bash
openclaw tasks show <lookup> [--json]
```

タスク ID、実行 ID、またはセッションキーを指定して、1 件のタスクを表示します。

### `notify`

```bash
openclaw tasks notify <lookup> <done_only|state_changes|silent>
```

実行中のタスクの通知ポリシーを変更します。

### `cancel`

```bash
openclaw tasks cancel <lookup>
```

実行中のバックグラウンドタスクをキャンセルします。

### `audit`

```bash
openclaw tasks audit [--severity <warn|error>] [--code <name>] [--limit <n>] [--json]
```

古い、消失した、配信に失敗した、またはその他の不整合があるタスクと
Task Flow のレコードを明らかにします。`cleanupAfter` まで保持される消失タスクは警告となり、
期限切れまたはスタンプのない消失タスクはエラーとなります。

`--code` には、タスクコード（`stale_queued`、`stale_running`、`lost`、
`delivery_failed`、`missing_cleanup`、`inconsistent_timestamps`）および Task
Flow コード（`restore_failed`、`stale_waiting`、`stale_blocked`、
`cancel_stuck`、`missing_linked_tasks`、`blocked_task_missing`）を指定できます。コードごとの重大度と
トリガーの詳細については、[バックグラウンドタスク](/ja-JP/automation/tasks)を参照してください。

### `maintenance`

```bash
openclaw tasks maintenance [--apply] [--json]
```

タスクと Task Flow の整合化、クリーンアップのスタンプ付与、
プルーニング、および古い Cron 実行セッションレジストリのクリーンアップをプレビューまたは適用します。

Cron タスクでは、古いアクティブタスクを `lost` としてマークする前に、
永続化された実行ログとジョブ状態を使用して整合化するため、メモリ内の Gateway ランタイム状態が
消失しただけで、完了済みの Cron 実行が誤って監査エラーになることはありません。
オフライン CLI 監査は、Gateway のプロセスローカルな Cron アクティブジョブセットに対する
信頼できる情報源ではありません。実行 ID／ソース ID を持つ CLI タスクは、古い子セッション行が
残っていても、稼働中の Gateway 実行コンテキストが消失すると `lost` としてマークされます。

適用した場合、メンテナンスは、現在実行中の Cron ジョブを保持し、Cron 以外のセッション行には
変更を加えずに、7 日より古い `cron:<jobId>:run:<uuid>` セッションレジストリ行もプルーニングします。

### `flow`

```bash
openclaw tasks flow list [--status <name>] [--json]
openclaw tasks flow show <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

タスク台帳にある永続的な Task Flow の状態を確認またはキャンセルします。
`flow list --status` には、`queued`、`running`、`waiting`、`blocked`、
`succeeded`、`failed`、`cancelled`、または `lost` を指定できます。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [バックグラウンドタスク](/ja-JP/automation/tasks)
