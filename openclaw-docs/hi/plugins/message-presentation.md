---
read_when:
    - संदेश कार्ड, चार्ट, तालिका, बटन या चयन रेंडरिंग जोड़ना या संशोधित करना
    - समृद्ध आउटबाउंड संदेशों का समर्थन करने वाला चैनल Plugin बनाना
    - संदेश टूल की प्रस्तुति या डिलीवरी क्षमताएँ बदलना
    - प्रदाता-विशिष्ट कार्ड/ब्लॉक/घटक रेंडरिंग रिग्रेशन की डीबगिंग
summary: चैनल plugins के लिए सिमैंटिक संदेश कार्ड, चार्ट, तालिकाएँ, नियंत्रण, फ़ॉलबैक टेक्स्ट और डिलीवरी संकेत
title: संदेश प्रस्तुति
x-i18n:
    generated_at: "2026-07-27T20:12:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fce3874c99627eb87ceb83aebe381b8a8466722703ec6322c609f187d15d9ae
    source_path: plugins/message-presentation.md
    workflow: 16
---

संदेश प्रस्तुति समृद्ध आउटबाउंड चैट UI के लिए OpenClaw का साझा अनुबंध है।
यह एजेंटों, CLI कमांडों, अनुमोदन प्रवाहों और plugins को संदेश का आशय
एक बार वर्णित करने देता है, जबकि प्रत्येक चैनल plugin अपनी क्षमता के अनुसार सर्वोत्तम नेटिव स्वरूप प्रस्तुत करता है।

पोर्टेबल संदेश UI के लिए प्रस्तुति का उपयोग करें: टेक्स्ट अनुभाग, छोटा संदर्भ/फुटर
टेक्स्ट, विभाजक, चार्ट, तालिकाएँ, बटन, चयन मेनू और कार्ड शीर्षक/टोन।

साझा संदेश टूल में Discord `components`, Slack
`blocks`, Telegram `buttons`, Teams `card`, या Feishu `card` जैसे नए प्रदाता-नेटिव फ़ील्ड न जोड़ें।
वे चैनल plugin के स्वामित्व वाले रेंडरर आउटपुट हैं।

## अनुबंध

Plugin लेखक सार्वजनिक अनुबंध को यहाँ से इंपोर्ट करते हैं:

```ts
import type {
  MessagePresentation,
  ReplyPayloadDelivery,
} from "openclaw/plugin-sdk/interactive-runtime";
```

संरचना:

```ts
type MessagePresentation = {
  title?: string;
  tone?: "neutral" | "info" | "success" | "warning" | "danger";
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] }
  | {
      type: "chart";
      chartType: "pie";
      title: string;
      segments: Array<{ label: string; value: number }>;
    }
  | {
      type: "chart";
      chartType: "bar" | "area" | "line";
      title: string;
      categories: string[];
      series: Array<{ name: string; values: number[] }>;
      xLabel?: string;
      yLabel?: string;
    }
  | {
      type: "table";
      caption: string;
      headers: string[];
      rows: Array<Array<string | number>>;
      rowHeaderColumnIndex?: number;
    };

type MessagePresentationAction =
  | { type: "command"; command: string }
  | { type: "callback"; value: string }
  | {
      type: "approval";
      approvalId: string;
      approvalKind: "exec" | "plugin";
      decision: "allow-once" | "allow-always" | "deny";
    }
  | {
      type: "question";
      questionId: string;
      optionValue: string;
    }
  | { type: "url"; url: string }
  | {
      type: "web-app";
      url: string;
      widgetId?: string;
    }
  | {
      type: "web-app";
      url?: string;
      widgetId: string;
    };

type MessagePresentationButton = {
  label: string;
  action?: MessagePresentationAction;
  /** पुराना कॉलबैक मान। नए नियंत्रणों के लिए action को प्राथमिकता दें। */
  value?: string;
  /** @deprecated "url" प्रकार वाली action का उपयोग करें। */
  url?: string;
  /** @deprecated "web-app" प्रकार वाली action का उपयोग करें। */
  webApp?: { url: string };
  /** @deprecated "web-app" प्रकार वाली action का उपयोग करें। */
  web_app?: { url: string };
  priority?: number;
  disabled?: boolean;
  reusable?: boolean;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  action?: Extract<MessagePresentationAction, { type: "command" | "callback" }>;
  /** पुराना कॉलबैक मान। नए नियंत्रणों के लिए action को प्राथमिकता दें। */
  value?: string;
};

type ReplyPayloadDelivery = {
  pin?:
    | boolean
    | {
        enabled: boolean;
        notify?: boolean;
        required?: boolean;
      };
};
```

बटन का अर्थ-विज्ञान:

