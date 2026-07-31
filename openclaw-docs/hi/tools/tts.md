---
read_when:
    - जवाबों के लिए टेक्स्ट-टू-स्पीच सक्षम करना
    - TTS प्रदाता, फ़ॉलबैक शृंखला या व्यक्तित्व को कॉन्फ़िगर करना
    - /tts कमांड या निर्देशों का उपयोग करना
sidebarTitle: Text to speech (TTS)
summary: आउटबाउंड उत्तरों के लिए टेक्स्ट-टू-स्पीच — प्रदाता, व्यक्तित्व, स्लैश कमांड और प्रति-चैनल आउटपुट
title: टेक्स्ट-टू-स्पीच
x-i18n:
    generated_at: "2026-07-27T20:40:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2ae9d0cc6f77c6a8b1b379c3712fd92fbbc22dae694ecdd46a0bb35cec0d29e7
    source_path: tools/tts.md
    workflow: 16
---

OpenClaw आउटबाउंड उत्तरों को **14 स्पीच प्रोवाइडर** के माध्यम से ऑडियो में बदलता है:
Feishu, Matrix, Telegram और WhatsApp पर नेटिव वॉइस मैसेज; अन्य सभी जगहों पर ऑडियो
अटैचमेंट; और टेलीफ़ोनी तथा Talk के लिए PCM/Ulaw स्ट्रीम।

TTS, Talk के `stt-tts` मोड का स्पीच-आउटपुट भाग है (`talk.speak` कॉल भी इसी
सिंथेसिस पथ का उपयोग करती हैं)। प्रोवाइडर-नेटिव `realtime` Talk सेशन
रीयलटाइम प्रोवाइडर के भीतर स्पीच सिंथेसाइज़ करते हैं; `transcription` सेशन कभी भी
असिस्टेंट का वॉइस उत्तर सिंथेसाइज़ नहीं करते।

## तुरंत शुरू करें

