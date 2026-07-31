---
read_when:
    - Signal सहायता सेट अप करना
    - Signal पर संदेश भेजने/प्राप्त करने की डीबगिंग
summary: signal-cli (नेटिव डेमन या bbernhard कंटेनर) के माध्यम से Signal समर्थन, सेटअप पथ और नंबर मॉडल
title: Signal
x-i18n:
    generated_at: "2026-07-27T17:22:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 744f817e425d378e9f3e160df534019a6fc865227eb3fc68959a12ad46c0b714
    source_path: channels/signal.md
    workflow: 16
---

Signal एक डाउनलोड करने योग्य चैनल plugin है (`@openclaw/signal`)। Gateway, `signal-cli` से HTTP पर संचार करता है: या तो नेटिव डेमन (JSON-RPC + SSE) या [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) कंटेनर (REST + WebSocket)। OpenClaw में libsignal अंतर्निहित नहीं है।

## नंबर मॉडल (इसे पहले पढ़ें)

- Gateway एक **Signal डिवाइस** से कनेक्ट होता है: `signal-cli` खाता।
- बॉट को **आपके व्यक्तिगत Signal खाते** पर चलाने से वह आपके अपने संदेशों को अनदेखा करता है (लूप सुरक्षा)।
- “मैं बॉट को संदेश भेजूँ और वह उत्तर दे,” इसके लिए एक **अलग बॉट नंबर** का उपयोग करें।

## इंस्टॉल करें

```bash
openclaw plugins install @openclaw/signal
```

बिना उपसर्ग वाले plugin विनिर्देश पहले ClawHub आज़माते हैं, फिर वैकल्पिक रूप से npm का उपयोग करते हैं। `openclaw plugins install clawhub:@openclaw/signal` या `npm:@openclaw/signal` से किसी स्रोत को बाध्य करें। `plugins install` plugin को पंजीकृत और सक्षम करता है; अलग `enable` चरण की आवश्यकता नहीं है। इंस्टॉल के सामान्य नियमों के लिए [Plugins](/hi/tools/plugin) देखें।

## त्वरित सेटअप

