---
read_when:
    - आप एजेंट हुक प्रबंधित करना चाहते हैं
    - आप हुक की उपलब्धता जाँचना या वर्कस्पेस हुक सक्षम करना चाहते हैं
summary: '`openclaw hooks` (एजेंट हुक्स) के लिए CLI संदर्भ'
title: हुक्स
x-i18n:
    generated_at: "2026-07-27T19:30:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d4d58ea2270cf5122018f7be2943401229929f48f448b15fdd126d1cc99e1e56
    source_path: cli/hooks.md
    workflow: 16
---

# `openclaw hooks`

एजेंट हुक प्रबंधित करें (इवेंट-संचालित स्वचालन, जैसे `/new`, `/reset`, और Gateway स्टार्टअप जैसे कमांड के लिए)। केवल `openclaw hooks` देना `openclaw hooks list` के समतुल्य है।

संबंधित: [हुक](/hi/automation/hooks) - [Plugin हुक](/hi/plugins/hooks)

## हुक सूचीबद्ध करें

```bash
openclaw hooks list [--eligible] [--json] [-v|--verbose]
```

वर्कस्पेस, प्रबंधित, अतिरिक्त और बंडल की गई डायरेक्टरियों से खोजे गए हुक सूचीबद्ध करता है।

- `--eligible`: केवल वे हुक जिनकी आवश्यकताएँ पूरी होती हैं।
- `--json`: संरचित आउटपुट।
- `-v, --verbose`: पूरी न हुई आवश्यकताओं वाला Missing कॉलम शामिल करें।

```
हुक (4/5 तैयार)

तैयार:
  🚀 boot-md ✓ - Gateway स्टार्टअप पर BOOT.md चलाएँ
  📎 bootstrap-extra-files ✓ - एजेंट बूटस्ट्रैप के दौरान अतिरिक्त वर्कस्पेस बूटस्ट्रैप फ़ाइलें प्रविष्ट करें
  📝 command-logger ✓ - सभी कमांड इवेंट को एक केंद्रीकृत ऑडिट फ़ाइल में लॉग करें
  💾 session-memory ✓ - /new या /reset कमांड जारी होने पर सेशन संदर्भ को मेमोरी में सहेजें
```

## हुक की जानकारी प्राप्त करें

```bash
openclaw hooks info <name> [--json]
```

`<name>` हुक का नाम या हुक कुंजी है (उदाहरण के लिए `session-memory`)। स्रोत, फ़ाइल/हैंडलर पथ, होमपेज, इवेंट और प्रत्येक आवश्यकता की स्थिति (बाइनरी, env, कॉन्फ़िगरेशन, OS) दिखाता है।

## पात्रता जाँचें

```bash
openclaw hooks check [--json]
```

तैयार/तैयार-नहीं संख्या का सारांश प्रिंट करता है; जो हुक तैयार नहीं हैं, उनमें से प्रत्येक को उसके अवरोधक कारण के साथ सूचीबद्ध करता है।

## हुक सक्षम करें

```bash
openclaw hooks enable <name>
```

कॉन्फ़िगरेशन में `hooks.internal.entries.<name>.enabled = true` जोड़ता/अपडेट करता है और `hooks.internal.enabled` मुख्य स्विच भी चालू करता है (कम-से-कम एक हुक कॉन्फ़िगर होने तक Gateway किसी आंतरिक हुक हैंडलर को लोड नहीं करता)। यदि हुक मौजूद नहीं है, Plugin द्वारा प्रबंधित है या पात्र नहीं है (आवश्यकताएँ अनुपलब्ध हैं), तो यह विफल हो जाता है।

Plugin द्वारा प्रबंधित हुक `hooks list` में `plugin:<id>` दिखाते हैं और उन्हें यहाँ सक्षम/अक्षम नहीं किया जा सकता; इसके बजाय स्वामी Plugin को सक्षम या अक्षम करें।

सक्षम करने के बाद Gateway को पुनः आरंभ करें (macOS मेनू बार ऐप पुनः आरंभ करें या डेवलपमेंट में अपनी Gateway प्रक्रिया पुनः आरंभ करें), ताकि वह हुक दोबारा लोड करे।

## हुक अक्षम करें

```bash
openclaw hooks disable <name>
```

`hooks.internal.entries.<name>.enabled = false` सेट करता है। इसके बाद Gateway को पुनः आरंभ करें।

## हुक पैक इंस्टॉल और अपडेट करें

```bash
openclaw plugins install <package>        # डिफ़ॉल्ट रूप से npm
openclaw plugins install npm:<package>    # केवल npm
openclaw plugins install <package> --pin  # निर्धारित संस्करण पिन करें
openclaw plugins install <path>           # स्थानीय डायरेक्टरी या आर्काइव
openclaw plugins install -l <path>        # कॉपी करने के बजाय स्थानीय डायरेक्टरी लिंक करें

openclaw plugins update <id>
openclaw plugins update --all
openclaw plugins update --dry-run
```

