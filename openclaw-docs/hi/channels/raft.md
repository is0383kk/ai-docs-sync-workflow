---
read_when:
    - आप OpenClaw को किसी Raft कार्यक्षेत्र से कनेक्ट करना चाहते हैं
    - आप एक Raft External Agent कॉन्फ़िगर कर रहे हैं
    - आप Raft वेक डिलीवरी को डीबग कर रहे हैं
sidebarTitle: Raft
summary: Raft CLI वेक ब्रिज के माध्यम से Raft बाहरी एजेंट समर्थन
title: Raft
x-i18n:
    generated_at: "2026-07-27T19:22:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 454d92d764a4ec3b0ec52467cba254dcad795870e04d1d32d4cf65d8b451a0de
    source_path: channels/raft.md
    workflow: 16
---

Raft स्थानीय Raft CLI के माध्यम से एक OpenClaw एजेंट को Raft External Agent से जोड़ता है। Raft Gateway को प्रमाणीकृत वेक संकेत भेजता है; इसके बाद एजेंट संदेशों की जाँच करने और उन्हें भेजने के लिए Raft CLI का उपयोग करता है। केवल प्रत्यक्ष चैट (कोई समूह नहीं)।

## इंस्टॉल करें

Raft एक आधिकारिक बाहरी plugin है। इसे Gateway होस्ट पर इंस्टॉल करें:

```bash
openclaw plugins install @openclaw/raft
openclaw gateway restart
```

विवरण: [Plugins](/hi/tools/plugin)

## पूर्वापेक्षाएँ

- External Agent वाला एक Raft कार्यस्थान।
- OpenClaw Gateway वाले होस्ट पर ही इंस्टॉल किया गया Raft CLI, सेवा के
  `PATH` पर।
- एक Raft CLI प्रोफ़ाइल, जिसमें पहले से साइन इन किया गया हो और जो उस
  External Agent से संबद्ध हो।

Plugin Raft क्रेडेंशियल संग्रहीत नहीं करता; Raft CLI उस प्रमाणीकरण को अपनी प्रोफ़ाइल में रखता है।

## कॉन्फ़िगर करें

कॉन्फ़िगरेशन में प्रोफ़ाइल सेट करें:

```json5
{
  channels: {
    raft: {
      enabled: true,
      profile: "openclaw",
    },
  },
}
```

डिफ़ॉल्ट खाते के लिए, इसके बजाय Gateway परिवेश में `RAFT_PROFILE` सेट किया जा सकता है:

```bash
RAFT_PROFILE=openclaw
```

जब एक Gateway एक से अधिक Raft External Agent से जुड़ता है, तो नामित खाते का उपयोग करें:

```json5
{
  channels: {
    raft: {
      accounts: {
        support: {
          profile: "support-agent",
        },
        engineering: {
          profile: "engineering-agent",
        },
      },
    },
  },
}
```

इंटरैक्टिव सेटअप उसी प्रोफ़ाइल को रिकॉर्ड करता है:

```bash
openclaw channels add --channel raft
```

## यह कैसे काम करता है

Gateway शुरू होने पर, plugin:

1. एक अस्थायी पोर्ट पर केवल-लूपबैक HTTP वेक एंडपॉइंट खोलता है।
2. उस एंडपॉइंट और प्रति-प्रक्रिया टोकन के साथ `raft --profile <profile> agent bridge` शुरू करता है।
3. स्थानीय ब्रिज से केवल प्रमाणीकृत, सामग्री-रहित और रीप्ले पहचान वाले वेक संकेत स्वीकार करता है।
4. प्रत्येक वेक पेलोड में `eventId`, `attemptId`, `messageId`, `delivery_id`,
   `wake_id`, या `id` में से किसी एक की आवश्यकता होती है।
5. ब्रिज इवेंट आईडी के आधार पर दोबारा भेजी गई वेक डिलीवरी को 24 घंटों तक डीडुप्लिकेट करता है,
   जिसमें Gateway के पुनरारंभ भी शामिल हैं।
6. वर्तमान ब्रिज के लिए एक स्थिर रनटाइम सत्र और Raft CLI प्रोटोकॉल के लिए एक खाली
   गतिविधि-निकासी बैच लौटाता है।
7. प्रत्येक स्वीकृत वेक के लिए एक क्रमबद्ध OpenClaw एजेंट टर्न शुरू करता है।

ब्रिज Raft डिलीवरी के पुनः प्रयासों और पुनः कनेक्शन का प्रबंधन करता है। OpenClaw टर्न को कॉपी किया गया Raft संदेश मुख्य भाग नहीं, केवल वेक सूचना मिलती है। वह लंबित संदेश पढ़ने और अपनी प्रतिक्रिया भेजने के लिए CLI का उपयोग करता है:

```bash
raft --profile openclaw message check
raft --profile openclaw message send
```

<Note>
Raft पुश-संदेश ट्रांसपोर्ट नहीं है। OpenClaw मॉडल के अंतिम टेक्स्ट को ब्रिज के माध्यम से अपने आप वापस नहीं भेजता, इसलिए एजेंट को वेक संसाधित करने के बाद Raft CLI का उपयोग करना आवश्यक है।
</Note>

## सत्यापित करें

जाँचें कि OpenClaw CLI को खोज सकता है और उसमें एक कॉन्फ़िगर की गई प्रोफ़ाइल है:

```bash
openclaw channels status --probe
openclaw plugins inspect raft --runtime --json
```

फिर Raft External Agent को एक संदेश भेजें। Gateway लॉग में पहले Raft ब्रिज का प्रारंभ होना और उसके बाद एक इनबाउंड वेक दिखाई देना चाहिए। एजेंट को अपने लंबित संदेशों की जाँच करने के लिए कॉन्फ़िगर की गई Raft प्रोफ़ाइल का उपयोग करना चाहिए।

## समस्या निवारण

<AccordionGroup>
  <Accordion title="Raft CLI उपलब्ध नहीं है">
    Gateway होस्ट पर Raft CLI इंस्टॉल करें और सेवा के
    `PATH` पर `raft` उपलब्ध कराएँ। इसे `raft --help` से सत्यापित करें, फिर Gateway पुनरारंभ करें।
  </Accordion>
  <Accordion title="ब्रिज तुरंत बंद हो जाता है">
    सत्यापित करें कि कॉन्फ़िगर की गई प्रोफ़ाइल में साइन इन किया गया है और वह अपेक्षित
    Raft External Agent से संबंधित है। CLI निदान देखने के लिए सीधे `raft --profile <profile> agent bridge`
    चलाएँ।
  </Accordion>
  <Accordion title="वेक आता है, लेकिन कोई Raft प्रतिक्रिया नहीं भेजी जाती">
    जब एजेंट Raft CLI का आह्वान नहीं करता, तो यह अपेक्षित है। वेक
    ब्रिज संदेशों के मुख्य भाग या स्वचालित अंतिम उत्तर नहीं ले जाता। एजेंट की
    टूल नीति जाँचें और सुनिश्चित करें कि वह `raft --profile <profile>
    message check` और `message send` चला सकता है।
  </Accordion>
</AccordionGroup>

## संदर्भ

- [Raft](https://raft.build/)
- [Raft दस्तावेज़](https://docs.raft.build/welcome/)
- [Hermes Raft एकीकरण](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/raft)
