---
read_when:
    - SecretRef kimlik bilgisi kapsamını doğrulama
    - Bir kimlik bilgisinin `secrets configure` veya `secrets apply` için uygun olup olmadığını denetleme
    - Bir kimlik bilgisinin neden desteklenen kapsamın dışında olduğunu doğrulama
summary: SecretRef kimlik bilgileri için standart desteklenen ve desteklenmeyen yüzey
title: SecretRef kimlik bilgisi yüzeyi
x-i18n:
    generated_at: "2026-07-27T00:17:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbb1ad6c5045780e5ca8d9c20f2a0e86425317e86ef9aaa59957a2094344dd0d
    source_path: reference/secretref-credential-surface.md
    workflow: 16
---

Bu sayfa, standart SecretRef kimlik bilgisi yüzeyini tanımlar: hangi kimlik bilgisi alanlarının ham gizli değer yerine bir `SecretRef` (env/file/exec destekli referans) kabul ettiği.

Kapsam:

- Kapsamda: yalnızca OpenClaw tarafından oluşturulmayan veya döndürülmeyen, kesinlikle kullanıcı tarafından sağlanan kimlik bilgileri.
- Kapsam dışında: çalışma zamanında oluşturulan veya döndürülen kimlik bilgileri, OAuth yenileme materyali ve oturum benzeri yapılar.

Aşağıdaki listeler kaynak hedef kayıt defterinden oluşturulur ve CI'da `docs/reference/secretref-user-supplied-credentials-matrix.json` ile karşılaştırılarak denetlenir; girdileri elle düzenlemeyin.

## Desteklenen kimlik bilgileri

### `openclaw.json` hedefleri (`secrets configure` + `secrets apply` + `secrets audit`)

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
- `channels.googlechat.serviceAccount`, kardeş `serviceAccountRef` üzerinden (uyumluluk istisnası)
- `channels.googlechat.accounts.*.serviceAccount`, kardeş `serviceAccountRef` üzerinden (uyumluluk istisnası)

### `auth-profiles.json` hedefleri (`secrets configure` + `secrets apply` + `secrets audit`)

- `profiles.*.keyRef` (`type: "api_key"`; `auth.profiles.<id>.mode = "oauth"` olduğunda desteklenmez)
- `profiles.*.tokenRef` (`type: "token"`; `auth.profiles.<id>.mode = "oauth"` olduğunda desteklenmez)

[//]: # "secretref-supported-list-end"

Notlar:

- Kimlik doğrulama profili plan hedefleri `agentId` gerektirir; plan girdileri `profiles.*.key` / `profiles.*.token` öğelerini hedefler ve kardeş referansları (`keyRef` / `tokenRef`) yazar. Kimlik doğrulama profili referansları, çalışma zamanı çözümlemesine ve denetim kapsamına dahildir.
- `openclaw.json` içinde SecretRef'ler, `{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}` gibi yapılandırılmış nesneler kullanmalıdır. Eski `secretref-env:<ENV_VAR>` işaretleyici dizeleri SecretRef kimlik bilgisi yollarında reddedilir; geçerli işaretleyicileri taşımak için `openclaw doctor --fix` komutunu çalıştırın.
- OAuth politika koruması: `auth.profiles.<id>.mode = "oauth"`, ilgili profil için SecretRef girdileriyle birleştirilemez. Bu politika ihlal edildiğinde başlatma/yeniden yükleme ve kimlik doğrulama profili çözümlemesi hızlıca başarısız olur.
- SecretRef tarafından yönetilen model sağlayıcılarında oluşturulan `agents/*/agent/models.json` girdileri, `apiKey`/başlık yüzeyleri için gizli olmayan işaretleyicileri (çözümlenmiş gizli değerleri değil) kalıcı olarak saklar. İşaretleyici kalıcılığında kaynak belirleyicidir: OpenClaw, işaretleyicileri çözümlenmiş çalışma zamanı gizli değerlerinden değil, etkin kaynak yapılandırma anlık görüntüsünden (çözümleme öncesi) yazar.
- Soğuk Gateway başlatması, eşlenmiş ve Gateway sahibi olmayan bileşenlerdeki yeniden denenebilir çözümleme hatalarını yalıtabilir. Şu anda eşlenen sınıflar; model sağlayıcılarını ve becerileri, medya/TTS/cron sağlayıcılarını, uygun kimlik doğrulama profillerini, ajan başına belleği, korumalı alan SSH'sini, kanal hesaplarını ve manifestte bildirilen Plugin rotalarını içerir. Başlatma, başarısız olan her sahibin açık referanslarını çalışma zamanı anlık görüntüsünde tutar, sahibi durum ve doctor aracılığıyla bildirir ve daha düşük öncelikli kimlik bilgilerini denemeden bu sahibin isteklerini reddeder. Yeniden yükleme ve yapılandırma yazma ön kontrolü aynı sahip farkındalıklı politikayı kullanır: sağlıklı sahipler yenilenir; uygun bir başarısız sahip yalnızca referans kimlikleri, sağlayıcı tanımları ve gizli olmayan eksiksiz sahip sözleşmesi değişmemişse eski durumda kalır; yeni veya değişmiş bir hata soğuk duruma geçer. Gateway giriş kimlik doğrulaması, yapısal olarak geçersiz referanslar veya değerler, kapalı kalarak başarısız olan sahipler ve şu anda eşlenmemiş sahipler katı kalır.
- Web araması için: açık sağlayıcı modunda (`tools.web.search.provider` ayarlanmışsa) yalnızca seçilen sağlayıcı anahtarı etkindir. Otomatik modda (`tools.web.search.provider` ayarlanmamışsa) yalnızca öncelik sırasına göre çözümlenen ilk sağlayıcı anahtarı etkindir ve seçilmeyen sağlayıcı referansları seçilene kadar devre dışı kabul edilir. Sağlayıcı kimlik bilgileri `plugins.entries.<plugin>.config.webSearch.*` kullanır.
- Slack `identity: "user"`, Socket Mode için `channels.slack.appToken` veya HTTP modu için `channels.slack.signingSecret` ile birlikte `channels.slack.userToken` kullanır. Aynı eşleştirme `channels.slack.accounts.*` altında da geçerlidir; bu kimlik için bot belirteci gerekmez.

## Desteklenmeyen kimlik bilgileri

Bu kimlik bilgileri; oluşturulan, döndürülen, oturum taşıyan veya OAuth kapsamında kalıcı sınıflardır ve salt okunur harici SecretRef çözümlemesine uygun değildir:

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

## İlgili

- [Gizli bilgi yönetimi](/tr/gateway/secrets)
- [Kimlik doğrulama bilgisi semantiği](/tr/auth-credential-semantics)
