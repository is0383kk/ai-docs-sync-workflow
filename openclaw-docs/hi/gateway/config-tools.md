---
read_when:
    - '`tools.*` नीति, अनुमति-सूचियाँ या प्रायोगिक सुविधाएँ कॉन्फ़िगर करना'
    - कस्टम प्रदाताओं को पंजीकृत करना या बेस URL को ओवरराइड करना
    - OpenAI-संगत स्व-होस्टेड एंडपॉइंट सेट अप करना
sidebarTitle: Tools and custom providers
summary: टूल्स कॉन्फ़िगरेशन (नीति, प्रयोगात्मक टॉगल, प्रदाता-समर्थित टूल्स) और कस्टम प्रदाता/बेस-URL सेटअप
title: कॉन्फ़िगरेशन — टूल और कस्टम प्रोवाइडर
x-i18n:
    generated_at: "2026-07-27T17:51:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2010a2e48e8f4c8d0049e5c707bb8286e291a92312baac94301a7b5a674583c1
    source_path: gateway/config-tools.md
    workflow: 16
---

`tools.*` कॉन्फ़िगरेशन कुंजियाँ और कस्टम प्रदाता / आधार-URL सेटअप। एजेंटों, चैनलों और अन्य शीर्ष-स्तरीय कॉन्फ़िगरेशन कुंजियों के लिए, [कॉन्फ़िगरेशन संदर्भ](/hi/gateway/configuration-reference) देखें।

## टूल

### टूल प्रोफ़ाइल

`tools.profile`, `tools.allow`/`tools.deny` से पहले एक आधार अनुमति-सूची निर्धारित करता है:

<Note>
सेट न होने पर स्थानीय ऑनबोर्डिंग नए स्थानीय कॉन्फ़िगरेशन को डिफ़ॉल्ट रूप से `tools.profile: "coding"` पर सेट करती है (मौजूदा स्पष्ट प्रोफ़ाइल सुरक्षित रखी जाती हैं)।
</Note>

| प्रोफ़ाइल     | शामिल टूल                                                                                                                                                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | केवल `session_status`                                                                                                                                                                                                                                   |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`, `image`, `image_generate`, `music_generate`, `video_generate`                |
| `messaging` | `group:messaging`, `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `ask_user` |
| `full`      | कोई प्रतिबंध नहीं (सेट न होने के समान)                                                                                                                                                                                                                          |

`coding` और `messaging`, `bundle-mcp` (कॉन्फ़िगर किए गए MCP सर्वर) को भी अंतर्निहित रूप से अनुमति देते हैं।

### टूल समूह

| समूह              | टूल                                                                                                                                                                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash` को `exec` के उपनाम के रूप में स्वीकार किया जाता है)                                                                                                                                                                        |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `spawn_task`, `dismiss_task` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`, `screen`, `terminal`, `canvas`, `show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`, `cron`, `gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`, `computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`, `image_generate`, `music_generate`, `video_generate`, `tts`                                                                                                                                                                                   |
| `group:openclaw`   | `read`/`write`/`edit`/`apply_patch`/`exec`/`process`/`canvas` को छोड़कर ऊपर दिए गए सभी अंतर्निर्मित टूल (Plugin टूल को शामिल नहीं करता)                                                                                                                                  |
| `group:plugins`    | लोड किए गए Plugin के स्वामित्व वाले टूल, जिनमें `bundle-mcp` के माध्यम से उपलब्ध कराए गए कॉन्फ़िगर किए गए MCP सर्वर शामिल हैं                                                                                                                                                           |

`spawn_task` किसी कोडिंग एजेंट को पुष्टि किए गए अनुवर्ती कार्य को शुरू किए बिना प्रस्तावित करने देता है। Control UI शीर्षक और सारांश को कार्रवाई-योग्य चिप के रूप में दिखाता है; Gateway-समर्थित TUI एक समकक्ष इंटरैक्टिव प्रॉम्प्ट दिखाता है। इनमें से किसी को भी स्वीकार करने पर एक नया प्रबंधित-वर्कट्री सत्र बनता है और पूरा प्रॉम्प्ट वहाँ भेजा जाता है, जबकि वर्तमान टर्न जारी रहता है। `dismiss_task`, `spawn_task` से लौटाई गई अस्थायी `task_id` के आधार पर अब भी लंबित सुझाव को वापस लेता है।

ये टूल केवल तभी उपलब्ध कराए जाते हैं, जब आरंभ करने वाला ऑपरेटर इंटरफ़ेस Gateway कार्य-सुझाव इवेंट प्राप्त करके उन पर कार्रवाई कर सकता हो। चैनल सत्र और स्थानीय/एम्बेडेड TUI सत्र उन्हें प्राप्त नहीं करते; इस प्रवाह को सुरक्षित रूप से उपलब्ध कराने से पहले चैनल ट्रांसपोर्ट को पोर्टेबल टाइप की गई कार्य कार्रवाई की आवश्यकता होती है। सुझाव प्रक्रिया-स्थानीय होते हैं और Gateway पुनः आरंभ होने पर गायब हो जाते हैं। दोनों टूल `coding` प्रोफ़ाइल और `group:sessions` में बने रहते हैं, इसलिए जब इंटरफ़ेस उनका समर्थन करता है, तब सामान्य `tools.allow` और `tools.deny` नीति उन्हें स्वचालित रूप से कॉन्फ़िगर करती है।

### सैंडबॉक्स टूल नीति के भीतर MCP और Plugin टूल

कॉन्फ़िगर किए गए MCP सर्वर, `bundle-mcp` Plugin आईडी के अंतर्गत Plugin-स्वामित्व वाले टूल के रूप में उपलब्ध कराए जाते हैं। सामान्य टूल प्रोफ़ाइल उन्हें अनुमति दे सकती हैं, लेकिन सैंडबॉक्स किए गए सत्रों के लिए `tools.sandbox.tools` एक अतिरिक्त गेट है। यदि सैंडबॉक्स मोड `"all"` या `"non-main"` है, तो MCP/Plugin टूल दिखाई देने चाहिए तो सैंडबॉक्स टूल अनुमति-सूची में इनमें से एक प्रविष्टि शामिल करें:

- `mcp.servers` से OpenClaw-प्रबंधित MCP सर्वरों के लिए `bundle-mcp`
- किसी विशिष्ट मूल Plugin के लिए Plugin आईडी
- लोड किए गए सभी Plugin-स्वामित्व वाले टूल के लिए `group:plugins`
- जब आपको केवल एक सर्वर चाहिए, तब सटीक MCP सर्वर टूल नाम या सर्वर ग्लोब, जैसे `outlook__send_mail` या `outlook__*`

