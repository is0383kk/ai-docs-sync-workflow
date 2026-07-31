---
read_when:
    - Discord Activity विजेट सेट अप करना या उनकी समस्याओं का निवारण करना
summary: Discord Activities के अंदर स्व-निहित OpenClaw HTML विजेट लॉन्च करें
title: Discord गतिविधियाँ
x-i18n:
    generated_at: "2026-07-27T18:53:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b1bc04443aef89fd514290c3bebdbdd3e9972298b45cae3806bec99344f6d8cd
    source_path: channels/discord-activities.md
    workflow: 16
---

Discord Activities किसी एजेंट को वर्तमान Discord चैनल में एक इंटरैक्टिव, स्व-निहित HTML विजेट पोस्ट करने देती हैं। संदेश में **Open widget** बटन शामिल होता है; इसे क्लिक करने पर विजेट Discord के भीतर खुलता है।

यह सुविधा डिफ़ॉल्ट रूप से बंद रहती है। OpenClaw Activity HTTP रूट, `show_widget` एजेंट टूल और लॉन्च-बटन हैंडलर को केवल तभी पंजीकृत करता है, जब `channels.discord.activities` मौजूद हो और क्लाइंट सीक्रेट उपलब्ध हो। अप्रचलित `discord_widget` उपनाम एक रिलीज़ तक उपलब्ध रहेगा।

## पूर्वापेक्षाएँ

- एक मौजूदा [OpenClaw Discord बॉट](/hi/channels/discord)
- एक सार्वजनिक HTTPS होस्टनाम, जो OpenClaw Gateway तक पहुँचता हो
- बॉट के Discord एप्लिकेशन के लिए Activities और OAuth2 कॉन्फ़िगर करने की अनुमति

कोई भी HTTPS रिवर्स प्रॉक्सी या टनल काम करेगी। नामित Cloudflare Tunnel, Gateway पोर्ट को सीधे उजागर किए बिना एक स्थिर होस्टनाम प्रदान करती है।

```yaml
# ~/.cloudflared/config.yml
tunnel: openclaw-discord
credentials-file: /home/you/.cloudflared/TUNNEL-ID.json
ingress:
  - hostname: openclaw.example.com
    service: http://127.0.0.1:18789
  - service: http_status:404
```

```bash
cloudflared tunnel login
cloudflared tunnel create openclaw-discord
cloudflared tunnel route dns openclaw-discord openclaw.example.com
cloudflared tunnel run openclaw-discord
```

सामान्य Gateway प्रमाणीकरण सक्षम रखें। केवल Activity प्रीफ़िक्स सार्वजनिक होता है, और Plugin स्वयं OAuth, Activity इंस्टेंस सदस्यता, चैनल बाइंडिंग, सत्रों तथा एक-बार उपयोग होने वाली दस्तावेज़ क्षमताओं को सत्यापित करता है।

## सेटअप

