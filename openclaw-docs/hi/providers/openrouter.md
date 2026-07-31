---
read_when:
    - आप कई LLM के लिए एक ही API कुंजी चाहते हैं
    - आप OpenClaw में OpenRouter के माध्यम से मॉडल चलाना चाहते हैं
    - आप इमेज जनरेशन के लिए OpenRouter का उपयोग करना चाहते हैं
    - आप संगीत बनाने के लिए OpenRouter का उपयोग करना चाहते हैं
    - आप वीडियो जनरेशन के लिए OpenRouter का उपयोग करना चाहते हैं
summary: OpenClaw में कई मॉडल एक्सेस करने के लिए OpenRouter के एकीकृत API का उपयोग करें
title: OpenRouter
x-i18n:
    generated_at: "2026-07-27T18:31:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0936a10222f44f376dee081b7ee0678cddc3bc4579ac0006321dc1012d59bcf
    source_path: providers/openrouter.md
    workflow: 16
---

OpenRouter एक API और एक कुंजी के पीछे कई मॉडलों तक अनुरोध रूट करता है। यह
OpenAI-संगत है, इसलिए OpenClaw अन्य प्रॉक्सी प्रदाताओं के लिए उपयोग किए जाने वाले समान
`openai-completions`-शैली के ट्रांसपोर्ट पर इससे संचार करता है।

## आरंभ करना

<Tabs>
  <Tab title="OAuth">
    <Steps>
      <Step title="OAuth ऑनबोर्डिंग चलाएँ">
        ```bash
        openclaw onboard --auth-choice openrouter-oauth
        ```

        OpenClaw OpenRouter का ब्राउज़र साइन-इन प्रवाह (PKCE) खोलता है, कोड को
        OpenRouter API कुंजी से बदलता है और उसे डिफ़ॉल्ट OpenRouter प्रमाणीकरण
        प्रोफ़ाइल में संग्रहीत करता है। रिमोट/हेडलेस होस्ट पर, OpenClaw साइन-इन
        URL प्रिंट करता है और साइन इन करने के बाद आपसे रीडायरेक्ट URL पेस्ट करने को कहता है।
      </Step>
      <Step title="(वैकल्पिक) किसी विशिष्ट मॉडल पर स्विच करें">
        ऑनबोर्डिंग का डिफ़ॉल्ट `openrouter/auto` है। बाद में कोई निश्चित मॉडल चुनें:

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
  <Tab title="API कुंजी">
    <Steps>
      <Step title="अपनी API कुंजी प्राप्त करें">
        [openrouter.ai/keys](https://openrouter.ai/keys) पर एक API कुंजी बनाएँ।
      </Step>
      <Step title="API-कुंजी ऑनबोर्डिंग चलाएँ">
        ```bash
        openclaw onboard --auth-choice openrouter-api-key
        ```
      </Step>
      <Step title="(वैकल्पिक) किसी विशिष्ट मॉडल पर स्विच करें">
        ऑनबोर्डिंग का डिफ़ॉल्ट `openrouter/auto` है। बाद में कोई निश्चित मॉडल चुनें:

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## कॉन्फ़िगरेशन उदाहरण

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/auto" },
    },
  },
}
```

## मॉडल संदर्भ

<Note>
मॉडल संदर्भ `openrouter/<provider>/<model>` पैटर्न का पालन करते हैं। उपलब्ध प्रदाताओं और
मॉडलों की पूरी सूची के लिए, [/concepts/model-providers](/hi/concepts/model-providers) देखें।
</Note>

लाइव कैटलॉग खोज अनुपलब्ध होने पर उपयोग किए जाने वाले बंडल किए गए फ़ॉलबैक मॉडल:

| मॉडल संदर्भ                         | टिप्पणियाँ                        |
| --------------------------------- | ---------------------------- |
| `openrouter/auto`                 | OpenRouter स्वचालित रूटिंग |
| `openrouter/moonshotai/kimi-k2.6` | MoonshotAI के माध्यम से Kimi K2.6     |
| `openrouter/moonshotai/kimi-k2.5` | MoonshotAI के माध्यम से Kimi K2.5     |

`openrouter/openrouter/fusion` सहित कोई भी अन्य `openrouter/<provider>/<model>` संदर्भ
([Fusion राउटर](#fusion-router) देखें), OpenRouter के लाइव मॉडल कैटलॉग के विरुद्ध
गतिशील रूप से रिज़ॉल्व होता है।

## इमेज जनरेशन

OpenRouter `image_generate` टूल का बैकएंड बन सकता है। `agents.defaults.mediaModels.image` के अंतर्गत
एक OpenRouter इमेज मॉडल सेट करें:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openrouter/google/gemini-3.1-flash-image-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

OpenClaw `modalities: ["image", "text"]` के साथ OpenRouter के चैट-कम्प्लीशन्स इमेज API को
इमेज अनुरोध भेजता है। Gemini इमेज मॉडलों को OpenRouter के `image_config` के माध्यम से
`aspectRatio` और `resolution` संकेत भी मिलते हैं; अन्य
इमेज मॉडलों को नहीं मिलते। धीमे मॉडलों के लिए `agents.defaults.mediaModels.image.timeoutMs` का उपयोग करें;
`image_generate` टूल का प्रति-कॉल `timeoutMs` फिर भी प्राथमिकता लेता है।

## वीडियो जनरेशन

OpenRouter अपने एसिंक्रोनस `/videos` API के माध्यम से
`video_generate` टूल का बैकएंड बन सकता है। `agents.defaults.mediaModels.video` के अंतर्गत
एक OpenRouter वीडियो मॉडल सेट करें:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openrouter/google/veo-3.1-fast",
      },
    },
  },
}
```

