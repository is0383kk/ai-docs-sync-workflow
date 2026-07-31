---
read_when:
    - Nextcloud Talk チャンネル機能の開発作業
summary: Nextcloud Talk のサポート状況、機能、設定
title: Nextcloud Talk
x-i18n:
    generated_at: "2026-07-26T08:52:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 59f4fe51555bcb13d630140866307b1a49ba077059818ec116ee50ef0c877b2b
    source_path: channels/nextcloud-talk.md
    workflow: 16
---

Nextcloud Talk は、Talk Webhook ボットを通じて OpenClaw をセルフホスト型 Nextcloud インスタンスに接続する、ダウンロード可能なチャンネル Plugin (`@openclaw/nextcloud-talk`) です。ダイレクトメッセージ、ルーム、リアクション、Markdown メッセージに対応しています。メディアは URL として送信されます。

## インストール

```bash
openclaw plugins install @openclaw/nextcloud-talk
```

現在の公式リリースタグに追従するには、バージョンを付けないパッケージ指定を使用します。再現可能なインストールが必要な場合にのみ、正確なバージョンを固定してください。

ローカルチェックアウトからインストールする場合（開発ワークフロー）:

```bash
openclaw plugins install ./path/to/local/nextcloud-talk-plugin
```

インストール後に Gateway を再起動します。詳細: [Plugin](/ja-JP/tools/plugin)

## クイックセットアップ（初心者向け）

1. Plugin をインストールします（上記を参照）。
2. Nextcloud サーバーでボットを作成します:

   ```bash
   ./occ talk:bot:install "OpenClaw" "<shared-secret>" "<webhook-url>" --feature webhook --feature response --feature reaction
   ```

   `--feature response` は維持してください。これがない場合、送信する返信は 401 で失敗します。既存のボットは `./occ talk:bot:state --feature webhook --feature response --feature reaction <botId> 1` で修復します。

