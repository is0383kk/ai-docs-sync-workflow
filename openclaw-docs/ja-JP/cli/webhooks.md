---
read_when:
    - Gmail Pub/Sub イベントを OpenClaw に連携する場合
    - 完全なフラグ一覧とデフォルト値が必要です
summary: '`openclaw webhooks`（Gmail Pub/Sub のセットアップとランナー）の CLI リファレンス'
title: Webhook
x-i18n:
    generated_at: "2026-07-26T09:57:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83fff0ac2ce247402f45523eda0b5cdd551bd65212636118698e45cb8740236c
    source_path: cli/webhooks.md
    workflow: 16
---

# `openclaw webhooks`

Webhook のヘルパーと統合。現在、このサーフェスは、同梱の `gog` ウォッチャー上に構築された Gmail Pub/Sub フローを対象としています。

## サブコマンド

```bash
openclaw webhooks gmail setup --account <email> [...]
openclaw webhooks gmail run   [--account <email>] [...]
```

| サブコマンド    | 説明                                                                           |
| ------------- | ------------------------------------------------------------------------------------- |
| `gmail setup` | 1 回限りのウィザード：Gmail ウォッチ、Pub/Sub トピック／サブスクリプション、OpenClaw フック配信を設定します。 |
| `gmail run`   | `gog watch serve` とウォッチの自動更新ループをフォアグラウンドで実行します。               |

<Note>
`hooks.enabled=true` と `hooks.gmail.account` が設定されると（`gmail setup` によって設定）、Gateway は起動時に `gog gmail watch serve` も自動的に開始します。`gmail run` は同じロジックをフォアグラウンドで実行するもので、デバッグ時や Gateway ウォッチャーが無効な場合に役立ちます。自動起動の詳細と `OPENCLAW_SKIP_GMAIL_WATCHER` によるオプトアウトについては、[Gmail Pub/Sub 統合](/ja-JP/automation/cron-jobs#gmail-pubsub-integration)を参照してください。
</Note>

## `webhooks gmail setup`

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail setup --account you@example.com --project my-gcp-project --json
openclaw webhooks gmail setup --account you@example.com --hook-url https://gateway.example.com/hooks/gmail
```

`gcloud` と `gog` がない場合はインストールし、`gcloud` を認証して、Pub/Sub トピックとサブスクリプションを作成し、Gmail ウォッチを開始して、`hooks.enabled=true` を含む `hooks.gmail` 設定を書き込みます。`Next: openclaw webhooks gmail run` を出力します。

### 必須

| フラグ                | 説明             |
| ------------------- | ----------------------- |
| `--account <email>` | 監視する Gmail アカウント。 |

### Pub/Sub オプション

| フラグ                    | デフォルト                | 説明                                                                                                                             |
| ----------------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `--project <id>`        | （なし）                 | GCP プロジェクト ID（OAuth クライアントの所有者）。トピック自体のプロジェクト ID、次に `gog` の認証情報から解決されたプロジェクトへフォールバックします。 |
| `--topic <name>`        | `gog-gmail-watch`      | Pub/Sub トピック名。                                                                                                                     |
| `--subscription <name>` | `gog-gmail-watch-push` | Pub/Sub サブスクリプション名。                                                                                                              |
| `--label <label>`       | `INBOX`                | 監視する Gmail ラベル。                                                                                                                   |
| `--push-endpoint <url>` | （なし）                 | 明示的な Pub/Sub プッシュエンドポイント。Tailscale より優先されます。                                                                                    |

### OpenClaw 配信オプション

| フラグ                   | デフォルト                                      | 説明                                |
| ---------------------- | -------------------------------------------- | ------------------------------------------ |
| `--hook-url <url>`     | `hooks.path` と Gateway ポートから構築 | OpenClaw Webhook URL。                      |
| `--hook-token <token>` | `hooks.token`、または生成されたトークン          | OpenClaw Webhook トークン。                    |
| `--push-token <token>` | 生成されたトークン                              | `gog watch serve` に転送されるプッシュトークン。 |

### `gog watch serve` オプション

| フラグ                  | デフォルト         | 説明                                                                                                                                  |
| --------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `--bind <host>`       | `127.0.0.1`     | `gog watch serve` のバインドホスト。                                                                                                                 |
| `--port <port>`       | `8788`          | `gog watch serve` のポート。                                                                                                                      |
| `--path <path>`       | `/gmail-pubsub` | `gog watch serve` のパス。Tailscale はプロキシする前にパスを取り除くため、明示的なターゲットなしで Tailscale が有効な場合は `/` に強制されます。 |
| `--include-body`      | `true`          | メール本文の抜粋を含めます。これを無効にする CLI フラグはありません。代わりに設定で `hooks.gmail.includeBody: false` を指定してください。                  |
| `--max-bytes <n>`     | `20000`         | 本文の抜粋ごとの最大バイト数。                                                                                                                  |
| `--renew-minutes <n>` | `720` (12h)     | N 分ごとに Gmail ウォッチを更新します。                                                                                                           |

### Tailscale での公開

| フラグ                      | デフォルト  | 説明                                                      |
| ------------------------- | -------- | ---------------------------------------------------------------- |
| `--tailscale <mode>`      | `funnel` | tailscale 経由でプッシュエンドポイントを公開します：`funnel`、`serve`、または `off`。 |
| `--tailscale-path <path>` | （なし）   | tailscale serve/funnel のパス。                                 |
| `--tailscale-target <t>`  | （なし）   | Tailscale serve/funnel のターゲット（ポート、`host:port`、または URL）。       |

### 出力

| フラグ     | 説明                                       |
| -------- | ------------------------------------------------- |
| `--json` | テキストの代わりに機械可読な概要を出力します。 |

## `webhooks gmail run`

```bash
openclaw webhooks gmail run --account you@example.com
```

`gog watch serve` とウォッチの自動更新ループをフォアグラウンドで実行し、`gog watch serve` が予期せず終了した場合は 2s の遅延後に再起動します。

`run` は `setup` と同じ Pub/Sub、OpenClaw 配信、`gog watch serve`、Tailscale の各フラグを受け付けますが、以下の点が異なります。

- `--account` は `run` では**任意**です。指定しない場合は `hooks.gmail.account` にフォールバックします。
- `run` は `--project`、`--push-endpoint`、`--json` を受け付け**ません**。
- すべてのフラグは、対応する `hooks.gmail.*` 設定値（`setup` によって書き込まれる）、次に `setup` が使用するものと同じ組み込みデフォルトへフォールバックします。ただし 1 つ例外があり、フラグと `hooks.gmail.tailscale.mode` のどちらも設定されていない場合、`--tailscale` は `run` では `funnel` ではなく `off` がデフォルトになります。

| カテゴリ          | フラグ                                                                            |
| ----------------- | -------------------------------------------------------------------------------- |
| Pub/Sub           | `--account`、`--topic`、`--subscription`、`--label`                              |
| OpenClaw 配信 | `--hook-url`、`--hook-token`、`--push-token`                                     |
| `gog watch serve` | `--bind`、`--port`、`--path`、`--include-body`、`--max-bytes`、`--renew-minutes` |
| Tailscale         | `--tailscale`、`--tailscale-path`、`--tailscale-target`                          |

<Note>
`run` では、`--topic` の値は短いトピック名だけではなく、完全な Pub/Sub トピックパス（`projects/.../topics/...`）です。
</Note>

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [Webhook 自動化](/ja-JP/automation/cron-jobs)
- [Gmail Pub/Sub 統合](/ja-JP/automation/cron-jobs#gmail-pubsub-integration)
