---
read_when:
    - API クライアントの構築
    - エンドポイントまたはスキーマの追加
summary: 公開 REST API（v1）の概要と規約。
x-i18n:
    generated_at: "2026-07-26T09:14:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 31b0051506912d2aa0d724ed7b6542e09ef16dc92998ddbdd3e379f783954436
    source_path: clawhub/api.md
    workflow: 16
---

# API v1

ベース: `https://clawhub.ai`

OpenAPI: `/api/v1/openapi.json`

## 公開カタログの再利用

ClawHub の公開読み取り API を基盤として、サードパーティのカタログ、ディレクトリ、または検索インターフェースを構築できます。公開されている Skills のメタデータとファイルは ClawHub の Skills ライセンス規則に従って公開されています。一方、API 自体にはレート制限があるため、責任を持って使用してください。

ガイドライン:

- カタログ一覧には、`GET /api/v1/skills`、`GET /api/v1/search`、`GET /api/v1/skills/{slug}` などの公開読み取りエンドポイントを使用します。
- 頻繁なポーリングは避け、レスポンスをキャッシュし、`429`、`Retry-After`、およびレート制限ヘッダーに従います。
- 一覧を表示する際は、ユーザーがソースレジストリのレコードを確認できるように、正規の ClawHub Skills URL へのリンクを設定します。
- ページの正規 URL には `https://clawhub.ai/<owner>/skills/<slug>` の形式を使用します。
- ClawHub がサードパーティサイトを推奨、検証、または運営しているかのような表現は使用しないでください。
- 公開 API のフィルターや認証境界を回避して、非表示、非公開、またはモデレーションによってブロックされたコンテンツをミラーリングしないでください。

## 認証

- 公開読み取り: トークンは不要です。
- 書き込み + アカウント: `Authorization: Bearer clh_...`。

## レート制限

認証状態を考慮した適用:

- 匿名リクエスト: IP ごと。
- 認証済みリクエスト（有効な Bearer トークン）: ユーザーバケットごと。
- トークンがない、または無効な場合は、IP ごとの適用にフォールバックします。

- 読み取り: IP ごとに 3000/分、キーごとに 12000/分
- 書き込み: IP ごとに 300/分、キーごとに 3000/分
- ダウンロード: IP ごとに 1200/分、キーごとに 6000/分

ヘッダー: `X-RateLimit-Limit`、`X-RateLimit-Reset`、`RateLimit-Limit`、`RateLimit-Reset`;
`X-RateLimit-Remaining`、`RateLimit-Remaining`、`Retry-After` は `429` に含まれます。

セマンティクス:

- `X-RateLimit-Reset`: Unix エポック秒（絶対リセット時刻）
- `RateLimit-Reset`: リセットまでの遅延秒数
- `X-RateLimit-Remaining` / `RateLimit-Remaining`: 存在する場合の正確な残り枠。シャーディングされた正常なリクエストでは、近似的なグローバル値を返さず、
  この値を省略します
- `Retry-After`: `429` の際に待機する遅延秒数

`429` の例:

```http
HTTP/2 429
x-ratelimit-limit: 20
x-ratelimit-remaining: 0
x-ratelimit-reset: 1771404540
ratelimit-limit: 20
ratelimit-remaining: 0
ratelimit-reset: 34
retry-after: 34
```

クライアント側の処理:

- `Retry-After` が存在する場合は、それを優先します。
- それ以外の場合は、`RateLimit-Reset` を使用するか、`X-RateLimit-Reset` から遅延時間を算出します。
- 再試行にジッターを加えます。

## エラー

- v1 のエラーはプレーンテキスト（`text/plain; charset=utf-8`）です。これには `400`、
  `401`、`403`、`404`、`429`、およびブロックされたダウンロードのレスポンスが含まれます。
- 互換性のため、不明なクエリパラメーターは無視されます。
- 既知のクエリパラメーターに無効な値を指定すると、`400` が返されます。

## エンドポイント

公開読み取り:

- `GET /api/v1/search?q=...`
  - 省略可能なフィルター: `highlightedOnly=true`、`nonSuspiciousOnly=true`
  - レガシーエイリアス: `nonSuspicious=true`
