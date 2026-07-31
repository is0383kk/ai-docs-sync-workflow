---
read_when:
    - आपको यह जानना आवश्यक है कि कौन-से env vars लोड किए जाते हैं और किस क्रम में।
    - आप Gateway में अनुपलब्ध API कुंजियों की समस्या का निदान कर रहे हैं
    - आप प्रदाता प्रमाणीकरण या परिनियोजन परिवेशों का दस्तावेज़ीकरण कर रहे हैं
summary: OpenClaw पर्यावरण चर कहाँ से लोड करता है और उनकी प्राथमिकता का क्रम
title: पर्यावरण चर
x-i18n:
    generated_at: "2026-07-27T19:24:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: db9990dea5df7731e54c8d442f4704bd4d6e0caf6f2c2fdea32d2583cd41128c
    source_path: help/environment.md
    workflow: 16
---

OpenClaw कई स्रोतों से पर्यावरण चर प्राप्त करता है। नियम है: **मौजूदा मानों को कभी ओवरराइड न करें**।
वर्कस्पेस `.env` फ़ाइलें कम-विश्वसनीय स्रोत हैं: प्राथमिकता लागू करने से पहले OpenClaw वर्कस्पेस `.env` से प्रदाता क्रेडेंशियल और संरक्षित रनटाइम नियंत्रणों को अनदेखा करता है।

## प्राथमिकता (उच्चतम से निम्नतम)

1. **प्रक्रिया पर्यावरण** (जो Gateway प्रक्रिया को पैरेंट शेल/डेमन से पहले ही मिला हुआ है)।
2. **वर्तमान कार्यशील डायरेक्टरी में `.env`** (dotenv डिफ़ॉल्ट; ओवरराइड नहीं करता; प्रदाता क्रेडेंशियल और संरक्षित रनटाइम नियंत्रण अनदेखे किए जाते हैं)।
3. `~/.openclaw/.env` पर **वैश्विक `.env`** (जिसे `$OPENCLAW_STATE_DIR/.env` भी कहा जाता है; प्रदाता API कुंजियों के लिए अनुशंसित; ओवरराइड नहीं करता)।
4. `~/.openclaw/openclaw.json` में **कॉन्फ़िग `env` ब्लॉक** (केवल अनुपस्थित होने पर लागू होता है)।
5. **वैकल्पिक लॉगिन-शेल आयात** (`env.shellEnv.enabled` या `OPENCLAW_LOAD_SHELL_ENV=1`), केवल अनुपस्थित अपेक्षित कुंजियों के लिए लागू होता है।

डिफ़ॉल्ट स्टेट डायरेक्टरी का उपयोग करने वाले नए Ubuntu इंस्टॉल पर, OpenClaw वैश्विक `.env` के बाद `~/.config/openclaw/gateway.env` को संगतता फ़ॉलबैक भी मानता है। यदि दोनों फ़ाइलें मौजूद हों और उनमें अंतर हो, तो OpenClaw `~/.openclaw/.env` को बनाए रखता है और चेतावनी प्रिंट करता है।

यदि कॉन्फ़िग फ़ाइल पूरी तरह अनुपस्थित है, तो चरण 4 छोड़ दिया जाता है; सक्षम होने पर शेल आयात फिर भी चलता है।

## ऑपरेटर के लिए समर्थित चर

नीचे दिए गए चर ऑपरेटरों के लिए समर्थित पर्यावरण अनुबंध हैं। बिना दस्तावेज़ वाले `OPENCLAW_*` चर आंतरिक कार्यान्वयन विवरण हैं और बिना सूचना के हट सकते हैं।

### पथ और इंस्टेंस

| चर                       | उद्देश्य                                                           |
| ------------------------ | ----------------------------------------------------------------- |
| `OPENCLAW_HOME`          | OpenClaw पथ डिफ़ॉल्ट के लिए उपयोग की जाने वाली होम डायरेक्टरी को ओवरराइड करें।      |
| `OPENCLAW_STATE_DIR`     | परिवर्तनीय स्टेट डायरेक्टरी को ओवरराइड करें।                             |
| `OPENCLAW_CONFIG_PATH`   | सक्रिय कॉन्फ़िग फ़ाइल पथ को ओवरराइड करें।                             |
| `OPENCLAW_WORKSPACE_DIR` | डिफ़ॉल्ट एजेंट वर्कस्पेस को ओवरराइड करें।                             |
| `OPENCLAW_PROFILE`       | किसी नामित प्रोफ़ाइल और उसके पृथक डिफ़ॉल्ट का चयन करें।                 |
| `OPENCLAW_GIT_DIR`       | डेवलपमेंट-चैनल अपडेट द्वारा उपयोग किए जाने वाले स्रोत चेकआउट को ओवरराइड करें। |
| `OPENCLAW_INCLUDE_ROOTS` | `$include` को अतिरिक्त रूट से रिज़ॉल्व होने दें।                |

