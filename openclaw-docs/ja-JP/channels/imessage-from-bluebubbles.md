---
read_when:
    - BlueBubbles から同梱の iMessage Plugin への移行計画
    - BlueBubbles の設定キーを iMessage の同等項目に変換する
    - iMessage Pluginを有効にする前のimsgの検証
summary: 古い BlueBubbles 設定をバンドル版 iMessage Plugin に移行する方法：キーのマッピング、グループ許可リストのゲート、切り替えの検証。
title: BlueBubbles からの移行
x-i18n:
    generated_at: "2026-07-26T09:12:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5984ad1319b4bb3060496666bea6de663eba0105a89f82d13030c015c5df159d
    source_path: channels/imessage-from-bluebubbles.md
    workflow: 16
---

BlueBubbles のサポートは削除されました。OpenClaw は、バンドルされている `imessage` plugin を通じてのみ iMessage をサポートします。この plugin は [`steipete/imsg`](https://github.com/steipete/imsg) を JSON-RPC 経由で操作し、BlueBubbles が利用していたものと同じプライベート API サーフェス（`react`、`edit`、`unsend`、`reply`、`sendWithEffect`、ネイティブ投票、グループ管理、添付ファイル）にアクセスします。1 つの CLI バイナリが BlueBubbles サーバー、クライアントアプリ、Webhook 接続処理を置き換えます。REST エンドポイントも Webhook 認証もありません。

このガイドでは、古い `channels.bluebubbles` 設定を `channels.imessage` に移行します。これ以外にサポートされる移行パスはありません。現在の OpenClaw では、残存する `channels.bluebubbles` ブロックは機能しません。これを読み取るランタイムはありません。

<Note>
簡潔な告知と運用者向け概要については、[BlueBubbles の削除と imsg iMessage パス](/ja-JP/announcements/bluebubbles-imessage)を参照してください。
</Note>

## 移行チェックリスト

古い BlueBubbles 設定をすでに把握している場合の、最短かつ安全な手順は次のとおりです。

1. Messages.app を実行している Mac 上で `imsg` を直接確認します（`imsg chats`、`imsg history`、`imsg send`、`imsg rpc --help`）。
2. `channels.bluebubbles` から `channels.imessage` に動作設定キーをコピーします：`dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`、`groups`、`includeAttachments`、`attachmentRoots`、`mediaMaxMb`、`textChunkLimit`、および `actions`。
3. 存在しなくなったトランスポート設定キーを削除します：`serverUrl`、`password`、Webhook URL、および BlueBubbles サーバー設定。
4. Gateway が Messages Mac 上で動作していない場合は、`channels.imessage.cliPath` に SSH ラッパーを設定し、リモート添付ファイル取得用に `remoteHost` を設定します。
5. `channels.imessage` を有効にして Gateway を再起動し、`openclaw channels status --probe --channel imessage` を実行します。
6. DM を 1 件、許可されたグループを 1 件、添付ファイルが有効な場合は添付ファイル、さらにエージェントが使用する予定のすべてのプライベート API アクションをテストします。
7. iMessage パスを確認した後、BlueBubbles サーバーと古い `channels.bluebubbles` 設定を削除します。

## imsg の動作

`imsg` は Messages 用のローカル macOS CLI です。OpenClaw は `imsg rpc` を子プロセスとして起動し、stdin/stdout を介して JSON-RPC で通信します。HTTP サーバー、Webhook URL、バックグラウンドデーモン、起動エージェント、公開するポートはありません。

- 読み取りは、読み取り専用 SQLite ハンドルを使用して `~/Library/Messages/chat.db` から行われます。
- リアルタイムの受信メッセージは `imsg watch` / `watch.subscribe` から取得されます。これは `chat.db` のファイルシステムイベントを追跡し、フォールバックとしてポーリングを使用します。
- 通常のテキストおよびファイル送信には Messages.app の自動操作を使用します。
- 高度なアクションでは、`imsg launch` を使用して `imsg` ヘルパーを Messages.app に注入します。これにより、開封確認、入力中インジケーター、リッチ送信、編集、送信取り消し、スレッド返信、Tapback、投票、グループ管理が利用可能になります。
- Linux ビルドでは、コピーされた `chat.db` を検査できますが、送信、Mac 上のライブデータベースの監視、Messages.app の操作はできません。OpenClaw iMessage を使用するには、サインイン済みの Mac 上で、またはその Mac への SSH ラッパーを介して `imsg` を実行します。

## 開始前の準備

1. Messages.app を実行している Mac に `imsg` をインストールします。

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg chats --limit 3
   ```

   通常のローカル設定では、OpenClaw のセットアップ時に、サインイン済みの Messages Mac 上の `imsg` をユーザーの確認を得て Homebrew でインストールまたは更新できます。手動設定および SSH ラッパートポロジーは引き続き運用者が管理します。`imsg` を実行するのと同じローカルまたはリモートのユーザーコンテキストで Homebrew の更新を繰り返してください。`imsg chats` が `unable to open database file`、空の出力、または `authorization denied` で失敗する場合は、`imsg` を起動するターミナル、エディター、Node プロセス、Gateway サービス、または SSH 親プロセスにフルディスクアクセスを許可してから、その親プロセスを再度開いてください。

2. OpenClaw の設定を変更する前に、読み取り、監視、送信、および RPC の各サーフェスを確認します。

   ```bash
   imsg chats --limit 10 --json | jq -s
   imsg history --chat-id 42 --limit 10 --attachments --json | jq -s
   imsg watch --chat-id 42 --reactions --json
   imsg send --chat-id 42 --text "OpenClaw imsg test"
   imsg rpc --help
   ```

   `42` を `imsg chats` で取得した実際のチャット ID に置き換えます。送信には Messages.app の Automation 権限が必要です。OpenClaw を SSH 経由で実行する場合は、OpenClaw が使用するものと同じ SSH ラッパーまたはユーザーコンテキストを通じて、これらのコマンドを実行してください。読み取りは成功するものの、送信が AppleEvents の `-1743` で失敗する場合は、Automation 権限が `/usr/libexec/sshd-keygen-wrapper` に付与されているか確認してください。[SSH ラッパーでの送信が AppleEvents -1743 で失敗する](/ja-JP/channels/imessage#requirements-and-permissions-macos)を参照してください。

3. プライベート API ブリッジを有効にします。返信、Tapback、エフェクト、投票、添付ファイルへの返信、グループアクションはこれに依存するため、OpenClaw iMessage では有効化を強く推奨します。

   ```bash
   imsg launch
   imsg status --json
   ```

   `imsg launch` を使用するには SIP を無効にする必要があります（最新の macOS ではライブラリ検証の緩和も必要です。[imsg プライベート API の有効化](/ja-JP/channels/imessage#enabling-the-imsg-private-api)を参照してください）。基本的な送信、履歴、監視は `imsg launch` なしでも動作しますが、OpenClaw iMessage のすべてのアクションサーフェスは利用できません。

4. `channels.imessage` を有効にして Gateway を起動した後、OpenClaw を通じてブリッジを確認します。

   ```bash
   openclaw channels status --probe
   ```

   iMessage アカウントは `works` と報告する必要があります。`--json` がある場合、プローブのペイロードには `privateApi.available: true` が含まれます。`false` と報告された場合は、まずそれを修正してください。[機能検出](/ja-JP/channels/imessage#private-api-actions)を参照してください。プローブには到達可能な Gateway が必要です（到達できない場合、CLI は設定情報のみの出力にフォールバックします）。また、設定済みかつ有効なアカウントのみがプローブされます。

5. 設定のスナップショットを作成します。

   ```bash
   cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
   ```

## 設定の変換

iMessage と BlueBubbles は、チャネルレベルの動作設定キーの大半を共有しています。変更されるのはトランスポート（REST サーバーとローカル CLI の違い）およびグループレジストリキーの形式です。

| BlueBubbles                                                | バンドル版 iMessage                          | 注記                                                                                                                                                                                                                                                                            |
| ---------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `channels.bluebubbles.enabled`                             | `channels.imessage.enabled`               | セマンティクスは同じです（ブロックが存在する場合のデフォルトは `true`）。                                                                                                                                                                                                                           |
| `channels.bluebubbles.serverUrl`                           | _（削除済み）_                               | REST サーバーはありません。Plugin が stdio 経由で `imsg rpc` を起動します。                                                                                                                                                                                                                        |
| `channels.bluebubbles.password`                            | _（削除済み）_                               | Webhook 認証は不要です。                                                                                                                                                                                                                                                |
| _（暗黙的）_                                               | `channels.imessage.cliPath`               | `imsg` へのパス（デフォルトは `imsg`）。SSH にはラッパースクリプトを使用します。                                                                                                                                                                                                                   |
| _（暗黙的）_                                               | `channels.imessage.dbPath`                | オプションの Messages.app `chat.db` オーバーライド。省略時は自動検出されます。                                                                                                                                                                                                            |
| _（暗黙的）_                                               | `channels.imessage.remoteHost`            | `host` または `user@host`。`cliPath` が SSH ラッパーであり、SCP による添付ファイル取得を行う場合にのみ必要です。                                                                                                                                                                        |
| `channels.bluebubbles.dmPolicy`                            | `channels.imessage.dmPolicy`              | 値は同じです（`pairing` / `allowlist` / `open` / `disabled`）。デフォルトは `pairing`。                                                                                                                                                                                                  |
| `channels.bluebubbles.allowFrom`                           | `channels.imessage.allowFrom`             | ハンドル形式は同じです（`+15555550123`、`user@example.com`）。ペアリングストアの承認は移行されません。以下を参照してください。                                                                                                                                                                   |
| `channels.bluebubbles.groupPolicy`                         | `channels.imessage.groupPolicy`           | 値は同じです（`allowlist` / `open` / `disabled`）。デフォルトは `allowlist`。                                                                                                                                                                                                            |
| `channels.bluebubbles.groupAllowFrom`                      | `channels.imessage.groupAllowFrom`        | 同じです。未設定の場合、iMessage は `allowFrom` にフォールバックします。明示的に空の `groupAllowFrom: []` を指定すると、`groupPolicy: "allowlist"` ではすべてのグループがブロックされます。                                                                                                                               |
| `channels.bluebubbles.groups`                              | `channels.imessage.groups`                | `"*"` ワイルドカードエントリをそのままコピーします。グループごとのエントリは、数値の iMessage `chat_id` をキーとして付け直します。「グループレジストリの落とし穴」を参照してください。`requireMention`、`tools`、`toolsBySender`、`systemPrompt` は引き継がれます。                                                                            |
| `channels.bluebubbles.sendReadReceipts`                    | `channels.imessage.sendReadReceipts`      | デフォルトは `true`。バンドル版 Plugin では、プライベート API のプローブが稼働している場合にのみ発火します。                                                                                                                                                                                        |
| `channels.bluebubbles.includeAttachments`                  | `channels.imessage.includeAttachments`    | 形式は同じで、デフォルトで無効なのも同じです。BlueBubbles で添付ファイルを処理していた場合は、これを明示的に設定してください。設定するまで、受信した写真やメディアは（`Inbound message` ログ行なしで）暗黙に破棄されます。                                                                                             |
| `channels.bluebubbles.attachmentRoots`                     | `channels.imessage.attachmentRoots`       | ローカルルートです。ワイルドカードのルールは同じです。                                                                                                                                                                                                                                                |
| _（該当なし）_                                                    | `channels.imessage.remoteAttachmentRoots` | `remoteHost` が SCP 取得用に設定されている場合にのみ使用されます。                                                                                                                                                                                                                              |
| `channels.bluebubbles.mediaMaxMb`                          | `channels.imessage.mediaMaxMb`            | iMessage のデフォルトは 16 MB です（BlueBubbles のデフォルトは 8 MB でした）。低い上限を維持するには明示的に設定してください。                                                                                                                                                                                  |
| `channels.bluebubbles.textChunkLimit`                      | `channels.imessage.textChunkLimit`        | どちらもデフォルトは 4000 です。                                                                                                                                                                                                                                                            |
| `channels.bluebubbles.coalesceSameSenderDms`               | _（削除済み）_                               | このキーは移行しないでください。`imsg` 0.13.1 以降では、OpenClaw が受信する前に Apple の URL プレビューによる分割送信を統合します。`openclaw doctor --fix` は古い iMessage キーを削除します。                                                                                                    |
| `channels.bluebubbles.enrichGroupParticipantsFromContacts` | _（該当なし）_                                   | `imsg` はすでに `chat.db` から送信者の表示名を公開します。                                                                                                                                                                                                                     |
| `channels.bluebubbles.actions.*`                           | `channels.imessage.actions.*`             | アクションごとのトグルは同じです（`reactions`、`edit`、`unsend`、`reply`、`sendWithEffect`、`renameGroup`、`setGroupIcon`、`addParticipant`、`removeParticipant`、`leaveGroup`、`sendAttachment`）。さらに新しい `polls` があります。すべてデフォルトで有効です。プライベート API アクションには引き続きブリッジが必要です。 |

複数アカウントの設定（`channels.bluebubbles.accounts.*`）は、`channels.imessage.accounts.*` に一対一で対応します。

## グループレジストリの落とし穴

バンドル版 iMessage Plugin は、2 つのグループゲートを連続して実行します。グループメッセージがエージェントに届くには、両方を通過する必要があります。

1. **送信者／チャット対象の許可リスト**（`channels.imessage.groupAllowFrom`）— 送信者ハンドルまたはチャット対象（`chat_id:`、`chat_guid:`、`chat_identifier:` のエントリ）と照合します。`groupAllowFrom` が未設定の場合、このゲートは `allowFrom` にフォールバックします。明示的に `groupAllowFrom: []` を指定するとそのフォールバックが無効になり、`groupPolicy: "allowlist"` ではすべてのグループメッセージが破棄されます。
2. **グループレジストリ**（`channels.imessage.groups`）— 数値の iMessage `chat_id` をキーとします。
   - `groups` ブロックがない場合（または空の場合）：ゲート 1 の実効的な送信者許可リストが空でなければ、グループはこのゲートを通過します。送信者フィルタリングがアクセスを制御し、全件破棄を示す起動時警告は発生しません。
   - `groups` にエントリがあるものの `"*"` がない場合：一覧にある `chat_id` キーだけが通過します。グループを 1 つでも列挙すると、`groupPolicy: "open"` の場合でもレジストリが許可リストになります。
   - `groups: { "*": { ... } }`：すべてのグループがこのゲートを通過します。

移行時の落とし穴：BlueBubbles は `groups` のエントリをチャット GUID／チャット識別子でキー付けしますが、iMessage レジストリは数値の `chat_id` をキーとします。グループごとのエントリをそのままコピーすると、どのキーも一致しない空ではないレジストリが作成されるため、すべてのグループメッセージがゲート 2 で破棄されます。`"*"` ワイルドカードはそのままコピーしてください。特定のグループのエントリは、`imsg chats` の `chat_id` 値を使用してキーを付け直します。

どちらの破棄経路も、デフォルトのログレベルで `warn` 行として確認できます。

- `groupPolicy: "allowlist"` が設定され、実効的なグループ送信者許可リストが空の場合、起動時にアカウントごとに 1 回：`imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...`。送信者を許可するには `groupAllowFrom`（または `allowFrom`）を設定します。`groups` を追加するだけでは送信者ゲートの条件を満たしません。
- レジストリがグループを破棄した場合、実行時に `chat_id` ごとに 1 回：`imessage: dropping group message from chat_id=<id> ... not in channels.imessage.groups allowlist`。追加すべき正確なキーが示されます。

どちらの場合でも DM は引き続き機能します。DM は別のコードパスを通るため、DM が成功してもグループルーティングが機能している証明にはなりません。

`groupPolicy: "allowlist"` を使用した、送信者スコープの最小設定：

```json5
{
  channels: {
    imessage: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123", "chat_guid:any;-;..."],
    },
  },
}
```

これにより、設定された送信者はどのグループでも許可されます。許可するチャットを限定したり、`requireMention` などのチャットごとのオプションを設定したりするには、`groups` エントリを追加します。BlueBubbles の `"*"` エントリはそのままコピーしますが、特定のエントリは数値の iMessage `chat_id` 値を使用してキーを付け直してください。

## 手順

1. 設定を変換します。編集中は新しいブロックを無効のままにしてください。現在の OpenClaw は古い `channels.bluebubbles` ブロックを無視するため、参照用として併記できます。

   ```json5
   {
     channels: {
       imessage: {
         enabled: false, // 切り替える準備ができたら true に変更
         cliPath: "/opt/homebrew/bin/imsg",
         dmPolicy: "pairing",
         allowFrom: ["+15555550123"], // bluebubbles.allowFrom からコピー
         groupPolicy: "allowlist",
         groupAllowFrom: [], // bluebubbles.groupAllowFrom からコピー
         groups: { "*": { requireMention: true } }, // ワイルドカードはそのままコピーし、チャットごとのエントリは chat_id をキーとして付け直す
         // アクションはデフォルトで有効。無効にするには個別のトグルを false に設定
       },
     },
   }
   ```

2. **切り替えてプローブします。** `channels.imessage.enabled: true` を設定し、Gateway を再起動して、チャンネルが正常と報告されることを確認します。

   ```bash
   openclaw gateway restart
   openclaw channels status --probe --channel imessage   # 「works」になることを確認。--json では privateApi.available: true と表示される
   ```

   プローブには到達可能な Gateway が必要で、設定済みかつ有効なアカウントのみがプローブされます。Mac 自体を検証するには、[開始する前に](#before-you-start)にある直接実行用の `imsg` コマンドを使用してください。

3. **DM を検証します。** エージェントにダイレクトメッセージを送信し、返信が届くことを確認します。

4. **グループを個別に検証します。** DM とグループでは異なるコードパスが使用されるため、DM の成功だけではグループがルーティングされていることを証明できません。許可されたグループチャットでメッセージを送信し、返信が届くことを確認します。グループが無反応になる場合（エージェントからの返信もエラーもない場合）は、Gateway ログで、上記の「グループレジストリの落とし穴」にある 2 行の `warn` を確認してください。起動時の警告は、有効な送信者許可リストが空であることを意味します。`chat_id` ごとの警告は、データが格納された `groups` レジストリにそのチャットが含まれていないことを意味します。

5. **アクションサーフェスを検証します。** ペアリング済みの DM から、リアクション、編集、送信取り消し、返信、写真の送信、および（グループ内での）グループ名の変更や参加者の追加・削除をエージェントに依頼します。各アクションは Messages.app でネイティブに反映される必要があります。いずれかのアクションで `iMessage <action> requires the imsg private API bridge` が発生した場合は、`imsg launch` を再度実行し、`openclaw channels status --probe` で更新してください。

6. iMessage の DM、グループ、アクションを検証したら、**BlueBubbles サーバーと `channels.bluebubbles` ブロックを削除します**。OpenClaw は `channels.bluebubbles` を読み取りません。

## アクション対応状況の概要

| アクション                                          | 旧 BlueBubbles     | バンドル版 iMessage                                                           |
| --------------------------------------------------- | ------------------ | ----------------------------------------------------------------------------- |
| テキスト送信 / SMS フォールバック                   | ✅                 | ✅                                                                            |
| メディア送信（写真、動画、ファイル、音声）          | ✅                 | ✅                                                                            |
| スレッド形式の返信（`reply_to_guid`）            | ✅                 | ✅（[#51892](https://github.com/openclaw/openclaw/issues/51892) を解決）       |
| Tapback（`react`）                       | ✅                 | ✅                                                                            |
| 編集 / 送信取り消し（macOS 13+ の受信者）           | ✅                 | ✅                                                                            |
| 画面エフェクト付き送信                              | ✅                 | ✅（[#9394](https://github.com/openclaw/openclaw/issues/9394) の一部を解決）   |
| リッチテキストの太字 / 斜体 / 下線 / 取り消し線     | ✅                 | ✅（attributedBody による型付きラン書式設定）                                 |
| Messages ネイティブの投票（作成と投票）             | ❌                 | ✅（`actions.polls`。ネイティブ表示には受信者側で iOS/macOS 26+ が必要）   |
| グループ名の変更 / グループアイコンの設定           | ✅                 | ✅                                                                            |
| 参加者の追加 / 削除、グループからの退出             | ✅                 | ✅                                                                            |
| 開封確認と入力中インジケーター                       | ✅                 | ✅（プライベート API プローブにより制御）                                     |
| Apple URL プレビュー分割送信の結合                  | ✅                 | ✅（`imsg` 0.13.1 以降で上流処理。OpenClaw の設定は不要）          |
| 再起動後の受信復旧                                  | ✅                 | ✅（自動：`since_rowid` の再生 + GUID 重複排除。ローカルでは期間が長い） |

iMessage は、Gateway の停止中に取りこぼしたメッセージを復旧します。起動時に、最後に配信した rowid から `imsg watch.subscribe` `since_rowid` を介して再生し、GUID で重複を排除します。また、古いバックログに対する経過時間の境界により、Push フラッシュの「バックログ爆弾」を抑制します。これは `imsg` RPC 接続を通じて実行されるため、リモート SSH の `cliPath` セットアップでも機能します。ローカルセットアップでは `chat.db` を読み取れるため、復旧期間が長くなります。[ブリッジまたは Gateway の再起動後の受信復旧](/ja-JP/channels/imessage#inbound-recovery-after-a-bridge-or-gateway-restart)を参照してください。

## ペアリング、セッション、ACP バインディング

- **許可リストはハンドル単位で引き継がれます。** `channels.imessage.allowFrom` は、BlueBubbles で使用していたものと同じ `+15555550123` / `user@example.com` 文字列を認識します。そのままコピーしてください。
- **ペアリングストアの承認は移行されません。** ペアリングストアはチャンネルごとに分かれており、古い BlueBubbles ストアは移行されません。ペアリングによってのみ承認されていた送信者は、iMessage で再度ペアリングする必要があります。または、そのハンドルを `allowFrom` に追加してください。
- **セッション**のスコープは、引き続きエージェントとチャットの組み合わせごとです。デフォルトの `session.dmScope=main` では、DM はエージェントのメインセッションに統合されます。グループセッションは `chat_id`（`agent:<agentId>:imessage:group:<chat_id>`）ごとに分離されたままです。BlueBubbles のセッションキーに保存された古い会話履歴は、iMessage セッションには引き継がれません。
- `match.channel: "bluebubbles"` を参照する **ACP バインディング**は、`"imessage"` に変更する必要があります。`match.peer.id` の形式（`chat_id:`、`chat_guid:`、`chat_identifier:`、ハンドルのみ）は同一です。

## ロールバック先のチャンネルなし

切り替え先としてサポートされている BlueBubbles ランタイムはありません。iMessage の検証に失敗した場合は、`channels.imessage.enabled: false` を設定して Gateway を再起動し、`imsg` の阻害要因を修正してから、切り替えを再試行してください。

返信キャッシュは SQLite の Plugin 状態に保存されます。`openclaw doctor --fix` は、古い `imessage/reply-cache.jsonl` サイドカーが存在する場合、それをインポートしてアーカイブします。

## 関連項目

- [BlueBubbles の削除と imsg iMessage パス](/ja-JP/announcements/bluebubbles-imessage) — 短い告知と運用者向け概要。
- [iMessage](/ja-JP/channels/imessage) — `imsg launch` のセットアップと機能検出を含む、iMessage チャンネルの完全なリファレンス。
- `/channels/bluebubbles` — この移行ガイドにリダイレクトされる旧 URL。
- [ペアリング](/ja-JP/channels/pairing) — DM の認証とペアリングの流れ。
- [チャンネルルーティング](/ja-JP/channels/channel-routing) — Gateway が送信返信に使用するチャンネルを選択する仕組み。
