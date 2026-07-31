---
read_when:
    - आप एक सरल OpenClaw Plugin बनाना चाहते हैं जो केवल एजेंट टूल जोड़ता है
    - आप Plugin मेनिफ़ेस्ट मेटाडेटा को हाथ से लिखने के बजाय `defineToolPlugin` का उपयोग करना चाहते हैं
    - आपको केवल टूल वाला Plugin स्कैफ़ोल्ड, जेनरेट, वैलिडेट, टेस्ट या प्रकाशित करना है
sidebarTitle: Tool Plugins
summary: defineToolPlugin और openclaw plugins init/build/validate के साथ सरल टाइप किए गए एजेंट टूल बनाएँ
title: टूल Plugin
x-i18n:
    generated_at: "2026-07-27T18:20:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac23d15ba79cbdd1d8b8eab7c87007b44af16361b2866b14123e18f816bf4075
    source_path: plugins/tool-plugins.md
    workflow: 16
---

`defineToolPlugin` एक ऐसा Plugin बनाता है जो केवल एजेंट द्वारा कॉल किए जा सकने वाले टूल जोड़ता है: कोई
चैनल, मॉडल प्रदाता, हुक, सेवा या सेटअप बैकएंड नहीं। यह वह
मैनिफ़ेस्ट मेटाडेटा जनरेट करता है जिसकी OpenClaw को Plugin
रनटाइम कोड लोड किए बिना टूल खोजने के लिए आवश्यकता होती है।

प्रदाता, चैनल, हुक, सेवा या मिश्रित-क्षमता वाले Plugins के लिए, इसके बजाय
[Plugins बनाना](/hi/plugins/building-plugins), [चैनल Plugins](/hi/plugins/sdk-channel-plugins),
या [प्रदाता Plugins](/hi/plugins/sdk-provider-plugins) से शुरू करें।

## आवश्यकताएँ

- Node 22.22.3+, Node 24.15+, या Node 25.9+।
- TypeScript ESM पैकेज आउटपुट।
- `typebox` को `dependencies` में रखें (केवल `devDependencies` में नहीं—जनरेट किया गया
  Plugin इसे रनटाइम पर इंपोर्ट करता है)।
- `openclaw >=2026.5.17`, पहला संस्करण जो
  `openclaw/plugin-sdk/tool-plugin` एक्सपोर्ट करता है।
- एक पैकेज रूट जो `dist/`, `openclaw.plugin.json`, और
  `package.json` वितरित करता है।

## त्वरित शुरुआत

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm install
npm run plugin:build
npm run plugin:validate
npm test
```

`plugins init` निम्नलिखित स्कैफ़ोल्ड करता है:

| फ़ाइल                   | उद्देश्य                                                           |
| ---------------------- | ----------------------------------------------------------------- |
| `src/index.ts`         | एक `echo` टूल वाली `defineToolPlugin` एंट्री                     |
| `src/index.test.ts`    | टूल सूची को सत्यापित करने वाला मेटाडेटा परीक्षण                             |
| `tsconfig.json`        | `dist/` में NodeNext TypeScript आउटपुट                             |
| `vitest.config.ts`     | `src/**/*.test.ts` के लिए Vitest कॉन्फ़िगरेशन                              |
| `package.json`         | स्क्रिप्ट, रनटाइम निर्भरताएँ, `openclaw.extensions: ["./dist/index.js"]` |
| `openclaw.plugin.json` | प्रारंभिक टूल के लिए जनरेट किया गया मैनिफ़ेस्ट मेटाडेटा                  |

`npm run plugin:build`, `npm run build` (tsc) और फिर
`openclaw plugins build --entry ./dist/index.js` चलाता है। `npm run plugin:validate`
दोबारा बिल्ड करके `openclaw plugins validate --entry ./dist/index.js` चलाता है।
सफल सत्यापन यह प्रिंट करता है:

```text
Plugin stock-quotes मान्य है।
```

`openclaw plugins init <id>` विकल्प:

| फ़्लैग                 | डिफ़ॉल्ट            | प्रभाव                                 |
| -------------------- | ------------------ | -------------------------------------- |
| `--directory <path>` | `<id>`             | आउटपुट डायरेक्टरी                       |
| `--name <name>`      | शीर्षक-शैली वाला `<id>` | प्रदर्शन नाम                           |
| `--type <type>`      | `tool`             | स्कैफ़ोल्ड प्रकार: `tool` या `provider`    |
| `--force`            | बंद                | मौजूदा आउटपुट डायरेक्टरी को अधिलेखित करें |

## टूल लिखें

`defineToolPlugin` Plugin की पहचान, एक वैकल्पिक कॉन्फ़िगरेशन स्कीमा और
टूल की स्थिर सूची लेता है। पैरामीटर और कॉन्फ़िगरेशन प्रकार
TypeBox स्कीमा से अनुमानित किए जाते हैं।

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "स्टॉक कोट के स्नैपशॉट प्राप्त करें।",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "कोट API कुंजी।" })),
    baseUrl: Type.Optional(Type.String({ description: "कोट API का आधार URL।" })),
  }),
  tools: (tool) => [
    tool({
      name: "stock_quote",
      label: "स्टॉक कोट",
      description: "स्टॉक कोट का स्नैपशॉट प्राप्त करें।",
      parameters: Type.Object({
        symbol: Type.String({ description: "टिकर प्रतीक, उदाहरण के लिए OPEN।" }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          configured: Type.Boolean(),
          baseUrl: Type.String(),
        },
        { additionalProperties: false },
      ),
      async execute({ symbol }, config, context) {
        context.signal?.throwIfAborted();
        return {
          symbol: symbol.toUpperCase(),
          configured: Boolean(config.apiKey),
          baseUrl: config.baseUrl ?? "https://api.example.com",
        };
      },
    }),
  ],
});
```

