---
read_when:
    - आपको OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED चेतावनी दिखाई देती है
    - आपको OPENCLAW_EXTENSION_API_DEPRECATED चेतावनी दिखाई देती है
    - आपने OpenClaw 2026.4.25 से पहले api.registerEmbeddedExtensionFactory का उपयोग किया था
    - आप एक Plugin को आधुनिक Plugin आर्किटेक्चर के अनुरूप अपडेट कर रहे हैं
    - आप एक बाहरी OpenClaw Plugin का रखरखाव करते हैं
sidebarTitle: Migrate to SDK
summary: पुरानी पश्चगामी-संगतता परत से आधुनिक Plugin SDK पर माइग्रेट करें
title: Plugin SDK माइग्रेशन
x-i18n:
    generated_at: "2026-07-27T18:21:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a483f9c0f8409505fc2688872995382944e002520ceb651214dbc5ad8e3554fb
    source_path: plugins/sdk-migration.md
    workflow: 16
---

OpenClaw ने व्यापक पश्च-संगतता परत को छोटे, केंद्रित इम्पोर्ट से निर्मित आधुनिक Plugin
आर्किटेक्चर से बदल दिया। यदि आपका Plugin उस
बदलाव से पहले का है, तो यह गाइड उसे वर्तमान अनुबंधों पर लाने में सहायता करती है।

## क्या बदला

पहले कई अत्यधिक खुले इम्पोर्ट सरफ़ेस Plugin को एक ही एंट्री पॉइंट से
लगभग किसी भी चीज़ तक पहुँचने देते थे:

- **`openclaw/plugin-sdk`** और **`openclaw/plugin-sdk/compat`** - केंद्रित SDK बनाए जाने के दौरान
  दर्जनों हेल्पर को पुनः एक्सपोर्ट करते थे। अब दोनों रूट
  हटा दिए गए हैं; इसके बजाय दस्तावेज़ीकृत सबपाथ इम्पोर्ट करें।
- **`openclaw/plugin-sdk/infra-runtime`** - एक व्यापक बैरल, जिसमें सिस्टम
  इवेंट, Heartbeat स्थिति, डिलीवरी क्यू, फ़ेच/प्रॉक्सी हेल्पर, फ़ाइल हेल्पर,
  अनुमोदन प्रकार और असंबंधित उपयोगिताएँ मिश्रित थीं।
- **`openclaw/plugin-sdk/config-runtime`** - एक व्यापक कॉन्फ़िग बैरल, जिसे
  केवल इसकी बाद की संगतता अवधि के लिए बनाए रखा गया था; सीधे रनटाइम लोड/राइट हेल्पर
  हटा दिए गए हैं।
- **`openclaw/extension-api`** - हटाया गया एक ब्रिज, जो Plugin को एम्बेडेड एजेंट रनर जैसे
  होस्ट-साइड हेल्पर तक सीधी पहुँच देता था।
