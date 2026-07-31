---
read_when:
    - आप एम्बेडेड एजेंट रनटाइम या हार्नेस रजिस्ट्री बदल रहे हैं
    - आप किसी बंडल किए गए या विश्वसनीय Plugin से एजेंट हार्नेस पंजीकृत कर रहे हैं
    - आपको यह समझना होगा कि Codex Plugin का मॉडल प्रदाताओं से क्या संबंध है
sidebarTitle: Agent Harness
summary: निम्न-स्तरीय एम्बेडेड एजेंट एक्ज़ीक्यूटर को प्रतिस्थापित करने वाले plugins के लिए प्रयोगात्मक SDK सतह
title: एजेंट हार्नेस Plugin्स
x-i18n:
    generated_at: "2026-07-27T21:31:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ff4e41a46ba0074fc6c8bf46da813b58d074f5e6c5c1d236d7ab78e824bdc02
    source_path: plugins/sdk-agent-harness.md
    workflow: 16
---

एक **एजेंट हार्नेस** एक तैयार OpenClaw एजेंट टर्न के लिए निम्न-स्तरीय निष्पादक है।
यह मॉडल प्रदाता, चैनल या टूल रजिस्ट्री नहीं है। उपयोगकर्ता-सामना मानसिक मॉडल के लिए,
[एजेंट रनटाइम](/hi/concepts/agent-runtimes) देखें।

इस सतह का उपयोग केवल बंडल किए गए या विश्वसनीय नेटिव plugins के लिए करें। यह अनुबंध
अभी भी प्रयोगात्मक है, क्योंकि पैरामीटर प्रकार जानबूझकर वर्तमान एम्बेडेड रनर को
प्रतिबिंबित करते हैं।

## हार्नेस का उपयोग कब करें

जब किसी मॉडल फ़ैमिली का अपना नेटिव सत्र रनटाइम हो और सामान्य OpenClaw प्रदाता
ट्रांसपोर्ट गलत अमूर्तन हो, तब एजेंट हार्नेस पंजीकृत करें:

- एक नेटिव कोडिंग-एजेंट सर्वर जो थ्रेड और Compaction का स्वामी हो
- एक स्थानीय CLI या डेमन जिसे नेटिव योजना/रीज़निंग/टूल इवेंट स्ट्रीम करने हों
- एक मॉडल रनटाइम जिसे OpenClaw सत्र ट्रांसक्रिप्ट के अतिरिक्त अपनी स्वयं की रिज़्यूम आईडी चाहिए

केवल नया LLM API जोड़ने के लिए हार्नेस पंजीकृत **न करें**। सामान्य HTTP या
WebSocket मॉडल API के लिए, एक [प्रदाता Plugin](/hi/plugins/sdk-provider-plugins) बनाएँ।

## कोर अब भी किनका स्वामी है

हार्नेस चुने जाने से पहले, OpenClaw पहले ही निम्न को हल कर चुका होता है:

- प्रदाता और मॉडल
- रनटाइम प्रमाणीकरण स्थिति, जब तक हार्नेस यह घोषित न करे कि प्रमाणीकरण बूटस्ट्रैप का स्वामी वही है
- विचार स्तर और संदर्भ बजट
- OpenClaw ट्रांसक्रिप्ट/सत्र फ़ाइल
- वर्कस्पेस, सैंडबॉक्स और टूल नीति
- चैनल उत्तर कॉलबैक और स्ट्रीमिंग कॉलबैक
- मॉडल फ़ॉलबैक और लाइव मॉडल स्विचिंग नीति

हार्नेस एक तैयार प्रयास चलाता है; वह प्रदाता नहीं चुनता, चैनल डिलीवरी को
प्रतिस्थापित नहीं करता और चुपचाप मॉडल नहीं बदलता।

### हार्नेस-स्वामित्व वाला प्रमाणीकरण बूटस्ट्रैप

डिफ़ॉल्ट रूप से, कोर हार्नेस को कॉल करने से पहले प्रदाता क्रेडेंशियल हल करता है। ऐसा
विश्वसनीय हार्नेस, जो अपने नेटिव रनटाइम के माध्यम से प्रमाणीकरण कर सकता है, अपनी स्थिर
`AgentHarness` पंजीकरण पर `authBootstrap: "harness"` सेट कर सकता है। इसके बाद कोर
उस हार्नेस द्वारा दावा किए गए प्रत्येक प्रयास के लिए अपना सामान्य प्रदाता क्रेडेंशियल
बूटस्ट्रैप और अनुपलब्ध-क्रेडेंशियल विफलता छोड़ देता है।

