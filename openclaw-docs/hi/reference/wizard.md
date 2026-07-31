---
read_when:
    - किसी विशिष्ट ऑनबोर्डिंग चरण या फ़्लैग को खोजना
    - गैर-इंटरैक्टिव मोड के साथ ऑनबोर्डिंग को स्वचालित करना
    - ऑनबोर्डिंग व्यवहार की डीबगिंग
sidebarTitle: Onboarding Reference
summary: 'CLI ऑनबोर्डिंग का संपूर्ण संदर्भ: प्रत्येक चरण, फ़्लैग और कॉन्फ़िगरेशन फ़ील्ड'
title: ऑनबोर्डिंग संदर्भ
x-i18n:
    generated_at: "2026-07-27T18:59:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e5e7e42fa3fc1a6d85ad422d0d28dfeda233c89a4d7e97eee4fb974831816372
    source_path: reference/wizard.md
    workflow: 16
---

यह `openclaw onboard` का पूर्ण संदर्भ है।
उच्च-स्तरीय अवलोकन के लिए, [ऑनबोर्डिंग (CLI)](/hi/start/wizard) देखें। चरण-दर-चरण
व्यवहार और आउटपुट के लिए, [CLI सेटअप संदर्भ](/hi/start/wizard-cli-reference) देखें।

## प्रवाह का विवरण (स्थानीय मोड)

