---
read_when:
    - thinking、fast-mode、verbose ディレクティブの解析またはデフォルトの調整
summary: /think、/fast、/verbose、/trace、および推論の可視性に関するディレクティブ構文
title: 思考レベル
x-i18n:
    generated_at: "2026-07-26T09:24:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80968ce58f642090ba0f807874e43eea1206cd31d919414c690b7537dc523658
    source_path: tools/thinking.md
    workflow: 16
---

## 機能

- 受信した本文内のインラインディレクティブ: `/t <level>`、`/think:<level>`、または `/thinking <level>`。
- レベル（エイリアス）: `off | minimal | low | medium | high | xhigh | adaptive | max | ultra`。Anthropic の従来の「think」<「think hard」<「think harder」<「ultrathink」というマジックワードの段階にほぼ対応します:
  - minimal ~ 「think」
  - low ~ 「think hard」
  - medium ~ 「think harder」
  - high ~ 「ultrathink」（最大予算）
  - xhigh ~ 「ultrathink+」（GPT-5.2+ および Codex モデル、さらに Anthropic Claude Opus 4.7+ の effort）
  - adaptive → プロバイダー管理の適応的思考（Anthropic/Bedrock 上の Claude 4.6、Anthropic Claude Opus 4.7+、Google Gemini の動的思考でサポート）
  - max → プロバイダーの最大推論（Anthropic Claude Opus 4.7+。Ollama では、これをネイティブの最高 `think` effort にマッピング）
  - ultra → 選択したモデル／ランタイムがサポートする場合、プロバイダーの最大推論に加えてプロアクティブなサブエージェントのオーケストレーション
  - `x-high`、`x_high`、`extra-high`、`extra high`、および `extra_high` は `xhigh` にマッピングされます。
  - `highest` は `high` にマッピングされます。
