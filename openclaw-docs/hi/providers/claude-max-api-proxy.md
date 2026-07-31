---
read_when:
    - आप OpenAI-संगत टूल्स के साथ Claude Max सदस्यता का उपयोग करना चाहते हैं
    - आप एक स्थानीय API सर्वर चाहते हैं जो Claude Code CLI को रैप करता हो
    - आप सदस्यता-आधारित बनाम API-की-आधारित Anthropic एक्सेस का मूल्यांकन करना चाहते हैं
summary: Claude सदस्यता क्रेडेंशियल को OpenAI-संगत एंडपॉइंट के रूप में उपलब्ध कराने के लिए सामुदायिक प्रॉक्सी
title: Claude Max API प्रॉक्सी
x-i18n:
    generated_at: "2026-07-27T20:19:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5d0d9a70e14d7d444e57e9bcf169816fec4013a2680dfc9b1761e6ab32109e9f
    source_path: providers/claude-max-api-proxy.md
    workflow: 16
---

**claude-max-api-proxy** एक सामुदायिक npm पैकेज है (OpenClaw plugin नहीं), जो
Claude Max/Pro सदस्यता को OpenAI-संगत API एंडपॉइंट के रूप में उपलब्ध कराता है, ताकि
आप Anthropic API कुंजी के बजाय किसी भी OpenAI-संगत टूल को अपनी सदस्यता पर
निर्देशित कर सकें।

<Warning>
केवल तकनीकी संगतता है, आधिकारिक रूप से स्वीकृत तरीका नहीं। Anthropic ने
पहले Claude Code के बाहर सदस्यता के कुछ उपयोगों को अवरुद्ध किया है; इस पर निर्भर होने से
पहले Anthropic के वर्तमान बिलिंग नियमों की पुष्टि करें।

