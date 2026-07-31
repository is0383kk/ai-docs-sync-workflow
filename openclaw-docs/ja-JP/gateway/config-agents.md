---
read_when:
    - エージェントのデフォルト設定を調整する（モデル、思考、ワークスペース、Heartbeat、メディア、Skills）
    - マルチエージェントのルーティングとバインディングの設定
    - セッション、メッセージ配信、トークモードの動作を調整する
summary: エージェントのデフォルト、マルチエージェントルーティング、セッション、メッセージ、トーク設定
title: 設定 — エージェント
x-i18n:
    generated_at: "2026-07-26T10:00:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7a161d65b02e3333c15a2d998421419ee37d36be4d02ebb3a86e66282df06adb
    source_path: gateway/config-agents.md
    workflow: 16
---

`agents.*`、`multiAgent.*`、`session.*`、
`messages.*`、`talk.*` 配下のエージェント単位の設定キー。
チャンネル、ツール、Gateway ランタイム、およびその他のトップレベルキーについては、[設定リファレンス](/ja-JP/gateway/configuration-reference)を参照してください。

## エージェントのデフォルト

### `agents.defaults.workspace`

デフォルト: `OPENCLAW_WORKSPACE_DIR` が設定されている場合はその値。それ以外は `~/.openclaw/workspace`（`OPENCLAW_PROFILE` がデフォルト以外のプロファイルに設定されている場合は `~/.openclaw/workspace-<profile>`）。

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

明示的な `agents.defaults.workspace` の値は
`OPENCLAW_WORKSPACE_DIR` より優先されます。設定にパスを書き込みたくない場合は、環境変数を使用してデフォルトのエージェントがマウント済みワークスペースを参照するようにします。

### `agents.defaults.repoRoot`

システムプロンプトの Runtime 行に表示される、オプションのリポジトリルート。未設定の場合、OpenClaw はワークスペースから上位へたどって自動検出します。

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

`agents.entries.*.skills` を設定していないエージェントに適用する、オプションのデフォルト Skills 許可リスト。

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // github、weather を継承
      { id: "docs", skills: ["docs-search"] }, // デフォルトを置換
      { id: "locked-down", skills: [] }, // Skills なし
    ],
  },
}
```

- デフォルトで Skills を無制限にするには、`agents.defaults.skills` を省略します。
- デフォルトを継承するには、`agents.entries.*.skills` を省略します。
- Skills を無効にするには、`agents.entries.*.skills: []` を設定します。
- 空でない `agents.entries.*.skills` リストが、そのエージェントの最終的なセットになります。デフォルトとは
  マージされません。

### `agents.defaults.skipBootstrap`

ワークスペースのブートストラップファイル（`AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`BOOTSTRAP.md`）の自動作成を無効にします。

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.skipOptionalBootstrapFiles`

必須のブートストラップファイル（`AGENTS.md`、`TOOLS.md`、`BOOTSTRAP.md`）は引き続き書き込みながら、選択したオプションのワークスペースファイルの作成をスキップします。有効な値: `SOUL.md`、`USER.md`、`IDENTITY.md`（`HEARTBEAT.md` も受け付けますが、Heartbeat コンテキストは Cron モニターのスクラッチ領域へ移動したため何も行いません）。

```json5
{
  agents: {
    defaults: {
      skipOptionalBootstrapFiles: ["SOUL.md", "USER.md"],
    },
  },
}
```

### `agents.defaults.contextInjection`

ワークスペースのブートストラップファイルをシステムプロンプトに注入するタイミングを制御します。デフォルト: `"always"`。

- `"continuation-skip"`: 安全に継続できるターン（アシスタントの応答完了後）では、ワークスペースのブートストラップを再注入せず、プロンプトサイズを削減します。Heartbeat の実行と Compaction 後の再試行では、引き続きコンテキストを再構築します。
- `"never"`: すべてのターンでワークスペースのブートストラップとコンテキストファイルの注入を無効にします。プロンプトのライフサイクルを完全に所有するエージェント（カスタムコンテキストエンジン、独自にコンテキストを構築するネイティブランタイム、ブートストラップを使用しない特殊なワークフロー）でのみ使用してください。Heartbeat および Compaction 復旧ターンでも注入をスキップします。

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

エージェント単位のオーバーライド: `agents.entries.*.contextInjection`。省略した値は
`agents.defaults.contextInjection` を継承します。

### `agents.defaults.bootstrapMaxChars`

切り詰め前の、ワークスペースのブートストラップファイルごとの最大文字数。デフォルト: `20000`。

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

エージェント単位のオーバーライド: `agents.entries.*.bootstrapMaxChars`。省略した値は
`agents.defaults.bootstrapMaxChars` を継承します。

### `agents.defaults.bootstrapTotalMaxChars`

すべてのワークスペースのブートストラップファイルから注入される合計最大文字数。デフォルト: `60000`。

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 60000 } },
}
```

エージェント単位のオーバーライド: `agents.entries.*.bootstrapTotalMaxChars`。省略した値は
`agents.defaults.bootstrapTotalMaxChars` を継承します。

### エージェント単位のブートストラッププロファイルのオーバーライド

あるエージェントで共有デフォルトとは異なるプロンプト注入動作が必要な場合は、エージェント単位のブートストラッププロファイルのオーバーライドを使用します。省略したフィールドは
`agents.defaults` から継承されます。

```json5
{
  agents: {
    defaults: {
      contextInjection: "continuation-skip",
      bootstrapMaxChars: 20000,
      bootstrapTotalMaxChars: 60000,
    },
    list: [
      {
        id: "strict-worker",
        contextInjection: "always",
        bootstrapMaxChars: 50000,
        bootstrapTotalMaxChars: 300000,
      },
    ],
  },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

ブートストラップコンテキストが切り詰められたとき、エージェントに表示されるシステムプロンプト通知を制御します。
デフォルト: `"always"`。

- `"off"`: 切り詰め通知のテキストをシステムプロンプトに注入しません。
- `"once"`: 一意の切り詰めシグネチャごとに、簡潔な通知を一度だけ注入します。
- `"always"`: 切り詰めが存在する場合、実行のたびに簡潔な通知を注入します（推奨）。

詳細な元データ／注入後の件数と設定調整フィールドは、コンテキスト／ステータスレポートやログなどの診断情報に保持されます。通常の WebChat のユーザー／ランタイムコンテキストには、簡潔な復旧通知のみが表示されます。

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "always" } }, // off | once | always
}
```

### コンテキスト予算の所有権マップ

OpenClaw には大容量のプロンプト／コンテキスト予算が複数あり、1 つの汎用設定項目にすべてを集約するのではなく、意図的にサブシステムごとに分割されています。