- `action.type: "command"` core के कमांड पथ के माध्यम से एक नेटिव स्लैश कमांड चलाता है।
इसका उपयोग बिल्ट-इन कमांड बटनों और मेनू के लिए करें।
- `action.type: "callback"` चैनल के इंटरैक्शन पथ के माध्यम से अपारदर्शी plugin डेटा ले जाता है।
चैनल plugins को कॉलबैक डेटा की दोबारा व्याख्या स्लैश कमांड के रूप में नहीं करनी चाहिए।
- `action.type: "approval"` एक स्थायी ऑपरेटर अनुमोदन, उसके स्पष्ट
`exec` या `plugin` प्रकार और अनुरोधित निर्णय की पहचान करता है। चैनल plugins
उस action को ट्रांसपोर्ट-निजी कॉलबैक में एन्कोड करके अनुमोदन सेवा के माध्यम से
समाधान करते हैं; उन्हें `/approve` कमांड टेक्स्ट पार्स नहीं करना चाहिए या ID से
प्रकार का अनुमान नहीं लगाना चाहिए।
- `action.type: "question"` किसी लाइव, रनटाइम-निर्मित
`ask_user` प्रश्न के एक विकल्प की पहचान करता है। `approval` की तरह, यह OpenClaw रनटाइम action है;
एजेंटों और plugins को प्रश्न ID स्वयं नहीं बनाने चाहिए। Telegram, Discord और
Slack इसे ट्रांसपोर्ट-निजी नेटिव कॉलबैक में मैप करते हैं और विकल्प का समाधान
Gateway के माध्यम से करते हैं। जब प्रश्न का उत्तर दिया जा चुका हो, उसकी अवधि समाप्त हो जाए या
उसे रद्द कर दिया जाए, तो वे चैनल डिलीवर किए गए संदेश को संपादित करते हैं, उसकी actions हटाते हैं
और अंतिम स्थिति जोड़ते हैं। WhatsApp, Signal और iMessage अधिकतम
चार एकल-चयन विकल्पों को `1️⃣` से `4️⃣` प्रतिक्रियाओं के रूप में प्रस्तुत करते हैं। अन्य प्रश्न
संरचनाएँ लेबल टेक्स्ट में अवनत हो जाती हैं और उपयोगकर्ता सादे टेक्स्ट
उत्तर से जवाब दे सकता है।
- `action.type: "url"` एक सामान्य लिंक खोलता है।
- `action.type: "web-app"` चैनल-नेटिव वेब ऐप लॉन्च करता है। URL-आधारित ऐप के लिए `url`
या OpenClaw द्वारा होस्ट किए गए ऐसे विजेट के लिए `widgetId` सेट करें जिसकी लॉन्च
प्रक्रिया चैनल के स्वामित्व में हो; इनमें से कम-से-कम एक आवश्यक है। जब दोनों
मौजूद हों, तो चैनल अपने नेटिव होस्टेड-विजेट लॉन्च को प्राथमिकता दे सकता है और जहाँ
वह तंत्र उपलब्ध न हो वहाँ URL का उपयोग कर सकता है।
- `value` पुराना अपारदर्शी कॉलबैक मान है। नए नियंत्रणों को `action` का उपयोग करना चाहिए
ताकि चैनल plugins टेक्स्ट से अनुमान लगाए बिना कमांड और कॉलबैक मैप कर सकें।
- `url`, `webApp` और `web_app` को अप्रचलित सीमा इनपुट के रूप में अब भी स्वीकार किया जाता है।
नॉर्मलाइज़र इन फ़ील्ड को सुरक्षित रखते हैं ताकि रेंडरर जारी किए जा चुके पुराने
अर्थ-विज्ञान को स्पष्ट टाइप की गई actions से अलग कर सकें। नए उत्पादकों को `action` का उपयोग करना चाहिए।
- `label` आवश्यक है और टेक्स्ट फ़ॉलबैक में भी उपयोग किया जाता है।
- `style` परामर्शात्मक है। रेंडरर को असमर्थित शैलियों को सुरक्षित
डिफ़ॉल्ट में मैप करना चाहिए, न कि प्रेषण विफल करना चाहिए।
- `priority` वैकल्पिक है। जब कोई चैनल action सीमाएँ घोषित करता है और नियंत्रणों
को हटाना आवश्यक हो, तो core पहले उच्च-प्राथमिकता वाले बटन रखता है और समान प्राथमिकता वाले
बटनों के बीच मूल क्रम सुरक्षित रखता है। जब सभी नियंत्रण समा जाते हैं, तो लेखकीय
क्रम सुरक्षित रहता है।
- `disabled` वैकल्पिक है। चैनलों को `supportsDisabled` के साथ इसे स्पष्ट रूप से सक्षम करना होगा; अन्यथा
core अक्षम नियंत्रण को गैर-इंटरैक्टिव फ़ॉलबैक टेक्स्ट में अवनत कर देता है। किसी
अक्षम बटन को फ़ॉलबैक टेक्स्ट में हमेशा केवल लेबल के रूप में प्रस्तुत किया जाता है, भले ही उसमें
`command` action हो।
- `reusable` वैकल्पिक है। पुनः उपयोग योग्य नेटिव कॉलबैक का समर्थन करने वाले चैनल
सफल इंटरैक्शन के बाद action को उपलब्ध रख सकते हैं। इसका उपयोग
रीफ़्रेश, निरीक्षण या अधिक विवरण जैसी दोहराने योग्य या आइडेम्पोटेंट actions के लिए करें;
सामान्य एकबारगी अनुमोदनों और विनाशकारी actions के लिए इसे सेट न करें।

