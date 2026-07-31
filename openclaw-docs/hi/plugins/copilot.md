---
read_when:
    - आप किसी एजेंट के लिए GitHub Copilot SDK हार्नेस का उपयोग करना चाहते हैं
    - आपको `copilot` रनटाइम के लिए कॉन्फ़िगरेशन उदाहरणों की आवश्यकता है
    - आप किसी एजेंट को सदस्यता-आधारित Copilot (github / openclaw / copilot) से जोड़ रहे हैं और चाहते हैं कि वह Copilot CLI के माध्यम से चले।
summary: बाहरी GitHub Copilot SDK हार्नेस के माध्यम से OpenClaw एम्बेडेड एजेंट टर्न चलाएँ
title: Copilot SDK हार्नेस
x-i18n:
    generated_at: "2026-07-27T21:20:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4b67959c2c72bda97a81d0b45bc32ba363373064ec40c54f9709705dd15dd9fc
    source_path: plugins/copilot.md
    workflow: 16
---

बाहरी `@openclaw/copilot` Plugin, OpenClaw के अंतर्निहित हार्नेस के बजाय GitHub Copilot CLI (`@github/copilot-sdk`) के माध्यम से एम्बेडेड सब्सक्रिप्शन Copilot
एजेंट टर्न चलाता है। Copilot CLI सत्र निम्न-स्तरीय
एजेंट लूप का स्वामी होता है: नेटिव टूल निष्पादन, नेटिव Compaction (`infiniteSessions`), और
`copilotHome` के अंतर्गत CLI-प्रबंधित थ्रेड स्थिति। OpenClaw अब भी चैट
चैनलों, सत्र फ़ाइलों, मॉडल चयन, डायनेमिक टूल (ब्रिज किए गए), अनुमोदनों,
मीडिया डिलीवरी, दृश्यमान ट्रांसक्रिप्ट मिरर, `/btw` सहायक प्रश्नों (देखें
[सहायक प्रश्न (`/btw`)](#side-questions-btw)), और `openclaw doctor` का स्वामी रहता है।

व्यापक मॉडल/प्रोवाइडर/रनटाइम विभाजन के लिए,
[एजेंट रनटाइम](/hi/concepts/agent-runtimes) से शुरू करें।

## आवश्यकताएँ

- `@openclaw/copilot` Plugin इंस्टॉल किया हुआ OpenClaw।
- यदि आपका कॉन्फ़िगरेशन `plugins.allow` का उपयोग करता है, तो `copilot` (Plugin द्वारा
  घोषित मैनिफ़ेस्ट आईडी) शामिल करें। npm पैकेज नाम
  `@openclaw/copilot` के लिए अलाउलिस्ट प्रविष्टि मेल नहीं खाएगी और
  `agentRuntime.id: "copilot"` सेट होने पर भी Plugin को अवरुद्ध रखेगी।
- ऐसा GitHub Copilot सब्सक्रिप्शन जो Copilot CLI चला सके, या
  हेडलेस अथवा Cron रन के लिए `gitHubToken` env var / ऑथ-प्रोफ़ाइल प्रविष्टि।
- लिखने योग्य `copilotHome` डायरेक्टरी। OpenClaw द्वारा एजेंट डायरेक्टरी
  दिए जाने पर डिफ़ॉल्ट `<agentDir>/copilot`, अन्यथा
  `~/.openclaw/agents/<agentId>/copilot` होता है।

`openclaw doctor` सत्र-स्थिति स्वामित्व और भावी कॉन्फ़िगरेशन माइग्रेशन के लिए
Plugin का [डॉक्टर अनुबंध](#doctor) चलाता है। यह
Copilot CLI परिवेश की जाँच नहीं करता।

## इंस्टॉल करना

Copilot रनटाइम बाहरी Plugin के रूप में वितरित होता है, ताकि मुख्य `openclaw`
पैकेज में `@github/copilot-sdk` या इसका प्लेटफ़ॉर्म-विशिष्ट
`@github/copilot-<platform>-<arch>` CLI बाइनरी (दोनों मिलाकर लगभग 260 MB) शामिल न हों।
इसे केवल उन एजेंटों के लिए इंस्टॉल करें जो इस रनटाइम को चुनते हैं:

```bash
openclaw plugins install @openclaw/copilot
```

जब आप पहली बार कोई `github-copilot/*` मॉडल चुनते हैं **और** आपका कॉन्फ़िगरेशन
`agentRuntime: { id: "copilot" }` के माध्यम से उस मॉडल (या उसके प्रोवाइडर) को Copilot रनटाइम पर रूट करता है,
तो सेटअप विज़ार्ड Plugin को अपने-आप इंस्टॉल करता है; देखें
[त्वरित शुरुआत](#quickstart)। इस विकल्प को चुने बिना OpenClaw अपने अंतर्निहित
GitHub Copilot प्रोवाइडर का उपयोग करता है और इस Plugin को कभी इंस्टॉल नहीं करता।

रनटाइम SDK को इस क्रम में रिज़ॉल्व करता है:

1. इंस्टॉल किए गए `@openclaw/copilot` पैकेज से `import("@github/copilot-sdk")`।
2. फ़ॉलबैक डायरेक्टरी `~/.openclaw/npm-runtime/copilot/` (पुराना ऑन-डिमांड
   इंस्टॉल लक्ष्य)।

SDK न मिलने पर `COPILOT_SDK_MISSING` कोड वाली एक त्रुटि और ऊपर दिया गया
पुनः इंस्टॉल कमांड दिखता है।

## त्वरित शुरुआत

एक मॉडल (या एक प्रोवाइडर) को हार्नेस पर पिन करें:

```json5
{
  agents: {
    defaults: {
      model: "github-copilot/auto",
      models: {
        "github-copilot/auto": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
}
```

केवल एक मॉडल को हार्नेस से रूट करने के लिए उसकी प्रविष्टि पर `agentRuntime.id`
सेट करें, या उस प्रोवाइडर के अंतर्गत प्रत्येक मॉडल को रूट करने के लिए प्रोवाइडर पर सेट करें।

`github-copilot/auto` पोर्टेबल शुरुआती बिंदु है। नामित Copilot मॉडल
अकाउंट और संगठन की नीति पर निर्भर होते हैं; किसी मॉडल को पिन करने से पहले पुष्टि करें कि आपका प्रमाणित
Copilot CLI वास्तव में उसे उपलब्ध कराता है।

## समर्थित प्रोवाइडर

हार्नेस कैनोनिकल `github-copilot` प्रोवाइडर (`extensions/github-copilot` के स्वामित्व वाला)
के साथ-साथ कस्टम `models.providers` प्रविष्टियों का समर्थन करता है, जब
मॉडल में गैर-रिक्त `baseUrl` और इन `api` स्वरूपों में से एक हो:

- `anthropic-messages`
- `azure-openai-responses`
- `ollama` (OpenAI-संगत कम्प्लीशन)
- `openai-completions`
- `openai-responses`

नेटिव प्रोवाइडर आईडी (`openai`, `anthropic`, `google`, `ollama`) अपने
नेटिव रनटाइम के स्वामित्व में रहते हैं। इसके बजाय Copilot BYOK के माध्यम से एंडपॉइंट
रूट करने के लिए अलग कस्टम प्रोवाइडर आईडी का उपयोग करें।

Copilot BYOK एंडपॉइंट सार्वजनिक HTTPS URL होने चाहिए। हार्नेस
Copilot SDK को प्रत्येक प्रयास के लिए एक लूपबैक प्रॉक्सी देता है, फिर प्रोवाइडर ट्रैफ़िक को
OpenClaw के सुरक्षित फ़ेच पथ से अग्रेषित करता है, ताकि DNS पिनिंग और SSRF नीति
OpenClaw के स्वामित्व में रहें। स्थानीय Ollama, LM
Studio, या LAN मॉडल सर्वर के लिए नेटिव OpenClaw रनटाइम का उपयोग करें।

## BYOK

Copilot BYOK, SDK के सत्र-स्तरीय कस्टम प्रोवाइडर अनुबंध का उपयोग करता है। OpenClaw
रिज़ॉल्व किया हुआ मॉडल एंडपॉइंट, API कुंजी, बेयरर-टोकन मोड, हेडर, मॉडल
आईडी और संदर्भ/आउटपुट सीमाएँ पास करता है; प्रोवाइडर ट्रांसपोर्ट लॉजिक SDK में रहता है, न कि
कोर में।

```json5
{
  agents: {
    defaults: {
      model: "custom-proxy/llama-3.1-8b",
      models: {
        "custom-proxy/llama-3.1-8b": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      "custom-proxy": {
        baseUrl: "https://api.example.com/v1",
        apiKey: "${CUSTOM_PROXY_API_KEY}",
        api: "openai-responses",
        authHeader: true,
        models: [{ id: "llama-3.1-8b", name: "Llama 3.1 8B" }],
      },
    },
  },
}
```

BYOK सत्रों की कुंजियाँ सब्सक्रिप्शन सत्रों और अन्य
BYOK एंडपॉइंट या क्रेडेंशियल से अलग होती हैं। कुंजी, हेडर, मॉडल या एंडपॉइंट
बदलने पर असंगत स्थिति को पुनः शुरू करने के बजाय नया Copilot SDK सत्र शुरू होता है।

## प्रमाणीकरण

`runCopilotAttempt` के दौरान प्रति एजेंट लागू होने वाली प्राथमिकता:

1. प्रयास इनपुट पर **स्पष्ट `useLoggedInUser: true`** — एजेंट के `copilotHome` के अंतर्गत
   Copilot CLI के लॉग-इन उपयोगकर्ता का उपयोग करता है।
2. प्रयास इनपुट पर **स्पष्ट `gitHubToken`** (`profileId` +
   `profileVersion` आवश्यक)। सीधे CLI आह्वानों और उन परीक्षणों के लिए जिन्हें
   ऑथ-प्रोफ़ाइल रिज़ॉल्यूशन को बायपास करना हो।
3. **अनुबंध द्वारा रिज़ॉल्व किया गया `resolvedApiKey` + `authProfileId`** — मुख्य
   प्रोडक्शन पथ। कोर हार्नेस को
   आह्वान करने से पहले एजेंट की कॉन्फ़िगर की गई `github-copilot` ऑथ
   प्रोफ़ाइल (`src/infra/provider-usage.auth.ts:resolveProviderAuths`) रिज़ॉल्व करता है, इसलिए `github-copilot:<profile>` ऑथ प्रोफ़ाइल
   env var के बिना हेडलेस, Cron या मल्टी-प्रोफ़ाइल सेटअप में
   शुरू से अंत तक काम करती है।
4. **env-var फ़ॉलबैक**, इस क्रम में जाँचा जाता है (पहला गैर-रिक्त मान स्वीकार होता है,
   रिक्त स्ट्रिंग अनुपस्थित मानी जाती हैं; `extensions/github-copilot/auth.ts` में वितरित `github-copilot`
   प्रोवाइडर की प्राथमिकता को प्रतिबिंबित करता है):
   1. `OPENCLAW_GITHUB_TOKEN` — हार्नेस-विशिष्ट ओवरराइड; इससे आप सिस्टम-व्यापी `gh` /
      Copilot CLI कॉन्फ़िगरेशन को प्रभावित किए बिना OpenClaw हार्नेस के लिए
      टोकन पिन कर सकते हैं।
   2. `COPILOT_GITHUB_TOKEN` — मानक Copilot SDK / CLI env var।
   3. `GH_TOKEN` — मानक `gh` CLI env var।
   4. `GITHUB_TOKEN` — सामान्य GitHub टोकन फ़ॉलबैक।

   संश्लेषित पूल प्रोफ़ाइल आईडी `env:<NAME>` है; प्रोफ़ाइल संस्करण टोकन का
   अपरिवर्तनीय sha256 फ़िंगरप्रिंट है, इसलिए env मान बदलने पर
   क्लाइंट पूल साफ़ रूप से अमान्य हो जाता है।

5. कोई टोकन संकेत उपलब्ध न होने पर **डिफ़ॉल्ट `useLoggedInUser`**।

प्रत्येक एजेंट को अपना `copilotHome` मिलता है, ताकि एक ही मशीन पर एजेंटों के बीच
Copilot CLI टोकन, सत्र और कॉन्फ़िगरेशन कभी लीक न हों। डिफ़ॉल्ट:
`<agentDir>/copilot` (SDK स्थिति को OpenClaw की
`models.json` / `auth-profiles.json` वाली डायरेक्टरी से बाहर रखता है), या
एजेंट डायरेक्टरी न दिए जाने पर `~/.openclaw/agents/<agentId>/copilot`।
कस्टम स्थान के लिए प्रयास इनपुट पर `copilotHome: <path>` से ओवरराइड करें
(उदाहरण के लिए, माइग्रेशन हेतु साझा माउंट)।

लाइव हार्नेस परीक्षण सीधे टोकन के लिए `OPENCLAW_COPILOT_AGENT_LIVE_TOKEN` का उपयोग करते हैं।
साझा लाइव-परीक्षण सेटअप वास्तविक ऑथ प्रोफ़ाइल को अलग परीक्षण
होम में स्टेज करने के बाद `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`,
और `GITHUB_TOKEN` को हटा देता है, इसलिए समर्पित वेरिएबल से पास किया गया
`gh auth token` मान असंबंधित सुइट में लीक हुए बिना गलत स्किप से बचाता है।

## कॉन्फ़िगरेशन सतह

हार्नेस प्रति-प्रयास इनपुट (`runCopilotAttempt({...})`) और
`extensions/copilot/src/` के भीतर env डिफ़ॉल्ट के एक छोटे समूह से कॉन्फ़िगरेशन पढ़ता है:

| फ़ील्ड                    | उद्देश्य                                                                                                                                                                                                                                                                                         |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `copilotHome`            | प्रति-एजेंट CLI स्थिति डायरेक्टरी (ऊपर दिए गए डिफ़ॉल्ट)।                                                                                                                                                                                                                                                 |
| `model`                  | स्ट्रिंग या `{ provider, id, api?, baseUrl?, headers?, authHeader? }`। एजेंट के सामान्य मॉडल चयन का उपयोग करने के लिए इसे छोड़ दें; हार्नेस पुष्टि करता है कि रिज़ॉल्व किया गया प्रोवाइडर समर्थित है।                                                                                                                   |
| `reasoningEffort`        | `"low" \| "medium" \| "high" \| "xhigh"`। `auto-reply/thinking.ts` में OpenClaw के `ThinkLevel` / `ReasoningLevel` रिज़ॉल्यूशन से मैप होता है।                                                                                                                                                          |
| `infiniteSessionConfig`  | `harness.compact` द्वारा संचालित SDK `infiniteSessions` ब्लॉक के लिए वैकल्पिक ओवरराइड। इसे यथावत छोड़ना सुरक्षित है।                                                                                                                                                                                        |
| `hooksConfig`            | टूल/MCP, उपयोगकर्ता-प्रॉम्प्ट, सत्र और त्रुटि कॉलबैक के लिए वैकल्पिक नेटिव Copilot SDK `SessionHooks` कॉन्फ़िगरेशन। OpenClaw के पोर्टेबल लाइफ़साइकल हुक से अलग।                                                                                                                                   |
| `permissionPolicy`       | अंतर्निहित SDK टूल प्रकारों (`shell`, `write`, `read`, `url`, `mcp`, `memory`, `hook`) के लिए SDK के `onPermissionRequest` हैंडलर का वैकल्पिक ओवरराइड। सुरक्षा उपाय के रूप में डिफ़ॉल्ट `rejectAllPolicy` होता है; यह वास्तव में कभी क्यों सक्रिय नहीं होता, इसके लिए [अनुमतियाँ और ask_user](#permissions-and-ask_user) देखें। |
| `enableSessionTelemetry` | वैकल्पिक SDK सत्र टेलीमेट्री फ़्लैग।                                                                                                                                                                                                                                                            |

OpenClaw Plugin हुक को Copilot-विशिष्ट प्रयास कॉन्फ़िगरेशन की आवश्यकता नहीं है।
हार्नेस मानक हार्नेस हेल्पर के माध्यम से `before_prompt_build`, `llm_input`, `llm_output`, और `agent_end` चलाता है।
सफल SDK Compaction भी
`before_compaction` और `after_compaction` चलाते हैं। ब्रिज किए गए OpenClaw टूल
`before_tool_call` चलाते हैं और `after_tool_call` की रिपोर्ट करते हैं; `hooksConfig` उन
नेटिव, केवल-SDK कॉलबैक के लिए बना रहता है जिनका कोई पोर्टेबल समतुल्य नहीं है।

OpenClaw में किसी अन्य चीज़ को इन फ़ील्ड के बारे में जानने की आवश्यकता नहीं है। अन्य Plugin,
चैनल और कोर कोड केवल मानक `AgentHarnessAttemptParams` /
`AgentHarnessAttemptResult` स्वरूप देखते हैं।

## Compaction

जब `harness.compact` चलता है, तो Copilot SDK हार्नेस:

1. लंबित कार्य जारी रखे बिना ट्रैक किए गए SDK सत्र को फिर से शुरू करता है।
2. SDK के सत्र-स्कोप वाले इतिहास Compaction RPC को कॉल करता है।
3. वर्कस्पेस के अंतर्गत संगतता मार्कर फ़ाइलें लिखे बिना SDK Compaction परिणाम
   लौटाता है।

OpenClaw-पक्षीय ट्रांसक्रिप्ट मिरर (नीचे) Compaction के बाद के
संदेश प्राप्त करता रहता है, इसलिए उपयोगकर्ता को दिखाई देने वाला चैट इतिहास सुसंगत रहता है।

## ट्रांसक्रिप्ट मिररिंग

`runCopilotAttempt` प्रत्येक टर्न के मिरर किए जा सकने वाले संदेशों को
`extensions/copilot/src/dual-write-transcripts.ts` के माध्यम से OpenClaw ऑडिट ट्रांसक्रिप्ट में
डुअल-राइट करता है। मिरर का दायरा प्रति सत्र
(`copilot:${sessionId}`) होता है और कुंजी प्रति संदेश
(`${role}:${sha256_16(role,content)}`) होती है, इसलिए दोबारा उत्सर्जित पिछले-टर्न की प्रविष्टियाँ
डुप्लिकेट होने के बजाय डिस्क पर मौजूद कुंजियों से टकराती हैं।

विफलता-नियंत्रण की दो परतें मिरर को आवृत करती हैं, ताकि ट्रांसक्रिप्ट लिखने की
विफलता से प्रयास कभी विफल न हो: एक आंतरिक सर्वोत्तम-प्रयास रैपर, और प्रयास
स्तर पर अतिरिक्त सुरक्षा हेतु `.catch(...)`। विफलताओं को लॉग किया जाता है,
सामने नहीं लाया जाता।

## सहायक प्रश्न (`/btw`)

इस हार्नेस पर `/btw` **नेटिव नहीं** है। `createCopilotAgentHarness()`
जानबूझकर `harness.runSideQuestion` को अपरिभाषित छोड़ता है
(`extensions/copilot/harness.test.ts`, `describe("runSideQuestion")` में अभिकथित),
इसलिए OpenClaw का `/btw` डिस्पैचर (`src/agents/btw.ts`) उसी
पथ पर पहुँचता है जिसका उपयोग वह प्रत्येक गैर-Codex रनटाइम के लिए करता है:
कॉन्फ़िगर किए गए मॉडल प्रदाता को एक छोटे सहायक-प्रश्न प्रॉम्प्ट के साथ सीधे
कॉल किया जाता है और `streamSimple` के माध्यम से वापस स्ट्रीम किया जाता है
(कोई CLI सत्र नहीं, कोई अतिरिक्त पूल स्लॉट नहीं)।

इससे Copilot CLI सत्र एजेंट के मुख्य टर्न लूप के लिए आरक्षित रहते हैं, और
`/btw` का व्यवहार अन्य गैर-Codex रनटाइम के समान बना रहता है।

## Doctor

`extensions/copilot/doctor-contract-api.ts` को
`src/plugins/doctor-contract-registry.ts` द्वारा स्वतः लोड किया जाता है। यह निम्नलिखित प्रदान करता है:

- एक खाली `legacyConfigRules` (अभी तक कोई सेवानिवृत्त फ़ील्ड नहीं)।
- एक नो-ऑप `normalizeCompatibilityConfig` (रखा गया है ताकि भविष्य में फ़ील्ड
  सेवानिवृत्त करने के लिए स्रोत-वृक्ष में एक स्थिर स्थान उपलब्ध हो)।
- एक `sessionRouteStateOwners` प्रविष्टि: प्रदाता `github-copilot`, रनटाइम
  `copilot`, CLI सत्र कुंजी `copilot`, प्रमाणीकरण प्रोफ़ाइल उपसर्ग `github-copilot:`।

## सीमाएँ

- हार्नेस `github-copilot` के साथ स्वामित्व-रहित कस्टम BYOK प्रदाता आईडी पर दावा करता है।
  मैनिफ़ेस्ट-स्वामित्व वाली नेटिव प्रदाता आईडी अपने स्वामी रनटाइम पर ही रहती हैं, भले ही
  `agentRuntime.id` को बाध्य करके `copilot` किया गया हो।
- कोई TUI सतह नहीं; बिना समकक्ष सतह वाले रनटाइम के लिए PI का TUI फ़ॉलबैक बना रहता है।
- जब कोई एजेंट `copilot` पर स्विच करता है, तो PI सत्र स्थिति माइग्रेट नहीं होती।
  चयन प्रति प्रयास होता है; मौजूदा PI सत्र वैध बने रहते हैं।
- `ask_user` प्रदाता-निरपेक्ष Gateway प्रश्न रनटाइम का उपयोग करता है। Control
  UI अन्य OpenClaw प्रश्नों जैसा ही प्रश्न कार्ड दिखाता है, समर्थित
  चैनल विकल्प बटन रेंडर करते हैं, और अगला कतारबद्ध सादा-पाठ संदेश
  SDK अनुरोध के लौटने से पहले उस Gateway रिकॉर्ड का समाधान करता है।

## अनुमतियाँ और ask_user

ब्रिज किए गए OpenClaw टूल के लिए अनुमति प्रवर्तन SDK के
`onPermissionRequest` कॉलबैक के माध्यम से नहीं, बल्कि **टूल रैपर के भीतर** होता है।
PI द्वारा उपयोग किया जाने वाला वही
`wrapToolWithBeforeToolCallHook`
(`src/agents/agent-tools.before-tool-call.ts`) हर कोडिंग टूल पर
`createOpenClawCodingTools` द्वारा लागू किया जाता है: लूप पहचान, विश्वसनीय
Plugin नीतियाँ, टूल-कॉल-पूर्व हुक, और Gateway के माध्यम से दो-चरणीय Plugin अनुमोदन
(`plugin.approval.request`) सभी नेटिव PI प्रयासों वाले ठीक उसी कोड
पथ से चलते हैं।

Copilot टूल ब्रिज द्वारा लौटाए गए प्रत्येक SDK टूल को इनके साथ चिह्नित किया जाता है:

- `overridesBuiltInTool: true` — समान नाम वाले Copilot CLI के अंतर्निहित टूल को
  प्रतिस्थापित करता है (edit, read, write, bash, ...), ताकि प्रत्येक टूल कॉल
  वापस OpenClaw पर रूट हो।
- `skipPermission: true` — SDK को टूल का आह्वान करने से पहले
  `onPermissionRequest({kind: "custom-tool"})` सक्रिय न करने का निर्देश देता है।
  आवृत `execute()` पहले ही अधिक समृद्ध OpenClaw नीति जाँच करता है; SDK-स्तरीय
  प्रॉम्प्ट या तो OpenClaw के प्रवर्तन को शॉर्ट-सर्किट करेगा
  (सभी को अनुमति) या प्रत्येक टूल कॉल को अवरुद्ध करेगा (सभी को अस्वीकार) — इनमें से कोई भी PI
  समानता से मेल नहीं खाता।

स्रोत-वृक्ष में मौजूद Codex हार्नेस भी इसी विभाजन का उपयोग करता है: ब्रिज किए गए OpenClaw टूल
आवृत होते हैं (`extensions/codex/src/app-server/dynamic-tools.ts`) और
codex-app-server के अपने नेटिव अनुमोदन प्रकार
(`item/commandExecution/requestApproval`, `item/fileChange/requestApproval`,
`item/permissions/requestApproval`) `plugin.approval.request`
(`extensions/codex/src/app-server/approval-bridge.ts`) के माध्यम से रूट होते हैं। Copilot SDK
का समतुल्य — `onPermissionRequest` तक कभी पहुँचने वाले किसी भी गैर-`custom-tool`
प्रकार के लिए विफलता-बंद `rejectAllPolicy` — वही सुरक्षा तंत्र है, और व्यवहार में यह
कभी सक्रिय नहीं होता क्योंकि `overridesBuiltInTool: true` प्रत्येक
अंतर्निहित टूल को विस्थापित करता है।

आवृत-टूल परत द्वारा PI के समतुल्य नीतिगत निर्णय लेने के लिए,
हार्नेस संपूर्ण PI प्रयास-टूल संदर्भ
`createOpenClawCodingTools` को अग्रेषित करता है: पहचान (`senderIsOwner`, `memberRoleIds`,
`ownerOnlyToolAllowlist`, ...), चैनल/रूटिंग (`groupId`,
`currentChannelId`, `replyToMode`, संदेश-टूल टॉगल), प्रमाणीकरण
(`authProfileStore`), रन पहचान (`sandboxSessionKey`, `runId` से व्युत्पन्न
`sessionKey` / `runSessionKey`), मॉडल संदर्भ (`modelApi`,
`modelContextWindowTokens`, `modelCompat`, `modelHasVision`), और रन हुक
(`onToolOutcome`, `onYield`)। इन फ़ील्ड के बिना, केवल-स्वामी अनुमतिसूचियाँ
डिफ़ॉल्ट रूप से चुपचाप अस्वीकार करती हैं, Plugin-विश्वास नीतियाँ सही
दायरा निर्धारित नहीं कर पातीं, और `session_status: "current"` एक पुराने सैंडबॉक्स कुंजी में
समाधान होता है। ब्रिज बिल्डर `extensions/copilot/src/tool-bridge.ts` है, जो
`src/agents/embedded-agent-runner/run/attempt.ts:1262` पर PI की आधिकारिक कॉल को प्रतिबिंबित करता है।
`runAttempt` साझा
`resolveSandboxContext` सीम के माध्यम से सैंडबॉक्स संदर्भ निर्धारित करता है, SDK को एक प्रभावी कार्यशील निर्देशिका देता है,
और `sandbox` के साथ उप-एजेंट-सृजन कार्यक्षेत्र को टूल
ब्रिज में अग्रेषित करता है। ब्रिज उन सीमित टूल-निर्माण नियंत्रणों को भी अग्रेषित करता है
जिन्हें वह SDK सीमा पर लागू कर सकता है: `includeCoreTools`, रनटाइम टूल
अनुमतिसूची, और `toolConstructionPlan`।

PI समानता के लिए ब्रिज
`openclaw/plugin-sdk/agent-harness-tool-runtime` के साझा हार्नेस टूल-सतह सहायक का भी उपयोग करता है।
टूल-खोज सक्षम होने पर, SDK प्रत्येक OpenClaw टूल स्कीमा के बजाय संक्षिप्त
नियंत्रण टूल और एक छिपा हुआ कैटलॉग निष्पादक देखता है। कोड मोड सक्षम होने पर,
सहायक वही कोड-मोड नियंत्रण सतह और कैटलॉग जीवनचक्र बनाता है जिसका उपयोग अन्य
एजेंट हार्नेस करते हैं। स्थानीय-मॉडल के हल्के डिफ़ॉल्ट,
रनटाइम-संगत स्कीमा फ़िल्टरिंग, निर्देशिका हाइड्रेशन और कैटलॉग
क्लीनअप सभी साझा सहायक में रहते हैं, ताकि Copilot और Codex-सन्निकट
हार्नेस में विचलन न हो।

### सत्र-स्तरीय GitHub टोकन

Copilot SDK अनुबंध **क्लाइंट-स्तरीय** GitHub टोकन
(`CopilotClientOptions.gitHubToken`, जो स्वयं CLI प्रक्रिया को प्रमाणित करता है)
और **सत्र-स्तरीय** टोकन (`SessionConfig.gitHubToken`, जो उस सत्र के लिए
सामग्री बहिष्करण, मॉडल रूटिंग और कोटा निर्धारित करता है; `createSession` और
`resumeSession` दोनों पर मान्य) के बीच अंतर करता है। हार्नेस
`resolveCopilotAuth` के माध्यम से एक बार प्रमाणीकरण निर्धारित करता है और प्रमाणीकरण मोड
`gitHubToken` होने पर दोनों फ़ील्ड सेट करता है
(एक स्पष्ट `auth.gitHubToken` या कॉन्फ़िगर की गई `github-copilot`
प्रमाणीकरण प्रोफ़ाइल से अनुबंध-द्वारा-निर्धारित `resolvedApiKey`)। जब निर्धारित मोड
`useLoggedInUser` होता है, तो सत्र-स्तरीय फ़ील्ड छोड़ दिया जाता है, ताकि SDK
लॉग-इन पहचान से पहचान व्युत्पन्न करता रहे।

`ask_user`, `SessionConfig.onUserInputRequest` का उपयोग करता है। ब्रिज SDK
विकल्पों या विकल्प-रहित मुक्त-पाठ प्रॉम्प्ट को Gateway प्रश्नों के रूप में पंजीकृत करता है, निश्चित-विकल्प
अनुरोधों के लिए विकल्प सूचकांक या लेबल स्वीकार करता है, और SDK अनुरोध द्वारा अनुमति मिलने पर
मुक्त-रूप उत्तर स्वीकार करता है। OpenClaw प्रयास को निरस्त करने से
Gateway रिकॉर्ड रद्द हो जाता है और एक खाली SDK उत्तर लौटता है।

## संबंधित

- [एजेंट रनटाइम](/hi/concepts/agent-runtimes)
- [Codex हार्नेस](/hi/plugins/codex-harness)
- [एजेंट हार्नेस Plugin (SDK संदर्भ)](/hi/plugins/sdk-agent-harness)
