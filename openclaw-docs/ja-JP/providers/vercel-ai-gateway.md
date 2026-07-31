---
read_when:
    - OpenClaw で Vercel AI Gateway を使用する場合
    - API キーの環境変数または CLI 認証方式の選択が必要です
summary: Vercel AI Gateway のセットアップ（認証 + モデル選択）
title: Vercel AI Gateway
x-i18n:
    generated_at: "2026-07-26T10:29:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1e4776604491900a914e75caebfd7e27a81e9f859213f5bd5b25582a923d92a
    source_path: providers/vercel-ai-gateway.md
    workflow: 16
---

[Vercel AI Gateway](https://vercel.com/ai-gateway) は、単一のエンドポイントを通じて数百のモデルにアクセスできる統一 API を提供します。

| プロパティ      | 値                                  |
| ------------- | -------------------------------------- |
| プロバイダー      | `vercel-ai-gateway`                    |
| パッケージ       | `@openclaw/vercel-ai-gateway-provider` |
| 認証          | `AI_GATEWAY_API_KEY`                   |
| API           | Anthropic Messages 互換          |
| ベース URL      | `https://ai-gateway.vercel.sh`         |
| モデルカタログ | `/v1/models` を介して自動検出       |

<Tip>
OpenClaw は Gateway の `/v1/models` カタログを自動検出するため、
`/models vercel-ai-gateway` チャットコマンドと
`openclaw models list --provider vercel-ai-gateway` の両方に、
`vercel-ai-gateway/openai/gpt-5.5` や
`vercel-ai-gateway/moonshotai/kimi-k2.6` などの最新のモデル参照が含まれます。
</Tip>

## はじめに

<Steps>
  <Step title="Plugin をインストールする">
    ```bash
    openclaw plugins install @openclaw/vercel-ai-gateway-provider
    ```
  </Step>
  <Step title="API キーを設定する">
    ```bash
    openclaw onboard --auth-choice ai-gateway-api-key
    ```
  </Step>
  <Step title="デフォルトモデルを設定する">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "vercel-ai-gateway/anthropic/claude-opus-4.6" },
        },
      },
    }
    ```
  </Step>
  <Step title="モデルが利用可能であることを確認する">
    ```bash
    openclaw models list --provider vercel-ai-gateway
    ```
  </Step>
</Steps>

## 非対話型の例

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice ai-gateway-api-key \
  --ai-gateway-api-key "$AI_GATEWAY_API_KEY"
```

## モデル ID の短縮表記

OpenClaw は実行時に Claude の短縮モデル参照を正規化します。

| 短縮入力                     | 正規化されたモデル参照                          |
| ----------------------------------- | --------------------------------------------- |
| `vercel-ai-gateway/claude-opus-4.6` | `vercel-ai-gateway/anthropic/claude-opus-4.6` |
| `vercel-ai-gateway/opus-4.6`        | `vercel-ai-gateway/anthropic/claude-opus-4-6` |

<Tip>
設定ではどちらの形式も使用できます。OpenClaw は正規の
`anthropic/...` 参照を自動的に解決します。
</Tip>

## 高度な設定

<AccordionGroup>
  <Accordion title="デーモンプロセス用の環境変数">
    OpenClaw Gateway をデーモン（launchd/systemd）として実行する場合は、
    そのプロセスで `AI_GATEWAY_API_KEY` を利用できることを確認してください。

    <Warning>
    対話型シェルでのみエクスポートされたキーは、その環境を明示的にインポートしない限り、
    launchd/systemd デーモンからは参照できません。Gateway
    プロセスがキーを読み取れるように、`~/.openclaw/.env` または
    `env.shellEnv` を使用してキーを設定してください。
    </Warning>

  </Accordion>

  <Accordion title="プロバイダーのルーティング">
    Vercel AI Gateway は、モデル参照のプレフィックスで指定されたアップストリームプロバイダーに
    各リクエストをルーティングします。たとえば、`vercel-ai-gateway/anthropic/claude-opus-4.6` は
    Anthropic 経由、`vercel-ai-gateway/openai/gpt-5.5` は
    OpenAI 経由、`vercel-ai-gateway/moonshotai/kimi-k2.6` は
    MoonshotAI 経由でルーティングされます。1 つの `AI_GATEWAY_API_KEY` ですべてのアップストリームプロバイダーを認証します。
  </Accordion>
  <Accordion title="思考レベル">
    OpenClaw がアップストリームモデルのプレフィックスを認識する場合、
    `/think` オプションはそのプレフィックスに従います。
    `vercel-ai-gateway/anthropic/...` は Claude 4.6 モデルの適応型デフォルトを含む
    Claude の思考プロファイルを使用します。信頼済みの
    `vercel-ai-gateway/openai/...` 参照（`gpt-5.2` 以降、および
    `gpt-5.1-codex` までの Codex バリアント）では `/think xhigh` が公開されます。
    その他の名前空間付き参照では、カタログのメタデータで追加レベルが宣言されていない限り、
    標準の推論レベルが維持されます。
  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="トラブルシューティング" href="/ja-JP/help/troubleshooting" icon="wrench">
    一般的なトラブルシューティングとよくある質問。
  </Card>
</CardGroup>
