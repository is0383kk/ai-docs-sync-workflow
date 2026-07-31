---
read_when:
    - '`tools.*` ポリシー、許可リスト、または実験的機能の設定'
    - カスタムプロバイダーの登録またはベース URL の上書き
    - OpenAI 互換のセルフホスト型エンドポイントのセットアップ
sidebarTitle: Tools and custom providers
summary: ツール設定（ポリシー、実験的な切り替え、プロバイダー対応ツール）およびカスタムプロバイダー／ベース URL のセットアップ
title: 設定 — ツールとカスタムプロバイダー
x-i18n:
    generated_at: "2026-07-26T09:02:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2010a2e48e8f4c8d0049e5c707bb8286e291a92312baac94301a7b5a674583c1
    source_path: gateway/config-tools.md
    workflow: 16
---

`tools.*` 設定キーとカスタムプロバイダー / ベース URL のセットアップ。エージェント、チャンネル、その他のトップレベル設定キーについては、[設定リファレンス](/ja-JP/gateway/configuration-reference)を参照してください。

## ツール

### ツールプロファイル

`tools.profile` は、`tools.allow`/`tools.deny` より前に基本許可リストを設定します。

<Note>
ローカルオンボーディングでは、未設定の場合、新しいローカル設定のデフォルトが `tools.profile: "coding"` になります（既存の明示的なプロファイルは保持されます）。
</Note>

| プロファイル     | 含まれるもの                                                                                                                                                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | `session_status` のみ                                                                                                                                                                                                                                   |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`, `image`, `image_generate`, `music_generate`, `video_generate`                |
| `messaging` | `group:messaging`, `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `ask_user` |
| `full`      | 制限なし（未設定の場合と同じ）                                                                                                                                                                                                                          |

`coding` と `messaging` は、`bundle-mcp`（設定済み MCP サーバー）も暗黙的に許可します。

### ツールグループ

| グループ              | ツール                                                                                                                                                                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`, `process`, `code_execution`（`bash` は `exec` のエイリアスとして受け入れられます）                                                                                                                                                                        |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `spawn_task`, `dismiss_task` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`, `screen`, `terminal`, `canvas`, `show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`, `cron`, `gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`, `computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`, `image_generate`, `music_generate`, `video_generate`, `tts`                                                                                                                                                                                   |
| `group:openclaw`   | 上記の組み込みツールのうち、`read`/`write`/`edit`/`apply_patch`/`exec`/`process`/`canvas` を除くすべて（Plugin ツールを除外）                                                                                                                                  |
| `group:plugins`    | 読み込まれた Plugin が所有するツール（`bundle-mcp` を通じて公開される設定済み MCP サーバーを含む）                                                                                                                                                           |

`spawn_task` を使用すると、コーディングエージェントは、確認済みの後続作業を開始せずに提案できます。Control UI はタイトルと概要を操作可能なチップとして表示し、Gateway バックエンドの TUI は同等の対話型プロンプトを表示します。いずれかを承認すると、新しい管理対象ワークツリーセッションが作成され、現在のターンを続行しながら、完全なプロンプトがそこへ送信されます。`dismiss_task` は、`spawn_task` から返された一時的な `task_id` を使用して、保留中の提案を取り下げます。

これらのツールは、開始元のオペレーター画面が Gateway のタスク提案イベントを受信して処理できる場合にのみ提供されます。チャンネルセッションおよびローカル / 組み込み TUI セッションはこれらを受信しません。チャンネルトランスポートでこのフローを安全に公開するには、移植可能な型付きタスクアクションが必要です。提案はプロセスローカルであり、Gateway の再起動時に消失します。両方のツールは `coding` プロファイルと `group:sessions` に残るため、画面が対応していれば、通常の `tools.allow` および `tools.deny` ポリシーによって自動的に設定されます。

### サンドボックスのツールポリシー内の MCP および Plugin ツール

設定済み MCP サーバーは、`bundle-mcp` Plugin ID 配下で Plugin 所有ツールとして公開されます。通常のツールプロファイルで許可できますが、`tools.sandbox.tools` はサンドボックス化されたセッション用の追加ゲートです。サンドボックスモードが `"all"` または `"non-main"` の場合、MCP / Plugin ツールを表示するには、サンドボックスのツール許可リストに次のいずれかのエントリを含めます。

- `mcp.servers` の OpenClaw 管理 MCP サーバーには `bundle-mcp`
- 特定のネイティブ Plugin にはその Plugin ID
- 読み込まれたすべての Plugin 所有ツールには `group:plugins`
- 1 つのサーバーのみを使用する場合は、正確な MCP サーバーツール名、または `outlook__send_mail` や `outlook__*` などのサーバー glob

サーバー glob では、生の `mcp.servers` キーと必ずしも同じではない、プロバイダーで安全に使用できる MCP サーバープレフィックスを使用します。`[A-Za-z0-9_-]` 以外の文字は `-` になり、文字で始まらない名前には `mcp-` プレフィックスが付き、長いプレフィックスや重複するプレフィックスは切り詰められたりサフィックスが付いたりする場合があります。たとえば、`mcp.servers["Outlook Graph"]` では `outlook-graph__*` のような glob を使用します。

