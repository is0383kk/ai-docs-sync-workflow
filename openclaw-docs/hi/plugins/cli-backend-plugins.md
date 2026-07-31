---
read_when:
    - आप एक स्थानीय AI CLI बैकएंड Plugin बना रहे हैं
    - आप acme-cli/model जैसे मॉडल रेफ़रेंस के लिए एक बैकएंड पंजीकृत करना चाहते हैं
    - आपको किसी तृतीय-पक्ष CLI को OpenClaw के टेक्स्ट फ़ॉलबैक रनर से मैप करना होगा
sidebarTitle: CLI backend plugins
summary: एक Plugin बनाएँ जो स्थानीय AI CLI बैकएंड पंजीकृत करे
title: CLI बैकएंड Plugin बनाना
x-i18n:
    generated_at: "2026-07-27T20:08:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1923b0829b46a309e4b5a6cbbbfd3dcb76a1e14fe4106310d7a9fb37bca41d70
    source_path: plugins/cli-backend-plugins.md
    workflow: 16
---

CLI बैकएंड Plugin, OpenClaw को टेक्स्ट इन्फ़रेंस बैकएंड के रूप में स्थानीय AI CLI को कॉल करने देते हैं। मॉडल रेफ़रेंस में बैकएंड, प्रोवाइडर प्रीफ़िक्स के रूप में दिखाई देता है:

```text
acme-cli/acme-large
```

CLI बैकएंड का उपयोग तब करें, जब अपस्ट्रीम इंटीग्रेशन पहले से स्थानीय कमांड के रूप में उपलब्ध हो, जब CLI स्थानीय लॉगिन स्थिति का स्वामी हो, या API प्रोवाइडर अनुपलब्ध होने पर फ़ॉलबैक के रूप में।

<Info>
  यदि अपस्ट्रीम सेवा सामान्य HTTP मॉडल API उपलब्ध कराती है, तो इसके बजाय
  [प्रोवाइडर Plugin](/hi/plugins/sdk-provider-plugins) लिखें। यदि अपस्ट्रीम
  रनटाइम पूर्ण एजेंट सेशन, टूल इवेंट, Compaction या बैकग्राउंड
  टास्क स्थिति का स्वामी है, तो [एजेंट हार्नेस](/hi/plugins/sdk-agent-harness) का उपयोग करें।
</Info>

## Plugin किन चीज़ों का स्वामी है

CLI बैकएंड Plugin के तीन कॉन्ट्रैक्ट होते हैं:

| कॉन्ट्रैक्ट             | फ़ाइल                   | उद्देश्य                                                   |
| -------------------- | ---------------------- | --------------------------------------------------------- |
| पैकेज एंट्री        | `package.json`         | OpenClaw को Plugin रनटाइम मॉड्यूल की ओर इंगित करती है              |
| मैनिफ़ेस्ट स्वामित्व   | `openclaw.plugin.json` | रनटाइम लोड होने से पहले बैकएंड आईडी घोषित करता है              |
| रनटाइम पंजीकरण | `index.ts`             | कमांड डिफ़ॉल्ट के साथ `api.registerCliBackend(...)` को कॉल करता है |

मैनिफ़ेस्ट डिस्कवरी मेटाडेटा है: यह CLI को निष्पादित या रनटाइम व्यवहार को पंजीकृत नहीं करता। रनटाइम व्यवहार तब शुरू होता है, जब Plugin एंट्री
`api.registerCliBackend(...)` को कॉल करती है।

## न्यूनतम बैकएंड Plugin

