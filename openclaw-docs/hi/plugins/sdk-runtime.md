---
read_when:
    - आपको किसी Plugin से मुख्य सहायक फ़ंक्शन कॉल करने हैं (TTS, STT, इमेज जनरेशन, वेब खोज, Gateway, सबएजेंट, नोड्स)
    - आप समझना चाहते हैं कि api.runtime क्या उपलब्ध कराता है
    - आप Plugin कोड से कॉन्फ़िगरेशन, एजेंट या मीडिया हेल्पर एक्सेस कर रहे हैं
sidebarTitle: Runtime helpers
summary: api.runtime -- plugins के लिए उपलब्ध इंजेक्ट किए गए रनटाइम सहायक फ़ंक्शन
title: Plugin रनटाइम सहायक सुविधाएँ
x-i18n:
    generated_at: "2026-07-27T19:41:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ff1d901de8ec70011eeaafbab7b3cc30709fc95894c7ba4f4346c026de682cd0
    source_path: plugins/sdk-runtime.md
    workflow: 16
---

पंजीकरण के दौरान प्रत्येक plugin में इंजेक्ट किए गए `api.runtime` ऑब्जेक्ट का संदर्भ। होस्ट की आंतरिक चीज़ों को सीधे इंपोर्ट करने के बजाय इन हेल्पर का उपयोग करें।

<CardGroup cols={2}>
  <Card title="चैनल plugin" href="/hi/plugins/sdk-channel-plugins">
    चरण-दर-चरण मार्गदर्शिका, जो चैनल plugin के संदर्भ में इन हेल्पर का उपयोग करती है।
  </Card>
  <Card title="प्रोवाइडर plugin" href="/hi/plugins/sdk-provider-plugins">
    चरण-दर-चरण मार्गदर्शिका, जो प्रोवाइडर plugin के संदर्भ में इन हेल्पर का उपयोग करती है।
  </Card>
</CardGroup>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

`api.runtime.version` वर्तमान OpenClaw उत्पाद संस्करण है, जिसे साझा संस्करण रिज़ॉल्वर से प्राप्त किया जाता है, ताकि plugin को वही मान दिखाई दे जिसकी रिपोर्ट CLI करता है।

## कॉन्फ़िग लोड करना और लिखना

उस कॉन्फ़िग को प्राथमिकता दें जो सक्रिय कॉल पथ में पहले ही पास किया जा चुका है, उदाहरण के लिए पंजीकरण के दौरान `api.config` या चैनल/प्रोवाइडर कॉलबैक पर `cfg` आर्ग्युमेंट। इससे हॉट पथों पर कॉन्फ़िग को दोबारा पार्स करने के बजाय पूरे कार्य में एक ही प्रोसेस स्नैपशॉट प्रवाहित होता रहता है।

`api.runtime.config.current()` का उपयोग केवल तभी करें, जब किसी लंबे समय तक सक्रिय रहने वाले हैंडलर को वर्तमान प्रोसेस स्नैपशॉट चाहिए और उस फ़ंक्शन को कोई कॉन्फ़िग पास नहीं किया गया हो। लौटाया गया मान केवल-पढ़ने योग्य है; संपादन से पहले उसे क्लोन करें या किसी म्यूटेशन हेल्पर का उपयोग करें।

टूल फ़ैक्टरी को `ctx.runtimeConfig` के साथ `ctx.getRuntimeConfig()` मिलता है। जब टूल की परिभाषा बनाए जाने के बाद कॉन्फ़िग बदल सकता हो, तब लंबे समय तक सक्रिय रहने वाले टूल के `execute` कॉलबैक में गेटर का उपयोग करें।

परिवर्तनों को `api.runtime.config.mutateConfigFile(...)` या `api.runtime.config.replaceConfigFile(...)` से स्थायी करें। प्रत्येक लेखन में एक स्पष्ट `afterWrite` नीति चुनी जानी चाहिए:

- `afterWrite: { mode: "auto" }` Gateway के रीलोड प्लानर को निर्णय लेने देता है।
- `afterWrite: { mode: "restart", reason: "..." }` तब क्लीन रीस्टार्ट अनिवार्य करता है, जब लेखक जानता हो कि हॉट रीलोड असुरक्षित है।
- `afterWrite: { mode: "none", reason: "..." }` स्वचालित रीलोड/रीस्टार्ट को केवल तभी रोकता है, जब आगे की कार्रवाई का स्वामित्व कॉलर के पास हो।

म्यूटेशन हेल्पर `afterWrite` के साथ एक टाइप किया हुआ `followUp` सारांश लौटाते हैं, ताकि कॉलर लॉग कर सकें या जाँच सकें कि उन्होंने रीस्टार्ट का अनुरोध किया था या नहीं। वह रीस्टार्ट वास्तव में कब होगा, इसका स्वामित्व फिर भी Gateway के पास रहता है।

रनटाइम कॉन्फ़िग तक पहुँचने और उसमें लिखने के लिए `current()`, पास किया गया `cfg`, `mutateConfigFile(...)`, या
`replaceConfigFile(...)` उपयोग करें।

सीधे SDK इंपोर्ट के लिए, व्यापक `openclaw/plugin-sdk/config-runtime` संगतता बैरल के बजाय केंद्रित कॉन्फ़िग सबपाथ को प्राथमिकता दें: टाइप के लिए `config-contracts`, वर्तमान प्रोसेस स्नैपशॉट के लिए `runtime-config-snapshot`, और लेखन के लिए `config-mutation`। एंट्री-स्कोप वाले मान `api.pluginConfig` से पढ़ें; दिए गए टूल संदर्भ का उपयोग केवल उसके रनटाइम-व्यापी कॉन्फ़िग स्नैपशॉट के लिए करें, और plugin-विशिष्ट मर्जिंग को उसी सीमा पर रखें। बंडल किए गए plugin के परीक्षणों को व्यापक संगतता बैरल मॉक करने के बजाय इन केंद्रित सबपाथ को सीधे मॉक करना चाहिए।

OpenClaw का आंतरिक रनटाइम कोड भी इसी दिशा का पालन करता है: CLI, Gateway, या प्रोसेस सीमा पर कॉन्फ़िग को एक बार लोड करें, फिर उस मान को आगे पास करें। सफल म्यूटेशन लेखन प्रोसेस रनटाइम स्नैपशॉट को रीफ़्रेश करते हैं और उसके आंतरिक रिविज़न को आगे बढ़ाते हैं; लंबे समय तक सक्रिय रहने वाले कैश को कॉन्फ़िग को स्थानीय रूप से सीरियलाइज़ करने के बजाय रनटाइम-स्वामित्व वाली कैश कुंजी का उपयोग करना चाहिए। लंबे समय तक सक्रिय रहने वाले रनटाइम मॉड्यूल में परिवेशी `loadConfig()` कॉल के लिए शून्य-सहनशीलता स्कैनर है; पास किए गए `cfg`, अनुरोध के `context.getRuntimeConfig()`, या स्पष्ट प्रोसेस सीमा पर `getRuntimeConfig()` का उपयोग करें।

प्रोवाइडर और चैनल निष्पादन पथों को सक्रिय रनटाइम कॉन्फ़िग स्नैपशॉट का उपयोग करना चाहिए, न कि कॉन्फ़िग को वापस पढ़ने या संपादित करने के लिए लौटाए गए फ़ाइल स्नैपशॉट का। फ़ाइल स्नैपशॉट UI और लेखन के लिए SecretRef मार्कर जैसे स्रोत मानों को सुरक्षित रखते हैं; प्रोवाइडर कॉलबैक को रिज़ॉल्व किया हुआ रनटाइम दृश्य चाहिए। जब किसी हेल्पर को सक्रिय स्रोत स्नैपशॉट या सक्रिय रनटाइम स्नैपशॉट में से किसी के साथ भी कॉल किया जा सकता हो, तब क्रेडेंशियल पढ़ने से पहले `selectApplicableRuntimeConfig()` के माध्यम से रूट करें।

## पुनः उपयोग योग्य रनटाइम यूटिलिटी

बॉट द्वारा लिखे गए इनबाउंड संदेशों के लिए इनबाउंड `botLoopProtection` तथ्यों का उपयोग करें। कोर, नीति को किसी एक चैनल से बाँधे बिना, सत्र रिकॉर्ड और डिस्पैच से पहले साझा इन-मेमोरी स्लाइडिंग-विंडो गार्ड लागू करता है। गार्ड `(scopeId, conversationId, participant pair)` कुंजियों को ट्रैक करता है, किसी जोड़ी की दोनों दिशाओं को एक साथ गिनता है, विंडो बजट पार होने पर कूलडाउन लागू करता है, और निष्क्रिय एंट्री को अवसर मिलने पर हटाता है।

इस व्यवहार को ऑपरेटरों के लिए उपलब्ध कराने वाले चैनल plugin को आधारभूत बजट के लिए साझा `channels.defaults.botLoopProtection` आकार को प्राथमिकता देनी चाहिए, फिर उसके ऊपर चैनल/प्रोवाइडर-विशिष्ट ओवरराइड जोड़ने चाहिए। साझा कॉन्फ़िग सेकंड का उपयोग करता है, क्योंकि यह उपयोगकर्ता के लिए दृश्यमान है:

```typescript
type ChannelBotLoopProtectionConfig = {
  enabled?: boolean;
  maxEventsPerWindow?: number;
  windowSeconds?: number;
  cooldownSeconds?: number;
};
```

रिज़ॉल्व किए गए टर्न के साथ सामान्यीकृत बॉट-जोड़ी तथ्य पास करें। कोर डिफ़ॉल्ट, इकाई रूपांतरण, और `enabled` सिमैंटिक्स रिज़ॉल्व करता है:

```typescript
return {
  channel: "example",
  routeSessionKey,
  storePath,
  ctxPayload,
  recordInboundSession,
  runDispatch,
  botLoopProtection: {
    scopeId: "account-1",
    conversationId: "channel-1",
    senderId: "bot-a",
    receiverId: "bot-b",
    config: channelConfig.botLoopProtection,
    defaultsConfig: runtimeConfig.channels?.defaults?.botLoopProtection,
    defaultEnabled: allowBotsMode !== "off",
  },
};
```

`openclaw/plugin-sdk/pair-loop-guard-runtime` का सीधे उपयोग केवल उन कस्टम
दो-पक्षीय इवेंट लूप के लिए करें, जो साझा इनबाउंड उत्तर रनर से होकर नहीं गुजरते।

## रनटाइम नेमस्पेस

