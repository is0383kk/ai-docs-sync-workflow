---
read_when:
    - exec ツールの使用または変更
    - 標準入力または TTY の動作をデバッグする
summary: Exec ツールの使用方法、標準入力モード、TTY サポート
title: 実行ツール
x-i18n:
    generated_at: "2026-07-26T09:47:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9c16b5122c527c069a4d1a0c1649726073339e95b9084100c1a0f45ebcae759d
    source_path: tools/exec.md
    workflow: 16
---

ワークスペースでシェルコマンドを実行します。`exec` は変更を伴うシェル実行面です。コマンドは、選択したホストまたはサンドボックスのファイルシステムで許可されている任意の場所に、ファイルを作成、編集、または削除できます。`write`、`edit`、`apply_patch` などの OpenClaw ファイルシステムツールを無効にしても、`exec` は読み取り専用にはなりません。

`process` を介したフォアグラウンド実行とバックグラウンド実行をサポートします。`process` が許可されていない場合、`exec` は同期的に実行され、`yieldMs`/`background` を無視します。バックグラウンドセッションのスコープはエージェント単位です。`process` から確認できるのは、同じエージェントのセッションのみです。

## パラメーター

<ParamField path="command" type="string" required>
実行するシェルコマンド。
</ParamField>

<ParamField path="workdir" type="string" default="cwd">
コマンドの作業ディレクトリ。
</ParamField>

<ParamField path="env" type="object">
継承した環境に上書き統合する、キーと値の環境変数。
</ParamField>

<ParamField path="yieldMs" type="number" default="10000">
この遅延時間（ミリ秒）の後、コマンドを自動的にバックグラウンド化します。
</ParamField>

<ParamField path="background" type="boolean" default="false">
`yieldMs` を待たず、コマンドを直ちにバックグラウンド化します。
</ParamField>

<ParamField path="timeout" type="number" default="tools.exec.timeoutSeconds">
この呼び出しに設定された exec タイムアウトを秒単位で上書きします。フォアグラウンド、バックグラウンド、`yieldMs`、gateway、サンドボックス、および node の `system.run` 実行に適用されます。`timeout: 0` は、その呼び出しの exec プロセスタイムアウトを無効にします。
</ParamField>

<ParamField path="pty" type="boolean" default="false">
利用可能な場合、疑似端末で実行します。TTY 専用 CLI、コーディングエージェント、およびターミナル UI に使用します。
</ParamField>

<ParamField path="host" type="'auto' | 'sandbox' | 'gateway' | 'node'" default="auto">
実行場所。サンドボックスランタイムがアクティブな場合、`auto` は `sandbox` に解決され、それ以外の場合は `gateway` に解決されます。
</ParamField>

<ParamField path="security" type="'deny' | 'allowlist' | 'full'">
通常のツール呼び出しでは無視されます。`gateway`/`node` のセキュリティは、`tools.exec.mode` とホストの承認ファイルから導出されます。昇格モードで完全アクセスを強制できるのは、オペレーターが昇格アクセスを明示的に許可した場合に限られます。
</ParamField>

<ParamField path="ask" type="'off' | 'on-miss' | 'always'">
基本の確認モードは、`tools.exec.mode` とホストの承認設定から導出されます。チャンネル起点のモデル呼び出しでは、有効なホスト確認設定が `off` の場合、呼び出しごとの `ask` は無視されます。それ以外の場合は、より厳格なモードへの強化のみが可能です。
</ParamField>

<ParamField path="node" type="string">
`host=node` の場合の Node ID／名前。
</ParamField>

<ParamField path="elevated" type="boolean" default="false">
昇格モードを要求します。サンドボックスを抜け、設定済みのホストパスで実行します。昇格が `full` に解決された場合に限り、`security=full` が強制されます。
</ParamField>

注:

- `host` が受け付けるのは、`auto`、`sandbox`、`gateway`、または `node` のみです。これはホスト名セレクターではありません。ホスト名のような値は、コマンドの実行前に拒否されます。
- 呼び出しごとの `host=node` は `auto` から許可されます。呼び出しごとの `host=gateway` が許可されるのは、サンドボックスランタイムがアクティブでない場合に限られます。
- 追加設定がなくても、`host=auto` は引き続き「そのまま動作」します。サンドボックスがない場合は `gateway` に解決され、稼働中のサンドボックスがある場合はサンドボックス内に留まります。
- `elevated` はサンドボックスを抜け、設定済みのホストパスで実行します。デフォルトでは `gateway`、`tools.exec.host=node`（またはセッションのデフォルト）が `host=node` の場合は `node` です。現在のセッション／プロバイダーで昇格アクセスが有効な場合にのみ使用できます。
- `gateway`/`node` の承認は、ホストの承認ファイルによって制御されます。
- `node` には、ペアリング済みの Node（コンパニオンアプリまたはヘッドレス Node ホスト）が必要です。複数の Node が利用可能な場合は、`exec.node` または `tools.exec.node` を設定して選択します。
- `exec host=node` は、Node でシェルを実行する唯一の経路です。従来の `nodes.run` ラッパーは削除されています。
- Windows 以外のホストでは、exec は `SHELL` が設定されている場合にそれを使用します。`SHELL` が `fish` の場合は、fish と互換性のない bash 構文を避けるため、`PATH` にある `bash`（または `sh`）を優先し、どちらも存在しない場合は `SHELL` にフォールバックします。
- Windows ホストでは、exec は PowerShell 7（`pwsh`）の検出（Program Files、ProgramW6432、PATH の順）を優先し、その後 Windows PowerShell 5.1 にフォールバックします。
- Windows 以外の gateway ホストでは、bash および zsh の exec コマンドは起動時スナップショットを使用します。OpenClaw は、source 可能なエイリアス／関数と、小規模で安全な環境変数のセットをシェル起動ファイルから `$OPENCLAW_STATE_DIR/cache/shell-snapshots/` に取り込み、各 exec コマンドの前にそのスナップショットを source します。シークレットと思われる変数は除外されます。サンドボックスおよび Node の exec は、このスナップショットを使用しません。このスナップショット経路を無効にするには、Gateway プロセス環境で `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` を設定します。
- ホスト実行（`gateway`/`node`）では、バイナリの乗っ取りやコードの注入を防ぐため、`env.PATH` とローダーの上書き（`LD_*`/`DYLD_*`）が拒否されます。
- OpenClaw は、起動したコマンドの環境（PTY およびサンドボックス実行を含む）に `OPENCLAW_SHELL=exec` を設定し、シェル／プロファイルのルールが exec ツールのコンテキストを検出できるようにします。
- チャンネル起点の実行では、チャンネルからそれらの ID が提供された場合、OpenClaw は送信者／チャットの ID を含む限定的な JSON ペイロードも `OPENCLAW_CHANNEL_CONTEXT` で公開します。
- `exec` では、`openclaw channels login` または `/approve` のシェルコマンドを実行できません。`openclaw channels login` は対話型のチャンネル認証フローであり、`/approve` はシェルではなく承認コマンドハンドラーを経由する必要があります。チャンネルログインは gateway ホストのターミナルで実行するか、存在する場合はチャンネル固有のログインエージェントツール（例: `whatsapp_login`）を使用します。
- 重要: サンドボックスは**デフォルトで無効**です。サンドボックスが無効の場合、暗黙の `host=auto` は `gateway` に解決されます。明示的な `host=sandbox` は、gateway ホストで暗黙に実行されるのではなく、引き続き安全側に失敗します。サンドボックスを有効にするか、承認付きで `host=gateway` を使用してください。
- スクリプトの事前チェック（一般的な Python／Node のシェル構文ミスを対象）は、有効な `workdir` 境界内のファイルのみを検査します。スクリプトパスが `workdir` の外部に解決される場合、そのファイルの事前チェックはスキップされます。また、`host=gateway` であり、有効なポリシーが `ask=off` を伴う `security=full` の場合、事前チェック全体がスキップされます。
- 今から開始する長時間の作業は一度だけ起動し、自動完了時のウェイクが有効で、コマンドが出力するか失敗した場合は、そのウェイクに任せます。ログ、ステータス、入力、または介入には `process` を使用してください。sleep ループ、タイムアウトループ、または繰り返しポーリングでスケジューリングを再現しないでください。
- エージェントが開始したバックグラウンドコマンドは、完了するまで Web、iOS、Android のバックグラウンドタスクビューに表示されます。完了 Heartbeat がエージェントを再度ウェイクする前に、タスク台帳が確定されます。
- 後で、またはスケジュールに従って実行する必要がある作業には、`exec` の sleep／delay パターンではなく cron を使用してください。