### Gateway और प्रमाणीकरण

| चर                          | उद्देश्य                                                         |
| --------------------------- | --------------------------------------------------------------- |
| `OPENCLAW_GATEWAY_URL`      | क्लाइंट द्वारा उपयोग किए जाने वाले रिमोट Gateway URL को ओवरराइड करें।                |
| `OPENCLAW_GATEWAY_PORT`     | स्थानीय Gateway पोर्ट को ओवरराइड करें।                                |
| `OPENCLAW_GATEWAY_TOKEN`    | Gateway सर्वर और क्लाइंट के लिए टोकन प्रमाणीकरण प्रदान करें।    |
| `OPENCLAW_GATEWAY_PASSWORD` | Gateway सर्वर और क्लाइंट के लिए पासवर्ड प्रमाणीकरण प्रदान करें। |

### प्रदाता क्रेडेंशियल

कोर और बंडल किए गए प्रदाता plugins निम्नलिखित क्रेडेंशियल और प्रदाता-चयन चरों को पहचानते हैं। जब आपको पूरी प्रक्रिया में एक ही मान के बजाय सीमित दायरे वाले क्रेडेंशियल चाहिए, तो प्रत्येक प्रदाता के कॉन्फ़िग या SecretRef फ़ील्ड को प्राथमिकता दें।

