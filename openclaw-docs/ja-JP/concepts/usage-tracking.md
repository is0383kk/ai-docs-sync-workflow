---
read_when:
    - プロバイダーの使用量／クォータ表示を接続しています
    - 使用状況の追跡動作または認証要件について説明する必要がある場合
summary: 使用量追跡の表示と認証情報の要件
title: 使用状況の追跡
x-i18n:
    generated_at: "2026-07-26T09:59:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5a1bc9aeb95cd80a48ab57a18fcd24894fdd6fb71e10e8bea8bae67a8688b78e
    source_path: concepts/usage-tracking.md
    workflow: 16
---

## 概要

- 各プロバイダーの使用量エンドポイントから、プロバイダーの使用量／クォータを直接取得します。プロバイダーの推定請求額ではなく、プロバイダーが報告したプラン名、クォータ期間、残高、支出額、予算、日別コスト履歴、トークン／モデルの内訳、またはアカウント状態の概要のみを使用します。
- 人が読めるクォータ期間の出力は、プロバイダーが消費済みクォータ、残りクォータ、または生のカウントのみを報告する場合でも、`X% left` に正規化されます。リセット可能なクォータ期間がないプロバイダーでは、代わりにプロバイダーの概要テキスト（残高など）が表示されます。
- セッションレベルの `/status` と `session_status` ツールは、ライブセッションのスナップショットにトークン／モデルデータがない場合、セッションのトランスクリプトログにフォールバックします。このフォールバックは、不足しているトークン／キャッシュカウンターを補完し、アクティブなランタイムモデルのラベルを復元できるほか、セッションメタデータが欠落しているか、より小さい場合（`totalTokensFresh !== true`、ゼロ、またはトランスクリプトから導出された値未満）、プロンプト指向のより大きい合計値を優先します。ゼロ以外のライブ値は常にフォールバックより優先されます。

## 表示場所

- チャット内の `/status`：セッショントークンと推定コストを示すステータスカード（API キーモデルのみ）。利用可能な場合は、**現在のモデルプロバイダー**のプロバイダー使用量が、正規化された `X% left` 期間またはプロバイダーの概要テキストとして表示されます。
- チャット内の `/usage off|tokens|full`：応答ごとの使用量フッター。
- チャット内の `/usage cost`：OpenClaw セッションログから集計されたローカルコストの概要。
- CLI：`openclaw status --usage` は、プロバイダーごとの使用量／クォータの完全な内訳を出力します。
- CLI：`openclaw models status` は OAuth／トークン認証プロファイルを一覧表示し、使用量期間がある各プロバイダーの横にその概要を表示します。
- Control UI：**使用量**には、OpenClaw のセッションから導出されたトークンおよび推定コスト分析の上に、プロバイダーのプランと請求カードが表示されます。Anthropic および OpenAI Admin API の認証情報を追加すると、プロバイダーが報告した本日、7 日間、30 日間の支出額、日別トレンド、トークン合計、上位モデル、コストカテゴリも表示されます。
- Control UI：チャットコンポーザーのコンテキストリングのポップオーバーには、サブスクリプションプロバイダーの**プラン使用量**が表示されます。期間ごとのバー（5 時間、週間、モデル単位）、リセット時刻、判明している場合はプロバイダーのプラン（`Max (20x)` など）、追加使用量クレジットが含まれます。プランを通じて請求されるセッションでは、トークン単位の金額見積もりが非表示になります。API 請求のセッションでは、`Est. cost` と種類別コストの内訳が維持されます。Claude Code CLI（`claude-cli`）のセットアップでは、同じ Anthropic サブスクリプション使用量が再利用されます。
- macOS メニューバー：プロバイダー使用量のスナップショットが利用可能な場合、コンテキストの下にルートの「使用量」セクションが表示されます。[メニューバー](/ja-JP/platforms/mac/menu-bar)を参照してください。

`openclaw channels list` はプロバイダー使用量を出力しなくなりました。代わりに、ユーザーを `openclaw status` または `openclaw models list` に案内します。

