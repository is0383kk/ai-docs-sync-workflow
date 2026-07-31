---
read_when:
    - रिमोट Gateway सेटअप चलाना या उनकी समस्या निवारण करना
summary: Gateway WS, SSH टनल और टेलनेट का उपयोग करके रिमोट एक्सेस
title: दूरस्थ पहुँच
x-i18n:
    generated_at: "2026-07-27T17:58:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8f05e32fcfa16d5ddfcd684d0550c9af311914e2b4d91c95edad3490dc2e56d9
    source_path: gateway/remote.md
    workflow: 16
---

OpenClaw किसी होस्ट पर एक Gateway (मास्टर) चलाता है और प्रत्येक क्लाइंट को उससे जोड़ता है। Gateway सत्रों, प्रमाणीकरण प्रोफ़ाइलों, चैनलों और स्थिति का स्वामी होता है; बाकी सब कुछ क्लाइंट है।

- **ऑपरेटर** (आप या macOS ऐप): Gateway पहुँच योग्य होने पर प्रत्यक्ष LAN/Tailnet WebSocket सबसे सरल है; SSH टनलिंग सार्वभौमिक वैकल्पिक उपाय है।
- **Nodes** (iOS/Android और अन्य डिवाइस): Gateway **WebSocket** (LAN/tailnet या SSH टनल) से कनेक्ट होते हैं।

## मूल अवधारणा

Gateway WebSocket डिफ़ॉल्ट रूप से पोर्ट `18789` (`gateway.port`) पर **लूपबैक** से बाइंड होता है। रिमोट उपयोग के लिए, इसे Tailscale Serve / किसी विश्वसनीय LAN-Tailnet बाइंड के माध्यम से उपलब्ध कराएँ या लूपबैक पोर्ट को SSH पर फ़ॉरवर्ड करें।

## टोपोलॉजी विकल्प

| सेटअप                             | Gateway कहाँ चलता है                                                                                    | किसके लिए सर्वोत्तम                                                                                                                                          |
| --------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| आपके tailnet में हमेशा चालू Gateway | स्थायी होस्ट (VPS या होम सर्वर), जिस तक Tailscale या SSH के माध्यम से पहुँचा जाता है                                        | ऐसे लैपटॉप जो अक्सर स्लीप मोड में चले जाते हैं, लेकिन जिनके लिए एजेंट का हमेशा चालू रहना आवश्यक है। [exe.dev](/hi/install/exe-dev) (आसान VM) या [Hetzner](/hi/install/hetzner) (प्रोडक्शन VPS) देखें। |
| होम डेस्कटॉप                      | डेस्कटॉप; लैपटॉप macOS ऐप के रिमोट मोड (Settings → Connection → OpenClaw runs) से रिमोट रूप से कनेक्ट होता है | एजेंट को ऐसे हार्डवेयर पर रखना जो चालू रहता है। रनबुक: [macOS रिमोट एक्सेस](/hi/platforms/mac/remote)।                                       |
| लैपटॉप                            | लैपटॉप, जिसे SSH टनल या Tailscale Serve के माध्यम से सुरक्षित रूप से उपलब्ध कराया गया हो (`gateway.bind: "loopback"` बनाए रखें)                | एकल-मशीन सेटअप। [Tailscale](/hi/gateway/tailscale) और [वेब](/hi/web) देखें।                                                                       |

हमेशा चालू और लैपटॉप सेटअप के लिए, `gateway.bind: "loopback"` बनाए रखना और Control UI के लिए **Tailscale Serve** या `gateway.remote.transport: "direct"` के साथ किसी विश्वसनीय LAN/Tailnet बाइंड का उपयोग करना बेहतर है। SSH टनल ऐसा वैकल्पिक उपाय है जो किसी भी मशीन से काम करता है।

## कमांड प्रवाह (क्या कहाँ चलता है)

एक Gateway स्थिति और चैनलों का स्वामी होता है; Nodes परिधीय डिवाइस होते हैं। उदाहरण (Telegram संदेश को किसी Node टूल पर रूट किया गया):

1. Telegram संदेश **Gateway** पर आता है।
2. Gateway **एजेंट** चलाता है, जो तय करता है कि किसी Node टूल को कॉल करना है या नहीं।
3. Gateway, Gateway WebSocket (`node.invoke` RPC) पर **Node** को कॉल करता है।
4. Node परिणाम लौटाता है; Gateway Telegram को उत्तर देता है।

