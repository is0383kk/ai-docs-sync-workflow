---
read_when:
    - स्वयं-होस्ट किए गए Synapse या Tuwunel के लिए Matrix शांत स्ट्रीमिंग सेट अप करना
    - उपयोगकर्ता केवल पूर्ण हो चुके ब्लॉकों पर सूचनाएँ चाहते हैं, हर पूर्वावलोकन संपादन पर नहीं
summary: शांत अंतिम रूप दिए गए पूर्वावलोकन संपादनों के लिए प्रति-प्राप्तकर्ता Matrix पुश नियम
title: शांत पूर्वावलोकनों के लिए Matrix पुश नियम
x-i18n:
    generated_at: "2026-07-27T18:55:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1c58e7e796c3ae6d1ee25de229e4592ab8b4fb4d0d50a9cf868ab5ef35b1dab5
    source_path: channels/matrix-push-rules.md
    workflow: 16
---

जब `channels.matrix.streaming.mode` का मान `"quiet"` होता है, तब OpenClaw एक ही पूर्वावलोकन इवेंट को उसी स्थान पर संपादित करके उत्तर स्ट्रीम करता है। पूर्वावलोकन, सूचना न देने वाले `m.notice` इवेंट के रूप में भेजे जाते हैं और अंतिम संपादन को `content["com.openclaw.finalized_preview"] = true` से चिह्नित किया जाता है। Matrix क्लाइंट उस अंतिम संपादन पर केवल तभी सूचना देते हैं, जब प्रति-उपयोगकर्ता पुश नियम उस मार्कर से मेल खाता हो। यह पृष्ठ उन ऑपरेटरों के लिए है जो Matrix को स्वयं होस्ट करते हैं और प्रत्येक प्राप्तकर्ता खाते के लिए वह नियम इंस्टॉल करना चाहते हैं।

`streaming.mode: "progress"` अपने ड्राफ़्ट को उसी पथ से अंतिम रूप देता है, इसलिए वही नियम प्रगति-मोड के अंतिम संपादनों पर भी सक्रिय होता है।

यदि आपको केवल Matrix का मानक सूचना व्यवहार चाहिए, तो `streaming.mode: "partial"` का उपयोग करें या स्ट्रीमिंग बंद रखें। [Matrix चैनल सेटअप](/hi/channels/matrix#streaming-previews) देखें।

## पूर्वापेक्षाएँ

- प्राप्तकर्ता उपयोगकर्ता = वह व्यक्ति जिसे सूचना मिलनी चाहिए
- बॉट उपयोगकर्ता = उत्तर भेजने वाला OpenClaw Matrix खाता
- नीचे दिए गए API कॉल के लिए प्राप्तकर्ता उपयोगकर्ता के एक्सेस टोकन का उपयोग करें
- पुश नियम में `sender` का मिलान बॉट उपयोगकर्ता के पूर्ण MXID से करें
- प्राप्तकर्ता खाते में पहले से कार्यरत पुशर होने चाहिए; शांत पूर्वावलोकन नियम केवल तभी काम करते हैं, जब सामान्य Matrix पुश डिलीवरी ठीक से काम कर रही हो

## चरण

<Steps>
  <Step title="शांत पूर्वावलोकन कॉन्फ़िगर करें">

```json5
{
  channels: {
    matrix: {
      streaming: { mode: "quiet" },
    },
  },
}
```

  </Step>

  <Step title="प्राप्तकर्ता का एक्सेस टोकन प्राप्त करें">
    जहाँ संभव हो, किसी मौजूदा क्लाइंट सत्र टोकन का दोबारा उपयोग करें। नया टोकन बनाने के लिए:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": { "type": "m.id.user", "user": "@alice:example.org" },
    "password": "REDACTED"
  }'
```

  </Step>

  <Step title="सत्यापित करें कि पुशर मौजूद हैं">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

यदि कोई पुशर वापस नहीं मिलता है, तो आगे बढ़ने से पहले इस खाते की सामान्य Matrix पुश डिलीवरी ठीक करें।

  </Step>

  <Step title="ओवरराइड पुश नियम इंस्टॉल करें">
    ऐसा नियम इंस्टॉल करें जो अंतिम-पूर्वावलोकन मार्कर और प्रेषक के रूप में बॉट MXID, दोनों से मेल खाता हो:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

    चलाने से पहले इन्हें बदलें:

    - `https://matrix.example.org`: आपका होमसर्वर आधार URL
    - `$USER_ACCESS_TOKEN`: प्राप्तकर्ता उपयोगकर्ता का एक्सेस टोकन
    - `openclaw-finalized-preview-botname`: प्रत्येक प्राप्तकर्ता के लिए प्रत्येक बॉट हेतु अद्वितीय नियम ID (पैटर्न: `openclaw-finalized-preview-<botname>`)
    - `@bot:example.org`: आपके OpenClaw बॉट का MXID, प्राप्तकर्ता का नहीं

  </Step>

  <Step title="सत्यापित करें">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