सर्वर ग्लोब प्रदाता-सुरक्षित MCP सर्वर उपसर्ग का उपयोग करते हैं, जो आवश्यक नहीं कि अपरिष्कृत `mcp.servers` कुंजी हो। गैर-`[A-Za-z0-9_-]` वर्ण `-` बन जाते हैं, जिन नामों की शुरुआत किसी अक्षर से नहीं होती उन्हें `mcp-` उपसर्ग मिलता है, और लंबे या डुप्लिकेट उपसर्गों को छोटा किया जा सकता है या उनमें प्रत्यय जोड़ा जा सकता है; उदाहरण के लिए, `mcp.servers["Outlook Graph"]` किसी `outlook-graph__*` जैसे ग्लोब का उपयोग करता है।

```json5
{
  agents: { defaults: { sandbox: { mode: "all" } } },
  mcp: {
    servers: {
      outlook: { command: "node", args: ["./outlook-mcp.js"] },
    },
  },
  tools: {
    sandbox: {
      tools: {
        alsoAllow: ["web_search", "web_fetch", "memory_search", "memory_get", "bundle-mcp"],
      },
    },
  },
}
```

उस सैंडबॉक्स-स्तरीय प्रविष्टि के बिना भी MCP सर्वर सफलतापूर्वक लोड हो सकता है, जबकि उसके टूल प्रदाता अनुरोध से पहले फ़िल्टर कर दिए जाते हैं। `mcp.servers` में OpenClaw-प्रबंधित सर्वरों के लिए इस स्थिति का पता लगाने हेतु `openclaw doctor` का उपयोग करें। बंडल किए गए Plugin मैनिफ़ेस्ट या Claude `.mcp.json` से लोड किए गए MCP सर्वर समान सैंडबॉक्स गेट का उपयोग करते हैं, लेकिन यह निदान अभी उन स्रोतों की गणना नहीं करता; यदि उनके टूल सैंडबॉक्स किए गए टर्न में गायब हो जाएँ, तो उन्हीं अनुमति-सूची प्रविष्टियों का उपयोग करें।

### `tools.codeMode`

`tools.codeMode` सामान्य OpenClaw कोड-मोड इंटरफ़ेस सक्षम करता है। टूल वाले किसी रन के लिए इसे सक्षम करने पर
सामान्य OpenClaw टूल, सैंडबॉक्स के भीतर मौजूद `tools.*`
कैटलॉग ब्रिज के पीछे चले जाते हैं, और MCP टूल जनरेट किए गए `MCP`
नेमस्पेस के माध्यम से उपलब्ध होते हैं। मॉडल सामान्यतः `exec` और `wait` देखता है; `computer` जैसे टूल,
जिनके संरचित परिणाम केवल-JSON ब्रिज को पार नहीं कर सकते, सीधे उपलब्ध रहते हैं।

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

संक्षिप्त रूप भी स्वीकार किया जाता है:

```json5
{
  tools: { codeMode: true },
}
```

MCP घोषणाएँ कोड मोड में केवल-पढ़ने योग्य वर्चुअल API फ़ाइल इंटरफ़ेस के माध्यम से उपलब्ध कराई जाती हैं।
गेस्ट कोड, `MCP.<server>.<tool>()` को कॉल करने से पहले TypeScript-शैली के सिग्नेचर का निरीक्षण करने के लिए
`API.list("mcp")` और `API.read("mcp/<server>.d.ts")` को कॉल कर सकता है।
रनटाइम अनुबंध, सीमाओं और डीबगिंग चरणों के लिए [कोड मोड](/hi/tools/code-mode) देखें।

### `tools.allow` / `tools.deny`

वैश्विक टूल अनुमति/अस्वीकृति नीति (अस्वीकृति प्रभावी होती है)। अक्षर-आकार से असंवेदनशील, `*` वाइल्डकार्ड का समर्थन करती है। Docker सैंडबॉक्स बंद होने पर भी लागू होती है।

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

`write` और `apply_patch` अलग-अलग टूल आईडी हैं। `allow: ["write"]` संगत मॉडलों के लिए `apply_patch` को भी सक्षम करता है, लेकिन `deny: ["write"]`, `apply_patch` को अस्वीकार नहीं करता। सभी फ़ाइल परिवर्तनों को अवरुद्ध करने के लिए, `group:fs` को अस्वीकार करें या प्रत्येक परिवर्तनकारी टूल को स्पष्ट रूप से सूचीबद्ध करें:

```json5
{
  tools: { deny: ["write", "edit", "apply_patch"] },
}
```

<Note>
`allow` और `alsoAllow` को एक ही दायरे (`tools`, `tools.byProvider.<id>`, `agents.entries.*.tools`) में दोनों सेट नहीं किया जा सकता — कॉन्फ़िगरेशन सत्यापन इसे अस्वीकार करता है। `alsoAllow` प्रविष्टियों को `allow` में मर्ज करें, या `allow` हटाकर इसके बजाय `profile` + `alsoAllow` का उपयोग करें।
</Note>

### `tools.byProvider`

विशिष्ट प्रदाताओं या मॉडलों के लिए टूल को और प्रतिबंधित करें। क्रम: आधार प्रोफ़ाइल → प्रदाता प्रोफ़ाइल → अनुमति/अस्वीकृति।

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.toolsBySender`

वर्तमान टर्न के मूल अनुरोधकर्ता के लिए टूल सीमित करता है। यह चैनल अभिगम नियंत्रण के ऊपर अतिरिक्त सुरक्षा परत है; प्रेषक के मान चैनल अडैप्टर से आने चाहिए, संदेश के टेक्स्ट से नहीं। यह मॉडल प्रॉम्प्ट की अन्य सामग्री को प्रमाणित नहीं करता; [अनुरोधकर्ता-स्कोप वाले नियंत्रण और प्रॉम्प्ट संदर्भ](/hi/gateway/security#requester-scoped-controls-and-prompt-context) देखें।

```json5
{
  tools: {
    toolsBySender: {
      "channel:discord:1234567890123": { alsoAllow: ["group:fs"] },
      "id:guest-user-id": { deny: ["group:runtime", "group:fs"] },
      "*": { deny: ["exec", "process", "write", "edit", "apply_patch"] },
    },
  },
}
```

कुंजियाँ स्पष्ट प्रीफ़िक्स का उपयोग करती हैं: `channel:<channelId>:<senderId>`, `id:<senderId>`, `e164:<phone>`, `username:<handle>`, `name:<displayName>`, या `"*"`। चैनल आईडी प्रामाणिक OpenClaw आईडी हैं; `teams` जैसे उपनाम `msteams` में सामान्यीकृत होते हैं। पुराने बिना-प्रीफ़िक्स वाले कुंजी मान केवल `id:` के रूप में स्वीकार किए जाते हैं। मिलान क्रम चैनल+आईडी, आईडी, e164, उपयोगकर्ता नाम, नाम और फिर वाइल्डकार्ड है।

प्रति-एजेंट `agents.entries.*.tools.toolsBySender` वैश्विक प्रेषक मिलान से मेल खाने पर उसे ओवरराइड करता है, भले ही `{}` नीति खाली हो।

### `tools.elevated`

सैंडबॉक्स के बाहर उन्नत exec अभिगम नियंत्रित करता है:

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- प्रति-एजेंट ओवरराइड (`agents.entries.*.tools.elevated`) केवल और अधिक प्रतिबंधित कर सकता है।
- `/elevated on|off|ask|full` प्रत्येक सत्र के लिए स्थिति संग्रहीत करता है; इनलाइन निर्देश केवल एक संदेश पर लागू होते हैं।
- उन्नत `exec` सैंडबॉक्सिंग को बायपास करता है और कॉन्फ़िगर किए गए एस्केप पथ का उपयोग करता है (डिफ़ॉल्ट रूप से `gateway`, या जब exec लक्ष्य `node` हो तब `node`)।

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      approvalRunningNoticeMs: 10000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      commandHighlighting: false,
      applyPatch: {
        enabled: true,
        allowModels: ["gpt-5.6-sol"],
      },
    },
  },
}
```

