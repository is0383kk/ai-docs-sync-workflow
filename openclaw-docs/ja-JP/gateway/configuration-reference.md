---
read_when:
    - フィールド単位の正確な設定の意味やデフォルト値が必要な場合
    - チャンネル、モデル、Gateway、またはツールの設定ブロックを検証しています
summary: OpenClaw コアのキー、デフォルト、および各サブシステム専用リファレンスへのリンクに関する Gateway 設定リファレンス
title: 設定リファレンス
x-i18n:
    generated_at: "2026-07-26T09:20:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7135554fda444fd1b8c072af5768c53a165f7be2dcd12a7909fc7fd4bd864428
    source_path: gateway/configuration-reference.md
    workflow: 16
---

`~/.openclaw/openclaw.json` のフィールドレベルのリファレンス：キー、デフォルト、および各サブシステムの詳細ページへのリンク。タスク指向のセットアップガイダンスについては、[設定](/ja-JP/gateway/configuration)を参照してください。チャンネルおよび Plugin が所有するコマンドカタログと、メモリ/QMD の詳細設定は、ここではなくそれぞれのページに記載されています。

設定形式は **JSON5**（コメントおよび末尾のカンマを使用可能）です。すべてのフィールドは任意です。省略した場合、OpenClaw は安全なデフォルトを使用します。

このページよりもコード上の実態が優先されます：

- `openclaw config schema` は、バンドル済み/Plugin/チャンネルのメタデータを統合した、検証および Control UI で使用されるライブ JSON Schema を出力します。
- エージェントは、設定を編集する前に、`gateway` ツールアクション `config.schema.lookup` を呼び出して、パスでスコープされた正確なスキーマノードを1つ取得する必要があります。
- `pnpm config:docs:check` / `pnpm config:docs:gen` は、このドキュメントのベースラインハッシュを現在のスキーマサーフェスに対して検証します。

スキーマ `uiHints` には、すべてのパスについて解決済みの `advanced` ブール値も含まれます。
Control UI はこれを使用して、一般的なフィールドを先に表示し、セクションごとに高度なフィールドを折りたたみます。検索対象には引き続き両方の階層が含まれます。階層メタデータは表示専用です。
キーを追加する場合は、リーフで階層を宣言するか、最も近い祖先の宣言を継承させます。宣言済みの祖先がないパスは、デフォルトで高度な項目になります。

専用の詳細リファレンス：

- `memory.search.*`、`memory.qmd.*`、`memory.citations`、および `plugins.entries.memory-core.config.dreaming` 配下の Dreaming 設定については、[メモリ設定リファレンス](/ja-JP/reference/memory-config)。
- 現在の組み込みおよびバンドル済みコマンドカタログについては、[スラッシュコマンド](/ja-JP/tools/slash-commands)。
- チャンネル固有のコマンドサーフェスについては、それを所有するチャンネル/Plugin のページ。

---

## チャンネル

チャンネルごとの設定キーは、[設定 - チャンネル](/ja-JP/gateway/config-channels)に記載されています：Slack、Discord、Telegram、WhatsApp、Matrix、iMessage、およびその他のバンドル済みチャンネル向けの `channels.*`（認証、アクセス制御、マルチアカウント、メンションゲート）。

## エージェントのデフォルト、マルチエージェント、セッション、メッセージ

以下については、[設定 - エージェント](/ja-JP/gateway/config-agents)を参照してください：

- `agents.defaults.*`（ワークスペース、モデル、思考、Heartbeat、メモリ、メディア、Skills、サンドボックス）
- `multiAgent.*`（マルチエージェントのルーティングとバインディング）
- `session.*`（セッションのライフサイクル、Compaction、プルーニング）
- `messages.*`（メッセージ配信、TTS、Markdown レンダリング）
- `talk.*`（Talk モード）
  - `talk.consultThinkingLevel`：Control UI Talk のリアルタイム相談の背後で実行される OpenClaw エージェント全体に対する思考レベルの上書き
  - `talk.consultFastMode`：Control UI Talk のリアルタイム相談に対する1回限りの高速モード上書き
  - `talk.speechLocale`：Android、iOS、macOS での Talk 音声認識に使用する任意の BCP 47 ロケール ID
  - `talk.silenceTimeoutMs`：未設定の場合、Talk は文字起こしを送信する前のプラットフォーム既定の一時停止時間を維持します（`700 ms on macOS and Android, 900 ms on iOS`）
  - `talk.realtime.consultRouting`：`openclaw_agent_consult` をスキップする、確定済みのリアルタイム Talk 文字起こし向けの Gateway リレーフォールバック

## ツールとカスタムプロバイダー

ツールポリシー、実験的な切り替え、プロバイダーを利用するツール設定、およびカスタムプロバイダー/base URL のセットアップについては、[設定 - ツールとカスタムプロバイダー](/ja-JP/gateway/config-tools)を参照してください。

## モデル

