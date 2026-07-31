---
read_when:
    - मॉडल प्रमाणीकरण या OAuth की समय-सीमा समाप्ति की डीबगिंग
    - प्रमाणीकरण या क्रेडेंशियल संग्रहण का दस्तावेज़ीकरण
summary: 'मॉडल प्रमाणीकरण: OAuth, API कुंजियाँ, Claude CLI का पुनः उपयोग, और Anthropic सेटअप-टोकन'
title: प्रमाणीकरण
x-i18n:
    generated_at: "2026-07-27T19:40:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fd4bf1c73f41d297638811f568c1b11e920eba3bd1527206cbb760df51531f2
    source_path: gateway/authentication.md
    workflow: 16
---

<Note>
यह पृष्ठ **मॉडल प्रदाता** प्रमाणीकरण (API कुंजियाँ, OAuth, Claude CLI का पुनः उपयोग, Anthropic सेटअप-टोकन) को कवर करता है। **Gateway कनेक्शन** प्रमाणीकरण (टोकन, पासवर्ड, trusted-proxy) के लिए, [कॉन्फ़िगरेशन](/hi/gateway/configuration) और [विश्वसनीय प्रॉक्सी प्रमाणीकरण](/hi/gateway/trusted-proxy-auth) देखें।
</Note>

OpenClaw मॉडल प्रदाताओं के लिए OAuth और API कुंजियों का समर्थन करता है। हमेशा चालू रहने वाले Gateway होस्ट के लिए API कुंजी सबसे पूर्वानुमेय विकल्प है; सदस्यता/OAuth प्रवाह भी तब काम करते हैं, जब वे आपके प्रदाता खाते के मॉडल से मेल खाते हों।

- पूरा OAuth प्रवाह और स्टोरेज लेआउट: [/concepts/oauth](/hi/concepts/oauth)
- SecretRef-आधारित प्रमाणीकरण (`env`/`file`/`exec` प्रदाता): [सीक्रेट प्रबंधन](/hi/gateway/secrets)
- `models status --probe` द्वारा उपयोग किए जाने वाले क्रेडेंशियल पात्रता/कारण कोड: [प्रमाणीकरण क्रेडेंशियल अर्थविज्ञान](/hi/auth-credential-semantics)

## अनुशंसित सेटअप: API कुंजी (कोई भी प्रदाता)

1. अपने प्रदाता कंसोल में एक API कुंजी बनाएँ।
2. इसे **Gateway होस्ट** (`openclaw gateway` चलाने वाली मशीन) पर रखें:

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. यदि Gateway systemd/launchd के अंतर्गत चलता है, तो कुंजी को `~/.openclaw/.env` में रखें, ताकि डेमन उसे पढ़ सके:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

4. Gateway प्रक्रिया (या डेमन) को पुनः आरंभ करें, फिर दोबारा जाँचें:

```bash
openclaw models status
openclaw doctor
```

यदि आप स्वयं env vars प्रबंधित नहीं करना चाहते हैं, तो `openclaw onboard` डेमन के उपयोग के लिए API कुंजियाँ भी संग्रहीत कर सकता है। env लोडिंग की पूरी प्राथमिकता (`env.shellEnv`, `~/.openclaw/.env`, systemd/launchd) के लिए [पर्यावरण चर](/hi/help/environment) देखें।

## Anthropic: Claude CLI का पुनः उपयोग

Anthropic सेटअप-टोकन प्रमाणीकरण अब भी समर्थित विकल्प है। Claude CLI का पुनः उपयोग (`claude -p`-शैली का उपयोग) भी इस एकीकरण के लिए अनुमोदित है; जब होस्ट पर Claude CLI लॉगिन उपलब्ध हो, तो स्थानीय/डेस्कटॉप उपयोग के लिए यही पसंदीदा विकल्प है। लंबे समय तक चलने वाले Gateway होस्ट के लिए, स्पष्ट सर्वर-साइड बिलिंग नियंत्रण के साथ Anthropic API कुंजी अब भी सबसे पूर्वानुमेय विकल्प है।

Claude CLI के पुनः उपयोग के लिए होस्ट सेटअप:

