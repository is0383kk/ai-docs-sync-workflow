---
read_when:
    - ローカルのInferrsサーバーに対してOpenClawを実行する場合
    - inferrs を通じて Gemma または別のモデルを提供している場合
    - inferrs 向けの正確な OpenClaw 互換フラグが必要です
summary: inferrs（OpenAI 互換ローカルサーバー）経由で OpenClaw を実行する
title: Inferrs
x-i18n:
    generated_at: "2026-07-26T09:48:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8b9b6fe337a2ec6536332dd62840052fd802fad0a5f3d885ce137523266ff3c9
    source_path: providers/inferrs.md
    workflow: 16
---

[inferrs](https://github.com/ericcurtin/inferrs) は、OpenAI 互換の `/v1` API の背後でローカルモデルを提供します。OpenClaw は汎用 `openai-completions` アダプターを介してこれと通信します。

| プロパティ         | 値                                                                   |
| ------------------ | -------------------------------------------------------------------- |
| プロバイダー ID    | `inferrs`（カスタム。`models.providers.inferrs` 配下で設定）        |
| Plugin             | なし — OpenClaw に同梱されたプロバイダー Plugin ではありません      |
| 認証環境変数       | 不要。inferrs サーバーで認証を使用しない場合は任意の値を使用できます |
| API                | OpenAI 互換（`openai-completions`）                                    |
| 推奨ベース URL     | `http://127.0.0.1:8080/v1`（または inferrs サーバーが待ち受ける場所）        |

<Note>
  `inferrs` は専用の OpenClaw プロバイダー Plugin ではなく、カスタムのセルフホスト型 OpenAI 互換バックエンドです。オンボーディングの認証オプションを選択する代わりに、`models.providers.inferrs` 配下で設定します。自動検出機能を備えた同梱 Plugin については、[SGLang](/ja-JP/providers/sglang) または [vLLM](/ja-JP/providers/vllm) を参照してください。
</Note>

## はじめに

<Steps>
  <Step title="モデルを指定して inferrs を起動する">
    ```bash
    inferrs serve google/gemma-4-E2B-it \
      --host 127.0.0.1 \
      --port 8080 \
      --device metal
    ```
  </Step>
  <Step title="サーバーに到達できることを確認する">
    ```bash
    curl http://127.0.0.1:8080/health
    curl http://127.0.0.1:8080/v1/models
    ```
  </Step>
  <Step title="OpenClaw のプロバイダーエントリを追加する">
    明示的なプロバイダーエントリを追加し、デフォルトモデルがそれを参照するようにします。以下の設定例を参照してください。
  </Step>
</Steps>

## 完全な設定例

ローカル `inferrs` サーバー上の Gemma 4：

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/google/gemma-4-E2B-it" },
      models: {
        "inferrs/google/gemma-4-E2B-it": {
          alias: "Gemma 4 (inferrs)",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

## オンデマンド起動

OpenClaw は、`inferrs/...` モデルが選択された場合に限り、`inferrs` 自体を起動できます。同じプロバイダーエントリに `localService` を追加します。

```json5
{
  models: {
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "/opt/homebrew/bin/inferrs",
          args: [
            "serve",
            "google/gemma-4-E2B-it",
            "--host",
            "127.0.0.1",
            "--port",
            "8080",
            "--device",
            "metal",
          ],
          healthUrl: "http://127.0.0.1:8080/v1/models",
          readyTimeoutMs: 180000,
          idleStopMs: 0,
        },
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

`command` は絶対パスである必要があります。Gateway ホスト上で `which inferrs` を実行し、そのパスを使用してください。全フィールドのリファレンス：[ローカルモデルサービス](/ja-JP/gateway/local-model-services)。

## 高度な設定

<AccordionGroup>
  <Accordion title="requiresStringContent が重要な理由">
    一部の `inferrs` Chat Completions ルートでは、構造化されたコンテンツパート配列ではなく、文字列の `messages[].content` のみを受け付けます。

    <Warning>
    OpenClaw の実行が次のエラーで失敗する場合：

    ```text
    messages[1].content: invalid type: sequence, expected a string
    ```

    モデルエントリに `compat.requiresStringContent: true` を設定してください。これにより OpenClaw は、リクエストを送信する前に、テキストのみのコンテンツパートをプレーン文字列に変換します。
    </Warning>

  </Accordion>

  <Accordion title="Gemma とツールスキーマに関する注意点">
    一部の `inferrs` と Gemma の組み合わせでは、小規模な直接 `/v1/chat/completions` リクエストは受け付けますが、OpenClaw のエージェントランタイムによる完全なターンでは失敗します。まずツールスキーマのサーフェスを無効にしてみてください。

    ```json5
    compat: {
      requiresStringContent: true,
      supportsTools: false
    }
    ```

    これにより、制約の厳しいローカルバックエンドにかかるプロンプトの負荷が軽減されます。小規模な直接リクエストが引き続き動作する一方で、通常の OpenClaw エージェントターンが `inferrs` 内でクラッシュし続ける場合は、OpenClaw のトランスポートの問題ではなく、上流のモデルまたはサーバーの制限として扱ってください。

  </Accordion>

  <Accordion title="手動スモークテスト">
    設定後、両方のレイヤーをテストします。

    ```bash
    curl http://127.0.0.1:8080/v1/chat/completions \
      -H 'content-type: application/json' \
      -d '{"model":"google/gemma-4-E2B-it","messages":[{"role":"user","content":"What is 2 + 2?"}],"stream":false}'
    ```

    ```bash
    openclaw infer model run \
      --model inferrs/google/gemma-4-E2B-it \
      --prompt "What is 2 + 2? Reply with one short sentence." \
      --json
    ```

    1 つ目のコマンドは動作するものの 2 つ目が失敗する場合は、以下のトラブルシューティングを参照してください。

  </Accordion>

  <Accordion title="プロキシ形式の動作">
    `inferrs` は `openai-responses` ではなく汎用 `openai-completions` アダプターを使用するため、OpenAI ネイティブ専用のリクエスト整形は適用されません。`service_tier`、Responses `store`、プロンプトキャッシュのヒント、および OpenAI の推論互換ペイロード整形は送信されません。
  </Accordion>
</AccordionGroup>

## トラブルシューティング

<AccordionGroup>
  <Accordion title="curl /v1/models が失敗する">
    `inferrs` が実行されていない、到達できない、または設定したホスト／ポートにバインドされていません。サーバーが起動し、そのアドレスで待ち受けていることを確認してください。
  </Accordion>

  <Accordion title="messages[].content に文字列が必要と表示される">
    モデルエントリに `compat.requiresStringContent: true` を設定してください（前述を参照）。
  </Accordion>

  <Accordion title="直接の /v1/chat/completions 呼び出しは成功するが openclaw infer model run は失敗する">
    ツールスキーマのサーフェスを無効にするため、`compat.supportsTools: false` を設定してください（前述の Gemma に関する注意点を参照）。
  </Accordion>

  <Accordion title="大規模なエージェントターンで inferrs が引き続きクラッシュする">
    スキーマエラーが解消されても、大規模なエージェントターンで `inferrs` が引き続きクラッシュする場合は、上流の `inferrs` またはモデルの制限として扱ってください。プロンプトの負荷を減らすか、バックエンド／モデルを切り替えてください。
  </Accordion>
</AccordionGroup>

<Tip>
一般的なヘルプについては、[トラブルシューティング](/ja-JP/help/troubleshooting)と[よくある質問](/ja-JP/help/faq)を参照してください。
</Tip>

## 関連項目

<CardGroup cols={2}>
  <Card title="ローカルモデル" href="/ja-JP/gateway/local-models" icon="server">
    ローカルモデルサーバーに対して OpenClaw を実行します。
  </Card>
  <Card title="ローカルモデルサービス" href="/ja-JP/gateway/local-model-services" icon="play">
    設定済みプロバイダー向けに、ローカルモデルサーバーをオンデマンドで起動します。
  </Card>
  <Card title="Gateway のトラブルシューティング" href="/ja-JP/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail" icon="wrench">
    プローブには成功するもののエージェント実行には失敗するローカル OpenAI 互換バックエンドをデバッグします。
  </Card>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    すべてのプロバイダー、モデル参照、フェイルオーバー動作の概要です。
  </Card>
</CardGroup>
