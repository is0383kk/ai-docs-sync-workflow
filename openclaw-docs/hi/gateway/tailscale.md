---
read_when:
    - Gateway Control UI को localhost के बाहर उपलब्ध कराना
    - टेलनेट या सार्वजनिक डैशबोर्ड एक्सेस को स्वचालित करना
summary: Gateway डैशबोर्ड के लिए एकीकृत Tailscale Serve/Funnel
title: Tailscale
x-i18n:
    generated_at: "2026-07-27T19:53:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e201a64ac427994401fae1b934d94e0c5afe976b4acd34d45b059978f5f1807e
    source_path: gateway/tailscale.md
    workflow: 16
---

OpenClaw, Gateway डैशबोर्ड और WebSocket पोर्ट के लिए Tailscale **Serve** (tailnet) या **Funnel** (सार्वजनिक) को स्वतः कॉन्फ़िगर कर सकता है। इससे Gateway लूपबैक से बंधा रहता है, जबकि Tailscale HTTPS, रूटिंग और (Serve के लिए) पहचान हेडर प्रदान करता है।

## मोड

`gateway.tailscale.mode`:

| मोड            | व्यवहार                                                                    |
| --------------- | --------------------------------------------------------------------------- |
| `serve`         | `tailscale serve` के माध्यम से केवल-tailnet Serve। Gateway `127.0.0.1` पर बना रहता है। |
| `funnel`        | `tailscale funnel` के माध्यम से सार्वजनिक HTTPS। साझा पासवर्ड आवश्यक है।            |
| `off` (डिफ़ॉल्ट) | कोई Tailscale स्वचालन नहीं।                                                    |

स्थिति और ऑडिट आउटपुट इस OpenClaw Serve/Funnel मोड के लिए **Tailscale एक्सपोज़र** का उपयोग करते हैं। `off` का अर्थ है कि OpenClaw, Serve या Funnel को प्रबंधित नहीं कर रहा है; इसका अर्थ यह नहीं है कि स्थानीय Tailscale डेमन बंद है या उससे लॉग आउट किया गया है।

## कॉन्फ़िगरेशन उदाहरण

### केवल-tailnet (Serve)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" },
  },
}
```

खोलें: `https://<magicdns>/` (या आपका कॉन्फ़िगर किया गया `gateway.controlUi.basePath`)

डिवाइस होस्टनेम के बजाय किसी नामित Tailscale Service के माध्यम से Control UI को उपलब्ध कराने के लिए, `gateway.tailscale.serviceName` को Service नाम पर सेट करें:

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve", serviceName: "svc:openclaw" },
  },
}
```

इसके बाद स्टार्टअप डिवाइस होस्टनेम के बजाय Service URL को `https://openclaw.<tailnet-name>.ts.net/` के रूप में रिपोर्ट करता है। Tailscale Services के लिए होस्ट का आपके tailnet में स्वीकृत टैग किया हुआ Node होना आवश्यक है—इसे सक्षम करने से पहले Tailscale में टैग कॉन्फ़िगर करें और Service को स्वीकृत करें, अन्यथा Gateway स्टार्टअप के दौरान `tailscale serve --service=...` विफल हो जाता है।

### केवल-tailnet (Tailnet IP से बाइंड करें)

Gateway को बिना Serve/Funnel के सीधे Tailnet IP पर सुनने के लिए इसका उपयोग करें:

```json5
{
  gateway: {
    bind: "tailnet",
    auth: { mode: "token", token: "your-token" },
  },
}
```

किसी अन्य Tailnet डिवाइस से कनेक्ट करें:

- Control UI: `http://<tailscale-ip>:18789/`
- WebSocket: `ws://<tailscale-ip>:18789`

<Note>
जब बाइंड किया जा सकने वाला Tailnet IPv4 उपलब्ध होता है, तब Gateway को प्रमाणित समान-होस्ट क्लाइंट के लिए `http://127.0.0.1:18789` भी आवश्यक होता है। यदि स्टार्टअप पर कोई Tailnet पता उपलब्ध नहीं है, तो यह केवल लूपबैक पर वापस चला जाता है; प्रत्यक्ष Tailnet एक्सेस जोड़ने के लिए Tailscale उपलब्ध होने के बाद पुनः आरंभ करें। कोई भी पथ LAN या सार्वजनिक एक्सपोज़र नहीं जोड़ता।
</Note>

