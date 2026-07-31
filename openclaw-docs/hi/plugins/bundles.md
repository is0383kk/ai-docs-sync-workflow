---
read_when:
    - आप Codex, Claude या Cursor-संगत बंडल इंस्टॉल करना चाहते हैं
    - आपको यह समझना होगा कि OpenClaw बंडल की सामग्री को नेटिव सुविधाओं से कैसे मैप करता है
    - आप बंडल पहचान या अनुपलब्ध क्षमताओं को डीबग कर रहे हैं
summary: Codex, Claude और Cursor बंडलों को OpenClaw plugins के रूप में इंस्टॉल और उपयोग करें
title: Plugin बंडल
x-i18n:
    generated_at: "2026-07-27T19:34:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d44006866238f53ee2e3e8126cc4f7ed6f7413534257775f7904c9b877778c59
    source_path: plugins/bundles.md
    workflow: 16
---

OpenClaw तीन बाहरी इकोसिस्टम से plugins इंस्टॉल कर सकता है: **Codex**, **Claude**,
और **Cursor**। इन्हें **बंडल** कहा जाता है—ऐसे कंटेंट और मेटाडेटा पैक जिन्हें
OpenClaw, skills, hooks और MCP tools जैसी मूल सुविधाओं में मैप करता है।

<Info>
  बंडल मूल OpenClaw plugins के समान **नहीं** हैं। मूल plugins
  इन-प्रोसेस चलते हैं और किसी भी क्षमता को पंजीकृत कर सकते हैं। बंडल चुनिंदा
  सुविधा मैपिंग और अधिक सीमित विश्वास-सीमा वाले कंटेंट पैक हैं।
</Info>

## बंडल क्यों मौजूद हैं

कई उपयोगी plugins Codex, Claude या Cursor प्रारूप में प्रकाशित किए जाते हैं। लेखकों से
उन्हें मूल OpenClaw plugins के रूप में दोबारा लिखने की अपेक्षा करने के बजाय, OpenClaw
इन प्रारूपों का पता लगाता है और उनके समर्थित कंटेंट को मूल सुविधा-समूह में मैप करता है।
आप Claude कमांड पैक या Codex skill बंडल इंस्टॉल करके तुरंत उसका उपयोग कर सकते हैं।

## बंडल इंस्टॉल करें

<Steps>
  <Step title="डायरेक्टरी, आर्काइव या मार्केटप्लेस से इंस्टॉल करें">
    ```bash
    # स्थानीय डायरेक्टरी
    openclaw plugins install ./my-bundle

    # आर्काइव
    openclaw plugins install ./my-bundle.tgz

    # Claude मार्केटप्लेस
    openclaw plugins marketplace list <source>
    openclaw plugins install <plugin> --marketplace <source>
    ```

    `<source>` एक स्थानीय मार्केटप्लेस पथ/रिपॉज़िटरी या git/GitHub स्रोत है।

  </Step>

  <Step title="पहचान सत्यापित करें">
    ```bash
    openclaw plugins list
    openclaw plugins inspect <id>
    ```

    बंडल में `Format: bundle` के साथ `Bundle format:` का मान `codex`,
    `claude` या `cursor` दिखाई देता है।

  </Step>

  <Step title="पुनः आरंभ करें और उपयोग करें">
    ```bash
    openclaw gateway restart
    ```

    मैप की गई सुविधाएँ (skills, hooks, MCP tools, LSP डिफ़ॉल्ट) अगले सत्र में उपलब्ध होती हैं।

  </Step>
</Steps>

## OpenClaw बंडल से क्या मैप करता है

आज प्रत्येक बंडल सुविधा OpenClaw में नहीं चलती। यहाँ बताया गया है कि क्या काम करता है और
किसका पता तो लगाया जाता है, लेकिन अभी जोड़ा नहीं गया है।

### अभी समर्थित

