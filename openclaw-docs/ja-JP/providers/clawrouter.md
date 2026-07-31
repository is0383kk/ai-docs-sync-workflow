---
read_when:
    - 複数のモデルプロバイダーに対して、1つの管理されたキーを使用したい場合
    - OpenClaw で ClawRouter のモデル検出またはクォータレポートが必要です
summary: 認証情報のスコープに応じたモデルを ClawRouter 経由でルーティングし、管理対象のクォータを表示する
title: ClawRouter
x-i18n:
    generated_at: "2026-07-26T10:16:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 929a93e8d1d003e21f792d0fdab9542553ffab374f59d4d0505819b0f719591f
    source_path: providers/clawrouter.md
    workflow: 16
---

ClawRouter は、複数の上流モデルプロバイダーに対して、ポリシーでスコープされた単一のキーを OpenClaw に提供します。バンドル済みの `clawrouter` Plugin は、そのキーに許可されたモデルのみを検出し、各モデルを宣言されたプロトコル経由でルーティングして、キーの予算と集計使用量を OpenClaw の使用量画面に報告します。

上流の認証情報とプロバイダー固有の転送処理は ClawRouter 内に保持されるため、OpenClaw ホスト上で上流プロバイダーごとの Plugin をインストールしたり認証したりする必要はありません。この Plugin は OpenClaw にバンドルされています（`enabledByDefault: true`）。必要なのは、発行済みの ClawRouter 認証情報だけです。

| プロパティ      | 値                                    |
| ------------- | ---------------------------------------- |
| プロバイダー      | `clawrouter`                             |
| Plugin        | バンドル済み（OpenClaw に同梱）           |
| 認証          | `CLAWROUTER_API_KEY`                     |
| デフォルト URL   | `https://clawrouter.openclaw.ai`         |
| モデルカタログ | `/v1/catalog` によって認証情報ごとにスコープ設定      |
| クォータ        | `/v1/usage` による月間予算と使用量 |

## はじめに

<Steps>
  <Step title="スコープされた認証情報を取得する">
    使用すべきプロバイダー、モデル、月間予算がポリシーに含まれる認証情報を ClawRouter 管理者に依頼します。認証情報は発行時に一度だけ表示されます。
  </Step>
  <Step title="OpenClaw を設定する">
    ```bash
    export CLAWROUTER_API_KEY="..."
    openclaw onboard --auth-choice clawrouter-api-key
    openclaw plugins enable clawrouter
    ```

    `clawrouter` はバンドル済みで、デフォルトで有効です。設定で `plugins.allow` を指定している場合は、有効化する前にそのリストへ `clawrouter` を追加します。カスタムデプロイでは、`models.providers.clawrouter.baseUrl` を ClawRouter のオリジンに設定します。デフォルトは `https://clawrouter.openclaw.ai` です。

  </Step>
  <Step title="許可されたモデルを一覧表示する">
    ```bash
    openclaw models list --all --provider clawrouter
    ```

    返されたモデル参照を表示どおり正確に使用します。これらは、`clawrouter/openai/gpt-5.5`、`clawrouter/anthropic/claude-sonnet-4-6`、`clawrouter/google/gemini-3.5-flash` など、上流の名前空間を保持します。`agents.defaults.modelPolicy.allow` が設定されている場合は、選択した各 ClawRouter 参照をそこへ追加します。

  </Step>
  <Step title="モデルを選択する">
    ```bash
    openclaw models set clawrouter/<provider>/<model>
    ```

    `openclaw agent --model clawrouter/<provider>/<model> --message "..."` を使用して、返されたモデルを 1 回の実行に対して選択することもできます。

  </Step>
</Steps>

## 管理された非対話型デプロイ

プロキシキーはワークロードのシークレット注入に保持し、`openclaw.json` には SecretRef のみを保存します。標準の管理対象フィールドは次のとおりです。

| 用途       | 設定または環境フィールド                                              |
| ------------- | ------------------------------------------------------------------------ |
| ルーターのオリジン | `models.providers.clawrouter.baseUrl`                                    |
| 認証情報    | `models.providers.clawrouter.apiKey` -> 環境変数 SecretRef                    |
| シークレット値  | Gateway プロセス環境内の `CLAWROUTER_API_KEY`                  |
| デフォルトモデル | `agents.defaults.model.primary` -> `clawrouter/<provider>/<model>`       |
| ワークロードタグ  | `models.providers.clawrouter.headers.X-ClawRouter-Project-Id`（任意） |

たとえば、デプロイコントローラーで次の JSON5 パッチを管理できます。

```json5
{
  plugins: {
    entries: { clawrouter: { enabled: true } },
  },
  models: {
    providers: {
      clawrouter: {
        baseUrl: "https://clawrouter.internal.example",
        apiKey: {
          source: "env",
          provider: "default",
          id: "CLAWROUTER_API_KEY",
        },
        headers: {
          "X-ClawRouter-Project-Id": "fakeco",
        },
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "clawrouter/openai/gpt-5.5" },
    },
  },
}
```

デプロイで `plugins.allow` を設定する場合は、既存のエントリを保持したまま `clawrouter` を追加します。対話型ウィザードを使用せずに検証して適用します。

