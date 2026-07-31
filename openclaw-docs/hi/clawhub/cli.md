---
read_when:
    - ClawHub CLI का उपयोग करना
    - इंस्टॉल, अपडेट या पब्लिश की डीबगिंग
summary: 'CLI संदर्भ: कमांड, फ़्लैग, कॉन्फ़िगरेशन और लॉकफ़ाइल का व्यवहार।'
x-i18n:
    generated_at: "2026-07-27T19:22:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eba91a83c5542c4b570bd22a526911633e43d0b4e921c013e6fd29451193f2a7
    source_path: clawhub/cli.md
    workflow: 16
---

# CLI

CLI पैकेज: `clawhub`, बाइनरी: `clawhub`.

इसे npm या pnpm से वैश्विक रूप से इंस्टॉल करें:

```bash
npm i -g clawhub
# या
pnpm add -g clawhub
```

फिर इसे सत्यापित करें:

```bash
clawhub --help
clawhub login
clawhub whoami
```

## वैश्विक फ़्लैग

- `--workdir <dir>`: कार्यशील डायरेक्टरी (डिफ़ॉल्ट: cwd; कॉन्फ़िगर होने पर Clawdbot वर्कस्पेस पर फ़ॉलबैक)
- `--dir <dir>`: workdir के अंतर्गत इंस्टॉल डायरेक्टरी (डिफ़ॉल्ट: `skills`)
- `--site <url>`: ब्राउज़र लॉगिन के लिए आधार URL (डिफ़ॉल्ट: `https://clawhub.ai`)
- `--registry <url>`: API आधार URL (डिफ़ॉल्ट: खोजा गया, अन्यथा `https://clawhub.ai`)
- `--no-input`: प्रॉम्प्ट अक्षम करें

समतुल्य परिवेश चर:

- `CLAWHUB_SITE` (पुराना `CLAWDHUB_SITE`)
- `CLAWHUB_REGISTRY` (पुराना `CLAWDHUB_REGISTRY`)
- `CLAWHUB_WORKDIR` (पुराना `CLAWDHUB_WORKDIR`)

### HTTP प्रॉक्सी

CLI कॉर्पोरेट प्रॉक्सी या प्रतिबंधित नेटवर्क के पीछे स्थित सिस्टम के लिए मानक
HTTP प्रॉक्सी परिवेश चरों का पालन करता है:

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `NO_PROXY` / `no_proxy`

इनमें से कोई भी चर सेट होने पर, CLI आउटबाउंड अनुरोधों को
निर्दिष्ट प्रॉक्सी के माध्यम से रूट करता है। HTTPS अनुरोधों के लिए `HTTPS_PROXY`, साधारण HTTP के लिए `HTTP_PROXY`
का उपयोग होता है। विशिष्ट होस्ट या डोमेन के लिए प्रॉक्सी को बायपास करने हेतु
`NO_PROXY` / `no_proxy` का पालन किया जाता है।

यह उन सिस्टम पर आवश्यक है जहाँ सीधे आउटबाउंड कनेक्शन अवरुद्ध होते हैं
(जैसे Docker कंटेनर, केवल-प्रॉक्सी इंटरनेट वाला Hetzner VPS, कॉर्पोरेट
फ़ायरवॉल)।

उदाहरण:

```bash
export HTTPS_PROXY=http://proxy.example.com:3128
export NO_PROXY=localhost,127.0.0.1
clawhub search "मेरी क्वेरी"
```

कोई प्रॉक्सी चर सेट न होने पर व्यवहार अपरिवर्तित रहता है (सीधे कनेक्शन)।

## कॉन्फ़िगरेशन फ़ाइल

आपका API टोकन और कैश किया हुआ रजिस्ट्री URL संग्रहीत करती है।

- macOS: `~/Library/Application Support/clawhub/config.json`
- Linux/XDG: `$XDG_CONFIG_HOME/clawhub/config.json` या `~/.config/clawhub/config.json`
- Windows: `%APPDATA%\\clawhub\\config.json`
- पुराना फ़ॉलबैक: यदि `clawhub/config.json` अभी मौजूद नहीं है, लेकिन `clawdhub/config.json` मौजूद है, तो CLI पुराने पथ का पुनः उपयोग करता है
- ओवरराइड: `CLAWHUB_CONFIG_PATH` (पुराना `CLAWDHUB_CONFIG_PATH`)

## कमांड

### `login` / `auth login`

- डिफ़ॉल्ट: ब्राउज़र में `<site>/cli/auth` खोलता है और लूपबैक कॉलबैक के माध्यम से प्रक्रिया पूरी करता है।
- हेडलेस: `clawhub login --token clh_...`
- रिमोट/हेडलेस इंटरैक्टिव: `clawhub login --device` एक कोड प्रिंट करता है और आपके द्वारा `<site>/cli/device` पर उसे अधिकृत किए जाने तक प्रतीक्षा करता है।

### `whoami`

- संग्रहीत टोकन को `/api/v1/whoami` के माध्यम से सत्यापित करता है।

### `token`

- संग्रहीत API टोकन को stdout पर प्रिंट करता है।
- स्थानीय लॉगिन टोकन को CI सीक्रेट सेटअप कमांड में पाइप करने के लिए उपयोगी।

### `star <skill>` / `unstar <skill>`

- आपके बुकमार्क में किसी स्किल को जोड़ता/हटाता है। संगतता के लिए कमांड नाम `star` और
  `unstar` ही रहते हैं।
- `POST /api/v1/stars/<slug>` और `DELETE /api/v1/stars/<slug>` को कॉल करता है।
- `--yes` पुष्टि को छोड़ देता है।

### `search <query...>`

- `/api/v1/search?q=...` को कॉल करता है।
- आउटपुट में स्किल स्लग, स्वामी हैंडल, प्रदर्शन नाम और प्रासंगिकता स्कोर शामिल होते हैं।
- खोज, डाउनलोड लोकप्रियता से पहले सटीक स्लग/नाम टोकन मिलानों को प्राथमिकता देती है। `map` जैसा स्वतंत्र स्लग टोकन, `amap` के अंदर मौजूद उपस्ट्रिंग की तुलना में `personal-map` से अधिक मज़बूती से मेल खाता है।
- लोकप्रियता रैंकिंग का एक छोटा पूर्व-कारक है, शीर्ष स्थान की गारंटी नहीं।
- यदि कोई स्किल दिखाई देनी चाहिए लेकिन दिखाई नहीं देती, तो मेटाडेटा का नाम बदलने से पहले, लॉग इन रहते हुए `clawhub inspect @owner/slug` चलाकर स्वामी को दिखाई देने वाले मॉडरेशन निदान जाँचें।

