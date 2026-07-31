---
read_when:
    - 安全なバイナリまたはカスタムの安全なバイナリプロファイルの設定
    - 承認を Slack/Discord/Telegram またはその他のチャットチャンネルに転送する
    - チャネル向けネイティブ承認クライアントの実装
summary: 高度な exec 承認：安全なバイナリ、インタープリターのバインド、承認の転送、ネイティブ配信
title: Exec 承認 — 高度な設定
x-i18n:
    generated_at: "2026-07-26T09:21:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac90d41f867a8ae4f14b6c9c13f3732d102a65707f456623932b858145a9bf46
    source_path: tools/exec-approvals-advanced.md
    workflow: 16
---

高度な exec 承認のトピック: `safeBins` 高速パス、インタープリター/ランタイムの
バインディング、およびチャットチャネルへの承認転送（ネイティブ配信を含む）。
コアポリシーと承認フローについては、[Exec 承認](/ja-JP/tools/exec-approvals)を参照してください。

## セーフバイナリ（stdin のみ）

`tools.exec.safeBins` は、明示的な許可リストエントリなしで許可リストモードで
実行される **stdin のみ** のバイナリ（例: `cut`）を指定します。セーフバイナリは
位置指定のファイル引数とパス形式のトークンを拒否するため、入力ストリームのみを
操作できます。これはストリームフィルター向けの限定的な高速パスとして扱い、
一般的な信頼リストとしては扱わないでください。

<Warning>
インタープリターまたはランタイムのバイナリ（例: `python3`、`node`、
`ruby`、`bash`、`sh`、`zsh`）を `safeBins` に追加しては**なりません**。設計上、コードの評価、
サブコマンドの実行、またはファイルの読み取りが可能なコマンドには、明示的な許可リストエントリを
使用し、承認プロンプトを有効にしたままにしてください。カスタムセーフバイナリでは、
`tools.exec.safeBinProfiles.<bin>` に明示的なプロファイルを定義する必要があります。
</Warning>

デフォルトのセーフバイナリ:

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`、`uniq`、`head`、`tail`、`tr`、`wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` と `sort` はデフォルトのリストに含まれていません。オプトインする場合は、stdin を使用しない
ワークフロー用に明示的な許可リストエントリを維持してください。セーフバイナリモードの `grep` では、
パターンを `-e`/`--regexp` で指定してください。位置指定形式のパターンは拒否されるため、
ファイルオペランドを曖昧な位置引数として紛れ込ませることはできません。

### argv の検証と拒否されるフラグ

検証は argv の形状のみから決定論的に行われ（ホストファイルシステム上の存在確認は
行われません）、許可/拒否の差異からファイルの存在を推測するオラクル動作を防止します。
デフォルトのセーフバイナリではファイル指向のオプションが拒否されます。長い形式の
オプションはフェイルクローズで検証されます（不明なフラグと曖昧な省略形は
拒否されます）。デフォルトバイナリの認識済み読み取り専用ブールフラグ（例:
`wc -l`、`tr -d`、`uniq -c`）は受け入れられますが、認識されない短い形式のフラグは
フェイルクローズのままとなり、手動承認にフォールスルーします。

セーフバイナリプロファイルごとに拒否されるフラグ:

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`: `--dereference-recursive`、`--directories`、`--exclude-from`、`--file`、`--recursive`、`-R`、`-d`、`-f`、`-r`
- `jq`: `--argfile`、`--from-file`、`--library-path`、`--rawfile`、`--slurpfile`、`-L`、`-f`
- `sort`: `--compress-program`、`--files0-from`、`--output`、`--random-source`、`--temporary-directory`、`-T`、`-o`
- `tail`: `--follow`、`--retry`、`-F`、`-f`
- `wc`: `--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

セーフバイナリでは、stdin のみのセグメントについて、実行時に argv トークンを
**リテラルテキスト**として扱うことも強制されます（グロブ展開および `$VARS` 展開は行われません）。そのため、
`*` や `$HOME/...` のようなパターンを使用してファイル読み取りを紛れ込ませることはできません。`awk`、
`sed`、および `jq` は、そのセマンティクスが stdin のみに限定されることを
検証できないため、セーフバイナリとして常に拒否されます。`jq` は環境データを読み取り、
モジュールまたは起動ファイルから jq コードを読み込むことができます。これらのツールには
`safeBins` の代わりに、明示的な許可リストエントリまたは承認プロンプトを使用してください。

### 信頼済みバイナリディレクトリ

セーフバイナリは、信頼済みバイナリディレクトリ（システムのデフォルトと
オプションの `tools.exec.safeBinTrustedDirs`）から解決される必要があります。`PATH` のエントリが自動的に信頼されることはありません。
デフォルトの信頼済みディレクトリは意図的に最小限に設定されています: `/bin`、`/usr/bin`。セーフバイナリの
実行可能ファイルがパッケージマネージャー/ユーザーのパス（例:
`/opt/homebrew/bin`、`/usr/local/bin`、`/opt/local/bin`、`/snap/bin`）にある場合は、それらを
`tools.exec.safeBinTrustedDirs` に明示的に追加してください。

### シェルの連結、ラッパー、マルチプレクサー

各トップレベルセグメントが許可リストを満たす場合（セーフバイナリまたは Skills の
自動許可を含む）、シェルの連結（`&&`、`||`、`;`）が許可されます。許可リストモードでは、
リダイレクトは引き続きサポートされません。コマンド置換（`$()` / バッククォート）は、
二重引用符内を含め、許可リストの解析中に拒否されます。リテラルの `$()` テキストが
必要な場合は、一重引用符を使用してください。

macOS コンパニオンアプリの承認では、シェル制御または展開構文（`&&`、`||`、`;`、`|`、`` ` ``、`$`、`<`、`>`、`(`、`)`）を
含む生のシェルテキストは、シェルバイナリ自体が許可リストに登録されていない限り、
許可リスト不一致として扱われます。

