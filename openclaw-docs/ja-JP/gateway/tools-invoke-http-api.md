---
read_when:
    - 完全なエージェントターンを実行せずにツールを呼び出す
    - ツールポリシーの適用が必要な自動化の構築
summary: Gateway HTTP エンドポイント経由で単一のツールを直接呼び出す
title: ツールが API を呼び出す
x-i18n:
    generated_at: "2026-07-26T10:16:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d07f765d63255e718d5e558b662589e77b2992538f43288cd83e6e3f2a06dda
    source_path: gateway/tools-invoke-http-api.md
    workflow: 16
---

OpenClaw の Gateway は、単一のツールを直接呼び出すための HTTP エンドポイントを公開します。このエンドポイントは常に有効で、Gateway 認証とツールポリシーを使用します。OpenAI 互換の `/v1/*` サーフェスと同様に、共有シークレットによる Bearer 認証は、Gateway 全体に対する信頼済みオペレーターアクセスとして扱われます。

- `POST /tools/invoke`
- Gateway と同じポート（WS + HTTP の多重化）: `http://<gateway-host>:<port>/tools/invoke`
- デフォルトの最大リクエスト本文サイズ: 2 MB

## 認証

Gateway の認証設定を使用します。

一般的な HTTP 認証パス:

- 共有シークレット認証（`gateway.auth.mode="token"` または `"password"`）: `Authorization: Bearer <token-or-password>`
- 信頼済みの ID 情報を伴う HTTP 認証（`gateway.auth.mode="trusted-proxy"`）: 設定済みの ID 対応プロキシ経由でルーティングし、必要な ID ヘッダーを挿入させます
- プライベートイングレスのオープン認証（`gateway.auth.mode="none"`）: 認証ヘッダーは不要です

注:

- `mode="token"` は `gateway.auth.token`（または `OPENCLAW_GATEWAY_TOKEN`）を使用します。
- `mode="password"` は `gateway.auth.password`（または `OPENCLAW_GATEWAY_PASSWORD`）を使用します。
- `mode="trusted-proxy"` では、HTTP リクエストが設定済みの信頼済みプロキシソースから送信される必要があります。同一ホストの loopback プロキシには、明示的な `gateway.auth.trustedProxy.allowLoopback = true` が必要です。
- プロキシを迂回する同一ホスト上の内部呼び出し元は、ローカルの直接フォールバックとして `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` を使用できます。ただし、`Forwarded`、`X-Forwarded-*`、または `X-Real-IP` ヘッダーの証拠がある場合、リクエストは引き続き信頼済みプロキシパスで処理されます。
- `gateway.auth.rateLimit` が設定されており、認証失敗が多すぎる場合、エンドポイントは `Retry-After` を設定した `429` を返します。

## セキュリティ境界（重要）

このエンドポイントは、Gateway インスタンスに対する**完全なオペレーターアクセス**を提供するサーフェスとして扱ってください。

- ここでの HTTP Bearer 認証は、ユーザーごとの限定的なスコープモデルではありません。
- このエンドポイントで有効な Gateway トークン／パスワードは、所有者／オペレーターの認証情報と同等に扱う必要があります。
- 共有シークレット認証モード（`token` および `password`）では、呼び出し元がより狭い `x-openclaw-scopes` ヘッダーを送信しても、エンドポイントは通常の完全なオペレーターデフォルトを復元します。
- 共有シークレット認証では、このエンドポイントでの直接的なツール呼び出しも、所有者が送信者であるターンとして扱われます。
- 信頼済みの ID 情報を伴う HTTP モード（信頼済みプロキシ認証、またはプライベートイングレス上の `gateway.auth.mode="none"`）では、`x-openclaw-scopes` が存在する場合はそれに従い、存在しない場合は通常のオペレーターデフォルトスコープセットにフォールバックします。
- このエンドポイントは loopback／tailnet／プライベートイングレス上にのみ配置し、公開インターネットへ直接公開しないでください。

