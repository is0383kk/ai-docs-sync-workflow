---
read_when:
    - स्थानीय नियंत्रण API के माध्यम से एजेंट ब्राउज़र की स्क्रिप्टिंग या डीबगिंग
    - '`openclaw browser` CLI संदर्भ खोज रहे हैं'
    - स्नैपशॉट और रेफ़ के साथ कस्टम ब्राउज़र ऑटोमेशन जोड़ना
summary: OpenClaw ब्राउज़र नियंत्रण API, CLI संदर्भ और स्क्रिप्टिंग क्रियाएँ
title: ब्राउज़र नियंत्रण API
x-i18n:
    generated_at: "2026-07-27T20:34:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 812358a5ad366e419413b78507d3620ea9f3981224bc8cc62fb512b87eaadd9b
    source_path: tools/browser-control.md
    workflow: 16
---

सेटअप, कॉन्फ़िगरेशन और समस्या निवारण के लिए, [ब्राउज़र](/hi/tools/browser) देखें।
यह पृष्ठ स्थानीय नियंत्रण HTTP API, `openclaw browser`
CLI और स्क्रिप्टिंग पैटर्न (स्नैपशॉट, रेफ़, प्रतीक्षा, डीबग प्रवाह) का संदर्भ है।

## नियंत्रण API (वैकल्पिक)

केवल स्थानीय एकीकरणों के लिए, Gateway एक छोटा लूपबैक HTTP API उपलब्ध कराता है।
यह स्वतंत्र सर्वर वैकल्पिक है — gateway सेवा के परिवेश में पर्यावरण चर
`OPENCLAW_EAGER_BROWSER_CONTROL_SERVER=1` सेट करें
और HTTP एंडपॉइंट उपलब्ध होने से पहले gateway पुनः आरंभ करें। इस चर के बिना
ब्राउज़र नियंत्रण रनटाइम CLI और
एजेंट टूल के माध्यम से फिर भी काम करता है, लेकिन लूपबैक नियंत्रण पोर्ट पर कुछ भी नहीं सुनता।

- स्थिति/आरंभ/रोकें: `GET /`, `GET /doctor`, `POST /start`, `POST /stop`, `POST /reset-profile`
- प्रोफ़ाइल: `GET /profiles`, `POST /profiles/create`, `DELETE /profiles/:name`
- टैब: `GET /tabs`, `POST /tabs/open`, `POST /tabs/focus`, `DELETE /tabs/:targetId`, `POST /tabs/action`
- स्नैपशॉट/स्क्रीनशॉट: `GET /snapshot`, `POST /screenshot`
- क्रियाएँ: `POST /navigate`, `POST /act`
- हुक: `POST /hooks/file-chooser`, `POST /hooks/dialog`
- डाउनलोड: `POST /download`, `POST /wait/download`
- अनुमतियाँ: `POST /permissions/grant`
- डीबगिंग: `GET /console`, `POST /pdf`
- डीबगिंग: `GET /errors`, `GET /requests`, `GET /dialogs`, `POST /trace/start`, `POST /trace/stop`, `POST /highlight`
- नेटवर्क: `POST /response/body`
- स्थिति: `GET /cookies`, `POST /cookies/set`, `POST /cookies/clear`
- स्थिति: `GET /storage/:kind`, `POST /storage/:kind/set`, `POST /storage/:kind/clear`
- सेटिंग: `POST /set/offline`, `POST /set/headers`, `POST /set/credentials`, `POST /set/geolocation`, `POST /set/media`, `POST /set/timezone`, `POST /set/locale`, `POST /set/device`

`POST /tabs/action` वह बैच रूप है जिसे CLI आंतरिक रूप से
`browser tab` उपकमांडों (`{"action":"new"|"label"|"select"|"close"|"list", ...}`) के लिए उपयोग करता है;
सीधे स्क्रिप्टिंग करते समय ऊपर दिए गए एकल-उद्देश्य वाले टैब रूट को प्राथमिकता दें।

सभी एंडपॉइंट `?profile=<name>` स्वीकार करते हैं। `POST /start?headless=true` स्थायी
ब्राउज़र कॉन्फ़िगरेशन बदले बिना स्थानीय प्रबंधित प्रोफ़ाइल के लिए एक बार का हेडलेस लॉन्च
अनुरोध करता है; केवल-अटैच, रिमोट CDP और मौजूदा-सत्र प्रोफ़ाइल
उस ओवरराइड को अस्वीकार करती हैं क्योंकि OpenClaw उन ब्राउज़र प्रक्रियाओं को लॉन्च नहीं करता।

