---
read_when:
    - आप OpenClaw में Z.AI / GLM मॉडल चाहते हैं
    - आपको बस एक सरल ZAI_API_KEY सेटअप चाहिए
summary: OpenClaw के साथ Z.AI (GLM मॉडल) का उपयोग करें
title: Z.AI
x-i18n:
    generated_at: "2026-07-27T20:29:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ca3e7ef743e908550f4d96ba6f78167e38cabd15b14044683b02493ebbf3025
    source_path: providers/zai.md
    workflow: 16
---

Z.AI **GLM** मॉडल के लिए API प्लेटफ़ॉर्म है। यह GLM के लिए REST API प्रदान करता है और
प्रमाणीकरण के लिए API कुंजियों का उपयोग करता है। Z.AI कंसोल में अपनी API कुंजी बनाएँ।
OpenClaw, Z.AI API कुंजी के साथ `zai` प्रदाता का उपयोग करता है।

| प्रॉपर्टी | मान                                        |
| -------- | -------------------------------------------- |
| प्रदाता | `zai`                                        |
| पैकेज  | `@openclaw/zai-provider`                     |
| प्रमाणीकरण     | `ZAI_API_KEY` (पुराना उपनाम: `Z_AI_API_KEY`) |
| API      | Z.AI Chat Completions (Bearer प्रमाणीकरण)          |

## GLM मॉडल

GLM एक मॉडल परिवार है, कोई अलग प्रदाता नहीं। OpenClaw में, GLM मॉडल
`zai/glm-5.2` जैसे संदर्भों का उपयोग करते हैं: प्रदाता `zai`, मॉडल आईडी `glm-5.2`।

## आरंभ करना

पहले प्रदाता Plugin इंस्टॉल करें:

```bash
openclaw plugins install @openclaw/zai-provider
```

<Tabs>
  <Tab title="एंडपॉइंट का स्वतः पता लगाएँ">
    **इनके लिए सर्वोत्तम:** अधिकांश उपयोगकर्ता। OpenClaw आपकी API कुंजी से समर्थित Z.AI एंडपॉइंट की जाँच करता है और सही बेस URL स्वचालित रूप से लागू करता है।

    <Steps>
      <Step title="ऑनबोर्डिंग चलाएँ">
        ```bash
        openclaw onboard --auth-choice zai-api-key
        ```
      </Step>
      <Step title="सत्यापित करें कि मॉडल सूचीबद्ध है">
        ```bash
        openclaw models list --all --provider zai
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="स्पष्ट क्षेत्रीय एंडपॉइंट">
    **इनके लिए सर्वोत्तम:** वे उपयोगकर्ता जो किसी विशिष्ट Coding Plan या सामान्य API सतह को अनिवार्य रूप से उपयोग करना चाहते हैं।

    <Steps>
      <Step title="सही ऑनबोर्डिंग विकल्प चुनें">
        ```bash
        # Coding Plan Global (Coding Plan उपयोगकर्ताओं के लिए अनुशंसित)
        openclaw onboard --auth-choice zai-coding-global

        # Coding Plan CN (चीन क्षेत्र)
        openclaw onboard --auth-choice zai-coding-cn

        # सामान्य API
        openclaw onboard --auth-choice zai-global

        # सामान्य API CN (चीन क्षेत्र)
        openclaw onboard --auth-choice zai-cn
        ```
      </Step>
      <Step title="सत्यापित करें कि मॉडल सूचीबद्ध है">
        ```bash
        openclaw models list --all --provider zai
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

### एंडपॉइंट

| ऑनबोर्डिंग विकल्प   | बेस URL                                      | डिफ़ॉल्ट मॉडल |
| ------------------- | --------------------------------------------- | ------------- |
| `zai-global`        | `https://api.z.ai/api/paas/v4`                | `glm-5.1`     |
| `zai-cn`            | `https://open.bigmodel.cn/api/paas/v4`        | `glm-5.1`     |
| `zai-coding-global` | `https://api.z.ai/api/coding/paas/v4`         | `glm-5.2`     |
| `zai-coding-cn`     | `https://open.bigmodel.cn/api/coding/paas/v4` | `glm-5.2`     |

