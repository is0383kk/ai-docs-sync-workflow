---
read_when:
    - डैशबोर्ड प्रमाणीकरण या एक्सपोज़र मोड बदलना
summary: Gateway डैशबोर्ड (नियंत्रण UI) की पहुँच और प्रमाणीकरण
title: डैशबोर्ड
x-i18n:
    generated_at: "2026-07-27T21:52:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ca531ad2943dfdee1cd90a4efdc1fb69c4517780e2be52237fd558b8638e7cd0
    source_path: web/dashboard.md
    workflow: 16
---

Gateway डैशबोर्ड ब्राउज़र Control UI है, जो डिफ़ॉल्ट रूप से `/` पर उपलब्ध होता है (`gateway.controlUi.basePath` से ओवरराइड करें)।

तुरंत खोलें (स्थानीय Gateway):

- [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (या [http://localhost:18789/](http://localhost:18789/))
- `gateway.tls.enabled: true` के साथ, WebSocket एंडपॉइंट के लिए `https://127.0.0.1:18789/` और `wss://127.0.0.1:18789` का उपयोग करें।

मुख्य संदर्भ:

- उपयोग और UI क्षमताओं के लिए [Control UI](/hi/web/control-ui)।
- Serve/Funnel ऑटोमेशन के लिए [Tailscale](/hi/gateway/tailscale)।
- बाइंड मोड और सुरक्षा संबंधी टिप्पणियों के लिए [वेब सतहें](/hi/web)।

कॉन्फ़िगर किए गए Gateway प्रमाणीकरण पथ के माध्यम से WebSocket हैंडशेक पर प्रमाणीकरण लागू किया जाता है:

- `connect.params.auth.token`
- `connect.params.auth.password`
- `gateway.auth.allowTailscale: true` होने पर Tailscale Serve पहचान हेडर
- `gateway.auth.mode: "trusted-proxy"` होने पर विश्वसनीय-प्रॉक्सी पहचान हेडर

[Gateway कॉन्फ़िगरेशन](/hi/gateway/configuration) में `gateway.auth` देखें।

<Warning>
Control UI एक **व्यवस्थापकीय सतह** है (चैट, कॉन्फ़िगरेशन, निष्पादन स्वीकृतियाँ)। इसे सार्वजनिक रूप से उजागर न करें। UI वर्तमान ब्राउज़र टैब और चयनित Gateway URL के लिए डैशबोर्ड URL टोकन sessionStorage में रखता है और लोड होने के बाद उन्हें URL से हटा देता है। localhost, Tailscale Serve या SSH टनल को प्राथमिकता दें।
</Warning>

## त्वरित तरीका (अनुशंसित)

- ऑनबोर्डिंग के बाद, CLI डैशबोर्ड को अपने-आप खोलता है और एक साफ़ (बिना टोकन वाला) लिंक प्रिंट करता है।
- कभी भी दोबारा खोलें: `openclaw dashboard` (लिंक कॉपी करता है, संभव होने पर ब्राउज़र खोलता है और हेडलेस होने पर SSH संकेत प्रिंट करता है)।
- यदि क्लिपबोर्ड और ब्राउज़र दोनों के माध्यम से पहुँचाना विफल हो जाए, तब भी `openclaw dashboard` साफ़ URL प्रिंट करता है और आपको अपना टोकन (`OPENCLAW_GATEWAY_TOKEN` या `gateway.auth.token` से) URL फ़्रैगमेंट कुंजी `token` के रूप में जोड़ने के लिए कहता है; यह लॉग में टोकन मान कभी प्रिंट नहीं करता।
- यदि UI साझा-गोपनीय प्रमाणीकरण के लिए संकेत दे, तो कॉन्फ़िगर किया गया टोकन या पासवर्ड Control UI सेटिंग्स में पेस्ट करें।

## प्रमाणीकरण की मूल बातें (स्थानीय बनाम रिमोट)

- **Localhost**: `http://127.0.0.1:18789/` खोलें।
- **Gateway TLS**: `gateway.tls.enabled: true` होने पर, डैशबोर्ड/स्थिति लिंक `https://` और Control UI WebSocket लिंक `wss://` का उपयोग करते हैं।
- **साझा-गोपनीय टोकन स्रोत**: `gateway.auth.token` (या `OPENCLAW_GATEWAY_TOKEN`)। `openclaw dashboard` इसे एक बार की बूटस्ट्रैप प्रक्रिया के लिए URL फ़्रैगमेंट के माध्यम से भेज सकता है; Control UI इसे वर्तमान टैब और चयनित Gateway URL के लिए sessionStorage में रखता है, localStorage में नहीं।
- **कॉन्फ़िगरेशन न होने पर रनटाइम टोकन**: यदि स्टार्टअप बताता है कि उसने रनटाइम टोकन जनरेट किया है, तो वह टोकन अस्थायी है और `openclaw config get gateway.auth.token` के माध्यम से उपलब्ध नहीं होता। लूपबैक के लिए भी प्रमाणीकरण आवश्यक है। `openclaw doctor --generate-gateway-token` चलाएँ, Gateway को पुनः आरंभ करें और फिर कॉन्फ़िगर किया गया टोकन Control UI सेटिंग्स में पेस्ट करें।
- यदि `gateway.auth.token` को SecretRef से प्रबंधित किया जाता है, तो बाहरी रूप से प्रबंधित टोकन को शेल लॉग, क्लिपबोर्ड इतिहास या ब्राउज़र लॉन्च आर्ग्युमेंट में उजागर होने से बचाने के लिए `openclaw dashboard` जानबूझकर बिना टोकन वाला URL प्रिंट/कॉपी/खोलता है। यदि आपके वर्तमान शेल में रेफ़रेंस हल नहीं होता, तब भी यह बिना टोकन वाला URL और प्रमाणीकरण सेटअप के लिए कार्रवाई योग्य मार्गदर्शन प्रिंट करता है।
- **साझा-गोपनीय पासवर्ड**: कॉन्फ़िगर किए गए `gateway.auth.password` (या `OPENCLAW_GATEWAY_PASSWORD`) का उपयोग करें। डैशबोर्ड पुनः लोड होने के बाद पासवर्ड बनाए नहीं रखता।
- **पहचान-युक्त मोड**: `gateway.auth.allowTailscale: true` होने पर Tailscale Serve पहचान हेडर के माध्यम से Control UI/WebSocket प्रमाणीकरण पूरा करता है; एक गैर-लूपबैक, पहचान-सक्षम रिवर्स प्रॉक्सी `gateway.auth.mode: "trusted-proxy"` को पूरा करता है। दोनों में से किसी को भी WebSocket के लिए साझा गोपनीय मान पेस्ट करने की आवश्यकता नहीं होती।
- **localhost नहीं**: Tailscale Serve, गैर-लूपबैक साझा-गोपनीय बाइंड, `gateway.auth.mode: "trusted-proxy"` वाला गैर-लूपबैक पहचान-सक्षम रिवर्स प्रॉक्सी या SSH टनल का उपयोग करें। HTTP API अब भी साझा-गोपनीय प्रमाणीकरण का उपयोग करते हैं, जब तक कि आप जानबूझकर निजी-इनग्रेस `gateway.auth.mode: "none"` या विश्वसनीय-प्रॉक्सी HTTP प्रमाणीकरण न चलाएँ। [वेब सतहें](/hi/web) देखें।

## Telegram में खोलें

Telegram बॉट `/dashboard` के साथ डैशबोर्ड को Telegram Mini App के रूप में खोल सकते हैं।

आवश्यकताएँ:

- `gateway.tailscale.mode: "serve"` या `"funnel"`, ताकि Telegram को HTTPS Mini App URL मिले।
- Telegram प्रेषक बॉट का स्वामी होना चाहिए: `commands.ownerAllowFrom` में संख्यात्मक Telegram उपयोगकर्ता ID या चयनित खाते का प्रभावी `channels.telegram.allowFrom`।
- बॉट के साथ DM में `/dashboard` चलाएँ। समूह में इसे चलाने पर केवल DM में कमांड खोलने के लिए कहा जाता है और कोई बटन शामिल नहीं होता।
- Docker इंस्टॉलेशन: Serve/Funnel मोड के लिए Gateway को `tailscaled` के पास लूपबैक से बाइंड करना आवश्यक है, जिसे प्रकाशित पोर्ट वाली ब्रिज नेटवर्किंग पूरा नहीं कर सकती। Gateway कंटेनर को `network_mode: host` के साथ चलाएँ और होस्ट `tailscaled` सॉकेट (`/var/run/tailscale`) तथा `tailscale` CLI को कंटेनर में माउंट करें।

Mini App स्वामी का एक बार हस्तांतरण करता है और अल्पकालिक बूटस्ट्रैप टोकन के साथ Control UI पर रीडायरेक्ट करता है। यह URL में साझा Gateway टोकन उजागर नहीं करता।

v1 के गैर-लक्ष्य:

- Telegram Web iframe समर्थित नहीं है।
- Tailscale Serve/Funnel ही एकमात्र समर्थित प्रकाशित URL पथ है।

<a id="if-you-see-unauthorized-1008"></a>

## यदि आपको "unauthorized" / 1008 दिखाई दे

- पुष्टि करें कि Gateway तक पहुँचा जा सकता है: स्थानीय रूप से `openclaw status`; रिमोट के लिए, SSH टनल `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host`, फिर `http://127.0.0.1:18789/` खोलें।
- `AUTH_TOKEN_MISMATCH` के लिए, Gateway द्वारा पुनः प्रयास के संकेत लौटाने पर क्लाइंट कैश किए गए डिवाइस टोकन के साथ एक विश्वसनीय पुनः प्रयास कर सकते हैं; यह पुनः प्रयास टोकन के कैश किए गए स्वीकृत स्कोप का दोबारा उपयोग करता है (स्पष्ट `deviceToken`/`scopes` कॉलर अपने अनुरोधित स्कोप सेट को बनाए रखते हैं)। यदि उस पुनः प्रयास के बाद भी प्रमाणीकरण विफल होता है, तो टोकन विसंगति को मैन्युअल रूप से हल करें।
- `AUTH_SCOPE_MISMATCH` के लिए, डिवाइस टोकन पहचाना गया था, लेकिन उसमें अनुरोधित स्कोप नहीं हैं; साझा Gateway टोकन बदलने के बजाय दोबारा पेयर करें या नए स्कोप सेट को स्वीकृति दें।
- उस पुनः प्रयास पथ के बाहर, कनेक्शन प्रमाणीकरण की प्राथमिकता है: स्पष्ट साझा टोकन/पासवर्ड, फिर स्पष्ट `deviceToken`, फिर संग्रहीत डिवाइस टोकन, फिर बूटस्ट्रैप टोकन।
- असिंक्रोनस Tailscale Serve पथ पर, समान `{scope, ip}` के विफल प्रयासों को विफल-प्रमाणीकरण सीमक द्वारा दर्ज किए जाने से पहले क्रमबद्ध किया जाता है, इसलिए दूसरा समकालिक गलत पुनः प्रयास पहले से ही `retry later` दिखा सकता है।
- टोकन विसंगति सुधारने के चरणों के लिए, [टोकन विसंगति पुनर्प्राप्ति चेकलिस्ट](/hi/cli/devices#token-drift-recovery-checklist) देखें।
- Gateway होस्ट से साझा गोपनीय मान प्राप्त करें या उपलब्ध कराएँ:
  - टोकन: `openclaw config get gateway.auth.token`
  - पासवर्ड: कॉन्फ़िगर किए गए `gateway.auth.password` या `OPENCLAW_GATEWAY_PASSWORD` को हल करें
  - SecretRef-प्रबंधित टोकन: बाहरी गोपनीय मान प्रदाता को हल करें, या इस शेल में `OPENCLAW_GATEWAY_TOKEN` एक्सपोर्ट करें और `openclaw dashboard` दोबारा चलाएँ
  - कोई साझा गोपनीय मान कॉन्फ़िगर न होने के कारण जनरेट किया गया रनटाइम टोकन: `openclaw doctor --generate-gateway-token` चलाएँ, Gateway को पुनः आरंभ करें, फिर कॉन्फ़िगर किए गए टोकन का उपयोग करें
- डैशबोर्ड सेटिंग्स में, टोकन या पासवर्ड को प्रमाणीकरण फ़ील्ड में पेस्ट करें, फिर कनेक्ट करें।
- UI भाषा चयनकर्ता **Settings -> General -> Language** में होता है, Appearance के अंतर्गत नहीं।

## संबंधित

- [Control UI](/hi/web/control-ui)
- [WebChat](/hi/web/webchat)
