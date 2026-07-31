---
read_when:
    - OpenClaw को पहली बार सेट अप करना
    - सामान्य कॉन्फ़िगरेशन पैटर्न खोजे जा रहे हैं
    - विशिष्ट कॉन्फ़िग अनुभागों पर जाना
summary: 'कॉन्फ़िगरेशन अवलोकन: सामान्य कार्य, त्वरित सेटअप और पूर्ण संदर्भ के लिंक'
title: कॉन्फ़िगरेशन
x-i18n:
    generated_at: "2026-07-27T19:18:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09cc04efa16f32e12d6ebcea7a1d36b336df32227fe66953c5d70107708ee6c3
    source_path: gateway/configuration.md
    workflow: 16
---

OpenClaw `~/.openclaw/openclaw.json` से एक वैकल्पिक <Tooltip tip="JSON5 टिप्पणियों और अंत में लगे अल्पविरामों का समर्थन करता है">**JSON5**</Tooltip> कॉन्फ़िग पढ़ता है। यदि फ़ाइल मौजूद नहीं है, तो OpenClaw सुरक्षित डिफ़ॉल्ट का उपयोग करता है।

सक्रिय कॉन्फ़िग पथ एक सामान्य फ़ाइल होना चाहिए। OpenClaw द्वारा किए गए लेखन इसे परमाण्विक रूप से बदलते हैं (पथ पर नाम बदलकर), इसलिए सिमलिंक किए गए `openclaw.json` में उसके लक्ष्य के माध्यम से लिखने के बजाय लक्ष्य को बदल दिया जाता है—सिमलिंक किए गए कॉन्फ़िग लेआउट से बचें। यदि आप कॉन्फ़िग को डिफ़ॉल्ट स्टेट डायरेक्टरी से बाहर रखते हैं, तो `OPENCLAW_CONFIG_PATH` को सीधे वास्तविक फ़ाइल पर इंगित करें।

कॉन्फ़िग जोड़ने के सामान्य कारण:

- चैनल कनेक्ट करें और नियंत्रित करें कि बॉट को कौन संदेश भेज सकता है
- मॉडल, टूल, सैंडबॉक्सिंग या ऑटोमेशन (cron, hooks) सेट करें
- सेशन, मीडिया, नेटवर्किंग या UI को समायोजित करें

उपलब्ध प्रत्येक फ़ील्ड के लिए [पूरा संदर्भ](/hi/gateway/configuration-reference) देखें।

कॉन्फ़िगरेशन दो-बकेट नियम का पालन करता है: रूट सिबलिंग में इन्फ़्रास्ट्रक्चर और क्रॉस-एजेंट डिफ़ॉल्ट होते हैं, जबकि `agents.defaults` में एजेंट-लूप व्यवहार होता है। जहाँ स्कीमा प्रति-एजेंट ओवरराइड का समर्थन करता है, वहाँ `agents.entries` के अंतर्गत प्रविष्टियाँ किसी भी बकेट को ओवरराइड कर सकती हैं।

एजेंट और ऑटोमेशन को कॉन्फ़िग संपादित करने से पहले सटीक फ़ील्ड-स्तरीय
दस्तावेज़ों के लिए `config.schema.lookup` का उपयोग करना चाहिए। कार्य-उन्मुख मार्गदर्शन के लिए इस पृष्ठ और
व्यापक फ़ील्ड मैप तथा डिफ़ॉल्ट के लिए
[कॉन्फ़िगरेशन संदर्भ](/hi/gateway/configuration-reference) का उपयोग करें।

<Tip>
**कॉन्फ़िगरेशन में नए हैं?** इंटरैक्टिव सेटअप के लिए `openclaw onboard` से शुरू करें, या पूरी तरह कॉपी-पेस्ट किए जा सकने वाले कॉन्फ़िग के लिए [कॉन्फ़िगरेशन उदाहरण](/hi/gateway/configuration-examples) मार्गदर्शिका देखें।
</Tip>

## न्यूनतम कॉन्फ़िग

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## कॉन्फ़िग संपादित करना

