---
read_when:
    - SSH を使わずに Gateway のログをリモートで追跡する必要があります
    - ツール用の JSON ログ行が必要な場合
summary: '`openclaw logs` の CLI リファレンス（RPC 経由で Gateway ログを追跡）'
title: ログ
x-i18n:
    generated_at: "2026-07-26T08:57:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7c8dc40e70f2eb4f8d6ba8b75b91a33337786a146abbe401079ee374daa5a0c6
    source_path: cli/logs.md
    workflow: 16
---

# `openclaw logs`

RPC 経由で Gateway のファイルログを追尾します。リモートモードでも動作します。

## オプション

- `--limit <n>`: 返すログ行の最大数（デフォルト: `200`）
- `--max-bytes <n>`: ログファイルから読み取る最大バイト数（デフォルト: `250000`）
- `--follow`: ログストリームを追尾する
- `--interval <ms>`: 追尾中のポーリング間隔（デフォルト: `1000`）
- `--json`: 行区切りの JSON イベントを出力する
- `--plain`: スタイル付き書式を使用せずプレーンテキストで出力する
- `--no-color`: ANSI カラーを無効にする
- `--local-time`: タイムスタンプをローカルタイムゾーンで表示する（デフォルト）
- `--utc`: タイムスタンプを UTC で表示する

## 共通の Gateway RPC オプション

- `--url <url>`: Gateway WebSocket URL
- `--token <token>`: Gateway トークン
- `--timeout <ms>`: タイムアウト（ミリ秒、デフォルト: `30000`）
- `--expect-final`: Gateway 呼び出しがエージェントによって処理される場合、最終応答を待つ

`--url` を渡すと、自動的に適用される設定済みの認証情報がスキップされます。対象の Gateway で認証が必要な場合は、`--token` を明示的に含めてください。

## 例

```bash
openclaw logs
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
openclaw logs --follow --interval 2000
openclaw logs --limit 500 --max-bytes 500000
openclaw logs --json
openclaw logs --plain
openclaw logs --no-color
openclaw logs --utc
openclaw logs --follow --local-time
openclaw logs --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

選択されたルートプロファイルは、Gateway のローテーションファイルと一致します。デフォルト
プロファイルでは `openclaw-YYYY-MM-DD.log` を使用し、名前付きプロファイルでは
`openclaw-<profile>-YYYY-MM-DD.log` を使用します（例:
`openclaw-dev-YYYY-MM-DD.log`）。

## フォールバックと復旧の動作

- 暗黙的な local loopback Gateway がペアリングを要求した場合、接続中に切断された場合、または `logs.tail` が応答する前にタイムアウトした場合、`openclaw logs` は設定済みの Gateway ファイルログへ自動的にフォールバックします。明示的な `--url` ターゲットでは、このフォールバックは使用されません。
- `--follow` は、暗黙的なローカル Gateway RPC の失敗後に、その設定済みファイルへフォールバックしません。並存する古いファイルによってライブ追尾が誤解を招く可能性があるためです。Linux では代わりに、利用可能な場合は PID に基づいてアクティブなユーザー systemd Gateway ジャーナルを使用し（選択したソースを表示します）、それ以外の場合はライブ Gateway への再試行を続けます。
- `--follow` 中に一時的な切断（WebSocket の切断、タイムアウト、接続断）が発生すると、指数バックオフによる自動再接続が実行されます。再試行は最大 8 回で、試行間隔の上限は 30 秒です。再試行のたびに警告が stderr に出力され、ポーリングが成功すると `[logs] gateway reconnected` 通知が一度出力されます。`--json` モードでは、どちらも stderr に `{"type":"notice"}` レコードとして出力されます。復旧不可能なエラー（認証失敗、不正な設定）では、引き続き即座に終了します。
- `--follow --json` モードでは、ログソースの遷移が `{"type":"meta"}` レコードとして出力されます。`sourceKind` ごとにカーソルを追跡してください。ストリームは、Gateway ファイル出力（`sourceKind: "file"`）からローカルジャーナルへのフォールバック（`sourceKind: "journal"`、`localFallback: true`、`service.pid`/`service.unit` を伴う）へ移行し、復旧後に Gateway ファイル出力へ戻ることがあります。セッション全体でソースやカーソルが常に同一であると想定せず、復旧時に Gateway ファイルのカーソルが再実行されることで行が重複する場合も許容してください。

## 関連項目

- [ログの概要](/ja-JP/logging)
- [Gateway CLI](/ja-JP/cli/gateway)
- [CLI リファレンス](/ja-JP/cli)
- [Gateway のログ](/ja-JP/gateway/logging)