| सुविधा        | यह कैसे मैप होती है                                                                                | इन पर लागू     |
| ------------- | ------------------------------------------------------------------------------------------------- | -------------- |
| Skill कंटेंट  | बंडल skill रूट सामान्य OpenClaw skills के रूप में लोड होते हैं                                    | सभी प्रारूप    |
| कमांड         | `commands/` और `.cursor/commands/` को skill रूट माना जाता है                                      | Claude, Cursor |
| Hook पैक      | OpenClaw-शैली के `HOOK.md` + `handler.ts` लेआउट                                           | Codex          |
| MCP tools     | बंडल MCP कॉन्फ़िग को एम्बेडेड OpenClaw सेटिंग में मर्ज किया जाता है; समर्थित stdio और HTTP सर्वर लोड होते हैं | सभी प्रारूप    |
| LSP सर्वर     | Claude `.lsp.json` और मैनिफ़ेस्ट में घोषित `lspServers` को एम्बेडेड OpenClaw LSP डिफ़ॉल्ट में मर्ज किया जाता है | Claude         |
| सेटिंग        | Claude `settings.json` को एम्बेडेड OpenClaw डिफ़ॉल्ट के रूप में इंपोर्ट किया जाता है                  | Claude         |

#### Skill कंटेंट

- बंडल skill रूट सामान्य OpenClaw skill रूट के रूप में लोड होते हैं।
- Claude `commands/` रूट को अतिरिक्त skill रूट माना जाता है।
- Cursor `.cursor/commands/` रूट को अतिरिक्त skill रूट माना जाता है।

Claude की Markdown कमांड फ़ाइलें और Cursor का कमांड Markdown, दोनों सामान्य
OpenClaw skill लोडर के माध्यम से काम करते हैं।

#### Hook पैक

बंडल hook रूट **केवल** तभी काम करते हैं, जब वे सामान्य OpenClaw hook-पैक
लेआउट का उपयोग करते हैं: `HOOK.md` के साथ `handler.ts` या `handler.js`। आज यह मुख्यतः
Codex-संगत स्थिति में लागू होता है।

#### एम्बेडेड OpenClaw के लिए MCP

- सक्षम बंडल MCP सर्वर कॉन्फ़िग प्रदान कर सकते हैं।
- OpenClaw बंडल MCP कॉन्फ़िग को प्रभावी एम्बेडेड OpenClaw
  सेटिंग में `mcpServers` के रूप में मर्ज करता है।
- OpenClaw, stdio सर्वर शुरू करके या HTTP सर्वर से कनेक्ट होकर, एम्बेडेड OpenClaw agent
  टर्न के दौरान समर्थित बंडल MCP tools उपलब्ध कराता है।
- `coding` और `messaging` टूल प्रोफ़ाइल में डिफ़ॉल्ट रूप से बंडल MCP tools शामिल होते हैं;
  किसी agent या gateway के लिए उनसे बाहर रहने हेतु `tools.deny: ["bundle-mcp"]` का उपयोग करें।
- प्रोजेक्ट-स्थानीय एम्बेडेड agent सेटिंग बंडल डिफ़ॉल्ट के बाद भी लागू होती हैं, इसलिए
  आवश्यकता होने पर कार्यक्षेत्र सेटिंग बंडल MCP प्रविष्टियों को ओवरराइड कर सकती हैं।
- बंडल MCP टूल कैटलॉग को पंजीकरण से पहले नियतात्मक रूप से क्रमबद्ध किया जाता है, ताकि
  अपस्ट्रीम `listTools()` क्रम में बदलाव prompt-cache टूल ब्लॉक में अनावश्यक उथल-पुथल न करें।

##### ट्रांसपोर्ट

MCP सर्वर stdio या HTTP ट्रांसपोर्ट का उपयोग कर सकते हैं।

**Stdio** एक चाइल्ड प्रोसेस शुरू करता है:

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "command": "node",
        "args": ["server.js"],
        "env": { "PORT": "3000" }
      }
    }
  }
}
```

**HTTP** चल रहे MCP सर्वर से कनेक्ट होता है और `streamable-http` का अनुरोध न किए जाने पर
डिफ़ॉल्ट रूप से `sse` का उपयोग करता है:

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "url": "http://localhost:3100/mcp",
        "transport": "streamable-http",
        "headers": {
          "Authorization": "Bearer ${MY_SECRET_TOKEN}"
        },
        "connectionTimeoutMs": 30000
      }
    }
  }
}
```

