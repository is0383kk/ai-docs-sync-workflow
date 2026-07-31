---
read_when:
    - पहचान-जागरूक प्रॉक्सी के पीछे OpenClaw चलाना
    - OpenClaw के आगे OAuth के साथ Pomerium, Caddy या nginx सेट अप करना
    - रिवर्स प्रॉक्सी सेटअप में WebSocket 1008 अनधिकृत त्रुटियों को ठीक करना
    - HSTS और अन्य HTTP सुरक्षा-सुदृढ़ीकरण हेडर कहाँ सेट किए जाएँ, इसका निर्णय लेना
sidebarTitle: Trusted proxy auth
summary: Gateway प्रमाणीकरण को किसी विश्वसनीय रिवर्स प्रॉक्सी (Pomerium, Caddy, nginx + OAuth) को सौंपें
title: विश्वसनीय प्रॉक्सी प्रमाणीकरण
x-i18n:
    generated_at: "2026-07-27T17:53:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39bf8f12b3ae95f53b21bfed12deb1c8ed8f767711955bbee52c74538052a89f
    source_path: gateway/trusted-proxy-auth.md
    workflow: 16
---

<Warning>
**सुरक्षा-संवेदनशील सुविधा।** यह मोड प्रमाणीकरण पूरी तरह आपके रिवर्स प्रॉक्सी को सौंप देता है। गलत कॉन्फ़िगरेशन आपके Gateway को अनधिकृत पहुँच के लिए उजागर कर सकता है। सक्षम करने से पहले इस पृष्ठ को ध्यान से पढ़ें।
</Warning>

## कब उपयोग करें

- आप OpenClaw को किसी **पहचान-जागरूक प्रॉक्सी** (Pomerium, Caddy + OAuth, nginx + oauth2-proxy, Traefik + forward auth) के पीछे चलाते हैं।
- आपका प्रॉक्सी सभी प्रमाणीकरण संभालता है और हेडर के माध्यम से उपयोगकर्ता की पहचान भेजता है।
- आप किसी Kubernetes या कंटेनर परिवेश में हैं, जहाँ Gateway तक पहुँचने का एकमात्र मार्ग प्रॉक्सी है।
- आपको WebSocket `1008 unauthorized` त्रुटियाँ मिल रही हैं क्योंकि ब्राउज़र WS पेलोड में टोकन नहीं भेज सकते।

## कब उपयोग न करें

- आपका प्रॉक्सी उपयोगकर्ताओं को प्रमाणित नहीं करता (यह केवल TLS टर्मिनेटर या लोड बैलेंसर है)।
- Gateway तक पहुँचने का कोई भी ऐसा मार्ग मौजूद है जो प्रॉक्सी को बायपास करता है (फ़ायरवॉल में छिद्र, आंतरिक नेटवर्क पहुँच)।
- आप सुनिश्चित नहीं हैं कि आपका प्रॉक्सी फ़ॉरवर्ड किए गए हेडर को सही ढंग से हटाता/ओवरराइट करता है।
- आपको केवल व्यक्तिगत एकल-उपयोगकर्ता पहुँच चाहिए (इसके बजाय Tailscale Serve + loopback पर विचार करें)।

## यह कैसे काम करता है

<Steps>
  <Step title="प्रॉक्सी उपयोगकर्ता को प्रमाणित करता है">
    आपका रिवर्स प्रॉक्सी उपयोगकर्ताओं को प्रमाणित करता है (OAuth, OIDC, SAML आदि)।
  </Step>
  <Step title="प्रॉक्सी पहचान हेडर जोड़ता है">
    प्रॉक्सी प्रमाणित उपयोगकर्ता की पहचान वाला हेडर जोड़ता है (उदाहरण के लिए, `x-forwarded-user: nick@example.com`)।
  </Step>
  <Step title="Gateway विश्वसनीय स्रोत सत्यापित करता है">
    OpenClaw जाँचता है कि अनुरोध किसी **विश्वसनीय प्रॉक्सी IP** (`gateway.trustedProxies`) से आया है और वह Gateway का अपना loopback या स्थानीय इंटरफ़ेस पता नहीं है।
  </Step>
  <Step title="Gateway पहचान निकालता है">
    OpenClaw आवश्यक हेडर पढ़ता है, फिर कॉन्फ़िगर किए गए हेडर से उपयोगकर्ता की पहचान निकालता है।
  </Step>
  <Step title="प्राधिकृत करना">
    यदि सभी जाँच सफल होती हैं और उपयोगकर्ता `allowUsers` (सेट होने पर) को पूरा करता है, तो अनुरोध प्राधिकृत कर दिया जाता है।
  </Step>
</Steps>

## कॉन्फ़िगरेशन

