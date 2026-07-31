---
read_when:
    - OpenAI Chat Completions を前提とするツールの統合
summary: Gateway から OpenAI 互換の /v1/chat/completions HTTP エンドポイントを公開する
title: OpenAI チャット補完
x-i18n:
    generated_at: "2026-07-26T10:01:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4cc5a1a56972bb9070da0f8f60d6efd673cc1d1d516b730c55bc9d171fc7a5b3
    source_path: gateway/openai-http-api.md
    workflow: 16
---

Gateway は、小規模な OpenAI 互換 Chat Completions サーフェスを提供できます。これは**デフォルトでは無効**です。

有効にすると、Gateway と同じポート（WS + HTTP 多重化）で以下すべてを提供します。

| メソッド | パス                   |
| ------ | ---------------------- |
| POST   | `/v1/chat/completions` |
| GET    | `/v1/models`           |
| GET    | `/v1/models/{id}`      |
| POST   | `/v1/embeddings`       |
| POST   | `/v1/responses`        |

リクエストは通常の Gateway エージェント実行として処理され（`openclaw agent` と同じコードパス）、ルーティング、権限、設定は使用中の Gateway と一致します。

## エンドポイントの有効化

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: true },
      },
    },
  },
}
```

無効にするには、`enabled: false` を設定します（または省略します）。

## セキュリティ境界（重要）

このエンドポイントは、Gateway インスタンスへの**完全なオペレーターアクセス**として扱ってください。

- このエンドポイントに対する有効な Gateway トークン／パスワードは、限定されたユーザー単位のスコープではなく、所有者／オペレーターの認証情報に相当します。
- リクエストは、信頼されたオペレーター操作と同じコントロールプレーンのエージェントパスを通じて実行されるため、対象エージェントのポリシーで機密性の高いツールが許可されている場合、このエンドポイントからそれらを使用できます。
- loopback／tailnet／プライベートイングレスのみに配置してください。公開インターネットに公開しないでください。

認証マトリクス：

| 認証パス                                                                                            | 動作                                                                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway.auth.mode="token"` または `"password"` + `Authorization: Bearer ...`                            | 共有 Gateway シークレットを保持していることを証明します。`x-openclaw-scopes` ヘッダーはすべて無視され、デフォルトの完全なオペレータースコープセット（`operator.admin`、`operator.approvals`、`operator.pairing`、`operator.read`、`operator.talk.secrets`、`operator.write`）が復元されます。チャットターンを所有者送信者のターンとして扱います。 |
| ID 情報を持つ信頼済み HTTP（trusted-proxy 認証、またはプライベートイングレス上の `gateway.auth.mode="none"`） | `x-openclaw-scopes` が存在する場合はそれに従い、存在しない場合はデフォルトのオペレータースコープセットにフォールバックします。呼び出し元が明示的にスコープを狭め、`operator.admin` を省略した場合にのみ、所有者のセマンティクスが失われます。`x-openclaw-model` などの所有者レベルの制御には `operator.admin` が必要です。                        |

[オペレータースコープ](/ja-JP/gateway/operator-scopes)、[セキュリティ](/ja-JP/gateway/security)、[リモートアクセス](/ja-JP/gateway/remote)を参照してください。

## 認証

Gateway の認証設定を使用します（このモードの詳細については、[信頼済みプロキシ認証](/ja-JP/gateway/trusted-proxy-auth)を参照してください）。