OpenClaw टेक्स्ट-टू-वीडियो और इमेज-टू-वीडियो जॉब सबमिट करता है, लौटाए गए
`polling_url` को पोल करता है और पूर्ण वीडियो को OpenRouter के
`unsigned_urls` या जॉब सामग्री एंडपॉइंट से डाउनलोड करता है। संदर्भ इमेज डिफ़ॉल्ट रूप से
पहले/अंतिम फ़्रेम की इमेज होती हैं; `reference_image` से टैग की गई इमेज इसके बजाय इनपुट
संदर्भ के रूप में भेजी जाती हैं। बंडल किया गया `google/veo-3.1-fast` डिफ़ॉल्ट 4/6/8
सेकंड की अवधियों, `720P`/`1080P` रिज़ॉल्यूशन और `16:9`/`9:16` आस्पेक्ट रेशियो का समर्थन करता है।
वीडियो-टू-वीडियो समर्थित नहीं है: अपस्ट्रीम API केवल टेक्स्ट और इमेज
संदर्भ स्वीकार करता है।

## संगीत जनरेशन

OpenRouter चैट-कम्प्लीशन्स ऑडियो आउटपुट के माध्यम से `music_generate` टूल का
बैकएंड बन सकता है। `agents.defaults.mediaModels.music` के अंतर्गत
एक OpenRouter ऑडियो मॉडल सेट करें:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "openrouter/google/lyria-3-pro-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

बंडल किए गए OpenRouter संगीत प्रदाता का डिफ़ॉल्ट `google/lyria-3-pro-preview`
है और वह `google/lyria-3-clip-preview` भी उपलब्ध कराता है। OpenClaw `modalities:
["text", "audio"]` भेजता है, प्रतिक्रिया स्ट्रीम करता है, ऑडियो खंड एकत्र करता है और
परिणाम को चैनल डिलीवरी के लिए जनरेट किए गए मीडिया के रूप में सहेजता है। Lyria मॉडल साझा
`music_generate image=...` पैरामीटर के माध्यम से एक संदर्भ इमेज स्वीकार करते हैं।
स्ट्रीमिंग ऑडियो, ट्रांसक्रिप्ट प्रतिधारण और व्युत्पन्न SSE इवेंट एन्वलप
`agents.defaults.mediaMaxMb` से सीमित होते हैं (डिफ़ॉल्ट ऑडियो सीमा 16 MB है)।

## टेक्स्ट-टू-स्पीच

OpenRouter अपने OpenAI-संगत `/audio/speech` एंडपॉइंट के माध्यम से
TTS प्रदाता के रूप में कार्य कर सकता है।

```json5
{
  tts: {
    auto: "always",
    provider: "openrouter",
    providers: {
      openrouter: {
        model: "hexgrad/kokoro-82m",
        speakerVoice: "af_alloy",
        responseFormat: "mp3",
      },
    },
  },
}
```

यदि `tts.providers.openrouter.apiKey` को छोड़ दिया जाता है, तो TTS पहले
`models.providers.openrouter.apiKey`, फिर `OPENROUTER_API_KEY` पर फ़ॉलबैक करता है।

## स्पीच-टू-टेक्स्ट (इनबाउंड ऑडियो)

