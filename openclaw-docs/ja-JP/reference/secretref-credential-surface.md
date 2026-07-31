---
read_when:
    - SecretRef 認証情報の適用範囲を検証する
    - 認証情報が `secrets configure` または `secrets apply` の対象となるかどうかを監査する
    - 認証情報がサポート対象範囲外である理由の確認
summary: SecretRef 認証情報サーフェスの正規のサポート対象と非サポート対象
title: SecretRef 認証情報サーフェス
x-i18n:
    generated_at: "2026-07-26T10:20:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbb1ad6c5045780e5ca8d9c20f2a0e86425317e86ef9aaa59957a2094344dd0d
    source_path: reference/secretref-credential-surface.md
    workflow: 16
---

このページでは、正規の SecretRef 認証情報サーフェスを定義します。つまり、どの認証情報フィールドが生のシークレット値の代わりに `SecretRef`（env/file/exec ベースの参照）を受け付けるかを示します。

範囲:

- 対象範囲: OpenClaw が発行またはローテーションしない、ユーザーが指定した認証情報のみ。
- 対象外: ランタイムが発行する認証情報またはローテーションされる認証情報、OAuth 更新用情報、セッションに類するアーティファクト。

以下のリストはソースターゲットレジストリから生成され、CI で `docs/reference/secretref-user-supplied-credentials-matrix.json` と照合されます。エントリを手動で編集しないでください。

## サポートされる認証情報

### `openclaw.json` ターゲット（`secrets configure` + `secrets apply` + `secrets audit`）

