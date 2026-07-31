---
read_when:
    - بررسی پوشش اعتبارنامه‌های SecretRef
    - بررسی اینکه آیا یک اعتبارنامه واجد شرایط `secrets configure` یا `secrets apply` است
    - بررسی دلیل خارج‌بودن یک اعتبارنامه از محدوده پشتیبانی‌شده
summary: سطح مرجع پشتیبانی‌شده و پشتیبانی‌نشدهٔ اعتبارنامه‌های SecretRef canonical
title: سطح اعتبارنامهٔ SecretRef
x-i18n:
    generated_at: "2026-07-27T17:07:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbb1ad6c5045780e5ca8d9c20f2a0e86425317e86ef9aaa59957a2094344dd0d
    source_path: reference/secretref-credential-surface.md
    workflow: 16
---

این صفحه سطح استاندارد اعتبارنامه‌های SecretRef را تعریف می‌کند: اینکه کدام فیلدهای اعتبارنامه به‌جای مقدار خام راز، یک `SecretRef` (ارجاع مبتنی بر env/file/exec) را می‌پذیرند.

دامنه:

- در دامنه: صرفاً اعتبارنامه‌هایی که کاربر ارائه می‌کند و OpenClaw آن‌ها را ایجاد یا چرخش نمی‌دهد.
- خارج از دامنه: اعتبارنامه‌های ایجادشده یا چرخشی در زمان اجرا، داده‌های تازه‌سازی OAuth و مصنوعات شبه‌نشست.

فهرست‌های زیر از رجیستری مقصدهای منبع تولید و در CI با `docs/reference/secretref-user-supplied-credentials-matrix.json` بررسی می‌شوند؛ ورودی‌ها را دستی ویرایش نکنید.

## اعتبارنامه‌های پشتیبانی‌شده

### مقصدهای `openclaw.json` ‏(`secrets configure` + `secrets apply` + `secrets audit`)

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
- `channels.googlechat.serviceAccount` از طریق `serviceAccountRef` هم‌سطح (استثنای سازگاری)
- `channels.googlechat.accounts.*.serviceAccount` از طریق `serviceAccountRef` هم‌سطح (استثنای سازگاری)

### مقصدهای `auth-profiles.json` ‏(`secrets configure` + `secrets apply` + `secrets audit`)

- `profiles.*.keyRef` ‏(`type: "api_key"`؛ هنگام `auth.profiles.<id>.mode = "oauth"` پشتیبانی نمی‌شود)
- `profiles.*.tokenRef` ‏(`type: "token"`؛ هنگام `auth.profiles.<id>.mode = "oauth"` پشتیبانی نمی‌شود)