- `GET /api/v1/skills?limit=&cursor=&sort=`
  - `sort`: `updated`（デフォルト）、`recommended`（`default`）、`createdAt`（`newest`）、`downloads`、`stars`（`rating`）、レガシーインストールエイリアス `installsCurrent`/`installs`/`installsAllTime` は `downloads`、`trending` にマッピングされます
  - 無効な `sort` 値を指定すると、`400` が返されます
  - `cursor` は、`trending` 以外の並べ替えに適用されます
  - 省略可能なフィルター: `nonSuspiciousOnly=true`
  - レガシーエイリアス: `nonSuspicious=true`
  - `nonSuspiciousOnly=true` を使用する場合、カーソルベースのページに含まれる項目数が `limit` 未満になることがあります。続行するには `nextCursor` を使用します。
  - `recommended` は、エンゲージメントと新しさのシグナルを使用します。
- `GET /api/v1/skills/{slug}`
- `GET /api/v1/skills/{slug}/moderation`
- `GET /api/v1/skills/{slug}/versions?limit=&cursor=`
- `GET /api/v1/skills/{slug}/versions/{version}`
- `GET /api/v1/skills/{slug}/scan?version=&tag=`
- `GET /api/v1/skills/{slug}/file?path=&version=&tag=`
- `GET /api/v1/resolve?slug=&hash=`
- `GET /api/v1/download?slug=&version=&tag=`
  - ホストされている Skills は、決定論的な ZIP バイト列を返します。
  - `clean` または `suspicious` のスキャン結果を持つ現在の GitHub ベースの Skills は、ClawHub のバイト列ではなく、
    JSON の `public-github` 引き継ぎ記述子を返します。
- `GET /api/v1/skills/export?startDate=&endDate=&limit=&cursor=`
  - ホストされている Skills は、保存されているファイルのままエクスポートされます。
  - `clean` または `suspicious` のスキャン結果を持つ現在の GitHub ベースの Skills は、
    `public-github` 引き継ぎ記述子としてエクスポートされます。
- `GET /api/v1/packages?limit=&cursor=&sort=`
  - `sort`: `updated`（デフォルト）、`recommended`、`downloads`、レガシーエイリアス `installs`
  - 無効な `sort` 値を指定すると、`400` が返されます
- `GET /api/v1/plugins?limit=&cursor=&sort=`
  - `sort`: `recommended`（デフォルト）、`downloads`、`updated`、レガシーエイリアス `installs`
- `GET /api/v1/plugins/search?q=...`
- `GET /api/v1/packages/{name}/versions/{version}/artifact`
- `GET /api/v1/packages/{name}/versions/{version}/security`
- `GET /api/v1/packages/{name}/versions/{version}/artifact/download`
- `GET /api/npm/{package}`
- `GET /api/npm/{package}/-/{tarball}.tgz`

認証必須:

- `POST /api/v1/skills`（公開、multipart を推奨）
- `DELETE /api/v1/skills/{slug}`
- `DELETE /api/v1/packages/{name}`
- `POST /api/v1/skills/{slug}/undelete`
- `POST /api/v1/packages/{name}/undelete`
- `POST /api/v1/skills/{slug}/rename`
- `POST /api/v1/skills/{slug}/merge`
- `POST /api/v1/skills/{slug}/transfer`
- `POST /api/v1/packages/{name}/transfer`
- `POST /api/v1/skills/{slug}/transfer/accept`
- `POST /api/v1/skills/{slug}/transfer/reject`
- `POST /api/v1/skills/{slug}/transfer/cancel`
- `GET /api/v1/skills/export?startDate=&endDate=&limit=&cursor=`
- `GET /api/v1/plugins/export?startDate=&endDate=&limit=&cursor=&family=`
- `GET /api/v1/transfers/incoming`
- `GET /api/v1/transfers/outgoing`
- `GET /api/v1/whoami`

管理者のみ:

- `POST /api/v1/users/reserve` は、所有者のハンドル用にルートスラッグと非公開のリリースなしパッケージプレースホルダーを予約します。

## レガシー

レガシーの `/api/*` と `/api/cli/*` は引き続き利用できます。`DEPRECATIONS.md` を参照してください。
