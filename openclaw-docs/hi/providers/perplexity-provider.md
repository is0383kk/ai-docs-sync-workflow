---
read_when:
    - आप Perplexity को वेब खोज प्रदाता के रूप में कॉन्फ़िगर करना चाहते हैं
    - आपको Perplexity API कुंजी या OpenRouter प्रॉक्सी सेटअप की आवश्यकता है
summary: Perplexity वेब खोज प्रदाता सेटअप (API कुंजी, खोज मोड, फ़िल्टरिंग)
title: Perplexity
x-i18n:
    generated_at: "2026-07-27T18:27:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea76a5cb7befce95756e9bcc8f9c1637fac87711d02d8a486ec2a1b9f51b73dc
    source_path: providers/perplexity-provider.md
    workflow: 16
---

Perplexity plugin दो ट्रांसपोर्ट के साथ एक `web_search` प्रदाता पंजीकृत करता है: नेटिव Perplexity Search API (फ़िल्टर सहित संरचित परिणाम) और Perplexity Sonar चैट कम्प्लीशन, सीधे या OpenRouter के माध्यम से (उद्धरणों सहित AI-संश्लेषित उत्तर)।

<Note>
यह पृष्ठ Perplexity **प्रदाता** के सेटअप के बारे में है। Perplexity **टूल** (एजेंट इसका उपयोग कैसे करता है) के लिए, [Perplexity खोज](/hi/tools/perplexity-search) देखें।
</Note>

| प्रॉपर्टी   | मान                                                                    |
| ----------- | ---------------------------------------------------------------------- |
| प्रकार      | वेब खोज प्रदाता (मॉडल प्रदाता नहीं)                                   |
| प्रमाणीकरण  | `PERPLEXITY_API_KEY` (नेटिव) या `OPENROUTER_API_KEY` (OpenRouter के माध्यम से) |
| कॉन्फ़िगरेशन पथ | `plugins.entries.perplexity.config.webSearch.apiKey`                   |
| ओवरराइड    | `plugins.entries.perplexity.config.webSearch.baseUrl` / `.model`       |
| कुंजी प्राप्त करें | [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api)   |

## Plugin इंस्टॉल करें

```bash
openclaw plugins install @openclaw/perplexity-plugin
openclaw gateway restart
```

## आरंभ करना

<Steps>
  <Step title="API कुंजी सेट करें">
    ```bash
    openclaw configure --section web
    ```

    या कुंजी सीधे सेट करें:

    ```bash
    openclaw config set plugins.entries.perplexity.config.webSearch.apiKey "pplx-xxxxxxxxxxxx"
    ```

    Gateway परिवेश में `PERPLEXITY_API_KEY` या `OPENROUTER_API_KEY` के रूप में एक्सपोर्ट की गई कुंजी भी काम करती है।

  </Step>
  <Step title="खोज शुरू करें">
    उपलब्ध खोज क्रेडेंशियल के रूप में Perplexity की कुंजी मौजूद होने पर `web_search` स्वतः Perplexity का पता लगा लेता है; किसी अतिरिक्त सेटअप की आवश्यकता नहीं है। प्रदाता को स्पष्ट रूप से निर्धारित करने के लिए:

    ```bash
    openclaw config set tools.web.search.provider perplexity
    ```

  </Step>
</Steps>

## खोज मोड

Plugin इस क्रम में ट्रांसपोर्ट निर्धारित करता है:

1. `webSearch.baseUrl` या `webSearch.model` सेट होने पर: कुंजी के प्रकार की परवाह किए बिना, हमेशा उस एंडपॉइंट पर Sonar चैट कम्प्लीशन के माध्यम से रूट करता है।
2. अन्यथा, कुंजी का स्रोत एंडपॉइंट निर्धारित करता है: कॉन्फ़िगर की गई कुंजी का प्रीफ़िक्स ट्रांसपोर्ट चुनता है (कॉन्फ़िगरेशन को परिवेश वेरिएबल पर प्राथमिकता मिलती है); परिवेश कुंजी सीधे अपने संगत एंडपॉइंट का उपयोग करती है।

