---
read_when:
    - 返信のテキスト読み上げを有効にする
    - TTS プロバイダー、フォールバックチェーン、またはペルソナの設定
    - /tts コマンドまたはディレクティブの使用
sidebarTitle: Text to speech (TTS)
summary: 送信返信のテキスト読み上げ — プロバイダー、ペルソナ、スラッシュコマンド、チャンネル別出力
title: テキスト読み上げ
x-i18n:
    generated_at: "2026-07-26T10:25:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2ae9d0cc6f77c6a8b1b379c3712fd92fbbc22dae694ecdd46a0bb35cec0d29e7
    source_path: tools/tts.md
    workflow: 16
---

OpenClaw は、送信する返信を **14 の音声プロバイダー**で音声に変換します。
Feishu、Matrix、Telegram、WhatsApp ではネイティブ音声メッセージ、それ以外では音声
添付ファイル、電話および Talk では PCM/Ulaw ストリームとして出力します。

TTS は、Talk の `stt-tts` モードにおける音声出力側の機能です（`talk.speak` も
同じ合成パスを呼び出します）。プロバイダーにネイティブ対応する `realtime` Talk セッションでは、
リアルタイムプロバイダー内で音声を合成します。一方、`transcription` セッションでは
アシスタントの音声返信を合成しません。

## クイックスタート

