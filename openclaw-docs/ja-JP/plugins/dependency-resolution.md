---
read_when:
    - Plugin パッケージのインストールをデバッグする
    - Plugin の起動、doctor、またはパッケージマネージャーのインストール動作を変更している場合
    - パッケージ版の OpenClaw インストールまたはバンドルされたプラグインマニフェストを保守している場合
sidebarTitle: Dependencies
summary: OpenClaw が Plugin パッケージをインストールし、Plugin の依存関係を解決する仕組み
title: Plugin の依存関係解決
x-i18n:
    generated_at: "2026-07-26T09:09:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ae24a82568e275399cb7b68729d2805956792852612f84d6918850305f0eb243
    source_path: plugins/dependency-resolution.md
    workflow: 16
---

OpenClaw は、Plugin の依存関係をインストール時または更新時にのみ処理します。ランタイムでの
読み込み時には、パッケージマネージャーの実行、依存関係ツリーの修復、
OpenClaw パッケージディレクトリの変更は一切行いません。

## 責任の分担

Plugin パッケージは、それぞれの依存関係グラフを管理します。

- ランタイム依存関係は、Plugin パッケージの `dependencies` または
  `optionalDependencies` に配置します。
- SDK/core のインポートは、peer dependency または OpenClaw が提供するインポートです。
- ローカル開発用 Plugin は、あらかじめインストール済みの依存関係を自身で用意します。
- npm および git の Plugin は、OpenClaw が管理するパッケージルートにインストールされます。

OpenClaw が管理するのは、Plugin のライフサイクルのみです。

- Plugin のソースを検出します。
- 明示的に要求された場合に、パッケージをインストールまたは更新します。
- インストールメタデータを記録します。
- Plugin のエントリポイントを読み込みます。
- 依存関係が欠落している場合は、対処方法が明確なエラーで失敗します。

## インストールルート

OpenClaw は、ソースごとに安定したルートを使用します。

- npm パッケージは、`~/.openclaw/npm/projects/<encoded-package>` 配下の Plugin ごとのプロジェクトに
  インストールされます。
- git パッケージは、`~/.openclaw/git` 配下にクローンされます。
- ローカル、パス、アーカイブからのインストールでは、依存関係を修復せずに
  コピーまたは参照されます。

npm のインストールは、Plugin ごとのプロジェクトルートで次のように実行されます。

```bash
cd ~/.openclaw/npm/projects/<encoded-package>
npm install --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts --no-audit --no-fund
```

`openclaw plugins install npm-pack:<path.tgz>` は、ローカルの npm-pack tarball にも同じ Plugin ごとの npm
プロジェクトルートを使用します。OpenClaw は tarball の npm
メタデータを読み取り、コピーされた `file:` 依存関係として管理対象プロジェクトに追加し、
前述の通常の npm インストールを実行してから、インストール済みの lockfile メタデータを
検証したうえで Plugin を信頼します。このパスは、ローカルの pack アーティファクトが、
シミュレートするレジストリアーティファクトと同様に動作すべきパッケージ受け入れ検証や
リリース候補の実証に使用します。

公開前に公式または外部の Plugin パッケージをテストする場合は、`npm-pack:` を使用します。
生のアーカイブまたはパスからのインストールはローカルデバッグに有用ですが、
インストール済みの npm または ClawHub パッケージと同じ依存関係パスを実証するものでは
ありません。`npm-pack:` は、管理対象パッケージのインストール形態を実証しますが、
それだけでは、その Plugin がカタログに紐づいた公式コンテンツであることの証明にはなりません。

動作が同梱 Plugin または信頼済み公式 Plugin のステータスに依存する場合は、
ローカルパッケージによる実証と、カタログに基づく公式インストール、または公式の信頼情報を
記録する公開済みパッケージのパスを組み合わせてください。特権ヘルパーへのアクセスと
信頼済み公式スコープの処理は、ローカル tarball のインストールから推測するのではなく、
その信頼済みインストールパスで検証する必要があります。

Plugin が実行時にインポートの欠落によって失敗する場合は、管理対象プロジェクトを手作業で
修復するのではなく、パッケージマニフェストを修正してください。ランタイムインポートは、
Plugin パッケージの `dependencies` または `optionalDependencies` に属します。
`devDependencies` は、管理対象のランタイムプロジェクトにはインストールされません。
`~/.openclaw/npm/projects/<encoded-package>` 内のローカルな `npm install` は、一時的な診断を進めるためには
利用できますが、次回のインストールまたは更新時にパッケージメタデータからプロジェクトが
再作成されるため、パッケージ受け入れ検証にはなりません。

