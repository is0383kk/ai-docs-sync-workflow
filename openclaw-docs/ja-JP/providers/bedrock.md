---
read_when:
    - OpenClaw で Amazon Bedrock モデルを使用する場合
    - モデル呼び出しには AWS の認証情報とリージョンの設定が必要です
summary: OpenClaw で Amazon Bedrock（Converse API）モデルを使用する
title: Amazon Bedrock
x-i18n:
    generated_at: "2026-07-26T09:14:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9cbc9534c0d06e0d5642b8d167c633c16880908812b97adbbf9c6bd6c5511603
    source_path: providers/bedrock.md
    workflow: 16
---

OpenClaw は、**Bedrock Converse** ストリーミングプロバイダー経由で
**Amazon Bedrock** モデルを使用できます。Bedrock の認証には API キーではなく、
**AWS SDK のデフォルト認証情報チェーン**を使用します。

| プロパティ | 値                                                       |
| -------- | ----------------------------------------------------------- |
| プロバイダー | `amazon-bedrock`                                            |
| API      | `bedrock-converse-stream`                                   |
| 認証     | AWS 認証情報（環境変数、共有設定、またはインスタンスロール） |
| リージョン   | `AWS_REGION` または `AWS_DEFAULT_REGION`（デフォルト: `us-east-1`） |

## はじめに

希望する認証方法を選択し、セットアップ手順に従います。

<Tabs>
  <Tab title="アクセスキー / 環境変数">
    **最適な用途:** 開発者マシン、CI、または AWS 認証情報を直接管理するホスト。

    <Steps>
      <Step title="Gateway ホストに AWS 認証情報を設定する">
        ```bash
        export AWS_ACCESS_KEY_ID="EXAMPLE_AWS_ACCESS_KEY_ID"
        export AWS_SECRET_ACCESS_KEY="..."
        export AWS_REGION="us-east-1"
        # 任意:
        export AWS_SESSION_TOKEN="..."
        export AWS_PROFILE="your-profile"
        # 任意（Bedrock API キー / ベアラートークン）:
        export AWS_BEARER_TOKEN_BEDROCK="..."
        ```
      </Step>
      <Step title="設定に Bedrock プロバイダーとモデルを追加する">
        `apiKey` は不要です。`auth: "aws-sdk"` を使用してプロバイダーを設定します。

        ```json5
        {
          models: {
            providers: {
              "amazon-bedrock": {
                baseUrl: "https://bedrock-runtime.us-east-1.amazonaws.com",
                api: "bedrock-converse-stream",
                auth: "aws-sdk",
                models: [
                  {
                    id: "us.anthropic.claude-opus-4-6-v1",
                    name: "Claude Opus 4.6 (Bedrock)",
                    reasoning: true,
                    input: ["text", "image"],
                    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                    contextWindow: 200000,
                    maxTokens: 8192,
                  },
                ],
              },
            },
          },
          agents: {
            defaults: {
              model: { primary: "amazon-bedrock/us.anthropic.claude-opus-4-6-v1" },
            },
          },
        }
        ```
      </Step>
      <Step title="モデルが利用可能であることを確認する">
        ```bash
        openclaw models list
        ```
      </Step>
    </Steps>

    <Tip>
    環境マーカー認証（`AWS_ACCESS_KEY_ID`、`AWS_PROFILE`、または `AWS_BEARER_TOKEN_BEDROCK`）では、OpenClaw は追加設定なしでモデル検出用の暗黙的な Bedrock プロバイダーを自動的に有効にします。
    </Tip>

  </Tab>

  <Tab title="EC2 インスタンスロール（IMDS）">
    **最適な用途:** IAM ロールがアタッチされ、認証にインスタンスメタデータサービスを使用する EC2 インスタンス。

    <Steps>
      <Step title="検出を明示的に有効にする">
        IMDS を使用する場合、OpenClaw は環境マーカーだけでは AWS 認証を検出できないため、明示的に有効化する必要があります。

        ```bash
        openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
        openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1
        ```
      </Step>
      <Step title="必要に応じて自動モード用の環境マーカーを追加する">
        環境マーカーによる自動検出パスも機能させたい場合（たとえば、`openclaw status` サーフェス向け）は、次のように設定します。

        ```bash
        export AWS_PROFILE=default
        export AWS_REGION=us-east-1
        ```

        偽の API キーは**不要**です。
      </Step>
      <Step title="モデルが検出されることを確認する">
        ```bash
        openclaw models list
        ```
      </Step>
    </Steps>

    <Warning>
    EC2 インスタンスにアタッチされた IAM ロールには、次の権限が必要です。

    - `bedrock:InvokeModel`
    - `bedrock:InvokeModelWithResponseStream`
    - `bedrock:ListFoundationModels`（自動検出用）
    - `bedrock:ListInferenceProfiles`（推論プロファイル検出用）

    または、マネージドポリシー `AmazonBedrockFullAccess` をアタッチします。
    </Warning>

    <Note>
    自動モードまたはステータスサーフェス用の環境マーカーが特に必要な場合にのみ、`AWS_PROFILE=default` が必要です。実際の Bedrock ランタイム認証パスでは AWS SDK のデフォルトチェーンを使用するため、環境マーカーがなくても IMDS インスタンスロール認証は機能します。
    </Note>

  </Tab>