```json5
{
  agents: { defaults: { sandbox: { mode: "all" } } },
  mcp: {
    servers: {
      outlook: { command: "node", args: ["./outlook-mcp.js"] },
    },
  },
  tools: {
    sandbox: {
      tools: {
        alsoAllow: ["web_search", "web_fetch", "memory_search", "memory_get", "bundle-mcp"],
      },
    },
  },
}
```

このサンドボックス層のエントリがなくても、MCP サーバーは正常に読み込まれる場合がありますが、そのツールはプロバイダーへのリクエスト前に除外されます。`mcp.servers` 内の OpenClaw 管理サーバーについて、この構成を検出するには `openclaw doctor` を使用します。バンドルされた Plugin マニフェストまたは Claude `.mcp.json` から読み込まれた MCP サーバーにも同じサンドボックスゲートが適用されますが、この診断ではまだそれらのソースは列挙されません。サンドボックス化されたターンでツールが表示されなくなった場合は、同じ許可リストエントリを使用してください。

### `tools.codeMode`

`tools.codeMode` は、汎用の OpenClaw コードモード画面を有効にします。ツールを使用する実行で有効にすると、通常の OpenClaw ツールはサンドボックス内の `tools.*` カタログブリッジの背後に移動し、MCP ツールは生成された `MCP` 名前空間を通じて利用できます。通常、モデルには `exec` と `wait` が表示されます。構造化された結果を JSON のみのブリッジで渡せない `computer` などのツールは、直接提供されたままです。

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

短縮形も使用できます。

```json5
{
  tools: { codeMode: true },
}
```

コードモードでは、MCP 宣言は読み取り専用の仮想 API ファイル画面を通じて公開されます。ゲストコードは、`MCP.<server>.<tool>()` を呼び出す前に、`API.list("mcp")` および `API.read("mcp/<server>.d.ts")` を呼び出して TypeScript 形式のシグネチャを確認できます。ランタイム契約、制限、デバッグ手順については、[コードモード](/tools/code-mode)を参照してください。

### `tools.allow` / `tools.deny`

グローバルツールの許可 / 拒否ポリシー（拒否が優先）。大文字と小文字を区別せず、`*` ワイルドカードをサポートします。Docker サンドボックスが無効でも適用されます。

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

`write` と `apply_patch` は別々のツール ID です。`allow: ["write"]` は互換性のあるモデルに対して `apply_patch` も有効にしますが、`deny: ["write"]` は `apply_patch` を拒否しません。すべてのファイル変更をブロックするには、`group:fs` を拒否するか、変更を行う各ツールを明示的に列挙します。

```json5
{
  tools: { deny: ["write", "edit", "apply_patch"] },
}
```

<Note>
`allow` と `alsoAllow` を同じスコープ（`tools`、`tools.byProvider.<id>`、`agents.entries.*.tools`）に同時に設定することはできません。設定検証によって拒否されます。`alsoAllow` のエントリを `allow` に統合するか、`allow` を削除して、代わりに `profile` + `alsoAllow` を使用してください。
</Note>

### `tools.byProvider`

特定のプロバイダーまたはモデル向けにツールをさらに制限します。順序: 基本プロファイル → プロバイダープロファイル → 許可 / 拒否。

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.toolsBySender`

現在のターンを開始したリクエスターに対してツールを制限します。これはチャネルアクセス制御に加える多層防御です。送信者の値はメッセージ本文ではなく、チャネルアダプターから取得する必要があります。モデルプロンプト内のその他のコンテンツを認証するものではありません。[リクエスター単位の制御とプロンプトコンテキスト](/ja-JP/gateway/security#requester-scoped-controls-and-prompt-context)を参照してください。

```json5
{
  tools: {
    toolsBySender: {
      "channel:discord:1234567890123": { alsoAllow: ["group:fs"] },
      "id:guest-user-id": { deny: ["group:runtime", "group:fs"] },
      "*": { deny: ["exec", "process", "write", "edit", "apply_patch"] },
    },
  },
}
```

キーには明示的なプレフィックスを使用します：`channel:<channelId>:<senderId>`、`id:<senderId>`、`e164:<phone>`、`username:<handle>`、`name:<displayName>`、または `"*"`。チャネル ID は正規の OpenClaw ID です。`teams` などのエイリアスは `msteams` に正規化されます。プレフィックスのない従来のキーは、`id:` としてのみ受け入れられます。照合順序は、チャネル＋ID、ID、e164、ユーザー名、名前、ワイルドカードの順です。

エージェント単位の `agents.entries.*.tools.toolsBySender` は、一致した場合、`{}` ポリシーが空でもグローバルな送信者の一致を上書きします。

### `tools.elevated`

サンドボックス外での昇格された exec アクセスを制御します。

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- エージェント単位のオーバーライド（`agents.entries.*.tools.elevated`）では、制限をさらに厳しくすることしかできません。
- `/elevated on|off|ask|full` はセッション単位で状態を保存します。インラインディレクティブは単一のメッセージに適用されます。
- 昇格された `exec` はサンドボックスを迂回し、設定されたエスケープパス（デフォルトでは `gateway`、exec ターゲットが `node` の場合は `node`）を使用します。

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      approvalRunningNoticeMs: 10000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      commandHighlighting: false,
      applyPatch: {
        enabled: true,
        allowModels: ["gpt-5.6-sol"],
      },
    },
  },
}
```

