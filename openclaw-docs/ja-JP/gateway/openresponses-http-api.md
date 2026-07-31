---
read_when:
    - OpenResponses API を使用するクライアントの統合
    - 項目ベースの入力、クライアントツール呼び出し、または SSE イベントが必要な場合
summary: Gateway から OpenResponses 互換の /v1/responses HTTP エンドポイントを公開する
title: OpenResponses API
x-i18n:
    generated_at: "2026-07-26T10:14:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bfd6ca3bf0cecd761fde865b41a95cff3fc5681f74f31b3adae5cd2e0b0be95
    source_path: gateway/openresponses-http-api.md
    workflow: 16
---

Gateway は OpenResponses 互換の `POST /v1/responses` エンドポイントを提供できます。このエンドポイントは**デフォルトでは無効**で、Gateway とポートを共有します（WS + HTTP の多重化）：`http://<gateway-host>:<port>/v1/responses`。

リクエストは通常の Gateway エージェント実行として処理されるため（`openclaw agent` と同じコードパス）、ルーティング、権限、設定は使用中の Gateway と一致します。

`gateway.http.endpoints.responses.enabled` で有効または無効にします。有効にすると、同じ互換サーフェスで `GET /v1/models`、`GET /v1/models/{id}`、`POST /v1/embeddings`、および `POST /v1/chat/completions` も提供されます。

## 認証、セキュリティ、ルーティング

運用上の動作は [OpenAI Chat Completions](/ja-JP/gateway/openai-http-api) と同じです。

- 認証パスは `gateway.auth.mode` と同じです。共有シークレット（`token`/`password`）では `Authorization: Bearer <token-or-password>` を使用します。trusted-proxy では ID 対応プロキシヘッダーを使用します（同一ホストのループバックプロキシには `gateway.auth.trustedProxy.allowLoopback = true` が必要です。`Forwarded`/`X-Forwarded-*`/`X-Real-IP` ヘッダーが存在しない場合は、`gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` による同一ホストの直接フォールバックを使用します）。プライベートイングレスの `none` には認証ヘッダーは不要です。[信頼済みプロキシ認証](/ja-JP/gateway/trusted-proxy-auth)を参照してください。
- このエンドポイントは、Gateway インスタンスへの完全なオペレーターアクセスとして扱ってください。
- 共有シークレット認証モードでは、Bearer で宣言されたより狭い `x-openclaw-scopes` は無視され、完全なデフォルトのオペレータースコープセット（`operator.admin`、`operator.approvals`、`operator.pairing`、`operator.read`、`operator.talk.secrets`、`operator.write`）が復元されます。このエンドポイントでのチャットターンは、所有者が送信したターンとして扱われます。
- 信頼済みの ID 保持 HTTP モード（trusted-proxy または `gateway.auth.mode="none"`）では、`x-openclaw-scopes` が存在する場合はそれを尊重し、存在しない場合はデフォルトのオペレータースコープセットにフォールバックします。所有者のセマンティクスが失われるのは、呼び出し元が明示的にスコープを狭め、かつ `operator.admin` を省略した場合のみです。
- `model: "openclaw"`、`"openclaw/default"`、`"openclaw/<agentId>"`、または `x-openclaw-agent-id` ヘッダーを使用してエージェントを選択します。
- 選択したエージェントのバックエンドモデルを上書きするには `x-openclaw-model` を使用します（ID 保持認証パスでは `operator.admin` が必要です）。
- 明示的なセッションルーティングには `x-openclaw-session-key` を使用します（予約済み名前空間 `subagent:`、`cron:`、`acp:` を使用すると、`400 invalid_request_error` で拒否されます）。
- デフォルト以外の合成イングレスチャネルコンテキストには `x-openclaw-message-channel` を使用します。

