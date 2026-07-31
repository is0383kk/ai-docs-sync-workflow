---
read_when:
    - Render पर OpenClaw परिनियोजित करना
    - आप Render Blueprints के साथ एक घोषणात्मक क्लाउड परिनियोजन चाहते हैं
summary: Infrastructure-as-Code के साथ Render पर OpenClaw परिनियोजित करें
title: Render
x-i18n:
    generated_at: "2026-07-27T21:07:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a5fbb3c6df04e186df958a62a6130da4e3e485acfeecc7e85fee0d5b69a0438f
    source_path: install/render.mdx
    workflow: 16
---

रेपो के `render.yaml` Blueprint का उपयोग करके [Render](https://render.com) पर OpenClaw डिप्लॉय करें। यह सेवा, डिस्क और एनवायरनमेंट वेरिएबल को एक फ़ाइल में घोषित करता है।

## पूर्वापेक्षाएँ

- एक [Render खाता](https://render.com) (मुफ़्त टियर उपलब्ध है)
- आपके पसंदीदा [मॉडल प्रदाता](/hi/providers) की एक API कुंजी

## डिप्लॉय करें

[Render पर डिप्लॉय करें](https://render.com/deploy?repo=https://github.com/openclaw/openclaw)

यह `render.yaml` से एक Render सेवा बनाता है, Docker इमेज बिल्ड करता है और उसे डिप्लॉय करता है। आपकी सेवा का URL `https://<service-name>.onrender.com` पैटर्न का अनुसरण करता है।

## Blueprint

```yaml
services:
  - type: web
    name: openclaw
    runtime: docker
    plan: starter
    healthCheckPath: /health
    envVars:
      - key: OPENCLAW_GATEWAY_PORT
        value: "8080"
      - key: OPENCLAW_STATE_DIR
        value: /data/.openclaw
      - key: OPENCLAW_WORKSPACE_DIR
        value: /data/workspace
      - key: OPENCLAW_GATEWAY_TOKEN
        generateValue: true # एक सुरक्षित टोकन अपने-आप जनरेट करता है
    disk:
      name: openclaw-data
      mountPath: /data
      sizeGB: 1
```

| सुविधा               | उद्देश्य                                                    |
| --------------------- | ---------------------------------------------------------- |
| `runtime: docker`     | रेपो की Dockerfile से बिल्ड करता है                          |
| `healthCheckPath`     | Render `/health` की निगरानी करता है और अस्वस्थ इंस्टेंस को पुनः आरंभ करता है |
| `generateValue: true` | क्रिप्टोग्राफ़िक रूप से सुरक्षित मान अपने-आप जनरेट करता है            |
| `disk`                | स्थायी स्टोरेज जो फिर से डिप्लॉय करने पर भी बना रहता है                 |

## प्लान चुनना

| प्लान      | बंद होना         | डिस्क          | इनके लिए सर्वोत्तम                      |
| --------- | ----------------- | ------------- | ----------------------------- |
| Free      | 15 मिनट निष्क्रिय रहने के बाद | उपलब्ध नहीं | परीक्षण, डेमो                |
| Starter   | कभी नहीं             | 1GB+          | व्यक्तिगत उपयोग, छोटी टीमें     |
| Standard+ | कभी नहीं             | 1GB+          | प्रोडक्शन, एकाधिक चैनल |

Blueprint में डिफ़ॉल्ट रूप से `starter` होता है। मुफ़्त टियर का उपयोग करने के लिए, अपने फ़ोर्क की `render.yaml` में `plan: free` बदलें — ध्यान दें कि स्थायी डिस्क न होने पर प्रत्येक डिप्लॉय के साथ OpenClaw की स्थिति रीसेट हो जाती है।

## डिप्लॉयमेंट के बाद

### Control UI तक पहुँचें

वेब डैशबोर्ड `https://<your-service>.onrender.com/` पर उपलब्ध है। साझा सीक्रेट का उपयोग करके कनेक्ट करें: अपने-आप जनरेट हुआ `OPENCLAW_GATEWAY_TOKEN` (इसे **Dashboard → your service → Environment** में खोजें), या यदि आपने पासवर्ड प्रमाणीकरण अपनाया है तो अपने पासवर्ड का उपयोग करें।

### लॉग

**Dashboard → your service → Logs** में बिल्ड लॉग (Docker इमेज बनाना), डिप्लॉय लॉग (सेवा शुरू होना) और रनटाइम लॉग (एप्लिकेशन आउटपुट) दिखाई देते हैं।

### शेल एक्सेस

**Dashboard → your service → Shell** एक शेल सत्र खोलता है। स्थायी डिस्क `/data` पर माउंट होती है।

### एनवायरनमेंट वेरिएबल

वेरिएबल को **Dashboard → your service → Environment** में संपादित करें। बदलाव अपने-आप पुनः डिप्लॉयमेंट शुरू करते हैं।

### स्वचालित डिप्लॉयमेंट

कनेक्ट किए गए रेपो की ब्रांच में नया कमिट आने पर Render अपने-आप फिर से डिप्लॉय करता है। यदि आपने अपने फ़ोर्क के बजाय सीधे `openclaw/openclaw` से डिप्लॉय किया है, तो आपके पास इसे ट्रिगर करने के लिए पुश एक्सेस नहीं होगा। इसलिए Dashboard से मैन्युअल Blueprint सिंक चलाकर अपडेट करें या सेवा को अपने फ़ोर्क से जोड़ें।

## कस्टम डोमेन

1. **Dashboard → your service → Settings → Custom Domains**
2. अपना डोमेन जोड़ें
3. निर्देशानुसार DNS कॉन्फ़िगर करें (`*.onrender.com` के लिए CNAME)
4. Render अपने-आप TLS प्रमाणपत्र उपलब्ध कराता है

## स्केलिंग

- **वर्टिकल**: अधिक CPU/RAM के लिए प्लान बदलें। आम तौर पर OpenClaw के लिए पर्याप्त है।
- **हॉरिज़ॉन्टल**: इंस्टेंस की संख्या बढ़ाएँ (Standard प्लान और उससे ऊपर)। चूँकि OpenClaw रनटाइम स्थिति को स्थानीय डिस्क पर रखता है, इसलिए स्टिकी सेशन या बाहरी स्थिति प्रबंधन आवश्यक है।

## बैकअप और माइग्रेशन

Render Dashboard शेल से किसी भी समय स्थिति, कॉन्फ़िगरेशन, प्रमाणीकरण प्रोफ़ाइल और वर्कस्पेस एक्सपोर्ट करें:

```bash
openclaw backup create
```

यह एक पोर्टेबल बैकअप आर्काइव बनाता है। [बैकअप](/hi/cli/backup) देखें।

## समस्या निवारण

### सेवा शुरू नहीं होती

Render Dashboard में डिप्लॉय लॉग जाँचें। सामान्य समस्याएँ:

- `OPENCLAW_GATEWAY_TOKEN` मौजूद नहीं है — सत्यापित करें कि इसे **Dashboard → Environment** में सेट किया गया है
- पोर्ट मेल नहीं खाता — सुनिश्चित करें कि `OPENCLAW_GATEWAY_PORT=8080` ताकि Gateway उस पोर्ट से बाइंड हो जिसकी Render अपेक्षा करता है

### धीमा कोल्ड स्टार्ट (मुफ़्त टियर)

मुफ़्त टियर की सेवाएँ 15 मिनट की निष्क्रियता के बाद बंद हो जाती हैं; बंद होने के बाद पहले अनुरोध में कंटेनर शुरू होते समय कुछ सेकंड लगते हैं। हमेशा चालू रखने के लिए Starter में अपग्रेड करें।

### पुनः डिप्लॉयमेंट के बाद डेटा की हानि

यह मुफ़्त टियर पर होता है (कोई स्थायी डिस्क नहीं)। सशुल्क प्लान में अपग्रेड करें या Render शेल से `openclaw backup create` का उपयोग करके नियमित रूप से बैकअप एक्सपोर्ट करें।

### हेल्थ चेक विफलताएँ

यदि बिल्ड सफल होते हैं, लेकिन डिप्लॉयमेंट विफल होता है, तो संभव है कि सेवा शुरू होने में बहुत अधिक समय ले रही हो या `/health` तक पहुँचा न जा सके। जाँचें:

- बिल्ड लॉग में त्रुटियाँ
- क्या कंटेनर `docker build && docker run` के साथ स्थानीय रूप से चलता है

## अगले चरण

- मैसेजिंग चैनल सेट अप करें: [चैनल](/hi/channels)
- Gateway कॉन्फ़िगर करें: [Gateway कॉन्फ़िगरेशन](/hi/gateway/configuration)
- OpenClaw को नवीनतम रखें: [अपडेट करना](/hi/install/updating)
