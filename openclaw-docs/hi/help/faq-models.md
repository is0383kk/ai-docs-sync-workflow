---
read_when:
    - मॉडल चुनना या बदलना, उपनाम कॉन्फ़िगर करना
    - मॉडल फ़ेलओवर / "सभी मॉडल विफल रहे" की डीबगिंग
    - प्रमाणीकरण प्रोफ़ाइलों को समझना और उन्हें प्रबंधित करना
sidebarTitle: Models FAQ
summary: 'अक्सर पूछे जाने वाले प्रश्न: मॉडल डिफ़ॉल्ट, चयन, उपनाम, स्विचिंग, फ़ेलओवर और प्रमाणीकरण प्रोफ़ाइल'
title: 'अक्सर पूछे जाने वाले प्रश्न: मॉडल और प्रमाणीकरण'
x-i18n:
    generated_at: "2026-07-27T19:55:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0c46d99352c5e51af5917c6f62b897dfa4030cb0201b8235e28f2f81f2954544
    source_path: help/faq-models.md
    workflow: 16
---

मॉडल और ऑथ-प्रोफ़ाइल से जुड़े प्रश्नोत्तर। सेटअप, सत्रों, Gateway, चैनलों और
समस्या निवारण के लिए मुख्य [अक्सर पूछे जाने वाले प्रश्न](/hi/help/faq) देखें।

## मॉडल: डिफ़ॉल्ट, चयन, उपनाम, स्विच करना