चयन का अर्थ-विज्ञान:

- `options[].action` केवल `command` या `callback` स्वीकार करता है; अनुमोदन और लिंक actions केवल बटन के लिए हैं।
- `options[].value` पुराना चयनित एप्लिकेशन मान है।
- `placeholder` परामर्शात्मक है और नेटिव चयन समर्थन के बिना चैनलों द्वारा
अनदेखा किया जा सकता है।
- यदि कोई चैनल चयन का समर्थन नहीं करता, तो फ़ॉलबैक टेक्स्ट लेबलों को सूचीबद्ध करता है।

चार्ट का अर्थ-विज्ञान:

- `pie` के लिए धनात्मक खंड मान आवश्यक हैं।
- `bar`, `area` और `line` एक क्रमबद्ध `categories` ऐरे का उपयोग करते हैं। प्रत्येक शृंखला
उसी क्रम में प्रत्येक श्रेणी के लिए ठीक एक परिमित मान प्रदान करती है।
- श्रेणी लेबल और शृंखला नाम अद्वितीय होने चाहिए। अमान्य या अपूर्ण चार्ट
ब्लॉक डेटा को चुपचाप बदलने के बजाय नॉर्मलाइज़ेशन के दौरान हटा दिए जाते हैं।
- नेटिव चार्ट रेंडरिंग को `presentationCapabilities.charts` के माध्यम से स्पष्ट रूप से सक्षम किया जाता है।
अन्य चैनलों को चार्ट शीर्षक, अक्ष, श्रेणियाँ, शृंखलाएँ और मान
नियतात्मक टेक्स्ट के रूप में मिलते हैं। यह अभिगम्यता फ़ॉलबैक भी है।

तालिका का अर्थ-विज्ञान:

- `caption` एक आवश्यक संक्षिप्त शीर्षक है। `headers` में कम-से-कम एक
अद्वितीय, गैर-रिक्त कॉलम लेबल होना चाहिए।
- `rows` में कम-से-कम एक पंक्ति होनी चाहिए। प्रत्येक पंक्ति में हर
हेडर के लिए ठीक एक सेल होना चाहिए और प्रत्येक सेल एक गैर-रिक्त स्ट्रिंग या परिमित संख्या होनी चाहिए।
- `rowHeaderColumnIndex` एक वैकल्पिक शून्य-आधारित इंडेक्स है, जो उस कॉलम की पहचान करता है
जिसके सेल नेटिव रेंडरर द्वारा पंक्ति हेडर के रूप में प्रदर्शित किए जाने चाहिए।
- तालिका नॉर्मलाइज़ेशन परमाण्विक है। अमान्य कैप्शन, हेडर, पंक्ति की चौड़ाई, सेल
या पंक्ति-हेडर इंडेक्स उसके डेटा को छोटा करने या सुधारने के बजाय
तालिका ब्लॉक को हटा देता है।
- नेटिव तालिका रेंडरिंग को `presentationCapabilities.tables` के माध्यम से स्पष्ट रूप से सक्षम किया जाता है।
अन्य चैनलों को कैप्शन और प्रत्येक पंक्ति नियतात्मक रैखिक
टेक्स्ट के रूप में मिलती है, जिसमें आंतरिक रिक्त स्थान संक्षिप्त कर दिया जाता है:

  ```text
  खुली पाइपलाइन (तालिका)
  - खाता: Acme; चरण: जीता गया; ARR: 125000
  - खाता: Globex; चरण: समीक्षा; ARR: 82000
  ```

कोई अलग `report` डिस्क्रिमिनेटर नहीं है। `title`,
`tone`, `text`, `context`, `chart`, `table` और action ब्लॉक से रिपोर्ट बनाएँ। इससे प्रत्येक
ब्लॉक स्वतंत्र रूप से रेंडर किया जा सकता है और संपूर्ण रिपोर्ट को वही
नियतात्मक टेक्स्ट फ़ॉलबैक मिलता है।

## उत्पादक उदाहरण

सरल कार्ड:

```json
{
  "title": "डिप्लॉयमेंट अनुमोदन",
  "tone": "warning",
  "blocks": [
    { "type": "text", "text": "Canary प्रमोट करने के लिए तैयार है।" },
    { "type": "context", "text": "बिल्ड 1234, स्टेजिंग सफल रही।" },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "अनुमोदित करें",
          "action": { "type": "callback", "value": "deploy:approve" },
          "style": "success"
        },
        {
          "label": "अस्वीकार करें",
          "action": { "type": "callback", "value": "deploy:decline" },
          "style": "danger"
        }
      ]
    }
  ]
}
```

केवल-URL लिंक बटन:

```json
{
  "blocks": [
    { "type": "text", "text": "रिलीज़ नोट्स तैयार हैं।" },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "नोट्स खोलें",
          "action": { "type": "url", "url": "https://example.com/release" }
        }
      ]
    }
  ]
}
```

Telegram Mini App बटन:

```json
{
  "blocks": [
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "लॉन्च करें",
          "action": { "type": "web-app", "url": "https://example.com/app" }
        }
      ]
    }
  ]
}
```

चयन मेनू:

```json
{
  "title": "परिवेश चुनें",
  "blocks": [
    {
      "type": "select",
      "placeholder": "परिवेश",
      "options": [
        { "label": "Canary", "value": "env:canary" },
        { "label": "प्रोडक्शन", "value": "env:prod" }
      ]
    }
  ]
}
```

चार्ट:

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "line",
      "title": "त्रैमासिक राजस्व",
      "categories": ["Q1", "Q2", "Q3"],
      "series": [
        { "name": "उत्पाद", "values": [120, 145, 138] },
        { "name": "सेवाएँ", "values": [80, 95, 104] }
      ],
      "xLabel": "तिमाही",
      "yLabel": "राजस्व"
    }
  ]
}
```

तालिका रिपोर्ट:

```json
{
  "title": "पाइपलाइन रिपोर्ट",
  "tone": "info",
  "blocks": [
    { "type": "text", "text": "चरण के अनुसार वर्तमान अवसर।" },
    {
      "type": "table",
      "caption": "खुली पाइपलाइन",
      "headers": ["खाता", "चरण", "ARR"],
      "rows": [
        ["Acme", "जीता गया", 125000],
        ["Globex", "समीक्षा", 82000]
      ],
      "rowHeaderColumnIndex": 0
    },
    { "type": "context", "text": "CRM स्नैपशॉट से अपडेट किया गया।" }
  ]
}
```

CLI प्रेषण:

```bash
openclaw message send --channel slack \
  --target channel:C123 \
  --message "डिप्लॉयमेंट अनुमोदन" \
  --presentation '{"title":"डिप्लॉयमेंट अनुमोदन","tone":"warning","blocks":[{"type":"text","text":"Canary तैयार है।"},{"type":"buttons","buttons":[{"label":"अनुमोदित करें","value":"deploy:approve","style":"success"},{"label":"अस्वीकार करें","value":"deploy:decline","style":"danger"}]}]}'
```

पिन की गई डिलीवरी:

```bash
openclaw message send --channel telegram \
  --target -1001234567890 \
  --message "विषय खोला गया" \
  --pin
