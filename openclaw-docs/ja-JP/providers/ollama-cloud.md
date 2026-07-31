---
read_when:
    - ローカルの Ollama サーバーを使わずに、ホストされた Ollama モデルを使用したい場合
    - ollama-cloud のプロバイダー ID、キー、またはエンドポイントが必要です
summary: OpenClaw で Ollama Cloud を直接使用する
title: Ollama Cloud
x-i18n:
    generated_at: "2026-07-26T09:42:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 966e5237e37134cef109979079db390e9844714001e921e7976dc8ca7f58bcc4
    source_path: providers/ollama-cloud.md
    workflow: 16
---

Ollama Cloud は、Ollama がホストするモデル API です。`ollama-cloud` プロバイダーは、
ローカルの Ollama サーバーやクラウドモードにサインインしたローカルの Ollama アプリを使用せず、
Ollama ネイティブの `/api/chat` API を介して `https://ollama.com` を直接呼び出します。
`ollama-cloud/kimi-k2.6` のようなモデル参照を使用します。

OpenClaw は `ollama-cloud` を独自のプロバイダー ID として登録するため、
クラウド専用の認証情報、ライブカタログ検出、モデル選択がローカルの
`ollama` ホストと混在することはありません。ローカル Ollama、クラウドとローカルの
ハイブリッドルーティング、埋め込み、カスタムホストの詳細については、[Ollama](/ja-JP/providers/ollama) を参照してください。

## セットアップ

[ollama.com/settings/keys](https://ollama.com/settings/keys) で Ollama Cloud API キーを作成し、次を実行します。

```bash
openclaw onboard --auth-choice ollama-cloud
```

または、次を設定します。

```bash
export OLLAMA_API_KEY="<your-ollama-cloud-api-key>" # pragma: allowlist secret
```

非対話型オンボーディングでは、キーを直接指定できます。

```bash
openclaw onboard --auth-choice ollama-cloud --ollama-cloud-api-key "<key>"
```

オンボーディングでは、デフォルトモデルが `ollama-cloud/kimi-k2.5:cloud` に設定されます。

## デフォルト

- プロバイダー: `ollama-cloud`
- ベース URL: `https://ollama.com`
- 環境変数: `OLLAMA_API_KEY`
- API 形式: Ollama ネイティブ `/api/chat`
- オンボーディングのデフォルトモデル: `ollama-cloud/kimi-k2.5:cloud`

## Ollama Cloud を選ぶ場合

- ローカルで `ollama serve` を実行せずに、ホストされた Ollama モデルを使用したい場合。
- OpenClaw がローカル Ollama に使用するものと同じネイティブ Ollama チャット API 形式を、
  `https://ollama.com` に向けて使用したい場合。
- Ollama のホスト型カタログにすでに存在するモデルへのシンプルなクラウド経由のアクセスが必要な場合。
- ローカルでのモデル取得、ローカル GPU の制御、LAN 専用の推論が不要な場合。

サインイン済みの Ollama ホストを介したローカル専用またはクラウドとローカルの
ルーティングが必要な場合は、代わりに [Ollama](/ja-JP/providers/ollama) を使用してください。
`/v1/chat/completions` のセマンティクスや、プロバイダー固有の OpenAI 形式の機能が必要な場合は、
代わりに OpenAI 互換プロバイダーを使用してください。

## モデル

このプロバイダーには API キーが必要です。キーがない場合は無効のままです。キーがある場合、
OpenClaw はホスト型カタログから Ollama Cloud モデルをライブ検出します。

```bash
openclaw models list --provider ollama-cloud
openclaw models set ollama-cloud/kimi-k2.6
```

ライブカタログのホスト型 ID には、`deepseek-v4-flash`、`glm-5`、
`gpt-oss:20b`、`kimi-k2.6`、`minimax-m2.7` などがあります。ライブ検出で
何も返されない場合、OpenClaw は同梱されている `kimi-k2.5:cloud`、
`minimax-m2.7:cloud`、`glm-5.1:cloud`、`glm-5.2:cloud` の行にフォールバックします。

モデル ID はクラウドカタログの ID であり、ローカルで取得する際の名前ではありません。
モデル名がローカルの Ollama ホストでは機能してもホスト型カタログに存在しない場合は、
代わりにそのローカルホストで `ollama` プロバイダーを使用してください。

## ライブテスト

Ollama Cloud API キーのスモークテストでは、Ollama ライブテストをホスト型
エンドポイントに向け、現在のカタログからモデルを選択します。

```bash
export OLLAMA_API_KEY="<your-ollama-cloud-api-key>" # pragma: allowlist secret

OPENCLAW_LIVE_TEST=1 \
OPENCLAW_LIVE_OLLAMA=1 \
OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com \
OPENCLAW_LIVE_OLLAMA_MODEL=kimi-k2.6 \
pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

クラウドスモークでは、テキスト、ネイティブストリーム、Web 検索を実行します。Web 検索を
スキップするには `OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0` を設定します。Ollama Cloud API キーでは
`/api/embed` が許可されていない場合があるため、`https://ollama.com` では
デフォルトで埋め込みをスキップします。強制的に実行するには `OPENCLAW_LIVE_OLLAMA_EMBEDDINGS=1` を使用します。

## トラブルシューティング

- `Ollama Cloud requires an API key` / `Set OLLAMA_API_KEY` エラー: 実際のクラウド API キーを指定してください。
  ローカルの `ollama-local` マーカーは、ローカルまたはプライベートの Ollama ホスト専用です。
- 不明なモデルのエラー: `openclaw models list --provider ollama-cloud` を実行し、
  ホスト型モデル ID を正確にコピーしてください。
- カスタム Ollama ホストでのツール呼び出しまたは生の JSON の問題:
  OpenAI 互換の `/v1` URL を誤って使用していないか確認してください。Ollama のルートでは、
  `/v1` サフィックスのないネイティブベース URL を使用する必要があります。

## 関連情報

- [Ollama](/ja-JP/providers/ollama)
- [モデルプロバイダー](/ja-JP/concepts/model-providers)
- [すべてのプロバイダー](/ja-JP/providers/index)
