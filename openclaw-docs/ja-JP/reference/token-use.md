---
read_when:
    - トークン使用量、コスト、コンテキストウィンドウの説明
    - コンテキスト増大または Compaction の動作のデバッグ
summary: OpenClaw がプロンプトコンテキストを構築し、トークン使用量とコストを報告する仕組み
title: トークン使用量とコスト
x-i18n:
    generated_at: "2026-07-26T09:19:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6624bceb0bcbca769c9d569389b73b82f1ea73133e09f0ae9859833196d85911
    source_path: reference/token-use.md
    workflow: 16
---

OpenClaw は文字数ではなく **トークン**を追跡します。トークンはモデルごとに異なりますが、ほとんどの
OpenAI 形式のモデルでは、英語テキストの場合、平均で 1 トークンあたり約 4 文字です。

## システムプロンプトの構築方法

OpenClaw は実行のたびに独自のシステムプロンプトを組み立てます。これには以下が含まれます。

- ツール一覧と簡単な説明
- Skills 一覧（メタデータのみ。指示は `read` でオンデマンドに読み込まれます）。ネイティブ
  Codex ターンでは、コンパクトな Skills ブロックがターン単位のコラボレーション用
  開発者指示として渡されます。その他のハーネスでは、通常のプロンプトサーフェスに含まれます。
  `skills.limits.maxSkillsPromptChars` によって制限され、`agents.entries.*.skillsLimits.maxSkillsPromptChars` でエージェントごとの
  オーバーライドを任意に指定できます。
- 自己更新の指示
- ワークスペースとブートストラップファイル（`AGENTS.md`、`SOUL.md`、`TOOLS.md`、
  `IDENTITY.md`、`USER.md`、`HEARTBEAT.md`、新規の場合は `BOOTSTRAP.md`、さらに
  存在する場合は `MEMORY.md`）。挿入される大きなファイルは
  `agents.defaults.bootstrapMaxChars`（デフォルト: `20000`）によって切り詰められ、ブートストラップへの
  挿入総量は `agents.defaults.bootstrapTotalMaxChars`（デフォルト:
  `60000`）に制限されます。
  - ネイティブ Codex ターンでは、そのワークスペースでメモリツールが
    利用可能な場合、生の `MEMORY.md` は貼り付けられません。代わりに、ターン単位のコラボレーション用開発者指示に
    短いメモリポインターが含まれ、必要に応じてメモリツールが使用されます。ツールが無効な場合、メモリ検索が利用できない場合、
    またはアクティブなワークスペースがエージェントのメモリワークスペースと異なる場合、`MEMORY.md` は
    通常の制限付きターンコンテキスト経路にフォールバックします。
  - 小文字のルート `memory.md` は挿入されません。これは
    `openclaw doctor --fix` のレガシー修復入力であり、`MEMORY.md` に移行されます。
  - `memory/*.md` の日次ファイルは通常のブートストラッププロンプトには含まれません。
    通常のターンでは、メモリツールを介してオンデマンドで利用されます。リセット時または起動時の
    モデル実行では、その最初のターンに限り、最近の日次メモリを含む単発の起動コンテキストブロックを
    先頭に追加できます。これは
    `agents.defaults.startupContext` で制御されます。単独のチャット `/new` と `/reset` は、
    モデルを呼び出さずに受け付けられます。
  - Compaction 後の `AGENTS.md` 抜粋には、明示的な
    `agents.defaults.compaction.postCompactionSections` によるオプトインが必要です。Plugin は
    `before_prompt_build` を通じて他のコンテキストを追加できます。
- 時刻（UTC + ユーザーのタイムゾーン）
- 返信タグと Heartbeat の動作
- ランタイムメタデータ（ホスト／OS／モデル／思考）

詳細な内訳については、[システムプロンプト](/ja-JP/concepts/system-prompt)を参照してください。

認証情報や認証スニペットを文書化する場合は、ドキュメントのみの変更で
シークレットスキャナーの誤検出を避けるために、
[シークレットプレースホルダー規則](/ja-JP/reference/secret-placeholder-conventions)を使用してください。

## コンテキストウィンドウに算入されるもの

モデルが受け取るすべての内容がコンテキスト上限に算入されます。

