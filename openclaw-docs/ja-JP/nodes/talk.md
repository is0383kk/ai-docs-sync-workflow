---
read_when:
    - macOS/iOS/Android でのトークモードの実装
    - 音声／TTS／割り込み動作の変更
summary: トークモード：ローカル STT/TTS とリアルタイム音声による連続的な音声会話
title: トークモード
x-i18n:
    generated_at: "2026-07-26T09:07:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b21319eee169ba898331f87279a2b2a5170441131a1e9cdc85c15b268d165e21
    source_path: nodes/talk.md
    workflow: 16
---

Talk モードには 5 つのランタイム形態があります。

- **ネイティブ macOS/iOS/Android Talk**：ネイティブ音声認識、Gateway チャット、`talk.speak` TTS を使用します。macOS/iOS の Apple Speech 認識ではネットワークサービスが使用される場合があります。Android の動作は、インストールされている音声サービスによって異なります。Node は `talk` ケイパビリティを公開し、サポートする `talk.*` コマンドを宣言します。
- **iOS Talk（リアルタイム）**：`webrtc` トランスポートを選択するか、トランスポートを省略した OpenAI リアルタイム設定では、クライアント所有の WebRTC を使用します。明示的な `gateway-relay`、`provider-websocket`、および OpenAI 以外のリアルタイム設定では、引き続き Gateway 所有のリレーを使用します。非リアルタイム設定では、ネイティブ音声ループを使用します。
- **ブラウザー Talk**：クライアント所有の `webrtc`/`provider-websocket` セッションには `talk.client.create` を、Gateway 所有の `gateway-relay` セッションには `talk.session.create` を使用します。`managed-room` は Gateway ハンドオフおよびトランシーバールーム用に予約されています。
- **Android Talk（リアルタイム）**：`talk.realtime.mode: "realtime"` と `talk.realtime.transport: "gateway-relay"` でオプトインします。それ以外の場合、Android はネイティブ音声認識、Gateway チャット、`talk.speak` を引き続き使用します。
- **文字起こし専用クライアント**：アシスタントの音声応答を伴わない字幕表示やディクテーションには、`talk.session.create({ mode: "transcription", transport: "gateway-relay", brain: "none" })`、続いて `talk.session.appendAudio`、`talk.session.cancelTurn`、`talk.session.close` を使用します。単発でアップロードされたボイスメモでは、引き続き [メディア理解](/ja-JP/nodes/media-understanding) の音声パスを使用します。

ネイティブ Talk は継続的なループです。音声を待ち受け、アクティブなセッションを通じて文字起こしをモデルに送信し、応答を待ってから、設定済みの Talk プロバイダー（`talk.speak`）を介して読み上げます。

クライアント所有のリアルタイム Talk は、`chat.send` を直接呼び出す代わりに、プロバイダーのツール呼び出しを `talk.client.toolCall` 経由で転送します。リアルタイム相談がアクティブな間、クライアントは `talk.client.steer` または `talk.session.steer` を呼び出して、音声入力を `status`、`steer`、`cancel`、または `followup` に分類できます。受理されたステアリングはアクティブな埋め込み実行のキューに追加され、拒否されたステアリングでは `no_active_run`、`not_streaming`、`compacting` などの理由が返されます。

確定したリアルタイムのユーザー発話とアシスタント発話は、常にアクティブなエージェントセッションへリアルタイムで追加されるため、以降のチャットと音声ターンは 1 つの履歴を共有します。クライアント所有のトランスポートは、安定したエントリ ID とともに確定済みの文字起こしを報告します。Gateway リレーセッションは同じイベントをサーバー側で追加します。プロバイダーセッションには、Discord 音声で使用される、サイズ制限付きのリアルタイムプロファイルコンテキストも渡されます。

音声から開始された相談実行では、メッセージの送信、Node の制御、ブラウザーやコンピューターの操作、サービスの変更、破壊的なシェルコマンド、公開など、影響の大きいアクションの前に、新たに正確な音声確認が必要です。この確認はブロックされたツールの正確な引数にのみ適用され、一度だけ使用されます。無関係な同時実行には影響しません。通話が終了すると、OpenClaw は変更を行うツールについて簡潔な **音声通話の変更内容** ダイジェストを、セッションで最後に使用された WebChat 以外の配信先へ送信できます。

文字起こし専用 Talk は、リアルタイムセッションおよび STT/TTS セッションと同じ Talk イベントエンベロープを生成しますが、`mode: "transcription"` と `brain: "none"` を使用します。すべての Talk セッションは `talk.event` チャンネルでイベントをブロードキャストします。クライアントは、部分的または確定済みの文字起こし更新（`transcript.delta`/`transcript.done`）やその他のセッションテレメトリを受信するために、このチャンネルを購読します。

ブラウザー Video Talk は、OpenAI Realtime WebRTC および Google Live
プロバイダー WebSocket セッションで利用できます。OpenAI は、
`describe_view` が視覚コンテキストを要求したときに、サイズ制限付きの JPEG を 1 枚受信します。
継続的なカメラトラックは受信しません。Google Live は、ブラウザーから
最大毎秒 1 フレームでサイズ制限付きの JPEG フレームを直接受信し、`describe_view` は
カメラストリームの状態を報告します。どちらの場合も、カメラフレームは Gateway を経由せず、
Talk を停止するとカメラとマイクのトラックが解放されます。