Z.AI, Anthropic-संगत Coding Plan बेस URL
`https://api.z.ai/api/anthropic` भी प्रकाशित करता है। OpenClaw के Z.AI विकल्प ऊपर दिए गए दस्तावेज़ीकृत
OpenAI Chat Completions एंडपॉइंट का उपयोग करते हैं; Anthropic URL उन क्लाइंट के लिए है जो
सीधे Anthropic Messages का उपयोग करते हैं।

`zai-api-key` आपकी कुंजी को प्रत्येक एंडपॉइंट की chat-completions API पर जाँचकर इन चार में से एक का स्वतः पता लगाता है,
पहले सामान्य एंडपॉइंट (`zai-global`,
फिर `zai-cn`) और उसके बाद Coding Plan एंडपॉइंट (`zai-coding-global`, फिर
`zai-coding-cn`) की जाँच करता है, और अनुरोध स्वीकार करने वाले पहले एंडपॉइंट पर रुक जाता है।
यदि आपकी कुंजी दोनों पर काम करती है, तो Coding Plan एंडपॉइंट को अनिवार्य करने के लिए स्पष्ट `--auth-choice` का उपयोग करें।

## दर सीमाएँ और अतिभार

Z.AI, Coding Plan और सामान्य-उद्देश्य वाले एजेंट टूल को क्षमता-प्रबंधित
सेवाओं के रूप में दस्तावेज़ीकृत करता है। Z.AI के अपने दस्तावेज़ों के अनुसार:

