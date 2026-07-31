---
read_when:
    - 音声添付ファイルの音声テキスト変換に SenseAudio を使用したい場合
    - SenseAudio API キーの環境変数または音声設定パスが必要です
summary: 受信した音声メモを SenseAudio で一括音声テキスト変換
title: SenseAudio
x-i18n:
    generated_at: "2026-07-26T10:29:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0ca4a31a32eed85c1d9dcd13ebc2eaea94be370d2b1013ae8b4677949bea91d
    source_path: providers/senseaudio.md
    workflow: 16
---

SenseAudio は、OpenClaw の共有 `tools.media.audio` パイプラインを通じて、受信した音声およびボイスメモの添付ファイルを文字起こしします。OpenClaw は、OpenAI 互換の文字起こしエンドポイントにマルチパート形式で音声を送信し、返されたテキストを `{{Transcript}}` および `[Audio]` ブロックとして挿入します。

| プロパティ      | 値                                            |
| ------------- | ------------------------------------------------ |
| プロバイダー ID   | `senseaudio`                                     |
| Plugin        | バンドル済み、`enabledByDefault: true`                |
| コントラクト      | `mediaUnderstandingProviders`（音声）            |
| 認証環境変数  | `SENSEAUDIO_API_KEY`                             |
| デフォルトモデル | `senseaudio-asr-pro-1.5-260319`                  |
| デフォルト URL   | `https://api.senseaudio.cn/v1`                   |
| ウェブサイト       | [senseaudio.cn](https://senseaudio.cn)           |
| ドキュメント          | [senseaudio.cn/docs](https://senseaudio.cn/docs) |

## はじめに

<Steps>
  <Step title="API キーを設定する">
    ```bash
    export SENSEAUDIO_API_KEY="..."
    ```
  </Step>
  <Step title="音声プロバイダーを有効にする">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [{ provider: "senseaudio", model: "senseaudio-asr-pro-1.5-260319" }],
          },
        },
      },
    }
    ```
  </Step>
  <Step title="ボイスメモを送信する">
    接続済みの任意のチャンネルから音声メッセージを送信します。OpenClaw は
    音声を SenseAudio にアップロードし、返信パイプラインで文字起こしを使用します。
  </Step>
</Steps>

## オプション

| オプション     | パス                            | 説明                         |
| ---------- | ------------------------------- | ----------------------------------- |
| `model`    | `tools.media.models[].model`    | SenseAudio ASR モデル ID             |
| `language` | `tools.media.models[].language` | オプションの言語ヒント              |
| `prompt`   | `tools.media.models[].prompt`   | オプションの文字起こしプロンプト       |
| `baseUrl`  | `tools.media.models[].baseUrl`  | OpenAI 互換ベース URL を上書き |
| `headers`  | `tools.media.models[].headers`  | 追加のリクエストヘッダー               |

<Note>
OpenClaw では、SenseAudio はバッチ STT のみに対応しています。Voice Call のリアルタイム文字起こしでは、
引き続きストリーミング STT をサポートするプロバイダーを使用します。
</Note>

## 関連項目

- [メディア理解（音声）](/ja-JP/nodes/audio)
- [モデルプロバイダー](/ja-JP/concepts/model-providers)
