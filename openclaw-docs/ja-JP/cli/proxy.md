---
read_when:
    - デプロイ前に、オペレーターが管理するプロキシルーティングを検証する必要があります
    - デバッグのために OpenClaw のトランスポートトラフィックをローカルでキャプチャする必要があります
    - デバッグプロキシのセッション、blob、または組み込みのクエリプリセットを確認したい場合
summary: '`openclaw proxy` の CLI リファレンス（オペレーター管理のプロキシ検証とローカルデバッグ用プロキシキャプチャインスペクターを含む）'
title: プロキシ
x-i18n:
    generated_at: "2026-07-26T09:16:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91583f785032bfffe455a1963804108550f6fbb735ac4de1dd91d0ca5ae0df35
    source_path: cli/proxy.md
    workflow: 16
---

# `openclaw proxy`

オペレーター管理のプロキシルーティングを検証するか、ローカルの明示的なデバッグプロキシを実行して、キャプチャされたトラフィックを調査します。

```bash
openclaw proxy validate [--json] [--proxy-url <url>] [--proxy-ca-file <path>] [--allowed-url <url>] [--denied-url <url>] [--apns-reachable] [--apns-authority <url>] [--timeout-ms <ms>]
openclaw proxy start [--host <host>] [--port <port>]
openclaw proxy run [--host <host>] [--port <port>] -- <cmd...>
openclaw proxy coverage
openclaw proxy sessions [--limit <count>]
openclaw proxy query --preset <name> [--session <id>]
openclaw proxy blob --id <blobId>
openclaw proxy purge
```

`validate` は、オペレーター管理のフォワードプロキシを事前検証します。残りはトランスポートレベルの調査用デバッグツールです。ローカルのキャプチャプロキシの起動、それを経由した子コマンドの実行、キャプチャセッションの一覧表示、トラフィックパターンの照会、キャプチャされた BLOB の読み取り、ローカルキャプチャデータの消去を行えます。

## 検証

`--proxy-url`、設定（`proxy.proxyUrl`）、または `OPENCLAW_PROXY_URL` から、この優先順位で有効なオペレーター管理プロキシ URL を確認します。プロキシが有効化および設定されていない場合は、設定上の問題を報告します。設定を変更せずに一度限りの事前検証を行うには、`--proxy-url` を渡します。

管理対象プロキシ URL には、通常のフォワードプロキシリスナーには `http://` を使用し、OpenClaw がプロキシリクエストを送信する前にプロキシエンドポイント自体への TLS 接続を開く必要がある場合は `https://` を使用します。その TLS 接続でプライベート CA を信頼するには、`--proxy-ca-file` を使用します。

デフォルトでは、次を実行します。

- `https://example.com/` に対する **許可される** チェック 1 回（`--allowed-url` で上書きまたは追加可能、繰り返し指定可能）
- 一時的なループバックカナリアに対する **拒否される** チェック 1 回（`--denied-url` で上書き可能、繰り返し指定可能）

カスタム `--denied-url` ターゲットはフェイルクローズです。デプロイ固有の拒否シグナルを独立して検証できない限り、HTTP レスポンスと曖昧なトランスポート障害はいずれも失敗として扱われます。トランスポートエラーがブロックの証明として扱われるのは、組み込みのループバックカナリアだけです。

プロキシ経由で APNs HTTP/2 CONNECT トンネルも開き、サンドボックス APNs が応答することを確認するには、`--apns-reachable` を追加します。このプローブは意図的に無効なプロバイダートークンを送信するため、APNs の `403 InvalidProviderToken` レスポンスは、失敗ではなく到達可能性を示す正常なシグナルとして扱われます。

### オプション

| フラグ                     | 効果                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `--json`                 | 機械可読な JSON を出力する                                                                                        |
| `--proxy-url <url>`      | 設定や環境変数の代わりに、この `http://`/`https://` プロキシ URL を検証する                                              |
| `--proxy-ca-file <path>` | HTTPS プロキシエンドポイントの TLS 検証で、この PEM CA ファイルを信頼する                                             |
| `--allowed-url <url>`    | プロキシ経由で成功することが期待される宛先（繰り返し指定可能）                                                     |
| `--denied-url <url>`     | プロキシによってブロックされることが期待される宛先（繰り返し指定可能）                                                       |
| `--apns-reachable`       | プロキシ経由でサンドボックス APNs HTTP/2 に到達可能であることも検証する                                                     |
| `--apns-authority <url>` | プローブする APNs オーソリティ（デフォルトは `https://api.sandbox.push.apple.com`、本番環境は `https://api.push.apple.com`） |
| `--timeout-ms <ms>`      | リクエストごとのタイムアウト                                                                                                |

プロキシ設定または宛先チェックが失敗した場合、終了コード 1 で終了します。

デプロイのガイダンスと拒否セマンティクスについては、[ネットワークプロキシ](/ja-JP/security/network-proxy)を参照してください。

## デバッグプロキシ

`start` はローカルのキャプチャプロキシを起動し、その URL、CA 証明書のパス、キャプチャ DB のパスを出力します。停止するには Ctrl+C を押します。`--host` が設定されていない限り、デフォルトでは `127.0.0.1` にバインドします。

`run` はローカルのデバッグプロキシを起動し、プロキシ環境変数を適用して、独自のキャプチャセッション内で `<cmd...>`（`--` の後）を実行します。

デバッグプロキシの直接アップストリーム転送は、診断用にアップストリームソケットを開きます。OpenClaw の管理対象プロキシモードが有効な場合、プロキシリクエストと CONNECT トンネルの直接転送はデフォルトで無効になります。承認済みのローカル診断に限り、`OPENCLAW_DEBUG_PROXY_ALLOW_DIRECT_CONNECT_WITH_MANAGED_PROXY=1` を設定してください。

`coverage` は、どのトランスポートがキャプチャ対象、プロキシのみ、または未対応であるかを示す JSON レポート（`summary` とトランスポートごとの `entries`）を出力します。

`sessions` は、最近のキャプチャセッションを一覧表示します（`--limit`、デフォルトは 20）。

`query --preset <name>` は、必要に応じて `--session <id>` に対象を限定し、キャプチャされたトラフィックに対して組み込みクエリを実行します。プリセットは次のとおりです。

- `double-sends`
- `retry-storms`
- `cache-busting`
- `ws-duplicate-frames`
- `missing-ack`
- `error-bursts`

`blob --id <blobId>` は、キャプチャされたペイロード BLOB の未加工の内容を出力します。

`purge` は、キャプチャされたすべてのトラフィックメタデータと BLOB を削除します。キャプチャはローカルのデバッグデータです。完了したら消去してください。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [ネットワークプロキシ](/ja-JP/security/network-proxy)
- [信頼済みプロキシ認証](/ja-JP/gateway/trusted-proxy-auth)
