---
read_when:
    - आप OpenClaw में StepFun मॉडल चाहते हैं
    - आपको StepFun सेटअप मार्गदर्शन चाहिए
summary: OpenClaw के साथ StepFun मॉडल का उपयोग करें
title: StepFun
x-i18n:
    generated_at: "2026-07-27T18:28:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 462a2588f15e8d6188914e238a3e472052d0da1da151751adecdb63cf009fc64
    source_path: providers/stepfun.md
    workflow: 16
---

StepFun दो प्रदाता आईडी वाले एक बाहरी आधिकारिक plugin (`@openclaw/stepfun-provider`) के रूप में उपलब्ध है:

- `stepfun` मानक एंडपॉइंट के लिए
- `stepfun-plan` Step Plan एंडपॉइंट के लिए

<Warning>
मानक और Step Plan अलग-अलग एंडपॉइंट और मॉडल संदर्भ उपसर्गों (`stepfun/...` बनाम `stepfun-plan/...`) वाले **अलग प्रदाता** हैं। `.com` एंडपॉइंट के साथ चीन की कुंजी और `.ai` एंडपॉइंट के साथ वैश्विक कुंजी का उपयोग करें।
</Warning>

## Plugin इंस्टॉल करें

```bash
openclaw plugins install @openclaw/stepfun-provider
openclaw gateway restart
```

## क्षेत्र और एंडपॉइंट का अवलोकन

| एंडपॉइंट  | चीन (`.com`)                         | वैश्विक (`.ai`)                        |
| --------- | -------------------------------------- | ------------------------------------- |
| मानक  | `https://api.stepfun.com/v1`           | `https://api.stepfun.ai/v1`           |
| Step Plan | `https://api.stepfun.com/step_plan/v1` | `https://api.stepfun.ai/step_plan/v1` |

प्रमाणीकरण परिवेश चर: `STEPFUN_API_KEY`

## अंतर्निहित कैटलॉग

मानक (`stepfun`):

| मॉडल संदर्भ                | संदर्भ | अधिकतम आउटपुट | टिप्पणियाँ                          |
| ------------------------ | ------- | ---------- | ------------------------------ |
| `stepfun/step-3.5-flash` | 262,144 | 65,536     | डिफ़ॉल्ट मानक मॉडल         |
| `stepfun/step-3.7-flash` | 262,144 | 262,144    | बहुमाध्यमीय छवि इनपुट समर्थन |

Step Plan (`stepfun-plan`):

| मॉडल संदर्भ                          | संदर्भ | अधिकतम आउटपुट | टिप्पणियाँ                          |
| ---------------------------------- | ------- | ---------- | ------------------------------ |
| `stepfun-plan/step-3.5-flash`      | 262,144 | 65,536     | डिफ़ॉल्ट Step Plan मॉडल        |
| `stepfun-plan/step-3.7-flash`      | 262,144 | 262,144    | बहुमाध्यमीय छवि इनपुट समर्थन |
| `stepfun-plan/step-3.5-flash-2603` | 262,144 | 65,536     | अतिरिक्त Step Plan मॉडल     |

## आरंभ करना

<Tabs>
  <Tab title="मानक">
    मानक StepFun एंडपॉइंट के माध्यम से सामान्य प्रयोजन के उपयोग के लिए सर्वोत्तम।

    <Steps>
      <Step title="अपना एंडपॉइंट क्षेत्र चुनें">
        | प्रमाणीकरण विकल्प                    | एंडपॉइंट                     | क्षेत्र        |
        | -------------------------------- | ----------------------------- | -------------- |
        | `stepfun-standard-api-key-intl` | `https://api.stepfun.ai/v1`  | अंतरराष्ट्रीय |
        | `stepfun-standard-api-key-cn`   | `https://api.stepfun.com/v1` | चीन          |
      </Step>
      <Step title="ऑनबोर्डिंग चलाएँ">
        ```bash
        openclaw onboard --auth-choice stepfun-standard-api-key-intl
        ```

        चीन एंडपॉइंट:

        ```bash
        openclaw onboard --auth-choice stepfun-standard-api-key-cn
        ```
      </Step>
      <Step title="गैर-संवादात्मक विकल्प">
        ```bash
        openclaw onboard --auth-choice stepfun-standard-api-key-intl \
          --stepfun-api-key "$STEPFUN_API_KEY"
        ```
      </Step>
      <Step title="सत्यापित करें कि मॉडल उपलब्ध हैं">
        ```bash
        openclaw models list --provider stepfun
        ```
      </Step>
    </Steps>

    डिफ़ॉल्ट मॉडल: `stepfun/step-3.5-flash`
    वैकल्पिक मॉडल: `stepfun/step-3.7-flash`

  </Tab>

  <Tab title="Step Plan">
    Step Plan तर्क एंडपॉइंट के लिए सर्वोत्तम।

    <Steps>
      <Step title="अपना एंडपॉइंट क्षेत्र चुनें">
        | प्रमाणीकरण विकल्प                 | एंडपॉइंट                                | क्षेत्र        |
        | ------------------------------ | ------------------------------------------ | -------------- |
        | `stepfun-plan-api-key-intl` | `https://api.stepfun.ai/step_plan/v1`  | अंतरराष्ट्रीय |
        | `stepfun-plan-api-key-cn`   | `https://api.stepfun.com/step_plan/v1` | चीन          |
      </Step>
      <Step title="ऑनबोर्डिंग चलाएँ">
        ```bash
        openclaw onboard --auth-choice stepfun-plan-api-key-intl
        ```

        चीन एंडपॉइंट:

        ```bash
        openclaw onboard --auth-choice stepfun-plan-api-key-cn
        ```
      </Step>
      <Step title="गैर-संवादात्मक विकल्प">
        ```bash
        openclaw onboard --auth-choice stepfun-plan-api-key-intl \
          --stepfun-api-key "$STEPFUN_API_KEY"
        ```
      </Step>
      <Step title="सत्यापित करें कि मॉडल उपलब्ध हैं">
        ```bash
        openclaw models list --provider stepfun-plan
        ```
      </Step>
    </Steps>

    डिफ़ॉल्ट मॉडल: `stepfun-plan/step-3.5-flash`
    वैकल्पिक मॉडल: `stepfun-plan/step-3.7-flash`, `stepfun-plan/step-3.5-flash-2603`

  </Tab>
