---
read_when:
    - आप acpx Plugin को इंस्टॉल, कॉन्फ़िगर या ऑडिट कर रहे हैं
summary: Plugin-स्वामित्व वाले सत्र और ट्रांसपोर्ट प्रबंधन सहित OpenClaw ACP रनटाइम बैकएंड।
title: ACPx Plugin
x-i18n:
    generated_at: "2026-07-27T18:45:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9816ca3ada81eb44883b641f3d761b76f894bd83c8aa978c516125c77842f664
    source_path: plugins/reference/acpx.md
    workflow: 16
---

# ACPx plugin

Plugin के स्वामित्व वाले सत्र और ट्रांसपोर्ट प्रबंधन सहित OpenClaw ACP रनटाइम बैकएंड।

## वितरण

- पैकेज: `@openclaw/acpx`
- इंस्टॉल मार्ग: npm; ClawHub

## सतह

कौशल

<!-- openclaw-plugin-reference:manual-start -->

## Pi के नेटिव सत्र

बंडल किया गया रनटाइम Gateway और युग्मित
नोड पर Pi के सत्र स्टोर का स्वतः पता लगाता है। संग्रहीत सत्र **Pi** सत्र-साइडबार समूह में दिखाई देते हैं, जहाँ
Pi के प्रलेखित JSONL सत्र प्रारूप से केवल-पढ़ने योग्य ट्रांस्क्रिप्ट ब्राउज़िंग उपलब्ध होती है।
कैटलॉग प्रोजेक्ट और वैश्विक `settings.json` सत्र डायरेक्टरी के साथ-साथ
`PI_CODING_AGENT_DIR` और `PI_CODING_AGENT_SESSION_DIR` का भी पालन करता है। सापेक्ष पथ
उनकी `settings.json` फ़ाइल वाली डायरेक्टरी से हल किए जाते हैं।

खोज को अक्षम करने के लिए **Config > Plugins > ACPX Runtime** के अंतर्गत
**Pi Session Catalog** को बंद करें। यह डिफ़ॉल्ट रूप से सक्षम है।

<!-- openclaw-plugin-reference:manual-end -->

## संबंधित दस्तावेज़

- [acpx](/hi/tools/acp-agents-setup)
