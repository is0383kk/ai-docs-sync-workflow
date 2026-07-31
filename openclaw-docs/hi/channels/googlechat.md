---
read_when:
    - Google Chat चैनल सुविधाओं पर कार्य करना
summary: Google Chat ऐप की समर्थन स्थिति, क्षमताएँ और कॉन्फ़िगरेशन
title: Google Chat
x-i18n:
    generated_at: "2026-07-27T20:26:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d3fb96564294b57040327bb21ab7331bf8412eb04f879a9c7ea1018ba2bddab
    source_path: channels/googlechat.md
    workflow: 16
---

Google Chat आधिकारिक `@openclaw/googlechat` plugin के रूप में चलता है: Google Chat API webhooks के माध्यम से DMs और स्पेस (केवल HTTP endpoint, कोई Pub/Sub नहीं)।

## इंस्टॉल करें

```bash
openclaw plugins install @openclaw/googlechat
```

स्थानीय चेकआउट (git repo से चलाते समय):

```bash
openclaw plugins install ./path/to/local/googlechat-plugin
```

## त्वरित सेटअप (शुरुआती उपयोगकर्ताओं के लिए)

1. एक Google Cloud प्रोजेक्ट बनाएँ और **Google Chat API** सक्षम करें।
   - यहाँ जाएँ: [Google Chat API Credentials](https://console.cloud.google.com/apis/api/chat.googleapis.com/credentials)
   - यदि API पहले से सक्षम नहीं है, तो इसे सक्षम करें।
2. एक **Service Account** बनाएँ:
   - **Create Credentials** > **Service Account** दबाएँ।
   - इसे अपनी पसंद का कोई भी नाम दें (उदाहरण के लिए, `openclaw-chat`)।
   - अनुमतियाँ और प्रिंसिपल खाली छोड़ें (**Continue**, फिर **Done**)।
3. **JSON key** बनाएँ और डाउनलोड करें:
   - नए सर्विस अकाउंट पर क्लिक करें > **Keys** टैब > **Add Key** > **Create new key** > **JSON** > **Create**।
4. डाउनलोड की गई JSON फ़ाइल को अपने Gateway होस्ट पर संग्रहीत करें (उदाहरण के लिए, `~/.openclaw/googlechat-service-account.json`)।
5. [Google Cloud Console Chat Configuration](https://console.cloud.google.com/apis/api/chat.googleapis.com/hangouts-chat) में एक Google Chat ऐप बनाएँ:
   - **Application info** (ऐप का नाम, अवतार URL, विवरण) भरें।
   - **Interactive features** सक्षम करें।
   - **Functionality** के अंतर्गत, **Join spaces and group conversations** चुनें।
   - **Connection settings** के अंतर्गत, **HTTP endpoint URL** चुनें।
   - **Triggers** के अंतर्गत, **Use a common HTTP endpoint URL for all triggers** चुनें और इसे अपने सार्वजनिक Gateway URL के बाद `/googlechat` लगाकर सेट करें ([सार्वजनिक URL](#public-url-webhook-only) देखें)।
   - **Visibility** के अंतर्गत, **Make this Chat app available to specific people and groups in `<Your Domain>`** चुनें और अपना ईमेल पता दर्ज करें।
   - **Save** पर क्लिक करें।
6. ऐप की स्थिति सक्षम करें: पृष्ठ रीफ़्रेश करें, **App status** ढूँढें, इसे **Live - available to users** पर सेट करें और फिर से **Save** करें।
7. OpenClaw को सर्विस अकाउंट और Webhook ऑडियंस के साथ कॉन्फ़िगर करें (यह Chat ऐप कॉन्फ़िगरेशन से मेल खाना चाहिए):
   - पर्यावरण चर: `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE=/path/to/service-account.json` (केवल डिफ़ॉल्ट अकाउंट), या
   - कॉन्फ़िगरेशन: [कॉन्फ़िगरेशन की मुख्य बातें](#config-highlights) देखें। `openclaw channels add --channel googlechat` में `--audience-type`, `--audience`, `--webhook-path`, और `--webhook-url` भी स्वीकार किए जाते हैं।
8. Gateway शुरू करें। Google Chat आपके Webhook पथ (डिफ़ॉल्ट `/googlechat`) पर POST करेगा।

## Google Chat में जोड़ें

Gateway चलने और आपका ईमेल दृश्यता सूची में होने के बाद:

1. [Google Chat](https://chat.google.com/) पर जाएँ।
2. **Direct Messages** के बगल में स्थित **+** (प्लस) आइकन पर क्लिक करें।
3. Google Cloud Console में कॉन्फ़िगर किया गया **App name** खोजें।
   - बॉट Marketplace की ब्राउज़ सूची में _नहीं_ दिखाई देता, क्योंकि यह एक निजी ऐप है; इसे नाम से खोजें।
4. बॉट चुनें, **Add** या **Chat** पर क्लिक करें और संदेश भेजें।

## सार्वजनिक URL (केवल Webhook)

Google Chat webhooks के लिए एक सार्वजनिक HTTPS endpoint आवश्यक है। सुरक्षा के लिए, इंटरनेट पर **केवल `/googlechat` पथ** उपलब्ध कराएँ और OpenClaw डैशबोर्ड तथा अन्य endpoints को निजी रखें।

### विकल्प A: Tailscale Funnel (अनुशंसित)

निजी डैशबोर्ड के लिए Tailscale Serve और सार्वजनिक Webhook पथ के लिए Funnel का उपयोग करें।

1. जाँचें कि आपका Gateway किस पते से बँधा है:

   ```bash
   ss -tlnp | grep 18789
   ```

   IP नोट करें (उदाहरण के लिए, `127.0.0.1`, `0.0.0.0`, या कोई Tailscale `100.x.x.x` पता)।

2. डैशबोर्ड को केवल tailnet पर उपलब्ध कराएँ (पोर्ट 8443):

   ```bash
   # यदि localhost (127.0.0.1 या 0.0.0.0) से बँधा है:
   tailscale serve --bg --https 8443 http://127.0.0.1:18789

   # यदि केवल Tailscale IP से बँधा है:
   tailscale serve --bg --https 8443 http://100.x.x.x:18789
   ```

3. केवल Webhook पथ को सार्वजनिक रूप से उपलब्ध कराएँ:

   ```bash
   # यदि localhost (127.0.0.1 या 0.0.0.0) से बँधा है:
   tailscale funnel --bg --set-path /googlechat http://127.0.0.1:18789/googlechat

   # यदि केवल Tailscale IP से बँधा है:
   tailscale funnel --bg --set-path /googlechat http://100.x.x.x:18789/googlechat
   ```

4. यदि संकेत दिया जाए, तो इस Node के लिए Funnel सक्षम करने हेतु आउटपुट में दिखाए गए प्राधिकरण URL पर जाएँ।

5. सत्यापित करें:

   ```bash
   tailscale serve status
   tailscale funnel status
   ```

आपका सार्वजनिक Webhook URL `https://<node-name>.<tailnet>.ts.net/googlechat` है; डैशबोर्ड `https://<node-name>.<tailnet>.ts.net:8443/` पर केवल tailnet के लिए उपलब्ध रहता है। Google Chat ऐप कॉन्फ़िगरेशन में सार्वजनिक URL (`:8443` के बिना) का उपयोग करें।

> नोट: यह कॉन्फ़िगरेशन रीबूट के बाद भी बना रहता है। इसे बाद में `tailscale funnel reset` और `tailscale serve reset` से हटाएँ।

### विकल्प B: रिवर्स प्रॉक्सी (Caddy)

केवल Webhook पथ को प्रॉक्सी करें:

```caddy
your-domain.com {
    reverse_proxy /googlechat* localhost:18789
}
```

`your-domain.com/` के अनुरोधों को अनदेखा किया जाता है या 404 मिलता है, जबकि `your-domain.com/googlechat` OpenClaw पर रूट होता है।

### विकल्प C: Cloudflare Tunnel

केवल Webhook पथ को रूट करने के लिए टनल इनग्रेस नियम कॉन्फ़िगर करें:

- **Path**: `/googlechat` -> `http://localhost:18789/googlechat`
- **Default rule**: HTTP 404 (Not Found)

## यह कैसे काम करता है

1. Google Chat Gateway Webhook पथ पर JSON POST करता है (केवल POST, JSON सामग्री प्रकार आवश्यक, प्रति-IP दर सीमित)।
2. OpenClaw प्रत्येक अनुरोध को भेजने से पहले प्रमाणित करता है:
   - Chat ऐप इवेंट में `Authorization: Bearer <token>` होता है; पूरा बॉडी पार्स करने से पहले टोकन सत्यापित किया जाता है।
   - Google Workspace ऐड-ऑन इवेंट में टोकन बॉडी (`authorizationEventObject.systemIdToken`) में होता है और सत्यापन से पहले इसे अधिक कड़े पूर्व-प्रमाणीकरण बजट (16 KB, 3 s) के अंतर्गत पढ़ा जाता है।
3. टोकन की जाँच `audienceType` + `audience` के विरुद्ध की जाती है:
   - `audienceType: "app-url"` → ऑडियंस आपका HTTPS Webhook URL है।
   - `audienceType: "project-number"` → ऑडियंस Cloud प्रोजेक्ट नंबर है।
   - `app-url` के अंतर्गत ऐड-ऑन टोकन के लिए अतिरिक्त रूप से `appPrincipal` को ऐप की संख्यात्मक OAuth 2.0 क्लाइंट ID (21 अंक, ईमेल नहीं) पर सेट करना आवश्यक है; अन्यथा लॉग चेतावनी के साथ सत्यापन विफल हो जाता है।
4. संदेश स्पेस के अनुसार रूट होते हैं:
   - स्पेस को प्रति-स्पेस सत्र `agent:<agentId>:googlechat:group:<spaceId>` मिलते हैं; उत्तर संदेश थ्रेड में जाते हैं।
   - डिफ़ॉल्ट रूप से DMs एजेंट के मुख्य सत्र में समाहित हो जाते हैं; प्रति-पीयर DM सत्रों के लिए `session.dmScope` सेट करें ([सत्र](/hi/concepts/session) देखें)।
5. DM एक्सेस डिफ़ॉल्ट रूप से पेयरिंग है। अज्ञात प्रेषकों को एक पेयरिंग कोड मिलता है; इससे स्वीकृत करें:
   - `openclaw pairing approve googlechat <code>`
6. ग्रुप स्पेस में डिफ़ॉल्ट रूप से @-उल्लेख आवश्यक है। उल्लेखों का पता ऐप को लक्षित करने वाली Chat `USER_MENTION` टिप्पणियों से लगाया जाता है; यदि पहचान के लिए ऐप के उपयोगकर्ता संसाधन नाम की आवश्यकता हो, तो `botUser` (उदाहरण के लिए, `users/1234567890`) सेट करें।
7. जब Google Chat से कोई exec या plugin अनुमोदन शुरू होता है और एक स्थिर `users/<id>` अनुमोदक कॉन्फ़िगर किया गया हो, तो OpenClaw मूल स्पेस या थ्रेड में एक नेटिव अनुमोदन कार्ड (`cardsV2`) पोस्ट करता है। कार्ड बटन में अपारदर्शी कॉलबैक टोकन होते हैं; मैन्युअल `/approve <id> <decision>` प्रॉम्प्ट केवल तब दिखाई देता है, जब नेटिव डिलीवरी उपलब्ध नहीं होती।

### इनबाउंड स्थायित्व

अनुरोध प्रमाणीकरण के बाद, OpenClaw ऐड-ऑन प्राधिकरण ऑब्जेक्ट को स्टोरेज से हटा देता है और `200` लौटाने से पहले Google Chat `MESSAGE` इवेंट को स्थायी रूप से कतारबद्ध करता है। स्थायित्व विफल होने पर `503` लौटाया जाता है, जिससे Google Chat किसी खो सकने वाले इवेंट को स्वीकार करने के बजाय फिर से प्रयास कर सकता है।

लंबित या पुनः प्रयास योग्य संदेश Gateway पुनरारंभ के बाद भी बने रहते हैं, प्रति स्पेस क्रमबद्ध रहते हैं और सक्रिय या सुरक्षित पूर्णता रिकॉर्ड मौजूद रहने तक डुप्लिकेट कतार प्रविष्टियों को रोकने के लिए Google Chat संदेश संसाधन नाम का उपयोग करते हैं। गैर-संदेश क्रियाएँ अपना मौजूदा अलग Webhook पथ बनाए रखती हैं और उन्हें स्थायी कतार की यह गारंटी नहीं मिलती। कतार-से-एजेंट सीमा पर डिलीवरी कम-से-कम-एक-बार बनी रहती है, इसलिए हैंडऑफ़ के दौरान क्रैश होने पर कोई टर्न फिर से चल सकता है।

## लक्ष्य

डिलीवरी और अनुमतिसूचियों के लिए इन पहचानकर्ताओं का उपयोग करें:

- सीधे संदेश: `users/<userId>` (अनुशंसित)।
- स्पेस: `spaces/<spaceId>`।
- रॉ ईमेल `name@example.com` परिवर्तनशील है और अनुमतिसूची मिलान के लिए केवल तब उपयोग होता है, जब `channels.googlechat.dangerouslyAllowNameMatching: true`।
- बहिष्कृत: `users/<email>` को उपयोगकर्ता ID माना जाता है, ईमेल अनुमतिसूची प्रविष्टि नहीं।
- उपसर्ग `googlechat:`, `google-chat:`, और `gchat:` स्वीकार करके हटा दिए जाते हैं।

## कॉन्फ़िगरेशन की मुख्य बातें

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      // या serviceAccountRef: { source: "file", provider: "filemain", id: "/channels/googlechat/serviceAccount" }
      audienceType: "app-url",
      audience: "https://gateway.example.com/googlechat",
      appPrincipal: "123456789012345678901", // केवल ऐड-ऑन सत्यापन; संख्यात्मक OAuth क्लाइंट ID
      webhookPath: "/googlechat",
      botUser: "users/1234567890", // वैकल्पिक; उल्लेख पहचान में सहायता करता है
      allowBots: false,
      dmPolicy: "pairing",
      allowFrom: ["users/1234567890"],
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": {
          enabled: true,
          requireMention: true,
          users: ["users/1234567890"],
          systemPrompt: "केवल संक्षिप्त उत्तर।",
        },
      },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

नोट्स:

- सर्विस अकाउंट क्रेडेंशियल: `serviceAccountFile` (पथ), `serviceAccount` (इनलाइन JSON स्ट्रिंग या ऑब्जेक्ट), या `serviceAccountRef` (पर्यावरण चर/फ़ाइल SecretRef)। पर्यावरण चर `GOOGLE_CHAT_SERVICE_ACCOUNT` (इनलाइन JSON) और `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE` (पथ) केवल डिफ़ॉल्ट अकाउंट पर लागू होते हैं। बहु-अकाउंट सेटअप समान कुंजियों के साथ `channels.googlechat.accounts.<id>` का उपयोग करते हैं, जिसमें प्रति-अकाउंट `serviceAccountRef` भी शामिल है।
- `webhookPath` सेट न होने पर डिफ़ॉल्ट Webhook पथ `/googlechat` होता है; इसके बजाय `webhookUrl` पथ प्रदान कर सकता है।
- ग्रुप कुंजियाँ स्थिर स्पेस ID (`spaces/<spaceId>`) होनी चाहिए। प्रदर्शन-नाम कुंजियाँ बहिष्कृत हैं और उसी रूप में लॉग की जाती हैं।
- `dangerouslyAllowNameMatching` अनुमतिसूचियों के लिए परिवर्तनशील ईमेल प्रिंसिपल मिलान को फिर से सक्षम करता है (आपातकालीन संगतता मोड); doctor ईमेल प्रविष्टियों के बारे में चेतावनी देता है।
- Google Chat प्रतिक्रिया क्रियाएँ उपलब्ध नहीं कराई जातीं। plugin सर्विस-अकाउंट प्रमाणीकरण का उपयोग करता है, जबकि Google Chat प्रतिक्रिया endpoints के लिए उपयोगकर्ता प्रमाणीकरण आवश्यक है। मौजूदा `actions.reactions` कॉन्फ़िगरेशन संगतता के लिए स्वीकार किया जाता है, लेकिन इसका कोई प्रभाव नहीं पड़ता।
- नेटिव अनुमोदन कार्ड प्रतिक्रिया इवेंट के बजाय Google Chat `cardsV2` बटन क्लिक का उपयोग करते हैं। अनुमोदक `allowFrom` या `defaultTo` से आते हैं और वे स्थिर संख्यात्मक `users/<id>` मान होने चाहिए।
- संदेश क्रियाएँ केवल टेक्स्ट `send` उपलब्ध कराती हैं। Google Chat अटैचमेंट अपलोड के लिए उपयोगकर्ता प्रमाणीकरण आवश्यक है, जबकि यह plugin सर्विस-अकाउंट प्रमाणीकरण का उपयोग करता है, इसलिए आउटबाउंड फ़ाइल अपलोड उपलब्ध नहीं कराया जाता।
- `typingIndicator`: `message` (डिफ़ॉल्ट) एक `_<Bot> is typing..._` प्लेसहोल्डर पोस्ट करता है और उसे पहले उत्तर में संपादित करता है; `none` इसे अक्षम करता है; `reaction` के लिए उपयोगकर्ता OAuth आवश्यक है और फ़िलहाल सर्विस-अकाउंट प्रमाणीकरण के अंतर्गत लॉग की गई त्रुटि के साथ `message` पर वापस जाता है।
- इनबाउंड अटैचमेंट (प्रति संदेश पहला अटैचमेंट) Chat API के माध्यम से मीडिया पाइपलाइन में डाउनलोड किए जाते हैं और `mediaMaxMb` (डिफ़ॉल्ट 20) द्वारा सीमित होते हैं।
- बॉट द्वारा लिखे गए संदेश डिफ़ॉल्ट रूप से अनदेखे किए जाते हैं। `allowBots: true` के साथ, स्वीकार किए गए बॉट संदेश साझा [बॉट लूप सुरक्षा](/hi/channels/bot-loop-protection) का उपयोग करते हैं: `channels.defaults.botLoopProtection` कॉन्फ़िगर करें, फिर `channels.googlechat.botLoopProtection` या `channels.googlechat.groups.<space>.botLoopProtection` से ओवरराइड करें।

सीक्रेट के संदर्भ विवरण: [सीक्रेट प्रबंधन](/hi/gateway/secrets)।

## समस्या निवारण

### 405 Method Not Allowed

यदि Google Cloud Logs Explorer में इस तरह की त्रुटियाँ दिखाई देती हैं:

```text
status code: 405, reason phrase: HTTP error response: HTTP/1.1 405 Method Not Allowed
```

Webhook हैंडलर पंजीकृत नहीं है। सामान्य कारण:

1. **चैनल कॉन्फ़िगर नहीं किया गया है**: `channels.googlechat` अनुभाग मौजूद नहीं है। इससे सत्यापित करें:

   ```bash
   openclaw config get channels.googlechat
   ```

   यदि यह "Config path not found" लौटाता है, तो कॉन्फ़िगरेशन जोड़ें ([कॉन्फ़िगरेशन की मुख्य बातें](#config-highlights) देखें)।

2. **Plugin सक्षम नहीं है**: Plugin की स्थिति जाँचें:

   ```bash
   openclaw plugins list | grep googlechat
   ```

   यदि यह "disabled" दिखाता है, तो अपने कॉन्फ़िगरेशन में `plugins.entries.googlechat.enabled: true` जोड़ें।

3. कॉन्फ़िगरेशन में बदलाव के बाद **Gateway को पुनः आरंभ नहीं किया गया है**:

   ```bash
   openclaw gateway restart
   ```

सत्यापित करें कि चैनल चल रहा है:

```bash
openclaw channels status
# यह दिखाई देना चाहिए: Google Chat default: enabled, configured, ...
```

### अन्य समस्याएँ

- `openclaw channels status --probe` प्रमाणीकरण त्रुटियाँ और अनुपस्थित ऑडियंस कॉन्फ़िगरेशन दिखाता है (`audience` और `audienceType` दोनों आवश्यक हैं)।
- यदि कोई संदेश नहीं आता है, तो Chat ऐप के Webhook URL और ट्रिगर कॉन्फ़िगरेशन की पुष्टि करें।
- यदि मेंशन गेटिंग उत्तरों को अवरुद्ध करती है, तो `botUser` को ऐप के उपयोगकर्ता संसाधन नाम पर सेट करें और `requireMention` जाँचें।
- परीक्षण संदेश भेजते समय `openclaw logs --follow` यह दिखाता है कि अनुरोध Gateway तक पहुँच रहे हैं या नहीं।

## संबंधित

- [चैनलों का अवलोकन](/hi/channels) — सभी समर्थित चैनल
- [चैनल रूटिंग](/hi/channels/channel-routing) — संदेशों के लिए सत्र रूटिंग
- [Gateway कॉन्फ़िगरेशन](/hi/gateway/configuration)
- [समूह](/hi/channels/groups) — समूह चैट का व्यवहार और मेंशन गेटिंग
- [पेयरिंग](/hi/channels/pairing) — DM प्रमाणीकरण और पेयरिंग प्रवाह
- [सुरक्षा](/hi/gateway/security) — पहुँच मॉडल और सुदृढ़ीकरण
