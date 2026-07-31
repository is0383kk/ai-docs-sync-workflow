---
read_when:
    - エージェントによる動画生成
    - 動画生成プロバイダーとモデルの設定
    - video_generate ツールのパラメーターを理解する
sidebarTitle: Video generation
summary: 16 のプロバイダーバックエンドで、テキスト、画像、または動画の参照から video_generate を使用して動画を生成します
title: 動画生成
x-i18n:
    generated_at: "2026-07-26T10:34:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4afc9338fdfdc269b50b949b6d1a186e3a2064ed4ee40a41722efea40ae791aa
    source_path: tools/video-generation.md
    workflow: 16
---

OpenClaw エージェントは、`video_generate` を通じて、テキストプロンプト、参照画像、または
既存の動画から動画を生成します。16 のプロバイダーバックエンドが
サポートされており、エージェントは設定と利用可能な API キーに基づいて適切なものを自動的に選択します。

<Note>
`video_generate` は、動画生成プロバイダーが少なくとも 1 つ
利用可能な場合にのみ表示されます。エージェントツールにない場合は、プロバイダーの API キーを設定するか、
`agents.defaults.mediaModels.video` を設定してください。
</Note>

`video_generate` には 3 つのランタイムモードがあり、呼び出し内の参照入力から
決定されます。

- `generate` - 参照メディアなし（テキストから動画）。
- `imageToVideo` - 1 つ以上の参照画像。
- `videoToVideo` - 1 つ以上の参照動画。

プロバイダーは、これらのモードの任意の組み合わせをサポートできます。ツールは送信前に
アクティブなモードを検証し、`action=list` でサポートされているモードを報告します。

## クイックスタート

