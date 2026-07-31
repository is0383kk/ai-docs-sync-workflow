---
read_when:
    - आपको Gateway लॉग दूरस्थ रूप से टेल करने हैं (SSH के बिना)
    - आप टूलिंग के लिए JSON लॉग पंक्तियाँ चाहते हैं
summary: '`openclaw logs` के लिए CLI संदर्भ (RPC के माध्यम से Gateway लॉग टेल करें)'
title: लॉग्स
x-i18n:
    generated_at: "2026-07-27T17:33:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7c8dc40e70f2eb4f8d6ba8b75b91a33337786a146abbe401079ee374daa5a0c6
    source_path: cli/logs.md
    workflow: 16
---

# `openclaw logs`

RPC के माध्यम से Gateway फ़ाइल लॉग को टेल करें। रिमोट मोड में काम करता है।

## विकल्प

- `--limit <n>`: लौटाई जाने वाली लॉग पंक्तियों की अधिकतम संख्या (डिफ़ॉल्ट `200`)
- `--max-bytes <n>`: लॉग फ़ाइल से पढ़े जाने वाले बाइट्स की अधिकतम संख्या (डिफ़ॉल्ट `250000`)
- `--follow`: लॉग स्ट्रीम का अनुसरण करें
- `--interval <ms>`: अनुसरण करते समय पोलिंग अंतराल (डिफ़ॉल्ट `1000`)
- `--json`: पंक्ति-सीमांकित JSON इवेंट उत्सर्जित करें
- `--plain`: शैलीबद्ध फ़ॉर्मैटिंग के बिना सादा टेक्स्ट आउटपुट
- `--no-color`: ANSI रंग अक्षम करें
- `--local-time`: टाइमस्टैम्प आपके स्थानीय समय क्षेत्र में रेंडर करें (डिफ़ॉल्ट)
- `--utc`: टाइमस्टैम्प UTC में रेंडर करें

## साझा Gateway RPC विकल्प

- `--url <url>`: Gateway WebSocket URL
- `--token <token>`: Gateway टोकन
- `--timeout <ms>`: मिलीसेकंड में टाइमआउट (डिफ़ॉल्ट `30000`)
- `--expect-final`: Gateway कॉल एजेंट-समर्थित होने पर अंतिम प्रतिक्रिया की प्रतीक्षा करें

`--url` पास करने से स्वतः लागू किए गए कॉन्फ़िगरेशन क्रेडेंशियल छोड़ दिए जाते हैं; यदि लक्षित Gateway को प्रमाणीकरण की आवश्यकता है, तो `--token` स्पष्ट रूप से शामिल करें।

## उदाहरण

```bash
openclaw logs
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
openclaw logs --follow --interval 2000
openclaw logs --limit 500 --max-bytes 500000
openclaw logs --json
openclaw logs --plain
openclaw logs --no-color
openclaw logs --utc
openclaw logs --follow --local-time
openclaw logs --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

चयनित रूट प्रोफ़ाइल Gateway की रोलिंग फ़ाइल से मेल खाती है: डिफ़ॉल्ट
प्रोफ़ाइल `openclaw-YYYY-MM-DD.log` का उपयोग करती है, जबकि नामित प्रोफ़ाइल
`openclaw-<profile>-YYYY-MM-DD.log` का उपयोग करती हैं (उदाहरण के लिए,
`openclaw-dev-YYYY-MM-DD.log`)।

## फ़ॉलबैक और पुनर्प्राप्ति व्यवहार

- यदि अंतर्निहित स्थानीय लूपबैक Gateway पेयरिंग माँगता है, कनेक्ट करते समय बंद हो जाता है, या `logs.tail` के उत्तर देने से पहले टाइमआउट हो जाता है, तो `openclaw logs` स्वचालित रूप से कॉन्फ़िगर की गई Gateway फ़ाइल लॉग पर फ़ॉलबैक करता है। स्पष्ट `--url` लक्ष्य कभी इस फ़ॉलबैक का उपयोग नहीं करते।
- अंतर्निहित स्थानीय Gateway RPC विफलता के बाद `--follow` उस कॉन्फ़िगर की गई फ़ाइल पर फ़ॉलबैक नहीं करता—साथ-साथ मौजूद पुरानी फ़ाइल लाइव टेल के बारे में भ्रमित कर सकती है। Linux पर, उपलब्ध होने पर यह इसके बजाय PID के आधार पर सक्रिय उपयोगकर्ता-systemd Gateway जर्नल का उपयोग करता है (चयनित स्रोत प्रिंट करता है); अन्यथा यह लाइव Gateway से दोबारा जुड़ने का प्रयास जारी रखता है।
- `--follow` के दौरान, अस्थायी डिस्कनेक्शन (WebSocket बंद होना, टाइमआउट, कनेक्शन टूटना) एक्सपोनेंशियल बैकऑफ़ के साथ स्वचालित पुनः कनेक्शन शुरू करते हैं: अधिकतम 8 पुनः प्रयास, प्रयासों के बीच अधिकतम 30s। प्रत्येक पुनः प्रयास पर stderr में चेतावनी प्रिंट होती है, और पोल सफल होने पर एक बार `[logs] gateway reconnected` सूचना प्रिंट होती है। `--json` मोड में दोनों stderr पर `{"type":"notice"}` रिकॉर्ड के रूप में उत्सर्जित होते हैं। अप्राप्य त्रुटियाँ (प्रमाणीकरण विफलता, खराब कॉन्फ़िगरेशन) अब भी तुरंत बाहर निकलती हैं।
- `--follow --json` मोड में, लॉग-स्रोत परिवर्तन `{"type":"meta"}` रिकॉर्ड के रूप में उत्सर्जित होते हैं। प्रत्येक `sourceKind` के लिए कर्सर ट्रैक करें: कोई स्ट्रीम Gateway फ़ाइल आउटपुट (`sourceKind: "file"`) से स्थानीय जर्नल फ़ॉलबैक (`sourceKind: "journal"`, `localFallback: true`, `service.pid`/`service.unit` के साथ) पर जा सकती है और पुनर्प्राप्ति के बाद वापस Gateway फ़ाइल आउटपुट पर आ सकती है। पूरे सत्र के लिए एक स्थिर स्रोत या कर्सर न मानें, और पुनर्प्राप्ति के दौरान Gateway फ़ाइल कर्सर फिर से चलाए जाने पर ओवरलैप होती पंक्तियों को स्वीकार करें।

## संबंधित

- [लॉगिंग अवलोकन](/hi/logging)
- [Gateway CLI](/hi/cli/gateway)
- [CLI संदर्भ](/hi/cli)
- [Gateway लॉगिंग](/hi/gateway/logging)