## Anthropic と OpenAI のコスト履歴

サブスクリプションのクォータと API 請求は、異なるプロバイダーサーフェスです。

- Anthropic のサブスクリプション／セットアップ認証情報では、引き続き Claude のクォータ期間と、任意の追加使用量予算が表示されます。代わりに組織の Usage and Cost API 履歴を表示するには、`ANTHROPIC_ADMIN_KEY` または `ANTHROPIC_ADMIN_API_KEY` を設定します。`sk-ant-admin` で始まる Anthropic プロバイダー認証情報は、自動的に検出されます。
- OpenAI ChatGPT／Codex OAuth では、引き続きプラン、クォータ期間、クレジット残高が表示されます。代わりに組織のコストおよび completions 使用量履歴を表示するには、`OPENAI_ADMIN_KEY` を設定します。必要に応じて `OPENAI_PROJECT_ID` を設定すると、1 つのプロジェクトに限定できます。OpenClaw は、`OPENAI_API_KEY`、プロバイダー設定、または認証プロファイルの推論用認証情報を組織 API に送信することはありません。これらのキーがカスタムエンドポイントに属している可能性があるためです。

Admin 認証情報は、実際の組織請求情報を提供するため優先されます。OpenClaw は、プロバイダーが報告したこれらの合計値と、ローカルのセッション見積もりを合算しません。この 2 つのセクションは、意図的に異なる問いに答えるものです。

## デフォルトの使用量フッターモード

`/usage off|tokens|full` はセッションのフッターを設定し、そのセッションで記憶されます。`messages.responseUsage` は、まだモードを選択していないセッションの初期モードを設定するため、毎回 `/usage` と入力しなくても、デフォルトでフッターを有効にできます。

すべてのチャンネルに 1 つのモードを設定するか、`default` フォールバックを持つチャンネル別のマップを設定します。

```jsonc
{
  "messages": {
    "responseUsage": "tokens",
    // または: { "default": "off", "discord": "full" }
  },
}
```

使用できる値：`"off"`、`"tokens"`、`"full"`、および旧来のエイリアス `"on"`（`"tokens"` として扱われます）。

### 3 つの異なるセッション状態

セッションの `responseUsage` フィールドには、それぞれ異なる意味を持つ 3 つの状態を表現できます。

| 状態               | 保存値                    | 有効なモード                                                        |
| ------------------- | ------------------------------- | --------------------------------------------------------------------- |
| **未設定／継承** | `undefined`（なし）            | `messages.responseUsage` 設定のデフォルト、次に `off` にフォールスルーします。 |
| **明示的にオフ**    | `"off"`（保存済み）                | 常にオフです。オフ以外の設定デフォルトでは、フッターを再度有効にできません。     |
| **明示的にオン**     | `"tokens"` または `"full"`（保存済み） | 設定デフォルトにかかわらず、そのモードになります。                              |

### 優先順位

有効なモード = セッションの上書き → チャンネル設定エントリ → `default` → `off`。

明示的な `/usage off` は、セッションにリテラル値 `"off"` として**永続化**され、「未設定」とは異なります。ユーザーが明示的に無効にした後は、オフ以外の `messages.responseUsage` デフォルトでフッターを再度有効にすることはできません。

### リセットとオフの違い

- `/usage off` はフッターを強制的にオフにし、その選択を永続化します。設定されたオフ以外のデフォルトでこれを上書きすることはできません。
- `/usage reset`（エイリアス：`default`、`inherit`、`inherited`、`clear`、`unpin`）は、セッションの上書きをクリアします。その後、セッションは有効な設定デフォルト（`messages.responseUsage`）を**継承**します。デフォルトが設定されていない場合、フッターはオフのままです。
- 完全なセッションリセット（`/reset` または `/new`）やセッションのロールオーバーでも、明示的な使用量モード設定は**保持**されるため、ユーザーの表示選択はセッションのロールオーバー後も維持されます。上書きをクリアするのは `/usage reset`（およびそのエイリアス）のみです。

### 切り替え動作

