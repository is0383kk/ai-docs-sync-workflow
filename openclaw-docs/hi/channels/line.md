---
read_when:
    - आप OpenClaw को LINE से कनेक्ट करना चाहते हैं
    - आपको LINE Webhook और क्रेडेंशियल सेटअप की आवश्यकता है
    - आप LINE-विशिष्ट संदेश विकल्प चाहते हैं
summary: LINE Messaging API Plugin सेटअप, कॉन्फ़िगरेशन और उपयोग
title: LINE
x-i18n:
    generated_at: "2026-07-27T20:27:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa160970278e0899637307136139f7d2fc83bf57defc30771d77649060f77274
    source_path: channels/line.md
    workflow: 16
---

LINE, LINE Messaging API के माध्यम से OpenClaw से जुड़ता है। Plugin, Gateway पर Webhook
रिसीवर के रूप में चलता है और प्रमाणीकरण के लिए आपके चैनल एक्सेस टोकन + चैनल सीक्रेट का
उपयोग करता है।

स्थिति: आधिकारिक Plugin, अलग से इंस्टॉल किया जाता है। डायरेक्ट मैसेज, ग्रुप चैट, मीडिया,
लोकेशन, Flex मैसेज, टेम्पलेट मैसेज और क्विक रिप्लाई समर्थित हैं।
रिएक्शन और थ्रेड समर्थित नहीं हैं।

## इंस्टॉल करें

चैनल को कॉन्फ़िगर करने से पहले LINE इंस्टॉल करें:

```bash
openclaw plugins install @openclaw/line
```

लोकल चेकआउट (git रिपॉज़िटरी से चलाते समय):

```bash
openclaw plugins install ./path/to/local/line-plugin
```

## सेटअप

1. एक LINE Developers खाता बनाएँ और Console खोलें:
   [https://developers.line.biz/console/](https://developers.line.biz/console/)
2. एक Provider बनाएँ (या चुनें) और एक **Messaging API** चैनल जोड़ें।
3. चैनल सेटिंग से **Channel access token** और **Channel secret** कॉपी करें।
4. Messaging API सेटिंग में **Use webhook** सक्षम करें।
5. Webhook URL को अपने Gateway एंडपॉइंट पर सेट करें (HTTPS आवश्यक है):

```text
https://gateway-host/line/webhook
```

Gateway, LINE के Webhook सत्यापन (GET) का उत्तर देता है। हस्ताक्षरित इनबाउंड इवेंट
(POST) के लिए, यह `200` लौटाने से पहले प्रत्येक इवेंट को टिकाऊ इनग्रेस क्यू में लिखता है;
एजेंट प्रोसेसिंग एसिंक्रोनस रूप से जारी रहती है। विफल डिलीवरी का क्यू से पुनः प्रयास किया जाता है,
जिसमें Gateway रीस्टार्ट के बाद के प्रयास भी शामिल हैं, और सीमित पुनः प्रयासों के बाद दोषपूर्ण इवेंट
विफल क्यू रिकॉर्ड बन जाते हैं। यदि टिकाऊ परसिस्टेंस विफल हो जाए, तो अनुरोध किसी ऐसे इवेंट की
पावती देने के बजाय `500` लौटाता है जो खो सकता है।
क्यू-से-एजेंट सीमा पर डिलीवरी कम-से-कम एक बार होती है: सक्रिय डिलीवरी के दौरान Gateway के
बंद होने या क्रैश होने से टर्न दोबारा चल सकता है। मैसेज इवेंट LINE मैसेज ID के आधार पर
डीडुप्लिकेट होते हैं; अन्य इवेंट प्रकार `webhookEventId` का उपयोग करते हैं। सुरक्षित रखे गए पूर्णता रिकॉर्ड
सामान्य डुप्लिकेट Webhook को रोकते हैं, लेकिन बाहरी साइड इफ़ेक्ट उत्पन्न करने वाले हैंडलर
फिर भी आइडेम्पोटेंट होने चाहिए।
यदि आपको कस्टम पाथ चाहिए, तो `channels.line.webhookPath` या
`channels.line.accounts.<id>.webhookPath` सेट करें और URL को उसी के अनुसार अपडेट करें।

सुरक्षा संबंधी टिप्पणियाँ:

- LINE हस्ताक्षर सत्यापन बॉडी पर निर्भर है (रॉ बॉडी पर HMAC), इसलिए OpenClaw सत्यापन से पहले 64 KB की सख्त प्री-ऑथ बॉडी सीमा और रीड टाइमआउट लागू करता है।
- OpenClaw सत्यापित रॉ अनुरोध बाइट से Webhook इवेंट प्रोसेस करता है। हस्ताक्षर-अखंडता की सुरक्षा के लिए अपस्ट्रीम मिडलवेयर द्वारा रूपांतरित `req.body` मानों को अनदेखा किया जाता है।

## कॉन्फ़िगर करें

न्यूनतम कॉन्फ़िग:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "pairing",
    },
  },
}
```

सार्वजनिक DM कॉन्फ़िग:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "open",
      allowFrom: ["*"],
    },
  },
}
```

