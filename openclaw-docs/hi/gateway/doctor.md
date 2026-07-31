---
read_when:
    - डॉक्टर माइग्रेशन जोड़ना या संशोधित करना
    - ब्रेकिंग कॉन्फ़िगरेशन बदलाव प्रस्तुत करना
sidebarTitle: Doctor
summary: 'Doctor कमांड: स्वास्थ्य जाँच, कॉन्फ़िगरेशन माइग्रेशन और सुधार के चरण'
title: डॉक्टर
x-i18n:
    generated_at: "2026-07-27T20:54:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f599553a2455759cd0fe56bafbc16948f7ab4d381d344b08a496bf19c9dc636
    source_path: gateway/doctor.md
    workflow: 16
---

`openclaw doctor` OpenClaw के लिए मरम्मत और माइग्रेशन टूल है। यह पुराने config/state को ठीक करता है, स्वास्थ्य की जाँच करता है और कार्रवाई योग्य मरम्मत चरण प्रदान करता है।

## त्वरित शुरुआत

```bash
openclaw doctor
```

### हेडलेस और ऑटोमेशन मोड

<Tabs>
  <Tab title="--yes">
    ```bash
    openclaw doctor --yes
    ```

    संकेत दिए बिना डिफ़ॉल्ट स्वीकार करें (लागू होने पर रीस्टार्ट/service/sandbox मरम्मत चरणों सहित)।

  </Tab>
  <Tab title="--fix">
    ```bash
    openclaw doctor --fix
    ```

    संकेत दिए बिना सुझाई गई मरम्मत लागू करें (`--repair` एक उपनाम है)।

  </Tab>
  <Tab title="--lint">
    ```bash
    openclaw doctor --lint
    openclaw doctor --lint --json
    ```

    CI या प्रीफ़्लाइट ऑटोमेशन के लिए संरचित स्वास्थ्य जाँच चलाएँ। केवल-पठन: कोई
    संकेत, मरम्मत, माइग्रेशन, रीस्टार्ट या state लेखन नहीं।

  </Tab>
  <Tab title="--fix --force">
    ```bash
    openclaw doctor --fix --force
    ```

    आक्रामक मरम्मत भी लागू करें (कस्टम supervisor configs को अधिलेखित करता है)।

  </Tab>
  <Tab title="--non-interactive">
    ```bash
    openclaw doctor --non-interactive
    ```

    संकेतों के बिना चलाएँ और केवल सुरक्षित माइग्रेशन लागू करें (config सामान्यीकरण +
    ऑन-डिस्क state स्थानांतरण)। मानव पुष्टि की आवश्यकता वाली रीस्टार्ट/service/sandbox
    कार्रवाइयाँ छोड़ देता है। पुराने state माइग्रेशन का पता लगने पर वे फिर भी स्वचालित रूप से चलते हैं।

  </Tab>
  <Tab title="--deep">
    ```bash
    openclaw doctor --deep
    ```

    अतिरिक्त gateway इंस्टॉल के लिए सिस्टम services को स्कैन करें (launchd/systemd/schtasks)।

  </Tab>
</Tabs>

लिखने से पहले परिवर्तनों की समीक्षा करने के लिए, पहले config फ़ाइल खोलें:

```bash
cat ~/.openclaw/openclaw.json
```

## केवल-पठन lint मोड

`openclaw doctor --lint`, `openclaw doctor --fix` का ऑटोमेशन-अनुकूल सहोदर है।
दोनों समान Doctor नियम रजिस्ट्री साझा करते हैं, लेकिन नियमों का चयन या उन पर कार्रवाई
एक ही तरह से नहीं करते:

| मोड                     | संकेत   | config/state में लेखन     | आउटपुट                 | इसका उपयोग करें                      |
| ------------------------ | --------- | ----------------------- | ---------------------- | ------------------------------- |
| `openclaw doctor`        | हाँ       | नहीं                      | अनुकूल स्वास्थ्य रिपोर्ट | किसी व्यक्ति द्वारा स्थिति जाँचने के लिए         |
| `openclaw doctor --fix`  | कभी-कभी | हाँ, मरम्मत नीति के साथ | अनुकूल मरम्मत लॉग    | स्वीकृत मरम्मत लागू करने के लिए       |
| `openclaw doctor --lint` | नहीं        | नहीं                      | संरचित निष्कर्ष    | CI, प्रीफ़्लाइट और समीक्षा गेट के लिए |

डिफ़ॉल्ट `doctor --lint` व्यापक-सुरक्षित ऑटोमेशन प्रोफ़ाइल चलाता है: ऐसी जाँचें जो
स्थिर, स्थानीय और CI या प्रीफ़्लाइट आउटपुट में उपयोगी हैं। यह उन ऑप्ट-इन जाँचों को
छोड़ देता है जो सलाहकारी, पर्यावरण-संवेदनशील, लाइव-service पर निर्भर, account/workspace
इन्वेंटरी या ऐतिहासिक सफ़ाई से संबंधित हैं। उन ऑप्ट-इन जाँचों सहित पूर्ण पंजीकृत lint
ऑडिट के लिए `doctor --lint --all`, या लक्षित जाँच के लिए `--only <id>` का उपयोग करें।

`doctor --fix` lint की डिफ़ॉल्ट प्रोफ़ाइल का उपयोग नहीं करता और
`--all` स्वीकार नहीं करता। यह Doctor का क्रमबद्ध मरम्मत पथ चलाता है: आधुनिक
स्वास्थ्य जाँचें वैकल्पिक `repair()` कार्यान्वयन प्रदान कर सकती हैं, जबकि पुराने
क्षेत्र अभी भी अपने पुराने Doctor मरम्मत प्रवाह का उपयोग करते हैं। कुछ lint निष्कर्ष
जानबूझकर केवल निदानात्मक होते हैं, इसलिए `--lint --all` में किसी जाँच के दिखाई देने
का अर्थ यह नहीं है कि `--fix` उस क्षेत्र में बदलाव करेगा। अनुबंध
`detect()` (निष्कर्षों की रिपोर्ट करता है) को `repair()` (परिवर्तनों/diffs/दुष्प्रभावों
की रिपोर्ट करता है) से अलग रखता है, जिससे lint जाँचों को बदलाव योजनाकार बनाए बिना
भविष्य के `doctor --fix --dry-run` के लिए मार्ग खुला रहता है।

