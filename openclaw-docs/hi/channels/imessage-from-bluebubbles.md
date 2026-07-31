---
read_when:
    - BlueBubbles से बंडल किए गए iMessage Plugin पर स्थानांतरण की योजना बनाना
    - BlueBubbles कॉन्फ़िगरेशन कुंजियों का iMessage समकक्षों में अनुवाद करना
    - iMessage Plugin सक्षम करने से पहले imsg का सत्यापन करना
summary: 'पुराने BlueBubbles कॉन्फ़िगरेशन को बंडल किए गए iMessage Plugin में माइग्रेट करें: कुंजी मैपिंग, समूह अनुमति-सूची गेट और कटओवर सत्यापन।'
title: BlueBubbles से आ रहे हैं
x-i18n:
    generated_at: "2026-07-27T17:41:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5984ad1319b4bb3060496666bea6de663eba0105a89f82d13030c015c5df159d
    source_path: channels/imessage-from-bluebubbles.md
    workflow: 16
---

BlueBubbles समर्थन हटा दिया गया है। OpenClaw अब iMessage का समर्थन केवल बंडल किए गए `imessage` plugin के माध्यम से करता है, जो JSON-RPC पर [`steipete/imsg`](https://github.com/steipete/imsg) चलाता है और उसी निजी API सतह तक पहुँचता है जो BlueBubbles के पास थी (`react`, `edit`, `unsend`, `reply`, `sendWithEffect`, मूल पोल, समूह प्रबंधन, अटैचमेंट)। एक CLI बाइनरी BlueBubbles सर्वर + क्लाइंट ऐप + webhook व्यवस्था की जगह लेती है: कोई REST एंडपॉइंट नहीं, कोई webhook प्रमाणीकरण नहीं।

यह मार्गदर्शिका पुराने `channels.bluebubbles` कॉन्फ़िगरेशन को `channels.imessage` में माइग्रेट करती है। कोई अन्य समर्थित माइग्रेशन पथ नहीं है। वर्तमान OpenClaw में बचा हुआ `channels.bluebubbles` ब्लॉक निष्क्रिय है — कोई रनटाइम इसे नहीं पढ़ता।

<Note>
संक्षिप्त घोषणा और ऑपरेटर सारांश के लिए, [BlueBubbles को हटाना और imsg iMessage पथ](/hi/announcements/bluebubbles-imessage) देखें।
</Note>

## माइग्रेशन चेकलिस्ट

जब आपको अपना पुराना BlueBubbles कॉन्फ़िगरेशन पहले से पता हो, तब सबसे छोटा सुरक्षित पथ:

1. Messages.app चलाने वाले Mac पर सीधे `imsg` सत्यापित करें (`imsg chats`, `imsg history`, `imsg send`, `imsg rpc --help`)।
2. `channels.bluebubbles` से `channels.imessage` में व्यवहार कुंजियाँ कॉपी करें: `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`, `includeAttachments`, `attachmentRoots`, `mediaMaxMb`, `textChunkLimit`, और `actions`।
3. वे ट्रांसपोर्ट कुंजियाँ हटा दें जो अब मौजूद नहीं हैं: `serverUrl`, `password`, webhook URL और BlueBubbles सर्वर सेटअप।
4. यदि Gateway, Messages वाले Mac पर नहीं चल रहा है, तो `channels.imessage.cliPath` को SSH रैपर पर सेट करें और दूरस्थ अटैचमेंट प्राप्त करने के लिए `remoteHost` सेट करें।
5. `channels.imessage` सक्षम करें, Gateway पुनः आरंभ करें, फिर `openclaw channels status --probe --channel imessage` चलाएँ।
6. एक DM, एक अनुमत समूह, सक्षम होने पर अटैचमेंट और हर उस निजी API कार्रवाई का परीक्षण करें जिसका एजेंट द्वारा उपयोग अपेक्षित है।
7. iMessage पथ सत्यापित होने के बाद BlueBubbles सर्वर और पुराना `channels.bluebubbles` कॉन्फ़िगरेशन हटा दें।

## imsg क्या करता है

`imsg`, Messages के लिए एक स्थानीय macOS CLI है। OpenClaw, `imsg rpc` को चाइल्ड प्रोसेस के रूप में शुरू करता है और stdin/stdout पर JSON-RPC के माध्यम से संचार करता है। उजागर करने के लिए कोई HTTP सर्वर, webhook URL, पृष्ठभूमि डेमन, लॉन्च एजेंट या पोर्ट नहीं है।

- रीड-ओनली SQLite हैंडल का उपयोग करके `~/Library/Messages/chat.db` से पठन होता है।
- लाइव इनबाउंड संदेश `imsg watch` / `watch.subscribe` से आते हैं, जो पोलिंग फ़ॉलबैक के साथ `chat.db` फ़ाइल सिस्टम घटनाओं का अनुसरण करता है।
- सामान्य टेक्स्ट और फ़ाइल भेजने के लिए Messages.app ऑटोमेशन का उपयोग होता है।
- उन्नत कार्रवाइयाँ, Messages.app में `imsg` सहायक को इंजेक्ट करने के लिए `imsg launch` का उपयोग करती हैं। यही पठन रसीदों, टाइपिंग संकेतकों, रिच प्रेषण, संपादन, प्रेषण रद्द करने, थ्रेडेड उत्तर, टैपबैक, पोल और समूह प्रबंधन को सक्षम करता है।
- Linux बिल्ड कॉपी किए गए `chat.db` का निरीक्षण कर सकते हैं, लेकिन संदेश भेज नहीं सकते, लाइव Mac डेटाबेस नहीं देख सकते या Messages.app को नियंत्रित नहीं कर सकते। OpenClaw iMessage के लिए, साइन-इन किए हुए Mac पर या उस Mac के SSH रैपर के माध्यम से `imsg` चलाएँ।

## शुरू करने से पहले

1. Messages.app चलाने वाले Mac पर `imsg` इंस्टॉल करें:

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg chats --limit 3
   ```

   सामान्य स्थानीय सेटअप के लिए, OpenClaw सेटअप साइन-इन किए हुए Messages Mac पर `imsg` के लिए उपयोगकर्ता-पुष्टि वाला Homebrew इंस्टॉल या अपडेट प्रस्तुत कर सकता है। मैन्युअल सेटअप और SSH-रैपर टोपोलॉजी ऑपरेटर द्वारा प्रबंधित रहती हैं: उसी स्थानीय या दूरस्थ उपयोगकर्ता संदर्भ में Homebrew अपडेट दोहराएँ जो `imsg` चलाएगा। यदि `imsg chats`, `unable to open database file`, खाली आउटपुट या `authorization denied` के साथ विफल होता है, तो `imsg` लॉन्च करने वाले टर्मिनल, एडिटर, Node प्रोसेस, Gateway सेवा या SSH पैरेंट प्रोसेस को Full Disk Access प्रदान करें, फिर उस पैरेंट प्रोसेस को दोबारा खोलें।

2. OpenClaw कॉन्फ़िगरेशन बदलने से पहले पठन, निगरानी, प्रेषण और RPC सतहों को सत्यापित करें:

   ```bash
   imsg chats --limit 10 --json | jq -s
   imsg history --chat-id 42 --limit 10 --attachments --json | jq -s
   imsg watch --chat-id 42 --reactions --json
   imsg send --chat-id 42 --text "OpenClaw imsg test"
   imsg rpc --help
   ```

   `42` को `imsg chats` से प्राप्त वास्तविक चैट आईडी से बदलें। भेजने के लिए Messages.app की Automation अनुमति आवश्यक है। यदि OpenClaw को SSH के माध्यम से चलाया जाएगा, तो ये कमांड उसी SSH रैपर या उपयोगकर्ता संदर्भ के माध्यम से चलाएँ जिसका OpenClaw उपयोग करेगा। यदि पठन काम करता है लेकिन प्रेषण AppleEvents `-1743` के साथ विफल होता है, तो जाँचें कि Automation, `/usr/libexec/sshd-keygen-wrapper` पर लागू हुआ है या नहीं; [SSH रैपर प्रेषण AppleEvents -1743 के साथ विफल होता है](/hi/channels/imessage#requirements-and-permissions-macos) देखें।

3. निजी API ब्रिज सक्षम करें। OpenClaw iMessage के लिए इसकी पुरज़ोर अनुशंसा की जाती है, क्योंकि उत्तर, टैपबैक, प्रभाव, पोल, अटैचमेंट उत्तर और समूह कार्रवाइयाँ इस पर निर्भर करती हैं:

   ```bash
   imsg launch
   imsg status --json
   ```

   `imsg launch` के लिए SIP अक्षम होना आवश्यक है (और आधुनिक macOS पर लाइब्रेरी सत्यापन शिथिल होना चाहिए — [imsg निजी API सक्षम करना](/hi/channels/imessage#enabling-the-imsg-private-api) देखें)। बुनियादी प्रेषण, इतिहास और निगरानी `imsg launch` के बिना काम करते हैं; पूर्ण OpenClaw iMessage कार्रवाई सतह काम नहीं करती।

4. `channels.imessage` सक्षम करने और Gateway शुरू करने के बाद, OpenClaw के माध्यम से ब्रिज सत्यापित करें:

   ```bash
   openclaw channels status --probe
   ```

   iMessage खाते को `works` रिपोर्ट करना चाहिए; `--json` के साथ, जाँच पेलोड में `privateApi.available: true` शामिल होता है। यदि यह `false` रिपोर्ट करता है, तो पहले उसे ठीक करें — [क्षमता पहचान](/hi/channels/imessage#private-api-actions) देखें। जाँच के लिए पहुँच योग्य Gateway आवश्यक है (अन्यथा CLI केवल-कॉन्फ़िगरेशन आउटपुट पर लौट जाता है) और यह केवल कॉन्फ़िगर किए गए, सक्षम खातों की जाँच करता है।

5. अपने कॉन्फ़िगरेशन का स्नैपशॉट लें:

   ```bash
   cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
   ```

## कॉन्फ़िगरेशन रूपांतरण

iMessage और BlueBubbles अधिकांश चैनल-स्तरीय व्यवहार कुंजियाँ साझा करते हैं। जो बदलता है वह ट्रांसपोर्ट (REST सर्वर बनाम स्थानीय CLI) और समूह रजिस्ट्री कुंजी का प्रारूप है।

| BlueBubbles                                                | बंडल किया गया iMessage                          | टिप्पणियाँ                                                                                                                                                                                                                                                                            |
| ---------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `channels.bluebubbles.enabled`                             | `channels.imessage.enabled`               | समान अर्थविज्ञान (ब्लॉक मौजूद होने के बाद डिफ़ॉल्ट `true`)।                                                                                                                                                                                                                           |
| `channels.bluebubbles.serverUrl`                           | _(हटाया गया)_                               | कोई REST सर्वर नहीं — Plugin, `imsg rpc` को stdio पर प्रारंभ करता है।                                                                                                                                                                                                                        |
| `channels.bluebubbles.password`                            | _(हटाया गया)_                               | किसी Webhook प्रमाणीकरण की आवश्यकता नहीं।                                                                                                                                                                                                                                                |
| _(अंतर्निहित)_                                               | `channels.imessage.cliPath`               | `imsg` का पथ (डिफ़ॉल्ट `imsg`); SSH के लिए रैपर स्क्रिप्ट का उपयोग करें।                                                                                                                                                                                                                   |
| _(अंतर्निहित)_                                               | `channels.imessage.dbPath`                | वैकल्पिक Messages.app `chat.db` ओवरराइड; छोड़ने पर स्वतः पता लगाया जाता है।                                                                                                                                                                                                            |
| _(अंतर्निहित)_                                               | `channels.imessage.remoteHost`            | `host` या `user@host` — केवल तब आवश्यक, जब `cliPath` एक SSH रैपर हो और आपको SCP अटैचमेंट फ़ेच चाहिए।                                                                                                                                                                        |
| `channels.bluebubbles.dmPolicy`                            | `channels.imessage.dmPolicy`              | समान मान (`pairing` / `allowlist` / `open` / `disabled`); डिफ़ॉल्ट `pairing`।                                                                                                                                                                                                  |
| `channels.bluebubbles.allowFrom`                           | `channels.imessage.allowFrom`             | समान हैंडल प्रारूप (`+15555550123`, `user@example.com`)। पेयरिंग-स्टोर स्वीकृतियाँ स्थानांतरित नहीं होतीं — नीचे देखें।                                                                                                                                                                   |
| `channels.bluebubbles.groupPolicy`                         | `channels.imessage.groupPolicy`           | समान मान (`allowlist` / `open` / `disabled`); डिफ़ॉल्ट `allowlist`।                                                                                                                                                                                                            |
| `channels.bluebubbles.groupAllowFrom`                      | `channels.imessage.groupAllowFrom`        | समान। सेट न होने पर iMessage, `allowFrom` पर वापस जाता है; स्पष्ट रूप से खाली `groupAllowFrom: []`, `groupPolicy: "allowlist"` के अंतर्गत सभी समूहों को अवरुद्ध करता है।                                                                                                                               |
| `channels.bluebubbles.groups`                              | `channels.imessage.groups`                | `"*"` वाइल्डकार्ड प्रविष्टि को ज्यों का त्यों कॉपी करें; प्रति-समूह प्रविष्टियों को संख्यात्मक iMessage `chat_id` के अनुसार फिर से कुंजीबद्ध करें — "समूह रजिस्ट्री की पेचीदगी" देखें। `requireMention`, `tools`, `toolsBySender`, `systemPrompt` आगे भी लागू रहते हैं।                                                                            |
| `channels.bluebubbles.sendReadReceipts`                    | `channels.imessage.sendReadReceipts`      | डिफ़ॉल्ट `true`। बंडल किए गए Plugin के साथ यह केवल तभी सक्रिय होता है, जब निजी API जाँच चालू हो।                                                                                                                                                                                        |
| `channels.bluebubbles.includeAttachments`                  | `channels.imessage.includeAttachments`    | समान संरचना, समान रूप से डिफ़ॉल्टतः बंद। यदि BlueBubbles पर अटैचमेंट आते थे, तो इसे स्पष्ट रूप से सेट करें — ऐसा करने तक आने वाली फ़ोटो/मीडिया चुपचाप छोड़ दी जाती हैं (कोई `Inbound message` लॉग पंक्ति नहीं)।                                                                                             |
| `channels.bluebubbles.attachmentRoots`                     | `channels.imessage.attachmentRoots`       | स्थानीय रूट; समान वाइल्डकार्ड नियम।                                                                                                                                                                                                                                                |
| _(लागू नहीं)_                                                    | `channels.imessage.remoteAttachmentRoots` | केवल तब उपयोग होता है, जब SCP फ़ेच के लिए `remoteHost` सेट हो।                                                                                                                                                                                                                              |
| `channels.bluebubbles.mediaMaxMb`                          | `channels.imessage.mediaMaxMb`            | iMessage पर डिफ़ॉल्ट 16 MB (BlueBubbles का डिफ़ॉल्ट 8 MB था)। निचली सीमा बनाए रखने के लिए इसे स्पष्ट रूप से सेट करें।                                                                                                                                                                                  |
| `channels.bluebubbles.textChunkLimit`                      | `channels.imessage.textChunkLimit`        | दोनों पर डिफ़ॉल्ट 4000।                                                                                                                                                                                                                                                            |
| `channels.bluebubbles.coalesceSameSenderDms`               | _(हटाया गया)_                               | इस कुंजी को माइग्रेट न करें। `imsg` 0.13.1 और उसके बाद के संस्करण, Apple URL-पूर्वावलोकन के विभाजित प्रेषणों को OpenClaw तक पहुँचने से पहले संयोजित कर देते हैं; `openclaw doctor --fix` एक पुरानी iMessage कुंजी हटाता है।                                                                                                    |
| `channels.bluebubbles.enrichGroupParticipantsFromContacts` | _(लागू नहीं)_                                   | `imsg`, `chat.db` से प्रेषक के प्रदर्शन नाम पहले ही उपलब्ध कराता है।                                                                                                                                                                                                                     |
| `channels.bluebubbles.actions.*`                           | `channels.imessage.actions.*`             | समान प्रति-कार्रवाई टॉगल (`reactions`, `edit`, `unsend`, `reply`, `sendWithEffect`, `renameGroup`, `setGroupIcon`, `addParticipant`, `removeParticipant`, `leaveGroup`, `sendAttachment`) और नया `polls`। सभी डिफ़ॉल्ट रूप से सक्षम हैं; निजी API कार्रवाइयों के लिए अभी भी ब्रिज आवश्यक है। |

बहु-अकाउंट कॉन्फ़िगरेशन (`channels.bluebubbles.accounts.*`), `channels.imessage.accounts.*` में एक-से-एक रूपांतरित होते हैं।

## समूह रजिस्ट्री की पेचीदगी

बंडल किया गया iMessage Plugin दो समूह गेट लगातार चलाता है। एजेंट तक पहुँचने के लिए समूह संदेश को दोनों से गुजरना आवश्यक है:

1. **प्रेषक / चैट-लक्ष्य अनुमति-सूची** (`channels.imessage.groupAllowFrom`) — प्रेषक हैंडल या चैट लक्ष्य (`chat_id:`, `chat_guid:`, `chat_identifier:` प्रविष्टियाँ) से मिलान करती है। `groupAllowFrom` सेट न होने पर यह गेट `allowFrom` पर वापस जाता है; स्पष्ट `groupAllowFrom: []` उस फ़ॉलबैक को अक्षम करता है और `groupPolicy: "allowlist"` के अंतर्गत प्रत्येक समूह संदेश छोड़ देता है।
2. **समूह रजिस्ट्री** (`channels.imessage.groups`) — संख्यात्मक iMessage `chat_id` द्वारा कुंजीबद्ध:
   - कोई `groups` ब्लॉक नहीं (या खाली ब्लॉक): जब तक गेट 1 में एक गैर-रिक्त प्रभावी प्रेषक अनुमति-सूची है, समूह इस गेट से गुजरते हैं; प्रेषक फ़िल्टरिंग पहुँच नियंत्रित करती है और प्रारंभ के समय सभी को छोड़ने वाली कोई चेतावनी सक्रिय नहीं होती।
   - `groups` में प्रविष्टियाँ हैं, लेकिन `"*"` नहीं है: केवल सूचीबद्ध `chat_id` कुंजियाँ गुजरती हैं। किसी भी समूह को सूचीबद्ध करने से रजिस्ट्री, `groupPolicy: "open"` के अंतर्गत भी अनुमति-सूची बन जाती है।
   - `groups: { "*": { ... } }`: प्रत्येक समूह इस गेट से गुजरता है।

माइग्रेशन का जाल: BlueBubbles ने `groups` प्रविष्टियों को चैट GUID / चैट पहचानकर्ता द्वारा कुंजीबद्ध किया था, जबकि iMessage रजिस्ट्री संख्यात्मक `chat_id` द्वारा कुंजीबद्ध होती है। प्रति-समूह प्रविष्टियाँ ज्यों की त्यों कॉपी करने पर एक गैर-रिक्त रजिस्ट्री बनती है जिसकी कुंजियाँ कभी मेल नहीं खातीं, इसलिए प्रत्येक समूह संदेश गेट 2 पर छोड़ दिया जाता है। `"*"` वाइल्डकार्ड को ज्यों का त्यों कॉपी करें; विशिष्ट समूह प्रविष्टियों को `imsg chats` से प्राप्त `chat_id` मानों के साथ फिर से कुंजीबद्ध करें।

दोनों ड्रॉप पथ डिफ़ॉल्ट लॉग स्तर पर `warn` पंक्तियों के माध्यम से दिखाई देते हैं:

- प्रारंभ के समय प्रति अकाउंट एक बार, जब `groupPolicy: "allowlist"` सेट हो और प्रभावी समूह प्रेषक अनुमति-सूची खाली हो: `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...`। प्रेषकों को अनुमति देने के लिए `groupAllowFrom` (या `allowFrom`) सेट करें; केवल `groups` जोड़ने से प्रेषक गेट संतुष्ट नहीं होता।
- रनटाइम पर प्रति `chat_id` एक बार, जब रजिस्ट्री किसी समूह को छोड़ती है: `imessage: dropping group message from chat_id=<id> ... not in channels.imessage.groups allowlist`, जिसमें जोड़ने योग्य सटीक कुंजी का नाम होता है।

DM दोनों स्थितियों में काम करते रहते हैं — वे अलग कोड पथ लेते हैं, इसलिए DM की सफलता समूह रूटिंग सिद्ध नहीं करती।

`groupPolicy: "allowlist"` के साथ न्यूनतम प्रेषक-स्कोप वाला कॉन्फ़िगरेशन:

```json5
{
  channels: {
    imessage: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123", "chat_guid:any;-;..."],
    },
  },
}
```

यह कॉन्फ़िगर किए गए प्रेषकों को किसी भी समूह में अनुमति देता है। अनुमत चैट का दायरा सीमित करने या `requireMention` जैसे प्रति-चैट विकल्प सेट करने के लिए `groups` प्रविष्टियाँ जोड़ें; BlueBubbles की `"*"` प्रविष्टि ज्यों की त्यों कॉपी करें, लेकिन विशिष्ट प्रविष्टियों को संख्यात्मक iMessage `chat_id` मानों के साथ फिर से कुंजीबद्ध करें।

## चरण-दर-चरण

1. कॉन्फ़िगरेशन का रूपांतरण करें। संपादन के दौरान नया ब्लॉक अक्षम रखें; वर्तमान OpenClaw पुराने `channels.bluebubbles` ब्लॉक को अनदेखा करता है और वह संदर्भ के रूप में साथ रह सकता है:

   ```json5
   {
     channels: {
       imessage: {
         enabled: false, // कटओवर के लिए तैयार होने पर true करें
         cliPath: "/opt/homebrew/bin/imsg",
         dmPolicy: "pairing",
         allowFrom: ["+15555550123"], // bluebubbles.allowFrom से कॉपी करें
         groupPolicy: "allowlist",
         groupAllowFrom: [], // bluebubbles.groupAllowFrom से कॉपी करें
         groups: { "*": { requireMention: true } }, // वाइल्डकार्ड ज्यों का त्यों कॉपी होता है; प्रति-चैट प्रविष्टियों को chat_id द्वारा फिर से कुंजीबद्ध करें
         // कार्रवाइयाँ डिफ़ॉल्ट रूप से सक्षम हैं; अक्षम करने के लिए अलग-अलग टॉगल false पर सेट करें
       },
     },
   }
   ```

2. **कटओवर और जाँच करें।** `channels.imessage.enabled: true` सेट करें, Gateway पुनः प्रारंभ करें और पुष्टि करें कि चैनल स्वस्थ स्थिति दिखाता है:

   ```bash
   openclaw gateway restart
   openclaw channels status --probe --channel imessage   # "works" अपेक्षित है; --json में privateApi.available: true दिखता है
   ```

   प्रोब के लिए पहुँच योग्य Gateway आवश्यक है और यह केवल कॉन्फ़िगर किए गए, सक्षम खातों को प्रोब करता है। स्वयं Mac को सत्यापित करने के लिए [शुरू करने से पहले](#before-you-start) में दिए गए प्रत्यक्ष `imsg` कमांड का उपयोग करें।

3. **DM सत्यापित करें।** एजेंट को एक प्रत्यक्ष संदेश भेजें; पुष्टि करें कि उत्तर पहुँचता है।

4. **समूहों को अलग से सत्यापित करें।** DM और समूह अलग-अलग कोड पथों का उपयोग करते हैं — DM की सफलता यह सिद्ध नहीं करती कि समूह सही ढंग से रूट हो रहे हैं। किसी अनुमत समूह चैट में संदेश भेजें और पुष्टि करें कि उत्तर पहुँचता है। यदि समूह मौन हो जाता है (एजेंट से कोई उत्तर नहीं, कोई त्रुटि नहीं), तो ऊपर दिए गए "समूह रजिस्ट्री की पेचीदगी" के दो `warn` लॉग के लिए Gateway लॉग जाँचें। स्टार्टअप चेतावनी का अर्थ है कि प्रभावी प्रेषक अनुमति-सूची खाली है; प्रति-`chat_id` चेतावनी का अर्थ है कि भरी हुई `groups` रजिस्ट्री में वह चैट नहीं है।

5. **कार्रवाई सतह सत्यापित करें।** युग्मित DM से एजेंट को प्रतिक्रिया देने, संपादित करने, भेजना रद्द करने, उत्तर देने, फ़ोटो भेजने और (किसी समूह में) समूह का नाम बदलने या किसी प्रतिभागी को जोड़ने/हटाने के लिए कहें। प्रत्येक कार्रवाई Messages.app में मूल रूप से दिखाई देनी चाहिए। यदि कोई कार्रवाई `iMessage <action> requires the imsg private API bridge` उत्पन्न करती है, तो `imsg launch` फिर से चलाएँ और `openclaw channels status --probe` से रीफ़्रेश करें।

6. iMessage के DM, समूह और कार्रवाइयाँ सत्यापित हो जाने के बाद **BlueBubbles सर्वर और `channels.bluebubbles` ब्लॉक हटाएँ**। OpenClaw `channels.bluebubbles` को नहीं पढ़ता है।

## कार्रवाई समानता एक नज़र में

| कार्रवाई                                              | पुराना BlueBubbles | बंडल किया गया iMessage                                                              |
| --------------------------------------------------- | ------------------ | ----------------------------------------------------------------------------- |
| टेक्स्ट भेजना / SMS फ़ॉलबैक                            | ✅                 | ✅                                                                            |
| मीडिया भेजना (फ़ोटो, वीडियो, फ़ाइल, आवाज़)              | ✅                 | ✅                                                                            |
| थ्रेड वाला उत्तर (`reply_to_guid`)                    | ✅                 | ✅ ([#51892](https://github.com/openclaw/openclaw/issues/51892) का समाधान करता है)       |
| Tapback (`react`)                                   | ✅                 | ✅                                                                            |
| संपादित करना / भेजना रद्द करना (macOS 13+ प्राप्तकर्ता)                | ✅                 | ✅                                                                            |
| स्क्रीन प्रभाव के साथ भेजना                             | ✅                 | ✅ ([#9394](https://github.com/openclaw/openclaw/issues/9394) के एक भाग का समाधान करता है) |
| रिच टेक्स्ट बोल्ड / इटैलिक / रेखांकित / स्ट्राइकथ्रू | ✅                 | ✅ (attributedBody के माध्यम से टाइप्ड-रन फ़ॉर्मैटिंग)                                  |
| मूल Messages पोल (बनाना और वोट करना)             | ❌                 | ✅ (`actions.polls`; मूल रेंडरिंग के लिए प्राप्तकर्ताओं को iOS/macOS 26+ चाहिए)      |
| समूह का नाम बदलना / समूह आइकन सेट करना                       | ✅                 | ✅                                                                            |
| प्रतिभागी जोड़ना / हटाना, समूह छोड़ना               | ✅                 | ✅                                                                            |
| पढ़ने की रसीदें और टाइपिंग संकेतक                  | ✅                 | ✅ (निजी API प्रोब पर निर्भर)                                               |
| Apple URL-पूर्वावलोकन स्प्लिट-सेंड संयोजन             | ✅                 | ✅ (अपस्ट्रीम में `imsg` 0.13.1 और नए संस्करणों द्वारा प्रबंधित; कोई OpenClaw सेटिंग नहीं)         |
| पुनरारंभ के बाद इनबाउंड पुनर्प्राप्ति                    | ✅                 | ✅ (स्वचालित: `since_rowid` रीप्ले + GUID डीडुप्लिकेशन; लोकल पर अधिक विस्तृत विंडो)     |

Gateway बंद रहने के दौरान छूटे संदेशों को iMessage पुनर्प्राप्त करता है: स्टार्टअप पर यह `imsg watch.subscribe` `since_rowid` के माध्यम से अंतिम डिस्पैच किए गए rowid से रीप्ले करता है, GUID के अनुसार डीडुप्लिकेट करता है और पुरानी-बैकलॉग आयु सीमा Push-flush "बैकलॉग बम" को रोकती है। यह `imsg` RPC कनेक्शन पर चलता है, इसलिए दूरस्थ SSH `cliPath` सेटअप के लिए भी काम करता है; लोकल सेटअप को अधिक विस्तृत पुनर्प्राप्ति विंडो मिलती है क्योंकि वे `chat.db` पढ़ सकते हैं। [ब्रिज या Gateway पुनरारंभ के बाद इनबाउंड पुनर्प्राप्ति](/hi/channels/imessage#inbound-recovery-after-a-bridge-or-gateway-restart) देखें।

## युग्मन, सत्र और ACP बाइंडिंग

- **अनुमति-सूचियाँ हैंडल के अनुसार स्थानांतरित होती हैं।** `channels.imessage.allowFrom` उन्हीं `+15555550123` / `user@example.com` स्ट्रिंग को पहचानता है जिनका उपयोग BlueBubbles करता था — उन्हें ज्यों का त्यों कॉपी करें।
- **युग्मन-स्टोर की स्वीकृतियाँ स्थानांतरित नहीं होतीं।** युग्मन स्टोर प्रत्येक चैनल के लिए अलग होता है और पुराने BlueBubbles स्टोर का कोई माइग्रेशन नहीं होता। जिन प्रेषकों को केवल युग्मन के माध्यम से स्वीकृति मिली थी, उन्हें iMessage के अंतर्गत एक बार फिर युग्मित होना होगा, या आप उनके हैंडल `allowFrom` में जोड़ें।
- **सत्र** प्रत्येक एजेंट + चैट के दायरे में बने रहते हैं। डिफ़ॉल्ट `session.dmScope=main` के अंतर्गत DM एजेंट के मुख्य सत्र में समाहित हो जाते हैं; समूह सत्र प्रत्येक `chat_id` (`agent:<agentId>:imessage:group:<chat_id>`) के अनुसार अलग बने रहते हैं। BlueBubbles सत्र कुंजियों के अंतर्गत पुराना वार्तालाप इतिहास iMessage सत्रों में स्थानांतरित नहीं होता।
- `match.channel: "bluebubbles"` को संदर्भित करने वाली **ACP बाइंडिंग** को `"imessage"` में बदलना होगा। `match.peer.id` आकार (`chat_id:`, `chat_guid:`, `chat_identifier:`, केवल हैंडल) समान हैं।

## कोई रोलबैक चैनल नहीं

वापस स्विच करने के लिए कोई समर्थित BlueBubbles रनटाइम नहीं है। यदि iMessage सत्यापन विफल होता है, तो `channels.imessage.enabled: false` सेट करें, Gateway पुनरारंभ करें, `imsg` अवरोधक ठीक करें और कटओवर का पुनः प्रयास करें।

उत्तर कैश SQLite Plugin स्थिति में रहता है। मौजूद होने पर `openclaw doctor --fix` पुराने `imessage/reply-cache.jsonl` साइडकार को आयात और संग्रहित करता है।

## संबंधित

- [BlueBubbles हटाना और imsg iMessage पथ](/hi/announcements/bluebubbles-imessage) — संक्षिप्त घोषणा और ऑपरेटर सारांश।
- [iMessage](/hi/channels/imessage) — संपूर्ण iMessage चैनल संदर्भ, जिसमें `imsg launch` सेटअप और क्षमता पहचान शामिल हैं।
- `/channels/bluebubbles` — पुराना URL जो इस माइग्रेशन मार्गदर्शिका पर रीडायरेक्ट करता है।
- [युग्मन](/hi/channels/pairing) — DM प्रमाणीकरण और युग्मन प्रवाह।
- [चैनल रूटिंग](/hi/channels/channel-routing) — Gateway आउटबाउंड उत्तरों के लिए चैनल कैसे चुनता है।
