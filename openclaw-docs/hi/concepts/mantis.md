---
read_when:
    - OpenClaw बग के लिए लाइव विज़ुअल QA बनाना या चलाना
    - पुल रिक्वेस्ट के लिए पहले और बाद का सत्यापन जोड़ना
    - Discord, Slack, WhatsApp या अन्य लाइव ट्रांसपोर्ट परिदृश्य जोड़ना
    - किसी उम्मीदवार रेफ़रेंस के लिए केंद्रित Control UI ब्राउज़र प्रमाण चलाना
    - स्क्रीनशॉट, ब्राउज़र ऑटोमेशन या VNC एक्सेस की आवश्यकता वाले QA रन की डीबगिंग
summary: Mantis लाइव ट्रांसपोर्ट तुलनाओं और केवल उम्मीदवार पर केंद्रित ब्राउज़र प्रमाणों के लिए विज़ुअल एंड-टू-एंड साक्ष्य कैप्चर करता है, फिर आर्टिफ़ैक्ट को PR से संलग्न करता है।
title: Mantis
x-i18n:
    generated_at: "2026-07-27T20:45:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 48a1b306e37aba7e8c67139df61f3680a9aec066361aa196d88c81270337bc1b
    source_path: concepts/mantis.md
    workflow: 16
---

Mantis, OpenClaw व्यवहार के लिए दृश्य CI साक्ष्य और एक PR टिप्पणी प्रकाशित करता है।
लाइव ट्रांसपोर्ट परिदृश्य किसी ज्ञात-खराब बेसलाइन की तुलना कैंडिडेट रेफ़ से करते हैं;
इसके बजाय केंद्रित ब्राउज़र लेन किसी नियतात्मक
मॉक किए गए ट्रांसपोर्ट के विरुद्ध एक कैंडिडेट को प्रमाणित कर सकती हैं। Discord सबसे पहले वास्तविक बॉट प्रमाणीकरण, गिल्ड चैनलों,
प्रतिक्रियाओं, थ्रेडों और ब्राउज़र साक्षी के साथ जारी हुआ। Slack, Telegram और केंद्रित Control
UI चैट लेन भी मौजूद हैं; WhatsApp और Matrix कार्यान्वित नहीं हैं।

## स्वामित्व

- OpenClaw (`extensions/qa-lab/src/mantis/*`): परिदृश्य रनटाइम, `pnpm openclaw qa mantis <command>` CLI, साक्ष्य स्कीमा।
- QA Lab (`extensions/qa-lab/src/live-transports/*`): लाइव ट्रांसपोर्ट हार्नेस, ड्राइवर/SUT बॉट, रिपोर्ट/साक्ष्य राइटर।
- Crabbox (`openclaw/crabbox`): तैयार Linux मशीनें, लीज़, VNC, `crabbox media preview`।
- GitHub Actions (`.github/workflows/mantis-*.yml`): रिमोट एंट्रीपॉइंट, आर्टिफ़ैक्ट प्रतिधारण।
- ClawSweeper: मेंटेनर PR कमांड पार्स करता है, वर्कफ़्लो डिस्पैच करता है और अंतिम PR टिप्पणी पोस्ट करता है।

## CLI कमांड

सभी कमांड `pnpm openclaw qa mantis <command>` हैं, जिन्हें
`extensions/qa-lab/src/mantis/cli.ts` में परिभाषित किया गया है। बिल्ड/रन समय पर `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1`
आवश्यक है (बंडल किए गए वर्कफ़्लो बिल्ड से पहले `OPENCLAW_BUILD_PRIVATE_QA=1` और
`OPENCLAW_ENABLE_PRIVATE_QA_CLI=1` सेट करते हैं)।

| कमांड                         | उद्देश्य                                                                                                                                                   |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `discord-smoke`                 | सत्यापित करें कि Mantis Discord बॉट गिल्ड/चैनल देख सकता है, पोस्ट कर सकता है और प्रतिक्रिया दे सकता है।                                                                                 |
| `run`                           | बेसलाइन और कैंडिडेट रेफ़ के विरुद्ध पहले/बाद का परिदृश्य चलाएँ (केवल Discord)।                                                                           |
| `desktop-browser-smoke`         | Crabbox डेस्कटॉप लीज़ पर लें/पुनः उपयोग करें, दृश्यमान ब्राउज़र खोलें, स्क्रीनशॉट + वीडियो कैप्चर करें।                                                                        |
| `slack-desktop-smoke`           | Crabbox डेस्कटॉप लीज़ पर लें/पुनः उपयोग करें, उसमें Slack QA चलाएँ, Slack Web खोलें, साक्ष्य कैप्चर करें।                                                                  |
| `telegram-desktop-builder`      | Crabbox डेस्कटॉप लीज़ पर लें/पुनः उपयोग करें, Telegram Desktop इंस्टॉल करें, वैकल्पिक रूप से OpenClaw Gateway कॉन्फ़िगर करें।                                                        |
| `visual-task` / `visual-driver` | वैकल्पिक छवि-समझ अभिकथनों के साथ सामान्य Crabbox डेस्कटॉप कैप्चर; `visual-driver`, `crabbox record --while` के अंतर्गत लॉन्च होने वाला ड्राइवर भाग है। |