### `explore`

- `/api/v1/skills?limit=...&sort=createdAt` के माध्यम से नवीनतम स्किल सूचीबद्ध करता है (`createdAt` के अनुसार अवरोही क्रम में)।
- फ़्लैग:
  - `--limit <n>` (1-200, डिफ़ॉल्ट: 25)
  - `--sort newest|updated|rating|downloads|trending` (डिफ़ॉल्ट: नवीनतम)। संगतता के लिए पुराने इंस्टॉल सॉर्ट उपनाम अभी भी काम करते हैं।
  - `--json` (मशीन-पठनीय आउटपुट)
- आउटपुट: `<slug>  v<version>  <age>  <summary>` (सारांश 50 वर्णों तक छोटा किया गया)।

### `inspect @owner/slug`

- इंस्टॉल किए बिना स्किल मेटाडेटा और संस्करण फ़ाइलें प्राप्त करता है।
- `--version <version>`: किसी विशिष्ट संस्करण का निरीक्षण करें (डिफ़ॉल्ट: नवीनतम)।
- `--tag <tag>`: किसी टैग किए गए संस्करण का निरीक्षण करें (जैसे `latest`)।
- `--versions`: संस्करण इतिहास सूचीबद्ध करें (पहला पृष्ठ)।
- `--limit <n>`: सूचीबद्ध करने के लिए अधिकतम संस्करण (1-200)।
- `--files`: चयनित संस्करण की फ़ाइलें सूचीबद्ध करें।
- `--file <path>`: कच्चे फ़ाइल बाइट प्राप्त करें (10MB सीमा)।
- `--json`: मशीन-पठनीय आउटपुट; उपलब्ध होने पर `--file` में सटीक बाइट base64 और UTF-8 पाठ के रूप में शामिल होते हैं।

### `install @owner/slug`

- नामित स्वामी और स्किल के लिए नवीनतम संस्करण निर्धारित करता है।
- `/api/v1/download` के माध्यम से zip डाउनलोड करता है।
- `<workdir>/<dir>/<slug>` में एक्सट्रैक्ट करता है।
- पिन की गई स्किल को ओवरराइट करने से मना करता है; पहले `clawhub unpin <skill>` चलाएँ।
- लिखता है:
  - `<workdir>/.clawhub/lock.json` (पुराना `.clawdhub`)
  - `<skill>/.clawhub/origin.json` (पुराना `.clawdhub`)

### `uninstall <skill>`

- `<workdir>/<dir>/<slug>` हटाता है और लॉकफ़ाइल प्रविष्टि मिटाता है।
- लॉग इन होने पर सर्वोत्तम-प्रयास टेलीमेट्री भेजता है, ताकि वर्तमान इंस्टॉल संख्या को
  निष्क्रिय किया जा सके।
- इंटरैक्टिव: पुष्टि माँगता है।
- गैर-इंटरैक्टिव (`--no-input`): `--yes` आवश्यक है।

### `list`

- `<workdir>/.clawhub/lock.json` पढ़ता है (पुराना `.clawdhub`)।
- `clawhub pin` से फ़्रीज़ की गई स्किल के आगे `pinned` दिखाता है, जिसमें वैकल्पिक कारण भी शामिल है।

### `pin <skill>`

- इंस्टॉल की गई स्किल को लॉकफ़ाइल में पिन की गई के रूप में चिह्नित करता है।
- `--reason <text>` दर्ज करता है कि स्किल क्यों फ़्रीज़ की गई है।
- पिन की गई स्किल को `update --all` छोड़ देता है और प्रत्यक्ष `update <skill>` उन्हें अस्वीकार करता है।
- पिन की गई स्किल `install --force` को भी अस्वीकार करती है, ताकि स्थानीय बाइट अनजाने में बदले न जा सकें।

### `unpin <skill>`

- इंस्टॉल की गई स्किल से लॉकफ़ाइल पिन हटाता है, ताकि भविष्य के अपडेट उसे संशोधित कर सकें।

### `update [@owner/slug]` / `update --all`

- स्थानीय फ़ाइलों से फ़िंगरप्रिंट की गणना करता है।
- यदि फ़िंगरप्रिंट किसी ज्ञात संस्करण से मेल खाता है: कोई प्रॉम्प्ट नहीं।
- यदि फ़िंगरप्रिंट मेल नहीं खाता:
  - डिफ़ॉल्ट रूप से अस्वीकार करता है
  - `--force` से ओवरराइट करता है (या इंटरैक्टिव होने पर प्रॉम्प्ट करता है)
- पिन की गई स्किल को `--force` कभी अपडेट नहीं करता।
- `update <skill>` पिन की गई स्किल के लिए तुरंत विफल होता है और पहले `clawhub unpin <skill>` चलाने को कहता है।
- `update --all` पिन किए गए स्लग छोड़ देता है और जो फ़्रीज़ रहे उनका सारांश प्रिंट करता है।

### `skill publish <path>`

- स्थानीय बंडल फ़िंगरप्रिंट की तुलना ClawHub से करता है और सामग्री पहले से प्रकाशित होने पर
  सफलतापूर्वक बाहर निकलता है।
- नई स्किल का डिफ़ॉल्ट `1.0.0` होता है; बदली गई स्किल का डिफ़ॉल्ट अगला पैच
  संस्करण होता है।
- `--version <version>` स्पष्ट रूप से एक संस्करण चुनता है और सामग्री किसी मौजूदा संस्करण से
  मेल खाने पर भी प्रकाशित करता है।
- `--dry-run` अपलोड किए बिना प्रकाशन का समाधान करता है; `--json` एक
  मशीन-पठनीय परिणाम प्रिंट करता है।
- जब कर्ता के पास प्रकाशक पहुँच होती है, तो `--owner <handle>` किसी संगठन/उपयोगकर्ता प्रकाशक हैंडल के अंतर्गत
  प्रकाशित करता है।
