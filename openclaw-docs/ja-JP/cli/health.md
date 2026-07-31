---
read_when:
    - 実行中の Gateway の正常性をすばやく確認したい場合
summary: '`openclaw health` の CLI リファレンス（RPC 経由の Gateway ヘルススナップショット）'
title: ヘルスチェック
x-i18n:
    generated_at: "2026-07-26T09:35:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 51cc0e3dd61af3e6fa460dd646bfa1c3e5bd1a52da860eac26c12101151d081d
    source_path: cli/health.md
    workflow: 16
---

# `openclaw health`

実行中の Gateway から WebSocket RPC 経由でヘルススナップショットを取得します（CLI からチャネルソケットへ直接接続することはありません）。

## オプション

| フラグ             | デフォルト | 説明                                                                       |
| ---------------- | ------- | --------------------------------------------------------------------------------- |
| `--json`         | `false` | テキストの代わりに機械可読な JSON を出力します。                                      |
| `--timeout <ms>` | `10000` | 接続タイムアウト（ミリ秒）。                                               |
| `--verbose`      | `false` | ライブプローブを強制し、設定済みのすべてのアカウントとエージェントにわたって出力を展開します。 |
| `--debug`        | `false` | `--verbose` のエイリアス。                                                            |

例：

```bash
openclaw health
openclaw health --json
openclaw health --timeout 2500
openclaw health --verbose
openclaw health --debug
```

## 動作

- `--verbose` を指定しない場合、Gateway はキャッシュされたスナップショット（最大 60 秒間有効で、ライブチャネルのランタイム状態と同一）を返し、次の呼び出し元のためにバックグラウンドで更新できます。
- `--verbose` はライブプローブ（チャネルごとのアカウントプローブ）を強制し、Gateway の接続詳細を出力します。また、デフォルトのエージェントだけでなく、設定済みのすべてのアカウントとエージェントにわたって人間が読みやすい出力を展開します。
- `--json` は常に完全なスナップショットを返します。これには、チャネル、アカウントごとのプローブ、Plugin の読み込み状態、コンテキストエンジンの隔離状態、モデル料金キャッシュの状態、イベントループの健全性、配信キューのデッドレター、エージェントごとのセッションストアが含まれます。
- 送信配信または受信チャネルイベントがデッドレター化されると、テキスト出力にはその件数と最も古い失敗からの経過時間が表示されます。受信件数はチャネルアカウント別にグループ化されます。[`openclaw channels dead-letters`](/ja-JP/cli/channels#inbound-dead-letters)を使用して、個々のイベントを確認または復旧できます。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [`openclaw status`](/ja-JP/cli/status) — 完全なヘルススナップショットを取得せずに行うローカル診断とチャネルプローブ
- [Gateway のヘルス](/ja-JP/gateway/health)
