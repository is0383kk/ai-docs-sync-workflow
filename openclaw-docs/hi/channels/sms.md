---
read_when:
    - आप Twilio के माध्यम से OpenClaw को SMS से कनेक्ट करना चाहते हैं
    - आपको SMS Webhook या अनुमति-सूची सेटअप की आवश्यकता है
summary: Twilio SMS चैनल सेटअप, अभिगम नियंत्रण और Webhook कॉन्फ़िगरेशन
title: एसएमएस
x-i18n:
    generated_at: "2026-07-27T17:45:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 99a76b2f2d66858f8eb699939084104e620af9bc024053bbe1c1d7350530bff0
    source_path: channels/sms.md
    workflow: 16
---

OpenClaw, Twilio फ़ोन नंबर या Messaging Service के माध्यम से SMS प्राप्त और भेजता है। Gateway एक इनबाउंड Webhook रूट (डिफ़ॉल्ट `/webhooks/sms`) पंजीकृत करता है, डिफ़ॉल्ट रूप से Twilio अनुरोध हस्ताक्षरों को सत्यापित करता है और उत्तरों को Twilio के Messages API के माध्यम से वापस भेजता है।

स्थिति: आधिकारिक Plugin, अलग से इंस्टॉल किया जाता है। केवल टेक्स्ट: कोई MMS/मीडिया नहीं, केवल डायरेक्ट मैसेज।

<CardGroup cols={3}>
  <Card title="पेयरिंग" icon="link" href="/hi/channels/pairing">
    SMS के लिए डिफ़ॉल्ट DM नीति पेयरिंग है।
  </Card>
  <Card title="Gateway सुरक्षा" icon="shield" href="/hi/gateway/security">
    Webhook एक्सपोज़र और प्रेषक एक्सेस नियंत्रणों की समीक्षा करें।
  </Card>
  <Card title="चैनल समस्या-निवारण" icon="wrench" href="/hi/channels/troubleshooting">
    विभिन्न चैनलों के लिए निदान और सुधार प्लेबुक।
  </Card>
</CardGroup>

## शुरू करने से पहले

आपको चाहिए:

- आधिकारिक SMS Plugin, जिसे `openclaw plugins install @openclaw/sms` से इंस्टॉल किया गया हो।
- SMS-सक्षम फ़ोन नंबर या Twilio Messaging Service वाला Twilio खाता।
- Twilio Account SID और Auth Token।
- आपके OpenClaw Gateway तक पहुँचने वाला सार्वजनिक HTTPS URL।
- प्रेषक नीति का चयन: निजी उपयोग के लिए `pairing` (डिफ़ॉल्ट), पहले से स्वीकृत फ़ोन नंबरों के लिए `allowlist`, या केवल जानबूझकर सार्वजनिक SMS एक्सेस के लिए `open`।

एक Twilio नंबर SMS और [वॉइस कॉल](/hi/plugins/voice-call), दोनों के लिए काम कर सकता है, बशर्ते उसमें दोनों क्षमताएँ हों। SMS Webhook और वॉइस Webhook को Twilio में अलग-अलग कॉन्फ़िगर किया जाता है और वे अलग Gateway पथों का उपयोग करते हैं; यह पेज केवल SMS Webhook को कवर करता है।

## त्वरित सेटअप

