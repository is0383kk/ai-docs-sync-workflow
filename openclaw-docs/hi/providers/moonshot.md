---
read_when:
    - आप Moonshot Kimi K3/K2 (Moonshot Open Platform) बनाम Kimi Coding सेटअप चाहते हैं
    - आपको अलग-अलग एंडपॉइंट, कुंजियाँ और मॉडल रेफ़रेंस समझने होंगे
    - आप किसी भी प्रदाता के लिए कॉपी/पेस्ट करने योग्य कॉन्फ़िगरेशन चाहते हैं
summary: Moonshot Kimi मॉडल बनाम Kimi Coding कॉन्फ़िगर करें (अलग-अलग प्रदाता + कुंजियाँ)
title: मूनशॉट एआई
x-i18n:
    generated_at: "2026-07-27T21:36:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 213379bf88fec26b052184a920e112f0887d6485601bfb47f590cf37ef983e58
    source_path: providers/moonshot.md
    workflow: 16
---

Moonshot OpenAI-संगत एंडपॉइंट के साथ Kimi API प्रदान करता है। Kimi K3 के लिए
`moonshot/kimi-k3` चुनें, ऑनबोर्डिंग डिफ़ॉल्ट
`moonshot/kimi-k2.6` बनाए रखें, या Kimi Coding के लिए `kimi/kimi-for-coding` का उपयोग करें।

<Warning>
Moonshot और Kimi Coding **अलग-अलग प्रदाता** हैं, और प्रत्येक अलग बाहरी plugin के रूप में उपलब्ध है। कुंजियाँ परस्पर विनिमेय नहीं हैं, एंडपॉइंट अलग हैं और मॉडल संदर्भ अलग हैं (`moonshot/...` बनाम `kimi/...`)।
</Warning>

## अंतर्निहित मॉडल कैटलॉग