`AI_GATEWAY_API_KEY`, `ANTHROPIC_ADMIN_API_KEY`, `ANTHROPIC_ADMIN_KEY`, `ANTHROPIC_API_KEY`, `ANTHROPIC_OAUTH_TOKEN`, `ARCEEAI_API_KEY`, `AZURE_OPENAI_API_KEY`, `AZURE_SPEECH_API_KEY`, `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION`, `BASETEN_API_KEY`, `BRAVE_API_KEY`, `BYTEPLUS_API_KEY`, `BYTEPLUS_SEED_SPEECH_API_KEY`, `CEREBRAS_API_KEY`, `CHUTES_API_KEY`, `CHUTES_OAUTH_TOKEN`, `CLAWROUTER_API_KEY`, `CLOUDFLARE_AI_GATEWAY_API_KEY`, `CODEX_API_KEY`, `COHERE_API_KEY`, `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY`, `COPILOT_GITHUB_TOKEN`, `DASHSCOPE_API_KEY`, `DEEPGRAM_API_KEY`, `DEEPINFRA_API_KEY`, `DEEPSEEK_API_KEY`, `ELEVENLABS_API_KEY`, `EXA_API_KEY`, `FAL_API_KEY`, `FAL_KEY`, `FEATHERLESS_API_KEY`, `FIRECRAWL_API_KEY`, `FIREWORKS_API_KEY`, `GCLOUD_PROJECT`, `GEMINI_API_KEY`, `GH_TOKEN`, `GITHUB_TOKEN`, `GMI_API_KEY`, `GOOGLE_API_KEY`, `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_API_KEY`, `GOOGLE_CLOUD_LOCATION`, `GOOGLE_CLOUD_PROJECT`, `GRADIUM_API_KEY`, `GROQ_API_KEY`, `HF_TOKEN`, `HUGGINGFACE_HUB_TOKEN`, `INWORLD_API_KEY`, `KILOCODE_API_KEY`, `KIMICODE_API_KEY`, `KIMI_API_KEY`, `LITELLM_API_KEY`, `LM_API_TOKEN`, `LONGCAT_API_KEY`, `MINIMAX_API_KEY`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN`, `MISTRAL_API_KEY`, `MODELSTUDIO_API_KEY`, `MODEL_API_KEY`, `MOONSHOT_API_KEY`, `NOVITA_API_KEY`, `NVIDIA_API_KEY`, `OLLAMA_API_KEY`, `OPENAI_ADMIN_KEY`, `OPENAI_API_KEY`, `OPENCODE_API_KEY`, `OPENCODE_ZEN_API_KEY`, `OPENROUTER_API_KEY`, `PARALLEL_API_KEY`, `PERPLEXITY_API_KEY`, `PIXVERSE_API_KEY`, `QIANFAN_API_KEY`, `QWEN_API_KEY`, `QWEN_TOKEN_PLAN_API_KEY`, `RUNWAYML_API_SECRET`, `RUNWAY_API_KEY`, `SENSEAUDIO_API_KEY`, `SGLANG_API_KEY`, `SPEECH_KEY`, `SPEECH_REGION`, `STEPFUN_API_KEY`, `SYNTHETIC_API_KEY`, `TAVILY_API_KEY`, `TOGETHER_API_KEY`, `TOKENHUB_API_KEY`, `TOKENPLAN_API_KEY`, `VENICE_API_KEY`, `VLLM_API_KEY`, `VOLCANO_ENGINE_API_KEY`, `VOLCENGINE_TTS_API_KEY`, `VOLCENGINE_TTS_APPID`, `VOLCENGINE_TTS_TOKEN`, `VOYAGE_API_KEY`, `VYDRA_API_KEY`, `XAI_API_KEY`, `XIAOMI_API_KEY`, `XIAOMI_TOKEN_PLAN_API_KEY`, `XI_API_KEY`, `ZAI_API_KEY`, और `Z_AI_API_KEY`।

इंस्टॉल किए गए तृतीय-पक्ष plugins अपने plugin मैनिफ़ेस्ट में अतिरिक्त क्रेडेंशियल चर घोषित कर सकते हैं; वे चर उन्हें घोषित करने वाले plugin के अनुबंध हैं, कोर OpenClaw चर नहीं।

### लॉगिंग और निदान

| चर                                   | उद्देश्य                                                       |
| ------------------------------------ | ------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`                 | फ़ाइल और कंसोल लॉग स्तरों को ओवरराइड करें।                         |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT`     | मॉडल ट्रांसपोर्ट टाइमिंग निदान सक्षम करें।                    |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`       | संपादित मॉडल पेलोड निदान चुनें।                    |
| `OPENCLAW_DEBUG_SSE`                 | SSE टाइमिंग या इवेंट-पीक निदान चुनें।                  |
| `OPENCLAW_DEBUG_CODE_MODE`           | कोड-मोड सतह निदान सक्षम करें।                         |
| `OPENCLAW_DIAGNOSTICS`               | नामित निदान फ़्लैग सक्षम करें, या `0` से सभी फ़्लैग अक्षम करें। |
| `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH` | टाइमलाइन निदान के लिए JSONL पथ चुनें।               |
| `OPENCLAW_DIAGNOSTICS_EVENT_LOOP`    | टाइमलाइन निदान में इवेंट-लूप नमूने जोड़ें।               |

### सुविधा और रनटाइम टॉगल

| चर                                   | उद्देश्य                                                                      |
| ------------------------------------ | ---------------------------------------------------------------------------- |
| `OPENCLAW_LOAD_SHELL_ENV`            | लॉगिन शेल से अनुपस्थित अपेक्षित चर आयात करें।                      |
| `OPENCLAW_SHELL_ENV_TIMEOUT_MS`      | लॉगिन-शेल आयात टाइमआउट सेट करें।                                          |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT`       | `0` से exec शेल स्नैपशॉट अक्षम करें।                                       |
| `OPENCLAW_OFFLINE`                   | पिन किए गए एजेंट सहायक बाइनरी के डाउनलोड रोकें।                           |
| `OPENCLAW_BROWSER_HEADLESS`          | प्रबंधित ब्राउज़र लॉन्च को हेडेड (`0`) या हेडलेस (`1`) के लिए बाध्य करें।               |
| `OPENCLAW_DISABLE_BONJOUR`           | Bonjour विज्ञापन को चालू (`0`) या बंद (`1`) करने के लिए बाध्य करें।                             |
| `OPENCLAW_NO_AUTO_UPDATE`            | स्वचालित अपडेट लागू करना अक्षम करें।                                            |
| `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS` | आपातकालीन ओवरराइड के रूप में विश्वसनीय निजी-DNS `ws://` कनेक्शन की अनुमति दें।     |
| `OPENCLAW_ALLOW_MULTI_GATEWAY`       | प्रति-स्टेट स्वामित्व लॉक बनाए रखते हुए कई Gateway प्रक्रियाओं की अनुमति दें। |
| `OPENCLAW_SKIP_CHANNELS`             | समस्या निवारण के लिए चैनल ट्रांसपोर्ट के बिना Gateway प्रारंभ करें।            |
| `OPENCLAW_THEME`                     | TUI पैलेट को `light` या `dark` के लिए बाध्य करें।                                  |

