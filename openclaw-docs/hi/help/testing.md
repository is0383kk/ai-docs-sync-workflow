---
read_when:
    - स्थानीय रूप से या CI में परीक्षण चलाना
    - मॉडल/प्रोवाइडर बग के लिए रिग्रेशन जोड़ना
    - Gateway + एजेंट के व्यवहार की डीबगिंग
summary: 'परीक्षण किट: यूनिट/e2e/लाइव सुइट, Docker रनर, और प्रत्येक परीक्षण में क्या शामिल है'
title: परीक्षण
x-i18n:
    generated_at: "2026-07-27T19:49:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 20e0aa22bf16561334f83342abffabb387ed0b41b901773939123ecfbc0ae330
    source_path: help/testing.md
    workflow: 16
---

OpenClaw में तीन Vitest सुइट (यूनिट/इंटीग्रेशन, e2e, लाइव) और Docker
रनर हैं। यह पृष्ठ बताता है कि प्रत्येक सुइट में क्या शामिल है, किसी दिए गए
वर्कफ़्लो के लिए कौन-सी कमांड चलानी है, लाइव टेस्ट क्रेडेंशियल कैसे खोजते हैं,
और वास्तविक दुनिया की प्रोवाइडर/मॉडल बग के लिए रिग्रेशन कैसे जोड़ने हैं।

<Note>
**QA स्टैक (qa-lab, qa-channel, लाइव ट्रांसपोर्ट लेन)** का दस्तावेज़ अलग से उपलब्ध है:

- [QA अवलोकन](/hi/concepts/qa-e2e-automation) - आर्किटेक्चर, कमांड सरफ़ेस, परिदृश्य लेखन और Matrix प्रोफ़ाइल।
- [परिपक्वता स्कोरकार्ड](/hi/maturity/scorecard) - रिलीज़ QA साक्ष्य स्थिरता और LTS निर्णयों का समर्थन कैसे करते हैं।
- [QA चैनल](/hi/channels/qa-channel) - रिपॉज़िटरी-समर्थित परिदृश्यों द्वारा उपयोग किया जाने वाला सिंथेटिक ट्रांसपोर्ट Plugin।