```

स्पष्ट JSON के साथ पिन की गई डिलीवरी:

```json
{
  "pin": {
    "enabled": true,
    "notify": true,
    "required": false
  }
}
```

## रेंडरर अनुबंध

चैनल Plugin अपने आउटबाउंड अडैप्टर पर रेंडर समर्थन घोषित करते हैं:

```ts
const adapter: ChannelOutboundAdapter = {
  deliveryMode: "direct",
  presentationCapabilities: {
    supported: true,
    buttons: true,
    selects: true,
    context: true,
    divider: true,
    charts: false,
    tables: false,
    limits: {
      actions: {
        maxActions: 25,
        maxActionsPerRow: 5,
        maxRows: 5,
        maxLabelLength: 80,
        maxValueBytes: 100,
        supportsStyles: true,
        supportsDisabled: false,
      },
      selects: {
        maxOptions: 25,
        maxLabelLength: 100,
        maxValueBytes: 100,
      },
      text: {
        maxLength: 2000,
        encoding: "characters",
        markdownDialect: "discord-markdown",
      },
    },
  },
  deliveryCapabilities: {
    pin: true,
  },
  renderPresentation({ payload, presentation, ctx }) {
    return renderNativePayload(payload, presentation, ctx);
  },
  async pinDeliveredMessage({ target, messageId, pin }) {
    await pinNativeMessage(target, messageId, { notify: pin.notify === true });
  },
};
```

क्षमता बूलियन बताते हैं कि रेंडरर किन चीज़ों को इंटरैक्टिव बना सकता है। वैकल्पिक
`limits` उस सामान्य एनवेलप का वर्णन करते हैं जिसे कोर रेंडरर को कॉल करने से पहले
अनुकूलित कर सकता है:

```ts
type ChannelPresentationCapabilities = {
  supported?: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  charts?: boolean;
  tables?: boolean;
  limits?: {
    actions?: {
      maxActions?: number;
      maxActionsPerRow?: number;
      maxRows?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
      supportsStyles?: boolean;
      supportsDisabled?: boolean;
      supportsLayoutHints?: boolean;
    };
    selects?: {
      maxOptions?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
    };
    text?: {
      maxLength?: number;
      encoding?: "characters" | "utf8-bytes" | "utf16-units";
      markdownDialect?: "plain" | "markdown" | "html" | "slack-mrkdwn" | "discord-markdown";
      supportsEdit?: boolean;
    };
  };
};
```

कोर रेंडरिंग से पहले सिमेंटिक नियंत्रणों पर सामान्य सीमाएँ लागू करता है। रेंडरर
अब भी नेटिव ब्लॉक संख्या, कार्ड आकार, URL सीमाओं और ऐसी प्रदाता-विशिष्ट
विशिष्टताओं के अंतिम सत्यापन और काट-छाँट के स्वामी हैं जिन्हें सामान्य
अनुबंध में व्यक्त नहीं किया जा सकता। यदि सीमाएँ किसी ब्लॉक से प्रत्येक नियंत्रण हटा देती हैं, तो कोर
लेबल को गैर-इंटरैक्टिव संदर्भ टेक्स्ट के रूप में रखता है, ताकि डिलीवर किए गए संदेश में फिर भी एक
दृश्य फ़ॉलबैक हो।

## कोर रेंडर प्रवाह

CLI और मानक संदेश कार्रवाइयों द्वारा उपयोग किए जाने वाले कैननिकल आउटबाउंड पथ पर, कोर:

1. प्रेज़ेंटेशन पेलोड को सामान्यीकृत करता है।
2. लक्ष्य चैनल के आउटबाउंड अडैप्टर का समाधान करता है।
3. `presentationCapabilities` को पढ़ता है।
4. जब अडैप्टर उन्हें विज्ञापित करता है, तब कार्रवाई संख्या, लेबल लंबाई और
   चयन विकल्प संख्या जैसी सामान्य क्षमता सीमाएँ लागू करता है। चार्ट और तालिका ब्लॉक
   तब तक नियतात्मक टेक्स्ट बन जाते हैं, जब तक अडैप्टर क्रमशः
   `charts: true` या `tables: true` को स्पष्ट रूप से विज्ञापित न करे।
5. जब अडैप्टर पेलोड को रेंडर कर सकता है, तब `renderPresentation` को कॉल करता है।
6. अडैप्टर अनुपस्थित होने या रेंडर न कर पाने पर सुरक्षित टेक्स्ट पर फ़ॉलबैक करता है।
7. परिणामी पेलोड को सामान्य चैनल डिलीवरी पथ से भेजता है।
8. पहला संदेश सफलतापूर्वक भेजे जाने के बाद `delivery.pin` जैसे डिलीवरी
   मेटाडेटा लागू करता है।

`ReplyPayload` का सीधे उपयोग करने वाले चैनल-स्थानीय उत्तर या पूर्वावलोकन फ़नल को
या तो उस कैननिकल पथ में प्रवेश करना चाहिए या पेलोड को सादे टेक्स्ट/मीडिया में
प्रक्षेपित करने से पहले वही प्रेज़ेंटेशन फ़ॉलबैक साकार करना चाहिए।

फ़ॉलबैक व्यवहार का स्वामित्व कोर के पास है, ताकि उत्पादक चैनल-अज्ञेय रह सकें। चैनल
Plugin नेटिव रेंडरिंग और इंटरैक्शन प्रबंधन के स्वामी हैं।

## अवक्रमण नियम

प्रेज़ेंटेशन को सीमित चैनलों पर भेजना सुरक्षित होना चाहिए।

फ़ॉलबैक टेक्स्ट में शामिल हैं:

- पहली पंक्ति के रूप में `title`
- सामान्य अनुच्छेदों के रूप में `text` ब्लॉक
- संक्षिप्त संदर्भ पंक्तियों के रूप में `context` ब्लॉक
- दृश्य विभाजक के रूप में `divider` ब्लॉक
- बटन लेबल, जिनमें लिंक बटन के URL शामिल हैं
- चयन विकल्प लेबल
- चार्ट शीर्षक, प्रकार, अक्ष, श्रेणियाँ, शृंखलाएँ और मान
- तालिका कैप्शन, हेडर और प्रत्येक पंक्ति का मान

### बटन मान की फ़ॉलबैक दृश्यता

जब कोई चैनल इंटरैक्टिव नियंत्रण रेंडर नहीं कर सकता, तो बटन और चयन मान
सादे टेक्स्ट पर फ़ॉलबैक होते हैं। फ़ॉलबैक व्यवहार अपारदर्शी कॉलबैक डेटा को
निजी रखते हुए उपयोगिता बनाए रखता है:

- **`command`-प्रकार की कार्रवाइयाँ** `` label: `command` `` के रूप में रेंडर होती हैं, ताकि उपयोगकर्ता
  कमांड कॉपी करके उसे चैनल इनपुट में मैन्युअल रूप से चला सकें।
- **`callback`-प्रकार की कार्रवाइयाँ** और पुराने **`value`** फ़ील्ड केवल
  लेबल के रूप में रेंडर होते हैं। अपारदर्शी कॉलबैक मान फ़ॉलबैक टेक्स्ट में उजागर नहीं किया जाता।
- **`approval`-प्रकार की कार्रवाइयाँ** केवल लेबल के रूप में रेंडर होती हैं। अनुमोदन ID और निर्णय
  ट्रांसपोर्ट डेटा हैं और सामान्य स्केलर सहायकों या फ़ॉलबैक
  टेक्स्ट के माध्यम से उजागर नहीं किए जाते।
- **`url` कार्रवाइयाँ**, URL-समर्थित **`web-app` कार्रवाइयाँ**, और अप्रचलित **`url` /
  `webApp` / `web_app`** इनपुट बटन लेबल के साथ URL टेक्स्ट रेंडर करते हैं,
  क्योंकि URL उपयोगकर्ता-दृश्य है। केवल होस्ट किए गए विजेट वाली कार्रवाइयाँ उन
  चैनलों पर केवल लेबल के रूप में रेंडर होती हैं जहाँ नेटिव विजेट लॉन्च उपलब्ध नहीं है।
- **चयन विकल्प** केवल लेबल के रूप में रेंडर होते हैं। अंतर्निहित विकल्प मान
  फ़ॉलबैक टेक्स्ट में उजागर नहीं किया जाता।

जो चैनल अडैप्टर अपने फ़ॉलबैक UI में मैन्युअल-कमांड मार्गदर्शन जोड़ते हैं (उदाहरण के लिए,
Feishu दस्तावेज़-टिप्पणी निर्देश), उन्हें कमांड-उपस्थिति जाँच उसी
प्रेज़ेंटेशन ब्लॉक से प्राप्त करनी चाहिए जिसका उपयोग फ़ॉलबैक रेंडरर करता है, ताकि
मार्गदर्शन टेक्स्ट केवल तभी दिखाई दे जब वास्तव में कोई मैन्युअल कमांड दिखाया गया हो।

असमर्थित नेटिव नियंत्रणों को पूरे प्रेषण को विफल करने के बजाय अवक्रमित होना चाहिए।
उदाहरण:

- इनलाइन बटन अक्षम होने पर Telegram टेक्स्ट फ़ॉलबैक भेजता है।
- चयन समर्थन के बिना चैनल चयन विकल्पों को टेक्स्ट के रूप में सूचीबद्ध करता है।
- नेटिव चार्ट समर्थन के बिना चैनल चार्ट डेटा को टेक्स्ट के रूप में सूचीबद्ध करता है।
- नेटिव तालिका समर्थन के बिना चैनल प्रत्येक तालिका पंक्ति को टेक्स्ट के रूप में सूचीबद्ध करता है।
- केवल-URL बटन या तो नेटिव लिंक बटन या फ़ॉलबैक URL पंक्ति बन जाता है।
- वैकल्पिक पिन विफलताएँ डिलीवर किए गए संदेश को विफल नहीं करतीं।

मुख्य अपवाद `delivery.pin.required: true` है; यदि पिन करना
अनिवार्य रूप से अनुरोधित है और चैनल भेजे गए संदेश को पिन नहीं कर सकता, तो डिलीवरी विफलता की रिपोर्ट करती है।

## प्रदाता मैपिंग

वर्तमान बंडल किए गए रेंडरर:

| चैनल          | नेटिव रेंडर लक्ष्य                         | टिप्पणियाँ                                                                                                                                                                                                          |
| --------------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discord         | कंपोनेंट और कंपोनेंट कंटेनर                | मौजूदा प्रदाता-नेटिव पेलोड उत्पादकों के लिए पुराने `channelData.discord.components` को संरक्षित रखता है, लेकिन नए साझा प्रेषणों को `presentation` का उपयोग करना चाहिए।                                                                 |
| Feishu          | इंटरैक्टिव कार्ड                           | कार्ड हेडर `title` का उपयोग कर सकता है; मुख्य भाग उस शीर्षक की पुनरावृत्ति से बचता है।                                                                                                                                                  |
| Matrix          | टेक्स्ट फ़ॉलबैक और संरचित इवेंट फ़ील्ड     | बटन/चयन समर्थित के रूप में विज्ञापित होते हैं, लेकिन प्रत्येक ब्लॉक वर्तमान में नेटिव इंटरैक्टिव विजेट के बजाय `com.openclaw.presentation` इवेंट फ़ील्ड में ले जाए गए `renderMessagePresentationFallbackText` आउटपुट के रूप में रेंडर होता है। |
| Mattermost      | टेक्स्ट और इंटरैक्टिव प्रॉप्स              | चयन और विभाजक समर्थित नहीं हैं; वे ब्लॉक टेक्स्ट में अवक्रमित हो जाते हैं।                                                                                                                                             |
| Microsoft Teams | Adaptive Cards                            | दोनों उपलब्ध होने पर कार्ड के साथ सादा `message` टेक्स्ट शामिल किया जाता है। चयन, शैलियाँ और अक्षम स्थिति समर्थित नहीं हैं।                                                                                     |
| Slack           | Block Kit                                 | `chart` को नेटिव `data_visualization` और `table` को नेटिव `data_table` के रूप में रेंडर करता है; पुराने `channelData.slack.blocks` को संरक्षित रखता है, लेकिन नए साझा प्रेषणों को `presentation` का उपयोग करना चाहिए।                                   |
| Telegram        | टेक्स्ट और इनलाइन कीबोर्ड                  | बटन/चयन को लक्ष्य सतह के लिए इनलाइन बटन क्षमता की आवश्यकता होती है; अन्यथा टेक्स्ट फ़ॉलबैक का उपयोग किया जाता है।                                                                                                         |
| सादे चैनल       | टेक्स्ट फ़ॉलबैक                            | रेंडरर के बिना चैनलों को भी पढ़ने योग्य आउटपुट मिलता है।                                                                                                                                                            |

प्रदाता-नेटिव पेलोड संगतता मौजूदा उत्तर उत्पादकों के लिए एक संक्रमण सुविधा है।
यह नए साझा नेटिव फ़ील्ड जोड़ने का कारण नहीं है।

## प्रेज़ेंटेशन बनाम InteractiveReply

`InteractiveReply` अनुमोदन और इंटरैक्शन सहायकों द्वारा उपयोग किया जाने वाला पुराना आंतरिक उपसमुच्चय है।
यह निम्न का समर्थन करता है:

- टेक्स्ट
- बटन
- चयन

`MessagePresentation` कैननिकल साझा प्रेषण अनुबंध है। यह निम्न जोड़ता है:

- शीर्षक
- लहजा
- संदर्भ
- विभाजक
- चार्ट
- तालिका
- केवल-URL बटन
- `ReplyPayload.delivery` के माध्यम से सामान्य डिलीवरी मेटाडेटा

पुराने कोड को जोड़ते समय `openclaw/plugin-sdk/interactive-runtime` के सहायकों का उपयोग करें:

```ts
import {
  adaptMessagePresentationForChannel,
  applyPresentationActionLimits,
  hasMessagePresentationBlocks,
  interactiveReplyToPresentation,
  isMessagePresentationInteractiveBlock,
  normalizeMessagePresentation,
  presentationPageSize,
  presentationToInteractiveControlsReply,
  presentationToInteractiveReply,
  renderMessagePresentationChartFallbackText,
  renderMessagePresentationFallbackText,
  renderMessagePresentationTableFallbackText,
  resolveMessagePresentationActionValue,
  resolveMessagePresentationButtonAction,
  resolveMessagePresentationControlValue,
  resolveMessagePresentationOptionAction,
} from "openclaw/plugin-sdk/interactive-runtime";
```

नए कोड को सीधे `MessagePresentation` स्वीकार या उत्पन्न करना चाहिए। मौजूदा
`interactive` पेलोड `presentation` का एक अप्रचलित उपसमुच्चय हैं; पुराने
उत्पादकों के लिए रनटाइम समर्थन बना हुआ है।

जानने योग्य गैर-अप्रचलित सहायक:

- `normalizeMessagePresentation(raw)` / `hasMessagePresentationBlocks(value)`
  एक अनटाइप्ड पेलोड (उदाहरण के लिए, CLI के
  `--presentation` फ़्लैग से JSON) को सत्यापित करके `MessagePresentation` में रूपांतरित करते हैं।
- `isMessagePresentationInteractiveBlock(block)` किसी ब्लॉक को
  `buttons` | `select` यूनियन तक सीमित करता है।
- `resolveMessagePresentationButtonAction(button)` और
  `resolveMessagePresentationOptionAction(option)` अप्रचलित सीमा फ़ील्ड स्वीकार करते हुए
  कैनोनिकल टाइप्ड ऐक्शन लौटाते हैं। स्पष्ट `action`
  को हमेशा प्राथमिकता मिलती है।
- `resolveMessagePresentationActionValue(action)` /
  `resolveMessagePresentationControlValue(control)` केवल कमांड/कॉलबैक
  स्केलर मान पढ़ते हैं। कोई गैर-स्केलर कैनोनिकल ऐक्शन कभी भी
  पुराने शैडो `value` पर नहीं जाता, इसलिए अनुमोदन ID और लिंक लक्ष्य टाइप्ड बने रहते हैं।
- `renderMessagePresentationChartFallbackText(block)` /
  `renderMessagePresentationTableFallbackText(block)` चैनल-विशिष्ट फ़ॉलबैक पथों के लिए एक संरचित
  डेटा ब्लॉक को नियतात्मक टेक्स्ट के रूप में रेंडर करते हैं।

पुराने `InteractiveReply*` प्रकारों और रूपांतरण सहायकों को SDK में
`@deprecated` के रूप में चिह्नित किया गया है:

- `InteractiveReply`, `InteractiveReplyBlock`, `InteractiveReplyButton`, और
  `InteractiveReplyOption`
- `normalizeInteractiveReply(...)`
- `hasInteractiveReplyBlocks(...)`
- `interactiveReplyToPresentation(...)`
- `presentationToInteractiveReply(...)`
- `presentationToInteractiveControlsReply(...)`
- `resolveInteractiveTextFallback(...)`
- `reduceInteractiveReply(...)`

`presentationToInteractiveReply(...)` और
`presentationToInteractiveControlsReply(...)` पुराने चैनल कार्यान्वयनों के लिए रेंडरर
ब्रिज के रूप में उपलब्ध रहते हैं। नए प्रोड्यूसर कोड को इन्हें कॉल नहीं करना चाहिए;
`presentation` भेजें और कोर/चैनल अनुकूलन को रेंडरिंग संभालने दें।

अनुमोदन सहायकों के लिए भी प्रस्तुति-प्रथम प्रतिस्थापन उपलब्ध हैं:

- `buildApprovalInteractiveReply(...)` के बजाय
  `buildApprovalPresentation(...)` का उपयोग करें
- `buildExecApprovalInteractiveReply(...)` के बजाय
  `buildExecApprovalPresentation(...)` का उपयोग करें

Plugin संगतता के लिए वे जारी किए गए बिल्डर कमांड-समर्थित बने रहते हैं। स्थायी अनुमोदन प्रकार
का स्वामित्व रखने वाले Gateway और बंडल किए गए चैनल कोड को
`buildTypedApprovalPresentation(...)`,
`buildTypedExecApprovalPendingReplyPayload(...)`, या
`buildTypedPluginApprovalPendingReplyPayload(...)` का उपयोग करना चाहिए, ताकि ट्रांसपोर्ट को `/approve` टेक्स्ट से अर्थ का अनुमान लगाने के बजाय
स्पष्ट `approval` ऐक्शन मिले।

`renderMessagePresentationFallbackText(...)` ऐसे प्रस्तुति ब्लॉक के लिए
खाली स्ट्रिंग लौटाता है जिनका कोई टेक्स्ट फ़ॉलबैक नहीं होता, जैसे केवल-विभाजक
प्रस्तुति। जिन ट्रांसपोर्ट को गैर-रिक्त प्रेषण बॉडी चाहिए, वे डिफ़ॉल्ट फ़ॉलबैक
अनुबंध बदले बिना न्यूनतम बॉडी चुनने के लिए `emptyFallback` पास कर सकते हैं।

## डिलीवरी पिन

पिन करना डिलीवरी व्यवहार है, प्रस्तुति नहीं। `channelData.telegram.pin` जैसे
प्रदाता-मूल फ़ील्ड के बजाय `delivery.pin` का उपयोग करें।

अर्थविधान:

- `pin: true` सफलतापूर्वक डिलीवर हुए पहले संदेश को पिन करता है।
- `pin.notify` का डिफ़ॉल्ट `false` है।
- `pin.required` का डिफ़ॉल्ट `false` है।
- वैकल्पिक पिन विफलताएँ घटित होने पर कार्यक्षमता सीमित हो जाती है और भेजा गया संदेश यथावत रहता है।
- आवश्यक पिन विफलताएँ डिलीवरी को विफल कर देती हैं।
- खंडित संदेश अंतिम खंड के बजाय डिलीवर हुए पहले खंड को पिन करते हैं।

मौजूदा संदेशों के लिए मैन्युअल `pin`, `unpin`, और `pins` संदेश ऐक्शन अब भी उपलब्ध हैं,
जहाँ प्रदाता उन संक्रियाओं का समर्थन करता है।

## Plugin लेखक चेकलिस्ट

- जब चैनल अर्थपूर्ण प्रस्तुति को रेंडर कर सकता हो या सुरक्षित रूप से उसका स्तर घटा सकता हो, तब `describeMessageTool(...)` से `presentation` घोषित करें।
- रनटाइम आउटबाउंड अडैप्टर में `presentationCapabilities` जोड़ें।
- नियंत्रण-प्लेन Plugin
  सेटअप कोड में नहीं, बल्कि रनटाइम कोड में `renderPresentation` लागू करें।
- मूल UI लाइब्रेरी को हॉट सेटअप/कैटलॉग पथों से बाहर रखें।
- ज्ञात होने पर `presentationCapabilities.limits` पर
  सामान्य क्षमता सीमाएँ घोषित करें।
- रेंडरर और परीक्षणों में अंतिम प्लेटफ़ॉर्म सीमाएँ बनाए रखें।
- असमर्थित चार्ट, तालिकाओं, बटन, चयन, URL
  बटन, शीर्षक/टेक्स्ट दोहराव, और मिश्रित `message` तथा `presentation`
  प्रेषणों के लिए फ़ॉलबैक परीक्षण जोड़ें।
- केवल तभी `deliveryCapabilities.pin` और
  `pinDeliveredMessage` के माध्यम से डिलीवरी पिन समर्थन जोड़ें, जब प्रदाता भेजे गए संदेश की ID पिन कर सकता हो।
- साझा संदेश ऐक्शन स्कीमा के माध्यम से नए प्रदाता-मूल कार्ड/ब्लॉक/घटक/बटन फ़ील्ड
  उजागर न करें।

## संबंधित दस्तावेज़

- [संदेश CLI](/hi/cli/message)
- [Plugin SDK का अवलोकन](/hi/plugins/sdk-overview)
- [Plugin आर्किटेक्चर](/hi/plugins/architecture-internals#message-tool-schemas)
- [चैनल प्रस्तुति रिफ़ैक्टर योजना](/hi/plan/ui-channels)
