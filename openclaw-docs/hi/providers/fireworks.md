---
read_when:
    - आप OpenClaw के साथ Fireworks का उपयोग करना चाहते हैं
    - आपको Fireworks API कुंजी के एनवायरनमेंट वेरिएबल या डिफ़ॉल्ट मॉडल आईडी की आवश्यकता है
    - आप Fireworks पर Kimi के thinking-off व्यवहार को डीबग कर रहे हैं
summary: Fireworks सेटअप (प्रमाणीकरण + मॉडल चयन)
title: Fireworks
x-i18n:
    generated_at: "2026-07-27T18:53:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7720b23b69aa716d2e2903f5644bb74f81ca1c5e753f71d72d4d7a25c0747884
    source_path: providers/fireworks.md
    workflow: 16
---

[Fireworks](https://fireworks.ai) OpenAI-संगत API के माध्यम से ओपन-वेट और रूट किए गए मॉडल उपलब्ध कराता है। पहले से कैटलॉग किए गए दो Kimi मॉडल और रनटाइम पर किसी भी Fireworks मॉडल या राउटर आईडी का उपयोग करने के लिए आधिकारिक Fireworks प्रोवाइडर Plugin इंस्टॉल करें।

| प्रॉपर्टी       | मान                                                    |
| --------------- | ------------------------------------------------------ |
| प्रोवाइडर आईडी  | `fireworks` (उपनाम: `fireworks-ai`)                    |
| पैकेज           | `@openclaw/fireworks-provider`                         |
| प्रमाणीकरण एन्वायरनमेंट वेरिएबल | `FIREWORKS_API_KEY`                                    |
| ऑनबोर्डिंग फ़्लैग | `--auth-choice fireworks-api-key`                      |
| प्रत्यक्ष CLI फ़्लैग | `--fireworks-api-key <key>`                            |
| API             | OpenAI-संगत (`openai-completions`)               |
| बेस URL         | `https://api.fireworks.ai/inference/v1`                |
| डिफ़ॉल्ट मॉडल   | `fireworks/accounts/fireworks/routers/kimi-k2p6-turbo` |
| डिफ़ॉल्ट उपनाम  | `Kimi K2.6 Turbo`                                      |

## शुरुआत करना

<Steps>
  <Step title="Plugin इंस्टॉल करें">
    ```bash
    openclaw plugins install @openclaw/fireworks-provider
    ```
  </Step>
  <Step title="Fireworks API कुंजी सेट करें">
    <CodeGroup>

```bash ऑनबोर्डिंग
openclaw onboard --auth-choice fireworks-api-key
```

```bash प्रत्यक्ष फ़्लैग
openclaw onboard --non-interactive \
  --auth-choice fireworks-api-key \
  --fireworks-api-key "$FIREWORKS_API_KEY"
```

```bash केवल एन्वायरनमेंट
export FIREWORKS_API_KEY=fw-...
```

    </CodeGroup>

    ऑनबोर्डिंग कुंजी को आपकी प्रमाणीकरण प्रोफ़ाइलों में `fireworks` प्रोवाइडर से संबद्ध करके संग्रहीत करती है और **Fire Pass** Kimi K2.6 Turbo राउटर को डिफ़ॉल्ट मॉडल के रूप में सेट करती है।

  </Step>
  <Step title="सत्यापित करें कि मॉडल उपलब्ध है">
    ```bash
    openclaw models list --provider fireworks
    ```

    सूची में `Kimi K2.6` और `Kimi K2.6 Turbo (Fire Pass)` शामिल होने चाहिए। यदि `FIREWORKS_API_KEY` का समाधान नहीं होता है, तो `openclaw models status --json`, `auth.unusableProfiles` के अंतर्गत अनुपलब्ध क्रेडेंशियल की सूचना देता है।

  </Step>
</Steps>

## गैर-इंटरैक्टिव सेटअप

स्क्रिप्टेड या CI इंस्टॉलेशन के लिए, सभी चीज़ें कमांड लाइन पर पास करें:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice fireworks-api-key \
  --fireworks-api-key "$FIREWORKS_API_KEY" \
  --skip-health \
  --accept-risk
```

## अंतर्निहित कैटलॉग

| मॉडल संदर्भ                                           | नाम                         | इनपुट       | कॉन्टेक्स्ट | अधिकतम आउटपुट | थिंकिंग                 |
| ------------------------------------------------------ | --------------------------- | ------------ | ------- | ---------- | -------------------- |
| `fireworks/accounts/fireworks/models/kimi-k2p6`        | Kimi K2.6                   | टेक्स्ट + इमेज | 262,144 | 262,144    | बलपूर्वक बंद           |
| `fireworks/accounts/fireworks/routers/kimi-k2p6-turbo` | Kimi K2.6 Turbo (Fire Pass) | टेक्स्ट + इमेज | 256,000 | 256,000    | बलपूर्वक बंद (डिफ़ॉल्ट) |

<Note>
  OpenClaw सभी Fireworks Kimi मॉडलों को `thinking: off` पर पिन करता है, क्योंकि अनुरोध में थिंकिंग को स्पष्ट रूप से अक्षम न करने पर Fireworks पर Kimi, चेन-ऑफ़-थॉट को दृश्यमान उत्तर में प्रकट कर सकता है। उसी मॉडल को सीधे [Moonshot](/hi/providers/moonshot) के माध्यम से रूट करने पर Kimi का रीजनिंग आउटपुट सुरक्षित रहता है। प्रोवाइडरों के बीच स्विच करने के लिए [थिंकिंग मोड](/hi/tools/thinking) देखें।
</Note>

## कस्टम Fireworks मॉडल आईडी

OpenClaw रनटाइम पर किसी भी Fireworks मॉडल या राउटर आईडी को स्वीकार करता है। Fireworks द्वारा दिखाई गई सटीक आईडी का उपयोग करें और उसके आगे `fireworks/` लगाएँ। डायनेमिक रिज़ॉल्यूशन Fire Pass टेम्पलेट (टेक्स्ट + इमेज इनपुट, OpenAI-संगत API, डिफ़ॉल्ट लागत शून्य) को क्लोन करता है और आईडी के Kimi पैटर्न से मेल खाने पर थिंकिंग को स्वचालित रूप से अक्षम कर देता है। जब तक आप इमेज इनपुट के साथ कोई कस्टम मॉडल प्रविष्टि कॉन्फ़िगर नहीं करते, तब तक GLM डायनेमिक आईडी को केवल-टेक्स्ट के रूप में चिह्नित किया जाता है।

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "fireworks/accounts/fireworks/models/<your-model-id>",
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="मॉडल आईडी प्रीफ़िक्सिंग कैसे काम करती है">
    OpenClaw में प्रत्येक Fireworks मॉडल संदर्भ की शुरुआत `fireworks/` से होती है, जिसके बाद Fireworks प्लेटफ़ॉर्म की सटीक आईडी या राउटर पाथ आता है। उदाहरण के लिए:

    - राउटर मॉडल: `fireworks/accounts/fireworks/routers/kimi-k2p6-turbo`
    - प्रत्यक्ष मॉडल: `fireworks/accounts/fireworks/models/<model-name>`

    API अनुरोध बनाते समय OpenClaw, `fireworks/` प्रीफ़िक्स हटा देता है और शेष पाथ को OpenAI-संगत `model` फ़ील्ड के रूप में Fireworks एंडपॉइंट पर भेजता है।

  </Accordion>

  <Accordion title="Kimi के लिए थिंकिंग को बलपूर्वक बंद क्यों किया जाता है">
    Fireworks, Kimi को अलग रीजनिंग चैनल के बिना उपलब्ध कराता है, इसलिए चेन-ऑफ़-थॉट दृश्यमान `content` स्ट्रीम में दिखाई दे सकता है। प्रत्येक Fireworks Kimi अनुरोध पर OpenClaw, `thinking: { type: "disabled" }` भेजता है और पेलोड (`extensions/fireworks/stream.ts`) से `reasoning`, `reasoning_effort`, और `reasoningEffort` हटा देता है। प्रोवाइडर नीति (`extensions/fireworks/thinking-policy.ts`), Kimi मॉडल आईडी के लिए केवल `off` थिंकिंग स्तर को घोषित करती है, ताकि मैन्युअल `/think` स्विच और प्रोवाइडर-नीति सतहें रनटाइम अनुबंध के अनुरूप रहें।

    Kimi रीजनिंग का शुरू से अंत तक उपयोग करने के लिए, [Moonshot प्रोवाइडर](/hi/providers/moonshot) कॉन्फ़िगर करें और उसी मॉडल को उसके माध्यम से रूट करें।

  </Accordion>

  <Accordion title="डेमन के लिए एन्वायरनमेंट उपलब्धता">
    यदि Gateway किसी प्रबंधित सेवा (launchd, systemd, Docker) के रूप में चलता है, तो Fireworks कुंजी उस प्रोसेस को दिखाई देनी चाहिए—केवल आपके इंटरैक्टिव शेल को नहीं।

    <Warning>
      केवल इंटरैक्टिव शेल में एक्सपोर्ट की गई कुंजी launchd या systemd डेमन के लिए तब तक उपयोगी नहीं होगी, जब तक उस एन्वायरनमेंट को वहाँ भी इंपोर्ट न किया जाए। कुंजी को Gateway प्रोसेस से पठनीय बनाने के लिए उसे `~/.openclaw/.env` में या `env.shellEnv` के माध्यम से सेट करें।
    </Warning>

    कॉन्फ़िग लोड करते समय OpenClaw, `~/.openclaw/.env` को लोड करता है, इसलिए वहाँ संग्रहीत कुंजियाँ प्रत्येक प्लेटफ़ॉर्म पर प्रबंधित Gateway सेवाओं तक पहुँचती हैं। कुंजी रोटेट करने के बाद Gateway को रीस्टार्ट करें (या `openclaw doctor --fix` दोबारा चलाएँ)।

  </Accordion>
</AccordionGroup>

## संबंधित

<CardGroup cols={2}>
  <Card title="मॉडल प्रोवाइडर" href="/hi/concepts/model-providers" icon="layers">
    प्रोवाइडर, मॉडल संदर्भ और फ़ेलओवर व्यवहार चुनना।
  </Card>
  <Card title="थिंकिंग मोड" href="/hi/tools/thinking" icon="brain">
    `/think` स्तर, प्रोवाइडर नीतियाँ और रीजनिंग-सक्षम मॉडलों की रूटिंग।
  </Card>
  <Card title="Moonshot" href="/hi/providers/moonshot" icon="moon">
    Moonshot के अपने API के माध्यम से मूल थिंकिंग आउटपुट के साथ Kimi चलाएँ।
  </Card>
  <Card title="समस्या निवारण" href="/hi/help/troubleshooting" icon="wrench">
    सामान्य समस्या निवारण और अक्सर पूछे जाने वाले प्रश्न।
  </Card>
</CardGroup>