</Tabs>

## モデルの自動検出

OpenClaw は、**ストリーミング**と**テキスト出力**をサポートする Bedrock モデルを
自動的に検出できます。検出には `bedrock:ListFoundationModels` と
`bedrock:ListInferenceProfiles` を使用し、結果はキャッシュされます（デフォルト: 1 時間）。

暗黙的なプロバイダーが有効になる仕組み:

- `plugins.entries.amazon-bedrock.config.discovery.enabled` が `true` の場合、
  AWS 環境マーカーが存在しなくても OpenClaw は検出を試みます。
- `plugins.entries.amazon-bedrock.config.discovery.enabled` が未設定の場合、
  OpenClaw は次のいずれかの AWS 認証マーカーを検出したときにのみ、
  暗黙的な Bedrock プロバイダーを自動追加します:
  `AWS_BEARER_TOKEN_BEDROCK`、`AWS_ACCESS_KEY_ID` +
  `AWS_SECRET_ACCESS_KEY`、または `AWS_PROFILE`。
- 実際の Bedrock ランタイム認証パスでは引き続き AWS SDK のデフォルトチェーンを使用するため、
  検出を有効化するために `enabled: true` が必要だった場合でも、
  共有設定、SSO、IMDS インスタンスロール認証は機能します。

<Note>
明示的な `models.providers["amazon-bedrock"]` エントリの場合、OpenClaw はランタイム認証全体を強制的に読み込まずに、`AWS_BEARER_TOKEN_BEDROCK` などの AWS 環境マーカーから Bedrock の環境マーカー認証を早期に解決できます。実際のモデル呼び出しの認証パスでは、引き続き AWS SDK のデフォルトチェーンを使用します。
</Note>

