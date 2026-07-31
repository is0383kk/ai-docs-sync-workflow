---
read_when:
    - メディア理解の設計またはリファクタリング
    - 受信音声・動画・画像の前処理の調整
sidebarTitle: Media understanding
summary: プロバイダーと CLI のフォールバックによる受信画像・音声・動画の理解（任意）
title: メディア理解
x-i18n:
    generated_at: "2026-07-26T10:18:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 38e9a0f89607bb9c4af85689ef0fbd3df9234b36e06d86c129e0d823d6e05143
    source_path: nodes/media-understanding.md
    workflow: 16
---

OpenClaw は、応答パイプラインが実行される前に受信メディア（画像/音声/動画）を要約できるため、コマンド解析とルーティングでは生のバイト列ではなく短いテキストを使用できます。理解処理はローカルツールまたはプロバイダーキーを自動検出します。明示的なモデルを設定することもできます。元のメディアは通常どおり常にモデルへ渡されます。理解処理が失敗した場合や無効な場合も、応答フローは変更されずに続行されます。

ベンダー Plugin は、機能メタデータ（各プロバイダーが対応するメディアタイプ、デフォルトモデル、優先順位）を登録します。OpenClaw コアは、共有 `tools.media` 設定、フォールバック順序、応答パイプライン統合を所有します。

## 仕組み

<Steps>
  <Step title="添付ファイルを収集">
    順序付けされた受信メディア情報（`path`、`url`、`contentType`、`kind`）を収集します。
  </Step>
  <Step title="機能ごとに選択">
    有効な各機能（画像/音声/動画）について、`attachments` ポリシーに従って添付ファイルを選択します（デフォルト: 最初の添付ファイルのみ）。
  </Step>
  <Step title="モデルを選択">
    最初の適格なモデルエントリ（サイズ、機能、認証が利用可能）を選択します。
  </Step>
  <Step title="失敗時にフォールバック">
    モデルでエラーが発生した場合、タイムアウトした場合、またはメディアが `maxBytes` を超える場合は、次のエントリを試します。
  </Step>
  <Step title="成功時に適用">
    `Body` は `[Image]`、`[Audio]`、または `[Video]` ブロックになります。音声では `{{Transcript}}` も設定されます。キャプションテキストが存在する場合、コマンド解析ではそれを使用し、存在しない場合は文字起こしを使用します。キャプションはブロック内に `User text:` として保持されます。
  </Step>
</Steps>

## 設定

`tools.media` には、機能タグ付きモデルリストを 1 つと、機能ごとの小規模な制御設定を保持します。

```json5
{
  tools: {
    media: {
      concurrency: 2, // 同時実行する機能処理の最大数（デフォルト）
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["image", "video"] },
      ],
      image: { preferredModel: "google/gemini-3-flash-preview" },
      audio: { enabled: true },
      video: { enabled: true },
    },
  },
}
```

機能ごとの（`image`/`audio`/`video`）キー:

| キー              | 型      | デフォルト                                | 備考                                                                |
| ---------------- | --------- | -------------------------------------- | -------------------------------------------------------------------- |
| `enabled`        | `boolean` | 自動（`false` で無効化）                | この機能の自動検出を無効にするには `false` を設定します              |
| `preferredModel` | `string`  | 最初の互換エントリ                 | `provider/model`、モデル ID、`provider:<id>`、または `cli:command` を優先します |
| `prompt`         | `string`  | 機能のデフォルト                     | エントリで上書きされていない場合のデフォルトプロンプト                    |
| `maxChars`       | `number`  | 画像/動画は `500`、音声は未設定         | デフォルトの出力上限                                                 |
| `maxBytes`       | `number`  | 画像 10MB、音声 20MB、動画 50MB     | デフォルトの入力上限                                                  |
| `timeoutSeconds` | `number`  | 画像/音声は `60`、動画は `120`          | デフォルトのリクエストタイムアウト                                              |
| `language`       | `string`  | 未設定                                  | 音声文字起こしの言語ヒント                                             |
| `scope`          | object    | 未設定                                  | チャンネル/チャットタイプ/ソースキーによる制限                                 |
| `attachments`    | object    | `{ mode: "first", maxAttachments: 1 }` | 一致する添付ファイルのうち処理するものを選択                      |
| `echoTranscript` | `boolean` | `false`                                | 音声のみ: エージェント処理の前に文字起こしをエコー表示              |
| `echoFormat`     | `string`  | `'📝 "{transcript}"'`                  | 音声のみ: エコー表示する文字起こしの形式                         |