<Steps>
  <Step title="पैकेज मेटाडेटा बनाएँ">
    ```json package.json
    {
      "name": "@acme/openclaw-acme-cli",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      },
      "dependencies": {
        "openclaw": "^2026.3.24"
      },
      "devDependencies": {
        "typescript": "^5.9.0"
      }
    }
    ```

    प्रकाशित पैकेज में बिल्ड की गई JavaScript रनटाइम फ़ाइलें शामिल होनी चाहिए। यदि आपकी सोर्स
    एंट्री `./src/index.ts` है, तो बिल्ड किए गए JavaScript समकक्ष की ओर इंगित करने वाला `openclaw.runtimeExtensions` जोड़ें। [एंट्री पॉइंट](/hi/plugins/sdk-entrypoints) देखें।

  </Step>

  <Step title="बैकएंड स्वामित्व घोषित करें">
    ```json openclaw.plugin.json
    {
      "id": "acme-cli",
      "name": "Acme CLI",
      "description": "OpenClaw के माध्यम से Acme का स्थानीय AI CLI चलाएँ",
      "cliBackends": ["acme-cli"],
      "setup": {
        "cliBackends": ["acme-cli"],
        "requiresRuntime": false
      },
      "activation": {
        "onStartup": false
      },
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```

    `cliBackends` रनटाइम स्वामित्व सूची है; मॉडल चयन या `agentRuntime.id` में `acme-cli` का उल्लेख होने पर यह OpenClaw को
    Plugin अपने-आप लोड करने देती है।

    `setup.cliBackends` डिस्क्रिप्टर-फ़र्स्ट सेटअप सतह है। इसे तब जोड़ें, जब मॉडल डिस्कवरी, ऑनबोर्डिंग या स्थिति को Plugin रनटाइम लोड किए बिना बैकएंड पहचानना चाहिए। `requiresRuntime: false` का उपयोग केवल तब करें,
    जब वे स्थिर डिस्क्रिप्टर सेटअप के लिए पर्याप्त हों।

  </Step>

  <Step title="बैकएंड पंजीकृत करें">
    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      CLI_FRESH_WATCHDOG_DEFAULTS,
      CLI_RESUME_WATCHDOG_DEFAULTS,
      type CliBackendPlugin,
    } from "openclaw/plugin-sdk/cli-backend";

    function buildAcmeCliBackend(): CliBackendPlugin {
      return {
        id: "acme-cli",
        liveTest: {
          defaultModelRef: "acme-cli/acme-large",
          defaultImageProbe: false,
          defaultMcpProbe: false,
          docker: {
            npmPackage: "@acme/acme-cli",
            binaryName: "acme",
          },
        },
        config: {
          command: "acme",
          args: ["chat", "--output-format", "stream-json", "--prompt", "{prompt}"],
          resumeArgs: [
            "chat",
            "--resume",
            "{sessionId}",
            "--output-format",
            "stream-json",
            "--prompt",
            "{prompt}",
          ],
          output: "jsonl",
          resumeOutput: "jsonl",
          jsonlDialect: "gemini-stream-json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            large: "acme-large-2026",
            fast: "acme-fast-2026",
          },
          sessionArgs: ["--session", "{sessionId}"],
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptFileArg: "--system-file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          imagePathScope: "workspace",
          reliability: {
            watchdog: {
              fresh: { ...CLI_FRESH_WATCHDOG_DEFAULTS },
              resume: { ...CLI_RESUME_WATCHDOG_DEFAULTS },
            },
          },
          serialize: true,
        },
      };
    }

    export default definePluginEntry({
      id: "acme-cli",
      name: "Acme CLI",
      description: "OpenClaw के माध्यम से Acme का स्थानीय AI CLI चलाएँ",
      register(api) {
        api.registerCliBackend(buildAcmeCliBackend());
      },
    });
    ```

    बैकएंड आईडी का मैनिफ़ेस्ट की `cliBackends` एंट्री से मेल खाना आवश्यक है। पंजीकृत
    अडैप्टर आधिकारिक Plugin कोड है; OpenClaw कॉन्फ़िगरेशन बैकएंड चुनता है,
    लेकिन उसके कमांड कॉन्ट्रैक्ट को दोबारा नहीं लिखता।

  </Step>
</Steps>

## कॉन्फ़िगरेशन का आकार

`CliBackendConfig` बताता है कि OpenClaw को CLI कैसे लॉन्च और पार्स करना चाहिए। ऊपर दिया गया व्यावहारिक उदाहरण जानबूझकर बंडल किए गए
`google-gemini-cli` अडैप्टर के समान कमांड, रिज़्यूम, JSONL,
मॉडल-अलायस, सेशन, इमेज और वॉचडॉग फ़ील्ड का उपयोग करता है:

| फ़ील्ड                                                     | उपयोग                                                                               |
| --------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `command`                                                 | बाइनरी नाम या पूर्ण कमांड पथ                                              |
| `args`                                                    | नए रन के लिए आधार argv                                                          |
| `resumeArgs`                                              | फिर से शुरू किए गए सेशन के लिए वैकल्पिक argv; `{sessionId}` समर्थित है                       |
| `output` / `resumeOutput`                                 | पार्सर: `json`, `jsonl`, या `text`                                                |
| `jsonlDialect`                                            | JSONL इवेंट डायलेक्ट: `claude-stream-json` या `gemini-stream-json`                 |
| `liveSession`                                             | दीर्घकालिक CLI प्रोसेस मोड (`claude-stdio`)                                      |
| `input`                                                   | प्रॉम्प्ट ट्रांसपोर्ट: `arg` या `stdin`                                                |
| `maxPromptArgChars`                                       | stdin पर फ़ॉलबैक करने से पहले `arg` मोड के लिए अधिकतम प्रॉम्प्ट लंबाई                     |
| `env` / `clearEnv`                                        | इंजेक्ट करने के लिए अतिरिक्त एनवायरनमेंट वेरिएबल, या लॉन्च से पहले हटाए जाने वाले नाम                         |
| `modelArg`                                                | मॉडल आईडी से पहले उपयोग किया जाने वाला फ़्लैग                                                     |
| `modelAliases`                                            | OpenClaw मॉडल आईडी को CLI-नेटिव आईडी से मैप करना                                          |
| `sessionArgs`                                             | `{sessionId}` का उपयोग करके सेशन आईडी कैसे पास करें                                      |
| `sessionMode`                                             | `always`, `existing`, या `none`                                                   |
| `sessionIdFields`                                         | CLI आउटपुट से OpenClaw द्वारा पढ़े जाने वाले JSON फ़ील्ड                                        |
| `systemPromptArg` / `systemPromptFileArg`                 | सिस्टम प्रॉम्प्ट ट्रांसपोर्ट                                                           |
| `systemPromptFileConfigArg` / `systemPromptFileConfigKey` | सिस्टम प्रॉम्प्ट फ़ाइल के लिए कॉन्फ़िगरेशन-ओवरराइड ट्रांसपोर्ट (उदाहरण के लिए `-c`)             |
| `systemPromptMode`                                        | `append` या `replace`                                                             |
| `systemPromptWhen`                                        | `first`, `always`, या `never`                                                     |
| `imageArg` / `imageMode`                                  | इमेज पथ फ़्लैग और एक से अधिक इमेज पास करने का तरीका (`repeat` या `list`)              |
| `imagePathScope`                                          | हैंडऑफ़ से पहले स्टेज की गई इमेज फ़ाइलों का स्थान: `temp` या `workspace`               |
| `serialize`                                               | समान-बैकएंड रन को क्रमबद्ध रखना                                                    |
| `reseedFromRawTranscriptWhenUncompacted`                  | सुरक्षित सेशन रीसेट के लिए Compaction से पहले सीमित रॉ-ट्रांस्क्रिप्ट रीसीड का विकल्प चुनना |
| `reliability.watchdog`                                    | नो-आउटपुट टाइमआउट ट्यूनिंग, नए और फिर से शुरू किए गए रन के लिए अलग-अलग                      |

CLI से मेल खाने वाले सबसे छोटे स्थिर कॉन्फ़िगरेशन को प्राथमिकता दें। Plugin कॉलबैक केवल ऐसे व्यवहार के लिए जोड़ें, जिसका स्वामित्व वास्तव में बैकएंड के पास होना चाहिए।

## उन्नत बैकएंड हुक

`CliBackendPlugin` निम्नलिखित भी परिभाषित कर सकता है:

| हुक                               | उपयोग                                                                         |
| ---------------------------------- | --------------------------------------------------------------------------- |
| `normalizeConfig(config, context)` | रनटाइम संदर्भ के साथ पंजीकृत स्थिर अडैप्टर को सामान्यीकृत करना                |
| `resolveExecutionArgs(ctx)`        | थिंकिंग एफर्ट या साइड-क्वेश्चन आइसोलेशन जैसे अनुरोध-स्कोप वाले फ़्लैग जोड़ना |
| `prepareExecution(ctx)`            | लॉन्च से पहले अस्थायी प्रमाणीकरण, कॉन्फ़िगरेशन या एनवायरनमेंट ब्रिज बनाना         |
| `transformSystemPrompt(ctx)`       | अंतिम CLI-विशिष्ट सिस्टम प्रॉम्प्ट रूपांतरण लागू करना                          |
| `textTransforms`                   | द्विदिश प्रॉम्प्ट/आउटपुट प्रतिस्थापन                                    |
| `defaultAuthProfileId`             | किसी विशिष्ट OpenClaw प्रमाणीकरण प्रोफ़ाइल को प्राथमिकता देना                                     |
| `authEpochMode`                    | यह निर्धारित करना कि प्रमाणीकरण परिवर्तन संग्रहित CLI सेशन को कैसे अमान्य करते हैं                      |
| `nativeToolMode`                   | यह घोषित करना कि नेटिव टूल अनुपस्थित हैं, हमेशा चालू हैं या होस्ट द्वारा चुने जा सकते हैं      |
| `toolAvailabilityEnforcement`      | यह घोषित करना कि सटीक टूल सीमाएँ argv या निष्पादन स्टेजिंग में लागू होती हैं   |
| `sideQuestionToolMode`             | `/btw` साइड क्वेश्चन के लिए अक्षम नेटिव टूल घोषित करना                     |
| `bundleMcp` / `bundleMcpMode`      | OpenClaw के लूपबैक MCP टूल ब्रिज का विकल्प चुनना                                |
| `ownsNativeCompaction`             | बैकएंड अपने Compaction का स्वामी है—OpenClaw इसे स्थगित करता है                           |
| `subscriptionAuthDispatch`         | सदस्यता क्रेडेंशियल पर विकल्पित एम्बेडेड रन इस बैकएंड के माध्यम से निष्पादित होते हैं |
| `runtimeArtifact`                  | स्क्रिप्ट लॉन्चर को उसके पूर्ण बंडल किए गए पैकेज ट्री से बाँधना                |

इन हुक का स्वामित्व प्रोवाइडर के पास रखें। जब कोई बैकएंड हुक व्यवहार को व्यक्त कर सकता हो, तब कोर में CLI-विशिष्ट शाखाएँ न जोड़ें।

`prepareExecution(ctx)` को `ctx.contextTokenBudget` प्राप्त होता है, जो रन के लिए चुनी गई प्रभावी टोकन
सीमा है। मूल Compaction का स्वामित्व रखने वाले बैकएंड उस
बजट को अपने CLI-विशिष्ट लॉन्च अनुबंध में मैप कर सकते हैं।

`runtimeArtifact` का स्वामित्व Plugin के पास है। इससे केवल
तभी परामर्श किया जाता है जब कोई लाइव अनुमान टर्न सत्यापित सेटअप प्राधिकार जारी करता है या पुनः सत्यापित करता है;
सामान्य CLI रन के लिए इसकी आवश्यकता नहीं होती। इस घोषणा के बिना कोई बैकएंड
सत्यापित CLI सेटअप प्राधिकार जारी नहीं कर सकता। `bundled-package-tree` घोषणा
सटीक `package.json` स्वामी का नाम देती है और पैकेज एंट्रीपॉइंट का
कमांड होना आवश्यक बनाती है। OpenClaw नेस्टेड निर्भरताओं सहित
सीमित, पूर्ण इंस्टॉल किए गए पैकेज ट्री को हैश करता है, और पुनर्निर्देशित करने वाले सिमलिंक,
घोषित पैकेज से बाहर के लॉन्चर, आवश्यक बाहरी निर्भरता
घोषणाओं, अत्यधिक बड़े ट्री और अज्ञात स्क्रिप्ट के लिए विफलता को सुरक्षित रूप से रोक देता है। इसे केवल तभी घोषित करें जब उस
ट्री में पूर्ण अनुमान कार्यान्वयन हो; वैकल्पिक टूल एकीकरण
किसी बाहरी कार्यान्वयन ग्राफ़ को सुरक्षित नहीं बनाते।

यदि वही बैकएंड एक स्व-निहित मूल एक्ज़िक्यूटेबल भी उपलब्ध कराता है, तो उसके
कैनोनिकल बेसनेम `nativeExecutableNames` में सूचीबद्ध करें। अन्य मूल कमांड
असत्यापित रहते हैं।

सामान्य टर्न के लिए `ctx.executionMode`, `"agent"` है और
अल्पकालिक `/btw` कॉल के लिए `"side-question"` है। इसका उपयोग तब करें जब CLI को
अलग एकबारगी फ़्लैग की आवश्यकता हो, जैसे BTW के लिए मूल टूल, सत्र
स्थायित्व या पुनः आरंभ व्यवहार अक्षम करना। यदि किसी बैकएंड में सामान्यतः
`nativeToolMode: "always-on"` है, लेकिन उसका साइड-क्वेश्चन argv उन टूल को विश्वसनीय रूप से
अक्षम करता है, तो `sideQuestionToolMode: "disabled"` भी सेट करें; अन्यथा जब BTW को
बिना टूल वाला CLI रन चाहिए, तो OpenClaw विफलता को सुरक्षित रूप से रोक देता है।

`nativeToolMode: "selectable"` केवल तभी सेट करें जब बैकएंड किसी व्यक्तिगत
रन के लिए प्रत्येक बैकएंड-मूल टूल अक्षम कर सकता हो। प्रतिबंधित रन को एक कैनोनिकल
अनुबंध प्राप्त होता है: `ctx.toolAvailability.native` सटीक बैकएंड-मूल सूची है और
`ctx.toolAvailability.openClaw` OpenClaw टूल नामों की सटीक सूची है। होस्ट
स्वतंत्र रूप से जनरेट किए गए MCP कॉन्फ़िगरेशन और अनुदान को उस
OpenClaw सूची तक सीमित करता है; plugins को इसे कोर में रूपांतरित नहीं करना चाहिए या ट्रांसपोर्ट प्रीफ़िक्स नहीं जोड़ने चाहिए।

घोषित करें कि बैकएंड उस अनुबंध को कैसे लागू करता है:

- `toolAvailabilityEnforcement: "execution-args"` के लिए
  `resolveExecutionArgs` आवश्यक है। हुक को परस्पर विरोधी टूल फ़्लैग बदलने होंगे, ऐसे
  अनुकूलन माध्यमों को अक्षम करना होगा जो चयनित टूल के बाहर निष्पादन कर सकते हैं, और
  नए तथा पुनः आरंभ किए गए दोनों रन के लिए अनुपालन लागू करने वाला argv लौटाना होगा।
- `toolAvailabilityEnforcement: "prepare-execution"` के लिए
  `prepareExecution` आवश्यक है। हुक को सटीक प्रति-रन नीति तैयार करनी होगी और
  `toolAvailabilityEnforced: true` लौटाना होगा; अभिस्वीकृति अनुपस्थित होने पर विफलता सुरक्षित रूप से रोक दी जाती है और
  OpenClaw लॉन्च से पहले तैयार किए गए संसाधनों को साफ़ कर देता है।

cron `toolsAllow` जैसी रनटाइम सीमाओं को यह अनुबंध बनाए जाने से पहले
OpenClaw सामान्यीकृत और समूह-विस्तारित करता है। मूल टूल अक्षम कर दिए जाते हैं, और
पूर्ण घोषित प्रवर्तन पथ के बिना बैकएंड निष्पादन से पहले विफल हो जाता है।

`v2026.7.2-beta.1` से `v2026.7.2-beta.3` तक के विरुद्ध बनाए गए plugins अब भी
बहिष्कृत `ctx.toolAvailability.mcp` ट्रांसपोर्ट-नाम प्रक्षेपण पढ़ सकते हैं और
जब कोई चयन योग्य बैकएंड `resolveExecutionArgs` लागू करता है, तब
`toolAvailabilityEnforcement` को छोड़ सकते हैं। OpenClaw, plugin पैकेज के आवश्यक
`openclaw.build.openclawVersion` मेटाडेटा से उस जारी किए गए बीटा पथ को पहचानता है और
उसे `2026.8.x` लाइन तक बनाए रखता है। नए और अपडेट किए गए plugins को कैनोनिकल
`ctx.toolAvailability.openClaw` नामों का उपयोग करना चाहिए और
`toolAvailabilityEnforcement: "execution-args"` स्पष्ट रूप से घोषित करना चाहिए; बीटा
संगतता पथ को उस अवधि के बाद हटाने की योजना है।

### `ownsNativeCompaction`: OpenClaw Compaction से बाहर निकलना

यदि आपका बैकएंड ऐसा एजेंट चलाता है जो अपनी **स्वयं की** ट्रांसक्रिप्ट को संकुचित करता है, तो
`ownsNativeCompaction: true` सेट करें, ताकि OpenClaw का सुरक्षा-सारांशकर्ता उसके सत्रों पर
कभी न चले—CLI Compaction जीवनचक्र कोई कार्रवाई न करते हुए लौटता है और
टर्न आगे बढ़ता है। `claude-cli` इसे घोषित करता है क्योंकि Claude Code
बिना किसी हार्नेस एंडपॉइंट के आंतरिक रूप से संकुचित करता है। Codex जैसे मूल-हार्नेस सत्र
इसके बजाय अपने हार्नेस Compaction एंडपॉइंट पर रूट होते रहते हैं।

**इसे केवल तभी घोषित करें जब निम्नलिखित सभी शर्तें पूरी हों**, अन्यथा कोई स्थगित
बजट-पार सत्र बजट से ऊपर बना रह सकता है या अप्रचलित हो सकता है (OpenClaw अब
उसे नहीं बचाता):

- बैकएंड अपनी विंडो के निकट पहुँचते समय अपनी ट्रांसक्रिप्ट को विश्वसनीय रूप से संकुचित या
  सीमित करता है;
- वह पुनः आरंभ योग्य सत्र को स्थायी रखता है, ताकि संकुचित स्थिति टर्न के बीच बनी रहे
  (उदाहरण के लिए `--resume` / `--session-id`);
- वह मूल-हार्नेस Compaction सत्र नहीं है—मेल खाने वाले `agentHarnessId`
  सत्र इसके बजाय हार्नेस एंडपॉइंट पर रूट होते हैं।

## MCP टूल ब्रिज

CLI बैकएंड को डिफ़ॉल्ट रूप से OpenClaw टूल प्राप्त नहीं होते। यदि CLI किसी
MCP कॉन्फ़िगरेशन का उपयोग कर सकता है, तो स्पष्ट रूप से विकल्प चुनें:

```typescript
return {
  id: "acme-cli",
  bundleMcp: true,
  bundleMcpMode: "codex-config-overrides",
  config: {
    command: "acme",
    args: ["chat", "--json"],
    output: "json",
  },
};
```

समर्थित ब्रिज मोड:

| मोड                     | उपयोग                                                              |
| ------------------------ | ---------------------------------------------------------------- |
| `claude-config-file`     | वे CLI जो MCP कॉन्फ़िगरेशन फ़ाइल स्वीकार करते हैं                              |
| `codex-config-overrides` | वे CLI जो argv पर कॉन्फ़िगरेशन ओवरराइड स्वीकार करते हैं                        |
| `gemini-system-settings` | वे CLI जो अपनी सिस्टम सेटिंग्स डायरेक्टरी से MCP सेटिंग्स पढ़ते हैं |

ब्रिज को केवल तभी सक्षम करें जब CLI वास्तव में उसका उपयोग कर सकता हो। यदि CLI की
अपनी अंतर्निहित टूल परत है जिसे अक्षम नहीं किया जा सकता, तो `nativeToolMode:
"always-on"` सेट करें, ताकि जब किसी कॉलर को कोई मूल
टूल नहीं चाहिए, तब OpenClaw विफलता को सुरक्षित रूप से रोक सके। यदि वह प्रति रन प्रत्येक मूल टूल अक्षम कर सकता है, तो ऊपर दिए गए
`resolveExecutionArgs` अनुबंध के साथ `"selectable"` का उपयोग करें।

## बैकएंड चुनना

उपयोगकर्ता किसी स्वतंत्र बैकएंड को उसके मॉडल-रेफ़ प्रीफ़िक्स के माध्यम से चुनते हैं। कैनोनिकल
`modelProvider` घोषित करने वाला बैकएंड इसके बजाय उस
प्रदाता मॉडल के `agentRuntime.id` के माध्यम से चुना जा सकता है। अडैप्टर की कार्यप्रणाली Plugin में रहती है:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "openai/gpt-5.6-sol",
        fallbacks: ["acme-cli/large"],
      },
    },
  },
}
```