<Steps>
  <Step title="Plugin इंस्टॉल करें">
    ```bash
    openclaw plugins install @openclaw/sms
    ```
  </Step>
  <Step title="Twilio प्रेषक बनाएँ या चुनें">
    Twilio में **Phone Numbers > Manage > Active numbers** खोलें और SMS-सक्षम नंबर चुनें। इन्हें सहेजें:

    - Account SID, उदाहरण के लिए `ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
    - Auth Token
    - प्रेषक फ़ोन नंबर, उदाहरण के लिए `+15551234567`

    यदि आप किसी निश्चित प्रेषक नंबर के बजाय Messaging Service का उपयोग करते हैं, तो Messaging Service SID सहेजें, उदाहरण के लिए `MGxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`।

  </Step>

  <Step title="SMS चैनल कॉन्फ़िगर करें">

इसे `sms.patch.json5` के रूप में सहेजें और प्लेसहोल्डर बदलें:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

इसे लागू करें:

```bash
openclaw config patch --file ./sms.patch.json5 --dry-run
openclaw config patch --file ./sms.patch.json5
```

  </Step>

  <Step title="Twilio को Gateway Webhook पर निर्देशित करें">
    Twilio फ़ोन नंबर सेटिंग में **Messaging** खोलें और **A message comes in** को इस पर सेट करें:

```text
https://gateway.example.com/webhooks/sms
```

    HTTP `POST` का उपयोग करें। डिफ़ॉल्ट स्थानीय पथ `/webhooks/sms` है; यदि आपको कोई अलग रूट चाहिए, तो `channels.sms.webhookPath` बदलें।

  </Step>

  <Step title="सटीक SMS Webhook पथ एक्सपोज़ करें">
    आपके सार्वजनिक URL को SMS पथ से Gateway प्रक्रिया (डिफ़ॉल्ट पोर्ट `18789`) तक रूट करना होगा। यदि आप स्थानीय परीक्षण के लिए Tailscale Funnel का उपयोग करते हैं, तो `/webhooks/sms` को स्पष्ट रूप से एक्सपोज़ करें:

```bash
tailscale funnel --bg --set-path /webhooks/sms http://127.0.0.1:<gateway-port>/webhooks/sms
tailscale funnel status
```

    वॉइस कॉल और SMS अलग-अलग Webhook पथों का उपयोग करते हैं। यदि एक ही Twilio नंबर दोनों को संभालता है, तो दोनों रूटों को Twilio और अपनी टनल में कॉन्फ़िगर रखें।

  </Step>

  <Step title="Gateway शुरू करें और पहले प्रेषक को स्वीकृति दें">

```bash
openclaw gateway
```

Twilio नंबर पर एक टेक्स्ट मैसेज भेजें। पहला मैसेज एक पेयरिंग अनुरोध बनाता है। इसे स्वीकृति दें:

```bash
openclaw pairing list sms
openclaw pairing approve sms <CODE>
```

    पेयरिंग कोड 1 घंटे बाद समाप्त हो जाते हैं।

  </Step>
</Steps>

## कॉन्फ़िगरेशन के उदाहरण

सभी कुंजियाँ `channels.sms` के अंतर्गत होती हैं (और प्रत्येक खाते के लिए `channels.sms.accounts.<id>` के अंतर्गत):

| कुंजी                                   | डिफ़ॉल्ट        | उद्देश्य                                                               |
| --------------------------------------- | --------------- | ------------------------------------------------------------------- |
| `enabled`                               | `true`          | चैनल/खाते को सक्षम या अक्षम करें।                              |
| `accountSid`                            | —               | Twilio Account SID (`AC...`)।                                       |
| `authToken`                             | —               | Twilio Auth Token; प्लेनटेक्स्ट स्ट्रिंग या SecretRef।                   |
| `fromNumber`                            | —               | E.164 प्रेषक नंबर।                                                |
| `messagingServiceSid`                   | —               | कोई `fromNumber` रिज़ॉल्व न होने पर उपयोग किया जाने वाला Messaging Service SID (`MG...`)। |
| `defaultTo`                             | —               | जब भेजने का प्रवाह स्पष्ट लक्ष्य छोड़ देता है, तब डिफ़ॉल्ट गंतव्य।      |
| `webhookPath`                           | `/webhooks/sms` | इनबाउंड Twilio Webhook के लिए Gateway HTTP पथ।                      |
| `publicWebhookUrl`                      | —               | Twilio में कॉन्फ़िगर किया गया सार्वजनिक URL; हस्ताक्षर सत्यापन के लिए आवश्यक। |
| `dangerouslyDisableSignatureValidation` | `false`         | `X-Twilio-Signature` जाँच छोड़ें; केवल स्थानीय टनल परीक्षण के लिए।        |
| `dmPolicy`                              | `"pairing"`     | `pairing`, `allowlist`, `open`, या `disabled`।                      |
| `allowFrom`                             | `[]`            | E.164 में अनुमत प्रेषक नंबर, या `dmPolicy: "open"` के साथ `"*"`।  |
| `textChunkLimit`                        | `1500`          | प्रत्येक आउटबाउंड SMS खंड में वर्णों की अधिकतम संख्या।                          |
| `accounts`, `defaultAccount`            | —               | बहु-खाता मैप और डिफ़ॉल्ट खाता आईडी।                           |

### कॉन्फ़िगरेशन फ़ाइल

जब आप चाहते हैं कि चैनल परिभाषा Gateway कॉन्फ़िगरेशन के साथ उपलब्ध रहे, तब कॉन्फ़िगरेशन-फ़ाइल सेटअप का उपयोग करें:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

### एनवायरनमेंट वेरिएबल

एनवायरनमेंट वेरिएबल केवल डिफ़ॉल्ट खाते पर लागू होते हैं; कॉन्फ़िगरेशन मानों को एनवायरनमेंट मानों पर प्राथमिकता मिलती है।

| वेरिएबल                                        | इससे मैप होता है                                            |
| ----------------------------------------------- | -------------------------------------------------- |
| `TWILIO_ACCOUNT_SID`                            | `accountSid`                                       |
| `TWILIO_AUTH_TOKEN`                             | `authToken`                                        |
| `TWILIO_PHONE_NUMBER` (उपनाम `TWILIO_SMS_FROM`) | `fromNumber`                                       |
| `TWILIO_MESSAGING_SERVICE_SID`                  | `messagingServiceSid`                              |
| `SMS_PUBLIC_WEBHOOK_URL`                        | `publicWebhookUrl`                                 |
| `SMS_WEBHOOK_PATH`                              | `webhookPath`                                      |
| `SMS_ALLOWED_USERS`                             | `allowFrom` (कॉमा से अलग किए गए)                      |
| `SMS_TEXT_CHUNK_LIMIT`                          | `textChunkLimit`                                   |
| `SMS_DANGEROUSLY_DISABLE_SIGNATURE_VALIDATION`  | `dangerouslyDisableSignatureValidation` (`"true"`) |

```bash
export TWILIO_ACCOUNT_SID="ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
export TWILIO_AUTH_TOKEN="<twilio-auth-token>"
export TWILIO_PHONE_NUMBER="+15551234567"
export SMS_PUBLIC_WEBHOOK_URL="https://gateway.example.com/webhooks/sms"
```

फिर कॉन्फ़िगरेशन में चैनल सक्षम करें:

```json5
{
  channels: {
    sms: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

### SecretRef Auth Token

`authToken` एक SecretRef (`source: "env" | "file" | "exec"`) हो सकता है। इसका उपयोग तब करें, जब Gateway को प्लेनटेक्स्ट कॉन्फ़िगरेशन संग्रहीत करने के बजाय OpenClaw सीक्रेट्स रनटाइम से Twilio Auth Token रिज़ॉल्व करना हो:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: { source: "env", provider: "default", id: "TWILIO_AUTH_TOKEN" },
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

संदर्भित एनवायरनमेंट वेरिएबल या सीक्रेट प्रदाता Gateway रनटाइम को दिखाई देना चाहिए। होस्ट एनवायरनमेंट वेरिएबल बदलने के बाद प्रबंधित Gateway प्रक्रियाओं को पुनः आरंभ करें।

### Messaging Service प्रेषक

जब Twilio को Messaging Service के माध्यम से प्रेषक चुनना हो, तब `fromNumber` के बजाय `messagingServiceSid` का उपयोग करें:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      messagingServiceSid: "MGxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

यदि कॉन्फ़िगरेशन और एनवायरनमेंट रिज़ॉल्यूशन के बाद `fromNumber` और `messagingServiceSid`, दोनों मौजूद हों, तो `fromNumber` का उपयोग किया जाता है।

### डिफ़ॉल्ट आउटबाउंड लक्ष्य

जब ऑटोमेशन या एजेंट द्वारा आरंभ की गई डिलीवरी के लिए किसी स्पष्ट लक्ष्य को छोड़ने वाले भेजने के प्रवाह में डिफ़ॉल्ट गंतव्य होना चाहिए, तब `defaultTo` सेट करें:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      defaultTo: "+15557654321",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
    },
  },
}
```

## एक्सेस नियंत्रण

`channels.sms.dmPolicy` सीधे SMS एक्सेस को नियंत्रित करता है:

- `pairing` (डिफ़ॉल्ट): अज्ञात प्रेषकों को पेयरिंग कोड मिलता है; `openclaw pairing approve sms <CODE>` से स्वीकृति दें।
- `allowlist`: केवल `allowFrom` में मौजूद प्रेषकों को प्रोसेस किया जाता है। खाली `allowFrom` प्रत्येक प्रेषक को अस्वीकार करता है (Gateway स्टार्टअप चेतावनी लॉग करता है)।
- `open`: कॉन्फ़िगरेशन सत्यापन के लिए `allowFrom` में `"*"` शामिल होना आवश्यक है। वाइल्डकार्ड के बिना केवल सूचीबद्ध नंबर चैट कर सकते हैं।
- `disabled`: सभी इनबाउंड DM हटा दिए जाते हैं।

`allowFrom` प्रविष्टियाँ `+15551234567` जैसे E.164 फ़ोन नंबर होनी चाहिए। `sms:` और `twilio-sms:` प्रीफ़िक्स स्वीकार किए जाते हैं और सामान्यीकृत किए जाते हैं। निजी सहायक के लिए स्पष्ट फ़ोन नंबरों के साथ `dmPolicy: "allowlist"` को प्राथमिकता दें:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "allowlist",
      allowFrom: ["+15557654321"],
    },
  },
}
```

