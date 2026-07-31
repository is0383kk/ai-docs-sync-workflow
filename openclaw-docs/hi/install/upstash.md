---
read_when:
    - OpenClaw को Upstash Box पर डिप्लॉय करना
    - आप OpenClaw के लिए एक प्रबंधित Linux परिवेश चाहते हैं, जिसमें SSH टनल के माध्यम से डैशबोर्ड की पहुँच हो
summary: कीप-अलाइव और SSH टनल एक्सेस के साथ Upstash Box पर OpenClaw होस्ट करें
title: Upstash Box
x-i18n:
    generated_at: "2026-07-27T18:29:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 29232c43e0e4940b7445ab8896c9ccd3e81d0fdbdd522d7f50cb8c8057ac18f0
    source_path: install/upstash.md
    workflow: 16
---

Upstash Box पर एक स्थायी OpenClaw Gateway चलाएँ, जो keep-alive जीवनचक्र समर्थन वाला एक प्रबंधित Linux परिवेश है।

डैशबोर्ड एक्सेस के लिए SSH टनल का उपयोग करें। Gateway पोर्ट को सीधे सार्वजनिक इंटरनेट पर उजागर न करें।

## पूर्वापेक्षाएँ

- Upstash खाता
- Keep-alive Upstash Box
- आपकी स्थानीय मशीन पर SSH क्लाइंट

## Box बनाएँ

Upstash Console में एक keep-alive Box बनाएँ। Box ID (उदाहरण के लिए
`right-flamingo-14486`) और अपनी Box API कुंजी नोट कर लें।

Upstash अपना वर्तमान OpenClaw Box चरण-दर-चरण मार्गदर्शक
[OpenClaw सेटअप](https://upstash.com/docs/box/guides/openclaw-setup) पर उपलब्ध रखता है।

## SSH टनल से कनेक्ट करें

OpenClaw डैशबोर्ड पोर्ट को अपनी स्थानीय मशीन पर फ़ॉरवर्ड करें। संकेत मिलने पर अपनी Box API कुंजी को
SSH पासवर्ड के रूप में उपयोग करें:

```bash
ssh -o ServerAliveInterval=15 -o ServerAliveCountMax=3 -L 18789:127.0.0.1:18789 <box-id>@us-east-1.box.upstash.com
```

keepalive विकल्प ऑनबोर्डिंग के दौरान निष्क्रिय टनल के डिस्कनेक्ट होने की घटनाएँ कम करते हैं।

## OpenClaw इंस्टॉल करें

Box के अंदर:

```bash
sudo npm install -g openclaw
```

## ऑनबोर्डिंग चलाएँ

```bash
openclaw onboard --install-daemon
```

संकेतों का पालन करें। ऑनबोर्डिंग पूर्ण होने पर डैशबोर्ड URL और टोकन कॉपी करें।

## Gateway शुरू करें

Box नेटवर्क के लिए Gateway को कॉन्फ़िगर करें और इसे बैकग्राउंड में शुरू करें:

```bash
openclaw config set gateway.bind lan
nohup openclaw gateway > gateway.log 2>&1 &
```

SSH टनल सक्रिय होने पर, डैशबोर्ड URL को स्थानीय रूप से खोलें:

```text
http://127.0.0.1:18789/#token=<your-token>
```

## स्वचालित पुनः आरंभ

इस कमांड को Box init स्क्रिप्ट के रूप में सेट करें, ताकि Box शुरू होने पर Gateway पुनः आरंभ हो:

```bash
nohup openclaw gateway > gateway.log 2>&1 &
```

## समस्या निवारण

यदि ऑनबोर्डिंग के दौरान SSH रुक जाए, तो साफ़ SSH कॉन्फ़िगरेशन और
keepalive विकल्पों के साथ दोबारा कनेक्ट करें:

```bash
ssh -F /dev/null -o ControlMaster=no -o ServerAliveInterval=15 -o ServerAliveCountMax=3 -L 18789:127.0.0.1:18789 <box-id>@us-east-1.box.upstash.com
```

यह पुराने स्थानीय `~/.ssh/config` सेटिंग्स को बायपास करता है और निष्क्रिय नेटवर्क अवधियों के दौरान
टनल को सक्रिय रखता है।

## संबंधित

- [रिमोट एक्सेस](/hi/gateway/remote)
- [Gateway सुरक्षा](/hi/gateway/security)
- [OpenClaw अपडेट करना](/hi/install/updating)
