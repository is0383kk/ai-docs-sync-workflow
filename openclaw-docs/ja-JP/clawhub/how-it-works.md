---
read_when:
    - リスティング、バージョン、インストール、公開、モデレーションについて理解する
summary: ClawHub の掲載、バージョン、インストール、公開、スキャン、更新の仕組み。
x-i18n:
    generated_at: "2026-07-26T10:07:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 747079343899e42d00f84b00c553447abe0b83f2c4f1c9cdbf54725e34779eaf
    source_path: clawhub/how-it-works.md
    workflow: 16
---

# ClawHub の仕組み

ClawHub は OpenClaw の Skills と Plugin のためのレジストリレイヤーです。ユーザーにはパッケージを見つける場所を、公開者にはバージョンをリリースする場所を提供し、OpenClaw がそれらのパッケージを安全にインストールおよび更新するために必要なメタデータを提供します。

## レジストリレコード

各公開リストは、以下を含むレジストリレコードです。

- 所有者とスラッグまたはパッケージ名
- 公開済みの 1 つ以上のバージョン
- メタデータ、概要、ファイル、ソースの帰属情報
- 変更履歴、および `latest` などのタグ情報
- ダウンロード、インストール、お気に入り登録のシグナル
- セキュリティスキャンとモデレーションのステータス

リストページは、インストール前に Skills や Plugin が何を行うと説明しているかをユーザーが確認するための正式な場所です。

## Skills

Skills は、`SKILL.md` を中心とするバージョン管理されたテキストバンドルです。補助ファイル、例、テンプレート、スクリプトを含めることができます。

ClawHub は `SKILL.md` のフロントマターを読み取り、Skills の名前、説明、要件、環境変数、メタデータを把握します。正確なメタデータは、ユーザーが Skills をインストールするかどうかを判断するのに役立ち、自動スキャンが宣言された動作と観測された動作の不一致を検出するのにも役立つため、重要です。

[Skills の形式](/clawhub/skill-format)を参照してください。

## Plugin

Plugin はパッケージ化された OpenClaw 拡張機能です。ClawHub は、パッケージのメタデータ、互換性情報、ソースへのリンク、成果物、バージョンレコードを保存します。

OpenClaw が ClawHub から Plugin をインストールする際は、インストール前に提示された互換性メタデータを確認します。パッケージレコードには、API の互換性、Gateway の最低バージョン、対象ホスト、環境要件、成果物のダイジェストを含めることができます。

レジストリを信頼できる唯一の情報源とする場合は、ClawHub のインストール元を明示的に指定します。

```bash
openclaw plugins install clawhub:<package>
```

## 公開

公開すると、新しい変更不可のバージョンレコードが作成されます。公開者は、認証が必要なレジストリ操作に `clawhub` CLI を使用します。

```bash
clawhub skill publish ./my-skill
clawhub package publish <source> --family code-plugin --dry-run
clawhub package publish <source> --family code-plugin
```

アップロード前に、解決されたペイロードをプレビューするにはドライランを使用します。その後、公開ページに、公開されたメタデータ、ファイル、ソースの帰属情報、スキャンステータスが表示されます。

## インストールと更新

OpenClaw のインストールコマンドは、ClawHub をパッケージソースとして使用します。

```bash
openclaw skills install @openclaw/demo
openclaw plugins install clawhub:<package>
```

OpenClaw はインストール元のメタデータを記録するため、後から更新する際に同じレジストリパッケージを解決できます。ClawHub CLI は、完全な OpenClaw ワークスペースの外部でレジストリ管理の Skills フォルダーを使用したいユーザー向けに、Skills を直接インストールおよび更新するワークフローにも対応しています。

## セキュリティ状態

ClawHub では誰でも公開できますが、リリースには引き続きアップロードゲート、自動チェック、ユーザーからの報告、モデレーターによる措置が適用されます。

公開ページには、利用可能な場合にスキャンの概要が表示されます。保留、非表示、またはブロックされたコンテンツは、所有者が診断のために引き続き確認できる一方で、公開検索やインストールの流れから表示されなくなる場合があります。

[セキュリティ](/clawhub/security)、[セキュリティ監査](/clawhub/security-audits)、
[モデレーションとアカウントの安全性](/clawhub/moderation)、および
[利用規定](/clawhub/acceptable-usage)を参照してください。

## API アクセス

ClawHub は、検出、検索、パッケージの詳細、ダウンロードのための公開読み取り API を提供します。サードパーティのカタログは、正式な ClawHub リストへのリンクを設置し、レート制限を遵守し、推奨を受けているかのような表現を避ける場合に、これらの API を使用できます。

[公開 API](/clawhub/api)および [HTTP API](/clawhub/http-api)を参照してください。