表示されている値は、`applyPatch.allowModels` を除いてデフォルトです（デフォルトでは空または未設定であり、互換性のある任意のモデルが `apply_patch` を使用できることを意味します）。`approvalRunningNoticeMs` は、承認に基づく exec が長時間実行されたときに実行中の通知を送信します。`0` にすると無効になります。

### `tools.loopDetection`

ツールループの安全性チェックは**デフォルトで無効**です。検出を有効にするには `enabled: true` を設定します。設定は `tools.loopDetection` でグローバルに定義でき、`agents.entries.*.tools.loopDetection` でエージェント単位に上書きできます。

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
    },
  },
}
```

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // または BRAVE_API_KEY 環境変数（Brave プロバイダー）
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // 省略可能。自動検出する場合は省略
        maxChars: 20000,
        maxCharsCap: 20000,
        maxResponseBytes: 750000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

表示されている値は、`provider` と `userAgent` を除いてデフォルトです。`maxResponseBytes` は 32000～10000000 の範囲に制限されます。`maxChars` は `maxCharsCap` に制限されます（より大きなレスポンスを許可するには `maxCharsCap` を増やします）。

### `tools.media`

受信メディアの理解（画像／音声／動画）を設定します。

```json5
{
  tools: {
    media: {
      concurrency: 2,
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          capabilities: ["audio"],
        },
        { provider: "ollama", model: "gemma4:26b", capabilities: ["image"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["video"] },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-mini-transcribe" },
      image: { enabled: true, preferredModel: "ollama/gemma4:26b" },
      video: { enabled: true },
    },
  },
}
```

`tools.media.models` が唯一の設定済みモデルリストです。各エントリは、処理する機能を宣言します。省略可能な `preferredModel` セレクターは、`provider/model`、モデル ID、プロバイダーのデフォルトエントリを表す `provider:<id>`、または `cli:command` を受け付けます。一致するエントリは、その機能のフォールバック順序の先頭に移動します。機能単位のプロンプト、制限、リクエスト設定、スコープ、添付ファイルポリシー、音声文字起こしのエコーには、設定済みモデルと自動検出モデルのどちらでもデフォルトが適用されます。モデルエントリでは、モデル固有のフィールドを上書きできます。

<AccordionGroup>
  <Accordion title="メディアモデルのエントリフィールド">
    **プロバイダーエントリ**（`type: "provider"` または省略）：

    - `provider`：API プロバイダー ID（`openai`、`anthropic`、`google`／`gemini`、`groq` など）
    - `model`：モデル ID のオーバーライド
    - `profile`／`preferredProfile`：`auth-profiles.json` プロファイルの選択

    **CLI エントリ**（`type: "cli"`）：

    - `command`：実行する実行ファイル
    - `args`：テンプレート化された引数（`{{AttachmentPath}}`、`{{AttachmentUrl}}`、`{{AttachmentContentType}}`、`{{AttachmentDir}}`、`{{AttachmentIndex}}`、`{{Prompt}}`、`{{MaxChars}}` などをサポート。`openclaw doctor --fix` は非推奨の `{input}` プレースホルダーを `{{AttachmentPath}}` に移行します）。旧 `{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}`、`{{MediaDir}}` エイリアスも互換期間中は引き続き使用できますが、非推奨です。

    **共通フィールド：**

    - `capabilities`：`image`、`audio`、`video` のうち 1 つ以上を含むリスト。
    - `prompt`、`maxChars`、`maxBytes`、`timeoutSeconds`、`language`：エントリ単位のオーバーライド。
    - 一致する画像モデルの `timeoutSeconds` エントリは、エージェントが明示的な `image` ツールを呼び出す場合にも適用されます。画像理解では、このタイムアウトはリクエスト自体に適用され、先行する準備作業によって短縮されることはありません。
    - 失敗した場合は次のエントリにフォールバックします。

    プロバイダー認証は標準の順序に従います：`auth-profiles.json` → 環境変数 → `models.providers.*.apiKey`。

  </Accordion>
</AccordionGroup>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

セッションツール（`sessions_list`、`sessions_history`、`sessions_send`）の対象にできるセッションを制御します。

デフォルト：`tree`（現在のセッション＋そこから生成されたサブエージェントなどのセッション、および同じエージェントのアンビエントに監視されているグループセッション）。

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

<AccordionGroup>
  <Accordion title="可視性スコープ">
    - `self`：現在のセッションキーのみ。
    - `tree`：現在のセッション＋現在のセッションから生成されたセッション（サブエージェント）。読み取り操作では、現在のセッションがアンビエントなグループ認識を通じて監視している、同じエージェントのグループセッションも含まれます。
    - `agent`：現在のエージェント ID に属する任意のセッション（同じエージェント ID で送信者単位のセッションを実行している場合、他のユーザーを含むことがあります）。
    - `all`：任意のセッション。エージェントをまたぐ対象指定には、引き続き `tools.agentToAgent` が必要です。
    - サンドボックスによる制限：現在のセッションがサンドボックス化され、`agents.defaults.sandbox.sessionToolsVisibility="spawned"`（デフォルト）の場合、`tools.sessions.visibility="all"` であっても可視性は強制的に `tree` になります。
    - `all` でない場合、`sessions_list` には、有効なモードと、現在のスコープ外にある一部のセッションが省略される可能性があるという警告を説明する簡潔な `visibility` フィールドが含まれます。

  </Accordion>
</AccordionGroup>

デフォルトの `session.dmScope: "main"` では、グループ内で人間が活動すると、その同じエージェントのグループセッションがエージェントのメインセッションからアンビエントに見えるようになります。マルチユーザー構成では、`"main"` によって 1 つの DM セッションもユーザー間で共有されるため、そこにルーティングされた各ユーザーは、セッションメモリの `memory_search` を介する場合も含め、アンビエントに監視されているグループから読み取ることができます。DM を分離するにはピア単位の `dmScope` を使用するか、`tools.sessions.visibility: "self"` を設定して、アンビエントに監視されているセッションからの読み取りを無効にします。

### `tools.sessions_spawn`

`sessions_spawn` のインライン添付ファイルサポートを制御します。

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // オプトイン：インラインファイル添付を許可するには true に設定
        maxTotalBytes: 5242880, // 全ファイル合計で 5 MB
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 ファイルあたり 1 MB
        retainOnSessionKeep: false, // cleanup="keep" の場合に添付ファイルを保持
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="添付ファイルに関する注意事項">
    - 添付ファイルには `enabled: true` が必要です。
    - サブエージェントの添付ファイルは、子ワークスペース内の `.openclaw/attachments/<uuid>/` に `.manifest.json` とともに実体化されます。
    - ACP の添付ファイルは画像のみで、同じファイル数、ファイル単位のバイト数、合計バイト数の制限を通過した後、ACP ランタイムにインラインで転送されます。
    - 添付ファイルの内容は、トランスクリプトの永続化時に自動的に秘匿化されます。
    - Base64 入力は、厳格なアルファベット／パディングチェックとデコード前のサイズガードによって検証されます。
    - サブエージェントの添付ファイル権限は、ディレクトリでは `0700`、ファイルでは `0600` です。
    - サブエージェントのクリーンアップは `cleanup` ポリシーに従います。`delete` は添付ファイルを常に削除し、`keep` は `retainOnSessionKeep: true` の場合にのみ保持します。

  </Accordion>
</AccordionGroup>

<a id="toolsexperimental"></a>

### `tools.experimental`

実験的な組み込みツールのフラグです。strict-agentic GPT-5 の自動有効化ルールが適用されない限り、デフォルトでは無効です。

```json5
{
  tools: {
    experimental: {
      planTool: true, // 実験的な update_plan を有効化
    },
  },
}
```

- `planTool`：複雑な複数ステップ作業を追跡するための構造化された `update_plan` ツールを有効にします。
- デフォルト：GPT-5 ファミリーのモデル ID に対する `openai` プロバイダーの実行で、`agents.defaults.embeddedAgent.executionContract`（またはエージェント単位のオーバーライド）が `"strict-agentic"` に設定されていない限り、`false` です（Codex の認証／モデルルーティングは `openai` プロバイダー配下にあるため、OpenAI Codex CLI の実行も対象です）。このスコープ外でツールを強制的に有効にするには `true` を設定し、strict-agentic GPT-5 の実行でも無効のままにするには `false` を設定します。
- 有効にすると、システムプロンプトにも使用ガイダンスが追加されるため、モデルは本格的な作業にのみ使用し、`in_progress` のステップを最大 1 つに保ちます。

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        announceTimeoutMs: 120000,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`: 生成されるサブエージェントのデフォルトモデル。省略した場合、サブエージェントは呼び出し元のモデルを継承します。
- `allowAgents`: リクエスト元のエージェントが独自の `subagents.allowAgents` を設定していない場合に、`sessions_spawn` で使用する、設定済みターゲットエージェント ID のデフォルト許可リスト（`["*"]` = 設定済みの任意のターゲット、デフォルト: 同じエージェントのみ）。エージェント設定が削除された古いエントリは `sessions_spawn` によって拒否され、`agents_list` から除外されます。クリーンアップするには `openclaw doctor --fix` を実行してください。
- `maxConcurrent`: 同時に実行できるサブエージェントの最大数。デフォルト: `8`。
- `runTimeoutSeconds`: 呼び出し元が独自の上書き値を渡さない場合の `sessions_spawn` のタイムアウト（秒）。デフォルト: `0`（タイムアウトなし）。上記の `900` は一般的なオプトイン値であり、組み込みのデフォルトではありません。
- `announceTimeoutMs`: Gateway の `agent` 通知配信試行に対する呼び出しごとのタイムアウト（ミリ秒）。デフォルト: `120000`。一時的な再試行により、通知の合計待機時間が設定された 1 回分のタイムアウトより長くなる場合があります。
- `archiveAfterMinutes`: サブエージェントセッションの完了後、自動的にアーカイブされるまでの時間（分）。デフォルト: `60`。`0` を指定すると自動アーカイブが無効になります。
- サブエージェントごとのツールポリシー: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`。

---

## カスタムプロバイダーとベース URL

プロバイダー Plugin は独自のモデルカタログ行を公開します。カスタムプロバイダーは、設定の `models.providers` または `~/.openclaw/agents/<agentId>/agent/models.json` を使用して追加します。

カスタムまたはローカルプロバイダーの `baseUrl` を設定することは、モデル HTTP リクエストに対する限定的なネットワーク信頼の決定でもあります。OpenClaw は、別の設定オプションを追加したり他のプライベートオリジンを信頼したりすることなく、その `scheme://host:port` の正確なオリジンを保護されたフェッチパスで許可します。

