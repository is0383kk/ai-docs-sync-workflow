---
read_when:
    - आप किसी URL से सामग्री प्राप्त करके पठनीय सामग्री निकालना चाहते हैं
    - आपको web_fetch या उसके Firecrawl फ़ॉलबैक को कॉन्फ़िगर करना होगा
    - आप web_fetch की सीमाओं और कैशिंग को समझना चाहते हैं
sidebarTitle: Web Fetch
summary: web_fetch टूल -- पठनीय सामग्री निष्कर्षण के साथ HTTP फ़ेच
title: वेब से प्राप्ति
x-i18n:
    generated_at: "2026-07-27T18:48:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf312245064672dcf489e8714740fa3e034827e16b33be8fb6a87db04f19ef8
    source_path: tools/web-fetch.md
    workflow: 16
---

`web_fetch` सामान्य HTTP GET करता है और पठनीय सामग्री (HTML से
markdown या टेक्स्ट) निकालता है। यह JavaScript निष्पादित **नहीं** करता। JS-प्रधान साइटों या
लॉगिन-संरक्षित पृष्ठों के लिए, इसके बजाय [वेब ब्राउज़र](/hi/tools/browser) का उपयोग करें।

## त्वरित शुरुआत

डिफ़ॉल्ट रूप से सक्षम, किसी कॉन्फ़िगरेशन की आवश्यकता नहीं:

```javascript
await web_fetch({ url: "https://example.com/article" });
```

## टूल पैरामीटर

<ParamField path="url" type="string" required>
प्राप्त किया जाने वाला URL। केवल `http(s)`।
</ParamField>

<ParamField path="extractMode" type="'markdown' | 'text'" default="markdown">
मुख्य सामग्री निकालने के बाद आउटपुट प्रारूप।
</ParamField>

<ParamField path="maxChars" type="number">
आउटपुट को इतने वर्णों तक काटें। `tools.web.fetch.maxCharsCap` तक सीमित।
</ParamField>

## परिणाम

`web_fetch` इन फ़ील्ड वाला एक बंद संरचित परिणाम लौटाता है:

- अनुरोध मेटाडेटा: `url`, `finalUrl`, `status`, `extractMode`, और `extractor`
- वैकल्पिक प्रतिक्रिया मेटाडेटा: `contentType`, `title`, और `warning` (अनुपस्थित होने पर छोड़ा जाता है)
- रैप की गई सामग्री का मेटाडेटा: `externalContent`, `truncated`, `length`, `rawLength`,
  `fetchedAt`, `tookMs`, और `text`
- कैश हिट पर वैकल्पिक `cached: true`
- जब काटी गई सामग्री किसी निजी अस्थायी फ़ाइल में लिखी गई हो, तब वैकल्पिक `spill: { path, chars, truncated? }`;
  `truncated` केवल तभी मौजूद होता है जब उस फ़ाइल में
  आंशिक स्रोत सामग्री हो

`length` रैप किए गए `text` की लंबाई है। `rawLength` बाहरी-सामग्री रैपिंग से
पहले निकाली गई सामग्री की लंबाई है।

## यह कैसे काम करता है

<Steps>
  <Step title="प्राप्त करें">
    Chrome-जैसे User-Agent और `Accept-Language`
    हेडर के साथ HTTP GET भेजता है। निजी/आंतरिक होस्टनाम अवरुद्ध करता है और रीडायरेक्ट की दोबारा जाँच करता है।
  </Step>
  <Step title="निकालें">
    HTML प्रतिक्रिया पर Readability (मुख्य-सामग्री निष्कर्षण) चलाता है।
  </Step>
  <Step title="फ़ॉलबैक (वैकल्पिक)">
    यदि Readability विफल हो और कोई फ़ेच प्रदाता उपलब्ध हो, तो उस प्रदाता के माध्यम से
    फिर प्रयास करता है (उदाहरण के लिए Firecrawl का बॉट-परिहार मोड)।
  </Step>
  <Step title="कैश">
    समान URL को बार-बार प्राप्त करने की संख्या घटाने के लिए परिणाम 15 मिनट
    (कॉन्फ़िगर करने योग्य) तक कैश किए जाते हैं।
  </Step>
</Steps>

## प्रगति अपडेट

`web_fetch` केवल तभी सार्वजनिक प्रगति पंक्ति उत्सर्जित करता है जब पाँच सेकंड बाद भी फ़ेच
लंबित हो:

```text
पृष्ठ की सामग्री प्राप्त की जा रही है...
```