Anthropic के Claude Code दस्तावेज़ `claude -p` को Agent SDK/प्रोग्रामेटिक
उपयोग के रूप में वर्णित करते हैं। Anthropic के 15 जून, 2026 के सहायता अपडेट के अनुसार, Claude Agent SDK,
`claude -p`, और तृतीय-पक्ष ऐप का उपयोग साइन-इन की गई सदस्यता की
उपयोग सीमाओं में गिना जाता है (पहले घोषित अलग Agent SDK क्रेडिट योजना
स्थगित है)। Anthropic का [Agent SDK योजना
लेख](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan),
[Pro/Max](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
और [Team/Enterprise](https://support.claude.com/en/articles/11845131-use-claude-code-with-your-team-or-enterprise-plan)
योजना लेख, तथा OpenClaw के
अपने Claude CLI बिलिंग नोट्स के लिए [Anthropic प्रदाता](/hi/providers/anthropic) देखें।
</Warning>

## इसका उपयोग क्यों करें

| तरीका                    | लागत का माध्यम                                      | इसके लिए सर्वोत्तम                              |
| ------------------------- | ----------------------------------------------- | ------------------------------------------ |
| Anthropic API कुंजी         | Claude Console के माध्यम से प्रति टोकन भुगतान            | प्रोडक्शन ऐप, साझा ऑटोमेशन, अधिक मात्रा |
| Claude सदस्यता प्रॉक्सी | Claude Code / `claude -p` योजना और क्रेडिट नियम | संगत टूल के साथ व्यक्तिगत प्रयोग |

यह प्रॉक्सी Claude Max या Pro सदस्यता को OpenAI-संगत
टूल के साथ काम करने देती है। यह असीमित एकमुश्त-दर वाला तरीका नहीं है—इस पर Claude Code की उपयोग
सीमाएँ लागू होती हैं। प्रोडक्शन उपयोग के लिए API कुंजियाँ अब भी अधिक स्पष्ट बिलिंग माध्यम हैं।

## यह कैसे काम करता है

```text
आपका ऐप -> claude-max-api-proxy -> Claude Code CLI / claude -p -> Anthropic
     (OpenAI प्रारूप)                (प्रारूप बदलता है)              (आपके लॉगिन का उपयोग करता है)
```

प्रॉक्सी प्रत्येक अनुरोध के लिए Claude Code CLI को एक सबप्रोसेस के रूप में शुरू करती है,
OpenAI-प्रारूप के चैट अनुरोधों को CLI प्रॉम्प्ट में बदलती है और प्रतिक्रिया को
OpenAI प्रारूप में वापस स्ट्रीम करती है (या लौटाती है)।

## आरंभ करना

<Steps>
  <Step title="प्रॉक्सी इंस्टॉल करें">
    Node.js 20+ और प्रमाणीकृत Claude Code CLI आवश्यक हैं।

    ```bash
    npm install -g claude-max-api-proxy

    # पुष्टि करें कि Claude CLI प्रमाणीकृत है
    claude --version
    claude auth login   # यदि पहले से प्रमाणीकृत नहीं है
    ```

  </Step>
  <Step title="सर्वर शुरू करें">
    ```bash
    claude-max-api
    # सर्वर http://localhost:3456 पर चलता है
    ```
  </Step>
  <Step title="प्रॉक्सी का परीक्षण करें">
    ```bash
    curl http://localhost:3456/health
    curl http://localhost:3456/v1/models

    curl http://localhost:3456/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d '{
        "model": "claude-opus-4",
        "messages": [{"role": "user", "content": "नमस्ते!"}]
      }'
    ```

  </Step>
  <Step title="OpenClaw कॉन्फ़िगर करें">
    OpenClaw को कस्टम OpenAI-संगत एंडपॉइंट के रूप में प्रॉक्सी पर निर्देशित करें:

    ```json5
    {
      env: {
        OPENAI_API_KEY: "not-needed",
        OPENAI_BASE_URL: "http://localhost:3456/v1",
      },
      agents: {
        defaults: {
          model: { primary: "openai/claude-opus-4" },
        },
      },
    }
    ```

  </Step>
</Steps>

<Note>
नीचे दिए गए मॉडल आईडी प्रॉक्सी की अपनी सूची से हैं, OpenClaw के Anthropic
मॉडल संदर्भ नहीं। प्रत्येक आईडी Claude Code CLI मॉडल उपनाम (`opus`, `sonnet`,
`haiku`) से मैप होता है, इसलिए जब भी Anthropic CLI में उस
उपनाम को अपडेट करता है, अंतर्निहित मॉडल बदल जाता है। किसी विशिष्ट मैपिंग पर निर्भर होने से
पहले प्रॉक्सी का वर्तमान README देखें।
</Note>

| मॉडल आईडी          | CLI उपनाम | वर्तमान मैपिंग |
| ----------------- | --------- | --------------- |
| `claude-opus-4`   | `opus`    | Claude Opus 4.5 |
| `claude-sonnet-4` | `sonnet`  | Claude Sonnet 4 |
| `claude-haiku-4`  | `haiku`   | Claude Haiku 4  |

## उन्नत कॉन्फ़िगरेशन

<AccordionGroup>
  <Accordion title="प्रॉक्सी-शैली OpenAI-संगत नोट्स">
    यह OpenClaw के सामान्य कस्टम `/v1` OpenAI-संगत मार्ग का उपयोग करता है, जो किसी भी
    अन्य स्वयं-होस्ट किए गए OpenAI-संगत बैकएंड के समान मार्ग है:

    - केवल मूल OpenAI अनुरोध आकार निर्धारण लागू नहीं होता।
    - `/fast` और `service_tier` केवल सीधे `api.anthropic.com`
      ट्रैफ़िक पर लागू होते हैं; प्रॉक्सी मार्ग `service_tier` को अपरिवर्तित छोड़ते हैं (देखें
      [Anthropic प्रदाता फ़ास्ट मोड](/hi/providers/anthropic#advanced-configuration))।
    - कोई Responses `store`, प्रॉम्प्ट-कैश संकेत या OpenAI रीजनिंग-संगत
      पेलोड आकार निर्धारण नहीं।
    - OpenClaw के OpenAI/Codex एट्रिब्यूशन हेडर (`originator`, `version`,
      `User-Agent`) केवल मूल `api.openai.com` OAuth ट्रैफ़िक पर भेजे जाते हैं, इस प्रॉक्सी जैसे
      कस्टम `OPENAI_BASE_URL` लक्ष्यों पर नहीं।

  </Accordion>

  <Accordion title="LaunchAgent के साथ macOS पर स्वतः शुरू करें">
    ```bash
    cat > ~/Library/LaunchAgents/com.claude-max-api.plist << 'EOF'
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
      <key>Label</key>
      <string>com.claude-max-api</string>
      <key>RunAtLoad</key>
      <true/>
      <key>KeepAlive</key>
      <true/>
      <key>ProgramArguments</key>
      <array>
        <string>/usr/local/bin/node</string>
        <string>/usr/local/lib/node_modules/claude-max-api-proxy/dist/server/standalone.js</string>
      </array>
      <key>EnvironmentVariables</key>
      <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/opt/homebrew/bin:~/.local/bin:/usr/bin:/bin</string>
      </dict>
    </dict>
    </plist>
    EOF

    launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude-max-api.plist
    ```

  </Accordion>
</AccordionGroup>

## नोट्स

- Claude Code के `claude -p` बिलिंग, उपयोग-क्रेडिट और दर-सीमा व्यवहार को प्राप्त करता है।
- केवल `127.0.0.1` से बाइंड होता है; CLI द्वारा Anthropic को किए गए अपने कॉल के अलावा किसी तृतीय-पक्ष सर्वर को डेटा नहीं भेजता।
- स्ट्रीमिंग प्रतिक्रियाएँ समर्थित हैं।
- प्रमाणीकरण विफलताओं की स्टार्टअप पर जाँच नहीं होती और वे केवल चैट अनुरोध के वास्तव में चलने पर सामने आती हैं; यदि CLI प्रमाणीकृत नहीं है, तो सर्वर द्वारा शुरू होने से इनकार करने के बजाय पहला अनुरोध विफल होगा।

<Note>
Claude CLI या API कुंजियों के साथ मूल Anthropic एकीकरण के लिए [Anthropic प्रदाता](/hi/providers/anthropic) देखें। OpenAI/Codex सदस्यताओं के लिए [OpenAI प्रदाता](/hi/providers/openai) देखें।
</Note>

## संबंधित

<CardGroup cols={2}>
  <Card title="Anthropic प्रदाता" href="/hi/providers/anthropic" icon="bolt">
    Claude CLI या API कुंजियों के साथ मूल OpenClaw एकीकरण।
  </Card>
  <Card title="OpenAI प्रदाता" href="/hi/providers/openai" icon="robot">
    OpenAI/Codex सदस्यताओं के लिए।
  </Card>
  <Card title="मॉडल चयन" href="/hi/concepts/model-providers" icon="layers">
    सभी प्रदाताओं, मॉडल संदर्भों और फ़ेलओवर व्यवहार का अवलोकन।
  </Card>
  <Card title="कॉन्फ़िगरेशन" href="/hi/gateway/configuration" icon="gear">
    पूर्ण कॉन्फ़िगरेशन संदर्भ।
  </Card>
</CardGroup>
