---
read_when:
    - Hermes または別のエージェントシステムから OpenClaw へ移行したい場合
    - Plugin が所有する移行プロバイダーを追加する場合
summary: '`openclaw migrate` の CLI リファレンス（別のエージェントシステムから状態をインポート）'
title: 移行する
x-i18n:
    generated_at: "2026-07-26T09:56:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f492535019f8a69706ff918462ba74cf5d26e733d2e4e9493b3c76bd77f2584d
    source_path: cli/migrate.md
    workflow: 16
---

# `openclaw migrate`

Plugin が所有する移行プロバイダーを通じて、別のエージェントシステムから状態をインポートします。同梱プロバイダーは Claude、Codex CLI、[Hermes](/ja-JP/install/migrating-hermes) に対応しています。Plugin は追加のプロバイダーを登録できます。

<Tip>
ユーザー向けの手順については、[Claude からの移行](/ja-JP/install/migrating-claude)および[Hermes からの移行](/ja-JP/install/migrating-hermes)を参照してください。[移行ハブ](/ja-JP/install/migrating)にはすべての移行経路が掲載されています。
</Tip>

## コマンド

```bash
openclaw migrate list
openclaw migrate claude --dry-run
openclaw migrate codex --dry-run
openclaw migrate codex --skill gog-vault77-google-workspace
openclaw migrate codex --plugin google-calendar --dry-run
openclaw migrate codex --plugin google-calendar --verify-plugin-apps --dry-run
openclaw migrate hermes --dry-run
openclaw migrate hermes
openclaw migrate apply codex --yes --skill gog-vault77-google-workspace
openclaw migrate apply codex --yes --plugin google-calendar
openclaw migrate apply codex --yes
openclaw migrate apply claude --yes
openclaw migrate apply hermes --yes
openclaw migrate apply hermes --include-secrets --yes
openclaw onboard --flow import
openclaw onboard --import-from claude --import-source ~/.claude
openclaw onboard --import-from hermes --import-source ~/.hermes
```

ほかのフラグを指定せずに `openclaw migrate <provider>` を実行すると、計画とプレビューが行われ、適用前に（TTY では）確認を求められます。`openclaw migrate plan <provider>` と `openclaw migrate apply <provider>` は、同じフラグを使用してプレビューと適用を別々のサブコマンドに分割します。

<ParamField path="<provider>" type="string">
  登録済み移行プロバイダーの名前（例: `hermes`）。インストール済みのプロバイダーを確認するには、`openclaw migrate list` を実行します。
</ParamField>
<ParamField path="--dry-run" type="boolean">
  計画を作成し、状態を変更せずに終了します。
</ParamField>
<ParamField path="--from <path>" type="string">
  ソース状態ディレクトリを上書きします。Hermes は `$HERMES_HOME` とアクティブなプロファイルに従い、その後プラットフォームのデフォルト（`~/.hermes` または `%LOCALAPPDATA%\hermes`）を使用します。Codex のデフォルトは `~/.codex`（または `$CODEX_HOME`）、Claude のデフォルトは `~/.claude` です。
</ParamField>
<ParamField path="--include-secrets" type="boolean">
  対応している認証情報を確認なしでインポートします。対話形式の適用では、検出された認証用の認証情報をインポートする前に確認が表示され、デフォルトでは「はい」が選択されています。非対話形式の `--yes` でインポートするには、`--include-secrets` が必要です。
</ParamField>
<ParamField path="--no-auth-credentials" type="boolean">
  対話形式の確認を含め、認証用の認証情報のインポートをスキップします。
</ParamField>
<ParamField path="--overwrite" type="boolean">
  計画で競合が報告された場合に、既存の対象を適用処理で置き換えられるようにします。
</ParamField>
<ParamField path="--yes" type="boolean">
  確認プロンプトをスキップします。非対話モードでは必須です。
</ParamField>
<ParamField path="--skill <name>" type="string">
  スキル名または項目 ID で、コピーするスキル項目を 1 つ選択します。複数のスキルを移行するには、このフラグを繰り返し指定します。省略した場合、対話形式の Codex 移行ではチェックボックス選択画面が表示され、非対話形式の移行では計画されたすべてのスキルが維持されます。