<Steps>
  <Step title="एक नंबर चुनें">
    बॉट के लिए एक **अलग Signal नंबर** का उपयोग करें (अनुशंसित)।
  </Step>
  <Step title="plugin इंस्टॉल करें">
    ```bash
    openclaw plugins install @openclaw/signal
    ```
  </Step>
  <Step title="निर्देशित सेटअप चलाएँ">
    ```bash
    openclaw channels add
    ```
    विज़ार्ड पता लगाता है कि `signal-cli`, `PATH` पर उपलब्ध है या नहीं और उपलब्ध न होने पर उसे इंस्टॉल करने की पेशकश करता है: Linux x86-64 पर आधिकारिक नेटिव GraalVM बिल्ड डाउनलोड करता है या macOS और अन्य आर्किटेक्चर पर Homebrew के माध्यम से इंस्टॉल करता है। फिर यह बॉट नंबर और `signal-cli` पथ माँगता है।

    गैर-संवादात्मक सेटअप के लिए, `openclaw channels add --channel signal` बॉट फ़ोन नंबर हेतु `--signal-number <e164>`, साथ ही Signal डेमन एंडपॉइंट के लिए `--http-host <host>` और `--http-port <port>` भी स्वीकार करता है (डिफ़ॉल्ट `127.0.0.1:8080`)।

  </Step>
  <Step title="खाता लिंक या पंजीकृत करें">
    - **QR लिंक (सबसे तेज़):** `signal-cli link -n "OpenClaw"`, फिर Signal से स्कैन करें। [पथ A](#setup-path-a-link-existing-signal-account-qr) देखें।
    - **SMS पंजीकरण:** कैप्चा + SMS सत्यापन वाला समर्पित नंबर। [पथ B](#setup-path-b-register-dedicated-bot-number-sms-linux) देखें।

  </Step>
  <Step title="सत्यापित करें और पेयर करें">
    ```bash
    openclaw gateway call channels.status --params '{"probe":true}'
    ```
    पहला DM भेजें और पेयरिंग स्वीकृत करें: `openclaw pairing approve signal <CODE>`।
  </Step>
</Steps>

न्यूनतम कॉन्फ़िगरेशन:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "managed-native",
        cliPath: "signal-cli",
      },
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

| फ़ील्ड       | विवरण                                       |
| ----------- | ------------------------------------------------- |
| `account`   | E.164 प्रारूप में बॉट फ़ोन नंबर (`+15551234567`) |
| `transport` | खाते के स्वामित्व वाला Signal कनेक्शन और प्रक्रिया मोड  |
| `dmPolicy`  | DM पहुँच नीति (`pairing` अनुशंसित)          |
| `allowFrom` | वे फ़ोन नंबर या `uuid:<id>` मान जिन्हें DM की अनुमति है |

बहु-खाता समर्थन: प्रति-खाता कॉन्फ़िगरेशन और वैकल्पिक `name` के साथ `channels.signal.accounts` का उपयोग करें। प्रत्येक नामित खाते का अपना `transport` होता है; उसे शीर्ष-स्तरीय ट्रांसपोर्ट विरासत में नहीं मिलता। शीर्ष-स्तरीय ट्रांसपोर्ट केवल अंतर्निहित `default` खाते का होता है। साझा प्रतिरूप के लिए [बहु-खाता चैनल](/hi/gateway/config-channels#multi-account-all-channels) देखें।

## यह क्या है

- नियतात्मक रूटिंग: उत्तर हमेशा Signal पर वापस जाते हैं।
- DM, एजेंट का मुख्य सत्र साझा करते हैं; समूह पृथक रहते हैं (`agent:<agentId>:signal:group:<groupId>`)।
- डिफ़ॉल्ट रूप से, Signal `/config set|unset` द्वारा आरंभ किए गए कॉन्फ़िगरेशन अपडेट लिख सकता है (`commands.config: true` आवश्यक)। `channels.signal.configWrites: false` से इसे अक्षम करें।

## सेटअप पथ A: मौजूदा Signal खाता लिंक करें (QR)

1. `signal-cli` (JVM या नेटिव बिल्ड) इंस्टॉल करें या `openclaw channels add` को आपके लिए इसे इंस्टॉल करने दें।
2. बॉट खाता लिंक करें: `signal-cli link -n "OpenClaw"`, फिर Signal में QR स्कैन करें।
3. Signal कॉन्फ़िगर करें और Gateway शुरू करें।

## सेटअप पथ B: समर्पित बॉट नंबर पंजीकृत करें (SMS, Linux)

मौजूदा Signal ऐप खाते को लिंक करने के बजाय समर्पित बॉट नंबर के लिए इसका उपयोग करें। नीचे दिए गए प्रवाह का परीक्षण Ubuntu 24 पर किया गया है।

1. ऐसा नंबर प्राप्त करें जो SMS प्राप्त कर सके (या लैंडलाइन के लिए वॉइस सत्यापन)। समर्पित बॉट नंबर खाता/सत्र टकराव से बचाता है।
2. Gateway होस्ट पर `signal-cli` इंस्टॉल करें:

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/
signal-cli --version
```

यदि आप JVM बिल्ड (`signal-cli-${VERSION}.tar.gz`) का उपयोग करते हैं, तो पहले JRE इंस्टॉल करें। `signal-cli` को अद्यतित रखें; अपस्ट्रीम के अनुसार Signal सर्वर API बदलने पर पुराने रिलीज़ काम करना बंद कर सकते हैं।

3. नंबर पंजीकृत और सत्यापित करें:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register
```

यदि कैप्चा आवश्यक हो (यह चरण पूरा करने के लिए ब्राउज़र पहुँच आवश्यक है):

1. `https://signalcaptchas.org/registration/generate.html` खोलें।
2. कैप्चा पूरा करें, "Open Signal" से `signalcaptcha://...` लिंक लक्ष्य कॉपी करें।
3. जहाँ संभव हो, ब्राउज़र सत्र के समान बाहरी IP से चलाएँ (कैप्चा टोकन शीघ्र समाप्त हो जाते हैं)।
4. तुरंत पंजीकृत और सत्यापित करें:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register --captcha '<SIGNALCAPTCHA_URL>'
signal-cli -a +<BOT_PHONE_NUMBER> verify <VERIFICATION_CODE>
```

4. OpenClaw कॉन्फ़िगर करें, Gateway पुनः आरंभ करें और चैनल सत्यापित करें:

```bash
# यदि आप Gateway को उपयोगकर्ता systemd सेवा के रूप में चलाते हैं:
systemctl --user restart openclaw-gateway.service

# फिर सत्यापित करें:
openclaw doctor
openclaw channels status --probe
```

5. अपने DM प्रेषक को पेयर करें:
   - बॉट नंबर पर कोई भी संदेश भेजें।
   - सर्वर पर स्वीकृत करें: `openclaw pairing approve signal <PAIRING_CODE>`।
   - “Unknown contact” से बचने के लिए बॉट नंबर को अपने फ़ोन में संपर्क के रूप में सहेजें।

<Warning>
`signal-cli` के साथ फ़ोन नंबर खाते को पंजीकृत करने से उस नंबर के मुख्य Signal ऐप सत्र का प्रमाणीकरण रद्द हो सकता है। समर्पित बॉट नंबर को प्राथमिकता दें या अपने मौजूदा फ़ोन ऐप सेटअप को बनाए रखने के लिए QR लिंक मोड का उपयोग करें।
</Warning>

अपस्ट्रीम संदर्भ:

- `signal-cli` README: `https://github.com/AsamK/signal-cli`
- कैप्चा प्रवाह: `https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha`
- लिंकिंग प्रवाह: `https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning)`

## बाहरी नेटिव डेमन मोड

`signal-cli` को स्वयं प्रबंधित करने के लिए (धीमा JVM कोल्ड स्टार्ट, कंटेनर आरंभीकरण, साझा CPU), डेमन को अलग से चलाएँ और OpenClaw को उसकी ओर इंगित करें:

गैर-संवादात्मक सेटअप के लिए, आवश्यकता होने पर एंडपॉइंट प्रकार स्पष्ट रूप से चुनें:

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://127.0.0.1:8080 --signal-transport external-native
```

```json5
{
  channels: {
    signal: {
      transport: {
        kind: "external-native",
        url: "http://127.0.0.1:8080",
      },
    },
  },
}
```

इससे स्वतः प्रारंभ करना और OpenClaw की स्टार्टअप प्रतीक्षा छोड़ दी जाती है। धीमे प्रारंभ वाले प्रबंधित डेमन के लिए `channels.signal.transport.startupTimeoutMs` सेट करें।

## कंटेनर मोड (bbernhard/signal-cli-rest-api)

`signal-cli` को नेटिव रूप से चलाने के बजाय [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) Docker कंटेनर का उपयोग करें, जो `signal-cli` को REST + WebSocket इंटरफ़ेस के पीछे आवृत करता है।

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://signal-cli:8080 --signal-transport container
```

आवश्यकताएँ:

- रीयल-टाइम संदेश प्राप्त करने के लिए कंटेनर को `MODE=json-rpc` के साथ चलना **अनिवार्य** है।
- OpenClaw कनेक्ट करने से पहले कंटेनर के भीतर अपना Signal खाता पंजीकृत या लिंक करें।

`docker-compose.yml` सेवा का उदाहरण:

```yaml
signal-cli:
  image: bbernhard/signal-cli-rest-api:latest
  environment:
    MODE: json-rpc
  ports:
    - "8080:8080"
  volumes:
    - signal-cli-data:/home/.local/share/signal-cli
```

OpenClaw कॉन्फ़िगरेशन:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "container",
        url: "http://signal-cli:8080",
      },
    },
  },
}
```

`transport.kind` नियंत्रित करता है कि OpenClaw किस प्रोटोकॉल और प्रक्रिया जीवनचक्र का उपयोग करता है:

| मान               | व्यवहार                                                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `"managed-native"`  | नेटिव signal-cli शुरू करें और `/api/v1/rpc` पर JSON-RPC तथा `/api/v1/events` पर SSE का उपयोग करें; `url` डेमन बाइंड से अलग कनेक्शन एंडपॉइंट चुन सकता है |
| `"external-native"` | पहले से चल रहे नेटिव signal-cli डेमन से कनेक्ट करें                                                                                                       |
| `"container"`       | `/v2/send` पर bbernhard REST और `/v1/receive/{account}` पर WebSocket से कनेक्ट करें                                                                             |

सेटअप और `openclaw doctor --fix` किसी मौजूदा एंडपॉइंट के ठोस प्रकार की पहचान करने के लिए उसकी एक बार जाँच कर सकते हैं। रनटाइम संचालन प्रोटोकॉल का स्वतः पता नहीं लगाते या उन्हें बदलते नहीं हैं।

जहाँ कंटेनर समकक्ष API उपलब्ध कराता है, वहाँ कंटेनर मोड नेटिव मोड जैसे ही Signal संचालन का समर्थन करता है: भेजना, प्राप्त करना, अटैचमेंट, टाइपिंग संकेतक, पढ़े/देखे जाने की रसीदें, प्रतिक्रियाएँ, समूह और शैलीबद्ध टेक्स्ट। OpenClaw नेटिव Signal RPC कॉल को कंटेनर के REST पेलोड में रूपांतरित करता है, जिसमें `group.{base64(internal_id)}` समूह ID और फ़ॉर्मैट किए गए टेक्स्ट के लिए `text_mode: "styled"` शामिल हैं।

संचालन संबंधी टिप्पणियाँ:

- प्राप्त करने के लिए `MODE=json-rpc` का उपयोग करें। `MODE=normal`, `/v1/about` को स्वस्थ दिखा सकता है, लेकिन `/v1/receive/{account}` WebSocket अपग्रेड नहीं करेगा, इसलिए कंटेनर प्राप्ति स्ट्रीमिंग की जाँच विफल होगी।
- bbernhard REST API के लिए `kind: "container"` और नेटिव `signal-cli` JSON-RPC/SSE के लिए `kind: "external-native"` सेट करें।
- कंटेनर अटैचमेंट डाउनलोड, नेटिव मोड जैसी ही मीडिया बाइट सीमाओं का पालन करते हैं। जब सर्वर `Content-Length` भेजता है, तब अत्यधिक बड़े प्रत्युत्तर पूरी तरह बफ़र होने से पहले अस्वीकार कर दिए जाते हैं; अन्यथा स्ट्रीमिंग के दौरान अस्वीकार किए जाते हैं।

## पहुँच नियंत्रण (DM + समूह)

DM:

- डिफ़ॉल्ट: `channels.signal.dmPolicy = "pairing"`।
- अज्ञात प्रेषकों को पेयरिंग कोड मिलता है; स्वीकृति मिलने तक संदेश अनदेखे किए जाते हैं (कोड 1 घंटे बाद समाप्त हो जाते हैं)।
- `openclaw pairing list signal` और `openclaw pairing approve signal <CODE>` के माध्यम से स्वीकृत करें।
- Signal DM के लिए पेयरिंग डिफ़ॉल्ट टोकन विनिमय है। विवरण: [पेयरिंग](/hi/channels/pairing)
- केवल UUID वाले प्रेषक (`sourceUuid` से) `channels.signal.allowFrom` में `uuid:<id>` के रूप में संग्रहीत किए जाते हैं।

समूह:

- `channels.signal.groupPolicy = open | allowlist | disabled`।
जब `allowlist` सेट हो, तब
- `channels.signal.groupAllowFrom` नियंत्रित करता है कि कौन-से समूह या प्रेषक समूह उत्तरों को ट्रिगर कर सकते हैं; प्रविष्टियाँ Signal समूह ID (रॉ, `group:<id>`, या `signal:group:<id>`), प्रेषक के फ़ोन नंबर, `uuid:<id>` मान, या `*` हो सकती हैं।
- `channels.signal.groups["<group-id>" | "*"]`, `requireMention`, `tools`, और `toolsBySender` के साथ समूह व्यवहार को ओवरराइड कर सकता है।
- बहु-अकाउंट सेटअप में प्रत्येक अकाउंट के ओवरराइड के लिए `channels.signal.accounts.<id>.groups` का उपयोग करें।
- `groupAllowFrom` के माध्यम से किसी Signal समूह को अनुमति-सूची में जोड़ना अपने आप उल्लेख गेटिंग को अक्षम नहीं करता। विशेष रूप से कॉन्फ़िगर की गई `channels.signal.groups["<group-id>"]` प्रविष्टि प्रत्येक समूह संदेश को संसाधित करती है, जब तक कि `requireMention=true` सेट न हो।
- `requireMention=true` के साथ, Signal के मूल @उल्लेख संरचित उल्लेख मेटाडेटा से बॉट अकाउंट के फ़ोन या `accountUuid` से मिलाए जाते हैं। कॉन्फ़िगर किए गए `mentionPatterns` सादे-पाठ फ़ॉलबैक बने रहते हैं।
- रनटाइम नोट: यदि `channels.signal` पूरी तरह अनुपस्थित है, तो रनटाइम समूह जाँच के लिए `groupPolicy="allowlist"` पर फ़ॉलबैक करता है (भले ही `channels.defaults.groupPolicy` सेट हो)।

सीमित संदर्भ वाला उल्लेख-गेटेड समूह:

```json5
{
  channels: {
    signal: {
      account: "+15551234567",
      accountUuid: "bot-signal-uuid",
      groupPolicy: "allowlist",
      groupAllowFrom: ["group:<signal-group-id>"],
      historyLimit: 8,
      groups: {
        "<signal-group-id>": { requireMention: true },
      },
    },
  },
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b"],
    },
  },
}
```

बॉट का उल्लेख न करने वाले अनुमत समूह संदेशों पर कोई प्रतिक्रिया नहीं होती और उन्हें केवल सीमित लंबित इतिहास विंडो में रखा जाता है। जब बाद में कोई मूल @उल्लेख या फ़ॉलबैक पाठ उल्लेख बॉट को ट्रिगर करता है, तो OpenClaw उस हालिया संदर्भ को शामिल करता है और उसी समूह को उत्तर देता है। छोड़ी गई अटैचमेंट सामग्री डाउनलोड नहीं की जाती; वह लंबित संदर्भ में केवल संक्षिप्त मीडिया प्लेसहोल्डर के रूप में दिखाई दे सकती है।

## यह कैसे काम करता है (व्यवहार)

- मूल मोड: `signal-cli` डेमन के रूप में चलता है; Gateway SSE के माध्यम से इवेंट पढ़ता है।
- कंटेनर मोड: Gateway REST API के माध्यम से भेजता है और WebSocket के माध्यम से प्राप्त करता है।
- आने वाले संदेशों को साझा चैनल एनवेलप में सामान्यीकृत किया जाता है।
- उत्तर हमेशा उसी नंबर या समूह पर वापस भेजे जाते हैं।
- जब बैकएंड आने वाले संदेश का टाइमस्टैम्प और लेखक स्वीकार करता है, तब उसके उत्तरों में मूल Signal उद्धरण मेटाडेटा शामिल होता है; यदि उद्धरण मेटाडेटा अनुपस्थित हो या अस्वीकार कर दिया जाए, तो OpenClaw उत्तर को सामान्य संदेश के रूप में भेजता है।
- मूल उद्धरण के उपयोग को `channels.signal.replyToMode = off | first | all | batched` से, या प्रत्येक चैट प्रकार के ओवरराइड के लिए `channels.signal.replyToModeByChatType.direct/group` से कॉन्फ़िगर करें। `channels.signal.accounts.<id>` के अंतर्गत अकाउंट-स्तरीय मानों को प्राथमिकता मिलती है।

## मीडिया + सीमाएँ

- आउटबाउंड पाठ को `channels.signal.textChunkLimit` के अनुसार खंडों में बाँटा जाता है (डिफ़ॉल्ट 4000)।
- वैकल्पिक नई-पंक्ति खंडीकरण: लंबाई के अनुसार खंडीकरण से पहले रिक्त पंक्तियों (अनुच्छेद सीमाओं) पर विभाजित करने के लिए `channels.signal.streaming.chunkMode="newline"` सेट करें।
- अटैचमेंट समर्थित हैं (`signal-cli` से प्राप्त base64)।
- जब `contentType` अनुपस्थित हो, तो वॉइस-नोट अटैचमेंट MIME फ़ॉलबैक के रूप में `signal-cli` फ़ाइल नाम का उपयोग करते हैं, ताकि ऑडियो ट्रांसक्रिप्शन फिर भी AAC वॉइस मेमो को वर्गीकृत कर सके।
- डिफ़ॉल्ट मीडिया सीमा: `channels.signal.mediaMaxMb` (डिफ़ॉल्ट 8)।
- किसी भी ट्रांसपोर्ट के लिए मीडिया डाउनलोड छोड़ने हेतु `channels.signal.ignoreAttachments` का उपयोग करें।
- समूह इतिहास संदर्भ `channels.signal.historyLimit` (या `channels.signal.accounts.*.historyLimit`) का उपयोग करता है और `messages.groupChat.historyLimit` पर फ़ॉलबैक करता है। अक्षम करने के लिए `0` सेट करें (डिफ़ॉल्ट 50)।

## टाइपिंग + पठन रसीदें

- **टाइपिंग संकेतक**: OpenClaw `signal-cli sendTyping` के माध्यम से टाइपिंग संकेत भेजता है और उत्तर तैयार होते समय उन्हें रीफ़्रेश करता है।
- **पठन रसीदें**: जब `channels.signal.sendReadReceipts` सत्य हो, तब OpenClaw अनुमत DM के लिए पठन रसीदें अग्रेषित करता है।
- `signal-cli` समूहों के लिए पठन रसीदें उपलब्ध नहीं कराता।

## जीवनचक्र स्थिति प्रतिक्रियाएँ

Signal को आने वाले टर्न पर साझा कतारबद्ध/विचाराधीन/टूल/Compaction/पूर्ण/त्रुटि प्रतिक्रिया जीवनचक्र दिखाने देने के लिए `messages.statusReactions.enabled: true` सेट करें। Signal आने वाले संदेश के टाइमस्टैम्प को प्रतिक्रिया लक्ष्य के रूप में उपयोग करता है; समूह प्रतिक्रियाएँ Signal समूह ID और लक्ष्य लेखक के रूप में मूल प्रेषक के साथ भेजी जाती हैं।

स्थिति प्रतिक्रियाओं के लिए एक अभिस्वीकृति प्रतिक्रिया और मेल खाता `messages.ackReactionScope` (`direct`, `group-all`, `group-mentions`, या `all`) भी आवश्यक है। Signal स्थिति प्रतिक्रियाएँ अक्षम करने के लिए `channels.signal.reactionLevel: "off"` सेट करें।

Signal अंतिम पूर्ण/त्रुटि स्थिति के बाद आरंभिक अभिस्वीकृति प्रतिक्रिया पुनर्स्थापित करता है।

## प्रतिक्रियाएँ (संदेश टूल)

`channel=signal` के साथ `message action=react` का उपयोग करें।

- लक्ष्य: प्रेषक E.164 या UUID (पेयरिंग आउटपुट से `uuid:<id>` का उपयोग करें; केवल UUID भी काम करता है)।
- `messageId` उस संदेश का Signal टाइमस्टैम्प है जिस पर आप प्रतिक्रिया दे रहे हैं।
- समूह प्रतिक्रियाओं के लिए `targetAuthor` या `targetAuthorUuid` आवश्यक है।

```text
message action=react channel=signal target=uuid:123e4567-e89b-12d3-a456-426614174000 messageId=1737630212345 emoji=🔥
message action=react channel=signal target=+15551234567 messageId=1737630212345 emoji=🔥 remove=true
message action=react channel=signal target=signal:group:<groupId> targetAuthor=uuid:<sender-uuid> messageId=1737630212345 emoji=✅
```

कॉन्फ़िगरेशन:

- `channels.signal.actions.reactions`: प्रतिक्रिया क्रियाएँ सक्षम/अक्षम करें (डिफ़ॉल्ट सत्य)।
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive` (डिफ़ॉल्ट `minimal`)।
  - `off`/`ack` एजेंट प्रतिक्रियाएँ अक्षम करता है (संदेश टूल `react` त्रुटि देता है)।
  - `minimal`/`extensive` एजेंट प्रतिक्रियाएँ सक्षम करता है और मार्गदर्शन स्तर सेट करता है।
