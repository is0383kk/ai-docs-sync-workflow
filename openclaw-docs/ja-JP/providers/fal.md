---
read_when:
    - OpenClaw で fal の画像生成を使用する場合
    - FAL_KEY 認証フローが必要です
    - image_generate、video_generate、または music_generate に fal のデフォルト設定を使用する場合
summary: OpenClaw での fal による画像、動画、音楽生成のセットアップ
title: Fal
x-i18n:
    generated_at: "2026-07-26T09:16:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9bd868aaf6771f6fa38bb8e2a83133460d150e2a5aa9e5b888e221c07f29e0ad
    source_path: providers/fal.md
    workflow: 16
---

OpenClaw には、ホスト型の画像、動画、音楽生成用の `fal` プロバイダーがバンドルされています。

| プロパティ | 値                                                                           |
| -------- | ------------------------------------------------------------------------------- |
| プロバイダー | `fal`                                                                           |
| 認証     | `FAL_KEY`（標準。フォールバックとして `FAL_API_KEY` も使用可能）                   |
| API      | fal モデルエンドポイント（`https://fal.run`。動画ジョブでは `https://queue.fal.run` を使用） |
| ベース URL | `models.providers.fal.baseUrl` で上書き                                    |

## はじめに

<Steps>
  <Step title="API キーを設定">
    ```bash
    openclaw onboard --auth-choice fal-api-key
    ```

    非対話型セットアップでは、`--fal-api-key <key>` を渡すか、`FAL_KEY` をエクスポートできます。
    オンボーディングでは、画像モデルが設定されていない場合、
    `fal/fal-ai/flux/dev` もデフォルトの画像モデルとして設定されます。

  </Step>
  <Step title="デフォルトの画像モデルを設定">
    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "fal/fal-ai/flux/dev",
          },
        },
      },
    }
    ```
  </Step>
</Steps>

## 画像生成

バンドルされた `fal` 画像生成プロバイダーのデフォルトは
`fal/fal-ai/flux/dev` です。

| 機能     | 値                                                              |
| -------------- | ------------------------------------------------------------------ |
| 最大画像数     | リクエストあたり 4 枚。Krea 2 はリクエストあたり 1 枚                               |
| サイズの上書き | `1024x1024`、`1024x1536`、`1536x1024`、`1024x1792`、`1792x1024`    |
| アスペクト比   | Flux の画像間変換を除くすべてで対応                    |
| 解像度     | `1K`、`2K`、`4K`（モデルごとの制限は下記を参照）                          |
| 出力形式  | `png`（デフォルト）または `jpeg`。Krea 2 は `outputFormat` の上書きを拒否 |

編集リクエスト（共有の `image` / `images` パラメーターによる参照画像）は、
モデルごとの参照数制限を持つモデル別の編集エンドポイントにルーティングされます。

| モデルファミリー              | `fal/` の後のモデル参照                 | 編集エンドポイント     | 最大参照画像数 |
| ------------------------- | -------------------------------------- | ----------------- | -------------------- |
| Flux およびその他の fal モデル | `fal-ai/flux/dev`（デフォルト）            | `/image-to-image` | 1                    |
| GPT Image                 | `openai/gpt-image-*`                   | `/edit`           | 10                   |
| Grok Imagine              | `xai/grok-imagine-image`               | `/edit`           | 3                    |
| Nano Banana（レガシー）      | `fal-ai/nano-banana`                   | `/edit`           | 3                    |
| Nano Banana 2             | `fal-ai/nano-banana-*`                 | `/edit`           | 14                   |
| Nano Banana 2 Lite        | `google/nano-banana-2-lite`            | `/edit`           | 14                   |
| Krea 2                    | `krea/v2/{medium,large}/text-to-image` | なし（スタイル参照） | 10 件のスタイル参照  |

<Warning>
Flux の画像間変換リクエストは、`aspectRatio` の上書きに対応していません。GPT
Image と Nano Banana 2 の編集リクエストは fal の `/edit` エンドポイントを使用し、
アスペクト比のヒントを受け付けます。Nano Banana 2 はさらに、
`4:1`、`1:4`、`8:1`、`1:8` など、ネイティブ範囲外の横長・縦長比率も受け付けます。Krea 2 は、
独自のより限定的なアスペクト比のサブセットを検証します。Grok Imagine には独自の比率一覧
（`2:1`、`20:9`、`19.5:9` とその逆比を含む）があり、`1K`/`2K` の解像度のみを受け付けます。
レガシー Nano Banana と Nano Banana 2 Lite は、`resolution` の上書きを拒否します。
</Warning>

Krea 2 モデルは、fal ネイティブの Krea ペイロードスキーマを使用します。OpenClaw は、
Flux で使用される汎用の `image_size` / 編集エンドポイント用ペイロードの代わりに、
`aspect_ratio`、`creativity`、`image_style_references` を送信します。モデル参照は次のとおりです。

- `fal/krea/v2/medium/text-to-image`
- `fal/krea/v2/large/text-to-image`

表現力豊かなイラスト、アニメ、絵画、アーティスティックなスタイルを高速に生成するには Medium を使用します。
より低速でも、フォトリアル、自然な質感、フィルムグレイン、細部まで作り込まれた表現を生成するには Large を使用します。
Krea のデフォルトは `fal.creativity: "medium"` で、対応する値は
`raw`、`low`、`medium`、`high` です。

fal のリクエストスキーマでは、Krea 2 は `image_size` ではなくアスペクト比を公開します。
`aspectRatio` の使用を推奨します。OpenClaw は `size` を最も近い対応済みの Krea アスペクト比にマッピングし、
破棄するのではなく、Krea に対する `resolution` を拒否します。

`output_format` を公開している fal モデルから PNG 出力を得るには、`outputFormat: "png"` を使用します。
OpenClaw では、fal は透明背景を明示的に制御する機能を宣言していないため、
fal モデルでは `background: "transparent"` が無視された上書きとして報告されます。
Krea 2 エンドポイントは fal を介して `output_format` リクエストフィールドを公開しないため、
OpenClaw は Krea リクエストに対する `outputFormat` の上書きを拒否します。

Krea 2 Medium を使用するには、次のように設定します。

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/krea/v2/medium/text-to-image",
      },
    },
  },
}
```

