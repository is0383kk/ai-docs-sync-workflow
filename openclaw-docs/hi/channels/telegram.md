---
read_when:
    - Telegram सुविधाओं या वेबहुक पर काम करना
summary: Telegram बॉट समर्थन की स्थिति, क्षमताएँ और कॉन्फ़िगरेशन
title: Telegram
x-i18n:
    generated_at: "2026-07-27T19:11:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f34067478f4a5a71ed8f18503234b4cfcf573ac740aa887b65d13d0e1f09ba54
    source_path: channels/telegram.md
    workflow: 16
---

बॉट DMs और समूहों के लिए grammY के माध्यम से उत्पादन हेतु तैयार। लॉन्ग पोलिंग डिफ़ॉल्ट ट्रांसपोर्ट है; Webhook मोड वैकल्पिक है।

<CardGroup cols={3}>
  <Card title="पेयरिंग" icon="link" href="/hi/channels/pairing">
    Telegram के लिए डिफ़ॉल्ट DM नीति पेयरिंग है।
  </Card>
  <Card title="चैनल समस्या निवारण" icon="wrench" href="/hi/channels/troubleshooting">
    विभिन्न चैनलों के लिए निदान और सुधार कार्यविधियाँ।
  </Card>
  <Card title="Gateway कॉन्फ़िगरेशन" icon="settings" href="/hi/gateway/configuration">
    संपूर्ण चैनल कॉन्फ़िगरेशन पैटर्न और उदाहरण।
  </Card>
</CardGroup>

## त्वरित सेटअप