- システムプロンプト（上記のすべてのセクション）
- 会話履歴（ユーザーとアシスタントのメッセージ）
- ツール呼び出しとツール結果
- 添付ファイル／文字起こし（画像、音声、ファイル）
- Compaction の要約とプルーニング生成物
- プロバイダーのラッパーまたは安全性ヘッダー（表示されませんが算入されます）

ランタイム負荷の高いサーフェスには、
`agents.defaults.contextLimits` で独自の明示的な上限が設定されています（エージェントごとのオーバーライドは
`agents.entries.*.contextLimits`）。

| キー                      | 目的                                                                  |
| ------------------------ | ------------------------------------------------------------------------ |
| `memoryGetMaxChars`      | `memory_get` が切り詰め前に返す最大文字数。                   |
| `postCompactionMaxChars` | Compaction 後の更新中に `AGENTS.md` から保持する最大文字数。 |

これらは制限付きのランタイム抜粋およびランタイム所有の挿入ブロックであり、
ブートストラップ上限、起動コンテキスト上限、Skills プロンプト上限とは
別に扱われます。

OpenClaw は、有効なモデルのコンテキストウィンドウからライブツール結果の上限を
導出します。100K トークン未満では `16000` 文字、
100K+ トークンでは `32000` 文字、200K+ トークンでは `64000` 文字です。
ランタイムのコンテキスト占有率ガードにより、単一のツール結果はコンテキストウィンドウの 30% にも
制限されます。

