---
read_when:
    - वर्कस्पेस को मैन्युअल रूप से बूटस्ट्रैप करना
summary: HEARTBEAT.md के लिए कार्यक्षेत्र टेम्पलेट
title: HEARTBEAT.md टेम्पलेट
x-i18n:
    generated_at: "2026-07-27T20:30:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d5b02cd62708a87515c4ae59bd2ffab3e4c8ebf81f4126fdd43ced756241b151
    source_path: reference/templates/HEARTBEAT.md
    workflow: 16
---

# HEARTBEAT.md टेम्पलेट

`HEARTBEAT.md` एजेंट कार्यक्षेत्र में स्थित होती है और आवधिक Heartbeat जाँच-सूची रखती है। OpenClaw द्वारा Heartbeat मॉडल कॉल को पूरी तरह छोड़ने के लिए इसे खाली रखें, या इसमें केवल रिक्त स्थान, Markdown टिप्पणियाँ, ATX शीर्षक, खाली सूची स्टब (`- `, `* [ ]`), या फ़ेंस मार्कर रखें (`reason=empty-heartbeat-file`)।

प्रदत्त डिफ़ॉल्ट सामग्री:

```markdown
<!-- Heartbeat template; comments-only content prevents scheduled heartbeat API calls. -->

# Heartbeat API कॉल छोड़ने के लिए इस फ़ाइल को खाली रखें (या इसमें केवल टिप्पणियाँ रखें)।

# जब Heartbeat को साझा संदर्भ की जाँच करनी हो, तब नीचे एक छोटी जाँच-सूची जोड़ें।
```

टिप्पणी पंक्तियों के नीचे छोटी जाँच-सूची केवल तभी जोड़ें, जब Heartbeat के एक चक्र में सभी मदों की एक साथ जाँच करनी हो। इसे छोटा रखें: Heartbeat का प्रत्येक टिक इस फ़ाइल को पढ़ता है (डिफ़ॉल्ट रूप से हर 30 मिनट में), इसलिए फूले हुए निर्देश हर बार सक्रिय होने पर टोकन खर्च करते हैं।

स्वतंत्र रूप से निर्धारित या केवल नियत समय पर की जाने वाली जाँचों के लिए, [Cron जॉब](/hi/automation/cron-jobs) बनाएँ। Heartbeat स्क्रैच अब शेड्यूलर सिंटैक्स का समर्थन नहीं करता। पुराने `tasks:` ब्लॉक को रूपांतरित करने के लिए `openclaw doctor --fix` चलाएँ।

## संबंधित

- [Heartbeat](/hi/gateway/heartbeat)
- [Heartbeat कॉन्फ़िगरेशन](/hi/gateway/config-agents)