एनवायरनमेंट वेरिएबल (केवल डिफ़ॉल्ट खाते के लिए):

- `LINE_CHANNEL_ACCESS_TOKEN`
- `LINE_CHANNEL_SECRET`

टोकन/सीक्रेट फ़ाइलें:

```json5
{
  channels: {
    line: {
      tokenFile: "/path/to/line-token.txt",
      secretFile: "/path/to/line-secret.txt",
    },
  },
}
```

`tokenFile` और `secretFile` का नियमित फ़ाइलों की ओर संकेत करना आवश्यक है। सिमलिंक अस्वीकार किए जाते हैं।
इनलाइन कॉन्फ़िग मानों को फ़ाइलों पर प्राथमिकता मिलती है; डिफ़ॉल्ट खाते के लिए एनवायरनमेंट वेरिएबल अंतिम फ़ॉलबैक हैं।

एकाधिक खाते:

```json5
{
  channels: {
    line: {
      accounts: {
        marketing: {
          channelAccessToken: "...",
          channelSecret: "...",
          webhookPath: "/line/marketing",
        },
      },
    },
  },
}
```

## एक्सेस नियंत्रण

डायरेक्ट मैसेज में डिफ़ॉल्ट रूप से पेयरिंग होती है। अज्ञात प्रेषकों को एक पेयरिंग कोड मिलता है और
स्वीकृत होने तक उनके मैसेज अनदेखे किए जाते हैं:

```bash
openclaw pairing list line
openclaw pairing approve line <CODE>
```

अनुमति सूचियाँ और नीतियाँ:

- `channels.line.dmPolicy`: `pairing | allowlist | open | disabled` (डिफ़ॉल्ट `pairing`)
- `channels.line.allowFrom`: DM के लिए अनुमति-सूचीबद्ध LINE यूज़र ID; `dmPolicy: "open"` के लिए `["*"]` आवश्यक है
- `channels.line.groupPolicy`: `allowlist | open | disabled` (डिफ़ॉल्ट `allowlist`)
- `channels.line.groupAllowFrom`: ग्रुप के लिए अनुमति-सूचीबद्ध LINE यूज़र ID; DM की `allowFrom` प्रविष्टियाँ ग्रुप प्रेषकों को प्रवेश नहीं देतीं
- प्रति-ग्रुप ओवरराइड: `channels.line.groups.<groupId>.allowFrom` (साथ में `enabled`, `requireMention`, `systemPrompt`, `skills`)। जब
  `groupPolicy: "allowlist"` हो, तब `groupAllowFrom` या प्रति-ग्रुप `allowFrom` सेट करें; खाली ग्रुप अनुमति सूची, DM खुले होने पर भी ग्रुप मैसेज ब्लॉक करती है।
- स्थिर प्रेषक एक्सेस ग्रुप को `allowFrom`, `groupAllowFrom` और प्रति-ग्रुप `allowFrom` से `accessGroup:<name>` के साथ संदर्भित किया जा सकता है; [एक्सेस ग्रुप](/hi/channels/access-groups) देखें।
- रनटाइम टिप्पणी: यदि `channels.line` पूरी तरह अनुपस्थित है, तो रनटाइम ग्रुप जाँच के लिए `groupPolicy="allowlist"` पर फ़ॉलबैक करता है (भले ही `channels.defaults.groupPolicy` सेट हो)।

LINE ID केस-सेंसिटिव हैं। मान्य ID इस प्रकार दिखते हैं:

- यूज़र: `U` + 32 हेक्स वर्ण
- ग्रुप: `C` + 32 हेक्स वर्ण
- रूम: `R` + 32 हेक्स वर्ण

## मैसेज व्यवहार

- टेक्स्ट को 5000 वर्णों के खंडों में बाँटा जाता है।
- Markdown फ़ॉर्मेटिंग हटा दी जाती है; जहाँ संभव हो, कोड ब्लॉक और टेबल को Flex
  कार्ड में बदला जाता है।
- स्ट्रीमिंग प्रतिक्रियाएँ बफ़र की जाती हैं; एजेंट के काम करते समय LINE को लोडिंग
  एनीमेशन के साथ पूर्ण खंड मिलते हैं।