</ParamField>
<ParamField path="--plugin <name>" type="string">
  Plugin 名または項目 ID で、インストールする Codex Plugin 項目を 1 つ選択します。複数の Codex Plugin を移行するには、このフラグを繰り返し指定します。省略した場合、対話形式の Codex 移行ではネイティブ Codex Plugin のチェックボックス選択画面が表示され、非対話形式の移行では計画されたすべての Plugin が維持されます。Codex app-server のインベントリで検出された、ソースにインストール済みの `openai-curated` Codex Plugin にのみ適用されます。
</ParamField>
<ParamField path="--verify-plugin-apps" type="boolean">
  Codex 専用です。ネイティブ Plugin の有効化を計画する前に、ソース Codex app-server の `app/list` トラバーサルを新たに強制実行します。移行計画を高速に保つため、デフォルトではオフです。
</ParamField>
<ParamField path="--backup-output <path>" type="string">
  移行前バックアップのアーカイブパスまたはディレクトリです。`openclaw backup create` にそのまま渡されます。
</ParamField>
<ParamField path="--no-backup" type="boolean">
  適用前のバックアップをスキップします。ローカルの OpenClaw 状態が存在する場合は、`--force` が必要です。
</ParamField>
<ParamField path="--force" type="boolean">
  適用処理がバックアップのスキップを拒否する場合に、`--no-backup` と併せて指定する必要があります。
</ParamField>
<ParamField path="--json" type="boolean">
  計画または適用結果を JSON として出力します。`--json` を指定し、`--yes` を指定していない場合、適用処理は計画を出力し、状態を変更しません。
</ParamField>

## 安全性モデル

`openclaw migrate` はプレビュー優先です。

<AccordionGroup>
  <Accordion title="適用前のプレビュー">
    何かが変更される前に、プロバイダーは競合、スキップされた項目、機密項目を含む項目別の計画を返します。JSON 計画、適用出力、移行レポートでは、API キー、トークン、Authorization ヘッダー、Cookie、パスワードなど、ネストされたシークレットらしいキーがマスキングされます。

    `openclaw migrate apply <provider>` は計画をプレビューし、`--yes` が設定されていない限り、状態を変更する前に確認を求めます。非対話モードでは、適用に `--yes` が必要です。

  </Accordion>
  <Accordion title="バックアップ">
    適用処理は、移行を適用する前に OpenClaw のバックアップを作成して検証します。ローカルの OpenClaw 状態がまだ存在しない場合、バックアップ手順はスキップされ、移行が続行されます。状態が存在する場合にバックアップをスキップするには、`--no-backup` と `--force` の両方を渡します。
  </Accordion>
  <Accordion title="競合">
    計画に競合がある場合、適用処理は続行を拒否します。計画を確認し、既存の対象を意図的に置き換える場合は、`--overwrite` を指定して再実行します。プロバイダーは、上書きされたファイルの項目単位のバックアップを移行レポートディレクトリに書き込むことがあります。
  </Accordion>
  <Accordion title="シークレット">
    対話形式の適用では、検出された認証用の認証情報をインポートするか確認され、デフォルトでは「はい」が選択されています。スキップするには `--no-auth-credentials` を使用し、無人で認証情報をインポートするには `--yes` とともに `--include-secrets` を使用します。
  </Accordion>
</AccordionGroup>

## Claude プロバイダー

同梱の Claude プロバイダーは、デフォルトで `~/.claude` にある Claude Code の状態を検出します。特定の Claude Code ホームまたはプロジェクトルートからインポートするには、`--from <path>` を使用します。

<Tip>
ユーザー向けの手順については、[Claude からの移行](/ja-JP/install/migrating-claude)を参照してください。
</Tip>

### Claude がインポートするもの

- `~/.claude/projects/*/memory` にある Claude Code の自動メモリ Markdown と、
  ユーザーが設定した `autoMemoryDirectory`。インデックス付きの再呼び出し用に
  `memory/imports/claude-code/` の下へコピーされます。
