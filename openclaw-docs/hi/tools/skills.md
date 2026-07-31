---
read_when:
    - Skills जोड़ना या संशोधित करना
    - Skills की गेटिंग, अनुमति-सूचियाँ या लोड नियम बदलना
    - Skill की प्राथमिकता और स्नैपशॉट व्यवहार को समझना
sidebarTitle: Skills
summary: Skills आपके एजेंट को टूल्स का उपयोग करना सिखाते हैं। जानें कि वे कैसे लोड होते हैं, प्राथमिकता कैसे काम करती है, और गेटिंग, अनुमति-सूचियों तथा एनवायरनमेंट इंजेक्शन को कैसे कॉन्फ़िगर किया जाता है।
title: Skills
x-i18n:
    generated_at: "2026-07-27T19:12:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6925add85652023e3dd2f51f607412fd0bf00581923f76ab2aafd2ca5b8d72be
    source_path: tools/skills.md
    workflow: 16
---

Skills मार्कडाउन निर्देश फ़ाइलें हैं, जो एजेंट को सिखाती हैं कि टूल्स का उपयोग कैसे और कब
करना है। प्रत्येक Skill एक ऐसी डायरेक्टरी में रहता है जिसमें YAML फ्रंटमैटर और मार्कडाउन
बॉडी वाली `SKILL.md` फ़ाइल होती है। OpenClaw बंडल किए गए Skills के साथ सभी स्थानीय
ओवरराइड लोड करता है और लोड के समय उन्हें परिवेश, कॉन्फ़िगरेशन तथा
बाइनरी की उपलब्धता के आधार पर फ़िल्टर करता है।

<CardGroup cols={2}>
  <Card title="Skills बनाना" href="/hi/tools/creating-skills" icon="hammer">
    शुरुआत से कस्टम Skill बनाएँ और उसका परीक्षण करें।
  </Card>
  <Card title="Skill कार्यशाला" href="/hi/tools/skill-workshop" icon="flask">
    एजेंट द्वारा तैयार किए गए Skill प्रस्तावों की समीक्षा करें और उन्हें स्वीकृत करें।
  </Card>
  <Card title="Skills कॉन्फ़िगरेशन" href="/hi/tools/skills-config" icon="gear">
    संपूर्ण `skills.*` कॉन्फ़िगरेशन स्कीमा और एजेंट अनुमति-सूचियाँ।
  </Card>
  <Card title="ClawHub" href="/hi/clawhub" icon="cloud">
    समुदाय के Skills ब्राउज़ और इंस्टॉल करें।
  </Card>
</CardGroup>

## लोडिंग क्रम

OpenClaw इन स्रोतों से लोड करता है, **सबसे उच्च प्राथमिकता पहले**। जब एक ही
Skill का नाम कई स्थानों पर दिखाई देता है, तो उच्चतम स्रोत प्रभावी होता है।

| प्राथमिकता     | स्रोत                    | पथ                                     |
| -------------- | ------------------------ | --------------------------------------- |
| 1 — सर्वोच्च   | कार्यक्षेत्र Skills      | `<workspace>/skills`                    |
| 2              | प्रोजेक्ट एजेंट Skills   | `<workspace>/.agents/skills`            |
| 3              | व्यक्तिगत एजेंट Skills   | `~/.agents/skills`                      |
| 4              | प्रबंधित / स्थानीय Skills | `~/.openclaw/skills`                    |
| 5              | बंडल किए गए Skills       | इंस्टॉलेशन के साथ प्रदान किए जाते हैं   |
| 6 — निम्नतम    | अतिरिक्त डायरेक्टरियाँ   | `skills.load.extraDirs` + Plugin Skills |

Skill रूट समूहीकृत लेआउट का समर्थन करते हैं। जब भी किसी कॉन्फ़िगर किए गए रूट के अंतर्गत
`SKILL.md` कहीं दिखाई देता है, OpenClaw उस Skill को खोज लेता है (अधिकतम 6 स्तर की गहराई तक):

```text
<workspace>/skills/research/SKILL.md          ✓ "research" के रूप में मिला
<workspace>/skills/personal/research/SKILL.md ✓ यह भी "research" के रूप में मिला
```

फ़ोल्डर पथ केवल संगठन के लिए है। Skill का नाम और स्लैश कमांड
`name` फ्रंटमैटर फ़ील्ड से आते हैं (या `name` अनुपस्थित होने पर डायरेक्टरी
के नाम से)। एजेंट अनुमति-सूचियाँ (नीचे) भी इसी `name` से मिलान करती हैं।

<Note>
  Codex CLI की मूल `$CODEX_HOME/skills` डायरेक्टरी OpenClaw
  Skill रूट **नहीं** है। उन Skills की सूची बनाने के लिए `openclaw migrate plan codex` का उपयोग करें, फिर
  उन्हें अपने OpenClaw कार्यक्षेत्र में कॉपी करने के लिए `openclaw migrate codex` का उपयोग करें।
</Note>

## Node पर होस्ट किए गए Skills

