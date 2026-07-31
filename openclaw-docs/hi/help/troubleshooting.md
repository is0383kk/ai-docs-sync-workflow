---
read_when:
    - OpenClaw काम नहीं कर रहा है और आपको इसे ठीक करने का सबसे तेज़ तरीका चाहिए
    - गहन रनबुक में जाने से पहले आपको एक ट्रायेज प्रवाह चाहिए
summary: OpenClaw के लिए लक्षण-आधारित समस्या निवारण केंद्र
title: सामान्य समस्या निवारण
x-i18n:
    generated_at: "2026-07-27T19:59:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de3554ed680ac536d105017220b44d94456a4408916e949352500b046f4d5f17
    source_path: help/troubleshooting.md
    workflow: 16
---

ट्रायेज का शुरुआती बिंदु। 2 मिनट में निदान करें, फिर विस्तृत पृष्ठ पर जाएँ।

## पहले 60 सेकंड

इस क्रम में ये चरण चलाएँ:

```bash
openclaw status
openclaw status --all
openclaw gateway probe
openclaw gateway status
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

सही आउटपुट, प्रत्येक के लिए एक पंक्ति:

- `openclaw status` कॉन्फ़िगर किए गए चैनल दिखाता है, कोई प्रमाणीकरण त्रुटि नहीं।
- `openclaw status --all` एक पूर्ण, साझा करने योग्य रिपोर्ट तैयार करता है।
- `openclaw gateway probe` में `Reachable: yes` दिखता है। `Capability: ...` वह
  प्रमाणीकरण स्तर है जिसे प्रोब ने प्रमाणित किया; `Read probe: limited - missing scope:
operator.read` निम्नीकृत निदान है, कनेक्शन विफलता नहीं।
- `openclaw gateway status` में `Runtime: running`, `Connectivity probe:
ok`, और एक विश्वसनीय `Capability: ...` दिखता है। रीड-स्कोप RPC प्रमाण भी आवश्यक करने के लिए
  `--require-rpc` जोड़ें।
- `openclaw doctor` कोई अवरोधक कॉन्फ़िगरेशन/सेवा त्रुटि रिपोर्ट नहीं करता।
- `openclaw channels status --probe` Gateway उपलब्ध होने पर प्रत्येक खाते की लाइव ट्रांसपोर्ट स्थिति
  (`works` / `audit ok`) लौटाता है; अनुपलब्ध होने पर
  केवल-कॉन्फ़िगरेशन सारांश पर वापस चला जाता है।
- `openclaw logs --follow` स्थिर गतिविधि दिखाता है, कोई बार-बार आने वाली गंभीर त्रुटि नहीं।

## सहायक सीमित लगता है या टूल अनुपलब्ध हैं

प्रभावी टूल प्रोफ़ाइल जाँचें:

```bash
openclaw status
openclaw status --all
openclaw doctor
```

सामान्य कारण:

- `tools.profile: "minimal"` केवल `session_status` की अनुमति देता है।
- `tools.profile: "messaging"` सीमित है और केवल-चैट एजेंटों के लिए है।
- `tools.profile: "coding"` नए स्थानीय कॉन्फ़िगरेशन के लिए डिफ़ॉल्ट है (रिपॉज़िटरी, फ़ाइल,
  शेल और रनटाइम कार्य)।
- `tools.profile: "full"` प्रोफ़ाइल प्रतिबंध हटा देता है; इसे विश्वसनीय
  ऑपरेटर-नियंत्रित एजेंटों तक सीमित रखें।
- प्रति-एजेंट `agents.entries.*.tools` एक एजेंट के लिए रूट प्रोफ़ाइल को सीमित या विस्तृत करते हैं।

प्रोफ़ाइल बदलें, Gateway पुनः आरंभ या रीलोड करें, फिर
`openclaw status --all` से दोबारा जाँचें। पूरी प्रोफ़ाइल/समूह तालिका: [टूल प्रोफ़ाइल](/hi/gateway/config-tools#tool-profiles)।

## Anthropic दीर्घ कॉन्टेक्स्ट 429

`HTTP 429: rate_limit_error: Extra usage is required for long context requests`
→ [दीर्घ कॉन्टेक्स्ट के लिए Anthropic 429 में अतिरिक्त उपयोग आवश्यक](/hi/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context)।

## स्थानीय OpenAI-संगत बैकएंड सीधे काम करता है, लेकिन OpenClaw में विफल होता है

आपका स्थानीय/स्व-होस्टेड `/v1` बैकएंड सीधे `/v1/chat/completions`
प्रोब का उत्तर देता है, लेकिन `openclaw infer model run` या सामान्य एजेंट टर्न पर विफल होता है:

1. त्रुटि में स्ट्रिंग की अपेक्षा करने वाले `messages[].content` का उल्लेख है: 
   `models.providers.<provider>.models[].compat.requiresStringContent: true` सेट करें।
2. अब भी केवल OpenClaw एजेंट टर्न पर विफल होता है:
   `models.providers.<provider>.models[].compat.supportsTools: false` सेट करके पुनः प्रयास करें।
3. छोटी सीधी कॉल काम करती हैं, लेकिन बड़े OpenClaw प्रॉम्प्ट बैकएंड को क्रैश कर देते हैं: यह
   अपस्ट्रीम मॉडल/सर्वर सीमा है, OpenClaw बग नहीं। आगे के लिए
   [स्थानीय OpenAI-संगत बैकएंड सीधे प्रोब पास करता है, लेकिन एजेंट रन विफल होते हैं](/hi/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail) देखें।

## OpenClaw एक्सटेंशन अनुपलब्ध होने के कारण Plugin इंस्टॉल विफल होता है

`package.json missing openclaw.extensions` का अर्थ है कि Plugin पैकेज ऐसी
संरचना का उपयोग करता है जिसे OpenClaw अब स्वीकार नहीं करता।

Plugin पैकेज में सुधार करें:

1. `package.json` में `openclaw.extensions` जोड़ें, जो निर्मित रनटाइम
   फ़ाइलों (आमतौर पर `./dist/index.js`) की ओर संकेत करे।
2. दोबारा प्रकाशित करें, फिर `openclaw plugins install <package>` पुनः चलाएँ।

```json
{
  "name": "@openclaw/my-plugin",
  "version": "1.2.3",
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

संदर्भ: [Plugin आर्किटेक्चर](/hi/plugins/architecture)

## इंस्टॉल नीति Plugin इंस्टॉल या अपडेट को अवरुद्ध करती है

अपडेट पूरा हो जाता है, लेकिन Plugin पुराने या अक्षम रहते हैं अथवा `blocked by install
policy`, `install policy failed closed`, या `Disabled "<plugin>" after plugin
update failure` दिखाते हैं: `security.installPolicy` जाँचें।

इंस्टॉल नीति Plugin इंस्टॉल और अपडेट पर चलती है। `@openclaw/*` Plugin
संस्करण सामान्यतः OpenClaw रिलीज़ के साथ बदलते हैं, इसलिए OpenClaw अपडेट के लिए
अपडेट-पश्चात सिंक के दौरान संबंधित Plugin अपडेट आवश्यक हो सकता है।

इन नीति संरचनाओं से बचें, जब तक कि आप संबंधित अपग्रेड नियम भी बनाए न रखें:

- OpenClaw-स्वामित्व वाले Plugin को किसी एक सटीक पुराने संस्करण पर स्थिर करना (उदाहरण के लिए, केवल
  `@openclaw/*@2026.5.3`)।
- केवल स्रोत प्रकार के आधार पर अवरुद्ध करना (प्रत्येक npm, नेटवर्क, या `request.mode:
"update"` अनुरोध)।
- नीति कमांड को वैकल्पिक मानना: जब `security.installPolicy`
  सक्षम हो, तो अनुपलब्ध, धीमा, अपठनीय या अनुमति द्वारा अवरुद्ध नीति
  निष्पादन योग्य फ़ाइल सुरक्षित रूप से विफल होकर पहुँच रोक देती है।
- Plugin प्रत्याशी मेटाडेटा के विरुद्ध अनुरोध का `openclawVersion` जाँचे बिना
  संस्करणों को स्वीकृत करना।

किसी एक रिलीज़ को हमेशा के लिए पिन करने के बजाय ऐसे नियमों को प्राथमिकता दें जो
वर्तमान होस्ट के साथ संगत विश्वसनीय `@openclaw/*` अपडेट की अनुमति देते हों। यदि आप डिफ़ॉल्ट रूप से npm
को अवरुद्ध करते हैं, तो अपने उपयोग वाले Plugin आईडी के लिए सीमित अपवाद जोड़ें और
`request.mode: "update"` पर भी वही विश्वास नियम लागू करें जो इंस्टॉल पर लागू होता है।

पुनर्प्राप्ति:

```bash
openclaw doctor --deep
openclaw plugins update --all
openclaw status --all
```

यदि नीति जानबूझकर सख्त है, तो विश्वसनीय अपग्रेड
अवधि के लिए इसे शिथिल करें, `openclaw plugins update --all` फिर से चलाएँ, और फिर अधिक सख्त नियम पुनर्स्थापित करें।
यदि अपडेट विफलता ने किसी Plugin को अक्षम कर दिया है, तो उसे पुनः सक्षम करने से पहले निरीक्षण करें:

```bash
openclaw plugins inspect <plugin-id> --runtime --json
openclaw plugins enable <plugin-id>
```

संदर्भ: [ऑपरेटर इंस्टॉल नीति](/hi/tools/skills-config#operator-install-policy-securityinstallpolicy)

## Plugin मौजूद है, लेकिन संदिग्ध स्वामित्व के कारण अवरुद्ध है

`openclaw doctor`, सेटअप या स्टार्टअप चेतावनियाँ दिखाती हैं:

```text
अवरुद्ध Plugin प्रत्याशी: संदिग्ध स्वामित्व (... uid=1000, अपेक्षित uid=0 या root)
Plugin मौजूद है, लेकिन अवरुद्ध है
```

Plugin फ़ाइलों का स्वामी उन्हें लोड करने वाली प्रक्रिया से अलग Unix उपयोगकर्ता है।
Plugin कॉन्फ़िगरेशन न हटाएँ; फ़ाइल स्वामित्व ठीक करें या OpenClaw को
स्टेट डायरेक्टरी के स्वामी उपयोगकर्ता के रूप में चलाएँ।

Docker इंस्टॉल `node` (uid `1000`) के रूप में चलते हैं। होस्ट बाइंड माउंट सुधारें:

```bash
sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
openclaw doctor --fix
```

यदि आप जानबूझकर OpenClaw को root के रूप में चलाते हैं, तो इसके बजाय प्रबंधित Plugin रूट
सुधारें:

```bash
sudo chown -R root:root /path/to/openclaw-config/npm
openclaw doctor --fix
```

विस्तृत दस्तावेज़: [अवरुद्ध Plugin पथ का स्वामित्व](/hi/tools/plugin#blocked-plugin-path-ownership), [Docker: अनुमतियाँ और EACCES](/hi/install/docker#shell-helpers-optional)

## निर्णय-वृक्ष

```mermaid
flowchart TD
  A[OpenClaw काम नहीं कर रहा है] --> B{सबसे पहले क्या विफल होता है}
  B --> C[कोई उत्तर नहीं]
  B --> D[डैशबोर्ड या नियंत्रण UI कनेक्ट नहीं होता]
  B --> E[Gateway आरंभ नहीं होता या सेवा नहीं चल रही]
  B --> F[चैनल कनेक्ट होता है, लेकिन संदेश प्रवाहित नहीं होते]
  B --> G[Cron या Heartbeat सक्रिय नहीं हुआ या वितरित नहीं हुआ]
  B --> H[Node युग्मित है, लेकिन कैमरा कैनवस स्क्रीन निष्पादन विफल होता है]
  B --> I[ब्राउज़र टूल विफल होता है]

  C --> C1[/कोई उत्तर नहीं अनुभाग/]
  D --> D1[/नियंत्रण UI अनुभाग/]
  E --> E1[/Gateway अनुभाग/]
  F --> F1[/चैनल प्रवाह अनुभाग/]
  G --> G1[/स्वचालन अनुभाग/]
  H --> H1[/Node टूल अनुभाग/]
  I --> I1[/ब्राउज़र अनुभाग/]
```

<AccordionGroup>
  <Accordion title="कोई उत्तर नहीं">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw channels status --probe
    openclaw pairing list --channel <channel> [--account <id>]
    openclaw logs --follow
    ```

    सही आउटपुट:

    - `Runtime: running`
    - `Connectivity probe: ok`
    - `Capability: read-only`, `write-capable`, या `admin-capable`
    - चैनल में ट्रांसपोर्ट कनेक्टेड और, जहाँ समर्थित हो, `channels status --probe` में `works` या
      `audit ok` दिखता है
    - प्रेषक स्वीकृत है (या DM नीति खुली/अनुमति-सूची वाली है)

    लॉग संकेत:

    - `drop guild message (mention required` → Discord उल्लेख नियंत्रण ने संदेश अवरुद्ध किया।
    - `pairing request` → प्रेषक अस्वीकृत है, DM युग्मन स्वीकृति की प्रतीक्षा है।
    - चैनल लॉग में `blocked` / `allowlist` → प्रेषक, कक्ष या समूह फ़िल्टर किया गया।

    विस्तृत पृष्ठ: [कोई उत्तर नहीं](/hi/gateway/troubleshooting#no-replies), [चैनल समस्या निवारण](/hi/channels/troubleshooting), [युग्मन](/hi/channels/pairing)

  </Accordion>

  <Accordion title="डैशबोर्ड या नियंत्रण UI कनेक्ट नहीं होता">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw logs --follow
    openclaw doctor
    openclaw channels status --probe
    ```

    सही आउटपुट:

    - `openclaw gateway status` में `Dashboard: http://...` दिखता है
    - `Connectivity probe: ok`
    - `Capability: read-only`, `write-capable`, या `admin-capable`
    - लॉग में कोई प्रमाणीकरण लूप नहीं

    लॉग संकेत:

    - `device identity required` → HTTP/असुरक्षित संदर्भ डिवाइस प्रमाणीकरण पूरा नहीं कर सकता।
    - `origin not allowed` → नियंत्रण UI के Gateway लक्ष्य के लिए ब्राउज़र `Origin` अनुमत नहीं है।
    - `canRetryWithDeviceToken=true` के साथ `AUTH_TOKEN_MISMATCH` → युग्मित टोकन के कैश किए गए स्कोप का पुनः उपयोग करते हुए, एक विश्वसनीय डिवाइस-टोकन पुनः प्रयास अपने आप हो सकता है।
    - उस पुनः प्रयास के बाद बार-बार `unauthorized` → गलत टोकन/पासवर्ड, प्रमाणीकरण मोड असंगति या पुराना युग्मित डिवाइस टोकन।
    - `too many failed authentication attempts (retry later)` → उस ब्राउज़र `Origin` से बार-बार होने वाली विफलताएँ अस्थायी रूप से लॉक कर दी गई हैं; अन्य localhost मूल अलग बकेट का उपयोग करते हैं। Tailscale Serve के समवर्ती-पुनः प्रयास संबंधी सूक्ष्म अंतर के लिए [डैशबोर्ड/नियंत्रण UI कनेक्टिविटी](/hi/gateway/troubleshooting#dashboard-control-ui-connectivity) देखें।
    - `gateway connect failed:` → UI गलत URL/पोर्ट को लक्षित कर रहा है या Gateway उपलब्ध नहीं है।

    विस्तृत पृष्ठ: [डैशबोर्ड/नियंत्रण UI कनेक्टिविटी](/hi/gateway/troubleshooting#dashboard-control-ui-connectivity), [नियंत्रण UI](/hi/web/control-ui), [प्रमाणीकरण](/hi/gateway/authentication)

  </Accordion>

  <Accordion title="Gateway आरंभ नहीं होता या सेवा इंस्टॉल है लेकिन चल नहीं रही">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw logs --follow
    openclaw doctor
    openclaw channels status --probe
    ```

    सही आउटपुट:

    - `Service: ... (loaded)`
    - `Runtime: running`
    - `Connectivity probe: ok`
    - `Capability: read-only`, `write-capable`, या `admin-capable`

    लॉग संकेत:

    - `Gateway start blocked: set gateway.mode=local` या `existing config is missing gateway.mode` → Gateway मोड रिमोट है या कॉन्फ़िगरेशन में स्थानीय-मोड चिह्न अनुपलब्ध है और उसे सुधारना आवश्यक है।
    - `refusing to bind gateway ... without auth` → वैध प्रमाणीकरण पथ के बिना गैर-लूपबैक बाइंड (टोकन/पासवर्ड या जहाँ कॉन्फ़िगर हो वहाँ विश्वसनीय प्रॉक्सी)।
    - `another gateway instance is already listening` या `EADDRINUSE` → पोर्ट पहले से उपयोग में है।

    विस्तृत पृष्ठ: [Gateway सेवा नहीं चल रही](/hi/gateway/troubleshooting#gateway-service-not-running), [पृष्ठभूमि प्रक्रिया](/hi/gateway/background-process), [कॉन्फ़िगरेशन](/hi/gateway/configuration)

  </Accordion>

  <Accordion title="चैनल कनेक्ट होता है, लेकिन संदेश प्रवाहित नहीं होते">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw logs --follow
    openclaw doctor
    openclaw channels status --probe
    ```

    सही आउटपुट:

    - चैनल ट्रांसपोर्ट कनेक्टेड है।
    - युग्मन/अनुमति-सूची जाँच सफल होती हैं।
    - जहाँ आवश्यक हो, उल्लेख पहचाने जाते हैं।

    लॉग संकेत:

    - `mention required` → समूह उल्लेख नियंत्रण ने प्रसंस्करण अवरुद्ध किया।
    - `pairing` / `pending` → DM प्रेषक अभी स्वीकृत नहीं है।
    - `not_in_channel`, `missing_scope`, `Forbidden`, `401/403` → चैनल अनुमति टोकन की समस्या।

    विस्तृत पृष्ठ: [चैनल कनेक्टेड है, संदेश प्रवाहित नहीं हो रहे](/hi/gateway/troubleshooting#channel-connected-messages-not-flowing), [चैनल समस्या निवारण](/hi/channels/troubleshooting)

  </Accordion>

  <Accordion title="Cron या Heartbeat सक्रिय नहीं हुआ या वितरित नहीं हुआ">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw cron status
    openclaw cron list
    openclaw cron runs --id <jobId> --limit 20
    openclaw logs --follow
    ```

    सही आउटपुट:

    - `cron status` शेड्यूलर को अगली सक्रियता के साथ सक्षम दिखाता है।
    - `cron runs` हाल की `ok` प्रविष्टियाँ दिखाता है।
    - Heartbeat सक्षम है और सक्रिय घंटों के भीतर है।

    लॉग संकेत:

    - `cron: scheduler disabled; jobs will not run automatically` → Cron अक्षम है।
    - `heartbeat skipped` कारण `quiet-hours` → कॉन्फ़िगर किए गए सक्रिय घंटों के बाहर।
    - `heartbeat skipped` कारण `empty-heartbeat-file` → Heartbeat मॉनिटर स्क्रैच में केवल रिक्त स्थान, टिप्पणी, हेडर, फ़ेंस या खाली-चेकलिस्ट स्कैफ़ोल्डिंग है।
    - `heartbeat skipped` कारण `alerts-disabled` → `showOk`, `showAlerts`, और `useIndicator` सभी बंद हैं।
    - `requests-in-flight` → मुख्य लेन व्यस्त है; Heartbeat सक्रियता स्थगित की गई।
    - `unknown accountId` → Heartbeat डिलीवरी लक्ष्य खाता मौजूद नहीं है।

    विस्तृत पृष्ठ: [Cron और Heartbeat डिलीवरी](/hi/gateway/troubleshooting#cron-and-heartbeat-delivery), [शेड्यूल किए गए कार्य: समस्या निवारण](/hi/automation/cron-jobs#troubleshooting), [Heartbeat](/hi/gateway/heartbeat)

  </Accordion>

  <Accordion title="Node युग्मित है, लेकिन टूल कैमरा कैनवस स्क्रीन exec में विफल होता है">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw nodes status
    openclaw nodes describe --node <idOrNameOrIp>
    openclaw logs --follow
    ```

    सही आउटपुट:

    - Node को भूमिका `node` के लिए कनेक्टेड और युग्मित के रूप में सूचीबद्ध किया गया है।
    - आप जिस कमांड का आह्वान कर रहे हैं, उसके लिए क्षमता मौजूद है।
    - टूल के लिए अनुमति की स्थिति स्वीकृत है।

    लॉग संकेत:

    - `NODE_BACKGROUND_UNAVAILABLE` → Node ऐप को अग्रभूमि में लाएँ।
    - `*_PERMISSION_REQUIRED` → OS अनुमति अस्वीकृत है या उपलब्ध नहीं है।
    - `SYSTEM_RUN_DENIED: approval required` → exec की स्वीकृति लंबित है।
    - `SYSTEM_RUN_DENIED: allowlist miss` → कमांड exec अनुमति-सूची में नहीं है।

    विस्तृत पृष्ठ: [Node युग्मित है, टूल विफल होता है](/hi/gateway/troubleshooting#node-paired-tool-fails), [Node समस्या निवारण](/hi/nodes/troubleshooting), [Exec स्वीकृतियाँ](/hi/tools/exec-approvals)

  </Accordion>

  <Accordion title="Exec अचानक स्वीकृति माँगता है">
    ```bash
    openclaw config get tools.exec.host
    openclaw config get tools.exec.security
    openclaw config get tools.exec.ask
    openclaw gateway restart
    ```

    क्या बदला:

    - असेट न किया गया `tools.exec.host`, डिफ़ॉल्ट रूप से `auto` होता है, जो सैंडबॉक्स रनटाइम सक्रिय होने पर
      `sandbox` में, अन्यथा `gateway` में रिज़ॉल्व होता है।
    - `host=auto` केवल रूटिंग करता है; बिना प्रॉम्प्ट वाला व्यवहार
      Gateway/Node पर `security=full` और `ask=off` से आता है।
    - असेट न किया गया `tools.exec.security`, `gateway`/`node` पर डिफ़ॉल्ट रूप से `full` होता है।
    - असेट न किया गया `tools.exec.ask`, डिफ़ॉल्ट रूप से `off` होता है।
    - यदि आपको स्वीकृतियाँ दिखाई दे रही हैं, तो किसी होस्ट-स्थानीय या प्रति-सत्र नीति ने
      exec को इन डिफ़ॉल्ट से अधिक सख्त कर दिया है।

    बिना स्वीकृति वाले वर्तमान डिफ़ॉल्ट पुनर्स्थापित करें:

    ```bash
    openclaw config set tools.exec.host gateway
    openclaw config set tools.exec.security full
    openclaw config set tools.exec.ask off
    openclaw gateway restart
    ```

    अधिक सुरक्षित विकल्प:

    - स्थिर होस्ट रूटिंग के लिए केवल `tools.exec.host=gateway` सेट करें।
    - अनुमति-सूची से चूक होने पर समीक्षा सहित होस्ट exec के लिए `security=allowlist` के साथ
      `ask=on-miss` का उपयोग करें।
    - सैंडबॉक्स मोड सक्षम करें, ताकि `host=auto` फिर से `sandbox` में रिज़ॉल्व हो।

    लॉग संकेत:

    - `Approval required.` → कमांड `/approve ...` की प्रतीक्षा कर रहा है।
    - `SYSTEM_RUN_DENIED: approval required` → Node-होस्ट exec स्वीकृति लंबित है।
    - `exec host=sandbox requires a sandbox runtime for this session` → अंतर्निहित/स्पष्ट सैंडबॉक्स चयन हुआ है, लेकिन सैंडबॉक्स मोड बंद है।

    विस्तृत पृष्ठ: [Exec](/hi/tools/exec), [Exec स्वीकृतियाँ](/hi/tools/exec-approvals), [सुरक्षा: ऑडिट क्या जाँचता है](/hi/gateway/security#what-the-audit-checks-high-level)

  </Accordion>

  <Accordion title="ब्राउज़र टूल विफल होता है">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw browser status
    openclaw logs --follow
    openclaw doctor
    ```

    सही आउटपुट:

    - ब्राउज़र स्थिति `running: true` और चुना गया ब्राउज़र/प्रोफ़ाइल दिखाती है।
    - `openclaw` प्रोफ़ाइल शुरू होती है, या `user` प्रोफ़ाइल स्थानीय Chrome टैब देखती है।

    लॉग संकेत:

    - `unknown command "browser"` → `plugins.allow` सेट है और `browser` को बाहर रखता है।
    - `Failed to start Chrome CDP on port` → स्थानीय ब्राउज़र लॉन्च विफल हुआ।
    - `browser.executablePath not found` → कॉन्फ़िगर किया गया बाइनरी पथ गलत है।
    - `browser.cdpUrl must be http(s) or ws(s)` → कॉन्फ़िगर किया गया CDP URL एक असमर्थित स्कीम का उपयोग करता है।
    - `browser.cdpUrl has invalid port` → कॉन्फ़िगर किए गए CDP URL में अमान्य या सीमा से बाहर का पोर्ट है।
    - `No Chrome tabs found for profile="user"` → Chrome MCP अटैच प्रोफ़ाइल में कोई स्थानीय Chrome टैब खुला नहीं है।
    - `Remote CDP for profile "<name>" is not reachable` → कॉन्फ़िगर किया गया रिमोट CDP एंडपॉइंट इस होस्ट से पहुँच योग्य नहीं है।
    - `Browser attachOnly is enabled ... not reachable` → केवल-अटैच प्रोफ़ाइल में कोई सक्रिय CDP लक्ष्य नहीं है।
    - केवल-अटैच या रिमोट CDP प्रोफ़ाइल पर पुराने व्यूपोर्ट/डार्क-मोड/लोकेल/ऑफ़लाइन ओवरराइड → Gateway को पुनः आरंभ किए बिना नियंत्रण सत्र बंद करने और इम्यूलेशन स्थिति मुक्त करने के लिए `openclaw browser stop --browser-profile <name>` चलाएँ।

    विस्तृत पृष्ठ: [ब्राउज़र टूल विफल होता है](/hi/gateway/troubleshooting#browser-tool-fails), [ब्राउज़र कमांड या टूल उपलब्ध नहीं है](/hi/tools/browser#missing-browser-command-or-tool), [ब्राउज़र: Linux समस्या निवारण](/hi/tools/browser-linux-troubleshooting), [ब्राउज़र: WSL2/Windows रिमोट CDP समस्या निवारण](/hi/tools/browser-wsl2-windows-remote-cdp-troubleshooting)

  </Accordion>

</AccordionGroup>

## संबंधित

- [अक्सर पूछे जाने वाले प्रश्न](/hi/help/faq) — अक्सर पूछे जाने वाले प्रश्न
- [Gateway समस्या निवारण](/hi/gateway/troubleshooting) — Gateway-विशिष्ट समस्याएँ
- [Doctor](/hi/gateway/doctor) — स्वचालित स्वास्थ्य जाँच और सुधार
- [चैनल समस्या निवारण](/hi/channels/troubleshooting) — चैनल कनेक्टिविटी समस्याएँ
- [शेड्यूल किए गए कार्य: समस्या निवारण](/hi/automation/cron-jobs#troubleshooting) — Cron और Heartbeat संबंधी समस्याएँ
