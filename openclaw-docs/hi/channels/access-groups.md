---
read_when:
    - एकाधिक संदेश चैनलों में समान अनुमति-सूची कॉन्फ़िगर करना
    - DM और समूह प्रेषक पहुँच नियम साझा करना
    - संदेश-चैनल अभिगम नियंत्रण की समीक्षा करना
summary: 'संदेश चैनलों के लिए पुन: उपयोग योग्य प्रेषक अनुमति-सूचियाँ'
title: एक्सेस समूह
x-i18n:
    generated_at: "2026-07-27T19:06:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 099abc95e90d9a7b7006d19062c46b4ffdb2aecb1e8e714454a3182131a786d0
    source_path: channels/access-groups.md
    workflow: 16
---

ऐक्सेस समूह नामित प्रेषक सूचियाँ हैं, जिन्हें आप `accessGroups` के अंतर्गत एक बार परिभाषित करते हैं और `accessGroup:<name>` के साथ चैनल की अनुमत सूचियों से संदर्भित करते हैं।

इनका उपयोग तब करें जब उन्हीं लोगों को कई संदेश चैनलों पर अनुमति दी जानी चाहिए, या जब एक विश्वसनीय समूह DMs और समूह प्रेषक प्राधिकरण—दोनों पर लागू होना चाहिए।

कोई समूह अपने-आप कुछ भी प्रदान नहीं करता। वह केवल वहाँ प्रभावी होता है जहाँ कोई अनुमत सूची फ़ील्ड उसका संदर्भ देती है।

## स्थिर संदेश प्रेषक समूह

स्थिर प्रेषक समूह `type: "message.senders"` का उपयोग करते हैं। `members` को संदेश-चैनल id के आधार पर कुंजीबद्ध किया जाता है, और हर चैनल द्वारा साझा की जाने वाली प्रविष्टियों के लिए `"*"` भी होता है:

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
        whatsapp: ["+15551234567"],
      },
    },
  },
}
```

| कुंजी                        | अर्थ                                                                     |
| -------------------------- | --------------------------------------------------------------------------- |
| `"*"`                      | समूह का संदर्भ देने वाले प्रत्येक संदेश चैनल के लिए जाँची जाने वाली साझा प्रविष्टियाँ। |
| `discord`, `telegram`, ... | केवल उस चैनल की अनुमत सूची से मिलान के लिए जाँची जाने वाली प्रविष्टियाँ।                 |

प्रविष्टियों का मिलान गंतव्य चैनल के सामान्य `allowFrom` नियमों के अनुसार किया जाता है। OpenClaw चैनलों के बीच प्रेषक ids का रूपांतरण नहीं करता: यदि Alice की एक Telegram id और एक Discord id है, तो दोनों ids को संबंधित चैनल कुंजियों के अंतर्गत सूचीबद्ध करें।

## अनुमत सूचियों से समूहों का संदर्भ देना

जहाँ भी संदेश चैनल का पथ प्रेषक अनुमत सूचियों का समर्थन करता है, वहाँ `accessGroup:<name>` के साथ किसी समूह का संदर्भ दें।

DM अनुमत सूची का उदाहरण:

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
      },
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
    telegram: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
  },
}
```

समूह प्रेषक अनुमत सूची का उदाहरण:

```json5
{
  accessGroups: {
    oncall: {
      type: "message.senders",
      members: {
        whatsapp: ["+15551234567"],
        googlechat: ["users/1234567890"],
      },
    },
  },
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["accessGroup:oncall"],
    },
    googlechat: {
      groups: {
        "spaces/AAA": {
          users: ["accessGroup:oncall"],
        },
      },
    },
  },
}
```

आप समूहों और प्रत्यक्ष प्रविष्टियों को मिला सकते हैं:

```json5
{
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators", "discord:123456789012345678"],
    },
  },
}
```

## समर्थित संदेश-चैनल पथ

ऐक्सेस समूह साझा संदेश-चैनल प्राधिकरण पथों में काम करते हैं:

- DM प्रेषक अनुमत सूचियाँ, जैसे `channels.<channel>.allowFrom`
- समूह प्रेषक अनुमत सूचियाँ, जैसे `channels.<channel>.groupAllowFrom`
- चैनल-विशिष्ट, प्रति-कक्ष प्रेषक अनुमत सूचियाँ, जो उन्हीं प्रेषक मिलान नियमों का उपयोग करती हैं (उदाहरण के लिए Google Chat `groups.<space>.users`)
- कमांड प्राधिकरण पथ, जो संदेश-चैनल प्रेषक अनुमत सूचियों का पुनः उपयोग करते हैं

चैनल समर्थन इस बात पर निर्भर करता है कि वह चैनल साझा OpenClaw प्रेषक-प्राधिकरण सहायकों से जुड़ा है या नहीं। मौजूदा बंडल किए गए समर्थन में ClickClack, Discord, Feishu, Google Chat, iMessage, IRC, LINE, Mattermost, Microsoft Teams, Nextcloud Talk, Nostr, QQ Bot, Signal, Slack, SMS, Telegram, WhatsApp, Zalo और Zalo Personal शामिल हैं। स्थिर `message.senders` समूह चैनल-निरपेक्ष होते हैं, इसलिए नए संदेश चैनल कस्टम अनुमत सूची विस्तार के बजाय साझा plugin SDK इनग्रेस सहायकों का उपयोग करके इन्हें प्राप्त करते हैं।

