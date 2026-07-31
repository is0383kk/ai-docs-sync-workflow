---
read_when:
    - WhatsAppグループを個別に設定する
    - WhatsApp の有効化モード（`mention` と `always`）の変更
    - WhatsApp グループのセッションキーまたは保留中メッセージのコンテキストの調整
sidebarTitle: WhatsApp groups
summary: WhatsAppグループメッセージの処理 — 有効化、許可リスト、セッション、コンテキスト注入
title: WhatsAppのグループメッセージ
x-i18n:
    generated_at: "2026-07-26T08:52:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7325dd3ae64d7abca8c1de0504f294ae280394fa5dd336d2532c5eaefcb03828
    source_path: channels/group-messages.md
    workflow: 16
---

クロスチャネルグループモデル（Discord、iMessage、Matrix、Microsoft Teams、QQBot、Signal、Slack、Telegram、WhatsApp、Zalo）については、[グループ](/ja-JP/channels/groups)を参照してください。このページでは、そのモデルに加えて、WhatsApp 固有の動作（アクティベーション、グループ許可リスト、グループごとのセッションキー、保留メッセージのコンテキスト注入）について説明します。

目標：OpenClaw を WhatsApp グループに参加させ、呼びかけられた場合にのみ起動し、そのスレッドを個人 DM セッションから分離して維持します。

<Note>
`agents.entries.*.groupChat.mentionPatterns` は、他のチャネルのメンションゲーティングと共有されます。マルチエージェント構成ではエージェントごとに設定するか、グローバルフォールバックとして `messages.groupChat.mentionPatterns` を使用します。どちらも設定されていない場合、パターンはエージェントのアイデンティティ名／絵文字から生成されます。
</Note>

## 動作

- アクティベーションモード：`mention`（デフォルト）または `always`。`mention` では、実際の WhatsApp @メンション（`mentionedJids`）、設定済みの正規表現パターン、テキスト内の任意の位置にあるボットの E.164 数字、またはボットのメッセージへの引用返信（共有番号によるセルフチャット構成を除く）のいずれかによる呼びかけが必要です。`always` はすべてのメッセージでエージェントを起動しますが、注入されるグループプロンプトでは、価値を加えられる場合にのみ返信し、それ以外の場合は正確なサイレントトークン `NO_REPLY`（大文字と小文字を区別しない）を返すよう指示されます。デフォルト値は設定（`channels.whatsapp.groups` `requireMention`）から取得され、`/activation` を使用してグループごとに上書きできます。
- グループ許可リスト：`channels.whatsapp.groups` が設定されている場合、リストに含まれるグループ JID のみが許可されます（すべてを許可するには `"*"` を含めます）。リストにないグループからのメッセージは、ログにヒントを残して破棄されます。
- グループポリシー：`channels.whatsapp.groupPolicy` は、グループメッセージを受け入れるかどうか（`open|disabled|allowlist`）を制御します。`allowlist` は `channels.whatsapp.groupAllowFrom`（フォールバック：明示的な `channels.whatsapp.allowFrom`）を使用します。デフォルトは `allowlist`（送信者を追加するまでブロック）です。
- グループごとのセッション：セッションキーは `agent:<agentId>:whatsapp:group:<jid>` のような形式になります（デフォルト以外のアカウントでは `:thread:whatsapp-account-<accountId>` が付加されます）。そのため、`/verbose on`、`/trace on`、`/think high` などのディレクティブ（単独のメッセージとして送信）は、そのグループにのみ適用され、個人 DM の状態には影響しません。
- コンテキスト注入：実行をトリガーしなかった**保留中のみ**のグループメッセージ（デフォルト 50 件）が `[Chat messages since your last reply - for context]` の下に、トリガーとなった行が `[Current message - respond to this]` の下に、それぞれ接頭辞付きで追加されます。保留ウィンドウは実行後にクリアされます。すでにセッション内にあるメッセージが再注入されることはありません。
- 送信者の帰属：各グループ行にはメッセージエンベロープ内に送信者ラベルが含まれます（例：`[WhatsApp <groupJid> <timestamp>] Alice (+447700900123): text`）。また、送信者のアイデンティティ、グループの件名、メンバー情報が、信頼されていない会話メタデータブロックに含まれます。
- 一時表示／一度だけ表示：テキストやメンションを抽出する前にラッパーが解除されるため、その中にある呼びかけでもトリガーされます。
- グループシステムプロンプト：グループセッションの最初のターン（および `/activation` によってモードが変更された後の各ターン）では、アクティベーションに関するガイダンスがシステムプロンプトに注入されます（`Activation: trigger-only ...` または `Activation: always-on ...` に加えて「特定の送信者に応答する」）。グループチャットへの配信に関する永続的なガイダンス（「WhatsApp グループチャットに参加しています…」）は常に含まれます。