## 動作（macOS）

- Talk モードが有効な間は、オーバーレイが常時表示されます。
- **リスニング &rarr; 思考中 &rarr; 発話中** のフェーズで遷移します。
- 短い休止（無音時間）があると、現在の文字起こしが送信されます。
- 応答は WebChat に書き込まれます（入力した場合と同様です）。
- **発話による割り込み**（デフォルトで有効）：アシスタントの発話中にユーザーが話すと、再生が停止し、次のプロンプト用に割り込みのタイムスタンプが記録されます。

## 応答内の音声ディレクティブ

アシスタントは、音声を制御するために応答の先頭へ JSON 行を 1 行追加できます。

```json
{ "voice": "<voice-id>", "once": true }
```

ルール：

- 空でない最初の行でのみ使用できます。JSON 行は TTS 再生前に削除されます。
- 不明なキーは無視されます。
- `once: true` は現在の応答にのみ適用されます。これがない場合、その音声が新しい Talk モードのデフォルトになります。

サポートされるキー：`voice` / `voice_id` / `voiceId`、`model` / `model_id` / `modelId`、`speed`、`rate`（WPM）、`stability`、`similarity`、`style`、`speakerBoost`、`seed`、`normalize`、`lang`、`output_format`、`latency_tier`、`once`。

## 設定（`~/.openclaw/openclaw.json`）

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "elevenlabs_voice_id",
        modelId: "eleven_v3",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
    realtime: {
      provider: "openai",
      providers: {
        openai: {
          apiKey: "openai_api_key",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
      instructions: "Speak warmly and keep answers brief.",
      mode: "realtime",
      transport: "webrtc",
      brain: "agent-consult",
    },
  },
}
```

| キー                                      | デフォルト                                    | 注記                                                                                                                                                                                                                                                                      |
| ---------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`                               | -                                          | アクティブな Talk TTS プロバイダー。macOS ローカル再生パスには `elevenlabs`、`mlx`、または `system` を使用します。                                                                                                                                                                             |
| `providers.<id>.voiceId`                 | -                                          | ElevenLabs は `ELEVENLABS_VOICE_ID` / `SAG_VOICE_ID` にフォールバックし、それらがない場合は API キーを持つ最初の利用可能な音声にフォールバックします。                                                                                                                                                             |
| `speechLocale`                           | デバイスのデフォルト                             | Android、iOS、および macOS のネイティブ音声認識で使用する BCP 47 ロケール。Apple Speech はネットワークサービスを使用する場合があります。Android は言語コンポーネントをリアルタイム入力文字起こしにも転送します。                                                                                  |
| `providers.elevenlabs.modelId`           | `eleven_v3`                                |                                                                                                                                                                                                                                                                            |
| `providers.mlx.modelId`                  | `mlx-community/Soprano-80M-bf16`           |                                                                                                                                                                                                                                                                            |
| `providers.elevenlabs.apiKey`            | -                                          | `ELEVENLABS_API_KEY` にフォールバックします（利用可能な場合は Gateway シェルプロファイルにもフォールバックします）。                                                                                                                                                                                                |
| `silenceTimeoutMs`                       | macOS/Android は `700` ms、iOS は `900` ms       | Talk が文字起こしを送信する前の一時停止時間。                                                                                                                                                                                                                             |
| `interruptOnSpeech`                      | `true`                                     |                                                                                                                                                                                                                                                                            |
| `outputFormat`                           | macOS/iOS は `pcm_44100`、Android は `pcm_24000` | MP3 ストリーミングを強制するには `mp3_*` を設定します。                                                                                                                                                                                                                                        |
| `consultThinkingLevel`                   | 未設定                                      | リアルタイム `openclaw_agent_consult` 呼び出しの背後で実行されるエージェントの思考レベルのオーバーライド。                                                                                                                                                                                  |
| `consultFastMode`                        | 未設定                                      | リアルタイム `openclaw_agent_consult` 呼び出しの高速モードのオーバーライド。                                                                                                                                                                                                            |
| `realtime.provider`                      | -                                          | WebRTC には `openai`、プロバイダー WebSocket には `google`、または Gateway リレー経由のブリッジ専用プロバイダーを使用します。                                                                                                                                                                     |
| `realtime.providers.<id>`                | -                                          | プロバイダーが所有するリアルタイム設定。ブラウザーが受け取るのは一時的または制約付きのセッション認証情報のみであり、標準の API キーを受け取ることはありません。                                                                                                                                                 |
| `realtime.providers.openai.speakerVoice` | `alloy`                                    | 組み込みの OpenAI Realtime 音声 ID（古い `voice` キーも引き続き機能しますが、非推奨です）。現在の `gpt-realtime-2.1` 音声: `alloy`、`ash`、`ballad`、`cedar`、`coral`、`echo`、`marin`、`sage`、`shimmer`、`verse`。最高品質を得るには `marin` と `cedar` を推奨します。 |
| `realtime.transport`                     | -                                          | `webrtc`: iOS およびブラウザーでクライアントが所有する OpenAI WebRTC。`provider-websocket`: ブラウザーが所有し、iOS では Gateway リレーを使用し続けます。`gateway-relay`: プロバイダーの音声を Gateway 上に保持します。Android でリアルタイムを使用できるのはこのトランスポートのみです。                                  |
| `realtime.brain`                         | -                                          | `agent-consult` はリアルタイムツール呼び出しを Gateway ポリシー経由でルーティングします。`direct-tools` は従来の直接ツール互換性用です。`none` は文字起こしや外部オーケストレーション用です。                                                                                                 |
| `realtime.consultRouting`                | -                                          | `provider-direct` は、プロバイダーが `openclaw_agent_consult` をスキップした場合にその直接応答を保持します。代わりに `force-agent-consult` は確定したユーザーの文字起こしを OpenClaw 経由でルーティングします。                                                                                          |
| `realtime.instructions`                  | -                                          | OpenClaw の組み込みリアルタイムプロンプトに、プロバイダー向けのシステム指示（音声のスタイルやトーン）を追加します。デフォルトの `openclaw_agent_consult` ガイダンスは維持されます。                                                                                                                |