<Steps>
  <Step title="प्रोवाइडर चुनें">
    OpenAI और ElevenLabs सबसे विश्वसनीय होस्टेड विकल्प हैं। Microsoft और
    Local CLI किसी API कुंजी के बिना काम करते हैं। पूरी सूची के लिए
    [प्रोवाइडर मैट्रिक्स](#supported-providers) देखें।
  </Step>
  <Step title="API कुंजी सेट करें">
    अपने प्रोवाइडर के लिए एनवायरनमेंट वेरिएबल एक्सपोर्ट करें (उदाहरण के लिए `OPENAI_API_KEY`,
    `ELEVENLABS_API_KEY`)। Microsoft और Local CLI को किसी कुंजी की आवश्यकता नहीं है।
  </Step>
  <Step title="कॉन्फ़िगरेशन में सक्षम करें">
    `tts.auto: "always"` और `tts.provider` सेट करें:

    ```json5
    {
      tts: {
        auto: "always",
        provider: "elevenlabs",
      },
    }
    ```

  </Step>
  <Step title="चैट में आज़माएँ">
    `/tts status` वर्तमान स्थिति दिखाता है। `/tts audio Hello from OpenClaw`
    एक बार का ऑडियो उत्तर भेजता है।
  </Step>
</Steps>

<Note>
Auto-TTS डिफ़ॉल्ट रूप से **बंद** है। जब `tts.provider` सेट नहीं होता,
OpenClaw रजिस्ट्री के ऑटो-सिलेक्ट क्रम में पहला कॉन्फ़िगर किया गया प्रोवाइडर चुनता है।
बिल्ट-इन `tts` एजेंट टूल केवल स्पष्ट अभिप्राय के लिए है: सामान्य चैट
टेक्स्ट ही रहती है, जब तक उपयोगकर्ता ऑडियो न माँगे, `/tts` का उपयोग न करे,
या Auto-TTS/डायरेक्टिव स्पीच सक्षम न करे।
</Note>

## समर्थित प्रोवाइडर

| प्रोवाइडर          | प्रमाणीकरण                                                                                                             | टिप्पणियाँ                                                                                       |
| ----------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Azure Speech**  | `AZURE_SPEECH_KEY` + `AZURE_SPEECH_REGION` (साथ ही `AZURE_SPEECH_API_KEY`, `SPEECH_KEY`, `SPEECH_REGION`)          | नेटिव Ogg/Opus वॉइस-नोट आउटपुट और टेलीफ़ोनी।                                            |
| **DeepInfra**     | `DEEPINFRA_API_KEY`                                                                                              | OpenAI-संगत TTS। डिफ़ॉल्ट `hexgrad/Kokoro-82M` है।                                    |
| **ElevenLabs**    | `ELEVENLABS_API_KEY` या `XI_API_KEY`                                                                             | वॉइस क्लोनिंग, बहुभाषी, `seed` के माध्यम से निर्धारक; Discord वॉइस प्लेबैक के लिए स्ट्रीम किया जाता है। |
| **Google Gemini** | `GEMINI_API_KEY` या `GOOGLE_API_KEY`                                                                             | Gemini API बैच TTS; `promptTemplate: "audio-profile-v1"` के माध्यम से पर्सोना-संवेदी।               |
| **Gradium**       | `GRADIUM_API_KEY`                                                                                                | वॉइस-नोट और टेलीफ़ोनी आउटपुट।                                                            |
| **Inworld**       | `INWORLD_API_KEY`                                                                                                | स्ट्रीमिंग TTS API। नेटिव Opus वॉइस-नोट और PCM टेलीफ़ोनी।                                |
| **Local CLI**     | कोई नहीं                                                                                                             | कॉन्फ़िगर की गई स्थानीय TTS कमांड चलाता है।                                                        |
| **Microsoft**     | कोई नहीं                                                                                                             | `node-edge-tts` के माध्यम से सार्वजनिक Edge न्यूरल TTS। सर्वोत्तम प्रयास, कोई SLA नहीं।                            |
| **MiniMax**       | `MINIMAX_API_KEY` (या Token Plan: `MINIMAX_OAUTH_TOKEN`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`)      | T2A v2 API। डिफ़ॉल्ट `speech-2.8-hd` है।                                                    |
| **OpenAI**        | `OPENAI_API_KEY`                                                                                                 | ऑटो-सारांश के लिए भी उपयोग किया जाता है; पर्सोना `instructions` का समर्थन करता है।                                |
| **OpenRouter**    | `OPENROUTER_API_KEY` (`models.providers.openrouter.apiKey` का पुनः उपयोग कर सकता है)                                            | डिफ़ॉल्ट मॉडल `hexgrad/kokoro-82m`।                                                         |
| **Volcengine**    | `VOLCENGINE_TTS_API_KEY` या `BYTEPLUS_SEED_SPEECH_API_KEY` (लेगेसी AppID/token: `VOLCENGINE_TTS_APPID`/`_TOKEN`) | BytePlus Seed Speech HTTP API।                                                              |
| **Vydra**         | `VYDRA_API_KEY`                                                                                                  | साझा इमेज, वीडियो और स्पीच प्रोवाइडर।                                                   |
| **xAI**           | `XAI_API_KEY`                                                                                                    | xAI बैच TTS। नेटिव Opus वॉइस-नोट **समर्थित नहीं** है।                                 |
| **Xiaomi MiMo**   | `XIAOMI_API_KEY`                                                                                                 | Xiaomi चैट कम्प्लीशन के माध्यम से MiMo TTS।                                                   |

यदि कई प्रोवाइडर कॉन्फ़िगर किए गए हैं, तो चयनित प्रोवाइडर का पहले उपयोग किया जाता है और
अन्य फ़ॉलबैक विकल्प होते हैं। ऑटो-सारांश `summaryModel` (या
`agents.defaults.model.primary`) का उपयोग करता है, इसलिए यदि आप सारांश सक्षम रखते हैं,
तो उस प्रोवाइडर का प्रमाणीकरण भी आवश्यक है।

<Warning>
बंडल किया गया **Microsoft** प्रोवाइडर `node-edge-tts` के माध्यम से Microsoft Edge की
ऑनलाइन न्यूरल TTS सेवा का उपयोग करता है। यह बिना प्रकाशित SLA या कोटा वाली एक सार्वजनिक
वेब सेवा है—इसे सर्वोत्तम-प्रयास सेवा मानें। लेगेसी प्रोवाइडर आईडी `edge` को
`microsoft` में सामान्यीकृत किया जाता है और `openclaw doctor --fix` स्थायी रूप से सहेजे गए
कॉन्फ़िगरेशन को पुनर्लिखता है; नए कॉन्फ़िगरेशन को हमेशा `microsoft` का उपयोग करना चाहिए।
</Warning>

## कॉन्फ़िगरेशन

TTS कॉन्फ़िगरेशन `~/.openclaw/openclaw.json` में `tts` के अंतर्गत होता है। कोई
प्रीसेट चुनें और प्रोवाइडर ब्लॉक को अनुकूलित करें। नीचे दिखाए गए `speakerVoice`/`speakerVoiceId`
फ़ील्ड कैनोनिकल हैं; प्रत्येक प्रोवाइडर के अपने `voice`/`voiceId`/
`voiceName` फ़ील्ड नाम अब भी लेगेसी उपनामों के रूप में काम करते हैं।

<Tabs>
  <Tab title="Azure Speech">
```json5
{
  tts: {
    auto: "always",
    provider: "azure-speech",
    providers: {
      "azure-speech": {
        apiKey: "${AZURE_SPEECH_KEY}",
        region: "eastus",
        speakerVoice: "en-US-JennyNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
        voiceNoteOutputFormat: "ogg-24khz-16bit-mono-opus",
      },
    },
  },
}
```
  </Tab>
  <Tab title="ElevenLabs">
```json5
{
  tts: {
    auto: "always",
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        model: "eleven_multilingual_v2",
        speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Google Gemini">
```json5
{
  tts: {
    auto: "always",
    provider: "google",
    providers: {
      google: {
        apiKey: "${GEMINI_API_KEY}",
        model: "gemini-3.1-flash-tts-preview",
        speakerVoice: "Kore",
        // वैकल्पिक प्राकृतिक-भाषा शैली प्रॉम्प्ट:
        // audioProfile: "शांत, पॉडकास्ट-होस्ट के लहज़े में बोलें।",
        // speakerName: "Alex",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Gradium">
```json5
{
  tts: {
    auto: "always",
    provider: "gradium",
    providers: {
      gradium: {
        apiKey: "${GRADIUM_API_KEY}",
        speakerVoiceId: "YTpq7expH9539ERJ",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Inworld">
```json5
{
  tts: {
    auto: "always",
    provider: "inworld",
    providers: {
      inworld: {
        apiKey: "${INWORLD_API_KEY}",
        modelId: "inworld-tts-1.5-max",
        speakerVoiceId: "Sarah",
        temperature: 0.7,
      },
    },
  },
}
```
  </Tab>
  <Tab title="Local CLI">
```json5
{
  tts: {
    auto: "always",
    provider: "tts-local-cli",
    providers: {
      "tts-local-cli": {
        command: "say",
        args: ["-o", "{{OutputPath}}", "{{Text}}"],
        outputFormat: "wav",
        timeoutMs: 120000,
      },
    },
  },
}
```
  </Tab>
  <Tab title="Microsoft (कुंजी आवश्यक नहीं)">
```json5
{
  tts: {
    auto: "always",
    provider: "microsoft",
    providers: {
      microsoft: {
        enabled: true,
        speakerVoice: "en-US-MichelleNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
        rate: "+0%",
        pitch: "+0%",
      },
    },
  },
}
```
  </Tab>
  <Tab title="MiniMax">
```json5
{
  tts: {
    auto: "always",
    provider: "minimax",
    providers: {
      minimax: {
        apiKey: "${MINIMAX_API_KEY}",
        model: "speech-2.8-hd",
        speakerVoiceId: "English_expressive_narrator",
        speed: 1.0,
        vol: 1.0,
        pitch: 0,
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenAI + ElevenLabs">
```json5
{
  tts: {
    auto: "always",
    provider: "openai",
    summaryModel: "openai/gpt-4.1-mini",
    modelOverrides: { enabled: true },
    providers: {
      openai: {
        apiKey: "${OPENAI_API_KEY}",
        model: "gpt-4o-mini-tts",
        speakerVoice: "alloy",
      },
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        model: "eleven_multilingual_v2",
        speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
        voiceSettings: { stability: 0.5, similarityBoost: 0.75, style: 0.0, useSpeakerBoost: true, speed: 1.0 },
        applyTextNormalization: "auto",
        languageCode: "en",
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenRouter">
```json5
{
  tts: {
    auto: "always",
    provider: "openrouter",
    providers: {
      openrouter: {
        apiKey: "${OPENROUTER_API_KEY}",
        model: "hexgrad/kokoro-82m",
        speakerVoice: "af_alloy",
        responseFormat: "mp3",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Volcengine">
```json5
{
  tts: {
    auto: "always",
    provider: "volcengine",
    providers: {
      volcengine: {
        apiKey: "${VOLCENGINE_TTS_API_KEY}",
        resourceId: "seed-tts-1.0",
        speakerVoice: "en_female_anna_mars_bigtts",
      },
    },
  },
}
```
  </Tab>
  <Tab title="xAI">
```json5
{
  tts: {
    auto: "always",
    provider: "xai",
    providers: {
      xai: {
        apiKey: "${XAI_API_KEY}",
        speakerVoiceId: "eve",
        language: "en",
        responseFormat: "mp3",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Xiaomi MiMo">
```json5
{
  tts: {
    auto: "always",
    provider: "xiaomi",
    providers: {
      xiaomi: {
        apiKey: "${XIAOMI_API_KEY}",
        model: "mimo-v2.5-tts",
        speakerVoice: "mimo_default",
        format: "mp3",
      },
    },
  },
}
```
  </Tab>
</Tabs>

Xiaomi `mimo-v2.5-tts-voicedesign` के लिए, `speakerVoice` को छोड़ दें और `style` को
वॉइस-डिज़ाइन प्रॉम्प्ट पर सेट करें। OpenClaw उस प्रॉम्प्ट को TTS `user` संदेश के रूप में
भेजता है और voicedesign मॉडल के लिए `audio.voice` नहीं भेजता।

### प्रति-एजेंट वॉइस ओवरराइड

जब किसी एक एजेंट को अलग प्रोवाइडर, वॉइस, मॉडल, पर्सोना या auto-TTS मोड के साथ बोलना हो,
तब `agents.entries.*.tts` का उपयोग करें। एजेंट ब्लॉक `tts` के ऊपर डीप-मर्ज होता है,
इसलिए प्रोवाइडर क्रेडेंशियल वैश्विक प्रोवाइडर कॉन्फ़िगरेशन में रह सकते हैं:

```json5
{
  tts: {
    auto: "always",
    provider: "elevenlabs",
    providers: {
      elevenlabs: { apiKey: "${ELEVENLABS_API_KEY}", model: "eleven_multilingual_v2" },
    },
  },
  agents: {
    list: [
      {
        id: "reader",
        tts: {
          providers: {
            elevenlabs: { speakerVoiceId: "EXAVITQu4vr4xnSDxMaL" },
          },
        },
      },
    ],
  },
}
```

प्रति-एजेंट पर्सोना को स्थिर करने के लिए, प्रदाता कॉन्फ़िगरेशन के साथ `agents.entries.*.tts.persona` सेट करें — यह केवल उस एजेंट के लिए वैश्विक `tts.persona` को ओवरराइड करता है।

स्वचालित उत्तरों, `/tts audio`, `/tts status`, और `tts` एजेंट टूल के लिए प्राथमिकता क्रम:

1. `tts`
2. सक्रिय `agents.entries.*.tts`
3. चैनल ओवरराइड, जब चैनल `channels.<channel>.tts` का समर्थन करता है
4. खाता ओवरराइड, जब चैनल `channels.<channel>.accounts.<id>.tts` पास करता है
5. इस होस्ट के लिए स्थानीय `/tts` प्राथमिकताएँ
6. जब [मॉडल-संचालित निर्देश](#model-driven-directives) सक्षम हों, तब इनलाइन `[[tts:...]]` निर्देश

चैनल और खाता ओवरराइड, `tts` जैसी ही संरचना का उपयोग करते हैं और पिछली परतों पर डीप-मर्ज होते हैं, इसलिए साझा प्रदाता क्रेडेंशियल `tts` में रह सकते हैं, जबकि कोई चैनल या बॉट खाता केवल वक्ता की आवाज़, मॉडल, पर्सोना या स्वचालित मोड बदलता है:

```json5
{
  tts: {
    provider: "openai",
    providers: {
      openai: { apiKey: "${OPENAI_API_KEY}", model: "gpt-4o-mini-tts" },
    },
  },
  channels: {
    feishu: {
      accounts: {
        english: {
          tts: {
            providers: {
              openai: { speakerVoice: "shimmer" },
            },
          },
        },
      },
    },
  },
}
```

## पर्सोना

**पर्सोना** एक स्थिर वाचिक पहचान है, जिसे सभी प्रदाताओं पर नियतात्मक रूप से लागू किया जा सकता है। यह किसी एक प्रदाता को प्राथमिकता दे सकता है, प्रदाता-निरपेक्ष प्रॉम्प्ट अभिप्राय परिभाषित कर सकता है, और आवाज़ों, मॉडलों, प्रॉम्प्ट टेम्पलेटों, सीड तथा आवाज़ सेटिंग के लिए प्रदाता-विशिष्ट बाइंडिंग रख सकता है।

### न्यूनतम पर्सोना

```json5
{
  tts: {
    auto: "always",
    persona: "narrator",
    personas: {
      narrator: {
        label: "Narrator",
        provider: "elevenlabs",
        providers: {
          elevenlabs: {
            speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
            modelId: "eleven_multilingual_v2",
          },
        },
      },
    },
  },
}
```

### पूर्ण पर्सोना (प्रदाता-विशिष्ट आकार निर्धारण)

```json5
{
  tts: {
    auto: "always",
    persona: "alfred",
    personas: {
      alfred: {
        label: "Alfred",
        description: "Dry, warm British butler narrator.",
        provider: "google",
        fallbackPolicy: "preserve-persona",
        providers: {
          google: {
            model: "gemini-3.1-flash-tts-preview",
            speakerVoice: "Algieba",
            promptTemplate: "audio-profile-v1",
          },
          openai: { model: "gpt-4o-mini-tts", speakerVoice: "cedar" },
          elevenlabs: {
            speakerVoiceId: "voice_id",
            modelId: "eleven_multilingual_v2",
            seed: 42,
            voiceSettings: {
              stability: 0.65,
              similarityBoost: 0.8,
              style: 0.25,
              useSpeakerBoost: true,
              speed: 0.95,
            },
          },
        },
      },
    },
  },
}
```

### पर्सोना निर्धारण

सक्रिय पर्सोना नियतात्मक रूप से चुना जाता है:

1. `/tts persona <id>` स्थानीय प्राथमिकता, यदि सेट हो।
2. `tts.persona`, यदि सेट हो।
3. कोई पर्सोना नहीं।

प्रदाता चयन में स्पष्ट विकल्प पहले लागू होता है:

1. प्रत्यक्ष ओवरराइड (CLI, gateway, Talk, अनुमत TTS निर्देश)।
2. `/tts provider <id>` स्थानीय प्राथमिकता।
3. सक्रिय पर्सोना का `provider`।
4. `tts.provider`।
5. रजिस्ट्री द्वारा स्वचालित चयन।

प्रत्येक प्रदाता प्रयास के लिए, OpenClaw कॉन्फ़िगरेशन को इस क्रम में मर्ज करता है:

1. `tts.providers.<id>`
2. `tts.personas.<persona>.providers.<id>`
3. विश्वसनीय अनुरोध ओवरराइड
4. अनुमत मॉडल-उत्सर्जित TTS निर्देश ओवरराइड

### कस्टम पर्सोना आकार निर्धारण

प्रदाता-निरपेक्ष `personas.<id>.prompt.*` कॉन्फ़िगरेशन हटा दिया गया है। Doctor उन फ़ील्ड को हटाता है और वाक्-प्रदाता सीम की ओर इंगित करता है। अंतर्निर्मित प्रदाता सेटिंग को `personas.<id>.providers.<provider>` के अंतर्गत रखें (उदाहरण के लिए Google `personaPrompt` या OpenAI `instructions`)। कस्टम आकार निर्धारण के लिए, `prepareSynthesis(ctx)` के साथ एक वाक् प्रदाता Plugin लागू करें और `synthesize()` चलने से पहले समायोजित टेक्स्ट, प्रदाता कॉन्फ़िगरेशन या ओवरराइड लौटाएँ। इससे अभिव्यंजक प्रॉम्प्ट निर्माण उस प्रदाता कोड में रहता है, जहाँ अनुरोध का अर्थ ज्ञात होता है।

### फ़ॉलबैक नीति

जब प्रयास किए जा रहे प्रदाता के लिए किसी पर्सोना की **कोई बाइंडिंग नहीं** होती, तब `fallbackPolicy` व्यवहार नियंत्रित करता है:

| नीति              | व्यवहार                                                                                                                                         |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `preserve-persona`  | **डिफ़ॉल्ट।** प्रदाता-निरपेक्ष प्रॉम्प्ट फ़ील्ड उपलब्ध रहते हैं; प्रदाता उनका उपयोग कर सकता है या उन्हें अनदेखा कर सकता है।                                            |
| `provider-defaults` | उस प्रयास के लिए प्रॉम्प्ट तैयार करते समय पर्सोना को छोड़ दिया जाता है; प्रदाता अपने निरपेक्ष डिफ़ॉल्ट का उपयोग करता है, जबकि अन्य प्रदाताओं पर फ़ॉलबैक जारी रहता है। |
| `fail`              | उस प्रदाता प्रयास को `reasonCode: "not_configured"` और `personaBinding: "missing"` के साथ छोड़ दें। फ़ॉलबैक प्रदाताओं को फिर भी आज़माया जाता है।              |

पूरा TTS अनुरोध केवल तभी विफल होता है, जब प्रयास किए गए **सभी** प्रदाता छोड़ दिए जाएँ या विफल हों।

Talk सत्र का प्रदाता चयन सत्र-सीमित होता है। Talk क्लाइंट को `talk.catalog` से प्रदाता आईडी, मॉडल आईडी, आवाज़ आईडी और लोकेल चुनकर उन्हें Talk सत्र या हैंडऑफ़ अनुरोध के माध्यम से पास करना चाहिए। आवाज़ सत्र खोलने से `tts` या वैश्विक Talk प्रदाता डिफ़ॉल्ट में बदलाव नहीं होना चाहिए।

## मॉडल-संचालित निर्देश

डिफ़ॉल्ट रूप से, सहायक एक उत्तर के लिए आवाज़, मॉडल या गति को ओवरराइड करने हेतु `[[tts:...]]` निर्देश उत्सर्जित **कर सकता है**, साथ ही उन अभिव्यंजक संकेतों के लिए वैकल्पिक `[[tts:text]]...[[/tts:text]]` ब्लॉक भी दे सकता है, जिन्हें केवल ऑडियो में दिखाई देना चाहिए:

```text
यह लीजिए।

[[tts:speakerVoiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]](हँसते हुए) गीत को एक बार फिर पढ़ें।[[/tts:text]]
```

जब `tts.auto`, `"tagged"` होता है, तो ऑडियो ट्रिगर करने के लिए **निर्देश आवश्यक होते हैं**। स्ट्रीमिंग ब्लॉक डिलीवरी, चैनल तक पहुँचने से पहले दृश्य टेक्स्ट से निर्देशों को हटा देती है, भले ही वे निकटवर्ती ब्लॉक में विभाजित हों।

`provider=...` को तब तक अनदेखा किया जाता है, जब तक `modelOverrides.allowProvider: true` न हो। जब कोई उत्तर `provider=...` घोषित करता है, तो उस निर्देश की अन्य कुंजियों को केवल वही प्रदाता पार्स करता है; असमर्थित कुंजियाँ हटा दी जाती हैं और TTS निर्देश चेतावनियों के रूप में रिपोर्ट की जाती हैं।

**उपलब्ध निर्देश कुंजियाँ:**

- `provider` (पंजीकृत प्रदाता आईडी; `allowProvider: true` आवश्यक)
- `speakerVoice` / `speakerVoiceId` (पुराने उपनाम: `voice`, `voiceName`, `voice_name`, `google_voice`, `voiceId`)
- `model` / `google_model`
- `stability`, `similarityBoost`, `style`, `speed`, `useSpeakerBoost`
- `vol` / `volume` (MiniMax वॉल्यूम, `(0, 10]`)
- `pitch` (MiniMax पूर्णांक पिच, −12 से 12; भिन्नात्मक मान काट दिए जाते हैं)
- `emotion` (Volcengine भावना टैग)
- `applyTextNormalization` (`auto|on|off`)
- `languageCode` (ISO 639-1)
- `seed`

**मॉडल ओवरराइड को पूरी तरह अक्षम करें:**

```json5
{ messages: { tts: { modelOverrides: { enabled: false } } } }
```

**अन्य नियंत्रणों को कॉन्फ़िगर करने योग्य रखते हुए प्रदाता बदलने की अनुमति दें:**

```json5
{ messages: { tts: { modelOverrides: { enabled: true, allowProvider: true, allowSeed: false } } } }
```

## स्लैश कमांड

एकल कमांड `/tts`। Discord पर, OpenClaw `/voice` भी पंजीकृत करता है, क्योंकि `/tts` एक अंतर्निर्मित Discord कमांड है — टेक्स्ट `/tts ...` फिर भी काम करता है।

```text
/tts off | on | status
/tts chat on | off | default
/tts latest
/tts provider <id>
/tts persona <id> | off
/tts limit <chars>
/tts summary off
/tts audio <text>
```

<Note>
कमांड के लिए अधिकृत प्रेषक आवश्यक है (अनुमति-सूची/स्वामी नियम लागू होते हैं) और या तो `commands.text` अथवा नेटिव कमांड पंजीकरण सक्षम होना चाहिए।
</Note>

व्यवहार संबंधी टिप्पणियाँ:

- `/tts on` स्थानीय TTS प्राथमिकता को `always` में लिखता है; `/tts off` इसे `off` में लिखता है।
- `/tts chat on|off|default` वर्तमान चैट के लिए सत्र-सीमित स्वचालित-TTS ओवरराइड लिखता है।
- `/tts persona <id>` स्थानीय पर्सोना प्राथमिकता लिखता है; `/tts persona off` इसे साफ़ करता है।
- `/tts latest` वर्तमान सत्र ट्रांस्क्रिप्ट से सहायक का नवीनतम उत्तर पढ़ता है और उसे एक बार ऑडियो के रूप में भेजता है। डुप्लिकेट आवाज़ प्रेषण रोकने के लिए यह सत्र प्रविष्टि में उस उत्तर का केवल एक हैश संग्रहीत करता है।
- `/tts audio` एक बार उपयोग होने वाला ऑडियो उत्तर बनाता है (TTS को चालू **नहीं** करता)।
- `/tts limit <chars>` **100–4096** स्वीकार करता है (4096 Telegram कैप्शन/संदेश की अधिकतम सीमा है); इस सीमा के बाहर के मान अस्वीकार कर दिए जाते हैं।
- `limit` और `summary` मुख्य कॉन्फ़िगरेशन में नहीं, बल्कि **स्थानीय प्राथमिकताओं** में संग्रहीत होते हैं।
- `/tts status` में नवीनतम प्रयास के लिए फ़ॉलबैक निदान शामिल होता है — `Fallback: <primary> -> <used>`, `Attempts: ...`, और प्रत्येक प्रयास का विवरण (`provider:outcome(reasonCode) latency`)।
- `/status` TTS सक्षम होने पर सक्रिय TTS मोड के साथ कॉन्फ़िगर किया गया प्रदाता, मॉडल, आवाज़ और सैनिटाइज़ किया गया कस्टम एंडपॉइंट मेटाडेटा दिखाता है।

## प्रति-उपयोगकर्ता प्राथमिकताएँ

स्लैश कमांड, TTS प्राथमिकता पथ में स्थानीय ओवरराइड लिखते हैं। डिफ़ॉल्ट `~/.openclaw/settings/tts.json` है; इसे `OPENCLAW_TTS_PREFS` से ओवरराइड करें। Doctor हटाए गए वैश्विक `tts.prefsPath` मान को साझा मशीन स्थिति में ले जाता है। उन्नत बहु-एजेंट सेटअप अब भी `agents.entries.<id>.tts.prefsPath` सेट कर सकते हैं, जब एजेंट जानबूझकर अलग-अलग प्राथमिकता स्टोर का उपयोग करते हैं।

| संग्रहीत फ़ील्ड | प्रभाव                                                                           |
| ------------ | -------------------------------------------------------------------------------- |
| `auto`       | स्थानीय स्वचालित-TTS ओवरराइड (`always`, `off`, …)                                     |
| `provider`   | स्थानीय प्राथमिक प्रदाता ओवरराइड                                                  |
| `persona`    | स्थानीय पर्सोना ओवरराइड                                                           |
| `maxLength`  | सारांश/काट-छाँट सीमा (डिफ़ॉल्ट `1500` वर्ण, `/tts limit` सीमा 100–4096) |
| `summarize`  | सारांश टॉगल (डिफ़ॉल्ट `true`)                                                  |

ये उस होस्ट के लिए `tts` और सक्रिय `agents.entries.*.tts` ब्लॉक से प्राप्त प्रभावी कॉन्फ़िगरेशन को ओवरराइड करते हैं।

## आउटपुट प्रारूप

TTS आवाज़ डिलीवरी चैनल की क्षमताओं से संचालित होती है। चैनल Plugin बताते हैं कि आवाज़-शैली TTS को प्रदाताओं से नेटिव `voice-note` लक्ष्य माँगना चाहिए या सामान्य `audio-file` संश्लेषण बनाए रखना चाहिए, और यह भी कि चैनल भेजने से पहले गैर-नेटिव आउटपुट को ट्रांसकोड करता है या नहीं।

| लक्ष्य                                | प्रारूप                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Feishu / Matrix / Telegram / WhatsApp | वॉइस-नोट उत्तरों के लिए **Opus** को प्राथमिकता दी जाती है (ElevenLabs से `opus_48000_64`, OpenAI से `opus`)। 48 kHz / 64 kbps स्पष्टता और आकार को संतुलित करता है। |
| अन्य चैनल                        | **MP3** (ElevenLabs से `mp3_44100_128`, OpenAI से `mp3`)। वाक् के लिए 44.1 kHz / 128 kbps डिफ़ॉल्ट संतुलन है।                  |
| बातचीत / टेलीफ़ोनी                      | प्रदाता-मूल **PCM** (Inworld 22050 Hz, Google 24 kHz), या टेलीफ़ोनी के लिए Gradium से `ulaw_8000`।                                 |

प्रति-प्रदाता टिप्पणियाँ:

- **Feishu / WhatsApp ट्रांसकोडिंग:** जब कोई वॉइस-नोट उत्तर MP3/WebM/WAV/M4A या किसी अन्य संभावित ऑडियो फ़ाइल के रूप में आता है, तो चैनल Plugin मूल वॉइस संदेश भेजने से पहले उसे `ffmpeg` (`libopus`, 64 kbps) के साथ 48 kHz Ogg/Opus में ट्रांसकोड करता है। WhatsApp परिणाम को Baileys `audio` पेलोड के माध्यम से `ptt: true` और `audio/ogg; codecs=opus` के साथ भेजता है। ट्रांसकोड विफल होने पर: Feishu त्रुटि को संभालता है और मूल फ़ाइल को सामान्य अटैचमेंट के रूप में भेजता है; WhatsApp में कोई फ़ॉलबैक नहीं है, इसलिए असंगत PTT पेलोड पोस्ट होने के बजाय भेजने की प्रक्रिया ही विफल हो जाती है।
- **MiniMax:** सामान्य ऑडियो अटैचमेंट के लिए MP3 (`speech-2.8-hd` मॉडल, 32 kHz सैंपल दर); चैनल द्वारा घोषित वॉइस-नोट लक्ष्यों के लिए `ffmpeg` के साथ 48 kHz Opus में ट्रांसकोड किया जाता है।
- **Xiaomi MiMo:** डिफ़ॉल्ट रूप से MP3, या कॉन्फ़िगर किए जाने पर WAV; चैनल द्वारा घोषित वॉइस-नोट लक्ष्यों के लिए `ffmpeg` के साथ 48 kHz Opus में ट्रांसकोड किया जाता है।
- **स्थानीय CLI:** कॉन्फ़िगर किए गए `outputFormat` का उपयोग करता है। वॉइस-नोट लक्ष्यों को Ogg/Opus में और टेलीफ़ोनी आउटपुट को `ffmpeg` के साथ रॉ 16 kHz मोनो PCM में बदला जाता है।
- **Google Gemini:** रॉ 24 kHz PCM लौटाता है। OpenClaw इसे ऑडियो अटैचमेंट के लिए WAV के रूप में रैप करता है, वॉइस-नोट लक्ष्यों के लिए 48 kHz Opus में ट्रांसकोड करता है और बातचीत/टेलीफ़ोनी के लिए सीधे PCM लौटाता है।
- **Gradium:** ऑडियो अटैचमेंट के लिए WAV, वॉइस-नोट लक्ष्यों के लिए Opus और टेलीफ़ोनी के लिए 8 kHz पर `ulaw_8000`।
- **Inworld:** सामान्य ऑडियो अटैचमेंट के लिए MP3, वॉइस-नोट लक्ष्यों के लिए मूल `OGG_OPUS` और बातचीत/टेलीफ़ोनी के लिए 22050 Hz पर रॉ `PCM`।
- **xAI:** डिफ़ॉल्ट रूप से MP3; ऑडियो-फ़ाइल संश्लेषण बफ़र किए गए और स्ट्रीमिंग आउटपुट, दोनों के लिए `mp3`, `wav`, `pcm`, `mulaw` या `alaw` का उपयोग कर सकता है। वॉइस-नोट लक्ष्य स्ट्रीमिंग और बफ़र किए गए फ़ॉलबैक के लिए MP3 का उपयोग करते हैं, क्योंकि xAI के `pcm`, `mulaw` और `alaw` आउटपुट हेडर-रहित रॉ ऑडियो होते हैं। बफ़र किया गया संश्लेषण xAI के बैच REST `/v1/tts` एंडपॉइंट का उपयोग करता है; `textToSpeechStream` मूल `wss://api.x.ai/v1/tts` का उपयोग करता है। यह रीयलटाइम वॉइस अनुबंध नहीं है। मूल Opus वॉइस-नोट प्रारूप समर्थित नहीं है।
- **Microsoft:** `microsoft.outputFormat` (डिफ़ॉल्ट `audio-24khz-48kbitrate-mono-mp3`) का उपयोग करता है।
  - बंडल किया गया ट्रांसपोर्ट एक `outputFormat` स्वीकार करता है, लेकिन सेवा से सभी प्रारूप उपलब्ध नहीं हैं।
  - आउटपुट प्रारूप मान Microsoft Speech आउटपुट प्रारूपों (Ogg/WebM Opus सहित) का अनुसरण करते हैं।
  - Telegram `sendVoice` OGG/MP3/M4A स्वीकार करता है; यदि सुनिश्चित Opus वॉइस संदेशों की आवश्यकता है, तो OpenAI/ElevenLabs का उपयोग करें।
  - यदि कॉन्फ़िगर किया गया Microsoft आउटपुट प्रारूप विफल हो जाता है, तो OpenClaw MP3 के साथ पुनः प्रयास करता है।
  - जब कोई स्पष्ट वॉइस ओवरराइड सेट नहीं होता और डिफ़ॉल्ट अंग्रेज़ी वॉइस का उपयोग किया जाता है, तो उत्तर का टेक्स्ट CJK-प्रधान होने पर OpenClaw स्वतः चीनी न्यूरल वॉइस (`zh-CN-XiaoxiaoNeural`, `zh-CN` लोकेल) पर स्विच हो जाता है।

OpenAI और ElevenLabs के आउटपुट प्रारूप ऊपर सूचीबद्ध चैनलों के अनुसार निश्चित हैं।

## स्वचालित TTS व्यवहार

जब `tts.auto` सक्षम होता है, तो OpenClaw:

- यदि उत्तर में पहले से संरचित मीडिया है, तो TTS छोड़ देता है।
- बहुत छोटे उत्तर (10 वर्णों से कम) छोड़ देता है।
- सारांश सक्षम होने पर लंबे उत्तरों का सारांश
  `summaryModel` (या `agents.defaults.model.primary`) का उपयोग करके तैयार करता है।
- जनरेट किए गए ऑडियो को उत्तर से जोड़ता है।
- `mode: "final"` में, टेक्स्ट स्ट्रीम पूर्ण होने के बाद भी स्ट्रीम किए गए अंतिम उत्तरों के लिए केवल-ऑडियो TTS भेजता है;
  जनरेट किया गया मीडिया सामान्य उत्तर अटैचमेंट की तरह उसी
  चैनल मीडिया सामान्यीकरण से गुजरता है।

यदि उत्तर `maxLength` से अधिक है, तो OpenClaw कभी भी ऑडियो को पूरी तरह नहीं छोड़ता:

- **सारांश चालू** (डिफ़ॉल्ट) और सारांश मॉडल उपलब्ध है: टेक्स्ट को लगभग
  `maxLength` वर्णों में सारांशित करता है, फिर सारांश को संश्लेषित करता है।
- **सारांश बंद**, सारांशीकरण विफल होता है, या सारांश मॉडल के लिए कोई API कुंजी उपलब्ध नहीं है:
  टेक्स्ट को `maxLength` वर्णों तक काटता है और काटे गए
  टेक्स्ट को संश्लेषित करता है।

```text
उत्तर -> TTS सक्षम है?
  नहीं  -> टेक्स्ट भेजें
  हाँ -> मीडिया है / छोटा है?
          हाँ -> टेक्स्ट भेजें
          नहीं  -> लंबाई > सीमा?
                   नहीं  -> TTS -> ऑडियो जोड़ें
                   हाँ -> सारांश सक्षम और उपलब्ध है?
                            नहीं  -> काटें -> TTS -> ऑडियो जोड़ें
                            हाँ -> सारांशित करें -> TTS -> ऑडियो जोड़ें
```

## फ़ील्ड संदर्भ

<AccordionGroup>
  <Accordion title="शीर्ष-स्तरीय tts.*">
    <ParamField path="auto" type='"off" | "always" | "inbound" | "tagged"'>
      स्वचालित TTS मोड। `inbound` केवल आने वाले वॉइस संदेश के बाद ऑडियो भेजता है; `tagged` केवल तभी ऑडियो भेजता है जब उत्तर में `[[tts:...]]` निर्देश या `[[tts:text]]` ब्लॉक शामिल हो।
    </ParamField>
    <ParamField path="enabled" type="boolean" deprecated>
      पुराना टॉगल। `openclaw doctor --fix` इसे `auto` में माइग्रेट करता है।
    </ParamField>
    <ParamField path="mode" type='"final" | "all"' default="final">
      `"all"` में अंतिम उत्तरों के अतिरिक्त टूल/ब्लॉक उत्तर भी शामिल होते हैं।
    </ParamField>
    <ParamField path="provider" type="string">
      वाक् प्रदाता आईडी। सेट न होने पर OpenClaw रजिस्ट्री के स्वचालित-चयन क्रम में पहले कॉन्फ़िगर किए गए प्रदाता का उपयोग करता है। पुराने `provider: "edge"` को `openclaw doctor --fix` द्वारा `"microsoft"` में पुनर्लिखा जाता है।
    </ParamField>
    <ParamField path="persona" type="string">
      `personas` से सक्रिय पर्सोना आईडी। लोअरकेस में सामान्यीकृत किया जाता है।
    </ParamField>
    <ParamField path="personas.<id>" type="object">
      स्थिर मौखिक पहचान। फ़ील्ड: `label`, `description`, `provider`, `fallbackPolicy`, `prompt`, `providers.<provider>`। [पर्सोना](#personas) देखें।
    </ParamField>
    <ParamField path="summaryModel" type="string">
      स्वचालित सारांश के लिए कम लागत वाला मॉडल; डिफ़ॉल्ट `agents.defaults.model.primary`। `provider/model` या कॉन्फ़िगर किए गए मॉडल उपनाम को स्वीकार करता है।
    </ParamField>
    <ParamField path="modelOverrides" type="object">
      मॉडल को TTS निर्देश उत्सर्जित करने की अनुमति दें। `enabled` का डिफ़ॉल्ट `true`; `allowProvider` का डिफ़ॉल्ट `false` है।
    </ParamField>
    <ParamField path="providers.<id>" type="object">
      वाक् प्रदाता आईडी द्वारा कुंजीबद्ध प्रदाता-स्वामित्व वाली सेटिंग। पुराने प्रत्यक्ष ब्लॉक (`tts.openai`, `.elevenlabs`, `.microsoft`, `.edge`) `openclaw doctor --fix` द्वारा पुनर्लिखे जाते हैं; केवल `tts.providers.<id>` कमिट करें।
    </ParamField>
    <ParamField path="maxTextLength" type="number" default="4096">
      TTS इनपुट वर्णों की कठोर सीमा। सीमा से अधिक होने पर `/tts audio`, `tts.convert` और `tts.speak` विफल हो जाते हैं।
    </ParamField>
    <ParamField path="timeoutMs" type="number" default="30000">
      मिलीसेकंड में अनुरोध टाइमआउट। सेट होने पर प्रति-कॉल `timeoutMs` (एजेंट टूल, gateway) प्रभावी होता है; अन्यथा स्पष्ट रूप से कॉन्फ़िगर किया गया `tts.timeoutMs` किसी भी Plugin-निर्मित प्रदाता डिफ़ॉल्ट पर प्रभावी होता है।
    </ParamField>
  </Accordion>

प्रदाता `apiKey` फ़ील्ड रॉ स्ट्रिंग या SecretRefs हो सकते हैं। ठंडे Gateway
स्टार्टअप के दौरान, अनुपलब्ध TTS SecretRef, Gateway को रोकने के बजाय अंतर्निहित TTS क्षमता को
कॉन्फ़िगर-अनुपलब्ध चिह्नित करता है। तब `tts.speak`, कारण `SECRET_SURFACE_UNAVAILABLE` के साथ
`UNAVAILABLE` लौटाता है और कोई प्रदाता अनुरोध
नहीं भेजा जाता। स्थिति और डॉक्टर निम्नीकृत TTS स्वामी तथा उसके कॉन्फ़िगरेशन पथ सूचीबद्ध करते हैं। स्पष्ट
रेफ़रेंस रनटाइम स्नैपशॉट में बने रहते हैं, ताकि परिवेश या प्रोफ़ाइल
क्रेडेंशियल चुपचाप किसी अन्य अकाउंट का चयन न कर सकें। रीलोड और कॉन्फ़िगरेशन-लेखन
प्रीफ़्लाइट स्वामी-सजग निम्नीकरण नीति लागू करते हैं: एक अपरिवर्तित पात्र TTS
स्वामी अपने अंतिम-ज्ञात-सही क्रेडेंशियल को पुराने रूप में रख सकता है, जबकि नई या बदली हुई
विफलता स्वस्थ स्वामियों को अवरुद्ध किए बिना ठंडी हो जाती है। संरचनात्मक रूप से अमान्य रेफ़रेंस
और हल किए गए मान अब भी स्टार्टअप को विफल करते हैं या अपडेट अस्वीकार करते हैं।

  <Accordion title="Azure Speech">
    <ParamField path="apiKey" type="string">परिवेश: `AZURE_SPEECH_KEY`, `AZURE_SPEECH_API_KEY`, या `SPEECH_KEY`।</ParamField>
    <ParamField path="region" type="string">Azure Speech क्षेत्र (उदा. `eastus`)। परिवेश: `AZURE_SPEECH_REGION` या `SPEECH_REGION`।</ParamField>
    <ParamField path="endpoint" type="string">वैकल्पिक Azure Speech एंडपॉइंट ओवरराइड (उपनाम `baseUrl`)।</ParamField>
    <ParamField path="speakerVoice" type="string">Azure वॉइस ShortName। डिफ़ॉल्ट `en-US-JennyNeural`। पुराना उपनाम: `voice`।</ParamField>
    <ParamField path="lang" type="string">SSML भाषा कोड। डिफ़ॉल्ट `en-US`।</ParamField>
    <ParamField path="outputFormat" type="string">मानक ऑडियो के लिए Azure `X-Microsoft-OutputFormat`। डिफ़ॉल्ट `audio-24khz-48kbitrate-mono-mp3`।</ParamField>
    <ParamField path="voiceNoteOutputFormat" type="string">वॉइस-नोट आउटपुट के लिए Azure `X-Microsoft-OutputFormat`। डिफ़ॉल्ट `ogg-24khz-16bit-mono-opus`।</ParamField>
  </Accordion>

  <Accordion title="ElevenLabs">
    <ParamField path="apiKey" type="string">`ELEVENLABS_API_KEY` या `XI_API_KEY` पर फ़ॉलबैक करता है।</ParamField>
    <ParamField path="model" type="string">मॉडल आईडी। डिफ़ॉल्ट `eleven_multilingual_v2`। पुराने आईडी `eleven_turbo_v2_5`/`eleven_turbo_v2` मिलान करने वाले `flash` मॉडल में सामान्यीकृत किए जाते हैं।</ParamField>
    <ParamField path="speakerVoiceId" type="string">ElevenLabs वॉइस आईडी। डिफ़ॉल्ट `pMsXgVXv3BLzUgSXRplE`। पुराना उपनाम: `voiceId`।</ParamField>
    <ParamField path="voiceSettings" type="object">
      `stability`, `similarityBoost`, `style` (प्रत्येक `0..1`, डिफ़ॉल्ट `0.5`/`0.75`/`0`), `useSpeakerBoost` (`true|false`, डिफ़ॉल्ट `true`), `speed` (`0.5..2.0`, डिफ़ॉल्ट `1.0`)।
    </ParamField>
    <ParamField path="applyTextNormalization" type='"auto" | "on" | "off"'>टेक्स्ट सामान्यीकरण मोड।</ParamField>
    <ParamField path="languageCode" type="string">2-अक्षरीय ISO 639-1 (उदा. `en`, `de`)।</ParamField>
    <ParamField path="seed" type="number">सर्वोत्तम-प्रयास नियतात्मकता के लिए पूर्णांक `0..4294967295`।</ParamField>
    <ParamField path="baseUrl" type="string">ElevenLabs API आधार URL को ओवरराइड करें।</ParamField>
  </Accordion>

  <Accordion title="Google Gemini">
    <ParamField path="apiKey" type="string">`GEMINI_API_KEY` / `GOOGLE_API_KEY` पर फ़ॉलबैक करता है। छोड़े जाने पर, env फ़ॉलबैक से पहले TTS `models.providers.google.apiKey` का पुनः उपयोग कर सकता है।</ParamField>
    <ParamField path="model" type="string">Gemini TTS मॉडल। डिफ़ॉल्ट `gemini-3.1-flash-tts-preview`।</ParamField>
    <ParamField path="speakerVoice" type="string">Gemini की पहले से निर्मित वॉइस का नाम। डिफ़ॉल्ट `Kore`। लेगेसी उपनाम: `voiceName`, `voice`।</ParamField>
    <ParamField path="audioProfile" type="string">बोले जाने वाले टेक्स्ट से पहले जोड़ा गया प्राकृतिक-भाषा शैली प्रॉम्प्ट।</ParamField>
    <ParamField path="speakerName" type="string">जब आपका प्रॉम्प्ट किसी नामित वक्ता का उपयोग करता है, तो बोले जाने वाले टेक्स्ट से पहले जोड़ा गया वैकल्पिक वक्ता लेबल।</ParamField>
    <ParamField path="promptTemplate" type='"audio-profile-v1"'>सक्रिय पर्सोना प्रॉम्प्ट फ़ील्ड को नियतात्मक Gemini TTS प्रॉम्प्ट संरचना में रैप करने के लिए इसे `audio-profile-v1` पर सेट करें।</ParamField>
    <ParamField path="personaPrompt" type="string">टेम्पलेट के निर्देशक के नोट्स में जोड़ा गया Google-विशिष्ट अतिरिक्त पर्सोना प्रॉम्प्ट टेक्स्ट।</ParamField>
    <ParamField path="baseUrl" type="string">केवल `https://generativelanguage.googleapis.com` स्वीकार किया जाता है।</ParamField>
  </Accordion>

  <Accordion title="Gradium">
    <ParamField path="apiKey" type="string">Env: `GRADIUM_API_KEY`।</ParamField>
    <ParamField path="baseUrl" type="string">`api.gradium.ai` पर HTTPS Gradium API URL। डिफ़ॉल्ट `https://api.gradium.ai`।</ParamField>
    <ParamField path="speakerVoiceId" type="string">डिफ़ॉल्ट Emma (`YTpq7expH9539ERJ`)। लेगेसी उपनाम: `voiceId`।</ParamField>
  </Accordion>

  <Accordion title="Inworld">
    ### Inworld प्राथमिक

    <ParamField path="apiKey" type="string">Env: `INWORLD_API_KEY`।</ParamField>
    <ParamField path="baseUrl" type="string">डिफ़ॉल्ट `https://api.inworld.ai`।</ParamField>
    <ParamField path="modelId" type="string">डिफ़ॉल्ट `inworld-tts-1.5-max`। ये भी: `inworld-tts-1.5-mini`, `inworld-tts-1-max`, `inworld-tts-1`।</ParamField>
    <ParamField path="speakerVoiceId" type="string">डिफ़ॉल्ट `Sarah`। लेगेसी उपनाम: `voiceId`।</ParamField>
    <ParamField path="temperature" type="number">सैंपलिंग तापमान `0..2` (0 को छोड़कर)।</ParamField>

  </Accordion>

  <Accordion title="स्थानीय CLI (tts-local-cli)">
    <ParamField path="command" type="string">CLI TTS के लिए स्थानीय एक्ज़िक्यूटेबल या कमांड स्ट्रिंग।</ParamField>
    <ParamField path="args" type="string[]">कमांड आर्ग्युमेंट। `{{Text}}`, `{{OutputPath}}`, `{{OutputDir}}`, `{{OutputBase}}` प्लेसहोल्डर समर्थित हैं।</ParamField>
    <ParamField path="outputFormat" type='"mp3" | "opus" | "wav"'>अपेक्षित CLI आउटपुट फ़ॉर्मैट। ऑडियो अटैचमेंट के लिए डिफ़ॉल्ट `mp3`।</ParamField>
    <ParamField path="timeoutMs" type="number">कमांड टाइमआउट मिलीसेकंड में। डिफ़ॉल्ट `120000`।</ParamField>
    <ParamField path="cwd" type="string">वैकल्पिक कमांड वर्किंग डायरेक्टरी।</ParamField>
    <ParamField path="env" type="Record<string, string>">कमांड के लिए वैकल्पिक एनवायरनमेंट ओवरराइड।</ParamField>

    कमांड stdout और जनरेट या कन्वर्ट किए गए ऑडियो की सीमा 50 MiB है। डायग्नोस्टिक stderr की सीमा 1 MiB है। किसी भी सीमा के पार होने पर OpenClaw कमांड को समाप्त कर देता है और संश्लेषण विफल कर देता है।

  </Accordion>

  <Accordion title="Microsoft (API कुंजी के बिना)">
    <ParamField path="enabled" type="boolean" default="true">Microsoft स्पीच के उपयोग की अनुमति दें।</ParamField>
    <ParamField path="speakerVoice" type="string">Microsoft न्यूरल वॉइस का नाम (उदा. `en-US-MichelleNeural`)। लेगेसी उपनाम: `voice`। यदि डिफ़ॉल्ट अंग्रेज़ी वॉइस प्रभावी है और उत्तर का टेक्स्ट मुख्यतः CJK है, तो OpenClaw स्वतः `zh-CN-XiaoxiaoNeural` पर स्विच करता है।</ParamField>
    <ParamField path="lang" type="string">भाषा कोड (उदा. `en-US`)।</ParamField>
    <ParamField path="outputFormat" type="string">Microsoft आउटपुट फ़ॉर्मैट। डिफ़ॉल्ट `audio-24khz-48kbitrate-mono-mp3`। बंडल किए गए Edge-समर्थित ट्रांसपोर्ट में सभी फ़ॉर्मैट समर्थित नहीं हैं।</ParamField>
    <ParamField path="rate / pitch / volume" type="string">प्रतिशत स्ट्रिंग (उदा. `+10%`, `-5%`)।</ParamField>
    <ParamField path="saveSubtitles" type="boolean">ऑडियो फ़ाइल के साथ JSON उपशीर्षक लिखें।</ParamField>
    <ParamField path="proxy" type="string">Microsoft स्पीच अनुरोधों के लिए प्रॉक्सी URL।</ParamField>
    <ParamField path="timeoutMs" type="number">अनुरोध टाइमआउट ओवरराइड (ms)।</ParamField>
    <ParamField path="edge.*" type="object" deprecated>लेगेसी उपनाम। स्थायी कॉन्फ़िग को `providers.microsoft` में पुनर्लिखने के लिए `openclaw doctor --fix` चलाएँ।</ParamField>
  </Accordion>

  <Accordion title="MiniMax">
    <ParamField path="apiKey" type="string">`MINIMAX_API_KEY` पर फ़ॉलबैक करता है। `MINIMAX_OAUTH_TOKEN`, `MINIMAX_CODE_PLAN_KEY`, या `MINIMAX_CODING_API_KEY` के माध्यम से Token Plan प्रमाणीकरण।</ParamField>
    <ParamField path="baseUrl" type="string">डिफ़ॉल्ट `https://api.minimax.io`। Env: `MINIMAX_API_HOST`।</ParamField>
    <ParamField path="model" type="string">डिफ़ॉल्ट `speech-2.8-hd`। Env: `MINIMAX_TTS_MODEL`।</ParamField>
    <ParamField path="speakerVoiceId" type="string">डिफ़ॉल्ट `English_expressive_narrator`। Env: `MINIMAX_TTS_VOICE_ID`। लेगेसी उपनाम: `voiceId`।</ParamField>
    <ParamField path="speed" type="number">`0.5..2.0`। डिफ़ॉल्ट `1.0`।</ParamField>
    <ParamField path="vol" type="number">`(0, 10]`। डिफ़ॉल्ट `1.0`।</ParamField>
    <ParamField path="pitch" type="number">पूर्णांक `-12..12`। डिफ़ॉल्ट `0`। अनुरोध से पहले भिन्नात्मक मान काट दिए जाते हैं।</ParamField>
  </Accordion>

  <Accordion title="OpenAI">
    <ParamField path="apiKey" type="string">`OPENAI_API_KEY` पर फ़ॉलबैक करता है।</ParamField>
    <ParamField path="model" type="string">OpenAI TTS मॉडल id। डिफ़ॉल्ट `gpt-4o-mini-tts`।</ParamField>
    <ParamField path="speakerVoice" type="string">वॉइस का नाम (उदा. `alloy`, `cedar`)। डिफ़ॉल्ट `coral`। लेगेसी उपनाम: `voice`।</ParamField>
    <ParamField path="instructions" type="string">स्पष्ट OpenAI `instructions` फ़ील्ड। इसे सेट करने पर पर्सोना प्रॉम्प्ट फ़ील्ड **स्वतः** मैप नहीं किए जाते।</ParamField>
    <ParamField path="extraBody / extra_body" type="Record<string, unknown>">जनरेट किए गए OpenAI TTS फ़ील्ड के बाद `/audio/speech` अनुरोध बॉडी में मर्ज किए जाने वाले अतिरिक्त JSON फ़ील्ड। इसका उपयोग Kokoro जैसे OpenAI-संगत एंडपॉइंट के लिए करें, जिन्हें `lang` जैसी प्रदाता-विशिष्ट कुंजियों की आवश्यकता होती है; असुरक्षित प्रोटोटाइप कुंजियों को अनदेखा किया जाता है।</ParamField>
    <ParamField path="baseUrl" type="string">
      OpenAI TTS एंडपॉइंट को ओवरराइड करें। समाधान क्रम: कॉन्फ़िग → `OPENAI_TTS_BASE_URL` → `https://api.openai.com/v1`। गैर-डिफ़ॉल्ट मानों को OpenAI-संगत TTS एंडपॉइंट माना जाता है, इसलिए कस्टम मॉडल और वॉइस नाम स्वीकार किए जाते हैं, और `speed` की `0.25..4.0` रेंज जाँच हट जाती है।
    </ParamField>
  </Accordion>

  <Accordion title="OpenRouter">
    <ParamField path="apiKey" type="string">Env: `OPENROUTER_API_KEY`। `models.providers.openrouter.apiKey` का पुनः उपयोग कर सकता है।</ParamField>
    <ParamField path="baseUrl" type="string">डिफ़ॉल्ट `https://openrouter.ai/api/v1`। लेगेसी `https://openrouter.ai/v1` को सामान्यीकृत किया जाता है।</ParamField>
    <ParamField path="model" type="string">डिफ़ॉल्ट `hexgrad/kokoro-82m`। उपनाम: `modelId`।</ParamField>
    <ParamField path="speakerVoice" type="string">डिफ़ॉल्ट `af_alloy`। लेगेसी उपनाम: `voice`, `voiceId`।</ParamField>
    <ParamField path="responseFormat" type='"mp3" | "pcm"'>डिफ़ॉल्ट `mp3`।</ParamField>
    <ParamField path="speed" type="number">प्रदाता-नेटिव गति ओवरराइड।</ParamField>
  </Accordion>

  <Accordion title="Volcengine (BytePlus Seed Speech)">
    <ParamField path="apiKey" type="string">Env: `VOLCENGINE_TTS_API_KEY` या `BYTEPLUS_SEED_SPEECH_API_KEY`।</ParamField>
    <ParamField path="resourceId" type="string">डिफ़ॉल्ट `seed-tts-1.0`। Env: `VOLCENGINE_TTS_RESOURCE_ID`। जब आपके प्रोजेक्ट के पास TTS 2.0 अधिकार हो, तो `seed-tts-2.0` का उपयोग करें।</ParamField>
    <ParamField path="appKey" type="string">ऐप कुंजी हेडर। डिफ़ॉल्ट `aGjiRDfUWi`। Env: `VOLCENGINE_TTS_APP_KEY`।</ParamField>
    <ParamField path="baseUrl" type="string">Seed Speech TTS HTTP एंडपॉइंट को ओवरराइड करें। Env: `VOLCENGINE_TTS_BASE_URL`।</ParamField>
    <ParamField path="speakerVoice" type="string">वॉइस प्रकार। डिफ़ॉल्ट `en_female_anna_mars_bigtts`। Env: `VOLCENGINE_TTS_VOICE`। लेगेसी उपनाम: `voice`।</ParamField>
    <ParamField path="speedRatio" type="number">प्रदाता-नेटिव गति अनुपात, `0.2..3`।</ParamField>
    <ParamField path="emotion" type="string">प्रदाता-नेटिव भाव टैग।</ParamField>
    <ParamField path="appId / token / cluster" type="string" deprecated>लेगेसी Volcengine Speech Console फ़ील्ड। Env: `VOLCENGINE_TTS_APPID`, `VOLCENGINE_TTS_TOKEN`, `VOLCENGINE_TTS_CLUSTER` (डिफ़ॉल्ट `volcano_tts`)।</ParamField>
  </Accordion>

  <Accordion title="xAI">
    <ParamField path="apiKey" type="string">Env: `XAI_API_KEY`।</ParamField>
    <ParamField path="baseUrl" type="string">डिफ़ॉल्ट `https://api.x.ai/v1`। Env: `XAI_BASE_URL`।</ParamField>
    <ParamField path="speakerVoiceId" type="string">डिफ़ॉल्ट `eve`। प्रमाणीकरण के साथ, `openclaw infer tts voices --provider xai` वर्तमान अंतर्निहित कैटलॉग प्राप्त करता है; प्रमाणीकरण के बिना यह ऑफ़लाइन फ़ॉलबैक `ara`, `eve`, `leo`, `rex`, और `sal` सूचीबद्ध करता है। खाते की कस्टम वॉइस ID अंतर्निहित सूची में अनुपस्थित होने पर भी अग्रेषित की जाती हैं। लेगेसी उपनाम: `voiceId`।</ParamField>
    <ParamField path="language" type="string">BCP-47 भाषा कोड या `auto`। डिफ़ॉल्ट `en`।</ParamField>
    <ParamField path="responseFormat" type='"mp3" | "wav" | "pcm" | "mulaw" | "alaw"'>डिफ़ॉल्ट `mp3`।</ParamField>
    <ParamField path="speed" type="number">प्रदाता-नेटिव गति ओवरराइड, `0.7..1.5`।</ParamField>
  </Accordion>

  <Accordion title="Xiaomi MiMo">
    <ParamField path="apiKey" type="string">Env: `XIAOMI_API_KEY`।</ParamField>
    <ParamField path="baseUrl" type="string">डिफ़ॉल्ट `https://api.xiaomimimo.com/v1`। Env: `XIAOMI_BASE_URL`।</ParamField>
    <ParamField path="model" type="string">डिफ़ॉल्ट `mimo-v2.5-tts`। Env: `XIAOMI_TTS_MODEL`। `mimo-v2.5-tts-voicedesign` भी समर्थित है।</ParamField>
    <ParamField path="speakerVoice" type="string">प्रीसेट-वॉइस मॉडल के लिए डिफ़ॉल्ट `mimo_default`। Env: `XIAOMI_TTS_VOICE`। लेगेसी उपनाम: `voice`। `mimo-v2.5-tts-voicedesign` के लिए नहीं भेजा जाता।</ParamField>
    <ParamField path="format" type='"mp3" | "wav"'>डिफ़ॉल्ट `mp3`। Env: `XIAOMI_TTS_FORMAT`।</ParamField>
    <ParamField path="style" type="string">उपयोगकर्ता संदेश के रूप में भेजा गया वैकल्पिक प्राकृतिक-भाषा शैली निर्देश; इसे बोला नहीं जाता। `mimo-v2.5-tts-voicedesign` के लिए, यह वॉइस-डिज़ाइन प्रॉम्प्ट है; छोड़े जाने पर OpenClaw एक डिफ़ॉल्ट प्रदान करता है।</ParamField>
  </Accordion>
</AccordionGroup>

## एजेंट टूल

`tts` टूल टेक्स्ट को स्पीच में बदलता है और उत्तर डिलीवरी के लिए
एक ऑडियो अटैचमेंट लौटाता है। Feishu, Matrix, Telegram और WhatsApp पर ऑडियो को
फ़ाइल अटैचमेंट के बजाय वॉइस संदेश के रूप में डिलीवर किया जाता है। इस पथ पर
`ffmpeg` उपलब्ध होने पर Feishu और WhatsApp गैर-Opus TTS आउटपुट को
ट्रांसकोड कर सकते हैं।

WhatsApp, Baileys के माध्यम से ऑडियो को PTT वॉइस नोट (`audio` के साथ
`ptt: true`) के रूप में भेजता है और दृश्यमान टेक्स्ट को PTT ऑडियो से **अलग**
भेजता है, क्योंकि क्लाइंट वॉइस नोट पर कैप्शन को एकसमान रूप से रेंडर नहीं करते।

टूल वैकल्पिक `channel` और `timeoutMs` फ़ील्ड स्वीकार करता है; `timeoutMs`
प्रति-कॉल प्रदाता अनुरोध टाइमआउट मिलीसेकंड में है। प्रति-कॉल मान
`tts.timeoutMs` को ओवरराइड करते हैं; कॉन्फ़िगर किए गए TTS टाइमआउट किसी भी Plugin-निर्मित
प्रदाता डिफ़ॉल्ट को ओवरराइड करते हैं।

## Gateway RPC

| विधि            | उद्देश्य                                      |
| ----------------- | -------------------------------------------- |
| `tts.status`      | वर्तमान TTS स्थिति और अंतिम प्रयास पढ़ें।     |
| `tts.enable`      | स्थानीय स्वचालित वरीयता को `always` पर सेट करें।       |
| `tts.disable`     | स्थानीय स्वचालित वरीयता को `off` पर सेट करें।          |
| `tts.convert`     | एकबारगी टेक्स्ट → ऑडियो।                        |
| `tts.setProvider` | स्थानीय प्रदाता वरीयता सेट करें।               |
| `tts.personas`    | कॉन्फ़िगर किए गए व्यक्तित्वों और सक्रिय व्यक्तित्व की सूची दिखाएँ। |
| `tts.setPersona`  | स्थानीय व्यक्तित्व वरीयता सेट करें।                |
| `tts.providers`   | कॉन्फ़िगर किए गए प्रदाताओं और उनकी स्थिति की सूची दिखाएँ।        |

## सेवा लिंक

- [OpenAI टेक्स्ट-टू-स्पीच मार्गदर्शिका](https://platform.openai.com/docs/guides/text-to-speech)
- [OpenAI Audio API संदर्भ](https://platform.openai.com/docs/api-reference/audio)
- [Azure Speech REST टेक्स्ट-टू-स्पीच](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech)
- [Azure Speech प्रदाता](/hi/providers/azure-speech)
- [ElevenLabs टेक्स्ट टू स्पीच](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [ElevenLabs प्रमाणीकरण](https://elevenlabs.io/docs/api-reference/authentication)
- [Gradium](/hi/providers/gradium)
- [Inworld TTS API](https://docs.inworld.ai/tts/tts)
- [MiniMax T2A v2 API](https://platform.minimaxi.com/document/T2A%20V2)
- [Volcengine TTS HTTP API](/hi/providers/volcengine#text-to-speech)
- [Xiaomi MiMo वाक् संश्लेषण](/hi/providers/xiaomi#text-to-speech)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [Microsoft Speech आउटपुट प्रारूप](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)
- [xAI टेक्स्ट टू स्पीच](https://docs.x.ai/developers/rest-api-reference/inference/voice#text-to-speech-rest)

## संबंधित

- [मीडिया अवलोकन](/hi/tools/media-overview)
- [संगीत निर्माण](/hi/tools/music-generation)
- [वीडियो निर्माण](/hi/tools/video-generation)
- [स्लैश कमांड](/hi/tools/slash-commands)
- [वॉइस कॉल Plugin](/hi/plugins/voice-call)
