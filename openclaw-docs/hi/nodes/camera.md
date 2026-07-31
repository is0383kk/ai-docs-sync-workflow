---
read_when:
    - Node प्लेटफ़ॉर्म पर कैमरा कैप्चर जोड़ना या संशोधित करना
    - एजेंट-सुलभ MEDIA अस्थायी-फ़ाइल कार्यप्रवाहों का विस्तार करना
summary: फ़ोटो और छोटे वीडियो क्लिप के लिए iOS, Android, macOS और Linux Node पर कैमरा कैप्चर
title: कैमरा कैप्चर
x-i18n:
    generated_at: "2026-07-27T18:00:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b819f7ff3fc9b51757ae998d27f540975bf6c1194ed32fd36b1fbe909e79400c
    source_path: nodes/camera.md
    workflow: 16
---

OpenClaw युग्मित **iOS**, **Android**, **macOS**, और **Linux** Node पर एजेंट कार्यप्रवाहों के लिए कैमरा कैप्चर का समर्थन करता है: Gateway `node.invoke` के माध्यम से फ़ोटो (`jpg`) या छोटी वीडियो क्लिप (`mp4`, वैकल्पिक ऑडियो सहित) कैप्चर करें।

सभी कैमरा एक्सेस प्रत्येक प्लेटफ़ॉर्म पर उपयोगकर्ता-नियंत्रित सेटिंग द्वारा नियंत्रित होते हैं।

## iOS Node

### iOS उपयोगकर्ता सेटिंग

- iOS Settings tab → **Camera** → **Allow Camera** (`camera.enabled`)।
  - डिफ़ॉल्ट: **चालू** (कुंजी अनुपस्थित होने पर इसे सक्षम माना जाता है)।
  - बंद होने पर: `camera.*` कमांड `CAMERA_DISABLED` लौटाते हैं।

### iOS कमांड (Gateway `node.invoke` के माध्यम से)

- `camera.list`
  - प्रतिक्रिया पेलोड: `devices` — `{ id, name, position, deviceType }` की सरणी।

- `camera.snap`
  - पैरामीटर:
    - `facing`: `front|back` (डिफ़ॉल्ट: `front`)
    - `maxWidth`: संख्या (वैकल्पिक; डिफ़ॉल्ट `1600`)
    - `quality`: `0..1` (वैकल्पिक; डिफ़ॉल्ट `0.9`, `[0.05, 1.0]` तक सीमित)
    - `format`: वर्तमान में `jpg`
    - `delayMs`: संख्या (वैकल्पिक; डिफ़ॉल्ट `0`, आंतरिक रूप से अधिकतम `10000`)
    - `deviceId`: स्ट्रिंग (वैकल्पिक; `camera.list` से)
  - प्रतिक्रिया पेलोड: `format: "jpg"`, `base64`, `width`, `height`।
  - पेलोड सुरक्षा सीमा: base64-एन्कोडेड पेलोड को 5MB से कम रखने के लिए फ़ोटो फिर से संपीड़ित किए जाते हैं।

- `camera.clip`
  - पैरामीटर:
    - `facing`: `front|back` (डिफ़ॉल्ट: `front`)
    - `durationMs`: संख्या (डिफ़ॉल्ट `3000`, `[250, 60000]` तक सीमित)
    - `includeAudio`: बूलियन (डिफ़ॉल्ट `true`)
    - `format`: वर्तमान में `mp4`
    - `deviceId`: स्ट्रिंग (वैकल्पिक; `camera.list` से)
  - प्रतिक्रिया पेलोड: `format: "mp4"`, `base64`, `durationMs`, `hasAudio`।

### iOS फ़ोरग्राउंड आवश्यकता

`canvas.*` की तरह, iOS Node केवल **फ़ोरग्राउंड** में `camera.*` कमांड की अनुमति देता है। बैकग्राउंड से किए गए आह्वान `NODE_BACKGROUND_UNAVAILABLE` लौटाते हैं।

### CLI सहायक

मीडिया फ़ाइलें प्राप्त करने का सबसे आसान तरीका CLI सहायक है, जो डीकोड किए गए मीडिया को अस्थायी फ़ाइल में लिखता है और सहेजा गया पथ प्रिंट करता है।

```bash
openclaw nodes camera snap --node <id>                 # डिफ़ॉल्ट: सामने + पीछे दोनों (2 MEDIA पंक्तियाँ)
openclaw nodes camera snap --node <id> --facing front
openclaw nodes camera clip --node <id> --duration 3000
openclaw nodes camera clip --node <id> --no-audio
```

`nodes camera snap` का डिफ़ॉल्ट `--facing both` है, जो एजेंट को दोनों दृश्य देने के लिए सामने और पीछे दोनों से कैप्चर करता है; एक स्पष्ट दिशा के साथ `--device-id` दें (`--device-id` सेट होने पर `both` अस्वीकार किया जाता है)। जब तक आप अपना रैपर नहीं बनाते, आउटपुट फ़ाइलें अस्थायी होती हैं (OS की अस्थायी डायरेक्टरी में)।

