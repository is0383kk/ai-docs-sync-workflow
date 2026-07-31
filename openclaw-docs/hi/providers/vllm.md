---
read_when:
    - आप OpenClaw को स्थानीय vLLM सर्वर के साथ चलाना चाहते हैं
    - आप अपने मॉडलों के साथ OpenAI-संगत /v1 एंडपॉइंट चाहते हैं
summary: vLLM (OpenAI-संगत स्थानीय सर्वर) के साथ OpenClaw चलाएँ
title: vLLM
x-i18n:
    generated_at: "2026-07-27T18:32:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98d1044c0a82efb6c9937e961d765d0cfcea8664cbaa043168921b457756512c
    source_path: providers/vllm.md
    workflow: 16
---

vLLM एक **OpenAI-संगत** HTTP API के माध्यम से ओपन-सोर्स (और कुछ कस्टम) मॉडल उपलब्ध कराता है। जब आप `VLLM_API_KEY` के साथ ऑप्ट इन करते हैं, तो OpenClaw `openai-completions` API का उपयोग करके कनेक्ट होता है और मॉडलों को **स्वतः खोज** सकता है।

| प्रॉपर्टी         | मान                                      |
| ---------------- | ------------------------------------------ |
| प्रोवाइडर ID      | `vllm`                                     |
| API              | `openai-completions` (OpenAI-संगत)   |
| प्रमाणीकरण             | `VLLM_API_KEY` पर्यावरण चर        |
| डिफ़ॉल्ट बेस URL | `http://127.0.0.1:8000/v1`                 |
| स्ट्रीमिंग उपयोग  | समर्थित (`stream_options.include_usage`) |

## शुरुआत करना

<Steps>
  <Step title="OpenAI-संगत सर्वर के साथ vLLM शुरू करें">
    आपके बेस URL को `/v1` एंडपॉइंट (`/v1/models`, `/v1/chat/completions`) उपलब्ध कराने होंगे। vLLM सामान्यतः इस पर चलता है:

    ```text
    http://127.0.0.1:8000/v1
    ```

  </Step>
  <Step title="API कुंजी पर्यावरण चर सेट करें">
    यदि आपका सर्वर प्रमाणीकरण लागू नहीं करता, तो कोई भी गैर-रिक्त मान काम करेगा:

    ```bash
    export VLLM_API_KEY="vllm-local"
    ```

  </Step>
  <Step title="कोई मॉडल चुनें">
    इसे अपने किसी vLLM मॉडल ID से बदलें:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "vllm/your-model-id" },
        },
      },
    }
    ```

  </Step>
  <Step title="सत्यापित करें कि मॉडल उपलब्ध है">
    ```bash
    openclaw models list --provider vllm
    ```
  </Step>
</Steps>

<Tip>
गैर-इंटरैक्टिव सेटअप (CI, स्क्रिप्टिंग) के लिए बेस URL, कुंजी और मॉडल सीधे पास करें:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice vllm \
  --custom-base-url "http://127.0.0.1:8000/v1" \
  --custom-api-key "vllm-local" \
  --custom-model-id "your-model-id"
```

</Tip>

## मॉडल खोज (अंतर्निहित प्रोवाइडर)

जब `VLLM_API_KEY` सेट हो (या कोई प्रमाणीकरण प्रोफ़ाइल मौजूद हो) और `models.providers.vllm` परिभाषित **नहीं** हो, तो OpenClaw `GET http://127.0.0.1:8000/v1/models` को क्वेरी करता है और लौटाई गई ID को मॉडल प्रविष्टियों में बदलता है।

<Note>
यदि आप `models.providers.vllm` को स्पष्ट रूप से सेट करते हैं, तो OpenClaw केवल आपके घोषित मॉडलों का उपयोग करता है। OpenClaw से उस कॉन्फ़िगर किए गए प्रोवाइडर के `/models` एंडपॉइंट को भी क्वेरी कराने और सभी विज्ञापित vLLM मॉडल शामिल कराने के लिए `agents.defaults.models` में `"vllm/*": {}` जोड़ें।
</Note>

## स्पष्ट कॉन्फ़िगरेशन