- プロジェクトの `CLAUDE.md` と `.claude/CLAUDE.md` を OpenClaw エージェントワークスペース（`AGENTS.md`）にインポートします。
- ユーザーの `~/.claude/CLAUDE.md` をワークスペースの `USER.md` に追記します。
- プロジェクトの `.mcp.json`、Claude Code の `~/.claude.json`（プロジェクトごとのエントリを含む）、Claude Desktop の `claude_desktop_config.json` から MCP サーバー定義をインポートします。
- `SKILL.md` を含む Claude スキルディレクトリ（ユーザーの `~/.claude/skills` とプロジェクトの `.claude/skills`）。
- Claude のコマンド Markdown ファイル（ユーザーの `~/.claude/commands` とプロジェクトの `.claude/commands`）を、手動実行専用の OpenClaw スキルに変換します。

### アーカイブおよび手動確認の状態

Claude のフック、権限、環境のデフォルト、プロジェクトの `CLAUDE.local.md`、`.claude/rules`、ユーザーおよびプロジェクトの `agents/` ディレクトリ、プロジェクト履歴（`~/.claude` 配下の `projects`、`cache`、`plans`）は、移行レポートに保存されるか、手動確認項目として報告されます。OpenClaw はフックを実行せず、広範な許可リストをコピーせず、OAuth/Desktop の認証情報の状態を自動的にインポートしません。

## Codex プロバイダー

同梱の Codex プロバイダーは、デフォルトで `~/.codex` にある Codex CLI の状態を検出し、その環境変数が設定されている場合は `CODEX_HOME` で検出します。特定の Codex ホームのインベントリを作成するには、`--from <path>` を使用します。

OpenClaw Codex ハーネスへ移行し、有用な個人用 Codex CLI アセットを意図的に昇格させる場合は、このプロバイダーを使用します。ローカルの Codex app-server の起動では、エージェントごとの `CODEX_HOME` が使用されるため、デフォルトでは個人用の `~/.codex` は読み込まれません。通常のプロセスの `HOME` は引き続き継承されるため、Codex は共有の `$HOME/.agents/*` Skills／Plugin マーケットプレイスのエントリを認識でき、サブプロセスはユーザーホームの設定とトークンを見つけられます。

対話型ターミナルで `openclaw migrate codex` を実行すると、完全な計画がプレビューされ、最終的な適用確認の前にチェックボックス選択画面が開きます。最初にスキルのコピー項目が表示されます。一括選択には `Toggle all on` または `Toggle all off` を使用します。Space キーで行の選択状態を切り替えるか、Enter キーで強調表示された行を有効にして続行します。計画されたスキルは選択済み、競合するスキルは未選択で開始されます。`Skip for now` を選ぶと、その実行ではスキルのコピーをスキップし、Plugin の選択へ進みます。ソースにインストール済みのキュレーション対象 Codex Plugin が移行可能で、`--plugin` が指定されていない場合、続いて Plugin 名によるネイティブ Codex Plugin の有効化を選択するよう求められます。対象の OpenClaw Codex Plugin 設定に同じ Plugin がすでに存在しない限り、Plugin 項目は選択済みで開始されます。既存の対象 Plugin は未選択で開始され、`conflict: plugin exists` のような競合のヒントが表示されます。その実行でネイティブ Codex Plugin を移行しない場合は `Toggle all off` を選択し、適用前に停止する場合は `Skip for now` を選択します。

スクリプト実行または厳密な実行では、1 つ以上のスキルまたは Plugin を明示的に選択します。

```bash
openclaw migrate codex --dry-run --skill gog-vault77-google-workspace
openclaw migrate apply codex --yes --skill gog-vault77-google-workspace
openclaw migrate codex --dry-run --plugin google-calendar
openclaw migrate apply codex --yes --plugin google-calendar
```

### Codex がインポートするもの

- `$CODEX_HOME/memories` にある統合済みの Codex `MEMORY.md` と `memory_summary.md`。
  インデックス付きの再呼び出し用に `memory/imports/codex/` の下へコピーされます。
  生のロールアウトメモリはインポートされません。