टैब एंडपॉइंट के लिए, `targetId` संगतता फ़ील्ड का नाम है। `GET /tabs` या `POST /tabs/open` से
`suggestedTargetId` पास करना बेहतर है; लेबल और `tabId`
हैंडल, जैसे `t1`, भी स्वीकार किए जाते हैं। कच्चे CDP लक्ष्य आईडी और अद्वितीय कच्चे
लक्ष्य-आईडी उपसर्ग अब भी काम करते हैं, लेकिन वे अस्थिर नैदानिक हैंडल हैं।

यदि साझा-गुप्त gateway प्रमाणीकरण कॉन्फ़िगर किया गया है, तो ब्राउज़र HTTP रूट के लिए भी प्रमाणीकरण आवश्यक है:

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` या उस पासवर्ड के साथ HTTP Basic प्रमाणीकरण

टिप्पणियाँ:

- यह स्वतंत्र लूपबैक ब्राउज़र API विश्वसनीय-प्रॉक्सी या
  Tailscale Serve पहचान हेडर का उपयोग **नहीं** करता।
- यदि `gateway.auth.mode`, `none` या `trusted-proxy` है, तो ये लूपबैक ब्राउज़र
  रूट उन पहचान-युक्त मोड को विरासत में नहीं लेते; इन्हें केवल लूपबैक तक सीमित रखें।

### `/act` त्रुटि अनुबंध

`POST /act` रूट-स्तरीय सत्यापन और
नीति विफलताओं के लिए संरचित त्रुटि प्रतिक्रिया का उपयोग करता है:

```json
{ "error": "<message>", "code": "ACT_*" }
```

वर्तमान `code` मान:

- `ACT_KIND_REQUIRED` (HTTP 400): `kind` अनुपस्थित है या पहचाना नहीं गया।
- `ACT_INVALID_REQUEST` (HTTP 400): क्रिया पेलोड सामान्यीकरण या सत्यापन में विफल रहा।
- `ACT_SELECTOR_UNSUPPORTED` (HTTP 400): `selector` का उपयोग असमर्थित क्रिया प्रकार के साथ किया गया।
- `ACT_EVALUATE_DISABLED` (HTTP 403): `evaluate` (या `wait --fn`) कॉन्फ़िगरेशन द्वारा अक्षम है।
- `ACT_TARGET_ID_MISMATCH` (HTTP 403): शीर्ष-स्तरीय या बैच किया हुआ `targetId` अनुरोध लक्ष्य से टकराता है।
- `ACT_EXISTING_SESSION_UNSUPPORTED` (HTTP 501): मौजूदा-सत्र प्रोफ़ाइल के लिए क्रिया समर्थित नहीं है।

अन्य रनटाइम विफलताएँ अब भी `code`
फ़ील्ड के बिना `{ "error": "<message>" }` लौटा सकती हैं।

### Playwright आवश्यकता

कुछ सुविधाओं (नेविगेट/क्रिया/AI स्नैपशॉट/भूमिका स्नैपशॉट, तत्व स्क्रीनशॉट,
PDF) के लिए Playwright आवश्यक है। यदि Playwright इंस्टॉल नहीं है, तो वे एंडपॉइंट
स्पष्ट 501 त्रुटि लौटाते हैं।

Playwright के बिना भी क्या काम करता है:

- ARIA स्नैपशॉट
- भूमिका-शैली अभिगम्यता स्नैपशॉट (`--interactive`, `--compact`,
  `--depth`, `--efficient`) जब प्रति-टैब CDP WebSocket उपलब्ध हो। यह
  निरीक्षण और रेफ़ खोजने के लिए फ़ॉलबैक है; Playwright प्राथमिक
  क्रिया इंजन बना रहता है।
- प्रबंधित `openclaw` ब्राउज़र के पृष्ठ स्क्रीनशॉट, जब प्रति-टैब CDP
  WebSocket उपलब्ध हो
- `existing-session` / Chrome MCP प्रोफ़ाइल के पृष्ठ स्क्रीनशॉट
- स्नैपशॉट आउटपुट से `existing-session` रेफ़-आधारित स्क्रीनशॉट (`--ref`)

किनके लिए अब भी Playwright आवश्यक है:

- `navigate`
- `act`
- Playwright के मूल AI स्नैपशॉट प्रारूप पर निर्भर AI स्नैपशॉट
- CSS-चयनकर्ता तत्व स्क्रीनशॉट (`--element`)
- पूर्ण ब्राउज़र PDF निर्यात

तत्व स्क्रीनशॉट `--full-page` को भी अस्वीकार करते हैं; रूट `fullPage is
not supported for element screenshots` लौटाता है।

यदि आपको `Playwright is not available in this gateway build` दिखाई देता है, तो पैकेज किए गए
Gateway में मुख्य ब्राउज़र रनटाइम निर्भरता अनुपस्थित है। OpenClaw को पुनः इंस्टॉल या अपडेट करें,
फिर gateway पुनः आरंभ करें। Docker के लिए, नीचे दिखाए अनुसार Chromium
ब्राउज़र बाइनरी भी इंस्टॉल करें।

#### Docker में Playwright इंस्टॉल करना

यदि आपका Gateway Docker में चलता है, तो `npx playwright` से बचें (npm ओवरराइड टकराव)।
कस्टम इमेज के लिए, Chromium को इमेज में शामिल करें:

```bash
OPENCLAW_INSTALL_BROWSER=1 ./scripts/docker/setup.sh
```

मौजूदा इमेज के लिए, इसके बजाय बंडल किए गए CLI के माध्यम से इंस्टॉल करें:

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

ब्राउज़र डाउनलोड स्थायी रखने के लिए, `PLAYWRIGHT_BROWSERS_PATH` सेट करें (उदाहरण के लिए,
`/home/node/.cache/ms-playwright`) और सुनिश्चित करें कि `/home/node` को
`OPENCLAW_HOME_VOLUME` या बाइंड माउंट के माध्यम से स्थायी रखा गया है। OpenClaw Linux पर स्थायी
Chromium का स्वतः पता लगाता है। [Docker](/hi/install/docker) देखें।

## यह कैसे काम करता है (आंतरिक)

एक छोटा लूपबैक नियंत्रण सर्वर HTTP अनुरोध स्वीकार करता है और CDP के माध्यम से Chromium-आधारित ब्राउज़र से जुड़ता है। उन्नत क्रियाएँ (क्लिक/टाइप/स्नैपशॉट/PDF) CDP के ऊपर Playwright के माध्यम से होती हैं; Playwright अनुपस्थित होने पर केवल गैर-Playwright क्रियाएँ उपलब्ध होती हैं। स्थानीय/रिमोट ब्राउज़र और प्रोफ़ाइल नीचे स्वतंत्र रूप से बदलते रहते हैं, जबकि एजेंट को एक स्थिर इंटरफ़ेस दिखाई देता है।

## CLI त्वरित संदर्भ

सभी कमांड किसी विशिष्ट प्रोफ़ाइल को लक्षित करने के लिए `--browser-profile <name>` और मशीन-पठनीय आउटपुट के लिए `--json` स्वीकार करते हैं।

<AccordionGroup>

<Accordion title="मूल बातें: स्थिति, टैब, खोलना/फ़ोकस करना/बंद करना">

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep    # सक्रिय स्नैपशॉट जाँच जोड़ें
openclaw browser start
openclaw browser start --headless # एक बार का स्थानीय प्रबंधित हेडलेस लॉन्च
openclaw browser stop            # केवल-अटैच/रिमोट CDP पर इम्यूलेशन भी साफ़ करता है
openclaw browser reset-profile   # प्रोफ़ाइल के ब्राउज़र डेटा को Trash में ले जाता है
openclaw browser tabs
openclaw browser tab             # वर्तमान टैब का शॉर्टकट
openclaw browser tab new
openclaw browser tab new --label research
openclaw browser tab label abcd1234 research
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://example.com
openclaw browser focus abcd1234
openclaw browser close abcd1234
```

