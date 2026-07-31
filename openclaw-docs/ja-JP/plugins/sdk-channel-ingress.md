---
read_when:
    - メッセージングチャネル Plugin の構築または移行
    - DM またはグループの許可リスト、ルートゲート、コマンド認証、イベント認証、メンションによる有効化の変更
    - チャネルの受信時の秘匿化または SDK 互換性境界のレビュー
sidebarTitle: Channel Ingress
summary: 受信メッセージ認可用の実験的なチャンネル受信 API
title: チャネル受信 API
x-i18n:
    generated_at: "2026-07-26T09:13:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 60feecb7bcf203cf37d2543a7855e89b5bfb2eb9d8263d804219e140facb8fc6
    source_path: plugins/sdk-channel-ingress.md
    workflow: 16
---

チャネル ingress は、受信チャネルイベントに対する実験的なアクセス制御境界です。Plugin はプラットフォーム固有の事実と副作用を所有し、コアは汎用ポリシー（DM/グループの許可リスト、ペアリングストアの DM エントリ、ルートゲート、コマンドゲート、イベント認証、メンションによるアクティベーション、秘匿化された診断、受け入れ）を所有します。

受信パスには `openclaw/plugin-sdk/channel-ingress-runtime` を使用します。

## ランタイムリゾルバー

```ts
import {
  defineStableChannelIngressIdentity,
  resolveChannelMessageIngress,
} from "openclaw/plugin-sdk/channel-ingress-runtime";

const identity = defineStableChannelIngressIdentity({
  key: "platform-user-id",
  normalize: normalizePlatformUserId,
  sensitivity: "pii",
});

const result = await resolveChannelMessageIngress({
  channelId: "my-channel",
  accountId,
  identity,
  subject: { stableId: platformUserId },
  conversation: { kind: isGroup ? "group" : "direct", id: conversationId },
  event: { kind: "message", authMode: "inbound", mayPair: !isGroup },
  policy: {
    dmPolicy: config.dmPolicy,
    groupPolicy: config.groupPolicy,
    groupAllowFromFallbackToAllowFrom: true,
  },
  allowFrom: config.allowFrom,
  groupAllowFrom: config.groupAllowFrom,
  accessGroups: cfg.accessGroups,
  route,
  readStoreAllowFrom,
  command: hasControlCommand ? { allowTextCommands: true, hasControlCommand } : undefined,
});
```

有効な許可リスト、コマンド所有者、コマンドグループを事前計算しないでください。リゾルバーは、生の許可リスト、ストアコールバック、ルート記述子、アクセスグループ、ポリシー、会話の種類からこれらを導出します。

## 結果

バンドルされた Plugin は、最新のプロジェクションを直接使用する必要があります。

| フィールド              | 意味                                                            |
| ------------------ | ------------------------------------------------------------------ |
| `ingress`          | 順序付けられたゲート判定と受け入れ                                |
| `senderAccess`     | 送信者と会話の認可のみ                             |
| `routeAccess`      | ルートとルート送信者のプロジェクション                                  |
| `commandAccess`    | コマンド認可。コマンドゲートが実行されなかった場合は `requested: false` |
| `activationAccess` | メンションとアクティベーションの結果                                          |

イベント認可は、順序付けられた `ingress.graph` と決定的な `ingress.reasonCode` で引き続き利用できます。個別のイベントプロジェクションは出力されません。

非推奨のサードパーティ SDK ヘルパーは、内部で以前の形式を再構築できます。新しいバンドル済み受信パスでは、最新の結果をローカル DTO に変換し直さないでください。

## アクセスグループ

`accessGroup:<name>` エントリは秘匿化されたままです。コアは静的な `message.senders` グループを自身で解決し、プラットフォーム検索が必要な動的グループに対してのみ `resolveAccessGroupMembership` を呼び出します。欠落しているグループ、サポートされていないグループ、解決に失敗したグループはフェイルクローズされます。

## イベントモード

| `authMode`       | 意味                                          |
| ---------------- | ------------------------------------------------ |
| `inbound`        | 通常の受信送信者ゲート                      |
| `command`        | コールバックまたはスコープ付きボタンのコマンドゲート    |
| `origin-subject` | アクターは元のメッセージのサブジェクトと一致する必要がある    |
| `route-only`     | ルートスコープの信頼済みイベントに対するルートゲートのみ |
| `none`           | Plugin 所有の内部イベントは共有認証をバイパスする  |

リアクション、ボタン、コールバック、ネイティブコマンドには `mayPair: false` を使用します。

## ルートとアクティベーション

ルーム、トピック、ギルド、スレッド、またはネストされたルートポリシーには、ルート記述子を使用します。

```ts
route: {
  id: "room",
  allowed: roomAllowed,
  enabled: roomEnabled,
  senderPolicy: "replace",
  senderAllowFrom: roomAllowFrom,
  blockReason: "room_sender_not_allowlisted",
}
```

Plugin に複数のオプションのルート記述子がある場合は、`channelIngressRoutes(...)` を使用します。これは、ルートの事実を汎用的に保ち、各記述子の `precedence` に従って順序付けながら、無効な分岐を除外します。

メンションゲートはアクティベーションゲートです。メンションが一致しない場合は `admission: "skip"` が返されるため、ターンカーネルは監視専用のターンを処理しません。ほとんどのチャネルでは、送信者ゲートとコマンドゲートの後にアクティベーションを配置する必要があります。送信者許可リストによるノイズより前に、メンションされていないトラフィックを抑制する必要がある公開チャットサーフェスでは、テキストコマンドのバイパスが無効な場合に `activation.order: "before-sender"` を選択できます。ボットスレッド内の返信など、暗黙的なアクティベーションを持つチャネルでは、`channels.defaults.implicitMentions` とチャネルおよびアカウントのオーバーライドを `resolveChannelImplicitMentions(...)` で解決し、その結果を `activation.implicitMentions` として渡します。プロジェクションされた `activationAccess.shouldBypassMention` は、コマンドまたは暗黙的なアクティベーションが明示的なメンションをバイパスした場合に報告します。

## 秘匿化

生の送信者値と生の許可リストエントリは、リゾルバーへの入力にのみ使用されます。これらを解決済み状態、判定、診断、スナップショット、互換性情報に含めてはなりません。不透明なサブジェクト ID、エントリ ID、ルート ID、診断 ID を使用してください。

## 検証

```bash
pnpm test src/channels/message-access/message-access.test.ts src/plugin-sdk/channel-ingress-runtime.test.ts
pnpm plugin-sdk:api:check
```
