---
read_when:
    - LiteLLM プロキシ経由で OpenClaw をルーティングする場合
    - LiteLLM を通じたコスト追跡、ログ記録、またはモデルルーティングが必要な場合
summary: LiteLLM Proxy 経由で OpenClaw を実行し、モデルへのアクセスとコスト追跡を一元化する
title: LiteLLM
x-i18n:
    generated_at: "2026-07-26T10:28:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 22451f0eefcf991a602409701fc752f97600a67752c67304137c7f17f3dd1a16
    source_path: providers/litellm.md
    workflow: 16
---

[LiteLLM](https://litellm.ai) は、100 以上のモデルプロバイダーに統一 API で接続できるオープンソースの LLM Gateway です。OpenClaw の設定を変更せずに、OpenClaw を LiteLLM 経由でルーティングして、コストの一元追跡、ログ記録、使用上限付き仮想キー、バックエンドのフェイルオーバーを利用できます。

## クイックスタート

<Tabs>
  <Tab title="オンボーディング（推奨）">
    ```bash
    openclaw onboard --auth-choice litellm-api-key
    ```

    リモートプロキシに対して非対話形式でセットアップする場合は、プロキシ URL を明示的に渡します。

    ```bash
    openclaw onboard --non-interactive --accept-risk --auth-choice litellm-api-key \
      --litellm-api-key "$LITELLM_API_KEY" --custom-base-url "https://litellm.example/v1"
    ```

  </Tab>

  <Tab title="手動セットアップ">
    <Steps>
      <Step title="LiteLLM Proxy を起動">
        ```bash
        pip install 'litellm[proxy]'
        litellm --model claude-opus-4-6
        ```
      </Step>
      <Step title="OpenClaw の接続先を LiteLLM に設定">
        ```bash
        export LITELLM_API_KEY="your-litellm-key"
        openclaw
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

## 設定

```json5
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "claude-opus-4-6",
            name: "Claude Opus 4.6",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 200000,
            maxTokens: 64000,
          },
          {
            id: "gpt-4o",
            name: "GPT-4o",
            reasoning: false,
            input: ["text", "image"],
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "litellm/claude-opus-4-6" },
    },
  },
}
```

オンボーディングによって書き込まれるデフォルトモデルは `litellm/claude-opus-4-6` です。

## 画像生成

LiteLLM は、OpenAI 互換の `/images/generations` および
`/images/edits` ルートを介して `image_generate` ツールのバックエンドとして機能できます。デフォルトの画像モデルは `gpt-image-2` です。別のモデルを使用するには、
`agents.defaults.mediaModels.image` で設定します。

```json5
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
      },
    },
  },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "litellm/gpt-image-2",
        timeoutMs: 180_000,
      },
    },
  },
}
```

ループバック LiteLLM URL（`http://localhost:4000`、`127.0.0.1`、`::1`、`host.docker.internal`）は、
プライベートネットワークのグローバルなオーバーライドなしで動作します。LAN でホストされているプロキシの場合は、API キーがそのホストに送信されるため、
`models.providers.litellm.request.allowPrivateNetwork: true` を設定します。

## 高度な設定

<AccordionGroup>
  <Accordion title="仮想キー">
    使用上限を設定した OpenClaw 専用キーを作成します。

    ```bash
    curl -X POST "http://localhost:4000/key/generate" \
      -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
      -H "Content-Type: application/json" \
      -d '{
        "key_alias": "openclaw",
        "max_budget": 50.00,
        "budget_duration": "monthly"
      }'
    ```

    生成されたキーを `LITELLM_API_KEY` として使用します。

  </Accordion>

  <Accordion title="モデルのルーティング">
    LiteLLM は、モデルリクエストを異なるバックエンドにルーティングできます。LiteLLM の `config.yaml` で設定します。

    ```yaml
    model_list:
      - model_name: claude-opus-4-6
        litellm_params:
          model: claude-opus-4-6
          api_key: os.environ/ANTHROPIC_API_KEY

      - model_name: gpt-4o
        litellm_params:
          model: gpt-4o
          api_key: os.environ/OPENAI_API_KEY
    ```

    OpenClaw は引き続き `claude-opus-4-6` をリクエストし、LiteLLM がルーティングを処理します。

  </Accordion>

  <Accordion title="使用状況の確認">
    ```bash
    # キー情報
    curl "http://localhost:4000/key/info" \
      -H "Authorization: Bearer sk-litellm-key"

    # 使用額ログ
    curl "http://localhost:4000/spend/logs" \
      -H "Authorization: Bearer $LITELLM_MASTER_KEY"
    ```

  </Accordion>

  <Accordion title="プロキシ動作に関する注意事項">
    - LiteLLM はデフォルトで `http://localhost:4000` で動作します。
    - OpenClaw は、LiteLLM のプロキシ形式の OpenAI 互換 `/v1` エンドポイントを介して接続します。
    - ネイティブ OpenAI 専用のリクエスト整形は、設定済みの LiteLLM ベース URL では適用されません。
      `service_tier`、Responses `store`、プロンプトキャッシュのヒント、OpenAI の reasoning-effort
      ペイロード整形はいずれも使用されません。
    - 非表示の OpenClaw 帰属ヘッダー（`originator`、`version`、`User-Agent`）は、検証済みの
      ネイティブ OpenAI エンドポイントにのみ送信されるため、カスタム LiteLLM ベース URL には挿入されません。
  </Accordion>
</AccordionGroup>

<Note>
一般的なプロバイダー設定とフェイルオーバーの動作については、[モデルプロバイダー](/ja-JP/concepts/model-providers)を参照してください。
</Note>

## 関連情報

<CardGroup cols={2}>
  <Card title="LiteLLM ドキュメント" href="https://docs.litellm.ai" icon="book">
    LiteLLM の公式ドキュメントおよび API リファレンスです。
  </Card>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    すべてのプロバイダー、モデル参照、フェイルオーバー動作の概要です。
  </Card>
  <Card title="設定" href="/ja-JP/gateway/configuration" icon="gear">
    完全な設定リファレンスです。
  </Card>
  <Card title="モデル" href="/ja-JP/concepts/models" icon="brain">
    モデルの選択方法と設定方法です。
  </Card>
</CardGroup>
