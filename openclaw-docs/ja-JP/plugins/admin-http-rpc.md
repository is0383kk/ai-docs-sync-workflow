---
read_when:
    - Gateway WebSocket RPC クライアントを使用できないホストツールの構築
    - プライベートで信頼されたイングレス経由で Gateway 管理自動化を公開する
    - Gateway メソッドへの HTTP アクセスに関するセキュリティモデルの監査
summary: バンドル済みのオプトイン式 admin-http-rpc Plugin を通じて、選択した Gateway コントロールプレーンメソッドを公開する
title: 管理用 HTTP RPC Plugin
x-i18n:
    generated_at: "2026-07-26T09:50:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0709081efd0ce65cef7edac54df9a71978cbad17e2b25df83ac9075de938376c
    source_path: plugins/admin-http-rpc.md
    workflow: 16
---

バンドルされている `admin-http-rpc` Plugin は、Gateway WebSocket 接続を開いたままにできない信頼されたホスト自動化向けに、許可リストに登録された Gateway コントロールプレーンメソッドを HTTP 経由で公開します。

これは OpenClaw に同梱されていますが、デフォルトでは無効です。無効な場合、ルートは登録されません。有効にすると、Gateway と同じリスナー（`http://<gateway-host>:<port>/api/v1/admin/rpc`）に `POST /api/v1/admin/rpc` が追加されます。

プライベートなホストツール、tailnet 自動化、または信頼された内部 ingress にのみ有効化してください。このルートを公開インターネットに直接公開しないでください。

## 有効化する前に

管理 HTTP RPC は、完全なオペレーターコントロールプレーンのサーフェスです。Gateway HTTP 認証を通過した呼び出し元は、以下の許可リストに登録されたメソッドを呼び出せます。次のすべての条件を満たす場合にのみ有効化してください。

- 呼び出し元が Gateway の操作を任せられる信頼された主体である。
- 呼び出し元が WebSocket RPC クライアントを使用できない。
- ルートには、loopback、tailnet、または認証済みのプライベート ingress からのみ到達できる。
- 許可されたメソッドを確認済みであり、実行予定の自動化に適合している。

Gateway WebSocket 接続を開いたままにできる OpenClaw クライアントや対話型ツールでは、代わりに WebSocket RPC を使用してください。

## 有効化

バンドルされた Plugin を有効化します。

<Tabs>
  <Tab title="CLI">
    ```bash
    openclaw plugins enable admin-http-rpc
    openclaw gateway restart
    ```
  </Tab>
  <Tab title="設定">
    ```json5
    {
      plugins: {
        entries: {
          "admin-http-rpc": { enabled: true },
        },
      },
    }
    ```
  </Tab>
</Tabs>

ルートは Plugin の起動時に登録されるため、Plugin 設定を変更した後は Gateway を再起動してください。

HTTP サーフェスが不要になったら無効化します。

```bash
openclaw plugins disable admin-http-rpc
openclaw gateway restart
```

## ルートの確認

最小限で安全なリクエストとして `health` を使用します。

```bash
curl -sS http://<gateway-host>:<port>/api/v1/admin/rpc \
  -H 'Authorization: Bearer <gateway-token>' \
  -H 'Content-Type: application/json' \
  -d '{"method":"health","params":{}}'
```

成功したレスポンスには `ok: true` が含まれます。

```json
{
  "id": "generated-request-id",
  "ok": true,
  "payload": {
    "status": "ok"
  }
}
```

Plugin が無効な場合、ルートが登録されていないため `404` が返されます。

## 認証

Plugin のルートは Gateway HTTP 認証を使用します。

一般的な認証経路：

- 共有シークレット認証（`gateway.auth.mode="token"` または `"password"`）：`Authorization: Bearer <token-or-password>`
- 信頼された ID 付き HTTP 認証（`gateway.auth.mode="trusted-proxy"`）：設定済みの ID 対応プロキシを経由させ、必要な ID ヘッダーを挿入させます
- プライベート ingress のオープン認証（`gateway.auth.mode="none"`）：認証ヘッダーは不要です

## セキュリティモデル

この Plugin は、完全な Gateway オペレーターサーフェスとして扱ってください。

- Plugin を有効化すると、`/api/v1/admin/rpc` で許可リストに登録された管理 RPC メソッドへのアクセスが意図的に提供されます。
- Plugin は予約済みの `contracts.gatewayMethodDispatch: ["authenticated-request"]` マニフェスト契約を宣言します。これにより、Gateway 認証済みの HTTP ルートからコントロールプレーンメソッドをプロセス内でディスパッチできます。これはサンドボックスではありません。この契約は予約済み SDK ヘルパーの誤使用を防ぎますが、信頼された Plugin は引き続き Gateway プロセス内で実行されます。
- 共有シークレットの Bearer 認証（`token`/`password` モード）は、Gateway オペレーターシークレットを保持していることを証明します。この経路では、より限定的な `x-openclaw-scopes` ヘッダーは無視され、通常の完全なオペレーターのデフォルト設定が復元されます。
- 信頼された ID 付き HTTP 認証（`trusted-proxy` モード）では、`x-openclaw-scopes` が存在する場合、それが適用されます。
- `gateway.auth.mode="none"` は、Plugin が有効な場合にこのルートが未認証になることを意味します。完全に信頼できるプライベート ingress の背後でのみ使用してください。
- リクエストは、Plugin ルートの認証を通過した後、WebSocket RPC と同じ Gateway メソッドハンドラーおよびスコープチェックを通じてディスパッチされます。
- このルートは、準備済みの一時停止リース中も到達可能な状態を維持します。制限付きのリクエスト検証とローカルの `commands.list` 検出レスポンスは引き続き利用できます。Gateway にディスパッチされるメソッドのうち、受け入れが閉じられている間に実行できるのは `gateway.suspend.prepare`、`gateway.suspend.status`、`gateway.suspend.resume` のみです。その他の許可リスト登録済みメソッドは、通常の再試行可能な Gateway `UNAVAILABLE` レスポンスを返します。
- このルートは loopback、tailnet、または信頼されたプライベート ingress 上に置いてください。公開インターネットに直接公開しないでください。呼び出し元が信頼境界をまたぐ場合は、別々の Gateway を使用してください。

