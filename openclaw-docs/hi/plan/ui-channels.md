---
read_when:
    - चैनल संदेश UI, इंटरैक्टिव पेलोड या नेटिव चैनल रेंडरर का पुनर्गठन
    - संदेश टूल की क्षमताएँ, डिलीवरी संकेत या क्रॉस-कॉन्टेक्स्ट मार्कर बदलना
    - Discord Carbon इम्पोर्ट फ़ैनआउट या चैनल Plugin रनटाइम लेज़ीनेस की डीबगिंग
summary: सार्थक संदेश प्रस्तुति को चैनल के नेटिव UI रेंडरर से अलग करें।
title: चैनल प्रस्तुति पुनर्संरचना योजना
x-i18n:
    generated_at: "2026-07-27T18:32:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0f0c4f64e0c503209ac0a5b763b1b5483bf8d55a28ceacffbbcd1337d4371e
    source_path: plan/ui-channels.md
    workflow: 16
---

## स्थिति

साझा एजेंट, CLI, Plugin क्षमता और आउटबाउंड डिलीवरी सतहों के लिए कार्यान्वित:

- `ReplyPayload.presentation` सिमैंटिक संदेश UI वहन करता है।
- `ReplyPayload.delivery.pin` भेजे गए संदेश को पिन करने के अनुरोध वहन करता है।
- साझा संदेश कार्रवाइयाँ प्रदाता-मूल `components`, `blocks`, `buttons` या `card` के बजाय `presentation`, `delivery` और `pin` उपलब्ध कराती हैं।
- कोर Plugin द्वारा घोषित आउटबाउंड क्षमताओं के माध्यम से प्रस्तुति रेंडर करता है या उसे स्वतः निम्नतर रूप में बदलता है।
- Discord, Slack, Telegram, Mattermost, MS Teams और Feishu रेंडरर सामान्य अनुबंध का उपयोग करते हैं।
- Discord चैनल कंट्रोल-प्लेन कोड अब Carbon-समर्थित UI कंटेनर आयात नहीं करता।

प्रामाणिक दस्तावेज़ अब [संदेश प्रस्तुति](/hi/plugins/message-presentation) में उपलब्ध हैं।
इस योजना को ऐतिहासिक कार्यान्वयन संदर्भ के रूप में रखें; अनुबंध, रेंडरर
या फ़ॉलबैक व्यवहार में बदलाव के लिए प्रामाणिक मार्गदर्शिका अपडेट करें।

## समस्या

चैनल UI वर्तमान में कई असंगत सतहों में विभाजित है:

- कोर `buildCrossContextComponents` के माध्यम से Discord-आकार का क्रॉस-कॉन्टेक्स्ट रेंडरर हुक नियंत्रित करता है।
- Discord `channel.ts`, `DiscordUiContainer` के माध्यम से मूल Carbon UI आयात कर सकता है, जिससे रनटाइम UI निर्भरताएँ चैनल Plugin कंट्रोल प्लेन में आ जाती हैं।
- एजेंट और CLI मूल पेलोड एस्केप हैच उपलब्ध कराते हैं, जैसे Discord `components`, Slack `blocks`, Telegram या Mattermost `buttons`, और Teams या Feishu `card`।
- `ReplyPayload.channelData` ट्रांसपोर्ट संकेत और मूल UI एनवेलप, दोनों वहन करता है।
- सामान्य `interactive` मॉडल मौजूद है, लेकिन यह Discord, Slack, Teams, Feishu, LINE, Telegram और Mattermost द्वारा पहले से उपयोग किए जा रहे अधिक समृद्ध लेआउट से सीमित है।

इससे कोर मूल UI आकारों से अवगत हो जाता है, Plugin रनटाइम की लेज़ीनेस कमजोर होती है और एजेंटों को समान संदेश आशय व्यक्त करने के लिए बहुत अधिक प्रदाता-विशिष्ट तरीके मिलते हैं।

## लक्ष्य

