---
read_when:
    - हमेशा सक्रिय समूह या चैनल रूम कॉन्फ़िगर करना
    - आप चाहते हैं कि एजेंट कमरे की बातचीत पर नज़र रखे, लेकिन अंतिम टेक्स्ट अपने-आप पोस्ट न करे
    - बिना किसी दृश्य रूम संदेश के टाइपिंग और टोकन उपयोग की डीबगिंग
sidebarTitle: Ambient room events
summary: समर्थित समूह कक्षों को शांत संदर्भ प्रदान करने दें, जब तक कि एजेंट संदेश टूल से न भेजे
title: परिवेशी कक्ष ईवेंट
x-i18n:
    generated_at: "2026-07-27T19:19:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 15c083c139058c9bd2c651794965bd8252d74691e536db2ad2a2ae0b4ac886e8
    source_path: channels/ambient-room-events.md
    workflow: 16
---

परिवेशी रूम इवेंट OpenClaw को बिना उल्लेख वाली समूह या चैनल बातचीत को शांत संदर्भ के रूप में संसाधित करने देते हैं। एजेंट मेमोरी और सेशन स्थिति अपडेट कर सकता है, लेकिन रूम तब तक शांत रहता है जब तक एजेंट स्पष्ट रूप से `message` टूल को कॉल नहीं करता।

हमेशा सक्रिय रहने वाली समूह चैट के लिए, `messages.groupChat.unmentionedInbound: "room_event"` को `messages.groupChat.visibleReplies: "message_tool"` के साथ संयोजित करें। एजेंट सुनता है, तय करता है कि उत्तर कब उपयोगी है, और उसे `NO_REPLY` का उत्तर देने वाले पुराने प्रॉम्प्ट पैटर्न की कभी आवश्यकता नहीं होती।

वर्तमान में समर्थित: Discord गिल्ड चैनल, Slack चैनल और निजी चैनल, Slack बहु-व्यक्ति DM, और Telegram समूह या सुपरग्रुप। अन्य समूह चैनल अपना मौजूदा समूह व्यवहार बनाए रखते हैं, जब तक उनके चैनल पेज पर यह न कहा गया हो कि वे परिवेशी रूम इवेंट का समर्थन करते हैं।

## अनुशंसित सेटअप

वैश्विक समूह-चैट व्यवहार सेट करें:

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
}
```

फिर उस रूम के लिए उल्लेख गेटिंग अक्षम करके उसे हमेशा सक्रिय बनाएँ। रूम को फिर भी अपने सामान्य `groupPolicy`, रूम अनुमति-सूची और प्रेषक अनुमति-सूची से गुजरना होगा।

कॉन्फ़िगरेशन सहेजने के बाद, Gateway `messages` सेटिंग को हॉट-अप्लाई करता है। केवल तभी पुनरारंभ करें जब फ़ाइल निगरानी या कॉन्फ़िगरेशन रीलोड अक्षम हो (`gateway.reload.mode: "off"`)।

## क्या बदलता है

`messages.groupChat.unmentionedInbound: "room_event"` के साथ:

- बिना उल्लेख वाले अनुमत समूह या चैनल संदेश शांत रूम इवेंट बन जाते हैं
- उल्लेख वाले संदेश उपयोगकर्ता अनुरोध बने रहते हैं
- टेक्स्ट नियंत्रण कमांड और नेटिव कमांड उपयोगकर्ता अनुरोध बने रहते हैं
- निरस्त करने या रोकने के अनुरोध उपयोगकर्ता अनुरोध बने रहते हैं
- प्रत्यक्ष संदेश उपयोगकर्ता अनुरोध बने रहते हैं

रूम इवेंट सख्त दृश्यमान डिलीवरी का उपयोग करते हैं। अंतिम सहायक टेक्स्ट निजी रहता है। रूम में पोस्ट करने के लिए एजेंट को `message(action=send)` कॉल करना होगा।

रूम इवेंट के लिए टाइपिंग और जीवनचक्र स्थिति प्रतिक्रियाएँ दबाई हुई रहती हैं। एकमात्र स्पष्ट प्राप्ति अपवाद `messages.ackReactionScope: "all"` है, जो कॉन्फ़िगर की गई स्वीकृति प्रतिक्रिया भेजता है; जब रूम को पूरी तरह शांत रखना हो, तो किसी अधिक सीमित दायरे या `"off"` का उपयोग करें।

## Discord उदाहरण

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "<DISCORD_SERVER_ID>": {
          requireMention: false,
          users: ["<YOUR_DISCORD_USER_ID>"],
        },
      },
    },
  },
}
```