シェルラッパー（`bash|sh|zsh ... -c/-lc`）では、リクエストスコープの環境変数オーバーライドが
小規模な明示的許可リスト（`TERM`、`LANG`、`LC_*`、`COLORTERM`、
`NO_COLOR`、`FORCE_COLOR`）に限定されます。

許可リストモードでの `allow-always` の決定では、透過的なディスパッチラッパー
（例: `env`、`flock`、`nice`、`nohup`、`stdbuf`、`timeout`）について、
ラッパーのパスではなく内部の実行可能ファイルのパスが永続化されます。シェルマルチプレクサー
（`busybox`、`toybox`）も、シェルアプレット（`sh`、`ash` など）について同様に
ラップ解除されます。ラッパーまたはマルチプレクサーを安全にラップ解除できない場合、
許可リストエントリは自動的に永続化されません。

`python3` や `node` のようなインタープリターを許可リストに登録する場合は、
インライン評価で引き続き明示的な承認が必要になるよう、`tools.exec.strictInlineEval=true` を推奨します。
厳格モードでは、`allow-always` によって無害なインタープリター/スクリプト呼び出しを
永続化できますが、インライン評価を運ぶ形式は自動的に永続化されません。

### セーフバイナリと許可リストの比較

| トピック            | `tools.exec.safeBins`                                  | 許可リスト（`exec-approvals.json`）                                                  |
| ---------------- | ------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| 目的             | 限定的な stdin フィルターを自動許可                        | 特定の実行可能ファイルを明示的に信頼                                               |
| 一致タイプ       | 実行可能ファイル名 + セーフバイナリ argv ポリシー            | 解決済み実行可能ファイルパスのグロブ、または PATH 経由で呼び出されるコマンド用のコマンド名のみのグロブ |
| 引数の範囲       | セーフバイナリプロファイルとリテラルトークン規則により制限     | デフォルトではパス一致。オプションの `argPattern` で解析済み argv を制限可能              |
| 一般的な例       | `head`、`tail`、`tr`、`wc`                             | `jq`、`python3`、`node`、`ffmpeg`、カスタム CLI                                     |
| 最適な用途       | パイプライン内の低リスクなテキスト変換                      | より広範な動作または副作用を持つあらゆるツール                                     |

