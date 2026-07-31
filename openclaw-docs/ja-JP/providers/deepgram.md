---
read_when:
    - 音声添付ファイルに Deepgram の音声テキスト変換を使用したい場合
    - Voice Call で Deepgram のストリーミング文字起こしを使用したい場合
    - Deepgram の簡単な設定例が必要です
summary: 受信した音声メモの Deepgram 文字起こし
title: Deepgram
x-i18n:
    generated_at: "2026-07-26T09:40:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c00473762c3bede1f6de9230043827d90daefd68d05e67ed4b3e3026b9d6ba4f
    source_path: providers/deepgram.md
    workflow: 16
---

Deepgram は音声テキスト変換 API です。OpenClaw は、`tools.media.audio` を介した受信音声／ボイスノートの
文字起こしと、`plugins.entries.voice-call.config.streaming` を介した Voice Call のストリーミング STT
に使用します。

バッチ文字起こしでは、音声ファイル全体を Deepgram にアップロードし、
文字起こしを応答パイプライン（`{{Transcript}}` + `[Audio]` ブロック）に挿入します。
Voice Call ストリーミングでは、ライブの G.711 u-law フレームを Deepgram の
WebSocket `listen` エンドポイント経由で転送し、Deepgram から返される
途中および最終の文字起こしを出力します。

| 詳細          | 値                                                         |
| ------------- | ---------------------------------------------------------- |
| Web サイト    | [deepgram.com](https://deepgram.com)                       |
| ドキュメント  | [developers.deepgram.com](https://developers.deepgram.com) |
| 認証          | `DEEPGRAM_API_KEY`                                         |
| デフォルトモデル | `nova-3`                                      |

## はじめに

<Steps>
  <Step title="API キーを設定する">
    ```bash
    DEEPGRAM_API_KEY=dg_...
    ```
  </Step>
  <Step title="音声プロバイダーを有効にする">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [{ provider: "deepgram", model: "nova-3" }],
          },
        },
      },
    }
    ```
  </Step>
  <Step title="ボイスノートを送信する">
    接続済みの任意のチャネルから音声メッセージを送信します。OpenClaw は Deepgram
    経由で文字起こしし、その結果を応答パイプラインに挿入します。
  </Step>
</Steps>

## 設定オプション

| オプション | パス                            | 説明                                  |
| ---------- | ------------------------------- | ------------------------------------- |
| `model` | `tools.media.models[].model`    | Deepgram モデル ID（デフォルト：`nova-3`） |
| `language` | `tools.media.models[].language` | 言語ヒント（任意）                    |

`providerOptions.deepgram` は追加のクエリパラメーターを Deepgram の
`/listen` リクエストに直接マージするため、Deepgram がサポートする任意のパラメーター名を使用できます
（例：`detect_language`、`punctuate`、`smart_format`）。

<Tabs>
  <Tab title="言語ヒントを使用">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [{ provider: "deepgram", model: "nova-3", language: "en" }],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Deepgram オプションを使用">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            providerOptions: {
              deepgram: {
                detect_language: true,
                punctuate: true,
                smart_format: true,
              },
            },
            models: [{ provider: "deepgram", model: "nova-3" }],
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## Voice Call ストリーミング STT

同梱の `deepgram` Plugin は、Voice Call Plugin 用の
リアルタイム文字起こしプロバイダーも登録します。

| 設定            | 設定パス                                                                | デフォルト                                   |
| --------------- | ----------------------------------------------------------------------- | -------------------------------------------- |
| API キー        | `plugins.entries.voice-call.config.streaming.providers.deepgram.apiKey` | `DEEPGRAM_API_KEY` にフォールバック          |
| ベース URL      | `...deepgram.baseUrl`                                                   | `DEEPGRAM_BASE_URL` または Deepgram の公開 API |
| モデル          | `...deepgram.model`                                                     | `nova-3`                           |
| 言語            | `...deepgram.language`                                                  | （未設定）                                   |
| エンコーディング | `...deepgram.encoding`                                                 | `mulaw`                           |
| サンプルレート  | `...deepgram.sampleRate`                                                | `8000`                           |
| エンドポイント判定 | `...deepgram.endpointingMs`                                          | `800`                           |
| 中間結果        | `...deepgram.interimResults`                                            | `true`                           |

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          streaming: {
            enabled: true,
            provider: "deepgram",
            providers: {
              deepgram: {
                apiKey: "${DEEPGRAM_API_KEY}",
                model: "nova-3",
                endpointingMs: 800,
                language: "en-US",
              },
            },
          },
        },
      },
    },
  },
}
```

[Deepgram カスタムエンドポイント](https://developers.deepgram.com/reference/custom-endpoints)を使用するには、
`baseUrl` を、`/listen` を含めず、ベースパスを含むエンドポイントルートに設定します。
リアルタイムエンドポイントでは、`http://`、`https://`、`ws://`、`wss://` を使用できます。HTTP
は WS に、HTTPS は WSS にマッピングされ、明示的な WebSocket スキームは変更されません。
不正な形式の URL やその他のスキームは、セッション設定中に失敗します。

<Note>
Voice Call は、電話音声を 8 kHz G.711 u-law として受信します。Deepgram
ストリーミングプロバイダーのデフォルトは `encoding: "mulaw"` と `sampleRate: 8000` であるため、
Twilio メディアフレームを直接転送できます。
</Note>

## 注記

<AccordionGroup>
  <Accordion title="認証">
    認証は標準のプロバイダー認証順序に従います。`DEEPGRAM_API_KEY` が
    最も簡単な方法です。
  </Accordion>
  <Accordion title="プロキシとカスタムエンドポイント">
    プロキシを使用する場合は、Deepgram の `tools.media.models[]` エントリでエンドポイントまたはヘッダーを上書きします。
  </Accordion>
  <Accordion title="出力動作">
    出力は、他のプロバイダーと同じ音声ルール（サイズ上限、タイムアウト、
    文字起こしの挿入）に従います。
  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="メディアツール" href="/ja-JP/tools/media-overview" icon="photo-film">
    音声、画像、動画の処理パイプラインの概要です。
  </Card>
  <Card title="設定" href="/ja-JP/gateway/configuration" icon="gear">
    メディアツール設定を含む完全な設定リファレンスです。
  </Card>
  <Card title="トラブルシューティング" href="/ja-JP/help/troubleshooting" icon="wrench">
    一般的な問題とデバッグ手順です。
  </Card>
  <Card title="よくある質問" href="/ja-JP/help/faq" icon="circle-question">
    OpenClaw のセットアップに関するよくある質問です。
  </Card>
</CardGroup>