| कुंजी प्रीफ़िक्स | ट्रांसपोर्ट                                                | विशेषताएँ                                         |
| ---------------- | ---------------------------------------------------------- | ------------------------------------------------- |
| `pplx-`    | नेटिव Perplexity Search API (`https://api.perplexity.ai`) | संरचित परिणाम, डोमेन/भाषा/तिथि फ़िल्टर           |
| `sk-or-`   | OpenRouter (`https://openrouter.ai/api/v1`), Sonar मॉडल   | उद्धरणों सहित AI-संश्लेषित उत्तर                  |

किसी अन्य प्रीफ़िक्स वाली कॉन्फ़िगर की गई कुंजी भी नेटिव Search API का उपयोग करती है। चैट-कम्प्लीशन पथ डिफ़ॉल्ट रूप से `perplexity/sonar-pro` मॉडल का उपयोग करता है; इसे `plugins.entries.perplexity.config.webSearch.model` से ओवरराइड करें।

## नेटिव API फ़िल्टरिंग

| फ़िल्टर                              | विवरण                                                           | ट्रांसपोर्ट       |
| ------------------------------------ | --------------------------------------------------------------- | ---------------- |
| `count`                              | प्रति खोज परिणाम, 1-10 (डिफ़ॉल्ट 5)                            | केवल नेटिव       |
| `freshness`                          | नवीनता अवधि: `day`, `week`, `month`, `year`                  | दोनों             |
| `country`                            | 2-अक्षर वाला देश कोड (`us`, `de`, `jp`)                        | केवल नेटिव       |
| `language`                           | ISO 639-1 भाषा कोड (`en`, `fr`, `zh`)                      | केवल नेटिव       |
| `date_after` / `date_before`         | `YYYY-MM-DD` में प्रकाशन-तिथि की सीमा                            | केवल नेटिव       |
| `domain_filter`                      | अधिकतम 20 डोमेन; अनुमति-सूची या `-`-प्रीफ़िक्स वाली निषेध-सूची, इन्हें कभी मिश्रित न करें | केवल नेटिव       |
| `max_tokens` / `max_tokens_per_page` | सभी परिणामों में कुल / प्रति पृष्ठ सामग्री सीमा                 | केवल नेटिव       |

केवल नेटिव फ़िल्टर चैट-कम्प्लीशन पथ पर एक वर्णनात्मक त्रुटि लौटाते हैं।
`freshness` को `date_after`/`date_before` के साथ संयोजित नहीं किया जा सकता।

## उन्नत कॉन्फ़िगरेशन

<AccordionGroup>
  <Accordion title="डेमन प्रक्रियाओं के लिए परिवेश वेरिएबल">
    <Warning>
    केवल इंटरैक्टिव शेल में एक्सपोर्ट की गई कुंजी launchd/systemd Gateway डेमन को दिखाई नहीं देती, जब तक कि उस परिवेश को स्पष्ट रूप से इंपोर्ट न किया जाए। कुंजी को `~/.openclaw/.env` में या `env.shellEnv` के माध्यम से सेट करें, ताकि Gateway प्रक्रिया उसे पढ़ सके। पूर्ण प्राथमिकता क्रम के लिए [परिवेश वेरिएबल](/hi/help/environment) देखें।
    </Warning>
  </Accordion>

  <Accordion title="OpenRouter प्रॉक्सी सेटअप">
    Perplexity खोजों को OpenRouter के माध्यम से रूट करने के लिए, नेटिव Perplexity कुंजी के बजाय एक `OPENROUTER_API_KEY` (प्रीफ़िक्स `sk-or-`) सेट करें। OpenClaw कुंजी का पता लगाता है और स्वतः Sonar ट्रांसपोर्ट पर स्विच हो जाता है। यह तब उपयोगी है जब आपने पहले ही OpenRouter बिलिंग सेट कर रखी हो और वहाँ प्रदाताओं को समेकित करना चाहते हों।
  </Accordion>
</AccordionGroup>

## संबंधित

<CardGroup cols={2}>
  <Card title="Perplexity खोज टूल" href="/hi/tools/perplexity-search" icon="magnifying-glass">
    एजेंट Perplexity खोजों को कैसे शुरू करता है और परिणामों की व्याख्या कैसे करता है।
  </Card>
  <Card title="कॉन्फ़िगरेशन संदर्भ" href="/hi/gateway/configuration-reference" icon="gear">
    Plugin प्रविष्टियों सहित पूर्ण कॉन्फ़िगरेशन संदर्भ।
  </Card>
</CardGroup>
