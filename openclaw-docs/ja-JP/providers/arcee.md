---
read_when:
    - OpenClaw で Arcee AI を使用する場合
    - API キーの環境変数または CLI 認証の選択が必要です
summary: Arcee AI のセットアップ（認証 + モデル選択）
title: Arcee AI
x-i18n:
    generated_at: "2026-07-26T09:57:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a4c2fc7b8d86dd0d2a300dfc48951657cbcfcd9250016f52c1804777b2966e11
    source_path: providers/arcee.md
    workflow: 16
---

[Arcee AI](https://arcee.ai) は、OpenAI 互換 API を通じて、エキスパート混合モデルの Trinity ファミリーを提供します。すべての Trinity モデルは Apache 2.0 ライセンスです。Arcee は OpenClaw の公式 Plugin ですが、コアにはバンドルされていないため、オンボーディングの前にインストールが必要です。

Arcee プラットフォームから直接、または [OpenRouter](/ja-JP/providers/openrouter) を通じて Arcee モデルにアクセスできます。

| プロパティ | 値                                                                                 |
| -------- | ------------------------------------------------------------------------------------- |
| プロバイダー | `arcee`                                                                               |
| 認証     | `ARCEEAI_API_KEY`（直接）または `OPENROUTER_API_KEY`（OpenRouter 経由）                   |
| API      | OpenAI 互換                                                                     |
| ベース URL | `https://api.arcee.ai/api/v1`（直接）または `https://openrouter.ai/api/v1`（OpenRouter） |

## Plugin をインストール

```bash
openclaw plugins install @openclaw/arcee-provider
openclaw gateway restart
```

## はじめに

<Tabs>
  <Tab title="直接（Arcee プラットフォーム）">
    <Steps>
      <Step title="API キーを取得">
        [Arcee AI](https://chat.arcee.ai/) で API キーを作成します。
      </Step>
      <Step title="オンボーディングを実行">
        ```bash
        openclaw onboard --auth-choice arceeai-api-key
        ```
      </Step>
      <Step title="デフォルトモデルを設定">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```
      </Step>
    </Steps>
  </Tab>

  <Tab title="OpenRouter 経由">
    <Steps>
      <Step title="API キーを取得">
        [OpenRouter](https://openrouter.ai/keys) で API キーを作成します。
      </Step>
      <Step title="オンボーディングを実行">
        ```bash
        openclaw onboard --auth-choice arceeai-openrouter
        ```
      </Step>
      <Step title="デフォルトモデルを設定">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```

        同じモデル参照を直接構成と OpenRouter 構成の両方で使用できます。
      </Step>
    </Steps>

  </Tab>
</Tabs>

## 非対話型セットアップ

<Tabs>
  <Tab title="直接（Arcee プラットフォーム）">
    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-api-key \
      --arceeai-api-key "$ARCEEAI_API_KEY"
    ```
  </Tab>

  <Tab title="OpenRouter 経由">
    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-openrouter \
      --openrouter-api-key "$OPENROUTER_API_KEY"
    ```
  </Tab>
</Tabs>

## Arcee 直接接続カタログ

| モデル参照                      | 名前                   | 入力 | コンテキスト | 最大出力 | コスト（100 万トークン当たりの入力／出力） | ツール | 注記                                     |
| ------------------------------ | ---------------------- | ----- | ------- | ---------- | -------------------- | ----- | ----------------------------------------- |
| `arcee/trinity-large-thinking` | Trinity Large Thinking | テキスト  | 256K    | 80K        | $0.25 / $0.90        | いいえ    | デフォルトモデル、拡張思考          |
| `arcee/trinity-large-preview`  | Trinity Large Preview  | テキスト  | 128K    | 16K        | $0.25 / $1.00        | はい   | 汎用、400B パラメーター、13B アクティブ  |
| `arcee/trinity-mini`           | Trinity Mini 26B       | テキスト  | 128K    | 80K        | $0.045 / $0.15       | はい   | 高速でコスト効率に優れる、関数呼び出し |

<Tip>
オンボーディングのプリセットは、`arcee/trinity-large-thinking` をデフォルトモデルとして設定します。
</Tip>

## OpenRouter カタログ

OpenRouter のオンボーディングでは、`arcee/trinity-large-preview` と `arcee/trinity-large-thinking` が利用可能になります。OpenClaw は、これらのプロバイダー修飾付きモデル参照を設定に保持し、OpenRouter の正規の `arcee-ai/*` ランタイム ID を送信します。Trinity Mini は OpenRouter では提供されなくなったため、このモデルには Arcee API の直接接続を使用してください。

## 対応機能

| 機能                                       | 対応状況                                    |
| --------------------------------------------- | -------------------------------------------- |
| ストリーミング                                     | はい                                          |
| ツール使用／関数呼び出し                   | はい（Trinity Mini、Trinity Large Preview）    |
| 構造化出力（JSON モードおよび JSON スキーマ） | はい                                          |
| 拡張思考                             | はい（Trinity Large Thinking、ツールは無効） |

<AccordionGroup>
  <Accordion title="環境に関する注意事項">
    Gateway がデーモン（launchd/systemd）として動作する場合は、`ARCEEAI_API_KEY`
    （または `OPENROUTER_API_KEY`）をそのプロセスから利用できるようにしてください。たとえば、
    `~/.openclaw/.env` または `env.shellEnv` 経由で設定します。
  </Accordion>

  <Accordion title="OpenRouter のルーティング">
    OpenRouter は同じ `arcee/trinity-large-thinking` OpenClaw モデル参照を使用します。
    OpenClaw は、正規の `arcee-ai/trinity-large-thinking`
    OpenRouter ランタイム ID を使用してルーティングします。OpenRouter 固有の
    構成の詳細については、
    [OpenRouter プロバイダーのドキュメント](/ja-JP/providers/openrouter)を参照してください。
  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="OpenRouter" href="/ja-JP/providers/openrouter" icon="shuffle">
    1 つの API キーで Arcee モデルやその他多数のモデルにアクセスできます。
  </Card>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作を選択します。
  </Card>
</CardGroup>
