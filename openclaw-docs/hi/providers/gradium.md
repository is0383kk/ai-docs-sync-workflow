---
read_when:
    - आप टेक्स्ट-टू-स्पीच के लिए Gradium चाहते हैं
    - आपको Gradium API कुंजी, वॉइस या डायरेक्टिव टोकन कॉन्फ़िगरेशन की आवश्यकता है
summary: OpenClaw में Gradium टेक्स्ट-टू-स्पीच का उपयोग करें
title: Gradium
x-i18n:
    generated_at: "2026-07-27T21:36:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5536426eb6d3c8f24c04643b033ebb519a1f2f9df9d97c917ced1c7e23ad180d
    source_path: providers/gradium.md
    workflow: 16
---

[Gradium](https://gradium.ai) OpenClaw के लिए टेक्स्ट-टू-स्पीच प्रदाता है। यह मानक ऑडियो उत्तर (WAV), वॉइस-नोट-संगत Opus आउटपुट और टेलीफ़ोनी सतहों के लिए 8 kHz u-law ऑडियो प्रस्तुत करता है।

| गुण           | मान                                  |
| ------------- | ------------------------------------ |
| प्रदाता आईडी  | `gradium`                            |
| प्रमाणीकरण    | `GRADIUM_API_KEY` या कॉन्फ़िगरेशन `apiKey` |
| आधार URL      | `https://api.gradium.ai` (डिफ़ॉल्ट)   |
| डिफ़ॉल्ट वॉइस | `Emma` (`YTpq7expH9539ERJ`)          |

## Plugin इंस्टॉल करें

Gradium एक आधिकारिक बाहरी Plugin है। इसे इंस्टॉल करें, फिर Gateway पुनः आरंभ करें:

```bash
openclaw plugins install @openclaw/gradium-speech
openclaw gateway restart
```

## सेटअप

Gradium API कुंजी बनाएँ, फिर उसे किसी परिवेश चर या कॉन्फ़िगरेशन कुंजी के माध्यम से उपलब्ध कराएँ। कॉन्फ़िगरेशन को परिवेश चर पर प्राथमिकता मिलती है।

<Tabs>
  <Tab title="परिवेश चर">
    ```bash
    export GRADIUM_API_KEY="gsk_..."
    ```
  </Tab>

  <Tab title="कॉन्फ़िगरेशन कुंजी">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "gradium",
        providers: {
          gradium: {
            apiKey: "${GRADIUM_API_KEY}",
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## कॉन्फ़िगरेशन

```json5
{
  tts: {
    auto: "always",
    provider: "gradium",
    providers: {
      gradium: {
        speakerVoiceId: "YTpq7expH9539ERJ",
        // apiKey: "${GRADIUM_API_KEY}",
        // baseUrl: "https://api.gradium.ai",
      },
    },
  },
}
```

| कुंजी                                  | प्रकार | विवरण                                                                                                  |
| -------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------- |
| `tts.providers.gradium.apiKey`         | string | समाधान की गई API कुंजी। `${ENV}` और सीक्रेट संदर्भों का समर्थन करती है।                                  |
| `tts.providers.gradium.baseUrl`        | string | `api.gradium.ai` पर HTTPS Gradium API URL। अंत के स्लैश हटा दिए जाते हैं। डिफ़ॉल्ट `https://api.gradium.ai`। |
| `tts.providers.gradium.speakerVoiceId` | string | निर्देश द्वारा ओवरराइड न किए जाने पर उपयोग की जाने वाली डिफ़ॉल्ट वॉइस आईडी।                              |

आउटपुट प्रारूप लक्ष्य सतह के अनुसार अपने-आप चुना जाता है ([आउटपुट](#output) देखें) और इसे `openclaw.json` में कॉन्फ़िगर नहीं किया जा सकता।

## वॉइस

| नाम                 | वॉइस आईडी          |
| ------------------ | ------------------ |
| Arthur             | `3jUdJyOi9pgbxBTK` |
| Christina          | `2H4HY2CBNyJHBCrP` |
| Emma **(डिफ़ॉल्ट)** | `YTpq7expH9539ERJ` |
| John               | `KWJiFWu2O9nMPYcR` |
| Kent               | `LFZvm12tW_z0xfGo` |
| Sydney             | `jtEKaLYNn6iif5PR` |
| Tiffany            | `Eu9iL_CYe8N-Gkx_` |

### प्रत्येक संदेश के लिए वॉइस ओवरराइड

जब सक्रिय स्पीच नीति वॉइस ओवरराइड की अनुमति देती है, तो निर्देश टोकन के साथ संदेश के भीतर ही वॉइस बदलें (इनमें से कोई भी इस्तेमाल किया जा सकता है; सभी प्रदाता-मूल वॉइस आईडी लेते हैं):

```text
/voice:LFZvm12tW_z0xfGo
/voice_id:LFZvm12tW_z0xfGo
/voiceid:LFZvm12tW_z0xfGo
/gradium_voice:LFZvm12tW_z0xfGo
/gradiumvoice:LFZvm12tW_z0xfGo
```

यदि स्पीच नीति वॉइस ओवरराइड अक्षम करती है, तो निर्देश उपयोग कर लिया जाता है, लेकिन अनदेखा कर दिया जाता है।

## आउटपुट

आउटपुट प्रारूप लक्ष्य सतह के अनुसार चुना जाता है; प्रदाता अन्य प्रारूप संश्लेषित नहीं करता।

| लक्ष्य         | प्रारूप      | फ़ाइल एक्सटेंशन | नमूना दर | वॉइस-संगत फ़्लैग |
| -------------- | ----------- | -------- | ----------- | --------------------- |
| मानक ऑडियो     | `wav`       | `.wav`   | प्रदाता    | नहीं                  |
| वॉइस नोट       | `opus`      | `.opus`  | प्रदाता    | हाँ                   |
| टेलीफ़ोनी      | `ulaw_8000` | लागू नहीं      | 8 kHz       | लागू नहीं             |

## स्वतः-चयन क्रम

कॉन्फ़िगर किए गए TTS प्रदाताओं में Gradium का स्वतः-चयन क्रम `30` है। जब `tts.provider` पिन नहीं किया गया हो, तब OpenClaw सक्रिय प्रदाता कैसे चुनता है, इसके लिए [टेक्स्ट-टू-स्पीच](/hi/tools/tts) देखें।

## संबंधित

- [टेक्स्ट-टू-स्पीच](/hi/tools/tts)
- [मीडिया का अवलोकन](/hi/tools/media-overview)
