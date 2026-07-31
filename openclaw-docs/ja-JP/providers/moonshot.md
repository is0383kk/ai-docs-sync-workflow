---
read_when:
    - Moonshot Kimi K3/K2（Moonshot Open Platform）と Kimi Coding のセットアップを比較したい場合
    - 個別のエンドポイント、キー、モデル参照を理解する必要があります
    - どちらのプロバイダーでもコピー＆ペーストできる設定が必要な場合
summary: Moonshot Kimi モデルと Kimi Coding を設定する（個別のプロバイダーとキー）
title: Moonshot AI
x-i18n:
    generated_at: "2026-07-26T10:28:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 213379bf88fec26b052184a920e112f0887d6485601bfb47f590cf37ef983e58
    source_path: providers/moonshot.md
    workflow: 16
---

Moonshot は、OpenAI 互換エンドポイントを備えた Kimi API を提供します。Kimi K3 には
`moonshot/kimi-k3` を選択し、オンボーディングのデフォルト
`moonshot/kimi-k2.6` を維持するか、Kimi Coding には `kimi/kimi-for-coding` を使用します。

<Warning>
Moonshot と Kimi Coding は**別々のプロバイダー**であり、それぞれ個別の外部 Plugin として提供されます。キーに互換性はなく、エンドポイントとモデル参照も異なります（`moonshot/...` と `kimi/...`）。
</Warning>

## 組み込みモデルカタログ