टूल नाम स्थिर API हैं। ऐसे नाम चुनें जो अद्वितीय, लोअरकेस और
कोर टूल या अन्य Plugins के साथ टकराव से बचने के लिए पर्याप्त रूप से विशिष्ट हों।

## वैकल्पिक और फ़ैक्टरी टूल

जब उपयोगकर्ताओं को मॉडल पर भेजे जाने से पहले टूल को स्पष्ट रूप से अनुमत-सूची में जोड़ना चाहिए, तब `optional: true` सेट करें।
`openclaw plugins build` संबंधित
`toolMetadata.<tool>.optional` मैनिफ़ेस्ट एंट्री लिखता है, ताकि OpenClaw
Plugin रनटाइम कोड लोड किए बिना देख सके कि टूल वैकल्पिक है।

```typescript
tool({
  name: "workflow_run",
  description: "बाहरी वर्कफ़्लो चलाएँ।",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  execute: ({ goal }) => ({ queued: true, goal }),
});
```

जब किसी टूल को बनाए जाने से पहले रनटाइम टूल संदर्भ की आवश्यकता हो—किसी विशिष्ट रन के लिए बाहर रहने, सैंडबॉक्स स्थिति जाँचने या
रनटाइम सहायक बाँधने के लिए—तब `factory` का उपयोग करें।
हालाँकि ठोस टूल रनटाइम पर बनता है, मेटाडेटा स्थिर रहता है।

```typescript
tool({
  name: "local_workflow",
  description: "सैंडबॉक्स किए गए सत्रों के बाहर स्थानीय वर्कफ़्लो चलाएँ।",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  factory({ api, toolContext }) {
    if (toolContext.sandboxed) {
      return null;
    }
    return createLocalWorkflowTool(api);
  },
});
```

फ़ैक्टरियाँ फिर भी पहले से एक निश्चित टूल नाम घोषित करती हैं। जब Plugin टूल नामों की
गतिशील रूप से गणना करता है या टूल को हुक, सेवाओं, प्रदाताओं या कमांड के साथ
जोड़ता है, तब सीधे `definePluginEntry` का उपयोग करें।

## रिटर्न मान

`defineToolPlugin` सामान्य रिटर्न मानों को OpenClaw टूल-परिणाम
प्रारूप में लपेटता है:

- जब मॉडल को वही सटीक टेक्स्ट दिखना चाहिए, तब स्ट्रिंग लौटाएँ।
- जब आप चाहते हैं कि मॉडल स्वरूपित JSON देखे
  और OpenClaw मूल मान को `details` में रखे, तब JSON-संगत मान लौटाएँ।

```typescript
tool({
  name: "echo_text",
  description: "इनपुट टेक्स्ट को दोहराएँ।",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => input,
});
```

```typescript
tool({
  name: "echo_json",
  description: "इनपुट को संरचित JSON के रूप में दोहराएँ।",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => ({ input, length: input.length }),
});
```

जब आपको कस्टम `AgentToolResult` की आवश्यकता हो या आप किसी मौजूदा
`api.registerTool` कार्यान्वयन का पुनः उपयोग करना चाहें, तब फ़ैक्टरी टूल का उपयोग करें।

