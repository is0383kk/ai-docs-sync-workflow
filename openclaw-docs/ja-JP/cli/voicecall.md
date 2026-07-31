---
read_when:
    - 音声通話 Plugin を使用し、すべての CLI エントリポイントを利用する場合
    - setup、smoke、call、continue、speak、dtmf、end、status、tail、latency、expose、start のフラグ表とデフォルト値が必要です
summary: '`openclaw voicecall` の CLI リファレンス（voice-call Plugin のコマンドサーフェス）'
title: 音声通話
x-i18n:
    generated_at: "2026-07-26T09:37:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aec445886cccb79c9212dd9f1f448ff9634274deb380632be786478c9bb29670
    source_path: cli/voicecall.md
    workflow: 16
---

# `openclaw voicecall`

`voicecall` は Plugin が提供するコマンドです。voice-call
Plugin がインストールされ、有効になっている場合にのみ表示されます。

Gateway が実行中の場合、操作コマンド（`call`、`start`、
`continue`、`speak`、`dtmf`、`end`、`status`）は、その Gateway の
voice-call ランタイムにルーティングされます。到達可能な Gateway がない場合は、スタンドアロンの
CLI ランタイムにフォールバックします。

## サブコマンド

```bash
openclaw voicecall setup    [--json]
openclaw voicecall smoke    [-t <phone>] [--message <text>] [--mode <m>] [--yes] [--json]
openclaw voicecall call     -m <text> [-t <phone>] [--mode <m>]
openclaw voicecall start    --to <phone> [--message <text>] [--mode <m>]
openclaw voicecall continue --call-id <id> --message <text>
openclaw voicecall speak    --call-id <id> --message <text>
openclaw voicecall dtmf     --call-id <id> --digits <digits>
openclaw voicecall end      --call-id <id>
openclaw voicecall status   [--call-id <id>] [--json]
openclaw voicecall tail     [--file <path>] [--since <n>] [--poll <ms>]
openclaw voicecall latency  [--file <path>] [--last <n>]
openclaw voicecall expose   [--mode <m>] [--path <p>] [--port <port>] [--serve-path <p>]
```

| サブコマンド | 説明                                                     |
| ---------- | --------------------------------------------------------------- |
| `setup`    | プロバイダーと Webhook の準備状況チェックを表示します。                     |
| `smoke`    | 準備状況チェックを実行します。`--yes` を指定した場合にのみ、実際のテスト通話を発信します。 |
| `call`     | 音声通話を発信します。                                |
| `start`    | `call` のエイリアスです。`--to` は必須で、`--message` は任意です。 |
| `continue` | メッセージを読み上げ、次の応答を待ちます。                 |
| `speak`    | 応答を待たずにメッセージを読み上げます。                 |
| `dtmf`     | アクティブな通話に DTMF 数字を送信します。                             |
| `end`      | アクティブな通話を切断します。                                         |
| `status`   | アクティブな通話を確認します（または `--call-id` で 1 件を指定します）。                   |
| `tail`     | `calls.jsonl` を追跡表示します（プロバイダーのテスト時に便利です）。              |
| `latency`  | `calls.jsonl` からターン遅延のメトリクスを要約します。              |
| `expose`   | Webhook エンドポイントに対する Tailscale serve/funnel を切り替えます。         |

## セットアップとスモークテスト

### `setup`

デフォルトでは、人が読みやすい形式で準備状況チェックを出力します。スクリプトでは `--json` を指定します。

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

### `smoke`

同じ準備状況チェックを実行します。`--to` と
`--yes` の両方が指定されている場合にのみ、実際の電話を発信します。

| フラグ               | デフォルト                           | 説明                             |
| ------------------ | --------------------------------- | --------------------------------------- |
| `-t, --to <phone>` | （なし）                            | ライブスモークテストで発信する電話番号。  |
| `--message <text>` | `OpenClaw voice call smoke test.` | スモークテスト通話中に読み上げるメッセージ。 |
| `--mode <mode>`    | `notify`                          | 通話モード：`notify` または `conversation`。  |
| `--yes`            | `false`                           | 実際にライブ発信を行います。  |
| `--json`           | `false`                           | 機械可読な JSON を出力します。            |

```bash
openclaw voicecall smoke
openclaw voicecall smoke --to "+15555550123"        # ドライラン
openclaw voicecall smoke --to "+15555550123" --yes  # ライブ通知通話
```

<Note>
外部プロバイダー（`plivo`、`telnyx`、`twilio`）では、`setup` と `smoke` に、`publicUrl`、トンネル、または Tailscale 公開によるパブリック Webhook URL が必要です。通信事業者から到達できないため、loopback またはプライベート serve へのフォールバックは拒否されます。
</Note>

## 通話のライフサイクル

### `call`

音声通話を発信します。