## Android Node

### Android उपयोगकर्ता सेटिंग

- Android Settings sheet → **Camera** → **Allow Camera** (`camera.enabled`)।
  - **नई इंस्टॉलेशन में डिफ़ॉल्ट रूप से बंद।** इस सेटिंग से पहले की मौजूदा इंस्टॉलेशन को **चालू** पर माइग्रेट किया जाता है, ताकि अपग्रेड के दौरान पहले से काम कर रहा कैमरा एक्सेस चुपचाप न खो जाए।
  - बंद होने पर: `camera.*` कमांड `CAMERA_DISABLED: enable Camera in Settings` लौटाते हैं।

### अनुमतियाँ

- `CAMERA` की आवश्यकता `camera.snap` और `camera.clip` दोनों के लिए होती है; अनुमति अनुपस्थित या अस्वीकृत होने पर `CAMERA_PERMISSION_REQUIRED` लौटाया जाता है।
- जब `includeAudio`, `true` हो, तब `camera.clip` के लिए `RECORD_AUDIO` आवश्यक है; अनुमति अनुपस्थित या अस्वीकृत होने पर `MIC_PERMISSION_REQUIRED` लौटाया जाता है।

संभव होने पर ऐप रनटाइम अनुमतियों के लिए संकेत देता है।

### Android फ़ोरग्राउंड आवश्यकता

`canvas.*` की तरह, Android Node केवल **फ़ोरग्राउंड** में `camera.*` कमांड की अनुमति देता है। बैकग्राउंड से किए गए आह्वान `NODE_BACKGROUND_UNAVAILABLE: command requires foreground` लौटाते हैं।

### Android कमांड (Gateway `node.invoke` के माध्यम से)

- `camera.list`
  - प्रतिक्रिया पेलोड: `devices` — `{ id, name, position, deviceType }` की सरणी।

- `camera.snap`
  - पैरामीटर: `facing` (`front|back`, डिफ़ॉल्ट `front`), `quality` (डिफ़ॉल्ट `0.95`, `[0.1, 1.0]` तक सीमित), `maxWidth` (डिफ़ॉल्ट `1600`), `deviceId` (वैकल्पिक; अज्ञात आईडी `INVALID_REQUEST` के साथ विफल होती है)।
  - प्रतिक्रिया पेलोड: `format: "jpg"`, `base64`, `width`, `height`।
  - पेलोड सुरक्षा सीमा: base64 को 5MB से कम रखने के लिए फिर से संपीड़ित किया जाता है (iOS के समान सीमा)।

- `camera.clip`
  - पैरामीटर: `facing` (डिफ़ॉल्ट `front`), `durationMs` (डिफ़ॉल्ट `3000`, `[200, 60000]` तक सीमित), `includeAudio` (डिफ़ॉल्ट `true`), `deviceId` (वैकल्पिक)।
  - प्रतिक्रिया पेलोड: `format: "mp4"`, `base64`, `durationMs`, `hasAudio`।
  - पेलोड सुरक्षा सीमा: base64 एन्कोडिंग से पहले मूल MP4 अधिकतम 18MB तक सीमित है; अधिक आकार वाली क्लिप `PAYLOAD_TOO_LARGE` के साथ विफल होती हैं (`durationMs` घटाएँ और पुनः प्रयास करें)।

## macOS ऐप

### macOS उपयोगकर्ता सेटिंग

macOS सहायक ऐप एक चेकबॉक्स उपलब्ध कराता है:

- **Settings → General → Allow Camera** (`openclaw.cameraEnabled`)।
  - डिफ़ॉल्ट: **बंद**।
  - बंद होने पर: कैमरा अनुरोध `CAMERA_DISABLED: enable Camera in Settings` लौटाते हैं।

### CLI सहायक (Node आह्वान)

macOS Node पर कैमरा कमांड आह्वान करने के लिए मुख्य `openclaw` CLI का उपयोग करें।

```bash
openclaw nodes camera list --node <id>                     # कैमरा आईडी सूचीबद्ध करता है
openclaw nodes camera snap --node <id>                     # सहेजा गया पथ प्रिंट करता है
openclaw nodes camera snap --node <id> --max-width 1280
openclaw nodes camera snap --node <id> --delay-ms 2000
openclaw nodes camera snap --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --duration 10s       # सहेजा गया पथ प्रिंट करता है
openclaw nodes camera clip --node <id> --duration-ms 3000   # सहेजा गया पथ प्रिंट करता है (पुराना फ़्लैग)
openclaw nodes camera clip --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --no-audio
```

