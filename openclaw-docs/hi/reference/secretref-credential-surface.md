---
read_when:
    - SecretRef क्रेडेंशियल कवरेज का सत्यापन
    - यह ऑडिट करना कि कोई क्रेडेंशियल `secrets configure` या `secrets apply` के लिए पात्र है या नहीं
    - यह सत्यापित करना कि कोई क्रेडेंशियल समर्थित दायरे से बाहर क्यों है
summary: SecretRef क्रेडेंशियल के लिए प्रामाणिक समर्थित बनाम असमर्थित सतह
title: SecretRef क्रेडेंशियल सतह
x-i18n:
    generated_at: "2026-07-27T20:32:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbb1ad6c5045780e5ca8d9c20f2a0e86425317e86ef9aaa59957a2094344dd0d
    source_path: reference/secretref-credential-surface.md
    workflow: 16
---

यह पृष्ठ प्रामाणिक SecretRef क्रेडेंशियल सतह को परिभाषित करता है: कौन-से क्रेडेंशियल फ़ील्ड कच्चे सीक्रेट मान के बजाय `SecretRef` (env/file/exec-समर्थित संदर्भ) स्वीकार करते हैं।

दायरा:

- दायरे में: केवल उपयोगकर्ता द्वारा दिए गए वे क्रेडेंशियल, जिन्हें OpenClaw जारी या रोटेट नहीं करता।
- दायरे से बाहर: रनटाइम द्वारा जारी या रोटेट किए जाने वाले क्रेडेंशियल, OAuth रीफ़्रेश सामग्री और सत्र-जैसी कलाकृतियाँ।

नीचे दी गई सूचियाँ स्रोत लक्ष्य रजिस्ट्री से जनरेट होती हैं और CI में `docs/reference/secretref-user-supplied-credentials-matrix.json` के विरुद्ध जाँची जाती हैं; प्रविष्टियों को हाथ से संपादित न करें।

## समर्थित क्रेडेंशियल

### `openclaw.json` लक्ष्य (`secrets configure` + `secrets apply` + `secrets audit`)

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
- `channels.googlechat.serviceAccount` सहोदर `serviceAccountRef` के माध्यम से (संगतता अपवाद)
- `channels.googlechat.accounts.*.serviceAccount` सहोदर `serviceAccountRef` के माध्यम से (संगतता अपवाद)

### `auth-profiles.json` लक्ष्य (`secrets configure` + `secrets apply` + `secrets audit`)

- `profiles.*.keyRef` (`type: "api_key"`; `auth.profiles.<id>.mode = "oauth"` होने पर असमर्थित)
- `profiles.*.tokenRef` (`type: "token"`; `auth.profiles.<id>.mode = "oauth"` होने पर असमर्थित)

