---
read_when:
    - आप उस कॉन्फ़िगरेशन को अपग्रेड कर रहे हैं जिसमें अनुमानित प्रतिबद्धताओं का उपयोग किया गया था
    - आप पहले से संग्रहीत फ़ॉलो-अप रिकॉर्ड की जाँच करना या उन्हें खारिज करना चाहते हैं
sidebarTitle: Commitments
summary: समाप्त किए गए अनुमानित फ़ॉलो-अप दायित्वों के लिए स्थिति और क्लीनअप मार्गदर्शन
title: अनुमानित प्रतिबद्धताएँ
x-i18n:
    generated_at: "2026-07-27T17:57:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cfaa8c44be4ffb8db48279dba5347d4f598a193bfc4e244aeaed7a93e00ffb79
    source_path: concepts/commitments.md
    workflow: 16
---

अनुमानित प्रतिबद्धताओं का प्रयोग समाप्त कर दिया गया है। OpenClaw अब बातचीत से नई
अनुवर्ती कार्रवाइयाँ नहीं निकालता या उन्हें Heartbeat के माध्यम से वितरित नहीं करता, और पूर्व
`commitments` कॉन्फ़िगरेशन ब्लॉक को `openclaw doctor --fix` द्वारा हटा दिया जाता है।

सटीक रिमाइंडर और निर्धारित कार्यों के लिए
[निर्धारित कार्यों](/hi/automation/cron-jobs) का उपयोग जारी रहता है। बातचीत से जुड़े स्थायी तथ्य
[मेमोरी](/hi/concepts/memory) में होने चाहिए।

## मौजूदा रिकॉर्ड

पहले संग्रहीत प्रतिबद्धताएँ साझा SQLite स्टेट डेटाबेस में बनी रहती हैं, ताकि
अपग्रेड से ऑपरेटर को दिखाई देने वाला इतिहास नष्ट न हो। उन पंक्तियों का निरीक्षण करने या उन्हें खारिज करने के लिए लीगेसी रखरखाव
CLI का उपयोग करें:

```bash
openclaw commitments --all
openclaw commitments dismiss cm_abc123
```

रखरखाव कमांड के संदर्भ के लिए [`openclaw commitments`](/hi/cli/commitments)
देखें।

## संबंधित

- [निर्धारित कार्य](/hi/automation/cron-jobs)
- [मेमोरी का अवलोकन](/hi/concepts/memory)
- [Heartbeat](/hi/gateway/heartbeat)