<AccordionGroup>
  <Accordion title='"डिफ़ॉल्ट मॉडल" क्या है?'>
    इसे इससे सेट करें:

    ```text
    agents.defaults.model.primary
    ```

    मॉडल `provider/model` संदर्भ होते हैं (उदाहरण: `openai/gpt-5.5`,
    `anthropic/claude-sonnet-4-6`)। हमेशा `provider/model` को स्पष्ट रूप से सेट करें। यदि
    आप प्रदाता को छोड़ देते हैं, तो OpenClaw पहले उपनाम मिलान आज़माता है, फिर उस मॉडल आईडी के लिए
    विशिष्ट कॉन्फ़िगर किए गए प्रदाता का मिलान करता है, और फिर
    कॉन्फ़िगर किए गए डिफ़ॉल्ट प्रदाता पर वापस जाता है (अप्रचलित संगतता पथ)। यदि उस
    प्रदाता के पास अब कॉन्फ़िगर किया गया डिफ़ॉल्ट मॉडल नहीं है, तो OpenClaw पुराने डिफ़ॉल्ट के बजाय
    पहले कॉन्फ़िगर किए गए प्रदाता/मॉडल पर वापस जाता है।

  </Accordion>

  <Accordion title="आप किस मॉडल की अनुशंसा करते हैं?">
    आपके प्रदाता स्टैक द्वारा उपलब्ध कराया गया नवीनतम पीढ़ी का सबसे शक्तिशाली मॉडल उपयोग करें,
    विशेष रूप से टूल-सक्षम या अविश्वसनीय इनपुट वाले एजेंटों के लिए — कमजोर या
    अत्यधिक क्वांटाइज़ किए गए मॉडल प्रॉम्प्ट इंजेक्शन और असुरक्षित
    व्यवहार के प्रति अधिक संवेदनशील होते हैं ([सुरक्षा](/hi/gateway/security) देखें)। एजेंट की भूमिका के अनुसार
    नियमित/कम-जोखिम वाली चैट के लिए कम लागत वाले मॉडल रूट करें।

    मॉडल को प्रत्येक एजेंट के अनुसार रूट करें और लंबे कार्यों को समानांतर करने के लिए उप-एजेंटों का उपयोग करें (प्रत्येक
    उप-एजेंट अपने स्वयं के टोकन उपयोग करता है)। [मॉडल](/hi/concepts/models),
    [उप-एजेंट](/hi/tools/subagents), [MiniMax](/hi/providers/minimax), और
    [स्थानीय मॉडल](/hi/gateway/local-models) देखें।

  </Accordion>

  <Accordion title="अपनी कॉन्फ़िग मिटाए बिना मॉडल कैसे बदलें?">
    केवल मॉडल फ़ील्ड बदलें — पूरी कॉन्फ़िग को प्रतिस्थापित करने से बचें।

    - चैट में `/model` (प्रति सत्र, [स्लैश कमांड](/hi/tools/slash-commands) देखें)
    - `openclaw models set ...` (केवल मॉडल कॉन्फ़िग अपडेट करता है)
    - `openclaw configure --section model` (इंटरैक्टिव)
    - `~/.openclaw/openclaw.json` में सीधे `agents.defaults.model` संपादित करें

    RPC संपादनों के लिए, पहले `config.schema.lookup` से निरीक्षण करें (सामान्यीकृत
    पथ, संक्षिप्त स्कीमा दस्तावेज़, चाइल्ड सारांश), फिर आंशिक ऑब्जेक्ट के साथ `config.apply`
    के बजाय `config.patch` को प्राथमिकता दें। यदि आपने कॉन्फ़िग ओवरराइट कर दी है,
    तो बैकअप से पुनर्स्थापित करें या सुधार के लिए `openclaw doctor` चलाएँ।

    दस्तावेज़: [मॉडल](/hi/concepts/models), [कॉन्फ़िगर करें](/hi/cli/configure),
    [कॉन्फ़िग](/hi/cli/config), [Doctor](/hi/gateway/doctor)।

  </Accordion>

  <Accordion title="क्या स्व-होस्ट किए गए मॉडल (llama.cpp, vLLM, Ollama) उपयोग किए जा सकते हैं?">
    हाँ — Ollama सबसे आसान तरीका है। त्वरित सेटअप:

    1. `https://ollama.com/download` से Ollama इंस्टॉल करें
    2. कोई स्थानीय मॉडल पुल करें, जैसे `ollama pull gemma4`
    3. क्लाउड मॉडल के लिए भी `ollama signin` चलाएँ
    4. `openclaw onboard` चलाएँ, `Ollama` चुनें, फिर `Local` या `Cloud + Local`

    `Cloud + Local` आपको क्लाउड मॉडल के साथ आपके स्थानीय Ollama मॉडल भी देता है;
    `kimi-k2.5:cloud` जैसे क्लाउड मॉडल के लिए स्थानीय पुल आवश्यक नहीं है। मैन्युअल रूप से स्विच करने के लिए:
    `openclaw models list`, फिर `openclaw models set ollama/<model>`।

    छोटे/अत्यधिक क्वांटाइज़ किए गए मॉडल प्रॉम्प्ट इंजेक्शन के प्रति अधिक संवेदनशील होते हैं।
    टूल एक्सेस वाले किसी भी बॉट के लिए बड़े मॉडल उपयोग करें; यदि फिर भी छोटे मॉडल उपयोग करते हैं,
    तो सैंडबॉक्सिंग और सख्त टूल अनुमतिसूचियाँ सक्षम करें।

    दस्तावेज़: [Ollama](/hi/providers/ollama), [स्थानीय मॉडल](/hi/gateway/local-models),
    [मॉडल प्रदाता](/hi/concepts/model-providers), [सुरक्षा](/hi/gateway/security),
    [सैंडबॉक्सिंग](/hi/gateway/sandboxing)।

  </Accordion>

  <Accordion title="बिना पुनः आरंभ किए तुरंत मॉडल कैसे बदलें?">
    `/model <name>` को एक स्वतंत्र संदेश के रूप में भेजें। संपूर्ण कमांड सूची के लिए
    [स्लैश कमांड](/hi/tools/slash-commands) देखें, जिसमें क्रमांकित चयनकर्ता
    (`/model`, `/model
    list`, `/model 3`), सत्र ओवरराइड हटाने के लिए `/model default`, और
    एंडपॉइंट/API-मोड विवरण के लिए `/model status` शामिल हैं।

    `@profile` से प्रति सत्र कोई विशिष्ट ऑथ प्रोफ़ाइल अनिवार्य करें:

    ```text
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    `@profile` से सेट प्रोफ़ाइल को अनपिन करने के लिए, प्रत्यय के बिना
    `/model` फिर से चलाएँ (जैसे `/model anthropic/claude-opus-4-6`), या
    `/model` से डिफ़ॉल्ट चुनें। सक्रिय ऑथ प्रोफ़ाइल की पुष्टि के लिए `/model status` उपयोग करें।

  </Accordion>

  <Accordion title="यदि दो प्रदाता एक ही मॉडल आईडी उपलब्ध कराते हैं, तो /model किसे उपयोग करता है?">
    `/model provider/model` ठीक उसी प्रदाता रूट को चुनता है। उदाहरण के लिए,
    मॉडल आईडी समान होने के बावजूद `qianfan/deepseek-v4-flash` और `deepseek/deepseek-v4-flash` अलग
    संदर्भ हैं — केवल आईडी मिलान होने पर OpenClaw चुपचाप प्रदाता नहीं बदलता।

    उपयोगकर्ता द्वारा चयनित `/model` संदर्भ फ़ॉलबैक के लिए सख्त होता है: यदि वह
    प्रदाता/मॉडल अनुपलब्ध हो जाता है, तो उत्तर `agents.defaults.model.fallbacks` पर
    वापस जाने के बजाय स्पष्ट रूप से विफल होता है। कॉन्फ़िगर की गई फ़ॉलबैक
    शृंखलाएँ अभी भी कॉन्फ़िगर किए गए डिफ़ॉल्ट, Cron जॉब के प्राथमिक मॉडल और
    स्वचालित रूप से चयनित फ़ॉलबैक स्थिति पर लागू होती हैं। जब बिना सत्र ओवरराइड वाले रन को
    फ़ॉलबैक उपयोग करने की अनुमति होती है, तो OpenClaw पहले अनुरोधित प्रदाता/मॉडल, फिर
    कॉन्फ़िगर किए गए फ़ॉलबैक और उसके बाद कॉन्फ़िगर किया गया प्राथमिक मॉडल आज़माता है — इसलिए एक जैसे केवल
    मॉडल आईडी कभी भी सीधे डिफ़ॉल्ट प्रदाता पर वापस नहीं जाते।

    [मॉडल](/hi/concepts/models) और [मॉडल फ़ेलओवर](/hi/concepts/model-failover) देखें।

  </Accordion>

  <Accordion title="क्या दैनिक कार्यों के लिए GPT 5.5 और कोडिंग के लिए Codex 5.5 उपयोग किए जा सकते हैं?">
    हाँ — मॉडल का चयन और रनटाइम का चयन अलग-अलग हैं:

    - **मूल Codex कोडिंग एजेंट:** `agents.defaults.model.primary` को
      `openai/gpt-5.5` पर सेट करें। ChatGPT/Codex सदस्यता ऑथ के लिए `openclaw models auth login --provider
      openai` से साइन इन करें।
    - **एजेंट लूप के बाहर सीधे OpenAI API कार्य:** इमेज, एम्बेडिंग,
      स्पीच, रीयलटाइम और अन्य गैर-एजेंट OpenAI API सतहों के लिए
      `OPENAI_API_KEY` कॉन्फ़िगर करें।
    - **OpenAI एजेंट API-कुंजी ऑथ:** क्रमबद्ध
      `openai` API-कुंजी प्रोफ़ाइल के साथ `/model openai/gpt-5.5`।
    - **उप-एजेंट:** कोडिंग कार्यों को अपने
      `openai/gpt-5.5` मॉडल वाले Codex-केंद्रित एजेंट पर रूट करें।

    [मॉडल](/hi/concepts/models) और [स्लैश कमांड](/hi/tools/slash-commands) देखें।

  </Accordion>

  <Accordion title="GPT 5.5 के लिए फ़ास्ट मोड कैसे कॉन्फ़िगर करें?">
    - **प्रति सत्र:** `openai/gpt-5.5` का उपयोग करते समय `/fast on` भेजें।
    - **प्रति मॉडल डिफ़ॉल्ट:** `agents.defaults.models["openai/gpt-5.5"].params.fastMode` को
      `true` पर सेट करें।
    - **स्वचालित कटऑफ़:** `/fast auto` या `params.fastMode: "auto"` नए
      मॉडल कॉल को कटऑफ़ तक तेज़ी से चलाता है, फिर बाद के पुनः प्रयास, फ़ॉलबैक,
      टूल-परिणाम या निरंतरता कॉल को फ़ास्ट मोड के बिना चलाता है। कटऑफ़ का डिफ़ॉल्ट
      60 सेकंड है; मॉडल पर `params.fastAutoOnSeconds` से इसे ओवरराइड करें।

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: {
                fastMode: "auto",
                fastAutoOnSeconds: 30,
              },
            },
          },
        },
      },
    }
    ```

    फ़ास्ट मोड मूल OpenAI Responses अनुरोधों पर `service_tier = "priority"` से मैप होता है;
    मौजूदा `service_tier` मान संरक्षित रहते हैं और फ़ास्ट मोड
    `reasoning` या `text.verbosity` को फिर से नहीं लिखता। सत्र `/fast` ओवरराइड
    कॉन्फ़िग डिफ़ॉल्ट पर प्राथमिकता लेते हैं।

    [सोच और फ़ास्ट मोड](/hi/tools/thinking) तथा [OpenAI](/hi/providers/openai) प्रदाता
    पृष्ठ पर उन्नत कॉन्फ़िगरेशन के अंतर्गत फ़ास्ट मोड अनुभाग देखें।

  </Accordion>

  <Accordion title='मुझे "Model ... is not allowed" क्यों दिखाई देता है और फिर कोई उत्तर क्यों नहीं मिलता?'>
    यदि `agents.defaults.modelPolicy.allow` खाली नहीं है, तो यह
    `/model`, सत्र ओवरराइड और `--model` के लिए **अनुमतिसूची** बन जाता है। उस सूची के बाहर का मॉडल चुनने पर
    सामान्य उत्तर के बजाय यह लौटता है:

    ```text
    मॉडल ओवरराइड "provider/model" को agents.defaults.modelPolicy.allow द्वारा अनुमति नहीं है।
    ```

    समाधान: सटीक मॉडल या `"provider/*"` जैसा प्रदाता वाइल्डकार्ड
    नामित `modelPolicy.allow` सूची में जोड़ें, उस सूची को हटाएँ/खाली करें, या
    `/model list` से कोई मॉडल चुनें। यदि कमांड में
    `--runtime codex` भी शामिल था, तो पहले अनुमतिसूची अपडेट करें, फिर वही
    `/model provider/model --runtime codex` कमांड दोबारा आज़माएँ।

  </Accordion>

  <Accordion title='मुझे "Unknown model: minimax/MiniMax-M3" क्यों दिखाई देता है?'>
    यदि आप OpenClaw की किसी पुरानी रिलीज़ पर हैं, तो पहले अपग्रेड करें (या स्रोत से
    `main` चलाएँ) और Gateway पुनः आरंभ करें — हो सकता है `MiniMax-M3` अभी आपकी
    इंस्टॉल की गई रिलीज़ के कैटलॉग में न हो। अन्यथा MiniMax प्रदाता
    कॉन्फ़िगर नहीं है (कोई प्रदाता प्रविष्टि या ऑथ प्रोफ़ाइल नहीं मिली), इसलिए मॉडल
    रिज़ॉल्व नहीं हो सकता। समाधान की पूरी जाँचसूची, प्रदाता/मॉडल आईडी तालिका और
    कॉन्फ़िग-ब्लॉक उदाहरण के लिए [MiniMax](/hi/providers/minimax) प्रदाता पृष्ठ का
    समस्या निवारण अनुभाग देखें।

  </Accordion>

  <Accordion title="क्या MiniMax को डिफ़ॉल्ट और जटिल कार्यों के लिए OpenAI उपयोग किया जा सकता है?">
    हाँ। MiniMax को डिफ़ॉल्ट बनाएँ और प्रत्येक सत्र के अनुसार मॉडल बदलें — फ़ॉलबैक
    त्रुटियों के लिए हैं, "कठिन कार्यों" के लिए नहीं, इसलिए `/model` या अलग एजेंट उपयोग करें।

    **विकल्प A: प्रति सत्र स्विच करें**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "minimax" },
            "openai/gpt-5.5": { alias: "gpt" },
          },
        },
      },
    }
    ```

    फिर `/model gpt`।

    **विकल्प B: अलग एजेंट** — एजेंट A का डिफ़ॉल्ट MiniMax और एजेंट B का
    डिफ़ॉल्ट OpenAI हो; एजेंट के अनुसार रूट करें या स्विच करने के लिए `/agent` उपयोग करें।

    दस्तावेज़: [मॉडल](/hi/concepts/models), [बहु-एजेंट रूटिंग](/hi/concepts/multi-agent),
    [MiniMax](/hi/providers/minimax), [OpenAI](/hi/providers/openai)।

  </Accordion>

  <Accordion title="क्या opus / sonnet / gpt अंतर्निहित शॉर्टकट हैं?">
    हाँ — अंतर्निहित संक्षिप्त नाम, जो केवल तब लागू होते हैं जब लक्ष्य मॉडल
    `agents.defaults.models` में मौजूद हो:

    | उपनाम | इससे रिज़ॉल्व होता है |
    | --- | --- |
    | `opus` | `anthropic/claude-opus-5` |
    | `sonnet` | `anthropic/claude-sonnet-5` |
    | `gpt` | `openai/gpt-5.4` |
    | `gpt-mini` | `openai/gpt-5.4-mini` |
    | `gpt-nano` | `openai/gpt-5.4-nano` |
    | `gemini` | `google/gemini-3.1-pro-preview` |
    | `gemini-flash` | `google/gemini-3-flash-preview` |
    | `gemini-flash-lite` | `google/gemini-3.1-flash-lite` |

    समान नाम वाला आपका अपना उपनाम अंतर्निहित उपनाम को ओवरराइड करता है।

  </Accordion>

  <Accordion title="मॉडल शॉर्टकट (उपनाम) कैसे परिभाषित/ओवरराइड करें?">
    उपनाम `agents.defaults.models.<modelId>.alias` पर स्थित होते हैं:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
          },
        },
      },
    }
    ```

    फिर `/model sonnet` (या समर्थित होने पर `/<alias>`) उस
    मॉडल आईडी पर रिज़ॉल्व होता है।

  </Accordion>

  <Accordion title="OpenRouter या Z.AI जैसे अन्य प्रदाताओं से मॉडल कैसे जोड़ें?">
    OpenRouter (प्रति-टोकन भुगतान; अनेक मॉडल):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI (GLM मॉडल):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5.1" },
          models: { "zai/glm-5.1": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    संदर्भित प्रदाता/मॉडल के लिए प्रदाता कुंजी न होने पर रनटाइम
    ऑथ त्रुटि उत्पन्न होती है (जैसे `No API key found for provider "zai"`)।

    **नया एजेंट जोड़ने के बाद प्रदाता के लिए कोई API कुंजी नहीं मिली**

    नए एजेंट का ऑथ स्टोर खाली होता है — ऑथ प्रत्येक एजेंट के अनुसार होता है और यहाँ संग्रहीत है:

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    सुधार: `openclaw agents add <id>` चलाएँ और विज़ार्ड में प्रमाणीकरण कॉन्फ़िगर करें, या
    मुख्य एजेंट के स्टोर से केवल पोर्टेबल स्थिर `api_key`/`token` प्रोफ़ाइल
    कॉपी करें। OAuth के लिए, जब नए एजेंट को अपने खाते की आवश्यकता हो, तो उससे
    साइन इन करें। `agentDir` के पूर्ण पुनः उपयोग और क्रेडेंशियल-साझाकरण
    नियमों के लिए [मल्टी-एजेंट रूटिंग](/hi/concepts/multi-agent) देखें — एजेंटों के बीच
    `agentDir` का कभी पुनः उपयोग न करें।

  </Accordion>
</AccordionGroup>

## मॉडल फ़ेलओवर और "सभी मॉडल विफल रहे"

<AccordionGroup>
  <Accordion title="फ़ेलओवर कैसे काम करता है?">
    दो चरण:

    1. एक ही प्रदाता के भीतर **प्रमाणीकरण प्रोफ़ाइल रोटेशन**।
    2. `agents.defaults.model.fallbacks` में अगले मॉडल पर **मॉडल फ़ॉलबैक**।

    विफल प्रोफ़ाइल पर कूलडाउन लागू होते हैं (एक्सपोनेंशियल बैकऑफ़), इसलिए प्रदाता
    के रेट-लिमिट होने या अस्थायी रूप से विफल होने पर भी OpenClaw
    उत्तर देता रहता है।

    रेट-लिमिट बकेट में केवल साधारण `429` से अधिक शामिल है: `Too many concurrent
    requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai
    ... quota limit exceeded`, `resource exhausted`, और आवधिक
    उपयोग-विंडो सीमाएँ (`weekly/monthly limit reached`) सभी को
    फ़ेलओवर योग्य रेट लिमिट माना जाता है।

    बिलिंग प्रतिक्रियाएँ हमेशा `402` नहीं होतीं, और कुछ `402` अस्थायी/रेट-लिमिट
    बकेट में ही रहते हैं, बिलिंग लेन में नहीं। `401`/`403` पर स्पष्ट
    बिलिंग टेक्स्ट फिर भी बिलिंग की ओर रूट कर सकता है; प्रदाता-विशिष्ट
    टेक्स्ट मिलानकर्ता (जैसे OpenRouter `Key limit exceeded`) अपने
    प्रदाता तक ही सीमित रहते हैं। ऐसा `402` जो पुनः प्रयास योग्य उपयोग-विंडो या
    संगठन/वर्कस्पेस खर्च सीमा (`daily limit reached, resets tomorrow`,
    `organization spending limit exceeded`) जैसा दिखाई देता है, उसे लंबी
    बिलिंग निष्क्रियता के बजाय `rate_limit` माना जाता है।

    संदर्भ-ओवरफ़्लो त्रुटियाँ फ़ॉलबैक पथ से पूरी तरह बाहर रहती हैं — `request_too_large`, `input exceeds the maximum number of tokens`,
    `input token count exceeds the maximum number of input tokens`, `input is
    too long for the model`, या `ollama error: context length exceeded` जैसे सिग्नेचर
    मॉडल फ़ॉलबैक को आगे बढ़ाने के बजाय
    Compaction/पुनः प्रयास में जाते हैं।

    सामान्य सर्वर-त्रुटि टेक्स्ट का दायरा "जिसमें भी unknown/error हो"
    उससे संकरा है। प्रदाता-सीमित अस्थायी स्वरूप जिन्हें फ़ेलओवर
    संकेत माना जाता है: Anthropic का केवल `An unknown error occurred`, OpenRouter का केवल
    `Provider returned error`, `Unhandled stop reason:
    error` जैसी स्टॉप-रीज़न त्रुटियाँ, अस्थायी सर्वर टेक्स्ट (`internal
    server error`, `unknown error, 520`, `upstream error`, `backend error`) वाले JSON `api_error` पेलोड,
    और प्रदाता संदर्भ मेल खाने पर `ModelNotReadyException` जैसी
    प्रदाता-व्यस्त त्रुटियाँ। `LLM request failed
    with an unknown error.` जैसा सामान्य आंतरिक फ़ॉलबैक टेक्स्ट सतर्क रहता है और अपने आप
    फ़ॉलबैक ट्रिगर नहीं करता।

  </Accordion>

  <Accordion title='"प्रोफ़ाइल anthropic:default के लिए कोई क्रेडेंशियल नहीं मिला" का क्या अर्थ है?'>
    प्रमाणीकरण प्रोफ़ाइल आईडी `anthropic:default` के लिए अपेक्षित
    प्रमाणीकरण स्टोर में कोई क्रेडेंशियल नहीं है।

    **सुधार जाँच-सूची:**

    - पुष्टि करें कि प्रोफ़ाइल कहाँ रहती हैं — वर्तमान:
      `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`; पुराना:
      `~/.openclaw/agent/*` (`openclaw doctor` द्वारा माइग्रेट किया गया)।
    - पुष्टि करें कि Gateway आपका एनवायरनमेंट वेरिएबल लोड करता है। केवल
      आपके शेल में सेट `ANTHROPIC_API_KEY`, systemd/launchd के माध्यम से चलने वाले Gateway तक नहीं पहुँचेगा — इसे
      `~/.openclaw/.env` में रखें या `env.shellEnv` सक्षम करें।
    - पुष्टि करें कि आप सही एजेंट को संपादित कर रहे हैं — मल्टी-एजेंट सेटअप में
      कई `auth-profiles.json` फ़ाइलें होती हैं।
    - कॉन्फ़िगर किए गए मॉडल और प्रदाता की प्रमाणीकरण स्थिति देखने के लिए
      `openclaw models status` चलाएँ।

    **"प्रोफ़ाइल anthropic के लिए कोई क्रेडेंशियल नहीं मिला" (ईमेल प्रत्यय के बिना) के लिए:**

    रन ऐसी Anthropic प्रोफ़ाइल पर पिन है जिसे Gateway नहीं ढूँढ़ पा रहा।

    - Claude CLI का उपयोग करें: Gateway होस्ट पर `openclaw models auth login --provider anthropic
      --method cli --set-default` चलाएँ।
    - इसके बजाय API कुंजी को प्राथमिकता दें: Gateway होस्ट पर
      `~/.openclaw/.env` में `ANTHROPIC_API_KEY` रखें, फिर गुम प्रोफ़ाइल को बाध्य करने वाला
      कोई भी पिन किया हुआ क्रम हटाएँ:

      ```bash
      openclaw models auth order clear --provider anthropic
      ```

    - रिमोट मोड: प्रमाणीकरण प्रोफ़ाइल आपके लैपटॉप पर नहीं, Gateway मशीन पर रहती हैं —
      पुष्टि करें कि आप वहीं कमांड चला रहे हैं।

  </Accordion>

  <Accordion title="इसने Google Gemini को भी क्यों आज़माया और विफल हुआ?">
    यदि आपके मॉडल कॉन्फ़िगरेशन में Google Gemini फ़ॉलबैक के रूप में शामिल है (या आपने
    Gemini शॉर्टहैंड पर स्विच किया है), तो OpenClaw फ़ॉलबैक के दौरान उसे आज़माता है। Google
    क्रेडेंशियल कॉन्फ़िगर न होने पर `No API key found for provider
    "google"` मिलता है। सुधार: Google प्रमाणीकरण जोड़ें, या
    `agents.defaults.model.fallbacks`/उपनामों से Google मॉडल हटाएँ।

    **LLM अनुरोध अस्वीकृत: थिंकिंग सिग्नेचर आवश्यक है (Google Antigravity)**

    कारण: सत्र इतिहास में बिना सिग्नेचर वाले थिंकिंग ब्लॉक हैं (अक्सर
    निरस्त/आंशिक स्ट्रीम से); Google Antigravity को थिंकिंग ब्लॉक पर सिग्नेचर
    आवश्यक होते हैं। OpenClaw, Google Antigravity Claude के लिए बिना सिग्नेचर वाले थिंकिंग ब्लॉक
    हटा देता है; यदि यह फिर भी दिखाई दे, तो नया सत्र शुरू करें या उस एजेंट के लिए
    `/thinking off` सेट करें।

  </Accordion>
</AccordionGroup>

## प्रमाणीकरण प्रोफ़ाइल: वे क्या हैं और उन्हें कैसे प्रबंधित करें

संबंधित: [/concepts/oauth](/hi/concepts/oauth) (OAuth प्रवाह, टोकन भंडारण, एकाधिक-खाता पैटर्न)

<AccordionGroup>
  <Accordion title="प्रमाणीकरण प्रोफ़ाइल क्या है?">
    किसी प्रदाता से जुड़ा नामित क्रेडेंशियल रिकॉर्ड (OAuth या API कुंजी), जिसे
    यहाँ संग्रहीत किया जाता है:

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    सीक्रेट प्रकट किए बिना सहेजी गई प्रोफ़ाइल की जाँच करें: `openclaw models auth
    list` (वैकल्पिक रूप से `--provider <id>` या `--json`)। [मॉडल CLI](/hi/cli/models#auth-profiles) देखें।

  </Accordion>

  <Accordion title="सामान्य प्रोफ़ाइल आईडी क्या होती हैं?">
    प्रदाता-उपसर्ग वाली: `anthropic:default` (जब ईमेल पहचान मौजूद न हो तब सामान्य),
    OAuth पहचानों के लिए `anthropic:<email>`, या आपकी चुनी हुई कस्टम आईडी
    (जैसे `anthropic:work`)।

  </Accordion>

  <Accordion title="क्या मैं नियंत्रित कर सकता हूँ कि कौन-सी प्रमाणीकरण प्रोफ़ाइल पहले आज़माई जाए?">
    हाँ। `auth.order.<provider>` कॉन्फ़िगरेशन प्रत्येक प्रदाता के लिए रोटेशन क्रम सेट करता है
    (केवल मेटाडेटा — कोई सीक्रेट संग्रहीत नहीं होता)।

    OpenClaw किसी प्रोफ़ाइल को छोटे **कूलडाउन** (रेट लिमिट,
    टाइमआउट, प्रमाणीकरण विफलताएँ) या लंबी **निष्क्रिय** स्थिति
    (बिलिंग/अपर्याप्त क्रेडिट) में छोड़ सकता है। `openclaw models status
    --json` से जाँचें और `auth.unusableProfiles` देखें। रेट-लिमिट कूलडाउन
    मॉडल-सीमित हो सकते हैं — एक मॉडल के लिए कूलडाउन में गई प्रोफ़ाइल उसी प्रदाता के
    किसी अन्य मॉडल को सेवा देना जारी रख सकती है; बिलिंग/निष्क्रिय विंडो पूरी
    प्रोफ़ाइल को अवरुद्ध करती हैं।

    प्रति-एजेंट क्रम ओवरराइड सेट करें (उस एजेंट के `auth-state.json` में संग्रहीत):

    ```bash
    # कॉन्फ़िगर किए गए डिफ़ॉल्ट एजेंट का उपयोग करता है (--agent छोड़ें)
    openclaw models auth order get --provider anthropic

    # रोटेशन को एक प्रोफ़ाइल तक सीमित करें
    openclaw models auth order set --provider anthropic anthropic:default

    # या स्पष्ट क्रम सेट करें (प्रदाता के भीतर फ़ॉलबैक)
    openclaw models auth order set --provider anthropic anthropic:work anthropic:default

    # ओवरराइड हटाएँ (कॉन्फ़िगरेशन auth.order / राउंड-रॉबिन पर वापस जाएँ)
    openclaw models auth order clear --provider anthropic

    # किसी विशिष्ट एजेंट को लक्षित करें
    openclaw models auth order set --provider anthropic --agent main anthropic:default
    ```

    वास्तव में क्या आज़माया जाएगा, इसकी पुष्टि करें: `openclaw models status --probe`। स्पष्ट क्रम से
    बाहर रखी गई संग्रहीत प्रोफ़ाइल को चुपचाप आज़माने के बजाय
    `excluded_by_auth_order` रिपोर्ट किया जाता है।

  </Accordion>

  <Accordion title="OAuth और API कुंजी में क्या अंतर है?">
    - जहाँ प्रदाता इसका समर्थन करता है, वहाँ **OAuth / CLI लॉगिन** अक्सर सदस्यता पहुँच का
      उपयोग करता है। Anthropic के लिए, OpenClaw का Claude CLI बैकएंड
      Claude Code `claude -p` का उपयोग करता है, जिसे Anthropic वर्तमान में
      सदस्यता उपयोग सीमाओं से खपत करने वाला Agent SDK/प्रोग्रामेटिक उपयोग मानता है —
      वर्तमान बिलिंग-विराम स्थिति और स्रोत लिंक के लिए [Anthropic](/hi/providers/anthropic) देखें।
    - **API कुंजियाँ** प्रति-टोकन भुगतान वाली बिलिंग का उपयोग करती हैं।

    विज़ार्ड Anthropic Claude CLI, OpenAI Codex OAuth और API
    कुंजियों का समर्थन करता है।

  </Accordion>
</AccordionGroup>

## संबंधित

- [अक्सर पूछे जाने वाले प्रश्न](/hi/help/faq) — मुख्य FAQ
- [अक्सर पूछे जाने वाले प्रश्न — त्वरित शुरुआत और पहली बार का सेटअप](/hi/help/faq-first-run)
- [मॉडल चयन](/hi/concepts/model-providers)
- [मॉडल फ़ेलओवर](/hi/concepts/model-failover)