プロンプト、上限、言語ヒント、リクエストの上書き、プロバイダーオプションは、機能のデフォルトとして設定することも、個々の `tools.media.models[]` エントリで上書きすることもできます。明示的なモデルが設定されていない場合、機能のデフォルトは自動検出されたプロバイダーにも適用されます。

### モデルエントリ

各 `models[]` エントリは、**プロバイダー**エントリ（デフォルト）または **CLI** エントリです。

<Tabs>
  <Tab title="プロバイダーエントリ">
    ```json5
    {
      type: "provider", // 省略時のデフォルト
      provider: "openai",
      model: "gpt-5.6-sol",
      prompt: "画像を 500 文字以内で説明してください。",
      maxChars: 500,
      maxBytes: 10485760,
      timeoutSeconds: 60,
      capabilities: ["image"],
      profile: "vision-profile",
      preferredProfile: "vision-fallback",
    }
    ```
  </Tab>
  <Tab title="CLI エントリ">
    ```json5
    {
      type: "cli",
      command: "gemini",
      args: [
        "-m",
        "gemini-3-flash",
        "--allowed-tools",
        "read_file",
        "{{AttachmentPath}} のメディアを読み取り、{{MaxChars}} 文字以内で説明してください。",
      ],
      maxChars: 500,
      maxBytes: 52428800,
      timeoutSeconds: 120,
      capabilities: ["video", "image"],
    }
    ```

    CLI テンプレートでは、`{{AttachmentUrl}}`、`{{AttachmentContentType}}`、`{{AttachmentDir}}`、`{{AttachmentIndex}}`、`{{OutputDir}}`（この実行用に作成されるスクラッチディレクトリ）、および `{{OutputBase}}`（拡張子なしのスクラッチファイルのベースパス）も使用できます。旧名称の `{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}`、`{{MediaDir}}` は、非推奨の互換エイリアスとして残されています。

  </Tab>
</Tabs>

### プロバイダーの認証情報

プロバイダーによるメディア理解では、通常のモデル呼び出しと同じ認証解決順序（認証プロファイル、環境変数、`models.providers.<providerId>.apiKey`）を使用します。`tools.media.models[]` エントリでは、インラインの `apiKey` フィールドを使用できません。

```json5
{
  models: {
    providers: {
      openai: { apiKey: "<OPENAI_API_KEY>" },
      moonshot: { apiKey: "<MOONSHOT_API_KEY>" },
    },
  },
}
```

プロファイル、環境変数、カスタムベース URL については、[ツールとカスタムプロバイダー](/ja-JP/gateway/config-tools)を参照してください。

## ルールと動作

- `maxBytes` を超えるメディアでは、そのモデルをスキップして次のモデルを試します。
- 1024 バイト未満の音声ファイルは空または破損しているものとして扱われ、文字起こしの前にスキップされます。代わりに、エージェントには決定的なプレースホルダー文字起こしが渡されます。
- アクティブなプライマリ画像モデルがすでにビジョンをネイティブサポートしている場合、OpenClaw は `[Image]` 要約ブロックをスキップし、元の画像をモデルへ直接渡します。MiniMax は例外です。従来の MiniMax M2.x チャットメタデータが画像入力をサポートすると示している場合でも、`minimax`、`minimax-cn`、`minimax-portal`、`minimax-portal-cn` は常に、Plugin が所有する `MiniMax-VL-01` メディアプロバイダーを介して画像理解をルーティングします（`MiniMax-M3` 以降のみがビジョンをネイティブサポートすると見なされます）。
- Gateway/WebChat のプライマリモデルがテキスト専用の場合、画像添付ファイルはオフロードされた `media://inbound/*` 参照として保持されるため、添付ファイルが失われることなく、画像/PDF ツールまたは設定済みの画像モデルで引き続き検査できます。
- 明示的な `openclaw infer image describe --file <path> --model <provider/model>`（エイリアス: `openclaw capability image describe`）は、画像対応のプロバイダー/モデルを直接実行します。これには、一致する画像対応モデルが `models.providers.ollama.models[]` に設定されている場合の `ollama/qwen2.5vl:7b` などの Ollama 参照も含まれます。
- `<capability>.enabled` が `false` ではなく、モデルが設定されていない場合、OpenClaw はアクティブな応答モデルのプロバイダーがその機能をサポートしていれば、そのモデルを試します。