- プロバイダーに関する注記:
  - 思考メニューと選択項目は、プロバイダープロファイルによって決まります。プロバイダー Plugin は、バイナリの `on` などのラベルを含め、選択したモデルで利用可能な正確なレベルセットを宣言します。
  - `adaptive`、`xhigh`、`max`、および `ultra` は、それらをサポートするプロバイダー／モデル／ランタイムのプロファイルでのみ提示されます。サポートされていないレベルを型付きディレクティブで指定すると、そのモデルの有効な選択肢とともに拒否されます。
  - 保存済みのサポートされていないレベルは、プロバイダープロファイルの順位に従って再マッピングされます。非適応型モデルでは `adaptive` は `medium` にフォールバックし、`xhigh` と `max` は、選択したモデルでサポートされる最大の非オフレベルにフォールバックします。
  - Anthropic Claude 4.6 モデルでは、思考レベルが明示的に設定されていない場合、デフォルトで `adaptive` が使用されます。
  - Anthropic Claude Opus 4.8 と Opus 4.7 では、思考レベルを明示的に設定しない限り、思考はオフのままです。適応的思考を有効にすると、Opus 4.8 のプロバイダー所有のデフォルト effort は `high` になります。
  - Anthropic Claude Opus 4.7+ では、`/think xhigh` は適応的思考と `output_config.effort: "xhigh"` の組み合わせにマッピングされます。これは、`/think` が思考ディレクティブで、`xhigh` が Opus の effort 設定であるためです。
  - Anthropic Claude Opus 4.7+ では `/think max` も公開され、同じプロバイダー所有の最大 effort パスにマッピングされます。
  - 直接接続の DeepSeek V4 モデルでは `/think xhigh|max` が公開されます。どちらも DeepSeek の `reasoning_effort: "max"` にマッピングされ、それより低い非オフレベルは `high` にマッピングされます。
  - OpenRouter 経由の DeepSeek V4 モデルでは `/think xhigh` が公開され、DeepSeek ネイティブのトップレベル `reasoning_effort` の代わりに、OpenRouter がサポートする `reasoning.effort` 値を送信します。それより低い非オフレベルは `high` にマッピングされ、保存済みの `max` オーバーライドは `xhigh` にフォールバックします。
  - 思考対応の Ollama モデルでは `/think low|medium|high|max` が公開されます。Ollama のネイティブ API は `low`、`medium`、および `high` の effort 文字列を受け付けるため、`max` はネイティブの `think: "high"` にマッピングされます。
  - OpenAI GPT モデルでは、モデル固有の Responses API の effort サポートを通じて `/think` がマッピングされます。`/think off` は、対象モデルがサポートする場合にのみ `reasoning.effort: "none"` を送信します。それ以外の場合、OpenClaw はサポートされていない値を送信せず、無効化された推論ペイロードを省略します。
  - GPT-5.6 Sol と Terra では、Codex ランタイムを通じてネイティブの `/think ultra` が公開されます。GPT-5.6 Luna では、Codex カタログが Ultra を提示しないため、`max` までのレベルが公開されます。
  - 組み込みの OpenClaw ランタイムでは、GPT-5.6 Sol、Terra、および Luna に対して論理的な `/think ultra` が公開されます。プロバイダーの最大 effort を送信し、実行スコープのプロアクティブなサブエージェントのオーケストレーションに関する指示を追加します。
  - カスタムの OpenAI 互換カタログエントリでは、`models.providers.<provider>.models[].compat.supportedReasoningEfforts` に `"xhigh"` を含めることで、`/think xhigh` を有効にできます。これは、送信される OpenAI の推論 effort ペイロードをマッピングするものと同じ互換性メタデータを使用するため、メニュー、セッション検証、エージェント CLI、および `llm-task` の動作がトランスポートと一致します。
  - 古い設定に残っている OpenRouter Hunter Alpha の参照では、廃止されたそのルートが推論フィールドを介して最終回答テキストを返す可能性があったため、プロキシ推論の注入をスキップします。
  - Google Gemini では、`/think adaptive` は Gemini のプロバイダー所有の動的思考にマッピングされます。Gemini 3 のリクエストでは固定の `thinkingLevel` を省略し、Gemini 2.5 のリクエストでは `thinkingBudget: -1` を送信します。固定レベルは引き続き、そのモデルファミリーで最も近い Gemini の `thinkingLevel` または予算にマッピングされます。
  - Anthropic 互換ストリーミングパス上の MiniMax M2.x（`minimax/MiniMax-M2*`）では、モデルパラメーターまたはリクエストパラメーターで思考を明示的に設定しない限り、デフォルトで `thinking: { type: "disabled" }` が使用されます。これにより、M2.x の非ネイティブ Anthropic ストリーム形式から `reasoning_content` の差分が漏れることを防ぎます。MiniMax-M3（および M3.x）は対象外です。M3 は適切な Anthropic 思考ブロックを出力し、思考が無効な場合は空のコンテンツを返すため、OpenClaw は M3 でプロバイダーの省略時／適応的思考パスを維持します。
  - Z.AI（`zai/*`）は、ほとんどの GLM モデルでバイナリ（`on`/`off`）です。GLM-5.2 は例外です。`/think off|low|high|max` を公開し、`low` と `high` を Z.AI の `reasoning_effort: "high"` にマッピングし、`max` を `reasoning_effort: "max"` にマッピングします。
  - Moonshot API Kimi K3（`moonshot/kimi-k3`）は常に `max` で思考し、`reasoning_effort: "max"` を送信し、K2 の `thinking` フィールドと固定サンプリングのオーバーライドを省略し、K3 がサポートするツール選択を維持します。Kimi Code K3（`kimi/k3` および `kimi/k3[1m]`）では `/think off|max` が公開されます。オフでは `thinking.type: "disabled"` を送信し、max では最大 effort を伴う適応的思考を送信します。現在の Kimi Code の参照には `kimi/kimi-for-coding` と `kimi/kimi-for-coding-highspeed` も含まれます。Kimi K2.7 Code（`moonshot/kimi-k2.7-code` および `moonshot/kimi-k2.7-code-highspeed`）は常に思考し、`on` のみを公開し、送信時には `thinking` と `reasoning_effort` の両方を省略します。その他の `moonshot/*` モデルでは、`/think off` を `thinking: { type: "disabled" }` にマッピングし、`off` 以外のレベルを `thinking: { type: "enabled" }` にマッピングします。K2 の思考が有効な場合、Moonshot が受け付ける `tool_choice` は `auto|none` のみです。OpenClaw は互換性のない値を `auto` に正規化します。

## 解決順序

1. メッセージ内のインラインディレクティブ（そのメッセージにのみ適用）。
2. セッションオーバーライド（ディレクティブのみのメッセージを送信して設定）。
3. エージェント単位のデフォルト（設定内の `agents.entries.*.thinkingDefault`）。
4. グローバルデフォルト（設定内の `agents.defaults.thinkingDefault`）。
5. フォールバック: 利用可能な場合はプロバイダーが宣言したデフォルト。それ以外の場合、推論対応モデルでは `medium`、またはそのモデルでサポートされる最も近い非 `off` レベルに解決され、推論非対応モデルでは `off` のままになります。