हर कमांड `--repo-root <path>` और `--output-dir <path>` स्वीकार करता है; Crabbox
कमांड `--crabbox-bin`, `--provider`, `--machine-class`/`--class`,
`--lease-id`, `--idle-timeout`, `--ttl` और `--keep-lease` भी स्वीकार करते हैं। प्रदाता/क्लास के लिए स्थानीय CLI डिफ़ॉल्ट
`hetzner`/`beast` हैं, जब तक अन्यथा न बताया गया हो; CI वर्कफ़्लो
आमतौर पर दोनों को ओवरराइड करते हैं।

### `discord-smoke`

```bash
pnpm openclaw qa mantis discord-smoke \
  --output-dir .artifacts/qa-e2e/mantis/discord-smoke
```

बॉट उपयोगकर्ता, गिल्ड, गिल्ड के चैनल और लक्ष्य चैनल प्राप्त करने के लिए Discord REST API (`https://discord.com/api/v10`) को
कॉल करता है, अभिकथित करता है कि चैनल गिल्ड से संबंधित है, फिर (`--skip-post` न होने पर) एक संदेश पोस्ट करता है और
`👀` प्रतिक्रिया जोड़ता है। `mantis-discord-smoke-summary.json` और
`mantis-discord-smoke-report.md` लिखता है।

टोकन समाधान क्रम: `--token-file` मान, फिर `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN`
(`--token-env` से ओवरराइड करें), फिर `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN_FILE` द्वारा नामित फ़ाइल
(`--token-file-env` से ओवरराइड करें)। गिल्ड/चैनल आईडी
`OPENCLAW_QA_DISCORD_GUILD_ID` / `OPENCLAW_QA_DISCORD_CHANNEL_ID` से आते हैं (`--guild-id` /
`--channel-id` से ओवरराइड करें) और 17-20 अंकों वाले Discord स्नोफ़्लेक होने चाहिए। प्रकाशित सारांश और रिपोर्ट में
बॉट/गिल्ड/चैनल/संदेश आईडी और नामों को `<redacted>` से बदलने के लिए
`OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` सेट करें।

### `run`

```bash
pnpm openclaw qa mantis run \
  --transport discord \
  --scenario discord-status-reactions-tool-only \
  --baseline origin/main \
  --candidate HEAD \
  --output-dir .artifacts/qa-e2e/mantis/local-discord-status-reactions
```

`--transport` वर्तमान में केवल `discord` स्वीकार करता है। `--scenario`, दो
अंतर्निहित आईडी में से एक है, प्रत्येक का अपना डिफ़ॉल्ट बेसलाइन रेफ़ और अपेक्षित पहले/बाद
लेबल (`extensions/qa-lab/src/mantis/run.runtime.ts`) है:

| परिदृश्य                                   | डिफ़ॉल्ट बेसलाइन                           | बेसलाइन की अपेक्षा                         | कैंडिडेट की अपेक्षा            |
| ------------------------------------------ | ------------------------------------------ | ---------------------------------------- | ---------------------------- |
| `discord-status-reactions-tool-only`       | `0bf06e953fdda290799fc9fb9244a8f67fdae593` | `queued-only`                            | `queued -> thinking -> done` |
| `discord-thread-reply-filepath-attachment` | `81349cdc2a9d5143fd0991ed858b739e7d96e05c` | थ्रेड उत्तर में `filePath` अटैचमेंट अनुपस्थित है | थ्रेड उत्तर में यह शामिल है     |

`--candidate` का डिफ़ॉल्ट `HEAD` है। अन्य फ़्लैग: `--credential-source`
(डिफ़ॉल्ट `convex`), `--credential-role` (डिफ़ॉल्ट `ci`), `--provider-mode`
(डिफ़ॉल्ट `live-frontier`), `--fast` (डिफ़ॉल्ट रूप से चालू), `--skip-install`, `--skip-build`।

रनर, `<output-dir>/worktrees/` के अंतर्गत बेसलाइन और
कैंडिडेट के लिए अलग किए गए `git worktree` चेकआउट बनाता है, प्रत्येक में
`pnpm install`/`pnpm build` चलाता है (यदि छोड़ा न गया हो), फिर प्रत्येक वर्कट्री के विरुद्ध
`pnpm openclaw qa discord --scenario <id> --model openai/gpt-5.4 --alt-model openai/gpt-5.4 --allow-failures`
चलाता है। प्रत्येक लेन `discord-qa-reaction-timelines.json`
के साथ एक `<scenario-id>-timeline.html`/`.png` युग्म लिखती है; रनर इस
साक्ष्य को वापस `baseline/`/`candidate/` के अंतर्गत कॉपी करता है, आउटपुट डायरेक्टरी में `comparison.json`,
`mantis-report.md` और `mantis-evidence.json` लिखता है, और
यदि तुलना पास नहीं हुई (बेसलाइन `fail` और कैंडिडेट
`pass`) तो शून्येतर स्थिति के साथ बाहर निकलता है।