## 設定例（WhatsApp）

WhatsApp がテキスト本文から視覚的な `@` を削除した場合でも、表示名による呼びかけが機能するようにします。

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
      },
      historyLimit: 50, // 保留中のグループコンテキストウィンドウ（デフォルト 50）
    },
  },
  agents: {
    entries: {
      main: {
        groupChat: {
          mentionPatterns: ["@?openclaw", "\\+?15555550123"],
        },
      },
    },
  },
}
```

注：

- 正規表現では大文字と小文字が区別されず、他の設定の正規表現サーフェスと同じ安全な正規表現のガードレールが使用されます。無効なパターンや安全でない入れ子の繰り返しは無視されます。
- 連絡先をタップすると、WhatsApp は引き続き `mentionedJids` を介して正規のメンションを送信するため、番号によるフォールバックが必要になることはほとんどありませんが、有用な安全策になります。
- 保留コンテキストウィンドウは、`channels.whatsapp.accounts.<id>.historyLimit` → `channels.whatsapp.historyLimit` → `messages.groupChat.historyLimit` → 50 の順で解決されます。

### アクティベーションコマンド（所有者のみ）

次のグループチャットコマンドを使用します。

- `/activation mention`
- `/activation always`

これを変更できるのは所有者の番号（`channels.whatsapp.allowFrom` から取得。未設定の場合はボット自身の E.164）のみです。それ以外のユーザーからの `/activation` は無視され、コンテキストとしてのみ保存されます。現在のアクティベーションモードを確認するには、グループ内で `/status` を単独のメッセージとして送信します。

## 使用方法

1. WhatsApp アカウント（OpenClaw を実行しているアカウント）をグループに追加します。
2. `@openclaw ...` と発言します（または番号を含めます）。`groupPolicy: "open"` を設定しない限り、許可リストに登録された送信者のみがトリガーできます。
3. エージェントプロンプトには、保留中のグループコンテキストと送信者ラベル付きの行が含まれるため、適切な相手に応答できます。
4. セッションディレクティブ（`/verbose on`、`/trace on`、`/think high`、`/new` または `/reset`、`/compact`）は、そのグループのセッションにのみ適用されます。認識されるように、単独のメッセージとして送信してください。個人 DM セッションは独立したままです。

## テスト／検証

- 手動スモークテスト：
  - グループで `@openclaw` による呼びかけを送信し、送信者名に言及する返信があることを確認します。
  - 2 回目の呼びかけを送信し、履歴ブロックが含まれていることを確認した後、次のターンでクリアされることを確認します。
- Gateway ログ（`--verbose` を指定して実行）を確認し、`from: <groupJid>` と送信者ラベル付き本文を示す `inbound web message` エントリがあることを確認します。

## 既知の考慮事項

- Heartbeat はエージェントのメインセッションで実行されます。グループセッションでは Heartbeat が実行されることはありません。
- エコー抑制では、セッションごとに結合済みプロンプト（履歴＋現在のメッセージ）が記憶されるため、ボット自身が配信したメッセージによって再トリガーされることはありません。同一のバッチが繰り返された場合、エコーとしてスキップされることがあります。
- セッションストアのエントリは、エージェントごとの SQLite セッションストア内に `agent:<agentId>:whatsapp:group:<jid>` として表示されます。エントリがない場合は、そのグループがまだ実行をトリガーしていないことを意味するだけです。
- 入力インジケーターは `agents.entries.*.typingMode`／`agents.defaults.typingMode` に従います。表示可能な返信でメッセージツール専用モードが選択されている場合、デフォルトでは入力状態が即座に開始されるため、自動的な最終返信が投稿されなくても、グループメンバーはエージェントが処理中であることを確認できます。明示的な入力モード設定がある場合は、そちらが優先されます。

## 関連項目

- [グループ](/ja-JP/channels/groups)
- [チャネルルーティング](/ja-JP/channels/channel-routing)
- [ブロードキャストグループ](/ja-JP/channels/broadcast-groups)
