---
read_when:
    - आपको नेटवर्क आर्किटेक्चर + सुरक्षा अवलोकन चाहिए
    - आप स्थानीय बनाम टेलनेट पहुँच या पेयरिंग की डीबगिंग कर रहे हैं
    - आप नेटवर्किंग दस्तावेज़ों की प्रामाणिक सूची चाहते हैं
summary: 'नेटवर्क हब: Gateway इंटरफ़ेस, पेयरिंग, खोज और सुरक्षा'
title: नेटवर्क
x-i18n:
    generated_at: "2026-07-27T21:08:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9751bb0fe71009455b243b109ef7ef4eda08d58f940f7dcef305800a5ed89586
    source_path: network.md
    workflow: 16
---

यह हब उन मुख्य दस्तावेज़ों के लिंक प्रदान करता है जो बताते हैं कि OpenClaw localhost, LAN और tailnet पर
डिवाइसों को कैसे कनेक्ट, पेयर और सुरक्षित करता है।

## मुख्य मॉडल

अधिकांश संचालन Gateway (`openclaw gateway`) से होकर गुजरते हैं, जो लंबे समय तक चलने वाली एकल प्रक्रिया है और चैनल कनेक्शन तथा WebSocket नियंत्रण तल का स्वामित्व रखती है।

- **पहले लूपबैक**: Gateway WS का डिफ़ॉल्ट `ws://127.0.0.1:18789` है।
  मान्य Gateway प्रमाणीकरण पथ के बिना गैर-लूपबैक बाइंड प्रारंभ होने से इनकार करते हैं:
  साझा-सीक्रेट टोकन/पासवर्ड प्रमाणीकरण, या सही ढंग से कॉन्फ़िगर किया गया गैर-लूपबैक
  `trusted-proxy` परिनियोजन।
- **प्रति होस्ट एक Gateway** की अनुशंसा की जाती है। पृथक्करण के लिए, अलग-अलग प्रोफ़ाइल और पोर्ट के साथ कई Gateway चलाएँ ([एकाधिक Gateway](/hi/gateway/multiple-gateways))।
- **Canvas होस्ट** को Gateway के समान पोर्ट (`/__openclaw__/canvas/`, `/__openclaw__/a2ui/`) पर सर्व किया जाता है और लूपबैक से परे बाइंड होने पर यह Gateway प्रमाणीकरण द्वारा सुरक्षित रहता है।
- **रिमोट पहुँच** आम तौर पर SSH टनल या Tailscale VPN होती है ([रिमोट पहुँच](/hi/gateway/remote))।

मुख्य संदर्भ:

- [Gateway आर्किटेक्चर](/hi/concepts/architecture)
- [Gateway प्रोटोकॉल](/hi/gateway/protocol)
- [Gateway रनबुक](/hi/gateway)
- [वेब सतहें + बाइंड मोड](/hi/web)

## पेयरिंग + पहचान

- [पेयरिंग का अवलोकन (DM + Node)](/hi/channels/pairing)
- [Gateway के स्वामित्व वाली Node पेयरिंग](/hi/gateway/pairing)
- [डिवाइस CLI (पेयरिंग + टोकन रोटेशन)](/hi/cli/devices)
- [पेयरिंग CLI (DM अनुमोदन)](/hi/cli/pairing)

स्थानीय विश्वास:

- प्रत्यक्ष स्थानीय लूपबैक कनेक्शन (बिना फ़ॉरवर्डेड/प्रॉक्सी हेडर के) को
  समान-होस्ट उपयोगकर्ता अनुभव सुचारु रखने के लिए पेयरिंग हेतु स्वतः अनुमोदित किया जा सकता है।
- OpenClaw में विश्वसनीय साझा-सीक्रेट सहायक प्रवाहों के लिए एक सीमित
  बैकएंड/कंटेनर-स्थानीय स्व-कनेक्शन पथ भी है।
- समान-होस्ट tailnet बाइंड सहित tailnet और LAN क्लाइंट को अब भी
  स्पष्ट पेयरिंग अनुमोदन की आवश्यकता होती है।

## खोज + ट्रांसपोर्ट

- [खोज और ट्रांसपोर्ट](/hi/gateway/discovery)
- [Bonjour / mDNS](/hi/gateway/bonjour)
- [रिमोट पहुँच (SSH)](/hi/gateway/remote)
- [Tailscale](/hi/gateway/tailscale)

## Node + ट्रांसपोर्ट

- [Node का अवलोकन](/hi/nodes)
- [ब्रिज प्रोटोकॉल (पुराने Node, ऐतिहासिक)](/hi/gateway/bridge-protocol)
- [Node रनबुक: iOS](/hi/platforms/ios)
- [Node रनबुक: Android](/hi/platforms/android)

## सुरक्षा

- [सुरक्षा का अवलोकन](/hi/gateway/security)
- [Gateway कॉन्फ़िगरेशन संदर्भ](/hi/gateway/configuration)
- [समस्या निवारण](/hi/gateway/troubleshooting)
- [Doctor](/hi/gateway/doctor)

## संबंधित

- [Gateway रनबुक](/hi/gateway)
- [रिमोट पहुँच](/hi/gateway/remote)