## セッションのデフォルト設定

- ディレクティブ**のみ**のメッセージ（空白文字は使用可能）を送信します。例: `/think:medium` または `/t high`。
- これは現在のセッションに保持されます（デフォルトでは送信者ごと）。`/think default` を使用すると、セッションオーバーライドをクリアして設定済み／プロバイダーのデフォルトを継承できます。エイリアスには `inherit`、`clear`、`reset`、および `unpin` があります。
- `/think off` は明示的なオフのオーバーライドを保存します。セッションオーバーライドを変更またはクリアするまで、思考は無効になります。
- 確認応答（`Thinking level set to high.` / `Thinking disabled.`）が送信されます。レベルが無効な場合（例: `/thinking big`）、コマンドはヒントとともに拒否され、セッション状態は変更されません。
- 引数なしで `/think`（または `/think:`）を送信すると、現在の思考レベルを確認できます。

## エージェントによる適用

- **組み込み OpenClaw**: 解決されたレベルは、プロセス内の OpenClaw エージェントランタイムに渡されます。
- **Claude CLI バックエンド**: `claude-cli` を使用する場合、具体的な非オフレベルは `--effort` として Claude Code に渡されます。`adaptive` は設定済みの effort フラグを削除し、実効的な effort を Claude Code の環境、設定、およびモデルのデフォルトに委ねます。[CLI バックエンド](/ja-JP/gateway/cli-backends)を参照してください。

## 高速モード（/fast）

- レベル: `auto|on|off|default`。
- ディレクティブのみのメッセージは、セッションの高速モードオーバーライドを切り替え、`Fast mode set to auto.`、`Fast mode enabled.`、または `Fast mode disabled.` を応答します。`/fast default` を使用すると、セッションオーバーライドをクリアして設定済みのデフォルトを継承できます。エイリアスには `inherit`、`clear`、`reset`、および `unpin` があります。
- モードを指定せずに `/fast`（または `/fast status`）を送信すると、現在の実効的な高速モードの状態を確認できます。
- OpenClaw は、次の順序で高速モードを解決します:
  1. インライン／ディレクティブのみの `/fast auto|on|off` オーバーライド（`/fast default` でこのレイヤーをクリア）
  2. セッションオーバーライド
  3. エージェント単位のデフォルト（`agents.entries.*.fastModeDefault`）
  4. モデル単位の設定: `agents.defaults.models["<provider>/<model>"].params.fastMode`
  5. フォールバック: `off`
- `auto` は、セッション／設定のモードを auto のまま維持しますが、新しいモデル呼び出しごとに独立して解決します。auto のカットオフより前に開始した呼び出しでは高速モードが有効になり、それ以降に開始した再試行、フォールバック、ツール結果、または継続の呼び出しでは高速モードが無効になります。カットオフのデフォルトは 60 秒です。変更するには、アクティブなモデルで `agents.defaults.models["<provider>/<model>"].params.fastAutoOnSeconds` を設定します。
- `openai/*` では、高速モードは、サポートされる Responses リクエストで `service_tier=priority` を送信することにより、OpenAI の優先処理にマッピングされます。
- Codex ベースの `openai/*` / `openai-codex/*` モデルでは、高速モードは Codex Responses に同じ `service_tier=priority` フラグを送信します。ネイティブ Codex app-server のターンは、`turn/start` またはスレッドの開始／再開時にのみティアを受け取るため、`auto` で実行中の app-server ターンのティアを変更することはできません。これは OpenClaw が次に開始するモデルターンに適用されます。
- OAuth 認証済みで `api.anthropic.com` に送信されるトラフィックを含む、直接公開の `anthropic/*` リクエストでは、高速モードは Anthropic のサービスティアにマッピングされます。`/fast on` は `service_tier=auto` を設定し、`/fast off` は `service_tier=standard_only` を設定します。
- Anthropic 互換パス上の `minimax/*` では、`/fast on`（または `params.fastMode: true`）が `MiniMax-M2.7` を `MiniMax-M2.7-highspeed` に書き換えます。
- Anthropic の `serviceTier` / `service_tier` モデルパラメーターが両方設定されている場合、それらが高速モードのデフォルトを上書きします。OpenClaw は引き続き、Anthropic 以外のプロキシベース URL に対する Anthropic サービスティアの注入をスキップします。
- `/status` は、高速モードが有効な場合は `Fast` を、設定されたモードが auto の場合は `Fast:auto` を表示します。

