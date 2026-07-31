---
read_when:
    - आपको पुराने दस्तावेज़ों या रिलीज़ नोट्स में `openclaw flows` मिलता है
    - आपको TaskFlow के त्वरित निरीक्षण का संदर्भ चाहिए
summary: 'रीडायरेक्ट: प्रवाह कमांड `openclaw tasks flow` के अंतर्गत मौजूद हैं'
title: प्रवाह (रीडायरेक्ट)
x-i18n:
    generated_at: "2026-07-27T17:51:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 05d27154190d6087649612d81ce15f0cbc9459aa89ab22211582c18f4fc2943c
    source_path: cli/flows.md
    workflow: 16
---

# `openclaw tasks flow`

शीर्ष-स्तर पर कोई `openclaw flows` कमांड नहीं है। स्थायी TaskFlow निरीक्षण `openclaw tasks flow` के अंतर्गत उपलब्ध है।

## उपकमांड

```bash
openclaw tasks flow list   [--json] [--status <name>]
openclaw tasks flow show   <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

| उपकमांड | विवरण                | आर्ग्युमेंट / विकल्प                                                                   |
| ---------- | -------------------------- | ------------------------------------------------------------------------------------- |
| `list`     | ट्रैक किए गए TaskFlows की सूची दिखाएँ।    | `--json` मशीन-पठनीय आउटपुट; `--status <name>` फ़िल्टर (नीचे स्थिति मान देखें)। |
| `show`     | एक TaskFlow दिखाएँ।         | `<lookup>` प्रवाह आईडी या स्वामी कुंजी; `--json` मशीन-पठनीय आउटपुट।                    |
| `cancel`   | चल रहे TaskFlow को रद्द करें। | `<lookup>` प्रवाह आईडी या स्वामी कुंजी।                                                      |

`<lookup>` में प्रवाह आईडी (`list` / `show` द्वारा लौटाई गई) या प्रवाह की स्वामी कुंजी (प्रवाह को ट्रैक करने के लिए स्वामी उपप्रणाली द्वारा उपयोग किया जाने वाला स्थिर पहचानकर्ता), दोनों में से कोई भी स्वीकार्य है।

### स्थिति फ़िल्टर के मान

`list` पर `--status` इनमें से कोई एक मान स्वीकार करता है: `queued`, `running`, `waiting`, `blocked`, `succeeded`, `failed`, `cancelled`, `lost`।

## उदाहरण

```bash
openclaw tasks flow list
openclaw tasks flow list --status running
openclaw tasks flow list --json
openclaw tasks flow show flow_abc123
openclaw tasks flow show flow_abc123 --json
openclaw tasks flow cancel flow_abc123
```

TaskFlow की अवधारणाओं और लेखन के लिए, [TaskFlow](/hi/automation/taskflow) देखें। मूल `tasks` कमांड के लिए, [tasks CLI संदर्भ](/hi/cli/tasks) देखें।

## संबंधित

- [CLI संदर्भ](/hi/cli)
- [स्वचालन](/hi/automation)
- [TaskFlow](/hi/automation/taskflow)