- `transport` में `"streamable-http"` या `"sse"` स्वीकार किया जाता है; छोड़े जाने पर डिफ़ॉल्ट `sse` होता है।
- `type: "http"` एक CLI-मूल डाउनस्ट्रीम आकार है; OpenClaw कॉन्फ़िग में `transport: "streamable-http"` का उपयोग करें। `openclaw mcp set` और `openclaw doctor --fix` सामान्य उपनाम को सामान्यीकृत करते हैं।
- केवल `http:` और `https:` URL स्कीम की अनुमति है।
- `headers` मान `${ENV_VAR}` इंटरपोलेशन का समर्थन करते हैं।
- `command` और `url` दोनों वाली सर्वर प्रविष्टि अस्वीकार कर दी जाती है।
- URL क्रेडेंशियल (यूज़र-इन्फ़ो और क्वेरी पैरामीटर) को टूल
  विवरण और लॉग से छिपा दिया जाता है।
- `connectionTimeoutMs`, stdio और HTTP दोनों ट्रांसपोर्ट के लिए डिफ़ॉल्ट 30-सेकंड कनेक्शन टाइमआउट को
  ओवरराइड करता है। अनुरोध टाइमआउट डिफ़ॉल्ट रूप से 60 सेकंड है और
  `requestTimeoutMs` से ओवरराइड किया जा सकता है।

##### टूल नामकरण

OpenClaw बंडल MCP tools को प्रदाता-सुरक्षित नामों के साथ
`serverName__toolName` प्रारूप में पंजीकृत करता है। उदाहरण के लिए, `"vigil-harbor"` कुंजी वाला सर्वर
जो `memory_search` टूल उपलब्ध कराता है, वह `vigil-harbor__memory_search` के रूप में पंजीकृत होता है।

- `A-Za-z0-9_-` के बाहर के वर्णों को `-` से बदल दिया जाता है।
- जो खंड किसी अक्षर के अलावा किसी अन्य वर्ण से शुरू होते, उनमें अक्षर उपसर्ग जोड़ा जाता है, ताकि
  `12306` जैसी संख्यात्मक सर्वर कुंजियाँ प्रदाता-सुरक्षित टूल उपसर्ग बनें।
- सर्वर उपसर्ग अधिकतम 30 वर्ण के होते हैं।
- पूरे टूल नाम अधिकतम 64 वर्ण के होते हैं।
- खाली सर्वर नामों के लिए `mcp` का उपयोग किया जाता है।
- टकराने वाले स्वच्छीकृत नामों को संख्यात्मक प्रत्ययों से अलग किया जाता है।
- अंततः उपलब्ध टूल का क्रम सुरक्षित नाम के अनुसार नियतात्मक होता है, जिससे बार-बार होने वाले
  एम्बेडेड-agent टर्न में कैश स्थिर रहता है।
- प्रोफ़ाइल फ़िल्टरिंग एक बंडल MCP सर्वर के प्रत्येक टूल को
  `bundle-mcp` के स्वामित्व वाला plugin मानती है, इसलिए प्रोफ़ाइल अनुमति/निषेध सूचियाँ
  अलग-अलग उपलब्ध टूल नामों या `bundle-mcp` plugin कुंजी को संदर्भित कर सकती हैं।

#### एम्बेडेड OpenClaw सेटिंग

बंडल सक्षम होने पर Claude `settings.json` को डिफ़ॉल्ट एम्बेडेड OpenClaw सेटिंग के रूप में
इंपोर्ट किया जाता है। OpenClaw शेल ओवरराइड कुंजियों को लागू करने से पहले स्वच्छीकृत करता है:

- `shellPath`
- `shellCommandPrefix`

#### एम्बेडेड OpenClaw LSP

