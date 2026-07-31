---
read_when:
    - Synthetic をモデルプロバイダーとして使用したい場合
    - Synthetic API キーまたはベース URL の設定が必要です
summary: OpenClaw で Synthetic の Anthropic 互換 API を使用する
title: Synthetic
x-i18n:
    generated_at: "2026-07-26T09:42:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3f6cc89a7b837f57555d176ce78e62a39095d4ef0765c96b6b7b93ffebd7388
    source_path: providers/synthetic.md
    workflow: 16
---

[Synthetic](https://synthetic.new) は Anthropic 互換エンドポイントを公開します。
OpenClaw はこれを `synthetic` プロバイダーとしてバンドルし、Anthropic
Messages API を使用します。

| プロパティ | 値                                    |
| ---------- | ------------------------------------- |
| プロバイダー | `synthetic`                  |
| 認証       | `SYNTHETIC_API_KEY`                   |
| API        | Anthropic Messages                    |
| ベース URL | `https://api.synthetic.new/anthropic`                    |

## はじめに

<Steps>
  <Step title="API キーを取得">
    Synthetic アカウントから `SYNTHETIC_API_KEY` を取得するか、オンボーディングで
    入力を求めるようにします。
  </Step>
  <Step title="オンボーディングを実行">
    ```bash
    openclaw onboard --auth-choice synthetic-api-key
    ```
  </Step>
  <Step title="デフォルトモデルを確認">
    オンボーディングにより、デフォルトモデルは次のように設定されます。
    ```text
    synthetic/hf:MiniMaxAI/MiniMax-M3
    ```
  </Step>
</Steps>

<Warning>
OpenClaw の Anthropic クライアントはベース URL に `/v1` を自動的に追加するため、
`https://api.synthetic.new/anthropic`（`/anthropic/v1` ではありません）を使用してください。Synthetic が
ベース URL を変更した場合は、`models.providers.synthetic.baseUrl` を上書きしてください。
</Warning>

## 設定例

```json5
{
  env: { SYNTHETIC_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
      models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "hf:MiniMaxAI/MiniMax-M3",
            name: "MiniMax M3",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

## 組み込みカタログ

すべての Synthetic モデルでは、コスト（入力／出力／キャッシュ）に `0` が使用されます。サービスの利用可否については、Synthetic の
[現在のモデル一覧](https://dev.synthetic.new/docs/api/models)を参照してください。

| モデル ID                                           | コンテキストウィンドウ | 最大トークン数 | 推論 | 入力             |
| --------------------------------------------------- | ---------------------- | -------------- | ---- | ---------------- |
| `hf:MiniMaxAI/MiniMax-M3`                                  | 262,144                | 65,536         | あり | テキスト + 画像 |
| `hf:moonshotai/Kimi-K2.7-Code`                                  | 262,144                | 8,192          | あり | テキスト + 画像 |
| `hf:nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4`                                  | 262,144                | 8,192          | あり | テキスト         |
| `hf:openai/gpt-oss-120b`                                  | 131,072                | 8,192          | あり | テキスト         |
| `hf:Qwen/Qwen3.6-27B`                                  | 262,144                | 81,920         | あり | テキスト + 画像 |
| `hf:zai-org/GLM-4.7-Flash`                                  | 196,608                | 131,072        | あり | テキスト         |
| `hf:zai-org/GLM-5.2`                                  | 524,288                | 131,072        | あり | テキスト         |

<Tip>
モデル参照は `synthetic/<modelId>` の形式を使用します。アカウントで利用可能な
すべてのモデルを確認するには、`openclaw models list --provider synthetic` を使用してください。
</Tip>

<AccordionGroup>
  <Accordion title="モデル許可リスト">
    モデル許可リスト（`agents.defaults.modelPolicy.allow`）を有効にする場合は、使用する予定の
    Synthetic モデルをすべて追加してください。許可リストにないモデルは
    エージェントから非表示になります。
  </Accordion>

  <Accordion title="ベース URL の上書き">
    Synthetic が API エンドポイントを変更した場合は、ベース URL を上書きします。

    ```json5
    {
      models: {
        providers: {
          synthetic: {
            baseUrl: "https://new-api.synthetic.new/anthropic",
          },
        },
      },
    }
    ```

    OpenClaw は引き続き `/v1` を自動的に追加します。

  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルプロバイダー" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダーのルール、モデル参照、フェイルオーバー動作。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/configuration-reference" icon="gear">
    プロバイダー設定を含む完全な設定スキーマ。
  </Card>
  <Card title="Synthetic" href="https://synthetic.new" icon="arrow-up-right-from-square">
    Synthetic のダッシュボードと API ドキュメント。
  </Card>
</CardGroup>