दूसरा Discord परिदृश्य (`discord-thread-reply-filepath-attachment`)
ड्राइवर बॉट के साथ एक पैरेंट संदेश पोस्ट करता है, वास्तविक थ्रेड बनाता है, रेपो-स्थानीय `filePath` के साथ SUT की
`message.thread-reply` कार्रवाई कॉल करता है, फिर उत्तर और अटैचमेंट फ़ाइलनाम के लिए
थ्रेड को पोल करता है। यह `mantis-thread-report.md` नाम वाले अटैचमेंट की अपेक्षा करता है।

### `desktop-browser-smoke`

```bash
pnpm openclaw qa mantis desktop-browser-smoke \
  --output-dir .artifacts/qa-e2e/mantis/desktop-browser
```

Crabbox डेस्कटॉप को लीज़ पर लेता या पुनः उपयोग करता है, VNC सत्र के भीतर
`--browser-url` (डिफ़ॉल्ट `https://openclaw.ai`) या रेंडर किए गए
`--html-file` की ओर इंगित ब्राउज़र लॉन्च करता है, प्रतीक्षा करता है, `scrot` से स्क्रीनशॉट लेता है, वैकल्पिक रूप से
`ffmpeg` से MP4 रिकॉर्ड करता है, और `desktop-browser-smoke.png` / `.mp4` / `remote-metadata.json`
को वापस `--output-dir` पर rsync करता है।

फ़्लैग:

- `--lease-id <cbx_...>` नया डेस्कटॉप बनाने के बजाय तैयार डेस्कटॉप का पुनः उपयोग करता है।
- `--browser-profile-dir <remote-path>` रिमोट Chrome user-data-dir का पुनः उपयोग करता है, ताकि स्थायी डेस्कटॉप रन के बीच लॉग इन रहे (दीर्घकालिक Discord Web व्यूअर प्रोफ़ाइल के लिए प्रयुक्त)।
- `--browser-profile-archive-env <name>` लॉन्च से पहले उस पर्यावरण चर से base64 `.tgz` Chrome प्रोफ़ाइल आर्काइव पुनर्स्थापित करता है (डिफ़ॉल्ट `OPENCLAW_MANTIS_BROWSER_PROFILE_TGZ_B64`); Discord Web जैसे लॉग-इन साक्षियों के लिए प्रयुक्त।
- `--video-duration <seconds>` MP4 कैप्चर अवधि नियंत्रित करता है (डिफ़ॉल्ट 10s)।
- `--keep-lease` (या `OPENCLAW_MANTIS_KEEP_VM=1`) इस रन द्वारा बनाई गई लीज़ को VNC निरीक्षण के लिए खुला रखता है; लीज़ बनाने वाले विफल रन भी डिफ़ॉल्ट रूप से उसे खुला रखते हैं।

Discord Web साक्ष्य के लिए Mantis, बॉट
टोकन नहीं, बल्कि समर्पित व्यूअर अकाउंट का उपयोग करता है। Discord REST ऑरेकल (`qa discord` के माध्यम से) प्रामाणिक बना रहता है; जब
`OPENCLAW_QA_DISCORD_CAPTURE_UI_METADATA=1` सेट होता है, तो परिदृश्य एक
Discord Web URL आर्टिफ़ैक्ट भी लिखता है, और `OPENCLAW_QA_DISCORD_KEEP_THREADS=1`
थ्रेड को इतना समय खुला रखता है कि ब्राउज़र उसे खोल सके।

GitHub वर्कफ़्लो
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR` के माध्यम से स्थायी व्यूअर प्रोफ़ाइल को प्राथमिकता देता है (पूर्ण प्रोफ़ाइल आर्काइव
GitHub की सीक्रेट आकार सीमा से बड़े हो सकते हैं); छोटे/बूटस्ट्रैप प्रोफ़ाइल के लिए यह इसके बजाय
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64` से base64 `.tgz` पुनर्स्थापित कर सकता है। इनमें से
कोई भी स्रोत कॉन्फ़िगर न होने पर भी वर्कफ़्लो नियतात्मक
बेसलाइन/कैंडिडेट स्क्रीनशॉट प्रकाशित करता है और लॉग करता है कि लॉग-इन साक्षी
छोड़ दिया गया था।

### `slack-desktop-smoke`

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --output-dir .artifacts/qa-e2e/mantis/slack-desktop \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

Crabbox डेस्कटॉप को लीज़ पर लेता या पुनः उपयोग करता है, चेकआउट को VM में सिंक करता है, उसके भीतर
`pnpm openclaw qa slack` चलाता है, VNC ब्राउज़र में Slack Web खोलता है,
डेस्कटॉप कैप्चर करता है और Slack QA आर्टिफ़ैक्ट (`slack-qa/`) तथा
VNC स्क्रीनशॉट/वीडियो दोनों को वापस स्थानीय रूप से कॉपी करता है। यह एकमात्र Mantis स्वरूप है जिसमें
SUT Gateway और ब्राउज़र दोनों समान VM के भीतर चलते हैं।