<Steps>
  <Step title="BotFather में बॉट टोकन बनाएँ">
    दोनों प्रवाहों के अंत में एक टोकन मिलता है, जिसे आप OpenClaw में पेस्ट करते हैं — इनमें से एक चुनें:

    - **चैट प्रवाह**: Telegram खोलें, **@BotFather** से चैट करें (पुष्टि करें कि हैंडल ठीक `@BotFather` ही है), `/newbot` चलाएँ, संकेतों का पालन करें और टोकन सहेजें।
    - **वेब प्रवाह**: [BotFather का वेब ऐप](https://t.me/BotFather?startapp) खोलें — यह [web.telegram.org](https://web.telegram.org) सहित प्रत्येक Telegram क्लाइंट में चलता है — UI में बॉट बनाएँ और उसका टोकन कॉपी करें।

  </Step>

  <Step title="टोकन और DM नीति कॉन्फ़िगर करें">

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

    एनवायरनमेंट फ़ॉलबैक: `TELEGRAM_BOT_TOKEN` (केवल डिफ़ॉल्ट अकाउंट; नामित अकाउंट को `botToken` या `tokenFile` का उपयोग करना होगा)।
    Telegram `openclaw channels login telegram` का उपयोग **नहीं** करता; टोकन को कॉन्फ़िगरेशन/एनवायरनमेंट में सेट करें, फिर Gateway शुरू करें।

  </Step>

  <Step title="Gateway शुरू करें और पहला DM स्वीकृत करें">

```bash
openclaw gateway
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

    पेयरिंग कोड 1 घंटे के बाद समाप्त हो जाते हैं।

  </Step>

  <Step title="बॉट को समूह में जोड़ें">
    बॉट को अपने समूह में जोड़ें, फिर समूह एक्सेस के लिए आवश्यक दो IDs प्राप्त करें:

    - आपकी Telegram उपयोगकर्ता ID, `allowFrom` / `groupAllowFrom` के लिए
    - Telegram समूह चैट ID, `channels.telegram.groups` के अंतर्गत कुंजी के रूप में

    समूह चैट ID को `openclaw logs --follow`, किसी फ़ॉरवर्डेड-ID बॉट या Bot API `getUpdates` से प्राप्त करें। समूह को अनुमति मिलने के बाद, `/whoami@<bot_username>` उपयोगकर्ता और समूह IDs की पुष्टि करता है।

    `-100` से शुरू होने वाली ऋणात्मक सुपरग्रुप IDs समूह चैट IDs होती हैं। वे `groupAllowFrom` में नहीं, बल्कि `channels.telegram.groups` के अंतर्गत जाती हैं।

  </Step>
</Steps>

<Note>
टोकन रिज़ॉल्यूशन अकाउंट-सचेत है: `tokenFile` की प्राथमिकता `botToken` से और उसकी प्राथमिकता एनवायरनमेंट से अधिक है, तथा कॉन्फ़िगरेशन की प्राथमिकता हमेशा `TELEGRAM_BOT_TOKEN` से अधिक होती है (जो केवल डिफ़ॉल्ट अकाउंट के लिए रिज़ॉल्व होता है)। सफल स्टार्टअप के बाद, OpenClaw बॉट की पहचान को अधिकतम 24 घंटे तक कैश करता है, ताकि रीस्टार्ट के समय अतिरिक्त `getMe` कॉल न करनी पड़े; टोकन को बदलने या हटाने से वह कैश साफ़ हो जाता है।
</Note>

## Telegram की ओर की सेटिंग्स

<AccordionGroup>
  <Accordion title="गोपनीयता मोड और समूह दृश्यता">
    Telegram बॉट के लिए डिफ़ॉल्ट रूप से **Privacy Mode** होता है, जो उन्हें प्राप्त होने वाले समूह संदेशों को सीमित करता है।

    सभी समूह संदेश देखने के लिए, इनमें से कोई एक करें:

    - `/setprivacy` के माध्यम से गोपनीयता मोड अक्षम करें, या
    - बॉट को समूह एडमिन बनाएँ।

    गोपनीयता मोड को टॉगल करने के बाद, प्रत्येक समूह में बॉट को हटाकर फिर से जोड़ें, ताकि Telegram परिवर्तन लागू करे।

  </Accordion>

  <Accordion title="समूह अनुमतियाँ">
    एडमिन स्थिति Telegram समूह सेटिंग्स में नियंत्रित होती है। एडमिन बॉट सभी समूह संदेश प्राप्त करते हैं, जो हमेशा सक्रिय समूह व्यवहार के लिए उपयोगी है।
  </Accordion>

  <Accordion title="उपयोगी BotFather टॉगल">

    - `/setjoingroups` — समूह में जोड़ने की अनुमति दें/अस्वीकार करें
    - `/setprivacy` — समूह दृश्यता का व्यवहार

    यदि आप चैट कमांड के बजाय UI पसंद करते हैं, तो यही सेटिंग्स [BotFather के वेब ऐप](https://t.me/BotFather?startapp) में भी उपलब्ध हैं।

  </Accordion>
</AccordionGroup>

## डैशबोर्ड Mini App

Telegram के भीतर OpenClaw डैशबोर्ड खोलने के लिए बॉट के साथ DM में `/dashboard` चलाएँ।

आवश्यकताएँ:

- प्रकाशित HTTPS Mini App URL के लिए `gateway.tailscale.mode: "serve"` या `"funnel"`।
- आपकी संख्यात्मक Telegram उपयोगकर्ता ID चयनित अकाउंट के प्रभावी `allowFrom` या `commands.ownerAllowFrom` में होनी चाहिए।
- DM का उपयोग करें। समूहों में, `/dashboard` का उत्तर `open this in a DM with the bot` होता है और कोई बटन नहीं भेजा जाता।
- Docker इंस्टॉलेशन: Serve/Funnel मोड के लिए Gateway को `tailscaled` के पास लूपबैक से बाइंड करना आवश्यक है, जिसे प्रकाशित पोर्ट वाली ब्रिज नेटवर्किंग पूरा नहीं कर सकती। Gateway कंटेनर को `network_mode: host` के साथ चलाएँ और होस्ट `tailscaled` सॉकेट (`/var/run/tailscale`) तथा `tailscale` CLI को कंटेनर में माउंट करें।

Mini App केवल Tailscale वाला v1 पथ है और Telegram Web iframe का समर्थन नहीं करता।

## एक्सेस नियंत्रण और सक्रियण

### समूह बॉट पहचान

समूहों और फ़ोरम विषयों में, कॉन्फ़िगर किए गए बॉट हैंडल का स्पष्ट उल्लेख (उदाहरण के लिए `@my_bot`) चयनित OpenClaw एजेंट को संबोधित करता है, भले ही एजेंट पर्सोना का नाम Telegram उपयोगकर्ता नाम से अलग हो। असंबंधित ट्रैफ़िक पर समूह मौन नीति फिर भी लागू होती है, लेकिन बॉट हैंडल स्वयं कभी भी "कोई और" नहीं होता।

<Tabs>
  <Tab title="DM नीति">
    `channels.telegram.dmPolicy` सीधे संदेश की पहुँच नियंत्रित करता है:

    - `pairing` (डिफ़ॉल्ट)
    - `allowlist` (`allowFrom` में कम-से-कम एक प्रेषक ID आवश्यक है)
    - `open` (`allowFrom` में `"*"` शामिल होना आवश्यक है)
    - `disabled`

    `allowFrom: ["*"]` के साथ `dmPolicy: "open"` ऐसी किसी भी Telegram खाते को बॉट को आदेश देने देता है जो बॉट उपयोगकर्ता नाम खोज या अनुमान कर लेता है। इसे केवल जानबूझकर सार्वजनिक बनाए गए, कड़ाई से प्रतिबंधित टूल वाले बॉट के लिए उपयोग करें; एकल-स्वामी बॉट को संख्यात्मक उपयोगकर्ता ID के साथ `allowlist` उपयोग करना चाहिए।

    `channels.telegram.allowFrom` संख्यात्मक Telegram उपयोगकर्ता ID स्वीकार करता है। `telegram:` / `tg:` उपसर्ग स्वीकार करके सामान्यीकृत किए जाते हैं।
    बहु-खाता कॉन्फ़िगरेशन में, प्रतिबंधात्मक शीर्ष-स्तरीय `channels.telegram.allowFrom` एक सुरक्षा सीमा है: खाता-स्तरीय `allowFrom: ["*"]` उस खाते को सार्वजनिक नहीं बनाता, जब तक कि मर्ज की गई प्रभावी अनुमति-सूची में फिर भी स्पष्ट वाइल्डकार्ड मौजूद न हो।
    रिक्त `allowFrom` वाला `dmPolicy: "allowlist"` सभी DM अवरुद्ध करता है और कॉन्फ़िगरेशन सत्यापन द्वारा अस्वीकार कर दिया जाता है।
    सेटअप केवल संख्यात्मक उपयोगकर्ता ID माँगता है। यदि आपके कॉन्फ़िगरेशन में पुराने सेटअप की `@username` अनुमति-सूची प्रविष्टियाँ हैं, तो उन्हें संख्यात्मक ID में बदलने के लिए `openclaw doctor --fix` चलाएँ (यथासंभव प्रयास; Telegram बॉट टोकन आवश्यक है)।
    यदि आप पहले पेयरिंग-स्टोर अनुमति-सूची फ़ाइलों पर निर्भर थे, तो `openclaw doctor --fix` अनुमति-सूची प्रवाहों के लिए प्रविष्टियों को `channels.telegram.allowFrom` में पुनर्प्राप्त कर सकता है (उदाहरण के लिए, जब `dmPolicy: "allowlist"` में अभी कोई स्पष्ट ID नहीं है)।

    एकल-स्वामी बॉट के लिए, पिछले पेयरिंग अनुमोदनों पर निर्भर रहने के बजाय स्पष्ट संख्यात्मक `allowFrom` ID के साथ `dmPolicy: "allowlist"` को प्राथमिकता दें।

    सामान्य भ्रम: DM पेयरिंग अनुमोदन का अर्थ यह नहीं है कि "यह प्रेषक हर जगह अधिकृत है।" पेयरिंग केवल DM पहुँच प्रदान करती है। यदि अभी तक कोई कमांड स्वामी मौजूद नहीं है, तो पहला अनुमोदित पेयरिंग `commands.ownerAllowFrom` भी सेट करता है, जिससे केवल-स्वामी कमांड और निष्पादन अनुमोदनों को एक स्पष्ट ऑपरेटर खाता मिलता है। समूह प्रेषक प्राधिकरण अब भी स्पष्ट कॉन्फ़िगरेशन अनुमति-सूचियों से आता है।
    एक पहचान के साथ DM और समूह कमांड, दोनों के लिए अधिकृत होने हेतु: अपनी संख्यात्मक Telegram उपयोगकर्ता ID को `channels.telegram.allowFrom` में रखें, और केवल-स्वामी कमांड के लिए सुनिश्चित करें कि `commands.ownerAllowFrom` में `telegram:<your user id>` शामिल है।

    ### अपनी Telegram उपयोगकर्ता ID ढूँढना

    अधिक सुरक्षित (कोई तृतीय-पक्ष बॉट नहीं): अपने बॉट को DM करें, `openclaw logs --follow` चलाएँ, `from.id` पढ़ें।

    आधिकारिक Bot API विधि:

```bash
curl "https://api.telegram.org/bot<bot_token>/getUpdates"
```

    तृतीय-पक्ष (कम निजी): `@userinfobot` या `@getidsbot`।

  </Tab>

  <Tab title="समूह नीति और अनुमति-सूचियाँ">
    दो नियंत्रण एक साथ लागू होते हैं:

    1. **किन समूहों को अनुमति है** (`channels.telegram.groups`)
       - कोई `groups` कॉन्फ़िगरेशन नहीं, `groupPolicy: "open"`: कोई भी समूह समूह-ID जाँच में सफल होता है
       - कोई `groups` कॉन्फ़िगरेशन नहीं, `groupPolicy: "allowlist"` (डिफ़ॉल्ट): जब तक आप `groups` प्रविष्टियाँ (या `"*"`) नहीं जोड़ते, सभी समूह अवरुद्ध रहते हैं
       - `groups` कॉन्फ़िगर किया गया: अनुमति-सूची के रूप में कार्य करता है (स्पष्ट ID या `"*"`)

    2. **समूहों में किन प्रेषकों को अनुमति है** (`channels.telegram.groupPolicy`)
       - `open` / `allowlist` (डिफ़ॉल्ट) / `disabled`

    `groupAllowFrom` समूह प्रेषकों को फ़िल्टर करता है; यदि यह सेट नहीं है, तो Telegram `allowFrom` पर वापस जाता है (पेयरिंग स्टोर पर नहीं — समूह प्रेषक प्रमाणीकरण कभी भी DM पेयरिंग-स्टोर अनुमोदनों को विरासत में नहीं लेता, जो `2026.2.25` से एक सुरक्षा सीमा है)।
    `groupAllowFrom` प्रविष्टियाँ संख्यात्मक Telegram उपयोगकर्ता ID होनी चाहिए (`telegram:` / `tg:` उपसर्ग सामान्यीकृत किए जाते हैं); गैर-संख्यात्मक प्रविष्टियों को अनदेखा किया जाता है। यहाँ समूह या सुपरग्रुप चैट ID न रखें — ऋणात्मक चैट ID `channels.telegram.groups` के अंतर्गत होनी चाहिए।
    एकल-स्वामी बॉट के लिए व्यावहारिक पैटर्न: अपनी उपयोगकर्ता ID को `channels.telegram.allowFrom` में सेट करें, `groupAllowFrom` को सेट न करें, और लक्षित समूहों को `channels.telegram.groups` के अंतर्गत अनुमति दें।
    यदि `channels.telegram` कॉन्फ़िगरेशन से पूरी तरह अनुपस्थित है, तो रनटाइम डिफ़ॉल्ट रूप से विफलता-पर-बंद `groupPolicy="allowlist"` का उपयोग करता है, जब तक कि `channels.defaults.groupPolicy` स्पष्ट रूप से सेट न हो।

    केवल-स्वामी समूह सेटअप:

```json5
{
  channels: {
    telegram: {
      enabled: true,
      dmPolicy: "pairing",
      allowFrom: ["<YOUR_TELEGRAM_USER_ID>"],
      groupPolicy: "allowlist",
      groups: {
        "<GROUP_CHAT_ID>": {
          requireMention: true,
        },
      },
    },
  },
}
```

    समूह से `@<bot_username> ping` के साथ परीक्षण करें। जब `requireMention: true` हो, तब सामान्य समूह संदेश बॉट को सक्रिय नहीं करते।

    किसी एक विशिष्ट समूह के किसी भी सदस्य को अनुमति दें:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

    किसी एक विशिष्ट समूह में केवल विशिष्ट उपयोगकर्ताओं को अनुमति दें:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          requireMention: true,
          allowFrom: ["8734062810", "745123456"],
        },
      },
    },
  },
}
```

    <Warning>
      सामान्य गलती: `groupAllowFrom` समूह अनुमति-सूची नहीं है।

      - ऋणात्मक Telegram समूह/सुपरग्रुप चैट ID (`-1001234567890`) `channels.telegram.groups` के अंतर्गत जाती हैं।
      - Telegram उपयोगकर्ता ID (`8734062810`) `groupAllowFrom` के अंतर्गत जाती हैं, ताकि यह सीमित किया जा सके कि अनुमत समूह में कौन-से लोग बॉट को सक्रिय कर सकते हैं।
      - अनुमत समूह के किसी भी सदस्य को बॉट से बात करने देने के लिए ही `groupAllowFrom: ["*"]` का उपयोग करें।

    </Warning>

  </Tab>

  <Tab title="उल्लेख व्यवहार">
    समूह उत्तरों के लिए डिफ़ॉल्ट रूप से उल्लेख आवश्यक है। उल्लेख इनमें से आ सकता है:

    - मूल `@botusername` उल्लेख, या
    - `agents.entries.*.groupChat.mentionPatterns` या `messages.groupChat.mentionPatterns` में उल्लेख पैटर्न

    सत्र-स्तरीय टॉगल (केवल स्थिति, स्थायी नहीं): `/activation always`, `/activation mention`। स्थायित्व के लिए कॉन्फ़िगरेशन का उपयोग करें:

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: false },
      },
    },
  },
}
```

    समूह इतिहास संदर्भ हमेशा चालू रहता है और `historyLimit` द्वारा सीमित होता है। समूह इतिहास विंडो अक्षम करने के लिए `channels.telegram.historyLimit: 0` सेट करें। `openclaw doctor --fix` सेवानिवृत्त `includeGroupHistoryContext` कुंजी को हटाता है।

    समूह चैट ID प्राप्त करना: समूह संदेश को `@userinfobot` / `@getidsbot` पर अग्रेषित करें, `openclaw logs --follow` से `chat.id` पढ़ें, Bot API `getUpdates` का निरीक्षण करें, या (समूह को अनुमति मिलने के बाद) `/whoami@<bot_username>` चलाएँ।

  </Tab>
</Tabs>

## रनटाइम व्यवहार

- Telegram Gateway प्रक्रिया के भीतर चलता है।
- रूटिंग निर्धारक है: Telegram से आने वाले संदेशों के उत्तर Telegram पर ही जाते हैं (मॉडल चैनल नहीं चुनता)।
- आने वाले संदेश उत्तर मेटाडेटा, मीडिया प्लेसहोल्डर और Gateway द्वारा देखे गए उत्तरों के लिए सुरक्षित रखे गए उत्तर-श्रृंखला संदर्भ सहित साझा चैनल एनवेलप में सामान्यीकृत होते हैं।
- समूह सत्र समूह ID के अनुसार अलग रखे जाते हैं। फ़ोरम विषयों में `:topic:<threadId>` जोड़ा जाता है।
- DM संदेशों में `message_thread_id` हो सकता है; OpenClaw इसे उत्तरों के लिए बनाए रखता है। DM विषय सत्र केवल तभी विभाजित होते हैं, जब Telegram `getMe` बॉट के लिए `has_topics_enabled: true` रिपोर्ट करता है; अन्यथा DM समतल सत्र पर बने रहते हैं।
- लॉन्ग पोलिंग प्रति-चैट/प्रति-थ्रेड अनुक्रमण के साथ grammY रनर का उपयोग करती है। रनर सिंक समवर्तीता `agents.defaults.maxConcurrent` का उपयोग करती है।
- बहु-अकाउंट स्टार्टअप समवर्ती `getMe` जाँचों को सीमित करता है, ताकि बड़े बॉट समूह हर अकाउंट की जाँच एक साथ शुरू न करें।
- प्रत्येक Gateway प्रक्रिया लॉन्ग पोलिंग को सुरक्षित करती है, ताकि एक समय में केवल एक सक्रिय पोलर बॉट टोकन का उपयोग कर सके। लगातार `getUpdates` 409 टकराव उसी टोकन का उपयोग करने वाले किसी अन्य OpenClaw Gateway, स्क्रिप्ट या बाहरी पोलर की ओर संकेत करते हैं।
- पूर्ण `getUpdates` जीवंतता के बिना 120 सेकंड बीतने पर पोलिंग वॉचडॉग पुनः आरंभ होता है।
- Telegram Bot API में रीड-रसीद समर्थन नहीं है (`sendReadReceipts` लागू नहीं होता)।

<Note>
  `channels.telegram.dm.threadReplies` और `channels.telegram.direct.<chatId>.threadReplies` हटा दिए गए हैं। अपग्रेड के बाद, यदि आपके कॉन्फ़िगरेशन में वे कुंजियाँ अभी भी हैं, तो `openclaw doctor --fix` चलाएँ। DM विषय रूटिंग अब Telegram `getMe.has_topics_enabled` (BotFather के थ्रेडेड मोड द्वारा नियंत्रित) का पालन करती है: विषय-सक्षम बॉट Telegram द्वारा `message_thread_id` भेजे जाने पर थ्रेड-स्कोप वाले DM सत्रों का उपयोग करते हैं; अन्य DM समतल सत्र पर बने रहते हैं।
</Note>

## सुविधा संदर्भ

<AccordionGroup>
  <Accordion title="लाइव स्ट्रीम पूर्वावलोकन (संदेश संपादन)">
    OpenClaw सीधे चैट, समूहों और विषयों में आंशिक उत्तरों को रीयल टाइम में स्ट्रीम करता है: एक पूर्वावलोकन संदेश भेजता है, फिर बार-बार `editMessageText` करता है और उसी स्थान पर अंतिम रूप देता है।

    - `channels.telegram.streaming` का मान `off | partial | block | progress` है (डिफ़ॉल्ट: `partial`)
    - छोटे आरंभिक उत्तर पूर्वावलोकनों को डिबाउंस किया जाता है, फिर यदि रन अभी भी सक्रिय हो तो सीमित विलंब के बाद उन्हें मूर्त रूप दिया जाता है
    - `progress` टूल की प्रगति के लिए एक संपादन योग्य स्थिति ड्राफ़्ट रखता है, टूल की प्रगति से पहले उत्तर गतिविधि आने पर स्थिर स्थिति लेबल दिखाता है, पूर्ण होने पर उसे साफ़ करता है और अंतिम उत्तर सामान्य संदेश के रूप में भेजता है
    - `streaming.preview.toolProgress` नियंत्रित करता है कि टूल/प्रगति अपडेट उसी संपादित पूर्वावलोकन संदेश का दोबारा उपयोग करें या नहीं (डिफ़ॉल्ट: पूर्वावलोकन स्ट्रीमिंग सक्रिय होने पर `true`)
    - `streaming.preview.commandText` उन पंक्तियों में कमांड/निष्पादन विवरण नियंत्रित करता है: `raw` (डिफ़ॉल्ट) या `status` (केवल टूल लेबल)
    - `streaming.progress.commentary` (डिफ़ॉल्ट: `false`) अस्थायी प्रगति ड्राफ़्ट में सहायक की टिप्पणी/प्रस्तावना टेक्स्ट को सक्षम करता है
    - पुराने `channels.telegram.streamMode`, बूलियन `streaming` मान और हटाई गई मूल ड्राफ़्ट पूर्वावलोकन कुंजियों का पता लगाया जाता है; उन्हें माइग्रेट करने के लिए `openclaw doctor --fix` चलाएँ

    टूल-प्रगति पंक्तियाँ वे संक्षिप्त स्थिति अपडेट हैं जो टूल चलने के दौरान दिखाई जाती हैं (कमांड निष्पादन, फ़ाइल पढ़ना, योजना अपडेट, पैच सारांश, ऐप-सर्वर मोड में Codex प्रस्तावना/टिप्पणी)। Telegram इन्हें डिफ़ॉल्ट रूप से चालू रखता है (`v2026.4.22`+ से जारी व्यवहार के अनुरूप)।

    उत्तर-पूर्वावलोकन संपादन बनाए रखें, लेकिन टूल-प्रगति पंक्तियाँ छिपाएँ:

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "partial",
            "preview": { "toolProgress": false }
          }
        }
      }
    }
    ```

    टूल-प्रगति दृश्यमान रखें, लेकिन कमांड/निष्पादन टेक्स्ट छिपाएँ:

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "partial",
            "preview": { "commandText": "status" }
          }
        }
      }
    }
    ```

    `progress` मोड अंतिम उत्तर को उस संदेश में संपादित किए बिना टूल की प्रगति दिखाता है। कमांड-टेक्स्ट नीति को `streaming.progress` के अंतर्गत रखें:

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "progress",
            "progress": {
              "toolProgress": true,
              "commandText": "status"
            }
          }
        }
      }
    }
    ```

    `streaming.mode: "off"` पूर्वावलोकन संपादनों को अक्षम करता है और सामान्य टूल/प्रगति संवाद को अलग स्थिति संदेशों के रूप में भेजने के बजाय दबा देता है; अनुमोदन संकेत, मीडिया और त्रुटियाँ अभी भी सामान्य अंतिम डिलीवरी के माध्यम से भेजी जाती हैं। `streaming.preview.toolProgress: false` केवल उत्तर-पूर्वावलोकन संपादन बनाए रखता है।

    <Note>
      चुने गए उद्धरण वाले उत्तर अपवाद हैं। जब `replyToMode` का मान `first`, `all` या `batched` हो और आने वाले संदेश में चुना गया उद्धरण टेक्स्ट हो, तब OpenClaw उत्तर पूर्वावलोकन संपादित करने के बजाय Telegram के मूल उद्धरण-उत्तर पथ से अंतिम उत्तर भेजता है, इसलिए `streaming.preview.toolProgress` उस बार स्थिति पंक्तियाँ नहीं दिखा सकता। चुने गए उद्धरण टेक्स्ट के बिना वर्तमान संदेश के उत्तर अभी भी स्ट्रीम होते हैं। जब मूल उद्धरण उत्तरों से अधिक महत्वपूर्ण टूल-प्रगति दृश्यता हो, तो `replyToMode: "off"` सेट करें, या इस समझौते को स्वीकार करने के लिए `streaming.preview.toolProgress: false` सेट करें।
    </Note>

    केवल टेक्स्ट वाले उत्तरों के लिए: छोटे पूर्वावलोकनों में अंतिम संपादन उसी स्थान पर होता है; कई संदेशों में विभाजित होने वाले लंबे अंतिम उत्तर पूर्वावलोकन को पहले खंड के रूप में पुनः उपयोग करते हैं, फिर केवल शेष भाग भेजते हैं; प्रगति-मोड के अंतिम उत्तर स्थिति ड्राफ़्ट साफ़ करते हैं और सामान्य अंतिम डिलीवरी का उपयोग करते हैं; यदि पूर्णता की पुष्टि से पहले अंतिम संपादन विफल हो जाता है, तो OpenClaw सामान्य अंतिम डिलीवरी पर वापस जाता है और पुराने पूर्वावलोकन को साफ़ करता है। जटिल उत्तरों (मीडिया पेलोड) के लिए, OpenClaw हमेशा सामान्य अंतिम डिलीवरी पर वापस जाता है और पूर्वावलोकन को साफ़ करता है।

    पूर्वावलोकन स्ट्रीमिंग और ब्लॉक स्ट्रीमिंग परस्पर अनन्य हैं — जब ब्लॉक स्ट्रीमिंग स्पष्ट रूप से सक्षम होती है, तब OpenClaw दोहरी स्ट्रीमिंग से बचने के लिए पूर्वावलोकन स्ट्रीम छोड़ देता है।

    तर्क: `/reasoning stream` जनरेट करते समय तर्क को लाइव पूर्वावलोकन में स्ट्रीम करता है, फिर अंतिम डिलीवरी के बाद तर्क पूर्वावलोकन हटा देता है (इसे दृश्यमान रखने के लिए `/reasoning on` का उपयोग करें)। अंतिम उत्तर तर्क टेक्स्ट के बिना भेजा जाता है।

  </Accordion>

  <Accordion title="समृद्ध संदेश स्वरूपण">
    आउटबाउंड टेक्स्ट डिफ़ॉल्ट रूप से मानक Telegram HTML संदेशों का उपयोग करता है, जो वर्तमान क्लाइंट में पठनीय हैं: बोल्ड, इटैलिक, लिंक, कोड, स्पॉइलर, उद्धरण — Bot API 10.2 के केवल-समृद्ध ब्लॉक (मूल तालिकाएँ, विवरण, समृद्ध मीडिया, सूत्र) नहीं।

    Bot API 10.2 के समृद्ध संदेशों को सक्षम करें:

```json5
{
  channels: {
    telegram: {
      richMessages: true,
    },
  },
}
```

    सक्षम होने पर: एजेंट को बताया जाता है कि इस बॉट/अकाउंट के लिए समृद्ध संदेश उपलब्ध हैं (समर्थित Markdown + HTML-आइलैंड लेखन अनुबंध के साथ); Markdown टेक्स्ट OpenClaw के Markdown IR के माध्यम से टाइप किए गए Bot API 10.2 समृद्ध ब्लॉक (शीर्षक, तालिकाएँ, विवरण, चेकलिस्ट, समृद्ध मीडिया, सूत्र, मानचित्र, कोलाज) के रूप में रेंडर होता है; मीडिया कैप्शन अभी भी Telegram HTML कैप्शन का उपयोग करते हैं (समृद्ध संदेश कैप्शन को प्रतिस्थापित नहीं करते और कैप्शन की सीमा 1024 वर्ण है)।

    इससे मॉडल टेक्स्ट Telegram के समृद्ध-Markdown चिह्नों से दूर रहता है, इसलिए `$400-600K` जैसी मुद्रा को गणित के रूप में पार्स नहीं किया जाता। लंबा समृद्ध टेक्स्ट Telegram की सीमाओं के अनुसार स्वतः विभाजित हो जाता है। 20-कॉलम सीमा से बड़ी तालिकाएँ कोड ब्लॉक पर वापस जाती हैं।

    डिफ़ॉल्ट: क्लाइंट संगतता के लिए बंद — कुछ वर्तमान Desktop, Web, Android और तृतीय-पक्ष क्लाइंट स्वीकृत समृद्ध संदेशों को असमर्थित के रूप में रेंडर करते हैं। जब तक बॉट के साथ उपयोग किया जाने वाला प्रत्येक क्लाइंट उन्हें रेंडर न कर सके, इसे बंद रखें। `/status` दिखाता है कि वर्तमान सत्र में समृद्ध संदेश चालू हैं या बंद।

    लिंक पूर्वावलोकन डिफ़ॉल्ट रूप से चालू हैं। `channels.telegram.linkPreview: false` समृद्ध टेक्स्ट के लिए स्वचालित एंटिटी पहचान अक्षम करता है।

  </Accordion>

  <Accordion title="मूल कमांड और कस्टम कमांड">
    Telegram का कमांड मेनू स्टार्टअप पर `setMyCommands` के साथ पंजीकृत होता है। `commands.native: "auto"` Telegram के लिए मूल कमांड सक्षम करता है।

    कस्टम कमांड मेनू प्रविष्टियाँ जोड़ें:

```json5
{
  channels: {
    telegram: {
      customCommands: [
        { command: "backup", description: "Git बैकअप" },
        { command: "generate", description: "एक छवि बनाएँ" },
      ],
    },
  },
}
```

    नियम: नाम सामान्यीकृत किए जाते हैं (आरंभिक `/` हटाकर, लोअरकेस में); मान्य पैटर्न `a-z`, `0-9`, `_`, लंबाई 1-32; कस्टम कमांड मूल कमांड को ओवरराइड नहीं कर सकते; टकराव/डुप्लिकेट छोड़ दिए जाते हैं और लॉग किए जाते हैं।

    कस्टम कमांड केवल मेनू प्रविष्टियाँ हैं — वे व्यवहार को स्वतः लागू नहीं करते। Plugin/स्किल कमांड टाइप किए जाने पर अभी भी काम कर सकते हैं, भले ही वे Telegram मेनू में न दिखें। यदि मूल कमांड अक्षम हैं, तो अंतर्निहित कमांड हटा दिए जाते हैं; कॉन्फ़िगर किए जाने पर कस्टम/Plugin कमांड अभी भी पंजीकृत हो सकते हैं।

    सामान्य सेटअप विफलताएँ:

    - ट्रिम के पुनः प्रयास के बाद `BOT_COMMANDS_TOO_MUCH` के साथ `setMyCommands failed` का अर्थ है कि मेनू अभी भी सीमा से अधिक है; Plugin/स्किल/कस्टम कमांड कम करें या `channels.telegram.commands.native` अक्षम करें।
    - सीधे Bot API curl कमांड के काम करने के बावजूद `deleteWebhook`, `deleteMyCommands` या `setMyCommands` का `404: Not Found` के साथ विफल होना आम तौर पर दर्शाता है कि `channels.telegram.apiRoot` को पूर्ण `/bot<TOKEN>` एंडपॉइंट पर सेट किया गया था। `apiRoot` केवल Bot API रूट होना चाहिए; `openclaw doctor --fix` गलती से अंत में जुड़े `/bot<TOKEN>` को हटाता है।
    - `getMe returned 401` का अर्थ है कि Telegram ने कॉन्फ़िगर किया गया बॉट टोकन अस्वीकार कर दिया। वर्तमान BotFather टोकन से `botToken`, `tokenFile` या `TELEGRAM_BOT_TOKEN` (डिफ़ॉल्ट अकाउंट) अपडेट करें; OpenClaw पोलिंग से पहले रुक जाता है, इसलिए इसे Webhook क्लीनअप विफलता के रूप में रिपोर्ट नहीं किया जाता।
    - नेटवर्क/फ़ेच त्रुटियों के साथ `setMyCommands failed` का आम तौर पर अर्थ है कि `api.telegram.org` तक आउटबाउंड DNS/HTTPS अवरुद्ध है।

    ### डिवाइस पेयरिंग कमांड (`device-pair` Plugin)

    इंस्टॉल होने पर:

    1. `/pair` एक सेटअप कोड जनरेट करता है
    2. कोड को iOS ऐप में पेस्ट करें
    3. `/pair pending` लंबित अनुरोधों (भूमिका/स्कोप सहित) को सूचीबद्ध करता है
    4. अनुमोदित करें: `/pair approve <requestId>`, `/pair approve` (केवल लंबित अनुरोध), या `/pair approve latest`

    यदि कोई डिवाइस बदले हुए प्रमाणीकरण विवरण (भूमिका, स्कोप, सार्वजनिक कुंजी) के साथ पुनः प्रयास करता है, तो पिछले लंबित अनुरोध को नए `requestId` द्वारा प्रतिस्थापित कर दिया जाता है; अनुमोदन से पहले `/pair pending` दोबारा चलाएँ।

    अधिक विवरण: [पेयरिंग](/hi/channels/pairing#pair-via-telegram)।

  </Accordion>

  <Accordion title="इनलाइन बटन">
    इनलाइन कीबोर्ड स्कोप कॉन्फ़िगर करें:

```json5
{
  channels: {
    telegram: {
      capabilities: {
        inlineButtons: "allowlist",
      },
    },
  },
}
```

    प्रति-अकाउंट ओवरराइड:

```json5
{
  channels: {
    telegram: {
      accounts: {
        main: {
          capabilities: {
            inlineButtons: "allowlist",
          },
        },
      },
    },
  },
}
```

    स्कोप: `off`, `dm`, `group`, `all`, `allowlist` (डिफ़ॉल्ट)। पुराना `capabilities: ["inlineButtons"]`, `"all"` से मैप होता है।

    संदेश क्रिया का उदाहरण:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "एक विकल्प चुनें:",
  buttons: [
    [
      { text: "हाँ", callback_data: "yes" },
      { text: "नहीं", callback_data: "no" },
    ],
    [{ text: "रद्द करें", callback_data: "cancel" }],
  ],
}
```

    Mini App बटन का उदाहरण:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "ऐप खोलें:",
  presentation: {
    blocks: [
      {
        type: "buttons",
        buttons: [{ label: "आरंभ करें", web_app: { url: "https://example.com/app" } }],
      },
    ],
  },
}
```

    `web_app` बटन केवल उपयोगकर्ता और बॉट के बीच निजी चैट में काम करते हैं।

    किसी पंजीकृत Plugin इंटरैक्टिव हैंडलर द्वारा ग्रहण न किए गए कॉलबैक क्लिक एजेंट को टेक्स्ट के रूप में भेजे जाते हैं: `callback_data: <value>`।

  </Accordion>

  <Accordion title="एजेंट और स्वचालन के लिए Telegram संदेश क्रियाएँ">
    क्रियाएँ:

    - `sendMessage` (`to`, `content`, वैकल्पिक `mediaUrl`, `replyToMessageId`, `messageThreadId`)
    - `react` (`chatId`, `messageId`, `emoji`)
    - `deleteMessage` (`chatId`, `messageId`)
    - `editMessage` (`chatId`, `messageId`, `content` या `caption`, वैकल्पिक `presentation` इनलाइन बटन; केवल-बटन वाले संपादन उत्तर मार्कअप को अपडेट करते हैं)
    - `createForumTopic` (`chatId`, `name`, वैकल्पिक `iconColor`, `iconCustomEmojiId`)

    सुविधाजनक उपनाम: `send`, `react`, `delete`, `edit`, `sticker`, `sticker-search`, `topic-create`।

    गेटिंग: `channels.telegram.actions.sendMessage`, `deleteMessage`, `reactions`, `sticker` (डिफ़ॉल्ट: अक्षम)। `edit`, `createForumTopic`, और `editForumTopic` बिना किसी समर्पित टॉगल के डिफ़ॉल्ट रूप से सक्षम हैं।
    रनटाइम भेजने की क्रियाएँ स्टार्टअप/रीलोड से सक्रिय कॉन्फ़िगरेशन/सीक्रेट्स स्नैपशॉट का उपयोग करती हैं, इसलिए कार्रवाई पथ प्रत्येक प्रेषण पर `SecretRef` मानों को फिर से रिज़ॉल्व नहीं करते।

    प्रतिक्रिया हटाने का अर्थ-विज्ञान: [/tools/reactions](/hi/tools/reactions)।

  </Accordion>

  <Accordion title="उत्तर थ्रेडिंग टैग">
    जनरेट किए गए आउटपुट में स्पष्ट उत्तर थ्रेडिंग टैग:

    - `[[reply_to_current]]` — ट्रिगर करने वाले संदेश का उत्तर देता है
    - `[[reply_to:<id>]]` — किसी विशिष्ट संदेश ID का उत्तर देता है

    `channels.telegram.replyToMode`: `off` (डिफ़ॉल्ट), `first`, `all`।

    जब उत्तर थ्रेडिंग सक्षम हो और मूल टेक्स्ट/कैप्शन उपलब्ध हो, तो OpenClaw स्वचालित रूप से मूल उद्धरण अंश जोड़ता है। Telegram मूल उद्धरण टेक्स्ट को 1024 UTF-16 कोड यूनिट तक सीमित करता है; लंबे संदेशों को शुरुआत से उद्धृत किया जाता है और यदि Telegram उद्धरण अस्वीकार करता है, तो सामान्य उत्तर का उपयोग किया जाता है।

    `off` केवल अंतर्निहित उत्तर थ्रेडिंग को अक्षम करता है; स्पष्ट `[[reply_to_*]]` टैग का अब भी पालन किया जाता है।

  </Accordion>

  <Accordion title="फ़ोरम विषय और थ्रेड व्यवहार">
    फ़ोरम सुपरग्रुप: विषय सत्र कुंजियों के अंत में `:topic:<threadId>` जोड़ा जाता है; उत्तर और टाइपिंग विषय थ्रेड को लक्षित करते हैं; विषय कॉन्फ़िगरेशन पथ `channels.telegram.groups.<chatId>.topics.<threadId>` है।

    सामान्य विषय (`threadId=1`) एक विशेष मामला है: संदेश भेजते समय `message_thread_id` छोड़ दिया जाता है (Telegram `sendMessage(...thread_id=1)` को "thread not found" के साथ अस्वीकार करता है), लेकिन टाइपिंग कार्रवाइयों में अब भी `message_thread_id` शामिल होता है (टाइपिंग संकेतक दिखाने के लिए अनुभवजन्य रूप से आवश्यक)।

    विषय प्रविष्टियाँ ओवरराइड न किए जाने पर समूह सेटिंग्स इनहेरिट करती हैं (`requireMention`, `allowFrom`, `skills`, `systemPrompt`, `enabled`, `groupPolicy`)। `agentId` केवल विषय के लिए है और समूह डिफ़ॉल्ट से इनहेरिट नहीं होता। `topics."*"` उस समूह के प्रत्येक विषय के लिए डिफ़ॉल्ट निर्धारित करता है; सटीक विषय ID को अब भी `"*"` पर प्राथमिकता मिलती है।

    **प्रति-विषय एजेंट रूटिंग**: प्रत्येक विषय, विषय कॉन्फ़िगरेशन में `agentId` के माध्यम से किसी अलग एजेंट को रूट कर सकता है, जिससे उसे अपना कार्यस्थान, मेमोरी और सत्र मिलता है:

    ```json5
    {
      channels: {
        telegram: {
          groups: {
            "-1001234567890": {
              topics: {
                "1": { agentId: "main" },      // सामान्य विषय -> मुख्य एजेंट
                "3": { agentId: "zu" },        // डेवलपमेंट विषय -> zu एजेंट
                "5": { agentId: "coder" }      // कोड समीक्षा -> coder एजेंट
              }
            }
          }
        }
      }
    }
    ```

    इसके बाद प्रत्येक विषय की अपनी सत्र कुंजी होती है, उदाहरण के लिए `agent:zu:telegram:group:-1001234567890:topic:3`।

    **स्थायी ACP विषय बाइंडिंग**: फ़ोरम विषय शीर्ष-स्तरीय टाइप की गई बाइंडिंग (`bindings[]` के साथ `type: "acp"`, `match.channel: "telegram"`, `peer.kind: "group"`, और `-1001234567890:topic:42` जैसी विषय-योग्य ID) के माध्यम से ACP हार्नेस सत्रों को पिन कर सकते हैं। वर्तमान में समूहों/सुपरग्रुपों के फ़ोरम विषयों तक सीमित है। [ACP एजेंट](/hi/tools/acp-agents) देखें।

    **चैट से थ्रेड-बाउंड ACP स्पॉन**: `/acp spawn <agent> --thread here|auto` वर्तमान विषय को एक नए ACP सत्र से बाँधता है; अनुवर्ती संदेश सीधे वहाँ रूट होते हैं, और OpenClaw स्पॉन पुष्टिकरण को विषय में पिन करता है। इसे `session.threadBindings.spawnSessions` नियंत्रित करता है (डिफ़ॉल्ट: `true`)।

    टेम्पलेट संदर्भ `MessageThreadId` और `IsForum` उपलब्ध कराता है। `message_thread_id` वाली DM चैट उत्तर मेटाडेटा बनाए रखती हैं, लेकिन थ्रेड-जागरूक सत्र कुंजियों का उपयोग केवल तब करती हैं जब Telegram `getMe`, `has_topics_enabled: true` रिपोर्ट करता है।
    सेवानिवृत्त `dm.threadReplies` और `direct.*.threadReplies` ओवरराइड हटा दिए गए हैं; BotFather का थ्रेडेड मोड सत्य का एकमात्र स्रोत है। पुराने कॉन्फ़िगरेशन कुंजियाँ हटाने के लिए `openclaw doctor --fix` चलाएँ।

  </Accordion>

  <Accordion title="ऑडियो, वीडियो और स्टिकर">
    ### ऑडियो संदेश

    Telegram वॉइस नोट और ऑडियो फ़ाइलों में अंतर करता है। डिफ़ॉल्ट: ऑडियो-फ़ाइल व्यवहार; वॉइस-नोट प्रेषण बाध्य करने के लिए एजेंट के उत्तर में `[[audio_as_voice]]` टैग लगाएँ। इनबाउंड वॉइस-नोट ट्रांसक्रिप्ट को एजेंट संदर्भ में मशीन-जनित, अविश्वसनीय टेक्स्ट के रूप में फ़्रेम किया जाता है, लेकिन उल्लेख पहचान अब भी कच्चे ट्रांसक्रिप्ट का उपयोग करती है, ताकि उल्लेख-गेटेड वॉइस संदेश काम करते रहें।

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/voice.ogg",
  asVoice: true,
}
```

    ### वीडियो संदेश

    Telegram वीडियो फ़ाइलों और वीडियो नोट में अंतर करता है। वीडियो नोट कैप्शन का समर्थन नहीं करते; दिया गया संदेश टेक्स्ट अलग से भेजा जाता है।

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/video.mp4",
  asVideoNote: true,
}
```

    ### स्थान और स्थल

    एक स्वतंत्र `location` ऑब्जेक्ट के साथ मौजूदा `send` कार्रवाई का उपयोग करें। निर्देशांक एक मूल पिन भेजते हैं; `name` और `address` दोनों जोड़ने पर एक मूल स्थल कार्ड भेजा जाता है। स्थान प्रेषण को संदेश टेक्स्ट या मीडिया के साथ संयोजित नहीं किया जा सकता।

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  location: {
    latitude: 48.858844,
    longitude: 2.294351,
    accuracy: 12,
    name: "Eiffel Tower",
    address: "Champ de Mars, Paris",
  },
}
```

    ### स्टिकर

    इनबाउंड: स्थिर WEBP डाउनलोड और प्रोसेस किया जाता है (प्लेसहोल्डर `<media:sticker>`); एनिमेटेड TGS और वीडियो WEBM छोड़ दिए जाते हैं।

    स्टिकर संदर्भ फ़ील्ड: `Sticker.emoji`, `Sticker.setName`, `Sticker.fileId`, `Sticker.fileUniqueId`, `Sticker.cachedDescription`। बार-बार होने वाली विज़न कॉल कम करने के लिए विवरण OpenClaw SQLite Plugin स्थिति में कैश किए जाते हैं।

    स्टिकर कार्रवाइयाँ सक्षम करें:

```json5
{
  channels: {
    telegram: {
      actions: {
        sticker: true,
      },
    },
  },
}
```

    भेजें:

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "123456789",
  fileId: "CAACAgIAAxkBAAI...",
}
```

    कैश किए गए स्टिकर खोजें:

```json5
{
  action: "sticker-search",
  channel: "telegram",
  query: "cat waving",
  limit: 5,
}
```

  </Accordion>

  <Accordion title="प्रतिक्रिया सूचनाएँ">
    Telegram प्रतिक्रियाएँ संदेश पेलोड से अलग `message_reaction` अपडेट के रूप में आती हैं। सक्षम होने पर OpenClaw `Telegram reaction added: 👍 by Alice (@alice) on msg 42` जैसी सिस्टम घटनाओं को कतार में डालता है।

    - `channels.telegram.reactionNotifications`: `off | own | all` (डिफ़ॉल्ट: `own`)
    - `channels.telegram.reactionLevel`: `off | ack | minimal | extensive` (डिफ़ॉल्ट: `minimal`)

    `own` का अर्थ केवल बॉट द्वारा भेजे गए संदेशों पर उपयोगकर्ता प्रतिक्रियाएँ है (भेजे गए संदेशों के कैश के माध्यम से सर्वोत्तम प्रयास)। प्रतिक्रिया घटनाएँ अब भी Telegram अभिगम नियंत्रणों (`dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`) का पालन करती हैं; अनधिकृत प्रेषक हटा दिए जाते हैं।

    Telegram प्रतिक्रिया अपडेट में थ्रेड ID प्रदान नहीं करता: गैर-फ़ोरम समूह, समूह चैट सत्र को रूट होते हैं; फ़ोरम समूह सटीक मूल विषय के बजाय सामान्य-विषय सत्र (`:topic:1`) को रूट होते हैं।

    पोलिंग/Webhook के लिए `allowed_updates` में `message_reaction` स्वचालित रूप से शामिल होता है।

  </Accordion>

  <Accordion title="पावती प्रतिक्रियाएँ">
    जब OpenClaw किसी इनबाउंड संदेश को प्रोसेस करता है, तब `ackReaction` एक पावती इमोजी भेजता है। `messages.ackReactionScope` तय करता है कि इसे *कब* भेजा जाए।

    **इमोजी रिज़ॉल्यूशन क्रम:**

    - `channels.telegram.accounts.<accountId>.ackReaction`
    - `channels.telegram.ackReaction`
    - `messages.ackReaction`
    - एजेंट पहचान इमोजी फ़ॉलबैक (`agents.entries.*.identity.emoji`, अन्यथा "👀")

    Telegram एक यूनिकोड इमोजी की अपेक्षा करता है (उदाहरण के लिए "👀"); किसी चैनल या खाते के लिए प्रतिक्रिया अक्षम करने हेतु `""` का उपयोग करें।

    **दायरा (`messages.ackReactionScope`, डिफ़ॉल्ट `"group-mentions"`; वर्तमान में कोई Telegram-खाता या Telegram-चैनल ओवरराइड नहीं):**

    `all` (DM + समूह, परिवेशी कक्ष घटनाओं सहित), `direct` (केवल DM), `group-all` (परिवेशी कक्ष घटनाओं को छोड़कर प्रत्येक समूह संदेश, कोई DM नहीं), `group-mentions` (जब बॉट का उल्लेख हो तब समूह; **कोई DM नहीं** — डिफ़ॉल्ट), `off` / `none` (अक्षम)।

    <Note>
    डिफ़ॉल्ट दायरा (`group-mentions`) DM या परिवेशी कक्ष घटनाओं में पावती प्रतिक्रियाएँ सक्रिय नहीं करता। DM के लिए `direct` या `all` का उपयोग करें; केवल `all` परिवेशी कक्ष घटनाओं की पावती देता है। यह मान Telegram प्रदाता के स्टार्टअप पर पढ़ा जाता है, इसलिए परिवर्तन प्रभावी करने के लिए Gateway पुनरारंभ आवश्यक है।
    </Note>

  </Accordion>

  <Accordion title="Telegram घटनाओं और कमांड से कॉन्फ़िगरेशन लेखन">
    चैनल कॉन्फ़िगरेशन लेखन डिफ़ॉल्ट रूप से सक्षम है (`configWrites !== false`)। Telegram से ट्रिगर होने वाले लेखनों में समूह माइग्रेशन घटनाएँ (`migrate_to_chat_id`, `channels.telegram.groups` को अपडेट करता है) और `/config set` / `/config unset` (कमांड सक्षम होना आवश्यक) शामिल हैं।

    अक्षम करें:

```json5
{
  channels: {
    telegram: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="लॉन्ग पोलिंग बनाम Webhook">
    डिफ़ॉल्ट लॉन्ग पोलिंग है। Webhook मोड के लिए `channels.telegram.webhookUrl` और `channels.telegram.webhookSecret` सेट करें; वैकल्पिक `webhookPath` (डिफ़ॉल्ट `/telegram-webhook`), `webhookHost` (डिफ़ॉल्ट `127.0.0.1`), `webhookPort` (डिफ़ॉल्ट `8787`), `webhookCertPath` (प्रत्यक्ष-IP या बिना-डोमेन सेटअप के लिए स्वयं-हस्ताक्षरित प्रमाणपत्र PEM)।

    लॉन्ग-पोलिंग मोड में OpenClaw अपना पुनरारंभ वॉटरमार्क केवल किसी अपडेट के सफलतापूर्वक डिस्पैच होने के बाद स्थायी करता है; विफल हैंडलर उस अपडेट को पूर्ण चिह्नित करने के बजाय उसी प्रक्रिया में पुनः प्रयास योग्य बनाए रखता है।

    स्थानीय लिसनर डिफ़ॉल्ट रूप से `127.0.0.1:8787` से बाइंड होता है। सार्वजनिक इनग्रेस के लिए स्थानीय पोर्ट के आगे एक रिवर्स प्रॉक्सी लगाएँ, या जानबूझकर `webhookHost: "0.0.0.0"` सेट करें।

    Webhook मोड अनुरोध गार्ड, Telegram सीक्रेट टोकन और JSON बॉडी को सत्यापित करता है, फिर खाली `200` लौटाने से पहले अपडेट को अपनी टिकाऊ इनग्रेस कतार में कमिट करता है। सफल टिकाऊ अंगीकरण में `x-openclaw-delivery-accepted: durable` शामिल होता है; स्वास्थ्य, रूटिंग, प्रमाणीकरण, सत्यापन और स्टोरेज-त्रुटि प्रतिक्रियाएँ यह हेडर छोड़ देती हैं। रिवर्स प्रॉक्सी और होस्ट नियंत्रक प्रतिक्रिया समय से स्वीकृति का अनुमान लगाए बिना OpenClaw अंगीकरण को सामान्य खाली `200` से अलग करने के लिए इस हेडर की आवश्यकता रख सकते हैं।

    टिकाऊ लेखन के बाद OpenClaw मुख्य चैनल-इनग्रेस ड्रेन के माध्यम से अपडेट का दावा और प्रोसेस करता है (प्रति-चैट/प्रति-विषय लेन, टर्न अंगीकरण पर पूर्ण, अंगीकरण-पूर्व स्टॉल टाइमआउट)। धीमे एजेंट टर्न Telegram की डिलीवरी ACK को रोके नहीं रखते।

  </Accordion>

  <Accordion title="सीमाएँ और CLI लक्ष्य">
    - `channels.telegram.textChunkLimit` डिफ़ॉल्ट 4000; `streaming.chunkMode="newline"` लंबाई के आधार पर विभाजन से पहले अनुच्छेद सीमाओं (रिक्त पंक्तियों) को प्राथमिकता देता है।
    - `channels.telegram.mediaMaxMb` (डिफ़ॉल्ट 100) इनबाउंड और आउटबाउंड मीडिया आकार को सीमित करता है।
    - समूह संदर्भ इतिहास `channels.telegram.historyLimit` या `messages.groupChat.historyLimit` (डिफ़ॉल्ट 50) का उपयोग करता है; `0` इसे अक्षम करता है।
    - जब Gateway ने मूल संदेशों को देखा हो, तब उत्तर/उद्धरण/अग्रेषण का पूरक संदर्भ एक चयनित वार्तालाप संदर्भ विंडो में सामान्यीकृत होता है; देखे गए संदेशों का कैश OpenClaw SQLite Plugin स्थिति में रहता है, और `openclaw doctor --fix` पुराने साइडकार आयात करता है। Telegram प्रत्येक अपडेट में केवल एक उथला `reply_to_message` शामिल करता है, इसलिए कैश से पुरानी शृंखलाएँ उस पेलोड तक सीमित रहती हैं।
    - Telegram अनुमति-सूचियाँ मुख्य रूप से यह नियंत्रित करती हैं कि एजेंट को कौन सक्रिय कर सकता है, न कि पूर्ण पूरक-संदर्भ संशोधन सीमा।
    - DM इतिहास: `channels.telegram.dmHistoryLimit`, `channels.telegram.dms["<user_id>"].historyLimit`।

    CLI और संदेश-टूल के प्रेषण लक्ष्य संख्यात्मक चैट ID, उपयोगकर्ता नाम या फ़ोरम विषय लक्ष्य स्वीकार करते हैं:

```bash
openclaw message send --channel telegram --target 123456789 --message "hi"
openclaw message send --channel telegram --target @name --message "hi"
openclaw message send --channel telegram --target -1001234567890:topic:42 --message "hi topic"
```

    पोल `openclaw message poll` का उपयोग करते हैं और फ़ोरम विषयों का समर्थन करते हैं:

```bash
openclaw message poll --channel telegram --target 123456789 \
  --poll-question "Ship it?" --poll-option "Yes" --poll-option "No"