認証マトリクス:

| 認証モード                                                                               | 動作                                                                                                                                                                                                                                                                                                               |
| --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `token` または `password` + `Authorization: Bearer ...`                                     | 共有 Gateway オペレーターシークレットを保持していることを証明します。より狭い `x-openclaw-scopes` は無視されます。完全なデフォルトのオペレータースコープセット（`operator.admin`、`operator.approvals`、`operator.pairing`、`operator.read`、`operator.talk.secrets`、`operator.write`）を復元します。直接的なツール呼び出しを、所有者が送信者であるターンとして扱います。 |
| 信頼済みの ID 情報を伴う HTTP（信頼済みプロキシ認証、またはプライベートイングレス上の `mode="none"`） | 外側の信頼済み ID またはデプロイ境界を認証します。`x-openclaw-scopes` が存在する場合はそれに従います。ヘッダーが存在しない場合は、通常のオペレーターデフォルトスコープセットにフォールバックします。呼び出し元がスコープを明示的に狭め、かつ `operator.admin` を省略した場合にのみ、所有者のセマンティクスが失われます。                               |

## リクエスト本文

```json
{
  "tool": "sessions_list",
  "action": "json",
  "args": {},
  "sessionKey": "main",
  "dryRun": false
}
```

フィールド:

- `tool` / `name`（文字列、必須）: 呼び出すツール名。両方が送信された場合は、`name` が優先されます。
- `action`（文字列、省略可能）: ツールスキーマが `action` プロパティをサポートし、`args` でまだ設定されていない場合、`args.action` にマージされます。
- `args`（オブジェクト、省略可能）: ツール固有の引数。
- `sessionKey`（文字列、省略可能）: 対象セッションキー。省略された場合、または `"main"` の場合、Gateway は設定済みのメインセッションキーを使用します（`session.mainKey` とデフォルトエージェント、またはグローバルセッションスコープの `global` に従います）。
- `agentId`（文字列、省略可能）: そのエージェントのセッションキーを解決します。明示的な `sessionKey` がすでに別のエージェントにマッピングされており、それと競合する場合は `400` エラーになります。
- `idempotencyKey`（文字列、省略可能）: 呼び出し用の安定したツール呼び出し ID を導出するために使用されます。
- `dryRun`（ブール値、省略可能）: 将来の使用のために予約されています。現在は無視されます。

## ポリシーとルーティングの動作

ツールの利用可否は、Gateway エージェントが使用するものと同じポリシーチェーンによってフィルタリングされます:

- `tools.profile` / `tools.byProvider.profile`
- `tools.allow` / `tools.byProvider.allow`
- `agents.<id>.tools.allow` / `agents.<id>.tools.byProvider.allow`
- グループポリシー（セッションキーがグループまたはチャンネルにマッピングされる場合）
- サブエージェントポリシー（サブエージェントのセッションキーで呼び出す場合）

ツールがポリシーで許可されていない場合、エンドポイントは **404** を返します。

重要な境界に関する注記:

- Exec の承認はオペレーター向けのガードレールであり、この HTTP エンドポイントに対する独立した認可境界ではありません。Gateway 認証とツールポリシーを介してツールがここから利用可能な場合、`/tools/invoke` によって呼び出しごとの追加承認プロンプトが表示されることはありません。
- `exec` がここから利用可能な場合、変更を伴うシェルサーフェスとして扱ってください。`write`、`edit`、`apply_patch`、または HTTP ファイルシステム書き込みツールを拒否しても、シェル実行が読み取り専用になるわけではありません。
- 信頼できない呼び出し元と Gateway の Bearer 認証情報を共有しないでください。信頼境界を分離する必要がある場合は、個別の Gateway を実行してください（可能であれば、異なる OS ユーザー／ホスト上で実行します）。

Gateway HTTP では、セッションポリシーでツールが許可されていても、デフォルトで強制拒否リストも適用されます:

| ツール             | 理由                                                    |
| ---------------- | --------------------------------------------------------- |
| `exec`           | コマンドの直接実行（RCE サーフェス）                    |
| `spawn`          | 任意の子プロセスの作成（RCE サーフェス）            |
| `shell`          | シェルコマンドの実行（RCE サーフェス）                     |
| `fs_write`       | ホスト上の任意のファイル変更                       |
| `fs_delete`      | ホスト上の任意のファイル削除                       |
| `fs_move`        | ホスト上の任意のファイル移動／名前変更                    |
| `apply_patch`    | パッチの適用によって任意のファイルを書き換え可能             |
| `sessions_spawn` | セッションのオーケストレーション。リモートでのエージェント生成は RCE に該当    |
| `sessions_send`  | セッション間のメッセージ挿入                           |
| `cron`           | 永続的な自動化コントロールプレーン                       |
| `gateway`        | Gateway コントロールプレーン。HTTP 経由での再設定を防止  |
| `nodes`          | Node コマンドリレーにより、ペアリングされたホスト上の `system.run` に到達可能 |

`cron`、`gateway`、および `nodes` も所有者専用です。このデフォルト拒否リストの対象外であっても、所有者ではない呼び出し元はこのサーフェスでこれらを呼び出せません。

一般的な拒否リストは `gateway.tools` でカスタマイズできます:

```json5
{
  gateway: {
    tools: {
      // HTTP /tools/invoke 経由でブロックする追加ツール
      deny: ["browser"],
      // 所有者／管理者の呼び出し元について、デフォルト拒否リストから除外するツール
      allow: ["gateway"],
    },
  },
}
```

`gateway.tools.allow` は公開範囲のオーバーライドであり、スコープの昇格ではありません。ID 情報を伴う HTTP モードでは、`gateway.tools.allow` に記載されていても、所有者／管理者の ID（`operator.admin`）を持たない呼び出し元は、`cron`、`gateway`、および `nodes` を引き続き利用できません。共有シークレットによる Bearer 認証には、前述の完全な信頼済みオペレーター規則が引き続き適用されます。

グループポリシーによるコンテキスト解決を支援するため、必要に応じて次を設定できます:

- `x-openclaw-message-channel: <channel>`（例: `slack`、`telegram`）
- `x-openclaw-account-id: <accountId>`（複数のアカウントが存在する場合）
- `x-openclaw-message-to: <target>`（メッセージツールポリシーの配信先）
- `x-openclaw-thread-id: <threadId>`（メッセージツールポリシーのスレッドコンテキスト）

## レスポンス

| ステータス | 意味                                                                                        |
| ------ | ---------------------------------------------------------------------------------------------- |
| `200`  | `{ ok: true, result }`                                                                         |
| `400`  | `{ ok: false, error: { type, message } }`（無効なリクエストまたはツール入力エラー）                |
| `401`  | 未認証                                                                                   |
| `403`  | `{ ok: false, error: { type, message, requiresApproval? } }`（ツール呼び出しがポリシーによってブロックされた）     |
| `404`  | ツールを利用できません（見つからないか、許可リストに含まれていません）                                              |
| `405`  | メソッドは許可されていません                                                                             |
| `408`  | リクエスト本文の読み取りがタイムアウトしました                                                                    |
| `413`  | リクエスト本文が最大ペイロードサイズを超えました                                                     |
| `429`  | 認証がレート制限されました（`Retry-After` が設定されます）                                                          |
| `500`  | `{ ok: false, error: { type, message } }`（予期しないツール実行エラー。メッセージはサニタイズ済み） |

## 例

```bash
curl -sS http://127.0.0.1:18789/tools/invoke \
  -H 'Authorization: Bearer secret' \
  -H 'Content-Type: application/json' \
  -d '{
    "tool": "sessions_list",
    "action": "json",
    "args": {}
  }'
```

## 関連項目

- [Gateway プロトコル](/ja-JP/gateway/protocol)
- [ツールと Plugin](/ja-JP/tools)