- सक्षम Claude बंडल LSP सर्वर कॉन्फ़िग प्रदान कर सकते हैं।
- OpenClaw `.lsp.json` के साथ मैनिफ़ेस्ट में घोषित सभी `lspServers` पथ लोड करता है।
- बंडल LSP कॉन्फ़िग को प्रभावी एम्बेडेड OpenClaw LSP
  डिफ़ॉल्ट में मर्ज किया जाता है।
- आज केवल समर्थित stdio-समर्थित LSP सर्वर चलाए जा सकते हैं; असमर्थित
  ट्रांसपोर्ट फिर भी `openclaw plugins inspect <id>` में दिखाई देते हैं।

### पहचाने गए, लेकिन निष्पादित नहीं

इन्हें पहचाना जाता है और डायग्नोस्टिक्स में दिखाया जाता है, लेकिन OpenClaw इन्हें नहीं चलाता:

- Claude `agents`, `hooks/hooks.json` ऑटोमेशन, `outputStyles`
- Cursor `.cursor/agents`, `.cursor/hooks.json`, `.cursor/rules`
- क्षमता रिपोर्टिंग से परे Codex `.app.json` मेटाडेटा

## बंडल प्रारूप

<AccordionGroup>
  <Accordion title="Codex बंडल">
    मार्कर: `.codex-plugin/plugin.json`

    वैकल्पिक कंटेंट: `skills/`, `hooks/`, `.mcp.json`, `.app.json`

    जब Codex बंडल skill रूट और OpenClaw-शैली की
    hook-पैक डायरेक्टरी (`HOOK.md` + `handler.ts`) का उपयोग करते हैं, तब वे OpenClaw में सबसे उपयुक्त होते हैं।

  </Accordion>

  <Accordion title="Claude बंडल">
    पहचान के दो तरीके:

    - **मैनिफ़ेस्ट-आधारित:** `.claude-plugin/plugin.json`
    - **मैनिफ़ेस्ट-रहित:** डिफ़ॉल्ट Claude लेआउट (`skills/`, `commands/`, `agents/`, `hooks/`, `.mcp.json`, `.lsp.json`, `settings.json`)

    Claude-विशिष्ट व्यवहार:

    - `commands/` को skill कंटेंट माना जाता है
    - `settings.json` को एम्बेडेड OpenClaw सेटिंग में इंपोर्ट किया जाता है (शेल ओवरराइड कुंजियाँ स्वच्छीकृत की जाती हैं)
    - `.mcp.json` एम्बेडेड OpenClaw को समर्थित stdio tools उपलब्ध कराता है
    - `.lsp.json` के साथ मैनिफ़ेस्ट में घोषित `lspServers` पथ एम्बेडेड OpenClaw LSP डिफ़ॉल्ट में लोड होते हैं
    - `hooks/hooks.json` का पता लगाया जाता है, लेकिन उसे निष्पादित नहीं किया जाता
    - मैनिफ़ेस्ट में कस्टम घटक पथ योगात्मक होते हैं; वे डिफ़ॉल्ट को प्रतिस्थापित नहीं, बल्कि विस्तारित करते हैं

  </Accordion>

  <Accordion title="Cursor बंडल">
    मार्कर: `.cursor-plugin/plugin.json`

    वैकल्पिक कंटेंट: `skills/`, `.cursor/commands/`, `.cursor/agents/`, `.cursor/rules/`, `.cursor/hooks.json`, `.mcp.json`

    - `.cursor/commands/` को skill कंटेंट माना जाता है
    - `.cursor/rules/`, `.cursor/agents/` और `.cursor/hooks.json` केवल पहचाने जाते हैं

  </Accordion>
</AccordionGroup>

## पहचान की प्राथमिकता

OpenClaw पहले मूल plugin प्रारूप की जाँच करता है:

1. `openclaw.plugin.json` या `openclaw.extensions` वाला मान्य `package.json`—इसे **मूल plugin** माना जाता है
2. बंडल मार्कर (`.codex-plugin/`, `.claude-plugin/` या डिफ़ॉल्ट Claude/Cursor लेआउट)—इसे **बंडल** माना जाता है