引数なしの `/usage` は、オフ → トークン → 完全 → オフの順に切り替わります。切り替えの開始点は、現在の**有効な**モード（セッションの上書きが未設定の場合は設定デフォルトにフォールスルー）であるため、常にユーザーが現在フッターで見ている状態と一致します。

### 設定

設定がない場合は、以前の動作（`/usage` までフッターはオフ）が維持されます。セッションの上書きをクリアし、設定されたデフォルトを再度継承するには、`/usage reset` を使用します。

## カスタム `/usage full` フッター

`/usage tokens` は常にプレーンな `Usage: X in / Y out` 行を表示します（利用可能な場合は、キャッシュと推定コストのサフィックスも追加されます）。以下で説明する高機能なフッターを表示するのは、`/usage full` のみです。

`/usage full` は、モデル、推論、fast／slow、コンテキストウィンドウ、コストを、それらのフィールドが利用可能な場合に表示する組み込みのコンパクトなフッターです。組み込みフッターにテンプレートファイルは不要です。

`messages.usageTemplate` は、高度なカスタムレイアウト専用です。値には JSON ファイルパス（`~` をサポート）またはインラインオブジェクトを指定でき、有効な場合は組み込みフッターを置き換えます。ファイルパスは監視され、変更時にライブで再読み込みされます。

```json
{
  "messages": {
    "usageTemplate": "~/.openclaw/usage-footer.json"
  }
}
```

テンプレートがないか空の場合は、通知なしで組み込みフッターにフォールバックします。設定されたテンプレートを読み取れない場合や無効な場合（不正な JSON、または表示可能な出力要素がない形式）も、組み込みフッターにフォールバックし、オペレーター向けの警告を出力します。

カスタムテンプレートは組み込みの形式から始め、変更したい部分を編集します。

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": {
    "braille": "⠐⡀⡄⡆⡇⣇⣧⣷⣿",
    "block": "░▏▎▍▌▋▊▉█",
    "shade": "░▒▓█",
    "moon": "🌑🌘🌗🌖🌕",
    "level": "▁▂▃▄▅▆▇█",
    "weather": ["🥶", "☁️", "🌥", "⛅️", "🌤", "☀️"],
    "plants": ["🪾", "🍂", "🌱", "☘️", "🍀", "🌿"],
    "moons6": ["🌑", "🌚", "🌘", "🌗", "🌖", "🌝"],
  },
  "aliases": {
    "models": {
      "claude-opus-4-6": "opus46",
      "claude-opus-4-8": "opus48",
      "claude-sonnet-4-6": "sonnet46",
      "claude-haiku-4-5": "haiku45",
      "gpt-5.5": "gpt5.5",
    },
    "reasoning": {
      "off": "🌑",
      "minimal": "🌚",
      "low": "🌘",
      "medium": "🌗",
      "high": "🌕",
      "xhigh": "🌝",
    },
  },
  "output": {
    "sep": "",
    "default": [
      { "text": "{model.provider}{identity.emoji|🤖}{model.display_name|alias:models}" },
      { "map": "model.is_fallback", "cases": { "true": "🔄" } },
      { "map": "model.is_override", "cases": { "true": "📌" } },
      { "when": "model.reasoning", "text": "{model.reasoning|alias:reasoning}" },
      { "map": "state.fast_mode", "cases": { "true": "⚡️", "false": "🐌" } },
      {
        "when": "context.max_tokens",
        "text": " | 📚[{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
      },
      { "when": "cost.turn_usd", "text": " 💰{cost.turn_usd|fixed:4}" },
    ],
    "surfaces": {
      "discord": [
        { "text": "-# -\n" },
        { "text": "-# {model.provider}{identity.emoji|🤖}{model.display_name|alias:models}" },
        { "map": "model.is_fallback", "cases": { "true": "🔄" } },
        { "map": "model.is_override", "cases": { "true": "📌" } },
        { "when": "model.reasoning", "text": "{model.reasoning|alias:reasoning}" },
        { "map": "state.fast_mode", "cases": { "true": "⚡️", "false": "🐌" } },
        {
          "when": "context.max_tokens",
          "text": " | 📚[{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
        },
        { "when": "cost.turn_usd", "text": " 💰{cost.turn_usd|fixed:4}" },
      ],
    },
  },
}
```

### 形式

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": { "<name>": "低から高へのグリフ" }, // 文字列（1 文字／グリフ）または配列
  "aliases": { "<table>": { "<value>": "<label>" } },
  "output": {
    "sep": "", // 残った要素を結合
    "default": [/* pieces */], // 任意のサーフェスのフォールバック
    "surfaces": {
      "discord": [/* pieces */],
      "telegram": [/* pieces */],
    },
  },
}
```