### 自動検出（デフォルト）

`tools.media.<capability>.enabled` が `false` ではなく、モデルが設定されていない場合、OpenClaw は次の順序で試し、最初に動作したオプションで停止します。

<Steps>
  <Step title="設定済み画像モデル（画像のみ）">
    アクティブな応答モデルがすでにビジョンをネイティブサポートしている場合を除き、`agents.defaults.imageModel` のプライマリ/フォールバック参照を使用します。`provider/model` 参照を優先します。修飾されていない参照は、一致が一意である場合に限り、設定済みの画像対応プロバイダーモデルエントリから修飾されます。
  </Step>
  <Step title="アクティブな応答モデル">
    プロバイダーがその機能をサポートしている場合、アクティブな応答モデルを使用します。
  </Step>
  <Step title="プロバイダー認証（音声のみ、ローカル CLI より前）">
    音声をサポートする設定済みの `models.providers.*` エントリを、ローカル CLI より先に試します。同梱プロバイダーの優先順位（同順位の場合はプロバイダー ID のアルファベット順）: Groq/OpenAI &rarr; xAI &rarr; Deepgram &rarr; OpenRouter &rarr; Google/SenseAudio &rarr; Deepinfra/ElevenLabs &rarr; Mistral。
  </Step>
  <Step title="ローカル CLI（音声のみ）">
    使用可能なローカルバイナリが、順序付けされたフォールバックリストになります。
    - 現在のプロセスで以前のモデル呼び出しにより Metal または CUDA が確認された後に限り、`whisper-cli` を最初に使用
    - CPU がデフォルトの `sherpa-onnx-offline`（`tokens.txt`/`encoder.onnx`/`decoder.onnx`/`joiner.onnx` を備えた `SHERPA_ONNX_MODEL_DIR` が必要）
    - アクセラレーションがビルド上対応しているだけか、未確認の場合は `whisper-cli`
    - Apple Silicon では `parakeet-mlx`（MLX 対応、デバイス使用は未確認）
    - `whisper`（Python CLI。デフォルトで `turbo` モデルを使用し、自動的にダウンロード）

    バックエンド機能の検査結果はキャッシュされ、モデルは読み込まれません。ビルド機能、要求されたバックエンドフラグ、実際の呼び出しで確認されたバックエンドは、個別に管理されます。自動検出された whisper.cpp ではモデル実行ログが有効のままとなり、上流で選択されたバックエンドを示す行を記録できます。明示的な CLI エントリでは、設定された順序、バックエンドフラグ、出力フラグが維持されます。

  </Step>
  <Step title="プロバイダー認証（画像/動画）">
    その機能をサポートする設定済みの `models.providers.*` エントリを、同梱のフォールバック順序より先に試します。画像対応モデルを持つ画像専用の設定プロバイダーは、同梱のベンダー Plugin でない場合でも、メディア理解用に自動登録されます。

    同梱プロバイダーの優先順位（同順位の場合はプロバイダー ID のアルファベット順）:
    - 画像: Anthropic/OpenAI &rarr; Google &rarr; MiniMax &rarr; Deepinfra &rarr; MiniMax Portal &rarr; Z.AI
    - 動画: Google &rarr; Qwen &rarr; Moonshot

  </Step>
  <Step title="Antigravity CLI（画像/動画のみ）">
    最初にインストールされている `agy` または `antigravity` バイナリ（`OPENCLAW_ANTIGRAVITY_CLI` で上書き可能）を、メディアのディレクトリに制限されたサンドボックス内で使用します。
  </Step>
</Steps>

機能の自動検出を無効にするには、次のように設定します。

```json5
{
  tools: {
    media: {
      audio: {
        enabled: false,
      },
    },
  },
}
```

<Note>
バイナリ検出は macOS/Linux/Windows 全体でベストエフォートです。CLI が `PATH` 上にあることを確認するか（`~` は展開されます）、完全なコマンドパスを指定した明示的な CLI モデルエントリを設定してください。
</Note>

### プロキシ対応（音声/動画プロバイダー呼び出し）