Nodes Gateway सेवा नहीं चलाते। प्रत्येक होस्ट पर केवल एक Gateway चलना चाहिए, जब तक कि आप जानबूझकर अलग-अलग प्रोफ़ाइल न चला रहे हों ([एकाधिक Gateway](/hi/gateway/multiple-gateways) देखें)। macOS ऐप का "Node मोड" केवल Gateway WebSocket पर चलने वाला एक Node क्लाइंट है।

## SSH टनल (CLI + टूल)

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

टनल चालू होने पर, `openclaw health` और `openclaw status --deep`, `ws://127.0.0.1:18789` के माध्यम से रिमोट Gateway तक पहुँचते हैं। `openclaw gateway status`, `openclaw gateway health`, `openclaw gateway probe`, और `openclaw gateway call` भी `--url` के माध्यम से किसी फ़ॉरवर्ड किए गए URL को लक्षित कर सकते हैं।

<Note>
`18789` को अपने कॉन्फ़िगर किए गए `gateway.port` (या `--port` / `OPENCLAW_GATEWAY_PORT`) से बदलें।
</Note>

<Warning>
`--url` कभी भी कॉन्फ़िगरेशन या परिवेश क्रेडेंशियल पर वापस नहीं जाता। `--token` या `--password` स्पष्ट रूप से दें; इनके बिना क्लाइंट कोई क्रेडेंशियल नहीं भेजता और यदि लक्षित Gateway को प्रमाणीकरण की आवश्यकता हो, तो कनेक्शन विफल हो जाता है।
</Warning>

## CLI रिमोट डिफ़ॉल्ट

रिमोट लक्ष्य सहेजें ताकि CLI कमांड डिफ़ॉल्ट रूप से उसका उपयोग करें:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://127.0.0.1:18789",
      token: "your-token",
    },
  },
}
```

जब Gateway केवल लूपबैक पर हो, तो URL को `ws://127.0.0.1:18789` पर रखें और पहले SSH टनल खोलें। macOS ऐप के SSH-टनल ट्रांसपोर्ट में, खोजा गया Gateway होस्टनाम `gateway.remote.sshTarget` (`user@host` या `user@host:port`) में जाता है; `gateway.remote.url` स्थानीय टनल URL बना रहता है। यदि रिमोट पोर्ट स्थानीय पोर्ट से अलग है, तो `gateway.remote.remotePort` सेट करें।

होस्ट-कुंजी सत्यापन डिफ़ॉल्ट रूप से सख्त है (`gateway.remote.sshHostKeyPolicy: "strict"`)। इसे अपने प्रभावी OpenSSH कॉन्फ़िगरेशन को ज़िम्मेदारी सौंपने के लिए `"openssh"` पर सेट करें; इसे सक्षम करने से पहले अपनी उपयोगकर्ता और सिस्टम SSH सेटिंग्स की समीक्षा करें।

किसी विश्वसनीय LAN या Tailnet पर पहले से पहुँच योग्य Gateway के लिए, प्रत्यक्ष मोड का उपयोग करें:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      transport: "direct",
      url: "ws://192.168.0.202:18789",
      token: "your-token",
    },
  },
}
```

## क्रेडेंशियल प्राथमिकता

Gateway क्रेडेंशियल रिज़ॉल्यूशन, कॉल/प्रोब/स्थिति पथों और Discord निष्पादन-अनुमोदन निगरानी में एक साझा अनुबंध का पालन करता है। Node-host एक स्थानीय-मोड अपवाद के साथ उसी अनुबंध का उपयोग करता है (यह `gateway.remote.*` को अनदेखा करता है)।

- स्पष्ट क्रेडेंशियल (`--token`, `--password`, या किसी टूल का `gatewayToken`) उन कॉल पथों पर हमेशा प्राथमिकता लेते हैं जो स्पष्ट प्रमाणीकरण स्वीकार करते हैं।
- URL ओवरराइड सुरक्षा:
  - CLI `--url` कभी भी निहित कॉन्फ़िगरेशन/परिवेश क्रेडेंशियल का पुनः उपयोग नहीं करता।
  - परिवेश `OPENCLAW_GATEWAY_URL` केवल परिवेश क्रेडेंशियल (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`) का उपयोग कर सकता है।
- स्थानीय मोड डिफ़ॉल्ट:
  - टोकन: `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token` -> `gateway.remote.token` (रिमोट वैकल्पिक उपाय केवल तभी जब स्थानीय टोकन सेट न हो)
  - पासवर्ड: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.auth.password` -> `gateway.remote.password` (रिमोट वैकल्पिक उपाय केवल तभी जब स्थानीय पासवर्ड सेट न हो)
- रिमोट मोड डिफ़ॉल्ट:
  - टोकन: `gateway.remote.token` -> `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token`
  - पासवर्ड: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.remote.password` -> `gateway.auth.password`
