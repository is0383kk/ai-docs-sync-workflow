---
read_when:
    - आपको फ़ील्ड-स्तरीय कॉन्फ़िगरेशन के सटीक अर्थ या डिफ़ॉल्ट मान चाहिए
    - आप चैनल, मॉडल, Gateway या टूल कॉन्फ़िगरेशन ब्लॉक का सत्यापन कर रहे हैं
summary: मुख्य OpenClaw कुंजियों, डिफ़ॉल्ट मानों और समर्पित उप-प्रणाली संदर्भों के लिंक के लिए Gateway कॉन्फ़िगरेशन संदर्भ
title: कॉन्फ़िगरेशन संदर्भ
x-i18n:
    generated_at: "2026-07-27T18:06:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7135554fda444fd1b8c072af5768c53a165f7be2dcd12a7909fc7fd4bd864428
    source_path: gateway/configuration-reference.md
    workflow: 16
---

`~/.openclaw/openclaw.json` के लिए फ़ील्ड-स्तरीय संदर्भ: कुंजियाँ, डिफ़ॉल्ट और अधिक विस्तृत सबसिस्टम पृष्ठों के लिंक। कार्य-उन्मुख सेटअप मार्गदर्शन के लिए, [कॉन्फ़िगरेशन](/hi/gateway/configuration) देखें। चैनल और Plugin के स्वामित्व वाले कमांड कैटलॉग तथा विस्तृत मेमोरी/QMD विकल्प उनके अपने पृष्ठों पर हैं, यहाँ नहीं।

कॉन्फ़िगरेशन प्रारूप **JSON5** है (टिप्पणियाँ + अंतिम कॉमा अनुमत हैं)। सभी फ़ील्ड वैकल्पिक हैं; उन्हें छोड़ने पर OpenClaw सुरक्षित डिफ़ॉल्ट का उपयोग करता है।

कोड की वास्तविकता इस पृष्ठ से अधिक प्रामाणिक है:

- `openclaw config schema` सत्यापन और Control UI के लिए उपयोग की जाने वाली लाइव JSON Schema प्रिंट करता है, जिसमें बंडल किए गए/Plugin/चैनल मेटाडेटा मर्ज होते हैं।
- कॉन्फ़िगरेशन संपादित करने से पहले एजेंटों को किसी एक सटीक पथ-स्कोप्ड स्कीमा नोड के लिए `gateway` टूल ऐक्शन `config.schema.lookup` को कॉल करना चाहिए।
- `pnpm config:docs:check` / `pnpm config:docs:gen` वर्तमान स्कीमा सतह के विरुद्ध इस दस्तावेज़ के बेसलाइन हैश को सत्यापित करते हैं।

स्कीमा `uiHints` प्रत्येक पथ के लिए एक रिज़ॉल्व किया गया `advanced` बूलियन भी रखते हैं।
Control UI इसका उपयोग सामान्य फ़ील्ड पहले दिखाने और प्रत्येक
अनुभाग के उन्नत फ़ील्ड संक्षिप्त करने के लिए करता है; खोज फिर भी दोनों स्तरों में होती है। स्तर मेटाडेटा केवल प्रस्तुतिकरण के लिए है।
कुंजी जोड़ते समय, उसका स्तर लीफ़ पर घोषित करें या उसे निकटतम
पूर्वज घोषणा से इनहेरिट होने दें। जिस पथ का कोई घोषित पूर्वज नहीं है, वह डिफ़ॉल्ट रूप से उन्नत होता है।

समर्पित विस्तृत संदर्भ:

- `memory.search.*`, `memory.qmd.*`, `memory.citations` और `plugins.entries.memory-core.config.dreaming` के अंतर्गत Dreaming कॉन्फ़िगरेशन के लिए [मेमोरी कॉन्फ़िगरेशन संदर्भ](/hi/reference/memory-config)।
- वर्तमान अंतर्निहित + बंडल किए गए कमांड कैटलॉग के लिए [स्लैश कमांड](/hi/tools/slash-commands)।
- चैनल-विशिष्ट कमांड सतहों के लिए स्वामी चैनल/Plugin पृष्ठ।

---

## चैनल

प्रति-चैनल कॉन्फ़िगरेशन कुंजियाँ [कॉन्फ़िगरेशन - चैनल](/hi/gateway/config-channels) में हैं: Slack, Discord, Telegram, WhatsApp, Matrix, iMessage और अन्य बंडल किए गए चैनलों के लिए `channels.*` (प्रमाणीकरण, अभिगम नियंत्रण, बहु-खाता, उल्लेख गेटिंग)।

## एजेंट डिफ़ॉल्ट, बहु-एजेंट, सत्र और संदेश

इनके लिए [कॉन्फ़िगरेशन - एजेंट](/hi/gateway/config-agents) देखें:

- `agents.defaults.*` (वर्कस्पेस, मॉडल, चिंतन, Heartbeat, मेमोरी, मीडिया, Skills, सैंडबॉक्स)
- `multiAgent.*` (बहु-एजेंट रूटिंग और बाइंडिंग)
- `session.*` (सत्र जीवनचक्र, Compaction, प्रूनिंग)
- `messages.*` (संदेश डिलीवरी, TTS, Markdown रेंडरिंग)
- `talk.*` (Talk मोड)
  - `talk.consultThinkingLevel`: Control UI Talk के रीयलटाइम परामर्श के पीछे चलने वाले पूर्ण OpenClaw एजेंट रन के लिए चिंतन स्तर ओवरराइड
  - `talk.consultFastMode`: Control UI Talk रीयलटाइम परामर्शों के लिए एक-बार का फ़ास्ट-मोड ओवरराइड
  - `talk.speechLocale`: Android, iOS और macOS पर Talk वाक् पहचान के लिए वैकल्पिक BCP 47 लोकेल आईडी
  - `talk.silenceTimeoutMs`: सेट न होने पर, Talk ट्रांसक्रिप्ट भेजने से पहले प्लेटफ़ॉर्म की डिफ़ॉल्ट विराम अवधि बनाए रखता है (`700 ms on macOS and Android, 900 ms on iOS`)
  - `talk.realtime.consultRouting`: `openclaw_agent_consult` को छोड़ने वाले अंतिम रूप दिए गए रीयलटाइम Talk ट्रांसक्रिप्ट के लिए Gateway रिले फ़ॉलबैक

## टूल और कस्टम प्रदाता

टूल नीति, प्रयोगात्मक टॉगल, प्रदाता-समर्थित टूल कॉन्फ़िगरेशन और कस्टम
प्रदाता / बेस-URL सेटअप
[कॉन्फ़िगरेशन - टूल और कस्टम प्रदाता](/hi/gateway/config-tools) में हैं।

## मॉडल