- `$CODEX_HOME/skills` 配下の Codex CLI スキルディレクトリ（Codex の `.system` キャッシュを除く）。
- `$HOME/.agents/skills` 配下の個人用 AgentSkills。エージェントごとの所有権を維持するため、現在の OpenClaw エージェントワークスペースにコピーされます。
- Codex app-server の `plugin/list` を通じて検出された、ソースにインストール済みの `openai-curated` Codex Plugin。計画時に、有効なインストール済み Plugin ごとに `plugin/read` を読み取ります。

アプリ連携 Plugin の移行には追加のゲートがあります。

- アプリ連携 Plugin を移行するには、ソース Codex app-server のアカウントが ChatGPT サブスクリプションアカウントである必要があります。ChatGPT 以外のアカウントまたはアカウント情報がない応答は、`codex_subscription_required` としてスキップされます。
- デフォルトでは、移行はソースの `app/list` を呼び出しません。そのため、アカウントゲートを通過したアプリ連携 Plugin は、ソースアプリへのアクセス可能性を検証せずに計画され、アカウント検索の通信エラーは `codex_account_unavailable` としてスキップされます。
- `--verify-plugin-apps` を渡すと、ソースの `app/list` スナップショットを新たに強制取得し、所有するすべてのアプリが存在し、有効で、アクセス可能であることを、ネイティブ有効化の計画前に必須とします。このモードでは、アカウント検索の通信エラーが発生した場合、ソースアプリのインベントリ検証へフォールスルーします。スナップショットは現在のプロセスのメモリ内にのみ保持され、移行出力や対象設定には書き込まれません。

無効な Plugin、読み取れない Plugin の詳細、サブスクリプションによって制限されたソースアカウント、および（`--verify-plugin-apps` が設定されている場合）存在しない、無効な、またはアクセス不能なアプリは、対象設定のエントリではなく、型付きの理由を伴う手動スキップ項目になります。適用処理は、対象の app-server がその Plugin をインストール済みかつ有効と報告している場合でも、選択された適格な Plugin ごとに app-server の `plugin/install` を呼び出します。移行された Codex Plugin は、ネイティブ Codex ハーネスを選択するセッションでのみ使用できます。OpenClaw プロバイダーの実行、ACP 会話バインディング、その他のハーネスには公開されません。

### 手動確認が必要な Codex の状態

Codex `config.toml`、ネイティブ `hooks/hooks.json`、厳選されていないマーケットプレイス、ソースからインストールされた厳選済み Plugin ではないキャッシュ済み Plugin バンドル、およびソースのサブスクリプションゲートを通過できないソースインストール済み Plugin は、自動的には有効化されません。`--verify-plugin-apps` が設定されている場合、ソースのアプリインベントリゲートを通過できない Plugin もスキップされます。これらはすべて、手動レビュー用に移行レポートへコピーまたは記録されます。

移行されたソースインストール済みの厳選済み Plugin については、次の内容を書き込みます。