## SMS भेजना

SMS चैनल चयनित होने पर लक्ष्य सीधे E.164 नंबर या `sms:` प्रीफ़िक्स स्वीकार करते हैं:

```bash
openclaw message send --channel sms --target sms:+15551234567 --message "hello"
```

जब चैनल चयन अप्रत्यक्ष हो, तब `twilio-sms:` प्रीफ़िक्स `sms:` सेवा प्रीफ़िक्स को अपने नियंत्रण में लिए बिना इस चैनल को चुनता है; iMessage अपने लक्ष्यों के लिए कैरियर SMS डिलीवरी चुनने हेतु उस सेवा प्रीफ़िक्स का उपयोग करता है:

```bash
openclaw message send --target twilio-sms:+15551234567 --message "hello"
```

CLI के लिए स्पष्ट `--target` आवश्यक है। `defaultTo` उन ऑटोमेशन और एजेंट द्वारा आरंभ किए गए डिलीवरी पथों के लिए है, जहाँ लक्ष्य को चैनल कॉन्फ़िगरेशन से रिज़ॉल्व किया जा सकता है।

इनबाउंड SMS वार्तालापों के लिए एजेंट के उत्तर कॉन्फ़िगर किए गए Twilio प्रेषक के माध्यम से स्वचालित रूप से प्रेषक को वापस भेजे जाते हैं।