```bash
openclaw config patch --file ./clawrouter.patch.json5 --dry-run --json
openclaw config patch --file ./clawrouter.patch.json5
```

ドライランでは SecretRef を解決しますが、その値は一切出力しません。認証情報をローテーションするには、`CLAWROUTER_API_KEY` を供給する外部 Secret を更新し、Gateway ワークロードを再起動して新しいプロセス環境を読み込みます。設定ファイルとモデル参照は変更されません。

ソースからビルドしたスタンドアロン Docker Gateway では、ClawRouter はすでにルートランタイムに含まれています。`OPENCLAW_EXTENSIONS=clickclack`、`slack`、`msteams` など、個別のパッケージ化が必要なチャンネル Plugin のみを選択してください。[選択した Plugin を含むソースビルドイメージ](/ja-JP/install/docker#source-built-images-with-selected-plugins)を参照してください。
アーカイブ／アプライアンス形式のデプロイでは、OCI イメージを利用するのではなく、同じ取り込み済みソースを独自のアーティファクトパイプラインでパッケージ化する必要があります。

## 準備完了状態とライブ検証

次のチェックはそれぞれ異なる境界を検証します。相互に代用しないでください。

```bash
# ClawRouter プロセスのヘルスのみ。認証情報や上流モデルは使用されません。
curl -fsS https://clawrouter.internal.example/v1/health

# OpenClaw Gateway の起動準備完了状態のみ。モデル呼び出しは行われません。
curl -fsS http://127.0.0.1:18789/readyz

# 認証情報でスコープされたカタログの検出。
openclaw models list --all --provider clawrouter --json

# 設定済み ClawRouter プロバイダーを経由する最小限の実推論プローブ。
openclaw models status --probe --probe-provider clawrouter --probe-max-tokens 8 --json

# 許可された正確なモデル参照を使用するワークロードカナリア。
openclaw agent --agent main \
  --model clawrouter/openai/gpt-5.5 \
  --message "正確に次のように応答してください: CLAWROUTER_CANARY_OK" \
  --json
```

例のモデルをそのままコピーせず、スコープされたカタログから返されたモデルを使用してください。`/readyz` の応答が成功した場合、Gateway がリクエストを処理できることを意味しますが、ClawRouter、その認証情報、または上流プロバイダーの準備が完了していることを示すものではありません。推論の検証になるのは、モデルプローブとエージェントカナリアです。

ライブ診断では、カナリアを実行し、Gateway の標準ログを確認します。既存のメタデータのみのモデル転送診断では、次の形式の行が出力されます。

```text
[model-fetch] 開始 provider=clawrouter api=openai-responses model=openai/gpt-5.5 method=POST url=https://clawrouter.internal.example/v1/responses
[model-fetch] 応答 provider=clawrouter api=openai-responses model=openai/gpt-5.5 status=200
```

この Plugin は、該当する識別子が利用可能な場合、長さを制限した `X-ClawRouter-Client`、`X-ClawRouter-Agent-Id`、`X-ClawRouter-Session-Id` ヘッダーを送信します。また、モデル呼び出しの診断用 `callId`（`<run-id>:model:<n>`）を `X-Request-ID` にマッピングするため、OpenClaw のモデル呼び出しイベントを ClawRouter のメタデータのみの監査証跡と関連付けられます。128 文字のリクエスト ID 上限内の値は同一です。これを超える値では `:model:<n>` サフィックスと決定論的ハッシュが保持されるため、異なる呼び出しを上限内に収めたまま関連付けられます。`X-ClawRouter-Project-Id` などの静的デプロイメタデータは、プロバイダーの `headers` マップで設定できます。
エージェントとセッションの属性ヘッダーには、それぞれ個別の 256 文字制限が引き続き適用されます。ClawRouter の ASCII 識別子セットに含まれない文字を持つ自動リクエスト ID にも、同じ決定論的な制限形式が使用されます。
`X-Request-ID` の大文字小文字違いを含む、明示的に設定されたヘッダーは、自動値より優先されます。転送診断にはルーティングと応答のメタデータが記録されますが、認証情報、リクエスト ID、プロンプト、生成結果は記録されません。ClawRouter 独自の監査イベントでは、選択された上流プロバイダーとコンテンツ保持状態が提供されます。

## モデル検出

`GET /v1/catalog` は `{ providers: [...] }` を返します。各プロバイダーエントリには、独自の `models[]`（上流 ID、機能、料金を含む）と、サポートされるリクエストルートが一覧表示されます。OpenClaw は ClawRouter モデルの固定リストを別途同梱しません。カタログモデルが OpenClaw モデルとして提示される条件は次のとおりです。

- 認証情報のポリシーでそのプロバイダーが許可されている。
- カタログモデルが、サポート対象の LLM 機能（`llm.responses`、`llm.chat`、`llm.messages`、または対応するストリーミングルートを持つ `llm.stream`）を提示している。
- プロバイダーが、以下の転送方式のいずれかに対応するルートを公開している。

サポートされている ClawRouter プロバイダーにモデルを追加しても、OpenClaw のリリースは不要です。次回のカタログ更新（認証情報のスコープごとに 60 秒間キャッシュ）で検出されます。新しいワイヤープロトコルを必要とするモデルには、先に Plugin の対応が必要です。

## プロトコルとプロバイダー Plugin

ClawRouter が上流の認証情報を管理し、そのカタログが使用する転送方式を OpenClaw に通知するため、上流企業ごとの認証 Plugin をすべてインストールする必要はありません。

| カタログの機能／ルート                               | OpenClaw の転送方式     |
| -------------------------------------------------------- | ---------------------- |
| `llm.responses`（OpenAI 互換プロバイダー）             | `openai-responses`     |
| `llm.chat`（OpenAI 互換プロバイダー）                  | `openai-completions`   |
| `llm.messages` + `anthropic.messages` ルート              | `anthropic-messages`   |
| `llm.stream` + ストリーミング `google.generate_content` ルート | `google-generative-ai` |

この Plugin は、これらのファミリーに対応する再生ポリシーとツールスキーマポリシーも適用します（OpenAI／DeepSeek／Gemini／Perplexity のツールスキーマ互換性、およびネイティブ Anthropic と Google Gemini の再生ポリシー）。Perplexity モデルには厳格なスキーマ書き換えが適用されます。Perplexity はこれらを含まないツールスキーマを拒否するため、`patternProperties` と `additionalProperties` が削除され、すべてのオブジェクトスキーマで `properties` が宣言されます。サポートされていないリクエスト形式しか公開しないカタログプロバイダーは、意図的に OpenClaw のテキストモデルとして提示されません。互換性のないペイロードを送信するのではなく、それらのプロバイダーを ClawRouter 内でサポート対象の契約のいずれかに正規化してください。

## クォータと使用量

ClawRouter の `/v1/usage` 応答は、通常の OpenClaw プロバイダー使用量画面に反映されます。これには、リクエスト数、トークン数、支出額の合計に加え、キーに上限がある場合は月間予算期間が含まれます。従量制限のないキーでも、割合期間なしで集計使用量が表示されます。

クォータ検索では、モデル検出と同じスコープ付きキーを使用します。クォータ検索に失敗しても、モデルの実行は妨げられません。

ライブスナップショットは次のコマンドで確認します。

```bash
openclaw status --usage
openclaw models status
```

同じプロバイダースナップショットは、チャット内の `/status` と OpenClaw の使用量 UI でも利用できます。予算はポリシー全体に適用されるため、同じ ClawRouter ポリシーを使用する別のクライアントからのリクエストによって、残りの割合が変化する可能性があります。

## トラブルシューティング

| 症状                                  | 確認事項                                                                                                                                          |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| ClawRouter モデルがない                     | Plugin が有効で `plugins.allow` によって許可されていることを確認し、認証情報が有効で、準備完了済みのプロバイダーが少なくとも 1 つ許可されていることを確認します。 |
| 設定済みの ClawRouter モデルが見つからない | その `/v1/catalog` 機能とルートの対応状況を確認します。サポートされていない転送契約は意図的に除外されます。                            |
| モデルの上書きがポリシーによって拒否される        | 正確なカタログ参照または `clawrouter/*` を `agents.defaults.modelPolicy.allow` に追加します。                                                            |
| カタログまたは使用量からの `401` または `403`     | ClawRouter 認証情報を再発行するか、スコープを再設定します。OpenClaw は上流プロバイダーのキーへフォールバックしません。                                          |
| 検出後にモデル呼び出しが失敗する         | ClawRouter 内のプロバイダー接続と上流のヘルスを確認し、準備完了状態が回復してから再試行します。                                |
| 使用量に合計はあるが割合がない       | ポリシーは従量制限なしです。割合期間を表示するには、ClawRouter で月間予算を追加します。                                                     |

## セキュリティ動作

- カタログ検出の範囲は、設定されたプロキシキーに限定され、認証情報のスコープ（エージェントディレクトリ、ワークスペースディレクトリ、認証プロファイル ID、ベース URL）ごとにキャッシュされます。
- プロキシキーはリクエストのディスパッチ時にのみ付加され、モデルのメタデータには保存されません。
- 自動アトリビューション値とリクエスト相関値は、ディスパッチ前に前後の空白が除去され、制御文字が含まれている場合は拒否されます。アトリビューション値は 256 文字、リクエスト ID は 128 文字に制限されます。
- モデル転送の診断情報にはメタデータのみが含まれ、プロキシキーやモデルのコンテンツは一切含まれません。
- ネイティブの Anthropic および Gemini モデル ID は、ディスパッチ時にのみアップストリームの ID に書き換えられます。
- サポートされていない、または許可されていないカタログ行はフェイルクローズとなり、選択できません。

## 関連情報

<CardGroup cols={2}>
  <Card title="モデルプロバイダー" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダーの設定とモデルの選択。
  </Card>
  <Card title="使用状況の追跡" href="/ja-JP/concepts/usage-tracking" icon="chart-line">
    OpenClaw の使用状況とステータスの表示。
  </Card>
</CardGroup>
