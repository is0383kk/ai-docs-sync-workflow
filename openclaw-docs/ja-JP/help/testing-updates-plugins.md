---
read_when:
    - OpenClaw のアップデート、doctor、パッケージ受け入れ、または Plugin インストールの動作の変更
    - リリース候補の準備または承認
    - パッケージ更新、Plugin の依存関係整理、または Plugin のインストールに関するリグレッションのデバッグ
sidebarTitle: Update and plugin tests
summary: OpenClaw が更新パス、パッケージ移行、Plugin のインストールおよび更新動作を検証する方法
title: テスト：アップデートと plugins
x-i18n:
    generated_at: "2026-07-26T09:37:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 96a11fe42472f758d4fd1cc568486e301f7460982fdb547cab8b39de04a8dabe
    source_path: help/testing-updates-plugins.md
    workflow: 16
---

更新およびPlugin検証のチェックリスト：インストール可能なパッケージが実際のユーザー状態を
更新し、`doctor`を通じて古いレガシー状態を修復でき、さらに
サポートされるすべてのソースからPluginをインストール、読み込み、更新、アンインストールできることを証明します。

より広範なテストランナーの一覧については、[テスト](/ja-JP/help/testing)を参照してください。実際のプロバイダー
キーとネットワークにアクセスするスイートについては、[ライブテスト](/ja-JP/help/testing-live)を参照してください。

## 保護するもの

- パッケージのtarballが完全で、有効な`dist/postinstall-inventory.json`を持ち、
  展開されていないリポジトリファイルに依存していないこと。
- ユーザーが、設定、エージェント、セッション、ワークスペース、Pluginの許可リスト、
  またはチャンネル設定を失うことなく、以前に公開されたパッケージから候補パッケージへ
  移行できること。
- `openclaw doctor --fix --non-interactive`がレガシーなクリーンアップおよび修復
  パスを所有すること。起動処理で、古いPlugin状態のための隠れた互換性移行を増やしてはなりません。
- Pluginのインストールが、ローカルディレクトリ、gitリポジトリ、npmパッケージ、
  およびClawHubレジストリパスから機能すること。
- Pluginのnpm依存関係が、Pluginごとに1つの管理対象npmプロジェクトへインストールされ、
  信頼する前にスキャンされ、Pluginのアンインストール時に`npm uninstall`を通じて
  削除されるため、ホイストされた依存関係が残らないこと。
- 何も変更されていない場合、Pluginの更新が何も行わないこと：インストール記録、
  解決済みソース、インストール済みの依存関係レイアウト、および有効化状態が維持されます。

## 開発中のローカル検証

狭い範囲から開始します：

```bash
pnpm changed:lanes --json
pnpm check:changed
pnpm test:changed
```

Pluginのインストール、アンインストール、依存関係、またはパッケージインベントリを変更した場合は、
編集した境界を対象とするテストも実行します：

```bash
pnpm test src/plugins/uninstall.test.ts src/infra/package-dist-inventory.test.ts test/scripts/package-acceptance-workflow.test.ts
```

パッケージのDockerレーンがtarballを使用する前に、パッケージ成果物を検証します：

```bash
pnpm release:check
```

`release:check`は、設定／ドキュメント／APIの差異チェック（設定スキーマ、設定ドキュメントの
ベースライン、Plugin SDK APIの契約マニフェストとエクスポート、Pluginのバージョン／インベントリ）を実行し、
パッケージ配布物のインベントリを書き込み、`npm pack --dry-run`を実行し、禁止された
梱包ファイルを拒否し、tarballを一時プレフィックスへインストールしてpostinstallを実行し、
同梱チャンネルのエントリポイントをスモークテストします。

## Dockerレーン

Dockerレーンは、製品レベルの検証です。Linuxコンテナ内で実際の
パッケージをインストールまたは更新し、CLIコマンド、
Gatewayの起動、HTTPプローブ、RPCステータス、およびファイルシステム状態を通じて動作を検証します。

反復作業中は、対象を絞ったレーンを使用します：

```bash
pnpm test:docker:plugins
pnpm test:docker:plugin-lifecycle-matrix
pnpm test:docker:plugin-update
pnpm test:docker:upgrade-survivor
pnpm test:docker:published-upgrade-survivor
pnpm test:docker:update-restart-auth
pnpm test:docker:update-migration
```

