---
read_when:
    - आप शीर्ष ओपन सोर्स LLM के लिए एक ही API कुंजी चाहते हैं
    - आप OpenClaw में DeepInfra के API के ज़रिए मॉडल चलाना चाहते हैं
summary: OpenClaw में सबसे लोकप्रिय ओपन-सोर्स और अग्रणी मॉडल तक पहुँचने के लिए DeepInfra की एकीकृत API का उपयोग करें
title: DeepInfra
x-i18n:
    generated_at: "2026-07-27T19:49:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a63bdd4ffd2189cde50f0ee601fd7ee32ca86c943a9899072f0c140823608004
    source_path: providers/deepinfra.md
    workflow: 16
---

DeepInfra लोकप्रिय ओपन सोर्स और फ़्रंटियर मॉडल के अनुरोधों को एक
OpenAI-संगत एंडपॉइंट और API कुंजी के पीछे रूट करता है। अधिकांश OpenAI SDK
बेस URL बदलने पर इसके साथ काम करते हैं।

## Plugin इंस्टॉल करें

```bash
openclaw plugins install @openclaw/deepinfra-provider
openclaw gateway restart
```

## API कुंजी प्राप्त करें

1. [deepinfra.com](https://deepinfra.com/) पर साइन इन करें
2. Dashboard / Keys पर जाएँ और एक कुंजी जनरेट करें, या अपने-आप बनाई गई कुंजी का उपयोग करें

## CLI सेटअप

```bash
openclaw onboard --deepinfra-api-key <key>
```

या एनवायरनमेंट वेरिएबल सेट करें:

```bash
export DEEPINFRA_API_KEY="<your-deepinfra-api-key>" # pragma: allowlist secret
```

## कॉन्फ़िगरेशन स्निपेट

```json5
{
  env: { DEEPINFRA_API_KEY: "<your-deepinfra-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "deepinfra/deepseek-ai/DeepSeek-V4-Flash" },
    },
  },
}
```

## समर्थित सरफ़ेस

`DEEPINFRA_API_KEY` कॉन्फ़िगर होने के बाद चैट, इमेज जनरेशन और वीडियो जनरेशन
अपने मॉडल कैटलॉग को `https://api.deepinfra.com/v1/openai/models?sort_by=openclaw&filter=with_meta` से
लाइव रीफ़्रेश करते हैं। लाइव डिस्कवरी चयन योग्य मॉडलों की सूची का विस्तार
करती है; प्रत्येक सरफ़ेस का डिफ़ॉल्ट मॉडल नीचे दिया गया स्थिर मान ही रहता है।
अन्य सरफ़ेस उसी लाइव कैटलॉग पर स्थानांतरित होने तक स्थिर कैटलॉग का उपयोग करते हैं।

| सरफ़ेस                  | डिफ़ॉल्ट मॉडल                                                                  | OpenClaw कॉन्फ़िगरेशन/टूल                                  |
| ------------------------ | ------------------------------------------------------------------------------ | ----------------------------------------------------- |
| चैट / मॉडल प्रोवाइडर    | `deepseek-ai/DeepSeek-V4-Flash` (लाइव कैटलॉग और चैट मॉडल जोड़ता है)           | `agents.defaults.model`                               |
| इमेज जनरेशन/एडिटिंग | `black-forest-labs/FLUX-1-schnell` (लाइव कैटलॉग और `image-gen` मॉडल जोड़ता है) | `image_generate`, `agents.defaults.mediaModels.image` |
| मीडिया की समझ      | इमेज के लिए `moonshotai/Kimi-K2.5`                                              | इनबाउंड इमेज की समझ                           |
| स्पीच-टू-टेक्स्ट           | `openai/whisper-large-v3-turbo`                                                | इनबाउंड ऑडियो ट्रांसक्रिप्शन                           |
| टेक्स्ट-टू-स्पीच           | `hexgrad/Kokoro-82M`                                                           | `tts.provider: "deepinfra"`                           |
| वीडियो जनरेशन         | `Pixverse/Pixverse-T2V` (लाइव कैटलॉग और `video-gen` मॉडल जोड़ता है)            | `video_generate`, `agents.defaults.mediaModels.video` |
| मेमोरी एम्बेडिंग        | `BAAI/bge-m3`                                                                  | `memory.search.provider: "deepinfra"`                 |

DeepInfra री-रैंकिंग, वर्गीकरण, ऑब्जेक्ट डिटेक्शन और अन्य
नेटिव मॉडल प्रकार भी उपलब्ध कराता है। OpenClaw में अभी उन श्रेणियों के लिए
कोई प्रोवाइडर कॉन्ट्रैक्ट नहीं है, इसलिए यह Plugin उन्हें रजिस्टर नहीं करता।

## उपलब्ध मॉडल

कुंजी कॉन्फ़िगर होने के बाद OpenClaw DeepInfra मॉडल को डायनेमिक रूप से खोजता है।
वर्तमान सूची देखने के लिए `/models deepinfra` या
`openclaw models list --provider deepinfra` का उपयोग करें।

[deepinfra.com](https://deepinfra.com/) पर उपलब्ध कोई भी मॉडल
`deepinfra/` प्रीफ़िक्स के साथ काम करता है:

```text
deepinfra/deepseek-ai/DeepSeek-V4-Flash
deepinfra/deepseek-ai/DeepSeek-V3.2
deepinfra/MiniMaxAI/MiniMax-M2.5
deepinfra/moonshotai/Kimi-K2.5
deepinfra/nvidia/NVIDIA-Nemotron-3-Super-120B-A12B
deepinfra/zai-org/GLM-5.1
...और कई अन्य
```

## टिप्पणियाँ

- मॉडल रेफ़रेंस `deepinfra/<provider>/<model>` होते हैं (उदाहरण के लिए `deepinfra/Qwen/Qwen3-Max`)।
- डिफ़ॉल्ट चैट मॉडल: `deepinfra/deepseek-ai/DeepSeek-V4-Flash`
- बेस URL: `https://api.deepinfra.com/v1/openai`
- वीडियो जनरेशन OpenAI-संगत एसिंक्रोनस एंडपॉइंट `https://api.deepinfra.com/v1/openai/videos` का उपयोग करता है (सबमिट करें, फिर पोल करें)। कॉन्फ़िगर किए गए `baseUrl` का पालन किया जाता है। `openclaw doctor --fix`, `api.deepinfra.com` पर लेगेसी `nativeBaseUrl` या `/v1/inference` मानों को अपने-आप `baseUrl` में माइग्रेट करता है; कस्टम नेटिव एंडपॉइंट को डॉक्टर सूचना के साथ हटा दिया जाता है और उनके लिए मैन्युअल रूप से कॉन्फ़िगर किया गया OpenAI-संगत `baseUrl` आवश्यक है। जब तक `baseUrl` हटाए गए `/v1/inference` सरफ़ेस को लक्षित करता है, वीडियो जनरेशन कोई अनुरोध भेजने से पहले कार्रवाई योग्य त्रुटि के साथ विफल हो जाता है।

## संबंधित

- [मॉडल प्रोवाइडर](/hi/concepts/model-providers)
- [सभी प्रोवाइडर](/hi/providers/index)
