---
read_when:
    - エージェントに Control UI のペインを分割、フォーカス、閉じる、または移動させたい場合
    - エージェントにサイドバー、ターミナル、またはブラウザのパネルを表示または非表示にさせたい場合
    - ui.command ケイパビリティとファンアウト契約が必要です
sidebarTitle: Screen
summary: エージェントに接続済みの Control UI を構成させる
title: 画面
x-i18n:
    generated_at: "2026-07-26T09:55:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: df2215db96af29fa6b0db8abad79a0a2787a194dab6d00f9ef32f45521907ae1
    source_path: tools/screen.md
    workflow: 16
---

`screen` ツールを使用すると、エージェントはブラウザベースの Control UI を配置できます。これは型付きのレイアウトおよびナビゲーションサーフェスであり、スクリーンショットの取得やブラウザの自動操作を行うものではありません。

このツールは、接続元のクライアントが `ui-commands` ケイパビリティを通知している場合にのみ公開されます。ツールの実行時にも、対応する Control UI が少なくとも 1 つ接続されている必要があります。それ以外の場合、Gateway は `UNAVAILABLE` を返します。

## アクション

| アクション                            | 効果                                     | オプション入力                                |
| --------------------------------- | ------------------------------------------ | ---------------------------------------------- |
| `split_right`                     | 対象セッションペインを右方向に分割する | `sessionKey`（デフォルトは現在のセッション） |
| `split_down`                      | 対象セッションペインを下方向に分割する     | `sessionKey`（デフォルトは現在のセッション） |
| `close_pane`                      | 対象セッションペインを閉じる              | `sessionKey`（デフォルトは現在のセッション） |
| `focus`                           | 対象セッションペインにフォーカスする              | `sessionKey`（デフォルトは現在のセッション） |
| `navigate`                        | 対象セッションを開く                    | `sessionKey`（デフォルトは現在のセッション） |
| `sidebar_show` / `sidebar_hide`   | メインサイドバーを表示または非表示にする              | -                                              |
| `terminal_show` / `terminal_hide` | オペレーター用ターミナルパネルを表示または非表示にする   | 表示時は `dock`（`bottom` または `right`）      |
| `browser_show` / `browser_hide`   | ブラウザパネルを表示または非表示にする             | 表示時は `dock`（`bottom` または `right`）      |

コマンドが成功すると、Gateway が型付きの `ui.command` イベントをブロードキャストした後、`{ "ok": true }` が返されます。

## ルーティングとセキュリティ

プロトコル v1 では、意図的に `ui-commands` を通知する接続済みのすべての Control UI にコマンドを送信します。特定のブラウザタブを対象にすることはありません。これは、同じオペレーターが複数のダッシュボードを開いている場合に重要です。

Gateway RPC には `operator.write` が必要です。このツールで変更できるのは表示状態のみです。ピクセルの読み取り、スクリーンショットの取得、ページ上の任意のコンテンツのクリック、選択したセッションおよびオペレーターパネルの権限の回避はできません。

## 関連項目

- [Control UI](/ja-JP/web/control-ui)
- [Gateway プロトコル](/ja-JP/gateway/protocol#method-families)
- [ブラウザツール](/ja-JP/tools/browser)