- प्रत्येक अकाउंट के ओवरराइड: `channels.signal.accounts.<id>.actions.reactions`, `channels.signal.accounts.<id>.reactionLevel`।

## अनुमोदन प्रतिक्रियाएँ

Signal exec और Plugin अनुमोदन प्रॉम्प्ट शीर्ष-स्तरीय `approvals.exec` और `approvals.plugin` रूटिंग ब्लॉक का उपयोग करते हैं। Signal में कोई `channels.signal.execApprovals` ब्लॉक नहीं है।

- `👍` एक बार के लिए अनुमोदित करता है।
- `👎` अस्वीकार करता है।
- जब कोई अनुरोध स्थायी अनुमोदन प्रस्तुत करता है, तब `/approve <id> allow-always` का उपयोग करें।

अनुमोदन प्रतिक्रिया के समाधान के लिए `channels.signal.allowFrom`, `channels.signal.defaultTo`, या मेल खाने वाले अकाउंट-स्तरीय फ़ील्ड से स्पष्ट Signal अनुमोदक आवश्यक हैं। उसी चैट के प्रत्यक्ष exec अनुमोदन प्रॉम्प्ट स्पष्ट अनुमोदकों के बिना भी डुप्लिकेट स्थानीय `/approve` फ़ॉलबैक को छिपा सकते हैं; अनुमोदक-रहित समूह अनुमोदनों में स्थानीय फ़ॉलबैक दिखाई देता रहता है।

