---
read_when:
    - API プロバイダーで障害が発生した場合に備えて、信頼性の高いフォールバックが必要な場合
    - ローカルの AI CLI を実行していて、それらを再利用したい場合
    - CLI バックエンドのツールにアクセスするための MCP ループバックブリッジについて理解する必要がある場合
summary: CLI バックエンド：オプションの MCP ツールブリッジを備えたローカル AI CLI フォールバック
title: CLI バックエンド
x-i18n:
    generated_at: "2026-07-26T09:19:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ce0427c587bf2a1e0a2ff24b5e76952eecae059e6f900af777b897b2d8d4210
    source_path: gateway/cli-backends.md
    workflow: 16
---

OpenClaw は、API プロバイダーが停止している、レート制限されている、または正常に動作していない場合に、テキスト専用のフォールバックとしてローカル AI CLI を実行できます。これは意図的に保守的な設計です。

- OpenClaw ツールは直接注入されませんが、`bundleMcp: true` を備えたバックエンドは、local loopback MCP ブリッジを介して Gateway ツールを受け取れます。
- 対応する CLI では JSONL ストリーミングを使用します。
- セッションに対応しているため、後続ターンの一貫性が保たれます。
- CLI が画像パスを受け付ける場合、画像も渡されます。

これはプライマリパスではなく、「常に動作する」テキスト応答の安全策として使用してください。ACP セッション制御、バックグラウンドタスク、スレッド／会話のバインド、永続的な外部コーディングセッションを備えた完全なハーネスランタイムには、代わりに [ACP エージェント](/ja-JP/tools/acp-agents)を使用してください。CLI バックエンドは ACP ではありません。

<Tip>
  新しいバックエンド Plugin を構築していますか？[CLI バックエンド Plugin](/ja-JP/plugins/cli-backend-plugins)を参照してください。このページでは、登録済みバックエンドの設定と運用について説明します。
</Tip>

## クイックスタート

同梱の Anthropic Plugin はデフォルトの `claude-cli` バックエンドを登録するため、Claude Code をインストールしてログインしておくだけで、追加設定なしに動作します。

```bash
openclaw agent --agent main --message "hi" --model claude-cli/claude-sonnet-4-6
```

明示的なエージェントリストが設定されていない場合、`main` がデフォルトのエージェント ID です。それ以外の場合は、独自のエージェント ID に置き換えてください。

Gateway サービスの `PATH` に CLI が存在する必要があります。デプロイで標準外の
実行可能ファイルパスや引数が必要な場合は、起動の仕組みを
`openclaw.json` に記述するのではなく、
[CLI バックエンド Plugin](/ja-JP/plugins/cli-backend-plugins)でそのアダプターを登録してください。

モデル選択またはモデルスコープの `agentRuntime.id` がバックエンドを参照すると、
OpenClaw はそのバックエンドを所有する同梱 Plugin を自動的に読み込みます。

## フォールバックとして使用する

プライマリモデルが失敗した場合にのみ実行されるよう、CLI バックエンドをフォールバックリストに追加します。

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["claude-cli/claude-sonnet-4-6"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "claude-cli/claude-sonnet-4-6": {},
      },
    },
  },
}
```

設定済みのフォールバックは、`agents.defaults.modelPolicy.allow` に含まれていない場合でも、プライマリプロバイダーが失敗（認証、レート制限、タイムアウト）すると引き続き候補になります。ユーザーが `/model`、セッションオーバーライド、または `--model` を介して直接選択できるようにする場合にのみ、CLI バックエンドモデルをそのポリシーに追加してください。`agents.defaults.models` が管理するのは、モデルごとのエイリアス、パラメーター、メタデータのみです。

## 設定

ユーザーはモデルとランタイムポリシーを通じて、登録済みバックエンドを選択します。
モデル参照を正規形に保ち、モデルごとに CLI ランタイムを選択します。

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-5",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
      },
    },
  },
}
```

