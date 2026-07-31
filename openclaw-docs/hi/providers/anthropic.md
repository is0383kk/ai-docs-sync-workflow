---
read_when:
    - आप OpenClaw में Anthropic मॉडल का उपयोग करना चाहते हैं
    - आप युग्मित कंप्यूटरों पर Claude CLI या Claude Desktop सत्रों को ब्राउज़ करना चाहते हैं
summary: OpenClaw में API कुंजियों या Claude CLI के माध्यम से Anthropic Claude का उपयोग करें
title: Anthropic
x-i18n:
    generated_at: "2026-07-27T19:44:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 08b34794352a559d549f7cf0cb88aca9cb537984049367f55be371bd8e0c10f0
    source_path: providers/anthropic.md
    workflow: 16
---

Anthropic **Claude** मॉडल परिवार बनाता है। OpenClaw दो प्रमाणीकरण मार्गों का समर्थन करता है:

- **API कुंजी** - उपयोग-आधारित बिलिंग के साथ Anthropic API तक सीधी पहुँच (`anthropic/*` मॉडल)
- **Claude CLI** - उसी होस्ट पर मौजूदा Claude Code लॉगिन का पुनः उपयोग

## उपयोग और लागत ट्रैकिंग

OpenClaw उपलब्ध Anthropic क्रेडेंशियल का पता लगाता है और उससे मेल खाने वाला उपयोग इंटरफ़ेस चुनता है:

- Claude सदस्यता/सेटअप क्रेडेंशियल कोटा अवधियाँ और वैकल्पिक अतिरिक्त-उपयोग बजट दिखाते हैं।
- `ANTHROPIC_ADMIN_KEY` या `ANTHROPIC_ADMIN_API_KEY` Control UI के **उपयोग** में प्रदाता द्वारा रिपोर्ट की गई संगठन की 30 दिनों की लागत और Messages API उपयोग दिखाता है, जिसमें दैनिक खर्च, टोकन/कैश योग, प्रमुख मॉडल और लागत श्रेणियाँ शामिल हैं।
- Anthropic प्रदाता प्रोफ़ाइल में संग्रहीत `sk-ant-admin...` क्रेडेंशियल को स्वचालित रूप से Admin API कुंजी के रूप में पहचाना जाता है।

Admin API लागत इतिहास Anthropic के [उपयोग और लागत API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api) से आता है। यह प्रदाता की वास्तविक बिलिंग है, जो OpenClaw की सत्र-व्युत्पन्न अनुमानित लागत से अलग है।

<Warning>
OpenClaw का Claude CLI बैकएंड इंस्टॉल किए गए Claude Code CLI को
गैर-इंटरैक्टिव प्रिंट मोड (`claude -p`) में चलाता है। Anthropic के वर्तमान Claude Code दस्तावेज़
इस मोड को Agent SDK/प्रोग्रामेटिक उपयोग बताते हैं। Anthropic के 15 जून, 2026 के
समर्थन अपडेट ने अलग Agent SDK बिलिंग के घोषित बदलाव को रोक दिया: Claude
Agent SDK, `claude -p`, और तृतीय-पक्ष ऐप का उपयोग अब भी साइन-इन की गई
सदस्यता की उपयोग सीमाओं में गिना जाता है, और पहले घोषित मासिक Agent SDK
क्रेडिट तब तक उपलब्ध नहीं है, जब तक Anthropic उस योजना में संशोधन कर रहा है।

इंटरैक्टिव Claude Code अब भी साइन-इन किए गए Claude प्लान की सीमाओं में गिना जाता है।
API कुंजी प्रमाणीकरण सीधे उपयोग के अनुसार भुगतान वाली बिलिंग है और उस प्लान पर निर्भर नहीं करता।
लंबे समय तक चलने वाले Gateway होस्ट, साझा स्वचालन और पूर्वानुमेय उत्पादन
खर्च के लिए Anthropic API कुंजी का उपयोग करें।

Anthropic के वर्तमान समर्थन लेख OpenClaw रिलीज़ के बिना भी इस व्यवहार को
बदल सकते हैं:

