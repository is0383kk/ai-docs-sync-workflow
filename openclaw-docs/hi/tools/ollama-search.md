---
read_when:
    - आप web_search के लिए Ollama का उपयोग करना चाहते हैं
    - आप एक कुंजी-रहित web_search प्रदाता चाहते हैं
    - आप `OLLAMA_API_KEY` के साथ होस्ट की गई Ollama वेब खोज का उपयोग करना चाहते हैं
    - आपको Ollama Web Search सेटअप संबंधी मार्गदर्शन चाहिए
summary: स्थानीय Ollama होस्ट या होस्टेड Ollama API के माध्यम से Ollama वेब खोज
title: Ollama वेब खोज
x-i18n:
    generated_at: "2026-07-27T20:09:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: edbbd887841339ab4c0c62ab7682a22fe99434a788957a91989fce6942187e9a
    source_path: tools/ollama-search.md
    workflow: 16
---

OpenClaw, Ollama के वेब-सर्च API से शीर्षक, URL और स्निपेट लौटाने वाले, बंडल किए गए `web_search` प्रदाता के रूप में **Ollama Web Search** का समर्थन करता है।

स्थानीय/स्वयं-होस्ट किए गए Ollama को डिफ़ॉल्ट रूप से किसी API कुंजी की आवश्यकता नहीं होती; इसके लिए पहुँच योग्य Ollama होस्ट और `ollama signin` आवश्यक हैं। सीधे होस्ट की गई खोज (स्थानीय Ollama के बिना) के लिए `baseUrl: "https://ollama.com"` और वास्तविक `OLLAMA_API_KEY` आवश्यक हैं।

## सेटअप

<Steps>
  <Step title="Ollama शुरू करें">
    सुनिश्चित करें कि Ollama इंस्टॉल है और चल रहा है।
  </Step>
  <Step title="साइन इन करें">
    ```bash
    ollama signin
    ```
  </Step>
  <Step title="Ollama Web Search चुनें">
    ```bash
    openclaw configure --section web
    ```

    प्रदाता के रूप में **Ollama Web Search** चुनें।

  </Step>
</Steps>

यदि आप मॉडल के लिए पहले से Ollama का उपयोग करते हैं, तो Ollama Web Search उसी कॉन्फ़िगर किए गए होस्ट का पुनः उपयोग करता है।

<Note>
  OpenClaw कभी भी उच्च-प्राथमिकता वाले क्रेडेंशियल-युक्त प्रदाता के स्थान पर Ollama Web Search को स्वतः नहीं चुनता; आपको इसे `tools.web.search.provider: "ollama"` के साथ स्पष्ट रूप से चुनना होगा।
</Note>

## कॉन्फ़िगरेशन

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

वैकल्पिक होस्ट ओवरराइड, केवल वेब खोज के दायरे में:

```json5
{
  plugins: {
    entries: {
      ollama: {
        config: {
          webSearch: {
            baseUrl: "http://ollama-host:11434",
          },
        },
      },
    },
  },
}
```

या Ollama मॉडल प्रदाता के लिए पहले से कॉन्फ़िगर किए गए होस्ट का पुनः उपयोग करें:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
      },
    },
  },
}
```

`models.providers.ollama.baseUrl` मानक कुंजी है; OpenAI SDK-शैली के कॉन्फ़िगरेशन उदाहरणों के साथ संगतता के लिए वेब-खोज प्रदाता वहाँ `baseURL` भी स्वीकार करता है। यदि कुछ भी सेट नहीं है, तो OpenClaw डिफ़ॉल्ट रूप से `http://127.0.0.1:11434` का उपयोग करता है।

सीधे होस्ट की गई Ollama Web Search (स्थानीय Ollama के बिना):

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "https://ollama.com",
        apiKey: "OLLAMA_API_KEY",
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

## प्रमाणीकरण और अनुरोध रूटिंग

- वेब-खोज-विशिष्ट API कुंजी फ़ील्ड मौजूद नहीं है; कॉन्फ़िगर किया गया होस्ट प्रमाणीकरण-सुरक्षित होने पर प्रदाता `models.providers.ollama.apiKey` (या उससे मेल खाने वाले परिवेश-समर्थित प्रदाता प्रमाणीकरण) का पुनः उपयोग करता है।
- होस्ट समाधान क्रम: `plugins.entries.ollama.config.webSearch.baseUrl` →
  `models.providers.ollama.baseUrl` (या `baseURL`) → `http://127.0.0.1:11434`।
- यदि समाधान किया गया होस्ट `https://ollama.com` है, तो OpenClaw API कुंजी को बियरर प्रमाणीकरण के रूप में उपयोग करके सीधे `https://ollama.com/api/web_search` को कॉल करता है।
- अन्यथा OpenClaw पहले स्थानीय प्रॉक्सी एंडपॉइंट `/api/experimental/web_search` को कॉल करता है (जो हस्ताक्षर करके अनुरोध को Ollama Cloud पर अग्रेषित करता है), फिर उसी होस्ट पर `/api/web_search` पर फ़ॉलबैक करता है। यदि दोनों विफल होते हैं और `OLLAMA_API_KEY` सेट है, तो वह उस कुंजी के साथ `https://ollama.com/api/web_search` पर एक बार पुनः प्रयास करता है—उसे स्थानीय होस्ट पर भेजे बिना।
- यदि Ollama पहुँच योग्य नहीं है या उसमें साइन इन नहीं किया गया है, तो OpenClaw सेटअप के दौरान चेतावनी देता है, लेकिन प्रदाता चुनने से नहीं रोकता।

## संबंधित

- [वेब खोज का अवलोकन](/hi/tools/web) -- सभी प्रदाता और स्वतः-पहचान
- [Ollama](/hi/providers/ollama) -- Ollama मॉडल सेटअप और क्लाउड/स्थानीय मोड