SMS आउटपुट सादा टेक्स्ट होता है। OpenClaw मार्कडाउन हटाता है, फ़ेंस किए गए कोड ब्लॉक को समतल करता है, लिंक को `label (url)` के रूप में फिर से लिखता है, और लंबे उत्तरों को Twilio के माध्यम से भेजने से पहले अधिकतम `textChunkLimit` वर्णों (डिफ़ॉल्ट 1500) के खंडों में विभाजित करता है।

## सेटअप सत्यापित करें

Gateway शुरू होने के बाद:

1. पुष्टि करें कि Gateway लॉग में SMS Webhook रूट दिख रहा है।
2. Twilio की ओर से एक जाँच चलाएँ (यह कॉन्फ़िगर किए गए Twilio Webhook URL/विधि और हाल की इनबाउंड त्रुटियों की जाँच करती है):

```bash
openclaw channels capabilities --channel sms
openclaw channels status --channel sms --probe --json
```

3. अपने फ़ोन से Twilio नंबर पर एक SMS भेजें।
4. `openclaw pairing list sms` चलाएँ।
5. `openclaw pairing approve sms <CODE>` के साथ पेयरिंग कोड को स्वीकृत करें।
6. एक और SMS भेजें और पुष्टि करें कि एजेंट उत्तर देता है।

केवल आउटबाउंड परीक्षण के लिए, इसका उपयोग करें:

```bash
openclaw message send --channel sms --target sms:+15557654321 --message "OpenClaw SMS test"
```

### macOS iMessage/SMS से शुरू से अंत तक परीक्षण

ऐसे Mac पर जो Messages के माध्यम से कैरियर SMS भेज सकता है, आप अपने फ़ोन को छुए बिना प्रेषक पक्ष को संचालित करने के लिए `imsg` का उपयोग कर सकते हैं:

```bash
imsg send --to "+15551234567" --service sms --text "OpenClaw SMS E2E $(date -u +%Y%m%dT%H%M%SZ)" --json
openclaw pairing list sms
openclaw pairing approve sms <CODE>
imsg send --to "+15551234567" --service sms --text "reply exactly SMS pong" --json
```

पहले संदेश से एक पेयरिंग अनुरोध बनना चाहिए। दूसरे संदेश को Twilio के माध्यम से एजेंट का उत्तर मिलना चाहिए।

## Webhook सुरक्षा

