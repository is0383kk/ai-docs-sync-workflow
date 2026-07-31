---
read_when:
    - チャネルのトランスポートは接続済みと表示されるが、返信に失敗する
    - プロバイダーの詳細なドキュメントを確認する前に、チャネル固有のチェックが必要です
summary: チャンネル別の障害パターンと修正方法による迅速なチャンネルレベルのトラブルシューティング
title: チャンネルのトラブルシューティング
x-i18n:
    generated_at: "2026-07-26T10:06:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3891595e4b5aca9de7997a6e908fa1c9246579032bfdfa1656a6992d644c3ecc
    source_path: channels/troubleshooting.md
    workflow: 16
---

チャンネルは接続されているものの、動作が正しくない場合はこのページを使用してください。

## コマンドの実行順序

まず、次のコマンドを順番に実行します。

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

正常時の基準：

- `Runtime: running`
- `Connectivity probe: ok`
- `Capability: read-only`、`write-capable`、または `admin-capable`
- チャンネルプローブでトランスポートが接続済みと表示され、対応している場合は `works` または `audit ok` と表示される

## アップデート後

アップデート後に Telegram、iMessage、BlueBubbles 時代の設定、または別の Plugin チャンネルが消えた場合は、次を使用します。

```bash
openclaw status --all
openclaw doctor --fix
openclaw gateway restart
openclaw status --all
```

`openclaw
status --all` で `plugin load failed: dependency tree corrupted; run openclaw doctor --fix` を探します。これはチャンネルが設定されているものの、Plugin のセットアップまたは読み込みが、チャンネルを登録する代わりに破損した依存関係ツリーに遭遇したことを意味します。`openclaw doctor --fix` は古い Plugin ランタイム依存関係のシンボリックリンクと古い認証シャドウを削除し、その後 `openclaw gateway restart` がクリーンな状態を再読み込みします。

## WhatsApp

### WhatsApp の障害パターン

| 症状                             | 最速の確認方法                                       | 修正                                                                                                                              |
| ----------------------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 接続済みだが DM に返信しない         | `openclaw pairing list whatsapp`                    | 送信者を承認するか、DM ポリシー／許可リストを変更します。                                                                                    |
| グループメッセージが無視される              | 設定内の `requireMention` とメンションパターンを確認 | ボットをメンションするか、そのグループのメンションポリシーを緩和します。                                                                          |
| QR ログインが 408 でタイムアウトする         | Gateway の `HTTPS_PROXY`／`HTTP_PROXY` 環境変数を確認      | 到達可能なプロキシを設定します。`NO_PROXY` はバイパスする場合にのみ使用してください。                                                                         |
| ランダムに切断／再ログインを繰り返す     | `openclaw channels status --probe` とログ           | 現在接続されていても、最近の再接続はフラグが付けられます。ログを監視し、Gateway を再起動して、状態が不安定なままなら再リンクします。 |
| `status=408 Request Time-out` のループ  | プローブ、ログ、doctor、Gateway のステータスの順に確認            | まずホストの接続性／タイミングを修正します。ループが続く場合は認証情報をバックアップし、アカウントを再リンクします。                                   |
| 返信が数秒／数分遅れて届く | `openclaw doctor --fix`                             | doctor は、Gateway のイベントループを低下させていることが確認された古いローカル TUI クライアントを停止します。                                    |