दिखाए गए मान `applyPatch.allowModels` को छोड़कर डिफ़ॉल्ट हैं (डिफ़ॉल्ट रूप से खाली/असेट नहीं, जिसका अर्थ है कि कोई भी संगत मॉडल `apply_patch` का उपयोग कर सकता है)। अनुमोदन-समर्थित exec के लंबे समय तक चलने पर `approvalRunningNoticeMs` एक चालू होने की सूचना जारी करता है; `0` इसे अक्षम करता है।

### `tools.loopDetection`

टूल-लूप सुरक्षा जाँच **डिफ़ॉल्ट रूप से अक्षम** हैं। पहचान सक्रिय करने के लिए `enabled: true` सेट करें। सेटिंग्स को `tools.loopDetection` में वैश्विक रूप से परिभाषित और `agents.entries.*.tools.loopDetection` पर प्रति-एजेंट ओवरराइड किया जा सकता है।

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
    },
  },
}
```

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // or BRAVE_API_KEY env (Brave provider)
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // optional; omit for auto-detect
        maxChars: 20000,
        maxCharsCap: 20000,
        maxResponseBytes: 750000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

दिखाए गए मान `provider` और `userAgent` को छोड़कर डिफ़ॉल्ट हैं। `maxResponseBytes` को 32000–10000000 तक सीमित किया जाता है; `maxChars` को `maxCharsCap` तक सीमित किया जाता है (बड़ी प्रतिक्रियाओं की अनुमति देने के लिए `maxCharsCap` बढ़ाएँ)।

### `tools.media`

आने वाले मीडिया को समझने की क्षमता (छवि/ऑडियो/वीडियो) कॉन्फ़िगर करता है:

```json5
{
  tools: {
    media: {
      concurrency: 2,
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          capabilities: ["audio"],
        },
        { provider: "ollama", model: "gemma4:26b", capabilities: ["image"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["video"] },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-mini-transcribe" },
      image: { enabled: true, preferredModel: "ollama/gemma4:26b" },
      video: { enabled: true },
    },
  },
}
```

`tools.media.models` ही एकमात्र कॉन्फ़िगर की गई मॉडल सूची है। प्रत्येक प्रविष्टि उन क्षमताओं को घोषित करती है जिन्हें वह संभालती है। वैकल्पिक `preferredModel` चयनकर्ता `provider/model`, किसी मॉडल आईडी, प्रदाता-डिफ़ॉल्ट प्रविष्टियों के लिए `provider:<id>`, या `cli:command` स्वीकार करता है; मेल खाने वाली प्रविष्टियाँ उस क्षमता के फ़ॉलबैक क्रम में आगे चली जाती हैं। प्रति-क्षमता प्रॉम्प्ट, सीमाएँ, अनुरोध सेटिंग्स, स्कोप, अटैचमेंट नीति और ऑडियो ट्रांसक्रिप्ट प्रतिध्वनि कॉन्फ़िगर किए गए तथा स्वतः पहचाने गए मॉडलों के लिए डिफ़ॉल्ट रहते हैं; कोई मॉडल प्रविष्टि मॉडल-विशिष्ट फ़ील्ड ओवरराइड कर सकती है।

<AccordionGroup>
  <Accordion title="मीडिया मॉडल प्रविष्टि फ़ील्ड">
    **प्रदाता प्रविष्टि** (`type: "provider"` या छोड़ी गई):

    - `provider`: API प्रदाता आईडी (`openai`, `anthropic`, `google`/`gemini`, `groq`, आदि)
    - `model`: मॉडल आईडी ओवरराइड
    - `profile` / `preferredProfile`: `auth-profiles.json` प्रोफ़ाइल चयन

    **CLI प्रविष्टि** (`type: "cli"`):

    - `command`: चलाने के लिए निष्पादन योग्य फ़ाइल
    - `args`: टेम्पलेट वाले आर्ग्युमेंट (`{{AttachmentPath}}`, `{{AttachmentUrl}}`, `{{AttachmentContentType}}`, `{{AttachmentDir}}`, `{{AttachmentIndex}}`, `{{Prompt}}`, `{{MaxChars}}`, आदि का समर्थन करता है; `openclaw doctor --fix` अप्रचलित `{input}` प्लेसहोल्डर को `{{AttachmentPath}}` में माइग्रेट करता है)। पुराने `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}`, और `{{MediaDir}}` उपनाम अपनी संगतता अवधि के दौरान उपलब्ध रहते हैं, लेकिन अप्रचलित हैं।

    **सामान्य फ़ील्ड:**

    - `capabilities`: `image`, `audio`, और `video` में से एक या अधिक वाली सूची।
    - `prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language`: प्रति-प्रविष्टि ओवरराइड।
    - एजेंट द्वारा स्पष्ट `image` टूल कॉल करने पर मेल खाने वाली छवि मॉडल `timeoutSeconds` प्रविष्टियाँ भी लागू होती हैं। छवि समझने के लिए यह टाइमआउट स्वयं अनुरोध पर लागू होता है और पहले किए गए तैयारी कार्य के कारण कम नहीं होता।
    - विफलताएँ अगली प्रविष्टि पर फ़ॉलबैक करती हैं।

    प्रदाता प्रमाणीकरण मानक क्रम का अनुसरण करता है: `auth-profiles.json` → परिवेश चर → `models.providers.*.apiKey`।

  </Accordion>
</AccordionGroup>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

नियंत्रित करता है कि सत्र टूल (`sessions_list`, `sessions_history`, `sessions_send`) किन सत्रों को लक्षित कर सकते हैं।

डिफ़ॉल्ट: `tree` (वर्तमान सत्र + इसके द्वारा शुरू किए गए सत्र, जैसे सबएजेंट, साथ ही उसी एजेंट के परिवेशी
रूप से देखे जाने वाले समूह सत्र)।

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

<AccordionGroup>
  <Accordion title="दृश्यता स्कोप">
    - `self`: केवल वर्तमान सत्र कुंजी।
    - `tree`: वर्तमान सत्र + वर्तमान सत्र द्वारा शुरू किए गए सत्र (सबएजेंट)। पठन संचालनों के लिए इसमें उसी एजेंट के वे समूह सत्र भी शामिल होते हैं जिन्हें वर्तमान सत्र परिवेशी समूह जागरूकता के माध्यम से देखता है।
    - `agent`: वर्तमान एजेंट आईडी से संबंधित कोई भी सत्र (यदि आप उसी एजेंट आईडी के अंतर्गत प्रति-प्रेषक सत्र चलाते हैं, तो इसमें अन्य उपयोगकर्ता शामिल हो सकते हैं)।
    - `all`: कोई भी सत्र। क्रॉस-एजेंट लक्ष्यीकरण के लिए अब भी `tools.agentToAgent` आवश्यक है।
    - सैंडबॉक्स सीमा: जब वर्तमान सत्र सैंडबॉक्स में हो और `agents.defaults.sandbox.sessionToolsVisibility="spawned"` (डिफ़ॉल्ट) हो, तो दृश्यता को `tree` पर बाध्य किया जाता है, भले ही `tools.sessions.visibility="all"` हो।
    - जब `all` न हो, तब `sessions_list` में प्रभावी मोड का वर्णन करने वाला एक संक्षिप्त `visibility` फ़ील्ड
      और यह चेतावनी शामिल होती है कि वर्तमान स्कोप के बाहर कुछ सत्र
      छोड़े जा सकते हैं।

  </Accordion>
</AccordionGroup>

डिफ़ॉल्ट `session.dmScope: "main"` के साथ, किसी समूह में मानवीय गतिविधि उसी एजेंट के उस समूह
सत्र को एजेंट के मुख्य सत्र के लिए परिवेशी रूप से दृश्यमान बनाती है। बहु-उपयोगकर्ता सेटअप में, `"main"` उपयोगकर्ताओं के बीच
एक DM सत्र भी साझा करता है, इसलिए वहाँ रूट किया गया प्रत्येक उपयोगकर्ता परिवेशी रूप से देखे जाने वाले समूहों से पढ़ सकता है,
जिसमें सत्र-मेमोरी `memory_search` के माध्यम से पढ़ना भी शामिल है। DM पृथक्करण के लिए प्रति-पीयर `dmScope` का उपयोग करें, या परिवेशी रूप से देखे जाने वाले सत्रों से पढ़ने से बाहर रहने के लिए
`tools.sessions.visibility: "self"` सेट करें।

### `tools.sessions_spawn`

`sessions_spawn` के लिए इनलाइन अटैचमेंट समर्थन नियंत्रित करता है।

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // opt-in: set true to allow inline file attachments
        maxTotalBytes: 5242880, // 5 MB total across all files
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 MB per file
        retainOnSessionKeep: false, // keep attachments when cleanup="keep"
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="अटैचमेंट संबंधी टिप्पणियाँ">
    - अटैचमेंट के लिए `enabled: true` आवश्यक है।
    - सबएजेंट अटैचमेंट को चाइल्ड कार्यस्थान में `.openclaw/attachments/<uuid>/` पर `.manifest.json` के साथ मूर्त रूप दिया जाता है।
    - ACP अटैचमेंट केवल छवियाँ होते हैं और समान फ़ाइल संख्या, प्रति-फ़ाइल बाइट तथा कुल बाइट सीमाएँ पार होने के बाद ACP रनटाइम को इनलाइन अग्रेषित किए जाते हैं।
    - अटैचमेंट सामग्री को ट्रांसक्रिप्ट स्थायित्व से स्वचालित रूप से संशोधित करके छिपा दिया जाता है।
    - Base64 इनपुट की सख्त वर्णमाला/पैडिंग जाँच और डीकोड-पूर्व आकार सुरक्षा के साथ पुष्टि की जाती है।
    - सबएजेंट अटैचमेंट फ़ाइल अनुमतियाँ डायरेक्टरी के लिए `0700` और फ़ाइलों के लिए `0600` होती हैं।
    - सबएजेंट क्लीनअप `cleanup` नीति का अनुसरण करता है: `delete` हमेशा अटैचमेंट हटाता है; `keep` उन्हें केवल तब बनाए रखता है जब `retainOnSessionKeep: true` हो।

  </Accordion>
</AccordionGroup>

<a id="toolsexperimental"></a>

### `tools.experimental`

प्रयोगात्मक अंतर्निहित टूल फ़्लैग। जब तक कोई strict-agentic GPT-5 स्वतः-सक्षम नियम लागू न हो, डिफ़ॉल्ट रूप से बंद।

```json5
{
  tools: {
    experimental: {
      planTool: true, // enable experimental update_plan
    },
  },
}
```

- `planTool`: सामान्य से अधिक जटिल बहु-चरणीय कार्य ट्रैकिंग के लिए संरचित `update_plan` टूल सक्षम करता है।
- डिफ़ॉल्ट: `false`, जब तक कि GPT-5-परिवार की मॉडल आईडी के विरुद्ध `openai` प्रदाता रन के लिए `agents.defaults.embeddedAgent.executionContract` (या प्रति-एजेंट ओवरराइड) को `"strict-agentic"` पर सेट न किया गया हो (इसमें OpenAI Codex CLI रन भी शामिल हैं, क्योंकि Codex प्रमाणीकरण/मॉडल रूटिंग `openai` प्रदाता के अंतर्गत रहती है)। उस स्कोप के बाहर टूल को चालू रखने के लिए `true`, या strict-agentic GPT-5 रन के लिए भी इसे बंद रखने हेतु `false` सेट करें।
- सक्षम होने पर, सिस्टम प्रॉम्प्ट उपयोग संबंधी मार्गदर्शन भी जोड़ता है ताकि मॉडल इसका उपयोग केवल पर्याप्त कार्य के लिए करे और अधिकतम एक चरण को `in_progress` रखे।

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        announceTimeoutMs: 120000,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`: प्रारंभ किए गए उप-एजेंटों के लिए डिफ़ॉल्ट मॉडल। छोड़े जाने पर, उप-एजेंट कॉलर का मॉडल इनहेरिट करते हैं।
- `allowAgents`: जब अनुरोधकर्ता एजेंट अपना `subagents.allowAgents` सेट नहीं करता, तब `sessions_spawn` के लिए कॉन्फ़िगर किए गए लक्ष्य एजेंट आईडी की डिफ़ॉल्ट अनुमत-सूची (`["*"]` = कोई भी कॉन्फ़िगर किया गया लक्ष्य; डिफ़ॉल्ट: केवल वही एजेंट)। जिन पुरानी प्रविष्टियों का एजेंट कॉन्फ़िगरेशन हटा दिया गया है, उन्हें `sessions_spawn` अस्वीकार करता है और `agents_list` से छोड़ दिया जाता है; उन्हें साफ़ करने के लिए `openclaw doctor --fix` चलाएँ।
- `maxConcurrent`: समवर्ती उप-एजेंट रन की अधिकतम संख्या। डिफ़ॉल्ट: `8`।
- `runTimeoutSeconds`: जब कॉलर अपना ओवरराइड पास नहीं करता, तब `sessions_spawn` के लिए टाइमआउट (सेकंड)। डिफ़ॉल्ट: `0` (कोई टाइमआउट नहीं); ऊपर दिखाया गया `900` आम तौर पर चुना जाने वाला मान है, अंतर्निहित डिफ़ॉल्ट नहीं।
- `announceTimeoutMs`: Gateway `agent` घोषणा डिलीवरी प्रयासों के लिए प्रति-कॉल टाइमआउट (मिलीसेकंड)। डिफ़ॉल्ट: `120000`। अस्थायी पुनःप्रयासों के कारण घोषणा की कुल प्रतीक्षा एक कॉन्फ़िगर किए गए टाइमआउट से अधिक हो सकती है।
- `archiveAfterMinutes`: उप-एजेंट सत्र पूरा होने के बाद उसे स्वतः संग्रहित किए जाने से पहले के मिनट। डिफ़ॉल्ट: `60`; `0` स्वतः-संग्रहण अक्षम करता है।
- प्रति-उप-एजेंट टूल नीति: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`।

---

## कस्टम प्रदाता और आधार URL

प्रदाता Plugin अपनी मॉडल कैटलॉग पंक्तियाँ प्रकाशित करते हैं। कस्टम प्रदाताओं को कॉन्फ़िगरेशन में `models.providers` या `~/.openclaw/agents/<agentId>/agent/models.json` के माध्यम से जोड़ें।

कस्टम/स्थानीय प्रदाता `baseUrl` को कॉन्फ़िगर करना मॉडल HTTP अनुरोधों के लिए सीमित नेटवर्क विश्वास निर्णय भी है: OpenClaw अलग कॉन्फ़िगरेशन विकल्प जोड़े या अन्य निजी मूलों पर विश्वास किए बिना, सुरक्षित फ़ेच पथ के माध्यम से ठीक उसी `scheme://host:port` मूल की अनुमति देता है।