## 設定

| キー                                  | デフォルト                  | 注記                                                                                                                                                   |
| ------------------------------------ | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tools.exec.timeoutSeconds`          | `1800`                   | コマンドごとのデフォルトの exec タイムアウト（秒）。呼び出しごとの `timeout` はこれを上書きし、呼び出しごとの `timeout: 0` は exec プロセスタイムアウトを無効にします。                  |
| `tools.exec.host`                    | `auto`                   | サンドボックスランタイムがアクティブな場合は `sandbox` に、それ以外の場合は `gateway` に解決されます。                                                                            |
| `tools.exec.mode`                    | ホストから導出             | 正規のポリシー設定項目です。後述の[モード](#modes)を参照してください。                                                                                                       |
| `tools.exec.reviewer.model`          | 設定済みのエージェントのプライマリ | `mode=auto` レビュー用の、任意のプロバイダー／モデル上書き。                                                                                                |
| `tools.exec.reviewer.timeoutMs`      | `30000`                  | 人間へのフォールバック前に行う、レビューモデルの準備および完了に対するステージごとのタイムアウト。                                                                  |
| `tools.exec.node`                    | 未設定                    |                                                                                                                                                         |
| `tools.exec.notifyOnExit`            | `true`                   | true の場合、バックグラウンド化された exec セッションは終了時にシステムイベントをキューに追加し、Heartbeat を要求します。                                                           |
| `tools.exec.approvalRunningNoticeMs` | `10000`                  | 承認が必要な exec がこの時間を超えて実行された場合、「実行中」の通知を 1 回だけ送信します（`0` で無効化）。                                                        |
| `tools.exec.strictInlineEval`        | `false`                  | [インライン評価](#inline-eval-strictinlineeval)を参照してください。                                                                                                       |
| `tools.exec.commandHighlighting`     | `false`                  | true の場合、承認プロンプトで、パーサーが抽出したコマンド範囲をコマンドテキスト内で強調表示できます。グローバルまたはエージェントごとに設定できます。承認ポリシーは変更されません。 |
| `tools.exec.pathPrepend`             | 未設定                    | exec 実行時に `PATH` の先頭へ追加するディレクトリの一覧（gateway とサンドボックスのみ）。                                                                        |
| `tools.exec.safeBins`                | 未設定                    | 明示的な許可リスト項目なしで実行できる、標準入力専用の安全なバイナリ。[安全なバイナリ](/ja-JP/tools/exec-approvals-advanced#safe-bins-stdin-only)を参照してください。         |
| `tools.exec.safeBinTrustedDirs`      | `/bin`, `/usr/bin`       | `safeBins` のパスチェックで信頼する追加の明示的なディレクトリ。`PATH` の項目が自動的に信頼されることはありません。                                              |
| `tools.exec.safeBinProfiles`         | 未設定                    | 安全なバイナリごとの任意のカスタム argv ポリシー（`minPositional`、`maxPositional`、`allowedValueFlags`、`deniedFlags`）。                                        |

承認なしのホスト exec は、gateway と Node（`mode=full`）のデフォルトです。これは `host=auto` ではなく、ホストポリシーのデフォルトに由来します。承認／許可リストの動作が必要な場合は、`tools.exec.mode` を設定してホストの承認ファイルを厳格化してください。[Exec の承認](/ja-JP/tools/exec-approvals#yolo-mode-no-approval)を参照してください。サンドボックスの状態に関係なく gateway または Node へのルーティングを強制するには、`tools.exec.host` を設定するか、`/exec host=...` を使用します。

例:

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### モード

`tools.exec.mode` は、永続化される正規のポリシー設定項目です。実行時のセキュリティおよび承認動作は、これから導出されます。

| モード        | security    | ask       | 動作                                                                                                                       |
| ----------- | ----------- | --------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `deny`      | `deny`      | `off`     | Exec は拒否されます。                                                                                                                |
| `allowlist` | `allowlist` | `off`     | 許可リストに登録されたコマンドまたは安全なバイナリのコマンドだけが実行され、それ以外について確認は行われません。                                                                 |
| `ask`       | `allowlist` | `on-miss` | 許可リストに一致するものは直接実行され、それ以外はすべて人間に確認されます。                                                                  |
| `auto`      | `allowlist` | `on-miss` | 許可リストまたは安全なバイナリに一致するものは直接実行され、それ以外はすべて、人間に確認する前に OpenClaw のネイティブ自動レビュー担当を経由します。 |
| `full`      | `full`      | `off`     | 承認ゲートはありません。                                                                                                              |

セッション単位の `/exec ask=always` では、永続化されたモードにかかわらず、引き続き毎回人間に確認します。

自動レビューによる承認は1回限りです。Gateway では、OpenClaw が解決済みの実行可能ファイルパスをレビュー担当に渡し、実行をその同じパスに固定します。ヒアドキュメント、シェル展開、サポートされていないラッパーのクォートなど、強制可能な単一の実行計画に還元できないコマンドは、モデルがそれ以外の点では許可するとしても、人間による承認にフォールバックします。

明示的なランタイムポリシーまたはネイティブポリシーですでに決定されていない Codex app-server のコマンド承認には、人間による承認経路が使用されます。Codex は、レビューの決定を Codex が実行するコマンドに結び付けられる、強制可能な解決済み実行可能ファイルを公開しないため、OpenClaw はこれらのリクエストに対して設定済みの Exec レビュー担当を実行しません。

### インライン評価 (`strictInlineEval`)

`tools.exec.strictInlineEval` が `true` の場合、インラインのインタープリター評価形式には、レビュー担当または明示的な承認が必要です。対象には、`python -c`、`node -e`、`ruby -e`、`perl -e`、`php -r`、`lua -e`、`osascript -e`、およびサポートされている他のインタープリターやコマンドキャリアにおける同様の形式（`awk`、`find -exec`、`make`、`sed`、`xargs` など）が含まれます。`mode=auto` では、通常の Exec 承認経路により、明らかに低リスクな単発コマンドをネイティブ自動レビュー担当が許可する場合があります。Node ホストを直接呼び出す `system.run` は、コマンドを人間による承認経路に渡せないため、引き続き明示的な承認が必要です。レビュー担当が確認を求めた場合、リクエストは人間に送られます。`allow-always` では無害なインタープリターやスクリプトの呼び出しを引き続き永続化できますが、インライン評価形式が永続的な許可ルールになることはありません。

### PATH の処理

- `host=gateway`: ログインシェルの `PATH` を Exec 環境にマージします。ホスト実行では `env.PATH` のオーバーライドが拒否されます。デーモン自体は引き続き最小限の `PATH` で実行されます。
  - macOS: `/opt/homebrew/bin`、`/usr/local/bin`、`/usr/bin`、`/bin`
  - Linux: `/usr/local/bin`、`/usr/bin`、`/bin`
  - ユーザーのシェル設定（`~/.zshenv` や `/etc/zshenv` など）が起動時に優先パスを上書きするのを防ぐため、実行直前にシェルコマンド内で `tools.exec.pathPrepend` のエントリが最終的な `PATH` の先頭に安全に追加されます。
- `host=sandbox`: コンテナ内で `sh -lc`（ログインシェル）を実行するため、`/etc/profile` によって `PATH` がリセットされる場合があります。OpenClaw はプロファイルの読み込み後、内部環境変数を介して `env.PATH` を先頭に追加します（シェル展開は行われません）。ここでも `tools.exec.pathPrepend` が適用されます。
- `host=node`: 渡した環境変数オーバーライドのうち、ブロックされていないものだけが Node に送信されます。ホスト実行では `env.PATH` のオーバーライドが拒否され、Node ホストでは無視されます。Node に PATH エントリを追加する必要がある場合は、Node ホストサービスの環境（systemd/launchd）を設定するか、標準の場所にツールをインストールしてください。

エージェント単位の Node バインディング（設定ではキー付きエージェント ID を使用）:

```bash
openclaw config get agents.entries
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
```

Control UI: **デバイス**ページには、同じ設定用の小さな「Exec Node バインディング」パネルがあります。

## セッションオーバーライド (`/exec`)

`/exec` を使用して、`host`、`security`、`ask`、`node` の**セッション単位**のデフォルトを設定します。現在の値を表示するには、引数なしで `/exec` を送信します。

例:

```text
/exec host=auto security=allowlist ask=on-miss node=mac-1
```

`/exec` は、チャンネルの許可リスト、ペアリング、アクセスグループを通じた**認可済み送信者**に対してのみ有効です。アクセスグループの適用は常に有効です。これは**セッション状態のみ**を更新し、設定には書き込みません。認可済みの外部チャンネル送信者は、これらのセッションデフォルトを設定できます。内部の Gateway/Webchat クライアントがそれらを永続化するには、`operator.admin` が必要です。

Exec を完全に無効化するには、ツールポリシー（`tools.deny: ["exec"]` またはエージェント単位）で拒否します。`security=full` と `ask=off` を明示的に設定しない限り、ホスト承認は引き続き適用されます。

## Exec 承認（コンパニオンアプリ / Node ホスト）

サンドボックス化されたエージェントでは、Gateway または Node ホストで `exec` を実行する前に、リクエスト単位の承認を必須にできます。ポリシー、許可リスト、UI フローについては、[Exec 承認](/ja-JP/tools/exec-approvals)を参照してください。

人間による承認が必要な場合、Node ホストおよび非ネイティブ Gateway のフローは、`status: "approval-pending"` と承認 ID を伴って直ちに返ります。一方、ネイティブチャットおよび Web UI の Gateway フローは、インラインで待機し、承認後に最終的なコマンド結果を返すことができます。`approval-pending` の結果はコマンドがまだ開始されていないことを意味するため、フォアグラウンドへのフォールバック警告は、承認されたコマンドが実際にインラインで実行された場合にのみ表示されます。承認された非同期実行では、コマンドの進捗および完了のシステムイベント（`Exec running` / `Exec finished`）が発行されます。拒否またはタイムアウトした承認は終端状態となり、拒否システムイベントによってエージェントセッションを再開することはありません。

ネイティブの承認カードやボタンがあるチャンネルでは、エージェントはまずそのネイティブ UI を使用し、ツールの結果でチャット承認が利用できない、または手動承認が唯一の経路であると明示された場合にのみ、手動の `/approve` コマンドを含める必要があります。

## 許可リストと安全なバイナリ

手動の許可リスト適用では、解決済みバイナリパスの glob と、パスを含まないコマンド名の glob が照合されます。パスを含まない名前は PATH 経由で呼び出されたコマンドにのみ一致するため、コマンドが `rg` の場合、`rg` は `/opt/homebrew/bin/rg` に一致できますが、`./rg` や `/tmp/rg` には一致しません。

`security=allowlist` の場合、シェルコマンドは、すべてのパイプラインセグメントが許可リストに登録されているか、安全なバイナリである場合にのみ自動許可されます。連結（`;`、`&&`、`||`）とリダイレクトは、すべてのトップレベルセグメントが許可リスト（安全なバイナリを含む）を満たさない限り、許可リストモードでは拒否されます。リダイレクトは引き続きサポートされません。永続的な `allow-always` の信頼設定もこのルールを回避しません。連結されたコマンドでは、引き続きすべてのトップレベルセグメントが一致する必要があります。

`autoAllowSkills` は Exec 承認における独立した利便性のための経路であり、手動のパス許可リストエントリと同じものではありません。厳密で明示的な信頼が必要な場合は、`autoAllowSkills` を無効のままにしてください。

2つの制御は用途に応じて使い分けます。

- `tools.exec.safeBins`: 小規模な標準入力専用ストリームフィルター。
- `tools.exec.safeBinTrustedDirs`: 安全なバイナリの実行可能ファイルパス用に明示的に追加する信頼済みディレクトリ。
- `tools.exec.safeBinProfiles`: カスタムの安全なバイナリに対する明示的な argv ポリシー。
- 許可リスト: 実行可能ファイルパスに対する明示的な信頼。

`safeBins` を汎用的な許可リストとして扱わず、インタープリターやランタイムのバイナリ（たとえば `python3`、`node`、`ruby`、`bash`）を追加しないでください。それらが必要な場合は、明示的な許可リストエントリを使用し、承認プロンプトを有効のままにしてください。

`openclaw security audit` は、インタープリターやランタイムの `safeBins` エントリに明示的なプロファイルがない場合に警告し、`openclaw doctor --fix` は不足しているカスタム `safeBinProfiles` エントリの雛形を作成できます。`openclaw security audit` と `openclaw doctor` は、`jq` のように広範な動作を行うバイナリを `safeBins` に明示的に再追加した場合にも警告します（`jq` は環境データを読み取り、モジュールや起動ファイルから jq コードを読み込めるため、代わりに明示的な許可リストエントリまたは承認ゲート付きの実行を使用してください）。`jq` は、明示的に記載されていても安全なバイナリとして拒否されます。インタープリターを明示的に許可リストへ登録する場合は、インラインコード評価形式にレビュー担当または明示的な承認が引き続き必要となるよう、`tools.exec.strictInlineEval` を有効にしてください。

ポリシーの詳細と例については、[Exec 承認](/ja-JP/tools/exec-approvals-advanced#safe-bins-stdin-only)および[安全なバイナリと許可リストの比較](/ja-JP/tools/exec-approvals-advanced#safe-bins-versus-allowlist)を参照してください。

## 例

フォアグラウンド:

```json
{ "tool": "exec", "command": "ls -la" }
```

バックグラウンドとポーリング:

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

ポーリングはオンデマンドの状態確認用であり、待機ループ用ではありません。自動完了による再開が有効な場合、コマンドは出力を生成するか失敗したときにセッションを再開できます。

キーの送信（tmux 形式）:

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

送信（CR のみを送信）:

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

貼り付け（デフォルトではブラケットペースト）:

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch

`apply_patch` は、構造化された複数ファイル編集を行う `exec` のサブツールです。デフォルトで有効で、どのモデルプロバイダーでも利用できます。`allowModels` で制限できます。無効にする場合、または特定のモデルに制限する場合にのみ設定を使用してください。

```json5
{
  tools: {
    exec: {
      applyPatch: { workspaceOnly: true, allowModels: ["gpt-5.6-sol"] },
    },
  },
}
```

注:

- ツールポリシーは引き続き適用されます。`allow: ["write"]` は暗黙的に `apply_patch` を許可します。
- `deny: ["write"]` は `apply_patch` を拒否しません。`apply_patch` を明示的に拒否するか、パッチによる書き込みもブロックする場合は `deny: ["group:fs"]` を使用してください。
- 設定は `tools.exec.applyPatch` 配下にあります。
- `tools.exec.applyPatch.enabled` のデフォルトは `true` です。ツールを無効にするには `false` に設定します。
- `tools.exec.applyPatch.workspaceOnly` のデフォルトは `true`（ワークスペース内に限定）です。`apply_patch` がワークスペースディレクトリ外へ書き込みまたは削除することを意図する場合にのみ、`false` に設定してください。
- `tools.exec.applyPatch.allowModels` はモデル ID の任意の許可リストです（`gpt-5.4` のような生の ID、または `openai/gpt-5.4` のような完全な ID）。設定すると、一致するモデルだけがツールを使用できます。未設定の場合は、すべてのモデルが使用できます。

## 関連項目

- [Exec 承認](/ja-JP/tools/exec-approvals) — シェルコマンドの承認ゲート
- [サンドボックス化](/ja-JP/gateway/sandboxing) — サンドボックス環境でのコマンド実行
- [バックグラウンドプロセス](/ja-JP/gateway/background-process) — 長時間実行される Exec とプロセスツール
- [セキュリティ](/ja-JP/gateway/security) — ツールポリシーと昇格アクセス