जहाँ कोई संगत, स्पष्ट रूप से चयनित या क्रमबद्ध OpenClaw प्रमाणीकरण प्रोफ़ाइल और उसका
स्कोप किया गया स्टोर मौजूद हो, वहाँ कोर उन्हें अब भी अग्रेषित करता है। मॉडल अनुरोध
जारी करने से पहले हार्नेस को उस प्रोफ़ाइल या अपने नेटिव क्रेडेंशियल को हल करना होगा,
रहस्यों को प्रयास के दायरे में रखना होगा और कार्रवाई-योग्य प्रमाणीकरण विफलताएँ दिखानी
होंगी। इस क्षमता को ऐसे हार्नेस पर सेट न करें जो केवल कभी-कभी प्रमाणीकरण का स्वामी होता है।

### सत्यापित सेटअप रनटाइम आर्टिफ़ैक्ट

पहले-रन सेटअप के लिए इनफ़रेंस प्रदान कर सकने वाले स्थानीय हार्नेस को उस कार्यान्वयन का
सत्यापन करना होगा जिसने प्रोब पूरा किया। जब
`params.captureRuntimeArtifact` सत्य हो, तो स्थिर आईडी और सामग्री फ़िंगरप्रिंट वाला एक अपारदर्शी
`result.runtimeArtifact` लौटाएँ। एक संगत `runtimeArtifact.validate(...)` क्षमता पंजीकृत करें, जो
किसी अलग हार्नेस को लोड किए या असंबंधित plugins को स्कैन किए बिना उस बाइंडिंग की
दोबारा जाँच करे।

सत्यापित OpenClaw निरंतरताएँ `params.expectedRuntimeArtifact` भी पास करती हैं।
हार्नेस को इसकी तुलना अपने द्वारा प्राप्त सटीक नेटिव प्रक्रिया से करनी होगी और यदि वे
अलग हों, तो नेटिव थ्रेड शुरू या रिज़्यूम करने से पहले विफल होना होगा। सामान्य एजेंट
टर्न दोनों फ़ील्ड छोड़ देते हैं, इसलिए सामग्री हैशिंग सामान्य अनुरोध हॉट पाथ से बाहर
रहती है। Remote/WebSocket हार्नेस को भाग लेने से पहले सर्वर सत्यापन अनुबंध चाहिए;
केवल संस्करण स्ट्रिंग आर्टिफ़ैक्ट पहचान नहीं है।

तैयार प्रयास में `params.runtimePlan` भी शामिल होता है, जो उन रनटाइम निर्णयों के लिए
OpenClaw-स्वामित्व वाला नीति बंडल है जिन्हें OpenClaw और नेटिव हार्नेस के बीच साझा
रहना चाहिए:

- `runtimePlan.tools.normalize(...)` और `runtimePlan.tools.logDiagnostics(...)`
  प्रदाता-जागरूक टूल स्कीमा नीति के लिए
- ट्रांसक्रिप्ट सैनिटाइज़ेशन और
  टूल-कॉल सुधार नीति के लिए `runtimePlan.transcript.resolvePolicy(...)`
- साझा `NO_REPLY` और मीडिया
  डिलीवरी दमन के लिए `runtimePlan.delivery.isSilentPayload(...)`
- मॉडल फ़ॉलबैक
  वर्गीकरण के लिए `runtimePlan.outcome.classifyRunResult(...)`
- हल किए गए प्रदाता/मॉडल/हार्नेस मेटाडेटा के लिए `runtimePlan.observability`

हार्नेस उन निर्णयों के लिए योजना का उपयोग कर सकते हैं जिन्हें OpenClaw व्यवहार से मेल
खाना चाहिए, लेकिन इसे होस्ट-स्वामित्व वाली प्रयास स्थिति मानें: इसे बदलें नहीं और किसी
टर्न के भीतर प्रदाता/मॉडल बदलने के लिए इसका उपयोग न करें।

### अनुरोध-ट्रांसपोर्ट अनुबंध

`supports(ctx)`, `ctx.modelProvider` में हल किया गया मॉडल ट्रांसपोर्ट प्राप्त करता है।
प्रदाता-स्वामित्व वाले दो रहस्य-मुक्त तथ्य चयनित रूट का वर्णन करते हैं:

- `runtimePolicy.compatibleIds` उन रनटाइम आईडी को सूचीबद्ध करता है जिन्हें प्रदाता
  उस ठोस रूट के साथ संगत घोषित करता है। अनुपस्थित नीति का अर्थ है कि प्रदाता ने
  रूट-स्तरीय संगतता घोषित नहीं की; यह समर्थन मान लेने की अनुमति नहीं है।
- `requestTransportOverrides: "none"` का अर्थ है कि किसी लिखित प्रदाता/मॉडल अनुरोध
  ओवरराइड को पुनरुत्पादित नहीं करना है। `"present"` का अर्थ है कि लिखित हेडर,
  प्रमाणीकरण ट्रांसपोर्ट, प्रॉक्सी, TLS, स्थानीय-सेवा, निजी-नेटवर्क व्यवहार या अनुरोध
  पैरामीटर मौजूद हैं। यह तथ्य उन मानों को उजागर नहीं करता।

