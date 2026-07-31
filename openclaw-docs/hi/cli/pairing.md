---
read_when:
    - आप पेयरिंग-मोड DMs का उपयोग कर रहे हैं और आपको प्रेषकों को स्वीकृत करना होगा
summary: '`openclaw pairing` के लिए CLI संदर्भ (पेयरिंग अनुरोधों को स्वीकृत करना/सूचीबद्ध करना)'
title: पेयरिंग
x-i18n:
    generated_at: "2026-07-27T19:28:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e4c6c53f1a3eefe50b4b7a45fa535e9a05faabb50df1ba5195a7635ee13d9da0
    source_path: cli/pairing.md
    workflow: 16
---

# `openclaw pairing`

पेयरिंग का समर्थन करने वाले चैनलों के लिए DM पेयरिंग अनुरोधों को स्वीकृत करें या उनका निरीक्षण करें (केवल चैट DM - नोड/डिवाइस पेयरिंग के लिए `openclaw devices` का उपयोग होता है)।

संबंधित: [पेयरिंग प्रवाह](/hi/channels/pairing)

इन्हीं लंबित अनुरोधों की समीक्षा Control UI में **Settings →
Channels → DM access requests** के अंतर्गत की जा सकती है। Control UI में स्वीकृति, वैकल्पिक
रूप से अनुरोधकर्ता को सूचना भेजने और अनुरोध हटाने की सुविधा है। अनुरोध हटाने से मौजूदा अनुरोध हट जाता है, लेकिन
प्रेषक स्थायी रूप से ब्लॉक नहीं होता।

## कमांड

```bash
openclaw pairing list telegram
openclaw pairing list --channel telegram --account work
openclaw pairing list telegram --json

openclaw pairing approve <code>
openclaw pairing approve telegram <code>
openclaw pairing approve --channel telegram --account work <code> --notify
```

## `pairing list`

किसी एक चैनल के लंबित पेयरिंग अनुरोध सूचीबद्ध करें।

| विकल्प                  | विवरण                           |
| ----------------------- | ------------------------------------- |
| `[channel]`             | स्थितीय चैनल आईडी                 |
| `--channel <channel>`   | स्पष्ट चैनल आईडी                   |
| `--account <accountId>` | एकाधिक खातों वाले चैनलों के लिए खाता आईडी |
| `--json`                | मशीन-पठनीय आउटपुट               |

यदि पेयरिंग की क्षमता वाले कई चैनल कॉन्फ़िगर किए गए हैं, तो चैनल को स्थितीय रूप से या `--channel` के साथ दें। चैनल आईडी मान्य होने पर एक्सटेंशन चैनल भी काम करते हैं।

## `pairing approve`

किसी लंबित पेयरिंग कोड को स्वीकृत करके उस प्रेषक को अनुमति दें।

उपयोग:

- `openclaw pairing approve <channel> <code>`
- `openclaw pairing approve --channel <channel> <code>`
- `openclaw pairing approve <code>` जब पेयरिंग की क्षमता वाला ठीक एक चैनल कॉन्फ़िगर किया गया हो

विकल्प: `--channel <channel>`, `--account <accountId>`, `--notify` (उसी चैनल पर अनुरोधकर्ता को पुष्टि भेजें)।

### स्वामी बूटस्ट्रैप

यदि किसी पेयरिंग कोड को स्वीकृत करते समय `commands.ownerAllowFrom` खाली है, तो CLI स्वीकृत प्रेषक को कमांड स्वामी के रूप में भी दर्ज करता है और इसके लिए `telegram:123456789` जैसी चैनल-स्कोप वाली प्रविष्टि का उपयोग करता है। यह केवल पहले स्वामी को बूटस्ट्रैप करता है - बाद में दी गई पेयरिंग स्वीकृतियाँ कभी भी `commands.ownerAllowFrom` को प्रतिस्थापित या विस्तृत नहीं करतीं। Control UI इस उन्नयन को अपने-आप लागू करने के बजाय एक अलग `operator.admin`-संरक्षित चेकबॉक्स के रूप में प्रस्तुत करता है।

कमांड स्वामी वह मानव ऑपरेटर खाता है जिसे केवल-स्वामी कमांड चलाने और `/diagnostics`, `/export-session`, `/export-trajectory`, `/config` तथा exec स्वीकृतियों जैसी खतरनाक कार्रवाइयों को स्वीकृति देने की अनुमति होती है। पेयरिंग केवल प्रेषक को एजेंट से बात करने देती है; इस एक बार के बूटस्ट्रैप से परे यह अपने-आप स्वामी विशेषाधिकार नहीं देती।

यदि आपने इस बूटस्ट्रैप के उपलब्ध होने से पहले किसी प्रेषक को स्वीकृति दी थी, तो `openclaw doctor` चलाएँ; कोई कमांड स्वामी कॉन्फ़िगर न होने पर यह चेतावनी देता है और उसे ठीक करने के लिए सटीक `openclaw config set commands.ownerAllowFrom ...` कमांड दिखाता है।

## संबंधित

- [CLI संदर्भ](/hi/cli)
- [चैनल पेयरिंग](/hi/channels/pairing)
