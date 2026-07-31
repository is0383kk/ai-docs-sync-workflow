---
read_when:
    - आप एक नया कस्टम स्किल बना रहे हैं
    - आपको SKILL.md-आधारित Skills के लिए एक त्वरित शुरुआती कार्यप्रवाह चाहिए
    - आप एजेंट समीक्षा के लिए कोई स्किल प्रस्तावित करने हेतु Skill Workshop का उपयोग करना चाहते हैं
sidebarTitle: Creating skills
summary: अपने OpenClaw एजेंटों के लिए कस्टम SKILL.md वर्कस्पेस Skills बनाएँ, उनका परीक्षण करें और उन्हें प्रकाशित करें।
title: Skills बनाना
x-i18n:
    generated_at: "2026-07-27T19:06:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cba2aa863ebd083d4592e8a764dbdc2c30a0dd8aff49d273927e82df0069bc81
    source_path: tools/creating-skills.md
    workflow: 16
---

Skills एजेंट को सिखाते हैं कि टूल्स का उपयोग कैसे और कब करना है। प्रत्येक Skill एक डायरेक्टरी है
जिसमें YAML frontmatter और Markdown निर्देशों वाली `SKILL.md` फ़ाइल होती है।
OpenClaw एक निर्धारित [प्राथमिकता क्रम](/hi/tools/skills#loading-order) में कई रूट से Skills लोड करता है।

## अपना पहला Skill बनाएँ

<Steps>
  <Step title="Skill डायरेक्टरी बनाएँ">
    Skills आपके वर्कस्पेस के `skills/` फ़ोल्डर में रहते हैं:

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/hello-world
    ```

    व्यवस्थित रखने के लिए आप Skills को सबफ़ोल्डर में समूहित कर सकते हैं — फिर भी Skill का
    नाम फ़ोल्डर पथ से नहीं, बल्कि `SKILL.md` frontmatter से निर्धारित होता है:

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/personal/hello-world
    # Skill का नाम अब भी "hello-world" है, इसे /hello-world के रूप में चलाया जाता है
    ```

  </Step>

  <Step title="SKILL.md लिखें">
    frontmatter मेटाडेटा निर्धारित करता है; बॉडी एजेंट को निर्देश देती है।

    ```markdown
    ---
    name: hello-world
    description: एक सरल Skill जो अभिवादन प्रिंट करता है।
    ---

    # नमस्ते दुनिया

    जब उपयोगकर्ता अभिवादन माँगे, तो यह चलाने के लिए `exec` टूल का उपयोग करें:

    ```bash
    echo "आपके कस्टम Skill की ओर से नमस्ते!"
    ```
    ```

    नामकरण नियम:
    - `name` के लिए छोटे अक्षर, अंक और हाइफ़न उपयोग करें।
    - डायरेक्टरी नाम और frontmatter के `name` को एक-समान रखें।
    - `description` एजेंट और स्लैश-कमांड खोज में दिखाई देता है —
      इसे एक पंक्ति में और 160 वर्णों से छोटा रखें।

  </Step>

  <Step title="सत्यापित करें कि Skill लोड हो गया है">
    ```bash
    openclaw skills list
    ```

    OpenClaw डिफ़ॉल्ट रूप से Skills रूट के अंतर्गत `SKILL.md` फ़ाइलों पर नज़र रखता है। यदि
    वॉचर अक्षम है या आप किसी मौजूदा सेशन को जारी रख रहे हैं, तो नया सेशन
    शुरू करें ताकि एजेंट को ताज़ा सूची प्राप्त हो:

    ```bash
    # चैट से — मौजूदा सेशन को आर्काइव करके नया शुरू करें
    /new

    # या Gateway को रीस्टार्ट करें
    openclaw gateway restart
    ```

  </Step>

  <Step title="इसका परीक्षण करें">
    ```bash
    openclaw agent --message "मुझे एक अभिवादन दें"
    ```

    या चैट खोलकर एजेंट से सीधे पूछें। इसे नाम से स्पष्ट रूप से
    चलाने के लिए `/skill hello-world` का उपयोग करें।

  </Step>
</Steps>

## SKILL.md संदर्भ

### आवश्यक फ़ील्ड

| फ़ील्ड         | विवरण                                                     |
| ------------- | --------------------------------------------------------------- |
| `name`        | छोटे अक्षरों, अंकों और हाइफ़न का उपयोग करने वाला अद्वितीय स्लग        |
| `description` | एजेंट और खोज आउटपुट में दिखाई देने वाला एक-पंक्ति विवरण |

### वैकल्पिक frontmatter कुंजियाँ

| फ़ील्ड                      | डिफ़ॉल्ट | विवरण                                                                      |
| -------------------------- | ------- | -------------------------------------------------------------------------------- |
| `user-invocable`           | `true`  | Skill को उपयोगकर्ता स्लैश कमांड के रूप में उपलब्ध कराएँ                                         |
| `disable-model-invocation` | `false` | Skill को एजेंट के सिस्टम प्रॉम्प्ट से बाहर रखें (यह फिर भी `/skill` के माध्यम से चलता है)        |
| `command-dispatch`         | —       | मॉडल को बायपास करते हुए स्लैश कमांड को सीधे किसी टूल तक रूट करने के लिए `tool` पर सेट करें |
| `command-tool`             | —       | `command-dispatch: tool` सेट होने पर चलाए जाने वाले टूल का नाम                         |
| `command-arg-mode`         | `raw`   | टूल डिस्पैच के लिए, मूल आर्ग्स स्ट्रिंग टूल को फ़ॉरवर्ड करता है                      |
| `homepage`                 | —       | macOS Skills UI में "Website" के रूप में दिखाई जाने वाली URL                                    |

गेटिंग फ़ील्ड (`requires.bins`, `requires.env` आदि) के लिए
[Skills — गेटिंग](/hi/tools/skills#gating) देखें।

### `{baseDir}` का उपयोग करना

पथों को हार्डकोड किए बिना Skill डायरेक्टरी के अंदर की फ़ाइलों को संदर्भित करें — एजेंट
`{baseDir}` को Skill की अपनी डायरेक्टरी के सापेक्ष रिज़ॉल्व करता है:

```markdown
`{baseDir}/scripts/run.sh` पर स्थित सहायक स्क्रिप्ट चलाएँ।
```

## सशर्त सक्रियण जोड़ना

अपने Skill को गेट करें ताकि वह केवल तभी लोड हो जब उसकी निर्भरताएँ उपलब्ध हों:

```markdown
---
name: gemini-search
description: Gemini CLI का उपयोग करके खोजें।
metadata: { "openclaw": { "requires": { "bins": ["gemini"] }, "primaryEnv": "GEMINI_API_KEY" } }
---
```

<AccordionGroup>
  <Accordion title="गेटिंग विकल्प">
    | कुंजी | विवरण |
    | --- | --- |
    | `requires.bins` | सभी बाइनरी `PATH` पर मौजूद होनी चाहिए |
    | `requires.anyBins` | कम-से-कम एक बाइनरी `PATH` पर मौजूद होनी चाहिए |
    | `requires.env` | प्रत्येक एनवायरनमेंट वेरिएबल प्रोसेस या कॉन्फ़िग में मौजूद होना चाहिए |
    | `requires.config` | प्रत्येक `openclaw.json` पथ का मान truthy होना चाहिए |
    | `os` | प्लेटफ़ॉर्म फ़िल्टर: `["darwin"]`, `["linux"]`, `["win32"]` |
    | `always` | सभी गेट छोड़कर Skill को हमेशा शामिल करने के लिए `true` सेट करें |

    पूरा संदर्भ: [Skills — गेटिंग](/hi/tools/skills#gating)।

  </Accordion>
  <Accordion title="एनवायरनमेंट और API कुंजियाँ">
    `openclaw.json` में किसी Skill प्रविष्टि से API कुंजी जोड़ें:

    ```json5
    {
      skills: {
        entries: {
          "gemini-search": {
            enabled: true,
            apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
          },
        },
      },
    }
    ```

    कुंजी केवल उस एजेंट टर्न के लिए होस्ट प्रोसेस में इंजेक्ट की जाती है।
    यह सैंडबॉक्स तक नहीं पहुँचती — देखें
    [सैंडबॉक्स किए गए एनवायरनमेंट वेरिएबल](/hi/tools/skills-config#sandboxed-skills-and-env-vars)।

  </Accordion>
</AccordionGroup>

## Skill Workshop के माध्यम से प्रस्तावित करें

एजेंट द्वारा तैयार किए गए Skills के लिए या जब आप किसी Skill को
लाइव करने से पहले ऑपरेटर समीक्षा चाहते हों, तो सीधे `SKILL.md` लिखने के बजाय
[Skill Workshop](/hi/tools/skill-workshop) प्रस्तावों का उपयोग करें।

```bash
# बिल्कुल नया Skill प्रस्तावित करें
openclaw skills workshop propose-create \
  --name "hello-world" \
  --description "अभिवादन प्रिंट करने वाला एक सरल Skill।" \
  --proposal ./PROPOSAL.md

# किसी मौजूदा Skill में अपडेट प्रस्तावित करें
openclaw skills workshop propose-update hello-world \
  --proposal ./PROPOSAL.md \
  --description "अपडेट किया गया अभिवादन Skill"
```

जब प्रस्ताव में सहायक फ़ाइलें शामिल हों, तो `--proposal-dir` का उपयोग करें:

```bash
openclaw skills workshop propose-create \
  --name "hello-world" \
  --description "अभिवादन प्रिंट करने वाला एक सरल Skill।" \
  --proposal-dir ./hello-world-proposal/
```

डायरेक्टरी के रूट में `PROPOSAL.md` होना आवश्यक है। सहायक फ़ाइलें
`assets/`, `examples/`, `references/`, `scripts/`, या `templates/` के अंतर्गत रखें।

समीक्षा के बाद:

```bash
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

प्रस्ताव के पूरे जीवनचक्र के लिए [Skill Workshop](/hi/tools/skill-workshop) देखें।

## ClawHub पर प्रकाशित करना

<Steps>
  <Step title="सुनिश्चित करें कि आपका SKILL.md पूरा है">
    सुनिश्चित करें कि `name`, `description`, और कोई भी `metadata.openclaw` गेटिंग फ़ील्ड
    सेट हैं। यदि आपके पास प्रोजेक्ट पेज है, तो `homepage` URL जोड़ें।
  </Step>
  <Step title="स्टैंडअलोन ClawHub CLI इंस्टॉल करें और लॉग इन करें">
    ```bash
    npm i -g clawhub
    clawhub login
    ```
  </Step>
  <Step title="प्रकाशित करें">
    ```bash
    clawhub skill publish ./path/to/hello-world
    ```

    अनुमानित संस्करण को ओवरराइड करने या किसी विशिष्ट स्वामी के अंतर्गत प्रकाशित करने के लिए
    `--version <version>` या `--owner <owner>` जोड़ें। पूरे प्रवाह, स्वामी-स्कोपिंग और अन्य
    रखरखाव कमांड (`clawhub sync`, `clawhub skill rename`, ...) के लिए
    [ClawHub — प्रकाशन](/hi/clawhub/publishing) और
    [ClawHub CLI](/hi/clawhub/cli) देखें।

  </Step>
</Steps>

## सर्वोत्तम अभ्यास

<Tip>
  - **संक्षिप्त रहें** — मॉडल को यह निर्देश दें कि *क्या* करना है, न कि AI कैसे बनना है।
  - **सुरक्षा पहले** — यदि आपका Skill `exec` का उपयोग करता है, तो सुनिश्चित करें कि प्रॉम्प्ट
    अविश्वसनीय इनपुट से मनमाना कमांड इंजेक्शन न होने दें।
  - **स्थानीय रूप से परीक्षण करें** — साझा करने से पहले `openclaw agent --message "..."` का उपयोग करें।
  - **ClawHub का उपयोग करें** — शुरुआत से बनाने से पहले [clawhub.ai](https://clawhub.ai)
    पर समुदाय के Skills ब्राउज़ करें।
</Tip>

## संबंधित

<CardGroup cols={2}>
  <Card title="Skills संदर्भ" href="/hi/tools/skills" icon="puzzle-piece">
    लोडिंग क्रम, गेटिंग, अनुमत-सूचियाँ और SKILL.md प्रारूप।
  </Card>
  <Card title="Skill Workshop" href="/hi/tools/skill-workshop" icon="flask">
    एजेंट द्वारा तैयार किए गए Skills के लिए प्रस्ताव कतार।
  </Card>
  <Card title="Skills कॉन्फ़िग" href="/hi/tools/skills-config" icon="gear">
    पूरा `skills.*` कॉन्फ़िग स्कीमा।
  </Card>
  <Card title="ClawHub" href="/hi/clawhub" icon="cloud">
    सार्वजनिक रजिस्ट्री पर Skills ब्राउज़ और प्रकाशित करें।
  </Card>
  <Card title="Plugins बनाना" href="/hi/plugins/building-plugins" icon="plug">
    Plugins अपने द्वारा प्रलेखित टूल्स के साथ Skills प्रदान कर सकते हैं।
  </Card>
</CardGroup>