<AccordionGroup>
  <Accordion title="検出設定オプション">
    設定オプションは `plugins.entries.amazon-bedrock.config.discovery` の下にあります。

    ```json5
    {
      plugins: {
        entries: {
          "amazon-bedrock": {
            config: {
              discovery: {
                enabled: true,
                region: "us-east-1",
                providerFilter: ["anthropic", "amazon"],
                refreshInterval: 3600,
                defaultContextWindow: 32000,
                defaultMaxTokens: 4096,
              },
            },
          },
        },
      },
    }
    ```

    | オプション | デフォルト | 説明 |
    | ------ | ------- | ----------- |
    | `enabled` | 自動 | 自動モードでは、OpenClaw はサポートされている AWS 環境マーカーを検出した場合にのみ、暗黙的な Bedrock プロバイダーを有効にします。検出を強制するには `true` を設定します。 |
    | `region` | `AWS_REGION` / `AWS_DEFAULT_REGION` / `us-east-1` | 検出 API 呼び出しに使用する AWS リージョン。 |
    | `providerFilter` | （すべて） | Bedrock プロバイダー名（たとえば `anthropic`、`amazon`）に一致します。 |
    | `refreshInterval` | `3600` | キャッシュ期間（秒）。キャッシュを無効にするには `0` に設定します。 |
    | `defaultContextWindow` | `32000` | トークン制限が不明な検出済みモデルに使用するコンテキストウィンドウ（モデルの制限が分かっている場合は上書きしてください）。 |
    | `defaultMaxTokens` | `4096` | トークン制限が不明な検出済みモデルに使用する最大出力トークン数（モデルの制限が分かっている場合は上書きしてください）。 |

  </Accordion>

  <Accordion title="コンテキストウィンドウと最大トークン制限">
    Bedrock の `ListFoundationModels` および `GetFoundationModel` API は、
    トークン制限のメタデータを返さず、モデル ID、名前、モダリティ、ライフサイクル
    ステータスのみを返します。OpenClaw には、主要な Bedrock モデル（Claude、Nova、Llama、Mistral、DeepSeek
    など）の既知のコンテキストウィンドウと出力制限のルックアップテーブルが付属しており、
    これらのモデルでセッション管理、Compaction のしきい値、
    コンテキストオーバーフロー検出が正しく機能します。

    テーブルにない検出済みモデルでは、`defaultContextWindow`
    および `defaultMaxTokens` にフォールバックします。使用するモデルに正確な制限が
    登録されていない場合は、明示的な
    `models.providers["amazon-bedrock"].models` エントリで上書きします。

  </Accordion>
</AccordionGroup>

## クイックセットアップ（AWS パス）

この手順では、IAM ロールを作成し、Bedrock 権限をアタッチし、
インスタンスプロファイルを関連付け、EC2 ホストで OpenClaw の検出を有効にします。

```bash
# 1. IAM ロールとインスタンスプロファイルを作成
aws iam create-role --role-name EC2-Bedrock-Access \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy --role-name EC2-Bedrock-Access \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess

aws iam create-instance-profile --instance-profile-name EC2-Bedrock-Access
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-Bedrock-Access \
  --role-name EC2-Bedrock-Access

# 2. EC2 インスタンスにアタッチ
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxx \
  --iam-instance-profile Name=EC2-Bedrock-Access

# 3. EC2 インスタンス上で検出を明示的に有効化
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# 4. 任意: 明示的に有効化せずに自動モードを使用する場合は環境マーカーを追加
echo 'export AWS_PROFILE=default' >> ~/.bashrc
echo 'export AWS_REGION=us-east-1' >> ~/.bashrc
source ~/.bashrc

# 5. モデルが検出されることを確認
openclaw models list
```

## 高度な設定