## Discord चैनल दर्शक-समूह

Discord एक गतिशील ऐक्सेस समूह प्रकार का भी समर्थन करता है:

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
      membership: "canViewChannel",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers"],
    },
  },
}
```

`discord.channelAudience` का अर्थ है, "उन Discord DM प्रेषकों को अनुमति दें जो वर्तमान में इस गिल्ड चैनल को देख सकते हैं।" OpenClaw प्राधिकरण के समय Discord के माध्यम से प्रेषक की पहचान करता है और Discord के `ViewChannel` अनुमति नियम लागू करता है। `membership` वैकल्पिक है और इसका डिफ़ॉल्ट `canViewChannel` है।

इसका उपयोग तब करें जब कोई Discord चैनल पहले से ही किसी टीम के लिए सत्य का स्रोत हो, जैसे `#maintainers` या `#on-call`।

आवश्यकताएँ और विफलता व्यवहार:

- बॉट को गिल्ड और चैनल तक पहुँच चाहिए।
- बॉट को Discord Developer Portal **Server Members Intent** चाहिए।
- जब Discord `Missing Access` लौटाता है, प्रेषक को गिल्ड सदस्य के रूप में पहचाना नहीं जा सकता, या चैनल किसी अन्य गिल्ड का होता है, तो ऐक्सेस समूह बंद रहकर विफल होता है।

Discord से संबंधित और उदाहरण: [Discord ऐक्सेस नियंत्रण](/hi/channels/discord#access-control-and-routing)

## Plugin निदान

Plugin लेखक संरचित ऐक्सेस-समूह स्थिति को वापस समतल अनुमत सूची में विस्तृत किए बिना उसका निरीक्षण कर सकते हैं:

```typescript
import { resolveAccessGroupAllowFromState } from "openclaw/plugin-sdk/access-groups";

const state = await resolveAccessGroupAllowFromState({
  accessGroups: cfg.accessGroups,
  allowFrom: channelConfig.allowFrom,
  channel: "my-channel",
  accountId: "default",
  senderId,
  isSenderAllowed,
});
```

परिणाम संदर्भित, मिलान किए गए, अनुपलब्ध, असमर्थित और विफल समूहों की सूचना देता है। इसका उपयोग निदान या अनुरूपता परीक्षणों के लिए करें। `expandAllowFromWithAccessGroups(...)` का उपयोग केवल उन संगतता पथों के लिए करें जो अभी भी एक समतल `allowFrom` सरणी की अपेक्षा करते हैं।

## सुरक्षा संबंधी टिप्पणियाँ

- ऐक्सेस समूह अनुमत सूची के उपनाम हैं, भूमिकाएँ नहीं। वे अपने-आप स्वामी नहीं बनाते, पेयरिंग अनुरोधों को स्वीकृत नहीं करते, या टूल अनुमतियाँ प्रदान नहीं करते।
- `dmPolicy: "open"` के लिए प्रभावी DM अनुमत सूची में अब भी `"*"` आवश्यक है। किसी ऐक्सेस समूह का संदर्भ देना सार्वजनिक पहुँच के समान नहीं है।
- अनुपलब्ध समूह नाम बंद रहकर विफल होते हैं। यदि `allowFrom` में `accessGroup:operators` है और `accessGroups.operators` अनुपस्थित है, तो वह प्रविष्टि किसी को भी अधिकृत नहीं करती।
- चैनल ids को स्थिर रखें। जब चैनल दोनों का समर्थन करता हो, तो प्रदर्शन नामों की तुलना में संख्यात्मक/उपयोगकर्ता ids को प्राथमिकता दें।

## समस्या निवारण

यदि किसी प्रेषक का मिलान होना चाहिए, लेकिन उसे अवरुद्ध किया गया है:

1. पुष्टि करें कि अनुमत सूची फ़ील्ड में सटीक `accessGroup:<name>` संदर्भ मौजूद है।
2. पुष्टि करें कि `accessGroups.<name>.type` सही है।
3. पुष्टि करें कि प्रेषक id संबंधित चैनल कुंजी के अंतर्गत या `"*"` के अंतर्गत सूचीबद्ध है।
4. पुष्टि करें कि प्रविष्टि उस चैनल के सामान्य अनुमत सूची सिंटैक्स का उपयोग करती है।
5. Discord चैनल दर्शक-समूहों के लिए पुष्टि करें कि बॉट गिल्ड चैनल देख सकता है और Server Members Intent सक्षम है।

ऐक्सेस-नियंत्रण कॉन्फ़िगरेशन संपादित करने के बाद `openclaw doctor` चलाएँ। यह रनटाइम से पहले कई अमान्य अनुमत सूची और नीति संयोजनों का पता लगा लेता है।
