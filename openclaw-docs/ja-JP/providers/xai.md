---
read_when:
    - OpenClaw で Grok モデルを使用する場合
    - xAI の認証またはモデル ID を設定している場合
summary: OpenClaw で xAI Grok モデルを使用する
title: xAI
x-i18n:
    generated_at: "2026-07-26T10:00:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71ae7b049649b08b6508b8331714fec3464628814629256ad23b584f0f8ca8b7
    source_path: providers/xai.md
    workflow: 16
---

OpenClaw には、Grok モデル用の `xai` プロバイダー Plugin が同梱されています。推奨される方法は、対象となる SuperGrok または X Premium サブスクリプションで Grok OAuth を使用することです。Gateway、設定、ルーティング、ツールはローカルに維持され、Grok リクエストのみが xAI の API に送信されます。

OAuth では、xAI API キーも Grok Build アプリも必要ありません。OpenClaw は xAI の共有 OAuth クライアントを使用するため、xAI の同意画面には引き続き Grok Build が表示される場合があります。

## セットアップ

<Steps>
  <Step title="新規インストール">
    デーモンのインストールを含めてオンボーディングを実行し、モデル／認証の手順で xAI/Grok OAuth を選択します。

    ```bash
    openclaw onboard --install-daemon
    ```

    VPS または SSH 経由の場合は、xAI OAuth を直接選択します。これはデバイスコード検証を使用し、localhost コールバックを必要としません。

    ```bash
    openclaw onboard --install-daemon --auth-choice xai-oauth
    ```

  </Step>
  <Step title="既存のインストール">
    xAI にのみサインインします。Grok に接続するためだけに、オンボーディング全体を再実行しないでください。

    ```bash
    openclaw models auth login --provider xai --method oauth
    ```

    Grok をデフォルトモデルとして別途適用します。

    ```bash
    openclaw models set xai/grok-4.3
    ```

    Gateway、デーモン、チャンネル、ワークスペース、またはその他のセットアップ項目を意図的に変更する場合にのみ、オンボーディング全体を再実行してください。

  </Step>
  <Step title="API キーを使用する方法">
    API キーによるセットアップは、xAI Console のキー、およびキーに基づくプロバイダー設定が必要なメディア機能でも引き続き利用できます。

    ```bash
    openclaw models auth login --provider xai --method api-key
    export XAI_API_KEY=xai-...
    ```

  </Step>
  <Step title="モデルを選択">
    ```json5
    {
      agents: { defaults: { model: { primary: "xai/grok-4.3" } } },
    }
    ```
  </Step>
</Steps>

<Note>
OpenClaw は、同梱の xAI トランスポートとして xAI Responses API を使用します。`openclaw models auth login --provider xai --method oauth` または
`--method api-key` の同じ認証情報は、`web_search`（プロバイダー ID `grok`）、`x_search`、
`code_execution`、音声／文字起こし、および xAI の画像／動画生成にも使用されます。xAI キーを
`plugins.entries.xai.config.webSearch.apiKey` に保存すると、同梱の xAI モデルプロバイダーもフォールバックとしてそのキーを再利用します。
</Note>

## OAuth のトラブルシューティング

- SSH、Docker、VPS、またはその他のリモートセットアップでは、
  `openclaw models auth login --provider xai --method oauth` を使用してください。これは localhost コールバックではなく、
  デバイスコード検証を使用します。
- サインインに成功しても Grok がデフォルトモデルになっていない場合は、
  `openclaw models set xai/grok-4.3` を実行してください。
- 保存済みの xAI 認証プロファイルを確認します。

  ```bash
  openclaw models auth list --provider xai
  openclaw models status
  ```

- OAuth API トークンを取得できるアカウントは xAI が決定します。アカウントが対象外の場合は、
  API キーを使用するか、xAI 側でサブスクリプションを確認してください。