プロバイダーベースの**音声**および**動画**理解では、`NO_PROXY`/`no_proxy` のバイパス規則を含む標準の送信プロキシ環境変数（`HTTPS_PROXY`、`HTTP_PROXY`、`ALL_PROXY`、`https_proxy`、`http_proxy`、`all_proxy`）が適用されます。小文字の変数は大文字より優先されます。いずれも設定されていない場合、メディア理解は直接外部接続を使用します。プロキシ値の形式が不正な場合、OpenClaw は警告をログに記録し、直接取得にフォールバックします。画像理解では、このプロキシ経路は使用されません。

## 機能

`models[]` エントリの `capabilities` を設定すると、特定のメディアタイプに制限できます。共有リストの場合、OpenClaw は同梱プロバイダーごとにデフォルトを推定します。

| プロバイダー                                                                 | 機能          |
| ------------------------------------------------------------------------ | --------------------- |
| `openai`, `anthropic`, `minimax`                                         | 画像                 |
| `minimax-portal`                                                         | 画像                 |
| `moonshot`                                                               | 画像 + 動画         |
| `openrouter`                                                             | 画像 + 音声         |
| `google` (Gemini API)                                                    | 画像 + 音声 + 動画 |
| `qwen`                                                                   | 画像 + 動画         |
| `deepinfra`                                                              | 画像 + 音声         |
| `mistral`                                                                | 音声                 |
| `zai`                                                                    | 画像                 |
| `groq`, `xai`, `deepgram`, `senseaudio`                                  | 音声                 |
| 画像対応モデルを含む任意の `models.providers.<id>.models[]` カタログ | 画像                 |

CLI エントリでは、予期しない一致を避けるために `capabilities` を明示的に設定してください。省略すると、そのエントリは掲載されているすべての機能リストの候補になります。

## プロバイダー対応マトリクス

| 機能 | プロバイダー                                                                                                                                               | 注記                                                                                                                                                                                   |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 画像      | Anthropic, Codex app-server, Deepinfra, Google, MiniMax, MiniMax Portal, Moonshot, OpenAI, OpenAI Codex OAuth, OpenRouter, Qwen, Z.AI, 設定済みプロバイダー | ベンダー Plugin が画像対応を登録します。`openai/*` は API キーまたは Codex OAuth ルーティングを使用できます。`codex/*` は制限付きの Codex app-server ターンを使用します。画像対応の設定済みプロバイダーは自動登録されます。 |
| 音声      | Deepgram, Deepinfra, ElevenLabs, Google, Groq, Mistral, OpenAI, OpenRouter, SenseAudio, xAI                                                             | プロバイダーによる文字起こし（Whisper/Groq/xAI/Deepgram/OpenRouter STT/Gemini/SenseAudio/Scribe/Voxtral）。                                                                                     |
| 動画      | Google, Moonshot, Qwen                                                                                                                                  | ベンダー Plugin を介したプロバイダーの動画理解。Qwen の動画理解では標準の DashScope エンドポイントを使用します。                                                                        |

<Note>
**MiniMax に関する注記**: `minimax`、`minimax-cn`、`minimax-portal`、`minimax-portal-cn` の画像理解には、従来の MiniMax M2.x チャットメタデータが画像入力対応を示している場合でも、常に Plugin が所有する `MiniMax-VL-01` メディアプロバイダーが使用されます。
</Note>

## モデル選択のガイダンス

- 品質と安全性が重要な場合は、メディア機能ごとに現行世代で最も高性能なモデルを優先してください。
- 信頼できない入力を処理するツール対応エージェントでは、古い、または性能の低いメディアモデルを避けてください。
- 可用性を確保するため、機能ごとに少なくとも 1 つのフォールバック（高品質モデル + より高速または低コストのモデル）を用意してください。
- CLI フォールバック（`whisper-cli`、`whisper`、`gemini`）は、プロバイダー API が利用できない場合に役立ちます。
- 既知のファイル出力モードが優先されます。推定された文字起こしファイルが空または存在しない場合、CLI の進行状況出力にフォールバックせず、文字起こしは生成されません。
- `parakeet-mlx`: `--output-dir` およびデフォルトの `{filename}` 出力テンプレートとともに、`--output-format txt`（または `all`）を使用してください。上流の `PARAKEET_OUTPUT_FORMAT` および `PARAKEET_OUTPUT_TEMPLATE` 環境変数も適用されます。OpenClaw は `<output-dir>/<media-basename>.txt` を読み取ります。デフォルトの `srt` 形式、その他の形式、カスタム出力テンプレートでは、引き続き標準出力が使用されます。

