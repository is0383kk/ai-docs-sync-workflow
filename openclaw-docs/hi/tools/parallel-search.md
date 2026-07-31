---
read_when:
    - आप API कुंजी के बिना वेब खोज चाहते हैं
    - आप Parallel का सशुल्क Search API चाहते हैं
    - आप LLM संदर्भ दक्षता के लिए क्रमबद्ध सघन अंश चाहते हैं
summary: समानांतर खोज -- वेब स्रोतों से LLM-अनुकूलित सघन अंश
title: समानांतर खोज
x-i18n:
    generated_at: "2026-07-27T20:48:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eff693f286015b287bbdacf44f11ff6f07f2f7d2605ef6f09259e7402b40515e
    source_path: tools/parallel-search.md
    workflow: 16
---

Parallel plugin दो [Parallel](https://parallel.ai/) `web_search`
प्रदाता उपलब्ध कराता है, जो दोनों AI एजेंटों के लिए बनाए गए वेब इंडेक्स से रैंक किए गए,
LLM-अनुकूलित अंश लौटाते हैं:

| प्रदाता               | id              | प्रमाणीकरण                                                                                       |
| ---------------------- | --------------- | ------------------------------------------------------------------------------------------ |
| Parallel Search (निःशुल्क) | `parallel-free` | कोई नहीं -- Parallel का निःशुल्क [Search MCP](https://docs.parallel.ai/integrations/mcp/search-mcp) |
| Parallel Search        | `parallel`      | `PARALLEL_API_KEY` -- सशुल्क Search API, उच्च दर सीमाएँ और उद्देश्य अनुकूलन             |

किसी एक को स्पष्ट रूप से चुनने के लिए `tools.web.search.provider` को `parallel-free` या
`parallel` पर सेट करें; इनमें से किसी का भी स्वतः पता नहीं लगाया जाता।

<Note>
  प्रत्यक्ष OpenAI Responses मॉडल (`api: "openai-responses"`, प्रदाता
  `openai`, आधिकारिक API आधार URL), जब `tools.web.search.provider` अनसेट, रिक्त,
  `"auto"`, या `"openai"` हो, तो OpenAI की होस्ट की गई मूल वेब खोज
  का स्वचालित रूप से उपयोग करते हैं -- इसलिए वे डिफ़ॉल्ट रूप से Parallel को बायपास करते हैं। उन्हें
  इसके बजाय Parallel के माध्यम से रूट करने के लिए `tools.web.search.provider` को
  `parallel-free` या `parallel` पर सेट करें। [वेब खोज का अवलोकन](/hi/tools/web) देखें।
</Note>

## Plugin इंस्टॉल करें

```bash
openclaw plugins install @openclaw/parallel-plugin
openclaw gateway restart
```

## API कुंजी (सशुल्क प्रदाता)

`parallel-free` को किसी कुंजी की आवश्यकता नहीं है, लेकिन फिर भी इसे स्पष्ट रूप से चुना जाना चाहिए। सशुल्क
`parallel` प्रदाता को एक API कुंजी की आवश्यकता होती है:

<Steps>
  <Step title="खाता बनाएँ">
    [platform.parallel.ai](https://platform.parallel.ai) पर साइन अप करें और
    अपने डैशबोर्ड से एक API कुंजी जनरेट करें।
  </Step>
  <Step title="कुंजी संग्रहीत करें">
    Gateway परिवेश में `PARALLEL_API_KEY` सेट करें, या इसके माध्यम से कॉन्फ़िगर करें:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## कॉन्फ़िगरेशन

```json5
{
  plugins: {
    entries: {
      parallel: {
        config: {
          webSearch: {
            apiKey: "par-...", // यदि PARALLEL_API_KEY सेट है, तो वैकल्पिक
            baseUrl: "https://api.parallel.ai", // वैकल्पिक; OpenClaw /v1/search जोड़ता है
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        // निःशुल्क Search MCP के लिए "parallel-free", या यहाँ दिखाए गए
        // सशुल्क API-समर्थित प्रदाता के लिए "parallel"।
        provider: "parallel",
      },
    },
  },
}
```

**परिवेश विकल्प:** Gateway परिवेश में `PARALLEL_API_KEY` सेट करें।
Gateway इंस्टॉलेशन के लिए, इसे `~/.openclaw/.env` में रखें।

## बेस URL ओवरराइड

केवल सशुल्क `parallel` प्रदाता पर लागू होता है; `parallel-free` हमेशा
`https://search.parallel.ai/mcp` का उपयोग करता है और इस सेटिंग को अनदेखा करता है।

सशुल्क अनुरोधों को किसी संगत प्रॉक्सी या वैकल्पिक एंडपॉइंट (उदाहरण के लिए,
Cloudflare AI Gateway) से रूट करने के लिए `plugins.entries.parallel.config.webSearch.baseUrl` सेट करें। OpenClaw बिना
स्कीम वाले होस्ट के आगे `https://` जोड़कर उन्हें सामान्यीकृत करता है और यदि पथ
पहले से वहाँ समाप्त नहीं होता है, तो `/v1/search` जोड़ता है। हल किया गया एंडपॉइंट
खोज कैश कुंजी का हिस्सा होता है, इसलिए अलग-अलग एंडपॉइंट के परिणाम कभी साझा नहीं किए जाते।

## टूल पैरामीटर

दोनों प्रदाता Parallel के मूल खोज प्रारूप को उपलब्ध कराते हैं, ताकि मॉडल एक
स्वाभाविक भाषा वाला लक्ष्य और कुछ छोटी कीवर्ड क्वेरी भर सके -- यह वह संयोजन है
जिसकी सर्वोत्तम परिणामों के लिए Parallel
[अनुशंसा करता है](https://docs.parallel.ai/search/best-practices)।

<ParamField path="objective" type="string" required>
अंतर्निहित प्रश्न या लक्ष्य का स्वाभाविक भाषा में वर्णन (अधिकतम 5000
वर्ण)। यह अपने आप में पूर्ण होना चाहिए।
</ParamField>

<ParamField path="search_queries" type="string[]" required>
संक्षिप्त कीवर्ड खोज क्वेरी, प्रत्येक में 3-6 शब्द (1-5 प्रविष्टियाँ, प्रत्येक अधिकतम
200 वर्ण)। सर्वोत्तम परिणामों के लिए 2-3 विविध क्वेरी दें।
</ParamField>

<ParamField path="count" type="number">
लौटाए जाने वाले परिणामों की संख्या (1-40)।
</ParamField>

<ParamField path="session_id" type="string">
पिछले परिणाम के `sessionId` से प्राप्त वैकल्पिक Parallel सत्र आईडी। उसी कार्य में
आगे की खोजों के लिए इसे पास करें, ताकि Parallel संबंधित कॉल को समूहित कर सके और
बाद के परिणामों को बेहतर बना सके। `parallel` पर अधिकतम 1000 वर्ण; निःशुल्क
`parallel-free` Search MCP इसे 100 तक सीमित करता है। सीमा से अधिक लंबी आईडी को
हटा दिया जाता है (सशुल्क) या एक नई आईडी बनाई जाती है (निःशुल्क)।
</ParamField>

<ParamField path="client_model" type="string">
कॉल करने वाले मॉडल का वैकल्पिक पहचानकर्ता (जैसे `claude-opus-4-7`,
`gpt-5.6-sol`), अधिकतम 100 वर्ण। इससे Parallel आपके मॉडल की क्षमताओं के अनुसार
डिफ़ॉल्ट सेटिंग अनुकूलित कर सकता है। सक्रिय मॉडल का सटीक स्लग पास करें; इसे किसी
फ़ैमिली उपनाम में छोटा न करें।
</ParamField>

## टिप्पणियाँ

- Parallel परिणामों को मानव द्वारा क्लिक करके पढ़ने के बजाय LLM तर्क-क्षमता में उपयोगिता के लिए रैंक और संक्षिप्त करता है; पूर्ण-पृष्ठ
  सामग्री के बजाय प्रत्येक परिणाम में सघन अंश अपेक्षित हैं।
- परिणाम अंश `excerpts` ऐरे के रूप में लौटते हैं और सामान्य `web_search` अनुबंध के साथ
  संगतता के लिए उन्हें `description` में भी जोड़ा जाता है।
- दोनों प्रदाता एक `session_id` लौटाते हैं; OpenClaw इसे टूल पेलोड में `sessionId` के रूप में
  उपलब्ध कराता है, ताकि कॉलर आगे की खोजों को समूहित कर सकें। Parallel द्वारा
  जनरेट की गई सत्र आईडी (जो कॉलर ने नहीं दी थी) को कैश प्रविष्टि से बाहर रखा
  जाता है, क्योंकि समान क्वेरी वाले असंबंधित कार्यों को वह आईडी विरासत में
  नहीं मिलनी चाहिए।
- Parallel से प्राप्त `searchId`, `warnings`, और `usage` मौजूद होने पर
  बिना बदलाव के पास किए जाते हैं।
- OpenClaw हमेशा हल की गई परिणाम संख्या को Parallel को
  `advanced_settings.max_results` (`parallel`) के रूप में अग्रेषित करता है या Parallel की निश्चित आकार वाली प्रतिक्रिया
  (`parallel-free`) के बाद क्लाइंट पक्ष पर `count` लागू करता है। कॉलर का
  `count` तर्क प्राथमिकता लेता है, फिर `tools.web.search.maxResults`, अन्यथा
  OpenClaw का सामान्य `web_search` डिफ़ॉल्ट (5) -- Parallel के अपने API का डिफ़ॉल्ट
  10 है।
- परिणाम डिफ़ॉल्ट रूप से 15 मिनट के लिए कैश किए जाते हैं (`cacheTtlMinutes`)।
- जब कॉलर कोई आईडी नहीं देता, तो `parallel-free` अपने MCP हैंडशेक के माध्यम से प्रत्येक कॉल के लिए
  एक नया `session_id` बनाता है; ऐसी स्थिति में `parallel` इसे सेट नहीं करता।

## संबंधित

- [वेब खोज का अवलोकन](/hi/tools/web) -- सभी प्रदाता और स्वचालित पहचान
- [Exa खोज](/hi/tools/exa-search) -- सामग्री निष्कर्षण के साथ न्यूरल खोज
- [Perplexity खोज](/hi/tools/perplexity-search) -- डोमेन फ़िल्टरिंग के साथ संरचित परिणाम