プロバイダー定義、モデルの許可リスト、およびカスタムプロバイダーのセットアップについては、[設定 - ツールとカスタムプロバイダー](/ja-JP/gateway/config-tools#custom-providers-and-base-urls)を参照してください。
`models` ルートは、グローバルなモデルカタログの動作も管理します。

```json5
{
  models: {
    // 任意。デフォルト：true。変更時は Gateway の再起動が必要です。
    pricing: { enabled: false },
  },
}
```

- `models.mode`：プロバイダーカタログの動作（`merge` または `replace`）。
- `models.providers`：プロバイダー ID をキーとするカスタムプロバイダーマップ。
- `models.providers.*.localService`：ローカルモデルサーバー用の任意のオンデマンドプロセスマネージャー。OpenClaw は設定されたヘルスエンドポイントをプローブし、必要に応じて絶対パスの `command` を起動し、準備完了を待ってからモデルリクエストを送信します。[ローカルモデルサービス](/ja-JP/gateway/local-model-services)を参照してください。
- `models.pricing.enabled`：サイドカーとチャンネルが Gateway の準備完了パスに到達した後に開始される、バックグラウンドの料金情報ブートストラップを制御します。`false` の場合、Gateway は OpenRouter および LiteLLM の料金カタログ取得をスキップします。設定済みの `models.providers.*.models[].cost` 値は、ローカルのコスト見積もりで引き続き機能します。

## MCP

OpenClaw が管理する MCP サーバー定義は `mcp.servers` 配下にあり、組み込み OpenClaw およびその他のランタイムアダプターによって使用されます。`openclaw mcp list`、`show`、`set`、`unset` コマンドは、設定編集時に対象サーバーへ接続することなく、このブロックを管理します。

```json5
{
  mcp: {
    servers: {
      docs: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-fetch"],
      },
      remote: {
        url: "https://example.com/mcp",
        transport: "streamable-http", // streamable-http | sse
        requestTimeoutMs: 20000,
        connectionTimeoutMs: 5000,
        supportsParallelToolCalls: true,
        headers: {
          Authorization: "Bearer ${MCP_REMOTE_TOKEN}",
        },
        auth: "oauth",
        oauth: {
          scope: "docs.read",
        },
        sslVerify: true,
        clientCert: "/path/to/client.crt",
        clientKey: "/path/to/client.key",
        toolFilter: {
          include: ["search_*"],
          exclude: ["admin_*"],
        },
        // 任意の Codex app-server プロジェクション制御。
        codex: {
          agents: ["main"],
          defaultToolsApprovalMode: "approve", // auto | prompt | approve
        },
      },
    },
  },
}
```

- `mcp.servers`：設定済みの MCP ツールを公開するランタイム向けの、名前付き stdio またはリモート MCP サーバー定義。
  リモートエントリは `transport: "streamable-http"` または `transport: "sse"` を使用します。
  `type: "http"` は CLI ネイティブのエイリアスであり、`openclaw mcp set` と
  `openclaw doctor --fix` によって正規の `transport` フィールドへ正規化されます。
- `mcp.servers.<name>.enabled`：保存済みのサーバー定義を維持しつつ、組み込み OpenClaw の MCP 検出およびツールプロジェクションから除外するには、`false` を設定します。
- `mcp.servers.<name>.requestTimeoutMs`：サーバーごとの MCP リクエストタイムアウト（ミリ秒）。
- `mcp.servers.<name>.connectionTimeoutMs`：サーバーごとの接続タイムアウト（ミリ秒）。
- `mcp.servers.<name>.supportsParallelToolCalls`：MCP ツール呼び出しを並列実行するかどうかを選択できるアダプター向けの、任意の並行性ヒント。
- `mcp.servers.<name>.auth`：OAuth を必要とする HTTP MCP サーバーでは、`"oauth"` を設定します。OpenClaw の状態領域にトークンを保存するには、`openclaw mcp login <name>` を実行します。
- `mcp.servers.<name>.oauth`：任意の OAuth スコープ、リダイレクト URL、およびクライアントメタデータ URL の上書き。
- `mcp.servers.<name>.sslVerify`、`clientCert`、`clientKey`：プライベートエンドポイントおよび相互 TLS 向けの HTTP TLS 制御。
- `mcp.servers.<name>.toolFilter`：任意のサーバーごとのツール選択。`include` は検出される MCP ツールを一致する名前に限定し、`exclude` は一致する名前を非表示にします。エントリには、MCP ツールの正確な名前または単純な `*` glob を指定します。リソースまたはプロンプトを持つサーバーでは、ユーティリティツール名（`resources_list`、`resources_read`、`prompts_list`、`prompts_get`）も生成され、それらの名前にも同じフィルターが適用されます。
- `mcp.servers.<name>.codex`：任意の Codex app-server プロジェクション制御。
  このブロックは Codex app-server スレッド専用の OpenClaw メタデータであり、ACP セッション、汎用 Codex ハーネス設定、その他のランタイムアダプターには影響しません。
  空でない `codex.agents` は、サーバーを一覧にある OpenClaw エージェント ID に限定します。
  空、空白、または無効なスコープ付きエージェントリストは、グローバルになるのではなく、設定検証で拒否され、ランタイムのプロジェクションパスから除外されます。
  `codex.defaultToolsApprovalMode` は、そのサーバーに対して Codex ネイティブの
  `default_tools_approval_mode` を出力します。OpenClaw は、ネイティブの `mcp_servers` 設定を Codex に渡す前に、`codex` ブロックを削除します。このブロックを省略すると、Codex のデフォルト MCP 承認動作を使用して、すべての Codex app-server エージェントにサーバーがプロジェクションされます。
- セッションスコープのバンドル済み MCP ランタイムは、組み込みの10分間のアイドル TTL を使用します。
  1回限りの組み込み実行では、実行終了時のクリーンアップを要求します。TTL は、長時間存続するセッションおよび将来の呼び出し元に対する最終的な保護手段です。
- `mcp.*` 配下の変更は、キャッシュされたセッション MCP ランタイムを破棄することでホット適用されます。
  次回のツール検出または使用時に新しい設定から再作成されるため、削除された
  `mcp.servers` エントリはアイドル TTL を待たずに直ちに回収されます。
- ランタイム検出では、MCP ツールリストの変更通知も尊重し、そのセッションのキャッシュ済みカタログを破棄します。リソースまたはプロンプトを公開するサーバーには、リソースの一覧表示/読み取り、およびプロンプトの一覧表示/取得を行うユーティリティツールが追加されます。ツール呼び出しが繰り返し失敗すると、次の呼び出しを試行する前に、影響を受けたサーバーが短時間一時停止されます。

ランタイムの動作については、[MCP](/ja-JP/cli/mcp#openclaw-as-an-mcp-client-registry)および
[CLI バックエンド](/ja-JP/gateway/cli-backends#bundle-mcp-overlays)を参照してください。

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
      allowUploadedArchives: false,
    },
    workshop: {
      allowSymlinkTargetWrites: false,
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // または平文文字列
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`：バンドル済み Skills のみを対象とする任意の許可リスト（管理対象/ワークスペースの Skills には影響しません）。
- `load.extraDirs`：追加の共有 Skill ルート（優先度は最低）。
- `load.allowSymlinkTargets`：Skill のシンボリックリンクが設定済みのソースルート外にある場合に、そのリンクの解決先として許可される、信頼済みの実ターゲットルート。
- `workshop.allowSymlinkTargetWrites`：Skill Workshop の適用時に、すでに信頼済みのシンボリックリンクターゲットを介して書き込めるようにします（デフォルト：false）。
- `install.preferBrew`：true の場合、`brew` が利用可能であれば、他の種類のインストーラーへフォールバックする前に Homebrew インストーラーを優先します。
- `install.nodeManager`：`metadata.openclaw.install` 仕様に対する Node インストーラーの優先設定（`npm` | `pnpm` | `yarn` | `bun`）。
- `install.allowUploadedArchives`：信頼済みの `operator.admin` Gateway クライアントが、`skills.upload.*` を介してステージングされたプライベート zip アーカイブをインストールできるようにします（デフォルト：false）。これはアップロード済みアーカイブのパスのみを有効にします。通常の ClawHub インストールでは必要ありません。
- `entries.<skillKey>.enabled: false` は、バンドル済みまたはインストール済みであっても Skill を無効にします。
- `entries.<skillKey>.apiKey`：主要な環境変数を宣言する Skills 向けの簡易設定（平文文字列または SecretRef オブジェクト）。
- `limits.maxCandidatesPerRoot`、`limits.maxSkillsLoadedPerSource`、`limits.maxSkillsInPrompt`、`limits.maxSkillsPromptChars`、`limits.maxSkillFileBytes`：Skill の検出範囲と、モデルに提示される Skills プロンプトを制限します。
- Skill Workshop の自律性/承認設定（`workshop.autonomous.enabled`、`workshop.approvalPolicy`、`workshop.maxPending`、`workshop.maxSkillBytes`）については、[Skills の設定](/ja-JP/tools/skills-config)に記載されています。

---

## Plugins

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-plugin"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- `~/.openclaw/extensions` および `<workspace>/.openclaw/extensions` 配下のパッケージまたはバンドルディレクトリに加え、`plugins.load.paths` に指定されたファイルまたはディレクトリから読み込まれます。
- スタンドアロンの Plugin ファイルは `plugins.load.paths` に配置してください。自動検出される拡張機能のルートでは、ルート内のヘルパースクリプトが起動を妨げないよう、トップレベルの `.js`、`.mjs`、`.ts` ファイルは無視されます。
- 検出では、ネイティブ OpenClaw Plugin に加え、互換性のある Codex バンドルと Claude バンドル（マニフェストなしの Claude デフォルトレイアウトバンドルを含む）を受け入れます。
- **設定の変更には Gateway の再起動が必要です。**
- `allow`: オプションの許可リスト（記載された Plugin のみ読み込まれます）。`deny` が優先されます。
- `plugins.entries.<id>.apiKey`: Plugin レベルの API キー用簡易フィールド（Plugin が対応している場合）。
- `plugins.entries.<id>.env`: Plugin スコープの環境変数マップ。
- `plugins.entries.<id>.hooks.allowPromptInjection`: `false` の場合、コアは `before_prompt_build` などのプロンプトを変更するフックをブロックします。ネイティブ Plugin フックおよび対応しているバンドル提供のフックディレクトリに適用されます。
- `plugins.entries.<id>.hooks.allowConversationAccess`: `true` の場合、信頼された非バンドル Plugin は、`llm_input`、`llm_output`、`before_model_resolve`、`before_agent_reply`、`before_agent_run`、`before_agent_finalize`、`agent_end` などの型付きフックから会話の未加工コンテンツを読み取れます。
- `plugins.entries.<id>.subagent.allowModelOverride`: バックグラウンドのサブエージェント実行について、実行ごとの `provider` および `model` のオーバーライドを要求する権限を、この Plugin に明示的に付与します。
- `plugins.entries.<id>.subagent.allowedModels`: 信頼されたサブエージェントのオーバーライドに使用できる正規 `provider/model` ターゲットのオプションの許可リスト。任意のモデルを意図的に許可する場合にのみ `"*"` を使用してください。
- `plugins.entries.<id>.llm.allowModelOverride`: `api.runtime.llm.complete` のモデルオーバーライドを要求する権限を、この Plugin に明示的に付与します。
- `plugins.entries.<id>.llm.allowedModels`: 信頼された Plugin の LLM 補完オーバーライドに使用できる正規 `provider/model` ターゲットのオプションの許可リスト。任意のモデルを意図的に許可する場合にのみ `"*"` を使用してください。
- `plugins.entries.<id>.llm.allowAgentIdOverride`: デフォルト以外のエージェント ID に対して `api.runtime.llm.complete` を実行する権限を、この Plugin に明示的に付与します。
- `plugins.entries.<id>.config`: Plugin が定義する設定オブジェクト（利用可能な場合はネイティブ OpenClaw Plugin スキーマで検証されます）。
- チャンネル Plugin のアカウント／ランタイム設定は `channels.<id>` 配下にあり、中央の OpenClaw オプションレジストリではなく、所有する Plugin のマニフェストにある `channelConfigs` メタデータで記述する必要があります。

### Codex ハーネス Plugin の設定

バンドルされている `codex` Plugin は、ネイティブ Codex app-server ハーネスの設定を
`plugins.entries.codex.config` 配下で所有します。設定項目の全体については
[Codex ハーネスリファレンス](/ja-JP/plugins/codex-harness-reference)を、ランタイムモデルについては
[Codex ハーネス](/ja-JP/plugins/codex-harness)を参照してください。

`codexPlugins` は、ネイティブ Codex ハーネスを選択したセッションにのみ適用されます。
OpenClaw プロバイダーの実行、ACP
会話バインディング、または Codex 以外のハーネスに対して Codex Plugin を有効にするものではありません。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
                allow_destructive_actions: false,
              },
            },
          },
        },
      },
    },
  },
}
```

- `plugins.entries.codex.config.codexPlugins.enabled`: Codex ハーネスでネイティブ Codex
  Plugin／アプリのサポートを有効にします。デフォルト: `false`。
- `plugins.entries.codex.config.codexPlugins.allow_all_plugins`: 認証済み Codex アカウントに接続され、現在アクセス可能なすべてのアプリを、
  新しい各ネイティブ Codex スレッドで公開します。デフォルト: `false`。
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions`:
  設定済み Plugin アプリからの確認要求に対する、破壊的アクションのデフォルトポリシーです。
  安全な Codex 承認スキーマを確認なしで受け入れるには `true`、拒否するには `false`、
  Codex が要求する承認を OpenClaw
  Plugin の承認経由で処理するには `"auto"`、永続的な承認なしですべての Plugin の書き込み／破壊的
  アクションについて確認するには `"ask"` を使用します。`"ask"` モードでは、対象アプリに対する
  Codex のツールごとの永続的な承認オーバーライドを消去し、Codex スレッドの開始前に
  そのアプリの承認レビュアーとして人間を選択します。
  デフォルト: `true`。
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.enabled`: グローバルな `codexPlugins.enabled` も true の場合に、
  設定済み Plugin エントリを有効にします。
  明示的なエントリのデフォルト: `true`。
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.marketplaceName`:
  安定したマーケットプレイス ID。解決されるすべてのエントリで `pluginName` とともに必須です。
  `"openai-curated"` および `"workspace-directory"` をサポートします。いずれかの
  ID フィールドがないエントリは無視されます。
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.pluginName`: 安定した
  Codex Plugin ID。`marketplaceName` とともに必須です。
  `workspace-directory` エントリでは、`plugin/list` が返すマーケットプレイス修飾済みの
  `summary.id` を正確に使用する必要があります。例:
  `"example-plugin@workspace-directory"`。
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.allow_destructive_actions`:
  Plugin ごとの破壊的アクションのオーバーライド。省略した場合は、グローバルな
  `allow_destructive_actions` の値が使用されます。Plugin ごとの値には、同じ
  `true`、`false`、`"auto"`、`"ask"` ポリシーを指定できます。

`"ask"` を使用する許可済みの各 Plugin アプリは、そのアプリの承認リクエストを
人間のレビュアーにルーティングします。他のアプリおよびアプリ以外のスレッド承認では、
設定済みのレビュアーが維持されるため、異なる Plugin ポリシーを混在させても `"ask"` の動作は継承されません。

`codexPlugins.enabled` はグローバルな有効化ディレクティブです。移行によって書き込まれた明示的な Plugin
エントリは、厳選されたインストールおよび修復の対象となる永続的なセットです。手動で設定した `workspace-directory` エントリは、
すでにインストールされ有効になっており、その所有アプリにアクセスできる必要があります。OpenClaw は、
それらのインストールや認証を行いません。Codex が明示的なワークスペース
カタログのリクエストを拒否した場合、有効なワークスペースエントリは
`marketplace_missing` によりフェイルクローズしますが、デフォルトカタログの厳選されたエントリは
引き続き利用できます。`plugins["*"]` はサポートされておらず、`install` スイッチもありません。また、
ローカルの `marketplacePath` 値はホスト固有であるため、意図的に設定フィールドにはしていません。
app-server のバージョンおよび準備要件については、
[ネイティブ Codex Plugin](/ja-JP/plugins/codex-native-plugins)を参照してください。

`app/list` の準備状況チェックは 1 時間キャッシュされ、古くなると
非同期で更新されます。Codex スレッドのアプリ設定は、毎ターンではなく Codex ハーネスの
セッション確立時に計算されます。ネイティブ Plugin の設定を変更した後は、`/new`、`/reset`、または Gateway の
再起動を使用してください。

`codexPlugins.allow_all_plugins` は、現在アクセス可能なすべてのアカウント
アプリを、新しい各ネイティブ Codex スレッドにスナップショットします。Plugin やアプリはインストールされず、
アクセスできないアプリは除外されたままです。アカウントアプリにはグローバルな
`codexPlugins.allow_destructive_actions` ポリシーが適用されます。同じアプリが両方の経路に存在する場合は、
明示的な Plugin エントリが優先されます。`app/list` を読み取れない場合、
アカウント全体への公開はフェイルクローズします。

- `plugins.entries.firecrawl.config.webFetch`: Firecrawl Web フェッチプロバイダーの設定。
  - `apiKey`: 上限を引き上げるためのオプションの Firecrawl API キー（SecretRef を受け入れます）。`plugins.entries.firecrawl.config.webSearch.apiKey` または `FIRECRAWL_API_KEY` 環境変数にフォールバックします。
  - `baseUrl`: Firecrawl API のベース URL（デフォルト: `https://api.firecrawl.dev`。セルフホストのオーバーライドはプライベート／内部エンドポイントを対象にする必要があります）。
  - `onlyMainContent`: ページから主要コンテンツのみを抽出します（デフォルト: `true`）。
  - `maxAgeMs`: キャッシュの最大有効期間（ミリ秒）（デフォルト: `172800000` / 2 日）。
  - `timeoutSeconds`: スクレイプリクエストのタイムアウト（秒）（デフォルト: `60`）。
- `plugins.entries.xai.config.xSearch`: xAI X Search（Grok Web 検索）の設定。
  - `enabled`: X Search プロバイダーを有効にします。
  - `model`: 検索に使用する Grok モデル（例: `"grok-4.3"`）。
- `plugins.entries.memory-core.config.dreaming`: メモリ Dreaming の設定。フェーズとしきい値については、[Dreaming](/ja-JP/concepts/dreaming)を参照してください。
  - `enabled`: Dreaming のマスタースイッチ（デフォルト `false`）。
  - `frequency`: Dreaming の完全な各スイープを実行する Cron 間隔（デフォルトは `"0 3 * * *"`）。
  - `model`: オプションの Dream Diary サブエージェントモデルのオーバーライド。`plugins.entries.memory-core.subagent.allowModelOverride: true` が必要です。ターゲットを制限するには `allowedModels` と組み合わせます。モデルが利用できないエラーの場合は、セッションのデフォルトモデルで 1 回再試行します。信頼または許可リストの失敗では、暗黙にフォールバックしません。
  - フェーズポリシーとしきい値は実装の詳細です（ユーザー向けの設定キーではありません）。
- メモリ設定の全体については、[メモリ設定リファレンス](/ja-JP/reference/memory-config)を参照してください。
  - `memory.search.*`
  - エージェントごとのオーバーライドには `agents.entries.*.memory.search.*`
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- 有効な Claude バンドル Plugin は、`settings.json` から埋め込みの OpenClaw デフォルトも提供できます。OpenClaw はそれらを、未加工の OpenClaw 設定パッチではなく、サニタイズされたエージェント設定として適用します。
- `plugins.slots.memory`: アクティブなメモリ Plugin ID を選択します。メモリ Plugin を無効にするには `"none"` を指定します。
- `plugins.slots.contextEngine`: アクティブなコンテキストエンジン Plugin ID を選択します。別のエンジンをインストールして選択しない限り、デフォルトは `"legacy"` です。

[Plugin](/ja-JP/tools/plugin)を参照してください。

---

## ブラウザ

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // opt in only for trusted private-network access
      // allowPrivateNetwork: true, // legacy alias
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    tabCleanup: {
      enabled: true,
      idleMinutes: 120,
      maxTabsPerSession: 8,
      sweepMinutes: 5,
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    // attachOnly: false,
  },
}
```

- `evaluateEnabled: false` は `act:evaluate` と `wait --fn` を無効にします。
- `tabCleanup` は、アイドル時間の経過後、またはセッションが上限を超えたときに、追跡対象のプライマリエージェントの
  タブをベストエフォートで定期的にクリーンアップする処理を制御します。追跡の対象となるのは、
  ブラウザツール `action: "open"` によって作成されたタブのみです。ユーザーが開いたタブや
  所有者が不明なタブが追跡対象として取り込まれることはありません。`tabCleanup` を無効にしても、明示的なセッションライフサイクルのクリーンアップは無効になりません。
- 安定したネイティブ CDP ターゲットとブラウザ ID を使用してホストローカルで開いたタブは、
  共有 SQLite 状態に保存され、Gateway の再起動後も
  `/new` とセッションライフサイクルのクリーンアップの対象となります。ネイティブツール向けの CDP ターゲットも、
  再起動後にアイドルおよび上限超過クリーンアップの対象となります。Chrome MCP は
  プロセスローカルのターゲットハンドルを使用するため、コールド状態の既存セッションレコードは、
  再起動後の帰属不明なアクティビティに対してアイドルスイープを実行するリスクを避け、ライフサイクルのクリーンアップを待機します。
  OpenClaw は閉じる前に、プロファイルとブラウザインスタンスを
  検証します。Chrome MCP の自動接続、`/json/version` ブラウザ
  ID の欠落、未解決のネイティブターゲットは完全にプロセスローカルのままになるため、
  再起動後に自動的に閉じられることはありません。追跡されていない古いタブは、
  手動で閉じる必要があります。一時的な障害は保留状態となり、後で再試行されます。
  [タブクリーンアップの所有権](/ja-JP/tools/browser#tab-cleanup-ownership)を参照してください。
- `ssrfPolicy.dangerouslyAllowPrivateNetwork` は未設定の場合に無効となるため、ブラウザナビゲーションはデフォルトで厳格なままです。
- プライベートネットワークへのブラウザナビゲーションを意図的に信頼する場合にのみ、`ssrfPolicy.dangerouslyAllowPrivateNetwork: true` を設定してください。
- 厳格モードでは、リモート CDP プロファイルエンドポイント（`profiles.*.cdpUrl`）にも、到達可能性および検出のチェック時に同じプライベートネットワークのブロックが適用されます。
- `ssrfPolicy.allowPrivateNetwork` は従来のエイリアスとして引き続きサポートされます。
- 厳格モードでは、明示的な例外として `ssrfPolicy.hostnameAllowlist` と `ssrfPolicy.allowedHostnames` を使用してください。
- リモートプロファイルはアタッチ専用です（開始、停止、リセットは無効）。
- `profiles.*.cdpUrl` は `http://`、`https://`、`ws://`、`wss://` を受け付けます。
  OpenClaw に `/json/version` を検出させる場合は HTTP(S) を使用し、
  プロバイダーから直接 DevTools WebSocket URL が提供される場合は WS(S) を使用してください。
- 外部で管理されている CDP サービスにループバック経由で到達できる場合は、
  そのプロファイルの `attachOnly: true` を設定してください。設定しない場合、OpenClaw はループバックポートを
  ローカル管理のブラウザプロファイルとして扱い、ローカルポートの所有権エラーを報告することがあります。
- `existing-session` プロファイルは CDP の代わりに Chrome MCP を使用し、
  選択したホスト上または接続済みのブラウザ Node 経由でアタッチできます。
- `existing-session` プロファイルでは、`userDataDir` を設定して、
  Brave や Edge など、Chromium ベースの特定のブラウザプロファイルを対象にできます。
- `existing-session` プロファイルでは、Chrome がすでに DevTools HTTP(S) 検出エンドポイントまたは
  直接 WS(S) エンドポイントの背後で実行されている場合に、`cdpUrl` を設定できます。この
  モードでは、OpenClaw は自動接続を使用せず、エンドポイントを Chrome MCP に渡します。
  Chrome MCP の起動引数では `userDataDir` は無視されます。
- `existing-session` プロファイルには、現在の Chrome MCP ルートの制限が引き続き適用されます。
  CSS セレクターによるターゲット指定ではなくスナップショット／参照ベースのアクション、単一ファイルのアップロード
  フック、ダイアログのタイムアウト上書き不可、`wait --load networkidle` なし、および
  `responsebody`、PDF エクスポート、ダウンロードのインターセプト、バッチアクションなし、という制限です。
- ローカル管理の `openclaw` プロファイルでは、`cdpPort` と `cdpUrl` が自動的に割り当てられます。
  `cdpUrl` を明示的に設定するのは、リモート CDP プロファイルまたは既存セッションのエンドポイントへの
  アタッチの場合のみです。
- ローカル管理プロファイルでは、`executablePath` を設定して、そのプロファイルのグローバル
  `browser.executablePath` を上書きできます。これを使用すると、1 つのプロファイルを
  Chrome で実行し、別のプロファイルを Brave で実行できます。
- 自動検出順序：Chromium ベースの場合はデフォルトブラウザ → Chrome → Brave → Edge → Chromium → Chrome Canary。
- `browser.executablePath` と `browser.profiles.<name>.executablePath` はどちらも、
  Chromium の起動前に、OS のホームディレクトリを表す `~` と `~/...` を受け付けます。
  `existing-session` プロファイルごとの `userDataDir` でもチルダが展開されます。
- 制御サービス：ループバックのみ（ポートは `gateway.port` から派生、デフォルトは `18791`）。
- `extraArgs` は、ローカル Chromium の起動時に追加の起動フラグを付加します（例：
  `--disable-gpu`、ウィンドウサイズ、デバッグフラグ）。

---

## UI

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // 絵文字、短いテキスト、画像 URL、またはデータ URI
    },
    prefs: {
      theme: "claw", // claw | knot | dash | custom
      themeMode: "system", // light | dark | system
      locale: "en",
      chatShowThinking: true,
      chatShowToolCalls: true,
      chatPersistCommentary: true, // Control UI で実行後もコメンタリーを保持します。チャンネルには配信されません
      chatSendShortcut: "enter", // enter | modifier-enter
      chatFollowUpMode: "steer", // steer | queue。サーバーのキューモードを使用する場合は省略します
      showAdvancedSettings: false, // Settings のすべての Advanced グループを展開します
    },
  },
}
```

- `seamColor`：ネイティブアプリの UI クローム用のアクセントカラー（Talk Mode のバブル色など）。
- `assistant`：Control UI の ID の上書き。アクティブなエージェントの ID にフォールバックします。
- `prefs`：デバイス間で共有されるオペレーター設定。これは標準の保存先であり、エージェントは
  承認ゲートを通じて設定を変更でき、すべての Control UI クライアントが同期された
  状態を維持できます。ブラウザは即時起動のために値をローカルストレージへミラーリングし、
  設定を書き込めない場合（閲覧者スコープ、オフライン）にはデバイスローカルのコピーを保持します。
  `chatPersistCommentary` のデフォルトは `true` です。`false` に設定すると、実行中はライブ
  コメンタリーが表示されたままになりますが、完了時に削除され、新しい
  Codex コメンタリーが永続的なトランスクリプトのミラーに追加されなくなります。メッセージングチャンネルへの
  配信は引き続き別個であり、変更されません。
  `showAdvancedSettings` のデフォルトは `false` です。Settings の検索では、この設定を変更せずに、
  一致する Advanced グループを一時的に 1 つ開くことがあります。
  テキストの拡大率、チャット幅、サイドバーのライブアクティビティなど、表示専用の
  設定はブラウザローカルのままであり、Settings で構成します。
  接続中のクライアントにはサーバー側の変更がリアルタイムで適用されます。Gateway は
  永続化された設定への書き込みのたびに、ハッシュのみの `config.changed` イベントをブロードキャストし、
  クライアントはスナップショットを更新します（ローカルの設定下書きに未保存の編集がある間はスキップされます）。
  再接続したクライアントは接続時に整合を取ります。

---

## Gateway

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // または OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // mode=trusted-proxy 用。/gateway/trusted-proxy-auth を参照
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },
    tailscale: {
      mode: "off", // off | serve | funnel
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // toolTitles: false, // ツール呼び出しに AI による目的タイトルを表示するオプトイン設定（ユーティリティモデルのトークンを消費します）
      // embedSandbox: "scripts", // strict | scripts | trusted
      // allowExternalEmbedUrls: false, // 危険：絶対外部 http(s) 埋め込み URL を許可します
      // allowedOrigins: ["https://control.example.com"], // ループバック以外の Control UI では必須です
      // dangerouslyAllowHostHeaderOriginFallback: false, // 危険な Host ヘッダーのオリジンフォールバックモード
    },
    terminal: {
      enabled: false,
      // shell: "/bin/zsh",
    },
    remote: {
      url: "ws://127.0.0.1:18789",
      transport: "ssh", // ssh | direct
      token: "your-token",
      // password: "your-password",
    },
    trustedProxies: ["10.0.0.1"],
    // 任意。デフォルトは false。
    allowRealIpFallback: false,
    nodes: {
      pairing: {
        // 任意。デフォルトでは未設定／無効。
        autoApproveCidrs: ["192.168.1.0/24", "fd00:1234:5678::/64"],
        // SSH 検証済みの自動承認。デフォルト：有効（true）。
        // false に設定すると SSH 検証のみが無効になります。上記の
        // autoApproveCidrs には影響しません。Node のペアリングを手動のみにするには、false に設定し、
        // さらに autoApproveCidrs を未設定にします。調整するにはオブジェクトを渡します：{ user, identity,
        // timeoutMs, cidrs }。
        sshVerify: true,
      },
      commands: {
        allow: ["canvas.navigate"],
        deny: ["system.run"],
      },
    },
    tools: {
      // 追加の /tools/invoke HTTP 拒否設定
      deny: ["browser"],
      // オーナー／管理者の呼び出し元について、デフォルトの HTTP 拒否リストからツールを削除します
      allow: ["gateway"],
    },
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },
  },
}
```