| モード                                | 認証方法                                                                                                                                                                     |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway.auth.mode="token"`         | `Authorization: Bearer <token>`。`gateway.auth.token` または `OPENCLAW_GATEWAY_TOKEN` で設定します。                                                                                              |
| `gateway.auth.mode="password"`      | `Authorization: Bearer <password>`。`gateway.auth.password` または `OPENCLAW_GATEWAY_PASSWORD` で設定します。                                                                                     |
| `gateway.auth.mode="trusted-proxy"` | 設定済みの ID 対応プロキシを経由してルーティングします。プロキシが必要な ID ヘッダーを挿入します。同一ホストの loopback プロキシでは、明示的な `gateway.auth.trustedProxy.allowLoopback = true` が必要です。 |
| `gateway.auth.mode="none"`          | 認証ヘッダーは不要です（プライベートイングレスのみ）。                                                                                                                                         |

注：

- `trusted-proxy` Gateway 上でプロキシを迂回する同一ホストの呼び出し元は、`gateway.auth.password`／`OPENCLAW_GATEWAY_PASSWORD` に直接フォールバックできます。`Forwarded`、`X-Forwarded-*`、または `X-Real-IP` のいずれかのヘッダー証拠がある場合、リクエストは代わりに trusted-proxy パスに維持されます。
- `gateway.auth.rateLimit` が設定されており、認証試行が何度も失敗した場合、エンドポイントは `Retry-After` ヘッダー付きで `429` を返します。

## このエンドポイントを使用する場合

- 統合が同じ Gateway に対する別のオペレーター／クライアントサーフェスにすぎない場合は、新しい組み込みチャネルを追加するよりも、これを優先してください。
- リモート Gateway に直接接続するネイティブモバイルクライアントでは、デバイスが共有 HTTP トークン／パスワードを必要としないように、ペアリング済みデバイスのブートストラップ／デバイストークンフローを備えた [WebChat](/ja-JP/web/webchat) または [Gateway プロトコル](/ja-JP/gateway/protocol)を使用することを推奨します。
- 独自のユーザー、ルーム、Webhook 配信、または送信トランスポートを持つ外部メッセージングネットワークと統合する場合は、代わりにチャネル Plugin を構築してください。[Plugin の構築](/ja-JP/plugins/building-plugins)を参照してください。

## エージェント優先のモデル契約

OpenClaw は、OpenAI の `model` フィールドを生のプロバイダーモデル ID ではなく、**エージェントの対象**として扱います。

| `model` の値                                | ルーティング先                                                                                                                |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `openclaw`                                   | 設定済みのデフォルトエージェント                                                                                                 |
| `openclaw/default`                           | 設定済みのデフォルトエージェント（安定したエイリアス。環境間で実際のデフォルトエージェント ID が変わっても安全にハードコード可能） |
| `openclaw/<agentId>` または `openclaw:<agentId>` | 特定のエージェント                                                                                                           |
| `agent:<agentId>`                            | 特定のエージェント（互換性エイリアス）                                                                                     |

任意のリクエストヘッダー：

| ヘッダー                                          | 効果                                                                                                                                                                                                                                                                      |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `x-openclaw-model: <provider/model-or-bare-id>` | 選択したエージェントのバックエンドモデルを上書きします。共有シークレットの Bearer 呼び出し元はこれを直接使用できます。ID 情報を持つ呼び出し元（trusted-proxy、または `x-openclaw-scopes` を使用する認証なしのプライベートイングレス）には `operator.admin` が必要であり、それ以外の場合は `403 missing scope: operator.admin` となります。 |
| `x-openclaw-agent-id: <agentId>`                | エージェント選択用の互換性オーバーライド。                                                                                                                                                                                                                                 |
| `x-openclaw-session-key: <sessionKey>`          | 明示的なセッションルーティング。予約済みの内部名前空間（`subagent:`、`cron:`、`acp:`）を使用している場合は、`400 invalid_request_error` で拒否されます。                                                                                                                                |
| `x-openclaw-message-channel: <channel>`         | チャネル対応のプロンプト／ポリシー用に、合成イングレスチャネルコンテキストを設定します。                                                                                                                                                                                              |

`/v1/models` は、バックエンドプロバイダーモデルやサブエージェントではなく、トップレベルのエージェント対象（`openclaw`、`openclaw/default`、`openclaw/<agentId>`）を一覧表示します。サブエージェントは内部の実行トポロジーに留まります。`x-openclaw-model` を省略した場合、選択されたエージェントは通常どおり設定済みのモデルで実行されます。

`/v1/embeddings` は、同じエージェント対象の `model` ID を使用します。特定の埋め込みモデルを選択するには、`x-openclaw-model` を送信します（共有シークレットの呼び出し元、または `operator.admin` を持つ ID 情報付きの呼び出し元から）。それ以外の場合、リクエストは選択されたエージェントの通常の埋め込み設定を使用します。

## セッションの動作

デフォルトでは、エンドポイントは**リクエストごとにステートレス**です（呼び出しごとに新しいセッションキーが生成されます）。

リクエストに OpenAI の `user` 文字列が含まれている場合、Gateway はそこから安定したセッションキーを導出するため、繰り返し呼び出すことでエージェントセッションを共有できます。カスタムアプリでは、会話スレッドごとに同じ `user` 値を再利用してください。複数の会話／デバイスで 1 つの OpenClaw セッションを共有したい場合を除き、アカウントレベルの識別子は避けてください。複数のクライアント／スレッド間で明示的なルーティング制御が必要な場合にのみ、上記の予約済み名前空間を回避するアプリケーション所有のキーとともに `x-openclaw-session-key` を使用してください。

## リクエスト制限

このエンドポイントには、リクエスト本文あたり 20 MB、最新のユーザーメッセージから 8 個の `image_url`
パート、デコード済み画像データの累計 20 MB という組み込み制限があります。
画像ソースポリシーは、
`gateway.http.endpoints.chatCompletions.images` で引き続き設定できます。

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: {
          enabled: true,
          images: {
            allowUrl: false,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "image/jpeg",
              "image/png",
              "image/gif",
              "image/webp",
              "image/heic",
              "image/heif",
            ],
            maxBytes: 10485760,
            maxRedirects: 3,
            timeoutMs: 10000,
          },
        },
      },
    },
  },
}
```