तेज़ कैश हिट और त्वरित नेटवर्क प्रतिक्रियाएँ टाइमर सक्रिय होने से पहले पूरी हो जाती हैं, इसलिए
वे कभी प्रगति पंक्ति नहीं दिखातीं। कॉल रद्द करने पर टाइमर साफ़ हो जाता है। प्रगति
पंक्ति केवल चैनल UI स्थिति है और इसमें प्राप्त पृष्ठ की सामग्री कभी नहीं होती।

## कॉन्फ़िगरेशन

```json5
{
  tools: {
    web: {
      fetch: {
        enabled: true, // डिफ़ॉल्ट: true
        provider: "firecrawl", // वैकल्पिक; स्वतः-पहचान के लिए छोड़ दें
        maxChars: 20000, // डिफ़ॉल्ट आउटपुट वर्ण; maxCharsCap द्वारा सीमित
        maxCharsCap: 20000, // maxChars पैरामीटर की कठोर सीमा
        maxResponseBytes: 750000, // काटने से पहले अधिकतम डाउनलोड आकार (32000-10000000)
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        useTrustedEnvProxy: false, // विश्वसनीय HTTP(S) एनवायरनमेंट प्रॉक्सी को DNS रिज़ॉल्व करने दें
        readability: true, // Readability निष्कर्षण का उपयोग करें
        userAgent: "Mozilla/5.0 ...", // User-Agent ओवरराइड करें
        ssrfPolicy: {
          allowRfc2544BenchmarkRange: true, // 198.18.0.0/15 का उपयोग करने वाले विश्वसनीय नकली-IP प्रॉक्सी के लिए ऑप्ट-इन
          allowIpv6UniqueLocalRange: true, // fc00::/7 का उपयोग करने वाले विश्वसनीय नकली-IP प्रॉक्सी के लिए ऑप्ट-इन
        },
      },
    },
  },
}
```

## Firecrawl फ़ॉलबैक

यदि Readability निष्कर्षण विफल हो जाता है, तो `web_fetch` बॉट-परिहार और बेहतर निष्कर्षण के लिए
[Firecrawl](/hi/tools/firecrawl) पर फ़ॉलबैक कर सकता है:

```json5
{
  tools: {
    web: {
      fetch: {
        provider: "firecrawl", // वैकल्पिक; उपलब्ध क्रेडेंशियल से स्वतः-पहचान के लिए छोड़ दें
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            // apiKey: "fc-...", // वैकल्पिक; बिना कुंजी वाले शुरुआती एक्सेस के लिए छोड़ दें
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 172800000, // कैश अवधि (2 दिन)
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

`plugins.entries.firecrawl.config.webFetch.apiKey` वैकल्पिक है और SecretRef ऑब्जेक्ट का समर्थन करता है।
पुराना `tools.web.fetch.firecrawl.*` कॉन्फ़िगरेशन `openclaw doctor --fix` के माध्यम से
स्वतः `plugins.entries.firecrawl.config.webFetch` में माइग्रेट हो जाता है।

<Note>
  यदि आप Firecrawl API-कुंजी SecretRef कॉन्फ़िगर करते हैं और वह किसी
  `FIRECRAWL_API_KEY` एनवायरनमेंट फ़ॉलबैक के बिना अनरिज़ॉल्व्ड है, तो Gateway स्टार्टअप तुरंत विफल हो जाता है।
</Note>

<Note>
  Firecrawl `baseUrl` ओवरराइड कड़े प्रतिबंधित हैं: होस्ट किया गया ट्रैफ़िक
  `https://api.firecrawl.dev` का उपयोग करता है; स्वयं होस्ट किए गए ओवरराइड को निजी या
  आंतरिक एंडपॉइंट लक्षित करने होंगे, और `http://` केवल उन्हीं निजी लक्ष्यों के लिए स्वीकार किया जाता है।
</Note>

वर्तमान रनटाइम व्यवहार:

- `tools.web.fetch.provider` फ़ेच फ़ॉलबैक प्रदाता को स्पष्ट रूप से चुनता है।
- यदि `provider` छोड़ा गया है, तो OpenClaw कॉन्फ़िगर किए गए क्रेडेंशियल से पहले तैयार वेब-फ़ेच
  प्रदाता का स्वतः पता लगाता है। गैर-सैंडबॉक्स्ड `web_fetch` ऐसे इंस्टॉल किए गए plugins का उपयोग कर सकता है
  जो `contracts.webFetchProviders` घोषित करते हैं और रनटाइम पर
  मेल खाने वाला प्रदाता पंजीकृत करते हैं। आधिकारिक Firecrawl plugin वर्तमान में यह
  फ़ॉलबैक प्रदान करता है।
