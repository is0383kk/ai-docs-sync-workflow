---
read_when:
    - OpenClaw で Qwen を使用する場合
    - Alibaba Cloud Token Plan サブスクリプションを契約していること
summary: OpenClaw Plugin を介して Qwen Cloud を使用する
title: Qwen
x-i18n:
    generated_at: "2026-07-26T09:59:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 74f94a35631dcdf8c9afc12e86d7a9d6b51a359411ba36f8820f8b1e7c03a27a
    source_path: providers/qwen.md
    workflow: 16
---

Qwen Cloud は、正規 id `qwen` を持つ公式の外部 OpenClaw プロバイダー Plugin です。Qwen Cloud / Alibaba DashScope の Standard および Coding Plan エンドポイントを対象とし、Token Plan を `qwen-token-plan` として公開し、`modelstudio` を互換性エイリアスとして維持するほか、Alibaba が文書化している `bailian-token-plan` カスタムプロバイダー id を独立して所有します。

| プロパティ             | 値                                         |
| ---------------------- | ------------------------------------------ |
| プロバイダー           | `qwen`                         |
| Token Plan プロバイダー | `qwen-token-plan`                         |
| 推奨環境変数           | `QWEN_API_KEY`                         |
| Token Plan 環境変数    | `QWEN_TOKEN_PLAN_API_KEY`                         |
| 追加で受け付ける値（互換性） | `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY` |
| API 形式               | OpenAI 互換                                |

<Tip>
`qwen3.7-plus` と `qwen3.6-plus` は、Coding Plan および Standard エンドポイントで動作します。
`qwen3.7-max` または `qwen3.6-flash` には、**Standard（従量課金）**エンドポイントを使用してください。
</Tip>

## Plugin のインストール

`qwen` は、コアに同梱されない公式の外部 Plugin として提供されます。インストールして Gateway を再起動します。

```bash
openclaw plugins install @openclaw/qwen-provider
openclaw gateway restart
```

## はじめに

プラン種別を選択し、セットアップ手順に従ってください。