npm は、推移的依存関係を Plugin パッケージの隣にある Plugin ごとのプロジェクトの
`node_modules` にホイストすることがあります。OpenClaw は、インストールを信頼する前に
管理対象プロジェクトのルートをスキャンし、アンインストール時にそのプロジェクトを削除するため、
ホイストされたランタイム依存関係もその Plugin のクリーンアップ境界内に維持されます。

公開された npm Plugin パッケージには、`npm-shrinkwrap.json` を含めることができます。npm は
インストール時にその公開可能な lockfile を使用し、OpenClaw の管理対象 npm プロジェクトルートは、
通常のインストールパスを通じてこれをサポートします。OpenClaw が管理する公開可能な
Plugin パッケージには、そのパッケージの公開済み依存関係グラフから生成された、
パッケージローカルの shrinkwrap を含める必要があります。

```bash
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check
```

ジェネレーターは Plugin の `devDependencies` を除去し、ワークスペースの override
ポリシーを適用して、`openclaw.release.publishToNpm: true` を持つ各 Plugin に
`extensions/<id>/npm-shrinkwrap.json` を書き込みます。サードパーティー製 Plugin パッケージにも
shrinkwrap を含めることができます。OpenClaw はコミュニティパッケージに shrinkwrap を
必須とはしていませんが、存在する場合は npm がそれを使用します。

ローカルパッケージをリリース候補の実証として扱う前に、インストール対象となる tarball を
調査してください。

```bash
npm pack --pack-destination /tmp
tar -xOf /tmp/<plugin-package>.tgz package/package.json
tar -tf /tmp/<plugin-package>.tgz | grep '^package/dist/'
```

依存関係を変更した場合は、dev dependency なしの本番インストールでランタイムパッケージを
解決できることも確認してください。

```bash
tmpdir=$(mktemp -d)
(
  cd "$tmpdir"
  npm init -y >/dev/null
  npm install --package-lock-only --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts /tmp/<plugin-package>.tgz
)
rm -rf "$tmpdir"
```

OpenClaw が管理する npm Plugin パッケージは、明示的な `bundledDependencies` を使用して
公開することもできます。npm の公開パスは、ランタイム依存関係の名前リストを上書きし、
公開されるマニフェストから開発専用のワークスペースメタデータを除去し、
パッケージローカルのランタイム依存関係に対してスクリプトなしの npm インストールを実行してから、
それらの依存関係ファイルを含む Plugin tarball を pack または公開します。
ネイティブ依存の大きいパッケージ（Codex、ACPX、Copilot、llama.cpp、
memory-lancedb、Tlon）は `openclaw.release.bundleRuntimeDependencies: false` を使用して
この処理を無効にします。これらも shrinkwrap は同梱しますが、すべてのプラットフォーム向け
バイナリを Plugin tarball に埋め込む代わりに、インストール時に npm がランタイム依存関係を
解決します。ルートの `openclaw` パッケージは、完全な依存関係ツリーを同梱しません。

`openclaw/plugin-sdk/*` をインポートする Plugin は、`openclaw` を peer dependency として
宣言します。OpenClaw は、ホストパッケージの古いコピーがその Plugin 内での npm の
peer 解決に影響する可能性があるため、npm がホストパッケージの別のレジストリコピーを
管理対象プロジェクトにインストールすることを許可しません。管理対象の npm インストールでは、
npm による peer の解決と実体化をスキップします。また、OpenClaw はインストールまたは更新後に、
ホスト peer を宣言するインストール済みパッケージについて、Plugin ローカルの
`node_modules/openclaw` リンクを再設定します。

git インストールでは、リポジトリをクローンまたは更新してから、次を実行します。

```bash
npm install --omit=dev --ignore-scripts --no-audit --no-fund
```

インストール済み Plugin は、そのパッケージディレクトリから読み込まれるため、
パッケージローカルおよび親の `node_modules` の解決は、通常の Node パッケージと
同じように機能します。

## ローカル Plugin

