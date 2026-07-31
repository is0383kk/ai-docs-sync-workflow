---
read_when:
    - आप OpenClaw द्वारा समर्थित सुविधाओं की पूरी सूची चाहते हैं
summary: चैनलों, रूटिंग, मीडिया और UX में OpenClaw की क्षमताएँ।
title: सुविधाएँ
x-i18n:
    generated_at: "2026-07-27T17:57:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bc3ebdd87a0f6ea0f3d75d029bf7cae469ecd9db84a165bd47c4896936fe303
    source_path: concepts/features.md
    workflow: 16
---

## मुख्य विशेषताएँ

<Columns>
  <Card title="चैनल" icon="message-square" href="/hi/channels">
    एक ही Gateway के साथ Discord, iMessage, Signal, Slack, Telegram, WhatsApp, WebChat और बहुत कुछ।
  </Card>
  <Card title="Plugins" icon="plug" href="/hi/tools/plugin">
    आधिकारिक plugins एक ही इंस्टॉल कमांड से Matrix, Nextcloud Talk, Nostr, Twitch, Zalo और दर्जनों अन्य सेवाएँ जोड़ते हैं।
  </Card>
  <Card title="रूटिंग" icon="route" href="/hi/concepts/multi-agent">
    पृथक सत्रों के साथ मल्टी-एजेंट रूटिंग।
  </Card>
  <Card title="मीडिया" icon="image" href="/hi/nodes/images">
    इमेज, ऑडियो, वीडियो, दस्तावेज़ और इमेज/वीडियो जनरेशन।
  </Card>
  <Card title="ऐप्स और UI" icon="monitor" href="/hi/platforms">
    Windows Hub, ब्राउज़र Control UI, macOS मेनू बार ऐप और मोबाइल nodes।
  </Card>
  <Card title="मोबाइल nodes" icon="smartphone" href="/hi/nodes">
    पेयरिंग, वॉइस/चैट और उन्नत डिवाइस कमांड वाले iOS और Android nodes।
  </Card>
</Columns>

## पूरी सूची

**चैनल:**

- iMessage, Telegram और WebChat मुख्य इंस्टॉलेशन के साथ आते हैं; हर अन्य चैनल एक
  आधिकारिक plugin है, जिसे `openclaw plugins install @openclaw/<id>` से इंस्टॉल किया जाता है (या
  `openclaw onboard` / `openclaw channels add` के दौरान आवश्यकता होने पर)
- आधिकारिक plugin चैनल: Discord, Feishu, Google Chat, IRC, LINE, Matrix, Mattermost,
  Microsoft Teams, Nextcloud Talk, Nostr, QQ Bot, Raft, Signal, Slack, SMS, Synology Chat,
  Tlon, Twitch, Voice Call, WhatsApp, Zalo और Zalo Personal
- OpenClaw रिपॉज़िटरी के बाहर अनुरक्षित बाहरी plugin चैनल: WeChat, Yuanbao और Zalo ClawBot
- मेंशन-आधारित सक्रियण के साथ समूह चैट समर्थन
- अनुमति-सूचियों और पेयरिंग के साथ DM सुरक्षा

**एजेंट:**

- टूल स्ट्रीमिंग के साथ एम्बेडेड एजेंट रनटाइम
- प्रत्येक वर्कस्पेस या प्रेषक के लिए पृथक सत्रों के साथ मल्टी-एजेंट रूटिंग
- सत्र: सीधे चैट साझा `main` में समाहित हो जाते हैं; समूह पृथक रहते हैं
- लंबे उत्तरों के लिए स्ट्रीमिंग और खंडों में विभाजन

**प्रमाणीकरण और प्रदाता:**

- 35+ मॉडल प्रदाता (Anthropic, OpenAI, Google और अन्य)
- OAuth के माध्यम से सदस्यता प्रमाणीकरण (जैसे OpenAI Codex)
- कस्टम और स्वयं-होस्ट किए गए प्रदाताओं के लिए समर्थन (vLLM, SGLang, Ollama, llama.cpp, LM Studio और
  कोई भी OpenAI-संगत या Anthropic-संगत एंडपॉइंट)

**मीडिया:**

- इनपुट और आउटपुट में इमेज, ऑडियो, वीडियो और दस्तावेज़
- साझा इमेज जनरेशन और वीडियो जनरेशन क्षमता सतहें
- वॉइस नोट का ट्रांसक्रिप्शन
- एकाधिक प्रदाताओं के साथ टेक्स्ट-टू-स्पीच

**ऐप्स और इंटरफ़ेस:**

- WebChat और ब्राउज़र Control UI
- macOS मेनू बार सहायक ऐप
- पेयरिंग, Canvas, कैमरा, स्क्रीन रिकॉर्डिंग, स्थान और वॉइस वाला iOS node
- पेयरिंग, चैट, वॉइस, Canvas, कैमरा और डिवाइस कमांड वाला Android node

**टूल और स्वचालन:**

- ब्राउज़र स्वचालन, निष्पादन और सैंडबॉक्सिंग
- वेब खोज (Brave, DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Perplexity, SearXNG, Tavily)
- Cron कार्य और Heartbeat शेड्यूलिंग
- Skills, plugins और वर्कफ़्लो पाइपलाइन (Lobster)

## संबंधित

<CardGroup cols={2}>
  <Card title="प्रायोगिक सुविधाएँ" href="/hi/concepts/experimental-features" icon="flask">
    वैकल्पिक सुविधाएँ जो अभी तक डिफ़ॉल्ट सतह पर जारी नहीं हुई हैं।
  </Card>
  <Card title="एजेंट रनटाइम" href="/hi/concepts/agent" icon="robot">
    एजेंट रनटाइम मॉडल और रन भेजने की प्रक्रिया।
  </Card>
  <Card title="चैनल" href="/hi/channels" icon="message-square">
    एक Gateway से Telegram, WhatsApp, Discord, Slack और अन्य सेवाएँ कनेक्ट करें।
  </Card>
  <Card title="Plugins" href="/hi/tools/plugin" icon="plug">
    OpenClaw का विस्तार करने वाले आधिकारिक और बाहरी plugins।
  </Card>
</CardGroup>