- Node-host स्थानीय-मोड अपवाद: `gateway.remote.token` / `gateway.remote.password` को अनदेखा किया जाता है।
- रिमोट प्रोब/स्थिति टोकन जाँच डिफ़ॉल्ट रूप से सख्त होती हैं: रिमोट मोड को लक्षित करते समय वे केवल `gateway.remote.token` का उपयोग करती हैं (कोई स्थानीय टोकन वैकल्पिक उपाय नहीं)।
- Gateway परिवेश ओवरराइड केवल `OPENCLAW_GATEWAY_*` का उपयोग करते हैं।

## चैट UI रिमोट एक्सेस

WebChat का कोई अलग HTTP पोर्ट नहीं है; SwiftUI चैट UI सीधे Gateway WebSocket से कनेक्ट होता है।

- `18789` को SSH पर फ़ॉरवर्ड करें (ऊपर देखें), फिर क्लाइंटों को `ws://127.0.0.1:18789` से कनेक्ट करें।
- LAN/Tailnet प्रत्यक्ष मोड के लिए, क्लाइंटों को कॉन्फ़िगर किए गए निजी `ws://` या सुरक्षित `wss://` URL से कनेक्ट करें।
- macOS पर, ऐप का रिमोट मोड चुने गए ट्रांसपोर्ट को स्वचालित रूप से प्रबंधित करता है।

## macOS ऐप रिमोट मोड

macOS मेन्यू बार ऐप उसी सेटअप को शुरू से अंत तक संचालित करता है: रिमोट स्थिति जाँच, WebChat और Voice Wake फ़ॉरवर्डिंग। रनबुक: [macOS रिमोट एक्सेस](/hi/platforms/mac/remote)।

## सुरक्षा नियम (रिमोट/VPN)

जब तक आपको बाइंड की आवश्यकता के बारे में निश्चित न हो, Gateway को **केवल लूपबैक** पर रखें।

- **लूपबैक + SSH/Tailscale Serve** सबसे सुरक्षित डिफ़ॉल्ट है (कोई सार्वजनिक एक्सपोज़र नहीं)।
- प्लेनटेक्स्ट `ws://` लूपबैक, निजी/LAN (RFC 1918), लिंक-लोकल, CGNAT, `.local`, और `.ts.net` होस्ट के लिए स्वीकार किया जाता है। सार्वजनिक रिमोट होस्ट को `wss://` का उपयोग करना आवश्यक है।
- **गैर-लूपबैक बाइंड** (`lan`/`tailnet`/`custom`, या लूपबैक अनुपलब्ध होने पर `auto`) को Gateway प्रमाणीकरण का उपयोग करना आवश्यक है: टोकन, पासवर्ड, या `gateway.auth.mode: "trusted-proxy"` वाला पहचान-सजग रिवर्स प्रॉक्सी।
- `gateway.remote.token` / `.password` क्लाइंट क्रेडेंशियल स्रोत हैं; वे स्वयं सर्वर प्रमाणीकरण कॉन्फ़िगर नहीं करते।
- स्थानीय कॉल पथ `gateway.remote.*` का उपयोग केवल तभी वैकल्पिक उपाय के रूप में कर सकते हैं जब `gateway.auth.*` सेट न हो।
- यदि `gateway.auth.token` / `gateway.auth.password` को SecretRef के माध्यम से स्पष्ट रूप से कॉन्फ़िगर किया गया है और उसका रिज़ॉल्यूशन नहीं होता, तो रिज़ॉल्यूशन बंद अवस्था में विफल होता है (रिमोट वैकल्पिक उपाय से त्रुटि नहीं छिपती)।
- `gateway.remote.tlsFingerprint`, `wss://` के लिए रिमोट TLS प्रमाणपत्र को पिन करता है, जिसमें ऑपरेटर/कंट्रोल ट्रैफ़िक और macOS प्रत्यक्ष मोड का सहायक Node, दोनों शामिल हैं। संग्रहीत पिन के बिना, macOS सामान्य सिस्टम ट्रस्ट सफल होने के बाद ही पहले उपयोग पर पिन करता है; स्व-हस्ताक्षरित या निजी-CA Gateway के लिए स्पष्ट फ़िंगरप्रिंट या SSH के माध्यम से रिमोट आवश्यक है।
- जब `gateway.auth.allowTailscale: true` हो, तब **Tailscale Serve** पहचान हेडर के माध्यम से Control UI/WebSocket ट्रैफ़िक को प्रमाणित कर सकता है। HTTP API एंडपॉइंट उस हेडर प्रमाणीकरण का उपयोग नहीं करते और इसके बजाय Gateway के सामान्य HTTP प्रमाणीकरण मोड का पालन करते हैं। यह टोकन-रहित प्रवाह मानता है कि Gateway होस्ट विश्वसनीय है; हर जगह साझा-सीक्रेट प्रमाणीकरण के लिए इसे `false` पर सेट करें।
- **विश्वसनीय-प्रॉक्सी** प्रमाणीकरण डिफ़ॉल्ट रूप से किसी गैर-लूपबैक पहचान-सजग प्रॉक्सी की अपेक्षा करता है। उसी होस्ट के लूपबैक रिवर्स प्रॉक्सी के लिए स्पष्ट `gateway.auth.trustedProxy.allowLoopback = true` आवश्यक है।
- ब्राउज़र नियंत्रण को ऑपरेटर एक्सेस की तरह मानें: केवल tailnet और सोच-समझकर किया गया Node पेयरिंग।