<Steps>
  <Step title="認証を設定">
    サポートされている任意のプロバイダーの API キーを設定します。

    ```bash
    export GEMINI_API_KEY="your-key"
    ```

  </Step>
  <Step title="デフォルトモデルを選択（任意）">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "google/veo-3.1-fast-generate-preview"
    ```
  </Step>
  <Step title="エージェントに依頼">
    > 夕暮れ時にサーフィンをする親しみやすいロブスターの、映画のような 5 秒間の動画を生成してください。

    エージェントは `video_generate` を自動的に呼び出します。ツールの許可リストへの登録は
    必要ありません。

  </Step>
</Steps>

## 非同期生成の仕組み

動画生成は非同期です。

1. OpenClaw はリクエストをプロバイダーに送信し、タスク ID を即座に返します。
2. プロバイダーはバックグラウンドでジョブを処理します（プロバイダーと解像度に応じて通常 30 秒から数分。キューを使用する低速なプロバイダーでは、設定されたタイムアウトまで実行される場合があります）。
3. 動画の準備が完了すると、OpenClaw は内部の完了イベントで同じセッションを起動します。
4. エージェントは、セッションの通常の可視返信モードを通じて報告します。
   自動の最終返信、またはセッションでメッセージツールが必要な場合は `message(action="send")` です。
   リクエスターのセッションが非アクティブである場合、またはその起動に失敗し、生成されたメディアが完了返信にも含まれていない場合、OpenClaw は
   メディアを含む冪等な直接フォールバックを送信します。

ジョブの実行中、同じセッション内で重複する `video_generate` 呼び出しは、
別の生成を開始する代わりに、現在のタスクステータスを返します。
新しい生成をトリガーせずに確認するには `action: "status"` を使用するか、
CLI から `openclaw tasks list` / `openclaw tasks show <lookup>` を使用します
（[バックグラウンドタスク](/ja-JP/automation/tasks)を参照）。

セッションを利用するエージェント実行の外部（たとえば、ツールの直接呼び出し）では、
ツールはインライン生成にフォールバックし、同じターンで最終メディアのパスを
返します。

プロバイダーがバイトデータを返す場合、生成された動画ファイルは OpenClaw が管理するメディアストレージに
保存されます。デフォルトの上限は 16MB（共有動画メディアの
上限）です。より大きなレンダリングには `agents.defaults.mediaMaxMb` で上限を引き上げます。
プロバイダーがホストされた出力 URL も返す場合、ローカルへの永続化でサイズ超過のファイルが拒否されても、OpenClaw は
タスクを失敗させる代わりにその URL を配信します。

### タスクのライフサイクル

| 状態       | 意味                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------ |
| `queued`    | タスクが作成され、プロバイダーによる受け入れを待機しています。                                                   |
| `running`   | プロバイダーが処理中です（プロバイダーと解像度に応じて通常 30 秒から数分）。 |
| `succeeded` | 動画の準備が完了し、エージェントが起動して会話に投稿します。                                         |
| `failed`    | プロバイダーのエラーまたはタイムアウトです。エージェントがエラーの詳細とともに起動します。                                         |

CLI からステータスを確認します。

```bash
openclaw tasks list
openclaw tasks show <lookup>
openclaw tasks cancel <lookup>
```

## サポートされているプロバイダー

| プロバイダー              | デフォルトモデル                   | テキスト | 画像参照                                            | 動画参照                                       | 認証                                     |
| --------------------- | ------------------------------- | :--: | ---------------------------------------------------- | ----------------------------------------------- | ---------------------------------------- |
| Alibaba               | `wan2.6-t2v`                    |  ✓   | 対応（リモート URL）                                     | 対応（リモート URL）                                | `MODELSTUDIO_API_KEY`                    |
| BytePlus（同梱）    | `seedance-1-0-pro-250528`       |  ✓   | 最大 2 画像（最初と最後のフレーム）                  | -                                               | `BYTEPLUS_API_KEY`                       |
| BytePlus 1.5 Plugin   | `seedance-1-5-pro-251215`       |  ✓   | 最大 2 画像（ロールを介した最初と最後のフレーム）         | -                                               | `BYTEPLUS_API_KEY`                       |
| BytePlus Seedance 2.0 | `dreamina-seedance-2-0-260128`  |  ✓   | 最大 9 つの参照画像                             | 最大 3 つの動画                                  | `BYTEPLUS_API_KEY`                       |
| ComfyUI               | `workflow`                      |  ✓   | 1 画像                                              | -                                               | `COMFY_API_KEY` または `COMFY_CLOUD_API_KEY` |
| DeepInfra             | `Pixverse/Pixverse-T2V`         |  ✓   | -                                                    | -                                               | `DEEPINFRA_API_KEY`                      |
| fal                   | `fal-ai/minimax/video-01-live`  |  ✓   | 1 画像。Seedance の参照から動画では最大 9 画像    | Seedance の参照から動画では最大 3 動画 | `FAL_KEY`                                |
| Google                | `veo-3.1-fast-generate-preview` |  ✓   | 1 画像                                              | 1 動画                                         | `GEMINI_API_KEY`                         |
| MiniMax               | `MiniMax-Hailuo-2.3`            |  ✓   | 1 画像                                              | -                                               | `MINIMAX_API_KEY` または MiniMax OAuth       |
| OpenAI                | `sora-2`                        |  ✓   | 1 画像                                              | 1 動画                                         | `OPENAI_API_KEY`                         |
| OpenRouter            | `google/veo-3.1-fast`           |  ✓   | 最大 4 画像（最初/最後のフレームまたは参照）      | -                                               | `OPENROUTER_API_KEY`                     |
| Qwen                  | `wan2.6-t2v`                    |  ✓   | 対応（リモート URL）                                     | 対応（リモート URL）                                | `QWEN_API_KEY`                           |
| Runway                | `gen4.5`                        |  ✓   | 1 画像                                              | 1 動画                                         | `RUNWAYML_API_SECRET`                    |
| Together              | `Wan-AI/Wan2.2-T2V-A14B`        |  ✓   | `Wan-AI/Wan2.2-I2V-A14B` のみ                        | -                                               | `TOGETHER_API_KEY`                       |
| Vydra                 | `veo3`                          |  ✓   | 1 画像（`kling`）                                    | -                                               | `VYDRA_API_KEY`                          |
| xAI                   | `grok-imagine-video`            |  ✓   | Classic: 最初の 1 フレームまたは 7 参照。1.5: 1 フレーム | Classic: 1 動画                                | `XAI_API_KEY`                            |

一部のプロバイダーは、追加または代替の API キー環境変数を受け付けます。詳細については、
各[プロバイダーページ](#related)を参照してください。

実行時に利用可能なプロバイダー、モデル、およびランタイムモードを確認するには、
`video_generate action=list` を実行します。

### 機能マトリックス

`video_generate`、コントラクトテスト、および
共有ライブスイープで使用される明示的なモードコントラクトです。

| プロバイダー   | `generate` | `imageToVideo` | `videoToVideo` | 現在の共有ライブレーン                                                                                                                 |
| ---------- | :--------: | :------------: | :------------: | --------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba    |     ✓      |       ✓        |       ✓        | `generate`、`imageToVideo`。このプロバイダーにはリモートの `http(s)` 動画 URL が必要なため、`videoToVideo` はスキップされます                              |
| BytePlus   |     ✓      |       ✓        |       -        | `generate`、`imageToVideo`                                                                                                              |
| ComfyUI    |     ✓      |       ✓        |       -        | 共有スイープには含まれません。ワークフロー固有のカバレッジは Comfy テストにあります                                                              |
| DeepInfra  |     ✓      |       -        |       -        | `generate`。ネイティブの DeepInfra 動画スキーマは Plugin コントラクトではテキストから動画です                                                     |
| fal        |     ✓      |       ✓        |       ✓        | `generate`、`imageToVideo`。`videoToVideo` は Seedance の参照から動画を使用する場合のみ                                                  |
| Google     |     ✓      |       ✓        |       ✓        | `generate`、`imageToVideo`。現在のバッファーベースの Gemini/Veo スイープがその入力を受け付けないため、共有 `videoToVideo` はスキップされます |
| MiniMax    |     ✓      |       ✓        |       -        | `generate`、`imageToVideo`                                                                                                              |
| OpenAI     |     ✓      |       ✓        |       ✓        | `generate`、`imageToVideo`。この組織/入力パスでは現在プロバイダー側の動画編集アクセスが必要なため、共有 `videoToVideo` はスキップされます   |
| OpenRouter |     ✓      |       ✓        |       -        | `generate`、`imageToVideo`                                                                                                              |
| Qwen       |     ✓      |       ✓        |       ✓        | `generate`、`imageToVideo`。このプロバイダーにはリモートの `http(s)` 動画 URL が必要なため、`videoToVideo` はスキップされます                              |
| Runway     |     ✓      |       ✓        |       ✓        | `generate`、`imageToVideo`。`videoToVideo` は選択されたモデルが `runway/gen4_aleph` の場合のみ実行されます                                     |
| Together   |     ✓      |       ✓        |       -        | `generate`、`imageToVideo`                                                                                                              |
| Vydra      |     ✓      |       ✓        |       -        | `generate`。同梱の `veo3` はテキスト専用で、同梱の `kling` にはリモート画像 URL が必要なため、共有 `imageToVideo` はスキップされます           |
| xAI        |     ✓      |       ✓        |       ✓        | Classic はすべてのモードをサポートします。Video 1.5 は画像から動画のみです。リモート MP4 入力のため `videoToVideo` は共有スイープに含まれません             |

## ツールパラメーター

### 必須

<ParamField path="prompt" type="string" required>
  生成する動画のテキスト説明。`action: "generate"` では必須です。
</ParamField>

### コンテンツ入力

<ParamField path="image" type="string">単一の参照画像（パスまたは URL）。</ParamField>
<ParamField path="images" type="string[]">複数の参照画像（最大 9 個）。</ParamField>
<ParamField path="imageRoles" type="string[]">
結合された画像リストと並行する、位置ごとの省略可能なロールヒント。
正規値: `first_frame`、`last_frame`、`reference_image`。
</ParamField>
<ParamField path="video" type="string">単一の参照動画（パスまたは URL）。</ParamField>
<ParamField path="videos" type="string[]">複数の参照動画（最大 4 個）。</ParamField>
<ParamField path="videoRoles" type="string[]">
結合された動画リストと並行する、位置ごとの省略可能なロールヒント。
正規値: `reference_video`。
</ParamField>
<ParamField path="audioRef" type="string">
単一の参照音声（パスまたは URL）。プロバイダーが音声入力をサポートしている場合、
BGM または音声リファレンスとして使用されます。
</ParamField>
<ParamField path="audioRefs" type="string[]">複数の参照音声（最大 3 個）。</ParamField>
<ParamField path="audioRoles" type="string[]">
結合された音声リストと並行する、位置ごとの省略可能なロールヒント。
正規値: `reference_audio`。
</ParamField>

<Note>
ロールヒントはそのままプロバイダーに転送されます。正規値は
`VideoGenerationAssetRole` ユニオンで定義されていますが、プロバイダーが追加の
ロール文字列を受け入れる場合もあります。`*Roles` 配列の要素数は、
対応する参照リストの要素数を超えてはなりません。1 つずれると、明確なエラーで失敗します。
スロットを未設定のままにするには、空文字列を使用します。xAI で
`reference_images` 生成モードを使用するには、すべての画像ロールを
`reference_image` に設定します。単一画像から動画を生成する場合は、
ロールを省略するか `first_frame` を使用します。
</Note>

### スタイル制御

<ParamField path="aspectRatio" type="string">
  `1:1`、`16:9`、`9:16`、`adaptive`、またはプロバイダー固有の値などのアスペクト比ヒント。OpenClaw は、サポートされていない値をプロバイダーごとに正規化または無視します。
</ParamField>
<ParamField path="resolution" type="string">`360P`、`480P`、`540P`、`720P`、`768P`、`1080P`、`4K`、またはプロバイダー固有の値などの解像度ヒント。OpenClaw は、サポートされていない値をプロバイダーごとに正規化または無視します。</ParamField>
<ParamField path="durationSeconds" type="number">
  目標時間（秒単位。プロバイダーがサポートする最も近い値に丸められます）。
</ParamField>
<ParamField path="size" type="string">プロバイダーがサポートしている場合のサイズヒント。</ParamField>
<ParamField path="audio" type="boolean">
  サポートされている場合、出力で生成音声を有効にします。`audioRef*`（入力）とは異なります。
</ParamField>
<ParamField path="watermark" type="boolean">サポートされている場合、プロバイダーの透かしを切り替えます。</ParamField>

`adaptive` はプロバイダー固有のセンチネルです。機能で
`adaptive` を宣言するプロバイダーにはそのまま転送されます（たとえば BytePlus
Seedance では、入力画像の寸法から比率を自動検出するために使用されます）。
これを宣言しないプロバイダーでは、破棄されたことを確認できるように、
ツール結果の `details.ignoredOverrides` で値が示されます。

### 高度な設定

<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  `"status"` は現在のセッションタスクを返し、`"list"` はプロバイダーを調査します。
</ParamField>
<ParamField path="model" type="string">プロバイダー／モデルのオーバーライド（例: `runway/gen4.5`）。</ParamField>
<ParamField path="filename" type="string">出力ファイル名のヒント。</ParamField>
<ParamField path="timeoutMs" type="number">省略可能なプロバイダー操作のタイムアウト（ミリ秒）。省略した場合、OpenClaw は設定されていれば `agents.defaults.mediaModels.video.timeoutMs` を使用し、それ以外では Plugin 作成者が定義したプロバイダーのデフォルトが存在する場合にそれを使用します。</ParamField>
<ParamField path="providerOptions" type="object">
  JSON オブジェクトとして指定するプロバイダー固有のオプション（例: `{"seed": 42, "draft": true}`）。
  型付きスキーマを宣言するプロバイダーでは、キーと型が検証されます。不明な
  キーや型の不一致がある場合、フォールバック中にその候補はスキップされます。
  スキーマを宣言していないプロバイダーには、オプションがそのまま渡されます。
  各プロバイダーが受け入れる内容を確認するには、`video_generate action=list` を実行します。
</ParamField>

<Note>
すべてのプロバイダーがすべてのパラメーターをサポートしているわけではありません。OpenClaw は、
時間をプロバイダーがサポートする最も近い値に正規化し、フォールバック先のプロバイダーで異なる
制御インターフェースが公開されている場合は、サイズからアスペクト比への変換など、
変換可能なジオメトリヒントを再マッピングします。実際にサポートされていないオーバーライドは
ベストエフォートで無視され、ツール結果で警告として報告されます。厳格な機能上の制限
（参照入力が多すぎる場合など）は、送信前に失敗します。ツール結果には
適用された設定が報告され、`details.normalization` には
要求値から適用値への変換が記録されます。
</Note>

参照入力によってランタイムモードが選択されます。

- 参照メディアなし -> `generate`
- 画像参照あり -> `imageToVideo`
- 動画参照あり -> `videoToVideo`
- 参照音声入力は、解決されたモードを**変更しません**。画像／動画参照によって
  選択されたモードに追加で適用され、`maxInputAudios` を宣言する
  プロバイダーでのみ機能します。

画像と動画の参照を混在させることは、安定した共通機能ではありません。
リクエストごとに 1 種類の参照を使用することを推奨します。

#### フォールバックと型付きオプション

一部の機能チェックはツール境界ではなくフォールバック層で適用されるため、
プライマリプロバイダーの制限を超えるリクエストでも、対応可能なフォールバックで
実行できる場合があります。

- 音声参照を含むリクエストで、アクティブな候補が `maxInputAudios`（または `0`）を宣言していない場合、
  その候補はスキップされ、次の候補が試されます。同じ
  ガードが、`maxInputImages`／`maxInputVideos` に対する画像および動画の参照数にも
  適用されます。
- アクティブな候補の `maxDurationSeconds` が要求された `durationSeconds` を下回り、
  `supportedDurationSeconds` リストも宣言されていない場合、その候補はスキップされます。
- リクエストに `providerOptions` が含まれ、アクティブな候補が型付き
  `providerOptions` スキーマを明示的に宣言している場合、指定されたキーが
  スキーマにないか、値の型が一致しなければスキップされます。スキーマを
  宣言していないプロバイダーには、オプションがそのまま渡されます（後方互換性のある
  パススルー）。プロバイダーは空のスキーマ
  （`capabilities.providerOptions: {}`）を宣言することで、すべてのプロバイダーオプションを無効化できます。
  この場合も、型の不一致と同様にスキップされます。

リクエスト内の最初のスキップ理由は `warn` でログに記録されるため、
プライマリプロバイダーが見送られたことを運用者が確認できます。後続のスキップは、
長いフォールバックチェーンでログが過剰にならないよう `debug` で記録されます。
すべての候補がスキップされた場合、集約エラーには各候補のスキップ理由が含まれます。

## アクション

| アクション     | 動作                                                                                             |
| ---------- | -------------------------------------------------------------------------------------------------------- |
| `generate` | デフォルト。指定したプロンプトと省略可能な参照入力から動画を作成します。                             |
| `status`   | 新しい生成を開始せずに、現在のセッションで実行中の動画タスクの状態を確認します。 |
| `list`     | 利用可能なプロバイダー、モデル、およびその機能を表示します。                                                |

## モデルの選択

OpenClaw は次の順序でモデルを解決します。

1. **`model` ツールパラメーター** - エージェントが呼び出しで指定した場合。
2. 設定の **`videoGenerationModel.primary`**。
3. 順番に **`videoGenerationModel.fallbacks`**。
4. **自動検出** - 有効な認証を持つプロバイダー。現在の
   デフォルトプロバイダーから開始し、残りのプロバイダーをアルファベット順に
   試します。

プロバイダーが失敗すると、次の候補が自動的に試されます。すべての
候補が失敗した場合、エラーには各試行の詳細が含まれます。

認証済みプロバイダー間の自動フォールバックは常に有効です。呼び出しごとの
`model` が優先されます。

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
        fallbacks: ["runway/gen4.5", "qwen/wan2.6-t2v"],
        timeoutMs: 180000, // ツールごとのプロバイダーリクエストタイムアウトの省略可能なオーバーライド
      },
    },
  },
}
```