### सार्वजनिक इंटरनेट (Funnel + साझा पासवर्ड)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "funnel" },
    auth: { mode: "password", password: "replace-me" },
  },
}
```

पासवर्ड को डिस्क पर कमिट करने के बजाय `OPENCLAW_GATEWAY_PASSWORD` को प्राथमिकता दें।

## CLI उदाहरण

```bash
openclaw gateway --tailscale serve
openclaw gateway --tailscale funnel --auth password
```

## प्रमाणीकरण

`gateway.auth.mode` हैंडशेक को नियंत्रित करता है:

| मोड                                                   | उपयोग का मामला                                                                            |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| `none`                                                 | केवल निजी इनग्रेस                                                                |
| `token` (`OPENCLAW_GATEWAY_TOKEN` सेट होने पर डिफ़ॉल्ट) | साझा टोकन                                                                        |
| `password`                                             | `OPENCLAW_GATEWAY_PASSWORD` या कॉन्फ़िगरेशन के माध्यम से साझा सीक्रेट                             |
| `trusted-proxy`                                        | पहचान-जागरूक रिवर्स प्रॉक्सी; [विश्वसनीय प्रॉक्सी प्रमाणीकरण](/hi/gateway/trusted-proxy-auth) देखें |

### Tailscale पहचान हेडर (केवल Serve)

जब `tailscale.mode: "serve"` और `gateway.auth.allowTailscale`, `true` होता है, तब Control UI/WebSocket प्रमाणीकरण टोकन/पासवर्ड के बजाय Tailscale पहचान हेडर (`tailscale-user-login`) का उपयोग कर सकता है। OpenClaw अनुरोध के `x-forwarded-for` पते को स्थानीय Tailscale डेमन (`tailscale whois`) के माध्यम से रिज़ॉल्व करके और उसे स्वीकार करने से पहले हेडर लॉगिन से मिलान करके हेडर को सत्यापित करता है। कोई अनुरोध इस पथ के लिए केवल तभी योग्य होता है, जब वह Tailscale के `x-forwarded-for`, `x-forwarded-proto`, और `x-forwarded-host` हेडर के साथ लूपबैक से आता है।

यह टोकन-रहित प्रवाह मानता है कि Gateway होस्ट विश्वसनीय है। यदि अविश्वसनीय स्थानीय कोड उसी होस्ट पर चल सकता है, तो `gateway.auth.allowTailscale: false` सेट करें और इसके बजाय टोकन/पासवर्ड प्रमाणीकरण आवश्यक करें।

बायपास का दायरा:

- केवल Control UI WebSocket प्रमाणीकरण सतह पर लागू होता है। HTTP API एंडपॉइंट (`/v1/*`, `/tools/invoke`, `/api/channels/*`, आदि) कभी भी Tailscale पहचान-हेडर प्रमाणीकरण का उपयोग नहीं करते; वे हमेशा Gateway के सामान्य HTTP प्रमाणीकरण मोड का पालन करते हैं।
- पहले से ब्राउज़र डिवाइस पहचान वाली Control UI ऑपरेटर सत्रों के लिए, सत्यापित Tailscale पहचान बूटस्ट्रैप-टोकन/QR पेयरिंग राउंड ट्रिप को छोड़ देती है।
- यह स्वयं डिवाइस पहचान को बायपास नहीं करता: डिवाइस-रहित क्लाइंट अभी भी अस्वीकार किए जाते हैं, और Node-भूमिका कनेक्शन अभी भी सामान्य पेयरिंग और प्रमाणीकरण जाँचों से गुजरते हैं।

## टिप्पणियाँ

- Tailscale Serve/Funnel के लिए `tailscale` CLI का इंस्टॉल और लॉग इन होना आवश्यक है।
- सार्वजनिक एक्सपोज़र से बचने के लिए, `tailscale.mode: "funnel"` तब तक शुरू होने से इनकार करता है, जब तक प्रमाणीकरण मोड `password` न हो।
- `gateway.tailscale.serviceName` केवल Serve मोड पर लागू होता है और `tailscale serve --service=<name>` को पास किया जाता है। मान को Tailscale के `svc:<dns-label>` प्रारूप का उपयोग करना आवश्यक है, उदाहरण के लिए `svc:openclaw`। Tailscale के लिए Service होस्ट का टैग किया हुआ Node होना आवश्यक है, और Serve द्वारा इसे प्रकाशित किए जाने से पहले Service को एडमिन-कंसोल की स्वीकृति की आवश्यकता हो सकती है।
- `gateway.tailscale.resetOnExit` शटडाउन पर `tailscale serve`/`tailscale funnel` कॉन्फ़िगरेशन को पूर्ववत करता है।
- `gateway.tailscale.preserveFunnel: true` बाहरी रूप से कॉन्फ़िगर किए गए `tailscale funnel` रूट को Gateway के पुनः आरंभों के दौरान सक्रिय रखता है। `mode: "serve"` के साथ, OpenClaw Serve को दोबारा लागू करने से पहले `tailscale funnel status` की जाँच करता है और जब कोई Funnel रूट पहले से Gateway पोर्ट को कवर करता है, तो उसे छोड़ देता है। OpenClaw द्वारा प्रबंधित Funnel की केवल-पासवर्ड नीति अपरिवर्तित रहती है।
- जब Tailnet IPv4 उपलब्ध होता है, तो `gateway.bind: "tailnet"` प्रत्यक्ष Tailnet बाइंड (कोई HTTPS नहीं, कोई Serve/Funnel नहीं) के साथ आवश्यक स्थानीय `127.0.0.1` का उपयोग करता है; अन्यथा यह केवल लूपबैक पर वापस चला जाता है।
- `gateway.bind: "auto"` लूपबैक को प्राथमिकता देता है; समान-होस्ट लूपबैक एक्सेस बनाए रखते हुए नेटवर्क एक्सपोज़र को Tailnet तक सीमित करने के लिए `tailnet` का उपयोग करें।
- Serve/Funnel केवल **Gateway Control UI + WS** को उपलब्ध कराते हैं। Node उसी Gateway WS एंडपॉइंट पर कनेक्ट होते हैं, इसलिए Serve Node एक्सेस के लिए भी काम करता है।

### Tailscale की पूर्वापेक्षाएँ और सीमाएँ

- Serve के लिए आपके tailnet पर HTTPS सक्षम होना आवश्यक है; यदि यह अनुपलब्ध है, तो CLI आपको संकेत देता है।
- Serve, Tailscale पहचान हेडर इंजेक्ट करता है; Funnel ऐसा नहीं करता।
- Funnel के लिए Tailscale v1.38.3+, MagicDNS, सक्षम HTTPS और funnel Node एट्रिब्यूट आवश्यक हैं।
- Funnel, TLS पर केवल पोर्ट `443`, `8443`, और `10000` का समर्थन करता है।
- macOS पर Funnel के लिए Tailscale ऐप का ओपन-सोर्स संस्करण आवश्यक है।

## ब्राउज़र नियंत्रण (दूरस्थ Gateway + स्थानीय ब्राउज़र)

Gateway को एक मशीन पर चलाने, लेकिन ब्राउज़र को दूसरी मशीन पर नियंत्रित करने के लिए, ब्राउज़र मशीन पर **Node होस्ट** चलाएँ और दोनों को एक ही tailnet पर रखें। Gateway ब्राउज़र कार्रवाइयों को Node तक प्रॉक्सी करता है; किसी अलग नियंत्रण सर्वर या Serve URL की आवश्यकता नहीं है।

ब्राउज़र नियंत्रण के लिए Funnel से बचें; Node पेयरिंग को ऑपरेटर एक्सेस की तरह मानें।

## अधिक जानें

- Tailscale Serve का अवलोकन: [https://tailscale.com/kb/1312/serve](https://tailscale.com/kb/1312/serve)
- `tailscale serve` कमांड: [https://tailscale.com/kb/1242/tailscale-serve](https://tailscale.com/kb/1242/tailscale-serve)
- Tailscale Funnel का अवलोकन: [https://tailscale.com/kb/1223/tailscale-funnel](https://tailscale.com/kb/1223/tailscale-funnel)
- `tailscale funnel` कमांड: [https://tailscale.com/kb/1311/tailscale-funnel](https://tailscale.com/kb/1311/tailscale-funnel)

## संबंधित

- [दूरस्थ एक्सेस](/hi/gateway/remote)
- [खोज](/hi/gateway/discovery)
- [प्रमाणीकरण](/hi/gateway/authentication)