| フラグ                   | 必須 | デフォルト           | 説明                                                                |
| ---------------------- | -------- | ----------------- | -------------------------------------------------------------------------- |
| `-m, --message <text>` | はい      | （なし）            | 通話が接続されたときに読み上げるメッセージ。                                   |
| `-t, --to <phone>`     | いいえ       | config `toNumber` | 発信先の E.164 電話番号。                                                |
| `--mode <mode>`        | いいえ       | `conversation`    | 通話モード：`notify`（メッセージ後に切断）または `conversation`（接続を維持）。 |

```bash
openclaw voicecall call --to "+15555550123" --message "Hello"
openclaw voicecall call -m "Heads up" --mode notify
```

### `start`

デフォルトのフラグ構成が異なる `call` のエイリアスです。

| フラグ               | 必須 | デフォルト        | 説明                              |
| ------------------ | -------- | -------------- | ---------------------------------------- |
| `--to <phone>`     | はい      | （なし）         | 発信先の電話番号。                    |
| `--message <text>` | いいえ       | （なし）         | 通話が接続されたときに読み上げるメッセージ。 |
| `--mode <mode>`    | いいえ       | `conversation` | 通話モード：`notify` または `conversation`。   |

### `continue`

メッセージを読み上げ、応答を待ちます。

| フラグ               | 必須 | 説明       |
| ------------------ | -------- | ----------------- |
| `--call-id <id>`   | はい      | 通話 ID。          |
| `--message <text>` | はい      | 読み上げるメッセージ。 |

### `speak`

応答を待たずにメッセージを読み上げます。

| フラグ               | 必須 | 説明       |
| ------------------ | -------- | ----------------- |
| `--call-id <id>`   | はい      | 通話 ID。          |
| `--message <text>` | はい      | 読み上げるメッセージ。 |

### `dtmf`

アクティブな通話に DTMF 数字を送信します。

| フラグ                | 必須 | 説明                                      |
| ------------------- | -------- | ------------------------------------------------ |
| `--call-id <id>`    | はい      | 通話 ID。                                         |
| `--digits <digits>` | はい      | DTMF 数字（待機には、たとえば `ww123456#` を使用）。 |

### `end`

アクティブな通話を切断します。

| フラグ             | 必須 | 説明 |
| ---------------- | -------- | ----------- |
| `--call-id <id>` | はい      | 通話 ID。    |

### `status`

アクティブな通話を確認します。

| フラグ             | デフォルト | 説明                  |
| ---------------- | ------- | ---------------------------- |
| `--call-id <id>` | （なし）  | 出力を 1 件の通話に限定します。 |
| `--json`         | `false` | 機械可読な JSON を出力します。 |

```bash
openclaw voicecall status
openclaw voicecall status --json
openclaw voicecall status --call-id <id>
```

## ログとメトリクス

### `tail`

voice-call の JSONL ログを追跡表示します。開始時に最後の `--since` 行を出力し、その後、
新しい行が書き込まれるたびにストリーミングします。

| フラグ            | デフォルト                    | 説明                    |
| --------------- | -------------------------- | ------------------------------ |
| `--file <path>` | Plugin ストアから解決 | `calls.jsonl` へのパス。         |
| `--since <n>`   | `25`                       | 追跡開始前に出力する行数。 |
| `--poll <ms>`   | `250`（最小 50）         | ミリ秒単位のポーリング間隔。 |

### `latency`

`calls.jsonl` からターン遅延とリッスン待機のメトリクスを要約します。出力は
`recordsScanned`、`turnLatency`、`listenWait` の要約を含む JSON です。

| フラグ            | デフォルト                    | 説明                          |
| --------------- | -------------------------- | ------------------------------------ |
| `--file <path>` | Plugin ストアから解決 | `calls.jsonl` へのパス。               |
| `--last <n>`    | `200`（最小 1）          | 分析する最近のレコード数。 |

## Webhook の公開

### `expose`

音声 Webhook の Tailscale serve/funnel 設定を有効化、無効化、または変更します。

| フラグ                  | デフォルト                                   | 説明                                     |
| --------------------- | ----------------------------------------- | ----------------------------------------------- |
| `--mode <mode>`       | `funnel`                                  | `off`、`serve`（tailnet）、または `funnel`（パブリック）。 |
| `--path <path>`       | config `tailscale.path` または `--serve-path` | 公開する Tailscale パス。                       |
| `--port <port>`       | config `serve.port` または `3334`             | ローカル Webhook ポート。                             |
| `--serve-path <path>` | config `serve.path` または `/voice/webhook`   | ローカル Webhook パス。                             |

```bash
openclaw voicecall expose --mode serve
openclaw voicecall expose --mode funnel
openclaw voicecall expose --mode off
```

<Warning>
信頼できるネットワークにのみ Webhook エンドポイントを公開してください。可能な場合は Funnel より Tailscale Serve を優先してください。
</Warning>

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [音声通話 Plugin](/ja-JP/plugins/voice-call)