जब vLLM किसी अलग होस्ट या पोर्ट पर चलता हो, आप `contextWindow`/`maxTokens` को निश्चित करना चाहते हों, आपके सर्वर को वास्तविक API कुंजी की आवश्यकता हो, या आप किसी विश्वसनीय लूपबैक, LAN अथवा Tailscale एंडपॉइंट से कनेक्ट करते हों, तब स्पष्ट रूप से कॉन्फ़िगर करें:

```json5
{
  models: {
    providers: {
      vllm: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "${VLLM_API_KEY}",
        api: "openai-completions",
        timeoutSeconds: 300, // वैकल्पिक: धीमे स्थानीय मॉडलों के लिए अनुरोध टाइमआउट बढ़ाएँ
        models: [
          {
            id: "your-model-id",
            name: "Local vLLM Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

हर मॉडल को सूचीबद्ध किए बिना प्रोवाइडर को डायनेमिक रखने के लिए दृश्यमान मॉडल कैटलॉग में वाइल्डकार्ड जोड़ें:

```json5
{
  agents: {
    defaults: {
      models: {
        "vllm/*": {},
      },
    },
  },
}
```

## उन्नत कॉन्फ़िगरेशन

<AccordionGroup>
  <Accordion title="प्रॉक्सी-शैली व्यवहार">
    vLLM को मूल OpenAI एंडपॉइंट के बजाय प्रॉक्सी-शैली के OpenAI-संगत `/v1` बैकएंड के रूप में माना जाता है:

    | व्यवहार                                | लागू है?                         |
    | --------------------------------------- | -------------------------------- |
    | मूल OpenAI अनुरोध आकार-निर्धारण           | नहीं                               |
    | `service_tier`                          | नहीं भेजा जाता                         |
    | Responses `store`                       | नहीं भेजा जाता                         |
    | प्रॉम्प्ट-कैश संकेत                      | नहीं भेजे जाते                         |
    | OpenAI रीजनिंग-संगत पेलोड आकार-निर्धारण | लागू नहीं किया जाता                      |
    | छिपे हुए OpenClaw एट्रिब्यूशन हेडर     | कस्टम बेस URL पर इंजेक्ट नहीं किए जाते |

  </Accordion>

  <Accordion title="Qwen थिंकिंग नियंत्रण">
    Qwen मॉडलों के लिए, जब सर्वर को Qwen चैट-टेम्पलेट kwargs अपेक्षित हों, तब मॉडल पंक्ति पर `compat.thinkingFormat: "qwen-chat-template"` सेट करें। ये मॉडल एक बाइनरी `/think` प्रोफ़ाइल (`off`, `on`) उपलब्ध कराते हैं, क्योंकि Qwen चैट-टेम्पलेट थिंकिंग एक चालू/बंद फ़्लैग है, OpenAI-शैली की प्रयास-सीढ़ी नहीं।

    ```json5
    {
      models: {
        providers: {
          vllm: {
            models: [
              {
                id: "Qwen/Qwen3-8B",
                name: "Qwen3 8B",
                reasoning: true,
                compat: { thinkingFormat: "qwen-chat-template" },
              },
            ],
          },
        },
      },
    }
    ```

    OpenClaw `/think off` को इसमें मैप करता है:

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "preserve_thinking": true
      }
    }
    ```

    गैर-`off` थिंकिंग स्तर `enable_thinking: true` भेजते हैं। यदि आपका एंडपॉइंट इसके बजाय DashScope-शैली के शीर्ष-स्तरीय फ़्लैग अपेक्षित करता है, तो अनुरोध रूट पर `enable_thinking` भेजने के लिए `compat.thinkingFormat: "qwen"` का उपयोग करें।

  </Accordion>

  <Accordion title="Nemotron 3 थिंकिंग नियंत्रण">
    थिंकिंग बंद वाले `vllm/nemotron-3-*` मॉडलों के लिए, बंडल किया गया Plugin यह भेजता है:

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "force_nonempty_content": true
      }
    }
    ```

    इन मानों को अनुकूलित करने के लिए मॉडल पैरामीटर के अंतर्गत `chat_template_kwargs` सेट करें। यदि आप `params.extra_body.chat_template_kwargs` भी सेट करते हैं, तो वह मान प्रभावी होगा, क्योंकि `extra_body` अनुरोध बॉडी का अंतिम ओवरराइड है।

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/nemotron-3-super": {
              params: {
                chat_template_kwargs: {
                  enable_thinking: false,
                  force_nonempty_content: true,
                },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Qwen टूल कॉल टेक्स्ट के रूप में दिखाई देते हैं">
    पहले पुष्टि करें कि vLLM को मॉडल के लिए सही टूल-कॉल पार्सर और चैट टेम्पलेट के साथ शुरू किया गया था। vLLM, Qwen2.5 मॉडलों के लिए `hermes` और Qwen3-Coder मॉडलों के लिए `qwen3_xml` का दस्तावेज़ीकरण करता है।

    लक्षण: Skills/टूल कभी नहीं चलते, सहायक `{"name":"read","arguments":...}` जैसा कच्चा JSON/XML प्रिंट करता है, या OpenClaw द्वारा `tool_choice: "auto"` भेजे जाने पर vLLM एक रिक्त `tool_calls` सरणी लौटाता है।

    कुछ Qwen/vLLM संयोजन केवल तब संरचित टूल कॉल लौटाते हैं, जब अनुरोध `tool_choice: "required"` का उपयोग करता है। इसे प्रत्येक मॉडल के लिए `params.extra_body` से बाध्य करें:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/Qwen-Qwen2.5-Coder-32B-Instruct": {
              params: {
                extra_body: {
                  tool_choice: "required",
                },
              },
            },
          },
        },
      },
    }
    ```

    मॉडल ID को `openclaw models list --provider vllm` से मिली सटीक ID से बदलें, या CLI से वही ओवरराइड लागू करें:

    ```bash
    openclaw config set agents.defaults.models '{"vllm/Qwen-Qwen2.5-Coder-32B-Instruct":{"params":{"extra_body":{"tool_choice":"required"}}}}' --strict-json --merge
    ```

    यह एक ऑप्ट-इन समाधान है: यह टूल वाले प्रत्येक टर्न को टूल कॉल करने के लिए बाध्य करता है, इसलिए इसका उपयोग केवल किसी समर्पित मॉडल प्रविष्टि के लिए करें, जहाँ यह स्वीकार्य हो। इसे सभी vLLM मॉडलों के लिए वैश्विक डिफ़ॉल्ट के रूप में सेट न करें और इसे ऐसे प्रॉक्सी के साथ युग्मित न करें, जो सहायक के मनमाने टेक्स्ट को निष्पादन योग्य टूल कॉल में बदलता हो।

  </Accordion>

  <Accordion title="कस्टम बेस URL">
    यदि आपका vLLM सर्वर किसी गैर-डिफ़ॉल्ट होस्ट या पोर्ट पर चलता है, तो स्पष्ट प्रोवाइडर कॉन्फ़िगरेशन में `baseUrl` सेट करें:

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:9000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [
              {
                id: "my-custom-model",
                name: "Remote vLLM Model",
                reasoning: false,
                input: ["text"],
                contextWindow: 64000,
                maxTokens: 4096,
              },
            ],
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## समस्या निवारण

