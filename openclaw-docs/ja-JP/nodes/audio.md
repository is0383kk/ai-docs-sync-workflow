---
read_when:
    - 音声文字起こしまたはメディア処理の変更
summary: 受信した音声／ボイスメモをダウンロード、文字起こしし、返信に挿入する仕組み
title: 音声とボイスメモ
x-i18n:
    generated_at: "2026-07-26T09:08:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4076e3e55eb5c6dcc94cfdd842619697c8d756b924956d7b266d18446b4dd9be
    source_path: nodes/audio.md
    workflow: 16
---

## 機能

音声理解が有効（または自動検出）になっている場合、OpenClaw は次の処理を行います。

1. 最初の音声添付ファイル（ローカルパスまたは URL）を特定し、必要に応じてダウンロードします。
2. 各モデルエントリへ送信する前に `maxBytes` を適用します。
3. 対象となる最初のモデルエントリを順番に実行します（プロバイダーまたは CLI）。エントリが失敗またはスキップされた場合（サイズ超過／タイムアウト）、次のエントリを試行します。
4. 成功すると、`Body` を `[Audio]` ブロックに置き換え、`{{Transcript}}` を設定します。

文字起こしに成功すると、スラッシュコマンドが引き続き機能するように、`CommandBody`/`RawBody` にも文字起こしが設定されます。`--verbose` を使用すると、文字起こしの実行時と本文の置換時にログが表示されます。

## 自動検出（デフォルト）

モデルを設定しておらず、`tools.media.audio.enabled` が `false` でない場合、OpenClaw は次の順序で自動検出し、最初に動作した選択肢で停止します。

1. **アクティブな返信モデル**（そのプロバイダーが音声理解をサポートしている場合）。
2. **設定済みのプロバイダー認証** — 音声文字起こしをサポートするプロバイダーで認証を利用できる任意の `models.providers.*` エントリ。これはローカル CLI より先に確認されるため、設定済みの API キーは常に `PATH` 上のローカルバイナリより優先されます。
   複数設定されている場合のプロバイダー優先順位：Groq、OpenAI、xAI、Deepgram、Google、SenseAudio、ElevenLabs、Mistral。
3. **ローカル CLI**（プロバイダー認証を解決できない場合のみ）。OpenClaw は順序付きのフォールバックリストを構築します。
   - `whisper-cli`。現在のプロセスにおける以前のモデル呼び出しで Metal または CUDA が確認された場合に限り、CPU のデフォルトより先に使用されます
   - `sherpa-onnx-offline` をデフォルトの CPU プロバイダーで使用（`tokens.txt`、`encoder.onnx`、`decoder.onnx`、`joiner.onnx` を含む `SHERPA_ONNX_MODEL_DIR` が必要）
   - Metal/CUDA がビルド可能であることだけが判明している場合、または選択したバックエンドがほかの方法では確認されていない場合は `whisper-cli`
   - Apple Silicon では `parakeet-mlx`（MLX 対応。デバイスの使用状況は未確認のまま）
   - `whisper`（Python CLI。モデルを自動的にダウンロード）

インストール元／リンク元の情報は機能の証拠であり、実行の証拠ではありません。それだけで候補が CPU sherpa より先に移動することはありません。OpenClaw はバックエンドを調査するためだけに、セットアップ時やステータス確認時にモデルを読み込みません。
自動検出された whisper.cpp では通常のモデル実行ログが有効なままになるため、OpenClaw はアップストリームの `using … backend` 行を記録できます。明示的な CLI エントリでは、設定された出力フラグが維持されます。

メディア理解用の Gemini CLI 自動検出は、画像／動画向けのサンドボックス化された Antigravity CLI（`agy`）フォールバックに置き換えられました。音声では、上記のローカルバイナリ以外の CLI フォールバックは使用されません。

自動検出を無効にするには、`tools.media.audio.enabled: false` を設定します。カスタマイズするには、機能タグ付きのエントリを `tools.media.models` に追加します。

<Note>
バイナリ検出は macOS/Linux/Windows の各環境でベストエフォートで行われます。CLI が `PATH` 上にあることを確認するか（`~` は展開されます）、完全なコマンドパスを指定した明示的な CLI モデルを設定してください。
</Note>

音声を文字起こしせずにローカルの選択結果を確認するには、次を実行します。

