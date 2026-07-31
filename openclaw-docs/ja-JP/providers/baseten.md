---
read_when:
    - OpenClaw で Thinking Machines Lab の Inkling を実行する場合
    - Baseten のホスト型モデル向けに、OpenAI 互換 API を 1 つに統一したい場合
summary: Inkling およびホスト型モデル API 向けの Baseten セットアップ
title: Baseten
x-i18n:
    generated_at: "2026-07-26T10:27:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2ccc3b5cf64b01859f9f022d7bc15a69a1cb42c87d4f914c118276c1151020de
    source_path: providers/baseten.md
    workflow: 16
---

[Baseten Model API](https://docs.baseten.co/inference/model-apis/overview) は、フロンティアモデルへのホスト型の OpenAI 互換アクセスを提供します。公式の外部 Plugin は認証済みディスカバリーを使用するため、OpenClaw は Baseten アカウントで有効になっている完全なモデルセットに追従します。オフラインフォールバックには、この OpenClaw リリースのビルド時に利用可能だったすべての Model API が含まれています。

| プロパティ      | 値                                                       |
| --------------- | -------------------------------------------------------- |
| プロバイダー ID | `baseten`                                       |
| Plugin          | 公式外部パッケージ (`@openclaw/baseten-provider`)                 |
| 認証環境変数    | `BASETEN_API_KEY`                                       |
| オンボーディングフラグ | `--auth-choice baseten-api-key`                               |
| 直接指定 CLI フラグ | `--baseten-api-key <key>`                                   |
| API             | OpenAI 互換 (`openai-completions`)                        |
| ベース URL      | `https://inference.baseten.co/v1`                                       |
| デフォルトモデル | `baseten/thinkingmachines/inkling`                                      |

## Plugin のインストール

```bash
openclaw plugins install @openclaw/baseten-provider
openclaw gateway restart
```

## はじめに

<Steps>
  <Step title="Baseten アカウントと API キーを作成する">
    Baseten の Basic プランには月額プラットフォーム料金がなく、Model API 呼び出しは使用量に応じて課金されます。[Baseten API key settings](https://app.baseten.co/settings/api_keys) でキーを作成し、[料金ページ](https://www.baseten.co/pricing)で現在の料金を確認してください。
  </Step>
  <Step title="オンボーディングを実行する">
    <CodeGroup>

```bash Onboarding
openclaw onboard --auth-choice baseten-api-key
```

```bash Direct flag
openclaw onboard --non-interactive \
  --auth-choice baseten-api-key \
  --baseten-api-key "$BASETEN_API_KEY"
```

```bash Env only
export BASETEN_API_KEY=...
```

    </CodeGroup>

  </Step>
  <Step title="ライブカタログを確認する">
    ```bash
    openclaw models list --provider baseten
    ```

    使用可能な認証情報がある場合、Plugin は `GET /v1/models` をリクエストし、アカウントについて返されたすべてのモデルを一覧表示します。認証情報がない場合はオフラインのまま、バンドルされたフォールバックを使用します。

  </Step>
</Steps>

## Inkling

[Thinking Machines Lab の Inkling](https://thinkingmachines.ai/news/introducing-inkling/) がデフォルトモデルです。OpenClaw では、テキストと画像の入力、ツール呼び出し、構造化ツールスキーマ、設定可能な推論エフォート、1.048M トークンのコンテキストウィンドウ、最大 32k 出力トークンをサポートします。

```json5
{
  agents: {
    defaults: {
      model: { primary: "baseten/thinkingmachines/inkling" },
    },
  },
}
```

既存のチャットを切り替えるには `/model baseten/thinkingmachines/inkling` を使用します。

## バンドルされたフォールバックカタログ

認証済みライブカタログが正式な情報源です。以下の行により、ディスカバリーが成功する前でもセットアップとモデル選択を利用できます。

| モデル参照                                         | 入力          | コンテキスト | 最大出力 |
| -------------------------------------------------- | ------------- | ------------: | -------: |
| `baseten/deepseek-ai/DeepSeek-V4-Pro`              | テキスト      |    262k |       262k |
| `baseten/zai-org/GLM-4.7`                          | テキスト      |    200k |       200k |
| `baseten/zai-org/GLM-5`                            | テキスト      |    202k |       202k |
| `baseten/zai-org/GLM-5.1`                          | テキスト      |    202k |       202k |
| `baseten/zai-org/GLM-5.2`                          | テキスト      |    202k |       202k |
| `baseten/thinkingmachines/inkling`                 | テキスト、画像 |  1.048M |        32k |
| `baseten/moonshotai/Kimi-K2.5`                     | テキスト、画像 |    262k |       262k |
| `baseten/moonshotai/Kimi-K2.6`                     | テキスト、画像 |    262k |       262k |
| `baseten/moonshotai/Kimi-K2.7-Code`                | テキスト、画像 |    262k |       262k |
| `baseten/nvidia/Nemotron-120B-A12B`                | テキスト      |    202k |       202k |
| `baseten/nvidia/NVIDIA-Nemotron-3-Ultra-550B-A55B` | テキスト      |    202k |       202k |
| `baseten/openai/gpt-oss-120b`                      | テキスト      |    128k |       128k |

バンドルされたすべてのモデルは、ツール呼び出しと推論をサポートします。OpenClaw は、その思考レベルをネイティブの `reasoning_effort` を持つモデルにマッピングします。Baseten のオプトイン式 GLM、Kimi、Nemotron モデルはデフォルトで思考がオフになっています。大部分はオフ／オンの二値制御を公開しますが、GLM 5.2 はオフ、高、最大を公開します。OpenClaw はこれらの選択を Baseten の `chat_template_args.enable_thinking` 制御を通じて送信し、GLM 5.2 については検証済みのトップレベル `reasoning_effort` パラメーターも使用します。

<Note>
Baseten は OpenClaw のリリースとは独立して Model API を追加、削除、または変更できます。Plugin は、モデル固有の OpenClaw トランスポートポリシーを維持しながら、認証済み API からモデル ID、コンテキスト制限、出力制限、入力、キャッシュ済み入力、出力の料金を更新します。
</Note>

## 手動設定

ほとんどのセットアップでは API キーだけが必要です。プロバイダーを明示的に固定するには、次のように設定します。

```json5
{
  env: { BASETEN_API_KEY: "..." },
  agents: {
    defaults: {
      model: { primary: "baseten/thinkingmachines/inkling" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      baseten: {
        baseUrl: "https://inference.baseten.co/v1",
        apiKey: "${BASETEN_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "thinkingmachines/inkling",
            name: "Inkling",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<Note>
Gateway がデーモン（launchd、systemd、Docker）として実行される場合は、そのプロセスから `BASETEN_API_KEY` を利用できることを確認してください。対話型シェルでのみエクスポートされたキーは、すでに実行中の管理対象サービスからは参照できません。
</Note>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルプロバイダー" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作を選択します。
  </Card>
  <Card title="思考モード" href="/ja-JP/tools/thinking" icon="brain">
    OpenClaw の推論エフォートレベルを選択します。
  </Card>
  <Card title="モデル CLI" href="/ja-JP/cli/models" icon="terminal">
    検出されたモデルを一覧表示、調査、選択します。
  </Card>
  <Card title="モデル FAQ" href="/ja-JP/help/faq-models" icon="circle-question">
    認証プロファイルとモデル選択のトラブルシューティングです。
  </Card>
</CardGroup>
