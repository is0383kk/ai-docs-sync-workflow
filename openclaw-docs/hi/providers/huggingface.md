---
read_when:
    - आप OpenClaw के साथ Hugging Face Inference का उपयोग करना चाहते हैं
    - आपको HF टोकन एनवायरनमेंट वेरिएबल या CLI प्रमाणीकरण विकल्प की आवश्यकता है
summary: Hugging Face Inference सेटअप (प्रमाणीकरण + मॉडल चयन)
title: Hugging Face (इन्फ़रेंस)
x-i18n:
    generated_at: "2026-07-27T18:25:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 92c400b78c5ad2cc724ad4029560dccc5bc2006fdeae400fc6b58998e727e17c
    source_path: providers/huggingface.md
    workflow: 16
---

[Hugging Face Inference Providers](https://huggingface.co/docs/inference-providers) एक टोकन के अंतर्गत कई होस्ट किए गए मॉडल (DeepSeek, Llama और अन्य) के लिए OpenAI-संगत चैट कम्प्लीशन्स राउटर उपलब्ध कराता है। OpenClaw केवल **चैट कम्प्लीशन्स एंडपॉइंट** से संचार करता है; टेक्स्ट-टू-इमेज, एम्बेडिंग या स्पीच के लिए सीधे [HF इन्फ़रेंस क्लाइंट](https://huggingface.co/docs/api-inference/quicktour) का उपयोग करें।

| गुण          | मान                                                                                                                         |
| ------------ | --------------------------------------------------------------------------------------------------------------------------- |
| प्रोवाइडर आईडी | `huggingface`                                                                                                         |
| Plugin       | बंडल किया गया (डिफ़ॉल्ट रूप से सक्षम, इंस्टॉल करने का कोई चरण नहीं)                                                         |
| प्रमाणीकरण एनवायरनमेंट वेरिएबल | `HUGGINGFACE_HUB_TOKEN` या `HF_TOKEN` (सूक्ष्म अनुमतियों वाला टोकन)                                  |
| API          | OpenAI-संगत (`https://router.huggingface.co/v1`)                                                                                            |
| बिलिंग       | एक HF टोकन; [मूल्य निर्धारण](https://huggingface.co/docs/inference-providers/pricing) मुफ़्त टियर के साथ प्रोवाइडर की दरों का अनुसरण करता है |

## आरंभ करना

<Steps>
  <Step title="सूक्ष्म अनुमतियों वाला टोकन बनाएँ">
    [Hugging Face Settings Tokens](https://huggingface.co/settings/tokens/new?ownUserPermissions=inference.serverless.write&tokenType=fineGrained) पर जाएँ और सूक्ष्म अनुमतियों वाला नया टोकन बनाएँ।

    <Warning>
    टोकन के लिए **Make calls to Inference Providers** अनुमति सक्षम होनी चाहिए, अन्यथा API अनुरोध अस्वीकार कर दिए जाएँगे।
    </Warning>

  </Step>
  <Step title="ऑनबोर्डिंग चलाएँ">
    प्रोवाइडर ड्रॉपडाउन में **Hugging Face** चुनें, फिर संकेत मिलने पर अपनी API कुंजी दर्ज करें:

    ```bash
    openclaw onboard --auth-choice huggingface-api-key
    ```

  </Step>
  <Step title="डिफ़ॉल्ट मॉडल चुनें">
    **Default Hugging Face model** ड्रॉपडाउन में कोई मॉडल चुनें। आपका टोकन मान्य होने पर सूची Inference API से लोड होती है; अन्यथा OpenClaw नीचे दिया गया अंतर्निहित कैटलॉग दिखाता है। आपका चयन `agents.defaults.model.primary` के रूप में सहेजा जाता है:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
        },
      },
    }
    ```

  </Step>
  <Step title="सत्यापित करें कि मॉडल उपलब्ध है">
    ```bash
    openclaw models list --provider huggingface
    ```
  </Step>
</Steps>

### गैर-इंटरैक्टिव सेटअप

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice huggingface-api-key \
  --huggingface-api-key "$HF_TOKEN"
```

