---
read_when:
    - आप युग्मित नोड्स (कैमरे, स्क्रीन, कैनवास) प्रबंधित कर रहे हैं
    - आपको अनुरोधों को अनुमोदित करना या Node कमांड चलाना होगा
summary: '`openclaw nodes` के लिए CLI संदर्भ (स्थिति, पेयरिंग, इनवोक, कैमरा/कैनवास/स्क्रीन/स्थान/सूचना)'
title: Nodes
x-i18n:
    generated_at: "2026-07-27T17:34:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 53003bcd3d30b0e754aa0717452700595c0cf69d9ecd6301b8a1bf320ea1838a
    source_path: cli/nodes.md
    workflow: 16
---

# `openclaw nodes`

युग्मित Node (डिवाइस) प्रबंधित करें और Node की क्षमताएँ आह्वान करें।

संबंधित: [Node का अवलोकन](/hi/nodes) - [सक्रिय कंप्यूटर उपस्थिति](/hi/nodes/presence) - [कैमरा Node](/hi/nodes/camera) - [इमेज Node](/hi/nodes/images)

प्रत्येक उपकमांड के सामान्य विकल्प: `--url <url>`, `--token <token>`, `--timeout <ms>` (डिफ़ॉल्ट `10000`), `--json`।

## स्थिति

```bash
openclaw nodes status
openclaw nodes status --connected
openclaw nodes status --last-connected 24h
openclaw nodes list
openclaw nodes describe --node <idOrNameOrIp>
```

`status` और `list` दोनों `--connected` (केवल कनेक्ट किए गए Node) और `--last-connected <duration>` (उदाहरण के लिए `24h`, `7d`; केवल वे Node जो इस अवधि के भीतर कनेक्ट हुए थे) स्वीकार करते हैं। `list` लंबित और युग्मित Node को अलग-अलग तालिकाओं में दिखाता है, जहाँ युग्मित पंक्तियों में सबसे हाल के कनेक्शन के बाद बीती अवधि (Last Connect) शामिल होती है; `status` प्रत्येक Node की क्षमता, संस्करण और अंतिम इनपुट के विवरण वाली एक संयुक्त तालिका दिखाता है। कनेक्ट किया गया macOS Node अंतिम इनपुट की जानकारी केवल तब देता है, जब उपयोगकर्ता **सक्रिय कंप्यूटर का पता लगाना** सक्षम करके Accessibility की अनुमति देता है; नवीनतम पंक्ति को `active` से चिह्नित किया जाता है। [सक्रिय कंप्यूटर उपस्थिति](/hi/nodes/presence) देखें। `describe` किसी एक Node की क्षमताएँ, अनुमतियाँ, गतिविधि और प्रभावी/लंबित आह्वान कमांड प्रिंट करता है।

## युग्मन

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name <displayName>
```

ये कमांड Gateway के स्वामित्व वाले `node.pair.*` स्टोर को संचालित करते हैं, जो डिवाइस युग्मन (`openclaw devices approve`) से अलग है और Node के WS `connect` हैंडशेक को नियंत्रित करता है। इन दोनों के संबंध के लिए [Node](/hi/nodes) देखें।

- `remove` Node की युग्मित भूमिका प्रविष्टि निरस्त करता है। डिवाइस-समर्थित Node के लिए, यह डिवाइस युग्मन स्टोर में `node` भूमिका निरस्त करता है और उसके Node-भूमिका सत्रों को डिस्कनेक्ट करता है: मिश्रित-भूमिका वाला डिवाइस अपनी पंक्ति बनाए रखता है और केवल `node` भूमिका खोता है, जबकि केवल Node वाले डिवाइस की पंक्ति हटा दी जाती है। यह मेल खाने वाला कोई भी पुराना Gateway-स्वामित्व वाला Node युग्मन रिकॉर्ड भी साफ़ करता है।
- `pending` के लिए केवल `operator.pairing` स्कोप आवश्यक है।
- `gateway.nodes.pairing.autoApproveCidrs` स्पष्ट रूप से विश्वसनीय, पहली बार किए जाने वाले `role: node` डिवाइस युग्मन के लिए लंबित चरण छोड़ सकता है। डिफ़ॉल्ट रूप से बंद है; भूमिका उन्नयन को अनुमोदित नहीं करता।
- `gateway.nodes.pairing.sshVerify` (डिफ़ॉल्ट रूप से चालू) पहली बार किए जाने वाले `role: node` डिवाइस युग्मन को स्वतः अनुमोदित करता है, जब Gateway SSH के माध्यम से Node होस्ट पर डिवाइस कुंजी सत्यापित कर सकता है; पहली क्षमता सतह को उसी चरण में अनुमोदित किया जाता है। [Node युग्मन](/hi/gateway/pairing#ssh-verified-device-auto-approval-default) देखें।
- `approve` स्कोप की आवश्यकताएँ लंबित अनुरोध में घोषित कमांड का अनुसरण करती हैं:
  - कमांड-रहित अनुरोध: `operator.pairing`
  - सामान्य Node कमांड: `operator.pairing` + `operator.write`
  - एडमिन-संवेदी कमांड (`system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `fs.listDir`, और `system.execApprovals.get/set`): `operator.pairing` + `operator.admin`
