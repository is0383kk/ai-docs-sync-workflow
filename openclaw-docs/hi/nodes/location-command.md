---
read_when:
    - लोकेशन Node समर्थन या अनुमतियों का UI जोड़ना
    - Android स्थान अनुमतियों या फ़ोरग्राउंड व्यवहार को डिज़ाइन करना
summary: Node के लिए स्थान कमांड, प्लेटफ़ॉर्म अनुमति मोड और Linux GeoClue सेटअप
title: स्थान कमांड
x-i18n:
    generated_at: "2026-07-27T20:03:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 644229c1eafc8fc7b59bc23ba01d4ba95687ea66c4f9bd4a4cda98a87f2b6085
    source_path: nodes/location-command.md
    workflow: 16
---

## TL;DR

- `location.get` एक Node कमांड है, जिसे `node.invoke` या `openclaw nodes location get` के माध्यम से चलाया जाता है।
- डिफ़ॉल्ट रूप से बंद।
- Android तृतीय-पक्ष बिल्ड एक चयनकर्ता का उपयोग करते हैं: बंद / उपयोग करते समय / हमेशा। Play बिल्ड में बंद / उपयोग करते समय विकल्प बने रहते हैं।
- सटीक स्थान एक अलग टॉगल है।

## चयनकर्ता क्यों (केवल स्विच क्यों नहीं)

OS की स्थान अनुमतियों के कई स्तर होते हैं। सटीक स्थान भी OS की एक अलग अनुमति है (iOS 14+ में "Precise", Android में "fine" बनाम "coarse")। ऐप के भीतर का चयनकर्ता अनुरोधित मोड निर्धारित करता है, लेकिन वास्तविक अनुमति का निर्णय फिर भी OS ही करता है।

## सेटिंग मॉडल

प्रति Node डिवाइस:

- `location.enabledMode`: `off | whileUsing | always`
- `location.preciseEnabled`: bool

UI व्यवहार:

- `whileUsing` चुनने पर अग्रभूमि अनुमति का अनुरोध किया जाता है।
- Android तृतीय-पक्ष बिल्ड में `always` चुनने पर पहले अग्रभूमि अनुमति का अनुरोध किया जाता है, फिर पृष्ठभूमि पहुँच की व्याख्या की जाती है और इसके बाद अलग **Allow all the time** अनुमति के लिए Android ऐप सेटिंग खोली जाती हैं।
- Android Play बिल्ड पृष्ठभूमि स्थान अनुमति घोषित नहीं करते और `always` नहीं दिखाते।
- यदि OS अनुरोधित स्तर अस्वीकार कर देता है, तो ऐप उपलब्ध उच्चतम अनुमत स्तर पर वापस आ जाता है और स्थिति दिखाता है।

## अनुमतियों का मैपिंग (node.permissions)

वैकल्पिक। macOS Node, `node.list`/`node.describe` पर `permissions` मैप के माध्यम से `location` की रिपोर्ट करता है; iOS/Android इसे छोड़ सकते हैं।

## कमांड: `location.get`

`node.invoke` या CLI सहायक के माध्यम से कॉल किया जाता है:

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

पैरामीटर:

```json
{
  "timeoutMs": 10000,
  "maxAgeMs": 15000,
  "desiredAccuracy": "coarse|balanced|precise"
}
```

CLI फ़्लैग सीधे मैप होते हैं: `--location-timeout` -> `timeoutMs`, `--max-age` -> `maxAgeMs`, `--accuracy` -> `desiredAccuracy`।

प्रतिक्रिया पेलोड:

```json
{
  "lat": 48.20849,
  "lon": 16.37208,
  "accuracyMeters": 12.5,
  "altitudeMeters": 182.0,
  "speedMps": 0.0,
  "headingDeg": 270.0,
  "timestamp": "2026-01-03T12:34:56.000Z",
  "isPrecise": true,
  "source": "gps|wifi|cell|unknown"
}
```

त्रुटियाँ (स्थिर कोड):

- `LOCATION_DISABLED`: चयनकर्ता बंद है।
- `LOCATION_PERMISSION_REQUIRED`: अनुरोधित मोड की अनुमति उपलब्ध नहीं है।
- `LOCATION_BACKGROUND_UNAVAILABLE`: ऐप पृष्ठभूमि में है, लेकिन केवल उपयोग करते समय की अनुमति दी गई है।
- `LOCATION_TIMEOUT`: समय पर स्थान निर्धारण नहीं हुआ।
- `LOCATION_UNAVAILABLE`: सिस्टम विफलता या कोई प्रदाता उपलब्ध नहीं।