## आउटपुट अनुबंध

जब कोई टूल स्थिर JSON-संगत डेटा लौटाता है, तब `outputSchema` जोड़ें। यह
`AgentToolResult.details` में संग्रहीत मूल मान का वर्णन करता है, न कि
`content` में स्वरूपित टेक्स्ट का:

```typescript
tool({
  name: "shipment_list",
  description: "शिपमेंट सूचीबद्ध करें।",
  parameters: Type.Object({
    buyer: Type.Optional(Type.String()),
  }),
  outputSchema: Type.Array(
    Type.Object(
      {
        id: Type.String(),
        buyer: Type.String(),
        paid: Type.Boolean(),
        tons: Type.Number(),
      },
      { additionalProperties: false },
    ),
  ),
  execute: ({ buyer }) => listShipments(buyer),
});
```

[कोड मोड](/hi/tools/code-mode) और [टूल खोज](/hi/tools/tool-search) इस
स्कीमा को सीमित TypeScript-शैली के आउटपुट संकेत में बदलते हैं। इससे मॉडल
उसके आकार को देखने के लिए एक और मॉडल टर्न खर्च करने के बजाय एक ही प्रोग्राम में
ज्ञात परिणाम को कॉल और रूपांतरित कर सकता है।

OpenClaw कैटलॉग कॉल निष्पादित करने से पहले स्कीमा को कंपाइल करता है, फिर ब्रिज के माध्यम से लौटाने से पहले
टूल हुक के बाद अंतिम `details` मान को सत्यापित करता है।
अमान्य स्कीमा टूल को चला नहीं सकती; परिणाम में असंगति पूर्ण हो चुकी
कॉल को विफल करती है। त्रुटि न फेंकने वाले प्रत्येक परिणाम प्रकार को शामिल करें, जिसमें संरचित त्रुटि
प्रकार भी शामिल हैं, या परिणाम स्थिर न होने पर स्कीमा छोड़ दें। स्कीमा विवरणों में गोपनीय
या संवेदनशील मान न रखें, क्योंकि विश्वसनीय आउटपुट मेटाडेटा
मॉडल को दिखाई दे सकता है।
जब आप पूर्ण
संक्षिप्त आउटपुट संकेत चाहते हैं, तब ऑब्जेक्ट परतों पर `{ additionalProperties: false }` का उपयोग करें; खुले या काटे गए स्कीमा
`tools.describe(...)` के माध्यम से उपलब्ध रहते हैं, लेकिन पूर्ण त्वरित-सूचकांक अनुबंधों के रूप में
प्रचारित नहीं होते।

फ़ैक्टरी टूल अपने द्वारा लौटाए गए ठोस `AnyAgentTool` पर `outputSchema` घोषित करते हैं।
स्थिर `tool({ factory })` घोषणा अलग
आउटपुट स्कीमा स्वीकार नहीं करती, क्योंकि वह रनटाइम टूल से अलग हो सकता है।

## कॉन्फ़िगरेशन

`configSchema` वैकल्पिक है। इसे छोड़ने पर OpenClaw एक सख्त खाली ऑब्जेक्ट
स्कीमा लागू करता है; जनरेट किए गए मैनिफ़ेस्ट में फिर भी `configSchema` शामिल रहता है।

```typescript
export default defineToolPlugin({
  id: "no-config-tools",
  name: "No Config Tools",
  description: "ऐसे टूल जोड़ता है जिन्हें कॉन्फ़िगरेशन की आवश्यकता नहीं होती।",
  tools: () => [],
});
```

`configSchema` के साथ, दूसरा `execute` आर्ग्युमेंट उसी से टाइप किया जाता है:

```typescript
const configSchema = Type.Object({
  apiKey: Type.String(),
});

export default defineToolPlugin({
  id: "configured-tools",
  name: "Configured Tools",
  description: "कॉन्फ़िगर किए गए टूल जोड़ता है।",
  configSchema,
  tools: (tool) => [
    tool({
      name: "configured_ping",
      description: "जाँचें कि कॉन्फ़िगरेशन उपलब्ध है या नहीं।",
      parameters: Type.Object({}),
      execute: (_params, config) => ({ hasKey: config.apiKey.length > 0 }),
    }),
  ],
});
```

