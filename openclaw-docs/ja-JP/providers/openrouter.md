---
read_when:
    - 多くの LLM に単一の API キーを使用したい場合
    - OpenClaw で OpenRouter 経由でモデルを実行する場合
    - 画像生成に OpenRouter を使用したい場合
    - 音楽生成に OpenRouter を使用する場合
    - 動画生成に OpenRouter を使用したい場合
summary: OpenRouter の統合 API を使用して、OpenClaw で多数のモデルにアクセスする
title: OpenRouter
x-i18n:
    generated_at: "2026-07-26T09:17:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0936a10222f44f376dee081b7ee0678cddc3bc4579ac0006321dc1012d59bcf
    source_path: providers/openrouter.md
    workflow: 16
---

OpenRouter は、1 つの API と 1 つのキーで多数のモデルにリクエストをルーティングします。
OpenAI 互換であるため、OpenClaw は他のプロキシプロバイダーで使用するものと同じ
`openai-completions` 形式のトランスポートを介して通信します。

## はじめに

<Tabs>
  <Tab title="OAuth">
    <Steps>
      <Step title="OAuth オンボーディングを実行">
        ```bash
        openclaw onboard --auth-choice openrouter-oauth
        ```

        OpenClaw は OpenRouter のブラウザーサインインフロー（PKCE）を開き、コードを
        OpenRouter API キーと交換して、デフォルトの OpenRouter 認証プロファイルに
        保存します。リモートまたはヘッドレスホストでは、OpenClaw がサインイン URL を
        表示し、サインイン後にリダイレクト URL を貼り付けるよう求めます。
      </Step>
      <Step title="（任意）特定のモデルに切り替える">
        オンボーディングのデフォルトは `openrouter/auto` です。後から具体的なモデルを選択できます。

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
  <Tab title="API キー">
    <Steps>
      <Step title="API キーを取得">
        [openrouter.ai/keys](https://openrouter.ai/keys) で API キーを作成します。
      </Step>
      <Step title="API キーのオンボーディングを実行">
        ```bash
        openclaw onboard --auth-choice openrouter-api-key
        ```
      </Step>
      <Step title="（任意）特定のモデルに切り替える">
        オンボーディングのデフォルトは `openrouter/auto` です。後から具体的なモデルを選択できます。

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## 設定例

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/auto" },
    },
  },
}
```

## モデル参照

<Note>
モデル参照は `openrouter/<provider>/<model>` のパターンに従います。利用可能な
プロバイダーとモデルの完全な一覧については、[/concepts/model-providers](/ja-JP/concepts/model-providers) を参照してください。
</Note>

ライブカタログの検出を利用できない場合に使用される組み込みのフォールバックモデル：

| モデル参照                         | 備考                        |
| --------------------------------- | ---------------------------- |
| `openrouter/auto`                 | OpenRouter の自動ルーティング |
| `openrouter/moonshotai/kimi-k2.6` | MoonshotAI 経由の Kimi K2.6     |
| `openrouter/moonshotai/kimi-k2.5` | MoonshotAI 経由の Kimi K2.5     |

`openrouter/openrouter/fusion`（[Fusion ルーター](#fusion-router)を参照）を含む、その他の
`openrouter/<provider>/<model>` 参照は、OpenRouter のライブモデルカタログに対して
動的に解決されます。

## 画像生成

OpenRouter は `image_generate` ツールのバックエンドとして使用できます。
`agents.defaults.mediaModels.image` 配下に OpenRouter の画像モデルを設定します。

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openrouter/google/gemini-3.1-flash-image-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

OpenClaw は `modalities: ["image", "text"]` を指定して、OpenRouter の chat-completions 画像 API に
画像リクエストを送信します。Gemini 画像モデルには、OpenRouter の `image_config` を介して
`aspectRatio` および `resolution` のヒントも渡されますが、他の
画像モデルには渡されません。低速なモデルには `agents.defaults.mediaModels.image.timeoutMs` を使用します。
ただし、`image_generate` ツールの呼び出しごとの `timeoutMs` が引き続き優先されます。

## 動画生成

OpenRouter は、非同期の `/videos` API を介して
`video_generate` ツールのバックエンドとして使用できます。
`agents.defaults.mediaModels.video` 配下に OpenRouter の動画モデルを設定します。

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openrouter/google/veo-3.1-fast",
      },
    },
  },
}
```