重要なレーン：

- `test:docker:plugins`は、Pluginインストールのスモークテスト、ローカルフォルダーからのインストール、
  ローカルフォルダー更新のスキップ動作、依存関係が事前インストールされたローカルフォルダー、
  `file:`パッケージのインストール、CLI実行を伴うgitインストール、gitの
  移動参照の更新、ホイストされた推移的依存関係を伴うnpmレジストリからのインストール、
  npm更新時に何も行わない動作、不正なnpmパッケージメタデータの拒否、
  ローカルClawHubフィクスチャのインストールと更新時に何も行わない動作、マーケットプレイスの更新動作、
  およびClaudeバンドルの有効化／検査を対象とします。ClawHubブロックを
  完全に隔離されたオフライン状態に保つには、`OPENCLAW_PLUGINS_E2E_CLAWHUB=0`を設定します。
- `test:docker:plugin-lifecycle-matrix`は、最小構成のコンテナに候補パッケージを
  インストールし、npm Pluginに対してインストール、検査、無効化、有効化、
  明示的なアップグレード、明示的なダウングレード、およびPluginコードを削除した後のアンインストールを
  実行します。フェーズごとにRSSおよびCPUメトリクスを記録します。
- `test:docker:plugin-update`は、変更されていないインストール済みPluginが
  `openclaw plugins update`中に再インストールされたり、インストールメタデータを失ったりしないことを検証します。
- `test:docker:upgrade-survivor`は、汚れた
  旧ユーザーフィクスチャに候補tarballを上書きインストールし、パッケージ更新と非対話型doctorを実行した後、
  local loopbackのGatewayを起動して状態の保持を確認します。
- `test:docker:published-upgrade-survivor`は、まず公開済みのベースラインをインストールし、
  組み込みの`openclaw config set`レシピを通じて設定し、候補tarballへ更新し、
  doctorを実行してレガシーなクリーンアップを確認し、Gatewayを起動して
  `/healthz`、`/readyz`、およびRPCステータスをプローブします。
- `test:docker:update-restart-auth`は、候補パッケージをインストールし、
  管理対象のトークン認証Gatewayを起動し、`openclaw update --yes --json`のために
  呼び出し元のGateway認証環境変数を解除し、通常のプローブの前に
  候補の更新コマンドがGatewayを再起動することを必須とします。
- `test:docker:update-migration`は、クリーンアップを重点的に行う公開済み更新レーンです。
  設定済みのDiscord／Telegram形式のユーザー状態から開始し、設定済みPluginの依存関係を
  実体化できるようにベースラインのdoctorを実行し、設定済みのパッケージ化Pluginに対する
  レガシーなPlugin依存関係の残骸を作成し、候補tarballへ更新して、
  更新後のdoctorがレガシーな依存関係ルートを削除することを必須とします。

公開済みアップグレードサバイバーの便利なバリエーション：

```bash
OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC=openclaw@2026.4.23 \
OPENCLAW_UPGRADE_SURVIVOR_SCENARIO=versioned-runtime-deps \
pnpm test:docker:published-upgrade-survivor

OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC=openclaw@latest \
OPENCLAW_UPGRADE_SURVIVOR_SCENARIO=bootstrap-persona \
pnpm test:docker:published-upgrade-survivor
```

利用可能なシナリオ：`base`、`acpx-openclaw-tools-bridge`、`feishu-channel`、
`bootstrap-persona`、`channel-post-core-restore`、`plugin-deps-cleanup`、
`configured-plugin-installs`、`stale-source-plugin-shadow`、`tilde-log-path`、
および`versioned-runtime-deps`。集約実行では、`OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues`
（別名`far-reaching`）が、設定済みPluginのインストール移行を含む
すべてのシナリオに展開されます。

完全な更新移行は、意図的にFull Release CIとは分離されています。リリース上の問いが
「2026.4.23以降に公開されたすべての安定版リリースからこの候補へ更新し、
Plugin依存関係の残骸をクリーンアップできるか？」である場合は、手動の
`Update Migration`ワークフローを使用します：

```bash
gh workflow run update-migration.yml \
  --ref main \
  -f workflow_ref=main \
  -f package_ref=main \
  -f baselines=all-since-2026.4.23 \
  -f scenarios=plugin-deps-cleanup
```