<AccordionGroup>
  <Accordion title="धीमी पहली प्रतिक्रिया या रिमोट सर्वर टाइमआउट">
    बड़े स्थानीय मॉडलों, रिमोट LAN होस्ट या टेलनेट लिंक के लिए प्रोवाइडर-स्कोप वाला अनुरोध टाइमआउट सेट करें:

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:8000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [{ id: "your-model-id", name: "Local vLLM Model" }],
          },
        },
      },
    }
    ```

    `timeoutSeconds` केवल vLLM मॉडल HTTP अनुरोधों पर लागू होता है: कनेक्शन सेटअप, प्रतिक्रिया हेडर, बॉडी स्ट्रीमिंग और संपूर्ण संरक्षित-फ़ेच निरस्तीकरण। यह इस प्रोवाइडर के लिए LLM निष्क्रिय/स्ट्रीम वॉचडॉग सीमा को अंतर्निहित ~120s डिफ़ॉल्ट से ऊपर भी बढ़ाता है। `agents.defaults.timeoutSeconds` बढ़ाने के बजाय इसे प्राथमिकता दें, क्योंकि वह पूरे एजेंट रन को नियंत्रित करता है।

  </Accordion>

  <Accordion title="सर्वर तक नहीं पहुँचा जा सकता">
    जाँचें कि vLLM सर्वर चल रहा है और उस तक पहुँचा जा सकता है:

    ```bash
    curl http://127.0.0.1:8000/v1/models
    ```

    यदि आपको कनेक्शन त्रुटि दिखाई देती है, तो होस्ट, पोर्ट और यह सत्यापित करें कि vLLM OpenAI-संगत सर्वर मोड में शुरू हुआ था। OpenClaw लूपबैक, LAN और Tailscale एंडपॉइंट पर संरक्षित मॉडल अनुरोधों के लिए सटीक कॉन्फ़िगर किए गए `models.providers.vllm.baseUrl` ओरिजिन पर भरोसा करता है। स्पष्ट ऑप्ट-इन के बिना मेटाडेटा/लिंक-लोकल ओरिजिन अवरुद्ध रहते हैं। `models.providers.vllm.request.allowPrivateNetwork: true` केवल तभी सेट करें, जब vLLM अनुरोधों को किसी अन्य निजी ओरिजिन तक पहुँचना आवश्यक हो, या सटीक-ओरिजिन विश्वास से ऑप्ट आउट करने के लिए `false` सेट करें।

  </Accordion>

  <Accordion title="अनुरोधों पर प्रमाणीकरण त्रुटियाँ">
    यदि अनुरोध प्रमाणीकरण त्रुटियों के साथ विफल होते हैं, तो अपने सर्वर कॉन्फ़िगरेशन से मेल खाने वाला वास्तविक `VLLM_API_KEY` सेट करें, या `models.providers.vllm` के अंतर्गत प्रोवाइडर को स्पष्ट रूप से कॉन्फ़िगर करें।

    <Tip>
    यदि आपका vLLM सर्वर प्रमाणीकरण लागू नहीं करता, तो `VLLM_API_KEY` के लिए कोई भी गैर-रिक्त मान OpenClaw के लिए ऑप्ट-इन संकेत के रूप में काम करता है।
    </Tip>

  </Accordion>

  <Accordion title="कोई मॉडल नहीं मिला">
    स्वतः खोज के लिए `VLLM_API_KEY` का सेट होना आवश्यक है। यदि आपने `models.providers.vllm` परिभाषित किया है, तो OpenClaw केवल आपके घोषित मॉडलों का उपयोग करता है, जब तक कि `agents.defaults.models` में `"vllm/*": {}` शामिल न हो।
  </Accordion>

  <Accordion title="टूल कच्चे टेक्स्ट के रूप में रेंडर होते हैं">
    यदि कोई Qwen मॉडल किसी Skill को निष्पादित करने के बजाय JSON/XML टूल सिंटैक्स प्रिंट करता है:

    - उस मॉडल के लिए सही पार्सर/टेम्पलेट के साथ vLLM शुरू करें।
    - `openclaw models list --provider vllm` से सटीक मॉडल ID की पुष्टि करें।
    - केवल तभी एक समर्पित प्रति-मॉडल `params.extra_body.tool_choice: "required"` ओवरराइड जोड़ें, जब `tool_choice: "auto"` अब भी रिक्त या केवल-टेक्स्ट टूल कॉल लौटाता हो।

  </Accordion>
</AccordionGroup>

<Warning>
अधिक सहायता: [समस्या निवारण](/hi/help/troubleshooting) और [अक्सर पूछे जाने वाले प्रश्न](/hi/help/faq)।
</Warning>

## संबंधित

<CardGroup cols={2}>
  <Card title="मॉडल चयन" href="/hi/concepts/model-providers" icon="layers">
    प्रोवाइडर, मॉडल संदर्भ और फ़ेलओवर व्यवहार चुनना।
  </Card>
  <Card title="OpenAI" href="/hi/providers/openai" icon="bolt">
    मूल OpenAI प्रोवाइडर और OpenAI-संगत रूट व्यवहार।
  </Card>
  <Card title="OAuth और प्रमाणीकरण" href="/hi/gateway/authentication" icon="key">
    प्रमाणीकरण विवरण और क्रेडेंशियल पुनः उपयोग नियम।
  </Card>
  <Card title="समस्या निवारण" href="/hi/help/troubleshooting" icon="wrench">
    सामान्य समस्याएँ और उन्हें हल करने के तरीके।
  </Card>
</CardGroup>
