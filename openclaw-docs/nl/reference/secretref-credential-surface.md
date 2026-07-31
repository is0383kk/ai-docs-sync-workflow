---
read_when:
    - Dekking van SecretRef-inloggegevens verifiëren
    - Controleren of een credential in aanmerking komt voor `secrets configure` of `secrets apply`
    - Verifiëren waarom een inloggegeven buiten het ondersteunde bereik valt
summary: Canoniek ondersteund versus niet-ondersteund SecretRef-referentieoppervlak voor inloggegevens
title: SecretRef-referentie voor inloggegevens
x-i18n:
    generated_at: "2026-07-27T06:12:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbb1ad6c5045780e5ca8d9c20f2a0e86425317e86ef9aaa59957a2094344dd0d
    source_path: reference/secretref-credential-surface.md
    workflow: 16
---

Deze pagina definieert het canonieke SecretRef-oppervlak voor referenties: welke referentievelden een `SecretRef` (een door env/file/exec ondersteunde referentie) accepteren in plaats van een onbewerkte geheime waarde.

Bereik:

- Binnen bereik: uitsluitend door de gebruiker aangeleverde referenties die OpenClaw niet uitgeeft of roteert.
- Buiten bereik: tijdens runtime uitgegeven of roterende referenties, OAuth-vernieuwingsmateriaal en sessieachtige artefacten.

De onderstaande lijsten worden gegenereerd vanuit het bronregister met doelen en in CI gecontroleerd aan de hand van `docs/reference/secretref-user-supplied-credentials-matrix.json`; bewerk de vermeldingen niet handmatig.

## Ondersteunde referenties

### `openclaw.json`-doelen (`secrets configure` + `secrets apply` + `secrets audit`)

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
- `channels.googlechat.serviceAccount` via naastgelegen `serviceAccountRef` (compatibiliteitsuitzondering)
- `channels.googlechat.accounts.*.serviceAccount` via naastgelegen `serviceAccountRef` (compatibiliteitsuitzondering)

### `auth-profiles.json`-doelen (`secrets configure` + `secrets apply` + `secrets audit`)

- `profiles.*.keyRef` (`type: "api_key"`; niet ondersteund wanneer `auth.profiles.<id>.mode = "oauth"`)
- `profiles.*.tokenRef` (`type: "token"`; niet ondersteund wanneer `auth.profiles.<id>.mode = "oauth"`)

[//]: # "secretref-supported-list-end"

Opmerkingen:

- Doelen van auth-profielplannen vereisen `agentId`; planvermeldingen zijn gericht op `profiles.*.key` / `profiles.*.token` en schrijven naastgelegen referenties (`keyRef` / `tokenRef`). Auth-profielreferenties zijn opgenomen in runtimeresolutie en auditdekking.
- In `openclaw.json` moeten SecretRefs gestructureerde objecten gebruiken, zoals `{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}`. Verouderde `secretref-env:<ENV_VAR>`-markeringstekenreeksen worden op SecretRef-referentiepaden geweigerd; voer `openclaw doctor --fix` uit om geldige markeringen te migreren.
- OAuth-beleidsbeveiliging: `auth.profiles.<id>.mode = "oauth"` kan voor dat profiel niet worden gecombineerd met SecretRef-invoer. Opstarten/herladen en auth-profielresolutie mislukken onmiddellijk wanneer dit beleid wordt geschonden.
- Voor door SecretRef beheerde modelproviders behouden gegenereerde `agents/*/agent/models.json`-vermeldingen niet-geheime markeringen (geen herleide geheime waarden) voor `apiKey`-/headeroppervlakken. Het behouden van markeringen is gebaseerd op de gezaghebbende bron: OpenClaw schrijft markeringen vanuit de actieve momentopname van de bronconfiguratie (vóór resolutie), niet vanuit herleide geheime runtimewaarden.
- Bij een koude start van de Gateway kunnen opnieuw uit te voeren resolutiefouten worden geïsoleerd voor toegewezen eigenaren die niet de Gateway zijn. De momenteel toegewezen klassen omvatten modelproviders en Skills, media-/TTS-/Cron-providers, geschikte auth-profielen, geheugen per agent, sandbox-SSH, kanaalaccounts en door het manifest gedeclareerde Plugin-routes. Bij het opstarten blijven de expliciete referenties van elke mislukte eigenaar in de runtimemomentopname behouden, wordt de eigenaar via status en doctor gemeld en worden aanvragen voor die eigenaar geweigerd zonder referenties met een lagere prioriteit te proberen. De voorafgaande controle bij herladen en schrijven van configuratie gebruikt hetzelfde eigenaarsbewuste beleid: gezonde eigenaren worden vernieuwd; een geschikte mislukte eigenaar blijft alleen verouderd wanneer de referentie-identiteiten, providerdefinities en volledige niet-geheime eigenaarscontracten ongewijzigd zijn; een nieuwe of gewijzigde fout wordt koud. Gateway-ingangsverificatie, structureel ongeldige referenties of waarden, eigenaren die bij fouten sluiten en momenteel niet-toegewezen eigenaren blijven strikt.
- Voor zoeken op het web: in expliciete providermodus (`tools.web.search.provider` ingesteld) is alleen de geselecteerde providersleutel actief. In automatische modus (`tools.web.search.provider` niet ingesteld) is alleen de eerste providersleutel die volgens prioriteit kan worden herleid actief, en worden niet-geselecteerde providerreferenties als inactief behandeld totdat ze worden geselecteerd. Providerreferenties gebruiken `plugins.entries.<plugin>.config.webSearch.*`.
- Slack `identity: "user"` gebruikt `channels.slack.userToken` met `channels.slack.appToken` voor Socket Mode of `channels.slack.signingSecret` voor HTTP-modus. Dezelfde combinatie geldt onder `channels.slack.accounts.*`; voor deze identiteit is geen bottoken vereist.

## Niet-ondersteunde referenties

Deze referenties zijn uitgegeven, roterend, sessiedragend of duurzaam voor OAuth en passen niet bij alleen-lezenresolutie van externe SecretRefs:

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

## Gerelateerd

- [Geheimenbeheer](/nl/gateway/secrets)
- [Semantiek van auth-referenties](/nl/auth-credential-semantics)