ローカル Plugin は、開発者が管理するディレクトリです。OpenClaw は、ローカル Plugin に対して
`npm install`、`pnpm install`、または依存関係の修復を一切実行しません。
ローカル Plugin に依存関係がある場合は、読み込む前にその Plugin 内でインストールしてください。

サードパーティー製のローカル TypeScript Plugin は、緊急用パスとして Jiti を通じて読み込まれます。
パッケージ化された JavaScript Plugin と同梱の内部 Plugin は、代わりにネイティブの
import/require を通じて読み込まれます。

## 起動と再読み込み

Gateway の起動時と設定の再読み込み時には、Plugin の依存関係を一切インストールしません。
Plugin のインストール記録を読み取り、エントリポイントを計算して読み込みます。

実行時に依存関係が欠落している場合、Plugin の読み込みは失敗し、運用者に明示的な修正方法を
示すエラーが表示されます。

```bash
openclaw plugins update <id>
openclaw plugins install <source>
openclaw doctor --fix
```

`doctor --fix` は、OpenClaw が生成した従来の依存関係状態をクリーンアップし、
設定から参照されているもののローカルのインストール記録に存在しない、ダウンロード可能な
Plugin を復旧できます。Doctor は、すでにインストール済みのローカル Plugin の依存関係を
修復しません。

## 同梱 Plugin

軽量またはコアに不可欠な同梱 Plugin は、OpenClaw の一部として提供されます。
大規模なランタイム依存関係ツリーを持たないようにするか、ClawHub/npm 上の
ダウンロード可能なパッケージへ移行する必要があります。

コアパッケージに同梱される Plugin、外部インストールされる Plugin、またはソース専用に
維持される Plugin の現在の生成済み一覧については、
[Plugin インベントリ](/ja-JP/plugins/plugin-inventory)を参照してください。

同梱 Plugin のマニフェストで、依存関係のステージングを要求してはなりません。
大規模またはオプションの Plugin 機能は、通常の Plugin としてパッケージ化し、
サードパーティー製 Plugin と同じ npm/git/ClawHub パスを通じてインストールする必要があります。

ソースチェックアウトでは、OpenClaw はリポジトリを pnpm モノレポとして扱います。
`pnpm install` の実行後、同梱 Plugin は `extensions/<id>` から読み込まれるため、
パッケージローカルのワークスペース依存関係を利用でき、編集内容も直接反映されます。
ソースチェックアウトでの開発は pnpm 専用です。リポジトリルートで単に
`npm install` を実行しても、同梱 Plugin の依存関係は準備されません。

| インストール形態                    | 同梱 Plugin の場所               | 依存関係の管理主体                                                     |
| -------------------------------- | ------------------------------------- | -------------------------------------------------------------------- |
| `npm install -g openclaw`        | パッケージ内のビルド済みランタイムツリー | OpenClaw パッケージと明示的な Plugin のインストール、更新、doctor フロー     |
| Git チェックアウトと `pnpm install` | `extensions/<id>` ワークスペースパッケージ  | 各 Plugin パッケージ固有の依存関係を含む pnpm ワークスペース |
| `openclaw plugins install ...`   | 管理対象の npm プロジェクト、git、ClawHub ルート  | Plugin のインストールおよび更新フロー                                       |

## 従来状態のクリーンアップ

旧バージョンの OpenClaw は、起動時または doctor による修復時に、同梱 Plugin の
依存関係ルートを生成していました。現在の doctor クリーンアップでは、
`--fix` を使用して、古い `plugin-runtime-deps` ルート、
削除済みの `plugin-runtime-deps` ターゲットを指すグローバル Node-prefix パッケージの
シンボリックリンク、`.openclaw-runtime-deps*` マニフェスト、生成された Plugin の
`node_modules`、インストールステージディレクトリ、パッケージローカルの pnpm ストアを含む、
古いディレクトリとシンボリックリンクを削除します。パッケージ化された postinstall でも、
従来のターゲットルートを削除する前にそれらのグローバルシンボリックリンクを削除するため、
アップグレード後に参照先のない ESM パッケージインポートが残ることはありません。

旧形式の npm インストールでは、共有の `~/.openclaw/npm/node_modules` ルートも使用していました。
現在のインストール、更新、アンインストール、および doctor の各フローは、
復旧とクリーンアップの目的に限り、この従来のフラットルートを引き続き認識します。
新しい npm インストールでは、代わりに Plugin ごとのプロジェクトルートが作成されます。