詳細なトラブルシューティング：[WhatsApp のトラブルシューティング](/ja-JP/channels/whatsapp#troubleshooting)

## Telegram

### Telegram の障害パターン

| 症状                              | 最速の確認方法                                    | 修正                                                                                                                    |
| ------------------------------------ | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `/start` だが使用可能な返信フローがない    | `openclaw pairing list telegram`                 | ペアリングを承認するか、DM ポリシーを変更します。                                                                                   |
| ボットはオンラインだがグループでは応答しない    | メンション要件とボットのプライバシーモードを確認  | グループで表示できるようにプライバシーモードを無効にするか、ボットをメンションします。                                                              |
| ネットワークエラーで送信に失敗する    | Telegram API 呼び出しの失敗がないかログを確認      | `api.telegram.org` への DNS／IPv6／プロキシルーティングを修正します。                                                                      |
| 起動時に `getMe returned 401` が報告される | 設定されたトークンの取得元を確認                    | BotFather トークンを再コピーまたは再生成し、`botToken`、`tokenFile`、またはデフォルトアカウントの `TELEGRAM_BOT_TOKEN` を更新します。 |
| ポーリングが停止する、または再接続が遅い  | ポーリング診断の `openclaw logs --follow` | アップグレードします。停止が続く場合は通常、プロキシ／DNS／IPv6 に問題があります。                                                            |
| 起動時に `setMyCommands` が拒否される  | ログで `BOT_COMMANDS_TOO_MUCH` を確認         | Plugin／Skill／カスタムの Telegram コマンドを減らすか、ネイティブメニューを無効にします。                                                  |
| アップグレード後に許可リストでブロックされる    | `openclaw security audit` と設定の許可リスト  | `openclaw doctor --fix` を実行するか、`@username` を数値の送信者 ID に置き換えます。                                            |

詳細なトラブルシューティング：[Telegram のトラブルシューティング](/ja-JP/channels/telegram#troubleshooting)

## Discord

### Discord の障害パターン

| 症状                                   | 最速の確認方法                                                                                                                | 修正                                                                                                                                                                                                                                                                   |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ボットはオンラインだがギルドで返信しない           | `openclaw channels status --probe`                                                                                           | ギルド／チャンネルを許可し、メッセージコンテンツインテントを確認します。                                                                                                                                                                                                                |
| グループメッセージが無視される                    | メンションゲートで破棄されていないかログを確認                                                                                          | ボットをメンションするか、ギルド／チャンネルの `requireMention: false` を設定します。                                                                                                                                                                                                             |
| 入力中／トークン使用量は表示されるが Discord メッセージがない | モデルが `message(action=send)` を行わなかったアンビエントルームイベント、またはオプトイン済みの `message_tool` ルームかどうかを確認 | 抑制された最終ペイロードのメタデータを Gateway の詳細ログで調べ、`messages.groupChat.unmentionedInbound` を確認し、[アンビエントルームイベント](/ja-JP/channels/ambient-room-events)を参照するか、通常のグループリクエストでは `messages.groupChat.visibleReplies: "automatic"` を維持します。 |
| DM の返信がない                        | `openclaw pairing list discord`                                                                                              | DM のペアリングを承認するか、DM ポリシーを調整します。                                                                                                                                                                                                                               |

詳細なトラブルシューティング：[Discord のトラブルシューティング](/ja-JP/channels/discord#troubleshooting)

## Slack

### Slack の障害パターン

| 症状                                | 最速の確認方法                             | 修正                                                                                                                                                  |
| -------------------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| ソケットモードは接続済みだが応答がない | `openclaw channels status --probe`        | アプリトークン、ボットトークン、および必要なスコープを確認します。SecretRef ベースのセットアップでは `botTokenStatus`／`appTokenStatus = configured_unavailable` に注意します。 |
| DM がブロックされる                            | `openclaw pairing list slack`             | ペアリングを承認するか、DM ポリシーを緩和します。                                                                                                                  |
| チャンネルメッセージが無視される                | `groupPolicy` とチャンネル許可リストを確認 | チャンネルを許可するか、ポリシーを `open` に変更します。                                                                                                        |

詳細なトラブルシューティング：[Slack のトラブルシューティング](/ja-JP/channels/slack#troubleshooting)

## iMessage

### iMessage の障害パターン

| 症状                              | 最速の確認方法                                           | 修正                                                                   |
| ------------------------------------ | ------------------------------------------------------- | --------------------------------------------------------------------- |
| macOS 以外で `imsg` が見つからない、または失敗する | `openclaw channels status --probe --channel imessage`   | メッセージアプリがある Mac で OpenClaw を実行するか、`cliPath` に SSH ラッパーを使用します。 |
| macOS で送信はできるが受信できない     | メッセージの自動操作に関する macOS のプライバシー権限を確認 | TCC 権限を再付与し、チャンネルプロセスを再起動します。                 |
| DM の送信者がブロックされる                    | `openclaw pairing list imessage`                        | ペアリングを承認するか、許可リストを更新します。                                  |

詳細なトラブルシューティング：[iMessage のトラブルシューティング](/ja-JP/channels/imessage#troubleshooting)

## Signal

### Signal の障害パターン

| 症状                         | 最速の確認方法                              | 修正                                                      |
| ------------------------------- | ------------------------------------------ | -------------------------------------------------------- |
| デーモンに到達できるがボットが応答しない | `openclaw channels status --probe`         | `signal-cli` のデーモン URL／アカウントと受信モードを確認します。 |
| DM がブロックされる                      | `openclaw pairing list signal`             | 送信者を承認するか、DM ポリシーを調整します。                      |
| グループの返信がトリガーされない    | グループ許可リストとメンションパターンを確認 | 送信者／グループを追加するか、ゲート条件を緩和します。                       |

詳細なトラブルシューティング：[Signal のトラブルシューティング](/ja-JP/channels/signal#troubleshooting)

## QQ Bot

### QQ Bot の障害パターン

| 症状                         | 最速の確認方法                               | 修正                                                             |
| ------------------------------- | ------------------------------------------- | --------------------------------------------------------------- |
| ボットが「火星に行った」と返信する      | 設定内の `appId` と `clientSecret` を確認 | 認証情報を設定するか、Gateway を再起動します。                         |
| 受信メッセージがない             | `openclaw channels status --probe`          | QQ Open Platform で認証情報を確認します。                     |
| 音声が文字起こしされない           | STT プロバイダーの設定を確認                   | `channels.qqbot.stt` または `tools.media.audio` を設定します。          |
| 能動的なメッセージが届かない | QQ プラットフォームのインタラクション要件を確認  | 最近インタラクションがない場合、QQ がボットから開始されたメッセージをブロックすることがあります。 |

詳細なトラブルシューティング：[QQ Bot のトラブルシューティング](/ja-JP/channels/qqbot#troubleshooting)

## Matrix

### Matrix の障害パターン

| 症状                             | 最速の確認方法                          | 修正方法                                                                       |
| ----------------------------------- | -------------------------------------- | ------------------------------------------------------------------------- |
| ログイン済みだがルームメッセージを無視する | `openclaw channels status --probe`     | `groupPolicy`、ルームの許可リスト、メンション制御を確認します。                  |
| DM が処理されない                  | `openclaw pairing list matrix`         | 送信者を承認するか、DM ポリシーを調整します。                                       |
| 暗号化されたルームで失敗する                | `openclaw matrix verify status`        | デバイスを再検証してから、`openclaw matrix verify backup status` を確認します。  |
| バックアップの復元が保留中または破損している    | `openclaw matrix verify backup status` | `openclaw matrix verify backup restore` を実行するか、リカバリーキーを使用して再実行します。 |
| クロス署名またはブートストラップの状態が不正に見える | `openclaw matrix verify bootstrap`     | シークレットストレージ、クロス署名、バックアップの状態を一度に修復します。       |

セットアップと設定の詳細: [Matrix](/ja-JP/channels/matrix)

## 関連項目

- [ペアリング](/ja-JP/channels/pairing)
- [チャンネルルーティング](/ja-JP/channels/channel-routing)
- [Gateway のトラブルシューティング](/ja-JP/gateway/troubleshooting)
