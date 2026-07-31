---
read_when:
    - आप ऑडियो अटैचमेंट के लिए SenseAudio वाक्-से-पाठ चाहते हैं
    - आपको SenseAudio API कुंजी का एनवायरनमेंट वेरिएबल या ऑडियो कॉन्फ़िगरेशन पथ चाहिए
summary: इनबाउंड वॉइस नोट्स के लिए SenseAudio बैच स्पीच-टू-टेक्स्ट
title: SenseAudio
x-i18n:
    generated_at: "2026-07-27T21:37:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0ca4a31a32eed85c1d9dcd13ebc2eaea94be370d2b1013ae8b4677949bea91d
    source_path: providers/senseaudio.md
    workflow: 16
---

SenseAudio, OpenClaw की साझा `tools.media.audio` पाइपलाइन के माध्यम से आने वाले ऑडियो और वॉइस-नोट अटैचमेंट का लिप्यंतरण करता है। OpenClaw, OpenAI-संगत लिप्यंतरण एंडपॉइंट पर मल्टीपार्ट ऑडियो पोस्ट करता है और लौटाए गए टेक्स्ट को `{{Transcript}}` के रूप में तथा एक `[Audio]` ब्लॉक के साथ प्रविष्ट करता है।

| गुण           | मान                                              |
| ------------- | ------------------------------------------------ |
| प्रदाता आईडी  | `senseaudio`                               |
| Plugin        | बंडल किया गया, `enabledByDefault: true`               |
| अनुबंध        | `mediaUnderstandingProviders` (ऑडियो)                      |
| प्रमाणीकरण एन्वायरनमेंट वेरिएबल | `SENSEAUDIO_API_KEY`               |
| डिफ़ॉल्ट मॉडल | `senseaudio-asr-pro-1.5-260319`                               |
| डिफ़ॉल्ट URL  | `https://api.senseaudio.cn/v1`                               |
| वेबसाइट       | [senseaudio.cn](https://senseaudio.cn)           |
| दस्तावेज़     | [senseaudio.cn/docs](https://senseaudio.cn/docs) |

## शुरू करना

<Steps>
  <Step title="अपनी API कुंजी सेट करें">
    ```bash
    export SENSEAUDIO_API_KEY="..."
    ```
  </Step>
  <Step title="ऑडियो प्रदाता सक्षम करें">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [{ provider: "senseaudio", model: "senseaudio-asr-pro-1.5-260319" }],
          },
        },
      },
    }
    ```
  </Step>
  <Step title="वॉइस नोट भेजें">
    किसी भी कनेक्टेड चैनल के माध्यम से एक ऑडियो संदेश भेजें। OpenClaw ऑडियो को
    SenseAudio पर अपलोड करता है और उत्तर पाइपलाइन में प्रतिलेख का उपयोग करता है।
  </Step>
</Steps>

## विकल्प

| विकल्प    | पथ                             | विवरण                              |
| ---------- | ------------------------------- | ----------------------------------- |
| `model`    | `tools.media.models[].model`    | SenseAudio ASR मॉडल आईडी            |
| `language` | `tools.media.models[].language` | वैकल्पिक भाषा संकेत                 |
| `prompt`   | `tools.media.models[].prompt`   | वैकल्पिक लिप्यंतरण प्रॉम्प्ट        |
| `baseUrl`  | `tools.media.models[].baseUrl`  | OpenAI-संगत बेस को ओवरराइड करें     |
| `headers`  | `tools.media.models[].headers`  | अतिरिक्त अनुरोध हेडर                |

<Note>
OpenClaw में SenseAudio केवल बैच STT है। Voice Call का रियलटाइम लिप्यंतरण
स्ट्रीमिंग STT समर्थन वाले प्रदाताओं का उपयोग करना जारी रखता है।
</Note>

## संबंधित

- [मीडिया की समझ (ऑडियो)](/hi/nodes/audio)
- [मॉडल प्रदाता](/hi/concepts/model-providers)
