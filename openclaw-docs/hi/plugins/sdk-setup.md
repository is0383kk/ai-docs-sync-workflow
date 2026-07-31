---
read_when:
    - आप किसी Plugin में सेटअप विज़ार्ड जोड़ रहे हैं
    - आपको setup-entry.ts और index.ts के बीच का अंतर समझना होगा
    - आप Plugin कॉन्फ़िगरेशन स्कीमा या package.json की OpenClaw मेटाडेटा परिभाषित कर रहे हैं
sidebarTitle: Setup and config
summary: सेटअप विज़ार्ड, setup-entry.ts, कॉन्फ़िगरेशन स्कीमा और package.json मेटाडेटा
title: Plugin सेटअप और कॉन्फ़िगरेशन
x-i18n:
    generated_at: "2026-07-27T20:17:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b07e3fa365939fa9c0885b31b7894f5e734313a7deef2297e316956063d97e45
    source_path: plugins/sdk-setup.md
    workflow: 16
---

Plugin पैकेजिंग (`package.json` मेटाडेटा), मैनिफ़ेस्ट (`openclaw.plugin.json`), सेटअप प्रविष्टियों और कॉन्फ़िग स्कीमा के लिए संदर्भ।

<Tip>
**चरण-दर-चरण मार्गदर्शिका खोज रहे हैं?** कार्य-विधि मार्गदर्शिकाएँ संदर्भ सहित पैकेजिंग को समझाती हैं: [चैनल Plugin](/plugins/sdk-channel-plugins#step-1-package-and-manifest) और [प्रोवाइडर Plugin](/hi/plugins/sdk-provider-plugins#step-1-package-and-manifest)।
</Tip>

## पैकेज मेटाडेटा

आपके `package.json` में एक `openclaw` फ़ील्ड होना आवश्यक है, जो Plugin सिस्टम को बताता है कि आपका Plugin क्या प्रदान करता है:

<Tabs>
  <Tab title="चैनल Plugin">
    ```json
    {
      "name": "@myorg/openclaw-my-channel",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "my-channel",
          "label": "मेरा चैनल",
          "blurb": "चैनल का संक्षिप्त विवरण।"
        }
      }
    }
    ```
  </Tab>
  <Tab title="प्रोवाइडर Plugin / ClawHub आधाररेखा">
    ```json openclaw-clawhub-package.json
    {
      "name": "@myorg/openclaw-my-plugin",
      "version": "1.0.0",
      "type": "module",
      "dependencies": {
        "typebox": "1.1.39"
      },
      "peerDependencies": {
        "openclaw": ">=2026.3.24-beta.2"
      },
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
      }
    }
    ```
  </Tab>
</Tabs>

<Note>
ClawHub पर बाहरी रूप से प्रकाशित करने के लिए `compat` और `build` आवश्यक हैं। मानक प्रकाशन स्निपेट `docs/snippets/plugin-publish/` में उपलब्ध हैं।
</Note>

### `openclaw` फ़ील्ड

<ParamField path="extensions" type="string[]">
  प्रवेश-बिंदु फ़ाइलें (पैकेज रूट के सापेक्ष)। वर्कस्पेस और git चेकआउट विकास के लिए मान्य स्रोत प्रविष्टियाँ।
</ParamField>
<ParamField path="runtimeExtensions" type="string[]">
  `extensions` के लिए निर्मित JavaScript समकक्ष, जिन्हें OpenClaw द्वारा इंस्टॉल किया गया npm पैकेज लोड करते समय प्राथमिकता दी जाती है। स्रोत/निर्मित समाधान क्रम के लिए [SDK प्रवेश-बिंदु](/hi/plugins/sdk-entrypoints) देखें।
</ParamField>
<ParamField path="setupEntry" type="string">
  हल्की, केवल-सेटअप प्रविष्टि (वैकल्पिक)।
</ParamField>
<ParamField path="runtimeSetupEntry" type="string">
  `setupEntry` के लिए निर्मित JavaScript समकक्ष। `setupEntry` को भी सेट करना आवश्यक है।
</ParamField>
<ParamField path="plugin" type="object">
  `{ id, label }` फ़ॉलबैक Plugin पहचान, जिसका उपयोग तब किया जाता है जब किसी Plugin में ऐसी चैनल/प्रोवाइडर मेटाडेटा न हो जिससे आईडी या लेबल प्राप्त किया जा सके।
</ParamField>
<ParamField path="channel" type="object">
  सेटअप, चयनकर्ता, त्वरित प्रारंभ और स्थिति सतहों के लिए चैनल कैटलॉग मेटाडेटा।
</ParamField>
<ParamField path="install" type="object">
  इंस्टॉल संकेत: `npmSpec`, `localPath`, `defaultChoice`, `minHostVersion`, `expectedIntegrity`, `allowInvalidConfigRecovery`, `requiredPlatformPackages`।
</ParamField>
<ParamField path="startup" type="object">
  स्टार्टअप व्यवहार फ़्लैग।
</ParamField>
<ParamField path="compat" type="object">
  इस Plugin द्वारा समर्थित `pluginApi` संस्करण सीमा। बाहरी ClawHub प्रकाशनों के लिए आवश्यक।
</ParamField>

<Note>
प्रोवाइडर आईडी (`providers: string[]`) मैनिफ़ेस्ट मेटाडेटा हैं, पैकेज मेटाडेटा नहीं। उन्हें यहाँ नहीं, बल्कि `openclaw.plugin.json` में घोषित करें—[Plugin मैनिफ़ेस्ट](/hi/plugins/manifest) देखें।
</Note>

### `openclaw.channel`

`openclaw.channel` रनटाइम लोड होने से पहले चैनल खोज और सेटअप सतहों के लिए कम लागत वाला पैकेज मेटाडेटा है।

### चैनल-स्वामित्व वाले सेटअप फ़ील्ड

चैनल Plugin को `defineChannelSetupContract(...)` के साथ रनटाइम कोड में सेटअप फ़ील्ड एक बार परिभाषित करने चाहिए और `openclaw.channel.setup.fields` के अंतर्गत उनका संगत क्रमांकन-योग्य प्रक्षेपण प्रकाशित करना चाहिए। रनटाइम परिभाषा Plugin-स्थानीय इनपुट प्रकार का अनुमान लगाती है, निर्देशित और गैर-इंटरैक्टिव दोनों मानों को पार्स करती है और चैनल-विशिष्ट कुंजियों को कोर प्रकारों से बाहर रखती है। पैकेज मेटाडेटा `openclaw channels add <channel-id> --help` और `openclaw channels add --channel <channel-id> --help` को Plugin लोड किए बिना केवल चयनित चैनल के विकल्प खोजने देता है।

```ts
import { defineChannelSetupContract } from "openclaw/plugin-sdk/channel-setup";

export const setupContract = defineChannelSetupContract({
  fields: {
    endpoint: {
      kind: "string",
      cli: { flags: "--endpoint <url>", description: "सेवा एंडपॉइंट" },
    },
    transport: {
      kind: "choice",
      choices: ["native", "container"],
      cli: { flags: "--transport <kind>", description: "ट्रांसपोर्ट स्वामी" },
    },
  },
  adapter: {
    applyAccountConfig: ({ cfg, input }) => ({
      ...cfg,
      channels: { ...cfg.channels, example: input },
    }),
  },
});
```

```json
{
  "openclaw": {
    "channel": {
      "id": "example",
      "setup": {
        "fields": [
          {
            "key": "endpoint",
            "kind": "string",
            "cli": { "flags": "--endpoint <url>", "description": "सेवा एंडपॉइंट" }
          },
          {
            "key": "transport",
            "kind": "choice",
            "choices": ["native", "container"],
            "cli": { "flags": "--transport <kind>", "description": "ट्रांसपोर्ट स्वामी" }
          }
        ]
      }
    }
  }
}
```

समर्थित फ़ील्ड प्रकार `string`, `boolean`, `integer`, `string-list` और `choice` हैं। क्रेडेंशियल के लिए `sensitive: true` का उपयोग करें। प्रत्येक फ़ील्ड कुंजी अपने दीर्घ CLI फ़्लैग के camelCased एट्रिब्यूट नाम के बराबर होनी चाहिए, जिसमें कोई निषेधात्मक रूप भी शामिल है, जैसे `--api-token` के लिए `apiToken`। जब सकारात्मक और `--no-*` दोनों रूप आवश्यक हों, तब बूलियन फ़ील्ड `cli.negatedFlags` जोड़ सकते हैं। `channel`, `account` और अकाउंट प्रदर्शन `name` साझा नियंत्रण आवरण बने रहते हैं।

जारी किया गया `setup`/`ChannelSetupInput` अडैप्टर मौजूदा बाहरी Plugin के लिए उपलब्ध रहता है। नए Plugin को `setupContract` उजागर करना चाहिए; दोनों मौजूद होने पर OpenClaw हमेशा इसे प्राथमिकता देता है।

| फ़ील्ड                                  | प्रकार       | इसका अर्थ                                                                 |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------------- |
| `id`                                   | `string`   | मानक चैनल आईडी।                                                         |
| `label`                                | `string`   | प्राथमिक चैनल लेबल।                                                        |
| `selectionLabel`                       | `string`   | चयनकर्ता/सेटअप लेबल, जब इसे `label` से अलग होना चाहिए।                        |
| `detailLabel`                          | `string`   | अधिक समृद्ध चैनल कैटलॉग और स्थिति सतहों के लिए द्वितीयक विवरण लेबल।       |
| `docsPath`                             | `string`   | सेटअप और चयन लिंक के लिए दस्तावेज़ पथ।                                      |
| `docsLabel`                            | `string`   | दस्तावेज़ लिंक के लिए प्रयुक्त ओवरराइड लेबल, जब इसे चैनल आईडी से अलग होना चाहिए। |
| `blurb`                                | `string`   | संक्षिप्त ऑनबोर्डिंग/कैटलॉग विवरण।                                         |
| `order`                                | `number`   | चैनल कैटलॉग में क्रमबद्धता।                                               |
| `aliases`                              | `string[]` | चैनल चयन के लिए अतिरिक्त खोज उपनाम।                                   |
| `preferOver`                           | `string[]` | निम्न-प्राथमिकता वाले Plugin/चैनल आईडी, जिनसे इस चैनल को ऊपर रखा जाना चाहिए।                |
| `systemImage`                          | `string`   | चैनल UI कैटलॉग के लिए वैकल्पिक आइकन/सिस्टम-इमेज नाम।                      |
| `selectionDocsPrefix`                  | `string`   | चयन सतहों में दस्तावेज़ लिंक से पहले का उपसर्ग पाठ।                          |
| `selectionDocsOmitLabel`               | `boolean`  | चयन पाठ में लेबलयुक्त दस्तावेज़ लिंक के बजाय दस्तावेज़ पथ सीधे दिखाएँ। |
| `selectionExtras`                      | `string[]` | चयन पाठ में जोड़ी गई अतिरिक्त छोटी स्ट्रिंग।                               |
| `markdownCapable`                      | `boolean`  | आउटबाउंड फ़ॉर्मेटिंग निर्णयों के लिए चैनल को Markdown-सक्षम चिह्नित करता है।      |
| `exposure`                             | `object`   | सेटअप, कॉन्फ़िगर की गई सूचियों और दस्तावेज़ सतहों के लिए चैनल दृश्यता नियंत्रण।   |
| `quickstartAllowFrom`                  | `boolean`  | इस चैनल को मानक त्वरित प्रारंभ `allowFrom` सेटअप प्रवाह में शामिल करें।         |
| `forceAccountBinding`                  | `boolean`  | केवल एक अकाउंट मौजूद होने पर भी स्पष्ट अकाउंट बाइंडिंग आवश्यक करें।           |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | इस चैनल के लिए घोषणा लक्ष्य हल करते समय सेशन खोज को प्राथमिकता दें।       |
| `setup`                                | `object`   | आलसी CLI विकल्प खोज के लिए प्रयुक्त क्रमांकन-योग्य चैनल-स्वामित्व वाले सेटअप फ़ील्ड।   |

उदाहरण:

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "मेरा चैनल",
      "selectionLabel": "मेरा चैनल (स्व-होस्टेड)",
      "detailLabel": "मेरा चैनल बॉट",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Webhook-आधारित स्व-होस्टेड चैट एकीकरण।",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "मार्गदर्शिका:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` निम्न का समर्थन करता है:

- `configured`: कॉन्फ़िगर की गई/स्थिति-शैली सूची सतहों में चैनल शामिल करें
- `setup`: इंटरैक्टिव सेटअप/कॉन्फ़िगर चयनकर्ताओं में चैनल शामिल करें
- `docs`: दस्तावेज़/नेविगेशन सतहों में चैनल को सार्वजनिक-सामना वाला चिह्नित करें

### `openclaw.install`

`openclaw.install` पैकेज मेटाडेटा है, मैनिफ़ेस्ट मेटाडेटा नहीं।

| फ़ील्ड                        | प्रकार                                | इसका अर्थ                                                                     |
| ---------------------------- | ----------------------------------- | --------------------------------------------------------------------------------- |
| `clawhubSpec`                | `string`                            | इंस्टॉल/अपडेट और ऑनबोर्डिंग के माँग पर इंस्टॉल प्रवाहों के लिए प्रामाणिक ClawHub विनिर्देश। |
| `npmSpec`                    | `string`                            | इंस्टॉल/अपडेट फ़ॉलबैक प्रवाहों के लिए प्रामाणिक npm विनिर्देश।                             |
| `localPath`                  | `string`                            | स्थानीय विकास या बंडल किया गया इंस्टॉल पथ।                                        |
| `defaultChoice`              | `"clawhub"` \| `"npm"` \| `"local"` | एकाधिक स्रोत उपलब्ध होने पर पसंदीदा इंस्टॉल स्रोत।                     |
| `minHostVersion`             | `string`                            | न्यूनतम समर्थित OpenClaw संस्करण, `>=x.y.z` या `>=x.y.z-prerelease`।            |
| `expectedIntegrity`          | `string`                            | पिन किए गए इंस्टॉल के लिए अपेक्षित npm dist अखंडता स्ट्रिंग, आमतौर पर `sha512-...`।    |
| `allowInvalidConfigRecovery` | `boolean`                           | बंडल किए गए Plugin के पुनः इंस्टॉल प्रवाहों को विशिष्ट पुराने-कॉन्फ़िगरेशन विफलताओं से उबरने देता है।  |
| `requiredPlatformPackages`   | `string[]`                          | npm इंस्टॉल के दौरान सत्यापित किए जाने वाले आवश्यक प्लेटफ़ॉर्म-विशिष्ट npm उपनाम।               |

<AccordionGroup>
  <Accordion title="ऑनबोर्डिंग व्यवहार">
    इंटरैक्टिव ऑनबोर्डिंग माँग पर इंस्टॉल सतहों के लिए `openclaw.install` का उपयोग करती है: यदि आपका Plugin रनटाइम लोड होने से पहले प्रदाता प्रमाणीकरण विकल्प या चैनल सेटअप/कैटलॉग मेटाडेटा उजागर करता है, तो ऑनबोर्डिंग ClawHub, npm या स्थानीय इंस्टॉल के लिए संकेत दे सकती है, Plugin को इंस्टॉल या सक्षम कर सकती है और फिर चयनित प्रवाह जारी रख सकती है। ClawHub विकल्प `clawhubSpec` का उपयोग करते हैं और मौजूद होने पर उन्हें प्राथमिकता दी जाती है; npm विकल्पों के लिए रजिस्ट्री `npmSpec` वाला विश्वसनीय कैटलॉग मेटाडेटा आवश्यक है (सटीक संस्करण और `expectedIntegrity` वैकल्पिक पिन हैं, जिन्हें सेट किए जाने पर इंस्टॉल/अपडेट के दौरान लागू किया जाता है)। "क्या दिखाना है" को `openclaw.plugin.json` में और "इसे कैसे इंस्टॉल करना है" को `package.json` में रखें।
  </Accordion>
  <Accordion title="minHostVersion का प्रवर्तन">
    यदि `minHostVersion` सेट है, तो इंस्टॉल और गैर-बंडल मैनिफ़ेस्ट-रजिस्ट्री लोडिंग दोनों इसे लागू करते हैं। पुराने होस्ट बाहरी Plugin छोड़ देते हैं; अमान्य संस्करण स्ट्रिंग अस्वीकार कर दी जाती हैं। बंडल किए गए स्रोत Plugin को होस्ट चेकआउट के समान संस्करण वाला माना जाता है।
  </Accordion>
  <Accordion title="पिन किए गए npm इंस्टॉल">
    पिन किए गए npm इंस्टॉल के लिए सटीक संस्करण `npmSpec` में रखें और अपेक्षित आर्टिफ़ैक्ट अखंडता जोड़ें:

    ```json
    {
      "openclaw": {
        "install": {
          "npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3",
          "expectedIntegrity": "sha512-REPLACE_WITH_NPM_DIST_INTEGRITY",
          "defaultChoice": "npm"
        }
      }
    }
    ```

  </Accordion>
  <Accordion title="allowInvalidConfigRecovery का दायरा">
    `allowInvalidConfigRecovery` खराब कॉन्फ़िगरेशन के लिए सामान्य बायपास नहीं है। यह केवल सीमित बंडल किए गए Plugin की पुनर्प्राप्ति के लिए है, जो पुनः इंस्टॉल/सेटअप को उसी Plugin के अनुपलब्ध बंडल किए गए Plugin पथ या पुराने `channels.<id>` प्रविष्टि जैसे ज्ञात अपग्रेड अवशेषों की मरम्मत करने देता है। यदि कॉन्फ़िगरेशन असंबंधित कारणों से खराब है, तो इंस्टॉल अब भी बंद अवस्था में विफल होता है और ऑपरेटर को `openclaw doctor --fix` चलाने के लिए कहता है।
  </Accordion>
</AccordionGroup>

### स्थगित पूर्ण लोड

चैनल Plugin निम्न के साथ स्थगित लोडिंग चुन सकते हैं:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

सक्षम होने पर, OpenClaw सुनना शुरू करने से पहले के स्टार्टअप चरण में केवल `setupEntry` लोड करता है, यहाँ तक कि पहले से कॉन्फ़िगर किए गए चैनलों के लिए भी। Gateway के सुनना शुरू करने के बाद पूर्ण प्रविष्टि लोड होती है।

<Warning>
स्थगित लोडिंग केवल तभी सक्षम करें जब आपका `setupEntry` Gateway के सुनना शुरू करने से पहले उसकी आवश्यक हर चीज़ पंजीकृत करता हो (चैनल पंजीकरण, HTTP रूट, Gateway विधियाँ)। यदि आवश्यक स्टार्टअप क्षमताएँ पूर्ण प्रविष्टि के स्वामित्व में हैं, तो डिफ़ॉल्ट व्यवहार बनाए रखें।
</Warning>

यदि आपकी सेटअप/पूर्ण प्रविष्टि Gateway RPC विधियाँ पंजीकृत करती है, तो उन्हें Plugin-विशिष्ट उपसर्ग पर रखें। आरक्षित कोर व्यवस्थापक नेमस्पेस (`config.*`, `exec.approvals.*`, `wizard.*`, `update.*`) कोर के स्वामित्व में रहते हैं और हमेशा `operator.admin` में सामान्यीकृत होते हैं।

## Plugin मैनिफ़ेस्ट

प्रत्येक नेटिव Plugin को पैकेज रूट में `openclaw.plugin.json` देना आवश्यक है। OpenClaw इसका उपयोग Plugin कोड निष्पादित किए बिना कॉन्फ़िगरेशन सत्यापित करने के लिए करता है।

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds My Plugin capabilities to OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook verification secret"
      }
    }
  }
}
```

चैनल Plugin के लिए `channels` जोड़ें (और प्रदाता Plugin `providers` जोड़ते हैं):

```json
{
  "id": "my-channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

बिना कॉन्फ़िगरेशन वाले Plugin को भी स्कीमा देना आवश्यक है। खाली स्कीमा मान्य है:

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

पूर्ण स्कीमा संदर्भ के लिए [Plugin मैनिफ़ेस्ट](/hi/plugins/manifest) देखें।

## ClawHub पर प्रकाशन

Skills और Plugin पैकेज अलग-अलग ClawHub प्रकाशन कमांड का उपयोग करते हैं। Plugin पैकेज के लिए पैकेज-विशिष्ट कमांड का उपयोग करें:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

<Note>
`clawhub skill publish <path>` किसी Skill फ़ोल्डर को प्रकाशित करने के लिए अलग कमांड है, Plugin पैकेज के लिए नहीं। [ClawHub पर प्रकाशन](/hi/clawhub/publishing) देखें।
</Note>

## सेटअप प्रविष्टि

`setup-entry.ts`, `index.ts` का एक हल्का विकल्प है, जिसे OpenClaw तब लोड करता है जब उसे केवल सेटअप सतहों की आवश्यकता होती है (ऑनबोर्डिंग, कॉन्फ़िगरेशन मरम्मत, अक्षम चैनल निरीक्षण):

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

यह सेटअप प्रवाहों के दौरान भारी रनटाइम कोड (क्रिप्टो लाइब्रेरी, CLI पंजीकरण, पृष्ठभूमि सेवाएँ) लोड करने से बचाता है।

साइडकार मॉड्यूल में सेटअप-सुरक्षित निर्यात रखने वाले बंडल किए गए कार्यस्थान चैनल `defineSetupPluginEntry(...)` के बजाय `openclaw/plugin-sdk/channel-entry-contract` से `defineBundledChannelSetupEntry(...)` का उपयोग कर सकते हैं। वह बंडल किया गया अनुबंध वैकल्पिक `runtime` निर्यात का भी समर्थन करता है, ताकि सेटअप-समय की रनटाइम वायरिंग हल्की और स्पष्ट बनी रह सके।

<AccordionGroup>
  <Accordion title="OpenClaw पूर्ण प्रविष्टि के बजाय setupEntry का उपयोग कब करता है">
    - चैनल अक्षम है, लेकिन उसे सेटअप/ऑनबोर्डिंग सतहों की आवश्यकता है।
    - चैनल सक्षम है, लेकिन कॉन्फ़िगर नहीं है।
    - स्थगित लोडिंग सक्षम है (`deferConfiguredChannelFullLoadUntilAfterListen`)।

  </Accordion>
  <Accordion title="setupEntry को क्या पंजीकृत करना आवश्यक है">
    - चैनल Plugin ऑब्जेक्ट (`defineSetupPluginEntry` के माध्यम से)।
    - Gateway के सुनने से पहले आवश्यक कोई भी HTTP रूट।
    - स्टार्टअप के दौरान आवश्यक कोई भी Gateway विधि।

    उन स्टार्टअप Gateway विधियों को अब भी `config.*` या `update.*` जैसे आरक्षित कोर व्यवस्थापक नेमस्पेस से बचना चाहिए।

  </Accordion>
  <Accordion title="setupEntry में क्या शामिल नहीं होना चाहिए">
    - CLI पंजीकरण।
    - पृष्ठभूमि सेवाएँ।
    - भारी रनटाइम आयात (क्रिप्टो, SDK)।
    - केवल स्टार्टअप के बाद आवश्यक Gateway विधियाँ।

  </Accordion>
</AccordionGroup>

### सीमित सेटअप सहायक आयात

तेज़ सेटअप-मात्र पथों के लिए, जब आपको सेटअप सतह के केवल एक भाग की आवश्यकता हो, तो व्यापक `plugin-sdk/setup` अम्ब्रेला के बजाय सीमित सेटअप सहायक सीम को प्राथमिकता दें:

| आयात पथ                | इसका उपयोग करें                                                                                | प्रमुख निर्यात                                                                                                                                                                                                                                                                                                           |
| -------------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime` | सेटअप-समय के रनटाइम सहायक, जो `setupEntry` / स्थगित चैनल स्टार्टअप में उपलब्ध रहते हैं | `createSetupTranslator`, `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-tools`   | सेटअप/इंस्टॉल CLI/आर्काइव/दस्तावेज़ सहायक                                                    | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`                                                                                                                                                                                                         |

जब आपको `moveSingleAccountChannelSectionToDefaultAccount(...)` जैसे कॉन्फ़िगरेशन-पैच सहायकों सहित पूर्ण साझा सेटअप टूलबॉक्स चाहिए, तो व्यापक `plugin-sdk/setup` सीम का उपयोग करें।

स्थिर सेटअप विज़ार्ड पाठ के लिए `createSetupTranslator(...)` का उपयोग करें। यह इसी क्रम में `OPENCLAW_LOCALE`, `LC_ALL`, `LC_MESSAGES` और `LANG` से पहला गैर-रिक्त मान उपयोग करता है, फिर अंग्रेज़ी पर वापस जाता है। स्पष्ट अंग्रेज़ी ओवरराइड के लिए `OPENCLAW_LOCALE=en` सेट करें। Plugin-विशिष्ट सेटअप पाठ को Plugin-स्वामित्व वाले कोड में रखें और साझा कैटलॉग कुंजियों का उपयोग केवल सामान्य सेटअप लेबल, स्थिति पाठ और आधिकारिक बंडल किए गए Plugin के सेटअप पाठ के लिए करें।

सेटअप पैच अडैप्टर आयात के समय हॉट-पाथ सुरक्षित रहते हैं। उनकी बंडल की गई एकल-अकाउंट प्रोमोशन अनुबंध-सतह लुकअप आलसी है, इसलिए `plugin-sdk/setup-runtime` को आयात करने से अडैप्टर के वास्तव में उपयोग किए जाने से पहले बंडल की गई अनुबंध-सतह खोज तुरंत लोड नहीं होती।

### चैनल-स्वामित्व वाले सेटअप इनपुट फ़ील्ड

`ChannelSetupInput` सेटअप कॉलर और चैनल
Plugin द्वारा साझा किया जाने वाला एक सामान्य आवरण है। इसके स्थायी रूप से टाइप किए गए फ़ील्ड `name`, `token`, `tokenFile`,
`useEnv`, `allowFrom` और `defaultTo` हैं। अतिरिक्त Plugin-स्वामित्व वाली कुंजियाँ अब भी
रनटाइम इनपुट ऑब्जेक्ट पर मौजूद हो सकती हैं, लेकिन साझा प्रकार किसी
इंडेक्स सिग्नेचर को घोषित नहीं करता। प्रत्येक Plugin को अपने सेटअप फ़ील्ड घोषित करके सीमित करना या
अडैप्टर सीमा पर Plugin-स्वामित्व वाले स्कीमा से उन्हें सत्यापित करना आवश्यक है:

```typescript
import type { ChannelSetupAdapter, ChannelSetupInput } from "openclaw/plugin-sdk/channel-setup";

type AcmeSetupInput = ChannelSetupInput & {
  workspaceId?: string;
  webhookUrl?: string;
};

export const acmeSetupAdapter: ChannelSetupAdapter = {
  applyAccountConfig: ({ cfg, input }) => {
    const setupInput = input as AcmeSetupInput;
    return {
      ...cfg,
      channels: {
        ...cfg.channels,
        acme: {
          token: setupInput.token,
          workspaceId: setupInput.workspaceId,
          webhookUrl: setupInput.webhookUrl,
        },
      },
    };
  },
};
```

चैनल-विशिष्ट फ़ील्ड, जिन्हें पहले सीधे
`ChannelSetupInput` पर घोषित किया गया था, बाहरी स्रोत संगतता के लिए अस्थायी रूप से टाइप किए हुए हैं।
वे बहिष्कृत हैं। 2026-07-22 को 426 प्रकाशित आउट-ऑफ़-ट्री
चैनल plugins की रजिस्ट्री जाँच में ऐसे 21 फ़ील्ड हटा दिए गए जिनके कोई रीडर नहीं थे और ज्ञात
रीडर वाले 22 फ़ील्ड बनाए रखे गए। जैसे ही कोई प्रकाशित plugin किसी बनाए रखे गए फ़ील्ड को पढ़ना बंद करता है, उसे हटा दिया जाता है;
किसी संस्करण सीमा की आवश्यकता नहीं है। नए और बंडल किए गए plugins को इस
स्तर पर निर्भर नहीं होना चाहिए; उनके स्वामित्व वाले फ़ील्ड स्थानीय रूप से घोषित करें।

### चैनल-स्वामित्व वाले एकल खाते का प्रोमोशन

जब कोई चैनल एकल-खाता शीर्ष-स्तरीय कॉन्फ़िगरेशन से `channels.<id>.accounts.*` में अपग्रेड होता है, तो डिफ़ॉल्ट साझा व्यवहार प्रोमोट किए गए खाता-स्कोप वाले मानों को `accounts.default` में ले जाता है।

प्रत्येक चैनल plugin अपने सेटअप अडैप्टर के माध्यम से उस प्रोमोशन को विस्तृत या सीमित कर सकता है:

- `singleAccountKeysToMove`: अतिरिक्त शीर्ष-स्तरीय कुंजियाँ जिन्हें प्रोमोट किए गए खाते में ले जाना चाहिए
- `namedAccountPromotionKeys`: जब नामित खाते पहले से मौजूद हों, तो केवल ये कुंजियाँ प्रोमोट किए गए खाते में जाती हैं; साझा नीति/डिलीवरी कुंजियाँ चैनल रूट पर रहती हैं
- `resolveSingleAccountPromotionTarget(...)`: चुनें कि कौन-सा मौजूदा खाता प्रोमोट किए गए मान प्राप्त करेगा

`singleAccountKeysToMove` की उपस्थिति दर्शाती है कि प्रोमोशन अनुबंध पूरा हो गया है। लेगेसी कुंजी प्रोमोशन से बाहर रहने के लिए फ़ील्ड को खाली ऐरे होने पर भी घोषित करें। फ़ील्ड छोड़ने वाले अडैप्टर पहले से प्रकाशित plugins के लिए रीडर-समर्थित पूर्व-घोषणा प्रोमोशन स्तर बनाए रखते हैं। 2026-07-22 की रजिस्ट्री जाँच में ऐसे 23 कुंजियाँ हटा दी गईं जिनके कोई प्रकाशित आश्रित नहीं थे और छह सामान्य कुंजियों के साथ केवल सेटअप वाली `rooms` कुंजी बनाए रखी गई। जैसे ही किसी बनाए रखी गई कुंजी के प्रकाशित रीडर घोषणाओं पर माइग्रेट होते हैं, उसे हटा दिया जाता है; किसी संस्करण सीमा की आवश्यकता नहीं है।

जब doctor को हल्के बंडल किए गए सेटअप आर्टिफ़ैक्ट से ये घोषणाएँ लोड करनी हों, तो plugin पैकेज मेनिफ़ेस्ट में `openclaw.setupFeatures.configPromotion: true` घोषित करें। केवल सेटअप वाली plugin सतह और पूर्ण चैनल plugin को समान घोषणाएँ उजागर करनी चाहिए।

पहले से रिज़ॉल्व किए गए plugin के साथ `moveSingleAccountChannelSectionToDefaultAccount(...)` को कॉल करते समय, उसके सेटअप अडैप्टर को `setupSurface` के रूप में पास करें। कॉलर द्वारा दी गई सेटअप सतहों को लोड किए गए और बंडल किए गए लुकअप पर प्राथमिकता मिलती है, जिससे स्कोप किए गए या केवल सेटअप वाले plugins वैश्विक पंजीकरण से स्वतंत्र रहते हैं।

<Note>
Matrix वर्तमान बंडल किया गया उदाहरण है। यदि ठीक एक नामित Matrix खाता पहले से मौजूद है, या यदि `defaultAccount` किसी मौजूदा गैर-कैनोनिकल कुंजी, जैसे `Ops`, की ओर संकेत करता है, तो प्रोमोशन नई `accounts.default` प्रविष्टि बनाने के बजाय उस खाते को सुरक्षित रखता है।
</Note>

## कॉन्फ़िगरेशन स्कीमा

Plugin कॉन्फ़िगरेशन को आपके मेनिफ़ेस्ट के JSON Schema के विरुद्ध सत्यापित किया जाता है। उपयोगकर्ता plugins को इस प्रकार कॉन्फ़िगर करते हैं:

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

पंजीकरण के दौरान आपके plugin को यह कॉन्फ़िगरेशन `api.pluginConfig` के रूप में मिलता है।

चैनल-विशिष्ट कॉन्फ़िगरेशन के लिए इसके बजाय चैनल कॉन्फ़िगरेशन अनुभाग का उपयोग करें:

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### चैनल कॉन्फ़िगरेशन स्कीमा बनाना

Zod स्कीमा को plugin-स्वामित्व वाले कॉन्फ़िगरेशन आर्टिफ़ैक्ट द्वारा उपयोग किए जाने वाले `ChannelConfigSchema` रैपर में बदलने के लिए `buildChannelConfigSchema` का उपयोग करें:

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

यदि आप अनुबंध पहले से JSON Schema या TypeBox के रूप में लिखते हैं, तो प्रत्यक्ष सहायक का उपयोग करें ताकि OpenClaw मेटाडेटा पथों पर Zod-से-JSON-Schema रूपांतरण छोड़ सके:

```typescript
import { Type } from "typebox";
import { buildJsonChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const configSchema = buildJsonChannelConfigSchema(
  Type.Object({
    token: Type.Optional(Type.String()),
    allowFrom: Type.Optional(Type.Array(Type.String())),
  }),
);
```

तृतीय-पक्ष plugins के लिए, कोल्ड-पाथ अनुबंध अब भी plugin मेनिफ़ेस्ट है: जनरेट किए गए JSON Schema को `openclaw.plugin.json#channelConfigs` में प्रतिबिंबित करें ताकि कॉन्फ़िगरेशन स्कीमा, सेटअप और UI सतहें रनटाइम कोड लोड किए बिना `channels.<id>` का निरीक्षण कर सकें।

## सेटअप विज़ार्ड

चैनल plugins `openclaw onboard` के लिए इंटरैक्टिव सेटअप विज़ार्ड प्रदान कर सकते हैं। विज़ार्ड, `ChannelPlugin` पर एक `ChannelSetupWizard` ऑब्जेक्ट होता है:

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

`ChannelSetupWizard` में `textInputs`, `dmPolicy`, `allowFrom`, `groupAccess`, `prepare`, `finalize` और अन्य भी समर्थित हैं। पूर्ण बंडल किए गए उदाहरण के लिए Discord plugin का `src/setup-core.ts` देखें।

<AccordionGroup>
  <Accordion title="साझा allowFrom प्रॉम्प्ट">
    उन DM अनुमतिसूची प्रॉम्प्ट के लिए जिन्हें केवल मानक `note -> prompt -> parse -> merge -> patch` प्रवाह की आवश्यकता है, `openclaw/plugin-sdk/setup` से साझा सेटअप सहायकों को प्राथमिकता दें: `createPromptParsedAllowFromForAccount(...)` और `createTopLevelChannelParsedAllowFromPrompt(...)`।
  </Accordion>
  <Accordion title="मानक चैनल सेटअप स्थिति">
    उन चैनल सेटअप स्थिति ब्लॉक के लिए जिनमें केवल लेबल, स्कोर और वैकल्पिक अतिरिक्त पंक्तियाँ अलग होती हैं, प्रत्येक plugin में समान `status` ऑब्जेक्ट स्वयं बनाने के बजाय `openclaw/plugin-sdk/setup` से `createStandardChannelSetupStatus(...)` को प्राथमिकता दें।
  </Accordion>
  <Accordion title="वैकल्पिक चैनल सेटअप सतह">
    उन वैकल्पिक सेटअप सतहों के लिए जिन्हें केवल कुछ संदर्भों में दिखाई देना चाहिए, `openclaw/plugin-sdk/channel-setup` से `createOptionalChannelSetupSurface` का उपयोग करें:

    ```typescript
    import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

    const setupSurface = createOptionalChannelSetupSurface({
      channel: "my-channel",
      label: "My Channel",
      npmSpec: "@myorg/openclaw-my-channel",
      docsPath: "/channels/my-channel",
    });
    // Returns { setupAdapter, setupWizard }
    ```

    जब आपको उस वैकल्पिक-इंस्टॉल सतह के केवल एक भाग की आवश्यकता हो, तो `plugin-sdk/channel-setup` निम्न-स्तरीय `createOptionalChannelSetupAdapter(...)` और `createOptionalChannelSetupWizard(...)` बिल्डर भी उजागर करता है।

    जनरेट किए गए वैकल्पिक अडैप्टर/विज़ार्ड वास्तविक कॉन्फ़िगरेशन लेखन पर फ़ेल-क्लोज़ होते हैं। वे `validateInput`, `applyAccountConfig` और `finalize` में एक ही इंस्टॉल-आवश्यक संदेश का पुनः उपयोग करते हैं और `docsPath` सेट होने पर दस्तावेज़ लिंक जोड़ते हैं।

  </Accordion>
  <Accordion title="बाइनरी-समर्थित सेटअप सहायक">
    बाइनरी-समर्थित सेटअप UI के लिए, प्रत्येक चैनल में समान बाइनरी/स्थिति संयोजन की प्रतिलिपि बनाने के बजाय साझा प्रत्यायोजित सहायकों को प्राथमिकता दें:

    - `createDetectedBinaryStatus(...)` उन स्थिति ब्लॉक के लिए जिनमें केवल लेबल, संकेत, स्कोर और बाइनरी पहचान अलग होती है
    - `createCliPathTextInput(...)` पथ-समर्थित टेक्स्ट इनपुट के लिए
    - `createDelegatedSetupWizardProxy(...)` जब `setupEntry` को स्थिति, तैयारी या अंतिम रूप देने का व्यवहार अधिक भारी पूर्ण विज़ार्ड को आलस्यपूर्वक अग्रेषित करना हो
    - `createDelegatedTextInputShouldPrompt(...)` जब `setupEntry` को केवल `textInputs[*].shouldPrompt` निर्णय प्रत्यायोजित करना हो

  </Accordion>
</AccordionGroup>

## प्रकाशन और इंस्टॉल करना

**बाहरी plugins:** [ClawHub](/hi/clawhub) पर प्रकाशित करें, फिर इंस्टॉल करें:

<Tabs>
  <Tab title="npm">
    ```bash
    openclaw plugins install @myorg/openclaw-my-plugin
    ```

    लॉन्च बदलाव के दौरान साधारण पैकेज विनिर्देश npm से इंस्टॉल होते हैं, जब तक नाम किसी बंडल किए गए या आधिकारिक plugin आईडी से मेल न खाता हो; उस स्थिति में OpenClaw इसके बजाय उस स्थानीय/आधिकारिक प्रति का उपयोग करता है। निर्धारक स्रोत चयन के लिए `clawhub:`, `npm:`, `git:` या `npm-pack:` का उपयोग करें — [plugins प्रबंधित करें](/hi/plugins/manage-plugins) देखें।

  </Tab>
  <Tab title="केवल ClawHub">
    ```bash
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```
  </Tab>
  <Tab title="npm पैकेज विनिर्देश">
    जब कोई पैकेज अभी तक ClawHub पर नहीं गया हो, या माइग्रेशन के दौरान आपको
    प्रत्यक्ष npm इंस्टॉल पथ की आवश्यकता हो, तब npm का उपयोग करें:

    ```bash
    openclaw plugins install npm:@myorg/openclaw-my-plugin
    ```

  </Tab>
</Tabs>

**रिपॉज़िटरी के भीतर के plugins:** इन्हें बंडल किए गए plugin वर्कस्पेस ट्री के अंतर्गत रखें; बिल्ड के दौरान इनकी स्वचालित रूप से खोज की जाती है।

<Info>
npm-स्रोत वाले इंस्टॉल के लिए, `openclaw plugins install` पैकेज को `~/.openclaw/npm/projects` के अंतर्गत प्रति-plugin प्रोजेक्ट में लाइफ़साइकल स्क्रिप्ट अक्षम करके (`--ignore-scripts`) इंस्टॉल करता है। Plugin निर्भरता ट्री को शुद्ध JS/TS रखें और ऐसे पैकेजों से बचें जिन्हें `postinstall` बिल्ड की आवश्यकता होती है।
</Info>

<Note>
Gateway स्टार्टअप plugin निर्भरताएँ इंस्टॉल नहीं करता। npm/git/ClawHub इंस्टॉल प्रवाह निर्भरता अभिसरण के स्वामी हैं; स्थानीय plugins की निर्भरताएँ पहले से इंस्टॉल होनी चाहिए।
</Note>

बंडल किया गया पैकेज मेटाडेटा स्पष्ट होता है, Gateway स्टार्टअप पर बिल्ड किए गए JavaScript से अनुमानित नहीं। रनटाइम निर्भरताएँ उनका स्वामित्व रखने वाले plugin पैकेज में होती हैं; पैकेज किया गया OpenClaw स्टार्टअप कभी भी plugin निर्भरताओं की मरम्मत या प्रतिलिपि नहीं करता।

## संबंधित

- [Plugins बनाना](/hi/plugins/building-plugins) — आरंभ करने की चरण-दर-चरण मार्गदर्शिका
- [Plugin मेनिफ़ेस्ट](/hi/plugins/manifest) — पूर्ण मेनिफ़ेस्ट स्कीमा संदर्भ
- [SDK प्रवेश बिंदु](/hi/plugins/sdk-entrypoints) — `definePluginEntry` और `defineChannelPluginEntry`
