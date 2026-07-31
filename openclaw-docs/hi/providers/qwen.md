---
read_when:
    - आप OpenClaw के साथ Qwen का उपयोग करना चाहते हैं
    - आपके पास Alibaba Cloud Token Plan की सदस्यता है
summary: इसके OpenClaw plugin के माध्यम से Qwen Cloud का उपयोग करें
title: Qwen
x-i18n:
    generated_at: "2026-07-27T20:24:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 74f94a35631dcdf8c9afc12e86d7a9d6b51a359411ba36f8820f8b1e7c03a27a
    source_path: providers/qwen.md
    workflow: 16
---

Qwen Cloud, canonical id `qwen` वाला एक आधिकारिक बाहरी OpenClaw provider plugin है। यह Qwen Cloud / Alibaba DashScope Standard और Coding Plan endpoints को लक्षित करता है, Token Plan को `qwen-token-plan` के रूप में उपलब्ध कराता है, `modelstudio` को compatibility alias के रूप में रखता है, और Alibaba के दस्तावेज़ीकृत `bailian-token-plan` custom-provider id का स्वतंत्र रूप से स्वामी है।

| प्रॉपर्टी                  | मान                                        |
| ------------------------- | ------------------------------------------ |
| Provider                  | `qwen`                         |
| Token Plan provider       | `qwen-token-plan`                         |
| पसंदीदा env var           | `QWEN_API_KEY`                         |
| Token Plan env var        | `QWEN_TOKEN_PLAN_API_KEY`                         |
| यह भी स्वीकृत (compat)    | `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY`     |
| API शैली                  | OpenAI-संगत                                |

<Tip>
`qwen3.7-plus` और `qwen3.6-plus`, Coding Plan और Standard endpoints के साथ काम करते हैं।
`qwen3.7-max` या `qwen3.6-flash` के लिए, **Standard (उपयोग के अनुसार भुगतान)** endpoint का उपयोग करें।
</Tip>

## Plugin इंस्टॉल करें

`qwen` एक आधिकारिक बाहरी plugin के रूप में वितरित होता है और core के साथ bundled नहीं है। इसे इंस्टॉल करें और Gateway को पुनः आरंभ करें:

```bash
openclaw plugins install @openclaw/qwen-provider
openclaw gateway restart
```

## आरंभ करना

अपना plan प्रकार चुनें और setup चरणों का पालन करें।