`talk.catalog` は、正規のプロバイダー ID とレジストリエイリアス、各プロバイダーで有効なモード、トランスポート、ブレイン戦略、リアルタイム音声形式、ケイパビリティフラグ、およびランタイムが選択した準備状態の結果を公開します。ファーストパーティの Talk クライアントは、プロバイダーエイリアスをローカルで管理せず、このカタログを読み取る必要があります。グループの準備状態を省略する古い Gateway は、明確に未設定と判断せず、未検証として扱ってください。ストリーミング文字起こしプロバイダーは `talk.catalog.transcription` を通じて検出されます。専用の Talk 文字起こし設定サーフェスが提供されるまでは、現在の Gateway リレーは Voice Call ストリーミングプロバイダー設定を使用します。

## macOS UI

- メニューバーの切り替え: **Talk**
- 設定タブ: **Talk Mode** グループ（音声 ID と割り込みの切り替え）
- オーバーレイ: オーブには共通の Talk 波形が表示されます（iOS、watchOS、Android と共有）。リスニング中はライブのマイクレベルに追従し、発話中は実際の TTS 再生エンベロープに追従し、思考中は静かに明滅します。オーブをクリックすると一時停止または再開し、ダブルクリックすると発話を停止し、X をクリックすると Talk モードを終了します。

## Android UI

- Android のメインナビゲーションは **Home**、**Chat**、**Settings** です。音声入力は
  独立した Voice タブではなく、Chat のコンポーザーにあります。
- デバイス上での音声入力にはコンポーザーのマイクをタップします。長押しすると
  ボイスメモの添付ファイルを録音します。継続的な Talk は Talk 波形から開始します。
- 音声入力、ボイスメモ録音、Talk は相互排他的なマイク
  パスです。いずれかを開始すると、ほかは停止またはブロックされます。
- リアルタイム Talk では、接続済みの Bluetooth Classic または BLE ヘッドセットの
  マイクが優先されます。切断された場合、アプリは別のヘッドセット入力を要求するか、
  デフォルトのマイクにフォールバックし、キャプチャが停止すると
  デフォルトの設定を復元します。
- アプリがフォアグラウンドから離れるか、ユーザーが Chat から離れると、
  音声入力とボイスメモ録音は停止します。
- Talk Mode は、オフに切り替えられるか Node が切断されるまで実行を継続し、アクティブな間は Android のマイク用フォアグラウンドサービスタイプを使用します。
- Android は、低遅延の `AudioTrack` ストリーミング向けに `pcm_16000`、`pcm_22050`、`pcm_24000`、`pcm_44100` の出力形式をサポートします。

## 注記

- 音声認識とマイクの権限が必要です。
- ネイティブ Talk はアクティブな Gateway セッションを使用し、応答イベントが利用できない場合にのみ履歴ポーリングへフォールバックします。
- Gateway は、アクティブな Talk プロバイダーを使用し、`talk.speak` を通じて Talk の再生を解決します。Android は、その RPC が利用できない場合にのみローカルシステム TTS へフォールバックします。
- macOS のローカル MLX 再生では、同梱の `openclaw-mlx-tts` ヘルパーが存在する場合はそれを使用し、存在しない場合は `PATH` 上の実行可能ファイルを使用します。開発中にカスタムヘルパーバイナリを指定するには `OPENCLAW_MLX_TTS_BIN` を設定します。
- 音声ディレクティブの値範囲（ElevenLabs）: `stability`、`similarity`、`style` は `0..1` を受け入れ、`speed` は `0.5..2` を受け入れ、`latency_tier` は `0..4` を受け入れます。

## 関連項目

- [音声ウェイク](/ja-JP/nodes/voicewake)
- [音声とボイスメモ](/ja-JP/nodes/audio)
- [メディア理解](/ja-JP/nodes/media-understanding)
