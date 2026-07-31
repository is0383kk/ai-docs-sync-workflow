---
read_when:
    - 推定されたフォローアップの約束を確認したい場合
    - 保留中のチェックインを破棄する場合
    - Heartbeat が配信する可能性のある内容を監査しています
summary: '`openclaw commitments` の CLI リファレンス（推測されたフォローアップの確認と破棄）'
title: '`openclaw commitments`'
x-i18n:
    generated_at: "2026-07-26T09:15:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4a7c573daad6a9bc6ce4532514c8cc22b3c510b4fc0cf9d1a79048413f08c1a2
    source_path: cli/commitments.md
    workflow: 16
---

廃止された推論コミットメント実験によって残されたレコードを確認し、破棄します。
OpenClaw は新しいコミットメントの作成や配信を行わなくなりましたが、アップグレード時に既存の SQLite 行を監査してクリーンアップできるよう、メンテナンスコマンドは維持されています。

サブコマンドを指定しない場合、`openclaw commitments` は保留中のコミットメントを一覧表示します。

## 使用方法

```bash
openclaw commitments [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments list [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments dismiss <id...> [--json]
```

## オプション

- `--all`: 保留中のコミットメントだけでなく、すべてのステータスを表示します。
- `--agent <id>`: 1 つのエージェント ID に絞り込みます。
- `--status <status>`: ステータスで絞り込みます。値: `pending`、`sent`、
  `dismissed`、`snoozed`、または `expired`。不明な値を指定すると、エラーで終了します。
- `--json`: 機械可読な JSON を出力します。

`dismiss` は、指定されたコミットメント ID を `dismissed` としてマークします。

## 例

保留中のコミットメントを一覧表示します。

```bash
openclaw commitments
```

保存されているすべてのコミットメントを一覧表示します。

```bash
openclaw commitments --all
```

1 つのエージェントに絞り込みます。

```bash
openclaw commitments --agent main
```

スヌーズされたコミットメントを検索します。

```bash
openclaw commitments --status snoozed
```

1 つ以上のコミットメントを破棄します。

```bash
openclaw commitments dismiss cm_abc123 cm_def456
```

JSON としてエクスポートします。

```bash
openclaw commitments --all --json
```

## 出力

テキスト出力には、コミットメント数、共有 SQLite データベースのパス、適用中のフィルター、
およびコミットメントごとの行が表示されます。

- コミットメント ID
- ステータス
- 種類（`event_check_in`、`deadline_check`、`care_check_in`、または `open_loop`）
- 最も早い期限
- スコープ（エージェント/チャンネル/ターゲット）
- 推奨されるチェックインテキスト

JSON 出力には、件数、適用中のステータスおよびエージェントフィルター、
共有 SQLite データベースのパス、および保存されている完全なレコードが含まれます。

## 関連項目

- [推論コミットメント](/ja-JP/concepts/commitments)
- [メモリの概要](/ja-JP/concepts/memory)
- [Heartbeat](/ja-JP/gateway/heartbeat)
- [スケジュールされたタスク](/ja-JP/automation/cron-jobs)