<Steps>
  <Step title="रीसेट (वैकल्पिक)">
    - `--reset` सेटअप चलने से पहले स्थिति रीसेट करता है; इसके बिना, ऑनबोर्डिंग को दोबारा चलाने पर
      मौजूदा कॉन्फ़िगरेशन बना रहता है और डिफ़ॉल्ट के रूप में फिर से उपयोग होता है।
    - `--reset-scope` नियंत्रित करता है कि `--reset` क्या हटाता है: `config` (केवल कॉन्फ़िगरेशन फ़ाइल
      ), `config+creds+sessions` (डिफ़ॉल्ट), या `full` (वर्कस्पेस भी
      हटाता है)।
    - यदि कॉन्फ़िगरेशन फ़ाइल अमान्य है, तो ऑनबोर्डिंग रुक जाती है और आपको पहले
      `openclaw doctor` चलाने, फिर सेटअप दोबारा चलाने के लिए कहती है।
    - रीसेट स्थिति को Trash में ले जाता है (कभी सीधे नहीं मिटाता)।

  </Step>
  <Step title="जोखिम की स्वीकृति">
    - पहली बार चलाने पर (या `wizard.securityAcknowledgedAt` सेट होने से पहले किसी भी बार)
      आपसे पुष्टि करने को कहा जाता है कि आप समझते हैं कि एजेंट शक्तिशाली होते हैं और उन्हें संपूर्ण
      सिस्टम एक्सेस देना जोखिमपूर्ण है।
    - `--non-interactive` में `--accept-risk` स्पष्ट रूप से देना आवश्यक है; इसके बिना,
      ऑनबोर्डिंग संकेत देने के बजाय त्रुटि के साथ बंद हो जाती है।
    - इंटरैक्टिव रन में फ़्लैग के बजाय पुष्टि संकेत मिलता है; अस्वीकार करने पर
      सेटअप रद्द हो जाता है।

  </Step>
  <Step title="मॉडल/प्रमाणीकरण">
    - **Anthropic API कुंजी**: उपलब्ध होने पर `ANTHROPIC_API_KEY` का उपयोग करता है या कुंजी माँगता है, फिर उसे डेमन के उपयोग हेतु सहेजता है।
    - **Anthropic Claude CLI**: जब Claude CLI साइन-इन पहले से मौजूद हो, तो यह पसंदीदा स्थानीय पथ है; OpenClaw विकल्प के रूप में Anthropic सेटअप-टोकन प्रमाणीकरण का समर्थन भी करता है।
    - **OpenAI Code (Codex) सदस्यता (OAuth)**: ब्राउज़र प्रवाह; `code#state` चिपकाएँ।
      - प्राथमिक मॉडल के बिना नए सेटअप पर, Codex रनटाइम के माध्यम से `agents.defaults.model` को `openai/gpt-5.6-sol` पर सेट करता है।
    - **OpenAI Code (Codex) सदस्यता (डिवाइस पेयरिंग)**: अल्पकालिक डिवाइस कोड वाला ब्राउज़र पेयरिंग प्रवाह।
      - प्राथमिक मॉडल के बिना नए सेटअप पर, Codex रनटाइम के माध्यम से `agents.defaults.model` को `openai/gpt-5.6-sol` पर सेट करता है।
    - **OpenAI API कुंजी**: उपलब्ध होने पर `OPENAI_API_KEY` का उपयोग करता है या कुंजी माँगता है, फिर उसे प्रमाणीकरण प्रोफ़ाइल में संग्रहीत करता है।
      - प्राथमिक मॉडल के बिना नए सेटअप पर, `agents.defaults.model` को `openai/gpt-5.6` पर सेट करता है; केवल प्रत्यक्ष-API मॉडल आईडी Sol टियर में रिज़ॉल्व होती है।
    - OpenAI को जोड़ने या फिर से प्रमाणित करने पर मौजूदा स्पष्ट प्राथमिक मॉडल संरक्षित रहता है, जिसमें `openai/gpt-5.5` भी शामिल है। यदि खाते में GPT-5.6 उपलब्ध नहीं है, तो `openai/gpt-5.5` स्पष्ट रूप से चुनें; OpenClaw मॉडल को चुपचाप डाउनग्रेड नहीं करता।
    - **xAI OAuth**: localhost कॉलबैक की आवश्यकता के बिना डिवाइस-कोड ब्राउज़र साइन-इन, इसलिए यह SSH/Docker/VPS पर भी काम करता है (`--auth-choice xai-oauth`)।
    - **xAI API कुंजी**: `XAI_API_KEY` माँगता है (`--auth-choice xai-api-key`)।
    - `--auth-choice xai-device-code` अभी भी उसी xAI OAuth डिवाइस-कोड प्रवाह के लिए केवल मैन्युअल संगतता उपनाम के रूप में काम करता है; नई स्क्रिप्ट के लिए `xai-oauth` का उपयोग करें।
    - **OpenCode**: `OPENCODE_API_KEY` (या `OPENCODE_ZEN_API_KEY`, इसे https://opencode.ai/auth पर प्राप्त करें) माँगता है और आपको Zen या Go कैटलॉग चुनने देता है।
    - **Ollama**: पहले **क्लाउड + स्थानीय**, **केवल क्लाउड**, या **केवल स्थानीय** का विकल्प देता है। `Cloud only`, `OLLAMA_API_KEY` माँगता है और `https://ollama.com` का उपयोग करता है; होस्ट-समर्थित मोड Ollama आधार URL (डिफ़ॉल्ट `http://127.0.0.1:11434`) माँगते हैं, उपलब्ध मॉडल खोजते हैं और आवश्यकता होने पर चुने गए स्थानीय मॉडल को स्वतः पुल करते हैं; `Cloud + Local` यह भी जाँचता है कि क्लाउड एक्सेस के लिए उस Ollama होस्ट में साइन-इन किया गया है या नहीं।
    - अधिक विवरण: [Ollama](/hi/providers/ollama)
    - **API कुंजी**: आपके लिए कुंजी संग्रहीत करता है।
    - **Vercel AI Gateway (बहु-मॉडल प्रॉक्सी)**: `AI_GATEWAY_API_KEY` माँगता है।
    - अधिक विवरण: [Vercel AI Gateway](/hi/providers/vercel-ai-gateway)
    - **Cloudflare AI Gateway**: Account ID, Gateway ID और `CLOUDFLARE_AI_GATEWAY_API_KEY` माँगता है।
    - अधिक विवरण: [Cloudflare AI Gateway](/hi/providers/cloudflare-ai-gateway)
    - **MiniMax**: कॉन्फ़िगरेशन स्वतः लिखा जाता है; होस्टेड डिफ़ॉल्ट `MiniMax-M3` है।
      API-कुंजी सेटअप `minimax/...` का उपयोग करता है और OAuth सेटअप
      `minimax-portal/...` का उपयोग करता है।
    - अधिक विवरण: [MiniMax](/hi/providers/minimax)
    - **StepFun**: चीन या वैश्विक एंडपॉइंट पर StepFun मानक या Step Plan के लिए कॉन्फ़िगरेशन स्वतः लिखा जाता है।
    - वर्तमान में मानक का डिफ़ॉल्ट `step-3.5-flash` है; Step Plan में `step-3.5-flash-2603` भी शामिल है।
    - अधिक विवरण: [StepFun](/hi/providers/stepfun)
    - **Synthetic (Anthropic-संगत)**: `SYNTHETIC_API_KEY` माँगता है।
    - अधिक विवरण: [Synthetic](/hi/providers/synthetic)
    - **Moonshot (Kimi K2)**: कॉन्फ़िगरेशन स्वतः लिखा जाता है।
    - **Kimi Coding**: कॉन्फ़िगरेशन स्वतः लिखा जाता है।
    - अधिक विवरण: [Moonshot AI (Kimi + Kimi Coding)](/hi/providers/moonshot)
    - **कस्टम प्रदाता**: OpenAI-संगत, OpenAI Responses-संगत या Anthropic-संगत एंडपॉइंट के साथ काम करता है। गैर-इंटरैक्टिव फ़्लैग: `--auth-choice custom-api-key`, `--custom-base-url`, `--custom-model-id`, `--custom-api-key` (वैकल्पिक; `CUSTOM_API_KEY` पर वापस जाता है), `--custom-provider-id` (वैकल्पिक; आधार URL से स्वतः व्युत्पन्न), `--custom-compatibility openai|openai-responses|anthropic` (डिफ़ॉल्ट `openai`), `--custom-image-input` / `--custom-text-input` (अनुमानित विज़न-मॉडल पहचान को ओवरराइड करते हैं)।
    - **छोड़ें**: अभी कोई प्रमाणीकरण कॉन्फ़िगर नहीं किया गया।
    - पहचाने गए विकल्पों में से डिफ़ॉल्ट मॉडल चुनें (या प्रदाता/मॉडल मैन्युअल रूप से दर्ज करें)। सर्वोत्तम गुणवत्ता और कम प्रॉम्प्ट-इंजेक्शन जोखिम के लिए, अपने प्रदाता स्टैक में उपलब्ध नवीनतम पीढ़ी का सबसे शक्तिशाली मॉडल चुनें।
    - ऑनबोर्डिंग मॉडल की जाँच करती है और कॉन्फ़िगर किया गया मॉडल अज्ञात होने या प्रमाणीकरण अनुपस्थित होने पर चेतावनी देती है।
    - API कुंजी संग्रहण मोड का डिफ़ॉल्ट प्लेनटेक्स्ट प्रमाणीकरण-प्रोफ़ाइल मान है। इसके बजाय परिवेश-समर्थित संदर्भ संग्रहीत करने के लिए `--secret-input-mode ref` का उपयोग करें (उदाहरण के लिए `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`); संदर्भित परिवेश चर पहले से सेट होना चाहिए, अन्यथा ऑनबोर्डिंग तुरंत विफल हो जाती है।
    - प्रमाणीकरण प्रोफ़ाइल `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` में रहती हैं (API कुंजियाँ + OAuth)। `~/.openclaw/credentials/oauth.json` केवल लीगेसी आयात के लिए है।
    - अधिक विवरण: [OAuth](/hi/concepts/oauth)
    <Note>
    हेडलेस/सर्वर सुझाव: ब्राउज़र वाली मशीन पर OAuth पूरा करें, फिर
    उस एजेंट की `auth-profiles.json` (उदाहरण के लिए
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`, या मेल खाता
    `$OPENCLAW_STATE_DIR/...` पथ) Gateway होस्ट पर कॉपी करें। `credentials/oauth.json`
    केवल लीगेसी आयात स्रोत है।
    </Note>
  </Step>
  <Step title="वर्कस्पेस">
    - डिफ़ॉल्ट `~/.openclaw/workspace` (कॉन्फ़िगर करने योग्य)।
    - एजेंट बूटस्ट्रैप प्रक्रिया के लिए आवश्यक वर्कस्पेस फ़ाइलें तैयार करता है।
    - पूर्ण वर्कस्पेस लेआउट + बैकअप मार्गदर्शिका: [एजेंट वर्कस्पेस](/hi/concepts/agent-workspace)

  </Step>
  <Step title="Gateway">
    - पोर्ट (डिफ़ॉल्ट **18789**), बाइंड, प्रमाणीकरण मोड, tailscale एक्सपोज़र।
    - प्रमाणीकरण अनुशंसा: लूपबैक के लिए भी **टोकन** रखें, ताकि स्थानीय WS क्लाइंट को प्रमाणित करना आवश्यक हो।
    - टोकन मोड में, इंटरैक्टिव सेटअप ये विकल्प देता है:
      - **प्लेनटेक्स्ट टोकन जनरेट/संग्रहीत करें** (डिफ़ॉल्ट)
      - **SecretRef का उपयोग करें** (ऑप्ट-इन)
      - त्वरित शुरुआत, ऑनबोर्डिंग प्रोब/डैशबोर्ड बूटस्ट्रैप के लिए `env`, `file` और `exec` प्रदाताओं में मौजूदा `gateway.auth.token` SecretRefs का पुनः उपयोग करती है।
      - यदि वह SecretRef कॉन्फ़िगर है लेकिन रिज़ॉल्व नहीं किया जा सकता, तो ऑनबोर्डिंग रनटाइम प्रमाणीकरण को चुपचाप कमजोर करने के बजाय स्पष्ट सुधार संदेश के साथ जल्दी विफल हो जाती है।
    - पासवर्ड मोड में, इंटरैक्टिव सेटअप प्लेनटेक्स्ट या SecretRef संग्रहण का समर्थन भी करता है।
    - गैर-इंटरैक्टिव टोकन SecretRef पथ: `--gateway-token-ref-env <ENV_VAR>`।
      - ऑनबोर्डिंग प्रक्रिया परिवेश में रिक्त न होने वाला परिवेश चर आवश्यक है।
      - `--gateway-token` के साथ संयोजित नहीं किया जा सकता।
    - प्रमाणीकरण केवल तभी अक्षम करें जब आपको प्रत्येक स्थानीय प्रक्रिया पर पूर्ण भरोसा हो।
    - गैर-लूपबैक बाइंड के लिए फिर भी प्रमाणीकरण आवश्यक है।

  </Step>
  <Step title="चैनल">
    - [WhatsApp](/hi/channels/whatsapp): वैकल्पिक QR लॉगिन।
    - [Telegram](/hi/channels/telegram): बॉट टोकन।
    - [Discord](/hi/channels/discord): बॉट टोकन।
    - [Google Chat](/hi/channels/googlechat): सेवा खाता JSON + Webhook ऑडियंस।
    - [Mattermost](/hi/channels/mattermost) (Plugin): बॉट टोकन + आधार URL।
    - [Signal](/hi/channels/signal) (Plugin): वैकल्पिक `signal-cli` इंस्टॉल + खाता कॉन्फ़िगरेशन।
    - [iMessage](/hi/channels/imessage): `imsg` CLI पथ + Messages DB एक्सेस; Gateway के Mac से बाहर चलने पर SSH रैपर का उपयोग करें।
    - Discord, Feishu, Microsoft Teams, QQ Bot, Slack और अन्य चैनल
      ऐसे plugins के रूप में आते हैं जिन्हें ऑनबोर्डिंग आपके लिए इंस्टॉल कर सकती है। पूर्ण कैटलॉग: [चैनल](/hi/channels)।
    - DM सुरक्षा: डिफ़ॉल्ट पेयरिंग है। पहला DM एक कोड भेजता है; `openclaw pairing approve <channel> <code>` के माध्यम से अनुमोदित करें या अनुमत-सूचियों का उपयोग करें।

  </Step>
  <Step title="वेब खोज">
    - Brave, Codex (होस्टेड खोज), DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Parallel, Perplexity, SearXNG या Tavily जैसा कोई समर्थित प्रदाता चुनें (या छोड़ें)।
    - API-समर्थित प्रदाता त्वरित सेटअप के लिए परिवेश चर या मौजूदा कॉन्फ़िगरेशन का उपयोग कर सकते हैं; बिना कुंजी वाले प्रदाता इसके बजाय अपनी प्रदाता-विशिष्ट पूर्वापेक्षाओं का उपयोग करते हैं।
    - `--skip-search` से छोड़ें।
    - बाद में कॉन्फ़िगर करें: `openclaw configure --section web`।

  </Step>
  <Step title="डेमन इंस्टॉल">
    - macOS: LaunchAgent
      - लॉग-इन उपयोगकर्ता सत्र आवश्यक है; हेडलेस के लिए, कस्टम LaunchDaemon का उपयोग करें (साथ में उपलब्ध नहीं)।
    - Linux (और WSL2 के माध्यम से Windows): systemd उपयोगकर्ता यूनिट
      - ऑनबोर्डिंग `loginctl enable-linger <user>` के माध्यम से लिंगरिंग सक्षम करने का प्रयास करती है, ताकि लॉगआउट के बाद Gateway चालू रहे।
      - sudo माँग सकता है (`/var/lib/systemd/linger` लिखता है); पहले यह sudo के बिना प्रयास करता है।
    - नेटिव Windows: पहले Scheduled Task; यदि टास्क बनाने की अनुमति नहीं मिलती, तो OpenClaw प्रति-उपयोगकर्ता Startup-folder लॉगिन आइटम पर वापस जाता है और Gateway तुरंत शुरू करता है।
    - **रनटाइम चयन:** Node आवश्यक है क्योंकि कैनोनिकल रनटाइम स्थिति स्टोर `node:sqlite` का उपयोग करता है। लीगेसी Bun सेवाएँ सुधार के दौरान Node में माइग्रेट की जाती हैं।
    - यदि टोकन प्रमाणीकरण के लिए टोकन आवश्यक है और `gateway.auth.token` को SecretRef द्वारा प्रबंधित किया जाता है, तो डेमन इंस्टॉल उसे सत्यापित करता है लेकिन रिज़ॉल्व किए गए प्लेनटेक्स्ट टोकन मानों को सुपरवाइज़र सेवा परिवेश मेटाडेटा में स्थायी रूप से संग्रहीत नहीं करता।
    - यदि टोकन प्रमाणीकरण के लिए टोकन आवश्यक है और कॉन्फ़िगर किया गया टोकन SecretRef अनरिज़ॉल्व्ड है, तो डेमन इंस्टॉल को कार्रवाई योग्य मार्गदर्शन के साथ अवरुद्ध कर दिया जाता है।
    - यदि `gateway.auth.token` और `gateway.auth.password` दोनों कॉन्फ़िगर हैं तथा `gateway.auth.mode` सेट नहीं है, तो मोड स्पष्ट रूप से सेट किए जाने तक डेमन इंस्टॉल अवरुद्ध रहता है।

  </Step>
  <Step title="स्वास्थ्य जाँच">
    - Gateway शुरू करता है (यदि आवश्यक हो) और `openclaw health` चलाता है।
    - सुझाव: `openclaw status --deep` स्थिति आउटपुट में लाइव Gateway स्वास्थ्य प्रोब जोड़ता है, जिसमें समर्थित होने पर चैनल प्रोब भी शामिल होते हैं (पहुँच योग्य Gateway आवश्यक है)।

  </Step>
  <Step title="Skills (अनुशंसित)">
    - उपलब्ध Skills पढ़ता है और आवश्यकताओं की जाँच करता है।
    - आपको Node प्रबंधक चुनने देता है: **npm / pnpm / bun**।
    - विश्वसनीय बंडल किए गए Skills के लिए वैकल्पिक निर्भरताएँ स्वतः इंस्टॉल करता है (कुछ macOS पर Homebrew का उपयोग करते हैं)।
    - उन Skills को छोड़ देता है जिनकी Homebrew, uv या Go इंस्टॉलर पूर्वापेक्षा उपलब्ध नहीं है, उन्हें मैन्युअल सेटअप मार्गदर्शन के साथ समूहित करता है और पूर्वापेक्षा इंस्टॉल होने के बाद आपको `openclaw doctor` की ओर निर्देशित करता है।

  </Step>
  <Step title="समापन">
    - सारांश + अगले चरण, जिनमें Terminal, Browser या बाद के लिए **आप अपने एजेंट को कैसे आरंभ करना चाहते हैं?** संकेत शामिल है।

  </Step>
</Steps>

<Note>
यदि कोई GUI नहीं मिलता है, तो ऑनबोर्डिंग ब्राउज़र खोलने के बजाय Control UI के लिए SSH पोर्ट-फ़ॉरवर्ड निर्देश प्रिंट करता है।
यदि Control UI एसेट अनुपलब्ध हैं, तो ऑनबोर्डिंग उन्हें बिल्ड करने का प्रयास करता है; फ़ॉलबैक `pnpm ui:build` है (UI डिपेंडेंसी अपने-आप इंस्टॉल करता है)।
</Note>

## गैर-इंटरैक्टिव मोड

ऑनबोर्डिंग को स्वचालित करने या स्क्रिप्ट में चलाने के लिए `--non-interactive --accept-risk` का उपयोग करें (यह
फ़्लैग आवश्यक जोखिम स्वीकृति है; इसके बिना ऑनबोर्डिंग त्रुटि के साथ
बंद हो जाता है):

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

मशीन-पठनीय सारांश के लिए `--json` जोड़ें।

गैर-इंटरैक्टिव मोड में Gateway टोकन SecretRef:

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN
```

`--gateway-token` और `--gateway-token-ref-env` परस्पर अनन्य हैं।

<Note>
`--json` गैर-इंटरैक्टिव मोड को **नहीं** दर्शाता। स्क्रिप्ट के लिए `--non-interactive --accept-risk` (और `--workspace`) का उपयोग करें।
</Note>

प्रोवाइडर-विशिष्ट कमांड उदाहरण [CLI स्वचालन](/hi/start/wizard-cli-automation#provider-specific-examples) में उपलब्ध हैं।
फ़्लैग के अर्थ और चरणों के क्रम के लिए इस संदर्भ पृष्ठ का उपयोग करें।

### एजेंट जोड़ें (गैर-इंटरैक्टिव)

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.6-sol \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

`main` एक आरक्षित एजेंट आईडी है और इसे `openclaw agents add` के लिए उपयोग नहीं किया जा सकता।

## Gateway विज़ार्ड RPC

Gateway ऑनबोर्डिंग प्रवाह को RPC (`wizard.start`, `wizard.next`, `wizard.cancel`, `wizard.status`) पर उपलब्ध कराता है।
क्लाइंट (macOS ऐप, Control UI) ऑनबोर्डिंग लॉजिक को दोबारा लागू किए बिना चरणों को रेंडर कर सकते हैं।

## Signal सेटअप (signal-cli)

ऑनबोर्डिंग पता लगाता है कि `signal-cli`, `PATH` पर उपलब्ध है या नहीं और अनुपलब्ध होने पर इसे इंस्टॉल करने का विकल्प देता है:

- Linux x86-64: `signal-cli` GitHub रिलीज़ से आधिकारिक नेटिव GraalVM बिल्ड डाउनलोड करके उसे `~/.openclaw/tools/signal-cli/<version>/` में संग्रहीत करता है।
- macOS और अन्य आर्किटेक्चर: इसके बजाय Homebrew के माध्यम से इंस्टॉल करता है।
- नेटिव Windows: अभी समर्थित नहीं है; Linux इंस्टॉलेशन पथ पाने के लिए WSL2 के भीतर ऑनबोर्डिंग चलाएँ।
- दोनों स्थितियों में `kind: "managed-native"` के साथ `channels.signal.transport.cliPath` लिखता है।

## विज़ार्ड क्या लिखता है

`~/.openclaw/openclaw.json` में सामान्य फ़ील्ड:

- `agents.defaults.workspace`
- `--skip-bootstrap` दिए जाने पर `agents.defaults.skipBootstrap`
- `agents.defaults.model` / `models.providers` (यदि Minimax चुना गया हो)
- `tools.profile` (सेट न होने पर स्थानीय ऑनबोर्डिंग का डिफ़ॉल्ट `"coding"` होता है; मौजूदा स्पष्ट मान संरक्षित रखे जाते हैं)
- `gateway.*` (मोड, बाइंड, प्रमाणीकरण, Tailscale)
- `session.dmScope` (ऑनबोर्डिंग स्पष्ट मानों को संरक्षित रखता है और अन्यथा इसे सेट नहीं करता, इसलिए `"main"` डिफ़ॉल्ट सभी चैनलों के प्रत्यक्ष संदेश एजेंट के रोलिंग मुख्य सत्र में रखता है—यह निजी एजेंट का डिफ़ॉल्ट है। साझा या बहु-उपयोगकर्ता इनबॉक्स के लिए `"per-channel-peer"` का उपयोग करें; बहु-उपयोगकर्ता DM ट्रैफ़िक का पता लगने पर `openclaw security audit` पृथक्करण की अनुशंसा करता है। विवरण: [CLI सेटअप संदर्भ](/hi/start/wizard-cli-reference#outputs-and-internals))
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- चैनल प्रॉम्प्ट के दौरान विकल्प चुनने पर चैनल DM अनुमति-सूचियाँ। संभव होने पर Discord, Matrix, Microsoft Teams और Slack नामों को आईडी में बदलते हैं; अन्य चैनल सीधे आईडी लेते हैं (उदाहरण के लिए संख्यात्मक Telegram प्रेषक आईडी या WhatsApp फ़ोन नंबर)।
- `skills.install.nodeManager`
  - `setup --node-manager`, `npm`, `pnpm`, या `bun` स्वीकार करता है।
  - मैन्युअल कॉन्फ़िगरेशन में सीधे `skills.install.nodeManager` सेट करके अब भी `yarn` का उपयोग किया जा सकता है।
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`
- `wizard.securityAcknowledgedAt`

`openclaw agents add`, `agents.entries.*` और वैकल्पिक `bindings` लिखता है।

WhatsApp क्रेडेंशियल `~/.openclaw/credentials/whatsapp/<accountId>/` के अंतर्गत रखे जाते हैं।
सक्रिय सत्र और ट्रांसक्रिप्ट
`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` में संग्रहीत होते हैं।
`~/.openclaw/agents/<agentId>/sessions/` डायरेक्टरी का उपयोग पुराने माइग्रेशन
इनपुट और संग्रह/सहायता आर्टिफ़ैक्ट के लिए किया जाता है।

कुछ चैनल plugins के रूप में उपलब्ध कराए जाते हैं। सेटअप के दौरान किसी एक को चुनने पर, उसे कॉन्फ़िगर करने से पहले ऑनबोर्डिंग
उसे इंस्टॉल करने (npm या स्थानीय पथ से) के लिए कहेगा।

## संबंधित दस्तावेज़

- ऑनबोर्डिंग अवलोकन: [ऑनबोर्डिंग (CLI)](/hi/start/wizard)
- CLI सेटअप संदर्भ: [CLI सेटअप संदर्भ](/hi/start/wizard-cli-reference)
- macOS ऐप ऑनबोर्डिंग: [ऑनबोर्डिंग](/hi/start/onboarding)
- कॉन्फ़िगरेशन संदर्भ: [Gateway कॉन्फ़िगरेशन](/hi/gateway/configuration)
- प्रोवाइडर: [WhatsApp](/hi/channels/whatsapp), [Telegram](/hi/channels/telegram), [Discord](/hi/channels/discord), [Google Chat](/hi/channels/googlechat), [Signal](/hi/channels/signal), [iMessage](/hi/channels/imessage)
- Skills: [Skills](/hi/tools/skills), [Skills कॉन्फ़िगरेशन](/hi/tools/skills-config)