क्रेडेंशियल OpenClaw प्रमाणीकरण प्रोफ़ाइल या Plugin-स्वामित्व वाले कॉन्फ़िगरेशन में रखें। सुनिश्चित करें कि
पंजीकृत कमांड Gateway सेवा के `PATH` में है; जिन परिनियोजनों को
अलग पथ या argv चाहिए, उन्हें Plugin पंजीकरण बदलना या रैप करना चाहिए।

## सत्यापन

बंडल किए गए plugins के लिए, बिल्डर और सेटअप
पंजीकरण के आसपास एक केंद्रित परीक्षण जोड़ें, फिर Plugin की लक्षित परीक्षण लेन चलाएँ:

```bash
pnpm test extensions/acme-cli
```

स्थानीय या इंस्टॉल किए गए plugins के लिए, खोज और एक वास्तविक मॉडल रन सत्यापित करें:

```bash
openclaw plugins inspect acme-cli --runtime --json
openclaw agent --message "ठीक इसी तरह उत्तर दें: बैकएंड ठीक है" --model acme-cli/acme-large
```

यदि बैकएंड चित्रों या MCP का समर्थन करता है, तो ऐसा लाइव स्मोक परीक्षण जोड़ें जो वास्तविक CLI के साथ उन
पथों को प्रमाणित करे। प्रॉम्प्ट, चित्र,
MCP या सत्र-पुनः आरंभ व्यवहार के लिए स्थिर निरीक्षण पर निर्भर न रहें।

