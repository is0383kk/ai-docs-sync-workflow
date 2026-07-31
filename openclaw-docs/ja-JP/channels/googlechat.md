---
read_when:
    - Google Chat チャンネル機能の開発
summary: Google Chat アプリのサポート状況、機能、設定
title: Google Chat
x-i18n:
    generated_at: "2026-07-26T10:04:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d3fb96564294b57040327bb21ab7331bf8412eb04f879a9c7ea1018ba2bddab
    source_path: channels/googlechat.md
    workflow: 16
---

Google Chat は公式の `@openclaw/googlechat` plugin として動作し、Google Chat API の Webhook を通じて DM とスペースをサポートします（HTTP エンドポイントのみ、Pub/Sub は使用しません）。

## インストール

```bash
openclaw plugins install @openclaw/googlechat
```

ローカルチェックアウト（git リポジトリから実行する場合）:

```bash
openclaw plugins install ./path/to/local/googlechat-plugin
```

## クイックセットアップ（初心者向け）

1. Google Cloud プロジェクトを作成し、**Google Chat API** を有効にします。
   - 次のページに移動します: [Google Chat API の認証情報](https://console.cloud.google.com/apis/api/chat.googleapis.com/credentials)
   - API がまだ有効になっていない場合は、有効にします。
2. **Service Account** を作成します:
   - **Create Credentials** > **Service Account** を押します。
   - 任意の名前を付けます（例: `openclaw-chat`）。
   - 権限とプリンシパルは空欄のままにします（**Continue**、続いて **Done**）。
3. **JSON キー**を作成してダウンロードします:
   - 新しいサービスアカウントをクリックし、**Keys** タブ > **Add Key** > **Create new key** > **JSON** > **Create** の順に選択します。
4. ダウンロードした JSON ファイルを Gateway ホストに保存します（例: `~/.openclaw/googlechat-service-account.json`）。
5. [Google Cloud Console の Chat Configuration](https://console.cloud.google.com/apis/api/chat.googleapis.com/hangouts-chat) で Google Chat アプリを作成します:
   - **Application info**（アプリ名、アバター URL、説明）を入力します。
   - **Interactive features** を有効にします。
   - **Functionality** で **Join spaces and group conversations** にチェックを入れます。
   - **Connection settings** で **HTTP endpoint URL** を選択します。
   - **Triggers** で **Use a common HTTP endpoint URL for all triggers** を選択し、公開 Gateway URL の末尾に `/googlechat` を付けた値を設定します（[公開 URL](#public-url-webhook-only)を参照）。
   - **Visibility** で **Make this Chat app available to specific people and groups in `<Your Domain>`** にチェックを入れ、自分のメールアドレスを入力します。
   - **Save** をクリックします。
6. アプリのステータスを有効にします。ページを更新して **App status** を見つけ、**Live - available to users** に設定し、もう一度 **Save** をクリックします。
7. サービスアカウントと Webhook のオーディエンスを OpenClaw に設定します（Chat アプリの設定と一致している必要があります）:
   - 環境変数: `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE=/path/to/service-account.json`（デフォルトアカウントのみ）、または
   - 設定: [設定の要点](#config-highlights)を参照してください。`openclaw channels add --channel googlechat` は `--audience-type`、`--audience`、`--webhook-path`、`--webhook-url` も受け付けます。
8. Gateway を起動します。Google Chat は Webhook パス（デフォルトは `/googlechat`）に POST します。

## Google Chat に追加する

Gateway が起動し、自分のメールアドレスが公開対象リストに登録されていることを確認します:

1. [Google Chat](https://chat.google.com/) に移動します。
2. **Direct Messages** の横にある **+**（プラス）アイコンをクリックします。
3. Google Cloud Console で設定した **App name** を検索します。
   - 非公開アプリであるため、ボットは Marketplace の閲覧リストには表示されません。名前で検索してください。
4. ボットを選択し、**Add** または **Chat** をクリックしてメッセージを送信します。

## 公開 URL（Webhook のみ）

Google Chat の Webhook には、公開 HTTPS エンドポイントが必要です。セキュリティのため、インターネットには **`/googlechat` パスのみ**を公開し、OpenClaw ダッシュボードとその他のエンドポイントは非公開にしてください。

### オプション A: Tailscale Funnel（推奨）

非公開ダッシュボードには Tailscale Serve、公開 Webhook パスには Funnel を使用します。

1. Gateway がバインドされているアドレスを確認します:

   ```bash
   ss -tlnp | grep 18789
   ```

   IP（例: `127.0.0.1`、`0.0.0.0`、または Tailscale の `100.x.x.x` アドレス）を控えます。

2. ダッシュボードを tailnet のみに公開します（ポート 8443）:

   ```bash
   # If bound to localhost (127.0.0.1 or 0.0.0.0):
   tailscale serve --bg --https 8443 http://127.0.0.1:18789

   # If bound to a Tailscale IP only:
   tailscale serve --bg --https 8443 http://100.x.x.x:18789
   ```

3. Webhook パスのみを公開します:

   ```bash
   # If bound to localhost (127.0.0.1 or 0.0.0.0):
   tailscale funnel --bg --set-path /googlechat http://127.0.0.1:18789/googlechat

   # If bound to a Tailscale IP only:
   tailscale funnel --bg --set-path /googlechat http://100.x.x.x:18789/googlechat
   ```

4. 求められた場合は、出力に表示された認証 URL にアクセスし、この Node で Funnel を有効にします。

5. 確認します:

   ```bash
   tailscale serve status
   tailscale funnel status
   ```

公開 Webhook URL は `https://<node-name>.<tailnet>.ts.net/googlechat` です。ダッシュボードは `https://<node-name>.<tailnet>.ts.net:8443/` で tailnet のみに公開されたままです。Google Chat アプリの設定には、公開 URL（`:8443` を除く）を使用します。

> 注: この設定は再起動後も維持されます。後で削除するには、`tailscale funnel reset` と `tailscale serve reset` を使用します。

### オプション B: リバースプロキシ（Caddy）

Webhook パスのみをプロキシします:

```caddy
your-domain.com {
    reverse_proxy /googlechat* localhost:18789
}
```

`your-domain.com/` へのリクエストは無視されるか 404 になり、`your-domain.com/googlechat` は OpenClaw にルーティングされます。

### オプション C: Cloudflare Tunnel

Webhook パスのみをルーティングするように、トンネルの ingress ルールを設定します:

- **Path**: `/googlechat` -> `http://localhost:18789/googlechat`
- **Default rule**: HTTP 404 (Not Found)

## 動作の仕組み

1. Google Chat は Gateway の Webhook パスに JSON を POST します（POST のみ、JSON コンテンツタイプ必須、IP ごとのレート制限あり）。
2. OpenClaw はディスパッチ前にすべてのリクエストを認証します:
   - Chat アプリイベントには `Authorization: Bearer <token>` が含まれます。本文全体を解析する前にトークンが検証されます。
   - Google Workspace アドオンイベントでは、トークンが本文（`authorizationEventObject.systemIdToken`）に含まれます。検証前に、より厳格な事前認証上限（16 KB、3 s）の下で読み取られます。
3. トークンは `audienceType` + `audience` に照らして確認されます:
   - `audienceType: "app-url"` → オーディエンスは HTTPS Webhook URL です。
   - `audienceType: "project-number"` → オーディエンスは Cloud プロジェクト番号です。
   - `app-url` のアドオントークンでは、さらに `appPrincipal` にアプリの数値 OAuth 2.0 クライアント ID（21 桁、メールアドレスではない）が設定されている必要があります。設定されていない場合、警告がログに記録され、検証は失敗します。
4. メッセージはスペースごとにルーティングされます:
   - スペースにはスペース単位のセッション `agent:<agentId>:googlechat:group:<spaceId>` が割り当てられ、返信はメッセージスレッドに送信されます。
   - DM はデフォルトでエージェントのメインセッションに統合されます。相手ごとの DM セッションを使用するには、`session.dmScope` を設定します（[セッション](/ja-JP/concepts/session)を参照）。
5. DM へのアクセスにはデフォルトでペアリングが使用されます。不明な送信者にはペアリングコードが送信されます。承認するには次を使用します:
   - `openclaw pairing approve googlechat <code>`
6. グループスペースでは、デフォルトで @メンションが必要です。メンションは、アプリを対象とする Chat の `USER_MENTION` アノテーションから検出されます。検出にアプリのユーザーリソース名が必要な場合は、`botUser`（例: `users/1234567890`）を設定します。
7. Google Chat から exec または plugin の承認が開始され、安定した `users/<id>` 承認者が設定されている場合、OpenClaw は開始元のスペースまたはスレッドにネイティブ承認カード（`cardsV2`）を投稿します。カードのボタンには不透明なコールバックトークンが含まれます。手動の `/approve <id> <decision>` プロンプトは、ネイティブ配信を利用できない場合にのみ表示されます。

### 受信処理の耐久性

リクエスト認証後、OpenClaw はアドオン認可オブジェクトをストレージから削除し、`200` を返す前に Google Chat の `MESSAGE` イベントを永続キューに格納します。永続化に失敗すると `503` が返されるため、失われる可能性があるイベントを受領済みとして扱う代わりに、Google Chat が再試行できます。

保留中または再試行可能なメッセージは Gateway の再起動後も保持され、スペースごとに直列化された状態を維持します。また、アクティブまたは保持済みの完了レコードが存在する間は、Google Chat のメッセージリソース名を使用してキューへの重複登録を抑止します。メッセージ以外のアクションでは、既存の切り離された Webhook パスが引き続き使用され、この永続キューの保証は適用されません。キューからエージェントへの境界を越える配信は少なくとも 1 回のままであるため、引き渡し中にクラッシュするとターンが再実行される可能性があります。

## ターゲット

配信と許可リストには、次の識別子を使用します:

- ダイレクトメッセージ: `users/<userId>`（推奨）。
- スペース: `spaces/<spaceId>`。
- 生のメールアドレス `name@example.com` は変更可能であり、`channels.googlechat.dangerouslyAllowNameMatching: true` の場合にのみ許可リスト照合に使用されます。
- 非推奨: `users/<email>` はメールアドレスの許可リストエントリではなく、ユーザー ID として扱われます。
- プレフィックス `googlechat:`、`google-chat:`、`gchat:` は受け付けられ、取り除かれます。

## 設定の要点

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      // or serviceAccountRef: { source: "file", provider: "filemain", id: "/channels/googlechat/serviceAccount" }
      audienceType: "app-url",
      audience: "https://gateway.example.com/googlechat",
      appPrincipal: "123456789012345678901", // add-on verification only; numeric OAuth client ID
      webhookPath: "/googlechat",
      botUser: "users/1234567890", // optional; helps mention detection
      allowBots: false,
      dmPolicy: "pairing",
      allowFrom: ["users/1234567890"],
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": {
          enabled: true,
          requireMention: true,
          users: ["users/1234567890"],
          systemPrompt: "Short answers only.",
        },
      },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

注:

- サービスアカウントの認証情報: `serviceAccountFile`（パス）、`serviceAccount`（インライン JSON 文字列またはオブジェクト）、または `serviceAccountRef`（環境変数またはファイルの SecretRef）。環境変数 `GOOGLE_CHAT_SERVICE_ACCOUNT`（インライン JSON）と `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`（パス）は、デフォルトアカウントにのみ適用されます。複数アカウント構成では、アカウントごとの `serviceAccountRef` を含む同じキーとともに `channels.googlechat.accounts.<id>` を使用します。
- `webhookPath` が未設定の場合、デフォルトの Webhook パスは `/googlechat` です。代わりに `webhookUrl` でパスを指定することもできます。
- グループキーには、安定したスペース ID（`spaces/<spaceId>`）を使用する必要があります。表示名のキーは非推奨であり、その旨がログに記録されます。
- `dangerouslyAllowNameMatching` は、許可リストで変更可能なメールプリンシパルの照合を再び有効にします（緊急時用の互換モード）。doctor はメールアドレスのエントリについて警告します。
- Google Chat のリアクションアクションは公開されません。この plugin はサービスアカウント認証を使用しますが、Google Chat のリアクションエンドポイントにはユーザー認証が必要です。既存の `actions.reactions` 設定は互換性のために受け付けられますが、効果はありません。
- ネイティブ承認カードでは、リアクションイベントではなく Google Chat の `cardsV2` ボタンクリックを使用します。承認者は `allowFrom` または `defaultTo` から取得され、安定した数値の `users/<id>` 値である必要があります。
- メッセージアクションでは、テキストの `send` のみが公開されます。Google Chat で添付ファイルをアップロードするにはユーザー認証が必要ですが、この plugin はサービスアカウント認証を使用するため、送信ファイルのアップロードは公開されません。
- `typingIndicator`: `message`（デフォルト）は `_<Bot> is typing..._` プレースホルダーを投稿し、それを最初の返信に編集します。`none` はこれを無効にします。`reaction` にはユーザー OAuth が必要であり、現在はサービスアカウント認証下でエラーをログに記録して `message` にフォールバックします。
- 受信添付ファイル（メッセージごとに最初の添付ファイル）は Chat API を通じてメディアパイプラインにダウンロードされ、`mediaMaxMb`（デフォルトは 20）で上限が設定されます。
- ボットが作成したメッセージはデフォルトで無視されます。`allowBots: true` を使用すると、受け付けたボットメッセージには共有の[ボットループ保護](/ja-JP/channels/bot-loop-protection)が使用されます。`channels.defaults.botLoopProtection` を設定し、`channels.googlechat.botLoopProtection` または `channels.googlechat.groups.<space>.botLoopProtection` で上書きします。

シークレットのリファレンス詳細：[シークレット管理](/ja-JP/gateway/secrets)。

## トラブルシューティング

### 405 Method Not Allowed

Google Cloud Logs Explorer に次のようなエラーが表示される場合：

```text
ステータスコード：405、理由フレーズ：HTTP エラーレスポンス：HTTP/1.1 405 Method Not Allowed
```

Webhook ハンドラーが登録されていません。一般的な原因：

1. **チャンネルが設定されていない**：`channels.googlechat` セクションがありません。次のコマンドで確認します：

   ```bash
   openclaw config get channels.googlechat
   ```

   「Config path not found」が返された場合は、設定を追加します（[設定の要点](#config-highlights)を参照）。

2. **Plugin が有効になっていない**：Plugin の状態を確認します：

   ```bash
   openclaw plugins list | grep googlechat
   ```

   「disabled」と表示された場合は、設定に `plugins.entries.googlechat.enabled: true` を追加します。

3. 設定変更後に **Gateway が再起動されていない**：

   ```bash
   openclaw gateway restart
   ```

チャンネルが実行中であることを確認します：

```bash
openclaw channels status
# 次のように表示されるはずです：Google Chat default: enabled, configured, ...
```

### その他の問題

- `openclaw channels status --probe` には、認証エラーとオーディエンス設定の欠落が表示されます（`audience` と `audienceType` は両方とも必須です）。
- メッセージが届かない場合は、Chat アプリの Webhook URL とトリガー設定を確認します。
- メンションゲーティングによって返信がブロックされる場合は、`botUser` をアプリのユーザーリソース名に設定し、`requireMention` を確認します。
- テストメッセージの送信中に `openclaw logs --follow` を実行すると、リクエストが Gateway に到達しているかどうかを確認できます。

## 関連項目

- [チャンネルの概要](/ja-JP/channels) — サポートされているすべてのチャンネル
- [チャンネルルーティング](/ja-JP/channels/channel-routing) — メッセージのセッションルーティング
- [Gateway の設定](/ja-JP/gateway/configuration)
- [グループ](/ja-JP/channels/groups) — グループチャットの動作とメンションゲーティング
- [ペアリング](/ja-JP/channels/pairing) — DM 認証とペアリングの流れ
- [セキュリティ](/ja-JP/gateway/security) — アクセスモデルと強化
