---
read_when:
    - OpenClaw で Vydra メディア生成を使用する場合
    - Vydra API キーの設定ガイドが必要です
summary: OpenClaw で Vydra の画像、動画、音声を使用する
title: Vydra
x-i18n:
    generated_at: "2026-07-26T09:43:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cc3856c2dd740e87d70d7eedefd9eae7905ab547aa0d68a1c479a305c59b2982
    source_path: providers/vydra.md
    workflow: 16
---

バンドルされたVydra Pluginは次の機能を追加します。

- `vydra/grok-imagine`による画像生成
- `vydra/veo3`（テキストから動画）および`vydra/kling`（画像から動画）による動画生成
- VydraのElevenLabsベースのTTSルートによる音声合成

OpenClawは3つの機能すべてで同じ`VYDRA_API_KEY`を使用します。

| プロパティ        | 値                                                                     |
| --------------- | ------------------------------------------------------------------------- |
| プロバイダーID     | `vydra`                                                                   |
| Plugin          | バンドル済み、`enabledByDefault: true`                                         |
| 認証環境変数    | `VYDRA_API_KEY`                                                           |
| オンボーディングフラグ | `--auth-choice vydra-api-key`                                             |
| 直接指定するCLIフラグ | `--vydra-api-key <key>`                                                   |
| コントラクト       | `imageGenerationProviders`、`videoGenerationProviders`、`speechProviders` |
| ベースURL        | `https://www.vydra.ai/api/v1`（`www`ホストを使用）                        |

<Warning>
ベースURLには`https://www.vydra.ai/api/v1`を使用してください。Vydraのapexホスト（`https://vydra.ai/api/v1`）は現在`www`にリダイレクトされます。一部のHTTPクライアントは、このホスト間リダイレクト時に`Authorization`を破棄するため、有効なAPIキーでも誤解を招く認証エラーになります。これを回避するため、バンドルされたPluginは、設定された`vydra.ai`のベースURLをすべて`www.vydra.ai`に正規化します。
</Warning>

## セットアップ

<Steps>
  <Step title="対話形式のオンボーディングを実行">
    ```bash
    openclaw onboard --auth-choice vydra-api-key
    ```

    または、環境変数を直接設定します。

    ```bash
    export VYDRA_API_KEY="vydra_live_..."
    ```

  </Step>
  <Step title="デフォルト機能を選択">
    以下の機能（画像、動画、音声）から1つ以上を選び、対応する設定を適用します。
  </Step>
</Steps>

## 機能

<AccordionGroup>
  <Accordion title="画像生成">
    デフォルトかつ唯一のバンドル済み画像モデル：

    - `vydra/grok-imagine`

    デフォルトの画像プロバイダーとして設定します。

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "vydra/grok-imagine",
          },
        },
      },
    }
    ```

    バンドルされたサポートはテキストから画像への生成のみに対応し、1リクエストにつき最大1枚です。Vydraのホスト型編集ルートではリモート画像URLが必要であり、バンドルされたPluginはVydra固有のアップロードブリッジを追加しません。

    <Note>
    共通ツールパラメーター、プロバイダーの選択、フェイルオーバー動作については、[画像生成](/ja-JP/tools/image-generation)を参照してください。
    </Note>

  </Accordion>

  <Accordion title="動画生成">
    登録済みの動画モデル：

    - `vydra/veo3`：テキストから動画（画像参照入力は拒否）
    - `vydra/kling`：画像から動画（リモート画像URLが1つだけ必要）

    Vydraをデフォルトの動画プロバイダーとして設定します。

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "vydra/veo3",
          },
        },
      },
    }
    ```

    注意事項：

    - `vydra/kling`はローカルファイルのアップロードを事前に拒否します。使用できるのはリモート画像URL参照のみです。
    - Vydraの`kling` HTTPルートでは、`image_url`と`video_url`のどちらが必要かについて一貫性がありません。バンドルされたプロバイダーは、両方のフィールドに同じリモート画像URLを送信します。
    - バンドルされたPluginは保守的な動作を維持し、アスペクト比、解像度、ウォーターマーク、生成音声など、文書化されていないスタイル調整項目を転送しません。

    <Note>
    共通ツールパラメーター、プロバイダーの選択、フェイルオーバー動作については、[動画生成](/ja-JP/tools/video-generation)を参照してください。
    </Note>

  </Accordion>

  <Accordion title="動画のライブテスト">
    プロバイダー固有のライブテスト範囲：

    ```bash
    OPENCLAW_LIVE_TEST=1 \
    OPENCLAW_LIVE_VYDRA_VIDEO=1 \
    pnpm test:live -- extensions/vydra/vydra.live.test.ts
    ```

    バンドルされたVydraのライブテストファイルは、次を対象とします。

    - `vydra/veo3`によるテキストから動画への生成
    - リモート画像URLを使用する`vydra/kling`による画像から動画への生成

    必要に応じてリモート画像のフィクスチャを上書きします。

    ```bash
    export OPENCLAW_LIVE_VYDRA_KLING_IMAGE_URL="https://example.com/reference.png"
    ```

  </Accordion>

  <Accordion title="音声合成">
    Vydraを音声プロバイダーとして設定します。

    ```json5
    {
      tts: {
        provider: "vydra",
        providers: {
          vydra: {
            apiKey: "${VYDRA_API_KEY}",
            voiceId: "21m00Tcm4TlvDq8ikWAM",
          },
        },
      },
    }
    ```

    デフォルト：

    - モデル：`elevenlabs/tts`
    - 音声ID：`21m00Tcm4TlvDq8ikWAM`（「Rachel」）

    バンドルされたPluginは、正常動作が確認されているこのデフォルト音声1つを公開し、MP3音声ファイルを返します。

  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="プロバイダーディレクトリ" href="/ja-JP/providers/index" icon="list">
    利用可能なすべてのプロバイダーを確認します。
  </Card>
  <Card title="画像生成" href="/ja-JP/tools/image-generation" icon="image">
    共通の画像ツールパラメーターとプロバイダーの選択。
  </Card>
  <Card title="動画生成" href="/ja-JP/tools/video-generation" icon="video">
    共通の動画ツールパラメーターとプロバイダーの選択。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/config-agents#agent-defaults" icon="gear">
    エージェントのデフォルト値とモデル設定。
  </Card>
</CardGroup>