फिर एक स्ट्रीम किया गया उत्तर जाँचें। शांत मोड में कक्ष एक शांत ड्राफ़्ट पूर्वावलोकन दिखाता है और ब्लॉक या टर्न समाप्त होने पर सूचना देता है।

  </Step>
</Steps>

बाद में नियम हटाने के लिए, प्राप्तकर्ता के टोकन से उसी नियम URL पर `DELETE` करें।

## एकाधिक बॉट से संबंधित टिप्पणियाँ

पुश नियमों की कुंजी `ruleId` होती है: उसी ID पर `PUT` को दोबारा चलाने से केवल एक नियम अपडेट होता है। एक ही प्राप्तकर्ता को सूचना देने वाले कई OpenClaw बॉट के लिए, अलग प्रेषक मिलान के साथ प्रत्येक बॉट हेतु एक नियम बनाएँ।

नए उपयोगकर्ता-परिभाषित `override` नियम सर्वर-डिफ़ॉल्ट दमन नियमों से पहले जोड़े जाते हैं, इसलिए किसी अतिरिक्त क्रम पैरामीटर की आवश्यकता नहीं होती। यह नियम केवल टेक्स्ट-आधारित पूर्वावलोकन संपादनों को प्रभावित करता है, जिन्हें उसी स्थान पर अंतिम रूप दिया जा सकता है; इसके बजाय मीडिया उत्तर, पुराने पूर्वावलोकन के फ़ॉलबैक और Matrix उल्लेखों को सक्रिय करने वाले अंतिम टेक्स्ट सामान्य सूचना देने वाले संदेशों के रूप में डिलीवर किए जाते हैं।

## होमसर्वर से संबंधित टिप्पणियाँ

<AccordionGroup>
  <Accordion title="Synapse">
    किसी विशेष `homeserver.yaml` बदलाव की आवश्यकता नहीं है। यदि सामान्य Matrix सूचनाएँ पहले से इस उपयोगकर्ता तक पहुँचती हैं, तो ऊपर दिया गया प्राप्तकर्ता टोकन + `pushrules` कॉल मुख्य सेटअप चरण है।

    यदि आप Synapse को किसी रिवर्स प्रॉक्सी या वर्कर के पीछे चलाते हैं, तो सुनिश्चित करें कि `/_matrix/client/.../pushrules/` सही ढंग से Synapse तक पहुँचे। पुश डिलीवरी मुख्य प्रक्रिया या `synapse.app.pusher` / कॉन्फ़िगर किए गए पुशर वर्कर द्वारा संभाली जाती है—सुनिश्चित करें कि वे ठीक से काम कर रहे हैं।

    नियम `event_property_is` पुश-नियम शर्त (MSC3758, पुश नियम v1.10) का उपयोग करता है, जिसे 2023 में Synapse में जोड़ा गया था। Synapse के पुराने रिलीज़ `PUT pushrules/...` कॉल स्वीकार करते हैं, लेकिन शर्त से कभी मेल नहीं खाते—यदि अंतिम पूर्वावलोकन संपादन पर कोई सूचना नहीं आती है, तो Synapse को अपग्रेड करें।

  </Accordion>

  <Accordion title="Tuwunel">
    प्रवाह Synapse जैसा ही है; अंतिम पूर्वावलोकन मार्कर के लिए किसी Tuwunel-विशिष्ट कॉन्फ़िगरेशन की आवश्यकता नहीं है।

    यदि उपयोगकर्ता के किसी अन्य डिवाइस पर सक्रिय रहने के दौरान सूचनाएँ गायब हो जाती हैं, तो जाँचें कि `suppress_push_when_active` सक्षम है या नहीं। Tuwunel ने यह विकल्प 1.4.2 (सितंबर 2025) में जोड़ा था और एक डिवाइस के सक्रिय रहने पर यह जानबूझकर अन्य डिवाइसों के पुश को दबा सकता है।

  </Accordion>
</AccordionGroup>

## संबंधित

- [Matrix चैनल सेटअप](/hi/channels/matrix)
- [स्ट्रीमिंग की अवधारणाएँ](/hi/concepts/streaming)
