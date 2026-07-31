---
read_when:
    - DM アクセス制御の設定
    - 新しい iOS/Android Node のペアリング
    - OpenClaw のセキュリティ態勢のレビュー
summary: ペアリングの概要：DM を送信できる相手と参加できる Node を承認する
title: ペアリング
x-i18n:
    generated_at: "2026-07-26T09:13:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dc874d660509f59bc26795c8b3ce13f5d238cd61154c717637f5d545b995fb08
    source_path: channels/pairing.md
    workflow: 16
---

「ペアリング」は、OpenClaw における明示的なアクセス承認手順です。
次の 2 つの場面で使用されます。

1. **DM ペアリング**（ボットとの会話を許可する相手）
2. **Node ペアリング**（Gateway ネットワークへの参加を許可するデバイス／Node）

セキュリティの背景情報：[セキュリティ](/ja-JP/gateway/security)

## 1) DM ペアリング（受信チャットへのアクセス）

チャンネルの DM ポリシーが `pairing` に設定されている場合、不明な送信者には短いコードが提示され、承認されるまでそのメッセージは**処理されません**。

デフォルトの DM ポリシーについては、[セキュリティ](/ja-JP/gateway/security)を参照してください。

`dmPolicy: "open"` が公開アクセスになるのは、有効な DM 許可リストに `"*"` が含まれている場合のみです。
公開設定のセットアップと検証には、このワイルドカードが必要です。既存の
状態に具体的な `allowFrom` エントリを持つ `open` が含まれている場合、ランタイムが許可するのは
引き続きそれらの送信者のみであり、ペアリングストアで承認しても `open` のアクセス範囲は広がりません。

ペアリングコード：

- 8 文字、大文字、紛らわしい文字（`0O1I`）は不使用。
- **1 時間後に期限切れ**。ボットがペアリングメッセージを送信するのは、新しいリクエストが作成されたときだけです（送信者ごとにおよそ 1 時間に 1 回）。
- 保留中の DM ペアリングリクエストは、**チャンネルアカウントごとに 3 件**までです。いずれかが期限切れになるか承認されるまで、それ以降のリクエストは無視されます。

### Control UI から承認する

**Settings → Channels → DM access requests** を開きます。このキューには、DM ポリシーが `pairing` である
設定済みのすべてのチャンネルアカウントから、保留中のリクエストがまとめて表示されます。
チャンネルまたはアカウントで絞り込み、送信者 ID とメタデータを確認してから、
**Approve** を選択します。

承認によって付与されるのは、ダイレクトメッセージへのアクセスのみです。グループへのアクセスは付与されません。
対応している場合、承認ダイアログには次の明示的なオプションも表示されます。

- **Notify the requester after approval**
- **Also make this sender the first command owner**。コマンド所有者が存在せず、Control UI セッションに `operator.admin` がある場合にのみ
  表示されます

保留中のリクエストを承認せずに削除するには、**Dismiss** を選択します。却下しても
永続的にブロックされるわけではなく、送信者は後で再びアクセスをリクエストできます。

### CLI から承認する

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

同じチャンネルでリクエスト元に通知するには、`--notify` を追加します。複数アカウント対応チャンネルでは
`--account <id>` を指定します。

Control UI の明示的なチェックボックスとは異なり、CLI はコマンド所有者が設定されていない場合、
`telegram:123456789` のようなエントリを使用して `commands.ownerAllowFrom` を自動的に初期設定します。
これにより、初回セットアップ時に、特権コマンドと実行承認プロンプトを扱う明示的な所有者が設定されます。所有者が存在した後は、
以降のペアリング承認では DM アクセスのみが付与され、所有者が追加されることはありません。

<Note>
WhatsApp のログイン QR は、WhatsApp アカウントを OpenClaw にリンクします。DM アクセスリクエストは、
そのアカウントにメッセージを送る相手を承認します。これらは別々のフローです。
</Note>

対応チャンネル（ペアリングを宣言するインストール済みのチャンネル Plugin。`openclaw-weixin` などの外部 Plugin で追加可能）：`discord`、`feishu`、`googlechat`、`imessage`、`irc`、`line`、`matrix`、`mattermost`、`msteams`、`nextcloud-talk`、`nostr`、`signal`、`slack`、`sms`、`synology-chat`、`telegram`、`twitch`、`whatsapp`、`zalo`、`zalouser`。

### 再利用可能な送信者グループ

同じ信頼済み送信者の集合を複数のメッセージチャンネル、または DM とグループの両方の許可リストに
適用する場合は、トップレベルの `accessGroups` を使用します。