जब केवल एक चैनल परिवेशी होना चाहिए, तो प्रति-चैनल Discord कॉन्फ़िगरेशन का उपयोग करें। `groupPolicy: "allowlist"` के अंतर्गत चैनल को सूचीबद्ध करना ही उसे अनुमति देता है (`enabled: false` किसी प्रविष्टि को अक्षम करता है):

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "<DISCORD_SERVER_ID>": {
          channels: {
            "<DISCORD_CHANNEL_ID_OR_NAME>": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

## Slack उदाहरण

Slack चैनल अनुमति-सूचियाँ मुख्यतः ID पर आधारित होती हैं। `#channel-name` के बजाय `C12345678` जैसी चैनल ID का उपयोग करें। `channels.slack.channels` के अंतर्गत चैनल को सूचीबद्ध करना ही उसे अनुमति देता है (`enabled: false` किसी प्रविष्टि को अक्षम करता है):

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    slack: {
      groupPolicy: "allowlist",
      channels: {
        "<SLACK_CHANNEL_ID>": {
          requireMention: false,
        },
      },
    },
  },
}
```

## Telegram उदाहरण

Telegram समूहों के लिए, बॉट को सामान्य समूह संदेश देखने में सक्षम होना चाहिए। यदि `requireMention: false`, तो BotFather गोपनीयता मोड अक्षम करें या किसी अन्य Telegram सेटअप का उपयोग करें जो समूह का पूरा ट्रैफ़िक बॉट तक पहुँचाता हो।

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    telegram: {
      groups: {
        "<TELEGRAM_GROUP_CHAT_ID>": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

Telegram समूह ID आमतौर पर `-1001234567890` जैसी ऋणात्मक संख्याएँ होती हैं। `openclaw logs --follow` से `chat.id` पढ़ें, किसी समूह संदेश को ID सहायक बॉट पर अग्रेषित करें, या Bot API `getUpdates` का निरीक्षण करें।

## एजेंट-विशिष्ट नीति

जब कई एजेंट एक ही रूम साझा करते हों, लेकिन केवल एक को बिना उल्लेख वाली बातचीत को परिवेशी संदर्भ मानना चाहिए, तब एजेंट ओवरराइड का उपयोग करें:

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "message_tool",
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          unmentionedInbound: "room_event",
          mentionPatterns: ["@openclaw", "openclaw"],
        },
      },
    ],
  },
}
```

एजेंट-विशिष्ट `agents.entries.*.groupChat.unmentionedInbound` मान उस एजेंट के लिए `messages.groupChat.unmentionedInbound` को ओवरराइड करता है।

## दृश्यमान उत्तर मोड

सामान्य समूह/चैनल उपयोगकर्ता अनुरोधों के लिए `messages.groupChat.visibleReplies` का डिफ़ॉल्ट `"automatic"` होता है। जब अंतिम सहायक टेक्स्ट को स्पष्ट संदेश-टूल कॉल के बिना दृश्यमान रूप से पोस्ट होना चाहिए, तो इस डिफ़ॉल्ट को बनाए रखें।

परिवेशी हमेशा सक्रिय रूम के लिए, `messages.groupChat.visibleReplies: "message_tool"` की अब भी अनुशंसा की जाती है, विशेष रूप से GPT-5.6 Sol जैसे नवीनतम पीढ़ी के, टूल के विश्वसनीय उपयोग वाले मॉडल के साथ। यह एजेंट को संदेश टूल कॉल करके तय करने देता है कि कब बोलना है। यदि मॉडल टूल कॉल किए बिना अंतिम टेक्स्ट लौटाता है, तो OpenClaw उस अंतिम टेक्स्ट को निजी रखता है और दबाई गई डिलीवरी का मेटाडेटा लॉग करता है।