<Tip>
SSH、Docker、または VPS からサインインする場合は、`xai-oauth` を使用してください。OpenClaw は
URL と短いコードを表示します。リモートプロセスが完了済みのトークン交換について xAI をポーリングしている間に、
任意のローカルブラウザーでサインインを完了してください。
</Tip>

## 組み込みカタログ

モデル選択画面で選択可能な ID です。Plugin は、既存の設定に対応するため、以前の Grok 3、
Grok 4、Grok 4 Fast、Grok 4.1 Fast、Grok Code の ID も引き続き解決します。
[従来バージョンとの互換性と可変エイリアス](#legacy-compatibility-and-moving-aliases)を参照してください。

| ファミリー         | モデル ID                                                    |
| -------------- | ------------------------------------------------------------ |
| Grok 4.5       | `grok-4.5`（エイリアス: `grok-4.5-latest`、`grok-build-latest`） |
| Grok Build 0.1 | `grok-build-0.1`                                             |
| Grok 4.3       | `grok-4.3`（エイリアス: `grok-4.3-latest`、`grok-latest`）       |
| Grok 4.20      | `grok-4.20-0309-reasoning`、`grok-4.20-0309-non-reasoning`   |

<Tip>
利用可能な場合、一般的なチャット、コーディング、エージェント型の作業には `grok-4.5` を使用してください。
Grok 4.3 は引き続き地域安全性を考慮したセットアップのデフォルトです。`grok-build-0.1` と、
日付付きの Grok 4.20 の両バリアントも引き続き選択できます。
</Tip>

カタログのコンテキストとトークンコストのメタデータは、xAI の最新の
[モデルページ](https://docs.x.ai/developers/models)と
[料金ページ](https://docs.x.ai/developers/pricing)に準拠しています。リクエストが文書化された長いコンテキストのしきい値を超えると、
xAI はより高い料金を適用します。OpenClaw のカタログにある一律のコストフィールドには、短いコンテキストの料金が記録されています。
xAI の独立したコーディングエージェント CLI である Grok Build は、[x.ai/cli](https://x.ai/cli) で利用でき、
現在は Grok 4.5 を使用しています。

## 機能対応状況

同梱の Plugin は、対応する xAI API を OpenClaw の共有プロバイダーおよび
ツール契約にマッピングします。共有契約に適合しない機能は、以下または既知の制限に記載されています。

| xAI の機能             | OpenClaw の機能                        | 状態                                               |
| -------------------------- | --------------------------------------- | ---------------------------------------------------- |
| チャット／Responses           | `xai/<model>` モデルプロバイダー            | 対応                                                  |
| サーバー側ウェブ検索     | `web_search` プロバイダー `grok`            | 対応                                                  |
| サーバー側 X 検索       | `x_search` ツール                         | 対応                                                  |
| サーバー側コード実行 | `code_execution` ツール                   | 対応                                                  |
| 画像                     | `image_generate`                        | 対応                                                  |
| 動画                     | `video_generate`                        | 対応                                                  |
| バッチテキスト読み上げ       | `tts.provider: "xai"`／`tts`           | 対応                                                  |
| ストリーミング TTS              | `textToSpeechStream`                    | `wss://api.x.ai/v1/tts` 経由で対応（リアルタイム音声ではありません） |
| バッチ音声テキスト変換       | `tools.media.audio` メディア理解 | 対応                                                  |
| ストリーミング音声テキスト変換   | Voice Call `streaming.provider: "xai"`  | 対応                                                  |
| リアルタイム音声             | Talk `talk.realtime.provider: "xai"`    | 対応。ネイティブ Talk Node では Gateway リレーを使用             |
| ファイル／バッチ            | 汎用モデル API との互換性のみ    | OpenClaw の第一級ツールではありません                      |

<Note>
OpenClaw は、メディア生成とバッチ文字起こしに xAI の REST 画像／動画／TTS／STT API、
ライブ音声通話の文字起こしに xAI のストリーミング STT WebSocket、
Talk のリアルタイムセッションに xAI の Grok Voice Agent WebSocket、
チャット、検索、コード実行ツールに Responses API を使用します。
</Note>

### 従来の高速モードとの互換性

`/fast on` または `agents.defaults.models["xai/<model>"].params.fastMode: true` は、
以前の xAI 設定を引き続き次のように書き換えます。これらのターゲット ID は
互換性のためだけに維持されています。新しい設定では、現在選択可能なモデルを使用してください。

| ソースモデル  | 高速モードのターゲット   |
| ------------- | ------------------ |
| `grok-3`      | `grok-3-fast`      |
| `grok-3-mini` | `grok-3-mini-fast` |
| `grok-4`      | `grok-4-fast`      |
| `grok-4-0709` | `grok-4-fast`      |

### 従来バージョンとの互換性と可変エイリアス

以前のエイリアスは次のように正規化されます。

| 従来のエイリアス                                                  | 正規化後の ID    |
| ------------------------------------------------------------- | ---------------- |
| `grok-code-fast-1`、`grok-code-fast`、`grok-code-fast-1-0825` | `grok-build-0.1` |

日付付きの 0309 ID が、選択可能なカタログエントリです。OpenClaw は、その他の現在の
Grok 4.20 エイリアスをすべてそのまま送信するため、stable、latest、
beta、experimental、および日付付きエイリアスのセマンティクスは xAI が引き続き制御します。
グローバルな `grok-latest` エイリアスもそのまま保持されます。

xAI は次の完全一致 ID を廃止しました。OpenClaw は、リリース済みの設定との互換性を保つため、
現在のリダイレクト先の制限と料金を適用する非表示の互換性行としてこれらを維持します。

| 廃止された ID                                                          | 現在の動作                 |
| -------------------------------------------------------------------- | -------------------------------- |
| `grok-4-1-fast-reasoning`、`grok-4-fast-reasoning`、`grok-4-0709`    | `low` 推論を使用する Grok 4.3    |
| `grok-4-1-fast-non-reasoning`、`grok-4-fast-non-reasoning`、`grok-3` | 推論を無効にした Grok 4.3 |
| `grok-code-fast-1`                                                   | Grok Build 0.1                   |
| `grok-imagine-image-pro`                                             | Grok Imagine Image Quality       |

`openclaw doctor --fix` は、永続化された xAI サーバーツールのデフォルトと
廃止された高品質画像のスラッグを更新し、古い生成済みカタログ行を削除し、
有効な 4.20 行の古いコンテキストメタデータを修復します。有効な 4.20 の
`beta-latest` エイリアスを日付付きスナップショットに固定することはありません。

## 機能

<Warning>
  `x_search` と `code_execution` は xAI のサーバー上で実行されます。xAI はツール呼び出し
  1,000 回あたり $5 に加え、モデルの入力トークンと出力トークンについて課金します。各ツールの
  `enabled` 設定を省略すると、OpenClaw は有効な xAI モデルに対してのみそのツールを公開します。
  既知の非 xAI モデルプロバイダーでは、ツールごとに明示的な `enabled: true` が必要です。
  プロバイダーが指定されていないか解決できない場合は、フェイルクローズします。xAI 認証は常に必要であり、
  `enabled: false` はすべてのプロバイダーに対してツールを無効にします。
</Warning>

<AccordionGroup>
  <Accordion title="ウェブ検索">
    同梱の `grok` ウェブ検索プロバイダーは xAI OAuth を優先し、その後
    `XAI_API_KEY` または Plugin のウェブ検索キーにフォールバックします。

    ```bash
    openclaw models auth login --provider xai --method oauth
    openclaw config set tools.web.search.provider grok
    ```

  </Accordion>

  <Accordion title="動画生成">
    同梱の `xai` Plugin は、共有の
    `video_generate` ツールを介して動画生成を登録します。

    - デフォルトモデル: `xai/grok-imagine-video`
    - 追加モデル: `xai/grok-imagine-video-1.5`
    - クラシックモード: テキストから動画、画像から動画、参照画像による生成、
      リモート動画編集、リモート動画拡張
    - Video 1.5 モード: 画像から動画のみ。最初のフレームとして使用する画像は正確に 1 枚
    - アスペクト比: `1:1`、`16:9`、`9:16`、`4:3`、`3:4`、`3:2`、`2:3`。
      省略した場合、クラシックおよび Video 1.5 の画像から動画では、ソース画像の比率を継承
    - 解像度: クラシックは `480P`／`720P`。Video 1.5 は `1080P` にも対応。すべての
      生成モードのデフォルトは `480P`
    - 時間: 生成／画像から動画では 1～15 秒、クラシックの `reference_image` ロールを
      使用する場合は 1～10 秒、クラシック拡張では 2～10 秒
    - 参照画像による生成: 指定するすべての画像で `imageRoles` を `reference_image` に設定。
      xAI はこのような画像を最大 7 枚受け付けます
    - 動画の編集／拡張では、入力動画のアスペクト比と解像度を継承します。
      これらの操作ではジオメトリの上書きは受け付けません
    - デフォルトの操作タイムアウト: `video_generate.timeoutMs`
      または `agents.defaults.mediaModels.video.timeoutMs` が設定されていない限り 600 秒

    <Warning>
    ローカルの動画バッファは受け付けられません。動画の編集／拡張の入力には、リモートの
    `http(s)` URL を使用してください。画像から動画ではローカル画像バッファを受け付けます。
    OpenClaw がそれらを xAI 用のデータ URL としてエンコードするためです。
    </Warning>

    Video 1.5 は、xAI の `grok-imagine-video-1.5-preview` および
    `grok-imagine-video-1.5-2026-05-30` 識別子も認識します。OpenClaw は選択された
    識別子を変更せずに転送しますが、同じ画像限定の検証を適用します。

    xAI をデフォルトの動画プロバイダーとして使用するには、次のように設定します。

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "xai/grok-imagine-video",
          },
        },
      },
    }
    ```

    <Note>
    共通ツールのパラメーター、プロバイダーの選択、フェイルオーバー動作については、[動画生成](/ja-JP/tools/video-generation)を参照してください。
    </Note>

  </Accordion>

  <Accordion title="画像生成">
    バンドルされている `xai` Plugin は、共有 `image_generate` ツールを通じて画像生成を登録します。

    - デフォルトの画像モデル: `xai/grok-imagine-image`
    - 追加モデル: `xai/grok-imagine-image-quality`
    - モード: テキストから画像への生成、および参照画像の編集
    - 参照入力: `image` 1つ、または `images` 最大3つ
    - アスペクト比: `1:1`、`16:9`、`9:16`、`4:3`、`3:4`、`3:2`、`2:3`、`2:1`、
      `1:2`、`19.5:9`、`9:19.5`、`20:9`、`9:20`
    - 解像度: `1K`、`2K`
    - 生成数: 最大4画像
    - デフォルトの処理タイムアウト: `image_generate.timeoutMs` または
      `agents.defaults.mediaModels.image.timeoutMs` が設定されていない限り600秒

    OpenClaw は、生成されたメディアを通常のチャンネル添付ファイルの経路で保存および配信できるよう、xAI に `b64_json` 形式の画像レスポンスを要求します。ローカルの参照画像はデータ URL に変換され、リモートの `http(s)` 参照は変更されずに渡されます。

    xAI をデフォルトの画像プロバイダーとして使用するには、次のように設定します。

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "xai/grok-imagine-image",
          },
        },
      },
    }
    ```

    <Note>
    xAI は、`quality`、`mask`、`user`、および `auto` のアスペクト比についても文書化しています。
    現在、OpenClaw が転送するのはプロバイダー間で共有される画像制御のみです。これらのネイティブ専用オプションは `image_generate` では公開されていません。
    </Note>

  </Accordion>

  <Accordion title="テキスト読み上げ">
    バンドルされている `xai` Plugin は、共有 `tts` プロバイダーサーフェスを通じてテキスト読み上げを登録します。

    - 音声: xAI から取得する認証済みライブカタログ。`openclaw infer tts voices --provider xai` で一覧表示可能
    - オフラインフォールバック音声: `ara`、`eve`、`leo`、`rex`、`sal`
    - デフォルト音声: `eve`
    - アカウントのカスタム音声 ID は、組み込みカタログのレスポンスに含まれていない場合でも転送されます
    - 形式: `mp3`、`wav`、`pcm`、`mulaw`、`alaw`
    - 言語: BCP-47 コードまたは `auto`
    - 速度: プロバイダーネイティブの速度オーバーライド
    - ネイティブの Opus ボイスメモ形式は未対応

    xAI をデフォルトの TTS プロバイダーとして使用するには、次のように設定します。

    ```json5
    {
      tts: {
        provider: "xai",
        providers: {
          xai: {
            voiceId: "eve",
          },
        },
      },
    }
    ```

    <Note>
    OpenClaw は、バッファリング合成に xAI のバッチ `/v1/tts` エンドポイント、認証済み `/v1/tts/voices` カタログの検出、およびストリーミング合成にネイティブの `wss://api.x.ai/v1/tts` を使用します。ストリーミングはネイティブの `api.x.ai` ホストに制限されるため、カスタムの `baseUrl` 値はこの経路では拒否されます。既存の言語、音声、コーデック、速度の制御を使用し、サンプルレートとビットレートには xAI のデフォルト値が適用されます。音声ファイルの合成では、設定されたすべてのコーデックが反映されます。xAI の RAW コーデックにはコーデックやレートのメタデータが含まれないため、ボイスメモの送信先ではストリーミングとバッファリングによるフォールバックの両方で MP3 を使用します。ストリームは `text.delta`、続いて
    `text.done` を送信し、`audio.delta`、`audio.done`、または `error` を受信します。また、音声チャンクごとに更新されるアイドル `timeoutMs` を適用します。これはリアルタイム音声セッションとは別のものです。xAI の [ストリーミング TTS API](https://docs.x.ai/developers/rest-api-reference/inference/voice) の契約を参照してください。
    </Note>

  </Accordion>

  <Accordion title="音声文字起こし">
    バンドルされている `xai` Plugin は、OpenClaw のメディア理解文字起こしサーフェスを通じてバッチ音声文字起こしを登録します。

    - エンドポイント: xAI REST `/v1/stt`
    - 入力経路: マルチパート音声ファイルのアップロード
    - モデルの選択: xAI が文字起こしモデルを内部で選択します。このエンドポイントにはモデルセレクターがありません
    - Discord ボイスチャンネルのセグメントやチャンネルの音声添付ファイルなど、受信音声の文字起こしで `tools.media.audio` を読み取るすべての箇所に使用されます

    受信音声の文字起こしで xAI を強制的に使用するには、次のように設定します。

    ```json5
    {
      tools: {
        media: {
          audio: {
            models: [
              {
                type: "provider",
                provider: "xai",
              },
            ],
          },
        },
      },
    }
    ```

    言語は、共有の音声メディア設定または呼び出しごとの文字起こしリクエストで指定できます。プロンプトのヒントは共有 OpenClaw サーフェスで受け入れられますが、現在公開されている xAI エンドポイントに対応するのはファイルと言語のみであるため、xAI REST STT 統合が転送するのはこの2つだけです。

  </Accordion>

  <Accordion title="ストリーミング音声文字起こし">
    バンドルされている `xai` Plugin は、ライブ音声通話のためのリアルタイム文字起こしプロバイダーも登録します。

    - エンドポイント: xAI WebSocket `wss://api.x.ai/v1/stt`
    - デフォルトのエンコーディング: `mulaw`
    - デフォルトのサンプルレート: `8000`
    - デフォルトのエンドポイント検出: `800ms`
    - 中間文字起こし: デフォルトで有効

    Voice Call の Twilio メディアストリームは G.711 mu-law 音声フレームを送信するため、xAI プロバイダーはトランスコードせずにそれらのフレームを直接転送します。

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              streaming: {
                enabled: true,
                provider: "xai",
                providers: {
                  xai: {
                    apiKey: "${XAI_API_KEY}",
                    endpointingMs: 800,
                    language: "en",
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

    プロバイダー所有の設定は
    `plugins.entries.voice-call.config.streaming.providers.xai` に配置します。対応するキーは `apiKey`、`baseUrl`、`sampleRate`、`encoding`（`pcm`、`mulaw`、または
    `alaw`）、`interimResults`、`endpointingMs`、`language` です。

    <Note>
    このストリーミングプロバイダーは、Voice Call のリアルタイム文字起こし経路用です。
    Discord の音声は短いセグメントとして録音され、代わりにバッチ
    `tools.media.audio` 文字起こし経路を使用します。
    </Note>

  </Accordion>

  <Accordion title="リアルタイム音声（Talk）">
    バンドルされている `xai` Plugin は、共有 `registerRealtimeVoiceProvider` 契約を通じて Talk モード用の Grok Voice Agent リアルタイムセッションを登録します。

    - エンドポイント: `wss://api.x.ai/v1/realtime?model=<voice-model>`
    - デフォルトモデル: `grok-voice-latest`
    - デフォルト音声: `eve`
    - トランスポート: `gateway-relay`（iOS、Android、Control UI のリレー経路）
    - 音声: PCM16 24 kHz または G.711 µ-law 8 kHz
    - 割り込み: xAI サーバーの VAD がレスポンスを中断します。OpenClaw はキューに入っている再生を消去し、未再生のプロバイダー履歴を切り詰めます

    Gateway で Talk を設定します。

    ```json5
    {
      talk: {
        realtime: {
          provider: "xai",
          mode: "realtime",
          transport: "gateway-relay",
          brain: "agent-consult",
          providers: {
            xai: {
              model: "grok-voice-latest",
              voice: "eve",
              // プロバイダー側でのセッション再生を許容できる場合のみ有効にします。
              sessionResumption: false,
            },
          },
        },
      },
      env: { XAI_API_KEY: "xai-..." },
    }
    ```

    Voice Call または共有リアルタイムセレクターが同じプロバイダーマップを再利用する場合、プロバイダー所有の設定は
    `plugins.entries.voice-call.config.realtime.providers.xai` からも解決されます。対応するキーは
    `apiKey`、`baseUrl`、`model`、`voice`、`vadThreshold`、`silenceDurationMs`、
    `prefixPaddingMs`、`reasoningEffort`、`sessionResumption` です。
    `reasoningEffort` は、xAI Voice Agent API に合わせて `high` または `none` のみを受け入れます。

    xAI のサーバー VAD は常にレスポンスを作成し、音声の割り込みを処理します。
    `consultRouting: "provider-direct"` を使用してください。強制的な文字起こしのルーティングと入力音声の割り込み無効化は、xAI Voice Agent プロトコルではサポートされていません。

    <Note>
    xAI OAuth または `XAI_API_KEY` でリアルタイム音声を認証できます。ブラウザー所有の
    WebRTC は、まだこのプロバイダーサーフェスには含まれていません。ネイティブ Node では gateway-relay Talk を、または Control UI のリレー経路を使用してください。
    </Note>

    <Note>
    `sessionResumption` のデフォルトは `false` です。`true` に設定すると、OpenClaw は再接続後に同じ会話を再開するために十分なセッション状態を保持するよう xAI に要求し、その後、返された会話 ID を使用して再接続します。プロバイダー側での再生や保持を許容できない場合は無効のままにしてください。その場合、中断されたソケットは新しい会話を暗黙に開始せず、フェイルクローズします。
    </Note>

  </Accordion>

  <Accordion title="x_search の設定">
    バンドルされている xAI Plugin は、Grok を介して X（旧 Twitter）のコンテンツを検索する OpenClaw ツールとして `x_search` を公開します。

    設定パス: `plugins.entries.xai.config.xSearch`

    | キー               | 型    | デフォルト                   | 説明                                      |
    | ----------------- | ------- | ------------------------- | ------------------------------------------------ |
    | `enabled`         | boolean | xAI モデルでは自動  | 無効化するか、既知の非 xAI プロバイダーで明示的に有効化 |
    | `model`           | string  | `grok-4.3`                | x_search リクエストに使用するモデル                 |
    | `baseUrl`         | string  | -                         | xAI Responses のベース URL のオーバーライド                  |
    | `inlineCitations` | boolean | -                         | 結果にインライン引用を含める              |
    | `maxTurns`        | number  | -                         | 会話ターンの最大数                       |
    | `timeoutSeconds`  | number  | `30`                      | リクエストのタイムアウト（秒）                       |
    | `cacheTtlMinutes` | number  | `15`                      | キャッシュの有効期間（分）                    |

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              xSearch: {
                enabled: true,
                model: "grok-4.3",
                baseUrl: "https://api.x.ai/v1",
                inlineCitations: true,
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="コード実行の設定">
    バンドルされている xAI Plugin は、xAI のサンドボックス環境でリモートコードを実行する OpenClaw ツールとして `code_execution` を公開します。

    設定パス: `plugins.entries.xai.config.codeExecution`

    | キー              | 型    | デフォルト                  | 説明                                      |
    | ---------------- | ------- | ------------------------ | ------------------------------------------------ |
    | `enabled`        | boolean | xAI モデルでは自動 | 無効化するか、既知の非 xAI プロバイダーで有効化 |
    | `model`          | string  | `grok-4.3`               | コード実行リクエストに使用するモデル           |
    | `maxTurns`       | number  | -                        | 会話の最大ターン数                       |
    | `timeoutSeconds` | number  | `30`                     | リクエストのタイムアウト（秒）                       |

    <Note>
    これはリモートの xAI サンドボックス実行であり、ローカルの [`exec`](/ja-JP/tools/exec) ではありません。
    </Note>

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              codeExecution: {
                enabled: true,
                model: "grok-4.3",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="既知の制限">
    - xAI 認証では、API キー、環境変数、Plugin 設定の
      フォールバック、または対象となる xAI アカウントでの OAuth を使用できます。OAuth は localhost
      コールバックを使わず、デバイスコード検証を使用します。OAuth API トークンを
      取得できるアカウントは xAI が決定します。また、OpenClaw は Grok Build アプリを
      必要としませんが、同意ページに Grok Build と表示される場合があります。
    - OpenClaw は現在、xAI のマルチエージェントモデルファミリーを公開していません。xAI は
      これらのモデルを Responses API 経由で提供していますが、OpenClaw の共有エージェントループで
      使用されるクライアント側ツールやカスタムツールは受け付けません。
      [xAI マルチエージェントの制限](https://docs.x.ai/developers/model-capabilities/text/multi-agent#limitations)
      を参照してください。
    - xAI Realtime 音声は現在、Gateway リレーの Talk トランスポートのみを公開しています。
      ブラウザが所有するプロバイダー WebSocket セッションは、まだ Control UI に
      接続されていません。
    - xAI の画像 `quality`、画像 `mask`、およびネイティブ専用の追加アスペクト比は、
      共有 `image_generate` ツールに対応する
      クロスプロバイダー制御が追加されるまで公開されません。
  </Accordion>

  <Accordion title="高度な注記">
    - OpenClaw は、共有ランナーパス上で xAI 固有のツールスキーマおよびツール呼び出しの互換性修正を
      自動的に適用します。
    - ネイティブ xAI リクエストでは、デフォルトで `tool_stream: true` が設定されます。無効にするには、
      `agents.defaults.models["xai/<model>"].params.tool_stream` を `false`
      に設定します。
    - バンドルされた xAI ラッパーは、ネイティブ xAI リクエストを送信する前に、サポートされていない contains-count スキーマ境界と
      サポートされていない推論 *effort* ペイロードキーを削除します。Grok 4.5 は low、medium、
      high の effort をサポートします（デフォルトは high）。Grok 4.3 は none、low、medium、high の
      effort をサポートします（デフォルトは low）。推論機能を持つその他の xAI モデルは、設定可能な
      effort 制御を公開しませんが、後続ターンで以前の暗号化された推論を
      再生できるように、引き続き `include: ["reasoning.encrypted_content"]`
      をリクエストします。
    - `web_search`、`x_search`、および `code_execution` は OpenClaw
      ツールとして公開されます。OpenClaw は、すべてのチャットターンにすべてのネイティブツールを
      添付するのではなく、各ツールが必要とする特定の xAI 組み込み機能のみを
      そのツールのリクエストに添付します。
    - Grok `web_search` は `plugins.entries.xai.config.webSearch.baseUrl` を読み取ります。
      `x_search` は `plugins.entries.xai.config.xSearch.baseUrl` を読み取り、その後
      Grok Web 検索のベース URL にフォールバックします。
    - `x_search` と `code_execution` は、コアモデルランタイムに
      ハードコードされるのではなく、バンドルされた xAI Plugin が所有します。
    - `code_execution` はリモートの xAI サンドボックス実行であり、ローカルの
      [`exec`](/ja-JP/tools/exec) ではありません。
  </Accordion>
</AccordionGroup>

## ライブテスト

xAI メディアパスは、ユニットテストとオプトインのライブスイートでカバーされています。ライブプローブを実行する前に、
プロセス環境で `XAI_API_KEY` をエクスポートしてください。

```bash
pnpm test extensions/xai
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_TEST_QUIET=1 pnpm test:live -- extensions/xai/xai.live.test.ts
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "classic Grok Imagine"
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "Grok Imagine Video 1.5"
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_TEST_QUIET=1 pnpm test:live -- extensions/xai/x-search.live.test.ts
OPENCLAW_LIVE_GATEWAY_MODELS="xai/grok-4.5,xai/grok-build-0.1,xai/grok-4.3,xai/grok-4.20-0309-reasoning,xai/grok-4.20-0309-non-reasoning" OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0 OPENCLAW_LIVE_GATEWAY_SMOKE=0 pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_TEST_QUIET=1 OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS=xai pnpm test:live -- test/image-generation.runtime.live.test.ts
```

プロバイダー固有のライブファイルは、通常の TTS、電話向け PCM
TTS を合成し、xAI バッチ STT で音声を文字起こしし、同じ PCM を xAI
リアルタイム STT でストリーミングし、テキストから画像への出力を生成し、参照画像を編集します。
共有画像ライブファイルは、OpenClaw のランタイム選択、フォールバック、正規化、
メディア添付パスを通じて同じ xAI プロバイダーを検証します。
オプトインの Video 1.5 ケースは、生成された最初のフレーム画像を 1080P で 1 枚送信し、
完成した動画のダウンロードを検証します。

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、およびフェイルオーバー動作の選択。
  </Card>
  <Card title="動画生成" href="/ja-JP/tools/video-generation" icon="video">
    共有動画ツールのパラメーターとプロバイダーの選択。
  </Card>
  <Card title="すべてのプロバイダー" href="/ja-JP/providers/index" icon="grid-2">
    より広範なプロバイダーの概要。
  </Card>
  <Card title="トラブルシューティング" href="/ja-JP/help/troubleshooting" icon="wrench">
    一般的な問題と修正方法。
  </Card>
</CardGroup>