- जब तक ओवरराइड न किया जाए, `openclaw nodes camera snap` का डिफ़ॉल्ट `maxWidth=1600` होता है।
- `camera.snap` कैप्चर करने से पहले वार्म-अप/एक्सपोज़र स्थिर होने के बाद `delayMs` (डिफ़ॉल्ट 2000ms, `[0, 10000]` तक सीमित) प्रतीक्षा करता है।
- base64 को 5MB से कम रखने के लिए फ़ोटो पेलोड फिर से संपीड़ित किए जाते हैं।

## Linux Node होस्ट

बंडल किया गया Linux Node Plugin CLI `openclaw node` सेवा में कैमरा कैप्चर जोड़ता है। यह हेडलेस होस्ट पर काम करता है और इसके लिए Linux डेस्कटॉप ऐप की आवश्यकता नहीं होती।

कैमरा एक्सेस डिफ़ॉल्ट रूप से बंद रहता है। इसे Plugin प्रविष्टि के अंतर्गत सक्षम करें, फिर Node सेवा पुनः आरंभ करें ताकि उसका Gateway विज्ञापन फिर से बनाया जा सके:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          camera: { enabled: true },
        },
      },
    },
  },
}
```

आवश्यकताएँ:

- V4L2 इनपुट, `libx264`, और AAC समर्थन वाला FFmpeg
- Node-सेवा उपयोगकर्ता द्वारा पठनीय `/dev/video*` डिवाइस; सामान्य वितरणों पर, उस उपयोगकर्ता को `video` समूह में जोड़ें
- डिफ़ॉल्ट `includeAudio: true` वाली क्लिप के लिए, डिफ़ॉल्ट स्रोत सहित कार्यरत PulseAudio सर्वर या PipeWire PulseAudio संगतता परत

Linux, `camera.list` से कैप्चर-सक्षम, पठनीय V4L2 डिवाइस पथ लौटाता है; FFmpeg प्रत्येक `/dev/video*` उम्मीदवार की जाँच करता है और मेटाडेटा या केवल-आउटपुट Node को छोड़ देता है। डिवाइस `position`, `unknown` है, इसलिए `deviceId` के बिना दिशा अनुरोध सामने या पीछे का कैमरा बताने के बजाय एक `unknown`-स्थिति वाली फ़ोटो या क्लिप उत्पन्न करते हैं। किसी होस्ट पर एकाधिक कैमरे होने पर `deviceId` का उपयोग करें। `camera.snap`, `delayMs` के लिए FFmpeg इनपुट वार्म-अप का उपयोग करता है और चौड़ाई सीमित करते हुए पक्षानुपात बनाए रखता है। `camera.clip` माइक्रोफ़ोन ऑडियो को MP4 ऑडियो ट्रैक के रूप में रिकॉर्ड करता है; OpenClaw जानबूझकर कोई स्वतंत्र माइक्रोफ़ोन कमांड उपलब्ध नहीं कराता।

Plugin MP4 वीडियो के लिए `libx264` का उपयोग करता है और कोडेक चुपचाप नहीं बदलता। आवश्यक इनपुट या एनकोडर के बिना FFmpeg बिल्ड `CAMERA_UNAVAILABLE` लौटाता है। 25MB base64 पेलोड सीमा से अधिक होने वाली फ़ोटो और क्लिप `PAYLOAD_TOO_LARGE` के साथ विफल होती हैं।

`camera.snap` और `camera.clip` जोखिमपूर्ण कमांड बने रहते हैं। उन्हें `gateway.nodes.commands.allow` में केवल तभी जोड़ें, जब आप कैप्चर को सक्रिय करना चाहते हों; केवल Plugin सक्षम करने से Gateway नीति बायपास नहीं होती।

## सुरक्षा + व्यावहारिक सीमाएँ

- कैमरा और माइक्रोफ़ोन एक्सेस सामान्य OS अनुमति संकेतों को सक्रिय करते हैं (और `Info.plist` में उपयोग स्ट्रिंग आवश्यक होती हैं)।
- Node पेलोड का आकार अत्यधिक बढ़ने से रोकने के लिए वीडियो क्लिप अधिकतम 60s तक सीमित हैं (base64 ओवरहेड और संदेश सीमाएँ)।

## macOS स्क्रीन वीडियो (OS-स्तरीय)

_स्क्रीन_ वीडियो (कैमरा नहीं) के लिए macOS सहायक का उपयोग करें:

```bash
openclaw nodes screen record --node <id> --duration 10s --fps 15   # सहेजा गया पथ प्रिंट करता है
```

macOS **Screen Recording** अनुमति (TCC) आवश्यक है।

## संबंधित

- [छवि और मीडिया समर्थन](/hi/nodes/images)
- [मीडिया को समझना](/hi/nodes/media-understanding)
- [स्थान कमांड](/hi/nodes/location-command)
