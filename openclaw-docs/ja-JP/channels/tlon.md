---
read_when:
    - Tlon/Urbit チャンネル機能の開発
summary: Tlon/Urbit のサポート状況、機能、設定
title: Tlon
x-i18n:
    generated_at: "2026-07-26T09:54:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d742628d6cf9aaf82d79a8d96b1685229905e9452c9fc4d3a494d2dee8d69943
    source_path: channels/tlon.md
    workflow: 16
---

Tlon は Urbit 上に構築された分散型メッセンジャーです。OpenClaw は Urbit ship に接続し、
DM とグループチャットのメッセージに応答します。グループでの返信にはデフォルトで @ メンションが必要で、
さらに認可ルールとオーナー承認フローが適用されます。

ステータス: バンドル済み Plugin。DM、グループメンション、スレッド、リッチテキスト、画像のアップロード／ダウンロード、
およびオーナー承認システムに対応しています。リアクションと投票には対応していません。

## バンドル済み Plugin

Tlon は現在の OpenClaw リリースにバンドルされており、パッケージビルドでは個別にインストールする必要はありません。

Tlon を含まない古いビルドまたはカスタムインストールでは、npm からインストールします。

```bash
openclaw plugins install @openclaw/tlon
```

現在のリリースタグを追跡するには、バージョンを付けないパッケージ名を使用します。再現可能なインストールが必要な場合にのみ、バージョン（`@openclaw/tlon@x.y.z`）
を固定してください。

ローカルチェックアウトからインストールする場合:

```bash
openclaw plugins install ./path/to/local/tlon-plugin
```

詳細: [Plugin](/ja-JP/tools/plugin)

## セットアップ

```bash
openclaw channels add --channel tlon --ship ~sampel-palnet --url https://your-ship-host --code lidlut-tabwed-pillex-ridrup
```

または、設定を直接編集します。

```json5
{
  channels: {
    tlon: {
      enabled: true,
      ship: "~sampel-palnet",
      url: "https://your-ship-host",
      code: "lidlut-tabwed-pillex-ridrup",
      ownerShip: "~your-main-ship", // 推奨: 自分の ship。常に認可される
    },
  },
}
```

設定を直接編集した後は Gateway を再起動します。その後、ボットに DM を送信するか、グループ
チャンネルで @ メンションします。

## 受信イベントの永続性

OpenClaw は、受理した Tlon の DM およびグループチャットイベントをエージェントへのディスパッチ前に永続化します。保留中または再試行可能なターンは Gateway の再起動後も維持され、処理はグループチャンネルまたは直接通信する相手ごとに直列化されたままです。安定した Urbit メッセージ ID により、キューレコードまたは保持された完了レコードが存在する間は、再配信されたイベントも抑制されます。

キューからエージェントへの境界をまたぐ配信は少なくとも 1 回行われます。引き渡し中にクラッシュすると、ターンが再実行される可能性があります。そのため、外部に副作用を生じさせるエージェントアクションは、実用上可能な限り冪等性を維持する必要があります。

## プライベート／LAN 上の ship

OpenClaw は、SSRF 防止のため、デフォルトでプライベート／内部ホスト名と IP 範囲をブロックします。
ship がプライベートネットワーク（localhost、LAN IP、内部ホスト名）で稼働している場合は、明示的に許可します。

```json5
{
  channels: {
    tlon: {
      url: "http://localhost:8080",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
    },
  },
}
```

`http://localhost:8080`、`http://192.168.x.x:8080`、
`http://my-ship.local:8080` などの対象に適用されます。信頼できる ship URL に対してのみ有効にしてください。この設定により、
そのアカウントの HTTP リクエストに対する SSRF 防止が無効になります。

<Note>
`channels.tlon.allowPrivateNetwork`（フラットキー）は廃止されています。`openclaw doctor --fix` により、
自動的に `channels.tlon.network.dangerouslyAllowPrivateNetwork` へ移動されます。
</Note>

## グループチャンネル

チャンネルを手動で固定するか、自動検出を有効にします。

```json5
{
  channels: {
    tlon: {
      groupChannels: ["chat/~host-ship/general", "chat/~host-ship/support"],
      autoDiscoverChannels: true,
    },
  },
}
```

設定で未指定の場合、`autoDiscoverChannels` のデフォルトは `false` です。セットアップウィザードでは
プロンプトのデフォルトが「はい」となり、`true` が明示的に書き込まれます。有効にすると、OpenClaw は起動時に参加済みグループを scry し、
グループ招待の承認に伴って新しいチャンネルを監視し、2 分ごとに再確認します。

## アクセス制御

DM 許可リスト（空の場合、送信者が `ownerShip` でない限り DM は許可されません）:

```json5
{
  channels: {
    tlon: {
      dmAllowlist: ["~zod", "~nec"],
    },
  },
}
```

グループ認可は、チャンネルごとにデフォルトで `restricted` になります。ベースラインとして
`defaultAuthorizedShips` を設定し、チャンネル nest ごとに上書きします。