## प्रश्न प्रतिक्रियाएँ

एक गैर-गोपनीय, एकल-चयन प्रश्न और एक से चार विकल्पों वाले `ask_user` प्रॉम्प्ट के लिए, Signal विकल्प लेबलों के पास `1️⃣` से `4️⃣` दिखाता है। उत्तर देने के लिए वितरित प्रॉम्प्ट पर मेल खाते नंबर से प्रतिक्रिया दें। OpenClaw सत्यापित करता है कि प्रतिक्रिया बॉट द्वारा लिखे गए संदेश को लक्षित करती है, फिर Gateway के माध्यम से नंबर को प्रामाणिक विकल्प से मैप करता है। पुराने या डुप्लिकेट टैप अनदेखे किए जाते हैं। बहु-प्रश्न, बहु-चयन और मुक्त-पाठ प्रॉम्प्ट का उत्तर केवल पाठ से दिया जा सकता है; सामान्य Signal DM/समूह प्रवेश नियम प्रेषक को अधिकृत करते हैं।

## वितरण लक्ष्य (CLI/Cron)

- DM: `signal:+15551234567` (या केवल E.164)।
- UUID DM: `uuid:<id>` (या केवल UUID)।
- समूह: `signal:group:<groupId>`।
- उपयोगकर्ता नाम: `username:<name>` (यदि आपके Signal अकाउंट द्वारा समर्थित हो)।