विस्तृत जानकारी: [सुरक्षा](/hi/gateway/security)।

### macOS: LaunchAgent के माध्यम से स्थायी SSH टनल

macOS क्लाइंटों के लिए, सबसे आसान स्थायी सेटअप एक SSH `LocalForward` कॉन्फ़िगरेशन प्रविष्टि और एक LaunchAgent का उपयोग करता है, जो रीबूट और क्रैश के बाद भी टनल को चालू रखता है।

#### चरण 1: SSH कॉन्फ़िगरेशन जोड़ें

`~/.ssh/config` संपादित करें:

```ssh
Host remote-gateway
    HostName <REMOTE_IP>
    User <REMOTE_USER>
    LocalForward 18789 127.0.0.1:18789
    IdentityFile ~/.ssh/id_rsa
```

`<REMOTE_IP>` और `<REMOTE_USER>` को अपने मानों से बदलें।

#### चरण 2: SSH कुंजी कॉपी करें (एक बार)

```bash
ssh-copy-id -i ~/.ssh/id_rsa <REMOTE_USER>@<REMOTE_IP>
```

#### चरण 3: Gateway टोकन कॉन्फ़िगर करें

```bash
openclaw config set gateway.remote.token "<your-token>"
```

यदि रिमोट Gateway पासवर्ड प्रमाणीकरण का उपयोग करता है, तो इसके बजाय `gateway.remote.password` का उपयोग करें। `OPENCLAW_GATEWAY_TOKEN` अभी भी शेल-स्तरीय ओवरराइड के रूप में मान्य है, लेकिन स्थायी रिमोट-क्लाइंट सेटअप `gateway.remote.token` / `gateway.remote.password` है।

#### चरण 4: LaunchAgent बनाएँ

`~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist` के रूप में सहेजें:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.ssh-tunnel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/ssh</string>
        <string>-N</string>
        <string>remote-gateway</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

#### चरण 5: LaunchAgent लोड करें

```bash
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist
```

टनल लॉगिन पर स्वचालित रूप से शुरू होती है, क्रैश होने पर पुनः शुरू होती है और फ़ॉरवर्ड किए गए पोर्ट को सक्रिय रखती है।

<Note>
यदि पुराने सेटअप का बचा हुआ `com.openclaw.ssh-tunnel` LaunchAgent मौजूद है, तो उसे अनलोड करके हटा दें।
</Note>

#### समस्या निवारण

```bash
# जाँचें कि टनल चल रही है या नहीं
ps aux | grep "ssh -N remote-gateway" | grep -v grep
lsof -i :18789

# टनल पुनः शुरू करें
launchctl kickstart -k gui/$UID/ai.openclaw.ssh-tunnel

# टनल रोकें
launchctl bootout gui/$UID/ai.openclaw.ssh-tunnel
```

| कॉन्फ़िग प्रविष्टि                         | इसका कार्य                                                 |
| ------------------------------------ | ------------------------------------------------------------ |
| `LocalForward 18789 127.0.0.1:18789` | स्थानीय पोर्ट 18789 को रिमोट पोर्ट 18789 पर फ़ॉरवर्ड करता है               |
| `ssh -N`                             | रिमोट कमांड निष्पादित किए बिना SSH (केवल पोर्ट फ़ॉरवर्डिंग) |
| `KeepAlive`                          | टनल क्रैश होने पर उसे स्वचालित रूप से पुनः आरंभ करता है              |
| `RunAtLoad`                          | लॉगिन पर LaunchAgent लोड होने के समय टनल आरंभ करता है        |

## संबंधित

- [Tailscale](/hi/gateway/tailscale)
- [प्रमाणीकरण](/hi/gateway/authentication)
- [रिमोट Gateway सेटअप](/hi/gateway/remote-gateway-readme)
