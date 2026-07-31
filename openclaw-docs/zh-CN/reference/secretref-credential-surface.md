---
read_when:
    - 验证 SecretRef 凭据覆盖情况
    - 审查凭据是否符合 `secrets configure` 或 `secrets apply` 的条件
    - 验证凭据为何不在受支持范围内
summary: SecretRef 凭据的规范支持与不支持范围
title: SecretRef 凭据界面
x-i18n:
    generated_at: "2026-07-26T07:00:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbb1ad6c5045780e5ca8d9c20f2a0e86425317e86ef9aaa59957a2094344dd0d
    source_path: reference/secretref-credential-surface.md
    workflow: 16
---

此页面定义规范的 SecretRef 凭据接口：哪些凭据字段接受 `SecretRef`（由环境变量/文件/exec 支持的引用），而不是原始密钥值。

范围：

- 范围内：严格限于由用户提供、OpenClaw 不会生成或轮换的凭据。
- 范围外：运行时生成或轮换的凭据、OAuth 刷新材料以及类似会话的工件。

以下列表由源目标注册表生成，并在 CI 中根据 `docs/reference/secretref-user-supplied-credentials-matrix.json` 进行检查；请勿手动编辑条目。

## 支持的凭据

### `openclaw.json` 目标（`secrets configure` + `secrets apply` + `secrets audit`）

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
- `channels.googlechat.serviceAccount` 通过同级 `serviceAccountRef`（兼容性例外）
- `channels.googlechat.accounts.*.serviceAccount` 通过同级 `serviceAccountRef`（兼容性例外）

### `auth-profiles.json` 目标（`secrets configure` + `secrets apply` + `secrets audit`）

- `profiles.*.keyRef`（`type: "api_key"`；当 `auth.profiles.<id>.mode = "oauth"` 时不受支持）
- `profiles.*.tokenRef`（`type: "token"`；当 `auth.profiles.<id>.mode = "oauth"` 时不受支持）

[//]: # "secretref-supported-list-end"

注意：

- 身份验证配置文件计划目标需要 `agentId`；计划条目以 `profiles.*.key` / `profiles.*.token` 为目标，并写入同级引用（`keyRef` / `tokenRef`）。身份验证配置文件引用包含在运行时解析和审计覆盖范围内。
- 在 `openclaw.json` 中，SecretRef 必须使用 `{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}` 等结构化对象。SecretRef 凭据路径会拒绝旧版 `secretref-env:<ENV_VAR>` 标记字符串；运行 `openclaw doctor --fix` 以迁移有效标记。
- OAuth 策略防护：对于该配置文件，`auth.profiles.<id>.mode = "oauth"` 不能与 SecretRef 输入结合使用。违反此策略时，启动/重新加载和身份验证配置文件解析会快速失败。
- 对于由 SecretRef 管理的模型提供商，生成的 `agents/*/agent/models.json` 条目会为 `apiKey`/标头接口持久化非密钥标记（而非解析后的密钥值）。标记持久化以源为权威：OpenClaw 从活动源配置快照（解析前）写入标记，而不是从解析后的运行时密钥值写入。
- Gateway 网关冷启动可以隔离已映射的非 Gateway 网关所有者中可重试的解析失败。当前映射的类别包括模型提供商和 Skills、媒体/TTS/定时任务提供商、符合条件的身份验证配置文件、按 Agent 配置的记忆、沙箱 SSH、渠道账户以及清单声明的插件路由。启动时会在运行时快照中保留每个失败所有者的显式引用，通过状态和 Doctor 报告该所有者，并拒绝针对该所有者的请求，而不会尝试优先级较低的凭据。重新加载和配置写入预检使用相同的所有者感知策略：健康的所有者会刷新；仅当其引用标识、提供商定义和完整的非密钥所有者契约保持不变时，符合条件的失败所有者才会保持陈旧状态；新的或发生变化的失败则会进入冷状态。Gateway 网关入口身份验证、结构无效的引用或值、故障关闭型所有者以及当前未映射的所有者仍采用严格处理。
- 对于 Web 搜索：在显式提供商模式下（已设置 `tools.web.search.provider`），只有所选提供商密钥处于活动状态。在自动模式下（未设置 `tools.web.search.provider`），只有按优先级成功解析的第一个提供商密钥处于活动状态，在选中其他提供商之前，其引用均视为非活动状态。提供商凭据使用 `plugins.entries.<plugin>.config.webSearch.*`。
- Slack `identity: "user"` 在 Socket Mode 下使用 `channels.slack.userToken` 和 `channels.slack.appToken`，在 HTTP 模式下则使用 `channels.slack.signingSecret`。在 `channels.slack.accounts.*` 下也采用相同的配对；此身份不需要 Bot 令牌。

## 不支持的凭据

这些凭据属于生成、轮换、携带会话或 OAuth 持久化类别，不适用于只读的外部 SecretRef 解析：

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

## 相关内容

- [密钥管理](/zh-CN/gateway/secrets)
- [身份验证凭据语义](/zh-CN/auth-credential-semantics)