[//]: # "secretref-supported-list-end"

टिप्पणियाँ:

- ऑथ-प्रोफ़ाइल योजना लक्ष्यों के लिए `agentId` आवश्यक है; योजना प्रविष्टियाँ `profiles.*.key` / `profiles.*.token` को लक्षित करती हैं और सहोदर संदर्भ (`keyRef` / `tokenRef`) लिखती हैं। ऑथ-प्रोफ़ाइल संदर्भ रनटाइम रिज़ॉल्यूशन और ऑडिट कवरेज में शामिल हैं।
- `openclaw.json` में, SecretRefs को `{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}` जैसी संरचित ऑब्जेक्ट का उपयोग करना आवश्यक है। पुराने `secretref-env:<ENV_VAR>` मार्कर स्ट्रिंग SecretRef क्रेडेंशियल पथों पर अस्वीकार कर दिए जाते हैं; मान्य मार्करों को माइग्रेट करने के लिए `openclaw doctor --fix` चलाएँ।
- OAuth नीति सुरक्षा: उस प्रोफ़ाइल के लिए `auth.profiles.<id>.mode = "oauth"` को SecretRef इनपुट के साथ संयोजित नहीं किया जा सकता। इस नीति का उल्लंघन होने पर स्टार्टअप/रीलोड और ऑथ-प्रोफ़ाइल रिज़ॉल्यूशन तुरंत विफल हो जाते हैं।
- SecretRef-प्रबंधित मॉडल प्रदाताओं के लिए, जनरेट की गई `agents/*/agent/models.json` प्रविष्टियाँ `apiKey`/हेडर सतहों हेतु गैर-सीक्रेट मार्कर (रिज़ॉल्व किए गए सीक्रेट मान नहीं) बनाए रखती हैं। मार्कर स्थायित्व स्रोत-प्रामाणिक है: OpenClaw मार्करों को सक्रिय स्रोत कॉन्फ़िगरेशन स्नैपशॉट (रिज़ॉल्यूशन से पहले) से लिखता है, न कि रिज़ॉल्व किए गए रनटाइम सीक्रेट मानों से।
- Gateway का कोल्ड स्टार्टअप मैप किए गए, गैर-Gateway स्वामियों के लिए पुनः प्रयास योग्य रिज़ॉल्यूशन विफलताओं को अलग कर सकता है। वर्तमान मैप की गई श्रेणियों में मॉडल प्रदाता और स्किल, मीडिया/TTS/cron प्रदाता, पात्र ऑथ प्रोफ़ाइल, प्रति-एजेंट मेमोरी, सैंडबॉक्स SSH, चैनल खाते और मैनिफ़ेस्ट में घोषित Plugin रूट शामिल हैं। स्टार्टअप प्रत्येक विफल स्वामी के स्पष्ट संदर्भों को रनटाइम स्नैपशॉट में रखता है, स्थिति और डॉक्टर के माध्यम से स्वामी की रिपोर्ट करता है, और निम्न-प्राथमिकता वाले क्रेडेंशियल आज़माए बिना उस स्वामी के अनुरोधों को अस्वीकार करता है। रीलोड और कॉन्फ़िगरेशन-लेखन प्रीफ़्लाइट समान स्वामी-सजग नीति का उपयोग करते हैं: स्वस्थ स्वामी रीफ़्रेश होते हैं; कोई पात्र विफल स्वामी केवल तभी पुरानी स्थिति में रहता है, जब उसकी संदर्भ पहचानें, प्रदाता परिभाषाएँ और पूर्ण गैर-सीक्रेट स्वामी अनुबंध अपरिवर्तित हों; नई या परिवर्तित विफलता कोल्ड हो जाती है। Gateway इनग्रेस प्रमाणीकरण, संरचनात्मक रूप से अमान्य संदर्भ या मान, विफलता पर बंद होने वाले स्वामी और वर्तमान में मैप न किए गए स्वामी सख्त बने रहते हैं।
- वेब खोज के लिए: स्पष्ट प्रदाता मोड (`tools.web.search.provider` सेट) में केवल चयनित प्रदाता कुंजी सक्रिय होती है। स्वचालित मोड (`tools.web.search.provider` अनसेट) में केवल प्राथमिकता के अनुसार रिज़ॉल्व होने वाली पहली प्रदाता कुंजी सक्रिय होती है, और अचयनित प्रदाता संदर्भों को चुने जाने तक निष्क्रिय माना जाता है। प्रदाता क्रेडेंशियल `plugins.entries.<plugin>.config.webSearch.*` का उपयोग करते हैं।
- Slack `identity: "user"`, Socket Mode के लिए `channels.slack.appToken` या HTTP मोड के लिए `channels.slack.signingSecret` के साथ `channels.slack.userToken` का उपयोग करता है। यही युग्मन `channels.slack.accounts.*` के अंतर्गत भी लागू होता है; इस पहचान के लिए किसी बॉट टोकन की आवश्यकता नहीं है।

## असमर्थित क्रेडेंशियल

ये क्रेडेंशियल जारी किए जाने वाले, रोटेट किए जाने वाले, सत्र-धारी या OAuth-स्थायी श्रेणियों के हैं, जो केवल-पठन बाहरी SecretRef रिज़ॉल्यूशन के अनुकूल नहीं हैं:

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

## संबंधित

- [सीक्रेट प्रबंधन](/hi/gateway/secrets)
- [ऑथ क्रेडेंशियल अर्थविज्ञान](/hi/auth-credential-semantics)