静的グループでは `type: "message.senders"` を使用し、チャンネルの許可リストから
`accessGroup:<name>` で参照します。

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
        whatsapp: ["+15551234567"],
      },
    },
  },
  channels: {
    telegram: { dmPolicy: "allowlist", allowFrom: ["accessGroup:operators"] },
    whatsapp: { groupPolicy: "allowlist", groupAllowFrom: ["accessGroup:operators"] },
  },
}
```

アクセスグループの詳細については、[アクセスグループ](/ja-JP/channels/access-groups)を参照してください。

### 状態の保存場所

共有 SQLite 状態データベース
`~/.openclaw/state/openclaw.sqlite` に保存されます。

- `channel_pairing_requests` 内の保留中リクエスト
- `channel_pairing_allow_entries` 内の承認済み送信者

アカウントのスコープ動作：

- 各リクエストと承認済み送信者は、チャンネルとアカウントをキーとして管理されます
- ランタイムは正規の SQLite 行のみを読み取り、レガシーファイルをマージしません

旧バージョンの Gateway は、`~/.openclaw/credentials/` の下に
`<channel>-pairing.json` と `<channel>-<accountId>-allowFrom.json` を書き込んでいました。
起動時の移行と `openclaw doctor --fix` は、これらのファイルを SQLite にインポートし、
インポートが成功すると各ソースを削除します。これらの行はアシスタントへのアクセスを制御するため、
SQLite データベースは機密情報として扱ってください。

<Note>
ペアリング許可リストストアは DM アクセス用です。グループの認可は別です。
DM ペアリングコードを承認しても、その送信者がグループでコマンドを実行したり、
グループ内のボットを操作したりできるようにはなりません。最初の所有者の初期設定は
`commands.ownerAllowFrom` 内の別の設定状態であり、グループチャットの配信には引き続き、
チャンネルのグループ許可リスト（たとえば `groupAllowFrom`、`groups`、またはチャンネルに応じたグループ単位
もしくはトピック単位のオーバーライド）が適用されます。
</Note>

## 2) Node デバイスのペアリング（iOS／Android／macOS／ヘッドレス Node）

Node は `role: node` を持つ**デバイス**として Gateway に接続します。Gateway は、
承認が必要なデバイスペアリングリクエストを作成します。

### Control UI からペアリングする（推奨）

`operator.admin` アクセスを持つ、接続済みの Control UI セッションを使用します。

1. Control UI を開き、**Settings → Devices** に移動します。
2. **Devices** ページで **Pair mobile device** をクリックします。
3. **Full access (recommended)** のままにするか、管理用 Gateway コントロールを除外するには
   **Limited access** を選択します。
4. **Create setup code** をクリックします。
5. スマートフォンで OpenClaw アプリを開き、**Settings** → **Gateway** に移動します。
6. QR コードをスキャンするかセットアップコードを貼り付けて、接続します。

公式の OpenClaw iOS／Android アプリは、セットアップコードのメタデータが一致すると
自動的に承認されます。**Pending approval** にリクエストが表示された場合（たとえば、
非公式クライアントやメタデータの不一致）、承認する前にロールとスコープを確認してください。

現在の Control UI セッションに管理者アクセスがない場合、ボタンは無効になります。
その場合は、Gateway ホストから以下の CLI 承認フローを使用してください。

### Telegram 経由でペアリングする

`device-pair` Plugin を使用している場合、初回のデバイスペアリングを Telegram だけで完了できます。

1. Telegram でボットに `/pair` とメッセージを送ります
2. ボットは 2 件のメッセージを返信します。1 件は手順のメッセージ、もう 1 件は独立した**セットアップコード**のメッセージです（Telegram で簡単にコピー＆ペーストできます）。
3. スマートフォンで OpenClaw iOS アプリを開き、Settings → Gateway に移動します。
4. QR コード（`/pair qr`）をスキャンするかセットアップコードを貼り付けて、接続します。
5. 公式モバイルアプリは自動的に接続されます。`/pair pending` にリクエストが表示された場合は、
   承認する前にロールとスコープを確認してください。

セットアップコードは、次の内容を含む base64 エンコード済み JSON ペイロードです。

- `url`：Gateway WebSocket URL（`ws://...` または `wss://...`）
- `urls`：利用可能な場合に、モバイルアプリが順番に試行できる LAN／Tailnet ルート
- `bootstrapToken`：初回ペアリングハンドシェイク用の 1 回限りのブートストラップトークン。Gateway は 10 分後に期限切れにします

ペアリングが完了したら、`/pair cleanup` を実行して未使用のセットアップコードを無効化します。

このブートストラップトークンには、組み込みのペアリング用ブートストラッププロファイルが含まれています。

- 安全な `wss://` セットアップ（または同一ホストのループバック）では、デフォルトで `node` に加えて、完全な
  ネイティブモバイル `operator` アクセスが付与されます
- 引き渡された `node` トークンは `scopes: []` のままです
- デフォルトで引き渡される `operator` トークンには、`operator.admin`、
  `operator.approvals`、`operator.read`、`operator.talk.secrets`、および
  `operator.write` が含まれます
- Control UI の **Limited access** と `openclaw qr --limited` では、
  他のオペレータースコープを維持しながら `operator.admin` が除外されます
- 平文 LAN `ws://` セットアップでは、同じ制限付きプロファイルが自動的に使用されます。
  完全なアクセスを得るには、`wss://` または Tailscale Serve を設定し、新しいコードを生成してください