## पृष्ठभूमि व्यवहार

- Android तृतीय-पक्ष बिल्ड पृष्ठभूमि में `location.get` केवल तभी स्वीकार करते हैं, जब उपयोगकर्ता ने `Always` चुना हो और Android ने पृष्ठभूमि स्थान की अनुमति दी हो। मौजूदा स्थायी Node सेवा `location` सेवा प्रकार जोड़ती है और सक्रिय रहते समय `Location: Always` की सूचना देती है।
- Android Play बिल्ड और `While Using` मोड पृष्ठभूमि में होने पर `location.get` को अस्वीकार करते हैं।
- अन्य Node प्लेटफ़ॉर्म का व्यवहार अलग हो सकता है।

## Linux Node होस्ट

बंडल किया गया Linux Node Plugin, CLI `openclaw node` सेवा में `location.get` जोड़ता है, जिसमें Linux डेस्कटॉप ऐप के बिना चलने वाले हेडलेस होस्ट भी शामिल हैं। स्थान डिफ़ॉल्ट रूप से बंद रहता है। इसे Plugin प्रविष्टि में सक्षम करें, फिर Node सेवा पुनः आरंभ करें:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          location: { enabled: true },
        },
      },
    },
  },
}
```

GeoClue2 और उसका `where-am-i` डेमो इंस्टॉल करें (Debian और Ubuntu पर `geoclue-2-demo`)। Node सेवा के उपयोगकर्ता को होस्ट की GeoClue नीति और प्राधिकरण एजेंट द्वारा अनुमति मिलनी चाहिए।

Plugin, `busctl` कॉल के क्रम के बजाय `where-am-i` का उपयोग करता है। GeoClue क्लाइंट निर्माण, प्रॉपर्टी, आरंभ, अपडेट और रोकने की प्रक्रिया को एक D-Bus क्लाइंट कनेक्शन से बाँधता है; डेमो इस जीवनचक्र को एक साथ रखता है, जबकि अलग-अलग `busctl` उपप्रक्रियाएँ ऐसा नहीं करतीं। कोई npm निर्भरता नहीं जोड़ी जाती।

Linux, `coarse`, `balanced` और `precise` को GeoClue सटीकता स्तरों `4`, `6` और `8` से मैप करता है। यह लौटाए गए टाइमस्टैम्प के आधार पर `maxAgeMs` को सत्यापित करता है। GeoClue का डेमो चुने गए प्रदाता को प्रकट नहीं करता, इसलिए `source` का मान `unknown` होता है; `isPrecise` केवल तभी true होता है, जब रिपोर्ट की गई सटीकता 100 मीटर या उससे बेहतर हो।

Linux इन्हीं स्थिर त्रुटियों का उपयोग करता है: `LOCATION_DISABLED`, `LOCATION_TIMEOUT` और `LOCATION_UNAVAILABLE`।

## मॉडल/टूलिंग एकीकरण

- एजेंट टूल: `nodes` टूल की `location_get` कार्रवाई (Node आवश्यक)।
- CLI: `openclaw nodes location get --node <id>`।
- एजेंट दिशानिर्देश: केवल तभी कॉल करें, जब उपयोगकर्ता ने स्थान सक्षम किया हो और वह इसके दायरे को समझता हो।

## UX पाठ (सुझाया गया)

- बंद: "स्थान साझा करना अक्षम है।"
- उपयोग करते समय: "केवल OpenClaw खुला होने पर।"
- हमेशा: "OpenClaw के पृष्ठभूमि में रहने के दौरान अनुरोधित स्थान जाँच की अनुमति दें।"
- सटीक: "सटीक GPS स्थान का उपयोग करें। अनुमानित स्थान साझा करने के लिए टॉगल बंद करें।"

## संबंधित

- [Nodes का अवलोकन](/hi/nodes)
- [चैनल स्थान पार्सिंग](/hi/channels/location)
- [कैमरा कैप्चर](/hi/nodes/camera)
- [वार्ता मोड](/hi/nodes/talk)