```json5
{
  gateway: {
    // विश्वसनीय-प्रॉक्सी प्रमाणीकरण डिफ़ॉल्ट रूप से अपेक्षा करता है कि प्रॉक्सी का स्रोत IP non-loopback हो
    bind: "lan",

    // अत्यंत महत्वपूर्ण: यहाँ केवल अपने प्रॉक्सी के IP जोड़ें
    trustedProxies: ["10.0.0.1", "172.17.0.1"],

    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        // प्रमाणित उपयोगकर्ता पहचान वाला हेडर (आवश्यक)
        userHeader: "x-forwarded-user",

        // वैकल्पिक: वे हेडर जिनका मौजूद होना आवश्यक है (प्रॉक्सी सत्यापन)
        requiredHeaders: ["x-forwarded-proto", "x-forwarded-host"],

        // वैकल्पिक: विशिष्ट उपयोगकर्ताओं तक सीमित करें (खाली = सभी को अनुमति दें)
        allowUsers: ["nick@example.com", "admin@company.org"],

        // वैकल्पिक: स्पष्ट रूप से स्वीकार करने के बाद समान-होस्ट loopback प्रॉक्सी को अनुमति दें
        allowLoopback: false,

        // वैकल्पिक: प्रमाणित प्रॉक्सी उपयोगकर्ताओं को नए ब्राउज़र डिवाइस नामांकित करने दें
        deviceAutoApprove: {
          enabled: false,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

<Warning>
**रनटाइम नियम, मूल्यांकन के क्रम में**

1. अनुरोध का स्रोत IP `gateway.trustedProxies` से मेल खाना चाहिए (CIDR-जागरूक), अन्यथा उसे अस्वीकार कर दिया जाता है (`trusted_proxy_untrusted_source`)।
2. Loopback-स्रोत अनुरोध (`127.0.0.1`, `::1`) तब तक अस्वीकार किए जाते हैं जब तक `gateway.auth.trustedProxy.allowLoopback = true` न हो और loopback पता भी `trustedProxies` में शामिल न हो (`trusted_proxy_loopback_source`)। यह जाँच हेडर जाँचों से पहले चलती है, इसलिए आवश्यक हेडर भी अनुपस्थित होने पर loopback स्रोत इसी प्रकार विफल होता है।
3. Gateway होस्ट के अपने स्थानीय नेटवर्क इंटरफ़ेस पतों में से किसी एक से मेल खाने वाले non-loopback स्रोतों को स्पूफ़िंग से सुरक्षा के लिए अस्वीकार कर दिया जाता है (`trusted_proxy_local_interface_source`)। यदि इंटरफ़ेस खोज स्वयं विफल होती है, तो अनुरोध भी अस्वीकार कर दिया जाता है (`trusted_proxy_local_interface_check_failed`)।
4. `requiredHeaders` और `userHeader` मौजूद और गैर-रिक्त होने चाहिए।
5. `allowUsers`, यदि गैर-रिक्त हो, तो उसमें निकाला गया उपयोगकर्ता शामिल होना चाहिए।

**फ़ॉरवर्ड किए गए हेडर के प्रमाण स्थानीय-प्रत्यक्ष फ़ॉलबैक के लिए loopback स्थानीयता को ओवरराइड करते हैं।** यदि कोई अनुरोध loopback पर आता है लेकिन उसमें `Forwarded`, कोई भी `X-Forwarded-*`, या `X-Real-IP` हेडर मौजूद है, तो वह प्रमाण उसे स्थानीय-प्रत्यक्ष पासवर्ड फ़ॉलबैक और डिवाइस-पहचान गेटिंग के लिए अयोग्य ठहराता है, भले ही वह loopback होने के कारण विश्वसनीय-प्रॉक्सी प्रमाणीकरण में फिर भी विफल हो।

`allowLoopback` Gateway होस्ट की स्थानीय प्रक्रियाओं पर रिवर्स प्रॉक्सी के समान स्तर तक भरोसा करता है। इसे केवल तभी सक्षम करें जब Gateway अभी भी प्रत्यक्ष दूरस्थ पहुँच से फ़ायरवॉल द्वारा सुरक्षित हो और स्थानीय प्रॉक्सी क्लाइंट द्वारा भेजे गए पहचान हेडर को हटाता या ओवरराइट करता हो।

जो आंतरिक Gateway क्लाइंट रिवर्स प्रॉक्सी से होकर नहीं जाते, उन्हें विश्वसनीय-प्रॉक्सी पहचान हेडर के बजाय `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` का उपयोग करना चाहिए। Non-loopback Control UI परिनियोजनों को अब भी स्पष्ट `gateway.controlUi.allowedOrigins` की आवश्यकता होती है।
</Warning>

### कॉन्फ़िगरेशन संदर्भ

<ParamField path="gateway.trustedProxies" type="string[]" required>
  विश्वास करने के लिए प्रॉक्सी IP पतों (या CIDR) की सरणी। अन्य IP से आए अनुरोध अस्वीकार कर दिए जाते हैं।
</ParamField>
<ParamField path="gateway.auth.mode" type="string" required>
  `"trusted-proxy"` होना आवश्यक है।
</ParamField>
<ParamField path="gateway.auth.trustedProxy.userHeader" type="string" required>
  प्रमाणित उपयोगकर्ता की पहचान वाला हेडर नाम।
</ParamField>
<ParamField path="gateway.auth.trustedProxy.requiredHeaders" type="string[]">
  अनुरोध पर विश्वास करने के लिए मौजूद होने वाले अतिरिक्त हेडर।
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowUsers" type="string[]">
  उपयोगकर्ता पहचानों की अनुमतिसूची। खाली होने का अर्थ सभी प्रमाणित उपयोगकर्ताओं को अनुमति देना है।
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowLoopback" type="boolean" default="false">
  समान-होस्ट loopback रिवर्स प्रॉक्सी के लिए स्पष्ट रूप से स्वीकार की जाने वाली सहायता।
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.enabled" type="boolean" default="false">
  विश्वसनीय-प्रॉक्सी प्रमाणीकरण के बाद नई Control UI और WebChat डिवाइस पहचानों को स्वतः स्वीकृत करें।
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.scopes" type="string[]" default='["operator.read", "operator.write", "operator.approvals"]'>
  स्वतः स्वीकृत ब्राउज़र डिवाइस को दिए जाने वाले अधिकतम स्कोप। `operator.admin` को स्पष्ट रूप से सूचीबद्ध करने पर प्रत्येक प्रॉक्सी-प्रमाणित उपयोगकर्ता स्वतः पूर्ण-एडमिन डिवाइस अनुदान का अनुरोध कर सकता है, बिना स्कोप वाले अनुरोधों को स्वतः पूर्ण एडमिन मिलता है, और CRITICAL `gateway.trusted_proxy_device_auto_approve_admin` सुरक्षा ऑडिट निष्कर्ष के साथ Gateway स्टार्टअप चेतावनी ट्रिगर होती है।
</ParamField>

<Warning>
`allowLoopback` को केवल तभी सक्षम करें जब स्थानीय रिवर्स प्रॉक्सी ही अभिप्रेत विश्वास सीमा हो। Gateway से कनेक्ट हो सकने वाली कोई भी स्थानीय प्रक्रिया प्रॉक्सी पहचान हेडर भेजने का प्रयास कर सकती है, इसलिए प्रत्यक्ष Gateway पहुँच को होस्ट तक निजी रखें और `x-forwarded-proto` जैसे प्रॉक्सी-स्वामित्व वाले हेडर, या जहाँ आपका प्रॉक्सी समर्थन करता हो वहाँ हस्ताक्षरित अभिकथन हेडर आवश्यक बनाएँ।
</Warning>

## स्वचालित डिवाइस स्वीकृति

विश्वसनीय-प्रॉक्सी प्रमाणीकरण वैकल्पिक रूप से नए ब्राउज़र डिवाइसों के लिए प्रॉक्सी पहचान को स्वीकृति सीमा के रूप में उपयोग कर सकता है:

```json5
{
  gateway: {
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-forwarded-user",
        allowUsers: ["operator@example.com"],
        deviceAutoApprove: {
          enabled: true,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

डिफ़ॉल्ट `enabled: false` है। सक्षम होने पर ये सभी नियम लागू होते हैं:

1. WebSocket को किसी गैर-रिक्त उपयोगकर्ता पहचान के साथ `trusted-proxy` विधि के माध्यम से प्रमाणित होना चाहिए, और अनुमतिसूची कॉन्फ़िगर होने पर उस पहचान को `allowUsers` में सफल होना चाहिए। टोकन, पासवर्ड, Tailscale और अप्रमाणित कनेक्शन कभी भी इस नीति का उपयोग नहीं करते।
2. केवल नए Control UI या WebChat ब्राउज़र डिवाइस को स्वतः स्वीकृत किया जा सकता है। स्कोप अपग्रेड सहित किसी मौजूदा डिवाइस का कोई भी अनुरोध `openclaw devices approve <requestId>` के साथ मैन्युअल स्वीकृति के लिए लंबित रहता है।
3. डिवाइस को `operator` भूमिका के साथ स्वीकृत किया जाता है। यदि कनेक्ट अनुरोध में स्कोप शामिल हैं, तो अनुदान अनुरोधित स्कोप और `deviceAutoApprove.scopes` का सटीक प्रतिच्छेद होता है। यदि अनुरोध में स्कोप नहीं हैं, तो कॉन्फ़िगर की गई सूची प्रदान की जाती है; वह सूची न दिए जाने पर यह डिफ़ॉल्ट रूप से `operator.read`, `operator.write`, और `operator.approvals` होती है। इसके बाद परिणामी अनुदान को कनेक्शन के [`x-openclaw-scopes`](#control-ui-pairing-behavior) प्रॉक्सी हेडर द्वारा, उसके मौजूद होने पर, अतिरिक्त रूप से सीमित किया जाता है। इसलिए किसी उपयोगकर्ता के स्कोप को सीमित करने वाला प्रॉक्सी केवल सत्र ही नहीं, बल्कि **स्थायी** डिवाइस अनुदान भी सीमित करता है—मौजूद लेकिन खाली हेडर से कोई स्कोप नहीं मिलता। यह सीमा तब भी लागू होती है जब क्लाइंट अपनी स्कोप सूची नहीं देता।
4. `operator.admin` की अनुमति केवल `deviceAutoApprove.scopes` में स्पष्ट रूप से सूचीबद्ध किए जाने पर है। सूचीबद्ध होने पर प्रत्येक प्रॉक्सी-प्रमाणित उपयोगकर्ता नए ब्राउज़र डिवाइस पर पूर्ण एडमिन का अनुरोध कर सकता है और उसे स्वतः प्राप्त कर सकता है; बिना स्कोप वाले अनुरोधों को स्वतः पूर्ण एडमिन मिलता है। `openclaw security audit` CRITICAL `gateway.trusted_proxy_device_auto_approve_admin` निष्कर्ष रिपोर्ट करता है और Gateway स्टार्टअप पर एक बार चेतावनी लॉग करता है। प्रति-पहचान भूमिकाएँ उपलब्ध होने तक `openclaw devices approve` या `openclaw devices rotate` के साथ मैन्युअल एडमिन स्वीकृति को प्राथमिकता दें।

<Warning>
इस विकल्प को सक्षम करने पर नए ब्राउज़र डिवाइस का नामांकन पूरी तरह रिवर्स-प्रॉक्सी पहचान को सौंप दिया जाता है। समझौता किया गया प्रॉक्सी खाता प्रत्येक कॉन्फ़िगर किए गए स्कोप वाले स्थायी डिवाइस का नामांकन कर सकता है। `operator.admin` को सूचीबद्ध करने पर वह डिवाइस मैन्युअल स्वीकृति के बिना पूर्ण प्रशासक बन जाता है। Gateway को केवल प्रॉक्सी के माध्यम से पहुँच योग्य रखें, मज़बूत प्रॉक्सी प्रमाणीकरण आवश्यक बनाएँ, पहचान हेडर ओवरराइट करें, और सीमित `allowUsers` सूची का उपयोग करें।
</Warning>

## Control UI पेयरिंग व्यवहार

जब `gateway.auth.mode = "trusted-proxy"` सक्रिय हो और अनुरोध विश्वसनीय-प्रॉक्सी जाँचों में सफल हो, तब Control UI WebSocket सत्र डिवाइस पेयरिंग पहचान के बिना कनेक्ट हो सकते हैं।

स्कोप संबंधी प्रभाव:

- डिवाइस-रहित Control UI WebSocket सत्र कनेक्ट होते हैं, लेकिन डिफ़ॉल्ट रूप से उन्हें कोई ऑपरेटर स्कोप नहीं मिलता। OpenClaw अनुरोधित स्कोप सूची को `[]` पर साफ़ कर देता है, ताकि किसी स्वीकृत पेयर्ड डिवाइस/टोकन से आबद्ध न होने वाला सत्र स्वयं अनुमतियाँ घोषित न कर सके।
- यदि सफल WebSocket कनेक्शन के बाद विधियाँ `missing scope` के साथ विफल होती हैं, तो HTTPS का उपयोग करें ताकि ब्राउज़र डिवाइस पहचान बना सके और पेयरिंग पूरी कर सके। [Control UI का असुरक्षित HTTP](/hi/web/control-ui#insecure-http) देखें।
- जिन पुराने कॉन्फ़िगरेशन में अब भी सेवानिवृत्त
  `gateway.controlUi.dangerouslyDisableDeviceAuth=true` कुंजी मौजूद है, वे सीमित
  [Control UI अपग्रेड माइग्रेशन](/hi/web/control-ui#device-pairing-first-connection) का उपयोग करते हैं।

रिवर्स-प्रॉक्सी स्कोप सीमा: यदि आपका प्रॉक्सी Control UI WebSocket अपग्रेड अनुरोध पर `x-openclaw-scopes` भेजता है, तो OpenClaw सत्र स्कोप को अनुरोधित स्कोप और घोषित स्कोप के प्रतिच्छेद तक सीमित कर देता है। यह हेडर स्कोप प्रदान नहीं करता; यह केवल सत्र द्वारा रखे जा सकने वाले स्कोप को सीमित करता है। जब `deviceAutoApprove.enabled` true हो, तो यही सीमा [स्वचालित डिवाइस स्वीकृति](#automatic-device-approval) द्वारा लिखे गए स्थायी डिवाइस अनुदान पर भी लागू होती है, इसलिए स्वतः स्वीकृत डिवाइस के पास प्रॉक्सी द्वारा घोषित स्कोप से अधिक स्कोप कभी नहीं होते।

प्रभाव:

- डिवाइस-रहित Control UI पहुँच के लिए पेयरिंग अब प्राथमिक गेट नहीं है। जब `deviceAutoApprove.enabled` true हो, तो प्रॉक्सी पहचान नए ब्राउज़र डिवाइस नामांकन के लिए भी स्वीकृति गेट बन जाती है।
- आपकी रिवर्स प्रॉक्सी प्रमाणीकरण नीति और `allowUsers` प्रभावी पहुँच नियंत्रण बन जाते हैं।
- Gateway प्रवेश को केवल विश्वसनीय प्रॉक्सी IP तक सीमित रखें (`gateway.trustedProxies` + फ़ायरवॉल)।

कस्टम WebSocket क्लाइंट Control UI सत्र नहीं हैं। सेवानिवृत्त Control UI
अपग्रेड इनपुट मनमाने
`client.mode: "backend"` या CLI-आकार वाले क्लाइंट को अस्थायी पहुँच प्रदान नहीं करता। कस्टम स्वचालन को
डिवाइस पहचान/पेयरिंग, आरक्षित प्रत्यक्ष-स्थानीय `client.id: "gateway-client"`
बैकएंड सहायक पथ, या जब HTTP अनुरोध/प्रतिक्रिया सतह अधिक उपयुक्त हो तब [एडमिन HTTP RPC Plugin](/hi/plugins/admin-http-rpc)
का उपयोग करना चाहिए।

## ऑपरेटर स्कोप हेडर

विश्वसनीय-प्रॉक्सी प्रमाणीकरण एक **पहचान-युक्त** HTTP मोड है, इसलिए कॉलर HTTP API अनुरोधों पर `x-openclaw-scopes` के साथ वैकल्पिक रूप से ऑपरेटर स्कोप घोषित कर सकते हैं।

नोट: WebSocket स्कोप Gateway प्रोटोकॉल हैंडशेक और डिवाइस पहचान बाइंडिंग द्वारा निर्धारित होते हैं। Control UI WebSocket अपग्रेड अनुरोधों पर, `x-openclaw-scopes` केवल तय किए गए सत्र स्कोप की अधिकतम सीमा है, कोई अनुमति नहीं। [Control UI पेयरिंग व्यवहार](#control-ui-pairing-behavior) देखें।

उदाहरण:

- `x-openclaw-scopes: operator.read`
- `x-openclaw-scopes: operator.read,operator.write`
- `x-openclaw-scopes: operator.admin,operator.write`

व्यवहार:

- हेडर मौजूद होने पर, OpenClaw घोषित स्कोप सेट का पालन करता है।
- हेडर मौजूद लेकिन खाली होने पर, अनुरोध **कोई भी** ऑपरेटर स्कोप घोषित नहीं करता।
- हेडर अनुपस्थित होने पर, सामान्य पहचान-युक्त HTTP API मानक डिफ़ॉल्ट ऑपरेटर स्कोप सेट (`operator.admin`, `operator.read`, `operator.write`, `operator.approvals`, `operator.pairing`, `operator.talk.secrets`) पर वापस जाते हैं।
- Gateway-प्रमाणीकरण वाली **Plugin HTTP रूट** डिफ़ॉल्ट रूप से अधिक सीमित होती हैं: जब `x-openclaw-scopes` अनुपस्थित होता है, तो उनका रनटाइम स्कोप केवल `operator.write` पर वापस जाता है।
- विश्वसनीय-प्रॉक्सी प्रमाणीकरण सफल होने के बाद भी ब्राउज़र-मूल HTTP अनुरोधों को `gateway.controlUi.allowedOrigins` (या जानबूझकर चुने गए Host-हेडर फ़ॉलबैक मोड) को पास करना होता है।

व्यावहारिक नियम: जब आप किसी विश्वसनीय-प्रॉक्सी अनुरोध को डिफ़ॉल्ट से अधिक सीमित रखना चाहते हों, या किसी Gateway-प्रमाणीकरण वाली Plugin रूट को लेखन स्कोप से अधिक शक्तिशाली अनुमति चाहिए, तो `x-openclaw-scopes` स्पष्ट रूप से भेजें।

## TLS समाप्ति और HSTS

एक TLS समाप्ति बिंदु का उपयोग करें और वहीं HSTS लागू करें।

<Tabs>
  <Tab title="प्रॉक्सी TLS समाप्ति (अनुशंसित)">
    जब आपका रिवर्स प्रॉक्सी `https://control.example.com` के लिए HTTPS संभालता है, तो उस डोमेन के लिए प्रॉक्सी पर `Strict-Transport-Security` सेट करें।

    - इंटरनेट-सामना करने वाले परिनियोजनों के लिए उपयुक्त।
    - प्रमाणपत्र और HTTP सुरक्षा-सुदृढ़ीकरण नीति को एक ही स्थान पर रखता है।
    - OpenClaw प्रॉक्सी के पीछे लूपबैक HTTP पर बना रह सकता है।

    उदाहरण हेडर मान:

    ```text
    Strict-Transport-Security: max-age=31536000; includeSubDomains
    ```

  </Tab>
  <Tab title="Gateway TLS समाप्ति">
    यदि OpenClaw स्वयं सीधे HTTPS प्रदान करता है (कोई TLS-समाप्त करने वाला प्रॉक्सी नहीं), तो यह सेट करें:

    ```json5
    {
      gateway: {
        tls: { enabled: true },
        http: {
          securityHeaders: {
            strictTransportSecurity: "max-age=31536000; includeSubDomains",
          },
        },
      },
    }
    ```

    `strictTransportSecurity` एक स्ट्रिंग हेडर मान स्वीकार करता है, या स्पष्ट रूप से अक्षम करने के लिए `false` स्वीकार करता है।

  </Tab>
</Tabs>

### चरणबद्ध लागू करने का मार्गदर्शन

- ट्रैफ़िक सत्यापित करते समय पहले छोटी अधिकतम अवधि (उदाहरण के लिए `max-age=300`) से शुरुआत करें।
- पूरा विश्वास होने के बाद ही दीर्घकालिक मानों (उदाहरण के लिए `max-age=31536000`) तक बढ़ाएँ।
- केवल तभी `includeSubDomains` जोड़ें, जब प्रत्येक सबडोमेन HTTPS के लिए तैयार हो।
- प्रीलोड का उपयोग केवल तभी करें, जब आप जानबूझकर अपने पूरे डोमेन सेट के लिए प्रीलोड आवश्यकताओं को पूरा करते हों।
- केवल-लूपबैक स्थानीय विकास को HSTS से लाभ नहीं मिलता।

## प्रॉक्सी सेटअप के उदाहरण

<AccordionGroup>
  <Accordion title="Pomerium">
    Pomerium `x-pomerium-claim-email` (या अन्य क्लेम हेडर) में पहचान और `x-pomerium-jwt-assertion` में JWT भेजता है।

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // Pomerium का IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-pomerium-claim-email",
            requiredHeaders: ["x-pomerium-jwt-assertion"],
          },
        },
      },
    }
    ```

    Pomerium कॉन्फ़िग स्निपेट:

    ```yaml
    routes:
      - from: https://openclaw.example.com
        to: http://openclaw-gateway:18789
        policy:
          - allow:
              or:
                - email:
                    is: nick@example.com
        pass_identity_headers: true
    ```

  </Accordion>
  <Accordion title="OAuth के साथ Caddy">
    `caddy-security` Plugin वाला Caddy उपयोगकर्ताओं को प्रमाणित कर सकता है और पहचान हेडर भेज सकता है।

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // Caddy/साइडकार प्रॉक्सी IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```

    Caddyfile स्निपेट:

    ```caddy
    openclaw.example.com {
        authenticate with oauth2_provider
        authorize with policy1

        reverse_proxy openclaw:18789 {
            header_up X-Forwarded-User {http.auth.user.email}
        }
    }
    ```

  </Accordion>
  <Accordion title="nginx + oauth2-proxy">
    oauth2-proxy उपयोगकर्ताओं को प्रमाणित करता है और `x-auth-request-email` में पहचान भेजता है।

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // nginx/oauth2-proxy IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-auth-request-email",
          },
        },
      },
    }
    ```

    nginx कॉन्फ़िग स्निपेट:

    ```nginx
    location / {
        auth_request /oauth2/auth;
        auth_request_set $user $upstream_http_x_auth_request_email;

        proxy_pass http://openclaw:18789;
        proxy_set_header X-Auth-Request-Email $user;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    ```

  </Accordion>
  <Accordion title="फ़ॉरवर्ड प्रमाणीकरण के साथ Traefik">
    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["172.17.0.1"], // Traefik कंटेनर IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## मिश्रित टोकन कॉन्फ़िगरेशन

यदि कोई साझा टोकन भी कॉन्फ़िगर किया गया हो (`gateway.auth.token` या `OPENCLAW_GATEWAY_TOKEN`), तो Gateway स्टार्टअप विश्वसनीय-प्रॉक्सी प्रमाणीकरण को अस्वीकार कर देता है। दोनों परस्पर अनन्य हैं, क्योंकि साझा टोकन समान होस्ट वाले कॉलर को उस प्रॉक्सी-सत्यापित पहचान से पूरी तरह अलग मार्ग पर प्रमाणित होने देगा, जिसे यह मोड लागू करने के लिए बनाया गया है।

यदि स्टार्टअप `gateway auth mode is trusted-proxy, but a shared token is also configured` जैसी त्रुटि के साथ विफल होता है:

- विश्वसनीय-प्रॉक्सी मोड का उपयोग करते समय साझा टोकन हटाएँ, या
- यदि आप टोकन-आधारित प्रमाणीकरण चाहते हैं, तो `gateway.auth.mode` को `"token"` पर बदलें।

लूपबैक विश्वसनीय-प्रॉक्सी पहचान हेडर अब भी सुरक्षित रूप से विफल होते हैं: समान होस्ट वाले कॉलर को चुपचाप प्रॉक्सी उपयोगकर्ता के रूप में प्रमाणित नहीं किया जाता। प्रॉक्सी को बायपास करने वाले आंतरिक OpenClaw कॉलर इसके बजाय `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` से प्रमाणित हो सकते हैं। विश्वसनीय-प्रॉक्सी मोड में टोकन फ़ॉलबैक जानबूझकर असमर्थित रहता है।

## सुरक्षा जाँच-सूची

विश्वसनीय-प्रॉक्सी प्रमाणीकरण सक्षम करने से पहले सत्यापित करें:

- [ ] **प्रॉक्सी ही एकमात्र मार्ग है**: Gateway पोर्ट को आपके प्रॉक्सी के अतिरिक्त हर स्रोत से फ़ायरवॉल द्वारा अवरुद्ध किया गया है।
- [ ] **trustedProxies न्यूनतम है**: केवल आपके वास्तविक प्रॉक्सी IP, पूरे सबनेट नहीं।
- [ ] **लूपबैक प्रॉक्सी स्रोत जानबूझकर चुना गया है**: लूपबैक-स्रोत अनुरोधों के लिए विश्वसनीय-प्रॉक्सी प्रमाणीकरण सुरक्षित रूप से विफल होता है, जब तक समान होस्ट वाले प्रॉक्सी के लिए `gateway.auth.trustedProxy.allowLoopback` स्पष्ट रूप से सक्षम न किया गया हो।
- [ ] **प्रॉक्सी हेडर हटाता है**: आपका प्रॉक्सी क्लाइंट से मिले `x-forwarded-*` हेडर को जोड़ने के बजाय अधिलेखित करता है।
- [ ] **TLS समाप्ति**: आपका प्रॉक्सी TLS संभालता है; उपयोगकर्ता HTTPS के माध्यम से कनेक्ट होते हैं।
- [ ] **allowedOrigins स्पष्ट है**: गैर-लूपबैक Control UI स्पष्ट `gateway.controlUi.allowedOrigins` का उपयोग करता है।
- [ ] **allowUsers सेट है** (अनुशंसित): प्रत्येक प्रमाणित व्यक्ति को अनुमति देने के बजाय ज्ञात उपयोगकर्ताओं तक सीमित करें।
- [ ] **कोई मिश्रित टोकन कॉन्फ़िगरेशन नहीं**: `gateway.auth.token` और `gateway.auth.mode: "trusted-proxy"` दोनों सेट न करें।
- [ ] **स्थानीय पासवर्ड फ़ॉलबैक निजी है**: यदि आप आंतरिक प्रत्यक्ष कॉलर के लिए `gateway.auth.password` कॉन्फ़िगर करते हैं, तो Gateway पोर्ट को फ़ायरवॉल से सुरक्षित रखें, ताकि गैर-प्रॉक्सी दूरस्थ क्लाइंट उस तक सीधे न पहुँच सकें।
- [ ] **डिवाइस स्वतः-अनुमोदन जानबूझकर सक्षम किया गया है**: यदि `deviceAutoApprove.enabled` सत्य है, तो रिवर्स-प्रॉक्सी खाते की सुरक्षा को डिवाइस-पंजीकरण सीमा मानें और दिए गए स्कोप की सूची को गैर-व्यवस्थापक और न्यूनतम रखें।

## सुरक्षा ऑडिट

`openclaw security audit` विश्वसनीय-प्रॉक्सी प्रमाणीकरण को **गंभीर** तीव्रता वाले निष्कर्ष के रूप में चिह्नित करता है। यह जानबूझकर है; यह याद दिलाता है कि आप सुरक्षा अपने प्रॉक्सी सेटअप को सौंप रहे हैं।

ऑडिट इनकी जाँच करता है:

- मूल `gateway.trusted_proxy_auth` चेतावनी/गंभीर अनुस्मारक।
- `trustedProxies` कॉन्फ़िगरेशन अनुपस्थित होना।
- `userHeader` कॉन्फ़िगरेशन अनुपस्थित होना।
- खाली `allowUsers` (किसी भी प्रमाणित उपयोगकर्ता को अनुमति देता है)।
- समान होस्ट वाले प्रॉक्सी स्रोतों के लिए `allowLoopback` सक्षम होना।
- ब्राउज़र डिवाइस स्वतः-अनुमोदन सक्षम होना (नए डिवाइस की पेयरिंग प्रॉक्सी पहचान को सौंपता है)।

Control UI उजागर होने पर अलग, गैर-विश्वसनीय-प्रॉक्सी-विशिष्ट निष्कर्ष भी लागू होते हैं: वाइल्डकार्ड या अनुपस्थित `gateway.controlUi.allowedOrigins`, और Host-हेडर मूल फ़ॉलबैक।

## समस्या निवारण

<AccordionGroup>
  <Accordion title="trusted_proxy_untrusted_source">
    अनुरोध `gateway.trustedProxies` में मौजूद किसी IP से नहीं आया। जाँचें:

    - क्या प्रॉक्सी IP सही है? (Docker कंटेनर IP बदल सकते हैं।)
    - क्या आपके प्रॉक्सी के आगे कोई लोड बैलेंसर है?
    - वास्तविक IP खोजने के लिए `docker inspect` या `kubectl get pods -o wide` का उपयोग करें।

  </Accordion>
  <Accordion title="trusted_proxy_loopback_source">
    OpenClaw ने लूपबैक-स्रोत वाले विश्वसनीय-प्रॉक्सी अनुरोध को अस्वीकार कर दिया।

    जाँचें:

    - क्या प्रॉक्सी `127.0.0.1` / `::1` से कनेक्ट हो रहा है?
    - क्या आप समान होस्ट वाले लूपबैक रिवर्स प्रॉक्सी के साथ विश्वसनीय-प्रॉक्सी प्रमाणीकरण का उपयोग करने का प्रयास कर रहे हैं?

    समाधान:

    - उन आंतरिक समान-होस्ट क्लाइंट के लिए टोकन/पासवर्ड प्रमाणीकरण को प्राथमिकता दें, जो प्रॉक्सी से होकर नहीं जाते, या
    - गैर-लूपबैक विश्वसनीय प्रॉक्सी पते से रूट करें और उस IP को `gateway.trustedProxies` में रखें, या
    - जानबूझकर उपयोग किए गए समान-होस्ट रिवर्स प्रॉक्सी के लिए, `gateway.auth.trustedProxy.allowLoopback = true` सेट करें, लूपबैक पते को `gateway.trustedProxies` में रखें और सुनिश्चित करें कि प्रॉक्सी पहचान हेडर हटाता या अधिलेखित करता है।

  </Accordion>
  <Accordion title="trusted_proxy_local_interface_source / trusted_proxy_local_interface_check_failed">
    अनुरोध का स्रोत IP Gateway होस्ट के अपने गैर-लूपबैक नेटवर्क इंटरफ़ेस पतों में से किसी एक से मेल खाता था (प्रॉक्सी से नहीं); यह टेलनेट या Docker ब्रिज नेटवर्क पर जाली समान-होस्ट ट्रैफ़िक से सुरक्षा प्रदान करता है। `..._check_failed` का अर्थ है कि इंटरफ़ेस खोज में ही त्रुटि हुई, इसलिए OpenClaw सुरक्षित रूप से विफल होता है।

    जाँचें:

    - क्या Gateway होस्ट पर ही कोई प्रक्रिया प्रॉक्सी को बायपास करके सीधे पहचान हेडर भेज रही है?
    - क्या प्रॉक्सी Gateway के समान नेटवर्क नेमस्पेस में ऐसे IP के साथ चलता है, जो स्थानीय इंटरफ़ेस के रूप में भी दिखाई देता है?

    समाधान: प्रॉक्सी ट्रैफ़िक को ऐसे पते से रूट करें, जो Gateway होस्ट पर स्थानीय रूप से भी बाइंड न हो, या केवल वास्तविक समान-होस्ट प्रॉक्सी सेटअप के लिए `allowLoopback` का उपयोग करें।

  </Accordion>
  <Accordion title="trusted_proxy_user_missing">
    उपयोगकर्ता हेडर खाली या अनुपस्थित था। जाँचें:

    - क्या आपका प्रॉक्सी पहचान हेडर भेजने के लिए कॉन्फ़िगर किया गया है?
    - क्या हेडर का नाम सही है? (अक्षरों के आकार से फ़र्क नहीं पड़ता, लेकिन वर्तनी सही होनी चाहिए)
    - क्या उपयोगकर्ता वास्तव में प्रॉक्सी पर प्रमाणित है?

  </Accordion>
  <Accordion title="trusted_proxy_missing_header_*">
    कोई आवश्यक हेडर मौजूद नहीं था। जाँचें:

    - उन विशिष्ट हेडर के लिए आपका प्रॉक्सी कॉन्फ़िगरेशन।
    - क्या श्रृंखला में कहीं हेडर हटाए जा रहे हैं।

  </Accordion>
  <Accordion title="trusted_proxy_user_not_allowed">
    उपयोगकर्ता प्रमाणित है, लेकिन `allowUsers` में नहीं है। या तो उन्हें जोड़ें या अनुमति-सूची हटाएँ।
  </Accordion>
  <Accordion title="trusted_proxy_no_proxies_configured / trusted_proxy_config_missing">
    `gateway.auth.mode`, `"trusted-proxy"` है, लेकिन `gateway.trustedProxies` खाली है, या स्वयं `gateway.auth.trustedProxy` मौजूद नहीं है। जब तक दोनों सेट नहीं किए जाते, प्रत्येक अनुरोध अस्वीकार कर दिया जाता है।
  </Accordion>
  <Accordion title="trusted_proxy_origin_not_allowed">
    विश्वसनीय प्रॉक्सी प्रमाणीकरण सफल हुआ, लेकिन ब्राउज़र का `Origin` हेडर Control UI की ओरिजिन जाँच में सफल नहीं हुआ।

    जाँचें:

    - `gateway.controlUi.allowedOrigins` में ब्राउज़र का सटीक ओरिजिन शामिल है।
    - आप वाइल्डकार्ड ओरिजिन पर निर्भर नहीं हैं, जब तक कि आप जानबूझकर सभी को अनुमति देने वाला व्यवहार नहीं चाहते।
    - यदि आप जानबूझकर Host-header फ़ॉलबैक मोड का उपयोग करते हैं, तो `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` सुविचारित रूप से सेट किया गया है।

  </Accordion>
  <Accordion title="कनेक्शन सफल होता है, लेकिन विधियाँ अनुपलब्ध स्कोप की सूचना देती हैं">
    WebSocket कनेक्ट हो जाता है, लेकिन `chat.history`, `sessions.list`, या
    `models.list`, `missing scope: operator.read` के साथ विफल हो जाता है।

    सामान्य कारण:

    - डिवाइस-रहित Control UI सत्र: विश्वसनीय प्रॉक्सी प्रमाणीकरण डिवाइस पहचान के बिना WebSocket कनेक्शन स्वीकार कर सकता है, लेकिन OpenClaw डिज़ाइन के अनुसार डिवाइस-रहित सत्रों से स्कोप हटा देता है।
    - कस्टम बैकएंड क्लाइंट: सेवानिवृत्त Control UI अपग्रेड इनपुट कभी भी मनमाने बैकएंड या CLI-जैसे WebSocket क्लाइंट को पहुँच प्रदान नहीं करता।
    - अत्यधिक सीमित `x-openclaw-scopes`: यदि आपका प्रॉक्सी Control UI के WebSocket अपग्रेड अनुरोध पर यह हेडर इंजेक्ट करता है, तो सत्र के स्कोप उस सेट तक सीमित हो जाते हैं। हेडर का खाली मान कोई स्कोप प्रदान नहीं करता।

    समाधान:

    - Control UI के लिए HTTPS का उपयोग करें, ताकि ब्राउज़र डिवाइस पहचान बना सके और पेयरिंग पूरी कर सके।
    - कस्टम स्वचालन के लिए डिवाइस पहचान/पेयरिंग, आरक्षित प्रत्यक्ष-स्थानीय `gateway-client` बैकएंड सहायक पथ, या [एडमिन HTTP RPC](/hi/plugins/admin-http-rpc) का उपयोग करें।
    - वर्तमान कॉन्फ़िगरेशन में सेवानिवृत्त `gateway.controlUi.dangerouslyDisableDeviceAuth` कुंजी न जोड़ें। पुराने इंस्टॉलेशन एकबारगी स्व-पेयरिंग माइग्रेशन का स्वचालित रूप से उपयोग करते हैं।

  </Accordion>
  <Accordion title="WebSocket अब भी विफल हो रहा है">
    सुनिश्चित करें कि आपका प्रॉक्सी:

    - WebSocket अपग्रेड (`Upgrade: websocket`, `Connection: upgrade`) का समर्थन करता है।
    - WebSocket अपग्रेड अनुरोधों पर पहचान हेडर भेजता है (केवल HTTP पर नहीं)।
    - WebSocket कनेक्शन के लिए अलग प्रमाणीकरण पथ नहीं रखता।

  </Accordion>
</AccordionGroup>

## टोकन प्रमाणीकरण से माइग्रेशन

<Steps>
  <Step title="प्रॉक्सी कॉन्फ़िगर करें">
    उपयोगकर्ताओं को प्रमाणित करने और हेडर भेजने के लिए अपना प्रॉक्सी कॉन्फ़िगर करें।
  </Step>
  <Step title="प्रॉक्सी का स्वतंत्र रूप से परीक्षण करें">
    प्रॉक्सी सेटअप का स्वतंत्र रूप से परीक्षण करें (हेडर के साथ curl)।
  </Step>
  <Step title="OpenClaw कॉन्फ़िगरेशन अपडेट करें">
    विश्वसनीय प्रॉक्सी प्रमाणीकरण के साथ OpenClaw कॉन्फ़िगरेशन अपडेट करें।
  </Step>
  <Step title="Gateway पुनः आरंभ करें">
    Gateway पुनः आरंभ करें।
  </Step>
  <Step title="WebSocket का परीक्षण करें">
    Control UI से WebSocket कनेक्शन का परीक्षण करें।
  </Step>
  <Step title="ऑडिट करें">
    `openclaw security audit` चलाएँ और निष्कर्षों की समीक्षा करें।
  </Step>
</Steps>

## संबंधित

- [कॉन्फ़िगरेशन](/hi/gateway/configuration) — कॉन्फ़िगरेशन संदर्भ
- [ऑपरेटर स्कोप](/hi/gateway/operator-scopes) — भूमिकाएँ, स्कोप और अनुमोदन जाँच
- [दूरस्थ पहुँच](/hi/gateway/remote) — दूरस्थ पहुँच के अन्य पैटर्न
- [सुरक्षा](/hi/gateway/security) — संपूर्ण सुरक्षा मार्गदर्शिका
- [Tailscale](/hi/gateway/tailscale) — केवल टेलनेट पहुँच के लिए सरल विकल्प
