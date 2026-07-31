---
read_when:
    - आप अभी भी स्क्रिप्ट में `openclaw daemon ...` का उपयोग करते हैं
    - आपको सेवा जीवनचक्र कमांड (इंस्टॉल/शुरू/बंद/पुनः शुरू/स्थिति) चाहिए
summary: '`openclaw daemon` के लिए CLI संदर्भ (Gateway सेवा प्रबंधन का लेगेसी उपनाम)'
title: डेमन
x-i18n:
    generated_at: "2026-07-27T17:32:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 629852ebf3efe86dedc4c84f6ddc9349b25ddde832df5d78521641fe4b137658
    source_path: cli/daemon.md
    workflow: 16
---

# `openclaw daemon`

Gateway सेवा प्रबंधन के लिए पुराना उपनाम। `openclaw daemon ...` उन्हीं सेवा-नियंत्रण कमांड से मैप होता है जिनसे `openclaw gateway ...` होता है। वर्तमान दस्तावेज़ों और उदाहरणों के लिए [`openclaw gateway`](/hi/cli/gateway) को प्राथमिकता दें।

## उपयोग

```bash
openclaw daemon status
openclaw daemon install
openclaw daemon start
openclaw daemon stop
openclaw daemon restart
openclaw daemon uninstall
```

## उपकमांड और विकल्प

| उपकमांड  | विकल्प                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------ |
| `status`    | `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json` |
| `install`   | `--port`, `--runtime <node>`, `--token`, `--wrapper <path>`, `--force`, `--json`                 |
| `uninstall` | `--json`                                                                                         |
| `start`     | `--json`                                                                                         |
| `stop`      | `--json`, `--disable` (केवल launchd: अगली बार शुरू होने तक KeepAlive/RunAtLoad को स्थायी रूप से दबाएँ) |
| `restart`   | `--force`, `--safe`, `--skip-deferral`, `--wait <duration>`, `--json`                            |

- `status`: सेवा की इंस्टॉल स्थिति (launchd/systemd/schtasks) दिखाता है और Gateway के स्वास्थ्य की जाँच करता है।
- `install`: सेवा इंस्टॉल करता है; `--force` किसी मौजूदा इंस्टॉलेशन को फिर से इंस्टॉल/ओवरराइट करता है।
- `restart --safe`: चल रहे Gateway से सक्रिय कार्य की प्रारंभिक जाँच करने और कार्य समाप्त होने के बाद एक समेकित पुनः आरंभ निर्धारित करने को कहता है, जिसकी अधिकतम सीमा 5 मिनट है। यह समय-सीमा समाप्त होने पर भी पुनः आरंभ अनिवार्य रूप से किया जाता है। साधारण `restart` सीधे सेवा प्रबंधक का उपयोग करता है; `--force` तत्काल ओवरराइड है।
- `restart --safe --skip-deferral`: सक्रिय-कार्य स्थगन गेट को बायपास करता है, ताकि अवरोधकों की सूचना मिलने पर भी Gateway तुरंत पुनः आरंभ हो। इसके लिए `--safe` आवश्यक है।

## टिप्पणियाँ

- `status` संभव होने पर जाँच प्रमाणीकरण के लिए कॉन्फ़िगर किए गए प्रमाणीकरण SecretRefs को रिज़ॉल्व करता है। यदि कोई आवश्यक SecretRef रिज़ॉल्व नहीं होता, तो `status --json`, `rpc.authWarning` की सूचना देता है; `--token`/`--password` स्पष्ट रूप से पास करें या पहले सीक्रेट स्रोत को रिज़ॉल्व करें। जाँच के अन्यथा सफल होते ही अनरिज़ॉल्व्ड प्रमाणीकरण चेतावनियाँ दबा दी जाती हैं।
- `status --deep` अन्य Gateway-जैसी सेवाओं के लिए सर्वोत्तम-प्रयास वाली सिस्टम-स्तरीय स्कैनिंग जोड़ता है (क्लीनअप संकेत प्रिंट करता है; फिर भी प्रति मशीन एक Gateway रखने की अनुशंसा है) और Plugin-जागरूक मोड में कॉन्फ़िगरेशन सत्यापन चलाता है, जिससे वे Plugin मैनिफ़ेस्ट चेतावनियाँ सामने आती हैं जिन्हें तेज़ डिफ़ॉल्ट पथ छोड़ देता है।
- Linux systemd इंस्टॉलेशन पर, टोकन-ड्रिफ़्ट जाँचें `Environment=` और `EnvironmentFile=` दोनों यूनिट स्रोतों का निरीक्षण करती हैं।
- टोकन-ड्रिफ़्ट जाँचें मर्ज किए गए रनटाइम env का उपयोग करके `gateway.auth.token` SecretRefs को रिज़ॉल्व करती हैं (पहले सेवा कमांड env, फिर प्रक्रिया env)। यदि टोकन प्रमाणीकरण प्रभावी रूप से सक्रिय नहीं है (`password`/`none`/`trusted-proxy` में से `gateway.auth.mode`, या सेट न होने पर पासवर्ड को प्राथमिकता मिल सकती है), तो कॉन्फ़िगरेशन टोकन रिज़ॉल्यूशन छोड़ दिया जाता है।
- `install` सत्यापित करता है कि SecretRef-प्रबंधित `gateway.auth.token` रिज़ॉल्व किया जा सकता है, लेकिन रिज़ॉल्व किए गए मान को सेवा परिवेश मेटाडेटा में कभी स्थायी नहीं करता; यदि इसे रिज़ॉल्व नहीं किया जा सकता, तो इंस्टॉलेशन सुरक्षित रूप से विफल हो जाता है।
- यदि `gateway.auth.token` और `gateway.auth.password` दोनों कॉन्फ़िगर हैं तथा `gateway.auth.mode` सेट नहीं है, तो `install` तब तक अवरुद्ध रहता है जब तक आप मोड को स्पष्ट रूप से सेट नहीं करते।
- macOS पर, `install`, सीक्रेट को `EnvironmentVariables` में एम्बेड करने के बजाय LaunchAgent plist और जनरेट की गई env फ़ाइल/रैपर को केवल स्वामी के लिए सुलभ (मोड `0600`/`0700`) रखता है।
- एक होस्ट पर कई Gateways चलाना: पोर्ट, कॉन्फ़िगरेशन/स्थिति और कार्यस्थानों को अलग रखें। [एकाधिक Gateway](/hi/gateway#multiple-gateways-same-host) देखें।

## संबंधित

- [CLI संदर्भ](/hi/cli)
- [Gateway रनबुक](/hi/gateway)