認証情報は、引き続き OpenClaw の認証プロファイルまたは所有元 Plugin の設定に保持されます。
コマンド、argv、環境、解析、セッション、画像、ウォッチドッグの仕組みは、
`api.registerCliBackend(...)` で登録される Plugin コードです。

## 仕組み

1. プロバイダープレフィックス（`claude-cli/...`）に基づいてバックエンドを選択します。
2. 同じ OpenClaw プロンプトとワークスペースコンテキストを使用して、システムプロンプトを構築します。
3. 履歴の一貫性を保つため、セッション ID（対応している場合）を指定して CLI を実行します。同梱の `claude-cli` バックエンドは、OpenClaw セッションごとに Claude の stdio プロセスを維持し、stream-json stdin 経由で後続ターンを送信します。
4. 出力（JSON またはプレーンテキスト）を解析し、最終テキストを返します。
5. バックエンドごとにセッション ID を永続化し、後続ターンで同じ CLI セッションを再利用します。

## タイムアウトと長時間実行される処理

CLI バックエンドには、互いに独立した 2 つの制限があります。

- `agents.defaults.timeoutSeconds` はエージェントターン全体を制限します。通常の Gateway ターンはデフォルトの 48 時間を継承し、`0` を指定するとターンの時間枠が無制限になります。`600` などの保存済みオーバーライドは、そのデフォルトを置き換えます。
- CLI の無出力ウォッチドッグは、無出力の状態が続くサブプロセスを停止します。各バックエンド Plugin は新規実行用と再開用に別々のプロファイルを所有し、全体のターン時間枠が無制限の場合でも、ウォッチドッグは引き続き有効です。

短い全体タイムアウトのオーバーライドを削除すると 48 時間のデフォルトに戻ります。または、12 時間などの明示的な時間枠を設定します。

```bash
# 48 時間のデフォルトに戻す:
openclaw config unset agents.defaults.timeoutSeconds

# または、明示的に 12 時間の制限を選択する:
openclaw config set agents.defaults.timeoutSeconds 43200
```

CLI 内で開始されたバックグラウンド処理も、その CLI サブプロセスの一部です。親ターンが全体の制限に達すると、OpenClaw はサブプロセスと CLI 内部のバックグラウンドタスクをまとめて停止します。永続的な長時間処理には、デタッチされた OpenClaw [サブエージェント](/ja-JP/tools/subagents)または [ACP エージェント](/ja-JP/tools/acp-agents)を使用してください。デタッチされたサブエージェントには、デフォルトで実行タイムアウトがありません。

`openclaw agent` コマンドにも独自のリクエスト期限があります。600 秒のフォールバックデフォルトは通常の Gateway ターンではなく、そのコマンド呼び出しに適用されます。[`openclaw agent`](/ja-JP/cli/agent)を参照してください。

### Claude CLI 固有の動作

同梱の `claude-cli` バックエンドは、Claude Code のネイティブ Skills リゾルバーを優先します。現在の Skills スナップショットに、実体化されたパスを持つ選択済み Skill が 1 つ以上ある場合、OpenClaw は `--plugin-dir` を介して一時的な Claude Code Plugin を渡し、追加されるシステムプロンプトから重複する OpenClaw Skills カタログを省略します。実体化された Plugin Skill がない場合、OpenClaw はフォールバックとしてプロンプトカタログを維持します。Skill の環境変数／API キーのオーバーライドは、実行時の子プロセス環境にも引き続き適用されます。

Claude CLI には独自の非対話型権限モードがあります。OpenClaw は Claude 固有の設定を追加する代わりに、それを既存の実行ポリシーに対応付けます。OpenClaw が管理する Claude ライブセッションでは、有効な実行ポリシーが決定権を持ちます。YOLO（`tools.exec.mode: "full"`）では通常、`--permission-mode bypassPermissions` を指定して Claude を起動し、制限的なポリシーでは `--permission-mode default` を指定して起動します。root で実行される Gateway でも、Claude Code が root に対するバイパスモードを拒否するため、`default` を使用します。エージェントごとの `agents.entries.*.tools.exec` 設定は、そのエージェントについてグローバルの `tools.exec` をオーバーライドします。Anthropic Plugin は、有効なポリシーとホストの制限に一致するよう Claude の権限フラグを正規化します。