OpenClaw はテキストから動画および画像から動画へのジョブを送信し、返された
`polling_url` をポーリングして、完成した動画を OpenRouter の
`unsigned_urls` またはジョブコンテンツのエンドポイントからダウンロードします。
参照画像はデフォルトで先頭／末尾フレーム画像として扱われ、`reference_image` と
タグ付けされた画像は代わりに入力参照として送信されます。組み込みの
`google/veo-3.1-fast` デフォルトは、4/6/8 秒の長さ、
`720P`／`1080P` の解像度、および
`16:9`／`9:16` のアスペクト比をサポートします。
動画から動画への変換はサポートされていません。アップストリーム API が受け付けるのは
テキストと画像の参照のみです。

## 音楽生成

OpenRouter は、chat-completions の音声出力を介して
`music_generate` ツールのバックエンドとして使用できます。
`agents.defaults.mediaModels.music` 配下に OpenRouter の音声モデルを設定します。

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "openrouter/google/lyria-3-pro-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

組み込みの OpenRouter 音楽プロバイダーのデフォルトは `google/lyria-3-pro-preview` で、
`google/lyria-3-clip-preview` も公開します。OpenClaw は `modalities:
["text", "audio"]` を送信し、
レスポンスをストリーミングして音声チャンクを収集し、その結果をチャンネル配信用の
生成メディアとして保存します。Lyria モデルは、共有の `music_generate image=...`
パラメーターを介して 1 枚の参照画像を受け付けます。
ストリーミング音声、文字起こしの保持、および派生した SSE イベントエンベロープは、
`agents.defaults.mediaMaxMb` によって上限が設定されます（デフォルトの音声上限は 16 MB）。

## テキスト読み上げ

OpenRouter は、OpenAI 互換の `/audio/speech` エンドポイントを介して
TTS プロバイダーとして機能できます。

```json5
{
  tts: {
    auto: "always",
    provider: "openrouter",
    providers: {
      openrouter: {
        model: "hexgrad/kokoro-82m",
        speakerVoice: "af_alloy",
        responseFormat: "mp3",
      },
    },
  },
}
```

`tts.providers.openrouter.apiKey` を省略すると、TTS は
`models.providers.openrouter.apiKey`、次に `OPENROUTER_API_KEY` へフォールバックします。

## 音声からテキストへの変換（受信音声）

OpenRouter は、共有の `tools.media.audio` パスから STT エンドポイント
（`/audio/transcriptions`）を使用して、受信した音声／オーディオ添付ファイルを
文字起こしできます。これは、受信した音声／オーディオをメディア理解の事前処理へ
転送するすべてのチャンネル Plugin に適用されます。

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "openrouter", model: "openai/whisper-large-v3-turbo" }],
      },
    },
  },
}
```

OpenClaw は、multipart の OpenAI フォームアップロードではなく、`input_audio`
配下に base64 音声を含む JSON（OpenRouter の STT コントラクト）として
OpenRouter STT リクエストを送信します。

## Fusion ルーター

OpenRouter Fusion は、1 つの OpenClaw モデル参照を複数の OpenRouter モデルへ
並列送信し、OpenRouter に回答を評価させ、通常の OpenRouter エンドポイントを介して
1 つの最終レスポンスを返します。アップストリームのモデルスラッグは
`openrouter/fusion` であるため、OpenClaw のモデル参照には OpenClaw の
プロバイダープレフィックスとアップストリームの OpenRouter 名前空間の両方が含まれます。

```bash
openclaw models set openrouter/openrouter/fusion
```

Fusion のパネルと評価モデルは、モデルの `params.extraBody` を介して設定します。
これらのフィールドは OpenRouter の chat-completions リクエスト本文へ直接転送されます。
Fusion は OAuth と API キーのどちらのオンボーディングでも機能します。OAuth を使用する場合は、
以下の `env.OPENROUTER_API_KEY` 行を省略してください。

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/openrouter/fusion" },
      models: {
        "openrouter/openrouter/fusion": {
          params: {
            extraBody: {
              plugins: [
                {
                  id: "fusion",
                  analysis_models: [
                    "google/gemini-3.5-flash",
                    "moonshotai/kimi-k2.6",
                    "deepseek/deepseek-v4-pro",
                  ],
                  model: "google/gemini-3.5-flash",
                },
              ],
            },
          },
        },
      },
    },
  },
}
```

