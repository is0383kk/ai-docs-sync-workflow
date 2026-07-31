---
read_when:
    - OpenClaw で Synology Chat をセットアップする
    - Synology Chat Webhook ルーティングのデバッグ
summary: Synology Chat Webhook のセットアップと OpenClaw の設定
title: Synology Chat
x-i18n:
    generated_at: "2026-07-26T09:27:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3c03379944ee4187260a7287f6d2aed1ad8fdd1c22b5581c8a5d55515bbb6ad5
    source_path: channels/synology-chat.md
    workflow: 16
---

Synology Chat は Webhook のペアを介して OpenClaw に接続します。Synology Chat の送信 Webhook が受信したダイレクトメッセージを Gateway に POST し、返信は Synology Chat の受信 Webhook を介して返されます。

ステータス: 公式 Plugin。別途インストールが必要です。ダイレクトメッセージのみ対応し、テキストおよび URL ベースのファイル送信をサポートします。

## インストール

```bash
openclaw plugins install @openclaw/synology-chat
```

ローカルチェックアウト（git リポジトリから実行する場合）:

```bash
openclaw plugins install ./path/to/local/synology-chat-plugin
```

詳細: [Plugin](/ja-JP/tools/plugin)

## クイックセットアップ

1. Plugin をインストールします（上記を参照）。
2. Synology Chat のインテグレーションで、次の操作を行います。
   - 受信 Webhook を作成し、その URL をコピーします。
   - シークレットトークンを使用して送信 Webhook を作成します。
3. 送信 Webhook の URL を OpenClaw Gateway に設定します。
   - デフォルトでは `https://gateway-host/webhook/synology` です。
   - または、カスタムの `channels.synology-chat.webhookPath` を使用します。
4. OpenClaw でセットアップを完了します。どちらのフローでも、Synology Chat は同じチャンネルセットアップ一覧に表示されます。
   - ガイド付き: `openclaw onboard` または `openclaw channels add`
   - 直接: `openclaw channels add --channel synology-chat --token <token> --url <incoming-webhook-url>`
5. Gateway を再起動し、Synology Chat ボットに DM を送信します。

Webhook 認証の詳細:

- OpenClaw は送信 Webhook トークンを、まず `body.token`、次に
  `?token=...`、最後にヘッダーから受け入れます。
- 受け入れられるヘッダー形式:
  - `x-synology-token`
  - `x-webhook-token`
  - `x-openclaw-token`
  - `Authorization: Bearer <token>`
- トークンが空または欠落している場合は、フェイルクローズします。
- ペイロードには `application/x-www-form-urlencoded` または `application/json` を使用できます。`token`、`user_id`、および `text` は必須です。

## 受信処理の耐久性

トークン、送信者ポリシー、レート制限のチェックに合格すると、OpenClaw は保存されるエンベロープから Webhook トークンを削除し、応答を返す前にイベントを永続キューへ追加します。この追加が成功した場合にのみ、ルートは `204` を返します。永続化に失敗した場合は `503` を返すため、Synology Chat はメッセージを通知なく失うことなく再試行できます。

保留中または再試行可能なイベントは、Gateway の再起動後も保持されます。対応するアクティブな完了レコードまたは保持中の完了レコードが存在する間、Synology の安定した `post_id` によりキューへの重複登録が抑制されます。キューからエージェントへの引き渡しでは少なくとも 1 回の配信が維持されるため、その境界でクラッシュするとターンが再実行される可能性があります。