- [Claude Code CLI संदर्भ](https://code.claude.com/docs/en/cli-usage)
- [अपने Claude प्लान के साथ Claude Agent SDK का उपयोग करें](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)
- [अपने Pro या Max प्लान के साथ Claude Code का उपयोग करें](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
- [अपने Team या Enterprise प्लान के साथ Claude Code का उपयोग करें](https://support.claude.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan)
- [Claude Code की लागत प्रबंधित करें](https://code.claude.com/docs/en/costs)

</Warning>

## शुरुआत करना

<Tabs>
  <Tab title="API कुंजी">
    **इनके लिए सर्वोत्तम:** मानक API पहुँच और उपयोग-आधारित बिलिंग।

    <Steps>
      <Step title="अपनी API कुंजी प्राप्त करें">
        [Anthropic Console](https://console.anthropic.com/) में API कुंजी बनाएँ।
      </Step>
      <Step title="ऑनबोर्डिंग चलाएँ">
        ```bash
        openclaw onboard
        # चुनें: Anthropic API कुंजी
        ```

        या कुंजी सीधे प्रदान करें:

        ```bash
        openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
        ```
      </Step>
      <Step title="सत्यापित करें कि मॉडल उपलब्ध है">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    ### कॉन्फ़िगरेशन उदाहरण

    ```json5
    {
      env: { ANTHROPIC_API_KEY: "example-anthropic-key-not-real" },
      agents: { defaults: { model: { primary: "anthropic/claude-opus-5" } } },
    }
    ```

  </Tab>

  <Tab title="Claude CLI">
    **इनके लिए सर्वोत्तम:** अलग API कुंजी के बिना मौजूदा Claude CLI लॉगिन का पुनः उपयोग।

    <Steps>
      <Step title="सुनिश्चित करें कि Claude CLI इंस्टॉल है और उसमें लॉगिन किया गया है">
        इससे सत्यापित करें:

        ```bash
        claude --version
        ```
      </Step>
      <Step title="ऑनबोर्डिंग चलाएँ">
        ```bash
        openclaw onboard
        # चुनें: Claude CLI
        ```

        OpenClaw मौजूदा Claude CLI क्रेडेंशियल का पता लगाकर उनका पुनः उपयोग करता है।
      </Step>
      <Step title="सत्यापित करें कि मॉडल उपलब्ध है">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    <Note>
    Claude CLI बैकएंड के सेटअप और रनटाइम का विवरण [CLI बैकएंड](/hi/gateway/cli-backends) में है।
    </Note>

    <Warning>
    Claude CLI के पुनः उपयोग के लिए आवश्यक है कि OpenClaw प्रक्रिया उसी होस्ट पर चले जहाँ
    Claude CLI लॉगिन मौजूद है। Docker इंस्टॉलेशन कंटेनर होम को स्थायी रख सकते हैं और
    वहीं Claude Code में लॉगिन कर सकते हैं; देखें
    [Docker में Claude CLI बैकएंड](/hi/install/docker#claude-cli-backend-in-docker)।
    [Podman](/hi/install/podman) जैसे अन्य कंटेनर इंस्टॉलेशन होस्ट के
    `~/.claude` को सेटअप या रनटाइम में माउंट नहीं करते; वहाँ Anthropic API कुंजी का उपयोग करें, या
    OpenClaw-प्रबंधित OAuth वाला प्रदाता चुनें, जैसे
    [OpenAI Codex](/hi/providers/openai)।
    </Warning>

    ### सेटअप टोकन प्राप्त करें

    Claude Code इंस्टॉल वाली किसी भी मशीन पर `claude setup-token` चलाएँ। यह
    `sk-ant-oat01-` से शुरू होने वाला दीर्घकालिक टोकन प्रिंट करता है।

    ऑनबोर्डिंग के दौरान macOS ऐप में **API कुंजी या टोकन से कनेक्ट करें** के अंतर्गत
    **Anthropic setup-token** चुनकर टोकन चिपकाएँ, या इसका उपयोग करें:

    ```bash
    openclaw models auth login --provider anthropic --method setup-token
    ```

    ### कॉन्फ़िगरेशन उदाहरण

    CLI रनटाइम ओवरराइड के साथ कैनोनिकल Anthropic मॉडल संदर्भ को प्राथमिकता दें:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-5" },
          models: {
            "anthropic/claude-opus-5": {
              agentRuntime: { id: "claude-cli" },
            },
          },
        },
      },
    }
    ```

    पुराने `claude-cli/claude-opus-4-7` मॉडल संदर्भ संगतता के लिए अब भी काम करते हैं,
    लेकिन नए कॉन्फ़िगरेशन में प्रदाता/मॉडल चयन को
    `anthropic/*` के रूप में रखना चाहिए और निष्पादन बैकएंड को प्रदाता/मॉडल रनटाइम नीति में रखना चाहिए।

    ### बिलिंग और `claude -p`

    OpenClaw Claude CLI रन के लिए Claude Code के गैर-इंटरैक्टिव `claude -p` पथ का उपयोग करता है।
    Anthropic वर्तमान में उस पथ को Agent SDK/प्रोग्रामेटिक उपयोग मानता है:

    - Anthropic के 15 जून, 2026 के समर्थन अपडेट ने पहले घोषित
      अलग Agent SDK क्रेडिट प्लान को रोक दिया।
    - सदस्यता-प्लान वाले Claude Agent SDK, `claude -p`, और तृतीय-पक्ष ऐप का उपयोग
      अब भी साइन-इन की गई सदस्यता की उपयोग सीमाओं में गिना जाता है।
    - पहले घोषित मासिक Agent SDK क्रेडिट तब तक उपलब्ध नहीं है, जब तक
      Anthropic उस योजना में संशोधन कर रहा है।
    - Console/API-कुंजी लॉगिन उपयोग के अनुसार भुगतान वाली API बिलिंग का उपयोग करते हैं और उन्हें
      सदस्यता Agent SDK क्रेडिट नहीं मिलता।

    रोक संबंधी सूचना के लिए Anthropic का [Agent SDK प्लान
    लेख](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)
    और सदस्यता व्यवहार के लिए Claude Code प्लान के
    [Pro/Max](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
    तथा
    [Team/Enterprise](https://support.claude.com/en/articles/11845131-use-claude-code-with-your-team-or-enterprise-plan)
    लेख देखें।

    Anthropic OpenClaw रिलीज़ के बिना Claude Code की बिलिंग और दर-सीमा व्यवहार को
    बदल सकता है। जब बिलिंग की पूर्वानुमेयता महत्वपूर्ण हो, तो `claude auth status`, `/status`, और
    Anthropic के लिंक किए गए दस्तावेज़ देखें।

    <Tip>
    साझा उत्पादन स्वचालन के लिए Claude CLI के बजाय Anthropic API कुंजी का उपयोग करें।
    OpenClaw
    [OpenAI Codex](/hi/providers/openai), [Qwen Cloud](/hi/providers/qwen),
    [MiniMax](/hi/providers/minimax), और [Z.AI / GLM](/hi/providers/zai) के सदस्यता-शैली विकल्पों का भी समर्थन करता है।
    </Tip>

  </Tab>
</Tabs>

## अलग-अलग कंप्यूटरों पर Claude सत्र

बंडल किया गया Anthropic Plugin सामान्य सत्र साइडबार में **Claude Code** समूह जोड़ता है।
पंक्तियाँ सामान्य चैट पेन में खुलती हैं। यह Gateway और कनेक्ट किए गए Node होस्ट पर
गैर-संग्रहीत Claude Code सत्र खोजता है:

- Claude CLI सत्र मान्य प्रोजेक्ट-इंडेक्स रिकॉर्ड से आते हैं। इंडेक्स न किए गए
  ट्रांसक्रिप्ट के लिए, सीमित मेटाडेटा फ़ॉलबैक `~/.claude/projects/` के अंतर्गत एक साथ चल रहे गैर-साइडचेन
  इंटरैक्टिव (`cli`) और हेडलेस Agent SDK CLI (`sdk-cli`) सत्रों को पहचानता है।
- जब Claude Desktop का मेटाडेटा उसी Claude Code सत्र ID की ओर संकेत करता है, तो उसके सत्र
  Desktop शीर्षक, गतिविधि समय और
  संग्रह स्थिति का उपयोग करते हैं।
- केवल CLI वाले सत्र में संग्रह फ़्लैग नहीं होता, इसलिए उसका
  ट्रांसक्रिप्ट मौजूद रहने तक वह दिखाई देता है।

खोज के लिए किसी अतिरिक्त OpenClaw कॉन्फ़िगरेशन की आवश्यकता नहीं है। Anthropic Plugin
बंडल किया गया है और डिफ़ॉल्ट रूप से सक्षम है; स्थानीय `~/.claude/projects/` डायरेक्टरी मौजूद होने पर
मूल macOS Node केवल-पढ़ने योग्य Claude सत्र कमांड की घोषणा करता है।
इन कमांड के पहली बार दिखाई देने पर Node पेयरिंग अपग्रेड को स्वीकृति दें।

साइडबार पंक्तियों को उनके Gateway या पेयर किए गए Node होस्ट के अनुसार समूहित करता है और प्रत्येक
कंप्यूटर से उत्तर मिलते ही उस होस्ट का नवीनतम सीमित पृष्ठ दिखाता है। होस्ट-कनेक्टिविटी
बदलने के बाद, पृष्ठ के दोबारा फ़ोकस में आने पर और दिखाई देते समय अधिकतम हर
30 सेकंड में यह फिर से मिलान करता है, ताकि OpenClaw के बाहर बनाए गए Claude सत्र
पुनः लोड किए बिना दिखाई दें। बदली हुई कैटलॉग पर अधिक तेज़ अनुवर्ती चरण चलता है। अधिक इतिहास वाले
प्रत्येक होस्ट का अगला पृष्ठ जोड़ने के लिए कैटलॉग समूह के नीचे **और सत्र
लोड करें** का उपयोग करें; जोड़ी गई पंक्तियाँ दिखाई देती रहती हैं और रीफ़्रेश के दौरान
उसी गहराई तक दोबारा प्राप्त की जाती हैं। कैटलॉग क्लाइंट `sessions.catalog.list` का उपयोग करते हैं; पंक्ति खोलने के लिए
`sessions.catalog.read` का उपयोग होता है।

टर्मिनल नियंत्रण `claude` को सेवा/डेमन PATH से पहले स्वामी होस्ट उपयोगकर्ता के लॉगिन-शेल
PATH से हल करता है। इससे ऐप द्वारा शुरू किए गए सत्र उस Claude CLI के अनुरूप रहते हैं
जो ऑपरेटर को सामान्य टर्मिनल में मिलता है।

पंक्ति चुनने पर सबसे नया ट्रांसक्रिप्ट पृष्ठ पहले पढ़ा जाता है। **पुराने ट्रांसक्रिप्ट
आइटम लोड करें** एक अपारदर्शी बाइट कर्सर का अनुसरण करता है और पूरा इतिहास लोड करने के बजाय
JSONL फ़ाइल का एक और सीमित खंड पढ़ता है। सामान्य उपयोगकर्ता, सहायक,
रीज़निंग, टूल-कॉल और टूल-परिणाम सामग्री संरक्षित रहती है। Node/Gateway की सुरक्षा सीमा से
बड़ा कोई व्यक्तिगत आइटम स्पष्ट रूप से काटा हुआ चिह्नित किया जाता है।

Gateway-स्थानीय `claude-cli` पंक्ति के लिए सामान्य कंपोज़र में टाइप करने पर
`sessions.catalog.continue` कॉल होता है। OpenClaw स्थानीय कैटलॉग रिकॉर्ड को फिर से हल करता है,
मॉडल-लॉक किया हुआ मूल सत्र बनाता या पुनः उपयोग करता है, अधिकतम 200 दृश्यमान
आइटम या 512 KiB आयात करता है और Claude CLI बाइंडिंग को आरंभिक डेटा देता है। पहला टर्न
`--fork-session` के साथ फिर शुरू होता है; Claude फ़ोर्क को नया सत्र ID देता है, इसलिए बाद के टर्न
फ़ोर्क का उपयोग करते हैं और स्रोत सत्र अपरिवर्तित रहता है।

हेडलेस Node होस्ट नीचे दी गई Node-स्थानीय सेटिंग सक्षम करके और Node होस्ट को पुनः प्रारंभ करके
अपनी Claude CLI पंक्तियों को जारी रखने योग्य भी बना सकता है:

```json5
{
  nodeHost: {
    agentRuns: {
      claude: { enabled: true },
    },
  },
}
```

Node `agent.cli.claude.run.v1` की घोषणा केवल तब करता है, जब सेटिंग सक्षम हो
और उसका स्थानीय `claude` निष्पादन योग्य प्रोग्राम हल हो जाए। OpenClaw उस Node पर कैटलॉग
रिकॉर्ड को फिर से हल करता है, वही सीमित इतिहास आयात करता है और अपनाए गए
सत्र को Node तथा कैटलॉग द्वारा रिपोर्ट की गई कार्यशील डायरेक्टरी से बाँधता है। प्रत्येक टर्न
उस Node की Claude फ़ाइलों और लॉगिन का उपयोग करके Node की वास्तविक `claude -p` प्रक्रिया चलाता है।
Node की exec स्वीकृति नीति अब भी लागू होती है; Gateway इस वैकल्पिक चयन को बाध्य नहीं कर सकता।

Node निरंतरता v1 केवल एक बार के लिए है। इसमें Gateway लूपबैक MCP कॉन्फ़िगरेशन और
Gateway Skills Plugin आर्ग्युमेंट शामिल नहीं होते, यह Gateway ट्रांसक्रिप्ट से दोबारा आरंभिक डेटा नहीं लेता और
अटैचमेंट तथा छवियों को अस्वीकार करता है। Claude Desktop पंक्तियाँ केवल देखने योग्य रहती हैं। मूल
macOS ऐप Node भी तब तक केवल देखने योग्य रहते हैं, जब तक ऐप रन कमांड की घोषणा नहीं करता।

<Note>
पेयर किए गए Node के Claude सत्र तब तक केवल-पढ़ने योग्य रहते हैं, जब तक हेडलेस Node स्पष्ट रूप से
`agent.cli.claude.run.v1` की घोषणा नहीं करता। OpenClaw कभी भी Claude Desktop
मेटाडेटा को संशोधित या Claude सत्रों को संग्रहीत नहीं करता। इस पृष्ठ को लेखन स्कोप वाला ऑपरेटर कनेक्शन
चाहिए, क्योंकि यह प्रमाणित `node.invoke` का उपयोग करता है; जारी रखने में सक्षम Node पर भी सूची बनाना और पढ़ना
केवल-पढ़ने योग्य रहता है।
</Note>

देखें [Nodes: Claude सत्र और ट्रांसक्रिप्ट](/hi/nodes#claude-sessions-and-transcripts)
Node कमांड और सुरक्षा सीमा के लिए।

## चिंतन के डिफ़ॉल्ट (Claude Opus 5, Sonnet 5, Mythos 5, Fable 5, 4.8, और 4.6)

`anthropic/claude-opus-5` डिफ़ॉल्ट रूप से `high` प्रयास पर अनुकूली चिंतन का उपयोग करता है।
चिंतन अक्षम करने के लिए `/think off`, या मॉडल के
उच्चतर मूल प्रयास स्तरों के लिए `/think xhigh|max` का उपयोग करें। OpenClaw, Opus 5 के लिए मैन्युअल चिंतन बजट, कस्टम
सैंपलिंग पैरामीटर, असिस्टेंट प्रीफिल और Priority Tier को छोड़ देता है, क्योंकि
Anthropic इस मॉडल पर इन अनुरोध सुविधाओं का समर्थन नहीं करता। कैटलॉग
इसकी 1,000,000-टोकन कॉन्टेक्स्ट विंडो, 128,000-टोकन आउटपुट सीमा, इमेज
इनपुट और `$5/$25` इनपुट/आउटपुट मूल्य प्रकाशित करता है।

`anthropic/claude-sonnet-5` उन्हीं अनुकूली-चिंतन डिफ़ॉल्ट और अनुरोध
प्रतिबंधों का उपयोग करता है। कैटलॉग 31 अगस्त, 2026 तक Anthropic के आरंभिक `$2/$10` इनपुट/आउटपुट
मूल्य का उपयोग करता है; मानक `$3/$15` मूल्य 1 सितंबर, 2026 से आरंभ होता है।

`anthropic/claude-fable-5` हमेशा अनुकूली चिंतन का उपयोग करता है और डिफ़ॉल्ट रूप से `high`
प्रयास का उपयोग करता है। Anthropic इस मॉडल के लिए चिंतन अक्षम करने की अनुमति नहीं देता, इसलिए
`/think off` और `/think minimal` इसके बजाय `low` प्रयास पर मैप होते हैं। OpenClaw
Fable 5 अनुरोधों के लिए कस्टम टेम्परेचर मान भी छोड़ देता है, क्योंकि Anthropic
चिंतन-सक्षम किसी भी अनुरोध पर टेम्परेचर ओवरराइड अस्वीकार करता है।

`anthropic/claude-mythos-5` सीमित-पहुँच वाला मॉडल है, जिसका अनुकूली-चिंतन अनुबंध भी हमेशा चालू
रहता है। OpenClaw डिफ़ॉल्ट रूप से `high` का उपयोग करता है, `/think off` और
`/think minimal` को `low` पर मैप करता है और कॉलर द्वारा चुने गए सैंपलिंग पैरामीटर छोड़ देता है।
कैटलॉग इसकी 1,000,000-टोकन कॉन्टेक्स्ट विंडो, 128,000-टोकन आउटपुट
सीमा, इमेज इनपुट और `$10/$50` इनपुट/आउटपुट मूल्य प्रकाशित करता है।

OpenClaw में Claude Opus 4.8 डिफ़ॉल्ट रूप से चिंतन बंद रखता है। जब आप
`/think high|xhigh|max` के साथ अनुकूली चिंतन स्पष्ट रूप से सक्षम करते हैं, तो OpenClaw
Anthropic के Opus 4.8 प्रयास मान भेजता है; Claude 4.6 मॉडल (Opus 4.6 और Sonnet 4.6)
डिफ़ॉल्ट रूप से `adaptive` का उपयोग करते हैं।

प्रति संदेश `/think:<level>` से या मॉडल पैरामीटर में ओवरराइड करें:

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-5": {
          params: { thinking: "high" },
        },
      },
    },
  },
}
```

<Note>
संबंधित Anthropic दस्तावेज़:
- [अनुकूली चिंतन](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [विस्तारित चिंतन](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

</Note>

## सुरक्षा अस्वीकृति फ़ॉलबैक (Claude Fable 5)

<Warning>
Claude Fable 5 का उपयोग करने का अर्थ Claude Opus 4.8 का भी उपयोग करना है। Fable 5 में
सुरक्षा क्लासिफ़ायर शामिल हैं जो किसी अनुरोध को अस्वीकार कर सकते हैं, और Anthropic द्वारा स्वीकृत
रिकवरी यह है कि उस टर्न को `claude-opus-4-8` पूरा करे। OpenClaw प्रत्यक्ष API-कुंजी अनुरोधों के लिए इसे
स्वचालित रूप से स्वीकार करता है, इसलिए कुछ Fable टर्न के उत्तर Claude Opus 4.8 द्वारा दिए
जाते हैं और उनका बिल भी उसी रूप में बनता है। यदि आपकी नीति या बजट
Opus द्वारा पूरे किए गए टर्न स्वीकार नहीं कर सकता, तो `anthropic/claude-fable-5` न चुनें।
</Warning>

### यह क्यों मौजूद है

Fable 5 क्लासिफ़ायर प्रतिबंधित डोमेन के अनुरोधों पर `stop_reason: "refusal"` लौटाते
हैं, और वे हानिरहित कार्यों से सटे मामलों पर भी गलत-सकारात्मक परिणाम देते हैं (सुरक्षा
टूलिंग, जीवन विज्ञान, या मॉडल से उसका कच्चा
तर्क दोहराने को कहना भी)। फ़ॉलबैक के बिना, टर्न त्रुटि के साथ समाप्त हो जाता है, भले ही
दूसरा Claude मॉडल उसे सहजता से पूरा कर सकता हो—Anthropic का अपना अस्वीकृति संदेश
API इंटीग्रेटरों से फ़ॉलबैक मॉडल कॉन्फ़िगर करने को कहता है।

### यह कैसे काम करता है

1. `anthropic/claude-fable-5` को भेजे जाने वाले प्रत्येक प्रत्यक्ष API-कुंजी अनुरोध के लिए, OpenClaw
   Anthropic का सर्वर-साइड फ़ॉलबैक ऑप्ट-इन भेजता है: 
   `server-side-fallback-2026-06-01` बीटा हेडर और
   `fallbacks: [{"model": "claude-opus-4-8"}]`। Claude Opus 4.8 ही एकमात्र
   फ़ॉलबैक लक्ष्य है जिसकी Anthropic Fable 5 के लिए अनुमति देता है।
2. केवल सुरक्षा-क्लासिफ़ायर की अस्वीकृति फ़ॉलबैक ट्रिगर करती है। दर सीमाएँ,
   ओवरलोड और सर्वर त्रुटियाँ पहले की तरह ही व्यवहार करती हैं और
   OpenClaw के सामान्य [मॉडल फ़ेलओवर](/hi/concepts/model-failover) से गुजरती हैं।
3. रिकवरी उसी कॉल के भीतर होती है। किसी भी आउटपुट से पहले हुई अस्वीकृति
   विलंबता के अतिरिक्त दिखाई नहीं देती; पूरा उत्तर Opus 4.8 से आता है। बीच स्ट्रीम में
   अस्वीकृति होने पर आंशिक टेक्स्ट उस उपसर्ग के रूप में रखा जाता है जहाँ से फ़ॉलबैक
   मॉडल जारी रखता है, जबकि अस्वीकृत मॉडल के तर्क और टूल कॉल
   Anthropic के रीप्ले नियमों के अनुसार त्याग दिए जाते हैं (उन्हें वापस प्रतिध्वनित या
   निष्पादित नहीं किया जाना चाहिए)।
4. यदि Claude Opus 4.8 भी अस्वीकार करता है, तो टर्न अस्वीकृति को
   त्रुटि के रूप में प्रदर्शित करता है, ठीक वैसे ही जैसे इस सुविधा से पहले होता था।

फ़ॉलबैक Anthropic API स्तर पर होता है, इसलिए `claude-opus-4-8` का
आपकी कॉन्फ़िगर की गई मॉडल सूची या फ़ॉलबैक श्रृंखला में होना आवश्यक नहीं है—Fable-सक्षम
API कुंजी हमेशा Opus को उपलब्ध करा सकती है।

### अवलोकनीयता और बिलिंग

- फ़ॉलबैक द्वारा पूरे किए गए टर्न में असिस्टेंट संदेश पर `provider_fallback` निदान दर्ज होता है,
  जो `fromModel` और `toModel` का नाम देता है, और संदेश का
  `responseModel`, `claude-opus-4-8` रिपोर्ट करता है।
- Anthropic प्रत्येक प्रयास के अनुसार बिल करता है: आउटपुट से पहले की अस्वीकृति निःशुल्क होती है, और रिकवरी
  का बिल Claude Opus 4.8 की दरों पर बनता है (वर्तमान में Fable 5 की दरों का आधा)। OpenClaw का
  प्रति-टर्न लागत अनुमान मेल बनाए रखने के लिए फ़ॉलबैक द्वारा पूरे किए गए टर्न का मूल्य Opus दरों पर लगाता है।
- बीच स्ट्रीम में हुई अस्वीकृति के लिए Anthropic की ओर से पहले ही स्ट्रीम किए गए Fable के आंशिक भाग का बिल भी
  लगता है; वह भाग API के प्रति-प्रयास उपयोग में रिपोर्ट होता है,
  लेकिन OpenClaw के प्रति-टर्न अनुमान में शामिल नहीं किया जाता।

### दायरा

यह `api.anthropic.com` के विरुद्ध API-कुंजी प्रमाणीकरण वाले
`anthropic/claude-fable-5` पर लागू होता है। OAuth (Claude CLI सदस्यता का पुनः उपयोग), प्रॉक्सी बेस URL,
Bedrock, Vertex और Foundry अनुरोध अपरिवर्तित रहते हैं और वहाँ अब भी
अस्वीकृतियों को त्रुटि के रूप में प्रदर्शित करते हैं।

लाइव सत्यापित: Fable 5 से उसकी कच्ची विचार-श्रृंखला दोहराने के लिए कहने वाला एक हानिरहित प्रॉम्प्ट,
फ़ॉलबैक के बिना भेजे जाने पर `category: "reasoning_extraction"` के साथ अस्वीकार होता है,
और OpenClaw के माध्यम से वही प्रॉम्प्ट `provider_fallback` निदान संलग्न
एक सामान्य Opus-द्वारा-प्रदत्त उत्तर लौटाता है।

अंतर्निहित व्यवहार के लिए Anthropic की [अस्वीकृतियाँ और फ़ॉलबैक
मार्गदर्शिका](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)
देखें।

## प्रॉम्प्ट कैशिंग

OpenClaw API-कुंजी प्रमाणीकरण के लिए Anthropic की प्रॉम्प्ट कैशिंग सुविधा का समर्थन करता है।

| मान               | कैश अवधि | विवरण                            |
| ------------------- | -------------- | -------------------------------------- |
| `"short"` (डिफ़ॉल्ट) | 5 मिनट      | API-कुंजी प्रमाणीकरण के लिए स्वचालित रूप से लागू |
| `"long"`            | 1 घंटा         | विस्तारित कैश                         |
| `"none"`            | कोई कैशिंग नहीं     | प्रॉम्प्ट कैशिंग अक्षम करें                 |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="प्रति-एजेंट कैश ओवरराइड">
    मॉडल-स्तरीय पैरामीटर को आधाररेखा के रूप में उपयोग करें, फिर `agents.entries.*.params` के माध्यम से विशिष्ट एजेंटों को ओवरराइड करें:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": {
              params: { cacheRetention: "long" },
            },
          },
        },
        list: [
          { id: "research", default: true },
          { id: "alerts", params: { cacheRetention: "none" } },
        ],
      },
    }
    ```

    कॉन्फ़िग मर्ज क्रम:

    1. `agents.defaults.models["provider/model"].params`
    2. `agents.entries.*.params` (मिलान करने वाला `id`, कुंजी के अनुसार ओवरराइड करता है)

    इससे एक एजेंट लंबे समय तक चलने वाला कैश रख सकता है, जबकि उसी मॉडल पर दूसरा एजेंट बर्स्टी/कम-पुनः उपयोग वाले ट्रैफ़िक के लिए कैशिंग अक्षम कर सकता है।

  </Accordion>

  <Accordion title="Bedrock Claude नोट्स">
    - Bedrock (`amazon-bedrock/*anthropic.claude*`) पर Anthropic Claude मॉडल कॉन्फ़िगर होने पर `cacheRetention` पास-थ्रू स्वीकार करते हैं।
    - गैर-Anthropic Bedrock मॉडल को रनटाइम पर अनिवार्य रूप से `cacheRetention: "none"` पर सेट किया जाता है।
    - जब कोई स्पष्ट मान सेट नहीं होता, तो API-कुंजी स्मार्ट डिफ़ॉल्ट Claude-on-Bedrock संदर्भों के लिए `cacheRetention: "short"` भी भरते हैं।

  </Accordion>
</AccordionGroup>

## उन्नत कॉन्फ़िगरेशन

<AccordionGroup>
  <Accordion title="तेज़ मोड">
    OpenClaw का साझा `/fast` टॉगल, `api.anthropic.com` को भेजे जाने वाले प्रत्यक्ष API-कुंजी ट्रैफ़िक के लिए Anthropic का `service_tier` फ़ील्ड सेट करता है।

    | कमांड | इस पर मैप होता है |
    |---------|---------|
    | `/fast on` | `service_tier: "auto"` |
    | `/fast off` | `service_tier: "standard_only"` |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-sonnet-4-6": {
              params: { fastMode: true },
            },
          },
        },
      },
    }
    ```

    <Note>
    - केवल API कुंजी से किए गए प्रत्यक्ष `api.anthropic.com` अनुरोधों पर लागू होता है। OAuth/सदस्यता-टोकन अनुरोधों और प्रॉक्सी रूट को कभी `service_tier` फ़ील्ड नहीं मिलता।
    - दोनों सेट होने पर स्पष्ट `serviceTier` या `service_tier` पैरामीटर, `/fast` को ओवरराइड करते हैं।
    - Claude Opus 5 और Sonnet 5 Priority Tier का समर्थन नहीं करते, इसलिए OpenClaw इन मॉडलों के लिए `service_tier` छोड़ देता है।
    - Priority Tier क्षमता के बिना खातों पर `service_tier: "auto"`, `standard` में परिणत हो सकता है।

    </Note>

  </Accordion>

  <Accordion title="मीडिया समझ (इमेज और PDF)">
    बंडल किया गया Anthropic Plugin इमेज और PDF समझ को पंजीकृत करता है। OpenClaw
    कॉन्फ़िगर किए गए Anthropic प्रमाणीकरण से मीडिया क्षमताओं को स्वतः निर्धारित करता है; किसी
    अतिरिक्त कॉन्फ़िग की आवश्यकता नहीं है।

    | गुण        | मान                 |
    | --------------- | --------------------- |
    | डिफ़ॉल्ट मॉडल   | `claude-opus-5`       |
    | समर्थित इनपुट | इमेज, PDF दस्तावेज़ |

    जब किसी वार्तालाप में इमेज या PDF संलग्न किया जाता है, तो OpenClaw उसे स्वचालित रूप से
    Anthropic मीडिया समझ प्रदाता के माध्यम से रूट करता है।

  </Accordion>

  <Accordion title="1M कॉन्टेक्स्ट विंडो">
    Claude Opus 5, Sonnet 5, Mythos 5 और Fable 5 में ठीक
    1,000,000-टोकन इनपुट विंडो है और ये अधिकतम 128,000 आउटपुट टोकन का समर्थन करते हैं।
    Anthropic की 1M कॉन्टेक्स्ट विंडो अनुकूली
    चिंतन वाले Claude 4.x मॉडलों पर भी GA है: Opus 4.8,
    Opus 4.7, Opus 4.6 और Sonnet 4.6। OpenClaw इन मॉडलों का आकार
    स्वचालित रूप से निर्धारित करता है, `params.context1m` की आवश्यकता नहीं है:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-opus-5": {},
            "anthropic/claude-sonnet-5": {},
            "anthropic/claude-mythos-5": {},
            "anthropic/claude-opus-4-6": {},
          },
        },
      },
    }
    ```

    पुराने कॉन्फ़िग `params.context1m: true` रख सकते हैं; इन मॉडलों के लिए यह हानिरहित नो-ऑप है
    और OpenClaw अब अप्रचलित
    `context-1m-2025-08-07` बीटा हेडर किसी भी स्थिति में नहीं भेजता। उस मान वाली पुरानी `anthropicBeta` कॉन्फ़िग
    प्रविष्टियाँ अनुरोध हेडर रिज़ॉल्यूशन के दौरान हटा दी जाती हैं, और
    असमर्थित पुराने Claude मॉडल अपनी सामान्य कॉन्टेक्स्ट विंडो पर बने रहते हैं।

    Claude CLI बैकएंड
    (`claude-cli/*`) के लिए `params.context1m: true` भी इसी तरह व्यवहार करता है: योग्य GA-सक्षम Opus और Sonnet मॉडल को पहले से ही
    1M विंडो स्वचालित रूप से मिलती है, इसलिए वहाँ भी यह पैरामीटर वैकल्पिक है।

    <Warning>
    आपके Anthropic क्रेडेंशियल पर लॉन्ग-कॉन्टेक्स्ट पहुँच आवश्यक है। OAuth/सदस्यता टोकन प्रमाणीकरण अपने आवश्यक Anthropic बीटा हेडर बनाए रखता है, लेकिन यदि अप्रचलित 1M बीटा हेडर पुराने कॉन्फ़िग में रह गया है, तो OpenClaw उसे हटा देता है।
    </Warning>

  </Accordion>

  <Accordion title="Claude Opus 5 1M कॉन्टेक्स्ट">
    `anthropic/claude-opus-5` और इसके `claude-cli` वैरिएंट में डिफ़ॉल्ट रूप से 1M कॉन्टेक्स्ट
    विंडो है; `params.context1m: true` की आवश्यकता नहीं है।
  </Accordion>
</AccordionGroup>

## समस्या निवारण

<AccordionGroup>
  <Accordion title="401 त्रुटियाँ / टोकन अचानक अमान्य">
    Anthropic टोकन प्रमाणीकरण की समय-सीमा समाप्त हो जाती है और इसे निरस्त किया जा सकता है। नए सेटअप के लिए इसके बजाय Anthropic API कुंजी का उपयोग करें।
  </Accordion>

  <Accordion title='प्रदाता "anthropic" के लिए कोई API कुंजी नहीं मिली'>
    Anthropic प्रमाणीकरण **प्रति एजेंट** होता है; नए एजेंट मुख्य एजेंट की कुंजियाँ प्राप्त नहीं करते। उस एजेंट के लिए ऑनबोर्डिंग फिर से चलाएँ (या Gateway होस्ट पर API कुंजी कॉन्फ़िगर करें), फिर `openclaw models status` से सत्यापित करें।
  </Accordion>

  <Accordion title='प्रोफ़ाइल "anthropic:default" के लिए कोई क्रेडेंशियल नहीं मिला'>
    कौन-सी प्रमाणीकरण प्रोफ़ाइल सक्रिय है, यह देखने के लिए `openclaw models status` चलाएँ। ऑनबोर्डिंग फिर से चलाएँ, या उस प्रोफ़ाइल पथ के लिए API कुंजी कॉन्फ़िगर करें।
  </Accordion>

  <Accordion title="कोई उपलब्ध प्रमाणीकरण प्रोफ़ाइल नहीं (सभी कूलडाउन में हैं)">
    `auth.unusableProfiles` के लिए `openclaw models status --json` जाँचें। Anthropic की दर-सीमा के कूलडाउन मॉडल-विशिष्ट हो सकते हैं, इसलिए समान स्तर का कोई अन्य Anthropic मॉडल अभी भी उपयोग योग्य हो सकता है। कोई अन्य Anthropic प्रोफ़ाइल जोड़ें या कूलडाउन समाप्त होने की प्रतीक्षा करें।
  </Accordion>
</AccordionGroup>

<Note>
अधिक सहायता: [समस्या निवारण](/hi/help/troubleshooting) और [अक्सर पूछे जाने वाले प्रश्न](/hi/help/faq)।
</Note>

## संबंधित

<CardGroup cols={2}>
  <Card title="मॉडल चयन" href="/hi/concepts/model-providers" icon="layers">
    प्रदाताओं, मॉडल संदर्भों और फ़ेलओवर व्यवहार का चयन।
  </Card>
  <Card title="CLI बैकएंड" href="/hi/gateway/cli-backends" icon="terminal">
    Claude CLI बैकएंड का सेटअप और रनटाइम विवरण।
  </Card>
  <Card title="प्रॉम्प्ट कैशिंग" href="/hi/reference/prompt-caching" icon="database">
    प्रदाताओं में प्रॉम्प्ट कैशिंग कैसे काम करती है।
  </Card>
  <Card title="OAuth और प्रमाणीकरण" href="/hi/gateway/authentication" icon="key">
    प्रमाणीकरण का विवरण और क्रेडेंशियल के पुनः उपयोग के नियम।
  </Card>
</CardGroup>
