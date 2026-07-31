---
read_when:
    - 送信する返信に Inworld の音声合成を使用する場合
    - Inworld から PCM テレフォニーまたは OGG_OPUS ボイスメモ出力を取得する必要があります
summary: OpenClaw の返信向け Inworld ストリーミング音声合成
title: Inworld
x-i18n:
    generated_at: "2026-07-26T09:58:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09560f5beda3b40d9c67f9408d34446f28ecddb8235fc0725c4265c813302946
    source_path: providers/inworld.md
    workflow: 16
---

Inworld はストリーミング型のテキスト読み上げ（TTS）プロバイダーです。OpenClaw では、送信する返信音声（デフォルトは MP3、ボイスメモには OGG_OPUS）と、Voice Call などの電話チャネル向けの生 PCM 音声を合成します。

OpenClaw は Inworld のストリーミング TTS エンドポイントにリクエストを送信し、返された Base64 音声チャンクを単一のバッファに連結して、その結果を標準の返信音声パイプラインに渡します。

| プロパティ      | 値                                                           |
| ------------- | --------------------------------------------------------------- |
| プロバイダー ID   | `inworld`                                                       |
| Plugin        | 公式外部パッケージ（`@openclaw/inworld-speech`）          |
| コントラクト      | `speechProviders`（TTS のみ）                                    |
| 認証環境変数  | `INWORLD_API_KEY`（HTTP Basic、Base64 ダッシュボード認証情報）     |
| ベース URL      | `https://api.inworld.ai`                                        |
| デフォルト音声 | `Sarah`                                                         |
| デフォルトモデル | `inworld-tts-1.5-max`                                           |
| 出力        | MP3（デフォルト）、OGG_OPUS（ボイスメモ）、PCM 22050 Hz（電話） |
| Web サイト       | [inworld.ai](https://inworld.ai)                                |
| ドキュメント          | [docs.inworld.ai/tts/tts](https://docs.inworld.ai/tts/tts)      |

## Plugin のインストール

```bash
openclaw plugins install @openclaw/inworld-speech
openclaw gateway restart
```

## はじめに

<Steps>
  <Step title="API キーを設定する">
    Inworld ダッシュボード（Workspace > API Keys）から認証情報をコピーし、環境変数として設定します。この値は HTTP Basic 認証情報としてそのまま送信されるため、再度 Base64 エンコードしたり、Bearer トークンに変換したりしないでください。

    ```bash
    INWORLD_API_KEY=<base64-credential-from-dashboard>
    ```

  </Step>
  <Step title="tts で Inworld を選択する">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "inworld",
        providers: {
          inworld: {
            voiceId: "Sarah",
            modelId: "inworld-tts-1.5-max",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="メッセージを送信する">
    接続済みの任意のチャネルから返信を送信します。OpenClaw は Inworld で音声を合成し、MP3（チャネルがボイスメモを想定している場合は OGG_OPUS）として配信します。
  </Step>
</Steps>

## 設定オプション

| オプション        | パス                                | 説明                                                         |
| ------------- | ----------------------------------- | ------------------------------------------------------------------- |
| `apiKey`      | `tts.providers.inworld.apiKey`      | Base64 ダッシュボード認証情報。`INWORLD_API_KEY` にフォールバックします。       |
| `baseUrl`     | `tts.providers.inworld.baseUrl`     | Inworld API のベース URL を上書きします（デフォルトは `https://api.inworld.ai`）。   |
| `voiceId`     | `tts.providers.inworld.voiceId`     | 音声識別子（デフォルトは `Sarah`）。従来のエイリアス: `speakerVoiceId`。 |
| `modelId`     | `tts.providers.inworld.modelId`     | TTS モデル ID（デフォルトは `inworld-tts-1.5-max`）。                       |
| `temperature` | `tts.providers.inworld.temperature` | サンプリング温度。`0`（含まない）から `2` まで（任意）。            |

## 注記

<AccordionGroup>
  <Accordion title="認証">
    Inworld は、単一の Base64 エンコード済み認証情報文字列を使用する HTTP Basic 認証を採用しています。Inworld ダッシュボードからそのままコピーしてください。プロバイダーは追加のエンコードを行わずに `Authorization: Basic <apiKey>` として送信するため、自分で Base64 エンコードしたり、Bearer 形式のトークンを渡したりしないでください。同じ注意事項については、[TTS の認証に関する注記](/ja-JP/tools/tts#inworld-primary)を参照してください。
  </Accordion>
  <Accordion title="モデル">
    サポートされるモデル ID: `inworld-tts-1.5-max`（デフォルト）、`inworld-tts-1.5-mini`、`inworld-tts-1-max`、`inworld-tts-1`。
  </Accordion>
  <Accordion title="音声出力">
    返信ではデフォルトで MP3 を使用します。チャネルのターゲットが `voice-note` の場合、音声がネイティブの音声バブルとして再生されるように、OpenClaw は Inworld に `OGG_OPUS` を要求します。電話用の音声合成では、電話ブリッジに供給するために 22050 Hz の生 `PCM` を使用します。
  </Accordion>
  <Accordion title="カスタムエンドポイント">
    `tts.providers.inworld.baseUrl` で API ホストを上書きします。リクエストの送信前に末尾のスラッシュが削除されます。
  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="テキスト読み上げ" href="/ja-JP/tools/tts" icon="waveform-lines">
    TTS の概要、プロバイダー、および `tts` の設定。
  </Card>
  <Card title="設定" href="/ja-JP/gateway/configuration" icon="gear">
    `tts` の設定を含む完全な設定リファレンス。
  </Card>
  <Card title="プロバイダー" href="/ja-JP/providers" icon="grid">
    OpenClaw がサポートするすべてのプロバイダー。
  </Card>
  <Card title="トラブルシューティング" href="/ja-JP/help/troubleshooting" icon="wrench">
    一般的な問題とデバッグ手順。
  </Card>
</CardGroup>