डिफ़ॉल्ट रूप से, OpenClaw `publicWebhookUrl` और `authToken` का उपयोग करके `X-Twilio-Signature` को सत्यापित करता है। `publicWebhookUrl` के एंडपॉइंट भाग को Twilio में कॉन्फ़िगर किए गए URL के साथ बाइट-दर-बाइट समान रखें, जिसमें स्कीम, होस्ट, पाथ और क्वेरी स्ट्रिंग शामिल हैं। Twilio की आवश्यकता के अनुसार, OpenClaw हस्ताक्षर की गणना से Twilio के [कनेक्शन ओवरराइड](https://www.twilio.com/docs/usage/webhooks/webhooks-connection-overrides) फ़्रैगमेंट (`#...`) को बाहर रखता है।

Webhook रूट, हस्ताक्षर सत्यापन से स्वतंत्र रूप से, इन्हें भी लागू करता है:

- केवल `POST`।
- प्रति SMS अकाउंट, Webhook रूट और निर्धारित क्लाइंट पते पर प्रति मिनट 300 अनुरोधों का विफल-अनुरोध बजट। सभी अनुरोध इस बजट में गिने जाते हैं, लेकिन HTTP 429 केवल तब लागू होता है जब कोई अनुरोध बॉडी पार्सिंग, Twilio सत्यापन या AccountSid मिलान में विफल हो जाता है।
- इन जाँचों के सफल होने के बाद प्रति SMS अकाउंट, Webhook रूट और निर्धारित क्लाइंट पते पर प्रति मिनट 30 स्वीकृत कॉलबैक की प्रेषण-योग्य दर सीमा (इससे अधिक पर HTTP 429)। यदि हस्ताक्षर सत्यापन अक्षम है, तो यह 30/मिनट सीमा अप्रमाणित प्रेषण की अधिकतम सीमा है।
- क्लाइंट पतों को साझा Gateway विश्वसनीय-प्रॉक्सी नियमों के माध्यम से निर्धारित किया जाता है। यदि `gateway.trustedProxies` में वह रिवर्स प्रॉक्सी शामिल है जो Twilio कॉलबैक अग्रेषित करता है, तो OpenClaw इन सीमाओं के लिए अग्रेषित क्लाइंट पते का उपयोग करता है; अन्यथा यह सीधे सॉकेट पते का उपयोग करता है।
- पेलोड का `AccountSid`, कॉन्फ़िगर किए गए `accountSid` से मेल खाना चाहिए (अन्यथा HTTP 403)।
- दोबारा चलाए गए `MessageSid` मानों को 10 मिनट के लिए डीडुप्लिकेट किया जाता है।
- प्रत्येक SMS अकाउंट का रीप्ले कैश अधिकतम 10,000 सक्रिय संदेश SID रखता है। जब प्रत्येक स्लॉट सक्रिय हो, तब उस अकाउंट के नए Webhook HTTP 429 और `Retry-After` हेडर के साथ विफलता-सुरक्षित ढंग से अस्वीकार कर दिए जाते हैं, जब तक सबसे पुराना स्लॉट समाप्त नहीं हो जाता।
- 32 KB से बड़ी अनुरोध बॉडी अस्वीकार कर दी जाती हैं।

Twilio डिफ़ॉल्ट रूप से HTTP 429 का पुनः प्रयास नहीं करता और `Retry-After` के समर्थन का दस्तावेज़ीकरण नहीं करता। `#rp=4xx` और `#rp=all` कनेक्शन ओवरराइड 4xx पुनः प्रयासों को सक्रिय करते हैं, लेकिन Twilio संपूर्ण पुनः प्रयास लेनदेन को 15 सेकंड तक सीमित करता है, इसलिए पुनः प्रयास फिर भी रीप्ले-कैश स्लॉट की समय-सीमा समाप्त होने से पहले समाप्त हो सकते हैं। जब किसी अन्य हैंडलर को विफल डिलीवरी प्राप्त करनी हो, तब फ़ॉलबैक URL कॉन्फ़िगर करें; 429 को विश्वसनीय बैकप्रेशर नहीं, बल्कि विफलता-सुरक्षित अस्वीकृति मानें।

केवल स्थानीय टनल परीक्षण के लिए, आप यह सेट कर सकते हैं:

```json5
{
  channels: {
    sms: {
      dangerouslyDisableSignatureValidation: true,
    },
  },
}
```

सार्वजनिक Gateway पर अक्षम हस्ताक्षर सत्यापन का उपयोग न करें।

## मल्टी-अकाउंट कॉन्फ़िगरेशन

जब आप एक से अधिक Twilio नंबर संचालित करते हैं, तब `accounts` का उपयोग करें:

```json5
{
  channels: {
    sms: {
      accounts: {
        support: {
          enabled: true,
          accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
          authToken: "twilio-auth-token",
          fromNumber: "+15551234567",
          publicWebhookUrl: "https://gateway.example.com/webhooks/sms/support",
          webhookPath: "/webhooks/sms/support",
          dmPolicy: "allowlist",
          allowFrom: ["+15557654321"],
        },
      },
    },
  },
}
```

प्रत्येक अकाउंट को अलग `webhookPath` का उपयोग करना चाहिए; Gateway ऐसे Webhook रूट को पंजीकृत करने से मना कर देता है जिसका पाथ पहले से किसी अन्य अकाउंट के स्वामित्व में हो। `TWILIO_*`/`SMS_*` एनवायरनमेंट फ़ॉलबैक केवल डिफ़ॉल्ट अकाउंट पर लागू होते हैं; डिफ़ॉल्ट अकाउंट बदलने के लिए `defaultAccount` सेट करें।

## समस्या निवारण

### Twilio 403 लौटाता है या OpenClaw Webhook को अस्वीकार करता है

जाँचें कि `publicWebhookUrl`, Twilio में कॉन्फ़िगर किए गए URL से बिल्कुल मेल खाता है, जिसमें स्कीम, होस्ट, पाथ और क्वेरी स्ट्रिंग शामिल हैं। Twilio सार्वजनिक URL स्ट्रिंग पर हस्ताक्षर करता है, इसलिए प्रॉक्सी द्वारा पुनर्लेखन और वैकल्पिक होस्टनाम हस्ताक्षर सत्यापन को विफल कर सकते हैं।

`Invalid account` वाला 403 बताता है कि इनबाउंड पेलोड का `AccountSid`, कॉन्फ़िगर किए गए `accountSid` से मेल नहीं खाता; जाँचें कि Webhook उस अकाउंट की ओर इंगित करता है जिसके स्वामित्व में नंबर है।

### कोई पेयरिंग अनुरोध दिखाई नहीं देता

Twilio नंबर का **Messaging** Webhook URL और विधि जाँचें। इसे SMS Webhook URL की ओर इंगित करना और `POST` का उपयोग करना चाहिए। यह भी पुष्टि करें कि Gateway सार्वजनिक इंटरनेट या आपकी टनल के माध्यम से पहुँच योग्य है।

यदि Twilio संदेश लॉग में त्रुटि `11200` दिखाई देती है, तो Twilio ने इनबाउंड SMS स्वीकार कर लिया था, लेकिन आपके Webhook तक नहीं पहुँच सका। जाँचें:

- Twilio में **Messaging > A message comes in**, `publicWebhookUrl` की ओर इंगित करता है।
- विधि `POST` है।
- टनल या रिवर्स प्रॉक्सी ठीक उसी `webhookPath` को एक्सपोज़ करता है; Tailscale Funnel के लिए, `tailscale funnel status` चलाएँ और पुष्टि करें कि `/webhooks/sms` सूचीबद्ध है।
- `publicWebhookUrl` उसी स्कीम, होस्ट, पाथ और क्वेरी स्ट्रिंग का उपयोग करता है जिसे Twilio भेजता है, ताकि हस्ताक्षर सत्यापन हस्ताक्षरित URL को पुनः बना सके।

`openclaw channels status --channel sms --probe`, बेमेल Twilio Webhook सेटिंग और हाल की `11200` त्रुटियाँ, दोनों दिखाता है।

### आउटबाउंड प्रेषण विफल होते हैं

पुष्टि करें कि `accountSid`, `authToken`, और `fromNumber` या `messagingServiceSid` में से कोई एक निर्धारित हो गया है। यदि आप परीक्षण वाला Twilio अकाउंट उपयोग करते हैं, तो आउटबाउंड SMS भेजे जाने से पहले गंतव्य नंबर को Twilio में सत्यापित करना आवश्यक हो सकता है।

### संदेश पहुँचते हैं लेकिन एजेंट उत्तर नहीं देता

`dmPolicy` और `allowFrom` जाँचें। डिफ़ॉल्ट `pairing` नीति के साथ, सामान्य एजेंट टर्न संसाधित किए जाने से पहले प्रेषक को स्वीकृत करना आवश्यक है।