[//]: # "moonshot-kimi-k2-ids:start"

| モデル参照                           | 名前                     | 推論       | 入力          | コンテキスト | 最大出力  |
| ----------------------------------- | ------------------------ | ---------- | ----------- | --------- | ---------- |
| `moonshot/kimi-k2.6`                | Kimi K2.6                | なし       | テキスト、画像 | 262,144   | 262,144    |
| `moonshot/kimi-k3`                  | Kimi K3                  | 常に最大   | テキスト、画像 | 1,048,576 | 1,048,576  |
| `moonshot/kimi-k2.7-code`           | Kimi K2.7 Code           | 常に有効   | テキスト、画像 | 262,144   | 262,144    |
| `moonshot/kimi-k2.7-code-highspeed` | Kimi K2.7 Code HighSpeed | 常に有効   | テキスト、画像 | 262,144   | 262,144    |
| `moonshot/kimi-k2.5`                | Kimi K2.5                | なし       | テキスト、画像 | 262,144   | 262,144    |

[//]: # "moonshot-kimi-k2-ids:end"

カタログの推定コストには、Moonshot が公開している従量課金制の料金を使用しています。コストに関する判断を行う前に、[Kimi K3](https://platform.kimi.ai/docs/pricing/chat-k3)、
[Kimi K2.7 Code](https://platform.kimi.ai/docs/pricing/chat-k27-code)、
[Kimi K2.6](https://platform.kimi.ai/docs/pricing/chat-k26)、および
[Kimi K2.5](https://platform.kimi.ai/docs/pricing/chat-k25) の最新のベンダーページを確認してください。

Kimi K3 は常に `reasoning_effort: "max"` で推論します。OpenClaw は
`/think max` のみを公開し、K2 専用の `thinking` フィールドを省略し、K3 がプロバイダーのデフォルトに固定するサンプリングのオーバーライド
（`temperature`、`top_p`、`n`、`presence_penalty`、および
`frequency_penalty`）を削除します。Kimi K2.7 Code も常にネイティブ思考を使用しますが、
`thinking` と `reasoning_effort` の両方を省略する必要があります。HighSpeed バリアントも同じ契約を使用します。
Kimi K2.6 は引き続きオンボーディングのデフォルトです。
Moonshot の [Kimi K3 クイックスタート](https://platform.kimi.ai/docs/guide/kimi-k3-quickstart)を参照してください。

## はじめに

Moonshot と Kimi Coding はどちらも外部 Plugin です。オンボーディングの前に、いずれかをインストールしてください。

<Tabs>
  <Tab title="Moonshot API">
    **最適な用途:** Moonshot Open Platform 経由での Kimi K3 および K2 モデルの利用。

    <Steps>
      <Step title="Plugin をインストールする">
        ```bash
        openclaw plugins install @openclaw/moonshot-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="エンドポイントのリージョンを選択する">
        | 認証の選択肢           | エンドポイント                 | リージョン    |
        | ---------------------- | ------------------------------ | ------------- |
        | `moonshot-api-key`     | `https://api.moonshot.ai/v1`   | 国際          |
        | `moonshot-api-key-cn`  | `https://api.moonshot.cn/v1`   | 中国          |
      </Step>
      <Step title="オンボーディングを実行する">
        ```bash
        openclaw onboard --auth-choice moonshot-api-key
        ```

        中国向けエンドポイントの場合:

        ```bash
        openclaw onboard --auth-choice moonshot-api-key-cn
        ```
      </Step>
      <Step title="Kimi K3 をデフォルトモデルに設定する">
        オンボーディングでは、初期デフォルトとして Kimi K2.6 が維持されます。Kimi K3 を使用する場合は、明示的に切り替えます。

        ```bash
        openclaw models set moonshot/kimi-k3
        ```
      </Step>
      <Step title="モデルが利用可能であることを確認する">
        ```bash
        openclaw models list --provider moonshot
        ```
      </Step>
      <Step title="ライブスモークテストを実行する">
        通常のセッションに影響を与えずにモデルへのアクセスとコスト追跡を確認する場合は、分離された状態ディレクトリを使用します。

        ```bash
        OPENCLAW_CONFIG_PATH=/tmp/openclaw-kimi/openclaw.json \
        OPENCLAW_STATE_DIR=/tmp/openclaw-kimi \
        openclaw agent --local \
          --session-id live-kimi-cost \
          --message 'Reply exactly: KIMI_LIVE_OK' \
          --thinking max \
          --json
        ```

        JSON レスポンスでは、`provider: "moonshot"` と
        `model: "kimi-k3"` が報告される必要があります。Moonshot が使用量メタデータを返す場合、アシスタントのトランスクリプトエントリには、正規化されたトークン使用量と推定コストが `usage.cost` の下に保存されます。
      </Step>
    </Steps>

    ### 設定例

    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: {
            // moonshot-kimi-k2-aliases:start
            "moonshot/kimi-k2.6": { alias: "Kimi K2.6" },
            "moonshot/kimi-k3": { alias: "Kimi K3" },
            "moonshot/kimi-k2.7-code": { alias: "Kimi K2.7 Code" },
            "moonshot/kimi-k2.7-code-highspeed": { alias: "Kimi K2.7 Code HighSpeed" },
            "moonshot/kimi-k2.5": { alias: "Kimi K2.5" },
            // moonshot-kimi-k2-aliases:end
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              // moonshot-kimi-k2-models:start
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k3",
                name: "Kimi K3",
                reasoning: true,
                thinkingLevelMap: {
                  off: null,
                  minimal: null,
                  low: null,
                  medium: null,
                  high: null,
                  xhigh: "max",
                  max: "max",
                },
                input: ["text", "image"],
                cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 0 },
                contextWindow: 1048576,
                maxTokens: 1048576,
              },
              {
                id: "kimi-k2.7-code",
                name: "Kimi K2.7 Code",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.19, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.7-code-highspeed",
                name: "Kimi K2.7 Code HighSpeed",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 1.9, output: 8, cacheRead: 0.38, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.5",
                name: "Kimi K2.5",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.6, output: 3, cacheRead: 0.1, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              // moonshot-kimi-k2-models:end
            ],
          },
        },
      },
    }
    ```

  </Tab>

  <Tab title="Kimi Coding">
    **最適な用途:** Kimi Coding エンドポイントを介したコード重視のタスク。

    <Note>
    Kimi Coding は、Moonshot（`moonshot/...`）とは異なる API キーとプロバイダープレフィックス（`kimi/...`）を使用します。現在の参照は、256K コンテキスト用の `kimi/k3`、1M ティア用の `kimi/k3[1m]`、`kimi/kimi-for-coding`、および `kimi/kimi-for-coding-highspeed` です。従来の参照 `kimi/kimi-code` と `kimi/k2p5` は引き続き受け入れられ、`kimi/kimi-for-coding` に正規化されます。
    </Note>

    コーディングサービスは、OpenAI 互換の
    `https://api.kimi.com/coding/v1` クライアントと Anthropic 互換の
    `https://api.kimi.com/coding/` クライアントの両方を受け入れます。この Plugin は Anthropic Messages を使用します。
    メンバーシップキーは
    [Kimi Code Console](https://www.kimi.com/code/console) で作成します。現在のメンバーシップ料金は [Kimi の料金ページ](https://www.kimi.com/membership/pricing)に掲載されています。

    <Steps>
      <Step title="Plugin をインストールする">
        ```bash
        openclaw plugins install @openclaw/kimi-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="オンボーディングを実行する">
        ```bash
        openclaw onboard --auth-choice kimi-code-api-key
        ```
      </Step>
      <Step title="デフォルトモデルを設定する">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "kimi/kimi-for-coding" },
            },
          },
        }
        ```
      </Step>
      <Step title="モデルが利用可能であることを確認する">
        ```bash
        openclaw models list --provider kimi
        ```
      </Step>
    </Steps>

    Kimi Code K3 は、デフォルトで `max` の深い思考を使用します。`/think off` は
    `thinking.type: "disabled"` を送信し、`/think max` は最大エフォートで K3 の適応的思考リクエストを送信します。古い低い思考レベルは、サポートされている
    `max` レベルに解決されます。1M モデルには Allegretto 以上の Kimi メンバーシップが必要です。Moderato では `kimi/k3` を使用してください。

    現在のプランでの利用可否については、公式の [Kimi Code モデル表](https://www.kimi.com/code/docs/en/kimi-code/models.html)を参照してください。

    ### 設定例

    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: {
            "kimi/kimi-for-coding": { alias: "Kimi" },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

## Kimi ウェブ検索

Moonshot Plugin は、Moonshot ウェブ検索を基盤とする `web_search` プロバイダーとして **Kimi** も登録します。

<Steps>
  <Step title="対話形式のウェブ検索設定を実行する">
    ```bash
    openclaw configure --section web
    ```

    ウェブ検索セクションで **Kimi** を選択し、
    `plugins.entries.moonshot.config.webSearch.*` を保存します。

  </Step>
  <Step title="ウェブ検索のリージョンとモデルを設定する">
    対話形式の設定では、次の項目の入力を求められます。

    | 設定                | オプション                                                           |
    | ------------------- | -------------------------------------------------------------------- |
    | API リージョン      | `https://api.moonshot.ai/v1`（国際）または `https://api.moonshot.cn/v1`（中国） |
    | ウェブ検索モデル    | デフォルトは `kimi-k2.6`                                             |

  </Step>
</Steps>

設定は `plugins.entries.moonshot.config.webSearch` の下にあります。

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // or use KIMI_API_KEY / MOONSHOT_API_KEY
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.6",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

## 高度な設定

<AccordionGroup>
  <Accordion title="ネイティブ思考モード">
    Moonshot API の Kimi K3 は、常に最大エフォートで推論します。OpenClaw は
    `/think max` のみを公開し、`reasoning_effort: "max"` を送信し、古い低い設定または
    `off` の設定を無視します。

    Kimi Code K3 は `/think off|max` を公開します。その Anthropic 互換エンドポイントは、
    オフの場合は `thinking.type: "disabled"` を受け取り、最大の場合は
    `output_config.effort: "max"` による適応的思考を受け取ります。これは `kimi/k3` と
    `kimi/k3[1m]` の両方に適用されます。
    Moonshot API K3 は `auto`、`none`、`required`、および固定されたツール選択をサポートするため、
    OpenClaw は要求された `tool_choice` を保持します。複数ターンのツール使用では、
    OpenClaw は Moonshot のリプレイ契約で必要とされるアシスタントの推論内容を
    保持します。

    Kimi K2.7 Code は常にネイティブ思考を使用します。Moonshot はクライアントに対し、
    このモデルでは `thinking` フィールドを省略するよう求めるため、OpenClaw は `on` のみを公開し、
    古い `off` 設定を無視します。K2.7 では `temperature`、`top_p`、`n`、
    `presence_penalty`、`frequency_penalty` も固定されており、OpenClaw はこれらのフィールドに設定された
    オーバーライドを省略します。

    その他の Moonshot Kimi モデルは、二値のネイティブ思考をサポートします。

    - `thinking: { type: "enabled" }`
    - `thinking: { type: "disabled" }`

    `agents.defaults.models.<provider/model>.params` を使用してモデルごとに設定します。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "disabled" },
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw は、これらのモデルの実行時 `/think` レベルを次のようにマッピングします。

    | `/think` レベル       | Moonshot の動作          |
    | -------------------- | -------------------------- |
    | `/think off`         | `thinking.type=disabled`   |
    | オフ以外の任意のレベル    | `thinking.type=enabled`    |

    <Warning>
    Moonshot K2 の思考が有効な場合、`tool_choice` は `auto` または `none` でなければなりません。固定されたツール選択（`type: "tool"` または `type: "function"`）では、要求されたツールが引き続き実行されるよう、思考は代わりに強制的に `disabled` に戻されます。`tool_choice: "required"` は代わりに `auto` に正規化されます。Kimi K2.7 Code では思考を無効にできないため、互換性のない `tool_choice` は `auto` に正規化されます。Kimi K3 は独自の推論強度契約を使用し、サポートされているツール選択を保持します。
    </Warning>

    Kimi K2.6 は、`reasoning_content` の
    複数ターンにわたる保持を制御するオプションの `thinking.keep` フィールドも受け付けます。ターン間で完全な
    推論を保持するには `"all"` に設定します。サーバーの
    デフォルト戦略を使用するには、省略するか `null` のままにします。OpenClaw は
    `moonshot/kimi-k2.6` に対してのみ `thinking.keep` を転送し、
    その他のモデルからは削除します。Kimi K2.7 Code は
    デフォルトで完全な推論履歴を保持し、OpenClaw は
    `thinking` フィールド全体を省略します。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "enabled", keep: "all" },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="ツール呼び出し ID のサニタイズ">
    Moonshot Kimi は、`functions.<name>:<index>` のような形式のネイティブ tool_call ID を返します。OpenClaw は各ネイティブ Kimi ID の最初の出現を保持し、それ以降の重複を決定論的な OpenAI 形式の `call_*` ID に書き換えます。対応するツール結果も同じ ID に再マッピングされるため、Kimi の最初のネイティブ ID を削除せずに、リプレイの一意性が維持されます。この動作はバンドルされた Moonshot プロバイダーに組み込まれており、ユーザーが設定できる項目ではありません。
  </Accordion>

  <Accordion title="ストリーミング使用量の互換性">
    ネイティブ Moonshot エンドポイント（`https://api.moonshot.ai/v1` および
    `https://api.moonshot.cn/v1`）は、ストリーミング使用量との互換性を明示しています。
    OpenClaw はこれをプロバイダー ID ではなくエンドポイントのホストに基づいて判定するため、同じネイティブ Moonshot ホストを指定した
    カスタムプロバイダー ID は、同じストリーミング使用量の動作を
    引き継ぎます。

    カタログの K2.6 料金設定では、入力、出力、
    キャッシュ読み取りトークンを含むストリーミング使用量も、`/status`、`/usage full`、`/usage cost`、およびトランスクリプトに基づくセッション
    会計用に、ローカルで推定される USD コストへ変換されます。

  </Accordion>

  <Accordion title="エンドポイントとモデル参照のリファレンス">
    | プロバイダー   | モデル参照プレフィックス | エンドポイント                      | 認証環境変数        |
    | ---------- | ---------------- | ------------------------------ | ------------------- |
    | Moonshot   | `moonshot/`      | `https://api.moonshot.ai/v1`  | `MOONSHOT_API_KEY`  |
    | Moonshot CN| `moonshot/`      | `https://api.moonshot.cn/v1`  | `MOONSHOT_API_KEY`  |
    | Kimi Coding| `kimi/`          | Kimi Coding エンドポイント           | `KIMI_API_KEY`      |
    | Web 検索 | 該当なし              | Moonshot API と同じリージョン    | `KIMI_API_KEY` または `MOONSHOT_API_KEY` |

    - Kimi Web 検索は `KIMI_API_KEY` または `MOONSHOT_API_KEY` を使用し、モデル `kimi-k2.6` ではデフォルトで `https://api.moonshot.ai/v1` を使用します。
    - 必要に応じて、`models.providers` で料金とコンテキストのメタデータをオーバーライドします。
    - Moonshot がモデルに対して異なるコンテキスト上限を公開した場合は、それに応じて `contextWindow` を調整します。

  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="Web 検索" href="/ja-JP/tools/web" icon="magnifying-glass">
    Kimi を含む Web 検索プロバイダーの設定。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/configuration-reference" icon="gear">
    プロバイダー、モデル、Plugin の完全な設定スキーマ。
  </Card>
  <Card title="Moonshot Open Platform" href="https://platform.moonshot.ai" icon="globe">
    Moonshot API キーの管理とドキュメント。
  </Card>
</CardGroup>