```json5
{
  models: {
    mode: "merge", // マージ（デフォルト）| 置換
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai | など
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="認証とマージの優先順位">
    - カスタム認証が必要な場合は、`authHeader: true` + `headers` を使用します。
    - `OPENCLAW_AGENT_DIR` でエージェント設定のルートを上書きします。
    - 一致するプロバイダー ID のマージ優先順位:
      - 空でないエージェントの `models.json` `baseUrl` 値が優先されます。
      - 空でないエージェントの `apiKey` 値は、現在の設定または認証プロファイルのコンテキストで、そのプロバイダーが SecretRef によって管理されていない場合にのみ優先されます。
      - SecretRef で管理されるプロバイダーの `apiKey` 値は、解決済みのシークレットを永続化する代わりに、ソースマーカー（環境変数参照の場合は `ENV_VAR_NAME`、ファイルまたは実行参照の場合は `secretref-managed`）から更新されます。
      - SecretRef で管理されるプロバイダーのヘッダー値は、ソースマーカー（環境変数参照の場合は `secretref-env:ENV_VAR_NAME`、ファイルまたは実行参照の場合は `secretref-managed`）から更新されます。
      - エージェントの `apiKey`/`baseUrl` が空または欠落している場合は、設定の `models.providers` にフォールバックします。
      - 一致するモデルの `contextWindow`/`maxTokens`: 明示的な設定値が存在し、有効（正の有限数）である場合はその値が優先されます。それ以外の場合は、暗黙的または生成されたカタログ値が使用されます。
      - 一致するモデルの `contextTokens` も同じ「明示値があれば優先し、それ以外は暗黙値」というルールに従います。ネイティブモデルのメタデータを変更せずに有効なコンテキストを制限するには、これを使用します。
      - プロバイダー Plugin のカタログは、エージェントの Plugin 状態内に生成された Plugin 所有のカタログシャードとして保存されます。
      - 設定で `models.json` を完全に書き換え、Plugin 所有のカタログシャードをマージしない場合は、`models.mode: "replace"` を使用します。
      - マーカーの永続化ではソースが正とされます。マーカーは解決済みのランタイムシークレット値からではなく、アクティブなソース設定のスナップショット（解決前）から書き込まれます。

  </Accordion>
</AccordionGroup>

### プロバイダーフィールドの詳細

<AccordionGroup>
  <Accordion title="最上位カタログ">
    - `models.mode`: プロバイダーカタログの動作（`merge` または `replace`）。
    - `models.providers`: プロバイダー ID をキーとするカスタムプロバイダーマップ。
      - 安全な編集: 追加更新には `openclaw config set models.providers.<id> '<json>' --strict-json --merge` または `openclaw config set models.providers.<id>.models '<json-array>' --strict-json --merge` を使用します。`config set` は、`--replace` を渡さない限り、破壊的な置換を拒否します。

  </Accordion>
  <Accordion title="プロバイダー接続と認証">
    - `models.providers.*.api`: リクエストアダプター（`openai-completions`、`openai-responses`、`openai-chatgpt-responses`、`anthropic-messages`、`google-generative-ai`、`google-vertex`、`github-copilot`、`bedrock-converse-stream`、`ollama`、`azure-openai-responses`）。MLX、vLLM、SGLang、およびほとんどの OpenAI 互換ローカルサーバーなど、セルフホストの `/v1/chat/completions` バックエンドには `openai-completions` を使用します。`baseUrl` があり、`api` がないカスタムプロバイダーは、デフォルトで `openai-completions` を使用します。バックエンドが `/v1/responses` をサポートする場合にのみ、`openai-responses` を設定してください。
    - `models.providers.*.apiKey`: プロバイダーの認証情報（SecretRef または環境変数置換を推奨）。
    - `models.providers.*.auth`: 認証方式（`api-key`、`token`、`oauth`、`aws-sdk`）。
    - `models.providers.*.contextWindow`: モデルエントリで `contextWindow` が設定されていない場合に、このプロバイダー配下のモデルに適用されるデフォルトのネイティブコンテキストウィンドウ。
    - `models.providers.*.contextTokens`: モデルエントリで `contextTokens` が設定されていない場合に、このプロバイダー配下のモデルに適用されるデフォルトの有効なランタイムコンテキスト上限。
    - `models.providers.*.maxTokens`: モデルエントリで `maxTokens` が設定されていない場合に、このプロバイダー配下のモデルに適用されるデフォルトの出力トークン上限。
    - `models.providers.*.timeoutSeconds`: 接続、ヘッダー、本文、およびリクエスト全体の中断処理を含む、プロバイダーごとの任意のモデル HTTP リクエストタイムアウト（秒）。
    - `models.providers.*.injectNumCtxForOpenAICompat`: Ollama + `openai-completions` の場合、リクエストに `options.num_ctx` を挿入します（デフォルト: `true`）。
    - `models.providers.*.authHeader`: 必要な場合、`Authorization` ヘッダーでの認証情報の送信を強制します。
    - `models.providers.*.baseUrl`: 上流 API のベース URL。
    - `models.providers.*.headers`: プロキシまたはテナントのルーティング用の追加静的ヘッダー。

  </Accordion>
  <Accordion title="リクエスト転送の上書き">
    `models.providers.*.request`: モデルプロバイダーの HTTP リクエストに対する転送の上書き。

    - `request.headers`: 追加ヘッダー（プロバイダーのデフォルトとマージ）。値には SecretRef を使用できます。
    - `request.auth`: 認証方式の上書き。モード: `"provider-default"`（プロバイダー組み込みの認証を使用）、`"authorization-bearer"`（`token` とともに使用）、`"header"`（`headerName`、`value`、および任意の `prefix` とともに使用）。
    - `request.proxy`: HTTP プロキシの上書き。モード: `"env-proxy"`（`HTTP_PROXY`/`HTTPS_PROXY` 環境変数を使用）、`"explicit-proxy"`（`url` とともに使用）。どちらのモードでも、任意の `tls` サブオブジェクトを指定できます。
    - `request.tls`: 直接接続用の TLS 上書き。フィールド: `ca`、`cert`、`key`、`passphrase`（すべて SecretRef を使用可能）、`serverName`、`insecureSkipVerify`。
    - `request.allowPrivateNetwork`: `true` の場合、プロバイダーの HTTP フェッチガードを通じて、プライベート、CGNAT、または同様の範囲へのモデルプロバイダー HTTP リクエストを許可します。カスタムまたはローカルプロバイダーのベース URL は、メタデータまたはリンクローカルのオリジンを除き、設定された正確なオリジンをすでに信頼します。これらのオリジンは明示的にオプトインしない限り、引き続きブロックされます。正確なオリジンの信頼をオプトアウトするには、これを `false` に設定します。WebSocket はヘッダーまたは TLS に同じ `request` を使用しますが、そのフェッチ SSRF ゲートは使用しません。デフォルト: `false`。

  </Accordion>
  <Accordion title="モデルカタログエントリ">
    - `models.providers.*.models`: 明示的なプロバイダーモデルカタログエントリ。
    - `models.providers.*.models.*.input`: モデルの入力モダリティ。テキスト専用モデルには `["text"]`、ネイティブの画像またはビジョンモデルには `["text", "image"]` を使用します。選択したモデルが画像対応としてマークされている場合にのみ、画像添付がエージェントのターンに挿入されます。
    - `models.providers.*.models.*.contextWindow`: ネイティブモデルのコンテキストウィンドウのメタデータ。このモデルでは、プロバイダーレベルの `contextWindow` よりも優先されます。
    - `models.providers.*.models.*.contextTokens`: 任意のランタイムコンテキスト上限。プロバイダーレベルの `contextTokens` よりも優先されます。モデルのネイティブな `contextWindow` よりも小さい有効コンテキスト予算を使用する場合に指定します。値が異なる場合、`openclaw models list` には両方の値が表示されます。

    #### カスタムプロバイダーの機能宣言

    プロバイダーカタログは、同梱モデルおよびカタログで既知のモデルルートに対する `compat` を所有します。これらのフラグを設定にコピーしないでください。設定された `api` と `baseUrl` が引き続きそのルートを識別する場合、OpenClaw はカタログ行を使用します。`openclaw doctor --fix` は一致するレガシー上書きを削除し、確認が必要な相違値を報告します。

    `compat` ブロックは、真にカスタムなプロバイダー、カスタムモデル、または別のエンドポイントにルーティングされるカタログモデルで引き続きサポートされます。そのエンドポイントに対して検証済みの機能のみを設定してください。

    | カスタムルートキー | ランタイム契約 |
    | --- | --- |
    | `supportsStore` | OpenAI の `store` リクエストフィールドを受け入れます。 |
    | `supportsPromptCacheKey` | OpenAI のプロンプトキャッシュまたはセッションアフィニティキーを受け入れます。 |
    | `supportsDeveloperRole` | `system` を必須とせず、`developer` メッセージを受け入れます。 |
    | `supportsReasoningEffort` | 推論強度の制御を受け入れます。 |
    | `supportsTemperature` | このモデルとアダプターで `temperature` を受け入れます。 |
    | `supportsUsageInStreaming` | ストリーミングレスポンスで使用量メタデータを出力します。 |
    | `supportsTools` | 構造化されたツールまたは関数呼び出しをサポートします。ツールを無効にするには `false` を設定します。 |
    | `supportsStrictMode` | 厳密なツールスキーマを受け入れます。 |
    | `requiresStringContent` | Chat Completions のメッセージ内容がプレーン文字列であることを必須とします。 |
    | `strictMessageKeys` | 送信メッセージに、受け入れられるキーのみが含まれることを必須とします。 |
    | `visibleReasoningDetailTypes` | トランスクリプトに安全に表示できる推論詳細ブロックの種類を指定します。 |
    | `supportedReasoningEfforts` | エンドポイントが受け入れる推論ラベルを列挙します。 |
    | `reasoningEffortMap` | OpenClaw の思考ラベルをエンドポイント固有のラベルにマッピングします。 |
    | `maxTokensField` | `max_tokens` または `max_completion_tokens` を選択します。 |
    | `thinkingFormat` | エンドポイントの推論ペイロード方言を選択します。 |
    | `requiresToolResultName` | ツール結果メッセージにツール名が含まれることを必須とします。 |
    | `requiresAssistantAfterToolResult` | ツール結果の後にアシスタントメッセージがあることを必須とします。 |
    | `requiresThinkingAsText` | 推論を構造化コンテンツではなくテキストとして再生します。 |
    | `requiresReasoningContentOnAssistantMessages` | 再生中に DeepSeek 形式の `reasoning_content` を保持します。 |
    | `toolSchemaProfile` | プロバイダー定義のツールスキーマ正規化プロファイルを選択します。 |
    | `unsupportedToolSchemaKeywords` | エンドポイントによって拒否される、指定された JSON Schema キーワードを削除します。 |
    | `toolCallArgumentsEncoding` | エンドポイントのツール呼び出し引数のエンコーディングを選択します。 |
    | `requiresOpenAiAnthropicToolPayload` | OpenAI 形式のツール呼び出しを Anthropic 系のペイロードに変換します。 |

  </Accordion>
  <Accordion title="Amazon Bedrock の検出">
    - `plugins.entries.amazon-bedrock.config.discovery`: Bedrock 自動検出設定のルート。
    - `plugins.entries.amazon-bedrock.config.discovery.enabled`: 暗黙的な検出をオンまたはオフにします。
    - `plugins.entries.amazon-bedrock.config.discovery.region`: 検出に使用する AWS リージョン。
    - `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: 対象を絞った検出に使用するオプションのプロバイダー ID フィルター。
    - `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: 検出の更新に使用するポーリング間隔。
    - `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: 検出されたモデルに使用するフォールバックのコンテキストウィンドウ。
    - `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: 検出されたモデルに使用するフォールバックの最大出力トークン数。

  </Accordion>
</AccordionGroup>

対話型のカスタムプロバイダーのオンボーディングでは、GPT-4o/GPT-4.1/GPT-5+、`o1`/`o3`/`o4` 推論ファミリー、Claude、Gemini、`-vl` で終わる任意の ID（Qwen-VL など）、および LLaVA、Pixtral、InternVL、Mllama、MiniCPM-V、GLM-4V などの名前付きファミリーを含む、既知のビジョンモデル ID パターンに対して画像入力を推論します。既知のテキスト専用ファミリー（Llama、DeepSeek、Mistral/Mixtral、Kimi/Moonshot、Codestral、Devstral、Phi、QwQ、CodeLlama、および vl/vision サフィックスのない単独の Qwen ID）では、追加の質問を省略します。不明なモデル ID の場合は、引き続き画像対応について確認します。非対話型オンボーディングでも同じ推論を使用します。画像対応のメタデータを強制するには `--custom-image-input`、テキスト専用のメタデータを強制するには `--custom-text-input` を渡します。

### プロバイダーの例

<AccordionGroup>
  <Accordion title="Cerebras（GLM 4.7 / GPT OSS）">
    公式の外部 `cerebras` プロバイダー Plugin では、`openclaw onboard --auth-choice cerebras-api-key` を使用してこれを設定できます。デフォルトを上書きする場合にのみ、明示的なプロバイダー設定を使用してください。

    ```json5
    {
      env: { CEREBRAS_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: {
            primary: "cerebras/zai-glm-4.7",
            fallbacks: ["cerebras/gpt-oss-120b"],
          },
          models: {
            "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
            "cerebras/gpt-oss-120b": { alias: "GPT OSS 120B (Cerebras)" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          cerebras: {
            baseUrl: "https://api.cerebras.ai/v1",
            apiKey: "${CEREBRAS_API_KEY}",
            api: "openai-completions",
            models: [
              { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
              { id: "gpt-oss-120b", name: "GPT OSS 120B (Cerebras)" },
            ],
          },
        },
      },
    }
    ```

    Cerebras には `cerebras/zai-glm-4.7`、Z.AI への直接接続には `zai/glm-4.7` を使用してください。

  </Accordion>
  <Accordion title="Kimi Coding">
    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: { "kimi/kimi-for-coding": { alias: "Kimi Code" } },
        },
      },
    }
    ```

    Anthropic 互換の組み込みプロバイダーです。ショートカット: `openclaw onboard --auth-choice kimi-code-api-key`。

  </Accordion>
  <Accordion title="ローカルモデル（LM Studio）">
    [ローカルモデル](/ja-JP/gateway/local-models)を参照してください。要点: 高性能なハードウェア上で LM Studio Responses API を介して大規模なローカルモデルを実行し、フォールバック用にホスト型モデルをマージしたままにします。
  </Accordion>
  <Accordion title="MiniMax M3（直接接続）">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "Minimax" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          minimax: {
            baseUrl: "https://api.minimax.io/anthropic",
            apiKey: "${MINIMAX_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.12, cacheWrite: 0 },
                contextWindow: 1000000,
                maxTokens: 131072,
              },
            ],
          },
        },
      },
    }
    ```

    `MINIMAX_API_KEY` を設定します。ショートカット: `openclaw onboard --auth-choice minimax-global-api` または `openclaw onboard --auth-choice minimax-cn-api`。モデルカタログのデフォルトは M3 で、M2.7 の各バリアントも含まれます。Anthropic 互換のストリーミング経路では、`thinking` を明示的に設定しない限り、OpenClaw はデフォルトで MiniMax M2.x の思考を無効にします。MiniMax-M3（および M3.x）は、デフォルトでプロバイダーの省略時／適応型思考の経路を維持します。`/fast on` または `params.fastMode: true` は、`MiniMax-M2.7` を `MiniMax-M2.7-highspeed` に書き換えます。

  </Accordion>
  <Accordion title="Moonshot AI（Kimi）">
    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: { "moonshot/kimi-k2.6": { alias: "Kimi K2.6" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
            ],
          },
        },
      },
    }
    ```

    中国向けエンドポイントの場合: `baseUrl: "https://api.moonshot.cn/v1"` または `openclaw onboard --auth-choice moonshot-api-key-cn`。

    Moonshot のネイティブエンドポイントは、共有 `openai-completions` トランスポートでストリーミング使用量との互換性を通知します。OpenClaw は、組み込みプロバイダー ID だけではなく、エンドポイントの機能に基づいてこれを有効にします。

  </Accordion>
  <Accordion title="OpenCode">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "opencode/claude-opus-4-6" },
          models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
        },
      },
    }
    ```

    `OPENCODE_API_KEY`（または `OPENCODE_ZEN_API_KEY`）を設定します。Zen カタログには `opencode/...` 参照、Go カタログには `opencode-go/...` 参照を使用します。ショートカット: `openclaw onboard --auth-choice opencode-zen` または `openclaw onboard --auth-choice opencode-go`。

  </Accordion>
  <Accordion title="Synthetic（Anthropic 互換）">
    ```json5
    {
      env: { SYNTHETIC_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
          models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          synthetic: {
            baseUrl: "https://api.synthetic.new/anthropic",
            apiKey: "${SYNTHETIC_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "hf:MiniMaxAI/MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```

    ベース URL では `/v1` を省略する必要があります（Anthropic クライアントが追加します）。ショートカット: `openclaw onboard --auth-choice synthetic-api-key`。

  </Accordion>
  <Accordion title="Z.AI（GLM-4.7）">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-4.7" },
          models: { "zai/glm-4.7": {} },
        },
      },
    }
    ```

    `ZAI_API_KEY` を設定します。モデル参照では、正規の `zai/*` プロバイダー ID を使用します。ショートカット: `openclaw onboard --auth-choice zai-api-key`。

    - 汎用エンドポイント: `https://api.z.ai/api/paas/v4`
    - コーディング用エンドポイント: `https://api.z.ai/api/coding/paas/v4`
    - デフォルトの `zai-api-key` 認証オプションではキーを検証し、そのキーが属するエンドポイントを自動検出します（検出結果が不確かな場合は確認プロンプトにフォールバックし、デフォルトで Global を選択します）。明示的に選択できる専用の CN および Coding-Plan 認証オプションもあります。
    - 汎用エンドポイントを使用する場合は、ベース URL を上書きしたカスタムプロバイダーを定義します。

  </Accordion>
</AccordionGroup>

---

## 関連項目

- [設定 — エージェント](/ja-JP/gateway/config-agents)
- [設定 — チャンネル](/ja-JP/gateway/config-channels)
- [設定リファレンス](/ja-JP/gateway/configuration-reference) — その他のトップレベルキー
- [ツールと Plugin](/ja-JP/tools)