## 動画生成

バンドルされた `fal` 動画生成プロバイダーのデフォルトは
`fal/fal-ai/minimax/video-01-live` です。

| 機能 | 値                                                              |
| ---------- | ------------------------------------------------------------------ |
| モード      | テキストから動画、単一画像参照、Seedance の参照から動画 |
| ランタイム    | 長時間実行ジョブ向けのキューベースの送信・ステータス・結果フロー       |
| タイムアウト    | デフォルトでジョブあたり 20 分。5 秒ごとにステータスをポーリング       |

<AccordionGroup>
  <Accordion title="利用可能な動画モデル">
    **MiniMax（デフォルト）：**

    - `fal/fal-ai/minimax/video-01-live`

    **HeyGen video-agent：**

    - `fal/fal-ai/heygen/v2/video-agent`

    **Kling と Wan：**

    - `fal/fal-ai/kling-video/v2.1/master/text-to-video`
    - `fal/fal-ai/wan/v2.2-a14b/text-to-video`
    - `fal/fal-ai/wan/v2.2-a14b/image-to-video`

    **Seedance 2.0：**

    - `fal/bytedance/seedance-2.0/fast/text-to-video`
    - `fal/bytedance/seedance-2.0/fast/image-to-video`
    - `fal/bytedance/seedance-2.0/fast/reference-to-video`
    - `fal/bytedance/seedance-2.0/text-to-video`
    - `fal/bytedance/seedance-2.0/image-to-video`
    - `fal/bytedance/seedance-2.0/reference-to-video`

    MiniMax Live と HeyGen のリクエストは、プロンプトと任意の
    単一参照画像のみを送信し、その他の上書きは転送されません。Seedance モデルは、
    `aspectRatio`、`size`、`resolution`、4～15 秒の長さ、
    および音声の切り替えを受け付けます。

  </Accordion>

  <Accordion title="Seedance 2.0 の設定例">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/bytedance/seedance-2.0/fast/text-to-video",
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="Seedance 2.0 の参照から動画への設定例">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/bytedance/seedance-2.0/fast/reference-to-video",
          },
        },
      },
    }
    ```

    参照から動画では、共有の `video_generate` `images`、`videos`、`audioRefs`
    パラメーターを通じて、最大 9 枚の画像、3 本の動画、3 件の音声参照を受け付けます。
    参照ファイルの合計は最大 12 件です。音声参照を使用するには、
    同じリクエスト内に少なくとも 1 件の画像または動画参照が必要です。

  </Accordion>

  <Accordion title="HeyGen video-agent の設定例">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/fal-ai/heygen/v2/video-agent",
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## 音楽生成

バンドルされた `fal` Plugin は、共有の `music_generate` ツール向けに
音楽生成プロバイダーも登録します。

| 機能    | 値                                                                                                                    |
| ------------- | ------------------------------------------------------------------------------------------------------------------------ |
| デフォルトモデル | `fal/fal-ai/minimax-music/v2.6`                                                                                          |
| モデル        | `fal-ai/minimax-music/v2.6`（mp3）、`fal-ai/ace-step/prompt-to-audio`（wav）、`fal-ai/stable-audio-25/text-to-audio`（wav） |
| 最大長  | 240 秒                                                                                                              |
| ランタイム       | 同期リクエストと生成済み音声のダウンロード                                                                        |

fal をデフォルトの音楽プロバイダーとして使用します。

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "fal/fal-ai/minimax-music/v2.6",
      },
    },
  },
}
```

`fal-ai/minimax-music/v2.6` は明示的な歌詞とインストゥルメンタルモードに対応していますが、
同じリクエストで両方を使用することはできません。ACE-Step と Stable Audio は、
プロンプトから音声を生成するエンドポイントです。これらのモデルファミリーを使用する場合は、
`model` の上書きで選択します。ACE-Step は明示的な歌詞を拒否し、Stable Audio は
歌詞とインストゥルメンタルモードの両方を拒否します。

<Tip>
上記の表とアコーディオンでは、バンドルされた fal プロバイダーが特別に処理するモデルファミリーを説明しています。
その他の fal 画像エンドポイント ID も画像モデルとして選択できます。それらは Flux と同様に扱われます
（汎用の `image_size` ペイロード、`/image-to-image` による 1 枚の参照画像）。
</Tip>

## 関連項目

<CardGroup cols={2}>
  <Card title="画像生成" href="/ja-JP/tools/image-generation" icon="image">
    共有画像ツールのパラメーターとプロバイダーの選択。
  </Card>
  <Card title="動画生成" href="/ja-JP/tools/video-generation" icon="video">
    共有動画ツールのパラメーターとプロバイダーの選択。
  </Card>
  <Card title="音楽生成" href="/ja-JP/tools/music-generation" icon="music">
    共有音楽ツールのパラメーターとプロバイダーの選択。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/config-agents#agent-defaults" icon="gear">
    画像、動画、音楽モデルの選択を含むエージェントのデフォルト設定。
  </Card>
</CardGroup>
