---
read_when:
    - 驗證 SecretRef 認證資訊涵蓋範圍
    - 稽核認證資訊是否符合 `secrets configure` 或 `secrets apply` 的資格
    - 驗證認證資訊為何不在支援範圍內
summary: 標準支援與不支援的 SecretRef 認證資訊介面
title: SecretRef 認證資訊介面
x-i18n:
    generated_at: "2026-07-26T08:42:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbb1ad6c5045780e5ca8d9c20f2a0e86425317e86ef9aaa59957a2094344dd0d
    source_path: reference/secretref-credential-surface.md
    workflow: 16
---

此頁面定義標準的 SecretRef 認證資訊介面：哪些認證資訊欄位可接受 `SecretRef`（由環境變數／檔案／執行命令支援的參照），而非原始機密值。

範圍：

- 範圍內：僅限由使用者提供，且 OpenClaw 不會建立或輪替的認證資訊。
- 範圍外：執行階段建立或輪替的認證資訊、OAuth 重新整理資料，以及類似工作階段的成品。

以下清單由來源目標登錄檔產生，並在 CI 中對照 `docs/reference/secretref-user-supplied-credentials-matrix.json` 檢查；請勿手動編輯項目。

## 支援的認證資訊

### `openclaw.json` 目標（`secrets configure` + `secrets apply` + `secrets audit`）

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
- `channels.googlechat.serviceAccount`，透過同層的 `serviceAccountRef`（相容性例外）
- `channels.googlechat.accounts.*.serviceAccount`，透過同層的 `serviceAccountRef`（相容性例外）

### `auth-profiles.json` 目標（`secrets configure` + `secrets apply` + `secrets audit`）

- `profiles.*.keyRef`（`type: "api_key"`；當 `auth.profiles.<id>.mode = "oauth"` 時不支援）
- `profiles.*.tokenRef`（`type: "token"`；當 `auth.profiles.<id>.mode = "oauth"` 時不支援）

[//]: # "secretref-supported-list-end"

注意事項：

- 認證設定檔計畫目標需要 `agentId`；計畫項目以 `profiles.*.key` / `profiles.*.token` 為目標，並寫入同層參照（`keyRef` / `tokenRef`）。認證設定檔參照已納入執行階段解析與稽核涵蓋範圍。
- 在 `openclaw.json` 中，SecretRef 必須使用如 `{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}` 的結構化物件。SecretRef 認證資訊路徑會拒絕舊版 `secretref-env:<ENV_VAR>` 標記字串；請執行 `openclaw doctor --fix` 以遷移有效標記。
- OAuth 原則防護：該設定檔的 `auth.profiles.<id>.mode = "oauth"` 不得與 SecretRef 輸入搭配使用。違反此原則時，啟動／重新載入及認證設定檔解析會立即失敗。
- 對於由 SecretRef 管理的模型提供者，產生的 `agents/*/agent/models.json` 項目會在 `apiKey`／標頭介面中保存非機密標記（而非已解析的機密值）。標記保存以來源為準：OpenClaw 從使用中的來源設定快照（解析前）寫入標記，而非從已解析的執行階段機密值寫入。
- 閘道冷啟動可隔離已對應之非閘道擁有者的可重試解析失敗。目前已對應的類別包括模型提供者與 Skills、媒體／TTS／排程提供者、符合資格的認證設定檔、各代理程式記憶體、沙箱 SSH、頻道帳號，以及由資訊清單宣告的外掛路由。啟動時會將每個失敗擁有者的明確參照保留在執行階段快照中，透過狀態與 doctor 回報該擁有者，並拒絕該擁有者的要求，而不嘗試較低優先順序的認證資訊。重新載入與設定寫入預檢使用相同的擁有者感知原則：健康的擁有者會重新整理；符合資格但失敗的擁有者，只有在其參照身分、提供者定義及完整的非機密擁有者合約均未變更時，才會維持過時狀態；新的或已變更的失敗則會轉為冷狀態。閘道入口驗證、結構無效的參照或值、失敗時關閉的擁有者，以及目前尚未對應的擁有者，仍採嚴格處理。
- 對於網頁搜尋：在明確提供者模式（已設定 `tools.web.search.provider`）下，只有所選的提供者金鑰處於啟用狀態。在自動模式（未設定 `tools.web.search.provider`）下，只有依優先順序第一個成功解析的提供者金鑰處於啟用狀態，未選取的提供者參照在被選取前視為未啟用。提供者認證資訊使用 `plugins.entries.<plugin>.config.webSearch.*`。
- Slack `identity: "user"` 在 Socket Mode 中使用 `channels.slack.userToken` 搭配 `channels.slack.appToken`，或在 HTTP 模式中搭配 `channels.slack.signingSecret`。在 `channels.slack.accounts.*` 下亦採用相同配對；此身分不需要機器人權杖。

## 不支援的認證資訊

這些認證資訊屬於由系統建立、會輪替、帶有工作階段，或需長期保存的 OAuth 類別，不適合唯讀的外部 SecretRef 解析：

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

## 相關內容

- [機密管理](/zh-TW/gateway/secrets)
- [驗證認證資訊語意](/zh-TW/auth-credential-semantics)