設定場所:

- `safeBins` は設定（`tools.exec.safeBins` またはエージェントごとの `agents.entries.*.tools.exec.safeBins`）から取得されます。
- `safeBinTrustedDirs` は設定（`tools.exec.safeBinTrustedDirs` またはエージェントごとの `agents.entries.*.tools.exec.safeBinTrustedDirs`）から取得されます。
- `safeBinProfiles` は設定（`tools.exec.safeBinProfiles` またはエージェントごとの `agents.entries.*.tools.exec.safeBinProfiles`）から取得されます。エージェントごとのプロファイルキーはグローバルキーを上書きします。
- 許可リストエントリは、`agents.<id>.allowlist` 配下のホストローカルな承認ファイル（または Control UI / `openclaw approvals allowlist ...` 経由）に保存されます。
- `openclaw security audit` は、明示的なプロファイルがないインタープリター/ランタイムのバイナリが `safeBins` に存在すると、`tools.exec.safe_bins_interpreter_unprofiled` で警告します。
- `openclaw doctor --fix` は、欠落しているカスタム `safeBinProfiles.<bin>` エントリを `{}` として生成できます（その後に確認して制限を強化してください）。インタープリター/ランタイムのバイナリは自動生成されません。

カスタムプロファイルの例:

```json5
{
  tools: {
    exec: {
      safeBins: ["myfilter"],
      safeBinProfiles: {
        myfilter: {
          minPositional: 0,
          maxPositional: 0,
          allowedValueFlags: ["-n", "--limit"],
          deniedFlags: ["-f", "--file", "-c", "--command"],
        },
      },
    },
  },
}
```

## インタープリター/ランタイムコマンド

承認に基づくインタープリター/ランタイムの実行は、意図的に保守的です:

- 正確な argv/cwd/env コンテキストが常にバインドされます。
- 直接指定されたシェルスクリプトおよびランタイムファイルの形式は、ベストエフォートで単一の具体的なローカル
  ファイルスナップショットにバインドされます。
- 単一の直接的なローカルファイルに解決される一般的なパッケージマネージャーのラッパー形式（例:
  `pnpm exec`、`pnpm node`、`npm exec`、`npx`）は、バインド前にラップ解除されます。
- OpenClaw がインタープリター/ランタイムコマンドについて具体的なローカルファイルを正確に 1 つ特定できない場合
  （例: パッケージスクリプト、評価形式、ランタイム固有のローダーチェーン、または曖昧な複数ファイル
  形式）、実際には提供できない意味的な網羅性を主張する代わりに、承認に基づく実行が拒否されます。
- これらのワークフローでは、サンドボックス化、別のホスト境界、またはオペレーターがより広範な
  ランタイムセマンティクスを受け入れる明示的に信頼された許可リスト/完全なワークフローを推奨します。

承認が必要な場合、exec ツールは承認 ID を伴って直ちに返ります。その ID を使用して、
後続の承認済み実行システムイベント（`Exec finished`、および設定されている場合は `Exec running`）を
関連付けます。タイムアウトまでに決定が届かない場合、リクエストは承認タイムアウトとして扱われ、
最終的なホストコマンド拒否として通知されます。発生元セッションを持つメインエージェントの
非同期承認では、OpenClaw は内部フォローアップによってそのセッションも再開します。これにより、
エージェントは後から欠落した結果を修復するのではなく、コマンドが実行されなかったことを認識できます。
保留中の exec 承認は、デフォルトで 30 分後に期限切れになります。

### フォローアップ配信の動作

承認された非同期 exec が完了すると、OpenClaw は同じセッションにフォローアップの `agent` ターンを送信します。
拒否された非同期承認では、拒否ステータスについて同じメインセッションのフォローアップパスが使用されますが、
昇格されたランタイムハンドオフは登録されず、コマンドも実行されません。再開可能なメインセッションがない
拒否は、抑制されるか、安全な直接ルートが存在する場合はそのルートを通じて報告されます。