- [सामान्य-उद्देश्य वाले एजेंट टूल](https://docs.z.ai/devpack/tool/others),
  जिनमें OpenClaw शामिल है, सर्वोत्तम-प्रयास के आधार पर उपलब्ध कराए जाते हैं। अधिक अनुमान
  लोड के दौरान, आमतौर पर सिंगापुर समय के अनुसार दोपहर 2-6 बजे के आसपास, कुछ अनुरोधों पर अस्थायी
  दर सीमाएँ लागू हो सकती हैं।
- [Coding Plan की दर और समवर्ती अनुरोध सीमाएँ](https://docs.z.ai/devpack/usage-policy)
  प्लान स्तर से जुड़ी होती हैं और संसाधन उपलब्धता के आधार पर गतिशील रूप से समायोजित की जा सकती हैं।
  कम व्यस्त घंटों में समवर्ती अनुरोधों की सीमा अधिक हो सकती है।
- [API त्रुटि कोड `1302`](https://docs.z.ai/api-reference/api-code) का अर्थ है "अनुरोधों की
  दर सीमा पूरी हो गई है"। API त्रुटि कोड `1305` का अर्थ है "सेवा अस्थायी रूप से
  अतिभारित हो सकती है, कृपया बाद में फिर प्रयास करें"।

यदि व्यस्त अवधि में आपको अस्थायी `429` या `1305` प्रतिक्रिया दिखाई देती है, तो प्रतीक्षा करें और
अनुरोध का फिर से प्रयास करें। यदि विफलताएँ अधिकतम व्यस्तता की अवधि के बाहर भी दोहराई जाती हैं, या केवल
एक एंडपॉइंट, मॉडल या अनुरोध संरचना के लिए होती हैं, तो पहले कॉन्फ़िगर किए गए एंडपॉइंट
और मॉडल की जाँच करें:

```bash
openclaw models list --all --provider zai
openclaw config get models.providers.zai.baseUrl
```

Coding Plan कुंजियों को `https://api.z.ai/api/coding/paas/v4` जैसे Coding Plan एंडपॉइंट का उपयोग करना चाहिए;
सामान्य API कुंजियों को `https://api.z.ai/api/paas/v4` जैसे सामान्य API
एंडपॉइंट का उपयोग करना चाहिए। समान कुंजी और एंडपॉइंट के साथ लगातार विफलताएँ
प्रदाता-पक्ष की अस्वीकृति या प्लान की सीमा का संकेत दे सकती हैं,
न कि सामान्य अधिकतम-लोड थ्रॉटलिंग का।

## कॉन्फ़िगरेशन उदाहरण

<Tip>
`zai-api-key` OpenClaw को कुंजी से मेल खाने वाले Z.AI एंडपॉइंट का पता लगाने और
सही बेस URL स्वचालित रूप से लागू करने देता है। जब आप किसी विशिष्ट Coding Plan या सामान्य API सतह को
अनिवार्य रूप से उपयोग करना चाहते हैं, तो स्पष्ट क्षेत्रीय विकल्पों का उपयोग करें।
</Tip>

```json5
{
  env: { ZAI_API_KEY: "sk-..." },
  models: {
    providers: {
      zai: {
        // GLM-5.2, Coding Plan एंडपॉइंट का उपयोग करता है।
        baseUrl: "https://api.z.ai/api/coding/paas/v4",
      },
    },
  },
  agents: { defaults: { model: { primary: "zai/glm-5.2" } } },
}
```

## अंतर्निहित कैटलॉग

`zai` प्रदाता Plugin अपने कैटलॉग को Plugin मैनिफ़ेस्ट में शामिल करता है, इसलिए केवल-पठन
सूचीकरण प्रदाता रनटाइम लोड किए बिना ज्ञात GLM पंक्तियाँ दिखा सकता है:

```bash
openclaw models list --all --provider zai
```

मैनिफ़ेस्ट-आधारित कैटलॉग में वर्तमान में ये शामिल हैं:

| मॉडल संदर्भ            | टिप्पणियाँ                           |
| -------------------- | ------------------------------- |
| `zai/glm-5.2`        | Coding Plan डिफ़ॉल्ट; 1M संदर्भ |
| `zai/glm-5.1`        | सामान्य API डिफ़ॉल्ट             |
| `zai/glm-5`          |                                 |
| `zai/glm-5-turbo`    |                                 |
| `zai/glm-5v-turbo`   |                                 |
| `zai/glm-4.7`        |                                 |
| `zai/glm-4.7-flash`  |                                 |
| `zai/glm-4.7-flashx` |                                 |
| `zai/glm-4.6`        |                                 |
| `zai/glm-4.6v`       |                                 |
| `zai/glm-4.5`        |                                 |
| `zai/glm-4.5-air`    |                                 |
| `zai/glm-4.5-flash`  |                                 |
| `zai/glm-4.5v`       |                                 |

कैटलॉग का टोकन-लागत मेटाडेटा Z.AI के वर्तमान
[उपयोग के अनुसार भुगतान वाले मूल्य निर्धारण](https://docs.z.ai/guides/overview/pricing) का अनुसरण करता है। Coding Plan
सदस्यताएँ प्रति-टोकन बिलिंग के बजाय प्लान कोटा का उपयोग करती हैं; प्लान के मूल्य निर्धारण और उपलब्धता के लिए लाइव
[सदस्यता पृष्ठ](https://z.ai/subscribe) देखें।

<Tip>
GLM मॉडल `zai/<model>` के रूप में उपलब्ध हैं (उदाहरण: `zai/glm-5`)।
</Tip>

<Note>
Coding Plan सेटअप का डिफ़ॉल्ट `zai/glm-5.2` है; सामान्य API सेटअप
`zai/glm-5.1` बनाए रखता है। Coding Plan एंडपॉइंट पर, जब कुंजी/प्लान GLM-5.2 उपलब्ध नहीं कराता है, तो स्वतः पहचान
`glm-5.1` और फिर `glm-4.7` पर फ़ॉलबैक करती है। GLM
संस्करण और उपलब्धता बदल सकते हैं; अपने इंस्टॉल किए गए संस्करण को ज्ञात कैटलॉग देखने के लिए `openclaw models list --all --provider zai`
चलाएँ।
</Note>

## चिंतन स्तर

<Tabs>
  <Tab title="GLM-5.2">
    पूर्ण श्रेणी: `off`, `low`, `high`, `max` (डिफ़ॉल्ट `off`)। OpenClaw
    `low` और `high` को Z.AI के `high` तर्क प्रयास से, और `max` को Z.AI के
    `max` प्रयास से, अनुरोध पेलोड पर `reasoning_effort` के माध्यम से मैप करता है।
  </Tab>
  <Tab title="अन्य GLM मॉडल">
    केवल द्विआधारी टॉगल: `off` और `low` (चयनकर्ताओं में `on` के रूप में दिखाया जाता है), डिफ़ॉल्ट
    `off`। चिंतन को `off` पर सेट करने से `thinking: { type: "disabled" }` भेजा जाता है;
    कोई भी अन्य स्तर अनुरोध पेलोड को अपरिवर्तित छोड़ता है (Z.AI का अपना डिफ़ॉल्ट
    तर्क व्यवहार लागू होता है)।
  </Tab>
</Tabs>

चिंतन को `off` पर सेट करने से ऐसी प्रतिक्रियाओं से बचा जा सकता है जो दृश्यमान टेक्स्ट से पहले आउटपुट बजट
`reasoning_content` पर खर्च कर देती हैं।

## उन्नत कॉन्फ़िगरेशन

<AccordionGroup>
  <Accordion title="अज्ञात GLM-5 मॉडलों का अग्रेषित समाधान">
    जब आईडी वर्तमान GLM-5 परिवार संरचना से मेल खाती है, तो अज्ञात `glm-5*` आईडी भी
    `glm-4.7` टेम्पलेट से प्रदाता-स्वामित्व वाला मेटाडेटा संश्लेषित करके प्रदाता पथ पर
    अग्रेषित रूप से हल हो जाती हैं।
  </Accordion>

  <Accordion title="टूल-कॉल स्ट्रीमिंग">
    Z.AI टूल-कॉल स्ट्रीमिंग के लिए `tool_stream` डिफ़ॉल्ट रूप से सक्षम है। इसे अक्षम करने के लिए:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "zai/<model>": {
              params: { tool_stream: false },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="संरक्षित चिंतन">
    संरक्षित चिंतन वैकल्पिक है क्योंकि Z.AI को पूर्ण ऐतिहासिक
    `reasoning_content` फिर से चलाने की आवश्यकता होती है, जिससे प्रॉम्प्ट टोकन बढ़ते हैं। इसे
    प्रति मॉडल सक्षम करें:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "zai/glm-5.2": {
              params: { preserveThinking: true },
            },
          },
        },
      },
    }
    ```

    सक्षम होने और चिंतन चालू होने पर, OpenClaw
    `thinking: { type: "enabled", clear_thinking: false }` भेजता है और उसी OpenAI-संगत ट्रांसक्रिप्ट के लिए पूर्व
    `reasoning_content` फिर से चलाता है। snake_case
    `preserve_thinking` पैरामीटर कुंजी उपनाम के रूप में काम करती है।

    उन्नत उपयोगकर्ता अभी भी सटीक प्रदाता पेलोड को
    `params.extra_body.thinking` से ओवरराइड कर सकते हैं।

  </Accordion>

  <Accordion title="छवि की समझ">
    Z.AI Plugin छवि की समझ को पंजीकृत करता है।

    | प्रॉपर्टी      | मान       |
    | ------------- | ----------- |
    | मॉडल         | `glm-4.6v`  |

    छवि की समझ कॉन्फ़िगर किए गए Z.AI प्रमाणीकरण से स्वतः हल होती है—
    किसी अतिरिक्त कॉन्फ़िगरेशन की आवश्यकता नहीं है।

  </Accordion>

  <Accordion title="प्रमाणीकरण विवरण">
    - Z.AI आपकी API कुंजी के साथ Bearer प्रमाणीकरण का उपयोग करता है।
    - `zai-api-key` ऑनबोर्डिंग विकल्प आपकी कुंजी से समर्थित एंडपॉइंट की जाँच करके मेल खाने वाले Z.AI एंडपॉइंट का स्वतः पता लगाता है।
    - जब आप किसी विशिष्ट API सतह को अनिवार्य रूप से उपयोग करना चाहते हैं, तो स्पष्ट क्षेत्रीय विकल्पों (`zai-coding-global`, `zai-coding-cn`, `zai-global`, `zai-cn`) का उपयोग करें।
    - पुराना पर्यावरण चर `Z_AI_API_KEY` अभी भी स्वीकार किया जाता है; यदि `ZAI_API_KEY` सेट नहीं है, तो OpenClaw स्टार्टअप पर इसे `ZAI_API_KEY` में कॉपी करता है।

  </Accordion>
</AccordionGroup>

## संबंधित

<CardGroup cols={2}>
  <Card title="मॉडल चयन" href="/hi/concepts/model-providers" icon="layers">
    प्रदाताओं, मॉडल संदर्भों और फ़ेलओवर व्यवहार का चयन।
  </Card>
  <Card title="कॉन्फ़िगरेशन संदर्भ" href="/hi/gateway/configuration-reference" icon="gear">
    प्रदाता और मॉडल सेटिंग सहित पूर्ण OpenClaw कॉन्फ़िगरेशन स्कीमा।
  </Card>
</CardGroup>