</Accordion>

<Accordion title="प्रोफ़ाइल: सूचीबद्ध करना, बनाना, हटाना">

```bash
openclaw browser profiles
openclaw browser create-profile --name research --color "#0066CC"
openclaw browser create-profile --name attach --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser delete-profile --name research
```

</Accordion>

<Accordion title="निरीक्षण: स्क्रीनशॉट, स्नैपशॉट, कंसोल, त्रुटियाँ, अनुरोध">

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref 12        # या भूमिका रेफ़ के लिए --ref e12
openclaw browser screenshot --labels
openclaw browser snapshot
openclaw browser snapshot --format aria --limit 200
openclaw browser snapshot --interactive --compact --depth 6
openclaw browser snapshot --efficient
openclaw browser snapshot --labels
openclaw browser snapshot --urls
openclaw browser snapshot --selector "#main" --interactive
openclaw browser snapshot --frame "iframe#main" --interactive
openclaw browser snapshot --out snapshot.txt
openclaw browser console --level error
openclaw browser errors --clear
openclaw browser requests --filter api --clear
openclaw browser pdf
openclaw browser responsebody "**/api" --max-chars 5000
```

</Accordion>

<Accordion title="क्रियाएँ: नेविगेट करना, क्लिक करना, टाइप करना, खींचना, प्रतीक्षा करना, मूल्यांकन करना">

```bash
openclaw browser navigate https://example.com
openclaw browser resize 1280 720
openclaw browser click 12 --double           # या भूमिका रेफ़ के लिए e12
openclaw browser click-coords 120 340        # व्यूपोर्ट निर्देशांक
openclaw browser type 23 "hello" --submit
openclaw browser press Enter
openclaw browser hover 44
openclaw browser scrollintoview e12
openclaw browser drag 10 11
openclaw browser select 9 OptionA OptionB
openclaw browser download e12 report.pdf
openclaw browser waitfordownload report.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref e12
openclaw browser upload media://inbound/file.pdf
openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
openclaw browser wait --text "Done"
openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"
openclaw browser evaluate --fn '(el) => el.textContent' --ref 7
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
openclaw browser highlight e12
openclaw browser trace start
openclaw browser trace stop
```

</Accordion>

<Accordion title="स्थिति: कुकी, स्टोरेज, ऑफ़लाइन, हेडर, भू-स्थान, डिवाइस">

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url "https://example.com"
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set theme dark
openclaw browser storage session clear
openclaw browser set offline on
openclaw browser set headers --headers-json '{"X-Debug":"1"}'
openclaw browser set credentials user pass            # हटाने के लिए --clear
openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"
openclaw browser set media dark
openclaw browser set timezone America/New_York
openclaw browser set locale en-US
openclaw browser set device "iPhone 14"
```

