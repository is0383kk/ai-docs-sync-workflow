---
read_when:
    - ClawHub とは何かを説明する
    - Skills またはプラグインの検索、インストール、更新
    - レジストリへの Skills または plugins の公開
    - OpenClaw と ClawHub の CLI フローの選択
sidebarTitle: ClawHub
summary: 検出、インストール、公開、セキュリティ、および clawhub CLI に関する ClawHub の公開概要。
title: ClawHub
x-i18n:
    generated_at: "2026-07-26T08:55:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fde96ccb410b84dc4d3a48d42bbdbc0a80ac11dfb053afac2ee9e7e9d1605a5b
    source_path: clawhub/index.md
    workflow: 16
---

# ClawHub

ClawHub は、OpenClaw の Skills と plugins の公開レジストリです。

- ネイティブの `openclaw` コマンドを使用して、Skills の検索、インストール、更新、および ClawHub からの plugins のインストールを行います。
- レジストリの認証、公開、および削除／削除取り消しのワークフローには、別個の `clawhub` CLI を使用します。

サイト: [clawhub.ai](https://clawhub.ai)

## クイックスタート

OpenClaw で Skills を検索してインストールします。

```bash
openclaw skills search "calendar"
openclaw skills install @openclaw/demo
openclaw skills update --all
```

OpenClaw で plugins を検索してインストールします。

```bash
openclaw plugins search "calendar"
openclaw plugins install clawhub:<package>
openclaw plugins update --all
```

公開や削除／削除取り消しなど、レジストリ認証が必要なワークフローを使用する場合は、ClawHub CLI をインストールします。

```bash
npm i -g clawhub
# または
pnpm add -g clawhub
```

## ClawHub がホストするもの

| サーフェス     | 保存するもの                                                   | 代表的なコマンド                             |
| -------------- | -------------------------------------------------------------- | -------------------------------------------- |
| Skills         | `SKILL.md` と補助ファイルを含むバージョン管理されたテキストバンドル | `openclaw skills install @openclaw/demo`     |
| コード plugins | 互換性メタデータを含む OpenClaw plugin パッケージ              | `openclaw plugins install clawhub:<package>` |
| バンドル plugins | OpenClaw 配布用にパッケージ化された plugin バンドル           | `clawhub package publish <source>`           |

ClawHub は、semver バージョン、`latest` などのタグ、変更履歴、ファイル、ダウンロード数、スター数、セキュリティスキャンの概要を追跡します。公開ページには現在のレジストリ状態が表示されるため、ユーザーはインストール前に Skill や plugin を確認できます。

## ネイティブ OpenClaw フロー

ネイティブ OpenClaw コマンドは、アクティブな OpenClaw ワークスペースにインストールし、後続の更新コマンドが引き続き ClawHub を使用できるようにソースメタデータを永続化します。

plugin のインストールを ClawHub 経由で解決する必要がある場合は、`clawhub:<package>` を使用します。ベア形式の npm-safe plugin 指定は、リリース切り替え時に npm 経由で解決される場合があります。ソースを明示する必要がある場合、`npm:<package>` は引き続き npm 専用です。

plugin のインストールでは、アーカイブのインストールを実行する前に、提示された `pluginApi` および `minGatewayVersion` の互換性を検証します。パッケージバージョンで ClawPack アーティファクトが公開されている場合、OpenClaw はアップロードされた npm-pack の `.tgz` と完全に一致するものを優先し、ClawHub のダイジェストヘッダーとダウンロードされたバイト列を検証して、後続の更新用にアーティファクトのメタデータを記録します。

## ClawHub CLI

ClawHub CLI は、レジストリ認証が必要な作業に使用します。

```bash
clawhub login
clawhub whoami
clawhub search "postgres backups"
clawhub skill publish ./my-skill --slug my-skill --name "My Skill" --version 1.0.0
clawhub package explore --family code-plugin
clawhub package inspect episodic-claw
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

CLI には、レジストリを直接使用するワークフロー向けの Skill インストール／更新コマンドもあります。

```bash
clawhub install @openclaw/demo
clawhub update @openclaw/demo
clawhub update --all
clawhub list
```

これらのコマンドは、現在の作業ディレクトリにある `./skills` に Skills をインストールし、インストール済みバージョンを `.clawhub/lock.json` に記録します。

## 公開

`SKILL.md` を含むローカルフォルダーから Skills を公開します。

```bash
clawhub skill publish <path>
```

一般的な公開オプション:

- `--slug <slug>`: 公開される Skill の URL 名。
- `--name <name>`: 表示名。
- `--version <version>`: semver バージョン。
- `--changelog <text>`: 変更履歴のテキスト。
- `--tags <tags>`: カンマ区切りのタグ。デフォルトは `latest`。

ローカルフォルダー、`owner/repo`、`owner/repo@ref`、または GitHub URL から plugins を公開します。

```bash
clawhub package publish <source>
```

アップロードせずに正確な公開計画を作成するには `--dry-run` を使用し、CI 向けの出力には `--json` を使用します。

コード plugins では、`package.json` に必要な OpenClaw 互換性メタデータ（`openclaw.compat.pluginApi` および `openclaw.build.openclawVersion` を含む）を指定する必要があります。コマンドの完全なリファレンスについては [CLI](/ja-JP/clawhub/cli)、Skill メタデータについては [Skill 形式](/clawhub/skill-format)を参照してください。

## セキュリティとモデレーション

ClawHub はデフォルトでオープンです。誰でもアップロードできますが、公開にはアップロード要件を満たす作成後一定期間が経過した GitHub アカウントが必要です。公開詳細ページには、インストールまたはダウンロードの前に最新のスキャン状態の概要が表示されます。

ClawHub は、公開された Skills と plugin リリースに対して自動チェックを実行します。スキャンによって保留またはブロックされたリリースは、`/dashboard` では所有者に引き続き表示されますが、公開カタログやインストール画面からは表示されなくなる場合があります。

サインイン済みのユーザーは、Skills とパッケージを報告できます。モデレーターは、報告の確認、コンテンツの非表示または復元、不正利用を行うアカウントの禁止を実行できます。ポリシーと適用の詳細については、[セキュリティ](/ja-JP/clawhub/security)、[セキュリティ監査](/clawhub/security-audits)、[モデレーションとアカウントの安全性](/clawhub/moderation)、[許容される使用方法](/ja-JP/clawhub/acceptable-usage)を参照してください。

## テレメトリと環境

ログイン中に `clawhub install` を実行すると、ClawHub がインストール数の集計値を算出できるよう、CLI がベストエフォートでインストールイベントを送信する場合があります。無効にするには、次のように設定します。

```bash
export CLAWHUB_DISABLE_TELEMETRY=1
```

便利な環境オーバーライド:

| 変数                          | 効果                                              |
| ----------------------------- | ------------------------------------------------- |
| `CLAWHUB_SITE`                | ブラウザログインに使用するサイト URL を上書きします。 |
| `CLAWHUB_REGISTRY`            | レジストリ API の URL を上書きします。            |
| `CLAWHUB_CONFIG_PATH`         | CLI がトークン／設定状態を保存する場所を上書きします。 |
| `CLAWHUB_WORKDIR`             | デフォルトの作業ディレクトリを上書きします。      |
| `CLAWHUB_DISABLE_TELEMETRY=1` | インストールテレメトリを無効にします。            |

さらに詳しいリファレンスについては、[テレメトリ](/clawhub/telemetry)、[HTTP API](/clawhub/http-api)、[トラブルシューティング](/clawhub/troubleshooting)を参照してください。
