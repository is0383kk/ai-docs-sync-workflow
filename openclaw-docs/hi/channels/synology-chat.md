---
read_when:
    - OpenClaw के साथ Synology Chat सेट अप करना
    - Synology Chat Webhook रूटिंग की डीबगिंग
summary: Synology Chat Webhook सेटअप और OpenClaw कॉन्फ़िगरेशन
title: Synology Chat
x-i18n:
    generated_at: "2026-07-27T18:56:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3c03379944ee4187260a7287f6d2aed1ad8fdd1c22b5581c8a5d55515bbb6ad5
    source_path: channels/synology-chat.md
    workflow: 16
---

Synology Chat एक Webhook जोड़ी के माध्यम से OpenClaw से जुड़ता है: एक Synology Chat आउटगोइंग Webhook आने वाले डायरेक्ट संदेशों को Gateway पर पोस्ट करता है, और उत्तर Synology Chat इनकमिंग Webhook के माध्यम से वापस जाते हैं।

स्थिति: आधिकारिक Plugin, अलग से इंस्टॉल किया जाता है। केवल डायरेक्ट संदेश; टेक्स्ट और URL-आधारित फ़ाइल भेजना समर्थित है।

## इंस्टॉल करें

```bash
openclaw plugins install @openclaw/synology-chat
```

स्थानीय चेकआउट (git रिपॉज़िटरी से चलाते समय):

```bash
openclaw plugins install ./path/to/local/synology-chat-plugin
```

विवरण: [Plugins](/hi/tools/plugin)

## त्वरित सेटअप

1. Plugin इंस्टॉल करें (ऊपर)।
2. Synology Chat इंटीग्रेशन में:
   - एक इनकमिंग Webhook बनाएँ और उसका URL कॉपी करें।
   - अपने सीक्रेट टोकन के साथ एक आउटगोइंग Webhook बनाएँ।
3. आउटगोइंग Webhook URL को अपने OpenClaw Gateway की ओर इंगित करें:
   - `https://gateway-host/webhook/synology` डिफ़ॉल्ट रूप से।
   - या आपका कस्टम `channels.synology-chat.webhookPath`।
4. OpenClaw में सेटअप पूरा करें। Synology Chat दोनों प्रवाहों में समान चैनल सेटअप सूची में दिखाई देता है:
   - निर्देशित: `openclaw onboard` या `openclaw channels add`
   - प्रत्यक्ष: `openclaw channels add --channel synology-chat --token <token> --url <incoming-webhook-url>`
5. Gateway पुनः प्रारंभ करें और Synology Chat बॉट को एक डायरेक्ट संदेश भेजें।

Webhook प्रमाणीकरण विवरण:

- OpenClaw आउटगोइंग Webhook टोकन को पहले `body.token`, फिर
  `?token=...`, और उसके बाद हेडर से स्वीकार करता है।
- स्वीकृत हेडर प्रारूप:
  - `x-synology-token`
  - `x-webhook-token`
  - `x-openclaw-token`
  - `Authorization: Bearer <token>`
- खाली या अनुपस्थित टोकन अस्वीकृत होते हैं।
- पेलोड `application/x-www-form-urlencoded` या `application/json` हो सकते हैं; `token`, `user_id`, और `text` आवश्यक हैं।

## इनबाउंड स्थायित्व

टोकन, प्रेषक नीति और दर-सीमा जाँच पास होने के बाद, OpenClaw संग्रहित एनवेलप से Webhook टोकन हटाता है और ईवेंट की अभिस्वीकृति देने से पहले उसे स्थायी रूप से कतारबद्ध करता है। रूट केवल तभी `204` लौटाता है जब वह जोड़ सफल हो जाता है; स्थायित्व विफल होने पर `503` लौटाया जाता है, ताकि Synology Chat संदेश को चुपचाप खोने के बजाय पुनः प्रयास कर सके।

लंबित या पुनः प्रयास योग्य ईवेंट Gateway पुनः प्रारंभ होने के बाद भी बने रहते हैं। जब तक संबंधित सक्रिय या संरक्षित पूर्णता रिकॉर्ड मौजूद रहता है, Synology का स्थिर `post_id` डुप्लिकेट कतार प्रविष्टियों को रोकता है। कतार से एजेंट तक हैंडऑफ़ के दौरान डिलीवरी कम-से-कम-एक-बार बनी रहती है, इसलिए उस सीमा पर क्रैश होने से कोई टर्न फिर भी दोबारा चल सकता है।