कनेक्ट किया गया हेडलेस Node अपनी सक्रिय OpenClaw Skills
डायरेक्टरी में इंस्टॉल किए गए Skills प्रकाशित कर सकता है (डिफ़ॉल्ट रूप से `~/.openclaw/skills`; प्रोफ़ाइल परिवेश
ओवरराइड लागू होते हैं)। Node के कनेक्ट रहने पर वे सामान्य एजेंट Skill सूची में दिखाई देते हैं
और डिस्कनेक्ट होने पर गायब हो जाते हैं। नाम टकराने पर स्थानीय या Gateway Skill अपना नाम
बनाए रखता है; Node Skill को नियतात्मक Node-प्रीफ़िक्स वाला नाम मिलता है।
Node-होस्टेड v1 के लिए डायरेक्टरी का नाम Skill के `name`
फ्रंटमैटर फ़ील्ड से मेल खाना आवश्यक है।

Skill प्रविष्टि में Node लोकेटर शामिल होता है। इसकी फ़ाइलें, सापेक्ष संदर्भ और
बाइनरियाँ Node पर रहती हैं, इसलिए इसे
`exec host=node node=<node-id>` के साथ लोड और निष्पादित करें। इसकी Skill
फ़ाइलें बदलने के बाद Node होस्ट को पुनः आरंभ करें। पेयरिंग और बंद करने के विकल्पों के लिए [Nodes](/hi/nodes#node-hosted-skills) देखें।

## प्रति-एजेंट बनाम साझा Skills

बहु-एजेंट सेटअप में प्रत्येक एजेंट का अपना कार्यक्षेत्र होता है। अपनी वांछित दृश्यता से
मेल खाने वाले पथ का उपयोग करें:

| दायरा             | पथ                          | इनके लिए दृश्यमान              |
| ----------------- | ---------------------------- | ------------------------------ |
| प्रति-एजेंट       | `<workspace>/skills`         | केवल वह एजेंट                  |
| प्रोजेक्ट-एजेंट   | `<workspace>/.agents/skills` | केवल उस कार्यक्षेत्र का एजेंट  |
| व्यक्तिगत-एजेंट   | `~/.agents/skills`           | इस मशीन के सभी एजेंट           |
| साझा प्रबंधित     | `~/.openclaw/skills`         | इस मशीन के सभी एजेंट           |
| अतिरिक्त डायरेक्टरियाँ | `skills.load.extraDirs`      | इस मशीन के सभी एजेंट           |

## एजेंट अनुमति-सूचियाँ

Skill का **स्थान** (प्राथमिकता) और Skill की **दृश्यता** (कौन-सा एजेंट उसका उपयोग कर सकता
है) अलग-अलग नियंत्रण हैं। एजेंट को दिखाई देने वाले Skills सीमित करने के लिए अनुमति-सूचियों का उपयोग करें,
चाहे वे कहीं से भी लोड किए गए हों।

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // साझा आधाररेखा
    },
    list: [
      { id: "writer" }, // github, weather इनहेरिट करता है
      { id: "docs", skills: ["docs-search"] }, // डिफ़ॉल्ट को पूरी तरह बदल देता है
      { id: "locked-down", skills: [] }, // कोई Skill नहीं
    ],
  },
}
```

<AccordionGroup>
  <Accordion title="अनुमति-सूची के नियम">
    - सभी Skills को डिफ़ॉल्ट रूप से अप्रतिबंधित रखने के लिए `agents.defaults.skills` को छोड़ दें।
    - `agents.defaults.skills` इनहेरिट करने के लिए `agents.entries.*.skills` को छोड़ दें।
    - उस एजेंट के लिए कोई भी Skill उपलब्ध न कराने हेतु `agents.entries.*.skills: []` सेट करें।
    - गैर-रिक्त `agents.entries.*.skills` सूची **अंतिम** समुच्चय है — यह
      डिफ़ॉल्ट के साथ मर्ज नहीं होती।
    - प्रभावी अनुमति-सूची प्रॉम्प्ट निर्माण, स्लैश-कमांड
      खोज, सैंडबॉक्स सिंक और Skill स्नैपशॉट पर लागू होती है।
    - यह होस्ट शेल प्राधिकरण सीमा नहीं है। यदि वही एजेंट
      `exec` का उपयोग कर सकता है, तो उस शेल को सैंडबॉक्सिंग, OS-उपयोगकर्ता
      पृथक्करण, निष्पादन निषेध/अनुमति-सूचियों और प्रति-संसाधन क्रेडेंशियल से अलग से सीमित करें।
  </Accordion>
</AccordionGroup>

## Plugins और Skills

Plugins, `openclaw.plugin.json` में `skills` डायरेक्टरियाँ सूचीबद्ध करके
अपने Skills प्रदान कर सकते हैं (पथ Plugin रूट के सापेक्ष होते हैं)। Plugin सक्षम होने पर
Plugin Skills लोड होते हैं — उदाहरण के लिए, ब्राउज़र Plugin बहु-चरणीय ब्राउज़र
नियंत्रण के लिए `browser-automation` Skill प्रदान करता है।

Plugin Skill डायरेक्टरियाँ `skills.load.extraDirs` के समान निम्न-प्राथमिकता स्तर पर
मर्ज होती हैं, इसलिए उसी नाम वाला बंडल, प्रबंधित, एजेंट या कार्यक्षेत्र
Skill उन्हें ओवरराइड करता है। किसी Plugin Skill की अपनी पात्रता को उसके फ्रंटमैटर में
`metadata.openclaw.requires` के माध्यम से नियंत्रित करें, जैसा किसी अन्य Skill के लिए किया जाता है।

संपूर्ण Plugin सिस्टम के लिए [Plugins](/hi/tools/plugin) और [टूल्स](/hi/tools) देखें।

## Skill कार्यशाला

[Skill कार्यशाला](/hi/tools/skill-workshop), एजेंट और आपकी सक्रिय Skill फ़ाइलों के बीच
एक प्रस्ताव कतार है। जब एजेंट को पुनः उपयोग योग्य काम मिलता है, तो वह सीधे
`SKILL.md` में लिखने के बजाय प्रस्ताव का मसौदा तैयार करता है। कोई भी बदलाव होने से पहले
आप उसकी समीक्षा और स्वीकृति करते हैं।

```bash
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