- 有効な外部配信先（配信可能なチャネルとターゲット `to`）が存在する場合、フォローアップ配信にはそのチャネルが使用されます。
- 外部ターゲットがない Web チャットのみのフローまたは内部セッションフローでは、フォローアップ配信はセッション内のみに留まります（`deliver: false`）。
- 解決可能な外部チャネルがない状態で呼び出し元が厳格な外部配信を明示的に要求した場合、リクエストは `INVALID_REQUEST` で失敗します。
- `bestEffortDeliver` が有効で、外部チャネルを解決できない場合、配信は失敗する代わりにセッション内のみにダウングレードされます。

## サードパーティークライアントの最小スコープ

Gateway の承認解決は、専用の `operator.approvals` スコープによって保護されます。これは、所有者固有の `exec.approval.resolve` メソッドと種類に依存しない `approval.resolve` メソッドの両方に適用され、`operator.write` には包含されません。ダッシュボードと統合機能は、使用するメソッドに必要なスコープのみを要求する必要があります。承認解決へのアクセスはリモート実行と同等の権限として扱い、クライアントが小規模な承認 UI のみを表示する場合でも、`operator.approvals` は慎重に付与してください。

## チャットチャネルへの承認転送

任意のチャットチャネル（Plugin チャネルを含む）に exec 承認プロンプトを転送し、`/approve` で承認できます。これには通常の送信配信パイプラインが使用されます。

設定:

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session", // "session" | "targets" | "both"
      agentFilter: ["main"],
      sessionFilter: ["discord"], // substring or regex
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

チャットで返信:

