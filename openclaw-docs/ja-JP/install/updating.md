---
read_when:
    - OpenClaw の更新
    - アップデート後に問題が発生する
summary: OpenClaw を安全に更新する方法（グローバルインストールまたはソース）とロールバック戦略
title: 更新中
x-i18n:
    generated_at: "2026-07-26T09:06:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83444d56e0aa34f47830610538b0c3012903abb812bfe0fffb8163a5db9ac2db
    source_path: install/updating.md
    workflow: 16
---

OpenClaw を最新の状態に保ちます。

Docker、Podman、Kubernetes のイメージ置換については、
[コンテナイメージのアップグレード](/ja-JP/install/docker#upgrading-container-images)を参照してください。Gateway は準備完了状態になる前に起動時に安全なアップグレード処理を実行し、マウントされた
状態に手動修復が必要な場合は終了します。

## 推奨: `openclaw update`

インストール種別（npm、pnpm、Bun、git）を検出し、最新バージョンを取得して `openclaw doctor` を実行し、Gateway を再起動します。

```bash
openclaw update
```

チャンネルを切り替えるか、特定のバージョンを指定します。

```bash
openclaw update --channel beta
openclaw update --channel extended-stable
openclaw update --channel dev
openclaw update --dry-run   # 適用せずにプレビュー
```

`openclaw update` には `--verbose` フラグがありません（インストーラーにはあります）。診断には、
予定されている操作をプレビューする `--dry-run`、構造化された結果を得る `--json`、または
チャンネルと利用可能性の状態を確認する `openclaw update status --json` を使用します。

`--channel beta` は beta の npm dist-tag を優先しますが、beta タグがない場合、またはそのバージョンが最新の stable
リリースより古い場合は stable/latest にフォールバックします。生の npm
beta dist-tag に固定した一度限りのパッケージ更新には、代わりに `--tag beta` を使用します。

`--channel extended-stable` はパッケージ専用であり、インストールは引き続き
フォアグラウンドでのみ実行されます。OpenClaw は公開 npm の `extended-stable` セレクターを読み取り、
選択された正確なパッケージを検証し、その正確なバージョンをインストールします。レジストリデータが欠落している場合や
整合しない場合は安全側に倒して失敗し、`latest` にフォールバックすることはありません。
選択されたバージョンがインストール済みバージョンより古い場合は、通常の
ダウングレード確認が引き続き適用されます。CLI はコアの更新が
成功するとチャンネルを保存しますが、`npm install -g openclaw@extended-stable` を直接実行しても
`update.channel` は更新されません。
コアの置換後、bare/default または
`latest` の意図を持つ対象の公式 npm Plugin は、その正確なコアバージョンに収束します。正確な固定指定と、明示的な
非 `latest` タグ、サードパーティ Plugin、npm 以外のソースは変更されません。
現在の OpenClaw バージョンで作成されたカタログインストールは、そのデフォルトの
意図を保持します。正確なバージョンしか含まない古いレコードは、
OpenClaw が古い自動固定とユーザーによる固定を安全に区別できないため、固定されたままになります。その Plugin を正確なコアバージョンの追跡対象へ戻すには、
extended-stable チャンネルで `openclaw plugins update @openclaw/name` を一度実行します。

`--channel dev` は、継続的に移動する GitHub `main` チェックアウトを提供します。一度限りの
パッケージ更新では、`--tag main` が `github:openclaw/openclaw#main` パッケージ
指定にマッピングされ、対象のパッケージマネージャー（npm/pnpm/bun）を介して直接インストールされます。

管理対象 Plugin では、beta リリースがないことは警告であり、失敗ではありません。
Plugin が記録済みの default/latest リリースにフォールバックしても、コアの更新は成功できます。

チャンネルのセマンティクスについては、[リリースチャンネル](/ja-JP/install/development-channels)を参照してください。

## npm インストールと git インストールの切り替え

インストール種別を変更するにはチャンネルを使用します。アップデーターは `~/.openclaw` 内の状態、設定、
認証情報、ワークスペースを保持し、CLI と Gateway が使用する OpenClaw
コードのインストールのみを変更します。

```bash
# npm パッケージインストール -> 編集可能な git チェックアウト
openclaw update --channel dev

# git チェックアウト -> npm パッケージインストール
openclaw update --channel stable
```

最初にインストールモードの切り替えをプレビューします。

```bash
openclaw update --channel dev --dry-run
openclaw update --channel stable --dry-run
```

`dev` は git チェックアウトを確保してビルドし、その
チェックアウトからグローバル CLI をインストールします。`stable`、`extended-stable`、`beta` チャンネルはパッケージ
インストールを使用します。git チェックアウトでは、変更や
変換を行わずに extended-stable が拒否されます。Gateway がすでにインストールされている場合、`--no-restart` を渡さない限り、`openclaw update` は
サービスメタデータを更新して再起動します。

管理対象 Gateway サービスを伴うパッケージインストールでは、`openclaw update` は
そのサービスが使用するパッケージルートを対象にします。シェルの `openclaw` コマンドが
別のインストールから提供されている場合、アップデーターは両方のルートと管理対象
サービスの Node パスを表示し、パッケージを置き換える前に、その Node バージョンを対象リリースの
`engines.node` 要件と照合します。

## ソースチェックアウトのサーバー（リファレンススクリプト）

サーバー上の git チェックアウトから Gateway を直接実行しているチームは、そのチェックアウト内から
`scripts/update-gateway.sh` を使用して更新できます。これは効率的なソースサーバー更新の
リファレンスです。`pnpm build` が書き換える追跡対象のビルド出力を復元し、それ以外のローカル変更がある場合は安全側に倒して失敗し、
`main` を fast-forward（またはローカルのサーバーブランチを `origin/main` 上にリベース）し、依存関係を
インストールしてクリーンにビルドし、Gateway を再起動します。

```bash
ssh you@server 'cd /path/to/openclaw && scripts/update-gateway.sh'
```

カスタムサービスユニット向けに再起動処理を上書きするか、完全にスキップします。

```bash
OPENCLAW_UPDATE_RESTART_CMD='systemctl --user restart openclaw-gateway.service' scripts/update-gateway.sh
OPENCLAW_UPDATE_RESTART_CMD='' scripts/update-gateway.sh
```

通常の単一ユーザー向けソースインストールでは、代わりに `openclaw update --channel dev` を使用してください。
チェックアウト、ビルド、Gateway の再起動が自動的に管理されます。

## 代替方法: インストーラーを再実行

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

オンボーディングをスキップするには `--no-onboard` を追加します。特定のインストール種別を強制するには、
`--install-method git --no-onboard` または `--install-method npm --no-onboard` を渡します。

npm パッケージのインストール段階後に `openclaw update` が失敗する場合は、
代わりにインストーラーを再実行します。これはアップデーターを呼び出さず、グローバルパッケージの
インストールを直接実行するため、部分的に更新された npm インストールを復旧できます。

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm
```

`--version` を使用して、復旧処理を特定のバージョンまたは dist-tag に固定します。

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm --version <version-or-dist-tag>
```

## 代替方法: npm、pnpm、bun を手動で使用

```bash
npm i -g openclaw@latest
```

管理下のインストールには `openclaw update` を推奨します。実行中の Gateway サービスと
パッケージの置換を連携できます。管理下のインストールを手動で更新する場合は、
まず管理対象 Gateway を停止します。パッケージマネージャーはファイルをその場で置き換えるため、
実行中の Gateway が置換途中のコアファイルや Plugin ファイルを読み込もうとする可能性があります。
パッケージマネージャーの完了後に Gateway を再起動し、
新しいインストールを読み込ませます。

root が所有する Linux のシステムグローバルインストールで、`openclaw update` が
`EACCES` で失敗する場合は、手動置換中に Gateway を停止したまま、
システム npm を使用して復旧します。その Gateway で通常使用しているものと同じプロファイルフラグや環境を使用します。
`/usr/bin/npm` は、ホスト上で root 所有のグローバルプレフィックスを管理する
システム npm に置き換えてください。

```bash
openclaw gateway stop
sudo /usr/bin/npm i -g openclaw@latest
openclaw gateway install --force
openclaw gateway restart
```

続いて検証します。

```bash
openclaw --version
curl -fsS http://127.0.0.1:18789/readyz
openclaw plugins list --json
openclaw gateway status --deep --json
openclaw doctor --lint --json
```

`openclaw update` がグローバル npm インストールを管理する場合、まず対象を
一時的な npm プレフィックスにインストールします。候補パッケージは `preinstall` 中にホストの
Node バージョンを検証します。その後にのみ、OpenClaw はパッケージ化された
`dist` インベントリを検証し、クリーンなパッケージツリーを実際のグローバルプレフィックスへ置き換えます。
パッケージ化された完了ガードは想定インベントリから除外され、`preinstall` が成功した後にのみ
削除されるため、ライフサイクルスクリプトがスキップされた場合も置換前に失敗します。npm 12 以降では、アップデーターは候補となる OpenClaw の
ライフサイクルのみを許可し、推移的依存関係のスクリプトは引き続きブロックします。これにより、npm が
古いパッケージの残存ファイル上に新しいパッケージを重ねて配置することを防ぎます。インストール
コマンドが失敗した場合、OpenClaw は `--omit=optional` を使用して一度再試行します。これは
ネイティブのオプション依存関係をコンパイルできないホストで役立ちます。

OpenClaw が管理する npm 更新コマンドと Plugin 更新コマンドは、子 npm プロセス向けに
npm の `min-release-age` サプライチェーン隔離（または旧 `before` 設定キー）も解除します。
このポリシーは一般的な保護のために存在しますが、明示的な OpenClaw 更新は
「選択したリリースを今すぐインストールする」ことを意味します。

```bash
pnpm add -g openclaw@latest
```

pnpm 11 で OpenClaw 2026.7.1 をインストールした場合は、この手動コマンドを一度実行してください。この
リリースは pnpm 11 の分離されたグローバルパッケージレイアウトより前のものであるため、アップデーターが
別の npm インストールを実行中の CLI と誤認する可能性があります。それ以降のリリースでは、
pnpm の所有権を維持し、更新中は置換対象パッケージのルートに追従します。また、
所有元マネージャーが報告するグローバル bin ディレクトリを使用し、利用可能な pnpm コマンドが別のグローバルルートまたはメジャーバージョンを報告する場合、
あるいは呼び出し元パッケージが孤立している場合や、その場所で唯一のアクティブな OpenClaw
インストールではない場合は、変更前に停止します。

OpenClaw が pnpm 11 のグローバルインストールグループを別のパッケージと共有している場合、
自動アップデーターはグループを変更する前に停止します。元の
カンマ区切りグループを手動で更新し、関連パッケージとビルドポリシーを
維持してください。

```bash
bun add -g openclaw@latest
```

### npm インストールの高度なトピック

<AccordionGroup>
  <Accordion title="読み取り専用パッケージツリー">
    OpenClaw は、現在のユーザーがグローバルパッケージディレクトリに書き込める場合でも、実行時にはパッケージ化されたグローバルインストールを読み取り専用として扱います。Plugin パッケージのインストール先は、ユーザー設定ディレクトリ配下の OpenClaw 所有の npm/git ルートであり、Gateway の起動時に OpenClaw のパッケージツリーが変更されることはありません。

    一部の Linux npm 環境では、`/usr/lib/node_modules/openclaw` などの root 所有ディレクトリ配下にグローバルパッケージがインストールされます。Plugin のインストールおよび更新コマンドは、そのグローバルパッケージディレクトリの外部へ書き込むため、OpenClaw はこのレイアウトをサポートします。

  </Accordion>
  <Accordion title="強化された systemd ユニット">
    明示的な Plugin のインストール、Plugin の更新、doctor のクリーンアップで変更を永続化できるように、OpenClaw に設定および状態ルートへの書き込み権限を付与します。

    ```ini
    ReadWritePaths=/var/lib/openclaw /home/openclaw/.openclaw /tmp
    ```

  </Accordion>
  <Accordion title="ディスク容量の事前確認">
    パッケージの更新および明示的な Plugin のインストール前に、OpenClaw は対象ボリュームのディスク容量をベストエフォートで確認します。容量が少ない場合は確認対象のパスを含む警告が表示されますが、ファイルシステムのクォータ、スナップショット、ネットワークボリュームは確認後に変化する可能性があるため、更新はブロックされません。実際のパッケージマネージャーによるインストールとインストール後の検証が引き続き最終的な判断基準となります。
  </Accordion>
</AccordionGroup>

## 自動アップデーター

デフォルトでは無効です。`~/.openclaw/openclaw.json` で有効にします。

```json5
{
  update: {
    channel: "stable",
    auto: {
      enabled: true,
    },
  },
}
```

| チャンネル           | 動作                                                                                                                      |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `stable`          | 分散ロールアウトのため、決定論的なジッターを伴う組み込みの遅延後に適用します。                                                |
| `extended-stable` | `checkOnStart` が有効な場合、起動時と 24 時間ごとに読み取り専用の更新通知を確認します。自動的に適用することはありません。 |
| `beta`            | 組み込みの間隔で確認し、直ちに適用します。                                                                        |
| `dev`             | 自動適用しません。`openclaw update` を手動で使用します。                                                                           |

Gateway は起動時にも更新ヒントをログに記録します（無効にするには
`update.checkOnStart: false` を使用します）。保存された extended-stable の選択では、この
読み取り専用のヒント経路と既存の 24 時間のヒント間隔を使用しますが、
自動インストール、ハンドオフ、再起動、stable の遅延／ジッター、beta のポーリングは一切実行しません。
ダウングレードまたはインシデント復旧では、`update.auto.enabled` が設定されている場合でも自動適用をブロックするため、Gateway 環境で `OPENCLAW_NO_AUTO_UPDATE=1` を設定します。`update.checkOnStart` も無効にしない限り、起動時の更新ヒントは引き続き実行されます。

稼働中の Gateway コントロールプレーン
（`update.run`）を通じて要求されたパッケージマネージャー更新では、実行中の Gateway
プロセス内のパッケージツリーは置き換えられません。管理対象サービスとしてインストールされている場合、Gateway は切り離されたハンドオフを開始して
終了し、通常の `openclaw update --yes --json` CLI 経路により、
サービスの停止、パッケージの置き換え、サービスメタデータの更新、再起動、Gateway のバージョンと
到達可能性の検証、および可能な場合はインストール済みだが読み込まれていない macOS
LaunchAgent の復旧を行います。Gateway がそのハンドオフを安全に実行できない場合、
`update.run` はパッケージマネージャーをプロセス内で実行せず、安全なシェルコマンドを
報告します。

Control UI のサイドバーにある更新カードは、この `update.run` フローを直接開始する場合、
**Gateway を更新**と表示します。これは、ブラウザーでホストされる Control UI、リモート
Gateway、および手動管理されるローカル Gateway が対象です。

署名済み macOS アプリでは、アプリが所有するローカル Gateway の場合、そのカードは
**Mac アプリ + Gateway を更新**に変わります。Sparkle は最初にアプリを更新します。再起動後、アプリは
`openclaw update --tag <app-version> --json` を実行して Gateway を再起動し、
セットアップ形式の進行状況ウィンドウで正常性を検証します。このウィンドウは、
その管理対象 Gateway に更新、修復、またはインストールが必要な場合にのみ表示されます。アプリのみの更新では、
再起動後に直接アプリが開きます。失敗の詳細は、再試行、[更新ガイド](/ja-JP/install/updating)、および
[Discord](https://discord.gg/clawd) のアクションとともに表示されたままになります。アプリは、リモートまたは外部管理される Gateway にこの連携
経路を使用せず、新しいバージョンの Gateway をダウングレードせず、`extended-stable` のチャンネル固定を
上書きすることもありません。

更新が成功すると、アプリは実際のユーザー／チャンネル操作がある直近の
最上位ダイレクトセッションに対して、1 回限りのウェルカムイベントをキューに追加します。Cron の実行、
Heartbeat、およびバックグラウンドのみのセッション更新では、この選択は変更されません。
リモートモードでは、アプリはローカル Mac Node ランタイムのみを更新し、接続先のリモート Gateway がアプリと同じかそれより新しい場合にのみ
イベントを送信します。

## 更新後

<Steps>

### doctor を実行する

```bash
openclaw doctor
```

設定を移行し、DM ポリシーを監査して、Gateway の正常性を確認します。詳細：[Doctor](/ja-JP/gateway/doctor)

### Gateway を再起動する

```bash
openclaw gateway restart
```

### 検証する

```bash
openclaw health
```

</Steps>

## ロールバック

ロールバックには 2 つの層があります。

1. 現在の状態を維持したまま、古い OpenClaw コードを再インストールします。
2. 古いコードが移行済みの
   設定またはデータベースを使用できない場合にのみ、更新前の状態を復元します。

まずコードのみをロールバックします。状態を復元すると、バックアップ後に行われた変更は
破棄されます。

### 更新前：検証済みバックアップを作成する

`openclaw update` は更新前の設定コピーを自動的に保持しますが、
完全な状態復旧ポイントは作成しません。重要な更新の前には、明示的に作成します。

```bash
mkdir -p ~/Backups/openclaw
openclaw backup create --output ~/Backups/openclaw --verify
```

アーカイブマニフェストには、OpenClaw のバージョンとバックアップに含まれる
ソースパスが記録されます。アーカイブには認証情報、認証プロファイル、チャンネルの
状態が含まれる場合があるため、所有者のみの権限を設定し、稼働中の状態ディレクトリと同等の保護を施して保存してください。含まれるファイルと意図的に
除外されるファイルについては、[バックアップ](/ja-JP/cli/backup)を参照してください。

ポータブルアーカイブでは除外される揮発性アーティファクトも含む、バイト単位で完全な復旧ポイントを作成するには、
Gateway を停止し、プラットフォームが提供するファイルシステム、ボリューム、または VM の
スナップショットを使用します。

### パッケージインストールをロールバックする

公開済みバージョンを一覧表示し、既知の正常なバージョンをプレビューしてインストールします。

```bash
npm view openclaw versions --json
openclaw update --tag <known-good-version> --dry-run
openclaw update --tag <known-good-version>
```

パッケージマネージャーで直接インストールするより、`openclaw update --tag` の使用が推奨されます。これは
ダウングレードを検出して確認を求め、インストール対象に対する管理対象 Plugin の収束処理と
互換性チェックを実行し、サービス
メタデータを更新して Gateway を再起動し、実行中のバージョンを検証します。保存された
チャンネルが `extended-stable` の場合は、
`--channel stable --tag <known-good-version>` を使用してください。1 回限りの完全一致タグは
`extended-stable` セレクターと組み合わせられないためです。

パッケージ更新では、有効化する前に候補をステージングして検証します。
ファイルシステムの入れ替えまたはコマンド shim の置き換えに失敗した場合、OpenClaw は古い
パッケージを自動的に復元します。入れ替えに成功した後、後続の Gateway 正常性チェックに失敗した場合は、
パッケージを再度自動的に置き換えるのではなく、以前のバージョンと手動ロールバック手順を
報告します。

CLI の更新経路が利用できない場合は、現在の Gateway を所有するものと同じパッケージマネージャーとインストール
スコープを使用します。

```bash
openclaw gateway stop
npm i -g openclaw@<known-good-version>
openclaw gateway install --force
openclaw gateway restart
```

そのマネージャーがインストールを所有している場合は、`npm` を `pnpm` または `bun` に置き換えます。
インシデント復旧中に、有効化された自動アップデーターが新しいリリースを即座に適用するのを防ぐには、
Gateway 環境で `OPENCLAW_NO_AUTO_UPDATE=1` を設定します。

### ソースチェックアウトをロールバックする

クリーンなチェックアウトを使用し、既知の正常なタグまたはコミットを選択します。

```bash
git fetch --all --tags
git checkout --detach <known-good-tag-or-commit>
pnpm install && pnpm build
openclaw gateway restart
```

最新バージョンに戻すには、`git checkout main && git pull` を使用します。

git 更新の開始後に依存関係のインストール、ビルド、UI ビルド、または doctor が失敗した場合、
アップデーターは git チェックアウトを以前のブランチと
SHA に自動的に戻します。意図的に古いコミットを選択する場合は、引き続き手動チェックアウトが
必要です。

### セッション SQLite 移行をまたいでダウングレードする

ファイルベースの古い OpenClaw リリースを起動する前に、現在の CLI を使用して
アーカイブ済みの旧トランスクリプトアーティファクトを復元します。

```bash
openclaw gateway stop
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

この操作では SQLite データは削除されません。SQLite 移行後に作成されたセッションは
SQLite にのみ存在し、古いランタイムには表示されません。
[セッション SQLite 移行後のダウングレード](/ja-JP/cli/doctor#downgrading-after-session-sqlite-migration)を参照してください。

### 必要な場合にのみ状態を復元する

古いコードが新しい設定またはデータベーススキーマを読み取れない場合は、
Gateway を停止し、検証済みの更新前ファイルシステム、ボリューム、または VM スナップショットを復元します。
復元するとスナップショット後に行われた変更が削除されるため、事前に現在の状態を別途
保存してください。

広範な `openclaw backup create` アーカイブは作成と検証をサポートしますが、
アーカイブ全体をその場で有効化する機能はサポートしません。広範なアーカイブをステージング
ディレクトリに展開し、その `manifest.json` のソースからアーカイブへのマッピングを使用してオフライン
復元を行います。同様に、`openclaw backup sqlite restore` は検証済みデータベースを
新しいターゲットに書き込みます。そのターゲットの有効化は、オペレーターが明示的に行うオフライン
手順です。

### ロールバックを検証する

```bash
openclaw --version
openclaw health
openclaw plugins list --json
openclaw gateway status --deep --json
openclaw doctor --lint --json
```

## 問題が解決しない場合

- `openclaw doctor` をもう一度実行し、出力を注意深く確認します。
- ソースチェックアウト上の `openclaw update --channel dev` では、必要に応じてアップデーターが `pnpm` を自動的にブートストラップします。pnpm/corepack のブートストラップエラーが表示された場合は、`pnpm` を手動でインストール（または `corepack` を再有効化）してから、更新を再実行します。
- 確認：[トラブルシューティング](/ja-JP/gateway/troubleshooting)
- Discord で質問：[https://discord.gg/clawd](https://discord.gg/clawd)

## 関連項目

- [インストールの概要](/ja-JP/install)：すべてのインストール方法。
- [Doctor](/ja-JP/gateway/doctor)：更新後の正常性チェック。
- [移行](/ja-JP/install/migrating)：メジャーバージョンの移行ガイド。