- 以降のトークンのローテーション／取り消しは、デバイスで承認された
  ロール契約と、呼び出し元セッションのオペレータースコープの両方によって引き続き制限されます

セットアップコードが有効な間は、パスワードと同様に扱ってください。

iOS／Android の **Settings → Gateway** ページには、**Full** または **Limited**
アクセスが表示されます。制限付きのスマートフォンをアップグレードするには、まず安全な `wss://` または
Tailscale Serve ルートを設定し、完全アクセス用の新しいセットアップコードを生成して、その設定ページでスキャンまたは
貼り付けてから再接続します。

Tailscale、公開環境、またはその他のリモートモバイルペアリングには、Tailscale Serve／Funnel
または別の `wss://` Gateway URL を使用します。平文 `ws://` セットアップコードが許可されるのは、
local loopback、プライベート LAN アドレス、`.local` Bonjour ホスト、および Android
エミュレーターホストのみです。local loopback 以外の平文ルートには制限付きアクセスが付与されます。Tailnet
CGNAT アドレス、`.ts.net` 名、および公開ホストについては、QR／セットアップコードが発行される前に引き続きフェイルクローズします。

`gateway.bind=lan` セットアップ URL の場合、OpenClaw は、アクティブな Gateway の local loopback ポートを
プロキシする永続的な Tailscale Serve HTTPS ルートを検出し、LAN ルートとともに通知します。
セットアップコマンドがこのフォールバックを追加するのは
`lan` の場合のみです。`custom` と `tailnet` は、明示的に通知されたルートを維持します。
iOS アプリは通知されたルートを順番にプローブし、最初に到達できた
エンドポイントを保存します。

### Node デバイスを承認する

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

ペアリング専用スコープで開かれたペアリング済みデバイスセッションによる明示的な承認が
拒否された場合、CLI は同じリクエストを `operator.admin` で再試行します。これにより、既存の管理者権限を持つペアリング済みデバイスは、
ペアリングストアを手動で編集せずに、新しい Control UI／ブラウザーのペアリングを復旧できます。
Gateway は再試行された接続も引き続き検証し、`operator.admin` で認証できないトークンは
ブロックされたままです。

同じデバイスが異なる認証情報（たとえば異なるロール／スコープ／公開鍵）で再試行すると、
以前の保留中リクエストは置き換えられ、新しい
`requestId` が作成されます。

<Note>
ペアリング済みのデバイスに、より広いアクセス権が暗黙的に付与されることはありません。より多くのスコープや、より広いロールを要求して再接続した場合、OpenClaw は既存の承認をそのまま維持し、新しい保留中のアップグレードリクエストを作成します。承認する前に、`openclaw devices list` を使用して、現在承認されているアクセスと新しく要求されたアクセスを比較してください。
</Note>

### 信頼済み CIDR による Node の自動承認（任意）

デバイスのペアリングは、デフォルトでは手動のままです。厳密に管理された Node ネットワークでは、
CIDR または正確な IP を明示して、初回の Node 自動承認を有効にできます。

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

これは、要求されたスコープがない新規の `role: node` ペアリングリクエストにのみ
適用されます。オペレーター、ブラウザー、Control UI、および WebChat クライアントには、引き続き手動での
承認が必要です。ロール、スコープ、メタデータ、および公開鍵の変更にも、引き続き手動での
承認が必要です。

### Node ペアリング状態の保存場所

共有 SQLite 状態データベース `~/.openclaw/state/openclaw.sqlite` に保存されます。

- 保留中のデバイスペアリングリクエスト（短期間のみ保持され、5 分後に期限切れ）
- ペアリング済みデバイスとトークン

以前の Gateway では、この状態を `~/.openclaw/devices/*.json` に保持していました。これらのファイルは
Gateway の起動時に SQLite へインポートされ、`.migrated` サフィックスを付けてアーカイブされます。

### 注意事項

- `node.pair.*` API（CLI: `openclaw nodes pending|approve|reject|remove|rename`）は、同じペアリング済みデバイスレコードに保存されている
  Node 機能の承認を管理します。WS Node には引き続きデバイスのペアリングが必要です。詳しくは [Node のペアリング](/ja-JP/gateway/pairing)を
  参照してください。
- ペアリングレコードは、承認済みロールの永続的な信頼できる唯一の情報源です。アクティブな
  デバイストークンは、その承認済みロールセットの範囲内に制限されます。承認済みロールに含まれない
  不正なトークンエントリによって、新たなアクセス権が作成されることはありません。

## 関連ドキュメント

- セキュリティモデルとプロンプトインジェクション: [セキュリティ](/ja-JP/gateway/security)
- 安全な更新（doctor を実行）: [更新](/ja-JP/install/updating)
- チャンネル設定:
  - Telegram: [Telegram](/ja-JP/channels/telegram)
  - WhatsApp: [WhatsApp](/ja-JP/channels/whatsapp)
  - Signal: [Signal](/ja-JP/channels/signal)
  - iMessage: [iMessage](/ja-JP/channels/imessage)
  - Discord: [Discord](/ja-JP/channels/discord)
  - Slack: [Slack](/ja-JP/channels/slack)
