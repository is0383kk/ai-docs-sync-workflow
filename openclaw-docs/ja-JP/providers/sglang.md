---
read_when:
    - ローカルの SGLang サーバーで OpenClaw を実行する場合
    - 独自のモデルで OpenAI 互換の /v1 エンドポイントを使用したい場合
summary: SGLang（OpenAI 互換のセルフホスト型サーバー）で OpenClaw を実行する
title: SGLang
x-i18n:
    generated_at: "2026-07-26T09:17:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54a7805315a7d65fdd2c7c9b6836aa2faccc88db7802cce0ba8c2d4a1aac9d65
    source_path: providers/sglang.md
    workflow: 16
---

SGLang は、OpenAI 互換の HTTP API を介してオープンウェイトモデルを提供します。OpenClaw は、利用可能なモデルを自動検出する `openai-completions` プロバイダーファミリーを使用して SGLang に接続します。

| プロパティ                | 値                                                           |
| ------------------------- | ------------------------------------------------------------ |
| プロバイダー ID           | `sglang`                                                     |
| Plugin                    | バンドル済み、`enabledByDefault: true`                            |
| 認証環境変数              | `SGLANG_API_KEY`（サーバーに認証がない場合は任意の空でない値） |
| オンボーディングフラグ    | `--auth-choice sglang`                                       |
| API                       | OpenAI 互換（`openai-completions`）                     |
| デフォルトのベース URL    | `http://127.0.0.1:30000/v1`                                  |
| デフォルトモデルのプレースホルダー | `sglang/Qwen/Qwen3-8B`                                       |
| ストリーミング使用量      | はい（`supportsStreamingUsage: true`）                         |
| 料金                      | 外部無料としてマーク（`modelPricing.external: false`）        |

また、`SGLANG_API_KEY` でオプトインすると、OpenClaw は SGLang から利用可能なモデルを**自動検出**します。カスタム SGLang ベース URL も設定しつつ検出を動的に保つには、`agents.defaults.models` で `sglang/*` を使用します。以下の[モデル検出（暗黙的プロバイダー）](#model-discovery-implicit-provider)を参照してください。

## はじめに

<Steps>
  <Step title="SGLang を起動する">
    OpenAI 互換サーバーで SGLang を起動します。ベース URL は
    `/v1` エンドポイント（例：`/v1/models`、`/v1/chat/completions`）を公開する必要があります。SGLang は
    通常、次の場所で実行されます。

    - `http://127.0.0.1:30000/v1`

  </Step>
  <Step title="API キーを設定する">
    サーバーに認証が設定されていない場合は、任意の値を使用できます。

    ```bash
    export SGLANG_API_KEY="sglang-local"
    ```

  </Step>
  <Step title="オンボーディングを実行するか、モデルを直接設定する">
    ```bash
    openclaw onboard
    ```

    または、モデルを手動で設定します。

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "sglang/your-model-id" },
        },
      },
    }
    ```

  </Step>
</Steps>

## モデル検出（暗黙的プロバイダー）

`SGLANG_API_KEY` が設定されている（または認証プロファイルが存在する）場合に、
`models.providers.sglang` を定義していなければ、OpenClaw は次を問い合わせます。

- `GET http://127.0.0.1:30000/v1/models`

そして、返された ID をモデルエントリに変換します。

<Note>
`models.providers.sglang` を明示的に設定した場合、OpenClaw はデフォルトで宣言された
モデルを使用します。設定済みプロバイダーの `/models` エンドポイントを
OpenClaw に問い合わせさせ、公開されているすべての SGLang モデルを含めるには、
`agents.defaults.models` に `"sglang/*": {}` を追加します。
</Note>

## 明示的な設定（手動モデル）

次の場合は明示的な設定を使用します。

- SGLang が別のホストまたはポートで実行されている。
- `contextWindow`/`maxTokens` の値を固定したい。
- サーバーで実際の API キーが必要である（またはヘッダーを制御したい）。

```json5
{
  models: {
    providers: {
      sglang: {
        baseUrl: "http://127.0.0.1:30000/v1",
        apiKey: "${SGLANG_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "your-model-id",
            name: "Local SGLang Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

## 高度な設定

<AccordionGroup>
  <Accordion title="プロキシ形式の動作">
    SGLang はネイティブの OpenAI エンドポイントではなく、プロキシ形式の OpenAI 互換
    `/v1` バックエンドとして扱われます。

    | 動作 | SGLang |
    |----------|--------|
    | OpenAI 専用のリクエスト整形 | 適用されない |
    | `service_tier`、Responses `store`、プロンプトキャッシュのヒント | 送信されない |
    | 推論互換ペイロードの整形 | 適用されない |
    | 非表示の帰属ヘッダー（`originator`、`version`、`User-Agent`） | カスタム SGLang ベース URL では挿入されない |

  </Accordion>

  <Accordion title="トラブルシューティング">
    **サーバーに接続できない**

    サーバーが実行中で応答していることを確認します。

    ```bash
    curl http://127.0.0.1:30000/v1/models
    ```

    **認証エラー**

    リクエストが認証エラーで失敗する場合は、サーバー設定と一致する実際の
    `SGLANG_API_KEY` を設定するか、`models.providers.sglang` で
    プロバイダーを明示的に設定します。

    <Tip>
    SGLang を認証なしで実行する場合、モデル検出にオプトインするには
    `SGLANG_API_KEY` に任意の空でない値を設定すれば十分です。
    </Tip>

  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/configuration-reference" icon="gear">
    プロバイダーエントリを含む完全な設定スキーマ。
  </Card>
</CardGroup>