<Steps>
  <Step title="プロバイダーを選択">
    OpenAI と ElevenLabs は、最も信頼性の高いホスティング型の選択肢です。Microsoft と
    Local CLI は API キーなしで動作します。完全な一覧については、[プロバイダー一覧表](#supported-providers)
    を参照してください。
  </Step>
  <Step title="API キーを設定">
    プロバイダーの環境変数（例: `OPENAI_API_KEY`、
    `ELEVENLABS_API_KEY`）をエクスポートします。Microsoft と Local CLI にはキーが不要です。
  </Step>
  <Step title="設定で有効化">
    `tts.auto: "always"` と `tts.provider` を設定します。

    ```json5
    {
      tts: {
        auto: "always",
        provider: "elevenlabs",
      },
    }
    ```

  </Step>
  <Step title="チャットで試す">
    `/tts status` で現在の状態を表示します。`/tts audio Hello from OpenClaw` で
    一度限りの音声返信を送信します。
  </Step>
</Steps>

<Note>
自動 TTS はデフォルトで**オフ**です。`tts.provider` が未設定の場合、
OpenClaw はレジストリの自動選択順で、最初に設定済みのプロバイダーを選択します。
組み込みの `tts` エージェントツールは、明示的な意図がある場合にのみ動作します。ユーザーが音声を要求するか、
`/tts` を使用するか、自動 TTS／ディレクティブ音声を有効にしない限り、通常のチャットは
テキストのままです。
</Note>

## 対応プロバイダー

| プロバイダー      | 認証                                                                                                             | 備考                                                                                       |
| ----------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Azure Speech**  | `AZURE_SPEECH_KEY` + `AZURE_SPEECH_REGION`（`AZURE_SPEECH_API_KEY`、`SPEECH_KEY`、`SPEECH_REGION` にも対応）          | ネイティブの Ogg/Opus 音声メモ出力および電話に対応。                                            |
| **DeepInfra**     | `DEEPINFRA_API_KEY`                                                                                              | OpenAI 互換 TTS。デフォルトは `hexgrad/Kokoro-82M`。                                    |
| **ElevenLabs**    | `ELEVENLABS_API_KEY` または `XI_API_KEY`                                                                             | 音声クローニング、多言語対応、`seed` による決定論的出力。Discord の音声再生ではストリーミングされます。 |
| **Google Gemini** | `GEMINI_API_KEY` または `GOOGLE_API_KEY`                                                                             | Gemini API のバッチ TTS。`promptTemplate: "audio-profile-v1"` によりペルソナに対応。               |
| **Gradium**       | `GRADIUM_API_KEY`                                                                                                | 音声メモおよび電話出力。                                                            |
| **Inworld**       | `INWORLD_API_KEY`                                                                                                | ストリーミング TTS API。ネイティブの Opus 音声メモおよび PCM 電話出力。                                |
| **Local CLI**     | なし                                                                                                             | 設定済みのローカル TTS コマンドを実行します。                                                        |
| **Microsoft**     | なし                                                                                                             | `node-edge-tts` を介した公開 Edge ニューラル TTS。ベストエフォートであり、SLA はありません。                            |
| **MiniMax**       | `MINIMAX_API_KEY`（または Token Plan: `MINIMAX_OAUTH_TOKEN`、`MINIMAX_CODE_PLAN_KEY`、`MINIMAX_CODING_API_KEY`）      | T2A v2 API。デフォルトは `speech-2.8-hd`。                                                    |
| **OpenAI**        | `OPENAI_API_KEY`                                                                                                 | 自動要約にも使用されます。ペルソナ `instructions` に対応。                                |
| **OpenRouter**    | `OPENROUTER_API_KEY`（`models.providers.openrouter.apiKey` を再利用可能）                                            | デフォルトモデルは `hexgrad/kokoro-82m`。                                                         |
| **Volcengine**    | `VOLCENGINE_TTS_API_KEY` または `BYTEPLUS_SEED_SPEECH_API_KEY`（従来の AppID／トークン: `VOLCENGINE_TTS_APPID`/`_TOKEN`） | BytePlus Seed Speech HTTP API。                                                              |
| **Vydra**         | `VYDRA_API_KEY`                                                                                                  | 画像、動画、音声で共通のプロバイダー。                                                   |
| **xAI**           | `XAI_API_KEY`                                                                                                    | xAI バッチ TTS。ネイティブの Opus 音声メモには**対応していません**。                                 |
| **Xiaomi MiMo**   | `XIAOMI_API_KEY`                                                                                                 | Xiaomi のチャット補完を介した MiMo TTS。                                                   |

複数のプロバイダーが設定されている場合、選択したプロバイダーが最初に使用され、
ほかのプロバイダーはフォールバック候補になります。自動要約では `summaryModel`（または
`agents.defaults.model.primary`）が使用されるため、要約を有効なままにする場合は、そのプロバイダーでも
認証が必要です。

<Warning>
バンドルされている **Microsoft** プロバイダーは、`node-edge-tts` を介して Microsoft Edge のオンラインニューラル TTS
サービスを使用します。公開された SLA やクォータのないパブリック Web サービスであるため、
ベストエフォートとして扱ってください。従来のプロバイダー ID `edge` は
`microsoft` に正規化され、`openclaw doctor --fix` により永続化された
設定が書き換えられます。新しい設定では常に `microsoft` を使用してください。
</Warning>

## 設定

TTS 設定は `~/.openclaw/openclaw.json` の `tts` 配下にあります。プリセットを
選択し、プロバイダーブロックを調整してください。以下に示す `speakerVoice`/`speakerVoiceId`
フィールドが正規の形式です。各プロバイダー固有の `voice`/`voiceId`/
`voiceName` フィールド名も、従来のエイリアスとして引き続き使用できます。

<Tabs>
  <Tab title="Azure Speech">
```json5
{
  tts: {
    auto: "always",
    provider: "azure-speech",
    providers: {
      "azure-speech": {
        apiKey: "${AZURE_SPEECH_KEY}",
        region: "eastus",
        speakerVoice: "en-US-JennyNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
        voiceNoteOutputFormat: "ogg-24khz-16bit-mono-opus",
      },
    },
  },
}
```
  </Tab>
  <Tab title="ElevenLabs">
```json5
{
  tts: {
    auto: "always",
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        model: "eleven_multilingual_v2",
        speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Google Gemini">
```json5
{
  tts: {
    auto: "always",
    provider: "google",
    providers: {
      google: {
        apiKey: "${GEMINI_API_KEY}",
        model: "gemini-3.1-flash-tts-preview",
        speakerVoice: "Kore",
        // Optional natural-language style prompts:
        // audioProfile: "Speak in a calm, podcast-host tone.",
        // speakerName: "Alex",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Gradium">
```json5
{
  tts: {
    auto: "always",
    provider: "gradium",
    providers: {
      gradium: {
        apiKey: "${GRADIUM_API_KEY}",
        speakerVoiceId: "YTpq7expH9539ERJ",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Inworld">
```json5
{
  tts: {
    auto: "always",
    provider: "inworld",
    providers: {
      inworld: {
        apiKey: "${INWORLD_API_KEY}",
        modelId: "inworld-tts-1.5-max",
        speakerVoiceId: "Sarah",
        temperature: 0.7,
      },
    },
  },
}
```
  </Tab>
  <Tab title="Local CLI">
```json5
{
  tts: {
    auto: "always",
    provider: "tts-local-cli",
    providers: {
      "tts-local-cli": {
        command: "say",
        args: ["-o", "{{OutputPath}}", "{{Text}}"],
        outputFormat: "wav",
        timeoutMs: 120000,
      },
    },
  },
}
```
  </Tab>
  <Tab title="Microsoft（キー不要）">
```json5
{
  tts: {
    auto: "always",
    provider: "microsoft",
    providers: {
      microsoft: {
        enabled: true,
        speakerVoice: "en-US-MichelleNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
        rate: "+0%",
        pitch: "+0%",
      },
    },
  },
}
```
  </Tab>
  <Tab title="MiniMax">
```json5
{
  tts: {
    auto: "always",
    provider: "minimax",
    providers: {
      minimax: {
        apiKey: "${MINIMAX_API_KEY}",
        model: "speech-2.8-hd",
        speakerVoiceId: "English_expressive_narrator",
        speed: 1.0,
        vol: 1.0,
        pitch: 0,
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenAI + ElevenLabs">
```json5
{
  tts: {
    auto: "always",
    provider: "openai",
    summaryModel: "openai/gpt-4.1-mini",
    modelOverrides: { enabled: true },
    providers: {
      openai: {
        apiKey: "${OPENAI_API_KEY}",
        model: "gpt-4o-mini-tts",
        speakerVoice: "alloy",
      },
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        model: "eleven_multilingual_v2",
        speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
        voiceSettings: { stability: 0.5, similarityBoost: 0.75, style: 0.0, useSpeakerBoost: true, speed: 1.0 },
        applyTextNormalization: "auto",
        languageCode: "en",
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenRouter">
```json5
{
  tts: {
    auto: "always",
    provider: "openrouter",
    providers: {
      openrouter: {
        apiKey: "${OPENROUTER_API_KEY}",
        model: "hexgrad/kokoro-82m",
        speakerVoice: "af_alloy",
        responseFormat: "mp3",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Volcengine">
```json5
{
  tts: {
    auto: "always",
    provider: "volcengine",
    providers: {
      volcengine: {
        apiKey: "${VOLCENGINE_TTS_API_KEY}",
        resourceId: "seed-tts-1.0",
        speakerVoice: "en_female_anna_mars_bigtts",
      },
    },
  },
}
```
  </Tab>
  <Tab title="xAI">
```json5
{
  tts: {
    auto: "always",
    provider: "xai",
    providers: {
      xai: {
        apiKey: "${XAI_API_KEY}",
        speakerVoiceId: "eve",
        language: "en",
        responseFormat: "mp3",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Xiaomi MiMo">
```json5
{
  tts: {
    auto: "always",
    provider: "xiaomi",
    providers: {
      xiaomi: {
        apiKey: "${XIAOMI_API_KEY}",
        model: "mimo-v2.5-tts",
        speakerVoice: "mimo_default",
        format: "mp3",
      },
    },
  },
}
```
  </Tab>
</Tabs>

Xiaomi `mimo-v2.5-tts-voicedesign` では、`speakerVoice` を省略し、`style` に
音声設計プロンプトを設定します。OpenClaw はそのプロンプトを TTS の `user` メッセージとして送信し、
voicedesign モデルには `audio.voice` を送信しません。

### エージェントごとの音声オーバーライド

1 つのエージェントで異なるプロバイダー、音声、モデル、ペルソナ、または自動 TTS モードを
使用する場合は、`agents.entries.*.tts` を使用します。エージェントブロックは
`tts` にディープマージされるため、プロバイダーの認証情報はグローバルなプロバイダー設定に保持できます。

```json5
{
  tts: {
    auto: "always",
    provider: "elevenlabs",
    providers: {
      elevenlabs: { apiKey: "${ELEVENLABS_API_KEY}", model: "eleven_multilingual_v2" },
    },
  },
  agents: {
    list: [
      {
        id: "reader",
        tts: {
          providers: {
            elevenlabs: { speakerVoiceId: "EXAVITQu4vr4xnSDxMaL" },
          },
        },
      },
    ],
  },
}
```

エージェントごとのペルソナを固定するには、プロバイダー設定とともに `agents.entries.*.tts.persona` を設定します。これは、そのエージェントに限りグローバルな `tts.persona` を上書きします。

自動返信、`/tts audio`、`/tts status`、および
`tts` エージェントツールの優先順位：

1. `tts`
2. 有効な `agents.entries.*.tts`
3. チャンネルが `channels.<channel>.tts` をサポートしている場合のチャンネル上書き
4. チャンネルが `channels.<channel>.accounts.<id>.tts` を渡す場合のアカウント上書き
5. このホストのローカルな `/tts` 設定
6. [モデル駆動ディレクティブ](#model-driven-directives)が有効な場合のインライン `[[tts:...]]` ディレクティブ

チャンネルとアカウントの上書きは `tts` と同じ形式を使用し、
前のレイヤーに対してディープマージされます。そのため、共有プロバイダー認証情報を
`tts` に保持したまま、チャンネルまたはボットアカウントで話者の声、モデル、ペルソナ、
または自動モードのみを変更できます。

```json5
{
  tts: {
    provider: "openai",
    providers: {
      openai: { apiKey: "${OPENAI_API_KEY}", model: "gpt-4o-mini-tts" },
    },
  },
  channels: {
    feishu: {
      accounts: {
        english: {
          tts: {
            providers: {
              openai: { speakerVoice: "shimmer" },
            },
          },
        },
      },
    },
  },
}
```

## ペルソナ

**ペルソナ**とは、プロバイダー間で決定論的に適用できる、安定した音声上のアイデンティティです。
特定のプロバイダーを優先し、プロバイダーに依存しないプロンプトの意図を定義し、
音声、モデル、プロンプトテンプレート、シード、音声設定に対する
プロバイダー固有のバインディングを保持できます。

### 最小構成のペルソナ

```json5
{
  tts: {
    auto: "always",
    persona: "narrator",
    personas: {
      narrator: {
        label: "ナレーター",
        provider: "elevenlabs",
        providers: {
          elevenlabs: {
            speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
            modelId: "eleven_multilingual_v2",
          },
        },
      },
    },
  },
}
```

### 完全なペルソナ（プロバイダー固有の調整）

```json5
{
  tts: {
    auto: "always",
    persona: "alfred",
    personas: {
      alfred: {
        label: "アルフレッド",
        description: "辛口ながら温かみのある、英国執事風のナレーター。",
        provider: "google",
        fallbackPolicy: "preserve-persona",
        providers: {
          google: {
            model: "gemini-3.1-flash-tts-preview",
            speakerVoice: "Algieba",
            promptTemplate: "audio-profile-v1",
          },
          openai: { model: "gpt-4o-mini-tts", speakerVoice: "cedar" },
          elevenlabs: {
            speakerVoiceId: "voice_id",
            modelId: "eleven_multilingual_v2",
            seed: 42,
            voiceSettings: {
              stability: 0.65,
              similarityBoost: 0.8,
              style: 0.25,
              useSpeakerBoost: true,
              speed: 0.95,
            },
          },
        },
      },
    },
  },
}
```

### ペルソナの解決

有効なペルソナは決定論的に選択されます。

1. 設定されている場合、ローカル設定の `/tts persona <id>`。
2. 設定されている場合、`tts.persona`。
3. ペルソナなし。

プロバイダー選択では、明示的な指定が最優先されます。

1. 直接上書き（CLI、Gateway、Talk、許可された TTS ディレクティブ）。
2. ローカル設定の `/tts provider <id>`。
3. 有効なペルソナの `provider`。
4. `tts.provider`。
5. レジストリによる自動選択。

プロバイダーを試行するたびに、OpenClaw は次の順序で設定をマージします。

1. `tts.providers.<id>`
2. `tts.personas.<persona>.providers.<id>`
3. 信頼されたリクエストによる上書き
4. 許可された、モデルが出力した TTS ディレクティブによる上書き

### カスタムペルソナの調整

プロバイダーに依存しない `personas.<id>.prompt.*` 設定は廃止されました。Doctor は
これらのフィールドを削除し、音声プロバイダーのシームを案内します。組み込みプロバイダーの
設定は `personas.<id>.providers.<provider>` 配下に配置します（たとえば Google の
`personaPrompt` または OpenAI の `instructions`）。カスタム調整を行うには、
`prepareSynthesis(ctx)` を備えた音声プロバイダー Plugin を実装し、
`synthesize()` が実行される前に、調整済みのテキスト、プロバイダー設定、または上書きを返します。
これにより、表現力のあるプロンプトの構築を、リクエストのセマンティクスを把握している
プロバイダーコード内に維持できます。

### フォールバックポリシー

`fallbackPolicy` は、試行対象のプロバイダーにペルソナの**バインディングがない**場合の
動作を制御します。

| ポリシー              | 動作                                                                                                                                         |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `preserve-persona`  | **デフォルト。** プロバイダーに依存しないプロンプトフィールドは引き続き利用可能で、プロバイダーはそれらを使用することも無視することもできます。                                            |
| `provider-defaults` | その試行のプロンプト準備からペルソナを除外します。プロバイダーは中立的なデフォルトを使用し、他のプロバイダーへのフォールバックは継続します。 |
| `fail`              | `reasonCode: "not_configured"` と `personaBinding: "missing"` を伴って、そのプロバイダーの試行をスキップします。フォールバックプロバイダーは引き続き試行されます。              |

TTS リクエスト全体が失敗するのは、試行した**すべての**プロバイダーがスキップされるか
失敗した場合のみです。

Talk セッションのプロバイダー選択はセッション単位です。Talk クライアントは
`talk.catalog` からプロバイダー ID、モデル ID、音声 ID、ロケールを選択し、
Talk セッションまたはハンドオフリクエストを通じて渡す必要があります。音声セッションを開く際に、
`tts` やグローバルな Talk プロバイダーのデフォルトを変更してはなりません。

## モデル駆動ディレクティブ

デフォルトでは、アシスタントは `[[tts:...]]` ディレクティブを出力して、
単一の返信に対する音声、モデル、または速度を上書き**できます**。さらに、
音声のみに含める表現上の指示として、任意の
`[[tts:text]]...[[/tts:text]]` ブロックも使用できます。

```text
どうぞ。

[[tts:speakerVoiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]]（笑う）その歌をもう一度読んでください。[[/tts:text]]
```

`tts.auto` が `"tagged"` の場合、音声をトリガーするには
**ディレクティブが必須**です。ストリーミングのブロック配信では、隣接するブロックに
分割されている場合でも、チャンネルに届く前に表示テキストからディレクティブが除去されます。

`provider=...` は、`modelOverrides.allowProvider: true` でない限り無視されます。
返信で `provider=...` が宣言されている場合、そのディレクティブ内の他のキーは
そのプロバイダーのみが解析します。サポートされていないキーは除去され、
TTS ディレクティブの警告として報告されます。

**利用可能なディレクティブキー：**

- `provider`（登録済みのプロバイダー ID。`allowProvider: true` が必要）
- `speakerVoice` / `speakerVoiceId`（旧エイリアス：`voice`、`voiceName`、`voice_name`、`google_voice`、`voiceId`）
- `model` / `google_model`
- `stability`、`similarityBoost`、`style`、`speed`、`useSpeakerBoost`
- `vol` / `volume`（MiniMax の音量、`(0, 10]`）
- `pitch`（MiniMax の整数ピッチ、−12～12。小数値は切り捨て）
- `emotion`（Volcengine の感情タグ）
- `applyTextNormalization`（`auto|on|off`）
- `languageCode`（ISO 639-1）
- `seed`

**モデルによる上書きを完全に無効化：**

```json5
{ messages: { tts: { modelOverrides: { enabled: false } } } }
```

**他の調整項目を設定可能なまま、プロバイダーの切り替えを許可：**

```json5
{ messages: { tts: { modelOverrides: { enabled: true, allowProvider: true, allowSeed: false } } } }
```

## スラッシュコマンド

単一コマンド `/tts`。Discord では、`/tts` が Discord の
組み込みコマンドであるため、OpenClaw は `/voice` も登録します。
テキストの `/tts ...` も引き続き機能します。

```text
/tts off | on | status
/tts chat on | off | default
/tts latest
/tts provider <id>
/tts persona <id> | off
/tts limit <chars>
/tts summary off
/tts audio <text>
```

<Note>
コマンドには認可済みの送信者が必要です（許可リスト／所有者ルールが適用されます）。
また、`commands.text` またはネイティブコマンド登録のいずれかを有効にする必要があります。
</Note>

動作に関する注記：

- `/tts on` はローカルの TTS 設定を `always` に書き込み、`/tts off` は `off` に書き込みます。
- `/tts chat on|off|default` は、現在のチャットに対するセッション単位の自動 TTS 上書きを書き込みます。
- `/tts persona <id>` はローカルのペルソナ設定を書き込み、`/tts persona off` はそれを消去します。
- `/tts latest` は現在のセッショントランスクリプトから最新のアシスタント返信を読み取り、音声として一度送信します。音声の重複送信を抑止するため、その返信のハッシュのみをセッションエントリに保存します。
- `/tts audio` は一回限りの音声返信を生成します（TTS をオンには**切り替えません**）。
- `/tts limit <chars>` は **100～4096** を受け付けます（4096 は Telegram のキャプション／メッセージの最大値）。範囲外の値は拒否されます。
- `limit` と `summary` はメイン設定ではなく、**ローカル設定**に保存されます。
- `/tts status` には、最新の試行に関するフォールバック診断（`Fallback: <primary> -> <used>`、`Attempts: ...`、および試行ごとの詳細（`provider:outcome(reasonCode) latency`））が含まれます。
- `/status` は、TTS が有効な場合に、有効な TTS モードと、設定済みのプロバイダー、モデル、音声、サニタイズ済みカスタムエンドポイントのメタデータを表示します。

## ユーザーごとの設定

スラッシュコマンドは、ローカルの上書きを TTS 設定パスに書き込みます。デフォルトは
`~/.openclaw/settings/tts.json` です。`OPENCLAW_TTS_PREFS` で上書きできます。Doctor は、
廃止されたグローバルな `tts.prefsPath` の値を共有マシン状態へ移動します。
高度なマルチエージェント構成では、エージェントが意図的に個別の設定ストアを使用する場合、
引き続き `agents.entries.<id>.tts.prefsPath` を設定できます。

| 保存フィールド | 効果                                                                           |
| ------------ | -------------------------------------------------------------------------------- |
| `auto`       | ローカルの自動 TTS 上書き（`always`、`off`、…）                                     |
| `provider`   | ローカルのプライマリプロバイダー上書き                                                  |
| `persona`    | ローカルのペルソナ上書き                                                           |
| `maxLength`  | 要約／切り詰めのしきい値（デフォルト `1500` 文字、`/tts limit` の範囲は 100～4096） |
| `summarize`  | 要約の切り替え（デフォルト `true`）                                                  |

これらは、そのホストにおける `tts` と有効な
`agents.entries.*.tts` ブロックから得られる実効設定を上書きします。

## 出力形式

TTS 音声配信は、チャンネルの機能によって決まります。チャンネル Plugin は、
音声形式の TTS でプロバイダーにネイティブな `voice-note` ターゲットを要求するか、
通常の `audio-file` 合成を維持するか、また、送信前にチャンネルが
非ネイティブ出力をトランスコードするかどうかを通知します。

| ターゲット                            | 形式                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Feishu / Matrix / Telegram / WhatsApp | 音声メモへの返信では **Opus**（ElevenLabs の `opus_48000_64`、OpenAI の `opus`）が優先されます。48 kHz / 64 kbps により、明瞭さとサイズのバランスを取ります。 |
| その他のチャネル                      | **MP3**（ElevenLabs の `mp3_44100_128`、OpenAI の `mp3`）。44.1 kHz / 128 kbps が音声におけるデフォルトのバランスです。                  |
| Talk / 電話                           | プロバイダー固有の **PCM**（Inworld 22050 Hz、Google 24 kHz）、または電話用の Gradium の `ulaw_8000`。                                 |

プロバイダー別の注記：

- **Feishu / WhatsApp のトランスコード：** 音声メモへの返信が MP3/WebM/WAV/M4A またはその他の音声ファイルと思われる形式で届いた場合、チャネル Plugin はネイティブ音声メッセージを送信する前に、`ffmpeg`（`libopus`、64 kbps）を使用して 48 kHz Ogg/Opus にトランスコードします。WhatsApp は、`ptt: true` および `audio/ogg; codecs=opus` を指定した Baileys の `audio` ペイロードを通じて結果を送信します。トランスコードに失敗した場合、Feishu はエラーを捕捉し、元のファイルを通常の添付ファイルとして送信する方式にフォールバックします。WhatsApp にはフォールバックがないため、互換性のない PTT ペイロードを投稿する代わりに、送信自体が失敗します。
- **MiniMax：** 通常の音声添付ファイルには MP3（`speech-2.8-hd` モデル、32 kHz サンプルレート）を使用し、チャネルが音声メモ対応として通知しているターゲットでは `ffmpeg` を使用して 48 kHz Opus にトランスコードします。
- **Xiaomi MiMo：** デフォルトでは MP3、設定されている場合は WAV を使用します。チャネルが音声メモ対応として通知しているターゲットでは `ffmpeg` を使用して 48 kHz Opus にトランスコードします。
- **ローカル CLI：** 設定された `outputFormat` を使用します。音声メモのターゲットは Ogg/Opus に変換され、電話出力は `ffmpeg` を使用して raw 16 kHz モノラル PCM に変換されます。
- **Google Gemini：** raw 24 kHz PCM を返します。OpenClaw は音声添付ファイル用にこれを WAV としてラップし、音声メモのターゲット用に 48 kHz Opus にトランスコードし、Talk/電話用には PCM を直接返します。
- **Gradium：** 音声添付ファイルには WAV、音声メモのターゲットには Opus、電話には 8 kHz の `ulaw_8000` を使用します。
- **Inworld：** 通常の音声添付ファイルには MP3、音声メモのターゲットにはネイティブの `OGG_OPUS`、Talk/電話には 22050 Hz の raw `PCM` を使用します。
- **xAI：** デフォルトでは MP3 を使用します。音声ファイルの合成では、バッファリング出力とストリーミング出力の両方に `mp3`、`wav`、`pcm`、`mulaw`、または `alaw` を使用できます。xAI の `pcm`、`mulaw`、および `alaw` の出力はヘッダーのない raw 音声であるため、音声メモのターゲットではストリーミングおよびバッファリング時のフォールバックに MP3 を使用します。バッファリング合成では xAI のバッチ REST `/v1/tts` エンドポイントを使用し、`textToSpeechStream` ではネイティブの `wss://api.x.ai/v1/tts` を使用します。これはリアルタイム音声の契約ではありません。ネイティブ Opus 音声メモ形式はサポートされていません。
- **Microsoft：** `microsoft.outputFormat`（デフォルトは `audio-24khz-48kbitrate-mono-mp3`）を使用します。
  - バンドルされたトランスポートは `outputFormat` を受け付けますが、サービスからすべての形式を利用できるわけではありません。
  - 出力形式の値は Microsoft Speech の出力形式（Ogg/WebM Opus を含む）に従います。
  - Telegram の `sendVoice` は OGG/MP3/M4A を受け付けます。Opus 音声メッセージを確実に使用する必要がある場合は OpenAI/ElevenLabs を使用してください。
  - 設定された Microsoft の出力形式が失敗した場合、OpenClaw は MP3 で再試行します。
  - 明示的な音声の上書きが設定されておらず、デフォルトの英語音声が使用されている場合、返信テキストで CJK が大部分を占めていれば、OpenClaw は中国語ニューラル音声（`zh-CN-XiaoxiaoNeural`、`zh-CN` ロケール）に自動的に切り替えます。

OpenAI と ElevenLabs の出力形式は、上記のとおりチャネルごとに固定されています。

## 自動 TTS の動作

`tts.auto` が有効な場合、OpenClaw は次のように動作します。

- 返信に構造化メディアがすでに含まれている場合は、TTS をスキップします。
- 非常に短い返信（10 文字未満）はスキップします。
- 要約が有効な場合、`summaryModel`（または `agents.defaults.model.primary`）を使用して
  長い返信を要約します。
- 生成された音声を返信に添付します。
- `mode: "final"` では、テキストストリームの完了後も、ストリーミングされた最終返信に対して
  音声のみの TTS を送信します。生成されたメディアには、通常の返信添付ファイルと同じ
  チャネルメディア正規化が適用されます。

返信が `maxLength` を超えた場合でも、OpenClaw が音声を完全にスキップすることはありません。

- **要約オン**（デフォルト）で要約モデルが利用可能な場合：テキストを
  約 `maxLength` 文字に要約し、その要約を音声合成します。
- **要約オフ**、要約に失敗した場合、または要約モデル用の API キーが利用できない場合：
  テキストを `maxLength` 文字に切り詰め、切り詰めたテキストを
  音声合成します。

```text
返信 -> TTS は有効？
  いいえ -> テキストを送信
  はい   -> メディアを含む / 短い？
            はい   -> テキストを送信
            いいえ -> 長さ > 上限？
                      いいえ -> TTS -> 音声を添付
                      はい   -> 要約が有効かつ利用可能？
                                いいえ -> 切り詰め -> TTS -> 音声を添付
                                はい   -> 要約 -> TTS -> 音声を添付
```

## フィールドリファレンス

<AccordionGroup>
  <Accordion title="トップレベルの tts.*">
    <ParamField path="auto" type='"off" | "always" | "inbound" | "tagged"'>
      自動 TTS モード。`inbound` は受信した音声メッセージの後にのみ音声を送信します。`tagged` は返信に `[[tts:...]]` ディレクティブまたは `[[tts:text]]` ブロックが含まれている場合にのみ音声を送信します。
    </ParamField>
    <ParamField path="enabled" type="boolean" deprecated>
      旧式の切り替え設定。`openclaw doctor --fix` はこれを `auto` に移行します。
    </ParamField>
    <ParamField path="mode" type='"final" | "all"' default="final">
      `"all"` は最終返信に加えて、ツール/ブロックの返信も含めます。
    </ParamField>
    <ParamField path="provider" type="string">
      音声プロバイダー ID。未設定の場合、OpenClaw はレジストリの自動選択順で最初に設定されているプロバイダーを使用します。旧式の `provider: "edge"` は `openclaw doctor --fix` によって `"microsoft"` に書き換えられます。
    </ParamField>
    <ParamField path="persona" type="string">
      `personas` のアクティブなペルソナ ID。小文字に正規化されます。
    </ParamField>
    <ParamField path="personas.<id>" type="object">
      安定した発話アイデンティティ。フィールド：`label`、`description`、`provider`、`fallbackPolicy`、`prompt`、`providers.<provider>`。[ペルソナ](#personas)を参照してください。
    </ParamField>
    <ParamField path="summaryModel" type="string">
      自動要約用の低コストモデル。デフォルトは `agents.defaults.model.primary` です。`provider/model` または設定済みのモデルエイリアスを受け付けます。
    </ParamField>
    <ParamField path="modelOverrides" type="object">
      モデルが TTS ディレクティブを出力できるようにします。`enabled` のデフォルトは `true`、`allowProvider` のデフォルトは `false` です。
    </ParamField>
    <ParamField path="providers.<id>" type="object">
      音声プロバイダー ID をキーとする、プロバイダー所有の設定。旧式の直接ブロック（`tts.openai`、`.elevenlabs`、`.microsoft`、`.edge`）は `openclaw doctor --fix` によって書き換えられます。`tts.providers.<id>` のみをコミットしてください。
    </ParamField>
    <ParamField path="maxTextLength" type="number" default="4096">
      TTS 入力文字数の上限。超過した場合、`/tts audio`、`tts.convert`、および `tts.speak` は失敗します。
    </ParamField>
    <ParamField path="timeoutMs" type="number" default="30000">
      リクエストのタイムアウト（ミリ秒単位）。呼び出しごとの `timeoutMs`（エージェントツール、Gateway）が設定されている場合はそれが優先されます。それ以外の場合、明示的に設定された `tts.timeoutMs` が、Plugin によって設定されたプロバイダーのデフォルト値より優先されます。
    </ParamField>
  </Accordion>

プロバイダーの `apiKey` フィールドには、raw 文字列または SecretRef を指定できます。Gateway のコールド
起動時に TTS SecretRef が利用できない場合、Gateway を停止する代わりに、組み込みの TTS 機能が
設定済みだが利用不可としてマークされます。その後、`tts.speak` は理由 `SECRET_SURFACE_UNAVAILABLE` とともに
`UNAVAILABLE` を返し、プロバイダーへのリクエストは
送信されません。ステータスと doctor には、機能が低下した TTS の所有者とその設定パスが表示されます。
明示的な参照はランタイムスナップショットに残るため、環境またはプロファイルの
認証情報によって別のアカウントが暗黙的に選択されることはありません。リロードおよび設定書き込みの
事前チェックでは、所有者を考慮した機能低下ポリシーが適用されます。変更されていない適格な TTS
所有者は、最後に正常だった認証情報を古い状態のまま維持できますが、新規または変更された
障害は、正常な所有者をブロックせずにコールド状態になります。構造的に無効な参照
および解決済みの値は、引き続き起動を失敗させるか、更新を拒否します。

  <Accordion title="Azure Speech">
    <ParamField path="apiKey" type="string">環境変数：`AZURE_SPEECH_KEY`、`AZURE_SPEECH_API_KEY`、または `SPEECH_KEY`。</ParamField>
    <ParamField path="region" type="string">Azure Speech のリージョン（例：`eastus`）。環境変数：`AZURE_SPEECH_REGION` または `SPEECH_REGION`。</ParamField>
    <ParamField path="endpoint" type="string">Azure Speech エンドポイントの任意の上書き（エイリアス `baseUrl`）。</ParamField>
    <ParamField path="speakerVoice" type="string">Azure 音声の ShortName。デフォルトは `en-US-JennyNeural`。旧式のエイリアス：`voice`。</ParamField>
    <ParamField path="lang" type="string">SSML 言語コード。デフォルトは `en-US`。</ParamField>
    <ParamField path="outputFormat" type="string">標準音声用の Azure `X-Microsoft-OutputFormat`。デフォルトは `audio-24khz-48kbitrate-mono-mp3`。</ParamField>
    <ParamField path="voiceNoteOutputFormat" type="string">音声メモ出力用の Azure `X-Microsoft-OutputFormat`。デフォルトは `ogg-24khz-16bit-mono-opus`。</ParamField>
  </Accordion>

  <Accordion title="ElevenLabs">
    <ParamField path="apiKey" type="string">`ELEVENLABS_API_KEY` または `XI_API_KEY` にフォールバックします。</ParamField>
    <ParamField path="model" type="string">モデル ID。デフォルトは `eleven_multilingual_v2`。旧式の ID `eleven_turbo_v2_5`/`eleven_turbo_v2` は、対応する `flash` モデルに正規化されます。</ParamField>
    <ParamField path="speakerVoiceId" type="string">ElevenLabs の音声 ID。デフォルトは `pMsXgVXv3BLzUgSXRplE`。旧式のエイリアス：`voiceId`。</ParamField>
    <ParamField path="voiceSettings" type="object">
      `stability`、`similarityBoost`、`style`（それぞれ `0..1`、デフォルトは `0.5`/`0.75`/`0`）、`useSpeakerBoost`（`true|false`、デフォルトは `true`）、`speed`（`0.5..2.0`、デフォルトは `1.0`）。
    </ParamField>
    <ParamField path="applyTextNormalization" type='"auto" | "on" | "off"'>テキスト正規化モード。</ParamField>
    <ParamField path="languageCode" type="string">2 文字の ISO 639-1（例：`en`、`de`）。</ParamField>
    <ParamField path="seed" type="number">可能な限り決定性を確保するための整数 `0..4294967295`。</ParamField>
    <ParamField path="baseUrl" type="string">ElevenLabs API のベース URL を上書きします。</ParamField>
  </Accordion>

  <Accordion title="Google Gemini">
    <ParamField path="apiKey" type="string">`GEMINI_API_KEY` / `GOOGLE_API_KEY` にフォールバックします。省略した場合、TTS は環境変数へのフォールバック前に `models.providers.google.apiKey` を再利用できます。</ParamField>
    <ParamField path="model" type="string">Gemini TTS モデル。デフォルトは `gemini-3.1-flash-tts-preview`。</ParamField>
    <ParamField path="speakerVoice" type="string">Gemini の事前構築済み音声名。デフォルトは `Kore`。レガシーエイリアス: `voiceName`、`voice`。</ParamField>
    <ParamField path="audioProfile" type="string">発話テキストの前に付加される自然言語のスタイルプロンプト。</ParamField>
    <ParamField path="speakerName" type="string">プロンプトで名前付き話者を使用する場合に、発話テキストの前に付加される任意の話者ラベル。</ParamField>
    <ParamField path="promptTemplate" type='"audio-profile-v1"'>`audio-profile-v1` に設定すると、アクティブなペルソナプロンプトフィールドが決定論的な Gemini TTS プロンプト構造でラップされます。</ParamField>
    <ParamField path="personaPrompt" type="string">テンプレートのディレクターノートに追加される、Google 固有の追加ペルソナプロンプトテキスト。</ParamField>
    <ParamField path="baseUrl" type="string">`https://generativelanguage.googleapis.com` のみ受け付けます。</ParamField>
  </Accordion>

  <Accordion title="Gradium">
    <ParamField path="apiKey" type="string">環境変数: `GRADIUM_API_KEY`。</ParamField>
    <ParamField path="baseUrl" type="string">`api.gradium.ai` 上の HTTPS Gradium API URL。デフォルトは `https://api.gradium.ai`。</ParamField>
    <ParamField path="speakerVoiceId" type="string">デフォルトは Emma（`YTpq7expH9539ERJ`）。レガシーエイリアス: `voiceId`。</ParamField>
  </Accordion>

  <Accordion title="Inworld">
    ### Inworld プライマリ

    <ParamField path="apiKey" type="string">環境変数: `INWORLD_API_KEY`。</ParamField>
    <ParamField path="baseUrl" type="string">デフォルトは `https://api.inworld.ai`。</ParamField>
    <ParamField path="modelId" type="string">デフォルトは `inworld-tts-1.5-max`。ほかに `inworld-tts-1.5-mini`、`inworld-tts-1-max`、`inworld-tts-1` も使用できます。</ParamField>
    <ParamField path="speakerVoiceId" type="string">デフォルトは `Sarah`。レガシーエイリアス: `voiceId`。</ParamField>
    <ParamField path="temperature" type="number">サンプリング温度 `0..2`（0 を除く）。</ParamField>

  </Accordion>

  <Accordion title="ローカル CLI (tts-local-cli)">
    <ParamField path="command" type="string">CLI TTS 用のローカル実行ファイルまたはコマンド文字列。</ParamField>
    <ParamField path="args" type="string[]">コマンド引数。プレースホルダー `{{Text}}`、`{{OutputPath}}`、`{{OutputDir}}`、`{{OutputBase}}` をサポートします。</ParamField>
    <ParamField path="outputFormat" type='"mp3" | "opus" | "wav"'>想定される CLI 出力形式。音声添付ファイルのデフォルトは `mp3`。</ParamField>
    <ParamField path="timeoutMs" type="number">コマンドのタイムアウト（ミリ秒）。デフォルトは `120000`。</ParamField>
    <ParamField path="cwd" type="string">任意のコマンド作業ディレクトリ。</ParamField>
    <ParamField path="env" type="Record<string, string>">コマンド用の任意の環境変数オーバーライド。</ParamField>

    コマンドの標準出力、および生成または変換された音声は 50 MiB に制限されます。診断用の標準エラー出力は 1 MiB に制限されます。いずれかの上限を超えると、OpenClaw はコマンドを終了し、音声合成を失敗させます。

  </Accordion>

  <Accordion title="Microsoft（API キー不要）">
    <ParamField path="enabled" type="boolean" default="true">Microsoft 音声の使用を許可します。</ParamField>
    <ParamField path="speakerVoice" type="string">Microsoft ニューラル音声名（例: `en-US-MichelleNeural`）。レガシーエイリアス: `voice`。デフォルトの英語音声が有効で、返信テキストの大部分が CJK の場合、OpenClaw は自動的に `zh-CN-XiaoxiaoNeural` に切り替えます。</ParamField>
    <ParamField path="lang" type="string">言語コード（例: `en-US`）。</ParamField>
    <ParamField path="outputFormat" type="string">Microsoft 出力形式。デフォルトは `audio-24khz-48kbitrate-mono-mp3`。バンドルされている Edge ベースのトランスポートでは、すべての形式がサポートされるわけではありません。</ParamField>
    <ParamField path="rate / pitch / volume" type="string">パーセント文字列（例: `+10%`、`-5%`）。</ParamField>
    <ParamField path="saveSubtitles" type="boolean">音声ファイルとともに JSON 字幕を書き込みます。</ParamField>
    <ParamField path="proxy" type="string">Microsoft 音声リクエスト用のプロキシ URL。</ParamField>
    <ParamField path="timeoutMs" type="number">リクエストタイムアウトのオーバーライド（ミリ秒）。</ParamField>
    <ParamField path="edge.*" type="object" deprecated>レガシーエイリアス。`openclaw doctor --fix` を実行して、永続化された設定を `providers.microsoft` に書き換えてください。</ParamField>
  </Accordion>

  <Accordion title="MiniMax">
    <ParamField path="apiKey" type="string">`MINIMAX_API_KEY` にフォールバックします。`MINIMAX_OAUTH_TOKEN`、`MINIMAX_CODE_PLAN_KEY`、または `MINIMAX_CODING_API_KEY` による Token Plan 認証。</ParamField>
    <ParamField path="baseUrl" type="string">デフォルトは `https://api.minimax.io`。環境変数: `MINIMAX_API_HOST`。</ParamField>
    <ParamField path="model" type="string">デフォルトは `speech-2.8-hd`。環境変数: `MINIMAX_TTS_MODEL`。</ParamField>
    <ParamField path="speakerVoiceId" type="string">デフォルトは `English_expressive_narrator`。環境変数: `MINIMAX_TTS_VOICE_ID`。レガシーエイリアス: `voiceId`。</ParamField>
    <ParamField path="speed" type="number">`0.5..2.0`。デフォルトは `1.0`。</ParamField>
    <ParamField path="vol" type="number">`(0, 10]`。デフォルトは `1.0`。</ParamField>
    <ParamField path="pitch" type="number">整数 `-12..12`。デフォルトは `0`。小数値はリクエスト前に切り捨てられます。</ParamField>
  </Accordion>

  <Accordion title="OpenAI">
    <ParamField path="apiKey" type="string">`OPENAI_API_KEY` にフォールバックします。</ParamField>
    <ParamField path="model" type="string">OpenAI TTS モデル ID。デフォルトは `gpt-4o-mini-tts`。</ParamField>
    <ParamField path="speakerVoice" type="string">音声名（例: `alloy`、`cedar`）。デフォルトは `coral`。レガシーエイリアス: `voice`。</ParamField>
    <ParamField path="instructions" type="string">明示的な OpenAI `instructions` フィールド。設定すると、ペルソナプロンプトフィールドは自動マッピング**されません**。</ParamField>
    <ParamField path="extraBody / extra_body" type="Record<string, unknown>">生成された OpenAI TTS フィールドの後で、`/audio/speech` リクエスト本文にマージされる追加の JSON フィールド。`lang` のようなプロバイダー固有のキーを必要とする Kokoro などの OpenAI 互換エンドポイントに使用します。安全でないプロトタイプキーは無視されます。</ParamField>
    <ParamField path="baseUrl" type="string">
      OpenAI TTS エンドポイントをオーバーライドします。解決順序: 設定 → `OPENAI_TTS_BASE_URL` → `https://api.openai.com/v1`。デフォルト以外の値は OpenAI 互換 TTS エンドポイントとして扱われるため、カスタムのモデル名と音声名が受け付けられ、`speed` には `0.25..4.0` の範囲チェックが適用されなくなります。
    </ParamField>
  </Accordion>

  <Accordion title="OpenRouter">
    <ParamField path="apiKey" type="string">環境変数: `OPENROUTER_API_KEY`。`models.providers.openrouter.apiKey` を再利用できます。</ParamField>
    <ParamField path="baseUrl" type="string">デフォルトは `https://openrouter.ai/api/v1`。レガシーの `https://openrouter.ai/v1` は正規化されます。</ParamField>
    <ParamField path="model" type="string">デフォルトは `hexgrad/kokoro-82m`。エイリアス: `modelId`。</ParamField>
    <ParamField path="speakerVoice" type="string">デフォルトは `af_alloy`。レガシーエイリアス: `voice`、`voiceId`。</ParamField>
    <ParamField path="responseFormat" type='"mp3" | "pcm"'>デフォルトは `mp3`。</ParamField>
    <ParamField path="speed" type="number">プロバイダー固有の速度オーバーライド。</ParamField>
  </Accordion>

  <Accordion title="Volcengine (BytePlus Seed Speech)">
    <ParamField path="apiKey" type="string">環境変数: `VOLCENGINE_TTS_API_KEY` または `BYTEPLUS_SEED_SPEECH_API_KEY`。</ParamField>
    <ParamField path="resourceId" type="string">デフォルトは `seed-tts-1.0`。環境変数: `VOLCENGINE_TTS_RESOURCE_ID`。プロジェクトに TTS 2.0 の利用資格がある場合は `seed-tts-2.0` を使用します。</ParamField>
    <ParamField path="appKey" type="string">App キーヘッダー。デフォルトは `aGjiRDfUWi`。環境変数: `VOLCENGINE_TTS_APP_KEY`。</ParamField>
    <ParamField path="baseUrl" type="string">Seed Speech TTS HTTP エンドポイントをオーバーライドします。環境変数: `VOLCENGINE_TTS_BASE_URL`。</ParamField>
    <ParamField path="speakerVoice" type="string">音声タイプ。デフォルトは `en_female_anna_mars_bigtts`。環境変数: `VOLCENGINE_TTS_VOICE`。レガシーエイリアス: `voice`。</ParamField>
    <ParamField path="speedRatio" type="number">プロバイダー固有の速度比率、`0.2..3`。</ParamField>
    <ParamField path="emotion" type="string">プロバイダー固有の感情タグ。</ParamField>
    <ParamField path="appId / token / cluster" type="string" deprecated>レガシーの Volcengine Speech Console フィールド。環境変数: `VOLCENGINE_TTS_APPID`、`VOLCENGINE_TTS_TOKEN`、`VOLCENGINE_TTS_CLUSTER`（デフォルトは `volcano_tts`）。</ParamField>
  </Accordion>

  <Accordion title="xAI">
    <ParamField path="apiKey" type="string">環境変数: `XAI_API_KEY`。</ParamField>
    <ParamField path="baseUrl" type="string">デフォルトは `https://api.x.ai/v1`。環境変数: `XAI_BASE_URL`。</ParamField>
    <ParamField path="speakerVoiceId" type="string">デフォルトは `eve`。認証がある場合、`openclaw infer tts voices --provider xai` は現在の組み込みカタログを取得します。認証がない場合は、オフラインフォールバック `ara`、`eve`、`leo`、`rex`、`sal` を一覧表示します。アカウントのカスタム音声 ID は、組み込みリストに存在しない場合でも転送されます。レガシーエイリアス: `voiceId`。</ParamField>
    <ParamField path="language" type="string">BCP-47 言語コードまたは `auto`。デフォルトは `en`。</ParamField>
    <ParamField path="responseFormat" type='"mp3" | "wav" | "pcm" | "mulaw" | "alaw"'>デフォルトは `mp3`。</ParamField>
    <ParamField path="speed" type="number">プロバイダー固有の速度オーバーライド、`0.7..1.5`。</ParamField>
  </Accordion>

  <Accordion title="Xiaomi MiMo">
    <ParamField path="apiKey" type="string">環境変数: `XIAOMI_API_KEY`。</ParamField>
    <ParamField path="baseUrl" type="string">デフォルトは `https://api.xiaomimimo.com/v1`。環境変数: `XIAOMI_BASE_URL`。</ParamField>
    <ParamField path="model" type="string">デフォルトは `mimo-v2.5-tts`。環境変数: `XIAOMI_TTS_MODEL`。`mimo-v2.5-tts-voicedesign` もサポートします。</ParamField>
    <ParamField path="speakerVoice" type="string">プリセット音声モデルのデフォルトは `mimo_default`。環境変数: `XIAOMI_TTS_VOICE`。レガシーエイリアス: `voice`。`mimo-v2.5-tts-voicedesign` には送信されません。</ParamField>
    <ParamField path="format" type='"mp3" | "wav"'>デフォルトは `mp3`。環境変数: `XIAOMI_TTS_FORMAT`。</ParamField>
    <ParamField path="style" type="string">ユーザーメッセージとして送信される任意の自然言語スタイル指示。読み上げられません。`mimo-v2.5-tts-voicedesign` では、これは音声デザインプロンプトです。省略すると OpenClaw がデフォルトを提供します。</ParamField>
  </Accordion>
</AccordionGroup>

## エージェントツール

`tts` ツールはテキストを音声に変換し、返信配信用の音声添付ファイルを返します。
Feishu、Matrix、Telegram、WhatsApp では、音声はファイル添付ではなく
音声メッセージとして配信されます。この経路では `ffmpeg` が
利用可能な場合、Feishu と WhatsApp は Opus 以外の TTS 出力をトランスコードできます。

WhatsApp は Baileys を介して音声を PTT ボイスノート（`audio` と
`ptt: true`）として送信します。また、クライアントがボイスノートのキャプションを
一貫して表示しないため、表示テキストを PTT 音声とは**別に**送信します。

このツールは任意の `channel` フィールドと `timeoutMs` フィールドを受け付けます。`timeoutMs` は
呼び出しごとのプロバイダーリクエストタイムアウト（ミリ秒）です。呼び出しごとの値は
`tts.timeoutMs` をオーバーライドします。設定された TTS タイムアウトは、Plugin が指定した
プロバイダーのデフォルト値をオーバーライドします。

## Gateway RPC

| メソッド            | 目的                                      |
| ----------------- | -------------------------------------------- |
| `tts.status`      | 現在の TTS 状態と前回の試行を読み取ります。     |
| `tts.enable`      | ローカルの自動設定を `always` に設定します。       |
| `tts.disable`     | ローカルの自動設定を `off` に設定します。          |
| `tts.convert`     | 1 回限りでテキストを音声に変換します。                        |
| `tts.setProvider` | ローカルのプロバイダー設定を指定します。               |
| `tts.personas`    | 設定済みのペルソナと現在有効なペルソナを一覧表示します。 |
| `tts.setPersona`  | ローカルのペルソナ設定を指定します。                |
| `tts.providers`   | 設定済みのプロバイダーとその状態を一覧表示します。        |

## サービスへのリンク

- [OpenAI テキスト読み上げガイド](https://platform.openai.com/docs/guides/text-to-speech)
- [OpenAI Audio API リファレンス](https://platform.openai.com/docs/api-reference/audio)
- [Azure Speech REST テキスト読み上げ](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech)
- [Azure Speech プロバイダー](/ja-JP/providers/azure-speech)
- [ElevenLabs テキスト読み上げ](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [ElevenLabs 認証](https://elevenlabs.io/docs/api-reference/authentication)
- [Gradium](/ja-JP/providers/gradium)
- [Inworld TTS API](https://docs.inworld.ai/tts/tts)
- [MiniMax T2A v2 API](https://platform.minimaxi.com/document/T2A%20V2)
- [Volcengine TTS HTTP API](/ja-JP/providers/volcengine#text-to-speech)
- [Xiaomi MiMo 音声合成](/ja-JP/providers/xiaomi#text-to-speech)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [Microsoft Speech の出力形式](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)
- [xAI テキスト読み上げ](https://docs.x.ai/developers/rest-api-reference/inference/voice#text-to-speech-rest)

## 関連項目

- [メディアの概要](/ja-JP/tools/media-overview)
- [音楽生成](/ja-JP/tools/music-generation)
- [動画生成](/ja-JP/tools/video-generation)
- [スラッシュコマンド](/ja-JP/tools/slash-commands)
- [音声通話 Plugin](/ja-JP/plugins/voice-call)
