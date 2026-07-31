---
read_when:
    - デフォルトモデルを変更する、またはプロバイダーの認証状態を確認する場合
    - 利用可能なモデル／プロバイダーを調べ、認証プロファイルをデバッグする場合
summary: '`openclaw models` の CLI リファレンス（status/list/set/scan、エイリアス、フォールバック、認証）'
title: モデル
x-i18n:
    generated_at: "2026-07-26T10:08:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f7405c25694f04afe9c3029a8af64ae3ae7e1bdcf4c4ac31b8b84ff512d6a90e
    source_path: cli/models.md
    workflow: 16
---

# `openclaw models`

モデルの検出、スキャン、設定（デフォルトモデル、フォールバック、認証プロファイル）。

関連項目:

- プロバイダーとモデル: [モデル](/ja-JP/providers/models)
- モデル選択の概念と `/models` スラッシュコマンド: [モデルの概念](/ja-JP/concepts/models)
- プロバイダー認証のセットアップ: [はじめに](/ja-JP/start/getting-started)

## よく使用するコマンド

```bash
openclaw models status
openclaw models list
openclaw models set <model-or-alias>
openclaw models set-image <model-or-alias>
openclaw models scan
```

`status` および `auth` サブコマンドでは、設定済みエージェントを対象にするために `--agent <id>` を指定できます。`list`、`scan`、`aliases`、および `fallbacks`/`image-fallbacks` は常に設定済みのデフォルトエージェントを使用し、`set`/`set-image` は `--agent` を一切受け付けません。省略した場合、`--agent` 対応コマンドは、設定されていれば `OPENCLAW_AGENT_DIR` を使用し、それ以外の場合は設定済みのデフォルトエージェントを使用します。

### ステータス

`openclaw models status` は、解決済みのデフォルトおよびフォールバックと、認証の概要を表示します。Codex など、Plugin が所有するエージェントランタイムについては、所有元の Plugin が有効であり、起動ペイロード検証に合格したかどうかも確認します。有効な認証情報があってもランタイムを利用できないルートでは、`usable` ではなく `status: unavailable` が報告されます。JSON 出力には、個別の `authStatus`、`runtimeStatus`、および制限付きのランタイム診断が含まれます。プロバイダー使用状況のスナップショットを利用できる場合、OAuth/API キーのステータスセクションには、プロバイダーの使用期間とクォータのスナップショットが含まれます。現在の使用期間対応プロバイダーは、Anthropic、GitHub Copilot、Gemini CLI、OpenAI、MiniMax、Xiaomi、z.ai です。使用状況の認証には、利用可能な場合はプロバイダー固有のフックが使用されます。それ以外の場合、OpenClaw は認証プロファイル、環境、または設定にある一致する OAuth/API キー認証情報へフォールバックします。

`--json` 出力では、`auth.providers` は環境変数、設定、ストアを考慮したプロバイダーの概要であり、`auth.oauth` は認証ストアのプロファイルの健全性のみを示します。

オプション:

| フラグ                      | 効果                                                                                                                                   |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `--json`                  | JSON 出力。標準出力を `jq` にパイプ可能な状態に保つため、認証プロファイル、プロバイダー、および起動診断は標準エラー出力へ送られます。                            |
| `--plain`                 | プレーンテキスト出力。                                                                                                                       |
| `--check`                 | 認証が期限切れ間近または期限切れの場合、あるいは選択したエージェントランタイムが利用できない場合に、ゼロ以外で終了します: `1` = 利用不可/期限切れ/欠落、`2` = 期限切れ間近。 |
| `--probe`                 | 設定済み認証プロファイルのライブプローブ。実際のリクエストを行うため、トークンを消費したりレート制限を引き起こしたりする可能性があります。                                       |
| `--probe-provider <name>` | 1 つのプロバイダーのみをプローブします。                                                                                                                 |
| `--probe-profile <id>`    | 特定の認証プロファイル ID をプローブします（繰り返し指定またはカンマ区切り）。                                                                             |
| `--probe-timeout <ms>`    | プローブごとのタイムアウト。                                                                                                                       |
| `--probe-concurrency <n>` | 同時プローブ数。                                                                                                                       |
| `--probe-max-tokens <n>`  | プローブの最大トークン数（ベストエフォート）。                                                                                                          |
| `--agent <id>`            | 設定済みエージェント ID。`OPENCLAW_AGENT_DIR` を上書きします。                                                                                     |

プローブ行は、認証プロファイル、環境の認証情報、または `models.json` から生成される場合があります。プローブステータスの分類: `ok`、`auth`、`rate_limit`、`billing`、`timeout`、`format`、`unknown`、`no_model`。

