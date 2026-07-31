---
read_when:
    - エージェント用の GitHub Copilot SDK ハーネスを使用する場合
    - '`copilot` ランタイムの設定例が必要です'
    - サブスクリプション版 Copilot（github / openclaw / copilot）にエージェントを接続し、Copilot CLI 経由で実行したい場合。
summary: 外部の GitHub Copilot SDK ハーネスを介して OpenClaw の組み込みエージェントターンを実行する
title: Copilot SDK ハーネス
x-i18n:
    generated_at: "2026-07-26T10:21:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4b67959c2c72bda97a81d0b45bc32ba363373064ec40c54f9709705dd15dd9fc
    source_path: plugins/copilot.md
    workflow: 16
---

外部 `@openclaw/copilot` Plugin は、OpenClaw の組み込みハーネスではなく GitHub Copilot CLI（`@github/copilot-sdk`）を介して、埋め込みサブスクリプション Copilot
エージェントターンを実行します。Copilot CLI セッションは、低レベルの
エージェントループ（ネイティブツールの実行、ネイティブ Compaction（`infiniteSessions`）、および
`copilotHome` 配下の CLI 管理スレッド状態）を所有します。OpenClaw は引き続き、チャット
チャネル、セッションファイル、モデル選択、動的ツール（ブリッジ経由）、承認、
メディア配信、表示用トランスクリプトミラー、`/btw` の補足質問（
[補足質問（`/btw`）](#side-questions-btw)を参照）、および `openclaw doctor` を所有します。

モデル／プロバイダー／ランタイムのより広範な分担については、
[エージェントランタイム](/ja-JP/concepts/agent-runtimes)から確認してください。

## 要件

- `@openclaw/copilot` Plugin がインストールされた OpenClaw。
- 設定で `plugins.allow` を使用している場合は、`copilot`（Plugin が
  宣言するマニフェスト ID）を含めます。npm パッケージ名
  `@openclaw/copilot` の許可リストエントリは一致しないため、
  `agentRuntime.id: "copilot"` が設定されていても Plugin はブロックされたままになります。
- Copilot CLI を実行できる GitHub Copilot サブスクリプション、または
  ヘッドレス実行や Cron 実行用の `gitHubToken` 環境変数／認証プロファイルエントリ。
- 書き込み可能な `copilotHome` ディレクトリ。OpenClaw が
  エージェントディレクトリを提供する場合はデフォルトで `<agentDir>/copilot`、それ以外は
  `~/.openclaw/agents/<agentId>/copilot` です。

`openclaw doctor` は、セッション状態の所有権と将来の設定移行のために、Plugin の
[doctor コントラクト](#doctor)を実行します。Copilot CLI 環境の検査は行いません。

## インストール

Copilot ランタイムは外部 Plugin として提供されるため、コアの `openclaw`
パッケージには `@github/copilot-sdk` や、そのプラットフォーム固有の
`@github/copilot-<platform>-<arch>` CLI バイナリ（合わせて約 260 MB）は含まれません。
このランタイムを選択するエージェントにのみインストールしてください。

```bash
openclaw plugins install @openclaw/copilot
```

セットアップウィザードは、初めて `github-copilot/*` モデルを選択し、**かつ**
設定が `agentRuntime: { id: "copilot" }` を介してそのモデル（またはそのプロバイダー）を Copilot ランタイムに
ルーティングしている場合に、Plugin を自動的にインストールします。
[クイックスタート](#quickstart)を参照してください。このオプトインがなければ、OpenClaw は組み込みの
GitHub Copilot プロバイダーを使用し、この Plugin をインストールしません。

ランタイムは次の順序で SDK を解決します。

1. インストール済みの `@openclaw/copilot`
   パッケージに含まれる `import("@github/copilot-sdk")`。
2. フォールバックディレクトリ `~/.openclaw/npm-runtime/copilot/`（従来のオンデマンド
   インストール先）。

SDK が見つからない場合は、コード `COPILOT_SDK_MISSING` と上記の再インストールコマンドを含む
単一のエラーが表示されます。

## クイックスタート

1 つのモデル（または 1 つのプロバイダー）をハーネスに固定します。

```json5
{
  agents: {
    defaults: {
      model: "github-copilot/auto",
      models: {
        "github-copilot/auto": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
}
```

単一のモデルエントリに `agentRuntime.id` を設定すると、そのモデルのみがハーネスを経由します。
プロバイダーに設定すると、そのプロバイダー配下のすべてのモデルが経由します。

`github-copilot/auto` は汎用的な出発点です。名前付き Copilot モデルは
アカウントおよび組織のポリシーに依存します。固定する前に、認証済みの
Copilot CLI が実際にそのモデルを公開していることを確認してください。

## サポート対象プロバイダー

ハーネスは、`extensions/github-copilot` が所有する標準の `github-copilot`
プロバイダーに加えて、モデルに空でない `baseUrl` があり、
`api` が次のいずれかの形式である場合、カスタム `models.providers` エントリをサポートします。

- `anthropic-messages`
- `azure-openai-responses`
- `ollama`（OpenAI 互換の補完）
- `openai-completions`
- `openai-responses`

ネイティブプロバイダー ID（`openai`、`anthropic`、`google`、`ollama`）は、引き続き
それぞれのネイティブランタイムが所有します。代わりに Copilot BYOK を介してエンドポイントを
ルーティングするには、別個のカスタムプロバイダー ID を使用してください。

Copilot BYOK エンドポイントは、公開 HTTPS URL でなければなりません。ハーネスは
試行ごとに loopback プロキシを Copilot SDK に提供し、その後、DNS ピンニングと SSRF ポリシーを
OpenClaw の所有下に維持するため、プロバイダートラフィックを OpenClaw の保護された fetch パス経由で
転送します。ローカルの Ollama、LM Studio、または LAN モデルサーバーには、
ネイティブ OpenClaw ランタイムを使用してください。

## BYOK

Copilot BYOK は、SDK のセッションレベルのカスタムプロバイダーコントラクトを使用します。OpenClaw は、
解決済みのモデルエンドポイント、API キー、ベアラートークンモード、ヘッダー、モデル
ID、およびコンテキスト／出力制限を渡します。プロバイダーのトランスポートロジックはコアではなく
SDK に残ります。

```json5
{
  agents: {
    defaults: {
      model: "custom-proxy/llama-3.1-8b",
      models: {
        "custom-proxy/llama-3.1-8b": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      "custom-proxy": {
        baseUrl: "https://api.example.com/v1",
        apiKey: "${CUSTOM_PROXY_API_KEY}",
        api: "openai-responses",
        authHeader: true,
        models: [{ id: "llama-3.1-8b", name: "Llama 3.1 8B" }],
      },
    },
  },
}
```

BYOK セッションは、サブスクリプションセッション、および他の
BYOK エンドポイントや認証情報とは別々にキー設定されます。キー、ヘッダー、モデル、またはエンドポイントを
ローテーションすると、互換性のない状態を再開する代わりに、新しい Copilot SDK セッションが開始されます。

## 認証

`runCopilotAttempt` 中にエージェントごとに適用される優先順位は次のとおりです。

1. 試行入力での**明示的な `useLoggedInUser: true`** — エージェントの
   `copilotHome` 配下で Copilot CLI にログインしているユーザーを使用します。
2. 試行入力での**明示的な `gitHubToken`**（`profileId` +
   `profileVersion` が必要）。認証プロファイルの解決をバイパスする必要がある
   CLI の直接呼び出しとテスト向けです。
3. **コントラクトで解決された `resolvedApiKey` + `authProfileId`** — 本番環境の
   メインパスです。コアはハーネスを呼び出す前に、エージェントに設定された
   `github-copilot` 認証プロファイル（`src/infra/provider-usage.auth.ts:resolveProviderAuths`）を
   解決します。そのため、`github-copilot:<profile>` 認証プロファイルは、
   環境変数を使わずに、ヘッドレス、Cron、または複数プロファイルの構成で
   エンドツーエンドに機能します。
4. **環境変数へのフォールバック**。次の順序で確認されます（最初の空でない値が優先され、
   空文字列は未指定として扱われます。`extensions/github-copilot/auth.ts` で提供されている
   `github-copilot` プロバイダーの優先順位と同じです）。
   1. `OPENCLAW_GITHUB_TOKEN` — ハーネス固有のオーバーライド。システム全体の `gh` /
      Copilot CLI 設定に影響を与えず、OpenClaw ハーネス用のトークンを固定できます。
   2. `COPILOT_GITHUB_TOKEN` — 標準の Copilot SDK / CLI 環境変数。
   3. `GH_TOKEN` — 標準の `gh` CLI 環境変数。
   4. `GITHUB_TOKEN` — 汎用 GitHub トークンのフォールバック。

   合成されたプールプロファイル ID は `env:<NAME>` です。プロファイルバージョンは
   トークンの不可逆な sha256 フィンガープリントであるため、環境変数の値をローテーションすると
   クライアントプールが確実に無効化されます。

5. トークンを示す情報がない場合の**デフォルトの `useLoggedInUser`**。

各エージェントには独自の `copilotHome` が割り当てられるため、同じマシン上の
エージェント間で Copilot CLI のトークン、セッション、および設定が漏れることはありません。デフォルトは
`<agentDir>/copilot`（SDK の状態を OpenClaw の
`models.json` / `auth-profiles.json` と同じディレクトリに保存しないようにします）、または
エージェントディレクトリが指定されていない場合は `~/.openclaw/agents/<agentId>/copilot` です。
カスタムの場所（たとえば、移行用の共有マウント）を使用するには、試行入力の
`copilotHome: <path>` でオーバーライドします。

ライブハーネステストでは、直接指定するトークンとして `OPENCLAW_COPILOT_AGENT_LIVE_TOKEN` を使用します。
共有ライブテストのセットアップでは、実際の認証プロファイルを隔離されたテストホームにステージングした後、
`COPILOT_GITHUB_TOKEN`、`GH_TOKEN`、および `GITHUB_TOKEN` を消去します。そのため、
専用変数を介して渡される `gh auth token` の値により、無関係なスイートに漏らすことなく、
誤ったスキップを回避できます。

## 設定項目

ハーネスは、試行ごとの入力（`runCopilotAttempt({...})`）に加えて、
`extensions/copilot/src/` 内の少数の環境変数デフォルトから設定を読み取ります。

| フィールド                    | 用途                                                                                                                                                                                                                                                                                         |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `copilotHome`            | エージェントごとの CLI 状態ディレクトリ（デフォルトは前述のとおり）。                                                                                                                                                                                                                                                 |
| `model`                  | 文字列または `{ provider, id, api?, baseUrl?, headers?, authHeader? }`。省略するとエージェントの通常のモデル選択を使用し、ハーネスが解決済みのプロバイダーがサポート対象であることを確認します。                                                                                                                   |
| `reasoningEffort`        | `"low" \| "medium" \| "high" \| "xhigh"`。`auto-reply/thinking.ts` 内の OpenClaw の `ThinkLevel` / `ReasoningLevel` 解決からマッピングされます。                                                                                                                                                          |
| `infiniteSessionConfig`  | `harness.compact` によって制御される SDK の `infiniteSessions` ブロック用の任意のオーバーライド。そのままでも安全です。                                                                                                                                                                                        |
| `hooksConfig`            | ツール／MCP、ユーザープロンプト、セッション、およびエラーのコールバック用の、任意のネイティブ Copilot SDK `SessionHooks` 設定。OpenClaw の移植可能なライフサイクルフックとは別です。                                                                                                                                   |
| `permissionPolicy`       | SDK 組み込みツール種別（`shell`、`write`、`read`、`url`、`mcp`、`memory`、`hook`）用の SDK の `onPermissionRequest` ハンドラーに対する任意のオーバーライド。安全策としてデフォルトは `rejectAllPolicy` です。実際には決して実行されない理由については、[権限と ask_user](#permissions-and-ask_user)を参照してください。 |
| `enableSessionTelemetry` | 任意の SDK セッションテレメトリフラグ。                                                                                                                                                                                                                                                            |

OpenClaw の Plugin フックには、Copilot 固有の試行設定は必要ありません。ハーネスは
標準のハーネスヘルパーを介して `before_prompt_build`、`llm_input`、`llm_output`、および `agent_end` を
実行します。SDK の Compaction が成功した場合は、`before_compaction` と
`after_compaction` も実行されます。ブリッジされた OpenClaw ツールは
`before_tool_call` を実行して `after_tool_call` を報告します。`hooksConfig` は、
移植可能な同等機能がないネイティブ SDK 専用コールバック用として残ります。

OpenClaw のその他の部分が、これらのフィールドを認識する必要はありません。他の Plugin、
チャネル、およびコアコードから見えるのは、標準の `AgentHarnessAttemptParams` /
`AgentHarnessAttemptResult` 形式だけです。

## Compaction

`harness.compact` の実行時、Copilot SDK ハーネスは次の処理を行います。

1. 保留中の作業を続行せずに、追跡対象の SDK セッションを再開します。
2. SDK のセッションスコープの履歴 Compaction RPC を呼び出します。
3. ワークスペース配下に互換性マーカーファイルを書き込まず、SDK の Compaction 結果を
   返します。

OpenClaw 側のトランスクリプトミラー（後述）は Compaction 後の
メッセージも引き続き受信するため、ユーザー向けチャット履歴の一貫性が保たれます。

## トランスクリプトのミラーリング

`runCopilotAttempt` は、各ターンのミラー可能なメッセージを
`extensions/copilot/src/dual-write-transcripts.ts` を介して OpenClaw の監査トランスクリプトに
二重書き込みします。ミラーのスコープはセッション
（`copilot:${sessionId}`）ごとに設定され、メッセージ
（`${role}:${sha256_16(role,content)}`）ごとにキーが付与されるため、再送信された以前のターンのエントリは
重複せず、ディスク上の既存キーと衝突します。

ミラーは 2 層の障害封じ込めでラップされているため、トランスクリプトの書き込み
失敗によって試行が失敗することはありません。内部のベストエフォートラッパーに加え、
試行レベルで多層防御の `.catch(...)` が適用されます。失敗はログに記録され、
表面化しません。

## 補助的な質問（`/btw`）

`/btw` は、このハーネスではネイティブでは**ありません**。`createCopilotAgentHarness()` は
意図的に `harness.runSideQuestion` を未定義のままにします
（`extensions/copilot/harness.test.ts`、`describe("runSideQuestion")` でアサート）。
そのため、OpenClaw の `/btw` ディスパッチャー（`src/agents/btw.ts`）は、
Codex 以外のすべてのランタイムで使用するものと同じパスにフォールスルーします。
設定済みのモデルプロバイダーが短い補助質問プロンプトで直接呼び出され、
`streamSimple` を介してストリーミングで返されます
（CLI セッションも追加のプールスロットも使用しません）。

これにより、Copilot CLI セッションはエージェントのメインターンループ用に確保され、
`/btw` の動作は Codex 以外の他のランタイムと同一に保たれます。

## Doctor

`extensions/copilot/doctor-contract-api.ts` は
`src/plugins/doctor-contract-registry.ts` によって自動ロードされます。次のものを提供します。

- 空の `legacyConfigRules`（廃止済みフィールドはまだありません）。
- 何もしない `normalizeCompatibilityConfig`（将来廃止されるフィールドのために、
  ツリー内の安定した配置場所として維持）。
- 1 件の `sessionRouteStateOwners` エントリ：プロバイダー `github-copilot`、ランタイム
  `copilot`、CLI セッションキー `copilot`、認証プロファイルのプレフィックス `github-copilot:`。

## 制限事項

- ハーネスは `github-copilot` と、所有者のないカスタム BYOK プロバイダー ID を対象とします。
  マニフェスト所有のネイティブプロバイダー ID は、
  `agentRuntime.id` が `copilot` に強制されている場合でも、
  所有元のランタイムに残ります。
- TUI サーフェスはありません。ピアサーフェスを持たないランタイムでは、
  PI の TUI が引き続きフォールバックになります。
- エージェントが `copilot` に切り替えても、PI セッション状態は移行されません。
  選択は試行ごとに行われ、既存の PI セッションは引き続き有効です。
- `ask_user` は、プロバイダー中立の Gateway 質問ランタイムを使用します。Control
  UI には他の OpenClaw 質問と同じ質問カードが表示され、対応する
  チャンネルでは選択肢ボタンがレンダリングされます。また、次にキューに入ったプレーンテキストメッセージが、
  SDK リクエストから制御が戻る前にその Gateway レコードを解決します。

## 権限と ask_user

ブリッジされた OpenClaw ツールに対する権限の適用は、SDK の
`onPermissionRequest` コールバック経由ではなく、**ツールラッパー内**で行われます。PI が使用するものと同じ
`wrapToolWithBeforeToolCallHook`
（`src/agents/agent-tools.before-tool-call.ts`）が、
`createOpenClawCodingTools` によってすべてのコーディングツールに適用されます。ループ検出、信頼済み
Plugin ポリシー、ツール呼び出し前フック、および Gateway
（`plugin.approval.request`）を介した 2 フェーズの Plugin 承認はすべて、ネイティブ PI 試行とまったく同じコード
パスを通ります。

Copilot ツールブリッジによって返される各 SDK ツールには、次の指定が付けられます。

- `overridesBuiltInTool: true` — 同名の Copilot CLI 組み込みツール
  （edit、read、write、bash、...）を置き換え、すべてのツール呼び出しが
  OpenClaw に戻るようにルーティングします。
- `skipPermission: true` — ツールを呼び出す前に
  `onPermissionRequest({kind: "custom-tool"})` を発火しないよう SDK に指示します。
  ラップされた `execute()` は、すでにより高度な OpenClaw ポリシーチェックを実行します。
  SDK レベルのプロンプトでは、OpenClaw の適用を短絡させる
  （すべて許可）か、すべてのツール呼び出しをブロックする（すべて拒否）ことになり、
  いずれも PI との同等性を満たしません。

ツリー内の Codex ハーネスも同じ分離を使用します。ブリッジされた OpenClaw ツールは
ラップされ（`extensions/codex/src/app-server/dynamic-tools.ts`）、
codex-app-server 独自のネイティブ承認種別
（`item/commandExecution/requestApproval`、`item/fileChange/requestApproval`、
`item/permissions/requestApproval`）は `plugin.approval.request`
（`extensions/codex/src/app-server/approval-bridge.ts`）を経由します。Copilot SDK
における同等の仕組み、すなわち `onPermissionRequest` に到達する
`custom-tool` 以外のあらゆる種別に対してフェイルクローズする `rejectAllPolicy` は、
同じセーフティネットです。`overridesBuiltInTool: true` がすべての組み込みツールを置き換えるため、
実際には発火しません。

ラップされたツール層が PI と同等のポリシー判断を行えるように、ハーネスは
PI の試行ツールコンテキスト全体を
`createOpenClawCodingTools` に転送します。これには、アイデンティティ（`senderIsOwner`、`memberRoleIds`、
`ownerOnlyToolAllowlist`、...）、チャンネル／ルーティング（`groupId`、
`currentChannelId`、`replyToMode`、メッセージツールの切り替え）、認証
（`authProfileStore`）、実行アイデンティティ（`sandboxSessionKey`、`runId` から導出される
`sessionKey` / `runSessionKey`）、モデルコンテキスト（`modelApi`、
`modelContextWindowTokens`、`modelCompat`、`modelHasVision`）、および実行フック
（`onToolOutcome`、`onYield`）が含まれます。これらのフィールドがないと、所有者限定の許可リストは
デフォルトで暗黙に拒否し、Plugin の信頼ポリシーは正しい
スコープを解決できず、`session_status: "current"` は古いサンドボックスキーに解決されます。
ブリッジビルダーは `extensions/copilot/src/tool-bridge.ts` であり、PI の
正式な呼び出しである `src/agents/embedded-agent-runner/run/attempt.ts:1262` を反映しています。
`runAttempt` は共有の
`resolveSandboxContext` シームを介してサンドボックスコンテキストを解決し、SDK に有効な作業ディレクトリを渡し、
`sandbox` とサブエージェント生成用ワークスペースをツール
ブリッジに転送します。また、ブリッジは SDK 境界で適用可能な、制限付きのツール構築制御も
転送します。具体的には、`includeCoreTools`、ランタイムツールの
許可リスト、および `toolConstructionPlan` です。

ブリッジは PI との同等性を保つため、
`openclaw/plugin-sdk/agent-harness-tool-runtime` の共有ハーネスツールサーフェスヘルパーも使用します。
ツール検索が有効な場合、SDK にはすべての OpenClaw ツールスキーマの代わりに、
コンパクトな制御ツールと非表示のカタログ実行機能が提示されます。コードモードが
有効な場合、ヘルパーは他のエージェントハーネスで使用されるものと同じコードモード制御サーフェスとカタログ
ライフサイクルを構築します。ローカルモデル向けの軽量なデフォルト、
ランタイム互換のスキーマフィルタリング、ディレクトリのハイドレーション、およびカタログの
クリーンアップはすべて共有ヘルパーに留まり、Copilot と Codex に隣接する
ハーネス間の乖離を防ぎます。

### セッションレベルの GitHub トークン

Copilot SDK の契約では、**クライアントレベル**の GitHub トークン
（`CopilotClientOptions.gitHubToken`、CLI プロセス自体を認証）と
**セッションレベル**のトークン（`SessionConfig.gitHubToken`、そのセッションの
コンテンツ除外、モデルルーティング、クォータを決定し、
`createSession` と `resumeSession` の両方で考慮される）を区別します。ハーネスは
`resolveCopilotAuth` を介して認証を一度解決し、認証モードが `gitHubToken`
（明示的な `auth.gitHubToken`、または設定済みの `github-copilot` 認証プロファイルから
契約に基づいて解決された `resolvedApiKey`）の場合に、両方のフィールドを設定します。解決されたモードが
`useLoggedInUser` の場合、SDK がログイン中のアイデンティティから
引き続きアイデンティティを導出できるよう、セッションレベルのフィールドは省略されます。

`ask_user` は `SessionConfig.onUserInputRequest` を使用します。ブリッジは SDK の
選択肢、または選択肢のない自由入力プロンプトを Gateway の質問として登録し、固定選択式リクエストでは選択肢の
インデックスまたはラベルを受け入れ、SDK リクエストで許可されている場合は自由形式の回答を
受け入れます。OpenClaw の試行を中止すると、
Gateway レコードがキャンセルされ、空の SDK 回答が返されます。

## 関連項目

- [エージェントランタイム](/ja-JP/concepts/agent-runtimes)
- [Codex ハーネス](/ja-JP/plugins/codex-harness)
- [エージェントハーネス Plugin（SDK リファレンス）](/ja-JP/plugins/sdk-agent-harness)
