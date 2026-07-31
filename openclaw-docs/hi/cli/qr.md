---
read_when:
    - आप किसी मोबाइल Node ऐप को Gateway के साथ जल्दी से पेयर करना चाहते हैं
    - आपको रिमोट/मैन्युअल साझाकरण के लिए सेटअप-कोड आउटपुट चाहिए
summary: '`openclaw qr` के लिए CLI संदर्भ (मोबाइल पेयरिंग QR + सेटअप कोड जनरेट करें)'
title: QR
x-i18n:
    generated_at: "2026-07-27T19:05:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d60a58126eae7eec5979f28bb511a09fa52b68cdd73727fca0b2de74efa84a
    source_path: cli/qr.md
    workflow: 16
---

# `openclaw qr`

अपने वर्तमान Gateway कॉन्फ़िगरेशन से मोबाइल पेयरिंग QR और सेटअप कोड जनरेट करें।

```bash
openclaw qr
openclaw qr --setup-code-only
openclaw qr --json
openclaw qr --remote
openclaw qr --limited
openclaw qr --url wss://gateway.example/ws
```

आधिकारिक OpenClaw iOS और Android ऐप तब अपने-आप कनेक्ट हो जाते हैं, जब उनका
सेटअप-कोड मेटाडेटा मेल खाता है। यदि कोई अनुरोध लंबित रहता है (उदाहरण के लिए,
किसी गैर-आधिकारिक क्लाइंट या मेल न खाने वाले मेटाडेटा के कारण), तो उसकी समीक्षा करके उसे स्वीकृत करें:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

## विकल्प

- `--remote`: `gateway.remote.url` को प्राथमिकता दें; यदि वह URL सेट नहीं है, तो `gateway.tailscale.mode=serve|funnel` का उपयोग करता है। `device-pair` Plugin `publicUrl` को अनदेखा करता है।
- `--url <url>`: पेलोड में उपयोग किए गए Gateway URL को ओवरराइड करें
- `--public-url <url>`: पेलोड में उपयोग किए गए सार्वजनिक URL को ओवरराइड करें
- `--token <token>`: उस Gateway टोकन को ओवरराइड करें जिसके विरुद्ध बूटस्ट्रैप प्रवाह प्रमाणित होता है
- `--password <password>`: उस Gateway पासवर्ड को ओवरराइड करें जिसके विरुद्ध बूटस्ट्रैप प्रवाह प्रमाणित होता है
- `--limited`: हस्तांतरित ऑपरेटर टोकन से प्रशासनिक Gateway एक्सेस हटाएँ
- `--setup-code-only`: केवल सेटअप कोड प्रिंट करें
- `--no-ascii`: ASCII QR रेंडरिंग छोड़ें
- `--json`: JSON उत्सर्जित करें (`setupCode`, `gatewayUrl`, वैकल्पिक `gatewayUrls`, `auth`, `access`, वैकल्पिक `accessDowngraded`, `urlSource`)

`--token` और `--password` परस्पर अनन्य हैं।

## सेटअप कोड की सामग्री

सेटअप कोड में साझा Gateway टोकन/पासवर्ड नहीं, बल्कि एक अपारदर्शी, अल्पकालिक `bootstrapToken` होता है। किसी `wss://` एंडपॉइंट (या समान-होस्ट लूपबैक) के लिए, डिफ़ॉल्ट बूटस्ट्रैप प्रवाह जारी करता है:

- `scopes: []` वाला एक प्राथमिक `node` टोकन
- `operator.admin`, `operator.approvals`, `operator.read`, `operator.talk.secrets`, और `operator.write` वाला एक पूर्ण नेटिव-मोबाइल `operator` हस्तांतरण टोकन

ऑपरेटर हस्तांतरण से `operator.admin` हटाते हुए उसी Node टोकन को बनाए रखने के लिए `--limited` का उपयोग करें। पेयरिंग-म्यूटेशन स्कोप कभी भी सेटअप कोड द्वारा हस्तांतरित नहीं किया जाता।