- `plugins.entries.codex.enabled: true`
- `plugins.entries.codex.config.codexPlugins.enabled: true`
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions: true`
- 選択した各 Plugin に対して、`marketplaceName: "openai-curated"` と `pluginName` を含む明示的な Plugin エントリを1つ

移行では `plugins["*"]` を書き込むことはなく、ローカルマーケットプレイスのキャッシュパスも保存しません。

スキップされた Plugin はターゲット設定に書き込まれません。ソース側のサブスクリプションエラーは、型付き理由 `codex_subscription_required`、`codex_account_unavailable`、`plugin_disabled`、または `plugin_read_unavailable` とともに手動対応項目として記録されます。`--verify-plugin-apps` を使用すると、ソースのアプリインベントリエラーも `app_inaccessible`、`app_disabled`、`app_missing`、または `app_inventory_unavailable` として表示されることがあります。ターゲット側で認証が必要なインストールは、該当する Plugin 項目に `status: "skipped"`、`reason: "auth_required"`、およびサニタイズ済みアプリ識別子とともに記録されます。それらの明示的な設定エントリは、再認証して有効化するまで無効な状態で書き込まれます。その他のインストールエラーは、項目単位の `error` 結果となります。

計画中に Codex アプリサーバーの Plugin インベントリを利用できない場合、移行全体を失敗させる代わりに、キャッシュ済みバンドルの参考項目へフォールバックします。

## Hermes プロバイダー

同梱の Hermes プロバイダーは `$HERMES_HOME` とアクティブなプロファイルに従い、その後プラットフォームのデフォルト（`~/.hermes` または `%LOCALAPPDATA%\hermes`）を使用します。検出を上書きするには `--from <path>` を使用します。

### Hermes がインポートするもの

- `config.yaml` からのデフォルトモデル設定。
- `model`、`providers`、および `custom_providers` からの設定済みモデルプロバイダーとカスタム OpenAI 互換エンドポイント。
- `mcp_servers` または `mcp.servers` からの MCP サーバー定義。OpenClaw への完全なマッピングは、デフォルトの Streamable HTTP ルーティング、OAuth スコープ、真偽値の TLS 検証、個別のクライアント証明書／鍵パス、および Hermes のネイティブ／リソース／プロンプトツールポリシーを対象とします。サポートされていない Hermes 専用のランタイムフィールドまたは認証情報フィールドは、手動レビュー用に記録されます。
- `SOUL.md` と `AGENTS.md` を OpenClaw エージェントワークスペースへ。
- `memories/MEMORY.md` と `memories/USER.md` をワークスペースのメモリファイルに追記。
  メモリ専用のサーフェス（オンボーディングのメモリページと Control UI のメモリ
  インポートページ）では、既存のワークスペースメモリには触れず、インデックス化された
  想起のために、これらのファイルを代わりに `memory/imports/hermes/` 配下へコピーします。
- OpenClaw ファイルメモリのメモリ設定デフォルト、および Honcho などの外部メモリプロバイダーに関するアーカイブ項目または手動レビュー項目。
- `skills/` 配下の任意の場所に `SKILL.md` ファイルを含む Skills。ネストされた Skills はワークスペースの Skills ディレクトリへフラット化されます。
- `skills.config` からの Skills ごとの設定値。
- 対話形式の認証情報移行が承認された場合、または `--include-secrets` が設定されている場合の、現在の Hermes OpenAI Codex OAuth 認証情報と OpenCode OpenAI OAuth 認証情報。Hermes と OpenClaw が、インポートされた同一の更新グラントを使用し続けないようにしてください。
- 対話形式の認証情報移行が承認された場合、または `--include-secrets` が設定されている場合の、Hermes `.env` および OpenCode `auth.json` からのサポート対象 API キーとトークン。

### サポートされる `.env` キー

`AI_GATEWAY_API_KEY`、`ALIBABA_API_KEY`、`ANTHROPIC_API_KEY`、`ARCEEAI_API_KEY`、`CEREBRAS_API_KEY`、`CHUTES_API_KEY`、`CLOUDFLARE_AI_GATEWAY_API_KEY`、`COPILOT_GITHUB_TOKEN`、`DASHSCOPE_API_KEY`、`DEEPINFRA_API_KEY`、`DEEPSEEK_API_KEY`、`FIREWORKS_API_KEY`、`GEMINI_API_KEY`、`GH_TOKEN`、`GITHUB_TOKEN`、`GLM_API_KEY`、`GOOGLE_API_KEY`、`GROQ_API_KEY`、`HF_TOKEN`、`HUGGINGFACE_HUB_TOKEN`、`KILOCODE_API_KEY`、`KIMICODE_API_KEY`、`KIMI_API_KEY`、`KIMI_CODING_API_KEY`、`MINIMAX_API_KEY`、`MINIMAX_CODING_API_KEY`、`MISTRAL_API_KEY`、`MODELSTUDIO_API_KEY`、`MOONSHOT_API_KEY`、`NVIDIA_API_KEY`、`OPENAI_API_KEY`、`OPENCODE_API_KEY`、`OPENCODE_GO_API_KEY`、`OPENCODE_ZEN_API_KEY`、`OPENROUTER_API_KEY`、`QIANFAN_API_KEY`、`QWEN_API_KEY`、`TOGETHER_API_KEY`、`VENICE_API_KEY`、`XAI_API_KEY`、`XIAOMI_API_KEY`、`ZAI_API_KEY`、`Z_AI_API_KEY`。

### アーカイブ専用の状態

OpenClaw が安全に解釈できない Hermes の状態は、手動レビュー用に移行レポートへコピーされますが、稼働中の OpenClaw の設定や認証情報には読み込まれません。これには、`plugins/`、`sessions/`、`logs/`、`cron/`、`mcp-tokens/`、`plans/`、`workspace/`、`skins/`、`kanban/`、ペアリング／プラットフォームの状態、Gateway のルーティング／プロセス状態、および検出された Hermes SQLite データベースが含まれます。

### 適用後

```bash
openclaw doctor
```

## Plugin コントラクト

移行元は Plugin です。Plugin は `openclaw.plugin.json` でプロバイダー ID を宣言します。

```json
{
  "contracts": {
    "migrationProviders": ["hermes"]
  }
}
```

実行時に Plugin は `api.registerMigrationProvider(...)` を呼び出します。プロバイダーは `detect`、`plan`、および `apply` を実装します。コアは CLI オーケストレーション、バックアップポリシー、プロンプト、JSON 出力、および競合の事前チェックを担当します。コアはレビュー済みの計画を `apply(ctx, plan)` に渡します。互換性のため、その引数がない場合に限り、プロバイダーは計画を再構築できます。移行項目では、ステージングされたローカルデータが永続的に公開されるまでオンボーディングが延期しなければならない外部の有効化効果に対して、`applyPhase: "after-promotion"` を設定できます。これらのプロバイダーは `deferredApply: { retrySafe: true }` を宣言し、中断されたプロセスの後でも、延期された各効果を安全に再実行できるようにする必要があります。オンボーディングは、宣言されていない延期効果を拒否します。べき等な何もしない処理は、復旧処理が完了済みとして記録できるよう、`deferredCompletion: true` を含む非変更項目を返す必要があります。スタンドアロンの `openclaw migrate` は、引き続き通常のバックアップに基づくフローを通じて完全な計画を適用します。

プロバイダー Plugin は、項目の構築と集計数に `openclaw/plugin-sdk/migration` を使用できるほか、競合を考慮したファイルコピー、アーカイブ専用レポートコピー、キャッシュ済み設定ランタイムラッパー、および移行レポートに `openclaw/plugin-sdk/migration-runtime` を使用できます。

## オンボーディングとの統合

プロバイダーが既知の移行元を検出した場合、オンボーディングは移行を提案できます。`openclaw onboard --flow import` と `openclaw setup --wizard --import-from hermes` はどちらも同じ Plugin 移行プロバイダーを使用し、適用前には引き続きプレビューを表示します。スタンドアロン移行とは異なり、新規ターゲットのオンボーディングパスは、ローカル成果物とインポートされた認証情報をステージングし、ステージング内でインポートされた推論を検証または修復してから、設定をコミットする前にワークスペースとエージェント状態を昇格させます。モード `0600` の昇格ジャーナルにより、次回の実行時に、インポートされたローカルデータを再実行することなく、延期された外部の有効化を含む中断された公開を完了またはロールバックできます。

<Note>
オンボーディングによるインポートには、新規の OpenClaw セットアップが必要です。ローカル状態がすでにある場合は、まず設定、認証情報、セッション、およびワークスペースをリセットしてください。バックアップして上書きするインポートまたはマージインポートは、既存のセットアップでは機能ゲートの対象です。
</Note>

## 関連項目

- [Hermes からの移行](/ja-JP/install/migrating-hermes)：ユーザー向けの手順。
- [Claude からの移行](/ja-JP/install/migrating-claude)：ユーザー向けの手順。
- [移行](/ja-JP/install/migrating)：OpenClaw を新しいマシンへ移動します。
- [Doctor](/ja-JP/gateway/doctor)：移行適用後の健全性チェック。
- [Plugins](/ja-JP/tools/plugin)：Plugin のインストールと登録。
