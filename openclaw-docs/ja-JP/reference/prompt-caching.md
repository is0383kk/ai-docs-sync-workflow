---
read_when:
    - キャッシュ保持によってプロンプトのトークンコストを削減したい場合
    - マルチエージェント構成では、エージェントごとのキャッシュ動作が必要です
    - Heartbeat とキャッシュ TTL のプルーニングを同時に調整しています
summary: プロンプトキャッシュの設定項目、マージ順序、プロバイダーの動作、チューニングパターン
title: プロンプトキャッシュ
x-i18n:
    generated_at: "2026-07-26T10:29:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 99dfd3d226d37014110adf16818051236114dcb0277e9b4d13eaced0f1fc03aa
    source_path: reference/prompt-caching.md
    workflow: 16
---

プロンプトキャッシュにより、モデルプロバイダーは変更されていないプロンプトのプレフィックス（システム／開発者向け指示、ツール定義、その他の安定したコンテキスト）を、リクエストごとに再処理する代わりにターン間で再利用できます。これにより、コンテキストが繰り返される長時間実行セッションで、トークンコストとレイテンシーが削減されます。

OpenClaw は、上流 API がこれらのカウンターを公開している場合、プロバイダーの使用量を `cacheRead` と `cacheWrite` に正規化します。使用量の概要（`/status` など）は、ライブセッションのスナップショットにキャッシュカウンターがない場合、最後のトランスクリプト使用量エントリにフォールバックします。ゼロ以外のライブ値は常にフォールバックより優先されます。

プロバイダーの参考資料：