エージェント対象モデル、`openclaw/default`、埋め込みのパススルー、およびバックエンドモデルの上書きに関する正式な説明については、[OpenAI Chat Completions](/ja-JP/gateway/openai-http-api#agent-first-model-contract) を参照してください。

[オペレータースコープ](/ja-JP/gateway/operator-scopes)と[セキュリティ](/ja-JP/gateway/security)を参照してください。

## セッションの動作

デフォルトでは、このエンドポイントは**リクエストごとにステートレス**です（呼び出しのたびに新しいセッションキーが生成されます）。

リクエストに OpenResponses の `user` 文字列が含まれている場合、Gateway はそこから安定したセッションキーを導出するため、繰り返しの呼び出しでエージェントセッションを共有できます。

`previous_response_id` は、リクエストが同じエージェント／ユーザー／要求セッションのスコープ内に留まる場合（認証サブジェクト、エージェント ID、および `x-openclaw-session-key` で照合）、以前のレスポンスのセッションを再利用します。

## リクエスト形式

| フィールド                                                            | サポート                                                                                                                        |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `input`                                                          | 文字列またはアイテムオブジェクトの配列。                                                                                               |
| `instructions`                                                   | システムプロンプトにマージされます。                                                                                                 |
| `tools`                                                          | クライアントツール定義（関数ツール）。                                                                                      |
| `tool_choice`                                                    | クライアントツールを絞り込む、または必須にするための `"auto"`、`"none"`、`"required"`、または `{ "type": "function", "name": "..." }`。                |
| `stream`                                                         | SSE ストリーミングを有効にします。                                                                                                         |
| `max_output_tokens`                                              | ベストエフォートの出力上限（プロバイダー依存）。                                                                                 |
| `temperature`                                                    | ベストエフォートのサンプリング温度。サーバー側で固定サンプリングを使用する ChatGPT ベースの Codex Responses バックエンドでは無視されます。 |
| `top_p`                                                          | ベストエフォートの nucleus sampling。`temperature` と同じ Codex Responses に関する注意事項が適用されます。                                                    |
| `user`                                                           | 安定したセッションルーティング。                                                                                                        |
| `previous_response_id`                                           | セッションの継続性（前述を参照）。                                                                                                |
| `max_tool_calls`、`reasoning`、`metadata`、`store`、`truncation` | 受け付けますが、現在は無視されます。                                                                                                |

## アイテム（入力）

### `message`

ロール：`system`、`developer`、`user`、`assistant`。

- `system` と `developer` はシステムプロンプトに追加されます。
- 最新の `user` または `function_call_output` アイテムが「現在のメッセージ」になります。
- それ以前のユーザー／アシスタントメッセージは、コンテキスト用の履歴として含まれます。

### `function_call_output`（ターンベースのツール）

ツールの結果をモデルに返送します。

```json
{
  "type": "function_call_output",
  "call_id": "call_123",
  "output": "{\"temperature\": \"72F\"}"
}
```

### `reasoning` と `item_reference`

スキーマ互換性のため受け付けますが、プロンプトの構築時には無視されます。

## ツール（クライアント側の関数ツール）

`tools: [{ type: "function", name, description?, parameters? }]` でツールを指定します。

エージェントがツールを呼び出すと、レスポンスは `function_call` 出力アイテムを返します。ターンを続行するには、`function_call_output` を含む後続リクエストを送信します。

`tool_choice: "required"` および関数に固定された `tool_choice` の場合、エンドポイントは公開するクライアント関数ツールのセットを絞り込み、応答前にクライアントツールを呼び出すようランタイムに指示し、`/v1/chat/completions` 契約に従って、一致する構造化クライアントツール呼び出しが含まれていなければターンを拒否します。非ストリーミングリクエストは `api_error` を伴う `502` を返し、ストリーミングリクエストは `response.failed` イベントを送出します。

## 画像（`input_image`）

base64 または URL ソースをサポートします。

```json
{
  "type": "input_image",
  "source": { "type": "url", "url": "https://example.com/image.png" }
}
```

許可される MIME タイプ（デフォルト）：`image/jpeg`、`image/png`、`image/gif`、`image/webp`、`image/heic`、`image/heif`。最大サイズ（デフォルト）：10MB。

## ファイル（`input_file`）

base64 または URL ソースをサポートします。

```json
{
  "type": "input_file",
  "source": {
    "type": "base64",
    "media_type": "text/plain",
    "data": "SGVsbG8gV29ybGQh",
    "filename": "hello.txt"
  }
}
```

許可される MIME タイプ（デフォルト）：`text/plain`、`text/markdown`、`text/html`、`text/csv`、`application/json`、`application/pdf`。最大サイズ（デフォルト）：5MB。

現在の動作：

- ファイルの内容はデコードされ、ユーザーメッセージではなく**システムプロンプト**に追加されるため、一時的なままです（セッション履歴には永続化されません）。
- デコードされたファイルテキストは、追加される前に**信頼されていない外部コンテンツ**としてラップされるため、ファイルのバイト列は信頼済みの指示ではなくデータとして扱われます。挿入されるブロックでは、明示的な境界マーカー（`<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>`）と `Source: External` メタデータ行が使用されます。プロンプトの容量を確保するため、長い `SECURITY NOTICE:` バナーは意図的に省略されますが、境界マーカーとメタデータは引き続き適用されます。
- PDF は最初にテキスト抽出のために解析されます。テキストがほとんど見つからない場合、最初の数ページが画像にラスタライズされてモデルに渡され、挿入されるファイルブロックではプレースホルダー `[PDF content rendered to images]` が使用されます。

PDF 解析は、バンドルされている `document-extract` Plugin によって提供されます。この Plugin は、テキスト抽出とページレンダリングに `clawpdf` および同梱の PDFium WebAssembly ランタイムを使用します。

URL 取得のデフォルト：

- `files.allowUrl`：`true`
- `images.allowUrl`：`true`
- `maxUrlParts`：`8`（リクエストごとの URL ベースの `input_file` + `input_image` パートの合計）
- リクエストには保護措置が適用されます（DNS 解決、プライベート IP のブロック、リダイレクト上限、タイムアウト）。
- 入力タイプごとに、オプションのホスト名許可リスト（`files.urlAllowlist`、`images.urlAllowlist`）がサポートされます。完全一致ホスト（`"cdn.example.com"`）またはワイルドカードサブドメイン（`"*.assets.example.com"`、apex には一致しません）を指定できます。許可リストが空または省略されている場合、ホスト名許可リストによる制限はありません。
- URL ベースの取得を完全に無効にするには、`files.allowUrl: false` および／または `images.allowUrl: false` を設定します。

## ファイルと画像の制限

このエンドポイントは、組み込みの 20 MB リクエストボディ上限を使用します。ファイルおよび画像ソースの
ポリシーは、引き続き `gateway.http.endpoints.responses` で設定できます。

```json5
{
  gateway: {
    http: {
      endpoints: {
        responses: {
          enabled: true,
          maxUrlParts: 8,
          files: {
            allowUrl: true,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "text/plain",
              "text/markdown",
              "text/html",
              "text/csv",
              "application/json",
              "application/pdf",
            ],
            maxBytes: 5242880,
            maxChars: 60000,
            maxRedirects: 3,
            timeoutMs: 10000,
            pdf: {
              maxPages: 4,
              maxPixels: 4000000,
              minTextChars: 200,
            },
          },
          images: {
            allowUrl: true,
            urlAllowlist: ["images.example.com"],
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

省略時のデフォルト：

| キー                      | デフォルト   |
| ------------------------ | --------- |
| `maxUrlParts`            | 8         |
| `files.maxBytes`         | 5MB       |
| `files.maxChars`         | 60k       |
| `files.maxRedirects`     | 3         |
| `files.timeoutMs`        | 10s       |
| `files.pdf.maxPages`     | 4         |
| `files.pdf.maxPixels`    | 4,000,000 |
| `files.pdf.minTextChars` | 200       |
| `images.maxBytes`        | 10MB      |
| `images.maxRedirects`    | 3         |
| `images.timeoutMs`       | 10s       |

HEIC/HEIF `input_image` ソースは、共有 OpenClaw 画像プロセッサ（Rastermill）を介してプロバイダーに配信される前に JPEG に正規化されます。外部コーデックのサポートが必要な形式では、システムコンバーター（`sips`、ImageMagick、GraphicsMagick、または ffmpeg）にフォールバックします。

セキュリティ上の注意: URL 許可リストは、取得前およびリダイレクトの各ホップで適用されます。ホスト名を許可リストに追加しても、プライベート IP または内部 IP のブロックは回避されません。インターネットに公開される Gateway では、アプリレベルの保護に加えて、ネットワークの外向き通信制御を適用してください。[セキュリティ](/ja-JP/gateway/security)を参照してください。

## ストリーミング（SSE）

Server-Sent Events を受信するには、`stream: true` を設定します。

- `Content-Type: text/event-stream`
- 各イベント行は `event: <type>` および `data: <json>` です
- ストリームは `data: [DONE]` で終了します

現在送出されるイベントタイプ: `response.created`、`response.in_progress`、`response.output_item.added`、`response.content_part.added`、`response.output_text.delta`、`response.output_text.done`、`response.content_part.done`、`response.output_item.done`、`response.completed`、`response.failed`（エラー時）。

## 使用量

基盤となるプロバイダーがトークン数を報告する場合、`usage` に値が設定されます。OpenClaw は、これらのカウンターが後続のステータス／セッション画面に到達する前に、`input_tokens` / `output_tokens` や `prompt_tokens` / `completion_tokens` など、一般的な OpenAI 形式のエイリアスを正規化します。

## エラー

エラーには、次のような JSON オブジェクトを使用します。

```json
{ "error": { "message": "...", "type": "invalid_request_error" } }
```

一般的なケース: `400` 無効なリクエスト本文、`401` 認証がないか無効、`403` オペレータースコープがない、`405` メソッドが誤っている、`429` 認証の失敗回数が多すぎる（`Retry-After` を伴う）。

## 例

非ストリーミング:

```bash
curl -sS http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "input": "hi"
  }'
```

ストリーミング:

```bash
curl -N http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "stream": true,
    "input": "hi"
  }'
```

## 関連項目

- [OpenAI チャット補完](/ja-JP/gateway/openai-http-api)
- [オペレータースコープ](/ja-JP/gateway/operator-scopes)
- [OpenAI](/ja-JP/providers/openai)