openclaw message poll --channel telegram --target -1001234567890:topic:42 \
  --poll-question "Pick a time" --poll-option "10am" --poll-option "2pm" \
  --poll-duration-seconds 300 --poll-public
```

    केवल Telegram के पोल फ़्लैग: `--poll-duration-seconds` (5-600), `--poll-anonymous`, `--poll-public`, `--thread-id` (या एक `:topic:` लक्ष्य)। `--poll-option` 2-12 बार दोहराता है (Telegram की विकल्प सीमा)।

    Telegram प्रेषण इनलाइन कीबोर्ड के लिए `buttons` ब्लॉक के साथ `--presentation` का भी समर्थन करता है (जब `channels.telegram.capabilities.inlineButtons` इसकी अनुमति देता है), बॉट द्वारा उस चैट में पिन कर सकने पर पिन की गई डिलीवरी का अनुरोध करने के लिए `--pin` या `--delivery '{"pin":true}'`, और आउटबाउंड चित्रों, GIF और वीडियो को संपीड़ित/एनिमेटेड/वीडियो अपलोड के बजाय दस्तावेज़ों के रूप में भेजने के लिए `--force-document` का समर्थन करता है।

    क्रिया नियंत्रण: `channels.telegram.actions.sendMessage=false` पोल सहित सभी आउटबाउंड संदेशों को अक्षम करता है; `channels.telegram.actions.poll=false` नियमित प्रेषण सक्षम रखते हुए पोल निर्माण अक्षम करता है।

  </Accordion>

  <Accordion title="Telegram में निष्पादन स्वीकृतियाँ">
    Telegram अनुमोदक DM में निष्पादन स्वीकृतियों का समर्थन करता है और वैकल्पिक रूप से आरंभिक चैट या विषय में संकेत पोस्ट कर सकता है। अनुमोदक संख्यात्मक Telegram उपयोगकर्ता ID होने चाहिए।

    - `channels.telegram.execApprovals.enabled` (कम-से-कम एक अनुमोदक का समाधान संभव होने पर `"auto"` सक्षम करता है)
    - `channels.telegram.execApprovals.approvers` (`commands.ownerAllowFrom` के संख्यात्मक स्वामी ID पर वापस लौटता है)
    - `channels.telegram.execApprovals.target`: `dm` (डिफ़ॉल्ट) | `channel` | `both`
    - `agentFilter`, `sessionFilter`

    `channels.telegram.allowFrom`, `groupAllowFrom`, और `defaultTo` नियंत्रित करते हैं कि बॉट से कौन बात कर सकता है और वह सामान्य उत्तर कहाँ भेजता है — वे किसी को निष्पादन अनुमोदक नहीं बनाते। जब अभी तक कोई कमांड स्वामी मौजूद न हो, तब पहली स्वीकृत DM पेयरिंग `commands.ownerAllowFrom` को बूटस्ट्रैप करती है, जिससे एक-स्वामी सेटअप `execApprovals.approvers` के अंतर्गत ID दोहराए बिना काम करते हैं।

    चैनल डिलीवरी चैट में कमांड पाठ दिखाती है; `channel` या `both` केवल विश्वसनीय समूहों/विषयों में सक्षम करें। जब संकेत किसी फ़ोरम विषय में पहुँचता है, OpenClaw स्वीकृति संकेत और अनुवर्ती कार्रवाई के लिए विषय को बनाए रखता है। निष्पादन स्वीकृतियाँ डिफ़ॉल्ट रूप से 30 मिनट बाद समाप्त हो जाती हैं।

    इनलाइन स्वीकृति बटनों के लिए यह भी आवश्यक है कि `channels.telegram.capabilities.inlineButtons` लक्ष्य सतह (`dm`, `group`, या `all`) को अनुमति दे। `plugin:` उपसर्ग वाली स्वीकृति ID का समाधान Plugin स्वीकृतियों के माध्यम से होता है; अन्य का समाधान पहले निष्पादन स्वीकृतियों के माध्यम से होता है।

    [निष्पादन स्वीकृतियाँ](/hi/tools/exec-approvals) देखें।

  </Accordion>
</AccordionGroup>

## त्रुटि उत्तर नियंत्रण

जब एजेंट को डिलीवरी या प्रदाता त्रुटि मिलती है, तो त्रुटि नीति नियंत्रित करती है कि त्रुटि संदेश Telegram चैट तक पहुँचें या नहीं:

| कुंजी                             | मान                     | डिफ़ॉल्ट  | विवरण                                                                                                                                                                |
| ------------------------------- | -------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `channels.telegram.errorPolicy` | `always`, `once`, `silent` | `always` | `always` प्रत्येक त्रुटि संदेश चैट में भेजता है। `once` प्रत्येक विशिष्ट त्रुटि संदेश को अंतर्निहित कूलडाउन विंडो में एक बार भेजता है। `silent` चैट में कभी भी त्रुटि संदेश नहीं भेजता। |

प्रति-खाता, प्रति-समूह और प्रति-विषय ओवरराइड समर्थित हैं (अन्य Telegram कॉन्फ़िगरेशन कुंजियों के समान विरासत)।

```json5
{
  channels: {
    telegram: {
      errorPolicy: "always",
      groups: {
        "-1001234567890": {
          errorPolicy: "silent", // इस समूह में त्रुटियाँ दबाएँ
        },
      },
    },
  },
}
```

## समस्या निवारण

<AccordionGroup>
  <Accordion title="बॉट उल्लेख-रहित समूह संदेशों का उत्तर नहीं देता">

    - यदि `requireMention=false`, तो Telegram गोपनीयता मोड को पूर्ण दृश्यता की अनुमति देनी चाहिए: BotFather `/setprivacy` -> Disable, फिर बॉट को समूह से हटाकर दोबारा जोड़ें।
    - जब कॉन्फ़िगरेशन उल्लेख-रहित समूह संदेशों की अपेक्षा करता है, तब `openclaw channels status` चेतावनी देता है।
    - `openclaw channels status --probe` स्पष्ट संख्यात्मक समूह ID की जाँच करता है; वाइल्डकार्ड `"*"` की सदस्यता जाँची नहीं जा सकती।
    - त्वरित सत्र परीक्षण: `/activation always`।

  </Accordion>

  <Accordion title="बॉट को समूह संदेश बिल्कुल नहीं दिखते">

    - जब `channels.telegram.groups` मौजूद हो, तो समूह सूचीबद्ध होना चाहिए (या `"*"` शामिल करें)।
    - समूह में बॉट की सदस्यता सत्यापित करें।
    - छोड़े जाने के कारणों के लिए `openclaw logs --follow` की समीक्षा करें।

  </Accordion>

  <Accordion title="कमांड आंशिक रूप से काम करते हैं या बिल्कुल नहीं करते">

    - अपनी प्रेषक पहचान अधिकृत करें (पेयरिंग और/या संख्यात्मक `allowFrom`); समूह नीति `open` होने पर भी कमांड प्राधिकरण लागू होता है।
    - `BOT_COMMANDS_TOO_MUCH` के साथ `setMyCommands failed` का अर्थ है कि मूल मेनू में बहुत अधिक प्रविष्टियाँ हैं; Plugin/Skill/कस्टम कमांड कम करें या मूल मेनू अक्षम करें।
    - `deleteMyCommands` / `setMyCommands` स्टार्टअप कॉल और `sendChatAction` टाइपिंग कॉल सीमित हैं और अनुरोध टाइमआउट पर Telegram के परिवहन फ़ॉलबैक के माध्यम से एक बार पुनः प्रयास करते हैं। लगातार नेटवर्क/फ़ेच त्रुटियों का सामान्यतः अर्थ है कि `api.telegram.org` तक DNS/HTTPS पहुँच उपलब्ध नहीं है।

  </Accordion>

  <Accordion title="स्टार्टअप अनधिकृत टोकन की रिपोर्ट करता है">

    - `getMe returned 401` कॉन्फ़िगर किए गए बॉट टोकन के लिए Telegram प्रमाणीकरण विफलता है। BotFather में टोकन दोबारा कॉपी या पुनः जनरेट करें, फिर `channels.telegram.botToken`, `tokenFile`, `accounts.<id>.botToken`, या `TELEGRAM_BOT_TOKEN` (डिफ़ॉल्ट खाता) अपडेट करें।
    - स्टार्टअप के दौरान `deleteWebhook 401 Unauthorized` भी प्रमाणीकरण विफलता है; इसे "कोई webhook मौजूद नहीं है" मानने से वही खराब-टोकन विफलता केवल बाद की API कॉल तक टलेगी।

  </Accordion>

  <Accordion title="पोलिंग या नेटवर्क अस्थिरता">

    - यदि `AbortSignal` प्रकार मेल न खाएँ, तो कस्टम फ़ेच/प्रॉक्सी के साथ Node 22+ तत्काल निरस्तीकरण व्यवहार सक्रिय कर सकता है।
    - कुछ होस्ट `api.telegram.org` को पहले IPv6 में समाधान करते हैं; खराब IPv6 निर्गमन रुक-रुककर API विफलताएँ उत्पन्न करता है।
    - `TypeError: fetch failed` या `Network request for 'getUpdates' failed!` वाले लॉग को पुनर्प्राप्ति-योग्य नेटवर्क त्रुटियों के रूप में पुनः प्रयास किया जाता है।
    - पोलिंग स्टार्टअप के दौरान OpenClaw grammY के लिए सफल स्टार्टअप `getMe` जाँच का पुनः उपयोग करता है, जिससे रनर को पहले `getUpdates` से पहले दूसरे `getMe` की आवश्यकता नहीं होती।
    - यदि पोलिंग स्टार्टअप के दौरान `deleteWebhook` अस्थायी नेटवर्क त्रुटि के साथ विफल होता है, तो OpenClaw एक और प्री-पोल नियंत्रण-प्लेन कॉल करने के बजाय लंबी पोलिंग जारी रखता है। तब भी सक्रिय webhook `getUpdates` विरोध के रूप में सामने आता है; OpenClaw परिवहन को फिर से बनाता है और webhook सफ़ाई का पुनः प्रयास करता है।
    - लॉग में `Polling stall detected` का अर्थ है कि डिफ़ॉल्ट रूप से 120 सेकंड तक पूर्ण लंबी-पोल सक्रियता न मिलने पर OpenClaw पोलिंग पुनः आरंभ करता है और परिवहन को फिर से बनाता है।
    - `openclaw channels status --probe` और `openclaw doctor` तब चेतावनी देते हैं जब किसी चल रहे पोलिंग खाते ने स्टार्टअप अनुग्रह अवधि के बाद `getUpdates` पूरा नहीं किया हो, किसी चल रहे webhook खाते ने स्टार्टअप अनुग्रह अवधि के बाद `setWebhook` पूरा नहीं किया हो, या अंतिम सफल पोलिंग परिवहन गतिविधि पुरानी हो।
    - Telegram Bot API परिवहन के लिए प्रक्रिया प्रॉक्सी परिवेश का सम्मान करता है: `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, और छोटे अक्षरों वाले प्रकार। `NO_PROXY` / `no_proxy` फिर भी `api.telegram.org` को बायपास कर सकते हैं।
    - यदि सेवा परिवेश के लिए `OPENCLAW_PROXY_URL` सेट है और कोई मानक प्रॉक्सी परिवेश मौजूद नहीं है, तो Telegram Bot API परिवहन के लिए भी उस URL का उपयोग करता है।
    - अस्थिर प्रत्यक्ष निर्गमन/TLS वाले VPS होस्ट पर Telegram API कॉल को प्रॉक्सी के माध्यम से रूट करें:

```yaml
channels:
  telegram:
    proxy: socks5://<user>:<password>@proxy-host:1080
```

    - Node 22+ का डिफ़ॉल्ट `autoSelectFamily=true` है (WSL2 को छोड़कर)। Telegram DNS परिणाम क्रम पहले `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER`, फिर `channels.telegram.network.dnsResultOrder`, फिर प्रक्रिया डिफ़ॉल्ट (उदाहरण के लिए `NODE_OPTIONS=--dns-result-order=ipv4first`) का सम्मान करता है; इनमें से कोई लागू न होने पर Node 22+ में `ipv4first` पर वापस लौटता है।
    - WSL2 पर, या जब केवल-IPv4 व्यवहार बेहतर काम करता हो, तब फ़ैमिली चयन बाध्य करें:

```yaml
channels:
  telegram:
    network:
      autoSelectFamily: false
```

    - RFC 2544 बेंचमार्क-रेंज उत्तर (`198.18.0.0/15`) Telegram मीडिया डाउनलोड के लिए डिफ़ॉल्ट रूप से पहले से अनुमत हैं। यदि कोई विश्वसनीय फ़ेक-IP या पारदर्शी प्रॉक्सी मीडिया डाउनलोड के दौरान `api.telegram.org` को किसी अन्य निजी/आंतरिक/विशेष-उपयोग पते में फिर से लिखता है, तो केवल Telegram वाले बायपास को स्वीकृति दें:

```yaml
channels:
  telegram:
    network:
      dangerouslyAllowPrivateNetwork: true
```

    - यही स्वीकृति प्रति खाते `channels.telegram.accounts.<accountId>.network.dangerouslyAllowPrivateNetwork` पर उपलब्ध है।
    - यदि आपका प्रॉक्सी Telegram मीडिया होस्ट को `198.18.x.x` में समाधान करता है, तो पहले खतरनाक फ़्लैग बंद रखें — वह रेंज डिफ़ॉल्ट रूप से पहले से अनुमत है।

    <Warning>
      `channels.telegram.network.dangerouslyAllowPrivateNetwork` Telegram मीडिया SSRF सुरक्षा को कमज़ोर करता है। इसका उपयोग केवल विश्वसनीय ऑपरेटर-नियंत्रित प्रॉक्सी परिवेशों (Clash, Mihomo, Surge फ़ेक-IP रूटिंग) के लिए करें, जो RFC 2544 बेंचमार्क रेंज के बाहर निजी या विशेष-उपयोग उत्तर बनाते हैं। सामान्य सार्वजनिक इंटरनेट Telegram पहुँच के लिए इसे बंद रखें।
    </Warning>

    - अस्थायी परिवेश ओवरराइड: `OPENCLAW_TELEGRAM_DISABLE_AUTO_SELECT_FAMILY=1`, `OPENCLAW_TELEGRAM_ENABLE_AUTO_SELECT_FAMILY=1`, `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER=ipv4first`।
    - DNS उत्तर सत्यापित करें:

```bash
dig +short api.telegram.org A
dig +short api.telegram.org AAAA
```

  </Accordion>
</AccordionGroup>

अधिक सहायता: [चैनल समस्या निवारण](/hi/channels/troubleshooting)।

## कॉन्फ़िगरेशन संदर्भ

प्राथमिक संदर्भ: [कॉन्फ़िगरेशन संदर्भ - Telegram](/hi/gateway/config-channels#telegram)।

<Accordion title="उच्च-संकेत Telegram फ़ील्ड">

- स्टार्टअप/प्रमाणीकरण: `enabled`, `botToken`, `tokenFile` (एक नियमित फ़ाइल होनी चाहिए; सिमलिंक अस्वीकार किए जाते हैं), `accounts.*`
- पहुँच नियंत्रण: `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`, `groups.*.topics.*`, शीर्ष-स्तरीय `bindings[]` (`type: "acp"`)
- विषय डिफ़ॉल्ट: `groups.<chatId>.topics."*"` बेमेल फ़ोरम विषयों पर लागू होता है; सटीक विषय ID इसे ओवरराइड करते हैं
- निष्पादन अनुमोदन: `execApprovals`, `accounts.*.execApprovals`
- कमांड/मेन्यू: `commands.native`, `commands.nativeSkills`, `customCommands`
- थ्रेडिंग/उत्तर: `replyToMode`, `threadBindings`
- स्ट्रीमिंग: `streaming` (मोड `off | partial | block | progress`), `streaming.preview.toolProgress`
- फ़ॉर्मैटिंग/डिलीवरी: `textChunkLimit`, `streaming.chunkMode`, `richMessages`, `markdown.tables` (`off | bullets | code | block`), `linkPreview`, `responsePrefix`
- मीडिया/नेटवर्क: `mediaMaxMb`, `network.autoSelectFamily`, `network.dangerouslyAllowPrivateNetwork`, `proxy`
- कस्टम API रूट: `apiRoot` (केवल Bot API रूट; `/bot<TOKEN>` शामिल न करें), `trustedLocalFileRoots` (स्वयं होस्ट किए गए Bot API के निरपेक्ष `file_path` रूट)
- Webhook: `webhookUrl`, `webhookSecret`, `webhookPath`, `webhookHost`, `webhookPort`, `webhookCertPath`
- कार्रवाइयाँ/क्षमताएँ: `capabilities.inlineButtons`, `actions.sendMessage|editMessage|deleteMessage|reactions|sticker|createForumTopic|editForumTopic`
- प्रतिक्रियाएँ: `reactionNotifications`, `reactionLevel`
- त्रुटियाँ: `errorPolicy`, `silentErrorReplies`
- लेखन/इतिहास: `configWrites`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`

</Accordion>

<Note>
बहु-खाता प्राथमिकता: दो या अधिक खाता ID कॉन्फ़िगर होने पर, डिफ़ॉल्ट रूटिंग को स्पष्ट बनाने के लिए `channels.telegram.defaultAccount` सेट करें (या `channels.telegram.accounts.default` शामिल करें)। अन्यथा OpenClaw पहले सामान्यीकृत खाता ID का उपयोग करता है और `openclaw doctor` चेतावनी देता है। नामित खातों को `channels.telegram.allowFrom` / `groupAllowFrom` विरासत में मिलते हैं, लेकिन `accounts.default.*` के मान नहीं।
</Note>

## संबंधित

<CardGroup cols={2}>
  <Card title="पेयरिंग" icon="link" href="/hi/channels/pairing">
    Telegram उपयोगकर्ता को Gateway से पेयर करें।
  </Card>
  <Card title="समूह" icon="users" href="/hi/channels/groups">
    समूह और विषय की अनुमति-सूची का व्यवहार।
  </Card>
  <Card title="चैनल रूटिंग" icon="route" href="/hi/channels/channel-routing">
    आने वाले संदेशों को एजेंटों तक रूट करें।
  </Card>
  <Card title="सुरक्षा" icon="shield" href="/hi/gateway/security">
    खतरा मॉडल और सुदृढ़ीकरण।
  </Card>
  <Card title="बहु-एजेंट रूटिंग" icon="sitemap" href="/hi/concepts/multi-agent">
    समूहों और विषयों को एजेंटों से मैप करें।
  </Card>
  <Card title="समस्या निवारण" icon="wrench" href="/hi/channels/troubleshooting">
    विभिन्न चैनलों में निदान।
  </Card>
</CardGroup>