- `--migrate-owner` नया संस्करण प्रकाशित करते समय किसी मौजूदा स्किल को `--owner` पर
  ले जाता है। दोनों प्रकाशकों पर व्यवस्थापक/स्वामी पहुँच आवश्यक है।
- स्वामी और समीक्षा व्यवहार की व्याख्या `docs/publishing.md` में की गई है।
- किसी स्किल को प्रकाशित करने का अर्थ है कि उसे ClawHub पर `MIT-0` के अंतर्गत जारी किया जाता है।
- प्रकाशित स्किल का उपयोग, संशोधन और पुनर्वितरण बिना श्रेय के निःशुल्क किया जा सकता है।
- ClawHub सशुल्क स्किल या प्रति-स्किल मूल्य निर्धारण का समर्थन नहीं करता।
- पुराना उपनाम: `publish <path>`।

```bash
clawhub skill publish ./my-skill --dry-run
clawhub skill publish ./my-skill
clawhub skill publish ./my-skill --version 2.0.0
```

#### GitHub Actions

ClawHub का पुनः उपयोग योग्य
[`skill-publish.yml`](https://github.com/openclaw/clawhub/blob/main/.github/workflows/skill-publish.yml)
वर्कफ़्लो एक `skill_path` के लिए, या `root` (डिफ़ॉल्ट: `skills`) के अंतर्गत प्रत्येक सीधे स्किल
फ़ोल्डर के लिए `skill publish` को कॉल करता है। यह अपरिवर्तित स्किल को छोड़ देता है और
उसी स्वचालित पैच-संस्करण व्यवहार का उपयोग करता है।

टोकन के बिना पूर्वावलोकन करने के लिए `dry_run: true` सेट करें। वास्तविक प्रकाशन के लिए
`clawhub_token` सीक्रेट आवश्यक है।

### `sync`

- वर्तमान workdir, कॉन्फ़िगर की गई स्किल डायरेक्टरी और किसी भी
  `--root <dir>` फ़ोल्डर को उन स्थानीय स्किल फ़ोल्डर के लिए स्कैन करता है जिनमें `SKILL.md` या
  `skill.md` हो।
- प्रत्येक स्थानीय स्किल फ़िंगरप्रिंट की तुलना ClawHub से करता है और केवल नई या
  बदली गई स्किल प्रकाशित करता है।
- नई स्किल `1.0.0` के रूप में प्रकाशित होती हैं; बदली गई स्किल डिफ़ॉल्ट रूप से अगला पैच संस्करण
  प्रकाशित करती हैं। उन अपडेट बैचों के लिए `--bump minor|major` का उपयोग करें जिन्हें
  बड़े semver चरण से आगे बढ़ना चाहिए।
- `--dry-run` अपलोड किए बिना प्रकाशन योजना दिखाता है; `--json` एक
  मशीन-पठनीय योजना प्रिंट करता है।
- `--all` बिना प्रॉम्प्ट किए प्रत्येक नई या बदली गई स्किल प्रकाशित करता है। `--all` के बिना,
  इंटरैक्टिव टर्मिनल आपको प्रकाशित करने के लिए स्किल चुनने देते हैं।
- जब कर्ता के पास प्रकाशक पहुँच होती है, तो `--owner <handle>` किसी संगठन/उपयोगकर्ता प्रकाशक हैंडल के अंतर्गत
  प्रकाशित करता है।
- `sync` केवल एक-तरफ़ा प्रकाशन है। यह इंस्टॉल, अपडेट, डाउनलोड या
  इंस्टॉल/डाउनलोड टेलीमेट्री रिपोर्ट नहीं करता।

```bash
clawhub sync --all --dry-run
clawhub sync --all
clawhub sync --root ./skills --owner openclaw --bump minor
```

### `scan --slug <slug>`

- `clawhub login` आवश्यक है।
- `POST /api/v1/skills/-/scan` के माध्यम से ClawHub ClawScan चलाता है, फिर स्कैन के अंतिम स्थिति में पहुँचने तक पोल करता है।
- स्कैन अतुल्यकालिक होते हैं और पूरे होने में समय लग सकता है। कतार में रहते समय, टर्मिनल स्पिनर वर्तमान प्राथमिकता-प्राप्त स्कैन स्थान और आगे मौजूद स्कैन की संख्या दिखाता है।
- प्रकाशित स्कैन के लिए स्वामित्व या प्रकाशक प्रबंधन पहुँच आवश्यक है। मॉडरेटर/व्यवस्थापक `clawhub-admin` के माध्यम से उसी बैकएंड का उपयोग कर सकते हैं।
- `--update` केवल `--slug` के साथ मान्य है; यह सफल प्रकाशित स्कैन परिणामों को चयनित संस्करण में वापस लिखता है।
- `--output <file.zip>`, `manifest.json`, `clawscan.json`, `skillspector.json`, `static-analysis.json`, `virustotal.json`, और `README.md` सहित पूरी रिपोर्ट आर्काइव डाउनलोड करता है।
- `--json` स्वचालन के लिए पूर्ण पोल प्रतिक्रिया प्रिंट करता है।
- स्थानीय पथ स्कैन अब समर्थित नहीं हैं। नया संस्करण अपलोड करें, फिर उस सबमिट किए गए संस्करण के संग्रहीत स्कैन परिणाम प्राप्त करने के लिए `scan download` का उपयोग करें।

```bash
clawhub scan --slug gifgrep
clawhub scan --slug gifgrep --version 1.2.3
clawhub scan --slug gifgrep --update --output report.zip
```

### `scan download <name>`

- के लिए `clawhub login` आवश्यक है।
- सबमिट किए गए skill या Plugin संस्करण के लिए संग्रहीत स्कैन रिपोर्ट ZIP डाउनलोड करता है, जिसमें वे संस्करण भी शामिल हैं जिन्हें ClawHub सुरक्षा जाँचों ने अवरुद्ध या छिपाया था।
- Skill डाउनलोड skill slug का उपयोग करते हैं और डिफ़ॉल्ट रूप से `--kind skill` का उपयोग करते हैं।
- Plugin डाउनलोड पैकेज नाम का उपयोग करते हैं और उनके लिए `--kind plugin` आवश्यक है।
- `--version` आवश्यक है, ताकि लेखक ClawHub द्वारा अवरुद्ध किए गए बिल्कुल उसी सबमिट किए गए संस्करण का निरीक्षण करें।
- `--output <file.zip>` गंतव्य पथ चुनता है।

```bash
clawhub scan download gifgrep --version 1.2.3
clawhub scan download @scope/demo --version 2.0.0 --kind plugin --output report.zip
```

#### GitHub Actions

ClawHub, skill रिपॉज़िटरी और कैटलॉग रिपॉज़िटरी के लिए
[`/.github/workflows/skill-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/skill-publish.yml)
पर एक आधिकारिक पुनः उपयोग योग्य वर्कफ़्लो प्रदान करता है।

सामान्य कैटलॉग सेटअप:

```yaml
name: Skill Publish

on:
  pull_request:
  workflow_dispatch:

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

टिप्पणियाँ:

- कैटलॉग रिपॉज़िटरी के लिए `root` का डिफ़ॉल्ट मान `skills` है।
- एक skill फ़ोल्डर को संसाधित करने के लिए `skill_path: skills/review-helper` पास करें।
- `owner` CLI के `--owner` फ़्लैग से मैप होता है; प्रमाणित उपयोगकर्ता के रूप में प्रकाशित करने के लिए इसे छोड़ दें।
- V1 skill प्रकाशन `clawhub_token` का उपयोग करता है; फिलहाल GitHub OIDC विश्वसनीय प्रकाशन केवल पैकेज के लिए है।

### `delete <skill>`

- `--version` के बिना, किसी skill को सॉफ़्ट-डिलीट करें (स्वामी, मॉडरेटर या एडमिन)।
- `DELETE /api/v1/skills/{slug}` को कॉल करता है।
- स्वामी द्वारा शुरू किए गए सॉफ़्ट-डिलीट slug को 30 दिनों के लिए आरक्षित रखते हैं; कमांड समाप्ति समय प्रिंट करता है।
- `--version <version>` विफल होने पर बंद होने वाले, संस्करण-विशिष्ट रूट के माध्यम से स्वामित्व वाले किसी एक गैर-नवीनतम संस्करण को वापस लेता है। संस्करण संख्या आरक्षित रहती है और अलग सामग्री के साथ दोबारा प्रकाशित नहीं की जा सकती। वर्तमान नवीनतम संस्करण को हटाने से पहले उसका प्रतिस्थापन प्रकाशित करें। इस केवल-संस्करण प्रवाह में प्लेटफ़ॉर्म कर्मचारी स्वामित्व को दरकिनार नहीं करते।
- `--reason <text>` पूरे skill के सॉफ़्ट-डिलीट और ऑडिट लॉग में मॉडरेशन टिप्पणी दर्ज करता है।
- `--note <text>`, `--reason` का उपनाम है।
- `--yes` पुष्टि को छोड़ देता है।

### `undelete <skill>`

- किसी छिपे हुए skill को पुनर्स्थापित करें (स्वामी, मॉडरेटर या एडमिन)।
- `POST /api/v1/skills/{slug}/undelete` को कॉल करता है।
- `--version <version>` केवल उसी स्वामी कर्ता द्वारा पहले वापस ली गई, सुरक्षित रखी गई बिल्कुल उसी कलाकृति को पुनर्स्थापित करता है। यह पुनर्स्थापित संस्करण को नवीनतम नहीं बनाता या हटाए गए टैग दोबारा नहीं बनाता।
- संस्करण पुनर्स्थापन `POST /api/v1/skills/{slug}/versions/{version}/restore` को कॉल करता है।
- `--reason <text>` skill और ऑडिट लॉग में मॉडरेशन टिप्पणी दर्ज करता है।
- `--note <text>`, `--reason` का उपनाम है।
- `--yes` पुष्टि को छोड़ देता है।

### `hide <skill>`

- किसी skill को छिपाएँ (स्वामी, मॉडरेटर या एडमिन)।
- `delete` का उपनाम।

### `unhide <skill>`

- किसी skill को फिर से दिखाएँ (स्वामी, मॉडरेटर या एडमिन)।
- `undelete` का उपनाम।

### `skill rename <skill> <new-name>`

- स्वामित्व वाले किसी skill का नाम बदलें और पिछले slug को रीडायरेक्ट उपनाम के रूप में रखें।
- `POST /api/v1/skills/{slug}/rename` को कॉल करता है।
- `--yes` पुष्टि को छोड़ देता है।

### `skill merge <source> <target>`

- स्वामित्व वाले एक skill को स्वामित्व वाले दूसरे skill में मर्ज करें।
- स्रोत slug सार्वजनिक रूप से सूचीबद्ध होना बंद कर देता है और लक्ष्य का रीडायरेक्ट उपनाम बन जाता है।
- `POST /api/v1/skills/{sourceSlug}/merge` को कॉल करता है।
- `--yes` पुष्टि को छोड़ देता है।

### `transfer`

- स्वामित्व हस्तांतरण वर्कफ़्लो।
- उपयोगकर्ता हैंडल को किए गए हस्तांतरण एक लंबित अनुरोध बनाते हैं, जिसे प्राप्तकर्ता स्वीकार करता है।
- संगठन/प्रकाशक हैंडल को किए गए हस्तांतरण तभी तुरंत लागू होते हैं, जब कर्ता के पास वर्तमान स्वामी और गंतव्य प्रकाशक दोनों पर एडमिन पहुँच हो।
- उप-कमांड:
  - `transfer request <skill> <handle> [--message "..."] [--yes]`
  - `transfer list [--outgoing]`
  - `transfer accept <skill> [--yes]`
  - `transfer reject <skill> [--yes]`
  - `transfer cancel <skill> [--yes]`
- एंडपॉइंट:
  - `POST /api/v1/skills/{slug}/transfer`
  - `POST /api/v1/skills/{slug}/transfer/accept`
  - `POST /api/v1/skills/{slug}/transfer/reject`
  - `POST /api/v1/skills/{slug}/transfer/cancel`
  - `GET /api/v1/transfers/incoming`
  - `GET /api/v1/transfers/outgoing`

### `package explore [query...]`

- `GET /api/v1/packages` और `GET /api/v1/packages/search` के माध्यम से एकीकृत पैकेज कैटलॉग ब्राउज़ करता है या उसमें खोज करता है।
- Plugin और अन्य पैकेज-परिवार प्रविष्टियों के लिए इसका उपयोग करें; शीर्ष-स्तरीय `search` skill खोज सतह बना रहता है।
- फ़्लैग:
  - `--family skill|code-plugin|bundle-plugin`
  - `--official`
  - `--executes-code`
  - `--target <target>`, `--os <os>`, `--arch <arch>`, `--libc <libc>`
  - `--requires-browser`, `--requires-desktop`, `--requires-native-deps`
  - `--requires-external-service`, `--external-service <name>`
  - `--binary <name>`, `--os-permission <name>`
  - `--artifact-kind legacy-zip|npm-pack`
  - `--npm-mirror`
  - `--limit <n>` (1-100, डिफ़ॉल्ट: 25)
  - `--json`

उदाहरण:

```bash
clawhub package explore --family code-plugin
clawhub package explore --family code-plugin --os darwin --requires-desktop
clawhub package explore --family code-plugin --artifact-kind npm-pack
clawhub package explore --npm-mirror
clawhub package explore episodic-claw --family code-plugin
```

### `package inspect <name>`

- इंस्टॉल किए बिना पैकेज मेटाडेटा प्राप्त करता है।
- Plugin मेटाडेटा, संगतता, सत्यापन, स्रोत और संस्करण/फ़ाइल निरीक्षण के लिए इसका उपयोग करें।
- `--version <version>`: किसी विशिष्ट संस्करण का निरीक्षण करें (डिफ़ॉल्ट: नवीनतम)।
- `--tag <tag>`: किसी टैग किए गए संस्करण का निरीक्षण करें (उदाहरण: `latest`)।
- `--versions`: संस्करण इतिहास सूचीबद्ध करें (पहला पृष्ठ)।
- `--limit <n>`: सूचीबद्ध किए जाने वाले संस्करणों की अधिकतम संख्या (1-100)।
- `--files`: चयनित संस्करण की फ़ाइलें सूचीबद्ध करें।
- `--file <path>`: सीमित UTF-8 टेक्स्ट पूर्वावलोकन प्राप्त करें (200KB सीमा)।
- `--json`: मशीन-पठनीय आउटपुट।

### `package download <name>`

- `GET /api/v1/packages/{name}/versions/{version}/artifact` के माध्यम से पैकेज संस्करण का समाधान करता है।
- रिज़ॉल्वर के `downloadUrl` से कलाकृति डाउनलोड करता है।
- सभी कलाकृतियों के लिए ClawHub SHA-256 सत्यापित करता है।
- ClawPack npm-pack कलाकृतियों के लिए npm `sha512` अखंडता, npm shasum और tarball के `package.json` नाम/संस्करण को भी सत्यापित करता है।
- पुराने ZIP संस्करण पुराने ZIP रूट के माध्यम से डाउनलोड होते हैं।
- फ़्लैग:
  - `--version <version>`: कोई विशिष्ट संस्करण डाउनलोड करें।
  - `--tag <tag>`: कोई टैग किया गया संस्करण डाउनलोड करें (डिफ़ॉल्ट: `latest`)।
  - `-o, --output <path>`: आउटपुट फ़ाइल या निर्देशिका।
  - `--force`: किसी मौजूदा आउटपुट फ़ाइल को अधिलेखित करें।
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package download @openclaw/example-plugin --tag latest
clawhub package download @openclaw/example-plugin --version 1.2.3 -o artifacts/
```

### `package verify <file>`

- स्थानीय कलाकृति के लिए ClawHub SHA-256, npm `sha512` अखंडता और npm shasum की गणना करता है।
- `--package` के साथ, ClawHub से अपेक्षित मेटाडेटा का समाधान करता है और स्थानीय फ़ाइल की प्रकाशित कलाकृति मेटाडेटा से तुलना करता है।
- प्रत्यक्ष डाइजेस्ट फ़्लैग के साथ, नेटवर्क लुकअप के बिना सत्यापित करता है।
- फ़्लैग:
  - `--package <name>`: अपेक्षित कलाकृति मेटाडेटा का समाधान करने के लिए पैकेज नाम।
  - `--version <version>` या `--tag <tag>`: अपेक्षित पैकेज संस्करण।
  - `--sha256 <hex>`: अपेक्षित ClawHub SHA-256।
  - `--npm-integrity <sri>`: अपेक्षित npm अखंडता।
  - `--npm-shasum <sha1>`: अपेक्षित npm shasum।
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package verify ./example-plugin-1.2.3.tgz --package @openclaw/example-plugin --version 1.2.3
clawhub package verify ./example-plugin-1.2.3.tgz --sha256 <hex>
```

### `package validate <source>`

- स्थानीय Plugin पैकेज फ़ोल्डर पर ClawHub CLI का बंडल किया हुआ Plugin Inspector चलाता है।
- स्थानीय OpenClaw चेकआउट खोजे या आयात किए बिना, डिफ़ॉल्ट रूप से ऑफ़लाइन/स्थैतिक सत्यापन करता है।
- गंभीर संगतता त्रुटियों पर गैर-शून्य निकास होता है। केवल-चेतावनी निष्कर्ष प्रिंट होते हैं, लेकिन निकास शून्य होता है।
- फ़्लैग:
  - `--out <dir>`: इस निर्देशिका में Plugin Inspector रिपोर्ट लिखें।
  - `--openclaw <path>`: किसी स्पष्ट स्थानीय OpenClaw चेकआउट के विरुद्ध निरीक्षण करें।
  - `--runtime`: रनटाइम कैप्चर सक्षम करें; Plugin कोड आयात करता है।
  - `--allow-execute`: अलग-थलग कार्यक्षेत्र में रनटाइम कैप्चर की अनुमति दें।
  - `--no-mock-sdk`: रनटाइम कैप्चर के दौरान मॉक किए गए OpenClaw SDK को अक्षम करें।
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package validate ./example-plugin
```

यदि सत्यापन किसी पैकेज, मैनिफ़ेस्ट, SDK आयात या कलाकृति संबंधी निष्कर्ष की रिपोर्ट करता है, तो
[Plugin सत्यापन सुधार](/hi/clawhub/plugin-validation-fixes) देखें, फिर कमांड दोबारा चलाएँ।

### `package delete <name>`

- `--version` के बिना, किसी पैकेज और उसके सभी रिलीज़ को सॉफ़्ट-डिलीट करता है।
- `--version <version>` विफल होने पर बंद होने वाले, संस्करण-विशिष्ट रूट के माध्यम से स्वामित्व वाली किसी एक गैर-नवीनतम रिलीज़ को वापस लेता है। संस्करण संख्या आरक्षित रहती है और अलग सामग्री के साथ दोबारा प्रकाशित नहीं की जा सकती। वर्तमान नवीनतम संस्करण को हटाने से पहले उसका प्रतिस्थापन प्रकाशित करें। इस केवल-संस्करण प्रवाह के लिए पैकेज स्वामी या संगठन प्रकाशक एडमिन आवश्यक है; प्लेटफ़ॉर्म कर्मचारी पैकेज स्वामित्व को दरकिनार नहीं करते।
- पूरे पैकेज के सॉफ़्ट-डिलीट के लिए पैकेज स्वामी, संगठन प्रकाशक स्वामी/एडमिन, प्लेटफ़ॉर्म मॉडरेटर या प्लेटफ़ॉर्म एडमिन आवश्यक है।
- फ़्लैग:
  - `--version <version>`: किसी एक गैर-नवीनतम संस्करण को वापस लें।
  - `--yes`: पुष्टि को छोड़ दें।
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package delete @openclaw/example-plugin --yes
clawhub package delete @openclaw/example-plugin --version 1.2.3 --yes
```

### `package undelete <name>`

- सॉफ़्ट-डिलीट किए गए पैकेज और रिलीज़ को पुनर्स्थापित करता है।
- इसके लिए पैकेज स्वामी, संगठन प्रकाशक स्वामी/एडमिन, प्लेटफ़ॉर्म मॉडरेटर या प्लेटफ़ॉर्म एडमिन आवश्यक है।
- `POST /api/v1/packages/{name}/undelete` को कॉल करता है।
- `--version <version>` केवल उसी स्वामी कर्ता द्वारा पहले वापस ली गई, सुरक्षित रखी गई बिल्कुल उसी रिलीज़ को पुनर्स्थापित करता है। यह रिलीज़ को नवीनतम नहीं बनाता या हटाए गए पैकेज टैग/dist-tags दोबारा नहीं बनाता।
- संस्करण पुनर्स्थापन `POST /api/v1/packages/{name}/versions/{version}/restore` को कॉल करता है।
- फ़्लैग:
  - `--version <version>`: स्वामी द्वारा वापस ली गई किसी एक रिलीज़ को पुनर्स्थापित करें।
  - `--yes`: पुष्टि को छोड़ दें।
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package undelete @openclaw/example-plugin --yes
```

### `package transfer <name>`

- किसी पैकेज को दूसरे प्रकाशक को स्थानांतरित करता है।
- वर्तमान पैकेज स्वामी और गंतव्य प्रकाशक, दोनों के लिए व्यवस्थापक पहुँच
  आवश्यक है, जब तक कि यह कार्य किसी प्लेटफ़ॉर्म व्यवस्थापक द्वारा न किया जाए।
- स्कोप किए गए पैकेज नामों को मेल खाने वाले स्कोप स्वामी को ही स्थानांतरित किया जाना चाहिए।
- `POST /api/v1/packages/{name}/transfer` को कॉल करता है।
- फ़्लैग:
  - `--to <owner>`: गंतव्य प्रकाशक हैंडल।
  - `--reason <text>`: वैकल्पिक ऑडिट कारण।
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package transfer @openclaw/example-plugin --to openclaw
```

### `package report`

- किसी पैकेज की मॉडरेटरों को रिपोर्ट करने के लिए प्रमाणीकृत कमांड।
- `POST /api/v1/packages/{name}/report` को कॉल करता है।
- रिपोर्ट पैकेज-स्तरीय होती हैं, वैकल्पिक रूप से किसी संस्करण से संबद्ध की जा सकती हैं, और समीक्षा के लिए
  मॉडरेटरों को दिखाई देने लगती हैं।
- रिपोर्ट अपने-आप पैकेज नहीं छिपातीं या डाउनलोड अवरुद्ध नहीं करतीं।
- फ़्लैग:
  - `--version <version>`: रिपोर्ट से संबद्ध करने के लिए वैकल्पिक पैकेज संस्करण।
  - `--reason <text>`: आवश्यक रिपोर्ट कारण।
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package report @openclaw/example-plugin --version 1.2.3 --reason "संदिग्ध नेटिव पेलोड"
```

### `package moderation-status`

- पैकेज की मॉडरेशन दृश्यता जाँचने के लिए स्वामी कमांड।
- `GET /api/v1/packages/{name}/moderation` को कॉल करता है।
- वर्तमान पैकेज स्कैन स्थिति, खुली रिपोर्टों की संख्या, नवीनतम रिलीज़ की मैन्युअल
  मॉडरेशन स्थिति, डाउनलोड अवरोध स्थिति और मॉडरेशन कारण दिखाता है।
- फ़्लैग:
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package moderation-status @openclaw/example-plugin
```

### `package readiness <name>`

- जाँचता है कि कोई पैकेज भविष्य में OpenClaw द्वारा उपयोग के लिए तैयार है या नहीं।
- `GET /api/v1/packages/{name}/readiness` को कॉल करता है।
- आधिकारिक स्थिति, ClawPack उपलब्धता, आर्टिफ़ैक्ट डाइजेस्ट,
  स्रोत उद्गम, OpenClaw संगतता, होस्ट लक्ष्य, परिवेश मेटाडेटा
  और स्कैन स्थिति में आने वाली बाधाओं की रिपोर्ट करता है।
- फ़्लैग:
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package readiness @openclaw/example-plugin
```

### `package migration-status <name>`

- ऐसे पैकेज के लिए ऑपरेटर-उन्मुख माइग्रेशन स्थिति दिखाता है, जो किसी
  बंडल किए गए OpenClaw Plugin को प्रतिस्थापित कर सकता है।
- `package readiness` वाले समान परिकलित तत्परता एंडपॉइंट को कॉल करता है, लेकिन
  माइग्रेशन-केंद्रित स्थिति, नवीनतम संस्करण, आधिकारिक-पैकेज स्थिति, जाँचें और
  बाधाएँ प्रिंट करता है।
- फ़्लैग:
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package migration-status @openclaw/example-plugin
```

### `publisher create <handle>`

- प्रमाणीकृत उपयोगकर्ता के स्वामित्व वाला संगठन प्रकाशक बनाता है।
- हैंडल को छोटे अक्षरों में सामान्यीकृत किया जाता है और इसे `@` के साथ या उसके बिना दिया जा सकता है।
- नए बनाए गए संगठन प्रकाशक डिफ़ॉल्ट रूप से विश्वसनीय/आधिकारिक नहीं होते।
- यदि हैंडल का उपयोग पहले से किसी मौजूदा प्रकाशक, उपयोगकर्ता या आरक्षित रूट द्वारा किया जा रहा हो, तो यह विफल हो जाता है।

```bash
clawhub publisher create opik --display-name "Opik"
```

### `package publish <source>`

- `POST /api/v1/packages` के माध्यम से कोड Plugin या बंडल Plugin प्रकाशित करता है।
- `<source>` यह स्वीकार करता है:
  - स्थानीय फ़ोल्डर पथ: `./my-plugin`
  - स्थानीय ClawPack npm-pack टारबॉल: `./my-plugin-1.2.3.tgz`
  - GitHub रेपो: `owner/repo` या `owner/repo@ref`
  - GitHub URL: `https://github.com/owner/repo`
- मेटाडेटा का स्वतः पता `package.json`, `openclaw.plugin.json` और
  वास्तविक OpenClaw बंडल मार्करों, जैसे `.codex-plugin/plugin.json`,
  `.claude-plugin/plugin.json` और `.cursor-plugin/plugin.json`, से लगाया जाता है।
- `.tgz` स्रोतों को ClawPack माना जाता है। CLI सटीक npm-pack
  बाइट अपलोड करता है और निकाली गई `package/` सामग्री का उपयोग केवल सत्यापन और
  मेटाडेटा पूर्व-भरण के लिए करता है।
- कोड-Plugin फ़ोल्डरों को अपलोड से पहले ClawPack npm टारबॉल में पैक किया जाता है, ताकि
  OpenClaw इंस्टॉलेशन सटीक आर्टिफ़ैक्ट सत्यापित कर सकें। बंडल-Plugin फ़ोल्डर अब भी
  निकाली गई फ़ाइलों वाला प्रकाशन पथ उपयोग करते हैं।
- GitHub स्रोतों के लिए, स्रोत श्रेय रेपो, निर्धारित कमिट, रेफ़ और उपपथ से स्वतः भरा जाता है।
- स्थानीय फ़ोल्डरों के लिए, यदि origin रिमोट GitHub की ओर संकेत करता है, तो स्थानीय git से स्रोत श्रेय का स्वतः पता लगाया जाता है।
- बाहरी कोड Plugin को `openclaw.compat.pluginApi` और
  `openclaw.build.openclawVersion` स्पष्ट रूप से घोषित करने चाहिए।
  शीर्ष-स्तरीय `package.json.version` का उपयोग प्रकाशन सत्यापन के लिए फ़ॉलबैक के रूप में नहीं किया जाता।
- `--dry-run` अपलोड किए बिना निर्धारित प्रकाशन पेलोड का पूर्वावलोकन करता है।
- `--json` CI के लिए मशीन-पठनीय आउटपुट देता है।
- जब कर्ता के पास प्रकाशक पहुँच होती है, तो `--owner <handle>` उपयोगकर्ता या संगठन प्रकाशक हैंडल के अंतर्गत प्रकाशित करता है।
- स्कोप किए गए पैकेज नाम चयनित स्वामी से मेल खाने चाहिए। `docs/publishing.md` देखें।
- मौजूदा फ़्लैग (`--family`, `--name`, `--version`, `--source-repo`, `--source-commit`, `--source-ref`, `--source-path`) अब भी ओवरराइड के रूप में काम करते हैं।
- निजी GitHub रेपो के लिए `GITHUB_TOKEN` आवश्यक है।

```bash
clawhub package publish ./plugin.tgz --owner openclaw
```

#### अनुशंसित स्थानीय प्रवाह

पहले `--dry-run` का उपयोग करें, ताकि लाइव रिलीज़ बनाने से पहले निर्धारित पैकेज मेटाडेटा और
स्रोत श्रेय की पुष्टि की जा सके:

```bash
npm pack
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin --dry-run
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin
```

#### स्थानीय फ़ोल्डर प्रवाह

कोड Plugin के लिए, फ़ोल्डर प्रकाशन पैकेज फ़ोल्डर से ClawPack आर्टिफ़ैक्ट बनाकर
उसे अपलोड करता है:

```bash
clawhub package publish ./my-plugin --family code-plugin --dry-run
clawhub package publish ./my-plugin --family code-plugin
```

#### `--family code-plugin` के लिए न्यूनतम `package.json`

बाहरी कोड Plugin को `package.json` में थोड़ी-सी OpenClaw मेटाडेटा की
आवश्यकता होती है। सफल प्रकाशन के लिए यह न्यूनतम मैनिफ़ेस्ट पर्याप्त है:

```json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2"
    }
  }
}
```

आवश्यक फ़ील्ड:

- `openclaw.compat.pluginApi`
- `openclaw.build.openclawVersion`

टिप्पणियाँ:

- `package.json.version` आपके पैकेज रिलीज़ का संस्करण है, लेकिन इसका उपयोग
  OpenClaw संगतता/बिल्ड सत्यापन के लिए फ़ॉलबैक के रूप में नहीं किया जाता।
- `openclaw.hostTargets` और `openclaw.environment` वैकल्पिक मेटाडेटा हैं।
  मौजूद होने पर ClawHub इन्हें दिखा सकता है, लेकिन प्रकाशन के लिए ये आवश्यक नहीं हैं।
- `openclaw.compat.minGatewayVersion` और
  `openclaw.build.pluginSdkVersion` वैकल्पिक अतिरिक्त फ़ील्ड हैं, यदि आप
  अधिक विस्तृत संगतता मेटाडेटा प्रकाशित करना चाहते हैं।
- यदि आप `clawhub` CLI का कोई पुराना रिलीज़ उपयोग कर रहे हैं, तो प्रकाशन से पहले अपग्रेड करें, ताकि
  स्थानीय प्रीफ़्लाइट जाँचें अपलोड से पहले चलें।
- यदि सत्यापन कोई सुधार कोड रिपोर्ट करता है, तो
  [Plugin सत्यापन सुधार](/hi/clawhub/plugin-validation-fixes) देखें।

#### GitHub Actions

ClawHub, Plugin रेपो के लिए
[`/.github/workflows/package-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/package-publish.yml)
पर एक आधिकारिक पुनःप्रयोग योग्य वर्कफ़्लो भी उपलब्ध कराता है।

सामान्य कॉलर सेटअप:

```yaml
name: Package Publish

on:
  pull_request:
  workflow_dispatch:
  push:
    tags:
      - "v*"

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch' || startsWith(github.ref, 'refs/tags/')
    permissions:
      contents: read
      id-token: write
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

टिप्पणियाँ:

- पुनःप्रयोग योग्य वर्कफ़्लो में `source` डिफ़ॉल्ट रूप से कॉलर रेपो होता है।
- मोनोरेपो के लिए `source_path` दें, ताकि वर्कफ़्लो Plugin
  पैकेज फ़ोल्डर प्रकाशित करे, उदाहरण के लिए `source_path: extensions/codex`।
- पुनःप्रयोग योग्य वर्कफ़्लो को किसी स्थिर टैग या पूर्ण कमिट SHA पर पिन करें। `@main` से रिलीज़ प्रकाशन न चलाएँ।
- `pull_request` में `dry_run: true` का उपयोग होना चाहिए, ताकि CI में अनावश्यक बदलाव न हों।
- वास्तविक प्रकाशन को `workflow_dispatch` या टैग पुश जैसे विश्वसनीय इवेंट तक सीमित रखा जाना चाहिए।
- बिना सीक्रेट के विश्वसनीय प्रकाशन केवल `workflow_dispatch` पर काम करता है; टैग पुश के लिए अब भी `clawhub_token` आवश्यक है।
- पहले प्रकाशन, अविश्वसनीय पैकेज या आपातकालीन प्रकाशन के लिए `clawhub_token` उपलब्ध रखें।
- वर्कफ़्लो JSON परिणाम को आर्टिफ़ैक्ट के रूप में अपलोड करता है और उसे वर्कफ़्लो आउटपुट के रूप में उपलब्ध कराता है।

### `package trusted-publisher get <name>`

- किसी पैकेज के लिए GitHub Actions विश्वसनीय प्रकाशक कॉन्फ़िगरेशन दिखाता है।
- कॉन्फ़िगरेशन सेट करने के बाद रेपॉज़िटरी, वर्कफ़्लो फ़ाइल नाम
  और वैकल्पिक परिवेश पिन की पुष्टि करने के लिए इसका उपयोग करें।
- फ़्लैग:
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package trusted-publisher get @openclaw/example-plugin
```

### `package trusted-publisher set <name>`

- किसी मौजूदा पैकेज के लिए GitHub Actions विश्वसनीय प्रकाशक कॉन्फ़िगरेशन जोड़ता
  या प्रतिस्थापित करता है।
- पैकेज को पहले सामान्य मैन्युअल या टोकन-प्रमाणीकृत
  `clawhub package publish` के माध्यम से बनाया जाना चाहिए।
- कॉन्फ़िगरेशन सेट होने के बाद, भविष्य के समर्थित GitHub Actions प्रकाशन
  दीर्घकालिक ClawHub टोकन के बिना OIDC/विश्वसनीय प्रकाशन का उपयोग कर सकते हैं।
- `--repository <repo>` को `owner/repo` होना चाहिए।
- `--workflow-filename <file>` को
  `.github/workflows/` में मौजूद वर्कफ़्लो फ़ाइल नाम से मेल खाना चाहिए।
- `--environment <name>` वैकल्पिक है। कॉन्फ़िगर किए जाने पर, OIDC क्लेम में
  GitHub Actions परिवेश बिल्कुल मेल खाना चाहिए।
- यह कमांड चलने पर ClawHub कॉन्फ़िगर की गई GitHub रेपॉज़िटरी सत्यापित करता है।
  सार्वजनिक रेपॉज़िटरी को सार्वजनिक GitHub मेटाडेटा के माध्यम से सत्यापित किया जा सकता है। निजी
  रेपॉज़िटरी के लिए ClawHub के पास उस रेपॉज़िटरी की GitHub पहुँच होनी चाहिए,
  उदाहरण के लिए भविष्य के ClawHub GitHub App इंस्टॉलेशन या किसी अन्य अधिकृत
  GitHub एकीकरण के माध्यम से।
- फ़्लैग:
  - `--repository <repo>`: GitHub रेपॉज़िटरी, उदाहरण के लिए `openclaw/example-plugin`।
  - `--workflow-filename <file>`: वर्कफ़्लो फ़ाइल नाम, उदाहरण के लिए `package-publish.yml`।
  - `--environment <name>`: वैकल्पिक सटीक-मिलान वाला GitHub Actions परिवेश।
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package trusted-publisher set @openclaw/example-plugin \
  --repository openclaw/example-plugin \
  --workflow-filename package-publish.yml \
  --environment release
```

### `package trusted-publisher delete <name>`

- किसी पैकेज से विश्वसनीय प्रकाशक कॉन्फ़िगरेशन हटाता है।
- यदि वर्कफ़्लो, रेपॉज़िटरी या परिवेश पिन को अक्षम या फिर से बनाने की
  आवश्यकता हो, तो इसे रोलबैक के रूप में उपयोग करें।
- कॉन्फ़िगरेशन फिर से सेट होने तक भविष्य के वास्तविक प्रकाशनों को सामान्य प्रमाणीकृत प्रकाशन का
  उपयोग करना होगा।
- फ़्लैग:
  - `--json`: मशीन-पठनीय आउटपुट।

उदाहरण:

```bash
clawhub package trusted-publisher delete @openclaw/example-plugin
```

### इंस्टॉल टेलीमेट्री

- लॉग इन होने पर `clawhub install <slug>` के बाद भेजी जाती है, जब तक कि
  `CLAWHUB_DISABLE_TELEMETRY=1` सेट न हो।
- रिपोर्टिंग सर्वोत्तम प्रयास के आधार पर होती है। टेलीमेट्री अनुपलब्ध होने पर इंस्टॉल कमांड
  विफल नहीं होते।
- विवरण: `docs/telemetry.md`।