<AccordionGroup>
  <Accordion title="推論プロファイル">
    OpenClaw は、基盤モデルとともに**リージョナルおよびグローバル推論プロファイル**を
    検出します。プロファイルが既知の基盤モデルにマッピングされている場合、
    そのプロファイルはモデルの機能（コンテキストウィンドウ、最大トークン数、
    推論、ビジョン）を継承し、正しい Bedrock リクエストリージョンが
    自動的に注入されます。これにより、クロスリージョン Claude プロファイルは
    プロバイダーを手動で上書きしなくても機能します。グローバルなクロスリージョンプロファイル（`global.*`）は、
    一般に容量と自動フェイルオーバーの面で優れているため、
    `openclaw models list` では最初に表示されます。

    推論プロファイル ID は `us.anthropic.claude-opus-4-6-v1`（リージョナル）
    または `anthropic.claude-opus-4-6-v1`（グローバル）のような形式です。基盤モデルがすでに
    検出結果に含まれている場合、プロファイルはその完全な機能セットを継承します。
    それ以外の場合は、安全なデフォルトが適用されます。

    追加の設定は必要ありません。検出が有効で、IAM
    プリンシパルに `bedrock:ListInferenceProfiles` が付与されていれば、プロファイルは
    `openclaw models list` で基盤モデルとともに表示されます。

  </Accordion>

  <Accordion title="サービスティア">
    一部の Bedrock モデルは、コストまたはレイテンシを最適化するための
    `service_tier` パラメーターをサポートしています。次のティアを利用できます。

    | ティア | 説明 |
    |------|-------------|
    | `default` | Bedrock の標準ティア |
    | `flex` | より長いレイテンシを許容できるワークロード向けの割引処理 |
    | `priority` | レイテンシ重視のワークロード向けの優先処理 |
    | `reserved` | 定常状態のワークロード向けの予約容量 |

    Bedrock モデルリクエストでは、`agents.defaults.params` を介して
    `serviceTier`（または `service_tier`）を設定するか、
    `agents.defaults.models["<model-key>"].params` でモデルごとに設定します。

    ```json5
    {
      agents: {
        defaults: {
          params: {
            serviceTier: "flex", // すべてのモデルに適用
          },
          models: {
            "amazon-bedrock/mistral.mistral-large-3-675b-instruct": {
              params: {
                serviceTier: "priority", // モデルごとの上書き
              },
            },
          },
        },
      },
    }
    ```

    有効な値は `default`、`flex`、`priority`、`reserved` です。Claude
    Fable 5、Opus 5、Sonnet 5 は `default` ティアのみをサポートします。これらのモデルに対して
    `flex`、`priority`、または `reserved` が要求された場合、OpenClaw は警告を表示して
    無視します。その他のモデルでは、すべてのモデルがすべてのティアをサポートしているわけではありません。サポートされていないティアを
    指定すると Bedrock の検証エラーが返され、そのエラーメッセージは
    誤解を招く場合があります（たとえば、問題がティアにあることを示さずに「The provided model identifier is invalid」
    と表示されます）。このエラーが表示された場合は、要求したティアを
    モデルがサポートしているか確認してください。

  </Accordion>

  <Accordion title="Claude Opus 5、4.8、4.7 の temperature">
    Bedrock は Claude Opus 5、Opus 4.8、
    Opus 4.7 に対する `temperature` パラメーターを拒否します。OpenClaw は、一致するすべての Bedrock
    参照について `temperature` を自動的に省略します。これには、基盤モデル ID、名前付き推論プロファイル、
    基になるモデルが `bedrock:GetInferenceProfile` を介して Opus 5/4.8/4.7 に解決されるアプリケーション
    推論プロファイル、および省略可能なリージョンプレフィックスを持つドット区切りの
    `opus-4.7`/`opus-4.8` バリアント
    （`us.`、`eu.`、`ap.`、`apac.`、`au.`、`jp.`、
    `global.`）が含まれます。設定項目は不要で、この省略は
    リクエストオプションオブジェクトと `inferenceConfig` ペイロードフィールドの両方に適用されます。
  </Accordion>

  <Accordion title="Claude Opus 5">
    Messages API の Bedrock
    エンドポイントでは `amazon-bedrock/anthropic.claude-opus-5` を使用します。または、Bedrock の検出結果に表示される場合は
    `global.anthropic.claude-opus-5` などのリージョン／グローバル推論プロファイルを使用します。
    OpenClaw は、1,000,000 トークンのコンテキストウィンドウ、128,000 トークンの出力
    上限、画像入力、プロンプトキャッシュ、拒否時にも安全なストリーミング、ネイティブの
    `xhigh`/`max` エフォートレベルを適用します。

    適応型思考のデフォルトは `high` です。`/think off` は思考を無効にし、
    `/think xhigh|max` は適応型思考を有効なままにします。OpenClaw は、カスタム
    サンプリングパラメーターと、サポートされていないデフォルト以外のサービスティアを省略します。

  </Accordion>

  <Accordion title="Claude Fable 5">
    `us-east-1` では `amazon-bedrock/anthropic.claude-fable-5` を使用するか、
    `us.anthropic.claude-fable-5` などのリージョン推論 ID を使用します。
    OpenClaw は、Fable の 1M コンテキストウィンドウ、128K の出力上限、常時有効な
    適応型思考、およびサポートされるエフォートのマッピングを適用します。`/think off` と
    `/think minimal` は `low` にマッピングされます。Opus 4.7/4.8 ルートと同様に、temperature とツール選択を強制する制御は
    省略されます。ストリーミング出力は、Bedrock が終了ステータスを返すまで
    保留されるため、ストリーム途中の拒否によって部分的なテキストが
    表示されることはありません。

    Fable を利用するには、AWS で `provider_data_share` データ保持への明示的なオプトインが
    必要です。プロンプトと補完は Anthropic と共有され、
    信頼性と安全性のために最大 30 日間保持されます。モデルを有効にする前に、
    [Bedrock のデータ保持](https://docs.aws.amazon.com/bedrock/latest/userguide/data-retention.html)
    を確認して設定してください。

  </Accordion>

  <Accordion title="Claude Mythos 5">
    Claude Mythos 5 は、必要な制限付きアクセスの承認を受けたアカウントでのみ
    Bedrock を通じて利用できます。OpenClaw は、基盤モデル
    `anthropic.claude-mythos-5` と、`us.anthropic.claude-mythos-5` などのリージョンまたはグローバル推論プロファイルを
    認識します。

    OpenClaw は、1,000,000 トークンのコンテキストウィンドウ、128,000 トークンの出力
    上限、画像入力、プロンプトキャッシュ、拒否時にも安全なストリーミング、ネイティブの
    エフォートレベルを適用します。適応型思考は常に有効です。`/think off` と
    `/think minimal` は `low` にマッピングされ、`xhigh` と `max` は引き続き利用できます。
    カスタムサンプリング値とツール選択を強制する値は省略されます。

  </Accordion>

  <Accordion title="Claude Sonnet 5">
    AWS は Sonnet 5 について、
    [`bedrock-runtime` と `bedrock-mantle` の両方のエンドポイント](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5.html)
    を文書化しています。
    OpenClaw は、Bedrock の基盤モデル
    `anthropic.claude-sonnet-5` と、`us.anthropic.claude-sonnet-5` などのリージョンまたはグローバル推論プロファイルを
    認識します。1,000,000 トークンのコンテキスト
    ウィンドウ、128,000 トークンの出力上限、画像入力、ネイティブのエフォートレベル、
    プロンプトキャッシュ、拒否時にも安全なストリーミングを適用します。

    Bedrock では Sonnet 5 の適応型思考が有効なままになります。OpenClaw のデフォルトは
    `high` です。このルートでは思考を無効にできないため、`/think off` と `/think minimal` は `low` にマッピングされます。
    適応型思考が有効な間は、カスタム temperature 値とツール選択を強制する値が
    省略されます。

  </Accordion>

  <Accordion title="ガードレール">
    `amazon-bedrock` Plugin の設定に `guardrail` オブジェクトを追加すると、
    すべての Bedrock モデル呼び出しに [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
    を適用できます。ガードレールを使用すると、コンテンツフィルタリング、
    トピック拒否、単語フィルター、機密情報フィルター、コンテキストに基づく
    グラウンディングチェックを適用できます。

    ```json5
    {
      plugins: {
        entries: {
          "amazon-bedrock": {
            config: {
              guardrail: {
                guardrailIdentifier: "abc123", // ガードレール ID または完全な ARN
                guardrailVersion: "1", // バージョン番号または "DRAFT"
                streamProcessingMode: "sync", // 省略可能: "sync" または "async"
                trace: "enabled", // 省略可能: "enabled"、"disabled"、または "enabled_full"
              },
            },
          },
        },
      },
    }
    ```

    `guardrailIdentifier` と `guardrailVersion` は必須です。

    | オプション | 説明 |
    | ------ | ----------- |
    | `guardrailIdentifier` | ガードレール ID（例: `abc123`）または完全な ARN（例: `arn:aws:bedrock:us-east-1:123456789012:guardrail/abc123`）。 |
    | `guardrailVersion` | 公開済みのバージョン番号、または作業中のドラフトを表す `"DRAFT"`。 |
    | `streamProcessingMode` | ストリーミング中のガードレール評価に使用する `"sync"` または `"async"`。省略した場合、Bedrock のデフォルトが使用されます。 |
    | `trace` | デバッグ用の `"enabled"` または `"enabled_full"`。本番環境では省略するか、`"disabled"` に設定します。 |

    <Warning>
    Gateway が使用する IAM プリンシパルには、標準の呼び出し権限に加えて `bedrock:ApplyGuardrail` 権限が必要です。
    </Warning>

  </Accordion>

  <Accordion title="メモリ検索用の埋め込み">
    Bedrock は、
    [メモリ検索](/ja-JP/concepts/memory-search)の埋め込みプロバイダーとしても使用できます。これは
    推論プロバイダーとは別に設定します。`memory.search.provider` を `"bedrock"` に設定してください。

    ```json5
    {
      memory: {
        search: {
          provider: "bedrock",
          model: "amazon.titan-embed-text-v2:0", // デフォルト
        },
      },
    }
    ```

    Bedrock の埋め込みでは、推論と同じ AWS SDK 認証情報チェーン（インスタンス
    ロール、SSO、アクセスキー、共有設定、ウェブアイデンティティ）が使用されます。API キーは
    必要ありません。

    サポートされる埋め込みモデルには、Amazon Titan Embed（v1、v2）、Amazon Nova
    Embed、Cohere Embed（v3、v4）、TwelveLabs Marengo が含まれます。モデルの完全な一覧と次元オプションについては、
    [メモリ設定リファレンス -- Bedrock](/ja-JP/reference/memory-config#bedrock-embedding-config)
    を参照してください。

  </Accordion>

  <Accordion title="注意事項と留意点">
    - Bedrock を使用するには、AWS アカウント／リージョンで **モデルアクセス** を有効にする必要があります。
    - 自動検出には、`bedrock:ListFoundationModels` および
      `bedrock:ListInferenceProfiles` 権限が必要です。
    - 自動モードを利用する場合は、Gateway ホストに、サポートされている AWS 認証環境マーカーのいずれかを設定してください。
      環境マーカーなしで IMDS／共有設定認証を使用する場合は、
      `plugins.entries.amazon-bedrock.config.discovery.enabled: true` を設定してください。
    - OpenClaw は認証情報のソースを次の順序で提示します。`AWS_BEARER_TOKEN_BEDROCK`、
      次に `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`、次に `AWS_PROFILE`、最後に
      デフォルトの AWS SDK チェーンです。
    - 推論のサポート状況はモデルによって異なります。現在の機能については Bedrock のモデルカードを
      確認してください。
    - 管理されたキーフローを使用する場合は、Bedrock の前段に OpenAI 互換
      プロキシを配置し、代わりに OpenAI プロバイダーとして設定することもできます。
  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="メモリ検索" href="/ja-JP/concepts/memory-search" icon="magnifying-glass">
    メモリ検索設定用の Bedrock 埋め込み。
  </Card>
  <Card title="メモリ設定リファレンス" href="/ja-JP/reference/memory-config#bedrock-embedding-config" icon="database">
    Bedrock 埋め込みモデルの完全な一覧と次元オプション。
  </Card>
  <Card title="トラブルシューティング" href="/ja-JP/help/troubleshooting" icon="wrench">
    一般的なトラブルシューティングと FAQ。
  </Card>
</CardGroup>