## 詳細出力ディレクティブ（/verbose または /v）

- レベル: `on`（最小） | `full` | `off`（デフォルト）。
- ディレクティブのみのメッセージは、セッションの詳細出力を切り替え、`Verbose logging enabled.` / `Verbose logging disabled.` と応答します。無効なレベルの場合は、状態を変更せずにヒントを返します。
- `/verbose off` は明示的なセッションオーバーライドを保存します。Sessions UI で `inherit` を選択して解除できます。
- 認可された外部チャネルの送信者は、セッションの詳細出力オーバーライドを永続化できます。内部の Gateway/Web チャットクライアントで永続化するには、`operator.admin` が必要です。
- インラインディレクティブは、そのメッセージだけに影響します。それ以外の場合は、セッションまたはグローバルのデフォルトが適用されます。
- 現在の詳細出力レベルを確認するには、引数なしで `/verbose`（または `/verbose:`）を送信します。
- 詳細出力が有効な場合、構造化されたツール結果を生成するエージェントは、各ツール呼び出しを個別のメタデータのみのメッセージとして送り返し、利用可能な場合は先頭に `<emoji> <tool-name>: <arg>` を付けます。これらのツール概要は、各ツールの開始直後に個別の吹き出しとして送信され、ストリーミング差分としては送信されません。
- ツール失敗の概要は通常モードでも表示されますが、生のエラー詳細の接尾辞は、詳細出力が `full` でない限り非表示になります。
- 詳細出力が `full` の場合、ツール出力も完了後に転送されます（個別の吹き出しとして、安全な長さに切り詰められます）。実行中に `/verbose on|full|off` を切り替えると、それ以降のツールの吹き出しには新しい設定が適用されます。
- `agents.defaults.toolProgressDetail` は、`/verbose` ツール概要および進捗ドラフトのツール行の形式を制御します。`🛠️ Exec: checking JS syntax` のような簡潔で人が読みやすいラベルには、`"explain"`（デフォルト）を使用します。デバッグ用に生のコマンドや詳細も追加する場合は、`"raw"` を使用します。エージェントごとの `agents.entries.*.toolProgressDetail` はデフォルトを上書きします。
  - `explain`: `🛠️ Exec: check JS syntax for /tmp/app.js`
  - `raw`: `🛠️ Exec: check JS syntax for /tmp/app.js, node --check /tmp/app.js`

## Plugin トレースディレクティブ（/trace）

- レベル: `on` | `off`（デフォルト）。
- ディレクティブのみのメッセージは、セッションの Plugin トレース出力を切り替え、`Plugin trace enabled.` / `Plugin trace disabled.` と応答します。
- インラインディレクティブは、そのメッセージだけに影響します。それ以外の場合は、セッションまたはグローバルのデフォルトが適用されます。
- 現在のトレースレベルを確認するには、引数なしで `/trace`（または `/trace:`）を送信します。
- `/trace` は `/verbose` より範囲が限定されており、Active Memory のデバッグ概要など、Plugin が所有するトレース行やデバッグ行だけを公開します。
- トレース行は、`/status` 内、および通常のアシスタント応答後に続く診断メッセージとして表示されることがあります。

## 推論の表示（/reasoning）

- レベル: `on|off|stream`。
- ディレクティブのみのメッセージは、応答に思考ブロックを表示するかどうかを切り替えます。
- 有効にすると、推論は先頭に `Thinking` が付いた**個別のメッセージ**として送信されます。
- `stream`: アクティブなチャネルが推論プレビューをサポートしている場合、応答の生成中に推論をストリーミングし、その後、推論を含まない最終回答を送信します。
- エイリアス: `/reason`。
- 現在の推論レベルを確認するには、引数なしで `/reasoning`（または `/reasoning:`）を送信します。
- 解決順序: インラインディレクティブ、セッションオーバーライド、エージェントごとのデフォルト（`agents.entries.*.reasoningDefault`）、グローバルデフォルト（`agents.defaults.reasoningDefault`）、フォールバック（`off`）の順です。

不正なローカルモデルの推論タグは、保守的に処理されます。閉じられた `<think>...</think>` ブロックは通常の応答では非表示のままとなり、すでに表示されているテキストの後にある閉じられていない推論も非表示になります。応答全体が閉じられていない単一の開始タグで囲まれており、そのままでは空のテキストとして配信される場合、OpenClaw は不正な開始タグを削除し、残りのテキストを配信します。

