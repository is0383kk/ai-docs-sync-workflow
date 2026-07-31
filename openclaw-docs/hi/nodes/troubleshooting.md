---
read_when:
    - Node कनेक्टेड है, लेकिन कैमरा/कैनवास/स्क्रीन/exec टूल विफल हो रहे हैं
    - आपको Node पेयरिंग और अनुमोदनों के बीच का मानसिक मॉडल समझना आवश्यक है
summary: Node पेयरिंग, फ़ोरग्राउंड आवश्यकताओं, अनुमतियों और टूल विफलताओं की समस्या निवारण करें
title: Node समस्या निवारण
x-i18n:
    generated_at: "2026-07-27T18:31:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4a7ee9e48985805e91cd5acfa1b9f6b676b7e67236ce29fe91e2c8d03002e5c4
    source_path: nodes/troubleshooting.md
    workflow: 16
---

जब कोई Node स्थिति में दिखाई दे रहा हो, लेकिन Node टूल विफल हों, तो इस पृष्ठ का उपयोग करें।

## कमांड क्रम

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

फिर Node-विशिष्ट जाँच चलाएँ:

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
```

सही स्थिति के संकेत:

- Node कनेक्ट है और भूमिका `node` के लिए युग्मित है।
- `nodes describe` में वह क्षमता शामिल है जिसे आप कॉल कर रहे हैं।
- निष्पादन अनुमोदन अपेक्षित मोड/अनुमति-सूची दिखाते हैं।

## अग्रभूमि आवश्यकताएँ

iOS/Android Node पर `canvas.*`, `camera.*`, और `screen.*` केवल अग्रभूमि में उपलब्ध हैं।

त्वरित जाँच और समाधान:

```bash
openclaw nodes describe --node <idOrNameOrIp>
openclaw nodes canvas snapshot --node <idOrNameOrIp>
openclaw logs --follow
```

यदि आपको `NODE_BACKGROUND_UNAVAILABLE` दिखाई दे, तो Node ऐप को अग्रभूमि में लाएँ और फिर से प्रयास करें।

## अनुमतियों की सारणी

| क्षमता                   | iOS                                     | Android                                      | macOS Node ऐप                   | सामान्य विफलता कोड                          |
| ---------------------------- | --------------------------------------- | -------------------------------------------- | -------------------------------- | --------------------------------------------- |
| `camera.snap`, `camera.clip` | कैमरा (+ क्लिप ऑडियो के लिए माइक)           | कैमरा (+ क्लिप ऑडियो के लिए माइक)                | कैमरा (+ क्लिप ऑडियो के लिए माइक)    | `*_PERMISSION_REQUIRED`                       |
| `screen.record`              | स्क्रीन रिकॉर्डिंग (+ माइक वैकल्पिक)       | स्क्रीन कैप्चर संकेत (+ माइक वैकल्पिक)       | स्क्रीन रिकॉर्डिंग                 | `*_PERMISSION_REQUIRED`                       |
| `computer.act`               | लागू नहीं                                     | लागू नहीं                                          | Accessibility + Screen Recording | `COMPUTER_DISABLED`, `ACCESSIBILITY_REQUIRED` |
| `location.get`               | While Using या Always (मोड पर निर्भर) | मोड के अनुसार अग्रभूमि/पृष्ठभूमि स्थान | स्थान अनुमति              | `LOCATION_PERMISSION_REQUIRED`                |
| `system.run`                 | लागू नहीं (Node होस्ट पथ)                    | लागू नहीं (Node होस्ट पथ)                         | निष्पादन अनुमोदन आवश्यक          | `SYSTEM_RUN_DENIED`                           |

## युग्मन बनाम अनुमोदन

तीन अलग-अलग द्वार नियंत्रित करते हैं कि कोई Node कमांड सफल होगा या नहीं:

1. **डिवाइस युग्मन**: क्या यह Node Gateway से कनेक्ट हो सकता है?
2. **Gateway Node कमांड नीति**: क्या RPC कमांड आईडी को `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny` और प्लेटफ़ॉर्म डिफ़ॉल्ट द्वारा अनुमति है?
3. **निष्पादन अनुमोदन**: क्या यह Node किसी विशिष्ट शेल कमांड को स्थानीय रूप से चला सकता है?

Node युग्मन पहचान/विश्वास का द्वार है, प्रत्येक कमांड के लिए अनुमोदन सतह नहीं। `system.run` के लिए, प्रति-Node नीति उस Node की निष्पादन अनुमोदन फ़ाइल (`openclaw approvals get --node ...`) में रहती है, Gateway युग्मन रिकॉर्ड में नहीं।

त्वरित जाँच:

```bash
openclaw devices list
openclaw nodes status
openclaw approvals get --node <idOrNameOrIp>
openclaw approvals allowlist add --node <idOrNameOrIp> "/usr/bin/uname"
```

- युग्मन अनुपस्थित है: पहले Node डिवाइस को अनुमोदित करें।
- `nodes describe` में कोई कमांड अनुपस्थित है: Gateway Node कमांड नीति जाँचें और देखें कि कनेक्ट होते समय Node ने वास्तव में वह कमांड घोषित किया था या नहीं।
- युग्मन ठीक है, लेकिन `system.run` विफल होता है: उस Node पर निष्पादन अनुमोदन/अनुमति-सूची ठीक करें।

अनुमोदन-समर्थित `host=node` संचालन के लिए, Gateway निष्पादन को तैयार किए गए प्रामाणिक `systemRunPlan` से भी बाँधता है। यदि बाद में कोई कॉलर अनुमोदित संचालन अग्रेषित होने से पहले कमांड, cwd, या सत्र मेटाडेटा बदलता है, तो Gateway संपादित पेलोड पर भरोसा करने के बजाय अनुमोदन बेमेल होने के कारण संचालन अस्वीकार कर देता है।

## सामान्य Node त्रुटि कोड

| कोड                                   | अर्थ                                                                                                                                                                                 |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NODE_BACKGROUND_UNAVAILABLE`          | ऐप पृष्ठभूमि में है; इसे अग्रभूमि में लाएँ।                                                                                                                                        |
| `CAMERA_DISABLED`                      | Node सेटिंग में कैमरा टॉगल अक्षम है।                                                                                                                                                |
| `*_PERMISSION_REQUIRED`                | OS अनुमति अनुपस्थित/अस्वीकृत है।                                                                                                                                                           |
| `LOCATION_DISABLED`                    | स्थान मोड बंद है।                                                                                                                                                                   |
| `LOCATION_PERMISSION_REQUIRED`         | अनुरोधित स्थान मोड प्रदान नहीं किया गया है।                                                                                                                                                    |
| `LOCATION_BACKGROUND_UNAVAILABLE`      | ऐप पृष्ठभूमि में है, लेकिन केवल While Using अनुमति उपलब्ध है।                                                                                                                             |
| `COMPUTER_DISABLED`                    | macOS ऐप में **Allow Computer Control** सक्षम करें, फिर युग्मन अपडेट अनुमोदित करें।                                                                                                    |
| `ACCESSIBILITY_REQUIRED`               | macOS System Settings में वर्तमान OpenClaw ऐप बंडल को Accessibility प्रदान करें।                                                                                                        |
| `SYSTEM_RUN_DENIED: approval required` | निष्पादन अनुरोध को स्पष्ट अनुमोदन चाहिए।                                                                                                                                                   |
| `SYSTEM_RUN_DENIED: allowlist miss`    | कमांड अनुमति-सूची मोड द्वारा अवरुद्ध है। Windows Node होस्ट पर, `cmd.exe /c ...` जैसे शेल-रैपर रूपों को अनुमति-सूची मोड में अनुमति-सूची से अनुपस्थित माना जाता है, जब तक कि उन्हें पूछताछ प्रवाह के माध्यम से अनुमोदित न किया जाए। |