अन्य समूह अनुरोधों द्वारा स्वचालित उत्तरों का उपयोग किए जाने पर भी रूम इवेंट सख्त बने रहते हैं। बिना उल्लेख वाले परिवेशी रूम इवेंट के दृश्यमान आउटपुट के लिए हमेशा `message(action=send)` आवश्यक है।

## इतिहास

`messages.groupChat.historyLimit` वैश्विक समूह इतिहास का डिफ़ॉल्ट सेट करता है (सेट न होने पर 50; धनात्मक पूर्णांक होना आवश्यक है)। चैनल इसे `channels.<channel>.historyLimit` से ओवरराइड कर सकते हैं, और कुछ चैनल प्रति-अकाउंट इतिहास सीमाओं का भी समर्थन करते हैं। उस चैनल के लिए समूह इतिहास संदर्भ अक्षम करने हेतु चैनल-स्तरीय `historyLimit: 0` सेट करें।

समर्थित रूम-इवेंट चैनल हाल के परिवेशी रूम संदेशों को संदर्भ के रूप में बनाए रखते हैं। Telegram `historyLimit` द्वारा सीमित, हमेशा सक्रिय रहने वाली प्रति-समूह क्रमिक विंडो रखता है; उपयोगकर्ता-अनुरोध टर्न बॉट के अंतिम रिकॉर्ड किए गए उत्तर के बाद की प्रविष्टियाँ चुनते हैं, जबकि रूम-इवेंट टर्न को पूरी हालिया विंडो मिलती है ताकि मॉडल अपनी हालिया पोस्ट देख सके। सेवानिवृत्त Telegram `includeGroupHistoryContext` मोड कुंजी को `openclaw doctor --fix` द्वारा हटा दिया जाता है।

## समस्या निवारण

यदि रूम टाइपिंग या टोकन उपयोग दिखाता है, लेकिन कोई दृश्यमान संदेश नहीं दिखाता:

1. पुष्टि करें कि चैनल अनुमति-सूची और प्रेषक अनुमति-सूची द्वारा रूम अनुमत है।
2. पुष्टि करें कि `requireMention: false` आपके अपेक्षित रूम स्तर पर सेट है।
3. जाँचें कि `messages.groupChat.unmentionedInbound` या एजेंट ओवरराइड `"room_event"` है या नहीं।
4. दबाए गए अंतिम पेलोड मेटाडेटा या `didSendViaMessagingTool: false` के लिए लॉग का निरीक्षण करें।
5. सामान्य समूह अनुरोधों के लिए, यदि आप अंतिम उत्तरों को स्वचालित रूप से पोस्ट करना चाहते हैं, तो `messages.groupChat.visibleReplies: "automatic"` बनाए रखें या पुनर्स्थापित करें। `message_tool` का उपयोग करने वाले परिवेशी रूम के लिए, ऐसे मॉडल/रनटाइम का उपयोग करें जो विश्वसनीय रूप से टूल कॉल करता हो।

यदि Telegram परिवेशी रूम बिल्कुल ट्रिगर नहीं होते, तो BotFather गोपनीयता मोड जाँचें और सत्यापित करें कि Gateway को सामान्य समूह संदेश प्राप्त हो रहे हैं।

यदि Slack परिवेशी रूम ट्रिगर नहीं होते, तो सत्यापित करें कि चैनल कुंजी Slack चैनल ID है और ऐप के पास उस रूम प्रकार के लिए इतिहास स्कोप है: `channels:history` (सार्वजनिक), `groups:history` (निजी), या `mpim:history` (बहु-व्यक्ति DM)।

## संबंधित

- [समूह](/hi/channels/groups)
- [Discord](/hi/channels/discord)
- [Slack](/hi/channels/slack)
- [Telegram](/hi/channels/telegram)
- [चैनल समस्या निवारण](/hi/channels/troubleshooting)
- [चैनल कॉन्फ़िगरेशन संदर्भ](/hi/gateway/config-channels)