न्यूनतम कॉन्फ़िगरेशन:

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      token: "synology-outgoing-token",
      incomingUrl: "https://nas.example.com/webapi/entry.cgi?api=SYNO.Chat.External&method=incoming&version=2&token=...",
      webhookPath: "/webhook/synology",
      dmPolicy: "allowlist",
      allowedUserIds: ["123456"],
      rateLimitPerMinute: 30,
      allowInsecureSsl: false,
    },
  },
}
```

## परिवेश चर

डिफ़ॉल्ट खाते के लिए, आप परिवेश चर उपयोग कर सकते हैं:

- `SYNOLOGY_CHAT_TOKEN`
- `SYNOLOGY_CHAT_INCOMING_URL`
- `SYNOLOGY_NAS_HOST`
- `SYNOLOGY_ALLOWED_USER_IDS` (अल्पविराम से अलग)
- `SYNOLOGY_RATE_LIMIT`
- `OPENCLAW_BOT_NAME`

कॉन्फ़िगरेशन मान परिवेश चरों को ओवरराइड करते हैं।

`SYNOLOGY_CHAT_INCOMING_URL` और `SYNOLOGY_NAS_HOST` को वर्कस्पेस `.env` से सेट नहीं किया जा सकता; [वर्कस्पेस `.env` फ़ाइलें](/hi/gateway/security#workspace-env-files) देखें।

## डायरेक्ट संदेश नीति और अभिगम नियंत्रण

- समर्थित `dmPolicy` मान: `allowlist` (डिफ़ॉल्ट), `open`, और `disabled`। Synology Chat में पेयरिंग प्रवाह नहीं है; प्रेषकों को स्वीकृत करने के लिए उनकी संख्यात्मक Synology उपयोगकर्ता ID को `allowedUserIds` में जोड़ें।
- `allowedUserIds` Synology उपयोगकर्ता ID की सूची (या अल्पविराम से अलग स्ट्रिंग) स्वीकार करता है।
- `allowlist` मोड में, खाली `allowedUserIds` सूची को गलत कॉन्फ़िगरेशन माना जाता है और Webhook रूट प्रारंभ नहीं होगा।
- `dmPolicy: "open"` सार्वजनिक डायरेक्ट संदेशों की अनुमति केवल तभी देता है जब `allowedUserIds` में `"*"` शामिल हो; प्रतिबंधात्मक प्रविष्टियों के साथ केवल मेल खाने वाले उपयोगकर्ता चैट कर सकते हैं। खाली `allowedUserIds` सूची के साथ `open` भी रूट प्रारंभ करने से इनकार करता है।
- `dmPolicy: "disabled"` डायरेक्ट संदेशों को अवरुद्ध करता है।
- उत्तर प्राप्तकर्ता बाइंडिंग डिफ़ॉल्ट रूप से स्थिर संख्यात्मक `user_id` पर रहती है। `channels.synology-chat.dangerouslyAllowNameMatching: true` आपातकालीन संगतता मोड है, जो उत्तर डिलीवरी के लिए परिवर्तनशील उपयोगकर्ता नाम/उपनाम लुकअप को फिर से सक्षम करता है।

## आउटबाउंड डिलीवरी

लक्ष्य के रूप में संख्यात्मक Synology Chat उपयोगकर्ता ID का उपयोग करें। `synology-chat:`, `synology_chat:`, और `synology:` उपसर्ग स्वीकार किए जाते हैं।

उदाहरण:

```bash
openclaw message send --channel synology-chat --target 123456 --message "Hello from OpenClaw"
openclaw message send --channel synology-chat --target synology-chat:123456 --message "Hello again"
openclaw message send --channel synology-chat --target synology:123456 --message "Short prefix"
```

आउटबाउंड टेक्स्ट को 2000 वर्णों पर खंडित किया जाता है। URL-आधारित फ़ाइल डिलीवरी के माध्यम से मीडिया भेजना समर्थित है: NAS फ़ाइल डाउनलोड करके संलग्न करता है (अधिकतम 32 MB)। आउटबाउंड फ़ाइल URL को `http` या `https` का उपयोग करना आवश्यक है, और OpenClaw द्वारा URL को NAS Webhook पर अग्रेषित करने से पहले निजी या अन्यथा अवरुद्ध नेटवर्क लक्ष्यों को अस्वीकार कर दिया जाता है।

## एकाधिक खाते

`channels.synology-chat.accounts` के अंतर्गत एकाधिक Synology Chat खाते समर्थित हैं।
प्रत्येक खाता टोकन, इनकमिंग URL, Webhook पथ, डायरेक्ट संदेश नीति और सीमाओं को ओवरराइड कर सकता है।
डायरेक्ट-संदेश सत्र प्रत्येक खाते और उपयोगकर्ता के अनुसार पृथक होते हैं, इसलिए दो अलग-अलग Synology खातों पर समान संख्यात्मक `user_id`
ट्रांसक्रिप्ट स्थिति साझा नहीं करता।
प्रत्येक सक्षम खाते को एक अलग `webhookPath` दें। OpenClaw समान सटीक पथों को अस्वीकार करता है
और एकाधिक-खाता सेटअप में केवल साझा Webhook पथ इनहेरिट करने वाले नामित खातों को प्रारंभ करने से इनकार करता है।
यदि आपको किसी नामित खाते के लिए जानबूझकर लेगेसी इनहेरिटेंस की आवश्यकता है, तो उस खाते पर
`dangerouslyAllowInheritedWebhookPath: true` या `channels.synology-chat` पर इसे सेट करें,
लेकिन समान सटीक पथ फिर भी सुरक्षित रूप से अस्वीकृत किए जाते हैं। प्रत्येक खाते के लिए स्पष्ट पथों को प्राथमिकता दें।

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      accounts: {
        default: {
          token: "token-a",
          incomingUrl: "https://nas-a.example.com/...token=...",
        },
        alerts: {
          token: "token-b",
          incomingUrl: "https://nas-b.example.com/...token=...",
          webhookPath: "/webhook/synology-alerts",
          dmPolicy: "allowlist",
          allowedUserIds: ["987654"],
        },
      },
    },
  },
}
```