प्लेनटेक्स्ट LAN `ws://` सेटअप उपलब्ध रहता है, लेकिन OpenClaw स्वचालित रूप से
सीमित प्रोफ़ाइल का उपयोग करता है, क्योंकि कोई नेटवर्क पर्यवेक्षक बेयरर बूटस्ट्रैप
टोकन को कैप्चर करके उससे पहले अनुरोध कर सकता है। पूर्ण एक्सेस पाने के लिए `wss://` या Tailscale Serve कॉन्फ़िगर करें, फिर नया कोड
जनरेट करें।

## Gateway URL निर्धारण

Tailscale/सार्वजनिक `ws://` Gateway URL के लिए मोबाइल पेयरिंग सुरक्षित रूप से विफल होती है: इनके लिए Tailscale Serve/Funnel या `wss://` Gateway URL का उपयोग करें। निजी LAN पते और `.local` Bonjour होस्ट प्लेन `ws://` पर समर्थित रहते हैं, जिनमें ऊपर बताए अनुसार सीमित ऑपरेटर एक्सेस मिलता है।

जब चयनित Gateway URL `gateway.bind=lan` से आता है, तो OpenClaw स्थायी `tailscale serve status --json` रूट की भी जाँच करता है। सक्रिय Gateway के लूपबैक पोर्ट को प्रॉक्सी करने वाला कोई भी HTTPS Serve रूट फ़ॉलबैक के रूप में शामिल किया जाता है। QR कमांड यह फ़ॉलबैक केवल `lan` के लिए जोड़ता है; `custom` और `tailnet` अपने स्पष्ट रूप से विज्ञापित रूट बनाए रखते हैं। वर्तमान iOS क्लाइंट विज्ञापित रूट की क्रम से जाँच करते हैं और पहुँच योग्य पहले रूट को सहेजते हैं; पुराने क्लाइंट के लिए लेगेसी `url` फ़ील्ड अपरिवर्तित रहता है।

`--remote` के साथ, `gateway.remote.url` या `gateway.tailscale.mode=serve|funnel` में से एक आवश्यक है।

## प्रमाणीकरण निर्धारण (`--remote` के बिना)

जब कोई CLI प्रमाणीकरण ओवरराइड पास नहीं किया जाता, तो स्थानीय Gateway प्रमाणीकरण SecretRefs का निर्धारण इस प्रकार होता है:

| शर्त                                                                                                                    | निर्धारित मान                                  |
| ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `gateway.auth.mode="token"`, या ऐसा अनुमानित मोड जिसमें कोई प्रभावी पासवर्ड स्रोत नहीं है                                                | `gateway.auth.token`                      |
| `gateway.auth.mode="password"`, या ऐसा अनुमानित मोड जिसमें प्रमाणीकरण/परिवेश से कोई प्रभावी टोकन नहीं है                                         | `gateway.auth.password`                   |
| `gateway.auth.token` और `gateway.auth.password` दोनों कॉन्फ़िगर हैं (SecretRefs सहित) और `gateway.auth.mode` सेट नहीं है | विफल होता है; `gateway.auth.mode` स्पष्ट रूप से सेट करें |

## प्रमाणीकरण निर्धारण (`--remote`)

यदि प्रभावी रूप से सक्रिय रिमोट क्रेडेंशियल SecretRefs के रूप में कॉन्फ़िगर हैं और न तो `--token` न ही `--password` पास किया गया है, तो कमांड सक्रिय Gateway स्नैपशॉट से उनका निर्धारण करता है। यदि Gateway उपलब्ध नहीं है, तो कमांड तुरंत विफल हो जाता है।

<Note>
इस कमांड पथ के लिए ऐसा Gateway आवश्यक है जो `secrets.resolve` RPC विधि का समर्थन करता हो। पुराने Gateway अज्ञात-विधि त्रुटि लौटाते हैं।
</Note>

## संबंधित

- [CLI संदर्भ](/hi/cli)
- [डिवाइस](/hi/cli/devices)
- [पेयरिंग](/hi/cli/pairing)
