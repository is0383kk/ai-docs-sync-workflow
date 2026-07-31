---
read_when:
    - Gateway Plugin または互換バンドルをインストールまたは管理する場合
    - シンプルなツール Plugin のひな形を作成または検証したい場合
    - Plugin の読み込み失敗をデバッグする場合
sidebarTitle: Plugins
summary: '`openclaw plugins` の CLI リファレンス（init、build、validate、list、install、marketplace、uninstall、enable/disable、doctor）'
title: Plugin
x-i18n:
    generated_at: "2026-07-26T10:08:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a1acba76fb1bc0ddae75e51fe573d3c2ac8f694607836e0c072ec7ca8fc0e262
    source_path: cli/plugins.md
    workflow: 16
---

Gateway Plugin、フックパック、互換バンドルを管理します。

<CardGroup cols={2}>
  <Card title="Plugin システム" href="/ja-JP/tools/plugin">
    Plugin のインストール、有効化、トラブルシューティングに関するエンドユーザー向けガイド。
  </Card>
  <Card title="Plugin の管理" href="/ja-JP/plugins/manage-plugins">
    インストール、一覧表示、更新、アンインストール、公開のクイック例。
  </Card>
  <Card title="Plugin バンドル" href="/ja-JP/plugins/bundles">
    バンドル互換性モデル。
  </Card>
  <Card title="Plugin マニフェスト" href="/ja-JP/plugins/manifest">
    マニフェストのフィールドと設定スキーマ。
  </Card>
  <Card title="セキュリティ" href="/ja-JP/gateway/security">
    Plugin インストールのセキュリティ強化。
  </Card>
</CardGroup>

## コマンド

```bash
openclaw plugins list [--enabled] [--verbose] [--json]
openclaw plugins search <query> [--limit <n>] [--json]
openclaw plugins install <path-or-spec> [--link] [--force] [--pin] [--marketplace <source>]
openclaw plugins inspect <id> [--runtime] [--json]
openclaw plugins inspect --all [--runtime] [--json]
openclaw plugins info <id>                    # inspect のエイリアス
openclaw plugins enable <id>
openclaw plugins disable <id>
openclaw plugins uninstall <id> [--dry-run] [--keep-files] [--force]
openclaw plugins update <id-or-npm-spec> | --all [--dry-run]
openclaw plugins registry [--refresh] [--json]
openclaw plugins doctor
openclaw plugins init <id> [--name <name>] [--type tool|provider] [--directory <path>]
openclaw plugins build [--entry <path>] [--check]
openclaw plugins validate [--entry <path>]
openclaw plugins marketplace entries [--offline] [--feed-profile <name>] [--json]
openclaw plugins marketplace list <source> [--json]
openclaw plugins marketplace refresh [--feed-profile <name>] [--expected-sha256 <sha256>] [--json]
```