## 添付ファイルポリシー

機能ごとの `attachments` で、処理する添付ファイルを制御します。

<ParamField path="mode" type='"first" | "all"' default="first">
  選択された最初の添付ファイルのみ、またはすべての添付ファイルを処理します。
</ParamField>
<ParamField path="maxAttachments" type="number" default="1">
  処理数に上限を設定します。
</ParamField>
<ParamField path="prefer" type='"first" | "last" | "path" | "url"'>
  添付ファイル候補から選択する際の優先条件です。
</ParamField>

`mode: "all"` の場合、出力には `[Image 1/2]`、`[Audio 2/2]` などのラベルが付けられます。

### 添付ファイルの抽出

- 抽出されたファイルテキストは、メディアプロンプトに追加される前に信頼できない外部コンテンツとしてラップされます。その際、`<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>` のような境界マーカーと `Source: External` メタデータ行が使用されます。
- この処理経路では、メディアプロンプトを短く保つため、長い `SECURITY NOTICE:` バナーを意図的に省略します。境界マーカーとメタデータは引き続き適用されます。
- 抽出可能なテキストがないファイルには `[No extractable text]` が設定されます。
- PDF がレンダリングされたページ画像にフォールバックした場合、OpenClaw はそれらの画像を画像認識対応の応答モデルに転送し、ファイルブロック内のプレースホルダー `[PDF content rendered to images]` を維持します。

## 設定例

<Tabs>
  <Tab title="共有モデル + オーバーライド">
    ```json5
    {
      tools: {
        media: {
          models: [
            { provider: "openai", model: "gpt-5.6-sol", capabilities: ["image"] },
            {
              provider: "google",
              model: "gemini-3-flash-preview",
              capabilities: ["image", "audio", "video"],
            },
            {
              type: "cli",
              command: "gemini",
              args: [
                "-m",
                "gemini-3-flash",
                "--allowed-tools",
                "read_file",
                "{{AttachmentPath}} のメディアを読み取り、{{MaxChars}} 文字以内で説明してください。",
              ],
              capabilities: ["image", "video"],
            },
          ],
          audio: {
            attachments: { mode: "all", maxAttachments: 2 },
          },
          video: {
            maxChars: 500,
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="音声 + 動画のみ">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [
              { provider: "openai", model: "gpt-4o-mini-transcribe" },
              {
                type: "cli",
                command: "whisper",
                args: ["--model", "base", "{{AttachmentPath}}"],
              },
            ],
          },
          video: {
            enabled: true,
            maxChars: 500,
            models: [
              { provider: "google", model: "gemini-3-flash-preview" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "{{AttachmentPath}} のメディアを読み取り、{{MaxChars}} 文字以内で説明してください。",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="画像のみ">
    ```json5
    {
      tools: {
        media: {
          image: {
            enabled: true,
            maxBytes: 10485760,
            maxChars: 500,
            models: [
              { provider: "openai", model: "gpt-5.6-sol" },
              { provider: "anthropic", model: "claude-opus-5" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "{{AttachmentPath}} のメディアを読み取り、{{MaxChars}} 文字以内で説明してください。",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="単一のマルチモーダルエントリ">
    ```json5
    {
      tools: {
        media: {
          image: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          audio: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          video: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## ステータス出力

メディア理解が実行されると、`/status` に機能ごとの概要行が含まれます。

```
📎 メディア: 画像 成功 (openai/gpt-5.6-sol) · 音声 成功 (whisper-cli 検出値=metal)
```

事前確認用のインベントリを取得するには、`openclaw capability audio providers` を実行します。ローカル行には、グローバルなプロバイダー選択、準備状況、個別の対応可能・要求済み・検出済みバックエンドフィールドとは別に、ローカルフォールバックの選択結果が表示されます。同じローカル選択は、情報レベルの doctor 検出結果としても確認できます。

```bash
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

## 注記

- 理解処理はベストエフォートです。エラーが応答を妨げることはありません。
- 理解処理が無効でも、添付ファイルはモデルに渡されます。
- 理解処理を実行する場所を制限するには、`scope` を使用します（たとえば、DM のみ）。

## 関連項目

- [設定](/ja-JP/gateway/configuration)
- [画像とメディアのサポート](/ja-JP/nodes/images)