## उपनाम

बार-बार उपयोग किए जाने वाले Signal लक्ष्यों के स्थिर नामों के लिए उपनाम कॉन्फ़िगर करें। उपनाम केवल OpenClaw-पक्ष का कॉन्फ़िगरेशन हैं; वे Signal संपर्क बनाते या संपादित नहीं करते।

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
        jane: "uuid:123e4567-e89b-12d3-a456-426614174000",
        ops: "group:<groupId>",
      },
      defaultTo: "signal:me",
    },
  },
}
```

Signal वितरण लक्ष्य स्वीकार किए जाने वाले किसी भी स्थान पर उपनामों का उपयोग करें:

```bash
openclaw message send --channel signal --target signal:ops --message "Deployment is complete"
```

प्रत्येक अकाउंट के उपनाम शीर्ष-स्तरीय उपनामों को इनहेरिट करते हैं और नाम जोड़ या ओवरराइड कर सकते हैं:

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
      },
      accounts: {
        work: {
          aliases: {
            ops: "group:<workGroupId>",
          },
        },
      },
    },
  },
}
```

`openclaw directory peers list --channel signal` और `openclaw directory groups list --channel signal` कॉन्फ़िगर किए गए उपनाम सूचीबद्ध करते हैं। Signal निर्देशिका कॉन्फ़िगरेशन-समर्थित है; यह Signal संपर्कों पर लाइव क्वेरी नहीं करती या Signal अकाउंट में बदलाव नहीं करती।

