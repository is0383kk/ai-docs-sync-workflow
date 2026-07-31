---
read_when:
    - モバイル Node アプリを Gateway とすばやくペアリングしたい場合
    - リモートまたは手動で共有するためのセットアップコードの出力が必要です
summary: '`openclaw qr` の CLI リファレンス（モバイルペアリング用 QR コードとセットアップコードを生成）'
title: QR
x-i18n:
    generated_at: "2026-07-26T09:31:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d60a58126eae7eec5979f28bb511a09fa52b68cdd73727fca0b2de74efa84a
    source_path: cli/qr.md
    workflow: 16
---

# `openclaw qr`

現在の Gateway 設定から、モバイルペアリング用の QR とセットアップコードを生成します。

```bash
openclaw qr
openclaw qr --setup-code-only
openclaw qr --json
openclaw qr --remote
openclaw qr --limited
openclaw qr --url wss://gateway.example/ws
```

公式の OpenClaw iOS および Android アプリは、セットアップコードのメタデータが一致すると自動的に接続します。リクエストが保留中のままの場合（たとえば、非公式クライアントやメタデータの不一致の場合）は、確認して承認します。

```bash
openclaw devices list
openclaw devices approve <requestId>
```

## オプション

- `--remote`: `gateway.remote.url` を優先します。その URL が未設定の場合は `gateway.tailscale.mode=serve|funnel` にフォールバックします。`device-pair` Plugin の `publicUrl` は無視します。
- `--url <url>`: ペイロードで使用する Gateway URL を上書きします
- `--public-url <url>`: ペイロードで使用する公開 URL を上書きします
- `--token <token>`: ブートストラップフローの認証先となる Gateway トークンを上書きします
- `--password <password>`: ブートストラップフローの認証先となる Gateway パスワードを上書きします
- `--limited`: 引き渡すオペレータートークンから Gateway の管理アクセスを除外します
- `--setup-code-only`: セットアップコードのみを出力します
- `--no-ascii`: ASCII QR の描画をスキップします
- `--json`: JSON を出力します（`setupCode`、`gatewayUrl`、任意の `gatewayUrls`、`auth`、`access`、任意の `accessDowngraded`、`urlSource`）

`--token` と `--password` は相互に排他的です。

## セットアップコードの内容

セットアップコードには、共有 Gateway トークン／パスワードではなく、不透明で有効期間の短い `bootstrapToken` が含まれます。`wss://` エンドポイント（または同一ホストのループバック）の場合、デフォルトのブートストラップフローは次を発行します。

- `scopes: []` を持つプライマリ `node` トークン
- `operator.admin`、`operator.approvals`、`operator.read`、`operator.talk.secrets`、`operator.write` を持つ、完全なネイティブモバイル用 `operator` 引き渡しトークン

オペレーターへの引き渡しから `operator.admin` を除外しながら同じ Node トークンを維持するには、`--limited` を使用します。ペアリング変更スコープがセットアップコードによって引き渡されることはありません。

平文 LAN `ws://` セットアップも引き続き利用できますが、ネットワーク監視者がベアラーブートストラップトークンを取得して先に使用する可能性があるため、OpenClaw は制限付きプロファイルを自動的に使用します。完全なアクセスを得るには、`wss://` または Tailscale Serve を設定してから、新しいコードを生成してください。

## Gateway URL の解決

Tailscale／公開 `ws://` Gateway URL では、モバイルペアリングはフェイルクローズします。この場合は Tailscale Serve／Funnel または `wss://` Gateway URL を使用してください。プライベート LAN アドレスと `.local` Bonjour ホストは、平文 `ws://` 経由でも引き続きサポートされますが、前述のとおりオペレーターアクセスは制限されます。

選択した Gateway URL が `gateway.bind=lan` から取得された場合、OpenClaw は永続的な `tailscale serve status --json` ルートも確認します。アクティブな Gateway のループバックポートをプロキシする HTTPS Serve ルートは、すべてフォールバックとして含まれます。QR コマンドがこのフォールバックを追加するのは `lan` のみです。`custom` と `tailnet` は、明示的に公開されたルートを維持します。現在の iOS クライアントは、公開されたルートを順番にプローブして最初に到達可能なものを保存します。古いクライアント向けの従来の `url` フィールドは変更されません。

`--remote` を使用する場合、`gateway.remote.url` または `gateway.tailscale.mode=serve|funnel` のいずれかが必要です。

## 認証の解決（`--remote` なし）

CLI の認証上書きが渡されていない場合、ローカル Gateway 認証の SecretRefs は次のように解決されます。

| 条件                                                                                                                    | 解決先                                  |
| ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `gateway.auth.mode="token"`、または優先されるパスワードソースがない推論モード                                                | `gateway.auth.token`                      |
| `gateway.auth.mode="password"`、または認証／環境から優先されるトークンがない推論モード                                         | `gateway.auth.password`                   |
| `gateway.auth.token` と `gateway.auth.password` の両方が設定され（SecretRefs を含む）、`gateway.auth.mode` が未設定 | 失敗します。`gateway.auth.mode` を明示的に設定してください |

## 認証の解決（`--remote`）

実質的に有効なリモート認証情報が SecretRefs として設定されており、`--token` と `--password` のどちらも渡されていない場合、このコマンドはアクティブな Gateway スナップショットからそれらを解決します。Gateway が利用できない場合、コマンドは即座に失敗します。

<Note>
このコマンドパスには、`secrets.resolve` RPC メソッドをサポートする Gateway が必要です。古い Gateway は不明なメソッドのエラーを返します。
</Note>

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [デバイス](/ja-JP/cli/devices)
- [ペアリング](/ja-JP/cli/pairing)
