---
read_when:
    - ブラウザ自動化のためにサイトへログインする必要があります
    - X/Twitter に更新を投稿したい場合
summary: ブラウザ自動化と X/Twitter への投稿のための手動ログイン
title: ブラウザログイン
x-i18n:
    generated_at: "2026-07-26T09:20:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bccd363cf7c9611f4687d50a92f7fb3e2fd1c1d67bb27a80c892f7ac58ae1f8f
    source_path: tools/browser-login.md
    workflow: 16
---

## 手動ログイン（推奨）

サイトでログインが必要な場合は、ホストブラウザーの `openclaw`
プロファイルで手動でサインインしてください。モデルに認証情報を与えないでください。自動ログインは
ボット対策を作動させることが多く、アカウントがロックされる可能性があります。

X/Twitter やその他のボット検出が厳しいサイトでは、閲覧（検索やスレッド）と
投稿の両方にホストブラウザー（手動ログイン）を使用してください。サンドボックス化されたブラウザーセッションは、
ボット検出を作動させる可能性が高くなります。

ブラウザーのメインドキュメントに戻る：[ブラウザー](/ja-JP/tools/browser)。

## どの Chrome プロファイルが使用されますか？

OpenClaw は、普段使用するブラウザープロファイルとは別に、`openclaw` という名前の専用 Chrome プロファイル（オレンジ色の
UI）を制御します。

エージェントのブラウザーツール呼び出しでは：

- デフォルトの選択：エージェントは隔離された `openclaw` ブラウザーを使用します。
- 既存のログイン済みセッションが必要で、接続プロンプトをクリックまたは承認するために
  コンピューターの前にいる場合にのみ、`profile="user"` を使用してください。
- ユーザーブラウザーのプロファイルが複数ある場合は、推測せずにプロファイルを明示的に
  指定してください。

`openclaw` プロファイルにアクセスするには、次の 2 つの方法があります：

1. エージェントにブラウザーを開くよう依頼し、自分でログインします。
2. CLI から開きます：

```bash
openclaw browser start
openclaw browser open https://x.com
```

デフォルト以外のプロファイルでは、サブコマンドの前に
`--browser-profile <name>` を付けます（デフォルトは `openclaw`）：

```bash
openclaw browser --browser-profile <name> open https://x.com
```

## サンドボックス化：ホストブラウザーへのアクセスを許可する

エージェントがサンドボックス化されている場合、その `browser` ツール呼び出しは、デフォルトでホストではなく
サンドボックスブラウザーを使用します。代わりにエージェントがホストブラウザーを対象にできるようにするには：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        browser: {
          allowHostControl: true,
        },
      },
    },
  },
}
```

CLI の呼び出しは常にホストブラウザーを対象とし、サンドボックスを対象にすることはありません。そのため、この設定に関係なく
自分でホストブラウザーを開けます：

```bash
openclaw browser --browser-profile openclaw open https://x.com
```

`sandbox.browser.allowHostControl: true` を設定すると、エージェントの `browser`
ツール呼び出しもホストを対象にできます。または、更新を投稿する
エージェントのサンドボックス化を無効にしてください。

## 関連項目

- [ブラウザー](/ja-JP/tools/browser)
- [ブラウザーの Linux トラブルシューティング](/ja-JP/tools/browser-linux-troubleshooting)
- [ブラウザーの WSL2 トラブルシューティング](/ja-JP/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