## जाँच-सूची

<Check>`package.json` में प्रकाशित पैकेजों के लिए `openclaw.extensions` और निर्मित रनटाइम प्रविष्टियाँ हैं</Check>
<Check>`openclaw.plugin.json`, `cliBackends` और अभिप्रेत `activation.onStartup` घोषित करता है</Check>
<Check>जब सेटअप/मॉडल खोज को बैकएंड को कोल्ड अवस्था में देखना चाहिए, तब `setup.cliBackends` मौजूद है</Check>
<Check>`api.registerCliBackend(...)`, मेनिफ़ेस्ट के समान बैकएंड आईडी का उपयोग करता है</Check>
<Check>बैकएंड मॉडल प्रीफ़िक्स या मॉडल-स्कोप वाला `agentRuntime.id` पंजीकरण चुनता है</Check>
<Check>सत्र, सिस्टम प्रॉम्प्ट, चित्र और आउटपुट पार्सर सेटिंग्स वास्तविक CLI अनुबंध से मेल खाती हैं</Check>
<Check>लक्षित परीक्षण और कम-से-कम एक लाइव CLI स्मोक परीक्षण बैकएंड पथ को प्रमाणित करते हैं</Check>

## संबंधित

- [CLI बैकएंड](/hi/gateway/cli-backends) - रनटाइम चयन और व्यवहार
- [plugins बनाना](/hi/plugins/building-plugins) - पैकेज और मेनिफ़ेस्ट की मूल बातें
- [Plugin SDK का अवलोकन](/hi/plugins/sdk-overview) - पंजीकरण API संदर्भ
- [Plugin मेनिफ़ेस्ट](/hi/plugins/manifest) - `cliBackends` और सेटअप वर्णनकर्ता
- [एजेंट हार्नेस](/hi/plugins/sdk-agent-harness) - पूर्ण बाहरी एजेंट रनटाइम