`--gateway-setup` के साथ कमांड VM में `$HOME/.openclaw-mantis/slack-openclaw` पर एक स्थायी, नष्ट की जा सकने वाली OpenClaw
होम डायरेक्टरी बनाता है, लक्ष्य चैनल के लिए Slack
Socket Mode कॉन्फ़िगरेशन पैच करता है,
`openclaw gateway run --dev --allow-unconfigured --port 38973` शुरू करता है, और
VNC सत्र में Chrome को चलता छोड़ता है; `--gateway-setup` को हटाने पर इसके बजाय सामान्य
बॉट-से-बॉट Slack QA लेन चलती है।

`--credential-source env` के लिए आवश्यक पर्यावरण चर (स्थानीय डिफ़ॉल्ट `env` है; भूमिका
डिफ़ॉल्ट `maintainer` है):

- `OPENCLAW_QA_SLACK_CHANNEL_ID`
- `OPENCLAW_QA_SLACK_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_APP_TOKEN`
- `OPENCLAW_LIVE_OPENAI_KEY` रिमोट मॉडल लेन के लिए (यदि स्थानीय रूप से केवल `OPENAI_API_KEY`
  सेट है, तो Crabbox को आह्वान करने से पहले Mantis इसे `OPENCLAW_LIVE_OPENAI_KEY` में कॉपी करता है)

`--credential-source convex` के साथ Mantis, VM बनाने से पहले
साझा पूल से Slack SUT क्रेडेंशियल लीज़ पर लेता है और चैनल आईडी, ऐप टोकन तथा
बॉट टोकन को `OPENCLAW_MANTIS_SLACK_*` पर्यावरण चरों के रूप में VM में अग्रेषित करता है, इसलिए GitHub
वर्कफ़्लो को कच्चे Slack टोकन नहीं, केवल Convex ब्रोकर सीक्रेट चाहिए।

अन्य फ़्लैग: `--slack-url <url>` एक विशिष्ट URL खोलता है (अन्यथा Mantis
`auth.test` से `https://app.slack.com/client/<team>/<channel>` प्राप्त करता है);
`--slack-channel-id <id>` Gateway अनुमतिसूची चैनल सेट करता है;
`OPENCLAW_MANTIS_SLACK_BROWSER_PROFILE_DIR` VM के भीतर स्थायी Chrome
प्रोफ़ाइल नियंत्रित करता है (डिफ़ॉल्ट `$HOME/.config/openclaw-mantis/slack-chrome-profile`);
`--approval-checkpoints` नेटिव Slack अनुमोदन परिदृश्य
(`slack-approval-exec-native`, `slack-approval-plugin-native`) चलाता है और
Gateway सेटअप के बजाय लंबित/समाधानित चेकपॉइंट स्क्रीनशॉट रेंडर करता है (`--gateway-setup` के साथ
परस्पर अनन्य); `--hydrate-mode source|prehydrated`,
`--provider-mode`, `--model`, `--alt-model` और `--fast`, Slack लाइव लेन को यथावत पास किए जाते हैं।

अनुमोदन चेकपॉइंट स्क्रीनशॉट, परिदृश्य द्वारा देखे गए Slack API संदेश से
रेंडर किए जाते हैं, लाइव Slack UI से नहीं; `slack-desktop-smoke.png` केवल तभी
Slack Web का प्रमाण है जब लीज़ की ब्राउज़र प्रोफ़ाइल पहले से लॉग इन थी।

### `telegram-desktop-builder`

```bash
pnpm openclaw qa mantis telegram-desktop-builder \
  --credential-source convex \
  --credential-role maintainer \
  --keep-lease
```

Crabbox डेस्कटॉप को लीज़ पर लेता या पुनः उपयोग करता है, नेटिव Linux Telegram Desktop इंस्टॉल करता है,
वैकल्पिक रूप से उपयोगकर्ता-सत्र आर्काइव पुनर्स्थापित करता है, लीज़ पर लिए गए Telegram SUT बॉट टोकन से OpenClaw कॉन्फ़िगर करता है,
`openclaw gateway run --dev --allow-unconfigured --port 38974` शुरू करता है, लीज़ पर लिए गए निजी समूह में
ड्राइवर-बॉट तत्परता संदेश पोस्ट करता है, फिर स्क्रीनशॉट और MP4 कैप्चर करता है। बॉट टोकन केवल OpenClaw कॉन्फ़िगर करता है; वह Telegram Desktop में
कभी लॉग इन नहीं करता। डेस्कटॉप व्यूअर एक अलग Telegram उपयोगकर्ता सत्र है,
जिसे `--telegram-profile-archive-env <name>` से पुनर्स्थापित किया जाता है या
VNC के माध्यम से मैन्युअल रूप से लॉग इन किया जाता है और `--keep-lease` के साथ सक्रिय रखा जाता है।

