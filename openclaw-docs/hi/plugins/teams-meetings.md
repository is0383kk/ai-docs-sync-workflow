---
read_when:
    - आप चाहते हैं कि एक OpenClaw एजेंट Microsoft Teams मीटिंग में शामिल हो
    - आप Teams मीटिंग में प्रत्युत्तर देने के लिए Chrome, BlackHole या SoX कॉन्फ़िगर कर रहे हैं
summary: 'Microsoft Teams मीटिंग Plugin: Chrome ब्राउज़र अतिथि के रूप में कार्यस्थल या उपभोक्ता मीटिंग में शामिल हों'
title: Microsoft Teams मीटिंग Plugin
x-i18n:
    generated_at: "2026-07-27T18:23:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f84e58d478185d026dd79a02a8500af48f51689ef6865d56badb0e27c6d2814
    source_path: plugins/teams-meetings.md
    workflow: 16
---

`teams-meetings` Plugin, OpenClaw Chrome प्रोफ़ाइल में अतिथि के रूप में Microsoft Teams लिंक से जुड़ता है। यह `teams.microsoft.com/l/meetup-join/...` के अंतर्गत कार्यस्थल लिंक और `teams.live.com/meet/...` के अंतर्गत उपभोक्ता लिंक स्वीकार करता है। यह मीटिंग नहीं बनाता, डायल इन नहीं करता, Microsoft Graph को कॉल नहीं करता और ऑडियो/वीडियो रिकॉर्डिंग कैप्चर नहीं करता।

## सेटअप

बोलकर उत्तर देने की सुविधा, [Google Meet Plugin](/hi/plugins/google-meet) जैसी ही स्थानीय ऑडियो पूर्वापेक्षाओं का उपयोग करती है: macOS, `BlackHole 2ch` वर्चुअल ऑडियो डिवाइस और SoX।

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Plugin शामिल है और डिफ़ॉल्ट रूप से सक्षम रहता है। इसे अनुकूलित करने के लिए ही प्रविष्टि जोड़ें, फिर सेटअप की जाँच करें:

```json5
{
  plugins: {
    entries: {
      "teams-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

यदि आप Plugin को सक्रिय नहीं रखना चाहते, तो `openclaw plugins disable teams-meetings` चलाएँ।

```bash
openclaw teamsmeetings setup
openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'
```

युग्मित macOS Node पर Chrome, BlackHole और SoX चलाने के लिए `chromeNode.node` का उपयोग करें। Node को `teamsmeetings.chrome` और `browser.proxy` की अनुमति देनी होगी।

## मोड

| मोड         | व्यवहार                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | रीयल-टाइम ट्रांसक्रिप्शन कॉन्फ़िगर किए गए OpenClaw एजेंट से परामर्श करता है; TTS उत्तर देता है। |
| `bidi`       | एक रीयल-टाइम वॉइस मॉडल सीधे सुनता और उत्तर देता है।                        |
| `transcribe` | लाइव-कैप्शन ट्रांसक्रिप्ट स्नैपशॉट के साथ केवल अवलोकन हेतु जुड़ना।                   |

हर मोड में प्रवेश मिलने के बाद Teams लाइव कैप्शन सक्षम किए जाते हैं, ताकि OpenClaw
वक्ता से संबद्ध नोट्स स्थायी रूप से सहेज सके। `transcript` क्रिया अब भी केवल
`transcribe` सत्रों के लिए सीमित लाइव बफ़र लौटाती है। मीटिंग छोड़ने पर, OpenClaw
स्थायी ट्रांसक्रिप्ट और उससे प्राप्त सारांश को साझा स्थिति डेटाबेस में संग्रहीत करता है; उन्हें
[`openclaw transcripts`](/hi/cli/transcripts) से सूचीबद्ध या निर्यात करें।

स्वचालित नोट्स डिफ़ॉल्ट रूप से सक्षम हैं। स्थायी नोट्स को वैश्विक रूप से
अक्षम करने के लिए `transcripts.enabled: false` सेट करें; स्पष्ट `transcribe` मोड अब भी केवल
अपना सीमित लाइव अंतिम भाग दिखाता है।

## अतिथि के रूप में जुड़ने की सीमाएँ

ब्राउज़र अडैप्टर ऐप इंटरस्टिशियल को बंद करता है, अतिथि का नाम भरता है, कैमरा बंद करता है, चुने गए मोड के लिए माइक्रोफ़ोन कॉन्फ़िगर करता है और जुड़ने का बटन क्लिक करता है। कॉल के दौरान की स्थिति हैंग-अप नियंत्रण का उपयोग करती है; लॉबी, टेनेंट साइन-इन और डिवाइस-अनुमति स्थितियाँ स्पष्ट रूप से मैन्युअल कार्रवाई के कारण लौटाती हैं। उपभोक्ता मीटिंग लॉन्चर के रीडायरेक्ट और Chrome द्वारा दिखाए गए `BlackHole 2ch (Virtual)` लेबल समर्थित हैं।

Teams टेनेंट नीति साइन-इन, ईमेल सत्यापन या आयोजक की स्वीकृति आवश्यक कर सकती है। वह चरण OpenClaw Chrome प्रोफ़ाइल में पूरा करें, फिर स्थिति या स्पीच का पुनः प्रयास करें। Plugin टेनेंट नीति को बायपास नहीं करता।

उपभोक्ता Teams वेब क्लाइंट को ऐप इंटरस्टिशियल, अतिथि-नाम प्रविष्टि, मीटिंग में शामिल होने से पहले माइक्रोफ़ोन/कैमरा टॉगल, जुड़ने, लॉबी में प्रवेश, मीडिया अनुमतियों, कॉल के दौरान स्थिति की पहचान, लाइव कैप्शन, BlackHole इनपुट/आउटपुट रूटिंग, कॉल छोड़ने और कॉल के बाद की स्थिति की पहचान के लिए लाइव-सत्यापित किया गया है। कार्यस्थल टेनेंट अलग-अलग साइन-इन, ईमेल-सत्यापन, प्रवेश और कॉल छोड़ने की पुष्टि संबंधी नीति लागू कर सकते हैं; रिपोर्ट की गई किसी भी मैन्युअल कार्रवाई को OpenClaw Chrome प्रोफ़ाइल में पूरा करें।

## टूल और Gateway सतह

`teams_meetings` एजेंट टूल `join`, `leave`, `status`, `transcript` और `speak` का समर्थन करता है। Gateway विधियाँ `teamsmeetings.*` उपसर्ग का उपयोग करती हैं। Node कमांड `teamsmeetings.chrome` है।

## संबंधित

- [मीटिंग Plugin का अवलोकन](/hi/plugins/meeting-plugins)
- [Microsoft Teams चैनल](/hi/channels/msteams)
