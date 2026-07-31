---
read_when:
    - OpenClaw के लिए Twitch चैट एकीकरण सेट अप करना
sidebarTitle: Twitch
summary: 'Twitch चैट बॉट: इंस्टॉलेशन, क्रेडेंशियल, अभिगम नियंत्रण, टोकन रीफ़्रेश'
title: Twitch
x-i18n:
    generated_at: "2026-07-27T17:23:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d827c742ded5fd0b071443dead27b975e2414419b0facb486d7f9c0c9800b060
    source_path: channels/twitch.md
    workflow: 16
---

Twurple क्लाइंट के माध्यम से Twitch के चैट (IRC) इंटरफ़ेस पर Twitch चैट समर्थन। OpenClaw एक Twitch बॉट खाते के रूप में साइन इन करता है, प्रत्येक कॉन्फ़िगर किए गए खाते के लिए एक चैनल से जुड़ता है और उसी चैनल में उत्तर देता है।

## इंस्टॉल करें

Twitch एक आधिकारिक Plugin के रूप में उपलब्ध है; यह मुख्य इंस्टॉलेशन का हिस्सा नहीं है।

<Tabs>
  <Tab title="npm रजिस्ट्री">
    ```bash
    openclaw plugins install @openclaw/twitch
    ```
  </Tab>
  <Tab title="स्थानीय चेकआउट">
    ```bash
    openclaw plugins install ./path/to/local/twitch-plugin
    ```
  </Tab>
</Tabs>

`plugins install` Plugin को पंजीकृत और सक्षम करता है। `openclaw onboard` या `openclaw channels add` के दौरान Twitch चुनने पर इसे आवश्यकता के अनुसार इंस्टॉल किया जाता है। वर्तमान रिलीज़ का अनुसरण करने के लिए केवल पैकेज नाम का उपयोग करें; पुनरुत्पाद्य इंस्टॉलेशन के लिए ही कोई सटीक संस्करण पिन करें। OpenClaw 2026.4.10 या उसके बाद का संस्करण आवश्यक है।

विवरण: [Plugins](/hi/tools/plugin)

## त्वरित सेटअप