- `remove` स्कोप: `operator.pairing` गैर-ऑपरेटर Node पंक्तियाँ हटा सकता है; मिश्रित-भूमिका वाले डिवाइस पर अपनी ही Node भूमिका निरस्त करने वाले डिवाइस-टोकन कॉलर को अतिरिक्त रूप से `operator.admin` की आवश्यकता होती है।

## आह्वान

```bash
openclaw nodes invoke --node <id> --command system.which --params '{"bins":["uname"]}'
```

फ़्लैग:

- `--command <command>` (आवश्यक): उदाहरण के लिए `canvas.eval`।
- `--params <json>`: JSON ऑब्जेक्ट स्ट्रिंग (डिफ़ॉल्ट `{}`)।
- `--invoke-timeout <ms>`: Node आह्वान की समय-सीमा (डिफ़ॉल्ट `15000`)।
- `--idempotency-key <key>`: वैकल्पिक आइडेम्पोटेंसी कुंजी।

`system.run` और `system.run.prepare` यहाँ अवरुद्ध हैं; इसके बजाय शेल निष्पादन के लिए `host=node` के साथ `exec` टूल का उपयोग करें। `system.which` को `invoke` के माध्यम से अनुमति है।

## सूचना, पुश, स्थान, स्क्रीन

```bash
openclaw nodes notify --node <id> --title "Build" --body "Done" --priority timeSensitive
openclaw nodes push --node <id> --title "OpenClaw" --environment sandbox
openclaw nodes location get --node <id> --accuracy precise
openclaw nodes screen record --node <id> --duration 10s --fps 10 --out ./clip.mp4
```

- `notify` ऐसे Node पर स्थानीय सूचना भेजता है जो `system.notify` घोषित करता है, जिसमें macOS, iOS, Android और सीधे watchOS Node शामिल हैं। सीधे watchOS पर डिलीवरी के लिए OpenClaw का सक्रिय होना आवश्यक है। `--title` या `--body` आवश्यक है। विकल्प: `--sound <name>`, `--priority <passive|active|timeSensitive>`, `--delivery <system|overlay|auto>` (डिफ़ॉल्ट `system`), `--invoke-timeout <ms>` (डिफ़ॉल्ट `15000`)।
- `push` किसी iOS Node को APNs परीक्षण पुश भेजता है। विकल्प: `--title <text>` (डिफ़ॉल्ट `OpenClaw`), `--body <text>`, पहचाने गए APNs परिवेश को ओवरराइड करने के लिए `--environment <sandbox|production>`।
- `location get` Node का वर्तमान स्थान प्राप्त करता है। विकल्प: `--max-age <ms>` (कैश किए गए स्थान निर्धारण का पुनः उपयोग), `--accuracy <coarse|balanced|precise>`, `--location-timeout <ms>` (डिफ़ॉल्ट `10000`), `--invoke-timeout <ms>` (डिफ़ॉल्ट `20000`)।
- `screen record` एक छोटी क्लिप कैप्चर करके सहेजा गया पथ प्रिंट करता है (या `--json` के साथ JSON लिखता है)। विकल्प: `--screen <index>` (डिफ़ॉल्ट `0`), `--duration <ms|10s>` (डिफ़ॉल्ट `10000`), `--fps <fps>` (डिफ़ॉल्ट `10`), `--no-audio`, `--out <path>`, `--invoke-timeout <ms>` (डिफ़ॉल्ट `120000`)।

कैमरा और Canvas कमांड के अपने दस्तावेज़ हैं: [कैमरा Node](/hi/nodes/camera), [Canvas](/hi/platforms/mac/canvas)। Canvas को बंडल किए गए प्रयोगात्मक Canvas Plugin द्वारा कार्यान्वित किया गया है; कोर `openclaw nodes canvas` को संगतता माउंट पॉइंट के रूप में बनाए रखता है।

## संबंधित

- [CLI संदर्भ](/hi/cli)
- [Node](/hi/nodes)