[//]: # "moonshot-kimi-k2-ids:start"

| मॉडल संदर्भ                           | नाम                     | रीजनिंग  | इनपुट       | कॉन्टेक्स्ट   | अधिकतम आउटपुट |
| ----------------------------------- | ------------------------ | ---------- | ----------- | --------- | ---------- |
| `moonshot/kimi-k2.6`                | Kimi K2.6                | नहीं         | टेक्स्ट, इमेज | 262,144   | 262,144    |
| `moonshot/kimi-k3`                  | Kimi K3                  | हमेशा अधिकतम | टेक्स्ट, इमेज | 1,048,576 | 1,048,576  |
| `moonshot/kimi-k2.7-code`           | Kimi K2.7 Code           | हमेशा चालू  | टेक्स्ट, इमेज | 262,144   | 262,144    |
| `moonshot/kimi-k2.7-code-highspeed` | Kimi K2.7 Code HighSpeed | हमेशा चालू  | टेक्स्ट, इमेज | 262,144   | 262,144    |
| `moonshot/kimi-k2.5`                | Kimi K2.5                | नहीं         | टेक्स्ट, इमेज | 262,144   | 262,144    |

[//]: # "moonshot-kimi-k2-ids:end"

कैटलॉग लागत अनुमान Moonshot की प्रकाशित उपयोगानुसार-भुगतान दरों का उपयोग करते हैं। लागत संबंधी निर्णय लेने से पहले
[Kimi K3](https://platform.kimi.ai/docs/pricing/chat-k3),
[Kimi K2.7 Code](https://platform.kimi.ai/docs/pricing/chat-k27-code),
[Kimi K2.6](https://platform.kimi.ai/docs/pricing/chat-k26), और
[Kimi K2.5](https://platform.kimi.ai/docs/pricing/chat-k25) के
लाइव विक्रेता पेज देखें।

Kimi K3 हमेशा `reasoning_effort: "max"` पर रीजनिंग करता है। OpenClaw केवल
`/think max` उपलब्ध कराता है, केवल K2 वाला `thinking` फ़ील्ड छोड़ देता है और उन सैंपलिंग
ओवरराइड (`temperature`, `top_p`, `n`, `presence_penalty`, और
`frequency_penalty`) को हटा देता है जिन्हें K3 प्रदाता के डिफ़ॉल्ट पर स्थिर रखता है। Kimi K2.7 Code भी
हमेशा नेटिव थिंकिंग का उपयोग करता है, लेकिन इसके लिए `thinking` और
`reasoning_effort` दोनों को छोड़ना आवश्यक है; HighSpeed वैरिएंट समान अनुबंध का उपयोग करता है।
Kimi K2.6 ऑनबोर्डिंग डिफ़ॉल्ट बना रहता है।
Moonshot का [Kimi K3 क्विकस्टार्ट](https://platform.kimi.ai/docs/guide/kimi-k3-quickstart) देखें।

## शुरुआत करना

Moonshot और Kimi Coding दोनों बाहरी plugin हैं—ऑनबोर्डिंग से पहले
इनमें से एक इंस्टॉल करें।

<Tabs>
  <Tab title="Moonshot API">
    **इनके लिए सर्वोत्तम:** Moonshot Open Platform के माध्यम से Kimi K3 और K2 मॉडल।

    <Steps>
      <Step title="Plugin इंस्टॉल करें">
        ```bash
        openclaw plugins install @openclaw/moonshot-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="अपना एंडपॉइंट क्षेत्र चुनें">
        | प्रमाणीकरण विकल्प            | एंडपॉइंट                       | क्षेत्र        |
        | ---------------------- | ------------------------------ | ------------- |
        | `moonshot-api-key`     | `https://api.moonshot.ai/v1`   | अंतरराष्ट्रीय |
        | `moonshot-api-key-cn`  | `https://api.moonshot.cn/v1`   | चीन         |
      </Step>
      <Step title="ऑनबोर्डिंग चलाएँ">
        ```bash
        openclaw onboard --auth-choice moonshot-api-key
        ```

        या चीन एंडपॉइंट के लिए:

        ```bash
        openclaw onboard --auth-choice moonshot-api-key-cn
        ```
      </Step>
      <Step title="Kimi K3 को डिफ़ॉल्ट मॉडल के रूप में सेट करें">
        ऑनबोर्डिंग Kimi K2.6 को आरंभिक डिफ़ॉल्ट बनाए रखती है। जब Kimi K3 का उपयोग करना हो,
        तो स्पष्ट रूप से स्विच करें:

        ```bash
        openclaw models set moonshot/kimi-k3
        ```
      </Step>
      <Step title="सत्यापित करें कि मॉडल उपलब्ध हैं">
        ```bash
        openclaw models list --provider moonshot
        ```
      </Step>
      <Step title="लाइव स्मोक टेस्ट चलाएँ">
        अपनी सामान्य सेशन को प्रभावित किए बिना मॉडल एक्सेस और लागत
        ट्रैकिंग सत्यापित करने के लिए अलग स्टेट डायरेक्टरी का उपयोग करें:

        ```bash
        OPENCLAW_CONFIG_PATH=/tmp/openclaw-kimi/openclaw.json \
        OPENCLAW_STATE_DIR=/tmp/openclaw-kimi \
        openclaw agent --local \
          --session-id live-kimi-cost \
          --message 'ठीक यही उत्तर दें: KIMI_LIVE_OK' \
          --thinking max \
          --json
        ```

        JSON प्रतिक्रिया में `provider: "moonshot"` और
        `model: "kimi-k3"` रिपोर्ट होने चाहिए। जब Moonshot उपयोग मेटाडेटा लौटाता है, तब सहायक ट्रांसक्रिप्ट प्रविष्टि
        सामान्यीकृत टोकन उपयोग और अनुमानित लागत को `usage.cost` के अंतर्गत संग्रहीत करती है।
      </Step>
    </Steps>

    ### कॉन्फ़िगरेशन उदाहरण

    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: {
            // moonshot-kimi-k2-aliases:start
            "moonshot/kimi-k2.6": { alias: "Kimi K2.6" },
            "moonshot/kimi-k3": { alias: "Kimi K3" },
            "moonshot/kimi-k2.7-code": { alias: "Kimi K2.7 Code" },
            "moonshot/kimi-k2.7-code-highspeed": { alias: "Kimi K2.7 Code HighSpeed" },
            "moonshot/kimi-k2.5": { alias: "Kimi K2.5" },
            // moonshot-kimi-k2-aliases:end
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              // moonshot-kimi-k2-models:start
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k3",
                name: "Kimi K3",
                reasoning: true,
                thinkingLevelMap: {
                  off: null,
                  minimal: null,
                  low: null,
                  medium: null,
                  high: null,
                  xhigh: "max",
                  max: "max",
                },
                input: ["text", "image"],
                cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 0 },
                contextWindow: 1048576,
                maxTokens: 1048576,
              },
              {
                id: "kimi-k2.7-code",
                name: "Kimi K2.7 Code",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.19, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.7-code-highspeed",
                name: "Kimi K2.7 Code HighSpeed",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 1.9, output: 8, cacheRead: 0.38, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.5",
                name: "Kimi K2.5",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.6, output: 3, cacheRead: 0.1, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              // moonshot-kimi-k2-models:end
            ],
          },
        },
      },
    }
    ```

  </Tab>

  <Tab title="Kimi Coding">
    **इनके लिए सर्वोत्तम:** Kimi Coding एंडपॉइंट के माध्यम से कोड-केंद्रित कार्य।

    <Note>
    Kimi Coding, Moonshot (`moonshot/...`) से अलग API कुंजी और प्रदाता उपसर्ग (`kimi/...`) का उपयोग करता है। वर्तमान संदर्भों में 256K कॉन्टेक्स्ट के लिए `kimi/k3`, 1M स्तर के लिए `kimi/k3[1m]`, `kimi/kimi-for-coding`, और `kimi/kimi-for-coding-highspeed` शामिल हैं। पुराने संदर्भ `kimi/kimi-code` और `kimi/k2p5` अभी भी स्वीकार किए जाते हैं और `kimi/kimi-for-coding` में सामान्यीकृत होते हैं।
    </Note>

    कोडिंग सेवा OpenAI-संगत
    `https://api.kimi.com/coding/v1` और Anthropic-संगत
    `https://api.kimi.com/coding/` क्लाइंट, दोनों को स्वीकार करती है। यह plugin Anthropic Messages का उपयोग करता है।
    सदस्यता कुंजियाँ
    [Kimi Code Console](https://www.kimi.com/code/console) में बनाएँ; वर्तमान सदस्यता
    मूल्य [Kimi के मूल्य निर्धारण पेज](https://www.kimi.com/membership/pricing) पर उपलब्ध हैं।

    <Steps>
      <Step title="Plugin इंस्टॉल करें">
        ```bash
        openclaw plugins install @openclaw/kimi-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="ऑनबोर्डिंग चलाएँ">
        ```bash
        openclaw onboard --auth-choice kimi-code-api-key
        ```
      </Step>
      <Step title="डिफ़ॉल्ट मॉडल सेट करें">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "kimi/kimi-for-coding" },
            },
          },
        }
        ```
      </Step>
      <Step title="सत्यापित करें कि मॉडल उपलब्ध है">
        ```bash
        openclaw models list --provider kimi
        ```
      </Step>
    </Steps>

    Kimi Code K3 में `max` पर डीप थिंकिंग डिफ़ॉल्ट होती है। `/think off`,
    `thinking.type: "disabled"` भेजता है; `/think max` अधिकतम प्रयास के साथ K3 का एडेप्टिव-थिंकिंग
    अनुरोध भेजता है। पुराने निम्न थिंकिंग स्तर समर्थित
    `max` स्तर में बदल जाते हैं। 1M मॉडल के लिए Allegretto या उससे उच्च Kimi
    सदस्यता आवश्यक है; Moderato पर `kimi/k3` का उपयोग करें।

    वर्तमान प्लान उपलब्धता के लिए आधिकारिक [Kimi Code मॉडल तालिका](https://www.kimi.com/code/docs/en/kimi-code/models.html) देखें।

    ### कॉन्फ़िगरेशन उदाहरण

    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: {
            "kimi/kimi-for-coding": { alias: "Kimi" },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

## Kimi वेब खोज

Moonshot plugin, Moonshot वेब खोज द्वारा समर्थित **Kimi** को `web_search` प्रदाता के रूप में भी पंजीकृत करता है।

<Steps>
  <Step title="इंटरैक्टिव वेब खोज सेटअप चलाएँ">
    ```bash
    openclaw configure --section web
    ```

    `plugins.entries.moonshot.config.webSearch.*` संग्रहीत करने के लिए वेब-खोज अनुभाग में
    **Kimi** चुनें।

  </Step>
  <Step title="वेब खोज क्षेत्र और मॉडल कॉन्फ़िगर करें">
    इंटरैक्टिव सेटअप इनके लिए संकेत देता है:

    | सेटिंग             | विकल्प                                                              |
    | ------------------- | -------------------------------------------------------------------- |
    | API क्षेत्र          | `https://api.moonshot.ai/v1` (अंतरराष्ट्रीय) या `https://api.moonshot.cn/v1` (चीन) |
    | वेब खोज मॉडल    | डिफ़ॉल्ट रूप से `kimi-k2.6`                                             |

  </Step>
</Steps>

कॉन्फ़िगरेशन `plugins.entries.moonshot.config.webSearch` के अंतर्गत रहता है:

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // या KIMI_API_KEY / MOONSHOT_API_KEY का उपयोग करें
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.6",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

## उन्नत कॉन्फ़िगरेशन

<AccordionGroup>
  <Accordion title="नेटिव थिंकिंग मोड">
    Moonshot API Kimi K3 हमेशा अधिकतम प्रयास से रीजनिंग करता है। OpenClaw केवल
    `/think max` उपलब्ध कराता है, `reasoning_effort: "max"` भेजता है और पुराने निम्न या
    `off` सेटिंग को अनदेखा करता है।

    Kimi Code K3 `/think off|max` उपलब्ध कराता है। इसका Anthropic-संगत एंडपॉइंट
    बंद करने के लिए `thinking.type: "disabled"` या अधिकतम के लिए
    `output_config.effort: "max"` वाली अनुकूली सोच प्राप्त करता है। यह `kimi/k3` और
    `kimi/k3[1m]` दोनों पर लागू होता है।
    Moonshot API K3 `auto`, `none`, `required` और पिन किए गए टूल विकल्पों का समर्थन करता है,
    इसलिए OpenClaw अनुरोधित `tool_choice` को बनाए रखता है। बहु-टर्न टूल उपयोग के लिए,
    OpenClaw Moonshot के रीप्ले अनुबंध द्वारा आवश्यक सहायक रीजनिंग सामग्री को
    बनाए रखता है।

    Kimi K2.7 Code हमेशा मूल सोच का उपयोग करता है। Moonshot के अनुसार क्लाइंट को
    इस मॉडल के लिए `thinking` फ़ील्ड छोड़ना आवश्यक है, इसलिए OpenClaw केवल `on` उपलब्ध कराता है और
    पुराने `off` सेटिंग्स की उपेक्षा करता है। K2.7 `temperature`, `top_p`, `n`,
    `presence_penalty` और `frequency_penalty` को भी नियत करता है; OpenClaw उन फ़ील्ड के लिए कॉन्फ़िगर किए गए
    ओवरराइड छोड़ देता है।

    अन्य Moonshot Kimi मॉडल द्विआधारी मूल सोच का समर्थन करते हैं:

    - `thinking: { type: "enabled" }`
    - `thinking: { type: "disabled" }`

    इसे प्रत्येक मॉडल के लिए `agents.defaults.models.<provider/model>.params` के माध्यम से कॉन्फ़िगर करें:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "disabled" },
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw उन मॉडलों के लिए रनटाइम `/think` स्तरों को मैप करता है:

    | `/think` स्तर       | Moonshot का व्यवहार          |
    | -------------------- | -------------------------- |
    | `/think off`         | `thinking.type=disabled`   |
    | बंद के अतिरिक्त कोई भी स्तर    | `thinking.type=enabled`    |

    <Warning>
    जब Moonshot K2 की सोच सक्षम होती है, तो `tool_choice` का `auto` या `none` होना आवश्यक है। इसके बजाय, पिन किया गया टूल विकल्प (`type: "tool"` या `type: "function"`) सोच को वापस `disabled` पर बाध्य करता है, ताकि अनुरोधित टूल फिर भी चले; इसके बजाय `tool_choice: "required"` को `auto` में सामान्यीकृत किया जाता है। Kimi K2.7 Code सोच को अक्षम नहीं कर सकता, इसलिए इसके असंगत `tool_choice` को `auto` में सामान्यीकृत किया जाता है। Kimi K3 अपने अलग रीजनिंग-एफ़र्ट अनुबंध का उपयोग करता है और समर्थित टूल विकल्पों को बनाए रखता है।
    </Warning>

    Kimi K2.6 एक वैकल्पिक `thinking.keep` फ़ील्ड भी स्वीकार करता है, जो
    `reasoning_content` की बहु-टर्न प्रतिधारण अवधि को नियंत्रित करता है। सभी टर्न में पूर्ण
    रीजनिंग बनाए रखने के लिए इसे `"all"` पर सेट करें; सर्वर की डिफ़ॉल्ट रणनीति का उपयोग करने के लिए इसे छोड़ दें
    (या इसे `null` ही रहने दें)। OpenClaw केवल
    `moonshot/kimi-k2.6` के लिए `thinking.keep` अग्रेषित करता है और इसे अन्य मॉडलों से हटा देता है। Kimi K2.7 Code
    डिफ़ॉल्ट रूप से पूर्ण रीजनिंग इतिहास बनाए रखता है, जबकि OpenClaw संपूर्ण
    `thinking` फ़ील्ड छोड़ देता है।

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "enabled", keep: "all" },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="टूल कॉल आईडी का स्वच्छीकरण">
    Moonshot Kimi `functions.<name>:<index>` जैसे मूल tool_call आईडी प्रदान करता है। OpenClaw प्रत्येक मूल Kimi आईडी की पहली उपस्थिति को बनाए रखता है और बाद की डुप्लिकेट आईडी को नियतात्मक OpenAI-शैली की `call_*` आईडी में फिर से लिखता है। मेल खाने वाले टूल परिणामों को उसी आईडी के साथ फिर से मैप किया जाता है, ताकि Kimi की पहली मूल आईडी हटाए बिना रीप्ले अद्वितीय बना रहे। यह व्यवहार बंडल किए गए Moonshot प्रोवाइडर में अंतर्निहित है और उपयोगकर्ता द्वारा कॉन्फ़िगर की जा सकने वाली सेटिंग नहीं है।
  </Accordion>

  <Accordion title="स्ट्रीमिंग उपयोग संगतता">
    मूल Moonshot एंडपॉइंट (`https://api.moonshot.ai/v1` और
    `https://api.moonshot.cn/v1`) स्ट्रीमिंग उपयोग संगतता घोषित करते हैं।
    OpenClaw इसका निर्धारण प्रोवाइडर आईडी के बजाय एंडपॉइंट होस्ट से करता है, इसलिए उसी मूल
    Moonshot होस्ट की ओर इंगित करने वाली कस्टम प्रोवाइडर आईडी को वही
    स्ट्रीमिंग-उपयोग व्यवहार मिलता है।

    कैटलॉग K2.6 मूल्य निर्धारण के साथ, इनपुट, आउटपुट और
    कैश-पठन टोकन शामिल करने वाले स्ट्रीम किए गए उपयोग को `/status`, `/usage full`, `/usage cost` और ट्रांसक्रिप्ट-समर्थित सत्र
    लेखांकन के लिए स्थानीय अनुमानित USD लागत में भी बदला जाता है।

  </Accordion>

  <Accordion title="एंडपॉइंट और मॉडल रेफ़रेंस संदर्भ">
    | प्रोवाइडर   | मॉडल रेफ़रेंस उपसर्ग | एंडपॉइंट                      | प्रमाणीकरण एनवायरनमेंट वैरिएबल        |
    | ---------- | ---------------- | ------------------------------ | ------------------- |
    | Moonshot   | `moonshot/`      | `https://api.moonshot.ai/v1`  | `MOONSHOT_API_KEY`  |
    | Moonshot CN| `moonshot/`      | `https://api.moonshot.cn/v1`  | `MOONSHOT_API_KEY`  |
    | Kimi Coding| `kimi/`          | Kimi Coding एंडपॉइंट           | `KIMI_API_KEY`      |
    | वेब खोज | लागू नहीं              | Moonshot API क्षेत्र के समान    | `KIMI_API_KEY` या `MOONSHOT_API_KEY` |

    - Kimi वेब खोज `KIMI_API_KEY` या `MOONSHOT_API_KEY` का उपयोग करती है और मॉडल `kimi-k2.6` के साथ डिफ़ॉल्ट रूप से `https://api.moonshot.ai/v1` का उपयोग करती है।
    - आवश्यकता होने पर `models.providers` में मूल्य निर्धारण और संदर्भ मेटाडेटा ओवरराइड करें।
    - यदि Moonshot किसी मॉडल के लिए अलग संदर्भ सीमाएँ प्रकाशित करता है, तो `contextWindow` को तदनुसार समायोजित करें।

  </Accordion>
</AccordionGroup>

## संबंधित

<CardGroup cols={2}>
  <Card title="मॉडल चयन" href="/hi/concepts/model-providers" icon="layers">
    प्रोवाइडर, मॉडल रेफ़रेंस और फ़ेलओवर व्यवहार चुनना।
  </Card>
  <Card title="वेब खोज" href="/hi/tools/web" icon="magnifying-glass">
    Kimi सहित वेब खोज प्रोवाइडर कॉन्फ़िगर करना।
  </Card>
  <Card title="कॉन्फ़िगरेशन संदर्भ" href="/hi/gateway/configuration-reference" icon="gear">
    प्रोवाइडर, मॉडल और plugins के लिए पूर्ण कॉन्फ़िग स्कीमा।
  </Card>
  <Card title="Moonshot Open Platform" href="https://platform.moonshot.ai" icon="globe">
    Moonshot API कुंजी प्रबंधन और दस्तावेज़।
  </Card>
</CardGroup>
