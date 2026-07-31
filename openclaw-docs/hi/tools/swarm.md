---
read_when:
    - आप कई एजेंटों के बीच काम वितरित करने के लिए Code Mode स्क्रिप्ट चाहते हैं
    - आपको संरचित चाइल्ड परिणामों, निर्णय गेटों या पहले पूर्ण होने वाली पाइपलाइनों की आवश्यकता है
    - आप `tools.swarm` सीमाएँ सक्षम या समायोजित कर रहे हैं
    - आप सत्र डैशबोर्ड में कलेक्टर चाइल्ड प्रक्रियाओं को देखना चाहते हैं
sidebarTitle: Swarm
summary: संरचित परिणामों, सीमित फ़ैन-आउट और लाइव प्रगति के साथ Code Mode स्क्रिप्ट से समवर्ती उप-एजेंटों का समन्वय करें
title: झुंड
x-i18n:
    generated_at: "2026-07-27T21:49:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0bec17da7a2e144df35189a65d9b35d829815b545a4bb89652e6a681ca971a9
    source_path: tools/swarm.md
    workflow: 16
---

Swarm, [Code Mode](/hi/tools/code-mode) स्क्रिप्ट से कई उप-एजेंटों को व्यवस्थित करने का एक प्रयोगात्मक, ऑप्ट-इन तरीका है। काम को फैलाने, परिणाम एकत्र करने और निर्णय लेने के लिए `Promise.all`, `while`, और `if` जैसे सामान्य JavaScript या TypeScript नियंत्रण प्रवाह का उपयोग करें।

कोई ग्राफ़ DSL और कोई अलग वर्कफ़्लो प्रारूप नहीं है। प्रोग्राम ही व्यवस्थापन है। Swarm उस प्रोग्राम में प्रतीक्षा-योग्य कलेक्टर चाइल्ड, संरचित परिणाम, सीमित समवर्तीता और प्रगति रिपोर्टिंग जोड़ता है।

## Swarm सक्षम करें

अनुशंसित तरीका Control UI में **सेटिंग्स → लैब्स → Swarm** है। टॉगल तुरंत प्रभावी होता है और आपके कॉन्फ़िगरेशन में `tools.swarm.enabled` लिखता है।

आप `openclaw.json` में भी Swarm को सीधे सक्षम कर सकते हैं:

```json5
{
  tools: {
    swarm: {
      enabled: true,
      maxConcurrent: 8,
      maxChildrenPerGroup: 50,
      maxTotalPerGroup: 200,
      waitTimeoutSecondsMax: 600,
      defaultAgentId: "",
    },
  },
}
```

बूलियन संक्षिप्त रूप अन्य सभी मानों को उनके डिफ़ॉल्ट पर रखते हुए सुविधा को सक्षम या अक्षम करता है:

```json5
{
  tools: {
    swarm: true,
  },
}
```

| फ़ील्ड                   | डिफ़ॉल्ट | विवरण                                                                                                                    |
| ----------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `enabled`               | `false` | कलेक्टर-मोड स्पॉन विकल्प, `agents_wait`, और Code Mode `agents.*` गेस्ट API को उपलब्ध कराता है।                                   |
| `maxConcurrent`         | `8`     | एक Swarm समूह में समवर्ती रूप से चलने वाले कलेक्टर चाइल्ड की अधिकतम संख्या। अतिरिक्त स्वीकृत चाइल्ड FIFO क्रम में कतारबद्ध होते हैं।          |
| `maxChildrenPerGroup`   | `50`    | एक समूह में सक्रिय कलेक्टर चाइल्ड की अधिकतम संख्या।                                                                                  |
| `maxTotalPerGroup`      | `200`   | किसी समूह द्वारा अपने जीवनकाल में स्पॉन किए जा सकने वाले कलेक्टर चाइल्ड की अधिकतम संख्या। यह अनियंत्रित स्पॉन के विरुद्ध अंतिम सुरक्षा है।                            |
| `waitTimeoutSecondsMax` | `600`   | एक `agents_wait` कॉल द्वारा स्वीकार की जाने वाली अधिकतम टाइमआउट अवधि। कॉल का डिफ़ॉल्ट 30 सेकंड है।                                            |
| `defaultAgentId`        | `""`    | जब कोई स्पॉन `agentId` छोड़ देता है, तब उपयोग किया जाने वाला लक्ष्य एजेंट। खाली मान अनुरोधकर्ता एजेंट का उपयोग करता है। मौजूदा उप-एजेंट अनुमति-सूचियाँ लागू होती हैं। |

