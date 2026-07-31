---
read_when:
    - チャンネルの連絡先／グループ／自分自身の ID を検索したい場合
    - チャネルディレクトリアダプターを開発しています
summary: '`openclaw directory`（自分、ピア、グループ）の CLI リファレンス'
title: ディレクトリ
x-i18n:
    generated_at: "2026-07-26T09:35:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 33f1cabd0954f2e6e6affbfbff9f8e1f543bffebc54baff7c1ffaa21778744a0
    source_path: cli/directory.md
    workflow: 16
---

# `openclaw directory`

対応しているチャンネルのディレクトリ検索: 連絡先/ピア、グループ、および「自分」（本人）。

結果は、特に `openclaw message send --target ...` などの他のコマンドに貼り付けて使用することを想定しています。

## 共通フラグ

- `--channel <name>`: チャンネル ID/エイリアス（複数のチャンネルが設定されている場合は必須。1 つだけ設定されている場合は自動選択）
- `--account <id>`: アカウント ID（デフォルト: チャンネルのデフォルト）
- `--json`: JSON を出力

デフォルト（非 JSON）の出力は、`id`（場合によっては `name` も）をタブで区切った形式です。

## 注意事項

- 多くのチャンネルでは、結果はライブのプロバイダーディレクトリではなく、設定（許可リスト/設定済みグループ）に基づきます。
- WhatsApp のグループ一覧はライブです。Gateway の検索では、Gateway が所有する接続を再利用します。スタンドアロンコマンドは、そのアカウントを所有する他のプロセスがない場合にのみリンク済みセッションを開き、それ以外の場合はライブグループを利用できないことを報告します。
- インストール済みのチャンネル Plugin がディレクトリをサポートしていない場合があります。その場合、コマンドは未対応の操作を報告します。サポートを追加するために Plugin の再インストールやアップグレードを試行することはありません。

## `message send` での結果の使用

```bash
openclaw directory peers list --channel slack --query "U0"
openclaw message send --channel slack --target user:U012ABCDEF --message "hello"
```

## チャンネル別の ID 形式

| チャンネル                          | ターゲット ID の形式                                                                                                        |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| WhatsApp                            | `+15551234567`（DM）、`1234567890-1234567890@g.us`（グループ）、`120363123456789@newsletter`（チャンネル/ニュースレター、送信のみ） |
| Signal                              | 設定済みエイリアスは E.164/UUID の DM ターゲットまたは `group:<id>` のグループターゲットに解決されます                   |
| Telegram                            | `@username` または数値のチャット ID。グループには数値 ID を使用します                                                  |
| Slack                               | `user:U…` および `channel:C…`                                                                                |
| Discord                             | `user:<id>` および `channel:<id>`                                                                                |
| Matrix（Plugin）                    | `user:@user:server`、`room:!roomId:server`、または `#alias:server`                                                           |
| Microsoft Teams（Plugin）           | `user:<id>` および `conversation:<id>`                                                                                |
| Zalo（Plugin）                      | ユーザー ID（Bot API）                                                                                                      |
| Zalo Personal / `zalouser`（Plugin） | `zca`（`me`、`friend list`、`group list`）から取得したスレッド ID（DM/グループ）        |

## 自分（「me」）

```bash
openclaw directory self --channel zalouser
```

## ピア（連絡先/ユーザー）

```bash
openclaw directory peers list --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory peers list --channel zalouser --limit 50
```

## グループ

```bash
openclaw directory groups list --channel zalouser
openclaw directory groups list --channel zalouser --query "work"
openclaw directory groups members --channel zalouser --group-id <id>
```

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