- कोर घोषित क्षमताओं के आधार पर संदेश की सर्वोत्तम सिमैंटिक प्रस्तुति तय करता है।
- एक्सटेंशन क्षमताएँ घोषित करते हैं और सिमैंटिक प्रस्तुति को मूल ट्रांसपोर्ट पेलोड में रेंडर करते हैं।
- वेब कंट्रोल UI, चैट के मूल UI से अलग रहता है।
- मूल चैनल पेलोड साझा एजेंट या CLI संदेश सतह के माध्यम से उजागर नहीं किए जाते।
- असमर्थित प्रस्तुति सुविधाएँ स्वतः सर्वोत्तम टेक्स्ट निरूपण में निम्नतर हो जाती हैं।
- भेजे गए संदेश को पिन करने जैसा डिलीवरी व्यवहार सामान्य डिलीवरी मेटाडेटा है, प्रस्तुति नहीं।

## गैर-लक्ष्य

- `buildCrossContextComponents` के लिए कोई पश्चगामी संगतता शिम नहीं।
- `components`, `blocks`, `buttons` या `card` के लिए कोई सार्वजनिक मूल एस्केप हैच नहीं।
- चैनल-मूल UI लाइब्रेरी का कोई कोर आयात नहीं।
- बंडल किए गए चैनलों के लिए कोई प्रदाता-विशिष्ट SDK सीम नहीं।

## लक्षित मॉडल

`ReplyPayload` में कोर-स्वामित्व वाला `presentation` फ़ील्ड जोड़ें।

```ts
type MessagePresentationTone = "neutral" | "info" | "success" | "warning" | "danger";

type MessagePresentation = {
  tone?: MessagePresentationTone;
  title?: string;
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] };

type MessagePresentationButton = {
  label: string;
  value?: string;
  url?: string;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  value: string;
};
```

माइग्रेशन के दौरान `interactive`, `presentation` का उपसमुच्चय बन जाता है:

- `interactive` टेक्स्ट ब्लॉक, `presentation.blocks[].type = "text"` से मैप होता है।
- `interactive` बटन ब्लॉक, `presentation.blocks[].type = "buttons"` से मैप होता है।
- `interactive` चयन ब्लॉक, `presentation.blocks[].type = "select"` से मैप होता है।

बाहरी एजेंट और CLI स्कीमा अब `presentation` का उपयोग करते हैं; मौजूदा उत्तर उत्पादकों के लिए `interactive` एक आंतरिक लेगेसी पार्सर/रेंडरिंग सहायक बना रहता है।
सार्वजनिक उत्पादक-संबंधी API, `interactive` को पदावनत मानता है। रनटाइम
समर्थन बना रहता है ताकि मौजूदा अनुमोदन सहायक और पुराने Plugin काम करना
जारी रखें, जबकि नया कोड `presentation` उत्सर्जित करता है।

## डिलीवरी मेटाडेटा

ऐसे प्रेषण व्यवहार के लिए कोर-स्वामित्व वाला `delivery` फ़ील्ड जोड़ें जो UI नहीं है।