संख्यात्मक मान धनात्मक पूर्णांक होने चाहिए। OpenClaw, `maxConcurrent` को `1`–`1000`, `maxChildrenPerGroup` को `1`–`10000`, `maxTotalPerGroup` को `1`–`100000`, और `waitTimeoutSecondsMax` को `1`–`86400` तक सीमित करता है।

आप `agents.entries.*.tools.swarm` के साथ किसी एक कॉन्फ़िगर किए गए एजेंट के लिए Swarm को ओवरराइड कर सकते हैं। प्रति-एजेंट ऑब्जेक्ट शीर्ष-स्तरीय `tools.swarm` ऑब्जेक्ट के ऊपर मर्ज होता है।

## आवश्यकताएँ

`agents.run`, `phase`, और `log` गेस्ट ग्लोबल्स के लिए Swarm और OpenClaw Code Mode, दोनों आवश्यक हैं:

```json5
{
  tools: {
    codeMode: true,
    swarm: true,
  },
}
```

Code Mode के पास `sessions_spawn` की प्रभावी पहुँच भी होनी चाहिए। टूल प्रोफ़ाइल, अनुमति/निषेध नीति, प्रदाता नियम और सैंडबॉक्स नीति उस टूल को हटा सकते हैं। यदि कोई स्क्रिप्ट रिपोर्ट करती है कि `sessions_spawn` उपलब्ध नहीं है, तो [Code Mode सक्रियण](/hi/tools/code-mode#activation) और [उप-एजेंट](/hi/tools/subagents) देखें।

`defaultAgentId` और प्रति-रन `agentId` मानों को अनुरोधकर्ता की `subagents.allowAgents` नीति द्वारा अनुमत किसी कॉन्फ़िगर किए गए लक्ष्य का नाम देना चाहिए। OpenClaw किसी अज्ञात या अस्वीकृत लक्ष्य के लिए किसी अन्य एजेंट पर वापस जाने के बजाय उसे अस्वीकार करता है।

## Swarm स्क्रिप्ट लिखें

Swarm सक्षम होने पर Code Mode यह गेस्ट API उपलब्ध कराता है:

```typescript
type AgentRunOptions = {
  label?: string;
  model?: string;
  thinking?: string;
  fastMode?: boolean | "auto";
  agentId?: string;
  schema?: Record<string, unknown>;
  phase?: string;
};

agents.run(prompt: string, options?: AgentRunOptions & { schema?: undefined }): Promise<string>;
agents.run<T>(prompt: string, options: AgentRunOptions & { schema: Record<string, unknown> }): Promise<T>;
phase(title: string): void;
log(message: string): void;
```

`schema` के बिना, `agents.run()` चाइल्ड के अंतिम टेक्स्ट में रिज़ॉल्व होता है। JSON Schema के साथ, यह चाइल्ड के `structured_output` टूल के माध्यम से सबमिट किए गए मान में रिज़ॉल्व होता है। विफल, बंद किए गए, टाइमआउट हुए या स्कीमा-अमान्य चाइल्ड का प्रॉमिस `SwarmAgentError` के साथ अस्वीकार होता है। सटीक जनरेट की गई घोषणाएँ और छोटे व्यवस्थापन मुहावरे Code Mode के भीतर `API.read("agents.d.ts")` से पढ़ें।

डैशबोर्ड और साइडबार में पहचानने योग्य चाइल्ड नाम के लिए `label` का उपयोग करें। उस चाइल्ड के शुरू होने से तुरंत पहले कोई चरण प्रकाशित करने के लिए विकल्पों में `phase` का उपयोग करें, या जब कई चाइल्ड एक ही चरण से संबंधित हों, तब `phase()` को कॉल करें। `log()` एक संक्षिप्त प्रगति टिप्पणी प्रकाशित करता है। प्रगति कॉल फायर-एंड-फ़ॉरगेट होते हैं; UI अनुपलब्ध होने पर वे स्क्रिप्ट को विलंबित नहीं करते।

### संरचित परिणामों के साथ समानांतर रूप से काम फैलाएँ

यह उदाहरण प्रत्येक विषय के लिए एक शोधकर्ता शुरू करता है, उन सभी की प्रतीक्षा करता है, फिर एक अंतिम चाइल्ड से उनकी संरचित रिपोर्टों का संश्लेषण करने के लिए कहता है:

```javascript
const reportSchema = {
  type: "object",
  properties: {
    finding: { type: "string" },
    evidence: { type: "array", items: { type: "string" } },
    confidence: { type: "number" },
  },
  required: ["finding", "evidence", "confidence"],
  additionalProperties: false,
};

const topics = ["authentication", "storage", "recovery"];
phase("स्वतंत्र समीक्षा");

const reports = await Promise.all(
  topics.map((topic) =>
    agents.run(`${topic} पथ की समीक्षा करें। साक्ष्य के साथ एक निष्कर्ष लौटाएँ।`, {
      label: `review-${topic}`,
      thinking: "high",
      fastMode: "auto",
      schema: reportSchema,
    }),
  ),
);

phase("संश्लेषण");
log(`${reports.length} स्वतंत्र रिपोर्टें एकत्र की गईं।`);

return await agents.run(
  `इन रिपोर्टों का सामंजस्य करें और असहमतियों को समझाएँ:\n${JSON.stringify(reports)}`,
  { label: "synthesis" },
);
```

`Promise.all` फ़ैन-आउट और फ़ैन-इन सीमा है। OpenClaw समूह के लिए अधिकतम `maxConcurrent` चाइल्ड शुरू करता है और शेष को सबमिशन क्रम में कतारबद्ध करता है।

Code Mode, `tools.codeMode.maxPendingToolCalls` (डिफ़ॉल्ट `16`, अधिकतम `128`) के साथ समवर्ती गेस्ट ब्रिज कॉल को अलग से सीमित करता है। बहुत बड़े समूहों के लिए, उस सीमा से नीचे सीमित बैच शुरू करें और `phase()`, `log()`, तथा चाइल्ड प्रतीक्षा संक्रमणों के लिए अतिरिक्त क्षमता रखें। `maxConcurrent` चल रहे चाइल्ड को सीमित करता है; यह गेस्ट ब्रिज-कॉल सीमा नहीं बढ़ाता।

### निर्णय गेट पर लूप चलाएँ

जब प्रत्येक पास यह तय करता है कि एक और पास की आवश्यकता है या नहीं, तब सीमित `while` लूप का उपयोग करें:

```javascript
const gateSchema = {
  type: "object",
  properties: {
    ready: { type: "boolean" },
    reason: { type: "string" },
    nextAction: { type: "string" },
  },
  required: ["ready", "reason", "nextAction"],
  additionalProperties: false,
};

let pass = 0;
let decision = { ready: false, reason: "जाँच नहीं की गई", nextAction: "समीक्षा करें" };

while (!decision.ready && pass < 4) {
  pass += 1;
  phase(`निर्णय पास ${pass}`);
  decision = await agents.run(
    `जाँचें कि रिलीज़ साक्ष्य पूर्ण है या नहीं। पिछला निर्णय: ${JSON.stringify(decision)}`,
    {
      label: `release-gate-${pass}`,
      schema: gateSchema,
    },
  );
  log(decision.reason);
}

if (!decision.ready) {
  throw new Error(`${pass} पास के बाद भी गेट बंद है: ${decision.nextAction}`);
}

return decision;
```

निर्णय लूप को हमेशा सीमित रखें। `maxTotalPerGroup` अंतिम सुरक्षा उपाय है, स्पष्ट समाप्ति शर्त का विकल्प नहीं।

### सबसे पहले समाप्त होने वाले चाइल्ड को प्रोसेस करें

`agents.run()` एक सामान्य प्रॉमिस लौटाता है, इसलिए `Promise.race` पहले Code Mode चाइल्ड पर प्रतिक्रिया कर सकता है। निम्न-स्तरीय टूल कॉल करने वाले हार्नेस के लिए, `agents_wait` वही प्रथम-पूर्णता सीमा प्रदान करता है: कम-से-कम एक अनुरोधित रन पूरा होते ही, या सीमित टाइमआउट समाप्त होने पर, यह लौट आता है। संपूर्ण ड्रेन लूप के लिए [अन्य हार्नेस से Swarm का उपयोग करें](#use-swarm-from-other-harnesses) देखें।

## कलेक्टर चाइल्ड का व्यवहार

कलेक्टर चाइल्ड अलग पूर्णता पथ वाले सामान्य पृथक उप-एजेंट सत्र होते हैं। वे पैरेंट सत्र में उत्तर की घोषणा करने या उसे निर्देशित करने के बजाय पैरेंट की प्रतीक्षा के लिए एक स्थायी कलेक्टर परिणाम लिखते हैं।

लक्ष्य एजेंट इस क्रम में रिज़ॉल्व होता है:

1. `agentId` स्पॉन या `agents.run()` कॉल पर।
2. `tools.swarm.defaultAgentId`।
3. अनुरोधकर्ता एजेंट।

एक समर्पित, हल्का वर्कर एजेंट तब उपयोगी होता है जब Swarm चाइल्ड को छोटे टूल दायरे, कम लागत वाले मॉडल या अधिक कड़ी सैंडबॉक्स नीति की आवश्यकता हो। OpenClaw अंतर्निहित `worker` एजेंट आईडी के साथ उपलब्ध नहीं होता; उसे डिफ़ॉल्ट नाम देने से पहले कॉन्फ़िगर करें। उस वर्कर को उसके प्रति-एजेंट कॉन्फ़िगरेशन में `tools.swarm: false` के साथ सुदृढ़ करें, ताकि उसे स्पॉन किया जा सके लेकिन वह अपने शीर्ष-स्तरीय सत्रों से Swarm शुरू न कर सके:

```json5
{
  tools: { swarm: { enabled: true, defaultAgentId: "worker" } },
  agents: {
    list: [
      {
        id: "main",
        default: true,
        subagents: { allowAgents: ["worker"] },
      },
      { id: "worker", tools: { swarm: false } },
    ],
  },
}
```

कलेक्टर अनुमोदन सुरक्षित रूप से विफल होते हैं। कोई चाइल्ड कभी ऑपरेटर अनुमोदन प्रॉम्प्ट नहीं खोलता। अनुमोदन की आवश्यकता वाली टूल कार्रवाई अस्वीकार कर दी जाती है, और चाइल्ड उस अस्वीकृति को अपने परिणाम में रिपोर्ट कर सकता है, ताकि स्क्रिप्ट आगे की कार्रवाई तय कर सके।

संरचित आउटपुट के लिए, OpenClaw चाइल्ड में एक कृत्रिम `structured_output` टूल जोड़ता है और दिए गए JSON Schema के विरुद्ध उसके पेलोड को सत्यापित करता है। अमान्य या अनुपस्थित पेलोड को सुधार के लिए एक संकेत मिलता है। यदि पुनः प्रयास भी सत्यापन में विफल रहता है, तो कलेक्टर पूर्णता चाइल्ड का कच्चा टेक्स्ट रखती है, `structured` को अनसेट छोड़ती है और `schemaError` शामिल करती है। निम्न-स्तरीय `agents_wait` परिणाम स्पष्ट पुनर्प्राप्ति तर्क के लिए इन फ़ील्ड को उपलब्ध कराता है।

### चाइल्ड लीफ़ होते हैं

Swarm चाइल्ड डिफ़ॉल्ट रूप से लीफ़ होते हैं। सार्वभौमिक `agents.defaults.subagents.maxSpawnDepth` गार्ड, `1` की डिफ़ॉल्ट गहराई पर किसी चाइल्ड को अपने चाइल्ड स्पॉन करने से रोकता है। सामान्य व्यवस्थापन तरीका किसी चाइल्ड से और काम स्पॉन करना नहीं, बल्कि काम को पैरेंट को लौटाना है:

```javascript
const plan = await agents.run("इस कार्य को स्वतंत्र टास्क के रूप में नियोजित करें।", {
  schema: {
    type: "object",
    properties: { tasks: { type: "array", items: { type: "string" } } },
    required: ["tasks"],
    additionalProperties: false,
  },
});
return await Promise.all(plan.tasks.map((task) => agents.run(task)));
```

नेस्टेड उप-एजेंट `agents.defaults.subagents.maxSpawnDepth` के माध्यम से ऑपरेटर का ऑप्ट-इन हैं और Swarm के लिए हतोत्साहित किए जाते हैं। समूह सीमाएँ, बजट और अवलोकनीयता सभी समतल कलेक्टर समूहों पर आधारित हैं।

प्रत्येक चाइल्ड का एक प्रवेश स्वामी होता है। घोषणा और इंटरैक्टिव चाइल्ड `agents.defaults.subagents.maxChildrenPerAgent` (डिफ़ॉल्ट `5`) का उपयोग करते हैं और कलेक्टर चाइल्ड की गणना नहीं करते। कलेक्टर चाइल्ड केवल `maxChildrenPerGroup` और `maxTotalPerGroup` का उपयोग करते हैं; वे प्रति-सत्र चाइल्ड बजट का उपभोग नहीं करते। स्पॉन गहराई गार्ड फिर भी दोनों मोड पर लागू होता है।

प्रवेश के बाद, `maxConcurrent` से ऊपर के चाइल्ड अपने Swarm समूह के भीतर FIFO क्रम में, वैश्विक उप-एजेंट लेन में नेस्टेड होकर कतारबद्ध होते हैं। ये समवर्तीता परतें काम को अस्वीकार करने के बजाय कतारबद्ध करती हैं। किसी भी समूह सीमा से अधिक होने वाला कलेक्टर स्पॉन, त्रुटि में संबंधित कॉन्फ़िगरेशन कुंजी के साथ अस्वीकार कर दिया जाता है।

## Swarm का अवलोकन करें

Swarm सक्रिय होने के दौरान Control UI में पैरेंट सत्र का डैशबोर्ड खोलें। Swarm विजेट प्रत्येक सक्रिय कलेक्टर समूह को प्रति चाइल्ड एक बिंदु के रूप में दिखाता है, जिसकी अवस्था कतारबद्ध, चल रही, पूर्ण या विफल होती है। लेबल बिंदु टूलटिप में दिखाई देते हैं, इसलिए छोटे और स्थिर लेबल बड़े Swarm को पढ़ना आसान बनाते हैं।

सत्र साइडबार सामान्य पैरेंट/चाइल्ड ट्री को बनाए रखता है। Swarm पदानुक्रम खोए बिना किसी कलेक्टर चाइल्ड का निरीक्षण करने या उसका ट्रांसक्रिप्ट खोलने के लिए पैरेंट पंक्ति विस्तृत करें।

Collector के परिणाम अपने समूह के आर्काइव होने तक प्रतीक्षा-योग्य रहते हैं। प्रत्येक
सदस्य के अपनी प्रतिधारण समय-सीमा पर पहुँचने के बाद, OpenClaw समूह के चाइल्ड को
एक बैच के रूप में आर्काइव करता है, ताकि पूर्ण हो चुके स्वार्म लाइव सत्र ट्री में न रहें।

## अन्य हार्नेस से Swarm का उपयोग करें

आप OpenClaw Code Mode के बिना Swarm का उपयोग कर सकते हैं। इसके मुख्य टूल
हार्नेस-स्वतंत्र हैं: Collector चाइल्ड को
`sessions_spawn({ collect: true })` से शुरू करें और सीमित `agents_wait`
कॉल से उन्हें ड्रेन करें।

Codex Code Mode पात्र डायनेमिक OpenClaw टूल को स्वचालित रूप से
`tools.*` के अंतर्गत उपलब्ध कराता है। यह OpenClaw की QuickJS गेस्ट API का उपयोग नहीं करता या
`tools.codeMode` की आवश्यकता नहीं रखता, लेकिन `tools.swarm` को फिर भी सक्षम होना चाहिए। Codex हार्नेस
`agents_wait` कॉल पूरे 600-सेकंड टाइमआउट का समर्थन करती हैं।

वर्तमान में समर्थित Codex रनटाइम के साथ, डायनेमिक OpenClaw टूल के परिणाम
Code Mode तक JSON टेक्स्ट के रूप में पहुँचते हैं। फ़ील्ड पढ़ने से पहले प्रत्येक परिणाम को पार्स करें। Codex
डायनेमिक टूल कॉल को क्रमिक रूप से निष्पादित भी करता है, इसलिए `Promise.all` कई
`sessions_spawn` कॉल को समवर्ती रूप से सबमिट नहीं करता। Collector को सीमित लूप में लॉन्च करें;
बाद के लॉन्च सबमिट होने के दौरान पहले से स्वीकार किए गए चाइल्ड फिर भी चल सकते हैं।

```javascript
function parseToolResult(value) {
  if (typeof value !== "string") return value;
  return JSON.parse(value);
}

const tasks = [
  "प्रमाणीकरण पथ की जाँच करें।",
  "स्टोरेज पथ की जाँच करें।",
  "रिकवरी पथ की जाँच करें।",
];
const launches = [];

for (const [index, task] of tasks.entries()) {
  const launch = parseToolResult(
    await tools.sessions_spawn({
      task,
      collect: true,
      label: `review-${index + 1}`,
    }),
  );
  if (launch.status !== "accepted") {
    throw new Error(launch.error ?? "Collector स्पॉन स्वीकार नहीं किया गया।");
  }
  launches.push(launch);
}

const pending = new Set(launches.map((launch) => launch.runId));
const completed = [];

while (pending.size > 0) {
  const ids = [...pending].slice(0, 1000);
  const batch = parseToolResult(
    await tools.agents_wait({
      ids,
      timeoutSeconds: 30,
    }),
  );

  // इस सीमित विंडो को उन ids के पीछे घुमाएँ जिनकी अभी तक जाँच नहीं हुई है।
  for (const runId of ids) {
    if (pending.delete(runId)) pending.add(runId);
  }

  for (const item of batch.completed) {
    pending.delete(item.runId);
    if (item.status !== "done") {
      throw new Error(item.schemaError ?? item.result ?? `${item.runId}: ${item.status}`);
    }
    completed.push(item); // प्रत्येक परिणाम के पूर्ण होते ही उसे प्रोसेस करें।
  }

  for (const failure of batch.errors ?? []) {
    pending.delete(failure.runId);
    throw new Error(`${failure.runId}: ${failure.error}`);
  }
}

return completed;
```

प्रत्येक `agents_wait` कॉल 1–1000 रन आईडी स्वीकार करती है। यह लौटाती है:

```typescript
type AgentsWaitResult = {
  completed: Array<{
    runId: string;
    status: "done" | "failed" | "killed" | "timeout";
    result: string;
    structured?: unknown;
    schemaError?: string;
    sessionKey: string;
    label?: string;
    usage?: { inputTokens: number; outputTokens: number };
  }>;
  pending: string[];
  errors?: Array<{
    runId: string;
    error: "not_found" | "not_owner";
  }>;
};
```

जब अनुरोधित कोई चाइल्ड पहले ही पूर्ण हो चुका हो,
जब कम-से-कम एक लंबित चाइल्ड पूर्ण हो जाए, जब कोई वैध लंबित आईडी शेष न रहे,
या जब इसका टाइमआउट समाप्त हो जाए, तब कॉल तुरंत लौटती है। पूर्ण रिकॉर्ड आइडेम्पोटेंट होते हैं, इसलिए
पहले से पूर्ण रन आईडी देने पर उसका परिणाम फिर से लौटता है। केवल स्पॉन करने वाला सत्र
या उसकी अधिकृत पैरेंट शृंखला ही किसी Collector की प्रतीक्षा कर सकती है।

यह सीमित लॉन्ग पोलिंग है, व्यस्त स्टेटस लूप नहीं। केवल शेष
रन आईडी देते रहें, जब तक `pending` खाली न हो जाए। Collector मोड नेटिव
OpenClaw सब-एजेंट का समर्थन करता है; यह ACP रनटाइम, थ्रेड बाइंडिंग, दृश्यमान
सत्रों या स्थायी सत्र मोड का समर्थन नहीं करता।

## सीमाएँ और रोडमैप

Swarm v1 एक-बार चलने वाले Collector चाइल्ड चलाता है; नियोजित `agents.session()` API
स्टेटफुल मल्टी-टर्न वर्कर जोड़ेगी। चाइल्ड वर्तमान में स्थानीय
Gateway की सब-एजेंट लेन पर चलते हैं; क्लाउड प्लेसमेंट को एक स्पष्ट स्पॉन
विकल्प के रूप में नियोजित किया गया है। सहेजी गई वर्कफ़्लो परिभाषाएँ और ग्राफ़ DSL, Swarm की
वर्तमान दिशा का भाग नहीं हैं।

## संबंधित

- [Code Mode](/hi/tools/code-mode) QuickJS गेस्ट रनटाइम और सक्रियण नियमों के लिए
- [सब-एजेंट](/hi/tools/subagents) चाइल्ड नीति, पृथक्करण और सत्र व्यवहार के लिए
- [मल्टी-एजेंट सैंडबॉक्स टूल](/hi/tools/multi-agent-sandbox-tools) प्रति-एजेंट प्रतिबंधों के लिए
- [टूल अवलोकन](/hi/tools) टूल प्रोफ़ाइल और नीति रूटिंग के लिए