## समस्या निवारण

पहले यह क्रम चलाएँ:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

फिर आवश्यकता होने पर DM पेयरिंग स्थिति की पुष्टि करें:

```bash
openclaw pairing list signal
```

सामान्य विफलताएँ:

- डेमन पहुँच योग्य है लेकिन उत्तर नहीं मिल रहे: `account`, `transport.kind`, ट्रांसपोर्ट URL, और प्राप्ति मोड सत्यापित करें।
- DM अनदेखे किए गए: प्रेषक का पेयरिंग अनुमोदन लंबित है।
- समूह संदेश अनदेखे किए गए: समूह प्रेषक/उल्लेख गेटिंग वितरण को अवरुद्ध करती है।
- संपादन के बाद कॉन्फ़िगरेशन सत्यापन त्रुटियाँ: `openclaw doctor --fix` चलाएँ।
- निदान में Signal अनुपस्थित है: `channels.signal.enabled: true` की पुष्टि करें।

अतिरिक्त जाँच:

```bash
openclaw pairing list signal
pgrep -af signal-cli
openclaw logs --plain --limit 500 | grep -i "signal" | tail -20
```

ट्रायेज प्रवाह के लिए: [चैनल समस्या निवारण](/hi/channels/troubleshooting)।

## सुरक्षा नोट्स