प्रदाता परिभाषाएँ, मॉडल अनुमत-सूचियाँ और कस्टम प्रदाता सेटअप
[कॉन्फ़िगरेशन - टूल और कस्टम प्रदाता](/hi/gateway/config-tools#custom-providers-and-base-urls) में हैं।
`models` रूट वैश्विक मॉडल-कैटलॉग व्यवहार का भी स्वामी है।

```json5
{
  models: {
    // वैकल्पिक। डिफ़ॉल्ट: true। बदलने पर Gateway पुनः आरंभ करना आवश्यक है।
    pricing: { enabled: false },
  },
}
```

- `models.mode`: प्रदाता कैटलॉग व्यवहार (`merge` या `replace`)।
- `models.providers`: प्रदाता आईडी द्वारा कुंजीबद्ध कस्टम प्रदाता मैप।
- `models.providers.*.localService`: स्थानीय मॉडल सर्वरों के लिए वैकल्पिक ऑन-डिमांड प्रक्रिया प्रबंधक। OpenClaw कॉन्फ़िगर किए गए स्वास्थ्य एंडपॉइंट की जाँच करता है, आवश्यकता होने पर निरपेक्ष `command` शुरू करता है, तत्परता की प्रतीक्षा करता है और फिर मॉडल अनुरोध भेजता है। [स्थानीय मॉडल सेवाएँ](/hi/gateway/local-model-services) देखें।
- `models.pricing.enabled`: साइडकार और चैनलों के Gateway तैयार पथ तक पहुँचने के बाद शुरू होने वाले पृष्ठभूमि मूल्य निर्धारण बूटस्ट्रैप को नियंत्रित करता है। जब `false` हो, तो Gateway OpenRouter और LiteLLM मूल्य निर्धारण-कैटलॉग फ़ेच छोड़ देता है; कॉन्फ़िगर किए गए `models.providers.*.models[].cost` मान स्थानीय लागत अनुमानों के लिए फिर भी काम करते हैं।

## MCP

OpenClaw द्वारा प्रबंधित MCP सर्वर परिभाषाएँ `mcp.servers` के अंतर्गत होती हैं और
एम्बेडेड OpenClaw तथा अन्य रनटाइम अडैप्टर उनका उपयोग करते हैं। `openclaw mcp list`,
`show`, `set` और `unset` कमांड कॉन्फ़िगरेशन संपादन के दौरान लक्ष्य सर्वर से कनेक्ट हुए बिना इस ब्लॉक का प्रबंधन करते हैं।

```json5
{
  mcp: {
    servers: {
      docs: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-fetch"],
      },
      remote: {
        url: "https://example.com/mcp",
        transport: "streamable-http", // streamable-http | sse
        requestTimeoutMs: 20000,
        connectionTimeoutMs: 5000,
        supportsParallelToolCalls: true,
        headers: {
          Authorization: "Bearer ${MCP_REMOTE_TOKEN}",
        },
        auth: "oauth",
        oauth: {
          scope: "docs.read",
        },
        sslVerify: true,
        clientCert: "/path/to/client.crt",
        clientKey: "/path/to/client.key",
        toolFilter: {
          include: ["search_*"],
          exclude: ["admin_*"],
        },
        // वैकल्पिक Codex app-server प्रोजेक्शन नियंत्रण।
        codex: {
          agents: ["main"],
          defaultToolsApprovalMode: "approve", // auto | prompt | approve
        },
      },
    },
  },
}
```

- `mcp.servers`: कॉन्फ़िगर किए गए MCP टूल उपलब्ध कराने वाले रनटाइम के लिए नामित stdio या रिमोट MCP सर्वर परिभाषाएँ।
  रिमोट प्रविष्टियाँ `transport: "streamable-http"` या `transport: "sse"` का उपयोग करती हैं;
  `type: "http"` एक CLI-मूल उपनाम है जिसे `openclaw mcp set` और
  `openclaw doctor --fix` कैनोनिकल `transport` फ़ील्ड में सामान्यीकृत करते हैं।
- `mcp.servers.<name>.enabled`: सहेजी गई सर्वर परिभाषा को बनाए रखते हुए उसे एम्बेडेड OpenClaw MCP खोज और टूल प्रोजेक्शन से बाहर रखने के लिए `false` सेट करें।
- `mcp.servers.<name>.requestTimeoutMs`: प्रति-सर्वर MCP अनुरोध टाइमआउट, मिलीसेकंड में।
- `mcp.servers.<name>.connectionTimeoutMs`: प्रति-सर्वर कनेक्शन टाइमआउट, मिलीसेकंड में।
- `mcp.servers.<name>.supportsParallelToolCalls`: उन अडैप्टरों के लिए वैकल्पिक समवर्तीता संकेत जो यह चुन सकते हैं कि समानांतर MCP टूल कॉल जारी किए जाएँ या नहीं।
- `mcp.servers.<name>.auth`: OAuth की आवश्यकता वाले HTTP MCP सर्वरों के लिए `"oauth"` सेट करें। OpenClaw स्थिति के अंतर्गत टोकन संग्रहीत करने के लिए `openclaw mcp login <name>` चलाएँ।
- `mcp.servers.<name>.oauth`: वैकल्पिक OAuth स्कोप, रीडायरेक्ट URL और क्लाइंट मेटाडेटा URL ओवरराइड।
- `mcp.servers.<name>.sslVerify`, `clientCert`, `clientKey`: निजी एंडपॉइंट और पारस्परिक TLS के लिए HTTP TLS नियंत्रण।
- `mcp.servers.<name>.toolFilter`: वैकल्पिक प्रति-सर्वर टूल चयन। `include`
  खोजे गए MCP टूल को मेल खाने वाले नामों तक सीमित करता है; `exclude` मेल खाने वाले
  नामों को छिपाता है। प्रविष्टियाँ सटीक MCP टूल नाम या सरल `*` ग्लॉब होती हैं। संसाधन या प्रॉम्प्ट वाले सर्वर उपयोगिता टूल नाम भी उत्पन्न करते हैं (`resources_list`,
  `resources_read`, `prompts_list`, `prompts_get`) और उन नामों पर भी यही
  फ़िल्टर लागू होता है।
- `mcp.servers.<name>.codex`: वैकल्पिक Codex app-server प्रोजेक्शन नियंत्रण।
  यह ब्लॉक केवल Codex app-server थ्रेड के लिए OpenClaw मेटाडेटा है; यह
  ACP सत्रों, सामान्य Codex हार्नेस कॉन्फ़िगरेशन या अन्य रनटाइम अडैप्टर को
  प्रभावित नहीं करता। गैर-रिक्त `codex.agents` सर्वर को सूचीबद्ध OpenClaw एजेंट आईडी तक सीमित करता है।
  खाली, रिक्त या अमान्य स्कोप्ड एजेंट सूचियाँ कॉन्फ़िगरेशन सत्यापन द्वारा अस्वीकार की जाती हैं
  और वैश्विक बनने के बजाय रनटाइम प्रोजेक्शन पथ से छोड़ दी जाती हैं।
  `codex.defaultToolsApprovalMode` उस सर्वर के लिए Codex का मूल
  `default_tools_approval_mode` उत्सर्जित करता है। मूल `mcp_servers` कॉन्फ़िगरेशन Codex को भेजने से पहले OpenClaw `codex`
  ब्लॉक हटा देता है। Codex के
  डिफ़ॉल्ट MCP अनुमोदन व्यवहार के साथ प्रत्येक Codex app-server एजेंट के लिए सर्वर को प्रोजेक्टेड रखने हेतु ब्लॉक छोड़ दें।
- सत्र-स्कोप्ड बंडल किए गए MCP रनटाइम अंतर्निहित 10-मिनट निष्क्रिय TTL का उपयोग करते हैं।
  एक-बार के एम्बेडेड रन, रन समाप्ति पर क्लीनअप का अनुरोध करते हैं; TTL दीर्घजीवी सत्रों और भविष्य के कॉलरों के लिए सुरक्षा उपाय है।
- `mcp.*` के अंतर्गत बदलाव कैश किए गए सत्र MCP रनटाइम को निस्तारित करके तुरंत लागू होते हैं।
  अगली टूल खोज/उपयोग उन्हें नए कॉन्फ़िगरेशन से पुनः बनाती है, इसलिए हटाई गई
  `mcp.servers` प्रविष्टियाँ निष्क्रिय TTL की प्रतीक्षा करने के बजाय तुरंत साफ़ हो जाती हैं।
- रनटाइम खोज उस सत्र के कैश किए गए कैटलॉग को हटाकर MCP टूल-सूची परिवर्तन सूचनाओं का भी सम्मान करती है। संसाधन या
  प्रॉम्प्ट घोषित करने वाले सर्वरों को संसाधनों को सूचीबद्ध/पढ़ने और प्रॉम्प्ट को सूचीबद्ध/प्राप्त करने के लिए उपयोगिता टूल मिलते हैं। बार-बार टूल-कॉल विफलताएँ एक और कॉल का प्रयास करने से पहले प्रभावित सर्वर को कुछ समय के लिए रोक देती हैं।

रनटाइम व्यवहार के लिए [MCP](/hi/cli/mcp#openclaw-as-an-mcp-client-registry) और
[CLI बैकएंड](/hi/gateway/cli-backends#bundle-mcp-overlays) देखें।

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
      allowUploadedArchives: false,
    },
    workshop: {
      allowSymlinkTargetWrites: false,
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // या प्लेनटेक्स्ट स्ट्रिंग
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: केवल बंडल किए गए Skills के लिए वैकल्पिक अनुमत-सूची (प्रबंधित/वर्कस्पेस Skills अप्रभावित रहते हैं)।
- `load.extraDirs`: अतिरिक्त साझा Skill रूट (सबसे कम प्राथमिकता)।
- `load.allowSymlinkTargets`: विश्वसनीय वास्तविक लक्ष्य रूट जिनमें Skill सिमलिंक रिज़ॉल्व हो सकते हैं, जब लिंक उसके कॉन्फ़िगर किए गए स्रोत रूट से बाहर हो।
- `workshop.allowSymlinkTargetWrites`: Skill Workshop के लागू करने की प्रक्रिया को पहले से विश्वसनीय सिमलिंक लक्ष्यों के माध्यम से लिखने की अनुमति देता है (डिफ़ॉल्ट: false)।
- `install.preferBrew`: true होने पर, अन्य इंस्टॉलर प्रकारों पर वापस जाने से पहले `brew` उपलब्ध होने पर Homebrew इंस्टॉलर को प्राथमिकता दें।
- `install.nodeManager`: `metadata.openclaw.install` विनिर्देशों के लिए Node इंस्टॉलर प्राथमिकता (`npm` | `pnpm` | `yarn` | `bun`)।
- `install.allowUploadedArchives`: विश्वसनीय `operator.admin` Gateway क्लाइंट को `skills.upload.*` के माध्यम से स्टेज किए गए निजी zip अभिलेख स्थापित करने की अनुमति देता है (डिफ़ॉल्ट: false)। यह केवल अपलोड किए गए अभिलेख का पथ सक्षम करता है; सामान्य ClawHub इंस्टॉलेशन के लिए इसकी आवश्यकता नहीं होती।
- `entries.<skillKey>.enabled: false` किसी Skill को बंडल/इंस्टॉल होने पर भी अक्षम करता है।
- `entries.<skillKey>.apiKey`: प्राथमिक एनवायरनमेंट वेरिएबल घोषित करने वाले Skills के लिए सुविधा (प्लेनटेक्स्ट स्ट्रिंग या SecretRef ऑब्जेक्ट)।
- `limits.maxCandidatesPerRoot`, `limits.maxSkillsLoadedPerSource`, `limits.maxSkillsInPrompt`, `limits.maxSkillsPromptChars`, `limits.maxSkillFileBytes`: Skill खोज और मॉडल-सामना करने वाले Skills प्रॉम्प्ट को सीमित करते हैं।
- Skill Workshop की स्वायत्तता/अनुमोदन सेटिंग्स (`workshop.autonomous.enabled`, `workshop.approvalPolicy`, `workshop.maxPending`, `workshop.maxSkillBytes`) [Skills कॉन्फ़िगरेशन](/hi/tools/skills-config) में प्रलेखित हैं।

---

## Plugins

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-plugin"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- `~/.openclaw/extensions` और `<workspace>/.openclaw/extensions` के अंतर्गत पैकेज या बंडल निर्देशिकाओं से, साथ ही `plugins.load.paths` में सूचीबद्ध फ़ाइलों या निर्देशिकाओं से लोड किया जाता है।
- स्वतंत्र Plugin फ़ाइलों को `plugins.load.paths` में रखें; स्वतः खोजे गए एक्सटेंशन रूट शीर्ष-स्तरीय `.js`, `.mjs`, और `.ts` फ़ाइलों को अनदेखा करते हैं, ताकि उन रूट में मौजूद सहायक स्क्रिप्ट स्टार्टअप को अवरुद्ध न करें।
- खोज मूल OpenClaw plugins के साथ संगत Codex बंडल और Claude बंडल भी स्वीकार करती है, जिनमें मैनिफ़ेस्ट-रहित Claude डिफ़ॉल्ट-लेआउट बंडल शामिल हैं।
- **कॉन्फ़िगरेशन परिवर्तनों के लिए Gateway को पुनः आरंभ करना आवश्यक है।**
- `allow`: वैकल्पिक अनुमति-सूची (केवल सूचीबद्ध plugins लोड होते हैं)। `deny` को प्राथमिकता मिलती है।
- `plugins.entries.<id>.apiKey`: Plugin-स्तरीय API कुंजी का सुविधाजनक फ़ील्ड (जब Plugin इसका समर्थन करता हो)।
- `plugins.entries.<id>.env`: Plugin-स्कोप वाला पर्यावरण चर मैप।
- `plugins.entries.<id>.hooks.allowPromptInjection`: जब `false` हो, तब कोर `before_prompt_build` जैसे प्रॉम्प्ट बदलने वाले हुक को अवरुद्ध करता है। यह मूल Plugin हुक और समर्थित बंडल द्वारा प्रदान की गई हुक निर्देशिकाओं पर लागू होता है।
- `plugins.entries.<id>.hooks.allowConversationAccess`: जब `true` हो, तब विश्वसनीय गैर-बंडल plugins `llm_input`, `llm_output`, `before_model_resolve`, `before_agent_reply`, `before_agent_run`, `before_agent_finalize`, और `agent_end` जैसे टाइप किए गए हुक से अपरिष्कृत वार्तालाप सामग्री पढ़ सकते हैं।
- `plugins.entries.<id>.subagent.allowModelOverride`: पृष्ठभूमि सबएजेंट रन के लिए प्रति-रन `provider` और `model` ओवरराइड का अनुरोध करने हेतु इस Plugin पर स्पष्ट रूप से भरोसा करें।
- `plugins.entries.<id>.subagent.allowedModels`: विश्वसनीय सबएजेंट ओवरराइड के लिए मानक `provider/model` लक्ष्यों की वैकल्पिक अनुमति-सूची। `"*"` का उपयोग केवल तभी करें, जब आप जानबूझकर किसी भी मॉडल को अनुमति देना चाहते हों।
- `plugins.entries.<id>.llm.allowModelOverride`: `api.runtime.llm.complete` के लिए मॉडल ओवरराइड का अनुरोध करने हेतु इस Plugin पर स्पष्ट रूप से भरोसा करें।
- `plugins.entries.<id>.llm.allowedModels`: विश्वसनीय Plugin LLM पूर्णता ओवरराइड के लिए मानक `provider/model` लक्ष्यों की वैकल्पिक अनुमति-सूची। `"*"` का उपयोग केवल तभी करें, जब आप जानबूझकर किसी भी मॉडल को अनुमति देना चाहते हों।
- `plugins.entries.<id>.llm.allowAgentIdOverride`: किसी गैर-डिफ़ॉल्ट एजेंट आईडी के लिए `api.runtime.llm.complete` चलाने हेतु इस Plugin पर स्पष्ट रूप से भरोसा करें।
- `plugins.entries.<id>.config`: Plugin द्वारा परिभाषित कॉन्फ़िगरेशन ऑब्जेक्ट (उपलब्ध होने पर मूल OpenClaw Plugin स्कीमा द्वारा सत्यापित)।
- चैनल Plugin खाते/रनटाइम की सेटिंग्स `channels.<id>` के अंतर्गत रहती हैं और उनका वर्णन किसी केंद्रीय OpenClaw विकल्प रजिस्ट्री के बजाय स्वामी Plugin के मैनिफ़ेस्ट की `channelConfigs` मेटाडेटा द्वारा किया जाना चाहिए।

### Codex हार्नेस Plugin कॉन्फ़िगरेशन

बंडल किया गया `codex` Plugin मूल Codex ऐप-सर्वर हार्नेस सेटिंग्स का स्वामी है, जो
`plugins.entries.codex.config` के अंतर्गत होती हैं। पूर्ण कॉन्फ़िगरेशन
सतह के लिए [Codex हार्नेस संदर्भ](/hi/plugins/codex-harness-reference) और रनटाइम मॉडल के लिए
[Codex हार्नेस](/hi/plugins/codex-harness) देखें।

`codexPlugins` केवल उन सत्रों पर लागू होता है जो मूल Codex हार्नेस चुनते हैं।
यह OpenClaw प्रदाता रन, ACP
वार्तालाप बाइंडिंग या किसी गैर-Codex हार्नेस के लिए Codex plugins सक्षम नहीं करता।

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
                allow_destructive_actions: false,
              },
            },
          },
        },
      },
    },
  },
}
```

- `plugins.entries.codex.config.codexPlugins.enabled`: Codex हार्नेस के लिए मूल Codex
  Plugin/ऐप समर्थन सक्षम करता है। डिफ़ॉल्ट: `false`।
- `plugins.entries.codex.config.codexPlugins.allow_all_plugins`: प्रमाणित Codex खाते से जुड़ा प्रत्येक
  वर्तमान में सुलभ ऐप, हर नए मूल Codex थ्रेड में उपलब्ध कराता है।
  डिफ़ॉल्ट: `false`।
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions`:
  कॉन्फ़िगर किए गए Plugin ऐप अनुरोधों के लिए डिफ़ॉल्ट विनाशकारी-कार्रवाई नीति।
  सुरक्षित Codex अनुमोदन स्कीमा को बिना पूछे स्वीकार करने के लिए `true`, उन्हें अस्वीकार करने के लिए `false`,
  Codex द्वारा आवश्यक अनुमोदनों को OpenClaw Plugin अनुमोदनों के माध्यम से भेजने के लिए `"auto"`,
  या स्थायी अनुमोदन के बिना प्रत्येक Plugin लेखन/विनाशकारी
  कार्रवाई के लिए पूछने हेतु `"ask"` का उपयोग करें। `"ask"` मोड प्रभावित ऐप के लिए स्थायी Codex
  प्रति-टूल अनुमोदन ओवरराइड साफ़ करता है और Codex थ्रेड आरंभ होने से पहले उस ऐप के लिए मानव
  अनुमोदन समीक्षक चुनता है।
  डिफ़ॉल्ट: `true`।
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.enabled`: वैश्विक `codexPlugins.enabled` भी सत्य होने पर
  कॉन्फ़िगर की गई Plugin प्रविष्टि सक्षम करता है।
  स्पष्ट प्रविष्टियों के लिए डिफ़ॉल्ट: `true`।
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.marketplaceName`:
  स्थिर मार्केटप्लेस पहचान, प्रत्येक हल की गई प्रविष्टि के लिए `pluginName` के साथ आवश्यक।
  `"openai-curated"` और `"workspace-directory"` का समर्थन करता है। जिन प्रविष्टियों में
  कोई भी पहचान फ़ील्ड अनुपस्थित हो, उन्हें अनदेखा कर दिया जाता है।
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.pluginName`: स्थिर
  Codex Plugin पहचान, `marketplaceName` के साथ आवश्यक। किसी
  `workspace-directory` प्रविष्टि को `plugin/list` द्वारा लौटाया गया ठीक वही मार्केटप्लेस-योग्य
  `summary.id` उपयोग करना होगा, उदाहरण के लिए
  `"example-plugin@workspace-directory"`।
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.allow_destructive_actions`:
  प्रति-Plugin विनाशकारी-कार्रवाई ओवरराइड। इसे छोड़ने पर वैश्विक
  `allow_destructive_actions` मान का उपयोग होता है। प्रति-Plugin मान समान
  `true`, `false`, `"auto"`, या `"ask"` नीतियाँ स्वीकार करता है।

`"ask"` का उपयोग करने वाला प्रत्येक स्वीकृत Plugin ऐप उस ऐप के अनुमोदन अनुरोधों को
मानव समीक्षक तक भेजता है। अन्य ऐप और गैर-ऐप थ्रेड अनुमोदन अपने
कॉन्फ़िगर किए गए समीक्षक को बनाए रखते हैं, इसलिए मिश्रित Plugin नीतियाँ `"ask"` व्यवहार को विरासत में नहीं लेतीं।

`codexPlugins.enabled` वैश्विक सक्षमता निर्देश है। माइग्रेशन द्वारा लिखी गई स्पष्ट Plugin
प्रविष्टियाँ स्थायी रूप से चयनित इंस्टॉल और मरम्मत
पात्रता सेट हैं। मैन्युअल रूप से कॉन्फ़िगर की गई `workspace-directory` प्रविष्टियाँ पहले से
इंस्टॉल और सक्षम होनी चाहिए, और उनके स्वामित्व वाले ऐप सुलभ होने चाहिए; OpenClaw
उन्हें इंस्टॉल या प्रमाणित नहीं करता। यदि Codex स्पष्ट कार्यस्थान
कैटलॉग अनुरोध अस्वीकार करता है, तो सक्षम कार्यस्थान प्रविष्टियाँ
`marketplace_missing` के साथ सुरक्षित रूप से विफल होती हैं, जबकि डिफ़ॉल्ट कैटलॉग की चयनित प्रविष्टियाँ
उपलब्ध रहती हैं। `plugins["*"]` समर्थित नहीं है, कोई `install` स्विच नहीं है, और
स्थानीय `marketplacePath` मान जानबूझकर कॉन्फ़िगरेशन फ़ील्ड नहीं हैं, क्योंकि वे
होस्ट-विशिष्ट हैं। ऐप-सर्वर संस्करण और
तत्परता आवश्यकताओं के लिए [मूल Codex plugins](/hi/plugins/codex-native-plugins) देखें।

`app/list` तत्परता जाँच एक घंटे के लिए कैश की जाती हैं और पुरानी होने पर
अतुल्यकालिक रूप से रीफ़्रेश की जाती हैं। Codex थ्रेड ऐप कॉन्फ़िगरेशन हर चरण पर नहीं,
बल्कि Codex हार्नेस सत्र स्थापित करते समय गणना किया जाता है; मूल Plugin कॉन्फ़िगरेशन बदलने के बाद
`/new`, `/reset`, या Gateway पुनः आरंभ का उपयोग करें।

`codexPlugins.allow_all_plugins` प्रत्येक वर्तमान में सुलभ खाता
ऐप का स्नैपशॉट हर नए मूल Codex थ्रेड में जोड़ता है। यह plugins या ऐप इंस्टॉल नहीं करता, और
अप्राप्य ऐप बाहर ही रहते हैं। खाता ऐप वैश्विक
`codexPlugins.allow_destructive_actions` नीति का उपयोग करते हैं। जब एक ही ऐप दोनों पथों में मौजूद हो, तो स्पष्ट Plugin प्रविष्टियों को
प्राथमिकता मिलती है। यदि `app/list` को पढ़ा नहीं जा सके, तो पूरे खाते का एक्सपोज़र सुरक्षित रूप से विफल होता है।

- `plugins.entries.firecrawl.config.webFetch`: Firecrawl वेब-फ़ेच प्रदाता सेटिंग्स।
  - `apiKey`: अधिक सीमाओं के लिए वैकल्पिक Firecrawl API कुंजी (SecretRef स्वीकार करती है)। उपलब्ध न होने पर `plugins.entries.firecrawl.config.webSearch.apiKey` या `FIRECRAWL_API_KEY` पर्यावरण चर का उपयोग करती है।
  - `baseUrl`: Firecrawl API आधार URL (डिफ़ॉल्ट: `https://api.firecrawl.dev`; स्वयं-होस्ट किए गए ओवरराइड को निजी/आंतरिक एंडपॉइंट लक्षित करने होंगे)।
  - `onlyMainContent`: पृष्ठों से केवल मुख्य सामग्री निकालें (डिफ़ॉल्ट: `true`)।
  - `maxAgeMs`: मिलीसेकंड में अधिकतम कैश आयु (डिफ़ॉल्ट: `172800000` / 2 दिन)।
  - `timeoutSeconds`: सेकंड में स्क्रेप अनुरोध समय-समाप्ति (डिफ़ॉल्ट: `60`)।
- `plugins.entries.xai.config.xSearch`: xAI X Search (Grok वेब खोज) सेटिंग्स।
  - `enabled`: X Search प्रदाता सक्षम करें।
  - `model`: खोज के लिए उपयोग किया जाने वाला Grok मॉडल (उदा. `"grok-4.3"`)।
- `plugins.entries.memory-core.config.dreaming`: मेमोरी Dreaming सेटिंग्स। चरणों और सीमाओं के लिए [Dreaming](/hi/concepts/dreaming) देखें।
  - `enabled`: मुख्य Dreaming स्विच (डिफ़ॉल्ट `false`)।
  - `frequency`: प्रत्येक पूर्ण Dreaming स्वीप के लिए Cron आवृत्ति (डिफ़ॉल्ट रूप से `"0 3 * * *"`)।
  - `model`: वैकल्पिक Dream Diary सबएजेंट मॉडल ओवरराइड। इसके लिए `plugins.entries.memory-core.subagent.allowModelOverride: true` आवश्यक है; लक्ष्यों को सीमित करने के लिए इसे `allowedModels` के साथ जोड़ें। मॉडल-अनुपलब्ध त्रुटियों पर सत्र के डिफ़ॉल्ट मॉडल से एक बार पुनः प्रयास किया जाता है; विश्वास या अनुमति-सूची विफलताओं पर बिना बताए फ़ॉलबैक नहीं होता।
  - चरण नीति और सीमाएँ कार्यान्वयन विवरण हैं (उपयोगकर्ता-सामना करने वाली कॉन्फ़िगरेशन कुंजियाँ नहीं)।