## प्रदाता क्रेडेंशियल और वर्कस्पेस `.env`

प्रदाता API कुंजियों को केवल वर्कस्पेस `.env` में न रखें। OpenClaw वर्कस्पेस `.env` फ़ाइलों से प्रदाता क्रेडेंशियल और एंडपॉइंट-रीडायरेक्ट कुंजियों के एक बड़े समूह को ब्लॉक करता है, जिसमें हर ज्ञात प्रदाता प्रमाणीकरण env var (उदाहरण के लिए `GEMINI_API_KEY`, `GOOGLE_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY`, `GROQ_API_KEY`, `DEEPSEEK_API_KEY`, `PERPLEXITY_API_KEY`, `BRAVE_API_KEY`, `TAVILY_API_KEY`, `EXA_API_KEY`, `FIRECRAWL_API_KEY`), साथ ही `_API_HOST`, `_BASE_URL`, `_ENDPOINT`, या `_HOMESERVER` पर समाप्त होने वाली कोई भी कुंजी, और संपूर्ण `OPENCLAW_*`, `CLAWHUB_*`, `ANTHROPIC_API_KEY_*`, और `OPENAI_API_KEY_*` नेमस्पेस शामिल हैं।

इसके बजाय प्रदाता क्रेडेंशियल के लिए इन विश्वसनीय स्रोतों में से किसी एक का उपयोग करें:

- Gateway प्रक्रिया पर्यावरण, जैसे कोई शेल, launchd/systemd यूनिट, कंटेनर सीक्रेट या CI सीक्रेट।
- `~/.openclaw/.env` या `$OPENCLAW_STATE_DIR/.env` पर वैश्विक रनटाइम dotenv फ़ाइल।
- `~/.openclaw/openclaw.json` में कॉन्फ़िग `env` ब्लॉक।
- `env.shellEnv.enabled` या `OPENCLAW_LOAD_SHELL_ENV=1` सक्षम होने पर वैकल्पिक लॉगिन-शेल आयात।

यदि आपने पहले प्रदाता कुंजियाँ या एंडपॉइंट रूटिंग मान केवल वर्कस्पेस `.env` में संग्रहीत किए थे, तो उन्हें ऊपर दिए गए विश्वसनीय स्रोतों में से किसी एक में ले जाएँ। वर्कस्पेस `.env` अब भी ऐसे सामान्य प्रोजेक्ट चर प्रदान कर सकता है जो क्रेडेंशियल, एंडपॉइंट रीडायरेक्ट, होस्ट ओवरराइड या `OPENCLAW_*` रनटाइम नियंत्रण नहीं हैं।