- **`api.registerEmbeddedExtensionFactory(...)`** - हटाया गया केवल-एम्बेडेड-रनर
  हुक, जो `tool_result` जैसे एम्बेडेड-रनर इवेंट देखता था। इसके बजाय एजेंट
  टूल-रिज़ल्ट मिडलवेयर का उपयोग करें ([एम्बेडेड टूल-रिज़ल्ट एक्सटेंशन को
  मिडलवेयर में माइग्रेट करें](#how-to-migrate) देखें)।

रूट SDK, कॉम्पैट बैरल, एक्सटेंशन ब्रिज और एम्बेडेड एक्सटेंशन फ़ैक्टरी
हटा दिए गए हैं। `infra-runtime` और `config-runtime` केवल अपनी
अलग से दर्ज बाद की अवधियों के लिए शेष हैं; नए Plugin को केंद्रित सबपाथ का उपयोग करना चाहिए।

<Warning>
  हटाए गए रूट, कॉम्पैट या एक्सटेंशन सरफ़ेस इम्पोर्ट करने वाले Plugin अब
  लोड नहीं होते। अपग्रेड करने से पहले नीचे दिए गए मैपिंग का पालन करें।
</Warning>

OpenClaw किसी प्रतिस्थापन को प्रस्तुत करने वाले उसी बदलाव में दस्तावेज़ीकृत
Plugin व्यवहार को हटाता या उसकी पुनर्व्याख्या नहीं करता। अनुबंध तोड़ने वाले बदलाव पहले
एक संगतता अडैप्टर, निदान, दस्तावेज़ और बहिष्करण अवधि से होकर गुजरते हैं। यह
SDK इम्पोर्ट, मैनिफ़ेस्ट फ़ील्ड, सेटअप API, हुक और रनटाइम
पंजीकरण व्यवहार पर लागू होता है।

### क्यों

- **धीमा स्टार्टअप** - एक हेल्पर इम्पोर्ट करने पर दर्जनों असंबंधित मॉड्यूल लोड हो जाते थे।
- **चक्रीय निर्भरताएँ** - व्यापक पुनः एक्सपोर्ट के कारण इम्पोर्ट चक्र बनाना
  आसान था।
- **अस्पष्ट API सरफ़ेस** - स्थिर एक्सपोर्ट को आंतरिक एक्सपोर्ट से अलग पहचानने का कोई तरीका नहीं था।

अब प्रत्येक `openclaw/plugin-sdk/<subpath>` दस्तावेज़ीकृत अनुबंध वाला एक छोटा,
स्व-निहित मॉड्यूल है।

बंडल किए गए चैनलों के लिए पुराने प्रोवाइडर सुविधा सीम भी हट गए हैं -
चैनल-ब्रांडेड हेल्पर शॉर्टकट निजी मोनो-रेपो सुविधाएँ थे, स्थिर
Plugin अनुबंध नहीं। इसके बजाय संकीर्ण सामान्य SDK सबपाथ का उपयोग करें। बंडल किए गए
Plugin वर्कस्पेस के भीतर, प्रोवाइडर-स्वामित्व वाले हेल्पर को उसी Plugin के
`api.ts` या `runtime-api.ts` में रखें:

- Anthropic Claude-विशिष्ट स्ट्रीम हेल्पर को अपने `api.ts` /
  `contract-api.ts` सीम में रखता है।
- OpenAI प्रोवाइडर बिल्डर, डिफ़ॉल्ट-मॉडल हेल्पर और रियलटाइम प्रोवाइडर
  बिल्डर को अपने `api.ts` में रखता है।
- OpenRouter प्रोवाइडर बिल्डर और ऑनबोर्डिंग/कॉन्फ़िग हेल्पर को अपने
  `api.ts` में रखता है।

## संगतता नीति

बाहरी-Plugin संगतता कार्य इस क्रम का पालन करता है:

1. नया अनुबंध जोड़ें।
2. पुराने व्यवहार को संगतता अडैप्टर के माध्यम से जोड़े रखें।
3. पुराने पाथ और उसके प्रतिस्थापन का नाम बताने वाला निदान या चेतावनी जारी करें।
4. परीक्षणों में दोनों पाथ को कवर करें।
5. बहिष्करण और माइग्रेशन पाथ का दस्तावेज़ीकरण करें।
6. घोषित माइग्रेशन अवधि के बाद ही हटाएँ, सामान्यतः किसी प्रमुख
   रिलीज़ में।

यदि कोई मैनिफ़ेस्ट फ़ील्ड अभी भी स्वीकार की जाती है, तो दस्तावेज़ और
निदान द्वारा अन्यथा बताए जाने तक उसका उपयोग जारी रखें। नए कोड को दस्तावेज़ीकृत प्रतिस्थापन को प्राथमिकता देनी चाहिए;
सामान्य लघु रिलीज़ के दौरान मौजूदा Plugin नहीं टूटने चाहिए।

### प्रकाशित चैनल सेटअप संगतता

`2026.7.1` के माध्यम से प्रकाशित Slack, Discord, Signal और Microsoft Teams पैकेज
`openclaw/plugin-sdk/bundled-channel-config-schema` से चैनल-विशिष्ट कॉन्फ़िग स्कीमा इम्पोर्ट करते हैं।
प्रकाशित Slack और Discord पैकेज
`openclaw/plugin-sdk/setup-runtime` से `createLegacyCompatChannelDmPolicy` और
`promptLegacyChannelAllowFromForAccount` भी इम्पोर्ट करते हैं।

वे एक्सपोर्ट बहिष्कृत रनटाइम संगतता अडैप्टर के रूप में उपलब्ध रहते हैं।
नए और पुनः प्रकाशित Plugin को `channel-config-schema` और
`setup-runtime` के सामान्य प्रिमिटिव का उपयोग करके अपने कॉन्फ़िग स्कीमा और सेटअप नीति का
स्वामित्व स्थानीय रूप से रखना चाहिए। संगतता एक्सपोर्ट केवल तभी हटाए जा सकते हैं, जब
न्यूनतम समर्थित प्रकाशित पैकेज संस्करण उन्हें इम्पोर्ट करना बंद कर दें।

### चैनल सेटअप इनपुट फ़ील्ड संगतता

`ChannelSetupInput` अब केवल क्रॉस-चैनल सेटअप एनवलप को स्थायी रूप से
टाइप किया हुआ रखता है। चैनल-विशिष्ट फ़ील्ड एक बहिष्कृत संगतता
स्तर में टाइप की हुई रहती हैं, ताकि मौजूदा बाहरी Plugin अब भी कम्पाइल हों, जबकि Plugin लेखक उन
फ़ील्ड को Plugin-स्थानीय सेटअप इनपुट प्रकारों में ले जाते हैं।

OpenClaw प्रमुख रिलीज़ जारी नहीं करता। 2026-07-22 को रजिस्ट्री के एक स्वीप ने
426 प्रकाशित आउट-ऑफ़-ट्री चैनल Plugin की जाँच की और बिना किसी रीडर वाली 21 फ़ील्ड हटा दीं।
बनाए रखी गई 22 फ़ील्ड में से प्रत्येक का एक ज्ञात प्रकाशित रीडर है। प्रत्येक अगली फ़ील्ड
जैसे ही किसी प्रकाशित Plugin द्वारा पढ़ी नहीं जाती, हटा दी जाती है; Plugin लेखक जैसे-जैसे
Plugin-स्थानीय सेटअप इनपुट प्रकारों पर माइग्रेट करते हैं, बनाए रखा गया सेट छोटा होता जाता है।

उसी स्वीप ने बिना किसी प्रकाशित आश्रित वाली 23 पुरानी अघोषित-अडैप्टर प्रमोशन कुंजियाँ हटा दीं।
छह सामान्य कुंजियाँ और केवल-सेटअप `rooms` कुंजी शेष हैं।
प्रकाशित Plugin द्वारा `singleAccountKeysToMove` घोषित किए जाने के साथ वह सेट भी छोटा होता जाता है।

साझा प्रकार में कोई इंडेक्स सिग्नेचर नहीं है। Plugin-स्वामित्व वाली कुंजियाँ अब भी रनटाइम
इनपुट ऑब्जेक्ट पर मौजूद हो सकती हैं; उन्हें Plugin-स्थानीय इंटरसेक्शन में घोषित करें या
स्वामी Plugin के सेटअप स्कीमा के माध्यम से संकीर्ण करें।

| `code`                                  | `owner`   | `replacement`                                                                                    | हटाने की शर्त                                                          |
| --------------------------------------- | --------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| `plugin-sdk-channel-setup-input-fields` | `channel` | `ChannelSetupInput` को ऐसे Plugin-स्थानीय प्रकार के साथ इंटरसेक्ट करें, जो स्वामी चैनल की फ़ील्ड घोषित करता हो | जब प्रकाशित-Plugin रजिस्ट्री स्वीप में कोई रीडर न हो, तब फ़ील्ड हटाएँ |

पुराना अघोषित-अडैप्टर प्रमोशन स्तर उसी रीडर-संचालित
नीति का पालन करता है। `singleAccountKeysToMove` घोषित करें, जिसमें तब एक खाली ऐरे भी शामिल हो जब
Plugin को किसी अतिरिक्त प्रमोशन कुंजी की आवश्यकता न हो, ताकि साझा फ़ॉलबैक को एक
समय में एक कुंजी करके हटाया जा सके।

#### रीडर का सत्यापन

1. प्रत्येक `nextCursor` के साथ `https://clawhub.ai/api/v1/packages?family=code-plugin&limit=100` के पृष्ठों से गुजरें और वे पैकेज रखें जिनके `categories` में `channels` शामिल है।
2. `npm search --json --searchlimit=1000 "openclaw channel plugin"` से npm उम्मीदवार जोड़ें। `openclaw/plugin-sdk/channel-setup`, `openclaw/plugin-sdk/setup` और `openclaw/plugin-sdk/core` के लिए GitHub कोड खोजों से केवल-स्रोत उम्मीदवार जोड़ें।
3. प्रत्येक उम्मीदवार का नवीनतम प्रकाशित संस्करण निर्धारित करें। `npm pack <package>@<version> --json --pack-destination <temp-dir>` चलाएँ, उसे अनपैक करें और प्रत्यक्ष या डिस्ट्रक्चर्ड फ़ील्ड रीड के लिए भेजे गए `dist` JavaScript और घोषणाओं की जाँच करें। जब किसी पैकेज का कोई npm रिलीज़ न हो, तो ClawHub आर्टिफ़ैक्ट डाउनलोड करें।
4. पैकेज, संस्करण, फ़ील्ड या प्रमोशन कुंजी और मेल खाने वाली फ़ाइल दर्ज करें। कोई फ़ील्ड या कुंजी केवल तभी हटाई जा सकती है, जब कोई प्रकाशित Plugin आर्टिफ़ैक्ट उसे न पढ़ता हो। बनाए रखी गई फ़ील्ड और कुंजी सूचियों के पास कोड टिप्पणियों में दिए रीडर नामों को स्वीप के साथ सिंक्रनाइज़ रखें।

यह केवल एक स्रोत/प्रकार संगतता रिकॉर्ड है। इसमें कोई रनटाइम अडैप्टर या
संगतता-रजिस्ट्री प्रविष्टि नहीं है, क्योंकि रनटाइम सेटअप इनपुट ऑब्जेक्ट और सेटअप
व्यवहार अपरिवर्तित हैं।

`pnpm plugins:boundary-report` के साथ वर्तमान माइग्रेशन क्यू का ऑडिट करें:

| फ़्लैग                                                    | प्रभाव                                                                           |
| ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `--summary` (या `pnpm plugins:boundary-report:summary`) | पूर्ण विवरण के बजाय संक्षिप्त गणनाएँ।                                            |
| `--json`                                                | मशीन-पठनीय रिपोर्ट।                                                              |
| `--owner <id>`                                          | एक Plugin या संगतता स्वामी तक फ़िल्टर करें।                                      |
| `--fail-on-cross-owner`                                 | क्रॉस-ओनर आरक्षित SDK इम्पोर्ट पर गैर-शून्य स्थिति के साथ बाहर निकलें।            |
| `--fail-on-eligible-compat`                             | जब किसी बहिष्कृत कॉम्पैट रिकॉर्ड की `removeAfter` तिथि बीत चुकी हो, तब गैर-शून्य स्थिति के साथ बाहर निकलें। |
| `--fail-on-unclassified-unused-reserved`                | अप्रयुक्त आरक्षित SDK शिम पर गैर-शून्य स्थिति के साथ बाहर निकलें।                 |

`pnpm plugins:boundary-report:ci` तीनों विफलता फ़्लैग के साथ चलता है। बहिष्कृत
रिकॉर्ड में सामान्यतः अस्पष्ट "अगली प्रमुख रिलीज़" के बजाय एक स्पष्ट `removeAfter` तिथि होती है।
जिस रिकॉर्ड के स्वामी ने किसी तिथि को अनुमोदित नहीं किया है, उसमें
`removeAfter` अनुपस्थित रहता है, वह `no-date` के रूप में दिखाई देता है और कभी भी हटाए जाने योग्य नहीं होता।
रिपोर्ट बहिष्कृत रिकॉर्ड को तिथि के अनुसार समूहित करती है, स्थानीय कोड/दस्तावेज़ संदर्भों की गणना करती है,
क्रॉस-ओनर आरक्षित SDK इम्पोर्ट सामने लाती है और निजी
मेमोरी-होस्ट SDK ब्रिज का सारांश देती है। आरक्षित SDK सबपाथ में ट्रैक किया गया स्वामी उपयोग होना चाहिए;
अप्रयुक्त आरक्षित एक्सपोर्ट को सार्वजनिक SDK से हटा देना चाहिए।

### पुराना मीडिया प्रोजेक्शन

`media-legacy-projection` संगतता रिकॉर्ड पुरानी समानांतर
मीडिया फ़ील्ड, पेलोड बिल्डर, हुक मेटाडेटा उपनाम और मीडिया टेम्पलेट
नामों को कवर करता है। इसकी अनुमोदित `removeAfter` तिथि **2026-10-01** है (फ़ैक्ट्स-फ़र्स्ट
प्रतिस्थापन भेजे जाने के दो रिलीज़ क्रम बाद)। हटाने के लिए उस समय
प्रकाशित-Plugin आर्टिफ़ैक्ट का स्वच्छ स्वीप भी आवश्यक है; तिथि से पहले माइग्रेट करें।

चैनल इनग्रेस के लिए, एकवचन/बहुवचन `MediaPath`, `MediaUrl`,
`MediaType`, `MediaPaths`, `MediaUrls`, `MediaTypes`,
`MediaTranscribedIndexes`, `MediaWorkspaceDir` और `MediaStaged` को क्रमबद्ध
फ़ैक्ट से बदलें:

```ts
import { toInboundMediaFacts } from "openclaw/plugin-sdk/channel-inbound";

const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

`inbound_claim` और `message_received` हुक में `event.media` का उपयोग करें। यदि रिमोट
मीडिया स्थानीय रूप से स्टेज नहीं किया गया है, तो पहचान/निदान के लिए `event.originalMedia` का उपयोग करें
और `event.media` की प्रतीक्षा करें; `event.mediaStagingPending` उस
स्थिति को अलग करता है। `event.metadata` से बहिष्कृत एकवचन/बहुवचन गुण
न पढ़ें।

CLI मीडिया मॉडल के लिए, `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}`
और `{{MediaDir}}` को `{{AttachmentPath}}`, `{{AttachmentUrl}}`,
`{{AttachmentContentType}}` और `{{AttachmentDir}}` से बदलें। जब अटैचमेंट की स्थिति
महत्त्वपूर्ण हो, तो `{{AttachmentIndex}}` का उपयोग करें।

स्थानीय मीडिया रीड नीति के लिए,
`openclaw/plugin-sdk/media-local-roots` से `getAgentScopedMediaLocalRoots(...)` या
`getAgentScopedMediaLocalRootsForSources(...)` इम्पोर्ट करें।
`openclaw/plugin-sdk/agent-media-payload` फ़साड और उसका
`buildAgentMediaPayload(...)` प्रोजेक्शन बहिष्कृत हैं।

## माइग्रेट कैसे करें

<Steps>
  <Step title="रनटाइम कॉन्फ़िग लोड/राइट हेल्पर माइग्रेट करें">
    बंडल किए गए Plugin को सीधे `api.runtime.config.loadConfig()` और
    `api.runtime.config.writeConfigFile(...)` कॉल करना बंद कर देना चाहिए। सक्रिय कॉल पाथ में पहले से
    पास किए गए कॉन्फ़िग को प्राथमिकता दें। वर्तमान प्रक्रिया स्नैपशॉट की आवश्यकता वाले
    दीर्घकालिक हैंडलर `api.runtime.config.current()` का उपयोग कर सकते हैं। दीर्घकालिक
    एजेंट टूल को `execute` के भीतर `ctx.getRuntimeConfig()` पढ़ना चाहिए, ताकि कॉन्फ़िग राइट
    से पहले बनाया गया टूल भी रीफ़्रेश किया गया कॉन्फ़िग देख सके।

    कॉन्फ़िग राइट स्पष्ट आफ़्टर-राइट नीति वाले ट्रांज़ैक्शनल हेल्पर से होकर जाते हैं:

    ```typescript
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    `afterWrite: { mode: "restart", reason: "..." }` का उपयोग तब करें जब परिवर्तन के लिए
    Gateway को साफ़ तरीके से पुनः आरंभ करना आवश्यक हो, और `afterWrite: { mode: "none", reason: "..." }`
    का उपयोग केवल तब करें जब कॉलर अनुवर्ती कार्रवाई का स्वामी हो और जानबूझकर
    रीलोड प्लानर को दबाता हो। म्यूटेशन परिणामों में परीक्षणों और लॉगिंग के लिए
    टाइप किया हुआ `followUp` सारांश शामिल होता है; पुनः आरंभ लागू करने या
    शेड्यूल करने की ज़िम्मेदारी Gateway की ही रहती है।

    `loadConfig` और `writeConfigFile` को Plugin
    रनटाइम से हटा दिया गया है। बंडल किए गए Plugins और रिपॉज़िटरी रनटाइम कोड को
    `pnpm check:deprecated-api-usage` और
    `pnpm check:no-runtime-action-load-config` द्वारा सुरक्षित किया जाता है: नया उत्पादन Plugin उपयोग
    सीधे विफल होता है, प्रत्यक्ष कॉन्फ़िगरेशन लेखन विफल होता है, Gateway सर्वर विधियों को
    अनुरोध रनटाइम स्नैपशॉट का उपयोग करना आवश्यक है, रनटाइम चैनल प्रेषण/कार्रवाई/क्लाइंट सहायकों को
    अपनी सीमा से कॉन्फ़िगरेशन प्राप्त करना आवश्यक है, और दीर्घजीवी रनटाइम मॉड्यूल
    शून्य परिवेशीय `loadConfig()` कॉल की अनुमति देते हैं।

    नए Plugin कोड को व्यापक `openclaw/plugin-sdk/config-runtime`
    बैरल से बचना चाहिए। कार्य के लिए संकीर्ण उपपथ का उपयोग करें:

    | आवश्यकता | आयात |
    | --- | --- |
    | `OpenClawConfig` जैसे कॉन्फ़िगरेशन प्रकार | `openclaw/plugin-sdk/config-contracts` |
    | Plugin-प्रविष्टि कॉन्फ़िगरेशन लुकअप | `api.pluginConfig` |
    | कॉन्फ़िगरेशन मर्ज करना | कॉन्फ़िगरेशन सीमा पर Plugin-स्थानीय तर्क |
    | वर्तमान रनटाइम स्नैपशॉट पठन | `openclaw/plugin-sdk/runtime-config-snapshot` |
    | कॉन्फ़िगरेशन लेखन | `openclaw/plugin-sdk/config-mutation` |
    | सत्र स्टोर सहायक | `openclaw/plugin-sdk/session-store-runtime` |
    | Markdown तालिका कॉन्फ़िगरेशन | `openclaw/plugin-sdk/markdown-table-runtime` |
    | समूह नीति रनटाइम सहायक | `openclaw/plugin-sdk/runtime-group-policy` |
    | गोपनीय इनपुट समाधान | `openclaw/plugin-sdk/secret-input-runtime` |
    | मॉडल/सत्र ओवरराइड | `openclaw/plugin-sdk/model-session-runtime` |

    बंडल किए गए Plugins और उनके परीक्षणों को स्कैनर द्वारा व्यापक
    बैरल से सुरक्षित किया जाता है, ताकि आयात और मॉक केवल आवश्यक व्यवहार तक स्थानीय रहें।
    बाहरी संगतता के लिए बैरल अब भी मौजूद है, लेकिन नए कोड को
    उस पर निर्भर नहीं होना चाहिए।

  </Step>

  <Step title="एम्बेडेड टूल-परिणाम एक्सटेंशन को मिडलवेयर में माइग्रेट करें">
    बंडल किए गए Plugins को केवल एम्बेडेड रनर वाले
    `api.registerEmbeddedExtensionFactory(...)` टूल-परिणाम हैंडलर को
    रनटाइम-निरपेक्ष मिडलवेयर से बदलना आवश्यक है:

    ```typescript
    // OpenClaw रनटाइम टूल और Codex रनटाइम डायनेमिक टूल (परिणाम को
    // रूपांतरित किया जा सकता है)। Codex-नेटिव टूल परिणाम भी अवलोकन के लिए रिले किए जाते हैं,
    // लेकिन उनका रूपांतरित आउटपुट मॉडल तक कभी नहीं पहुँचता: Codex
    // PostToolUse हुक अनुबंध किसी नेटिव टूल प्रतिक्रिया को प्रतिस्थापित नहीं कर सकता।
    api.registerAgentToolResultMiddleware(async (event) => {
      return compactToolResult(event);
    }, {
      runtimes: ["openclaw", "codex"],
    });
    ```

    उसी समय Plugin मैनिफ़ेस्ट को अपडेट करें:

    ```json
    {
      "contracts": {
        "agentToolResultMiddleware": ["openclaw", "codex"]
      }
    }
    ```

    इंस्टॉल किए गए Plugins भी टूल-परिणाम मिडलवेयर पंजीकृत कर सकते हैं, जब वह स्पष्ट रूप से
    सक्षम हो और प्रत्येक लक्षित रनटाइम
    `contracts.agentToolResultMiddleware` में घोषित हो। अघोषित इंस्टॉल किए गए मिडलवेयर
    पंजीकरण अस्वीकार कर दिए जाते हैं।

  </Step>

  <Step title="अनुमोदन-नेटिव हैंडलर को क्षमता तथ्यों में माइग्रेट करें">
    अनुमोदन-सक्षम चैनल Plugins नेटिव अनुमोदन व्यवहार को
    `approvalCapability.nativeRuntime` और साझा रनटाइम-संदर्भ
    रजिस्ट्री के माध्यम से उजागर करते हैं:

    - `approvalCapability.handler.loadRuntime(...)` को
      `approvalCapability.nativeRuntime` से बदलें।
    - अनुमोदन-विशिष्ट प्रमाणीकरण/वितरण को पुराने `plugin.auth` /
      `plugin.approvals` वायरिंग से हटाकर `approvalCapability` पर ले जाएँ।
    - `ChannelPlugin.approvals` को सार्वजनिक
      चैनल-Plugin अनुबंध से हटा दिया गया है; वितरण/नेटिव/रेंडर फ़ील्ड को
      `approvalCapability` पर ले जाएँ।
    - `plugin.auth` केवल चैनल लॉगिन/लॉगआउट प्रवाहों के लिए बना हुआ है; कोर अब
      वहाँ अनुमोदन प्रमाणीकरण हुक नहीं पढ़ता।
    - चैनल-स्वामित्व वाले रनटाइम ऑब्जेक्ट (क्लाइंट, टोकन, Bolt ऐप्स)
      `openclaw/plugin-sdk/channel-runtime-context` के माध्यम से पंजीकृत करें।
    - नेटिव अनुमोदन हैंडलर से Plugin-स्वामित्व वाली पुनः-रूट सूचना न भेजें;
      वास्तविक वितरण परिणामों से अन्यत्र रूट की गई सूचनाओं का स्वामी कोर है।
    - `channelRuntime` को `createChannelManager(...)` में पास करते समय, एक
      वास्तविक `createPluginRuntime().channel` सतह प्रदान करें—आंशिक स्टब
      अस्वीकार कर दिए जाते हैं।

    वर्तमान अनुमोदन क्षमता संरचना के लिए [चैनल Plugins](/hi/plugins/sdk-channel-plugins) देखें।

  </Step>

  <Step title="Windows रैपर फ़ॉलबैक व्यवहार का ऑडिट करें">
    यदि आपका Plugin `openclaw/plugin-sdk/windows-spawn` का उपयोग करता है, तो समाधान न हो पाने वाले Windows
    `.cmd`/`.bat` रैपर अब सुरक्षित रूप से विफल होते हैं, जब तक आप स्पष्ट रूप से
    `allowShellFallback: true` पास न करें:

    ```typescript
    // पहले
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // बाद में
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // इसे केवल उन विश्वसनीय संगतता कॉलर के लिए सेट करें जो जानबूझकर
      // शेल-मध्यस्थ फ़ॉलबैक स्वीकार करते हैं।
      allowShellFallback: true,
    });
    ```

    यदि आपका कॉलर जानबूझकर शेल फ़ॉलबैक पर निर्भर नहीं है, तो
    `allowShellFallback` सेट न करें और इसके बजाय उत्पन्न त्रुटि को संभालें।

  </Step>

  <Step title="अप्रचलित आयात खोजें">
    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "plugin-sdk/infra-runtime" my-plugin/
    grep -r "plugin-sdk/config-runtime" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```
  </Step>

  <Step title="केंद्रित आयातों से बदलें">
    पुरानी सतह से प्रत्येक निर्यात एक विशिष्ट आधुनिक आयात पथ से मैप होता है:

    ```typescript
    // पहले (अप्रचलित पश्च-संगतता परत)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // बाद में (आधुनिक केंद्रित आयात)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    होस्ट-पक्ष के सहायकों के लिए सीधे आयात करने के बजाय
    इंजेक्ट किए गए Plugin रनटाइम का उपयोग करें:

    ```typescript
    // पहले (अप्रचलित extension-api ब्रिज)
    import { runEmbeddedAgent } from "openclaw/extension-api";
    const result = await runEmbeddedAgent({ sessionId, prompt });

    // बाद में (इंजेक्ट किया गया रनटाइम)
    const result = await api.runtime.agent.runEmbeddedAgent({ sessionId, prompt });
    ```

    अन्य पुराने ब्रिज सहायकों के लिए भी यही पैटर्न है:

    | पुराना आयात | आधुनिक समकक्ष |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | सत्र स्टोर सहायक | `api.runtime.agent.session.*` |

  </Step>

  <Step title="व्यापक infra-runtime आयातों को बदलें">
    `openclaw/plugin-sdk/infra-runtime` बाहरी
    संगतता के लिए अब भी मौजूद है, लेकिन नए कोड को वह केंद्रित सतह आयात करनी चाहिए जिसकी उसे वास्तव में
    आवश्यकता है:

    | आवश्यकता | आयात |
    | --- | --- |
    | सिस्टम इवेंट कतार सहायक | `openclaw/plugin-sdk/system-event-runtime` |
    | Heartbeat जागरण, इवेंट और दृश्यता सहायक | `openclaw/plugin-sdk/heartbeat-runtime` |
    | लंबित वितरण कतार निकासी | `openclaw/plugin-sdk/delivery-queue-runtime` |
    | चैनल गतिविधि टेलीमेट्री | `openclaw/plugin-sdk/channel-activity-runtime` |
    | इन-मेमोरी और स्थायी-बैक्ड डीडुप कैश | `openclaw/plugin-sdk/dedupe-runtime` |
    | सुरक्षित स्थानीय-फ़ाइल/मीडिया पथ सहायक | `openclaw/plugin-sdk/file-access-runtime` |
    | डिस्पैचर-जागरूक फ़ेच | `openclaw/plugin-sdk/runtime-fetch` |
    | प्रॉक्सी और सुरक्षित फ़ेच सहायक | `openclaw/plugin-sdk/fetch-runtime` |
    | SSRF डिस्पैचर नीति प्रकार | `openclaw/plugin-sdk/ssrf-dispatcher` |
    | अनुमोदन अनुरोध/समाधान प्रकार | `openclaw/plugin-sdk/approval-runtime` |
    | अनुमोदन उत्तर पेलोड और कमांड सहायक | `openclaw/plugin-sdk/approval-reply-runtime` |
    | त्रुटि फ़ॉर्मैटिंग सहायक | `openclaw/plugin-sdk/error-runtime` |
    | ट्रांसपोर्ट तत्परता प्रतीक्षा | `openclaw/plugin-sdk/transport-ready-runtime` |
    | सुरक्षित टोकन सहायक | `openclaw/plugin-sdk/secure-random-runtime` |
    | सीमित एसिंक्रोनस कार्य समवर्तीता | `openclaw/plugin-sdk/concurrency-runtime` |
    | सिद्ध किए जा सकने वाले अपरिवर्तनीय नियमों के लिए आवश्यक-मान अभिकथन | `openclaw/plugin-sdk/expect-runtime` |
    | संख्यात्मक रूपांतरण | `openclaw/plugin-sdk/number-runtime` |
    | प्रक्रिया-स्थानीय एसिंक्रोनस लॉक | `openclaw/plugin-sdk/async-lock-runtime` |
    | फ़ाइल लॉक | `openclaw/plugin-sdk/file-lock` |

    बंडल किए गए Plugins को स्कैनर द्वारा `infra-runtime` से सुरक्षित किया जाता है, इसलिए रिपॉज़िटरी कोड
    व्यापक बैरल पर वापस नहीं जा सकता।

  </Step>

  <Step title="चैनल रूट सहायकों को माइग्रेट करें">
    नया चैनल रूट कोड `openclaw/plugin-sdk/channel-route` का उपयोग करता है। पुराने
    रूट-कुंजी नाम संगतता उपनाम के रूप में बने हुए हैं:

    | पुराना सहायक | आधुनिक सहायक |
    | --- | --- |
    | `channelRouteIdentityKey(...)` | `channelRouteDedupeKey(...)` |
    | `channelRouteKey(...)` | `channelRouteCompactKey(...)` |

    आधुनिक रूट सहायक नेटिव अनुमोदनों, उत्तर दमन, इनबाउंड डीडुप,
    Cron वितरण और सत्र रूटिंग में `{ channel, to, accountId, threadId }` को
    सुसंगत रूप से सामान्यीकृत करते हैं।

    `plugin-sdk/channel-route` से `ChannelMessagingAdapter.parseExplicitTarget` या
    `resolveChannelRouteTargetWithParser(...)` के नए उपयोग न जोड़ें—ये अप्रचलित हैं और केवल पुराने
    Plugins के लिए बने हुए हैं। नए चैनल Plugins को लक्ष्य-ID सामान्यीकरण
    और डायरेक्टरी-मिस फ़ॉलबैक के लिए
    `messaging.targetResolver.resolveTarget(...)`, जब कोर को आरंभिक पीयर प्रकार चाहिए तब
    `messaging.inferTargetChatType(...)`, और प्रदाता-नेटिव
    सत्र तथा थ्रेड पहचान के लिए `messaging.resolveOutboundSessionRoute(...)` का उपयोग करना चाहिए।

  </Step>

  <Step title="बिल्ड और परीक्षण करें">
    ```bash
    pnpm build
    pnpm test my-plugin/
    ```
  </Step>
</Steps>

## आयात पथ संदर्भ

सार्वजनिक पैकेज निर्यात मैप आयात योग्य SDK
उपपथों के लिए सत्य का स्रोत है। [SDK अवलोकन](/hi/plugins/sdk-overview) से लिंक की गई विषयगत SDK मार्गदर्शिकाओं का उपयोग करें
और सबसे संकीर्ण दस्तावेज़ीकृत सार्वजनिक उपपथ को प्राथमिकता दें। `scripts/lib/plugin-sdk-entrypoints.json` की
कंपाइलर सूची में बंडल किए गए Plugins बनाने के लिए उपयोग की जाने वाली निजी-स्थानीय प्रविष्टियाँ भी हैं;
वहाँ उनकी उपस्थिति उन्हें सार्वजनिक पैकेज निर्यात नहीं बनाती।

यह तालिका सामान्य माइग्रेशन उपसमुच्चय है, संपूर्ण SDK सतह नहीं। कंपाइलर
प्रवेश-बिंदु सूची `scripts/lib/plugin-sdk-entrypoints.json` में है;
पैकेज निर्यात सार्वजनिक उपसमुच्चय से जनरेट किए जाते हैं।

आरक्षित बंडल-Plugin सहायक सीमों को सार्वजनिक SDK
निर्यात मैप से हटा दिया गया है, सिवाय स्पष्ट रूप से दस्तावेज़ीकृत संगतता फ़साड के, जैसे कि
अप्रचलित `plugin-sdk/discord` शिम, जिसे उन बाहरी Plugins के लिए बनाए रखा गया है जो अब भी
प्रकाशित `@openclaw/discord` पैकेज को सीधे आयात करते हैं। स्वामी-विशिष्ट
सहायक स्वामी Plugin पैकेज के भीतर रहते हैं; साझा होस्ट व्यवहार
`plugin-sdk/gateway-runtime`, `plugin-sdk/security-runtime` और इंजेक्ट किए गए Plugin API जैसे
जेनेरिक SDK अनुबंधों के माध्यम से जाता है।

कार्य से मेल खाने वाले सबसे संकीर्ण आयात का उपयोग करें। यदि आपको कोई निर्यात नहीं मिलता,
तो `src/plugin-sdk/` पर स्रोत जाँचें या अनुरक्षकों से पूछें कि उसका स्वामी कौन-सा जेनेरिक
अनुबंध होना चाहिए।

## हटाई गई संगतता सतहें

जुलाई 2026 की छँटाई में रूट SDK और संगतता बैरल, एक्सटेंशन API
ब्रिज, समाप्त SDK उपपथ उपनाम, अप्रयुक्त SDK उपपथ और केवल बंडल के लिए बने SDK मॉड्यूल के सार्वजनिक
निर्यात हटा दिए गए। केवल बंडल वाले मॉड्यूल निजी-स्थानीय बिल्ड मैपिंग के माध्यम से
उनके रिपॉज़िटरी स्वामियों के लिए उपलब्ध रहते हैं; उन्हें प्रकाशित पैकेज से
आयात नहीं किया जा सकता।

### प्रक्रिया-वैश्विक API-प्रदाता प्रकाशन

`registerApiProvider(...)` और `unregisterApiProviders(...)` को
`openclaw/plugin-sdk/llm` से हटा दिया गया। वे API ट्रांसपोर्ट को प्रक्रिया-वैश्विक
स्थिति में प्रकाशित करते थे, जिसे जीवनचक्र-स्वामित्व वाले मॉडल रनटाइम को फिर प्रत्येक तैयार
रजिस्ट्री में कॉपी करना पड़ता था।

प्रदाता Plugins को टेक्स्ट-इन्फ़रेंस प्रदाताओं को
`api.registerProvider(...)` के माध्यम से पंजीकृत करना चाहिए। `ApiRegistry` बनाने वाले होस्ट-स्वामित्व वाले
कोड और परीक्षणों को सीधे उस रजिस्ट्री पर पंजीकरण करना चाहिए, ताकि प्रदाता स्वामित्व
और टियरडाउन तैयार रनटाइम तक सीमित रहें।

### निजी परीक्षण बैरल

`openclaw/plugin-sdk/testing` रिपॉज़िटरी-स्थानीय था और वितरित पैकेज
आर्टिफ़ैक्ट से बाहर रखा गया था, इसलिए इसे इसकी 2026-07-28 `removeAfter` तिथि से पहले हटा दिया गया। रिपॉज़िटरी
परीक्षण `plugin-sdk/plugin-test-runtime`, `plugin-sdk/channel-test-helpers`, `plugin-sdk/channel-target-testing`,
`plugin-sdk/test-env` और `plugin-sdk/test-fixtures` जैसे केंद्रित उपपथों का उपयोग करते हैं।

## माइग्रेशन संदर्भ

  ये मैपिंग जुलाई 2026 में हटाई गई सतहों और बाद की समयावधि में सक्रिय
  बहिष्करणों, दोनों को कवर करती हैं। कोई मैपिंग माइग्रेशन मार्गदर्शन है, इसका प्रमाण नहीं कि पुरानी
  सतह अब भी उपलब्ध है; वर्तमान स्थिति के लिए संगतता रजिस्ट्री और हटाने की
  समयरेखा देखें।

  <AccordionGroup>
  <Accordion title="command-auth सहायता बिल्डर -> command-status">
    **पुराना (`openclaw/plugin-sdk/command-auth`)**: `buildCommandsMessage`,
    `buildCommandsMessagePaginated`, `buildHelpMessage`।

    **नया (`openclaw/plugin-sdk/command-status`)**: समान सिग्नेचर, अधिक सीमित
    उपपथ से इंपोर्ट किए गए। `command-auth` संगतता री-एक्सपोर्ट
    हटा दिए गए हैं।

    ```typescript
    // पहले
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-auth";

    // बाद में
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-status";
    ```

  </Accordion>

  <Accordion title="मेंशन गेटिंग सहायक -> resolveInboundMentionDecision">
    **पुराना**: `resolveMentionGating(params)` और
    `resolveMentionGatingWithBypass(params)`, जो
    `openclaw/plugin-sdk/channel-inbound` या
    `openclaw/plugin-sdk/channel-mention-gating` से मिलते थे।

    **नया**: `resolveInboundMentionDecision({ facts, policy })`—दो अलग-अलग कॉल
    आकृतियों के बजाय एक निर्णय ऑब्जेक्ट।

    इसे Discord, iMessage, Matrix, MS Teams, QQBot, Signal,
    Telegram, WhatsApp और Zalo में अपनाया गया है। Slack का अपना `app_mention` इवेंट मॉडल
    इस सहायक का उपयोग नहीं करता।

  </Accordion>

  <Accordion title="चैनल रनटाइम शिम और चैनल कार्रवाई सहायक">
    `openclaw/plugin-sdk/channel-runtime` हटा दिया गया है। रनटाइम
    ऑब्जेक्ट पंजीकृत करने के लिए `openclaw/plugin-sdk/channel-runtime-context` का उपयोग करें।

    `openclaw/plugin-sdk/channel-actions` में मौजूद मूल संदेश स्कीमा सहायक
    कच्चे "actions" चैनल एक्सपोर्ट के साथ हटा दिए गए थे। इसके बजाय क्षमताओं को
    सिमैंटिक `presentation` सतह के माध्यम से उजागर करें—चैनल plugins यह घोषित
    करते हैं कि वे क्या रेंडर करते हैं (कार्ड, बटन, चयन), न कि यह कि वे किन कच्चे
    कार्रवाई नामों को स्वीकार करते हैं।

  </Accordion>

  <Accordion title="वेब खोज प्रदाता का tool() सहायक -> Plugin पर createTool()">
    **पुराना**: `openclaw/plugin-sdk/provider-web-search` से `tool()` फैक्टरी।

    **नया**: प्रदाता Plugin पर सीधे `createTool(...)` लागू करें।
    टूल रैपर पंजीकृत करने के लिए OpenClaw को अब SDK सहायक की आवश्यकता नहीं है।

  </Accordion>

  <Accordion title="प्लेनटेक्स्ट चैनल एनवेलप -> BodyForAgent">
    **पुराना**: इनबाउंड चैनल संदेशों से एक समतल
    प्लेनटेक्स्ट प्रॉम्प्ट एनवेलप बनाने के लिए `api.runtime.channel.reply.formatInboundEnvelope(...)` (और इनबाउंड संदेश ऑब्जेक्ट पर
    `channelEnvelope` फ़ील्ड)।

    **नया**: `BodyForAgent` और संरचित उपयोगकर्ता-संदर्भ ब्लॉक। चैनल
    plugins रूटिंग मेटाडेटा (थ्रेड, विषय, प्रत्युत्तर-लक्ष्य, प्रतिक्रियाएँ) को
    प्रॉम्प्ट स्ट्रिंग में जोड़ने के बजाय टाइप किए गए फ़ील्ड के रूप में संलग्न करते हैं।
    `formatAgentEnvelope(...)` सहायक संश्लेषित
    असिस्टेंट-सामना करने वाले एनवेलप के लिए अब भी समर्थित है, लेकिन इनबाउंड प्लेनटेक्स्ट एनवेलप हटाए
    जा रहे हैं।

    प्रभावित क्षेत्र: `inbound_claim`, `message_received`, और ऐसा कोई भी कस्टम
    चैनल Plugin जिसने पुराने एनवेलप टेक्स्ट को बाद में संसाधित किया था।

  </Accordion>

  <Accordion title="deactivate हुक -> gateway_stop">
    **पुराना**: `api.on("deactivate", handler)`।

    **नया**: `api.on("gateway_stop", handler)`। वही शटडाउन क्लीनअप
    अनुबंध; केवल हुक का नाम बदलता है।

    ```typescript
    // पहले
    api.on("deactivate", async (event, ctx) => {
      await stopPluginService(ctx);
    });

    // बाद में
    api.on("gateway_stop", async (event, ctx) => {
      await stopPluginService(ctx);
    });
    ```

    `deactivate` बहिष्कृत संगतता उपनाम के रूप में तब तक जुड़ा रहेगा, जब तक इसे
    2026-08-16 के बाद हटा नहीं दिया जाता।

  </Accordion>

  <Accordion title="subagent_spawning हुक -> कोर थ्रेड बाइंडिंग">
    **पुराना**: `api.on("subagent_spawning", handler)`, जो
    `threadBindingReady` या `deliveryOrigin` लौटाता था।

    **नया**: कोर को चैनल सत्र-बाइंडिंग अडैप्टर के माध्यम से `thread: true` सबएजेंट बाइंडिंग
    तैयार करने दें। केवल लॉन्च-पश्चात अवलोकन के लिए `api.on("subagent_spawned", handler)`
    का उपयोग करें।

    ```typescript
    // पहले
    api.on("subagent_spawning", async () => ({
      status: "ok",
      threadBindingReady: true,
      deliveryOrigin: { channel: "discord", to: "channel:123", threadId: "456" },
    }));

    // बाद में
    api.on("subagent_spawned", async (event) => {
      await observeSubagentLaunch(event);
    });
    ```

    बाहरी plugins के माइग्रेट होने तक `subagent_spawning`, `PluginHookSubagentSpawningEvent`,
    `PluginHookSubagentSpawningResult`, और
    `SubagentLifecycleHookRunner.runSubagentSpawning(...)` केवल
    बहिष्कृत संगतता सतहों के रूप में बने रहेंगे और 2026-08-30 के बाद हटा दिए
    जाएँगे।

  </Accordion>

  <Accordion title="प्रदाता खोज प्रकार -> प्रदाता कैटलॉग प्रकार">
    चार खोज प्रकार उपनाम अब कैटलॉग-युग के प्रकारों पर पतले रैपर
    हैं:

    | पुराना उपनाम                 | नया प्रकार                  |
    | ------------------------- | ------------------------- |
    | `ProviderDiscoveryOrder`  | `ProviderCatalogOrder`    |
    | `ProviderDiscoveryContext`| `ProviderCatalogContext`  |
    | `ProviderDiscoveryResult` | `ProviderCatalogResult`   |
    | `ProviderPluginDiscovery` | `ProviderPluginCatalog`   |

    उपनाम और विरासती `ProviderCapabilities` स्थिर संग्रह हटा दिए गए हैं।
    प्रदाता plugins को स्थिर ऑब्जेक्ट के बजाय `buildReplayPolicy`,
    `normalizeToolSchemas`, और `wrapStreamFn` जैसे स्पष्ट प्रदाता हुक
    का उपयोग करना चाहिए।

  </Accordion>

  <Accordion title="चिंतन नीति हुक -> resolveThinkingProfile">
    **पुराना** (`ProviderThinkingPolicy` पर तीन अलग-अलग हुक):
    `isBinaryThinking(ctx)`, `supportsXHighThinking(ctx)`, और
    `resolveDefaultThinkingLevel(ctx)`।

    **नया**: एकल `resolveThinkingProfile(ctx)`, जो कैनोनिकल `id`, वैकल्पिक `label`, और
    रैंक की गई स्तर सूची वाला
    `ProviderThinkingProfile` लौटाता है। OpenClaw पुराने संग्रहीत मानों को प्रोफ़ाइल रैंक के अनुसार
    स्वचालित रूप से डाउनग्रेड करता है।

    संदर्भ में `provider`, `modelId`, वैकल्पिक मर्ज किया गया `reasoning`,
    और वैकल्पिक मर्ज किए गए मॉडल के `compat` तथ्य शामिल होते हैं। प्रदाता plugins उन
    कैटलॉग तथ्यों का उपयोग करके मॉडल-विशिष्ट प्रोफ़ाइल केवल तभी उजागर कर सकते हैं, जब कॉन्फ़िगर किया गया
    अनुरोध अनुबंध उसका समर्थन करता हो।

    तीन के बजाय एक हुक लागू करें। विरासती हुक हटा दिए गए हैं।

  </Accordion>

  <Accordion title="बाहरी प्रमाणीकरण प्रदाता -> contracts.externalAuthProviders">
    **पुराना**: Plugin मेनिफ़ेस्ट में प्रदाता घोषित किए बिना बाहरी प्रमाणीकरण हुक
    लागू करना।

    **नया**: Plugin मेनिफ़ेस्ट में `contracts.externalAuthProviders` घोषित करें
    **और** `resolveExternalAuthProfiles(...)` लागू करें।

    ```json
    {
      "contracts": {
        "externalAuthProviders": ["anthropic", "openai"]
      }
    }
    ```

  </Accordion>

  <Accordion title="प्रदाता env-var लुकअप -> setup.providers[].envVars">
    **पुराना** मेनिफ़ेस्ट फ़ील्ड: `providerAuthEnvVars: { anthropic: ["ANTHROPIC_API_KEY"] }`।

    **नया**: उसी env-var लुकअप को मेनिफ़ेस्ट पर `setup.providers[].envVars`
    में भी प्रतिबिंबित करें। इससे सेटअप/स्थिति env मेटाडेटा एक स्थान पर समेकित होता है
    और केवल env-var लुकअप का उत्तर देने के लिए Plugin रनटाइम बूट करने से बचा जाता है।

    `providerAuthEnvVars` अब स्वीकार नहीं किया जाता।

  </Accordion>

  <Accordion title="मेमोरी Plugin पंजीकरण -> registerMemoryCapability">
    **पुराना**: तीन अलग-अलग कॉल—`api.registerMemoryPromptSection(...)`,
    `api.registerMemoryFlushPlan(...)`, `api.registerMemoryRuntime(...)`।

    **नया**: मेमोरी-स्टेट API पर एक कॉल—
    `registerMemoryCapability(pluginId, { promptBuilder, flushPlanResolver, runtime })`।

    वही स्लॉट, एकल पंजीकरण कॉल। योगात्मक प्रॉम्प्ट और कॉर्पस सहायक
    (`registerMemoryPromptSupplement`, `registerMemoryCorpusSupplement`) प्रभावित नहीं हैं।

  </Accordion>

  <Accordion title="मेमोरी एम्बेडिंग प्रदाता API">
    **पुराना**: `api.registerMemoryEmbeddingProvider(...)` और
    `contracts.memoryEmbeddingProviders`।

    **नया**: `api.registerEmbeddingProvider(...)` और
    `contracts.embeddingProviders`।

    सामान्य एम्बेडिंग प्रदाता अनुबंध मेमोरी के बाहर भी पुनः उपयोग योग्य है और
    नए प्रदाताओं के लिए समर्थित मार्ग है। मौजूदा प्रदाताओं के
    माइग्रेट होने तक मेमोरी-विशिष्ट पंजीकरण API बहिष्कृत संगतता के रूप में जुड़ा
    रहेगा। Plugin निरीक्षण गैर-बंडल उपयोग को संगतता
    ऋण के रूप में रिपोर्ट करता है।

  </Accordion>

  <Accordion title="कच्चे चैनल प्रेषण परिणाम -> OutboundDeliveryResult">
    **पुराना**: `ChannelSendRawResult` के माध्यम से `{ ok, messageId, error }` लौटाएँ
    और उसे `createRawChannelSendResultAdapter(...)` से सामान्यीकृत करें।

    **नया**: `OutboundDeliveryResult` फ़ील्ड लौटाएँ और चैनल को
    `createAttachedChannelResultAdapter(...)` से संलग्न करें। विफल प्रेषण को त्रुटि स्ट्रिंग
    लौटाने के बजाय अपवाद फेंकना चाहिए। कच्चा परिणाम प्रकार अगले
    Plugin-SDK प्रमुख रिलीज़ तक उपलब्ध रहेगा।

  </Accordion>

  <Accordion title="सबएजेंट सत्र संदेश प्रकारों के नाम बदले गए">
    `src/plugins/runtime/types.ts` से अब भी एक्सपोर्ट किए जाने वाले दो विरासती प्रकार उपनाम:

    | पुराना                           | नया                             |
    | ----------------------------- | ------------------------------- |
    | `SubagentReadSessionParams`   | `SubagentGetSessionMessagesParams` |
    | `SubagentReadSessionResult`   | `SubagentGetSessionMessagesResult` |

    रनटाइम विधि `readSession` को
    `getSessionMessages` के पक्ष में बहिष्कृत किया गया है। समान सिग्नेचर; पुरानी विधि
    नई विधि को कॉल करती है।

  </Accordion>

  <Accordion title="हटाए गए सत्र और ट्रांसक्रिप्ट फ़ाइल API">
    SQLite सत्र/ट्रांसक्रिप्ट परिवर्तन उन Plugin-सामना करने वाले API को हटाता या बहिष्कृत करता है
    जो सक्रिय `sessions.json` स्टोर, JSONL ट्रांसक्रिप्ट पथ, या सत्र
    फ़ाइलों की सूचियाँ उजागर करते थे। रनटाइम plugins को सक्रिय फ़ाइलें हल या परिवर्तित करने के बजाय
    सत्र पहचान और SDK रनटाइम सहायकों का उपयोग करना चाहिए।

    | माइग्रेट की जाने वाली सतह | प्रतिस्थापन |
    | ----------------- | ----------- |
    | बहिष्कृत `loadSessionStore(...)`, `updateSessionStore(...)`, और `resolveSessionStoreEntry(...)` | `getSessionEntry(...)`, `listSessionEntries(...)`, और पंक्ति-स्तरीय सत्र परिवर्तन। |
    | बहिष्कृत `resolveSessionFilePath(...)` | सत्र पहचान (`sessionKey`, `sessionId`, और SDK रनटाइम लक्ष्य सहायक) तथा वर्तमान सत्र पर कार्य करने वाली Gateway विधियाँ। |
    | हटाया गया `saveSessionStore(...)` | Gateway-स्वामित्व वाले सत्र रनटाइम API; Plugin कोड को सक्रिय स्टोर फ़ाइल लिखने के बजाय प्रलेखित रनटाइम/संदर्भ सहायकों के माध्यम से सत्र स्थिति का अनुरोध या परिवर्तन करना चाहिए। |
    | हटाए गए `resolveSessionTranscriptPathInDir(...)` और `resolveAndPersistSessionFile(...)` | सत्र पहचान और वर्तमान सत्र पर कार्य करने वाली Gateway विधियाँ। |
    | `readLatestAssistantTextFromSessionTranscript(...)` | वर्तमान रनटाइम संदर्भ द्वारा उजागर किए गए पहचान-समर्थित ट्रांसक्रिप्ट रीडर, या Plugin के ट्रांसक्रिप्ट स्वामी पथ से बाहर होने पर Gateway इतिहास/सत्र विधियाँ। |
    | `SessionTranscriptUpdate.sessionFile` | `SessionTranscriptUpdate.target`, जिसमें `agentId`, `sessionKey`, और `sessionId` हों। |
    | `sessionFiles` जैसे मेमोरी सिंक इनपुट | होस्ट द्वारा प्रदान किए गए पहचान-समर्थित ट्रांसक्रिप्ट/सत्र स्रोत; लाइव सत्रों के लिए सक्रिय JSONL फ़ाइलें क्रॉल न करें। |
    | सक्रिय सत्रों के लिए `transcriptPath` या `sessionFile` नाम वाले रनटाइम विकल्प | `sessionTarget`/रनटाइम लक्ष्य ऑब्जेक्ट, जो भंडारण-निरपेक्ष सत्र पहचान रखते हैं। |

    विरासती JSONL ट्रांसक्रिप्ट फ़ाइलें इंपोर्ट, अभिलेख, एक्सपोर्ट और
    सहायता आर्टिफ़ैक्ट के रूप में वैध रहती हैं। वे अब सक्रिय सत्रों के लिए
    स्थिर-अवस्था रनटाइम अनुबंध नहीं हैं।

    `v2026.7.1-beta.5` के साथ जारी आधिकारिक plugins ने ऊपर दिए गए चार
    बहिष्कृत सहायक इंपोर्ट किए थे। `openclaw/plugin-sdk/session-store-runtime`
    उस सटीक ब्रिज को 2026-10-12 तक बनाए रखता है; नए plugins को प्रतिस्थापनों का उपयोग करना होगा।
    `resolveStorePath(...)` समर्थित SDK सहायक बना रहेगा और
    इस बहिष्करण का भाग नहीं है।

    `openclaw plugins inspect --all --runtime` उन गैर-बंडल plugins की रिपोर्ट करता है जिनकी
    लोड त्रुटियाँ या निदान अब भी इन हटाए गए फ़ाइल API का संदर्भ देते हैं। रिलीज़ से पहले
    बाहरी पैकेज स्कैन द्वारा संपूर्ण-स्टोर सत्र सहायक,
    सत्र फ़ाइल-पथ सहायक, विरासती ट्रांसक्रिप्ट फ़ाइल लक्ष्य, और निम्न-स्तरीय
    ट्रांसक्रिप्ट सहायक भी चिह्नित किए जाएँ, इसके लिए `@openclaw/plugin-inspector` सलाहकारी स्वीप को संस्करण `0.3.17` या
    उससे नया उपयोग करना होगा।

  </Accordion>

  <Accordion title="runtime.tasks.flow -> runtime.tasks.managedFlows">
    **पुराना**: `runtime.tasks.flow` (एकवचन) एक लाइव टास्क-फ़्लो
    एक्सेसर लौटाता था।

    **नया**: `runtime.tasks.managedFlows` उन plugins के लिए प्रबंधित TaskFlow परिवर्तन
    रनटाइम बनाए रखता है, जो किसी फ़्लो से चाइल्ड टास्क बनाते, अपडेट करते, रद्द करते या चलाते हैं।
    जब Plugin को केवल DTO-आधारित रीड की आवश्यकता हो, तब `runtime.tasks.flows` का उपयोग करें।

    ```typescript
    // पहले
    const flow = api.runtime.tasks.flow.fromToolContext(ctx);
    // बाद में
    const flow = api.runtime.tasks.managedFlows.fromToolContext(ctx);
    ```

    पुराने उपनाम जुलाई 2026 में हटा दिए गए थे।

  </Accordion>

  <Accordion title="अंतर्निहित एक्सटेंशन फ़ैक्ट्रियाँ -> एजेंट टूल-परिणाम मिडलवेयर">
    इसे ऊपर [माइग्रेट करने का तरीका](#how-to-migrate) में शामिल किया गया है। पूर्णता के लिए
    यहाँ भी दिया गया है: हटाए गए केवल-अंतर्निहित-रनर
    `api.registerEmbeddedExtensionFactory(...)` पथ को स्पष्ट रनटाइम सूची वाले
    `api.registerAgentToolResultMiddleware(...)` से
    `contracts.agentToolResultMiddleware` में प्रतिस्थापित किया गया है।
  </Accordion>

  <Accordion title="OpenClawSchemaType उपनाम -> OpenClawConfig">
    `OpenClawSchemaType` रूट-SDK उपनाम हटा दिया गया था। प्रामाणिक
    `OpenClawConfig` नाम का उपयोग करें।

    ```typescript
    // पहले
    import type { OpenClawSchemaType } from "openclaw/plugin-sdk";
    // बाद में
    import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
    ```

  </Accordion>
</AccordionGroup>

<Note>
एक्सटेंशन-स्तरीय अप्रचलन (बंडल किए गए चैनल/प्रोवाइडर Plugins के भीतर
`extensions/`) को उनके अपने `api.ts` और `runtime-api.ts`
बैरल में ट्रैक किया जाता है। वे तृतीय-पक्ष Plugin अनुबंधों को प्रभावित नहीं करते
और यहाँ सूचीबद्ध नहीं हैं। यदि आप किसी बंडल किए गए Plugin के स्थानीय बैरल का
सीधे उपयोग करते हैं, तो अपग्रेड करने से पहले उस बैरल की अप्रचलन टिप्पणियाँ पढ़ें।
</Note>

## Talk और रियलटाइम वॉइस माइग्रेशन

रियलटाइम वॉइस, टेलीफ़ोनी, मीटिंग और ब्राउज़र Talk कोड, `openclaw/plugin-sdk/realtime-voice` द्वारा
निर्यात किए गए एक Talk सेशन कंट्रोलर को साझा करते हैं। कंट्रोलर सामान्य Talk
इवेंट एनवेलप, सक्रिय टर्न स्थिति, कैप्चर स्थिति, आउटपुट-ऑडियो स्थिति, हालिया
इवेंट इतिहास और पुराने टर्न की अस्वीकृति का स्वामी है। प्रोवाइडर Plugins
वेंडर-विशिष्ट रियलटाइम सेशन के स्वामी हैं। ब्राउज़र-मीटिंग Plugins सेशन,
ब्राउज़र, ऑडियो, Node-होस्ट, एजेंट-परामर्श और वॉइस-कॉल तंत्र के लिए
`openclaw/plugin-sdk/meeting-runtime` का उपयोग करते हैं, फिर URL नियमों, DOM स्क्रिप्ट,
मैन्युअल-कार्रवाई मैपिंग, कैप्शन, निर्माण और डायल-इन योजनाओं के लिए
`MeetingPlatformAdapter` लागू करते हैं। प्लेटफ़ॉर्म REST API, OAuth, आर्टिफ़ैक्ट,
सेलेक्टर और वायर नाम Plugin में रहते हैं। ब्राउज़र अनुमति योजनाओं को अनुरोधित
मीटिंग URL मिलता है, ताकि प्रत्येक प्लेटफ़ॉर्म केवल अपने सटीक समर्थित ओरिजिन
की अनुमति दे सके। ब्राउज़र से प्रस्थान की पुष्टि होने के बाद सेशन रनटाइम को
प्लेटफ़ॉर्म-विशिष्ट लाइव स्वास्थ्य भी सामान्यीकृत करना होगा; ऐतिहासिक
ट्रांसक्रिप्ट फ़ील्ड बने रह सकते हैं, लेकिन निकलने के बाद कैप्शन और ऑडियो की
तत्परता सक्रिय नहीं रहनी चाहिए।

सभी बंडल किए गए सरफ़ेस साझा कंट्रोलर पर चलते हैं: ब्राउज़र रिले,
प्रबंधित-रूम हैंडऑफ़, वॉइस-कॉल रियलटाइम, वॉइस-कॉल स्ट्रीमिंग STT, Google
Meet रियलटाइम और नेटिव पुश-टू-टॉक। Gateway, `hello-ok.features.events` में एक लाइव
Talk इवेंट चैनल घोषित करता है: `talk.event`।

नए कोड को `createTalkEventSequencer(...)` को सीधे कॉल नहीं करना चाहिए, जब तक कि
निम्न-स्तरीय अडैप्टर या टेस्ट फ़िक्स्चर लागू न किया जा रहा हो। साझा कंट्रोलर
का उपयोग करें, ताकि टर्न आईडी के बिना टर्न-स्कोप्ड इवेंट उत्सर्जित न किए जा
सकें, पुराने `turnEnd` / `turnCancel` कॉल किसी नए सक्रिय टर्न
को साफ़ न कर सकें, और आउटपुट-ऑडियो जीवनचक्र इवेंट टेलीफ़ोनी, मीटिंग,
ब्राउज़र रिले, प्रबंधित-रूम हैंडऑफ़ और नेटिव Talk क्लाइंट में सुसंगत रहें।

सार्वजनिक API का स्वरूप:

```typescript
// Gateway के स्वामित्व वाला Talk सेशन API।
await gateway.request("talk.session.create", {
  mode: "realtime",
  transport: "gateway-relay",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.session.appendAudio", { sessionId, audioBase64 });
await gateway.request("talk.session.cancelOutput", { sessionId, reason: "barge-in" });
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "working" },
  options: { willContinue: true },
});
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "already_delivered" },
  options: { suppressResponse: true },
});
await gateway.request("talk.session.submitToolResult", { sessionId, callId, result });
await gateway.request("talk.session.close", { sessionId });

// क्लाइंट के स्वामित्व वाला प्रोवाइडर सेशन API।
await gateway.request("talk.client.create", {
  mode: "realtime",
  transport: "webrtc",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.client.toolCall", { sessionKey, callId, name, args });
await gateway.request("talk.client.steer", { sessionKey, text, mode: "steer" });
```

ब्राउज़र के स्वामित्व वाले WebRTC/प्रोवाइडर-वेबसॉकेट सेशन
`talk.client.create` का उपयोग करते हैं, क्योंकि ब्राउज़र प्रोवाइडर नेगोशिएशन और
मीडिया ट्रांसपोर्ट का स्वामी है, जबकि Gateway क्रेडेंशियल, निर्देश और टूल
नीति का स्वामी है। `talk.session.*` Gateway द्वारा प्रबंधित सामान्य सरफ़ेस
है, जिसका उपयोग Gateway-रिले रियलटाइम, Gateway-रिले ट्रांसक्रिप्शन और
प्रबंधित-रूम नेटिव STT/TTS सेशन के लिए होता है।

`talk.provider` / `talk.providers` के पास रियलटाइम सेलेक्टर रखने वाले
पुराने कॉन्फ़िग को `openclaw doctor --fix` से सुधारा जाना चाहिए; रनटाइम Talk,
स्पीच/TTS प्रोवाइडर कॉन्फ़िग को रियलटाइम प्रोवाइडर कॉन्फ़िग के रूप में फिर से
व्याख्यायित नहीं करता।

समर्थित `talk.session.create` संयोजन जानबूझकर सीमित हैं:

| मोड            | ट्रांसपोर्ट       | ब्रेन           | स्वामी              | टिप्पणियाँ                                                                                                              |
| --------------- | --------------- | --------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `realtime`      | `gateway-relay` | `agent-consult` | Gateway            | Gateway के माध्यम से ब्रिज किया गया फ़ुल-डुप्लेक्स प्रोवाइडर ऑडियो; टूल कॉल एजेंट-परामर्श टूल के माध्यम से रूट होते हैं।           |
| `transcription` | `gateway-relay` | `none`          | Gateway            | केवल स्ट्रीमिंग STT; कॉलर इनपुट ऑडियो भेजते हैं और ट्रांसक्रिप्ट इवेंट प्राप्त करते हैं।                                        |
| `stt-tts`       | `managed-room`  | `agent-consult` | नेटिव/क्लाइंट रूम | पुश-टू-टॉक और वॉकी-टॉकी शैली के रूम, जहाँ क्लाइंट कैप्चर/प्लेबैक का और Gateway टर्न स्थिति का स्वामी होता है। |
| `stt-tts`       | `managed-room`  | `direct-tools`  | नेटिव/क्लाइंट रूम | विश्वसनीय प्रथम-पक्ष सरफ़ेस के लिए केवल-एडमिन रूम मोड, जो सीधे Gateway टूल कार्रवाइयाँ निष्पादित करते हैं।                  |

पुराने `talk.realtime.*` / `talk.transcription.*` / `talk.handoff.*` परिवारों
(सभी हटाए गए) से माइग्रेट करने वाले पाठकों के लिए मेथड मैप:

| पुराना                              | नया                                                      |
| -------------------------------- | -------------------------------------------------------- |
| `talk.realtime.session`          | `talk.client.create`                                     |
| `talk.realtime.toolCall`         | `talk.client.toolCall`                                   |
| `talk.realtime.relayAudio`       | `talk.session.appendAudio`                               |
| `talk.realtime.relayCancel`      | `talk.session.cancelOutput` या `talk.session.cancelTurn` |
| `talk.realtime.relayToolResult`  | `talk.session.submitToolResult`                          |
| `talk.realtime.relayStop`        | `talk.session.close`                                     |
| `talk.transcription.session`     | `talk.session.create({ mode: "transcription" })`         |
| `talk.transcription.relayAudio`  | `talk.session.appendAudio`                               |
| `talk.transcription.relayCancel` | `talk.session.cancelTurn`                                |
| `talk.transcription.relayStop`   | `talk.session.close`                                     |
| `talk.handoff.create`            | `talk.session.create({ transport: "managed-room" })`     |
| `talk.handoff.join`              | `talk.session.join`                                      |
| `talk.handoff.revoke`            | `talk.session.close`                                     |

एकीकृत नियंत्रण शब्दावली भी जानबूझकर सीमित है:

| मेथड                          | इन पर लागू                                              | अनुबंध                                                                                                                                                                                                                  |
| ------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `talk.session.appendAudio`      | `realtime/gateway-relay`, `transcription/gateway-relay` | उसी Gateway कनेक्शन के स्वामित्व वाले प्रोवाइडर सेशन में base64 PCM ऑडियो खंड जोड़ें।                                                                                                                             |
| `talk.session.startTurn`        | `stt-tts/managed-room`                                  | प्रबंधित-रूम उपयोगकर्ता टर्न शुरू करें।                                                                                                                                                                                           |
| `talk.session.endTurn`          | `stt-tts/managed-room`                                  | पुराने-टर्न के सत्यापन के बाद सक्रिय टर्न समाप्त करें।                                                                                                                                                                          |
| `talk.session.cancelTurn`       | Gateway के स्वामित्व वाले सभी सेशन                              | किसी टर्न के लिए सक्रिय कैप्चर/प्रोवाइडर/एजेंट/TTS कार्य रद्द करें।                                                                                                                                                                 |
| `talk.session.cancelOutput`     | `realtime/gateway-relay`                                | उपयोगकर्ता टर्न को अनिवार्य रूप से समाप्त किए बिना सहायक ऑडियो आउटपुट रोकें।                                                                                                                                                     |
| `talk.session.submitToolResult` | `realtime/gateway-relay`                                | उसके ब्रिज द्वारा उजागर किसी भी एसिंक्रोनस पूर्णता के बाद प्रोवाइडर टूल कॉल पूरा करें; अंतरिम आउटपुट के लिए `options.willContinue` या, समर्थित होने पर, किसी अन्य सहायक प्रतिक्रिया से बचने के लिए `options.suppressResponse` पास करें। |
| `talk.session.steer`            | एजेंट-समर्थित Talk सेशन                              | Talk सेशन से समाधान किए गए सक्रिय अंतर्निहित रन को मौखिक `status`, `steer`, `cancel` या `followup` नियंत्रण भेजें।                                                                                                 |
| `talk.session.close`            | सभी एकीकृत सेशन                                    | रिले सेशन रोकें या प्रबंधित-रूम स्थिति निरस्त करें, फिर एकीकृत सेशन आईडी भूल जाएँ।                                                                                                                                     |

इसे कार्यशील बनाने के लिए कोर में प्रोवाइडर या प्लेटफ़ॉर्म के विशेष मामले
प्रस्तुत न करें। कोर Talk सेशन के अर्थ-विज्ञान का स्वामी है। प्रोवाइडर Plugins
वेंडर सेशन सेटअप के स्वामी हैं। वॉइस-कॉल और Google Meet टेलीफ़ोनी/मीटिंग
अडैप्टर के स्वामी हैं। ब्राउज़र और नेटिव ऐप डिवाइस कैप्चर/प्लेबैक UX के
स्वामी हैं।

## हटाने की समयरेखा

| कब                                        | क्या होता है                                                                                                                              |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **अभी**                                     | चेतावनी देने में सक्षम अप्रचलित सतहें रनटाइम चेतावनियाँ जारी करती हैं; रिपॉज़िटरी गार्ड, कोर और बंडल किए गए plugins से अप्रचलित SDK इंपोर्ट अस्वीकार करते हैं। |
| **स्वामी का निर्णय लंबित**                  | बिना तारीख वाले रिकॉर्ड तब तक अप्रचलित और हटाने के लिए अपात्र रहते हैं, जब तक उनका स्वामी `removeAfter` तारीख प्रकाशित नहीं करता।                          |
| **प्रत्येक संगतता रिकॉर्ड की `removeAfter` तारीख** | वह विशिष्ट सतह हटाने के लिए पात्र हो जाती है; तारीख बीतने के बाद `pnpm plugins:boundary-report --fail-on-eligible-compat` CI को विफल कर देता है।    |
| **अगला प्रमुख रिलीज़**                      | तारीख वाली सतहें केवल उनकी `removeAfter` तारीख के बाद हटाई जा सकती हैं; बिना तारीख वाले रिकॉर्ड के लिए अब भी स्वामी की स्वीकृति और प्रकाशित तारीख आवश्यक है।   |

नीचे दिए गए शेष सार्वजनिक SDK उपपथों के लिए रजिस्ट्री-समर्थित निष्कासन अवधियाँ हैं।
30 जुलाई वाली पंक्तियाँ, अनुरक्षकों द्वारा अधिकृत उनकी शुरुआती समीक्षा के बाद हटा दी गईं:
अप्रयुक्त उपपथ हटा दिए गए, पहले के संगतता उपनाम हटा दिए गए, और
केवल-बंडल मॉड्यूल को निजी-स्थानीय बिल्ड मैपिंग में अवनत कर दिया गया।

| `removeAfter` | स्तर                               | SDK उपपथ                                                                                                                                                                        |
| ------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `2026-08-15`  | पहले के संगतता अप्रचलन | `agent-config-primitives`, `channel-logging`, `channel-secret-runtime`, `channel-streaming`, `group-access`, `inbound-reply-dispatch`, `matrix`, `text-runtime`, `zod`              |
| `2026-09-01`  | पहले के संगतता अप्रचलन | `channel-lifecycle`, `channel-message`, `channel-reply-pipeline`, `config-runtime`, `infra-runtime`                                                                                 |
| `2026-10-01`  | मीडिया लेगेसी प्रोजेक्शन            | `agent-media-payload`, साथ ही गैर-उपपथ `MsgContext Media*` फ़ील्ड, चैनल इनबाउंड मीडिया पेलोड बिल्डर, `buildMediaPayload`, हुक मीडिया उपनाम और `{{Media*}}` टेम्पलेट |

सभी कोर plugins पहले ही माइग्रेट हो चुके हैं। बाहरी plugins को
अगले प्रमुख रिलीज़ से पहले माइग्रेट करना चाहिए। आपका plugin जिन सतहों का उपयोग करता है, उनके
संगतता रिकॉर्ड में से कौन-से सबसे जल्द देय हैं, यह देखने के लिए `pnpm plugins:boundary-report` चलाएँ।

## चेतावनियों को अस्थायी रूप से दबाना

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

यह एक अस्थायी बचाव-मार्ग है, स्थायी समाधान नहीं।

## संबंधित

- [शुरुआत करें](/hi/plugins/building-plugins) - अपना पहला plugin बनाएँ
- [SDK अवलोकन](/hi/plugins/sdk-overview) - संपूर्ण उपपथ इंपोर्ट संदर्भ
- [चैनल Plugins](/hi/plugins/sdk-channel-plugins) - चैनल plugins बनाना
- [प्रदाता Plugins](/hi/plugins/sdk-provider-plugins) - प्रदाता plugins बनाना
- [Plugin की आंतरिक संरचना](/hi/plugins/architecture) - आर्किटेक्चर का गहन विश्लेषण
- [Plugin मैनिफ़ेस्ट](/hi/plugins/manifest) - मैनिफ़ेस्ट स्कीमा संदर्भ