<Steps>
  <Step title="Plugin इंस्टॉल करें">
    ऊपर [इंस्टॉल करें](#install) देखें।
  </Step>
  <Step title="Twitch बॉट खाता बनाएँ">
    बॉट के लिए एक समर्पित Twitch खाता बनाएँ (या किसी मौजूदा खाते का उपयोग करें)।
  </Step>
  <Step title="क्रेडेंशियल जनरेट करें">
    [Twitch Token Generator](https://twitchtokengenerator.com/) का उपयोग करें:

    - **Bot Token** चुनें
    - सत्यापित करें कि स्कोप `chat:read` और `chat:write` चुने गए हैं
    - **Client ID** और **Access Token** कॉपी करें

  </Step>
  <Step title="अपनी Twitch उपयोगकर्ता ID खोजें">
    किसी उपयोगकर्ता नाम को Twitch उपयोगकर्ता ID में बदलने के लिए [https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) का उपयोग करें।
  </Step>
  <Step title="टोकन कॉन्फ़िगर करें">
    - परिवेश: `OPENCLAW_TWITCH_ACCESS_TOKEN=...` (केवल डिफ़ॉल्ट खाता)
    - या कॉन्फ़िगरेशन: `channels.twitch.accessToken`

    यदि दोनों सेट हैं, तो कॉन्फ़िगरेशन को प्राथमिकता मिलती है (परिवेश चर केवल डिफ़ॉल्ट खाते के लिए फ़ॉलबैक है)।

  </Step>
  <Step title="Gateway शुरू करें">
    ```bash
    openclaw gateway run
    ```
  </Step>
</Steps>

<Warning>
अनधिकृत उपयोगकर्ताओं को बॉट ट्रिगर करने से रोकने के लिए अभिगम नियंत्रण (`allowFrom` या `allowedRoles`) जोड़ें। `requireMention` का डिफ़ॉल्ट मान `true` है।
</Warning>

न्यूनतम कॉन्फ़िगरेशन:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw", // बॉट का Twitch खाता (प्रमाणीकरण करता है)
      accessToken: "oauth:abc123...", // OAuth अभिगम टोकन (या OPENCLAW_TWITCH_ACCESS_TOKEN परिवेश चर का उपयोग करें)
      clientId: "xyz789...", // Token Generator से Client ID
      channel: "yourchannel", // किस Twitch चैनल की चैट से जुड़ना है (आवश्यक)
      allowFrom: ["123456789"], // (अनुशंसित) केवल आपकी Twitch उपयोगकर्ता ID
    },
  },
}
```

## यह क्या है

- Gateway के स्वामित्व वाला एक Twitch चैनल।
- नियतात्मक रूटिंग: उत्तर हमेशा उसी Twitch चैनल पर वापस जाते हैं जहाँ से संदेश आया था।
- जुड़ा हुआ प्रत्येक चैनल एक पृथक समूह सत्र कुंजी `agent:<agentId>:twitch:group:<channel>` से मैप होता है।
- `username` बॉट का खाता है (जो प्रमाणीकरण करता है), `channel` वह चैट रूम है जिससे जुड़ना है। प्रत्येक खाता प्रविष्टि ठीक एक चैनल से जुड़ती है।
- टोकन `oauth:` उपसर्ग के साथ या उसके बिना काम करते हैं; OpenClaw दोनों रूपों को सामान्यीकृत करता है (सेटअप विज़ार्ड `oauth:` रूप की अपेक्षा करता है)।

## इनबाउंड टिकाऊपन

OpenClaw सामान्य प्रेषण से पहले स्वीकार किए गए प्रत्येक Twitch चैट संदेश को टिकाऊ रूप से कतारबद्ध करता है। लंबित या पुनः प्रयास योग्य संदेश Gateway के पुनः आरंभ होने के बाद भी बने रहते हैं, कॉन्फ़िगर किए गए चैनल के लिए क्रमबद्ध रहते हैं और सक्रिय या संरक्षित पूर्णता रिकॉर्ड मौजूद रहने तक डुप्लिकेट कतार प्रविष्टियों को रोकने के लिए Twitch की संदेश ID का उपयोग करते हैं।

क्लाइंट द्वारा स्वीकार किए जाने के बाद Twitch चैट किसी `PRIVMSG` को दोबारा नहीं भेजती। यह स्थानीय स्वीकार-से-प्रेषण क्रैश अवधि से सुरक्षा देता है, लेकिन टिकाऊ स्वीकृति से पहले छूटे संदेशों को पुनर्प्राप्त नहीं कर सकता। यदि कतार में जोड़ना ही विफल हो जाए, तो OpenClaw विफलता को लॉग करता है; दोबारा कनेक्ट होने पर Twitch से उस संदेश को पुनः भेजने का अनुरोध नहीं किया जाता।

## टोकन रीफ़्रेश (वैकल्पिक)

[Twitch Token Generator](https://twitchtokengenerator.com/) से प्राप्त टोकन को OpenClaw रीफ़्रेश नहीं कर सकता—समाप्त होने पर उन्हें फिर से जनरेट करें (वे कुछ घंटों तक चलते हैं; ऐप पंजीकरण आवश्यक नहीं है)।

स्वचालित रीफ़्रेश के लिए [Twitch Developer Console](https://dev.twitch.tv/console) पर अपना ऐप बनाएँ और यह जोड़ें:

```json5
{
  channels: {
    twitch: {
      clientSecret: "your_client_secret",
      refreshToken: "your_refresh_token",
    },
  },
}
```

दोनों सेट होने पर Plugin ऐसे रीफ़्रेशिंग प्रमाणीकरण प्रदाता का उपयोग करता है जो टोकन की समय-सीमा समाप्त होने से पहले उन्हें नवीनीकृत करता है और प्रत्येक रीफ़्रेश को लॉग करता है। `refreshToken` के बिना यह `token refresh disabled (no refresh token)` लॉग करता है; `clientSecret` के बिना यह स्थिर (रीफ़्रेश न होने वाले) टोकन पर फ़ॉलबैक करता है।

## बहु-खाता समर्थन

प्रत्येक खाते के अलग क्रेडेंशियल के साथ `channels.twitch.accounts` का उपयोग करें। साझा पैटर्न के लिए [कॉन्फ़िगरेशन](/hi/gateway/configuration) देखें।

उदाहरण (दो चैनलों में एक बॉट खाता):

```json5
{
  channels: {
    twitch: {
      accounts: {
        channel1: {
          username: "openclaw",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "yourchannel",
        },
        channel2: {
          username: "openclaw",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "secondchannel",
        },
      },
    },
  },
}
```

<Note>
प्रत्येक खाता प्रविष्टि को अपना `accessToken` चाहिए (परिवेश चर केवल डिफ़ॉल्ट खाते को कवर करता है)। एक खाता ठीक एक चैनल से जुड़ता है, इसलिए दो चैनलों से जुड़ने का अर्थ है दो खाते। `channels.twitch.defaultAccount` यह चुनता है कि कौन-सा खाता डिफ़ॉल्ट है।
</Note>

## अभिगम नियंत्रण

`allowFrom` Twitch उपयोगकर्ता ID की कठोर अनुमति-सूची है। इसे सेट करने पर `allowedRoles` को अनदेखा किया जाता है; इसके बजाय भूमिका-आधारित अभिगम का उपयोग करने के लिए `allowFrom` को सेट न करें।

**उपलब्ध भूमिकाएँ:** `"moderator"`, `"owner"`, `"vip"`, `"subscriber"`, `"all"`।

<Tabs>
  <Tab title="उपयोगकर्ता ID अनुमति-सूची (सबसे सुरक्षित)">
    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowFrom: ["123456789", "987654321"],
            },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="भूमिका-आधारित">
    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowedRoles: ["moderator", "vip"],
            },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="@mention आवश्यकता अक्षम करें">
    डिफ़ॉल्ट रूप से, `requireMention` का मान `true` है। सभी अनुमत संदेशों का उत्तर देने के लिए:

    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              requireMention: false,
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

<Note>
**उपयोगकर्ता ID क्यों?** उपयोगकर्ता नाम बदल सकते हैं, जिससे प्रतिरूपण संभव होता है। उपयोगकर्ता ID स्थायी होती हैं।

अपनी ID [उपयोगकर्ता नाम से ID कन्वर्टर](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) से खोजें।
</Note>

## समस्या निवारण

पहले नैदानिक कमांड चलाएँ:

```bash
openclaw doctor
openclaw channels status --probe
```

<AccordionGroup>
  <Accordion title="बॉट संदेशों का उत्तर नहीं देता">
    - **अभिगम नियंत्रण जाँचें:** सुनिश्चित करें कि आपकी उपयोगकर्ता ID `allowFrom` में है, या परीक्षण के लिए अस्थायी रूप से `allowFrom` हटाकर `allowedRoles: ["all"]` सेट करें।
    - **उल्लेख गेट जाँचें:** `requireMention: true` (डिफ़ॉल्ट) के साथ संदेशों में बॉट उपयोगकर्ता नाम को @mention करना आवश्यक है।
    - **जाँचें कि बॉट चैनल में है:** बॉट केवल `channel` में नामित चैनल से जुड़ता है।

  </Accordion>
  <Accordion title="टोकन संबंधी समस्याएँ">
    "कनेक्ट करने में विफल" या प्रमाणीकरण त्रुटियाँ:

    - सत्यापित करें कि `accessToken` OAuth अभिगम टोकन का मान है (`oauth:` उपसर्ग वैकल्पिक है)
    - जाँचें कि टोकन में `chat:read` और `chat:write` स्कोप हैं
    - यदि टोकन रीफ़्रेश का उपयोग कर रहे हैं, तो सत्यापित करें कि `clientSecret` और `refreshToken` सेट हैं

  </Accordion>
  <Accordion title="टोकन रीफ़्रेश काम नहीं कर रहा">
    रीफ़्रेश घटनाओं के लिए लॉग जाँचें:

    ```text
    mybot के लिए परिवेश टोकन स्रोत का उपयोग किया जा रहा है
    उपयोगकर्ता 123456 के लिए अभिगम टोकन रीफ़्रेश किया गया (14400s में समाप्त होगा)
    ```

    यदि आपको `token refresh disabled (no refresh token)` दिखाई दे:

    - सुनिश्चित करें कि `clientSecret` दिया गया है
    - सुनिश्चित करें कि `refreshToken` दिया गया है

  </Accordion>
</AccordionGroup>

## कॉन्फ़िगरेशन

### खाता कॉन्फ़िगरेशन

<ParamField path="username" type="string" required>
  बॉट उपयोगकर्ता नाम (प्रमाणीकरण करने वाला खाता)।
</ParamField>
<ParamField path="accessToken" type="string" required>
  `chat:read` और `chat:write` वाला OAuth अभिगम टोकन (डिफ़ॉल्ट खाते के लिए कॉन्फ़िगरेशन या परिवेश)।
</ParamField>
<ParamField path="clientId" type="string" required>
  Twitch Client ID (Token Generator या आपके ऐप से)। स्कीमा में वैकल्पिक, लेकिन कनेक्ट करने के लिए आवश्यक।
</ParamField>
<ParamField path="channel" type="string" required>
  जुड़ने वाला चैनल।
</ParamField>
<ParamField path="enabled" type="boolean" default="true">
  यह खाता सक्षम करें।
</ParamField>
<ParamField path="clientSecret" type="string">
  वैकल्पिक: स्वचालित टोकन रीफ़्रेश के लिए।
</ParamField>
<ParamField path="refreshToken" type="string">
  वैकल्पिक: स्वचालित टोकन रीफ़्रेश के लिए।
</ParamField>
<ParamField path="expiresIn" type="number">
  टोकन समाप्ति अवधि सेकंड में (रीफ़्रेश ट्रैकिंग)।
</ParamField>
<ParamField path="obtainmentTimestamp" type="number">
  टोकन प्राप्त होने का टाइमस्टैम्प (रीफ़्रेश ट्रैकिंग)।
</ParamField>
<ParamField path="allowFrom" type="string[]">
  उपयोगकर्ता ID अनुमति-सूची। सेट होने पर भूमिकाओं को अनदेखा किया जाता है।
</ParamField>
<ParamField path="allowedRoles" type='Array<"moderator" | "owner" | "vip" | "subscriber" | "all">'>
  भूमिका-आधारित अभिगम नियंत्रण।
</ParamField>
<ParamField path="requireMention" type="boolean" default="true">
  बॉट को ट्रिगर करने के लिए @mention आवश्यक करें।
</ParamField>
<ParamField path="responsePrefix" type="string">
  इस खाते के लिए आउटबाउंड उत्तर उपसर्ग का ओवरराइड।
</ParamField>

### प्रदाता विकल्प

- `channels.twitch.enabled` - चैनल स्टार्टअप सक्षम/अक्षम करें
- `channels.twitch.username` / `accessToken` / `clientId` / `channel` - सरलीकृत एकल-खाता कॉन्फ़िगरेशन (अंतर्निहित `default` खाता; `accounts.default` पर प्राथमिकता लेता है)
- `channels.twitch.accounts.<accountName>` - बहु-खाता कॉन्फ़िगरेशन (ऊपर दिए सभी खाता फ़ील्ड)
- `channels.twitch.defaultAccount` - कौन-सा खाता नाम डिफ़ॉल्ट है
- `channels.twitch.markdown.tables` - Markdown तालिका रेंडरिंग मोड (`off` | `bullets` | `code` | `block`)

पूर्ण उदाहरण:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw",
      accessToken: "oauth:abc123...",
      clientId: "xyz789...",
      channel: "yourchannel",
      clientSecret: "secret123...",
      refreshToken: "refresh456...",
      allowFrom: ["123456789"],
      accounts: {
        second: {
          username: "mybot",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "your_channel",
          enabled: true,
          expiresIn: 14400,
          obtainmentTimestamp: 1706092800000,
          allowedRoles: ["moderator"],
        },
      },
    },
  },
}
```