- सैंडबॉक्स्ड `web_fetch` कॉल बंडल किए गए प्रदाताओं के साथ उन इंस्टॉल किए गए प्रदाताओं की अनुमति देते हैं
  जिनकी आधिकारिक npm या ClawHub उत्पत्ति सत्यापित है। वर्तमान में इससे
  आधिकारिक Firecrawl plugin की अनुमति मिलती है; तृतीय-पक्ष बाहरी फ़ेच plugins बाहर रहते हैं।
- यदि Readability अक्षम है, तो `web_fetch` सीधे चयनित
  प्रदाता फ़ॉलबैक पर जाता है। यदि कोई प्रदाता उपलब्ध नहीं है, तो यह बंद स्थिति में विफल होता है।

## विश्वसनीय एनवायरनमेंट प्रॉक्सी

यदि आपके डिप्लॉयमेंट में `web_fetch` को किसी विश्वसनीय आउटबाउंड
HTTP(S) प्रॉक्सी से होकर जाना आवश्यक है, तो `tools.web.fetch.useTrustedEnvProxy: true` सेट करें।

इस मोड में, OpenClaw अनुरोध भेजने से पहले होस्टनाम-आधारित SSRF जाँच अब भी लागू करता है,
लेकिन स्थानीय DNS पिनिंग करने के बजाय प्रॉक्सी को DNS रिज़ॉल्व करने देता है।
इसे केवल तभी सक्षम करें जब प्रॉक्सी ऑपरेटर-नियंत्रित हो और DNS रिज़ॉल्यूशन के बाद
आउटबाउंड नीति लागू करता हो।

<Note>
  यदि कोई HTTP(S) प्रॉक्सी एनवायरनमेंट वेरिएबल कॉन्फ़िगर नहीं है, या लक्ष्य होस्ट
  `NO_PROXY` द्वारा बाहर रखा गया है, तो `web_fetch` स्थानीय DNS
  पिनिंग वाले सामान्य सख्त पथ पर फ़ॉलबैक करता है।
</Note>

## सीमाएँ और सुरक्षा

- `maxChars` को `tools.web.fetch.maxCharsCap` (डिफ़ॉल्ट `20000`) तक सीमित किया जाता है
- प्रतिक्रिया बॉडी को पार्सिंग से पहले `maxResponseBytes` (डिफ़ॉल्ट `750000`, 32000-10000000 तक सीमित)
  पर सीमित किया जाता है; अत्यधिक बड़ी प्रतिक्रियाएँ चेतावनी के साथ काट दी जाती हैं
- निजी/आंतरिक होस्टनाम अवरुद्ध किए जाते हैं
- `tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange` और
  `tools.web.fetch.ssrfPolicy.allowIpv6UniqueLocalRange` विश्वसनीय नकली-IP प्रॉक्सी स्टैक के लिए सीमित ऑप्ट-इन हैं;
  इन्हें तब तक सेट न करें जब तक आपका प्रॉक्सी उन कृत्रिम रेंज का स्वामी न हो और
  अपनी गंतव्य नीति लागू न करता हो
- रीडायरेक्ट की जाँच की जाती है और उन्हें `maxRedirects` (डिफ़ॉल्ट `3`) द्वारा सीमित किया जाता है
- `useTrustedEnvProxy` एक स्पष्ट ऑप्ट-इन है और इसे केवल
  ऑपरेटर-नियंत्रित प्रॉक्सी के लिए सक्षम किया जाना चाहिए जो DNS
  रिज़ॉल्यूशन के बाद भी आउटबाउंड नीति लागू करते हैं
- `web_fetch` सर्वोत्तम-प्रयास है -- कुछ साइटों को [वेब ब्राउज़र](/hi/tools/browser) की आवश्यकता होती है

## टूल प्रोफ़ाइल

यदि आप टूल प्रोफ़ाइल या अनुमतिसूचियों का उपयोग करते हैं, तो `web_fetch` या `group:web` जोड़ें:

```json5
{
  tools: {
    allow: ["web_fetch"],
    // या: allow: ["group:web"]  (इसमें web_fetch, web_search, और x_search शामिल हैं)
  },
}
```

## संबंधित

- [वेब खोज](/hi/tools/web) -- कई प्रदाताओं के साथ वेब पर खोजें
- [वेब ब्राउज़र](/hi/tools/browser) -- JS-प्रधान साइटों के लिए पूर्ण ब्राउज़र स्वचालन
- [Firecrawl](/hi/tools/firecrawl) -- Firecrawl खोज और स्क्रैप टूल