## リクエスト

```http
POST /api/v1/admin/rpc
Authorization: Bearer <gateway-token>
Content-Type: application/json
```

```json
{
  "id": "optional-request-id",
  "method": "health",
  "params": {}
}
```

フィールド：

- `id`（文字列、省略可能）：レスポンスにコピーされます。省略した場合は UUID が生成されます。
- `method`（文字列、必須）：許可された Gateway メソッド名。
- `params`（任意、省略可能）：メソッド固有のパラメーター。

デフォルトの最大リクエストボディサイズは 1 MB です。

## レスポンス

成功レスポンスは Gateway RPC 形式を使用します。

```json
{
  "id": "optional-request-id",
  "ok": true,
  "payload": {}
}
```

Gateway メソッドエラーは次の形式を使用します。

```json
{
  "id": "optional-request-id",
  "ok": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "bad params"
  }
}
```

HTTP ステータスはエラーコードに従います。

| エラーコード                 | HTTP ステータス |
| -------------------------- | ----------- |
| `INVALID_REQUEST`          | 400         |
| `APPROVAL_NOT_FOUND`       | 404         |
| `NOT_LINKED`, `NOT_PAIRED` | 409         |
| `UNAVAILABLE`              | 503         |
| `AGENT_TIMEOUT`            | 504         |
| その他のコード             | 500         |

## 許可されたメソッド

- 検出：`commands.list`
  この Plugin で許可されている HTTP RPC メソッド名を返します。
- Gateway：`health`、`status`、`logs.tail`、`usage.status`、`usage.cost`、`gateway.restart.request`、`gateway.suspend.prepare`、`gateway.suspend.status`、`gateway.suspend.resume`
- 設定：`config.get`、`config.schema`、`config.schema.lookup`、`config.set`、`config.patch`、`config.apply`
- チャンネル：`channels.status`、`channels.start`、`channels.stop`、`channels.logout`
- Web：`web.login.start`、`web.login.wait`
- モデル：`models.list`、`models.authStatus`
- エージェント：`agents.list`、`agents.create`、`agents.update`、`agents.delete`
- 承認：`exec.approvals.get`、`exec.approvals.set`、`exec.approvals.node.get`、`exec.approvals.node.set`
- Cron：`cron.status`、`cron.list`、`cron.get`、`cron.runs`、`cron.add`、`cron.update`、`cron.remove`、`cron.run`
- デバイス：`device.pair.list`、`device.pair.approve`、`device.pair.reject`、`device.pair.remove`
- Node：`node.list`、`node.describe`、`node.pair.list`、`node.pair.approve`、`node.pair.reject`、`node.pair.remove`、`node.rename`
- タスク：`tasks.list`、`tasks.get`、`tasks.cancel`
- 診断：`doctor.memory.status`、`update.status`

その他の Gateway メソッドは、意図的に追加されるまでブロックされます。

## WebSocket との比較

通常の Gateway WebSocket RPC 経路は、OpenClaw クライアント向けの推奨コントロールプレーン API です。管理 HTTP RPC は、リクエスト／レスポンス型の HTTP サーフェスを必要とするホストツールにのみ使用してください。

信頼されたデバイス ID を持たない共有トークンの WebSocket クライアントは、接続時に管理スコープを自己申告できません。管理 HTTP RPC は、既存の信頼された HTTP オペレーターモデルに意図的に従います。Plugin が有効な場合、共有シークレットの Bearer 認証は、この管理サーフェスに対する完全なオペレーターアクセスとして扱われます。

## トラブルシューティング

`404 Not Found`

: Plugin が無効になっている、Plugin を有効化してから Gateway が再起動されていない、またはリクエストが別の Gateway プロセスに送信されています。

`401 Unauthorized`

: リクエストが Gateway HTTP 認証を満たしていません。Bearer トークンまたは信頼されたプロキシの ID ヘッダーを確認してください。

`405 Method Not Allowed`

: リクエストで `POST` 以外が使用されました。

`413 Payload Too Large`

: リクエストボディが 1 MB の制限を超えました。

`400 INVALID_REQUEST`

: リクエストボディが有効な JSON ではない、`method` フィールドがない、メソッドが Plugin の許可リストに含まれていない、または一時停止の再開 ID がアクティブなリースと一致していません。

`503 UNAVAILABLE`

: Gateway メソッドが起動中、レート制限中、一時停止中、または競合する一時停止／再開操作を待機中です。`error.details` が存在する場合は確認し、再試行する前に `error.retryAfterMs` に従ってください。

## 関連項目

- [オペレータースコープ](/ja-JP/gateway/operator-scopes)
- [Gateway のセキュリティ](/ja-JP/gateway/security)
- [リモートアクセス](/ja-JP/gateway/remote)
- [Plugin マニフェスト](/ja-JP/plugins/manifest#contracts-reference)
- [SDK サブパス](/ja-JP/plugins/sdk-subpaths)