[//]: # "secretref-supported-list-end"

یادداشت‌ها:

- مقصدهای طرح پروفایل احراز هویت به `agentId` نیاز دارند؛ ورودی‌های طرح، `profiles.*.key` / `profiles.*.token` را هدف می‌گیرند و ارجاع‌های هم‌سطح (`keyRef` / `tokenRef`) را می‌نویسند. ارجاع‌های پروفایل احراز هویت در تفکیک زمان اجرا و پوشش ممیزی گنجانده شده‌اند.
- در `openclaw.json`، ‏SecretRefها باید از اشیای ساخت‌یافته‌ای مانند `{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}` استفاده کنند. رشته‌های نشانگر قدیمی `secretref-env:<ENV_VAR>` در مسیرهای اعتبارنامه SecretRef رد می‌شوند؛ برای مهاجرت نشانگرهای معتبر، `openclaw doctor --fix` را اجرا کنید.
- محافظ خط‌مشی OAuth: ‏`auth.profiles.<id>.mode = "oauth"` را نمی‌توان برای آن پروفایل با ورودی‌های SecretRef ترکیب کرد. هنگام نقض این خط‌مشی، راه‌اندازی/بارگذاری مجدد و تفکیک پروفایل احراز هویت فوراً با خطا متوقف می‌شوند.
- برای ارائه‌دهندگان مدلِ مدیریت‌شده با SecretRef، ورودی‌های تولیدشده `agents/*/agent/models.json` نشانگرهای غیرمحرمانه را (نه مقادیر تفکیک‌شده راز) برای سطوح `apiKey`/هدر نگه می‌دارند. ماندگاری نشانگرها مبتنی بر مرجعیت منبع است: OpenClaw نشانگرها را از اسنپ‌شات پیکربندی منبع فعال (پیش از تفکیک) می‌نویسد، نه از مقادیر تفکیک‌شده راز در زمان اجرا.
- راه‌اندازی سرد Gateway می‌تواند شکست‌های قابل‌تلاش‌مجدد در تفکیک را برای مالکان نگاشت‌شده و غیر Gateway ایزوله کند. کلاس‌های نگاشت‌شده فعلی شامل ارائه‌دهندگان مدل و Skills، ارائه‌دهندگان رسانه/TTS/cron، پروفایل‌های احراز هویت واجد شرایط، حافظه هر عامل، SSH سندباکس، حساب‌های کانال و مسیرهای Plugin اعلام‌شده در مانیفست هستند. راه‌اندازی، ارجاع‌های صریح هر مالک ناموفق را در اسنپ‌شات زمان اجرا نگه می‌دارد، مالک را از طریق وضعیت و doctor گزارش می‌کند و درخواست‌های آن مالک را بدون امتحان اعتبارنامه‌های دارای تقدم پایین‌تر رد می‌کند. بارگذاری مجدد و پیش‌بررسی نوشتن پیکربندی از همان خط‌مشی آگاه از مالک استفاده می‌کنند: مالکان سالم تازه‌سازی می‌شوند؛ یک مالک واجد شرایطِ ناموفق فقط زمانی منقضی باقی می‌ماند که شناسه‌های ارجاع، تعریف‌های ارائه‌دهنده و قرارداد کامل و غیرمحرمانه مالک آن بدون تغییر باشند؛ شکست جدید یا تغییریافته سرد می‌شود. احراز هویت ورودی Gateway، ارجاع‌ها یا مقادیر نامعتبر از نظر ساختاری، مالکان با خط‌مشی شکست‌بسته و مالکان فعلاً نگاشت‌نشده همچنان سخت‌گیرانه باقی می‌مانند.
- برای جست‌وجوی وب: در حالت ارائه‌دهنده صریح (`tools.web.search.provider` تنظیم شده)، فقط کلید ارائه‌دهنده انتخاب‌شده فعال است. در حالت خودکار (`tools.web.search.provider` تنظیم نشده)، فقط نخستین کلید ارائه‌دهنده‌ای که طبق تقدم تفکیک می‌شود فعال است و ارجاع‌های ارائه‌دهندگان انتخاب‌نشده تا زمان انتخاب، غیرفعال تلقی می‌شوند. اعتبارنامه‌های ارائه‌دهنده از `plugins.entries.<plugin>.config.webSearch.*` استفاده می‌کنند.
- ‏Slack ‏`identity: "user"` از `channels.slack.userToken` همراه با `channels.slack.appToken` برای Socket Mode یا `channels.slack.signingSecret` برای حالت HTTP استفاده می‌کند. همین جفت‌سازی در `channels.slack.accounts.*` نیز اعمال می‌شود؛ برای این هویت به توکن ربات نیازی نیست.

## اعتبارنامه‌های پشتیبانی‌نشده

این اعتبارنامه‌ها در دسته‌هایی قرار می‌گیرند که ایجادشده، چرخشی، حامل نشست یا ماندگار در OAuth هستند و با تفکیک خارجی و فقط‌خواندنی SecretRef سازگار نیستند:

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

## مرتبط

- [مدیریت رازها](/fa/gateway/secrets)
- [معناشناسی اعتبارنامه احراز هویت](/fa/auth-credential-semantics)