## プロバイダーに関する注意事項

<AccordionGroup>
  <Accordion title="Alibaba">
    DashScope／Model Studio の非同期エンドポイントを使用します。参照画像と
    動画はリモートの `http(s)` URL である必要があります。
  </Accordion>
  <Accordion title="BytePlus（バンドル）">
    プロバイダー ID: `byteplus`。

    モデル: `seedance-1-0-pro-250528`（デフォルト）、
    `seedance-1-5-pro-251215`。

    統合 `content[]` API を使用します。最大 2 枚の入力画像
    （`first_frame` + `last_frame`）をサポートします。画像を位置順に渡すか、各画像の
    `role` を明示的に設定します。

    サポートされる `providerOptions` キー: `seed`（数値）、`draft`（ブール値 -
    480p を強制）、`camera_fixed`（ブール値）。

  </Accordion>
  <Accordion title="BytePlus Seedance 1.5 Plugin">
    [`@openclaw/byteplus-modelark`](https://www.npmjs.com/package/@openclaw/byteplus-modelark)
    Plugin（外部、非バンドル）が必要です。プロバイダー ID: `byteplus-seedance15`。モデル:
    `seedance-1-5-pro-251215`。

    統合 `content[]` API を使用します。最大 2 枚の入力画像
    （`first_frame` + `last_frame`）をサポートします。すべての入力はリモートの `https://`
    URL である必要があります。各画像に `role: "first_frame"`／`"last_frame"` を設定するか、
    画像を位置順に渡します。

    `aspectRatio: "adaptive"` は入力画像から比率を自動検出します。
    `audio: true` は `generate_audio` にマッピングされます。`providerOptions.seed`
    （数値）は転送されます。

  </Accordion>
  <Accordion title="BytePlus Seedance 2.0">
    [`@openclaw/byteplus-modelark`](https://www.npmjs.com/package/@openclaw/byteplus-modelark)
    Plugin（外部、非バンドル）が必要です。プロバイダー ID: `byteplus-seedance2`。モデル:
    `dreamina-seedance-2-0-260128`、
    `dreamina-seedance-2-0-fast-260128`。

    統合 `content[]` API を使用します。最大 9 枚の参照画像、
    3 本の参照動画、3 個の参照音声をサポートします。すべての入力はリモートの
    `https://` URL である必要があります。各アセットに `role` を設定します。サポートされる値:
    `"first_frame"`、`"last_frame"`、`"reference_image"`、
    `"reference_video"`、`"reference_audio"`。

    `aspectRatio: "adaptive"` は入力画像から比率を自動検出します。
    `audio: true` は `generate_audio` にマッピングされます。`providerOptions.seed`
    （数値）は転送されます。

  </Accordion>
  <Accordion title="ComfyUI">
    ワークフロー駆動のローカルまたはクラウド実行。構成されたグラフを通じて、
    テキストから動画および画像から動画への生成をサポートします。
  </Accordion>
  <Accordion title="fal">
    長時間実行ジョブにはキューを利用するフローを使用します。OpenClaw は、処理中の fal キュージョブを
    タイムアウトとして扱うまで、デフォルトで最大 20 分待機します。ほとんどの fal 動画モデルは、
    単一の画像参照を受け付けます。Seedance 2.0 の参照から動画への生成モデルは、
    最大 9 枚の画像、3 本の動画、3 件の音声参照を受け付け、
    参照ファイルの合計は最大 12 件です。
  </Accordion>
  <Accordion title="Google (Gemini / Veo)">
    1 件の画像参照または 1 件の動画参照をサポートします。現在の Veo 動画生成では
    Gemini API が `generateAudio` パラメーターを拒否するため、Gemini API 経由の
    音声生成リクエストは警告とともに無視されます。
  </Accordion>
  <Accordion title="MiniMax">
    画像参照は 1 件のみです。MiniMax は `768P` および `1080P`
    の解像度を受け付けます。`720P` などのリクエストは、送信前に最も近い
    サポート対象値へ正規化されます。
  </Accordion>
  <Accordion title="OpenAI">
    `size` オーバーライドのみが転送されます。その他のスタイルオーバーライド
    （`aspectRatio`、`resolution`、`audio`、`watermark`）は、
    警告とともに無視されます。
  </Accordion>
  <Accordion title="OpenRouter">
    OpenRouter の非同期 `/videos` API を使用します。OpenClaw はジョブを送信し、
    `polling_url` をポーリングして、`unsigned_urls` または文書化されたジョブコンテンツの
    エンドポイントからダウンロードします。同梱の `google/veo-3.1-fast` のデフォルトでは、
    4/6/8 秒の長さ、`720P`/`1080P` の解像度、
    `16:9`/`9:16` のアスペクト比が提示されます。
  </Accordion>
  <Accordion title="Qwen">
    Alibaba と同じ DashScope バックエンドを使用します。参照入力にはリモートの
    `http(s)` URL が必要です。ローカルファイルは事前に拒否されます。
  </Accordion>
  <Accordion title="Runway">
    データ URI を介してローカルファイルをサポートします。動画から動画への変換には
    `runway/gen4_aleph` が必要です。テキストのみの実行では、`16:9` および
    `9:16` のアスペクト比を利用できます。
  </Accordion>
  <Accordion title="Together">
    画像参照は 1 件のみです。
  </Accordion>
  <Accordion title="Vydra">
    認証情報が失われるリダイレクトを避けるため、`https://www.vydra.ai/api/v1` を直接使用します。
    `veo3` はテキストから動画への生成専用として同梱されています。
    `kling` にはリモート画像 URL が必要です。
  </Accordion>
  <Accordion title="xAI">
    デフォルトの `grok-imagine-video` モデルは、テキストから動画への生成、最初のフレームとなる
    単一画像から動画への生成、xAI `reference_images` を介した最大 7 件の
    `reference_image` 入力、およびリモート動画の編集・延長フローをサポートします。
    生成のデフォルトは `480P` です。単一画像から動画への生成では、
    `aspectRatio` を省略すると元画像の比率を継承します。動画の編集・延長では
    入力のジオメトリを継承し、アスペクト比や解像度のオーバーライドは受け付けません。
    延長では 2～10 秒を指定できます。

    `grok-imagine-video-1.5` は画像から動画への生成専用です。画像をちょうど 1 枚指定してください。
    1～15 秒と `480P`、`720P`、または `1080P` をサポートし、
    デフォルトは `480P` です。元画像の比率を継承するには
    `aspectRatio` を省略します。プレビュー識別子と日付付き 1.5 識別子には
    同じ検証が適用され、変更されずに転送されます。

  </Accordion>
</AccordionGroup>

## プロバイダーの機能モード

共有の動画生成コントラクトは、フラットな集約上限だけでなく、
モード固有の機能をサポートします。新しいプロバイダー実装では、
明示的なモードブロックを優先してください。

```typescript
capabilities: {
  generate: {
    maxVideos: 1,
    maxDurationSeconds: 10,
    supportsResolution: true,
  },
  imageToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputImages: 1,
    maxInputImagesByModel: { "provider/reference-to-video": 9 },
    maxDurationSeconds: 5,
  },
  videoToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputVideos: 1,
    maxDurationSeconds: 5,
  },
}
```

`maxInputImages` や `maxInputVideos` などのフラットな集約フィールドだけでは、
変換モードのサポートを提示するには**不十分**です。ライブテスト、コントラクトテスト、
共有の `video_generate` ツールがモードのサポートを決定論的に検証できるよう、
プロバイダーは `generate`、`imageToVideo`、`videoToVideo` を
明示的に宣言してください。

プロバイダー内の 1 つのモデルだけが他より広範な参照入力をサポートする場合は、
モード全体の上限を引き上げる代わりに、`maxInputImagesByModel`、
`maxInputVideosByModel`、または `maxInputAudiosByModel` を使用してください。

## ライブテスト

共有の同梱プロバイダー向けのオプトイン式ライブカバレッジ：

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts
```

リポジトリラッパー：

```bash
pnpm test:live:media video
```

このライブファイルはデフォルトで、保存済み認証プロファイルよりも
すでにエクスポートされているプロバイダー環境変数を優先し、
リリースに安全なスモークテストを実行します。

- `generate`：スイープ内の FAL 以外の各プロバイダー。
- 1 秒の Lobster プロンプト。
- `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` によるプロバイダーごとの操作上限
  （デフォルトは `180000`）。

プロバイダー側のキュー待ち時間がリリース所要時間の大部分を占める可能性があるため、
FAL はオプトインです。

```bash
pnpm test:live:media video --video-providers fal
```

共有スイープがローカルメディアを使用して安全に実行できる、宣言済みの
変換モードも実行するには、`OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1` を設定します。

- `capabilities.imageToVideo.enabled` の場合は `imageToVideo`。
- `capabilities.videoToVideo.enabled` であり、共有スイープにおいて
  プロバイダーまたはモデルがバッファーを基盤とするローカル動画入力を受け付ける場合は
  `videoToVideo`。

現在、共有の `videoToVideo` ライブレーンで `runway` が対象になるのは、
`runway/gen4_aleph` を選択した場合のみです。

## 構成

OpenClaw の構成でデフォルトの動画生成モデルを設定します。

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-r2v-flash"],
      },
    },
  },
}
```

または CLI を使用します。

```bash
openclaw config set agents.defaults.mediaModels.video.primary "qwen/wan2.6-t2v"
```

## 関連項目

- [Alibaba Model Studio](/ja-JP/providers/alibaba)
- [バックグラウンドタスク](/ja-JP/automation/tasks) - 非同期動画生成のタスク追跡
- [BytePlus](/ja-JP/concepts/model-providers#byteplus-international)
- [ComfyUI](/ja-JP/providers/comfy)
- [構成リファレンス](/ja-JP/gateway/config-agents#agent-defaults)
- [fal](/ja-JP/providers/fal)
- [Google (Gemini)](/ja-JP/providers/google)
- [MiniMax](/ja-JP/providers/minimax)
- [モデル](/ja-JP/concepts/models)
- [OpenAI](/ja-JP/providers/openai)
- [Qwen](/ja-JP/providers/qwen)
- [Runway](/ja-JP/providers/runway)
- [Together AI](/ja-JP/providers/together)
- [ツールの概要](/ja-JP/tools)
- [Vydra](/ja-JP/providers/vydra)
- [xAI](/ja-JP/providers/xai)
