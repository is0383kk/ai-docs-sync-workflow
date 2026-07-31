---
read_when:
    - 送信する返信に Azure Speech の音声合成を使用する場合
    - Azure Speech からネイティブの Ogg Opus ボイスメモ出力が必要です
summary: OpenClaw の返信向け Azure AI Speech テキスト読み上げ
title: Azure Speech
x-i18n:
    generated_at: "2026-07-26T10:15:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cfeeb9daa8d7d6aa24e497d57d64e07efa94c3c0c6b16f793343a450286ab3c1
    source_path: providers/azure-speech.md
    workflow: 16
---

Azure Speech は、バンドルされている Azure AI Speech のテキスト読み上げプロバイダーです。OpenClaw は
SSML を使用して Azure Speech REST API を直接呼び出し、標準の返信には MP3、
音声メモにはネイティブの Ogg/Opus、Voice Call などの電話チャネルには
8 kHz mulaw を合成します。リクエストでは、プロバイダーが管理する出力形式を
`X-Microsoft-OutputFormat` ヘッダーで送信します。

| 詳細                    | 値                                                                                                             |
| ----------------------- | -------------------------------------------------------------------------------------------------------------- |
| プロバイダー ID         | `azure-speech`（エイリアス: `azure`）                                                          |
| Web サイト              | [Azure AI Speech](https://azure.microsoft.com/products/ai-services/ai-speech)                                  |
| ドキュメント            | [Speech REST テキスト読み上げ](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech) |
| 認証                    | `AZURE_SPEECH_KEY` と `AZURE_SPEECH_REGION`                                                                      |
| デフォルトの音声        | `en-US-JennyNeural`                                                                                            |
| デフォルトのファイル出力 | `audio-24khz-48kbitrate-mono-mp3`                                                                                           |
| デフォルトの音声メモファイル | `ogg-24khz-16bit-mono-opus`                                                                                       |

## はじめに

<Steps>
  <Step title="Azure Speech リソースを作成する">
    Azure portal で Speech リソースを作成します。Resource Management > Keys and Endpoint から
    **KEY 1** をコピーし、`eastus` などのリソースの場所をコピーします。

    ```
    AZURE_SPEECH_KEY=<speech-resource-key>
    AZURE_SPEECH_REGION=eastus
    ```

  </Step>
  <Step title="tts で Azure Speech を選択する">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "azure-speech",
        providers: {
          "azure-speech": {
            voice: "en-US-JennyNeural",
            lang: "en-US",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="メッセージを送信する">
    接続済みの任意のチャネルから返信を送信します。OpenClaw は Azure Speech で音声を合成し、
    標準音声には MP3 を配信し、チャネルが音声メモを必要とする場合は Ogg/Opus を配信します。
  </Step>
</Steps>

## 設定オプション

すべてのオプションは `tts.providers["azure-speech"]` の下にあります。

| オプション              | 説明                                                                                                  |
| ----------------------- | ----------------------------------------------------------------------------------------------------- |
| `apiKey`      | Azure Speech リソースキー。`AZURE_SPEECH_KEY`、`AZURE_SPEECH_API_KEY`、または `SPEECH_KEY` にフォールバックします。 |
| `region`      | Azure Speech リソースのリージョン。`AZURE_SPEECH_REGION` または `SPEECH_REGION` にフォールバックします。 |
| `endpoint`      | 省略可能な Azure Speech エンドポイントの上書き。信頼済みの `AZURE_SPEECH_ENDPOINT` にフォールバックします。 |
| `baseUrl`      | 省略可能な Azure Speech ベース URL の上書き。                                                          |
| `voice`      | Azure 音声の ShortName（デフォルトは `en-US-JennyNeural`）。レガシーエイリアス: `voiceId`。 |
| `lang`      | SSML 言語コード（デフォルトは `en-US`）。                                                  |
| `outputFormat`      | 音声ファイルの出力形式（デフォルトは `audio-24khz-48kbitrate-mono-mp3`）。                                           |
| `voiceNoteOutputFormat`      | 音声メモの出力形式（デフォルトは `ogg-24khz-16bit-mono-opus`）。                                               |
| `timeoutMs`      | リクエストタイムアウトの上書き（ミリ秒）。グローバルな `tts.timeoutMs` にフォールバックします。     |

`apiKey` に加えて、`region`、`endpoint`、または
`baseUrl` のいずれかが設定されると、プロバイダーは設定済みとみなされます。
環境変数は、未設定のままの設定キーに対するフォールバックとしてのみ確認されます。
ワークスペースの `.env` ファイルでは `AZURE_SPEECH_ENDPOINT` を設定できません。
エンドポイントのルーティングには、プロセス環境、グローバルランタイムの dotenv、
または明示的な設定を使用してください。

## 注記

<AccordionGroup>
  <Accordion title="認証">
    Azure Speech は Azure OpenAI キーではなく、Speech リソースキーを使用します。
    キーは `Ocp-Apim-Subscription-Key` として送信されます。`endpoint` または
    `baseUrl` を指定しない限り、OpenClaw は `region` から
    `https://<region>.tts.speech.microsoft.com` を導出します。
  </Accordion>
  <Accordion title="音声名">
    Azure Speech 音声の `ShortName` 値（例: `en-US-JennyNeural`）を使用します。
    バンドルされているプロバイダーは、同じ Speech リソースを通じて音声を一覧表示でき、
    非推奨、廃止済み、または無効とマークされた音声を除外します。
  </Accordion>
  <Accordion title="音声出力">
    Azure は `audio-24khz-48kbitrate-mono-mp3`、`ogg-24khz-16bit-mono-opus`、`riff-24khz-16bit-mono-pcm` などの
    出力形式を受け付けます。OpenClaw は `voice-note` ターゲットに Ogg/Opus を
    リクエストするため、チャネルは MP3 への追加変換なしでネイティブの音声バブルを送信できます。
    また、電話ターゲットでは `raw-8khz-8bit-mono-mulaw` を強制します。
  </Accordion>
  <Accordion title="エイリアス">
    既存の設定では `azure` がプロバイダーのエイリアスとして受け付けられますが、
    Azure OpenAI モデルプロバイダーとの混同を避けるため、新しい設定では
    `azure-speech` を使用してください。
  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="テキスト読み上げ" href="/ja-JP/tools/tts" icon="waveform-lines">
    TTS の概要、プロバイダー、および `tts` の設定。
  </Card>
  <Card title="設定" href="/ja-JP/gateway/configuration" icon="gear">
    `tts` の設定を含む、完全な設定リファレンス。
  </Card>
  <Card title="プロバイダー" href="/ja-JP/providers" icon="grid">
    OpenClaw にバンドルされているすべてのプロバイダー。
  </Card>
  <Card title="トラブルシューティング" href="/ja-JP/help/troubleshooting" icon="wrench">
    一般的な問題とデバッグ手順。
  </Card>
</CardGroup>