<Tabs>
  <Tab title="Coding Plan (सदस्यता)">
    **इनके लिए सर्वोत्तम:** Qwen Coding Plan के माध्यम से सदस्यता-आधारित एक्सेस।

    <Steps>
      <Step title="अपनी API key प्राप्त करें">
        [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) से API key बनाएँ या कॉपी करें।
      </Step>
      <Step title="ऑनबोर्डिंग चलाएँ">
        **Global** endpoint के लिए:

        ```bash
        openclaw onboard --auth-choice qwen-api-key
        ```

        **China** endpoint के लिए:

        ```bash
        openclaw onboard --auth-choice qwen-api-key-cn
        ```
      </Step>
      <Step title="डिफ़ॉल्ट मॉडल सेट करें">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "qwen/qwen3.5-plus" },
            },
          },
        }
        ```
      </Step>
      <Step title="पुष्टि करें कि मॉडल उपलब्ध है">
        ```bash
        openclaw models list --provider qwen
        ```
      </Step>
    </Steps>

    <Note>
    पुराने `modelstudio-*` auth-choice ids और `modelstudio/...` model refs अभी भी
    compatibility aliases के रूप में काम करते हैं, लेकिन नए setup flows में canonical
    `qwen-*` auth-choice ids और `qwen/...` model refs को प्राथमिकता देनी चाहिए। यदि आप किसी अन्य `api` मान के साथ सटीक
    custom `models.providers.modelstudio` entry परिभाषित करते हैं, तो वह
    custom provider, Qwen compatibility
    alias के बजाय `modelstudio/...` refs का स्वामी होता है।
    </Note>

  </Tab>

  <Tab title="Standard (उपयोग के अनुसार भुगतान)">
    **इनके लिए सर्वोत्तम:** Standard Model Studio endpoint के माध्यम से उपयोग के अनुसार भुगतान वाला एक्सेस, जिसमें `qwen3.7-max` और `qwen3.6-flash` भी शामिल हैं, जो Coding Plan पर उपलब्ध नहीं हैं।

    <Steps>
      <Step title="अपनी API key प्राप्त करें">
        [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) से API key बनाएँ या कॉपी करें।
      </Step>
      <Step title="ऑनबोर्डिंग चलाएँ">
        **Global** endpoint के लिए:

        ```bash
        openclaw onboard --auth-choice qwen-standard-api-key
        ```

        **China** endpoint के लिए:

        ```bash
        openclaw onboard --auth-choice qwen-standard-api-key-cn
        ```
      </Step>
      <Step title="डिफ़ॉल्ट मॉडल सेट करें">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "qwen/qwen3.5-plus" },
            },
          },
        }
        ```
      </Step>
      <Step title="पुष्टि करें कि मॉडल उपलब्ध है">
        ```bash
        openclaw models list --provider qwen
        ```
      </Step>
    </Steps>

    <Note>
    पुराने `modelstudio-*` auth-choice ids और `modelstudio/...` model refs अभी भी
    compatibility aliases के रूप में काम करते हैं, लेकिन नए setup flows में canonical
    `qwen-*` auth-choice ids और `qwen/...` model refs को प्राथमिकता देनी चाहिए। यदि आप किसी अन्य `api` मान के साथ सटीक
    custom `models.providers.modelstudio` entry परिभाषित करते हैं, तो वह
    custom provider, Qwen compatibility
    alias के बजाय `modelstudio/...` refs का स्वामी होता है।
    </Note>

  </Tab>

  <Tab title="Token Plan (Team Edition)">
    **इनके लिए सर्वोत्तम:** Alibaba Cloud Model Studio के माध्यम से Qwen और समर्थित तृतीय-पक्ष मॉडलों तक क्रेडिट-आधारित टीम सदस्यता एक्सेस।

    <Steps>
      <Step title="अपनी समर्पित key प्राप्त करें">
        Token Plan seat असाइन करें और उसकी समर्पित `sk-sp-...` key बनाएँ। Token Plan, Coding Plan और उपयोग के अनुसार भुगतान वाली keys परस्पर विनिमेय नहीं हैं। [Global Token Plan का अवलोकन](https://www.alibabacloud.com/help/en/model-studio/token-plan-overview) या [China Token Plan का अवलोकन](https://help.aliyun.com/zh/model-studio/token-plan-overview) देखें।
      </Step>
      <Step title="ऑनबोर्डिंग चलाएँ">
        Singapore में **Global / International** endpoint के लिए:

        ```bash
        openclaw onboard --auth-choice qwen-token-plan
        ```

        Beijing में **China** endpoint के लिए:

        ```bash
        openclaw onboard --auth-choice qwen-token-plan-cn
        ```
      </Step>
      <Step title="Provider की पुष्टि करें">
        ```bash
        openclaw models list --provider qwen-token-plan
        openclaw agent --model qwen-token-plan/qwen3.7-plus --message "उत्तर दें: टोकन प्लान तैयार है"
        ```
      </Step>
    </Steps>

    <Note>
    Alibaba की OpenClaw मार्गदर्शिका, मैन्युअल custom
    provider के लिए `bailian-token-plan` का उपयोग करती है। Plugin उस id को compatibility owner के रूप में पंजीकृत करता है, लेकिन नए
    configs में `qwen-token-plan` का उपयोग करना चाहिए। सटीक custom
    `models.providers.bailian-token-plan` entry अपने configured
    transport और catalog का स्वामित्व बनाए रखती है; इसे canonical OpenAI catalog में कभी merge नहीं किया जाता।
    </Note>

    <Warning>
    Token Plan का उपयोग केवल interactive OpenClaw sessions के लिए करें। इसे
    cron jobs, unattended scripts या application backends के लिए न चुनें। Alibaba के अनुसार,
    non-interactive उपयोग से सदस्यता निलंबित हो सकती है या उसकी API key निरस्त की जा सकती है।
    </Warning>

  </Tab>

</Tabs>

## Plan के प्रकार और endpoints

| Plan                         | क्षेत्र | Auth choice                | Endpoint                                                         |
| ---------------------------- | ------- | -------------------------- | ---------------------------------------------------------------- |
| Coding Plan (सदस्यता)        | China   | `qwen-api-key-cn`         | `coding.dashscope.aliyuncs.com/v1`                                               |
| Coding Plan (सदस्यता)        | Global  | `qwen-api-key`         | `coding-intl.dashscope.aliyuncs.com/v1`                                               |
| Standard (उपयोग के अनुसार भुगतान) | China   | `qwen-standard-api-key-cn`         | `dashscope.aliyuncs.com/compatible-mode/v1`                                               |
| Standard (उपयोग के अनुसार भुगतान) | Global  | `qwen-standard-api-key`         | `dashscope-intl.aliyuncs.com/compatible-mode/v1`                                               |
| Token Plan (Team Edition)    | China   | `qwen-token-plan-cn`         | `token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`                                               |
| Token Plan (Team Edition)    | Global  | `qwen-token-plan`         | `token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1`                                               |

Provider आपके auth choice के आधार पर endpoint स्वतः चुनता है। Canonical
choices, `qwen-*` family का उपयोग करते हैं; `modelstudio-*` केवल compatibility के लिए बना हुआ है।
Config में custom `baseUrl` से इसे override करें।

<Tip>
**Keys प्रबंधित करें:** [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) |
**दस्तावेज़:** [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)
</Tip>

## अंतर्निहित catalog

OpenClaw इस Qwen static catalog के साथ आता है। Catalog endpoint-aware है: Coding
Plan configs उन मॉडलों को छोड़ देते हैं जो केवल Standard endpoint पर काम करते हैं।

| Model ref                   | इनपुट      | Context   | टिप्पणियाँ                   |
| --------------------------- | ---------- | --------- | ---------------------------- |
| `qwen/qwen3.5-plus`          | टेक्स्ट, इमेज | 1,000,000 | डिफ़ॉल्ट मॉडल             |
| `qwen/qwen3.6-flash`          | टेक्स्ट, इमेज | 1,000,000 | केवल Standard endpoints   |
| `qwen/qwen3.6-plus`          | टेक्स्ट, इमेज | 1,000,000 | Coding Plan + Standard     |
| `qwen/qwen3.7-max`          | टेक्स्ट     | 1,000,000 | केवल Standard endpoints     |
| `qwen/qwen3.7-plus`          | टेक्स्ट, इमेज | 1,000,000 | Coding Plan + Standard     |
| `qwen/qwen3-max-2026-01-23`          | टेक्स्ट     | 262,144   | Qwen Max श्रृंखला            |
| `qwen/qwen3-coder-next`          | टेक्स्ट     | 262,144   | कोडिंग                       |
| `qwen/qwen3-coder-plus`          | टेक्स्ट     | 1,000,000 | कोडिंग                       |
| `qwen/MiniMax-M2.5`          | टेक्स्ट     | 1,000,000 | रीजनिंग सक्षम                |
| `qwen/glm-5`          | टेक्स्ट     | 202,752   | GLM                          |
| `qwen/glm-4.7`          | टेक्स्ट     | 202,752   | GLM                          |
| `qwen/kimi-k2.5`          | टेक्स्ट, इमेज | 262,144   | Alibaba के माध्यम से Moonshot AI |

<Note>
Static catalog में मॉडल मौजूद होने पर भी उसकी उपलब्धता endpoint और billing plan के अनुसार
अलग-अलग हो सकती है।
</Note>

### Token Plan catalog

Token Plan एक अलग exact-string allowlist का उपयोग करता है। केवल image-generation वाले plan
models यहाँ शामिल नहीं हैं, क्योंकि वे अलग APIs का उपयोग करते हैं।

| Model ref                   | इनपुट      | Context   |
| --------------------------- | ---------- | --------- |
| `qwen-token-plan/qwen3.7-max`          | टेक्स्ट     | 1,000,000 |
| `qwen-token-plan/qwen3.7-plus`          | टेक्स्ट, इमेज | 1,000,000 |
| `qwen-token-plan/qwen3.6-plus`          | टेक्स्ट, इमेज | 1,000,000 |
| `qwen-token-plan/qwen3.6-flash`          | टेक्स्ट, इमेज | 1,000,000 |
| `qwen-token-plan/deepseek-v4-pro`          | टेक्स्ट     | 1,000,000 |
| `qwen-token-plan/deepseek-v4-flash`          | टेक्स्ट     | 1,000,000 |
| `qwen-token-plan/deepseek-v3.2`          | टेक्स्ट     | 131,072   |
| `qwen-token-plan/kimi-k2.7-code`          | टेक्स्ट, इमेज | 262,144   |
| `qwen-token-plan/kimi-k2.6`          | टेक्स्ट, इमेज | 262,144   |
| `qwen-token-plan/kimi-k2.5`          | टेक्स्ट, इमेज | 262,144   |
| `qwen-token-plan/glm-5.2`          | टेक्स्ट     | 1,000,000 |
| `qwen-token-plan/glm-5.1`          | टेक्स्ट     | 202,752   |
| `qwen-token-plan/glm-5`          | टेक्स्ट     | 202,752   |
| `qwen-token-plan/MiniMax-M2.5`          | टेक्स्ट     | 196,608   |

## चिंतन नियंत्रण

`qwen3.7-max`, `qwen3.7-plus`, `qwen3.6-flash` और `qwen3.6-plus`,
अंतर्निहित catalog में रीजनिंग-सक्षम हैं। `qwen`
family के रीजनिंग मॉडलों के लिए, provider OpenClaw के thinking levels को DashScope के शीर्ष-स्तरीय
`enable_thinking` request flag से मैप करता है: thinking अक्षम होने पर `enable_thinking: false` भेजा जाता है,
और किसी अन्य level पर `enable_thinking: true` भेजा जाता है। Custom models,
model entry पर `compat.thinkingFormat: "qwen-chat-template"` सेट करके
वैकल्पिक chat-template thinking payload चुन सकते हैं।

Token Plan models भी रीजनिंग-सक्षम के रूप में चिह्नित हैं। `kimi-k2.7-code` और
`MiniMax-M2.5` केवल thinking वाले हैं, इसलिए session द्वारा `/think off` का अनुरोध करने पर भी
OpenClaw thinking को सक्षम रखता है। DeepSeek V4, `minimal` से `high` को
service के `high` effort से मैप करता है और `xhigh` या `max` को `max` से मैप करता है। GLM 5.2,
`minimal` से `max` तक की पूरी range स्वीकार करता है; GLM 5.1 और GLM 5,
`xhigh` तक स्वीकार करते हैं, और तीनों का डिफ़ॉल्ट `high` है। अन्य hybrid models
अनुरोधित on/off स्थिति का पालन करते हैं।

## मल्टीमॉडल ऐड-ऑन

`qwen` plugin केवल **Standard** DashScope
endpoints पर मल्टीमॉडल क्षमताएँ उपलब्ध कराता है, Coding Plan endpoints पर नहीं:

- **इमेज और वीडियो की समझ** `qwen3.6-plus` के माध्यम से
- **Wan वीडियो जनरेशन** `wan2.6-t2v` (डिफ़ॉल्ट), `wan2.6-i2v`, `wan2.6-r2v`, `wan2.6-r2v-flash`, `wan2.7-r2v` के माध्यम से

Media understanding को configured Qwen auth से स्वतः resolve किया जाता है; किसी अतिरिक्त
config की आवश्यकता नहीं है। Media understanding के काम करने के लिए सुनिश्चित करें कि आप Standard (उपयोग के अनुसार भुगतान) endpoint पर हैं।

Qwen को डिफ़ॉल्ट video provider बनाने के लिए:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

वीडियो जनरेशन सीमाएँ: प्रति अनुरोध 1 आउटपुट वीडियो, अधिकतम 1 इनपुट इमेज
(इमेज-टू-वीडियो), अधिकतम 4 इनपुट वीडियो (वीडियो-टू-वीडियो), अधिकतम 10 सेकंड
की अवधि। `size`, `aspectRatio`, `resolution`, `audio`, और
`watermark` का समर्थन करता है। संदर्भ इमेज/वीडियो इनपुट के लिए दूरस्थ http(s) URL आवश्यक हैं; स्थानीय
फ़ाइल पाथ पहले ही अस्वीकार कर दिए जाते हैं, क्योंकि DashScope वीडियो एंडपॉइंट उन संदर्भों के लिए
अपलोड किए गए स्थानीय बफ़र स्वीकार नहीं करता।

<Note>
साझा टूल पैरामीटर, प्रदाता चयन और फ़ेलओवर व्यवहार के लिए [वीडियो जनरेशन](/hi/tools/video-generation) देखें।
</Note>

## उन्नत कॉन्फ़िगरेशन

<AccordionGroup>
  <Accordion title="Qwen 3.6 और 3.7 की उपलब्धता">
    `qwen3.7-plus` और `qwen3.6-plus` Coding Plan तथा Standard एंडपॉइंट पर उपलब्ध हैं। `qwen3.7-max` और `qwen3.6-flash` केवल Standard पर उपलब्ध हैं। Standard (उपयोग के अनुसार भुगतान) एंडपॉइंट ये हैं:

    - चीन: `dashscope.aliyuncs.com/compatible-mode/v1`
    - वैश्विक: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

    OpenClaw Coding Plan कैटलॉग से `qwen3.7-max` और `qwen3.6-flash` को शामिल नहीं करता।
    यदि Coding Plan एंडपॉइंट इनमें से किसी के लिए "unsupported model" त्रुटि लौटाता है,
    तो संबंधित Standard एंडपॉइंट और कुंजी पर स्विच करें।

  </Accordion>

  <Accordion title="वीडियो जनरेशन क्षेत्र रूटिंग">
    वीडियो जॉब सबमिट करने से पहले OpenClaw कॉन्फ़िगर किए गए Qwen क्षेत्र को संबंधित DashScope AIGC होस्ट से
    मैप करता है:

    - वैश्विक/अंतरराष्ट्रीय: `https://dashscope-intl.aliyuncs.com`
    - चीन: `https://dashscope.aliyuncs.com`

    Coding Plan या Standard Qwen होस्ट में से किसी की ओर संकेत करने वाला सामान्य
    `models.providers.qwen.baseUrl` भी वीडियो जनरेशन को संबंधित
    क्षेत्रीय DashScope वीडियो एंडपॉइंट पर रूट करता है।

  </Accordion>

  <Accordion title="स्ट्रीमिंग उपयोग संगतता">
    नेटिव Qwen एंडपॉइंट साझा `openai-completions` ट्रांसपोर्ट पर स्ट्रीमिंग उपयोग संगतता
    दर्शाते हैं, इसलिए समान नेटिव होस्ट को लक्षित करने वाली DashScope-संगत कस्टम प्रदाता आईडी,
    विशेष रूप से अंतर्निहित `qwen` प्रदाता आईडी की आवश्यकता के बिना,
    वही व्यवहार प्राप्त करती हैं। यह Coding Plan, Standard और Token Plan एंडपॉइंट पर लागू होता है:

    - `https://coding.dashscope.aliyuncs.com/v1`
    - `https://coding-intl.dashscope.aliyuncs.com/v1`
    - `https://dashscope.aliyuncs.com/compatible-mode/v1`
    - `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
    - `https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1`
    - `https://token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`

  </Accordion>

  <Accordion title="क्षमता योजना">
    `qwen` Plugin को केवल कोडिंग/टेक्स्ट मॉडल के लिए नहीं, बल्कि संपूर्ण Qwen
    Cloud सतह के विक्रेता-केंद्र के रूप में स्थापित किया जा रहा है।

    - **टेक्स्ट/चैट मॉडल:** Plugin के माध्यम से उपलब्ध
    - **टूल कॉलिंग, संरचित आउटपुट, चिंतन:** OpenAI-संगत ट्रांसपोर्ट से प्राप्त
    - **इमेज जनरेशन:** प्रदाता-Plugin स्तर पर नियोजित
    - **इमेज/वीडियो समझ:** Standard एंडपॉइंट पर Plugin के माध्यम से उपलब्ध
    - **स्पीच/ऑडियो:** प्रदाता-Plugin स्तर पर नियोजित
    - **मेमोरी एम्बेडिंग/रीरैंकिंग:** एम्बेडिंग अडैप्टर सतह के माध्यम से नियोजित
    - **वीडियो जनरेशन:** साझा वीडियो-जनरेशन क्षमता के माध्यम से Plugin में उपलब्ध

  </Accordion>

  <Accordion title="परिवेश और डेमन सेटअप">
    यदि Gateway डेमन (launchd/systemd) के रूप में चलता है, तो सुनिश्चित करें कि `QWEN_API_KEY`
    या `QWEN_TOKEN_PLAN_API_KEY` उस प्रक्रिया के लिए उपलब्ध हो (उदाहरण के लिए,
    `~/.openclaw/.env` में या `env.shellEnv` के माध्यम से)।
  </Accordion>
</AccordionGroup>

## संबंधित

<CardGroup cols={2}>
  <Card title="मॉडल चयन" href="/hi/concepts/model-providers" icon="layers">
    प्रदाताओं, मॉडल संदर्भों और फ़ेलओवर व्यवहार का चयन।
  </Card>
  <Card title="वीडियो जनरेशन" href="/hi/tools/video-generation" icon="video">
    साझा वीडियो टूल पैरामीटर और प्रदाता चयन।
  </Card>
  <Card title="Alibaba Model Studio" href="/hi/providers/alibaba" icon="cloud">
    समान DashScope प्लेटफ़ॉर्म पर बंडल किया गया Wan वीडियो जनरेशन प्रदाता।
  </Card>
  <Card title="समस्या निवारण" href="/hi/help/troubleshooting" icon="wrench">
    सामान्य समस्या निवारण और अक्सर पूछे जाने वाले प्रश्न।
  </Card>
</CardGroup>