各サーフェスは、順序付けられた**要素**のリストです。エンジンは各要素を表示し、空のものを除外して、残ったものを `sep` で結合します。エントリのないサーフェスでは、`output.default` が使用されます。

### コントラクトパス

各要素は、ターンごとのコントラクトからドットパスで値を読み取ります。存在しない値は空になります（そのため、`when` ガードまたは `|fallback` により、要素をクリーンに保てます）。

| パス                                                                                | 意味                                                                                              |
| ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `surface`                                                                           | チャンネル ID（`discord`/`telegram`/など）                                                               |
| `agentId` / `chat_type`                                                             | 所有エージェント ID / チャットサーフェス種別                                                                  |
| `model.id` / `model.display_name` / `model.provider`                                | モデル ID / 表示名 / プロバイダー ID                                                                |
| `model.actual`, `model.resolved_ref`                                                | そのターンで実際に使用されたプロバイダー/モデル参照                                                        |
| `model.requested`                                                                   | 要求されたプロバイダー/モデル参照（フォールバック前）                                                       |
| `model.reasoning`                                                                   | エフォート（`off` から `xhigh`）                                                                       |
| `model.is_fallback` / `model.is_override`                                           | 真偽値：フォールバック使用済み / モデル固定済み                                                                   |
| `model.override_source` / `model.auth_mode`                                         | オーバーライド元ラベル / 認証情報モード（`oauth`, `api-key`, `token`, `mixed`, `aws-sdk`, `unknown`） |
| `state.fast_mode`                                                                   | 真偽値：高速か低速か                                                                                   |
| `state.compactions`                                                                 | セッションの Compaction 回数                                                                     |
| `context.max_tokens` / `context.used_tokens` / `context.pct_used`                   | ウィンドウ予算 / 使用中のトークン数 / 使用率 0～100                                                         |
| `usage.input_tokens` / `usage.output_tokens` / `usage.total_tokens`                 | ターン集計                                                                                       |
| `usage.cache_read_tokens` / `usage.cache_write_tokens`                              | そのターンのキャッシュ読み取りトークン数とキャッシュ書き込みトークン数                                                       |
| `usage.has_tokens` / `usage.has_split_tokens` / `usage.has_total_only_tokens`       | トークン表示ガード                                                                                 |
| `usage.cache_hit_pct`                                                               | プロンプトトークン総数に占めるキャッシュ読み取りの割合                                                              |
| `usage.last.input_tokens` / `usage.last.output_tokens` / `usage.last.cache_hit_pct` | 最後のモデル呼び出しのみ（`cache_read_tokens`, `cache_write_tokens`, `total_tokens` も含む）           |
| `cost.turn_usd` / `cost.available`                                                  | 推定ターンコスト / コスト表を解決できたかどうか                                                  |
| `timing.duration_ms`                                                                | 実時間でのターン所要時間                                                                             |
| `identity.name` / `identity.emoji` / `identity.avatar`                              | エージェントのアイデンティティ名 / 絵文字 / アバター                                                                 |
| `session.id`                                                                        | セッション ID                                                                                           |

（プロバイダーのレート制限ウィンドウはこの契約には**含まれません**。現在、配列値のパスは存在しないため、`each` ピースには反復する対象がありません。）

### 動詞

値を左から右へ動詞に通します。動詞ではないセグメントはフォールバックになります。