</Accordion>

</AccordionGroup>

टिप्पणियाँ:

- एजेंट के लिए उपलब्ध `browser` टूल `action=download` (आवश्यक `ref` और
  `path`) तथा `action=waitfordownload` (वैकल्पिक `path`) को उजागर करता है। दोनों सहेजा गया
  डाउनलोड URL, सुझाया गया फ़ाइल नाम और सुरक्षित स्थानीय पथ लौटाते हैं। प्रबंधित Playwright प्रोफ़ाइलों के लिए स्पष्ट डाउनलोड
  अवरोधन उपलब्ध है; मौजूदा-सत्र
  प्रोफ़ाइलें असमर्थित-संचालन त्रुटि लौटाती हैं।
- एटॉमिक चयनकर्ता अपलोड को प्राथमिकता दें: अपलोड के साथ ट्रिगर `--ref` पास करें, ताकि OpenClaw एक ही अनुरोध में तैयार होकर क्लिक करे। जब बाद में ट्रिगर करना जानबूझकर हो, तब केवल-पथ `upload` समर्थित रहता है। किसी फ़ाइल इनपुट को सीधे सेट करने के लिए `--input-ref` या `--element` का उपयोग करें। `dialog` तैयार करने वाला कॉल है; संवाद को ट्रिगर करने वाले क्लिक/कुंजी-दबाव से पहले इसे चलाएँ। यदि कोई क्रिया मोडल खोलती है, तो क्रिया की प्रतिक्रिया में `blockedByDialog` और `browserState.dialogs.pending` शामिल होते हैं; सीधे प्रतिक्रिया देने के लिए वह `dialogId` पास करें। OpenClaw के बाहर संभाले गए संवाद `browserState.dialogs.recent` के अंतर्गत दिखाई देते हैं।
- `click`/`type`/आदि के लिए `snapshot` से प्राप्त `ref` आवश्यक है (संख्यात्मक `12`, भूमिका रेफ़ `e12`, या क्रिया-योग्य ARIA रेफ़ `ax12`)। क्रियाओं के लिए CSS चयनकर्ता जानबूझकर समर्थित नहीं हैं। जब दृश्यमान व्यूपोर्ट की स्थिति ही एकमात्र विश्वसनीय लक्ष्य हो, तब `click-coords` का उपयोग करें।
- डाउनलोड और ट्रेस पथ OpenClaw के अस्थायी रूट तक सीमित हैं: `/tmp/openclaw{,/downloads}` (फ़ॉलबैक: `${os.tmpdir()}/openclaw/...`)।
- `upload` OpenClaw के अस्थायी अपलोड रूट और
  OpenClaw द्वारा प्रबंधित इनबाउंड मीडिया से फ़ाइलें स्वीकार करता है। प्रबंधित इनबाउंड मीडिया को
  `media://inbound/<id>`, सैंडबॉक्स-सापेक्ष `media/inbound/<id>`, या प्रबंधित इनबाउंड मीडिया
  डायरेक्टरी के भीतर किसी हल किए गए पथ के रूप में संदर्भित किया जा सकता है। नेस्टेड मीडिया रेफ़,
  ट्रैवर्सल, सिमलिंक, हार्डलिंक और मनमाने स्थानीय पथ अब भी अस्वीकार किए जाते हैं।