## 関連項目

- 昇格モードのドキュメントについては、[昇格モード](/ja-JP/tools/elevated)を参照してください。

## Heartbeat

- Heartbeat プローブの本文は、設定された Heartbeat プロンプトです（デフォルト: `Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`）。Heartbeat メッセージ内のインラインディレクティブは通常どおり適用されますが、Heartbeat からセッションのデフォルトを変更することは避けてください。
- Heartbeat の配信は、デフォルトでは最終ペイロードのみです。個別の `Thinking` メッセージも送信するには（利用可能な場合）、`agents.defaults.heartbeat.includeReasoning: true` またはエージェントごとの `agents.entries.*.heartbeat.includeReasoning: true` を設定します。

## Web チャット UI

- Web チャットの思考セレクターは、ページの読み込み時に、受信セッションストアまたは設定に保存されているセッションレベルを反映します。
- 別のレベルを選択すると、`sessions.patch` を介してセッションオーバーライドが即座に書き込まれます。次の送信を待つことはなく、1 回限りの `thinkingOnce` オーバーライドでもありません。
- モデル、推論、または速度のピッカー変更がまだ適用中に送信すると、保留中のすべてのピッカーパッチが完了するまで待機します。変更に失敗した場合、メッセージは確認できるよう未送信のままになります。
- 最初のオプションは常にオーバーライドを解除する選択肢です。継承された思考が無効な場合の `Inherited: Off` を含め、`Inherited: <resolved level>` と表示されます。
- 明示的なピッカーの選択肢では、プロバイダーラベルがある場合はそれを維持しながら、対応するレベルラベルを直接使用します（たとえば、プロバイダーラベル付きの `max` オプションには `Maximum`）。
- ピッカーは、Gateway のセッション行またはデフォルトから返される `thinkingLevels` を使用し、`thinkingOptions` は従来のラベル一覧として維持されます。ブラウザー UI は独自のプロバイダー正規表現一覧を保持しません。モデル固有のレベルセットは Plugin が所有します。
- `/think:<level>` も引き続き機能し、同じ保存済みセッションレベルを更新するため、チャットディレクティブとピッカーは同期された状態を維持します。

## プロバイダープロファイル

- プロバイダー Plugin は、`resolveThinkingProfile(ctx)` を公開して、モデルがサポートするレベルとデフォルトを定義できます。
- Claude モデルをプロキシするプロバイダー Plugin は、直接の Anthropic カタログとプロキシカタログの整合性を維持するため、`openclaw/plugin-sdk/provider-model-shared` の `resolveClaudeThinkingProfile(modelId)` を再利用する必要があります。
- 各プロファイルレベルには、保存される正規の `id`（`off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`adaptive`、`max`、または `ultra`）があり、表示用の `label` を含めることもできます。バイナリプロバイダーは `{ id: "low", label: "on" }` を使用します。
- プロファイルフックは、利用可能な場合、`reasoning`、`compat.thinkingFormat`、`compat.supportedReasoningEfforts` を含む、統合されたカタログ情報を受け取ります。これらの情報を使用し、設定されたリクエスト契約が対応するペイロードをサポートする場合に限って、バイナリまたはカスタムプロファイルを公開してください。
- 明示的な思考オーバーライドを検証する必要があるツール Plugin は、`api.runtime.agent.resolveThinkingPolicy({ provider, model, agentRuntime })` と `api.runtime.agent.normalizeThinkingLevel(...)` を使用する必要があります。独自のプロバイダーまたはモデルレベル一覧を保持してはいけません。常に埋め込みで実行される場合など、ツールが実行パスを所有しているときは、`agentRuntime` を渡します。
- 設定済みのカスタムモデルメタデータにアクセスできるツール Plugin は、`catalog` を `resolveThinkingPolicy` に渡すことで、`compat.supportedReasoningEfforts` のオプトインを Plugin 側の検証に反映できます。
- 公開済みの従来のフック（`supportsXHighThinking`、`isBinaryThinking`、`resolveDefaultThinkingLevel`）は互換性アダプターとして残りますが、新しいカスタムレベルセットには `resolveThinkingProfile` を使用する必要があります。
- Gateway の行およびデフォルトは、`thinkingLevels`、`thinkingOptions`、`thinkingDefault` を公開するため、ACP/チャットクライアントは、ランタイム検証で使用されるものと同じプロファイル ID およびラベルをレンダリングできます。