- पूर्ण मेमोरी कॉन्फ़िगरेशन [मेमोरी कॉन्फ़िगरेशन संदर्भ](/hi/reference/memory-config) में उपलब्ध है:
  - `memory.search.*`
  - `agents.entries.*.memory.search.*` प्रति-एजेंट ओवरराइड के लिए
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- सक्षम Claude बंडल plugins `settings.json` से अंतर्निहित OpenClaw डिफ़ॉल्ट भी प्रदान कर सकते हैं; OpenClaw उन्हें अपरिष्कृत OpenClaw कॉन्फ़िगरेशन पैच के रूप में नहीं, बल्कि परिशोधित एजेंट सेटिंग्स के रूप में लागू करता है।
- `plugins.slots.memory`: सक्रिय मेमोरी Plugin आईडी चुनें, या मेमोरी plugins अक्षम करने के लिए `"none"` चुनें।
- `plugins.slots.contextEngine`: सक्रिय संदर्भ इंजन Plugin आईडी चुनें; जब तक आप कोई अन्य इंजन इंस्टॉल और चयन नहीं करते, इसका डिफ़ॉल्ट `"legacy"` होता है।

[Plugins](/hi/tools/plugin) देखें।

---

## ब्राउज़र

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // opt in only for trusted private-network access
      // allowPrivateNetwork: true, // legacy alias
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    tabCleanup: {
      enabled: true,
      idleMinutes: 120,
      maxTabsPerSession: 8,
      sweepMinutes: 5,
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    // attachOnly: false,
  },
}
```

- `evaluateEnabled: false`, `act:evaluate` और `wait --fn` को अक्षम करता है।
- `tabCleanup`, निष्क्रिय समय के बाद या किसी सत्र के अपनी सीमा से अधिक होने पर ट्रैक किए गए प्राथमिक-एजेंट
  टैब की यथासंभव आवधिक सफ़ाई नियंत्रित करता है। ट्रैकिंग केवल
  ब्राउज़र टूल `action: "open"` द्वारा बनाए गए टैब पर लागू होती है; उपयोगकर्ता द्वारा खोले गए या
  अज्ञात स्वामित्व वाले टैब कभी नहीं अपनाए जाते। `tabCleanup` को अक्षम करने से स्पष्ट सत्र जीवनचक्र सफ़ाई अक्षम नहीं होती।
- स्थिर नेटिव CDP लक्ष्य और ब्राउज़र पहचान वाले होस्ट-स्थानीय ओपन
  साझा SQLite स्थिति में संग्रहीत होते हैं और Gateway के पुनः आरंभ होने पर भी
  `/new` तथा सत्र जीवनचक्र सफ़ाई के लिए पात्र बने रहते हैं। नेटिव टूल-संबंधी CDP लक्ष्य भी
  पुनः आरंभ होने के बाद निष्क्रियता और सीमा सफ़ाई के लिए पात्र बने रहते हैं। Chrome MCP
  प्रक्रिया-स्थानीय लक्ष्य हैंडल का उपयोग करता है, इसलिए पुराने मौजूदा-सत्र रिकॉर्ड
  पुनः आरंभ के बाद ऐसी गतिविधि पर निष्क्रियता स्वीप का जोखिम उठाने के बजाय
  जीवनचक्र सफ़ाई की प्रतीक्षा करते हैं जिसका किसी से संबंध निर्धारित नहीं किया जा सकता।
  OpenClaw बंद करने से पहले प्रोफ़ाइल और ब्राउज़र इंस्टेंस को सत्यापित करता है।
  Chrome MCP स्वतः-कनेक्ट, अनुपलब्ध `/json/version` ब्राउज़र पहचान और
  अनसुलझे नेटिव लक्ष्य पूरी तरह प्रक्रिया-स्थानीय रहते हैं, इसलिए
  पुनः आरंभ के बाद वे अपने-आप बंद नहीं किए जाते। पुराने गैर-ट्रैक किए गए टैब को
  मैन्युअल रूप से बंद करना आवश्यक है। अस्थायी विफलताएँ बाद में पुनः प्रयास के लिए लंबित रहती हैं। देखें
  [टैब सफ़ाई का स्वामित्व](/hi/tools/browser#tab-cleanup-ownership)।
- `ssrfPolicy.dangerouslyAllowPrivateNetwork` सेट न होने पर अक्षम रहता है, इसलिए ब्राउज़र नेविगेशन डिफ़ॉल्ट रूप से सख़्त रहता है।
- `ssrfPolicy.dangerouslyAllowPrivateNetwork: true` केवल तभी सेट करें जब आप जानबूझकर निजी-नेटवर्क ब्राउज़र नेविगेशन पर भरोसा करते हों।
- सख़्त मोड में, रिमोट CDP प्रोफ़ाइल एंडपॉइंट (`profiles.*.cdpUrl`) पहुँच-योग्यता/खोज जाँचों के दौरान उसी निजी-नेटवर्क अवरोधन के अधीन होते हैं।
- `ssrfPolicy.allowPrivateNetwork` एक लीगेसी उपनाम के रूप में समर्थित रहता है।
- सख़्त मोड में, स्पष्ट अपवादों के लिए `ssrfPolicy.hostnameAllowlist` और `ssrfPolicy.allowedHostnames` का उपयोग करें।
- रिमोट प्रोफ़ाइल केवल अटैच करने योग्य हैं (आरंभ/रोकना/रीसेट अक्षम हैं)।
- `profiles.*.cdpUrl`, `http://`, `https://`, `ws://` और `wss://` स्वीकार करता है।
  जब आप चाहते हैं कि OpenClaw `/json/version` खोजे, तब HTTP(S) का उपयोग करें; जब आपका
  प्रदाता आपको सीधा DevTools WebSocket URL देता है, तब WS(S) का उपयोग करें।
- यदि कोई बाहरी रूप से प्रबंधित CDP सेवा लूपबैक के माध्यम से पहुँच योग्य है, तो उस
  प्रोफ़ाइल का `attachOnly: true` सेट करें; अन्यथा OpenClaw लूपबैक पोर्ट को
  स्थानीय प्रबंधित ब्राउज़र प्रोफ़ाइल मानता है और स्थानीय पोर्ट स्वामित्व त्रुटियाँ रिपोर्ट कर सकता है।
- `existing-session` प्रोफ़ाइल CDP के बजाय Chrome MCP का उपयोग करती हैं और
  चयनित होस्ट पर या कनेक्टेड ब्राउज़र Node के माध्यम से अटैच हो सकती हैं।
- `existing-session` प्रोफ़ाइल Brave या Edge जैसी किसी विशिष्ट
  Chromium-आधारित ब्राउज़र प्रोफ़ाइल को लक्षित करने के लिए `userDataDir` सेट कर सकती हैं।
- `existing-session` प्रोफ़ाइल `cdpUrl` सेट कर सकती हैं जब Chrome पहले से
  DevTools HTTP(S) खोज एंडपॉइंट या सीधे WS(S) एंडपॉइंट के पीछे चल रहा हो। उस
  मोड में OpenClaw स्वतः-कनेक्ट का उपयोग करने के बजाय एंडपॉइंट को Chrome MCP को देता है;
  Chrome MCP लॉन्च आर्ग्युमेंट के लिए `userDataDir` को अनदेखा किया जाता है।
- `existing-session` प्रोफ़ाइल वर्तमान Chrome MCP रूट सीमाएँ बनाए रखती हैं:
  CSS चयनकर्ता लक्ष्यीकरण के बजाय स्नैपशॉट/रेफ़रेंस-आधारित क्रियाएँ, एक-फ़ाइल अपलोड
  हुक, कोई डायलॉग टाइमआउट ओवरराइड नहीं, कोई `wait --load networkidle` नहीं, और कोई
  `responsebody`, PDF निर्यात, डाउनलोड इंटरसेप्शन या बैच क्रियाएँ नहीं।
- स्थानीय प्रबंधित `openclaw` प्रोफ़ाइल स्वतः `cdpPort` और `cdpUrl` निर्धारित करती हैं; `cdpUrl`
  को स्पष्ट रूप से केवल रिमोट CDP प्रोफ़ाइल या मौजूदा-सत्र एंडपॉइंट
  अटैच के लिए सेट करें।
- स्थानीय प्रबंधित प्रोफ़ाइल उस प्रोफ़ाइल के लिए वैश्विक
  `browser.executablePath` को ओवरराइड करने हेतु `executablePath` सेट कर सकती हैं। इसका उपयोग एक प्रोफ़ाइल को
  Chrome में और दूसरी को Brave में चलाने के लिए करें।
- स्वतः-पहचान क्रम: यदि डिफ़ॉल्ट ब्राउज़र Chromium-आधारित हो → Chrome → Brave → Edge → Chromium → Chrome Canary।
- `browser.executablePath` और `browser.profiles.<name>.executablePath` दोनों
  Chromium लॉन्च से पहले आपके OS होम डायरेक्टरी के लिए `~` और `~/...` स्वीकार करते हैं।
  `existing-session` प्रोफ़ाइल पर प्रति-प्रोफ़ाइल `userDataDir` में भी टिल्ड का विस्तार किया जाता है।
- नियंत्रण सेवा: केवल लूपबैक (पोर्ट `gateway.port` से प्राप्त होता है, डिफ़ॉल्ट `18791`)।
- `extraArgs` स्थानीय Chromium स्टार्टअप में अतिरिक्त लॉन्च फ़्लैग जोड़ता है (उदाहरण के लिए
  `--disable-gpu`, विंडो आकार निर्धारण या डीबग फ़्लैग)।

---

## UI

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // इमोजी, छोटा टेक्स्ट, इमेज URL या डेटा URI
    },
    prefs: {
      theme: "claw", // claw | knot | dash | custom
      themeMode: "system", // light | dark | system
      locale: "en",
      chatShowThinking: true,
      chatShowToolCalls: true,
      chatPersistCommentary: true, // रन के बाद Control UI में टिप्पणी बनाए रखें; इसे चैनलों पर वितरित नहीं करता
      chatSendShortcut: "enter", // enter | modifier-enter
      chatFollowUpMode: "steer", // steer | queue; सर्वर क्यू मोड का उपयोग करने के लिए इसे छोड़ दें
      showAdvancedSettings: false, // Settings में प्रत्येक Advanced समूह का विस्तार करें
    },
  },
}
```

- `seamColor`: नेटिव ऐप UI क्रोम के लिए एक्सेंट रंग (Talk Mode बबल टिंट आदि)।
- `assistant`: Control UI पहचान ओवरराइड। सक्रिय एजेंट पहचान पर फ़ॉलबैक करता है।
- `prefs`: क्रॉस-डिवाइस ऑपरेटर प्राथमिकताएँ। यह कैनोनिकल स्थान है, ताकि एजेंट
  अनुमोदन गेट के माध्यम से इन्हें बदल सकें और प्रत्येक Control UI क्लाइंट
  समन्वयित रहे; ब्राउज़र तुरंत बूट होने के लिए मानों को स्थानीय स्टोरेज में प्रतिबिंबित करते हैं और
  जब वे कॉन्फ़िगरेशन नहीं लिख सकते (व्यूअर स्कोप, ऑफ़लाइन), तब डिवाइस-स्थानीय प्रति रखते हैं।
  `chatPersistCommentary` का डिफ़ॉल्ट `true` है। इसे `false` पर सेट करने से रन के दौरान लाइव
  टिप्पणी दिखाई देती रहती है, लेकिन पूर्ण होने पर हट जाती है और नई
  Codex टिप्पणी को स्थायी ट्रांसक्रिप्ट मिरर में प्रवेश करने से रोकती है। मैसेजिंग-चैनल
  वितरण अलग और अपरिवर्तित रहता है।
  `showAdvancedSettings` का डिफ़ॉल्ट `false` है; Settings खोज इस प्राथमिकता को बदले बिना
  अस्थायी रूप से एक मेल खाने वाला Advanced समूह खोल सकती है।
  केवल प्रस्तुति से संबंधित प्राथमिकताएँ, जैसे टेक्स्ट स्केल, चैट की चौड़ाई और लाइव
  साइडबार गतिविधि, ब्राउज़र-स्थानीय रहती हैं और Settings में कॉन्फ़िगर की जाती हैं।
  कनेक्टेड क्लाइंट सर्वर-साइड परिवर्तनों को लाइव लागू करते हैं: प्रत्येक स्थायी कॉन्फ़िगरेशन लेखन के बाद
  Gateway केवल-हैश `config.changed` इवेंट प्रसारित करता है और
  क्लाइंट अपना स्नैपशॉट रीफ़्रेश करते हैं (जब किसी स्थानीय Settings ड्राफ़्ट में
  सहेजे न गए संपादन हों, तब इसे छोड़ दिया जाता है)। पुनः कनेक्ट होने वाले क्लाइंट कनेक्ट होते समय मिलान करते हैं।

---

## Gateway

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // या OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // mode=trusted-proxy के लिए; /gateway/trusted-proxy-auth देखें
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },
    tailscale: {
      mode: "off", // off | serve | funnel
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // toolTitles: false, // टूल कॉल के लिए वैकल्पिक AI उद्देश्य शीर्षक (उपयोगिता-मॉडल टोकन खर्च करता है)
      // embedSandbox: "scripts", // strict | scripts | trusted
      // allowExternalEmbedUrls: false, // ख़तरनाक: पूर्ण बाहरी http(s) एम्बेड URL की अनुमति दें
      // allowedOrigins: ["https://control.example.com"], // गैर-लूपबैक Control UI के लिए आवश्यक
      // dangerouslyAllowHostHeaderOriginFallback: false, // ख़तरनाक Host-header ओरिजिन फ़ॉलबैक मोड
    },
    terminal: {
      enabled: false,
      // shell: "/bin/zsh",
    },
    remote: {
      url: "ws://127.0.0.1:18789",
      transport: "ssh", // ssh | direct
      token: "your-token",
      // password: "your-password",
    },
    trustedProxies: ["10.0.0.1"],
    // वैकल्पिक। डिफ़ॉल्ट false।
    allowRealIpFallback: false,
    nodes: {
      pairing: {
        // वैकल्पिक। डिफ़ॉल्ट रूप से सेट नहीं/अक्षम।
        autoApproveCidrs: ["192.168.1.0/24", "fd00:1234:5678::/64"],
        // SSH-सत्यापित स्वतः-अनुमोदन। डिफ़ॉल्ट: सक्षम (true)।
        // केवल SSH सत्यापन अक्षम करने के लिए false सेट करें; इससे ऊपर दिए गए
        // autoApproveCidrs पर प्रभाव नहीं पड़ता। केवल मैन्युअल Node पेयरिंग के लिए, false सेट करें और
        // autoApproveCidrs को अनसेट करें। समायोजन के लिए ऑब्जेक्ट दें: { user, identity,
        // timeoutMs, cidrs }।
        sshVerify: true,
      },
      commands: {
        allow: ["canvas.navigate"],
        deny: ["system.run"],
      },
    },
    tools: {
      // अतिरिक्त /tools/invoke HTTP निषेध
      deny: ["browser"],
      // स्वामी/व्यवस्थापक कॉलर के लिए डिफ़ॉल्ट HTTP निषेध सूची से टूल हटाएँ
      allow: ["gateway"],
    },
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },
  },
}
```

<Accordion title="Gateway फ़ील्ड विवरण">

