---
read_when:
    - ソースチェックアウトを安全に更新したい場合
    - '`openclaw update` の出力またはオプションをデバッグしています'
    - '`--update` の省略記法の動作を理解する必要があります'
summary: '`openclaw update` の CLI リファレンス（比較的安全なソース更新 + Gateway の自動再起動）'
title: 更新
x-i18n:
    generated_at: "2026-07-26T09:31:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b46696f6b9cba5c318f870bcb6c5ea8e0652940968da2ad85e86709fe4c11146
    source_path: cli/update.md
    workflow: 16
---

# `openclaw update`

OpenClaw を更新し、stable/extended-stable/beta/dev チャンネルを切り替えます。

**npm/pnpm/bun** 経由でインストールした場合（グローバルインストール、git メタデータなし）、
更新は[更新](/ja-JP/install/updating)で説明されている
パッケージマネージャーのフローを使用します。

## 使用方法

```bash
openclaw update
openclaw update status
openclaw update repair
openclaw update wizard
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
openclaw update --tag beta
openclaw update --tag main
openclaw update --dry-run
openclaw update --no-restart
openclaw update --yes
openclaw update --acknowledge-clawhub-risk
openclaw update --json
openclaw --update
```

`openclaw --update` は `openclaw update` に書き換えられます（シェルや
ランチャースクリプトに便利です）。

## オプション

| フラグ                                             | 説明                                                                                                                                                                                                                                                                                                                                  |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--no-restart`                                   | 更新の成功後に Gateway サービスを再起動しません。再起動を行うパッケージマネージャー更新では、コマンドが成功する前に、再起動されたサービスが想定されるバージョンを報告することを検証します。                                                                                                                                                |
| `--channel <stable\|extended-stable\|beta\|dev>` | 更新チャンネルを設定し、コアの更新成功後も保持します。Extended-stable はパッケージでのみ利用できます。                                                                                                                                                                                                                                            |
| `--tag <dist-tag\|version\|spec>`                | この更新に限りパッケージターゲットを上書きします。検証済みの正確なターゲットが必須となる、有効な `extended-stable` チャンネルとは併用できません。その他のパッケージインストールでは、`main` は `github:openclaw/openclaw#main` にマッピングされます。GitHub/git ソース仕様は、段階的なグローバル npm インストールの前に一時 tarball にパックされます。 |
| `--dry-run`                                      | 設定の書き込み、インストール、Plugin の同期、再起動を行わずに、予定されている処理（チャンネル/タグ/ターゲット/再起動フロー）をプレビューします。                                                                                                                                                                                                                |
| `--json`                                         | 機械可読な `UpdateRunResult` JSON を出力します。管理対象 Plugin に修復が必要な場合は `postUpdate.plugins.warnings`、beta チャンネルの Plugin フォールバックの詳細、および更新後の同期中に npm Plugin アーティファクトの不整合が検出された場合は `postUpdate.plugins.integrityDrifts` が含まれます。                                                                 |
| `--timeout <seconds>`                            | ステップごとのタイムアウト。デフォルトは `1800` です。                                                                                                                                                                                                                                                                                                            |
| `--yes`                                          | 確認プロンプト（ダウングレード確認など）をスキップします。                                                                                                                                                                                                                                                                              |
| `--acknowledge-clawhub-risk`                     | 対話型プロンプトなしで、コミュニティ ClawHub の信頼性に関する警告を無視して更新後の Plugin 同期を続行できるようにします。これを指定せず、OpenClaw がプロンプトを表示できない場合、リスクのあるコミュニティリリースはスキップされ、変更されません。公式 ClawHub パッケージと同梱 Plugin ソースでは、このプロンプトは表示されません。                                                     |