जब हार्नेस तैयार ट्रांसपोर्ट को पुनरुत्पादित न कर सके, तब `{ supported: false, reason }` लौटाएँ।
चयन के बाद रॉ कॉन्फ़िग पढ़कर समर्थन का अनुमान न लगाएँ। जब प्रमाणीकरण तैयारी से कई
पुनःप्रयास रूट मिलें, तो डिस्पैच से पहले एक हार्नेस को उन सभी का समर्थन करना होगा।
यदि कोई Plugin पूरे सेट का स्वामी नहीं बन सकता, तो अंतर्निहित चयन OpenClaw का उपयोग
करता है; स्पष्ट या स्थायी Plugin चयन सुरक्षित रूप से विफल होता है।

## हार्नेस पंजीकृत करें

**इम्पोर्ट:** `openclaw/plugin-sdk/agent-harness`

```typescript
import type { AgentHarness } from "openclaw/plugin-sdk/agent-harness";
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const myHarness: AgentHarness = {
  id: "my-harness",
  label: "मेरा नेटिव एजेंट हार्नेस",

  supports(ctx) {
    const routeSupportsHarness =
      ctx.modelProvider?.runtimePolicy?.compatibleIds.includes("my-harness") === true;
    const canReproduceRequest = ctx.modelProvider?.requestTransportOverrides !== "present";
    return ctx.provider === "my-provider" && routeSupportsHarness && canReproduceRequest
      ? { supported: true, priority: 100 }
      : { supported: false, reason: "प्रभावी रूट हार्नेस-संगत नहीं है" };
  },

  async runAttempt(params) {
    // अपना नेटिव थ्रेड शुरू या रिज़्यूम करें।
    // params.prompt, params.tools, params.images, params.onPartialReply,
    // params.onAgentEvent और अन्य तैयार प्रयास फ़ील्ड का उपयोग करें।
    return await runMyNativeTurn(params);
  },
};

export default definePluginEntry({
  id: "my-native-agent",
  name: "मेरा नेटिव एजेंट",
  description: "चयनित मॉडल को नेटिव एजेंट डेमन के माध्यम से चलाता है।",
  register(api) {
    api.registerAgentHarness(myHarness);
  },
});
```

इस सामान्य उदाहरण में `authBootstrap` जानबूझकर अनुपस्थित है। केवल तभी
`authBootstrap: "harness"` जोड़ें जब हार्नेस ऊपर दिए गए अनुबंध को पूरा करता हो।

### प्रत्यायोजित निष्पादन

हार्नेस स्वामी `delegatedExecutionPluginIds` को उन विश्वसनीय plugins की आईडी पर सेट कर सकता है
जिन्हें किसी मौजूदा मॉडल-लॉक सत्र को निष्पादित करना हो, जैसे Codex-समर्थित बातचीत जारी
रखने वाला वॉइस ट्रांसपोर्ट। यह स्थिर स्वामी सहमति है, कोर अनुमति-सूची नहीं। इसे सीमित रखें।

प्रतिनिधियों को केवल कार्य प्रवेश और एम्बेडेड निष्पादन मिलता है। OpenClaw को सटीक
संग्रहीत सत्र कुंजी, स्टोर पथ और सत्र आईडी; `modelSelectionLocked:
true`; तथा मेल खाते
`agentHarnessId` और `agentHarnessRuntimeOverride` मान चाहिए।
इसके बाद रन को हार्नेस स्वामी के माध्यम से स्कोप किया जाता है। सत्र निर्माण, पैचिंग,
रीसेट, हटाना, आर्काइव और Gateway म्यूटेशन केवल स्वामी के अधिकार में रहते हैं।

## चयन नीति

OpenClaw प्रदाता/मॉडल समाधान के बाद हार्नेस चुनता है:

1. मॉडल-स्कोप रनटाइम नीति को प्राथमिकता मिलती है।
2. इसके बाद प्रदाता-स्कोप रनटाइम नीति आती है।
3. `auto` पंजीकृत हार्नेस से पूछता है कि क्या वे हल किए गए प्रभावी
   रूट का समर्थन करते हैं। केवल प्रदाता/मॉडल प्रीफ़िक्स कभी हार्नेस नहीं चुनते।
4. यदि कोई पंजीकृत हार्नेस मेल नहीं खाता, तो OpenClaw अपने एम्बेडेड रनटाइम का उपयोग करता है।

Plugin हार्नेस विफलताएँ रन विफलताओं के रूप में दिखाई देती हैं। `auto` मोड
में, एम्बेडेड फ़ॉलबैक केवल तभी लागू होता है जब कोई पंजीकृत Plugin हार्नेस हल किए गए
प्रदाता/मॉडल का समर्थन न करे। किसी Plugin हार्नेस द्वारा रन का दावा कर लेने के बाद,
OpenClaw उसी टर्न को किसी अन्य रनटाइम से दोबारा नहीं चलाता, क्योंकि इससे प्रमाणीकरण/
रनटाइम अर्थ बदल सकते हैं या साइड इफ़ेक्ट दोहराए जा सकते हैं।