<Steps>
  <Step title="Gateway को HTTPS पर उपलब्ध कराएँ">
    अपनी टनल या रिवर्स प्रॉक्सी शुरू करें और Activities कॉन्फ़िगरेशन जोड़ने के बाद सत्यापित करें कि `https://openclaw.example.com/discord/activity/` Gateway तक पहुँचता है। उदाहरण होस्टनाम को अपने होस्टनाम से बदलें।
  </Step>

  <Step title="Discord में Activities सक्षम करें">
    मौजूदा बॉट एप्लिकेशन को [Discord Developer Portal](https://discord.com/developers/applications) में खोलें। **Activities** खोलें, Activities सक्षम करें और एक URL मैपिंग बनाएँ:

    - प्रीफ़िक्स: `ROOT` (`/`)
    - लक्ष्य: `openclaw.example.com/discord/activity`

    लक्ष्य सार्वजनिक होस्टनाम और `/discord/activity` का संयोजन है, जिसके अंत में स्लैश नहीं होना चाहिए।

  </Step>

  <Step title="OAuth2 क्लाइंट सीक्रेट कॉपी करें">
    Developer Portal में **OAuth2** खोलें। Discord को कम-से-कम एक रीडायरेक्ट URI चाहिए, इसलिए यदि एप्लिकेशन में अभी कोई नहीं है, तो लूपबैक पते जैसा कोई स्थानीय प्लेसहोल्डर जोड़ें; Embedded App SDK Activity की वापसी प्रक्रिया को संभालती है। एप्लिकेशन क्लाइंट सीक्रेट को कॉपी या रीसेट करें। इसे क्रेडेंशियल मानें: इसे चैट, लॉग या कमिट की गई कॉन्फ़िगरेशन फ़ाइल में पेस्ट न करें।
  </Step>

  <Step title="OpenClaw कॉन्फ़िगर करें">
    उस Discord खाते में एक ब्लॉक जोड़ें, जिसे विजेट उपलब्ध कराने चाहिए:

    ```json5
    {
      channels: {
        discord: {
          token: "${DISCORD_BOT_TOKEN}",
          activities: {
            clientSecret: "${DISCORD_CLIENT_SECRET}",
            // वैकल्पिक। डिफ़ॉल्ट रूप से स्टार्टअप पर ज्ञात हुए बॉट एप्लिकेशन ID का उपयोग होता है।
            applicationId: "YOUR_DISCORD_APPLICATION_ID",
          },
        },
      },
    }
    ```

    जब `DISCORD_CLIENT_SECRET` सेट हो, तब आप ब्लॉक से `clientSecret` हटा सकते हैं। सुविधा के लिए सहमति देने हेतु ब्लॉक स्वयं मौजूद रहना चाहिए।

    सामान्य Discord पहुँच सेटिंग अलग रहती हैं। उदाहरण के लिए, `allowFrom` अब भी नियंत्रित करता है कि एजेंट को कौन DM भेज सकता है; यह नियंत्रित नहीं करता कि चैनल में पहले से पोस्ट किया गया विजेट कौन खोल सकता है।

  </Step>

  <Step title="पुनः आरंभ करें और परीक्षण करें">
    Gateway को पुनः आरंभ करें। किसी Discord वार्तालाप में एजेंट से एक इंटरैक्टिव विजेट दिखाने के लिए कहें। एजेंट `show_widget` को कॉल करता है; पोस्ट किए गए संदेश पर **Open widget** क्लिक करें।
  </Step>
</Steps>

## सुरक्षा मॉडल

- विजेट मेटाडेटा लौटाए जाने से पहले OAuth Discord उपयोगकर्ता की पहचान करता है।
- Discord की Get Activity Instance API को पुष्टि करनी होगी कि OAuth उपयोगकर्ता वर्तमान Activity इंस्टेंस में मौजूद है। इंस्टेंस चैनल उस चैनल से मेल खाना चाहिए, जहाँ विजेट पोस्ट किया गया था।
- Discord जिस किसी को उस चैनल में प्रवेश की अनुमति देता है, वह उसके विजेट खोल सकता है। दर्शकों को सीमित करने के लिए Discord चैनल अनुमतियों का उपयोग करें। OpenClaw कमांड और DM अनुमतिसूचियाँ पहले से पोस्ट की गई चैनल सामग्री की पहुँच प्रदान या समाप्त नहीं करतीं।
- OAuth सत्र 15 मिनट बाद समाप्त हो जाते हैं। विजेट दस्तावेज़ क्षमताएँ 60 सेकंड बाद समाप्त हो जाती हैं और केवल एक बार काम करती हैं।
- विजेट सात दिन बाद समाप्त हो जाते हैं और प्रत्येक Discord Plugin इंस्टेंस में अधिकतम 64 विजेट रखे जाते हैं।
- विजेट HTML आपके एजेंट द्वारा बनाया जाता है और इसे विश्वसनीय सामग्री माना जाना चाहिए। ऐसे सीक्रेट एम्बेड न करें, जिन्हें आप किसी त्रुटिपूर्ण विजेट द्वारा उजागर होते नहीं देखना चाहेंगे।
- विजेट अपने नेस्टेड फ़्रेम के भीतर नेविगेट कर सकता है। `sandbox="allow-scripts"` iframe शीर्ष-स्तरीय नेविगेशन, पॉपअप और समान-ओरिजिन पहुँच को अवरुद्ध करता है, जबकि उसकी Content Security Policy नेटवर्क कनेक्शन और बाहरी संसाधनों को अवरुद्ध करती है। ये नियंत्रण बहुस्तरीय सुरक्षा प्रदान करते हैं, लेकिन विजेट बनाने वाले एजेंट के विरुद्ध सुरक्षा सीमा नहीं हैं।
- Activities अक्षम होने पर `/discord/activity` बिल्कुल पंजीकृत नहीं होता।

सक्षम होने पर सार्वजनिक Activity शेल और टोकन-एक्सचेंज रूट आपकी टनल के माध्यम से पहुँच योग्य हो जाते हैं। वे वैध OAuth सत्र और एक-बार उपयोग होने वाली दस्तावेज़ क्षमता के बिना विजेट HTML उजागर नहीं करते।

## समस्या निवारण

### Activity में “Gateway offline” दिखाई देता है

- पुष्टि करें कि टनल चल रही है और Gateway के वास्तविक बाइंड पोर्ट पर रूट करती है
- पुष्टि करें कि Developer Portal के लक्ष्य में `/discord/activity` शामिल है
- Discord या OpenClaw कॉन्फ़िगरेशन बदलने के बाद Gateway को पुनः आरंभ करें
- Activities क्लाइंट सीक्रेट अनुपलब्ध होने से संबंधित एक-पंक्ति चेतावनी के लिए Gateway लॉग जाँचें

### Discord एक खाली पृष्ठ खोलता है या `blocked:csp` की सूचना देता है

- सत्यापित करें कि URL मैपिंग `ROOT` का उपयोग करती है और दूसरा `/discord/activity` खंड नहीं जोड़ती
- पुष्टि करें कि शेल, `shell.js` और SDK मॉड्यूल सभी Discord प्रॉक्सी के माध्यम से लौटते हैं
- `/discord/activity/` के अंतर्गत अनुरोधों के लिए Gateway लॉग देखें

विजेट नेटवर्क अनुरोध जानबूझकर अवरुद्ध किए जाते हैं। विजेट के लिए आवश्यक सभी CSS, JavaScript, छवियाँ और डेटा इनलाइन करें।

### “Widget unavailable”

बटन को उसी चैनल से लॉन्च करें, जहाँ एजेंट ने उसे पोस्ट किया था। क्लिक किए जाने पर OpenClaw सर्वर की ओर लॉन्च को ट्रैक करता है, इसलिए Discord द्वारा बटन की कस्टम ID छोड़े जाने या बिगाड़े जाने पर भी नया लॉन्च रिकॉर्ड सटीक विजेट का पता लगा सकता है। जब न तो कस्टम ID और न ही लॉन्च रिकॉर्ड का पता चलता है, तब OpenClaw उस चैनल में सबसे हाल में पोस्ट किया गया सक्रिय विजेट खोलता है। जिन बटनों में उनकी कस्टम ID सुरक्षित रहती है, उनके माध्यम से पुराने विजेट भी पहुँच योग्य रहते हैं।

### “You cannot launch Activities in this channel”

Discord फ़ोरम-पोस्ट थ्रेड से Activities लॉन्च नहीं करता। OpenClaw वहाँ विजेट संदेश और बटन पोस्ट कर सकता है, लेकिन Activity को इसके बजाय किसी सामान्य टेक्स्ट चैनल से लॉन्च करें। यह प्रतिबंध Discord से आता है, OpenClaw से नहीं।
