---
doc-schema-version: 1
read_when:
    - आप Control UI में plugins ब्राउज़, इंस्टॉल, सक्षम या अक्षम करना चाहते हैं
    - आप Plugin की सूची देखने, इंस्टॉल करने, अपडेट करने, निरीक्षण करने या अनइंस्टॉल करने के त्वरित उदाहरण चाहते हैं
    - आप Plugin इंस्टॉल करने का स्रोत चुनना चाहते हैं
    - आप Plugin पैकेज प्रकाशित करने के लिए सही संदर्भ चाहते हैं
sidebarTitle: Manage plugins
summary: Control UI या CLI से OpenClaw plugins प्रबंधित करें
title: Plugin प्रबंधित करें
x-i18n:
    generated_at: "2026-07-27T21:25:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9101d5c3630b618a043f1e71fdf5fa083698cc23694ccdc773d295a37c4c1ef3
    source_path: plugins/manage-plugins.md
    workflow: 16
---

Control UI सामान्य खोज, इंस्टॉल, सक्षम और अक्षम करने के
कार्यप्रवाह को समेटता है। CLI अपडेट, अनइंस्टॉल, उन्नत कॉन्फ़िगरेशन और स्पष्ट
इंस्टॉल-स्रोत नियंत्रण जोड़ता है। इसके संपूर्ण कमांड अनुबंध, फ़्लैग, स्रोत-चयन
नियमों और सीमांत स्थितियों के लिए, [`openclaw plugins`](/hi/cli/plugins) देखें।

सामान्य CLI कार्यप्रवाह: कोई पैकेज खोजें, उसे ClawHub, npm, git या किसी
स्थानीय पथ से इंस्टॉल करें, प्रबंधित Gateway को अपने-आप पुनः आरंभ होने दें
(या उसे मैन्युअल रूप से पुनः आरंभ करें), फिर Plugin के रनटाइम पंजीकरणों को सत्यापित करें।

## Control UI का उपयोग करें

Control UI में **Plugins** खोलें, या कॉन्फ़िगर किए गए Control UI आधार पथ के
सापेक्ष `/settings/plugins` का उपयोग करें। उदाहरण के लिए, `/openclaw` आधार पथ
`/openclaw/settings/plugins` का उपयोग करता है। पृष्ठ में दो टैब हैं:

- **इंस्टॉल किए गए** श्रेणी के अनुसार समूहित पूरी स्थानीय सूची दिखाता है (चैनल,
  मॉडल प्रदाता, मेमोरी, टूल)। प्रत्येक पंक्ति विस्तृत दृश्य खोलती है; इसका ओवरफ़्लो
  (`…`) मेन्यू Plugin को सक्षम या अक्षम करता है और बाहरी रूप से इंस्टॉल किए गए
  Plugins के लिए **हटाएँ** विकल्प देता है। यह टैब कॉन्फ़िगर किए गए
  [MCP सर्वर](/hi/cli/mcp) भी सूचीबद्ध करता है, जिनके लिए वही मेन्यू-संचालित सक्षम, अक्षम और हटाने की
  कार्रवाइयाँ उपलब्ध हैं, जो Gateway कॉन्फ़िगरेशन में `mcp.servers` को संपादित करती हैं।