OpenClaw, Gateway कॉन्फ़िगरेशन में Plugin की एंट्री से Plugin कॉन्फ़िगरेशन पढ़ता है।
स्रोत या दस्तावेज़ उदाहरणों में गोपनीय मान हार्ड-कोड न करें; Plugin के सुरक्षा मॉडल के अनुसार कॉन्फ़िगरेशन, पर्यावरण
चर या SecretRefs का उपयोग करें।

## जनरेट किया गया मेटाडेटा

OpenClaw को Plugin रनटाइम कोड इंपोर्ट करने से पहले Plugin मैनिफ़ेस्ट पढ़ना आवश्यक है।
`defineToolPlugin` इसके लिए स्थिर मेटाडेटा उपलब्ध कराता है, और
`openclaw plugins build` इसे पैकेज में लिखता है। Plugin आईडी, नाम, विवरण, कॉन्फ़िगरेशन स्कीमा, सक्रियण या टूल
नाम बदलने के बाद जनरेटर को दोबारा चलाएँ:

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

एक टूल वाले Plugin के लिए जनरेट किया गया मैनिफ़ेस्ट:

```json
{
  "id": "stock-quotes",
  "name": "Stock Quotes",
  "description": "स्टॉक कोट के स्नैपशॉट प्राप्त करें।",
  "version": "0.1.0",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  },
  "activation": {
    "onStartup": true
  },
  "contracts": {
    "tools": ["stock_quote"]
  }
}
```

`contracts.tools` महत्वपूर्ण खोज अनुबंध है: यह OpenClaw को बताता है कि प्रत्येक
टूल का स्वामी कौन-सा Plugin है, बिना प्रत्येक इंस्टॉल किए गए Plugin का रनटाइम लोड किए।
पुराने मैनिफ़ेस्ट का अर्थ है कि कोई टूल खोज से गायब हो सकता है, या पंजीकरण
त्रुटि का दोष गलत Plugin पर लगाया जा सकता है।

## पैकेज मेटाडेटा

`openclaw plugins build`, `package.json` को भी चुनी गई रनटाइम
एंट्री के अनुरूप बनाता है:

```json
{
  "type": "module",
  "files": ["dist", "openclaw.plugin.json", "README.md"],
  "dependencies": {
    "typebox": "^1.1.38"
  },
  "peerDependencies": {
    "openclaw": ">=2026.5.17"
  },
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

बिल्ड किया गया JavaScript (`./dist/index.js`) वितरित करें, TypeScript स्रोत एंट्री नहीं।
स्रोत एंट्रियाँ केवल वर्कस्पेस-स्थानीय विकास के लिए काम करती हैं।

## CI में सत्यापन करें

जनरेट किया गया मेटाडेटा पुराना होने पर `plugins build --check` फ़ाइलों को दोबारा लिखे बिना
विफल हो जाता है:

```bash
npm run build
openclaw plugins build --entry ./dist/index.js --check
openclaw plugins validate --entry ./dist/index.js
npm test
```

OpenClaw SDK संगतता फ़ील्ड में TypeScript `@deprecated` एनोटेशन होते हैं,
जिन्हें संपादक माइग्रेशन चेतावनियों के रूप में दिखाते हैं। इन्हें CI में लागू करने के लिए
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/)
जैसा टाइप-जागरूक नियम सक्षम करें।
Oxlint टाइप-जागरूक नहीं है, इसलिए यह इन एनोटेशन को लागू नहीं कर सकता। इसलिए जनरेट किया गया
`plugins init` स्कैफ़ोल्ड अप्रचलन लिंट कॉन्फ़िगरेशन नहीं जोड़ता।

`plugins validate` जाँचता है कि:

- `openclaw.plugin.json` मौजूद है और सामान्य मैनिफ़ेस्ट लोडर से सफलतापूर्वक गुजरता है।
- वर्तमान एंट्री `defineToolPlugin` मेटाडेटा एक्सपोर्ट करती है।
- जनरेट किए गए मैनिफ़ेस्ट फ़ील्ड एंट्री मेटाडेटा से मेल खाते हैं।
- `contracts.tools` घोषित टूल नामों से मेल खाता है।
- `package.json`, `openclaw.extensions` को चयनित रनटाइम एंट्री पर इंगित करता है।

## स्थानीय रूप से इंस्टॉल और निरीक्षण करें

किसी अलग OpenClaw चेकआउट या इंस्टॉल किए गए CLI से पैकेज पाथ इंस्टॉल करें:

```bash
openclaw plugins install ./stock-quotes
openclaw plugins inspect stock-quotes --runtime
```

पैकेज किए गए स्मोक टेस्ट के लिए, पहले पैक करें और टारबॉल इंस्टॉल करें:

```bash
npm pack
openclaw plugins install npm-pack:./openclaw-plugin-stock-quotes-0.1.0.tgz
openclaw plugins inspect stock-quotes --runtime --json
```

इंस्टॉल करने के बाद, Gateway को रीस्टार्ट या रीलोड करें और एजेंट से
टूल का उपयोग करने को कहें। यदि टूल दिखाई नहीं देता है, तो कोड बदलने से पहले Plugin
रनटाइम और प्रभावी टूल कैटलॉग का निरीक्षण करें ([समस्या निवारण](#troubleshooting) देखें)।

## प्रकाशित करें

पैकेज तैयार हो जाने पर उसे ClawHub के माध्यम से प्रकाशित करें। `clawhub package publish`
एक स्रोत स्वीकार करता है: कोई स्थानीय फ़ोल्डर, GitHub रेपो (`owner/repo[@ref]`), या
टारबॉल URL।

```bash
clawhub package publish ./stock-quotes --dry-run
clawhub package publish ./stock-quotes
```

स्पष्ट ClawHub लोकेटर के साथ इंस्टॉल करें:

```bash
openclaw plugins install clawhub:your-org/stock-quotes
```

लॉन्च बदलाव के दौरान साधारण npm पैकेज स्पेक अब भी npm से इंस्टॉल होते हैं, लेकिन
OpenClaw plugins की खोज और वितरण के लिए ClawHub पसंदीदा माध्यम है।
स्वामी के दायरे और रिलीज़ समीक्षा के लिए [ClawHub प्रकाशन](/hi/clawhub/publishing) देखें।

## समस्या निवारण

### `plugin entry not found: ./dist/index.js`

चयनित एंट्री फ़ाइल मौजूद नहीं है। `npm run build` चलाएँ, फिर
`openclaw plugins build --entry ./dist/index.js` या
`openclaw plugins validate --entry ./dist/index.js` दोबारा चलाएँ।

### `plugin entry does not expose defineToolPlugin metadata`

एंट्री ने `defineToolPlugin` द्वारा बनाया गया मान एक्सपोर्ट नहीं किया। पुष्टि करें कि
मॉड्यूल का डिफ़ॉल्ट एक्सपोर्ट `defineToolPlugin(...)` का परिणाम है, या
`--entry` के साथ सही एंट्री दें।

### `openclaw.plugin.json generated metadata is stale`

मैनिफ़ेस्ट अब एंट्री मेटाडेटा से मेल नहीं खाता। चलाएँ:

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

`openclaw.plugin.json` और `package.json`, दोनों के बदलाव कमिट करें।

### `package.json openclaw.extensions must include ./dist/index.js`

पैकेज मेटाडेटा किसी दूसरी रनटाइम एंट्री की ओर इंगित करता है।
`openclaw plugins build --entry ./dist/index.js` चलाएँ, ताकि जनरेटर पैकेज मेटाडेटा को उस एंट्री के अनुरूप करे
जिसे आप शिप करना चाहते हैं।

### `Cannot find package 'typebox'`

बिल्ट Plugin रनटाइम पर `typebox` इंपोर्ट करता है। इसे `dependencies` में रखें,
फिर से इंस्टॉल और बिल्ड करें तथा सत्यापन दोबारा चलाएँ।

### इंस्टॉल के बाद टूल दिखाई नहीं देता

इनकी इसी क्रम में जाँच करें:

1. `openclaw plugins inspect <plugin-id> --runtime`
2. `openclaw plugins validate --root <plugin-root> --entry ./dist/index.js`
3. `openclaw.plugin.json` में अपेक्षित टूल नामों वाला `contracts.tools` मौजूद है।
4. `package.json` में `openclaw.extensions: ["./dist/index.js"]` मौजूद है।
5. Plugin इंस्टॉल करने के बाद Gateway को रीस्टार्ट या रीलोड किया गया था।

## यह भी देखें

- [Plugins बनाना](/hi/plugins/building-plugins)
- [Plugin एंट्री पॉइंट](/hi/plugins/sdk-entrypoints)
- [Plugin SDK सबपाथ](/hi/plugins/sdk-subpaths)
- [Plugin मैनिफ़ेस्ट](/hi/plugins/manifest)
- [Plugins CLI](/hi/cli/plugins)
- [ClawHub प्रकाशन](/hi/clawhub/publishing)
