---
read_when:
    - आप GitHub Copilot को मॉडल प्रदाता के रूप में उपयोग करना चाहते हैं
    - आपको `openclaw models auth login-github-copilot` प्रवाह की आवश्यकता है
    - आप अंतर्निहित Copilot प्रदाता, Copilot SDK हार्नेस और Copilot Proxy के बीच चयन कर रहे हैं
summary: डिवाइस प्रवाह या गैर-इंटरैक्टिव टोकन आयात का उपयोग करके OpenClaw से GitHub Copilot में साइन इन करें
title: GitHub Copilot
x-i18n:
    generated_at: "2026-07-27T19:50:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e839e6c72e7e7cb106a2f98c62c4994b4f3d6f34a2e76b549f2f6ccfdac91fe6
    source_path: providers/github-copilot.md
    workflow: 16
---

GitHub Copilot, GitHub का AI कोडिंग सहायक है। यह आपके GitHub खाते और प्लान के लिए Copilot
मॉडल तक पहुँच प्रदान करता है। OpenClaw, Copilot को मॉडल प्रदाता या एजेंट रनटाइम के रूप में
तीन अलग-अलग तरीकों से उपयोग कर सकता है।

## OpenClaw में Copilot उपयोग करने के तीन तरीके

<Tabs>
  <Tab title="अंतर्निहित प्रदाता (github-copilot)">
    GitHub टोकन प्राप्त करने के लिए मूल डिवाइस-लॉगिन प्रवाह का उपयोग करें, फिर OpenClaw के
    चलने पर उसे Copilot API टोकन से बदलें। यह **डिफ़ॉल्ट** और सबसे सरल तरीका है,
    क्योंकि इसके लिए VS Code की आवश्यकता नहीं होती।

    <Steps>
      <Step title="लॉगिन कमांड चलाएँ">
        ```bash
        openclaw models auth login-github-copilot
        ```

        आपको एक URL पर जाने और एक बार उपयोग होने वाला कोड दर्ज करने के लिए कहा जाएगा।
        प्रक्रिया पूरी होने तक टर्मिनल खुला रखें।
      </Step>
      <Step title="डिफ़ॉल्ट मॉडल सेट करें">
        ```bash
        openclaw models set github-copilot/claude-opus-4.7
        ```

        या कॉन्फ़िगरेशन में:

        ```json5
        {
          agents: {
            defaults: { model: { primary: "github-copilot/claude-opus-4.7" } },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Copilot SDK हार्नेस Plugin (copilot)">
    जब आप चाहते हैं कि चयनित `github-copilot/*` मॉडल के लिए निम्न-स्तरीय एजेंट लूप का
    नियंत्रण GitHub के Copilot CLI और SDK के पास हो, तब बाहरी `@openclaw/copilot`
    Plugin इंस्टॉल करें।

    ```bash
    openclaw plugins install @openclaw/copilot
    ```

    फिर किसी मॉडल या प्रदाता को रनटाइम का उपयोग करने के लिए चुनें:

    ```json5
    {
      agents: {
        defaults: {
          model: "github-copilot/gpt-5.5",
          models: {
            "github-copilot/gpt-5.5": {
              agentRuntime: { id: "copilot" },
            },
          },
        },
      },
    }
    ```

    इसे तब चुनें जब आप उन एजेंट टर्न के लिए मूल Copilot CLI सत्र, SDK-प्रबंधित थ्रेड
    स्थिति और Copilot-प्रबंधित Compaction चाहते हैं। स्पष्ट `agentRuntime`
    चयन के बिना, `github-copilot/*` मॉडल अंतर्निहित प्रदाता का उपयोग करते रहते हैं।
    संपूर्ण रनटाइम अनुबंध के लिए [Copilot SDK हार्नेस](/hi/plugins/copilot) देखें।

  </Tab>

  <Tab title="Copilot Proxy Plugin (copilot-proxy)">
    स्थानीय ब्रिज के रूप में **Copilot Proxy** VS Code एक्सटेंशन का उपयोग करें। OpenClaw,
    प्रॉक्सी के `/v1` एंडपॉइंट (डिफ़ॉल्ट `http://localhost:3000/v1`) से संचार करता है
    और आपके द्वारा कॉन्फ़िगर की गई मॉडल सूची का उपयोग करता है।

    `copilot-proxy` Plugin, OpenClaw के साथ आता है और डिफ़ॉल्ट रूप से सक्षम होता है।
    बेस URL और मॉडल ID को इससे कॉन्फ़िगर करें:

    ```bash
    openclaw models auth login --provider copilot-proxy --set-default
    ```

    <Note>
    इसे तब चुनें जब आप पहले से VS Code में Copilot Proxy चलाते हों या इसके माध्यम से
    रूटिंग की आवश्यकता हो। VS Code एक्सटेंशन का चलते रहना आवश्यक है।
    </Note>

  </Tab>
</Tabs>

## GitHub Enterprise (डेटा रेज़िडेंसी)

यदि आपका संगठन डेटा-रेज़िडेंसी GitHub Enterprise टेनेंट (जैसे
`your-org.ghe.com` जैसा कोई `*.ghe.com` होस्ट) उपयोग करता है, तो Copilot सार्वजनिक
`github.com` के बजाय टेनेंट-स्थानीय एंडपॉइंट पर रहता है। OpenClaw इसे
प्रथम-श्रेणी प्रमाणीकरण विकल्प के रूप में प्रस्तुत करता है, ताकि आपको URL हाथ से संपादित न करने पड़ें।

<Steps>
  <Step title="Enterprise प्रमाणीकरण विकल्प चुनें">
    ऑनबोर्डिंग या `openclaw models auth` में,
    **GitHub Copilot (Enterprise / डेटा रेज़िडेंसी)** चुनें। आपसे आपका Enterprise
    डोमेन (उदाहरण के लिए `your-org.ghe.com`) पूछा जाएगा, फिर उस टेनेंट के विरुद्ध
    डिवाइस लॉगिन चलेगा।

    केवल टेनेंट रूट (`your-org.ghe.com`) दर्ज करें। `api.your-org.ghe.com` या
    `copilot-api.your-org.ghe.com` जैसे व्युत्पन्न सेवा होस्ट स्वीकार नहीं किए जाते;
    OpenClaw उन एंडपॉइंट को टेनेंट रूट से स्वचालित रूप से प्राप्त करता है।

    ```bash
    openclaw models auth login --provider github-copilot --method device-enterprise
    ```

  </Step>
  <Step title="डोमेन कॉन्फ़िगरेशन में सहेजा जाता है">
    चुना गया होस्ट प्रदाता पैरामीटर के अंतर्गत संग्रहीत होता है, ताकि बाद के टोकन
    रीफ़्रेश और कम्प्लीशन स्वचालित रूप से उस टेनेंट को लक्षित करें:

    ```json5
    {
      models: {
        providers: {
          "github-copilot": { params: { githubDomain: "your-org.ghe.com" } },
        },
      },
    }
    ```

  </Step>
</Steps>

डिवाइस प्रवाह, टोकन एक्सचेंज और कम्प्लीशन क्रमशः
`https://your-org.ghe.com/login/device/code`,
`https://api.your-org.ghe.com/copilot_internal/v2/token` और
`https://copilot-api.your-org.ghe.com` पर रिज़ॉल्व होते हैं। डेटा-रेज़िडेंसी टोकन में
टेनेंट स्टैम्प होता है और कोई प्रॉक्सी संकेत नहीं होता, इसलिए कम्प्लीशन का बेस URL
सार्वजनिक एंडपॉइंट के बजाय टेनेंट Copilot होस्ट पर फ़ॉलबैक करता है।

<Note>
डोमेन बदलने पर डिवाइस लॉगिन हमेशा दोबारा चलता है। यदि आपके पास पहले से संग्रहीत
Copilot टोकन है और आप कोई अलग डोमेन चुनते हैं (सार्वजनिक `github.com` ↔ कोई
`*.ghe.com` टेनेंट, या एक टेनेंट से दूसरा), तो OpenClaw मौजूदा टोकन का पुनः उपयोग
नहीं करेगा—यह नया लॉगिन अनिवार्य करता है, ताकि टोकन कॉन्फ़िगरेशन में लिखे जा रहे
डोमेन तक सीमित हो। *उसी* डोमेन के लिए लॉगिन दोबारा चलाने पर मौजूदा टोकन के पुनः उपयोग
का विकल्प फिर भी दिया जाता है। सार्वजनिक `github.com` पर वापस जाने से सहेजा गया
`githubDomain` हट जाता है, जिससे कॉन्फ़िगरेशन फिर डिफ़ॉल्ट पर लौट आता है।
</Note>

<Note>
`COPILOT_GITHUB_DOMAIN` एनवायरनमेंट वेरिएबल हर उस Copilot पथ के लिए रिज़ॉल्व किए गए डोमेन
को ओवरराइड करता है जो इसे रिज़ॉल्व करता है—Enterprise डिवाइस लॉगिन
(`--method device-enterprise`), स्वतंत्र
`openclaw models auth login-github-copilot` शॉर्टकट, टोकन रीफ़्रेश, एम्बेडिंग
और कम्प्लीशन। पूरी तरह हेडलेस या CI सेटअप के लिए इसे अपने `*.ghe.com` होस्ट पर
सेट करें। सार्वजनिक `github.com` का उपयोग करने के लिए इसे सेट न करें (और कॉन्फ़िगरेशन
पैरामीटर भी अनुपस्थित रखें)। लॉगिन उस डोमेन को सहेजते हैं जिसके लिए उन्होंने टोकन बनाया था
(और सार्वजनिक `github.com` के विरुद्ध लॉगिन करने पर उसे हटा देते हैं), इसलिए
एनवायरनमेंट वेरिएबल हटाए जाने के बाद भी रूटिंग सही रहती है।
</Note>

## वैकल्पिक फ़्लैग

| कमांड                                                                  | फ़्लैग          | विवरण                                               |
| ---------------------------------------------------------------------- | --------------- | ---------------------------------------------------- |
| `openclaw models auth login-github-copilot`                            | `--yes`         | बिना पूछे मौजूदा प्रमाणीकरण प्रोफ़ाइल ओवरराइट करें |
| `openclaw models auth login --provider github-copilot --method device` | `--set-default` | प्रदाता का अनुशंसित डिफ़ॉल्ट मॉडल भी लागू करें     |

```bash
# दोबारा लॉगिन की पुष्टि छोड़ें
openclaw models auth login-github-copilot --yes

# एक ही चरण में लॉगिन करके डिफ़ॉल्ट मॉडल सेट करें
openclaw models auth login --provider github-copilot --method device --set-default
```

## गैर-इंटरैक्टिव ऑनबोर्डिंग

डिवाइस-लॉगिन प्रवाह के लिए इंटरैक्टिव TTY आवश्यक है। हेडलेस सेटअप के लिए,
`openclaw onboard --non-interactive` के साथ मौजूदा GitHub OAuth एक्सेस टोकन आयात करें:

```bash
openclaw onboard --non-interactive --accept-risk \
  --auth-choice github-copilot \
  --github-copilot-token "$COPILOT_GITHUB_TOKEN" \
  --skip-channels --skip-health
```

आप `--auth-choice` को छोड़ भी सकते हैं; `--github-copilot-token` पास करने पर
GitHub Copilot प्रदाता प्रमाणीकरण विकल्प का अनुमान लगाया जाता है। यदि फ़्लैग छोड़ दिया जाए,
तो ऑनबोर्डिंग `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, फिर `GITHUB_TOKEN` पर
फ़ॉलबैक करती है। `auth-profiles.json` में सादे टेक्स्ट के बजाय एनवायरनमेंट-समर्थित
`tokenRef` संग्रहीत करने के लिए `COPILOT_GITHUB_TOKEN` सेट करके
`--secret-input-mode ref` का उपयोग करें।

<AccordionGroup>
  <Accordion title="इंटरैक्टिव TTY आवश्यक है">
    डिवाइस-लॉगिन प्रवाह के लिए इंटरैक्टिव TTY आवश्यक है। इसे सीधे टर्मिनल में चलाएँ,
    किसी गैर-इंटरैक्टिव स्क्रिप्ट या CI पाइपलाइन में नहीं।
  </Accordion>

  <Accordion title="मॉडल उपलब्धता आपके प्लान पर निर्भर करती है">
    Copilot मॉडल की उपलब्धता आपके GitHub प्लान पर निर्भर करती है। यदि कोई मॉडल
    अस्वीकार हो जाए, तो कोई अन्य ID आज़माएँ (उदाहरण के लिए `github-copilot/gpt-5.5`)।
    मौजूदा मॉडल सूची के लिए GitHub के
    [प्रत्येक Copilot प्लान के समर्थित मॉडल](https://docs.github.com/en/copilot/reference/ai-models/supported-models#supported-ai-models-per-copilot-plan)
    देखें।
  </Accordion>

  <Accordion title="Copilot API से लाइव कैटलॉग रीफ़्रेश">
    डिवाइस-लॉगिन (या एनवायरनमेंट-वेरिएबल) प्रमाणीकरण पथ द्वारा GitHub टोकन रिज़ॉल्व
    किए जाने के बाद, OpenClaw माँग पर `${baseUrl}/models` से मॉडल कैटलॉग रीफ़्रेश
    करता है (यही एंडपॉइंट VS Code Copilot उपयोग करता है), ताकि रनटाइम बिना मैनिफ़ेस्ट
    बदलाव के प्रति-खाता पात्रता और सटीक कॉन्टेक्स्ट विंडो के अनुरूप रहे। नए प्रकाशित
    Copilot मॉडल OpenClaw अपग्रेड के बिना दिखाई देने लगते हैं और कॉन्टेक्स्ट विंडो
    वास्तविक प्रति-मॉडल सीमाएँ दर्शाती हैं (उदाहरण के लिए gpt-5.x शृंखला के लिए
    400k और आंतरिक `claude-opus-*-1m` प्रकारों के लिए 1M)।

    जब डिस्कवरी अक्षम हो, उपयोगकर्ता के पास कोई GitHub प्रमाणीकरण प्रोफ़ाइल न हो,
    टोकन एक्सचेंज विफल हो जाए या `/models` HTTPS कॉल में त्रुटि आए, तब
    बंडल किया हुआ स्थिर कैटलॉग दृश्यमान फ़ॉलबैक बना रहता है। इससे बाहर निकलकर पूरी तरह
    स्थिर मैनिफ़ेस्ट कैटलॉग पर निर्भर रहने के लिए (ऑफ़लाइन / एयर-गैप परिदृश्य):

    ```json5
    {
      plugins: {
        entries: {
          "github-copilot": {
            config: { discovery: { enabled: false } },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="ट्रांसपोर्ट चयन">
    Claude मॉडल ID स्वचालित रूप से Anthropic Messages ट्रांसपोर्ट का उपयोग करते हैं।
    Gemini मॉडल OpenAI Chat Completions ट्रांसपोर्ट का उपयोग करते हैं; GPT और o-series
    मॉडल OpenAI Responses ट्रांसपोर्ट का उपयोग जारी रखते हैं। OpenClaw मॉडल रेफ़रेंस
    के आधार पर सही ट्रांसपोर्ट चुनता है।
  </Accordion>

  <Accordion title="अनुरोध संगतता">
    OpenClaw, Copilot ट्रांसपोर्ट पर Copilot IDE-शैली के अनुरोध हेडर
    (VS Code एडिटर/Plugin संस्करण और `vscode-chat` इंटीग्रेशन ID) भेजता है,
    टूल-परिणाम के अनुवर्ती टर्न को एजेंट द्वारा आरंभ किया गया चिह्नित करता है और
    किसी टर्न में इमेज इनपुट होने पर Copilot विज़न हेडर सेट करता है।
  </Accordion>

  <Accordion title="एनवायरनमेंट वेरिएबल रिज़ॉल्यूशन क्रम">
    OpenClaw निम्नलिखित प्राथमिकता क्रम में एनवायरनमेंट वेरिएबल से Copilot
    प्रमाणीकरण रिज़ॉल्व करता है:

    | प्राथमिकता | वेरिएबल              | टिप्पणियाँ                              |
    | ----------- | --------------------- | ---------------------------------------- |
    | 1           | `COPILOT_GITHUB_TOKEN` | सर्वोच्च प्राथमिकता, Copilot-विशिष्ट    |
    | 2           | `GH_TOKEN`            | GitHub CLI टोकन (फ़ॉलबैक)                |
    | 3           | `GITHUB_TOKEN`        | मानक GitHub टोकन (सबसे कम प्राथमिकता)   |

    जब एकाधिक वेरिएबल सेट हों, तो OpenClaw सर्वोच्च प्राथमिकता वाले वेरिएबल का उपयोग
    करता है। डिवाइस-लॉगिन प्रवाह (`openclaw models auth login-github-copilot`) अपना टोकन प्रमाणीकरण प्रोफ़ाइल
    स्टोर में संग्रहीत करता है और सभी एनवायरनमेंट वेरिएबल पर प्राथमिकता रखता है।

  </Accordion>

  <Accordion title="टोकन संग्रहण">
    लॉगिन एक GitHub टोकन को प्रमाणीकरण प्रोफ़ाइल स्टोर (प्रोफ़ाइल ID
    `github-copilot:github`) में संग्रहीत करता है और OpenClaw के चलने पर उसे अल्पकालिक
    Copilot API टोकन से बदलता है। आपको टोकन मैन्युअल रूप से प्रबंधित करने की आवश्यकता
    नहीं है।
  </Accordion>
</AccordionGroup>

## मेमोरी खोज एम्बेडिंग

GitHub Copilot, [मेमोरी खोज](/hi/concepts/memory-search) के लिए एम्बेडिंग प्रदाता के रूप में
भी काम कर सकता है। यदि आपके पास Copilot सदस्यता है और आपने लॉगिन किया हुआ है, तो
OpenClaw अलग API कुंजी के बिना एम्बेडिंग के लिए इसका उपयोग कर सकता है।

### कॉन्फ़िगरेशन

GitHub Copilot एम्बेडिंग का उपयोग करने के लिए `memory.search.provider` को स्पष्ट रूप से
सेट करें। यदि GitHub टोकन उपलब्ध है, तो OpenClaw Copilot API से उपलब्ध एम्बेडिंग
मॉडल खोजता है और स्वचालित रूप से सर्वोत्तम मॉडल चुनता है।

```json5
{
  memory: {
    search: {
      provider: "github-copilot",
      // वैकल्पिक: स्वचालित रूप से खोजे गए मॉडल को ओवरराइड करें
      model: "text-embedding-3-small",
    },
  },
}
```

### यह कैसे काम करता है

1. OpenClaw आपका GitHub टोकन रिज़ॉल्व करता है (एनवायरनमेंट वेरिएबल या प्रमाणीकरण प्रोफ़ाइल से)।
2. उसे अल्पकालिक Copilot API टोकन से बदलता है।
3. उपलब्ध एम्बेडिंग मॉडल खोजने के लिए Copilot के `/models` एंडपॉइंट से क्वेरी करता है।
4. सर्वोत्तम मॉडल चुनता है (वरीयता क्रम: `text-embedding-3-small`,
   `text-embedding-3-large`, `text-embedding-ada-002`)।
5. Copilot के `/embeddings` एंडपॉइंट पर एम्बेडिंग अनुरोध भेजता है।

मॉडल उपलब्धता आपके GitHub प्लान पर निर्भर करती है। यदि कोई एम्बेडिंग मॉडल उपलब्ध
नहीं है, तो OpenClaw Copilot को छोड़कर अगला प्रदाता आज़माता है।

## संबंधित

<CardGroup cols={2}>
  <Card title="मॉडल चयन" href="/hi/concepts/model-providers" icon="layers">
    प्रदाताओं, मॉडल संदर्भों और फ़ेलओवर व्यवहार का चयन।
  </Card>
  <Card title="OAuth और प्रमाणीकरण" href="/hi/gateway/authentication" icon="key">
    प्रमाणीकरण का विवरण और क्रेडेंशियल के पुनः उपयोग के नियम।
  </Card>
</CardGroup>