यदि किसी डायरेक्टरी में दोनों मौजूद हों, तो OpenClaw मूल पथ का उपयोग करता है। इससे
दोहरे-प्रारूप वाले पैकेज बंडल के रूप में आंशिक रूप से इंस्टॉल नहीं होते।

## रनटाइम निर्भरताएँ और सफ़ाई

- तृतीय-पक्ष संगत बंडल को स्टार्टअप `npm install` सुधार नहीं मिलता। उन्हें
  `openclaw plugins install` के माध्यम से इंस्टॉल किया जाना चाहिए और आवश्यक सभी सामग्री
  इंस्टॉल की गई plugin डायरेक्टरी में ही भेजनी चाहिए।
- OpenClaw के स्वामित्व वाले बंडल plugins या तो core में हल्के रूप में भेजे जाते हैं या
  plugin इंस्टॉलर के माध्यम से डाउनलोड किए जा सकते हैं। Gateway स्टार्टअप उनके लिए कभी
  पैकेज मैनेजर नहीं चलाता।
- `openclaw doctor --fix` पुराने स्थानीय बंडल-plugin इंस्टॉलेशन रिकॉर्ड हटाता है
  और यदि कॉन्फ़िग अब भी उनका संदर्भ देता हो, तो स्थानीय plugin इंडेक्स से गायब डाउनलोड-योग्य plugins
  को पुनर्प्राप्त कर सकता है।

## सुरक्षा

बंडल की विश्वास-सीमा मूल plugins से अधिक सीमित होती है:

- OpenClaw मनमाने बंडल रनटाइम मॉड्यूल को इन-प्रोसेस लोड **नहीं** करता।
- Skills और hook-पैक पथ plugin रूट के भीतर ही होने चाहिए (सीमा की जाँच की जाती है)।
- सेटिंग फ़ाइलें समान सीमा जाँच के साथ पढ़ी जाती हैं।
- समर्थित stdio MCP सर्वर उप-प्रक्रियाओं के रूप में शुरू किए जा सकते हैं।

इससे बंडल डिफ़ॉल्ट रूप से अधिक सुरक्षित होते हैं, लेकिन फिर भी तृतीय-पक्ष
बंडल जिन सुविधाओं को उपलब्ध कराते हैं, उनके लिए उन्हें विश्वसनीय कंटेंट ही मानना चाहिए।

## समस्या निवारण

<AccordionGroup>
  <Accordion title="बंडल का पता चलता है, लेकिन क्षमताएँ नहीं चलतीं">
    `openclaw plugins inspect <id>` चलाएँ। यदि कोई क्षमता सूचीबद्ध है, लेकिन उसे
    कनेक्ट नहीं किया गया बताया गया है, तो यह उत्पाद की सीमा है, खराब इंस्टॉलेशन नहीं।
  </Accordion>

  <Accordion title="Claude कमांड फ़ाइलें दिखाई नहीं देतीं">
    सुनिश्चित करें कि बंडल सक्षम है और मार्कडाउन फ़ाइलें पहचाने गए
    `commands/` या `skills/` रूट के अंदर हैं।
  </Accordion>

  <Accordion title="Claude सेटिंग्स लागू नहीं होतीं">
    केवल `settings.json` की एम्बेड की गई OpenClaw सेटिंग्स समर्थित हैं। OpenClaw
    बंडल सेटिंग्स को अपरिष्कृत कॉन्फ़िगरेशन पैच के रूप में नहीं मानता।
  </Accordion>

  <Accordion title="Claude हुक निष्पादित नहीं होते">
    `hooks/hooks.json` केवल पहचान के लिए है। यदि आपको चलने योग्य हुक चाहिए, तो
    OpenClaw हुक-पैक लेआउट का उपयोग करें या कोई नेटिव Plugin वितरित करें।
  </Accordion>
</AccordionGroup>

## संबंधित

- [Plugin इंस्टॉल और कॉन्फ़िगर करें](/hi/tools/plugin)
- [Plugin बनाना](/hi/plugins/building-plugins) - एक नेटिव Plugin बनाएँ
- [Plugin मेनिफ़ेस्ट](/hi/plugins/manifest) - नेटिव मेनिफ़ेस्ट स्कीमा