कुछ अंतर्निहित जाँचें आंतरिक रूप से डिफ़ॉल्ट-अक्षम हैं, ताकि वे डिफ़ॉल्ट
`doctor --lint` ऑटोमेशन प्रोफ़ाइल का भाग बने बिना `--all`,
`--only` और Doctor मरम्मत प्रवाहों के लिए उपलब्ध रहें। निष्कर्ष की गंभीरता
फिर भी प्रत्येक निष्कर्ष के लिए उत्सर्जित होती है (`info`, `warning`,
या `error`); डिफ़ॉल्ट चयन कोई गंभीरता स्तर नहीं है।

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --only core/doctor/gateway-config --json
```

JSON आउटपुट फ़ील्ड:

- `ok`: क्या कोई निष्कर्ष चयनित गंभीरता सीमा तक पहुँचा
- `checksRun` / `checksSkipped`: गणनाएँ (प्रोफ़ाइल, `--only` या `--skip` के कारण छोड़ी गईं)
- `findings`: `checkId`, `severity`, `message` और वैकल्पिक `path`, `line`, `column`, `ocPath`, `source`, `target`, `requirement`, `fixHint` के साथ संरचित निदान

निकास कोड:

| कोड | अर्थ                                                  |
| ---- | -------------------------------------------------------- |
| `0`  | चयनित सीमा पर या उससे ऊपर कोई निष्कर्ष नहीं           |
| `1`  | एक या अधिक निष्कर्ष चयनित सीमा तक पहुँचे          |
| `2`  | निष्कर्ष उत्सर्जित होने से पहले command/runtime विफलता |

फ़्लैग:

- `--severity-min info|warning|error` (डिफ़ॉल्ट `warning`): क्या प्रिंट होता है और किस कारण गैर-शून्य निकास होता है, दोनों को नियंत्रित करता है।
- `--all`: डिफ़ॉल्ट ऑटोमेशन समूह से बाहर रखी गई ऑप्ट-इन जाँचों सहित प्रत्येक पंजीकृत lint जाँच चलाता है।
- `--only <id>` (दोहराने योग्य): केवल नामित जाँच id चलाएँ; अज्ञात id को त्रुटि निष्कर्ष के रूप में रिपोर्ट किया जाता है।
- `--skip <id>` (दोहराने योग्य): शेष रन को सक्रिय रखते हुए किसी जाँच को बाहर रखें।
- `--json`, `--severity-min`, `--all`, `--only` और `--skip` के लिए `--lint` आवश्यक है; साधारण `openclaw doctor` और `--fix` रन इन्हें अस्वीकार करते हैं।

## यह क्या करता है (सारांश)

<AccordionGroup>
  <Accordion title="स्वास्थ्य, UI और अपडेट">
    - git इंस्टॉल के लिए वैकल्पिक प्री-फ़्लाइट अपडेट (केवल इंटरैक्टिव)।
    - UI प्रोटोकॉल ताज़गी जाँच (प्रोटोकॉल schema नया होने पर Control UI को दोबारा बनाती है)।
    - स्वास्थ्य जाँच + रीस्टार्ट संकेत।
    - केवल समस्या वाले skill और plugin नोट्स; स्वस्थ इन्वेंटरी `openclaw skills check` और `openclaw plugins list` में रहती है।

  </Accordion>
  <Accordion title="Config और माइग्रेशन">
    - पुराने value shapes के लिए config सामान्यीकरण।
    - पुराने समतल `talk.*` फ़ील्ड से `talk.provider` + `talk.providers.<provider>` में Talk config माइग्रेशन।
    - पुराने Chrome extension configs और Chrome MCP तत्परता के लिए browser माइग्रेशन जाँच।
    - OpenCode provider override चेतावनियाँ (`models.providers.opencode` / `opencode-zen` / `opencode-go`)।
    - पुराना OpenAI Codex provider/profile माइग्रेशन (`openai-codex` → `openai`) और पुराने `models.providers.openai-codex` के लिए shadowing चेतावनियाँ।
    - OpenAI Codex OAuth profiles के लिए OAuth TLS पूर्वापेक्षा जाँच।
    - जब `plugins.allow` प्रतिबंधात्मक हो, लेकिन tool नीति फिर भी wildcard या plugin-स्वामित्व वाले tools माँगे, तब Plugin/tool allowlist चेतावनियाँ।
    - पुराना ऑन-डिस्क state माइग्रेशन (sessions/agent dir/WhatsApp auth)।
    - पुराना plugin manifest अनुबंध key माइग्रेशन (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`)।
    - पुराना Cron store माइग्रेशन (`jobId`, `schedule.cron`, शीर्ष-स्तरीय delivery/payload फ़ील्ड, payload `provider`, `notify: true` Webhook fallback jobs)।
    - `agents.defaults`, `agents.entries.*` और `models.providers.*` (प्रति-model प्रविष्टियों सहित) में Codex CLI runtime pin मरम्मत (`agentRuntime.id: "codex-cli"` → `"codex"`)।
    - plugins सक्षम होने पर पुराने plugin config की सफ़ाई; `plugins.enabled=false` होने पर पुराने plugin संदर्भों को निष्क्रिय containment config के रूप में संरक्षित रखा जाता है।

  </Accordion>
  <Accordion title="State और अखंडता">
    - Session lock फ़ाइल का निरीक्षण और पुराने lock की सफ़ाई।
    - प्रभावित 2026.4.24 builds द्वारा बनाई गई डुप्लिकेट prompt-rewrite branches के लिए Session transcript मरम्मत।
    - अटके हुए मुख्य-session और subagent restart-recovery tombstone का पता लगाना। Doctor अवरुद्ध sessions की रिपोर्ट करता है और केवल उन पुराने aborted flags की मरम्मत करता है जो मौजूदा tombstone से टकराते हैं; यह स्वचालित recovery को दोबारा सक्षम नहीं करता।
    - State अखंडता और अनुमतियों की जाँच (sessions, transcripts, state dir)।
    - स्थानीय रूप से चलाते समय Config फ़ाइल अनुमति जाँच (chmod 600)।
    - Model auth स्वास्थ्य: OAuth समाप्ति की जाँच करता है, समाप्त होने वाले tokens को refresh कर सकता है और auth-profile cooldown/अक्षम अवस्थाओं की रिपोर्ट करता है।

  </Accordion>
  <Accordion title="Gateway, services और supervisors">
    - sandboxing सक्षम होने पर Sandbox image मरम्मत।
    - पुराना service माइग्रेशन और अतिरिक्त Gateway का पता लगाना।
    - Matrix channel का पुराना state माइग्रेशन (`--fix` / `--repair` मोड में)।
    - Gateway runtime जाँच (service इंस्टॉल है, लेकिन चल नहीं रही; कैश किया गया launchd label)।
    - Channel स्थिति चेतावनियाँ (चल रहे Gateway से जाँची गईं)।
    - Channel-विशिष्ट अनुमति जाँचें `openclaw channels capabilities` के अंतर्गत रहती हैं; उदाहरण के लिए, Discord voice channel अनुमतियों का ऑडिट `openclaw channels capabilities --channel discord --target channel:<channel-id>` से किया जाता है।
    - स्थानीय TUI clients के अभी भी चलने के दौरान खराब Gateway event-loop स्वास्थ्य के लिए WhatsApp प्रत्युत्तरशीलता जाँच; `--fix` केवल सत्यापित स्थानीय TUI clients को रोकता है।
    - प्राथमिक models, fallbacks, image/video generation models, Heartbeat/subagent/Compaction overrides, hooks, channel model overrides और session route pins में पुराने `openai-codex/*` model refs के लिए Codex route मरम्मत; `--fix` उन्हें `openai/*` में दोबारा लिखता है, `openai-codex:*` auth profiles/order को `openai:*` में माइग्रेट करता है, पुराने session/पूरे-agent runtime pins हटाता है और मरम्मत किए गए प्रभावी route को यह निर्धारित करने देता है कि Codex संगत है या नहीं।
    - वैकल्पिक मरम्मत के साथ Supervisor config ऑडिट (launchd/systemd/schtasks)।
    - उन Gateway services के लिए अंतर्निहित proxy environment की सफ़ाई, जिन्होंने इंस्टॉल या अपडेट के दौरान shell `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` मान कैप्चर किए थे।
    - Gateway runtime जाँच (असमर्थित पुराने Bun services, version-manager paths)।
    - Gateway port टकराव निदान (डिफ़ॉल्ट `18789`)।

  </Accordion>
  <Accordion title="Auth, सुरक्षा और pairing">
    - खुली DM नीतियों के लिए सुरक्षा चेतावनियाँ।
    - स्थानीय token मोड के लिए Gateway auth जाँच (कोई token स्रोत मौजूद न होने पर token निर्माण की पेशकश करता है; token SecretRef configs को अधिलेखित नहीं करता)।
    - Device pairing समस्या का पता लगाना (लंबित पहली बार के pair अनुरोध, लंबित role/scope upgrades, पुराने स्थानीय device-token cache का विचलन और paired-record auth विचलन)।

  </Accordion>
  <Accordion title="Workspace और shell">
    - Linux पर systemd linger जाँच।
    - Workspace bootstrap फ़ाइल आकार जाँच (context फ़ाइलों के लिए truncation/सीमा के निकट होने की चेतावनियाँ)।
    - डिफ़ॉल्ट agent के लिए Skills तत्परता जाँच; अनुपलब्ध bins, env, config या OS आवश्यकताओं वाले अनुमत skills की रिपोर्ट करता है और `--fix`, `skills.entries` में अनुपलब्ध skills को अक्षम कर सकता है।
    - Shell completion स्थिति जाँच और स्वचालित इंस्टॉल/अपग्रेड।
    - Memory search embedding provider तत्परता जाँच (स्थानीय model, remote API key या QMD binary)।
    - Source install जाँच (pnpm workspace असंगति, अनुपलब्ध UI assets, अनुपलब्ध tsx binary)।
    - अपडेट किया हुआ config + wizard metadata लिखता है।

  </Accordion>
</AccordionGroup>