## सुरक्षा संबंधी टिप्पणियाँ

- `token` को गुप्त रखें और लीक होने पर उसे बदलें।
- `allowInsecureSsl: false` को बनाए रखें, जब तक कि आप किसी स्व-हस्ताक्षरित स्थानीय NAS प्रमाणपत्र पर स्पष्ट रूप से भरोसा न करते हों।
- इनबाउंड Webhook अनुरोध टोकन द्वारा सत्यापित किए जाते हैं और प्रत्येक प्रेषक के लिए दर-सीमित होते हैं (`rateLimitPerMinute`, डिफ़ॉल्ट 30)।
- अमान्य टोकन जाँच नियत-समय सीक्रेट तुलना का उपयोग करती है और सुरक्षित रूप से अस्वीकार करती है; बार-बार अमान्य टोकन प्रयास स्रोत IP को अस्थायी रूप से लॉक कर देते हैं।
- इनबाउंड संदेश टेक्स्ट को ज्ञात प्रॉम्प्ट-इंजेक्शन पैटर्न से सुरक्षित करने के लिए साफ़ किया जाता है और 4000 वर्णों पर काट दिया जाता है।
- प्रोडक्शन के लिए `dmPolicy: "allowlist"` को प्राथमिकता दें।
- `dangerouslyAllowNameMatching` को बंद रखें, जब तक कि आपको स्पष्ट रूप से लेगेसी उपयोगकर्ता-नाम-आधारित उत्तर डिलीवरी की आवश्यकता न हो।
- `dangerouslyAllowInheritedWebhookPath` को बंद रखें, जब तक कि आप एकाधिक-खाता सेटअप में साझा-पथ रूटिंग जोखिम को स्पष्ट रूप से स्वीकार न करते हों।

## समस्या निवारण

- `Missing required fields (token, user_id, text)`:
  - आउटगोइंग Webhook पेलोड में आवश्यक फ़ील्ड में से कोई एक अनुपस्थित है
  - यदि Synology टोकन को हेडर में भेजता है, तो सुनिश्चित करें कि Gateway/प्रॉक्सी उन हेडर को बनाए रखता है
- `Invalid token`:
  - आउटगोइंग Webhook सीक्रेट `channels.synology-chat.token` से मेल नहीं खाता
  - अनुरोध गलत खाते/Webhook पथ पर पहुँच रहा है
  - रिवर्स प्रॉक्सी ने अनुरोध के OpenClaw तक पहुँचने से पहले टोकन हेडर हटा दिया
- `Rate limit exceeded`:
  - एक ही स्रोत से बहुत अधिक अमान्य टोकन प्रयास उस स्रोत को अस्थायी रूप से लॉक कर सकते हैं
  - प्रमाणित प्रेषकों के लिए भी एक अलग प्रति-उपयोगकर्ता संदेश दर सीमा होती है
- `Allowlist is empty. Configure allowedUserIds or use dmPolicy=open with allowedUserIds=["*"].`:
  - `dmPolicy="allowlist"` सक्षम है, लेकिन कोई उपयोगकर्ता कॉन्फ़िगर नहीं है
- `User not authorized`:
  - प्रेषक का संख्यात्मक `user_id`, `allowedUserIds` में नहीं है

## संबंधित

- [चैनल अवलोकन](/hi/channels) — सभी समर्थित चैनल
- [समूह](/hi/channels/groups) — समूह चैट व्यवहार और उल्लेख नियंत्रण
- [चैनल रूटिंग](/hi/channels/channel-routing) — संदेशों के लिए सत्र रूटिंग
- [सुरक्षा](/hi/gateway/security) — अभिगम मॉडल और सुदृढ़ीकरण