`huggingface/deepseek-ai/DeepSeek-R1` को डिफ़ॉल्ट मॉडल के रूप में सेट करता है।

## मॉडल आईडी

मॉडल संदर्भ `huggingface/<org>/<model>` प्रारूप (Hub-शैली की आईडी) का उपयोग करते हैं। OpenClaw का अंतर्निहित कैटलॉग:

| मॉडल          | संदर्भ (`huggingface/` उपसर्ग के साथ) |
| ------------- | -------------------------------- |
| DeepSeek R1   | `deepseek-ai/DeepSeek-R1`        |
| DeepSeek V3.1 | `deepseek-ai/DeepSeek-V3.1`      |
| GPT-OSS 120B  | `openai/gpt-oss-120b`            |

<Tip>
आपका टोकन मान्य होने पर OpenClaw ऑनबोर्डिंग के समय और Gateway के आरंभ होने पर **GET** `https://router.huggingface.co/v1/models` से अन्य सभी मॉडल भी खोजता है, इसलिए आपके कैटलॉग में ऊपर दिए गए तीन मॉडलों से कहीं अधिक मॉडल शामिल हो सकते हैं। आप किसी भी मॉडल आईडी के अंत में `:fastest` या `:cheapest` जोड़ सकते हैं; HF का राउटर अनुरूप इन्फ़रेंस प्रोवाइडर तक रूट करता है। [Inference Provider settings](https://hf.co/settings/inference-providers) में अपना डिफ़ॉल्ट प्रोवाइडर क्रम सेट करें।
</Tip>

## उन्नत कॉन्फ़िगरेशन

<AccordionGroup>
  <Accordion title="मॉडल खोज और ऑनबोर्डिंग ड्रॉपडाउन">
    OpenClaw निम्न अनुरोध से मॉडल खोजता है:

    ```bash
    GET https://router.huggingface.co/v1/models
    Authorization: Bearer $HUGGINGFACE_HUB_TOKEN   # या $HF_TOKEN
    ```

    प्रतिक्रिया OpenAI-शैली की होती है: `{ "object": "list", "data": [ { "id": "Qwen/Qwen3-8B", "owned_by": "Qwen", ... }, ... ] }`।

    कॉन्फ़िगर की गई कुंजी (ऑनबोर्डिंग, `HUGGINGFACE_HUB_TOKEN` या `HF_TOKEN`) के साथ, इंटरैक्टिव सेटअप के दौरान **Default Hugging Face model** ड्रॉपडाउन में इस एंडपॉइंट से प्राप्त मॉडल भरे जाते हैं। Gateway आरंभ होने पर कैटलॉग को रीफ़्रेश करने के लिए वही कॉल दोहराई जाती है। खोजे गए मॉडलों को ऊपर दिए गए अंतर्निहित कैटलॉग के साथ मर्ज किया जाता है (आईडी मेल खाने पर कॉन्टेक्स्ट विंडो और लागत जैसे मेटाडेटा के लिए इसका उपयोग होता है)। यदि अनुरोध विफल होता है, कोई डेटा नहीं लौटाता या कोई कुंजी सेट नहीं है, तो OpenClaw केवल अंतर्निहित कैटलॉग का उपयोग करता है।

    प्रोवाइडर हटाए बिना खोज अक्षम करें:

    ```bash
    openclaw config set plugins.entries.huggingface.config.discovery.enabled false
    ```

  </Accordion>

  <Accordion title="मॉडल नाम, उपनाम और नीति प्रत्यय">
    - **API से नाम:** उपलब्ध होने पर खोजे गए मॉडल API के `name`, `title` या `display_name` का उपयोग करते हैं; अन्यथा OpenClaw मॉडल आईडी से नाम प्राप्त करता है (उदाहरण के लिए, `deepseek-ai/DeepSeek-R1` "DeepSeek R1" बन जाता है)।
    - **प्रदर्शित नाम बदलें:** कॉन्फ़िगरेशन में प्रत्येक मॉडल के लिए कस्टम लेबल सेट करें:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1 (fast)" },
            "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheap)" },
          },
        },
      },
    }
    ```

    - **नीति प्रत्यय:** `:fastest` और `:cheapest` HF राउटर के नियम हैं, इन्हें OpenClaw पुनर्लिखित नहीं करता: प्रत्यय को मॉडल आईडी के भाग के रूप में ज्यों का त्यों भेजा जाता है और HF का राउटर अनुरूप इन्फ़रेंस प्रोवाइडर चुनता है। यदि आप प्रत्येक प्रत्यय के लिए अलग उपनाम चाहते हैं, तो प्रत्येक वेरिएंट को `models.providers.huggingface.models` के अंतर्गत (या `model.primary` में) अलग प्रविष्टि के रूप में जोड़ें।
    - **कॉन्फ़िगरेशन मर्ज:** कॉन्फ़िगरेशन मर्ज के दौरान `models.providers.huggingface.models` की मौजूदा प्रविष्टियाँ (उदाहरण के लिए, `models.json` में) रखी जाती हैं, इसलिए वहाँ सेट किए गए कस्टम `name`, `alias` या मॉडल विकल्प पुनः आरंभ होने के बाद भी बने रहते हैं।

  </Accordion>

  <Accordion title="एनवायरनमेंट और डेमन सेटअप">
    यदि Gateway डेमन (launchd/systemd) के रूप में चलता है, तो सुनिश्चित करें कि उस प्रोसेस के लिए `HUGGINGFACE_HUB_TOKEN` या `HF_TOKEN` उपलब्ध हो (उदाहरण के लिए, `~/.openclaw/.env` में या `env.shellEnv` के माध्यम से)।

    <Note>
    OpenClaw, `HUGGINGFACE_HUB_TOKEN` और `HF_TOKEN` दोनों स्वीकार करता है। यदि दोनों सेट हैं, तो `HUGGINGFACE_HUB_TOKEN` को प्राथमिकता मिलती है।
    </Note>

  </Accordion>

  <Accordion title="कॉन्फ़िगरेशन: फ़ॉलबैक के साथ DeepSeek R1">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "huggingface/deepseek-ai/DeepSeek-R1",
            fallbacks: ["huggingface/openai/gpt-oss-120b"],
          },
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
            "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="कॉन्फ़िगरेशन: सबसे सस्ते और सबसे तेज़ वेरिएंट के साथ DeepSeek">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
            "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheapest)" },
            "huggingface/deepseek-ai/DeepSeek-R1:fastest": { alias: "DeepSeek R1 (fastest)" },
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="कॉन्फ़िगरेशन: उपनामों के साथ DeepSeek + GPT-OSS">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "huggingface/deepseek-ai/DeepSeek-V3.1",
            fallbacks: ["huggingface/openai/gpt-oss-120b"],
          },
          models: {
            "huggingface/deepseek-ai/DeepSeek-V3.1": { alias: "DeepSeek V3.1" },
            "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## संबंधित

<CardGroup cols={2}>
  <Card title="मॉडल चयन" href="/hi/concepts/model-providers" icon="layers">
    सभी प्रोवाइडर, मॉडल संदर्भ और फ़ेलओवर व्यवहार का अवलोकन।
  </Card>
  <Card title="मॉडल चयन" href="/hi/concepts/models" icon="brain">
    मॉडल चुनने और कॉन्फ़िगर करने का तरीका।
  </Card>
  <Card title="Inference Providers दस्तावेज़" href="https://huggingface.co/docs/inference-providers" icon="book">
    आधिकारिक Hugging Face Inference Providers दस्तावेज़।
  </Card>
  <Card title="कॉन्फ़िगरेशन" href="/hi/gateway/configuration" icon="gear">
    संपूर्ण कॉन्फ़िगरेशन संदर्भ।
  </Card>
</CardGroup>