[//]: # "secretref-supported-list-start"

- `models.providers.*.apiKey`
- `models.providers.*.headers.*`
- `models.providers.*.request.auth.token`
- `models.providers.*.request.auth.value`
- `models.providers.*.request.headers.*`
- `models.providers.*.request.proxy.tls.ca`
- `models.providers.*.request.proxy.tls.cert`
- `models.providers.*.request.proxy.tls.key`
- `models.providers.*.request.proxy.tls.passphrase`
- `models.providers.*.request.tls.ca`
- `models.providers.*.request.tls.cert`
- `models.providers.*.request.tls.key`
- `models.providers.*.request.tls.passphrase`
- `skills.entries.*.apiKey`
- `memory.search.remote.apiKey`
- `agents.entries.*.tts.providers.*.apiKey`
- `agents.entries.*.memory.search.remote.apiKey`
- `talk.providers.*.apiKey`
- `talk.realtime.providers.*.apiKey`
- `tts.providers.*.apiKey`
- `plugins.entries.acpx.config.mcpServers.*.env.*`
- `plugins.entries.brave.config.webSearch.apiKey`
- `plugins.entries.codex.config.appServer.authToken`
- `plugins.entries.codex.config.appServer.headers.*`
- `plugins.entries.exa.config.webSearch.apiKey`
- `plugins.entries.firecrawl.config.webFetch.apiKey`
- `plugins.entries.google-meet.config.realtime.providers.*.apiKey`
- `plugins.entries.google.config.webSearch.apiKey`
- `plugins.entries.xai.config.webSearch.apiKey`
- `plugins.entries.moonshot.config.webSearch.apiKey`
- `plugins.entries.perplexity.config.webSearch.apiKey`
- `plugins.entries.firecrawl.config.webSearch.apiKey`
- `plugins.entries.minimax.config.webSearch.apiKey`
- `plugins.entries.tavily.config.webSearch.apiKey`
- `plugins.entries.parallel.config.webSearch.apiKey`
- `plugins.entries.voice-call.config.realtime.providers.*.apiKey`
- `plugins.entries.voice-call.config.streaming.providers.*.apiKey`
- `plugins.entries.voice-call.config.tts.providers.*.apiKey`
- `plugins.entries.voice-call.config.twilio.authToken`
- `plugins.entries.webhooks.config.routes.*.secret`
- `gateway.auth.password`
- `gateway.auth.token`
- `gateway.remote.token`
- `gateway.remote.password`
- `cron.webhookToken`
- `channels.telegram.botToken`
- `channels.telegram.webhookSecret`
- `channels.telegram.accounts.*.botToken`
- `channels.telegram.accounts.*.webhookSecret`
- `channels.slack.botToken`
- `channels.slack.appToken`
- `channels.slack.relay.authToken`
- `channels.slack.userToken`
- `channels.slack.signingSecret`
- `channels.slack.accounts.*.botToken`
- `channels.slack.accounts.*.appToken`
- `channels.slack.accounts.*.relay.authToken`
- `channels.slack.accounts.*.userToken`
- `channels.slack.accounts.*.signingSecret`
- `channels.sms.authToken`
- `channels.sms.accounts.*.authToken`
- `channels.clickclack.token`
- `channels.clickclack.accounts.*.token`
- `channels.discord.token`
- `channels.discord.pluralkit.token`
- `channels.discord.voice.tts.providers.*.apiKey`
- `channels.discord.accounts.*.token`
- `channels.discord.accounts.*.pluralkit.token`
- `channels.discord.accounts.*.voice.tts.providers.*.apiKey`
- `channels.irc.password`
- `channels.irc.nickserv.password`
- `channels.irc.accounts.*.password`
- `channels.irc.accounts.*.nickserv.password`
- `channels.feishu.appSecret`
- `channels.feishu.encryptKey`
- `channels.feishu.verificationToken`
- `channels.feishu.accounts.*.appSecret`
- `channels.feishu.accounts.*.encryptKey`
- `channels.feishu.accounts.*.verificationToken`
- `channels.qqbot.clientSecret`
- `channels.qqbot.accounts.*.clientSecret`
- `channels.msteams.appPassword`
- `channels.mattermost.botToken`
- `channels.mattermost.accounts.*.botToken`
- `channels.matrix.accessToken`
- `channels.matrix.password`
- `channels.matrix.accounts.*.accessToken`
- `channels.matrix.accounts.*.password`
- `channels.nextcloud-talk.botSecret`
- `channels.nextcloud-talk.apiPassword`
- `channels.nextcloud-talk.accounts.*.botSecret`
- `channels.nextcloud-talk.accounts.*.apiPassword`
- `channels.zalo.botToken`
- `channels.zalo.webhookSecret`
- `channels.zalo.accounts.*.botToken`
- `channels.zalo.accounts.*.webhookSecret`
- `channels.googlechat.serviceAccount`（兄弟 `serviceAccountRef` 経由、互換性の例外）
- `channels.googlechat.accounts.*.serviceAccount`（兄弟 `serviceAccountRef` 経由、互換性の例外）

### `auth-profiles.json` ターゲット（`secrets configure` + `secrets apply` + `secrets audit`）

- `profiles.*.keyRef`（`type: "api_key"`。`auth.profiles.<id>.mode = "oauth"` の場合はサポート対象外）
- `profiles.*.tokenRef`（`type: "token"`。`auth.profiles.<id>.mode = "oauth"` の場合はサポート対象外）

[//]: # "secretref-supported-list-end"

注記:

- 認証プロファイルのプランターゲットには `agentId` が必要です。プランエントリは `profiles.*.key` / `profiles.*.token` をターゲットとし、兄弟参照（`keyRef` / `tokenRef`）を書き込みます。認証プロファイル参照は、ランタイム解決および監査の対象に含まれます。
- `openclaw.json` では、SecretRef は `{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}` のような構造化オブジェクトを使用する必要があります。従来の `secretref-env:<ENV_VAR>` マーカー文字列は SecretRef 認証情報パスで拒否されます。有効なマーカーを移行するには、`openclaw doctor --fix` を実行してください。
- OAuth ポリシーガード: `auth.profiles.<id>.mode = "oauth"` は、そのプロファイルの SecretRef 入力と組み合わせることができません。このポリシーに違反すると、起動/再読み込みおよび認証プロファイルの解決は即座に失敗します。
- SecretRef で管理されるモデルプロバイダーでは、生成された `agents/*/agent/models.json` エントリに、`apiKey`/ヘッダーのサーフェス用として、シークレットではないマーカー（解決済みのシークレット値ではない）が永続化されます。マーカーの永続化ではソースが正となります。OpenClaw は、解決済みのランタイムシークレット値からではなく、アクティブなソース設定のスナップショット（解決前）からマーカーを書き込みます。
- Gateway のコールドスタートでは、マッピング済みの Gateway 以外の所有者について、再試行可能な解決エラーを分離できます。現在マッピングされているクラスには、モデルプロバイダーと Skills、メディア/TTS/cron プロバイダー、対象となる認証プロファイル、エージェントごとのメモリ、サンドボックス SSH、チャネルアカウント、マニフェストで宣言された Plugin ルートが含まれます。起動時には、失敗した各所有者の明示的な参照がランタイムスナップショットに保持され、ステータスと doctor を通じてその所有者が報告され、優先順位の低い認証情報を試すことなく、その所有者へのリクエストが拒否されます。再読み込みと設定書き込みの事前チェックでも、同じ所有者認識ポリシーが使用されます。正常な所有者は更新されます。対象となる失敗した所有者が古い状態のまま維持されるのは、その参照 ID、プロバイダー定義、およびシークレットではない所有者の完全な契約が変更されていない場合に限られます。新規または変更された失敗はコールド状態になります。Gateway の受信認証、構造的に無効な参照または値、フェイルクローズの所有者、現在マッピングされていない所有者については、引き続き厳格に処理されます。
- Web 検索について: 明示的なプロバイダーモード（`tools.web.search.provider` が設定されている場合）では、選択されたプロバイダーキーのみが有効です。自動モード（`tools.web.search.provider` が未設定の場合）では、優先順位に従って最初に解決されるプロバイダーキーのみが有効であり、選択されていないプロバイダー参照は、選択されるまで無効として扱われます。プロバイダーの認証情報には `plugins.entries.<plugin>.config.webSearch.*` を使用します。
- Slack `identity: "user"` は、Socket Mode では `channels.slack.appToken` とともに、HTTP モードでは `channels.slack.signingSecret` とともに `channels.slack.userToken` を使用します。`channels.slack.accounts.*` 配下でも同じ組み合わせが適用されます。この ID にはボットトークンは必要ありません。

## サポートされない認証情報

これらの認証情報は、発行、ローテーション、セッション保持、または OAuth で永続化されるクラスであり、読み取り専用の外部 SecretRef 解決には適合しません。

[//]: # "secretref-unsupported-list-start"

- `hooks.token`
- `hooks.gmail.pushToken`
- `hooks.mappings[].sessionKey`
- `auth-profiles.oauth.*`
- `channels.discord.threadBindings.webhookToken`
- `channels.discord.accounts.*.threadBindings.webhookToken`
- `channels.whatsapp.creds.json`
- `channels.whatsapp.accounts.*.creds.json`

[//]: # "secretref-unsupported-list-end"

## 関連項目

- [シークレット管理](/ja-JP/gateway/secrets)
- [認証情報のセマンティクス](/ja-JP/auth-credential-semantics)