OpenRouter अपने STT एंडपॉइंट (`/audio/transcriptions`) का उपयोग करके साझा
`tools.media.audio` पथ के माध्यम से इनबाउंड वॉइस/ऑडियो अटैचमेंट को ट्रांसक्राइब कर सकता है।
यह ऐसे किसी भी चैनल Plugin पर लागू होता है जो इनबाउंड वॉइस/ऑडियो को
मीडिया समझ प्रीफ़्लाइट में फ़ॉरवर्ड करता है।

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "openrouter", model: "openai/whisper-large-v3-turbo" }],
      },
    },
  },
}
```

OpenClaw OpenRouter STT अनुरोधों को `input_audio` के अंतर्गत base64 ऑडियो वाले
JSON के रूप में भेजता है (OpenRouter का STT अनुबंध), multipart OpenAI फ़ॉर्म
अपलोड के रूप में नहीं।

## Fusion राउटर

OpenRouter Fusion एक OpenClaw मॉडल संदर्भ को समानांतर रूप से कई OpenRouter मॉडलों को
भेजता है, OpenRouter से उनके उत्तरों का मूल्यांकन करवाता है और सामान्य OpenRouter
एंडपॉइंट के माध्यम से एक अंतिम प्रतिक्रिया लौटाता है। अपस्ट्रीम मॉडल स्लग
`openrouter/fusion` है, इसलिए OpenClaw मॉडल संदर्भ में OpenClaw
प्रदाता प्रीफ़िक्स और अपस्ट्रीम OpenRouter नेमस्पेस दोनों होते हैं:

```bash
openclaw models set openrouter/openrouter/fusion
```

मॉडल के `params.extraBody` के माध्यम से Fusion के पैनल और निर्णायक को कॉन्फ़िगर करें;
वे फ़ील्ड सीधे OpenRouter चैट-कम्प्लीशन्स अनुरोध
बॉडी में फ़ॉरवर्ड होते हैं। Fusion OAuth या API-कुंजी ऑनबोर्डिंग, दोनों के साथ काम करता है;
यदि आप OAuth का उपयोग करते हैं, तो नीचे दी गई `env.OPENROUTER_API_KEY` पंक्ति छोड़ दें।

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/openrouter/fusion" },
      models: {
        "openrouter/openrouter/fusion": {
          params: {
            extraBody: {
              plugins: [
                {
                  id: "fusion",
                  analysis_models: [
                    "google/gemini-3.5-flash",
                    "moonshotai/kimi-k2.6",
                    "deepseek/deepseek-v4-pro",
                  ],
                  model: "google/gemini-3.5-flash",
                },
              ],
            },
          },
        },
      },
    },
  },
}
```

`analysis_models` समानांतर पैनल है; Fusion Plugin कॉन्फ़िगरेशन के भीतर
`model` निर्णायक मॉडल है। सामान्य एजेंट/चैट टर्न में Fusion को बाध्य करने के लिए
शीर्ष-स्तरीय `tool_choice` को `"required"` पर सेट न करें: OpenClaw टर्न में
उसकी अपनी टूल परिभाषाएँ शामिल हो सकती हैं और शीर्ष-स्तरीय आवश्यक टूल चयन Fusion राउटर
के बजाय उनमें से किसी एक को चुन सकता है। जब यह Fusion Plugin कॉन्फ़िगरेशन मौजूद होता है,
OpenClaw कॉन्फ़िगर किए गए विश्लेषण मॉडलों और निर्णायक मॉडल को सूचीबद्ध करने वाला
एक सैनिटाइज़ किया गया सिस्टम-प्रॉम्प्ट नोट जोड़ता है, ताकि एजेंट अपने Fusion पैनल के बारे में
प्रश्नों का उत्तर दे सके। अन्य `extraBody` फ़ील्ड प्रॉम्प्ट में कॉपी नहीं किए जाते।

Fusion डिज़ाइन के अनुसार धीमा है: OpenRouter प्रॉम्प्ट को कई
विश्लेषण मॉडलों में वितरित करता है, फिर निर्णायक/संश्लेषण चरण चलाता है, इसलिए विलंबता
सीधे एकल-मॉडल अनुरोध से अधिक होती है। इसका उपयोग सोच-विचार वाले, उच्च-गुणवत्ता उत्तरों या
एस्केलेशन पथों के लिए करें, विलंबता-संवेदी डिफ़ॉल्ट के रूप में नहीं। पैनल को छोटा रखें और
शीघ्र प्रतिक्रियाओं के लिए तेज़ विश्लेषण/निर्णायक मॉडल चुनें।

कॉन्फ़िगर किए गए संदर्भ को एक बार की स्थानीय कॉल से जाँचें:

```bash
openclaw infer model run --local \
  --model openrouter/openrouter/fusion \
  --prompt "ठीक इसी तरह उत्तर दें: FUSION_OK" \
  --json
```

## प्रमाणीकरण और हेडर