```json5
{
  models: {
    mode: "merge", // मर्ज (डिफ़ॉल्ट) | प्रतिस्थापित करें
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai | आदि।
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="प्रमाणीकरण और मर्ज प्राथमिकता">
    - कस्टम प्रमाणीकरण आवश्यकताओं के लिए `authHeader: true` + `headers` का उपयोग करें।
    - एजेंट कॉन्फ़िगरेशन रूट को `OPENCLAW_AGENT_DIR` से ओवरराइड करें।
    - मेल खाने वाली प्रदाता आईडी के लिए मर्ज प्राथमिकता:
      - एजेंट के गैर-रिक्त `models.json` `baseUrl` मानों को प्राथमिकता मिलती है।
      - एजेंट के गैर-रिक्त `apiKey` मानों को केवल तभी प्राथमिकता मिलती है, जब वह प्रदाता वर्तमान कॉन्फ़िगरेशन/प्रमाणीकरण-प्रोफ़ाइल संदर्भ में SecretRef द्वारा प्रबंधित न हो।
      - SecretRef द्वारा प्रबंधित प्रदाता के `apiKey` मानों को समाधान किए गए सीक्रेट स्थायी रूप से सहेजने के बजाय स्रोत मार्करों (पर्यावरण संदर्भों के लिए `ENV_VAR_NAME`, फ़ाइल/निष्पादन संदर्भों के लिए `secretref-managed`) से रीफ़्रेश किया जाता है।
      - SecretRef द्वारा प्रबंधित प्रदाता हेडर मानों को स्रोत मार्करों (पर्यावरण संदर्भों के लिए `secretref-env:ENV_VAR_NAME`, फ़ाइल/निष्पादन संदर्भों के लिए `secretref-managed`) से रीफ़्रेश किया जाता है।
      - एजेंट के रिक्त या अनुपस्थित `apiKey`/`baseUrl` कॉन्फ़िगरेशन में `models.providers` पर वापस जाते हैं।
      - मेल खाने वाले मॉडल `contextWindow`/`maxTokens`: स्पष्ट कॉन्फ़िगरेशन मान मौजूद और मान्य (धनात्मक परिमित संख्या) होने पर उसे प्राथमिकता मिलती है; अन्यथा अंतर्निहित/जनरेट किया गया कैटलॉग मान उपयोग होता है।
      - मेल खाने वाला मॉडल `contextTokens` भी स्पष्ट-मान-को-प्राथमिकता-अन्यथा-अंतर्निहित नियम का पालन करता है; मूल मॉडल मेटाडेटा बदले बिना प्रभावी संदर्भ सीमित करने के लिए इसका उपयोग करें।
      - प्रदाता-Plugin कैटलॉग एजेंट की Plugin स्थिति के अंतर्गत जनरेट किए गए Plugin-स्वामित्व वाले कैटलॉग खंडों के रूप में संग्रहित होते हैं।
      - जब आप चाहते हैं कि कॉन्फ़िगरेशन `models.json` को पूरी तरह पुनः लिखे और Plugin-स्वामित्व वाले कैटलॉग खंडों को मर्ज करना छोड़ दे, तब `models.mode: "replace"` का उपयोग करें।
      - मार्कर का स्थायित्व स्रोत-प्रामाणिक है: मार्कर समाधान किए गए रनटाइम सीक्रेट मानों से नहीं, बल्कि सक्रिय स्रोत कॉन्फ़िगरेशन स्नैपशॉट (समाधान-पूर्व) से लिखे जाते हैं।

  </Accordion>
</AccordionGroup>

### प्रदाता फ़ील्ड का विवरण

<AccordionGroup>
  <Accordion title="शीर्ष-स्तरीय कैटलॉग">
    - `models.mode`: प्रदाता कैटलॉग व्यवहार (`merge` या `replace`)।
    - `models.providers`: प्रदाता आईडी द्वारा कुंजीबद्ध कस्टम प्रदाता मैप।
      - सुरक्षित संपादन: योगात्मक अपडेट के लिए `openclaw config set models.providers.<id> '<json>' --strict-json --merge` या `openclaw config set models.providers.<id>.models '<json-array>' --strict-json --merge` का उपयोग करें। जब तक आप `--replace` पास नहीं करते, `config set` विनाशकारी प्रतिस्थापनों को अस्वीकार करता है।

  </Accordion>
  <Accordion title="प्रदाता कनेक्शन और प्रमाणीकरण">
    - `models.providers.*.api`: अनुरोध अडैप्टर (`openai-completions`, `openai-responses`, `openai-chatgpt-responses`, `anthropic-messages`, `google-generative-ai`, `google-vertex`, `github-copilot`, `bedrock-converse-stream`, `ollama`, `azure-openai-responses`)। MLX, vLLM, SGLang और अधिकांश OpenAI-संगत स्थानीय सर्वरों जैसे स्वयं होस्ट किए गए `/v1/chat/completions` बैकएंड के लिए `openai-completions` का उपयोग करें। `baseUrl` वाले लेकिन `api` रहित कस्टम प्रदाता का डिफ़ॉल्ट `openai-completions` होता है; `openai-responses` केवल तभी सेट करें, जब बैकएंड `/v1/responses` का समर्थन करता हो।
    - `models.providers.*.apiKey`: प्रदाता क्रेडेंशियल (SecretRef/पर्यावरण प्रतिस्थापन को प्राथमिकता दें)।
    - `models.providers.*.auth`: प्रमाणीकरण रणनीति (`api-key`, `token`, `oauth`, `aws-sdk`)।
    - `models.providers.*.contextWindow`: जब मॉडल प्रविष्टि `contextWindow` सेट नहीं करती, तब इस प्रदाता के अंतर्गत मॉडलों के लिए डिफ़ॉल्ट मूल संदर्भ विंडो।
    - `models.providers.*.contextTokens`: जब मॉडल प्रविष्टि `contextTokens` सेट नहीं करती, तब इस प्रदाता के अंतर्गत मॉडलों के लिए डिफ़ॉल्ट प्रभावी रनटाइम संदर्भ सीमा।
    - `models.providers.*.maxTokens`: जब मॉडल प्रविष्टि `maxTokens` सेट नहीं करती, तब इस प्रदाता के अंतर्गत मॉडलों के लिए डिफ़ॉल्ट आउटपुट-टोकन सीमा।
    - `models.providers.*.timeoutSeconds`: वैकल्पिक प्रति-प्रदाता मॉडल HTTP अनुरोध टाइमआउट, सेकंड में, जिसमें कनेक्ट, हेडर, बॉडी और कुल अनुरोध निरस्तीकरण प्रबंधन शामिल हैं।
    - `models.providers.*.injectNumCtxForOpenAICompat`: Ollama + `openai-completions` के लिए अनुरोधों में `options.num_ctx` इंजेक्ट करें (डिफ़ॉल्ट: `true`)।
    - `models.providers.*.authHeader`: आवश्यकता होने पर `Authorization` हेडर में क्रेडेंशियल परिवहन बाध्य करें।
    - `models.providers.*.baseUrl`: अपस्ट्रीम API आधार URL।
    - `models.providers.*.headers`: प्रॉक्सी/टेनेंट रूटिंग के लिए अतिरिक्त स्थिर हेडर।

  </Accordion>
  <Accordion title="अनुरोध परिवहन ओवरराइड">
    `models.providers.*.request`: मॉडल-प्रदाता HTTP अनुरोधों के लिए परिवहन ओवरराइड।

    - `request.headers`: अतिरिक्त हेडर (प्रदाता डिफ़ॉल्ट के साथ मर्ज किए जाते हैं)। मान SecretRef स्वीकार करते हैं।
    - `request.auth`: प्रमाणीकरण रणनीति ओवरराइड। मोड: `"provider-default"` (प्रदाता के अंतर्निहित प्रमाणीकरण का उपयोग करें), `"authorization-bearer"` (`token` के साथ), `"header"` (`headerName`, `value` और वैकल्पिक `prefix` के साथ)।
    - `request.proxy`: HTTP प्रॉक्सी ओवरराइड। मोड: `"env-proxy"` (`HTTP_PROXY`/`HTTPS_PROXY` पर्यावरण चरों का उपयोग करें), `"explicit-proxy"` (`url` के साथ)। दोनों मोड वैकल्पिक `tls` उप-ऑब्जेक्ट स्वीकार करते हैं।
    - `request.tls`: सीधे कनेक्शन के लिए TLS ओवरराइड। फ़ील्ड: `ca`, `cert`, `key`, `passphrase` (सभी SecretRef स्वीकार करते हैं), `serverName`, `insecureSkipVerify`।
    - `request.allowPrivateNetwork`: जब `true` हो, तब प्रदाता HTTP फ़ेच गार्ड के माध्यम से निजी, CGNAT या समान श्रेणियों के लिए मॉडल-प्रदाता HTTP अनुरोधों की अनुमति दें। कस्टम/स्थानीय प्रदाता आधार URL पहले से ही ठीक उसी कॉन्फ़िगर किए गए मूल पर विश्वास करते हैं, सिवाय मेटाडेटा/लिंक-लोकल मूलों के, जो स्पष्ट स्वीकृति के बिना अवरुद्ध रहते हैं। सटीक-मूल विश्वास से बाहर निकलने के लिए इसे `false` पर सेट करें। WebSocket हेडर/TLS के लिए उसी `request` का उपयोग करता है, लेकिन उस फ़ेच SSRF गेट का नहीं। डिफ़ॉल्ट `false`।

  </Accordion>
  <Accordion title="मॉडल कैटलॉग प्रविष्टियाँ">
    - `models.providers.*.models`: स्पष्ट प्रदाता मॉडल कैटलॉग प्रविष्टियाँ।
    - `models.providers.*.models.*.input`: मॉडल इनपुट माध्यम। केवल-पाठ मॉडलों के लिए `["text"]` और मूल छवि/विज़न मॉडलों के लिए `["text", "image"]` का उपयोग करें। छवि अनुलग्नक एजेंट टर्न में केवल तभी इंजेक्ट किए जाते हैं, जब चयनित मॉडल को छवि-सक्षम चिह्नित किया गया हो।
    - `models.providers.*.models.*.contextWindow`: मूल मॉडल संदर्भ विंडो मेटाडेटा। यह उस मॉडल के लिए प्रदाता-स्तरीय `contextWindow` को ओवरराइड करता है।
    - `models.providers.*.models.*.contextTokens`: वैकल्पिक रनटाइम संदर्भ सीमा। यह प्रदाता-स्तरीय `contextTokens` को ओवरराइड करता है; इसका उपयोग तब करें, जब आप मॉडल के मूल `contextWindow` से छोटा प्रभावी संदर्भ बजट चाहते हों; दोनों मान अलग होने पर `openclaw models list` उन्हें दिखाता है।

    #### कस्टम प्रदाता क्षमता घोषणाएँ

    प्रदाता कैटलॉग बंडल किए गए और कैटलॉग-ज्ञात मॉडल मार्गों के लिए `compat` के स्वामी होते हैं। उन फ़्लैग को कॉन्फ़िगरेशन में कॉपी न करें: जब कॉन्फ़िगर किए गए `api` और `baseUrl` अभी भी उस मार्ग की पहचान करते हैं, तब OpenClaw कैटलॉग पंक्ति का उपयोग करता है। `openclaw doctor --fix` मेल खाने वाले पुराने ओवरराइड हटाता है और समीक्षा के लिए भिन्न मानों की रिपोर्ट करता है।

    वास्तव में कस्टम प्रदाता, कस्टम मॉडल या किसी अलग एंडपॉइंट पर रूट किए गए कैटलॉग मॉडल के लिए `compat` ब्लॉक का समर्थन बना रहता है। केवल उस एंडपॉइंट के विरुद्ध सत्यापित क्षमताएँ सेट करें:

    | कस्टम-रूट कुंजी | रनटाइम अनुबंध |
    | --- | --- |
    | `supportsStore` | OpenAI `store` अनुरोध फ़ील्ड स्वीकार करता है। |
    | `supportsPromptCacheKey` | OpenAI प्रॉम्प्ट-कैश/सत्र-संबद्धता कुंजियाँ स्वीकार करता है। |
    | `supportsDeveloperRole` | `system` की आवश्यकता के बजाय `developer` संदेश स्वीकार करता है। |
    | `supportsReasoningEffort` | रीजनिंग-एफ़र्ट नियंत्रण स्वीकार करता है। |
    | `supportsTemperature` | इस मॉडल और अडैप्टर के लिए `temperature` स्वीकार करता है। |
    | `supportsUsageInStreaming` | स्ट्रीमिंग प्रतिक्रियाओं में उपयोग मेटाडेटा उत्सर्जित करता है। |
    | `supportsTools` | संरचित टूल/फ़ंक्शन कॉलिंग का समर्थन करता है। टूल अक्षम करने के लिए `false` सेट करें। |
    | `supportsStrictMode` | सख़्त टूल स्कीमा स्वीकार करता है। |
    | `requiresStringContent` | सामान्य-स्ट्रिंग Chat Completions संदेश सामग्री आवश्यक करता है। |
    | `strictMessageKeys` | आउटगोइंग संदेशों में केवल स्वीकृत कुंजियाँ होना आवश्यक करता है। |
    | `visibleReasoningDetailTypes` | ट्रांसक्रिप्ट में दिखाने के लिए सुरक्षित रीजनिंग विवरण ब्लॉक प्रकारों के नाम देता है। |
    | `supportedReasoningEfforts` | एंडपॉइंट द्वारा स्वीकृत रीजनिंग लेबल सूचीबद्ध करता है। |
    | `reasoningEffortMap` | OpenClaw के चिंतन लेबलों को एंडपॉइंट-विशिष्ट लेबलों से मैप करता है। |
    | `maxTokensField` | `max_tokens` या `max_completion_tokens` चुनता है। |
    | `thinkingFormat` | एंडपॉइंट की रीजनिंग पेलोड बोली चुनता है। |
    | `requiresToolResultName` | टूल-परिणाम संदेशों में टूल का नाम आवश्यक करता है। |
    | `requiresAssistantAfterToolResult` | टूल परिणामों के बाद सहायक संदेश आवश्यक करता है। |
    | `requiresThinkingAsText` | रीजनिंग को संरचित सामग्री के बजाय पाठ के रूप में पुनः चलाता है। |
    | `requiresReasoningContentOnAssistantMessages` | पुनः चलाने के दौरान DeepSeek-शैली `reasoning_content` को सुरक्षित रखता है। |
    | `toolSchemaProfile` | प्रदाता-परिभाषित टूल-स्कीमा सामान्यीकरण प्रोफ़ाइल चुनता है। |
    | `unsupportedToolSchemaKeywords` | एंडपॉइंट द्वारा अस्वीकार किए गए नामित JSON Schema कीवर्ड हटाता है। |
    | `toolCallArgumentsEncoding` | एंडपॉइंट का टूल-कॉल आर्ग्युमेंट एन्कोडिंग चुनता है। |
    | `requiresOpenAiAnthropicToolPayload` | OpenAI-आकार वाले टूल कॉल को Anthropic-परिवार पेलोड में बदलता है। |

  </Accordion>
  <Accordion title="Amazon Bedrock खोज">
    - `plugins.entries.amazon-bedrock.config.discovery`: Bedrock स्वचालित खोज सेटिंग्स का मूल।
    - `plugins.entries.amazon-bedrock.config.discovery.enabled`: अंतर्निहित खोज चालू/बंद करें।
    - `plugins.entries.amazon-bedrock.config.discovery.region`: खोज के लिए AWS क्षेत्र।
    - `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: लक्षित खोज के लिए वैकल्पिक प्रदाता-ID फ़िल्टर।
    - `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: खोज रीफ़्रेश के लिए पोलिंग अंतराल।
    - `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: खोजे गए मॉडल के लिए फ़ॉलबैक कॉन्टेक्स्ट विंडो।
    - `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: खोजे गए मॉडल के लिए फ़ॉलबैक अधिकतम आउटपुट टोकन।

  </Accordion>
</AccordionGroup>

इंटरैक्टिव कस्टम-प्रदाता ऑनबोर्डिंग ज्ञात विज़न-मॉडल-ID पैटर्न के लिए इमेज इनपुट का अनुमान लगाती है, जिनमें GPT-4o/GPT-4.1/GPT-5+, `o1`/`o3`/`o4` रीजनिंग फ़ैमिली, Claude, Gemini, कोई भी `-vl`-प्रत्यय वाला ID (Qwen-VL और इसी तरह के), और LLaVA, Pixtral, InternVL, Mllama, MiniCPM-V तथा GLM-4V जैसी नामित फ़ैमिली शामिल हैं; यह केवल-टेक्स्ट वाली ज्ञात फ़ैमिली (Llama, DeepSeek, Mistral/Mixtral, Kimi/Moonshot, Codestral, Devstral, Phi, QwQ, CodeLlama और vl/vision प्रत्यय के बिना मूल Qwen ID) के लिए अतिरिक्त प्रश्न छोड़ देती है। अज्ञात मॉडल ID के लिए अब भी इमेज समर्थन के बारे में पूछा जाता है। गैर-इंटरैक्टिव ऑनबोर्डिंग भी यही अनुमान उपयोग करती है; इमेज-सक्षम मेटाडेटा लागू करने के लिए `--custom-image-input` या केवल-टेक्स्ट मेटाडेटा लागू करने के लिए `--custom-text-input` पास करें।

### प्रदाता के उदाहरण

<AccordionGroup>
  <Accordion title="Cerebras (GLM 4.7 / GPT OSS)">
    आधिकारिक बाहरी `cerebras` प्रदाता Plugin इसे `openclaw onboard --auth-choice cerebras-api-key` के माध्यम से कॉन्फ़िगर कर सकता है। स्पष्ट प्रदाता कॉन्फ़िगरेशन का उपयोग केवल डिफ़ॉल्ट को ओवरराइड करते समय करें।

    ```json5
    {
      env: { CEREBRAS_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: {
            primary: "cerebras/zai-glm-4.7",
            fallbacks: ["cerebras/gpt-oss-120b"],
          },
          models: {
            "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
            "cerebras/gpt-oss-120b": { alias: "GPT OSS 120B (Cerebras)" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          cerebras: {
            baseUrl: "https://api.cerebras.ai/v1",
            apiKey: "${CEREBRAS_API_KEY}",
            api: "openai-completions",
            models: [
              { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
              { id: "gpt-oss-120b", name: "GPT OSS 120B (Cerebras)" },
            ],
          },
        },
      },
    }
    ```

    Cerebras के लिए `cerebras/zai-glm-4.7`; सीधे Z.AI के लिए `zai/glm-4.7` का उपयोग करें।

  </Accordion>
  <Accordion title="Kimi Coding">
    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: { "kimi/kimi-for-coding": { alias: "Kimi Code" } },
        },
      },
    }
    ```

    Anthropic-संगत, अंतर्निर्मित प्रदाता। शॉर्टकट: `openclaw onboard --auth-choice kimi-code-api-key`।

  </Accordion>
  <Accordion title="स्थानीय मॉडल (LM Studio)">
    [स्थानीय मॉडल](/hi/gateway/local-models) देखें। संक्षेप में: सक्षम हार्डवेयर पर LM Studio Responses API के माध्यम से एक बड़ा स्थानीय मॉडल चलाएँ; फ़ॉलबैक के लिए होस्ट किए गए मॉडल मर्ज किए रखें।
  </Accordion>
  <Accordion title="MiniMax M3 (प्रत्यक्ष)">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "Minimax" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          minimax: {
            baseUrl: "https://api.minimax.io/anthropic",
            apiKey: "${MINIMAX_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.12, cacheWrite: 0 },
                contextWindow: 1000000,
                maxTokens: 131072,
              },
            ],
          },
        },
      },
    }
    ```

    `MINIMAX_API_KEY` सेट करें। शॉर्टकट: `openclaw onboard --auth-choice minimax-global-api` या `openclaw onboard --auth-choice minimax-cn-api`। मॉडल कैटलॉग डिफ़ॉल्ट रूप से M3 उपयोग करता है और इसमें M2.7 संस्करण भी शामिल हैं। Anthropic-संगत स्ट्रीमिंग पथ पर, OpenClaw डिफ़ॉल्ट रूप से MiniMax M2.x थिंकिंग अक्षम करता है, जब तक कि आप स्वयं स्पष्ट रूप से `thinking` सेट न करें; MiniMax-M3 (और M3.x) डिफ़ॉल्ट रूप से प्रदाता के छोड़े गए/अनुकूली थिंकिंग पथ पर बना रहता है। `/fast on` या `params.fastMode: true`, `MiniMax-M2.7` को `MiniMax-M2.7-highspeed` में पुनर्लिखता है।

  </Accordion>
  <Accordion title="Moonshot AI (Kimi)">
    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: { "moonshot/kimi-k2.6": { alias: "Kimi K2.6" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
            ],
          },
        },
      },
    }
    ```

    चीन एंडपॉइंट के लिए: `baseUrl: "https://api.moonshot.cn/v1"` या `openclaw onboard --auth-choice moonshot-api-key-cn`।

    मूल Moonshot एंडपॉइंट साझा `openai-completions` ट्रांसपोर्ट पर स्ट्रीमिंग उपयोग संगतता घोषित करते हैं और OpenClaw इसे केवल अंतर्निर्मित प्रदाता ID के बजाय एंडपॉइंट क्षमताओं के आधार पर निर्धारित करता है।

  </Accordion>
  <Accordion title="OpenCode">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "opencode/claude-opus-4-6" },
          models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
        },
      },
    }
    ```

    `OPENCODE_API_KEY` (या `OPENCODE_ZEN_API_KEY`) सेट करें। Zen कैटलॉग के लिए `opencode/...` संदर्भ या Go कैटलॉग के लिए `opencode-go/...` संदर्भ उपयोग करें। शॉर्टकट: `openclaw onboard --auth-choice opencode-zen` या `openclaw onboard --auth-choice opencode-go`।

  </Accordion>
  <Accordion title="Synthetic (Anthropic-संगत)">
    ```json5
    {
      env: { SYNTHETIC_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
          models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          synthetic: {
            baseUrl: "https://api.synthetic.new/anthropic",
            apiKey: "${SYNTHETIC_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "hf:MiniMaxAI/MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```

    आधार URL में `/v1` नहीं होना चाहिए (Anthropic क्लाइंट इसे जोड़ता है)। शॉर्टकट: `openclaw onboard --auth-choice synthetic-api-key`।

  </Accordion>
  <Accordion title="Z.AI (GLM-4.7)">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-4.7" },
          models: { "zai/glm-4.7": {} },
        },
      },
    }
    ```

    `ZAI_API_KEY` सेट करें। मॉडल संदर्भ मानक `zai/*` प्रदाता ID का उपयोग करते हैं। शॉर्टकट: `openclaw onboard --auth-choice zai-api-key`।

    - सामान्य एंडपॉइंट: `https://api.z.ai/api/paas/v4`
    - कोडिंग एंडपॉइंट: `https://api.z.ai/api/coding/paas/v4`
    - डिफ़ॉल्ट `zai-api-key` प्रमाणीकरण विकल्प आपकी कुंजी की जाँच करता है और स्वचालित रूप से पता लगाता है कि वह किस एंडपॉइंट से संबंधित है (यदि पहचान अनिर्णायक हो, तो प्रॉम्प्ट पर वापस जाता है और डिफ़ॉल्ट रूप से Global चुनता है)। स्पष्ट चयन के लिए समर्पित CN और Coding-Plan प्रमाणीकरण विकल्प भी उपलब्ध हैं।
    - सामान्य एंडपॉइंट के लिए आधार URL ओवरराइड के साथ एक कस्टम प्रदाता परिभाषित करें।

  </Accordion>
</AccordionGroup>

---

## संबंधित

- [कॉन्फ़िगरेशन — एजेंट](/hi/gateway/config-agents)
- [कॉन्फ़िगरेशन — चैनल](/hi/gateway/config-channels)
- [कॉन्फ़िगरेशन संदर्भ](/hi/gateway/configuration-reference) — अन्य शीर्ष-स्तरीय कुंजियाँ
- [टूल और Plugin](/hi/tools)