फ़्लैग: `--lease-id <cbx_...>`, Telegram Desktop में पहले से लॉग इन
VM के विरुद्ध पुनः चलाता है; `--telegram-profile-archive-env <name>` लॉन्च से पहले base64
`.tgz` प्रोफ़ाइल आर्काइव पुनर्स्थापित करता है; `--telegram-profile-dir <remote-path>`
रिमोट प्रोफ़ाइल डायरेक्टरी सेट करता है (डिफ़ॉल्ट `$HOME/.local/share/TelegramDesktop`);
`--no-gateway-setup` केवल Telegram Desktop इंस्टॉल करके खोलता है;
`--credential-source`/`--credential-role` के डिफ़ॉल्ट `convex`/`maintainer` हैं।

## साक्ष्य मैनिफ़ेस्ट

हर वह परिदृश्य जो किसी PR पर प्रकाशित करता है, अपनी रिपोर्ट के पास
`mantis-evidence.json` लिखता है:

```json
{
  "schemaVersion": 1,
  "id": "discord-status-reactions",
  "title": "Mantis Discord स्थिति प्रतिक्रियाएँ QA",
  "summary": "PR टिप्पणी के लिए मानव-पठनीय शीर्ष सारांश।",
  "scenario": "discord-status-reactions-tool-only",
  "comparison": {
    "baseline": { "sha": "...", "status": "fail", "expected": "queued-only" },
    "candidate": { "sha": "...", "status": "pass", "expected": "queued -> thinking -> done" },
    "pass": true
  },
  "artifacts": [
    {
      "kind": "timeline",
      "lane": "baseline",
      "label": "बेसलाइन केवल कतारबद्ध",
      "path": "baseline/timeline.png",
      "targetPath": "baseline.png",
      "alt": "बेसलाइन Discord टाइमलाइन",
      "width": 420
    }
  ]
}
```

आर्टिफ़ैक्ट `path` मैनिफ़ेस्ट की डायरेक्टरी के सापेक्ष है; `targetPath`
कॉन्फ़िगर किए गए R2/S3 आर्टिफ़ैक्ट प्रीफ़िक्स के सापेक्ष है। फ़ाइल अनुपस्थित होने पर
`scripts/mantis/publish-pr-evidence.mjs` पाथ ट्रैवर्सल अस्वीकार करता है और `"required": false` वाली
प्रविष्टियाँ छोड़ देता है।

आर्टिफ़ैक्ट प्रकार: `timeline` (निर्धारित पहले/बाद का स्क्रीनशॉट),
`desktopScreenshot` (VNC/ब्राउज़र स्क्रीनशॉट), `motionPreview` (रिकॉर्डिंग से इनलाइन एनिमेटेड
GIF), `motionClip` (गतिविधि-अनुसार ट्रिम किया गया MP4), `fullVideo` (पूर्ण
रिकॉर्डिंग), `metadata` (JSON/लॉग साइडकार), `report` (Markdown रिपोर्ट)।

किसी रन का डिस्क पर आर्टिफ़ैक्ट लेआउट:

```text
.artifacts/qa-e2e/mantis/<run-id>/
  mantis-report.md
  mantis-evidence.json
  baseline/
  candidate/
  comparison.json
```

स्क्रीनशॉट साक्ष्य हैं, सीक्रेट नहीं, लेकिन फिर भी उनमें रिडैक्शन अनुशासन आवश्यक है:
निजी चैनल नाम, उपयोगकर्ता नाम या संदेश सामग्री दिखाई दे सकती है। सार्वजनिक आर्टिफ़ैक्ट
अपलोड के लिए `OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` सेट करें; यह
Discord/Slack/Telegram GitHub वर्कफ़्लो में डिफ़ॉल्ट रूप से सक्षम है।

## GitHub स्वचालन

`scripts/mantis/publish-pr-evidence.mjs` पुनः उपयोग योग्य प्रकाशक है। वर्कफ़्लो
इसे मैनिफ़ेस्ट, लक्ष्य PR, आर्टिफ़ैक्ट लक्ष्य रूट, टिप्पणी मार्कर,
आर्टिफ़ैक्ट URL, रन URL और अनुरोध स्रोत के साथ कॉल करते हैं। यह घोषित आर्टिफ़ैक्ट को
Mantis R2 बकेट में अपलोड करता है, इनलाइन छवियों/पूर्वावलोकनों और लिंक किए गए वीडियो वाली
सारांश-प्रथम PR टिप्पणी बनाता है, फिर मौजूदा मार्कर टिप्पणी अपडेट करता है या
नई टिप्पणी बनाता है। आवश्यक एनवायरनमेंट:

- `MANTIS_ARTIFACT_R2_ACCESS_KEY_ID`
- `MANTIS_ARTIFACT_R2_SECRET_ACCESS_KEY`
- `MANTIS_ARTIFACT_R2_BUCKET` (वर्कफ़्लो `openclaw-crabbox-artifacts` सेट करते हैं)
- `MANTIS_ARTIFACT_R2_ENDPOINT`
- `MANTIS_ARTIFACT_R2_REGION` (वर्कफ़्लो `auto` सेट करते हैं)
- `MANTIS_ARTIFACT_R2_PUBLIC_BASE_URL` (वर्कफ़्लो `https://artifacts.openclaw.ai` सेट करते हैं)