OpenRouter आपकी API कुंजी से Bearer टोकन का उपयोग करता है। OpenRouter OAuth एक PKCE
लॉगिन प्रवाह है जो OpenRouter API कुंजी जारी करता है, इसलिए OpenClaw परिणाम को
मैन्युअल API-कुंजी सेटअप में उपयोग की जाने वाली उसी `openrouter:default` API-कुंजी
प्रमाणीकरण प्रोफ़ाइल में संग्रहीत करता है।

पूर्ण ऑनबोर्डिंग दोबारा चलाए बिना किसी मौजूदा इंस्टॉलेशन पर साइन इन करने या संग्रहीत कुंजी
बदलने के लिए:

```bash
openclaw models auth login --provider openrouter --method oauth
openclaw models auth login --provider openrouter --method api-key
```

सत्यापित OpenRouter अनुरोधों (`https://openrouter.ai/api/v1`) पर, OpenClaw
OpenRouter के दस्तावेज़ीकृत ऐप-एट्रिब्यूशन हेडर जोड़ता है:

| हेडर                    | मान                                                                                                  |
| ------------------------- | ------------------------------------------------------------------------------------------------------ |
| `HTTP-Referer`            | `https://openclaw.ai`                                                                                  |
| `X-OpenRouter-Title`      | `OpenClaw`                                                                                             |
| `X-OpenRouter-Categories` | `cli-agent,cloud-agent,programming-app,creative-writing,writing-assistant,general-chat,personal-agent` |

<Warning>
यदि आप OpenRouter प्रदाता को किसी अन्य प्रॉक्सी या बेस URL की ओर इंगित करते हैं, तो OpenClaw
उन OpenRouter-विशिष्ट हेडर या Anthropic कैश मार्कर को इंजेक्ट **नहीं** करता।
</Warning>

## उन्नत कॉन्फ़िगरेशन

