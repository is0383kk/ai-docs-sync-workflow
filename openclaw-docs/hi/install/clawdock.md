---
read_when:
    - आप अक्सर Docker के साथ OpenClaw चलाते हैं और रोज़मर्रा के लिए छोटे कमांड चाहते हैं
    - आपको डैशबोर्ड, लॉग, टोकन सेटअप और पेयरिंग प्रवाहों के लिए एक सहायक परत चाहिए
summary: Docker-आधारित OpenClaw इंस्टॉलेशन के लिए ClawDock शेल सहायक फ़ंक्शन
title: ClawDock
x-i18n:
    generated_at: "2026-07-27T19:27:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bb829a3301178503f910931e86a39f7befeaf186044f4088a25dc80ea99130d
    source_path: install/clawdock.md
    workflow: 16
---

ClawDock, Docker-आधारित OpenClaw इंस्टॉलेशन के लिए एक छोटी shell-helper परत है।

यह लंबे `docker compose ...` आह्वानों के बजाय `clawdock-start`, `clawdock-dashboard`, और `clawdock-fix-token` जैसे छोटे कमांड प्रदान करता है।

यदि आपने अभी तक Docker सेट अप नहीं किया है, तो [Docker](/hi/install/docker) से शुरू करें।

## इंस्टॉल करें

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

यदि आपने पहले `scripts/shell-helpers/clawdock-helpers.sh` से ClawDock इंस्टॉल किया था, तो मौजूदा `scripts/clawdock/clawdock-helpers.sh` पथ से दोबारा इंस्टॉल करें; पुराना raw GitHub पथ हटा दिया गया है।

पहली बार उपयोग करने पर helpers आपके OpenClaw checkout का स्वतः पता लगाते हैं (`~/openclaw`, `~/projects/openclaw` जैसे सामान्य पथों की जाँच करके) और परिणाम को `~/.clawdock/config` में कैश करते हैं। यदि आपका checkout कहीं और है, तो `CLAWDOCK_DIR` स्वयं सेट करें।

## आपको क्या मिलता है

### बुनियादी संचालन

| कमांड            | विवरण            |
| ------------------ | ---------------------- |
| `clawdock-start`   | Gateway शुरू करें      |
| `clawdock-stop`    | Gateway रोकें       |
| `clawdock-restart` | Gateway पुनः शुरू करें    |
| `clawdock-status`  | कंटेनर की स्थिति जाँचें |
| `clawdock-logs`    | Gateway लॉग फ़ॉलो करें    |

### कंटेनर एक्सेस

| कमांड                   | विवरण                                   |
| ------------------------- | --------------------------------------------- |
| `clawdock-shell`          | Gateway कंटेनर के अंदर shell खोलें     |
| `clawdock-cli <command>`  | Docker में OpenClaw CLI कमांड चलाएँ           |
| `clawdock-exec <command>` | कंटेनर में कोई भी कमांड निष्पादित करें |

### वेब UI और पेयरिंग

| कमांड                 | विवरण                  |
| ----------------------- | ---------------------------- |
| `clawdock-dashboard`    | Control UI URL खोलें      |
| `clawdock-devices`      | लंबित डिवाइस पेयरिंग सूचीबद्ध करें |
| `clawdock-approve <id>` | पेयरिंग अनुरोध स्वीकृत करें    |

### सेटअप और रखरखाव

| कमांड              | विवरण                                       |
| -------------------- | ------------------------------------------------- |
| `clawdock-fix-token` | कंटेनर कॉन्फ़िगरेशन में Gateway टोकन लिखें |
| `clawdock-update`    | पुल करें, दोबारा बिल्ड करें और पुनः शुरू करें                        |
| `clawdock-rebuild`   | केवल Docker इमेज दोबारा बिल्ड करें                     |
| `clawdock-clean`     | कंटेनर और वॉल्यूम हटाएँ                     |

### उपयोगिताएँ

| कमांड                | विवरण                             |
| ---------------------- | --------------------------------------- |
| `clawdock-health`      | Gateway स्वास्थ्य जाँच चलाएँ              |
| `clawdock-token`       | Gateway टोकन प्रिंट करें                 |
| `clawdock-cd`          | OpenClaw प्रोजेक्ट डायरेक्टरी पर जाएँ  |
| `clawdock-config`      | `~/.openclaw` खोलें                      |
| `clawdock-show-config` | संपादित मानों के साथ कॉन्फ़िगरेशन फ़ाइलें प्रिंट करें |
| `clawdock-workspace`   | कार्यस्थान डायरेक्टरी खोलें            |
| `clawdock-help`        | सभी ClawDock कमांड सूचीबद्ध करें              |

## पहली बार की प्रक्रिया

```bash
clawdock-start
clawdock-fix-token
clawdock-dashboard
```

यदि ब्राउज़र बताता है कि पेयरिंग आवश्यक है:

```bash
clawdock-devices
clawdock-approve <request-id>
```

## कॉन्फ़िगरेशन और सीक्रेट

ClawDock दो अलग-अलग `.env` फ़ाइलें पढ़ता है, जो [Docker](/hi/install/docker) में वर्णित विभाजन के अनुरूप हैं:

- `docker-compose.yml` के पास स्थित प्रोजेक्ट `.env`: इमेज नाम, पोर्ट और `OPENCLAW_GATEWAY_TOKEN` जैसे Docker-विशिष्ट मान। `clawdock-token` यहाँ से टोकन पढ़ता है।
- `~/.openclaw/.env` (कंटेनर में माउंट की गई): env-समर्थित सीक्रेट, जिन्हें OpenClaw स्वयं `openclaw.json` और `agents/<agentId>/agent/auth-profiles.json` के साथ प्रबंधित करता है।

`clawdock-fix-token` प्रोजेक्ट `.env` से टोकन को कंटेनर के `gateway.remote.token` और `gateway.auth.token` कॉन्फ़िगरेशन मानों में कॉपी करता है और Gateway को पुनः शुरू करता है।

`openclaw.json` और दोनों `.env` फ़ाइलों की तुरंत जाँच करने के लिए `clawdock-show-config` का उपयोग करें; यह अपने प्रिंट किए गए आउटपुट में `.env` मानों को छिपा देता है।

## संबंधित

<CardGroup cols={2}>
  <Card title="Docker" href="/hi/install/docker" icon="docker">
    OpenClaw के लिए मानक Docker इंस्टॉलेशन।
  </Card>
  <Card title="Docker VM रनटाइम" href="/hi/install/docker-vm-runtime" icon="cube">
    सुदृढ़ आइसोलेशन के लिए Docker-प्रबंधित VM रनटाइम।
  </Card>
  <Card title="अपडेट करना" href="/hi/install/updating" icon="arrow-up-right-from-square">
    OpenClaw पैकेज और प्रबंधित सेवाओं को अपडेट करना।
  </Card>
</CardGroup>
