---
read_when:
    - आप अनुमानित अनुवर्ती प्रतिबद्धताओं की जाँच करना चाहते हैं
    - आप लंबित चेक-इन खारिज करना चाहते हैं
    - आप ऑडिट कर रहे हैं कि Heartbeat क्या डिलीवर कर सकता है
summary: '`openclaw commitments` के लिए CLI संदर्भ (अनुमानित फ़ॉलो-अप का निरीक्षण करें और उन्हें खारिज करें)'
title: '`openclaw commitments`'
x-i18n:
    generated_at: "2026-07-27T17:50:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4a7c573daad6a9bc6ce4532514c8cc22b3c510b4fc0cf9d1a79048413f08c1a2
    source_path: cli/commitments.md
    workflow: 16
---

सेवानिवृत्त अनुमानित प्रतिबद्धता प्रयोग द्वारा छोड़े गए रिकॉर्ड का निरीक्षण करें और उन्हें खारिज करें।
OpenClaw अब नई प्रतिबद्धताएँ बनाता या वितरित नहीं करता, लेकिन रखरखाव
कमांड बनाए रखता है ताकि अपग्रेड मौजूदा SQLite पंक्तियों का ऑडिट और क्लीनअप कर सकें।

कोई उपकमांड न देने पर, `openclaw commitments` लंबित प्रतिबद्धताओं को सूचीबद्ध करता है।

## उपयोग

```bash
openclaw commitments [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments list [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments dismiss <id...> [--json]
```

## विकल्प

- `--all`: केवल लंबित प्रतिबद्धताओं के बजाय सभी स्थितियाँ दिखाएँ।
- `--agent <id>`: किसी एक एजेंट आईडी के अनुसार फ़िल्टर करें।
- `--status <status>`: स्थिति के अनुसार फ़िल्टर करें। मान: `pending`, `sent`,
  `dismissed`, `snoozed`, या `expired`। अज्ञात मान होने पर त्रुटि के साथ बाहर निकलता है।
- `--json`: मशीन-पठनीय JSON आउटपुट करें।

`dismiss` दिए गए प्रतिबद्धता आईडी को `dismissed` के रूप में चिह्नित करता है।

## उदाहरण

लंबित प्रतिबद्धताएँ सूचीबद्ध करें:

```bash
openclaw commitments
```

प्रत्येक संग्रहीत प्रतिबद्धता सूचीबद्ध करें:

```bash
openclaw commitments --all
```

किसी एक एजेंट के अनुसार फ़िल्टर करें:

```bash
openclaw commitments --agent main
```

स्नूज़ की गई प्रतिबद्धताएँ खोजें:

```bash
openclaw commitments --status snoozed
```

एक या अधिक प्रतिबद्धताएँ खारिज करें:

```bash
openclaw commitments dismiss cm_abc123 cm_def456
```

JSON के रूप में निर्यात करें:

```bash
openclaw commitments --all --json
```

## आउटपुट

टेक्स्ट आउटपुट प्रतिबद्धताओं की संख्या, साझा SQLite डेटाबेस पथ, कोई भी सक्रिय फ़िल्टर
और प्रत्येक प्रतिबद्धता के लिए एक पंक्ति प्रिंट करता है:

- प्रतिबद्धता आईडी
- स्थिति
- प्रकार (`event_check_in`, `deadline_check`, `care_check_in`, या `open_loop`)
- सबसे पहला नियत समय
- दायरा (एजेंट/चैनल/लक्ष्य)
- सुझाया गया चेक-इन टेक्स्ट

JSON आउटपुट में संख्या, सक्रिय स्थिति और एजेंट फ़िल्टर,
साझा SQLite डेटाबेस पथ और पूर्ण संग्रहीत रिकॉर्ड शामिल होते हैं।

## संबंधित

- [अनुमानित प्रतिबद्धताएँ](/hi/concepts/commitments)
- [मेमोरी अवलोकन](/hi/concepts/memory)
- [Heartbeat](/hi/gateway/heartbeat)
- [निर्धारित कार्य](/hi/automation/cron-jobs)