```json5
{
  channels: {
    tlon: {
      defaultAuthorizedShips: ["~zod"],
      authorization: {
        channelRules: {
          "chat/~host-ship/general": {
            mode: "restricted",
            allowedShips: ["~zod", "~nec"],
          },
          "chat/~host-ship/announcements": {
            mode: "open",
          },
        },
      },
    },
  },
}
```

ボットがスレッド内で一度返信すると、そのスレッド内の以降のメッセージには、
再度メンションしなくても応答し続けます。

これらの後続メッセージにも新しい明示的なメンションを必須にするには、`channels.tlon.implicitMentions.threadParticipation: false` を設定します。
アカウント単位の上書きには `channels.tlon.accounts.<id>.implicitMentions` を使用します。Tlon は現在、
`replyToBot` または `quotedBot` のファクトを生成しないため、これらのフラグはここでは効果がありません。

## オーナーと承認システム

```json5
{
  channels: {
    tlon: {
      ownerShip: "~your-main-ship",
    },
  },
}
```

オーナー ship はすべての場所で認可されます。DM 招待は常に自動承認され、グループ招待も
常に自動承認され、チャンネルメッセージは常に認可を通過します。オーナーを
`dmAllowlist`、`defaultAuthorizedShips`、または `groupInviteAllowlist` に含める必要はありません。

`ownerShip` が設定されている場合、未認可のリクエストは単に破棄されるのではなく、承認待ちとして
キューに追加され、オーナーに DM で通知されます。

- `dmAllowlist` に含まれていない ship からの DM リクエスト
- 送信者が認可に失敗したチャンネルでのメンション
- `groupInviteAllowlist` に含まれていない ship からのグループ招待（自動承認が無効な場合、または有効でも
  招待者が許可リストに含まれていない場合）

オーナーはリクエストを処理するため、DM で返信します。

| オーナーの返信                  | 効果                                               |
| ---------------------------- | ---------------------------------------------------- |
| `approve` / `deny` / `block` | 最新の承認待ちを処理します             |
| `approve <id>` / `deny <id>` | ID を指定して特定の承認を処理します                    |
| `block`                      | ship をネイティブにブロックし、再接続も禁止します |
| `unblock ~ship`              | ネイティブブロックを解除します                              |
| `blocked`                    | 現在ブロック中の ship を一覧表示します                        |
| `pending`                    | 承認待ちのリクエストを一覧表示します                      |

`ownerShip` が設定されていない場合、未認可の DM とチャンネルメンションは単に破棄され、ログに記録されます。
承認プロンプトは表示されません。

## 自動承認設定

すでに `dmAllowlist` に含まれている ship からの DM 招待を自動承認します（このフラグに関係なく、
オーナーは常に自動承認されます）。

```json5
{
  channels: {
    tlon: {
      autoAcceptDmInvites: true,
    },
  },
}
```

許可リストに基づいてグループ招待を自動承認します（フェイルクローズ: `autoAcceptGroupInvites: true` で
`groupInviteAllowlist` が空の場合、オーナー以外からの招待は一切承認されません）。

```json5
{
  channels: {
    tlon: {
      autoAcceptGroupInvites: true,
      groupInviteAllowlist: ["~zod"],
    },
  },
}
```

## Urbit 設定ストアによるホットリロード

上記の設定の大半（`dmAllowlist`、`groupInviteAllowlist`、`groupChannels`、
`defaultAuthorizedShips`、`autoDiscoverChannels`、`autoAcceptDmInvites`、
`autoAcceptGroupInvites`、`ownerShip`、`showModelSignature`）は、初回実行時に ship の
`%settings` エージェント（desk `moltbot`、bucket `tlon`）へミラーリングされ、その後はそこからリアルタイムに読み込まれます。
そのため、Landscape クライアントまたはバンドル済み Skill の設定コマンドで行った変更は、
Gateway を再起動せずに適用されます。`channelRules` と承認待ちも JSON としてそこに永続化されます。
設定ストアに一度も書き込まれていない値については、ファイル設定が引き続き信頼できる情報源となります。

## 配信先（CLI/Cron）

`openclaw message send` または Cron 配信で使用します。

- DM: `~sampel-palnet` または `dm/~sampel-palnet`
- グループ: `chat/~host-ship/channel` または `group:~host-ship/channel`

## バンドル済み Skill