- `upload` भी `--input-ref` या `--element` के माध्यम से फ़ाइल इनपुट सीधे सेट कर सकता है।

स्थिर टैब आईडी और लेबल Chromium द्वारा रॉ-टार्गेट प्रतिस्थापन के बाद भी बने रहते हैं, जब OpenClaw
प्रतिस्थापन टैब को प्रमाणित कर सकता है, जैसे समान URL के लिए एक अद्वितीय पुरानी/नई जोड़ी या
फ़ॉर्म सबमिट करने के बाद एक पुराना टैब एक नए टैब में बदलना। समान URL वाली अस्पष्ट
प्रतिस्थापन स्थितियों को नए हैंडल मिलते हैं। रॉ टार्गेट आईडी फिर भी
अस्थिर होते हैं; स्क्रिप्ट में `tabs` से प्राप्त `suggestedTargetId` को प्राथमिकता दें।

स्नैपशॉट फ़्लैग एक नज़र में:

- `--format ai` (Playwright के साथ डिफ़ॉल्ट): संख्यात्मक रेफ़ (`aria-ref="<n>"`) वाला AI स्नैपशॉट।
- `--format aria`: `axN` रेफ़ वाला अभिगम्यता ट्री। Playwright उपलब्ध होने पर, OpenClaw रेफ़ को बैकएंड DOM आईडी के साथ लाइव पृष्ठ से बाँधता है, ताकि बाद की क्रियाएँ उनका उपयोग कर सकें; अन्यथा आउटपुट को केवल निरीक्षण के लिए मानें।
- `--efficient` (या `--mode efficient`): संक्षिप्त भूमिका स्नैपशॉट प्रीसेट। इसे डिफ़ॉल्ट बनाने के लिए `browser.snapshotDefaults.mode: "efficient"` सेट करें ([Gateway कॉन्फ़िगरेशन](/hi/gateway/configuration-reference#browser) देखें)।
- `--interactive`, `--compact`, `--depth`, `--selector` `ref=e12` रेफ़ वाला भूमिका स्नैपशॉट बाध्य करते हैं। `--frame "<iframe>"` भूमिका स्नैपशॉट को एक iframe तक सीमित करता है।
- Playwright के साथ, `--labels` ऊपर आरोपित रेफ़ लेबल वाला स्क्रीनशॉट जोड़ता है
  (`MEDIA:<path>` प्रिंट करता है), साथ ही प्रत्येक रेफ़ के बाउंडिंग
  बॉक्स वाली `annotations` सरणी भी जोड़ता है। `screenshot` पर, Playwright-समर्थित लेबल `--full-page`,
  `--ref` और `--element` के साथ काम करते हैं; `snapshot` पर, साथ वाला स्क्रीनशॉट
  केवल व्यूपोर्ट तक सीमित रहता है। मौजूदा-सत्र/chrome-mcp प्रोफ़ाइलें पृष्ठ
  स्क्रीनशॉट पर ओवरले लेबल रेंडर करती हैं, लेकिन `annotations` नहीं लौटातीं और Playwright के
  पूर्ण-पृष्ठ/रेफ़/एलिमेंट प्रोजेक्शन सहायक का उपयोग नहीं करतीं। Playwright या chrome-mcp के बिना,
  लेबल वाले स्क्रीनशॉट उपलब्ध नहीं होते।
- `--urls` खोजे गए लिंक गंतव्यों को AI स्नैपशॉट में जोड़ता है।

## स्नैपशॉट और रेफ़

OpenClaw दो "स्नैपशॉट" शैलियों का समर्थन करता है:

- **AI स्नैपशॉट (संख्यात्मक रेफ़)**: `openclaw browser snapshot` (डिफ़ॉल्ट; `--format ai`)
  - आउटपुट: संख्यात्मक रेफ़ वाला टेक्स्ट स्नैपशॉट।
  - क्रियाएँ: `openclaw browser click 12`, `openclaw browser type 23 "hello"`।
  - आंतरिक रूप से, रेफ़ को Playwright के `aria-ref` के माध्यम से हल किया जाता है।

- **भूमिका स्नैपशॉट (`e12` जैसे भूमिका रेफ़)**: `openclaw browser snapshot --interactive` (या `--compact`, `--depth`, `--selector`, `--frame`)
  - आउटपुट: `[ref=e12]` (और वैकल्पिक `[nth=1]`) वाली भूमिका-आधारित सूची/ट्री।
  - क्रियाएँ: `openclaw browser click e12`, `openclaw browser highlight e12`।
  - आंतरिक रूप से, रेफ़ को `getByRole(...)` (डुप्लिकेट के लिए `nth()` के साथ) के माध्यम से हल किया जाता है।
  - ऊपर आरोपित `e12` लेबल वाला स्क्रीनशॉट शामिल करने के लिए `--labels` जोड़ें। Playwright-समर्थित
    प्रोफ़ाइलों पर यह प्रति-रेफ़ बाउंडिंग-बॉक्स मेटाडेटा
    (`annotations[]`) भी लौटाता है।
  - जब लिंक टेक्स्ट अस्पष्ट हो और एजेंट को ठोस
    नेविगेशन लक्ष्य चाहिए हों, तब `--urls` जोड़ें।

- **ARIA स्नैपशॉट (`ax12` जैसे ARIA रेफ़)**: `openclaw browser snapshot --format aria`
  - आउटपुट: संरचित नोड के रूप में अभिगम्यता ट्री।
  - क्रियाएँ: जब स्नैपशॉट पथ Playwright और Chrome बैकएंड DOM आईडी के माध्यम से
    रेफ़ को बाँध सकता है, तब `openclaw browser click ax12` काम करता है।
- यदि Playwright उपलब्ध नहीं है, तो ARIA स्नैपशॉट फिर भी
  निरीक्षण के लिए उपयोगी हो सकते हैं, लेकिन रेफ़ क्रिया-योग्य नहीं हो सकते। क्रिया रेफ़ की आवश्यकता होने पर
  `--format ai` या `--interactive` के साथ फिर से स्नैपशॉट लें।
- रॉ-CDP फ़ॉलबैक पथ के लिए Docker प्रमाण: `pnpm test:docker:browser-cdp-snapshot`
  Chromium को CDP के साथ शुरू करता है, `browser doctor --deep` चलाता है और सत्यापित करता है कि भूमिका
  स्नैपशॉट में लिंक URL, कर्सर द्वारा क्लिक-योग्य बनाए गए एलिमेंट और iframe मेटाडेटा शामिल हैं।

रेफ़ का व्यवहार:

- रेफ़ **नेविगेशन के दौरान स्थिर नहीं रहते**; यदि कुछ विफल होता है, तो `snapshot` फिर से चलाएँ और नया रेफ़ उपयोग करें।
- जब `/act` क्रिया द्वारा ट्रिगर किए गए प्रतिस्थापन टैब को प्रमाणित कर सकता है, तब यह प्रतिस्थापन के बाद
  वर्तमान रॉ `targetId` लौटाता है। बाद की कमांड के लिए स्थिर टैब आईडी/लेबल का
  उपयोग जारी रखें।
- यदि भूमिका स्नैपशॉट `--frame` के साथ लिया गया था, तो भूमिका रेफ़ अगले भूमिका स्नैपशॉट तक उसी iframe तक सीमित रहते हैं।
- अज्ञात या पुराने `axN` रेफ़ Playwright के `aria-ref` चयनकर्ता पर
  जाने के बजाय तुरंत विफल हो जाते हैं। ऐसा होने पर उसी टैब पर नया स्नैपशॉट
  चलाएँ।

## ब्राउज़र बैच CLI

`openclaw browser batch` एक `/act` कॉल में नेस्टेड `/act` क्रियाओं की सरणी चलाता है
(वही `kind="batch"` रनटाइम जिस तक एजेंट टूल के माध्यम से पहुँचा जाता है), ताकि CLI
उपयोगकर्ता और स्क्रिप्ट `wait`, `click`, `type` और
`evaluate` जैसी क्रियाओं को प्रति-क्रिया राउंड ट्रिप के बिना एक पुनः चलाने योग्य योजना में संयोजित कर सकें। `actions[]` की प्रत्येक
प्रविष्टि एक `BrowserActRequest` है — वह बंद यूनियन जिसे `/act`
रूट स्वीकार करता है (`click`, `clickCoords`, `type`, `press`, `hover`,
`scrollIntoView`, `drag`, `select`, `fill`, `resize`, `wait`, `evaluate`,
`close`, `batch`) — मनमानी `openclaw browser` उपकमांड नहीं। `batch`
`profile="user"` और अन्य मौजूदा-सत्र (chrome-mcp)
प्रोफ़ाइलों पर समर्थित नहीं है; वहाँ क्रियाएँ अलग-अलग भेजें।

- CLI: stdin से JSON सरणी पढ़ने के लिए `openclaw browser batch --actions '<json>'`, `openclaw browser batch
--actions-file plan.json`, या `openclaw browser batch --actions-file -`।
  `--continue`, `stopOnError=false` सेट करता है; डिफ़ॉल्ट व्यवहार
  पहली त्रुटि पर रुकना है। `--target-id` पूरे बैच को
  एक टैब तक सीमित करता है।
- रेफ़ जीवनचक्र: रेफ़ बैच से पहले चलाए गए `snapshot` से आते हैं (स्नैपशॉट
  नेस्टेड क्रिया नहीं है)। पृष्ठ की स्थिति बदलने वाली कोई नेस्टेड क्रिया — जैसे
  नेविगेशन ट्रिगर करने वाला `click`, या DOM बदलने वाला `evaluate` —
  शेष बैच के लिए पहले के रेफ़ को अमान्य कर सकती है। स्थिति बदलने वाली क्रियाएँ
  पहले रखें, या फिर से स्नैपशॉट लेने के बाद उन्हें अगले बैच में विभाजित करें। नेविगेशन और
  पुनः स्नैपशॉट लेना बैच के बाहर होता है (`openclaw browser navigate` /
  `snapshot`), क्योंकि `open`, `navigate` और `snapshot`, `/act` प्रकार नहीं हैं।
- लक्ष्य आईडी टकराव: कोई नेस्टेड क्रिया `targetId` को छोड़ सकती है या
  अनुरोध-स्तरीय `targetId` दोहरा सकती है; किसी अलग टैब में हल होने वाला स्पष्ट नेस्टेड `targetId`
  कोई भी क्रिया चलने से पहले `ACT_TARGET_ID_MISMATCH` के साथ अस्वीकार कर दिया जाता है।
  बैच की क्रियाएँ डिज़ाइन के अनुसार अनुरोध का टैब साझा करती हैं।
- त्रुटि सारांश: प्रतिक्रिया `{ "results": [{ "ok": true }, { "ok": false,
"error": "<message>" }, ...] }` होती है, जिसमें प्रत्येक क्रिया के लिए क्रम से एक प्रविष्टि होती है। जब
  `stopOnError` डिफ़ॉल्ट होता है, तो सरणी पहली विफलता पर समाप्त हो जाती है; `--continue` के साथ
  इसमें प्रत्येक क्रिया शामिल होती है। कोई भी विफल प्रविष्टि CLI को गैर-शून्य स्थिति के साथ
  बंद करती है; स्क्रिप्ट के लिए पूर्ण क्रमबद्ध प्रतिक्रिया बनाए रखने हेतु `--json` पास करें।

## प्रतीक्षा की उन्नत क्षमताएँ

आप केवल समय/टेक्स्ट से अधिक के लिए प्रतीक्षा कर सकते हैं:

- URL की प्रतीक्षा करें (Playwright द्वारा ग्लॉब समर्थित):
  - `openclaw browser wait --url "**/dash"`
- लोड स्थिति की प्रतीक्षा करें:
  - `openclaw browser wait --load networkidle`
  - प्रबंधित `openclaw` और रॉ/रिमोट CDP प्रोफ़ाइलों पर समर्थित। `existing-session` ड्राइवर का उपयोग करने वाली प्रोफ़ाइलें (डिफ़ॉल्ट `user` प्रोफ़ाइल सहित) `networkidle` को अस्वीकार करती हैं; वहाँ `--url`, `--text`, किसी चयनकर्ता या `--fn` प्रतीक्षाओं का उपयोग करें।
- JS प्रेडिकेट की प्रतीक्षा करें:
  - `openclaw browser wait --fn "window.ready===true"`
- किसी चयनकर्ता के दृश्यमान होने की प्रतीक्षा करें:
  - `openclaw browser wait "#main"`

इन्हें संयोजित किया जा सकता है:

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## डीबग कार्यप्रवाह

जब कोई क्रिया विफल होती है (जैसे "दृश्यमान नहीं", "सख्त मोड उल्लंघन", "ढका हुआ"):

1. `openclaw browser snapshot --interactive`
2. `click <ref>` / `type <ref>` का उपयोग करें (इंटरैक्टिव मोड में भूमिका रेफ़ को प्राथमिकता दें)
3. यदि यह फिर भी विफल हो: Playwright किसे लक्ष्य बना रहा है, यह देखने के लिए `openclaw browser highlight <ref>`
4. यदि पृष्ठ असामान्य व्यवहार करता है:
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. गहन डीबगिंग के लिए: ट्रेस रिकॉर्ड करें:
   - `openclaw browser trace start`
   - समस्या को पुनरुत्पादित करें
   - `openclaw browser trace stop` (`TRACE:<path>` प्रिंट करता है)

## JSON आउटपुट

`--json` स्क्रिप्टिंग और संरचित टूलिंग के लिए है।

उदाहरण:

```bash
openclaw browser --json status
openclaw browser --json snapshot --interactive
openclaw browser --json requests --filter api
openclaw browser --json cookies
```

JSON में भूमिका स्नैपशॉट में `refs` के साथ एक छोटा `stats` ब्लॉक (पंक्तियाँ/वर्ण/रेफ़/इंटरैक्टिव) शामिल होता है, ताकि टूल पेलोड के आकार और घनत्व का आकलन कर सकें।

## स्थिति और परिवेश नियंत्रण

ये "साइट को X जैसा व्यवहार कराएँ" कार्यप्रवाहों के लिए उपयोगी हैं:

- कुकीज़: `cookies`, `cookies set`, `cookies clear`
- स्टोरेज: `storage local|session get|set|clear`
- ऑफ़लाइन: `set offline on|off`
- हेडर: `set headers --headers-json '{"X-Debug":"1"}'` (या स्थितीय रूप `set headers '{"X-Debug":"1"}'`)
- HTTP बेसिक प्रमाणीकरण: `set credentials user pass` (या `--clear`)
- भौगोलिक स्थान: `set geo <lat> <lon> --origin "https://example.com"` (या `--clear`)
- मीडिया: `set media dark|light|no-preference|none`
- समय क्षेत्र / लोकेल: `set timezone ...`, `set locale ...`
- डिवाइस / व्यूपोर्ट:
  - `set device "iPhone 14"` (Playwright डिवाइस प्रीसेट)
  - `set viewport 1280 720`

## सुरक्षा और गोपनीयता

- openclaw ब्राउज़र प्रोफ़ाइल में लॉग-इन सत्र हो सकते हैं; इसे संवेदनशील मानें।
- `browser act kind=evaluate` / `openclaw browser evaluate` और `wait --fn`
  पृष्ठ के संदर्भ में मनमाना JavaScript निष्पादित करते हैं। प्रॉम्प्ट इंजेक्शन इसे
  निर्देशित कर सकता है। यदि आपको इसकी आवश्यकता नहीं है, तो इसे `browser.evaluateEnabled=false` से अक्षम करें।
- `openclaw browser evaluate --fn` फ़ंक्शन स्रोत, एक्सप्रेशन या
  स्टेटमेंट बॉडी स्वीकार करता है। स्टेटमेंट बॉडी को async फ़ंक्शन के रूप में रैप किया जाता है, इसलिए जो मान
  आप वापस पाना चाहते हैं, उसके लिए `return` का उपयोग करें। जब पृष्ठ-पक्षीय फ़ंक्शन को
  डिफ़ॉल्ट मूल्यांकन टाइमआउट से अधिक समय लग सकता हो, तब `--timeout-ms <ms>` का उपयोग करें।
- लॉगिन और एंटी-बॉट संबंधी टिप्पणियों (X/Twitter आदि) के लिए, [ब्राउज़र लॉगिन + X/Twitter पर पोस्ट करना](/hi/tools/browser-login) देखें।
- Gateway/Node होस्ट को निजी रखें (केवल लूपबैक या टेलनेट)।
- रिमोट CDP एंडपॉइंट शक्तिशाली होते हैं; उन्हें टनल करें और सुरक्षित रखें।

सख़्त-मोड उदाहरण (डिफ़ॉल्ट रूप से निजी/आंतरिक गंतव्यों को ब्लॉक करें):

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // वैकल्पिक सटीक अनुमति
    },
  },
}
```

## संबंधित

- [ब्राउज़र](/hi/tools/browser) - अवलोकन, कॉन्फ़िगरेशन, प्रोफ़ाइल, सुरक्षा
- [ब्राउज़र लॉगिन](/hi/tools/browser-login) - साइटों में साइन इन करना
- [ब्राउज़र Linux समस्या निवारण](/hi/tools/browser-linux-troubleshooting)
- [ब्राउज़र WSL2 समस्या निवारण](/hi/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
