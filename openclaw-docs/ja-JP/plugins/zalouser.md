---
read_when:
    - OpenClaw で Zalo Personal（非公式）のサポートを利用したい場合
    - zalouser Pluginを設定または開発しています
summary: Zalo Personal Plugin：ネイティブ zca-js による QR ログインとメッセージング（Plugin のインストール + チャンネル設定 + ツール）
title: Zalo 個人用 Plugin
x-i18n:
    generated_at: "2026-07-26T10:27:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb0bdaa10340b5d78dc32abf6b0520fda6cf5f65e2e17b551b4e9bd72acfbbf2
    source_path: plugins/zalouser.md
    workflow: 16
---

Zalo Personal は、ネイティブの `zca-js` を使用して通常の Zalo ユーザーアカウントを
自動化する Plugin を介して OpenClaw をサポートします。外部の `zca`/`openzca` CLI バイナリは
必要ありません。

<Warning>
非公式の自動化により、アカウントが一時停止または禁止される可能性があります。自己責任で使用してください。
</Warning>

## 命名

チャンネル ID は `zalouser` です。これは、**個人用 Zalo
ユーザーアカウント**（非公式）を自動化することを明示するためです。別の `zalo` チャンネル ID は、公式の
バンドル済み Zalo Bot/Webhook 統合です。[Zalo](/ja-JP/channels/zalo) を参照してください。

## 実行場所

この Plugin は **Gateway プロセス内**で実行されます。リモート Gateway の場合は、
そのホストにインストールして設定し、Gateway を再起動してください。

## インストール

### npm から

```bash
openclaw plugins install @openclaw/zalouser
```

現在の公式リリースタグに追従するには、バージョンなしのパッケージを使用してください。再現可能な
インストールが必要な場合にのみ、正確なバージョンを固定してください。その後、Gateway を
再起動してください。

### ローカルフォルダーから（開発用）

```bash
PLUGIN_SRC=./path/to/local/zalouser-plugin
openclaw plugins install "$PLUGIN_SRC"
cd "$PLUGIN_SRC" && pnpm install
```

その後、Gateway を再起動してください。

## 設定

チャンネル設定は `plugins.entries.*` ではなく `channels.zalouser` に配置します。

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

DM/グループのアクセス制御、複数アカウントのセットアップ、環境変数、トラブルシューティングについては、
[Zalo Personal チャンネル設定](/ja-JP/channels/zalouser)を参照してください。

## CLI

```bash
openclaw channels login --channel zalouser
openclaw channels login --channel zalouser --account <name>
openclaw channels logout --channel zalouser
openclaw channels status --probe
openclaw message send --channel zalouser --target <threadId> --message "Hello from OpenClaw"
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "name"
openclaw directory groups members --channel zalouser --group-id <id>
```

## エージェントツール

ツール名: `zalouser`

アクション: `send`、`image`、`link`、`friends`、`groups`、`me`、`status`

チャンネルメッセージアクション（エージェントツールではありません）は、メッセージへの
リアクション用に `react` もサポートします。

## 関連項目

- [Zalo Personal チャンネル設定](/ja-JP/channels/zalouser)
- [Zalo（公式 Bot/Webhook チャンネル）](/ja-JP/channels/zalo)
- [Plugin の構築](/ja-JP/plugins/building-plugins)
- [ClawHub](/clawhub)