- मीडिया डाउनलोड की सीमा `channels.line.mediaMaxMb` (डिफ़ॉल्ट 10) द्वारा निर्धारित होती है।
- इनबाउंड मीडिया को एजेंट को भेजने से पहले `~/.openclaw/media/inbound/` के अंतर्गत सहेजा जाता है,
  जो अन्य चैनल Plugin द्वारा उपयोग किए जाने वाले साझा मीडिया स्टोर के अनुरूप है।

## चैनल डेटा (रिच मैसेज)

क्विक रिप्लाई, लोकेशन, Flex कार्ड या टेम्पलेट मैसेज भेजने के लिए
`channelData.line` का उपयोग करें।

```json5
{
  text: "Here you go",
  channelData: {
    line: {
      quickReplies: ["Status", "Help"],
      location: {
        title: "Office",
        address: "123 Main St",
        latitude: 35.681236,
        longitude: 139.767125,
      },
      flexMessage: {
        altText: "Status card",
        contents: {/* Flex payload */},
      },
      templateMessage: {
        type: "confirm",
        text: "Proceed?",
        confirmLabel: "Yes",
        confirmData: "yes",
        cancelLabel: "No",
        cancelData: "no",
      },
    },
  },
}
```

LINE Plugin में Flex मैसेज प्रीसेट के लिए `/card` कमांड भी शामिल है:

```text
/card info "Welcome" "Thanks for joining!"
```

## ACP समर्थन

LINE, ACP (Agent Communication Protocol) वार्तालाप बाइंडिंग का समर्थन करता है:

- `/acp spawn <agent> --bind here` चाइल्ड थ्रेड बनाए बिना वर्तमान LINE चैट को ACP सेशन से बाँधता है।
- कॉन्फ़िगर की गई ACP बाइंडिंग और सक्रिय वार्तालाप-बद्ध ACP सेशन, LINE पर अन्य वार्तालाप चैनलों की तरह काम करते हैं।

विवरण के लिए [ACP एजेंट](/hi/tools/acp-agents) देखें।

## आउटबाउंड मीडिया

LINE Plugin, एजेंट मैसेज टूल के माध्यम से इमेज, वीडियो और ऑडियो भेजता है:

- **इमेज**: LINE इमेज मैसेज के रूप में भेजी जाती हैं; प्रीव्यू इमेज डिफ़ॉल्ट रूप से मीडिया URL होती है।
- **वीडियो**: प्रीव्यू इमेज आवश्यक है; `channelData.line.previewImageUrl` को किसी इमेज URL पर सेट करें।
- **ऑडियो**: LINE ऑडियो मैसेज के रूप में भेजा जाता है; जब तक `channelData.line.durationMs` सेट न हो, अवधि डिफ़ॉल्ट रूप से 60 सेकंड होती है।

सेट होने पर मीडिया प्रकार `channelData.line.mediaKind` से लिया जाता है, अन्यथा इसे
अन्य LINE विकल्पों या URL फ़ाइल सफ़िक्स से निर्धारित किया जाता है और इमेज को फ़ॉलबैक माना जाता है।

आउटबाउंड मीडिया URL सार्वजनिक HTTPS URL होने चाहिए और अधिकतम 2000 वर्ण लंबे हो सकते हैं। OpenClaw
URL को LINE को सौंपने से पहले लक्षित होस्टनेम को सत्यापित करता है और लूपबैक,
लिंक-लोकल तथा निजी-नेटवर्क लक्ष्यों को अस्वीकार करता है।

LINE-विशिष्ट विकल्पों के बिना सामान्य मीडिया प्रेषण इमेज रूट का उपयोग करता है।

## समस्या निवारण

- **Webhook सत्यापन विफल होता है:** सुनिश्चित करें कि Webhook URL HTTPS है और
  `channelSecret`, LINE console से मेल खाता है।
- **कोई इनबाउंड इवेंट नहीं:** पुष्टि करें कि Webhook पाथ `channels.line.webhookPath` से मेल खाता है
  और Gateway तक LINE की पहुँच है।
- **मीडिया डाउनलोड त्रुटियाँ:** यदि मीडिया डिफ़ॉल्ट सीमा से बड़ा है, तो `channels.line.mediaMaxMb`
  बढ़ाएँ।

## संबंधित

- [चैनलों का अवलोकन](/hi/channels) — सभी समर्थित चैनल
- [पेयरिंग](/hi/channels/pairing) — DM प्रमाणीकरण और पेयरिंग प्रवाह
- [ग्रुप](/hi/channels/groups) — ग्रुप चैट व्यवहार और मेंशन गेटिंग
- [चैनल रूटिंग](/hi/channels/channel-routing) — मैसेज के लिए सेशन रूटिंग
- [सुरक्षा](/hi/gateway/security) — एक्सेस मॉडल और हार्डनिंग