| 動詞            | 効果                                | 例                           |
| --------------- | ------------------------------------- | --------------------------------- |
| `num`           | コンパクトな数値表記                         | `272000 -> 272k`                  |
| `fixed:N`       | 小数点以下 N 桁（`0..100`、デフォルトは 2）      | `0.0377`                          |
| `dur`           | 秒を期間表記に変換                   | `14820 -> 4h07m`                  |
| `pct`           | `%` を追加                            | `96 -> 96%`                       |
| `inv`           | `100 - x`                             | 使用量から残量への変換             |
| `alias:TABLE`   | `aliases` で検索し、未登録ならそのまま出力 | `medium -> 🌗`                    |
| `meter:W:SCALE` | 0～100 の値を W セルのグリフバーで表示   | `[⣿⣿⠐⠐⠐]`（`meter:1` = 1 グリフ） |

`fixed:N` は、0 から 100 までの完全な十進整数のみを受け付けます。無効な
精度引数を指定すると、その補間は空になります。

`meter:W:SCALE` は、1 から 100 までの完全な十進整数の幅のみを受け付けます。幅を空欄にするとデフォルトの 5（`meter::braille`）が使用されます。無効な
幅を指定すると、その補間は空になります。

### ピースの形式

- `{ "text": "📚 {context.max_tokens|num}" }`：リテラル + 補間。
- `{ "when": "<path>", "text": "..." }`：パスが真値の場合のみレンダリング。
- `{ "map": "<path>", "cases": { "true": "⚡", "false": "🐌" } }`：値をグリフに変換（`_default` ケースが一致しない値を処理）。
- `{ "each": "<array-path>", "item": "{label}" }`：配列値のパスを反復（現在の契約では配列となるパスはありません）。