`analysis_models` は並列パネルで、Fusion Plugin 設定内の
`model` は評価モデルです。Fusion を強制する目的で、通常の
エージェント／チャットターンにおいてトップレベルの `tool_choice` を
`"required"` に設定しないでください。OpenClaw のターンには独自の
ツール定義が含まれる場合があり、トップレベルで必須のツール選択を指定すると、
Fusion ルーターではなくそれらのいずれかが選択される可能性があります。
この Fusion Plugin 設定が存在する場合、OpenClaw は設定済みの分析モデルと
評価モデルを列挙したサニタイズ済みのシステムプロンプト注記を追加します。
これにより、エージェントは自身の Fusion パネルに関する質問に回答できます。
その他の `extraBody` フィールドはプロンプトにコピーされません。

Fusion は設計上低速です。OpenRouter はプロンプトを複数の分析モデルへ
ファンアウトし、その後に評価／統合ステップを実行するため、単一モデルへの
直接リクエストよりもレイテンシーが高くなります。意図的に高品質な回答を得る場合や
エスカレーション経路で使用し、レイテンシー重視のデフォルトには使用しないでください。
パネルは小さく保ち、応答を速めるには高速な分析モデルと評価モデルを選択してください。

設定した参照を 1 回限りのローカル呼び出しでテストします。

```bash
openclaw infer model run --local \
  --model openrouter/openrouter/fusion \
  --prompt "Reply with exactly: FUSION_OK" \
  --json
```

## 認証とヘッダー

OpenRouter は API キーの Bearer トークンを使用します。OpenRouter OAuth は
OpenRouter API キーを発行する PKCE ログインフローであるため、OpenClaw は
手動の API キー設定で使用するものと同じ `openrouter:default` API キー認証
プロファイルに結果を保存します。

完全なオンボーディングを再実行せずに、既存のインストール環境でサインインまたは
保存済みキーのローテーションを行うには、次を実行します。

```bash
openclaw models auth login --provider openrouter --method oauth
openclaw models auth login --provider openrouter --method api-key
```

検証済みの OpenRouter リクエスト（`https://openrouter.ai/api/v1`）では、OpenClaw は
OpenRouter のドキュメントに記載されたアプリ帰属ヘッダーを追加します。

| ヘッダー                    | 値                                                                                                  |
| ------------------------- | ------------------------------------------------------------------------------------------------------ |
| `HTTP-Referer`            | `https://openclaw.ai`                                                                                  |
| `X-OpenRouter-Title`      | `OpenClaw`                                                                                             |
| `X-OpenRouter-Categories` | `cli-agent,cloud-agent,programming-app,creative-writing,writing-assistant,general-chat,personal-agent` |

<Warning>
OpenRouter プロバイダーの接続先を別のプロキシまたはベース URL に変更した場合、
OpenClaw は OpenRouter 固有のヘッダーや Anthropic キャッシュマーカーを
挿入**しません**。
</Warning>

## 高度な設定