| 予算                                                         | 対象                                                                                                                                                          |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agents.defaults.bootstrapMaxChars` / `bootstrapTotalMaxChars` | 通常のワークスペースブートストラップ注入                                                                                                                            |
| `agents.defaults.startupContext.*`                             | 最近の日次 `memory/*.md` ファイルを含む、リセット／起動時の単発モデル実行プレリュード。単独のチャット `/new` および `/reset` はモデルを呼び出さずに受領確認されます |
| `skills.limits.*`                                              | システムプロンプトに注入されるコンパクトな Skills リスト                                                                                                         |
| `agents.defaults.contextLimits.*`                              | 上限付きのランタイム抜粋と、注入されるランタイム所有ブロック                                                                                                      |
| `memory.qmd.limits.*`                                          | インデックス化されたメモリ検索スニペットと注入サイズ                                                                                                              |

対応するエージェント単位のオーバーライド:

- `agents.entries.*.skillsLimits.maxSkillsPromptChars`
- `agents.entries.*.contextInjection`
- `agents.entries.*.bootstrapMaxChars`
- `agents.entries.*.bootstrapTotalMaxChars`
- `agents.entries.*.contextLimits.*`

#### `agents.defaults.startupContext`

リセット／起動時のモデル実行で最初のターンに注入される起動プレリュードを制御します。
単独のチャット `/new` および `/reset` コマンドは、モデルを呼び出さずにリセットを受領確認するため、このプレリュードを読み込みません。

```json5
{
  agents: {
    defaults: {
      startupContext: {
        enabled: true,
        applyOn: ["new", "reset"],
        dailyMemoryDays: 2,
        maxFileBytes: 16384,
        maxFileChars: 1200,
        maxTotalChars: 2800,
      },
    },
  },
}
```

#### `agents.defaults.contextLimits`

上限付きランタイムコンテキスト領域の共有デフォルト。

```json5
{
  agents: {
    defaults: {
      contextLimits: {
        memoryGetMaxChars: 12000,
        postCompactionMaxChars: 1800,
      },
    },
  },
}
```

- `memoryGetMaxChars`: 切り詰めメタデータと継続通知が追加される前の、デフォルトの `memory_get` 抜粋上限。
- `memory_get` で `lines` が省略されている場合、OpenClaw は組み込みの 120 行ウィンドウを使用し、
  その後 `memoryGetMaxChars` を適用します。
- ライブツール結果には、モデルコンテキストに応じた自動上限が適用されます。100K トークン未満では `16000` 文字、100K+ トークンでは `32000` 文字、200K+ トークンでは `64000` 文字です。
- `postCompactionMaxChars`: Compaction 後の更新注入時に使用される AGENTS.md 抜粋上限。

#### `agents.entries.*.contextLimits`

共有 `contextLimits` 設定項目のエージェント単位のオーバーライド。省略したフィールドは
`agents.defaults.contextLimits` から継承されます。

```json5
{
  agents: {
    defaults: {
      contextLimits: { memoryGetMaxChars: 12000 },
    },
    list: [
      {
        id: "tiny-local",
        contextLimits: {
          memoryGetMaxChars: 6000,
        },
      },
    ],
  },
}
```

#### `skills.limits.maxSkillsPromptChars`

システムプロンプトに注入されるコンパクトな Skills リストのグローバル上限。オンデマンドでの `SKILL.md` ファイルの読み取りには影響しません。

```json5
{
  skills: { limits: { maxSkillsPromptChars: 18000 } },
}
```

#### `agents.entries.*.skillsLimits.maxSkillsPromptChars`

Skills プロンプト予算のエージェント単位のオーバーライド。

```json5
{
  agents: {
    list: [{ id: "tiny-local", skillsLimits: { maxSkillsPromptChars: 6000 } }],
  },
}
```

### `agents.defaults.imageMaxDimensionPx`

プロバイダー呼び出し前に、トランスクリプト／ツールの画像ブロック内で最も長い画像辺に適用する最大ピクセルサイズ。
デフォルト: `1200`。

値を小さくすると通常、スクリーンショットが多い実行でビジョントークン使用量とリクエストペイロードサイズが減少します。
値を大きくすると、より多くの視覚的詳細が保持されます。

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.imageQuality`

ファイルパス、URL、メディア参照から読み込まれる画像に対する、画像ツールの圧縮／詳細度の設定。
デフォルト: `auto`。

OpenClaw は、選択された画像モデルに応じてリサイズ段階を調整します。たとえば Claude Opus 4.8、OpenAI GPT-5.6 Sol、Qwen VL、およびホスト型 Llama 4 ビジョンモデルでは、従来またはデフォルトの高詳細ビジョン処理より大きな画像を使用できます。一方、複数画像のターンでは、トークンとレイテンシのコストを抑えるため、`auto` モードでより積極的に圧縮されます。

値:

- `auto`: モデルの制限と画像数に適応します。
- `efficient`: トークンとバイトの使用量を抑えるため、小さい画像を優先します。
- `balanced`: 標準的な中間段階を使用します。
- `high`: スクリーンショット、図、文書画像でより多くの詳細を保持します。

```json5
{
  agents: { defaults: { imageQuality: "auto" } },
}
```

### `agents.defaults.userTimezone`

システムプロンプトのコンテキストに使用するタイムゾーン（メッセージのタイムスタンプには使用されません）。ホストのタイムゾーンにフォールバックします。

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

システムプロンプト内の時刻形式。デフォルト: `auto`（OS の設定）。

```json5
{
  agents: { defaults: { timeFormat: "auto" } }, // auto | 12 | 24
}
```

### `agents.defaults.model`

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
      utilityModel: "openai/gpt-5.4-mini",
      imageModel: {
        primary: "openrouter/qwen/qwen-2.5-vl-72b-instruct:free",
        fallbacks: ["openrouter/google/gemini-2.0-flash-vision:free"],
      },
      mediaModels: {
        image: {
          primary: "openai/gpt-image-2",
          fallbacks: ["google/gemini-3.1-flash-image"],
        },
        video: {
          primary: "qwen/wan2.6-t2v",
          fallbacks: ["qwen/wan2.6-i2v"],
        },
      },
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      params: { cacheRetention: "long" }, // プロバイダーのグローバルデフォルトパラメータ
      pdfMaxMb: 10,
      pdfMaxPages: 20,
      thinkingDefault: "low",
      verboseDefault: "off",
      toolProgressDetail: "explain",
      reasoningDefault: "off",
      elevatedDefault: "on",
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      contextTokens: 200000,
      maxConcurrent: 4,
    },
  },
}
```

- `model`: 文字列（`"provider/model"`）またはオブジェクト（`{ primary, fallbacks }`）を受け入れます。
  - 文字列形式では、プライマリモデルのみを設定します。
  - オブジェクト形式では、プライマリモデルと、順序付けられたフェイルオーバーモデルを設定します。
- `utilityModel`: 短い内部タスク用の任意の `provider/model` 参照またはエイリアスです。現在は、生成される Control UI セッションタイトル、Telegram DM トピックタイトル、Discord 自動スレッドタイトル、および[進捗ドラフトのナレーション](/ja-JP/concepts/progress-drafts#narrated-status)に使用されます。未設定の場合、OpenClaw は、プライマリプロバイダーに宣言済みの小規模モデルのデフォルトが存在すれば、それを導出します（OpenAI → `gpt-5.6-luna`、Anthropic → `claude-haiku-4-5`）。それ以外の場合、タイトルタスクではエージェントのプライマリモデルを使用し、ナレーションは無効のままになります。個別のユーティリティモデルで生成タイトルを準備または完成できない場合、OpenClaw はそのタイトルについてプライマリモデルで一度再試行します。ダッシュボードタイトルでは、ユーティリティモデルの自動導出と通常のフォールバックに、有効なセッションプロバイダーと認証プロファイルが使用されます。明示的なユーティリティモデルでは、設定済みのプロバイダーと認証が維持されます。代替ユーティリティ経路をスキップするには `utilityModel: ""` を設定します。ダッシュボードタイトルの生成は、引き続き通常のセッションモデルで直接実行されます。`agents.entries.*.utilityModel` はデフォルトを上書きし、処理固有のモデルオーバーライドはその両方より優先されます。ユーティリティタスクは個別にモデルを呼び出し、タスク固有の内容を選択したモデルプロバイダーへ送信します。ダッシュボードタイトルの生成では、コマンドではない最初のメッセージの先頭最大 1,000 文字が送信されます。ナレーションでは、受信リクエストと、簡潔に編集されたツール概要が送信されます。コストとデータ処理の要件に合うプロバイダーを選択してください。
- `imageModel`: 文字列（`"provider/model"`）またはオブジェクト（`{ primary, fallbacks }`）を受け入れます。
  - アクティブなモデルが画像を受け入れられない場合、`image` ツール経路でビジョンモデル設定として使用されます。ネイティブにビジョンをサポートするモデルには、読み込まれた画像バイトが代わりに直接渡されます。
  - 選択したモデルまたはデフォルトモデルが画像入力を受け入れられない場合のフォールバックルーティングにも使用されます。
  - 明示的な `provider/model` 参照を推奨します。互換性のため、修飾されていない ID も受け入れられます。修飾されていない ID が `models.providers.*.models` に設定された画像対応エントリの 1 つと一意に一致する場合、OpenClaw はその ID にプロバイダーを付加します。設定済みエントリとの一致が曖昧な場合は、明示的なプロバイダープレフィックスが必要です。
- `mediaModels.image`: 文字列（`"provider/model"`）またはオブジェクト（`{ primary, fallbacks }`）を受け入れます。
  - 共有の画像生成機能と、今後追加される画像生成ツールまたは Plugin のサーフェスで使用されます。
  - 一般的な値: Gemini のネイティブ画像生成には `google/gemini-3.1-flash-image`、fal には `fal/fal-ai/flux/dev`、OpenAI Images には `openai/gpt-image-2`、背景が透明な OpenAI PNG/WebP 出力には `openai/gpt-image-1.5` を使用します。
  - プロバイダーまたはモデルを直接選択する場合、対応するプロバイダー認証も設定してください（例: `google/*` には `GEMINI_API_KEY` または `GOOGLE_API_KEY`、`openai/gpt-image-2` / `openai/gpt-image-1.5` には `OPENAI_API_KEY` または OpenAI Codex OAuth、`fal/*` には `FAL_KEY`）。
  - 省略した場合でも、`image_generate` は認証情報に基づいてプロバイダーのデフォルトを推測できます。最初に現在のデフォルトプロバイダーを試し、続いて登録済みの残りの画像生成プロバイダーをプロバイダー ID 順に試します。
- `mediaModels.music`: 文字列（`"provider/model"`）またはオブジェクト（`{ primary, fallbacks }`）を受け入れます。
  - 共有の音楽生成機能と、組み込みの `music_generate` ツールで使用されます。
  - 一般的な値: `google/lyria-3-clip-preview`、`google/lyria-3-pro-preview`、または `minimax/music-2.6`。
  - 省略した場合でも、`music_generate` は認証情報に基づいてプロバイダーのデフォルトを推測できます。最初に現在のデフォルトプロバイダーを試し、続いて登録済みの残りの音楽生成プロバイダーをプロバイダー ID 順に試します。
  - プロバイダーまたはモデルを直接選択する場合、対応するプロバイダー認証/API キーも設定してください。
- `mediaModels.video`: 文字列（`"provider/model"`）またはオブジェクト（`{ primary, fallbacks }`）を受け入れます。
  - 共有の動画生成機能と、組み込みの `video_generate` ツールで使用されます。
  - 一般的な値: `qwen/wan2.6-t2v`、`qwen/wan2.6-i2v`、`qwen/wan2.6-r2v`、`qwen/wan2.6-r2v-flash`、または `qwen/wan2.7-r2v`。
  - 省略した場合でも、`video_generate` は認証情報に基づいてプロバイダーのデフォルトを推測できます。最初に現在のデフォルトプロバイダーを試し、続いて登録済みの残りの動画生成プロバイダーをプロバイダー ID 順に試します。
  - プロバイダーまたはモデルを直接選択する場合、対応するプロバイダー認証/API キーも設定してください。
  - 公式の Qwen 動画生成 Plugin は、最大 1 本の出力動画、1 枚の入力画像、4 本の入力動画、10 秒の長さ、およびプロバイダーレベルの `size`、`aspectRatio`、`resolution`、`audio`、`watermark` オプションをサポートします。
- `pdfModel`: 文字列（`"provider/model"`）またはオブジェクト（`{ primary, fallbacks }`）を受け入れます。
  - モデルルーティングのために `pdf` ツールで使用されます。
  - 省略した場合、PDF ツールは `imageModel`、続いて解決済みのセッションモデルまたはデフォルトモデルへフォールバックします。
- `pdfMaxMb`: 呼び出し時に `maxBytesMb` が渡されなかった場合に、`pdf` ツールで使用されるデフォルトの PDF サイズ上限です。
- `pdfMaxPages`: `pdf` ツールの抽出フォールバックモードで考慮されるデフォルトの最大ページ数です。
- `verboseDefault`: エージェントのデフォルトの詳細出力レベルです。値: `"off"`、`"on"`、`"full"`。デフォルト: `"off"`。
- `toolProgressDetail`: `/verbose` ツールの概要と進捗ドラフトのツール行の詳細モードです。値: `"explain"`（デフォルト、簡潔で人間が読めるラベル）または `"raw"`（利用可能な場合に未加工のコマンドや詳細を追加）。エージェントごとの `agents.entries.*.toolProgressDetail` は、このデフォルトを上書きします。
- `reasoningDefault`: エージェントのデフォルトの推論表示設定です。値: `"off"`、`"on"`、`"stream"`。エージェントごとの `agents.entries.*.reasoningDefault` は、このデフォルトを上書きします。設定された推論のデフォルトは、メッセージ単位またはセッション単位の推論オーバーライドが設定されていない場合に限り、所有者、承認済み送信者、またはオペレーター管理者の Gateway コンテキストにのみ適用されます。
- `elevatedDefault`: エージェントのデフォルトの昇格出力レベルです。値: `"off"`、`"on"`、`"ask"`、`"full"`。デフォルト: `"on"`。
- `model.primary`: 形式は `provider/model`（例: Codex OAuth アクセスの場合は `openai/gpt-5.6-sol`）。プロバイダーを省略すると、OpenClaw は最初にエイリアスを試し、次にその正確なモデル ID と一意に一致する設定済みプロバイダーを試し、その後に限り設定済みのデフォルトプロバイダーへフォールバックします（非推奨の互換動作であるため、明示的な `provider/model` を推奨します）。そのプロバイダーが設定済みのデフォルトモデルを提供しなくなった場合、OpenClaw は削除済みプロバイダーの古いデフォルトをエラーとして提示する代わりに、最初に設定されたプロバイダー/モデルへフォールバックします。
- `contextTokens`: エージェント全体に適用できる任意の上限です。より大きなモデルの有効な予算を引き下げることはできますが、モデルを設定済みまたは検出済みの `contextTokens` より引き上げることはできません。個別の OpenAI モデルで、より大きなネイティブウィンドウを有効にするには、そのモデルに `models.providers.openai.models[].contextWindow` と `contextTokens` を設定します。[OpenAI のコンテキストウィンドウのデフォルト](/ja-JP/providers/openai#context-window-defaults-and-long-context-opt-in)を参照してください。
- `models`: 設定済みのエイリアスとモデルごとの設定です。各エントリには、`alias`（ショートカット）と `params`（プロバイダー固有。例: `temperature`、`maxTokens`、`cacheRetention`、`context1m`、`responsesServerCompaction`、`responsesCompactThreshold`、OpenRouter の `provider` ルーティング、`chat_template_kwargs`、`extra_body`/`extraBody`）を含められます。エントリを追加しても、モデルのオーバーライドは制限されません。
  - 選択したプロバイダーで検出されたすべてのモデルを、各モデル ID を手動で列挙せずに表示するには、`"openai/*": {}` や `"vllm/*": {}` などの `provider/*` エントリを使用します。
  - そのプロバイダーで動的に検出されたすべてのモデルに同じランタイムを使用する場合は、`provider/*` エントリに `agentRuntime` を追加します。正確な `provider/model` ランタイムポリシーが、引き続きワイルドカードより優先されます。
  - 安全なメタデータ編集: エントリの追加には `openclaw config set agents.defaults.models '<json>' --strict-json --merge` を使用します。`--replace` を渡さない限り、`config set` は既存のエントリを削除する置換を拒否します。
- `modelPolicy.allow`: 明示的なオーバーライド許可リストです。エイリアス、正確な `provider/model` 参照、および `openai/*` や `clawrouter/anthropic/*` などの末尾プレフィックスワイルドカードを受け入れます。すべてのモデルを許可するには、省略するか `[]` を使用します。`agents.entries.*.modelPolicy.allow` は、そのエージェントのデフォルトポリシーを置き換えます。明示的な空のリストを指定すると、そのエージェントではすべてが許可されます。
  - プロバイダー単位の設定/オンボーディングフローでは、選択したプロバイダーのモデルがこのマップにマージされ、設定済みの無関係なプロバイダーは保持されます。
  - OpenAI Responses の直接モデルでは、サーバー側の Compaction が自動的に有効になります。`context_management` の挿入を停止するには `params.responsesServerCompaction: false` を使用し、しきい値を上書きするには `params.responsesCompactThreshold` を使用します。[OpenAI のサーバー側 Compaction](/ja-JP/providers/openai#advanced-configuration)を参照してください。
- `params`: すべてのモデルに適用されるグローバルなデフォルトプロバイダーパラメーターです。`agents.defaults.params` で設定します（例: `{ cacheRetention: "long" }`）。
- `params` のマージ優先順位（設定）: `agents.defaults.params`（グローバルベース）は `agents.defaults.models["provider/model"].params`（モデル単位）で上書きされ、続いて `agents.entries.*.params`（一致するエージェント ID）がキー単位で上書きします。詳細は[プロンプトキャッシュ](/ja-JP/reference/prompt-caching)を参照してください。
- `models.providers.openrouter.params.provider`: OpenRouter 全体に適用されるデフォルトのプロバイダールーティングポリシーです。OpenClaw はこれを OpenRouter のリクエストの `provider` オブジェクトへ転送します。モデルごとの `agents.defaults.models["openrouter/<model>"].params.provider` とエージェントのパラメーターがキー単位で上書きします。[OpenRouter のプロバイダールーティング](/ja-JP/providers/openrouter#advanced-configuration)を参照してください。
- `params.extra_body`/`params.extraBody`: OpenAI 互換プロキシ向けの `api: "openai-completions"` リクエスト本文へマージされる、高度なパススルー JSON です。生成されたリクエストキーと競合する場合は追加本文が優先され、ネイティブではない completions 経路では、その後も OpenAI 専用の `store` が除去されます。
- `params.chat_template_kwargs`: トップレベルの `api: "openai-completions"` リクエスト本文へマージされる、vLLM/OpenAI 互換のチャットテンプレート引数です。思考が無効な `vllm/nemotron-3-*` では、同梱の vLLM Plugin が `enable_thinking: false` と `force_nonempty_content: true` を自動的に送信します。明示的な `chat_template_kwargs` は生成されたデフォルトを上書きし、`extra_body.chat_template_kwargs` が引き続き最終的に優先されます。設定済みの vLLM Qwen および Nemotron 思考モデルでは、複数レベルのエフォート段階ではなく、二値の `/think` 選択肢（`off`、`on`）が公開されます。
- `compat.thinkingFormat`: OpenAI 互換の思考ペイロード形式です。Together 形式の `reasoning.enabled` には `"together"`、Qwen 形式のトップレベル `enable_thinking` には `"qwen"`、vLLM など、リクエスト単位のチャットテンプレート kwargs をサポートする Qwen 系バックエンドの `chat_template_kwargs.enable_thinking` には `"qwen-chat-template"` を使用します。OpenClaw は、思考の無効化を `false`、有効化を `true` に対応付けます。また、設定済みの vLLM Qwen モデルでは、これらの形式に対する二値の `/think` 選択肢が公開されます。
- `compat.supportedReasoningEfforts`: モデルごとの OpenAI 互換推論エフォート一覧。実際に受け付けるカスタムエンドポイントには `"xhigh"` を含めてください。これにより OpenClaw は、その構成済みプロバイダー／モデルについて、コマンドメニュー、Gateway セッション行、セッションパッチ検証、エージェント CLI 検証、および `llm-task` 検証で `/think xhigh` を公開します。正規レベルに対してバックエンドがプロバイダー固有の値を必要とする場合は、`compat.reasoningEffortMap` を使用します。
- `params.preserveThinking`: 保持された思考を利用するための Z.AI 専用オプトイン。これを有効にして思考をオンにすると、OpenClaw は `thinking.clear_thinking: false` を送信し、以前の `reasoning_content` を再実行します。[Z.AI の思考と保持された思考](/ja-JP/providers/zai#advanced-configuration)を参照してください。
- `localService`: ローカル／セルフホスト型モデルサーバー向けの、オプションのプロバイダーレベルのプロセスマネージャー。選択したモデルがそのプロバイダーに属する場合、OpenClaw は `healthUrl`（または `baseUrl + "/models"`）をプローブし、エンドポイントが停止していれば `args` を指定して `command` を起動し、最大 `readyTimeoutMs` 待機してからモデルリクエストを送信します。`command` は絶対パスでなければなりません。`idleStopMs: 0` は OpenClaw が終了するまでプロセスを存続させます。正の値を指定すると、OpenClaw が起動したプロセスを、そのミリ秒数だけアイドル状態が続いた後に停止します。[ローカルモデルサービス](/ja-JP/gateway/local-model-services)を参照してください。
- ランタイムポリシーは `agents.defaults` ではなく、プロバイダーまたはモデルに設定します。プロバイダー全体のルールには `models.providers.<provider>.agentRuntime`、モデル固有のルールには `agents.defaults.models["provider/model"].agentRuntime`／`agents.entries.*.models["provider/model"].agentRuntime` を使用します。プロバイダー／モデルのプレフィックスだけでハーネスが選択されることはありません。ランタイムが未設定または `auto` の場合、作成者によるリクエストのオーバーライドがなく、公式の HTTPS Platform Responses または ChatGPT Responses のルートに完全一致するときに限り、OpenAI は暗黙的に Codex を選択することがあります。[OpenAI の暗黙的なエージェントランタイム](/ja-JP/providers/openai#implicit-agent-runtime)を参照してください。
- これらのフィールドを変更する構成ライター（たとえば `/models set`、`/models set-image`、フォールバックの追加／削除コマンド）は、正規のオブジェクト形式で保存し、可能な場合は既存のフォールバック一覧を保持します。
- `maxConcurrent`: セッション全体で並列実行できるエージェント実行の最大数（各セッション内では引き続き直列化されます）。デフォルト: `4`。

### ランタイムポリシー

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: { id: "codex" },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
        "vllm/*": {
          agentRuntime: { id: "openclaw" },
        },
      },
    },
  },
}
```

- `id`: `"auto"`、`"openclaw"`、登録済みPluginハーネスID、またはサポートされているCLIバックエンドのエイリアス。バンドルされているCodex Pluginは`codex`を登録し、バンドルされているAnthropic Pluginは`claude-cli` CLIバックエンドを提供します。
- `id: "auto"`を使用すると、登録済みPluginハーネスは、サポート契約を宣言するか、その他の方法で満たす有効なルートを引き受けることができ、一致するハーネスがない場合はOpenClawを使用します。`id: "codex"`のように明示的なPluginランタイムを指定すると、そのハーネスと互換性のある有効なルートが必要になります。いずれかが利用できない場合や実行に失敗した場合は、フェイルクローズします。
- `id: "pi"`は、v2026.5.22以前のリリース済み設定を維持するため、`openclaw`の非推奨エイリアスとしてのみ受け付けられます。新しい設定では`openclaw`を使用してください。
- ランタイムの優先順位は、最初に完全一致するモデルポリシー（`agents.entries.*.models["provider/model"]`、`agents.defaults.models["provider/model"]`、または`models.providers.<provider>.models[]`）、次に`agents.entries.*` / `agents.defaults.models["provider/*"]`、最後に`models.providers.<provider>.agentRuntime`のプロバイダー全体のポリシーです。
- エージェント全体のランタイムキーはレガシーです。`agents.defaults.agentRuntime`、`agents.entries.*.agentRuntime`、セッションランタイムの固定指定、および`OPENCLAW_AGENT_RUNTIME`は、ランタイムの選択時に無視されます。古い値を削除するには`openclaw doctor --fix`を実行してください。
- 作成者によるリクエストの上書きがない、対象となる完全一致の公式HTTPS OpenAI Responses/ChatGPTルートでは、Codexハーネスが暗黙的に使用される場合があります。プロバイダー／モデルの`agentRuntime.id: "codex"`はCodexをフェイルクローズ要件にしますが、互換性のないルートを互換にするものではありません。
- Claude CLIのデプロイでは、`model: "anthropic/claude-opus-5"`とモデルスコープの`agentRuntime.id: "claude-cli"`を組み合わせることを推奨します。レガシーの`claude-cli/<model>`参照も互換性のため引き続き機能しますが、新しい設定ではプロバイダー／モデルの選択を正規形に保ち、実行バックエンドをプロバイダー／モデルのランタイムポリシーに配置してください。
- これはテキストエージェントのターン実行のみを制御します。メディア生成、ビジョン、PDF、音楽、動画、TTSでは、引き続きそれぞれのプロバイダー／モデル設定が使用されます。

**組み込みエイリアスの短縮形**（モデルが`agents.defaults.models`に含まれる場合にのみ適用）:

| エイリアス          | モデル                          |
| ------------------- | ------------------------------- |
| `opus`              | `anthropic/claude-opus-5`       |
| `sonnet`            | `anthropic/claude-sonnet-5`     |
| `gpt`               | `openai/gpt-5.4`                |
| `gpt-mini`          | `openai/gpt-5.4-mini`           |
| `gpt-nano`          | `openai/gpt-5.4-nano`           |
| `gemini`            | `google/gemini-3.1-pro-preview` |
| `gemini-flash`      | `google/gemini-3-flash-preview` |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite`  |

設定したエイリアスは、常にデフォルトより優先されます。

Z.AI GLM-4.xモデルでは、`--thinking off`を設定するか、`agents.defaults.models["zai/<model>"].params.thinking`を独自に定義しない限り、思考モードが自動的に有効になります。
Z.AIモデルでは、ツール呼び出しのストリーミング用に`tool_stream`がデフォルトで有効になります。無効にするには、`agents.defaults.models["zai/<model>"].params.tool_stream`を`false`に設定してください。
Anthropic Claude Opus 4.8では、OpenClawで思考がデフォルトで無効になっています。適応的思考を明示的に有効にすると、Anthropicプロバイダーが管理するエフォートのデフォルト値は`high`になります。Claude 4.6モデルでは、明示的な思考レベルが設定されていない場合、デフォルトで`adaptive`が使用されます。

### CLIバックエンドの選択

CLIアダプターの仕組みはPluginによって登録され、エージェントの
デフォルトでは設定されません。上記のように、モデルスコープの`agentRuntime.id`を使用して、
登録済みCLIバックエンドを選択してください。運用については[CLIバックエンド](/ja-JP/gateway/cli-backends)を、
コマンド、セッション、画像、パーサーの登録については
[CLIバックエンドPluginの構築](/ja-JP/plugins/cli-backend-plugins)を参照してください。

### `agents.defaults.promptOverlays`

OpenClawが組み立てるプロンプトサーフェスにモデルファミリー単位で適用される、プロバイダーに依存しないプロンプトオーバーレイ。GPT-5ファミリーのモデルIDは、OpenClaw／プロバイダーの各ルートで共通の動作契約を受け取ります。`personality`は、親しみやすい対話スタイルのレイヤーのみを制御します。ネイティブCodex app-serverルートでは、このOpenClaw GPT-5オーバーレイの代わりにCodexが管理するベース／モデル命令が維持され、OpenClawはネイティブスレッドでCodexの組み込みパーソナリティを無効にします。

```json5
{
  agents: {
    defaults: {
      promptOverlays: {
        gpt5: {
          personality: "friendly", // friendly | on | off
        },
      },
    },
  },
}
```

- `"friendly"`（デフォルト）と`"on"`は、親しみやすい対話スタイルのレイヤーを有効にします。
- `"off"`は親しみやすいレイヤーのみを無効にします。タグ付けされたGPT-5の動作契約は引き続き有効です。
- この共有設定が未設定の場合、レガシーの`plugins.entries.openai.config.personality`が引き続き読み込まれます。

### `agents.defaults.heartbeat`

定期的なHeartbeatの実行。

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m disables
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        includeSystemPromptSection: true, // default: true; false omits the Heartbeat section from the system prompt
        lightContext: false, // default: false; true skips workspace bootstrap files for heartbeat runs
        isolatedSession: false, // default: false; true runs each heartbeat in a fresh session (no conversation history)
        skipWhenBusy: false, // default: false; true also waits for this agent's subagent/nested lanes
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (default) | block
        target: "none", // default: none | options: last | whatsapp | telegram | discord | ...
        prompt: "Follow the heartbeat monitor scratch context...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
        timeoutSeconds: 45,
      },
    },
  },
}
```

- `every`: 期間を表す文字列（ms/s/m/h）。デフォルト: `30m`（APIキー認証）または`1h`（OAuth認証）。無効にするには`0m`に設定します。
- 実行間隔は、システムが管理するCronモニター行に書き込まれます。欠落または古くなった行を実体化するには、`openclaw doctor --fix`を実行してください。Cronが無効な場合、スケジュールされたHeartbeatは実行されず、Gatewayは起動時の警告をログに記録します。
- `includeSystemPromptSection`: falseの場合、システムプロンプトからHeartbeatセクションを省略します。デフォルト: `true`。
- `suppressToolErrorWarnings`: trueの場合、Heartbeatの実行中にツールエラーの警告ペイロードを抑制します。
- `timeoutSeconds`: Heartbeatエージェントのターンが中止されるまでに許容される最大時間（秒）。未設定のままにすると、`agents.defaults.timeoutSeconds`が設定されている場合はその値を使用し、それ以外の場合はHeartbeatの実行間隔を最大600秒に制限した値を使用します。
- `directPolicy`: ダイレクト／DM配信ポリシー。`allow`（デフォルト）はダイレクトターゲットへの配信を許可します。`block`はダイレクトターゲットへの配信を抑制し、`reason=dm-blocked`を出力します。
- `lightContext`: trueの場合、Heartbeatの実行では軽量なブートストラップコンテキストを使用し、ワークスペースのブートストラップファイルをスキップします。どちらの場合も、モニターのスクラッチ情報はHeartbeatランナーによって挿入されます。
- `isolatedSession`: trueの場合、各Heartbeatは以前の会話履歴がない新しいセッションで実行されます。Cronの`sessionTarget: "isolated"`と同じ分離パターンです。Heartbeatごとのトークンコストを約100Kから約2-5Kトークンに削減します。
- `skipWhenBusy`: trueの場合、そのエージェントに追加のビジーなレーンがある間、Heartbeatの実行を延期します。対象は、そのエージェント自身のセッションキーに紐づくサブエージェントまたはネストされたコマンド処理です。このフラグがなくても、Cronレーンは常にHeartbeatを延期します。
- エージェント単位: `agents.entries.*.heartbeat`を設定します。いずれかのエージェントが`heartbeat`を定義すると、**それらのエージェントのみ**がHeartbeatを実行します。
- Heartbeatはエージェントの完全なターンを実行します。間隔を短くすると、より多くのトークンを消費します。

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        provider: "my-provider", // id of a registered compaction provider plugin (optional)
        thinkingLevel: "low", // optional compaction-only thinking override
        timeoutSeconds: 180,
        keepRecentTokens: 50000,
        recentTurnsPreserve: 3,
        identifierPolicy: "strict", // strict | off
        qualityGuard: { enabled: true, maxRetries: 1 },
        midTurnPrecheck: { enabled: false }, // optional tool-loop pressure check
        postIndexSync: "async", // off | async | await
        postCompactionSections: ["Session Startup", "Red Lines"],
        model: "openrouter/anthropic/claude-sonnet-4-6", // optional compaction-only model override
        truncateAfterCompaction: true, // rotate to a smaller successor JSONL after compaction
        maxActiveTranscriptBytes: "20mb", // optional preflight local compaction trigger
        notifyUser: true, // notices when compaction starts/completes and on memory-flush degradation (default: false)
        memoryFlush: {
          enabled: true,
          model: "ollama/qwen3:8b", // optional memory-flush-only model override
          softThresholdTokens: 6000,
          forceFlushTranscriptBytes: "2mb",
        },
      },
    },
  },
}
```

- `mode`: `default` または `safeguard`（長い履歴向けのチャンク分割要約）。[Compaction](/ja-JP/concepts/compaction)を参照してください。
- `provider`: 登録済みの Compaction プロバイダー Plugin の ID。設定すると、組み込みの LLM 要約の代わりにプロバイダーの `summarize()` が呼び出されます。失敗時は組み込み機能にフォールバックします。プロバイダーを設定すると `mode: "safeguard"` が強制されます。[Compaction](/ja-JP/concepts/compaction)を参照してください。
- `thinkingLevel`: 埋め込み OpenClaw の Compaction 要約にのみ使用されるオプションの思考レベル（`off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`adaptive`、`max`、または `ultra`）。セッションの現在の思考レベルを上書きし、選択された Compaction モデル／ランタイムに合わせて制限されます。セッションレベルを継承するには未設定のままにします。ネイティブの compact リクエストには操作単位の思考オーバーライドがないため、ネイティブ Codex app-server の Compaction ではこの設定は無視されます。設定されている場合、OpenClaw は警告をログに記録します。
- `timeoutSeconds`: OpenClaw が中止するまでに、1 回の Compaction 操作に許可される最大秒数。デフォルト: `180`。
- `keepRecentTokens`: 最新のトランスクリプト末尾をそのまま保持するためのエージェント切断点予算。明示的に設定されている場合、手動の `/compact` はこれに従います。それ以外の場合、手動 Compaction はハードチェックポイントになります。
- `recentTurnsPreserve`: セーフガード要約の外側でそのまま保持される、直近のユーザー／アシスタントのターン数。デフォルト: `3`。
- `identifierPolicy`: `strict`（デフォルト）または `off`。`strict` は、Compaction 要約時に、不透明な識別子を保持するための組み込みガイダンスを先頭に追加します。
- `qualityGuard`: セーフガード要約に対する、不正形式の出力時の再試行チェック。セーフガードモードではデフォルトで有効です。監査をスキップするには `enabled: false` を設定します。
- `midTurnPrecheck`: オプションのツールループ負荷チェック。`enabled: true` の場合、OpenClaw はツール結果の追加後、次のモデル呼び出し前にコンテキスト負荷を確認します。コンテキストが収まらなくなった場合は、プロンプトを送信する前に現在の試行を中止し、既存の事前チェック復旧パスを再利用してツール結果を切り詰めるか、Compaction を実行して再試行します。`default` と `safeguard` の両方の Compaction モードで動作します。デフォルト: 無効。
- `postIndexSync`: Compaction 後のセッションメモリ再インデックスモード。デフォルト: `"async"`。鮮度を最優先する場合は `"await"`、Compaction のレイテンシを抑える場合は `"async"`、セッションメモリの同期がほかで処理される場合にのみ `"off"` を使用します。
- `postCompactionSections`: Compaction 後に再注入する、オプションの AGENTS.md H2／H3 セクション名。無効にするには未設定のままにするか、`[]` を使用します。
- `model`: Compaction 要約にのみ使用する、オプションの `provider/model-id` または `agents.defaults.models` の単純なエイリアス。単純なエイリアスはディスパッチ前に解決されます。競合時は、設定されたリテラルモデル ID が優先されます。メインセッションではあるモデルを維持しつつ、Compaction 要約を別のモデルで実行する場合に使用します。未設定の場合、Compaction はセッションのプライマリモデルを使用します。
- `truncateAfterCompaction`: Compaction 後にアクティブなセッショントランスクリプトをローテーションし、以降のターンでは要約と未要約の末尾だけを読み込むようにします。以前の完全なトランスクリプトはアーカイブされたままになります。長時間実行されるセッションで、アクティブなトランスクリプトが際限なく増大するのを防ぎます。デフォルト: `false`。
- `maxActiveTranscriptBytes`: トランスクリプト履歴がしきい値を超えたとき、実行前に通常のローカル Compaction を開始するオプションのバイトしきい値（`number` または `"20mb"` のような文字列）。Compaction の成功後に、より小さい後継トランスクリプトへローテーションできるようにするため、`truncateAfterCompaction` が必要です。未設定または `0` の場合は無効です。
- `notifyUser`: `true` の場合、簡潔なコンテキスト保守通知をユーザーに送信します。通知されるのは、Compaction の開始時と完了時（例: 「コンテキストを圧縮しています...」「Compaction が完了しました」）、および Compaction 前のメモリフラッシュが枯渇し、機能低下状態で応答を継続する場合（例: 「メモリ保守が一時的に失敗しました。応答を続行します。」）です。これらの通知を表示しないようにするため、デフォルトでは無効です。
- `memoryFlush`: 永続メモリを保存するために、自動 Compaction の前に実行されるサイレントなエージェントターン。この保守ターンをローカルモデル上で維持する必要がある場合は、`model` を `ollama/qwen3:8b` のような正確なプロバイダー／モデルに設定します。このオーバーライドは、アクティブなセッションのフォールバックチェーンを継承しません。`forceFlushTranscriptBytes` は、トークンカウンターが古くても、トランスクリプトサイズがしきい値に達した時点でフラッシュを強制します。ワークスペースが読み取り専用の場合はスキップされます。

カスタム Compaction 命令はコード側で管理されます。カスタム要約を構築するには
`summarize()` を備えた Compaction プロバイダー Plugin を実装し、Compaction 後の
コンテキストを後続のモデルプロンプトへ注入する必要がある場合は
`before_prompt_build` を使用します。Doctor は廃止された命令フィールドを削除し、これらの
接続点を案内します。

### `agents.defaults.contextPruning`

LLM に送信する前に、メモリ内コンテキストから**古いツール結果**を除去します。ディスク上のセッション履歴は変更**しません**。デフォルトでは無効です。有効にするには `mode: "cache-ttl"` を設定します。

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // オフ（デフォルト）| cache-ttl
      },
    },
  },
}
```

<Accordion title="cache-ttl モードの動作">

- `mode: "cache-ttl"` は除去処理を有効にします。
- 除去では、まず大きすぎるツール結果をソフトトリミングし、必要に応じて古いツール結果を完全に消去します。

**ソフトトリミング**では先頭と末尾を保持し、中央に `...` を挿入します。

**完全消去**では、ツール結果全体をプレースホルダーに置き換えます。

注:

- 画像ブロックはトリミングも消去もされません。
- 比率は文字数に基づく概算であり、正確なトークン数ではありません。
- 直近のアシスタントメッセージは保持されます。

</Accordion>

動作の詳細については、[セッションの除去](/ja-JP/concepts/session-pruning)を参照してください。

### ブロックストリーミング

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200, breakPreference: "paragraph" },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off（デフォルト）| natural | custom（minMs/maxMs を使用）
    },
  },
}
```

- Telegram 以外のチャンネルでブロック応答を有効にするには、明示的な `*.streaming.block.enabled: true` が必要です。QQ Bot は例外で、`streaming.block` キーがなく、`channels.qqbot.streaming.mode` が `"off"` でない限りブロック応答をストリーミングします。
- チャンネル単位のオーバーライド: `channels.<channel>.streaming.block.coalesce`（アカウント単位のバリアントも含む）。Discord、Google Chat、Mattermost、MS Teams、Signal、Slack のデフォルトは `minChars: 1500`／`idleMs: 1000` です。
- `blockStreamingChunk.breakPreference`: 優先するチャンク境界（`"paragraph" | "newline" | "sentence"`）。
- `humanDelay`: ブロック応答間のランダムな一時停止。デフォルト: `off`。`natural` = 800-2500ms。`custom` は `minMs`／`maxMs` を使用します（未設定の境界値には自然な範囲を使用します）。エージェント単位のオーバーライド: `agents.entries.*.humanDelay`。

動作とチャンク分割の詳細については、[ストリーミング](/ja-JP/concepts/streaming)を参照してください。

### 入力中インジケーター

```json5
{
  agents: {
    defaults: {
      typingMode: "instant", // never | instant | thinking | message
      typingIntervalSeconds: 6,
    },
  },
}
```

- デフォルト: ダイレクトチャット／メンションでは `instant`、メンションのないグループチャットでは `message`。
- `typingIntervalSeconds` のデフォルト: `6`。
- エージェント単位のオーバーライド: `agents.entries.*.typingMode`。

[入力中インジケーター](/ja-JP/concepts/typing-indicators)を参照してください。

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

埋め込みエージェント向けのオプションのサンドボックス化。完全なガイドについては、[サンドボックス化](/ja-JP/gateway/sandboxing)を参照してください。

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off（デフォルト）| non-main | all
        backend: "docker", // docker（デフォルト）| ssh | openshell
        scope: "agent", // session | agent（デフォルト）| shared
        workspaceAccess: "none", // none（デフォルト）| ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          containerPrefix: "openclaw-sbx-",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          gpus: "all",
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
          binds: ["/home/user/source:/source:rw"],
        },
        ssh: {
          target: "user@gateway-host:22",
          command: "ssh",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // SecretRefs／インライン内容にも対応:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
        browser: {
          enabled: false,
          image: "openclaw-sandbox-browser:bookworm-slim",
          network: "openclaw-sandbox-browser",
          cdpPort: 9222,
          cdpSourceRange: "172.21.0.1/32",
          vncPort: 5900,
          noVncPort: 6080,
          headless: false,
          enableNoVnc: true,
          allowHostControl: false,
          autoStart: true,
          autoStartTimeoutMs: 12000,
        },
        prune: {
          idleHours: 24,
          maxAgeDays: 7,
        },
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        allow: [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "apply_patch",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

上記のデフォルト（`off`／`docker`／`agent`／`none`／`bookworm-slim` イメージ／`none` ネットワークなど）は、単なる例示値ではなく、実際の OpenClaw のデフォルトです。

<Accordion title="サンドボックスの詳細">

**バックエンド:**

- `docker`: ローカル Docker ランタイム（デフォルト）
- `ssh`: 汎用 SSH ベースのリモートランタイム
- `openshell`: OpenShell ランタイム

`backend: "openshell"` を選択すると、ランタイム固有の設定は
`plugins.entries.openshell.config` に移動します。

**SSH バックエンド設定:**

- `target`: `user@host[:port]` 形式の SSH ターゲット
- `command`: SSH クライアントコマンド（デフォルト: `ssh`）
- `workspaceRoot`: スコープごとのワークスペースに使用するリモートの絶対ルート（デフォルト: `/tmp/openclaw-sandboxes`）
- `identityFile` / `certificateFile` / `knownHostsFile`: OpenSSH に渡される既存のローカルファイル
- `identityData` / `certificateData` / `knownHostsData`: OpenClaw が実行時に一時ファイルとして具現化するインラインコンテンツまたは SecretRef
- `strictHostKeyChecking` / `updateHostKeys`: OpenSSH のホストキーポリシー設定（両方ともデフォルトは `true`）

**SSH 認証の優先順位:**

- `identityData` は `identityFile` より優先されます
- `certificateData` は `certificateFile` より優先されます
- `knownHostsData` は `knownHostsFile` より優先されます
- SecretRef を使用する `*Data` の値は、サンドボックスセッションの開始前に、アクティブなシークレットランタイムのスナップショットから解決されます

**SSH バックエンドの動作:**

- 作成または再作成後にリモートワークスペースへ一度だけシードします
- その後はリモート SSH ワークスペースを正規の状態として維持します
- `exec`、ファイルツール、メディアパスを SSH 経由で処理します
- リモートでの変更をホストへ自動的に同期しません
- サンドボックス化されたブラウザコンテナには対応していません

**ワークスペースへのアクセス:**

- `none`: `~/.openclaw/sandboxes` 配下のスコープごとのサンドボックスワークスペース（デフォルト）
- `ro`: `/workspace` にあるサンドボックスワークスペース。エージェントワークスペースは `/agent` に読み取り専用でマウントされます
- `rw`: エージェントワークスペースは `/workspace` に読み書き可能でマウントされます

**スコープ:**

- `session`: セッションごとのコンテナとワークスペース
- `agent`: エージェントごとに 1 つのコンテナとワークスペース（デフォルト）
- `shared`: 共有コンテナとワークスペース（セッション間の分離なし）

**OpenShell Plugin の設定:**

```json5
{
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          mode: "mirror", // mirror (default) | remote
          command: "openshell",
          from: "openclaw",
          remoteWorkspaceDir: "/sandbox",
          remoteAgentWorkspaceDir: "/agent",
          gateway: "lab", // optional
          gatewayEndpoint: "https://lab.example", // optional
          policy: "strict", // optional OpenShell policy id
          providers: ["openai"], // optional
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**OpenShell モード:**

- `mirror`: 実行前にローカルからリモートへシードし、実行後に同期して戻します。ローカルワークスペースが正規の状態として維持されます
- `remote`: サンドボックスの作成時にリモートへ一度だけシードし、その後はリモートワークスペースを正規の状態として維持します

`remote` モードでは、シード処理後に OpenClaw の外部で行われたホストローカルの編集は、サンドボックスへ自動的に同期されません。
転送には OpenShell サンドボックスへの SSH を使用しますが、サンドボックスのライフサイクルと任意のミラー同期は Plugin が管理します。

**`setupCommand`** はコンテナ作成後に一度だけ実行されます（`sh -lc` 経由）。ネットワークへの送信アクセス、書き込み可能なルート、root ユーザーが必要です。

**コンテナのデフォルトは `network: "none"` です** — エージェントに外部へのアクセスが必要な場合は、`"bridge"`（またはカスタムブリッジネットワーク）に設定します。
`"host"` はブロックされます。`"container:<id>"` は、`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`（緊急時用）を明示的に設定しない限り、デフォルトでブロックされます。
アクティブな OpenClaw サンドボックス内の Codex app-server ターンでは、ネイティブコードモードのネットワークアクセスにも同じ送信設定が使用されます。

**受信した添付ファイル**は、アクティブなワークスペースの `media/inbound/*` に配置されます。

**`docker.binds`** は追加のホストディレクトリをマウントします。グローバルバインドとエージェントごとのバインドはマージされます。

**サンドボックス化されたブラウザ**（`sandbox.browser.enabled`、デフォルトは `false`）: コンテナ内の Chromium + CDP。noVNC URL がシステムプロンプトに挿入されます。`openclaw.json` 内の `browser.enabled` は必要ありません。
noVNC のオブザーバーアクセスではデフォルトで VNC 認証が使用され、OpenClaw は共有 URL 内でパスワードを公開する代わりに、有効期間の短いトークン URL を発行します。

- `allowHostControl: false`（デフォルト）は、サンドボックス化されたセッションがホストブラウザをターゲットにすることをブロックします。
- `network` のデフォルトは `openclaw-sandbox-browser`（専用ブリッジネットワーク）です。グローバルなブリッジ接続を明示的に必要とする場合に限り、`bridge` に設定します。ここでも `"host"` はブロックされます。
- `cdpSourceRange` を使用すると、コンテナ境界での CDP 受信アクセスを CIDR 範囲（例: `172.21.0.1/32`）に制限できます。
- `sandbox.browser.binds` は、追加のホストディレクトリをサンドボックスブラウザコンテナのみにマウントします。設定すると（`[]` を含む）、ブラウザコンテナでは `docker.binds` の代わりに使用されます。
- サンドボックスブラウザコンテナの Chromium は常に `--no-sandbox --disable-setuid-sandbox` を指定して起動します（コンテナには Chrome 独自のサンドボックスに必要なカーネルプリミティブがありません）。これを切り替える設定はありません。
- 起動時のデフォルトは `scripts/sandbox-browser-entrypoint.sh` で定義され、コンテナホスト向けに調整されています:
  - `--remote-debugging-address=127.0.0.1`
  - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
  - `--user-data-dir=${HOME}/.chrome`
  - `--no-first-run`
  - `--no-default-browser-check`
  - `--disable-dev-shm-usage`
  - `--disable-background-networking`
  - `--disable-breakpad`
  - `--disable-crash-reporter`
  - `--no-zygote`
  - `--metrics-recording-only`
  - `--password-store=basic`
  - `--use-mock-keychain`
  - `--disable-3d-apis`、`--disable-gpu`、`--disable-software-rasterizer` は
    デフォルトで有効です。WebGL/3D の使用に必要な場合は、
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` で無効にできます。
  - `--disable-extensions`（デフォルトで有効）。ワークフローが拡張機能に依存する場合は、
    `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` で拡張機能を再度有効にできます。
  - デフォルトでは `--renderer-process-limit=2`。`OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` で変更できます。
    Chromium のデフォルトのプロセス上限を使用するには、`0` を設定します。
  - `headless` が有効な場合に限り `--headless=new`。
  - デフォルト値はコンテナイメージのベースラインです。コンテナのデフォルトを変更するには、
    カスタムエントリポイントを備えたカスタムブラウザイメージを使用します。

</Accordion>

ブラウザのサンドボックス化と `sandbox.docker.binds` は Docker でのみ使用できます。

イメージをビルドする場合（ソースチェックアウトから）:

```bash
scripts/sandbox-setup.sh           # main sandbox image
scripts/sandbox-browser-setup.sh   # optional browser image
```

ソースチェックアウトなしで npm インストールを行う場合は、インラインの `docker build` コマンドについて [サンドボックス化 § イメージとセットアップ](/ja-JP/gateway/sandboxing#images-and-setup)を参照してください。

### `agents.entries`（エージェントごとのオーバーライド）

`agents.entries.*.tts` を使用すると、エージェントごとに独自の TTS プロバイダー、音声、モデル、
スタイル、または自動 TTS モードを指定できます。エージェントブロックはグローバルな
`tts` にディープマージされるため、共有認証情報を 1 か所に保持しながら、各
エージェントは必要な音声またはプロバイダーフィールドのみをオーバーライドできます。アクティブなエージェントの
オーバーライドは、自動音声応答、`/tts audio`、`/tts status`、および
`tts` エージェントツールに適用されます。プロバイダーの例と優先順位については、
[テキスト読み上げ](/ja-JP/tools/tts#per-agent-voice-overrides)を参照してください。

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "Main Agent",
        workspace: "~/.openclaw/workspace",
        agentDir: "~/.openclaw/agents/main/agent",
        model: "anthropic/claude-opus-4-6", // or { primary, fallbacks }
        utilityModel: "openai/gpt-5.4-mini",
        thinkingDefault: "high", // per-agent thinking level override
        reasoningDefault: "on", // per-agent reasoning visibility override
        fastModeDefault: false, // per-agent fast mode override
        params: { cacheRetention: "none" }, // overrides matching defaults.models params by key
        tts: {
          providers: {
            elevenlabs: { speakerVoiceId: "EXAVITQu4vr4xnSDxMaL" },
          },
        },
        skills: ["docs-search"], // replaces agents.defaults.skills when set
        identity: {
          name: "Samantha",
          theme: "helpful sloth",
          emoji: "🦥",
          avatar: "avatars/samantha.png",
        },
        groupChat: { mentionPatterns: ["@openclaw"] },
        sandbox: { mode: "off" },
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent", // persistent | oneshot
            cwd: "/workspace/openclaw",
          },
        },
        subagents: { allowAgents: ["*"] },
        tools: {
          profile: "coding",
          allow: ["browser"],
          deny: ["canvas"],
          elevated: { enabled: true },
        },
      },
    ],
  },
}
```

- `id`: 安定したエージェント ID（必須）。
- `default`: 複数設定されている場合、最初のものが優先されます（警告がログに記録されます）。何も設定されていない場合、リストの最初のエントリがデフォルトになります。
- `model`: 文字列形式では、モデルのフォールバックを使用しない厳密なエージェント単位のプライマリが設定されます。オブジェクト形式の `{ primary }` も、`fallbacks` を追加しない限り厳密です。`{ primary, fallbacks: [...] }` を使用するとそのエージェントでフォールバックが有効になり、`{ primary, fallbacks: [] }` を使用すると厳密な動作を明示できます。`primary` のみを上書きする Cron ジョブは、`fallbacks: []` を設定しない限り、引き続きデフォルトのフォールバックを継承します。
- `utilityModel`: 生成されるセッションやスレッドのタイトルなど、短い内部タスク向けの、エージェント単位のオプションの上書きです。`agents.defaults.utilityModel`、次に有効なセッションプロバイダーが宣言する小規模モデルのデフォルトへフォールバックします。ダッシュボードのタイトルでは、有効な通常のセッションモデルを使用して 1 回再試行します。空文字列を指定すると、ダッシュボードのタイトル生成を無効にせず、このエージェントの代替ユーティリティルートをスキップします。
- `params`: `agents.defaults.models` で選択されたモデルエントリにマージされる、エージェント単位のストリームパラメーターです。モデルカタログ全体を複製せずに、`cacheRetention`、`temperature`、`maxTokens` などのエージェント固有の上書きを行う場合に使用します。
- `tts`: エージェント単位のオプションのテキスト読み上げ上書きです。このブロックは `tts` にディープマージされるため、共有プロバイダー認証情報とフォールバックポリシーは `tts` に保持し、ここではプロバイダー、音声、モデル、スタイル、自動モードなど、ペルソナ固有の値のみを設定してください。
- `skills`: エージェント単位のオプションのスキル許可リストです。省略した場合、設定されていればエージェントは `agents.defaults.skills` を継承します。明示的なリストはデフォルトとマージされず、置き換えます。また、`[]` はスキルなしを意味します。
- `thinkingDefault`: エージェント単位のオプションのデフォルト思考レベル（`off | minimal | low | medium | high | xhigh | adaptive | max`）。メッセージ単位またはセッション単位の上書きが設定されていない場合、このエージェントでは `agents.defaults.thinkingDefault` を上書きします。有効な値は、選択したプロバイダー／モデルプロファイルによって決まります。Google Gemini では、`adaptive` によりプロバイダー側の動的思考が維持されます（Gemini 3/3.1 では `thinkingLevel` を省略、Gemini 2.5 では `thinkingBudget: -1`）。
- `reasoningDefault`: エージェント単位のオプションのデフォルト推論表示（`on | off | stream`）。メッセージ単位またはセッション単位の推論上書きが設定されていない場合、このエージェントでは `agents.defaults.reasoningDefault` を上書きします。
- `fastModeDefault`: エージェント単位のオプションの高速モードのデフォルト（`"auto" | true | false`）。メッセージ単位またはセッション単位の高速モード上書きが設定されていない場合に適用されます。
- `models`: 完全な `provider/model` ID をキーとする、エージェント単位のオプションのモデルカタログ／ランタイム上書きです。エージェント単位のランタイム例外には `models["provider/model"].agentRuntime` を使用します。
- `runtime`: エージェント単位のオプションのランタイム記述子です。エージェントで ACP ハーネスセッションをデフォルトにする場合は、`runtime.acp` のデフォルト（`agent`、`backend`、`mode`、`cwd`）とともに `type: "acp"` を使用します。
- `identity.avatar`: ワークスペース相対パス、`http(s)` URL、または `data:` URI。
- ローカルのワークスペース相対 `identity.avatar` 画像ファイルは 2 MB に制限されます。`http(s)` URL と `data:` URI には、ローカルファイルサイズの制限は適用されません。
- `identity` はデフォルトを導出します。`emoji` から `ackReaction`、`name`/`emoji` から `mentionPatterns`。
- `subagents.allowAgents`: 明示的な `sessions_spawn.agentId` ターゲットに使用できる、設定済みエージェント ID の許可リストです（`["*"]` = 設定済みの任意のターゲット、デフォルト: 同じエージェントのみ）。自身をターゲットとする `agentId` 呼び出しを許可する場合は、リクエスター ID を含めます。エージェント設定が削除された古いエントリは `sessions_spawn` によって拒否され、`agents_list` から除外されます。クリーンアップするには `openclaw doctor --fix` を実行します。または、デフォルトを継承しつつそのターゲットを引き続き生成可能にする場合は、最小限の `agents.entries.*` エントリを追加します。
- サンドボックス継承ガード: リクエスターのセッションがサンドボックス化されている場合、`sessions_spawn` はサンドボックス外で実行されるターゲットを拒否します。
- `subagents.requireAgentId`: true の場合、`agentId` を省略した `sessions_spawn` 呼び出しをブロックします（明示的なプロファイル選択を強制、デフォルト: false）。
- `subagents.maxConcurrent`: サブエージェント実行全体で同時に実行できる子エージェントの最大数。デフォルト: `8`。
- `subagents.maxChildrenPerAgent`: 1 つのエージェントセッションが生成できるアクティブな子の最大数。デフォルト: `5`。
- `subagents.maxSpawnDepth`: サブエージェント生成の最大ネスト深度（`1`～`5`）。デフォルト: `1`（ネストなし）。
- `subagents.archiveAfterMinutes`: 完了したサブエージェントの状態がアーカイブされるまでの期間。デフォルト: `60`。

---

## マルチエージェントルーティング

1 つの Gateway 内で、分離された複数のエージェントを実行します。[マルチエージェント](/ja-JP/concepts/multi-agent)を参照してください。

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],
}
```

### バインディングの照合フィールド

- `type`（任意）: 通常のルーティングでは `route`（type がない場合は route がデフォルト）、永続的な ACP 会話バインディングでは `acp`。
- `match.channel`（必須）
- `match.accountId`（任意、`*` = 任意のアカウント、省略時 = デフォルトアカウント）
- `match.peer`（任意、`{ kind: direct|group|channel, id }`）
- `match.guildId` / `match.teamId`（任意、チャンネル固有）
- `acp`（任意、`type: "acp"` のみ）: `{ mode, label, cwd, backend }`

**決定的な照合順序:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId`（完全一致、peer/guild/team なし）
5. `match.accountId: "*"`（チャンネル全体）
6. デフォルトエージェント

各階層内では、最初に一致した `bindings` エントリが優先されます。

`type: "acp"` エントリの場合、OpenClaw は会話 ID（`match.channel` + アカウント + `match.peer.id`）の完全一致によって解決し、前述のルートバインディングの階層順序は使用しません。

### エージェント単位のアクセスプロファイル

<Accordion title="フルアクセス（サンドボックスなし）">

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="読み取り専用ツール + ワークスペース">

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
        tools: {
          allow: [
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="ファイルシステムアクセスなし（メッセージングのみ）">

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
        tools: {
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
            "gateway",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

</Accordion>

優先順位の詳細については、[マルチエージェントのサンドボックスとツール](/ja-JP/tools/multi-agent-sandbox-tools)を参照してください。

---

## セッション

```json5
{
  session: {
    scope: "per-sender",
    dmScope: "main", // main | per-peer | per-channel-peer | per-account-channel-peer
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"],
    },
    reset: {
      mode: "daily", // daily | idle
      atHour: 4,
      idleMinutes: 60,
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      direct: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 30 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    maintenance: {
      mode: "enforce", // enforce (default) | warn
      pruneAfter: "30d",
      maxEntries: 500,
      resetArchiveRetention: "30d", // duration or false
      maxDiskBytes: "500mb", // optional hard budget
      highWaterBytes: "400mb", // optional cleanup target
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // default inactivity auto-unfocus in hours (`0` disables)
      maxAgeHours: 0, // default hard max age in hours (`0` disables)
    },
    sharing: {
      readOnly: true,
      suggest: true,
      drafts: true,
    },
    mainKey: "main", // legacy (runtime always uses "main")
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="セッションフィールドの詳細">

- **`scope`**: グループチャットコンテキストの基本セッショングループ化戦略。
  - `per-sender`（デフォルト）: 各送信者に、チャネルコンテキスト内で分離されたセッションが割り当てられます。
  - `global`: チャネルコンテキスト内のすべての参加者が単一のセッションを共有します（共有コンテキストを意図する場合にのみ使用してください）。
- **`dmScope`**: DM のグループ化方法。
  - `main`: すべての DM がメインセッションを共有します。
  - `per-peer`: チャネルをまたいで送信者 ID ごとに分離します。
  - `per-channel-peer`: チャネルと送信者の組み合わせごとに分離します（複数ユーザーの受信トレイに推奨）。
  - `per-account-channel-peer`: アカウント、チャネル、送信者の組み合わせごとに分離します（複数アカウントに推奨）。
- **`identityLinks`**: チャネル間でセッションを共有するため、正規 ID をプロバイダー接頭辞付きのピアにマッピングします。`/dock_discord` などのドックコマンドも同じマップを使用し、アクティブなセッションの返信経路を、リンクされた別のチャネルピアへ切り替えます。[チャネルのドッキング](/ja-JP/concepts/channel-docking)を参照してください。
- **`reset`**: 主要なリセットポリシー。`none` は自動リセットを無効にし、デフォルトです。代わりに Compaction がアクティブコンテキストを制限します。`daily` は local time の `atHour` にリセットし、`idle` は `idleMinutes` 後にリセットします。両方が設定されている場合、先に期限を迎えた方が適用されます。`/new` と `/reset` はすべてのモードで引き続き使用できます。日次リセットの鮮度判定にはセッション行の `sessionStartedAt` を使用し、アイドルリセットの鮮度判定には `lastInteractionAt` を使用します。Heartbeat、Cron のウェイクアップ、exec 通知、Gateway の管理処理などのバックグラウンド／システムイベントによる書き込みは `updatedAt` を更新することがありますが、日次／アイドルセッションの鮮度は維持しません。
  - **`resetByType`**: タイプごとのオーバーライド（`direct`、`group`、`thread`）。Doctor は従来の `dm` エントリを `direct` に移行します。スキーマは `dm` を拒否します。
- **`resetByChannel`**: プロバイダー／チャネル ID をキーとする、チャネルごとのリセットオーバーライド。セッションのチャネルに一致するエントリがある場合、そのセッションでは `resetByType`/`reset` より無条件に優先されます。あるチャネルだけにタイプ単位のポリシーとは異なるリセット動作が必要な場合にのみ使用してください。
- **`mainKey`**: 従来のフィールド。ランタイムはメインのダイレクトチャット用バケットに常に `"main"` を使用します。
- **`sendPolicy`**: `channel`、`chatType`（`direct|group|channel`、従来の `dm` エイリアスを含む）、`keyPrefix`、または `rawKeyPrefix` で照合します。最初に一致した拒否が優先されます。
- **`maintenance`**: セッションストアのクリーンアップと保持に関する制御。
  - `mode`: `enforce` はクリーンアップを適用し、デフォルトです。`warn` は警告のみを出力します。
  - `pruneAfter`: 古いエントリを判定する経過時間のしきい値（デフォルトは `30d`）。
  - `maxEntries`: SQLite セッションエントリの最大数（デフォルトは `500`）。ランタイムによる書き込みでは、本番規模の上限に対して小さな上限超過バッファーを設けて一括クリーンアップを行います。`openclaw sessions cleanup --enforce` は上限を即時に適用します。
  - 短時間のみ使用される Gateway モデル実行プローブセッションには固定の `24h` 保持期間が適用されますが、クリーンアップは負荷に応じて実行されます。セッションエントリのメンテナンス／上限負荷に達した場合にのみ、古い厳密なモデル実行プローブ行を削除します。`agent:*:explicit:model-run-<uuid>` に一致する、明示された厳密なプローブキーだけが対象です。通常のダイレクト、グループ、スレッド、Cron、フック、Heartbeat、ACP、サブエージェントの各セッションには、この 24h の保持期間は継承されません。モデル実行のクリーンアップは、より広範な `pruneAfter` による古いエントリのクリーンアップと `maxEntries` の上限適用より先に実行されます。
  - 従来の `rotateBytes` は現在のスキーマで拒否されます。`openclaw doctor --fix` は古い設定からこれを削除します。
  - `resetArchiveRetention`: リセット／削除されたトランスクリプトアーカイブの経過時間ベースの保持期間。デフォルトでは、アーカイブはディスク容量の制限によって削除されるまで保持されます。実時間に基づく削除を有効にするには期間を設定し、明示的に無効にするには `false` を設定します。
  - `maxDiskBytes`: セッションディレクトリに対する任意のディスク容量制限。`warn` モードでは警告をログに記録し、`enforce` モードでは古いアーティファクト／セッションから順に削除します。
  - `highWaterBytes`: 容量制限のクリーンアップ後に目指す任意の目標値。デフォルトは `maxDiskBytes` の `80%` です。
- **`threadBindings`**: スレッドに紐付けられたセッション機能のグローバルデフォルト。
  - `enabled`: 対応チャネルのスレッド紐付けに対するマスタースイッチ
  - `idleHours`: 非アクティブ時に自動でフォーカスを解除するまでのデフォルト時間（時間単位。`0` で無効化。プロバイダーによるオーバーライドが可能）
  - `maxAgeHours`: デフォルトの絶対最大存続時間（時間単位。`0` で無効化。プロバイダーによるオーバーライドが可能）
  - `spawnSessions`: `sessions_spawn` および ACP スレッド生成からスレッド紐付け作業セッションを作成するためのデフォルトゲート。スレッド紐付けが有効な場合、デフォルトは `true` です。プロバイダー／アカウントによるオーバーライドが可能です。
  - `defaultSpawnContext`: スレッド紐付け生成に使用するデフォルトのネイティブサブエージェントコンテキスト（`"fork"` または `"isolated"`）。デフォルトは `"fork"` です。
- **`sharing`**: 所有者および `operator.admin` 接続が選択できるセッションごとのコラボレーションモードを制御します。すべてのフラグのデフォルトは `true` です。いずれかを `false` に設定すると、その選択肢が Control UI から削除され、作成時の公開範囲指定または `session.visibility.set` で拒否されるようになります。Control UI でドラフトとして開始しない限り、新しいセッションは `shared` で開始されます。
  - `readOnly`: `read-only` を許可します。このモードでは、非メンバーは閲覧できますが、送信、誘導、中止、承認、セッション状態の変更はできません。
  - `suggest`: `suggest` を許可します。現段階では `read-only` と同じ参加許可動作を適用します。提案キューは今後追加される機能です。
  - `drafts`: `draft` を許可します。これにより、管理者でも所有者でもないユーザーのセッション一覧およびイベントブロードキャストからセッションが非表示になります。

メンバーシップと公開範囲の変更は、システムノートとしてセッションのトランスクリプトに書き込まれます。これらの制御は、1 つのエージェントを共有するオペレーター間の調整を目的としており、テナント間のセキュリティ境界ではありません。作業の分離が必要な場合は、個別の Gateway またはエージェントを使用してください。

</Accordion>

---

## メッセージ

```json5
{
  messages: {
    responsePrefix: "🦞", // または "auto"
    ackReaction: "👀",
    ackReactionScope: "group-mentions", // group-mentions | group-all | direct | all | off | none
    queue: {
      mode: "steer", // steer（デフォルト）| followup | collect | interrupt
      debounceMs: 500,
      cap: 20,
      drop: "summarize", // old | new | summarize（デフォルト）
      byChannel: {
        whatsapp: "followup",
        telegram: "followup",
      },
    },
    inbound: {
      debounceMs: 2000, // 0 で無効化
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### 応答接頭辞

チャネル／アカウントごとのオーバーライド: `channels.<channel>.responsePrefix`、`channels.<channel>.accounts.<id>.responsePrefix`。

解決順序（最も具体的な設定が優先）: アカウント → チャネル → グローバル。`""` は無効化し、カスケードを停止します。`"auto"` は `[{identity.name}]` を導出します。

**テンプレート変数:**

| 変数              | 説明                   | 例                          |
| ----------------- | ---------------------- | --------------------------- |
| `{model}`         | 短いモデル名           | `claude-opus-4-6`           |
| `{modelFull}`     | 完全なモデル識別子     | `anthropic/claude-opus-4-6` |
| `{provider}`      | プロバイダー名         | `anthropic`                 |
| `{thinkingLevel}` | 現在の思考レベル       | `high`、`low`、`off`        |
| `{identity.name}` | エージェントの識別名   | （`"auto"` と同じ）          |

変数では大文字と小文字を区別しません。`{think}` は `{thinkingLevel}` のエイリアスです。

### 確認リアクション

- デフォルトはアクティブなエージェントの `identity.emoji` で、設定されていない場合は `"👀"` です。無効にするには `""` を設定します。
- チャネルごとのオーバーライド: `channels.<channel>.ackReaction`、`channels.<channel>.accounts.<id>.ackReaction`。
- 解決順序: アカウント → チャネル → `messages.ackReaction` → アイデンティティのフォールバック。
- 範囲: `group-mentions`（デフォルト）、`group-all`、`direct`、`all`、または `off`/`none`（確認リアクションを完全に無効化）。
- `messages.statusReactions.enabled`: Slack、Discord、Signal、Telegram、WhatsApp でライフサイクルステータスのリアクションを有効にします。
  Discord では、未設定の場合、確認リアクションが有効であればステータスリアクションも有効のままです。
  Slack、Signal、Telegram、WhatsApp では、ライフサイクルステータスのリアクションを有効にするため、明示的に `true` を設定します。
  Slack はデフォルトで、ネイティブのアシスタントスレッドステータスと切り替わる読み込みメッセージを進捗表示に使用し、設定済みの確認リアクションは固定したままにします。

### キュー

- `mode`: セッション実行中に到着した受信メッセージのキュー戦略。デフォルト: `"steer"`。
  - `steer`: 新しいプロンプトをアクティブな実行に挿入します。
  - `followup`: アクティブな実行が終了した後に新しいプロンプトを実行します。
  - `collect`: 互換性のあるメッセージをまとめ、後で一括して実行します。
  - `interrupt`: 最新のプロンプトを開始する前に、アクティブな実行を中止します。
- `debounceMs`: キューに追加／誘導されたメッセージをディスパッチするまでの遅延。デフォルト: `500`。
- `cap`: 破棄ポリシーが適用されるまでのキューメッセージの最大数。デフォルト: `20`。
- `drop`: 上限を超えた場合の戦略。`"summarize"`（デフォルト）は最も古いエントリを破棄しますが、簡潔な要約は保持します。`"old"` は要約を残さず最も古いエントリを破棄し、`"new"` は最新の項目を拒否します。
- `byChannel`: プロバイダー ID をキーとする、チャネルごとの `mode` オーバーライド。
- `debounceMsByChannel`: プロバイダー ID をキーとする、チャネルごとの `debounceMs` オーバーライド。

### 受信デバウンス

同じ送信者から短時間に連続して届いたテキストのみのメッセージを、1 回のエージェントターンにまとめます。メディア／添付ファイルは即座にフラッシュされます。制御コマンドはデバウンスを迂回します。デフォルトの `debounceMs`: `2000`。

### その他のメッセージキー

- `channels.whatsapp.responsePrefix`: WhatsApp の送信返信接頭辞。Doctor は、この正規値が未設定の場合に限り、廃止された受信用 `messagePrefix` の値をここへ移動します。
- `messages.visibleReplies`: ダイレクト、グループ、チャネルの各会話で表示される送信元返信を制御します（`"message_tool"` で表示可能な出力を得るには `message(action=send)` が必要です。`"automatic"` は従来どおり通常の返信を投稿します）。
- `messages.usageTemplate` / `messages.responseUsage`: カスタム `/usage` フッターテンプレートと、返信ごとのデフォルト使用モード（`off | tokens | full`、および `tokens` に対する従来の `on` エイリアス）。
- `messages.groupChat.mentionPatterns` / `historyLimit`: グループメッセージのメンショントリガーと履歴ウィンドウのサイズ設定。
- `messages.suppressToolErrors`: `true` の場合、ユーザーに表示される `⚠️` ツールエラー警告を抑制します（エージェントは引き続きコンテキスト内でエラーを確認でき、再試行できます）。デフォルト: `false`。

### TTS（テキスト読み上げ）

```json5
{
  tts: {
    auto: "off", // off (default) | always | inbound | tagged
    mode: "final", // final | all
    provider: "elevenlabs",
    summaryModel: "openai/gpt-5.4-mini",
    modelOverrides: { enabled: true },
    maxTextLength: 4000,
    timeoutMs: 30000,
    providers: {
      elevenlabs: {
        apiKey: "example-elevenlabs-api-key",
        baseUrl: "https://api.elevenlabs.io",
        speakerVoiceId: "voice_id",
        modelId: "eleven_multilingual_v2",
        seed: 42,
        applyTextNormalization: "auto",
        languageCode: "en",
        voiceSettings: {
          stability: 0.5,
          similarityBoost: 0.75,
          style: 0.0,
          useSpeakerBoost: true,
          speed: 1.0,
        },
      },
      microsoft: {
        speakerVoice: "en-US-MichelleNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
      },
      openai: {
        apiKey: "example-openai-api-key",
        baseUrl: "https://api.openai.com/v1",
        model: "gpt-4o-mini-tts",
        speakerVoice: "coral",
      },
    },
  },
}
```

グローバル設定のパスはマシン状態です（デフォルトは
`~/.openclaw/settings/tts.json`、`OPENCLAW_TTS_PREFS` で上書き可能）。高度な
マルチエージェント構成では、エージェントごとに異なる設定ストアを使用するために
`agents.entries.<id>.tts.prefsPath` を設定できます。

- `auto` はデフォルトの自動 TTS モードを制御します：`off`、`always`、`inbound`、または `tagged`。`/tts on|off` でローカル設定を上書きでき、`/tts status` で有効な状態を確認できます。
- `summaryModel` は、自動要約に使用する `agents.defaults.model.primary` を上書きします。
- `modelOverrides` はデフォルトで有効です（`enabled !== false`）。`modelOverrides.allowProvider` は明示的な有効化が必要です。
- API キーは、`ELEVENLABS_API_KEY`/`XI_API_KEY` および `OPENAI_API_KEY` にフォールバックします。
- バンドルされた音声プロバイダーは Plugin が所有します。`plugins.allow` が設定されている場合は、使用する各 TTS プロバイダー Plugin を含めてください。たとえば、Edge TTS の場合は `microsoft` です。従来の `edge` プロバイダー ID は、`microsoft` のエイリアスとして受け入れられます。
- `providers.openai.baseUrl` は OpenAI TTS エンドポイントを上書きします。解決順序は、設定、`OPENAI_TTS_BASE_URL`、`https://api.openai.com/v1` の順です。
- `providers.openai.baseUrl` が OpenAI 以外のエンドポイントを指す場合、OpenClaw はそれを OpenAI 互換 TTS サーバーとして扱い、モデルと音声の検証を緩和します。

---

## トーク

トークモード（macOS/iOS/Android およびブラウザーの Control UI）のデフォルト設定です。

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        speakerVoiceId: "elevenlabs_voice_id",
        voiceAliases: {
          Clawd: "EXAVITQu4vr4xnSDxMaL",
          Roger: "CwhRBWXzGAHq8TQ4Fs17",
        },
        modelId: "eleven_multilingual_v2",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    consultThinkingLevel: "low",
    consultFastMode: true,
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
    realtime: {
      provider: "openai",
      providers: {
        openai: {
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
      instructions: "Speak warmly and keep answers brief.",
      mode: "realtime", // realtime | stt-tts | transcription
      transport: "webrtc", // webrtc | provider-websocket | gateway-relay | managed-room
      vadThreshold: 0.5,
      silenceDurationMs: 500,
      prefixPaddingMs: 300,
      reasoningEffort: "medium",
      brain: "agent-consult", // agent-consult | direct-tools | none
    },
  },
}
```

- 複数のトークプロバイダーを設定する場合、`talk.provider` は `talk.providers` 内のキーと一致する必要があります。
- 従来のフラットなトークキー（`talk.voiceId`、`talk.voiceAliases`、`talk.modelId`、`talk.outputFormat`、`talk.apiKey`）は互換性のためだけに存在します。`openclaw doctor --fix` を実行して、保存済みの設定を `talk.providers.<provider>` に書き換えてください。
- 音声 ID は、`ELEVENLABS_VOICE_ID` または `SAG_VOICE_ID` にフォールバックします（macOS トーククライアントの動作）。
- `providers.*.apiKey` には、プレーンテキスト文字列または SecretRef オブジェクトを指定できます。
- `ELEVENLABS_API_KEY` へのフォールバックは、トーク API キーが設定されていない場合にのみ適用されます。
- `providers.*.voiceAliases` を使用すると、トークディレクティブでわかりやすい名前を使用できます。
- `providers.mlx.modelId` は、macOS のローカル MLX ヘルパーが使用する Hugging Face リポジトリを選択します。省略した場合、macOS は `mlx-community/Soprano-80M-bf16` を使用します。
- macOS の MLX 再生は、バンドルされた `openclaw-mlx-tts` ヘルパーが存在する場合はそれを介して実行され、存在しない場合は `PATH` 上の実行可能ファイルを使用します。開発時には `OPENCLAW_MLX_TTS_BIN` でヘルパーのパスを上書きできます。
- `consultThinkingLevel` は、Control UI のトークリアルタイム `openclaw_agent_consult` 呼び出しの背後で実行される OpenClaw エージェント全体の思考レベルを制御します。通常のセッションやモデルの動作を維持するには、未設定のままにしてください。
- `consultFastMode` は、セッションの通常の高速モード設定を変更せずに、Control UI のトークリアルタイムコンサルトに対する一度限りの高速モード上書きを設定します。
- `speechLocale` は、Android、iOS、macOS のトーク音声認識で使用する BCP 47 ロケール ID を設定します。Android では、その言語部分をリアルタイム入力の文字起こしにも使用します。デバイスのデフォルトを使用するには、未設定のままにしてください。
- `silenceTimeoutMs` は、ユーザーが話し終えてからトークモードが文字起こしを送信するまでの待機時間を制御します。未設定の場合は、プラットフォームのデフォルトの一時停止時間（`700 ms on macOS and Android, 900 ms on iOS`）が維持されます。
- `realtime.instructions` は、プロバイダー向けのシステム指示を OpenClaw の組み込みリアルタイムプロンプトに追加します。これにより、デフォルトの `openclaw_agent_consult` ガイダンスを失うことなく音声スタイルを設定できます。
- `realtime.vadThreshold` は、プロバイダーの音声アクティビティしきい値を `0`（最高感度）から `1`（最低感度）の範囲で設定します。未設定の場合は、プロバイダーのデフォルトが維持されます。
- `realtime.silenceDurationMs` は、プロバイダーがリアルタイムのユーザーターンを確定するまでの無音時間を正の整数で設定します。未設定の場合は、プロバイダーのデフォルトが維持されます。
- `realtime.prefixPaddingMs` は、音声の検出が始まる前に保持する音声量を負でない整数で設定します。未設定の場合は、プロバイダーのデフォルトが維持されます。
- `realtime.reasoningEffort` は、リアルタイムセッションにおけるプロバイダー固有の推論レベルを設定します。未設定の場合は、プロバイダーのデフォルトが維持されます。
- `realtime.consultRouting`：`"provider-direct"`（デフォルト）は、リアルタイムプロバイダーが `openclaw_agent_consult` なしで最終的なユーザー文字起こしを生成した場合に、プロバイダーからの直接応答を維持します。代わりに、`"force-agent-consult"` は確定したリクエストを OpenClaw 経由で処理します。

---

## 関連項目

- [設定リファレンス](/ja-JP/gateway/configuration-reference) — その他すべての設定キー
- [設定](/ja-JP/gateway/configuration) — 一般的なタスクと簡単なセットアップ
- [設定例](/ja-JP/gateway/configuration-examples)