<Accordion title="Gateway フィールドの詳細">

- `mode`: `local`（Gateway を実行）または `remote`（リモート Gateway に接続）。`local` でない限り、Gateway は起動を拒否します。
- `port`: WS + HTTP 用の単一多重化ポート。優先順位: `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`。
- `bind`: `auto`、`loopback`（デフォルト）、`lan`（`0.0.0.0`）、`tailnet`（利用可能な場合は Tailscale IPv4、それ以外はループバック）、または `custom`（1 つの IPv4 アドレス）。解決された `tailnet` アドレス、および `127.0.0.1` または `0.0.0.0` 以外の `custom` アドレスでは、同一ホストのクライアント用に同じポート上の `127.0.0.1` が必要です。いずれかのリスナーがバインドできない場合、起動は失敗します。非ループバックへの公開は、選択したインターフェースに限定されたままです。
- **従来のバインドエイリアス**: ホストエイリアス（`0.0.0.0`、`127.0.0.1`、`localhost`、`::`、`::1`）ではなく、`gateway.bind` のバインドモード値（`auto`、`loopback`、`lan`、`tailnet`、`custom`）を使用してください。
- **Docker に関する注意**: デフォルトの `loopback` バインドは、コンテナ内の `127.0.0.1` でリッスンします。Docker ブリッジネットワーク（`-p 18789:18789`）では、トラフィックが `eth0` に到達するため、Gateway にアクセスできません。`--network host` を使用するか、`bind: "lan"`（または `customBindHost: "0.0.0.0"` を指定した `bind: "custom"`）を設定して、すべてのインターフェースでリッスンしてください。
- **認証**: デフォルトで必須です。非ループバックへのバインドには Gateway 認証が必要です。実際には、共有トークン／パスワード、または `gateway.auth.mode: "trusted-proxy"` を使用する ID 対応リバースプロキシが必要です。オンボーディングウィザードはデフォルトでトークンを生成します。
- `gateway.auth.token` と `gateway.auth.password` の両方が設定されている場合（SecretRef を含む）、`gateway.auth.mode` を明示的に `token` または `password` に設定してください。両方が設定され、モードが未設定の場合、起動およびサービスのインストール／修復フローは失敗します。
- `gateway.auth.mode: "none"`: 明示的な認証なしモード。信頼できる local loopback セットアップにのみ使用してください。これは意図的にオンボーディングプロンプトでは提供されません。
- `gateway.auth.mode: "trusted-proxy"`: ブラウザー／ユーザー認証を ID 対応リバースプロキシに委任し、`gateway.trustedProxies` からの ID ヘッダーを信頼します（[信頼済みプロキシ認証](/ja-JP/gateway/trusted-proxy-auth)を参照）。このモードはデフォルトで**非ループバック**のプロキシソースを想定します。同一ホストのループバックリバースプロキシには、明示的な `gateway.auth.trustedProxy.allowLoopback = true` が必要です。同一ホストの内部呼び出し元は、ローカル直接フォールバックとして `gateway.auth.password` を使用できます。`gateway.auth.token` は引き続き信頼済みプロキシモードと相互排他的です。
- `gateway.auth.allowTailscale`: `true` の場合、Tailscale Serve の ID ヘッダーで Control UI／WebSocket 認証を満たすことができます（`tailscale whois` で検証）。HTTP API エンドポイントは、その Tailscale ヘッダー認証を使用**しません**。代わりに、Gateway の通常の HTTP 認証モードに従います。このトークン不要のフローでは、Gateway ホストが信頼されていることを前提とします。`tailscale.mode = "serve"` の場合、デフォルトは `true` です。
- `gateway.auth.rateLimit`: オプションの認証失敗リミッター。クライアント IP ごと、および認証スコープごとに適用されます（共有シークレットとデバイストークンは個別に追跡されます）。ブロックされた試行は `429` + `Retry-After` を返します。
  - 非同期の Tailscale Serve Control UI パスでは、同じ `{scope, clientIp}` に対する失敗した試行は、失敗の書き込み前に直列化されます。そのため、同じクライアントから不正な試行が同時に行われた場合、両方が単純な不一致として競合通過するのではなく、2 番目のリクエストでリミッターが作動することがあります。
  - `gateway.auth.rateLimit.exemptLoopback` のデフォルトは `true` です。localhost トラフィックも意図的にレート制限する場合（テスト用セットアップや厳格なプロキシデプロイなど）は、`false` を設定してください。
