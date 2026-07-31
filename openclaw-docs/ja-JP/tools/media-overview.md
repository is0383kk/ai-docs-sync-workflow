---
read_when:
    - OpenClawのメディア機能の概要を探す
    - 設定するメディアプロバイダーの選択
    - 非同期メディア生成の仕組みを理解する
sidebarTitle: Media overview
summary: 画像、動画、音楽、音声、メディア理解機能の概要
title: メディアの概要
x-i18n:
    generated_at: "2026-07-26T10:33:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 18eb79e6915c5dc8d705bf5cadfcdddecaf7d21a037f102696d4f2bcd41e5bea
    source_path: tools/media-overview.md
    workflow: 16
---

OpenClaw は画像、動画、音楽を生成し、受信メディア
（画像、音声、動画）を理解し、テキスト読み上げで応答を音声化します。すべての
メディア機能はツール駆動です。エージェントは会話に基づいて使用するタイミングを
判断し、各ツールは少なくとも 1 つの基盤プロバイダーが設定されている場合にのみ
表示されます。

ライブ音声では、単発のメディアツール経路ではなく Talk セッション契約を使用します。
Talk には 3 つのモードがあります。プロバイダーネイティブの `realtime`、ローカルまたはストリーミングの
`stt-tts`、観察専用の音声キャプチャ用の `transcription` です。これらのモードは、
テレフォニー、会議、ブラウザーのリアルタイム機能、ネイティブのプッシュトゥトーククライアントと、
プロバイダーカタログ、イベントエンベロープ、キャンセルのセマンティクスを共有します。

## 機能

<CardGroup cols={2}>
  <Card title="画像生成" href="/ja-JP/tools/image-generation" icon="image">
    テキストプロンプトまたは参照画像から `image_generate` を介して
    画像を作成および編集します。チャットセッションでは非同期で、バックグラウンドで
    実行され、準備が整うと結果を投稿します。
  </Card>
  <Card title="動画生成" href="/ja-JP/tools/video-generation" icon="video">
    `video_generate` を介したテキストから動画、画像から動画、動画から動画への変換です。
    非同期で、バックグラウンドで実行され、準備が整うと結果を投稿します。
  </Card>
  <Card title="音楽生成" href="/ja-JP/tools/music-generation" icon="music">
    `music_generate` を介して音楽または音声トラックを生成します。チャット
    セッションでは、共有メディア生成タスクのライフサイクル上で非同期に実行されます。
  </Card>
  <Card title="テキスト読み上げ" href="/ja-JP/tools/tts" icon="microphone">
    `tts` ツールと `tts` 設定を使用して、
    送信応答を音声に変換します。同期処理です。
  </Card>
  <Card title="メディア理解" href="/ja-JP/nodes/media-understanding" icon="eye">
    視覚対応モデルプロバイダーと専用のメディア理解 Plugin を使用して、
    受信した画像、音声、動画を要約します。
  </Card>
  <Card title="音声テキスト変換" href="/ja-JP/nodes/audio" icon="ear-listen">
    バッチ STT または Voice Call のストリーミング STT プロバイダーを介して、
    受信した音声メッセージを文字起こしします。
  </Card>
</CardGroup>

## プロバイダー機能マトリックス