## टूल क्रियाएँ

एजेंट संदेश टूल की `send` क्रिया के माध्यम से Twitch संदेश भेज सकता है:

```json5
{
  channel: "twitch",
  action: "send",
  to: "#mychannel",
  message: "नमस्ते Twitch!",
}
```

`to` वैकल्पिक है और इसका डिफ़ॉल्ट मान खाते का कॉन्फ़िगर किया गया `channel` है।

## सुरक्षा और संचालन

- **टोकन को पासवर्ड की तरह मानें** - टोकन को कभी भी git में कमिट न करें।
- लंबे समय तक चलने वाले बॉट के लिए **स्वचालित टोकन रिफ़्रेश का उपयोग करें**।
- एक्सेस नियंत्रण के लिए उपयोगकर्ता नामों के बजाय **उपयोगकर्ता ID की अनुमतिसूचियों का उपयोग करें**।
- टोकन रिफ़्रेश घटनाओं और कनेक्शन स्थिति के लिए **लॉग की निगरानी करें**।
- **टोकन का दायरा न्यूनतम रखें** - केवल `chat:read` और `chat:write` का अनुरोध करें।
- **यदि अटक जाएँ**: यह पुष्टि करने के बाद Gateway को पुनः आरंभ करें कि सत्र का स्वामी कोई अन्य प्रोसेस नहीं है।

## सीमाएँ

- प्रति संदेश **500 वर्ण**; लंबे उत्तरों को शब्द सीमाओं पर खंडों में बाँटा जाता है।
- भेजने से पहले Markdown हटा दिया जाता है (Twitch चैट सादा टेक्स्ट है; नई पंक्तियाँ रिक्त स्थान में बदल जाती हैं)।
- OpenClaw स्वयं कोई दर सीमा लागू नहीं करता; Twurple चैट क्लाइंट Twitch की दर सीमाओं को संभालता है।

## संबंधित

- [चैनल रूटिंग](/hi/channels/channel-routing) — संदेशों के लिए सत्र रूटिंग
- [चैनलों का अवलोकन](/hi/channels) — सभी समर्थित चैनल
- [समूह](/hi/channels/groups) — समूह चैट का व्यवहार और उल्लेख नियंत्रण
- [पेयरिंग](/hi/channels/pairing) — DM प्रमाणीकरण और पेयरिंग प्रवाह
- [सुरक्षा](/hi/gateway/security) — एक्सेस मॉडल और सुरक्षा सुदृढ़ीकरण