この Plugin には、Urbit を直接操作するための CLI である
[`@tloncorp/tlon-skill`](https://github.com/tloncorp/tlon-skill) がバンドルされており、Plugin のインストール後に自動的に使用できます。

- **アクティビティ**: メンション、返信、未読
- **チャンネル**: 一覧表示、作成、名前変更
- **連絡先**: プロフィールの一覧表示／取得／更新
- **グループ**: 作成、参加、招待／リクエストフロー、ロール
- **フック**: チャンネルフックの管理
- **メッセージ**: 履歴、検索
- **DM**: 送信、リアクション、承認／拒否
- **投稿**: リアクション、削除
- **ノートブック**: 日記チャンネルへの投稿
- **設定**: 上記の設定ストアを介した Plugin 設定のホットリロード

## 機能

| 機能         | ステータス                                        |
| --------------- | --------------------------------------------- |
| ダイレクトメッセージ | 対応                                     |
| グループ／チャンネル | 対応（デフォルトではメンション必須）          |
| スレッド         | 対応（参加後は返信を継続） |
| リッチテキスト       | Markdown を Tlon のネイティブ形式に変換    |
| 画像          | 受信時にダウンロード、送信時にアップロード         |
| リアクション       | [バンドル済み Skill](#bundled-skill) 経由のみ  |
| 投票           | 未対応                                 |
| ネイティブコマンド | デフォルトではオーナーのみ                         |

## トラブルシューティング

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
```

よくある問題:

- **DM が無視される**: 送信者が `dmAllowlist` に含まれておらず、承認フロー用の `ownerShip` も設定されていません。
- **グループメッセージが無視される**: チャンネルが検出／固定されていないか、送信者が認可に失敗し、承認を
  キューに追加するための `ownerShip` がありません。
- **接続エラー**: ship URL に到達できることを確認してください。ローカルの ship には
  `network.dangerouslyAllowPrivateNetwork` を設定します。
- **認証エラー**: ログインコードはローテーションされます。ship から現在のコードをコピーしてください。

## 設定リファレンス

完全な設定: [設定](/ja-JP/gateway/configuration)

| キー                                                    | 意味                                                        |
| ------------------------------------------------------ | -------------------------------------------------------------- |
| `channels.tlon.enabled`                                | チャンネルの起動を有効／無効にします。                                |
| `channels.tlon.ship`                                   | ボットの Urbit ship 名（例: `~sampel-palnet`）。                 |
| `channels.tlon.url`                                    | ship URL（例: `https://sampel-palnet.tlon.network`）。          |
| `channels.tlon.code`                                   | ship のログインコード。                                               |
| `channels.tlon.network.dangerouslyAllowPrivateNetwork` | localhost／LAN の ship URL を許可します（SSRF の明示的な許可）。                   |
| `channels.tlon.ownerShip`                              | オーナー ship。常に認可され、承認リクエストを受信します。     |
| `channels.tlon.dmAllowlist`                            | DM を許可する ship（空の場合、オーナー以外は許可されません）。              |
| `channels.tlon.autoAcceptDmInvites`                    | `dmAllowlist` に含まれる ship からの DM を自動承認します。                   |
| `channels.tlon.autoAcceptGroupInvites`                 | `groupInviteAllowlist` に基づいてグループ招待を自動承認します。         |
| `channels.tlon.groupInviteAllowlist`                   | グループ招待を自動承認する ship。                   |
| `channels.tlon.autoDiscoverChannels`                   | 参加済みグループチャンネルを自動検出します（デフォルト: `false`）。        |
| `channels.tlon.implicitMentions.threadParticipation`   | 参加済みスレッドの後続メッセージがメンション制限を回避できるようにします。      |
| `channels.tlon.groupChannels`                          | 手動で固定したチャンネル nest。                                 |
| `channels.tlon.defaultAuthorizedShips`                 | すべてのチャンネルで認可される ship（どのルールにも一致しない場合に使用）。 |
| `channels.tlon.authorization.channelRules`             | チャンネル nest ごとの認証モードと許可リスト。                        |
| `channels.tlon.showModelSignature`                     | 返信に `_[Generated by <model>]_` を追加します。                  |
| `channels.tlon.responsePrefix`                         | 送信する返信の先頭に付加する固定プレフィックス。                   |
| `channels.tlon.accounts.<id>`                          | 追加の名前付きアカウント（複数 ship 構成）。                 |

## 注記

- グループへの返信には、ボットがすでにそのスレッドに参加している場合を除き、@メンション（例: `~your-bot-ship`）が必要です。
- スレッドへの返信はスレッド内に投稿されます。また、エージェント向けに、スレッドの直近10件のメッセージがコンテキストとして先頭に追加されます。
- リッチテキスト（太字、斜体、コード、見出し、リスト）はTlonのネイティブ形式に変換されます。
- チャンネルの要約を求める受信メッセージ（例: 「このチャンネルを要約して」）を送信すると、通常の返信フローではなく、組み込みの履歴要約が実行されます。

## 関連項目

- [チャンネルの概要](/ja-JP/channels) — サポートされているすべてのチャンネル
- [ペアリング](/ja-JP/channels/pairing) — DM認証とペアリングの流れ
- [グループ](/ja-JP/channels/groups) — グループチャットの動作とメンションによる制御
- [チャンネルルーティング](/ja-JP/channels/channel-routing) — メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) — アクセスモデルと堅牢化