最小構成:

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      token: "synology-outgoing-token",
      incomingUrl: "https://nas.example.com/webapi/entry.cgi?api=SYNO.Chat.External&method=incoming&version=2&token=...",
      webhookPath: "/webhook/synology",
      dmPolicy: "allowlist",
      allowedUserIds: ["123456"],
      rateLimitPerMinute: 30,
      allowInsecureSsl: false,
    },
  },
}
```

## 環境変数

デフォルトアカウントでは、環境変数を使用できます。

- `SYNOLOGY_CHAT_TOKEN`
- `SYNOLOGY_CHAT_INCOMING_URL`
- `SYNOLOGY_NAS_HOST`
- `SYNOLOGY_ALLOWED_USER_IDS`（カンマ区切り）
- `SYNOLOGY_RATE_LIMIT`
- `OPENCLAW_BOT_NAME`

設定値は環境変数より優先されます。

`SYNOLOGY_CHAT_INCOMING_URL` および `SYNOLOGY_NAS_HOST` は、ワークスペースの `.env` から設定できません。[ワークスペースの `.env` ファイル](/ja-JP/gateway/security#workspace-env-files)を参照してください。

## DM ポリシーとアクセス制御

- サポートされる `dmPolicy` の値は、`allowlist`（デフォルト）、`open`、および `disabled` です。Synology Chat にはペアリングフローがありません。送信者を承認するには、その数値形式の Synology ユーザー ID を `allowedUserIds` に追加します。
- `allowedUserIds` には、Synology ユーザー ID のリスト（またはカンマ区切りの文字列）を指定できます。
- `allowlist` モードでは、`allowedUserIds` のリストが空の場合は設定ミスとして扱われ、Webhook ルートは起動しません。
- `dmPolicy: "open"` で公開 DM が許可されるのは、`allowedUserIds` に `"*"` が含まれている場合のみです。制限的なエントリがある場合、該当するユーザーのみがチャットできます。`open` で `allowedUserIds` のリストが空の場合も、ルートの起動を拒否します。
- `dmPolicy: "disabled"` は DM をブロックします。
- デフォルトでは、返信先のバインディングは安定した数値形式の `user_id` に維持されます。`channels.synology-chat.dangerouslyAllowNameMatching: true` は、返信配信で変更可能なユーザー名またはニックネームの検索を再び有効にする緊急時用の互換モードです。

## 送信配信

送信先には、数値形式の Synology Chat ユーザー ID を使用します。`synology-chat:`、`synology_chat:`、および `synology:` のプレフィックスを使用できます。

例:

```bash
openclaw message send --channel synology-chat --target 123456 --message "Hello from OpenClaw"
openclaw message send --channel synology-chat --target synology-chat:123456 --message "Hello again"
openclaw message send --channel synology-chat --target synology:123456 --message "Short prefix"
```

送信テキストは 2000 文字単位で分割されます。メディア送信では、URL ベースのファイル配信がサポートされます。NAS がファイルをダウンロードして添付します（最大 32 MB）。送信ファイルの URL には `http` または `https` を使用する必要があります。プライベートなネットワーク送信先やその他のブロック対象のネットワーク送信先は、OpenClaw が URL を NAS の Webhook に転送する前に拒否されます。

## マルチアカウント

`channels.synology-chat.accounts` では複数の Synology Chat アカウントがサポートされます。
各アカウントで、トークン、受信 URL、Webhook パス、DM ポリシー、および制限を上書きできます。
ダイレクトメッセージのセッションはアカウントおよびユーザーごとに分離されるため、2 つの異なる Synology アカウントで同じ数値形式の `user_id` を使用しても、トランスクリプトの状態は共有されません。
有効化する各アカウントには、固有の `webhookPath` を指定してください。OpenClaw は完全に同一のパスの重複を拒否し、マルチアカウント構成で共有 Webhook パスを継承するだけの名前付きアカウントについても起動を拒否します。
名前付きアカウントで意図的に従来の継承を使用する必要がある場合は、そのアカウントまたは `channels.synology-chat` に `dangerouslyAllowInheritedWebhookPath: true` を設定します。ただし、完全に同一のパスの重複は引き続きフェイルクローズで拒否されます。アカウントごとに明示的なパスを指定することを推奨します。

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      accounts: {
        default: {
          token: "token-a",
          incomingUrl: "https://nas-a.example.com/...token=...",
        },
        alerts: {
          token: "token-b",
          incomingUrl: "https://nas-b.example.com/...token=...",
          webhookPath: "/webhook/synology-alerts",
          dmPolicy: "allowlist",
          allowedUserIds: ["987654"],
        },
      },
    },
  },
}
```

## セキュリティ上の注意

- `token` は秘密に保ち、漏洩した場合はローテーションしてください。
- 自己署名されたローカル NAS 証明書を明示的に信頼する場合を除き、`allowInsecureSsl: false` のままにしてください。
- 受信 Webhook リクエストはトークンで検証され、送信者ごとにレート制限されます（`rateLimitPerMinute`、デフォルト 30）。
- 無効なトークンのチェックでは、定数時間のシークレット比較を使用してフェイルクローズします。無効なトークンによる試行が繰り返されると、送信元 IP が一時的にロックアウトされます。
- 受信メッセージのテキストは、既知のプロンプトインジェクションパターンに対してサニタイズされ、4000 文字で切り詰められます。
- 本番環境では `dmPolicy: "allowlist"` を推奨します。
- 従来のユーザー名ベースの返信配信が明示的に必要な場合を除き、`dangerouslyAllowNameMatching` はオフのままにしてください。
- マルチアカウント構成で共有パスのルーティングリスクを明示的に許容する場合を除き、`dangerouslyAllowInheritedWebhookPath` はオフのままにしてください。

## トラブルシューティング

- `Missing required fields (token, user_id, text)`:
  - 送信 Webhook のペイロードに必須フィールドのいずれかがありません
  - Synology がヘッダーでトークンを送信する場合は、Gateway またはプロキシがそれらのヘッダーを保持していることを確認してください
- `Invalid token`:
  - 送信 Webhook のシークレットが `channels.synology-chat.token` と一致していません
  - リクエストが誤ったアカウントまたは Webhook パスに送信されています
  - リクエストが OpenClaw に到達する前に、リバースプロキシによってトークンヘッダーが削除されています
- `Rate limit exceeded`:
  - 同じ送信元から無効なトークンによる試行が多すぎると、その送信元が一時的にロックアウトされる場合があります
  - 認証済みの送信者には、ユーザーごとのメッセージレート制限も別途適用されます
- `Allowlist is empty. Configure allowedUserIds or use dmPolicy=open with allowedUserIds=["*"].`:
  - `dmPolicy="allowlist"` が有効ですが、ユーザーが設定されていません
- `User not authorized`:
  - 送信者の数値形式の `user_id` が `allowedUserIds` に含まれていません

## 関連項目

- [チャンネルの概要](/ja-JP/channels) — サポートされるすべてのチャンネル
- [グループ](/ja-JP/channels/groups) — グループチャットの動作とメンションの制御
- [チャンネルルーティング](/ja-JP/channels/channel-routing) — メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) — アクセスモデルと堅牢化
