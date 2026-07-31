---
read_when:
    - एक ही मशीन पर एक से अधिक Gateway चलाना
    - आपको प्रत्येक Gateway के लिए अलग-अलग कॉन्फ़िगरेशन/स्थिति/पोर्ट चाहिए
summary: एक होस्ट पर कई OpenClaw Gateway चलाएँ (आइसोलेशन, पोर्ट और प्रोफ़ाइल)
title: एकाधिक गेटवे
x-i18n:
    generated_at: "2026-07-27T19:20:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 655fa865a98064d7c017a7c2eb08ea9a9683002d96a3dbe45a8c16cbd3c86ba1
    source_path: gateway/multiple-gateways.md
    workflow: 16
---

अधिकांश सेटअपों को एक Gateway की आवश्यकता होती है - एक ही Gateway कई मैसेजिंग कनेक्शन और एजेंटों को संभालता है। अलग-अलग प्रोफ़ाइल/पोर्ट वाले पृथक Gateway केवल तभी चलाएँ, जब आपको अधिक मजबूत पृथक्करण या रिडंडेंसी की आवश्यकता हो (जैसे, एक रेस्क्यू बॉट)।

## रेस्क्यू-बॉट क्विकस्टार्ट

सबसे सरल रेस्क्यू-बॉट सेटअप:

- मुख्य बॉट को डिफ़ॉल्ट प्रोफ़ाइल पर रखें।
- रेस्क्यू बॉट को उसके अपने Telegram बॉट टोकन के साथ `--profile rescue` पर चलाएँ।
- रेस्क्यू बॉट को किसी अलग बेस पोर्ट पर रखें, जैसे `19789`।

इससे प्राथमिक बॉट के बंद होने पर भी रेस्क्यू बॉट डीबग कर सकता है या कॉन्फ़िगरेशन परिवर्तन लागू कर सकता है। बेस पोर्टों के बीच कम-से-कम 20 पोर्ट का अंतर रखें, ताकि व्युत्पन्न ब्राउज़र/CDP पोर्ट कभी टकराएँ नहीं।

```bash
# रेस्क्यू बॉट (अलग Telegram बॉट, अलग प्रोफ़ाइल, पोर्ट 19789)
openclaw --profile rescue onboard
openclaw --profile rescue gateway install --port 19789
```

यदि आपका मुख्य बॉट पहले से चल रहा है, तो सामान्यतः आपको केवल इतना ही करना होगा। यदि ऑनबोर्डिंग ने रेस्क्यू सेवा पहले ही इंस्टॉल कर दी है, तो अंतिम `gateway install` छोड़ दें।

`openclaw --profile rescue onboard` के दौरान:

- रेस्क्यू खाते के लिए समर्पित एक अलग Telegram बॉट टोकन का उपयोग करें (इसे केवल ऑपरेटरों तक सीमित रखना आसान है, यह मुख्य बॉट के चैनल/ऐप इंस्टॉल से स्वतंत्र रहता है और DM-आधारित पुनर्प्राप्ति का सरल मार्ग प्रदान करता है)।
- `rescue` प्रोफ़ाइल नाम बनाए रखें।
- मुख्य बॉट से कम-से-कम 20 अधिक बेस पोर्ट का उपयोग करें।
- जब तक आप पहले से स्वयं किसी वर्कस्पेस का प्रबंधन नहीं करते, डिफ़ॉल्ट रेस्क्यू वर्कस्पेस स्वीकार करें।

### `--profile rescue onboard` क्या बदलता है

`--profile rescue onboard` सामान्य ऑनबोर्डिंग प्रवाह चलाता है, लेकिन सब कुछ एक अलग प्रोफ़ाइल में लिखता है, इसलिए रेस्क्यू बॉट को उसके अपने निम्न संसाधन मिलते हैं:

- प्रोफ़ाइल/कॉन्फ़िगरेशन फ़ाइल
- स्थिति डायरेक्टरी
- वर्कस्पेस (डिफ़ॉल्ट: `~/.openclaw/workspace-rescue`)
- प्रबंधित सेवा का नाम
- बेस पोर्ट (और व्युत्पन्न पोर्ट)
- Telegram बॉट टोकन

अन्यथा प्रॉम्प्ट सामान्य ऑनबोर्डिंग के समान होते हैं।

## सामान्य मल्टी-Gateway सेटअप

एक ही होस्ट पर Gateway की किसी भी जोड़ी या समूह के लिए यही पृथक्करण पैटर्न काम करता है - प्रत्येक अतिरिक्त Gateway को उसकी अपनी नामित प्रोफ़ाइल और बेस पोर्ट दें:

```bash
# मुख्य (डिफ़ॉल्ट प्रोफ़ाइल)
openclaw setup
openclaw gateway --port 18789

# अतिरिक्त Gateway
openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

दोनों ओर नामित प्रोफ़ाइल भी काम करती हैं:

```bash
openclaw --profile main setup
openclaw --profile main gateway --port 18789

openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

सेवाएँ भी इसी पैटर्न का पालन करती हैं:

```bash
openclaw gateway install
openclaw --profile ops gateway install --port 19789
```

फ़ॉलबैक ऑपरेटर लेन के लिए रेस्क्यू-बॉट क्विकस्टार्ट का उपयोग करें; अलग-अलग चैनलों, टेनेंटों, वर्कस्पेसों या परिचालन भूमिकाओं में लंबे समय तक चलने वाले कई Gateway के लिए सामान्य प्रोफ़ाइल पैटर्न का उपयोग करें।

## पृथक्करण चेकलिस्ट

प्रत्येक Gateway इंस्टेंस के लिए इन्हें अद्वितीय रखें:

| सेटिंग                      | उद्देश्य                              |
| ---------------------------- | ------------------------------------ |
| `OPENCLAW_CONFIG_PATH`       | प्रत्येक इंस्टेंस की कॉन्फ़िगरेशन फ़ाइल             |
| `OPENCLAW_STATE_DIR`         | प्रत्येक इंस्टेंस के सेशन, क्रेडेंशियल और कैश |
| `agents.defaults.workspace`  | प्रत्येक इंस्टेंस का वर्कस्पेस रूट          |
| `gateway.port` (या `--port`) | प्रत्येक इंस्टेंस के लिए अद्वितीय                  |
| व्युत्पन्न ब्राउज़र/CDP पोर्ट    | नीचे देखें                            |

इनमें से कुछ भी साझा करने से कॉन्फ़िगरेशन, स्थिति या पोर्ट में टकराव होता है। Gateway का स्टार्टअप
अद्वितीय स्थिति-डायरेक्टरी स्वामित्व लागू करता है, तब भी जब
`OPENCLAW_ALLOW_MULTI_GATEWAY=1` प्रत्येक कॉन्फ़िगरेशन के सिंगलटन को छोड़ देता है।

## पोर्ट मैपिंग (व्युत्पन्न)

बेस पोर्ट = `gateway.port` (या `OPENCLAW_GATEWAY_PORT` / `--port`)।

- ब्राउज़र नियंत्रण सेवा पोर्ट = बेस + 2 (केवल लूपबैक)।
- Canvas होस्ट स्वयं Gateway HTTP सर्वर पर प्रस्तुत किया जाता है (`gateway.port` वाला ही पोर्ट)।
- ब्राउज़र प्रोफ़ाइल CDP पोर्ट `browser control port + 9` से `+ 108` तक स्वचालित रूप से आवंटित होते हैं।

कॉन्फ़िगरेशन या परिवेश में इनमें से किसी को भी ओवरराइड करने पर आपको उन्हें प्रत्येक इंस्टेंस के लिए अद्वितीय रखना होगा।

## ब्राउज़र/CDP नोट्स (सामान्य चूक)

- कई इंस्टेंसों पर `browser.cdpUrl` को एक ही मान पर **पिन न करें**।
- प्रत्येक इंस्टेंस को अपना ब्राउज़र नियंत्रण पोर्ट और CDP रेंज चाहिए (उसके Gateway पोर्ट से व्युत्पन्न)।
- स्पष्ट CDP पोर्ट के लिए, प्रत्येक इंस्टेंस पर `browser.profiles.<name>.cdpPort` सेट करें।
- रिमोट Chrome के लिए, `browser.profiles.<name>.cdpUrl` का उपयोग करें (प्रत्येक प्रोफ़ाइल और प्रत्येक इंस्टेंस के लिए)।

## मैन्युअल परिवेश उदाहरण

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/main.json \
OPENCLAW_STATE_DIR=~/.openclaw \
openclaw gateway --port 18789

OPENCLAW_CONFIG_PATH=~/.openclaw/rescue.json \
OPENCLAW_STATE_DIR=~/.openclaw-rescue \
openclaw gateway --port 19789
```

## त्वरित जाँच

```bash
openclaw gateway status --deep
openclaw --profile rescue gateway status --deep
openclaw --profile rescue gateway probe
openclaw status
openclaw --profile rescue status
openclaw --profile rescue browser status
```

- `gateway status --deep` पुराने इंस्टॉल से बची हुई launchd/systemd/schtasks सेवाओं का पता लगाता है।
- `gateway probe` चेतावनी पाठ, जैसे `multiple reachable gateway identities detected`, केवल तभी अपेक्षित है जब आप जानबूझकर एक से अधिक पृथक Gateway चलाते हैं या जब OpenClaw यह प्रमाणित नहीं कर पाता कि पहुँच योग्य प्रोब लक्ष्य एक ही Gateway हैं। एक ही Gateway के लिए SSH टनल, प्रॉक्सी URL या कॉन्फ़िगर किया गया रिमोट URL कई ट्रांसपोर्ट वाला एक Gateway ही है, भले ही ट्रांसपोर्ट पोर्ट अलग-अलग हों।

## संबंधित

- [Gateway रनबुक](/hi/gateway)
- [Gateway लॉक](/hi/gateway/gateway-lock)
- [कॉन्फ़िगरेशन](/hi/gateway/configuration)