```
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

`/approve` コマンドは、exec 承認と Plugin 承認の両方を処理します。ID が保留中の exec 承認に一致しない場合は、代わりに Plugin 承認を自動的に確認します。このフォールバックは「approval not found」エラーに限定されます。実際の exec 承認の拒否またはエラー時に、Plugin 承認として暗黙に再試行されることはありません。

### Plugin 承認の転送

Plugin 承認の転送では exec 承認と同じ配信パイプラインを使用しますが、`approvals.plugin` の下に独立した設定があります。一方を有効または無効にしても、もう一方には影響しません。
Plugin 作成時の動作、リクエストフィールド、決定のセマンティクスについては、[Plugin 権限リクエスト](/plugins/plugin-permission-requests)を参照してください。

```json5
{
  approvals: {
    plugin: {
      enabled: true,
      mode: "targets",
      agentFilter: ["main"],
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

設定の形式は `approvals.exec` と同じです。`enabled`、`mode`、`agentFilter`、`sessionFilter`、`targets` は同じように機能します。

共有の対話型返信をサポートするチャネルでは、exec 承認と Plugin 承認の両方に同じ承認ボタンが表示されます。共有の対話型 UI がないチャネルでは、`/approve` の手順を含むプレーンテキストにフォールバックします。Plugin 承認リクエストでは、利用可能な決定が制限される場合があります。承認画面ではリクエストで宣言された決定セットが使用され、提示されなかった決定を送信しようとすると Gateway が拒否します。

### 任意のチャネルでの同一チャット内承認

exec または Plugin の承認リクエストが配信可能なチャット画面から発生した場合、デフォルトではその同じチャットで `/approve` を使用して承認できます。これは既存の Web UI とターミナル UI のフローに加えて、Slack、Matrix、Microsoft Teams、および同様の配信可能なチャットに適用され、その会話に対する通常のチャネル認証モデルが使用されます。発生元のチャットがすでにコマンドを送信して返信を受信できる場合、承認リクエストを保留状態に維持するためだけに個別のネイティブ配信アダプターを用意する必要はなくなります。

Discord、Telegram、QQ bot も同一チャット内の `/approve` をサポートしますが、これらのチャネルではネイティブ承認配信が無効な場合でも、解決済みの承認者リストを認可に使用します。

### ネイティブ承認配信

一部のチャネルは、ネイティブ承認クライアントとしても機能します。対象は Discord、Slack、Telegram、Matrix、QQ bot です。
ネイティブクライアントは、共有の同一チャット内 `/approve` フローに加えて、承認者への DM、発生元チャットへのファンアウト、チャネル固有の対話型承認 UX を提供します。

ネイティブの承認カードまたはボタンが利用可能な場合、そのネイティブ UI がエージェント向けの主要な経路です。ツールの結果でチャット承認が利用できない、または手動承認が唯一残された経路であると示されない限り、エージェントは重複するプレーンチャットの `/approve` コマンドを追加で表示すべきではありません。

ネイティブ承認クライアントが設定されていても、発生元チャネルでネイティブランタイムがアクティブでない場合、OpenClaw はローカルの決定論的な `/approve` プロンプトを表示したままにします。ネイティブランタイムがアクティブで配信を試みたものの、どのターゲットにもカードが届かなかった場合、OpenClaw は正確な `/approve <id> <decision>` コマンドを含む同一チャット内のフォールバック通知を送信し、リクエストを引き続き解決できるようにします。

一般モデル:

- exec 承認が必要かどうかは、引き続きホストの exec ポリシーによって決まります
- `approvals.exec` は、承認プロンプトを他のチャット宛先に転送するかどうかを制御します
- `channels.<channel>.execApprovals` は、Discord、Slack、Telegram、QQ bot、および同様のチャネル固有のネイティブクライアントを有効にするかどうかを制御します
- リクエストが Slack から送られ、Slack Plugin 承認者を解決できる場合、Slack Plugin 承認では Slack のネイティブ承認クライアントを使用できます。`approvals.plugin` は、Slack の exec 承認が無効な場合でも、Plugin 承認を Slack のセッションまたはターゲットにルーティングできます
- Google Chat のネイティブ承認カードは、`dm.allowFrom` または `defaultTo` から安定した `users/<id>` 承認者を解決できる場合に、Google Chat のスペースまたはスレッドから発生した exec 承認と Plugin 承認を処理します。決定にはリアクションイベントを使用しません
- WhatsApp と Signal のリアクション承認配信は、`approvals.exec` と `approvals.plugin` によって制限されます。これらには `channels.<channel>.execApprovals` ブロックはありません

ネイティブ承認クライアントは、以下の条件をすべて満たすと、DM 優先の配信を自動的に有効にします。

- チャネルがネイティブ承認配信をサポートしている
- 明示的な `execApprovals.approvers`、または `commands.ownerAllowFrom` などの所有者 ID から承認者を解決できる
- `channels.<channel>.execApprovals.enabled` が未設定、または `"auto"` である

ネイティブ承認クライアントを明示的に無効にするには、`enabled: false` を設定します。承認者を解決できる場合に強制的に有効にするには、`enabled: true` を設定します。公開される発生元チャットへの配信は、`channels.<channel>.execApprovals.target` を通じて明示的に指定します。ネイティブの `target` によって発生元チャットへの配信が有効になると、承認プロンプトにコマンドテキストが含まれます。

FAQ: [チャット承認に exec 承認設定が 2 つあるのはなぜですか？](/help/faq-first-run)

- Discord: `channels.discord.execApprovals.*`
- Slack: `channels.slack.execApprovals.*`
- Telegram: `channels.telegram.execApprovals.*`
- QQ bot: `channels.qqbot.execApprovals.*`
- Google Chat: `channels.googlechat.dm.allowFrom` または `channels.googlechat.defaultTo` で安定した承認者を設定します。`execApprovals` ブロックは不要です
- WhatsApp: `approvals.exec` と `approvals.plugin` を使用して、承認プロンプトを WhatsApp にルーティングします
- Signal: `approvals.exec` と `approvals.plugin` を使用して、承認プロンプトを Signal にルーティングします

ネイティブクライアント固有のルーティング:

- Telegram はデフォルトで承認者への DM（`target: "dm"`）を使用します。発生元の Telegram チャットまたはトピックにも承認プロンプトを表示するには、`channel` または `both` に切り替えます。Telegram のフォーラムトピックでは、OpenClaw は承認プロンプトと承認後のフォローアップでトピックを維持します。
- Discord と Telegram の承認者は、明示的に指定する（`execApprovals.approvers`）か、`commands.ownerAllowFrom` から推論できます。解決済みの承認者だけが承認または拒否できます。
- Slack の承認者は、明示的に指定する（`execApprovals.approvers`）か、`commands.ownerAllowFrom` から推論できます。Slack Plugin 承認の DM では、Slack の exec 承認者ではなく、`allowFrom` の Slack Plugin 承認者とアカウントのデフォルトルーティングを使用します。Slack のネイティブボタンは承認 ID の種類を保持するため、`plugin:` ID は Slack 内の第 2 フォールバックレイヤーを使用せずに Plugin 承認を解決できます。
- Google Chat のネイティブカードは、メッセージテキスト内に手動の `/approve` フォールバックを保持しますが、カードボタンのコールバックには不透明なアクショントークンのみが含まれます。承認 ID と決定は、サーバー側の保留状態から復元されます。
- WhatsApp の絵文字承認は、対応するトップレベルの転送ファミリーが WhatsApp にルーティングされている場合、exec と Plugin の両方のプロンプトを処理します。ネイティブ由来のプロンプトは直接バインドされます。共有ターゲットモードの配信では、同じ型付き承認メタデータが受理済みの WhatsApp メッセージ受信記録にバインドされます。
- Signal のリアクション承認は、対応するトップレベルの転送ファミリーが有効で Signal にルーティングされている場合にのみ、exec と Plugin の両方のプロンプトを処理します。同一チャット内の直接的な Signal exec 承認では、明示的な承認者なしでもローカルの `/approve` フォールバックを抑制できます。Signal のリアクション解決には、引き続き `channels.signal.allowFrom` または `defaultTo` の明示的な Signal 承認者が必要です。
- Matrix のネイティブ DM/チャネルルーティングとリアクションショートカットは、exec と Plugin の両方の承認を処理します。Plugin の認可には引き続き `channels.matrix.dm.allowFrom` が使用されます。Matrix のネイティブプロンプトでは、最初のプロンプトイベントに `com.openclaw.approval` カスタムイベント内容が含まれるため、OpenClaw 対応の Matrix クライアントは構造化された承認状態を読み取ることができ、標準クライアントではプレーンテキストの `/approve` フォールバックが維持されます。
- Discord と Telegram のネイティブ承認ボタンでは、トランスポート非公開のコールバックデータに exec または Plugin の明示的な所有者種別が含まれ、その所有者だけが解決されます。種別を持たない古い `/approve` コントロールは、限定的な互換経路として維持されます。アクターが承認できる所有者種別のみを試し、承認が見つからなかった場合にのみ続行し、承認 ID から所有権を推論することはありません。
- リクエスト送信者が承認者である必要はありません。
- オペレーター UI または設定済みの承認クライアントでリクエストを受理できない場合、プロンプトは `askFallback` にフォールバックします。

`/diagnostics` や `/export-trajectory` など、機密性の高い所有者専用グループコマンドでは、承認プロンプトと最終結果に所有者向けの非公開ルーティングを使用します。OpenClaw はまず、所有者がコマンドを実行した同じ画面上で非公開経路を試します。その画面に所有者向けの非公開経路がない場合、`commands.ownerAllowFrom` で利用可能な最初の所有者経路にフォールバックします。そのため、Telegram が主要な非公開インターフェースとして設定されていれば、Discord のグループコマンドからでも承認と結果を所有者の Telegram DM に送信できます。グループチャットには短い確認応答のみが送られます。

関連項目:

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)
- [QQ bot](/channels/qqbot)

### 公式モバイルオペレーターアプリ

公式の iOS および Android アプリも、`operator.admin` 接続が使用されている場合、またはペアリングされた `operator.approvals` デバイスがリクエストの対象として明示的に指定されている場合に、Gateway が所有する保留中の exec 承認を確認できます。これらのアプリは Control UI と同じサニタイズ済みの永続レコードを読み取り、種類を考慮した決定を送信し、Gateway の正規の先着回答結果を表示します。Apple Watch は、ペアリングされた iPhone を通じてこれらの承認プロンプトをミラーリングし、1 回のみ許可と拒否のアクションを提供します。Watch の直接 Gateway モードでは承認を確認できません。

解決確認応答が失われても、送信した選択が確定するわけではありません。アプリはコントロールを無効にして、レコードを再度読み取ります。別の画面が先に回答していた場合、アプリにはその記録済みの決定が表示されます。保留中のプロンプトは、それを発行した Gateway に引き続き紐付けられるため、アクティブな Gateway を切り替えても、古い承認 ID をリダイレクトすることはできません。

### macOS IPC フロー

```
Gateway -> Node サービス (WS)
                 |  IPC (UDS + トークン + HMAC + TTL)
                 v
             Mac アプリ (UI + 承認 + system.run)
```

セキュリティ上の注意:

- Unix ソケットモード `0600`、トークンは `exec-approvals.json` に保存されます。
- 同一 UID のピアチェック。
- チャレンジ/レスポンス（nonce + HMAC トークン + リクエストハッシュ）+ 短い TTL。

## FAQ

### 承認ターゲットで `accountId` と `threadId` が使用されるのはどのような場合ですか？

チャネルに複数の ID が設定されており、承認プロンプトを特定のアカウントから送信する必要がある場合は、`accountId` を使用します。宛先がトピックまたはスレッドをサポートし、プロンプトをトップレベルのチャットではなくそのスレッド内に留める必要がある場合は、`threadId` を使用します。

具体的な Telegram の例として、フォーラムトピックと 2 つの Telegram bot アカウントを持つ運用スーパ―グループがあります。`to` の値はスーパ―グループを指定し、`accountId` は bot アカウントを選択し、`threadId` はフォーラムトピックを選択します。

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "targets",
      targets: [
        {
          channel: "telegram",
          to: "-1001234567890",
          accountId: "ops-bot",
          threadId: "77",
        },
      ],
    },
  },
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "env:TELEGRAM_PRIMARY_BOT_TOKEN",
        },
        "ops-bot": {
          name: "Operations bot",
          botToken: "env:TELEGRAM_OPS_BOT_TOKEN",
        },
      },
    },
  },
}
```

この設定では、転送された exec 承認は、`ops-bot` Telegram アカウントによって、チャット `-1001234567890` のトピック
`77` に投稿されます。`accountId` のないターゲットでは、そのチャンネルのデフォルトアカウントが使用され、`threadId` のないターゲットでは、最上位の宛先に投稿されます。

### 承認がセッションに送信された場合、そのセッション内の誰でも承認できますか？

いいえ。セッションへの配信は、プロンプトが表示される場所のみを制御します。それだけで、そのチャットのすべての参加者に承認権限が付与されるわけではありません。

一般的な同一チャットの `/approve` では、送信者がそのチャンネルセッションでコマンドの実行をすでに許可されている必要があります。チャンネルが明示的な承認者を公開している場合、その承認者は、そのセッションでコマンドの実行を許可されていなくても、`/approve` アクションを承認できます。

一部のチャンネルでは、より厳格です。Discord、Telegram、Matrix、Slack のネイティブ承認 DM、および同様のネイティブ承認クライアントでは、解決された承認者リストを承認権限の判定に使用します。たとえば、Telegram のフォーラムトピックに表示される承認プロンプトは、そのトピック内の全員が閲覧できますが、承認または拒否できるのは、`channels.telegram.execApprovals.approvers` または
`commands.ownerAllowFrom` から解決された数値の Telegram ユーザー ID のみです。

## 関連項目

- [Exec 承認](/ja-JP/tools/exec-approvals) — コアポリシーと承認フロー
- [Exec ツール](/ja-JP/tools/exec)
- [昇格モード](/ja-JP/tools/elevated)
- [Skills](/ja-JP/tools/skills) — スキルに基づく自動許可の動作
