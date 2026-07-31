---
read_when:
    - बाहरी CLI एकीकरण जोड़ना या बदलना
    - RPC अडैप्टर डीबग करना (signal-cli, imsg)
summary: बाहरी CLI (signal-cli, imsg) के लिए RPC अडैप्टर और Gateway पैटर्न
title: RPC अडैप्टर
x-i18n:
    generated_at: "2026-07-27T20:00:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7deee8154dc824db4eccca9a26381711693972ba2606aec47d657e3724b3a5dd
    source_path: reference/rpc.md
    workflow: 16
---

OpenClaw JSON-RPC के माध्यम से बाहरी CLI को एकीकृत करता है। वर्तमान में दो पैटर्न उपयोग किए जाते हैं।

## पैटर्न A: HTTP डेमन (signal-cli)

- `signal-cli` HTTP पर JSON-RPC के साथ डेमन के रूप में चलता है।
- इवेंट स्ट्रीम SSE है (`/api/v1/events`)।
- स्वास्थ्य जाँच: `/api/v1/check`।
- जब `channels.signal.transport.kind="managed-native"` हो (डिफ़ॉल्ट), तब OpenClaw जीवनचक्र का स्वामी होता है।

सेटअप और एंडपॉइंट के लिए [Signal](/hi/channels/signal) देखें।

## पैटर्न B: stdio चाइल्ड प्रोसेस (imsg)

- OpenClaw [iMessage](/hi/channels/imessage) के लिए `imsg rpc` को चाइल्ड प्रोसेस के रूप में प्रारंभ करता है।
- JSON-RPC stdin/stdout पर पंक्ति-सीमांकित होता है (प्रति पंक्ति एक JSON ऑब्जेक्ट)।
- किसी TCP पोर्ट या डेमन की आवश्यकता नहीं है।

उपयोग की जाने वाली मुख्य विधियाँ:

- `watch.subscribe` → सूचनाएँ (`method: "message"`)
- `watch.unsubscribe`
- `send`
- `chats.list` (जाँच/निदान)

सेटअप और एड्रेसिंग के लिए [iMessage](/hi/channels/imessage) देखें (प्रदर्शन स्ट्रिंग के बजाय `chat_id` को प्राथमिकता दी जाती है)।

## अडैप्टर दिशानिर्देश

- Gateway प्रोसेस का स्वामी होता है (प्रारंभ/समापन प्रोवाइडर के जीवनचक्र से जुड़ा होता है)।
- RPC क्लाइंट को लचीला रखें: टाइमआउट और बाहर निकलने पर पुनः प्रारंभ।
- प्रदर्शन स्ट्रिंग के बजाय स्थिर ID (जैसे, `chat_id`) को प्राथमिकता दें।

## संबंधित

- [Gateway प्रोटोकॉल](/hi/gateway/protocol)