- **खोजें** स्टोर है: OpenClaw के साथ शामिल चुनिंदा Plugins, आधिकारिक
  बाहरी Plugins और क्यूरेट किया गया कनेक्टर संग्रह। कनेक्टर कार्ड या तो एक
  क्लिक में होस्ट किया गया MCP सर्वर जोड़ते हैं (GitHub, Notion, Linear, Sentry,
  Home Assistant), या पहले से भरी हुई ClawHub खोज खोलते हैं। खोज
  बॉक्स में टाइप करने पर [ClawHub](https://clawhub.ai/plugins) में इनलाइन क्वेरी की जाती है और डाउनलोड
  संख्या तथा स्रोत-सत्यापन बैज वाला **ClawHub से** अनुभाग जोड़ा जाता है।

शामिल Plugins के लिए पैकेज इंस्टॉल करने की आवश्यकता नहीं होती। उनकी मेन्यू कार्रवाई **सक्षम करें**
या **अक्षम करें** होती है। उदाहरण के लिए, Workboard OpenClaw के साथ शामिल है और डिफ़ॉल्ट
रूप से अक्षम होता है, इसलिए उसे चालू करने के लिए **सक्षम करें** चुनें। बंडल किए गए Plugins
हटाए नहीं जा सकते, केवल अक्षम किए जा सकते हैं।

कैटलॉग और खोज पहुँच के लिए `operator.read` आवश्यक है। इंस्टॉल, सक्षम, अक्षम,
हटाने और MCP सर्वर में बदलाव के लिए `operator.admin` आवश्यक है। ClawHub इंस्टॉल
Gateway द्वारा किया जाता है और उसकी विश्वास, अखंडता तथा Plugin-इंस्टॉल
नीति जाँचों को बनाए रखता है। प्रशासक के रूप में इंस्टॉल किए गए Plugin को सक्षम करने पर
चयनित Plugin को मौजूदा प्रतिबंधात्मक `plugins.allow` सूची में जोड़कर
उस स्पष्ट विश्वास को भी दर्ज किया जाता है। स्पष्ट `plugins.deny` प्रविष्टि प्रामाणिक बनी रहती है और
Plugin को सक्षम करने से पहले उसे हटाना आवश्यक है।

Plugin कोड इंस्टॉल करने या हटाने के लिए Gateway पुनः आरंभ करना आवश्यक है। जब इंस्टॉल किया गया
Plugin और वर्तमान Gateway रनटाइम इसका समर्थन करते हैं, तो सक्षमता में बदलाव
पुनः आरंभ किए बिना लागू किए जा सकते हैं; अन्यथा UI बताता है कि पुनः आरंभ करना आवश्यक है।
OAuth-समर्थित MCP कनेक्टर जोड़े जाने के बाद भी CLI से एक बार
`openclaw mcp login <name>` चलाना आवश्यक होता है।

Control UI मनमाने npm, git या स्थानीय-पथ स्रोतों से इंस्टॉल नहीं करता,
Plugins अपडेट नहीं करता और विस्तृत Plugin कॉन्फ़िगरेशन उपलब्ध नहीं कराता। इन कार्रवाइयों के लिए
नीचे दिए गए CLI कार्यप्रवाहों का उपयोग करें।

## Plugins सूचीबद्ध करें और खोजें

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
openclaw plugins search "calendar"
```

स्क्रिप्ट के लिए `--json`:

```bash
openclaw plugins list --json \
  | jq '.plugins[] | {id, enabled, format, source, dependencyStatus}'
```

`plugins list` एक कोल्ड सूची जाँच है: OpenClaw कॉन्फ़िगरेशन,
मैनिफ़ेस्ट और स्थायी Plugin रजिस्ट्री से क्या खोज सकता है। यह प्रमाणित नहीं करता कि
पहले से चल रहे Gateway ने Plugin रनटाइम आयात किया है। JSON आउटपुट में
रजिस्ट्री निदान और प्रत्येक Plugin का `dependencyStatus` शामिल होता है (क्या घोषित
`dependencies`/`optionalDependencies` डिस्क पर रिज़ॉल्व होते हैं)।

`plugins search` इंस्टॉल किए जा सकने वाले Plugin पैकेजों के लिए ClawHub से क्वेरी करता है और
प्रत्येक परिणाम के लिए इंस्टॉल संकेत (`openclaw plugins install clawhub:<package>`) प्रिंट करता है।

## Plugins सक्षम और अक्षम करें

```bash
openclaw plugins enable <plugin-id>
openclaw plugins disable <plugin-id>
```

इंस्टॉल की गई फ़ाइलों को बदले बिना Plugin की कॉन्फ़िगरेशन प्रविष्टि टॉगल करता है। कुछ
बंडल किए गए Plugins (बंडल मॉडल/वाक् प्रदाता, बंडल ब्राउज़र Plugin)
डिफ़ॉल्ट रूप से सक्षम होते हैं; अन्य को इंस्टॉल करने के बाद `enable` की आवश्यकता होती है।

## Plugins इंस्टॉल करें

```bash
# Plugin पैकेजों के लिए ClawHub खोजें।
openclaw plugins search "calendar"

# ClawHub से इंस्टॉल करें।
openclaw plugins install clawhub:<package>
openclaw plugins install clawhub:<package>@1.2.3
openclaw plugins install clawhub:<package>@beta

# npm से इंस्टॉल करें।
openclaw plugins install npm:<package>
openclaw plugins install npm:@scope/openclaw-plugin@1.2.3
openclaw plugins install npm:@openclaw/codex

# स्थानीय npm-pack आर्टिफ़ैक्ट से इंस्टॉल करें।
openclaw plugins install npm-pack:<path.tgz>

# git या स्थानीय डेवलपमेंट चेकआउट से इंस्टॉल करें।
openclaw plugins install git:github.com/acme/openclaw-plugin@v1.0.0
openclaw plugins install ./my-plugin
openclaw plugins install --link ./my-plugin
```

लॉन्च परिवर्तन के दौरान बिना उपसर्ग वाले पैकेज विनिर्देश npm से इंस्टॉल होते हैं, जब तक कि
नाम किसी बंडल किए गए या आधिकारिक Plugin आईडी से मेल न खाता हो; उस स्थिति में OpenClaw
इसके बजाय उस स्थानीय/आधिकारिक प्रति का उपयोग करता है। निर्धारक स्रोत चयन के लिए
`clawhub:`, `npm:`, `git:` या `npm-pack:` का उपयोग करें।
OpenClaw के बंडल और आधिकारिक कैटलॉग पैकेजों को ClawHub पैकेजों के साथ विश्वसनीय माना जाता है।
नए मनमाने npm, git, स्थानीय पथ/अभिलेखागार, `npm-pack:` या मार्केटप्लेस स्रोतों के लिए,
स्रोत की समीक्षा और उस पर विश्वास करने के बाद गैर-संवादात्मक इंस्टॉल में
`--force` आवश्यक है।

`--force` बिना संकेत दिए गैर-ClawHub स्रोत की पुष्टि करता है और आवश्यकता होने पर
मौजूदा इंस्टॉल लक्ष्य को अधिलेखित करता है। ट्रैक किए गए npm, ClawHub या hook-pack इंस्टॉल के
नियमित अपग्रेड के लिए इसके बजाय `openclaw plugins update` का उपयोग करें। `--link` के साथ,
`--force` केवल स्रोत की पुष्टि करता है; लिंक की गई डायरेक्टरी कॉपी या अधिलेखित नहीं होती।

यदि नए इंस्टॉल किए गए Plugin के लिए ऐसा कॉन्फ़िगरेशन आवश्यक है जो अभी मौजूद नहीं है,
तो OpenClaw इंस्टॉल दर्ज करता है लेकिन Plugin को अक्षम छोड़ देता है।
`plugins.entries.<id>.config` कॉन्फ़िगर करें, फिर `openclaw plugins enable <id>` चलाएँ। यदि
मौजूदा कॉन्फ़िगरेशन प्रविष्टि मौजूद है लेकिन अमान्य है, तो उसे दोबारा लिखे बिना इंस्टॉल विफल हो जाता है।

## पुनः आरंभ करें और निरीक्षण करें

कॉन्फ़िगरेशन रीलोड सक्षम होने पर चल रहा प्रबंधित Gateway, Plugin कोड इंस्टॉल,
अपडेट या अनइंस्टॉल करने के बाद अपने-आप पुनः आरंभ होता है। यदि Gateway
अप्रबंधित है या रीलोड अक्षम है, तो लाइव रनटाइम सतहों की जाँच से पहले
उसे स्वयं पुनः आरंभ करें:

```bash
openclaw gateway restart
openclaw plugins inspect <plugin-id> --runtime --json
```

`inspect --runtime` Plugin मॉड्यूल लोड करता है और प्रमाणित करता है कि उसने रनटाइम
सतहें (टूल, हुक, सेवाएँ, Gateway विधियाँ, HTTP रूट, Plugin-स्वामित्व वाले
CLI कमांड) पंजीकृत की हैं। साधारण `inspect` और `list` केवल कोल्ड
मैनिफ़ेस्ट/कॉन्फ़िगरेशन/रजिस्ट्री जाँच हैं।

## Plugins अपडेट करें

```bash
openclaw plugins update <plugin-id>
openclaw plugins update <npm-package-or-spec>
openclaw plugins update --all
openclaw plugins update <plugin-id> --dry-run
```

Plugin आईडी देने पर उसके ट्रैक किए गए इंस्टॉल विनिर्देश का पुनः उपयोग किया जाता है:
संग्रहीत dist-tags (`@beta`) और सटीक पिन किए गए संस्करण बाद के
`update <plugin-id>` रन में भी बने रहते हैं।

`openclaw plugins update --all` सामूहिक रखरखाव पथ है। यह अब भी
सामान्य ट्रैक किए गए इंस्टॉल विनिर्देशों का सम्मान करता है, लेकिन विश्वसनीय आधिकारिक OpenClaw
Plugin रिकॉर्ड किसी पुराने सटीक आधिकारिक पैकेज पर पिन रहने के बजाय
वर्तमान आधिकारिक कैटलॉग लक्ष्य के साथ सिंक होते हैं; जब `update.channel`
`beta` होता है, तो वह सिंक बीटा रिलीज़ शृंखला को प्राथमिकता देता है। किसी सटीक
या टैग किए गए आधिकारिक विनिर्देश को अपरिवर्तित रखने के लिए लक्षित
`update <plugin-id>` का उपयोग करें।

npm इंस्टॉल के लिए, ट्रैक किए गए रिकॉर्ड को बदलने हेतु स्पष्ट पैकेज विनिर्देश दें:

```bash
openclaw plugins update @scope/openclaw-plugin@beta
openclaw plugins update @scope/openclaw-plugin
```

जब कोई Plugin पहले किसी सटीक संस्करण या टैग पर पिन किया गया था, तो दूसरा कमांड
उसे रजिस्ट्री की डिफ़ॉल्ट रिलीज़ शृंखला पर वापस ले जाता है।

सटीक फ़ॉलबैक और पिनिंग नियमों के लिए [`openclaw plugins`](/hi/cli/plugins#update)
देखें।

## Plugins अनइंस्टॉल करें

```bash
openclaw plugins uninstall <plugin-id> --dry-run
openclaw plugins uninstall <plugin-id>
openclaw plugins uninstall <plugin-id> --keep-files
```

अनइंस्टॉल Plugin की कॉन्फ़िगरेशन प्रविष्टि, स्थायी Plugin इंडेक्स रिकॉर्ड,
अनुमति/अस्वीकृति सूची प्रविष्टियाँ और लागू होने पर लिंक की गई `plugins.load.paths`
प्रविष्टियाँ हटाता है। जब तक आप `--keep-files` नहीं देते, प्रबंधित इंस्टॉल
डायरेक्टरी हटा दी जाती है। अनइंस्टॉल से Plugin स्रोत बदलने पर चल रहा प्रबंधित Gateway
अपने-आप पुनः आरंभ होता है।

Nix मोड (`OPENCLAW_NIX_MODE=1`) में Plugin इंस्टॉल, अपडेट, अनइंस्टॉल,
सक्षम और अक्षम करना—सभी अक्षम होते हैं; इसके बजाय इंस्टॉल के Nix स्रोत में
इन विकल्पों को प्रबंधित करें।

## स्रोत चुनें

| स्रोत       | इसका उपयोग कब करें                                                          | उदाहरण                                                        |
| ----------- | --------------------------------------------------------------------------- | -------------------------------------------------------------- |
| ClawHub     | जब आपको OpenClaw-मूल खोज, स्कैन सारांश, संस्करण और संकेत चाहिए             | `openclaw plugins install clawhub:<package>`                   |
| git         | जब आपको किसी रिपॉज़िटरी से शाखा, टैग या कमिट चाहिए                          | `openclaw plugins install git:github.com/<owner>/<repo>@<ref>` |
| स्थानीय पथ | जब आप उसी मशीन पर किसी Plugin का विकास या परीक्षण कर रहे हों                | `openclaw plugins install --link ./my-plugin`                  |
| मार्केटप्लेस | जब आप Claude-संगत मार्केटप्लेस Plugin इंस्टॉल कर रहे हों                   | `openclaw plugins install <plugin> --marketplace <source>`     |
| npm pack    | जब आप npm इंस्टॉल अर्थ-विज्ञान के माध्यम से स्थानीय पैकेज आर्टिफ़ैक्ट प्रमाणित कर रहे हों | `openclaw plugins install npm-pack:<path.tgz>`                 |
| npmjs.com   | जब आप पहले से JavaScript पैकेज वितरित करते हों या npm dist-tags/निजी रजिस्ट्री की आवश्यकता हो | `openclaw plugins install npm:@acme/openclaw-plugin`           |

प्रबंधित स्थानीय पथ इंस्टॉल Plugin डायरेक्टरी या अभिलेखागार होने चाहिए। स्वतंत्र
Plugin फ़ाइलों को `plugins install` से इंस्टॉल करने के बजाय
`plugins.load.paths` में रखें।

## Plugins प्रकाशित करें

ClawHub, OpenClaw Plugins के लिए प्राथमिक सार्वजनिक खोज सतह है। जब आप चाहते हैं कि
उपयोगकर्ता इंस्टॉल करने से पहले Plugin मेटाडेटा, संस्करण इतिहास, रजिस्ट्री
स्कैन परिणाम और इंस्टॉल संकेत खोज सकें, तो वहाँ प्रकाशित करें।

```bash
npm i -g clawhub
clawhub login
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
clawhub package publish your-org/your-plugin@v1.0.0
```

प्रकाशित करने से पहले मूल npm Plugins में Plugin मैनिफ़ेस्ट (`openclaw.plugin.json`) और
`package.json` मेटाडेटा होना आवश्यक है:

```json package.json
{
  "name": "@acme/openclaw-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

```bash
npm publish --access public
openclaw plugins install npm:@acme/openclaw-plugin
openclaw plugins install npm:@acme/openclaw-plugin@beta
openclaw plugins install npm:@acme/openclaw-plugin@1.0.0
```

इस पृष्ठ को प्रकाशन संदर्भ मानने के बजाय संपूर्ण प्रकाशन अनुबंध के लिए
इन पृष्ठों का उपयोग करें:

- [ClawHub प्रकाशन](/hi/clawhub/publishing) स्वामी, स्कोप,
  रिलीज़, समीक्षा, पैकेज सत्यापन और पैकेज हस्तांतरण की व्याख्या करता है।
- [Plugins बनाना](/hi/plugins/building-plugins) संपूर्ण Plugin
  पैकेज संरचना (`openclaw.plugin.json` सहित) और पहला प्रकाशन
  कार्यप्रवाह दिखाता है।
- [Plugin मैनिफ़ेस्ट](/hi/plugins/manifest) मूल Plugin मैनिफ़ेस्ट
  फ़ील्ड परिभाषित करता है।

यदि वही पैकेज ClawHub और npm दोनों पर उपलब्ध है, तो किसी एक स्रोत को बाध्य करने के लिए
स्पष्ट `clawhub:` या `npm:` उपसर्ग का उपयोग करें।

## संबंधित

- [Plugins](/hi/tools/plugin) - इंस्टॉल करें, कॉन्फ़िगर करें, पुनः आरंभ करें और समस्याओं का निवारण करें
- [`openclaw plugins`](/hi/cli/plugins) - संपूर्ण CLI संदर्भ
- [सामुदायिक Plugins](/hi/plugins/community) - सार्वजनिक खोज और ClawHub पर प्रकाशन
- [ClawHub](/hi/clawhub/cli) - रजिस्ट्री CLI संचालन
- [Plugins बनाना](/hi/plugins/building-plugins) - एक Plugin पैकेज बनाएँ
- [Plugin मैनिफ़ेस्ट](/hi/plugins/manifest) - मैनिफ़ेस्ट और पैकेज मेटाडेटा
