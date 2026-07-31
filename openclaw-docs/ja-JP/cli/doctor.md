---
read_when:
    - 接続や認証に問題があり、ガイド付きの修正を行いたい場合
    - 更新したので、問題がないか確認したい場合
summary: '`openclaw doctor` の CLI リファレンス（ヘルスチェックとガイド付き修復）'
title: 診断
x-i18n:
    generated_at: "2026-07-26T08:57:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e2b0aa9b51d7bccd4357d3ec747be514a0245b44a90e6e6c7ea789ab68420465
    source_path: cli/doctor.md
    workflow: 16
---

# `openclaw doctor`

Gateway、チャンネル、plugins、Skills、モデルルーティング、ローカル状態、設定移行のヘルスチェックとクイック修正です。何かが期待どおりに動作せず、問題の内容を1つのコマンドで確認したい場合に使用します。

Gateway のステータスで劣化した SecretRef オーナーが報告されると、doctor は、コールドまたは古い状態のすべてのオーナー、影響を受ける設定パス、秘匿化された理由、および `openclaw secrets reload` 再試行コマンドを含む **シークレットランタイムの劣化** 警告を表示します。

チャンネルの受信イベントがデッドレター化されると、doctor は影響を受ける各チャンネルアカウントを示し、調査と復旧のために [`openclaw channels dead-letters list`](/ja-JP/cli/channels#inbound-dead-letters) を案内します。

関連項目:

- トラブルシューティング: [トラブルシューティング](/ja-JP/gateway/troubleshooting)
- セキュリティ監査: [セキュリティ](/ja-JP/gateway/security)

## 動作モード

Doctor には5つの動作モードがあります:

| 動作モード                | コマンド                                  | 動作                                                                                     |
| ------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------- |
| 検査                      | `openclaw doctor`                        | 人間向けのチェックとガイド付きプロンプト。                                               |
| 修復                      | `openclaw doctor --fix`                        | サポート対象の修復を適用します。非対話修復が安全でない場合はプロンプトを使用します。       |
| Lint                      | `openclaw doctor --lint`                        | CI、事前チェック、レビューゲート向けの読み取り専用の構造化された検出結果。                |
| 共有 SQLite メンテナンス  | `openclaw doctor --state-sqlite compact`                        | 正規の共有状態 DB に対して、チェックポイント、圧縮、検証を明示的に実行します。            |
| セッション SQLite 移行    | `openclaw doctor --session-sqlite <mode>`                        | セッション状態を検査、インポート、検証、圧縮、復旧、または復元します。                    |

自動化で安定した結果が必要な場合は `--lint` を使用してください。人間のオペレーターが doctor に設定または状態を編集させる場合は `--fix` を使用してください。

## 例

```bash
openclaw doctor
openclaw doctor --lint
openclaw doctor --lint --json
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --all
openclaw doctor --lint --allow-exec
openclaw doctor --deep
openclaw doctor --fix
openclaw doctor --fix --non-interactive
openclaw doctor --generate-gateway-token
openclaw doctor --post-upgrade
openclaw doctor --post-upgrade --json
openclaw doctor --state-sqlite compact
openclaw doctor --state-sqlite compact --json
openclaw doctor --session-sqlite inspect --session-sqlite-all-agents
openclaw doctor --session-sqlite dry-run --session-sqlite-agent main --json
openclaw doctor --session-sqlite import --session-sqlite-all-agents
openclaw doctor --session-sqlite validate --session-sqlite-all-agents --json
openclaw doctor --session-sqlite compact --session-sqlite-all-agents
openclaw doctor --session-sqlite recover --github-issue
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

チャンネル固有の権限については、`doctor` の代わりにチャンネルプローブを使用してください:

```bash
openclaw channels capabilities --channel discord --target channel:<channel-id>
openclaw channels status --probe
```

`channels capabilities` は、特定のチャンネルターゲットに対してボットが実際に持つ権限を報告します。`channels status --probe` は、設定済みのすべてのチャンネルと音声自動参加ターゲットを監査します。

## オプション

| オプション                      | 効果                                                                                                                                                                                                   |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--no-workspace-suggestions`              | ワークスペースのメモリ／検索候補を無効にします。                                                                                                                                                       |
| `--yes`              | プロンプトを表示せずにデフォルトを受け入れます。                                                                                                                                                       |
| `--repair` / `--fix` | 推奨されるサービス以外の修復をプロンプトなしで適用します（`--fix` は別名です）。Gateway サービスのインストール／書き換えには、引き続き対話形式の確認または明示的な `gateway` コマンドが必要です。 |
| `--force`              | カスタムサービス設定の上書きを含む、積極的な修復を適用します。                                                                                                                                         |
| `--non-interactive`              | プロンプトなしで実行します。安全な移行とサービス以外の修復のみが対象です。                                                                                                                             |
| `--generate-gateway-token`              | Gateway トークンを生成して設定します。                                                                                                                                                                 |
| `--allow-exec`              | シークレットの検証時に、設定済みの `exec` SecretRef を doctor が実行できるようにします。                                                                                                    |
| `--deep`              | システムサービスを走査して追加の Gateway インストールを検出し、最近の Gateway スーパーバイザー再起動の引き継ぎを報告します。                                                                            |
| `--lint`              | 最新化されたヘルスチェックを読み取り専用モードで実行し、診断結果を出力します。                                                                                                                         |
| `--post-upgrade`              | アップグレード後の Plugin 互換性プローブを実行します。結果は標準出力に送られ、エラーレベルの結果が1つでも存在する場合は終了コード1になります。                                                          |
| `--state-sqlite <mode>`              | 共有状態 SQLite の明示的なメンテナンスを実行します。唯一のモードは `compact` です。                                                                                                            |
| `--session-sqlite <mode>`              | 対象を絞ったセッション SQLite 移行モードを実行します: `inspect`、`dry-run`、`import`、`validate`、`compact`、`recover`、または `restore`。 |
| `--session-sqlite-store <path>`              | `--session-sqlite` と併用し、従来の `sessions.json` ストアパスを1つ選択します。                                                                                                                      |
| `--session-sqlite-agent <id>`              | `--session-sqlite` と併用し、設定済みのエージェントを1つ選択します。                                                                                                                                   |
| `--session-sqlite-all-agents`              | `--session-sqlite` と併用し、設定済みおよび検出済みのエージェントストアを選択します。                                                                                                                   |
| `--github-issue`              | `--session-sqlite recover` と併用し、秘匿化された openclaw/openclaw の Issue レポートを準備します。`--yes` または対話形式の確認後、doctor が `gh` を使用して作成します。                  |
| `--json`              | `--lint` と併用すると JSON 形式の結果になります。`--post-upgrade` と併用すると `{ probesRun, findings }` になります。`--state-sqlite` または `--session-sqlite` と併用すると、メンテナンスレポートが JSON 形式になります。 |
| `--severity-min <level>`              | `--lint` と併用し、`info`、`warning`、または `error` 未満の結果を除外します。                                                                                  |
| `--all`              | `--lint` と併用し、デフォルトセットから除外されるオプトインチェックを含む、登録済みのすべてのチェックを実行します。                                                                            |
| `--skip <id>`              | `--lint` と併用し、指定したチェック ID をスキップします。繰り返し指定できます。                                                                                                               |
| `--only <id>`              | `--lint` と併用し、指定したチェック ID のみを実行します。繰り返し指定できます。                                                                                                               |

`--severity-min`、`--all`、`--only`、および `--skip` は、`--lint` と併用する場合にのみ指定できます。`--json` は、`--lint`、`--post-upgrade`、`--state-sqlite`、および `--session-sqlite` と併用できます。

## Lint モード

`openclaw doctor --lint` は読み取り専用です。プロンプト、修復、設定／状態の書き換えは行いません。

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --allow-exec
openclaw doctor --lint --only core/doctor/gateway-config --json
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

人間向けの出力は簡潔です:

```text
doctor --lint: 6件のチェックを実行、1件の検出結果
  [warning] core/doctor/gateway-config gateway.mode - gateway.mode が設定されていません。gateway の起動はブロックされます。
    修正: `openclaw configure` を実行して Gateway モード（local/remote）を設定するか、`openclaw config set gateway.mode local` を実行してください。
```

JSON 出力はスクリプト用のインターフェースです:

```json
{
  "ok": false,
  "checksRun": 5,
  "checksSkipped": 0,
  "findings": [
    {
      "checkId": "core/doctor/gateway-config",
      "severity": "warning",
      "message": "gateway.mode が設定されていません。gateway の起動はブロックされます。",
      "path": "gateway.mode",
      "fixHint": "`openclaw configure` を実行して Gateway モード（local/remote）を設定するか、`openclaw config set gateway.mode local` を実行してください。"
    }
  ]
}
```

終了コード:

| コード | 意味                                                                   |
| ------ | ---------------------------------------------------------------------- |
| `0` | 選択した重大度しきい値以上の検出結果はありません。                     |
| `1` | 選択したしきい値を満たす検出結果が1つ以上あります。                     |
| `2` | Lint の検出結果を生成する前にコマンド／ランタイムで障害が発生しました。 |

`--severity-min` は、表示する検出結果と終了しきい値の両方を制御します。重大度の低い `info`/`warning` の検出結果が存在する場合でも、`openclaw doctor --lint --severity-min error` では何も表示せずに `0` で終了することがあります。

`--all` は、重大度によるフィルタリングの前に選択するチェックを制御します。デフォルトの Lint 実行では、詳細チェック、履歴チェック、または修復可能な従来の残留物を検出しやすいチェックが除外されます。完全な一覧には `--all` を使用してください。`--only <id>` は最も精密なセレクターであり、登録済みの任意のチェックを ID で実行できます。

`core/doctor/local-audio-acceleration` は、音声モデルを読み込まずに、自動選択されたローカル STT コマンド、対応可能／要求済み／観測済みの各バックエンドの個別の根拠、およびフォールバック順序を報告します。情報レベルの検出結果を出力するため、表示するには `--severity-min info` を含めてください。

## 構造化ヘルスチェック

最新の doctor チェックでは、小さく分割された次のコントラクトを使用します:

```ts
detect(ctx, scope?) -> HealthFinding[]
repair?(ctx, findings) -> HealthRepairResult
```

`detect()` は `doctor --lint` を動作させます。`repair()` は任意であり、`doctor --fix` / `doctor --repair` の場合にのみ実行されます。この形式へ移行していないチェックでは、引き続き従来の doctor コントリビューションフローを使用します。

修復コンテキストには `dryRun`/`diff` リクエストを含めることができます。修復結果では、構造化された `diffs`（設定/ファイルの編集）および `effects`（サービス、プロセス、パッケージ、状態、その他の副作用）を返せるため、変換済みチェックは変更計画を `detect()` に移すことなく `doctor --fix --dry-run` へ発展できます。

`repair()` は `status: "repaired" | "skipped" | "failed"` を報告します（ステータスが省略された場合は `repaired` を意味します）。修復が `skipped` または `failed` を返した場合、doctor は理由を報告し、そのチェックの検証をスキップします。修復が成功すると、doctor は修復対象の検出事項に限定して `detect()` を再実行します。検出事項がまだ存在する場合、doctor は変更を完了済みとして扱わず、修復警告を報告します。

検出事項には以下が含まれます。

| フィールド             | 目的                                                |
| ----------------- | ------------------------------------------------------ |
| `checkId`         | スキップ/限定フィルターおよび CI 許可リスト用の安定した ID。     |
| `severity`        | `info`、`warning`、または `error`。                         |
| `message`         | 人間が読める問題の説明。                      |
| `path`            | 利用可能な場合は、設定、ファイル、または論理パス。          |
| `line` / `column` | 利用可能な場合はソース位置。                        |
| `ocPath`          | チェックが特定の対象を指せる場合の正確な `oc://` アドレス。 |
| `fixHint`         | 推奨されるオペレーター操作または修復の概要。           |

最新化されたコア doctor チェックは、人間向けの `doctor` / `doctor --fix` 動作を所有する、順序付けされた doctor コントリビューションに引き続き関連付けられます。共有の構造化ヘルスレジストリが拡張ポイントです。バンドル済みおよび Plugin ベースのチェックは、所有するパッケージがアクティブなコマンドパスに登録すると、コア doctor チェックの後に実行されます。`openclaw/plugin-sdk/health` は Plugin 作成者向けに同じ契約を公開します。

## チェックの選択

```bash
openclaw doctor --lint --only core/doctor/gateway-config --json
openclaw doctor --lint --skip core/doctor/skills-readiness
openclaw doctor --lint --all --skip core/doctor/session-locks
```

`--only` と `--skip` は完全なチェック ID を受け取り、繰り返し指定できます。`--only` ID が登録されていない場合、その ID に対するチェックは実行されません。出力の `checksRun`/`checksSkipped` を使用して、限定されたゲートが想定どおりのチェックを選択していることを確認してください。

## アップグレード後モード

`openclaw doctor --post-upgrade` は、ビルドまたはアップグレード後に連続実行するための Plugin 互換性プローブを実行します。検出事項は標準出力に送られます。いずれかの検出事項に `level: "error"` がある場合、終了コードは 1 です。CI、コミュニティの `fork-upgrade` スキル、その他のアップグレード後スモークツールに適した、機械可読エンベロープ（`{ probesRun, findings }`）を出力するには `--json` を追加します。インストール済み Plugin インデックスが存在しないか不正な形式の場合でも、JSON モードは `plugin.index_unavailable` エラー検出事項を含むエンベロープを出力します。

コンテナイメージの起動は、通常の「更新後に doctor を実行する」
フローの例外です。`openclaw gateway run` が新しい OpenClaw バージョンで起動すると、
準備完了を報告する前に、安全な状態修復と Plugin 修復を実行します。修復を
安全に完了できない場合、起動は終了し、コンテナを通常どおり再起動する前に、
同じマウント済み状態/設定に対して同じイメージを `openclaw doctor --fix` 付きで
一度実行するよう指示します。

## レガシー状態の移行

`openclaw doctor --fix` は、永続ファイルから SQLite への移行を所有する唯一の機能です。認識された各ソースを検証して取得し、正規行を書き込んで検証し、移行レシートを記録してから、廃止されたソースを削除します。ランタイムコードは遅延インポートやフォールバック読み取りを実行しません。

これには、`<state-dir>/mcp-oauth/*.json` 配下の廃止された MCP OAuth ファイルが含まれます。修復前に Gateway を停止してください。Doctor は有効な認証情報を `<state-dir>/state/openclaw.sqlite` にインポートし、両方のストアが存在する場合は既存の正規 SQLite セッションを保持し、廃止された永続 OAuth `state` 値を削除します。また、レシートを使用して、再作成された古いファイルによってログアウト済みの認証情報が復活することを防ぎます。廃止された `.lock` サイドカーはフェイルクローズします。Doctor が古い所有者を報告した場合は、古い OpenClaw プロセスが実行されていないことを確認し、そのサイドカーを削除して Doctor を再実行してください。

## 共有状態 SQLite の Compaction

スキーマのバージョン管理、整合性チェック、ダウングレード復旧については、[データベーススキーマ](/ja-JP/reference/database-schemas)を参照してください。

`openclaw doctor --state-sqlite compact` は、
`<state-dir>/state/openclaw.sqlite` にある正規の共有状態データベースに対する
明示的なオフラインメンテナンスです。任意のデータベース
パスは受け付けず、通常の Gateway 動作によって呼び出されることはなく、
`openclaw doctor --fix` の一部でもありません。このコマンドは、
Gateway 起動時と同じ状態所有権ロックを取得し、検証、チェックポイント処理、`VACUUM`、
最終整合性チェックが完了するまで保持します。Gateway または別の
SQLite メンテナンスコマンドがそのロックを所有している間は実行を拒否します。
`OPENCLAW_ALLOW_MULTI_GATEWAY=1` が設定ごとの Gateway シングルトンをスキップする場合でも
状態ロックは有効なため、メンテナンスで Gateway を検出するために、
オペレーターのシェルが Gateway サービスの環境を継承する必要はありません。

最初に Gateway を停止し、検証済みバックアップを作成します。

```bash
openclaw gateway stop
openclaw backup create --verify
openclaw doctor --state-sqlite compact --json
openclaw gateway start
```

このコマンドは以下を実行します。

1. 正規の共有状態パスに通常ファイルが必要です。データベースが
   存在しない場合は `skipped` として報告され、正常終了します。
2. チェックポイント処理またはファイルの変更前に、現在サポートされているスキーマバージョンと
   `schema_meta.role = "global"` を検証します。
3. ビジー状態でない `wal_checkpoint(TRUNCATE)` が必要です。チェックポイントがビジーの場合は、残っている OpenClaw
   プロセスをすべて停止して再試行してください。
4. `auto_vacuum` を `INCREMENTAL` に設定し、完全な `VACUUM` を実行してから、再度チェックポイント処理を
   行います。
5. `quick_check`、`integrity_check`、`foreign_key_check` を実行してから、
   データベースおよび SQLite サイドカーファイルに所有者専用権限を再適用します。

JSON 出力では、Compaction 前後のデータベースと WAL のサイズ、フリーリストページ数、ページサイズ、
`auto_vacuum` 値に加えて、回収されたバイト数、`quick_check` と
`integrity_check` の結果が報告されます。`foreign_key_check` は
フェイルクローズで適用され、個別の成功フィールドはありません。SQLite は `auto_vacuum` を、
なしの場合は `0`、完全の場合は `1`、増分の場合は `2` として報告します。

スキーマが古い場合、実行中の OpenClaw ビルドより新しい場合、または
エージェントデータベースに属する場合、Compaction は変更を加えずに失敗します。
古い共有状態スキーマには、最初に `openclaw doctor --fix` を実行してください。
新しいスキーマには、互換性のあるバックアップを復元するか OpenClaw をアップグレードしてください。

## セッション SQLite の移行

OpenClaw は、Gateway 起動時および
`openclaw doctor --fix` の実行時に、レガシーセッション行とトランスクリプト履歴を各エージェントの
SQLite データベースへ自動的にインポートします。`openclaw doctor --session-sqlite <mode>` は、
その移行を対象とする検査および検証ツールです。現在のランタイム
セッション行は
`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` にあります。レガシー
`sessions.json` ファイルは移行元です。ホットトランスクリプト JSONL ファイルは、
インポートが成功するとインポートされ、アクティブなセッションディレクトリの外へアーカイブされます。
アーカイブ階層の JSONL ファイルはサポート用アーティファクトとして残り、ランタイムの
フォールバックにはなりません。

モード:

| モード       | 動作                                                                                                               |
| ---------- | ---------------------------------------------------------------------------------------------------------------------- |
| `inspect`  | インポートせずに、レガシーと SQLite の件数、および参照されていない JSONL ファイルを読み取ります。                                       |
| `dry-run`  | レガシーエントリとトランスクリプト JSONL ファイルを解析し、インポート可能な行を数え、SQLite 行を書き込まずに問題を報告します。 |
| `import`   | 選択した対象について、レガシーエントリとトランスクリプトイベントを SQLite にインポートします。                                      |
| `validate` | 選択したレガシーソースを SQLite 行およびトランスクリプトイベント数と比較します。                                   |
| `compact`  | 大量削除またはアーカイブ整理後に空きページを回収するため、選択したエージェント SQLite データベースをチェックポイント処理して VACUUM します。    |
| `recover`  | 最新の失敗した移行実行を復元し、その対象を検証して、サニタイズ済みの GitHub Issue レポートを準備します。            |
| `restore`  | SQLite データを削除せずに、記録された移行マニフェストからアーカイブ済みトランスクリプトアーティファクトを復元します。                  |

セレクター:

- デフォルト: そのレガシーストアファイルが存在する場合、設定済みのデフォルトエージェントストア。
- `--session-sqlite-agent <id>`: 設定済みエージェント 1 つ。
- `--session-sqlite-all-agents`: 設定済みエージェントストアと検出されたエージェントストア。
- `--session-sqlite-store <path>`: 明示的なレガシー `sessions.json` パス 1 つ。

手動検査の手順:

```bash
openclaw doctor --session-sqlite inspect --session-sqlite-all-agents
openclaw doctor --session-sqlite dry-run --session-sqlite-all-agents --json
openclaw doctor --session-sqlite import --session-sqlite-all-agents
openclaw doctor --session-sqlite validate --session-sqlite-all-agents --json
openclaw doctor --session-sqlite compact --session-sqlite-all-agents
openclaw doctor --session-sqlite recover --github-issue
```

重要な履歴があるインストール環境で `import` を実行する前に、OpenClaw 状態ディレクトリをバックアップしてください。
選択したレガシーエントリが SQLite にない場合、セッション ID が異なる場合、またはトランスクリプトイベント数が異なる場合、
`validate` は非ゼロで終了します。
`--session-sqlite-store <path>` を使用する場合は、レポートに
想定される対象数が含まれていることを確認してください。存在しない明示的なストアパスを指定すると、対象は選択されません。

SQLite の削除では、まずデータベース内のページが回収されます。必ずしも
データベースファイルがすぐに縮小されるわけではありません。大量の
トランスクリプトを削除またはアーカイブした後、`openclaw doctor --session-sqlite compact --session-sqlite-all-agents` を実行して
WAL ファイルをチェックポイント処理し、`VACUUM` を実行して、処理前後のデータベースと WAL の
サイズを報告します。Compaction には、現在のエージェントスキーマを持つ通常ファイル、
選択したエージェントの永続的な所有者メタデータ、および doctor
プロセス内に開いているハンドルがないことが必要です。破壊的な `import`、`compact`、`recover`、`restore` モードは、
操作全体にわたって Gateway 起動時と同じ状態所有権ロックを保持します。
`inspect`、`dry-run`、`validate` は読み取り専用のままであり、そのロックを取得しません。
最初に Gateway を停止してください。破壊的モードは、ライブ書き込みや
別のメンテナンスコマンドと競合する代わりに失敗します。破壊的な `--session-sqlite-store`
の対象はアクティブな状態ディレクトリ内にある必要があります。別のインストール環境を
メンテナンスする前に、`OPENCLAW_STATE_DIR` をそのストアを所有する状態ディレクトリに
設定してください。既存のハードリンク対象は拒否されます。これは、ロックされた状態ディレクトリの外にある
別のパスが同じデータベース inode を共有できるためです。同じ所有権
チェックが SQLite WAL、共有メモリ、ロールバックジャーナルのサイドカーにも適用されます。

各インポートは、トランスクリプトアーティファクトを
アーカイブへ移動する前に、`~/.openclaw/session-sqlite-migration-runs/` 配下へマニフェストを書き込みます。
アーティファクトの移動後に、起動処理がセッション SQLite の移行失敗を報告した場合は、
復旧を実行してください。

```bash
openclaw doctor --session-sqlite recover --github-issue
```

復旧では、最新の失敗した移行マニフェストを選択し、そのマニフェストにアーカイブされた成果物のみを復元し、影響を受けた対象を検証し、サニタイズ済みの `.failure.md` および `.failure.json` レポートを更新したうえで、トランスクリプトの内容、生の環境情報、シークレット、無制限の設定を含まない GitHub issue 本文を準備します。失敗した移行マニフェストが存在しないものの、選択したエージェントの SQLite データベースが破損している、データベースではない、またはメインデータベースがない状態でジャーナルサイドカーファイルが存在する場合、復旧ではファイル一式を一時検査ディレクトリにコピーします。その使い捨てコピーでは、`quick_check`、`integrity_check`、および `foreign_key_check` を実行する前に、SQLite が有効なホットジャーナルをロールバックできます。その間も、元のフォレンジックファイルには一切変更を加えません。整合性チェックに失敗した場合や孤立したサイドカーファイルがある場合は、検出されたファイル一式の名前を単一の `.corrupt-<timestamp>` サフィックス付きに変更し、DB、WAL、SHM、およびロールバックジャーナルファイルを保持します。名前変更の失敗が捕捉されると、失敗を報告する前に、すでに移動したファイルを元に戻すため、復旧可能なファイル一式が暗黙に分断されることはありません。復旧前に Gateway を停止してください。変更中の SQLite ファイル一式をコピーまたは名前変更するのは安全ではなく、オペレーティングシステムによって動作も異なります。`--github-issue --yes` を指定すると、doctor は GitHub CLI を使用して `openclaw/openclaw` に issue を作成します。確認を行わない場合は、ローカルサポートレポートを書き込み、入力済みの issue URL を出力します。

`restore` は、引き続き低レベルの取り消し操作です。マニフェストの `sourcePath -> archivePath` レコードを使用し、元のパスが存在しない場合にのみ、アーカイブされた成果物を元の場所へ戻します。両方のパスが存在する場合は競合を報告し、SQLite データベースはそのまま残します。

### セッションの SQLite 移行後のダウングレード

以前のファイルベースの OpenClaw バージョンを起動する前に、アーカイブされた従来のトランスクリプト成果物を復元します。

```bash
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

以前のバージョンは、`sessions.json` エントリと、それらのエントリに記録された `sessionFile` パスを読み取ります。SQLite への移行後、正常にインポートされた使用中の JSONL トランスクリプトは `session-sqlite-import-archive/` に移動されます。そのため、restore によってマニフェストに記録された成果物が元のパスへ戻されるまで、以前のランタイムからその履歴を参照することはできません。

復元では SQLite データを削除しません。SQLite への切り替え後に作成されたセッションは SQLite にのみ存在し、以前のランタイムには表示されません。後で再びアップグレードする場合は、前述の通常の移行検証手順を実行してください。これにより、OpenClaw はインポート前に、復元された従来の成果物と SQLite の行を比較できます。

## 注記

- Nix モード (`OPENCLAW_NIX_MODE=1`) では、読み取り専用の doctor チェックは引き続き機能しますが、`openclaw.json` は変更不可能なため、`doctor --fix`、`doctor --repair`、`doctor --yes`、および `doctor --generate-gateway-token` は無効になります。代わりに、このインストールの Nix ソースを編集してください。nix-openclaw については、エージェント優先の[クイックスタート](https://github.com/openclaw/nix-openclaw#quick-start)を参照してください。
- 対話型プロンプト（キーチェーン/OAuth の修正など）は、stdin が TTY であり、かつ `--non-interactive` が設定されて**いない**場合にのみ実行されます。ヘッドレス実行（cron、Telegram、ターミナルなし）ではプロンプトをスキップします。
- 非対話型の `doctor` 実行では、ヘッドレスなヘルスチェックを高速に保つため、先行的な Plugin 読み込みをスキップします。対話型セッションでは、従来のヘルスチェック/修復フローに必要な Plugin サーフェスを引き続き読み込みます。
- `--lint` は `--non-interactive` より厳格です。常に読み取り専用で、プロンプトを表示せず、安全な移行も適用しません。doctor で変更を行う場合は、`doctor --fix` または `doctor --repair` を使用してください。
- doctor は、デフォルトではシークレットのチェック時に `exec` SecretRefs を実行しません。設定済みのシークレットリゾルバーを doctor に意図的に実行させる場合にのみ、`--allow-exec` を使用してください（`--lint` の有無は問いません）。
- 設定への書き込み（`--fix` による修復を含む）が行われると、バックアップが `~/.openclaw/openclaw.json.bak` にローテーションされます（番号付きの `.bak.1`..`.bak.4` リングを使用）。`--fix` は、スキーマ検証で報告された不明な設定キーも削除し、削除した各キーを一覧表示します。ただし、更新の進行中はこの処理をスキップし、部分的に書き込まれたアップグレード状態が移行完了前に除去されないようにします。
- `openclaw.json` を解析できず、最後に正常だった設定も復元できない場合、`doctor --fix` は元の設定を `openclaw.json.clobbered.<timestamp>` として保存し、現在のファイルを変更せず、部分的な代替設定を書き込む代わりにエラーで終了します。
- 別のスーパーバイザーが Gateway のライフサイクルを管理している場合は、`OPENCLAW_SERVICE_REPAIR_POLICY=external` を設定します。doctor は引き続き Gateway/サービスの健全性を報告し、サービス以外の修復を適用しますが、サービスのインストール/開始/再起動/ブートストラップ、および従来のサービスのクリーンアップはスキップします。
- doctor は、管理対象 Gateway に適用されたヒープ制限と、現在のホストまたはコンテナのメモリ制限に使用された適応的な導出方法を報告します。修復処理以外で同じレポートを確認するには、`openclaw gateway status` を使用してください。
- Linux では、doctor は非アクティブな追加の Gateway 類似 systemd ユニットを無視し、修復中に実行中の systemd Gateway サービスのコマンド/エントリポイントメタデータを書き換えません。最初にサービスを停止するか、`openclaw gateway install --force` を使用してアクティブなランチャーを置き換えてください。
- `doctor --fix --non-interactive` は、欠落または古くなった Gateway サービス定義を報告しますが、更新修復モード以外ではインストールや書き換えを行いません。サービスがない場合は `openclaw gateway install` を、ランチャーを置き換える場合は `openclaw gateway install --force` を実行してください。
- 状態整合性チェックでは、セッションディレクトリ内の孤立したトランスクリプトファイルを検出します。それらを `.deleted.<timestamp>` としてアーカイブするには、対話型の確認が必要です。`--fix`、`--yes`、およびヘッドレス実行では、そのまま残します。
- doctor は、`~/.openclaw/cron/jobs.json`（または `cron.store`）で従来形式の cron ジョブをスキャンして書き換えた後、正規化された行を SQLite にインポートします。
- doctor は、明示的な `payload.model` オーバーライドを持つ cron ジョブについて、プロバイダー名前空間ごとの件数と `agents.defaults.model` との不一致を含めて報告します。これにより、デフォルトモデルを継承しないスケジュール済みジョブを、認証や請求の調査時に確認できます。
- doctor は、実行中としてマークされたままの cron ジョブ（`state.runningAtMs`）を報告します。この状態では、`openclaw cron list` に `running` と表示されることがあります。このチェックは読み取り専用です。現在マーク付きジョブを実行している Gateway がない場合、次回の cron サービス起動時に中断された実行が記録され、マーカーがクリアされます。
- Linux では、ユーザーの crontab が保守されていない従来の `~/.openclaw/bin/ensure-whatsapp.sh` をまだ実行している場合、doctor が警告します。cron に systemd ユーザーバス環境がないと、これは `Gateway inactive` を誤って報告する可能性があります。
- WhatsApp が有効な場合、doctor はローカルの `openclaw-tui` クライアントが実行されたままの、劣化した Gateway イベントループをチェックします。`doctor --fix` は、検証済みのローカル TUI クライアントのみを停止し、WhatsApp の返信が古い TUI 更新ループの後ろでキューに滞留しないようにします。
- HTTP(S) プロキシ環境変数が存在する一方で `tools.web.fetch.useTrustedEnvProxy` が無効な場合、doctor は `web_fetch` が引き続き直接ルーティングを使用することを説明し、短時間の直接 TLS 接続プローブを実行して、明示的なオプトイン方法を示します。プロキシの信頼を自動的に有効にすることはありません。
- doctor は、プライマリモデル、フォールバック、モデル許可リスト、画像/動画生成モデル、Heartbeat/サブエージェント/Compaction のオーバーライド、フック、チャンネルモデルのオーバーライド、cron ペイロード、古いセッション/トランスクリプトのルート固定に含まれる従来の `codex/*` および `openai-codex/*` モデル参照を、正規の `openai/*` 参照に書き換えます。`--fix` は、安全な場合に従来の `models.providers.codex` および `models.providers.openai-codex` 設定も統合し、従来の `openai-codex:*` 認証プロファイルと `auth.order.openai-codex` エントリを `openai:*` に移行し、Codex の意図をプロバイダー/モデル単位の `agentRuntime.id: "codex"` エントリへ移し、古いエージェント全体/セッションのランタイム固定を削除します。また、修復された OpenAI エージェント参照では、直接の OpenAI API キー認証ではなく Codex 認証ルーティングを維持します。
- doctor は、参照先プロファイルがすべて失われている一方で、互換性のある保存済み認証情報が存在する、空でない `auth.order.<provider>` リストを報告します。`doctor --fix` は、それらの古いオーバーライドのみを削除し、エージェントごとの認証情報の自動選択を復元します。明示的な空の順序、一部が有効なリスト、および互換性のある保存済み認証情報がない順序は変更されません。アクティブな SQLite 認証ストアが読み取れないか不正な形式の場合、doctor はこの修復をスキップした理由を説明します。設定の再読み込みモードで書き込みが自動適用されない場合は、認証状態を再確認する前に、実行中の Gateway を再起動してください。
- doctor は、古い OpenClaw バージョンの従来の Plugin 依存関係ステージング状態をクリーンアップし、ホストの `openclaw` パッケージをピア依存関係として宣言している管理対象 npm Plugin に再リンクします。また、設定で参照されている欠落したダウンロード可能な Plugin（`plugins.entries`、設定済みチャンネル、設定済みプロバイダー/検索設定、設定済みエージェントランタイム）も修復します。パッケージ更新中は、パッケージの入れ替えが完了するまで、doctor はパッケージマネージャーによる Plugin 修復をスキップします。設定済み Plugin の復旧がまだ必要な場合は、その後 `openclaw doctor --fix` を再実行してください。ダウンロードに失敗した場合、doctor はインストールエラーを報告し、次回の修復試行に備えて設定済み Plugin エントリを保持します。
- Plugin の検出が正常な場合、doctor は `plugins.allow`/`plugins.deny`/`plugins.entries` から欠落した Plugin ID を削除し、それに対応する参照先のないチャンネル設定、Heartbeat ターゲット、およびチャンネルモデルのオーバーライドも削除して、古い Plugin 設定を修復します。
- doctor は、影響を受ける `plugins.entries.<id>` エントリを無効にし、不正な `config` ペイロードを削除することで、不正な Plugin 設定を隔離します。Gateway の起動時には、その不正な Plugin のみがすでにスキップされるため、他の Plugin とチャンネルは引き続き動作します。
- doctor は、廃止された `plugins.entries.codex.config.codexDynamicToolsProfile` を削除します。Codex app-server は、Codex ネイティブのワークスペースツールを常にネイティブのまま維持します。
- doctor は、従来のフラットな Talk 設定（`talk.voiceId`、`talk.modelId` など）を `talk.provider` + `talk.providers.<provider>` に自動移行します。相違点がオブジェクトキーの順序だけの場合、`doctor --fix` を繰り返し実行しても Talk の正規化は報告/適用されなくなりました。
- doctor にはメモリ検索の準備状況チェックが含まれ、埋め込み用の認証情報がない場合は `openclaw configure --section model` を推奨できます。
- コマンド所有者が設定されていない場合、doctor が警告します。コマンド所有者とは、所有者専用コマンドの実行と危険な操作の承認を許可された人間のオペレーターアカウントです。DM ペアリングで可能になるのは、ボットとの会話だけです。最初の所有者を設定するブートストラップが導入される前に送信者を承認していた場合は、`commands.ownerAllowFrom` を明示的に設定してください。
- Codex モードのエージェントが設定されており、オペレーターの Codex ホームに個人用 Codex CLI アセットが存在する場合、doctor は情報メモを報告します。ローカルの Codex app-server 起動では、エージェントごとに分離されたホームを使用します。必要な場合は、まず Codex Plugin をインストールしてから、`openclaw migrate plan codex` を使用し、意図的に昇格させるべきアセットの一覧を確認してください。
- デフォルトエージェントに許可された Skills が、現在のランタイム環境で利用できない場合（バイナリ、環境変数、設定、または OS 要件の不足）、doctor が警告します。`doctor --fix` では、`skills.entries.<skill>.enabled=false` を使用して、それらの利用できない Skills を無効にできます。Skills を有効なままにする場合は、代わりに不足している要件をインストール/設定してください。
- サンドボックスモードが有効で Docker が利用できない場合、doctor は修復方法（`install Docker` または `openclaw config set agents.defaults.sandbox.mode off`）を含む、重要度の高い警告を報告します。
- 従来のサンドボックスレジストリファイルまたはシャードディレクトリ（`~/.openclaw/sandbox/containers.json`、`~/.openclaw/sandbox/browsers.json`、`~/.openclaw/sandbox/containers/`、または `~/.openclaw/sandbox/browsers/`）が存在する場合、doctor が報告します。`--fix` は、有効なエントリを SQLite に移行し、不正な従来ファイルを隔離します。
- `gateway.auth.token`/`gateway.auth.password` が SecretRef で管理され、現在のコマンドパスで利用できない場合、doctor は読み取り専用の警告を報告し、平文のフォールバック認証情報を書き込みません。exec ベースの SecretRefs については、`--allow-exec` が存在しない限り、doctor は実行をスキップします。
- 修正処理でチャンネル SecretRef の検査に失敗した場合、doctor は早期終了せず、処理を継続して警告を報告します。
- 状態ディレクトリの移行後、有効なデフォルトの Telegram または Discord アカウントが環境変数フォールバックに依存しており、`TELEGRAM_BOT_TOKEN` または `DISCORD_BOT_TOKEN` を doctor プロセスから利用できない場合、doctor が警告します。
- Telegram の `allowFrom` ユーザー名の自動解決（`doctor --fix`）には、現在のコマンドパスで解決可能な Telegram トークンが必要です。トークンを検査できない場合、doctor は警告を報告し、その処理での自動解決をスキップします。

## macOS: `launchctl` 環境変数のオーバーライド

以前に `launchctl setenv OPENCLAW_GATEWAY_TOKEN ...`（または `...PASSWORD`）を実行した場合、その値が設定ファイルをオーバーライドし、永続的な「unauthorized」エラーの原因になる可能性があります。

```bash
launchctl getenv OPENCLAW_GATEWAY_TOKEN
launchctl getenv OPENCLAW_GATEWAY_PASSWORD

launchctl unsetenv OPENCLAW_GATEWAY_TOKEN
launchctl unsetenv OPENCLAW_GATEWAY_PASSWORD
```

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [Gateway doctor](/ja-JP/gateway/doctor)