画像設定のデフォルトは次のとおりです。

| キー                   | デフォルト                                                             |
| --------------------- | ------------------------------------------------------------------- |
| `images.allowUrl`     | `false`（URL をソースとする `image_url` パートは、有効にしない限り拒否されます） |
| `images.maxBytes`     | 画像あたり 10MB                                                      |
| `images.maxRedirects` | 3                                                                   |
| `images.timeoutMs`    | 10s                                                                 |

HEIC／HEIF の `image_url` ソースは受け入れられ、共有 OpenClaw 画像プロセッサ（Rastermill）を通じてプロバイダーに配信される前に JPEG に正規化されます。このプロセッサは、外部コーデックのサポートが必要な形式について、システムコンバーター（`sips`、ImageMagick、GraphicsMagick、または ffmpeg）にフォールバックします。

セキュリティ上の注意：ホスト名を許可リストに登録しても、プライベート／内部 IP のブロックは回避されません。インターネットに公開される Gateway では、アプリレベルのガードに加えて、ネットワークの外向き通信制御を適用してください。[セキュリティ](/ja-JP/gateway/security)を参照してください。

## チャットツール契約

`/v1/chat/completions` は、一般的な OpenAI Chat クライアントと互換性のある関数ツールのサブセットをサポートします。

### サポートされるリクエストフィールド

| フィールド                      | 注記                                                                                                                                         |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `tools`                    | `{ "type": "function", "function": { ... } }` の配列                                                                                        |
| `tool_choice`              | `"auto"`、`"none"`、`"required"`、または `{ "type": "function", "function": { "name": "..." } }`                                                  |
| `messages[*].role: "tool"` | 後続ターン                                                                                                                               |
| `messages[*].tool_call_id` | ツール結果を以前のツール呼び出しに関連付けます                                                                                                 |
| `max_completion_tokens`    | 数値。呼び出しごとの完了トークン総数の上限（推論トークンを含む）。現在のフィールド名。これと `max_tokens` の両方が送信された場合に使用されます。 |
| `max_tokens`               | 数値。レガシーエイリアス。`max_completion_tokens` も存在する場合は無視されます。                                                                   |
| `temperature`              | 0～2 の数値。ベストエフォートで上流プロバイダーに転送されます。範囲外の場合は `400 invalid_request_error`。                                     |
| `top_p`                    | 0～1 の数値。ベストエフォート。範囲外の場合は `400 invalid_request_error`。                                                                         |
| `frequency_penalty`        | -2.0～2.0 の数値。ベストエフォート。範囲外の場合は `400 invalid_request_error`。                                                                 |
| `presence_penalty`         | -2.0～2.0 の数値。ベストエフォート。範囲外の場合は `400 invalid_request_error`。                                                                 |
| `seed`                     | 整数。ベストエフォート。整数でない値の場合は `400 invalid_request_error`。                                                                     |
| `stop`                     | 文字列、または最大 4 個の文字列の配列。ベストエフォート。シーケンスが 4 個を超える場合、または文字列でない項目や空の項目がある場合は `400 invalid_request_error`。           |

すべてのサンプリングフィールドとトークン上限フィールドは同じエージェントストリームパラメーターチャネルを通り、ベストエフォートで転送されます。

- トークン上限: ワイヤー上のフィールド名はプロバイダーのトランスポートによって選択されます。OpenAI 系エンドポイントでは `max_completion_tokens`、レガシー名のみを受け付けるプロバイダー（Mistral、Chutes）では `max_tokens` です。
- `stop` はトランスポートの停止フィールドにマッピングされます。Chat Completions バックエンドでは `stop`、Anthropic では `stop_sequences` です。OpenAI Responses API には停止パラメーターがないため、Responses ベースのモデルには `stop` が適用されません。
- ChatGPT ベースの Codex Responses バックエンドは、固定されたサーバー側サンプリングを使用し、リクエストがそのバックエンドに到達する前に `temperature`/`top_p`（および `max_output_tokens`、`metadata`、`prompt_cache_retention`、`service_tier`）を除去します。

