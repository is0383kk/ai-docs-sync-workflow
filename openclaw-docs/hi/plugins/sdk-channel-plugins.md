---
read_when:
    - आप एक नया मैसेजिंग चैनल Plugin बना रहे हैं
    - आप OpenClaw को किसी मैसेजिंग प्लेटफ़ॉर्म से कनेक्ट करना चाहते हैं
    - आपको ChannelPlugin अडैप्टर सतह को समझना होगा
sidebarTitle: Channel Plugins
summary: OpenClaw के लिए मैसेजिंग चैनल Plugin बनाने की चरण-दर-चरण मार्गदर्शिका
title: चैनल Plugin बनाना
x-i18n:
    generated_at: "2026-07-27T23:09:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ff8ad04346babf3eece7e10bd38946ee290947b2e504b6b5ca438865531bf38
    source_path: plugins/sdk-channel-plugins.md
    workflow: 16
---

यह मार्गदर्शिका एक ऐसा चैनल plugin बनाती है जो OpenClaw को एक मैसेजिंग
प्लेटफ़ॉर्म से जोड़ता है: DM सुरक्षा, पेयरिंग, उत्तर थ्रेडिंग और आउटबाउंड मैसेजिंग।

<Info>
  OpenClaw plugins में नए हैं? पैकेज संरचना और मैनिफ़ेस्ट सेटअप के लिए पहले
  [आरंभ करना](/hi/plugins/building-plugins) पढ़ें।
</Info>

## आपका plugin किन चीज़ों का स्वामी है

चैनल plugins भेजने/संपादित करने/प्रतिक्रिया देने वाले टूल लागू नहीं करते; कोर एक
साझा `message` टूल प्रदान करता है। आपका plugin इनका स्वामी है:

- **कॉन्फ़िगरेशन** - अकाउंट निर्धारण और सेटअप विज़ार्ड
- **सुरक्षा** - DM नीति और अनुमति-सूचियाँ
- **पेयरिंग** - DM अनुमोदन प्रवाह
- **सेशन व्याकरण** - प्रदाता-विशिष्ट वार्तालाप आईडी को आधार
  चैट, थ्रेड आईडी और पैरेंट फ़ॉलबैक से मैप करने का तरीका
- **आउटबाउंड** - प्लेटफ़ॉर्म पर टेक्स्ट, मीडिया और पोल भेजना
- **थ्रेडिंग** - उत्तरों को थ्रेड करने का तरीका
- **Heartbeat टाइपिंग** - Heartbeat डिलीवरी लक्ष्यों के लिए वैकल्पिक टाइपिंग/व्यस्तता संकेत

कोर साझा संदेश टूल, प्रॉम्प्ट वायरिंग, बाहरी सेशन-कुंजी आकार,
सामान्य `:thread:` लेखांकन और डिस्पैच का स्वामी है।

## संदेश अडैप्टर

`openclaw/plugin-sdk/channel-outbound` से `defineChannelMessageAdapter` वाला
एक `message` अडैप्टर उपलब्ध कराएँ। केवल उन्हीं टिकाऊ अंतिम-प्रेषण
क्षमताओं को घोषित करें जिन्हें आपका नेटिव ट्रांसपोर्ट वास्तव में समर्थित करता है, और इसका समर्थन
ऐसे अनुबंध परीक्षण से करें जो नेटिव साइड इफ़ेक्ट और लौटाई गई रसीद को प्रमाणित करता हो। टेक्स्ट/मीडिया
प्रेषण को उन्हीं ट्रांसपोर्ट फ़ंक्शन की ओर इंगित करें जिनका उपयोग पुराना `outbound` अडैप्टर करता है।
पूर्ण API अनुबंध, क्षमता मैट्रिक्स, रसीद नियम, लाइव पूर्वावलोकन
अंतिमीकरण, प्राप्ति अभिस्वीकृति नीति, परीक्षण और माइग्रेशन तालिका के लिए
[चैनल आउटबाउंड API](/hi/plugins/sdk-channel-outbound) देखें।

यदि आपके मौजूदा `outbound` अडैप्टर में पहले से सही प्रेषण विधियाँ और
क्षमता मेटाडेटा हैं, तो एक और ब्रिज हाथ से लिखने के बजाय
`createChannelMessageAdapterFromOutbound(...)` से `message` अडैप्टर व्युत्पन्न करें।
अडैप्टर प्रेषण `MessageReceipt` मान लौटाते हैं। पुराने आईडी के लिए समानांतर
`messageIds` फ़ील्ड रखने के बजाय उन्हें
`listMessageReceiptPlatformIds(...)` या `resolveMessageReceiptPrimaryId(...)` से व्युत्पन्न करें।

लाइव और अंतिमकर्ता क्षमताएँ सटीक रूप से घोषित करें - कोर इनसे तय करता है
कि चैनल क्या कर सकता है, और घोषित तथा वास्तविक व्यवहार के बीच अंतर
अनुबंध परीक्षण विफलता है:

| सतह                                  | मान                                                                                              |
| ------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `message.live.capabilities`           | `draftPreview`, `previewFinalization`, `progressUpdates`, `nativeStreaming`, `quietFinalization` |
| `message.live.finalizer.capabilities` | `finalEdit`, `normalFallback`, `discardPending`, `previewReceipt`, `retainOnAmbiguousFailure`    |

ड्राफ़्ट पूर्वावलोकन को उसी स्थान पर अंतिम रूप देने वाले चैनलों को रनटाइम लॉजिक
`defineFinalizableLivePreviewAdapter(...)` और
`deliverWithFinalizableLivePreviewAdapter(...)` के माध्यम से रूट करना चाहिए, और घोषित
क्षमताओं को `verifyChannelMessageLiveCapabilityAdapterProofs(...)`
तथा `verifyChannelMessageLiveFinalizerProofs(...)` परीक्षणों से समर्थित रखना चाहिए, ताकि नेटिव पूर्वावलोकन,
प्रगति, संपादन, फ़ॉलबैक/प्रतिधारण, सफ़ाई और रसीद व्यवहार में बिना सूचना के
अंतर न आ सके।

प्लेटफ़ॉर्म अभिस्वीकृतियों को स्थगित करने वाले इनबाउंड रिसीवरों को अभिस्वीकृति
समय को मॉनिटर-स्थानीय स्थिति में छिपाने के बजाय
`message.receive.defaultAckPolicy` और `supportedAckPolicies` घोषित करना चाहिए।
हर घोषित नीति को `verifyChannelMessageReceiveAckPolicyAdapterProofs(...)` से कवर करें।

`dispatchInboundReplyWithBase` और
`recordInboundSessionAndDispatchReply` जैसे पुराने उत्तर सहायक संगतता
डिस्पैचरों के लिए उपलब्ध रहते हैं। नए चैनल कोड में उनका उपयोग न करें; इसके बजाय
`message` अडैप्टर, रसीदों और
`openclaw/plugin-sdk/channel-outbound` पर प्राप्ति/प्रेषण जीवनचक्र सहायकों से शुरुआत करें।

### इनबाउंड प्रवेश (प्रयोगात्मक)

इनबाउंड प्राधिकरण माइग्रेट करने वाले चैनल रनटाइम प्राप्ति
पथों से प्रयोगात्मक `openclaw/plugin-sdk/channel-ingress-runtime` उपपथ का उपयोग कर सकते हैं।
यह प्लेटफ़ॉर्म तथ्य, अपरिष्कृत अनुमति-सूचियाँ, रूट वर्णनकर्ता, कमांड
तथ्य और एक्सेस समूह कॉन्फ़िगरेशन स्वीकार करता है, फिर प्रेषक/रूट/कमांड/सक्रियण
प्रक्षेपण और क्रमबद्ध प्रवेश ग्राफ़ लौटाता है, जबकि प्लेटफ़ॉर्म लुकअप और साइड
इफ़ेक्ट plugin में रहते हैं। plugin पहचान सामान्यीकरण को उस
वर्णनकर्ता में रखें जिसे आप रिज़ॉल्वर को देते हैं; समाधान की गई स्थिति या निर्णय से
अपरिष्कृत मिलान मान क्रमबद्ध न करें। API डिज़ाइन,
स्वामित्व सीमा और परीक्षण अपेक्षाओं के लिए
[चैनल प्रवेश API](/hi/plugins/sdk-channel-ingress) देखें।