```bash
openclaw capability audio providers
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

プロバイダー一覧には、グローバルなプロバイダー選択とは別に、ローカルフォールバックの選択結果に加え、対応可能、要求済み、確認済みの各バックエンドフィールドが表示されます。文字起こしの実行後、`/status` はメディア行に要求済みまたは確認済みのバックエンドを表示します。音声対応が明示された `tools.media.models` CLI エントリでは引き続き自動選択がバイパスされます。sherpa の `--provider=cuda` や whisper.cpp の `--no-gpu`/`--device` など、バックエンド固有のフラグを使用してください。

## 設定例

### プロバイダー + CLI フォールバック（OpenAI + Whisper CLI）

```json5
{
  tools: {
    media: {
      models: [
        { provider: "openai", model: "gpt-4o-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          timeoutSeconds: 45,
          capabilities: ["audio"],
        },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-transcribe" },
    },
  },
}
```

### プロバイダーのみ（Deepgram）

```json5
{
  tools: {
    media: {
      models: [{ provider: "deepgram", model: "nova-3", capabilities: ["audio"] }],
      audio: { enabled: true },
    },
  },
}
```

### プロバイダーのみ（Mistral Voxtral）

```json5
{
  tools: {
    media: {
      models: [{ provider: "mistral", model: "voxtral-mini-latest", capabilities: ["audio"] }],
      audio: { enabled: true },
    },
  },
}
```

### プロバイダーのみ（SenseAudio）

```json5
{
  tools: {
    media: {
      models: [
        {
          provider: "senseaudio",
          model: "senseaudio-asr-pro-1.5-260319",
          capabilities: ["audio"],
        },
      ],
      audio: { enabled: true },
    },
  },
}
```

### 文字起こしをチャットにエコー（オプトイン）

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        echoTranscript: true,
        echoFormat: '📝 "{transcript}"',
      },
    },
  },
}
```

## 注意事項と制限

- プロバイダー認証は標準のモデル認証順序（認証プロファイル、環境変数、`models.providers.*.apiKey`）に従います。
- Groq のセットアップ詳細：[Groq](/ja-JP/providers/groq)。
- `provider: "deepgram"` を使用する場合、Deepgram は `DEEPGRAM_API_KEY` を取得します。セットアップ詳細：[Deepgram](/ja-JP/providers/deepgram)。
- Mistral のセットアップ詳細：[Mistral](/ja-JP/providers/mistral)。
- `provider: "senseaudio"` を使用する場合、SenseAudio は `SENSEAUDIO_API_KEY` を取得します。セットアップ詳細：[SenseAudio](/ja-JP/providers/senseaudio)。
- 音声プロバイダーでは、`tools.media.audio` 配下のデフォルト値を使用するか、各 `tools.media.models[]` エントリで `baseUrl`、`headers`、`providerOptions`、および制限を上書きできます。
- 組み込みの音声サイズ上限は 20MB です。エントリ単位の `maxBytes` 上書きで変更できます。上限を超える音声はそのモデルではスキップされ、次のエントリが試行されます。
- 1024 バイト未満の音声ファイルは、プロバイダー／CLI による文字起こしの前にスキップされます。
- 音声のデフォルト `maxChars` は**未設定**です（文字起こし全文）。出力を切り詰めるには、`tools.media.audio.maxChars` またはエントリ単位の `maxChars` を設定します。
- OpenAI の自動検出デフォルトは `gpt-4o-transcribe` です。より安価で高速な選択肢には `model: "gpt-4o-mini-transcribe"` を設定します。
- 文字起こしは、テンプレート内で `{{Transcript}}` として利用できます。
- `tools.media.audio.echoTranscript` はデフォルトでオフです。`echoFormat` では `{transcript}` プレースホルダーを使用できます。
- CLI の stdout は 5MB に制限されます。CLI の出力は簡潔にしてください。
- CLI の `args` では、ローカル音声ファイルのパスに `{{AttachmentPath}}` を使用する必要があります。以前の `audio.transcription.command` 設定にある非推奨の `{input}` プレースホルダーを移行するには、`openclaw doctor --fix` を実行します（廃止済みキー：`audio.transcription`、後継：`tools.media.models`）。`{{MediaPath}}` は非推奨の互換性エイリアスとして残っています。
- `tools.media.concurrency` はメディアタスクを制限します。GPU スケジューラーではありません。

### 常駐ローカル STT

自動検出されたローカル STT は、引き続きリクエストごとにプロセスを起動します。標準の Homebrew `whisper-cpp` パッケージではサーバーが無効化されており、アップストリームの例には設定済みの有界受付キューがないため、OpenClaw は現在、常駐 whisper.cpp サーバーを管理しません。Plugin が所有する常駐ライフサイクルを安全に有効化するには、正常性確認／起動、モデル常駐、有界キューイング、キャンセル／タイムアウト、local loopback のみに限定した認証なしの動作、クラウドフォールバックなしを備えた、保守されているパッケージ化済みワーカーが必要です。

### プロキシ環境のサポート

プロバイダーによる音声文字起こしでは、undici の `EnvHttpProxyAgent` セマンティクスに従い、標準の送信プロキシ環境変数が使用されます。

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `ALL_PROXY` / `all_proxy`

小文字の変数は大文字の変数より優先されます。`NO_PROXY`/`no_proxy` のエントリ（ホスト名、`*.suffix`、または `host:port`）はプロキシをバイパスします。プロキシ環境変数が設定されていない場合は、直接外部接続を使用します。プロキシの設定に失敗した場合（不正な形式の URL）、OpenClaw は警告をログに記録し、直接フェッチへフォールバックします。

## グループでのメンション検出

音声の事前処理をサポートするチャンネルでは、グループチャットに `requireMention: true` が設定されている場合、OpenClaw はメンションを確認する**前に**音声を文字起こしします。これにより、キャプションのないボイスメモでも、その文字起こしに設定済みのメンションパターンが含まれていればメンションゲートを通過できます。入力されたメンションが必要なトランスポートについては、チャンネル固有のドキュメントで説明されています。

**動作の仕組み：**

1. 音声メッセージにテキスト本文がなく、グループでメンションが必須の場合、OpenClaw は最初の音声添付ファイルを事前に文字起こしします。
2. 文字起こしにメンションパターン（例：`@BotName`、絵文字トリガー）が含まれているか確認します。
3. メンションが見つかると、メッセージは完全な返信パイプラインへ進みます。

**フォールバック動作：**事前文字起こしに失敗した場合（タイムアウト、API エラーなど）、メッセージはテキストのみのメンション検出へフォールバックするため、混合メッセージ（テキスト + 音声）が破棄されることはありません。

**Telegram のグループ／トピック単位でオプトアウト：**

- そのグループで事前文字起こしによるメンション確認をスキップするには、`channels.telegram.groups.<chatId>.disableAudioPreflight: true` を設定します。
- トピック単位で上書きするには、`channels.telegram.groups.<chatId>.topics.<threadId>.disableAudioPreflight` を設定します（スキップするには `true`、強制的に有効化するには `false`）。
- デフォルトは `false` です（メンションゲートの条件に一致した場合、事前処理が有効になります）。

**例：**`requireMention: true` が設定された Telegram グループで、ユーザーが「ねえ @Claude、天気はどう？」と言うボイスメモを送信します。ボイスメモが文字起こしされ、メンションが検出されると、エージェントが返信します。

## 注意点

- スコープルールでは最初に一致したものが優先されます。`chatType` は `direct`、`group`、または `channel` に正規化されます。
- CLI が終了コード 0 で終了し、プレーンテキストを出力することを確認してください。JSON 出力は `jq -r .text` を介して加工する必要があります。
- 既知のファイル出力モードが優先されます。推定された文字起こしファイルが空または存在しない場合、CLI の進行状況出力へフォールバックせず、文字起こしなしとなります。
- `parakeet-mlx` では、`--output-dir` およびデフォルトの `{filename}` 出力テンプレートとともに `--output-format txt`（または `all`）を使用します。アップストリームの `PARAKEET_OUTPUT_FORMAT` および `PARAKEET_OUTPUT_TEMPLATE` 環境変数も使用されます。OpenClaw は `<output-dir>/<media-basename>.txt` を読み取ります。デフォルトの `srt` 形式、その他の形式、カスタム出力テンプレートでは、引き続き stdout が使用されます。
- 返信キューのブロックを避けるため、タイムアウト（`timeoutSeconds`、デフォルト 60s）は適切な値にしてください。
- 事前文字起こしでは、メンション検出用に**最初の**音声添付ファイルだけを処理します。追加の音声添付ファイルは、メインのメディア理解フェーズで処理されます。

## 関連項目

- [メディア理解](/ja-JP/nodes/media-understanding)
- [トークモード](/ja-JP/nodes/talk)
- [音声ウェイク](/ja-JP/nodes/voicewake)