## Dreams UI बैकफ़िल और रीसेट

  Control UI के Dreams दृश्य में grounded dreaming कार्यप्रवाह के लिए **Backfill**, **Reset**, और **Clear Grounded** क्रियाएँ शामिल हैं। ये Gateway की doctor-शैली वाली RPC विधियों का उपयोग करती हैं, लेकिन `openclaw doctor` CLI मरम्मत/माइग्रेशन का भाग **नहीं** हैं।

  | क्रिया          | यह क्या करती है                                                                                                                                                     |
  | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | Backfill       | सक्रिय कार्यक्षेत्र में ऐतिहासिक `memory/YYYY-MM-DD.md` फ़ाइलों को स्कैन करती है, grounded REM डायरी पास चलाती है, और `DREAMS.md` में वापस लिए जा सकने वाले backfill प्रविष्टियाँ लिखती है। |
  | Reset          | `DREAMS.md` से केवल चिह्नित backfill डायरी प्रविष्टियाँ हटाती है।                                                                                                  |
  | Clear Grounded | ऐतिहासिक रीप्ले से केवल वे चरणबद्ध grounded-only अल्पकालिक प्रविष्टियाँ हटाती है, जिनमें अभी तक लाइव स्मरण या दैनिक समर्थन संचित नहीं हुआ है।                           |

  इनमें से कोई भी `MEMORY.md` को संपादित नहीं करती, पूर्ण doctor माइग्रेशन नहीं चलाती, या grounded उम्मीदवारों को स्वयं लाइव अल्पकालिक प्रमोशन स्टोर में चरणबद्ध नहीं करती। grounded ऐतिहासिक रीप्ले को सामान्य डीप प्रमोशन लेन में भेजने के लिए, इसके बजाय CLI प्रवाह का उपयोग करें:

  ```bash
  openclaw memory rem-backfill --path ./memory --stage-short-term
  ```

  यह grounded टिकाऊ उम्मीदवारों को अल्पकालिक dreaming स्टोर में चरणबद्ध करता है, जबकि `DREAMS.md` समीक्षा सतह बनी रहती है।

  ## विस्तृत व्यवहार और औचित्य

  <AccordionGroup>
  <Accordion title="0. वैकल्पिक अपडेट (git इंस्टॉल)">
    यदि यह git checkout है और doctor इंटरैक्टिव रूप से चल रहा है, तो यह doctor चलाने से पहले अपडेट (fetch/rebase/build) करने की पेशकश करता है।
  </Accordion>
  <Accordion title="1. कॉन्फ़िग सामान्यीकरण">
    Doctor पुराने मान आकारों को वर्तमान स्कीमा में सामान्यीकृत करता है। वर्तमान Talk स्पीच कॉन्फ़िग `talk.provider` + `talk.providers.<provider>` है, जिसमें रीयलटाइम वॉइस कॉन्फ़िग `talk.realtime.*` के अंतर्गत है। Doctor पुराने `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` / `talk.apiKey` आकारों को प्रदाता मैप में फिर से लिखता है, और पुराने शीर्ष-स्तरीय रीयलटाइम चयनकर्ताओं (`talk.mode`, `talk.transport`, `talk.brain`, `talk.model`, `talk.voice`) को `talk.realtime` में फिर से लिखता है।

    जब `plugins.allow` खाली नहीं होता और टूल नीति वाइल्डकार्ड या Plugin-स्वामित्व वाली टूल प्रविष्टियों का उपयोग करती है, तब Doctor चेतावनी भी देता है। `tools.allow: ["*"]` केवल वास्तव में लोड होने वाले plugins के टूल से मेल खाता है; यह विशिष्ट Plugin अनुमति-सूची को बायपास नहीं करता।

  </Accordion>
  <Accordion title="2. पुराने कॉन्फ़िग कुंजी माइग्रेशन">
    जब कॉन्फ़िग में सक्रिय माइग्रेशन वाली कोई अप्रचलित कुंजी होती है, तो अन्य कमांड चलने से इनकार करते हैं और आपसे `openclaw doctor` चलाने को कहते हैं। Doctor बताता है कि कौन-सी पुरानी कुंजियाँ मिलीं, लागू किया गया माइग्रेशन दिखाता है, और अपडेट किए गए स्कीमा के साथ `~/.openclaw/openclaw.json` को फिर से लिखता है। Gateway स्टार्टअप पुराने कॉन्फ़िग प्रारूपों को अस्वीकार करता है और आपसे `openclaw doctor --fix` चलाने को कहता है; यह स्टार्टअप पर `openclaw.json` को फिर से नहीं लिखता। Cron जॉब स्टोर माइग्रेशन भी `openclaw doctor --fix` द्वारा संभाले जाते हैं।

    <Note>
      किसी कुंजी को हटाए जाने के बाद Doctor लगभग दो महीने तक ही स्वचालित
      माइग्रेशन उपलब्ध रखता है। अधिक पुरानी विरासती कुंजियों (उदाहरण के लिए मूल
      `routing.queue`, `routing.bindings`, `routing.agents`/`defaultAgentId`,
      `routing.transcribeAudio`, शीर्ष-स्तरीय `agent.*`, या प्री-मल्टी-एजेंट कॉन्फ़िग
      आकार का शीर्ष-स्तरीय `identity`) के लिए अब कोई माइग्रेशन पथ नहीं है;
      उनका उपयोग करने वाला कॉन्फ़िग अब फिर से लिखे जाने के बजाय सत्यापन में विफल
      हो जाता है। Doctor के आगे बढ़ने से पहले उन कुंजियों को वर्तमान कॉन्फ़िग
      संदर्भ के अनुसार मैन्युअल रूप से ठीक करें।
    </Note>

    सक्रिय माइग्रेशन:

    | पुरानी कुंजी                                                                                    | वर्तमान कुंजी                                                                 |
    | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
    | `routing.allowFrom`                                                                              | `channels.whatsapp.allowFrom`                                                |
    | `routing.groupChat.requireMention`                                                               | `channels.whatsapp/telegram/imessage.groups."*".requireMention`             |
    | `routing.groupChat.historyLimit`                                                                 | `messages.groupChat.historyLimit`                                            |
    | `routing.groupChat.mentionPatterns`                                                              | `messages.groupChat.mentionPatterns`                                         |
    | `channels.telegram.requireMention`                                                               | `channels.telegram.groups."*".requireMention`                               |
    | `channels.webchat`, `gateway.webchat`                                                            | हटाया गया (WebChat को सेवानिवृत्त कर दिया गया है)                                                 |
    | `channels.feishu.accounts.<accountId>.botName`                                                   | `channels.feishu.accounts.<accountId>.name`                                 |
    | `session.threadBindings.ttlHours`, `channels.<id>.threadBindings.ttlHours` (और प्रति खाता)      | `...threadBindings.idleHours`                                               |
    | पुरानी `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey`        | `talk.provider` + `talk.providers.<provider>`                               |
    | पुराने शीर्ष-स्तरीय रीयलटाइम Talk चयनकर्ता (`talk.mode`/`talk.transport`/`talk.brain`/`talk.model`/`talk.voice`) | `talk.realtime`                                                              |
    | `messages.tts`                                                                                  | शीर्ष-स्तरीय `tts`                                                              |
    | `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`)                             | `tts.providers.<provider>`                                                   |
    | `messages.tts.provider: "edge"` / `messages.tts.providers.edge`                                  | `tts.provider: "microsoft"` / `tts.providers.microsoft`                    |
    | `tools.exec.security` + `tools.exec.ask`                                                         | `tools.exec.mode`                                                            |
    | `session.idleMinutes`                                                                            | `session.reset.idleMinutes`                                                  |
    | स्पष्ट चैनल ब्लॉक के साथ `messages.responsePrefix`                                           | कॉन्फ़िगर किए गए चैनल/खाते के `responsePrefix` में कॉपी किया गया; अंतर्निहित/कस्टम चैनलों के लिए वैश्विक फ़ॉलबैक बनाए रखा गया |
    | `web.enabled`                                                                                    | `channels.whatsapp.enabled`                                                  |
    | `meta.lastTouchedAt`, हुक इंस्टॉलेशन, Cron स्टोर, बंडल की गई खोज, वैश्विक TTS प्राथमिकता पथ            | साझा SQLite स्थिति                                                       |
    | TTS स्पीकर फ़ील्ड `voice`/`voiceName`/`voiceId`                                                 | `speakerVoice`/`speakerVoiceId`                                              |
    | `channels.<id>.tts.<provider>` / `channels.<id>.accounts.<accountId>.tts.<provider>` (Discord को छोड़कर सभी चैनल)                                          | `...tts.providers.<provider>`                                                |
    | `channels.<id>.voice.tts.<provider>` / `channels.<id>.accounts.<accountId>.voice.tts.<provider>` (Discord सहित सभी चैनल)                          | `...voice.tts.providers.<provider>`                                          |
    | `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`)     | `plugins.entries.voice-call.config.tts.providers.<provider>`                |
    | `plugins.entries.voice-call.config.tts.provider: "edge"` / `...tts.providers.edge`                | `provider: "microsoft"` / `...tts.providers.microsoft`                      |
    | `plugins.entries.voice-call.config.provider: "log"`                                              | `"mock"`                                                                      |
    | `plugins.entries.voice-call.config.twilio.from`                                                  | `plugins.entries.voice-call.config.fromNumber`                              |
    | `plugins.entries.voice-call.config.streaming.sttProvider`                                        | `plugins.entries.voice-call.config.streaming.provider`                      |
    | `plugins.entries.voice-call.config.streaming.openaiApiKey`/`sttModel`/`silenceDurationMs`/`vadThreshold` | `plugins.entries.voice-call.config.streaming.providers.openai.*`             |
    | `models.providers.*.api: "openai"`                                                               | `"openai-completions"` (Gateway स्टार्टअप उन प्रदाताओं को भी छोड़ देता है जिनका `api` विफल होकर बंद होने के बजाय भावी/अज्ञात एनम मान है) |
    | `browser.ssrfPolicy.allowPrivateNetwork`                                                         | `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`                          |
    | `browser.profiles.*.driver: "extension"`                                                         | `"existing-session"`                                                          |
    | `browser.relayBindHost`                                                                          | हटाया गया (पुरानी Chrome एक्सटेंशन रिले सेटिंग)                             |
    | `mcp.servers.*.type` (CLI-मूल उपनाम)                                                        | `mcp.servers.*.transport`                                                    |
    | `mcp.servers.*.disabled`                                                                         | व्युत्क्रम `mcp.servers.*.enabled`                                              |
    | MCP टाइमआउट उपनाम `connectTimeout`/`connect_timeout`/`timeout`                                 | `connectionTimeoutMs`/`requestTimeoutMs`                                    |
    | MCP स्नेक-केस सर्वर फ़ील्ड                                                                     | कैमलकेस MCP सर्वर फ़ील्ड                                                   |
    | `tools.media.image/audio/video.models`                                                           | क्षमता-टैग युक्त `tools.media.models`                                        |
    | `tools.media.asyncCompletion`                                                                    | हटाया गया                                                                       |
    | `tools.message.allowCrossContextSend`                                                            | `tools.message.crossContext`                                                  |
    | मीडिया मॉडल के `deepgram` विकल्प                                                                   | `providerOptions.deepgram`                                                    |
    | `talk.realtime.voice`, Discord रीयलटाइम `voice`                                                 | `speakerVoice`                                                                |
    | `agents.defaults.pdfMaxBytesMb`                                                                  | `agents.defaults.pdfMaxMb`                                                    |
    | `tools.exec.timeoutSec`                                                                          | `tools.exec.timeoutSeconds`                                                   |
    | `browser.ssrfPolicy.hostnameAllowlist`                                                           | वाइल्डकार्ड-सचेत `browser.ssrfPolicy.allowedHostnames`                          |
    | सैंडबॉक्स ब्राउज़र `enableNoVnc`                                                                    | `noVncEnabled`                                                                |
    | रूट `media`                                                                                     | `attachments`                                                                |
    | चैनल/खाता `heartbeat` दृश्यता ब्लॉक                                                   | `heartbeatVisibility`                                                         |
    | `channels.slack.identity`                                                                        | `channels.slack.postAs`                                                       |
    | रूट `audit`                                                                                     | `logging.audit`                                                               |
    | `gateway.nodes.skills.enabled`                                                                   | `gateway.nodes.allowSkills`                                                   |
    | `gateway.nodes.allowCommands`/`denyCommands`                                                    | `gateway.nodes.commands.allow`/`deny`                                         |
    | जनरेशन मॉडल डिफ़ॉल्ट                                                                       | `agents.defaults.mediaModels.{image,video,music}`                              |
    | सेवानिवृत्त अंतिम-लेआउट समायोजन नियंत्रण                                                               | अंतर्निहित डिफ़ॉल्ट व्यवहार                                                     |
    | `channels.whatsapp.messagePrefix` और पुराना `messages.messagePrefix`                            | `channels.whatsapp.responsePrefix`                                            |
    | `channels.whatsapp.ackReaction`                                                                  | वैश्विक `messages.ackReaction` और जहाँ अनुवाद योग्य हो वहाँ `ackReactionScope`        |
    | `cron.failureDestination`                                                                        | `cron.failureAlert` पर गंतव्य फ़ील्ड                                     |
    | `gateway.controlUi.chatMessageMaxWidth`, केवल-प्रस्तुति `ui.prefs` कुंजियाँ                       | हटाया गया (टेक्स्ट स्केल, चैट चौड़ाई और लाइव साइडबार गतिविधि ब्राउज़र-स्थानीय हैं) |
    | `agents.list`                                                                                    | कुंजीबद्ध `agents.entries`                                                        |
    | शीर्ष-स्तरीय `defaultModel`                                                                         | `agents.defaults.model`                                                      |
    | `messages.messagePrefix`                                                                         | `channels.whatsapp.responsePrefix`                                            |
    | `session.maintenance.pruneDays`, `session.resetByType.dm`                                        | `session.maintenance.pruneAfter`, `session.resetByType.direct`               |
    | शीर्ष-स्तरीय `tui`                                                                                  | हटाया गया (TUI फ़ुटर संक्षिप्त डिफ़ॉल्ट का उपयोग करता है)                            |
    | `plugins.entries.codex.config.codexDynamicToolsProfile`                                          | हटाया गया (Codex ऐप-सर्वर हमेशा Codex-मूल वर्कस्पेस टूल को मूल रूप में रखता है) |
    | `commands.modelsWrite`                                                                           | हटाया गया (`/models add` अप्रचलित है)                                       |
    | `agents.defaults/list[].silentReplyRewrite`, `surfaces.*.silentReplyRewrite`                     | हटाया गया (सटीक `NO_REPLY` को अब दृश्यमान फ़ॉलबैक टेक्स्ट में पुनर्लिखित नहीं किया जाता)  |
    | `agents.defaults/list[].systemPromptOverride`                                                    | हटाया गया (जनरेट किए गए सिस्टम प्रॉम्प्ट का स्वामित्व OpenClaw के पास है)                        |
    | `agents.defaults/list[].embeddedPi`                                                              | `embeddedAgent`                                                              |
    | `agents.defaults/list[].sandbox.perSession`                                                      | `sandbox.scope`                                                              |
    | `agents.defaults.llm`                                                                             | हटाया गया (धीमे मॉडल/प्रदाता टाइमआउट के लिए `models.providers.<id>.timeoutSeconds` का उपयोग करें, जिसे एजेंट/रन टाइमआउट की अधिकतम सीमा से नीचे रखा गया है) |
    | शीर्ष-स्तरीय `memorySearch`, `agents.defaults.memorySearch`                                         | `memory.search`                                                             |
    | `agents.entries.*.memorySearch`                                                                     | `agents.entries.*.memory.search`                                               |
    | `memorySearch.provider: "auto"`                                                                  | `"openai"`                                                                    |
    | `memorySearch.store.path` (किसी भी स्तर पर)                                                            | हटाया गया (मेमोरी इंडेक्स प्रत्येक एजेंट डेटाबेस में रहते हैं)                       |
    | शीर्ष-स्तरीय `heartbeat`                                                                            | `agents.defaults.heartbeat` / `channels.defaults.heartbeat`                 |
    | `plugins.openai-codex` नीति आईडी                                                                | `plugins.openai`                                                             |
    | `tools.web.x_search.apiKey`                                                                      | `plugins.entries.xai.config.webSearch.apiKey`                               |
    | `session.maintenance.rotateBytes`, `session.parentForkMaxTokens`                                 | हटाया गया (बहिष्कृत)                                                        |
    | 2026.7 में Runtime और चैनल समायोजन विकल्पों को हटा दिया गया                                               | हटाया गया (अंतर्निहित प्रोडक्शन डिफ़ॉल्ट लागू होते हैं)                               |

    <Note>
      ऊपर की `plugins.entries.voice-call.config.*` पंक्तियाँ प्रत्येक कॉन्फ़िगरेशन लोड पर
      Voice Call plugin द्वारा ही सामान्यीकृत की जाती हैं, `openclaw
      doctor` द्वारा नहीं। Plugin स्टार्टअप पर `openclaw
      doctor --fix` की ओर संकेत करने वाली चेतावनी भी लॉग करता है, लेकिन doctor फ़िलहाल इन कुंजियों के लिए
      `openclaw.json` को फिर से नहीं लिखता; रनटाइम पर परिवर्तन
      plugin के अपने सामान्यीकरण से लागू होता है।
    </Note>

    बहु-अकाउंट चैनलों के लिए अकाउंट-डिफ़ॉल्ट मार्गदर्शन:

    - यदि दो या अधिक `channels.<channel>.accounts` प्रविष्टियाँ `channels.<channel>.defaultAccount` या `accounts.default` के बिना कॉन्फ़िगर की गई हैं, तो doctor चेतावनी देता है कि फ़ॉलबैक रूटिंग किसी अनपेक्षित अकाउंट को चुन सकती है।
    - यदि `channels.<channel>.defaultAccount` को किसी अज्ञात अकाउंट ID पर सेट किया गया है, तो doctor चेतावनी देता है और कॉन्फ़िगर किए गए अकाउंट ID सूचीबद्ध करता है।

  </Accordion>
  <Accordion title="2b. OpenCode प्रदाता ओवरराइड">
    यदि आपने `models.providers.opencode`, `opencode-zen`, या `opencode-go` को मैन्युअल रूप से जोड़ा है, तो यह `openclaw/plugin-sdk/llm` के अंतर्निहित OpenCode कैटलॉग को ओवरराइड करता है। इससे मॉडल गलत API पर जाने के लिए बाध्य हो सकते हैं या लागत शून्य हो सकती है। Doctor चेतावनी देता है ताकि आप ओवरराइड हटाकर प्रति-मॉडल API रूटिंग + लागत पुनर्स्थापित कर सकें।
  </Accordion>
  <Accordion title="2c. ब्राउज़र माइग्रेशन और Chrome MCP तत्परता">
    यदि आपका ब्राउज़र कॉन्फ़िगरेशन अब भी हटाए गए Chrome एक्सटेंशन पथ की ओर संकेत करता है, तो doctor उसे वर्तमान होस्ट-लोकल Chrome MCP अटैच मॉडल में सामान्यीकृत करता है (`browser.profiles.*.driver: "extension"` → `"existing-session"`; `browser.relayBindHost` हटाया गया)।

    जब आप `defaultProfile: "user"` या कॉन्फ़िगर की गई `existing-session` प्रोफ़ाइल का उपयोग करते हैं, तो Doctor होस्ट-लोकल Chrome MCP पथ का ऑडिट भी करता है:

    - डिफ़ॉल्ट ऑटो-कनेक्ट प्रोफ़ाइलों के लिए जाँचता है कि Google Chrome उसी होस्ट पर इंस्टॉल है या नहीं
    - पता लगाए गए Chrome संस्करण की जाँच करता है और उसके Chrome 144 से कम होने पर चेतावनी देता है
    - आपको ब्राउज़र निरीक्षण पृष्ठ में रिमोट डीबगिंग सक्षम करने की याद दिलाता है (उदाहरण के लिए `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging`, या `edge://inspect/#remote-debugging`)

    Doctor आपके लिए Chrome-पक्ष की सेटिंग सक्षम नहीं कर सकता। होस्ट-लोकल Chrome MCP के लिए अब भी gateway/node होस्ट पर स्थानीय रूप से चलने वाला Chromium-आधारित ब्राउज़र 144+ आवश्यक है, जिसमें रिमोट डीबगिंग सक्षम हो और ब्राउज़र में पहली अटैच सहमति प्रॉम्प्ट स्वीकृत हो।

    यहाँ तत्परता केवल स्थानीय अटैच की पूर्वापेक्षाओं को कवर करती है। Existing-session वर्तमान Chrome MCP रूट सीमाएँ बनाए रखता है; `responsebody`, PDF निर्यात, डाउनलोड इंटरसेप्शन और बैच कार्रवाइयों जैसे उन्नत रूट के लिए अब भी प्रबंधित ब्राउज़र या रॉ CDP प्रोफ़ाइल आवश्यक है। यह जाँच Docker, सैंडबॉक्स, रिमोट-ब्राउज़र या अन्य हेडलेस प्रवाहों पर लागू नहीं होती, जो रॉ CDP का उपयोग जारी रखते हैं।

  </Accordion>
  <Accordion title="2d. OAuth TLS पूर्वापेक्षाएँ">
    जब OpenAI Codex OAuth प्रोफ़ाइल कॉन्फ़िगर होती है, तो doctor यह सत्यापित करने के लिए OpenAI प्राधिकरण एंडपॉइंट की जाँच करता है कि स्थानीय Node/OpenSSL TLS स्टैक प्रमाणपत्र शृंखला को मान्य कर सकता है। यदि जाँच प्रमाणपत्र त्रुटि के साथ विफल होती है (उदाहरण के लिए `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, समय-सीमा समाप्त प्रमाणपत्र या स्व-हस्ताक्षरित प्रमाणपत्र), तो doctor प्लेटफ़ॉर्म-विशिष्ट समाधान मार्गदर्शन प्रिंट करता है। Homebrew Node वाले macOS पर समाधान आम तौर पर `brew postinstall ca-certificates` होता है। `--deep` के साथ, gateway के स्वस्थ होने पर भी जाँच चलती है।
  </Accordion>
  <Accordion title="2e. Codex OAuth प्रदाता ओवरराइड">
    यदि आपने पहले `models.providers.openai-codex` के अंतर्गत पुराने OpenAI ट्रांसपोर्ट सेटिंग्स जोड़ी थीं, तो वे अंतर्निहित Codex OAuth प्रदाता पथ को छिपा सकती हैं। Codex OAuth के साथ उन पुरानी ट्रांसपोर्ट सेटिंग्स को देखने पर Doctor चेतावनी देता है, ताकि आप पुराने ट्रांसपोर्ट ओवरराइड को हटा या फिर से लिख सकें और वर्तमान रूटिंग व्यवहार पुनर्स्थापित कर सकें। कस्टम प्रॉक्सी और केवल-हेडर ओवरराइड समर्थित रहते हैं और यह चेतावनी ट्रिगर नहीं करते, लेकिन वे स्वयं-निर्धारित अनुरोध रूट अप्रत्यक्ष Codex चयन के योग्य नहीं होते।
  </Accordion>
  <Accordion title="2f. Codex रूट मरम्मत">
    Doctor पुराने `openai-codex/*` मॉडल संदर्भों की जाँच करता है। मूल Codex हार्नेस रूटिंग विहित `openai/*` मॉडल संदर्भों का उपयोग करती है, लेकिन केवल उपसर्ग कभी भी Codex का चयन नहीं करता। रनटाइम नीति अनसेट या `auto` होने पर, केवल ऐसा सटीक आधिकारिक HTTPS Platform Responses या ChatGPT Responses रूट योग्य होता है जिसमें स्वयं-निर्धारित अनुरोध ओवरराइड न हो। [OpenAI अप्रत्यक्ष एजेंट रनटाइम](/hi/providers/openai#implicit-agent-runtime) देखें।

    `--fix` / `--repair` मोड में, doctor प्रभावित डिफ़ॉल्ट-एजेंट और प्रति-एजेंट संदर्भों को फिर से लिखता है, जिनमें प्राथमिक मॉडल, फ़ॉलबैक, इमेज/वीडियो जनरेशन मॉडल, Heartbeat/सबएजेंट/Compaction ओवरराइड, हुक, चैनल मॉडल ओवरराइड और पुरानी सहेजी गई सत्र रूट स्थिति शामिल हैं:

    - `openai-codex/gpt-*` बदलकर `openai/gpt-*` हो जाता है।
    - सुधारे गए एजेंट मॉडल संदर्भों के लिए Codex अभिप्राय प्रदाता/मॉडल-स्कोप वाली `agentRuntime.id: "codex"` प्रविष्टियों में चला जाता है।
    - पुराना संपूर्ण-एजेंट रनटाइम कॉन्फ़िगरेशन और सहेजे गए सत्र रनटाइम पिन हटा दिए जाते हैं क्योंकि रनटाइम चयन प्रदाता/मॉडल-स्कोप वाला होता है।
    - मौजूदा प्रदाता/मॉडल रनटाइम नीति तब तक सुरक्षित रखी जाती है, जब तक सुधारे गए पुराने मॉडल संदर्भ को पुराना प्रमाणीकरण पथ बनाए रखने के लिए Codex रूटिंग की आवश्यकता न हो।
    - मौजूदा मॉडल फ़ॉलबैक सूचियाँ उनकी पुरानी प्रविष्टियों को फिर से लिखकर सुरक्षित रखी जाती हैं; कॉपी की गई प्रति-मॉडल सेटिंग्स पुरानी कुंजी से विहित `openai/*` कुंजी में चली जाती हैं।
    - सहेजे गए सत्र `modelProvider`/`providerOverride`, `model`/`modelOverride`, फ़ॉलबैक सूचनाएँ और प्रमाणीकरण-प्रोफ़ाइल पिन सभी खोजे गए एजेंट सत्र स्टोर में सुधारे जाते हैं।
    - Doctor अलग से पुराने `agentRuntime.id: "codex-cli"` पिन (एक अलग पुरानी रनटाइम ID) को `agents.defaults`, `agents.entries.*`, और `models.providers.*` मॉडल प्रविष्टियों में `"codex"` में सुधारता है।
    - `/codex ...` का अर्थ है "चैट से किसी मूल Codex वार्तालाप को नियंत्रित या बाइंड करना।"
    - `/acp ...` या `runtime: "acp"` का अर्थ है "बाहरी ACP/acpx अडैप्टर का उपयोग करना।"

  </Accordion>
  <Accordion title="2g. सत्र रूट सफ़ाई">
    कॉन्फ़िगर किए गए मॉडल या रनटाइम को Codex जैसे Plugin-स्वामित्व वाले रूट से हटाने के बाद Doctor खोजे गए एजेंट सत्र स्टोर में पुरानी स्वतः-निर्मित रूट स्थिति भी स्कैन करता है।

    `openclaw doctor --fix` स्वतः-निर्मित पुरानी स्थिति साफ़ कर सकता है, जैसे `modelOverrideSource: "auto"` मॉडल पिन, रनटाइम मॉडल मेटाडेटा, पिन किए गए हार्नेस ID, CLI सत्र बाइंडिंग और स्वतः प्रमाणीकरण-प्रोफ़ाइल ओवरराइड, जब उनका स्वामी रूट अब कॉन्फ़िगर नहीं होता। स्पष्ट उपयोगकर्ता या पुराने सत्र मॉडल विकल्प मैन्युअल समीक्षा के लिए रिपोर्ट किए जाते हैं और अपरिवर्तित छोड़े जाते हैं; जब वह रूट अब अपेक्षित न हो, तो उन्हें `/model ...`, `/new` से बदलें या सत्र रीसेट करें।

  </Accordion>
  <Accordion title="3. पुरानी स्थिति के माइग्रेशन (डिस्क लेआउट)">
    Doctor पुराने ऑन-डिस्क लेआउट को वर्तमान संरचना में माइग्रेट कर सकता है:

    - सत्र स्टोर + ट्रांसक्रिप्ट: `~/.openclaw/sessions/` से `~/.openclaw/agents/<agentId>/sessions/` में
    - एजेंट डायरेक्टरी: `~/.openclaw/agent/` से `~/.openclaw/agents/<agentId>/agent/` में
    - WhatsApp प्रमाणीकरण स्थिति (Baileys): पुराने `~/.openclaw/credentials/*.json` (`oauth.json` को छोड़कर) से `~/.openclaw/credentials/whatsapp/<accountId>/...` में (डिफ़ॉल्ट अकाउंट ID: `default`)
    - हस्ताक्षरित डिवाइस पहचान: `~/.openclaw/identity/device.json` से `state/openclaw.sqlite` की `primary` `device_identities` पंक्ति में; अलग डिवाइस-प्रमाणीकरण फ़ाइल अपरिवर्तित छोड़ी जाती है

    ये माइग्रेशन सर्वोत्तम-प्रयास वाले और पुनरावृत्ति-सुरक्षित हैं; बैकअप के रूप में कोई भी पुराना फ़ोल्डर छोड़ने पर doctor चेतावनी देता है। Gateway/CLI भी स्टार्टअप पर पुराने सत्र + एजेंट डायरेक्टरी को स्वतः माइग्रेट करता है, ताकि इतिहास/प्रमाणीकरण/मॉडल मैन्युअल doctor रन के बिना प्रति-एजेंट पथ में पहुँच जाएँ। WhatsApp प्रमाणीकरण को जानबूझकर केवल `openclaw doctor` के माध्यम से माइग्रेट किया जाता है। Talk प्रदाता/प्रदाता-मैप सामान्यीकरण संरचनात्मक समानता के आधार पर तुलना करता है, इसलिए केवल कुंजी-क्रम के अंतर अब बार-बार निष्प्रभावी `doctor --fix` परिवर्तन ट्रिगर नहीं करते।

  </Accordion>
  <Accordion title="3a. पुराने plugin मैनिफ़ेस्ट माइग्रेशन">
    Doctor सभी इंस्टॉल किए गए plugin मैनिफ़ेस्ट में अप्रचलित शीर्ष-स्तरीय क्षमता कुंजियों (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders`) को स्कैन करता है। मिलने पर, वह उन्हें `contracts` ऑब्जेक्ट में ले जाने और मैनिफ़ेस्ट फ़ाइल को उसी स्थान पर फिर से लिखने की पेशकश करता है। यह माइग्रेशन पुनरावृत्ति-सुरक्षित है; यदि `contracts` में पहले से समान मान हैं, तो डेटा की नकल किए बिना पुरानी कुंजी हटा दी जाती है।
  </Accordion>
  <Accordion title="3b. पुराने Cron स्टोर माइग्रेशन">
    Doctor SQLite में विहित पंक्तियाँ आयात करने से पहले पुराने जॉब आकारों के लिए पुराने Cron जॉब स्टोर (`~/.openclaw/cron/jobs.json`) की भी जाँच करता है।

    वर्तमान Cron सफ़ाइयों में शामिल हैं:

    - `jobId` → `id`
    - `schedule.cron` → `schedule.expr`
    - शीर्ष-स्तरीय पेलोड फ़ील्ड (`message`, `model`, `thinking`, ...) → `payload`
    - शीर्ष-स्तरीय डिलीवरी फ़ील्ड (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
    - पेलोड `provider` डिलीवरी उपनाम → स्पष्ट `delivery.channel`
    - पुराने `notify: true` Webhook फ़ॉलबैक जॉब → मान्य होने पर सेवानिवृत्त रॉ `cron.webhook` मान से स्पष्ट Webhook डिलीवरी; घोषणा जॉब अपनी चैट डिलीवरी बनाए रखते हैं और उन्हें `delivery.completionDestination` मिलता है। इसके बाद Doctor पुरानी कॉन्फ़िगरेशन कुंजी हटा देता है। उपयोग योग्य पुराना Webhook न होने पर, निष्प्रभावी शीर्ष-स्तरीय `notify` चिह्न बिना-लक्ष्य वाले जॉब के लिए हटा दिया जाता है (घोषणा सहित मौजूदा डिलीवरी सुरक्षित रखी जाती है), क्योंकि रनटाइम डिलीवरी इसे कभी नहीं पढ़ती।

    Gateway लोड समय पर विकृत Cron पंक्तियों को भी स्वच्छ करता है, ताकि मान्य जॉब चलते रहें। हटाने से पहले रॉ विकृत पंक्तियाँ सक्रिय स्टोर के पास `jobs-quarantine.json` में कॉपी की जाती हैं और फिर `jobs.json` से हटाई जाती हैं; doctor क्वारंटीन की गई पंक्तियों की रिपोर्ट करता है, ताकि आप उनकी मैन्युअल समीक्षा या मरम्मत कर सकें।

    Gateway स्टार्टअप रनटाइम प्रोजेक्शन को सामान्यीकृत करता है और शीर्ष-स्तरीय `notify` चिह्न की उपेक्षा करता है, लेकिन सहेजी गई Cron स्थिति को doctor की मरम्मत के लिए छोड़ देता है। Doctor बिना किसी माइग्रेशन लक्ष्य वाले जॉब के निष्प्रभावी चिह्न हटा देता है (`delivery.mode` कोई नहीं/अनुपस्थित, अनुपयोगी पुराना Webhook लक्ष्य, या मौजूदा घोषणा/चैट डिलीवरी), जबकि मौजूदा डिलीवरी अपरिवर्तित रहती है, इसलिए बार-बार `doctor --fix` चलाने पर उसी जॉब के बारे में दोबारा चेतावनी नहीं मिलती।

    Linux पर, doctor तब भी चेतावनी देता है जब उपयोगकर्ता का crontab अब भी पुराने `~/.openclaw/bin/ensure-whatsapp.sh` को चलाता है। वह होस्ट-लोकल स्क्रिप्ट वर्तमान OpenClaw द्वारा अनुरक्षित नहीं है और जब Cron systemd उपयोगकर्ता बस तक नहीं पहुँच पाता, तो `~/.openclaw/logs/whatsapp-health.log` में झूठे `Gateway inactive` संदेश लिख सकती है। पुरानी crontab प्रविष्टि को `crontab -e` से हटाएँ; वर्तमान स्वास्थ्य जाँचों के लिए `openclaw channels status --probe`, `openclaw doctor`, और `openclaw gateway status` का उपयोग करें।

  </Accordion>
  <Accordion title="3c. सत्र लॉक की सफ़ाई">
    Doctor असामान्य रूप से समाप्त हुए सत्रों द्वारा छोड़ी गई पुरानी राइट-लॉक फ़ाइलों के लिए प्रत्येक एजेंट सत्र डायरेक्टरी को स्कैन करता है। मिली प्रत्येक लॉक फ़ाइल के लिए यह रिपोर्ट करता है: पथ, PID, क्या PID अब भी सक्रिय है, लॉक की आयु, और क्या उसे पुराना माना गया है (मृत PID, विकृत स्वामी मेटाडेटा, 30 मिनट से अधिक पुराना, या ऐसा सक्रिय PID जिसका किसी गैर-OpenClaw प्रक्रिया से संबंधित होना प्रमाणित हो)। `--fix` / `--repair` मोड में यह मृत, अनाथ, पुनः उपयोग किए गए, विकृत-पुराने, या गैर-OpenClaw स्वामियों वाले लॉक स्वचालित रूप से हटा देता है। किसी सक्रिय OpenClaw प्रक्रिया के स्वामित्व वाले पुराने लॉक की रिपोर्ट की जाती है, लेकिन उन्हें यथावत रखा जाता है ताकि Doctor किसी सक्रिय ट्रांसक्रिप्ट राइटर को बाधित न करे।
  </Accordion>
  <Accordion title="3d. सत्र ट्रांसक्रिप्ट शाखा की मरम्मत">
    Doctor एजेंट सत्र की JSONL फ़ाइलों को 2026.4.24 के प्रॉम्प्ट ट्रांसक्रिप्ट रीराइट बग से बनी डुप्लिकेट शाखा संरचना के लिए स्कैन करता है: OpenClaw के आंतरिक रनटाइम संदर्भ वाला एक परित्यक्त उपयोगकर्ता टर्न और उसी दृश्यमान उपयोगकर्ता प्रॉम्प्ट वाला एक सक्रिय सिबलिंग। `--fix` / `--repair` मोड में, Doctor प्रत्येक प्रभावित फ़ाइल का मूल फ़ाइल के पास बैकअप बनाता है और ट्रांसक्रिप्ट को सक्रिय शाखा पर पुनः लिखता है, ताकि Gateway इतिहास और मेमोरी रीडर अब डुप्लिकेट टर्न न देखें।
  </Accordion>
  <Accordion title="4. स्टेट अखंडता जाँच (सत्र स्थायित्व, रूटिंग और सुरक्षा)">
    स्टेट डायरेक्टरी संचालन का मस्तिष्क-तना है। यदि यह गायब हो जाती है, तो अन्यत्र बैकअप न होने पर आपके सत्र, क्रेडेंशियल, लॉग और कॉन्फ़िगरेशन खो जाते हैं।

    Doctor जाँचता है:

    - **स्टेट डायरेक्टरी अनुपस्थित**: विनाशकारी स्टेट हानि की चेतावनी देता है, डायरेक्टरी फिर से बनाने के लिए संकेत देता है और याद दिलाता है कि यह अनुपस्थित डेटा पुनर्प्राप्त नहीं कर सकता।
    - **स्टेट डायरेक्टरी अनुमतियाँ**: लिखने की क्षमता सत्यापित करता है; अनुमतियाँ सुधारने की पेशकश करता है (और स्वामी/समूह बेमेल मिलने पर `chown` संकेत देता है)।
    - **macOS क्लाउड-सिंक की गई स्टेट डायरेक्टरी**: जब स्टेट iCloud Drive (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) या `~/Library/CloudStorage/...` के अंतर्गत रिज़ॉल्व होता है, तो चेतावनी देता है, क्योंकि सिंक-समर्थित पथ धीमे I/O और लॉक/सिंक रेस उत्पन्न कर सकते हैं।
    - **Linux SD या eMMC स्टेट डायरेक्टरी**: जब स्टेट किसी `mmcblk*` माउंट स्रोत पर रिज़ॉल्व होता है, तो चेतावनी देता है, क्योंकि SD/eMMC-समर्थित रैंडम I/O धीमा हो सकता है और सत्र तथा क्रेडेंशियल लेखन के दौरान अधिक तेज़ी से घिस सकता है।
    - **Linux अस्थायी स्टेट डायरेक्टरी**: जब स्टेट `tmpfs` या `ramfs` पर रिज़ॉल्व होता है, तो चेतावनी देता है, क्योंकि रीबूट होने पर सत्र, क्रेडेंशियल, कॉन्फ़िगरेशन और SQLite स्टेट (WAL/जर्नल साइडकार सहित) गायब हो जाते हैं। Docker `overlay` माउंट को जानबूझकर चिह्नित नहीं किया जाता, क्योंकि कंटेनर के बने रहने तक उनकी लिखने योग्य परतें होस्ट रीबूट के दौरान बनी रहती हैं।
    - **सत्र डायरेक्टरियाँ अनुपस्थित**: इतिहास बनाए रखने और `ENOENT` क्रैश से बचने के लिए `sessions/` और सत्र स्टोर डायरेक्टरी आवश्यक हैं।
    - **ट्रांसक्रिप्ट बेमेल**: हाल की सत्र प्रविष्टियों की ट्रांसक्रिप्ट फ़ाइलें अनुपस्थित होने पर चेतावनी देता है।
    - **मुख्य सत्र "1-पंक्ति JSONL"**: मुख्य ट्रांसक्रिप्ट में केवल एक पंक्ति होने पर चिह्नित करता है (इतिहास संचित नहीं हो रहा है)।
    - **एकाधिक स्टेट डायरेक्टरियाँ**: होम डायरेक्टरियों में एकाधिक `~/.openclaw` फ़ोल्डर मौजूद होने पर, या `OPENCLAW_STATE_DIR` के कहीं और इंगित करने पर चेतावनी देता है (इतिहास इंस्टॉलेशन के बीच विभाजित हो सकता है)।
    - **रिमोट मोड अनुस्मारक**: यदि `gateway.mode=remote`, तो Doctor आपको इसे रिमोट होस्ट पर चलाने की याद दिलाता है (स्टेट वहीं रहता है)।
    - **कॉन्फ़िगरेशन फ़ाइल अनुमतियाँ**: यदि `~/.openclaw/openclaw.json` समूह/सभी के लिए पठनीय है, तो चेतावनी देता है और इसे `600` तक सीमित करने की पेशकश करता है।

  </Accordion>
  <Accordion title="5. मॉडल प्रमाणीकरण की स्थिति (OAuth समाप्ति)">
    Doctor प्रमाणीकरण स्टोर में OAuth प्रोफ़ाइलों का निरीक्षण करता है, टोकन के शीघ्र समाप्त होने या समाप्त हो जाने पर चेतावनी देता है और सुरक्षित होने पर उन्हें रीफ़्रेश कर सकता है। यदि Anthropic OAuth/टोकन प्रोफ़ाइल पुरानी है, तो यह Anthropic API कुंजी या Anthropic सेटअप-टोकन पथ सुझाता है। रीफ़्रेश संकेत केवल इंटरैक्टिव रूप से (TTY) चलाते समय दिखाई देते हैं; `--non-interactive` रीफ़्रेश प्रयासों को छोड़ देता है।

    जब कोई OAuth रीफ़्रेश स्थायी रूप से विफल होता है (उदाहरण के लिए `refresh_token_reused`, `invalid_grant`, या कोई प्रदाता आपको फिर से साइन इन करने को कहता है), तो Doctor रिपोर्ट करता है कि पुनः प्रमाणीकरण आवश्यक है और चलाने के लिए सटीक `openclaw models auth login --provider ...` कमांड प्रिंट करता है।

    Doctor उन प्रमाणीकरण प्रोफ़ाइलों की भी रिपोर्ट करता है जो छोटे कूलडाउन (दर सीमाएँ/टाइमआउट/प्रमाणीकरण विफलताएँ) या लंबे समय के निष्क्रियकरण (बिलिंग/क्रेडिट विफलताएँ) के कारण अस्थायी रूप से अनुपयोगी हैं।

    वे लीगेसी Codex OAuth प्रोफ़ाइलें, जिनके टोकन macOS Keychain में रहते हैं (फ़ाइल-आधारित साइडकार लेआउट से पहले की पुरानी ऑनबोर्डिंग), केवल Doctor द्वारा सुधारी जाती हैं। Keychain-समर्थित लीगेसी टोकन को उसी स्थान पर `auth-profiles.json` में माइग्रेट करने के लिए किसी इंटरैक्टिव टर्मिनल से एक बार `openclaw doctor --fix` चलाएँ; इसके बाद एम्बेडेड टर्न (Telegram, Cron, सब-एजेंट डिस्पैच) उन्हें कैनोनिकल OpenAI OAuth प्रोफ़ाइलों के रूप में रिज़ॉल्व करते हैं।

  </Accordion>
  <Accordion title="6. Hooks मॉडल सत्यापन">
    यदि `hooks.gmail.model` सेट है, तो Doctor कैटलॉग और अनुमत सूची के विरुद्ध मॉडल संदर्भ को सत्यापित करता है और उसके रिज़ॉल्व न होने या अस्वीकृत होने पर चेतावनी देता है।
  </Accordion>
  <Accordion title="7. सैंडबॉक्स इमेज की मरम्मत">
    सैंडबॉक्सिंग सक्षम होने पर, Doctor Docker इमेज की जाँच करता है और वर्तमान इमेज अनुपस्थित होने पर उसे बनाने या लीगेसी नामों पर स्विच करने की पेशकश करता है।
  </Accordion>
  <Accordion title="7b. Plugin इंस्टॉलेशन की सफ़ाई">
    Doctor `openclaw doctor --fix` / `openclaw doctor --repair` मोड में OpenClaw द्वारा जनरेट की गई लीगेसी Plugin निर्भरता स्टेजिंग स्टेट को हटाता है: पुराने जनरेट किए गए निर्भरता रूट, पुरानी इंस्टॉल-स्टेज डायरेक्टरियाँ, पहले के बंडल किए गए Plugin निर्भरता मरम्मत कोड से बचा पैकेज-स्थानीय मलबा और बंडल किए गए `@openclaw/*` Plugins की अनाथ या पुनर्प्राप्त प्रबंधित npm प्रतियाँ, जो वर्तमान बंडल किए गए मैनिफ़ेस्ट को ओवरराइड कर सकती हैं। Doctor होस्ट `openclaw` पैकेज को उन प्रबंधित npm Plugins में फिर से लिंक भी करता है जो `peerDependencies.openclaw` घोषित करते हैं, ताकि `openclaw/plugin-sdk/*` जैसे पैकेज-स्थानीय रनटाइम इम्पोर्ट अपडेट या npm मरम्मत के बाद भी रिज़ॉल्व होते रहें।

    कॉन्फ़िगरेशन में संदर्भित, लेकिन स्थानीय Plugin रजिस्ट्री द्वारा न मिल सकने वाले डाउनलोड योग्य Plugins को भी Doctor फिर से इंस्टॉल कर सकता है (महत्त्वपूर्ण `plugins.entries`, कॉन्फ़िगर की गई चैनल/प्रदाता/खोज सेटिंग्स, कॉन्फ़िगर किए गए एजेंट रनटाइम)। पैकेज अपडेट के दौरान, कोर पैकेज बदले जाते समय Doctor Plugin पैकेजों को फिर से इंस्टॉल करने से बचता है; यदि किसी कॉन्फ़िगर किए गए Plugin को अब भी पुनर्प्राप्ति की आवश्यकता है, तो अपडेट के बाद `openclaw doctor --fix` फिर से चलाएँ। नीचे दिए गए कंटेनर इमेज स्टार्टअप अपवाद के बाहर, Gateway स्टार्टअप और कॉन्फ़िगरेशन रीलोड पैकेज मरम्मत नहीं चलाते; Plugin इंस्टॉलेशन स्पष्ट Doctor/इंस्टॉल/अपडेट कार्य बने रहते हैं।

    कंटेनरीकृत Gateway स्टार्टअप में एक सीमित अपग्रेड अपवाद है: जब `openclaw gateway run` किसी नए OpenClaw संस्करण पर शुरू होता है, तो यह तैयार होने से पहले सुरक्षित स्टेट माइग्रेशन और मौजूदा पोस्ट-कोर Plugin अभिसरण चलाता है, फिर प्रति-संस्करण चेकपॉइंट दर्ज करता है। यह स्टार्टअप चरण पुराने बंडल किए गए Plugin रिकॉर्ड साफ़ कर सकता है, स्थानीय Plugin लिंक सुधार सकता है, अभिसरण पथ द्वारा आवश्यक होने पर कॉन्फ़िगर किए गए Plugin पैकेज फिर से इंस्टॉल कर सकता है और सक्रिय Plugin पेलोड की जाँच कर सकता है। यदि स्टार्टअप सुरक्षित रूप से मरम्मत नहीं कर सकता, तो कंटेनर को सामान्य रूप से पुनः शुरू करने से पहले उसी माउंट किए गए स्टेट/कॉन्फ़िगरेशन के विरुद्ध उसी इमेज को `openclaw doctor --fix` के साथ एक बार चलाएँ।

  </Accordion>
  <Accordion title="8. Gateway सेवा माइग्रेशन और सफ़ाई संकेत">
    Doctor लीगेसी Gateway सेवाओं (launchd/systemd/schtasks) का पता लगाता है और उन्हें हटाकर वर्तमान Gateway पोर्ट का उपयोग करते हुए OpenClaw सेवा इंस्टॉल करने की पेशकश करता है। यह अतिरिक्त Gateway-जैसी सेवाओं को भी स्कैन कर सकता है और सफ़ाई संकेत प्रिंट कर सकता है। प्रोफ़ाइल-नाम वाली OpenClaw Gateway सेवाओं को प्रथम-श्रेणी माना जाता है और उन्हें "अतिरिक्त" के रूप में चिह्नित नहीं किया जाता।

    Linux पर, यदि उपयोगकर्ता-स्तरीय Gateway सेवा अनुपस्थित है लेकिन सिस्टम-स्तरीय OpenClaw Gateway सेवा मौजूद है, तो Doctor दूसरी उपयोगकर्ता-स्तरीय सेवा स्वचालित रूप से इंस्टॉल नहीं करता। `openclaw gateway status --deep` या `openclaw doctor --deep` से निरीक्षण करें, फिर डुप्लिकेट हटाएँ या जब कोई सिस्टम पर्यवेक्षक Gateway जीवनचक्र का स्वामी हो तब `OPENCLAW_SERVICE_REPAIR_POLICY=external` सेट करें।

  </Accordion>
  <Accordion title="8b. स्टार्टअप Matrix माइग्रेशन">
    जब किसी Matrix चैनल खाते में लंबित या कार्रवाई योग्य लीगेसी स्टेट माइग्रेशन होता है, तो Doctor (`--fix` / `--repair` मोड में) माइग्रेशन-पूर्व स्नैपशॉट बनाता है और फिर सर्वोत्तम-प्रयास वाले माइग्रेशन चरण चलाता है: लीगेसी Matrix स्टेट माइग्रेशन और लीगेसी एन्क्रिप्टेड-स्टेट तैयारी। दोनों चरण गैर-घातक हैं; त्रुटियाँ लॉग की जाती हैं और स्टार्टअप जारी रहता है। केवल-पठन मोड (`openclaw doctor`, `--fix` के बिना) में यह जाँच पूरी तरह छोड़ दी जाती है।
  </Accordion>
  <Accordion title="8c. डिवाइस पेयरिंग और प्रमाणीकरण विचलन">
    Doctor सामान्य स्वास्थ्य जाँच के भाग के रूप में डिवाइस-पेयरिंग स्टेट का निरीक्षण करते हुए निम्नलिखित रिपोर्ट करता है:

    - पहली बार पेयरिंग के लंबित अनुरोध
    - पहले से पेयर किए गए डिवाइसों के लिए लंबित भूमिका या स्कोप अपग्रेड
    - सार्वजनिक-कुंजी बेमेल मरम्मत, जहाँ डिवाइस ID अब भी मेल खाती है लेकिन डिवाइस पहचान अब अनुमोदित रिकॉर्ड से मेल नहीं खाती
    - अनुमोदित भूमिका के लिए सक्रिय टोकन से रहित पेयर किए गए रिकॉर्ड
    - ऐसे पेयर किए गए टोकन जिनके स्कोप अनुमोदित पेयरिंग आधार-रेखा से बाहर विचलित हो गए हैं
    - वर्तमान मशीन की स्थानीय रूप से कैश की गई डिवाइस-टोकन प्रविष्टियाँ, जो Gateway-पक्ष के टोकन रोटेशन से पहले की हैं या जिनमें पुराना स्कोप मेटाडेटा है

    Doctor पेयरिंग अनुरोधों को स्वतः अनुमोदित या डिवाइस टोकन को स्वतः रोटेट नहीं करता। यह अगले सटीक चरण प्रिंट करता है:

    - `openclaw devices list` से लंबित अनुरोधों का निरीक्षण करें
    - `openclaw devices approve <requestId>` से सटीक अनुरोध अनुमोदित करें
    - `openclaw devices rotate --device <deviceId> --role <role>` से नया टोकन रोटेट करें
    - `openclaw devices remove <deviceId>` से पुराना रिकॉर्ड हटाकर फिर से अनुमोदित करें

    यह पहली बार की पेयरिंग को लंबित भूमिका/स्कोप अपग्रेड और पुराने टोकन/डिवाइस-पहचान विचलन से अलग करता है, जिससे सामान्य "पहले से पेयर किया हुआ, फिर भी पेयरिंग आवश्यक मिल रहा है" कमी दूर होती है।

  </Accordion>
  <Accordion title="9. सुरक्षा चेतावनियाँ">
    Doctor केवल चेतावनी मिलने पर सुरक्षा नोट देता है, जैसे अनुमत सूची के बिना DMs के लिए खुला प्रदाता या खतरनाक ढंग से कॉन्फ़िगर की गई नीति। संपूर्ण सुरक्षा इन्वेंटरी के लिए `openclaw security audit` का उपयोग करें।
  </Accordion>
  <Accordion title="10. systemd linger (Linux)">
    systemd उपयोगकर्ता सेवा के रूप में चलने पर, Doctor सुनिश्चित करता है कि linger सक्षम हो, ताकि लॉगआउट के बाद भी Gateway सक्रिय रहे।
  </Accordion>
  <Accordion title="11. वर्कस्पेस स्थिति (Skills, Plugins और TaskFlows)">
    Doctor स्वस्थ-स्थिति इन्वेंटरी के बजाय डिफ़ॉल्ट एजेंट की समस्याएँ और कार्रवाइयाँ प्रिंट करता है:

    - **Skills**: अनुमत लेकिन अनुपयोगी Skills के नाम सूचीबद्ध करता है; आवश्यकता विवरण और पूरी गणना के लिए `openclaw skills check` का उपयोग करें।
    - **Plugins**: केवल त्रुटियुक्त Plugin ID की रिपोर्ट करता है; लोड किए गए, इम्पोर्ट किए गए, अक्षम और बंडल-Plugin इन्वेंटरी के लिए `openclaw plugins list` का उपयोग करें।
    - **Plugin संगतता चेतावनियाँ**: वर्तमान रनटाइम के साथ संगतता समस्याओं वाले Plugins को चिह्नित करता है।
    - **Plugin निदान**: Plugin रजिस्ट्री द्वारा लोड के समय दी गई चेतावनियाँ या त्रुटियाँ दिखाता है।
    - **TaskFlow पुनर्प्राप्ति**: संदिग्ध प्रबंधित TaskFlows दिखाता है जिन्हें मैन्युअल निरीक्षण या रद्द करने की आवश्यकता है।
    - **Claude CLI**: केवल बाइनरी, प्रमाणीकरण, प्रोफ़ाइल, वर्कस्पेस या प्रोजेक्ट-डायरेक्टरी समस्याओं की रिपोर्ट करता है; स्वस्थ प्रोब विवरण छोड़ दिए जाते हैं।

  </Accordion>
  <Accordion title="11b. बूटस्ट्रैप फ़ाइल आकार">
    Doctor जाँचता है कि वर्कस्पेस बूटस्ट्रैप फ़ाइलें (उदाहरण के लिए `AGENTS.md`, `CLAUDE.md`, या अन्य इंजेक्ट की गई संदर्भ फ़ाइलें) कॉन्फ़िगर की गई वर्ण सीमा के पास या उससे अधिक हैं या नहीं। यह प्रत्येक फ़ाइल के लिए मूल बनाम इंजेक्ट किए गए वर्णों की संख्या, ट्रंकेशन प्रतिशत, ट्रंकेशन का कारण (`max/file` या `max/total`) और कुल सीमा के अनुपात के रूप में इंजेक्ट किए गए कुल वर्णों की रिपोर्ट करता है। फ़ाइलें ट्रंकेट होने या सीमा के पास होने पर, Doctor `agents.defaults.bootstrapMaxChars` और `agents.defaults.bootstrapTotalMaxChars` को ट्यून करने के सुझाव प्रिंट करता है।
  </Accordion>
  <Accordion title="11c. शेल कम्प्लीशन">
    Doctor जाँचता है कि वर्तमान शेल (zsh, bash, fish या PowerShell) के लिए टैब कम्प्लीशन इंस्टॉल है या नहीं:

    - यदि शेल प्रोफ़ाइल धीमे डायनेमिक कम्प्लीशन पैटर्न (`source <(openclaw completion ...)`) का उपयोग करती है, तो doctor उसे अधिक तेज़ कैश्ड फ़ाइल वेरिएंट में अपग्रेड करता है।
    - यदि प्रोफ़ाइल में कम्प्लीशन कॉन्फ़िगर है लेकिन कैश फ़ाइल मौजूद नहीं है, तो doctor कैश को स्वचालित रूप से फिर से जनरेट करता है।
    - यदि कोई कम्प्लीशन बिल्कुल भी कॉन्फ़िगर नहीं है, तो doctor उसे इंस्टॉल करने का संकेत देता है (केवल इंटरैक्टिव मोड; `--non-interactive` के साथ छोड़ दिया जाता है)।

    कैश को मैन्युअल रूप से फिर से जनरेट करने के लिए `openclaw completion --write-state` चलाएँ।

  </Accordion>
  <Accordion title="11d. पुराने चैनल Plugin की सफ़ाई">
    जब `openclaw doctor --fix` किसी अनुपलब्ध चैनल Plugin को हटाता है, तो वह उस Plugin को संदर्भित करने वाले लटके हुए चैनल-स्कोप्ड कॉन्फ़िगरेशन को भी हटा देता है: `channels.<id>` प्रविष्टियाँ, चैनल का नाम रखने वाले Heartbeat लक्ष्य और `agents.*.models["<channel>/*"]` ओवरराइड। इससे ऐसे Gateway बूट लूप रुकते हैं जिनमें चैनल रनटाइम हट चुका होता है, लेकिन कॉन्फ़िगरेशन अब भी Gateway को उससे बाइंड होने के लिए कहता है।
  </Accordion>
  <Accordion title="12. Gateway प्रमाणीकरण जाँच (स्थानीय टोकन)">
    Doctor स्थानीय Gateway टोकन प्रमाणीकरण की तत्परता जाँचता है।

    - यदि टोकन मोड को टोकन चाहिए और कोई टोकन स्रोत मौजूद नहीं है, तो doctor एक टोकन जनरेट करने का विकल्प देता है।
    - यदि `gateway.auth.token` SecretRef द्वारा प्रबंधित है लेकिन उपलब्ध नहीं है, तो doctor चेतावनी देता है और उसे प्लेनटेक्स्ट से ओवरराइट नहीं करता।
    - `openclaw doctor --generate-gateway-token` केवल तभी जनरेशन को बाध्य करता है, जब कोई टोकन SecretRef कॉन्फ़िगर न हो।

  </Accordion>
  <Accordion title="12b. केवल-पढ़ने योग्य SecretRef-जागरूक मरम्मत">
    कुछ मरम्मत प्रवाहों को रनटाइम के तुरंत विफल होने वाले व्यवहार को कमज़ोर किए बिना कॉन्फ़िगर किए गए क्रेडेंशियल की जाँच करनी होती है।

    - `openclaw doctor --fix` लक्षित कॉन्फ़िगरेशन मरम्मत के लिए स्टेटस-फ़ैमिली कमांड जैसा ही केवल-पढ़ने योग्य SecretRef सारांश मॉडल उपयोग करता है।
    - उदाहरण: Telegram `allowFrom` / `groupAllowFrom` `@username` मरम्मत उपलब्ध होने पर कॉन्फ़िगर किए गए बॉट क्रेडेंशियल का उपयोग करने का प्रयास करती है।
    - यदि Telegram बॉट टोकन SecretRef के माध्यम से कॉन्फ़िगर किया गया है, लेकिन वर्तमान कमांड पथ में उपलब्ध नहीं है, तो doctor बताता है कि क्रेडेंशियल कॉन्फ़िगर है लेकिन उपलब्ध नहीं है और क्रैश करने या टोकन को अनुपलब्ध बताने के बजाय स्वचालित समाधान छोड़ देता है।

  </Accordion>
  <Accordion title="13. Gateway स्वास्थ्य जाँच + पुनः आरंभ">
    Doctor स्वास्थ्य जाँच चलाता है और Gateway के अस्वस्थ दिखाई देने पर उसे पुनः आरंभ करने का विकल्प देता है।
  </Accordion>
  <Accordion title="13b. मेमोरी खोज की तत्परता">
    Doctor जाँचता है कि कॉन्फ़िगर किया गया मेमोरी खोज एम्बेडिंग प्रदाता डिफ़ॉल्ट एजेंट के लिए तैयार है या नहीं। व्यवहार कॉन्फ़िगर किए गए बैकएंड और प्रदाता पर निर्भर करता है:

    - **QMD बैकएंड**: जाँचता है कि `qmd` बाइनरी उपलब्ध और प्रारंभ करने योग्य है या नहीं। यदि नहीं, तो `npm install -g @tobilu/qmd` (या Bun समकक्ष) और मैन्युअल बाइनरी पथ विकल्प सहित सुधार मार्गदर्शन प्रिंट करता है।
    - **स्पष्ट स्थानीय प्रदाता**: स्थानीय मॉडल फ़ाइल या किसी मान्यता-प्राप्त रिमोट/डाउनलोड करने योग्य मॉडल URL की जाँच करता है। अनुपलब्ध होने पर रिमोट प्रदाता पर स्विच करने का सुझाव देता है।
    - **स्पष्ट रिमोट प्रदाता** (`openai`, `voyage`, आदि): सत्यापित करता है कि परिवेश या प्रमाणीकरण स्टोर में API कुंजी मौजूद है। अनुपलब्ध होने पर कार्रवाई योग्य सुधार संकेत प्रिंट करता है।
    - **लेगेसी स्वचालित प्रदाता**: `memorySearch.provider: "auto"` को OpenAI मानता है, OpenAI की तत्परता जाँचता है और `doctor --fix` उसे `provider: "openai"` के रूप में फिर से लिखता है।

    जब कैश किया हुआ Gateway जाँच परिणाम उपलब्ध होता है (जाँच के समय Gateway स्वस्थ था), तो doctor उसके परिणाम का CLI में दिखाई देने वाले कॉन्फ़िगरेशन से मिलान करता है और किसी भी विसंगति का उल्लेख करता है। Doctor डिफ़ॉल्ट पथ पर नई एम्बेडिंग पिंग शुरू नहीं करता; लाइव प्रदाता जाँच के लिए गहन मेमोरी स्थिति कमांड का उपयोग करें।

    रनटाइम पर एम्बेडिंग की तत्परता सत्यापित करने के लिए `openclaw memory status --deep` का उपयोग करें।

  </Accordion>
  <Accordion title="14. चैनल स्थिति चेतावनियाँ">
    यदि Gateway स्वस्थ है, तो doctor चैनल स्थिति जाँच चलाता है और सुझाए गए सुधारों के साथ चेतावनियाँ रिपोर्ट करता है।
  </Accordion>
  <Accordion title="15. सुपरवाइज़र कॉन्फ़िगरेशन ऑडिट + मरम्मत">
    Doctor अनुपलब्ध या पुराने डिफ़ॉल्ट के लिए इंस्टॉल किए गए सुपरवाइज़र कॉन्फ़िगरेशन (launchd/systemd/schtasks) की जाँच करता है (उदाहरण के लिए systemd network-online निर्भरताएँ और पुनः आरंभ विलंब)। विसंगति मिलने पर वह अपडेट की अनुशंसा करता है और सर्विस फ़ाइल/टास्क को वर्तमान डिफ़ॉल्ट के अनुसार फिर से लिख सकता है।

    नोट्स:

    - `openclaw doctor` सुपरवाइज़र कॉन्फ़िगरेशन को फिर से लिखने से पहले संकेत देता है।
    - `openclaw doctor --yes` डिफ़ॉल्ट मरम्मत संकेतों को स्वीकार करता है।
    - `openclaw doctor --fix` बिना संकेत के अनुशंसित सुधार लागू करता है (`--repair` एक उपनाम है)।
    - `openclaw doctor --fix --force` कस्टम सुपरवाइज़र कॉन्फ़िगरेशन को ओवरराइट करता है।
    - `OPENCLAW_SERVICE_REPAIR_POLICY=external` Gateway सर्विस जीवनचक्र के लिए doctor को केवल-पढ़ने योग्य रखता है। यह फिर भी सर्विस स्वास्थ्य की रिपोर्ट करता है और गैर-सर्विस मरम्मत चलाता है, लेकिन सर्विस इंस्टॉल/प्रारंभ/पुनः आरंभ/बूटस्ट्रैप, सुपरवाइज़र कॉन्फ़िगरेशन पुनर्लेखन और लेगेसी सर्विस सफ़ाई छोड़ देता है, क्योंकि उस जीवनचक्र का स्वामित्व किसी बाहरी सुपरवाइज़र के पास होता है।
    - Linux पर, मिलान करने वाली systemd Gateway यूनिट सक्रिय होने के दौरान doctor कमांड/एंट्रीपॉइंट मेटाडेटा को फिर से नहीं लिखता। डुप्लिकेट-सर्विस स्कैन के दौरान वह निष्क्रिय, गैर-लेगेसी अतिरिक्त Gateway-जैसी यूनिट को भी अनदेखा करता है, ताकि सहयोगी सर्विस फ़ाइलें अनावश्यक सफ़ाई सूचनाएँ उत्पन्न न करें।
    - यदि टोकन प्रमाणीकरण को टोकन की आवश्यकता है और `gateway.auth.token` SecretRef द्वारा प्रबंधित है, तो doctor सर्विस इंस्टॉल/मरम्मत SecretRef को सत्यापित करती है, लेकिन समाधान किए गए प्लेनटेक्स्ट टोकन मानों को सुपरवाइज़र सर्विस परिवेश मेटाडेटा में स्थायी नहीं करती।
    - Doctor पुराने LaunchAgent, systemd या Windows Scheduled Task इंस्टॉलेशन द्वारा इनलाइन एम्बेड किए गए प्रबंधित `.env`/SecretRef-समर्थित सर्विस परिवेश मानों का पता लगाता है और सर्विस मेटाडेटा को फिर से लिखता है, ताकि वे मान सुपरवाइज़र परिभाषा के बजाय रनटाइम स्रोत से लोड हों।
    - `gateway.port` बदलने के बाद भी सर्विस कमांड द्वारा पुराने `--port` को पिन किए जाने का doctor पता लगाता है और सर्विस मेटाडेटा को वर्तमान पोर्ट के अनुसार फिर से लिखता है।
    - यदि टोकन प्रमाणीकरण को टोकन की आवश्यकता है और कॉन्फ़िगर किया गया टोकन SecretRef अनसुलझा है, तो doctor कार्रवाई योग्य मार्गदर्शन के साथ इंस्टॉल/मरम्मत पथ को अवरुद्ध करता है।
    - यदि `gateway.auth.token` और `gateway.auth.password` दोनों कॉन्फ़िगर हैं और `gateway.auth.mode` सेट नहीं है, तो doctor मोड को स्पष्ट रूप से सेट किए जाने तक इंस्टॉल/मरम्मत को अवरुद्ध करता है।
    - Linux उपयोगकर्ता-systemd यूनिट के लिए, सर्विस प्रमाणीकरण मेटाडेटा की तुलना करते समय doctor टोकन ड्रिफ़्ट जाँच में `Environment=` और `EnvironmentFile=` दोनों स्रोतों को शामिल करता है।
    - यदि कॉन्फ़िगरेशन को अंतिम बार किसी नए संस्करण द्वारा लिखा गया था, तो Doctor सर्विस मरम्मत किसी पुराने OpenClaw बाइनरी से Gateway सर्विस को फिर से लिखने, रोकने या पुनः आरंभ करने से इनकार करती है। [Gateway समस्या निवारण](/hi/gateway/troubleshooting#split-brain-installs-and-newer-config-guard) देखें।
    - आप `openclaw gateway install --force` के माध्यम से हमेशा पूर्ण पुनर्लेखन बाध्य कर सकते हैं।

  </Accordion>
  <Accordion title="16. Gateway रनटाइम + पोर्ट निदान">
    Doctor सर्विस रनटाइम (PID, अंतिम निकास स्थिति) की जाँच करता है और सर्विस इंस्टॉल होने पर भी वास्तव में न चल रही हो तो चेतावनी देता है। वह Gateway पोर्ट (डिफ़ॉल्ट `18789`) पर पोर्ट टकराव की भी जाँच करता है और संभावित कारणों (Gateway पहले से चल रहा है, SSH टनल) की रिपोर्ट करता है।
  </Accordion>
  <Accordion title="17. Gateway रनटाइम की सर्वोत्तम प्रथाएँ">
    Gateway सर्विस के Bun या संस्करण-प्रबंधित Node पथ (`nvm`, `fnm`, `volta`, `asdf`, आदि) पर चलने पर doctor चेतावनी देता है। Bun, OpenClaw के `node:sqlite` स्टेट स्टोर को नहीं खोल सकता, इसलिए मरम्मत लेगेसी Bun सर्विसों को Node पर माइग्रेट करती है। संस्करण-प्रबंधक पथ अपग्रेड के बाद टूट सकते हैं, क्योंकि सर्विस आपका शेल आरंभीकरण लोड नहीं करती। उपलब्ध होने पर doctor किसी सिस्टम Node इंस्टॉलेशन (Homebrew/apt/choco) पर माइग्रेट करने का विकल्प देता है।

    नए इंस्टॉल या मरम्मत किए गए macOS LaunchAgents इंटरैक्टिव शेल PATH की प्रतिलिपि बनाने के बजाय एक मानक सिस्टम PATH (`/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin`) का उपयोग करते हैं, ताकि Homebrew द्वारा प्रबंधित सिस्टम बाइनरी उपलब्ध रहें, जबकि Volta, asdf, fnm, pnpm और अन्य संस्करण-प्रबंधक डायरेक्टरी यह न बदलें कि कौन-से Node चाइल्ड प्रोसेस रिज़ॉल्व होते हैं। Linux सर्विसें अब भी स्पष्ट परिवेश रूट (`NVM_DIR`, `FNM_DIR`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `BUN_INSTALL`, `PNPM_HOME`) और स्थिर उपयोगकर्ता-बिन डायरेक्टरी बनाए रखती हैं, लेकिन अनुमानित संस्करण-प्रबंधक फ़ॉलबैक डायरेक्टरी केवल तभी सर्विस PATH में लिखी जाती हैं, जब वे डायरेक्टरी डिस्क पर मौजूद हों।

  </Accordion>
  <Accordion title="18. कॉन्फ़िगरेशन लेखन + विज़ार्ड मेटाडेटा">
    Doctor सभी कॉन्फ़िगरेशन बदलाव स्थायी करता है और doctor रन को रिकॉर्ड करने के लिए विज़ार्ड मेटाडेटा अंकित करता है।
  </Accordion>
  <Accordion title="19. कार्यक्षेत्र सुझाव (बैकअप + मेमोरी सिस्टम)">
    Doctor अनुपलब्ध होने पर कार्यक्षेत्र मेमोरी सिस्टम का सुझाव देता है और यदि कार्यक्षेत्र पहले से git के अंतर्गत नहीं है, तो बैकअप सुझाव प्रिंट करता है।

    कार्यक्षेत्र संरचना और git बैकअप (निजी GitHub या GitLab अनुशंसित) की संपूर्ण मार्गदर्शिका के लिए [/अवधारणाएँ/एजेंट-कार्यस्थान](/hi/concepts/agent-workspace) देखें।

  </Accordion>
</AccordionGroup>

## संबंधित

- [Gateway संचालन पुस्तिका](/hi/gateway)
- [Gateway समस्या निवारण](/hi/gateway/troubleshooting)