</Tabs>

एकल प्रमाणीकरण प्रवाह `stepfun` और `stepfun-plan` दोनों के लिए क्षेत्र से मेल खाने वाली प्रोफ़ाइल लिखता है, इसलिए एक बार ऑनबोर्डिंग चलाने के बाद दोनों सतहों का एक साथ पता लगाया जाता है।

## उन्नत कॉन्फ़िगरेशन

<AccordionGroup>
  <Accordion title="पूर्ण कॉन्फ़िगरेशन: मानक प्रदाता">
    ```json5
    {
      env: { STEPFUN_API_KEY: "your-key" },
      agents: { defaults: { model: { primary: "stepfun/step-3.5-flash" } } },
      models: {
        mode: "merge",
        providers: {
          stepfun: {
            baseUrl: "https://api.stepfun.ai/v1",
            api: "openai-completions",
            apiKey: "${STEPFUN_API_KEY}",
            models: [
              {
                id: "step-3.7-flash",
                name: "Step 3.7 Flash",
                reasoning: true,
                input: ["text", "image"],
                thinkingLevelMap: { off: "low", minimal: "low", xhigh: "high", max: "high" },
                cost: { input: 0.2, output: 1.15, cacheRead: 0.04, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "step-3.5-flash",
                name: "Step 3.5 Flash",
                reasoning: true,
                input: ["text"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="पूर्ण कॉन्फ़िगरेशन: Step Plan प्रदाता">
    ```json5
    {
      env: { STEPFUN_API_KEY: "your-key" },
      agents: { defaults: { model: { primary: "stepfun-plan/step-3.5-flash" } } },
      models: {
        mode: "merge",
        providers: {
          "stepfun-plan": {
            baseUrl: "https://api.stepfun.ai/step_plan/v1",
            api: "openai-completions",
            apiKey: "${STEPFUN_API_KEY}",
            models: [
              {
                id: "step-3.7-flash",
                name: "Step 3.7 Flash",
                reasoning: true,
                input: ["text", "image"],
                thinkingLevelMap: { off: "low", minimal: "low", xhigh: "high", max: "high" },
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "step-3.5-flash",
                name: "Step 3.5 Flash",
                reasoning: true,
                input: ["text"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
              {
                id: "step-3.5-flash-2603",
                name: "Step 3.5 Flash 2603",
                reasoning: true,
                input: ["text"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="टिप्पणियाँ">
    - `step-3.7-flash` OpenClaw के माध्यम से टेक्स्ट और छवि इनपुट स्वीकार करता है। StepFun का API वीडियो का भी समर्थन करता है, जो अभी तक OpenClaw में मॉडल इनपुट माध्यम नहीं है।
    - Step 3.7 `low`, `medium`, और `high` तर्क प्रयास का समर्थन करता है। चूँकि मॉडल में गैर-तर्क मोड नहीं है, इसलिए `/think off` को `low` पर मैप किया जाता है।
    - `step-3.5-flash-2603` वर्तमान में केवल `stepfun-plan` पर उपलब्ध है।
    - मॉडल देखने या बदलने के लिए `openclaw models list` और `openclaw models set <provider/model>` का उपयोग करें।

  </Accordion>
</AccordionGroup>

## संबंधित

<CardGroup cols={2}>
  <Card title="मॉडल प्रदाता" href="/hi/concepts/model-providers" icon="layers">
    सभी प्रदाताओं, मॉडल संदर्भों और फ़ेलओवर व्यवहार का अवलोकन।
  </Card>
  <Card title="कॉन्फ़िगरेशन संदर्भ" href="/hi/gateway/configuration-reference" icon="gear">
    प्रदाताओं, मॉडलों और plugins के लिए पूर्ण कॉन्फ़िगरेशन स्कीमा।
  </Card>
  <Card title="मॉडल CLI" href="/hi/concepts/models" icon="brain">
    मॉडल चुनने और कॉन्फ़िगर करने का तरीका।
  </Card>
  <Card title="StepFun प्लेटफ़ॉर्म" href="https://platform.stepfun.com" icon="globe">
    StepFun API कुंजी प्रबंधन और दस्तावेज़।
  </Card>
</CardGroup>
