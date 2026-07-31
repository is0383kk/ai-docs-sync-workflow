---
read_when:
    - आप anthropic-vertex Plugin को इंस्टॉल, कॉन्फ़िगर या ऑडिट कर रहे हैं
summary: Google Vertex AI पर Claude मॉडल के लिए OpenClaw Anthropic Vertex प्रोवाइडर Plugin।
title: Anthropic Vertex Plugin
x-i18n:
    generated_at: "2026-07-27T18:17:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bd73b80b4e49a85cd6b1d8e47df6bf8d2d791c36a677124112f299027bfd9af5
    source_path: plugins/reference/anthropic-vertex.md
    workflow: 16
---

# Anthropic Vertex plugin

Google Vertex AI पर Claude मॉडल के लिए OpenClaw Anthropic Vertex प्रदाता plugin।

## वितरण

- पैकेज: `@openclaw/anthropic-vertex-provider`
- इंस्टॉल करने का माध्यम: npm; ClawHub

## उपलब्ध सतह

प्रदाता: `anthropic-vertex`

<!-- openclaw-plugin-reference:manual-start -->

## Claude Fable 5

जहाँ मॉडल आपके Google Cloud क्षेत्र में उपलब्ध हो, वहाँ `anthropic-vertex/claude-fable-5` का उपयोग करें।
Fable 5 हमेशा अनुकूली चिंतन का उपयोग करता है और डिफ़ॉल्ट रूप से `high` प्रयास का उपयोग करता है। `/think off` और
`/think minimal`, `low` प्रयास का उपयोग करते हैं क्योंकि मॉडल चिंतन को अक्षम करने का समर्थन नहीं करता।

## Claude Sonnet 5

Vertex के `global`, `us`, या `eu`
एंडपॉइंट के साथ `anthropic-vertex/claude-sonnet-5` का उपयोग करें। Sonnet 5 डिफ़ॉल्ट रूप से `high` प्रयास पर अनुकूली चिंतन का उपयोग करता है और
`/think off` या नेटिव `/think xhigh|max` स्तरों का समर्थन करता है। OpenClaw इसकी
1,000,000-टोकन संदर्भ विंडो और 128,000-टोकन आउटपुट सीमा को स्वचालित रूप से प्रकाशित करता है।

कैटलॉग मूल्य निर्धारण 31 अगस्त, 2026 तक प्रति दस लाख इनपुट/आउटपुट टोकन के लिए Vertex की `$2/$10` की प्रारंभिक वैश्विक दर का पालन करता है, फिर
1 सितंबर से `$3/$15` का। `us` और `eu` बहु-क्षेत्रीय एंडपॉइंट Vertex के दस्तावेज़ीकृत
10% प्रीमियम का उपयोग करते हैं।

<!-- openclaw-plugin-reference:manual-end -->