プローブがモデル呼び出しに到達しない場合に想定される詳細/理由コード:

- `excluded_by_auth_order`: 保存済みプロファイルは存在しますが、明示的な `auth.order.<provider>` で省略されたため、プローブは試行せず除外を報告します。
- `missing_credential`、`invalid_expires`、`expired`、`unresolved_ref`: プロファイルは存在しますが、適格でないか解決できません。
- `ineligible_profile`: 別の理由により、プロファイルがプロバイダー設定と互換性を持ちません。
- `no_model`: プロバイダー認証は存在しますが、OpenClaw はそのプロバイダーでプローブ可能なモデル候補を解決できませんでした。

OpenAI ChatGPT/Codex OAuth のトラブルシューティングでは、`openclaw models status`、`openclaw models auth list --provider openai`、および `openclaw config get agents.defaults.model --json` を使用することで、エージェントがネイティブ Codex ランタイム経由で `openai/*` に使用可能な `openai` OAuth プロファイルを持っているかどうかを最も迅速に確認できます。[OpenAI プロバイダーのセットアップ](/ja-JP/providers/openai#check-and-recover-codex-oauth-routing)を参照してください。

### 一覧

`openclaw models list` は読み取り専用です。設定、認証プロファイル、既存のカタログ状態、およびプロバイダー所有のカタログ行を読み取りますが、`models.json` を書き換えることはありません。

オプション: `--all`（完全なカタログ）、`--local`（ローカルモデルのみに絞り込み）、`--provider <id>`、`--json`、`--plain`。

注:

- `Auth` 列は読み取り専用です。OpenAI などのプロバイダー所有モデルルートでは、各行の API/ベース URL ルートを、有効な `auth.order`、環境変数/設定の認証情報、および解決済みのコマンドスコープ SecretRef に含まれる適格なプロファイルと照合します。具体的な OpenAI 行は、そのルートポリシーを利用できない場合、プロバイダーレベルの認証を借用せず不明のままになります。プロバイダーのみのレガシーチェックおよびその他のプロバイダーでは、プロバイダーレベルの動作が維持されます。Plugin の合成認証メタデータはランタイム機能のヒントにすぎず、ネイティブアカウント認証の証明ではありません。そのため、アカウントに依存するルートは、レジストリに肯定的な証拠がない限り不明のままです。このコマンドは、プロバイダーランタイムの読み込み、キーチェーンのシークレットの読み取り、プロバイダー API の呼び出し、または正確な実行準備状況の証明を行いません。
- `models list --all --provider <id>` には、そのプロバイダーでまだ認証していない場合でも、Plugin マニフェストまたはバンドルされたプロバイダーカタログのメタデータに含まれる、プロバイダー所有の静的カタログ行が含まれる場合があります。一致する認証が設定されるまでは、それらの行も利用不可として表示されます。
- `models list` は、プロバイダーカタログの検出が遅い場合でもコントロールプレーンの応答性を維持します。デフォルトビューと設定済みビューは、短時間待機した後に設定済みまたは合成モデル行へフォールバックし、バックグラウンドで検出を完了させます。検出された正確で完全なカタログが必要で、プロバイダー検出の完了を待機できる場合は、`--all` を使用してください。
- 広範な `models list --all` は、プロバイダーランタイムの補足フックを読み込まずに、レジストリ行へマニフェストカタログ行をマージします。プロバイダーで絞り込んだマニフェストの高速パスでは、`static` とマークされたプロバイダーのみが使用されます。`refreshable` とマークされたプロバイダーはレジストリ/キャッシュを引き続き基盤とし、マニフェスト行を補足として追加します。一方、`runtime` とマークされたプロバイダーは、引き続きレジストリ/ランタイム検出を使用します。
- `models list` は、ネイティブモデルのメタデータとランタイム上限を区別して保持します。表形式の出力では、有効なランタイム上限がネイティブのコンテキストウィンドウと異なる場合、`Ctx` に `contextTokens/contextWindow` が表示されます。プロバイダーがその上限を公開している場合、JSON 行には `contextTokens` が含まれます。
- プロバイダー所有のルートでは、`models list` は 1 つの論理プロバイダー/モデル行を選択されたルートへ投影します。`Input` と `Ctx` は、完全に一致する物理ルートのカタログ行からのみ取得され、明示的に設定された論理オーバーライドが最後に適用されます。ルート選択を解決できない場合、兄弟ルートのメタデータを借用せず、機能フィールドは不明として表示されます。
- `models list --provider <id>` は、`moonshot` や `openai` などのプロバイダー ID で絞り込みます。`Moonshot AI` など、対話型プロバイダー選択画面の表示ラベルは受け付けません。
- モデル参照は、**最初の** `/` で分割して解析されます。モデル ID に `/` が含まれる場合（OpenRouter 形式）は、プロバイダーのプレフィックスを含めてください（例: `openrouter/moonshotai/kimi-k2`）。
- プロバイダーを省略すると、OpenClaw は最初に入力をエイリアスとして解決し、次にその正確なモデル ID に一意に一致する設定済みプロバイダーとして解決します。その後に限り、非推奨警告を表示して設定済みのデフォルトプロバイダーへフォールバックします。そのプロバイダーが設定済みのデフォルトモデルを公開しなくなった場合、OpenClaw は削除済みプロバイダーの古いデフォルトを表示する代わりに、最初の設定済みプロバイダー/モデルへフォールバックします。
- `models status` は、シークレットではないプレースホルダー（例: `OPENAI_API_KEY`、`secretref-managed`、`minimax-oauth`、`oauth:chutes`、`ollama-local`）について、シークレットとしてマスクする代わりに、認証出力に `marker(<value>)` を表示する場合があります。

### デフォルトモデル / 画像モデルの設定

```bash
openclaw models set <model-or-alias>
openclaw models set-image <model-or-alias>
```

`set` は `agents.defaults.model.primary` を書き込み、`set-image` は `agents.defaults.imageModel.primary` を書き込みます。どちらも `provider/model` または設定済みのエイリアスを受け付けます。`set` は、新しく選択したモデルで必要な場合に Codex/Copilot ランタイム Plugin のインストールも修復しますが、`set-image` は修復しません。どちらのコマンドも `--agent` を受け付けず、常にエージェントのデフォルトを書き込みます。

### スキャン

`models scan` は OpenRouter の公開 `:free` カタログを読み取り、フォールバック用途の候補をランク付けします。カタログ自体は公開されているため、メタデータのみのスキャンには OpenRouter キーは不要です。

デフォルトでは、OpenClaw はライブモデル呼び出しを使用してツールと画像の対応状況をプローブしようとします。OpenRouter キーが設定されていない場合、コマンドはメタデータのみの出力へフォールバックし、`:free` モデルでのプローブと推論には引き続き `OPENROUTER_API_KEY` が必要であることを説明します。

オプション:

- `--no-probe`（メタデータのみ。設定/シークレットの参照なし）
- `--min-params <b>`
- `--max-age-days <days>`
- `--provider <name>`
- `--max-candidates <n>`
- `--timeout <ms>`（カタログリクエストおよびプローブごとのタイムアウト）
- `--concurrency <n>`
- `--yes`
- `--no-input`
- `--set-default`
- `--set-image`
- `--json`

`--set-default` と `--set-image` にはライブプローブが必要です。メタデータのみのスキャン結果は参考情報であり、設定には適用されません。

## エイリアス

```bash
openclaw models aliases list [--json] [--plain]
openclaw models aliases add <alias> <model-or-alias>
openclaw models aliases remove <alias>
```

エイリアスは、モデルエントリごとに `agents.defaults.models.<key>.alias` として保存されます。`add` は最初に `<model-or-alias>` を正規のプロバイダー/モデルキーへ解決するため、エイリアスに別のエイリアスを設定すると、チェーン化するのではなく参照先が変更されます。
エイリアスを追加しても、`agents.defaults.modelPolicy.allow` は変更されず、モデルのオーバーライドも制限されません。

## フォールバック

```bash
openclaw models fallbacks list [--json] [--plain]
openclaw models fallbacks add <model-or-alias>
openclaw models fallbacks remove <model-or-alias>
openclaw models fallbacks clear
```

`agents.defaults.model.fallbacks` を管理します。`openclaw models image-fallbacks list|add|remove|clear` は、同じサブコマンド形式で並列の `agents.defaults.imageModel.fallbacks` リストを管理します。

## 認証プロファイル

```bash
openclaw models auth add
openclaw models auth list [--provider <id>] [--json]
openclaw models auth login --provider <id>
openclaw models auth login --provider openai --profile-id openai:work
openclaw models auth login-github-copilot
openclaw models auth paste-api-key --provider <id>
openclaw models auth setup-token --provider <id>
openclaw models auth paste-token --provider <id>
openclaw models auth order get --provider <id>
openclaw models auth order set --provider <id> <profileIds...>
openclaw models auth order clear --provider <id>
```

`models auth add` は対話型認証ヘルパーです。選択したプロバイダーに応じて、プロバイダーの認証フロー（OAuth/API キー）を開始するか、トークンを手動で貼り付ける手順を案内します。

`models auth list` は、トークン、API キー、OAuth シークレットの内容を表示せずに、選択したエージェントに保存されている認証プロファイルを一覧表示します。`--provider <id>` を使用すると、`openai` などの単一プロバイダーに絞り込めます。スクリプトでは `--json` を使用します。

`models auth login` は、プロバイダー Plugin の認証フロー（OAuth/API キー）を実行します。インストールされているプロバイダーを確認するには、`openclaw plugins list` を使用します。`login` では、ログイン時に名前付きプロファイルをサポートするプロバイダー向けの `--profile-id <id>`（同じプロバイダーへの複数のログインを分けて保持する場合に使用）、特定の認証方式を選択する `--method <id>`、`--method device-code` のショートカットである `--device-code`、プロバイダーが推奨するデフォルトモデルを適用する `--set-default`、およびそのプロバイダーの既存プロファイルを先に削除する `--force`（キャッシュされた OAuth プロファイルが機能しない場合や、アカウントを切り替える場合に使用）を指定できます。

`models auth login-github-copilot` は `models auth login --provider github-copilot --method device`（GitHub デバイスフロー）のショートカットです。`--yes` を指定すると、確認プロンプトを表示せずに既存のプロファイルを上書きします。

認証結果を設定済みの特定エージェントストアに書き込むには、`openclaw models auth --agent <id> <subcommand>` を使用します。親の `--agent` フラグは、`add`、`list`、`login`、`paste-api-key`、`setup-token`、`paste-token`、`login-github-copilot`、および `order get`/`set`/`clear` で有効です。

OpenAI モデルの場合、`--provider openai` のデフォルトは ChatGPT/Codex アカウントへのログインです。OpenAI API キープロファイルを追加する場合にのみ `--method api-key` を使用します。通常は、Codex サブスクリプションの上限に備えたバックアップとして使用します。以前のレガシーな OpenAI Codex プレフィックスの認証/プロファイル状態を `openai` に移行するには、`openclaw doctor --fix` を実行します。

例:

```bash
openclaw models auth login --provider openai --set-default
openclaw models auth login --provider openai --method api-key
openclaw models auth paste-api-key --provider openai
openclaw models auth list --provider openai
```

注:

- `paste-api-key` は別の場所で生成された API キーを受け付け、キーの値を入力するよう求めます。`--profile-id` を指定しない限り、デフォルトのプロファイル ID `<provider>:manual` に書き込みます。自動化では、たとえば `printf "%s\n" "$OPENAI_API_KEY" | openclaw models auth paste-api-key --provider openai` のように、標準入力からキーをパイプで渡します。
- `setup-token` と `paste-token` は、トークン認証方式を公開するプロバイダー向けの汎用トークンコマンドとして引き続き使用できます。
- `setup-token` には対話型 TTY が必要で、プロバイダーのトークン認証方式を実行します（プロバイダーが `setup-token` 方式を公開している場合は、それがデフォルトになります）。
- `paste-token` には `--provider` が必要です。デフォルトではトークンの値を入力するよう求め、`--profile-id` を指定しない限り、デフォルトのプロファイル ID `<provider>:manual` に書き込みます。自動化では、プロバイダーの認証情報がシェル履歴やプロセス一覧に表示されないように、トークンを引数として渡すのではなく、標準入力からパイプで渡します。
- `paste-token --expires-in <duration>` は、`365d` や `12h` などの相対期間から算出したトークンの絶対有効期限を保存します。
- `openai` では、OpenAI API キーと ChatGPT/OAuth トークンの内容は異なる認証形式です。`sk-...` OpenAI API キーには `paste-api-key` を使用し、トークン認証の内容にのみ `paste-token` を使用します。
- Anthropic: `setup-token`/`paste-token` は、`anthropic` 向けにサポートされている OpenClaw の認証経路ですが、ホストで Claude CLI（`claude -p`）を利用できる場合、OpenClaw はその再利用を優先します。
- `auth order get/set/clear` は、1 つのプロバイダーについて、エージェントごとの認証プロファイル順序の上書きを管理します。これは `auth-state.json` に保存されます（`auth.order.<provider>` 設定キーとは別です）。`set` は、優先順位順に 1 つ以上のプロファイル ID を受け取ります。`clear` は、設定/ラウンドロビン順序にフォールバックします。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [モデルの選択](/ja-JP/concepts/model-providers)
- [モデルのフェイルオーバー](/ja-JP/concepts/model-failover)