- ブラウザー起点の WS 認証試行は、ブラウザーを利用した localhost への総当たり攻撃に対する多層防御として、ループバック除外を無効にした状態で常にスロットリングされます。
- ループバックでは、ブラウザー起点のこれらのロックアウトは、正規化された `Origin`
  値ごとに分離されるため、ある localhost オリジンから繰り返し失敗しても、
  別のオリジンが自動的にロックアウトされることはありません。
- `tailscale.mode`: `serve`（tailnet のみ、ループバックバインド）または `funnel`（公開、認証が必要）。
- `tailscale.serviceName`: Serve モード用のオプションの Tailscale Service 名（例:
  `svc:openclaw`）。設定すると、OpenClaw はこれを `tailscale serve
--service` に渡し、Control UI をデバイスのホスト名ではなく
  名前付き Service を通じて公開できるようにします。値は Tailscale の `svc:<dns-label>`
  Service 名形式を使用する必要があります。起動時に、導出された Service URL が報告されます。
- `tailscale.preserveFunnel`: `true` かつ `tailscale.mode = "serve"` の場合、OpenClaw は
  起動時に Serve を再適用する前に `tailscale funnel status` を確認し、外部で設定された Funnel ルートがすでに Gateway ポートを
  対象としている場合は再適用をスキップします。
  デフォルトは `false` です。