<AccordionGroup>
  <Accordion title="レスポンスキャッシュ">
    OpenRouter のレスポンスキャッシュはオプトインです。モデルごとに有効化します。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/auto": {
              params: {
                responseCache: true,
                responseCacheTtlSeconds: 300,
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw は `X-OpenRouter-Cache: true` を送信し、設定されている場合は
    `X-OpenRouter-Cache-TTL` も送信します。`responseCacheClear: true` は現在の
    リクエストに対して強制的に更新を行い、置換後のレスポンスを保存します。
    snake_case のエイリアス（`response_cache`、`response_cache_ttl_seconds`、
    `response_cache_clear`）に加えて、`Seconds` サフィックスなしの
    `responseCacheTtl`／`response_cache_ttl` も受け付けます。

    これはプロバイダーのプロンプトキャッシュおよび OpenRouter の Anthropic
    `cache_control` マーカーとは別の機能です。カスタムプロキシのベース URL ではなく、
    検証済みの `openrouter.ai` ルートにのみ適用されます。

  </Accordion>

  <Accordion title="Anthropic キャッシュマーカー">
    検証済みの OpenRouter ルートでは、Anthropic モデル参照はシステム／開発者
    プロンプトブロックでプロンプトキャッシュをより効率的に再利用できるよう、
    OpenRouter の Anthropic `cache_control` マーカーを維持します。
  </Accordion>

  <Accordion title="Anthropic 推論プリフィル">
    検証済みの OpenRouter ルートでは、推論が有効な Anthropic モデル参照について、
    OpenRouter にリクエストが到達する前に末尾のアシスタントプリフィルターンを削除します。
    これにより、推論会話はユーザーターンで終了する必要があるという Anthropic の要件に適合します。
  </Accordion>

  <Accordion title="思考 / 推論の注入">
    サポートされている非 `auto` ルートでは、OpenClaw は選択された思考レベルを
    OpenRouter プロキシの推論ペイロードにマッピングします。`openrouter/auto` およびサポートされていない
    モデルヒントでは、この注入を行いません。古い `openrouter/hunter-alpha` 参照でも
    注入を行いません。これは、その廃止済みルートでは OpenRouter が最終回答のテキストを推論
    フィールドで返す可能性があるためです。
  </Accordion>

  <Accordion title="DeepSeek V4 の推論リプレイ">
    検証済みの OpenRouter ルートでは、`openrouter/deepseek/deepseek-v4-flash` と
    `openrouter/deepseek/deepseek-v4-pro` が、リプレイされたアシスタントターンで欠落している `reasoning_content` を補完し、
    思考とツールの会話を DeepSeek V4 が後続処理に求める形式に維持します。OpenClaw はこれらのルートに対して、
    OpenRouter がサポートする `reasoning.effort` 値を送信します。`xhigh`/`max` は `xhigh` にマッピングされ、
    オフ以外のその他すべてのレベルは `high` にマッピングされます。
  </Accordion>

  <Accordion title="OpenAI 専用のリクエスト整形">
    OpenRouter はプロキシ形式の OpenAI 互換パスを通るため、
    `serviceTier`、Responses の `store`、
    OpenAI の推論互換ペイロード、プロンプトキャッシュのヒントなど、ネイティブ OpenAI 専用のリクエスト整形は転送されません。
  </Accordion>

  <Accordion title="Gemini バックエンドのルート">
    Gemini バックエンドの OpenRouter 参照は、プロキシ Gemini パスに留まります。OpenClaw はそこで
    Gemini の思考署名のサニタイズを維持しますが、ネイティブ Gemini のリプレイ検証や
    ブートストラップの書き換えは有効にしません。
  </Accordion>

  <Accordion title="プロバイダーのルーティングメタデータ">
    OpenRouter は、基盤プロバイダーのルーティング用に `provider` リクエストオブジェクトを
    サポートしています。すべての OpenRouter テキストモデルリクエストに適用するデフォルトポリシーを
    `models.providers.openrouter.params.provider` で設定します。

    ```json5
    {
      models: {
        providers: {
          openrouter: {
            params: {
              provider: {
                sort: "latency",
                require_parameters: true,
                data_collection: "deny",
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw は、そのオブジェクトをリクエストの `provider`
    ペイロードとして OpenRouter に転送します。`sort`、
    `only`、`ignore`、`order`、`allow_fallbacks`、`require_parameters`、
    `data_collection`、`quantizations`、`max_price`、`preferred_max_latency`、
    `preferred_min_throughput`、`zdr`、`enforce_distillable_text` など、OpenRouter のドキュメントに記載された snake_case フィールドを使用してください。

    モデル単位のパラメーターは、プロバイダー全体のルーティングオブジェクトを上書きします。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/anthropic/claude-sonnet-4-6": {
              params: {
                provider: {
                  order: ["anthropic"],
                  allow_fallbacks: false,
                },
              },
            },
          },
        },
      },
    }
    ```

    これは OpenRouter の chat-completions ルートにのみ適用されます。Anthropic、
    Google、OpenAI、またはカスタムプロバイダーの直接ルートでは、OpenRouter のルーティングパラメーターは無視されます。

  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/configuration-reference" icon="gear">
    エージェント、モデル、プロバイダーの完全な設定リファレンス。
  </Card>
</CardGroup>
