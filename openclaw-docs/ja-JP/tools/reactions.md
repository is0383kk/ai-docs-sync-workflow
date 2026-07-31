---
read_when:
    - 任意のチャンネルでのリアクションの操作
    - プラットフォームごとの絵文字リアクションの違いを理解する
summary: サポートされているすべてのチャンネルにおけるリアクションツールのセマンティクス
title: リアクション
x-i18n:
    generated_at: "2026-07-26T09:48:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e148a93edbcfbe997075f6e9e191667ec257f76fa48162688fd1f333479661f0
    source_path: tools/reactions.md
    workflow: 16
---

エージェントは、`message` ツールの `react`
アクションを使用して絵文字リアクションを追加および削除します。動作はチャンネルによって異なります。

## 仕組み

```json
{
  "action": "react",
  "messageId": "msg-123",
  "emoji": "thumbsup"
}
```

- リアクションを追加する場合、`emoji` は必須です。
- 対応しているチャンネルでボットのリアクションを削除するには、`emoji` を空文字列（`""`）に設定します。
- 特定の絵文字を1つ削除するには、`remove: true` を設定します（空でない
  `emoji` が必要です）。
- ステータスリアクションに対応するチャンネルでは、リアクションで `trackToolCalls: true` を指定すると、
  ランタイムは同じターン中にそのリアクション対象メッセージを、以降のツール進行状況
  リアクションに再利用できます。

## チャンネルごとの動作

<AccordionGroup>
  <Accordion title="Discord と Slack">
    - 空の `emoji` を指定すると、メッセージ上のボットのすべてのリアクションが削除されます。
    - `remove: true` を指定すると、指定した絵文字のみが削除されます。

  </Accordion>

  <Accordion title="Nextcloud Talk">
    - リアクションの追加のみ対応しています。`emoji` は必須で、空にできません。
    - リアクションの削除はまだ削除呼び出しに接続されていません。`remove: true` は何もせずに暗黙的に処理されるのではなく、明示的なエラーで拒否されます。
    - `reaction` 機能を使用して登録された Talk ボットが必要です（[Nextcloud Talk チャンネルのドキュメント](/ja-JP/channels/nextcloud-talk)を参照）。

  </Accordion>

  <Accordion title="Telegram">
    - 空の `emoji` を指定すると、ボットのリアクションが削除されます。
    - `remove: true` でもリアクションは削除されますが、ツールの検証では引き続き空でない `emoji` が必要です。

  </Accordion>

  <Accordion title="WhatsApp">
    - 空の `emoji` を指定すると、ボットのリアクションが削除されます。
    - `remove: true` は内部的に空の絵文字へマッピングされます（ツール呼び出しでは引き続き `emoji` が必要です）。
    - WhatsApp では、メッセージごとにボット用のリアクション枠が1つあります。新しいリアクションを送信すると、複数の絵文字が積み重なるのではなく、既存のリアクションが置き換えられます。

  </Accordion>

  <Accordion title="Zalo Personal (zalouser)">
    - 追加と削除のどちらにも、空でない `emoji` が必要です。
    - `remove: true` を指定すると、その特定の絵文字リアクションが削除されます。

  </Accordion>

  <Accordion title="Feishu/Lark">
    - 個別のツールではなく、他のチャンネルと同じ `react` アクションを使用します（メッセージリアクション ID を介した追加、削除、一覧取得）。
    - 追加には空でない `emoji` が必要です（Feishu の `emoji_type` にマッピングされます。例：`SMILE`、`THUMBSUP`、`HEART`）。
    - `remove: true` には空でない `emoji` が必要で、その絵文字タイプに一致するボット自身のリアクションを削除します。
    - `clearAll: true` とともに空の `emoji` を指定すると、メッセージ上のボットのすべてのリアクションが削除されます。

  </Accordion>

  <Accordion title="Signal">
    - 受信リアクション通知は `channels.signal.reactionNotifications` で制御します。`"off"` は通知を無効にし、`"own"`（デフォルト）はユーザーがボットのメッセージにリアクションしたときにイベントを発行し、`"all"` はすべてのリアクションについてイベントを発行し、`"allowlist"` は `channels.signal.reactionAllowlist` に含まれる送信者についてのみイベントを発行します。

  </Accordion>

  <Accordion title="iMessage">
    - 送信リアクションは iMessage のタップバック（`love`、`like`、`dislike`、`laugh`、`emphasize`、`question`）です。リアクションを追加するには、`emoji` がこれらの種類のいずれかにマッピングされる必要があります。
    - 認識されるタップバックの種類を指定せずに `remove: true` を使用すると、すべてのタップバックの種類が削除されます。認識される種類を指定すると、その種類のみが削除されます。

  </Accordion>
</AccordionGroup>

## リアクションレベル

チャンネルごとの `reactionLevel` は、エージェントが自身のリアクションを送信する頻度を
制限します。値：`off`、`ack`、`minimal`、`extensive`。

- [Telegram のリアクション通知](/ja-JP/channels/telegram#feature-reference) - `channels.telegram.reactionLevel`（デフォルト：`minimal`）
- [WhatsApp のリアクションレベル](/ja-JP/channels/whatsapp#reaction-level) - `channels.whatsapp.reactionLevel`（デフォルト：`minimal`）
- [Signal のリアクション](/ja-JP/channels/signal#reactions-message-tool) - `channels.signal.reactionLevel`（デフォルト：`minimal`）

## 関連項目

- [エージェント送信](/ja-JP/tools/agent-send) - `react` を含む `message` ツール
- [チャンネル](/ja-JP/channels) - チャンネル固有の設定