<AccordionGroup>
  <Accordion title="api.runtime.agent">
    एजेंट की पहचान, डायरेक्टरी, और सत्र प्रबंधन।

    ```typescript
    // एजेंट की कार्यशील डायरेक्टरी रिज़ॉल्व करें (agentId आवश्यक है)
    const agentDir = api.runtime.agent.resolveAgentDir(cfg, agentId);

    // एजेंट वर्कस्पेस रिज़ॉल्व करें
    const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId);

    // एजेंट की पहचान प्राप्त करें
    const identity = api.runtime.agent.resolveAgentIdentity(cfg);

    // डिफ़ॉल्ट चिंतन स्तर प्राप्त करें
    const thinking = api.runtime.agent.resolveThinkingDefault({
      cfg,
      provider,
      model,
    });

    // उपयोगकर्ता द्वारा दिए गए चिंतन स्तर को सक्रिय प्रोवाइडर प्रोफ़ाइल के विरुद्ध सत्यापित करें
    const policy = api.runtime.agent.resolveThinkingPolicy({ provider, model });
    const level = api.runtime.agent.normalizeThinkingLevel("extra high");
    if (level && policy.levels.some((entry) => entry.id === level)) {
      // स्तर को एम्बेड किए गए रन में पास करें
    }

    // एजेंट टाइमआउट प्राप्त करें
    const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

    // सुनिश्चित करें कि वर्कस्पेस मौजूद है
    await api.runtime.agent.ensureAgentWorkspace(cfg);

    // एम्बेड किया हुआ एजेंट टर्न चलाएँ
    const result = await api.runtime.agent.runEmbeddedAgent({
      sessionId: "my-plugin:task-1",
      runId: crypto.randomUUID(),
      workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId),
      prompt: "नवीनतम परिवर्तनों का सारांश दें",
      timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
    });
    ```

    `runEmbeddedAgent(...)` plugin कोड से सामान्य OpenClaw एजेंट टर्न शुरू करने के लिए तटस्थ हेल्पर है। यह चैनल द्वारा ट्रिगर किए गए उत्तरों के समान प्रोवाइडर/मॉडल रिज़ॉल्यूशन और एजेंट-हार्नेस चयन का उपयोग करता है।

    `runEmbeddedPiAgent(...)` मौजूदा plugin के लिए अप्रचलित संगतता उपनाम के रूप में बना हुआ है। नए कोड को `runEmbeddedAgent(...)` का उपयोग करना चाहिए।

    `resolveCliBackendDispatchEligibility({ provider, model, agentId, authProfileId, config, agentDir, workspaceDir })` उन कॉलर के साथ एम्बेडेड रनर का CLI-बैकएंड डिस्पैच निर्णय (रूट, बैकएंड की घोषित `subscriptionAuthDispatch` क्षमता, संग्रहित क्रेडेंशियल मोड—स्पष्ट रूप से पिन किए गए `authProfileId` का सम्मान करते हुए) साझा करता है, जो एम्बेड किए गए रन के लिए `cliBackendDispatch: "subscription-auth"` चुनते हैं। जब रन CLI बैकएंड के माध्यम से निष्पादित होगा तब यह `{ provider }` लौटाता है, और सीधे पासथ्रू पर बने रहने पर `undefined`, ताकि कॉलर वास्तव में निष्पादित होने वाले रन के लिए टाइमआउट बजट निर्धारित कर सकें।

    `resolveThinkingPolicy(...)` प्रोवाइडर/मॉडल के समर्थित चिंतन स्तर और वैकल्पिक डिफ़ॉल्ट लौटाता है। प्रोवाइडर plugin अपने चिंतन हुक के माध्यम से मॉडल-विशिष्ट प्रोफ़ाइल का स्वामित्व रखते हैं, इसलिए टूल plugin को प्रोवाइडर सूचियाँ इंपोर्ट या डुप्लिकेट करने के बजाय इस रनटाइम हेल्पर को कॉल करना चाहिए।

    `normalizeThinkingLevel(...)` उपयोगकर्ता के टेक्स्ट, जैसे `on`, `x-high`, या `extra high`, को रिज़ॉल्व की गई नीति के विरुद्ध जाँचने से पहले कैनोनिकल संग्रहित स्तर में बदलता है।

    **सत्र स्टोर हेल्पर** `api.runtime.agent.session` के अंतर्गत हैं:

    ```typescript
    const entry = api.runtime.agent.session.getSessionEntry({ agentId, sessionKey });
    for (const { sessionKey, entry } of api.runtime.agent.session.listSessionEntries({ agentId })) {
      // पुराने sessions.json आकार पर निर्भर हुए बिना सत्र पंक्तियों पर इटरेट करें।
    }
    await api.runtime.agent.session.patchSessionEntry({
      agentId,
      sessionKey,
      update: (entry) => ({ thinkingLevel: "high" }),
    });

    const created = await api.runtime.agent.session.createSessionEntry({
      cfg,
      key: "agent:main:my-plugin:task-1",
      initialEntry: {
        agentHarnessId: "my-harness",
        modelSelectionLocked: true,
        pluginExtensions: { "my-plugin": { phase: "initializing" } },
      },
      afterCreate: async () => ({
        pluginExtensions: { "my-plugin": { phase: "ready" } },
      }),
    });

    const storePath = api.runtime.agent.session.resolveStorePath(cfg.session?.store, { agentId });
    await api.runtime.agent.session.runWithWorkAdmission(
      { storePath, sessionKey },
      async (signal) => {
        // सत्र बनाएँ या अपडेट करें, फिर signal को स्वीकृत एजेंट रन में पास करें।
      },
    );
    ```

    सत्र वर्कफ़्लो के लिए `getSessionEntry(...)`, `listSessionEntries(...)`, `patchSessionEntry(...)`, या `upsertSessionEntry(...)` को प्राथमिकता दें। ये हेल्पर सत्रों को एजेंट/सत्र पहचान द्वारा संबोधित करते हैं, ताकि plugin पुराने `sessions.json` स्टोरेज आकार पर निर्भर न रहें। केवल-मेटाडेटा पैच, जिन्हें सत्र गतिविधि रीफ़्रेश नहीं करनी चाहिए, उनके लिए `preserveActivity: true` का उपयोग करें; और `replaceEntry: true` का उपयोग केवल तभी करें, जब कॉलबैक एक पूर्ण एंट्री लौटाता हो और हटाए गए फ़ील्ड हटे ही रहने चाहिए। डॉक्टर और माइग्रेशन पथ एक परमाणु कैनोनिकल-स्टोर मरम्मत के लिए `fallbackEntry`, `skipMaintenance`, और `requireWriteSuccess` को संयोजित कर सकते हैं।

    `createSessionEntry(...)` एक नई कैनोनिकल सत्र पंक्ति और ट्रांसक्रिप्ट बनाता है। इसकी विश्वसनीय `initialEntry` सतह जानबूझकर सीमित है: एक गैर-रिक्त `agentHarnessId`, वैकल्पिक `modelSelectionLocked: true`, और वैकल्पिक `pluginExtensions`। इंजेक्ट किया गया रनटाइम केवल `registerAgentHarness(...)` के माध्यम से कॉल करने वाले plugin के स्वामित्व वाली हार्नेस आईडी स्वीकार करता है; यह एक स्वामित्व इनवेरिएंट है, इन-प्रोसेस plugin के बीच कोई सैंडबॉक्स नहीं। यह मौजूदा पंक्ति को अस्वीकार करता है; `label` और `spawnedCwd` विश्वसनीय-एंट्री पैच के बजाय अलग निर्माण फ़ील्ड हैं।

    निर्माण `afterCreate` के माध्यम से सत्र जीवनचक्र म्यूटेशन फ़ेंस को थामे रखता है, इसलिए नया कार्य plugin-स्वामित्व वाली शुरुआत पूरी होने की प्रतीक्षा करता है और पहले से स्वीकृत कार्य के कारण निर्माण विफल हो जाता है। कॉलबैक को बनाई गई स्थिति का क्लोन मिलता है। यदि वह कोई पैच लौटाता है, तो उस पैच में केवल `pluginExtensions` हो सकता है, और उसका मान पूर्ण अंतिम `pluginExtensions` फ़ील्ड होता है। कॉलबैक या अंतिम स्थायीकरण की विफलता अपरिवर्तित नई पंक्ति और ट्रांसक्रिप्ट को रोलबैक कर देती है; संरक्षित रोलबैक समवर्ती रूप से बदली या क्लेम की गई पंक्ति को सुरक्षित रखता है। `recoverMatchingInitialEntry: true` केवल बाधित शुरुआत का पुनः प्रयास करने के लिए है, जब स्थायी किए गए विश्वसनीय फ़ील्ड बिल्कुल मेल खाते हों, और रिकवरी के लिए `afterCreate` द्वारा अंतिम पैच लौटाया जाना आवश्यक है।

    जब कोई plugin स्थायी सत्र पर कार्य शुरू करता है, तब `runWithWorkAdmission(...)` का उपयोग करें। कॉलबैक आर्काइव किए गए या समवर्ती रूप से बदले गए सत्रों को अस्वीकार करता है, आर्काइव/रीसेट/डिलीट म्यूटेशन को पूर्णता तक समन्वित रखता है, और उसे एक `AbortSignal` मिलता है, जिसे एजेंट रन तक अग्रेषित करना अनिवार्य है। कोई हार्नेस अपने प्रयोगात्मक `delegatedExecutionPluginIds` पंजीकरण फ़ील्ड के माध्यम से विश्वसनीय निष्पादन प्रतिनिधियों को स्पष्ट रूप से नाम दे सकता है। प्रतिनिधि केवल किसी हूबहू मौजूदा मॉडल-लॉक सत्र को स्वीकृत और चला सकते हैं; सभी सत्र म्यूटेशन हार्नेस स्वामी तक ही सीमित रहते हैं। [एजेंट हार्नेस plugin](/hi/plugins/sdk-agent-harness#delegated-execution) देखें।

    रखरखाव और मरम्मत Plugin एक सीमित-क्षेत्र वाले सत्र प्रविष्टि के लिए `deleteSessionEntry(...)`, जीवनचक्र-स्वामित्व वाले अस्थायी सत्रों के लिए `cleanupSessionLifecycleArtifacts(...)`, और किसी स्टोर में परिवर्तन करने से पहले `resolveSessionStoreBackupPaths(...)` का उपयोग कर सकते हैं। जब विलोपन को समवर्ती सत्र अपडेट के साथ टकराव से बचाना आवश्यक हो, तब `expectedSessionId` और `expectedUpdatedAt` पास करें; जब पहले के स्नैपशॉट में कोई सत्र आईडी न हो, तब `expectedSessionId: null` का उपयोग करें। ये सहायक सीमित मरम्मत/जीवनचक्र सतहें हैं, सामान्य स्टोर विलोपन API नहीं।

    `resolveStorePath(...)` और `updateSessionStoreEntry(...)` सत्र सहायकों को पूरा करते हैं: `resolveStorePath` दिए गए दायरे के लिए सत्र स्टोर पथ निर्धारित करता है, और जब कॉलर को वह पहले से ज्ञात हो, तब `updateSessionStoreEntry({ storePath, sessionKey, update })` स्टोर पथ के आधार पर सीधे एक प्रविष्टि में संशोधन करता है।

    `loadTranscriptEventsSync(...)` उन समकालिक डॉक्टर और मरम्मत पथों के लिए उपलब्ध है जो असमकालिक ट्रांसक्रिप्ट रनटाइम का उपयोग नहीं कर सकते। यह अपरिष्कृत `SessionStoreTranscriptEvent` रिकॉर्ड लौटाता है। सामान्य Plugin रनटाइम कोड को `openclaw/plugin-sdk/session-transcript-runtime` को प्राथमिकता देनी चाहिए।

    `formatSqliteSessionFileMarker(...)`, `parseSqliteSessionFileMarker(...)`, और `sqliteSessionFileMarkerMatchesSession(...)` उस कोड के लिए संक्रमणकालीन सहायक हैं जिसे अब भी `sessionFile` नामक पुराना फ़ील्ड मिलता है। पार्स किया गया SQLite मार्कर किसी सक्रिय SQLite ट्रांसक्रिप्ट लक्ष्य की पहचान करता है; यह फ़ाइल सिस्टम पथ नहीं है। नए API को मार्कर स्ट्रिंग के बजाय टाइपयुक्त सत्र पहचान ले जानी चाहिए।

    ट्रांसक्रिप्ट पढ़ने और लिखने के लिए, `openclaw/plugin-sdk/session-transcript-runtime` आयात करें और `{ agentId, sessionKey, sessionId }` के साथ `resolveSessionTranscriptIdentity(...)`, `resolveSessionTranscriptTarget(...)`, `readSessionTranscriptEvents(...)`, `readSessionTranscriptRawDelta(...)`, `readSessionTranscriptVisibleMessageDelta(...)`, `readVisibleSessionTranscriptMessageEntries(...)`, `appendSessionTranscriptMessageByIdentity(...)`, `publishSessionTranscriptUpdateByIdentity(...)`, या `withSessionTranscriptWriteLock(...)` का उपयोग करें। ये API Plugin को सक्रिय ट्रांसक्रिप्ट फ़ाइल पथों पर निर्भर हुए बिना किसी ट्रांसक्रिप्ट की पहचान करने, अपरिष्कृत घटनाएँ या दृश्यमान शाखा-सुरक्षित संदेश प्रविष्टियाँ पढ़ने, संदेश जोड़ने, अपडेट प्रकाशित करने और उसी ट्रांसक्रिप्ट लेखन लॉक के अंतर्गत संबंधित संचालन चलाने देते हैं। `readVisibleSessionTranscriptMessageEntries(...)` क्रमबद्ध पठन मेटाडेटा लौटाता है; इसका `seq` फ़ील्ड पुनः आरंभ करने योग्य कर्सर नहीं है।

    `appendSessionTranscriptMessageByIdentity(...)` पहले से कैनोनिकल संदेश का निम्न-स्तरीय जोड़ है। Plugin को शीर्ष-स्तरीय `MediaPath`, `MediaPaths`, `MediaUrl`, `MediaUrls`, `MediaType`, या `MediaTypes` के साथ मीडिया-युक्त उपयोगकर्ता पंक्तियाँ कृत्रिम रूप से नहीं बनानी चाहिए। चैनल इनग्रेस को `MsgContext.media` के माध्यम से क्रमबद्ध तथ्य पास करने चाहिए और होस्ट को उपयोगकर्ता टर्न के स्थायी संग्रहण का स्वामित्व देना चाहिए। होस्ट द्वारा तैयार किए गए स्थायी उपयोगकर्ता संदेश में `message.__openclaw.media` के अंतर्गत कैनोनिकल क्रमबद्ध तथ्य होते हैं; सामान्य जोड़ API पुरानी समानांतर सरणियों का अनुमान या सुधार नहीं करता।

    `readSessionTranscriptRawDelta(...)` एक सीमित `page`, `reset`, या `missing` परिणाम लौटाता है। अगले कॉल में अपारदर्शी `page.cursor` पास करें। केवल जोड़ने की क्रियाएँ कर्सर को सुरक्षित रखती हैं, जबकि ट्रांसक्रिप्ट प्रतिस्थापन नए बूटस्ट्रैप कर्सर के साथ `reset` लौटाता है। पृष्ठ डिफ़ॉल्ट रूप से 1,000 घटनाओं और 1,000,000 क्रमबद्ध बाइट तक सीमित होते हैं; कॉलर अधिकतम 10,000 घटनाएँ और 64 MiB का अनुरोध कर सकते हैं। जब अगली घटना अकेले ही `maxBytes` से अधिक हो, तो पृष्ठ खाली होता है और `requiredBytes` की रिपोर्ट करता है; यदि वह बाइट सीमा 64 MiB से अधिक न हो, तो कम-से-कम उसी सीमा के साथ पुनः प्रयास करें। इससे बड़ी व्यक्तिगत घटनाओं के लिए पूर्ण-पठन API आवश्यक है। कर्सर केवल स्थिति की पहचान करता है और कभी भी किसी अन्य सत्र तक पहुँच प्रदान नहीं करता।

    `readSessionTranscriptVisibleMessageDelta(...)` होस्ट-स्वामित्व वाले सक्रिय संदेश प्रक्षेपण पर समान सीमित बूटस्ट्रैप-और-पुनरारंभ संरचना प्रदान करता है। यह सबसे पुराने से नवीनतम क्रम में संदेश लौटाता है, ताकि संदर्भ इंजन प्रारंभिक इतिहास को निकाल सकें और अपारदर्शी कर्सर को अपने वॉटरमार्क के रूप में स्थायी रूप से सहेज सकें। कर्सर को अपरिवर्तित रूप में संग्रहित और वापस करें; यह निरंतरता संकेत है, प्राधिकरण क्रेडेंशियल नहीं। रैखिक जोड़ अंतिम लौटाए गए संदेश के बाद पुनः आरंभ होते हैं। ट्रांसक्रिप्ट प्रतिस्थापन, ऐसा कर्सर जिसका एंकर सक्रिय शाखा से बाहर हो गया या उसके भीतर स्थानांतरित हुआ, विकृत कर्सर और क्रॉस-सत्र कर्सर नए बूटस्ट्रैप कर्सर के साथ `reset` लौटाते हैं। गणना और बाइट के डिफ़ॉल्ट तथा अधिकतम सीमाएँ अपरिष्कृत डेल्टा API के समान हैं। शाखा परिवर्तन के बाद सक्रिय प्रक्षेपण के पुनर्निर्माण के दौरान, परिणाम `projection_rebuilding` कारण के साथ `unavailable` होता है; सक्रिय ट्रांसक्रिप्ट फ़ाइल पर फ़ॉलबैक करने के बजाय बाद में पुनः प्रयास करें।

    पुराने संपूर्ण-स्टोर और सक्रिय ट्रांसक्रिप्ट फ़ाइल सहायक अब Plugin SDK से निर्यात नहीं किए जाते। सत्र मेटाडेटा के लिए सीमित प्रविष्टि सहायकों और सक्रिय ट्रांसक्रिप्ट संचालनों के लिए ट्रांसक्रिप्ट पहचान सहायकों का उपयोग करें। जिन संग्रह/समर्थन कार्यप्रवाहों को फ़ाइल आर्टिफ़ैक्ट की आवश्यकता है, उन्हें सक्रिय सत्र रनटाइम API के बजाय अपनी समर्पित संग्रह सतहों का उपयोग करना चाहिए।

  </Accordion>
  <Accordion title="api.runtime.agent.defaults">
    डिफ़ॉल्ट मॉडल और प्रदाता स्थिरांक:

    ```typescript
    const model = api.runtime.agent.defaults.model; // उदाहरण: "gpt-5.6-sol"
    const provider = api.runtime.agent.defaults.provider; // उदाहरण: "openai"
    ```

  </Accordion>

  <Accordion title="api.runtime.llm">
    प्रदाता की आंतरिक कार्यप्रणाली आयात किए बिना या OpenClaw मॉडल/प्रमाणीकरण/मूल URL तैयारी की नकल किए बिना होस्ट-स्वामित्व वाला पाठ पूर्णता संचालन चलाएँ।

    ```typescript
    const result = await api.runtime.llm.complete({
      messages: [{ role: "user", content: "इस ट्रांसक्रिप्ट का सारांश प्रस्तुत करें।" }],
      purpose: "my-plugin.summary",
      maxTokens: 512,
      temperature: 0.2,
      reasoning: "high",
    });
    ```

    प्रदाता ऑर्केस्ट्रेशन HTTP अनुरोध जारी करने से पहले कॉन्फ़िगर की गई स्थानीय-सेवा जीवनचक्र को भी प्राप्त कर सकता है:

    ```typescript
    const lease = await api.runtime.llm.acquireLocalService(
      {
        providerId,
        baseUrl,
        headers,
      },
      signal,
    );
    try {
      // प्रदाता अनुरोध भेजें और उसका पूरी तरह उपयोग करें।
    } finally {
      await lease?.release();
    }
    ```

    `acquireLocalService(...)` एक स्थिर, सामान्य प्रदाता-सेवा SDK अनुबंध है। होस्ट `models.providers.<providerId>.localService` से प्रक्रिया कॉन्फ़िगरेशन निर्धारित करता है; कॉलर कमांड, आर्ग्युमेंट, परिवेश या जीवनचक्र नीति प्रदान नहीं कर सकते। प्रक्रिया शुरू करना, तत्परता, निदान और निष्क्रिय-अवस्था में रोकने की नीति होस्ट की आंतरिक कार्यप्रणाली बनी रहती है।

    ठीक वही कॉन्फ़िगर किया गया प्रदाता आईडी और निर्धारित अनुरोध मूल URL पास करें। उपनामों को अडैप्टर आईडी से न बदलें: अलग-अलग उपनाम अलग-अलग स्थानीय GPU होस्ट की ओर संकेत कर सकते हैं। होस्ट उन एंडपॉइंट को अस्वीकार करता है जो कॉन्फ़िगर किए गए प्रदाता मूल URL से मेल नहीं खाते, सिवाय Ollama और LM Studio अडैप्टर द्वारा उपयोग किए जाने वाले `/v1` सामान्यीकरण के। होस्ट स्टार्टअप क्रमबद्धता, तत्परता जाँच, अनुरोध लीज़, निरस्तीकरण प्रबंधन और निष्क्रिय शटडाउन का स्वामित्व रखता है।

    सहायक OpenClaw के अंतर्निहित रनटाइम वाला ही सरल-पूर्णता तैयारी पथ और होस्ट-स्वामित्व वाला रनटाइम कॉन्फ़िगरेशन स्नैपशॉट उपयोग करता है। संदर्भ इंजनों को सत्र-बद्ध `llm.complete` क्षमता मिलती है, इसलिए मॉडल कॉल सक्रिय सत्र के एजेंट का उपयोग करते हैं और चुपचाप डिफ़ॉल्ट एजेंट पर फ़ॉलबैक नहीं करते। उपलब्ध होने पर परिणाम में प्रदाता/मॉडल/एजेंट अभिलेखन के साथ सामान्यीकृत टोकन, कैश और अनुमानित लागत उपयोग शामिल होता है।

    चयनित मॉडल के लिए रीजनिंग प्रयास का अनुरोध करने हेतु `reasoning` सेट करें। होस्ट पूर्णता भेजने से पहले चयनित प्रदाता और मॉडल के लिए कैनोनिकल चिंतन स्तरों (`off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `adaptive`, `max`, और `ultra`) को सामान्यीकृत करता है। `adaptive`, `medium` बन जाता है; समर्थित होने पर `max` और `ultra`, `max` बन जाते हैं, अन्यथा `xhigh`।

    <Warning>
    मॉडल ओवरराइड के लिए कॉन्फ़िगरेशन में `plugins.entries.<id>.llm.allowModelOverride: true` के माध्यम से ऑपरेटर की स्पष्ट सहमति आवश्यक है। विश्वसनीय Plugin को विशिष्ट कैनोनिकल `provider/model` लक्ष्यों तक सीमित करने के लिए `plugins.entries.<id>.llm.allowedModels` का उपयोग करें। क्रॉस-एजेंट पूर्णताओं के लिए `plugins.entries.<id>.llm.allowAgentIdOverride: true` आवश्यक है।
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.gateway">
    वर्तमान Plugin की विश्वसनीय रनटाइम पहचान सुरक्षित रखते हुए प्रक्रिया के भीतर किसी अन्य Gateway विधि को कॉल करें। यह उन बंडल किए गए या विश्वसनीय आधिकारिक Plugin के लिए है जो लूपबैक WebSocket कनेक्शन खोले बिना Plugin-स्वामित्व वाली Gateway क्षमताओं को संयोजित करते हैं।

    ```typescript
    if (await api.runtime.gateway.isAvailable()) {
      const result = await api.runtime.gateway.request<{ callId: string }>(
        "voicecall.start",
        { to: "+15550001234", mode: "conversation" },
        { timeoutMs: 60_000 },
      );
    }
    ```

    अनुरोध `operator.write` दायरे का उपयोग करते हैं और व्यवस्थापक दायरा प्रदान नहीं करते। मनमाने बाहरी Plugin से आने वाले कॉल अस्वीकार कर दिए जाते हैं। विफल विधियाँ संरचित `details`, पुनः प्रयास मेटाडेटा और पुनर्प्राप्ति प्रवाहों के लिए Gateway त्रुटि कोड सुरक्षित रखते हुए `GatewayClientRequestError` थ्रो करती हैं। उन टूल से यह पथ चुनने से पहले `isAvailable()` का उपयोग करें जो स्वतंत्र एजेंट प्रक्रियाओं में भी चल सकते हैं।

  </Accordion>
  <Accordion title="api.runtime.subagent">
    पृष्ठभूमि सबएजेंट रन आरंभ और प्रबंधित करें।

    ```typescript
    // सबएजेंट रन शुरू करें
    const { runId } = await api.runtime.subagent.run({
      sessionKey: "agent:main:subagent:search-helper",
      message: "इस क्वेरी को केंद्रित अनुवर्ती खोजों में विस्तृत करें।",
      toolsAlsoAllow: ["my_plugin_progress"],
      provider: "openai", // वैकल्पिक ओवरराइड
      model: "gpt-5.6-sol", // वैकल्पिक ओवरराइड
      deliver: false,
    });

    // पूर्ण होने की प्रतीक्षा करें
    const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

    // सत्र संदेश पढ़ें
    const { messages } = await api.runtime.subagent.getSessionMessages({
      sessionKey: "agent:main:subagent:search-helper",
      limit: 10,
    });

    // सत्र हटाएँ
    await api.runtime.subagent.deleteSession({
      sessionKey: "agent:main:subagent:search-helper",
    });
    ```

    <Warning>
    मॉडल ओवरराइड (`provider`/`model`) के लिए कॉन्फ़िगरेशन में `plugins.entries.<id>.subagent.allowModelOverride: true` के माध्यम से ऑपरेटर की स्पष्ट सहमति आवश्यक है। अविश्वसनीय Plugin अब भी सबएजेंट चला सकते हैं, लेकिन ओवरराइड अनुरोध अस्वीकार कर दिए जाते हैं।
    </Warning>

    `toolsAlsoAllow` कॉल करने वाले Plugin द्वारा पंजीकृत सटीक, अद्वितीय स्वामित्व वाले टूल को वर्कर की सामान्य टूल सतह में जोड़ता है। रनटाइम मुख्य टूल और किसी अन्य Plugin के साथ साझा नामों को अस्वीकार करता है। स्पष्ट अनुमत-सूचियों और निषेधों सहित प्रोफ़ाइल तथा ऑपरेटर टूल नीतियाँ अब भी लागू होती हैं।

    `deleteSession(...)`, उसी Plugin द्वारा `api.runtime.subagent.run(...)` के माध्यम से बनाए गए सत्रों को हटा सकता है। मनमाने उपयोगकर्ता या ऑपरेटर सत्रों को हटाने के लिए अब भी व्यवस्थापक-दायरे वाला Gateway अनुरोध आवश्यक है।

  </Accordion>
  <Accordion title="api.runtime.sandbox">
    किसी एजेंट सत्र के लिए प्रभावी सैंडबॉक्स कार्यक्षेत्र प्राधिकार का निरीक्षण करें।

    ```typescript
    const authority = api.runtime.sandbox.resolveWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
    });

    const liveAuthority = await api.runtime.sandbox.prepareWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
      workspaceDir,
      confinedToolNames: ["my_plugin_safe_tool"],
    });
    ```

    परिणाम बताता है कि यह सत्र सैंडबॉक्स में है या नहीं, इसका कार्यक्षेत्र अनुपलब्ध, केवल-पठन या लेखन-योग्य है, और जब प्रभावी Docker, टूल, सत्र, ब्राउज़र या उन्नत नीति उस कार्यक्षेत्र से बाहर निकल सकती हो तब वैकल्पिक `confinementError` देता है। इसका उपयोग होस्ट-स्वामित्व वाले उन प्रत्यायोजन निर्णयों के लिए करें जिन्हें किसी वर्कर को उसके कॉलर से अधिक प्राधिकार नहीं देना चाहिए। यह सत्यापन सहायक है, कॉलर के स्वयं के प्राधिकरण की जाँच का प्रतिस्थापन नहीं।

    `prepareWorkspaceAuthority(...)` वही नीति जाँच करता है और `workspaceDir` के लिए Docker सैंडबॉक्स भी तैयार करता है। यह ऐसे सक्रिय कंटेनर को अस्वीकार करता है जिसका लाइव कॉन्फ़िगरेशन हैश अनुरोधित माउंट या नीति से मेल नहीं खाता। केवल उन्हीं सटीक टूल नामों को पास करें जिनके पंजीकृत कार्यान्वयन को कॉल करने वाला Plugin सीमित करता है; वाइल्डकार्ड उपसर्ग टूल स्वामित्व सिद्ध नहीं करते।

  </Accordion>
  <Accordion title="api.runtime.nodes">
    Gateway द्वारा लोड किए गए Plugin कोड या Plugin CLI कमांड से जुड़े हुए Node सूचीबद्ध करें और Node-होस्ट कमांड चलाएँ। इसका उपयोग तब करें जब किसी Plugin के पास युग्मित डिवाइस पर स्थानीय कार्य का स्वामित्व हो, उदाहरण के लिए किसी अन्य Mac पर ब्राउज़र या ऑडियो ब्रिज।

    ```typescript
    const { nodes } = await api.runtime.nodes.list({ connected: true });

    const result = await api.runtime.nodes.invoke({
      nodeId: "mac-studio",
      command: "my-plugin.command",
      params: { action: "start" },
      timeoutMs: 30000,
    });
    ```

    `nodes.list(...)` में प्रत्येक कनेक्टेड Node के विज्ञापित
    `nodePluginTools` डिस्क्रिप्टर शामिल होते हैं, जब वह Node एजेंट को Plugin या MCP-समर्थित
    टूल उपलब्ध कराता है। वे डिस्क्रिप्टर लाइव कनेक्शन स्थिति हैं: Node के डिस्कनेक्ट होने पर Gateway
    उन्हें हटा देता है, और स्थानीय Plugin/MCP इन्वेंट्री में बदलाव के बाद कोई Node उन्हें
    `node.pluginTools.update` से बदल सकता है।

    Gateway के भीतर यह रनटाइम इन-प्रोसेस होता है। Plugin CLI कमांड में यह कॉन्फ़िगर किए गए Gateway को RPC के माध्यम से कॉल करता है, इसलिए `openclaw googlemeet recover-tab` जैसे कमांड टर्मिनल से पेयर किए गए Node की जाँच कर सकते हैं। Node कमांड फिर भी सामान्य Gateway Node पेयरिंग, कमांड अनुमतिसूचियों, Plugin Node-इनवोक नीतियों और Node-स्थानीय कमांड प्रबंधन से होकर गुजरते हैं।

    Node पर होस्ट किए गए एजेंट टूल उपलब्ध कराने वाले Plugin गैर-खतरनाक कमांड के लिए `agentTool.defaultPlatforms` सेट कर सकते हैं, जिन्हें डिफ़ॉल्ट रूप से अनुमतिसूची में होना चाहिए। जब ऑपरेटरों को `gateway.nodes.commands.allow` के साथ स्पष्ट रूप से सहमति देनी आवश्यक हो, तो इसे छोड़ दें। खतरनाक Node-होस्ट कमांड को `api.registerNodeInvokePolicy(...)` के साथ Node-इनवोक नीति पंजीकृत करनी चाहिए; यह नीति कमांड अनुमतिसूची जाँचों के बाद और कमांड को Node पर अग्रेषित करने से पहले Gateway में चलती है, इसलिए सीधे `node.invoke` कॉल, Node पर होस्ट किए गए Plugin टूल और उच्च-स्तरीय Plugin टूल समान प्रवर्तन पथ साझा करते हैं।

    <Warning>
    वैकल्पिक `scopes` फ़ील्ड इनवोकेशन के लिए Gateway ऑपरेटर स्कोप का अनुरोध करता है। OpenClaw इसे केवल बंडल किए गए Plugin और विश्वसनीय आधिकारिक Plugin इंस्टॉलेशन के लिए स्वीकार करता है; अन्य Plugin से आने वाले अनुरोध कॉल के विशेषाधिकार नहीं बढ़ाते। इसका उपयोग केवल तभी करें, जब किसी विश्वसनीय Plugin को अधिक कठोर Gateway स्कोप वाले Node कमांड को इनवोक करना आवश्यक हो, जैसे `operator.admin`।
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.tasks">
    Task Flow और Task Run की स्थिति को किसी मौजूदा OpenClaw सेशन कुंजी या विश्वसनीय टूल संदर्भ से बाँधें।

    - `api.runtime.tasks.managedFlows` परिवर्तन करने में सक्षम है: Task Flow बनाएँ, आगे बढ़ाएँ और रद्द करें।
    - `api.runtime.tasks.flows` और `api.runtime.tasks.runs` सूचीकरण और स्थिति लुकअप के लिए केवल-पढ़ने योग्य DTO दृश्य हैं; दोनों `bindSession(...)` / `fromToolContext(...)` के साथ `get`, `list`, `findLatest` और `resolve` उपलब्ध कराते हैं।

    Task Flow स्थायी बहु-चरणीय वर्कफ़्लो स्थिति को ट्रैक करता है। यह शेड्यूलर नहीं है:
    भविष्य में सक्रिय करने के लिए Cron या `api.session.workflow.scheduleSessionTurn(...)` का उपयोग करें,
    फिर शेड्यूल किए गए टर्न से `managedFlows` का उपयोग करें, जब उस कार्य को
    Flow स्थिति, चाइल्ड टास्क, प्रतीक्षा या रद्दीकरण की आवश्यकता हो।

    ```typescript
    const taskFlow = api.runtime.tasks.managedFlows.fromToolContext(ctx);

    const created = taskFlow.createManaged({
      controllerId: "my-plugin/review-batch",
      goal: "नए पुल रिक्वेस्ट की समीक्षा करें",
    });

    const child = taskFlow.runTask({
      flowId: created.flowId,
      runtime: "acp",
      childSessionKey: "agent:main:subagent:reviewer",
      task: "PR #123 की समीक्षा करें",
      status: "running",
      startedAt: Date.now(),
    });

    const waiting = taskFlow.setWaiting({
      flowId: created.flowId,
      expectedRevision: created.revision,
      currentStep: "await-human-reply",
      waitJson: { kind: "reply", channel: "telegram" },
    });
    ```

    जब आपकी अपनी बाइंडिंग परत से आपके पास पहले से विश्वसनीय OpenClaw सेशन कुंजी हो, तब `bindSession({ sessionKey, requesterOrigin })` का उपयोग करें। सीधे उपयोगकर्ता इनपुट से बाइंड न करें।

  </Accordion>
  <Accordion title="api.runtime.tts">
    टेक्स्ट-टू-स्पीच संश्लेषण।

    ```typescript
    // मानक TTS
    const clip = await api.runtime.tts.textToSpeech({
      text: "OpenClaw की ओर से नमस्ते",
      cfg: api.config,
    });

    // टेलीफ़ोनी-अनुकूलित TTS
    const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
      text: "OpenClaw की ओर से नमस्ते",
      cfg: api.config,
    });

    // उपलब्ध आवाज़ों की सूची बनाएँ
    const voices = await api.runtime.tts.listVoices({
      provider: "elevenlabs",
      cfg: api.config,
    });
    ```

    यह कोर `tts` कॉन्फ़िगरेशन और प्रदाता चयन का उपयोग करता है। PCM ऑडियो बफ़र + सैंपल दर लौटाता है। स्ट्रीमिंग संश्लेषण के लिए `textToSpeechStream` भी उपलब्ध है।

  </Accordion>
  <Accordion title="api.runtime.mediaUnderstanding">
    इमेज, ऑडियो और वीडियो विश्लेषण।

    ```typescript
    // किसी इमेज का वर्णन करें
    const image = await api.runtime.mediaUnderstanding.describeImageFile({
      filePath: "/tmp/inbound-photo.jpg",
      cfg: api.config,
      agentDir: "/tmp/agent",
    });

    // ऑडियो का लिप्यंतरण करें
    const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
      filePath: "/tmp/inbound-audio.ogg",
      cfg: api.config,
      mime: "audio/ogg", // वैकल्पिक, जब MIME का अनुमान नहीं लगाया जा सकता
    });

    // किसी वीडियो का वर्णन करें
    const video = await api.runtime.mediaUnderstanding.describeVideoFile({
      filePath: "/tmp/inbound-video.mp4",
      cfg: api.config,
    });

    // सामान्य फ़ाइल विश्लेषण
    const result = await api.runtime.mediaUnderstanding.runFile({
      filePath: "/tmp/inbound-file.pdf",
      cfg: api.config,
    });

    // किसी विशिष्ट प्रदाता/मॉडल के माध्यम से संरचित इमेज निष्कर्षण।
    // कम-से-कम एक इमेज शामिल करें; टेक्स्ट इनपुट पूरक संदर्भ हैं।
    const evidence = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
      provider: "codex",
      model: "gpt-5.6-sol",
      input: [
        {
          type: "image",
          buffer: receiptImageBuffer,
          fileName: "receipt.png",
          mime: "image/png",
        },
        { type: "text", text: "हस्तलिखित नोट के बजाय मुद्रित कुल राशि को प्राथमिकता दें।" },
      ],
      instructions: "विक्रेता, कुल राशि और खोजने योग्य टैग निकालें।",
      schemaName: "receipt.evidence",
      jsonSchema: {
        type: "object",
        properties: {
          vendor: { type: "string" },
          total: { type: "number" },
          tags: { type: "array", items: { type: "string" } },
        },
        required: ["vendor", "total"],
      },
      cfg: api.config,
    });
    ```

    कोई आउटपुट उत्पन्न न होने पर `{ text: undefined }` लौटाता है (जैसे, छोड़ा गया इनपुट)।

    `describeImageFileWithModel(...)` किसी विशिष्ट प्रदाता/मॉडल के माध्यम से पहले से ज्ञात इमेज का वर्णन करता है और `describeImageFile(...)` द्वारा उपयोग किए जाने वाले डिफ़ॉल्ट सक्रिय-मॉडल रिज़ॉल्यूशन को बायपास करता है।

  </Accordion>
  <Accordion title="api.runtime.imageGeneration">
    इमेज जनरेशन।

    ```typescript
    const result = await api.runtime.imageGeneration.generate({
      prompt: "सूर्यास्त को चित्रित करता हुआ एक रोबोट",
      cfg: api.config,
    });

    const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.videoGeneration">
    इमेज जनरेशन की संरचना के अनुरूप वीडियो जनरेशन।

    ```typescript
    const result = await api.runtime.videoGeneration.generate({
      prompt: "सूर्योदय के समय समुद्रतट के ऊपर उड़ता हुआ ड्रोन शॉट",
      cfg: api.config,
    });

    const providers = api.runtime.videoGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.musicGeneration">
    इमेज जनरेशन की संरचना के अनुरूप संगीत जनरेशन।

    ```typescript
    const result = await api.runtime.musicGeneration.generate({
      prompt: "कोडिंग सेशन के लिए एक उत्साहपूर्ण लो-फ़ाई ट्रैक",
      cfg: api.config,
    });

    const providers = api.runtime.musicGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.webSearch">
    वेब खोज।

    ```typescript
    const providers = api.runtime.webSearch.listProviders({ config: api.config });

    const result = await api.runtime.webSearch.search({
      config: api.config,
      args: { query: "OpenClaw Plugin SDK", count: 5 },
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.media">
    निम्न-स्तरीय मीडिया यूटिलिटी।

    ```typescript
    const webMedia = await api.runtime.media.loadWebMedia(url);
    const mime = await api.runtime.media.detectMime(buffer);
    const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "इमेज"
    const isVoice = api.runtime.media.isVoiceCompatibleAudio(filePath);
    const metadata = await api.runtime.media.getImageMetadata(filePath);
    const resized = await api.runtime.media.resizeToJpeg(buffer, { maxWidth: 800 });
    const terminalQr = await api.runtime.media.renderQrTerminal("https://openclaw.ai");
    const pngQr = await api.runtime.media.renderQrPngBase64("https://openclaw.ai", {
      scale: 6, // 1-12
      marginModules: 4, // 0-16
    });
    const pngQrDataUrl = await api.runtime.media.renderQrPngDataUrl("https://openclaw.ai");
    const tmpRoot = resolvePreferredOpenClawTmpDir();
    const pngQrFile = await api.runtime.media.writeQrPngTempFile("https://openclaw.ai", {
      tmpRoot,
      dirPrefix: "my-plugin-qr-",
      fileName: "qr.png",
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.config">
    वर्तमान रनटाइम कॉन्फ़िगरेशन स्नैपशॉट और ट्रांज़ैक्शनल कॉन्फ़िगरेशन लेखन। सक्रिय कॉल पथ में
    पहले से पास किए गए कॉन्फ़िगरेशन को प्राथमिकता दें; `current()` का उपयोग
    केवल तभी करें, जब हैंडलर को सीधे प्रोसेस स्नैपशॉट की आवश्यकता हो।

    ```typescript
    const cfg = api.runtime.config.current();
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    `mutateConfigFile(...)` और `replaceConfigFile(...)` एक `followUp`
    मान लौटाते हैं, उदाहरण के लिए `{ mode: "restart", requiresRestart: true, reason }`,
    जो Gateway से पुनरारंभ नियंत्रण लिए बिना लेखक की मंशा दर्ज करता है।

  </Accordion>
  <Accordion title="api.runtime.system">
    सिस्टम-स्तरीय यूटिलिटी।

    ```typescript
    await api.runtime.system.enqueueSystemEvent(event);
    api.runtime.system.requestHeartbeat({
      source: "other",
      intent: "event",
      reason: "plugin-event",
    });
    api.runtime.system.requestHeartbeatNow({ reason: "plugin-event" }); // अप्रचलित संगतता उपनाम।
    const heartbeatResult = await api.runtime.system.runHeartbeatOnce({
      reason: "plugin-triggered-check",
    });
    const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
    const hint = api.runtime.system.formatNativeDependencyHint(pkg);
    ```

    `runHeartbeatOnce(...)` सामान्य कोएलैस टाइमर को बायपास करते हुए एक Heartbeat चक्र तुरंत चलाता है। डिफ़ॉल्ट `target: "none"` दमन के बजाय अंतिम सक्रिय चैनल पर डिलीवरी बाध्य करने के लिए `{ heartbeat: { target: "last" } }` पास करें।

    `runCommandWithTimeout(...)` कैप्चर किए गए `stdout` और `stderr`, वैकल्पिक
    ट्रंकेशन गणनाएँ, `code`, `signal`, `killed`, `termination` और
    `noOutputTimedOut` लौटाता है। जब चाइल्ड प्रोसेस गैर-शून्य एग्ज़िट कोड प्रदान नहीं करता, तब टाइमआउट और नो-आउटपुट-टाइमआउट परिणाम `code: 124`
    रिपोर्ट करते हैं। गैर-टाइमआउट
    सिग्नल एग्ज़िट फिर भी `code: null` लौटा सकते हैं, इसलिए टाइमआउट के कारणों में अंतर करने के लिए `termination` और
    `noOutputTimedOut` का उपयोग करें।

  </Accordion>
  <Accordion title="api.runtime.events">
    इवेंट सदस्यताएँ।

    ```typescript
    api.runtime.events.onAgentEvent((event) => {
      /* ... */
    });
    api.runtime.events.onSessionTranscriptUpdate((update) => {
      /* ... */
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.logging">
    लॉगिंग।

    ```typescript
    const verbose = api.runtime.logging.shouldLogVerbose();
    const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
    ```

  </Accordion>
  <Accordion title="api.runtime.modelAuth">
    मॉडल और प्रदाता प्रमाणीकरण रिज़ॉल्यूशन।

    ```typescript
    const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });

    // अनुरोध के लिए तैयार प्रमाणीकरण, जिसमें प्रदाता रनटाइम आदान-प्रदान शामिल हैं (जैसे OAuth रीफ़्रेश)
    const runtimeAuth = await api.runtime.modelAuth.getRuntimeAuthForModel({ model, cfg });

    const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
      provider: "openai",
      cfg,
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.state">
    स्टेट डायरेक्टरी रिज़ॉल्यूशन और SQLite-समर्थित कुंजीबद्ध स्टोरेज।

    ```typescript
    const stateDir = api.runtime.state.resolveStateDir(process.env);
    const store = api.runtime.state.openKeyedStore<MyRecord>({
      namespace: "my-feature",
      maxEntries: 200,
      defaultTtlMs: 15 * 60_000,
    });

    await store.register("key-1", { value: "hello" });
    const claimed = await store.registerIfAbsent("dedupe-key", { value: "first" });
    const value = await store.lookup("key-1");
    await store.deleteIf?.("key-1", (current) => current.value === "hello");
    await store.consume("key-1");
    await store.clear();

    const blobs = api.runtime.state.openBlobStore<MyBlobMetadata>({
      namespace: "rendered-artifacts",
      maxEntries: 100,
      maxBytesPerEntry: 4 * 1024 * 1024,
      maxBytesPerNamespace: 64 * 1024 * 1024,
      defaultTtlMs: 15 * 60_000,
    });
    await blobs.register(
      "artifact-1",
      new TextEncoder().encode("binary or text payload"),
      { contentType: "text/plain" },
    );
    const blob = await blobs.lookup("artifact-1");

    await api.runtime.state.withLease(
      {
        namespace: "my-feature",
        key: "writer",
        database: { scope: "agent", agentId },
        leaseMs: 5 * 60_000,
        waitMs: 30_000,
      },
      async ({ signal, assertOwned }) => {
        await runExternalWriter({ signal });
        assertOwned();
      },
    );
    ```

    कुंजीबद्ध स्टोर पुनः प्रारंभ होने के बाद भी बने रहते हैं और रनटाइम-बाउंड Plugin आईडी के अनुसार अलग रखे जाते हैं। परमाण्विक डीडुप दावों के लिए `registerIfAbsent(...)` का उपयोग करें: जब कुंजी अनुपस्थित या समाप्त हो चुकी थी और पंजीकृत कर दी गई हो, तो यह `true` लौटाता है; या जब कोई सक्रिय मान पहले से मौजूद हो, तो उसके मान, निर्माण समय या TTL को अधिलेखित किए बिना `false` लौटाता है। जब सफ़ाई में केवल पहले देखे गए मान को हटाना आवश्यक हो, तो `deleteIf(...)` का उपयोग करें; इसका समकालिक प्रेडिकेट और विलोपन एक SQLite ट्रांज़ैक्शन में चलते हैं। सीमाएँ: प्रति नेमस्पेस `maxEntries`, प्रति Plugin 50,000 सक्रिय पंक्तियाँ, 64KB से छोटे JSON मान और वैकल्पिक TTL समाप्ति। डिफ़ॉल्ट रूप से, किसी भी पंक्ति सीमा पर लेखन उस नेमस्पेस की सबसे पुरानी सक्रिय पंक्तियाँ हटाता है जिसमें लिखा जा रहा है; उस लेखन के लिए सहोदर नेमस्पेस से पंक्तियाँ नहीं हटाई जातीं, और यदि नेमस्पेस पर्याप्त पंक्तियाँ खाली नहीं कर पाता, तो लेखन फिर भी विफल होता है। स्थायी स्वामित्व रिकॉर्डों के लिए, जिन्हें कभी हटाया नहीं जाना चाहिए, `overflowPolicy: "reject-new"` सेट करें: नई कुंजियाँ किसी भी सीमा पर विफल होती हैं, जबकि मौजूदा कुंजियाँ अपडेट की जा सकती हैं।

    `openSyncKeyedStore<T>(...)` उन कॉलर के लिए समकालिक विधियों (`register`, `registerIfAbsent`, `deleteIf`, `lookup`, `consume`, `clear` सभी प्रॉमिस के बजाय सीधे मान लौटाते हैं) वाला वही स्टोर स्वरूप लौटाता है जो प्रतीक्षा नहीं कर सकते।

    `openBlobStore<TMetadata>(...)` सीमित बाइनरी पेलोड को base64 या फ़ाइल साइडकार के बिना साझा SQLite में संग्रहीत करता है। इसके लिए प्रति-प्रविष्टि, प्रति-नेमस्पेस बाइट और पंक्ति सीमाएँ आवश्यक हैं; यह API सीमा पर बाइट ऐरे की प्रतिलिपि बनाता है; और प्रत्येक BLOB लोड किए बिना मेटाडेटा सूचीबद्ध करता है। `register(...)` एक स्पष्ट अपसर्ट है, जिसमें समाप्त हो चुकी कुंजियाँ भी शामिल हैं। `registerIfAbsent(...)` टकराव-सुरक्षित निर्माण प्रदान करता है: समाप्त हो चुकी कुंजी तब तक अधिकृत रहती है, जब तक उसका स्वामी `deleteExpiredKey(key)` या `deleteExpired()` के साथ उसका दावा नहीं करता, जिससे SQLite कमिट के बाद संबंधित नामित आर्टिफ़ैक्ट हटाने के लिए आवश्यक मेटाडेटा सुरक्षित रहता है। TTL वाली कोई भी पंक्ति क्षणिक होती है और समाप्त होने से पहले भी बैकअप/पुनर्स्थापना से बाहर रखी जाती है; स्थायी, पुनर्स्थापनीय स्टेट के लिए TTL न दें। होस्ट फ़्यूज़ प्रत्येक BLOB को 100 MiB, प्रत्येक Plugin को भौतिक रूप से संग्रहीत BLOB के 512 MiB और प्रत्येक Plugin को भौतिक रूप से संग्रहीत 50,000 पंक्तियों तक सीमित करते हैं, जिनमें स्वामी द्वारा सफ़ाई की प्रतीक्षा कर रही समाप्त पंक्तियाँ भी शामिल हैं। जब बाहरी मूर्त रूपों को प्रतिस्थापन या निष्कासन के कारण चुपचाप अनाथ नहीं छोड़ा जाना चाहिए, तब `overflowPolicy: "reject-new"` के साथ `registerIfAbsent(...)` का उपयोग करें।

    `openChannelIngressQueue<TPayload>(...)` कॉल करने वाले Plugin के दायरे में एक स्थायी इनग्रेस कतार खोलता है, ताकि उन इनबाउंड इवेंट को बफ़र किया जा सके जिन्हें पुनः प्रारंभ होने के दौरान कम-से-कम-एक-बार प्रोसेसिंग की आवश्यकता होती है। जब पुराने दावे की पुनर्प्राप्ति `shouldRecover` का उपयोग करती है, तब यदि दूषित दावा किए गए पेलोड को क्वारंटीन किया जाना चाहिए, तो `shouldRecoverCorrupt` भी प्रदान करें: इसकी पेलोड-स्वतंत्र दावा पहचान Plugin को कतार द्वारा पंक्ति को टूमस्टोन करने से पहले सक्रिय स्वामी और लेन नीति सुरक्षित रखने देती है।

    `withLease(...)` OpenClaw प्रक्रियाओं में सहयोगात्मक Plugin कार्य को क्रमबद्ध करता है। एक वैश्विक स्वामी के लिए `database: { scope: "shared" }` या स्वतंत्र प्रति-एजेंट स्वामित्व के लिए `{ scope: "agent", agentId }` चुनें। कॉलबैक के `AbortSignal` को प्रत्येक विफल हो सकने वाले ऑपरेशन में अग्रेषित करें। `assertOwned()` अगला महत्वपूर्ण चरण शुरू करने से पहले समय-बिंदु चेकपॉइंट है; होस्ट कॉलबैक के बाद भी स्वामित्व सत्यापित करता है। लीज़ खोने या कॉलर के रद्द करने पर सिग्नल निरस्त हो जाता है। अधिग्रहण की प्रतीक्षा और Heartbeat छोटे समकालिक SQLite ट्रांज़ैक्शन के बाहर होते हैं; Plugins को कभी डेटाबेस पथ या हैंडल नहीं मिलते। यह सहयोगात्मक रद्दीकरण है, फ़ेंसिंग टोकन या बिना फ़ेंस वाले बाहरी लेखन के लिए प्राधिकरण नहीं।

    `openChannelIngressDrain(...)` उस कतार पर कोर चैनल-अज्ञेय वर्कर खोलता है (या कोई कतार न दिए जाने पर उसे बनाता है)। ड्रेन पुराने दावे की पुनर्प्राप्ति, प्रति-लेन दावा क्रमांकन, अंगीकरण पर पूर्णता या डिस्पैच-वापसी पर पूर्णता, पुनः प्रयास/डेड-लेटर निपटान, वैकल्पिक पूर्व-अंगीकरण प्रतिस्थापन और दावा→अंगीकरण ठहराव टाइमआउट का स्वामी होता है। `turnAdoptionLifecycle` के साथ दावा स्वामित्व को उत्तर निर्माण से जोड़ें (`plugin-sdk/channel-outbound` से `bindIngressLifecycleToReplyOptions` के माध्यम से)। चैनल Plugins स्वीकार-पक्ष एनक्यू, लेन व्युत्पत्ति, पुनः प्रयास न किए जाने योग्य वर्गीकरण और किसी भी प्रतिस्थापन प्राधिकरण नीति को बनाए रखते हैं।

    <Warning>
    इस रिलीज़ में `openBlobStore`, `openKeyedStore`, `openSyncKeyedStore`, `withLease`, `openChannelIngressQueue` और `openChannelIngressDrain` केवल बंडल किए गए Plugins और विश्वसनीय आधिकारिक Plugin इंस्टॉलेशन के लिए उपलब्ध हैं।
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.channel">
    चैनल-विशिष्ट रनटाइम सहायक (चैनल Plugin लोड होने पर उपलब्ध)। उद्देश्य के अनुसार समूहित:

    | समूह | उद्देश्य |
    | --- | --- |
    | `text` | खंडन (`chunkText`, `chunkMarkdownText`, `resolveChunkMode`), नियंत्रण-कमांड पहचान, Markdown तालिका रूपांतरण। |
    | `reply` | बफ़र-ब्लॉक उत्तर डिस्पैच, एनवलप स्वरूपण, प्रभावी संदेश/मानवीय-विलंब कॉन्फ़िग रिज़ॉल्यूशन। |
    | `routing` | `buildAgentSessionKey`, `resolveAgentRoute`। |
    | `pairing` | `buildPairingReply`, अनुमति-सूची पढ़ना/हटाना, पेयरिंग-अनुरोध अपसर्ट और अनुरोध से प्राप्त अनुमोदन प्रविष्टियाँ। |
    | `media` | दूरस्थ मीडिया डाउनलोड/सहेजना (नीचे देखें)। |
    | `activity` | अंतिम चैनल गतिविधि रिकॉर्ड करना/पढ़ना। |
    | `session` | इनबाउंड इवेंट से सत्र मेटाडेटा, अंतिम-रूट अपडेट। |
    | `mentions` | उल्लेख-नीति सहायक (नीचे देखें)। |
    | `reactions` | प्रक्रियाधीन संकेतकों के लिए अभिस्वीकृति-प्रतिक्रिया हैंडल। |
    | `groups` | समूह नीति और उल्लेख-आवश्यकता रिज़ॉल्यूशन। |
    | `debounce` | इनबाउंड संदेश डिबाउंसिंग। |
    | `commands` | कमांड प्राधिकरण और टेक्स्ट-कमांड गेटिंग। |
    | `outbound` | चैनल का आउटबाउंड अडैप्टर लोड करना। |
    | `inbound` | इनबाउंड इवेंट संदर्भ बनाना और साझा इनबाउंड-इवेंट/उत्तर कर्नेल चलाना। |
    | `threadBindings` | बाउंड सत्र थ्रेड के लिए निष्क्रियता-टाइमआउट/अधिकतम-आयु समायोजित करना। |
    | `runtimeContexts` | प्रक्रिया-स्थानीय प्रति-चैनल/खाता/क्षमता संदर्भ पंजीकृत करना, पढ़ना और देखना। |

    चैनल मीडिया डाउनलोड और स्टोरेज के लिए `api.runtime.channel.media` पसंदीदा सतह है:

    ```typescript
    const saved = await api.runtime.channel.media.saveRemoteMedia({
      url,
      subdir: "inbound",
      maxBytes,
      filePathHint: fileName,
    });
    ```

    जब किसी दूरस्थ URL को OpenClaw मीडिया बनाना हो, तब `saveRemoteMedia(...)` का उपयोग करें। जब Plugin ने पहले ही Plugin-स्वामित्व वाले प्रमाणीकरण, रीडायरेक्ट या अनुमति-सूची प्रबंधन के साथ `Response` प्राप्त कर लिया हो, तब `saveResponseMedia(...)` का उपयोग करें। `readRemoteMediaBuffer(...)` का उपयोग केवल तब करें जब Plugin को निरीक्षण, रूपांतरण, डिक्रिप्शन या पुनः अपलोड के लिए रॉ बाइट चाहिए। `fetchRemoteMedia(...)`, `readRemoteMediaBuffer(...)` के लिए एक बहिष्कृत संगतता उपनाम बना हुआ है।

    `api.runtime.channel.mentions` उन बंडल किए गए चैनल Plugins के लिए साझा इनबाउंड उल्लेख-नीति सतह है, जो रनटाइम इंजेक्शन का उपयोग करते हैं:

    ```typescript
    const mentionMatch = api.runtime.channel.mentions.matchesMentionWithExplicit(text, {
      mentionRegexes,
      mentionPatterns,
    });

    const decision = api.runtime.channel.mentions.resolveInboundMentionDecision({
      facts: {
        canDetectMention: true,
        wasMentioned: mentionMatch.matched,
        implicitMentionKinds: api.runtime.channel.mentions.implicitMentionKindWhen(
          "reply_to_bot",
          isReplyToBot,
        ),
      },
      policy: {
        isGroup,
        requireMention,
        allowTextCommands,
        hasControlCommand,
        commandAuthorized,
      },
    });
    ```

    उपलब्ध उल्लेख सहायक:

    - `buildMentionRegexes`
    - `matchesMentionPatterns`
    - `matchesMentionWithExplicit`
    - `implicitMentionKindWhen`
    - `resolveInboundMentionDecision`

    उल्लेख निर्णयों के लिए सामान्यीकृत `{ facts, policy }` पथ का उपयोग करें।

    `reply`, `session` और `inbound` के अंतर्गत कई फ़ील्ड में प्रति-फ़ील्ड `@deprecated` नोट हैं, जो वर्तमान चैनल-टर्न कर्नेल या चैनल-आउटबाउंड अडैप्टर की ओर संकेत करते हैं; उस पर नया कोड बनाने से पहले विशिष्ट सहायक पर इनलाइन JSDoc देखें।

  </Accordion>
</AccordionGroup>

## रनटाइम संदर्भ संग्रहीत करना

`register` कॉलबैक के बाहर उपयोग के लिए रनटाइम संदर्भ संग्रहीत करने हेतु `createPluginRuntimeStore` का उपयोग करें:

<Steps>
  <Step title="स्टोर बनाएँ">
    ```typescript
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

    const store = createPluginRuntimeStore<PluginRuntime>({
      pluginId: "my-plugin",
      errorMessage: "my-plugin runtime not initialized",
    });
    ```

  </Step>
  <Step title="एंट्री पॉइंट से जोड़ें">
    ```typescript
    export default defineChannelPluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Example",
      plugin: myPlugin,
      setRuntime: store.setRuntime,
    });
    ```
  </Step>
  <Step title="अन्य फ़ाइलों से एक्सेस करें">
    ```typescript
    export function getRuntime() {
      return store.getRuntime(); // आरंभ न होने पर त्रुटि देता है
    }

    export function tryGetRuntime() {
      return store.tryGetRuntime(); // आरंभ न होने पर null लौटाता है
    }
    ```

  </Step>
</Steps>

<Note>
रनटाइम-स्टोर पहचान के लिए `pluginId` को प्राथमिकता दें। निम्न-स्तरीय `key` स्वरूप उन असामान्य मामलों के लिए है, जहाँ एक Plugin को जानबूझकर एक से अधिक रनटाइम स्लॉट की आवश्यकता होती है।
</Note>

## अन्य शीर्ष-स्तरीय `api` फ़ील्ड

`api.runtime` के अतिरिक्त, API ऑब्जेक्ट यह भी प्रदान करता है:

<ParamField path="api.id" type="string">
  Plugin आईडी।
</ParamField>
<ParamField path="api.name" type="string">
  Plugin का प्रदर्शन नाम।
</ParamField>
<ParamField path="api.config" type="OpenClawConfig">
  वर्तमान कॉन्फ़िगरेशन स्नैपशॉट (उपलब्ध होने पर सक्रिय इन-मेमोरी रनटाइम स्नैपशॉट)।
</ParamField>
<ParamField path="api.pluginConfig" type="Record<string, unknown>">
  `plugins.entries.<id>.config` से Plugin-विशिष्ट कॉन्फ़िगरेशन।
</ParamField>
<ParamField path="api.logger" type="PluginLogger">
  सीमित दायरे वाला लॉगर (`debug`, `info`, `warn`, `error`)।
</ParamField>
<ParamField path="api.registrationMode" type="PluginRegistrationMode">
  वर्तमान लोड मोड: `"full"` (लाइव सक्रियण), `"discovery"` / `"tool-discovery"` (केवल-पढ़ने योग्य क्षमता खोज), `"setup-only"` (हल्की सेटअप प्रविष्टि), `"setup-runtime"` (सेटअप प्रवाह जिसे रनटाइम चैनल प्रविष्टि की भी आवश्यकता होती है), या `"cli-metadata"` (CLI कमांड मेटाडेटा संग्रह)।
</ParamField>
<ParamField path="api.resolvePath(input)" type="(string) => string">
  Plugin रूट के सापेक्ष पथ का समाधान करें।
</ParamField>

## संबंधित

- [Plugin की आंतरिक संरचना](/hi/plugins/architecture) — क्षमता मॉडल और रजिस्ट्री
- [SDK प्रवेश बिंदु](/hi/plugins/sdk-entrypoints) — `definePluginEntry` विकल्प
- [SDK अवलोकन](/hi/plugins/sdk-overview) — उपपथ संदर्भ