## パッケージ受け入れ

パッケージ受け入れは、GitHubネイティブのパッケージゲートです。1つの候補
パッケージを`package-under-test` tarballへ解決し、バージョンとSHA-256を記録してから、
その厳密なtarballに対して再利用可能なDocker E2Eレーンを実行します。ワークフローハーネスの
参照はパッケージソースの参照から分離されているため、現在のテストロジックで
信頼済みの古いリリースを検証できます。

候補ソース：

- `source=npm`：`openclaw@extended-stable`、`openclaw@beta`、
  `openclaw@latest`、または公開済みの正確なバージョンを検証します。
- `source=ref`：選択した現在の
  ハーネスを使用して、信頼済みのブランチ、タグ、またはコミットをパックします。
- `source=url`：必須の`package_sha256`を使用して、公開HTTPS tarballを検証します。
  このパスは、URL認証情報、デフォルト以外のHTTPSポート、プライベート／内部
  ホスト名またはDNS／IP結果、特殊用途IP空間、および安全でないリダイレクトを拒否します。
- `source=trusted-url`：必須の
  `package_sha256`および`trusted_source_id`を使用し、`.github/package-trusted-sources.json`内のメンテナー所有ポリシーに照らして
  HTTPS tarballを検証します。入力レベルのプライベート許可
  スイッチで`source=url`を弱める代わりに、エンタープライズ／プライベート
  ミラーにはこれを使用します。ポリシーで設定されている場合、Bearer認証は固定の
  `OPENCLAW_TRUSTED_PACKAGE_TOKEN`シークレットを使用します。
- `source=artifact`：別のActions実行によってアップロードされたtarballを再利用します。

Full Release Validationは、解決されたリリースSHAからビルドされた
`source=artifact`をデフォルトで使用します。公開後の検証では、
`package_acceptance_package_spec=openclaw@YYYY.M.PATCH`を渡して、同じアップグレードマトリクスが
出荷済みのnpmパッケージを対象とするようにします。

リリースチェックは、次のパッケージ／更新／再起動／Pluginセットでパッケージ受け入れを呼び出します：

```text
doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape
```

リリースのソークテストが有効な場合（`release_profile=stable`および
`full`では強制的に有効）、次も渡します：

```text
published_upgrade_survivor_baselines=last-stable-4 2026.4.23 2026.5.2 2026.4.15
published_upgrade_survivor_scenarios=reported-issues
telegram_mode=mock-openai
```

これにより、デフォルトのリリースパッケージゲートですべての公開済みリリースを
走査することなく、パッケージ移行、更新チャンネルの切り替え、破損した管理対象Pluginへの
耐性、古いPlugin依存関係のクリーンアップ、オフラインPluginのカバレッジ、Pluginの
更新動作、およびTelegramパッケージQAを同じ解決済み成果物に対して維持できます。

`last-stable-4`は、npmで公開された最新4つのOpenClaw安定版
リリースに解決されます。リリースのパッケージ受け入れでは、`2026.4.23`を最初のPlugin更新
互換性境界として、`2026.5.2`をPluginアーキテクチャの変動境界として、
`2026.4.15`を古い2026.4.1x公開済み更新ベースラインとして固定します。リゾルバーは、
最新4つにすでに含まれる固定値を重複排除します。公開済み更新移行を網羅的に検証するには、
Full Release CIではなく、独立したUpdate Migrationワークフローで`all-since-2026.4.23`を
使用します。レガシーな日付以前の基準点も含めて、より広範囲を手動で
サンプリングする場合は、`release-history`も引き続き使用できます。

公開済みアップグレードサバイバーのベースラインを複数選択した場合、再利用可能な
Dockerワークフローは、各ベースラインを個別の対象ランナージョブへ分割します。各
ベースラインシャードでは選択されたシナリオセットが引き続き実行されますが、ログと成果物は
ベースラインごとに保持され、実時間は1つの大きな直列ジョブではなく、
最も遅いシャードによって制限されます。

リリース前に候補を検証する場合は、パッケージプロファイルを手動で実行します：

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=package \
  -f published_upgrade_survivor_baselines="last-stable-4 2026.4.23 2026.5.2 2026.4.15" \
  -f published_upgrade_survivor_scenarios=reported-issues \
  -f telegram_mode=mock-openai