कॉन्फ़िगर की गई रनटाइम नीति वांछित रनटाइम के संबंध में प्रामाणिक बनी रहती है। स्थायी
सत्र `agentHarnessId` अपने नेटिव ट्रांसक्रिप्ट का स्वामित्व बनाए रखता है, जबकि
रूट/प्रमाणीकरण तैयारी अभी लंबित होती है। इनमें से कोई भी असंगत रूट को संगत नहीं बनाता:
तैयार तथ्य उपलब्ध होने पर चयनित या पिन किया गया हार्नेस उनका समर्थन करे, अन्यथा रन
सुरक्षित रूप से विफल हो जाता है। `/status` नीति, स्थायी स्वामित्व और रूट
समर्थन से चयनित प्रभावी रनटाइम दिखाता है।
तैयार स्थिति स्पष्ट होती है: अनुपस्थित `runtimePolicy` को मौजूद ट्रांसपोर्ट फ़ील्ड
से अनुमानित करने के बजाय अघोषित रखा जाता है।
जब हार्नेस-स्वामित्व वाला प्रमाणीकरण कई भौतिक रूट को अनसुलझा छोड़ देता है, तो तैयार
समर्थन तथ्य उनकी संगत रनटाइम आईडी का प्रतिच्छेद होता है और यदि किसी उम्मीदवार में
अनुरोध ओवरराइड हों, तो उनकी रिपोर्ट करता है। इसलिए एक अघोषित उम्मीदवार नेटिव संगतता
को रिक्त कर देता है; `preparedAuth.source: "harness"` प्रमाणीकरण स्वामी है, रूट समर्थन का अनुमान
लगाने की अनुमति नहीं।

यदि चयनित हार्नेस अप्रत्याशित हो, तो `agents/harness` डीबग लॉगिंग सक्षम करें और
Gateway का संरचित `agent harness selected` रिकॉर्ड जाँचें: इसमें चयनित हार्नेस आईडी,
चयन कारण, रनटाइम/फ़ॉलबैक नीति और `auto` मोड में प्रत्येक Plugin
उम्मीदवार का समर्थन परिणाम शामिल होता है।

बंडल किया गया Codex Plugin अपनी हार्नेस आईडी के रूप में `codex` पंजीकृत
करता है। कोर इसे सामान्य Plugin हार्नेस आईडी मानता है; Codex-विशिष्ट उपनाम साझा
रनटाइम चयनकर्ता में नहीं, बल्कि Plugin या ऑपरेटर कॉन्फ़िग में होने चाहिए।

## प्रदाता और हार्नेस की जोड़ी

अधिकांश हार्नेस को एक प्रदाता भी पंजीकृत करना चाहिए। प्रदाता मॉडल संदर्भ, प्रमाणीकरण
स्थिति, मॉडल मेटाडेटा और `/model` चयन को शेष OpenClaw के लिए दृश्यमान
बनाता है। इसके बाद हार्नेस `supports(...)` में उस प्रदाता का दावा करता है।

बंडल किया गया Codex Plugin इस पैटर्न का पालन करता है:

- पसंदीदा उपयोगकर्ता मॉडल संदर्भ: `openai/gpt-5.6-sol`
- संगतता संदर्भ: पुराने `codex/gpt-*` संदर्भ अब भी स्वीकार किए जाते हैं, लेकिन नए
  कॉन्फ़िग को सामान्य प्रदाता/मॉडल संदर्भ के रूप में उनका उपयोग नहीं करना चाहिए
- हार्नेस आईडी: `codex`
- प्रमाणीकरण: कृत्रिम प्रदाता उपलब्धता, क्योंकि Codex हार्नेस नेटिव
  Codex लॉगिन/सत्र का स्वामी है
- ऐप-सर्वर अनुरोध: OpenClaw Codex को केवल मॉडल आईडी भेजता है और
  हार्नेस को नेटिव ऐप-सर्वर प्रोटोकॉल से संवाद करने देता है