### サポートされていないバリアント

次の場合は `400 invalid_request_error` を返します。

- 配列でない `tools`、関数でないツール項目、または `tool.function.name` の欠落
- `allowed_tools` や `custom` などの `tool_choice` バリアント
- 指定されたツールと一致しない `tool_choice.function.name` 値

`tool_choice: "required"` および関数が固定された `tool_choice` の場合、エンドポイントは公開されるクライアント関数ツールのセットを絞り込み、応答前にクライアントツールを呼び出すようランタイムに指示し、エージェント応答に一致する構造化クライアントツール呼び出しがない場合はエラーにします。これは呼び出し元が指定した HTTP `tools` リストに適用され、OpenClaw エージェントのすべての内部ツールに適用されるわけではありません。

### 非ストリーミングのツール応答形式

エージェントがツールを呼び出す場合、応答には以下が使用されます。

- `choices[0].finish_reason = "tool_calls"`
- `id`、`type: "function"`、`function.name`、`function.arguments`（JSON 文字列）を含む `choices[0].message.tool_calls[]` 項目
- ツール呼び出し前のアシスタントの解説。`choices[0].message.content` に格納されます（空の場合があります）

### ストリーミングのツール応答形式

`stream: true` の場合、ツール呼び出しは増分 SSE チャンクとして到着します。最初にアシスタントロールの差分、任意のアシスタント解説の差分、次にツールの識別情報と引数の断片を伝える 1 個以上の `delta.tool_calls` チャンク、最後に `finish_reason: "tool_calls"` と `data: [DONE]` を含むチャンクが続きます。

`stream_options.include_usage=true` の場合、`[DONE]` の前に末尾の使用量チャンクが送出されます。

### ツールの後続ループ

`tool_calls` を受信したら、要求された関数を実行し、以前のアシスタントのツール呼び出しメッセージと、一致する `tool_call_id` を持つ 1 個以上の `role: "tool"` メッセージを含めた後続リクエストを送信します。これにより、同じエージェント推論ループが継続され、最終回答が生成されます。

## ストリーミング（SSE）

Server-Sent Events を受信するには `stream: true` を設定します。

- `Content-Type: text/event-stream`
- 各イベント行は `data: <json>` です
- ストリームは `data: [DONE]` で終了します

## Open WebUI のクイックセットアップ

- ベース URL: `http://127.0.0.1:18789/v1`
- macOS 上の Docker のベース URL: `http://host.docker.internal:18789/v1`
- API キー: Gateway のベアラートークン
- モデル: `openclaw/default`

期待される動作: `GET /v1/models` は `openclaw/default` を一覧表示し、Open WebUI はこれをチャットモデル ID として使用します。特定のバックエンドプロバイダーやモデルを使用するには、エージェントの通常のデフォルトモデルを設定するか、`x-openclaw-model` を送信します（共有シークレットを使用する呼び出し元、または `operator.admin` を持つアイデンティティ付きの呼び出し元）。

簡単なスモークテスト:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

これが `openclaw/default` を返す場合、ほとんどの Open WebUI セットアップでは同じベース URL とトークンで接続できます。

## 例

1 つのアプリ会話で安定したセッションを使用する場合:

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "user": "conv:YOUR_CONVERSATION_ID",
    "messages": [{"role":"user","content":"今日のタスクを要約してください"}]
  }'
```

その会話の後続の呼び出しで同じ `user` 値を再利用すると、同じエージェントセッションを継続できます。

非ストリーミング:

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "messages": [{"role":"user","content":"こんにちは"}]
  }'
```

ストリーミング:

```bash
curl -N http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/gpt-5.4' \
  -d '{
    "model": "openclaw/research",
    "stream": true,
    "messages": [{"role":"user","content":"こんにちは"}]
  }'
```

モデル一覧:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

1 つのモデルを取得:

```bash
curl -sS http://127.0.0.1:18789/v1/models/openclaw%2Fdefault \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

埋め込みを作成:

```bash
curl -sS http://127.0.0.1:18789/v1/embeddings \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/text-embedding-3-small' \
  -d '{
    "model": "openclaw/default",
    "input": ["alpha", "beta"]
  }'
```

`/v1/embeddings` は、文字列または文字列の配列として `input` をサポートします。

## 関連項目

- [設定リファレンス](/ja-JP/gateway/configuration-reference)
- [オペレータースコープ](/ja-JP/gateway/operator-scopes)
- [OpenAI](/ja-JP/providers/openai)
