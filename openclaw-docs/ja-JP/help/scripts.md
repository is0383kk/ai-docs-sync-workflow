---
read_when:
    - リポジトリからスクリプトを実行する
    - ./scripts 配下のスクリプトの追加または変更
summary: リポジトリスクリプト：目的、適用範囲、安全上の注意事項
title: スクリプト
x-i18n:
    generated_at: "2026-07-26T09:04:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 323069190ea6647101ee7120e06f6b2a018833d0904a11787fa1b610f5b3d9e1
    source_path: help/scripts.md
    workflow: 16
---

`scripts/` には、ローカルワークフローや運用タスク用の補助スクリプトが含まれています。タスクが明らかにスクリプトに関連している場合はこれらを使用し、それ以外の場合は CLI を優先してください。

## 規約

- ドキュメントまたはリリースチェックリストで参照されていない限り、スクリプトは**任意**です。
- CLI が提供されている場合は、CLI を優先してください（例: `openclaw models status --check`）。
- スクリプトはホスト固有であると想定し、新しいマシンで実行する前に内容を確認してください。

## 認証監視スクリプト

一般的なモデル認証については、[認証](/ja-JP/gateway/authentication)で説明しています。以下のスクリプトは、リモートまたはヘッドレスホスト上の **Claude Code CLI サブスクリプショントークン**を監視し、スマートフォンから再認証するための、独立した任意のシステムです。

- `scripts/setup-auth-system.sh` - 1 回限りのセットアップ: 現在の認証を確認し、長期間有効な `claude setup-token` の生成を支援して、systemd/Termux のインストール手順を表示します。
- `scripts/claude-auth-status.sh [full|json|simple]` - Claude Code と OpenClaw の認証状態を確認します。
- `scripts/auth-monitor.sh` - 状態をポーリングし、トークンの有効期限が近づくと（OpenClaw send や ntfy.sh を介して）通知を送信します。環境変数: `WARN_HOURS`（デフォルトは `2`）、`NOTIFY_PHONE`、`NOTIFY_NTFY`。同梱の `scripts/systemd/openclaw-auth-monitor.{service,timer}` を使用し、スケジュールに従って実行します（30 分ごと）。
- `scripts/mobile-reauth.sh` - `claude setup-token` を再実行し、スマートフォンで開く URL を表示します。Termux から SSH 経由で使用します。
- `scripts/termux-quick-auth.sh`、`scripts/termux-auth-widget.sh`、`scripts/termux-sync-widget.sh` - ホストに SSH 接続し、状態をトーストで表示して、認証の有効期限が切れたときに再認証コンソールまたは手順を開く Termux:Widget スクリプトです。

## GitHub 読み取りヘルパー

通常の `gh` では書き込み操作用の個人ログインを維持したまま、リポジトリスコープの読み取り呼び出しに GitHub App インストールトークンを `gh` で使用する場合は、`scripts/gh-read` を使用してください。

必須の環境変数:

- `OPENCLAW_GH_READ_APP_ID`
- `OPENCLAW_GH_READ_PRIVATE_KEY_FILE`

任意の環境変数:

- リポジトリに基づくインストール検索を省略する場合は `OPENCLAW_GH_READ_INSTALLATION_ID`
- 要求する読み取り権限のサブセットをカンマ区切りで上書きする場合は `OPENCLAW_GH_READ_PERMISSIONS`

リポジトリの解決順序:

- `gh ... -R owner/repo`
- `GH_REPO`
- `git remote origin`

例:

- `scripts/gh-read pr view 123`
- `scripts/gh-read run list -R openclaw/openclaw`
- `scripts/gh-read api repos/openclaw/openclaw/pulls/123`

## スクリプトを追加する場合

- スクリプトは用途を絞り、ドキュメントを整備してください。
- 関連するドキュメントに短い項目を追加してください（存在しない場合は作成してください）。

## 関連項目

- [テスト](/ja-JP/help/testing)
- [ライブ環境でのテスト](/ja-JP/help/testing-live)