```

公開済みextended-stableカナリアの場合は、
`package_spec=openclaw@extended-stable`を設定します。パッケージ受け入れは、Dockerレーンを実行する前に、
そのセレクターを正確なtarballへ解決します。

リリース上の問いにMCPチャンネル、cron／サブエージェントのクリーンアップ、
OpenAIウェブ検索、またはOpenWebUIが含まれる場合は、`suite_profile=product`を使用します。
Dockerによるリリースパス全体のカバレッジが必要な場合にのみ、`suite_profile=full`を使用します。

## リリースのデフォルト

リリース候補のデフォルトの検証スタックは次のとおりです：

1. ソースレベルの回帰には、`pnpm check:changed`および`pnpm test:changed`。
2. パッケージ成果物の完全性には、`pnpm release:check`。
3. インストール／更新／再起動／Plugin契約には、パッケージ受け入れの
   `package`プロファイル、またはリリースチェックのカスタムパッケージレーン。
4. OS固有のインストーラー、オンボーディング、およびプラットフォーム動作には、
   クロスOSリリースチェック。
5. 変更された対象がプロバイダーまたはホステッドサービスの動作に関係する場合にのみ、
   ライブスイート。

メンテナーのマシンでは、ローカル検証を明示的に行う場合を除き、広範なゲートと
Docker／パッケージの製品検証をTestboxで実行する必要があります。

## レガシー互換性

互換性に関する寛容措置は、範囲と期間が限定されています：

- `2026.4.25-beta.*`を含む`2026.4.25`までのパッケージでは、
  パッケージ受け入れにおいて、すでに出荷済みのパッケージメタデータの欠落を許容する場合があります。
- 公開済みの`2026.4.26`パッケージでは、すでに出荷されたローカルビルドの
  メタデータスタンプファイルについて警告する場合があります。
- それ以降のパッケージは、現行の契約を満たす必要があります。同じ欠落は、
  警告またはスキップではなく失敗となります。

これらの古い形式に対して、新しい起動時移行を追加しないでください。doctorの修復を追加または
拡張し、更新コマンドが再起動を所有する場合は、`upgrade-survivor`、`published-upgrade-survivor`、
または`update-restart-auth`を使用して検証します。

## カバレッジの追加

更新またはPluginの動作を変更する場合は、適切な理由で失敗し得る最下層に
カバレッジを追加します。

- 純粋なパスまたはメタデータのロジック: ソースの隣に単体テスト。
- パッケージインベントリまたはパック済みファイルの動作: `package-dist-inventory` またはtarball
  チェッカーテスト。
- CLIのインストール/更新動作: Dockerレーンのアサーションまたはフィクスチャ。
- 公開済みリリースの移行動作: `published-upgrade-survivor` シナリオ。
- 更新処理が担う再起動動作: `update-restart-auth`。
- レジストリ/パッケージソースの動作: `test:docker:plugins` フィクスチャまたはClawHub
  フィクスチャサーバー。
- 依存関係の配置またはクリーンアップ動作: ランタイム実行と
  ファイルシステム境界の両方をアサートします。npmの依存関係はPluginの
  管理対象npmプロジェクト内でホイストされる場合があるため、Pluginパッケージローカルの
  `node_modules` ツリーだけを想定せず、そのプロジェクトがスキャン/クリーンアップされることを
  テストで実証する必要があります。

新しいDockerフィクスチャはデフォルトで自己完結型にします。テストの目的が
実際のレジストリ動作でない限り、ローカルのフィクスチャレジストリと
偽のパッケージを使用します。

## 障害のトリアージ

まずアーティファクトの識別情報を確認します。

- Package Acceptanceの`resolve_package`サマリー: ソース、バージョン、SHA-256、
  アーティファクト名。
- Dockerアーティファクト: `.artifacts/docker-tests/**/summary.json`、
  `failures.json`、レーンログ、再実行コマンド。
- アップグレード生存確認のサマリー: `.artifacts/upgrade-survivor/summary.json`。
  ベースラインバージョン、候補バージョン、シナリオ、フェーズごとの所要時間、
  構成レシピのカバレッジを含みます。

リリース全体を再実行するよりも、同じパッケージアーティファクトを使用して、
失敗したレーンをそのまま正確に再実行することを優先します。