Codex Plugin योगात्मक है। रनटाइम नीति अनसेट या `auto` होने पर, OpenAI
Codex को केवल तभी चुन सकता है जब उसका प्रदाता-स्वामित्व वाला रूट अनुबंध
`codex` को संगत घोषित करे: बिना किसी लिखित अनुरोध ओवरराइड के सटीक आधिकारिक
HTTPS Platform Responses या ChatGPT Responses रूट। केवल `openai/*` प्रीफ़िक्स
कभी Codex का चयन नहीं करता। कस्टम एंडपॉइंट, Completions अडैप्टर और लिखित अनुरोध
व्यवहार OpenClaw पर बने रहते हैं। प्लेनटेक्स्ट आधिकारिक HTTP एंडपॉइंट अस्वीकार किए जाते
हैं। पुराने `codex/gpt-*` संदर्भ संगतता इनपुट बने रहते हैं। देखें
[OpenAI अंतर्निहित एजेंट रनटाइम](/hi/providers/openai#implicit-agent-runtime)।

ऑपरेटर सेटअप, मॉडल प्रीफ़िक्स उदाहरणों और केवल-Codex कॉन्फ़िग के लिए,
[Codex हार्नेस](/hi/plugins/codex-harness) देखें।

Codex Plugin [Codex हार्नेस](/hi/plugins/codex-harness) में प्रलेखित न्यूनतम ऐप-सर्वर
संस्करण लागू करता है। यह इनिशियलाइज़ हैंडशेक की जाँच करता है और पुराने या संस्करण-रहित
सर्वर को अवरुद्ध करता है, ताकि OpenClaw केवल उस प्रोटोकॉल सतह के विरुद्ध चले जिसका
उसने परीक्षण किया है।

### टूल-परिणाम मिडलवेयर

बंडल किए गए plugins और स्पष्ट रूप से सक्षम ऐसे इंस्टॉल किए गए plugins, जिनके
मैनिफ़ेस्ट अनुबंध मेल खाते हों, `api.registerAgentToolResultMiddleware(...)` के माध्यम से रनटाइम-निरपेक्ष
टूल-परिणाम मिडलवेयर संलग्न कर सकते हैं, जब उनका मैनिफ़ेस्ट `contracts.agentToolResultMiddleware` में
लक्षित रनटाइम आईडी घोषित करता हो। यह विश्वसनीय सीम उन असिंक्रोनस टूल-परिणाम
रूपांतरणों के लिए है जिन्हें OpenClaw या Codex द्वारा टूल आउटपुट वापस मॉडल को देने से
पहले चलना आवश्यक है।

पुराने बंडल किए गए plugins अब भी Codex app-server-केवल
middleware के लिए `api.registerCodexAppServerExtensionFactory(...)` का उपयोग कर सकते हैं, लेकिन नए परिणाम रूपांतरणों को runtime-निरपेक्ष API का उपयोग करना चाहिए। केवल
embedded-runner वाला `api.registerEmbeddedExtensionFactory(...)` hook हटा दिया गया है;
embedded tool-result रूपांतरणों को runtime-निरपेक्ष middleware का उपयोग करना होगा।

### टर्मिनल परिणाम वर्गीकरण

अपने स्वयं के protocol projection का स्वामित्व रखने वाले native harnesses,
जब कोई पूर्ण turn कोई दृश्यमान assistant text उत्पन्न न करे, तब
`openclaw/plugin-sdk/agent-harness-runtime` से
`classifyAgentHarnessTerminalOutcome(...)` का उपयोग कर सकते हैं। यह helper `empty`, `reasoning-only`, या
`planning-only` लौटाता है, ताकि OpenClaw की fallback policy यह तय कर सके कि किसी
भिन्न model पर पुनः प्रयास करना है या नहीं। `planning-only` के लिए harness का स्पष्ट `planText`
field आवश्यक है; OpenClaw assistant prose से इसका अनुमान नहीं लगाता। यह helper
जानबूझकर prompt errors, प्रगति पर मौजूद turns, और `NO_REPLY` जैसे जानबूझकर मौन
replies को अवर्गीकृत छोड़ता है।

### Agent-end दुष्प्रभाव

Native harnesses को attempt को अंतिम रूप देने के बाद
`openclaw/plugin-sdk/agent-harness-runtime` से `runAgentEndSideEffects(...)` को कॉल करना होगा। यह
interactive replies में विलंब किए बिना पोर्टेबल `agent_end` hook और OpenClaw के research capture
को dispatch करता है। स्थानीय, non-interactive runs के लिए
`awaitAgentEndSideEffects(...)` का उपयोग करें, जहाँ उन दुष्प्रभावों के पूर्ण होने तक attempt का समाधान नहीं होना चाहिए।
दोनों helpers, `runAgentHarnessAgentEndHook(...)` के समान `{ event, ctx }` payload स्वीकार करते हैं;
उनकी विफलताएँ पूर्ण किए गए attempt के परिणाम को नहीं बदलतीं।

### उपयोगकर्ता इनपुट और tool surfaces

Runtime-स्तरीय user-input request उजागर करने वाले native harnesses को
prompt format करने, उसे OpenClaw के blocking reply path के माध्यम से भेजने, और
choice/free-form उत्तरों को वापस runtime के native response shape में normalize करने के लिए
`openclaw/plugin-sdk/agent-harness-runtime` के user-input helpers का उपयोग करना चाहिए। यह
helper channel/TUI प्रस्तुति को एकरूप रखता है, जबकि प्रत्येक harness अपने
protocol parsing और pending-request lifecycle का स्वामित्व बनाए रखता है।

PI-जैसी compact tool routing की आवश्यकता वाले native harnesses को
`openclaw/plugin-sdk/agent-harness-tool-runtime` से
`createAgentHarnessToolSurfaceRuntime(...)` का उपयोग करना चाहिए। यह
tool-search/code-mode control selection, local-model lean defaults,
runtime-संगत schema filtering, छिपे हुए catalog execution, directory
hydration, और catalog cleanup का स्वामित्व रखता है। Harnesses अब भी अपने SDK-विशिष्ट tool
conversion और native execution callback का स्वामित्व रखते हैं।

### Native Codex harness mode

बंडल किया गया `codex` harness, embedded OpenClaw
agent turns के लिए native Codex mode है। पहले बंडल किया गया `codex` plugin सक्षम करें, और यदि आपकी config
प्रतिबंधात्मक allowlist का उपयोग करती है, तो `plugins.allow` में `codex` शामिल करें। Native app-server
configs को `openai/gpt-*` का उपयोग करना चाहिए; OpenAI agent turns Codex harness
केवल तभी चुनते हैं, जब प्रभावी route Codex संगतता घोषित करता है। पुराने Codex model
refs की मरम्मत `openclaw doctor --fix` से की जानी चाहिए, और पुराने `codex/*`
model refs native harness के लिए संगतता aliases बने रहते हैं।

जब यह mode चलता है, Codex native thread id, resume behavior,
Compaction, और app-server execution का स्वामित्व रखता है। OpenClaw अब भी chat channel,
दृश्यमान transcript mirror, tool policy, approvals, media delivery, और session
selection का स्वामित्व रखता है। जब आपको यह सिद्ध करना हो कि केवल Codex app-server path ही run
का दावा कर सकता है, तब provider/model `agentRuntime.id: "codex"` का उपयोग करें। स्पष्ट plugin
runtimes fail closed करते हैं; Codex app-server selection failures और runtime failures
का किसी अन्य runtime के माध्यम से पुनः प्रयास नहीं किया जाता।

## Runtime की कठोरता

डिफ़ॉल्ट रूप से, OpenClaw `auto` provider/model runtime policy का उपयोग करता है: पंजीकृत
plugin harnesses संगत प्रभावी routes का दावा कर सकते हैं, और कोई भी मेल न खाने पर embedded
runtime turn संभालता है। केवल provider/model prefix कभी भी
harness नहीं चुनता। जब अनुपलब्ध harness selection को embedded runtime से route होने के बजाय
विफल होना चाहिए, तब `agentRuntime.id: "codex"` जैसे स्पष्ट provider/model plugin runtime का उपयोग करें।
स्पष्ट selection किसी असंगत route को संगत नहीं बनाता। चुने गए plugin harness की विफलताएँ हमेशा
कठोर रूप से विफल होती हैं। यह स्पष्ट provider/model
`agentRuntime.id: "openclaw"` को अवरुद्ध नहीं करता।

केवल Codex वाले embedded runs के लिए:

```json
{
  "models": {
    "providers": {
      "openai": {
        "agentRuntime": {
          "id": "codex"
        }
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "openai/gpt-5.6-sol"
    }
  }
}
```

यदि आप एक canonical model के लिए CLI backend चाहते हैं, तो runtime को उस
model entry पर रखें:

```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-5",
      "models": {
        "anthropic/claude-opus-5": {
          "agentRuntime": {
            "id": "claude-cli"
          }
        }
      }
    }
  }
}
```

प्रति-agent overrides समान model-scoped shape का उपयोग करते हैं:

```json
{
  "agents": {
    "list": [
      {
        "id": "codex-only",
        "model": "openai/gpt-5.6-sol",
        "models": {
          "openai/gpt-5.6-sol": {
            "agentRuntime": { "id": "codex" }
          }
        }
      }
    ]
  }
}
```

इस तरह के पुराने whole-agent runtime उदाहरणों को अनदेखा किया जाता है:

```json
{
  "agents": {
    "defaults": {
      "agentRuntime": {
        "id": "codex"
      }
    }
  }
}
```

स्पष्ट plugin runtime के साथ, जब अनुरोधित
harness पंजीकृत न हो, resolved provider/model का समर्थन न करता हो, या
turn के दुष्प्रभाव उत्पन्न करने से पहले विफल हो जाए, तो session जल्दी विफल हो जाता है। यह केवल Codex वाले
deployments और उन live tests के लिए जानबूझकर किया गया है, जिन्हें यह सिद्ध करना होता है कि Codex app-server path
वास्तव में उपयोग में है।

यह setting केवल embedded agent harness को नियंत्रित करती है। यह
image, video, music, TTS, PDF, या अन्य provider-विशिष्ट model routing को अक्षम नहीं करती।

## Native sessions और transcript mirror

कोई harness native session id, thread id, या daemon-side resume
token रख सकता है। उस binding को OpenClaw session से स्पष्ट रूप से संबद्ध रखें, और
उपयोगकर्ता को दिखाई देने वाले assistant/tool output को OpenClaw
transcript में mirror करते रहें।

OpenClaw transcript निम्न के लिए संगतता परत बना रहता है:

- channel पर दिखाई देने वाला session इतिहास
- transcript खोज और indexing
- बाद के turn पर अंतर्निर्मित OpenClaw harness पर वापस जाना
- सामान्य `/new`, `/reset`, और session deletion व्यवहार

यदि आपका harness sidecar binding संग्रहीत करता है, तो `reset(...)` लागू करें, ताकि स्वामित्व वाला
OpenClaw session reset होने पर OpenClaw उसे साफ़ कर सके।

## Tool और media परिणाम

Core OpenClaw tool list बनाता है और उसे तैयार
attempt में भेजता है। जब कोई harness dynamic tool call निष्पादित करता है, तो channel media
स्वयं भेजने के बजाय tool result को harness result shape के माध्यम से वापस लौटाएँ।

इससे text, image, video, music, TTS, approval, और messaging-tool
outputs, OpenClaw-backed runs के समान delivery path पर बने रहते हैं।

`AgentHarnessAttemptResult.hostOwnedToolMediaUrls` केवल उन native artifacts के लिए सेट करें,
जिन्हें विश्वसनीय harness runtime ने स्वयं बनाया और persist किया हो। प्रत्येक entry
`toolMediaUrls` में भी दिखाई देनी चाहिए। Model द्वारा चुने गए dynamic-tool या
OpenClaw-tool media को कभी शामिल न करें। `message_tool_only` routes पर, यह सीमित provenance
native runtime artifacts को source-reply suppression के बावजूद बनाए रखता है; सामान्य send policy
और ambient-room admission अब भी लागू होते हैं।

### टर्मिनल tool परिणाम

`AgentHarnessAttemptParams.observeToolTerminal` host के स्वामित्व वाला terminal
outcome accumulator है। OpenClaw dynamic tools या native
tools निष्पादित करने वाले harness को, प्रत्येक tool के एक terminal outcome तक पहुँचने पर और
attempt result को अंतिम रूप देने से पहले, इसे कॉल करना होगा। Tools निष्पादित न करने वाले harnesses को
इसे कॉल करने की आवश्यकता नहीं है।

Execution boundary से तथ्य report करें:

- Protocol call id उपलब्ध होने पर उसे, canonical tool नाम, और
  preparation या hook rewrites के बाद वास्तव में tool तक पहुँचे arguments पास करें।
- जब validation, approval, या किसी अन्य guard ने
  tool implementation शुरू होने से पहले call रोक दिया हो, तब `executionStarted: false` सेट करें। Dispatch होने की
  संभावना बन जाने के बाद, सावधानीपूर्वक `true` report करें।
- `outcome: "success"` या `outcome: "failure"` report करें। Display text से
  failure का अनुमान लगाने के बजाय runtime से उपलब्ध structured
  failure fields शामिल करें।
- `nativeMutation` का उपयोग केवल उन native tools के लिए करें, जो OpenClaw tool
  definition का उपयोग नहीं करते। वहाँ protocol के स्वामित्व वाले mutation और replay facts दें;
  OpenClaw के mutation classifier को harness में copy न करें।

Callback उस call के लिए canonical resolution लौटाता है। उसके
`lastToolError` को `AgentHarnessAttemptResult` में ले जाएँ और समानांतर
state प्राप्त करने के बजाय harness projection में उसके execution,
arguments, और side-effect facts का उपयोग करें। Host असंबंधित सफल tools के दौरान unresolved
mutating failure को बनाए रखता है और उसे केवल संबंधित action के सफल होने के बाद साफ़ करता है।

पुराने experimental
harnesses के साथ source compatibility के लिए callback वैकल्पिक बना रहता है। वैकल्पिक का अर्थ tools निष्पादित करने वाले harness के लिए
उपेक्षणीय नहीं है: terminal reports के बिना, OpenClaw बाद के tool calls में,
quiet Heartbeat completion सहित, mutating-tool failure की वास्तविकता संरक्षित नहीं रख सकता।

### Settled tool finalization

जब harness प्रत्येक
tool call पूरा कर चुका हो, लेकिन उसका native turn assistant text के बिना समाप्त हुआ हो, तब OpenClaw को एक अंतिम दृश्यमान उत्तर की आवश्यकता हो सकती है। कोई harness
`finalizeSettledTurn({ attempt,
settledAttempt })` लागू करके उस recovery को अपना सकता है।

यह callback एक अलग capability है, कोई अन्य सामान्य attempt नहीं। इसे:

- या तो सटीक प्रतिबंधित native transcript या settled tool-result boundary तक स्थिर किया गया पूर्ण application
  transcript उपयोग करना होगा;
- कोई tools, permission-grant या user-input capabilities, native execution
  hooks, agents, skills, memory, scheduling, extensions, या remote control उजागर नहीं करना होगा;
- केवल host द्वारा प्रदान किया गया finalization prompt भेजना होगा; और
- यदि चुनी गई transcript/isolation strategy उन प्रतिबंधों को लागू नहीं कर सकती, तो
  fail closed करना होगा।

OpenClaw callback को सामान्य
attempt और retry loop के बाहर, terminal sub-operation के रूप में एक बार invoke करता है। कोई failure run को
side-effect-aware incomplete-turn warning के साथ समाप्त कर देता है; वह सामान्य
auth/profile rotation, model fallback, context recovery, Compaction
continuation, या hook-requested revision paths में प्रवेश नहीं कर सकता। Finalization plugin
prompt mutation, `before_agent_run`, LLM input/output, terminal revision, और
`agent_end` hooks को भी छोड़ देता है। Core diagnostics फिर भी operation और उसकी failure को record करते हैं।

Callback सामान्य
attempt result नहीं, बल्कि `AgentHarnessSettledTurnFinalizationResult` लौटाता है। इसके public fields पूर्ण किए गए
assistant message, finalization-call usage, transcript-ownership metadata, और
diagnostic trace तक सीमित हैं। Tool, delivery, media, spawn, lifecycle, replay, session, और
fallback state इस result boundary को पार नहीं कर सकते। अज्ञात fields और assistant
tool calls fail closed करते हैं।

अपने पूर्ण attempt engine का आंतरिक रूप से पुनः उपयोग करने वाला harness लौटने से पहले
`projectSettledTurnFinalizationAttemptResult(...)` को कॉल कर सकता है। यह helper
canonical failure, tool, delivery, replay, और lifecycle evidence को अस्वीकार करता है, फिर
केवल सीमित result project करता है। यह native isolation के बाद defense in depth है,
native capability surface हटाने का विकल्प नहीं।

Projection-backed harness को
`source: "openclaw-transcript"` के साथ `settledAttempt.settledTurnFinalizationContext` पर पूर्ण context रखना होगा।
उसे settled turn mirror होने के बाद active branch capture करनी होगी,
यह सिद्ध करना होगा कि मौजूदा prompt और प्रत्येक मौजूदा tool
call/result उस boundary तक उपस्थित हैं, और attempt लौटाने से पहले परिणामी message
array को स्थिर करना होगा। Finalizer को अनुपलब्ध,
असमर्थित, संदिग्ध, या बहुत बड़े context को अस्वीकार करना होगा। उसे messages truncate नहीं करने चाहिए,
पहले का history हटाना नहीं चाहिए, या इस application transcript को सटीक native
history के रूप में वर्णित नहीं करना चाहिए। एक प्रतिबंधित native session resume करने वाले harnesses को इस
projection field की आवश्यकता नहीं है।

इस callback को best-effort
`disableTools` hint के साथ `runAttempt` कॉल करके लागू न करें। Harness के स्वामी को पूर्ण native
capability boundary लागू करनी होगी। OpenClaw generic fallback प्रदान नहीं करता, क्योंकि वह
यह प्रमाणित नहीं कर सकता कि किसी मनमाने native runtime ने उन प्रतिबंधों का पालन किया।

प्रयोगात्मक तृतीय-पक्ष हार्नेस संगतता के लिए कॉलबैक वैकल्पिक बना रहता है। जब चयनित हार्नेस इसे छोड़ देता है, तो OpenClaw बार-बार होने वाले दुष्प्रभावों का जोखिम लेने के बजाय मौजूदा अपूर्ण-टर्न त्रुटि को बनाए रखता है।

## वर्तमान सीमाएँ

- सार्वजनिक इंपोर्ट पथ सामान्य है, लेकिन संगतता के लिए कुछ प्रयास/परिणाम प्रकार उपनामों में अभी भी पुराने नाम मौजूद हैं।
- तृतीय-पक्ष हार्नेस इंस्टॉलेशन प्रयोगात्मक है। जब तक आपको नेटिव सत्र रनटाइम की आवश्यकता न हो, प्रदाता Plugins को प्राथमिकता दें।
- टर्न के बीच हार्नेस बदलना समर्थित है। नेटिव टूल, अनुमोदन, सहायक टेक्स्ट या संदेश भेजना शुरू होने के बाद, किसी टर्न के बीच में हार्नेस न बदलें।

## संबंधित

- [SDK का अवलोकन](/hi/plugins/sdk-overview)
- [रनटाइम सहायक](/hi/plugins/sdk-runtime)
- [प्रदाता Plugins](/hi/plugins/sdk-provider-plugins)
- [Codex हार्नेस](/hi/plugins/codex-harness)
- [मॉडल प्रदाता](/hi/concepts/model-providers)
