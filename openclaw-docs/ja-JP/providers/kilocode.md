---
read_when:
    - 多数の LLM に単一の API キーを使用したい場合
    - OpenClaw で Kilo Gateway 経由でモデルを実行する場合
summary: Kilo Gateway の統合 API を使用して OpenClaw から多くのモデルにアクセスする
title: Kilo Gateway
x-i18n:
    generated_at: "2026-07-26T10:17:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0246a1a77f4265168b213e0167360e1cd89dc2ca864997f08cae5331037f9e89
    source_path: providers/kilocode.md
    workflow: 16
---

Kilo Gateway は、単一の OpenAI 互換エンドポイントと API キーを通じて、多数のモデルにリクエストをルーティングします。

| プロパティ | 値                              |
| -------- | ---------------------------------- |
| プロバイダー | `kilocode`                         |
| 認証     | `KILOCODE_API_KEY`                 |
| API      | OpenAI 互換                  |
| ベース URL | `https://api.kilo.ai/api/gateway/` |

## Plugin のインストール

```bash
openclaw plugins install @openclaw/kilocode-provider
openclaw gateway restart
```

## セットアップ

<Steps>
  <Step title="アカウントを作成">
    [app.kilo.ai](https://app.kilo.ai) にアクセスし、サインインするかアカウントを作成してから、API キーを生成します。
  </Step>
  <Step title="オンボーディングを実行">
    ```bash
    openclaw onboard --auth-choice kilocode-api-key
    ```

    または、環境変数を直接設定します。

    ```bash
    export KILOCODE_API_KEY="<your-kilocode-api-key>" # pragma: allowlist secret
    ```

  </Step>
  <Step title="モデルが利用可能であることを確認">
    ```bash
    openclaw models list --provider kilocode
    ```
  </Step>
</Steps>

## デフォルトモデルとカタログ

デフォルトモデルは、Kilo Gateway のバランス型スマートルーティング階層である `kilocode/kilo-auto/balanced` です。
OpenClaw は、そのタスクからアップストリームモデルへのマッピングを公開していません。
`kilo-auto/balanced` の背後にあるルーティングは Kilo Gateway が管理します。

起動時に OpenClaw は `GET https://api.kilo.ai/api/gateway/models` を照会し、検出したモデルを
静的フォールバックカタログより前にマージします。静的フォールバックに含まれるのは
`kilocode/kilo-auto/balanced` (`Auto Balanced`, `input: ["text", "image"]`, `reasoning: true`,
`contextWindow: 1000000`, `maxTokens: 65536`) のみです。

Gateway 上の任意のモデルは `kilocode/<upstream-id>` として指定できます（例:
`kilocode/anthropic/claude-sonnet-4`, `kilocode/openai/gpt-5.5`）。検出された完全な一覧を確認するには、
`/models kilocode` または `openclaw models list --provider kilocode` を実行します。

## 設定例

```json5
{
  env: { KILOCODE_API_KEY: "<your-kilocode-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "kilocode/kilo-auto/balanced" },
    },
  },
}
```

## 動作に関する注記

<AccordionGroup>
  <Accordion title="トランスポートと互換性">
    Kilo Gateway は OpenRouter 互換であるため、ネイティブな OpenAI リクエスト形式ではなく、
    プロキシ形式の OpenAI 互換リクエストパスを使用します（`store` なし、OpenAI の reasoning-effort ペイロードなし）。

    - Gemini ベースの Kilo 参照は、引き続きプロキシ Gemini パスを使用します。OpenClaw はそこで Gemini の思考
      シグネチャをサニタイズしますが、ネイティブ Gemini のリプレイ検証やブートストラップ書き換えは有効にしません。
    - リクエストでは、API キーから構築された Bearer トークンを使用します。

  </Accordion>

  <Accordion title="ストリームラッパーと推論">
    Kilo ストリームラッパーは `X-KILOCODE-FEATURE` リクエストヘッダー（デフォルトは `openclaw`、
    `KILOCODE_FEATURE` 環境変数で上書き可能）を追加し、対応するモデルの
    reasoning-effort ペイロードを正規化します。

    <Warning>
    `kilocode/kilo-auto/balanced` および `x-ai/*` の参照では、reasoning-effort の注入がスキップされます。推論サポートが必要な場合は、
    `kilocode/anthropic/claude-sonnet-4` などの具体的なモデル参照を使用してください。
    </Warning>

  </Accordion>

  <Accordion title="トラブルシューティング">
    - 起動時にモデル検出が失敗すると、OpenClaw は `kilocode/kilo-auto/balanced` を含む静的カタログにフォールバックします。
    - API キーが有効であり、Kilo アカウントで目的のモデルが有効になっていることを確認してください。
    - Gateway をデーモンとして実行する場合は、`KILOCODE_API_KEY` がそのプロセスで利用できることを確認してください（たとえば `~/.openclaw/.env` 内、または `env.shellEnv` 経由）。

  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/configuration-reference" icon="gear">
    OpenClaw の完全な設定リファレンス。
  </Card>
  <Card title="Kilo Gateway" href="https://app.kilo.ai" icon="arrow-up-right-from-square">
    Kilo Gateway のダッシュボード、API キー、アカウント管理。
  </Card>
</CardGroup>