- `mode`: `local` (Gateway चलाएँ) या `remote` (दूरस्थ Gateway से कनेक्ट करें)। जब तक `local` न हो, Gateway प्रारंभ होने से इनकार करता है।
- `port`: WS + HTTP के लिए एकल मल्टीप्लेक्स्ड पोर्ट। प्राथमिकता: `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`।
- `bind`: `auto`, `loopback` (डिफ़ॉल्ट), `lan` (`0.0.0.0`), `tailnet` (उपलब्ध होने पर Tailscale IPv4, अन्यथा लूपबैक), या `custom` (एक IPv4 पता)। रिज़ॉल्व किया गया `tailnet` पता और `127.0.0.1` या `0.0.0.0` के अलावा कोई भी `custom` पता, समान होस्ट के क्लाइंट के लिए उसी पोर्ट पर `127.0.0.1` की आवश्यकता रखते हैं; यदि कोई भी लिसनर बाइंड नहीं कर पाता, तो स्टार्टअप विफल हो जाता है। गैर-लूपबैक एक्सपोज़र चयनित इंटरफ़ेस तक सीमित रहता है।
- **पुराने बाइंड उपनाम**: होस्ट उपनामों (`0.0.0.0`, `127.0.0.1`, `localhost`, `::`, `::1`) के बजाय `gateway.bind` में बाइंड मोड मानों (`auto`, `loopback`, `lan`, `tailnet`, `custom`) का उपयोग करें।
- **Docker नोट**: डिफ़ॉल्ट `loopback` बाइंड कंटेनर के भीतर `127.0.0.1` पर सुनता है। Docker ब्रिज नेटवर्किंग (`-p 18789:18789`) के साथ, ट्रैफ़िक `eth0` पर पहुँचता है, इसलिए Gateway पहुँच योग्य नहीं होता। `--network host` का उपयोग करें, या सभी इंटरफ़ेस पर सुनने के लिए `bind: "lan"` (या `customBindHost: "0.0.0.0"` के साथ `bind: "custom"`) सेट करें।
- **प्रमाणीकरण**: डिफ़ॉल्ट रूप से आवश्यक। गैर-लूपबैक बाइंड के लिए Gateway प्रमाणीकरण आवश्यक है। व्यवहार में इसका अर्थ साझा टोकन/पासवर्ड या `gateway.auth.mode: "trusted-proxy"` वाला पहचान-जागरूक रिवर्स प्रॉक्सी है। ऑनबोर्डिंग विज़ार्ड डिफ़ॉल्ट रूप से एक टोकन उत्पन्न करता है।
- यदि `gateway.auth.token` और `gateway.auth.password` दोनों कॉन्फ़िगर किए गए हैं (SecretRefs सहित), तो `gateway.auth.mode` को स्पष्ट रूप से `token` या `password` पर सेट करें। दोनों कॉन्फ़िगर होने और मोड सेट न होने पर स्टार्टअप तथा सेवा इंस्टॉल/मरम्मत प्रवाह विफल हो जाते हैं।
- `gateway.auth.mode: "none"`: स्पष्ट बिना-प्रमाणीकरण मोड। केवल विश्वसनीय स्थानीय लूपबैक सेटअप के लिए उपयोग करें; इसे जानबूझकर ऑनबोर्डिंग प्रॉम्प्ट में प्रस्तुत नहीं किया जाता।
- `gateway.auth.mode: "trusted-proxy"`: ब्राउज़र/उपयोगकर्ता प्रमाणीकरण पहचान-जागरूक रिवर्स प्रॉक्सी को सौंपें और `gateway.trustedProxies` से पहचान हेडर पर भरोसा करें ([विश्वसनीय प्रॉक्सी प्रमाणीकरण](/hi/gateway/trusted-proxy-auth) देखें)। यह मोड डिफ़ॉल्ट रूप से **गैर-लूपबैक** प्रॉक्सी स्रोत की अपेक्षा करता है; समान होस्ट के लूपबैक रिवर्स प्रॉक्सी के लिए स्पष्ट `gateway.auth.trustedProxy.allowLoopback = true` आवश्यक है। आंतरिक समान-होस्ट कॉलर स्थानीय प्रत्यक्ष फ़ॉलबैक के रूप में `gateway.auth.password` का उपयोग कर सकते हैं; `gateway.auth.token` विश्वसनीय-प्रॉक्सी मोड के साथ परस्पर अनन्य रहता है।
- `gateway.auth.allowTailscale`: जब `true` हो, तो Tailscale Serve पहचान हेडर Control UI/WebSocket प्रमाणीकरण को पूरा कर सकते हैं (`tailscale whois` के माध्यम से सत्यापित)। HTTP API एंडपॉइंट उस Tailscale हेडर प्रमाणीकरण का उपयोग **नहीं** करते; इसके बजाय वे Gateway के सामान्य HTTP प्रमाणीकरण मोड का पालन करते हैं। यह टोकन-रहित प्रवाह मानता है कि Gateway होस्ट विश्वसनीय है। `tailscale.mode = "serve"` होने पर डिफ़ॉल्ट `true` होता है।
- `gateway.auth.rateLimit`: वैकल्पिक विफल-प्रमाणीकरण लिमिटर। यह प्रति क्लाइंट IP और प्रति प्रमाणीकरण दायरे पर लागू होता है (साझा-सीक्रेट और डिवाइस-टोकन अलग-अलग ट्रैक किए जाते हैं)। अवरुद्ध प्रयास `429` + `Retry-After` लौटाते हैं।
  - असिंक्रोनस Tailscale Serve Control UI पथ पर, समान `{scope, clientIp}` के विफल प्रयासों को विफलता लिखे जाने से पहले क्रमबद्ध किया जाता है। इसलिए समान क्लाइंट से एक साथ होने वाले गलत प्रयास, दोनों के साधारण बेमेल के रूप में आगे निकल जाने के बजाय दूसरे अनुरोध पर लिमिटर सक्रिय कर सकते हैं।
  - `gateway.auth.rateLimit.exemptLoopback` का डिफ़ॉल्ट `true` है; जब आप जानबूझकर localhost ट्रैफ़िक पर भी दर सीमा लगाना चाहते हों (परीक्षण सेटअप या कठोर प्रॉक्सी परिनियोजन के लिए), तब `false` सेट करें।
- ब्राउज़र-मूल WS प्रमाणीकरण प्रयासों को हमेशा लूपबैक छूट अक्षम रखते हुए सीमित किया जाता है (ब्राउज़र-आधारित localhost ब्रूट फ़ोर्स के विरुद्ध बहुस्तरीय सुरक्षा)।
- लूपबैक पर, वे ब्राउज़र-मूल लॉकआउट प्रत्येक सामान्यीकृत `Origin`
  मान के अनुसार अलग रखे जाते हैं, इसलिए एक localhost मूल से बार-बार विफलताएँ किसी
  दूसरे मूल को स्वतः लॉक नहीं करतीं।
- `tailscale.mode`: `serve` (केवल tailnet, लूपबैक बाइंड) या `funnel` (सार्वजनिक, प्रमाणीकरण आवश्यक)।
- `tailscale.serviceName`: Serve मोड के लिए वैकल्पिक Tailscale Service नाम, जैसे
  `svc:openclaw`। सेट होने पर OpenClaw इसे `tailscale serve
--service` को देता है, ताकि Control UI को डिवाइस होस्टनाम के बजाय
  किसी नामित Service के माध्यम से प्रदर्शित किया जा सके। मान को Tailscale के `svc:<dns-label>`
  Service नाम प्रारूप का उपयोग करना चाहिए; स्टार्टअप व्युत्पन्न Service URL की सूचना देता है।
- `tailscale.preserveFunnel`: जब `true` और `tailscale.mode = "serve"` हों, तब OpenClaw
  स्टार्टअप पर Serve को दोबारा लागू करने से पहले `tailscale funnel status` की जाँच करता है और यदि बाहरी रूप से कॉन्फ़िगर किया गया Funnel रूट पहले से Gateway पोर्ट को कवर करता है, तो
  इसे छोड़ देता है। डिफ़ॉल्ट `false`।