- `controlUi.allowedOrigins`: Gateway WebSocket 接続用の明示的なブラウザーオリジン許可リスト。公開された非ループバックのブラウザーオリジンでは必須です。ループバック、RFC1918／リンクローカル、`.local`、`.ts.net`、または Tailscale CGNAT ホストから読み込まれるプライベートな同一オリジンの LAN／Tailnet UI は、Host ヘッダーのフォールバックを有効にしなくても許可されます。
- `controlUi.toolTitles`: Control UI チャット内のツール呼び出しに対して、AI が生成する目的タイトルを有効にします。デフォルト: `false`（ツールのレンダリングはバックグラウンドでモデルを呼び出さず、完全に決定的なままです）。有効にすると、`chat.toolTitles` メソッドは標準のユーティリティモデルルーティングを通じて複雑な呼び出しにラベルを付けます。使用されるのは、エージェントの `utilityModel`（他のすべてのユーティリティタスクと同様に、限定されたツール引数を選択したプロバイダーへ送信する可能性がある、オペレーターによる決定）、またはセッションプロバイダーが宣言した小規模モデルのデフォルト（OpenAI → `gpt-5.6-luna`、Anthropic → `claude-haiku-4-5`）です。結果はエージェントごとの状態データベースにキャッシュされるため、同じ表示で再度課金されることはありません。`utilityModel: \"\"` は、他のすべてのユーティリティタスクと同様にタイトルを無効にします。タイトルがプライマリモデルへフォールバックすることはありません。
- `controlUi.dangerouslyAllowHostHeaderOriginFallback`: Host ヘッダーのオリジンポリシーに意図的に依存するデプロイ向けに、Host ヘッダーのオリジンフォールバックを有効にする危険なモード。
- `terminal.enabled`: 管理者スコープのオペレーターターミナルを有効にします。デフォルト: `false`。ターミナルは選択されたエージェントワークスペース内でホスト PTY を起動し、Gateway プロセスの環境を継承します。`sandbox.mode: "all"` のエージェントでは使用が拒否されます。信頼できるオペレーターデプロイでのみ有効にしてください。この設定を変更すると Gateway が再起動し、Control UI のコンテンツセキュリティポリシーが更新されます。
- `terminal.shell`: オプションのシェル実行ファイル。未設定の場合、OpenClaw は Unix では `$SHELL`、Windows では `%ComSpec%` を使用します。
- `terminal.detachedSessionTimeoutSeconds`: 接続が切断された後（ページの再読み込み、ノート PC のスリープ）もターミナルセッションを存続させる時間。この間、最近の出力を再生しながら `terminal.attach` 経由で再接続できます。デフォルト: `300`。接続が切断された瞬間にセッションを終了するには、`0` を設定します。切断されたセッションでもコマンドは実行され続けるため、共有ホストや公開ホストではこの時間を短くしてください。
- `remote.transport`: `ssh`（デフォルト）または `direct`（ws/wss）。`direct` の場合、公開ホストでは `remote.url` を `wss://` にする必要があります。平文の `ws://` は、ループバック、LAN、リンクローカル、`.local`、`.ts.net`、および Tailscale CGNAT ホストでのみ許可されます。
- `remote.remotePort`: リモート SSH ホスト上の Gateway ポート。デフォルトは `18789` です。ローカルトンネルポートがリモート Gateway ポートと異なる場合に使用してください。
- `remote.tlsFingerprint`: リモート `wss://` Gateway で期待される SHA-256 証明書フィンガープリント。macOS アプリは、これをオペレーター／制御接続とコンパニオン Node 接続の両方に適用します。明示的な値がない場合、macOS は通常のシステム信頼検証が成功した後に限り、初回使用時のピンを記録します。
- `remote.sshHostKeyPolicy`: macOS SSH トンネルのホストキーポリシー。`strict` がデフォルトで、すでに信頼されているキーが必要です。`openssh` は、管理対象エイリアスに対して実効 OpenSSH 設定を使用するための明示的なオプトインです。使用する前に、該当するユーザーおよびシステムの SSH 設定を確認してください。macOS アプリと `configure-remote` は、ターゲットを変更すると、再度明示的にオプトインしない限り、このポリシーを `strict` にリセットします。
- `gateway.remote.token` / `.password` はリモートクライアントの認証情報フィールドです。それだけでは Gateway 認証を設定しません。
- `gateway.push.apns.relay.baseUrl`: リレー対応の iOS ビルドが登録情報を Gateway に公開した後に使用する、外部 APNs リレーのベース HTTPS URL。公開 App Store ビルドは、ホストされている OpenClaw リレーを使用します。カスタムリレー URL は、リレー URL がそのリレーを指すよう意図的に分離された iOS ビルド／デプロイパスと一致する必要があります。
- `gateway.push.apns.relay.timeoutMs`: Gateway からリレーへの送信タイムアウト（ミリ秒）。デフォルトは `10000` です。
- リレー対応の登録は、特定の Gateway ID に委任されます。ペアリング済みの iOS アプリは `gateway.identity.get` を取得し、その ID をリレー登録に含め、登録スコープの送信許可を Gateway に転送します。別の Gateway は、その保存済み登録を再利用できません。
- `OPENCLAW_APNS_RELAY_BASE_URL` / `OPENCLAW_APNS_RELAY_TIMEOUT_MS`: 上記のリレー設定に対する一時的な環境変数オーバーライド。
- `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true`: ループバック HTTP リレー URL 用の、開発専用の緊急回避手段。本番環境のリレー URL では HTTPS を維持してください。
- `OPENCLAW_HANDSHAKE_TIMEOUT_MS`: 組み込みの認証前 Gateway WebSocket ハンドシェイクタイムアウトに対する、オプションの環境変数オーバーライド。
- `channels.<provider>.healthMonitor.enabled`: グローバルモニターを有効にしたまま、ヘルスモニターによる再起動をチャンネル単位で無効にします。
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: 複数アカウント対応チャンネル用のアカウント単位のオーバーライド。設定すると、チャンネル単位のオーバーライドより優先されます。
- ローカル Gateway の呼び出しパスでは、`gateway.auth.*` が未設定の場合に限り、`gateway.remote.*` をフォールバックとして使用できます。
- `gateway.auth.token` / `gateway.auth.password` が SecretRef を介して明示的に設定され、解決できない場合、解決はフェイルクローズします（リモートフォールバックによる隠蔽は行われません）。
- `trustedProxies`: TLS を終端するか、転送クライアントヘッダーを挿入するリバースプロキシの IP。制御下にあるプロキシのみを列挙してください。ループバックエントリは、同一ホストのプロキシ／ローカル検出セットアップ（Tailscale Serve やローカルリバースプロキシなど）でも有効ですが、ループバックリクエストが `gateway.auth.mode: "trusted-proxy"` の対象になるわけでは**ありません**。
- `allowRealIpFallback`: `true` の場合、`X-Forwarded-For` がないときに Gateway は `X-Real-IP` を受け入れます。フェイルクローズ動作のデフォルトは `false` です。
- `gateway.nodes.pairing.autoApproveCidrs`: 要求スコープがない初回 Node デバイスペアリングを自動承認するための、オプションの CIDR／IP 許可リスト。未設定の場合は無効です。これは、オペレーター／ブラウザー／Control UI／WebChat のペアリングを自動承認せず、ロール、スコープ、メタデータ、公開キーのアップグレードも自動承認しません。
- `gateway.nodes.pairing.sshVerify`: 初回 Node デバイスペアリングに対する SSH 検証済み自動承認（デフォルト: 有効）。Gateway はペアリングホストへ SSH で接続し直し（BatchMode、厳格なホストキー）、`openclaw node identity` デバイスキーが完全に一致した場合にのみ承認します。適格性の最低条件は `autoApproveCidrs` と同じです。`cidrs` で上書きしない限り、プローブはプライベート／CGNAT ソースアドレスに限定されます。無効にするには `false` を設定し、調整するには `{ user, identity, timeoutMs, cidrs }` を設定してください。[Node のペアリング](/ja-JP/gateway/pairing#ssh-verified-device-auto-approval-default)を参照してください。
- `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny`: ペアリングおよびプラットフォームの許可リスト評価後に、宣言された Node コマンドをグローバルに許可または拒否するための設定。`camera.snap`、`camera.clip`、`screen.record`、`health.summary`、`sms.search`、`sms.send` などの危険な Node コマンドを有効にするには、`commands.allow` を使用します。`commands.deny` は、プラットフォームのデフォルト設定または明示的な許可によって含まれるコマンドも削除します。iOS のヘルスケア権限、Android の SMS 権限、および Gateway のコマンド認可は、それぞれ独立しています。Node が宣言済みコマンドリストを変更した後、Gateway が更新済みのコマンドスナップショットを保存できるよう、そのデバイスのペアリングを拒否してから再承認してください。
- `gateway.tools.deny`: HTTP `POST /tools/invoke` で追加でブロックするツール名（デフォルトの拒否リストを拡張）。
- `gateway.tools.allow`: オーナーまたは管理者の呼び出し元に対して、デフォルトの HTTP 拒否リストから
  ツール名を削除します。これにより、ID 情報を持つ `operator.write` の
  呼び出し元がオーナーまたは管理者のアクセス権に昇格することはありません。`cron`、`gateway`、および `nodes` は、
  許可リストに含まれていても、オーナー以外の呼び出し元は引き続き利用できません。

</Accordion>

### OpenAI 互換エンドポイント

- 管理 HTTP RPC: `admin-http-rpc` Plugin と同様にデフォルトでは無効です。Plugin を有効にすると `POST /api/v1/admin/rpc` が登録されます。[管理 HTTP RPC](/ja-JP/plugins/admin-http-rpc)を参照してください。
- Chat Completions: デフォルトでは無効です。`gateway.http.endpoints.chatCompletions.enabled: true` で有効にします。
- Responses API: `gateway.http.endpoints.responses.enabled`。
- Responses の URL 入力の強化:
  - `gateway.http.endpoints.responses.maxUrlParts`
  - `gateway.http.endpoints.responses.files.urlAllowlist`
  - `gateway.http.endpoints.responses.images.urlAllowlist`
    空の許可リストは未設定として扱われます。URL 取得を無効にするには、
    `gateway.http.endpoints.responses.files.allowUrl=false` および／または `gateway.http.endpoints.responses.images.allowUrl=false` を使用します。
- オプションのレスポンス強化ヘッダー:
  - `gateway.http.securityHeaders.strictTransportSecurity`（管理下にある HTTPS オリジンにのみ設定してください。[信頼済みプロキシ認証](/ja-JP/gateway/trusted-proxy-auth#tls-termination-and-hsts)を参照）

### 複数インスタンスの分離

一意のポートと状態ディレクトリを使用して、1 台のホスト上で複数の Gateway を実行します。

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

便利なフラグ: `--dev`（`~/.openclaw-dev` + ポート `19001` を使用）、`--profile <name>`（`~/.openclaw-<name>` を使用）。

[複数の Gateway](/ja-JP/gateway/multiple-gateways)を参照してください。

### `gateway.tls`

```json5
{
  gateway: {
    tls: {
      enabled: false,
      autoGenerate: false,
      certPath: "/etc/openclaw/tls/server.crt",
      keyPath: "/etc/openclaw/tls/server.key",
      caPath: "/etc/openclaw/tls/ca-bundle.crt",
    },
  },
}
```

- `enabled`: Gateway リスナー（HTTPS/WSS）で TLS 終端を有効にします（デフォルト: `false`）。
- `autoGenerate`: 明示的なファイルが設定されていない場合に、ローカルの自己署名証明書／鍵ペアを自動生成します。ローカル／開発用途専用です。
- `certPath`: TLS 証明書ファイルへのファイルシステムパス。
- `keyPath`: TLS 秘密鍵ファイルへのファイルシステムパス。アクセス権限を制限してください。
- `caPath`: クライアント検証またはカスタム信頼チェーン用のオプションの CA バンドルパス。

### `gateway.reload`

```json5
{
  gateway: {
    reload: {
      mode: "hybrid", // off | restart | hot | hybrid
      debounceMs: 500,
      deferralTimeoutMs: 300000,
    },
  },
}
```

- `mode`: 設定の編集を実行時に適用する方法を制御します。
  - `"off"`: ライブ編集を無視します。変更には明示的な再起動が必要です。
  - `"restart"`: 設定変更時に常に Gateway プロセスを再起動します。
  - `"hot"`: 再起動せずにプロセス内で変更を適用します。
  - `"hybrid"`（デフォルト）: 最初にホットリロードを試行し、必要な場合は再起動にフォールバックします。
- `debounceMs`: 設定変更を適用する前のデバウンス時間（ミリ秒）（非負整数、デフォルト: `300`）。
- `deferralTimeoutMs`: 再起動またはチャンネルのホットリロードを強制する前に、進行中の処理を待機するオプションの最大時間（ミリ秒）。省略するとデフォルトの制限付き待機時間（`300000`）を使用します。無期限に待機し、未完了であることを定期的に警告ログへ記録するには `0` を設定します。

---

## クラウドワーカー環境

クラウドワーカーはオプトインです。`cloudWorkers` が存在しない場合、または `profiles` が空の場合、OpenClaw は新しいワーカーの作成を受け付けません。以前に作成された永続レコードは引き続き整合され、表示されたままです。既存の Gateway／Node プロジェクションは変更されません。

各ワーカープロバイダーは、信頼済みのプロビジョニング出力から SSH `hostKey` を、ホスト名やコメントを含めず正確に `algorithm base64` として返す必要があります。ブートストラップはその鍵を分離された `known_hosts` ファイルに書き込み、`StrictHostKeyChecking=yes` を使用します。プロバイダーが鍵を省略した場合は、接続を開く前に失敗します。初回使用時に信頼するフォールバックはありません。

トンネルのセットアップはプロビジョニングの一部ではなく、オンデマンドで行われます。開始すると、Gateway はワーカーローカルの Unix ソケットを、その local loopback WebSocket エンドポイントへリバースフォワードします。ソケットはランダムに割り当てられた所有者専用のリモートディレクトリに配置されます。local loopback TCP ポートとは異なり、マルチユーザーワーカー上の他のアカウントから到達できず、別の環境のポートと衝突することもありません。SSH キープアライブと上限付きの再接続バックオフは、トンネル所有者が現在の所有者である間のみ実行されます。トンネルを停止すると、SSH プロセスを閉じる前に再接続を遮断します。

制御トラフィックとワークスペース転送には別々の SSH 接続を使用します。どちらも同じ解決済みアイデンティティと分離された固定 `known_hosts` ファイルを再利用しますが、ワークスペース転送は長時間稼働するトンネルと SSH 接続の多重化を共有しないため、rsync が制御トラフィックをブロックすることはありません。

### Crabbox プロファイル

同梱の `crabbox` プロバイダーは、ローカルの Crabbox CLI を介して SSH 対応リースをプロビジョニングします。内側の `settings.provider` は Crabbox バックエンドを選択します。これは外側の OpenClaw プロバイダー ID とは別です。

```json5
{
  cloudWorkers: {
    profiles: {
      production: {
        provider: "crabbox",
        install: "bundle", // デフォルト。リリース済みの Gateway バージョンにのみ "npm" を使用します。
        settings: {
          provider: "aws",
          class: "standard",
          ttl: "24h",
          idleTimeout: "60m",
          // オプションの絶対パス。デフォルト: 兄弟の ../crabbox/bin/crabbox、次に PATH。
          binary: "/usr/local/bin/crabbox",
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `settings.provider`（必須）: `--provider` を介して渡される Crabbox バックエンド。inspect 出力に SSH エンドポイントが含まれるバックエンドを使用してください。`aws` は直接 AWS バックエンドを選択します。
- `settings.class`（必須）: `--class` に渡される Crabbox マシンクラス。
- `settings.ttl` と `settings.idleTimeout`（必須）: `--ttl` と `--idle-timeout` に渡される正の Go 期間文字列。これらのプロバイダー側フェイルセーフは、後述する OpenClaw の保存済み `lifetime` ポリシーとは別です。
- `settings.binary`: オプションの Crabbox 実行可能ファイルの絶対パス。指定しない場合、OpenClaw は兄弟の Crabbox チェックアウト、`PATH` 上の実行可能エントリの順に確認し、最後に `crabbox` を呼び出します。これにより、CLI がない場合もプロバイダーエラーとして可視化されます。

不明な設定は拒否されます。Crabbox の認証情報およびバックエンド固有のアカウント設定は、引き続き Crabbox が所有します。これらを `settings` に配置しないでください。OpenClaw はローカル CLI のみを呼び出し、この Plugin からプロバイダーへのネットワーク呼び出しは行いません。プロビジョニングでは常に `--keep=true` を渡します。OpenClaw が外部ライフサイクルを所有し、`crabbox stop` でリースを破棄します。

<Note>
  OpenClaw は、プロバイダー所有のシークレットリゾルバーを介して Crabbox のリースローカルな `sshKey` パスを解決し、`crabbox inspect --json` が返す権威ある `sshHostKey` を固定します。AWS の受け入れには `providerMetadata.instanceProfileAttached` も必要です。この閉じた検査契約には Crabbox 0.38.1 以降をインストールしてください。
</Note>

### 静的 SSH 開発プロファイル

```json5
{
  cloudWorkers: {
    profiles: {
      development: {
        provider: "static-ssh",
        settings: {
          host: "worker.example.test",
          port: 22,
          user: "openclaw",
          hostKey: "ssh-ed25519 <base64-public-host-key>",
          keyRef: {
            source: "env",
            provider: "default",
            id: "OPENCLAW_WORKER_SSH_KEY",
          },
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `profiles`: 空でなく、前後の空白を除去した ID を持つ名前付きワーカープロファイル。各プロファイルは、Plugin によって登録されたプロバイダーを選択します。
- `provider`: 空でないワーカープロバイダー ID。例では、同梱の `crabbox` プロバイダーと QA Lab の `static-ssh` プロバイダーを使用します。
- `install`: ワーカーのインストール方法。`"bundle"`（デフォルト）は、Gateway にインストールされたビルドのコンテンツハッシュ付きバンドルを転送し、リリース済み、開発中、未リリースのバージョンをサポートします。`"npm"` は、変更されていないパッケージ版リリース向けのオプトイン最適化です。公開 npm レジストリから `openclaw@<exact gateway version>` をインストールし、`latest` は決してインストールしません。
- 同梱のプロバイダー Plugin は設定されると自動的に選択されますが、明示的な無効化と `plugins.allow` は引き続き適用されます。許可リストが設定されている場合は、プロバイダー ID（例: `crabbox`）を含めてください。外部プロバイダー Plugin は、インストールしたうえで明示的に有効化する必要もあります。
- `settings`: プロバイダー所有のサイズ制限付き JSON。選択された Plugin がキーを定義して検証します。シークレットを含む値には [SecretRef オブジェクト](/ja-JP/gateway/secrets)を使用してください。静的 SSH プロバイダーには `host`、`user`、`hostKey`、`keyRef` が必要です。`port` のデフォルトは `22` です。`hostKey` は、既知のホストまたは別の信頼済みチャンネルから取得した OpenSSH 公開ホスト鍵の 1 行（`algorithm base64`）である必要があり、オプションのプレフィックスを含めてはなりません。
- `lifetime.idleTimeoutMinutes`: 後のアイドル再利用ポリシー用に保存される正の整数の分数。
- `lifetime.maxLifetimeMinutes`: 後のライフサイクルポリシー用に保存される正の整数の分数。

WAL リセットに安全な SQLite を備えた、サポート対象の Node ランタイム（22.22.3+、24.15+、または 25.9+）がワーカーにすでにインストールされている必要があります。オプトインの `"npm"` メソッドには、`npm` と公開 npm レジストリへのアウトバウンド HTTPS アクセスも必要です。ネットワークを使用するツールチェーンのセットアップはプロバイダーポリシーです。ブートストラップはツールチェーン自体をインストールせず、対処可能なエラーを報告します。

この基盤は Gateway ビルドをインストールして検証し、トンネルの開始／停止ライフサイクルを提供しますが、一般的な OpenClaw CLI は起動しません。自己完結型のワーカーエントリとループは、次のクラウドワーカーのマイルストーンで導入されます。

各永続環境レコードは、検証済みのプロバイダー設定、解決済みのインストール方法、ライフタイムポリシーを、作成時のプロファイルスナップショットに保持します。名前付きプロファイルを変更または削除すると新規作成に影響します。既存のレコードは、所有する Plugin が引き続き利用可能である限り、そのスナップショットを使用してライフサイクルの整合を継続します。

最初のクラウドワーカーリリースでは、ライフタイム値はデータとしてのみ扱われます。自動適用は後続のライフサイクル作業で導入されます。プロファイルの変更には Gateway の再起動が必要です。

<Warning>
  `static-ssh` プロバイダーは、ソースツリー用の QA Lab 開発ハーネスであり、パッケージ配布から除外されています。共有ホスト上で実行されるワーカーは、無関係なホストデータを読み取れるため、このプロバイダーを本番環境の分離境界として使用しないでください。
  オペレーターは想定される `hostKey` を指定する必要があります。OpenClaw は最初の接続から鍵を学習したり受け入れたりしません。
  リースを破棄しても OpenClaw の論理レコードが解放されるだけで、ホストは停止もクリーンアップもされません。
</Warning>

---

## フック

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: true,
    allowedSessionKeyPrefixes: ["hook:", "hook:gmail:"],
    allowedAgentIds: ["hooks", "main"],
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        agentId: "hooks",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "From: {{messages[0].from}}\nSubject: {{messages[0].subject}}\n{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

認証: `Authorization: Bearer <token>` または `x-openclaw-token: <token>`。
クエリ文字列のフックトークンは拒否されます。

検証と安全性に関する注意事項:

- `hooks.enabled=true` には空でない `hooks.token` が必要です。
- `hooks.token` は、有効な Gateway 共有シークレット認証（`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` または `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`）とは別のものにする必要があります。再利用が検出されると、起動時に致命的ではないセキュリティ警告がログに記録されます。
- `openclaw security audit` は、監査時にのみ指定された Gateway パスワード認証（`--auth password --password <password>`）を含め、フックと Gateway の認証の再利用を重大な検出事項として報告します。永続化された再利用済みの `hooks.token` をローテーションするには `openclaw doctor --fix` を実行し、その後、外部のフック送信元が新しいフックトークンを使用するように更新します。
- `hooks.path` を `/` にすることはできません。`/hooks` のような専用のサブパスを使用してください。
- `hooks.allowRequestSessionKey=true` の場合は、`hooks.allowedSessionKeyPrefixes` を制限してください（例: `["hook:"]`）。
- マッピングまたはプリセットがテンプレート化された `sessionKey` を使用する場合は、`hooks.allowedSessionKeyPrefixes` と `hooks.allowRequestSessionKey=true` を設定してください。静的なマッピングキーでは、このオプトインは不要です。

**エンドポイント:**

- `POST /hooks/wake` → `{ text, mode?: "now"|"next-heartbeat" }`
- `POST /hooks/agent` → `{ message, name?, agentId?, sessionKey?, wakeMode?, deliver?, channel?, to?, model?, thinking?, timeoutSeconds? }`
  - リクエストペイロードの `sessionKey` は、`hooks.allowRequestSessionKey=true` の場合にのみ受け入れられます（デフォルト: `false`）。
- `POST /hooks/<name>` → `hooks.mappings` によって解決
  - テンプレートでレンダリングされたマッピングの `sessionKey` 値は外部から指定されたものとして扱われ、`hooks.allowRequestSessionKey=true` も必要です。

<Accordion title="マッピングの詳細">

- `match.path` は、`/hooks` の後のサブパスと一致します（例: `/hooks/gmail` → `gmail`）。
- `match.source` は、汎用パスのペイロードフィールドと一致します。
- `{{messages[0].subject}}` のようなテンプレートはペイロードから読み取ります。
- `transform` は、フックアクションを返す JS/TS モジュールを指すことができます。
  - `transform.module` は相対パスである必要があり、`hooks.transformsDir` 内に限定されます（絶対パスとディレクトリトラバーサルは拒否されます）。
  - `hooks.transformsDir` は `~/.openclaw/hooks/transforms` の下に配置してください。ワークスペースの Skills ディレクトリは拒否されます。`openclaw doctor` がこのパスを無効と報告する場合は、変換モジュールをフック変換ディレクトリへ移動するか、`hooks.transformsDir` を削除してください。
- `agentId` は特定のエージェントへルーティングします。不明な ID はデフォルトエージェントへフォールバックします。
- `allowedAgentIds`: `agentId` が省略された場合のデフォルトエージェントパスを含め、有効なエージェントルーティングを制限します（`*` または省略 = すべて許可、`[]` = すべて拒否）。
- `defaultSessionKey`: 明示的な `sessionKey` がないフックエージェント実行用の、任意の固定セッションキーです。
- `allowRequestSessionKey`: `/hooks/agent` の呼び出し元およびテンプレート駆動のマッピングセッションキーが `sessionKey` を設定することを許可します（デフォルト: `false`）。
- `allowedSessionKeyPrefixes`: 明示的な `sessionKey` 値（リクエスト + マッピング）に対する任意のプレフィックス許可リストです（例: `["hook:"]`）。マッピングまたはプリセットのいずれかがテンプレート化された `sessionKey` を使用する場合は必須になります。
- `deliver: true` は最終応答をチャンネルへ送信します。`channel` のデフォルトは `last` です。
- `model` は、このフック実行で使用する LLM を上書きします（モデルカタログが設定されている場合は許可対象である必要があります）。

</Accordion>

### Gmail 連携

- 組み込みの Gmail プリセットは `sessionKey: "hook:gmail:{{messages[0].id}}"` を使用します。
- このメッセージ単位のキーが分離するのは会話コンテキストであり、ツールやワークスペースへのアクセスではありません。`agentId` を設定するカスタムマッピングがない場合、プリセットはデフォルトエージェントを使用します。
- 信頼できない受信トレイでは、Gmail を専用の閲覧エージェントへルーティングし、そのエージェントを[エージェント単位のサンドボックスとツールポリシー](/ja-JP/tools/multi-agent-sandbox-tools)で制限してください。閲覧エージェントがメインエージェントへ通知する必要がある場合は、[`tools.agentToAgent`](/ja-JP/gateway/config-tools#toolsagenttoagent) で引き継ぎを制限してください。推奨される脅威モデルとモデル階層については、[プロンプトインジェクション](/ja-JP/gateway/security#prompt-injection)を参照してください。
- このメッセージ単位のルーティングを維持する場合は、`hooks.allowRequestSessionKey: true` を設定し、`hooks.allowedSessionKeyPrefixes` が Gmail 名前空間と一致するように制限してください（例: `["hook:", "hook:gmail:"]`）。
- `hooks.allowRequestSessionKey: false` が必要な場合は、テンプレート化されたデフォルトではなく、静的な `sessionKey` でプリセットを上書きしてください。

```json5
{
  hooks: {
    gmail: {
      account: "openclaw@gmail.com",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

- 設定されている場合、Gateway は起動時に `gog gmail watch serve` を自動起動します。無効にするには `OPENCLAW_SKIP_GMAIL_WATCHER=1` を設定してください。
- Gateway と並行して別の `gog gmail watch serve` を実行しないでください。

---

## Canvas Plugin ホスト

```json5
{
  plugins: {
    entries: {
      canvas: {
        config: {
          host: {
            root: "~/.openclaw/workspace/canvas",
            liveReload: true,
            // enabled: false, // または OPENCLAW_SKIP_CANVAS_HOST=1
          },
        },
      },
    },
  },
}
```

- エージェントが編集可能な HTML/CSS/JS と A2UI を、Gateway ポート配下の HTTP で提供します:
  - `http://<gateway-host>:<gateway.port>/__openclaw__/canvas/`
  - `http://<gateway-host>:<gateway.port>/__openclaw__/a2ui/`
- ローカル専用: `gateway.bind: "loopback"`（デフォルト）のままにしてください。
- 非 local loopback バインド: Canvas ルートには、他の Gateway HTTP サーフェスと同様に Gateway 認証（トークン/パスワード/信頼済みプロキシ）が必要です。
- Node WebView は通常、認証ヘッダーを送信しません。Node がペアリングされ接続されると、Gateway は Canvas/A2UI アクセス用の Node スコープのケイパビリティ URL を通知します。
- ケイパビリティ URL はアクティブな Node WS セッションに紐付けられ、短時間で期限切れになります。IP ベースのフォールバックは使用されません。
- 提供する HTML にライブリロードクライアントを注入します。
- 空の場合は、初期 `index.html` を自動作成します。
- A2UI も `/__openclaw__/a2ui/` で提供します。
- 変更を反映するには Gateway の再起動が必要です。
- 大規模なディレクトリまたは `EMFILE` エラーが発生する場合は、ライブリロードを無効にしてください。

---

## 検出

### mDNS（Bonjour）

```json5
{
  discovery: {
    mdns: {
      mode: "minimal", // minimal | full | off
    },
  },
}
```

- `minimal`（デフォルト）: TXT レコードから `cliPath` + `sshPort` を省略します。
- `full`: `cliPath` + `sshPort` を含めます。LAN マルチキャスト広告には、引き続きバンドルされた `bonjour` Plugin を有効にする必要があります。
- `off`: Plugin の有効化状態を変更せずに LAN マルチキャスト広告を抑制します。
- バンドルされた `bonjour` Plugin は macOS ホストでは自動起動し、Linux、Windows、およびコンテナ化された Gateway デプロイメントではオプトインです。
- ホスト名が有効な DNS ラベルの場合、デフォルトでシステムのホスト名が使用され、それ以外の場合は `openclaw` にフォールバックします。`OPENCLAW_MDNS_HOSTNAME` で上書きできます。
- `OPENCLAW_DISABLE_BONJOUR=1` は mDNS 広告を完全に無効にし、`discovery.mdns.mode` を上書きします。

### 広域（DNS-SD）

```json5
{
  discovery: {
    wideArea: { enabled: true },
  },
}
```

`~/.openclaw/dns/` 配下にユニキャスト DNS-SD ゾーンを書き込みます。ネットワークをまたぐ検出には、DNS サーバー（CoreDNS 推奨）と Tailscale スプリット DNS を組み合わせてください。

セットアップ: `openclaw dns setup --apply`。

---

## 環境

### `env`（インライン環境変数）

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

- インライン環境変数は、プロセス環境にそのキーが存在しない場合にのみ適用されます。
- `.env` ファイル: CWD の `.env` + `~/.openclaw/.env`（どちらも既存の変数を上書きしません）。
- `shellEnv`: ログインシェルのプロファイルから、不足している想定キーをインポートします。
- 完全な優先順位については、[環境](/ja-JP/help/environment)を参照してください。

### 環境変数の置換

任意の設定文字列内で `${VAR_NAME}` を使用して環境変数を参照します:

```json5
{
  gateway: {
    auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" },
  },
}
```

- 一致するのは大文字の名前のみです: `[A-Z_][A-Z0-9_]*`。
- 変数が存在しないか空の場合、設定の読み込み時にエラーが発生します。
- リテラルの `${VAR}` には `$${VAR}` でエスケープしてください。
- `$include` でも機能します。

---

## シークレット

シークレット参照は追加的な機能です。プレーンテキスト値も引き続き機能します。

### `SecretRef`

次のいずれかのオブジェクト形式を使用します:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

検証:

- `provider` のパターン: `^[a-z][a-z0-9_-]{0,63}$`
- `source: "env"` の ID パターン: `^[A-Z][A-Z0-9_]{0,127}$`
- `source: "file"` の ID: 絶対 JSON ポインター（例: `"/providers/openai/apiKey"`）
- `source: "exec"` の ID パターン: `^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$`（AWS 形式の `secret#json_key` セレクターをサポート）
- `source: "exec"` の ID には、スラッシュで区切られたパスセグメントとして `.` または `..` を含めることはできません（例: `a/../b` は拒否されます）

### サポートされる認証情報サーフェス

- 正規マトリクス: [SecretRef 認証情報サーフェス](/ja-JP/reference/secretref-credential-surface)
- `secrets apply` は、サポートされている `openclaw.json` 認証情報パスを対象とします。
- `auth-profiles.json` 参照は、ランタイム解決と監査の対象範囲に含まれます。

### シークレットプロバイダーの設定

```json5
{
  secrets: {
    providers: {
      default: { source: "env" }, // 任意の明示的な環境変数プロバイダー
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
        timeoutMs: 5000,
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        passEnv: ["PATH", "VAULT_ADDR"],
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

注記:

- `file` プロバイダーは `mode: "json"` と `mode: "singleValue"` をサポートします（singleValue モードでは `id` は `"value"` である必要があります）。
- Windows ACL の検証を利用できない場合、ファイルおよび exec プロバイダーのパスは安全側に倒して失敗します。検証できない信頼済みパスに限り、`allowInsecurePath: true` を設定してください。
- `exec` プロバイダーには絶対 `command` パスが必要で、標準入力/標準出力でプロトコルペイロードを使用します。
- デフォルトでは、シンボリックリンクのコマンドパスは拒否されます。解決後のターゲットパスを検証しながらシンボリックリンクパスを許可するには、`allowSymlinkCommand: true` を設定してください。
- `trustedDirs` が設定されている場合、信頼済みディレクトリのチェックは解決後のターゲットパスに適用されます。
- `exec` の子プロセス環境は、デフォルトでは最小限です。必要な変数は `passEnv` で明示的に渡してください。
- シークレット参照は、アクティベーション時にメモリ内スナップショットへ解決され、その後、リクエストパスはスナップショットのみを読み取ります。
- アクティブサーフェスのフィルタリングはアクティベーション中に適用されます。有効なサーフェスで未解決の参照があると起動または再読み込みに失敗し、非アクティブなサーフェスは診断情報とともにスキップされます。

---

## 認証ストレージ

```json5
{
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:personal": { provider: "openai", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      openai: ["openai:personal"],
    },
  },
}
```

- エージェントごとのプロファイルは `<agentDir>/auth-profiles.json` に保存されます。
- `auth-profiles.json` は、静的認証情報モード向けに値レベルの参照（`api_key` には `keyRef`、`token` には `tokenRef`）をサポートします。
- `{ "provider": { "apiKey": "..." } }` のような従来のフラットな `auth-profiles.json` マップはランタイム形式ではありません。`openclaw doctor --fix` は、`.legacy-flat.*.bak` バックアップを作成したうえで、それらを正規の `provider:default` API キープロファイルに書き換えます。
- OAuth モードのプロファイル（`auth.profiles.<id>.mode = "oauth"`）は、SecretRef を使用する認証プロファイルの認証情報をサポートしません。
- 静的ランタイム認証情報は、メモリ内で解決されたスナップショットから取得されます。従来の静的な `auth.json` エントリは、検出時に消去されます。
- 従来の OAuth インポート元は `~/.openclaw/credentials/oauth.json` です。
- [OAuth](/ja-JP/concepts/oauth) を参照してください。
- シークレットのランタイム動作と `audit/configure/apply` ツールについては、[シークレット管理](/ja-JP/gateway/secrets)を参照してください。

---

## 監査

```json5
{
  audit: {
    enabled: true,
    messages: "off", // off | direct | all
  },
}
```

Gateway は、エージェント実行とツールアクションに関する**メタデータのみ**の監査イベントを共有状態データベースに記録します。メッセージのライフサイクルメタデータは、別途オプトインする必要があります。台帳には、アイデンティティ、タイミング、ツール名、正規化された結果が保存されますが、プロンプト、メッセージ本文、ツール引数、結果、生のエラーテキストは一切保存されません。メッセージ行には、プラットフォームの生のアカウント ID、会話 ID、メッセージ ID、ターゲット ID は保存されません。実行／ツールのセッションキーは相関分析に引き続き使用でき、それ自体にプラットフォームのアカウント ID やピア ID が含まれる場合があります。レコードは30日後に期限切れとなり、台帳の上限は100,000行です。[`openclaw audit`](/ja-JP/cli/audit) または [`audit.activity.list`](/ja-JP/gateway/protocol#audit-ledger-rpc) Gateway RPC で照会できます。完全なデータモデル、プライバシー上の意味、対象範囲の制限については、[監査履歴](/ja-JP/gateway/audit)を参照してください。

- `enabled`: 新しい監査イベントを記録します（デフォルト: `true`）。インシデント発生後にのみ有効化された監査証跡では、そのインシデントを説明できないため、台帳はデフォルトで有効です。`false` に設定すると、Gateway の再起動後に新しいイベントの挿入が停止します。既存のレコードは期限切れになるまで引き続き読み取り可能です。再度有効にすると、その時点から記録が再開されますが、空白期間はバックフィルされません。
- `messages`: メッセージメタデータの範囲です（デフォルト: `"off"`）。`"direct"` は、既知の直接会話のみを記録します。`"all"` は、グループ、チャンネル、不明な種類の会話も記録します。どちらのモードもコンテンツを含まず、相関付けが可能な場合は、生の識別子をインストール環境ローカルの鍵付き仮名に置き換えます。これらは匿名化ではなく相関分析の補助です。状態データベースには導出キーが保存されますが、RPC および CLI のエクスポートには含まれません。

実行中の Gateway は、起動時に `audit.enabled` と `audit.messages` を取得します。いずれかの設定を変更した後は、Gateway を再起動してください。現在のメッセージ対象範囲には、コアディスパッチに到達した受理済みの受信メッセージと、共有の永続的配信に到達した元の論理的な送信返信ペイロードごとの1件の終端行が含まれます。これらの共有境界を迂回する Plugin ローカルおよび直接送信の経路は、まだ対象外です。上限付きのバックグラウンドライターはベストエフォートであり、損失のないコンプライアンスアーカイブではありません。

---

## ログ

```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty", // pretty | compact | json
    redactSensitive: "tools", // off | tools
    redactPatterns: ["\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1"],
  },
}
```

- デフォルトのログファイル: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`。名前付きプロファイルでは `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log` が使用されます。
- 安定したパスを使用するには、`logging.file` を設定します。
- `--verbose` の場合、`consoleLevel` は `debug` に引き上げられます。
- `maxFileBytes`: ローテーション前のアクティブなログファイルの最大サイズ（バイト単位、正の整数、デフォルト: `104857600` = 100 MB）。OpenClaw は、アクティブなファイルの隣に番号付きアーカイブを最大5個保持します。
- `redactSensitive` / `redactPatterns`: コンソール出力、ファイルログ、OTLP ログレコード、永続化されたセッショントランスクリプトのテキストに対するベストエフォートのマスキングです。`redactSensitive: "off"` は、この一般的なログ／トランスクリプトポリシーのみを無効にします。UI、ツール、診断の安全性に関わるサーフェスでは、出力前に引き続きシークレットが編集されます。

---

## 診断

```json5
{
  diagnostics: {
    enabled: true,
    flags: ["telegram.*"],

    otel: {
      enabled: false,
      endpoint: "https://otel-collector.example.com:4318",
      tracesEndpoint: "https://traces.example.com/v1/traces",
      metricsEndpoint: "https://metrics.example.com/v1/metrics",
      logsEndpoint: "https://logs.example.com/v1/logs",
      protocol: "http/protobuf", // http/protobuf | grpc
      headers: { "x-tenant-id": "my-org" },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: false,
      logsExporter: "otlp",
      sampleRate: 1.0,
      flushIntervalMs: 5000,
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },

    cacheTrace: {
      enabled: false,
      filePath: "~/.openclaw/logs/cache-trace.jsonl",
      includeMessages: true,
      includePrompt: true,
      includeSystem: true,
    },
  },
}
```

- `enabled`: 計装出力のマスタートグルです（デフォルト: `true`）。
- `flags`: 対象を絞ったログ出力を有効にするフラグ文字列の配列です（`"telegram.*"` や `"*"` などのワイルドカードをサポートします）。
- `otel.enabled`: OpenTelemetry エクスポートパイプラインを有効にします（デフォルト: `false`）。完全な設定、シグナルカタログ、プライバシーモデルについては、[OpenTelemetry エクスポート](/ja-JP/gateway/opentelemetry)を参照してください。
- `otel.endpoint`: OTel エクスポート用のコレクター URL です。
- `otel.tracesEndpoint` / `otel.metricsEndpoint` / `otel.logsEndpoint`: シグナル固有のオプションの OTLP エンドポイントです。設定されている場合、そのシグナルに限り `otel.endpoint` を上書きします。
- `otel.protocol`: `"http/protobuf"`（デフォルト）または `"grpc"`。
- `otel.headers`: OTel エクスポートリクエストとともに送信される追加の HTTP/gRPC メタデータヘッダーです。
- `otel.serviceName`: リソース属性に使用するサービス名です。
- `otel.traces` / `otel.metrics` / `otel.logs`: トレース、メトリクス、ログのエクスポートを有効にします。
- `otel.logsExporter`: ログのエクスポート先です。`"otlp"`（デフォルト）、標準出力の1行につき1つの JSON オブジェクトを出力する `"stdout"`、または `"both"` を指定します。
- `otel.sampleRate`: トレースのサンプリング率 `0`～`1`。
- `otel.flushIntervalMs`: 定期的にテレメトリをフラッシュする間隔（ミリ秒単位）です。
- `otel.captureContent`: OTEL スパン属性への生コンテンツの取得をオプトインで有効にします。デフォルトでは無効です。ブール値の `true` はシステム以外のメッセージ／ツールコンテンツを取得します。オブジェクト形式では、`inputMessages`、`outputMessages`、`toolInputs`、`toolOutputs`、`systemPrompt`、`toolDefinitions` を個別に有効化できます。
- `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`: 最新の実験的な GenAI 推論スパン形式を有効にする環境トグルです。`{gen_ai.operation.name} {gen_ai.request.model}` スパン名、`CLIENT` スパン種別、従来の `gen_ai.system` に代わる `gen_ai.provider.name` が含まれます。デフォルトでは、互換性のためにスパンは `openclaw.model.call` と `gen_ai.system` を維持し、GenAI メトリクスは上限付きのセマンティック属性を使用します。
- `OPENCLAW_OTEL_PRELOADED=1`: グローバルな OpenTelemetry SDK がすでに登録されているホスト向けの環境トグルです。この場合、OpenClaw は診断リスナーをアクティブに保ちながら、Plugin が所有する SDK の起動／シャットダウンを省略します。
- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`、`OTEL_EXPORTER_OTLP_METRICS_ENDPOINT`、`OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`: 対応する設定キーが未設定の場合に使用される、シグナル固有のエンドポイント環境変数です。
- `cacheTrace.enabled`: 組み込み実行用のキャッシュトレーススナップショットをログに記録します（デフォルト: `false`）。
- `cacheTrace.filePath`: キャッシュトレース JSONL の出力パスです（デフォルト: `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`）。
- `cacheTrace.includeMessages` / `includePrompt` / `includeSystem`: キャッシュトレース出力に含める内容を制御します（すべてデフォルト: `true`）。

---

## 更新

```json5
{
  update: {
    channel: "stable", // stable | extended-stable | beta | dev
    checkOnStart: true,

    auto: {
      enabled: false,
    },
  },
}
```

- `channel`: リリースチャンネル（`"stable"`、`"extended-stable"`、`"beta"`、または `"dev"`）です。Extended-stable はパッケージ専用です。フォアグラウンドコマンドがインストールを担い、Gateway は読み取り専用の更新通知を出力する場合があります。
- `checkOnStart`: Gateway の起動時に npm の更新を確認します（デフォルト: `true`）。保存済みの Extended-stable 選択では、同じ読み取り専用の通知と24時間間隔の通知スケジュールが使用されます。
- `auto.enabled`: stable および beta のパッケージインストールに対するバックグラウンド自動更新を有効にします（デフォルト: `false`）。Extended-stable は自動的に適用されることはありません。

---

## ACP

```json5
{
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    fallbacks: ["acpx-secondary"],
    defaultAgent: "main",
    allowedAgents: ["main", "ops"],
    stream: {
      repeatSuppression: true,
      deliveryMode: "live", // live | final_only
    },
  },
}
```

- `enabled`: グローバルな ACP 機能ゲートです（デフォルト: `true`。ACP のディスパッチとスポーン機能を非表示にするには `false` に設定します）。
- `dispatch.enabled`: ACP セッションのターンディスパッチ用の独立したゲートです（デフォルト: `true`）。実行をブロックしつつ ACP コマンドを利用可能な状態に保つには、`false` に設定します。
- `backend`: デフォルトの ACP ランタイムバックエンド ID です（登録済みの ACP ランタイム Plugin と一致する必要があります）。
  まずバックエンド Plugin をインストールしてください。`plugins.allow` が設定されている場合は、バックエンド Plugin の ID（例: `acpx`）を含める必要があります。含めない場合、ACP バックエンドは読み込まれません。
- `fallbacks`: プライマリバックエンドが何らかの出力を生成する前に、一時的と思われるエラー（利用不可、レート制限、クォータ枯渇、過負荷）で早期に失敗した場合に試行される、フォールバック ACP バックエンド ID の順序付きリストです。各エントリは、登録済みの ACP ランタイム Plugin バックエンドと一致する必要があります。
- `defaultAgent`: スポーンで明示的なターゲットを指定しない場合に使用する、フォールバック ACP ターゲットエージェント ID です。
- `allowedAgents`: ACP ランタイムセッションで許可されるエージェント ID の許可リストです。空の場合、追加の制限はありません。
- `stream.repeatSuppression`: ターンごとに繰り返されるステータス／ツール行を抑制します（デフォルト: `true`）。
- `stream.deliveryMode`: `"live"` は逐次ストリーミングし、`"final_only"` はターンの終端イベントまでバッファリングします。
- `stream.tagVisibility`: ストリーミングイベントのタグ名とブール値の可視性オーバーライドを対応付けるレコードです。
- `runtime.installCommand`: ACP ランタイム環境のブートストラップ時に実行する、オプションのインストールコマンドです。

---

## ウィザード

CLI のガイド付きセットアップフロー（`onboard`、`configure`、`doctor`）の動作とメタデータ:

```json5
{
  wizard: {
    accessMode: "full",
    appRecommendations: true,
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
    securityAcknowledgedAt: "2026-01-01T00:00:00.000Z",
  },
}
```

- `wizard.accessMode`: ガイド付きオンボーディングの開始時に選択する検出への同意。`"full"`（推奨）では、セットアップが AI アプリ、キー、ローカルランタイムを自動的に検索できます。`"guarded"`では、検索前にセットアップが一度確認し、代わりに手動構成を提示します。

- `wizard.appRecommendations`のデフォルトは`true`です。`false`に設定すると、ガイド付きまたはクラシックオンボーディング中のインストール済みアプリケーションの推奨を無効にし、Gateway の`device.apps`アクセスをブロックします。Node ホストがコマンドを公開するには、別途用意されたデフォルトで無効のインストール済みアプリ共有フラグも有効にする必要があります。

---

## ID

[エージェントのデフォルト](/ja-JP/gateway/config-agents#agent-defaults)にある`agents.entries`の ID フィールドを参照してください。

---

## ブリッジ（レガシー、削除済み）

現在のビルドには TCP ブリッジが含まれなくなりました。Node は Gateway WebSocket 経由で接続します。`bridge.*`キーは構成スキーマに含まれなくなりました（削除するまで検証に失敗します。`openclaw doctor --fix`で不明なキーを除去できます）。

<Accordion title="レガシーブリッジの構成（履歴参照用）">

```json
{
  "bridge": {
    "enabled": true,
    "port": 18790,
    "bind": "tailnet",
    "tls": {
      "enabled": true,
      "autoGenerate": true
    }
  }
}
```

</Accordion>

---

## Cron

```json5
{
  cron: {
    enabled: true,
    webhook: "https://example.invalid/legacy", // deprecated fallback for stored notify:true jobs
    webhookToken: "replace-with-dedicated-token", // optional bearer token for outbound webhook auth
    sessionRetention: "24h", // duration string or false
  },
}
```

- `sessionRetention`: 完了した分離 Cron 実行セッションについて、SQLite のセッション行を削除するまで保持する期間。アーカイブされた削除済み Cron トランスクリプトのクリーンアップも制御します。デフォルト: `24h`。無効にするには`false`を設定します。
- 実行履歴では、ジョブごとに最新の 2000 件の終端行が自動的に保持されます。失われた行には、引き続き 24 時間のクリーンアップ期間が適用されます。
- `webhookToken`: Cron Webhook の POST 配信（`delivery.mode = "webhook"`）に使用されるベアラートークン。省略した場合、認証ヘッダーは送信されません。
- `webhook`: `notify: true`が残っている保存済みジョブを移行するために`openclaw doctor --fix`が使用する、非推奨のレガシーフォールバック Webhook URL（http/https）。ランタイム配信ではジョブ単位の`delivery.mode="webhook"`と`delivery.to`を使用し、アナウンス配信を維持する場合は`delivery.completionDestination`を使用します。

### `cron.failureAlert`

```json5
{
  cron: {
    failureAlert: {
      enabled: false,
      after: 3,
      cooldownMs: 3600000,
      includeSkipped: false,
      mode: "announce",
      accountId: "main",
    },
  },
}
```

- `enabled`: Cron ジョブの失敗アラートを有効にします（デフォルト: `false`）。
- `after`: アラートが発生するまでの連続失敗回数（正の整数、最小値: `1`）。
- `cooldownMs`: 同じジョブについて繰り返しアラートを送る際の最小間隔（ミリ秒、非負整数）。
- `includeSkipped`: 連続してスキップされた実行をアラートしきい値に算入します（デフォルト: `false`）。スキップされた実行は個別に追跡され、実行エラーのバックオフには影響しません。
- `mode`: 配信モード - `"announce"`はチャンネルメッセージで送信し、`"webhook"`は構成済みの Webhook に投稿します。
- `accountId`: アラート配信の範囲を限定するための、任意のアカウント ID またはチャンネル ID。

### `cron.failureDestination`

```json5
{
  cron: {
    failureDestination: {
      mode: "announce",
      channel: "last",
      to: "channel:C1234567890",
      accountId: "main",
    },
  },
}
```

- すべてのジョブに適用される Cron 失敗通知のデフォルト送信先。
- `mode`: `"announce"`または`"webhook"`。十分な送信先データがある場合、デフォルトは`"announce"`です。
- `channel`: アナウンス配信用のチャンネル上書き。`"last"`は、最後に確認された配信チャンネルを再利用します。
- `to`: 明示的なアナウンス送信先または Webhook URL。Webhook モードでは必須です。
- `accountId`: 配信用の任意のアカウント上書き。
- ジョブ単位の`delivery.failureDestination`は、このグローバルデフォルトを上書きします。
- グローバルにもジョブ単位にも失敗時の送信先が設定されていない場合、すでに`announce`で配信しているジョブは、失敗時にその主要なアナウンス送信先へフォールバックします。
- `delivery.failureDestination`は、ジョブの主要な`delivery.mode`が`"webhook"`でない限り、`sessionTarget="isolated"`ジョブでのみサポートされます。

[Cron ジョブ](/ja-JP/automation/cron-jobs)を参照してください。分離された Cron 実行は[バックグラウンドタスク](/ja-JP/automation/tasks)として追跡されます。

## メディアモデルのテンプレート変数

`tools.media.models[].args`内で展開されるテンプレートプレースホルダー:

| 変数                        | 説明                                              |
| --------------------------- | ------------------------------------------------- |
| `{{Body}}`                  | 受信メッセージ本文全体                            |
| `{{RawBody}}`               | 生の本文（履歴や送信者のラッパーなし）            |
| `{{BodyStripped}}`          | グループメンションを除去した本文                  |
| `{{From}}`                  | 送信者識別子                                      |
| `{{To}}`                    | 送信先識別子                                      |
| `{{MessageSid}}`            | チャンネルメッセージ ID                           |
| `{{SessionId}}`             | 現在のセッション UUID                             |
| `{{IsNewSession}}`          | 新しいセッションが作成された場合は`"true"`       |
| `{{AttachmentUrl}}`         | 現在の添付ファイルの URL またはプロバイダー参照   |
| `{{AttachmentPath}}`        | 現在の添付ファイルのローカルパス                  |
| `{{AttachmentContentType}}` | 現在の添付ファイルの MIME コンテンツタイプ        |
| `{{AttachmentDir}}`         | `AttachmentPath`を含むディレクトリ                 |
| `{{AttachmentIndex}}`       | 0 始まりのソースファクトインデックス              |
| `{{Transcript}}`            | 音声トランスクリプト                              |
| `{{Prompt}}`                | CLI エントリ用に解決されたメディアプロンプト      |
| `{{MaxChars}}`              | CLI エントリ用に解決された最大出力文字数          |
| `{{ChatType}}`              | `"direct"`または`"group"`                  |
| `{{GroupSubject}}`          | グループ件名（ベストエフォート）                  |
| `{{GroupMembers}}`          | グループメンバーのプレビュー（ベストエフォート）  |
| `{{SenderName}}`            | 送信者の表示名（ベストエフォート）                |
| `{{SenderE164}}`            | 送信者の電話番号（ベストエフォート）              |
| `{{Provider}}`              | プロバイダーのヒント（whatsapp、telegram、discord など） |

レガシーの`{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}`、`{{MediaDir}}`
という名前は、Plugin SDK の互換期間中は引き続き利用できますが、
非推奨です。新しい構成では`Attachment*`変数を使用してください。

---

## 構成のインクルード（`$include`）

構成を複数のファイルに分割します:

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

**マージ動作:**

- 単一ファイル: それを含むオブジェクトを置き換えます。
- ファイルの配列: 順番にディープマージされます（後のファイルが前のファイルを上書きします）。
- 兄弟キー: インクルード後にマージされます（インクルードされた値を上書きします）。
- ネストされたインクルード: 最大 10 階層まで。
- パス: インクルード元ファイルを基準に解決されますが、最上位の構成ディレクトリ（`openclaw.json`の`dirname`）内に収まる必要があります。絶対パスまたは`../`形式は、解決後もこの境界内に収まる場合にのみ許可されます。構成ディレクトリ外のルートを追加で許可するには、`OPENCLAW_INCLUDE_ROOTS`（絶対パス）を設定します。
- 制限: パスに null バイトを含めることはできず、解決前後のどちらでも厳密に 4096 文字未満である必要があります。インクルードされる各ファイルの上限は 2 MB です。
- 単一ファイルのインクルードを参照する最上位セクションを 1 つだけ変更する OpenClaw 所有の書き込みは、そのインクルード先ファイルに直接書き込まれます。たとえば、`plugins install`は`plugins.json5`内の`plugins: { $include: "./plugins.json5" }`を更新し、`openclaw.json`はそのまま維持します。
- ルートインクルード、インクルード配列、および兄弟キーによる上書きを含むインクルードは、OpenClaw 所有の書き込みに対して読み取り専用です。このような書き込みは、構成をフラット化する代わりに安全側で失敗します。
- エラー: ファイルの欠落、解析エラー、循環インクルード、無効なパス形式、長さ超過に対して明確なメッセージを表示します。

---

## 関連項目

- [構成](/ja-JP/gateway/configuration)
- [構成例](/ja-JP/gateway/configuration-examples)
- [Doctor](/ja-JP/gateway/doctor)