सुरक्षा तर्क के लिए [वर्कस्पेस `.env` फ़ाइलें](/hi/gateway/security#workspace-env-files) देखें।

## कॉन्फ़िग `env` ब्लॉक

इनलाइन env vars सेट करने के दो समतुल्य तरीके (दोनों ओवरराइड नहीं करते):

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
  },
}
```

कॉन्फ़िग `env` ब्लॉक केवल शाब्दिक स्ट्रिंग मान स्वीकार करता है। यह
`file:...` मानों का विस्तार नहीं करता; उदाहरण के लिए, `XAI_API_KEY: "file:secrets/xai-api-key.txt"`
प्रदाताओं को ठीक उसी स्ट्रिंग के रूप में दिया जाता है।

फ़ाइल-आधारित प्रदाता कुंजियों के लिए, उस क्रेडेंशियल फ़ील्ड पर SecretRef का उपयोग करें जो
इसका समर्थन करता है:

```json5
{
  secrets: {
    providers: {
      xai_key_file: {
        source: "file",
        path: "~/.openclaw/secrets/xai-api-key.txt",
        mode: "singleValue",
      },
    },
  },
  models: {
    providers: {
      xai: {
        apiKey: { source: "file", provider: "xai_key_file", id: "value" },
      },
    },
  },
}
```

समर्थित फ़ील्ड के लिए [सीक्रेट प्रबंधन](/hi/gateway/secrets) और
[SecretRef क्रेडेंशियल सतह](/hi/reference/secretref-credential-surface)
देखें।

## शेल env आयात

`env.shellEnv` आपका लॉगिन शेल चलाता है और केवल **अनुपस्थित** अपेक्षित कुंजियाँ आयात करता है:

```json5
{
  env: {
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

समतुल्य env var:

- `OPENCLAW_LOAD_SHELL_ENV=1`
- `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000` (डिफ़ॉल्ट `15000`)

## Exec शेल स्नैपशॉट

गैर-Windows Gateway होस्ट पर, bash और zsh `exec` कमांड डिफ़ॉल्ट रूप से स्टार्टअप स्नैपशॉट का उपयोग करते हैं।
इस पथ को अक्षम करने के लिए Gateway प्रक्रिया पर्यावरण में `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` सेट करें।
`false`, `no`, और `off` मान भी इसे अक्षम करते हैं। प्रति-कॉल `exec.env` मान
स्नैपशॉट को टॉगल या स्नैपशॉट कैश को रीडायरेक्ट नहीं कर सकते।

## रनटाइम द्वारा इंजेक्ट किए गए env vars

OpenClaw प्रारंभ की गई चाइल्ड प्रक्रियाओं में संदर्भ मार्कर भी इंजेक्ट करता है:

- `OPENCLAW_SHELL=exec`: `exec` टूल के माध्यम से चलाए गए कमांड के लिए सेट किया जाता है।
- `OPENCLAW_SHELL=acp-client`: जब `openclaw acp client` ACP ब्रिज प्रक्रिया आरंभ करता है, तब उसके लिए सेट किया जाता है।
- `OPENCLAW_SHELL=tui-local`: स्थानीय TUI `!` शेल कमांड के लिए सेट किया जाता है।
- `OPENCLAW_CLI=1`: CLI प्रवेश बिंदु द्वारा आरंभ की गई चाइल्ड प्रक्रियाओं के लिए सेट किया जाता है।

ये रनटाइम मार्कर हैं (आवश्यक उपयोगकर्ता कॉन्फ़िगरेशन नहीं)। संदर्भ-विशिष्ट नियम लागू करने के लिए
इनका उपयोग शेल/प्रोफ़ाइल लॉजिक में किया जा सकता है।

## UI परिवेश चर

- `OPENCLAW_THEME=light`: यदि आपके टर्मिनल की पृष्ठभूमि हल्की है, तो हल्की TUI रंग-संयोजना को बाध्य करें।
- `OPENCLAW_THEME=dark`: गहरी TUI रंग-संयोजना को बाध्य करें।
- `COLORFGBG`: यदि आपका टर्मिनल इसे एक्सपोर्ट करता है, तो OpenClaw TUI रंग-संयोजना को स्वतः चुनने के लिए पृष्ठभूमि रंग संकेत का उपयोग करता है।

## कॉन्फ़िगरेशन में परिवेश चर प्रतिस्थापन

आप `${VAR_NAME}` सिंटैक्स का उपयोग करके कॉन्फ़िगरेशन के स्ट्रिंग मानों में सीधे परिवेश चरों का संदर्भ दे सकते हैं:

```json5
{
  models: {
    providers: {
      "vercel-gateway": {
        apiKey: "${VERCEL_GATEWAY_API_KEY}",
      },
    },
  },
}
```

पूरी जानकारी के लिए [कॉन्फ़िगरेशन: परिवेश चर प्रतिस्थापन](/hi/gateway/configuration-reference#env-var-substitution) देखें।

## सीक्रेट संदर्भ बनाम `${ENV}` स्ट्रिंग

OpenClaw परिवेश-आधारित दो प्रतिरूपों का समर्थन करता है:

- कॉन्फ़िगरेशन मानों में `${VAR}` स्ट्रिंग प्रतिस्थापन।
- सीक्रेट संदर्भों का समर्थन करने वाले फ़ील्ड के लिए SecretRef ऑब्जेक्ट (`{ source: "env", provider: "default", id: "VAR" }`)।

सक्रियण के समय दोनों का समाधान प्रक्रिया परिवेश से किया जाता है। SecretRef का विवरण [सीक्रेट प्रबंधन](/hi/gateway/secrets) में प्रलेखित है।
कॉन्फ़िगरेशन का `env` ब्लॉक स्वयं SecretRefs या `file:...`
संक्षिप्त मानों का समाधान नहीं करता।

## पथ-संबंधी परिवेश चर

| चर                 | उद्देश्य                                                                                                                                                                                                                                 |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_HOME`          | आंतरिक OpenClaw पथ डिफ़ॉल्ट के लिए उपयोग की जाने वाली होम डायरेक्टरी को ओवरराइड करें (`~/.openclaw/`, एजेंट डायरेक्टरियाँ, सत्र, क्रेडेंशियल, इंस्टॉलर ऑनबोर्डिंग और डिफ़ॉल्ट डेवलपमेंट चेकआउट)। OpenClaw को समर्पित सेवा उपयोगकर्ता के रूप में चलाते समय उपयोगी। |
| `OPENCLAW_STATE_DIR`     | स्टेट डायरेक्टरी को ओवरराइड करें (डिफ़ॉल्ट `~/.openclaw`)।                                                                                                                                                                                   |
| `OPENCLAW_CONFIG_PATH`   | कॉन्फ़िगरेशन फ़ाइल पथ को ओवरराइड करें (डिफ़ॉल्ट `~/.openclaw/openclaw.json`)।                                                                                                                                                                    |
| `OPENCLAW_INCLUDE_ROOTS` | उन डायरेक्टरियों की पथ-सूची जहाँ `$include` निर्देश कॉन्फ़िगरेशन डायरेक्टरी के बाहर की फ़ाइलों का समाधान कर सकते हैं (डिफ़ॉल्ट: कोई नहीं - `$include` कॉन्फ़िगरेशन डायरेक्टरी तक सीमित रहता है)। टिल्ड विस्तार लागू होता है।                                                         |

## एजेंट सहायक टूल डाउनलोड

OpenClaw को उसके पिन किए गए `fd`
और `ripgrep` सहायक बाइनरी डाउनलोड करने से रोकने के लिए `OPENCLAW_OFFLINE=1` सेट करें। OpenClaw टूल
डायरेक्टरी के अंतर्गत मौजूदा सहायक और कार्यशील सिस्टम बाइनरी उपयोग के योग्य बने रहते हैं; अनुपलब्ध सहायक
नेटवर्क अनुरोध आरंभ करने के बजाय अनुपलब्ध ही रहता है।

## लॉगिंग

| चर                         | उद्देश्य                                                                                                                                                                                      |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`             | फ़ाइल और कंसोल दोनों के लिए लॉग स्तर ओवरराइड करें (जैसे `debug`, `trace`)। यह कॉन्फ़िगरेशन में `logging.level` और `logging.consoleLevel` से अधिक प्राथमिकता लेता है। अमान्य मान चेतावनी के साथ अनदेखे कर दिए जाते हैं। |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT` | वैश्विक डीबग लॉग सक्षम किए बिना `info` स्तर पर लक्षित मॉडल अनुरोध/प्रतिक्रिया समय-निर्धारण निदान उत्सर्जित करें।                                                                                  |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`   | मॉडल पेलोड निदान: `summary`, `tools`, या `full-redacted`। `full-redacted` सीमित और संशोधित होता है, लेकिन इसमें प्रॉम्प्ट/संदेश का टेक्स्ट शामिल हो सकता है।                                               |
| `OPENCLAW_DEBUG_SSE`             | स्ट्रीमिंग निदान: प्रथम/पूर्ण समय-निर्धारण के लिए `events`, पहले पाँच संशोधित SSE इवेंट शामिल करने के लिए `peek`।                                                                                 |
| `OPENCLAW_DEBUG_CODE_MODE`       | कोड-मोड मॉडल-सतह निदान, जिसमें प्रदाता-टूल छिपाना और संक्षिप्त नियंत्रण/प्रत्यक्ष प्रवर्तन शामिल हैं।                                                                                  |

### `OPENCLAW_HOME`

सेट होने पर, आंतरिक OpenClaw पथ डिफ़ॉल्ट के लिए `OPENCLAW_HOME` सिस्टम होम डायरेक्टरी (`$HOME` / `os.homedir()`) को प्रतिस्थापित करता है। इसमें डिफ़ॉल्ट स्टेट डायरेक्टरी, कॉन्फ़िगरेशन पथ, एजेंट डायरेक्टरियाँ, क्रेडेंशियल, इंस्टॉलर ऑनबोर्डिंग कार्यस्थान और `openclaw update --channel dev` द्वारा उपयोग किया जाने वाला डिफ़ॉल्ट डेवलपमेंट चेकआउट शामिल है।

**प्राथमिकता:** `OPENCLAW_HOME` > `$HOME` > `USERPROFILE` > Android पर Termux `PREFIX` होम फ़ॉलबैक > `os.homedir()`

**उदाहरण** (macOS LaunchDaemon):

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>OPENCLAW_HOME</key>
  <string>/Users/user</string>
</dict>
```

`OPENCLAW_HOME` को टिल्ड पथ (जैसे `~/svc`) पर भी सेट किया जा सकता है, जिसे उपयोग से पहले उसी OS होम फ़ॉलबैक शृंखला से विस्तारित किया जाता है।

`OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH`, और `OPENCLAW_GIT_DIR` जैसे स्पष्ट पथ चर फिर भी अधिक प्राथमिकता लेते हैं। शेल स्टार्टअप फ़ाइल पहचान, पैकेज-मैनेजर सेटअप और होस्ट `~` विस्तार जैसे OS-अकाउंट कार्य अभी भी वास्तविक सिस्टम होम का उपयोग कर सकते हैं।

## nvm उपयोगकर्ता: web_fetch TLS विफलताएँ

यदि Node.js को सिस्टम पैकेज मैनेजर के बजाय **nvm** के माध्यम से इंस्टॉल किया गया था, तो अंतर्निहित `fetch()`
nvm के साथ बंडल किए गए CA स्टोर का उपयोग करता है, जिसमें आधुनिक रूट CA (Let's Encrypt के लिए ISRG Root X1/X2,
DigiCert Global Root G2 आदि) अनुपस्थित हो सकते हैं। इसके कारण अधिकांश HTTPS साइटों पर `web_fetch`, `"fetch failed"` के साथ विफल हो जाता है।

Linux पर, OpenClaw स्वचालित रूप से nvm का पता लगाता है और वास्तविक स्टार्टअप परिवेश में सुधार लागू करता है:

- `openclaw gateway install` सिस्टमd सेवा परिवेश में `NODE_EXTRA_CA_CERTS` लिखता है
- `openclaw` CLI प्रवेश बिंदु Node स्टार्टअप से पहले `NODE_EXTRA_CA_CERTS` सेट करके स्वयं को फिर से निष्पादित करता है

**मैन्युअल सुधार (पुराने संस्करणों या सीधे `node ...` लॉन्च के लिए):**

OpenClaw शुरू करने से पहले चर को एक्सपोर्ट करें:

```bash
export NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt
openclaw gateway run
```

इस चर के लिए केवल `~/.openclaw/.env` में लिखने पर निर्भर न रहें; Node प्रक्रिया स्टार्टअप के समय
`NODE_EXTRA_CA_CERTS` को पढ़ता है।

## पुराने परिवेश चर

OpenClaw केवल `OPENCLAW_*` परिवेश चर पढ़ता है। पुराने रिलीज़ के
`CLAWDBOT_*` और `MOLTBOT_*` उपसर्गों को बिना सूचना के
अनदेखा कर दिया जाता है।

यदि स्टार्टअप के समय Gateway प्रक्रिया पर इनमें से कोई अभी भी सेट है, तो OpenClaw एकल
Node अप्रचलन चेतावनी (`OPENCLAW_LEGACY_ENV_VARS`) उत्सर्जित करता है, जिसमें पता लगाए गए
उपसर्ग और कुल संख्या सूचीबद्ध होती है। पुराने उपसर्ग को `OPENCLAW_` से
प्रतिस्थापित करके प्रत्येक मान का नाम बदलें (उदाहरण के लिए `CLAWDBOT_GATEWAY_TOKEN` से
`OPENCLAW_GATEWAY_TOKEN`); पुराने नामों का कोई प्रभाव नहीं पड़ता।

## संबंधित

- [Gateway कॉन्फ़िगरेशन](/hi/gateway/configuration)
- [अक्सर पूछे जाने वाले प्रश्न: परिवेश चर और .env लोडिंग](/hi/help/faq#env-vars-and-env-loading)
- [मॉडल का अवलोकन](/hi/concepts/models)