- `controlUi.allowedOrigins`: Gateway WebSocket कनेक्शन के लिए स्पष्ट ब्राउज़र-मूल अनुमति सूची। सार्वजनिक गैर-लूपबैक ब्राउज़र मूल के लिए आवश्यक। लूपबैक, RFC1918/लिंक-लोकल, `.local`, `.ts.net`, या Tailscale CGNAT होस्ट से निजी समान-मूल LAN/Tailnet UI लोड को Host-header फ़ॉलबैक सक्षम किए बिना स्वीकार किया जाता है।
- `controlUi.toolTitles`: Control UI चैट में टूल कॉल के लिए AI-जनित उद्देश्य शीर्षकों को चुनकर सक्षम करें। डिफ़ॉल्ट: `false` (टूल रेंडरिंग बिना किसी पृष्ठभूमि मॉडल कॉल के पूर्णतः नियतात्मक रहती है)। सक्षम होने पर, `chat.toolTitles` विधि मानक यूटिलिटी-मॉडल रूटिंग के माध्यम से जटिल कॉल को लेबल करती है—एजेंट का `utilityModel` (एक ऑपरेटर निर्णय, जो प्रत्येक यूटिलिटी कार्य की तरह चुने गए प्रदाता को सीमित टूल आर्ग्युमेंट भेज सकता है), या सत्र प्रदाता का घोषित छोटे-मॉडल का डिफ़ॉल्ट (OpenAI → `gpt-5.6-luna`, Anthropic → `claude-haiku-4-5`)—और परिणामों को प्रति-एजेंट स्थिति डेटाबेस में कैश करती है, ताकि दोबारा देखने पर कभी फिर शुल्क न लगे। `utilityModel: \"\"` प्रत्येक अन्य यूटिलिटी कार्य की तरह शीर्षकों को अक्षम करता है; शीर्षक कभी भी प्राथमिक मॉडल पर फ़ॉलबैक नहीं करते।
- `controlUi.dangerouslyAllowHostHeaderOriginFallback`: खतरनाक मोड, जो जानबूझकर Host-header मूल नीति पर निर्भर परिनियोजनों के लिए Host-header मूल फ़ॉलबैक सक्षम करता है।
- `terminal.enabled`: एडमिन-दायरे वाले ऑपरेटर टर्मिनल को चुनकर सक्षम करें। डिफ़ॉल्ट: `false`। टर्मिनल चयनित एजेंट कार्यक्षेत्र में होस्ट PTY प्रारंभ करता है, Gateway प्रक्रिया का परिवेश विरासत में लेता है, और `sandbox.mode: "all"` वाले एजेंटों के लिए अस्वीकार कर दिया जाता है। इसे केवल विश्वसनीय ऑपरेटर परिनियोजनों के लिए सक्षम करें; इसे बदलने पर Gateway पुनः प्रारंभ होता है और Control UI की सामग्री सुरक्षा नीति अपडेट होती है।
- `terminal.shell`: वैकल्पिक शेल निष्पादन योग्य। सेट न होने पर, OpenClaw Unix पर `$SHELL` और Windows पर `%ComSpec%` का उपयोग करता है।
- `terminal.detachedSessionTimeoutSeconds`: किसी टर्मिनल सत्र का कनेक्शन टूटने के बाद (पृष्ठ पुनः लोड, लैपटॉप स्लीप) वह कितनी देर जीवित रहता है और अपने हालिया आउटपुट को दोबारा चलाते हुए `terminal.attach` के माध्यम से पुनः संलग्न करने योग्य बना रहता है। डिफ़ॉल्ट: `300`। कनेक्शन टूटते ही सत्र समाप्त करने के लिए `0` सेट करें। अलग किए गए सत्र अपने कमांड चलाते रहते हैं, इसलिए साझा या उजागर होस्ट पर इसे कम करें।
- `remote.transport`: `ssh` (डिफ़ॉल्ट) या `direct` (ws/wss)। `direct` के लिए, सार्वजनिक होस्ट पर `remote.url` का `wss://` होना आवश्यक है; सादा-पाठ `ws://` केवल लूपबैक, LAN, लिंक-लोकल, `.local`, `.ts.net`, और Tailscale CGNAT होस्ट के लिए स्वीकार किया जाता है।
- `remote.remotePort`: दूरस्थ SSH होस्ट पर Gateway पोर्ट। डिफ़ॉल्ट `18789`; इसका उपयोग तब करें जब स्थानीय टनल पोर्ट दूरस्थ Gateway पोर्ट से अलग हो।
- `remote.tlsFingerprint`: दूरस्थ `wss://` Gateway के लिए अपेक्षित SHA-256 प्रमाणपत्र फ़िंगरप्रिंट। macOS ऐप इसे ऑपरेटर/कंट्रोल और सहयोगी-Node, दोनों कनेक्शन पर लागू करता है। स्पष्ट मान के बिना, macOS सामान्य सिस्टम विश्वास सफल होने के बाद ही प्रथम-उपयोग पिन दर्ज करता है।
- `remote.sshHostKeyPolicy`: macOS SSH टनल होस्ट-कुंजी नीति। `strict` डिफ़ॉल्ट है और पहले से विश्वसनीय कुंजी आवश्यक करता है। `openssh` प्रबंधित उपनामों के लिए प्रभावी OpenSSH कॉन्फ़िगरेशन में स्पष्ट ऑप्ट-इन है; इसका उपयोग करने से पहले मेल खाने वाली उपयोगकर्ता और सिस्टम SSH सेटिंग की समीक्षा करें। लक्ष्य बदलते समय macOS ऐप और `configure-remote` इस नीति को `strict` पर रीसेट करते हैं, जब तक कि दोबारा स्पष्ट रूप से ऑप्ट-इन न किया जाए।
- `gateway.remote.token` / `.password` दूरस्थ-क्लाइंट क्रेडेंशियल फ़ील्ड हैं। वे अपने आप Gateway प्रमाणीकरण कॉन्फ़िगर नहीं करते।
- `gateway.push.apns.relay.baseUrl`: रिले-समर्थित iOS बिल्ड द्वारा Gateway पर पंजीकरण प्रकाशित करने के बाद उपयोग किए जाने वाले बाहरी APNs रिले का आधार HTTPS URL। सार्वजनिक App Store बिल्ड होस्ट किए गए OpenClaw रिले का उपयोग करते हैं। कस्टम रिले URL को जानबूझकर अलग iOS बिल्ड/परिनियोजन पथ से मेल खाना चाहिए, जिसका रिले URL उस रिले की ओर संकेत करता हो।
- `gateway.push.apns.relay.timeoutMs`: Gateway से रिले भेजने की समय-सीमा मिलीसेकंड में। डिफ़ॉल्ट `10000`।
- रिले-समर्थित पंजीकरण किसी विशिष्ट Gateway पहचान को सौंपे जाते हैं। युग्मित iOS ऐप `gateway.identity.get` प्राप्त करता है, उस पहचान को रिले पंजीकरण में शामिल करता है, और पंजीकरण-दायरे वाला भेजने का अनुदान Gateway को अग्रेषित करता है। कोई अन्य Gateway उस संग्रहीत पंजीकरण का पुनः उपयोग नहीं कर सकता।
- `OPENCLAW_APNS_RELAY_BASE_URL` / `OPENCLAW_APNS_RELAY_TIMEOUT_MS`: ऊपर दिए गए रिले कॉन्फ़िगरेशन के लिए अस्थायी परिवेश ओवरराइड।
- `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true`: लूपबैक HTTP रिले URL के लिए केवल-विकास एस्केप हैच। उत्पादन रिले URL को HTTPS पर ही रहना चाहिए।
- `OPENCLAW_HANDSHAKE_TIMEOUT_MS`: अंतर्निहित पूर्व-प्रमाणीकरण Gateway WebSocket हैंडशेक समय-सीमा के लिए वैकल्पिक परिवेश ओवरराइड।
- `channels.<provider>.healthMonitor.enabled`: वैश्विक मॉनिटर सक्षम रखते हुए स्वास्थ्य-मॉनिटर पुनः प्रारंभ से प्रति-चैनल ऑप्ट-आउट।
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: बहु-खाता चैनलों के लिए प्रति-खाता ओवरराइड। सेट होने पर, यह चैनल-स्तरीय ओवरराइड पर प्राथमिकता लेता है।
- स्थानीय Gateway कॉल पथ `gateway.remote.*` का फ़ॉलबैक के रूप में केवल तभी उपयोग कर सकते हैं, जब `gateway.auth.*` सेट न हो।
- यदि `gateway.auth.token` / `gateway.auth.password` को SecretRef के माध्यम से स्पष्ट रूप से कॉन्फ़िगर किया गया है और वह रिज़ॉल्व नहीं हुआ, तो रिज़ॉल्यूशन बंद-सुरक्षित तरीके से विफल होता है (कोई दूरस्थ फ़ॉलबैक इसे छिपाता नहीं है)।
- `trustedProxies`: वे रिवर्स प्रॉक्सी IP जो TLS समाप्त करते हैं या अग्रेषित-क्लाइंट हेडर इंजेक्ट करते हैं। केवल अपने नियंत्रण वाले प्रॉक्सी सूचीबद्ध करें। लूपबैक प्रविष्टियाँ समान-होस्ट प्रॉक्सी/स्थानीय-पहचान सेटअप (उदाहरण के लिए Tailscale Serve या स्थानीय रिवर्स प्रॉक्सी) के लिए अब भी मान्य हैं, लेकिन वे लूपबैक अनुरोधों को `gateway.auth.mode: "trusted-proxy"` के योग्य **नहीं** बनातीं।
- `allowRealIpFallback`: जब `true` हो, तो `X-Forwarded-For` अनुपस्थित होने पर Gateway `X-Real-IP` स्वीकार करता है। बंद-सुरक्षित व्यवहार के लिए डिफ़ॉल्ट `false`।
- `gateway.nodes.pairing.autoApproveCidrs`: बिना अनुरोधित दायरों वाले पहली बार के Node डिवाइस युग्मन को स्वतः स्वीकृत करने के लिए वैकल्पिक CIDR/IP अनुमति सूची। सेट न होने पर यह अक्षम रहती है। यह ऑपरेटर/ब्राउज़र/Control UI/WebChat युग्मन को स्वतः स्वीकृत नहीं करती, और भूमिका, दायरा, मेटाडेटा या सार्वजनिक-कुंजी अपग्रेड को भी स्वतः स्वीकृत नहीं करती।
- `gateway.nodes.pairing.sshVerify`: पहली बार के Node डिवाइस युग्मन के लिए SSH-सत्यापित स्वतः-स्वीकृति (डिफ़ॉल्ट: सक्षम)। Gateway युग्मन होस्ट पर वापस SSH करता है (BatchMode, कठोर होस्ट कुंजियाँ) और केवल सटीक `openclaw node identity` डिवाइस-कुंजी मिलान पर स्वीकृति देता है। पात्रता की न्यूनतम सीमा `autoApproveCidrs` जैसी ही है; जाँच निजी/CGNAT स्रोत पतों तक सीमित होती है, जब तक `cidrs` उन्हें ओवरराइड न करे। अक्षम करने के लिए `false`, या समायोजित करने के लिए `{ user, identity, timeoutMs, cidrs }` सेट करें। [Node युग्मन](/hi/gateway/pairing#ssh-verified-device-auto-approval-default) देखें।
- `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny`: पेयरिंग और प्लेटफ़ॉर्म अनुमति-सूची के मूल्यांकन के बाद घोषित Node कमांड के लिए वैश्विक अनुमति/अस्वीकृति निर्धारण। `camera.snap`, `camera.clip`, `screen.record`, `health.summary`, `sms.search`, और `sms.send` जैसे खतरनाक Node कमांड को स्वीकार करने के लिए `commands.allow` का उपयोग करें; `commands.deny` किसी कमांड को हटा देता है, भले ही प्लेटफ़ॉर्म डिफ़ॉल्ट या स्पष्ट अनुमति अन्यथा उसे शामिल करती। iOS Health अनुमति, Android SMS अनुमति और Gateway कमांड प्राधिकरण स्वतंत्र हैं। किसी Node द्वारा अपनी घोषित कमांड सूची बदलने के बाद, उस डिवाइस की पेयरिंग अस्वीकार करके फिर से स्वीकृत करें, ताकि Gateway अद्यतन कमांड स्नैपशॉट संग्रहीत करे।
- `gateway.tools.deny`: HTTP `POST /tools/invoke` के लिए अवरुद्ध अतिरिक्त टूल नाम (डिफ़ॉल्ट अस्वीकृति सूची का विस्तार करता है)।
- `gateway.tools.allow`: स्वामी/व्यवस्थापक कॉलर के लिए डिफ़ॉल्ट HTTP अस्वीकृति सूची से
  टूल नाम हटाएँ। यह पहचान-युक्त `operator.write`
  कॉलर को स्वामी/व्यवस्थापक पहुँच प्रदान नहीं करता; अनुमति-सूची में शामिल होने पर भी `cron`, `gateway`, और `nodes`
  गैर-स्वामी कॉलर के लिए अनुपलब्ध रहते हैं।

</Accordion>

### OpenAI-संगत एंडपॉइंट

- एडमिन HTTP RPC: `admin-http-rpc` plugin की तरह डिफ़ॉल्ट रूप से बंद है। `POST /api/v1/admin/rpc` को पंजीकृत करने के लिए plugin सक्षम करें। [एडमिन HTTP RPC](/hi/plugins/admin-http-rpc) देखें।
- चैट कम्प्लीशन्स: डिफ़ॉल्ट रूप से अक्षम। `gateway.http.endpoints.chatCompletions.enabled: true` से सक्षम करें।
- रिस्पॉन्सेज़ API: `gateway.http.endpoints.responses.enabled`।
- रिस्पॉन्सेज़ URL-इनपुट सुदृढ़ीकरण:
  - `gateway.http.endpoints.responses.maxUrlParts`
  - `gateway.http.endpoints.responses.files.urlAllowlist`
  - `gateway.http.endpoints.responses.images.urlAllowlist`
    खाली अनुमत-सूचियों को अनसेट माना जाता है; URL फ़ेचिंग अक्षम करने के लिए `gateway.http.endpoints.responses.files.allowUrl=false`
    और/या `gateway.http.endpoints.responses.images.allowUrl=false` का उपयोग करें।
- वैकल्पिक रिस्पॉन्स सुदृढ़ीकरण हेडर:
  - `gateway.http.securityHeaders.strictTransportSecurity` (केवल आपके नियंत्रण वाले HTTPS ओरिजिन के लिए सेट करें; [विश्वसनीय प्रॉक्सी प्रमाणीकरण](/hi/gateway/trusted-proxy-auth#tls-termination-and-hsts) देखें)

### बहु-इंस्टेंस पृथक्करण

अद्वितीय पोर्ट और स्टेट डायरेक्टरी के साथ एक होस्ट पर कई Gateway चलाएँ:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

सुविधाजनक फ़्लैग: `--dev` (`~/.openclaw-dev` + पोर्ट `19001` का उपयोग करता है), `--profile <name>` (`~/.openclaw-<name>` का उपयोग करता है)।

[एकाधिक Gateway](/hi/gateway/multiple-gateways) देखें।

### `gateway.tls`

```json5
{
  gateway: {
    tls: {
      enabled: false,
      autoGenerate: false,
      certPath: "/etc/openclaw/tls/server.crt",
      keyPath: "/etc/openclaw/tls/server.key",
      caPath: "/etc/openclaw/tls/ca-bundle.crt",
    },
  },
}
```

- `enabled`: Gateway लिसनर पर TLS टर्मिनेशन (HTTPS/WSS) सक्षम करता है (डिफ़ॉल्ट: `false`)।
- `autoGenerate`: स्पष्ट फ़ाइलें कॉन्फ़िगर न होने पर स्थानीय स्व-हस्ताक्षरित प्रमाणपत्र/कुंजी युग्म स्वतः जनरेट करता है; केवल स्थानीय/डेवलपमेंट उपयोग के लिए।
- `certPath`: TLS प्रमाणपत्र फ़ाइल का फ़ाइल-सिस्टम पथ।
- `keyPath`: TLS निजी कुंजी फ़ाइल का फ़ाइल-सिस्टम पथ; अनुमतियाँ प्रतिबंधित रखें।
- `caPath`: क्लाइंट सत्यापन या कस्टम ट्रस्ट चेन के लिए वैकल्पिक CA बंडल पथ।

### `gateway.reload`

```json5
{
  gateway: {
    reload: {
      mode: "hybrid", // off | restart | hot | hybrid
      debounceMs: 500,
      deferralTimeoutMs: 300000,
    },
  },
}
```

- `mode`: यह नियंत्रित करता है कि कॉन्फ़िगरेशन संपादन रनटाइम पर कैसे लागू किए जाते हैं।
  - `"off"`: लाइव संपादनों को अनदेखा करें; परिवर्तनों के लिए स्पष्ट रीस्टार्ट आवश्यक है।
  - `"restart"`: कॉन्फ़िगरेशन बदलने पर Gateway प्रक्रिया को हमेशा रीस्टार्ट करें।
  - `"hot"`: रीस्टार्ट किए बिना प्रक्रिया के भीतर परिवर्तन लागू करें।
  - `"hybrid"` (डिफ़ॉल्ट): पहले हॉट रीलोड का प्रयास करें; आवश्यक होने पर रीस्टार्ट पर फ़ॉलबैक करें।
- `debounceMs`: कॉन्फ़िगरेशन परिवर्तन लागू होने से पहले मिलीसेकंड में डीबाउंस विंडो (गैर-ऋणात्मक पूर्णांक; डिफ़ॉल्ट: `300`)।
- `deferralTimeoutMs`: रीस्टार्ट या चैनल हॉट रीलोड बाध्य करने से पहले जारी ऑपरेशनों की प्रतीक्षा के लिए मिलीसेकंड में वैकल्पिक अधिकतम समय। डिफ़ॉल्ट सीमित प्रतीक्षा (`300000`) का उपयोग करने के लिए इसे छोड़ दें; अनिश्चितकाल तक प्रतीक्षा करने और समय-समय पर अभी-भी-लंबित चेतावनियाँ लॉग करने के लिए `0` सेट करें।

---

## क्लाउड वर्कर परिवेश

क्लाउड वर्कर ऑप्ट-इन हैं। यदि `cloudWorkers` अनुपस्थित है, या `profiles` खाली है, तो OpenClaw किसी नए वर्कर का निर्माण स्वीकार नहीं करता। पहले बनाए गए टिकाऊ रिकॉर्ड फिर भी समन्वित होते और दिखाई देते रहते हैं; मौजूदा Gateway/Node प्रोजेक्शन अपरिवर्तित रहता है।

प्रत्येक वर्कर प्रदाता को विश्वसनीय प्रोविज़निंग आउटपुट से SSH `hostKey` बिल्कुल `algorithm base64` के रूप में लौटाना होगा, जिसमें होस्टनेम या टिप्पणी न हो। बूटस्ट्रैप उस कुंजी को एक पृथक `known_hosts` फ़ाइल में लिखता है, `StrictHostKeyChecking=yes` का उपयोग करता है, और प्रदाता द्वारा इसे छोड़े जाने पर कनेक्शन खोलने से पहले विफल हो जाता है। प्रथम उपयोग पर विश्वास करने वाला कोई फ़ॉलबैक नहीं है।

टनल सेटअप प्रोविज़निंग का हिस्सा होने के बजाय माँग पर होता है। शुरू होने पर Gateway, वर्कर-स्थानीय Unix सॉकेट को उसके लूपबैक WebSocket एंडपॉइंट पर रिवर्स-फ़ॉरवर्ड करता है। सॉकेट यादृच्छिक रूप से आवंटित, केवल स्वामी के लिए सुलभ रिमोट डायरेक्टरी में रहता है; लूपबैक TCP पोर्ट के विपरीत, बहु-उपयोगकर्ता वर्कर पर अन्य अकाउंट उस तक नहीं पहुँच सकते और वह किसी अन्य परिवेश के पोर्ट से टकरा नहीं सकता। SSH कीपअलाइव और सीमित रीकनेक्ट बैकऑफ़ केवल तब तक चलते हैं जब तक टनल का स्वामी वर्तमान रहता है। टनल रोकने पर SSH प्रक्रिया बंद करने से पहले रीकनेक्ट को अवरुद्ध किया जाता है।

कंट्रोल ट्रैफ़िक और वर्कस्पेस ट्रांसफ़र अलग-अलग SSH कनेक्शन का उपयोग करते हैं। दोनों समान रिज़ॉल्व की गई पहचान और पृथक पिन की गई `known_hosts` फ़ाइल का पुनः उपयोग करते हैं, लेकिन वर्कस्पेस ट्रांसफ़र लंबे समय तक चलने वाली टनल के साथ SSH कनेक्शन मल्टीप्लेक्सिंग साझा नहीं करता, इसलिए rsync कंट्रोल ट्रैफ़िक को अवरुद्ध नहीं कर सकता।

### Crabbox प्रोफ़ाइल

बंडल किया गया `crabbox` प्रदाता स्थानीय Crabbox CLI के माध्यम से SSH-सक्षम लीज़ प्रोविज़न करता है। आंतरिक `settings.provider` Crabbox बैकएंड चुनता है; यह बाहरी OpenClaw प्रदाता आईडी से अलग है।

```json5
{
  cloudWorkers: {
    profiles: {
      production: {
        provider: "crabbox",
        install: "bundle", // डिफ़ॉल्ट; केवल जारी किए गए Gateway संस्करण के लिए "npm" का उपयोग करें।
        settings: {
          provider: "aws",
          class: "standard",
          ttl: "24h",
          idleTimeout: "60m",
          // वैकल्पिक निरपेक्ष पथ। डिफ़ॉल्ट: सहोदर ../crabbox/bin/crabbox, फिर PATH।
          binary: "/usr/local/bin/crabbox",
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `settings.provider` (आवश्यक): `--provider` के माध्यम से भेजा गया Crabbox बैकएंड। ऐसे बैकएंड का उपयोग करें जिसके निरीक्षण आउटपुट में SSH एंडपॉइंट शामिल हो; `aws` प्रत्यक्ष AWS बैकएंड चुनता है।
- `settings.class` (आवश्यक): `--class` को भेजी गई Crabbox मशीन श्रेणी।
- `settings.ttl` और `settings.idleTimeout` (आवश्यक): `--ttl` और `--idle-timeout` को भेजी गई धनात्मक Go अवधि स्ट्रिंग। ये प्रदाता-पक्षीय विफलता-सुरक्षा उपाय नीचे दी गई OpenClaw की संग्रहीत `lifetime` नीति से अलग हैं।
- `settings.binary`: वैकल्पिक निरपेक्ष Crabbox निष्पादन योग्य पथ। इसके बिना, OpenClaw पहले सहोदर Crabbox चेकआउट, फिर `PATH` पर निष्पादन योग्य प्रविष्टियाँ जाँचता है, और अंत में `crabbox` चलाता है, ताकि अनुपस्थित CLI एक दृश्यमान प्रदाता त्रुटि बनी रहे।

अज्ञात सेटिंग अस्वीकार की जाती हैं। Crabbox क्रेडेंशियल और बैकएंड-विशिष्ट अकाउंट कॉन्फ़िगरेशन का स्वामित्व Crabbox के पास रहता है; उन्हें `settings` में न रखें। OpenClaw केवल स्थानीय CLI चलाता है और इस plugin से कोई प्रदाता नेटवर्क कॉल नहीं करता। प्रोविज़निंग हमेशा `--keep=true` भेजती है; OpenClaw बाहरी जीवनचक्र का स्वामी है और `crabbox stop` से लीज़ नष्ट करता है।

<Note>
  OpenClaw, Crabbox के लीज़-स्थानीय `sshKey` पथ को प्रदाता-स्वामित्व वाले सीक्रेट रिज़ॉल्वर के माध्यम से रिज़ॉल्व करता है और `crabbox inspect --json` द्वारा लौटाए गए आधिकारिक `sshHostKey` को पिन करता है। AWS प्रवेश के लिए `providerMetadata.instanceProfileAttached` भी आवश्यक है। इस बंद निरीक्षण अनुबंध के लिए Crabbox 0.38.1 या नया संस्करण इंस्टॉल करें।
</Note>

### स्थिर SSH डेवलपमेंट प्रोफ़ाइल

```json5
{
  cloudWorkers: {
    profiles: {
      development: {
        provider: "static-ssh",
        settings: {
          host: "worker.example.test",
          port: 22,
          user: "openclaw",
          hostKey: "ssh-ed25519 <base64-public-host-key>",
          keyRef: {
            source: "env",
            provider: "default",
            id: "OPENCLAW_WORKER_SSH_KEY",
          },
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `profiles`: गैर-रिक्त, व्हाइटस्पेस हटाई गई आईडी वाले नामित वर्कर प्रोफ़ाइल। प्रत्येक प्रोफ़ाइल किसी plugin द्वारा पंजीकृत प्रदाता चुनती है।
- `provider`: गैर-रिक्त वर्कर प्रदाता आईडी। उदाहरण बंडल किए गए `crabbox` प्रदाता और QA Lab के `static-ssh` प्रदाता का उपयोग करते हैं।
- `install`: वर्कर इंस्टॉलेशन विधि। `"bundle"` (डिफ़ॉल्ट) Gateway के इंस्टॉल किए गए बिल्ड का सामग्री-हैशयुक्त बंडल ट्रांसफ़र करता है और जारी, डेवलपमेंट तथा अजारी संस्करणों का समर्थन करता है। `"npm"` किसी अपरिवर्तित पैकेज्ड रिलीज़ के लिए ऑप्ट-इन अनुकूलन है; यह सार्वजनिक npm रजिस्ट्री से `openclaw@<exact gateway version>` इंस्टॉल करता है और कभी भी `latest` इंस्टॉल नहीं करता।
- कॉन्फ़िगर होने पर बंडल किए गए प्रदाता plugin स्वचालित रूप से चुने जाते हैं, लेकिन स्पष्ट अक्षमता और `plugins.allow` फिर भी लागू होते हैं। अनुमत-सूची कॉन्फ़िगर होने पर प्रदाता आईडी (उदाहरण के लिए, `crabbox`) शामिल करें। बाहरी प्रदाता plugin भी इंस्टॉल और स्पष्ट रूप से सक्षम होने चाहिए।
- `settings`: प्रदाता-स्वामित्व वाला सीमित JSON। चयनित plugin इसकी कुंजियाँ परिभाषित और सत्यापित करता है; सीक्रेट रखने वाले मानों के लिए [SecretRef ऑब्जेक्ट](/hi/gateway/secrets) का उपयोग करें। स्थिर SSH प्रदाता को `host`, `user`, `hostKey`, और `keyRef` की आवश्यकता होती है; `port` का डिफ़ॉल्ट `22` है। `hostKey` ज्ञात होस्ट या किसी अन्य विश्वसनीय चैनल से प्राप्त एक OpenSSH सार्वजनिक होस्ट-कुंजी पंक्ति (`algorithm base64`) होनी चाहिए, जिसमें कोई विकल्प उपसर्ग न हो।
- `lifetime.idleTimeoutMinutes`: बाद की निष्क्रियता-पुनरुद्धार नीति के लिए संग्रहीत धनात्मक पूर्णांक मिनट।
- `lifetime.maxLifetimeMinutes`: बाद की जीवनचक्र नीति के लिए संग्रहीत धनात्मक पूर्णांक मिनट।

WAL-रीसेट-सुरक्षित SQLite वाला समर्थित Node रनटाइम (22.22.3+, 24.15+, या 25.9+) वर्कर पर पहले से इंस्टॉल होना चाहिए। ऑप्ट-इन `"npm"` विधि के लिए `npm` और सार्वजनिक npm रजिस्ट्री तक आउटबाउंड HTTPS पहुँच भी आवश्यक है। नेटवर्कयुक्त टूलचेन सेटअप प्रदाता नीति है; बूटस्ट्रैप स्वयं टूलचेन इंस्टॉल करने के बजाय कार्रवाई योग्य त्रुटि रिपोर्ट करता है।

यह आधार Gateway बिल्ड को इंस्टॉल और सत्यापित करता है तथा टनल शुरू/रोक जीवनचक्र प्रदान करता है, लेकिन यह सामान्य OpenClaw CLI लॉन्च नहीं करता। स्व-निहित वर्कर एंट्री और लूप अगले क्लाउड-वर्कर माइलस्टोन में आएँगे।

प्रत्येक टिकाऊ परिवेश रिकॉर्ड निर्माण-समय प्रोफ़ाइल स्नैपशॉट में अपनी सत्यापित प्रदाता सेटिंग, रिज़ॉल्व की गई इंस्टॉलेशन विधि और जीवनकाल नीति बनाए रखता है। किसी नामित प्रोफ़ाइल को बदलना या हटाना नए निर्माणों को प्रभावित करता है; मौजूदा रिकॉर्ड उस स्नैपशॉट के साथ जीवनचक्र समन्वयन जारी रखते हैं, बशर्ते स्वामी plugin उपलब्ध रहे।

पहले क्लाउड-वर्कर रिलीज़ में जीवनकाल मान केवल डेटा हैं; स्वचालित प्रवर्तन बाद के जीवनचक्र कार्य के साथ आएगा। प्रोफ़ाइल परिवर्तनों के लिए Gateway रीस्टार्ट आवश्यक है।

<Warning>
  `static-ssh` प्रदाता स्रोत-वृक्ष QA Lab डेवलपमेंट हार्नेस है और पैकेज्ड वितरणों से बाहर रखा गया है। इसके साझा होस्ट पर चलने वाला वर्कर असंबंधित होस्ट डेटा पढ़ सकता है, इसलिए इस प्रदाता को प्रोडक्शन पृथक्करण सीमा के रूप में उपयोग न करें।
  इसके ऑपरेटर को अपेक्षित `hostKey` देना होगा; OpenClaw पहले कनेक्शन से कोई कुंजी सीखेगा या स्वीकार नहीं करेगा।
  इसकी लीज़ नष्ट करने से केवल OpenClaw का तार्किक रिकॉर्ड रिलीज़ होता है; यह होस्ट को रोकता या साफ़ नहीं करता।
</Warning>

---

## हुक

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: true,
    allowedSessionKeyPrefixes: ["hook:", "hook:gmail:"],
    allowedAgentIds: ["hooks", "main"],
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        agentId: "hooks",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "प्रेषक: {{messages[0].from}}\nविषय: {{messages[0].subject}}\n{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

प्रमाणीकरण: `Authorization: Bearer <token>` या `x-openclaw-token: <token>`।
क्वेरी-स्ट्रिंग हुक टोकन अस्वीकार किए जाते हैं।

सत्यापन और सुरक्षा संबंधी टिप्पणियाँ:

- `hooks.enabled=true` के लिए गैर-रिक्त `hooks.token` आवश्यक है।
- `hooks.token` सक्रिय Gateway साझा-सीक्रेट प्रमाणीकरण (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` या `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`) से अलग होना चाहिए; पुनः उपयोग का पता चलने पर स्टार्टअप लॉग एक गैर-घातक सुरक्षा चेतावनी दर्ज करता है।
- `openclaw security audit` हुक/Gateway प्रमाणीकरण के पुनः उपयोग को एक गंभीर निष्कर्ष के रूप में चिह्नित करता है, जिसमें केवल ऑडिट के समय प्रदान किया गया Gateway पासवर्ड प्रमाणीकरण (`--auth password --password <password>`) भी शामिल है। स्थायी रूप से सहेजे गए पुनः प्रयुक्त `hooks.token` को बदलने के लिए `openclaw doctor --fix` चलाएँ, फिर बाहरी हुक प्रेषकों को नया हुक टोकन उपयोग करने के लिए अपडेट करें।
- `hooks.path`, `/` नहीं हो सकता; `/hooks` जैसे समर्पित उपपथ का उपयोग करें।
- यदि `hooks.allowRequestSessionKey=true`, तो `hooks.allowedSessionKeyPrefixes` को सीमित करें (उदाहरण के लिए `["hook:"]`)।
- यदि कोई मैपिंग या प्रीसेट टेम्पलेट-आधारित `sessionKey` का उपयोग करता है, तो `hooks.allowedSessionKeyPrefixes` और `hooks.allowRequestSessionKey=true` सेट करें। स्थिर मैपिंग कुंजियों के लिए इस स्पष्ट सहमति की आवश्यकता नहीं होती।

**एंडपॉइंट:**

- `POST /hooks/wake` → `{ text, mode?: "now"|"next-heartbeat" }`
- `POST /hooks/agent` → `{ message, name?, agentId?, sessionKey?, wakeMode?, deliver?, channel?, to?, model?, thinking?, timeoutSeconds? }`
  - अनुरोध पेलोड से `sessionKey` केवल तभी स्वीकार किया जाता है जब `hooks.allowRequestSessionKey=true` (डिफ़ॉल्ट: `false`)।
- `POST /hooks/<name>` → `hooks.mappings` के माध्यम से समाधान किया जाता है
  - टेम्पलेट द्वारा रेंडर किए गए मैपिंग `sessionKey` मानों को बाहरी रूप से प्रदान किया गया माना जाता है और इनके लिए भी `hooks.allowRequestSessionKey=true` आवश्यक है।

<Accordion title="मैपिंग विवरण">

- `match.path`, `/hooks` के बाद के उपपथ से मेल खाता है (जैसे `/hooks/gmail` → `gmail`)।
- `match.source` सामान्य पथों के लिए किसी पेलोड फ़ील्ड से मेल खाता है।
- `{{messages[0].subject}}` जैसे टेम्पलेट पेलोड से पढ़ते हैं।
- `transform` किसी हुक क्रिया को लौटाने वाले JS/TS मॉड्यूल की ओर संकेत कर सकता है।
  - `transform.module` एक सापेक्ष पथ होना चाहिए और `hooks.transformsDir` के भीतर ही रहता है (निरपेक्ष पथ और डायरेक्टरी ट्रैवर्सल अस्वीकार किए जाते हैं)।
  - `hooks.transformsDir` को `~/.openclaw/hooks/transforms` के अंतर्गत रखें; वर्कस्पेस स्किल डायरेक्टरियाँ अस्वीकार की जाती हैं। यदि `openclaw doctor` इस पथ को अमान्य बताता है, तो ट्रांसफ़ॉर्म मॉड्यूल को हुक ट्रांसफ़ॉर्म डायरेक्टरी में ले जाएँ या `hooks.transformsDir` हटाएँ।
- `agentId` किसी विशिष्ट एजेंट को रूट करता है; अज्ञात ID डिफ़ॉल्ट एजेंट पर वापस आते हैं।
- `allowedAgentIds`: प्रभावी एजेंट रूटिंग को सीमित करता है, जिसमें `agentId` छोड़े जाने पर डिफ़ॉल्ट-एजेंट पथ भी शामिल है (`*` या छोड़ा गया = सभी को अनुमति, `[]` = सभी को अस्वीकार)।
- `defaultSessionKey`: स्पष्ट `sessionKey` के बिना हुक एजेंट रन के लिए वैकल्पिक स्थिर सत्र कुंजी।
- `allowRequestSessionKey`: `/hooks/agent` कॉलर और टेम्पलेट-संचालित मैपिंग सत्र कुंजियों को `sessionKey` सेट करने की अनुमति देता है (डिफ़ॉल्ट: `false`)।
- `allowedSessionKeyPrefixes`: स्पष्ट `sessionKey` मानों (अनुरोध + मैपिंग) के लिए वैकल्पिक उपसर्ग अनुमति-सूची, जैसे `["hook:"]`। जब कोई भी मैपिंग या प्रीसेट टेम्पलेट-आधारित `sessionKey` का उपयोग करता है, तो यह आवश्यक हो जाता है।
- `deliver: true` अंतिम उत्तर किसी चैनल को भेजता है; `channel` का डिफ़ॉल्ट `last` है।
- `model` इस हुक रन के लिए LLM को ओवरराइड करता है (यदि मॉडल कैटलॉग सेट है, तो इसकी अनुमति होनी चाहिए)।

</Accordion>

### Gmail एकीकरण

- अंतर्निहित Gmail प्रीसेट `sessionKey: "hook:gmail:{{messages[0].id}}"` का उपयोग करता है।
- यह प्रति-संदेश कुंजी वार्तालाप संदर्भ को अलग करती है, टूल या वर्कस्पेस पहुँच को नहीं। `agentId` सेट करने वाली कस्टम मैपिंग के बिना, प्रीसेट डिफ़ॉल्ट एजेंट का उपयोग करता है।
- अविश्वसनीय इनबॉक्स के लिए, Gmail को एक समर्पित रीडर एजेंट पर रूट करें और उस एजेंट को [प्रति-एजेंट सैंडबॉक्स और टूल नीति](/hi/tools/multi-agent-sandbox-tools) से सीमित करें। यदि रीडर को मुख्य एजेंट को सूचित करना आवश्यक है, तो हैंडऑफ़ को [`tools.agentToAgent`](/hi/gateway/config-tools#toolsagenttoagent) से सीमित करें। अनुशंसित खतरा मॉडल और मॉडल स्तर के लिए [प्रॉम्प्ट इंजेक्शन](/hi/gateway/security#prompt-injection) देखें।
- यदि आप वह प्रति-संदेश रूटिंग बनाए रखते हैं, तो `hooks.allowRequestSessionKey: true` सेट करें और Gmail नेमस्पेस से मेल खाने के लिए `hooks.allowedSessionKeyPrefixes` को सीमित करें, उदाहरण के लिए `["hook:", "hook:gmail:"]`।
- यदि आपको `hooks.allowRequestSessionKey: false` की आवश्यकता है, तो टेम्पलेट-आधारित डिफ़ॉल्ट के बजाय स्थिर `sessionKey` से प्रीसेट को ओवरराइड करें।

```json5
{
  hooks: {
    gmail: {
      account: "openclaw@gmail.com",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

- कॉन्फ़िगर होने पर Gateway बूट के समय `gog gmail watch serve` को स्वतः प्रारंभ करता है। अक्षम करने के लिए `OPENCLAW_SKIP_GMAIL_WATCHER=1` सेट करें।
- Gateway के साथ अलग `gog gmail watch serve` न चलाएँ।

---

## Canvas Plugin होस्ट

```json5
{
  plugins: {
    entries: {
      canvas: {
        config: {
          host: {
            root: "~/.openclaw/workspace/canvas",
            liveReload: true,
            // enabled: false, // या OPENCLAW_SKIP_CANVAS_HOST=1
          },
        },
      },
    },
  },
}
```

- Gateway पोर्ट के अंतर्गत HTTP पर एजेंट द्वारा संपादन योग्य HTML/CSS/JS और A2UI प्रदान करता है:
  - `http://<gateway-host>:<gateway.port>/__openclaw__/canvas/`
  - `http://<gateway-host>:<gateway.port>/__openclaw__/a2ui/`
- केवल स्थानीय: `gateway.bind: "loopback"` बनाए रखें (डिफ़ॉल्ट)।
- गैर-लूपबैक बाइंड: अन्य Gateway HTTP सतहों की तरह canvas रूटों के लिए Gateway प्रमाणीकरण (टोकन/पासवर्ड/विश्वसनीय-प्रॉक्सी) आवश्यक है।
- Node WebView सामान्यतः प्रमाणीकरण हेडर नहीं भेजते; किसी Node के युग्मित और कनेक्ट होने के बाद, Gateway canvas/A2UI पहुँच के लिए Node-स्कोप वाली क्षमता URL घोषित करता है।
- क्षमता URL सक्रिय Node WS सत्र से बंधे होते हैं और शीघ्र समाप्त हो जाते हैं। IP-आधारित फ़ॉलबैक का उपयोग नहीं होता।
- प्रदान किए गए HTML में लाइव-रीलोड क्लाइंट इंजेक्ट करता है।
- खाली होने पर आरंभिक `index.html` स्वतः बनाता है।
- `/__openclaw__/a2ui/` पर A2UI भी प्रदान करता है।
- परिवर्तनों के लिए Gateway पुनः प्रारंभ करना आवश्यक है।
- बड़ी डायरेक्टरियों या `EMFILE` त्रुटियों के लिए लाइव रीलोड अक्षम करें।

---

## खोज

### mDNS (Bonjour)

```json5
{
  discovery: {
    mdns: {
      mode: "minimal", // minimal | full | off
    },
  },
}
```

- `minimal` (डिफ़ॉल्ट): TXT रिकॉर्ड से `cliPath` + `sshPort` छोड़ें।
- `full`: `cliPath` + `sshPort` शामिल करें; LAN मल्टीकास्ट विज्ञापन के लिए फिर भी बंडल किए गए `bonjour` Plugin का सक्षम होना आवश्यक है।
- `off`: Plugin की सक्षम स्थिति बदले बिना LAN मल्टीकास्ट विज्ञापन रोकें।
- बंडल किया गया `bonjour` Plugin macOS होस्ट पर स्वतः प्रारंभ होता है और Linux, Windows तथा कंटेनरीकृत Gateway परिनियोजनों में वैकल्पिक रूप से सक्षम किया जाता है।
- वैध DNS लेबल होने पर होस्टनाम का डिफ़ॉल्ट सिस्टम होस्टनाम होता है, अन्यथा `openclaw` उपयोग होता है। `OPENCLAW_MDNS_HOSTNAME` से ओवरराइड करें।
- `OPENCLAW_DISABLE_BONJOUR=1`, `discovery.mdns.mode` को ओवरराइड करके mDNS विज्ञापन को पूर्णतः अक्षम करता है।

### विस्तृत क्षेत्र (DNS-SD)

```json5
{
  discovery: {
    wideArea: { enabled: true },
  },
}
```

`~/.openclaw/dns/` के अंतर्गत एक यूनिकास्ट DNS-SD ज़ोन लिखता है। विभिन्न नेटवर्कों के बीच खोज के लिए, इसे DNS सर्वर (CoreDNS अनुशंसित) + Tailscale स्प्लिट DNS के साथ उपयोग करें।

सेटअप: `openclaw dns setup --apply`।

---

## परिवेश

### `env` (इनलाइन परिवेश चर)

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

- इनलाइन परिवेश चर केवल तभी लागू होते हैं जब प्रक्रिया परिवेश में कुंजी अनुपस्थित हो।
- `.env` फ़ाइलें: CWD `.env` + `~/.openclaw/.env` (दोनों में से कोई भी मौजूदा चरों को ओवरराइड नहीं करता)।
- `shellEnv`: आपके लॉगिन शेल प्रोफ़ाइल से अनुपस्थित अपेक्षित कुंजियाँ आयात करता है।
- पूर्ण प्राथमिकता क्रम के लिए [परिवेश](/hi/help/environment) देखें।

### परिवेश चर प्रतिस्थापन

किसी भी कॉन्फ़िगरेशन स्ट्रिंग में `${VAR_NAME}` के साथ परिवेश चरों का संदर्भ दें:

```json5
{
  gateway: {
    auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" },
  },
}
```

- केवल अपरकेस नाम मेल खाते हैं: `[A-Z_][A-Z0-9_]*`।
- अनुपस्थित/रिक्त चर कॉन्फ़िगरेशन लोड करते समय त्रुटि उत्पन्न करते हैं।
- शाब्दिक `${VAR}` के लिए `$${VAR}` से एस्केप करें।
- `$include` के साथ काम करता है।

---

## सीक्रेट

सीक्रेट संदर्भ योगात्मक हैं: सादा-पाठ मान अब भी काम करते हैं।

### `SecretRef`

एक ऑब्जेक्ट संरचना का उपयोग करें:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

सत्यापन:

- `provider` पैटर्न: `^[a-z][a-z0-9_-]{0,63}$`
- `source: "env"` ID पैटर्न: `^[A-Z][A-Z0-9_]{0,127}$`
- `source: "file"` ID: निरपेक्ष JSON पॉइंटर (उदाहरण के लिए `"/providers/openai/apiKey"`)
- `source: "exec"` ID पैटर्न: `^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$` (AWS-शैली के `secret#json_key` चयनकर्ताओं का समर्थन करता है)
- `source: "exec"` ID में `.` या `..` स्लैश-सीमांकित पथ खंड नहीं होने चाहिए (उदाहरण के लिए `a/../b` अस्वीकार किया जाता है)

### समर्थित क्रेडेंशियल सतह

- कैनोनिकल मैट्रिक्स: [SecretRef क्रेडेंशियल सतह](/hi/reference/secretref-credential-surface)
- `secrets apply` समर्थित `openclaw.json` क्रेडेंशियल पथों को लक्षित करता है।
- `auth-profiles.json` संदर्भ रनटाइम समाधान और ऑडिट कवरेज में शामिल होते हैं।

### सीक्रेट प्रदाता कॉन्फ़िगरेशन

```json5
{
  secrets: {
    providers: {
      default: { source: "env" }, // वैकल्पिक स्पष्ट env प्रदाता
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
        timeoutMs: 5000,
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        passEnv: ["PATH", "VAULT_ADDR"],
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

नोट:

- `file` प्रदाता `mode: "json"` और `mode: "singleValue"` का समर्थन करता है (singleValue मोड में `id`, `"value"` होना चाहिए)।
- Windows ACL सत्यापन अनुपलब्ध होने पर फ़ाइल और exec प्रदाता पथ विफलता पर बंद हो जाते हैं। केवल उन विश्वसनीय पथों के लिए `allowInsecurePath: true` सेट करें जिन्हें सत्यापित नहीं किया जा सकता।
- `exec` प्रदाता के लिए निरपेक्ष `command` पथ आवश्यक है और यह stdin/stdout पर प्रोटोकॉल पेलोड का उपयोग करता है।
- डिफ़ॉल्ट रूप से, सिमलिंक कमांड पथ अस्वीकार किए जाते हैं। समाधान किए गए लक्ष्य पथ को सत्यापित करते हुए सिमलिंक पथों की अनुमति देने के लिए `allowSymlinkCommand: true` सेट करें।
- यदि `trustedDirs` कॉन्फ़िगर है, तो विश्वसनीय-डायरेक्टरी जाँच समाधान किए गए लक्ष्य पथ पर लागू होती है।
- `exec` चाइल्ड परिवेश डिफ़ॉल्ट रूप से न्यूनतम होता है; आवश्यक चर `passEnv` से स्पष्ट रूप से पास करें।
- सीक्रेट संदर्भ सक्रियण के समय इन-मेमोरी स्नैपशॉट में समाधान किए जाते हैं, फिर अनुरोध पथ केवल स्नैपशॉट को पढ़ते हैं।
- सक्रियण के दौरान सक्रिय-सतह फ़िल्टरिंग लागू होती है: सक्षम सतहों पर अनसुलझे संदर्भ स्टार्टअप/रीलोड को विफल करते हैं, जबकि निष्क्रिय सतहों को निदान के साथ छोड़ दिया जाता है।

---

## प्रमाणीकरण संग्रहण

```json5
{
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:personal": { provider: "openai", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      openai: ["openai:personal"],
    },
  },
}
```

- प्रति-एजेंट प्रोफ़ाइल `<agentDir>/auth-profiles.json` पर संग्रहीत होती हैं।
- `auth-profiles.json` स्थिर क्रेडेंशियल मोड के लिए मान-स्तरीय संदर्भों (`api_key` के लिए `keyRef`, `token` के लिए `tokenRef`) का समर्थन करता है।
- `{ "provider": { "apiKey": "..." } }` जैसे पुराने समतल `auth-profiles.json` मैप रनटाइम प्रारूप नहीं हैं; `openclaw doctor --fix` उन्हें `.legacy-flat.*.bak` बैकअप के साथ मानक `provider:default` API-कुंजी प्रोफ़ाइल में पुनर्लिखता है।
- OAuth-मोड प्रोफ़ाइल (`auth.profiles.<id>.mode = "oauth"`) SecretRef-समर्थित प्रमाणीकरण-प्रोफ़ाइल क्रेडेंशियल का समर्थन नहीं करतीं।
- स्थिर रनटाइम क्रेडेंशियल इन-मेमोरी समाधान किए गए स्नैपशॉट से आते हैं; पुराने स्थिर `auth.json` प्रविष्टियों का पता चलने पर उन्हें हटा दिया जाता है।
- पुराने OAuth आयात `~/.openclaw/credentials/oauth.json` से आते हैं।
- [OAuth](/hi/concepts/oauth) देखें।
- सीक्रेट के रनटाइम व्यवहार और `audit/configure/apply` टूलिंग के लिए: [सीक्रेट प्रबंधन](/hi/gateway/secrets)।

---

## ऑडिट

```json5
{
  audit: {
    enabled: true,
    messages: "off", // off | direct | all
  },
}
```

Gateway, एजेंट रन और टूल कार्रवाइयों के लिए **केवल-मेटाडेटा** ऑडिट घटनाएँ
साझा स्थिति डेटाबेस में दर्ज करता है। संदेश जीवनचक्र मेटाडेटा एक अलग
ऑप्ट-इन है। लेजर पहचान, समय, टूल के नाम और सामान्यीकृत परिणाम संग्रहीत
करता है, लेकिन प्रॉम्प्ट, संदेश का मुख्य भाग, टूल आर्ग्युमेंट, परिणाम या अपरिष्कृत
त्रुटि पाठ कभी संग्रहीत नहीं करता। संदेश पंक्तियाँ अपरिष्कृत प्लेटफ़ॉर्म खाता, वार्तालाप,
संदेश और लक्ष्य आईडी संग्रहीत नहीं करतीं। रन/टूल सत्र कुंजियाँ सहसंबंध के लिए उपलब्ध
रहती हैं और उनमें स्वयं प्लेटफ़ॉर्म खाता या पीयर आईडी हो सकती हैं। रिकॉर्ड
30 दिनों के बाद समाप्त हो जाते हैं और लेजर की सीमा 100,000 पंक्तियाँ है। उन्हें
[`openclaw audit`](/hi/cli/audit) या
[`audit.activity.list`](/hi/gateway/protocol#audit-ledger-rpc) Gateway RPC से क्वेरी करें। पूर्ण डेटा मॉडल, गोपनीयता अर्थ-विज्ञान
और कवरेज सीमाओं के लिए [ऑडिट इतिहास](/hi/gateway/audit) देखें।

- `enabled`: नई ऑडिट घटनाएँ दर्ज करें (डिफ़ॉल्ट: `true`)। लेजर डिफ़ॉल्ट रूप से
  चालू रहता है क्योंकि किसी घटना के बाद ही सक्षम किया गया ऑडिट ट्रेल
  उस घटना की व्याख्या नहीं कर सकता। `false` सेट करने से Gateway के पुनः आरंभ होने के बाद नई घटनाओं का सम्मिलन रुक जाता है;
  मौजूदा रिकॉर्ड समाप्त होने तक पढ़ने योग्य रहते हैं। इसे फिर से चालू करने पर
  उस बिंदु से रिकॉर्डिंग पुनः शुरू होती है—बीच के अंतराल को बाद में नहीं भरा जाता।
- `messages`: संदेश मेटाडेटा का दायरा (डिफ़ॉल्ट: `"off"`)। `"direct"` केवल
  ज्ञात प्रत्यक्ष वार्तालाप दर्ज करता है। `"all"` समूह, चैनल और
  अज्ञात प्रकार के वार्तालाप भी दर्ज करता है। दोनों मोड सामग्री-रहित रहते हैं और जहाँ सहसंबंध
  उपलब्ध है, वहाँ अपरिष्कृत पहचानकर्ताओं को इंस्टॉलेशन-स्थानीय कुंजीबद्ध छद्मनामों से
  बदल देते हैं। ये गुमनामीकरण के बजाय सहसंबंध सहायक हैं; स्थिति
  डेटाबेस व्युत्पत्ति कुंजी संग्रहीत करता है, लेकिन RPC और CLI निर्यात नहीं करते।

चल रहा Gateway प्रारंभ के समय `audit.enabled` और `audit.messages` को कैप्चर करता है;
किसी भी सेटिंग को बदलने के बाद इसे पुनः आरंभ करें। संदेश कवरेज में वर्तमान में
वे स्वीकृत इनबाउंड संदेश शामिल हैं जो कोर डिस्पैच तक पहुँचते हैं और प्रत्येक
मूल तार्किक आउटबाउंड उत्तर पेलोड के लिए एक अंतिम पंक्ति शामिल है जो साझा टिकाऊ डिलीवरी तक पहुँचता है।
उन साझा सीमाओं को बायपास करने वाले Plugin-स्थानीय और सीधे-भेजने के पथ
अभी तक कवर नहीं किए गए हैं। सीमित पृष्ठभूमि
राइटर सर्वोत्तम-प्रयास वाला है, दोषरहित अनुपालन अभिलेखागार नहीं।

---

## लॉगिंग

```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty", // pretty | compact | json
    redactSensitive: "tools", // off | tools
    redactPatterns: ["\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1"],
  },
}
```

- डिफ़ॉल्ट लॉग फ़ाइल: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`; नामित प्रोफ़ाइल `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log` का उपयोग करती हैं।
- स्थिर पथ के लिए `logging.file` सेट करें।
- `--verbose` होने पर `consoleLevel`, `debug` तक बढ़ जाता है।
- `maxFileBytes`: रोटेशन से पहले सक्रिय लॉग फ़ाइल का अधिकतम आकार बाइट में (धनात्मक पूर्णांक; डिफ़ॉल्ट: `104857600` = 100 MB)। OpenClaw सक्रिय फ़ाइल के पास अधिकतम पाँच क्रमांकित अभिलेख रखता है।
- `redactSensitive` / `redactPatterns`: कंसोल आउटपुट, फ़ाइल लॉग, OTLP लॉग रिकॉर्ड और संग्रहीत सत्र ट्रांसक्रिप्ट पाठ के लिए सर्वोत्तम-प्रयास मास्किंग। `redactSensitive: "off"` केवल इस सामान्य लॉग/ट्रांसक्रिप्ट नीति को अक्षम करता है; UI/टूल/निदान सुरक्षा सतहें उत्सर्जन से पहले सीक्रेट को अब भी संपादित करती हैं।

---

## निदान

```json5
{
  diagnostics: {
    enabled: true,
    flags: ["telegram.*"],

    otel: {
      enabled: false,
      endpoint: "https://otel-collector.example.com:4318",
      tracesEndpoint: "https://traces.example.com/v1/traces",
      metricsEndpoint: "https://metrics.example.com/v1/metrics",
      logsEndpoint: "https://logs.example.com/v1/logs",
      protocol: "http/protobuf", // http/protobuf | grpc
      headers: { "x-tenant-id": "my-org" },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: false,
      logsExporter: "otlp",
      sampleRate: 1.0,
      flushIntervalMs: 5000,
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },

    cacheTrace: {
      enabled: false,
      filePath: "~/.openclaw/logs/cache-trace.jsonl",
      includeMessages: true,
      includePrompt: true,
      includeSystem: true,
    },
  },
}
```

- `enabled`: इंस्ट्रूमेंटेशन आउटपुट का मुख्य टॉगल (डिफ़ॉल्ट: `true`)।
- `flags`: लक्षित लॉग आउटपुट सक्षम करने वाली फ़्लैग स्ट्रिंग की सरणी (`"telegram.*"` या `"*"` जैसे वाइल्डकार्ड का समर्थन करती है)।
- `otel.enabled`: OpenTelemetry निर्यात पाइपलाइन सक्षम करता है (डिफ़ॉल्ट: `false`)। पूर्ण कॉन्फ़िगरेशन, सिग्नल कैटलॉग और गोपनीयता मॉडल के लिए [OpenTelemetry निर्यात](/hi/gateway/opentelemetry) देखें।
- `otel.endpoint`: OTel निर्यात के लिए कलेक्टर URL।
- `otel.tracesEndpoint` / `otel.metricsEndpoint` / `otel.logsEndpoint`: वैकल्पिक सिग्नल-विशिष्ट OTLP एंडपॉइंट। सेट होने पर ये केवल उस सिग्नल के लिए `otel.endpoint` को ओवरराइड करते हैं।
- `otel.protocol`: `"http/protobuf"` (डिफ़ॉल्ट) या `"grpc"`।
- `otel.headers`: OTel निर्यात अनुरोधों के साथ भेजे गए अतिरिक्त HTTP/gRPC मेटाडेटा हेडर।
- `otel.serviceName`: संसाधन एट्रिब्यूट के लिए सेवा नाम।
- `otel.traces` / `otel.metrics` / `otel.logs`: ट्रेस, मेट्रिक्स या लॉग निर्यात सक्षम करें।
- `otel.logsExporter`: लॉग निर्यात सिंक: `"otlp"` (डिफ़ॉल्ट), प्रत्येक stdout पंक्ति में एक JSON ऑब्जेक्ट के लिए `"stdout"`, या `"both"`।
- `otel.sampleRate`: ट्रेस सैंपलिंग दर `0`-`1`।
- `otel.flushIntervalMs`: आवधिक टेलीमेट्री फ्लश अंतराल मिलीसेकंड में।
- `otel.captureContent`: OTEL स्पैन एट्रिब्यूट के लिए अपरिष्कृत सामग्री कैप्चर का ऑप्ट-इन। डिफ़ॉल्ट रूप से बंद। बूलियन `true` गैर-सिस्टम संदेश/टूल सामग्री कैप्चर करता है; ऑब्जेक्ट रूप से आप `inputMessages`, `outputMessages`, `toolInputs`, `toolOutputs`, `systemPrompt` और `toolDefinitions` को स्पष्ट रूप से सक्षम कर सकते हैं।
- `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`: नवीनतम प्रायोगिक GenAI इन्फ़रेंस स्पैन आकार के लिए परिवेश टॉगल, जिसमें `{gen_ai.operation.name} {gen_ai.request.model}` स्पैन नाम, `CLIENT` स्पैन प्रकार और पुराने `gen_ai.system` के बजाय `gen_ai.provider.name` शामिल हैं। डिफ़ॉल्ट रूप से अनुकूलता के लिए स्पैन `openclaw.model.call` और `gen_ai.system` बनाए रखते हैं; GenAI मेट्रिक्स सीमित सिमेंटिक एट्रिब्यूट का उपयोग करते हैं।
- `OPENCLAW_OTEL_PRELOADED=1`: उन होस्ट के लिए परिवेश टॉगल जिन्होंने पहले ही वैश्विक OpenTelemetry SDK पंजीकृत कर लिया है। इसके बाद OpenClaw निदान लिसनर सक्रिय रखते हुए Plugin-स्वामित्व वाले SDK का प्रारंभ/शटडाउन छोड़ देता है।
- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` और `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`: संबंधित कॉन्फ़िगरेशन कुंजी अनसेट होने पर उपयोग किए जाने वाले सिग्नल-विशिष्ट एंडपॉइंट परिवेश चर।
- `cacheTrace.enabled`: एम्बेडेड रन के लिए कैश ट्रेस स्नैपशॉट लॉग करें (डिफ़ॉल्ट: `false`)।
- `cacheTrace.filePath`: कैश ट्रेस JSONL का आउटपुट पथ (डिफ़ॉल्ट: `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`)।
- `cacheTrace.includeMessages` / `includePrompt` / `includeSystem`: कैश ट्रेस आउटपुट में शामिल सामग्री नियंत्रित करें (सभी का डिफ़ॉल्ट: `true`)।

---

## अपडेट

```json5
{
  update: {
    channel: "stable", // stable | extended-stable | beta | dev
    checkOnStart: true,

    auto: {
      enabled: false,
    },
  },
}
```

- `channel`: रिलीज़ चैनल—`"stable"`, `"extended-stable"`, `"beta"` या `"dev"`। Extended-stable केवल पैकेज के लिए है: फ़ोरग्राउंड कमांड इंस्टॉलेशन नियंत्रित करते हैं, जबकि Gateway केवल-पठन अपडेट संकेत उत्सर्जित कर सकता है।
- `checkOnStart`: Gateway प्रारंभ होने पर npm अपडेट की जाँच करें (डिफ़ॉल्ट: `true`)। संग्रहीत extended-stable चयन उसी केवल-पठन संकेत और 24-घंटे की संकेत अनुसूची का उपयोग करते हैं।
- `auto.enabled`: stable और beta पैकेज इंस्टॉलेशन के लिए पृष्ठभूमि स्वतः-अपडेट सक्षम करें (डिफ़ॉल्ट: `false`)। Extended-stable कभी स्वतः लागू नहीं होता।

---

## ACP

```json5
{
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    fallbacks: ["acpx-secondary"],
    defaultAgent: "main",
    allowedAgents: ["main", "ops"],
    stream: {
      repeatSuppression: true,
      deliveryMode: "live", // live | final_only
    },
  },
}
```

- `enabled`: वैश्विक ACP सुविधा गेट (डिफ़ॉल्ट: `true`; ACP डिस्पैच और स्पॉन सुविधाएँ छिपाने के लिए `false` सेट करें)।
- `dispatch.enabled`: ACP सत्र टर्न डिस्पैच के लिए स्वतंत्र गेट (डिफ़ॉल्ट: `true`)। निष्पादन अवरुद्ध रखते हुए ACP कमांड उपलब्ध रखने के लिए `false` सेट करें।
- `backend`: डिफ़ॉल्ट ACP रनटाइम बैकएंड आईडी (पंजीकृत ACP रनटाइम Plugin से मेल खाना आवश्यक है)।
  पहले बैकएंड Plugin इंस्टॉल करें और यदि `plugins.allow` सेट है, तो बैकएंड Plugin आईडी (उदाहरण के लिए `acpx`) शामिल करें, अन्यथा ACP बैकएंड लोड नहीं होगा।
- `fallbacks`: प्राथमिक बैकएंड द्वारा कोई आउटपुट उत्पन्न करने से पहले अस्थायी प्रतीत होने वाली त्रुटि (अनुपलब्ध, दर-सीमित, कोटा समाप्त या अतिभारित) के साथ जल्दी विफल होने पर आज़माए जाने वाले फ़ॉलबैक ACP बैकएंड आईडी की क्रमबद्ध सूची। प्रत्येक प्रविष्टि को पंजीकृत ACP रनटाइम Plugin बैकएंड से मेल खाना आवश्यक है।
- `defaultAgent`: जब स्पॉन स्पष्ट लक्ष्य निर्दिष्ट न करें, तब फ़ॉलबैक ACP लक्ष्य एजेंट आईडी।
- `allowedAgents`: ACP रनटाइम सत्रों के लिए अनुमत एजेंट आईडी की अनुमति-सूची; खाली होने का अर्थ कोई अतिरिक्त प्रतिबंध नहीं है।
- `stream.repeatSuppression`: प्रत्येक टर्न में दोहराई गई स्थिति/टूल पंक्तियाँ दबाएँ (डिफ़ॉल्ट: `true`)।
- `stream.deliveryMode`: `"live"` क्रमिक रूप से स्ट्रीम करता है; `"final_only"` टर्न की अंतिम घटनाओं तक बफ़र करता है।
- `stream.tagVisibility`: स्ट्रीम की गई घटनाओं के लिए टैग नामों से बूलियन दृश्यता ओवरराइड तक का रिकॉर्ड।
- `runtime.installCommand`: ACP रनटाइम परिवेश बूटस्ट्रैप करते समय चलाने के लिए वैकल्पिक इंस्टॉल कमांड।

---

## विज़ार्ड

CLI निर्देशित सेटअप प्रवाहों (`onboard`, `configure`, `doctor`) के लिए व्यवहार और मेटाडेटा:

```json5
{
  wizard: {
    accessMode: "full",
    appRecommendations: true,
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
    securityAcknowledgedAt: "2026-01-01T00:00:00.000Z",
  },
}
```

- `wizard.accessMode`: निर्देशित ऑनबोर्डिंग की शुरुआत में चुनी गई खोज सहमति। `"full"` (अनुशंसित) सेटअप को AI ऐप्स, कुंजियों और स्थानीय रनटाइम को स्वचालित रूप से खोजने देता है; `"guarded"` सेटअप को खोज शुरू करने से पहले एक बार पूछने के लिए बाध्य करता है और इसके बजाय मैन्युअल कॉन्फ़िगरेशन प्रस्तुत करता है।

- `wizard.appRecommendations` का डिफ़ॉल्ट मान `true` है। निर्देशित या क्लासिक ऑनबोर्डिंग के दौरान इंस्टॉल किए गए एप्लिकेशन की अनुशंसाएँ अक्षम करने और Gateway की `device.apps` पहुँच अवरुद्ध करने के लिए इसे `false` पर सेट करें। कमांड का विज्ञापन करने से पहले Node होस्ट को अभी भी अपने अलग, डिफ़ॉल्ट रूप से बंद इंस्टॉल किए गए ऐप साझाकरण फ़्लैग की आवश्यकता होती है।

---

## पहचान

[एजेंट डिफ़ॉल्ट](/hi/gateway/config-agents#agent-defaults) के अंतर्गत `agents.entries` पहचान फ़ील्ड देखें।

---

## ब्रिज (विरासती, हटाया गया)

वर्तमान बिल्ड में अब TCP ब्रिज शामिल नहीं है। Node, Gateway WebSocket के माध्यम से कनेक्ट होते हैं। `bridge.*` कुंजियाँ अब कॉन्फ़िगरेशन स्कीमा का हिस्सा नहीं हैं (उन्हें हटाए जाने तक सत्यापन विफल रहता है; `openclaw doctor --fix` अज्ञात कुंजियाँ हटा सकता है)।

<Accordion title="विरासती ब्रिज कॉन्फ़िगरेशन (ऐतिहासिक संदर्भ)">

```json
{
  "bridge": {
    "enabled": true,
    "port": 18790,
    "bind": "tailnet",
    "tls": {
      "enabled": true,
      "autoGenerate": true
    }
  }
}
```

</Accordion>

---

## Cron

```json5
{
  cron: {
    enabled: true,
    webhook: "https://example.invalid/legacy", // संग्रहीत notify:true जॉब के लिए अप्रचलित फ़ॉलबैक
    webhookToken: "replace-with-dedicated-token", // आउटबाउंड webhook प्रमाणीकरण के लिए वैकल्पिक bearer टोकन
    sessionRetention: "24h", // अवधि स्ट्रिंग या false
  },
}
```

- `sessionRetention`: SQLite सत्र पंक्तियों को हटाने से पहले पूर्ण हो चुके पृथक Cron रन सत्रों को कितने समय तक रखना है। यह संग्रहित और हटाई गई Cron ट्रांसक्रिप्ट की सफ़ाई को भी नियंत्रित करता है। डिफ़ॉल्ट: `24h`; अक्षम करने के लिए `false` सेट करें।
- रन इतिहास प्रत्येक जॉब के लिए नवीनतम 2000 टर्मिनल पंक्तियाँ स्वचालित रूप से रखता है। खोई हुई पंक्तियों के लिए उनकी 24-घंटे की सफ़ाई अवधि बनी रहती है।
- `webhookToken`: Cron webhook POST डिलीवरी (`delivery.mode = "webhook"`) के लिए उपयोग किया जाने वाला bearer टोकन; इसे छोड़ने पर कोई प्रमाणीकरण हेडर नहीं भेजा जाता।
- `webhook`: अप्रचलित विरासती फ़ॉलबैक webhook URL (http/https), जिसका उपयोग `openclaw doctor --fix` उन संग्रहीत जॉब को माइग्रेट करने के लिए करता है जिनमें अभी भी `notify: true` है; रनटाइम डिलीवरी प्रति-जॉब `delivery.mode="webhook"` के साथ `delivery.to`, या घोषणा डिलीवरी बनाए रखते समय `delivery.completionDestination` का उपयोग करती है।

### `cron.failureAlert`

```json5
{
  cron: {
    failureAlert: {
      enabled: false,
      after: 3,
      cooldownMs: 3600000,
      includeSkipped: false,
      mode: "announce",
      accountId: "main",
    },
  },
}
```

- `enabled`: Cron जॉब के लिए विफलता अलर्ट सक्षम करें (डिफ़ॉल्ट: `false`)।
- `after`: अलर्ट सक्रिय होने से पहले लगातार विफलताओं की संख्या (धनात्मक पूर्णांक, न्यूनतम: `1`)।
- `cooldownMs`: उसी जॉब के लिए दोहराए गए अलर्ट के बीच न्यूनतम मिलीसेकंड (गैर-ऋणात्मक पूर्णांक)।
- `includeSkipped`: लगातार छोड़े गए रन को अलर्ट सीमा में गिनें (डिफ़ॉल्ट: `false`)। छोड़े गए रन अलग से ट्रैक किए जाते हैं और निष्पादन-त्रुटि बैकऑफ़ को प्रभावित नहीं करते।
- `mode`: डिलीवरी मोड - `"announce"` चैनल संदेश के माध्यम से भेजता है; `"webhook"` कॉन्फ़िगर किए गए webhook पर पोस्ट करता है।
- `accountId`: अलर्ट डिलीवरी का दायरा निर्धारित करने के लिए वैकल्पिक खाता या चैनल आईडी।

### `cron.failureDestination`

```json5
{
  cron: {
    failureDestination: {
      mode: "announce",
      channel: "last",
      to: "channel:C1234567890",
      accountId: "main",
    },
  },
}
```

- सभी जॉब में Cron विफलता सूचनाओं के लिए डिफ़ॉल्ट गंतव्य।
- `mode`: `"announce"` या `"webhook"`; पर्याप्त लक्ष्य डेटा उपलब्ध होने पर डिफ़ॉल्ट रूप से `"announce"` होता है।
- `channel`: घोषणा डिलीवरी के लिए चैनल ओवरराइड। `"last"` अंतिम ज्ञात डिलीवरी चैनल का पुनः उपयोग करता है।
- `to`: स्पष्ट घोषणा लक्ष्य या webhook URL। webhook मोड के लिए आवश्यक।
- `accountId`: डिलीवरी के लिए वैकल्पिक खाता ओवरराइड।
- प्रति-जॉब `delivery.failureDestination` इस वैश्विक डिफ़ॉल्ट को ओवरराइड करता है।
- जब न तो वैश्विक और न ही प्रति-जॉब विफलता गंतव्य सेट हो, तो वे जॉब जो पहले से `announce` के माध्यम से डिलीवरी करते हैं, विफलता पर उसी प्राथमिक घोषणा लक्ष्य का फ़ॉलबैक के रूप में उपयोग करते हैं।
- `delivery.failureDestination` केवल `sessionTarget="isolated"` जॉब के लिए समर्थित है, जब तक कि जॉब का प्राथमिक `delivery.mode`, `"webhook"` न हो।

[Cron जॉब](/hi/automation/cron-jobs) देखें। पृथक Cron निष्पादन को [पृष्ठभूमि कार्यों](/hi/automation/tasks) के रूप में ट्रैक किया जाता है।

## मीडिया मॉडल टेम्पलेट चर

`tools.media.models[].args` में विस्तारित टेम्पलेट प्लेसहोल्डर:

| चर                         | विवरण                                            |
| --------------------------- | ------------------------------------------------- |
| `{{Body}}`                  | पूरा इनबाउंड संदेश मुख्य भाग                     |
| `{{RawBody}}`               | अपरिष्कृत मुख्य भाग (इतिहास/प्रेषक रैपर के बिना) |
| `{{BodyStripped}}`          | समूह उल्लेख हटाया हुआ मुख्य भाग                  |
| `{{From}}`                  | प्रेषक पहचानकर्ता                                 |
| `{{To}}`                    | गंतव्य पहचानकर्ता                                 |
| `{{MessageSid}}`            | चैनल संदेश आईडी                                  |
| `{{SessionId}}`             | वर्तमान सत्र UUID                                 |
| `{{IsNewSession}}`          | नया सत्र बनाए जाने पर `"true"`                  |
| `{{AttachmentUrl}}`         | वर्तमान अटैचमेंट URL या प्रदाता संदर्भ           |
| `{{AttachmentPath}}`        | वर्तमान अटैचमेंट का स्थानीय पथ                   |
| `{{AttachmentContentType}}` | वर्तमान अटैचमेंट का MIME सामग्री प्रकार          |
| `{{AttachmentDir}}`         | `AttachmentPath` वाली डायरेक्टरी                  |
| `{{AttachmentIndex}}`       | शून्य-आधारित स्रोत तथ्य सूचकांक                   |
| `{{Transcript}}`            | ऑडियो ट्रांसक्रिप्ट                               |
| `{{Prompt}}`                | CLI प्रविष्टियों के लिए निर्धारित मीडिया प्रॉम्प्ट |
| `{{MaxChars}}`              | CLI प्रविष्टियों के लिए निर्धारित अधिकतम आउटपुट वर्ण |
| `{{ChatType}}`              | `"direct"` या `"group"`                           |
| `{{GroupSubject}}`          | समूह विषय (सर्वोत्तम प्रयास)                     |
| `{{GroupMembers}}`          | समूह सदस्यों का पूर्वावलोकन (सर्वोत्तम प्रयास)   |
| `{{SenderName}}`            | प्रेषक का प्रदर्शन नाम (सर्वोत्तम प्रयास)         |
| `{{SenderE164}}`            | प्रेषक का फ़ोन नंबर (सर्वोत्तम प्रयास)            |
| `{{Provider}}`              | प्रदाता संकेत (whatsapp, telegram, discord आदि)   |

विरासती `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}`, और `{{MediaDir}}`
नाम Plugin SDK संगतता अवधि के दौरान उपलब्ध रहते हैं, लेकिन
अप्रचलित हैं। नए कॉन्फ़िगरेशन को `Attachment*` चर का उपयोग करना चाहिए।

---

## कॉन्फ़िगरेशन समावेशन (`$include`)

कॉन्फ़िगरेशन को कई फ़ाइलों में विभाजित करें:

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

**मर्ज व्यवहार:**

- एकल फ़ाइल: समाहित करने वाले ऑब्जेक्ट को प्रतिस्थापित करती है।
- फ़ाइलों की सरणी: क्रम से डीप-मर्ज की जाती है (बाद वाली पहले वाली को ओवरराइड करती है)।
- सहोदर कुंजियाँ: समावेशन के बाद मर्ज की जाती हैं (समावेशित मानों को ओवरराइड करती हैं)।
- नेस्टेड समावेशन: अधिकतम 10 स्तर गहरे।
- पथ: समावेश करने वाली फ़ाइल के सापेक्ष निर्धारित किए जाते हैं, लेकिन शीर्ष-स्तरीय कॉन्फ़िगरेशन डायरेक्टरी (`openclaw.json` का `dirname`) के भीतर ही रहने चाहिए। निरपेक्ष/`../` रूपों की अनुमति केवल तभी है जब वे इस सीमा के भीतर निर्धारित होते हों। कॉन्फ़िगरेशन डायरेक्टरी के बाहर अतिरिक्त रूट की अनुमति देने के लिए `OPENCLAW_INCLUDE_ROOTS` (निरपेक्ष पथ) सेट करें।
- सीमाएँ: पथों में null बाइट नहीं होनी चाहिए और निर्धारण से पहले तथा बाद में उनकी लंबाई अनिवार्य रूप से 4096 वर्णों से कम होनी चाहिए; प्रत्येक समावेशित फ़ाइल की सीमा 2 MB है।
- OpenClaw के स्वामित्व वाले ऐसे लेखन, जो एकल-फ़ाइल समावेशन द्वारा समर्थित केवल एक शीर्ष-स्तरीय अनुभाग को बदलते हैं, सीधे उस समावेशित फ़ाइल में लिखते हैं। उदाहरण के लिए, `plugins install`, `plugins.json5` में `plugins: { $include: "./plugins.json5" }` को अपडेट करता है और `openclaw.json` को यथावत रखता है।
- रूट समावेशन, समावेशन सरणियाँ और सहोदर ओवरराइड वाले समावेशन, OpenClaw के स्वामित्व वाले लेखन के लिए केवल-पढ़ने योग्य हैं; ऐसे लेखन कॉन्फ़िगरेशन को समतल करने के बजाय सुरक्षित रूप से विफल होते हैं।
- त्रुटियाँ: अनुपलब्ध फ़ाइलों, पार्स त्रुटियों, चक्रीय समावेशन, अमान्य पथ प्रारूप और अत्यधिक लंबाई के लिए स्पष्ट संदेश।

---

## संबंधित

- [कॉन्फ़िगरेशन](/hi/gateway/configuration)
- [कॉन्फ़िगरेशन उदाहरण](/hi/gateway/configuration-examples)
- [Doctor](/hi/gateway/doctor)