टिप्पणियाँ `github-actions[bot]` के बजाय Mantis GitHub App (`MANTIS_GITHUB_APP_ID` /
`MANTIS_GITHUB_APP_PRIVATE_KEY`) के माध्यम से पोस्ट होती हैं, और छिपी हुई
मार्कर टिप्पणी को अपसर्ट कुंजी के रूप में उपयोग करती हैं।

| वर्कफ़्लो                          | ट्रिगर                                                                                    | यह क्या करता है                                                                                                                                                                                                                                                                                                     |
| --------------------------------- | ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Mantis Discord Smoke`            | मैन्युअल डिस्पैच                                                                            | चुने गए रेफ़ के विरुद्ध `discord-smoke` चलाता है।                                                                                                                                                                                                                                                                       |
| `Mantis Discord Status Reactions` | PR टिप्पणी या मैन्युअल डिस्पैच                                                              | अलग-अलग बेसलाइन/कैंडिडेट वर्कट्री बनाता है, प्रत्येक पर `discord-status-reactions-tool-only` चलाता है, Crabbox डेस्कटॉप ब्राउज़र में प्रत्येक लेन की टाइमलाइन रेंडर करता है, `crabbox media preview` से गतिविधि-अनुसार ट्रिम किए गए GIF/MP4 पूर्वावलोकन बनाता है, आर्टिफ़ैक्ट अपलोड करता है और इनलाइन PR साक्ष्य पोस्ट करता है।                                 |
| `Mantis Scenario`                 | मैन्युअल डिस्पैच                                                                            | सामान्य डिस्पैचर: `scenario_id` (`discord-status-reactions-tool-only`, `discord-thread-reply-filepath-attachment`, `slack-desktop-smoke`, `telegram-live`, `telegram-desktop-proof`, `web-ui-chat-proof`), `baseline_ref`, `candidate_ref`, `pr_number` लेता है और मेल खाने वाले परिदृश्य वर्कफ़्लो को अग्रेषित करता है। |
| `Mantis Slack Desktop Smoke`      | मैन्युअल डिस्पैच                                                                            | Crabbox Linux डेस्कटॉप लीज़ करता है (डिफ़ॉल्ट `aws`, `hetzner` का विकल्प), कैंडिडेट के विरुद्ध `slack-desktop-smoke --gateway-setup` चलाता है, डेस्कटॉप रिकॉर्ड करता है, गतिविधि पूर्वावलोकन बनाता है, आर्टिफ़ैक्ट अपलोड करता है और PR संख्या दिए जाने पर PR साक्ष्य पोस्ट करता है।                                                      |
| `Mantis Telegram Live`            | PR टिप्पणी या मैन्युअल डिस्पैच                                                              | बॉट-API Telegram लाइव QA लेन (`openclaw qa telegram`) चलाता है, QA सारांश से `mantis-evidence.json` लिखता है, Crabbox डेस्कटॉप ब्राउज़र के माध्यम से रिडैक्ट किया गया साक्ष्य HTML रेंडर करता है, गतिविधि GIF बनाता है और PR साक्ष्य पोस्ट करता है। इस लेन के लिए Telegram Web लॉगिन आवश्यक नहीं है।                               |
| `Mantis Telegram Desktop Proof`   | मेंटेनर PR लेबल (`mantis: telegram-visible-proof`) और PR टिप्पणी, या मैन्युअल डिस्पैच | एजेंट-संचालित मूल Telegram Desktop पहले/बाद का प्रमाण। PR, बेसलाइन/कैंडिडेट रेफ़ और मेंटेनर निर्देश Codex को देता है, जो दोनों रेफ़ के लिए वास्तविक-उपयोगकर्ता Crabbox Telegram Desktop प्रमाण लेन चलाता है और 2-कॉलम वाली PR साक्ष्य तालिका पोस्ट करता है।                                                              |
| `Mantis Web UI Chat Proof`        | PR टिप्पणी या मैन्युअल डिस्पैच                                                              | कैंडिडेट के विरुद्ध केंद्रित OpenClaw Control UI चैट Playwright प्रमाण चलाता है, सत्यापित करता है कि ब्राउज़र मॉक किए गए Gateway के माध्यम से भेजता है, स्क्रीनशॉट/वीडियो आर्टिफ़ैक्ट कैप्चर करता है और PR साक्ष्य पोस्ट करता है। यह लेन केवल वेब चैट का प्रमाण है, WinUI/मूल-ऐप या मनमाने विज़ुअल प्रमाण का नहीं।                           |

`Mantis Discord Status Reactions` और `Mantis Telegram Live` दोनों
`baseline_ref`/`candidate_ref` (या PR टिप्पणी में `baseline=`/`candidate=`) स्वीकार करते हैं
और सीक्रेट वाले क्रेडेंशियल के साथ चलने से पहले सत्यापित करते हैं कि समाधान किया गया SHA या तो
`origin/main` का पूर्वज है, कोई रिलीज़ टैग (`v*`) है, या किसी खुले PR का हेड है।

लिखने/मेंटेन/admin की पहुँच वाले PR से टिप्पणी ट्रिगर:

```text
@openclaw-mantis discord status reactions
@openclaw-mantis discord status reactions baseline=origin/main candidate=HEAD
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
@openclaw-mantis web ui chat
@openclaw-mantis web-ui-chat candidate=HEAD
```

Telegram टिप्पणी ट्रिगर कैंडिडेट के लिए डिफ़ॉल्ट रूप से PR हेड SHA और
परिदृश्य के लिए `telegram-status-command` का उपयोग करते हैं; किसी विशिष्ट Crabbox प्रदाता या पहले से वार्म किए गए
डेस्कटॉप को लक्षित करने के लिए वे `provider=aws|hetzner` और
`lease=<cbx_...>` स्वीकार करते हैं। `Mantis Telegram Desktop Proof` किसी PR टिप्पणी पर केवल तभी प्रतिक्रिया देता है जब
PR पर पहले से `mantis: telegram-visible-proof` लेबल मौजूद हो।

वेब UI चैट टिप्पणी ट्रिगर कैंडिडेट के लिए डिफ़ॉल्ट रूप से PR हेड SHA का उपयोग करते हैं। वे
Control UI का मॉक-Gateway चैट प्रमाण चलाते हैं और ब्राउज़र आर्टिफ़ैक्ट प्रकाशित करते हैं; अन्य
वेब पेजों और मूल ऐप सतहों के लिए सामान्य Playwright/ब्राउज़र प्रमाण, मेंटेनर स्क्रीनशॉट,
Crabbox या स्थानीय आर्टिफ़ैक्ट का उपयोग करें।

ClawSweeper किसी परिदृश्य को सीधे भी डिस्पैच कर सकता है:

```text
@clawsweeper mantis discord discord-status-reactions-tool-only
```

## मशीनें और सीक्रेट

स्थानीय CLI Crabbox के डिफ़ॉल्ट `--provider hetzner --class beast` हैं; इन्हें
`--provider`, `--class`/`--machine-class`, या
`OPENCLAW_MANTIS_CRABBOX_PROVIDER` / `OPENCLAW_MANTIS_CRABBOX_CLASS` से ओवरराइड करें। GitHub
वर्कफ़्लो आम तौर पर दोनों को ओवरराइड करते हैं (उदाहरण के लिए `--class standard`, और
Slack वर्कफ़्लो का `aws`/`hetzner` प्रदाता विकल्प इनपुट)। यदि कोई प्रदाता बहुत
धीमा या अनुपलब्ध है, तो फ़ॉलबैक को हार्डकोड करने के बजाय उसे उसी Crabbox इंटरफ़ेस के पीछे जोड़ें।

VM बेसलाइन: डेस्कटॉप-सक्षम Chrome/Chromium, CDP पहुँच, VNC/
noVNC, Node 22.22.3+, 24.15+, या 25.9+ और pnpm वाला Linux, एक OpenClaw चेकआउट, तथा
लक्ष्य ट्रांसपोर्ट, GitHub, मॉडल प्रदाताओं और
क्रेडेंशियल ब्रोकर तक आउटबाउंड पहुँच।

Mantis कमांड और वर्कफ़्लो में उपयोग किए जाने वाले क्रेडेंशियल और एनवायरनमेंट नाम:

- `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_GUILD_ID`
- `OPENCLAW_QA_DISCORD_CHANNEL_ID`
- स्थानीय `qa mantis run --credential-source env` के लिए
  `OPENCLAW_QA_DISCORD_DRIVER_BOT_TOKEN`, `OPENCLAW_QA_DISCORD_SUT_BOT_TOKEN`,
  और `OPENCLAW_QA_DISCORD_SUT_APPLICATION_ID` भी आवश्यक हैं। GitHub वर्कफ़्लो सामान्यतः कच्चे
  Discord बॉट टोकन के बजाय `--credential-source convex` और नीचे दिए गए ब्रोकर क्रेडेंशियल का उपयोग करते हैं।
- सार्वजनिक आर्टिफ़ैक्ट अपलोड के लिए `OPENCLAW_QA_REDACT_PUBLIC_METADATA=1`
- `OPENCLAW_QA_CONVEX_SITE_URL`, `OPENCLAW_QA_CONVEX_SECRET_CI`
- `OPENAI_API_KEY` (या Telegram Desktop प्रमाण-विशिष्ट
  `OPENCLAW_MANTIS_AGENT_OPENAI_API_KEY`)
- `CRABBOX_COORDINATOR` / `CRABBOX_COORDINATOR_TOKEN` (वर्कफ़्लो
  फ़ॉलबैक के रूप में `OPENCLAW_QA_MANTIS_CRABBOX_COORDINATOR` / `_TOKEN` भी स्वीकार करते हैं और
  Crabbox को इनवोक करने से पहले उन्हें सामान्य नामों पर मैप करते हैं)
- `CRABBOX_ACCESS_CLIENT_ID`, `CRABBOX_ACCESS_CLIENT_SECRET`
- `MANTIS_GITHUB_APP_ID`, `MANTIS_GITHUB_APP_PRIVATE_KEY`

Mantis रनर को कभी भी Discord/Slack/Telegram बॉट टोकन,
प्रदाता API कुंजियाँ, ब्राउज़र कुकी, प्रमाणीकरण प्रोफ़ाइल की सामग्री, VNC पासवर्ड या
कच्चे क्रेडेंशियल पेलोड प्रिंट नहीं करने चाहिए। यदि कोई टोकन किसी इश्यू, PR, चैट या लॉग में लीक हो जाए,
तो प्रतिस्थापन सीक्रेट संग्रहीत करने के बाद उसे रोटेट करें।

## रन परिणाम

पहले/बाद के ट्रांसपोर्ट परिदृश्य इन परिणामों को अलग करते हैं ताकि अस्थिर
एनवायरनमेंट को उत्पाद रिग्रेशन न समझा जाए:

- **बग पुनरुत्पादित हुआ**: बेसलाइन उस तरीके से विफल हुई जिसकी परिदृश्य अपेक्षा करता है।
- **हार्नेस विफलता**: ऑरेकल के अर्थपूर्ण होने से पहले एनवायरनमेंट सेटअप, क्रेडेंशियल, ट्रांसपोर्ट API, ब्राउज़र
  या प्रदाता विफल हो गया।

केवल-कैंडिडेट ब्राउज़र प्रमाण बताता है कि कैंडिडेट मॉक किए गए
Gateway और दृश्यमान UI अभिकथनों में सफल हुआ या नहीं; यह बेसलाइन पुनरुत्पादन का दावा नहीं करता।

## परिदृश्य जोड़ना

लाइव ट्रांसपोर्ट परिदृश्य प्रत्येक ट्रांसपोर्ट के लिए TypeScript में परिभाषित होते हैं (Discord के
पहले/बाद के आकार के लिए `extensions/qa-lab/src/mantis/run.runtime.ts` में
`MANTIS_SCENARIO_CONFIGS` देखें), वे कोई स्वतंत्र घोषणात्मक फ़ाइल फ़ॉर्मैट नहीं हैं।
प्रत्येक परिदृश्य को चाहिए: id और शीर्षक, ट्रांसपोर्ट, आवश्यक क्रेडेंशियल, बेसलाइन
रेफ़ नीति, कैंडिडेट रेफ़ नीति, OpenClaw कॉन्फ़िगरेशन पैच, सेटअप/उद्दीपन चरण,
अपेक्षित बेसलाइन और कैंडिडेट ऑरेकल, विज़ुअल कैप्चर लक्ष्य, टाइमआउट
बजट और क्लीनअप चरण।

केंद्रित केवल-कैंडिडेट ब्राउज़र प्रमाण किसी समर्पित निर्धारित E2E परीक्षण
और वर्कफ़्लो का उपयोग कर सकता है। इसका दायरा स्पष्ट रखें, निष्पादन से पहले
कैंडिडेट रेफ़ सत्यापित करें, सीक्रेट-समर्थित प्रकाशन को पृथक रखें और वही साक्ष्य
मैनिफ़ेस्ट अनुबंध उत्सर्जित करें।

विज़न जाँचों की तुलना में छोटे, टाइप किए गए ऑरेकल को प्राथमिकता दें: Discord प्रतिक्रिया स्थिति या
संदेश संदर्भ, Slack थ्रेड `ts`/प्रतिक्रिया API स्थिति, ईमेल संदेश id
और हेडर। जब UI ही एकमात्र विश्वसनीय अवलोकन हो, तब ब्राउज़र स्क्रीनशॉट का उपयोग करें,
और जहाँ प्लेटफ़ॉर्म-API ऑरेकल उपलब्ध हो वहाँ विज़न जाँचों को उसके अतिरिक्त रखें।

Discord, Slack और Telegram के बाद, यही रनर आकार WhatsApp
(QR लॉगिन, पुनः पहचान, डिलीवरी, मीडिया, प्रतिक्रियाएँ) और Matrix
(एन्क्रिप्टेड रूम, थ्रेड/उत्तर संबंध, रीस्टार्ट के बाद पुनः आरंभ) तक विस्तारित होता है; इनमें से
कोई भी अभी लागू नहीं है।

## खुले प्रश्न

- मौजूदा Mantis बॉट का दोबारा उपयोग किए जाने पर, कौन-सा Discord बॉट ड्राइवर होना चाहिए और कौन-सा SUT?
- GitHub को PR के लिए Mantis आर्टिफ़ैक्ट कितने समय तक रखने चाहिए?
- ClawSweeper को मेंटेनर कमांड की प्रतीक्षा करने के बजाय कब स्वचालित रूप से Mantis परिदृश्य की अनुशंसा करनी चाहिए?
- सार्वजनिक PR के लिए अपलोड करने से पहले क्या स्क्रीनशॉट में संवेदनशील जानकारी छिपाई जानी चाहिए या उन्हें क्रॉप किया जाना चाहिए?