<AccordionGroup>
  <Accordion title="प्रतिक्रिया कैशिंग">
    OpenRouter प्रतिक्रिया कैशिंग ऑप्ट-इन है। इसे प्रत्येक मॉडल के लिए सक्षम करें:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/auto": {
              params: {
                responseCache: true,
                responseCacheTtlSeconds: 300,
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw `X-OpenRouter-Cache: true` और, कॉन्फ़िगर होने पर,
    `X-OpenRouter-Cache-TTL` भेजता है। `responseCacheClear: true` वर्तमान अनुरोध के लिए
    रीफ़्रेश को बाध्य करता है और प्रतिस्थापन प्रतिक्रिया संग्रहीत करता है। Snake_case
    उपनाम (`response_cache`, `response_cache_ttl_seconds`,
    `response_cache_clear`) स्वीकार किए जाते हैं, और `Seconds` प्रत्यय के बिना
    `responseCacheTtl` / `response_cache_ttl` भी स्वीकार किए जाते हैं।

    यह प्रदाता प्रॉम्प्ट कैशिंग और OpenRouter के Anthropic
    `cache_control` मार्कर से अलग है। यह केवल सत्यापित
    `openrouter.ai` रूट पर लागू होता है, कस्टम प्रॉक्सी बेस URL पर नहीं।

  </Accordion>

  <Accordion title="Anthropic कैश मार्कर">
    सत्यापित OpenRouter रूट पर, Anthropic मॉडल संदर्भ सिस्टम/डेवलपर प्रॉम्प्ट ब्लॉक पर
    बेहतर प्रॉम्प्ट-कैश पुनः उपयोग के लिए OpenRouter के Anthropic
    `cache_control` मार्कर बनाए रखते हैं।
  </Accordion>

  <Accordion title="Anthropic रीजनिंग प्रीफ़िल">
    सत्यापित OpenRouter रूट पर, रीजनिंग सक्षम वाले Anthropic मॉडल संदर्भ अनुरोध के
    OpenRouter तक पहुँचने से पहले अंतिम असिस्टेंट प्रीफ़िल टर्न हटा देते हैं, जो Anthropic
    की इस आवश्यकता से मेल खाता है कि रीजनिंग वार्तालाप उपयोगकर्ता टर्न पर समाप्त हों।
  </Accordion>

  <Accordion title="थिंकिंग / रीजनिंग इंजेक्शन">
    समर्थित गैर-`auto` रूटों पर, OpenClaw चयनित थिंकिंग स्तर को
    OpenRouter प्रॉक्सी रीजनिंग पेलोड में मैप करता है। `openrouter/auto` और असमर्थित
    मॉडल संकेत उस इंजेक्शन को छोड़ देते हैं। पुराने `openrouter/hunter-alpha` संदर्भ भी
    इसे छोड़ देते हैं, क्योंकि OpenRouter उस सेवानिवृत्त रूट पर रीजनिंग
    फ़ील्ड में अंतिम उत्तर का टेक्स्ट लौटा सकता था।
  </Accordion>

  <Accordion title="DeepSeek V4 रीजनिंग रीप्ले">
    सत्यापित OpenRouter रूटों पर, `openrouter/deepseek/deepseek-v4-flash` और
    `openrouter/deepseek/deepseek-v4-pro` रीप्ले किए गए असिस्टेंट टर्न में अनुपस्थित `reasoning_content` को
    भरते हैं, जिससे थिंकिंग/टूल वार्तालाप DeepSeek
    V4 के आवश्यक फ़ॉलो-अप स्वरूप में बने रहते हैं। OpenClaw इन रूटों के लिए OpenRouter-समर्थित
    `reasoning.effort` मान भेजता है: `xhigh`/`max` को `xhigh` से मैप किया जाता है,
    अन्य प्रत्येक गैर-ऑफ़ स्तर को `high` से मैप किया जाता है।
  </Accordion>

  <Accordion title="केवल OpenAI के लिए अनुरोध का स्वरूप निर्धारण">
    OpenRouter प्रॉक्सी-शैली वाले OpenAI-संगत पथ से चलता है, इसलिए केवल मूल
    OpenAI के लिए अनुरोध का स्वरूप निर्धारण, जैसे `serviceTier`, Responses `store`,
    OpenAI रीजनिंग-संगतता पेलोड और प्रॉम्प्ट-कैश संकेत फ़ॉरवर्ड नहीं किए जाते।
  </Accordion>

  <Accordion title="Gemini-समर्थित रूट">
    Gemini-समर्थित OpenRouter संदर्भ प्रॉक्सी-Gemini पथ पर बने रहते हैं: OpenClaw वहाँ
    Gemini थॉट-सिग्नेचर सैनिटाइज़ेशन बनाए रखता है, लेकिन मूल
    Gemini रीप्ले सत्यापन या बूटस्ट्रैप रीराइट सक्षम नहीं करता।
  </Accordion>

  <Accordion title="प्रोवाइडर रूटिंग मेटाडेटा">
    OpenRouter अंतर्निहित प्रोवाइडर रूटिंग के लिए एक `provider` अनुरोध ऑब्जेक्ट का
    समर्थन करता है। सभी OpenRouter टेक्स्ट-मॉडल अनुरोधों के लिए
    `models.providers.openrouter.params.provider` के साथ एक डिफ़ॉल्ट नीति कॉन्फ़िगर करें:

    ```json5
    {
      models: {
        providers: {
          openrouter: {
            params: {
              provider: {
                sort: "latency",
                require_parameters: true,
                data_collection: "deny",
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw उस ऑब्जेक्ट को अनुरोध के `provider`
    पेलोड के रूप में OpenRouter को फ़ॉरवर्ड करता है। OpenRouter के प्रलेखित snake_case फ़ील्ड का उपयोग करें, जिनमें `sort`,
    `only`, `ignore`, `order`, `allow_fallbacks`, `require_parameters`,
    `data_collection`, `quantizations`, `max_price`, `preferred_max_latency`,
    `preferred_min_throughput`, `zdr`, और `enforce_distillable_text` शामिल हैं।

    प्रति-मॉडल पैरामीटर, प्रोवाइडर-व्यापी रूटिंग ऑब्जेक्ट को ओवरराइड करते हैं:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/anthropic/claude-sonnet-4-6": {
              params: {
                provider: {
                  order: ["anthropic"],
                  allow_fallbacks: false,
                },
              },
            },
          },
        },
      },
    }
    ```

    यह केवल OpenRouter चैट-कम्प्लीशंस रूटों पर लागू होता है। प्रत्यक्ष Anthropic,
    Google, OpenAI या कस्टम प्रोवाइडर रूट OpenRouter रूटिंग पैरामीटर की उपेक्षा करते हैं।

  </Accordion>
</AccordionGroup>

## संबंधित

<CardGroup cols={2}>
  <Card title="मॉडल चयन" href="/hi/concepts/model-providers" icon="layers">
    प्रोवाइडर, मॉडल संदर्भ और फ़ेलओवर व्यवहार चुनना।
  </Card>
  <Card title="कॉन्फ़िगरेशन संदर्भ" href="/hi/gateway/configuration-reference" icon="gear">
    एजेंट, मॉडल और प्रोवाइडर के लिए पूर्ण कॉन्फ़िगरेशन संदर्भ।
  </Card>
</CardGroup>
