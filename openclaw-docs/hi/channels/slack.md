---
read_when:
    - Slack सेट अप करना या Slack सॉकेट, HTTP अथवा रिले मोड को डीबग करना
summary: Slack सेटअप और रनटाइम व्यवहार (Socket Mode, HTTP Request URLs और रिले मोड)
title: Slack
x-i18n:
    generated_at: "2026-07-27T17:27:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0f974ddf8e6965b09cede6a16f171434915a994fa3c1fc744d2350399941bee
    source_path: channels/slack.md
    workflow: 16
---

Slack समर्थन, Slack ऐप इंटीग्रेशन के माध्यम से DM और चैनलों को कवर करता है। डिफ़ॉल्ट ट्रांसपोर्ट Socket Mode है; HTTP Request URLs भी समर्थित हैं। रिले मोड उन प्रबंधित डिप्लॉयमेंट के लिए है जहाँ कोई विश्वसनीय राउटर Slack इनग्रेस का स्वामी होता है।

<CardGroup cols={3}>
  <Card title="पेयरिंग" icon="link" href="/hi/channels/pairing">
    Slack DM डिफ़ॉल्ट रूप से पेयरिंग मोड का उपयोग करते हैं।
  </Card>
  <Card title="स्लैश कमांड" icon="terminal" href="/hi/tools/slash-commands">
    नेटिव कमांड व्यवहार और कमांड कैटलॉग।
  </Card>
  <Card title="चैनल समस्या निवारण" icon="wrench" href="/hi/channels/troubleshooting">
    क्रॉस-चैनल निदान और सुधार कार्यपुस्तिकाएँ।
  </Card>
</CardGroup>

## ट्रांसपोर्ट चुनना

Socket Mode और HTTP Request URLs, मैसेजिंग, स्लैश कमांड, App Home और इंटरैक्टिविटी के लिए समान सुविधाएँ प्रदान करते हैं। सुविधाओं के आधार पर नहीं, बल्कि डिप्लॉयमेंट के स्वरूप के आधार पर चुनें।

| विचारणीय पहलू                      | Socket Mode (डिफ़ॉल्ट)                                                                                                                                | HTTP Request URLs                                                                                              |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| सार्वजनिक Gateway URL           | आवश्यक नहीं                                                                                                                                         | आवश्यक (DNS, TLS, रिवर्स प्रॉक्सी या टनल)                                                                   |
| आउटबाउंड नेटवर्क             | `wss-primary.slack.com` तक आउटबाउंड WSS पहुँच योग्य होना चाहिए                                                                                            | कोई आउटबाउंड WS नहीं; केवल इनबाउंड HTTPS                                                                             |
| आवश्यक टोकन                | बॉट पहचान: बॉट टोकन + `connections:write` वाला App-Level Token; उपयोगकर्ता पहचान: उपयोगकर्ता टोकन + App-Level Token                                      | बॉट पहचान: बॉट टोकन + Signing Secret; उपयोगकर्ता पहचान: उपयोगकर्ता टोकन + Signing Secret                           |
| डेवलपमेंट लैपटॉप / फ़ायरवॉल के पीछे | बिना बदलाव के काम करता है                                                                                                                                          | सार्वजनिक टनल (ngrok, Cloudflare Tunnel, Tailscale Funnel) या स्टेजिंग Gateway की आवश्यकता है                          |
| हॉरिज़ॉन्टल स्केलिंग           | प्रति होस्ट, प्रति ऐप एक Socket Mode सेशन; एकाधिक Gateway के लिए अलग-अलग Slack ऐप आवश्यक हैं                                                                 | स्टेटलेस POST हैंडलर; लोड बैलेंसर के पीछे एकाधिक Gateway प्रतिकृतियाँ एक ऐप साझा कर सकती हैं                     |
| एक Gateway पर एकाधिक अकाउंट | समर्थित; प्रत्येक अकाउंट अपना WS खोलता है                                                                                                             | समर्थित; प्रत्येक अकाउंट को एक विशिष्ट `webhookPath` (डिफ़ॉल्ट `/slack/events`) चाहिए, ताकि पंजीकरण आपस में न टकराएँ |
| स्लैश कमांड ट्रांसपोर्ट      | WS कनेक्शन पर डिलीवर किया जाता है; `slash_commands[].url` को अनदेखा किया जाता है                                                                                  | Slack, `slash_commands[].url` पर POST करता है; कमांड डिस्पैच करने के लिए फ़ील्ड आवश्यक है                           |
| अनुरोध हस्ताक्षर              | उपयोग नहीं किया जाता (प्रमाणीकरण App-Level Token है)                                                                                                               | Slack प्रत्येक अनुरोध पर हस्ताक्षर करता है; OpenClaw, `signingSecret` से सत्यापन करता है                                              |
| कनेक्शन टूटने पर पुनर्प्राप्ति  | Slack SDK का स्वतः पुनः कनेक्ट होना सक्षम है; OpenClaw विफल Socket Mode सेशन को सीमित बैकऑफ़ के साथ पुनः आरंभ भी करता है। Pong-timeout ट्रांसपोर्ट ट्यूनिंग लागू होती है। | टूटने के लिए कोई स्थायी कनेक्शन नहीं; पुनः प्रयास Slack द्वारा प्रत्येक अनुरोध के लिए किए जाते हैं                                           |

<Note>
  एकल-Gateway होस्ट, डेवलपमेंट लैपटॉप और ऐसे ऑन-प्रिमाइसेस नेटवर्क के लिए **Socket Mode चुनें**, जो आउटबाउंड `*.slack.com` तक पहुँच सकते हैं लेकिन इनबाउंड HTTPS स्वीकार नहीं कर सकते।

लोड बैलेंसर के पीछे एकाधिक Gateway प्रतिकृतियाँ चलाते समय, आउटबाउंड WSS अवरुद्ध लेकिन इनबाउंड HTTPS अनुमत होने पर, या जब Slack Webhook पहले से ही रिवर्स प्रॉक्सी पर समाप्त किए जाते हैं, तब **HTTP Request URLs चुनें**।
</Note>

