---
read_when:
    - आप OpenClaw में Runway वीडियो जनरेशन का उपयोग करना चाहते हैं
    - आपको Runway API कुंजी/एनवायरनमेंट सेटअप की आवश्यकता है
    - आप Runway को डिफ़ॉल्ट वीडियो प्रदाता बनाना चाहते हैं
summary: OpenClaw में Runway वीडियो जनरेशन सेटअप
title: Runway
x-i18n:
    generated_at: "2026-07-27T20:28:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a56e768893e327b56d70e8b8c2d426123a861b3cf05c0107d98104e2cee856c
    source_path: providers/runway.md
    workflow: 16
---

OpenClaw होस्टेड वीडियो जनरेशन के लिए एक बंडल किया हुआ `runway` प्रदाता प्रदान करता है, जो डिफ़ॉल्ट रूप से सक्षम है और `videoGenerationProviders` अनुबंध के साथ पंजीकृत है।

| प्रॉपर्टी        | मान                                                             |
| --------------- | ----------------------------------------------------------------- |
| प्रदाता आईडी     | `runway`                                                          |
| Plugin          | बंडल किया हुआ, `enabledByDefault: true`                                 |
| प्रमाणीकरण एनवायरनमेंट वेरिएबल   | `RUNWAYML_API_SECRET` (कैनोनिकल) या `RUNWAY_API_KEY`             |
| ऑनबोर्डिंग फ़्लैग | `--auth-choice runway-api-key`                                    |
| प्रत्यक्ष CLI फ़्लैग | `--runway-api-key <key>`                                          |
| API             | Runway का टास्क-आधारित वीडियो जनरेशन (`GET /v1/tasks/{id}` पोलिंग) |
| डिफ़ॉल्ट मॉडल   | `runway/gen4.5`                                                   |

## आरंभ करना

<Steps>
  <Step title="API कुंजी सेट करें">
    ```bash
    openclaw onboard --auth-choice runway-api-key
    ```
  </Step>
  <Step title="Runway को डिफ़ॉल्ट वीडियो प्रदाता के रूप में सेट करें">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "runway/gen4.5"
    ```
  </Step>
  <Step title="वीडियो जनरेट करें">
    एजेंट से वीडियो जनरेट करने के लिए कहें। Runway का उपयोग अपने-आप किया जाएगा।
  </Step>
</Steps>

## समर्थित मोड और मॉडल

प्रदाता तीन मोड में विभाजित सात Runway मॉडल उपलब्ध कराता है। एक ही मॉडल आईडी एक से अधिक मोड के लिए उपयोग की जा सकती है (उदाहरण के लिए, `gen4.5` टेक्स्ट-से-वीडियो और इमेज-से-वीडियो दोनों के लिए काम करता है)।

| मोड           | मॉडल                                                                 | संदर्भ इनपुट         |
| -------------- | ---------------------------------------------------------------------- | ----------------------- |
| टेक्स्ट-से-वीडियो  | `gen4.5` (डिफ़ॉल्ट), `veo3.1`, `veo3.1_fast`, `veo3`                    | कोई नहीं                    |
| इमेज-से-वीडियो | `gen4.5`, `gen4_turbo`, `gen3a_turbo`, `veo3.1`, `veo3.1_fast`, `veo3` | 1 स्थानीय या रिमोट इमेज |
| वीडियो-से-वीडियो | `gen4_aleph`                                                           | 1 स्थानीय या रिमोट वीडियो |

डेटा URI के माध्यम से स्थानीय इमेज और वीडियो संदर्भ समर्थित हैं।

| आस्पेक्ट रेशियो         | अनुमत मान                              |
| --------------------- | ------------------------------------------- |
| टेक्स्ट-से-वीडियो         | `16:9`, `9:16`                              |
| इमेज और वीडियो संपादन | `1:1`, `16:9`, `9:16`, `3:4`, `4:3`, `21:9` |

<Warning>
  वीडियो-से-वीडियो के लिए वर्तमान में `runway/gen4_aleph` आवश्यक है। अन्य Runway मॉडल आईडी वीडियो संदर्भ इनपुट को अस्वीकार करती हैं।
</Warning>

<Note>
  गलत कॉलम से Runway मॉडल आईडी चुनने पर API अनुरोध के OpenClaw से बाहर जाने से पहले एक स्पष्ट त्रुटि उत्पन्न होती है। प्रदाता `extensions/runway/video-generation-provider.ts` में मोड की अनुमति-सूची (`TEXT_ONLY_MODELS`, `IMAGE_MODELS`, `VIDEO_MODELS`) के आधार पर `model` को सत्यापित करता है।
</Note>

## कॉन्फ़िगरेशन

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "runway/gen4.5",
      },
    },
  },
}
```

## उन्नत कॉन्फ़िगरेशन

<AccordionGroup>
  <Accordion title="एनवायरनमेंट वेरिएबल उपनाम">
    OpenClaw `RUNWAYML_API_SECRET` (कैनोनिकल) और `RUNWAY_API_KEY` दोनों को पहचानता है।
    इनमें से कोई भी वेरिएबल Runway प्रदाता को प्रमाणित करता है।
  </Accordion>

  <Accordion title="टास्क पोलिंग">
    Runway टास्क-आधारित API का उपयोग करता है। जनरेशन अनुरोध सबमिट करने के बाद, वीडियो तैयार होने तक OpenClaw
    `GET /v1/tasks/{id}` को पोल करता है। पोलिंग व्यवहार के लिए किसी अतिरिक्त
    कॉन्फ़िगरेशन की आवश्यकता नहीं है।
  </Accordion>
</AccordionGroup>

## संबंधित

<CardGroup cols={2}>
  <Card title="वीडियो जनरेशन" href="/hi/tools/video-generation" icon="video">
    साझा टूल पैरामीटर, प्रदाता चयन और एसिंक्रोनस व्यवहार।
  </Card>
  <Card title="कॉन्फ़िगरेशन संदर्भ" href="/hi/gateway/config-agents#agent-defaults" icon="gear">
    वीडियो जनरेशन मॉडल सहित एजेंट की डिफ़ॉल्ट सेटिंग।
  </Card>
</CardGroup>