制限的なポリシーでは、Claude は自身のネイティブツールまたは拡張ツール（独自の Bash、WebFetch、Claude in Chrome ブラウザツール）を使用する前に、stdio 経由で OpenClaw に確認します。有効な実行時の確認設定が `on-miss` または `always` の場合、OpenClaw は各リクエストを対話型の承認としてセッションのチャンネルに中継します。**Allow once** はその 1 回の呼び出しを許可し、**Allow always** はライブ Claude セッションの残りの期間、そのツール名を許可します（メモリ内のみで、永続化されることはありません）。**Deny**、タイムアウト、または到達不能な承認経路では、いずれも呼び出しが拒否されます。確認を行わないポリシーは従来の動作を維持します。`security: "deny"` はすべてのリクエストを拒否し、完全なセキュリティ未満（実行モード `allowlist`）での確認 `off` は、確認せずに拒否します。

### Claude ブラウザツールと 1Password サインイン

Claude Code は、[Claude in Chrome extension](https://code.claude.com/docs/en/chrome)を介して Chrome ブラウザを操作でき、[1Password for Claude](/ja-JP/gateway/1password#browser-sign-in-with-1password-for-claude)による認証情報の自動入力も利用できます。同梱のバックエンドでは有効化されていません。`claude-stream-json` 方言のバックエンドの起動引数に `--chrome` を追加する [CLI バックエンド Plugin](/ja-JP/plugins/cli-backend-plugins)を登録してください。OpenClaw は通常の実行では設定済みの `--chrome` を維持し、補足質問などツールポリシーが制限された実行では、常に `--no-chrome` を強制します。Chrome ウィンドウ、拡張機能、1Password の承認プロンプトはいずれも Gateway ホスト上に表示されるため、認証情報の使用を承認するには誰かがそのマシンにいる必要があります。

このバックエンドは、OpenClaw の `/think` レベルを Claude Code のネイティブな `--effort` フラグにも対応付けます。`minimal`/`low` -> `low`、`medium` -> `medium` となり、`high`/`xhigh`/`max` はそのまま渡されます。これにより、サブスクリプションを利用する Claude CLI と API キー経路で、対応する Fable 5 の effort レベルが同一に保たれます。`adaptive` は設定済みの `--effort` フラグを削除し、代替を指定しません。そのため、Claude Code は自身の環境、設定、モデルのデフォルトから有効な effort を決定します。他の CLI バックエンドでは、`/think` が生成された CLI に影響する前に、所有元 Plugin が同等の argv マッパーを宣言する必要があります。

OpenClaw が `claude-cli` を使用するには、事前に同じホスト上で Claude Code 自体にログインしておく必要があります。

```bash
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

Docker インストールでは、ホスト上だけでなく、永続化されたコンテナホーム内にも Claude Code をインストールしてログインする必要があります。[Docker での Claude CLI バックエンド](/ja-JP/install/docker#claude-cli-backend-in-docker)を参照してください。

Gateway サービスは `PATH` 上で `claude` を解決できる必要があります。標準外のパスを使用する場合は、
小さなラッパーバックエンド Plugin を登録してください。

## セッション

- CLI がセッションに対応している場合は、`{sessionId}` プレースホルダー（例: `["--session-id", "{sessionId}"]`）を含めて `sessionArgs` を設定します。
- CLI が異なるフラグを持つ resume サブコマンドを使用する場合は、`resumeArgs`（再開時に `args` を置き換えます）を設定し、JSON 以外の再開には必要に応じて `resumeOutput` を設定します。
- `sessionMode`:
  - `always`: 常にセッション ID を送信します（保存済みの ID がない場合は新しい UUID）。
  - `existing`: 以前に保存されたセッション ID がある場合にのみ送信します。
  - `none`: セッション ID を送信しません。
- `claude-cli` のデフォルトは `liveSession: "claude-stdio"`、`output: "jsonl"`、`input: "stdin"` です。そのため、トランスポートフィールドを省略したカスタム設定を含め、ライブ Claude プロセスが稼働中は後続ターンでそのプロセスを再利用します。Gateway が再起動するか、アイドル状態のプロセスが終了した場合、OpenClaw は保存済みの Claude セッション ID から再開します。再開前に、保存済みセッション ID に対応する読み取り可能なプロジェクトトランスクリプトがあることを検証します。トランスクリプトがない場合、`--resume` の下で暗黙に新しいセッションを開始するのではなく、バインドを解除します（`reason=transcript-missing` としてログに記録されます）。
- Claude ライブセッションでは、JSONL 出力ガードが制限付きで維持されます。1 ターンあたり 8 MiB、未加工の JSONL 20,000 行です。
- 保存済み CLI セッションは、プロバイダーが所有する継続性です。自動リセットはデフォルトで無効です。`/reset` と、明示的な日次またはアイドル状態の `session.reset` ポリシーでは、引き続きセッションが切断されます。
- 新しい CLI セッションは通常、OpenClaw の Compaction 要約と Compaction 後の末尾部分のみから再シードされます。Compaction 前に無効化された短いセッションを復元するため、バックエンドは `reseedFromRawTranscriptWhenUncompacted: true` でオプトインできます。未加工のトランスクリプトによる再シードは引き続き制限され、CLI トランスクリプトの欠落、孤立したツール使用の末尾、メッセージポリシー／システムプロンプト／cwd／MCP の変更、セッション期限切れによる再試行など、安全な無効化に限定されます。認証プロファイルまたは認証情報エポックの変更では、未加工のトランスクリプト履歴が再シードされることはありません。

直列化: `serialize: true` は同じレーンの実行順序を維持します（ほとんどの CLI は単一のプロバイダーレーン上で直列化されます）。また、選択された認証アイデンティティが変更された場合、OpenClaw は保存済み CLI セッションの再利用を停止します。これには、認証プロファイル ID、静的 API キー、静的トークン、または CLI が公開している場合の OAuth アカウントアイデンティティの変更が含まれます。OAuth のアクセストークン／リフレッシュトークンのローテーションだけでは、セッションは切断されません。CLI に安定した OAuth アカウント ID がない場合、OpenClaw はその CLI 自体に再開権限の適用を委ねます。

## claude-cli セッションからのフォールバック前置き

`claude-cli` の試行が [`agents.defaults.model.fallbacks`](/ja-JP/concepts/model-failover) で非 CLI 候補へフェイルオーバーすると、OpenClaw は Claude Code のローカル JSONL トランスクリプト（`~/.claude/projects/` 配下にあり、ワークスペースごとにキー設定される）から取得したコンテキスト前置きを次の試行にシードします。このシードがない場合、`claude-cli` 実行では OpenClaw 自身のセッショントランスクリプトが空であるため、フォールバックプロバイダーはコールドスタートします。

- 前置きでは、最新の `/compact` サマリーまたは `compact_boundary` マーカーが優先され、その後に境界以降の最新ターンが文字数予算の範囲内で追加されます。境界以前のターンはサマリーにすでに含まれているため、破棄されます。
- プロンプト予算を正確に保つため、ツールブロックはコンパクトな `(tool call: name)` および `(tool result: …)` ヒントに統合されます。大きすぎるサマリーは切り詰められ、`(truncated)` とラベル付けされます。
- 同一プロバイダー内の `claude-cli` から `claude-cli` へのフォールバックは Claude 自身の `--resume` に依存し、前置きをスキップします。
- シードは既存の Claude セッションファイルパス検証を再利用するため、任意のパスを読み取ることはできません。

## 画像

Plugin 作成者は `imageArg` を使用して画像パスのサポートを宣言します。

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw は base64 画像を一時ファイルに書き込みます。`imageArg` が設定されている場合、それらのパスは CLI 引数として渡されます。設定されていない場合、OpenClaw はファイルパスをプロンプトに追加します（パスインジェクション）。これは、プレーンなパスからローカルファイルを自動的に読み込む CLI で機能します。

## 入出力

- `output: "text"`（デフォルト）は stdout を最終応答として扱います。
- `output: "json"` は JSON の解析を試み、テキストとセッション ID を抽出します。
- `output: "jsonl"` は JSONL ストリームを解析し、最終エージェントメッセージと、存在する場合はセッション識別子を抽出します。
- Gemini CLI の JSON 出力では、OpenClaw は `usage` が存在しないか空の場合、`response` から応答テキストを、`stats` から使用量を読み取ります。同梱の Gemini CLI アダプターは `stream-json` を使用します。

入力モード：

- `input: "arg"`（デフォルト）はプロンプトを最後の CLI 引数として渡します。
- `input: "stdin"` はプロンプトを stdin 経由で送信します。
- プロンプトが非常に長く、`maxPromptArgChars` が設定されている場合は、代わりに stdin が使用されます。

## Plugin 所有のデフォルト

CLI バックエンドのデフォルトは Plugin サーフェスの一部です。

- Plugin は `api.registerCliBackend(...)` を使用してそれらを登録します。
- バックエンドの `id` はモデル参照内のプロバイダープレフィックスになります。
- コマンド、argv、環境、パーサー、セッション、およびウォッチドッグの動作は Plugin コード内に保持されます。
- バックエンド固有の正規化は、オプションの `normalizeConfig` フックを通じて Plugin が所有します。

Anthropic は `claude-cli` を、Google は `google-gemini-cli` を所有します。OpenAI Codex エージェントの実行では、`openai/*` を通じて Codex アプリサーバーハーネスを使用します。OpenClaw は同梱の `codex-cli` バックエンドを登録しなくなりました。

同梱の Anthropic Plugin は `claude-cli` 用に次を登録します。

| キー                   | 値                                                                                                                                                                                                         |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`             | `claude`                                                                                                                                                                                                      |
| `args`                | `-p --output-format stream-json --include-partial-messages --verbose --setting-sources user --allowedTools mcp__openclaw__* --disallowedTools ScheduleWakeup,CronCreate,Bash(run_in_background:true),Monitor` |
| `output`              | `jsonl`                                                                                                                                                                                                       |
| `input`               | `stdin`                                                                                                                                                                                                       |
| `modelArg`            | `--model`                                                                                                                                                                                                     |
| `sessionArgs`         | `["--session-id", "{sessionId}"]`                                                                                                                                                                             |
| `sessionMode`         | `always`                                                                                                                                                                                                      |
| `imageArg`            | `@`                                                                                                                                                                                                           |
| `imagePathScope`      | `workspace`                                                                                                                                                                                                   |
| `systemPromptFileArg` | `--append-system-prompt-file`                                                                                                                                                                                 |
| `systemPromptMode`    | `append`                                                                                                                                                                                                      |

同梱の Google Plugin は `google-gemini-cli` 用に次を登録します。

| キー                       | 値                                                                                  |
| ------------------------- | -------------------------------------------------------------------------------------- |
| `command`                 | `gemini`                                                                               |
| `args`                    | `--skip-trust --approval-mode auto_edit --output-format stream-json --prompt {prompt}` |
| `resumeArgs`              | 同じ（`--resume {sessionId}` を指定）                                                      |
| `output` / `resumeOutput` | `jsonl`                                                                                |
| `jsonlDialect`            | `gemini-stream-json`                                                                   |
| `imageArg`                | `@`                                                                                    |
| `imagePathScope`          | `workspace`                                                                            |
| `modelArg`                | `--model`                                                                              |
| `sessionMode`             | `existing`                                                                             |
| `sessionIdFields`         | `["session_id", "sessionId"]`                                                          |

前提条件：ローカルの Gemini CLI がインストールされ、`gemini`（`brew install gemini-cli` または `npm install -g @google/gemini-cli`）として `PATH` 上に存在する必要があります。

Gemini CLI 出力に関する注意事項：

- デフォルトの `stream-json` パーサーは、アシスタントの `message` イベント、ツールイベント、最終的な `result` 使用量、および致命的な Gemini エラーイベントを読み取ります。
- `usage` が存在しないか空の場合、使用量は `stats` にフォールバックします。`stats.cached` は OpenClaw の `cacheRead` に正規化され、`stats.input` がない場合、入力トークンは `stats.input_tokens - stats.cached` から算出されます。

## テキスト変換オーバーレイ

小規模なプロンプト／メッセージ互換シムを必要とする Plugin は、プロバイダーや CLI バックエンドを置き換えることなく、双方向テキスト変換を宣言できます。

```typescript
api.registerTextTransforms({
  input: [{ from: /red basket/g, to: "blue basket" }],
  output: [{ from: /blue basket/g, to: "red basket" }],
});
```

`input` は CLI に渡されるシステムプロンプトとユーザープロンプトを書き換えます。`output` は、OpenClaw が自身の制御マーカーとチャンネル配信を処理する前に、ストリーミングされたアシスタントテキストと解析済みの最終テキストを書き換えます。プロバイダー経由のモデル呼び出しでは、ストリーム修復後かつツール実行前に、構造化されたツール呼び出し引数内の文字列値も復元します。生のプロバイダー JSON フラグメントは変更されません。コンシューマーは構造化された部分、終了、または結果のペイロードを使用してください。

プロバイダー固有の JSONL イベントを出力する CLI では、そのバックエンドの設定で `jsonlDialect` を設定します。Claude Code 互換ストリームには `claude-stream-json`、Gemini CLI の `stream-json` イベントには `gemini-stream-json` を使用します。

## ネイティブ Compaction の所有権

一部の CLI バックエンドは自身のトランスクリプトを Compaction するエージェントを実行するため、OpenClaw はそれらに対して保護用サマライザーを実行してはなりません。実行すると、バックエンド自身の Compaction と競合し、ターンが完全に失敗する可能性があります。

`claude-cli` にはハーネスエンドポイントがなく（Claude Code は内部で Compaction します）、そのため `ownsNativeCompaction: true` を宣言し、OpenClaw の Compaction パスはセッションエントリを変更せずに返します。OpenClaw は実行時の有効なコンテキスト予算を、Claude Code で文書化されている [`CLAUDE_CODE_AUTO_COMPACT_WINDOW`](https://code.claude.com/docs/en/env-vars) を通じて渡し、ネイティブの自動 Compaction を設定済みの Anthropic `contextTokens` 制限に合わせます。一方、Codex などのネイティブハーネスセッションは、引き続き各ハーネスの Compaction エンドポイントにルーティングされます。

```typescript
api.registerCliBackend({ id: "my-cli", ownsNativeCompaction: true /* ... */ });
```

`ownsNativeCompaction` は、Compaction を実際に所有するバックエンドにのみ宣言してください。コンテキストウィンドウ付近で自身のトランスクリプトを確実に制限し、再開可能なセッション（例：`--resume` / `--session-id`）を永続化する必要があります。そうでない場合、遅延されたセッションが予算超過のままになる可能性があります。

## バンドル MCP オーバーレイ

CLI バックエンドは OpenClaw のツール呼び出しを直接受信しませんが、バックエンドは `bundleMcp: true` を使用して生成された MCP 設定オーバーレイを有効にできます。現在の同梱動作：

- `claude-cli`：生成された厳格な MCP 設定ファイル。
- `google-gemini-cli`：生成された Gemini システム設定ファイル。

バンドル MCP が有効な場合、OpenClaw は次を行います。

- Gateway ツールを CLI プロセスに公開する loopback HTTP MCP サーバーを起動します。このサーバーは、現在の実行試行中のみ有効な実行ごとのコンテキスト許可（`OPENCLAW_MCP_TOKEN`）で認証されます。
- 子プロセスのヘッダーを信頼する代わりに、ツールアクセスを Gateway が選択したセッション、アカウント、およびチャンネルのコンテキストに関連付けます。
- 現在のワークスペースで有効なバンドル MCP サーバーを読み込み、既存のバックエンド MCP 設定／セッティング形式とマージします。
- 所有元 Plugin のバックエンド所有統合モードを使用して、起動設定を書き換えます。

制限付き実行（`toolsAllow` を使用する cron ジョブなど）には、バックエンドが所有する厳密な変換が必要です。バンドルされた `claude-cli` バックエンドは、Claude のネイティブツールと、フック、plugins、エージェント、Skills、`CLAUDE.md` を含むユーザー、プロジェクト、ローカルのカスタマイズを無効にします。その後、許可されたすべての OpenClaw ツールを、付与スコープの MCP サーバーを通じて公開します。これにより、Claude のネイティブツールやカスタマイズプロセスへ権限を拡大するのではなく、ファイルシステム、プロセス、exec、承認、サンドボックスポリシーを OpenClaw 内に保持します。同じ MCP リストが Claude の生成済み設定で適用され、さらにツールの一覧取得時と実行時に Gateway によって再度適用されます。付与を発行する前に、コアは元の許可リストにない MCP 権限を指定するバックエンド変換を拒否します。厳密な変換がないバックエンドは、引き続きフェイルクローズします。

MCP サーバーが有効になっていない場合でも、バックエンドがバンドル MCP の使用を選択すると、OpenClaw は厳格な設定を注入するため、バックグラウンド実行は分離されたままになります。

セッションスコープのバンドル MCP ランタイムは、セッション内で再利用するためにキャッシュされ、アイドル状態が 10 分続くと破棄されます。認証プローブ、スラッグ生成、Active Memory の呼び出しなどのワンショット組み込み実行では、stdio 子プロセスや Streamable HTTP/SSE ストリームが実行終了後も存続しないよう、実行終了時のクリーンアップを要求します。

`claude-cli` では、互換性のある、選択済みまたは順序付けされた OpenClaw OAuth/トークンプロファイルが、その Claude 子プロセスに転送されます。これにより、互換性のあるプロファイルが存在しない場合は Claude のネイティブホストログインを維持しつつ、そのターンではエージェントごとのプロファイルが信頼できる情報源になります。

## 履歴再シードの上限

新しい CLI セッションが以前の OpenClaw トランスクリプトからシードされる場合（たとえば `session_expired` の再試行後）、再シードプロンプトが過度に膨張しないよう、レンダリングされる `<conversation_history>` ブロックに上限が設定されます。デフォルトは 12,288 文字（約 3,000 トークン）です。

Claude CLI バックエンドでは、代わりに解決された Claude コンテキストウィンドウに応じてこの上限を調整します。コンテキストウィンドウが大きいほど以前の履歴から取得する範囲も大きくなりますが、固定の最大値までに制限されます。他の CLI バックエンドでは、保守的なデフォルト値が維持されます。この上限が制御するのは、再シードプロンプト内の以前の履歴ブロックだけです。

## 制限事項

- OpenClaw は、CLI バックエンドプロトコルにツール呼び出しを注入しません。バックエンドが Gateway ツールを認識するのは、`bundleMcp: true` の使用を選択した場合だけです。
- ストリーミングはバックエンド固有です。一部のバックエンドは JSONL をストリーミングし、その他は終了するまでバッファリングします。
- 構造化出力は、CLI 独自の JSON 形式に依存します。

## トラブルシューティング

| 症状                  | 修正方法                                                                                                     |
| --------------------- | ------------------------------------------------------------------------------------------------------------ |
| CLI が見つからない    | CLI を Gateway サービスの `PATH` に配置するか、所有元 Plugin の登録済みコマンドを更新します。 |
| モデル名が正しくない  | Plugin の `modelAliases` マッピングを更新します。                                                       |
| セッションが継続しない | Plugin の `sessionArgs` と `sessionMode` を確認します。                                           |
| 画像が無視される      | Plugin の `imageArg` と CLI のファイルパス対応を確認します。                                        |

## 関連項目

- [Gateway 運用ガイド](/ja-JP/gateway)
- [ローカルモデル](/ja-JP/gateway/local-models)