<Note>
この表は、専用のメディア生成、TTS、STT Plugin を対象としています。多くの
チャットモデルプロバイダー（Anthropic、Google、OpenAI など）も応答モデルを通じて
受信メディアを理解します。プロバイダーの完全な一覧については、
[メディア理解](/ja-JP/nodes/media-understanding#provider-support-matrix)を参照してください。
</Note>

| プロバイダー          | 画像 | 動画 | 音楽 | TTS | STT | リアルタイム音声 | メディア理解 |
| ----------------- | :---: | :---: | :---: | :-: | :-: | :------------: | :-----------------: |
| Alibaba           |       |   ✓   |       |     |     |                |                     |
| Azure Speech      |       |       |       |  ✓  |     |                |                     |
| BytePlus          |       |   ✓   |       |     |     |                |                     |
| ComfyUI           |   ✓   |   ✓   |   ✓   |     |     |                |                     |
| Deepgram          |       |       |       |     |  ✓  |                |                     |
| DeepInfra         |   ✓   |   ✓   |       |  ✓  |  ✓  |                |          ✓          |
| ElevenLabs        |       |       |       |  ✓  |  ✓  |                |                     |
| fal               |   ✓   |   ✓   |   ✓   |     |     |                |                     |
| Google            |   ✓   |   ✓   |   ✓   |  ✓  |  ✓  |       ✓        |          ✓          |
| Gradium           |       |       |       |  ✓  |     |                |                     |
| Inworld           |       |       |       |  ✓  |     |                |                     |
| LiteLLM           |   ✓   |       |       |     |     |                |                     |
| Local CLI         |       |       |       |  ✓  |     |                |                     |
| Microsoft         |       |       |       |  ✓  |     |                |                     |
| Microsoft Foundry |   ✓   |       |       |     |     |                |                     |
| MiniMax           |   ✓   |   ✓   |   ✓   |  ✓  |     |                |                     |
| Mistral           |       |       |       |     |  ✓  |                |                     |
| OpenAI            |   ✓   |   ✓   |       |  ✓  |  ✓  |       ✓        |          ✓          |
| OpenRouter        |   ✓   |   ✓   |   ✓   |  ✓  |  ✓  |                |          ✓          |
| PixVerse          |       |   ✓   |       |     |     |                |                     |
| Qwen              |       |   ✓   |       |     |     |                |          ✓          |
| Runway            |       |   ✓   |       |     |     |                |                     |
| SenseAudio        |       |       |       |     |  ✓  |                |                     |
| Together          |       |   ✓   |       |     |     |                |                     |
| Volcengine        |       |       |       |  ✓  |     |                |                     |
| Vydra             |   ✓   |   ✓   |       |  ✓  |     |                |                     |
| xAI               |   ✓   |   ✓   |       |  ✓  |  ✓  |                |          ✓          |
| Xiaomi MiMo       |       |       |       |  ✓  |     |                |                     |

<Note>
ここでの**リアルタイム音声**とは、プロバイダーネイティブの双方向リアルタイム機能
（Talk の `realtime` モード。例: Gemini Live または OpenAI Realtime API）を意味し、
現在登録しているのは Google と OpenAI のみです。Deepgram、ElevenLabs、Mistral、OpenAI、xAI は
これとは別に Voice Call のストリーミング STT（音声からテキストへの一方向変換）を登録しています。以下の
[音声テキスト変換と Voice Call](#speech-to-text-and-voice-call)を参照してください。
xAI のリアルタイム音声はアップストリームの機能ですが、共有リアルタイム音声契約で
表現できるようになるまで OpenClaw には登録されません。
</Note>

## 非同期と同期

| 機能     | モード         | 理由                                                                                                  |
| -------------- | ------------ | ---------------------------------------------------------------------------------------------------- |
| 画像          | 非同期 | プロバイダーの処理がチャットターンより長く続く可能性があり、生成された添付ファイルは共有完了経路を使用します。   |
| テキスト読み上げ | 同期  | プロバイダーの応答は数秒で返され、応答音声に添付されます。                                   |
| 動画          | 非同期 | プロバイダーの処理には 30 s から数分かかり、低速なキューは設定されたタイムアウトまで実行される場合があります。 |
| 音楽          | 非同期 | 動画と同じプロバイダー処理特性です。                                                    |

非同期ツールでは、OpenClaw はリクエストをプロバイダーに送信し、タスク
id をすぐに返して、タスク台帳でジョブを追跡します。ジョブの実行中も、エージェントは
他のメッセージへの応答を続けます。プロバイダーの処理が完了すると、
OpenClaw は生成されたメディアのパスとともにエージェントを起動し、セッションの通常の可視応答モードで
ユーザーに通知できるようにします。設定されている場合は最終応答を
自動配信し、セッションでメッセージツールが必要な場合は `message(action="send")` を使用します。
リクエスト元のセッションが非アクティブであるか、アクティブな起動に失敗し、
生成されたメディアの一部が完了応答に含まれていない場合、OpenClaw は
不足しているメディアのみを含む冪等な直接フォールバックを送信します。完了応答ですでに
配信されたメディアは再投稿されません。

## 音声テキスト変換と Voice Call

Deepgram、DeepInfra、ElevenLabs、Google、Groq、Mistral、OpenAI、OpenRouter、
SenseAudio、xAI は、設定されている場合、すべてバッチ
`tools.media.audio` 経路を通じて受信音声を文字起こしできます。メンションゲーティングや
コマンド解析のために音声メモを事前処理するチャンネル Plugin は、
文字起こし済みの添付ファイルを受信コンテキストに記録するため、共有メディア理解処理では
同じ音声に対して 2 回目の STT 呼び出しを行わず、その文字起こしを再利用します。

Deepgram、ElevenLabs、Mistral、OpenAI、xAI は Voice Call の
ストリーミング STT プロバイダーも登録しているため、録音の完了を待たずに
ライブ通話音声を選択したベンダーへ転送できます。

ライブのユーザー会話では、[Talk モード](/ja-JP/nodes/talk)を推奨します。バッチ音声
添付ファイルはメディア経路に留めます。ブラウザーのリアルタイム機能、ネイティブのプッシュトゥトーク、
テレフォニー、会議音声では、Talk イベントと Gateway が返すセッションスコープの
カタログを使用する必要があります。

## プロバイダーマッピング（各ベンダーの機能面への分割方法）

<AccordionGroup>
  <Accordion title="Google">
    画像、動画、音楽、バッチ TTS、バッチ STT、バックエンドのリアルタイム音声、
    メディア理解の各機能面を提供します。
  </Accordion>
  <Accordion title="OpenAI">
    画像、動画、バッチ TTS、バッチ STT、Voice Call のストリーミング STT、バックエンドの
    リアルタイム音声、メモリ埋め込みの各機能面を提供します。
  </Accordion>
  <Accordion title="DeepInfra">
    チャット／モデルルーティング、画像生成／編集、テキストから動画への変換、バッチ TTS、
    バッチ STT、画像メディア理解、メモリ埋め込みの各機能面を提供します。
    DeepInfra は再ランキング、分類、物体検出などの
    ネイティブモデルタイプも公開していますが、OpenClaw にはそれらのカテゴリ向けの
    プロバイダー契約がまだないため、この Plugin は登録しません。
  </Accordion>
  <Accordion title="xAI">
    画像、動画、検索、コード実行、バッチ TTS、バッチ STT、Voice
    Call のストリーミング STT を提供します。xAI のリアルタイム音声はアップストリームの機能ですが、
    共有リアルタイム音声契約で表現できるようになるまで
    OpenClaw には登録されません。
  </Accordion>
</AccordionGroup>

## 関連項目

- [画像生成](/ja-JP/tools/image-generation)
- [動画生成](/ja-JP/tools/video-generation)
- [音楽生成](/ja-JP/tools/music-generation)
- [テキスト読み上げ](/ja-JP/tools/tts)
- [メディア理解](/ja-JP/nodes/media-understanding)
- [音声 Node](/ja-JP/nodes/audio)
- [Talk モード](/ja-JP/nodes/talk)