### 例

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": { "braille": "⠐⡀⡄⡆⡇⣇⣧⣷⣿" },
  "aliases": { "reasoning": { "medium": "🌗", "high": "🌕" } },
  "output": {
    "surfaces": {
      "discord": [
        { "text": "{model.display_name}" },
        { "when": "model.reasoning", "text": " {model.reasoning|alias:reasoning}" },
        { "map": "state.fast_mode", "cases": { "true": " ⚡", "false": " 🐌" } },
        {
          "when": "context.max_tokens",
          "text": " | 📚 [{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
        },
      ],
    },
  },
}
```

たとえば `claude-sonnet-4-6 🌗 🐌 | 📚 [⣿⣿⣿⣿⣧]272k` のようにレンダリングされます。

## プロバイダーと認証情報

使用可能なプロバイダー使用量認証を解決できない場合、使用量は非表示になります。OpenClaw は、
`contracts.usageProviders` を宣言し、`resolveUsageAuth` と
`fetchUsageSnapshot` の両方を実装する、有効なプロバイダー Plugin を自動的に検出します。
コアに独立したプロバイダー許可リストはありません。静的な
契約により、すべてのプロバイダー Plugin をインポートすることなく検出範囲を限定できます。各
Plugin は、独自のアップストリームエンドポイントとレスポンスマッピングを所有します。
共有スナップショットは、プラン名、クォータウィンドウ、残高、支出、予算を
プロバイダーに依存しない形式で保持し、CLI、アプリ、Control UI の利用側に提供します。

- **Anthropic（Claude）**：認証プロファイル内の OAuth トークン。OAuth トークンに
  `user:profile` スコープがない場合、設定されていれば `claude.ai` Web セッション（`CLAUDE_AI_SESSION_KEY`、
  `CLAUDE_WEB_SESSION_KEY`、または `CLAUDE_WEB_COOKIE` 内の `sessionKey=` Cookie）にフォールバックします。
  Anthropic から報告された場合は、モデル単位の制限と、有効化された追加使用量の月間支出/予算が
  含まれます。明示的な Anthropic Admin API キー、または
  自動検出された `sk-ant-admin...` プロバイダープロファイルを使用すると、代わりに過去 30 日間の
  組織コストと Messages API 履歴が表示されます。
- **ClawRouter**：API キー（`CLAWROUTER_API_KEY`）。設定されている場合は月次予算ウィンドウと
  型付きの USD 予算を表示します。それ以外の場合は、支出合計と
  リクエスト/トークン/コストの概要を表示します。
- **DeepSeek**：環境変数/設定/認証ストア経由の API キー（`DEEPSEEK_API_KEY`）。
  プロバイダーから報告された各通貨の残高を表示します。
- **GitHub Copilot**：認証プロファイル内の OAuth トークン。
- **Gemini CLI**：認証プロファイル内の OAuth トークン。
- **MiniMax**：API キーまたは MiniMax OAuth 認証プロファイル。OpenClaw は
  `minimax`、`minimax-cn`、`minimax-portal` を同じ MiniMax クォータ
  サーフェスとして扱い、保存済みの MiniMax OAuth が存在する場合はそれを優先し、それ以外の場合は
  `MINIMAX_CODE_PLAN_KEY`、`MINIMAX_CODING_API_KEY`、または `MINIMAX_API_KEY` にフォールバックします。
  使用量のポーリングでは、設定されている場合は `models.providers.minimax-portal.baseUrl`
  または `models.providers.minimax.baseUrl` から Coding Plan ホストを導出し、それ以外の場合は
  MiniMax CN ホストを使用します。
  MiniMax の生の `usage_percent` / `usagePercent` フィールドは**残り**
  クォータを意味するため、OpenClaw は表示前に値を反転します。存在する場合は
  カウントベースのフィールドが優先されます。
  - ウィンドウラベルは、存在する場合はプロバイダーの時間/分フィールドから取得し、それ以外の場合は
    `start_time` / `end_time` の期間にフォールバックします。
  - コーディングプランのエンドポイントが `model_remains` を返す場合、OpenClaw は
    チャットモデルのエントリを優先し、明示的な
    `window_hours` / `window_minutes` フィールドがない場合はタイムスタンプからウィンドウラベルを導出し、プラン
    ラベルにモデル名を含めます。
- **OpenAI（Codex/ChatGPT プラン）**：認証プロファイル内の OAuth トークン（アカウント ID がある場合は
  `ChatGPT-Account-Id` ヘッダーを送信）。ChatGPT プラン、リセット可能な
  Codex ウィンドウ、報告された場合はクレジット残高を表示します。クレジットはプロバイダーの
  クレジットのままであり、OpenClaw はドルとして表示しません。`OPENAI_ADMIN_KEY` は、
  キーに Usage Dashboard へのアクセス権がある場合、過去 30 日間の組織コストと completions 使用量履歴を追加します。
  推論用の認証情報が組織 API に転送されることはありません。
- **OpenRouter**：API キーまたは OAuth を基盤とする API キー（`OPENROUTER_API_KEY` または認証
  プロファイル）。アカウントクレジットエンドポイントとキークォータエンドポイントを組み合わせるため、
  認証情報からアクセスできる場合は、アカウント残高/支出、キー予算、日次/週次/月次の使用量が
  表示されます。どちらのエンドポイントも個別にスナップショットを
  補足できます。
- **Venice**：環境変数/設定/認証ストア経由の API キー（`VENICE_API_KEY`）。報告された場合は USD と
  DIEM の残高、および DIEM エポック割り当ての使用量を表示します。
- **Xiaomi MiMo**：2 つの独立した使用量サーフェス。従量課金では API キー
  （`XIAOMI_API_KEY`）を使用し、Token Plan では別のキー（`XIAOMI_TOKEN_PLAN_API_KEY`）を使用します。
  現在、どちらもクォータウィンドウを報告しません。
- **z.ai**：環境変数/設定/認証ストア経由の API キー（`ZAI_API_KEY` または `Z_AI_API_KEY`）。

## 関連項目

- [トークン使用量とコスト](/ja-JP/reference/token-use)
- [API 使用量とコスト](/ja-JP/reference/api-usage-costs)
- [プロンプトキャッシュ](/ja-JP/reference/prompt-caching)
- [メニューバー](/ja-JP/platforms/mac/menu-bar)