<Tabs>
  <Tab title="इंटरैक्टिव विज़ार्ड">
    ```bash
    openclaw onboard       # पूरा ऑनबोर्डिंग प्रवाह
    openclaw configure     # कॉन्फ़िग विज़ार्ड
    ```
  </Tab>
  <Tab title="CLI (एक-पंक्ति कमांड)">
    ```bash
    openclaw config get agents.defaults.workspace
    openclaw config set agents.defaults.heartbeat.every "2h"
    openclaw config unset plugins.entries.brave.config.webSearch.apiKey
    ```
  </Tab>
  <Tab title="कंट्रोल UI">
    [http://127.0.0.1:18789](http://127.0.0.1:18789) खोलें और **Config** टैब का उपयोग करें।
    कंट्रोल UI लाइव कॉन्फ़िग स्कीमा से एक फ़ॉर्म रेंडर करता है, जिसमें उपलब्ध होने पर फ़ील्ड
    `title` / `description` दस्तावेज़ मेटाडेटा के साथ Plugin और चैनल स्कीमा भी
    शामिल होते हैं, और वैकल्पिक उपाय के रूप में एक **Raw JSON** संपादक उपलब्ध होता है। विस्तृत
    UI और अन्य टूलिंग के लिए, Gateway एक पथ-स्कोप्ड स्कीमा नोड और उसके निकटतम चाइल्ड सारांश
    प्राप्त करने हेतु `config.schema.lookup` भी उपलब्ध कराता है।
    सेटिंग्स पहले सामान्य फ़ील्ड दिखाती हैं। प्रत्येक अनुभाग अपने उन्नत फ़ील्ड को
    संकुचित **Advanced (N)** समूह में रखता है; सभी समूह विस्तृत करने के लिए **Show advanced** का उपयोग करें।
    सेटिंग्स खोज में हमेशा दोनों स्तर शामिल होते हैं और आवश्यकता पड़ने पर मेल खाने वाला
    उन्नत समूह खुल जाता है।
  </Tab>
  <Tab title="प्रत्यक्ष संपादन">
    `~/.openclaw/openclaw.json` को सीधे संपादित करें। Gateway फ़ाइल पर नज़र रखता है और बदलाव स्वचालित रूप से लागू करता है ([हॉट रीलोड](#config-hot-reload) देखें)।
  </Tab>
</Tabs>

## कठोर सत्यापन

<Warning>
OpenClaw केवल उन्हीं कॉन्फ़िगरेशन को स्वीकार करता है जो स्कीमा से पूरी तरह मेल खाते हैं। अज्ञात कुंजियों, विकृत प्रकारों या अमान्य मानों के कारण Gateway **शुरू होने से इनकार करता है**। रूट स्तर पर एकमात्र अपवाद `$schema` (स्ट्रिंग) है, ताकि संपादक JSON Schema मेटाडेटा संलग्न कर सकें।
</Warning>

`openclaw config schema` कंट्रोल UI और सत्यापन द्वारा उपयोग किया जाने वाला कैनोनिकल JSON Schema प्रिंट करता है।
`config.schema.lookup` विस्तृत टूलिंग के लिए एक पथ-स्कोप्ड नोड और
चाइल्ड सारांश प्राप्त करता है। फ़ील्ड `title`/`description` दस्तावेज़ मेटाडेटा
नेस्टेड ऑब्जेक्ट, वाइल्डकार्ड (`*`), ऐरे-आइटम (`[]`), और `anyOf`/
`oneOf`/`allOf` शाखाओं में आगे बढ़ता है। मैनिफ़ेस्ट रजिस्ट्री लोड होने पर रनटाइम Plugin और चैनल स्कीमा मर्ज हो जाते हैं।

प्रत्येक कॉन्फ़िग लीफ़ का `uiHints` में एक सामान्य या उन्नत प्रस्तुति स्तर होता है।
`advanced: false` सामान्य सेटिंग्स को और `advanced: true` उन्नत
सेटिंग्स को चिह्नित करता है। यदि किसी लीफ़ का कोई प्रत्यक्ष संकेत नहीं है, तो वह निकटतम पूर्वज का स्तर ग्रहण करता है;
बिना घोषित पूर्वज वाले पथ डिफ़ॉल्ट रूप से उन्नत होते हैं। यह केवल प्रस्तुति को प्रभावित करता है,
सत्यापन, डिफ़ॉल्ट, रीलोड व्यवहार या कुंजी सेट की जा सकती है या नहीं, इसे नहीं।

सत्यापन विफल होने पर:

- Gateway बूट नहीं होता
- केवल निदान कमांड काम करते हैं (`openclaw doctor`, `openclaw logs`, `openclaw health`, `openclaw status`)
- सटीक समस्याएँ देखने के लिए `openclaw doctor` चलाएँ
- मरम्मत लागू करने के लिए `openclaw doctor --fix` चलाएँ (`--repair` वही फ़्लैग है; `--yes` प्रॉम्प्ट छोड़ देता है)

Gateway प्रत्येक सफल स्टार्टअप के बाद अंतिम ज्ञात सही कॉपी को विश्वसनीय रूप से रखता है,
लेकिन स्टार्टअप और हॉट रीलोड इसे स्वचालित रूप से पुनर्स्थापित नहीं करते—केवल `openclaw doctor --fix`
ऐसा करता है। यदि `openclaw.json` सत्यापन में विफल होता है (Plugin-स्थानीय सत्यापन सहित), तो Gateway
स्टार्टअप विफल हो जाता है या रीलोड छोड़ दिया जाता है और वर्तमान रनटाइम अंतिम स्वीकृत
कॉन्फ़िग का उपयोग जारी रखता है। अस्वीकृत लेखन को निरीक्षण के लिए `<path>.rejected.<timestamp>` के रूप में भी सहेजा जाता है।
Gateway उन लेखनों को रोकता है जो आकस्मिक ओवरराइट जैसे लगते हैं—`gateway.mode` को हटाना,
`meta` ब्लॉक खोना या फ़ाइल को आधे से अधिक छोटा करना—जब तक कि लेखन
विनाशकारी बदलावों को स्पष्ट रूप से अनुमति न दे। यदि किसी उम्मीदवार में `***` या `[redacted]` जैसा
रिडैक्ट किया हुआ सीक्रेट प्लेसहोल्डर हो, तो उसे अंतिम ज्ञात सही कॉपी के रूप में पदोन्नत नहीं किया जाता।

## सामान्य कार्य

<AccordionGroup>
  <Accordion title="चैनल सेट अप करें (WhatsApp, Telegram, Discord आदि)">
    प्रत्येक चैनल का `channels.<provider>` के अंतर्गत अपना कॉन्फ़िग अनुभाग होता है। सेटअप चरणों के लिए समर्पित चैनल पृष्ठ देखें:

    - [Discord](/hi/channels/discord) - `channels.discord`
    - [Feishu](/hi/channels/feishu) - `channels.feishu`
    - [Google Chat](/hi/channels/googlechat) - `channels.googlechat`
    - [iMessage](/hi/channels/imessage) - `channels.imessage`
    - [Mattermost](/hi/channels/mattermost) - `channels.mattermost`
    - [Microsoft Teams](/hi/channels/msteams) - `channels.msteams`
    - [Signal](/hi/channels/signal) - `channels.signal`
    - [Slack](/hi/channels/slack) - `channels.slack`
    - [Telegram](/hi/channels/telegram) - `channels.telegram`
    - [WhatsApp](/hi/channels/whatsapp) - `channels.whatsapp`

    सभी चैनल समान DM नीति पैटर्न साझा करते हैं:

    ```json5
    {
      channels: {
        telegram: {
          enabled: true,
          botToken: "123:abc",
          dmPolicy: "pairing",   // पेयरिंग | अनुमति-सूची | खुला | अक्षम
          allowFrom: ["tg:123"], // केवल अनुमति-सूची/खुले के लिए
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="मॉडल चुनें और कॉन्फ़िगर करें">
    प्राथमिक मॉडल और वैकल्पिक फ़ॉलबैक सेट करें:

    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "anthropic/claude-sonnet-4-6",
            fallbacks: ["openai/gpt-5.4"],
          },
          models: {
            "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
            "openai/gpt-5.4": { alias: "GPT" },
          },
        },
      },
    }
    ```

    - `agents.defaults.models` उपनाम और प्रति-मॉडल सेटिंग्स संग्रहीत करता है; प्रविष्टि जोड़ने से `/model` या `--model` ओवरराइड कभी प्रतिबंधित नहीं होते।
    - `agents.defaults.modelPolicy.allow` ओवरराइड और मॉडल चयनकर्ताओं के लिए स्पष्ट अनुमति-सूची है। यह सटीक संदर्भ और `provider/*` वाइल्डकार्ड स्वीकार करता है; किसी भी मॉडल की अनुमति देने के लिए इसे छोड़ दें या `[]` का उपयोग करें।
    - मॉडल संदर्भ `provider/model` प्रारूप का उपयोग करते हैं (उदाहरण: `anthropic/claude-opus-4-6`)।
    - `agents.defaults.imageMaxDimensionPx` ट्रांसक्रिप्ट/टूल इमेज डाउनस्केलिंग को नियंत्रित करता है (डिफ़ॉल्ट `1200`); कम मान आम तौर पर स्क्रीनशॉट-प्रधान रन में विज़न-टोकन उपयोग घटाते हैं।
    - चैट में मॉडल बदलने के लिए [मॉडल CLI](/hi/concepts/models) और प्रमाणीकरण रोटेशन तथा फ़ॉलबैक व्यवहार के लिए [मॉडल फ़ेलओवर](/hi/concepts/model-failover) देखें।
    - कस्टम/स्वयं-होस्ट किए गए प्रदाताओं के लिए, संदर्भ में [कस्टम प्रदाता](/hi/gateway/config-tools#custom-providers-and-base-urls) देखें।

  </Accordion>

  <Accordion title="नियंत्रित करें कि बॉट को कौन संदेश भेज सकता है">
    DM पहुँच को प्रति चैनल `dmPolicy` (डिफ़ॉल्ट `"pairing"`) के माध्यम से नियंत्रित किया जाता है:

    - `"pairing"`: अज्ञात प्रेषकों को स्वीकृति के लिए एक बार उपयोग होने वाला पेयरिंग कोड मिलता है
    - `"allowlist"`: केवल `allowFrom` (या पेयर्ड अनुमति स्टोर) में मौजूद प्रेषक
    - `"open"`: सभी इनबाउंड DM की अनुमति दें (`allowFrom: ["*"]` आवश्यक है)
    - `"disabled"`: सभी DM को अनदेखा करें

    समूहों के लिए, `groupPolicy` (`"allowlist" | "open" | "disabled"`) के साथ `groupAllowFrom` या चैनल-विशिष्ट अनुमति-सूचियों का उपयोग करें।

    प्रति-चैनल विवरण के लिए [पूरा संदर्भ](/hi/gateway/config-channels#dm-and-group-access) देखें।

  </Accordion>

  <Accordion title="समूह चैट उल्लेख गेटिंग सेट अप करें">
    समूह संदेशों में डिफ़ॉल्ट रूप से **उल्लेख आवश्यक** होता है। प्रति एजेंट ट्रिगर पैटर्न कॉन्फ़िगर करें। सामान्य समूह/चैनल उत्तर स्वचालित रूप से पोस्ट होते हैं; उन साझा कक्षों के लिए संदेश-टूल पथ चुनें जहाँ एजेंट को तय करना चाहिए कि कब बोलना है:

    ```json5
    {
      messages: {
        visibleReplies: "automatic", // हर जगह संदेश-टूल से भेजना आवश्यक करने के लिए "message_tool" सेट करें
        groupChat: {
          visibleReplies: "message_tool", // वैकल्पिक; दृश्यमान आउटपुट के लिए message(action=send) आवश्यक है
          unmentionedInbound: "room_event", // बिना उल्लेख वाली हमेशा चालू समूह बातचीत शांत संदर्भ है
        },
      },
      agents: {
        list: [
          {
            id: "main",
            groupChat: {
              mentionPatterns: ["@openclaw", "openclaw"],
            },
          },
        ],
      },
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```

    - **मेटाडेटा उल्लेख**: मूल @-उल्लेख (WhatsApp टैप-टू-मेंशन, Telegram @bot आदि)
    - **टेक्स्ट पैटर्न**: `mentionPatterns` में सुरक्षित रेगेक्स पैटर्न
    - **दृश्यमान उत्तर**: `messages.visibleReplies` विश्व स्तर पर संदेश-टूल से भेजना आवश्यक कर सकता है; `messages.groupChat.visibleReplies` समूहों/चैनलों के लिए इसे ओवरराइड करता है।
    - दृश्यमान उत्तर मोड, प्रति-चैनल ओवरराइड और स्वयं-चैट मोड के लिए [पूरा संदर्भ](/hi/gateway/config-channels#group-chat-mention-gating) देखें।

  </Accordion>

  <Accordion title="प्रति एजेंट Skills प्रतिबंधित करें">
    साझा आधाररेखा के लिए `agents.defaults.skills` का उपयोग करें, फिर विशिष्ट
    एजेंटों को `agents.entries.*.skills` से ओवरराइड करें:

    ```json5
    {
      agents: {
        defaults: {
          skills: ["github", "weather"],
        },
        list: [
          { id: "writer" }, // github, weather प्राप्त करता है
          { id: "docs", skills: ["docs-search"] }, // डिफ़ॉल्ट को बदलता है
          { id: "locked-down", skills: [] }, // कोई Skills नहीं
        ],
      },
    }
    ```

    - डिफ़ॉल्ट रूप से अप्रतिबंधित Skills के लिए `agents.defaults.skills` को छोड़ दें।
    - डिफ़ॉल्ट प्राप्त करने के लिए `agents.entries.*.skills` को छोड़ दें।
    - कोई Skills न रखने के लिए `agents.entries.*.skills: []` सेट करें।
    - [Skills](/hi/tools/skills), [Skills कॉन्फ़िग](/hi/tools/skills-config), और
      [कॉन्फ़िगरेशन संदर्भ](/hi/gateway/config-agents#agents-defaults-skills) देखें।

  </Accordion>

  <Accordion title="प्रति-चैनल स्वास्थ्य निगरानी कॉन्फ़िगर करें">
    किसी चैनल या खाते के लिए स्वचालित स्वास्थ्य पुनरारंभ अक्षम या सक्षम करें:

    ```json5
    {
      channels: {
        telegram: {
          healthMonitor: { enabled: false },
          accounts: {
            alerts: {
              healthMonitor: { enabled: true },
            },
          },
        },
      },
    }
    ```

    - किसी एक चैनल या खाते के लिए स्वचालित पुनरारंभ नियंत्रित करने हेतु `channels.<provider>.healthMonitor.enabled` या `channels.<provider>.accounts.<id>.healthMonitor.enabled` का उपयोग करें।
    - परिचालन डीबगिंग के लिए [स्वास्थ्य जाँच](/hi/gateway/health) और सभी फ़ील्ड के लिए [पूरा संदर्भ](/hi/gateway/configuration-reference#gateway) देखें।

  </Accordion>

  <Accordion title="सेशन और रीसेट कॉन्फ़िगर करें">
    सेशन वार्तालाप की निरंतरता और पृथक्करण नियंत्रित करते हैं:

    ```json5
    {
      session: {
        dmScope: "per-channel-peer",  // बहु-उपयोगकर्ता के लिए अनुशंसित
        threadBindings: {
          enabled: true,
          idleHours: 24,
          maxAgeHours: 0,
        },
        reset: {
          mode: "daily",
          atHour: 4,
          idleMinutes: 120,
        },
      },
    }
    ```

    - `dmScope`: `main` (साझा) | `per-peer` | `per-channel-peer` | `per-account-channel-peer`
    - `threadBindings`: थ्रेड से बंधी सेशन रूटिंग के वैश्विक डिफ़ॉल्ट। `/focus`, `/unfocus`, `/agents`, `/session idle`, और `/session max-age` प्रत्येक सेशन के लिए इसे बाँधते, अनबाइंड करते, सूचीबद्ध करते और समायोजित करते हैं (Discord थ्रेड बाँधता है, Telegram विषय/वार्तालाप बाँधता है)।
    - स्कोपिंग, पहचान लिंक और प्रेषण नीति के लिए [सेशन प्रबंधन](/hi/concepts/session) देखें।
    - सभी फ़ील्ड के लिए [पूर्ण संदर्भ](/hi/gateway/config-agents#session) देखें।

  </Accordion>

  <Accordion title="सैंडबॉक्सिंग सक्षम करें">
    एजेंट सेशन को पृथक सैंडबॉक्स रनटाइम में चलाएँ:

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",  // off | non-main | all
            scope: "agent",    // session | agent | shared
          },
        },
      },
    }
    ```

    पहले इमेज बनाएँ—स्रोत चेकआउट से `scripts/sandbox-setup.sh` चलाएँ, या npm इंस्टॉल से [सैंडबॉक्सिंग § इमेज और सेटअप](/hi/gateway/sandboxing#images-and-setup) में इनलाइन `docker build` कमांड देखें।

    पूरी मार्गदर्शिका के लिए [सैंडबॉक्सिंग](/hi/gateway/sandboxing) और सभी विकल्पों के लिए [पूर्ण संदर्भ](/hi/gateway/config-agents#agentsdefaultssandbox) देखें।

  </Accordion>

  <Accordion title="आधिकारिक iOS बिल्ड के लिए रिले-समर्थित पुश सक्षम करें">
    सार्वजनिक App Store बिल्ड के लिए रिले-समर्थित पुश होस्ट किए गए OpenClaw रिले का उपयोग करता है: `https://ios-push-relay.openclaw.ai`।

    कस्टम रिले परिनियोजन के लिए जानबूझकर अलग iOS बिल्ड/परिनियोजन पथ आवश्यक है, जिसका रिले URL Gateway के रिले URL से मेल खाता हो। यदि आप कस्टम रिले बिल्ड का उपयोग कर रहे हैं, तो Gateway कॉन्फ़िगरेशन में इसे सेट करें:

    ```json5
    {
      gateway: {
        push: {
          apns: {
            relay: {
              baseUrl: "https://relay.example.com",
              // वैकल्पिक। डिफ़ॉल्ट: 10000
              timeoutMs: 10000,
            },
          },
        },
      },
    }
    ```

    समकक्ष CLI:

    ```bash
    openclaw config set gateway.push.apns.relay.baseUrl https://relay.example.com
    ```

    यह क्या करता है:

    - Gateway को बाहरी रिले के माध्यम से `push.test`, सक्रिय करने के संकेत और पुनः कनेक्ट करने के संकेत भेजने देता है।
    - युग्मित iOS ऐप द्वारा अग्रेषित, पंजीकरण-स्कोप वाले प्रेषण अनुदान का उपयोग करता है। Gateway को पूरे परिनियोजन के लिए रिले टोकन की आवश्यकता नहीं होती।
    - प्रत्येक रिले-समर्थित पंजीकरण को उस Gateway पहचान से बाँधता है जिसके साथ iOS ऐप युग्मित हुआ था, ताकि कोई अन्य Gateway संग्रहीत पंजीकरण का पुनः उपयोग न कर सके।
    - स्थानीय/मैन्युअल iOS बिल्ड को प्रत्यक्ष APNs पर रखता है। रिले-समर्थित प्रेषण केवल रिले के माध्यम से पंजीकृत आधिकारिक वितरित बिल्ड पर लागू होते हैं।
    - यह iOS बिल्ड में अंतर्निहित रिले बेस URL से मेल खाना चाहिए, ताकि पंजीकरण और प्रेषण ट्रैफ़िक समान रिले परिनियोजन तक पहुँचे।

    आरंभ-से-अंत प्रवाह:

    1. आधिकारिक iOS ऐप इंस्टॉल करें।
    2. वैकल्पिक: केवल जानबूझकर अलग कस्टम रिले बिल्ड का उपयोग करते समय Gateway पर `gateway.push.apns.relay.baseUrl` कॉन्फ़िगर करें।
    3. iOS ऐप को Gateway से युग्मित करें और Node तथा ऑपरेटर, दोनों सेशन को कनेक्ट होने दें।
    4. iOS ऐप Gateway पहचान प्राप्त करता है, App Attest और ऐप रसीद का उपयोग करके रिले के साथ पंजीकरण करता है, फिर रिले-समर्थित `push.apns.register` पेलोड को युग्मित Gateway पर प्रकाशित करता है।
    5. Gateway रिले हैंडल और प्रेषण अनुदान संग्रहीत करता है, फिर `push.test`, सक्रिय करने के संकेत और पुनः कनेक्ट करने के संकेत के लिए उनका उपयोग करता है।

    संचालन संबंधी टिप्पणियाँ:

    - यदि आप iOS ऐप को किसी अलग Gateway पर स्विच करते हैं, तो ऐप को पुनः कनेक्ट करें ताकि वह उस Gateway से बंधा नया रिले पंजीकरण प्रकाशित कर सके।
    - यदि आप किसी अलग रिले परिनियोजन की ओर संकेत करने वाला नया iOS बिल्ड जारी करते हैं, तो ऐप पुराने रिले मूल का पुनः उपयोग करने के बजाय अपने कैश किए गए रिले पंजीकरण को रीफ़्रेश करता है।

    संगतता टिप्पणी:

    - `OPENCLAW_APNS_RELAY_BASE_URL` और `OPENCLAW_APNS_RELAY_TIMEOUT_MS` अभी भी अस्थायी एनवायरनमेंट ओवरराइड के रूप में काम करते हैं।
    - कस्टम Gateway रिले URL, iOS बिल्ड में अंतर्निहित रिले बेस URL से मेल खाने चाहिए; सार्वजनिक App Store रिलीज़ लेन कस्टम iOS रिले URL ओवरराइड अस्वीकार करती है।
    - `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true` केवल लूपबैक वाला विकास अपवाद बना रहता है; HTTP रिले URL को कॉन्फ़िगरेशन में स्थायी रूप से संग्रहीत न करें।

    आरंभ-से-अंत प्रवाह के लिए [iOS ऐप](/hi/platforms/ios#relay-backed-push-for-official-builds) और रिले सुरक्षा मॉडल के लिए [प्रमाणीकरण और विश्वास प्रवाह](/hi/platforms/ios#authentication-and-trust-flow) देखें।

  </Accordion>

  <Accordion title="Heartbeat सेट अप करें (आवधिक चेक-इन)">
    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "30m",
            target: "last",
          },
        },
      },
    }
    ```

    - `every`: अवधि स्ट्रिंग (`30m`, `2h`)। अक्षम करने के लिए `0m` सेट करें। डिफ़ॉल्ट: `30m`।
    - `target`: `last` | `none` | `<channel-id>` (उदाहरण के लिए `discord`, `matrix`, `telegram`, या `whatsapp`)
    - `directPolicy`: DM-शैली Heartbeat लक्ष्यों के लिए `allow` (डिफ़ॉल्ट) या `block`
    - पूरी मार्गदर्शिका के लिए [Heartbeat](/hi/gateway/heartbeat) देखें।

  </Accordion>

  <Accordion title="Cron जॉब कॉन्फ़िगर करें">
    ```json5
    {
      cron: {
        enabled: true,
        sessionRetention: "24h",
      },
    }
    ```

    - `sessionRetention`: SQLite सेशन पंक्तियों से पूर्ण हो चुके पृथक रन सेशन हटाएँ (डिफ़ॉल्ट `24h`; अक्षम करने के लिए `false` सेट करें)।
    - रन इतिहास स्वचालित रूप से प्रत्येक जॉब की नवीनतम 2000 टर्मिनल पंक्तियाँ रखता है; खोई हुई पंक्तियाँ अपनी 24-घंटे की सफ़ाई अवधि बनाए रखती हैं।
    - सुविधा अवलोकन और CLI उदाहरणों के लिए [Cron जॉब](/hi/automation/cron-jobs) देखें।

  </Accordion>

  <Accordion title="Webhook सेट अप करें (हुक)">
    Gateway पर HTTP Webhook एंडपॉइंट सक्षम करें:

    ```json5
    {
      hooks: {
        enabled: true,
        token: "shared-secret",
        path: "/hooks",
        defaultSessionKey: "hook:ingress",
        allowRequestSessionKey: false,
        allowedSessionKeyPrefixes: ["hook:"],
        mappings: [
          {
            match: { path: "gmail" },
            action: "agent",
            agentId: "main",
            deliver: true,
          },
        ],
      },
    }
    ```

    सुरक्षा टिप्पणी:
    - सभी हुक/Webhook पेलोड सामग्री को अविश्वसनीय इनपुट मानें।
    - एक समर्पित `hooks.token` का उपयोग करें; सक्रिय Gateway प्रमाणीकरण सीक्रेट (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` या `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`) का पुनः उपयोग न करें।
    - हुक प्रमाणीकरण केवल हेडर-आधारित है (`Authorization: Bearer ...` या `x-openclaw-token`); क्वेरी-स्ट्रिंग टोकन अस्वीकार किए जाते हैं।
    - `hooks.path`, `/` नहीं हो सकता; Webhook प्रवेश को `/hooks` जैसे समर्पित उपपथ पर रखें।
    - असुरक्षित-सामग्री बायपास फ़्लैग (`hooks.gmail.allowUnsafeExternalContent`, `hooks.mappings[].allowUnsafeExternalContent`) अक्षम रखें, जब तक कि अत्यंत सीमित दायरे में डीबगिंग न की जा रही हो।
    - यदि आप `hooks.allowRequestSessionKey` सक्षम करते हैं, तो कॉलर द्वारा चुनी गई सेशन कुंजियों को सीमित करने के लिए `hooks.allowedSessionKeyPrefixes` भी सेट करें।
    - हुक-संचालित एजेंट के लिए, सशक्त आधुनिक मॉडल टियर और सख्त टूल नीति को प्राथमिकता दें (उदाहरण के लिए, केवल संदेश-प्रेषण और जहाँ संभव हो वहाँ सैंडबॉक्सिंग)।

    सभी मैपिंग विकल्पों और Gmail एकीकरण के लिए [पूर्ण संदर्भ](/hi/gateway/configuration-reference#hooks) देखें।

  </Accordion>

  <Accordion title="बहु-एजेंट रूटिंग कॉन्फ़िगर करें">
    अलग-अलग वर्कस्पेस और सेशन वाले कई पृथक एजेंट चलाएँ:

    ```json5
    {
      agents: {
        list: [
          { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
          { id: "work", workspace: "~/.openclaw/workspace-work" },
        ],
      },
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
      ],
    }
    ```

    बाइंडिंग नियमों और प्रति-एजेंट एक्सेस प्रोफ़ाइल के लिए [बहु-एजेंट](/hi/concepts/multi-agent) और [पूर्ण संदर्भ](/hi/gateway/config-agents#multi-agent-routing) देखें।

  </Accordion>

  <Accordion title="कॉन्फ़िगरेशन को कई फ़ाइलों में विभाजित करें ($include)">
    बड़े कॉन्फ़िगरेशन व्यवस्थित करने के लिए `$include` का उपयोग करें:

    ```json5
    // ~/.openclaw/openclaw.json
    {
      gateway: { port: 18789 },
      agents: { $include: "./agents.json5" },
      broadcast: {
        $include: ["./clients/a.json5", "./clients/b.json5"],
      },
    }
    ```

    - **एकल फ़ाइल**: समाहित करने वाले ऑब्जेक्ट को प्रतिस्थापित करती है
    - **फ़ाइलों की सरणी**: क्रम से गहराई तक मर्ज होती है (बाद वाली प्रभावी होती है), अधिकतम 10 नेस्टेड स्तर तक
    - **सिबलिंग कुंजियाँ**: इन्क्लूड के बाद मर्ज होती हैं (शामिल मानों को ओवरराइड करती हैं)
    - **सापेक्ष पथ**: इन्क्लूड करने वाली फ़ाइल के सापेक्ष हल किए जाते हैं
    - **पथ प्रारूप**: इन्क्लूड पथों में नल बाइट नहीं होने चाहिए और समाधान से पहले तथा बाद में उनकी लंबाई 4096 वर्णों से पूर्णतः कम होनी चाहिए
    - **OpenClaw के स्वामित्व वाले लेखन**: जब कोई लेखन केवल एक शीर्ष-स्तरीय अनुभाग बदलता है
      जो `plugins: { $include: "./plugins.json5" }` जैसे एकल-फ़ाइल इन्क्लूड द्वारा समर्थित हो,
      तो OpenClaw उस शामिल फ़ाइल को अपडेट करता है और `openclaw.json` को यथावत रखता है
    - **असमर्थित राइट-थ्रू**: रूट इन्क्लूड, इन्क्लूड सरणियाँ और सिबलिंग ओवरराइड वाले इन्क्लूड
      कॉन्फ़िगरेशन को समतल करने के बजाय OpenClaw के स्वामित्व वाले लेखन के लिए
      सुरक्षित रूप से विफल होते हैं
    - **परिसीमन**: `$include` पथों को उस डायरेक्टरी के अंतर्गत हल होना चाहिए जिसमें
      `openclaw.json` स्थित है। कई मशीनों या उपयोगकर्ताओं के बीच कोई ट्री साझा करने के लिए,
      `OPENCLAW_INCLUDE_ROOTS` को अतिरिक्त डायरेक्टरियों की पथ-सूची (`:` POSIX पर,
      `;` Windows पर) पर सेट करें, जिन्हें इन्क्लूड संदर्भित कर सकते हैं। सिमलिंक हल किए
      जाते हैं और पुनः जाँचे जाते हैं, इसलिए ऐसा पथ जो शाब्दिक रूप से कॉन्फ़िगरेशन डायरेक्टरी में हो,
      लेकिन जिसका वास्तविक लक्ष्य हर अनुमत रूट से बाहर निकलता हो, तब भी अस्वीकार किया जाता है।
    - **त्रुटि प्रबंधन**: अनुपलब्ध फ़ाइलों, पार्स त्रुटियों, चक्रीय इन्क्लूड, अमान्य पथ प्रारूप और अत्यधिक लंबाई के लिए स्पष्ट त्रुटियाँ

  </Accordion>
</AccordionGroup>

## कॉन्फ़िगरेशन हॉट रीलोड

Gateway `~/.openclaw/openclaw.json` पर नज़र रखता है और परिवर्तनों को स्वचालित रूप से लागू करता है—अधिकांश सेटिंग के लिए मैन्युअल रीस्टार्ट की आवश्यकता नहीं होती।

प्रत्यक्ष फ़ाइल संपादन को सत्यापन होने तक अविश्वसनीय माना जाता है। वॉचर एडिटर की अस्थायी-लेखन/नाम-बदलने की गतिविधि के स्थिर होने की प्रतीक्षा करता है, अंतिम फ़ाइल पढ़ता है और `openclaw.json` को दोबारा लिखे बिना अमान्य बाहरी संपादन अस्वीकार करता है। OpenClaw के स्वामित्व वाले कॉन्फ़िगरेशन लेखन लिखने से पहले उसी स्कीमा गेट का उपयोग करते हैं (प्रत्येक लेखन पर लागू क्लॉबर/रोलबैक नियमों के लिए [सख्त सत्यापन](#strict-validation) देखें)।

यदि आपको `config reload skipped (invalid config)` दिखाई देता है या स्टार्टअप `Invalid
config` रिपोर्ट करता है, तो कॉन्फ़िगरेशन का निरीक्षण करें, `openclaw config validate` चलाएँ, फिर सुधार के लिए `openclaw
doctor --fix` चलाएँ। चेकलिस्ट के लिए [Gateway समस्या निवारण](/hi/gateway/troubleshooting#gateway-rejected-invalid-config) देखें।

### रीलोड मोड

| मोड                   | व्यवहार                                                                                |
| ---------------------- | --------------------------------------------------------------------------------------- |
| **`hybrid`** (डिफ़ॉल्ट) | सुरक्षित बदलाव तुरंत हॉट-अप्लाई करता है। महत्वपूर्ण बदलावों के लिए अपने-आप रीस्टार्ट करता है।           |
| **`hot`**              | केवल सुरक्षित बदलाव हॉट-अप्लाई करता है। रीस्टार्ट की आवश्यकता होने पर चेतावनी लॉग करता है—इसे आप संभालते हैं। |
| **`restart`**          | किसी भी कॉन्फ़िग बदलाव पर Gateway को रीस्टार्ट करता है, चाहे वह सुरक्षित हो या नहीं।                                 |
| **`off`**              | फ़ाइल निगरानी अक्षम करता है। बदलाव अगले मैन्युअल रीस्टार्ट पर प्रभावी होते हैं।                 |

```json5
{
  gateway: {
    reload: { mode: "hybrid", debounceMs: 300 },
  },
}
```

### क्या हॉट-अप्लाई होता है और किसके लिए रीस्टार्ट आवश्यक है

अधिकांश फ़ील्ड बिना डाउनटाइम के हॉट-अप्लाई होते हैं; कुछ हॉट-अप्लाई किए गए अनुभाग पूरे Gateway के बजाय केवल संबंधित
सब-सिस्टम (चैनल, Cron, Heartbeat, स्वास्थ्य मॉनिटर) को रीस्टार्ट करते हैं। 
`hybrid` मोड में, Gateway रीस्टार्ट की आवश्यकता वाले बदलाव अपने-आप संभाले जाते हैं।

| श्रेणी            | फ़ील्ड                                                                  | क्या Gateway रीस्टार्ट आवश्यक है?      |
| ------------------- | ----------------------------------------------------------------------- | ---------------------------- |
| चैनल            | `channels.*`, `web` (WhatsApp)—सभी बिल्ट-इन और Plugin चैनल       | नहीं (उस चैनल को रीस्टार्ट करता है)   |
| एजेंट और मॉडल      | `agent`, `agents`, `models`, `routing`                                  | नहीं                           |
| ऑटोमेशन          | `hooks`, `cron`, `agent.heartbeat`                                      | नहीं (उस सब-सिस्टम को रीस्टार्ट करता है) |
| सेशन और संदेश | `session`, `messages`                                                   | नहीं                           |
| टूल और मीडिया       | `tools`, `skills`, `mcp`, `audio`, `talk`                               | नहीं                           |
| Plugin कॉन्फ़िग       | `plugins.entries.*`, `plugins.allow`, `plugins.deny`, `plugins.enabled` | नहीं (Plugin रनटाइम को रीलोड करता है)  |
| UI और विविध           | `ui`, `logging`, `identity`, `bindings`                                 | नहीं                           |
| Gateway सर्वर      | `gateway.*` (पोर्ट, बाइंड, प्रमाणीकरण, Tailscale, TLS, HTTP, पुश)              | **हाँ**                      |
| इंफ़्रास्ट्रक्चर      | `discovery`, `browser`, `plugins.load`, `plugins.installs`              | **हाँ**                      |

<Note>
`gateway.reload` और `gateway.remote`, `gateway.*` के अंतर्गत अपवाद हैं—इन्हें बदलने से रीस्टार्ट **ट्रिगर नहीं** होता। अलग-अलग Plugin भी इस तालिका को ओवरराइड कर सकते हैं: लोड किया गया Plugin रीस्टार्ट ट्रिगर करने वाले अपने कॉन्फ़िग प्रीफ़िक्स घोषित कर सकता है (उदाहरण के लिए, बंडल किया गया Canvas Plugin केवल अपने `plugins.entries.canvas` के लिए ही नहीं, बल्कि `plugins.enabled`, `plugins.allow`, और `plugins.deny` के लिए भी Gateway को रीस्टार्ट करता है), इसलिए वास्तविक व्यवहार इस बात पर निर्भर करता है कि कौन-से Plugin सक्रिय हैं।
</Note>

### रीलोड की योजना बनाना

जब आप `$include` के माध्यम से संदर्भित किसी स्रोत फ़ाइल को संपादित करते हैं, तो OpenClaw
फ़्लैट किए गए इन-मेमोरी दृश्य के बजाय स्रोत में लिखे गए लेआउट से रीलोड की योजना बनाता है।
इससे हॉट-रीलोड के निर्णय (हॉट-अप्लाई बनाम रीस्टार्ट) पूर्वानुमेय रहते हैं, भले ही
कोई एक शीर्ष-स्तरीय अनुभाग अपनी अलग शामिल फ़ाइल में हो, जैसे
`plugins: { $include: "./plugins.json5" }`। स्रोत लेआउट अस्पष्ट होने पर रीलोड योजना
सुरक्षित रूप से विफल हो जाती है।

## कॉन्फ़िग RPC (प्रोग्रामेटिक अपडेट)

Gateway API के माध्यम से कॉन्फ़िग लिखने वाले टूल के लिए, इस प्रवाह को प्राथमिकता दें:

- `config.schema.lookup` से एक सबट्री का निरीक्षण करें (उथला स्कीमा Node + चाइल्ड
  सारांश)
- `config.get` से मौजूदा स्नैपशॉट और `hash` प्राप्त करें
- `config.patch` का उपयोग आंशिक अपडेट के लिए करें (JSON मर्ज पैच: ऑब्जेक्ट मर्ज होते हैं, `null`
  हटाता है, और यदि प्रविष्टियाँ हटेंगी तो `replacePaths` से स्पष्ट पुष्टि किए जाने पर
  ऐरे प्रतिस्थापित होते हैं)
- `config.apply` का उपयोग केवल तब करें जब आपका उद्देश्य पूरे कॉन्फ़िग को प्रतिस्थापित करना हो
- `update.run` का उपयोग स्पष्ट सेल्फ़-अपडेट और रीस्टार्ट के लिए करें; यदि रीस्टार्ट के बाद सेशन को एक फ़ॉलो-अप टर्न चलाना चाहिए, तो `continuationMessage` शामिल करें
- `update.status` से नवीनतम अपडेट रीस्टार्ट सेंटिनल का निरीक्षण करें और रीस्टार्ट के बाद चल रहे संस्करण को सत्यापित करें

एजेंटों को फ़ील्ड-स्तरीय सटीक दस्तावेज़ और प्रतिबंधों के लिए `config.schema.lookup` को पहला पड़ाव मानना चाहिए।
जब उन्हें विस्तृत कॉन्फ़िग मानचित्र, डिफ़ॉल्ट, या समर्पित
सब-सिस्टम संदर्भों के लिंक चाहिए हों, तो [कॉन्फ़िगरेशन संदर्भ](/hi/gateway/configuration-reference)
का उपयोग करें।

<Note>
कंट्रोल-प्लेन राइट (`config.apply`, `config.patch`, `update.run`) पर
प्रति विधि, प्रति `deviceId+clientIp`, 60 सेकंड में 30 अनुरोधों की
दर-सीमा लागू होती है; [दर सीमित करना](/hi/gateway/security/rate-limiting) देखें। रीस्टार्ट
अनुरोध एक साथ समेकित होते हैं और फिर रीस्टार्ट चक्रों के बीच 30-सेकंड का कूलडाउन लागू करते हैं।
`update.status` केवल-पढ़ने योग्य है, लेकिन एडमिन-स्कोप्ड है क्योंकि रीस्टार्ट सेंटिनल में
अपडेट चरण सारांश और कमांड आउटपुट के अंतिम हिस्से शामिल हो सकते हैं।
</Note>

आंशिक पैच का उदाहरण:

```bash
openclaw gateway call config.get --params '{}'  # capture payload.hash
openclaw gateway call config.patch --params '{
  "raw": "{ channels: { telegram: { groups: { \"*\": { requireMention: false } } } } }",
  "baseHash": "<hash>"
}'
```

`config.apply` और `config.patch` दोनों `raw`, `baseHash`, `sessionKey`,
`note`, और `restartDelayMs` स्वीकार करते हैं। कॉन्फ़िग फ़ाइल पहले से मौजूद होने पर दोनों विधियों के लिए
`baseHash` आवश्यक है (कोई मौजूदा कॉन्फ़िग न होने पर पहली राइट इस जाँच को छोड़ देती है)।

`config.patch`, `replacePaths` भी स्वीकार करता है, जो उन कॉन्फ़िग पथों का ऐरे है जिनका ऐरे
प्रतिस्थापन जानबूझकर किया गया है। यदि कोई पैच किसी मौजूदा ऐरे को कम प्रविष्टियों वाले ऐरे से
प्रतिस्थापित या हटाएगा, तो Gateway उस राइट को अस्वीकार कर देता है, जब तक वह सटीक पथ
`replacePaths` में मौजूद न हो; ऐरे प्रविष्टियों के अंतर्गत नेस्टेड ऐरे `[]` का उपयोग करते हैं, जैसे
`agents.entries.*.skills`। इससे संक्षिप्त किए गए `config.get` स्नैपशॉट
रूटिंग या अलाउलिस्ट ऐरे को चुपचाप ओवरराइट नहीं कर पाते। जब आपका उद्देश्य पूरा कॉन्फ़िग प्रतिस्थापित करना हो,
तो `config.apply` का उपयोग करें।

## एनवायरनमेंट वेरिएबल

OpenClaw पैरेंट प्रोसेस के साथ-साथ निम्न स्थानों से env var पढ़ता है:

- वर्तमान कार्यशील डायरेक्टरी से `.env` (यदि मौजूद हो)
- `~/.openclaw/.env` (ग्लोबल फ़ॉलबैक)

कोई भी फ़ाइल मौजूदा env var को ओवरराइड नहीं करती। आप कॉन्फ़िग में इनलाइन env var भी सेट कर सकते हैं:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
  },
}
```

<Accordion title="शेल env आयात (वैकल्पिक)">
  यदि सक्षम है और अपेक्षित कुंजियाँ सेट नहीं हैं, तो OpenClaw आपका लॉगिन शेल चलाता है और केवल अनुपलब्ध कुंजियाँ आयात करता है:

```json5
{
  env: {
    shellEnv: { enabled: true, timeoutMs: 15000 },
  },
}
```

समतुल्य env var: `OPENCLAW_LOAD_SHELL_ENV=1`। डिफ़ॉल्ट `timeoutMs`: `15000`।
</Accordion>

<Accordion title="कॉन्फ़िग मानों में env var प्रतिस्थापन">
  किसी भी कॉन्फ़िग स्ट्रिंग मान में `${VAR_NAME}` से env var का संदर्भ दें:

```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } },
  models: { providers: { custom: { apiKey: "${CUSTOM_API_KEY}" } } },
}
```

नियम:

- केवल अपरकेस नामों का मिलान होता है: `[A-Z_][A-Z0-9_]*`
- अनुपलब्ध/रिक्त var लोड के समय त्रुटि उत्पन्न करते हैं
- लिटरल आउटपुट के लिए `$${VAR}` से एस्केप करें
- `$include` फ़ाइलों के भीतर काम करता है
- इनलाइन प्रतिस्थापन: `"${BASE}/v1"` → `"https://api.example.com/v1"`

</Accordion>

<Accordion title="सीक्रेट रेफ़रेंस (env, फ़ाइल, exec)">
  SecretRef ऑब्जेक्ट का समर्थन करने वाले फ़ील्ड के लिए, आप इसका उपयोग कर सकते हैं:

```json5
{
  models: {
    providers: {
      openai: { apiKey: { source: "env", provider: "default", id: "OPENAI_API_KEY" } },
    },
  },
  skills: {
    entries: {
      "image-lab": {
        apiKey: {
          source: "file",
          provider: "filemain",
          id: "/skills/entries/image-lab/apiKey",
        },
      },
    },
  },
  channels: {
    googlechat: {
      serviceAccount: {
        source: "exec",
        provider: "vault",
        id: "channels/googlechat/serviceAccount",
      },
    },
  },
}
```

SecretRef का विवरण (`env`/`file`/`exec` के लिए `secrets.providers` सहित) [सीक्रेट प्रबंधन](/hi/gateway/secrets) में उपलब्ध है।
समर्थित क्रेडेंशियल पथ [SecretRef क्रेडेंशियल सरफ़ेस](/hi/reference/secretref-credential-surface) में सूचीबद्ध हैं।
</Accordion>

पूर्ण प्राथमिकता क्रम और स्रोतों के लिए [एनवायरनमेंट](/hi/help/environment) देखें।

## पूर्ण संदर्भ

फ़ील्ड-दर-फ़ील्ड संपूर्ण संदर्भ के लिए, **[कॉन्फ़िगरेशन संदर्भ](/hi/gateway/configuration-reference)** देखें।

---

_संबंधित: [कॉन्फ़िगरेशन उदाहरण](/hi/gateway/configuration-examples) · [कॉन्फ़िगरेशन संदर्भ](/hi/gateway/configuration-reference) · [Doctor](/hi/gateway/doctor)_

## संबंधित

- [कॉन्फ़िगरेशन संदर्भ](/hi/gateway/configuration-reference)
- [कॉन्फ़िगरेशन उदाहरण](/hi/gateway/configuration-examples)
- [Gateway रनबुक](/hi/gateway)