時間のかかるインストール、検査、アンインストール、またはレジストリ更新を調査する場合は、
`OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1` を付けてコマンドを実行します。トレースは各フェーズの所要時間を
標準エラー出力に書き込み、JSON 出力を解析可能な状態に保ちます。[デバッグ](/ja-JP/help/debugging#plugin-lifecycle-trace)を参照してください。

<Note>
Nix モード（`OPENCLAW_NIX_MODE=1`）では、`openclaw.json` は変更できません。`install`、`update`、`uninstall`、`enable`、`disable` はいずれも実行を拒否します。代わりに、このインストールの Nix ソース（nix-openclaw の場合は `programs.openclaw.config` または `instances.<name>.config`）を編集してから、再ビルドしてください。エージェントを優先した[クイックスタート](https://github.com/openclaw/nix-openclaw#quick-start)を参照してください。
</Note>

<Note>
同梱 Plugin は OpenClaw とともに提供されます。一部（同梱モデルプロバイダー、同梱音声プロバイダー、同梱ブラウザー Plugin など）はデフォルトで有効ですが、その他は `plugins enable` が必要です。

ネイティブ OpenClaw Plugin には、インライン JSON Schema（空の場合も `configSchema`）を含む `openclaw.plugin.json` が付属します。互換バンドルは、代わりに独自のバンドルマニフェストを使用します。

`plugins list` は `Format: openclaw` または `Format: bundle` を表示します。詳細な一覧／情報出力には、バンドルのサブタイプ（`codex`、`claude`、または `cursor`）と、検出されたバンドル機能も表示されます。
</Note>

## 作成

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm run plugin:build
npm run plugin:validate
```

`plugins init` は、デフォルトで最小構成の TypeScript ツール Plugin を作成します。最初の
引数は Plugin ID で、`--name` は表示名を設定します。OpenClaw はこの
ID をデフォルトの出力ディレクトリとパッケージ名に使用します。ツールのスキャフォールドは
`defineToolPlugin` を使用し、`openclaw plugins build`/`validate` を呼び出す前にビルドする
`package.json` スクリプト `plugin:build` と `plugin:validate` を生成します。

`plugins build` はビルド済みエントリをインポートし、その静的ツールメタデータを読み取り、
`openclaw.plugin.json` を書き込み、`package.json` の `openclaw.extensions` との整合性を維持します。
`plugins validate` は、生成されたマニフェスト、パッケージメタデータ、現在のエントリエクスポートが
引き続き一致していることを確認します。完全な作成ワークフローについては、
[ツール Plugin](/ja-JP/plugins/tool-plugins)を参照してください。

スキャフォールドは TypeScript ソースを書き込みますが、ビルド済みの
`./dist/index.js` エントリからメタデータを生成するため、このワークフローは公開済み CLI でも機能します。
エントリがデフォルトのパッケージエントリではない場合は `--entry <path>` を使用します。生成されたメタデータが古い場合に、
ファイルを書き換えず CI を失敗させるには `plugins build --check` を使用します。

### プロバイダーのスキャフォールド

  ```bash
  openclaw plugins init acme-models --name "Acme Models" --type provider
  cd acme-models
  npm install
  npm run build
  npm test
  npm run validate
  ```

  プロバイダーのスキャフォールドは、API キー認証の基盤、`clawhub package validate` を実行する
  `npm run validate` スクリプト、ClawHub パッケージメタデータ、および将来 GitHub
  OIDC を介して信頼された公開を行うための手動実行 GitHub Actions ワークフローを備えた、汎用の OpenAI 互換モデルプロバイダー Plugin を作成します。プロバイダーのスキャフォールドは Skills を生成せず、
  `openclaw plugins build`/`validate` も使用しません。これらのコマンドは、ツールスキャフォールドの生成メタデータパス用です。

  公開する前に、プレースホルダーの API ベース URL、モデルカタログ、ドキュメントルート、認証情報のテキスト、および README の文面を実際のプロバイダーの詳細に置き換えてください。初回の ClawHub 公開と信頼された公開者の設定には、生成された README を使用してください。

  ## インストール

  ```bash
  openclaw plugins search "calendar"                      # ClawHub の Plugin を検索
  openclaw plugins install @openclaw/<package>            # 信頼された公式カタログ
  openclaw plugins install <package>                       # 任意の npm パッケージ
  openclaw plugins install clawhub:<package>                # ClawHub のみ
  openclaw plugins install npm:<package>                    # npm のみ
  openclaw plugins install npm-pack:<path.tgz>               # ローカルの npm-pack tarball
  openclaw plugins install git:github.com/<owner>/<repo>     # git リポジトリ
  openclaw plugins install git:github.com/<owner>/<repo>@<ref>
  openclaw plugins install <path>                            # ローカルパスまたはアーカイブ
  openclaw plugins install -l <path>                         # コピーの代わりにリンク
  openclaw plugins install <plugin>@<marketplace>             # マーケットプレイスの省略記法
  openclaw plugins install <plugin> --marketplace <name>      # マーケットプレイス（明示指定）
  openclaw plugins install <package> --force                  # ソースを確認／既存を上書き
  openclaw plugins install <package> --pin                    # 解決された npm バージョンを固定
  openclaw plugins install clawhub:<package> --acknowledge-clawhub-risk
  openclaw plugins install <package> --dangerously-force-unsafe-install
  ```

  セットアップ時のインストールをテストするメンテナーは、保護された環境変数を使用して Plugin の自動インストールソースを上書きできます。[Plugin インストールの上書き](/ja-JP/plugins/install-overrides)を参照してください。

  <Warning>
  起動移行期間中、パッケージ名のみを指定するとデフォルトで npm からインストールされます。ただし、同梱または公式の Plugin ID と一致する場合、OpenClaw は npm レジストリにアクセスせず、代わりにそのローカル／公式コピーを使用します。外部の npm パッケージを意図的に使用する場合は、`npm:<package>` を使用してください。ClawHub には `clawhub:<package>` を使用してください。Plugin のインストールはコードの実行と同様に扱い、固定バージョンを優先してください。
  </Warning>

  <Warning>
  ClawHub パッケージと OpenClaw の同梱／公式カタログは、信頼されたインストールソースです。新しい任意の npm、`npm-pack:`、git、ローカルパス／アーカイブ、またはマーケットプレイスのソースでは、続行前に警告と確認が表示されます。非対話型で任意のソースからインストールする場合は、そのソースを確認して信頼した後に `--force` を渡す必要があります。同じフラグは、必要に応じて既存のインストール先も上書きします。すでに追跡されているインストールの通常の更新には必要ありません。この確認は、危険性のある ClawHub リリースの信頼に関する警告にのみ適用される `--acknowledge-clawhub-risk` とは別です。`--force` は `security.installPolicy` や残りのインストール安全性チェックを回避しません。
  </Warning>

  `plugins search` は、インストール可能な `code-plugin` および
  `bundle-plugin` パッケージについて ClawHub に問い合わせます（Skills ではありません。Skills には `openclaw skills search` を使用してください）。
  デフォルトの `--limit` は 20 で、上限は 100 です。リモートカタログの読み取りのみを行い、ローカル状態の検査、設定の変更、パッケージのインストール、Plugin ランタイムの読み込みは行いません。結果には、ClawHub パッケージ名、ファミリー、チャンネル、バージョン、概要、および `openclaw plugins install clawhub:<package>` のようなインストールヒントが含まれます。

  <Note>
  ClawHub は、ほとんどの Plugin にとって主要な配布および検索手段です。npm は、サポート対象のフォールバックおよび直接インストール経路として引き続き利用できます。OpenClaw が所有する
  `@openclaw/*` Plugin パッケージは npm で再び公開されています。現在の一覧は [npmjs.com/org/openclaw](https://www.npmjs.com/org/openclaw) または
  [Plugin 一覧](/ja-JP/plugins/plugin-inventory)を参照してください。安定版のインストールでは `latest` を使用します。
  ベータチャンネルのインストールと更新では、利用可能な場合は npm の `beta` dist-tag を優先し、利用できない場合は `latest` にフォールバックします。拡張安定版チャンネルでは、パッケージ名のみ／デフォルトまたは `latest` を指定した公式 npm Plugin は、インストール済みコアの正確なバージョンに解決されます。正確な固定バージョン、明示的な `latest` 以外のタグ、サードパーティパッケージ、および npm 以外のソースは書き換えられません。
  </Note>

  <AccordionGroup>
  <Accordion title="設定のインクルードと無効な設定の修復">
    `plugins` セクションが単一ファイルの `$include` に基づいている場合、`plugins install/update/enable/disable/uninstall` はそのインクルードされたファイルに書き込み、`openclaw.json` は変更しません。ルートのインクルード、インクルード配列、および兄弟オーバーライドを含むインクルードは、平坦化する代わりに安全側で失敗します。サポートされている形式については、[設定のインクルード](/ja-JP/gateway/configuration)を参照してください。

    インストール前に設定が無効な場合、`plugins install` は通常、安全側で失敗し、先に `openclaw doctor --fix` を実行するよう通知します。Gateway の起動時およびホットリロード時には、無効な Plugin 設定は他の無効な設定と同様に安全側で失敗します。`openclaw doctor --fix` は、無効な Plugin エントリを隔離できます。既存設定に対する唯一の例外は、`openclaw.install.allowInvalidConfigRecovery` を明示的に有効化している Plugin 向けの限定的な同梱 Plugin 復旧経路です。

    既存のホスト設定は有効でも、新しくインストールした Plugin 自身の設定が存在しない場合、OpenClaw は無効な有効化済みエントリを書き込む代わりに、そのインストールを無効として記録します。`plugins.entries.<id>.config` を設定してから、`openclaw plugins enable <id>` を実行してください。既存の Plugin 設定エントリが存在するものの無効な場合、インストールはそれを書き換えずに失敗します。

  </Accordion>
  <Accordion title="--force による確認と再インストール／更新の違い">
    `--force` は、確認プロンプトを表示せずに ClawHub 以外のソースを承認します。`security.installPolicy` や残りのインストール安全性チェックは回避しません。Plugin またはフックパックがすでにインストールされている場合は、既存のインストール先を再利用し、その場で上書きします。任意の npm、ローカル、アーカイブ、git、マーケットプレイスのソースを確認した後、または同じ ID を意図的に再インストールする場合に使用してください。すでに追跡されている npm Plugin の通常のアップグレードには、`openclaw plugins update <id-or-npm-spec>` を優先してください。

    すでにインストールされている Plugin ID に対して `plugins install` を実行すると、OpenClaw は処理を停止し、通常のアップグレードには `plugins update <id-or-npm-spec>`、別のソースから現在のインストールを本当に上書きする場合には `plugins install <package> --force` を使用するよう案内します。任意のソースでは引き続き対話型の出所警告が表示されます。非対話型インストールでは、確認後に `--force` を渡す必要があります。信頼された ClawHub および OpenClaw カタログのソースには必要ありません。`--link` とともに使用した場合、`--force` はソースを承認しますが、リンクパスのインストールモードは変更しません。

  </Accordion>
  <Accordion title="--pin の適用範囲">
    `--pin` は npm インストールにのみ適用され、解決された正確な `<name>@<version>` を記録します。`git:` インストールではサポートされません（代わりに spec 内で ref を固定してください。例: `git:github.com/acme/plugin@v1.2.3`）。また、`--marketplace` でもサポートされません（マーケットプレイスからのインストールでは、npm spec の代わりにマーケットプレイスのソースメタデータが保持されます）。
  </Accordion>
  <Accordion title="--dangerously-force-unsafe-install">
    `--dangerously-force-unsafe-install` は非推奨となり、現在は何も行いません。OpenClaw は、Plugin のインストール時に組み込みの危険なコードのブロックを実行しなくなりました。

    ホスト固有のインストールポリシーが必要な場合は、オペレーターが管理する `security.installPolicy` サーフェスを使用してください。Plugin の `before_install` フックは Plugin ランタイムのライフサイクルフックであり、CLI インストールの主要なポリシー境界ではありません。

    ClawHub で公開した Plugin がレジストリスキャンによって非表示またはブロックされた場合は、[ClawHub での公開](/ja-JP/clawhub/publishing)に記載された公開者向けの手順を使用してください。`--dangerously-force-unsafe-install` は、ClawHub に Plugin の再スキャンや、ブロックされたリリースの公開を要求するものではありません。

  </Accordion>
  <Accordion title="--acknowledge-clawhub-risk">
    コミュニティの ClawHub からインストールする場合、ダウンロード前に選択したリリースの信頼記録が確認されます。ClawHub がそのリリースのダウンロードを無効にしている場合、悪意のあるスキャン結果を報告している場合、またはリリースをブロック対象のモデレーション状態（隔離、取り消し）にしている場合、OpenClaw はこのフラグにかかわらず無条件に拒否します。ブロック対象ではないもののリスクのあるスキャンステータスまたはモデレーション状態の場合、OpenClaw は信頼情報の詳細を表示し、続行前に確認を求めます。

    `--acknowledge-clawhub-risk` は、ClawHub の警告を確認し、対話型プロンプトなしで続行すると判断した場合にのみ使用してください。保留中または古い（まだクリーンではない）スキャン結果では警告が表示されますが、確認は必須ではありません。ClawHub の公式パッケージと OpenClaw にバンドルされた Plugin ソースでは、このリリース信頼チェックが完全に省略されます。

  </Accordion>
  <Accordion title="フックパックと npm spec">
    `plugins install` は、`package.json` で `openclaw.hooks` を公開するフックパックのインストールサーフェスでもあります。パッケージのインストールではなく、フィルタリングされたフックの表示とフック単位の有効化には `openclaw hooks` を使用してください。

    npm spec は**レジストリのみ**に対応します（パッケージ名に、任意で**正確なバージョン**または **dist-tag** を追加）。Git/URL/ファイル spec および semver 範囲は拒否されます。シェルにグローバルな npm インストール設定がある場合でも、安全のため、依存関係のインストールは Plugin ごとに `--ignore-scripts` を使用する単一の管理対象 npm プロジェクトで実行されます。管理対象の Plugin npm プロジェクトは、OpenClaw のパッケージレベルの npm `overrides` を継承するため、ホストのセキュリティ固定はホイストされた Plugin の依存関係にも適用されます。

    npm の解決を明示するには `npm:<package>` を使用してください。公式 Plugin ID と一致しない限り、ベアパッケージ spec もローンチ移行期間中は npm から直接インストールされます。

    バンドル済み Plugin と一致する生の `@openclaw/*` spec は、npm フォールバックの前に、イメージが所有するバンドル済みコピーへ解決されます。たとえば、`openclaw plugins install @openclaw/discord@2026.5.20 --pin` は管理対象の npm オーバーライドを作成せず、現在の OpenClaw ビルドにバンドルされた Discord Plugin を使用します。外部の npm パッケージを強制的に使用するには、`openclaw plugins install npm:@openclaw/discord@2026.5.20 --pin` を使用してください。

    ベア spec と `@latest` は安定版トラックに留まります。`2026.5.3-1` のような OpenClaw の日付付き修正版は、このチェックでは安定版として扱われます。npm がいずれかの形式をプレリリースへ解決した場合、OpenClaw は停止し、プレリリースタグ（`@beta`/`@rc`）または正確なプレリリースバージョン（`@1.2.3-beta.4`）を指定して明示的にオプトインするよう求めます。

    正確なバージョンを指定しない npm インストール（`npm:<package>` または `npm:<package>@latest`）では、OpenClaw はインストール前に解決されたパッケージメタデータを確認します。最新の安定版パッケージが、より新しい OpenClaw Plugin API またはホストの最小バージョンを必要とする場合、OpenClaw は以前の安定版を調べ、互換性のある最新リリースを代わりにインストールします。正確なバージョンと明示的な dist-tag は厳密に扱われます。互換性のない選択は失敗し、OpenClaw をアップグレードするか、互換性のあるバージョンを選択するよう求められます。

    ベアインストール spec が公式 Plugin ID（例: `diffs`）と一致する場合、OpenClaw はカタログエントリを直接インストールします。同じ名前の npm パッケージをインストールするには、明示的なスコープ付き spec（例: `@scope/diffs`）を使用してください。

  </Accordion>
  <Accordion title="Git リポジトリ">
    Git リポジトリから直接インストールするには `git:<repo>` を使用してください。サポートされる形式: `git:github.com/owner/repo`、`git:owner/repo`、完全な `https://`、`ssh://`、`git://`、`file://`、および `git@host:owner/repo.git` クローン URL。インストール前にブランチ、タグ、またはコミットをチェックアウトするには、`@<ref>` または `#<ref>` を追加します。

    Git インストールでは、一時ディレクトリにクローンし、指定された ref があればチェックアウトしてから、通常の Plugin ディレクトリインストーラーを使用します。そのため、マニフェスト検証、オペレーターのインストールポリシー、パッケージマネージャーによるインストール処理、インストール記録は npm インストールと同様に動作します。記録される Git インストールには、ソース URL/ref と解決済みコミットが含まれるため、`openclaw plugins update` は後でソースを再解決できます。

    Git からインストールした後は、`openclaw plugins inspect <id> --runtime --json` を使用して、Gateway メソッドや CLI コマンドなどのランタイム登録を確認してください。Plugin が `api.registerCli` を使用して CLI ルートを登録した場合は、そのコマンドを OpenClaw のルート CLI から直接実行します。例: `openclaw demo-plugin ping`。

  </Accordion>
  <Accordion title="アーカイブ">
    サポートされるアーカイブ: `.zip`、`.tgz`、`.tar.gz`、`.tar`。OpenClaw ネイティブの Plugin アーカイブには、展開された Plugin ルートに有効な `openclaw.plugin.json` が含まれている必要があります。`package.json` しか含まれていないアーカイブは、OpenClaw がインストール記録を書き込む前に拒否されます。

    ファイルが npm-pack tarball であり、レジストリインストールと同じ Plugin 単位の管理対象 npm プロジェクトパスを使用する場合は、`npm-pack:<path.tgz>` を使用してください。これには、`package-lock.json` の検証、ホイストされた依存関係のスキャン、npm インストール記録が含まれます。通常のアーカイブパスは、引き続き Plugin 拡張機能ルートの配下にローカルアーカイブとしてインストールされます。

    Claude マーケットプレイスからのインストールにも対応しています。

  </Accordion>
</AccordionGroup>

ClawHub からのインストールでは、明示的な `clawhub:<package>` ロケーターを使用します。

```bash
openclaw plugins install clawhub:openclaw-codex-app-server
openclaw plugins install clawhub:openclaw-codex-app-server@1.2.3
```

npm で安全に使用できるベア Plugin spec は、公式 Plugin ID と一致しない限り、ローンチ移行期間中はデフォルトで npm からインストールされます。

```bash
openclaw plugins install openclaw-codex-app-server
```

npm のみでの解決を明示するには `npm:` を使用してください。

```bash
openclaw plugins install npm:openclaw-codex-app-server
openclaw plugins install npm:@openclaw/discord@2026.5.20
openclaw plugins install npm:@scope/plugin-name@1.0.1
```

OpenClaw はインストール前に、公開されている Plugin API / Gateway の最小互換性を確認します。選択した ClawHub バージョンで ClawPack アーティファクトが公開されている場合、OpenClaw はバージョン付きの npm-pack `.tgz` をダウンロードし、ClawHub のダイジェストヘッダーとアーティファクトのダイジェストを検証してから、通常のアーカイブパスを通じてインストールします。ClawPack メタデータのない古い ClawHub バージョンは、引き続き従来のパッケージアーカイブ検証パスを通じてインストールされます。記録されるインストールには、後の更新に備えて、ClawHub のソースメタデータ、アーティファクトの種類、npm integrity、npm shasum、tarball 名、および ClawPack ダイジェスト情報が保持されます。
バージョンを指定しない ClawHub インストールでは、バージョン未指定の spec が記録されるため、`openclaw plugins update` はより新しい ClawHub リリースに追随できます。`clawhub:pkg@1.2.3` や `clawhub:pkg@beta` のような明示的なバージョンまたはタグセレクターは、そのセレクターに固定されたままになります。

### マーケットプレイスの省略記法

マーケットプレイス名が `~/.claude/plugins/known_marketplaces.json` にある Claude のローカルレジストリキャッシュに存在する場合は、`plugin@marketplace` の省略記法を使用してください。

```bash
openclaw plugins marketplace list <marketplace-name>
openclaw plugins install <plugin-name>@<marketplace-name>
```

マーケットプレイスのソースを明示的に渡すには `--marketplace` を使用してください。

```bash
openclaw plugins install <plugin-name> --marketplace <marketplace-name>
openclaw plugins install <plugin-name> --marketplace <owner/repo>
openclaw plugins install <plugin-name> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <plugin-name> --marketplace ./my-marketplace
```

<Tabs>
  <Tab title="マーケットプレイスのソース">
    - `~/.claude/plugins/known_marketplaces.json` にある Claude の既知のマーケットプレイス名
    - ローカルのマーケットプレイスルートまたは `marketplace.json` パス
    - `owner/repo` のような GitHub リポジトリの省略記法
    - `https://github.com/owner/repo` のような GitHub リポジトリ URL
    - Git URL

  </Tab>
  <Tab title="リモートマーケットプレイスのルール">
    GitHub または Git から読み込まれるリモートマーケットプレイスでは、Plugin エントリはクローンされたマーケットプレイスリポジトリ内に留まる必要があります。OpenClaw はそのリポジトリからの相対パスソースを受け入れ、リモートマニフェストに含まれる HTTP(S)、絶対パス、Git、GitHub、およびその他のパス以外の Plugin ソースを拒否します。
  </Tab>
</Tabs>

ローカルパスとアーカイブについて、OpenClaw は次を自動検出します。

- OpenClaw ネイティブ Plugin（`openclaw.plugin.json`）
- Codex 互換バンドル（`.codex-plugin/plugin.json`）
- Claude 互換バンドル（`.claude-plugin/plugin.json`、またはそのマニフェストファイルがない場合はデフォルトの Claude コンポーネントレイアウト）
- Cursor 互換バンドル（`.cursor-plugin/plugin.json`）

管理対象のローカルインストールは、Plugin ディレクトリまたはアーカイブでなければなりません。単独の `.js`、
`.mjs`、`.cjs`、および `.ts` Plugin ファイルは、`plugins install` によって管理対象の Plugin
ルートへコピーされません。また、`~/.openclaw/extensions` または `<workspace>/.openclaw/extensions` に直接配置しても読み込まれません。これらの
自動検出ルートは Plugin パッケージまたはバンドルのディレクトリを読み込み、トップレベルのスクリプトファイルはローカルヘルパーとしてスキップします。代わりに、単独ファイルを
`plugins.load.paths` に明示的に指定してください。

<Note>
互換バンドルは通常の Plugin ルートにインストールされ、同じ一覧表示/情報表示/有効化/無効化のフローに参加します。現在、バンドルの Skills、Claude のコマンド Skills、Claude の `settings.json` デフォルト、Claude の `.lsp.json` / マニフェストで宣言された `lspServers` デフォルト、Cursor のコマンド Skills、および互換性のある Codex フックディレクトリがサポートされています。検出されたその他のバンドル機能は診断/情報に表示されますが、ランタイム実行にはまだ接続されていません。
</Note>

ローカル Plugin ディレクトリをコピーせずに参照するには、`-l`/`--link` を使用してください（
`plugins.load.paths` に追加されます）。

```bash
openclaw plugins install -l ./my-plugin
```

`--link` は `--marketplace` または `git:` インストールではサポートされず、
すでに存在するローカルパスが必要です。非対話型のローカルリンクでは、
ソースを確認した後に `--force` を渡してください。これは出所を確認しますが、リンク先ディレクトリを
コピーまたは上書きしません。

<Note>
ワークスペースの拡張機能ルートから検出されたワークスペース由来の Plugin は、
明示的に有効化されるまでインポートも実行もされません。ローカル開発では、
`openclaw plugins enable <plugin-id>` を実行するか、
`plugins.entries.<plugin-id>.enabled: true` を設定してください。設定で
`plugins.allow` を使用している場合は、同じ Plugin ID もそこに含めてください。このフェイルクローズルールは、
チャンネル設定がセットアップ専用の読み込み対象としてワークスペース由来の Plugin を明示的に指定する場合にも
適用されます。そのため、ワークスペース Plugin が無効なまま、または許可リストから除外されたままでは、
ローカルチャンネル Plugin のセットアップコードは実行されません。リンクされたインストールと
明示的な `plugins.load.paths` エントリには、解決された Plugin の出所に対する通常のポリシーが適用されます。詳細は、
[Plugin ポリシーの設定](/ja-JP/tools/plugin#configure-plugin-policy)
および[設定リファレンス](/ja-JP/gateway/configuration-reference#plugins)を参照してください。

npm インストールで `--pin` を使用すると、デフォルトの固定なしの動作を維持しながら、解決された正確な spec（`name@version`）を管理対象 Plugin インデックスに保存できます。
</Note>

## 一覧

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
```

<ParamField path="--enabled" type="boolean">
  有効なプラグインのみを表示します。
</ParamField>
<ParamField path="--verbose" type="boolean">
  テーブル表示から、形式/ソース/オリジン/バージョン/有効化メタデータを含むプラグインごとの詳細行に切り替えます。
</ParamField>
<ParamField path="--json" type="boolean">
  機械可読なインベントリに加え、レジストリ診断とパッケージ依存関係のインストール状態を出力します。
</ParamField>

<Note>
`plugins list` は、永続化されたローカルプラグインレジストリを最初に読み取り、レジストリが存在しないか無効な場合は、マニフェストのみから導出したフォールバックを使用します。プラグインがインストール済みで有効になっており、コールドスタート計画から認識可能かどうかを確認するのに役立ちますが、すでに稼働している Gateway プロセスのライブランタイムプローブではありません。プラグインのコード、有効化状態、フックポリシー、または `plugins.load.paths` を変更した後は、新しい `register(api)` のコードやフックが実行されることを期待する前に、そのチャネルを提供する Gateway を再起動してください。リモート/コンテナデプロイでは、ラッパープロセスだけでなく、実際の `openclaw gateway run` 子プロセスを再起動していることを確認してください。

`plugins list --json` には、`package.json`
`dependencies` および `optionalDependencies` にある各プラグインの `dependencyStatus` が含まれます。OpenClaw は、それらのパッケージ名がプラグインの通常の Node `node_modules` 検索パス上に存在するかどうかを確認します。プラグインのランタイムコードをインポートしたり、パッケージマネージャーを実行したり、不足している依存関係を修復したりすることはありません。
</Note>

起動ログに `plugins.allow is empty; discovered non-bundled plugins may auto-load: ...` が記録された場合は、
`openclaw plugins list --enabled --verbose`、または一覧に表示されたプラグイン ID を指定して
`openclaw plugins inspect <id>` を実行し、プラグイン ID を確認して、信頼できる ID を `openclaw.json` の `plugins.allow` にコピーします。警告に検出されたすべてのプラグインを一覧表示できる場合は、それらの ID がすでに含まれた、そのまま貼り付け可能な
`plugins.allow` スニペットが出力されます。インストール元/ロードパスの来歴情報なしでプラグインがロードされる場合は、そのプラグイン ID を調査し、信頼できる ID を `plugins.allow` に固定するか、信頼できるソースからプラグインを再インストールして、OpenClaw にインストール元の来歴情報を記録させます。

パッケージ化された Docker イメージ内でバンドル済みプラグインを扱う場合は、プラグインのソースディレクトリを、
`/app/extensions/synology-chat` などの対応するパッケージ内ソースパスにバインドマウントします。OpenClaw は、そのマウントされたソースオーバーレイを
`/app/dist/extensions/synology-chat` より前に検出します。単にコピーしたソースディレクトリは非アクティブなままなので、通常のパッケージインストールでは引き続きコンパイル済みの dist が使用されます。

ランタイムフックをデバッグする場合:

- `openclaw plugins inspect <id> --runtime --json` は、モジュールをロードする検査パスから登録済みのフックと診断を表示します。ランタイム検査で依存関係がインストールされることはありません。従来の依存関係状態をクリーンアップするか、設定から参照されている不足中のダウンロード可能なプラグインを復旧するには、`openclaw doctor --fix` を使用します。
- `openclaw gateway status --deep --require-rpc` は、到達可能な Gateway の URL/プロファイル、サービス/プロセスのヒント、設定パス、RPC の正常性を確認します。
- バンドルされていない会話フック（`llm_input`、`llm_output`、`before_model_resolve`、`before_agent_reply`、`before_agent_run`、`before_agent_finalize`、`agent_end`）には `plugins.entries.<id>.hooks.allowConversationAccess=true` が必要です。

### プラグインインデックス

プラグインのインストールメタデータは機械管理される状態であり、ユーザー設定ではありません。インストールと更新では、アクティブな OpenClaw 状態ディレクトリ配下の共有 SQLite 状態データベースに書き込まれます。`installed_plugin_index` 行には、破損または欠落しているプラグインマニフェストのレコードを含む永続的な `installRecords` メタデータに加え、`openclaw plugins update`、アンインストール、診断、およびコールドプラグインレジストリで使用される、マニフェストから導出したコールドレジストリキャッシュが格納されます。

`plugins.installs` は廃止された手動設定サーフェスです。ランタイムおよび更新コマンドは、SQLite のインストール済みプラグインインデックスのみを読み取ります。通常のランタイム利用を開始する前に `openclaw doctor --fix` を実行し、従来の設定レコードをインデックスへインポートして、廃止されたキーを削除してください。

## アンインストール

```bash
openclaw plugins uninstall <id>
openclaw plugins uninstall <id> --dry-run
openclaw plugins uninstall <id> --keep-files
openclaw plugins uninstall <id> --force
```

`uninstall` は、`plugins.entries`、永続化されたプラグインインデックス、プラグインの許可/拒否リストのエントリ、および該当する場合はリンクされた `plugins.load.paths` エントリからプラグインレコードを削除します。`--keep-files` が設定されていない限り、アンインストール時には追跡対象の管理インストールディレクトリも削除されますが、削除されるのは、そのディレクトリが OpenClaw のプラグイン拡張機能ルート内に解決される場合に限られます。プラグインが現在 `memory` または `contextEngine` スロットを所有している場合、そのスロットはデフォルト（メモリでは `memory-core`、コンテキストエンジンでは `legacy`）にリセットされます。

`uninstall` は削除対象のプレビューを表示し、変更を加える前に `Uninstall plugin "<id>"?` と確認を求めます。確認プロンプトを省略するには `--force` を渡します（スクリプトや非対話実行に便利です）。これを指定しない場合、アンインストールには対話型 TTY が必要です。`--dry-run` は同じプレビューを表示し、確認を求めたり変更を加えたりせずに終了します。

<Note>
`--keep-config` は、`--keep-files` の非推奨エイリアスとしてサポートされています。
</Note>

## 更新

```bash
openclaw plugins update <id-or-npm-spec>
openclaw plugins update --all
openclaw plugins update <id-or-npm-spec> --dry-run
openclaw plugins update @openclaw/voice-call
openclaw plugins update @acme/demo
openclaw plugins update openclaw-codex-app-server --acknowledge-clawhub-risk
openclaw plugins update openclaw-codex-app-server --dangerously-force-unsafe-install
```

更新は、管理対象プラグインインデックスで追跡されているプラグインのインストールと、共有 SQLite 状態で追跡されているフックパックのインストールに適用されます。プラグインのインストール時にユーザーがすでに選択したソースを再利用するため、ソースについて再度確認する必要はありません。

<AccordionGroup>
  <Accordion title="プラグイン ID と npm spec の解決">
    プラグイン ID を渡すと、OpenClaw はそのプラグインについて記録されたインストール spec を再利用します。つまり、以前に保存された `@beta` などの dist-tag や、正確に固定されたバージョンが、以降の `update <id>` 実行でも引き続き使用されます。

    `update <id> --dry-run` の実行中、正確に固定された npm インストールは固定されたままです。OpenClaw がパッケージのレジストリデフォルトラインも解決でき、そのデフォルトラインがインストール済みの固定バージョンより新しい場合、ドライランでは固定状態を報告し、レジストリのデフォルトラインに追従するための明示的な `@latest` パッケージ更新コマンドを出力します。

    この対象指定更新ルールは、一括 `openclaw plugins update --all` メンテナンスパスとは異なります。一括更新でも通常の追跡対象インストール spec は引き続き尊重されますが、信頼できる公式 OpenClaw プラグインレコードは、古い正確な公式パッケージに留まる代わりに、現在の公式カタログターゲットへ同期できます。正確な、またはタグ付きの公式 spec を意図的に変更せず維持する場合は、対象を指定した `update <id>` を使用します。

    npm インストールでは、dist-tag または正確なバージョンを含む明示的な npm パッケージ spec を渡すこともできます。OpenClaw は、そのパッケージ名を追跡対象のプラグインレコードに対応付け、そのインストール済みプラグインを更新し、今後の ID ベース更新に使用する新しい npm spec を記録します。

    バージョンやタグを付けずに npm パッケージ名を渡した場合も、追跡対象のプラグインレコードに対応付けられます。プラグインが正確なバージョンに固定されており、レジストリのデフォルトリリースラインに戻したい場合に使用します。

  </Accordion>
  <Accordion title="ベータチャネルの更新">
    対象指定の `openclaw plugins update <id-or-npm-spec>` は、新しい spec を渡さない限り、追跡対象のプラグイン spec を再利用します。一括 `openclaw plugins update --all` は、信頼できる公式プラグインレコードを公式カタログターゲットに同期する際に、設定された `update.channel` を使用します。そのため、ベータチャネルのインストールは、暗黙的に stable/latest へ正規化されることなく、ベータリリースラインに留まることができます。

    `openclaw update` は、アクティブな OpenClaw 更新チャネルも認識します。ベータチャネルでは、デフォルトラインの npm および ClawHub プラグインレコードは、最初に `@beta` を試します。プラグインのベータリリースが存在しない場合は、記録された default/latest spec にフォールバックします。npm プラグインでは、ベータパッケージが存在してもインストール検証に失敗した場合にもフォールバックします。このフォールバックは警告として報告され、コアの更新を失敗させません。正確なバージョンおよび明示的なタグは、対象指定更新でそのセレクターに固定されたままです。

  </Accordion>
  <Accordion title="バージョンチェックと整合性のずれ">
    npm のライブ更新前に、OpenClaw はインストール済みパッケージのバージョンを npm レジストリのメタデータと照合します。インストール済みバージョンと記録されたアーティファクト識別情報が、解決されたターゲットとすでに一致している場合、ダウンロード、再インストール、または `openclaw.json` の書き換えを行わずに更新をスキップします。

    保存済みの整合性ハッシュが存在し、取得したアーティファクトのハッシュが変化している場合、OpenClaw はそれを npm アーティファクトのずれとして扱います。対話型の `openclaw plugins update` コマンドは、期待されるハッシュと実際のハッシュを表示し、続行前に確認を求めます。非対話型の更新ヘルパーは、呼び出し元が明示的な続行ポリシーを指定しない限り、安全側に失敗します。

  </Accordion>
  <Accordion title="更新時の --dangerously-force-unsafe-install">
    `--dangerously-force-unsafe-install` は互換性のため `plugins update` でも受け付けられますが、非推奨であり、プラグインの更新動作を変更しなくなりました。オペレーターの `security.installPolicy` は引き続き更新をブロックできます。プラグインの `before_install` フックは、プラグインフックがロードされているプロセスでのみ適用されます。
  </Accordion>
  <Accordion title="更新時の --acknowledge-clawhub-risk">
    コミュニティの ClawHub ベースのプラグイン更新では、代替パッケージをダウンロードする前に、インストール時と同じ正確なリリース信頼性チェックが実行されます。選択された ClawHub リリースに危険性を示す信頼警告がある場合でも続行すべき、レビュー済みの自動化では `--acknowledge-clawhub-risk` を使用します。公式 ClawHub パッケージおよびバンドル済み OpenClaw プラグインソースでは、このリリース信頼性プロンプトは省略されます。
  </Accordion>
</AccordionGroup>

## 検査

```bash
openclaw plugins inspect <id>
openclaw plugins inspect <id> --runtime
openclaw plugins inspect <id> --json
openclaw plugins inspect --all
```

デフォルトではプラグインランタイムをインポートせずに、識別情報、ロード状態、ソース、マニフェストの機能、ポリシーフラグ、診断、インストールメタデータ、バンドル機能、および検出された MCP または LSP サーバーのサポートを表示します。JSON 出力には `contracts.agentToolResultMiddleware` や `contracts.trustedToolPolicies` などのプラグインマニフェスト契約が含まれるため、オペレーターはプラグインを有効化または再起動する前に、信頼対象サーフェスの宣言を監査できます。`--runtime` を追加すると、プラグインモジュールがロードされ、登録済みのフック、ツール、コマンド、サービス、Gateway メソッド、および HTTP ルートが含まれます。ランタイム検査では、不足しているプラグイン依存関係が直接報告されます。インストールと修復は、引き続き `openclaw plugins install`、`openclaw plugins update`、および `openclaw doctor --fix` で行います。

プラグイン所有の CLI コマンドは通常、ルートの `openclaw` コマンドグループとしてインストールされますが、プラグインは `openclaw nodes` などのコアの親コマンド配下にネストされたコマンドを登録することもできます。`inspect --runtime` で `cliCommands` 配下にコマンドが表示されたら、表示されたパスで実行します。たとえば、`demo-git` を登録するプラグインは、`openclaw demo-git ping` で検証できます。

各プラグインは、ランタイムで実際に登録する内容に基づいて分類されます。

| 形態                | 意味                                                              |
| ------------------- | ----------------------------------------------------------------- |
| `plain-capability`  | 機能タイプが正確に 1 つ（例: プロバイダー専用プラグイン）         |
| `hybrid-capability` | 複数の機能タイプ（例: テキスト + 音声 + 画像）                    |
| `hook-only`         | フックのみで、機能、ツール、コマンド、サービス、ルートはなし      |
| `non-capability`    | ツール/コマンド/サービスはあるが、機能はなし                      |

機能モデルの詳細については、[プラグインの形態](/ja-JP/plugins/architecture#plugin-shapes)を参照してください。

<Note>
`--json` フラグは、スクリプト処理と監査に適した機械可読レポートを出力します。`inspect --all` は、形態、機能の種類、互換性に関する通知、バンドル機能、およびフックの概要を列に含む、フリート全体のテーブルを表示します。`info` は `inspect` のエイリアスです。
</Note>

## Doctor

```bash
openclaw plugins doctor
```

`doctor` は、Plugin の読み込みエラー、マニフェスト／検出の診断、互換性に関する通知、および欠落した Plugin スロットなどの古い Plugin 設定参照を報告します。インストールツリーと Plugin 設定に問題がなければ、`No plugin issues detected.` と出力されます。古い設定が残っているものの、インストールツリーがそれ以外は正常な場合、概要では Plugin が完全に正常であると示唆せず、その状態を明示します。

設定済みの Plugin がディスク上に存在していても、ローダーのパス安全性チェックによってブロックされている場合、設定検証は Plugin エントリを維持し、`present but blocked` として報告します。`plugins.entries.<id>` または `plugins.allow` の設定を削除するのではなく、パスの所有権や全ユーザー書き込み可能な権限など、先に表示されたブロック済み Plugin の診断内容を修正してください。

`register`/`activate` エクスポートの欠落など、モジュール形状のエラーについては、`OPENCLAW_PLUGIN_LOAD_DEBUG=1` を指定して再実行すると、診断出力にエクスポート形状の簡潔な概要が含まれます。

## レジストリ

```bash
openclaw plugins registry
openclaw plugins registry --refresh
openclaw plugins registry --json
```

ローカル Plugin レジストリは、インストール済み Plugin の識別情報、有効化状態、ソースメタデータ、およびコントリビューションの所有権に関する、OpenClaw の永続化されたコールドリードモデルです。通常の起動、プロバイダー所有者の検索、チャネル設定の分類、Plugin インベントリでは、Plugin のランタイムモジュールをインポートせずにこのレジストリを読み取れます。

`plugins registry` を使用して、永続化されたレジストリが存在するか、最新か、古くなっているかを確認します。`--refresh` を使用すると、永続化された Plugin インデックス、設定ポリシー、マニフェスト／パッケージメタデータからレジストリを再構築できます。これは修復手段であり、ランタイムを有効化する手段ではありません。

`openclaw doctor --fix` は、レジストリ周辺の管理対象 npm の不整合も修復します。管理対象 Plugin の npm プロジェクトまたは従来のフラットな管理対象 npm ルートにある、孤立または復元された `@openclaw/*` パッケージがバンドル済み Plugin を覆い隠している場合、doctor はその古いパッケージを削除し、起動時にバンドル済みマニフェストを基準として検証されるようにレジストリを再構築します。信頼できるインストールレコードが管理対象の世代を1つ選択していても、古いフラットディレクトリや世代ディレクトリが残っている場合、doctor は Gateway の再起動後に剪定できるよう、それらの古いツリーを廃止対象にします。また doctor は、`peerDependencies.openclaw` を宣言する管理対象 npm Plugin にホストの `openclaw` パッケージを再リンクし、更新または npm 修復後も `openclaw/plugin-sdk/*` などのパッケージローカルなランタイムインポートが解決されるようにします。

## マーケットプレイス

```bash
openclaw plugins marketplace entries
openclaw plugins marketplace entries --offline
openclaw plugins marketplace entries --json
openclaw plugins marketplace entries --feed-profile <name>
openclaw plugins marketplace entries --feed-url <url>
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json
openclaw plugins marketplace refresh
openclaw plugins marketplace refresh --feed-profile <name>
openclaw plugins marketplace refresh --feed-url <url>
openclaw plugins marketplace refresh --expected-sha256 <sha256> --json
```

`plugins marketplace entries` は、設定された OpenClaw マーケットプレイスフィードのエントリを一覧表示します。デフォルトではホストされたフィードの取得を試み、取得できない場合は、最後に受理されたスナップショットまたはバンドル済みデータへフォールバックします。特定の設定済みプロファイルを読み取るには `--feed-profile <name>`、明示的なホスト済みフィード URL を読み取るには `--feed-url <url>`、フィードを取得せずに最後に受理されたスナップショットを読み取るには `--offline` を使用します。

`plugins marketplace refresh` は、設定されたホスト済みフィードのスナップショットを更新し、OpenClaw がホスト済みデータ、ホスト済みスナップショット、バンドル済みフォールバックデータのいずれを受理したかを報告します。固定されたチェックサムに一致する新しいホスト済みペイロードが得られない限りコマンドを失敗させる必要がある場合は、`--expected-sha256` を使用します。

マーケットプレイスの `list` は、ローカルマーケットプレイスのパス、`marketplace.json` パス、`owner/repo` のような GitHub 短縮表記、GitHub リポジトリ URL、または git URL を受け付けます。`--json` は、解決されたソースラベルと、解析されたマーケットプレイスマニフェストおよび Plugin エントリを出力します。

マーケットプレイスの更新では、ホストされた OpenClaw マーケットプレイスフィードを読み込み、
検証済みのレスポンスをローカルのホスト済みフィードスナップショットとして永続化します。オプションを指定しない場合は、
設定されたデフォルトのフィードプロファイルを使用します。特定の設定済みプロファイルを
更新するには `--feed-profile <name>`、明示的なホスト済み
フィード URL を更新するには `--feed-url <url>`、一致するペイロードチェックサム
（`sha256:<hex>` または64文字の16進ダイジェストのみ）を必須にするには `--expected-sha256 <sha256>`、
機械可読出力には `--json` を使用します。明示的なホスト済みフィード URL に
認証情報、クエリ文字列、フラグメントを含めてはなりません。固定しない更新では、
コマンドを失敗させずにホスト済みスナップショットまたはバンドル済みフォールバックの結果を
報告できます。固定した更新は、新しいホスト済みペイロードを受理した場合にのみ成功し、
ホスト済みデータの更新に成功しても、OpenClaw が検証済みスナップショットを永続化できない場合は
失敗します。

組み込みの `clawhub-public` プロファイルでは、ペイロードの識別情報として
`clawhub-official` が想定されています。ClawHub が本番環境用の公開鍵を生成して
引き渡した後、OpenClaw はその鍵をバンドルします。それまでは、組み込みプロファイルに
署名付きフィードからインストールする権限はありません。公開鍵は、フィードホスト上の
鍵エンドポイントではなく、信頼できるリリースまたは運用者のチャネルから取得する必要があります。

OpenClaw は DSSE エンベロープを検証し、プロファイルで `feedId` が宣言されている場合は、
デコードされたペイロード ID がそれに一致することを要求します。組み込みの `clawhub-public`
プロファイルでは常に識別情報が宣言されるため、別のフィード用の有効なドキュメントが
そのプロファイルを介して再利用されることを防ぎます。

段階的な展開中は、`feedId` を省略している既存のカスタム署名付きプロファイルで、
ペイロード識別情報とのバインドなしの署名検証が維持されます。新しいカスタム
プロファイルでは `feedId` を宣言する必要があります。フィードプロファイルの設定項目は、
Control UI に必要な表示メタデータとともに別途導入されます。その
Doctor 診断では、識別情報が欠落している場合に運用者へ入力を求める必要があり、
フィード URL から推測してはなりません。この信頼バインドによって、廃止された
ルートの `marketplaces` キーが復元されることはありません。

## 関連項目

- [Plugin の構築](/ja-JP/plugins/building-plugins)
- [CLI リファレンス](/ja-JP/cli)
- [ClawHub](/clawhub)