### टिकाऊ प्रवेश और रीप्ले डीडुप्लीकेशन

टिकाऊ प्रवेश अपनाने वाले चैनलों को, जब तक उन्हें कोई सार्थक रूप से
भिन्न प्रवेश या पंप अनुबंध आवश्यक न हो, `openclaw/plugin-sdk/channel-outbound` से
`createChannelIngressMonitor` का उपयोग करना चाहिए। अपरिष्कृत ट्रांसपोर्ट एनवेलप को
एकल प्राप्ति अवरोध बिंदु पर कतार में लगाएँ (प्राप्ति के समय कोई सामान्यीकरण नहीं), Webhook
ट्रांसपोर्ट के लिए ट्रांसपोर्ट अभिस्वीकृति को टिकाऊ संलग्नता पर निर्भर रखें, हर
वार्तालाप के लिए एक क्रमबद्ध लेन व्युत्पन्न करें, और डिस्पैच
अंगीकरण पर इवेंट को पूर्ण चिह्नित करें। कतार की प्राथमिक कुंजी `(queue_name, event_id)` है और पूर्णता
पंक्ति को हटाने के बजाय उसकी टूम्बस्टोन प्रविष्टि बनाती है, इसलिए उसी
`event_id` की देर से की गई प्लेटफ़ॉर्म पुनः डिलीवरी को टूम्बस्टोन प्रतिधारण अवधि में टिकाऊ रूप से अस्वीकार कर दिया जाता है।
मॉनिटर API और शटडाउन अनुबंध के लिए
[चैनल आउटबाउंड API](/hi/plugins/sdk-channel-outbound#durable-ingress-monitors) देखें।

वह टूम्बस्टोन रीप्ले गार्ड
(`openclaw/plugin-sdk/persistent-dedupe`) के लिए लेयरिंग नियम है: ड्रेन किया गया चैनल
अलग रीप्ले गार्ड केवल तभी रखता है जब गार्ड की पहचान या प्रतिधारण कतार से अधिक हो
— ऐसी तार्किक संदेश कुंजी जो ट्रांसपोर्ट डिलीवरी आईडी से भिन्न हो (Telegram
`chat_id:message_id` का डीडुप्लीकेशन करता है क्योंकि डिबाउंस मर्ज किसी संदेश को
नए `update_id` के अंतर्गत फिर से सामने ला सकते हैं), या चैनल के टूम्बस्टोन
प्रतिधारण से लंबी अवधि हो। यदि आपकी गार्ड कुंजी ड्रेन `event_id` के बराबर होगी, तो
ड्रेन अपनाते समय गार्ड हटाएँ और इसके बजाय पुराने गार्ड अवधि को कवर करने के लिए
`completedTtlMs`/`completedMaxEntries` का आकार निर्धारित करें।
आयु सीमाओं जैसी गैर-डीडुप्लीकेशन सुरक्षाएँ इस नियम से असंबंधित हैं।
स्थिर आउटबाउंड संदेश आईडी चैनल-स्थानीय TTL कैश के बजाय
`openclaw/plugin-sdk/channel-outbound` की साझा आउटबाउंड-प्रतिध्वनि रजिस्ट्री का उपयोग करते हैं।

#### ट्रांसपोर्ट वर्ग और प्रतिधारण

ट्रांसपोर्ट को उसकी प्राप्ति सीमा पर पुनर्प्राप्ति गारंटी के अनुसार वर्गीकृत करें:

- **अभिस्वीकृति-निर्भर Webhook या इवेंट डिलीवरी:** केवल
  टिकाऊ संलग्नता के बाद अभिस्वीकृति दें या सफलता लौटाएँ। संलग्नता विफल होने पर डिलीवरी
  पुनः प्रयास के योग्य बनी रहनी चाहिए या प्राप्ति सीमा विफल होनी चाहिए। इस वर्ग में Slack, SMS, Zalo,
  Microsoft Teams, Google Chat, LINE और Synology Chat शामिल हैं।
- **प्रतीक्षित पोलिंग या स्ट्रीम डिलीवरी:** केवल संलग्नता के बाद रिमोट कर्सर आगे बढ़ाएँ या
  ट्रांसपोर्ट अभिस्वीकृति भेजें। जब कोई स्पष्ट कर्सर मौजूद न हो, तो
  प्राप्ति कॉलबैक को क्रमबद्ध और प्रतीक्षित रखें, ताकि संलग्नता विफल होने पर
  प्राप्ति लूप आगे न बढ़ सके। Telegram पोलिंग, Signal और Tlon इस वर्ग का उपयोग करते हैं;
  Telegram Webhook डिलीवरी ऊपर दिए अभिस्वीकृति-निर्भर नियम का पालन करती है।
- **गैर-रीप्ले सॉकेट:** IRC, Mattermost, Twitch और Zalo Personal
  प्लेटफ़ॉर्म से किसी स्वीकृत इवेंट को फिर से डिलीवर करने के लिए नहीं कह सकते। उनकी टिकाऊ कतार
  प्रक्रिया क्रैश अवधि की सुरक्षा करती है और स्थानीय पुनरारंभ पुनर्प्राप्ति का समर्थन करती है; पूर्णता
  टूम्बस्टोन प्लेटफ़ॉर्म रीप्ले के विरुद्ध लगभग निष्क्रिय हैं।

30 दिन को फ़्लीट टूम्बस्टोन-TTL परंपरा के रूप में उपयोग करें, SDK डिफ़ॉल्ट के रूप में नहीं।
उच्च-मात्रा पुनः डिलीवरी अवधि सामान्यतः 20,000-प्रविष्टि पूर्णता सीमा का उपयोग करती है;
कम-मात्रा वाले प्रतीक्षित और गैर-रीप्ले ट्रांसपोर्ट सामान्यतः 1,000-2,000 का उपयोग करते हैं।
वर्तमान अपवादों में LINE की 4,096-प्रविष्टि सीमाएँ, SMS का 24-घंटे का पूर्णता
TTL और Tlon का केवल-सीमा पूर्णता प्रतिधारण शामिल हैं। विफल पंक्ति सीमाएँ भी
पूर्णता सीमाओं से कम हो सकती हैं। TTL और सीमा दोनों पंक्तियाँ हटाते हैं, इसलिए प्रभावी प्रतिधारण
पहली सीमा तक पहुँचते ही समाप्त हो जाता है। केवल दस्तावेज़ित प्लेटफ़ॉर्म पुनः प्रयास
अवधि, संरक्षित जारी रीप्ले-गार्ड अवधि, अपेक्षित मात्रा या डिस्क बजट,
या गैर-रीप्ले ट्रांसपोर्ट के लिए ही विचलन करें, और प्रतिधारण अनुबंध को परीक्षणों से कवर करें।

#### कम-से-कम-एक-बार साइड इफ़ेक्ट

ड्रेन डिस्पैच, प्रवेश पंक्ति के पूर्णता टूम्बस्टोन तक पहुँचने से पहले कमांड साइड इफ़ेक्ट चलाता है।
इन चरणों के बीच प्रक्रिया क्रैश होने पर पंक्ति रीप्ले होती है और
साइड इफ़ेक्ट फिर से निष्पादित हो सकता है। यह कम-से-कम-एक-बार क्रैश अवधि
डिफ़ॉल्ट अनुबंध है। कॉन्फ़िगरेशन लेखन, स्टोरेज
साफ़ करने या उत्तर लेन के बाहर दृश्य अभिस्वीकृतियों जैसे गैर-आइडेम्पोटेंट कार्य के लिए
`openclaw/plugin-sdk/ingress-effect-once` से
`createIngressEffectOnce(...)` का उपयोग करें। प्रत्येक कॉल को स्थिर प्रवेश
`eventId` और एक इफ़ेक्ट नाम दें। प्रत्येक प्रवेश कतार/अकाउंट के लिए एक सहायक बनाएँ और
उस स्कोप के लिए स्थिर, अद्वितीय `namespacePrefix` का उपयोग करें क्योंकि ट्रांसपोर्ट इवेंट
आईडी कतार-स्थानीय हो सकते हैं। सहायक अपना टिकाऊ दावा केवल इफ़ेक्ट सफल होने के बाद
कमिट करता है; थ्रो किया गया इफ़ेक्ट दावा मुक्त कर देता है ताकि ड्रेन पुनः प्रयास उसे
दोबारा निष्पादित कर सके, जबकि समवर्ती कॉलर सक्रिय दावे की प्रतीक्षा करते हैं। टिकाऊ
स्थिति त्रुटियाँ, उपलब्ध होने पर, `onDiskError` को कॉल करती हैं और प्रक्रिया मेमोरी पर
फ़ॉलबैक करने के बजाय अस्वीकार करती हैं।

सहायक के `ttlMs` को कम-से-कम चैनल के प्रवेश टूम्बस्टोन प्रतिधारण
और इफ़ेक्ट कमिट तथा पंक्ति पूर्णता के बीच अधिकतम विलंब के योग पर सेट करें, जिसमें
सीमित डाउनटाइम और ड्रेन पुनः प्रयास शामिल हों। इफ़ेक्ट रिकॉर्ड का TTL कमिट पर शुरू होता है,
जबकि टूम्बस्टोन प्रतिधारण बाद में पूर्णता पर शुरू होता है; यदि लंबित-पंक्ति जीवनकाल
असीमित है, तो कोई सीमित TTL मनमाने डाउनटाइम को कवर नहीं करता। टूम्बस्टोन द्वारा
पंक्ति को रीप्ले न कर पाने के बाद पुराने इफ़ेक्ट रिकॉर्ड अनुपयोगी भार हैं।
`stateMaxEntries` का आकार उस प्रतिधारण अवधि में मौजूद हो सकने वाली हर अलग इवेंट/इफ़ेक्ट कुंजी
के लिए निर्धारित करें, जिसमें कतार की पूर्ण-प्रविष्टि सीमा और प्रति इवेंट अधिकतम इफ़ेक्ट
को ध्यान में रखा जाए। कम सीमा सबसे पुराने रिकॉर्ड को उसके TTL से पहले हटा देती है
और उस इफ़ेक्ट को फिर से निष्पादित होने देती है। अवशिष्ट कम-से-कम-एक-बार अवधियाँ बनी रहती हैं
यदि इफ़ेक्ट सफल होने के बाद लेकिन दावे के कमिट होने से पहले प्रक्रिया बंद हो जाए या स्थायित्व
विफल हो जाए, अथवा रिकॉर्ड उस समय समाप्त हो जाए जब उसकी प्रवेश पंक्ति अभी भी लंबित हो।

#### अकाउंट-स्कोप वाला पुनरारंभ अनुबंध

चैनल कॉन्फ़िगरेशन परिवर्तन डिफ़ॉल्ट रूप से पूरे चैनल को पुनरारंभ करते हैं। बहु-अकाउंट
चैनल `reload.accountScopedRestart: true` केवल तभी सेट कर सकता है जब कॉन्फ़िगरेशन
निर्धारण चैनल-व्यापी साझा फ़ील्ड और चयनित अकाउंट को पढ़ता हो, किसी
सहयोगी अकाउंट को कभी नहीं, और Gateway सहयोगी रनटाइम को बदले बिना एक `(channel, accountId)`
रनटाइम रोक और शुरू कर सके।

स्कोप वाला पथ केवल
`channels.<channel>.accounts.<non-default-id>.*` के अंतर्गत परिवर्तनों पर लागू होता है। साझा चैनल
फ़ील्ड, `accounts.default`, हटाए गए या समाधान न किए जा सकने वाले अकाउंट, और
इनहेरिटेंस को प्रभावित कर सकने वाले मिश्रित परिवर्तन पूरे-चैनल पुनरारंभ में प्रोन्नत किए जाते हैं।
जो plugins विकल्प नहीं चुनते वे हमेशा पूरे-चैनल पथ का उपयोग करते हैं।

टिकाऊ प्रवेश ड्रेन का उपयोग करने वाले चैनलों के लिए, अकाउंट मॉनिटर के रोकने वाले पथ को
पहले सभी स्वीकृत ट्रांसपोर्ट प्रवेशों को निपटाना होगा, फिर अपने
ड्रेन को डिस्पोज़ करके उसकी प्रतीक्षा करनी होगी। अकाउंट शुरू करने पर वही अकाउंट-कुंजी वाली कतार खुलती है, जिसका आरंभिक
ड्रेन बिना डिस्पैच की गई टिकाऊ पंक्तियाँ पुनर्प्राप्त करता है। दूसरा पुनः लोड-विशिष्ट
रीप्ले चरण न जोड़ें; कतार पुनर्प्राप्ति प्रामाणिक पुनरारंभ पथ है।

इस फ़्लैग को प्रदर्शन प्राथमिकता नहीं, बल्कि क्षमता दावा मानें। अनुबंध
परीक्षणों को प्रमाणित करना चाहिए कि एक नामित अकाउंट जोड़ने और संपादित करने से सहयोगी का
निर्धारित कॉन्फ़िगरेशन अपरिवर्तित रहता है, एक अकाउंट रोकने पर केवल उसी अकाउंट का
मॉनिटर और ड्रेन निपटते हैं, और नया मॉनिटर उस अकाउंट की पंक्तियाँ ठीक
एक बार पुनर्प्राप्त करता है। यदि किसी गारंटी को प्रमाणित नहीं किया जा सकता, तो फ़्लैग छोड़ दें।

### टाइपिंग संकेतक

यदि आपका चैनल इनबाउंड उत्तरों के बाहर टाइपिंग संकेतकों का समर्थन करता है, तो चैनल plugin पर
`heartbeat.sendTyping(...)` उपलब्ध कराएँ। कोर इसे Heartbeat मॉडल रन शुरू होने से पहले
निर्धारित Heartbeat डिलीवरी लक्ष्य के साथ कॉल करता है और
साझा टाइपिंग कीपअलाइव/सफ़ाई जीवनचक्र का उपयोग करता है। जब प्लेटफ़ॉर्म को स्पष्ट रोक संकेत की आवश्यकता हो,
तो `heartbeat.clearTyping(...)` जोड़ें।

### मीडिया स्रोत पैरामीटर

यदि आपका चैनल ऐसे संदेश-टूल पैरामीटर जोड़ता है जिनमें मीडिया स्रोत होते हैं, तो
उन पैरामीटर नामों को `plugin.actions.describeMessageTool(...).mediaSourceParams` के माध्यम से उपलब्ध कराएँ।
कोर सैंडबॉक्स पथ सामान्यीकरण और आउटबाउंड मीडिया-पहुँच नीति के लिए उस स्पष्ट सूची का उपयोग करता है,
जिससे plugins को प्रदाता-विशिष्ट अवतार, अटैचमेंट या कवर-इमेज पैरामीटर के लिए
साझा कोर में विशेष मामले जोड़ने की आवश्यकता नहीं पड़ती।

`{ "set-profile": ["avatarUrl", "avatarPath"] }` जैसे action-keyed मैप को प्राथमिकता दें
ताकि असंबंधित actions को किसी अन्य action के media args विरासत में न मिलें। एक flat array
उन params के लिए अब भी काम करता है जिन्हें जानबूझकर प्रत्येक प्रदर्शित action में साझा किया जाता है।

जिन channels को platform-side media fetch के लिए अस्थायी public URL प्रदर्शित करना आवश्यक है,
वे plugin state stores के साथ
`openclaw/plugin-sdk/outbound-media` से `createHostedOutboundMediaStore(...)` का उपयोग कर सकते हैं। platform
route parsing और token enforcement को channel plugin में रखें; shared helper
केवल media loading, expiry metadata, chunk rows और cleanup का स्वामी है।

Inbound attachments parallel `Media*` fields के बजाय ordered facts का उपयोग करते हैं। channel
records को `openclaw/plugin-sdk/channel-inbound` से `toInboundMediaFacts(...)` के साथ normalize करें
और inbound context बनाते समय उन्हें `media` के रूप में पास करें।
जब किसी plugin को local media reads अधिकृत करने हों, तो केंद्रित
`openclaw/plugin-sdk/media-local-roots` subpath से
`getAgentScopedMediaLocalRoots(...)` या
`getAgentScopedMediaLocalRootsForSources(...)` import करें। पुराना
`agent-media-payload` builder/root facade deprecated compatibility है।

### नेटिव payload का आकार निर्धारण

यदि आपके channel को `message(action="send")` के लिए provider-specific आकार निर्धारण चाहिए,
तो `actions.prepareSendPayload(...)` को प्राथमिकता दें। native cards, blocks, embeds या
अन्य durable data को `payload.channelData.<channel>` के अंतर्गत रखें और core को
outbound/message adapter के माध्यम से भेजने दें। केवल ऐसे payloads के लिए, जिन्हें serialize
करके दोबारा आज़माया नहीं जा सकता, compatibility fallback के रूप में send हेतु `actions.handleAction(...)` का उपयोग करें।

### Session conversation grammar

यदि आपका platform conversation ids के अंदर अतिरिक्त scope संग्रहीत करता है, तो उस parsing को
plugin में `messaging.resolveSessionConversation(...)` के साथ रखें। यह
`rawId` को base conversation id, वैकल्पिक
thread id, स्पष्ट `baseConversationId` और किसी भी
`parentConversationCandidates` पर map करने का canonical hook है। जब आप `parentConversationCandidates` लौटाएँ,
तो उन्हें सबसे सीमित parent से सबसे व्यापक/base conversation के क्रम में रखें।

`messaging.resolveParentConversationCandidates(...)` उन plugins के लिए deprecated
compatibility fallback है जिन्हें generic/raw id के ऊपर केवल parent fallbacks चाहिए।
यदि दोनों hooks मौजूद हों, तो core पहले
`resolveSessionConversation(...).parentConversationCandidates` का उपयोग करता है और canonical
hook द्वारा उन्हें छोड़े जाने पर ही `resolveParentConversationCandidates(...)` पर fallback करता है।

जिन bundled plugins को channel registry boot होने से पहले समान parsing चाहिए,
वे matching `resolveSessionConversation(...)` export के साथ top-level
`session-key-api.ts` file प्रदर्शित कर सकते हैं (Feishu और Telegram
plugins देखें)। core उस bootstrap-safe surface का उपयोग केवल तब करता है, जब runtime plugin
registry अभी उपलब्ध नहीं होती।

जब plugin code को route-like fields normalize करने, child thread की उसके parent route से तुलना करने,
या `{ channel, to, accountId, threadId }` से stable dedupe key बनाने की आवश्यकता हो, तो
`openclaw/plugin-sdk/channel-route` का उपयोग करें। helper
numeric thread ids को core के समान तरीके से normalize करता है, इसलिए तदर्थ
`String(threadId)` comparisons के बजाय इसे प्राथमिकता दें। provider-specific target grammar वाले plugins को
`messaging.resolveOutboundSessionRoute(...)` प्रदर्शित करना चाहिए, ताकि core को
parser shims के बिना provider-native session और thread identity मिले।

### Account-scoped conversation binding support

जब channel generic current-conversation bindings का समर्थन करता है, तो
`conversationBindings.supportsCurrentConversationBinding` सेट करें। `createChatChannelPlugin(...)`
इस static capability को default रूप से `true` पर सेट करता है।

यदि support configured account के अनुसार अलग होता है, तो
`conversationBindings.isCurrentConversationBindingSupported({ accountId })` भी implement करें।
core इस synchronous hook का मूल्यांकन static capability enable होने के बाद ही करता है।
`false` लौटाने पर उस account के लिए generic current-conversation capability,
bind, lookup, list, touch और unbind operations अनुपलब्ध हो जाते हैं।
hook को छोड़ने पर static capability प्रत्येक account पर लागू होती है।

उत्तर को पहले से loaded account config या runtime state से resolve करें। यह
hook केवल generic current-conversation bindings को gate करता है; यह
configured binding rules या plugin-owned session routing को प्रतिस्थापित नहीं करता। Contract tests में
`openclaw/plugin-sdk/channel-core` द्वारा export किए गए
`ChannelPlugin["conversationBindings"]` contract के माध्यम से कम-से-कम एक supported और एक unsupported account
शामिल होना चाहिए।

## अनुमोदन और channel capabilities

अधिकांश channel plugins को approval-specific code की आवश्यकता नहीं होती। core same-chat
`/approve`, shared approval button payloads और generic fallback delivery का स्वामी है।
`ChannelPlugin.approvals` हटा दिया गया था; इसके बजाय approval delivery/native/render/auth
facts को एक `approvalCapability` object पर रखें। `plugin.auth` केवल login/logout
के लिए है—core अब उस object से approval auth hooks नहीं पढ़ता।

`approvalCapability.delivery` का उपयोग केवल native approval routing या fallback
suppression के लिए करें, और `approvalCapability.render` का उपयोग केवल तब करें जब किसी channel को
shared renderer के बजाय वास्तव में custom approval payloads चाहिए।

### अनुमोदन auth

- `approvalCapability.authorizeActorAction` और
  `approvalCapability.getActionAvailabilityState` canonical
  approval-auth seam हैं।
- same-chat approval auth availability के लिए `getActionAvailabilityState` का उपयोग करें।
  native delivery disabled होने पर भी configured approvers को `/approve` के लिए उपलब्ध रखें;
  इसके बजाय delivery/setup guidance के लिए native initiating-surface state का उपयोग करें।
- यदि आपका channel native exec approvals प्रदर्शित करता है, तो
  initiating-surface/native-client state के लिए
  `approvalCapability.getExecInitiatingSurfaceState` का उपयोग करें, जब वह same-chat
  approval auth से अलग हो। core उस exec-specific hook का उपयोग `enabled` और
  `disabled` में अंतर करने, initiating channel द्वारा native exec
  approvals के support का निर्णय लेने और channel को native-client fallback guidance में शामिल करने के लिए करता है।
  सामान्य स्थिति में `createApproverRestrictedNativeApprovalCapability(...)` इसे भरता है।
- यदि कोई channel मौजूदा config से stable owner-like DM identities का अनुमान लगा सकता है,
  तो approval-specific core logic जोड़े बिना same-chat `/approve` को सीमित करने के लिए
  `openclaw/plugin-sdk/approval-runtime` से `createResolvedApproverActionAuthAdapter` का उपयोग करें।
- यदि custom approval auth जानबूझकर केवल same-chat fallback की अनुमति देता है, तो
  `openclaw/plugin-sdk/approval-auth-runtime` से `markImplicitSameChatApprovalAuthorization({ authorized: true })` लौटाएँ; अन्यथा core
  परिणाम को स्पष्ट approver authorization मानता है।
- यदि channel-owned native callback approvals को सीधे resolve करता है, तो resolve करने से पहले
  `isImplicitSameChatApprovalAuthorization(...)` का उपयोग करें, ताकि implicit
  fallback फिर भी channel के सामान्य actor authorization से होकर जाए।

### Payload lifecycle और setup guidance

- channel-specific payload lifecycle
  behavior के लिए `outbound.shouldSuppressLocalPayloadPrompt` या
  `outbound.beforeDeliverPayload` का उपयोग करें, जैसे duplicate local approval prompts छिपाना या delivery
  से पहले typing indicators भेजना।
- जब channel चाहता है कि disabled-path reply native exec approvals enable करने के लिए
  आवश्यक सटीक config knobs समझाए, तो `approvalCapability.describeExecApprovalSetup` का उपयोग करें।
  hook को `{ channel, channelLabel, accountId }` मिलता है;
  named-account channels को top-level
  defaults के बजाय `channels.<channel>.accounts.<id>.execApprovals.*` जैसे account-scoped paths render करने चाहिए।
- जब plugin approval failure guidance को plugin approval no-route और timeout
  failures के लिए दिखाना सुरक्षित हो, तो `approvalCapability.describePluginApprovalSetup` का उपयोग करें।
  `createApproverRestrictedNativeApprovalCapability(...)`, `describeExecApprovalSetup` से इसका
  अनुमान नहीं लगाता; उसी helper को स्पष्ट रूप से केवल तभी पास करें, जब plugin और exec approvals वास्तव में समान native setup का उपयोग करते हों।

### नेटिव अनुमोदन delivery

यदि किसी channel को native approval delivery चाहिए, तो channel code को
target normalization और transport/presentation facts पर केंद्रित रखें।
`openclaw/plugin-sdk/approval-runtime` से
`createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`,
`createChannelApproverDmTargetResolver` और
`createApproverRestrictedNativeApprovalCapability` का उपयोग करें। channel-specific facts को
`approvalCapability.nativeRuntime` के पीछे रखें, आदर्श रूप से
`createChannelApprovalNativeRuntimeAdapter(...)` या
`createLazyChannelApprovalNativeRuntimeAdapter(...)` के माध्यम से, ताकि core handler को assemble कर सके और
request filtering, routing, dedupe, expiry, Gateway
subscription तथा routed-elsewhere notices का स्वामी रहे।

`nativeRuntime` को कुछ छोटे seams में विभाजित किया गया है:

- `availability` - account configured है या नहीं और request को
  handle किया जाना चाहिए या नहीं
- `presentation` - shared approval view model को
  pending/resolved/expired native payloads या final actions में map करें
- `transport` - targets तैयार करें तथा native approval
  messages send/update/delete करें
- `interactions` - native buttons
  या reactions के लिए वैकल्पिक bind/unbind/clear-action hooks, साथ ही वैकल्पिक `cancelDelivered` hook। जब `deliverPending` in-process या persistent
  state (जैसे reaction target store) register करता है, तो `cancelDelivered` implement करें,
  ताकि handler stop द्वारा `bindPending` चलने से पहले delivery cancel करने पर,
  या `bindPending` द्वारा कोई handle न लौटाए जाने पर, उस state को release किया जा सके
- `observe` - वैकल्पिक delivery diagnostics hooks

अन्य approval helpers:

- जब कोई channel session-origin native delivery और explicit approval forwarding targets दोनों का समर्थन करता है,
  तो `openclaw/plugin-sdk/approval-native-runtime` से
  `createNativeApprovalChannelRouteGates` का उपयोग करें। helper
  approval config selection, `mode` handling, agent/session
  filters, account binding, session-target matching और target-list matching को centralize करता है,
  जबकि callers अभी भी channel id, default forwarding mode, account
  lookup, transport-enabled check, target normalization और turn-source
  target resolution के स्वामी रहते हैं। core-owned channel policy
  defaults बनाने के लिए इसका उपयोग न करें; channel के documented default mode को स्पष्ट रूप से पास करें।
- `createChannelNativeOriginTargetResolver`, `{ to, accountId, threadId }` targets के लिए default रूप से shared channel-route
  matcher का उपयोग करता है। `targetsMatch` केवल तब पास करें, जब किसी channel के पास provider-specific equivalence rules हों,
  जैसे Slack timestamp prefix matching। जब
  default route matcher या custom `targetsMatch` callback चलने से पहले channel को provider ids canonicalize करने की आवश्यकता हो,
  तब original target को delivery के लिए संरक्षित रखते हुए `normalizeTargetForMatch` पास करें।
  `normalizeTarget` का उपयोग केवल तब करें, जब resolved
  delivery target को ही canonicalize किया जाना चाहिए।
- यदि channel को client, token, Bolt
  app या webhook receiver जैसे runtime-owned objects चाहिए, तो उन्हें
  `openclaw/plugin-sdk/channel-runtime-context` के माध्यम से register करें। generic runtime-context
  registry core को approval-specific wrapper glue जोड़े बिना channel
  startup state से capability-driven handlers bootstrap करने देता है।
- lower-level `createChannelApprovalHandler` या
  `createChannelNativeApprovalRuntime` का उपयोग केवल तब करें, जब capability-driven seam
  अभी पर्याप्त रूप से expressive न हो।
- native approval channels को `accountId` और `approvalKind` दोनों को
  उन helpers के माध्यम से route करना आवश्यक है। `accountId` multi-account approval policy को
  सही bot account तक scoped रखता है, और `approvalKind` exec बनाम plugin
  approval behavior को core में hardcoded branches के बिना channel के लिए उपलब्ध रखता है।
- core approval reroute notices का भी स्वामी है। channel plugins को
  `createChannelNativeApprovalRuntime` से अपने "approval DMs / किसी अन्य channel पर गया" follow-up messages
  नहीं भेजने चाहिए; इसके बजाय shared approval capability helpers के माध्यम से accurate origin +
  approver-DM routing प्रदर्शित करें और initiating chat पर कोई notice वापस post करने से पहले
  core को actual deliveries aggregate करने दें।
- delivered approval id kind को शुरू से अंत तक संरक्षित रखें। native clients को
  channel-local state से exec बनाम plugin approval routing का अनुमान नहीं लगाना या उसे rewrite नहीं करना चाहिए।
- उस explicit `approvalKind` को `resolveApprovalOverGateway` में पास करें। यह
  canonical `approval.resolve` service का उपयोग करता है और किसी अन्य surface द्वारा पहले उत्तर दिए जाने पर
  recorded winner लौटाता है। पुराना explicit `resolveMethod` input
  command-backed controls के लिए बना हुआ है; नए native actions को इसका उपयोग नहीं करना चाहिए या
  ID से kind का अनुमान नहीं लगाना चाहिए।
- अलग-अलग approval kinds जानबूझकर अलग native
  surfaces प्रदर्शित कर सकते हैं। वर्तमान bundled examples: Matrix exec और plugin approvals के लिए समान native DM/channel
  routing और reaction UX बनाए रखता है, जबकि auth को approval kind के अनुसार अलग होने देता है;
  Slack exec और plugin ids दोनों के लिए native approval routing उपलब्ध रखता है।
- `createApproverRestrictedNativeApprovalAdapter` अब भी
  compatibility wrapper के रूप में मौजूद है, लेकिन नए code को capability builder को प्राथमिकता देनी चाहिए
  और plugin पर `approvalCapability` प्रदर्शित करना चाहिए।

### अधिक सीमित approval runtime subpaths

hot channel entrypoints के लिए, जब आपको उस family के केवल एक भाग की आवश्यकता हो,
तो व्यापक `approval-runtime` barrel के बजाय इन अधिक सीमित subpaths को प्राथमिकता दें:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-reference-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

इसी तरह, जब आपको सभी की आवश्यकता न हो, तो व्यापक अम्ब्रेला सतहों के बजाय
`openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference`, और
`openclaw/plugin-sdk/reply-chunking` को प्राथमिकता दें।

### सेटअप उपपथ

- `openclaw/plugin-sdk/setup-runtime` रनटाइम-सुरक्षित सेटअप सहायकों को समेटता है:
  `createSetupTranslator`, आयात-सुरक्षित सेटअप पैच अडैप्टर
  (`createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), लुकअप-नोट आउटपुट,
  `promptResolvedAllowFrom`, `splitSetupEntries`, और प्रत्यायोजित
  सेटअप-प्रॉक्सी बिल्डर।
- `openclaw/plugin-sdk/channel-setup` वैकल्पिक-इंस्टॉल सेटअप
  बिल्डरों के साथ कुछ सेटअप-सुरक्षित प्रिमिटिव को समेटता है: `createOptionalChannelSetupSurface`,
  `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`,
  `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`,
  `setSetupChannelEnabled`, और `splitSetupEntries`।
- व्यापक `openclaw/plugin-sdk/setup` सीम का उपयोग केवल तभी करें, जब आपको
  `moveSingleAccountChannelSectionToDefaultAccount(...)` जैसे अधिक भारी साझा सेटअप/कॉन्फ़िग सहायक भी
  चाहिए।

यदि आपका चैनल सेटअप सतहों में केवल "पहले इस Plugin को इंस्टॉल करें" दिखाना
चाहता है, तो `createOptionalChannelSetupSurface(...)` को प्राथमिकता दें। जनरेट किया गया
अडैप्टर/विज़ार्ड कॉन्फ़िग लेखन और अंतिम रूप देने पर फ़ेल-क्लोज़ होता है, और वह
सत्यापन, अंतिम रूप देने तथा दस्तावेज़-लिंक कॉपी में उसी इंस्टॉल-आवश्यक संदेश का
पुनः उपयोग करता है।

यदि आपका चैनल परिवेश-चालित सेटअप या प्रमाणीकरण का समर्थन करता है, तो उसे
चैनल कॉन्फ़िग स्कीमा और सेटअप डिस्क्रिप्टर के माध्यम से उजागर करें। ऑपरेटर-दृश्य
कॉपी के लिए ही चैनल रनटाइम `envVars` या स्थानीय स्थिरांक रखें।

यदि Plugin रनटाइम शुरू होने से पहले आपका चैनल `status`, `channels list`, `channels status`, या
SecretRef स्कैन में दिखाई दे सकता है, तो `package.json` में
`openclaw.setupEntry` जोड़ें। वह एंट्रीपॉइंट केवल-पठन कमांड
पथों में आयात करने के लिए सुरक्षित होना चाहिए और उन सारांशों के लिए आवश्यक
चैनल मेटाडेटा, सेटअप-सुरक्षित कॉन्फ़िग अडैप्टर, स्थिति अडैप्टर और चैनल सीक्रेट
लक्ष्य मेटाडेटा लौटाना चाहिए। सेटअप प्रविष्टि से क्लाइंट, लिसनर या ट्रांसपोर्ट
रनटाइम शुरू न करें।

मुख्य चैनल प्रविष्टि के आयात पथ को भी सीमित रखें। डिस्कवरी चैनल को सक्रिय किए
बिना क्षमताएँ पंजीकृत करने के लिए प्रविष्टि और चैनल Plugin मॉड्यूल का मूल्यांकन
कर सकती है। `channel-plugin-api.ts` जैसी फ़ाइलों को सेटअप विज़ार्ड,
ट्रांसपोर्ट क्लाइंट, सॉकेट लिसनर, सबप्रोसेस लॉन्चर या सेवा स्टार्टअप मॉड्यूल
आयात किए बिना चैनल Plugin ऑब्जेक्ट निर्यात करना चाहिए। उन रनटाइम हिस्सों को
`registerFull(...)`, रनटाइम सेटर या लेज़ी क्षमता अडैप्टर से लोड किए गए मॉड्यूल
में रखें।

### अन्य सीमित चैनल उपपथ

अन्य हॉट चैनल पथों के लिए व्यापक लेगेसी सतहों के बजाय सीमित सहायकों को
प्राथमिकता दें:

- बहु-खाता कॉन्फ़िग और डिफ़ॉल्ट-खाता फ़ॉलबैक के लिए
  `openclaw/plugin-sdk/account-core`, `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution`, और
  `openclaw/plugin-sdk/account-helpers`
- इनबाउंड रूट/एनवेलप और रिकॉर्ड-एंड-डिस्पैच वायरिंग के लिए
  `openclaw/plugin-sdk/inbound-envelope` और
  `openclaw/plugin-sdk/channel-inbound`
- लक्ष्य पार्सिंग सहायकों के लिए `openclaw/plugin-sdk/channel-targets`
- आउटबाउंड पहचान/प्रेषण डेलिगेट और टाइपयुक्त पेलोड योजना के लिए
  `openclaw/plugin-sdk/channel-outbound`
- जब किसी आउटबाउंड रूट को स्पष्ट
  `replyToId`/`threadId` संरक्षित करना चाहिए या आधार सेशन कुंजी के अब भी मेल खाने पर
  वर्तमान `:thread:` सेशन पुनर्प्राप्त करना चाहिए, तब
  `openclaw/plugin-sdk/channel-core` से `buildThreadAwareOutboundSessionRoute(...)`।
  जब उनके प्लेटफ़ॉर्म में नेटिव थ्रेड डिलीवरी अर्थविज्ञान हो, तो प्रदाता
  Plugin वरीयता, प्रत्यय व्यवहार और थ्रेड आईडी सामान्यीकरण को ओवरराइड कर सकते हैं।
- थ्रेड-बाइंडिंग जीवनचक्र और अडैप्टर पंजीकरण के लिए
  `openclaw/plugin-sdk/thread-bindings-runtime`

केवल-प्रमाणीकरण चैनल सामान्यतः डिफ़ॉल्ट पथ पर रुक सकते हैं: कोर अनुमोदनों को
संभालता है और Plugin केवल आउटबाउंड/प्रमाणीकरण क्षमताएँ उजागर करता है। Matrix,
Slack, Telegram और कस्टम चैट ट्रांसपोर्ट जैसे नेटिव अनुमोदन चैनलों को अपना
अनुमोदन जीवनचक्र स्वयं बनाने के बजाय साझा नेटिव सहायकों का उपयोग करना चाहिए।

## इनबाउंड उल्लेख नीति

इनबाउंड उल्लेख प्रबंधन को दो परतों में विभाजित रखें:

- Plugin-स्वामित्व वाला साक्ष्य संग्रह
- साझा नीति मूल्यांकन

उल्लेख-नीति निर्णयों के लिए `openclaw/plugin-sdk/channel-mention-gating` का उपयोग करें।
व्यापक इनबाउंड सहायक बैरल की आवश्यकता होने पर ही
`openclaw/plugin-sdk/channel-inbound` का उपयोग करें।

Plugin-स्थानीय लॉजिक के लिए उपयुक्त:

- बॉट को उत्तर देने की पहचान
- उद्धृत बॉट की पहचान
- थ्रेड सहभागिता जाँच
- सेवा/सिस्टम-संदेश अपवर्जन
- बॉट की सहभागिता सिद्ध करने के लिए आवश्यक प्लेटफ़ॉर्म-नेटिव कैश

साझा सहायक के लिए उपयुक्त:

- `requireMention`
- स्पष्ट उल्लेख परिणाम
- अंतर्निहित उल्लेख अनुमति-सूची
- कमांड बायपास
- अंतिम छोड़ने का निर्णय

वरीय प्रवाह:

1. स्थानीय उल्लेख तथ्य परिकलित करें।
2. उन तथ्यों को `resolveInboundMentionDecision({ facts, policy })` में भेजें।
3. अपने इनबाउंड गेट में `decision.effectiveWasMentioned`, `decision.shouldBypassMention`, और
   `decision.shouldSkip` का उपयोग करें।

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";
import { resolveChannelImplicitMentions } from "openclaw/plugin-sdk/channel-ingress-runtime";

const wasMentioned = matchesMentionWithExplicit({
  text,
  mentionRegexes,
  explicit: {
    hasAnyMention,
    isExplicitlyMentioned,
    canResolveExplicit,
  },
});

const facts = {
  canDetectMention: true,
  wasMentioned,
  hasAnyMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const implicitMentions = resolveChannelImplicitMentions({
  cfg,
  channel: channelId,
  accountId,
});

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    implicitMentions,
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`matchesMentionWithExplicit(...)` एक बूलियन लौटाता है। `hasAnyMention`,
`isExplicitlyMentioned`, और `canResolveExplicit` चैनल के अपने
नेटिव उल्लेख मेटाडेटा (संदेश इकाइयाँ, रिप्लाई-टू-बॉट फ़्लैग और इनके समान) से
आते हैं; यदि आपका प्लेटफ़ॉर्म उनका पता नहीं लगा सकता, तो
`false`/`undefined` मान दें।

`api.runtime.channel.mentions` उन बंडल किए गए चैनल Plugin के लिए वही साझा उल्लेख सहायक
उजागर करता है जो पहले से रनटाइम इंजेक्शन पर निर्भर हैं:
`buildMentionRegexes`, `matchesMentionPatterns`, `matchesMentionWithExplicit`,
`implicitMentionKindWhen`, `resolveInboundMentionDecision`।

यदि आपको केवल `implicitMentionKindWhen` और `resolveInboundMentionDecision` चाहिए,
तो असंबंधित इनबाउंड रनटाइम सहायकों को लोड करने से बचने के लिए
`openclaw/plugin-sdk/channel-mention-gating` से आयात करें।

## चरण-दर-चरण मार्गदर्शिका

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="पैकेज और मैनिफ़ेस्ट">
    मानक Plugin फ़ाइलें बनाएँ। `openclaw.plugin.json` में
    `channels` फ़ील्ड (`kind` फ़ील्ड नहीं) ही किसी मैनिफ़ेस्ट को
    चैनल का स्वामी चिह्नित करती है। संपूर्ण पैकेज-मेटाडेटा सतह के लिए
    [Plugin सेटअप और कॉन्फ़िग](/hi/plugins/sdk-setup#openclaw-channel) देखें:

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "OpenClaw को Acme Chat से कनेक्ट करें।"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Acme Chat चैनल Plugin",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {}
      },
      "channelConfigs": {
        "acme-chat": {
          "schema": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          },
          "uiHints": {
            "token": {
              "label": "बॉट टोकन",
              "sensitive": true
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

    `configSchema`, `plugins.entries.acme-chat.config` को सत्यापित करता है। इसका उपयोग
    Plugin-स्वामित्व वाली उन सेटिंग्स के लिए करें जो चैनल खाता कॉन्फ़िग नहीं हैं।
    `channelConfigs.acme-chat.schema`, `channels.acme-chat` को सत्यापित करता है और वह
    Plugin रनटाइम लोड होने से पहले कॉन्फ़िग स्कीमा, सेटअप और UI सतहों द्वारा
    प्रयुक्त कोल्ड-पाथ स्रोत है। शीर्ष-स्तरीय फ़ील्ड के संपूर्ण संदर्भ के लिए
    [Plugin मैनिफ़ेस्ट](/hi/plugins/manifest) देखें।

  </Step>

  <Step title="चैनल Plugin ऑब्जेक्ट बनाएँ">
    `ChannelPlugin` इंटरफ़ेस में कई वैकल्पिक अडैप्टर सतहें हैं। न्यूनतम -
    `id`, `config`, और `setup` - से शुरू करें और आवश्यकतानुसार
    अडैप्टर जोड़ें।

    `src/channel.ts` बनाएँ:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        // Account resolution/inspection belongs on `config`, not `setup`.
        // `setup` covers onboarding writes (applyAccountConfig, validateInput).
        config: {
          listAccountIds: () => ["default"],
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
        setup: {
          applyAccountConfig: ({ cfg, input }) => ({
            ...cfg,
            channels: {
              ...cfg.channels,
              "acme-chat": { ...(cfg.channels as any)?.["acme-chat"], ...input },
            },
          }),
        },
      }),

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
      outbound: {
        attachedResults: {
          channel: "acme-chat",
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    उन चैनलों के लिए जो मानक शीर्ष-स्तरीय DM कुंजियाँ और पुराने नेस्टेड कुंजियाँ, दोनों स्वीकार करते हैं, `plugin-sdk/channel-config-helpers` के सहायक फ़ंक्शन का उपयोग करें: `resolveChannelDmAccess`, `resolveChannelDmPolicy`, `resolveChannelDmAllowFrom`, और `normalizeChannelDmPolicy` खाता-स्थानीय मानों को इनहेरिट किए गए रूट मानों से पहले रखते हैं। उसी रिज़ॉल्वर को `normalizeLegacyDmAliases` के माध्यम से doctor मरम्मत के साथ जोड़ें, ताकि रनटाइम और माइग्रेशन एक ही अनुबंध को पढ़ें।

    <Accordion title="createChatChannelPlugin आपके लिए क्या करता है">
      निम्न-स्तरीय अडैप्टर इंटरफ़ेस को मैन्युअल रूप से लागू करने के बजाय, आप
      घोषणात्मक विकल्प देते हैं और बिल्डर उन्हें संयोजित करता है:

      | विकल्प | यह क्या जोड़ता है |
      | --- | --- |
      | `security.dm` | कॉन्फ़िगरेशन फ़ील्ड से स्कोप किया गया DM सुरक्षा रिज़ॉल्वर |
      | `pairing.text` | कोड विनिमय वाला टेक्स्ट-आधारित DM पेयरिंग प्रवाह |
      | `threading` | रिप्लाई-टू-मोड रिज़ॉल्वर (निश्चित, खाता-स्कोप किया गया या कस्टम) |
      | `outbound.attachedResults` | परिणाम मेटाडेटा (संदेश ID) लौटाने वाले प्रेषण फ़ंक्शन; इसके लिए समान स्तर पर `channel` ID आवश्यक है, ताकि कोर लौटाए गए डिलीवरी परिणाम पर उसे अंकित कर सके |

      यदि आपको पूर्ण नियंत्रण चाहिए, तो आप घोषणात्मक विकल्पों के बजाय
      अपरिष्कृत अडैप्टर ऑब्जेक्ट भी दे सकते हैं।

      अपरिष्कृत आउटबाउंड अडैप्टर एक `chunker(text, limit, ctx)` फ़ंक्शन परिभाषित कर सकते हैं।
      वैकल्पिक `ctx.formatting` डिलीवरी-समय के फ़ॉर्मेटिंग निर्णय रखता है,
      जैसे `maxLinesPerMessage`; इसे भेजने से पहले लागू करें, ताकि रिप्लाई थ्रेडिंग
      और खंड सीमाएँ साझा आउटबाउंड डिलीवरी द्वारा केवल एक बार निर्धारित हों।
      जब कोई नेटिव रिप्लाई लक्ष्य निर्धारित हो जाता है, तब प्रेषण संदर्भ में
      `replyToIdSource` (`implicit` या `explicit`) भी शामिल होता है,
      ताकि पेलोड सहायक किसी अंतर्निहित एकल-उपयोग रिप्लाई स्लॉट का उपभोग किए बिना
      स्पष्ट रिप्लाई टैग सुरक्षित रख सकें।
    </Accordion>

    ### समूह टूल-नीति अडैप्टर

    एक चैनल जो `group.resolveToolPolicy` को लागू करता है और
    `toolsBySender` का समर्थन करता है, उसे पूर्ण `ChannelGroupContext` अपने
    साझा नीति रिज़ॉल्वर को अग्रेषित करना आवश्यक है। विशेष रूप से, मेल खाने वाले समूह और वाइल्डकार्ड
    दोनों स्कोप पर प्रेषक-विशिष्ट ओवरले छोड़कर `senderPolicyMode: "never"`
    का पालन करें, जबकि आधार `tools` नीति फिर भी लागू करें।

    OpenClaw यह मोड केवल विश्वसनीय गैर-इनग्रेस निष्पादन के लिए सेट करता है, जिसमें प्रेषक का
    प्राधिकार पहले ही सर्वर-स्वामित्व वाले एनवेलप में कैप्चर किया जा चुका हो, जैसे कि
    स्पष्ट रूप से सीमित शेड्यूल किया गया रन। Plugins को यह मोड
    इनबाउंड मेटाडेटा से प्राप्त नहीं करना चाहिए, इसे चैनल स्थिति के रूप में बनाए नहीं रखना चाहिए, या इसे कॉन्फ़िगरेशन के रूप में उजागर नहीं करना चाहिए। ऐसा
    अडैप्टर परीक्षण जोड़ें जो सिद्ध करे कि यह मोड मेल खाते आधार `tools`
    प्रतिबंध को हटाए बिना वाइल्डकार्ड `toolsBySender` प्रविष्टि छोड़ देता है।

  </Step>

  <Step title="प्रवेश बिंदु को जोड़ें">
    `index.ts` बनाएँ:

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme Chat channel plugin",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme Chat management");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Acme Chat management",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    चैनल-स्वामित्व वाले CLI डिस्क्रिप्टर `registerCliMetadata(...)` में रखें, ताकि OpenClaw
    पूर्ण चैनल रनटाइम सक्रिय किए बिना उन्हें रूट सहायता में दिखा सके,
    जबकि सामान्य पूर्ण लोड वास्तविक कमांड पंजीकरण के लिए उन्हीं डिस्क्रिप्टर को
    प्राप्त करते रहें। `registerFull(...)` को केवल रनटाइम के कार्यों के लिए रखें।
    `defineChannelPluginEntry` पंजीकरण-मोड विभाजन को स्वचालित रूप से संभालता है।
    यदि `registerFull(...)` Gateway RPC विधियाँ पंजीकृत करता है, तो
    Plugin-विशिष्ट उपसर्ग का उपयोग करें। मुख्य व्यवस्थापक नेमस्पेस (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) आरक्षित रहते हैं और हमेशा
    `operator.admin` में रिज़ॉल्व होते हैं। सभी विकल्पों के लिए
    [प्रवेश बिंदु](/hi/plugins/sdk-entrypoints#definechannelpluginentry) देखें।

  </Step>

  <Step title="सेटअप प्रविष्टि जोड़ें">
    ऑनबोर्डिंग के दौरान हल्के लोडिंग के लिए `setup-entry.ts` बनाएँ:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    चैनल अक्षम या कॉन्फ़िगर न होने पर OpenClaw पूर्ण एंट्री के बजाय इसे लोड करता है।
    यह सेटअप प्रवाहों के दौरान भारी रनटाइम कोड लोड करने से बचाता है।
    विवरण के लिए [सेटअप और कॉन्फ़िगरेशन](/hi/plugins/sdk-setup#setup-entry) देखें।

    साइडकार मॉड्यूल में सेटअप-सुरक्षित एक्सपोर्ट विभाजित करने वाले बंडल किए गए
    वर्कस्पेस चैनल, जब उन्हें स्पष्ट सेटअप-समय रनटाइम सेटर की भी आवश्यकता हो,
    तब `openclaw/plugin-sdk/channel-entry-contract` से `defineBundledChannelSetupEntry(...)` का उपयोग कर सकते हैं।

  </Step>

  <Step title="इनबाउंड संदेश संभालें">
    आपके Plugin को प्लेटफ़ॉर्म से संदेश प्राप्त करके उन्हें OpenClaw को अग्रेषित
    करना होगा। सामान्य पैटर्न ऐसा Webhook है जो अनुरोध सत्यापित करता है और
    उसे आपके चैनल के इनबाउंड हैंडलर के माध्यम से डिस्पैच करता है:

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // Plugin द्वारा प्रबंधित प्रमाणीकरण (हस्ताक्षर स्वयं सत्यापित करें)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // आपका इनबाउंड हैंडलर संदेश को OpenClaw में डिस्पैच करता है।
          // सटीक वायरिंग आपके प्लेटफ़ॉर्म SDK पर निर्भर करती है -
          // बंडल किए गए Microsoft Teams या Google Chat Plugin पैकेज में वास्तविक उदाहरण देखें।
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      इनबाउंड संदेश प्रबंधन चैनल-विशिष्ट है। प्रत्येक चैनल Plugin अपनी
      इनबाउंड पाइपलाइन का स्वामी होता है। वास्तविक पैटर्न के लिए बंडल किए गए चैनल Plugins
      (उदाहरण के लिए Microsoft Teams या Google Chat Plugin पैकेज) देखें।
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="परीक्षण करें">
`src/channel.test.ts` में साथ स्थित परीक्षण लिखें:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat Plugin", () => {
      it("कॉन्फ़िगरेशन से खाता निर्धारित करता है", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.config.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("सीक्रेट्स को वास्तविक रूप दिए बिना खाते का निरीक्षण करता है", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("अनुपस्थित कॉन्फ़िगरेशन की रिपोर्ट करता है", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test <bundled-plugin-root>/acme-chat/
    ```

    साझा परीक्षण सहायकों के लिए, [परीक्षण](/hi/plugins/sdk-testing) देखें।

</Step>
</Steps>

## फ़ाइल संरचना

```text
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channel मेटाडेटा
├── openclaw.plugin.json      # कॉन्फ़िगरेशन स्कीमा सहित मैनिफ़ेस्ट
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # सार्वजनिक एक्सपोर्ट (वैकल्पिक)
├── runtime-api.ts            # आंतरिक रनटाइम एक्सपोर्ट (वैकल्पिक)
└── src/
    ├── channel.ts            # createChatChannelPlugin के माध्यम से ChannelPlugin
    ├── channel.test.ts       # परीक्षण
    ├── client.ts             # प्लेटफ़ॉर्म API क्लाइंट
    └── runtime.ts            # रनटाइम स्टोर (यदि आवश्यक हो)
```

## उन्नत विषय

<CardGroup cols={2}>
  <Card title="थ्रेडिंग विकल्प" icon="git-branch" href="/hi/plugins/sdk-entrypoints#registration-mode">
    निश्चित, अकाउंट-स्कोप वाले या कस्टम उत्तर मोड
  </Card>
  <Card title="संदेश टूल एकीकरण" icon="puzzle" href="/hi/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool और कार्रवाई की खोज
  </Card>
  <Card title="लक्ष्य निर्धारण" icon="crosshair" href="/hi/plugins/architecture-internals#channel-target-resolution">
    inferTargetChatType, looksLikeId, reservedLiterals, resolveTarget
  </Card>
  <Card title="रनटाइम सहायक" icon="settings" href="/hi/plugins/sdk-runtime">
    api.runtime के माध्यम से TTS, STT, मीडिया और उप-एजेंट
  </Card>
  <Card title="चैनल इनबाउंड API" icon="bolt" href="/hi/plugins/sdk-channel-inbound">
    साझा इनबाउंड इवेंट जीवनचक्र: ग्रहण, निर्धारण, रिकॉर्डिंग, प्रेषण, अंतिम रूप देना
  </Card>
</CardGroup>

<Note>
बंडल किए गए Plugin के रखरखाव और संगतता के लिए कुछ बंडल किए गए सहायक सीम अभी भी
मौजूद हैं। नए चैनल Plugin के लिए यह अनुशंसित प्रतिरूप नहीं है; जब तक आप सीधे
उस बंडल किए गए Plugin परिवार का रखरखाव नहीं कर रहे हों, सामान्य SDK सतह के
जेनेरिक channel/setup/reply/runtime उपपथों को प्राथमिकता दें।
</Note>

## अगले चरण

- [प्रदाता Plugin](/hi/plugins/sdk-provider-plugins) - यदि आपका Plugin मॉडल भी प्रदान करता है
- [SDK अवलोकन](/hi/plugins/sdk-overview) - संपूर्ण उपपथ आयात संदर्भ
- [SDK परीक्षण](/hi/plugins/sdk-testing) - परीक्षण उपयोगिताएँ और अनुबंध परीक्षण
- [Plugin मैनिफ़ेस्ट](/hi/plugins/manifest) - संपूर्ण मैनिफ़ेस्ट स्कीमा

## संबंधित

- [Plugin SDK सेटअप](/hi/plugins/sdk-setup)
- [Plugin बनाना](/hi/plugins/building-plugins)
- [एजेंट हार्नेस Plugin](/hi/plugins/sdk-agent-harness)