<Warning>
  Slack एक ऐप के लिए एकाधिक Socket Mode कनेक्शन बनाए रख सकता है और प्रत्येक पेलोड को किसी भी कनेक्शन पर डिलीवर कर सकता है। इसलिए Slack ऐप साझा करने वाले अलग-अलग OpenClaw Gateway में समान रूटिंग और प्राधिकरण कॉन्फ़िगरेशन होना आवश्यक है। अन्यथा, प्रत्येक Gateway के लिए अलग Slack ऐप, एकल रिले इनग्रेस या लोड बैलेंसर के पीछे HTTP Request URLs का उपयोग करें। [Socket Mode का उपयोग करना](https://docs.slack.dev/apis/events-api/using-socket-mode#using-multiple-connections) देखें।
</Warning>

### रिले मोड

रिले मोड Slack इनग्रेस को OpenClaw Gateway से अलग करता है। एक विश्वसनीय राउटर एकल Slack Socket Mode कनेक्शन का स्वामी होता है, गंतव्य Gateway चुनता है और प्रमाणीकृत WebSocket पर टाइप किया गया इवेंट अग्रेषित करता है। आउटबाउंड Slack Web API कॉल के लिए Gateway अब भी अपने बॉट टोकन का उपयोग करता है।

```json5
{
  channels: {
    slack: {
      mode: "relay",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      relay: {
        url: "wss://router.example.com/gateway/ws",
        authToken: { source: "env", provider: "default", id: "SLACK_RELAY_AUTH_TOKEN" },
        gatewayId: "team-gateway",
      },
    },
  },
}
```

रिले URL को `wss://` का उपयोग करना चाहिए, जब तक कि उसका लक्ष्य localhost न हो। बेयरर टोकन और राउटर रूट तालिका को Slack प्राधिकरण सीमा का हिस्सा मानें: रूट किए गए इवेंट अधिकृत सक्रियण के रूप में सामान्य Slack संदेश हैंडलर में प्रवेश करते हैं। WebSocket `hello` फ़्रेम में राउटर द्वारा दिया गया `slack_identity`, डिफ़ॉल्ट आउटबाउंड उपयोगकर्ता नाम और आइकन सेट कर सकता है; कॉलर द्वारा स्पष्ट रूप से दी गई पहचान को फिर भी प्राथमिकता मिलती है। रिले कनेक्शन, Socket Mode के समान सीमित बैकऑफ़ समय के साथ पुनः कनेक्ट होता है और डिस्कनेक्ट होने पर राउटर द्वारा दी गई पहचान को हटा देता है।

### Enterprise Grid के संगठन-व्यापी इंस्टॉलेशन

एक Slack अकाउंट, Enterprise Grid के संगठन-व्यापी इंस्टॉलेशन द्वारा कवर किए गए प्रत्येक वर्कस्पेस से संदेश प्राप्त कर सकता है। सीधे Socket Mode या HTTP Request URLs चुनें; एंटरप्राइज़ अकाउंट के लिए रिले मोड समर्थित नहीं है। नीचे दिए गए दोनों न्यूनतम-विशेषाधिकार मैनिफ़ेस्ट केवल V1 `message` और `app_mention` इवेंट पथ, तत्काल उत्तर और लिसनर-स्वामित्व वाली स्थिति प्रतिक्रियाएँ सक्षम करते हैं।

#### Socket Mode

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack connector for OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

किसी Enterprise Grid Org Admin या Org Owner से ऐप अनुमोदित करवाएँ, उसे संगठन स्तर पर इंस्टॉल करें और वे वर्कस्पेस चुनें जिन्हें इंस्टॉलेशन कवर करता है। OpenClaw आरंभ करने से पहले पुष्टि करें कि ऐप प्रत्येक अपेक्षित वर्कस्पेस में उपलब्ध है। Socket Mode के लिए `connections:write` वाला ऐप-स्तरीय टोकन जनरेट करें, फिर संगठन इंस्टॉलेशन से बॉट टोकन कॉपी करें। संगठन में इंस्टॉल किए गए बॉट टोकन का उपयोग करने वाले अकाउंट को कॉन्फ़िगर करें:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      enterpriseOrgInstall: true,
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

#### HTTP Request URLs

HTTP मोड का उपयोग तब करें जब Gateway के पास सार्वजनिक HTTPS एंडपॉइंट हो और वह Socket Mode कनेक्शन न खोलता हो। उदाहरण URL को Gateway के सार्वजनिक `webhookPath` URL (डिफ़ॉल्ट `/slack/events`) से बदलें:

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack connector for OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

किसी Enterprise Grid Org Admin या Org Owner से ऐप अनुमोदित करवाएँ, उसे संगठन स्तर पर इंस्टॉल करें और वे वर्कस्पेस चुनें जिन्हें इंस्टॉलेशन कवर करता है। Slack द्वारा Request URL सत्यापित किए जाने के बाद, संगठन इंस्टॉलेशन का बॉट टोकन और ऐप का **Basic Information -> App Credentials -> Signing Secret** कॉपी करें। एंटरप्राइज़ अकाउंट को उसी Request URL पथ के साथ कॉन्फ़िगर करें:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      enterpriseOrgInstall: true,
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: {
        source: "env",
        provider: "default",
        id: "SLACK_SIGNING_SECRET",
      },
      webhookPath: "/slack/events",
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

स्टार्टअप पर OpenClaw, Slack `auth.test` के साथ `enterpriseOrgInstall` को सत्यापित करता है। फ़्लैग के बिना संगठन में इंस्टॉल किया गया टोकन, या फ़्लैग वाला वर्कस्पेस टोकन, स्टार्टअप को विफल कर देता है। किन वर्कस्पेस ने इंस्टॉलेशन की अनुमति दी है, इसके लिए Slack सत्य का स्रोत बना रहता है; फिर OpenClaw प्रत्येक डिलीवर किए गए इवेंट पर कॉन्फ़िगर की गई चैनल, उपयोगकर्ता, DM और उल्लेख नीतियाँ लागू करता है। Enterprise V1, डिस्पैच से पहले बॉट द्वारा बनाए गए सभी `message` और `app_mention` इवेंट को अस्वीकार करता है, चाहे `allowBots` कुछ भी हो, क्योंकि संगठन इंस्टॉलेशन लूप की रोकथाम के लिए स्थिर वर्कस्पेस-योग्य बॉट पहचान प्रदान नहीं करते हैं।

Enterprise समर्थन जानबूझकर सीधे Socket Mode या HTTP `message` और `app_mention` इवेंट तथा उनके तत्काल उत्तरों तक सीमित है। रिले मोड, स्लैश कमांड, इंटरैक्शन, App Home, प्रतिक्रिया इवेंट लिसनर, पिन, Slack एक्शन टूल, Slack-नेटिव अनुमोदन, बाइंडिंग, कतारबद्ध या निर्धारित डिलीवरी और सक्रिय प्रेषण किसी एंटरप्राइज़ अकाउंट के लिए उपलब्ध नहीं हैं। आउटबाउंड अभिस्वीकृति, टाइपिंग और स्थिति प्रतिक्रियाएँ लिसनर-स्वामित्व वाले Slack क्लाइंट के माध्यम से समर्थित हैं और इनके लिए `reactions:write` आवश्यक है; इनबाउंड प्रतिक्रिया सूचनाएँ और प्रतिक्रिया एक्शन टूल अनुपलब्ध रहते हैं।

तत्काल उत्तर खंडों, मीडिया, मेटाडेटा, पहचान फ़ॉलबैक, अनफ़र्ल और रसीदों के लिए मानक Slack डिलीवरी व्यवहार का पुनः उपयोग करते हैं, लेकिन केवल तब तक, जब तक सत्यापित लिसनर-स्वामित्व वाला क्लाइंट सक्रिय इवेंट टर्न में रहता है। इन-मेमोरी प्रेषण कतार और थ्रेड-भागीदारी रिकॉर्ड उस इवेंट के वर्कस्पेस के अनुसार विभाजित होते हैं; क्लाइंट स्वयं कभी सीरियलाइज़ या स्थायी रूप से संग्रहीत नहीं किया जाता।

चैनल नीति कुंजियों और `dm.groupChannels` प्रविष्टियों में मूल स्थिर Slack चैनल ID या
`channel:<id>` प्रारूप का उपयोग करना आवश्यक है। OpenClaw रनटाइम मिलान के लिए दोनों प्रारूपों को मूल चैनल ID में सामान्यीकृत करता है; `slack:`, `group:`, और `mpim:` उपसर्ग स्टार्टअप विफल कर देते हैं।
उपयोगकर्ता नीति प्रविष्टियों में स्थिर Slack उपयोगकर्ता ID का उपयोग करना आवश्यक है; नाम, स्लग, प्रदर्शन नाम और ईमेल पते स्टार्टअप विफल कर देते हैं। ID में Slack के मानक अपरकेस उपसर्ग और बॉडी (उदाहरण के लिए, `C0123456789` या `U0123456789`) का उपयोग करना आवश्यक है; लोअरकेस और छोटे मिलते-जुलते ID स्टार्टअप विफल कर देते हैं। एंटरप्राइज़ खाते
`dangerouslyAllowNameMatching` सक्षम नहीं कर सकते। एंटरप्राइज़ खाते वैश्विक
`mentionPatterns.mode` सेट कर सकते हैं, लेकिन `mentionPatterns.allowIn` और
`mentionPatterns.denyIn` स्टार्टअप विफल कर देते हैं, क्योंकि केवल Slack चैनल ID वर्कस्पेस-योग्य नहीं होते और विभिन्न वर्कस्पेस में पुनः उपयोग किए जा सकते हैं। वर्कस्पेस इंस्टॉलेशन मौजूदा दायरा-निर्धारित उल्लेख-पैटर्न व्यवहार बनाए रखते हैं। प्रत्येक स्वीकृत वर्कस्पेस को अलग रूटिंग, सत्र, ट्रांसक्रिप्ट, डीडुप्लिकेशन, इतिहास और कैश पहचान मिलती है, भले ही Slack ID परस्पर समान हों। `message` स्ट्रीम में सामान्य उपयोगकर्ता संदेश और उपयोगकर्ता-लिखित `file_share` इवेंट समर्थित हैं; अन्य संदेश उपप्रकार प्राधिकरण या सिस्टम-इवेंट प्रबंधन से पहले अस्वीकार कर दिए जाते हैं।

एंटरप्राइज़ DM या तो अक्षम होने चाहिए (`dm.enabled=false` या
`dmPolicy="disabled"`), या `dmPolicy="open"` और प्रभावी खाता
`allowFrom`, जिसमें शाब्दिक `"*"` शामिल हो, के साथ स्पष्ट रूप से खुले होने चाहिए। खाली अनुमति-सूची या `"*"` के बिना उपयोगकर्ता-विशिष्ट ID स्टार्टअप विफल कर देते हैं। पेयरिंग और प्रति-उपयोगकर्ता DM अनुमति-सूचियाँ अस्वीकार की जाती हैं, क्योंकि उन प्राधिकरण स्टोर में Slack उपयोगकर्ता ID वर्कस्पेस-योग्य नहीं होते। चैनल और प्रेषक नीति चैनल संदेशों पर लागू रहती है।

## इंस्टॉल करें

```bash
openclaw plugins install @openclaw/slack
```

`plugins install` Plugin को पंजीकृत और सक्षम करता है। जब तक आप नीचे Slack ऐप और चैनल सेटिंग कॉन्फ़िगर नहीं करते, यह कुछ नहीं करता। सामान्य Plugin इंस्टॉल नियमों के लिए [Plugins](/hi/tools/plugin) देखें।

## त्वरित सेटअप

इस अनुभाग के मैनिफ़ेस्ट वर्कस्पेस-दायरा वाला इंस्टॉलेशन बनाते हैं। Enterprise Grid संगठन इंस्टॉलेशन के लिए इसके बजाय समर्पित
[संगठन-व्यापी मैनिफ़ेस्ट और कार्यप्रवाह](#enterprise-grid-org-wide-installs) का उपयोग करें।

<Tabs>
  <Tab title="Socket Mode (डिफ़ॉल्ट)">
    <Steps>
      <Step title="नया Slack ऐप बनाएँ">
        [api.slack.com/apps](https://api.slack.com/apps/new) खोलें → **Create New App** → **From a manifest** → अपना वर्कस्पेस चुनें → नीचे दिए गए मैनिफ़ेस्ट में से कोई एक पेस्ट करें → **Next** → **Create**।

        <CodeGroup>

```json Recommended
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw के लिए Slack कनेक्टर"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw, Slack Agent View वार्तालापों को OpenClaw एजेंटों से जोड़ता है।",
      "suggested_prompts": [
        { "title": "आप क्या कर सकते हैं?", "message": "आप मेरी किस काम में सहायता कर सकते हैं?" },
        {
          "title": "इस चैनल का सारांश दें",
          "message": "इस चैनल की हाल की गतिविधि का सारांश दें।"
        },
        { "title": "उत्तर का मसौदा बनाएँ", "message": "उत्तर का मसौदा बनाने में मेरी सहायता करें।" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw को संदेश भेजें",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw के लिए Slack कनेक्टर"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw, Slack Agent View वार्तालापों को OpenClaw एजेंटों से जोड़ता है।",
      "suggested_prompts": [
        { "title": "आप क्या कर सकते हैं?", "message": "आप मेरी किस काम में सहायता कर सकते हैं?" },
        {
          "title": "इस चैनल का सारांश दें",
          "message": "इस चैनल की हाल की गतिविधि का सारांश दें।"
        },
        { "title": "उत्तर का मसौदा बनाएँ", "message": "उत्तर का मसौदा बनाने में मेरी सहायता करें।" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw को संदेश भेजें",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    }
  }
}
```

        </CodeGroup>

        <Note>
          **Recommended** Slack Plugin के पूर्ण फ़ीचर सेट से मेल खाता है: App Home, स्लैश कमांड, फ़ाइलें, प्रतिक्रियाएँ, पिन, समूह DM और इमोजी/उपयोगकर्ता-समूह रीड। जब वर्कस्पेस नीति स्कोप सीमित करती हो, तब **Minimal** चुनें — यह DM, चैनल/समूह इतिहास, उल्लेख और स्लैश कमांड को शामिल करता है, लेकिन फ़ाइलें, प्रतिक्रियाएँ, पिन, समूह-DM (`mpim:*`), `emoji:read`, और `usergroups:read` को हटा देता है। प्रति-स्कोप तर्क और अतिरिक्त स्लैश कमांड जैसे योगात्मक विकल्पों के लिए [मैनिफ़ेस्ट और स्कोप चेकलिस्ट](#manifest-and-scope-checklist) देखें।
        </Note>

        Slack द्वारा ऐप बनाए जाने के बाद:

        - **Basic Information -> App-Level Tokens -> Generate Token and Scopes**: `connections:write` जोड़ें, सहेजें और App-Level Token कॉपी करें।
        - **Install App -> Install to Workspace**: Bot User OAuth Token कॉपी करें।

      </Step>

      <Step title="OpenClaw कॉन्फ़िगर करें">

        अनुशंसित SecretRef सेटअप:

```bash
export SLACK_APP_TOKEN=slack-app-token-example
export SLACK_BOT_TOKEN=slack-bot-token-example
cat > slack.socket.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./slack.socket.patch.json5 --dry-run
openclaw config patch --file ./slack.socket.patch.json5
```

        एन्वायरनमेंट फ़ॉलबैक (केवल डिफ़ॉल्ट खाता):

```bash
SLACK_APP_TOKEN=slack-app-token-example
SLACK_BOT_TOKEN=slack-bot-token-example
```

      </Step>

      <Step title="Gateway शुरू करें">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="HTTP अनुरोध URL">
    <Steps>
      <Step title="नया Slack ऐप बनाएँ">
        [api.slack.com/apps](https://api.slack.com/apps/new) खोलें → **Create New App** → **From a manifest** → अपना वर्कस्पेस चुनें → नीचे दिए गए मैनिफ़ेस्ट में से कोई एक पेस्ट करें → `https://gateway-host.example.com/slack/events` को अपने सार्वजनिक Gateway URL से बदलें → **Next** → **Create**।

        <CodeGroup>

```json Recommended
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw के लिए Slack कनेक्टर"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw, Slack Agent View वार्तालापों को OpenClaw एजेंटों से जोड़ता है।",
      "suggested_prompts": [
        { "title": "आप क्या कर सकते हैं?", "message": "आप मेरी किस काम में सहायता कर सकते हैं?" },
        {
          "title": "इस चैनल का सारांश दें",
          "message": "इस चैनल की हाल की गतिविधि का सारांश दें।"
        },
        { "title": "उत्तर का मसौदा बनाएँ", "message": "उत्तर का मसौदा बनाने में मेरी सहायता करें।" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw को संदेश भेजें",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw के लिए Slack कनेक्टर"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw, Slack Agent View वार्तालापों को OpenClaw एजेंटों से जोड़ता है।",
      "suggested_prompts": [
        { "title": "आप क्या कर सकते हैं?", "message": "आप मेरी किस काम में सहायता कर सकते हैं?" },
        {
          "title": "इस चैनल का सारांश दें",
          "message": "इस चैनल की हाल की गतिविधि का सारांश दें।"
        },
        { "title": "उत्तर का मसौदा बनाएँ", "message": "उत्तर का मसौदा बनाने में मेरी सहायता करें।" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw को संदेश भेजें",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

        </CodeGroup>

        <Note>
          **अनुशंसित** Slack Plugin की पूरी सुविधा-श्रृंखला से मेल खाता है; **न्यूनतम** प्रतिबंधित कार्यस्थानों के लिए फ़ाइलें, प्रतिक्रियाएँ, पिन, समूह-DM (`mpim:*`), `emoji:read`, और `usergroups:read` हटा देता है। प्रत्येक स्कोप के औचित्य के लिए [मैनिफ़ेस्ट और स्कोप जाँच-सूची](#manifest-and-scope-checklist) देखें।
        </Note>

        <Info>
          तीनों URL फ़ील्ड (`slash_commands[].url`, `event_subscriptions.request_url`, और `interactivity.request_url` / `message_menu_options_url`) एक ही OpenClaw एंडपॉइंट की ओर संकेत करते हैं। Slack की मैनिफ़ेस्ट स्कीमा के अनुसार इनके नाम अलग-अलग होना आवश्यक है, लेकिन OpenClaw पेलोड प्रकार के अनुसार रूट करता है, इसलिए एक `webhookPath` (डिफ़ॉल्ट `/slack/events`) पर्याप्त है। `slash_commands[].url` के बिना स्लैश कमांड HTTP मोड में बिना कोई सूचना दिए निष्क्रिय रहते हैं।
        </Info>

        Slack द्वारा ऐप बनाए जाने के बाद:

        - **Basic Information → App Credentials**: अनुरोध सत्यापन के लिए **Signing Secret** कॉपी करें।
        - **Install App -> Install to Workspace**: Bot User OAuth Token कॉपी करें।

      </Step>

      <Step title="OpenClaw कॉन्फ़िगर करें">

        अनुशंसित SecretRef सेटअप:

```bash
export SLACK_BOT_TOKEN=slack-bot-token-example
export SLACK_SIGNING_SECRET=...
cat > slack.http.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: { source: "env", provider: "default", id: "SLACK_SIGNING_SECRET" },
      webhookPath: "/slack/events",
    },
  },
}
JSON5
openclaw config patch --file ./slack.http.patch.json5 --dry-run
openclaw config patch --file ./slack.http.patch.json5
```

        <Note>
        बहु-खाता HTTP के लिए अद्वितीय Webhook पथों का उपयोग करें

        प्रत्येक खाते को अलग `webhookPath` (डिफ़ॉल्ट `/slack/events`) दें, ताकि पंजीकरण आपस में न टकराएँ।
        </Note>

      </Step>

      <Step title="Gateway शुरू करें">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## उपयोगकर्ता पहचान (वास्तविक व्यक्ति के रूप में पोस्ट करें)

उपयोगकर्ता पहचान OpenClaw को Slack ऐप को अधिकृत करने वाले व्यक्ति के रूप में पढ़ने और पोस्ट करने देती है। `userToken` कार्यरत पहचान है; एक सहयोगी Slack ऐप Socket Mode या HTTP Request URL के माध्यम से Events API ट्रैफ़िक संभालता है। सहयोगी ऐप को बॉट उपयोगकर्ता या बॉट टोकन की आवश्यकता नहीं होती।

सहयोगी ऐप को इस प्रकार सेट अप करें:

1. **OAuth & Permissions -> User Token Scopes** के अंतर्गत, ये उपयोगकर्ता-स्कोप अनुमतियाँ जोड़ें:

   - इतिहास: `channels:history`, `groups:history`, `im:history`, `mpim:history`
   - वार्तालाप खोज: `channels:read`, `groups:read`, `im:read`, `mpim:read`
   - लोग: `users:read`
   - पोस्ट करना: `chat:write` (संदेश अधिकृत करने वाले उपयोगकर्ता के रूप में पोस्ट किए जाते हैं)
   - DM खोलना: `im:write`, `mpim:write`

2. **Event Subscriptions -> Subscribe to events on behalf of users** के अंतर्गत, ये उपयोगकर्ता इवेंट जोड़ें। इन्हें केवल बॉट-इवेंट सूची में न जोड़ें:

   - `message.channels`
   - `message.groups`
   - `message.im`
   - `message.mpim`

3. एक इवेंट ट्रांसपोर्ट चुनें:

   - **Socket Mode:** Socket Mode सक्षम करें और `connections:write` वाला ऐप-स्तरीय टोकन बनाएँ। इसे `appToken` के रूप में कॉन्फ़िगर करें।
   - **HTTP Request URL:** Event Subscriptions को सार्वजनिक OpenClaw Slack एंडपॉइंट पर निर्देशित करें और **Basic Information -> App Credentials -> Signing Secret** कॉपी करें। इसे `signingSecret` के रूप में कॉन्फ़िगर करें।

4. ऐप को इंस्टॉल या पुनः इंस्टॉल करें, इच्छित व्यक्ति के रूप में इसे अधिकृत करें, और प्राप्त उपयोगकर्ता OAuth टोकन को `userToken` में कॉपी करें।

Socket Mode कॉन्फ़िगरेशन:

```json5
{
  channels: {
    slack: {
      identity: "user",
      userToken: "<xoxp>",
      appToken: "<xapp>",
    },
  },
}
```

HTTP Request URL कॉन्फ़िगरेशन:

```json5
{
  channels: {
    slack: {
      identity: "user",
      mode: "http",
      userToken: "<xoxp>",
      signingSecret: "<signing-secret>",
      webhookPath: "/slack/events",
    },
  },
}
```

<Warning>
  DM और समूह DM केवल ऊपर दी गई उपयोगकर्ता-स्कोप इवेंट सदस्यता के माध्यम से काम करते हैं। कोई बॉट किसी व्यक्ति की 1:1 DM में शामिल नहीं हो सकता या किसी मौजूदा समूह DM में जोड़ा नहीं जा सकता। सहयोगी ऐप अदृश्य आधारभूत व्यवस्था है: Slack के अन्य सदस्यों को संदेश अधिकृत करने वाले व्यक्ति की ओर से दिखाई देते हैं, OpenClaw बॉट की ओर से नहीं।
</Warning>

OpenClaw समाधान की गई मानवीय पहचान द्वारा लिखे गए उपयोगकर्ता-स्कोप संदेश इवेंट को स्वचालित रूप से हटा देता है, इसलिए उसके द्वारा भेजे गए संदेश स्व-उत्तर ट्रिगर नहीं करते।

## Socket Mode ट्रांसपोर्ट ट्यूनिंग

OpenClaw, Socket Mode के लिए Slack SDK क्लाइंट पोंग टाइमआउट को डिफ़ॉल्ट रूप से 15 सेकंड पर सेट करता है। ट्रांसपोर्ट सेटिंग्स को केवल तभी ओवरराइड करें, जब आपको कार्यस्थान- या होस्ट-विशिष्ट ट्यूनिंग की आवश्यकता हो:

```json5
{
  channels: {
    slack: {
      mode: "socket",
      socketMode: {
        clientPingTimeout: 20000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
    },
  },
}
```

इसका उपयोग केवल उन Socket Mode कार्यस्थानों के लिए करें जो Slack वेबसॉकेट पोंग/सर्वर-पिंग टाइमआउट लॉग करते हैं या ऐसे होस्ट पर चलते हैं जहाँ इवेंट-लूप अवरोध ज्ञात है। `clientPingTimeout`, SDK द्वारा क्लाइंट पिंग भेजने के बाद पोंग की प्रतीक्षा अवधि है; `serverPingTimeout`, Slack सर्वर पिंग की प्रतीक्षा अवधि है। ऐप संदेश और इवेंट अनुप्रयोग स्थिति बने रहते हैं, ट्रांसपोर्ट सक्रियता संकेत नहीं।

टिप्पणियाँ:

- `socketMode` को HTTP Request URL मोड में अनदेखा किया जाता है।
- मूल `channels.slack.socketMode` सेटिंग्स सभी Slack खातों पर लागू होती हैं, जब तक कि उन्हें ओवरराइड न किया जाए। प्रति-खाता ओवरराइड `channels.slack.accounts.<accountId>.socketMode` का उपयोग करते हैं; क्योंकि यह एक ऑब्जेक्ट ओवरराइड है, उस खाते के लिए इच्छित प्रत्येक सॉकेट ट्यूनिंग फ़ील्ड शामिल करें।
- केवल `clientPingTimeout` का OpenClaw डिफ़ॉल्ट (`15000`) है। `serverPingTimeout` और `pingPongLoggingEnabled` केवल कॉन्फ़िगर किए जाने पर Slack SDK को दिए जाते हैं।
- Socket Mode पुनरारंभ बैकऑफ़ लगभग 2 सेकंड से शुरू होता है और लगभग 30 सेकंड तक सीमित रहता है। पुनर्प्राप्त करने योग्य आरंभ, आरंभ-प्रतीक्षा और डिस्कनेक्ट विफलताओं पर चैनल रुकने तक पुनः प्रयास किया जाता है। अमान्य प्रमाणीकरण, निरस्त टोकन या अनुपलब्ध स्कोप जैसी स्थायी खाता और क्रेडेंशियल त्रुटियाँ अनिश्चितकाल तक पुनः प्रयास करने के बजाय तुरंत विफल होती हैं।

## मैनिफ़ेस्ट और स्कोप जाँच-सूची

मूल Slack ऐप मैनिफ़ेस्ट Socket Mode और HTTP Request URLs दोनों के लिए समान है। केवल `settings` ब्लॉक (और स्लैश कमांड `url`) अलग होता है।

मूल मैनिफ़ेस्ट (Socket Mode डिफ़ॉल्ट):

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw के लिए Slack कनेक्टर"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw, Slack Agent View वार्तालापों को OpenClaw एजेंटों से जोड़ता है।",
      "suggested_prompts": [
        { "title": "आप क्या कर सकते हैं?", "message": "आप मेरी किस काम में सहायता कर सकते हैं?" },
        {
          "title": "इस चैनल का सारांश दें",
          "message": "इस चैनल की हाल की गतिविधि का सारांश दें।"
        },
        { "title": "उत्तर का मसौदा बनाएँ", "message": "उत्तर का मसौदा बनाने में मेरी सहायता करें।" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw को संदेश भेजें",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

**HTTP Request URLs मोड** के लिए, `settings` को HTTP प्रकार से बदलें और प्रत्येक स्लैश कमांड में `url` जोड़ें। सार्वजनिक URL आवश्यक है:

```json
{
  "features": {
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw को संदेश भेजें",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

### अतिरिक्त मैनिफ़ेस्ट सेटिंग्स

ऊपर दिए गए डिफ़ॉल्ट का विस्तार करने वाली अलग-अलग सुविधाएँ प्रदर्शित करें।

डिफ़ॉल्ट मैनिफ़ेस्ट Slack App Home के **Home** टैब को सक्षम करता है और `app_home_opened` की सदस्यता लेता है। जब किसी वर्कस्पेस का सदस्य Home टैब खोलता है, तो OpenClaw `views.publish` के साथ एक सुरक्षित डिफ़ॉल्ट Home दृश्य प्रकाशित करता है; इसमें कोई वार्तालाप पेलोड या निजी कॉन्फ़िगरेशन शामिल नहीं होता। जब एकल स्लैश कमांड मोड सक्षम होता है, तो कमांड संकेत में `channels.slack.slashCommand.name` का उपयोग होता है; नेटिव कमांड या बिना स्लैश कमांड वाले इंस्टॉलेशन उस संकेत को छोड़ देते हैं। Slack DM के लिए **Messages** टैब सक्षम रहता है। नए ऐप `features.agent_view`, `assistant:write`, और `app_context_changed` के माध्यम से Slack Agent View का उपयोग करते हैं। प्रत्येक दृश्यमान Agent View रूट अपने अलग OpenClaw थ्रेड सत्र पर रूट होता है, और Slack की क्रमबद्ध सक्रिय-दृश्य इकाइयाँ एजेंट तक केवल अविश्वसनीय संदर्भ के रूप में पहुँचती हैं।

पहले से `features.assistant_view` का उपयोग करने वाले मौजूदा ऐप अपना वर्तमान मैनिफ़ेस्ट रख सकते हैं। OpenClaw उन इंस्टॉलेशन के लिए `assistant_thread_started` और `assistant_thread_context_changed` को संभालना जारी रखता है। Slack Assistant View से Agent View में माइग्रेशन को अपरिवर्तनीय बनाता है और इसके बाद उपयोगकर्ताओं को हार्ड रिफ़्रेश करना आवश्यक होता है, इसलिए किसी मौजूदा ऐप पर `assistant_view` को तब तक न बदलें, जब तक आप पूरे वर्कस्पेस को माइग्रेट करने का इरादा न रखते हों।

<AccordionGroup>
  <Accordion title="वैकल्पिक नेटिव स्लैश कमांड">

    थोड़े अंतर के साथ एकल कॉन्फ़िगर किए गए कमांड के बजाय कई [नेटिव स्लैश कमांड](#commands-and-slash-behavior) का उपयोग किया जा सकता है:

    - `/status` के बजाय `/agentstatus` का उपयोग करें क्योंकि `/status` कमांड आरक्षित है।
    - Slack ऐप पर एक समय में 25 से अधिक स्लैश कमांड पंजीकृत नहीं किए जा सकते (Slack प्लेटफ़ॉर्म सीमा)।

    OpenClaw सक्षम नेटिव कमांड के लिए हैंडलर पंजीकृत करता है, लेकिन Slack मैनिफ़ेस्ट प्रविष्टियाँ व्यवस्थापक द्वारा प्रबंधित रहती हैं और रनटाइम पर सिंक्रोनाइज़ नहीं होतीं। मैनिफ़ेस्ट में `/login` को मैन्युअल रूप से जोड़ें; 25 कमांड की सीमा में बने रहने के लिए नीचे दिए गए उदाहरण में वैकल्पिक `/side` उपनाम के बजाय इसे शामिल किया गया है। `/login` को कहीं भी प्रदर्शित किया जा सकता है, लेकिन यह पेयरिंग कोड केवल निजी चैट या Web UI में जारी करता है।

    अपने मौजूदा `features.slash_commands` अनुभाग को [उपलब्ध कमांड](/hi/tools/slash-commands#command-list) के किसी उपसमुच्चय से बदलें:

    <Tabs>
      <Tab title="Socket Mode (डिफ़ॉल्ट)">

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "नया सत्र शुरू करें",
      "usage_hint": "[model]"
    },
    {
      "command": "/reset",
      "description": "वर्तमान सत्र रीसेट करें"
    },
    {
      "command": "/compact",
      "description": "सत्र संदर्भ को संक्षिप्त करें",
      "usage_hint": "[instructions]"
    },
    {
      "command": "/stop",
      "description": "वर्तमान रन रोकें"
    },
    {
      "command": "/session",
      "description": "थ्रेड-बाइंडिंग समाप्ति प्रबंधित करें",
      "usage_hint": "निष्क्रिय <duration|off> या अधिकतम आयु <duration|off>"
    },
    {
      "command": "/think",
      "description": "चिंतन स्तर सेट करें",
      "usage_hint": "<level>"
    },
    {
      "command": "/verbose",
      "description": "विस्तृत आउटपुट टॉगल करें",
      "usage_hint": "on|off|full"
    },
    {
      "command": "/fast",
      "description": "तेज़ मोड दिखाएँ या सेट करें",
      "usage_hint": "[status|on|off]"
    },
    {
      "command": "/reasoning",
      "description": "तर्क की दृश्यता टॉगल करें",
      "usage_hint": "[on|off|stream]"
    },
    {
      "command": "/elevated",
      "description": "उन्नत मोड टॉगल करें",
      "usage_hint": "[on|off|ask|full]"
    },
    {
      "command": "/exec",
      "description": "exec डिफ़ॉल्ट दिखाएँ या सेट करें",
      "usage_hint": "host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>"
    },
    {
      "command": "/approve",
      "description": "लंबित अनुमोदन अनुरोध स्वीकार या अस्वीकार करें",
      "usage_hint": "<id> <decision>"
    },
    {
      "command": "/model",
      "description": "मॉडल दिखाएँ या सेट करें",
      "usage_hint": "[name|#|status]"
    },
    {
      "command": "/models",
      "description": "प्रदाता/मॉडल सूचीबद्ध करें",
      "usage_hint": "[provider] [page] [limit=<n>|size=<n>|all]"
    },
    {
      "command": "/help",
      "description": "संक्षिप्त सहायता सारांश दिखाएँ"
    },
    {
      "command": "/commands",
      "description": "जनरेट किया गया कमांड कैटलॉग दिखाएँ"
    },
    {
      "command": "/tools",
      "description": "दिखाएँ कि वर्तमान एजेंट अभी क्या उपयोग कर सकता है",
      "usage_hint": "[compact|verbose]"
    },
    {
      "command": "/agentstatus",
      "description": "रनटाइम स्थिति दिखाएँ, जिसमें उपलब्ध होने पर प्रदाता उपयोग/कोटा शामिल हो"
    },
    {
      "command": "/tasks",
      "description": "वर्तमान सत्र के सक्रिय/हालिया बैकग्राउंड कार्य सूचीबद्ध करें"
    },
    {
      "command": "/context",
      "description": "समझाएँ कि संदर्भ कैसे संयोजित किया जाता है",
      "usage_hint": "[list|detail|json]"
    },
    {
      "command": "/whoami",
      "description": "अपनी प्रेषक पहचान दिखाएँ"
    },
    {
      "command": "/skill",
      "description": "नाम से कोई स्किल चलाएँ",
      "usage_hint": "<name> [input]"
    },
    {
      "command": "/btw",
      "description": "सत्र संदर्भ बदले बिना एक पूरक प्रश्न पूछें",
      "usage_hint": "<question>"
    },
    {
      "command": "/login",
      "description": "Codex लॉगिन पेयर करें",
      "usage_hint": "[codex|openai]"
    },
    {
      "command": "/usage",
      "description": "उपयोग फ़ुटर नियंत्रित करें या लागत सारांश दिखाएँ",
      "usage_hint": "off|tokens|full|cost"
    }
  ]
}
```

      </Tab>
      <Tab title="HTTP अनुरोध URL">
        ऊपर दिए गए Socket Mode की वही `slash_commands` सूची उपयोग करें और प्रत्येक प्रविष्टि में `"url": "https://gateway-host.example.com/slack/events"` जोड़ें। उदाहरण:

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "नया सत्र शुरू करें",
      "usage_hint": "[model]",
      "url": "https://gateway-host.example.com/slack/events"
    },
    {
      "command": "/help",
      "description": "संक्षिप्त सहायता सारांश दिखाएँ",
      "url": "https://gateway-host.example.com/slack/events"
    }
  ]
}
```

        सूची के प्रत्येक कमांड पर उस `url` मान को दोहराएँ।

      </Tab>
    </Tabs>

  </Accordion>
  <Accordion title="वैकल्पिक लेखकत्व स्कोप (लेखन ऑपरेशन)">
    यदि आप चाहते हैं कि आउटगोइंग संदेश डिफ़ॉल्ट Slack ऐप पहचान के बजाय सक्रिय एजेंट पहचान (कस्टम उपयोगकर्ता नाम और आइकन) का उपयोग करें, तो `chat:write.customize` बॉट स्कोप जोड़ें।

    यदि आप इमोजी आइकन का उपयोग करते हैं, तो Slack `:emoji_name:` सिंटैक्स की अपेक्षा करता है।

  </Accordion>
  <Accordion title="वैकल्पिक उपयोगकर्ता-टोकन स्कोप (पठन ऑपरेशन)">
    यदि आप `channels.slack.userToken` कॉन्फ़िगर करते हैं, तो सामान्य पठन स्कोप ये हैं:

    - `channels:history`, `groups:history`, `im:history`, `mpim:history`
    - `channels:read`, `groups:read`, `im:read`, `mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read` (यदि आप Slack खोज पठन पर निर्भर हैं)

  </Accordion>
</AccordionGroup>

## टोकन मॉडल

- बॉट पहचान (डिफ़ॉल्ट) के लिए Socket Mode में `botToken` + `appToken`, या HTTP मोड में `botToken` + `signingSecret` आवश्यक हैं।
- उपयोगकर्ता पहचान के लिए Socket Mode में `userToken` + `appToken`, या HTTP मोड में `userToken` + `signingSecret` आवश्यक हैं। यह बॉट टोकन का उपयोग नहीं करती।
- रिले मोड के लिए `botToken` के साथ `relay.url`, `relay.authToken`, और `relay.gatewayId` आवश्यक हैं; यह ऐप टोकन या साइनिंग सीक्रेट का उपयोग नहीं करता।
- `botToken`, `appToken`, `signingSecret`, `relay.authToken`, और `userToken` सादे टेक्स्ट
  स्ट्रिंग या SecretRef ऑब्जेक्ट स्वीकार करते हैं।
- कॉन्फ़िगरेशन टोकन एनवायरनमेंट फ़ॉलबैक को ओवरराइड करते हैं।
- `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`, और `SLACK_USER_TOKEN` एनवायरनमेंट फ़ॉलबैक में से प्रत्येक केवल डिफ़ॉल्ट खाते पर लागू होता है।
- `userToken` डिफ़ॉल्ट रूप से केवल-पठन व्यवहार (`userTokenReadOnly: true`) का उपयोग करता है।

स्थिति स्नैपशॉट व्यवहार:

- Slack खाता निरीक्षण प्रत्येक क्रेडेंशियल के `*Source` और `*Status`
  फ़ील्ड (`botToken`, `appToken`, `signingSecret`, `userToken`) ट्रैक करता है।
- स्थिति `available`, `configured_unavailable`, या `missing` होती है।
- `configured_unavailable` का अर्थ है कि खाता SecretRef
  या किसी अन्य गैर-इनलाइन सीक्रेट स्रोत के माध्यम से कॉन्फ़िगर किया गया है, लेकिन वर्तमान कमांड/रनटाइम पथ
  वास्तविक मान को हल नहीं कर सका।
- HTTP मोड में, `signingSecretStatus` शामिल होता है। Socket Mode बॉट पहचान के लिए
  `botTokenStatus` + `appTokenStatus` और उपयोगकर्ता पहचान के लिए
  `userTokenStatus` + `appTokenStatus` का उपयोग करता है।

<Tip>
बॉट पहचान के लिए, क्रियाएँ और डायरेक्टरी पठन वैकल्पिक उपयोगकर्ता टोकन को प्राथमिकता दे सकते हैं; लेखन में बॉट टोकन का उपयोग जारी रहता है, जब तक `userTokenReadOnly: false` फ़ॉलबैक की अनुमति न दे। `identity: "user"` के लिए, पठन और लेखन हमेशा `userToken` का उपयोग करते हैं।
</Tip>

## क्रियाएँ और गेट

Slack क्रियाएँ `channels.slack.actions.*` द्वारा नियंत्रित होती हैं।

वर्तमान Slack टूलिंग में उपलब्ध क्रिया समूह:

| समूह       | डिफ़ॉल्ट |
| ---------- | ------- |
| messages   | सक्षम |
| reactions  | सक्षम |
| pins       | सक्षम |
| memberInfo | सक्षम |
| emojiList  | सक्षम |

वर्तमान Slack संदेश क्रियाओं में `send`, `upload-file`, `download-file`, `read`, `edit`, `delete`, `pin`, `unpin`, `list-pins`, `member-info`, और `emoji-list` शामिल हैं। `download-file` इनबाउंड फ़ाइल प्लेसहोल्डर में दिखाई देने वाली Slack फ़ाइल ID स्वीकार करता है और छवियों के लिए छवि पूर्वावलोकन या अन्य फ़ाइल प्रकारों के लिए स्थानीय फ़ाइल मेटाडेटा लौटाता है।

## अभिगम नियंत्रण और रूटिंग

<Tabs>
  <Tab title="DM नीति">
    `channels.slack.dmPolicy` DM अभिगम को नियंत्रित करता है। `channels.slack.allowFrom` प्रामाणिक DM अनुमत-सूची है।

    - `pairing` (डिफ़ॉल्ट)
    - `allowlist`
    - `open` (`channels.slack.allowFrom` में `"*"` का शामिल होना आवश्यक है)
    - `disabled`

    DM फ़्लैग:

    - `dm.enabled` (डिफ़ॉल्ट true)
    - `channels.slack.allowFrom`
    - `dm.allowFrom` (विरासती)
    - `dm.groupEnabled` (समूह DM का डिफ़ॉल्ट false)
    - `dm.groupChannels` (वैकल्पिक MPIM अनुमत-सूची)

    बहु-खाता प्राथमिकता:

    - `channels.slack.accounts.default.allowFrom` केवल `default` खाते पर लागू होता है।
    - जब नामित खातों का अपना `allowFrom` सेट न हो, तो वे `channels.slack.allowFrom` इनहेरिट करते हैं।
    - नामित खाते `channels.slack.accounts.default.allowFrom` इनहेरिट नहीं करते।

    विरासती `channels.slack.dm.policy` और `channels.slack.dm.allowFrom` अब भी संगतता के लिए पढ़े जाते हैं। जब अभिगम बदले बिना ऐसा करना संभव हो, तो `openclaw doctor --fix` उन्हें `dmPolicy` और `allowFrom` में माइग्रेट करता है।

    DM में पेयरिंग `openclaw pairing approve slack <code>` का उपयोग करती है।

  </Tab>

  <Tab title="चैनल नीति">
    `channels.slack.groupPolicy` चैनल प्रबंधन को नियंत्रित करता है:

    - `open`
    - `allowlist`
    - `disabled`

    चैनल अनुमत-सूची `channels.slack.channels` के अंतर्गत होती है और कॉन्फ़िगरेशन कुंजियों के रूप में **स्थिर Slack चैनल ID का उपयोग करना अनिवार्य है** (उदाहरण के लिए `C12345678`)।

    रनटाइम टिप्पणी: यदि `channels.slack` पूरी तरह अनुपस्थित है (केवल एनवायरनमेंट वाला सेटअप), तो रनटाइम `groupPolicy="allowlist"` पर फ़ॉलबैक करता है और चेतावनी लॉग करता है (भले ही `channels.defaults.groupPolicy` सेट हो)।

    नाम/ID समाधान:

    - टोकन अभिगम की अनुमति होने पर चैनल अनुमत-सूची प्रविष्टियों और DM अनुमत-सूची प्रविष्टियों को स्टार्टअप पर हल किया जाता है
    - अनसुलझी चैनल-नाम प्रविष्टियाँ कॉन्फ़िगर किए गए रूप में रखी जाती हैं, लेकिन डिफ़ॉल्ट रूप से रूटिंग के लिए अनदेखी की जाती हैं
    - इनबाउंड प्राधिकरण और चैनल रूटिंग डिफ़ॉल्ट रूप से ID-प्रथम होते हैं; प्रत्यक्ष उपयोगकर्ता नाम/स्लग मिलान के लिए `channels.slack.dangerouslyAllowNameMatching: true` आवश्यक है

    <Warning>
    नाम-आधारित कुंजियाँ (`#channel-name` या `channel-name`) `groupPolicy: "allowlist"` के अंतर्गत मेल **नहीं** खातीं। चैनल लुकअप डिफ़ॉल्ट रूप से पहले ID का उपयोग करता है, इसलिए नाम-आधारित कुंजी कभी भी सफलतापूर्वक रूट नहीं होगी और उस चैनल के सभी संदेश बिना सूचना के ब्लॉक कर दिए जाएँगे। यह `groupPolicy: "open"` से अलग है, जहाँ रूटिंग के लिए चैनल कुंजी आवश्यक नहीं होती और नाम-आधारित कुंजी काम करती हुई दिखाई देती है।

    हमेशा Slack चैनल ID को कुंजी के रूप में उपयोग करें। इसे खोजने के लिए: Slack में चैनल पर राइट-क्लिक करें → **Copy link** — ID (`C...`) URL के अंत में दिखाई देती है।

    सही:

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            C12345678: { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```

    गलत (`groupPolicy: "allowlist"` के अंतर्गत बिना सूचना के ब्लॉक):

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            "#eng-my-channel": { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```
    </Warning>

  </Tab>

  <Tab title="उल्लेख और चैनल उपयोगकर्ता">
    चैनल संदेशों के लिए डिफ़ॉल्ट रूप से उल्लेख आवश्यक होता है।

    उल्लेख के स्रोत:

    - स्पष्ट ऐप उल्लेख (`<@botId>`)
    - Slack उपयोगकर्ता-समूह उल्लेख (`<!subteam^S...>`), जब बॉट उपयोगकर्ता उस उपयोगकर्ता समूह का सदस्य हो; `usergroups:read` आवश्यक है
    - उल्लेख रेगेक्स पैटर्न (`agents.entries.*.groupChat.mentionPatterns`, फ़ॉलबैक `messages.groupChat.mentionPatterns`)
    - बॉट के अपने Slack संदेश के उत्तर (`implicitMentions.replyToBot`)
    - उन थ्रेड में फ़ॉलो-अप जहाँ बॉट ने भाग लिया हो (`implicitMentions.threadParticipation`)

    प्रति-चैनल नियंत्रण (`channels.slack.channels.<id>`; नाम केवल स्टार्टअप समाधान या `dangerouslyAllowNameMatching` के माध्यम से):

    - `requireMention`
    - `ignoreOtherMentions`
    - `replyToMode` (`off|first|all|batched`; इस चैनल के लिए खाता/चैट-प्रकार उत्तर मोड को ओवरराइड करता है)
    - `users` (अनुमति-सूची)
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`, `toolsBySender`
    - `toolsBySender` कुंजी प्रारूप: `channel:`, `id:`, `e164:`, `username:`, `name:`, या `"*"` वाइल्डकार्ड
      (पुरानी उपसर्ग-रहित कुंजियाँ अब भी केवल `id:` से मैप होती हैं)

    `ignoreOtherMentions` (डिफ़ॉल्ट `false`) उन चैनल संदेशों को हटा देता है जो किसी अन्य उपयोगकर्ता या उपयोगकर्ता समूह का उल्लेख करते हैं, लेकिन इस बॉट का नहीं। DM और समूह DM (MPIM) अप्रभावित रहते हैं। फ़िल्टर को `auth.test` से समाधान किया गया बॉट उपयोगकर्ता ID चाहिए; यदि वह पहचान उपलब्ध नहीं है (उदाहरण के लिए केवल उपयोगकर्ता-टोकन वाली पहचान), तो गेट खुला विफल होता है और संदेश बिना बदलाव के आगे भेज दिए जाते हैं।

    `allowBots` चैनलों और निजी चैनलों के लिए रूढ़िवादी है: बॉट द्वारा लिखे गए रूम संदेश केवल तभी स्वीकार किए जाते हैं, जब भेजने वाला बॉट उस रूम की `users` अनुमति-सूची में स्पष्ट रूप से सूचीबद्ध हो, या `channels.slack.allowFrom` से कम-से-कम एक स्पष्ट Slack स्वामी ID वर्तमान में रूम का सदस्य हो। वाइल्डकार्ड और प्रदर्शन-नाम वाली स्वामी प्रविष्टियाँ स्वामी की उपस्थिति की शर्त पूरी नहीं करतीं। स्वामी की उपस्थिति Slack `conversations.members` का उपयोग करती है; सुनिश्चित करें कि ऐप के पास रूम प्रकार के लिए संबंधित रीड स्कोप हो (सार्वजनिक चैनलों के लिए `channels:read`, निजी चैनलों के लिए `groups:read`)। यदि सदस्य लुकअप विफल होता है, तो OpenClaw बॉट द्वारा लिखा गया रूम संदेश हटा देता है।

    स्वीकार किए गए, बॉट द्वारा लिखे गए Slack संदेश साझा [बॉट लूप सुरक्षा](/hi/channels/bot-loop-protection) का उपयोग करते हैं। डिफ़ॉल्ट बजट के लिए `channels.defaults.botLoopProtection` कॉन्फ़िगर करें, फिर जब किसी कार्यस्थान या चैनल को अलग सीमा की आवश्यकता हो, तो `channels.slack.botLoopProtection` या `channels.slack.channels.<id>.botLoopProtection` से ओवरराइड करें।

  </Tab>
</Tabs>

## थ्रेडिंग, सत्र और उत्तर टैग

- DM को `direct` के रूप में रूट किया जाता है; चैनलों को `channel` के रूप में; MPIM को `group` के रूप में।
- Slack रूट बाइंडिंग अपरिष्कृत पीयर ID के साथ-साथ `channel:C12345678`, `user:U12345678`, और `<@U12345678>` जैसे Slack लक्ष्य प्रारूप स्वीकार करती हैं।
- डिफ़ॉल्ट `session.dmScope=main` के साथ, सामान्य Slack DM एजेंट के मुख्य सत्र में समाहित हो जाते हैं। Agent View रूट और मौजूदा Assistant View थ्रेड `:thread:<threadTs>` सत्रों के रूप में अलग बने रहते हैं।
- चैनल सत्र: `agent:<agentId>:slack:channel:<channelId>`।
- सामान्य शीर्ष-स्तरीय चैनल संदेश प्रति-चैनल सत्र में बने रहते हैं, भले ही `replyToMode`, गैर-`off` हो।
- Slack चैनल, MPIM, Agent View और Assistant View के थ्रेड उत्तर सत्र प्रत्ययों (`:thread:<threadTs>`) के लिए पैरेंट Slack `thread_ts` का उपयोग करते हैं। सामान्य DM उत्तर थ्रेड मूल DM सत्र पर एक UI सुविधा बने रहते हैं।
- जब किसी योग्य शीर्ष-स्तरीय चैनल रूट से दृश्यमान Slack थ्रेड शुरू होने की अपेक्षा होती है, तो OpenClaw उस रूट को `agent:<agentId>:slack:channel:<channelId>:thread:<rootTs>` में सीड करता है, ताकि रूट और बाद के थ्रेड उत्तर एक OpenClaw सत्र साझा करें। यह `app_mention` इवेंट, स्पष्ट बॉट या कॉन्फ़िगर किए गए उल्लेख-पैटर्न के मेल, और गैर-`off` `replyToMode` वाले `requireMention: false` चैनलों पर लागू होता है।
- `channels.slack.thread.historyScope` का डिफ़ॉल्ट `thread` है; `thread.inheritParent` का डिफ़ॉल्ट `false` है।
- `channels.slack.thread.initialHistoryLimit` नियंत्रित करता है कि नया थ्रेड सत्र शुरू होने पर कितने मौजूदा थ्रेड संदेश प्राप्त किए जाएँ (डिफ़ॉल्ट `20`; अक्षम करने के लिए `0` सेट करें)।
- `channels.slack.implicitMentions.replyToBot` नियंत्रित करता है कि बॉट के अपने संदेश का उत्तर उल्लेख गेटिंग को बायपास करता है या नहीं (डिफ़ॉल्ट `true`)।
- `channels.slack.implicitMentions.threadParticipation` नियंत्रित करता है कि जिस थ्रेड में बॉट ने उत्तर दिया हो, उसके फ़ॉलो-अप उल्लेख गेटिंग को बायपास करते हैं या नहीं (डिफ़ॉल्ट `true`)। उन फ़ॉलो-अप में नया स्पष्ट उल्लेख आवश्यक करने के लिए इसे `false` पर सेट करें। `openclaw doctor --fix` पुरानी `channels.slack.thread.requireExplicitMention` कुंजी को इस सकारात्मक कैनोनिकल फ़्लैग में माइग्रेट करता है।
- खाता ओवरराइड `channels.slack.accounts.<id>.implicitMentions` पर होते हैं; साझा डिफ़ॉल्ट `channels.defaults.implicitMentions` पर होते हैं।

उत्तर थ्रेडिंग नियंत्रण:

- `channels.slack.channels.<id>.replyToMode`: Slack चैनल/निजी-चैनल संदेशों के लिए प्रति-चैनल ओवरराइड
- `channels.slack.replyToMode`: `off|first|all|batched` (डिफ़ॉल्ट `off`)
- `channels.slack.replyToModeByChatType`: प्रति `direct|group|channel`
- सीधी चैट के लिए पुराना फ़ॉलबैक: `channels.slack.dm.replyToMode`

मैन्युअल उत्तर टैग समर्थित हैं:

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

`message` टूल से स्पष्ट Slack थ्रेड उत्तरों के लिए, `replyBroadcast: true` को `action: "send"` और `threadId` या `replyTo` के साथ सेट करें, ताकि Slack से थ्रेड उत्तर को पैरेंट चैनल में भी प्रसारित करने के लिए कहा जा सके। यह Slack के `chat.postMessage` `reply_broadcast` फ़्लैग से मैप होता है और केवल टेक्स्ट या Block Kit प्रेषण के लिए समर्थित है, मीडिया अपलोड के लिए नहीं।

जब कोई `message` टूल कॉल Slack थ्रेड के भीतर चलता है और उसी चैनल को लक्षित करता है, तो OpenClaw सामान्यतः प्रभावी खाता, चैट-प्रकार या प्रति-चैनल `replyToMode` के अनुसार वर्तमान Slack थ्रेड प्राप्त करता है। स्वचालित उत्तर और उसी चैनल के `send` या `upload-file` कॉल समान प्रति-चैनल ओवरराइड का उपयोग करते हैं। इसके बजाय नया पैरेंट-चैनल संदेश बाध्य करने के लिए `action: "send"` या `action: "upload-file"` पर `topLevel: true` सेट करें। `threadId: null` भी समान शीर्ष-स्तरीय ऑप्ट-आउट के रूप में स्वीकार किया जाता है।

<Note>
`replyToMode="off"` वैकल्पिक आउटबाउंड Slack उत्तर थ्रेडिंग को अक्षम करता है, जिसमें स्पष्ट `[[reply_to_*]]` टैग शामिल हैं। Agent View और Assistant View, Slack द्वारा प्रबंधित थ्रेडेड अनुभव हैं, इसलिए उनके उत्तर और स्थिति इस सेटिंग की परवाह किए बिना दृश्यमान रूट पर बने रहते हैं। यह अन्य इनबाउंड Slack थ्रेड सत्रों को समतल नहीं करता। यह Telegram से अलग है, जहाँ `"off"` मोड में भी स्पष्ट टैग का पालन किया जाता है। Slack थ्रेड चैनल से संदेश छिपाते हैं, जबकि Telegram उत्तर इनलाइन दृश्यमान रहते हैं।
</Note>

## अभिस्वीकृति प्रतिक्रियाएँ

जब OpenClaw किसी इनबाउंड संदेश को संसाधित कर रहा होता है, तब `ackReaction` एक अभिस्वीकृति इमोजी भेजता है। `ackReactionScope` तय करता है कि वह इमोजी वास्तव में _कब_ भेजा जाए।

डिफ़ॉल्ट रूप से, अभिस्वीकृति स्थिर रहती है, जबकि Slack की मूल एजेंट/सहायक थ्रेड स्थिति बदलते हुए लोडिंग संदेशों के साथ प्रगति दिखाती है। इसके बजाय कतारबद्ध/सोच/टूल/पूर्ण/त्रुटि प्रतिक्रिया जीवनचक्र अपनाने के लिए `messages.statusReactions.enabled: true` सेट करें।

### इमोजी (`ackReaction`)

समाधान क्रम:

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- एजेंट पहचान इमोजी फ़ॉलबैक (`agents.entries.*.identity.emoji`, अन्यथा `"eyes"` / 👀)

टिप्पणियाँ:

- Slack शॉर्टकोड की अपेक्षा करता है (उदाहरण के लिए `"eyes"`)।
- Slack खाते या वैश्विक रूप से प्रतिक्रिया अक्षम करने के लिए `""` का उपयोग करें।

### दायरा (`messages.ackReactionScope`)

Slack प्रदाता `messages.ackReactionScope` से दायरा पढ़ता है (डिफ़ॉल्ट `"group-mentions"`)। वर्तमान में Slack-खाता या Slack-चैनल-स्तरीय ओवरराइड नहीं है; मान पूरे gateway के लिए वैश्विक है।

मान:

- `"all"`: परिवेशी रूम इवेंट सहित DM और समूहों में प्रतिक्रिया दें।
- `"direct"`: केवल DM में प्रतिक्रिया दें।
- `"group-all"`: परिवेशी रूम इवेंट को छोड़कर प्रत्येक समूह संदेश पर प्रतिक्रिया दें (DM में नहीं)।
- `"group-mentions"` (डिफ़ॉल्ट): समूहों में प्रतिक्रिया दें, लेकिन केवल तब जब बॉट का उल्लेख किया गया हो (या उन समूह उल्लेखनीयों में जिन्होंने ऑप्ट इन किया हो)। **DM शामिल नहीं हैं।**
- `"off"` / `"none"`: कभी प्रतिक्रिया न दें।

<Note>
डिफ़ॉल्ट दायरा (`"group-mentions"`) सीधे संदेशों या परिवेशी रूम इवेंट में अभिस्वीकृति प्रतिक्रियाएँ ट्रिगर नहीं करता। इनबाउंड Slack DM और शांत रूम इवेंट पर कॉन्फ़िगर किया गया `ackReaction` (उदाहरण के लिए `"eyes"`) देखने के लिए, `messages.ackReactionScope` को `"all"` पर सेट करें। `messages.ackReactionScope` को Slack प्रदाता के स्टार्टअप पर पढ़ा जाता है, इसलिए बदलाव प्रभावी करने के लिए gateway को पुनः आरंभ करना आवश्यक है।
</Note>

```json5
{
  messages: {
    ackReaction: "eyes",
    ackReactionScope: "all", // DM और समूहों में प्रतिक्रिया दें
  },
}
```

## टेक्स्ट स्ट्रीमिंग

`channels.slack.streaming` लाइव पूर्वावलोकन व्यवहार नियंत्रित करता है:

- `off`: लाइव पूर्वावलोकन स्ट्रीमिंग अक्षम करें।
- `partial` (डिफ़ॉल्ट): पूर्वावलोकन टेक्स्ट को नवीनतम आंशिक आउटपुट से बदलें।
- `block`: खंडित पूर्वावलोकन अपडेट जोड़ें।
- `progress`: जनरेट करते समय प्रगति स्थिति टेक्स्ट दिखाएँ, फिर अंतिम टेक्स्ट भेजें।
- `streaming.preview.toolProgress`: ड्राफ़्ट पूर्वावलोकन सक्रिय होने पर टूल/प्रगति अपडेट को उसी संपादित पूर्वावलोकन संदेश में रूट करें (डिफ़ॉल्ट: `true`)। अलग टूल/प्रगति संदेश बनाए रखने के लिए `false` सेट करें।
- `streaming.preview.commandText` / `streaming.progress.commandText`: अपरिष्कृत कमांड/निष्पादन टेक्स्ट छिपाते हुए संक्षिप्त टूल-प्रगति पंक्तियाँ बनाए रखने के लिए `status` पर सेट करें (डिफ़ॉल्ट: `raw`)।

संक्षिप्त प्रगति पंक्तियाँ बनाए रखते हुए अपरिष्कृत कमांड/निष्पादन टेक्स्ट छिपाएँ:

```json
{
  "channels": {
    "slack": {
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

जब `channels.slack.streaming.mode`, `partial` हो, तब `channels.slack.streaming.nativeTransport` Slack की मूल टेक्स्ट स्ट्रीमिंग को नियंत्रित करता है (डिफ़ॉल्ट: `true`)।

Slack के मूल प्रगति कार्य कार्ड प्रगति मोड के लिए ऑप्ट-इन हैं। कार्य चलते समय Slack-मूल योजना/कार्य कार्ड भेजने और पूर्ण होने पर उसी कार्य कार्ड को अपडेट करने के लिए `channels.slack.streaming.progress.nativeTaskCards` को `true` पर `channels.slack.streaming.mode="progress"` के साथ सेट करें। इस फ़्लैग के बिना, प्रगति मोड पोर्टेबल ड्राफ़्ट-पूर्वावलोकन व्यवहार बनाए रखता है।

- नेटिव टेक्स्ट स्ट्रीमिंग और Slack सहायक थ्रेड की स्थिति दिखाई देने के लिए एक उत्तर थ्रेड उपलब्ध होना आवश्यक है। थ्रेड चयन अब भी `replyToMode` का अनुसरण करता है।
- नेटिव स्ट्रीमिंग अनुपलब्ध होने या कोई उत्तर थ्रेड मौजूद न होने पर भी चैनल, समूह-चैट और शीर्ष-स्तरीय DM रूट सामान्य ड्राफ़्ट पूर्वावलोकन का उपयोग कर सकते हैं।
- शीर्ष-स्तरीय Slack DM डिफ़ॉल्ट रूप से थ्रेड से बाहर रहते हैं, इसलिए वे Slack का थ्रेड-शैली वाला नेटिव स्ट्रीम/स्थिति पूर्वावलोकन नहीं दिखाते; इसके बजाय OpenClaw DM में ड्राफ़्ट पूर्वावलोकन पोस्ट और संपादित करता है।
- मीडिया और गैर-टेक्स्ट पेलोड सामान्य डिलीवरी पर वापस चले जाते हैं।
- मीडिया/त्रुटि के अंतिम परिणाम लंबित पूर्वावलोकन संपादनों को रद्द कर देते हैं; योग्य टेक्स्ट/ब्लॉक के अंतिम परिणाम केवल तभी फ़्लश होते हैं, जब वे पूर्वावलोकन को उसी स्थान पर संपादित कर सकें।
- यदि उत्तर के बीच में स्ट्रीमिंग विफल हो जाती है, तो OpenClaw शेष पेलोड के लिए सामान्य डिलीवरी पर वापस चला जाता है।

Slack नेटिव टेक्स्ट स्ट्रीमिंग के बजाय ड्राफ़्ट पूर्वावलोकन का उपयोग करें:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "partial",
        nativeTransport: false,
      },
    },
  },
}
```

Slack नेटिव प्रगति कार्य कार्ड के लिए ऑप्ट इन करें:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "progress",
        progress: {
          nativeTaskCards: true,
          render: "rich",
        },
      },
    },
  },
}
```

पुरानी कुंजियाँ:

- `channels.slack.streamMode` (`replace | status_final | append`), `channels.slack.streaming.mode` का एक पुराना उपनाम है।
- बूलियन `channels.slack.streaming`, `channels.slack.streaming.mode` और `channels.slack.streaming.nativeTransport` का एक पुराना उपनाम है।
- शीर्ष-स्तरीय `channels.slack.chunkMode` और `channels.slack.nativeStreaming`, `channels.slack.streaming.chunkMode` और `channels.slack.streaming.nativeTransport` के पुराने उपनाम हैं।
- पुराने उपनाम रनटाइम पर नहीं पढ़े जाते; सहेजे गए Slack स्ट्रीमिंग कॉन्फ़िगरेशन को मानक कुंजियों में फिर से लिखने के लिए `openclaw doctor --fix` चलाएँ।

## टाइपिंग प्रतिक्रिया फ़ॉलबैक

जब OpenClaw किसी उत्तर को संसाधित कर रहा होता है, तब `typingReaction` आने वाले Slack संदेश में एक अस्थायी प्रतिक्रिया जोड़ता है और रन समाप्त होने पर उसे हटा देता है। यह थ्रेड उत्तरों के बाहर सबसे उपयोगी है, क्योंकि थ्रेड उत्तर डिफ़ॉल्ट "is typing..." स्थिति संकेतक का उपयोग करते हैं।

समाधान क्रम:

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

टिप्पणियाँ:

- Slack शॉर्टकोड की अपेक्षा करता है (उदाहरण के लिए `"hourglass_flowing_sand"`)।
- प्रतिक्रिया सर्वोत्तम-प्रयास के आधार पर होती है और उत्तर या विफलता पथ पूरा होने के बाद सफ़ाई का प्रयास स्वचालित रूप से किया जाता है।

## वॉइस इनपुट

आज Slack में OpenClaw से बोलने के लिए, OpenClaw ऐप को एक Slack ऑडियो क्लिप भेजें। Slackbot का डिक्टेशन माइक्रोफ़ोन Slack के स्वामित्व वाली एक अलग सुविधा है, ऐप API नहीं।

- **[Slackbot वॉइस डिक्टेशन](https://slack.com/help/articles/202026038-How-to-use-Slackbot)** उपयोगकर्ता की निजी Slackbot बातचीत के भीतर उपलब्ध होता है। Slack रिकॉर्डिंग को Slackbot प्रॉम्प्ट में बदल देता है, लेकिन Events API के माध्यम से तृतीय-पक्ष Slack ऐप्स को कोई ऑडियो फ़ाइल, डिक्टेशन इवेंट, प्रॉम्प्ट या इनपुट-स्रोत मार्कर नहीं भेजता। OpenClaw Slack Plugin इसे सक्षम या प्राप्त नहीं कर सकता।
- **[Slack ऑडियो क्लिप](https://slack.com/help/articles/4406235165587-Record-audio-and-video-clips-in-Slack)** संग्रहित Slack फ़ाइलें हैं, जिन्हें OpenClaw DM, चैनल या थ्रेड में पोस्ट किया जा सकता है। OpenClaw बॉट टोकन से सुलभ क्लिप डाउनलोड करता है, Slack के क्लिप MIME मेटाडेटा को सामान्यीकृत करता है और उसे साझा [ऑडियो ट्रांसक्रिप्शन पाइपलाइन](/hi/nodes/audio) से भेजता है। अनुशंसित ऐप मैनिफ़ेस्ट में आवश्यक `files:read` स्कोप शामिल होता है।

ऑडियो क्लिप और Slackbot डिक्टेशन के गोपनीयता संबंधी अर्थ अलग हैं: क्लिप Slack की फ़ाइल-प्रतिधारण नीति का पालन करती हैं और OpenClaw उन्हें ट्रांसक्रिप्शन के लिए डाउनलोड करता है, जबकि Slack के अनुसार डिक्टेशन ऑडियो संग्रहित नहीं किया जाता।

`requireMention: true` वाले चैनल में, बिना कैप्शन वाली ऑडियो क्लिप कॉन्फ़िगर किया गया उल्लेख पैटर्न बोलकर गेट को संतुष्ट कर सकती है (`agents.entries.*.groupChat.mentionPatterns`, जो उपलब्ध न होने पर `messages.groupChat.mentionPatterns` का उपयोग करता है)। OpenClaw क्लिप को डाउनलोड या ट्रांसक्राइब करने से पहले प्रेषक को अधिकृत करता है, फिर उसे केवल तभी स्वीकार करता है जब ट्रांसक्रिप्ट मेल खाता हो। विफल या मेल न खाने वाला अस्थायी ट्रांसक्रिप्ट डाउनलोड की गई क्लिप के साथ हटा दिया जाता है; उसे चैनल इतिहास में बनाए नहीं रखा जाता। मूल Slack `@bot` पहचान का अनुमान भाषण से नहीं लगाया जा सकता, इसलिए बोले गए नाम का पैटर्न कॉन्फ़िगर करें या टाइप किया हुआ उल्लेख शामिल करें। यदि ट्रांसक्रिप्ट प्रतिध्वनि सक्षम है, तो प्रतिध्वनि केवल स्वीकार किए जाने के बाद भेजी जाती है।

## मीडिया, खंडन और डिलीवरी

<AccordionGroup>
  <Accordion title="आने वाले अटैचमेंट">
    Slack फ़ाइल अटैचमेंट Slack द्वारा होस्ट किए गए निजी URL से डाउनलोड किए जाते हैं (टोकन-प्रमाणित अनुरोध प्रवाह) और फ़ेच सफल होने तथा आकार सीमाओं की अनुमति मिलने पर मीडिया स्टोर में लिखे जाते हैं। फ़ाइल प्लेसहोल्डर में Slack `fileId` शामिल होता है, ताकि एजेंट `download-file` से मूल फ़ाइल फ़ेच कर सकें।

    डाउनलोड सीमाबद्ध निष्क्रियता और कुल टाइमआउट का उपयोग करते हैं। यदि Slack फ़ाइल प्राप्ति रुक जाती है या विफल होती है, तो OpenClaw संदेश को संसाधित करना जारी रखता है और फ़ाइल प्लेसहोल्डर पर वापस चला जाता है।

    रनटाइम में आने वाले डेटा की आकार सीमा डिफ़ॉल्ट रूप से `20MB` होती है, जब तक कि `channels.slack.mediaMaxMb` द्वारा इसे ओवरराइड न किया जाए।

  </Accordion>

  <Accordion title="जाने वाला टेक्स्ट और फ़ाइलें">
    - टेक्स्ट खंड `channels.slack.textChunkLimit` का उपयोग करते हैं (डिफ़ॉल्ट `8000`, Slack की अपनी संदेश-लंबाई सीमा तक सीमित)
    - `channels.slack.streaming.chunkMode="newline"` अनुच्छेद-प्रथम विभाजन सक्षम करता है
    - फ़ाइल प्रेषण Slack अपलोड API का उपयोग करते हैं और उनमें थ्रेड उत्तर (`thread_ts`) शामिल हो सकते हैं
    - लंबे फ़ाइल कैप्शन पहले Slack-सुरक्षित टेक्स्ट खंड को अपलोड टिप्पणी के रूप में उपयोग करते हैं और शेष खंडों को अनुवर्ती संदेशों के रूप में भेजते हैं
    - कॉन्फ़िगर होने पर जाने वाले मीडिया की सीमा `channels.slack.mediaMaxMb` का अनुसरण करती है; अन्यथा चैनल प्रेषण मीडिया पाइपलाइन के MIME-प्रकार डिफ़ॉल्ट का उपयोग करते हैं

  </Accordion>

  <Accordion title="डिलीवरी लक्ष्य">
    प्राथमिकता वाले स्पष्ट लक्ष्य:

    - DM के लिए `user:<id>`
    - चैनलों के लिए `channel:<id>`

    केवल टेक्स्ट/ब्लॉक वाले Slack DM सीधे उपयोगकर्ता ID पर पोस्ट कर सकते हैं; फ़ाइल अपलोड और थ्रेड वाले प्रेषण पहले Slack बातचीत API के माध्यम से DM खोलते हैं, क्योंकि उन पथों को एक ठोस बातचीत ID की आवश्यकता होती है।

  </Accordion>
</AccordionGroup>

## कमांड और स्लैश व्यवहार

स्लैश कमांड Slack में या तो एकल कॉन्फ़िगर किए गए कमांड या कई नेटिव कमांड के रूप में दिखाई देते हैं। कमांड डिफ़ॉल्ट बदलने के लिए `channels.slack.slashCommand` कॉन्फ़िगर करें:

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

```txt
/openclaw /help
```

नेटिव कमांड के लिए आपके Slack ऐप में [अतिरिक्त मैनिफ़ेस्ट सेटिंग](#additional-manifest-settings) आवश्यक हैं और इसके बजाय वैश्विक कॉन्फ़िगरेशन में `channels.slack.commands.native: true` या `commands.native: true` से सक्षम किए जाते हैं।

- Slack के लिए नेटिव कमांड ऑटो-मोड **बंद** है, इसलिए `commands.native: "auto"` Slack नेटिव कमांड सक्षम नहीं करता।

```txt
/help
```

नेटिव आर्ग्युमेंट मेनू प्राथमिकता क्रम में निम्न में से किसी एक रूप में रेंडर होते हैं:

- पर्याप्त रूप से छोटे 3-5 विकल्प: एक ओवरफ़्लो ("...") मेनू
- 100 से अधिक विकल्प, जब एसिंक्रोनस विकल्प फ़िल्टरिंग उपलब्ध हो: बाहरी चयन
- 1-2 विकल्प, या ऐसा कोई विकल्प जिसका एन्कोड किया गया मान चयन के लिए बहुत लंबा हो: बटन ब्लॉक
- अन्यथा (6-100 विकल्प, या एसिंक्रोनस फ़िल्टरिंग के बिना 100 से अधिक): स्थिर चयन मेनू, प्रति मेनू 100 विकल्पों के खंडों में

```txt
/think
```

स्लैश सत्र `agent:<agentId>:slack:slash:<userId>` जैसी पृथक कुंजियों का उपयोग करते हैं और फिर भी `CommandTargetSessionKey` का उपयोग करके कमांड निष्पादन को लक्षित बातचीत सत्र तक रूट करते हैं।

## नेटिव चार्ट

Slack का सार्वजनिक [`data_visualization` Block Kit ब्लॉक](https://docs.slack.dev/reference/block-kit/blocks/data-visualization-block/)
संदेशों में लाइन, बार, एरिया और पाई चार्ट रेंडर करता है। OpenClaw पोर्टेबल
`presentation` `chart` ब्लॉक को उस नेटिव आकार में मैप करता है; सामान्य
`chat:write` संदेश पहुँच के अतिरिक्त किसी OAuth स्कोप,
फ़ाइल अपलोड, इमेज रेंडरर या Slack कॉन्फ़िगरेशन की आवश्यकता नहीं है।

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "bar",
      "title": "Quarterly revenue",
      "categories": ["Q1", "Q2"],
      "series": [{ "name": "Revenue", "values": [120, 145] }],
      "xLabel": "Quarter"
    }
  ]
}
```

नेटिव रेंडरिंग से पहले Slack की सीमाएँ लागू की जाती हैं:

- शीर्षक और वैकल्पिक अक्ष लेबल: 50 वर्ण
- पाई: 1-12 धनात्मक खंड
- लाइन/बार/एरिया: विशिष्ट नाम वाली 1-12 शृंखलाएँ और साझा 1-20 श्रेणियाँ
- खंड, श्रेणी और शृंखला लेबल: 20 वर्ण
- प्रत्येक शृंखला में हर श्रेणी के लिए एक परिमित मान होना आवश्यक है; गैर-पाई मान
  ऋणात्मक हो सकते हैं

हर नेटिव चार्ट में स्क्रीन रीडर, सूचनाओं, सत्र मिररिंग और ब्लॉक को रेंडर न कर सकने वाले
क्लाइंट के लिए शीर्ष-स्तरीय टेक्स्ट प्रस्तुति भी होती है। अन्य OpenClaw चैनलों को भेजी जाने वाली
मानक प्रस्तुतियों में वही नियतात्मक चार्ट डेटा टेक्स्ट के रूप में मिलता है, जब तक कि वे नेटिव चार्ट
समर्थन घोषित न करें। यदि चरणबद्ध रोलआउट के दौरान Slack चार्ट को `invalid_blocks` के साथ अस्वीकार करता है,
तो OpenClaw अस्वीकृत नेटिव डेटा ब्लॉक हटा देता है, किसी भी सहोदर नियंत्रण को बनाए रखता है और
चार्ट की संपूर्ण प्रस्तुति को दृश्यमान टेक्स्ट के रूप में भेजता है।

Slack वर्तमान में प्रति संदेश अधिकतम दो `data_visualization` ब्लॉक स्वीकार करता है। जब
किसी प्रस्तुति में दो से अधिक मान्य चार्ट होते हैं, तो OpenClaw उनका क्रम बनाए रखता है
और अनुवर्ती संदेशों में नेटिव रेंडरिंग जारी रखता है, जिसमें प्रत्येक संदेश में दो से अधिक
चार्ट नहीं होते।

Slack का [डेवलपर लॉन्च](https://docs.slack.dev/changelog/2026/06/16/block-kit-data-visualization-block/)
ब्लॉक को ऐप-संबंधी Block Kit सुविधा के रूप में दस्तावेज़ित करता है और कोई सशुल्क
प्लान प्रतिबंध प्रकाशित नहीं करता। Business+/Enterprise पात्रता वाली भाषा
Slackbot की स्वचालित AI चार्ट जनरेशन पर लागू होती है, जो पहले से संरचित
Block Kit चार्ट भेजने वाले ऐप से अलग है। चार्ट केवल संदेश ब्लॉक हैं, App
Home, मोडल या Canvas सामग्री नहीं।

## नेटिव तालिकाएँ

Slack का वर्तमान [`data_table` Block Kit ब्लॉक](https://docs.slack.dev/reference/block-kit/blocks/data-table-block/)
संदेशों में संरचित पंक्तियाँ और स्तंभ रेंडर करता है। OpenClaw एक स्पष्ट
पोर्टेबल `presentation` `table` ब्लॉक को `data_table` पर मैप करता है; यह Slack के
पुराने [`table` ब्लॉक](https://docs.slack.dev/reference/block-kit/blocks/table-block/) का उपयोग नहीं करता।
सामान्य `chat:write` संदेश पहुँच के अतिरिक्त किसी OAuth स्कोप या Slack कॉन्फ़िगरेशन
की आवश्यकता नहीं है।

```json
{
  "blocks": [
    {
      "type": "table",
      "caption": "Open pipeline",
      "headers": ["Account", "Stage", "ARR"],
      "rows": [
        ["Acme", "Won", 125000],
        ["Globex", "Review", 82000]
      ],
      "rowHeaderColumnIndex": 0
    }
  ]
}
```

OpenClaw हेडर और स्ट्रिंग सेल को Slack `raw_text` सेल में मैप करता है। संख्यात्मक सेल
`raw_number` में मैप होते हैं और नेटिव क्रमबद्धता तथा फ़िल्टरिंग के लिए परिमित संख्यात्मक मान
संरक्षित रहता है। मौजूद होने पर `rowHeaderColumnIndex` उस शून्य-आधारित
स्तंभ को Slack पंक्ति हेडर के रूप में चिह्नित करता है।

Slack की प्रकाशित `data_table` सीमाएँ नेटिव रेंडरिंग से पहले लागू की जाती हैं:

- 1-20 स्तंभ
- 1-100 डेटा पंक्तियाँ, साथ में हेडर पंक्ति
- प्रत्येक पंक्ति में सेल की समान संख्या
- एक संदेश में सभी तालिका सेल में कुल अधिकतम 10,000 वर्ण

जब तक संदेश समग्र वर्ण सीमा के भीतर रहता है, कई मान्य तालिका ब्लॉक नेटिव रूप से
रेंडर हो सकते हैं। जो तालिका नेटिव सीमा के भीतर रेंडर नहीं हो सकती, वह पंक्तियाँ या
सेल खोने के बजाय संपूर्ण नियतात्मक टेक्स्ट बन जाती है। यदि वह टेक्स्ट एक Slack संदेश
से अधिक हो जाता है, तो प्रेषण और स्लैश प्रतिक्रियाएँ क्रमबद्ध टेक्स्ट खंडों का उपयोग करती हैं।
तालिका संपादन किसी मौजूदा संदेश से पंक्तियों को चुपचाप काटने के बजाय स्पष्ट आकार त्रुटि के साथ
विफल होते हैं।

पोर्टेबल प्रस्तुति से बनाई गई प्रत्येक नेटिव तालिका में स्क्रीन रीडर, सूचनाओं, सत्र मिररिंग और
ब्लॉक को रेंडर न कर सकने वाले क्लाइंट के लिए शीर्ष-स्तरीय टेक्स्ट
प्रस्तुति भी होती है। फ़ॉलबैक में कच्चे चार्ट और तालिका मान अक्षरशः बने
रहते हैं, इसलिए `<@U123>` जैसा सेल डेटा Slack मेंशन नहीं बनता।
यदि Slack नेटिव चार्ट या तालिका ब्लॉक को `invalid_blocks` के साथ अस्वीकार करता है, तो OpenClaw
एक सीमित पुनर्प्राप्ति चरण में प्रत्येक नेटिव डेटा ब्लॉक हटा देता है, बटन और चयन जैसे
मान्य सहोदर ब्लॉक बनाए रखता है, और Slack फ़ॉर्मैटिंग अक्षम करके पूरा दृश्यमान चार्ट
और तालिका टेक्स्ट भेजता है। स्लैश-कमांड डिलीवरी पूरे कमांड में
Slack के पाँच-कॉल वाले `response_url` बजट का हिसाब रखती है। प्रत्येक
उत्तर बैच से पहले, यह ऐसा पूरा प्लान चुनती है जो शेष कॉल में समा जाए या उस बैच को पोस्ट
करने से पहले विफल हो जाती है।

केवल स्पष्ट `presentation` तालिका ब्लॉक को नेटिव तालिकाओं में उन्नत किया जाता है।
Markdown पाइप तालिकाएँ लिखित टेक्स्ट ही रहती हैं; OpenClaw तालिका
संरचना या सेल प्रकारों का अनुमान नहीं लगाता। मौजूदा विश्वसनीय Slack-नेटिव उत्पादक
`channelData.slack.blocks` के माध्यम से कच्चे ब्लॉक भेजना जारी रख सकते हैं; OpenClaw मान्य कच्चे
`data_table` सेल से फ़ॉलबैक टेक्स्ट प्राप्त करता है, जबकि विकृत कस्टम ब्लॉक
अपने कैप्शन या सामान्य Block Kit फ़ॉलबैक तक सीमित हो सकते हैं। पोर्टेबल एजेंट, CLI
और plugin आउटपुट को `presentation` का उपयोग करना चाहिए।

## इंटरैक्टिव उत्तर

Slack एजेंट द्वारा बनाए गए इंटरैक्टिव उत्तर नियंत्रण रेंडर कर सकता है, लेकिन यह सुविधा डिफ़ॉल्ट रूप से अक्षम है।
नए एजेंट, CLI और plugin आउटपुट के लिए साझा
`presentation` बटन या चयन ब्लॉक को प्राथमिकता दें। वे उसी Slack इंटरैक्शन
पथ का उपयोग करते हैं और अन्य चैनलों पर भी उपयुक्त रूप से सीमित हो जाते हैं।

इसे वैश्विक रूप से सक्षम करें:

```json5
{
  channels: {
    slack: {
      capabilities: {
        interactiveReplies: true,
      },
    },
  },
}
```

या इसे केवल एक Slack खाते के लिए सक्षम करें:

```json5
{
  channels: {
    slack: {
      accounts: {
        ops: {
          capabilities: {
            interactiveReplies: true,
          },
        },
      },
    },
  },
}
```

सक्षम होने पर, एजेंट अब भी अप्रचलित केवल-Slack उत्तर निर्देश उत्सर्जित कर सकते हैं:

- `[[slack_buttons: Approve:approve, Reject:reject]]`
- `[[slack_select: Choose a target | Canary:canary, Production:production]]`

ये निर्देश Slack Block Kit में संकलित होते हैं और क्लिक या चयन को
मौजूदा Slack इंटरैक्शन इवेंट पथ से वापस रूट करते हैं। इन्हें पुराने
प्रॉम्प्ट और Slack-विशिष्ट वैकल्पिक उपायों के लिए रखें; नए पोर्टेबल
नियंत्रणों के लिए साझा प्रस्तुति का उपयोग करें।

नई उत्पादक कोड के लिए निर्देश कंपाइलर API भी अप्रचलित हैं:

- `compileSlackInteractiveReplies(...)`
- `parseSlackOptionsLine(...)`
- `isSlackInteractiveRepliesEnabled(...)`
- `buildSlackInteractiveBlocks(...)`

नए Slack-रेंडर किए गए नियंत्रणों के लिए `presentation` पेलोड और
`buildSlackPresentationBlocks(...)` का उपयोग करें।

टिप्पणियाँ:

- यह Slack-विशिष्ट विरासती UI है। अन्य चैनल Slack Block
  Kit निर्देशों को अपने बटन सिस्टम में रूपांतरित नहीं करते।
- इंटरैक्टिव कॉलबैक मान OpenClaw द्वारा बनाए गए अपारदर्शी टोकन हैं, एजेंट द्वारा लिखे गए कच्चे मान नहीं।
- यदि बनाए गए इंटरैक्टिव ब्लॉक Slack Block Kit की सीमाओं से अधिक होंगे, तो OpenClaw अमान्य ब्लॉक पेलोड भेजने के बजाय मूल टेक्स्ट उत्तर पर वापस जाता है।

### Plugin-स्वामित्व वाले मोडल सबमिशन

इंटरैक्टिव हैंडलर पंजीकृत करने वाले Slack plugin, OpenClaw द्वारा
एजेंट-दृश्य सिस्टम इवेंट के लिए पेलोड को संक्षिप्त करने से पहले मोडल
`view_submission` और `view_closed` जीवनचक्र इवेंट भी प्राप्त कर सकते हैं।
Slack मोडल खोलते समय इनमें से किसी एक रूटिंग
पैटर्न का उपयोग करें:

- `callback_id` को `openclaw:<namespace>:<payload>` पर सेट करें।
- या मौजूदा `callback_id` बनाए रखें और मोडल `private_metadata` में `pluginInteractiveData:
"<namespace>:<payload>"` रखें।

हैंडलर को `ctx.interaction.kind`, `view_submission` या
`view_closed` के रूप में, सामान्यीकृत `inputs`, और Slack का पूरा कच्चा
`stateValues` ऑब्जेक्ट मिलता है। केवल कॉलबैक-ID रूटिंग plugin हैंडलर को लागू करने के लिए पर्याप्त है; जब
मोडल को एजेंट-दृश्य सिस्टम इवेंट भी बनाना हो, तब मौजूदा मोडल
`private_metadata` उपयोगकर्ता/सत्र रूटिंग फ़ील्ड शामिल करें। एजेंट को
एक संक्षिप्त, संशोधित `Slack interaction: ...` सिस्टम इवेंट मिलता है। यदि हैंडलर
`systemEvent.summary`, `systemEvent.reference`, या `systemEvent.data` लौटाता है, तो वे
फ़ील्ड उस संक्षिप्त इवेंट में शामिल होते हैं, ताकि एजेंट पूरे फ़ॉर्म पेलोड को देखे बिना
plugin-स्वामित्व वाले स्टोरेज को संदर्भित कर सके।

## Slack में नेटिव अनुमोदन

Slack, Web UI या टर्मिनल पर फ़ॉलबैक करने के बजाय, इंटरैक्टिव बटन और इंटरैक्शन वाला नेटिव अनुमोदन क्लाइंट बन सकता है।

- Exec और plugin अनुमोदन Slack-नेटिव Block Kit प्रॉम्प्ट के रूप में रेंडर हो सकते हैं।
- `channels.slack.execApprovals.*` नेटिव exec अनुमोदन क्लाइंट सक्षमता और DM/चैनल रूटिंग कॉन्फ़िगरेशन बना रहता है।
- Exec अनुमोदन DM, `channels.slack.execApprovals.approvers` या `commands.ownerAllowFrom` का उपयोग करते हैं।
- Plugin अनुमोदन Slack-नेटिव बटनों का उपयोग करते हैं, जब मूल सत्र के लिए Slack नेटिव अनुमोदन क्लाइंट के रूप में सक्षम हो, या जब `approvals.plugin` मूल Slack सत्र या किसी Slack लक्ष्य पर रूट करता हो।
- Plugin अनुमोदन DM, `channels.slack.allowFrom` से Slack plugin अनुमोदकों, नामित-खाता `allowFrom`, या खाते के डिफ़ॉल्ट रूट का उपयोग करते हैं।
- अनुमोदक प्राधिकरण अब भी लागू होता है: केवल-exec अनुमोदक plugin अनुरोधों को तब तक अनुमोदित नहीं कर सकते, जब तक वे plugin अनुमोदक भी न हों।

यह अन्य चैनलों के समान साझा अनुमोदन बटन सतह का उपयोग करता है। जब आपकी Slack ऐप सेटिंग में `interactivity` सक्षम हो, तो अनुमोदन प्रॉम्प्ट बातचीत में सीधे Block Kit बटन के रूप में रेंडर होते हैं।
जब वे बटन मौजूद हों, तो वे प्राथमिक अनुमोदन UX होते हैं; OpenClaw को
मैन्युअल `/approve` कमांड केवल तभी शामिल करना चाहिए जब टूल परिणाम बताए कि चैट
अनुमोदन अनुपलब्ध हैं या मैन्युअल अनुमोदन ही एकमात्र पथ है।

कॉन्फ़िगरेशन पथ:

- `channels.slack.execApprovals.enabled`
- `channels.slack.execApprovals.approvers` (वैकल्पिक; संभव होने पर `commands.ownerAllowFrom` पर फ़ॉलबैक करता है)
- `channels.slack.execApprovals.target` (`dm` | `channel` | `both`, डिफ़ॉल्ट: `dm`)
- `agentFilter`, `sessionFilter`

जब `enabled` सेट न हो या `"auto"` हो और कम-से-कम एक
exec अनुमोदक हल हो जाए, तो Slack नेटिव exec अनुमोदन स्वतः सक्षम कर देता है। जब Slack plugin अनुमोदक हल हो जाएँ और अनुरोध नेटिव-क्लाइंट फ़िल्टर से मेल खाए, तो Slack इस नेटिव-क्लाइंट
पथ के माध्यम से नेटिव plugin अनुमोदन भी संभाल सकता है। Slack को नेटिव अनुमोदन क्लाइंट के रूप में स्पष्ट रूप से अक्षम करने के लिए
`enabled: false` सेट करें। अनुमोदक हल होने पर नेटिव अनुमोदन बाध्य करने के लिए `enabled: true` सेट करें।
Slack exec अनुमोदन अक्षम करने से `approvals.plugin` के माध्यम से सक्षम नेटिव Slack plugin अनुमोदन
डिलीवरी अक्षम नहीं होती; इसके बजाय plugin अनुमोदन डिलीवरी Slack plugin अनुमोदकों का उपयोग करती है।

बिना स्पष्ट Slack exec अनुमोदन कॉन्फ़िगरेशन के डिफ़ॉल्ट व्यवहार:

```json5
{
  commands: {
    ownerAllowFrom: ["slack:U12345678"],
  },
}
```

स्पष्ट Slack-नेटिव कॉन्फ़िगरेशन की आवश्यकता केवल तब होती है, जब आप अनुमोदकों को ओवरराइड करना, फ़िल्टर जोड़ना, या
मूल-चैट डिलीवरी चुनना चाहते हों:

```json5
{
  channels: {
    slack: {
      execApprovals: {
        enabled: true,
        approvers: ["U12345678"],
        target: "both",
      },
    },
  },
}
```

साझा `approvals.exec` फ़ॉरवर्डिंग अलग है। इसका उपयोग केवल तब करें, जब exec अनुमोदन प्रॉम्प्ट को
अन्य चैट या स्पष्ट आउट-ऑफ़-बैंड लक्ष्यों पर भी रूट करना आवश्यक हो। साझा `approvals.plugin` फ़ॉरवर्डिंग भी
अलग है; Slack नेटिव डिलीवरी उस फ़ॉलबैक को केवल तभी रोकती है, जब Slack plugin
अनुमोदन अनुरोध को नेटिव रूप से संभाल सकता हो।

समान-चैट `/approve` उन Slack चैनलों और DM में भी काम करता है जो पहले से कमांड का समर्थन करते हैं। पूरे अनुमोदन फ़ॉरवर्डिंग मॉडल के लिए [Exec अनुमोदन](/hi/tools/exec-approvals) देखें।

## इवेंट और परिचालन व्यवहार

- संदेश संपादन/हटाना सिस्टम इवेंट में मैप किए जाते हैं।
- थ्रेड प्रसारण ("Also send to channel" थ्रेड उत्तर) सामान्य उपयोगकर्ता संदेशों के रूप में संसाधित होते हैं।
- प्रतिक्रिया जोड़ने/हटाने के इवेंट सिस्टम इवेंट में मैप किए जाते हैं।
- सदस्य का शामिल होना/छोड़ना, चैनल का बनना/नाम बदलना, और पिन जोड़ने/हटाने के इवेंट सिस्टम इवेंट में मैप किए जाते हैं।
- वैकल्पिक उपस्थिति पोलिंग, देखे गए मानव प्रतिभागी के `away` से `active` संक्रमण को प्रतिभागी के सबसे हाल में सक्रिय पात्र Slack सत्र में मैप कर सकती है। डिफ़ॉल्ट रूप से यह बंद है।
- `configWrites` सक्षम होने पर `channel_id_changed` चैनल कॉन्फ़िगरेशन कुंजियों को माइग्रेट कर सकता है।
- चैनल विषय/उद्देश्य मेटाडेटा को अविश्वसनीय संदर्भ माना जाता है और इसे रूटिंग संदर्भ में इंजेक्ट किया जा सकता है।
- Agent View `app_context` इकाइयों को Slack प्रासंगिकता क्रम में सत्यापित किया जाता है और केवल संरचित अविश्वसनीय संदर्भ के रूप में उजागर किया जाता है; संदर्भ छोड़े जाने पर पुरानी इकाइयों का पुनः उपयोग करने के बजाय टर्न साफ़ हो जाता है।
- जहाँ लागू हो, थ्रेड आरंभकर्ता और आरंभिक थ्रेड-इतिहास संदर्भ सीडिंग को कॉन्फ़िगर की गई प्रेषक अनुमति-सूचियों के अनुसार फ़िल्टर किया जाता है।
- ब्लॉक क्रियाएँ, शॉर्टकट और मोडल इंटरैक्शन विस्तृत पेलोड फ़ील्ड वाले संरचित `Slack interaction: ...` सिस्टम इवेंट उत्सर्जित करते हैं:
  - ब्लॉक क्रियाएँ: चयनित मान, लेबल, पिकर मान, और `workflow_*` मेटाडेटा
  - वैश्विक शॉर्टकट: कॉलबैक और कर्ता मेटाडेटा, कर्ता के प्रत्यक्ष सत्र पर रूट किया गया
  - संदेश शॉर्टकट: कॉलबैक, कर्ता, चैनल, थ्रेड, और चयनित-संदेश संदर्भ
  - रूट किए गए चैनल मेटाडेटा और फ़ॉर्म इनपुट वाले मोडल `view_submission` और `view_closed` इवेंट

अपने Slack ऐप कॉन्फ़िगरेशन में वैश्विक या संदेश शॉर्टकट परिभाषित करें और किसी भी गैर-रिक्त कॉलबैक ID का उपयोग करें। OpenClaw मेल खाने वाले शॉर्टकट पेलोड को स्वीकार करता है, अन्य Slack इंटरैक्शन जैसी ही DM/चैनल प्रेषक नीति लागू करता है, और रूट किए गए एजेंट सत्र के लिए स्वच्छ किया गया इवेंट कतार में डालता है। ट्रिगर ID और प्रतिक्रिया URL को एजेंट संदर्भ से संशोधित कर दिया जाता है।

### उपस्थिति इवेंट

Slack, Events API या Socket Mode के माध्यम से उपस्थिति परिवर्तन नहीं भेजता। इसके बजाय OpenClaw उन मानव प्रतिभागियों के लिए [`users.getPresence`](https://docs.slack.dev/reference/methods/users.getPresence/) को पोल कर सकता है, जिनके संदेश सामान्य Slack पहुँच और रूटिंग जाँच में सफल हुए हों।

```json5
{
  channels: {
    slack: {
      presenceEvents: { mode: "auto" },
      channels: {
        C0123456789: { presenceEvents: { mode: "on" } },
        C0987654321: { presenceEvents: { mode: "off" } },
      },
    },
  },
}
```

- `off` (डिफ़ॉल्ट): कोई उपस्थिति टाइमर या Slack API कॉल नहीं।
- `auto`: पिछले 24 घंटों में सक्रिय DM, MPIM और Slack थ्रेड की निगरानी करें, जिनमें अधिकतम 8 देखे गए मानव प्रतिभागी हों। शीर्ष-स्तरीय चैनल सत्र शामिल नहीं हैं।
- `on`: प्रतिभागी सीमा के बिना उन्हीं वार्तालापों की निगरानी करें और शीर्ष-स्तरीय चैनल सत्र शामिल करें। किसी एक चैनल को बाध्य या निषिद्ध करने के लिए प्रति-चैनल ओवरराइड का उपयोग करें।

OpenClaw प्रत्येक Slack खाते के लिए प्रति मिनट अधिकतम 45 अद्वितीय उपयोगकर्ताओं को पोल करता है, एजेंट को जगाए बिना पहला परिणाम सीड करता है, और केवल देखे गए `away` से `active` संक्रमण पर जगाता है। प्रत्येक Slack खाते और उपयोगकर्ता पर स्थायी 8-घंटे का कूलडाउन लागू होता है, भले ही वह व्यक्ति कई थ्रेड में भाग लेता हो। इवेंट केवल उस व्यक्ति की सबसे हाल में सक्रिय पात्र बातचीत पर रूट होता है और एजेंट को एक छोटा अभिवादन भेजने का निर्णय लेने से पहले मेमोरी/wiki और ज्ञात समय-क्षेत्र संदर्भ देखने के लिए कहता है। एजेंट मौन रह सकता है।

बॉट टोकन को `users:read` की आवश्यकता होती है, जो अनुशंसित मैनिफ़ेस्ट में पहले से शामिल है। Enterprise Grid के संगठन-व्यापी इंस्टॉल के लिए उपस्थिति इवेंट उपलब्ध नहीं हैं।

## कॉन्फ़िगरेशन संदर्भ

प्राथमिक संदर्भ: [कॉन्फ़िगरेशन संदर्भ - Slack](/hi/gateway/config-channels#slack)।

<Accordion title="उच्च-संकेत वाले Slack फ़ील्ड">

- मोड/प्रमाणीकरण: `identity`, `mode`, `enterpriseOrgInstall`, `botToken`, `appToken`, `userToken`, `signingSecret`, `webhookPath`, `accounts.*`
- DM पहुँच: `dm.enabled`, `dmPolicy`, `allowFrom` (पुराना: `dm.policy`, `dm.allowFrom`), `dm.groupEnabled`, `dm.groupChannels`
- संगतता टॉगल: `dangerouslyAllowNameMatching` (आपातकालीन उपयोग; आवश्यकता न होने पर बंद रखें)
- चैनल पहुँच: `groupPolicy`, `channels.*`, `channels.*.users`, `channels.*.requireMention`, `implicitMentions.*`
- थ्रेडिंग/इतिहास: `replyToMode`, `replyToModeByChatType`, `thread.*`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- उपस्थिति से सक्रियण: `presenceEvents.mode`, `channels.*.presenceEvents.mode` (`off|auto|on`; डिफ़ॉल्ट `off`)
- डिलीवरी: `textChunkLimit`, `streaming.chunkMode`, `mediaMaxMb`, `streaming`, `streaming.nativeTransport`, `streaming.preview.toolProgress`
- अनफ़र्ल: `unfurlLinks` (डिफ़ॉल्ट: `false`), `chat.postMessage` लिंक/मीडिया पूर्वावलोकन नियंत्रण के लिए `unfurlMedia`; लिंक पूर्वावलोकन फिर से सक्षम करने के लिए `unfurlLinks: true` सेट करें
- संचालन/सुविधाएँ: `configWrites`, `commands.native`, `slashCommand.*`, `actions.*`, `userToken`, `userTokenReadOnly`

</Accordion>

## समस्या निवारण

<AccordionGroup>
  <Accordion title="चैनलों में कोई उत्तर नहीं">
    इस क्रम में जाँचें:

    - `groupPolicy`
    - चैनल अनुमति-सूची (`channels.slack.channels`) — **कुंजियाँ चैनल ID होनी चाहिए** (`C12345678`), नाम नहीं (`#channel-name`)। `groupPolicy: "allowlist"` के अंतर्गत नाम-आधारित कुंजियाँ बिना कोई त्रुटि दिखाए विफल हो जाती हैं, क्योंकि डिफ़ॉल्ट रूप से चैनल रूटिंग में ID को प्राथमिकता मिलती है। ID खोजने के लिए: Slack में चैनल पर राइट-क्लिक करें → **Copy link** — URL के अंत में मौजूद `C...` मान चैनल ID है।
    - `requireMention`
    - प्रति-चैनल `users` अनुमति-सूची
    - `messages.groupChat.visibleReplies`: सामान्य समूह/चैनल अनुरोधों में डिफ़ॉल्ट `"automatic"` होता है। यदि आपने `"message_tool"` चुना है और लॉग में `message(action=send)` कॉल के बिना सहायक का टेक्स्ट दिखाई देता है, तो मॉडल दृश्य संदेश-टूल पथ का उपयोग नहीं कर पाया। इस मोड में अंतिम टेक्स्ट निजी रहता है; दबाए गए पेलोड मेटाडेटा के लिए Gateway का विस्तृत लॉग देखें, या यदि आप चाहते हैं कि सहायक का प्रत्येक सामान्य अंतिम उत्तर पुराने पथ से पोस्ट हो, तो इसे `"automatic"` पर सेट करें।
    - `messages.groupChat.unmentionedInbound`: यदि यह `"room_event"` है, तो अनुमति प्राप्त चैनल की बिना उल्लेख वाली बातचीत परिवेशी संदर्भ होती है और तब तक मौन रहती है, जब तक एजेंट `message` टूल को कॉल न करे। [परिवेशी कक्ष इवेंट](/hi/channels/ambient-room-events) देखें।

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "automatic",
    },
  },
}
```

    उपयोगी कमांड:

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="DM संदेशों की अनदेखी की जा रही है">
    जाँचें:

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy` (या पुराना `channels.slack.dm.policy`)
    - पेयरिंग अनुमोदन / अनुमति-सूची प्रविष्टियाँ (`dmPolicy: "open"` के लिए अब भी `channels.slack.allowFrom: ["*"]` आवश्यक है)
    - समूह DM में MPIM प्रबंधन का उपयोग होता है; `channels.slack.dm.groupEnabled` सक्षम करें और, यदि कॉन्फ़िगर किया गया हो, तो MPIM को `channels.slack.dm.groupChannels` में शामिल करें
    - Slack Assistant DM इवेंट: `drop message_changed` का उल्लेख करने वाले विस्तृत लॉग का आमतौर पर अर्थ है कि Slack ने संदेश मेटाडेटा में पुनर्प्राप्त किए जा सकने वाले मानव प्रेषक के बिना संपादित Assistant-थ्रेड इवेंट भेजा है

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="Socket मोड कनेक्ट नहीं हो रहा">
    Slack ऐप सेटिंग में बॉट और ऐप टोकन तथा Socket Mode के सक्षम होने की पुष्टि करें।
    App-Level Token को `connections:write` चाहिए और Bot User OAuth Token
    बॉट टोकन उसी Slack ऐप/वर्कस्पेस का होना चाहिए जिसका ऐप टोकन है।

    यदि `openclaw channels status --probe --json` में `botTokenStatus` या
    `appTokenStatus: "configured_unavailable"` दिखाई देता है, तो Slack खाता
    कॉन्फ़िगर है, लेकिन मौजूदा रनटाइम SecretRef-समर्थित मान को हल नहीं कर सका।

    `slack socket mode failed to start; retry ...` जैसे लॉग पुनर्प्राप्त किए जा सकने वाले
    स्टार्ट विफलता हैं। इसके बजाय अनुपलब्ध स्कोप, निरस्त टोकन और अमान्य प्रमाणीकरण तुरंत विफल होते हैं।
    `slack token mismatch ...` लॉग का अर्थ है कि बॉट टोकन और ऐप टोकन
    अलग-अलग Slack ऐप के लगते हैं; Slack ऐप क्रेडेंशियल ठीक करें।

  </Accordion>

  <Accordion title="HTTP मोड को इवेंट प्राप्त नहीं हो रहे">
    पुष्टि करें:

    - साइनिंग सीक्रेट
    - Webhook पथ
    - Slack Request URL (Events + Interactivity + Slash Commands)
    - प्रत्येक HTTP खाते के लिए अद्वितीय `webhookPath`
    - सार्वजनिक URL TLS समाप्त करता है और अनुरोधों को Gateway पथ पर फ़ॉरवर्ड करता है
    - Slack ऐप का `request_url` पथ `channels.slack.webhookPath` (डिफ़ॉल्ट `/slack/events`) से बिल्कुल मेल खाता है

    यदि खाता स्नैपशॉट में `signingSecretStatus: "configured_unavailable"` दिखाई देता है,
    तो HTTP खाता कॉन्फ़िगर है, लेकिन मौजूदा रनटाइम SecretRef-समर्थित
    साइनिंग सीक्रेट को हल नहीं कर सका।

    बार-बार आने वाले `slack: webhook path ... already registered` लॉग का अर्थ है कि दो HTTP
    खाते समान `webhookPath` का उपयोग कर रहे हैं; प्रत्येक खाते को अलग पथ दें।

  </Accordion>

  <Accordion title="नेटिव/स्लैश कमांड सक्रिय नहीं हो रहे">
    पुष्टि करें कि आपका आशय इनमें से किससे था:

    - Slack में पंजीकृत मेल खाते स्लैश कमांड के साथ नेटिव कमांड मोड (`channels.slack.commands.native: true`)
    - या एकल स्लैश कमांड मोड (`channels.slack.slashCommand.enabled: true`)

    Slack स्लैश कमांड को स्वचालित रूप से बनाता या हटाता नहीं है। `commands.native: "auto"` Slack नेटिव कमांड सक्षम नहीं करता; `true` का उपयोग करें और Slack ऐप में मेल खाते कमांड बनाएँ। HTTP मोड में, प्रत्येक Slack स्लैश कमांड में Gateway URL शामिल होना चाहिए। Socket Mode में, कमांड पेलोड वेबसॉकेट पर आते हैं और Slack `slash_commands[].url` की अनदेखी करता है।

    `commands.useAccessGroups`, DM प्राधिकरण, चैनल अनुमति-सूचियाँ,
    और प्रति-चैनल `users` अनुमति-सूचियाँ भी जाँचें। Slack अवरुद्ध
    स्लैश-कमांड प्रेषकों के लिए अस्थायी त्रुटियाँ लौटाता है, जिनमें शामिल हैं:

    - `This channel is not allowed.`
    - `You are not authorized to use this command here.`

  </Accordion>
</AccordionGroup>

## अटैचमेंट मीडिया संदर्भ

Slack फ़ाइल डाउनलोड सफल होने और आकार सीमा की अनुमति होने पर डाउनलोड किए गए मीडिया को एजेंट टर्न से जोड़ सकता है। ऑडियो क्लिप का लिप्यंतरण किया जा सकता है, इमेज फ़ाइलें मीडिया-बोध पथ से या सीधे दृष्टि-सक्षम उत्तर मॉडल तक पहुँच सकती हैं, और अन्य फ़ाइलें डाउनलोड योग्य फ़ाइल संदर्भ के रूप में उपलब्ध रहती हैं।

### समर्थित मीडिया प्रकार

| मीडिया प्रकार                     | स्रोत               | मौजूदा व्यवहार                                                                  | टिप्पणियाँ                                                                     |
| ------------------------------ | -------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Slack ऑडियो क्लिप              | Slack फ़ाइल URL       | डाउनलोड करके साझा ऑडियो लिप्यंतरण के माध्यम से रूट किया जाता है                          | इसके लिए `files:read` और कार्यरत `tools.media.audio` मॉडल या CLI आवश्यक है      |
| JPEG / PNG / GIF / WebP इमेज | Slack फ़ाइल URL       | डाउनलोड करके दृष्टि-सक्षम प्रबंधन के लिए टर्न से जोड़ी जाती हैं                   | प्रति-फ़ाइल सीमा: `channels.slack.mediaMaxMb` (डिफ़ॉल्ट 20 MB)                 |
| PDF फ़ाइलें                      | Slack फ़ाइल URL       | डाउनलोड करके `download-file` या `pdf` जैसे टूल के लिए फ़ाइल संदर्भ के रूप में प्रस्तुत की जाती हैं | Slack इनबाउंड PDF को स्वचालित रूप से इमेज-दृष्टि इनपुट में परिवर्तित नहीं करता |
| अन्य फ़ाइलें                    | Slack फ़ाइल URL       | संभव होने पर डाउनलोड करके फ़ाइल संदर्भ के रूप में प्रस्तुत की जाती हैं                              | बाइनरी फ़ाइलों को इमेज इनपुट नहीं माना जाता                               |
| थ्रेड उत्तर                 | थ्रेड आरंभकर्ता फ़ाइलें | जब उत्तर में कोई प्रत्यक्ष मीडिया न हो, तब रूट-संदेश फ़ाइलों को संदर्भ के रूप में भरा जा सकता है  | केवल फ़ाइल वाले आरंभकर्ता अटैचमेंट प्लेसहोल्डर का उपयोग करते हैं                          |
| बहु-फ़ाइल संदेश            | एकाधिक Slack फ़ाइलें | प्रत्येक फ़ाइल का स्वतंत्र रूप से मूल्यांकन किया जाता है                                              | Slack प्रसंस्करण प्रति संदेश आठ फ़ाइलों तक सीमित है                     |

### इनबाउंड पाइपलाइन

फ़ाइल अटैचमेंट वाला Slack संदेश आने पर:

1. OpenClaw बॉट टोकन का उपयोग करके Slack के निजी URL से फ़ाइल डाउनलोड करता है।
2. सफल होने पर फ़ाइल मीडिया स्टोर में लिखी जाती है।
3. डाउनलोड किए गए मीडिया पथ और सामग्री प्रकार इनबाउंड संदर्भ में जोड़े जाते हैं।
4. ऑडियो क्लिप को साझा लिप्यंतरण पाइपलाइन में रूट किया जाता है; इमेज-सक्षम मॉडल/टूल पथ उसी संदर्भ से इमेज अटैचमेंट का उपयोग कर सकते हैं।
5. अन्य फ़ाइलें उन्हें संभाल सकने वाले टूल के लिए फ़ाइल मेटाडेटा या मीडिया संदर्भ के रूप में उपलब्ध रहती हैं।

### थ्रेड-रूट अटैचमेंट इनहेरिटेंस

जब कोई संदेश किसी थ्रेड में आता है (उसका `thread_ts` पैरेंट होता है):

- यदि उत्तर में कोई प्रत्यक्ष मीडिया नहीं है और शामिल रूट संदेश में फ़ाइलें हैं, तो Slack रूट फ़ाइलों को थ्रेड-आरंभकर्ता संदर्भ के रूप में भर सकता है।
- रूट फ़ाइलें केवल नया या रीसेट किया गया थ्रेड सत्र आरंभ करते समय भरी जाती हैं। बाद के केवल-टेक्स्ट उत्तर मौजूदा सत्र संदर्भ का पुनः उपयोग करते हैं और रूट फ़ाइलों को नए मीडिया के रूप में दोबारा नहीं जोड़ते।
- प्रत्यक्ष उत्तर अटैचमेंट को रूट-संदेश अटैचमेंट पर प्राथमिकता मिलती है।
- केवल फ़ाइलों और बिना टेक्स्ट वाले रूट संदेश को अटैचमेंट प्लेसहोल्डर से दर्शाया जाता है, ताकि फ़ॉलबैक में फिर भी उसकी फ़ाइलें शामिल हो सकें।

### बहु-अटैचमेंट प्रबंधन

जब किसी एक Slack संदेश में कई फ़ाइल अटैचमेंट हों:

- प्रत्येक अटैचमेंट को मीडिया पाइपलाइन के माध्यम से स्वतंत्र रूप से संसाधित किया जाता है।
- डाउनलोड किए गए मीडिया संदर्भों को संदेश संदर्भ में एकत्रित किया जाता है।
- प्रसंस्करण क्रम इवेंट पेलोड में Slack के फ़ाइल क्रम का अनुसरण करता है।
- एक अटैचमेंट का डाउनलोड विफल होने पर अन्य अटैचमेंट अवरुद्ध नहीं होते।

### आकार, डाउनलोड और मॉडल सीमाएँ

- **आकार सीमा**: डिफ़ॉल्ट रूप से प्रति फ़ाइल 20 MB। `channels.slack.mediaMaxMb` से कॉन्फ़िगर किया जा सकता है।
- **ऑडियो लिप्यंतरण सीमा**: डाउनलोड की गई फ़ाइल किसी लिप्यंतरण प्रदाता या CLI को भेजे जाने पर चयनित ऑडियो-सक्षम `tools.media.models[]` प्रविष्टि का `maxBytes` भी लागू होता है।
- **डाउनलोड विफलताएँ**: ऐसी फ़ाइलें जिन्हें Slack प्रस्तुत नहीं कर सकता, समय-सीमा समाप्त URL, अप्राप्य फ़ाइलें, सीमा से बड़ी फ़ाइलें और Slack प्रमाणीकरण/लॉगिन HTML प्रतिक्रियाएँ असमर्थित प्रारूप के रूप में रिपोर्ट किए जाने के बजाय छोड़ दी जाती हैं।
- **दृष्टि मॉडल**: इमेज विश्लेषण सक्रिय उत्तर मॉडल का उपयोग करता है, यदि वह दृष्टि का समर्थन करता है, या `agents.defaults.imageModel` पर कॉन्फ़िगर किए गए इमेज मॉडल का उपयोग करता है।

### ज्ञात सीमाएँ

| परिदृश्य                                      | वर्तमान व्यवहार                                                                   | समाधान                                                                    |
| --------------------------------------------- | ---------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| समय-सीमा समाप्त Slack फ़ाइल URL                        | फ़ाइल छोड़ दी जाती है; कोई त्रुटि नहीं दिखाई जाती                                                       | फ़ाइल को Slack में फिर से अपलोड करें                                                   |
| ऑडियो ट्रांसक्रिप्शन अनुपलब्ध               | क्लिप संलग्न रहती है, लेकिन कोई ट्रांसक्रिप्ट तैयार नहीं होता                                | `tools.media.audio` कॉन्फ़िगर करें या समर्थित स्थानीय ट्रांसक्रिप्शन CLI इंस्टॉल करें  |
| कैप्शन-रहित क्लिप उल्लेख गेट से पार नहीं होती | निजी अनुमानात्मक ट्रांसक्रिप्शन के बाद हटा दी जाती है; ट्रांसक्रिप्ट और डाउनलोड त्याग दिए जाते हैं | बोले गए नाम के उल्लेख का पैटर्न कॉन्फ़िगर करें, टाइप किया हुआ बॉट उल्लेख जोड़ें, या DM का उपयोग करें |
| विज़न मॉडल कॉन्फ़िगर नहीं है                   | छवि अटैचमेंट मीडिया संदर्भों के रूप में संग्रहीत होते हैं, लेकिन छवियों के रूप में उनका विश्लेषण नहीं होता       | `agents.defaults.imageModel` कॉन्फ़िगर करें या विज़न-सक्षम उत्तर मॉडल का उपयोग करें    |
| बहुत बड़ी छवियाँ (डिफ़ॉल्ट रूप से > 20 MB)        | आकार सीमा के अनुसार छोड़ दी जाती हैं                                                               | यदि Slack अनुमति देता है, तो `channels.slack.mediaMaxMb` बढ़ाएँ                          |
| फ़ॉरवर्ड किए गए/साझा अटैचमेंट                  | टेक्स्ट और Slack पर होस्ट की गई छवि/फ़ाइल मीडिया का सर्वोत्तम-प्रयास से प्रबंधन होता है                             | सीधे OpenClaw थ्रेड में फिर से साझा करें                                      |
| PDF अटैचमेंट                               | फ़ाइल/मीडिया संदर्भ के रूप में संग्रहीत होते हैं, छवि विज़न से स्वचालित रूप से रूट नहीं किए जाते        | फ़ाइल मेटाडेटा के लिए `download-file` या PDF विश्लेषण के लिए `pdf` टूल का उपयोग करें      |

### संबंधित दस्तावेज़

- [मीडिया समझ पाइपलाइन](/hi/nodes/media-understanding)
- [ऑडियो और वॉइस नोट्स](/hi/nodes/audio)
- [PDF टूल](/hi/tools/pdf)

## संबंधित

<CardGroup cols={2}>
  <Card title="पेयरिंग" icon="link" href="/hi/channels/pairing">
    किसी Slack उपयोगकर्ता को Gateway से पेयर करें।
  </Card>
  <Card title="समूह" icon="users" href="/hi/channels/groups">
    चैनल और समूह DM का व्यवहार।
  </Card>
  <Card title="चैनल रूटिंग" icon="route" href="/hi/channels/channel-routing">
    आने वाले संदेशों को एजेंटों तक रूट करें।
  </Card>
  <Card title="सुरक्षा" icon="shield" href="/hi/gateway/security">
    खतरा मॉडल और सुदृढ़ीकरण।
  </Card>
  <Card title="कॉन्फ़िगरेशन" icon="sliders" href="/hi/gateway/configuration">
    कॉन्फ़िगरेशन संरचना और प्राथमिकता क्रम।
  </Card>
  <Card title="स्लैश कमांड" icon="terminal" href="/hi/tools/slash-commands">
    कमांड सूची और व्यवहार।
  </Card>
</CardGroup>