3. 対象ルームの設定でボットを有効にします。
4. OpenClaw を設定します:
   - 設定: `channels.nextcloud-talk.baseUrl` + `channels.nextcloud-talk.botSecret`
   - または環境変数: `NEXTCLOUD_TALK_BOT_SECRET`（デフォルトアカウントのみ）

   CLI セットアップ（`--url`/`--token` は明示的なフィールドのエイリアスです。`nc-talk` と `nc` はチャンネルのエイリアスとして使用できます）:

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --url https://cloud.example.com \
     --token "<shared-secret>"
   ```

   同等の明示的なフィールド:

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --base-url https://cloud.example.com \
     --secret "<shared-secret>"
   ```

   ファイルに保存されたシークレット:

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --base-url https://cloud.example.com \
     --secret-file /path/to/nextcloud-talk-secret
   ```

5. Gateway を再起動します（またはセットアップを完了します）。

最小構成:

```json5
{
  channels: {
    "nextcloud-talk": {
      enabled: true,
      baseUrl: "https://cloud.example.com",
      botSecret: "shared-secret",
      dmPolicy: "pairing",
    },
  },
}
```

## 注意事項

- ボットから DM を開始することはできません。ユーザーが最初にボットへメッセージを送信する必要があります。
- Webhook URL は Nextcloud サーバーから到達可能である必要があります。Gateway がプロキシの背後にある場合は `webhookPublicUrl` を設定してください。Webhook リクエストはボットのシークレットを使用して HMAC-SHA256 で署名されます。無効な署名は拒否され、レート制限の対象になります。
- ボット API はメディアのアップロードに対応していません。送信メディアは `Attachment: <url>` 行として追加されます。
- Webhook ペイロードでは DM とルームを区別できません。ルーム種別の検索を有効にするには、`apiUser` + `apiPassword` を設定してください（約 5 分間キャッシュされます）。これらを設定しない場合、すべての会話がルームとして扱われます。
- 送信リクエストは SSRF ガードを通過します。信頼できるプライベート／内部ネットワーク上の Nextcloud ホストについては、`channels.nextcloud-talk.network.dangerouslyAllowPrivateNetwork: true` で明示的に許可してください。
- `apiUser`/`apiPassword` と `webhookPublicUrl` を設定すると、`openclaw channels status` がボットを検査し、`response` 機能がない場合に警告します。

## アクセス制御（DM）

- デフォルト: `channels.nextcloud-talk.dmPolicy = "pairing"`。不明な送信者にはペアリングコードが発行されます。
- 次の方法で承認します:
  - `openclaw pairing list nextcloud-talk`
  - `openclaw pairing approve nextcloud-talk <CODE>`
- 公開 DM: `channels.nextcloud-talk.dmPolicy="open"` と `channels.nextcloud-talk.allowFrom=["*"]`。
- `allowFrom` は Nextcloud のユーザー ID のみに一致します（小文字化されます）。表示名は無視されます。

## ルーム（グループ）

- デフォルト: `channels.nextcloud-talk.groupPolicy = "allowlist"`（メンション必須）。
- ルームトークンをキーとする `channels.nextcloud-talk.rooms` でルームを許可リストに登録します。`"*"` はワイルドカードのデフォルトを設定します:

```json5
{
  channels: {
    "nextcloud-talk": {
      rooms: {
        "room-token": { requireMention: true },
      },
    },
  },
}
```

- ルームごとのキー: `requireMention`（デフォルトは true）、`enabled`（false にするとルームを無効化）、`allowFrom`（ルームごとの送信者許可リスト）、`tools`（ツールの許可／拒否のオーバーライド）、`skills`（読み込む Skills を制限）、`systemPrompt`。
- すべてのルームを許可しない場合は、許可リストを空のままにするか、`channels.nextcloud-talk.groupPolicy="disabled"` を設定します。

## 機能

| 機能                 | 対応状況   |
| -------------------- | ---------- |
| ダイレクトメッセージ | 対応       |
| ルーム               | 対応       |
| スレッド             | 非対応     |
| メディア             | URL のみ   |
| リアクション         | 対応       |
| ネイティブコマンド   | 非対応     |

## 設定リファレンス（Nextcloud Talk）

すべての設定: [設定](/ja-JP/gateway/configuration)

プロバイダーオプション:

- `channels.nextcloud-talk.enabled`: チャンネルの起動を有効化／無効化します。
- `channels.nextcloud-talk.baseUrl`: Nextcloud インスタンスの URL。
- `channels.nextcloud-talk.botSecret`: ボットの共有シークレット（文字列またはシークレット参照）。
- `channels.nextcloud-talk.botSecretFile`: 通常ファイルのシークレットパス。シンボリックリンクは拒否されます。
- `channels.nextcloud-talk.apiUser`: ルーム検索（DM の検出）およびステータス検査に使用する API ユーザー。
- `channels.nextcloud-talk.apiPassword`: ルーム検索用の API／アプリパスワード。
- `channels.nextcloud-talk.apiPasswordFile`: API パスワードファイルのパス。
- `channels.nextcloud-talk.webhookPort`: Webhook リスナーポート（デフォルト: 8788）。
- `channels.nextcloud-talk.webhookHost`: Webhook ホスト（デフォルト: 0.0.0.0）。
- `channels.nextcloud-talk.webhookPath`: Webhook パス（デフォルト: /nextcloud-talk-webhook）。
- `channels.nextcloud-talk.webhookPublicUrl`: 外部から到達可能な Webhook URL。
- `channels.nextcloud-talk.dmPolicy`: `pairing | allowlist | open | disabled`（デフォルト: pairing）。`open` には `allowFrom=["*"]` が必要です。
- `channels.nextcloud-talk.allowFrom`: DM 許可リスト（ユーザー ID）。
- `channels.nextcloud-talk.groupPolicy`: `allowlist | open | disabled`（デフォルト: allowlist）。
- `channels.nextcloud-talk.groupAllowFrom`: ルーム送信者の許可リスト（ユーザー ID）。未設定の場合は `allowFrom` にフォールバックします。
- `channels.nextcloud-talk.rooms`: ルームごとの設定と許可リスト（上記を参照）。
- 静的な送信者アクセスグループは、`accessGroup:<name>` を使用して `allowFrom` および `groupAllowFrom` から参照できます。
- `channels.nextcloud-talk.historyLimit`: グループ履歴の上限（0 で無効）。
- `channels.nextcloud-talk.dmHistoryLimit`: DM 履歴の上限（0 で無効）。
- `channels.nextcloud-talk.dms`: ユーザー ID をキーとする DM ごとのオーバーライド（`historyLimit`）。
- `channels.nextcloud-talk.textChunkLimit`: 送信テキストのチャンクサイズ（文字数、デフォルト: 4000）。
- `channels.nextcloud-talk.streaming.chunkMode`: 長さによるチャンク分割の前に空行（段落の境界）で分割するには、`length`（デフォルト）または `newline`。
- `channels.nextcloud-talk.streaming.block.enabled`: このチャンネルのブロックストリーミングを有効化または無効化します。
- `channels.nextcloud-talk.streaming.block.coalesce`: ブロックストリーミングの結合調整。
- `channels.nextcloud-talk.responsePrefix`: 送信返信のプレフィックス。
- `channels.nextcloud-talk.markdown.tables`: Markdown テーブルのレンダリングモード（`off | bullets | code | block`）。
- `channels.nextcloud-talk.mediaMaxMb`: 受信メディアの上限（MB）。
- `channels.nextcloud-talk.network.dangerouslyAllowPrivateNetwork`: プライベート／内部 Nextcloud ホストが SSRF ガードを通過することを許可します。
- `channels.nextcloud-talk.accounts.<id>`: アカウントごとのオーバーライド（同じキー）。`defaultAccount` がデフォルトを選択します。環境変数 `NEXTCLOUD_TALK_BOT_SECRET` / `NEXTCLOUD_TALK_API_PASSWORD` はデフォルトアカウントのみに適用されます。

## 関連項目

- [チャンネルの概要](/ja-JP/channels) — 対応しているすべてのチャンネル
- [ペアリング](/ja-JP/channels/pairing) — DM の認証とペアリングの流れ
- [グループ](/ja-JP/channels/groups) — グループチャットの動作とメンション制御
- [チャンネルルーティング](/ja-JP/channels/channel-routing) — メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) — アクセスモデルと堅牢化