- `signal-cli` अकाउंट कुंजियाँ स्थानीय रूप से संग्रहीत करता है (आमतौर पर `~/.local/share/signal-cli/data/`)।
- सर्वर माइग्रेशन या पुनर्निर्माण से पहले Signal अकाउंट स्थिति का बैकअप लें।
- `channels.signal.dmPolicy: "pairing"` बनाए रखें, जब तक आप स्पष्ट रूप से अधिक व्यापक DM पहुँच नहीं चाहते।
- SMS सत्यापन केवल पंजीकरण या पुनर्प्राप्ति प्रवाहों के लिए आवश्यक है, लेकिन नंबर/अकाउंट का नियंत्रण खोने से पुनः पंजीकरण जटिल हो सकता है।

## कॉन्फ़िगरेशन संदर्भ (Signal)

पूर्ण कॉन्फ़िगरेशन: [कॉन्फ़िगरेशन](/hi/gateway/configuration)

प्रदाता विकल्प:

- `channels.signal.enabled`: चैनल स्टार्टअप सक्षम/अक्षम करें।
- `channels.signal.account`: बॉट खाते के लिए E.164।
- `channels.signal.accountUuid`: मूल @mention पहचान और लूप सुरक्षा के लिए वैकल्पिक बॉट खाता UUID।
- `channels.signal.transport`: खाते के स्वामित्व वाला ट्रांसपोर्ट। प्रबंधित मूल डिफ़ॉल्ट के लिए इसे छोड़ दें।
- `channels.signal.transport.kind`: `managed-native | external-native | container`।
- `channels.signal.transport.url`: `external-native` और `container` के लिए आवश्यक; `managed-native` के लिए वैकल्पिक, जब उसका कनेक्शन एंडपॉइंट डेमन बाइंड से अलग हो।
- `channels.signal.transport.cliPath`: `signal-cli` का प्रबंधित-मूल पथ।
- `channels.signal.transport.configPath`: वैकल्पिक प्रबंधित-मूल `signal-cli --config` डायरेक्टरी।
- `channels.signal.transport.httpHost`, `channels.signal.transport.httpPort`: प्रबंधित-मूल डेमन बाइंड (डिफ़ॉल्ट `127.0.0.1:8080`)।
- `channels.signal.transport.startupTimeoutMs`: प्रबंधित-मूल स्टार्टअप प्रतीक्षा, ms में (न्यूनतम 1000, अधिकतम 120000; डिफ़ॉल्ट 30000)।
- `channels.signal.transport.receiveMode`: प्रबंधित-मूल `on-start | manual`।
- `channels.signal.ignoreAttachments`: इस खाते के लिए आने वाले अटैचमेंट का डाउनलोड छोड़ें।
- `channels.signal.transport.ignoreStories`: प्रबंधित-मूल स्टोरी टॉगल।
- `channels.signal.sendReadReceipts`: पठन रसीदें अग्रेषित करें।
- `channels.signal.dmPolicy`: `pairing | allowlist | open | disabled` (डिफ़ॉल्ट: पेयरिंग)।
- `channels.signal.allowFrom`: DM अनुमति-सूची (E.164 या `uuid:<id>`)। `open` के लिए `"*"` आवश्यक है। Signal में उपयोगकर्ता नाम नहीं होते; फ़ोन/UUID ID का उपयोग करें।
- `channels.signal.aliases`: DM या समूह डिलीवरी लक्ष्यों के लिए OpenClaw-पक्ष के उपनाम।
- `channels.signal.groupPolicy`: `open | allowlist | disabled` (डिफ़ॉल्ट: अनुमति-सूची)।
- `channels.signal.groupAllowFrom`: समूह अनुमति-सूची; Signal समूह ID (अपरिष्कृत, `group:<id>`, या `signal:group:<id>`), प्रेषक के E.164 नंबर, या `uuid:<id>` मान स्वीकार करती है।
- `channels.signal.groups`: Signal समूह ID (या `"*"`) द्वारा कुंजीबद्ध प्रति-समूह ओवरराइड। समर्थित फ़ील्ड: `requireMention`, `tools`, `toolsBySender`।
- `channels.signal.accounts.<id>.groups`: एकाधिक-खाता सेटअप के लिए `channels.signal.groups` का प्रति-खाता संस्करण।
- `channels.signal.accounts.<id>.aliases`: प्रति-खाता उपनाम, शीर्ष-स्तरीय उपनामों के साथ मर्ज किए जाते हैं।
- `channels.signal.replyToMode`: मूल उत्तर उद्धरण मोड, `off | first | all | batched` (डिफ़ॉल्ट: `all`)।
- `channels.signal.replyToModeByChatType.direct`, `channels.signal.replyToModeByChatType.group`: प्रति-चैट-प्रकार मूल उत्तर उद्धरण ओवरराइड।
- `channels.signal.accounts.<id>.replyToMode`, `channels.signal.accounts.<id>.replyToModeByChatType.direct`, `channels.signal.accounts.<id>.replyToModeByChatType.group`: प्रति-खाता उत्तर उद्धरण ओवरराइड।
- `channels.signal.historyLimit`: संदर्भ के रूप में शामिल किए जाने वाले समूह संदेशों की अधिकतम संख्या (0 अक्षम करता है)।
- `channels.signal.dmHistoryLimit`: उपयोगकर्ता टर्न में DM इतिहास सीमा। प्रति-उपयोगकर्ता ओवरराइड: `channels.signal.dms["<phone_or_uuid>"].historyLimit`।
- `channels.signal.textChunkLimit`: वर्णों में आउटबाउंड खंड आकार (डिफ़ॉल्ट 4000)।
- `channels.signal.streaming.chunkMode`: लंबाई के आधार पर खंडित करने से पहले रिक्त पंक्तियों (अनुच्छेद सीमाओं) पर विभाजित करने के लिए `length` (डिफ़ॉल्ट) या `newline`।
- `channels.signal.mediaMaxMb`: इनबाउंड/आउटबाउंड मीडिया सीमा, MB में (डिफ़ॉल्ट 8)।
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive` (डिफ़ॉल्ट `minimal`)। [प्रतिक्रियाएँ](#reactions-message-tool) देखें।
- `channels.signal.reactionNotifications`: `off | own | all | allowlist` (डिफ़ॉल्ट `own`) - एजेंट को अन्य लोगों से आने वाली प्रतिक्रियाओं की सूचना कब दी जाती है।
- `channels.signal.reactionAllowlist`: वे प्रेषक जिनकी प्रतिक्रियाएँ `reactionNotifications: "allowlist"` होने पर एजेंट को सूचित करती हैं।
- `channels.signal.streaming.block.enabled`, `channels.signal.streaming.block.coalesce`: सभी चैनलों में साझा किए गए ब्लॉक-मोड स्ट्रीमिंग नियंत्रण। [स्ट्रीमिंग](/hi/concepts/streaming) देखें।

संबंधित वैश्विक विकल्प:

- `agents.entries.*.groupChat.mentionPatterns` (सादा-पाठ फ़ॉलबैक; बॉट खाते की पहचान कॉन्फ़िगर होने पर Signal के मूल @mentions की पहचान संरचित मेटाडेटा से की जाती है)।
- `messages.groupChat.mentionPatterns` (वैश्विक फ़ॉलबैक)।
- `channels.signal.responsePrefix` या खाता-स्तरीय `responsePrefix`।

## संबंधित

- [चैनल अवलोकन](/hi/channels) - सभी समर्थित चैनल
- [पेयरिंग](/hi/channels/pairing) - DM प्रमाणीकरण और पेयरिंग प्रवाह
- [समूह](/hi/channels/groups) - समूह चैट व्यवहार और उल्लेख गेटिंग
- [चैनल रूटिंग](/hi/channels/channel-routing) - संदेशों के लिए सत्र रूटिंग
- [सुरक्षा](/hi/gateway/security) - पहुँच मॉडल और सुदृढ़ीकरण