<Tabs>
  <Tab title="Coding Plan（サブスクリプション）">
    **最適な用途:** Qwen Coding Plan を通じたサブスクリプションベースのアクセス。

    <Steps>
      <Step title="API キーを取得する">
        [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) で API キーを作成するか、コピーします。
      </Step>
      <Step title="オンボーディングを実行する">
        **グローバル**エンドポイントの場合:

        ```bash
        openclaw onboard --auth-choice qwen-api-key
        ```

        **中国**エンドポイントの場合:

        ```bash
        openclaw onboard --auth-choice qwen-api-key-cn
        ```
      </Step>
      <Step title="デフォルトモデルを設定する">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "qwen/qwen3.5-plus" },
            },
          },
        }
        ```
      </Step>
      <Step title="モデルが利用可能であることを確認する">
        ```bash
        openclaw models list --provider qwen
        ```
      </Step>
    </Steps>

    <Note>
    従来の `modelstudio-*` auth-choice id と `modelstudio/...` モデル参照は、
    互換性エイリアスとして引き続き動作しますが、新しいセットアップフローでは正規の
    `qwen-*` auth-choice id と `qwen/...` モデル参照を使用してください。別の `api` 値を持つ
    完全一致のカスタム `models.providers.modelstudio` エントリを定義した場合、その
    カスタムプロバイダーが Qwen 互換性エイリアスの代わりに `modelstudio/...` 参照を
    所有します。
    </Note>

  </Tab>

  <Tab title="Standard（従量課金）">
    **最適な用途:** Coding Plan では利用できない `qwen3.7-max` および `qwen3.6-flash` を含む、Standard Model Studio エンドポイントを通じた従量課金アクセス。

    <Steps>
      <Step title="API キーを取得する">
        [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) で API キーを作成するか、コピーします。
      </Step>
      <Step title="オンボーディングを実行する">
        **グローバル**エンドポイントの場合:

        ```bash
        openclaw onboard --auth-choice qwen-standard-api-key
        ```

        **中国**エンドポイントの場合:

        ```bash
        openclaw onboard --auth-choice qwen-standard-api-key-cn
        ```
      </Step>
      <Step title="デフォルトモデルを設定する">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "qwen/qwen3.5-plus" },
            },
          },
        }
        ```
      </Step>
      <Step title="モデルが利用可能であることを確認する">
        ```bash
        openclaw models list --provider qwen
        ```
      </Step>
    </Steps>

    <Note>
    従来の `modelstudio-*` auth-choice id と `modelstudio/...` モデル参照は、
    互換性エイリアスとして引き続き動作しますが、新しいセットアップフローでは正規の
    `qwen-*` auth-choice id と `qwen/...` モデル参照を使用してください。別の `api` 値を持つ
    完全一致のカスタム `models.providers.modelstudio` エントリを定義した場合、その
    カスタムプロバイダーが Qwen 互換性エイリアスの代わりに `modelstudio/...` 参照を
    所有します。
    </Note>

  </Tab>

  <Tab title="Token Plan（Team Edition）">
    **最適な用途:** Alibaba Cloud Model Studio を通じた、Qwen およびサポート対象のサードパーティモデルへのクレジットベースのチームサブスクリプションアクセス。

    <Steps>
      <Step title="専用キーを取得する">
        Token Plan シートを割り当て、専用の `sk-sp-...` キーを作成します。Token Plan、Coding Plan、従量課金のキーに互換性はありません。[グローバル Token Plan の概要](https://www.alibabacloud.com/help/en/model-studio/token-plan-overview)または[中国 Token Plan の概要](https://help.aliyun.com/zh/model-studio/token-plan-overview)を参照してください。
      </Step>
      <Step title="オンボーディングを実行する">
        シンガポールの**グローバル / 国際**エンドポイントの場合:

        ```bash
        openclaw onboard --auth-choice qwen-token-plan
        ```

        北京の**中国**エンドポイントの場合:

        ```bash
        openclaw onboard --auth-choice qwen-token-plan-cn
        ```
      </Step>
      <Step title="プロバイダーを確認する">
        ```bash
        openclaw models list --provider qwen-token-plan
        openclaw agent --model qwen-token-plan/qwen3.7-plus --message "Token Plan の準備完了と返信してください"
        ```
      </Step>
    </Steps>

    <Note>
    Alibaba の OpenClaw ガイドでは、手動のカスタム
    プロバイダーに `bailian-token-plan` を使用しています。Plugin はその id を互換性のための所有者として登録しますが、新しい
    設定では `qwen-token-plan` を使用してください。完全一致のカスタム
    `models.providers.bailian-token-plan` エントリは、設定された
    トランスポートとカタログの所有権を維持します。正規の OpenAI カタログに統合されることはありません。
    </Note>

    <Warning>
    Token Plan は、対話型の OpenClaw セッションにのみ使用してください。
    Cron ジョブ、無人スクリプト、アプリケーションバックエンドには選択しないでください。Alibaba は、
    非対話型で使用するとサブスクリプションが停止されたり、API キーが取り消されたりする可能性があるとしています。
    </Warning>

  </Tab>

</Tabs>

## プラン種別とエンドポイント

| プラン                     | リージョン | 認証選択肢                 | エンドポイント                                                   |
| -------------------------- | ---------- | -------------------------- | ---------------------------------------------------------------- |
| Coding Plan（サブスクリプション） | 中国       | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`                               |
| Coding Plan（サブスクリプション） | グローバル | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`                          |
| Standard（従量課金）       | 中国       | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`                      |
| Standard（従量課金）       | グローバル | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1`                 |
| Token Plan（Team Edition） | 中国       | `qwen-token-plan-cn`       | `token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`     |
| Token Plan（Team Edition） | グローバル | `qwen-token-plan`          | `token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1` |

プロバイダーは、認証選択肢に基づいてエンドポイントを自動選択します。正規の
選択肢では `qwen-*` ファミリーを使用し、`modelstudio-*` は互換性専用として維持されます。
設定内のカスタム `baseUrl` で上書きできます。

<Tip>
**キーの管理:** [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) |
**ドキュメント:** [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)
</Tip>

## 組み込みカタログ

OpenClaw には、この Qwen 静的カタログが付属しています。カタログはエンドポイントを認識します。Coding
Plan の設定では、Standard エンドポイントでのみ動作するモデルが除外されます。

| モデル参照                  | 入力        | コンテキスト | 備考                    |
| --------------------------- | ----------- | ------------ | ----------------------- |
| `qwen/qwen3.5-plus`         | テキスト、画像 | 1,000,000 | デフォルトモデル        |
| `qwen/qwen3.6-flash`        | テキスト、画像 | 1,000,000 | Standard エンドポイントのみ |
| `qwen/qwen3.6-plus`         | テキスト、画像 | 1,000,000 | Coding Plan + Standard  |
| `qwen/qwen3.7-max`          | テキスト    | 1,000,000 | Standard エンドポイントのみ |
| `qwen/qwen3.7-plus`         | テキスト、画像 | 1,000,000 | Coding Plan + Standard  |
| `qwen/qwen3-max-2026-01-23` | テキスト    | 262,144   | Qwen Max 系列           |
| `qwen/qwen3-coder-next`     | テキスト    | 262,144   | コーディング            |
| `qwen/qwen3-coder-plus`     | テキスト    | 1,000,000 | コーディング            |
| `qwen/MiniMax-M2.5`         | テキスト    | 1,000,000 | 推論有効                |
| `qwen/glm-5`                | テキスト    | 202,752   | GLM                     |
| `qwen/glm-4.7`              | テキスト    | 202,752   | GLM                     |
| `qwen/kimi-k2.5`            | テキスト、画像 | 262,144   | Alibaba 経由の Moonshot AI |

<Note>
モデルが静的カタログに存在する場合でも、利用可否はエンドポイントや課金プランによって
異なることがあります。
</Note>

### Token Plan カタログ

Token Plan は、完全一致文字列の個別の許可リストを使用します。画像生成専用のプラン
モデルは異なる API を使用するため、ここには含まれていません。

| モデル参照                          | 入力        | コンテキスト |
| ----------------------------------- | ----------- | ------------ |
| `qwen-token-plan/qwen3.7-max`       | テキスト    | 1,000,000 |
| `qwen-token-plan/qwen3.7-plus`      | テキスト、画像 | 1,000,000 |
| `qwen-token-plan/qwen3.6-plus`      | テキスト、画像 | 1,000,000 |
| `qwen-token-plan/qwen3.6-flash`     | テキスト、画像 | 1,000,000 |
| `qwen-token-plan/deepseek-v4-pro`   | テキスト    | 1,000,000 |
| `qwen-token-plan/deepseek-v4-flash` | テキスト    | 1,000,000 |
| `qwen-token-plan/deepseek-v3.2`     | テキスト    | 131,072   |
| `qwen-token-plan/kimi-k2.7-code`    | テキスト、画像 | 262,144   |
| `qwen-token-plan/kimi-k2.6`         | テキスト、画像 | 262,144   |
| `qwen-token-plan/kimi-k2.5`         | テキスト、画像 | 262,144   |
| `qwen-token-plan/glm-5.2`           | テキスト    | 1,000,000 |
| `qwen-token-plan/glm-5.1`           | テキスト    | 202,752   |
| `qwen-token-plan/glm-5`             | テキスト    | 202,752   |
| `qwen-token-plan/MiniMax-M2.5`      | テキスト    | 196,608   |

## 思考制御

`qwen3.7-max`、`qwen3.7-plus`、`qwen3.6-flash`、`qwen3.6-plus` は、
組み込みカタログで推論が有効です。`qwen`
ファミリーの推論モデルでは、プロバイダーが OpenClaw の思考レベルを DashScope のトップレベル
`enable_thinking` リクエストフラグにマッピングします。思考を無効にすると `enable_thinking: false` を送信し、
それ以外のレベルでは `enable_thinking: true` を送信します。カスタムモデルでは、モデルエントリに
`compat.thinkingFormat: "qwen-chat-template"` を設定することで、別のチャットテンプレート思考ペイロードを
有効にできます。

Token Plan モデルにも推論対応のマークが付いています。`kimi-k2.7-code` と
`MiniMax-M2.5` は思考専用であるため、セッションが `/think off` を要求しても
OpenClaw は思考を有効なままにします。DeepSeek V4 は `minimal` から `high` を
サービスの `high` effort にマッピングし、`xhigh` または `max` を `max` にマッピングします。GLM 5.2 は
`minimal` から `max` までの全範囲を受け付けます。GLM 5.1 と GLM 5 は
`xhigh` までを受け付け、3 つすべてのデフォルトは `high` です。その他のハイブリッドモデルは、
要求されたオン / オフ状態に従います。

## マルチモーダルアドオン

`qwen` Plugin は、Coding Plan エンドポイントではなく、**Standard** DashScope
エンドポイントでのみマルチモーダル機能を提供します。

- **画像および動画の理解**（`qwen3.6-plus` 経由）
- **Wan 動画生成**（`wan2.6-t2v`〔デフォルト〕、`wan2.6-i2v`、`wan2.6-r2v`、`wan2.6-r2v-flash`、`wan2.7-r2v` 経由）

メディア理解は、設定済みの Qwen 認証から自動的に解決されるため、追加の
設定は不要です。メディア理解を機能させるには、Standard（従量課金）エンドポイントを
使用していることを確認してください。

Qwen をデフォルトの動画プロバイダーにするには:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

動画生成の制限: リクエストあたり出力動画は1本、入力画像は最大1枚
（画像から動画）、入力動画は最大4本（動画から動画）、長さは最大10秒です。
`size`、`aspectRatio`、`resolution`、`audio`、
および`watermark`をサポートします。参照画像/動画の入力にはリモート http(s) URL が必要です。
DashScope の動画エンドポイントはこれらの参照用にアップロードされたローカルバッファを
受け付けないため、ローカルファイルパスは事前に拒否されます。

<Note>
共通ツールパラメータ、プロバイダーの選択、フェイルオーバーの動作については、[動画生成](/ja-JP/tools/video-generation)を参照してください。
</Note>

## 高度な設定

<AccordionGroup>
  <Accordion title="Qwen 3.6 と 3.7 の利用可否">
    `qwen3.7-plus`と`qwen3.6-plus`は、Coding Plan と Standard のエンドポイントで利用できます。`qwen3.7-max`と`qwen3.6-flash`は Standard でのみ利用できます。Standard（従量課金制）のエンドポイントは次のとおりです。

    - 中国: `dashscope.aliyuncs.com/compatible-mode/v1`
    - グローバル: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

    OpenClaw は Coding Plan のカタログから`qwen3.7-max`と`qwen3.6-flash`を除外します。
    Coding Plan のエンドポイントでいずれかに対して「unsupported model」エラーが返された場合は、
    対応する Standard エンドポイントとキーに切り替えてください。

  </Accordion>

  <Accordion title="動画生成のリージョンルーティング">
    OpenClaw は動画ジョブを送信する前に、設定された Qwen リージョンを
    対応する DashScope AIGC ホストにマッピングします。

    - グローバル/国際: `https://dashscope-intl.aliyuncs.com`
    - 中国: `https://dashscope.aliyuncs.com`

    Coding Plan または Standard の Qwen ホストのいずれかを指す通常の
    `models.providers.qwen.baseUrl`でも、動画生成は対応するリージョンの
    DashScope 動画エンドポイントにルーティングされます。

  </Accordion>

  <Accordion title="ストリーミング使用量の互換性">
    ネイティブ Qwen エンドポイントは、共有`openai-completions`トランスポート上で
    ストリーミング使用量の互換性を通知します。そのため、同じネイティブホストを
    対象とする DashScope 互換のカスタムプロバイダー ID は、組み込みの
    `qwen`プロバイダー ID を明示的に使用しなくても、同じ動作を継承します。
    これは Coding Plan、Standard、Token Plan の各エンドポイントに適用されます。

    - `https://coding.dashscope.aliyuncs.com/v1`
    - `https://coding-intl.dashscope.aliyuncs.com/v1`
    - `https://dashscope.aliyuncs.com/compatible-mode/v1`
    - `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
    - `https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1`
    - `https://token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`

  </Accordion>

  <Accordion title="機能計画">
    `qwen` Plugin は、コーディング/テキストモデルだけでなく、Qwen Cloud の
    全機能を扱うベンダー側の基盤として位置づけられています。

    - **テキスト/チャットモデル:** Plugin を通じて利用可能
    - **ツール呼び出し、構造化出力、思考:** OpenAI 互換トランスポートから継承
    - **画像生成:** プロバイダー Plugin レイヤーで対応予定
    - **画像/動画理解:** Standard エンドポイント上の Plugin を通じて利用可能
    - **音声/オーディオ:** プロバイダー Plugin レイヤーで対応予定
    - **メモリの埋め込み/再ランキング:** 埋め込みアダプターのサーフェスを通じて対応予定
    - **動画生成:** 共有動画生成機能を介して Plugin から利用可能

  </Accordion>

  <Accordion title="環境とデーモンのセットアップ">
    Gateway をデーモン（launchd/systemd）として実行する場合は、`QWEN_API_KEY`
    または`QWEN_TOKEN_PLAN_API_KEY`をそのプロセスから利用できることを確認してください
    （たとえば、`~/.openclaw/.env`内、または`env.shellEnv`経由）。
  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="動画生成" href="/ja-JP/tools/video-generation" icon="video">
    共通の動画ツールパラメータとプロバイダーの選択。
  </Card>
  <Card title="Alibaba Model Studio" href="/ja-JP/providers/alibaba" icon="cloud">
    同じ DashScope プラットフォーム上にバンドルされた Wan 動画生成プロバイダー。
  </Card>
  <Card title="トラブルシューティング" href="/ja-JP/help/troubleshooting" icon="wrench">
    一般的なトラブルシューティングと FAQ。
  </Card>
</CardGroup>