コストやレイテンシーを大きく変える大容量のプロバイダーウィンドウは、自動的には
有効化されません。たとえば、OpenAI の GPT-5.5 および GPT-5.6 の直接利用モデルでは
合計 `1050000` トークンのウィンドウが公開されていますが、OpenClaw のアクティブな
ランタイム予算はデフォルトで `272000` トークンです。オプトインの `922000` 入力予算では、
`128000` の出力許容量全体が予約されます。また、入力が `272000` トークンを超えると、OpenAI は
リクエスト全体に長コンテキスト向けの高い料金を適用します。
[OpenAI のコンテキストウィンドウのデフォルト](/ja-JP/providers/openai#context-window-defaults-and-long-context-opt-in)を参照してください。

画像については、OpenClaw はプロバイダーを呼び出す前に、文字起こし／ツールの画像ペイロードを
縮小します。`agents.defaults.imageMaxDimensionPx`（デフォルト:
`1200`）で調整します。

- 値を小さくすると、ビジョントークンの使用量とペイロードサイズが減少します。
- 値を大きくすると、OCR／UI を多用するスクリーンショットでより多くの視覚的詳細が保持されます。

実用的な内訳（挿入ファイルごと、ツール、Skills、システム
プロンプトのサイズ）を確認するには、`/context list` または `/context detail` を使用してください。
[コンテキスト](/ja-JP/concepts/context)を参照してください。

## 現在のトークン使用量を確認する方法

チャットでは次を使用します。

- `/status` -> セッションのモデル、コンテキスト使用量、
  最後の応答の入力／出力トークン、アクティブなモデルにローカル料金が設定されている場合の推定コストを示す、
  絵文字を多用したステータスカード。
- `/usage off|tokens|full` -> すべての返信に応答ごとの使用量フッターを追加します。
  セッションごとに永続化されます（`responseUsage` として保存）。
  - `/usage reset`（別名: `inherit`、`clear`、`default`）は
    セッションのオーバーライドをクリアし、設定済みのデフォルトを再継承させます。
  - `/usage tokens` はターンのトークン／キャッシュの詳細を表示します。
  - `/usage full` はモデル／コンテキスト／コストの簡潔な詳細を表示します。推定コストは、
    OpenClaw にアクティブなモデルの使用量メタデータとローカル料金がある場合にのみ表示されます。
    カスタム `messages.usageTemplate` レイアウトには、トークン／キャッシュフィールドを含めることができます。
- `/usage cost` -> OpenClaw セッションログから取得したローカルコストの要約。

その他のサーフェス:

- **TUI／Web TUI:** `/status` と `/usage` がサポートされています。
- **CLI:** `openclaw status --usage` と `openclaw channels list` は、
  正規化されたプロバイダーのクォータウィンドウ（`X% left`、応答ごとのコストではありません）を表示します。
  現在の使用量ウィンドウ対応プロバイダー: Claude（Anthropic）、ClawRouter、Copilot
  （GitHub）、DeepSeek、Gemini（Google Gemini CLI）、MiniMax、OpenAI、Xiaomi、
  Xiaomi Token Plan、z.ai。

使用量サーフェスでは、表示前に一般的なプロバイダーネイティブのフィールド別名を
正規化します。OpenAI ファミリーの Responses トラフィックでは、
`input_tokens`/`output_tokens` と `prompt_tokens`/`completion_tokens` の両方が含まれるため、
トランスポート固有のフィールド名によって `/status`、`/usage`、またはセッションの
要約が変わることはありません。Gemini CLI の使用量も正規化されます。デフォルトの `stream-json`
パーサーはアシスタントの `message` イベントを読み取り、`stats.cached` は
`cacheRead` にマッピングされます。CLI が明示的な `stats.input` フィールドを省略した場合は、
`stats.input_tokens - stats.cached` が使用されます。レガシー JSON オーバーライドでは、引き続き
`response` から返信テキストを読み取ります。

ネイティブの OpenAI ファミリー Responses トラフィックでは、WebSocket／SSE の使用量別名も
同様に正規化され、`total_tokens` が欠落している場合または `0` の場合、
合計値は正規化された入力と出力の合計にフォールバックします。

現在のセッションスナップショットの情報が少ない場合、`/status` と `session_status` は、
最新の文字起こし使用量ログからトークン／キャッシュカウンターとアクティブなランタイムモデルのラベルを
復元できます。既存のゼロ以外のライブ値は、引き続き文字起こしのフォールバック値より
優先されます。また、保存された合計値が欠落しているか小さい場合は、プロンプト指向の
より大きな文字起こし合計値が優先されることがあります。

プロバイダーのクォータウィンドウに対する使用量認証では、まずプロバイダー固有のフックが
使用されます。プロバイダーにフックがない場合（またはフックでトークンを解決できない場合）、
OpenClaw は認証プロファイル、環境変数、または設定から一致する OAuth／API キー認証情報に
フォールバックします。

アシスタントの文字起こしエントリには、同じ正規化された使用量形式が永続化されます。
アクティブなモデルに料金が設定され、プロバイダーが使用量メタデータを返す場合は、
`usage.cost` も含まれます。これにより、ライブのランタイム状態が失われた後でも、
`/usage cost` と文字起こしベースのセッションステータスで安定したソースを利用できます。

OpenClaw は、プロバイダーの使用量集計と現在のコンテキストスナップショットを
分離して保持します。プロバイダーの `usage.total` には、キャッシュ済み入力、出力、
複数回のツールループでのモデル呼び出しが含まれる場合があるため、コストとテレメトリーには有用ですが、
ライブのコンテキストウィンドウを過大に示す可能性があります。コンテキスト表示と診断では、
`context.used` に最新のプロンプトスナップショット（`promptTokens`、またはプロンプトスナップショットが
利用できない場合は最後のモデル呼び出し）を使用します。

## コストの見積もり（表示される場合）

コストはモデルの料金設定から見積もられます。

```text
models.providers.<provider>.models[].cost
```

これは、`input`、`output`、`cacheRead`、`cacheWrite` についての
**100 万トークンあたりの USD** です。料金が設定されていない場合、`/usage full` はコストを省略します。
すべての返信にトークン／キャッシュの詳細が必要な場合は、`/usage tokens` またはカスタム
`messages.usageTemplate` を使用してください。コスト表示は API キー認証に限定されません。
`aws-sdk` などの API キーを使用しないプロバイダーでも、設定されたモデルエントリに
ローカル料金が含まれ、プロバイダーが使用量メタデータを返す場合は、推定コストを表示できます。

サイドカーとチャンネルが Gateway の準備完了経路に到達すると、OpenClaw は、
ローカル料金がまだ設定されていない構成済みモデル参照に対して、任意のバックグラウンド料金ブートストラップを
開始します。このブートストラップは、リモートの OpenRouter および LiteLLM の料金カタログを取得します。
オフラインまたは制限されたネットワークでこれらのカタログ取得をスキップするには、
`models.pricing.enabled: false` を設定してください。明示的な `models.providers.*.models[].cost` エントリは、
引き続きローカルコストの見積もりに使用されます。

## キャッシュ TTL とプルーニングの影響

プロバイダーのプロンプトキャッシュは、キャッシュ TTL ウィンドウ内でのみ適用されます。OpenClaw は、
任意で **キャッシュ TTL プルーニング**を実行できます。キャッシュ TTL の期限切れ後にセッションを
プルーニングし、キャッシュウィンドウをリセットします。これにより、後続のリクエストは履歴全体を再キャッシュする代わりに、
新たにキャッシュされたコンテキストを再利用します。
セッションが TTL を超えてアイドル状態になった場合でも、キャッシュ書き込みコストを低く抑えられます。

[Gateway の設定](/ja-JP/gateway/configuration)で構成し、動作の詳細については
[セッションのプルーニング](/ja-JP/concepts/session-pruning)を参照してください。

Heartbeat を使用すると、アイドル期間をまたいでキャッシュを**ウォーム**な状態に保つことができます。モデルのキャッシュ
TTL が `1h` の場合、Heartbeat 間隔をそれより少し短く（例: `55m`）設定すると、
プロンプト全体の再キャッシュを回避し、キャッシュ書き込みコストを削減できます。

マルチエージェント構成では、1 つの共有モデル設定を維持したまま、
`agents.entries.*.params.cacheRetention` を使用してエージェントごとにキャッシュ動作を調整できます。

各設定項目の詳細なガイドについては、[プロンプトキャッシュ](/ja-JP/reference/prompt-caching)を参照してください。

Anthropic API の料金体系では、キャッシュ読み取りは入力トークンより大幅に安価ですが、
キャッシュ書き込みにはより高い倍率で課金されます。最新の料金と TTL 倍率については、Anthropic の
プロンプトキャッシュ料金を参照してください。
[https://docs.anthropic.com/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/docs/build-with-claude/prompt-caching)

### 例: Heartbeat で 1h キャッシュをウォームに保つ

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
    heartbeat:
      every: "55m"
```

### 例: エージェントごとのキャッシュ戦略を使用した混在トラフィック

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long" # ほとんどのエージェント向けのデフォルト基準値
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m" # 詳細なセッション用に長期キャッシュをウォーム状態に保つ
    - id: "alerts"
      params:
        cacheRetention: "none" # 集中的に発生する通知ではキャッシュへの書き込みを避ける
```

`agents.entries.*.params` は、選択したモデルの `params` に重ねてマージされるため、
`cacheRetention` だけを上書きし、その他のモデルのデフォルト値は
変更せずに継承できます。

### Anthropic の 1M コンテキスト

OpenClaw は、Opus 4.8、Opus 4.7、Opus 4.6、Sonnet 4.6 などの GA 対応 Claude 4.x モデルで、
Anthropic の 1M コンテキストウィンドウを使用します。これらのモデルでは
`params.context1m: true` は必要ありません。

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        alias: opus
```

古い設定では `context1m: true` を維持できますが、OpenClaw はこの設定について
Anthropic が廃止した `context-1m-2025-08-07` ベータヘッダーを送信しなくなり、
サポート対象外の古い Claude モデルを 1M に拡張することもありません。

要件: 認証情報が長いコンテキストの使用対象である必要があります。対象でない場合、
Anthropic はそのリクエストに対してプロバイダー側のレート制限エラーを返します。

OAuth/サブスクリプショントークン
（`sk-ant-oat-*`）で Anthropic を認証する場合、OpenClaw は OAuth に必要な Anthropic のベータ
ヘッダーを維持しつつ、古い設定に廃止済みの `context-1m-*` ベータが残っていれば削除します。

## トークン負荷を軽減するためのヒント

- 長いセッションを要約するには `/compact` を使用します。
- ワークフロー内の大きなツール出力を切り詰めます。
- スクリーンショットを多用するセッションでは `agents.defaults.imageMaxDimensionPx` を小さくします。
- スキルの説明は短くします（スキル一覧はプロンプトに挿入されます）。
- 冗長な探索作業には、より小さなモデルを使用します。

スキル一覧のオーバーヘッドの正確な計算式については、[Skills](/ja-JP/tools/skills)を参照してください。

## 関連項目

- [API の使用量とコスト](/ja-JP/reference/api-usage-costs)
- [プロンプトキャッシュ](/ja-JP/reference/prompt-caching)
- [使用量の追跡](/ja-JP/concepts/usage-tracking)