```ts
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

सिमैंटिक्स:

- `delivery.pin = true` का अर्थ है सफलतापूर्वक डिलीवर किए गए पहले संदेश को पिन करना।
- `notify`, `false` पर डिफ़ॉल्ट होता है।
- `required`, `false` पर डिफ़ॉल्ट होता है; असमर्थित चैनल या विफल पिनिंग, डिलीवरी जारी रखते हुए स्वतः निम्नतर हो जाते हैं।
- मौजूदा संदेशों के लिए मैन्युअल `pin`, `unpin` और `list-pins` संदेश कार्रवाइयाँ बनी रहती हैं।

वर्तमान Telegram ACP विषय बाइंडिंग को `channelData.telegram.pin = true` से `delivery.pin = true` में ले जाना चाहिए।

## रनटाइम क्षमता अनुबंध

प्रस्तुति और डिलीवरी रेंडर हुक रनटाइम आउटबाउंड अडैप्टर में जोड़ें, कंट्रोल-प्लेन चैनल Plugin में नहीं।

```ts
type ChannelPresentationCapabilities = {
  supported: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  tones?: MessagePresentationTone[];
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

type ChannelDeliveryCapabilities = {
  pinSentMessage?: boolean;
};

type ChannelOutboundAdapter = {
  presentationCapabilities?: ChannelPresentationCapabilities;

  renderPresentation?: (params: {
    payload: ReplyPayload;
    presentation: MessagePresentation;
    ctx: ChannelOutboundSendContext;
  }) => ReplyPayload | null;

  deliveryCapabilities?: ChannelDeliveryCapabilities;

  pinDeliveredMessage?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
    to: string;
    threadId?: string | number | null;
    messageId: string;
    notify: boolean;
  }) => Promise<void>;
};
```

कोर व्यवहार:

- लक्षित चैनल और रनटाइम अडैप्टर को रिज़ॉल्व करें।
- प्रस्तुति क्षमताएँ पूछें।
- रेंडरिंग से पहले असमर्थित ब्लॉक को निम्नतर करें और सामान्य क्षमता सीमाएँ
  लागू करें।
- `renderPresentation` को कॉल करें।
- यदि कोई रेंडरर मौजूद नहीं है, तो प्रस्तुति को टेक्स्ट फ़ॉलबैक में बदलें।
- सफल प्रेषण के बाद, जब `delivery.pin` का अनुरोध किया गया हो और वह समर्थित हो, तो `pinDeliveredMessage` को कॉल करें।

## चैनल मैपिंग

Discord:

- `presentation` को केवल-रनटाइम मॉड्यूल में components v2 और Carbon कंटेनर में रेंडर करें।
- एक्सेंट रंग सहायकों को हल्के मॉड्यूल में रखें।
- चैनल Plugin कंट्रोल-प्लेन कोड से `DiscordUiContainer` आयात हटाएँ।

Slack:

- `presentation` को Block Kit में रेंडर करें।
- एजेंट और CLI से `blocks` इनपुट हटाएँ।

Telegram:

- टेक्स्ट, संदर्भ और डिवाइडर को टेक्स्ट के रूप में रेंडर करें।
- कॉन्फ़िगर और लक्षित सतह के लिए अनुमत होने पर कार्रवाइयों तथा चयन को इनलाइन कीबोर्ड के रूप में रेंडर करें।
- इनलाइन बटन अक्षम होने पर टेक्स्ट फ़ॉलबैक का उपयोग करें।
- ACP विषय पिनिंग को `delivery.pin` में ले जाएँ।

Mattermost:

- कॉन्फ़िगर होने पर कार्रवाइयों को इंटरैक्टिव बटन के रूप में रेंडर करें।
- अन्य ब्लॉक को टेक्स्ट फ़ॉलबैक के रूप में रेंडर करें।

MS Teams:

- `presentation` को Adaptive Cards में रेंडर करें।
- मैन्युअल पिन/अनपिन/पिन-सूची कार्रवाइयाँ बनाए रखें।
- यदि लक्षित वार्तालाप के लिए Graph समर्थन विश्वसनीय है, तो वैकल्पिक रूप से `pinDeliveredMessage` कार्यान्वित करें।

Feishu:

- `presentation` को इंटरैक्टिव कार्ड में रेंडर करें।
- मैन्युअल पिन/अनपिन/पिन-सूची कार्रवाइयाँ बनाए रखें।
- यदि API व्यवहार विश्वसनीय है, तो भेजे गए संदेश की पिनिंग के लिए वैकल्पिक रूप से `pinDeliveredMessage` कार्यान्वित करें।

LINE:

- जहाँ संभव हो, `presentation` को Flex या टेम्पलेट संदेशों में रेंडर करें।
- असमर्थित ब्लॉक के लिए टेक्स्ट पर फ़ॉलबैक करें।
- `channelData` से LINE UI पेलोड हटाएँ।

सादे या सीमित चैनल:

- सावधानीपूर्ण फ़ॉर्मेटिंग के साथ प्रस्तुति को टेक्स्ट में बदलें।

## रीफ़ैक्टर चरण

1. Discord रिलीज़ फ़िक्स को फिर लागू करें, जो `ui-colors.ts` को Carbon-समर्थित UI से अलग करता है और `extensions/discord/src/channel.ts` से `DiscordUiContainer` हटाता है।
2. `ReplyPayload`, आउटबाउंड पेलोड सामान्यीकरण, डिलीवरी सारांश और हुक पेलोड में `presentation` और `delivery` जोड़ें।
3. एक संकीर्ण SDK/रनटाइम उपपथ में `MessagePresentation` स्कीमा और पार्सर सहायक जोड़ें।
4. संदेश क्षमताओं `buttons`, `cards`, `components` और `blocks` को सिमैंटिक प्रस्तुति क्षमताओं से बदलें।
5. प्रस्तुति रेंडर और डिलीवरी पिनिंग के लिए रनटाइम आउटबाउंड अडैप्टर हुक जोड़ें।
6. क्रॉस-कॉन्टेक्स्ट घटक निर्माण को `buildCrossContextPresentation` से बदलें।
7. `src/infra/outbound/channel-adapters.ts` हटाएँ और चैनल Plugin प्रकारों से `buildCrossContextComponents` हटाएँ।
8. मूल पैरामीटर के बजाय `presentation` संलग्न करने के लिए `maybeApplyCrossContextMarker` बदलें।
9. केवल सिमैंटिक प्रस्तुति और डिलीवरी मेटाडेटा का उपयोग करने के लिए Plugin-डिस्पैच प्रेषण पथ अपडेट करें।
10. एजेंट और CLI के मूल पेलोड पैरामीटर हटाएँ: `components`, `blocks`, `buttons` और `card`।
11. मूल संदेश-टूल स्कीमा बनाने वाले SDK सहायक हटाकर उन्हें प्रस्तुति स्कीमा सहायकों से बदलें।
12. `channelData` से UI/मूल एनवेलप हटाएँ; प्रत्येक शेष फ़ील्ड की समीक्षा होने तक केवल ट्रांसपोर्ट मेटाडेटा रखें।
13. Discord, Slack, Telegram, Mattermost, MS Teams, Feishu और LINE रेंडरर माइग्रेट करें।
14. संदेश CLI, चैनल पृष्ठों, Plugin SDK और क्षमता कुकबुक के दस्तावेज़ अपडेट करें।
15. Discord और प्रभावित चैनल प्रवेश-बिंदुओं के लिए आयात फैनआउट प्रोफ़ाइलिंग चलाएँ।

इस रीफ़ैक्टर में साझा एजेंट, CLI, Plugin क्षमता और आउटबाउंड अडैप्टर अनुबंधों के लिए चरण 1-11 और 13-14 कार्यान्वित किए गए हैं। चरण 12 प्रदाता-निजी `channelData` ट्रांसपोर्ट एनवेलप के लिए अधिक गहन आंतरिक सफ़ाई चरण बना हुआ है। यदि हमें प्रकार/परीक्षण गेट से आगे आयात-फैनआउट की परिमाणित संख्याएँ चाहिए, तो चरण 15 अनुवर्ती सत्यापन बना हुआ है।

## परीक्षण

जोड़ें या अपडेट करें:

- प्रस्तुति सामान्यीकरण परीक्षण।
- असमर्थित ब्लॉक के लिए प्रस्तुति स्वतः-निम्नीकरण परीक्षण।
- Plugin डिस्पैच और कोर डिलीवरी पथों के लिए क्रॉस-कॉन्टेक्स्ट मार्कर परीक्षण।
- Discord, Slack, Telegram, Mattermost, MS Teams, Feishu, LINE और टेक्स्ट फ़ॉलबैक के लिए चैनल रेंडर मैट्रिक्स परीक्षण।
- यह सिद्ध करने वाले संदेश टूल स्कीमा परीक्षण कि मूल फ़ील्ड हट चुके हैं।
- यह सिद्ध करने वाले CLI परीक्षण कि मूल फ़्लैग हट चुके हैं।
- Carbon को कवर करने वाला Discord प्रवेश-बिंदु आयात-लेज़ीनेस रिग्रेशन।
- Telegram और सामान्य फ़ॉलबैक को कवर करने वाले डिलीवरी पिन परीक्षण।

## खुले प्रश्न

- क्या पहले चरण में `delivery.pin` को Discord, Slack, MS Teams और Feishu के लिए लागू किया जाना चाहिए, या पहले केवल Telegram के लिए?
- क्या `delivery` को अंततः `replyToId`, `replyToCurrent`, `silent` और `audioAsVoice` जैसे मौजूदा फ़ील्ड समाहित करने चाहिए, या इसे भेजने के बाद के व्यवहारों पर केंद्रित रहना चाहिए?
- क्या प्रस्तुति में सीधे इमेज या फ़ाइल संदर्भों का समर्थन होना चाहिए, या फ़िलहाल मीडिया को UI लेआउट से अलग रखना चाहिए?

## संबंधित

- [चैनलों का अवलोकन](/hi/channels)
- [संदेश प्रस्तुति](/hi/plugins/message-presentation)
