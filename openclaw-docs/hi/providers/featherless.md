---
read_when:
    - आप OpenClaw के साथ Featherless AI का उपयोग करना चाहते हैं
    - आपको Featherless API कुंजी के एनवायरनमेंट वेरिएबल या मॉडल रेफ़रेंस फ़ॉर्मैट की आवश्यकता है
summary: Featherless AI सेटअप, मॉडल चयन और टूल कॉलिंग
title: Featherless AI
x-i18n:
    generated_at: "2026-07-27T18:24:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9112f7e65b4089bf96933c632d0b62f7fb87d42998d985ca85eb92dc392636b6
    source_path: providers/featherless.md
    workflow: 16
---

[Featherless AI](https://featherless.ai) एक
OpenAI-संगत API के माध्यम से ओपन मॉडल उपलब्ध कराता है। OpenClaw, Featherless को एक आधिकारिक बाहरी
प्रदाता Plugin के रूप में इंस्टॉल करता है और रनटाइम पर Featherless से सटीक
मॉडल आईडी स्वीकार करते हुए अंतर्निहित कैटलॉग को छोटा रखता है।

| प्रॉपर्टी        | मान                                    |
| --------------- | ---------------------------------------- |
| प्रदाता आईडी     | `featherless`                            |
| पैकेज         | `@openclaw/featherless-provider`         |
| प्रमाणीकरण एनवायरनमेंट वेरिएबल    | `FEATHERLESS_API_KEY`                    |
| ऑनबोर्डिंग फ़्लैग | `--auth-choice featherless-api-key`      |
| प्रत्यक्ष CLI फ़्लैग | `--featherless-api-key <key>`            |
| API             | OpenAI-संगत (`openai-completions`) |
| बेस URL        | `https://api.featherless.ai/v1`          |
| डिफ़ॉल्ट मॉडल   | `featherless/Qwen/Qwen3-32B`             |

## सेटअप

Plugin इंस्टॉल करें और Gateway को पुनः प्रारंभ करें:

```bash
openclaw plugins install @openclaw/featherless-provider
openclaw gateway restart
```

ऑनबोर्डिंग चलाएँ:

```bash
openclaw onboard --auth-choice featherless-api-key
```

गैर-इंटरैक्टिव सेटअप के लिए:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice featherless-api-key \
  --featherless-api-key "$FEATHERLESS_API_KEY"
```

या कुंजी को Gateway प्रक्रिया के लिए उपलब्ध कराएँ:

```bash
export FEATHERLESS_API_KEY="<your-featherless-api-key>" # pragma: allowlist secret
```

प्रदाता सत्यापित करें:

```bash
openclaw models list --provider featherless
```

## डिफ़ॉल्ट मॉडल

Plugin सेटअप के डिफ़ॉल्ट के रूप में `Qwen/Qwen3-32B` का उपयोग करता है, क्योंकि Featherless
Qwen 3 परिवार के लिए नेटिव टूल कॉलिंग का दस्तावेज़ीकरण करता है। OpenClaw इसकी
32,768-टोकन संदर्भ विंडो, 4,096-टोकन की सतर्क आउटपुट सीमा और
Qwen चैट-टेम्पलेट थिंकिंग नियंत्रण कॉन्फ़िगर करता है।

कैटलॉग के लागत फ़ील्ड शून्य हैं, क्योंकि Featherless कई बिलिंग
मोड का समर्थन करता है और OpenClaw खाते-विशिष्ट प्लान या प्रति-अनुरोध मूल्य-निर्धारण
दरें एम्बेड नहीं करता।

## Featherless के अन्य मॉडल

`featherless/` प्रदाता प्रीफ़िक्स के बाद सटीक Featherless मॉडल आईडी का उपयोग करें:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "featherless/moonshotai/Kimi-K2-Instruct",
      },
    },
  },
}
```

OpenClaw जानबूझकर Featherless की पूरी सार्वजनिक मॉडल इंडेक्स को
चयनकर्ता में कॉपी नहीं करता। इंडेक्स बड़ा है और प्रत्येक टेक्स्ट, विज़न, एम्बेडिंग और रीजनिंग मॉडल को सुरक्षित रूप से
वर्गीकृत करने के लिए पर्याप्त संरचित क्षमता
मेटाडेटा उपलब्ध नहीं कराता।
इसलिए अज्ञात आईडी सतर्क केवल-टेक्स्ट, गैर-रीजनिंग
डिफ़ॉल्ट के साथ रिज़ॉल्व होती हैं: 4,096-टोकन संदर्भ विंडो और 1,024-टोकन आउटपुट सीमा।

जब किसी मॉडल को अलग मेटाडेटा चाहिए, तो स्पष्ट प्रदाता मॉडल प्रविष्टि जोड़ें:

```json5
{
  models: {
    mode: "merge",
    providers: {
      featherless: {
        baseUrl: "https://api.featherless.ai/v1",
        apiKey: "${FEATHERLESS_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "google/gemma-3-27b-it",
            name: "Gemma 3 27B",
            input: ["text", "image"],
            reasoning: false,
            contextWindow: 32768,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

कस्टम मेटाडेटा जोड़ने से पहले वर्तमान मॉडल उपलब्धता और क्षमता
टैग के लिए Featherless का मॉडल कैटलॉग जाँचें।

## समस्या निवारण

- `401` या `403`: पुष्टि करें कि `FEATHERLESS_API_KEY` Gateway
  प्रक्रिया को दिखाई देता है, या ऑनबोर्डिंग फिर से चलाएँ।
- अज्ञात मॉडल: Featherless से प्राप्त सटीक केस-सेंसिटिव आईडी का
  `featherless/` प्रीफ़िक्स के बाद उपयोग करें।
- टूल कॉल टेक्स्ट के रूप में लौटे: ऐसा मॉडल परिवार चुनें जिसके लिए Featherless
  नेटिव फ़ंक्शन कॉलिंग का दस्तावेज़ीकरण करता है, जैसे Qwen 3।
- प्रबंधित Gateway कुंजी नहीं देख सकता: इसे `~/.openclaw/.env` या सेवा द्वारा लोड किए गए किसी अन्य
  एनवायरनमेंट स्रोत में रखें, फिर Gateway को पुनः प्रारंभ करें।

## संबंधित

- [मॉडल प्रदाता](/hi/concepts/model-providers)
- [सभी प्रदाता](/hi/providers/index)
- [थिंकिंग मोड](/hi/tools/thinking)