संपूर्ण जीवनचक्र, CLI संदर्भ और कॉन्फ़िगरेशन के लिए
[Skill कार्यशाला](/hi/tools/skill-workshop) देखें।

## ClawHub से इंस्टॉल करना

[ClawHub](https://clawhub.ai) सार्वजनिक Skills रजिस्ट्री है। इंस्टॉल और अपडेट के लिए
`openclaw skills` कमांड का या प्रकाशित और सिंक करने के लिए `clawhub` CLI का
उपयोग करें।

| कार्रवाई                                  | कमांड                                                  |
| ----------------------------------------- | ------------------------------------------------------ |
| कार्यक्षेत्र में Skill इंस्टॉल करें       | `openclaw skills install @owner/<slug>`                |
| Git रिपॉज़िटरी से इंस्टॉल करें            | `openclaw skills install git:owner/repo@ref`           |
| स्थानीय Skill डायरेक्टरी इंस्टॉल करें     | `openclaw skills install ./path/to/skill --as my-tool` |
| सभी स्थानीय एजेंट के लिए इंस्टॉल करें     | `openclaw skills install @owner/<slug> --global`       |
| सभी कार्यक्षेत्र Skills अपडेट करें        | `openclaw skills update --all`                         |
| साझा प्रबंधित Skill अपडेट करें            | `openclaw skills update @owner/<slug> --global`        |
| सभी साझा प्रबंधित Skills अपडेट करें       | `openclaw skills update --all --global`                |
| Skill के विश्वास आवरण को सत्यापित करें    | `openclaw skills verify @owner/<slug>`                 |
| जनरेट किया गया Skill कार्ड प्रिंट करें    | `openclaw skills verify @owner/<slug> --card`          |
| ClawHub CLI से प्रकाशित / सिंक करें        | `clawhub sync --all`                                   |

<AccordionGroup>
  <Accordion title="इंस्टॉल विवरण">
    `openclaw skills install` डिफ़ॉल्ट रूप से सक्रिय कार्यक्षेत्र की `skills/`
    डायरेक्टरी में इंस्टॉल करता है। साझा `~/.openclaw/skills`
    डायरेक्टरी में इंस्टॉल करने के लिए `--global` जोड़ें, जो सभी स्थानीय एजेंट को दिखाई देती है,
    जब तक एजेंट अनुमति-सूचियाँ इसे सीमित न करें।

    Git और स्थानीय इंस्टॉल स्रोत रूट पर `SKILL.md` अपेक्षित करते हैं। मान्य होने पर स्लग
    `SKILL.md` फ्रंटमैटर `name` से आता है, फिर
    डायरेक्टरी या रिपॉज़िटरी नाम पर फ़ॉलबैक करता है। इसे ओवरराइड करने के लिए `--as <slug>` का उपयोग करें।
    `openclaw skills update` केवल ClawHub इंस्टॉल ट्रैक करता है — उन्हें रीफ़्रेश करने के लिए Git या
    स्थानीय स्रोतों को दोबारा इंस्टॉल करें।

  </Accordion>
  <Accordion title="सत्यापन और सुरक्षा स्कैनिंग">
    `openclaw skills verify @owner/<slug>`, ClawHub से Skill का
    `clawhub.skill.verify.v1` विश्वास आवरण माँगता है। इंस्टॉल किए गए ClawHub Skills का सत्यापन
    `.clawhub/origin.json` में दर्ज संस्करण और रजिस्ट्री के विरुद्ध होता है।
    मौजूदा इंस्टॉल किए गए या असंदिग्ध Skills के लिए केवल स्लग अब भी स्वीकार किए जाते हैं, लेकिन
    स्वामी-योग्य संदर्भ प्रकाशक की अस्पष्टता से बचाते हैं।

    ClawHub Skill पृष्ठ इंस्टॉल से पहले नवीनतम सुरक्षा स्कैन स्थिति दिखाते हैं,
    जिसमें VirusTotal, ClawScan और स्थैतिक विश्लेषण के विस्तृत पृष्ठ होते हैं। जब ClawHub
    सत्यापन को विफल चिह्नित करता है, तो कमांड गैर-शून्य स्थिति के साथ समाप्त होता है। प्रकाशक
    ClawHub डैशबोर्ड या `clawhub skill rescan @owner/<slug>` के माध्यम से गलत सकारात्मक परिणामों से उबर सकते हैं।

  </Accordion>
  <Accordion title="निजी आर्काइव इंस्टॉल">
    गैर-ClawHub वितरण की आवश्यकता वाले Gateway क्लाइंट
    `skills.upload.begin`, `skills.upload.chunk` और `skills.upload.commit` के साथ ज़िप Skill आर्काइव तैयार कर सकते हैं,
    फिर `skills.install({ source: "upload", ... })` से इंस्टॉल कर सकते हैं। यह पथ
    डिफ़ॉल्ट रूप से बंद होता है और इसके लिए `openclaw.json` में
    `skills.install.allowUploadedArchives: true` आवश्यक है। सामान्य ClawHub इंस्टॉल को कभी उस सेटिंग की आवश्यकता नहीं होती।
  </Accordion>
</AccordionGroup>

## सुरक्षा

<Warning>
  तृतीय-पक्ष Skills को **अविश्वसनीय कोड** मानें। सक्षम करने से पहले उन्हें पढ़ें।
  अविश्वसनीय इनपुट और जोखिमपूर्ण टूल्स के लिए सैंडबॉक्स किए गए रन को प्राथमिकता दें। एजेंट-साइड
  नियंत्रणों के लिए [सैंडबॉक्सिंग](/hi/gateway/sandboxing) देखें।
</Warning>

<AccordionGroup>
  <Accordion title="पथ परिसीमन">
    कार्यक्षेत्र, प्रोजेक्ट-एजेंट और अतिरिक्त-डायरेक्टरी Skill खोज केवल उन्हीं Skill
    रूट को स्वीकार करती है जिनका समाधान किया गया वास्तविक पथ कॉन्फ़िगर किए गए रूट के भीतर रहता है, जब तक कि
    `skills.load.allowSymlinkTargets` किसी लक्षित रूट पर स्पष्ट रूप से विश्वास न करे।
    `skills.workshop.allowSymlinkTargetWrites` सक्षम होने पर ही Skill कार्यशाला उन विश्वसनीय लक्ष्यों के माध्यम से लिखती है।
    प्रबंधित `~/.openclaw/skills` और व्यक्तिगत `~/.agents/skills` में
    सिमलिंक किए गए Skill फ़ोल्डर हो सकते हैं, लेकिन प्रत्येक `SKILL.md` का वास्तविक पथ फिर भी
    उसकी समाधान की गई Skill डायरेक्टरी के भीतर रहना चाहिए।
  </Accordion>
  <Accordion title="ऑपरेटर इंस्टॉल नीति">
    Skill इंस्टॉल जारी रहने से पहले विश्वसनीय स्थानीय नीति कमांड चलाने के लिए
    `security.installPolicy` कॉन्फ़िगर करें। नीति को मेटाडेटा और तैयार किया गया
    स्रोत पथ मिलता है, यह ClawHub, अपलोड किए गए, Git, स्थानीय, अपडेट और
    निर्भरता-इंस्टॉलर पथों पर लागू होती है तथा कमांड के मान्य निर्णय न लौटाने पर
    बंद अवस्था में विफल होती है।
  </Accordion>
  <Accordion title="सीक्रेट इंजेक्शन का दायरा">
    `skills.entries.*.env` और `skills.entries.*.apiKey` केवल उस एजेंट टर्न के लिए
    **होस्ट** प्रक्रिया में सीक्रेट इंजेक्ट करते हैं — सैंडबॉक्स में नहीं। सीक्रेट को
    प्रॉम्प्ट और लॉग से बाहर रखें।
  </Accordion>
</AccordionGroup>

व्यापक खतरा मॉडल और सुरक्षा जाँच-सूचियों के लिए
[सुरक्षा](/hi/gateway/security) देखें।

## SKILL.md प्रारूप

प्रत्येक Skill के फ्रंटमैटर में कम-से-कम `name` और `description` आवश्यक हैं:

```markdown
---
name: image-lab
description: प्रदाता-समर्थित इमेज कार्यप्रवाह के माध्यम से इमेज जनरेट या संपादित करें
---

जब उपयोगकर्ता इमेज जनरेट करने के लिए कहे, तो `image_generate` टूल का उपयोग करें...
```

<Note>
  OpenClaw [AgentSkills](https://agentskills.io) विनिर्देश का पालन करता है। फ्रंटमैटर को
  पहले YAML के रूप में पार्स किया जाता है; यदि यह विफल हो, तो यह केवल-एकल-पंक्ति
  पार्सर पर फ़ॉलबैक करता है। नेस्ट किए गए `metadata` ब्लॉक (बहु-पंक्ति YAML मैपिंग सहित)
  JSON स्ट्रिंग में समतल किए जाते हैं और JSON5 के रूप में फिर से पार्स होते हैं, इसलिए
  [गेटिंग](#gating) के अंतर्गत दिखाया गया ब्लॉक प्रारूप काम करता है। Skill फ़ोल्डर पथ का संदर्भ देने के लिए
  बॉडी में `{baseDir}` का उपयोग करें।
</Note>

### वैकल्पिक फ्रंटमैटर कुंजियाँ

<ParamField path="homepage" type="string">
  macOS Skills UI में "Website" के रूप में दिखाया जाने वाला URL। यह
  `metadata.openclaw.homepage` के माध्यम से भी समर्थित है।
</ParamField>

<ParamField path="user-invocable" type="boolean" default="true">
  जब `true` हो, तो skill उपयोगकर्ता द्वारा आह्वान योग्य स्लैश कमांड के रूप में उपलब्ध होता है।
</ParamField>

<ParamField path="disable-model-invocation" type="boolean" default="false">
  जब `true` हो, तो OpenClaw skill के निर्देशों को एजेंट के सामान्य
  प्रॉम्प्ट से बाहर रखता है। जब `user-invocable`
  भी `true` हो, तब भी skill स्लैश कमांड के रूप में उपलब्ध रहता है।
</ParamField>

<ParamField path="command-dispatch" type='"tool"'>
  `tool` पर सेट होने पर, स्लैश कमांड मॉडल को बायपास करके
  सीधे किसी पंजीकृत टूल को भेजा जाता है।
</ParamField>

<ParamField path="command-tool" type="string">
  `command-dispatch: tool` सेट होने पर आह्वान किए जाने वाले टूल का नाम।
</ParamField>

<ParamField path="command-arg-mode" type='"raw"' default="raw">
  टूल डिस्पैच के लिए, बिना किसी
  कोर पार्सिंग के कच्ची आर्ग्युमेंट स्ट्रिंग टूल को अग्रेषित करता है। टूल को
  `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }` प्राप्त होता है।
</ParamField>

## गेटिंग

OpenClaw लोड के समय `metadata.openclaw` (फ्रंटमैटर में एम्बेड किया गया JSON5 ऑब्जेक्ट,
ऊपर पार्सिंग नोट देखें) का उपयोग करके skills को फ़िल्टर करता है। बिना
`metadata.openclaw` ब्लॉक वाला skill हमेशा पात्र होता है, जब तक कि उसे स्पष्ट रूप से अक्षम न किया गया हो।

```markdown
---
name: image-lab
description: प्रदाता-समर्थित इमेज वर्कफ़्लो के माध्यम से इमेज जनरेट या संपादित करें
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
---
```

<ParamField path="always" type="boolean">
  जब `true` हो, तो skill को हमेशा शामिल करें और अन्य सभी गेट छोड़ दें।
</ParamField>

<ParamField path="emoji" type="string">
  macOS Skills UI में दिखाया जाने वाला वैकल्पिक इमोजी।
</ParamField>

<ParamField path="homepage" type="string">
  macOS Skills UI में "Website" के रूप में दिखाया जाने वाला वैकल्पिक URL।
</ParamField>

<ParamField path="os" type='("darwin" | "linux" | "win32")[]'>
  प्लेटफ़ॉर्म फ़िल्टर। सेट होने पर, skill केवल सूचीबद्ध OS पर पात्र होता है।
</ParamField>

<ParamField path="requires.bins" type="string[]">
  प्रत्येक बाइनरी `PATH` पर मौजूद होनी चाहिए।
</ParamField>

<ParamField path="requires.anyBins" type="string[]">
  कम-से-कम एक बाइनरी `PATH` पर मौजूद होनी चाहिए।
</ParamField>

<ParamField path="requires.env" type="string[]">
  प्रत्येक एनवायरनमेंट वेरिएबल प्रक्रिया में मौजूद होना चाहिए या कॉन्फ़िगरेशन के माध्यम से प्रदान किया जाना चाहिए।
</ParamField>

<ParamField path="requires.config" type="string[]">
  प्रत्येक `openclaw.json` पथ ट्रूथी होना चाहिए।
</ParamField>

<ParamField path="primaryEnv" type="string">
  `skills.entries.<name>.apiKey` से संबद्ध एनवायरनमेंट वेरिएबल का नाम।
</ParamField>

<ParamField path="install" type="object[]">
  macOS Skills UI द्वारा उपयोग किए जाने वाले वैकल्पिक इंस्टॉलर विनिर्देश (brew / node / go / uv / download)।
</ParamField>

<Note>
  `metadata.openclaw` अनुपस्थित होने पर पुराने `metadata.clawdbot` ब्लॉक अभी भी स्वीकार किए जाते हैं,
  ताकि पहले से इंस्टॉल किए गए skills अपने
  निर्भरता गेट और इंस्टॉलर संकेत बनाए रखें। नए skills को
  `metadata.openclaw` का उपयोग करना चाहिए।
</Note>

### इंस्टॉलर विनिर्देश

इंस्टॉलर विनिर्देश macOS Skills UI को बताते हैं कि निर्भरता कैसे इंस्टॉल करनी है:

```markdown
---
name: gemini
description: कोडिंग सहायता और Google खोज लुकअप के लिए Gemini CLI का उपयोग करें।
metadata:
  {
    "openclaw":
      {
        "emoji": "♊️",
        "requires": { "bins": ["gemini"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gemini-cli",
              "bins": ["gemini"],
              "label": "Gemini CLI इंस्टॉल करें (brew)",
            },
          ],
      },
  }
---
```

<AccordionGroup>
  <Accordion title="इंस्टॉलर चयन नियम">
    - जब कई इंस्टॉलर सूचीबद्ध हों, तो Gateway एक पसंदीदा
      विकल्प चुनता है (उपलब्ध होने पर brew, अन्यथा node)।
    - यदि सभी इंस्टॉलर `download` हों, तो OpenClaw प्रत्येक प्रविष्टि सूचीबद्ध करता है ताकि आप
      सभी उपलब्ध आर्टिफ़ैक्ट देख सकें।
    - प्लेटफ़ॉर्म के अनुसार फ़िल्टर करने के लिए विनिर्देशों में `os: ["darwin"|"linux"|"win32"]` शामिल किया जा सकता है।
    - Node इंस्टॉल `openclaw.json` में `skills.install.nodeManager` का पालन करते हैं
      (डिफ़ॉल्ट: npm; विकल्प: npm / pnpm / yarn / bun)। यह केवल skill
      इंस्टॉल को प्रभावित करता है; Gateway रनटाइम फिर भी Node होना चाहिए।
    - Gateway इंस्टॉलर वरीयता: Homebrew → uv → कॉन्फ़िगर किया गया node मैनेजर →
      go → download।
  </Accordion>
  <Accordion title="प्रति-इंस्टॉलर विवरण">
    - **Homebrew:** OpenClaw, Homebrew को स्वतः इंस्टॉल नहीं करता या brew
      फ़ॉर्मूलों को सिस्टम पैकेज कमांड में रूपांतरित नहीं करता। `brew` के बिना Linux कंटेनरों में,
      केवल brew वाले इंस्टॉलर छिपे रहते हैं; कस्टम इमेज का उपयोग करें या
      निर्भरता को मैन्युअल रूप से इंस्टॉल करें।
    - **Go:** स्वचालित skill इंस्टॉल के लिए OpenClaw को Go 1.21 या उससे नया संस्करण चाहिए।
      यदि `go` अनुपस्थित है और Homebrew उपलब्ध है, तो OpenClaw पहले Homebrew के माध्यम से Go
      इंस्टॉल करता है; Homebrew के बिना Linux पर, जब रीफ़्रेश किया गया `golang-go`
      कैंडिडेट न्यूनतम संस्करण पूरा करता है, तब वह इसके बजाय root के रूप में या पासवर्ड-रहित `sudo` के माध्यम से `apt-get`
      का उपयोग कर सकता है। निर्भरता के लिए वास्तविक `go install` हमेशा
      एक समर्पित OpenClaw-प्रबंधित bin डायरेक्टरी को लक्षित करता है
      (नए इंस्टॉल पर Homebrew का `bin`, अन्यथा `~/.local/bin`), न कि
      आपके कॉन्फ़िगर किए गए `GOBIN` को — आपके अपने `GOBIN`, `GOPATH`, और `GOTOOLCHAIN`
      एनवायरनमेंट वेरिएबल पढ़े जाते हैं, लेकिन कभी ओवरराइट नहीं किए जाते।
    - **डाउनलोड:** `url` (आवश्यक), `archive` (`tar.gz` | `tar.bz2` | `zip`),
      `extract` (डिफ़ॉल्ट: आर्काइव मिलने पर स्वतः), `stripComponents`,
      `targetDir` (डिफ़ॉल्ट: `~/.openclaw/tools/<skillKey>`)।
  </Accordion>
  <Accordion title="सैंडबॉक्सिंग नोट्स">
    skill लोड के समय `requires.bins` को **होस्ट** पर जाँचा जाता है। यदि कोई एजेंट
    सैंडबॉक्स में चलता है, तो बाइनरी **कंटेनर के भीतर** भी मौजूद होनी चाहिए।
    इसे `agents.defaults.sandbox.docker.setupCommand` या कस्टम
    इमेज के माध्यम से इंस्टॉल करें। `setupCommand` कंटेनर बनने के बाद एक बार चलता है और इसके लिए
    नेटवर्क ईग्रेस, लिखने योग्य root FS और सैंडबॉक्स में root उपयोगकर्ता आवश्यक हैं।
  </Accordion>
</AccordionGroup>

## कॉन्फ़िगरेशन ओवरराइड

`~/.openclaw/openclaw.json` में `skills.entries` के अंतर्गत बंडल किए गए या प्रबंधित skills को
टॉगल और कॉन्फ़िगर करें:

```json5
{
  skills: {
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
        config: {
          endpoint: "https://example.invalid",
          model: "nano-pro",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<ParamField path="enabled" type="boolean">
  `false` बंडल या इंस्टॉल होने पर भी skill को अक्षम कर देता है। `coding-agent`
  बंडल किया गया skill ऑप्ट-इन है — `skills.entries.coding-agent.enabled: true` सेट करें
  और सुनिश्चित करें कि `claude`, `codex`, `opencode`, या कोई अन्य समर्थित CLI
  इंस्टॉल और प्रमाणित है।
</ParamField>

<ParamField path="apiKey" type='string | { source, provider, id }'>
  `metadata.openclaw.primaryEnv` घोषित करने वाले skills के लिए सुविधा फ़ील्ड।
  सादे टेक्स्ट स्ट्रिंग या SecretRef ऑब्जेक्ट का समर्थन करता है।
</ParamField>

<ParamField path="env" type="Record<string, string>">
  एजेंट रन के लिए इंजेक्ट किए गए एनवायरनमेंट वेरिएबल। केवल तभी इंजेक्ट किए जाते हैं जब
  वेरिएबल प्रक्रिया में पहले से सेट न हो।
</ParamField>

<ParamField path="config" type="object">
  कस्टम प्रति-skill कॉन्फ़िगरेशन फ़ील्ड के लिए वैकल्पिक संग्रह।
</ParamField>

<ParamField path="allowBundled" type="string[]">
  केवल **बंडल किए गए** skills के लिए वैकल्पिक अनुमति-सूची। सेट होने पर, सूची में मौजूद केवल बंडल किए गए skills
  ही पात्र होते हैं। प्रबंधित और वर्कस्पेस skills अप्रभावित रहते हैं।
</ParamField>

<Note>
  डिफ़ॉल्ट रूप से कॉन्फ़िगरेशन कुंजियाँ **skill नाम** से मेल खाती हैं। यदि कोई skill
  `metadata.openclaw.skillKey` परिभाषित करता है, तो इसके बजाय `skills.entries` के अंतर्गत उस कुंजी का उपयोग करें।
  हाइफ़न वाले नामों को उद्धरण-चिह्नों में रखें: JSON5 उद्धृत कुंजियों की अनुमति देता है।
</Note>

## एनवायरनमेंट इंजेक्शन

जब कोई एजेंट रन शुरू होता है, OpenClaw:

<Steps>
  <Step title="skill मेटाडेटा पढ़ता है">
    OpenClaw गेटिंग नियम, अनुमति-सूचियाँ और कॉन्फ़िगरेशन ओवरराइड लागू करके
    एजेंट के लिए प्रभावी skill सूची निर्धारित करता है।
  </Step>
  <Step title="एनवायरनमेंट वेरिएबल और API कुंजियाँ इंजेक्ट करता है">
    रन की अवधि के लिए `skills.entries.<key>.env` और `skills.entries.<key>.apiKey` को
    `process.env` पर लागू किया जाता है।
  </Step>
  <Step title="सिस्टम प्रॉम्प्ट बनाता है">
    पात्र skills को एक संक्षिप्त XML ब्लॉक में संकलित करके
    सिस्टम प्रॉम्प्ट में इंजेक्ट किया जाता है।
  </Step>
  <Step title="एनवायरनमेंट पुनर्स्थापित करता है">
    रन समाप्त होने के बाद मूल एनवायरनमेंट पुनर्स्थापित किया जाता है।
  </Step>
</Steps>

<Warning>
  एनवायरनमेंट इंजेक्शन **होस्ट** एजेंट रन तक सीमित है, सैंडबॉक्स तक नहीं। किसी
  सैंडबॉक्स के भीतर `env` और `apiKey` का कोई प्रभाव नहीं होता। सैंडबॉक्स किए गए रन में
  सीक्रेट पास करने के तरीके के लिए [Skills कॉन्फ़िगरेशन](/hi/tools/skills-config#sandboxed-skills-and-env-vars) देखें।
</Warning>

बंडल किए गए `claude-cli` बैकएंड के लिए, OpenClaw उसी
पात्र skill स्नैपशॉट को अस्थायी Claude Code plugin के रूप में भी तैयार करता है और उसे
`--plugin-dir` के माध्यम से पास करता है। अन्य CLI बैकएंड केवल प्रॉम्प्ट कैटलॉग का उपयोग करते हैं।

## स्नैपशॉट और रीफ़्रेश

OpenClaw पात्र skills का स्नैपशॉट **सत्र शुरू होने पर** लेता है और सत्र के
बाद के सभी टर्न के लिए उस सूची का पुनः उपयोग करता है। skills या कॉन्फ़िगरेशन में बदलाव
अगले नए सत्र में प्रभावी होते हैं।

सत्र के बीच में skills दो स्थितियों में रीफ़्रेश होते हैं:

- skills वॉचर `SKILL.md` में बदलाव पहचानता है।
- कोई नया पात्र रिमोट node कनेक्ट होता है।

रीफ़्रेश की गई सूची अगले एजेंट टर्न में अपनाई जाती है। यदि प्रभावी एजेंट
अनुमति-सूची बदलती है, तो OpenClaw दृश्यमान skills को
संरेखित रखने के लिए स्नैपशॉट रीफ़्रेश करता है।

<AccordionGroup>
  <Accordion title="Skills वॉचर">
    डिफ़ॉल्ट रूप से, OpenClaw skill फ़ोल्डरों पर नज़र रखता है और
    `SKILL.md` फ़ाइलें बदलने पर स्नैपशॉट को अपडेट करता है। `skills.load` के अंतर्गत कॉन्फ़िगर करें:

    ```json5
    {
      skills: {
        load: {
          extraDirs: ["~/Projects/agent-scripts/skills"],
          allowSymlinkTargets: ["~/Projects/manager/skills"],
          watch: true, // default
        },
      },
    }
    ```

    वॉचर इवेंट अंतर्निहित 250 ms डीबाउंस का उपयोग करते हैं। ऐसे सुनियोजित सिमलिंक लेआउट के लिए
    `allowSymlinkTargets` का उपयोग करें जहाँ कोई skill
    root सिमलिंक कॉन्फ़िगर किए गए root से बाहर इंगित करता है, उदाहरण के लिए
    `<workspace>/skills/manager -> ~/Projects/manager/skills`।
    `skills.workshop.allowSymlinkTargetWrites` को केवल तभी सक्षम करें जब Skill Workshop को
    उन विश्वसनीय सिमलिंक पथों के माध्यम से प्रस्ताव भी लागू करने चाहिए।

  </Accordion>
  <Accordion title="रिमोट macOS nodes (Linux gateway)">
    यदि Gateway Linux पर चलता है, लेकिन `system.run` की अनुमति वाला कोई **macOS node**
    कनेक्ट है, तो आवश्यक बाइनरी उस node पर मौजूद होने पर OpenClaw केवल macOS वाले skills को पात्र मान सकता है।
    एजेंट को उन skills को `host=node` के साथ `exec` टूल के माध्यम से चलाना चाहिए।

    ऑफ़लाइन nodes रिमोट-केवल skills को दृश्यमान **नहीं** बनाते। यदि कोई node
    bin प्रोब का उत्तर देना बंद कर देता है, तो OpenClaw उसके कैश किए गए bin मिलान साफ़ कर देता है।

  </Accordion>
</AccordionGroup>

## टोकन प्रभाव

जब skills पात्र होते हैं, तो OpenClaw सिस्टम
प्रॉम्प्ट में एक संक्षिप्त XML ब्लॉक इंजेक्ट करता है। लागत नियतात्मक होती है और प्रति skill रैखिक रूप से बढ़ती है:

- **मूल ओवरहेड** (केवल तब जब 1+ skills पात्र हों): परिचयात्मक
  गद्य का एक निश्चित ब्लॉक और `<available_skills>` रैपर।
- **प्रति skill:** ~97 वर्ण + आपके `name`, `description`, और `location`
  फ़ील्ड की लंबाई।
- XML एस्केपिंग `& < > " '` को एंटिटी में विस्तारित करती है, जिससे प्रत्येक
  उपस्थिति पर कुछ वर्ण जुड़ते हैं।
- ~4 वर्ण/टोकन पर, फ़ील्ड लंबाई से पहले प्रति skill 97 वर्ण ≈ 24 टोकन।

यदि रेंडर किया गया ब्लॉक कॉन्फ़िगर किए गए प्रॉम्प्ट बजट
(`skills.limits.maxSkillsPromptChars`) से अधिक हो जाए, तो OpenClaw पहले उतनी skill
पहचानें (नाम, स्थान और संस्करण) सुरक्षित रखता है, जितनी विवरण-रहित संक्षिप्त प्रारूप
में समा सकती हैं। इसके बाद, वह शेष बजट का उपयोग संक्षिप्त विवरणों के लिए करता है। यदि
विवरण के लिए कोई बजट शेष नहीं रहता, तो विवरण छोड़ दिए जाते हैं। जब भी संक्षिप्त प्रारूपण या सूची
को छोटा करना आवश्यक होता है, प्रॉम्प्ट में `openclaw skills check` की ओर इंगित करने वाला एक
नोट शामिल होता है।

प्रॉम्प्ट का अतिरिक्त भार न्यूनतम रखने के लिए विवरण छोटे और वर्णनात्मक रखें।

## संबंधित

<CardGroup cols={2}>
  <Card title="Skills बनाना" href="/hi/tools/creating-skills" icon="hammer">
    कस्टम skill बनाने के लिए चरण-दर-चरण मार्गदर्शिका।
  </Card>
  <Card title="Skill कार्यशाला" href="/hi/tools/skill-workshop" icon="flask">
    एजेंट द्वारा तैयार की गई skills के लिए प्रस्ताव कतार।
  </Card>
  <Card title="Skills कॉन्फ़िगरेशन" href="/hi/tools/skills-config" icon="gear">
    पूर्ण `skills.*` कॉन्फ़िगरेशन स्कीमा और एजेंट अनुमति-सूचियाँ।
  </Card>
  <Card title="स्लैश कमांड" href="/hi/tools/slash-commands" icon="terminal">
    skill स्लैश कमांड कैसे पंजीकृत और रूट किए जाते हैं।
  </Card>
  <Card title="ClawHub" href="/hi/clawhub" icon="cloud">
    सार्वजनिक रजिस्ट्री पर skills ब्राउज़ और प्रकाशित करें।
  </Card>
  <Card title="Plugins" href="/hi/tools/plugin" icon="plug">
    Plugins उन टूल के साथ skills प्रदान कर सकते हैं जिनका वे दस्तावेज़ीकरण करते हैं।
  </Card>
</CardGroup>