यह पृष्ठ नियमित टेस्ट सुइट और Docker/Parallels रनर के बारे में है। नीचे [QA-विशिष्ट रनर](#qa-specific-runners) में ठोस `qa` आह्वान सूचीबद्ध हैं और ऊपर दिए गए संदर्भों की ओर संकेत किया गया है।
</Note>

## त्वरित शुरुआत

अधिकांश दिनों में:

- पूर्ण गेट (पुश से पहले अपेक्षित): `pnpm build && pnpm check && pnpm check:test-types && pnpm test`
- पर्याप्त संसाधनों वाली मशीन पर तेज़ स्थानीय पूर्ण-सुइट रन: `pnpm test:max`
- प्रत्यक्ष Vitest वॉच लूप: `pnpm test:watch`
- प्रत्यक्ष फ़ाइल लक्ष्यीकरण Plugin/चैनल पथों को भी रूट करता है: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- किसी एक विफलता पर पुनरावृत्ति करते समय पहले लक्षित रन को प्राथमिकता दें।
- Docker-समर्थित QA साइट: `pnpm qa:lab:up`
- Linux VM-समर्थित QA लेन: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

जब आप टेस्ट में बदलाव करें या अतिरिक्त भरोसा चाहें:

- सूचनात्मक V8 कवरेज रिपोर्ट: `pnpm test:coverage`
- E2E सुइट: `pnpm test:e2e`

## टेस्ट की अस्थायी डायरेक्टरियाँ

टेस्ट-स्वामित्व वाली अस्थायी डायरेक्टरियों के लिए `test/helpers/temp-dir.ts` में साझा
हेल्पर का उपयोग करें, ताकि स्वामित्व स्पष्ट रहे और क्लीनअप टेस्ट जीवनचक्र में बना रहे:

```ts
import { afterEach } from "vitest";
import { useAutoCleanupTempDirTracker } from "../helpers/temp-dir.js";

const tempDirs = useAutoCleanupTempDirTracker(afterEach);

it("अस्थायी वर्कस्पेस का उपयोग करता है", () => {
  const workspace = tempDirs.make("openclaw-example-");
  // वर्कस्पेस का उपयोग करें
});
```

`useAutoCleanupTempDirTracker(afterEach)` जानबूझकर कोई मैन्युअल
क्लीनअप विधि उपलब्ध नहीं कराता—प्रत्येक टेस्ट के बाद क्लीनअप का स्वामित्व Vitest के पास है। पुराने निम्न-स्तरीय
हेल्पर (`makeTempDir`, `cleanupTempDirs`, `createTempDirTracker`) उन टेस्ट के लिए अब भी मौजूद हैं
जिनका माइग्रेशन नहीं हुआ है; उनका नया उपयोग न करें और नए प्रत्यक्ष
`fs.mkdtemp*` कॉल से भी बचें, जब तक कोई टेस्ट स्पष्ट रूप से मूल अस्थायी-डायरेक्टरी
व्यवहार सत्यापित न कर रहा हो। जब प्रत्यक्ष अस्थायी डायरेक्टरी वास्तव में आवश्यक हो, तो कारण सहित
ऑडिट-योग्य अनुमति टिप्पणी जोड़ें:

```ts
// openclaw-temp-dir: allow मूल fs क्लीनअप व्यवहार सत्यापित करता है
const workspace = fs.mkdtempSync(prefix);
```

`node scripts/report-test-temp-creations.mjs` जोड़ी गई डिफ़ पंक्तियों में नई प्रत्यक्ष अस्थायी-डायरेक्टरी
रचना और साझा हेल्पर के नए मैन्युअल उपयोग की रिपोर्ट करता है,
बिना मौजूदा क्लीनअप शैलियों को अवरुद्ध किए। यह `scripts/changed-lanes.mjs` जैसी ही टेस्ट-पथ वर्गीकरण
प्रक्रिया अपनाता है और स्वयं साझा हेल्पर कार्यान्वयन को छोड़ देता है।
`check:changed` बदले गए टेस्ट पथों के लिए इस रिपोर्ट को
केवल-चेतावनी CI संकेत (GitHub चेतावनी एनोटेशन, विफलताएँ नहीं) के रूप में चलाता है।

## लाइव और Docker/Parallels वर्कफ़्लो

वास्तविक प्रोवाइडर/मॉडल डीबग करते समय (वास्तविक क्रेडेंशियल आवश्यक):

- लाइव सुइट (मॉडल + Gateway टूल/इमेज प्रोब): `pnpm test:live`
- एक लाइव फ़ाइल को शांत रूप से लक्षित करें: `pnpm test:live -- src/agents/models.profiles.live.test.ts`
- रनटाइम प्रदर्शन रिपोर्ट: वास्तविक `openai/gpt-5.6-luna` एजेंट टर्न के लिए
  `live_openai_candidate=true` के साथ या Kova CPU/हीप/ट्रेस आर्टिफ़ैक्ट के लिए
  `deep_profile=true` के साथ `OpenClaw Performance` डिस्पैच करें। दैनिक निर्धारित रन
  एक अलग आर्टिफ़ैक्ट-उपभोग करने वाली प्रकाशक जॉब से मॉक-प्रोवाइडर, डीप-प्रोफ़ाइल और GPT-5.6 Luna लेन रिपोर्ट
  `openclaw/clawgrit-reports` पर प्रकाशित करते हैं;
  प्रकाशक प्रमाणीकरण का अनुपस्थित या अमान्य होना निर्धारित और
  `profile=release` रन को विफल करता है। मैन्युअल गैर-रिलीज़ डिस्पैच GitHub आर्टिफ़ैक्ट
  बनाए रखते हैं और रिपोर्ट प्रकाशन को परामर्शात्मक मानते हैं। मॉक-प्रोवाइडर रिपोर्ट में
  स्रोत-स्तरीय Gateway बूट, मेमोरी, Plugin-दबाव, बार-बार चलने वाला
  नकली-मॉडल हेलो-लूप और CLI स्टार्टअप संख्याएँ भी शामिल हैं।
- Docker लाइव मॉडल स्वीप: `pnpm test:docker:live-models`
  - प्रत्येक चयनित मॉडल एक टेक्स्ट टर्न और एक छोटी फ़ाइल-पठन-शैली की प्रोब चलाता है।
    जिन मॉडलों का मेटाडेटा `image` इनपुट दर्शाता है, वे एक छोटा इमेज टर्न भी चलाते हैं।
    प्रोवाइडर विफलताओं को अलग करते समय अतिरिक्त प्रोब को `OPENCLAW_LIVE_MODEL_FILE_PROBE=0` या
    `OPENCLAW_LIVE_MODEL_IMAGE_PROBE=0` से अक्षम करें।
  - CI कवरेज: दैनिक `OpenClaw Scheduled Live And E2E Checks` और मैन्युअल
    `OpenClaw Release Checks`, दोनों पुन: प्रयोज्य लाइव/E2E वर्कफ़्लो को
    `include_live_suites: true` के साथ कॉल करते हैं, जिसमें प्रोवाइडर के अनुसार
    शार्ड की गई Docker लाइव मॉडल मैट्रिक्स जॉब शामिल हैं।
  - केंद्रित CI पुनः-रन के लिए, `OpenClaw Live And E2E Checks (Reusable)` को
    `include_live_suites: true` और `live_models_only: true` के साथ डिस्पैच करें।
  - नए उच्च-संकेत प्रोवाइडर सीक्रेट को `scripts/ci-hydrate-live-auth.sh`
    के साथ `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml` और उसके
    निर्धारित/रिलीज़ कॉलर में जोड़ें।
- नेटिव Codex बाउंड-चैट स्मोक: `pnpm test:docker:live-codex-bind`
  - Codex ऐप-सर्वर पथ के विरुद्ध Docker लाइव लेन चलाता है,
    `/codex bind` के साथ सिंथेटिक Slack DM बाँधता है, `/codex fast` और
    `/codex permissions` का अभ्यास करता है, फिर सत्यापित करता है कि एक साधारण उत्तर और इमेज अटैचमेंट
    ACP के बजाय नेटिव Plugin बाइंडिंग के माध्यम से रूट होते हैं।
- Codex ऐप-सर्वर हार्नेस स्मोक: `pnpm test:docker:live-codex-harness`
  - Plugin-स्वामित्व वाले Codex ऐप-सर्वर
    हार्नेस के माध्यम से Gateway एजेंट टर्न चलाता है, `/codex status` और `/codex models` सत्यापित करता है और डिफ़ॉल्ट रूप से
    इमेज, Cron MCP, सब-एजेंट और Guardian प्रोब का अभ्यास करता है। अन्य विफलताओं को
    अलग करते समय सब-एजेंट प्रोब को `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=0` से
    अक्षम करें। केंद्रित सब-एजेंट जाँच के लिए अन्य
    प्रोब अक्षम करें:
    `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 pnpm test:docker:live-codex-harness`.
    यह सब-एजेंट प्रोब के बाद बाहर निकल जाता है, जब तक
    `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_ONLY=0` सेट न हो।
- Codex ऑन-डिमांड इंस्टॉल स्मोक: `pnpm test:docker:codex-on-demand`
  - पैकेज किए गए OpenClaw टारबॉल को Docker में इंस्टॉल करता है, OpenAI API-कुंजी
    ऑनबोर्डिंग चलाता है और सत्यापित करता है कि Codex Plugin तथा `@openai/codex` डिपेंडेंसी
    माँग पर प्रबंधित npm प्रोजेक्ट रूट में डाउनलोड हुईं।
- Codex npm-Plugin लाइव पैकेज स्मोक: `pnpm test:docker:live-codex-npm-plugin`
  - उम्मीदवार OpenClaw पैकेज और सटीक Codex Plugin को Docker में इंस्टॉल करता है,
    फिर CLI प्रीफ़्लाइट और समान-सत्र टर्न के लिए वास्तविक OpenAI कुंजी का उपयोग करता है।
  - इसके शून्य-पुनःप्रयास मध्यम-थिंकिंग फ़ॉलो-थ्रू टर्न को प्रगति भेजनी होगी,
    यादृच्छिक वर्कस्पेस पठन और सटीक आर्टिफ़ैक्ट लेखन के दौरान काम जारी रखना होगा,
    फिर पूर्णता भेजनी होगी। केवल-प्रगति वाला अंतिम टर्न लेन को विफल करता है।
- लाइव Plugin टूल डिपेंडेंसी स्मोक: `pnpm test:docker:live-plugin-tool`
  - वास्तविक `slugify` डिपेंडेंसी वाले फ़िक्स्चर Plugin को पैक करता है, उसे
    `npm-pack:` के माध्यम से इंस्टॉल करता है, प्रबंधित npm
    प्रोजेक्ट रूट के अंतर्गत डिपेंडेंसी सत्यापित करता है, फिर लाइव OpenAI मॉडल से Plugin टूल कॉल करने और
    छिपा हुआ स्लग लौटाने को कहता है।
- OpenClaw रेस्क्यू कमांड स्मोक: `pnpm test:live:system-agent-rescue-channel`
  - संदेश-चैनल रेस्क्यू कमांड
    सरफ़ेस के लिए वैकल्पिक अतिरिक्त-सुरक्षा जाँच। `/openclaw status` का अभ्यास करता है, स्थायी मॉडल
    बदलाव कतारबद्ध करता है, `/openclaw yes` उत्तर देता है और ऑडिट/कॉन्फ़िग लेखन
    पथ सत्यापित करता है।
- OpenClaw प्रथम-रन Docker स्मोक: `pnpm test:docker:system-agent-first-run`
  - खाली OpenClaw स्टेट डायरेक्टरी से शुरू होता है और पहले प्रमाणित करता है कि पैकेज किया गया
    `openclaw setup` CLI अनुमान के बिना बंद रहते हुए विफल होता है। फिर यह
    पैकेज किए गए सक्रियण मॉड्यूल के माध्यम से नकली Claude का परीक्षण और सक्रियण करता है।
    केवल उसके बाद ही एक अस्पष्ट पैकेज की गई CLI अनुरोध प्लानर तक पहुँचती है और
    टाइप किए गए सेटअप में हल होती है, जिसके बाद वन-शॉट मॉडल, एजेंट, Discord कॉन्फ़िग
    और SecretRef ऑपरेशन होते हैं। यह कॉन्फ़िग और ऑडिट प्रविष्टियाँ सत्यापित करता है। यह
    सहायक गेट/ऑपरेशन साक्ष्य है, इंटरैक्टिव ऑनबोर्डिंग या
    OpenClaw एजेंट/टूल/अनुमोदन प्रमाण नहीं। यही लेन QA Lab में
    `pnpm openclaw qa suite --scenario system-agent-ring-zero-setup` द्वारा उपलब्ध है।
- Moonshot/Kimi लागत स्मोक: `MOONSHOT_API_KEY` सेट होने पर,
  `openclaw models list --provider moonshot --json` चलाएँ, फिर `moonshot/kimi-k2.6` के विरुद्ध एक पृथक
  `openclaw agent --local --session-id live-kimi-cost --message 'Reply exactly: KIMI_LIVE_OK' --thinking off --json`
  चलाएँ। सत्यापित करें कि JSON Moonshot/K2.6 की रिपोर्ट करता है और
  सहायक ट्रांसक्रिप्ट सामान्यीकृत `usage.cost` संग्रहीत करता है।

<Tip>
जब केवल एक विफल मामला आवश्यक हो, तो नीचे वर्णित अनुमतिसूची एन्वायरनमेंट वेरिएबल के माध्यम से लाइव टेस्ट सीमित करने को प्राथमिकता दें।
</Tip>

## QA-विशिष्ट रनर

जब QA-lab जैसी वास्तविकता आवश्यक हो, तब ये कमांड मुख्य टेस्ट सुइट के साथ उपलब्ध हैं।

CI समर्पित वर्कफ़्लो में QA Lab चलाता है। एजेंटिक समानता
`QA-Lab - All Lanes` और रिलीज़ सत्यापन के अंतर्गत नेस्टेड है, यह कोई स्वतंत्र PR वर्कफ़्लो नहीं है।
व्यापक सत्यापन के लिए `Full Release Validation` का
`rerun_group=qa-parity` या रिलीज़-जाँच QA समूह के साथ उपयोग करना चाहिए। स्थिर/डिफ़ॉल्ट रिलीज़
जाँच संपूर्ण लाइव/Docker सोक को `run_release_soak=true` के पीछे रखती हैं;
`full` प्रोफ़ाइल सोक को अनिवार्य रूप से चालू करती है। `QA-Lab - All Lanes` रात में `main` पर और
मैन्युअल डिस्पैच से मॉक समानता लेन, लाइव Matrix लेन,
Convex-प्रबंधित लाइव Telegram लेन और Convex-प्रबंधित लाइव Discord लेन को
समानांतर जॉब के रूप में चलाता है। निर्धारित QA और रिलीज़ जाँच साझा लाइव अडैप्टर के माध्यम से
Matrix रिलीज़ प्रोफ़ाइल चलाती हैं। Matrix CLI और मैन्युअल वर्कफ़्लो इनपुट का
डिफ़ॉल्ट `all` बना रहता है; मैन्युअल `all` डिस्पैच ट्रांसपोर्ट, मीडिया और
E2EE प्रोफ़ाइल में फैलते हैं, जबकि केंद्रित डिस्पैच `fast`, `release` या
`transport` चुन सकते हैं। `OpenClaw Release Checks` रिलीज़ अनुमोदन से पहले समानता के साथ पुन: प्रयोज्य Matrix
लाइव-अडैप्टर प्रोफ़ाइल और Telegram लेन चलाता है। रिलीज़
ट्रांसपोर्ट जाँच `mock-openai/gpt-5.6-luna` का उपयोग करती हैं, ताकि वे नियतात्मक रहें और
सामान्य प्रोवाइडर-Plugin स्टार्टअप से बचें। ये लाइव ट्रांसपोर्ट Gateway
मेमोरी खोज अक्षम करते हैं; मेमोरी व्यवहार QA समानता सुइट द्वारा कवर रहता है।

पूर्ण रिलीज़ लाइव मीडिया शार्ड
`ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04` का उपयोग करते हैं, जिसमें पहले से
`ffmpeg` और `ffprobe` हैं। Docker लाइव मॉडल/बैकएंड शार्ड साझा
`ghcr.io/openclaw/openclaw-live-test:<sha>` इमेज का उपयोग करते हैं, जिसे प्रत्येक चयनित
कमिट के लिए एक बार बनाया जाता है, फिर प्रत्येक शार्ड के भीतर दोबारा बनाने के बजाय
`OPENCLAW_SKIP_DOCKER_BUILD=1` से पुल किया जाता है।

- `pnpm openclaw qa suite`
  - होस्ट पर सीधे रिपॉज़िटरी-समर्थित QA परिदृश्य चलाता है।
  - चयनित परिदृश्य सेट के लिए शीर्ष-स्तरीय `qa-evidence.json`, `qa-suite-summary.json`, और
    `qa-suite-report.md` आर्टिफ़ैक्ट लिखता है, जिनमें
    मिश्रित प्रवाह, Vitest, और Playwright परिदृश्य चयन शामिल हैं।
  - `pnpm openclaw qa run --qa-profile <profile>` द्वारा डिस्पैच किए जाने पर,
    चयनित टैक्सोनॉमी प्रोफ़ाइल स्कोरकार्ड को उसी `qa-evidence.json` में एम्बेड करता है।
    `smoke-ci` संक्षिप्त साक्ष्य लिखता है (`evidenceMode: "slim"`, प्रति-प्रविष्टि
    `execution` नहीं)। `release` क्यूरेट किया गया रिलीज़-तत्परता खंड कवर करता है; `all`
    प्रत्येक सक्रिय परिपक्वता श्रेणी चुनता है और पूर्ण स्कोरकार्ड आर्टिफ़ैक्ट की आवश्यकता होने पर स्पष्ट QA Profile
    Evidence वर्कफ़्लो डिस्पैच को लक्षित करता है।
  - डिफ़ॉल्ट रूप से पृथक gateway वर्कर के साथ कई चयनित परिदृश्य समानांतर में
    चलाता है। `qa-channel` की डिफ़ॉल्ट समवर्तीता 4 है (चयनित
    परिदृश्यों की संख्या तक सीमित)। वर्कर की संख्या समायोजित करने के लिए `--concurrency <count>`,
    या पुराने क्रमिक लेन के लिए `--concurrency 1` का उपयोग करें।
  - कोई भी परिदृश्य विफल होने पर गैर-शून्य स्थिति के साथ बाहर निकलता है। विफल निकास कोड के बिना
    आर्टिफ़ैक्ट के लिए `--allow-failures` का उपयोग करें।
  - प्रोवाइडर मोड `live-frontier`, `mock-openai`, और `aimock` का समर्थन करता है।
    `aimock` परिदृश्य-सजग
    `mock-openai` लेन को बदले बिना प्रायोगिक फ़िक्स्चर और प्रोटोकॉल-मॉक कवरेज के लिए स्थानीय AIMock-समर्थित प्रोवाइडर सर्वर शुरू करता है।
- `pnpm openclaw qa coverage --match <query>`
  - परिदृश्य आईडी, शीर्षक, सरफ़ेस, कवरेज आईडी, दस्तावेज़ संदर्भ, कोड
    संदर्भ, plugins, और प्रोवाइडर आवश्यकताएँ खोजता है, फिर मेल खाने वाले सुइट
    लक्ष्य प्रिंट करता है।
  - QA Lab रन से पहले इसका उपयोग तब करें, जब आपको प्रभावित व्यवहार या फ़ाइल
    पथ ज्ञात हो लेकिन सबसे छोटा परिदृश्य ज्ञात न हो। यह केवल सलाहकारी है—फिर भी बदले जा रहे व्यवहार के आधार पर मॉक,
    लाइव, Multipass, Matrix, या ट्रांसपोर्ट प्रमाण चुनें।
- `pnpm test:plugins:kitchen-sink-live`
  - QA Lab के माध्यम से लाइव OpenAI Kitchen Sink plugin परीक्षण-शृंखला चलाता है।
    बाहरी Kitchen Sink पैकेज इंस्टॉल करता है, plugin SDK
    सरफ़ेस इन्वेंट्री सत्यापित करता है, `/healthz` और `/readyz` की जाँच करता है, gateway
    CPU/RSS साक्ष्य रिकॉर्ड करता है, एक लाइव OpenAI टर्न चलाता है, और प्रतिकूल
    डायग्नोस्टिक्स जाँचता है। `OPENAI_API_KEY` जैसे लाइव OpenAI प्रमाणीकरण की आवश्यकता है। हाइड्रेटेड
    Testbox सत्रों में, `openclaw-testbox-env` सहायक मौजूद होने पर यह स्वचालित रूप से Testbox लाइव-प्रमाणीकरण
    प्रोफ़ाइल को सोर्स करता है।
- `pnpm test:gateway:cpu-scenarios`
  - gateway स्टार्टअप बेंच और एक छोटा मॉक QA Lab परिदृश्य पैक
    (`channel-chat-baseline`, `memory-failure-fallback`,
    `gateway-restart-inflight-run`) चलाता है और `.artifacts/gateway-cpu-scenarios/` के अंतर्गत संयुक्त CPU अवलोकन
    सारांश लिखता है।
  - डिफ़ॉल्ट रूप से केवल लगातार उच्च CPU अवलोकनों को चिह्नित करता है (`--cpu-core-warn`,
    डिफ़ॉल्ट `0.9`; `--hot-wall-warn-ms`, डिफ़ॉल्ट `30000`), इसलिए स्टार्टअप के छोटे
    उछाल मेट्रिक्स के रूप में रिकॉर्ड होते हैं और कई मिनट तक gateway को पूर्ण क्षमता पर चलाने वाले
    रिग्रेशन जैसे नहीं दिखते।
  - निर्मित `dist` आर्टिफ़ैक्ट के विरुद्ध चलता है; यदि चेकआउट में पहले से नवीन रनटाइम आउटपुट
    नहीं है, तो पहले बिल्ड चलाएँ।
- `pnpm openclaw qa suite --runner multipass`
  - उसी QA सुइट को एक डिस्पोज़ेबल Multipass Linux VM के भीतर चलाता है और
    `qa suite` वाले परिदृश्य-चयन तथा प्रोवाइडर/मॉडल फ़्लैग समान रखता है।
  - लाइव रन अतिथि के लिए व्यावहारिक QA प्रमाणीकरण इनपुट फ़ॉरवर्ड करते हैं:
    env-आधारित प्रोवाइडर कुंजियाँ, QA लाइव प्रोवाइडर कॉन्फ़िगरेशन पथ, और
    मौजूद होने पर `CODEX_HOME`।
  - आउटपुट डायरेक्टरी रिपॉज़िटरी रूट के अंतर्गत ही रहनी चाहिए, ताकि अतिथि माउंट किए गए वर्कस्पेस के माध्यम से
    वापस लिख सके।
  - सामान्य QA रिपोर्ट + सारांश के साथ Multipass लॉग
    `.artifacts/qa-e2e/...` के अंतर्गत लिखता है।
- `pnpm qa:lab:up`
  - ऑपरेटर-शैली के QA कार्य के लिए Docker-समर्थित QA साइट शुरू करता है।
- `pnpm test:docker:npm-onboard-channel-agent`
  - वर्तमान चेकआउट से npm टारबॉल बनाता है, उसे Docker में वैश्विक रूप से इंस्टॉल करता है,
    गैर-इंटरैक्टिव OpenAI API-कुंजी ऑनबोर्डिंग चलाता है, डिफ़ॉल्ट रूप से
    Telegram कॉन्फ़िगर करता है, सत्यापित करता है कि पैकेज किया गया plugin रनटाइम स्टार्टअप निर्भरता सुधार के बिना लोड होता है,
    doctor चलाता है, और मॉक किए गए OpenAI एंडपॉइंट के विरुद्ध एक स्थानीय एजेंट टर्न चलाता है।
  - Discord के साथ वही पैकेज-इंस्टॉल
    लेन चलाने के लिए `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` का उपयोग करें।
- `pnpm test:docker:session-runtime-context`
  - एम्बेडेड रनटाइम संदर्भ
    ट्रांसक्रिप्ट के लिए नियतात्मक बिल्ट-ऐप Docker स्मोक चलाता है। सत्यापित करता है कि छिपा हुआ OpenClaw रनटाइम संदर्भ दृश्यमान उपयोगकर्ता
    टर्न में लीक होने के बजाय गैर-प्रदर्शन कस्टम संदेश के रूप में बना रहता है, फिर प्रभावित टूटा हुआ सत्र JSONL सीड करता है और सत्यापित करता है कि
    `openclaw doctor --fix` उसे बैकअप सहित सक्रिय ब्रांच में पुनर्लिखता है।
- `pnpm test:docker:npm-telegram-live`
  - Docker में OpenClaw पैकेज उम्मीदवार इंस्टॉल करता है, इंस्टॉल किए गए पैकेज की
    ऑनबोर्डिंग चलाता है, इंस्टॉल किए गए CLI के माध्यम से Telegram कॉन्फ़िगर करता है, फिर उसी इंस्टॉल किए गए पैकेज को SUT
    Gateway के रूप में उपयोग करके लाइव Telegram QA लेन का पुनः उपयोग करता है।
  - रैपर चेकआउट से केवल `qa-lab` हार्नेस स्रोत माउंट करता है;
    इंस्टॉल किया गया पैकेज `dist`, `openclaw/plugin-sdk`, और बंडल किए गए
    plugin रनटाइम का स्वामी होता है, इसलिए लेन वर्तमान चेकआउट plugins को परीक्षणाधीन
    पैकेज में नहीं मिलाता।
  - डिफ़ॉल्ट `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@beta` है; रजिस्ट्री से इंस्टॉल करने के बजाय
    हल किए गए स्थानीय टारबॉल का परीक्षण करने के लिए
    `OPENCLAW_NPM_TELEGRAM_PACKAGE_TGZ=/path/to/openclaw-current.tgz` या
    `OPENCLAW_CURRENT_PACKAGE_TGZ` सेट करें।
  - डिफ़ॉल्ट रूप से `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES=20` के साथ `qa-evidence.json` में
    बार-बार RTT समय उत्सर्जित करता है। रन समायोजित करने के लिए
    `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`,
    `OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS`, या
    `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES` ओवरराइड करें।
    `OPENCLAW_NPM_TELEGRAM_RTT_CHECKS` नमूना लेने के लिए Telegram QA परिदृश्य चुनता है;
    समर्थित RTT लक्ष्य `channel-canary` है।
  - `pnpm openclaw qa telegram` के समान Telegram env क्रेडेंशियल या Convex क्रेडेंशियल स्रोत का
    उपयोग करता है। CI/रिलीज़ स्वचालन के लिए
    `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex` के साथ
    `OPENCLAW_QA_CONVEX_SITE_URL` और एक भूमिका सीक्रेट सेट करें। यदि
    `OPENCLAW_QA_CONVEX_SITE_URL` और Convex भूमिका सीक्रेट CI में मौजूद हैं,
    तो Docker रैपर स्वचालित रूप से Convex चुनता है।
  - रैपर Docker बिल्ड/इंस्टॉल कार्य से पहले होस्ट पर Telegram या Convex क्रेडेंशियल env
    सत्यापित करता है। केवल प्री-क्रेडेंशियल सेटअप को जानबूझकर
    डीबग करते समय `OPENCLAW_NPM_TELEGRAM_SKIP_CREDENTIAL_PREFLIGHT=1` सेट करें।
  - `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci|maintainer` केवल इस लेन के लिए
    साझा `OPENCLAW_QA_CREDENTIAL_ROLE` को ओवरराइड करता है। Convex
    क्रेडेंशियल चुने जाने और कोई भूमिका सेट न होने पर, रैपर CI में `ci`
    और CI के बाहर `maintainer` का उपयोग करता है।
  - GitHub Actions इस लेन को मैन्युअल मेंटेनर वर्कफ़्लो
    `NPM Telegram Beta E2E` के रूप में उपलब्ध कराता है। यह मर्ज पर नहीं चलता। वर्कफ़्लो
    `qa-live-shared` परिवेश और Convex CI क्रेडेंशियल लीज़ का उपयोग करता है।
- GitHub Actions एक उम्मीदवार पैकेज के विरुद्ध साइड-रन उत्पाद प्रमाण के लिए
  `Package Acceptance` भी उपलब्ध कराता है। यह Git संदर्भ, प्रकाशित npm स्पेक,
  HTTPS टारबॉल URL तथा SHA-256, विश्वसनीय-URL नीति, या किसी अन्य रन के टारबॉल आर्टिफ़ैक्ट
  (`source=ref|npm|url|trusted-url|artifact`) को स्वीकार करता है, सामान्यीकृत
  `openclaw-current.tgz` को `package-under-test` के रूप में अपलोड करता है, फिर
  मौजूदा Docker E2E शेड्यूलर को `smoke`, `package`, `product`, `full`,
  या `custom` लेन प्रोफ़ाइल के साथ चलाता है। उसी
  `package-under-test` आर्टिफ़ैक्ट के विरुद्ध Telegram QA वर्कफ़्लो चलाने के लिए `telegram_mode=mock-openai` या
  `live-frontier` सेट करें।
  - नवीनतम बीटा उत्पाद प्रमाण:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai
```

- सटीक टारबॉल URL प्रमाण के लिए डाइजेस्ट आवश्यक है और यह सार्वजनिक URL सुरक्षा नीति का उपयोग करता है:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=url \
  -f package_url=https://registry.npmjs.org/openclaw/-/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

- एंटरप्राइज़/निजी टारबॉल मिरर स्पष्ट विश्वसनीय-स्रोत नीति का उपयोग करते हैं:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

`source=trusted-url` विश्वसनीय वर्कफ़्लो संदर्भ से `.github/package-trusted-sources.json` पढ़ता है और URL क्रेडेंशियल या वर्कफ़्लो-इनपुट निजी-नेटवर्क बायपास स्वीकार नहीं करता। यदि नामित नीति bearer प्रमाणीकरण घोषित करती है, तो निश्चित `OPENCLAW_TRUSTED_PACKAGE_TOKEN` सीक्रेट कॉन्फ़िगर करें।

- आर्टिफ़ैक्ट प्रमाण किसी अन्य Actions रन से टारबॉल आर्टिफ़ैक्ट डाउनलोड करता है:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=<artifact-name> \
  -f suite_profile=smoke
```

- `pnpm test:docker:plugins`
  - वर्तमान OpenClaw बिल्ड को Docker में पैक और इंस्टॉल करता है, OpenAI कॉन्फ़िगर करके
    Gateway शुरू करता है, फिर कॉन्फ़िगरेशन संपादनों के माध्यम से बंडल किए गए चैनल/plugins सक्षम करता है।
  - सत्यापित करता है कि सेटअप खोज में अकॉन्फ़िगर किए गए डाउनलोड योग्य plugins
    अनुपस्थित रहते हैं, पहला कॉन्फ़िगर किया गया doctor सुधार प्रत्येक अनुपस्थित
    डाउनलोड योग्य plugin को स्पष्ट रूप से इंस्टॉल करता है, और दूसरा पुनरारंभ
    छिपा हुआ निर्भरता सुधार नहीं चलाता।
  - एक ज्ञात पुराना npm बेसलाइन भी इंस्टॉल करता है, `openclaw update --tag <candidate>` चलाने से पहले
    Telegram सक्षम करता है, और सत्यापित करता है कि
    उम्मीदवार का अपडेट-पश्चात doctor किसी हार्नेस-पक्षीय postinstall सुधार के बिना पुराने plugin निर्भरता अवशेष
    साफ़ करता है।
- `pnpm test:parallels:npm-update`
  - Parallels अतिथियों में मूल पैकेज-इंस्टॉल अपडेट स्मोक चलाता है।
    प्रत्येक चयनित प्लेटफ़ॉर्म पहले अनुरोधित बेसलाइन पैकेज इंस्टॉल करता है,
    फिर उसी अतिथि में इंस्टॉल किया गया `openclaw update` कमांड चलाता है और
    इंस्टॉल किया गया संस्करण, अपडेट स्थिति, gateway तत्परता, और
    एक स्थानीय एजेंट टर्न सत्यापित करता है।
  - एक अतिथि पर पुनरावृत्ति करते समय `--platform macos`, `--platform windows`, या `--platform linux`
    का उपयोग करें। सारांश आर्टिफ़ैक्ट
    पथ और प्रति-लेन स्थिति के लिए `--json` का उपयोग करें।
  - OpenAI लेन डिफ़ॉल्ट रूप से लाइव एजेंट-टर्न प्रमाण के लिए `openai/gpt-5.6-luna` का उपयोग करता है।
    किसी अन्य OpenAI मॉडल को सत्यापित करने के लिए `--model <provider/model>` पास करें या
    `OPENCLAW_PARALLELS_OPENAI_MODEL` सेट करें।
  - लंबे स्थानीय रन को होस्ट टाइमआउट में लपेटें, ताकि Parallels ट्रांसपोर्ट रुकावटें
    शेष परीक्षण अवधि का उपभोग न कर सकें:

    ```bash
    timeout --foreground 150m pnpm test:parallels:npm-update -- --json
    timeout --foreground 90m pnpm test:parallels:npm-update -- --platform windows --json
    ```

  - स्क्रिप्ट नेस्टेड लेन लॉग
    `/tmp/openclaw-parallels-npm-update.*` के अंतर्गत लिखती है। बाहरी
    रैपर को अटका हुआ मानने से पहले `windows-update.log`,
    `macos-update.log`, या `linux-update.log` जाँचें।
  - ठंडे अतिथि पर Windows अपडेट को अपडेट-पश्चात doctor और
    पैकेज अपडेट कार्य में 10 से 15 मिनट लग सकते हैं; नेस्टेड npm डीबग लॉग आगे बढ़ रहा हो, तो यह अभी भी स्वस्थ है।
  - इस समग्र रैपर को अलग-अलग Parallels
    macOS, Windows, या Linux स्मोक लेन के समानांतर न चलाएँ। वे VM स्थिति साझा करते हैं और
    स्नैपशॉट पुनर्स्थापन, पैकेज सर्विंग, या अतिथि gateway स्थिति पर टकरा सकते हैं।
  - अपडेट-पश्चात प्रमाण सामान्य बंडल किया गया plugin सरफ़ेस चलाता है, क्योंकि
    वाक्, छवि निर्माण, और मीडिया
    समझ जैसी क्षमता फ़साड बंडल किए गए रनटाइम API के माध्यम से लोड होती हैं, भले ही एजेंट
    टर्न स्वयं केवल एक साधारण पाठ प्रतिक्रिया जाँचता हो।

- `pnpm openclaw qa aimock`
  - प्रत्यक्ष प्रोटोकॉल स्मोक परीक्षण के लिए केवल स्थानीय AIMock प्रदाता सर्वर
    प्रारंभ करता है।
- `pnpm openclaw qa matrix`
  - डिस्पोज़ेबल Docker-समर्थित Tuwunel होमसर्वर के विरुद्ध Matrix लाइव QA लेन
    चलाता है। केवल स्रोत चेकआउट के लिए - पैकेज किए गए इंस्टॉल में
    `qa-lab` शामिल नहीं होता।
  - संपूर्ण CLI, प्रोफ़ाइल/परिदृश्य कैटलॉग, परिवेश चर और आर्टिफ़ैक्ट लेआउट:
    [Matrix स्मोक लेन](/hi/concepts/qa-e2e-automation#matrix-smoke-lanes)।
- `pnpm openclaw qa telegram`
  - परिवेश से ड्राइवर और SUT बॉट टोकन का उपयोग करके वास्तविक निजी समूह के
    विरुद्ध Telegram लाइव QA लेन चलाता है।
  - इसके लिए `OPENCLAW_QA_TELEGRAM_GROUP_ID`,
    `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`, और
    `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN` आवश्यक हैं। समूह आईडी संख्यात्मक
    Telegram चैट आईडी होनी चाहिए।
  - साझा पूल किए गए क्रेडेंशियल के लिए `--credential-source convex` का समर्थन करता है।
    डिफ़ॉल्ट रूप से परिवेश मोड का उपयोग करें, या पूल किए गए लीज़ चुनने के लिए
    `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` सेट करें।
  - डिफ़ॉल्ट में कैनरी, उल्लेख गेटिंग, कमांड एड्रेसिंग, `/status`,
    बॉट-से-बॉट उल्लिखित उत्तर और कोर नेटिव कमांड उत्तर शामिल हैं।
    `mock-openai` डिफ़ॉल्ट में नियतात्मक उत्तर-शृंखला और
    Telegram अंतिम-संदेश स्ट्रीमिंग रिग्रेशन भी शामिल हैं। `session_status` जैसे
    वैकल्पिक प्रोब के लिए `--list-scenarios` का उपयोग करें।
  - कोई भी परिदृश्य विफल होने पर गैर-शून्य कोड के साथ बाहर निकलता है। विफलता
    निकास कोड के बिना आर्टिफ़ैक्ट के लिए `--allow-failures` का उपयोग करें।
  - एक ही निजी समूह में दो अलग-अलग बॉट आवश्यक हैं, और SUT बॉट को
    Telegram उपयोगकर्ता नाम उजागर करना चाहिए।
  - स्थिर बॉट-से-बॉट अवलोकन के लिए, दोनों बॉट हेतु `@BotFather` में
    Bot-to-Bot Communication Mode सक्षम करें और सुनिश्चित करें कि ड्राइवर बॉट
    समूह के बॉट ट्रैफ़िक को देख सकता है।
  - `.artifacts/qa-e2e/...` के अंतर्गत Telegram QA रिपोर्ट, सारांश और
    `qa-evidence.json` लिखता है। उत्तर देने वाले परिदृश्यों में ड्राइवर के भेजने के
    अनुरोध से देखे गए SUT उत्तर तक का RTT शामिल होता है।

`Mantis Telegram Live` इस लेन के चारों ओर PR-साक्ष्य रैपर है। यह
Convex से लीज़ किए गए Telegram क्रेडेंशियल के साथ उम्मीदवार रेफ़ चलाता है,
Crabbox डेस्कटॉप ब्राउज़र में संशोधित QA रिपोर्ट/साक्ष्य बंडल रेंडर करता है, MP4
साक्ष्य रिकॉर्ड करता है, गति-ट्रिम किया गया GIF जनरेट करता है, आर्टिफ़ैक्ट बंडल
अपलोड करता है, और `pr_number` सेट होने पर Mantis GitHub App के माध्यम से
इनलाइन PR साक्ष्य पोस्ट करता है। अनुरक्षक इसे Actions UI से `Mantis Scenario`
(`scenario_id: telegram-live`) के माध्यम से या सीधे पुल रिक्वेस्ट टिप्पणी से प्रारंभ कर सकते हैं:

```text
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
```

`Mantis Telegram Desktop Proof` PR दृश्य प्रमाण के लिए एजेंट-आधारित नेटिव Telegram Desktop
पहले/बाद रैपर है। इसे Actions UI से मुक्त-रूप `instructions` के साथ,
`Mantis Scenario` (`scenario_id:
telegram-desktop-proof`) के माध्यम से, या PR टिप्पणी से प्रारंभ करें:

```text
@openclaw-mantis telegram desktop proof
```

Mantis एजेंट PR पढ़ता है, तय करता है कि कौन-सा Telegram-दृश्य व्यवहार
परिवर्तन को प्रमाणित करता है, बेसलाइन और उम्मीदवार रेफ़ पर वास्तविक-उपयोगकर्ता
Crabbox Telegram Desktop प्रमाण लेन चलाता है, नेटिव GIF उपयोगी होने तक पुनरावृत्ति
करता है, युग्मित `motionPreview` मैनिफ़ेस्ट लिखता है, और `pr_number`
सेट होने पर Mantis GitHub App के माध्यम से वही 2-स्तंभ GIF तालिका पोस्ट करता है।

- `pnpm openclaw qa mantis telegram-desktop-builder`
  - Crabbox Linux डेस्कटॉप लीज़ पर लेता है या पुनः उपयोग करता है, नेटिव Telegram
    Desktop इंस्टॉल करता है, लीज़ किए गए Telegram SUT बॉट टोकन से OpenClaw
    कॉन्फ़िगर करता है, Gateway प्रारंभ करता है और दृश्यमान VNC डेस्कटॉप से
    स्क्रीनशॉट/MP4 साक्ष्य रिकॉर्ड करता है।
  - डिफ़ॉल्ट रूप से `--credential-source convex` का उपयोग करता है, ताकि वर्कफ़्लो को केवल
    Convex ब्रोकर सीक्रेट की आवश्यकता हो। `pnpm openclaw qa telegram` के समान
    `OPENCLAW_QA_TELEGRAM_*` चरों के साथ `--credential-source env` का उपयोग करें।
  - Telegram Desktop को अभी भी उपयोगकर्ता लॉगिन/प्रोफ़ाइल की आवश्यकता होती है।
    बॉट टोकन केवल OpenClaw को कॉन्फ़िगर करता है। base64 `.tgz`
    प्रोफ़ाइल संग्रह के लिए `--telegram-profile-archive-env <name>` का उपयोग करें, या `--keep-lease`
    का उपयोग करके एक बार VNC के माध्यम से मैन्युअल रूप से लॉग इन करें।
  - आउटपुट निर्देशिका के अंतर्गत `mantis-telegram-desktop-builder-report.md`,
    `mantis-telegram-desktop-builder-summary.json`,
    `telegram-desktop-builder.png`, और `telegram-desktop-builder.mp4`
    लिखता है।

लाइव ट्रांसपोर्ट लेन एक मानक अनुबंध साझा करती हैं ताकि नए ट्रांसपोर्ट में
असंगति न आए; प्रत्येक लेन का कवरेज मैट्रिक्स
[QA अवलोकन - लाइव ट्रांसपोर्ट कवरेज](/hi/concepts/qa-e2e-automation#live-transport-coverage) में है।
`qa-channel` व्यापक सिंथेटिक सुइट है और उस मैट्रिक्स का भाग नहीं है।

### Convex के माध्यम से साझा Telegram क्रेडेंशियल (v1)

जब लाइव ट्रांसपोर्ट QA के लिए `--credential-source convex` (या `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`)
सक्षम होता है, तो QA लैब Convex-समर्थित पूल से एक विशिष्ट लीज़ प्राप्त करती है,
लेन चलने के दौरान उस लीज़ को Heartbeat भेजती है और शटडाउन पर लीज़ छोड़ती है।
अनुभाग का नाम Discord, Slack और WhatsApp समर्थन से पहले का है; लीज़ अनुबंध
सभी प्रकारों में साझा है।

संदर्भ Convex प्रोजेक्ट स्कैफ़ोल्ड: `qa/convex-credential-broker/`

आवश्यक परिवेश चर:

- `OPENCLAW_QA_CONVEX_SITE_URL` (उदाहरण के लिए `https://your-deployment.convex.site`)
- चयनित भूमिका के लिए एक सीक्रेट:
  - `maintainer` के लिए `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`
  - `ci` के लिए `OPENCLAW_QA_CONVEX_SECRET_CI`
- क्रेडेंशियल भूमिका चयन:
  - CLI: `--credential-role maintainer|ci`
  - परिवेश डिफ़ॉल्ट: `OPENCLAW_QA_CREDENTIAL_ROLE` (CI में डिफ़ॉल्ट `ci`, अन्यथा `maintainer`)

वैकल्पिक परिवेश चर:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS` (डिफ़ॉल्ट `1200000`)
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS` (डिफ़ॉल्ट `30000`)
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS` (डिफ़ॉल्ट `90000`)
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS` (डिफ़ॉल्ट `15000`)
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX` (डिफ़ॉल्ट `/qa-credentials/v1`)
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID` (वैकल्पिक ट्रेस आईडी)
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` केवल स्थानीय विकास के लिए लूपबैक `http://` Convex URL की अनुमति देता है।

सामान्य संचालन में `OPENCLAW_QA_CONVEX_SITE_URL` को `https://` का उपयोग करना चाहिए।

अनुरक्षक व्यवस्थापक कमांड (पूल जोड़ना/हटाना/सूचीबद्ध करना) के लिए विशेष रूप से
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` आवश्यक है।

अनुरक्षकों के लिए CLI सहायक:

```bash
pnpm openclaw qa credentials doctor
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

सीक्रेट मान प्रिंट किए बिना Convex साइट URL, ब्रोकर सीक्रेट, एंडपॉइंट प्रीफ़िक्स,
HTTP टाइमआउट और व्यवस्थापक/सूची पहुँच की जाँच करने के लिए लाइव रन से पहले
`doctor` का उपयोग करें। स्क्रिप्ट और CI उपयोगिताओं में मशीन-पठनीय
आउटपुट के लिए `--json` का उपयोग करें।

डिफ़ॉल्ट एंडपॉइंट अनुबंध (`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`)।
अनुरोध `Authorization: Bearer <role secret>` हेडर से प्रमाणित होते हैं;
नीचे दिए गए बॉडी में वह हेडर शामिल नहीं है:

- `POST /acquire`
  - अनुरोध: `{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - सफलता: `{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - समाप्त/पुनः प्रयास योग्य: `{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /payload-chunk`
  - अनुरोध: `{ kind, ownerId, actorRole, credentialId, leaseToken, index }`
  - सफलता: `{ status: "ok", index, data }`
- `POST /heartbeat`
  - अनुरोध: `{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - सफलता: `{ status: "ok" }` (या रिक्त `2xx`)
- `POST /release`
  - अनुरोध: `{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - सफलता: `{ status: "ok" }` (या रिक्त `2xx`)
- `POST /admin/add` (केवल अनुरक्षक सीक्रेट)
  - अनुरोध: `{ kind, actorId, payload, note?, status? }`
  - सफलता: `{ status: "ok", credential }`
- `POST /admin/remove` (केवल अनुरक्षक सीक्रेट)
  - अनुरोध: `{ credentialId, actorId }`
  - सफलता: `{ status: "ok", changed, credential }`
  - सक्रिय लीज़ सुरक्षा: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list` (केवल अनुरक्षक सीक्रेट)
  - अनुरोध: `{ kind?, status?, includePayload?, limit? }`
  - सफलता: `{ status: "ok", credentials, count }`

Telegram प्रकार के लिए पेलोड संरचना:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` एक संख्यात्मक Telegram चैट आईडी स्ट्रिंग होनी चाहिए।
- `admin/add`, `kind: "telegram"` के लिए इस संरचना को सत्यापित करता है और विकृत पेलोड अस्वीकार करता है।

Telegram वास्तविक-उपयोगकर्ता प्रकार के लिए पेलोड संरचना:

- `{ groupId: string, sutToken: string, testerUserId: string, testerUsername: string, telegramApiId: string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string, tdlibArchiveBase64: string, tdlibArchiveSha256: string, desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }`
- `groupId`, `testerUserId`, और `telegramApiId` संख्यात्मक स्ट्रिंग होनी चाहिए।
- `tdlibArchiveSha256` और `desktopTdataArchiveSha256` SHA-256 हेक्स स्ट्रिंग होनी चाहिए।
- `kind: "telegram-user"` Mantis Telegram Desktop प्रमाण वर्कफ़्लो के लिए आरक्षित है। सामान्य QA लैब लेन को इसे प्राप्त नहीं करना चाहिए।

ब्रोकर द्वारा सत्यापित बहु-चैनल पेलोड:

- Discord: `{ guildId: string, channelId: string, driverBotToken: string, sutBotToken: string, sutApplicationId: string, voiceChannelId?: string }`
- WhatsApp: `{ driverPhoneE164: string, sutPhoneE164: string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string, groupJid?: string }`

Slack लेन भी पूल से लीज़ ले सकती हैं, लेकिन Slack पेलोड सत्यापन
वर्तमान में ब्रोकर के बजाय Slack QA रनर में होता है। Slack पंक्तियों के लिए
`{ channelId: string, driverBotToken: string, sutBotToken: string, sutAppToken: string }`
का उपयोग करें।

### QA में चैनल जोड़ना

नए चैनल अडैप्टर के लिए आर्किटेक्चर और परिदृश्य-सहायक नाम
[QA अवलोकन - चैनल जोड़ना](/hi/concepts/qa-e2e-automation#adding-a-channel) में हैं।
न्यूनतम मानदंड: साझा `qa-lab` होस्ट सीम पर ट्रांसपोर्ट रनर लागू करें,
साझा परिदृश्यों के लिए `adapterFactory` जोड़ें, Plugin मैनिफ़ेस्ट में
`qaRunners` घोषित करें, `openclaw qa <runner>` के रूप में माउंट करें और
`qa/scenarios/` के अंतर्गत परिदृश्य लिखें।

## परीक्षण सुइट (कहाँ क्या चलता है)

सुइट को "बढ़ती यथार्थता" (और बढ़ती अस्थिरता/लागत) के रूप में समझें।

### यूनिट / एकीकरण (डिफ़ॉल्ट)

- कमांड: `pnpm test`
- कॉन्फ़िगरेशन: अलक्षित रन `vitest.full-*.config.ts` शार्ड सेट का उपयोग करते हैं और
  समानांतर शेड्यूलिंग के लिए बहु-प्रोजेक्ट शार्ड को प्रति-प्रोजेक्ट कॉन्फ़िगरेशन में
  विस्तृत कर सकते हैं
- फ़ाइलें: `src/**/*.test.ts`,
  `packages/**/*.test.ts`, और `test/**/*.test.ts` के अंतर्गत कोर/यूनिट इन्वेंटरी;
  UI यूनिट परीक्षण समर्पित `unit-ui` शार्ड में चलते हैं
- दायरा:
  - शुद्ध यूनिट परीक्षण
  - इन-प्रोसेस एकीकरण परीक्षण (Gateway प्रमाणीकरण, रूटिंग, टूलिंग, पार्सिंग, कॉन्फ़िगरेशन)
  - ज्ञात बग के लिए नियतात्मक रिग्रेशन
- अपेक्षाएँ:
  - CI में चलता है
  - वास्तविक कुंजियों की आवश्यकता नहीं
  - तेज़ और स्थिर होना चाहिए
  - रिज़ॉल्वर और सार्वजनिक-सतह लोडर परीक्षणों को वास्तविक बंडल किए गए Plugin
    स्रोत API के बजाय जनरेट किए गए छोटे Plugin फ़िक्स्चर के साथ व्यापक
    `api.js` और `runtime-api.js` फ़ॉलबैक व्यवहार प्रमाणित करना चाहिए।
    वास्तविक Plugin API लोड Plugin-स्वामित्व वाले अनुबंध/एकीकरण सुइट में होने चाहिए।

नेटिव निर्भरता नीति:

- डिफ़ॉल्ट परीक्षण इंस्टॉल वैकल्पिक नेटिव Discord opus बिल्ड छोड़ देते हैं।
  Discord वॉइस बंडल किए गए `libopus-wasm` का उपयोग करती है, और
  `allowBuilds` में `@discordjs/opus` अक्षम रहता है ताकि स्थानीय परीक्षण और
  Testbox लेन नेटिव ऐड-ऑन संकलित न करें।
- नेटिव opus प्रदर्शन की तुलना डिफ़ॉल्ट OpenClaw इंस्टॉल/परीक्षण लूप में नहीं,
  बल्कि `libopus-wasm` बेंचमार्क रेपो में करें। डिफ़ॉल्ट
  `allowBuilds` में `@discordjs/opus` को `true` पर सेट न करें;
  इससे असंबंधित इंस्टॉल/परीक्षण लूप नेटिव कोड संकलित करने लगते हैं।

<AccordionGroup>
  <Accordion title="प्रोजेक्ट, शार्ड और सीमित लेन">

    - बिना लक्ष्य वाले `pnpm test` एक विशाल नेटिव रूट-प्रोजेक्ट प्रक्रिया के बजाय तेरह छोटे शार्ड कॉन्फ़िगरेशन (`core-unit-fast`, `core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-tooling`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) चलाते हैं। इससे लोड वाली मशीनों पर अधिकतम RSS घटता है और ऑटो-रिप्लाई/Plugin कार्य असंबंधित सुइट्स के संसाधन नहीं रोकता।
    - `pnpm test --watch` अब भी नेटिव रूट `vitest.config.ts` प्रोजेक्ट ग्राफ़ का उपयोग करता है, क्योंकि मल्टी-शार्ड वॉच लूप व्यावहारिक नहीं है।
    - `pnpm test`, `pnpm test:watch`, और `pnpm test:perf:imports` स्पष्ट फ़ाइल/डायरेक्टरी लक्ष्यों को पहले सीमित लेनों से रूट करते हैं, इसलिए `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` को पूरे रूट प्रोजेक्ट की स्टार्टअप लागत नहीं चुकानी पड़ती।
    - `pnpm test:changed` बदले हुए git पथों को डिफ़ॉल्ट रूप से कम लागत वाली सीमित लेनों में विस्तारित करता है: सीधे परीक्षण संपादन, सहोदर `*.test.ts` फ़ाइलें, स्पष्ट स्रोत मैपिंग और स्थानीय इम्पोर्ट-ग्राफ़ पर निर्भर फ़ाइलें। कॉन्फ़िगरेशन/सेटअप/पैकेज संपादन व्यापक रूप से परीक्षण नहीं चलाते, जब तक आप स्पष्ट रूप से `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` का उपयोग न करें।
    - `pnpm check:changed` सीमित कार्य के लिए सामान्य स्मार्ट स्थानीय जाँच गेट है। यह अंतर को कोर, कोर परीक्षणों, एक्सटेंशन, एक्सटेंशन परीक्षणों, ऐप्स, दस्तावेज़ों, रिलीज़ मेटाडेटा, लाइव Docker टूलिंग और टूलिंग में वर्गीकृत करता है, फिर उनसे मेल खाने वाले टाइपचेक, लिंट और गार्ड कमांड चलाता है। यह Vitest परीक्षण नहीं चलाता; परीक्षण प्रमाण के लिए `pnpm test:changed` या स्पष्ट `pnpm test <target>` चलाएँ। केवल रिलीज़ मेटाडेटा वाले संस्करण बदलाव लक्षित संस्करण/कॉन्फ़िगरेशन/रूट-निर्भरता जाँच चलाते हैं, साथ ही एक गार्ड शीर्ष-स्तरीय संस्करण फ़ील्ड के बाहर पैकेज बदलावों को अस्वीकार करता है।
    - लाइव Docker ACP हार्नेस संपादन केंद्रित जाँच चलाते हैं: लाइव Docker प्रमाणीकरण स्क्रिप्ट के लिए शेल सिंटैक्स और लाइव Docker शेड्यूलर का ड्राई-रन। `package.json` बदलाव केवल तब शामिल होते हैं जब अंतर `scripts["test:docker:live-*"]` तक सीमित हो; निर्भरता, एक्सपोर्ट, संस्करण और पैकेज सतह के अन्य संपादन अब भी व्यापक गार्ड का उपयोग करते हैं।
    - एजेंट, कमांड, Plugin, ऑटो-रिप्लाई सहायकों, `plugin-sdk` और इसी तरह के शुद्ध उपयोगिता क्षेत्रों के कम इम्पोर्ट वाले यूनिट परीक्षण `unit-fast` लेन से रूट होते हैं, जो `test/setup-openclaw-runtime.ts` को छोड़ देती है; स्टेटफ़ुल/रनटाइम-भारी फ़ाइलें मौजूदा लेनों पर रहती हैं।
    - चुनी गई `plugin-sdk` और `commands` सहायक स्रोत फ़ाइलें भी बदले-मोड रन को उन हल्की लेनों में स्पष्ट सहोदर परीक्षणों से मैप करती हैं, ताकि सहायक संपादनों के कारण उस डायरेक्टरी का पूरा भारी सुइट दोबारा न चले।
    - `auto-reply` में शीर्ष-स्तरीय कोर सहायकों, शीर्ष-स्तरीय `reply.*` एकीकरण परीक्षणों और `src/auto-reply/reply/**` उपट्री के लिए समर्पित बकेट हैं। CI रिप्लाई उपट्री को आगे एजेंट-रनर, डिस्पैच और कमांड/स्टेट-रूटिंग शार्ड में बाँटता है, ताकि एक इम्पोर्ट-भारी बकेट पूरे Node टेल का भार न उठाए।
    - सामान्य PR/मुख्य CI जानबूझकर बंडल किए गए Plugin बैच स्वीप और केवल रिलीज़ वाले `agentic-plugins` शार्ड को छोड़ देता है। पूर्ण रिलीज़ सत्यापन रिलीज़ उम्मीदवारों पर उन Plugin-भारी सुइट्स के लिए अलग `Plugin Prerelease` चाइल्ड वर्कफ़्लो डिस्पैच करता है।

  </Accordion>

  <Accordion title="एम्बेडेड रनर कवरेज">

    - संदेश-टूल खोज इनपुट या Compaction रनटाइम
      संदर्भ बदलते समय, कवरेज के दोनों स्तर बनाए रखें।
    - शुद्ध रूटिंग और सामान्यीकरण
      सीमाओं के लिए केंद्रित सहायक रिग्रेशन जोड़ें।
    - एम्बेडेड रनर एकीकरण सुइट्स को स्वस्थ रखें:
      `src/agents/embedded-agent-runner/compact.hooks.test.ts`,
      `src/agents/embedded-agent-runner/run.overflow-compaction.test.ts`, और
      `src/agents/embedded-agent-runner/run.overflow-compaction.loop.test.ts`।
    - ये सुइट सत्यापित करते हैं कि सीमित आईडी और Compaction व्यवहार अब भी
      वास्तविक `run.ts` / `compact.ts` पथों से प्रवाहित होते हैं; केवल सहायक परीक्षण
      उन एकीकरण पथों के लिए पर्याप्त विकल्प नहीं हैं।

  </Accordion>

  <Accordion title="Vitest पूल और आइसोलेशन डिफ़ॉल्ट">

    - मूल Vitest कॉन्फ़िगरेशन डिफ़ॉल्ट रूप से `threads` का उपयोग करता है।
    - साझा Vitest कॉन्फ़िगरेशन `isolate: false` को नियत करता है और
      रूट प्रोजेक्ट, e2e तथा लाइव कॉन्फ़िगरेशन में नॉन-आइसोलेटेड रनर का उपयोग करता है।
    - रूट UI लेन अपना `jsdom` सेटअप और ऑप्टिमाइज़र बनाए रखती है, लेकिन
      साझा नॉन-आइसोलेटेड रनर पर भी चलती है।
    - प्रत्येक `pnpm test` शार्ड साझा Vitest कॉन्फ़िगरेशन से समान `threads` + `isolate: false`
      डिफ़ॉल्ट प्राप्त करता है।
    - `scripts/run-vitest.mjs` बड़े स्थानीय रन के दौरान V8 कंपाइल गतिविधि घटाने के लिए
      डिफ़ॉल्ट रूप से Vitest चाइल्ड Node प्रक्रियाओं हेतु `--no-maglev` जोड़ता है।
      मानक V8 व्यवहार से तुलना करने के लिए `OPENCLAW_VITEST_ENABLE_MAGLEV=1` सेट करें।
    - `scripts/run-vitest.mjs` बिना stdout या stderr आउटपुट के 5 मिनट बाद
      स्पष्ट नॉन-वॉच Vitest रन समाप्त कर देता है। जानबूझकर मौन जाँच के लिए
      वॉचडॉग अक्षम करने हेतु `OPENCLAW_VITEST_NO_OUTPUT_TIMEOUT_MS=0` सेट करें।

  </Accordion>

  <Accordion title="तेज़ स्थानीय पुनरावृत्ति">

    - `pnpm changed:lanes` दिखाता है कि कोई अंतर किन आर्किटेक्चरल लेनों को ट्रिगर करता है।
    - प्री-कमिट हुक केवल फ़ॉर्मैटिंग करता है। यह फ़ॉर्मैट की गई फ़ाइलों को
      फिर से स्टेज करता है और लिंट, टाइपचेक या परीक्षण नहीं चलाता।
    - स्मार्ट स्थानीय जाँच गेट की आवश्यकता होने पर हैंडऑफ़ या पुश से पहले
      स्पष्ट रूप से `pnpm check:changed` चलाएँ।
    - `pnpm test:changed` डिफ़ॉल्ट रूप से कम लागत वाली सीमित लेनों से रूट होता है। `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` का उपयोग
      केवल तभी करें जब एजेंट तय करे कि किसी हार्नेस, कॉन्फ़िगरेशन, पैकेज या अनुबंध संपादन को वास्तव में
      अधिक व्यापक Vitest कवरेज चाहिए।
    - `pnpm test:max` और `pnpm test:changed:max` समान रूटिंग
      व्यवहार बनाए रखते हैं, बस उच्च वर्कर सीमा के साथ।
    - स्थानीय वर्कर का ऑटो-स्केलिंग जानबूझकर संयमित है और होस्ट का लोड औसत
      पहले से अधिक होने पर पीछे हट जाता है, इसलिए एक साथ चलने वाले कई
      Vitest रन डिफ़ॉल्ट रूप से कम नुकसान करते हैं।
    - मूल Vitest कॉन्फ़िगरेशन प्रोजेक्ट/कॉन्फ़िगरेशन फ़ाइलों को
      `forceRerunTriggers` के रूप में चिह्नित करता है, ताकि परीक्षण वायरिंग बदलने पर बदले-मोड के पुनः रन
      सही रहें।
    - कॉन्फ़िगरेशन समर्थित होस्ट पर `OPENCLAW_VITEST_FS_MODULE_CACHE` को सक्षम रखता है;
      प्रत्यक्ष प्रोफ़ाइलिंग के लिए एक स्पष्ट कैश स्थान हेतु `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`
      सेट करें।

  </Accordion>

  <Accordion title="प्रदर्शन डीबगिंग">

    - `pnpm test:perf:imports` Vitest इम्पोर्ट-अवधि रिपोर्टिंग और
      इम्पोर्ट-ब्रेकडाउन आउटपुट सक्षम करता है।
    - `pnpm test:perf:imports:changed` उसी प्रोफ़ाइलिंग दृश्य को
      `origin/main` के बाद बदली गई फ़ाइलों तक सीमित करता है।
    - शार्ड समय डेटा `.artifacts/vitest-shard-timings.json` में लिखा जाता है।
      पूरे-कॉन्फ़िगरेशन रन कॉन्फ़िगरेशन पथ को कुंजी के रूप में उपयोग करते हैं; शामिल-पैटर्न वाले CI
      शार्ड शार्ड नाम जोड़ते हैं, ताकि फ़िल्टर किए गए शार्ड अलग से ट्रैक किए जा सकें।
    - जब कोई सक्रिय परीक्षण अब भी अपना अधिकांश समय स्टार्टअप इम्पोर्ट में बिताता है,
      तो भारी निर्भरताओं को संकीर्ण स्थानीय `*.runtime.ts` सीम के पीछे रखें और
      रनटाइम सहायकों को केवल `vi.mock(...)` से गुजारने के लिए डीप-इम्पोर्ट करने के बजाय
      उस सीम को सीधे मॉक करें।
    - `pnpm test:perf:changed:bench -- --ref <git-ref>` उस कमिट किए गए अंतर के लिए रूट किए गए
      `test:changed` की तुलना नेटिव रूट-प्रोजेक्ट पथ से करता है और
      दीवार-घड़ी समय के साथ macOS का अधिकतम RSS प्रिंट करता है।
    - `pnpm test:perf:changed:bench -- --worktree` बदली हुई फ़ाइल सूची को
      `scripts/test-projects.mjs` और रूट Vitest कॉन्फ़िगरेशन से रूट करके वर्तमान
      डर्टी ट्री का बेंचमार्क करता है।
    - `pnpm test:perf:profile:main` Vitest/Vite स्टार्टअप और ट्रांसफ़ॉर्म ओवरहेड के लिए
      मुख्य-थ्रेड CPU प्रोफ़ाइल लिखता है।
    - `pnpm test:perf:profile:runner` फ़ाइल समानांतरता अक्षम करके
      यूनिट सुइट के लिए रनर CPU+हीप प्रोफ़ाइल लिखता है।

  </Accordion>
</AccordionGroup>

### स्थिरता (Gateway)

- कमांड: `pnpm test:stability:gateway`
- कॉन्फ़िगरेशन: `test/vitest/vitest.gateway.config.ts`, `test/vitest/vitest.logging.config.ts`, और `test/vitest/vitest.infra.config.ts`, प्रत्येक को एक वर्कर पर बाध्य किया गया है
- दायरा:
  - डिफ़ॉल्ट रूप से डायग्नोस्टिक्स सक्षम करके वास्तविक लूपबैक Gateway शुरू करता है
  - डायग्नोस्टिक इवेंट पथ के माध्यम से सिंथेटिक Gateway संदेश, मेमोरी और बड़े-पेलोड का तीव्र आवागमन कराता है
  - Gateway WS RPC पर `diagnostics.stability` की क्वेरी करता है
  - डायग्नोस्टिक स्थिरता बंडल के परसिस्टेंस सहायकों को कवर करता है
  - सुनिश्चित करता है कि रिकॉर्डर सीमित रहे, सिंथेटिक RSS नमूने दबाव बजट के भीतर रहें और प्रति-सत्र कतार गहराइयाँ वापस शून्य तक खाली हो जाएँ
- अपेक्षाएँ:
  - CI-सुरक्षित और बिना कुंजी के
  - स्थिरता-रिग्रेशन अनुवर्ती कार्रवाई के लिए सीमित लेन, पूर्ण Gateway सुइट का विकल्प नहीं

### E2E (रेपो समुच्चय)

- कमांड: `pnpm test:e2e`
- दायरा:
  - Gateway स्मोक E2E लेन चलाता है
  - मॉक की गई Control UI ब्राउज़र E2E लेन चलाता है
- अपेक्षाएँ:
  - CI-सुरक्षित और बिना कुंजी के
  - Playwright Chromium इंस्टॉल होना आवश्यक है

### E2E (Gateway स्मोक)

- कमांड: `pnpm test:e2e:gateway`
- कॉन्फ़िगरेशन: `test/vitest/vitest.e2e.config.ts`
- फ़ाइलें: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`, और `extensions/` के अंतर्गत बंडल किए गए Plugin के E2E परीक्षण
- रनटाइम डिफ़ॉल्ट:
  - बाकी रेपो के अनुरूप, Vitest `threads` का `isolate: false` के साथ उपयोग करता है।
  - अनुकूली वर्कर का उपयोग करता है (CI: अधिकतम 2, स्थानीय: डिफ़ॉल्ट रूप से 1)।
  - कंसोल I/O ओवरहेड घटाने के लिए डिफ़ॉल्ट रूप से मौन मोड में चलता है।
- उपयोगी ओवरराइड:
  - `OPENCLAW_E2E_WORKERS=<n>` वर्कर संख्या बाध्य करने के लिए (अधिकतम सीमा 16)।
  - `OPENCLAW_E2E_VERBOSE=1` विस्तृत कंसोल आउटपुट फिर से सक्षम करने के लिए।
- दायरा:
  - बहु-इंस्टेंस Gateway का एंड-टू-एंड व्यवहार
  - WebSocket/HTTP सतहें, Node पेयरिंग और अधिक भारी नेटवर्किंग
- अपेक्षाएँ:
  - CI में चलता है (पाइपलाइन में सक्षम होने पर)
  - वास्तविक कुंजियाँ आवश्यक नहीं हैं
  - यूनिट परीक्षणों की तुलना में अधिक गतिशील भाग (धीमा हो सकता है)

### E2E (Control UI मॉक किया गया ब्राउज़र)

- कमांड: `pnpm test:ui:e2e`
- कॉन्फ़िगरेशन: `test/vitest/vitest.ui-e2e.config.ts`
- फ़ाइलें: `ui/src/**/*.e2e.test.ts`
- दायरा:
  - Vite Control UI शुरू करता है
  - Playwright के माध्यम से वास्तविक Chromium पृष्ठ चलाता है
  - Gateway WebSocket को नियतात्मक इन-ब्राउज़र मॉक से बदलता है
- अपेक्षाएँ:
  - CI में `pnpm test:e2e` के भाग के रूप में चलता है
  - वास्तविक Gateway, एजेंट या प्रदाता कुंजियाँ आवश्यक नहीं हैं
  - ब्राउज़र निर्भरता मौजूद होनी चाहिए (`pnpm --dir ui exec playwright install chromium`)

### E2E: OpenShell बैकएंड स्मोक

- कमांड: `pnpm test:e2e:openshell`
- फ़ाइल: `extensions/openshell/src/backend.e2e.test.ts`
- दायरा:
  - सक्रिय स्थानीय OpenShell Gateway का पुनः उपयोग करता है
  - अस्थायी स्थानीय Dockerfile से सैंडबॉक्स बनाता है
  - वास्तविक `sandbox ssh-config` + SSH exec पर OpenClaw के OpenShell बैकएंड का अभ्यास करता है
  - सैंडबॉक्स fs ब्रिज के माध्यम से रिमोट-कैनोनिकल फ़ाइल सिस्टम व्यवहार सत्यापित करता है
- अपेक्षाएँ:
  - केवल ऑप्ट-इन; डिफ़ॉल्ट `pnpm test:e2e` रन का भाग नहीं
  - स्थानीय `openshell` CLI और कार्यशील Docker डेमन आवश्यक हैं
  - सक्रिय स्थानीय OpenShell Gateway और उसका कॉन्फ़िगरेशन स्रोत आवश्यक है
  - पृथक `HOME` / `XDG_CONFIG_HOME` का उपयोग करता है, फिर परीक्षण सैंडबॉक्स नष्ट कर देता है
- उपयोगी ओवरराइड:
  - व्यापक e2e सुइट मैन्युअल रूप से चलाते समय परीक्षण सक्षम करने के लिए `OPENCLAW_E2E_OPENSHELL=1`
  - गैर-डिफ़ॉल्ट CLI बाइनरी या रैपर स्क्रिप्ट निर्दिष्ट करने के लिए `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`
  - पंजीकृत Gateway कॉन्फ़िगरेशन को पृथक परीक्षण में उपलब्ध कराने के लिए `OPENCLAW_E2E_OPENSHELL_CONFIG_HOME=/path/to/config`
  - होस्ट नीति फ़िक्स्चर द्वारा उपयोग किए जाने वाले Docker Gateway IP को ओवरराइड करने के लिए `OPENCLAW_E2E_OPENSHELL_HOST_IP=172.18.0.1`

### लाइव (वास्तविक प्रदाता + वास्तविक मॉडल)

- कमांड: `pnpm test:live`
- कॉन्फ़िगरेशन: `test/vitest/vitest.live.config.ts`
- फ़ाइलें: `src/**/*.live.test.ts`, `test/**/*.live.test.ts`, और `extensions/` के अंतर्गत बंडल किए गए Plugin के लाइव परीक्षण
- डिफ़ॉल्ट: `pnpm test:live` द्वारा **सक्षम** (`OPENCLAW_LIVE_TEST=1` सेट करता है)
- दायरा:
  - "क्या यह प्रदाता/मॉडल वास्तविक क्रेडेंशियल के साथ वास्तव में _आज_ काम करता है?"
  - प्रदाता फ़ॉर्मैट में बदलाव, टूल-कॉलिंग की विचित्रताएँ, प्रमाणीकरण संबंधी समस्याएँ और दर-सीमा व्यवहार पकड़ना
- अपेक्षाएँ:
  - डिज़ाइन के अनुसार CI-स्थिर नहीं (वास्तविक नेटवर्क, वास्तविक प्रदाता नीतियाँ, कोटा, आउटेज)
  - पैसा खर्च होता है / दर सीमाओं का उपयोग होता है
  - "सब कुछ" चलाने के बजाय सीमित उपसमुच्चय चलाना बेहतर है
- लाइव रन पहले से निर्यात की गई API कुंजियों और तैयार किए गए प्रमाणीकरण प्रोफ़ाइल का उपयोग करते हैं।
- डिफ़ॉल्ट रूप से, लाइव रन फिर भी `HOME` को अलग रखते हैं और कॉन्फ़िगरेशन/प्रमाणीकरण सामग्री को एक अस्थायी परीक्षण होम में कॉपी करते हैं, ताकि यूनिट फ़िक्स्चर आपके वास्तविक `~/.openclaw` को बदल न सकें।
- `OPENCLAW_LIVE_USE_REAL_HOME=1` केवल तभी सेट करें, जब आप जानबूझकर चाहते हों कि लाइव परीक्षण आपकी वास्तविक होम डायरेक्टरी का उपयोग करें।
- `pnpm test:live` डिफ़ॉल्ट रूप से अधिक शांत मोड का उपयोग करता है: यह `[live] ...` की प्रगति आउटपुट बनाए रखता है और Gateway बूटस्ट्रैप लॉग/Bonjour की अनावश्यक संदेशबाज़ी को म्यूट करता है। यदि आप पूरे स्टार्टअप लॉग वापस चाहते हैं, तो `OPENCLAW_LIVE_TEST_QUIET=0` सेट करें।
- API कुंजी रोटेशन (प्रदाता-विशिष्ट): कॉमा/सेमीकोलन फ़ॉर्मैट के साथ `*_API_KEYS` या `*_API_KEY_1`, `*_API_KEY_2` सेट करें (उदाहरण के लिए `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`), अथवा `OPENCLAW_LIVE_*_KEY` के माध्यम से प्रत्येक लाइव रन के लिए ओवरराइड करें; परीक्षण दर-सीमा प्रतिक्रियाओं पर पुनः प्रयास करते हैं।
- प्रगति/Heartbeat आउटपुट:
  - लाइव सूट stderr पर प्रगति पंक्तियाँ भेजते हैं, ताकि लंबे प्रदाता कॉल स्पष्ट रूप से सक्रिय दिखें, भले ही Vitest का कंसोल कैप्चर शांत हो।
  - `test/vitest/vitest.live.config.ts` Vitest कंसोल इंटरसेप्शन अक्षम करता है, ताकि लाइव रन के दौरान प्रदाता/Gateway की प्रगति पंक्तियाँ तुरंत स्ट्रीम हों।
  - प्रत्यक्ष-मॉडल Heartbeat को `OPENCLAW_LIVE_HEARTBEAT_MS` से समायोजित करें।
  - Gateway/प्रोब Heartbeat को `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` से समायोजित करें।

## मुझे कौन-सा सूट चलाना चाहिए?

इस निर्णय तालिका का उपयोग करें:

- लॉजिक/परीक्षण संपादित करना: `pnpm test` चलाएँ (और यदि आपने बहुत कुछ बदला है, तो `pnpm test:coverage` भी)
- Gateway नेटवर्किंग / WS प्रोटोकॉल / पेयरिंग में बदलाव करना: `pnpm test:e2e` जोड़ें
- "मेरा बॉट बंद है" / प्रदाता-विशिष्ट विफलताओं / टूल कॉलिंग को डीबग करना: सीमित `pnpm test:live` चलाएँ

## लाइव (नेटवर्क का उपयोग करने वाले) परीक्षण

लाइव मॉडल मैट्रिक्स, CLI बैकएंड स्मोक, ACP स्मोक, Codex ऐप-सर्वर
हार्नेस और सभी मीडिया-प्रदाता लाइव परीक्षणों (Deepgram, BytePlus, ComfyUI,
इमेज, संगीत, वीडियो, मीडिया हार्नेस) के साथ-साथ लाइव रन के लिए क्रेडेंशियल प्रबंधन हेतु

- [लाइव सूट का परीक्षण](/hi/help/testing-live) देखें। समर्पित अपडेट और
  Plugin सत्यापन चेकलिस्ट के लिए,
  [अपडेट और Plugin का परीक्षण](/hi/help/testing-updates-plugins) देखें।

## Docker रनर (वैकल्पिक "Linux में काम करता है" जाँच)

ये Docker रनर दो समूहों में विभाजित हैं:

- लाइव-मॉडल रनर: `test:docker:live-models` और `test:docker:live-gateway` रेपो Docker इमेज (`src/agents/models.profiles.live.test.ts` और `src/gateway/gateway-models.profiles.live.test.ts`) के अंदर केवल अपनी मेल खाती प्रोफ़ाइल-कुंजी लाइव फ़ाइल चलाते हैं और आपकी स्थानीय कॉन्फ़िगरेशन डायरेक्टरी, कार्यक्षेत्र तथा वैकल्पिक प्रोफ़ाइल env फ़ाइल माउंट करते हैं। इनके मेल खाते स्थानीय एंट्रीपॉइंट `test:live:models-profiles` और `test:live:gateway-profiles` हैं।
- जहाँ आवश्यक हो, Docker लाइव रनर अपनी व्यावहारिक सीमाएँ बनाए रखते हैं:
  `test:docker:live-models` डिफ़ॉल्ट रूप से चुने हुए समर्थित उच्च-संकेत सेट का उपयोग करता है, और
  `test:docker:live-gateway` डिफ़ॉल्ट रूप से `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`, और
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000` का उपयोग करता है। जब आप स्पष्ट रूप से छोटी सीमा या बड़ा स्कैन चाहते हों, तो `OPENCLAW_LIVE_MAX_MODELS`
  या Gateway env वेरिएबल सेट करें।
- `test:docker:all` पहले `test:docker:live-build` के माध्यम से लाइव Docker इमेज एक बार बनाता है, फिर `scripts/package-openclaw-for-docker.mjs` के माध्यम से OpenClaw को npm टारबॉल के रूप में एक बार पैक करता है और उसके बाद दो `scripts/e2e/Dockerfile` इमेज बनाता/पुनः उपयोग करता है। बेयर इमेज केवल इंस्टॉल/अपडेट/Plugin-निर्भरता लेन के लिए Node/Git रनर है; वे लेन पहले से निर्मित टारबॉल माउंट करती हैं। कार्यात्मक इमेज बिल्ट-ऐप कार्यक्षमता लेन के लिए उसी टारबॉल को `/app` में इंस्टॉल करती है। Docker लेन की परिभाषाएँ `scripts/lib/docker-e2e-scenarios.mjs` में हैं; प्लानर लॉजिक `scripts/lib/docker-e2e-plan.mjs` में है; `scripts/test-docker-all.mjs` चयनित योजना को निष्पादित करता है। समग्र प्रक्रिया भारित स्थानीय शेड्यूलर का उपयोग करती है: `OPENCLAW_DOCKER_ALL_PARALLELISM` प्रक्रिया स्लॉट नियंत्रित करता है, जबकि संसाधन सीमाएँ भारी लाइव, npm-इंस्टॉल और बहु-सेवा लेन को एक साथ शुरू होने से रोकती हैं। यदि कोई एक लेन सक्रिय सीमाओं से भारी है, तो पूल खाली होने पर शेड्यूलर फिर भी उसे शुरू कर सकता है और क्षमता दोबारा उपलब्ध होने तक उसे अकेले चलाता है। डिफ़ॉल्ट 10 स्लॉट, `OPENCLAW_DOCKER_ALL_LIVE_LIMIT=9`, `OPENCLAW_DOCKER_ALL_NPM_LIMIT=5`, और `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT=7` हैं; `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` या `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT` (और अन्य `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT` ओवरराइड) केवल तभी समायोजित करें, जब Docker होस्ट में अधिक अतिरिक्त क्षमता हो। रनर डिफ़ॉल्ट रूप से Docker प्रीफ़्लाइट करता है, पुराने OpenClaw E2E कंटेनर हटाता है, हर 30 सेकंड में स्थिति प्रिंट करता है, सफल लेन की टाइमिंग `.artifacts/docker-tests/lane-timings.json` में संग्रहीत करता है और बाद के रन में लंबी लेन पहले शुरू करने के लिए उन टाइमिंग का उपयोग करता है। बिना Docker बनाए या चलाए भारित लेन मैनिफ़ेस्ट प्रिंट करने के लिए `OPENCLAW_DOCKER_ALL_DRY_RUN=1`, या चयनित लेन, पैकेज/इमेज आवश्यकताओं और क्रेडेंशियल के लिए CI योजना प्रिंट करने हेतु `node scripts/test-docker-all.mjs --plan-json` का उपयोग करें।
- `Package Acceptance` इस प्रश्न के लिए GitHub-मूल पैकेज गेट है: "क्या यह इंस्टॉल करने योग्य टारबॉल एक उत्पाद के रूप में काम करता है?" यह `source=npm`, `source=ref`, `source=url`, `source=trusted-url`, या `source=artifact` से एक उम्मीदवार पैकेज निर्धारित करता है, उसे `package-under-test` के रूप में अपलोड करता है और फिर चयनित रेफ़ को दोबारा पैक करने के बजाय ठीक उसी टारबॉल पर पुनः उपयोग योग्य Docker E2E लेन चलाता है। प्रोफ़ाइल विस्तार के क्रम में हैं: `smoke`, `package`, `product`, और `full` (साथ ही स्पष्ट लेन सूची के लिए `custom`)। पैकेज/अपडेट/Plugin अनुबंध, प्रकाशित-अपग्रेड उत्तरजीवी मैट्रिक्स, रिलीज़ डिफ़ॉल्ट और विफलता ट्रायेज के लिए [अपडेट और Plugin का परीक्षण](/hi/help/testing-updates-plugins) देखें।
- बिल्ड और रिलीज़ जाँच tsdown के बाद `scripts/check-cli-bootstrap-imports.mjs` चलाती हैं। गार्ड `dist/entry.js` और `dist/cli/run-main.js` से स्थिर बिल्ट ग्राफ़ की जाँच करता है और यदि कमांड डिस्पैच से पहले वह प्री-डिस्पैच बूटस्ट्रैप ग्राफ़ किसी बाहरी पैकेज को स्थिर रूप से आयात करता है, तो विफल हो जाता है (Commander, प्रॉम्प्ट UI, undici, लॉगिंग और ऐसी ही स्टार्टअप-भारी निर्भरताएँ सभी गिनी जाती हैं); यह बंडल किए गए Gateway रन चंक को 70 KB तक सीमित करता है और उस चंक से ज्ञात कोल्ड Gateway पथों (`control-ui-assets`, `diagnostic-stability-bundle`, `onboard-helpers`, `process-respawn`, `restart-sentinel`, `server-close`, `server-reload-handlers`) के स्थिर आयात अस्वीकार करता है। `scripts/release-check.ts` अलग से पैक की गई CLI का `--help`, `onboard --help`, `doctor --help`, `status --json --timeout 1`, `config schema`, और `models list --provider openai` के साथ स्मोक-परीक्षण करता है।
- Package Acceptance की लेगेसी संगतता `2026.4.25` तक सीमित है (`2026.4.25-beta.*` शामिल)। इस सीमा तक, हार्नेस केवल जारी किए गए पैकेज की मेटाडेटा कमियों को सहन करता है: छोड़ी गई निजी QA इन्वेंट्री प्रविष्टियाँ, अनुपलब्ध `gateway install --wrapper`, टारबॉल-व्युत्पन्न git फ़िक्स्चर में अनुपलब्ध पैच फ़ाइलें, अनुपलब्ध स्थायी `update.channel`, लेगेसी Plugin इंस्टॉल-रिकॉर्ड स्थान, अनुपलब्ध मार्केटप्लेस इंस्टॉल-रिकॉर्ड स्थायित्व और `plugins update` के दौरान कॉन्फ़िगरेशन मेटाडेटा माइग्रेशन। `2026.4.25` के बाद के पैकेज के लिए ये पथ सख्त विफलताएँ हैं।
- कंटेनर स्मोक रनर: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:npm-onboard-channel-agent`, `test:docker:release-user-journey`, `test:docker:release-typed-onboarding`, `test:docker:release-media-memory`, `test:docker:release-upgrade-user-journey`, `test:docker:release-plugin-marketplace`, `test:docker:skill-install`, `test:docker:update-channel-switch`, `test:docker:upgrade-survivor`, `test:docker:published-upgrade-survivor`, `test:docker:session-runtime-context`, `test:docker:agents-delete-shared-workspace`, `test:docker:gateway-network`, `test:docker:browser-cdp-snapshot`, `test:docker:mcp-channels`, `test:docker:agent-bundle-mcp-tools`, `test:docker:cron-mcp-cleanup`, `test:docker:plugins`, `test:docker:plugin-update`, `test:docker:plugin-lifecycle-matrix`, और `test:docker:config-reload` एक या अधिक वास्तविक कंटेनर बूट करते हैं और उच्च-स्तरीय एकीकरण पथ सत्यापित करते हैं।
- `scripts/lib/openclaw-e2e-instance.sh` के माध्यम से पैक किया गया OpenClaw टारबॉल इंस्टॉल करने वाली Docker/Bash E2E लेन `npm install` को `OPENCLAW_E2E_NPM_INSTALL_TIMEOUT` तक सीमित करती हैं (डिफ़ॉल्ट `600s`; डीबगिंग के लिए रैपर अक्षम करने हेतु `0` सेट करें)।

लाइव-मॉडल Docker रनर भी केवल आवश्यक CLI प्रमाणीकरण होम
(या रन सीमित न होने पर सभी समर्थित होम) को बाइंड-माउंट करते हैं, फिर रन से पहले
उन्हें कंटेनर होम में कॉपी करते हैं, ताकि बाहरी-CLI OAuth होस्ट प्रमाणीकरण स्टोर
को बदले बिना टोकन रीफ़्रेश कर सके:

- प्रत्यक्ष मॉडल: `pnpm test:docker:live-models` (स्क्रिप्ट: `scripts/test-live-models-docker.sh`)
- ACP बाइंड स्मोक: `pnpm test:docker:live-acp-bind` (स्क्रिप्ट: `scripts/test-live-acp-bind-docker.sh`; डिफ़ॉल्ट रूप से Claude, Codex और Gemini को कवर करता है, तथा `pnpm test:docker:live-acp-bind:droid` और `pnpm test:docker:live-acp-bind:opencode` के माध्यम से Droid/OpenCode का सख्त कवरेज देता है)
- CLI बैकएंड स्मोक: `pnpm test:docker:live-cli-backend` (स्क्रिप्ट: `scripts/test-live-cli-backend-docker.sh`)
- Codex ऐप-सर्वर हार्नेस स्मोक: `pnpm test:docker:live-codex-harness` (स्क्रिप्ट: `scripts/test-live-codex-harness-docker.sh`)
- Gateway + डेवलपमेंट एजेंट: `pnpm test:docker:live-gateway` (स्क्रिप्ट: `scripts/test-live-gateway-models-docker.sh`)
- ऑब्ज़र्वेबिलिटी स्मोक: `pnpm qa:otel:smoke`, `pnpm qa:prometheus:smoke`, और `pnpm qa:observability:smoke` निजी QA स्रोत-चेकआउट लेन हैं। वे जानबूझकर पैकेज Docker रिलीज़ लेन का भाग नहीं हैं, क्योंकि npm टारबॉल में QA Lab शामिल नहीं है।
- Open WebUI लाइव स्मोक: `pnpm test:docker:openwebui` (स्क्रिप्ट: `scripts/e2e/openwebui-docker.sh`)
- ऑनबोर्डिंग विज़ार्ड (TTY, पूर्ण स्कैफ़ोल्डिंग): `pnpm test:docker:onboard` (स्क्रिप्ट: `scripts/e2e/onboard-docker.sh`)
- Npm टारबॉल ऑनबोर्डिंग/चैनल/एजेंट स्मोक: `pnpm test:docker:npm-onboard-channel-agent` पैक किए गए OpenClaw टारबॉल को Docker में वैश्विक रूप से इंस्टॉल करता है, env-ref ऑनबोर्डिंग के माध्यम से OpenAI तथा डिफ़ॉल्ट रूप से Telegram कॉन्फ़िगर करता है, doctor चलाता है और एक मॉक किया गया OpenAI एजेंट टर्न चलाता है। `OPENCLAW_CURRENT_PACKAGE_TGZ=/path/to/openclaw-*.tgz` के साथ पहले से निर्मित टारबॉल का पुनः उपयोग करें, `OPENCLAW_NPM_ONBOARD_HOST_BUILD=0` के साथ होस्ट रीबिल्ड छोड़ें, या `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` अथवा `OPENCLAW_NPM_ONBOARD_CHANNEL=slack` के साथ चैनल बदलें।

- रिलीज़ उपयोगकर्ता यात्रा स्मोक: `pnpm test:docker:release-user-journey` पैक किए गए OpenClaw टारबॉल को एक साफ़ Docker होम में वैश्विक रूप से इंस्टॉल करता है, ऑनबोर्डिंग चलाता है, मॉक किए गए OpenAI प्रदाता को कॉन्फ़िगर करता है, एजेंट टर्न चलाता है, बाहरी plugins इंस्टॉल/अनइंस्टॉल करता है, स्थानीय फ़िक्स्चर के लिए ClickClack कॉन्फ़िगर करता है, आउटबाउंड/इनबाउंड संदेश सेवा सत्यापित करता है, Gateway पुनः आरंभ करता है और डॉक्टर चलाता है।
- रिलीज़ टाइप्ड ऑनबोर्डिंग स्मोक: `pnpm test:docker:release-typed-onboarding` पैक किया गया टारबॉल इंस्टॉल करता है, वास्तविक TTY के माध्यम से `openclaw onboard` चलाता है, OpenAI को env-ref प्रदाता के रूप में कॉन्फ़िगर करता है, सत्यापित करता है कि रॉ कुंजी स्थायी रूप से संग्रहीत नहीं हुई है और मॉक किया गया एजेंट टर्न चलाता है।
- रिलीज़ मीडिया/मेमोरी स्मोक: `pnpm test:docker:release-media-memory` पैक किया गया टारबॉल इंस्टॉल करता है, PNG अटैचमेंट से छवि की समझ, OpenAI-संगत छवि निर्माण आउटपुट, मेमोरी खोज रिकॉल और Gateway पुनरारंभ के दौरान रिकॉल का बने रहना सत्यापित करता है।
- रिलीज़ अपग्रेड उपयोगकर्ता यात्रा स्मोक: `pnpm test:docker:release-upgrade-user-journey` डिफ़ॉल्ट रूप से उम्मीदवार टारबॉल से पुरानी नवीनतम प्रकाशित बेसलाइन इंस्टॉल करता है, प्रकाशित पैकेज पर प्रदाता/plugin/ClickClack स्थिति कॉन्फ़िगर करता है, उम्मीदवार टारबॉल में अपग्रेड करता है और फिर मुख्य एजेंट/plugin/चैनल यात्रा दोबारा चलाता है। यदि कोई पुरानी प्रकाशित बेसलाइन उपलब्ध नहीं है, तो यह उम्मीदवार संस्करण का पुनः उपयोग करता है। `OPENCLAW_RELEASE_UPGRADE_BASELINE_SPEC=openclaw@<version>` से बेसलाइन ओवरराइड करें।
- रिलीज़ plugin मार्केटप्लेस स्मोक: `pnpm test:docker:release-plugin-marketplace` स्थानीय फ़िक्स्चर मार्केटप्लेस से इंस्टॉल करता है, इंस्टॉल किए गए plugin को अपडेट करता है, उसे अनइंस्टॉल करता है और सत्यापित करता है कि इंस्टॉल मेटाडेटा की छँटाई के साथ plugin CLI गायब हो जाता है।
- Skill इंस्टॉल स्मोक: `pnpm test:docker:skill-install` पैक किए गए OpenClaw टारबॉल को Docker में वैश्विक रूप से इंस्टॉल करता है, कॉन्फ़िगरेशन में अपलोड किए गए आर्काइव के इंस्टॉल अक्षम करता है, खोज से वर्तमान लाइव ClawHub Skill स्लग का समाधान करता है, उसे `openclaw skills install` से इंस्टॉल करता है और इंस्टॉल किए गए Skill के साथ `.clawhub` मूल/लॉक मेटाडेटा सत्यापित करता है।
- अपडेट चैनल स्विच स्मोक: `pnpm test:docker:update-channel-switch` पैक किए गए OpenClaw टारबॉल को Docker में वैश्विक रूप से इंस्टॉल करता है, पैकेज `stable` से git `dev` पर स्विच करता है, स्थायी चैनल और अपडेट के बाद plugin का कार्य सत्यापित करता है, फिर वापस पैकेज `stable` पर स्विच करके अपडेट स्थिति जाँचता है।
- अपग्रेड सर्वाइवर स्मोक: `pnpm test:docker:upgrade-survivor` पैक किए गए OpenClaw टारबॉल को एजेंट, चैनल कॉन्फ़िगरेशन, plugin अनुमतिसूचियों, पुराने plugin निर्भरता स्थिति और मौजूदा कार्यस्थान/सत्र फ़ाइलों वाले अव्यवस्थित पुराने-उपयोगकर्ता फ़िक्स्चर पर इंस्टॉल करता है। यह लाइव प्रदाता या चैनल कुंजियों के बिना पैकेज अपडेट और गैर-इंटरैक्टिव डॉक्टर चलाता है, फिर लूपबैक Gateway आरंभ करता है और कॉन्फ़िगरेशन/स्थिति संरक्षण के साथ स्टार्टअप/स्थिति बजट जाँचता है।
- प्रकाशित अपग्रेड सर्वाइवर स्मोक: `pnpm test:docker:published-upgrade-survivor` डिफ़ॉल्ट रूप से `openclaw@latest` इंस्टॉल करता है, वास्तविक मौजूदा-उपयोगकर्ता फ़ाइलें सीड करता है, अंतर्निहित कमांड रेसिपी से उस बेसलाइन को कॉन्फ़िगर करता है, परिणामी कॉन्फ़िगरेशन सत्यापित करता है, उस प्रकाशित इंस्टॉल को उम्मीदवार टारबॉल में अपडेट करता है, गैर-इंटरैक्टिव डॉक्टर चलाता है, `.artifacts/upgrade-survivor/summary.json` लिखता है, फिर लूपबैक Gateway आरंभ करके कॉन्फ़िगर किए गए आशय, स्थिति संरक्षण, स्टार्टअप, `/healthz`, `/readyz` और RPC स्थिति बजट जाँचता है। एक बेसलाइन को `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` से ओवरराइड करें, समेकित शेड्यूलर से `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` के माध्यम से `openclaw@2026.5.2 openclaw@2026.4.23 openclaw@2026.4.15` जैसी सटीक स्थानीय बेसलाइन विस्तृत करने को कहें और `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS` के माध्यम से `reported-issues` जैसे समस्या-आकार वाले फ़िक्स्चर विस्तृत करें; रिपोर्ट की गई समस्याओं के सेट में स्वचालित बाहरी OpenClaw plugin इंस्टॉल सुधार के लिए `configured-plugin-installs` शामिल है। पैकेज स्वीकृति इन्हें `published_upgrade_survivor_baseline`, `published_upgrade_survivor_baselines` और `published_upgrade_survivor_scenarios` के रूप में उपलब्ध कराती है, `last-stable-4` या `all-since-2026.4.23` जैसे मेटा बेसलाइन टोकन का समाधान करती है और पूर्ण रिलीज़ सत्यापन रिलीज़-सोक पैकेज गेट को `last-stable-4 2026.4.23 2026.5.2 2026.4.15` तथा `reported-issues` तक विस्तृत करता है।
- सत्र रनटाइम संदर्भ स्मोक: `pnpm test:docker:session-runtime-context` छिपे हुए रनटाइम संदर्भ ट्रांसक्रिप्ट की स्थायित्व और प्रभावित डुप्लिकेट प्रॉम्प्ट-पुनर्लेखन शाखाओं के डॉक्टर सुधार को सत्यापित करता है।
- Bun वैश्विक इंस्टॉल स्मोक: `bash scripts/e2e/bun-global-install-smoke.sh` वर्तमान ट्री को पैक करता है, उसे पृथक होम में `bun install -g` से इंस्टॉल करता है और सत्यापित करता है कि `openclaw infer image providers --json` अटकने के बजाय बंडल किए गए छवि प्रदाता लौटाता है। पहले से निर्मित टारबॉल का `OPENCLAW_BUN_GLOBAL_SMOKE_PACKAGE_TGZ=/path/to/openclaw-*.tgz` से पुनः उपयोग करें, होस्ट बिल्ड को `OPENCLAW_BUN_GLOBAL_SMOKE_HOST_BUILD=0` से छोड़ें या निर्मित Docker छवि से `dist/` को `OPENCLAW_BUN_GLOBAL_SMOKE_DIST_IMAGE=openclaw-dockerfile-smoke:local` द्वारा कॉपी करें।
- इंस्टॉलर Docker स्मोक: `bash scripts/test-install-sh-docker.sh` अपने रूट, अपडेट और direct-npm कंटेनरों के बीच एक npm कैश साझा करता है। उम्मीदवार टारबॉल में अपग्रेड करने से पहले अपडेट स्मोक स्थिर बेसलाइन के रूप में डिफ़ॉल्टतः npm `latest` का उपयोग करता है। स्थानीय रूप से `OPENCLAW_INSTALL_SMOKE_UPDATE_BASELINE=2026.4.22` से या GitHub पर इंस्टॉल स्मोक वर्कफ़्लो के `update_baseline_version` इनपुट से ओवरराइड करें। गैर-रूट इंस्टॉलर जाँच पृथक npm कैश बनाए रखती हैं ताकि रूट-स्वामित्व वाली कैश प्रविष्टियाँ उपयोगकर्ता-स्थानीय इंस्टॉल व्यवहार को न छिपाएँ। स्थानीय पुनः चलावों के बीच रूट/अपडेट/direct-npm कैश का पुनः उपयोग करने के लिए `OPENCLAW_INSTALL_SMOKE_NPM_CACHE_DIR=/path/to/cache` सेट करें।
- इंस्टॉल स्मोक CI, `OPENCLAW_INSTALL_SMOKE_SKIP_NPM_GLOBAL=1` के साथ डुप्लिकेट direct-npm वैश्विक अपडेट छोड़ देता है; प्रत्यक्ष `npm install -g` कवरेज की आवश्यकता होने पर उस env के बिना स्क्रिप्ट स्थानीय रूप से चलाएँ।
- एजेंट साझा कार्यस्थान हटाने का CLI स्मोक: `pnpm test:docker:agents-delete-shared-workspace` (स्क्रिप्ट: `scripts/e2e/agents-delete-shared-workspace-docker.sh`) डिफ़ॉल्ट रूप से रूट Dockerfile छवि बनाता है, पृथक कंटेनर होम में एक कार्यस्थान वाले दो एजेंट सीड करता है, `agents delete --json` चलाता है और वैध JSON के साथ बनाए रखे गए कार्यस्थान व्यवहार को सत्यापित करता है। इंस्टॉल-स्मोक छवि का `OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_IMAGE=openclaw-dockerfile-smoke:local OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_SKIP_BUILD=1` से पुनः उपयोग करें।
- Gateway नेटवर्किंग और होस्ट जीवनचक्र: `pnpm test:docker:gateway-network` (स्क्रिप्ट: `scripts/e2e/gateway-network-docker.sh`) दो-कंटेनर LAN WebSocket प्रमाणीकरण/स्वास्थ्य स्मोक को बनाए रखता है, फिर तैयारी फेंसिंग, बनाए रखी गई नियंत्रण पहुँच, फिर से शुरू करने पर पुनर्प्राप्ति और उसी कंटेनर में तैयार स्टॉप/स्टार्ट सिद्ध करने के लिए लूपबैक एडमिन HTTP का उपयोग करता है। पुनरारंभ जाँच मूल लीज़ समाप्त होने से पहले पूरी होनी चाहिए, यह सत्यापित करती है कि निलंबन स्थिति प्रक्रिया-स्थानीय है जबकि स्थायी Gateway कॉन्फ़िगरेशन और कंटेनर पहचान बनी रहती है तथा मशीन-पठनीय चरण समय JSON उत्सर्जित करती है।
- ब्राउज़र CDP स्नैपशॉट स्मोक: `pnpm test:docker:browser-cdp-snapshot` (स्क्रिप्ट: `scripts/e2e/browser-cdp-snapshot-docker.sh`) स्रोत E2E छवि के साथ Chromium परत बनाता है, रॉ CDP के साथ Chromium आरंभ करता है, `browser doctor --deep` चलाता है और सत्यापित करता है कि CDP भूमिका स्नैपशॉट लिंक URL, कर्सर-प्रोन्नत क्लिक योग्य तत्व, iframe संदर्भ और फ़्रेम मेटाडेटा को कवर करते हैं।
- OpenAI Responses web_search न्यूनतम रीजनिंग रिग्रेशन: `pnpm test:docker:openai-web-search-minimal` (स्क्रिप्ट: `scripts/e2e/openai-web-search-minimal-docker.sh`) Gateway के माध्यम से मॉक किया गया OpenAI सर्वर चलाता है, सत्यापित करता है कि `web_search`, `reasoning.effort` को `minimal` से `low` तक बढ़ाता है, फिर प्रदाता स्कीमा अस्वीकृति बाध्य करता है और जाँचता है कि रॉ विवरण Gateway लॉग में दिखाई देता है।
- MCP चैनल ब्रिज (सीड किया गया Gateway + stdio ब्रिज + रॉ Claude सूचना-फ़्रेम स्मोक): `pnpm test:docker:mcp-channels` (स्क्रिप्ट: `scripts/e2e/mcp-channels-docker.sh`)
- OpenClaw बंडल MCP टूल (वास्तविक stdio MCP सर्वर + अंतःस्थापित OpenClaw प्रोफ़ाइल अनुमति/अस्वीकृति स्मोक): `pnpm test:docker:agent-bundle-mcp-tools` (स्क्रिप्ट: `scripts/e2e/agent-bundle-mcp-tools-docker.sh`)
- Cron/उप-एजेंट MCP क्लीनअप (वास्तविक Gateway + पृथक Cron और एकबारगी उप-एजेंट चलावों के बाद stdio MCP चाइल्ड टियरडाउन): `pnpm test:docker:cron-mcp-cleanup` (स्क्रिप्ट: `scripts/e2e/cron-mcp-cleanup-docker.sh`)
- Plugins (स्थानीय पथ, `file:`, होइस्ट की गई निर्भरताओं वाली npm रजिस्ट्री, विकृत npm पैकेज मेटाडेटा, git गतिशील संदर्भ, ClawHub किचन-सिंक, मार्केटप्लेस अपडेट और Claude-बंडल सक्षम/निरीक्षण के लिए इंस्टॉल/अपडेट स्मोक): `pnpm test:docker:plugins` (स्क्रिप्ट: `scripts/e2e/plugins-docker.sh`)
  ClawHub ब्लॉक छोड़ने के लिए `OPENCLAW_PLUGINS_E2E_CLAWHUB=0` सेट करें या डिफ़ॉल्ट किचन-सिंक पैकेज/रनटाइम युग्म को `OPENCLAW_PLUGINS_E2E_CLAWHUB_SPEC` और `OPENCLAW_PLUGINS_E2E_CLAWHUB_ID` से ओवरराइड करें। `OPENCLAW_CLAWHUB_URL`/`CLAWHUB_URL` के बिना परीक्षण पूर्णतः पृथक स्थानीय ClawHub फ़िक्स्चर सर्वर का उपयोग करता है।
- Plugin अपरिवर्तित अपडेट स्मोक: `pnpm test:docker:plugin-update` (स्क्रिप्ट: `scripts/e2e/plugin-update-unchanged-docker.sh`)
- Plugin जीवनचक्र मैट्रिक्स स्मोक: `pnpm test:docker:plugin-lifecycle-matrix` पैक किए गए OpenClaw टारबॉल को रिक्त कंटेनर में इंस्टॉल करता है, npm plugin इंस्टॉल करता है, सक्षम/अक्षम टॉगल करता है, स्थानीय npm रजिस्ट्री के माध्यम से उसे अपग्रेड और डाउनग्रेड करता है, इंस्टॉल किया गया कोड हटाता है, फिर सत्यापित करता है कि अनइंस्टॉल अब भी पुरानी स्थिति हटाता है और प्रत्येक जीवनचक्र चरण के लिए RSS/CPU मेट्रिक्स लॉग करता है।
- कॉन्फ़िगरेशन रीलोड मेटाडेटा स्मोक: `pnpm test:docker:config-reload` (स्क्रिप्ट: `scripts/e2e/config-reload-source-docker.sh`)
- Plugins: `pnpm test:docker:plugins` स्थानीय पथ, `file:`, होइस्ट की गई निर्भरताओं वाली npm रजिस्ट्री, git गतिशील संदर्भ, ClawHub फ़िक्स्चर, मार्केटप्लेस अपडेट और Claude-बंडल सक्षम/निरीक्षण के लिए इंस्टॉल/अपडेट स्मोक कवर करता है। `pnpm test:docker:plugin-update` इंस्टॉल किए गए plugins के अपरिवर्तित अपडेट व्यवहार को कवर करता है। `pnpm test:docker:plugin-lifecycle-matrix` संसाधन-ट्रैक किए गए npm plugin के इंस्टॉल, सक्षम करने, अक्षम करने, अपग्रेड, डाउनग्रेड और कोड अनुपलब्ध होने पर अनइंस्टॉल को कवर करता है।

साझा कार्यात्मक छवि को मैन्युअल रूप से पहले से बनाने और पुनः उपयोग करने के लिए:

```bash
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local pnpm test:docker:e2e-build
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local OPENCLAW_SKIP_DOCKER_BUILD=1 pnpm test:docker:mcp-channels
```

`OPENCLAW_GATEWAY_NETWORK_E2E_IMAGE` जैसे सुइट-विशिष्ट छवि ओवरराइड सेट होने पर अब भी प्राथमिकता लेते हैं। जब `OPENCLAW_SKIP_DOCKER_BUILD=1` किसी दूरस्थ साझा छवि की ओर संकेत करता है, तो स्क्रिप्ट उसे स्थानीय रूप से पहले से उपलब्ध न होने पर पुल करती हैं। QR और इंस्टॉलर Docker परीक्षण अपने Dockerfiles बनाए रखते हैं क्योंकि वे साझा निर्मित-ऐप रनटाइम के बजाय पैकेज/इंस्टॉल व्यवहार सत्यापित करते हैं।

लाइव-मॉडल Docker रनर वर्तमान चेकआउट को केवल-पढ़ने योग्य रूप में बाइंड-माउंट भी करते हैं
और उसे कंटेनर के भीतर अस्थायी कार्यशील निर्देशिका में स्टेज करते हैं। इससे रनटाइम
छवि हल्की रहती है, फिर भी Vitest आपके बिल्कुल सटीक स्थानीय स्रोत/कॉन्फ़िगरेशन के विरुद्ध
चलता है। स्टेजिंग चरण `.pnpm-store`, `.worktrees`, `__openclaw_vitest__` और
ऐप-स्थानीय `.build` या Gradle आउटपुट निर्देशिकाओं जैसे बड़े केवल-स्थानीय कैश और ऐप बिल्ड
आउटपुट छोड़ देता है, ताकि Docker लाइव चलाव मशीन-विशिष्ट आर्टिफ़ैक्ट कॉपी करने में
कई मिनट खर्च न करें। वे `OPENCLAW_SKIP_CHANNELS=1` भी सेट करते हैं ताकि Gateway लाइव
प्रोब कंटेनर के भीतर वास्तविक Telegram/Discord/आदि चैनल वर्कर आरंभ न करें।
`test:docker:live-models` अब भी `pnpm test:live` चलाता है, इसलिए जब आपको उस Docker लेन से
Gateway लाइव कवरेज को सीमित या बाहर करना हो, तब `OPENCLAW_LIVE_GATEWAY_*` भी पास करें।

`test:docker:openwebui` एक उच्च-स्तरीय संगतता स्मोक है: यह OpenAI-संगत HTTP एंडपॉइंट
सक्षम करके OpenClaw Gateway कंटेनर आरंभ करता है, उस Gateway के विरुद्ध पिन किया गया
Open WebUI कंटेनर आरंभ करता है, Open WebUI के माध्यम से साइन इन करता है,
सत्यापित करता है कि `/api/models`, `openclaw/default` को उजागर करता है, फिर Open WebUI के
`/api/chat/completions` प्रॉक्सी के माध्यम से वास्तविक चैट अनुरोध भेजता है। उन रिलीज़-पथ
CI जाँचों के लिए `OPENWEBUI_SMOKE_MODE=models` सेट करें जिन्हें लाइव मॉडल
पूर्णता की प्रतीक्षा किए बिना Open WebUI साइन-इन और मॉडल खोज के बाद रुकना चाहिए।
पहला चलाव स्पष्ट रूप से धीमा हो सकता है क्योंकि Docker को Open WebUI छवि
पुल करनी पड़ सकती है और Open WebUI को अपना कोल्ड-स्टार्ट सेटअप पूरा करना पड़ सकता है।
यह लेन उपयोग योग्य लाइव मॉडल कुंजी की अपेक्षा करती है, जो प्रक्रिया परिवेश,
स्टेज की गई प्रमाणीकरण प्रोफ़ाइल या स्पष्ट `OPENCLAW_PROFILE_FILE` के माध्यम से प्रदान की जाती है।
सफल चलाव `{ "ok": true, "model": "openclaw/default", ... }` जैसा छोटा JSON पेलोड प्रिंट करते हैं।

`test:docker:mcp-channels` जानबूझकर निर्धारक है और इसके लिए वास्तविक Telegram, Discord या
iMessage खाते की आवश्यकता नहीं है। यह सीड किया गया Gateway कंटेनर बूट करता है,
एक दूसरा कंटेनर आरंभ करता है जो `openclaw mcp serve` स्पॉन करता है, फिर वास्तविक stdio
MCP ब्रिज पर रूट की गई वार्तालाप खोज, ट्रांसक्रिप्ट पठन, अटैचमेंट मेटाडेटा,
लाइव इवेंट कतार व्यवहार, आउटबाउंड प्रेषण रूटिंग और Claude-शैली चैनल तथा
अनुमति सूचनाएँ सत्यापित करता है। सूचना जाँच रॉ stdio MCP फ़्रेम का सीधे निरीक्षण करती है,
ताकि स्मोक यह सत्यापित करे कि ब्रिज वास्तव में क्या उत्सर्जित करता है, न कि केवल वह जो
किसी विशिष्ट क्लाइंट SDK में संयोग से दिखाई देता है।

`test:docker:agent-bundle-mcp-tools` नियतात्मक है और इसे लाइव मॉडल कुंजी की
आवश्यकता नहीं है। यह रेपो Docker इमेज बनाता है, कंटेनर के भीतर एक वास्तविक stdio MCP
जाँच सर्वर शुरू करता है, एम्बेड किए गए OpenClaw बंडल MCP रनटाइम के माध्यम से उस सर्वर को
साकार करता है, टूल निष्पादित करता है, फिर सत्यापित करता है कि
`coding` और `messaging`, `bundle-mcp` टूल बनाए रखते हैं, जबकि `minimal` और
`tools.deny: ["bundle-mcp"]` उन्हें फ़िल्टर करते हैं।

`test:docker:cron-mcp-cleanup` नियतात्मक है और इसे लाइव
मॉडल कुंजी की आवश्यकता नहीं है। यह एक वास्तविक stdio MCP जाँच सर्वर के साथ पहले से डेटा-युक्त Gateway शुरू करता है,
एक पृथक cron टर्न और एक `sessions_spawn` एकल-निष्पादन चाइल्ड टर्न चलाता है, फिर
सत्यापित करता है कि MCP चाइल्ड प्रक्रिया प्रत्येक रन के बाद समाप्त हो जाती है।

मैन्युअल ACP सरल-भाषा थ्रेड स्मोक परीक्षण (CI नहीं):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- इस स्क्रिप्ट को रिग्रेशन/डीबग कार्यप्रवाहों के लिए रखें। ACP थ्रेड रूटिंग सत्यापन के लिए इसकी फिर आवश्यकता पड़ सकती है, इसलिए इसे हटाएँ नहीं।

उपयोगी एनवायरनमेंट वेरिएबल:

- `OPENCLAW_CONFIG_DIR=...` (डिफ़ॉल्ट: `~/.openclaw`) को `/home/node/.openclaw` पर माउंट किया जाता है
- `OPENCLAW_WORKSPACE_DIR=...` (डिफ़ॉल्ट: `~/.openclaw/workspace`) को `/home/node/.openclaw/workspace` पर माउंट किया जाता है
- `OPENCLAW_PROFILE_FILE=...` को माउंट किया जाता है और परीक्षण चलाने से पहले सोर्स किया जाता है
- `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1` यह सत्यापित करने के लिए कि केवल `OPENCLAW_PROFILE_FILE` से सोर्स किए गए एनवायरनमेंट वेरिएबल ही उपयोग हों; इसमें अस्थायी कॉन्फ़िगरेशन/वर्कस्पेस डायरेक्टरियाँ और कोई बाहरी CLI प्रमाणीकरण माउंट नहीं होता
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (डिफ़ॉल्ट: `~/.cache/openclaw/docker-cli-tools`, जब तक रन पहले से किसी CI/प्रबंधित बाइंड डायरेक्टरी का उपयोग न करता हो) को Docker के भीतर कैश किए गए CLI इंस्टॉलेशन के लिए `/home/node/.npm-global` पर माउंट किया जाता है
- `$HOME` के अंतर्गत बाहरी CLI प्रमाणीकरण डायरेक्टरियाँ/फ़ाइलें `/host-auth...` के अंतर्गत केवल-पढ़ने योग्य रूप में माउंट की जाती हैं, फिर परीक्षण शुरू होने से पहले `/home/node/...` में कॉपी की जाती हैं
  - डिफ़ॉल्ट डायरेक्टरियाँ (जब रन विशिष्ट प्रदाताओं तक सीमित नहीं होता): `.factory`, `.gemini`, `.minimax`
  - डिफ़ॉल्ट फ़ाइलें: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - सीमित प्रदाता रन केवल `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` से अनुमानित आवश्यक डायरेक्टरियाँ/फ़ाइलें माउंट करते हैं
  - `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none`, या `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` जैसी अल्पविराम-सूची से मैन्युअल रूप से ओवरराइड करें
- रन को सीमित करने के लिए `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`
- कंटेनर के भीतर प्रदाताओं को फ़िल्टर करने के लिए `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`
- ऐसे पुनः-रनों के लिए मौजूदा `openclaw:local-live` इमेज का पुनः उपयोग करने हेतु `OPENCLAW_SKIP_DOCKER_BUILD=1`, जिन्हें पुनर्निर्माण की आवश्यकता नहीं है
- यह सुनिश्चित करने के लिए `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` कि क्रेडेंशियल प्रोफ़ाइल स्टोर से आएँ (एनवायरनमेंट से नहीं)
- Open WebUI स्मोक परीक्षण के लिए Gateway द्वारा प्रदर्शित मॉडल चुनने हेतु `OPENCLAW_OPENWEBUI_MODEL=...`
- Open WebUI स्मोक परीक्षण द्वारा उपयोग किए जाने वाले नॉन्स-जाँच प्रॉम्प्ट को ओवरराइड करने हेतु `OPENCLAW_OPENWEBUI_PROMPT=...`
- पिन किए गए Open WebUI इमेज टैग को ओवरराइड करने हेतु `OPENWEBUI_IMAGE=...`

## दस्तावेज़ों की शुद्धता-जाँच

दस्तावेज़ संपादित करने के बाद दस्तावेज़ जाँच चलाएँ: `pnpm check:docs`।
जब पृष्ठ के भीतर शीर्षक जाँच भी आवश्यक हो, तब पूर्ण Mintlify एंकर सत्यापन चलाएँ: `pnpm docs:check-links:anchors`।

## ऑफ़लाइन रिग्रेशन (CI-सुरक्षित)

ये वास्तविक प्रदाताओं के बिना "वास्तविक पाइपलाइन" रिग्रेशन हैं:

- Gateway टूल कॉलिंग (मॉक OpenAI, वास्तविक Gateway + एजेंट लूप): `src/gateway/gateway.test.ts` (केस: "Gateway एजेंट लूप के माध्यम से मॉक OpenAI टूल कॉल को शुरू से अंत तक चलाता है")
- Gateway विज़ार्ड (WS `wizard.start`/`wizard.next`, कॉन्फ़िगरेशन लिखता है + प्रमाणीकरण लागू करता है): `src/gateway/gateway.test.ts` (केस: "ws पर विज़ार्ड चलाता है और प्रमाणीकरण टोकन कॉन्फ़िगरेशन लिखता है")

## एजेंट विश्वसनीयता मूल्यांकन (Skills)

हमारे पास पहले से कुछ CI-सुरक्षित परीक्षण हैं जो "एजेंट विश्वसनीयता मूल्यांकन" की तरह व्यवहार करते हैं:

- वास्तविक Gateway + एजेंट लूप के माध्यम से मॉक टूल-कॉलिंग (`src/gateway/gateway.test.ts`)।
- सत्र वायरिंग और कॉन्फ़िगरेशन प्रभावों को सत्यापित करने वाले शुरू से अंत तक विज़ार्ड प्रवाह (`src/gateway/gateway.test.ts`)।

Skills के लिए अभी भी क्या अनुपलब्ध है ([Skills](/hi/tools/skills) देखें):

- **निर्णय-निर्धारण:** जब प्रॉम्प्ट में Skills सूचीबद्ध हों, तो क्या एजेंट सही Skill चुनता है (या अप्रासंगिक Skills से बचता है)?
- **अनुपालन:** क्या एजेंट उपयोग से पहले `SKILL.md` पढ़ता है और आवश्यक चरणों/आर्ग्युमेंट का पालन करता है?
- **कार्यप्रवाह अनुबंध:** बहु-टर्न परिदृश्य जो टूल क्रम, सत्र इतिहास हस्तांतरण और सैंडबॉक्स सीमाओं की पुष्टि करते हैं।

भविष्य के मूल्यांकन पहले नियतात्मक ही रहने चाहिए:

- टूल कॉल और उनके क्रम, Skill फ़ाइल पठन तथा सत्र वायरिंग की पुष्टि करने के लिए मॉक प्रदाताओं का उपयोग करने वाला परिदृश्य रनर।
- Skill-केंद्रित परिदृश्यों का एक छोटा सुइट (उपयोग बनाम परहेज़, गेटिंग, प्रॉम्प्ट इंजेक्शन)।
- CI-सुरक्षित सुइट स्थापित होने के बाद ही वैकल्पिक लाइव मूल्यांकन (ऑप्ट-इन, एनवायरनमेंट द्वारा नियंत्रित)।

## अनुबंध परीक्षण (Plugin और चैनल संरचना)

अनुबंध परीक्षण सत्यापित करते हैं कि प्रत्येक पंजीकृत Plugin और चैनल
अपने इंटरफ़ेस अनुबंध के अनुरूप है। वे खोजे गए सभी Plugin पर पुनरावृत्ति करते हैं और
संरचना तथा व्यवहार संबंधी अभिकथनों का एक सुइट चलाते हैं। डिफ़ॉल्ट `pnpm test` यूनिट लेन
जानबूझकर इन साझा सीमांत और स्मोक फ़ाइलों को छोड़ देती है; साझा चैनल
या प्रदाता सतहों में बदलाव करते समय अनुबंध कमांड स्पष्ट रूप से चलाएँ।

### कमांड

- सभी अनुबंध: `pnpm test:contracts`
- केवल चैनल अनुबंध: `pnpm test:contracts:channels`
- केवल प्रदाता अनुबंध: `pnpm test:contracts:plugins`

### चैनल अनुबंध

`src/channels/plugins/contracts/*.contract.test.ts` में स्थित। वर्तमान
शीर्ष-स्तरीय श्रेणियाँ:

- **channel-catalog** - बंडल/रजिस्ट्री चैनल कैटलॉग प्रविष्टि मेटाडेटा
- **plugin** (रजिस्ट्री-समर्थित, शार्डेड) - मूल Plugin पंजीकरण संरचना
- **surfaces-only** (रजिस्ट्री-समर्थित, शार्डेड) - `actions`, `setup`, `status`, `outbound`, `messaging`, `threading`, `directory`, और `gateway` के लिए प्रति-सतह संरचना जाँच
- **session-binding** (रजिस्ट्री-समर्थित) - सत्र बाइंडिंग व्यवहार
- **outbound-payload** - संदेश पेलोड संरचना और सामान्यीकरण
- **group-policy** (फ़ॉलबैक) - प्रति चैनल डिफ़ॉल्ट समूह नीति प्रवर्तन
- **threading** (रजिस्ट्री-समर्थित, शार्डेड) - थ्रेड आईडी प्रबंधन
- **directory** (रजिस्ट्री-समर्थित, शार्डेड) - डायरेक्टरी/रोस्टर API
- **registry** और **plugins-core.\*** - चैनल Plugin रजिस्ट्री, लोडर और कॉन्फ़िगरेशन-लेखन प्राधिकरण आंतरिक तंत्र

इन सुइट द्वारा उपयोग किए जाने वाले इनबाउंड डिस्पैच-कैप्चर और आउटबाउंड-पेलोड हार्नेस सहायक
`src/plugin-sdk/channel-contract-testing.ts` के माध्यम से आंतरिक रूप से उपलब्ध कराए जाते हैं
(npm से बहिष्कृत, सार्वजनिक SDK सबपाथ नहीं); इस डायरेक्टरी में कोई स्वतंत्र
`inbound.contract.test.ts` फ़ाइल नहीं है।

### प्रदाता अनुबंध

`src/plugins/contracts/*.contract.test.ts` में स्थित। वर्तमान श्रेणियों
में शामिल हैं:

- **shape** - Plugin मैनिफ़ेस्ट, API और रनटाइम निर्यात संरचना
- **plugin-registration** (+ समानांतर) - मैनिफ़ेस्ट पंजीकरण केस
- **package-manifest** - पैकेज मैनिफ़ेस्ट आवश्यकताएँ
- **loader** - Plugin लोडर सेटअप/टियरडाउन व्यवहार
- **registry** - Plugin अनुबंध रजिस्ट्री सामग्री और लुकअप
- **providers** - बंडल किए गए प्रदाताओं में साझा प्रदाता व्यवहार, साथ ही वेब-खोज प्रदाता
- **auth-choice** - प्रमाणीकरण विकल्प मेटाडेटा और सेटअप व्यवहार
- **provider-catalog-deprecation** - अप्रचलित प्रदाता कैटलॉग मेटाडेटा
- **wizard.choice-resolution**, **wizard.model-picker**, **wizard.setup-options** - प्रदाता सेटअप विज़ार्ड अनुबंध
- **embedding-provider**, **memory-embedding-provider**, **web-fetch-provider**, **tts** - क्षमता-विशिष्ट प्रदाता अनुबंध
- **session-actions**, **session-attachments**, **session-entry-projection** - Plugin-स्वामित्व वाले सत्र स्थिति अनुबंध
- **scheduled-turns** - Plugin निर्धारित टर्न मेटाडेटा और टाइमस्टैम्प सीमाएँ
- **host-hooks**, **run-context-lifecycle**, **runtime-import-side-effects**, **runtime-seams** - Plugin होस्ट/रनटाइम जीवनचक्र और आयात-सीमा अनुबंध
- **extension-runtime-dependencies** - एक्सटेंशन के लिए रनटाइम निर्भरता स्थान

### कब चलाएँ

- plugin-sdk निर्यात या सबपाथ बदलने के बाद
- चैनल या प्रदाता Plugin जोड़ने अथवा संशोधित करने के बाद
- Plugin पंजीकरण या खोज को रीफ़ैक्टर करने के बाद

अनुबंध परीक्षण CI में चलते हैं और उन्हें वास्तविक API कुंजियों की आवश्यकता नहीं होती।

## रिग्रेशन जोड़ना (मार्गदर्शन)

जब आप लाइव में मिली किसी प्रदाता/मॉडल समस्या को ठीक करें:

- यदि संभव हो तो CI-सुरक्षित रिग्रेशन जोड़ें (मॉक/स्टब प्रदाता, या सटीक अनुरोध-संरचना रूपांतरण कैप्चर करें)
- यदि वह स्वाभाविक रूप से केवल लाइव है (दर सीमाएँ, प्रमाणीकरण नीतियाँ), तो लाइव परीक्षण को सीमित रखें और एनवायरनमेंट वेरिएबल के माध्यम से ऑप्ट-इन बनाएँ
- बग पकड़ने वाली सबसे छोटी परत को लक्षित करना बेहतर है:
  - प्रदाता अनुरोध रूपांतरण/रीप्ले बग -> प्रत्यक्ष मॉडल परीक्षण
  - Gateway सत्र/इतिहास/टूल पाइपलाइन बग -> Gateway लाइव स्मोक या CI-सुरक्षित Gateway मॉक परीक्षण
- SecretRef ट्रैवर्सल सुरक्षा-सीमा:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` रजिस्ट्री मेटाडेटा (`listSecretTargetRegistryEntries()`) से प्रति SecretRef वर्ग एक नमूना लक्ष्य प्राप्त करता है, फिर पुष्टि करता है कि ट्रैवर्सल-सेगमेंट exec आईडी अस्वीकार कर दी जाती हैं।
  - यदि आप `src/secrets/target-registry-data.ts` में कोई नया `includeInPlan` SecretRef लक्ष्य परिवार जोड़ते हैं, तो उस परीक्षण में `classifyTargetClass` अपडेट करें। परीक्षण अवर्गीकृत लक्ष्य आईडी पर जानबूझकर विफल होता है, ताकि नए वर्गों को चुपचाप छोड़ा न जा सके।

## संबंधित

- [लाइव परीक्षण](/hi/help/testing-live)
- [अपडेट और Plugin का परीक्षण](/hi/help/testing-updates-plugins)
- [CI](/hi/ci)