- [Anthropic のプロンプトキャッシュ](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [OpenAI のプロンプトキャッシュ](https://developers.openai.com/api/docs/guides/prompt-caching)

## 主要な設定項目

### `cacheRetention`

値：`"none" | "short" | "long"`。グローバルデフォルト、モデル単位、エージェント単位で設定できます。
`"standard"` はエイリアスではありません。プロバイダーのデフォルトキャッシュ期間には `"short"` を使用してください。無効な値は警告とともに無視されます。

```yaml
agents:
  defaults:
    params:
      cacheRetention: "long" # none | short | long
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "short" # このモデルについてグローバルデフォルトを上書き
  list:
    - id: "alerts"
      params:
        cacheRetention: "none" # このエージェントについて両方のデフォルトを上書き
```

マージ順序（後の設定が優先）：

1. `agents.defaults.params` - すべてのモデルに対するグローバルデフォルト
2. `agents.defaults.models["provider/model"].params` - モデル単位の上書き
3. `agents.entries.*.params` - エージェント ID で照合される、エージェント単位の上書き

ソース：`src/agents/embedded-agent-runner/extra-params.ts`（`resolveExtraParams`）。

### `contextPruning.mode: "cache-ttl"`

キャッシュの TTL 期間が経過した後に古いツール結果のコンテキストを削除し、アイドル後のリクエストで肥大化した履歴が再キャッシュされないようにします。

```yaml
agents:
  defaults:
    contextPruning:
      mode: "cache-ttl"
      ttl: "1h"
```

完全な動作については、[セッションのプルーニング](/ja-JP/concepts/session-pruning)を参照してください。

### Heartbeat によるウォーム状態の維持

Heartbeat はキャッシュ期間をウォーム状態に保ち、アイドル期間後に繰り返されるキャッシュ書き込みを削減できます。グローバル（`agents.defaults.heartbeat`）またはエージェント単位（`agents.entries.*.heartbeat`）で設定できます。

```yaml
agents:
  defaults:
    heartbeat:
      every: "55m"
```

## プロバイダーの動作

### Anthropic（直接 API および Vertex AI）

- `cacheRetention` は、`anthropic` および `anthropic-vertex` プロバイダー、さらに `amazon-bedrock` 上の Claude モデル、および `cacheRetention` が明示的に設定されたカスタムの `anthropic-messages` 互換エンドポイントでサポートされます。
- 未設定の場合、OpenClaw は直接 Anthropic に対して `cacheRetention: "short"` を設定します（`anthropic` および `anthropic-vertex` プロバイダーのみ。他の Anthropic 系ルートでは明示的な値が必要です）。
- ネイティブの Anthropic Messages レスポンスは `cache_read_input_tokens` と `cache_creation_input_tokens` を公開し、それぞれ `cacheRead` と `cacheWrite` にマッピングされます。
- `cacheRetention: "short"` はデフォルトの 5 分間の一時キャッシュにマッピングされます。`cacheRetention: "long"` を明示的に設定すると、1 時間の TTL（`cache_control: { type: "ephemeral", ttl: "1h" }`）を要求します。暗黙的または環境変数による長期保持（明示的な `cacheRetention` を伴わない `OPENCLAW_CACHE_RETENTION=long`）は、`api.anthropic.com` または Vertex AI（`aiplatform.googleapis.com` / `*-aiplatform.googleapis.com`）ホストでのみ 1 時間の TTL に延長されます。他のホストでは 5 分間のキャッシュが維持されます。

ソース：`packages/ai/src/transports/anthropic-payload-policy.ts`（`resolveAnthropicEphemeralCacheControl`、`isLongTtlEligibleEndpoint`）。

### OpenAI（直接 API）

- サポート対象の新しいモデルではプロンプトキャッシュが自動的に機能します。OpenClaw はブロック単位のキャッシュマーカーを挿入しません。
- OpenClaw は、ターン間でキャッシュルーティングを安定させるために `prompt_cache_key` を送信します。直接の `api.openai.com` ホストでは、これは自動的に行われます。OpenAI 互換プロキシ（oMLX、llama.cpp、カスタムエンドポイント）では、オプトインするためにモデル設定で `compat.supportsPromptCacheKey: true` が必要です。プロキシに対して自動検出されることはありません。
- `prompt_cache_retention: "24h"` は、`cacheRetention: "long"` が選択され、解決されたエンドポイントがキャッシュキーと長期保持の両方をサポートする場合にのみ追加されます（`compat.supportsLongCacheRetention`、デフォルトは true。Together AI および Cloudflare の互換プロファイルでは無効）。`cacheRetention: "none"` は両方のフィールドを抑制します。
- キャッシュヒットは `usage.prompt_tokens_details.cached_tokens`（Chat Completions）または `input_tokens_details.cached_tokens`（Responses API）を通じて公開され、`cacheRead` にマッピングされます。
- Responses API のペイロードは `input_tokens_details.cache_write_tokens` も公開する場合があり、これは `cacheWrite` にマッピングされ、モデルのキャッシュ書き込みレートで課金されます。このフィールドを省略する Responses ペイロードでは、`cacheWrite` は `0` のままです。OpenAI の Chat Completions API は `cache_write_tokens` カウンターを文書化も出力もしませんが、OpenClaw は個別の書き込み数を報告する OpenRouter 互換および DeepSeek 形式のプロキシ向けに、そこで `prompt_tokens_details.cache_write_tokens` を引き続き読み取ります。
- 実際には、OpenAI は Anthropic の移動する全履歴再利用よりも、初期プレフィックスキャッシュに近い動作をします。以下の [OpenAI のライブ環境での想定](#openai-live-expectations)を参照してください。

### Amazon Bedrock

- Anthropic Claude のモデル参照（`amazon-bedrock/*anthropic.claude*`、および AWS システム推論プロファイルのプレフィックス `us.`/`eu.`/`global.anthropic.claude*`）は、明示的な `cacheRetention` のパススルーをサポートします。
- Anthropic 以外の Bedrock モデル（例：`amazon.nova-*`）は、設定された `cacheRetention` の値にかかわらず、実行時にはキャッシュ保持なしとして解決されます。
- 不透明な Bedrock アプリケーション推論プロファイル ARN（`claude` を含まないプロファイル ID）も、ARN だけではモデルファミリーを推測できないため、`cacheRetention` が明示的に設定されていない限り、キャッシュ保持なしとして解決されます。

### OpenRouter

`openrouter/anthropic/*` のモデル参照では、リクエストの送信先が検証済みの OpenRouter ルート（デフォルトエンドポイント上の `openrouter`、または `openrouter.ai` に解決される任意のプロバイダー／ベース URL）のままである場合にのみ、OpenClaw はシステム／開発者向けプロンプトブロックに Anthropic の `cache_control` マーカーを挿入します。モデルの送信先を任意の OpenAI 互換プロキシ URL に変更すると、この挿入は停止します。

`contextPruning.mode: "cache-ttl"` は、`openrouter/anthropic/*`、`openrouter/deepseek/*`、`openrouter/moonshot/*`、`openrouter/moonshotai/*`、および `openrouter/zai/*` のモデル参照で使用できます。これらのルートは OpenClaw によるマーカー挿入を必要とせず、プロバイダー側のプロンプトキャッシュを処理するためです。

ソース：`extensions/openrouter/index.ts`（`OPENROUTER_CACHE_TTL_MODEL_PREFIXES`）。

OpenRouter 上での DeepSeek キャッシュの構築はベストエフォートであり、数秒かかる場合があります。直後のフォローアップリクエストでは、引き続き `cached_tokens: 0` が表示されることがあります。少し待った後、同じプレフィックスを繰り返すリクエストを使用し、`usage.prompt_tokens_details.cached_tokens` をキャッシュヒットのシグナルとして確認してください。

### Google Gemini（直接 API）

- 直接 Gemini トランスポート（`api: "google-generative-ai"`）は、上流の `cachedContentTokenCount` を通じてキャッシュヒットを報告し、`cacheRead` にマッピングされます。
- 対象となるモデルファミリー：`gemini-2.5*` および `gemini-3*`（このプレフィックス一致の対象外である Live／プレビューのバリアント、たとえば `gemini-live-2.5-flash-preview` は除外）。
- 対象モデルに `cacheRetention` が設定されている場合、OpenClaw はシステムプロンプト用の `cachedContents` リソースを自動的に作成、再利用、更新します。キャッシュ済みコンテンツのハンドルを手動で指定する必要はありません。TTL は `cacheRetention: "short"` の場合は `300s`、`"long"` の場合は `3600s` です。
- 既存の Gemini キャッシュ済みコンテンツのハンドルを `params.cachedContent`（または従来の `params.cached_content`）として引き続き渡すこともできます。明示的なハンドルを指定すると、自動キャッシュ管理パスは完全にスキップされます。
- これは Anthropic／OpenAI のプロンプトプレフィックスキャッシュとは別の仕組みです。OpenClaw はインラインキャッシュマーカーを挿入する代わりに、Gemini 用のプロバイダーネイティブな `cachedContents` リソースを管理します。

ソース：`src/agents/embedded-agent-runner/google-prompt-cache.ts`。

### CLI ハーネスプロバイダー（Claude Code、Gemini CLI）

JSONL 使用量イベント（`jsonlDialect: "claude-stream-json"` または `"gemini-stream-json"`）を出力する CLI バックエンドは、複数のフィールド名バリアントを認識する共有使用量パーサーを経由します。これには、`cacheRead` にマッピングされる単純な `cached` カウンターも含まれます。CLI の JSON ペイロードに直接の入力トークンフィールドがない場合、OpenClaw はそれを `input_tokens - cached` として導出します。これは使用量の正規化のみであり、CLI で駆動されるこれらのモデルに Anthropic／OpenAI 形式のプロンプトキャッシュマーカーを作成するものではありません。

ソース：`src/agents/cli-output.ts`（`toCliUsage`）。

### その他のプロバイダー

プロバイダーが上記のいずれのキャッシュモードもサポートしていない場合、`cacheRetention` は効果を持ちません。

## システムプロンプトのキャッシュ境界

OpenClaw は内部のキャッシュプレフィックス境界で、システムプロンプトを**安定したプレフィックス**と**変動するサフィックス**に分割します。境界より上のコンテンツ（ツール定義、Skills メタデータ、ワークスペースファイル）は、ターン間でバイト単位の同一性を維持するように順序付けられます。境界より下のコンテンツ（たとえば `HEARTBEAT.md`、実行時のタイムスタンプ、その他のターン単位のメタデータ）は、キャッシュされたプレフィックスを無効にせずに変更できます。

主な設計上の選択：

- 安定したワークスペースのプロジェクトコンテキストファイルは `HEARTBEAT.md` より前に配置されるため、Heartbeat による変動で安定したプレフィックスが無効になることはありません。
- この境界は Anthropic 系、OpenAI 系、Google、および CLI のトランスポート整形に共通して適用されるため、サポートされるすべてのプロバイダーが同じプレフィックス安定性の恩恵を受けます。
- Codex Responses および Anthropic Vertex のリクエストは境界を考慮したキャッシュ整形を経由するため、キャッシュの再利用がプロバイダーに実際に送信される内容と一致します。
- システムプロンプトのフィンガープリントは正規化されるため（空白、改行、フックによって追加されたコンテキスト、実行時機能の順序）、意味的に変更されていないプロンプトはターン間でキャッシュを共有します。

設定またはワークスペースの変更後に予期しない `cacheWrite` の急増が見られる場合、その変更がキャッシュ境界の上側と下側のどちらに配置されているかを確認してください。変動するコンテンツを境界より下に移動する（または安定化する）ことで、通常は問題が解決します。

## OpenClaw のキャッシュ安定性ガード

- バンドルされた MCP ツールカタログは、ツール登録前に決定論的に（サーバー名、次にツール名の順で）ソートされます。そのため、`listTools()` の順序変更によってツールブロックが変動し、プロンプトキャッシュのプレフィックスが無効になることはありません。
- 永続化された画像ブロックを含む従来のセッションでは、**完了済みの最新 3 ターン**がそのまま維持されます（画像を含むターンだけでなく、すべての完了済みターンを数えます）。それ以前の処理済み画像ブロックはテキストマーカーに置き換えられるため、画像を多用するフォローアップで古く大きなペイロードが繰り返し送信されることはありません。

## チューニングパターン

### 混在トラフィック（推奨デフォルト）

メインエージェントでは長期間維持されるベースラインを使用し、突発的に動作する通知エージェントではキャッシュを無効にします。

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m"
    - id: "alerts"
      params:
        cacheRetention: "none"
```

### コスト優先のベースライン

- ベースラインの `cacheRetention: "short"` を設定します。
- `contextPruning.mode: "cache-ttl"` を有効にします。
- ウォームキャッシュの恩恵を受けるエージェントについてのみ、Heartbeat の間隔を TTL より短く保ちます。

## ライブ回帰テスト

OpenClaw は、繰り返されるプレフィックス、ツールターン、画像ターン、MCP 形式のツールトランスクリプト、および Anthropic のキャッシュなし制御を対象とする、単一の統合ライブキャッシュ回帰ゲートを実行します。

- `src/agents/live-cache-regression.live.test.ts`
- `src/agents/live-cache-regression-runner.ts`
- `src/agents/live-cache-regression-baseline.ts`

次のコマンドで実行します。

```sh
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache
```

ベースラインファイルには、直近で観測されたライブ数値と、テストで照合するプロバイダー固有の回帰下限値が保存されます。各実行では、実行ごとに新しいセッション ID とプロンプト名前空間を使用するため、以前のキャッシュ状態が現在のサンプルに影響することはありません。Anthropic と OpenAI では適用方法が異なります。Anthropic で下限値を下回ると重大な回帰として扱われ（テストは失敗します）、OpenAI で下限値を下回っても監視対象にとどまります（警告として記録されますが、実行は失敗しません）。両者で単一のプロバイダー横断しきい値を共有することはありません。

### Anthropic のライブ環境での期待値

- `cacheWrite` を介した明示的なウォームアップ書き込みを期待します。
- Anthropic のキャッシュ制御では、会話の進行に伴ってキャッシュのブレークポイントが移動するため、ターンを繰り返した際には履歴のほぼ全体が再利用されることを期待します。
- 安定、ツール、画像、MCP 形式の各レーンに対するベースライン下限値は、厳格な回帰ゲートです。

### OpenAI のライブ環境での期待値

- `cacheRead` のみを期待します。Chat Completions では `cacheWrite` は `0` のままです。
- ターンを繰り返した際のキャッシュ再利用は、Anthropic 形式の移動する全履歴再利用ではなく、プロバイダー固有のプラトーとして扱います。
- 下限値は監視専用です（下回った場合は警告として記録されますが、テストは失敗しません）。`gpt-5.4-mini` で観測されたライブ動作から導出されています。

| シナリオ             | `cacheRead` の下限値 | ヒット率の下限値 |
| -------------------- | ----------------: | -------------: |
| 安定したプレフィックス        |             4,608 |           0.90 |
| ツールトランスクリプト      |             4,096 |           0.85 |
| 画像トランスクリプト     |             3,840 |           0.82 |
| MCP 形式のトランスクリプト |             4,096 |           0.85 |

直近で観測されたベースライン数値（`live-cache-regression-baseline.ts` から）は、安定したプレフィックスが `cacheRead=4864`、ヒット率が `0.966`、ツールトランスクリプトが `cacheRead=4608`、ヒット率が `0.896`、画像トランスクリプトが `cacheRead=4864`、ヒット率が `0.954`、MCP 形式のトランスクリプトが `cacheRead=4608`、ヒット率が `0.891` でした。

アサーションが異なる理由は、Anthropic が明示的なキャッシュブレークポイントと、移動する会話履歴の再利用を公開しているのに対し、ライブトラフィックにおける OpenAI の実質的に再利用可能なプレフィックスは、プロンプト全体に達する前にプラトーになる場合があるためです。2 つのプロバイダーを単一のプロバイダー横断パーセンテージしきい値と比較すると、誤った回帰判定が発生します。

## `diagnostics.cacheTrace` の設定

```yaml
diagnostics:
  cacheTrace:
    enabled: true
    filePath: "~/.openclaw/logs/cache-trace.jsonl" # 任意
    includeMessages: false # デフォルトは true
    includePrompt: false # デフォルトは true
    includeSystem: false # デフォルトは true
```

デフォルト：

| キー               | デフォルト                                      |
| ----------------- | -------------------------------------------- |
| `filePath`        | `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl` |
| `includeMessages` | `true`                                       |
| `includePrompt`   | `true`                                       |
| `includeSystem`   | `true`                                       |

### 環境変数による切り替え（一時的なデバッグ）

| 変数                             | 効果                               |
| ------------------------------------ | ------------------------------------ |
| `OPENCLAW_CACHE_TRACE=1`             | キャッシュトレースを有効にする                |
| `OPENCLAW_CACHE_TRACE_FILE=path`     | 出力パスを上書きする                |
| `OPENCLAW_CACHE_TRACE_MESSAGES=0\|1` | メッセージペイロード全体の取得を切り替える |
| `OPENCLAW_CACHE_TRACE_PROMPT=0\|1`   | プロンプトテキストの取得を切り替える          |
| `OPENCLAW_CACHE_TRACE_SYSTEM=0\|1`   | システムプロンプトの取得を切り替える        |

### 確認する内容

- キャッシュトレースイベントは JSONL 形式で、`session:loaded`、`prompt:before`、`stream:context`、`session:after` などの段階別スナップショットが含まれます。
- ターンごとのキャッシュトークンへの影響は、通常の使用状況画面で確認できます。`cacheRead` と `cacheWrite` は、`/usage tokens`、`/status`、セッション使用量の概要、カスタム `messages.usageTemplate` レイアウトに表示されます。
- Anthropic では、キャッシュが有効な場合、`cacheRead` と `cacheWrite` の両方が存在することを期待します。
- OpenAI では、キャッシュヒット時に `cacheRead` が存在することを期待します。`cacheWrite` は、それを含む Responses API ペイロードでのみ設定されます（前述の [OpenAI](#openai-direct-api) を参照）。
- OpenAI は、`x-request-id`、`openai-processing-ms`、`x-ratelimit-*` などのトレースヘッダーやレート制限ヘッダーも返します。これらはリクエストのトレースに使用しますが、キャッシュヒットの集計にはヘッダーではなく、引き続き使用量ペイロードを使用する必要があります。

## クイックトラブルシューティング

- **ほとんどのターンで `cacheWrite` が高い**：変動するシステムプロンプト入力がないか確認し、モデルまたはプロバイダーがキャッシュ設定をサポートしていることを確認します。
- **Anthropic で `cacheWrite` が高い**：多くの場合、リクエストごとに変化するコンテンツにキャッシュのブレークポイントが設定されていることを意味します。
- **OpenAI の `cacheRead` が低い**：安定したプレフィックスが先頭にあり、繰り返されるプレフィックスが少なくとも 1024 トークンあり、キャッシュを共有すべきターンで同じ `prompt_cache_key` が再利用されていることを確認します。
- **`cacheRetention` の効果がない**：モデルキーが `agents.defaults.models["provider/model"]` と一致することを確認します。
- **キャッシュ設定を含む Bedrock Nova リクエスト**：想定どおりです。これらは実行時にキャッシュを保持しない設定として解決されます。

関連ドキュメント：

- [Anthropic](/ja-JP/providers/anthropic)
- [トークン使用量とコスト](/ja-JP/reference/token-use)
- [セッションのプルーニング](/ja-JP/concepts/session-pruning)
- [Gateway 設定リファレンス](/ja-JP/gateway/configuration-reference)

## 関連項目

- [トークン使用量とコスト](/ja-JP/reference/token-use)
- [API の使用量とコスト](/ja-JP/reference/api-usage-costs)