हुक पैक एकीकृत Plugin इंस्टॉलर/अपडेटर के माध्यम से इंस्टॉल होते हैं; `openclaw hooks install` / `openclaw hooks update` अब भी अप्रचलित उपनामों के रूप में काम करते हैं, जो चेतावनी प्रिंट करके `plugins` कमांड को अग्रेषित करते हैं।

- Npm विनिर्देश केवल रजिस्ट्री के लिए हैं: पैकेज नाम और वैकल्पिक सटीक संस्करण या dist-tag। Git/URL/file विनिर्देश और semver रेंज अस्वीकार किए जाते हैं। निर्भरता इंस्टॉलेशन `--ignore-scripts` के साथ प्रोजेक्ट-स्थानीय रूप से चलते हैं।
- साधारण विनिर्देश और `@latest` स्थिर ट्रैक पर रहते हैं; यदि npm किसी प्रीरिलीज़ को निर्धारित करता है, तो OpenClaw रुककर आपसे स्पष्ट रूप से सहमति देने को कहता है (`@beta`, `@rc`, या सटीक प्रीरिलीज़ संस्करण)।
- समर्थित आर्काइव: `.zip`, `.tgz`, `.tar.gz`, `.tar`।
- `-l, --link` स्थानीय डायरेक्टरी को कॉपी करने के बजाय लिंक करता है (उसे `hooks.internal.load.extraDirs` में जोड़ता है); लिंक किए गए हुक पैक ऑपरेटर द्वारा कॉन्फ़िगर की गई डायरेक्टरी के प्रबंधित हुक होते हैं, वर्कस्पेस हुक नहीं।
- `--pin` npm इंस्टॉलेशन को साझा SQLite स्थिति में सटीक निर्धारित `name@version` के रूप में रिकॉर्ड करता है।
- इंस्टॉलेशन पैक को `~/.openclaw/hooks/<id>` में कॉपी करता है, उसके हुक को `hooks.internal.entries.*` के अंतर्गत सक्षम करता है और इंस्टॉलेशन उद्गम को साझा SQLite स्थिति में रिकॉर्ड करता है।
- यदि संग्रहीत अखंडता हैश अब प्राप्त आर्टिफ़ैक्ट से मेल नहीं खाता, तो OpenClaw चेतावनी देता है और आगे बढ़ने से पहले संकेत देता है; संकेत को छोड़ने के लिए वैश्विक `--yes` पास करें (उदाहरण के लिए CI में)।

## बंडल किए गए हुक

| हुक                   | इवेंट                                            | इसका कार्य                                                                                       |
| --------------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| boot-md               | `gateway:startup`                                 | प्रत्येक कॉन्फ़िगर किए गए एजेंट दायरे के लिए Gateway स्टार्टअप पर `BOOT.md` चलाता है                                  |
| bootstrap-extra-files | `agent:bootstrap`                                 | एजेंट बूटस्ट्रैप के दौरान अतिरिक्त बूटस्ट्रैप फ़ाइलें (उदाहरण के लिए मोनोरेपो `AGENTS.md`/`TOOLS.md`) प्रविष्ट करता है |
| command-logger        | `command`                                         | कमांड इवेंट को `~/.openclaw/logs/commands.log` में लॉग करता है                                             |
| compaction-notifier   | `session:compact:before`, `session:compact:after` | सेशन Compaction शुरू और पूरा होने पर दृश्यमान चैट सूचनाएँ भेजता है                             |
| session-memory        | `command:new`, `command:reset`                    | `/new` या `/reset` पर सेशन संदर्भ को मेमोरी में सहेजता है                                              |

किसी भी बंडल किए गए हुक को `openclaw hooks enable <hook-name>` से सक्षम करें। पूर्ण विवरण, कॉन्फ़िगरेशन कुंजियाँ और डिफ़ॉल्ट: [बंडल किए गए हुक](/hi/automation/hooks#bundled-hooks)।

### command-logger लॉग फ़ाइल

```bash
tail -n 20 ~/.openclaw/logs/commands.log        # हाल के कमांड
cat ~/.openclaw/logs/commands.log | jq .          # सुव्यवस्थित प्रिंट
grep '"action":"new"' ~/.openclaw/logs/commands.log | jq .   # कार्रवाई के अनुसार फ़िल्टर करें
```

## टिप्पणियाँ

- `hooks list --json`, `info --json`, और `check --json` संरचित JSON को सीधे stdout पर लिखते हैं।

## संबंधित

- [CLI संदर्भ](/hi/cli)
- [स्वचालन हुक](/hi/automation/hooks)