`--verbose` フラグはありません。予定されている処理のプレビューには `--dry-run`、
機械可読な結果には `--json`、チャンネル/利用可否のみの確認には
`openclaw update status --json` を使用します。Gateway コンソールの詳細度（`--verbose`）と
ファイルログレベル（`logging.level: "debug"`/`"trace"`）は独立した設定です。
[Gateway のログ](/ja-JP/gateway/logging)を参照してください。

<Note>
Nix モード（`OPENCLAW_NIX_MODE=1`）では、状態を変更する `openclaw update` の実行は無効です。代わりに、このインストールの Nix ソースまたは flake 入力を更新してください。nix-openclaw では、エージェント優先の[クイックスタート](https://github.com/openclaw/nix-openclaw#quick-start)を使用してください。`openclaw update status` と `openclaw update --dry-run` は引き続き読み取り専用です。
</Note>

<Warning>
古いバージョンでは設定が壊れる可能性があるため、ダウングレードには確認が必要です。
インストール済み環境でセッションがすでに SQLite に移行されている場合は、古いファイルベースの
バージョンを起動する前に、アーカイブされた従来のトランスクリプトアーティファクトを復元してください。
[Doctor：セッションの SQLite 移行後のダウングレード](/ja-JP/cli/doctor#downgrading-after-session-sqlite-migration)を参照してください。
</Warning>

## `update status`

有効な更新チャンネル、git タグ/ブランチ/SHA（ソースチェックアウトのみ）、
および更新の利用可否を表示します。

```bash
openclaw update status
openclaw update status --json
openclaw update status --timeout 10
```

| フラグ                  | デフォルト | 説明                         |
| --------------------- | ------- | ----------------------------------- |
| `--json`              | `false` | 機械可読なステータス JSON を出力します。 |
| `--timeout <seconds>` | `3`     | チェックのタイムアウト。                 |

extended-stable のパッケージインストールでは、ステータス確認はフォアグラウンド更新と同じ公開セレクターの解決と
正確なパッケージの検証を実行します。インストール済みバージョンの方が新しい場合は
`ahead of extended-stable` を報告することがあります。JSON の失敗には
`registry.reason`（`selector_missing`、`selector_query_failed`、
`exact_package_mismatch`、または `unsupported_git_channel`）が含まれます。

## `update repair`

コアパッケージはすでに変更されたものの、その後の
修復処理が正常に完了しなかった場合に、更新の最終処理を再実行します。`openclaw update` により
新しいコアパッケージはインストールされたものの、コア更新後の Plugin 同期、
管理対象 npm Plugin のメタデータ、レジストリの更新、または Doctor の修復が
収束しなかった場合にサポートされる復旧手段です。

```bash
openclaw update repair
openclaw update repair --channel beta
openclaw update repair --acknowledge-clawhub-risk
openclaw update repair --json
```

| フラグ                                             | 説明                                                                                                                                                                                                                                                         |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--channel <stable\|extended-stable\|beta\|dev>` | 修復前にコアの更新チャンネルを保持します。extended-stable では、bare/default または `latest` の指定に従う対象の公式 npm Plugin は、インストール済みのコアの正確なバージョンをターゲットにします。extended-stable の修復は、設定を変更せずに Git チェックアウトでは拒否されます。 |
| `--json`                                         | 機械可読な最終処理 JSON を出力します。                                                                                                                                                                                                                           |
| `--timeout <seconds>`                            | 修復ステップのタイムアウト。デフォルトは `1800` です。                                                                                                                                                                                                                           |
| `--yes`                                          | 確認プロンプトをスキップします。                                                                                                                                                                                                                                          |
| `--acknowledge-clawhub-risk`                     | `openclaw update` と同じ動作です。                                                                                                                                                                                                                              |
| `--no-restart`                                   | 一貫性のため受け付けられますが、修復では Gateway を再起動しません。                                                                                                                                                                                                             |

`update repair` は `openclaw doctor --fix` を実行し、修復された設定と
インストール記録を再読み込みし、有効な更新チャンネルに対応する追跡対象 Plugin を同期し、
管理対象 npm Plugin のインストールを更新し、欠落している設定済み Plugin ペイロードを修復し、
Plugin レジストリを更新して、収束したインストール記録のメタデータを書き込みます。
新しいコアパッケージのインストールも Gateway の再起動も行いません。

## `update wizard`

更新チャンネルを選択し、その後 Gateway を再起動するかどうかを確認する
対話型フローです（デフォルトでは再起動します）。git チェックアウトなしで `dev` を選択すると、
チェックアウトの作成を提案します。

| フラグ                  | デフォルト | 説明                   |
| --------------------- | ------- | ----------------------------- |
| `--timeout <seconds>` | `1800`  | 各更新ステップのタイムアウト。 |

## 動作内容

チャンネルを明示的に切り替えると（`--channel ...`）、インストール方法も
一致するように維持されます。

- `dev` -> git チェックアウト（デフォルトは `~/openclaw`、
  `OPENCLAW_HOME` が設定されている場合は `$OPENCLAW_HOME/openclaw`。`OPENCLAW_GIT_DIR` で上書き可能）を
  確保して更新し、そのチェックアウトからグローバル CLI を
  インストールします。
- `stable` -> `latest` を使用して npm からインストールします。
- `extended-stable` -> 公開 npm の `extended-stable` セレクターを解決し、
  選択された正確なパッケージを検証して、その正確なバージョンをインストールします。
  別のセレクターへのフォールバックは行わず、Git チェックアウトでは拒否されます。
- `beta` -> npm dist-tag `beta` を優先し、beta が
  存在しない場合、または現在の stable リリースより古い場合は `latest` にフォールバックします。

### 再起動の引き継ぎ

Gateway コアの自動更新機能（設定で有効化されている場合）は、稼働中の Gateway リクエストハンドラーの
外部で CLI 更新パスを起動します。コントロールプレーンの
`update.run` パッケージマネージャー更新と監視下の git チェックアウト更新では、
稼働中の Gateway プロセス内でパッケージツリーを置き換えたり
`dist/` を再構築したりせず、同じ管理サービスへの引き継ぎを使用します。Gateway は
デタッチされたヘルパーを起動して終了し、そのヘルパーが Gateway プロセスツリーの外部から
`openclaw update --yes --json` を実行します。引き継ぎを利用できない場合、
`update.run` は、手動で実行する安全なシェルコマンドを含む構造化レスポンスを返します。

保存された extended-stable の選択では、`update.checkOnStart` が有効な場合、起動時および24時間ごとに読み取り専用の更新
ヒントを受け取ります。これらのチェックが更新を適用したり、
ハンドオフを開始したり、Gateway を再起動したり、stable の遅延／ジッターを使用したり、beta の
ポーリング間隔を使用したりすることはありません。明示的なフォアグラウンド更新、保存された
`update.channel: "extended-stable"` を使用する引数なしのフォアグラウンド更新、オンデマンドのステータス確認、およびそれらの管理対象
Gateway ハンドオフは引き続きサポートされます。

ローカルの管理対象 Gateway サービスがインストールされ、再起動が有効な場合、
パッケージマネージャーおよび git チェックアウトの更新では、パッケージツリーを
置き換えたり、チェックアウト／ビルド出力を変更したりする前に、実行中のサービスを停止します。続いてアップデーターは
サービスメタデータを更新し、サービスを再起動して、再起動後の
Gateway を検証してから `Gateway: restarted and verified.` を報告します。
パッケージマネージャーの更新ではさらに、再起動後の Gateway が想定される
パッケージバージョンを報告することを検証します。git チェックアウトの更新では、再ビルド後に Gateway の健全性と
サービスの準備完了状態を検証します。

パッケージマネージャーの更新では通常、管理対象サービスに記録されている Node バイナリを
引き続き使用します。その Node では対象リリースを実行できないものの、現在の
CLI の Node では実行でき、かつサービスが更新対象パッケージに属することが確認された場合、
再起動が有効な更新では、最終処理に現在の Node を使用し、
サービスメタデータをそのランタイムに書き換えます。`--no-restart` ではサービス
メタデータを修復できないため、同じランタイム不一致が発生すると、パッケージを変更する前に停止します。

macOS では、更新後のチェックで、アクティブなプロファイルに対して LaunchAgent が
読み込まれ、実行中であること、および設定されたループバックポートが
正常であることも検証します。plist がインストールされていても launchd が監視していない場合、OpenClaw は
LaunchAgent を自動的に再ブートストラップし、健全性／バージョン／
チャネル準備完了チェックを再実行します（新規ブートストラップでは `RunAtLoad` ジョブを直接読み込むため、
復旧処理によって新しく生成された Gateway が直ちに `kickstart -k` されることはありません）。それでも
Gateway が正常にならない場合、コマンドは非ゼロで終了し、
再起動ログのパスに加えて、再起動、再インストール、およびパッケージのロールバック
手順を出力します。

再起動を実行できない場合、コマンドは手動の `openclaw gateway restart` ヒントとともに
`Gateway: restart skipped (...)` または `Gateway: restart failed: ...` を出力します。
`--no-restart` を指定すると、パッケージの置換または git の再ビルドは引き続き実行されますが、
管理対象サービスは停止も再起動もされないため、手動で再起動するまで実行中の Gateway は古い
コードを使用し続けます。

### コントロールプレーンのレスポンス形式

パッケージマネージャーによるインストールまたは監視対象の git チェックアウトで、`update.run` が Gateway コントロールプレーン経由で
実行される場合、ハンドラーは Gateway の終了後も続行される CLI 更新とは
別に、ハンドオフの開始を報告します。

- `ok: true`、`result.status: "skipped"`、
  `result.reason: "managed-service-handoff-started"`、および
  `handoff.status: "started"`：Gateway は管理対象サービスのハンドオフを作成し、
  切り離されたヘルパーが稼働中のサービスプロセス外で
  `openclaw update --yes --json` を実行できるよう、自身の再起動をスケジュールしました。
- `ok: false`、`result.reason: "managed-service-handoff-unavailable"`、および
  `handoff.status: "unavailable"`：OpenClaw は、安全なハンドオフに必要な監視サービス境界と
  永続的なサービス ID を見つけられませんでした（たとえば、
  systemd のハンドオフには、単なる周辺の systemd プロセスマーカーではなく、
  `OPENCLAW_SYSTEMD_UNIT` ユニット ID が必要です）。レスポンスには、
  Gateway の外部から実行するシェルコマンドである
  `handoff.command` が含まれます。
- `ok: false`、`result.reason: "managed-service-handoff-failed"`：Gateway は
  ハンドオフの作成を試みましたが、切り離されたヘルパーを起動できませんでした。

`sentinel` ペイロードは Gateway の終了前に書き込まれ、CLI
ハンドオフは、管理対象サービスの再起動後の健全性チェックが完了すると、同じ再起動センチネルを
更新します。ハンドオフ中、センチネルには成功時の継続処理なしで
`stats.reason: "restart-health-pending"` が格納されることがあります。再起動した
Gateway はこれをポーリングし、CLI がサービスの健全性を検証して、
最終的な `ok` の結果でセンチネルを書き換えた後にのみ、継続処理を実行します。
センチネルが保留中または失敗している間、`openclaw status` と `openclaw status --all` は
`Update restart` 行を表示し、`update.status` は更新して
最新のセンチネルを返します。

## Git チェックアウトのフロー

### チャネルの選択

- `stable`：最新の非 beta タグをチェックアウトし、ビルドして doctor を実行します。
- `beta`：最新の `-beta` タグを優先し、beta が存在しないか、
  stable の最新タグより古い場合は、stable の最新タグにフォールバックします。
- `dev`：`main` をチェックアウトしてから、フェッチとリベースを実行します。
- `extended-stable`：Git チェックアウトではサポートされません。チェックアウトの変更は
  行われません。

### 更新手順

<Steps>
  <Step title="クリーンなワークツリーを検証">
    コミットされていない変更がないことが必要です。
  </Step>
  <Step title="チャネルを切り替え">
    選択したチャネル（タグまたはブランチ）に切り替えます。
  </Step>
  <Step title="上流をフェッチ">
    dev のみです。
  </Step>
  <Step title="事前ビルド（dev のみ）">
    一時ワークツリーで TypeScript ビルドを実行します。先端が失敗した場合、最大10コミット遡り、ビルド可能な最新コミットを探します。この事前チェック中に lint も実行するには、`OPENCLAW_UPDATE_PREFLIGHT_LINT=1` を設定します。ユーザーの更新ホストは CI ランナーより小規模なことが多いため、lint はリソースを制限した直列モードで実行されます。
  </Step>
  <Step title="リベース">
    選択したコミット上にリベースします（dev のみ）。
  </Step>
  <Step title="依存関係をインストール">
    リポジトリのパッケージマネージャーを使用します。pnpm チェックアウトの場合、アップデーターは pnpm ワークスペース内で `npm run build` を実行する代わりに、必要に応じて `pnpm` をブートストラップします（最初に `corepack` を使用し、次に一時的な `npm install pnpm@11` へフォールバックします）。pnpm のブートストラップがそれでも失敗した場合、アップデーターはチェックアウト内で `npm run build` を試行せず、パッケージマネージャー固有のエラーで早期に停止します。
  </Step>
  <Step title="Control UI をビルド">
    Gateway と Control UI をビルドします。
  </Step>
  <Step title="doctor を実行">
    最終的な安全更新チェックとして `openclaw doctor` を実行します。
  </Step>
  <Step title="plugins を同期">
    plugins をアクティブなチャネルに同期します。dev では同梱 plugins を使用し、stable と beta では npm を使用します。追跡対象の Plugin インストールを更新します。
  </Step>
</Steps>

### Plugin 同期の詳細

beta チャネルでは、default/latest 系列に従う追跡対象の npm および ClawHub Plugin インストールに対して、
最初に Plugin の `@beta` リリースを試します。Plugin に
beta リリースがない場合、OpenClaw は記録された default/latest 指定にフォールバックし、
警告を報告します。npm plugins では、beta
パッケージが存在してもインストール検証に失敗した場合にも OpenClaw がフォールバックします。これらのフォールバック警告によって
コアの更新が失敗することはありません。正確なバージョンと明示的なタグは書き換えられません。

<Warning>
正確に固定された npm Plugin の更新が、保存済みのインストール記録と整合性が異なる成果物に解決された場合、`openclaw update` はその Plugin 成果物をインストールせず、更新を中止します。新しい成果物を信頼できることを確認した後にのみ、Plugin を明示的に再インストールまたは更新してください。
</Warning>

<Note>
管理対象 Plugin に限定され、同期パスが回避可能な更新後の Plugin 同期失敗（たとえば、必須ではない Plugin に対する npm レジストリへ到達できない場合）は、コア更新の成功後に警告として報告されます。JSON の結果では、トップレベルの更新 `status: "ok"` を維持し、`openclaw update repair` および `openclaw plugins inspect <id> --runtime --json` のガイダンスとともに `postUpdate.plugins.status: "warning"` を報告します。予期しないアップデーターまたは同期の例外では、引き続き更新結果が失敗になります。Plugin のインストールまたは更新エラーを修正してから、`openclaw update repair` を再実行してください。失敗した更新によって管理対象 Plugin が使用不能になった場合、OpenClaw は運用者が作成した `plugins.allow` または `plugins.deny` ポリシーを変更せずに、そのランタイムエントリを無効化し、アクティブスロットをリセットします。

Plugin ごとの同期手順後、Gateway の再起動前に、`openclaw update` は必須の**コア更新後の収束**処理を実行します。これにより、欠落している設定済み Plugin ペイロードを修復し、ディスク上の各_アクティブな_追跡対象インストール記録を検証し、その `package.json` が解析可能であること（および明示的に宣言された `main` が存在すること）を静的に検証します。この処理での失敗、および無効な設定スナップショットでは、`postUpdate.plugins.status: "error"` が返され、トップレベルの更新 `status` が `"error"` に切り替わります。そのため、`openclaw update` は非ゼロで終了し、未検証の Plugin セットで Gateway が再起動されることはありません。エラーには、`openclaw update repair` および `openclaw plugins inspect <id> --runtime --json` を指す構造化された `postUpdate.plugins.warnings[].guidance` 行が含まれます。無効化された Plugin エントリ、および信頼済みソースにリンクされた公式同期対象ではない記録は、ここではスキップされます（欠落ペイロードのチェックで使用される `skipDisabledPlugins` ポリシーと同じです）。そのため、古い無効化済み Plugin の記録によって、それ以外は有効な更新が妨げられることはありません。

更新された Gateway の起動時、Plugin の読み込みは検証専用です。起動処理でパッケージマネージャーを実行したり、依存関係ツリーを変更したりすることはありません。パッケージマネージャーの `update.run` 再起動は CLI の管理対象サービスパスに引き渡されるため、パッケージの交換は古い Gateway プロセスの外部で行われ、サービスの健全性チェックによって更新を完了として報告できるかどうかが決まります。
</Note>

extended-stable のコア更新が成功すると、コア更新後の Plugin 整合性チェックと
収束処理は、対象となる公式 npm plugins に対し、インストール済みコアの正確な
バージョンを使用します。default/`latest` の意図の場合、OpenClaw は Plugin の
`@extended-stable` を照会せず、npm の `latest` にもフォールバックしません。パッケージバージョンは
インストール済みコアから導出されます。明示的なバージョン固定、明示的な非 `latest` タグ、
サードパーティ製パッケージ、および npm 以外のソースでは、既存の意図が維持されます。

パッケージマネージャーによるインストールでは、`openclaw update` がパッケージマネージャーを
呼び出す前に、対象パッケージのバージョンを解決します。npm のグローバルインストールでは段階的
インストールを使用します。OpenClaw は新しいパッケージを一時的な npm prefix にインストールし、
候補パッケージが `preinstall` 中にホストの Node バージョンを検証できるようにして、
そこでパッケージ化された `dist` インベントリを検証します。パッケージ化された完了ガードは
`preinstall` が成功するまでそのインベントリの外部に置かれるため、ライフサイクルスクリプトを
スキップするパッケージマネージャーもアクティベーション前に停止します。npm 12 以降では、
アップデーターは候補 OpenClaw のライフサイクルのみを承認し、推移的な
依存関係のスクリプトは引き続きブロックされます。その後、OpenClaw はクリーンなパッケージツリーを
実際のグローバル prefix に交換します。検証に失敗した場合、更新後の doctor、Plugin
同期、および再起動処理が疑わしいツリーから実行されることはありません。
インストール済みバージョンがすでに対象と一致している場合でも、コマンドは
グローバルパッケージのインストールを更新してから、Plugin 同期、コアコマンドの補完情報の
更新、および再起動処理を実行します。これにより、完全な
Plugin コマンドの補完情報再ビルドは明示的な
`openclaw completion --write-state` の実行に委ねながら、パッケージ化されたサイドカーとチャネル所有の
Plugin 記録を、インストール済みの OpenClaw ビルドと整合させます。

## 関連項目

- `openclaw doctor`（git チェックアウトでは先に更新を実行するよう提案します）
- [開発チャネル](/ja-JP/install/development-channels)
- [更新](/ja-JP/install/updating)
- [CLI リファレンス](/ja-JP/cli)