```bash
# Gateway होस्ट पर चलाएँ
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

इसके दो चरण हैं: होस्ट पर Claude Code को Anthropic में लॉग इन करें, फिर OpenClaw को Anthropic मॉडल चयन स्थानीय `claude-cli` बैकएंड के माध्यम से रूट करने और उससे मेल खाने वाली OpenClaw प्रमाणीकरण प्रोफ़ाइल संग्रहीत करने के लिए कहें।

Gateway सेवा को `PATH` पर `claude` को रिज़ॉल्व करना आवश्यक है। यदि किसी परिनियोजन को
गैर-मानक निष्पादन योग्य पथ की आवश्यकता हो, तो
[CLI बैकएंड Plugin](/hi/plugins/cli-backend-plugins) के माध्यम से एक रैपर पंजीकृत करें।

## मैन्युअल टोकन प्रविष्टि

किसी भी प्रदाता के लिए काम करती है; प्रति-एजेंट SQLite प्रमाणीकरण स्टोर में लिखती है और कॉन्फ़िगरेशन अपडेट करती है:

```bash
openclaw models auth paste-token --provider openrouter
```

OpenClaw प्रत्येक एजेंट के `openclaw-agent.sqlite` से प्रमाणीकरण प्रोफ़ाइल पढ़ता है। एंडपॉइंट विवरण (`baseUrl`, `api`, मॉडल आईडी, हेडर, टाइमआउट) प्रमाणीकरण प्रोफ़ाइल में नहीं, बल्कि `openclaw.json` या `models.json` के `models.providers.<id>` के अंतर्गत होने चाहिए।

यदि किसी पुराने इंस्टॉल में अब भी `auth-profiles.json`, `auth-state.json`, या `{ "openrouter": { "apiKey": "..." } }` जैसा समतल आकार मौजूद है, तो उसे SQLite में आयात करने के लिए `openclaw doctor --fix` चलाएँ; doctor मूल JSON फ़ाइलों के पास टाइमस्टैम्प वाले बैकअप रखता है।

Bedrock `auth: "aws-sdk"` जैसे बाहरी प्रमाणीकरण रूट क्रेडेंशियल नहीं हैं। नामित Bedrock रूट के लिए `openclaw.json` में `auth.profiles.<id>.mode: "aws-sdk"` सेट करें—प्रमाणीकरण प्रोफ़ाइल स्टोर में `type: "aws-sdk"` न लिखें। `openclaw doctor --fix` पुराने AWS SDK मार्कर को क्रेडेंशियल स्टोर से कॉन्फ़िगरेशन मेटाडेटा में माइग्रेट करता है।

### SecretRef-समर्थित क्रेडेंशियल

- `api_key` क्रेडेंशियल `keyRef: { source, provider, id }` का उपयोग कर सकते हैं
- `token` क्रेडेंशियल `tokenRef: { source, provider, id }` का उपयोग कर सकते हैं
- OAuth-मोड प्रोफ़ाइल SecretRef क्रेडेंशियल अस्वीकार करती हैं: यदि `auth.profiles.<id>.mode`, `"oauth"` है, तो उस प्रोफ़ाइल के लिए SecretRef-समर्थित `keyRef`/`tokenRef` अस्वीकार कर दिया जाता है।

## मॉडल प्रमाणीकरण स्थिति की जाँच

```bash
openclaw models status
openclaw doctor
```

ऑटोमेशन-अनुकूल जाँच, समाप्त/अनुपलब्ध होने पर निकास `1`, शीघ्र समाप्त होने पर `2`:

```bash
openclaw models status --check
```

लाइव प्रमाणीकरण जाँच (दायरा सीमित करने के लिए `--probe-provider`, `--probe-profile`, `--probe-timeout`, `--probe-concurrency`, या `--probe-max-tokens` जोड़ें):

```bash
openclaw models status --probe
```

टिप्पणियाँ:

- जाँच पंक्तियाँ प्रमाणीकरण प्रोफ़ाइल, env क्रेडेंशियल, या `models.json` से आ सकती हैं।
- यदि `auth.order.<provider>` किसी संग्रहीत प्रोफ़ाइल को छोड़ देता है, तो जाँच उसे आज़माने के बजाय उस प्रोफ़ाइल के लिए `excluded_by_auth_order` रिपोर्ट करती है।
- यदि प्रमाणीकरण मौजूद है, लेकिन OpenClaw उस प्रदाता के लिए जाँच योग्य मॉडल रिज़ॉल्व नहीं कर पाता, तो जाँच `status: no_model` रिपोर्ट करती है।
- दर-सीमा कूलडाउन मॉडल-दायरे वाले हो सकते हैं: एक मॉडल के लिए कूलडाउन में मौजूद प्रोफ़ाइल उसी प्रदाता के दूसरे मॉडल को फिर भी सेवा दे सकती है।

वैकल्पिक संचालन स्क्रिप्ट (systemd/Termux): [प्रमाणीकरण निगरानी स्क्रिप्ट](/hi/help/scripts#auth-monitoring-scripts)।

## API कुंजी रोटेशन (Gateway)

जब किसी कॉल पर प्रदाता की दर-सीमा लागू होती है, तो कुछ प्रदाता किसी वैकल्पिक कॉन्फ़िगर की गई कुंजी के साथ अनुरोध का पुनः प्रयास करते हैं।

प्रति प्रदाता कुंजी प्राथमिकता क्रम:

1. `OPENCLAW_LIVE_<PROVIDER>_KEY` (एकल ओवरराइड, एक कुंजी को पिन करता है)
2. `<PROVIDER>_API_KEYS` (अल्पविराम/स्पेस/अर्धविराम से अलग की गई सूची)
3. `<PROVIDER>_API_KEY`
4. `<PROVIDER>_API_KEY_*` (इस उपसर्ग वाला कोई भी env var)

Google प्रदाता (`google`, `google-vertex`) अतिरिक्त रूप से `GOOGLE_API_KEY` पर फ़ॉलबैक करते हैं। संयुक्त सूची का उपयोग करने से पहले उसमें से डुप्लिकेट हटा दिए जाते हैं।

OpenClaw अगली कुंजी पर केवल तभी रोटेट करता है, जब त्रुटि संदेश `rate_limit`, `rate limit`, `429`, `quota exceeded`/`quota_exceeded`, `resource exhausted`/`resource_exhausted`, या `too many requests` से मेल खाता हो। अन्य त्रुटियों पर वैकल्पिक कुंजियों के साथ पुनः प्रयास नहीं किया जाता। यदि सभी कुंजियाँ विफल हो जाती हैं, तो अंतिम प्रयास की अंतिम त्रुटि लौटाई जाती है।

<Note>
`ThrottlingException`, `concurrency limit reached`, या `workers_ai ... quota limit exceeded` जैसे प्रदाता-विशिष्ट वाक्यांश **फ़ेलओवर/पुनः प्रयास वर्गीकरण** (बार-बार विफलता पर मॉडल या प्रदाता बदलना) को संचालित करते हैं, जो ऊपर दिए API-कुंजी रोटेशन से अलग तंत्र है।
</Note>

सहेजा गया प्रमाणीकरण हटाने से प्रदाता के पास कुंजी निरस्त नहीं होती—जब प्रदाता-साइड अमान्यकरण आवश्यक हो, तो प्रदाता डैशबोर्ड में उसे रोटेट या निरस्त करें।

## Gateway के चलने के दौरान प्रदाता प्रमाणीकरण हटाना

जब आप Gateway कंट्रोल प्लेन के माध्यम से प्रदाता प्रमाणीकरण हटाते हैं, तो OpenClaw उस प्रदाता की सहेजी गई प्रमाणीकरण प्रोफ़ाइल मिटा देता है और उन सक्रिय चैट/एजेंट रन को रोक देता है, जिनका चयनित मॉडल प्रदाता हटाए गए प्रदाता से मेल खाता है। रोके गए रन `stopReason: "auth-revoked"` के साथ सामान्य रद्दीकरण/जीवनचक्र इवेंट उत्सर्जित करते हैं, ताकि कनेक्टेड क्लाइंट दिखा सकें कि क्रेडेंशियल हटाए जाने के कारण रन रुक गया।

## यह नियंत्रित करना कि कौन-सा क्रेडेंशियल उपयोग किया जाए

### OpenAI और पुराने `openai-codex` आईडी

OpenAI API-कुंजी प्रोफ़ाइल और ChatGPT/Codex OAuth प्रोफ़ाइल दोनों मानक प्रदाता आईडी `openai` का उपयोग करती हैं। नए कॉन्फ़िगरेशन के लिए `openai:*` प्रोफ़ाइल आईडी और `auth.order.openai` का उपयोग करें।

यदि पुराने कॉन्फ़िगरेशन, प्रमाणीकरण प्रोफ़ाइल आईडी, या `auth.order.openai-codex` में `openai-codex` दिखाई दे, तो उसे पुराना माइग्रेशन इनपुट मानें—नई `openai-codex` प्रोफ़ाइल न बनाएँ। चलाएँ:

```bash
openclaw doctor --fix
openclaw models auth list --provider openai
```

Doctor पुराने `openai-codex:*` प्रोफ़ाइल आईडी और `auth.order.openai-codex` प्रविष्टियों को मानक `openai` रूट में पुनर्लिखता है। OpenAI-विशिष्ट मॉडल/रनटाइम रूटिंग के लिए [OpenAI](/hi/providers/openai) देखें।

### लॉगिन के दौरान (CLI)

```bash
openclaw models auth login --provider openai --profile-id openai:ritsuko
openclaw models auth login --provider openai --profile-id openai:lain
```

`--profile-id` एक ही एजेंट के भीतर समान प्रदाता के कई OAuth लॉगिन को अलग रखता है।

`--force` चयनित एजेंट डायरेक्टरी में उस प्रदाता की सहेजी गई प्रमाणीकरण प्रोफ़ाइल मिटाता है, फिर वही प्रमाणीकरण प्रवाह दोबारा चलाता है। इसका उपयोग तब करें, जब कोई सहेजी गई प्रोफ़ाइल अटकी हो, समाप्त हो चुकी हो, या गलत खाते से जुड़ी हो। यह प्रदाता के पास क्रेडेंशियल निरस्त नहीं करता।

```bash
openclaw models auth login --provider anthropic --force
```

### प्रति-सत्र (चैट कमांड)

- `/model <alias-or-id>@<profileId>` वर्तमान सत्र के लिए किसी विशिष्ट प्रदाता क्रेडेंशियल को पिन करता है (उदाहरण प्रोफ़ाइल आईडी: `anthropic:default`, `anthropic:work`)।
- `/model` (या `/model list`) एक संक्षिप्त चयनकर्ता दिखाता है; `/model status` पूर्ण दृश्य दिखाता है (उम्मीदवार + अगली प्रमाणीकरण प्रोफ़ाइल, साथ ही कॉन्फ़िगर होने पर प्रदाता एंडपॉइंट विवरण)।

यदि आप पहले से चल रही चैट के लिए प्रमाणीकरण क्रम या प्रोफ़ाइल पिनिंग बदलते हैं, तो नया सत्र शुरू करने के लिए `/new` या `/reset` भेजें—मौजूदा सत्र रीसेट होने तक अपने वर्तमान मॉडल/प्रोफ़ाइल चयन को बनाए रखते हैं।

### प्रति-एजेंट (CLI ओवरराइड)

प्रमाणीकरण क्रम ओवरराइड उस एजेंट की SQLite प्रमाणीकरण स्थिति में संग्रहीत होते हैं:

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

किसी विशिष्ट एजेंट को लक्षित करने के लिए `--agent <id>` का उपयोग करें; कॉन्फ़िगर किए गए डिफ़ॉल्ट एजेंट का उपयोग करने के लिए इसे छोड़ दें। `openclaw models status --probe` छोड़ी गई संग्रहीत प्रोफ़ाइल को चुपचाप छोड़ने के बजाय `excluded_by_auth_order` के रूप में दिखाता है।

## समस्या निवारण

### "कोई क्रेडेंशियल नहीं मिला"

**Gateway होस्ट** पर Anthropic API कुंजी कॉन्फ़िगर करें, या Anthropic सेटअप-टोकन पथ सेट करें, फिर दोबारा जाँचें:

```bash
openclaw models status
```

### टोकन शीघ्र समाप्त होने वाला/समाप्त

कौन-सी प्रोफ़ाइल समाप्त होने वाली है, यह देखने के लिए `openclaw models status` चलाएँ। यदि Anthropic टोकन प्रोफ़ाइल अनुपलब्ध या समाप्त हो गई है, तो उसे सेटअप-टोकन के माध्यम से रीफ़्रेश करें या Anthropic API कुंजी पर माइग्रेट करें।

## संबंधित

- [सीक्रेट प्रबंधन](/hi/gateway/secrets)
- [दूरस्थ पहुँच](/hi/gateway/remote)
- [प्रमाणीकरण स्टोरेज](/hi/concepts/oauth)