## त्वरित पुनर्प्राप्ति क्रम

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
```

यदि समस्या फिर भी बनी रहे:

- डिवाइस युग्मन को फिर से अनुमोदित करें।
- Node ऐप को फिर से खोलें (अग्रभूमि में)।
- OS अनुमतियाँ फिर से प्रदान करें।
- निष्पादन अनुमोदन नीति फिर से बनाएँ/समायोजित करें।

कंप्यूटर नियंत्रण के लिए यह भी सत्यापित करें कि दृष्टि-सक्षम एजेंट `computer` टूल उपलब्ध कराता है, `screen.snapshot` स्क्रीन रिकॉर्डिंग अनुमति के साथ सफल होता है, और `/phone status` आपका इच्छित अस्थायी या स्थायी Gateway प्राधिकरण दिखाता है। `gateway.nodes.commands.deny` प्रविष्टि हमेशा `gateway.nodes.commands.allow` को अधिलेखित करती है।

## संबंधित

- [Nodes का अवलोकन](/hi/nodes)
- [कैमरा Node](/hi/nodes/camera)
- [स्थान कमांड](/hi/nodes/location-command)
- [कंप्यूटर का उपयोग](/hi/nodes/computer-use)
- [निष्पादन अनुमोदन](/hi/tools/exec-approvals)
- [Gateway युग्मन](/hi/gateway/pairing)
- [Gateway समस्या निवारण](/hi/gateway/troubleshooting)
- [चैनल समस्या निवारण](/hi/channels/troubleshooting)
