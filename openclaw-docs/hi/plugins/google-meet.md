---
read_when:
    - आप चाहते हैं कि एक OpenClaw एजेंट Google Meet कॉल में शामिल हो जाए
    - आप चाहते हैं कि कोई OpenClaw एजेंट एक नई Google Meet कॉल बनाए
    - आप Chrome, Chrome node या Twilio को Google Meet ट्रांसपोर्ट के रूप में कॉन्फ़िगर कर रहे हैं
summary: 'Google Meet Plugin: एजेंट टॉक-बैक डिफ़ॉल्ट के साथ Chrome या Twilio के माध्यम से स्पष्ट Meet URL से जुड़ें'
title: Google Meet Plugin
x-i18n:
    generated_at: "2026-07-27T18:07:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8a611e283fe900984a29b563969936a641c7af430b05933eb03b98dc93b5d0c8
    source_path: plugins/google-meet.md
    workflow: 16
---

`google-meet` Plugin किसी OpenClaw एजेंट की ओर से स्पष्ट Meet URL से जुड़ता है। इसका दायरा जानबूझकर सीमित है:

- यह केवल `https://meet.google.com/...` URL से जुड़ता है; यह स्वयं खोजे गए फ़ोन नंबर से कभी किसी मीटिंग में डायल नहीं करता।
- `googlemeet create` Google Meet API (या ब्राउज़र फ़ॉलबैक) के माध्यम से नया Meet URL बना सकता है और डिफ़ॉल्ट रूप से उससे जुड़ सकता है।
- Chrome से भागीदारी के लिए साइन-इन की हुई Chrome प्रोफ़ाइल का उपयोग होता है, जो वैकल्पिक रूप से किसी युग्मित Node पर हो सकती है। Twilio से भागीदारी में [वॉइस कॉल Plugin](/hi/plugins/voice-call) के माध्यम से फ़ोन नंबर और PIN/DTMF डायल किया जाता है; यह सीधे Meet URL डायल नहीं कर सकता।
- `mode: "agent"` (डिफ़ॉल्ट) किसी रीयलटाइम प्रदाता की मदद से प्रतिभागियों की वाणी का ट्रांसक्रिप्शन करता है, उसे कॉन्फ़िगर किए गए OpenClaw एजेंट तक भेजता है और सामान्य OpenClaw TTS से उत्तर बोलता है। `mode: "bidi"` किसी रीयलटाइम वॉइस मॉडल को सीधे उत्तर देने देता है। `mode: "transcribe"` केवल अवलोकन के लिए जुड़ता है और प्रत्युत्तर में बोलता नहीं है।
- Plugin के कॉल में जुड़ने पर सहमति की कोई स्वचालित घोषणा नहीं होती।
- CLI कमांड `googlemeet` है; `meet` व्यापक एजेंट टेलीकॉन्फ़्रेंस वर्कफ़्लो के लिए आरक्षित है।

## त्वरित शुरुआत

Plugin और स्थानीय ऑडियो निर्भरताएँ इंस्टॉल करें, फिर रीयलटाइम प्रदाता की कुंजी सेट करें। `agent` मोड के लिए OpenAI डिफ़ॉल्ट ट्रांसक्रिप्शन प्रदाता है; Google Gemini Live, `bidi`-मोड वॉइस प्रदाता के रूप में उपलब्ध है:

```bash
openclaw plugins install npm:@openclaw/google-meet
brew install blackhole-2ch sox
export OPENAI_API_KEY=sk-...
# केवल तब आवश्यक है जब bidi मोड के लिए realtime.voiceProvider "google" हो
export GEMINI_API_KEY=...
```

`blackhole-2ch` वह `BlackHole 2ch` वर्चुअल ऑडियो डिवाइस इंस्टॉल करता है, जिसके माध्यम से Chrome ऑडियो रूट करता है। macOS पर डिवाइस उपलब्ध होने से पहले Homebrew के इंस्टॉलर के बाद रीबूट करना आवश्यक है:

```bash
sudo reboot
```

रीबूट के बाद दोनों घटकों को सत्यापित करें:

```bash
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

इंस्टॉलेशन के बाद Plugin डिफ़ॉल्ट रूप से सक्षम रहता है। इसे अनुकूलित करने के लिए ही प्रविष्टि जोड़ें:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        config: {},
      },
    },
  },
}
```

यदि आप Plugin को सक्रिय नहीं रखना चाहते, तो `openclaw plugins disable google-meet` चलाएँ।

सेटअप जाँचें, फिर जुड़ें:

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

`setup` का आउटपुट एजेंट-पठनीय और मोड/ट्रांसपोर्ट के अनुरूप होता है: यह Chrome प्रोफ़ाइल, Node पिनिंग और रीयलटाइम Chrome जॉइन के लिए BlackHole/SoX ऑडियो ब्रिज तथा विलंबित-परिचय जाँच की रिपोर्ट देता है। केवल-अवलोकन जॉइन में रीयलटाइम पूर्वापेक्षाएँ छोड़ दी जाती हैं:

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
```

जब Twilio प्रत्यायोजन कॉन्फ़िगर हो, तब `setup` यह भी बताता है कि `voice-call`, Twilio क्रेडेंशियल और सार्वजनिक Webhook एक्सपोज़र तैयार हैं या नहीं। एजेंट के जुड़ने से पहले किसी भी `ok: false` जाँच को उस ट्रांसपोर्ट/मोड के लिए अवरोधक मानें। मशीन-पठनीय आउटपुट के लिए `--json` और किसी विशिष्ट ट्रांसपोर्ट की पहले से प्रारंभिक जाँच के लिए `--transport chrome|chrome-node|twilio` का उपयोग करें:

```bash
openclaw googlemeet setup --transport twilio
```

या एजेंट को `google_meet` टूल के माध्यम से जुड़ने दें:

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

गैर-macOS Gateway होस्ट पर `google_meet`, आर्टिफ़ैक्ट, कैलेंडर, सेटअप, ट्रांसक्राइब, Twilio और `chrome-node` कार्रवाइयों के लिए उपलब्ध रहता है, लेकिन स्थानीय Chrome प्रत्युत्तर (`transport: "chrome"` के साथ `mode: "agent"` या `"bidi"`) ऑडियो ब्रिज तक पहुँचने से पहले ही अवरुद्ध हो जाता है, क्योंकि यह पथ वर्तमान में macOS `BlackHole 2ch` पर निर्भर है। इसके बजाय `mode: "transcribe"`, Twilio डायल-इन या macOS `chrome-node` होस्ट का उपयोग करें।

### मीटिंग बनाएँ

```bash
openclaw googlemeet create --transport chrome-node --mode agent
openclaw googlemeet create --no-join
```

`create` के दो पथ हैं, जिनकी रिपोर्ट परिणाम के `source` फ़ील्ड में दी जाती है:

- **`api`**: Google Meet OAuth क्रेडेंशियल कॉन्फ़िगर होने पर उपयोग होता है। यह निर्धारक है; ब्राउज़र UI की स्थिति पर निर्भर नहीं करता।
- **`browser`**: OAuth क्रेडेंशियल के बिना उपयोग होता है। OpenClaw पिन किए गए Chrome Node पर `https://meet.google.com/new` खोलता है और Google के किसी वास्तविक मीटिंग-कोड URL पर रीडायरेक्ट करने की प्रतीक्षा करता है; उस Node पर OpenClaw Chrome प्रोफ़ाइल पहले से Google में साइन-इन होनी चाहिए। जॉइन और क्रिएट, नया टैब खोलने से पहले मौजूदा Meet टैब (या प्रगति पर मौजूद `.../new` / Google खाता प्रॉम्प्ट टैब) का पुनः उपयोग करते हैं; टैब मिलान में `authuser` जैसी अहानिकर क्वेरी स्ट्रिंग को अनदेखा किया जाता है।

`create` डिफ़ॉल्ट रूप से जुड़ता है और जॉइन सत्र के साथ `joined: true` लौटाता है। केवल URL बनाने के लिए `--no-join` (CLI) या `"join": false` (टूल) पास करें।

API द्वारा बनाए गए कक्षों के लिए Google खाते का डिफ़ॉल्ट विरासत में लेने के बजाय स्पष्ट एक्सेस नीति सेट करें:

```bash
openclaw googlemeet create --access-type OPEN --transport chrome-node --mode agent
```

| `--access-type` | बिना अनुमति माँगे कौन जुड़ सकता है                                  |
| --------------- | ------------------------------------------------------------------- |
| `OPEN`          | Meet URL वाला कोई भी व्यक्ति                                       |
| `TRUSTED`       | होस्ट संगठन के विश्वसनीय उपयोगकर्ता, आमंत्रित बाहरी उपयोगकर्ता और डायल-इन उपयोगकर्ता |
| `RESTRICTED`    | केवल आमंत्रित व्यक्ति                                               |

यह केवल API द्वारा बनाए गए कक्षों पर लागू होता है, इसलिए OAuth कॉन्फ़िगर होना आवश्यक है। यदि आपने इस विकल्प के उपलब्ध होने से पहले प्रमाणीकरण किया था, तो अपनी OAuth सहमति स्क्रीन में `meetings.space.settings` स्कोप जोड़ने के बाद `openclaw googlemeet auth login --json` दोबारा चलाएँ।

यदि ब्राउज़र फ़ॉलबैक Google लॉगिन या Meet अनुमति अवरोधक का सामना करता है, तो टूल `manualActionReason`, `manualActionMessage` और `browser.nodeId`/`browser.targetId`/`browserUrl` के साथ `manualActionRequired: true` लौटाता है। उस संदेश की रिपोर्ट करें और ऑपरेटर द्वारा ब्राउज़र चरण पूरा किए जाने तक नए Meet टैब खोलना बंद रखें।

### केवल-अवलोकन जॉइन

डुप्लेक्स रीयलटाइम ब्रिज छोड़ने के लिए `"mode": "transcribe"` सेट करें (BlackHole/SoX की आवश्यकता नहीं, कोई प्रत्युत्तर नहीं)। ट्रांसक्राइब-मोड Chrome जॉइन में OpenClaw की माइक्रोफ़ोन/कैमरा अनुमति और Meet का **Use microphone** पथ भी छोड़ दिया जाता है; यदि Meet ऑडियो-विकल्प मध्यवर्ती स्क्रीन दिखाता है, तो स्वचालन पहले **Continue without microphone** आज़माता है। प्रबंधित Chrome ट्रांसपोर्ट हर मोड में सर्वोत्तम-प्रयास वाला Meet कैप्शन पर्यवेक्षक इंस्टॉल करते हैं, ताकि लाइव एजेंट-परामर्श पथ को बदले बिना स्थायी नोट उपलब्ध हों। `googlemeet status --json` और `googlemeet doctor`, `captioning`, `captionsEnabledAttempted`, `transcriptLines`, `lastCaptionAt`, `lastCaptionSpeaker`, `lastCaptionText` और `recentTranscript` टेल की रिपोर्ट देते हैं।

सीमित सत्र ट्रांसक्रिप्ट के लिए, सटीक रूप से ट्रैक किया गया Meet टैब पढ़ें:

```bash
openclaw googlemeet transcript <session-id>
openclaw googlemeet transcript <session-id> --since <next-index> --json
```

पर्यवेक्षक Meet पेज पर अधिकतम 2,000 पूर्ण कैप्शन पंक्तियाँ रखता है। कैप्शन पंक्ति पूरी होने तक दिखाई देने वाला क्रमिक टेक्स्ट स्थिति स्वास्थ्य टेल में बना रहता है, इसलिए `nextIndex` सहेजने पर बाद का टेक्स्ट विस्तार नहीं छूट सकता; बाहर निकलने पर स्नैपशॉट से पहले दृश्यमान पंक्तियाँ अंतिम रूप ले लेती हैं। सीमा पार होने पर `droppedLines` शुरुआत से खोई हुई पंक्तियों की रिपोर्ट देता है। सीमित `googlemeet transcript` टेल अब भी केवल हाल में समाप्त हुए चार सत्र रखता है और Gateway के साथ रीसेट होता है। अलग से, OpenClaw पूरी मीटिंग के दौरान पूर्ण कैप्शन पंक्तियाँ साझा स्थिति डेटाबेस में जोड़ता है और बाहर निकलते समय व्युत्पन्न सारांश लिखता है। इन स्थायी नोटों का निरीक्षण या निर्यात करने के लिए [`openclaw transcripts`](/hi/cli/transcripts) का उपयोग करें।

स्वचालित नोट डिफ़ॉल्ट रूप से सक्षम होते हैं। स्थायी नोटों को वैश्विक रूप से
अक्षम करने के लिए `transcripts.enabled: false` सेट करें; स्पष्ट `transcribe` मोड फिर भी केवल
अपनी सीमित लाइव टेल दिखाता है। Twilio जॉइन में ब्राउज़र कैप्शन स्ट्रीम नहीं होती और
वे इस पथ से कैप्चर नहीं किए जाते।

हाँ/नहीं सुनने की जाँच के लिए:

```bash
openclaw googlemeet test-listen <meet-url> --transport chrome-node
```

यह ट्रांसक्राइब मोड में जुड़ता है, नए कैप्शन/ट्रांसक्रिप्ट बदलाव की प्रतीक्षा करता है और `listenVerified`, `listenTimedOut`, मैन्युअल-कार्रवाई फ़ील्ड तथा वर्तमान कैप्शन स्वास्थ्य लौटाता है।

### रीयलटाइम सत्र स्वास्थ्य

प्रत्युत्तर सत्रों के दौरान `google_meet` स्थिति Chrome/ऑडियो ब्रिज के स्वास्थ्य की रिपोर्ट देती है: `inCall`, `manualActionRequired`, `providerConnected`, `realtimeReady`, `audioInputActive`, `audioOutputActive`, अंतिम इनपुट/आउटपुट टाइमस्टैम्प, बाइट काउंटर और ब्रिज-बंद स्थिति। प्रबंधित Chrome सत्र परिचय/परीक्षण वाक्यांश तभी बोलते हैं, जब स्वास्थ्य `inCall: true` रिपोर्ट करता है; अन्यथा `speechReady: false` होता है और वाणी प्रयास को चुपचाप निष्प्रभावी करने के बजाय अवरुद्ध कर दिया जाता है।

स्थानीय Chrome, साइन-इन की हुई OpenClaw ब्राउज़र प्रोफ़ाइल के माध्यम से जुड़ता है और माइक/स्पीकर पथ के लिए `BlackHole 2ch` की आवश्यकता होती है। पहली स्मोक जाँच के लिए एक BlackHole डिवाइस पर्याप्त है, लेकिन इससे प्रतिध्वनि हो सकती है; स्वच्छ डुप्लेक्स ऑडियो के लिए अलग-अलग वर्चुअल डिवाइस या Loopback-शैली के ग्राफ़ का उपयोग करें।

## स्थानीय Gateway + Parallels Chrome

केवल macOS VM को Chrome उपलब्ध कराने के लिए उसके भीतर पूर्ण Gateway या मॉडल API कुंजी की आवश्यकता नहीं होती। Gateway और एजेंट स्थानीय रूप से चलाएँ; VM में Node होस्ट चलाएँ।

| कहाँ चलता है           | क्या                                                                                            |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| Gateway होस्ट         | OpenClaw Gateway, एजेंट वर्कस्पेस, मॉडल/API कुंजियाँ, रीयलटाइम प्रदाता, Google Meet Plugin कॉन्फ़िगरेशन |
| Parallels macOS VM   | OpenClaw CLI/Node होस्ट, Chrome, SoX, BlackHole 2ch, Google में साइन-इन की हुई Chrome प्रोफ़ाइल        |
| VM में आवश्यक नहीं | Gateway सेवा, एजेंट कॉन्फ़िगरेशन, मॉडल प्रदाता सेटअप                                             |

VM निर्भरताएँ इंस्टॉल करें, रीबूट करें और सत्यापित करें:

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

VM में Plugin इंस्टॉल करें, जहाँ यह डिफ़ॉल्ट रूप से सक्षम होता है, और Node होस्ट शुरू करें:

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw node run --host <gateway-host> --port 18789 --display-name parallels-macos
```

यदि `<gateway-host>` TLS के बिना LAN IP है, तो उस विश्वसनीय निजी नेटवर्क के लिए स्पष्ट रूप से अनुमति दें:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

LaunchAgent के रूप में इंस्टॉल करते समय भी इसी फ़्लैग का उपयोग करें (यह प्रक्रिया का परिवेश है, जो इंस्टॉल कमांड में मौजूद होने पर LaunchAgent परिवेश में संग्रहित होता है, न कि `openclaw.json` सेटिंग):

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install --host <gateway-lan-ip> --port 18789 --display-name parallels-macos --force
openclaw node restart
```

Gateway होस्ट से Node को स्वीकृत करें, फिर पुष्टि करें कि यह `googlemeet.chrome` और ब्राउज़र क्षमता/`browser.proxy`, दोनों को विज्ञापित करता है:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Meet को उस Node के माध्यम से रूट करें:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["googlemeet.chrome", "browser.proxy"] },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          chrome: {
            guestName: "OpenClaw Agent",
            autoJoin: true,
            reuseExistingTab: true,
          },
          chromeNode: {
            node: "parallels-macos",
          },
        },
      },
    },
  },
}
```

अब Gateway होस्ट से सामान्य रूप से जुड़ें:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

एक-कमांड स्मोक जाँच के लिए, जो सत्र बनाती या पुनः उपयोग करती है, ज्ञात वाक्यांश बोलती है और सत्र स्वास्थ्य प्रिंट करती है:

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij
```

रीयलटाइम जुड़ने के दौरान, ब्राउज़र स्वचालन अतिथि का नाम भरता है, Join/Ask to join पर क्लिक करता है, और Meet का पहली बार दिखाई देने वाला "Use microphone" संकेत स्वीकार करता है (या केवल-अवलोकन जुड़ाव और केवल-ब्राउज़र मीटिंग निर्माण के दौरान "Continue without microphone" चुनता है)। यदि प्रोफ़ाइल साइन आउट है, Meet होस्ट की स्वीकृति की प्रतीक्षा कर रहा है, Chrome को माइक/कैमरा अनुमति चाहिए, या Meet किसी अनसुलझे संकेत पर अटका है, तो परिणाम `manualActionReason` और `manualActionMessage` के साथ `manualActionRequired: true` रिपोर्ट करता है। पुनः प्रयास करना रोकें, वह संदेश तथा `browserUrl`/`browserTitle` रिपोर्ट करें, और मैन्युअल कार्रवाई पूरी होने के बाद ही पुनः प्रयास करें।

यदि `chromeNode.node` छोड़ा गया है, तो OpenClaw केवल तभी स्वतः चयन करता है जब ठीक एक कनेक्टेड Node `googlemeet.chrome` और ब्राउज़र नियंत्रण दोनों की उपलब्धता घोषित करता है; कई सक्षम Node कनेक्टेड होने पर `chromeNode.node` (Node आईडी, प्रदर्शन नाम या रिमोट IP) को पिन करें।

### सामान्य विफलता जाँच

| लक्षण                                                  | समाधान                                                                                                                                                                                                                                                                                   |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Configured Google Meet node ... is not usable: offline` | पिन किया गया Node ज्ञात है, लेकिन अनुपलब्ध है। सेटअप अवरोधक रिपोर्ट करें; कहे जाने तक चुपचाप किसी अन्य ट्रांसपोर्ट पर फ़ॉलबैक न करें।                                                                                                                                                      |
| `No connected Google Meet-capable node`                  | VM में `npm:@openclaw/google-meet` इंस्टॉल करें, `openclaw plugins enable browser` चलाएँ, `openclaw node run` शुरू करें और पेयरिंग स्वीकृत करें। यदि Google Meet स्पष्ट रूप से अक्षम किया गया था, तो उसे भी सक्षम करें। पुष्टि करें कि `gateway.nodes.commands.allow` में `googlemeet.chrome` और `browser.proxy` शामिल हैं। |
| `BlackHole 2ch audio device not found`                   | जाँचे जा रहे होस्ट पर `blackhole-2ch` इंस्टॉल करें और रीबूट करें।                                                                                                                                                                                                                         |
| `BlackHole 2ch audio device not found on the node`       | VM में `blackhole-2ch` इंस्टॉल करें और VM को रीबूट करें।                                                                                                                                                                                                                                  |
| Chrome खुलता है, लेकिन जुड़ नहीं सकता                             | VM में ब्राउज़र प्रोफ़ाइल में साइन इन करें, या `chrome.guestName` सेट रखें। अतिथि का स्वतः जुड़ाव Node ब्राउज़र प्रॉक्सी के माध्यम से OpenClaw ब्राउज़र स्वचालन का उपयोग करता है; Node के `browser.defaultProfile` (या किसी नामित मौजूदा-सत्र प्रोफ़ाइल) को इच्छित प्रोफ़ाइल की ओर निर्देशित करें।                   |
| डुप्लिकेट Meet टैब                                      | `chrome.reuseExistingTab: true` छोड़ दें। OpenClaw उसी URL के मौजूदा टैब को सक्रिय करता है, और दूसरा टैब खोलने से पहले निर्माण प्रक्रिया में चल रहे `.../new` या Google खाता संकेत टैब का पुनः उपयोग करता है।                                                                                        |
| कोई ऑडियो नहीं                                                 | Meet के माइक/स्पीकर को OpenClaw द्वारा उपयोग किए जाने वाले वर्चुअल ऑडियो पथ से रूट करें; स्वच्छ डुप्लेक्स ऑडियो के लिए अलग वर्चुअल डिवाइस या Loopback-शैली रूटिंग का उपयोग करें।                                                                                                                                |

## इंस्टॉल संबंधी टिप्पणियाँ

Chrome टॉक-बैक का डिफ़ॉल्ट दो बाहरी टूल का उपयोग करता है, जिन्हें OpenClaw बंडल या पुनर्वितरित नहीं करता; इन्हें Homebrew के माध्यम से होस्ट निर्भरताओं के रूप में इंस्टॉल करें:

- `sox`: कमांड-लाइन ऑडियो उपयोगिता। Plugin डिफ़ॉल्ट 24 kHz PCM16 ऑडियो ब्रिज के लिए स्पष्ट CoreAudio डिवाइस कमांड जारी करता है।
- `blackhole-2ch`: `BlackHole 2ch` डिवाइस प्रदान करने वाला macOS वर्चुअल ऑडियो ड्राइवर, जिसके माध्यम से Chrome/Meet रूट होते हैं।

SoX को `LGPL-2.0-only AND GPL-2.0-only` के अंतर्गत लाइसेंस प्राप्त है; BlackHole GPL-3.0 है। यदि आप ऐसा इंस्टॉलर या उपकरण बनाते हैं जो BlackHole को OpenClaw के साथ बंडल करता है, तो BlackHole के अपस्ट्रीम लाइसेंस की समीक्षा करें या Existential Audio से अलग लाइसेंस प्राप्त करें।

## ट्रांसपोर्ट

| ट्रांसपोर्ट     | इसका उपयोग तब करें जब                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------- |
| `chrome`      | Chrome/ऑडियो Gateway होस्ट पर सक्रिय हों                                                        |
| `chrome-node` | Chrome/ऑडियो पेयर किए गए Node पर सक्रिय हों (उदाहरण के लिए Parallels macOS VM)                        |
| `twilio`      | Chrome भागीदारी उपलब्ध न होने पर Voice Call Plugin के माध्यम से फ़ोन डायल-इन फ़ॉलबैक |

### Chrome

OpenClaw ब्राउज़र नियंत्रण के माध्यम से Meet URL खोलता है और साइन-इन की गई OpenClaw ब्राउज़र प्रोफ़ाइल के रूप में जुड़ता है। macOS पर, Plugin लॉन्च से पहले `BlackHole 2ch` की जाँच करता है और कॉन्फ़िगर होने पर Chrome खोलने से पहले ऑडियो ब्रिज स्वास्थ्य/स्टार्टअप कमांड चलाता है। स्थानीय Chrome के लिए `browser.defaultProfile` से प्रोफ़ाइल चुनें; इसके बजाय `chrome.browserProfile` को `chrome-node` होस्ट को भेजा जाता है।

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome-node
```

Chrome माइक/स्पीकर ऑडियो स्थानीय OpenClaw ऑडियो ब्रिज के माध्यम से रूट होता है। यदि `BlackHole 2ch` इंस्टॉल नहीं है, तो बिना ऑडियो पथ के जुड़ने के बजाय जुड़ाव सेटअप त्रुटि के साथ विफल हो जाता है।

### Twilio

[Voice call Plugin](/hi/plugins/voice-call) को सौंपा गया एक सख्त डायल प्लान। यह फ़ोन नंबरों के लिए Meet पेजों को पार्स नहीं करता; Google Meet को मीटिंग के लिए फ़ोन डायल-इन नंबर और PIN उपलब्ध कराना होगा।

Voice Call को Chrome Node पर नहीं, Gateway होस्ट पर सक्षम करें:

```json5
{
  plugins: {
    allow: ["google-meet", "voice-call", "google"],
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          // या यदि Twilio डिफ़ॉल्ट होना चाहिए, तो "twilio" सेट करें
        },
      },
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          inboundPolicy: "allowlist",
          realtime: {
            enabled: true,
            provider: "google",
            instructions: "OpenClaw एजेंट के रूप में इस Google Meet से जुड़ें। संक्षिप्त रहें।",
            toolPolicy: "safe-read-only",
            providers: {
              google: {
                silenceDurationMs: 500,
                startSensitivity: "high",
              },
            },
          },
        },
      },
      google: {
        enabled: true,
      },
    },
  },
}
```

गोपनीय मानों को `openclaw.json` से बाहर रखने के लिए पर्यावरण के माध्यम से Twilio क्रेडेंशियल प्रदान करें:

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
export GEMINI_API_KEY=...
```

यदि OpenAI रीयलटाइम वॉइस प्रदाता है, तो इसके बजाय `OPENAI_API_KEY` के साथ `realtime.provider: "openai"` का उपयोग करें।

`voice-call` सक्षम करने के बाद Gateway को पुनः आरंभ या रीलोड करें; Plugin कॉन्फ़िगरेशन परिवर्तन रीलोड होने तक प्रभावी नहीं होते। सत्यापित करें:

```bash
openclaw config validate
openclaw plugins list | grep -E 'google-meet|voice-call'
openclaw googlemeet setup
```

Twilio प्रत्यायोजन जुड़ा होने पर, `googlemeet setup` में `twilio-voice-call-plugin`, `twilio-voice-call-credentials` और `twilio-voice-call-webhook` जाँच शामिल होती हैं।

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

कस्टम अनुक्रम के लिए `--dtmf-sequence` का उपयोग करें, जिसमें PIN से पहले विराम के लिए प्रारंभिक `w` या कॉमा हों:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

## OAuth और प्रीफ़्लाइट

Meet लिंक बनाने के लिए OAuth वैकल्पिक है, क्योंकि `googlemeet create` ब्राउज़र स्वचालन पर फ़ॉलबैक कर सकता है। आधिकारिक API निर्माण, स्पेस समाधान या Meet Media API प्रीफ़्लाइट के लिए OAuth कॉन्फ़िगर करें। Chrome/Chrome-node जुड़ाव कभी भी OAuth पर निर्भर नहीं होते; वे साइन-इन की गई Chrome प्रोफ़ाइल, BlackHole/SoX और (`chrome-node` के लिए) किसी भी स्थिति में कनेक्टेड Node का उपयोग करते हैं।

### Google क्रेडेंशियल बनाएँ

Google Cloud Console में:

<Steps>
<Step title="कोई प्रोजेक्ट बनाएँ या चुनें">
</Step>
<Step title="Google Meet REST API सक्षम करें">
</Step>
<Step title="OAuth सहमति स्क्रीन कॉन्फ़िगर करें">
Google Workspace संगठन के लिए Internal सबसे सरल है। व्यक्तिगत/परीक्षण सेटअप के लिए External काम करता है; जब ऐप Testing में हो, तब उसे अधिकृत करने वाले प्रत्येक Google खाते को टेस्ट उपयोगकर्ता के रूप में जोड़ें।
</Step>
<Step title="अनुरोधित स्कोप जोड़ें">
- `https://www.googleapis.com/auth/meetings.space.created`
- `https://www.googleapis.com/auth/meetings.space.readonly`
- `https://www.googleapis.com/auth/meetings.space.settings`
- `https://www.googleapis.com/auth/meetings.conference.media.readonly`
- `https://www.googleapis.com/auth/calendar.events.readonly` (Calendar खोज)
- `https://www.googleapis.com/auth/drive.meet.readonly` (ट्रांसक्रिप्ट/स्मार्ट-नोट दस्तावेज़ बॉडी निर्यात)

</Step>
<Step title="OAuth क्लाइंट ID बनाएँ">
एप्लिकेशन प्रकार **Web application**। अधिकृत रीडायरेक्ट URI:

```text
http://localhost:8085/oauth2callback
```

</Step>
<Step title="क्लाइंट ID और क्लाइंट सीक्रेट कॉपी करें">
</Step>
</Steps>

`spaces.create` के लिए `meetings.space.created` आवश्यक है। `meetings.space.readonly` Meet URL/कोड को स्पेस में हल करता है। `meetings.space.settings` OpenClaw को API कक्ष निर्माण के दौरान `accessType` जैसी `SpaceConfig` सेटिंग पास करने देता है। `meetings.conference.media.readonly` Meet Media API प्रीफ़्लाइट और मीडिया कार्य के लिए है; वास्तविक Media API उपयोग के लिए Google को Developer Preview नामांकन की आवश्यकता हो सकती है। `calendar.events.readonly` की आवश्यकता केवल `--today`/`--event` Calendar खोज के लिए है। `drive.meet.readonly` की आवश्यकता केवल `--include-doc-bodies` निर्यात के लिए है। यदि आपको केवल ब्राउज़र-आधारित Chrome जुड़ाव चाहिए, तो OAuth को पूरी तरह छोड़ दें।

### रीफ़्रेश टोकन बनाएँ

`oauth.clientId` और वैकल्पिक रूप से `oauth.clientSecret` कॉन्फ़िगर करें (या उन्हें पर्यावरण चर के रूप में पास करें), फिर चलाएँ:

```bash
openclaw googlemeet auth login --json
```

यह `http://localhost:8085/oauth2callback` पर localhost कॉलबैक के साथ PKCE प्रवाह चलाता है और रीफ़्रेश टोकन वाला `oauth` कॉन्फ़िगरेशन ब्लॉक प्रिंट करता है। जब ब्राउज़र स्थानीय कॉलबैक तक नहीं पहुँच सकता, तब कॉपी/पेस्ट प्रवाह के लिए `--manual` जोड़ें:

```bash
OPENCLAW_GOOGLE_MEET_CLIENT_ID="your-client-id" \
OPENCLAW_GOOGLE_MEET_CLIENT_SECRET="your-client-secret" \
openclaw googlemeet auth login --json --manual
```

JSON आउटपुट:

```json
{
  "oauth": {
    "clientId": "your-client-id",
    "clientSecret": "your-client-secret",
    "refreshToken": "refresh-token",
    "accessToken": "access-token",
    "expiresAt": 1770000000000
  },
  "scope": "..."
}
```

Plugin कॉन्फ़िगरेशन के अंतर्गत `oauth` ऑब्जेक्ट संग्रहीत करें:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          oauth: {
            clientId: "your-client-id",
            clientSecret: "your-client-secret",
            refreshToken: "refresh-token",
          },
        },
      },
    },
  },
}
```

जब आप रीफ़्रेश टोकन को कॉन्फ़िगरेशन में नहीं रखना चाहते, तब पर्यावरण चर को प्राथमिकता दें; पहले कॉन्फ़िगरेशन हल किया जाता है, फिर फ़ॉलबैक के रूप में पर्यावरण का उपयोग होता है। यदि आपने मीटिंग निर्माण, Calendar खोज या दस्तावेज़-बॉडी निर्यात समर्थन उपलब्ध होने से पहले प्रमाणीकरण किया था, तो `openclaw googlemeet auth login --json` फिर से चलाएँ, ताकि रीफ़्रेश टोकन वर्तमान स्कोप समूह को कवर करे।

### डॉक्टर से OAuth सत्यापित करें

```bash
openclaw googlemeet doctor --oauth --json
```

यह जाँचता है कि OAuth कॉन्फ़िग मौजूद है और रिफ़्रेश टोकन, Chrome रनटाइम लोड किए बिना या कनेक्टेड Node की आवश्यकता के बिना, एक ऐक्सेस टोकन जारी कर सकता है। रिपोर्ट में केवल स्थिति फ़ील्ड (`ok`, `configured`, `tokenSource`, `expiresAt`, जाँच संदेश) शामिल होते हैं और यह कभी भी ऐक्सेस टोकन, रिफ़्रेश टोकन या क्लाइंट सीक्रेट प्रिंट नहीं करती।

| जाँच                 | अर्थ                                                                             |
| -------------------- | -------------------------------------------------------------------------------- |
| `oauth-config`       | `oauth.clientId` के साथ `oauth.refreshToken`, या कैश किया गया ऐक्सेस टोकन, मौजूद है |
| `oauth-token`        | कैश किया गया ऐक्सेस टोकन अब भी मान्य है, या रिफ़्रेश टोकन ने नया टोकन जारी किया |
| `meet-spaces-get`    | वैकल्पिक `--meeting` जाँच ने किसी मौजूदा Meet स्पेस का समाधान किया |
| `meet-spaces-create` | वैकल्पिक `--create-space` जाँच ने नया Meet स्पेस बनाया |

साइड इफ़ेक्ट उत्पन्न करने वाली क्रिएट जाँच से Meet API का सक्षम होना और `spaces.create` स्कोप प्रमाणित करें:

```bash
openclaw googlemeet doctor --oauth --create-space --json
```

किसी मौजूदा स्पेस के लिए रीड ऐक्सेस प्रमाणित करें:

```bash
openclaw googlemeet doctor --oauth --meeting https://meet.google.com/abc-defg-hij --json
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
```

इन जाँचों से प्राप्त `403` का सामान्यतः अर्थ है कि Meet REST API अक्षम है, रिफ़्रेश टोकन में आवश्यक स्कोप नहीं है या Google खाता उस स्पेस को ऐक्सेस नहीं कर सकता। रिफ़्रेश-टोकन त्रुटि का अर्थ है कि `openclaw googlemeet auth login --json` फिर से चलाएँ और नया `oauth` ब्लॉक संग्रहीत करें।

ब्राउज़र फ़ॉलबैक के लिए OAuth की आवश्यकता नहीं है; वहाँ Google प्रमाणीकरण, OpenClaw कॉन्फ़िग से नहीं, बल्कि चयनित Node पर साइन-इन किए गए Chrome प्रोफ़ाइल से आता है।

इन एनवायरनमेंट वेरिएबल को फ़ॉलबैक के रूप में स्वीकार किया जाता है:

- `OPENCLAW_GOOGLE_MEET_CLIENT_ID` या `GOOGLE_MEET_CLIENT_ID`
- `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET` या `GOOGLE_MEET_CLIENT_SECRET`
- `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` या `GOOGLE_MEET_REFRESH_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN` या `GOOGLE_MEET_ACCESS_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` या `GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT`
- `OPENCLAW_GOOGLE_MEET_DEFAULT_MEETING` या `GOOGLE_MEET_DEFAULT_MEETING`
- `OPENCLAW_GOOGLE_MEET_PREVIEW_ACK` या `GOOGLE_MEET_PREVIEW_ACK`

### आर्टिफ़ैक्ट का समाधान, प्रीफ़्लाइट और पठन

```bash
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet preflight --meeting https://meet.google.com/abc-defg-hij
```

Meet द्वारा कॉन्फ़्रेंस रिकॉर्ड बनाए जाने के बाद:

```bash
openclaw googlemeet artifacts --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet attendance --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet export --meeting https://meet.google.com/abc-defg-hij --output ./meet-export
```

`--meeting` के साथ, `artifacts` और `attendance` डिफ़ॉल्ट रूप से नवीनतम कॉन्फ़्रेंस रिकॉर्ड का उपयोग करते हैं; सुरक्षित रखे गए प्रत्येक रिकॉर्ड के लिए `--all-conference-records` पास करें।

Calendar लुकअप आर्टिफ़ैक्ट पढ़ने से पहले Google Calendar से मीटिंग URL का समाधान करता है (इसके लिए ऐसा रिफ़्रेश टोकन आवश्यक है जिसमें Calendar इवेंट का केवल-पढ़ने योग्य स्कोप शामिल हो):

```bash
openclaw googlemeet latest --today
openclaw googlemeet calendar-events --today --json
openclaw googlemeet artifacts --event "Weekly sync"
openclaw googlemeet attendance --today --format csv --output attendance.csv
```

`--today` Meet लिंक वाले इवेंट के लिए आज का `primary` कैलेंडर खोजता है; `--event <query>` मेल खाने वाला इवेंट टेक्स्ट खोजता है; `--calendar <id>` किसी गैर-प्राथमिक कैलेंडर को लक्षित करता है। `calendar-events` मेल खाने वाले इवेंट का पूर्वावलोकन करता है और चिह्नित करता है कि `latest`/`artifacts`/`attendance`/`export` किसे चुनेगा।

यदि कॉन्फ़्रेंस रिकॉर्ड ID पहले से ज्ञात है, तो उसे सीधे संबोधित करें:

```bash
openclaw googlemeet latest --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 --json
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 --json
```

API द्वारा बनाए गए स्पेस के लिए रूम बंद करें:

```bash
openclaw googlemeet end-active-conference https://meet.google.com/abc-defg-hij
```

यह `spaces.endActiveConference` को कॉल करता है और ऐसे स्पेस के लिए `meetings.space.created` स्कोप वाले OAuth की आवश्यकता होती है जिसे अधिकृत खाता प्रबंधित कर सकता है। यह Meet URL, मीटिंग कोड या `spaces/{id}` स्वीकार करता है और पहले उसका API स्पेस रिसोर्स में समाधान करता है। यह `googlemeet leave` से अलग है: `leave` OpenClaw की स्थानीय/सेशन भागीदारी रोकता है; `end-active-conference` Google Meet से उस स्पेस की सक्रिय कॉन्फ़्रेंस समाप्त करने के लिए कहता है।

पठनीय रिपोर्ट लिखें:

```bash
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 \
  --format markdown --output meet-artifacts.md
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 \
  --format csv --output meet-attendance.csv
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --zip --output meet-export
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --dry-run
```

जब Google इन्हें उपलब्ध कराता है, तो `artifacts` कॉन्फ़्रेंस रिकॉर्ड मेटाडेटा के साथ प्रतिभागी, रिकॉर्डिंग, ट्रांसक्रिप्ट, संरचित ट्रांसक्रिप्ट-एंट्री और स्मार्ट-नोट रिसोर्स मेटाडेटा लौटाता है। `--no-transcript-entries` बड़ी मीटिंग के लिए एंट्री लुकअप छोड़ देता है। `attendance` प्रतिभागियों को प्रतिभागी-सेशन पंक्तियों में विस्तारित करता है, जिनमें पहली/अंतिम बार देखे जाने का समय, कुल सेशन अवधि, देर से आने/जल्दी जाने के फ़्लैग और साइन-इन किए गए उपयोगकर्ता या डिस्प्ले नाम के आधार पर मर्ज किए गए डुप्लिकेट प्रतिभागी रिसोर्स होते हैं; `--no-merge-duplicates` रॉ रिसोर्स को अलग रखता है, और `--late-after-minutes`/`--early-before-minutes` थ्रेशोल्ड समायोजित करते हैं।

`export`, `summary.md`, `attendance.csv`, `transcript.md`, `artifacts.json`, `attendance.json` और `manifest.json` वाली फ़ोल्डर लिखता है। `manifest.json` चुने गए इनपुट, एक्सपोर्ट विकल्पों, कॉन्फ़्रेंस रिकॉर्ड, आउटपुट फ़ाइलों, गणनाओं, टोकन स्रोत, उपयोग किए गए किसी Calendar इवेंट और आंशिक पुनर्प्राप्ति चेतावनियों को रिकॉर्ड करता है। `--zip` फ़ोल्डर के पास एक पोर्टेबल आर्काइव भी लिखता है। `--include-doc-bodies`, Drive `files.export` के माध्यम से लिंक किए गए ट्रांसक्रिप्ट/स्मार्ट-नोट Google Docs टेक्स्ट को एक्सपोर्ट करता है (इसके लिए Drive Meet का केवल-पढ़ने योग्य स्कोप आवश्यक है); इसके बिना, एक्सपोर्ट में केवल Meet मेटाडेटा और संरचित ट्रांसक्रिप्ट एंट्री शामिल होती हैं। आंशिक आर्टिफ़ैक्ट विफलता (स्मार्ट-नोट सूचीकरण, ट्रांसक्रिप्ट-एंट्री या दस्तावेज़-बॉडी त्रुटि) पूरे एक्सपोर्ट को विफल करने के बजाय चेतावनी को सारांश/मैनिफ़ेस्ट में बनाए रखती है। `--dry-run` वही डेटा फ़ेच करता है और फ़ोल्डर या ZIP बनाए बिना मैनिफ़ेस्ट JSON प्रिंट करता है।

एजेंट `google_meet` टूल (`export`, `create` के साथ `accessType`, `end_active_conference`, `test_listen`) के माध्यम से उन्हीं कार्रवाइयों का उपयोग करते हैं; [टूल](#tool) देखें।

### लाइव स्मोक टेस्ट

```bash
OPENCLAW_LIVE_TEST=1 \
OPENCLAW_GOOGLE_MEET_LIVE_MEETING=https://meet.google.com/abc-defg-hij \
pnpm test:live -- extensions/google-meet/google-meet.live.test.ts
```

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
openclaw googlemeet test-listen https://meet.google.com/abc-defg-hij --transport chrome-node --timeout-ms 30000
```

| वेरिएबल                                                                                                                  | उद्देश्य                                                                |
| ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `OPENCLAW_LIVE_TEST=1`                                                                                                    | सुरक्षित लाइव टेस्ट सक्षम करता है |
| `OPENCLAW_GOOGLE_MEET_LIVE_MEETING`                                                                                       | सुरक्षित रखा गया Meet URL, कोड या `spaces/{id}` |
| `OPENCLAW_GOOGLE_MEET_CLIENT_ID` / `GOOGLE_MEET_CLIENT_ID`                                                                | OAuth क्लाइंट ID |
| `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` / `GOOGLE_MEET_REFRESH_TOKEN`                                                        | रिफ़्रेश टोकन |
| `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET`, `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN`, `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` | वैकल्पिक; `OPENCLAW_` प्रीफ़िक्स के बिना समान फ़ॉलबैक नाम भी काम करते हैं |

मूल आर्टिफ़ैक्ट/उपस्थिति स्मोक के लिए `meetings.space.readonly` और `meetings.conference.media.readonly` आवश्यक हैं। Calendar लुकअप के लिए `calendar.events.readonly` आवश्यक है। Drive दस्तावेज़-बॉडी एक्सपोर्ट के लिए `drive.meet.readonly` आवश्यक है।

### बनाने के उदाहरण

```bash
openclaw googlemeet create
```

नई मीटिंग URI, स्रोत और जॉइन सेशन प्रिंट करता है। OAuth के साथ यह Meet API का उपयोग करता है; उसके बिना, पिन किए गए Chrome Node के साइन-इन प्रोफ़ाइल का। ब्राउज़र फ़ॉलबैक JSON:

```json
{
  "source": "browser",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

यदि ब्राउज़र फ़ॉलबैक पहले Google लॉगिन या Meet अनुमति अवरोधक का सामना करता है, तो `google_meet` साधारण स्ट्रिंग के बजाय संरचित विवरण लौटाता है:

```json
{
  "source": "browser",
  "error": "google-login-required: OpenClaw ब्राउज़र प्रोफ़ाइल में Google में साइन इन करें, फिर मीटिंग बनाना पुनः आज़माएँ।",
  "manualActionRequired": true,
  "manualActionReason": "google-login-required",
  "manualActionMessage": "OpenClaw ब्राउज़र प्रोफ़ाइल में Google में साइन इन करें, फिर मीटिंग बनाना पुनः आज़माएँ।",
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1",
    "browserUrl": "https://accounts.google.com/signin",
    "browserTitle": "Sign in - Google Accounts"
  }
}
```

API क्रिएट JSON:

```json
{
  "source": "api",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "space": {
    "name": "spaces/abc-defg-hij",
    "meetingCode": "abc-defg-hij",
    "meetingUri": "https://meet.google.com/abc-defg-hij"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

बनाते समय डिफ़ॉल्ट रूप से जॉइन किया जाता है, लेकिन Chrome/Chrome-Node को ब्राउज़र के माध्यम से जॉइन करने के लिए फिर भी साइन-इन किया गया Google प्रोफ़ाइल चाहिए; साइन आउट होने पर, OpenClaw `manualActionRequired: true` या ब्राउज़र फ़ॉलबैक त्रुटि की रिपोर्ट करता है और ऑपरेटर से पुनः प्रयास करने से पहले Google लॉगिन पूरा करने को कहता है।

`preview.enrollmentAcknowledged: true` केवल यह पुष्टि करने के बाद सेट करें कि आपका Cloud प्रोजेक्ट, OAuth प्रिंसिपल और मीटिंग प्रतिभागी, Meet मीडिया API के Google Workspace Developer Preview Program में नामांकित हैं।

## कॉन्फ़िग

सामान्य Chrome एजेंट पथ को केवल सक्षम Plugin, BlackHole, SoX, एक रियलटाइम प्रोवाइडर कुंजी और कॉन्फ़िगर किया गया OpenClaw TTS प्रोवाइडर चाहिए:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {},
      },
    },
  },
}
```

### डिफ़ॉल्ट

| कुंजी                              | डिफ़ॉल्ट                                 | टिप्पणियाँ                                                                                                                                                                                                        |
| --------------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `defaultTransport`                | `"chrome"`                               |                                                                                                                                                                                                                   |
| `defaultMode`                     | `"agent"`                                | `"realtime"` को `"agent"` के लिए एक लीगेसी उपनाम के रूप में स्वीकार किया जाता है; नए कॉलर को `"agent"` कहना चाहिए                                                                                                                        |
| `chromeNode.node`                 | सेट नहीं                                    | `chrome-node` के लिए Node आईडी/नाम/IP; जब एक से अधिक सक्षम Node कनेक्ट हो सकते हों, तब आवश्यक                                                                                                                      |
| `chrome.launch`                   | `true`                                   | शामिल होने के लिए Chrome लॉन्च करता है; पहले से खुले सत्र का पुनः उपयोग करते समय ही `false` सेट करें                                                                                                                                 |
| `chrome.audioBackend`             | `"blackhole-2ch"`                        |                                                                                                                                                                                                                   |
| `chrome.guestName`                | `"OpenClaw Agent"`                       | साइन-आउट किए हुए Meet अतिथि स्क्रीन पर दिखाया जाता है                                                                                                                                                                         |
| `chrome.autoJoin`                 | `true`                                   | `chrome-node` पर यथासंभव अतिथि-नाम भरना और Join Now पर क्लिक करना                                                                                                                                                   |
| `chrome.reuseExistingTab`         | `true`                                   | डुप्लिकेट खोलने के बजाय मौजूदा Meet टैब को सक्रिय करता है                                                                                                                                                      |
| `chrome.waitForInCallMs`          | `20000`                                  | प्रत्युत्तर-वार्ता परिचय शुरू होने से पहले Meet टैब द्वारा कॉल में होने की सूचना देने की प्रतीक्षा करता है                                                                                                                                          |
| `chrome.audioFormat`              | `"pcm16-24khz"`                          | कमांड-युग्म ऑडियो प्रारूप; `"g711-ulaw-8khz"` केवल टेलीफ़ोनी ऑडियो उत्सर्जित करने वाले लीगेसी/कस्टम कमांड युग्मों के लिए है                                                                                                   |
| `chrome.audioBufferBytes`         | `4096`                                   | जनरेट किए गए कमांड-युग्म ऑडियो कमांड के लिए SoX प्रोसेसिंग बफ़र (SoX के डिफ़ॉल्ट 8192-बाइट बफ़र का आधा, जिससे पाइप विलंबता घटती है); मान न्यूनतम 17 बाइट तक सीमित किए जाते हैं                                         |
| `chrome.audioInputCommand`        | जनरेट किया गया SoX कमांड                    | CoreAudio `BlackHole 2ch` से पढ़ता है, `chrome.audioFormat` में ऑडियो लिखता है                                                                                                                                        |
| `chrome.audioOutputCommand`       | जनरेट किया गया SoX कमांड                    | `chrome.audioFormat` में ऑडियो पढ़ता है, CoreAudio `BlackHole 2ch` में लिखता है                                                                                                                                          |
| `chrome.bargeInInputCommand`      | सेट नहीं                                    | सहायक के प्लेबैक के दौरान मानव के बीच में बोलने का पता लगाने हेतु साइन किया हुआ 16-बिट लिटिल-एंडियन मोनो PCM लिखने वाला वैकल्पिक स्थानीय माइक्रोफ़ोन कमांड; Gateway द्वारा होस्ट किए गए कमांड-युग्म ब्रिज पर लागू होता है                          |
| `chrome.bargeInRmsThreshold`      | `650`                                    | मानव व्यवधान के रूप में गिना जाने वाला RMS स्तर                                                                                                                                                                           |
| `chrome.bargeInPeakThreshold`     | `2500`                                   | मानव व्यवधान के रूप में गिना जाने वाला पीक स्तर                                                                                                                                                                          |
| `chrome.bargeInCooldownMs`        | `900`                                    | बार-बार व्यवधान हटाने के बीच न्यूनतम विलंब                                                                                                                                                                |
| `mode` (प्रति अनुरोध)              | `"agent"`                                | प्रत्युत्तर-वार्ता मोड; [एजेंट और द्विदिश मोड](#agent-and-bidi-modes) तालिका देखें                                                                                                                                       |
| `realtime.provider`               | `"openai"`                               | नीचे दिए गए दायरा-निर्धारित फ़ील्ड सेट न होने पर उपयोग किया जाने वाला संगतता फ़ॉलबैक                                                                                                                                                |
| `realtime.transcriptionProvider`  | `"openai"`                               | रीयलटाइम ट्रांसक्रिप्शन के लिए `agent` मोड द्वारा उपयोग की जाने वाली प्रदाता आईडी                                                                                                                                                       |
| `realtime.voiceProvider`          | सेट नहीं                                    | सीधे रीयलटाइम वॉइस के लिए `bidi` मोड द्वारा उपयोग की जाने वाली प्रदाता आईडी; OpenAI पर एजेंट-मोड ट्रांसक्रिप्शन बनाए रखते हुए Gemini Live के लिए इसे `"google"` पर सेट करें। विशिष्ट Gemini Live मॉडल चुनने के लिए इसे `realtime.model` के साथ युग्मित करें। |
| `realtime.toolPolicy`             | `"safe-read-only"`                       | [एजेंट और द्विदिश मोड](#agent-and-bidi-modes) देखें                                                                                                                                                                 |
| `realtime.instructions`           | संक्षिप्त मौखिक-उत्तर निर्देश          | मॉडल को संक्षेप में बोलने और अधिक विस्तृत उत्तरों के लिए `openclaw_agent_consult` का उपयोग करने को कहता है                                                                                                                              |
| `realtime.introMessage`           | `"Say exactly: I'm here and listening."` | रीयलटाइम ब्रिज कनेक्ट होने पर एक बार बोला जाता है; चुपचाप शामिल होने के लिए इसे `""` पर सेट करें                                                                                                                                       |
| `realtime.agentId`                | `"main"`                                 | `openclaw_agent_consult` के लिए उपयोग की जाने वाली OpenClaw एजेंट आईडी                                                                                                                                                               |
| `voiceCall.enabled`               | `true`                                   | Twilio PSTN कॉल, DTMF और परिचय अभिवादन को Voice Call Plugin को सौंपता है                                                                                                                                 |
| `voiceCall.dtmfDelayMs`           | `12000`                                  | Twilio पर PIN से प्राप्त DTMF अनुक्रम चलाने से पहले आरंभिक प्रतीक्षा                                                                                                                                               |
| `voiceCall.postDtmfSpeechDelayMs` | `5000`                                   | Voice Call द्वारा Twilio चरण शुरू किए जाने के बाद रीयलटाइम परिचय अभिवादन का अनुरोध करने से पहले विलंब                                                                                                                        |

`chrome.audioBridgeCommand` और `chrome.audioBridgeHealthCommand`, `chrome.audioInputCommand`/`chrome.audioOutputCommand` के बजाय किसी बाहरी ब्रिज को संपूर्ण स्थानीय ऑडियो पथ का स्वामित्व लेने देते हैं; कौन-सा मोड उनका उपयोग कर सकता है, इसकी बाध्यता के लिए [टिप्पणियाँ](#notes) देखें।

लीगेसी `realtime.provider: "google"` आकार के लिए एक `openclaw doctor --fix` माइग्रेशन मौजूद है: यदि वे फ़ील्ड पहले से सेट नहीं हैं, तो यह उस आशय को `realtime.voiceProvider: "google"` और `realtime.transcriptionProvider: "openai"` में स्थानांतरित करता है।

### वैकल्पिक ओवरराइड

```json5
{
  defaults: {
    meeting: "https://meet.google.com/abc-defg-hij",
  },
  browser: {
    defaultProfile: "openclaw",
  },
  chrome: {
    guestName: "OpenClaw Agent",
    waitForInCallMs: 30000,
    bargeInInputCommand: [
      "sox",
      "-q",
      "-t",
      "coreaudio",
      "External Microphone",
      "-r",
      "24000",
      "-c",
      "1",
      "-b",
      "16",
      "-e",
      "signed-integer",
      "-t",
      "raw",
      "-",
    ],
  },
  chromeNode: {
    node: "parallels-macos",
  },
  defaultMode: "agent",
  realtime: {
    provider: "openai",
    transcriptionProvider: "openai",
    voiceProvider: "google",
    model: "gemini-3.1-flash-live-preview",
    agentId: "jay",
    toolPolicy: "owner",
    introMessage: "ठीक यही कहें: मैं यहाँ हूँ।",
    providers: {
      google: {
        speakerVoice: "Kore",
      },
    },
  },
}
```

एजेंट-मोड में सुनने और बोलने, दोनों के लिए ElevenLabs:

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        modelId: "eleven_v3",
        speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
      },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        config: {
          realtime: {
            transcriptionProvider: "elevenlabs",
            providers: {
              elevenlabs: {
                modelId: "scribe_v2_realtime",
                audioFormat: "ulaw_8000",
                sampleRate: 8000,
                commitStrategy: "vad",
              },
            },
          },
        },
      },
    },
  },
}
```

स्थायी Meet वॉइस `tts.providers.elevenlabs.speakerVoiceId` से आती है। TTS मॉडल ओवरराइड सक्षम होने पर एजेंट के उत्तर प्रति-उत्तर `[[tts:speakerVoiceId=... model=eleven_v3]]` निर्देशों का भी उपयोग कर सकते हैं, लेकिन मीटिंग के लिए कॉन्फ़िग नियतात्मक डिफ़ॉल्ट है। शामिल होने पर लॉग `transcriptionProvider=elevenlabs` दिखाते हैं, और प्रत्येक बोले गए उत्तर के लिए `provider=elevenlabs model=eleven_v3 speakerVoiceId=<voiceId>` लॉग होता है।

केवल Twilio के लिए कॉन्फ़िग:

```json5
{
  defaultTransport: "twilio",
  twilio: {
    defaultDialInNumber: "+15551234567",
    defaultPin: "123456",
  },
  voiceCall: {
    gatewayUrl: "ws://127.0.0.1:18789",
  },
}
```

`voiceCall.enabled: true` (डिफ़ॉल्ट) और Twilio ट्रांसपोर्ट के साथ, Voice Call रीयलटाइम मीडिया स्ट्रीम खोलने से पहले DTMF अनुक्रम स्थापित करता है, फिर सहेजे गए परिचय टेक्स्ट को आरंभिक रीयलटाइम अभिवादन के रूप में उपयोग करता है। यदि `voice-call` सक्षम नहीं है, तो Google Meet फिर भी डायल योजना को सत्यापित और रिकॉर्ड कर सकता है, लेकिन Twilio कॉल नहीं कर सकता।

स्थानीय विश्वसनीय Gateway रनटाइम का उपयोग करने के लिए `voiceCall.gatewayUrl` को अनसेट छोड़ें, जो पूरे
कॉल के दौरान आह्वान करने वाले एजेंट को बनाए रखता है। कॉन्फ़िगर किया गया Gateway URL स्पष्ट WebSocket लक्ष्य बना रहता है और
Plugin के उद्गम को प्रमाणित नहीं कर सकता; गैर-डिफ़ॉल्ट एजेंट जॉइन किसी अन्य एजेंट का चुपचाप
उपयोग करने के बजाय सुरक्षित रूप से विफल होते हैं। जब प्रति-एजेंट
रूटिंग आवश्यक हो, तब Google Meet और Voice Call को समान Gateway प्रक्रिया में चलाएँ।

## टूल

एजेंट `google_meet` टूल का उपयोग करते हैं:

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

| `action`                | उद्देश्य                                                                                           |
| ----------------------- | ------------------------------------------------------------------------------------------------- |
| `join`                  | किसी स्पष्ट Meet URL से जुड़ें                                                                         |
| `create`                | एक स्पेस बनाएँ (और डिफ़ॉल्ट रूप से जुड़ें); `accessType`/`entryPointAccess` का समर्थन करता है                    |
| `status`                | सक्रिय सत्र सूचीबद्ध करें, या `sessionId` द्वारा किसी एक का निरीक्षण करें                                               |
| `setup_status`          | `googlemeet setup` जैसी ही जाँच चलाएँ                                                         |
| `resolve_space`         | `spaces.get` के माध्यम से URL/कोड/`spaces/{id}` का समाधान करें                                                 |
| `preflight`             | OAuth और मीटिंग समाधान की पूर्वापेक्षाओं को सत्यापित करें                                                 |
| `latest`                | किसी मीटिंग के लिए नवीनतम कॉन्फ़्रेंस रिकॉर्ड खोजें                                                   |
| `calendar_events`       | Meet लिंक वाले Calendar ईवेंट का पूर्वावलोकन करें                                                           |
| `artifacts`             | कॉन्फ़्रेंस रिकॉर्ड और प्रतिभागी/रिकॉर्डिंग/ट्रांसक्रिप्ट/स्मार्ट-नोट मेटाडेटा सूचीबद्ध करें                  |
| `attendance`            | प्रतिभागियों और प्रतिभागी सत्रों को सूचीबद्ध करें                                                        |
| `export`                | आर्टिफ़ैक्ट/उपस्थिति/ट्रांसक्रिप्ट/मैनिफ़ेस्ट बंडल लिखें; केवल मैनिफ़ेस्ट के लिए `"dryRun": true` सेट करें |
| `recover_current_tab`   | नया टैब खोले बिना किसी मौजूदा Meet टैब पर फ़ोकस करें/उसका निरीक्षण करें                                      |
| `transcript`            | सीमित कैप्शन ट्रांसक्रिप्ट पढ़ें; `sinceIndex` पिछले `nextIndex` से फिर शुरू करता है           |
| `leave`                 | सत्र समाप्त करें (Chrome Leave पर क्लिक करता है; केवल अपने खोले टैब बंद करता है; Twilio कॉल काटता है)                  |
| `end_active_conference` | API-प्रबंधित स्पेस के लिए सक्रिय Google Meet कॉन्फ़्रेंस समाप्त करें                                    |
| `speak`                 | `sessionId` और `message` दिए जाने पर रीयलटाइम एजेंट से तुरंत बुलवाएँ                        |
| `test_speech`           | सत्र बनाएँ/पुनः उपयोग करें, ज्ञात वाक्यांश ट्रिगर करें और Chrome की स्थिति लौटाएँ                              |
| `test_listen`           | केवल-अवलोकन सत्र बनाएँ/पुनः उपयोग करें और कैप्शन/ट्रांसक्रिप्ट की गतिविधि की प्रतीक्षा करें                        |

`test_speech` हमेशा `mode: "agent"` या `"bidi"` को बाध्य करता है और `mode: "transcribe"` में चलाने के लिए कहे जाने पर विफल हो जाता है, क्योंकि केवल-अवलोकन सत्र वाणी उत्सर्जित नहीं कर सकते। `speechOutputVerified` के लिए ताज़ा रीयलटाइम आउटपुट बाइट और उस आउटपुट के दौरान ब्रिज के माइक्रोफ़ोन कैप्चर पथ पर लौटने वाला ताज़ा गैर-मौन ऑडियो, दोनों आवश्यक हैं। पुनः उपयोग किए गए सत्र का पुराना आउटपुट या लूपबैक संकेत मान्य नहीं होता, और केवल सिंक-बाइट की वृद्धि अब सत्यापित वाणी की सूचना नहीं देती।

Chrome ट्रांसपोर्ट के लिए, `leave` Meet का Leave call बटन क्लिक करने के बाद पुनः उपयोग किए गए उपयोगकर्ता-स्वामित्व वाले टैब को खुला रखता है। OpenClaw द्वारा खोले गए टैब प्रस्थान के बाद बंद हो जाते हैं।

जब Chrome Gateway होस्ट पर चलता हो तो `transport: "chrome"` का उपयोग करें, और युग्मित Node पर चलने पर `transport: "chrome-node"` का। दोनों स्थितियों में मॉडल प्रदाता और `openclaw_agent_consult` Gateway होस्ट पर चलते हैं, इसलिए मॉडल क्रेडेंशियल वहीं रहते हैं। एजेंट-मोड लॉग में ब्रिज स्टार्टअप के समय समाधान किया गया ट्रांसक्रिप्शन प्रदाता/मॉडल और प्रत्येक संश्लेषित उत्तर के बाद TTS प्रदाता/मॉडल/वॉइस/आउटपुट फ़ॉर्मैट/सैंपल दर शामिल होते हैं। कच्चा `mode: "realtime"` अब भी `mode: "agent"` के लिए पुरानी संगतता उपनाम के रूप में स्वीकार किया जाता है, लेकिन अब टूल के `mode` एनम में प्रदर्शित नहीं किया जाता।

API-समर्थित रूम और स्पष्ट ऐक्सेस नीति वाला `create`:

```json
{
  "action": "create",
  "transport": "chrome-node",
  "mode": "agent",
  "accessType": "OPEN"
}
```

किसी ज्ञात रूम की सक्रिय कॉन्फ़्रेंस समाप्त करना:

```json
{
  "action": "end_active_conference",
  "meeting": "https://meet.google.com/abc-defg-hij"
}
```

किसी मीटिंग को उपयोगी घोषित करने से पहले सुनने-प्रथम सत्यापन:

```json
{
  "action": "test_listen",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "timeoutMs": 30000
}
```

माँग पर बोलना:

```json
{
  "action": "speak",
  "sessionId": "meet_...",
  "message": "ठीक यही कहें: मैं यहाँ हूँ और सुन रहा हूँ।"
}
```

उपलब्ध होने पर `status` में Chrome की स्थिति शामिल होती है:

| फ़ील्ड                                                                 | अर्थ                                                                                                                |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `inCall`                                                              | Chrome Meet कॉल के भीतर प्रतीत होता है                                                                              |
| `micMuted`                                                            | सर्वोत्तम-प्रयास Meet माइक्रोफ़ोन स्थिति                                                                                      |
| `manualActionRequired` / `manualActionReason` / `manualActionMessage` | वाणी के काम करने से पहले ब्राउज़र प्रोफ़ाइल को मैन्युअल लॉगिन, Meet होस्ट प्रवेश, अनुमतियों या ब्राउज़र-नियंत्रण सुधार की आवश्यकता है |
| `speechReady` / `speechBlockedReason` / `speechBlockedMessage`        | प्रबंधित Chrome वाणी की अभी अनुमति है या नहीं; `speechReady: false` का अर्थ है कि OpenClaw ने परिचय/परीक्षण वाक्यांश नहीं भेजा   |
| `providerConnected` / `realtimeReady`                                 | रीयलटाइम वॉइस ब्रिज स्थिति                                                                                            |
| `lastInputAt` / `lastOutputAt`                                        | ब्रिज से प्राप्त/ब्रिज को भेजा गया अंतिम ऑडियो                                                                                |
| `audioOutputRouted` / `audioOutputDeviceLabel`                        | Meet टैब का मीडिया आउटपुट सक्रिय रूप से ब्रिज के BlackHole डिवाइस पर रूट किया गया था या नहीं                               |
| `lastOutputLoopbackAt` / `outputLoopbackSignalBytes`                  | ताज़ा आउटपुट जिसका वेवफ़ॉर्म फ़िंगरप्रिंट BlackHole माइक्रोफ़ोन कैप्चर पथ पर सहसंबद्ध हुआ                        |
| `lastOutputLoopbackCorrelation`                                       | कैप्चर किए गए संकेत को वर्तमान सहायक-आउटपुट जनरेशन से जोड़ने वाला सहसंबंध स्कोर                                 |
| `outputGeneration` / `verifiedOutputGeneration`                       | एकदिश रूप से बढ़ने वाली आईडी; समानता का अर्थ है कि पुराने उच्चारण के बजाय वर्तमान आउटपुट ने लूपबैक प्रमाण पार किया                |
| `lastOutputLoopbackRms` / `lastOutputLoopbackPeak`                    | नवीनतम सत्यापित लूपबैक कैप्चर खंड के लिए ऑडियो-ऊर्जा निदान                                                |
| `lastSuppressedInputAt` / `suppressedInputBytes`                      | सहायक प्लेबैक सक्रिय होने के दौरान लूपबैक इनपुट अनदेखा किया गया                                                              |

## एजेंट और द्विदिश मोड

| मोड    | उत्तर कौन तय करता है        | वाणी आउटपुट पथ                     | कब उपयोग करें                                              |
| ------- | ----------------------------- | -------------------------------------- | ----------------------------------------------------- |
| `agent` | कॉन्फ़िगर किया गया OpenClaw एजेंट | सामान्य OpenClaw TTS रनटाइम            | जब आपको "मेरा एजेंट मीटिंग में है" जैसा व्यवहार चाहिए        |
| `bidi`  | रीयलटाइम वॉइस मॉडल      | रीयलटाइम वॉइस प्रदाता की ऑडियो प्रतिक्रिया | जब आपको न्यूनतम-विलंबता वाला संवादात्मक वॉइस लूप चाहिए |

`agent` मोड: रीयलटाइम ट्रांसक्रिप्शन प्रदाता मीटिंग ऑडियो सुनता है, अंतिम प्रतिभागी ट्रांसक्रिप्ट कॉन्फ़िगर किए गए OpenClaw एजेंट के माध्यम से रूट होते हैं, और उत्तर नियमित OpenClaw TTS के माध्यम से बोला जाता है। परामर्श से पहले आस-पास के अंतिम-ट्रांसक्रिप्ट अंश एकत्र किए जाते हैं, ताकि एक बोला गया क्रम कई पुराने आंशिक उत्तर उत्पन्न न करे; कतारबद्ध सहायक ऑडियो चलने के दौरान रीयलटाइम इनपुट दबा दिया जाता है, और परामर्श से पहले हाल के सहायक-जैसे ट्रांसक्रिप्ट प्रतिध्वनियों को अनदेखा किया जाता है, ताकि BlackHole लूपबैक एजेंट से उसकी अपनी वाणी का उत्तर न दिलवाए।

`bidi` मोड: रीयलटाइम वॉइस मॉडल सीधे उत्तर देता है और गहन तर्क, वर्तमान जानकारी या सामान्य OpenClaw टूल के लिए `openclaw_agent_consult` को कॉल कर सकता है। परामर्श टूल हाल के मीटिंग ट्रांसक्रिप्ट संदर्भ के साथ पृष्ठभूमि में नियमित OpenClaw एजेंट चलाता है और संक्षिप्त मौखिक उत्तर लौटाता है; `agent` मोड में OpenClaw उस उत्तर को सीधे TTS को भेजता है, और `bidi` मोड में रीयलटाइम वॉइस मॉडल उसे बोल सकता है। यह Voice Call वाली साझा परामर्श प्रणाली का ही उपयोग करता है।

डिफ़ॉल्ट रूप से परामर्श `main` एजेंट के विरुद्ध चलते हैं; Meet लेन को समर्पित एजेंट वर्कस्पेस, मॉडल डिफ़ॉल्ट, टूल नीति, मेमोरी और सत्र इतिहास की ओर इंगित करने के लिए `realtime.agentId` सेट करें। एजेंट-मोड परामर्श प्रति-मीटिंग `agent:<id>:subagent:google-meet:<session>` सत्र कुंजी का उपयोग करते हैं, इसलिए अनुवर्ती प्रश्न सामान्य एजेंट नीति प्राप्त करते हुए मीटिंग संदर्भ बनाए रखते हैं। जब कोई एजेंट, एजेंट मोड में `google_meet` को कॉल करता है, तो परामर्शदाता सत्र प्रतिभागी की वाणी का उत्तर देने से पहले कॉलर के वर्तमान ट्रांसक्रिप्ट को फ़ोर्क करता है; Meet सत्र अलग रहता है, इसलिए मीटिंग के अनुवर्ती प्रश्न कॉलर के ट्रांसक्रिप्ट को सीधे परिवर्तित नहीं करते।

`realtime.toolPolicy` परामर्श रन को नियंत्रित करता है:

| नीति           | व्यवहार                                                                                                                         |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | परामर्श टूल उपलब्ध कराएँ; नियमित एजेंट को `read`, `web_search`, `web_fetch`, `x_search`, `memory_search`, `memory_get` तक सीमित करें |
| `owner`          | परामर्श टूल उपलब्ध कराएँ; नियमित एजेंट को उसकी सामान्य टूल नीति का उपयोग करने दें                                                        |
| `none`           | रीयलटाइम वॉइस मॉडल को परामर्श टूल उपलब्ध न कराएँ                                                                       |

परामर्श सत्र कुंजी प्रत्येक Meet सत्र के दायरे में होती है, इसलिए अनुवर्ती परामर्श कॉल उसी मीटिंग के दौरान पिछले परामर्श संदर्भ का पुनः उपयोग करते हैं।

Chrome के पूरी तरह जुड़ने के बाद मौखिक तत्परता जाँच बाध्य करें:

```bash
openclaw googlemeet speak meet_... "ठीक यही कहें: मैं यहाँ हूँ और सुन रहा हूँ।"
```

पूर्ण जॉइन-और-बोलने का स्मोक परीक्षण:

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "ठीक यही कहें: मैं यहाँ हूँ और सुन रहा हूँ।"
```

## लाइव परीक्षण चेकलिस्ट

किसी मीटिंग को अप्रत्यक्ष एजेंट को सौंपने से पहले:

```bash
openclaw googlemeet setup
openclaw nodes status
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "ठीक यही कहें: Google Meet वाणी परीक्षण पूर्ण हुआ।"
```

अपेक्षित Chrome-node स्थिति:

- `googlemeet setup` पूरी तरह हरा है, और जब Chrome-node डिफ़ॉल्ट ट्रांसपोर्ट हो या कोई नोड पिन किया गया हो, तब इसमें `chrome-node-connected` शामिल होता है।
- `nodes status` चयनित नोड को कनेक्टेड दिखाता है, जो `googlemeet.chrome` और `browser.proxy` दोनों को विज्ञापित करता है।
- Meet टैब जुड़ता है, और `test-speech`, `inCall: true` के साथ Chrome की स्थिति लौटाता है।

Parallels macOS VM जैसे रिमोट Chrome होस्ट के लिए, Gateway या VM को अपडेट करने के बाद सबसे छोटी सुरक्षित जाँच:

```bash
openclaw googlemeet setup
openclaw nodes status --connected
openclaw nodes invoke \
  --node parallels-macos \
  --command googlemeet.chrome \
  --params '{"action":"setup"}'
```

यह प्रमाणित करता है कि Gateway Plugin लोड है, VM नोड वर्तमान टोकन के साथ कनेक्टेड है, और किसी एजेंट द्वारा वास्तविक मीटिंग टैब खोलने से पहले Meet ऑडियो ब्रिज उपलब्ध है।

Twilio स्मोक जाँच के लिए, ऐसी मीटिंग का उपयोग करें जो फ़ोन डायल-इन विवरण उपलब्ध कराती हो:

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

अपेक्षित Twilio स्थिति:

- `googlemeet setup` में हरी `twilio-voice-call-plugin`, `twilio-voice-call-credentials`, और `twilio-voice-call-webhook` जाँचें शामिल होती हैं।
- Gateway को पुनः लोड करने के बाद `voicecall` CLI में उपलब्ध होता है।
- लौटाए गए सेशन में `transport: "twilio"` और एक `twilio.voiceCallId` होता है।
- `openclaw logs --follow` दिखाता है कि रियलटाइम TwiML से पहले DTMF TwiML सर्व किया गया, फिर शुरुआती अभिवादन को कतार में रखकर एक रियलटाइम ब्रिज बनाया गया।
- `googlemeet leave <sessionId>` प्रत्यायोजित वॉइस कॉल समाप्त करता है।

## समस्या निवारण

### एजेंट को Google Meet टूल दिखाई नहीं देता

पुष्टि करें कि Plugin सक्षम है और Gateway को पुनः लोड करें; चल रहा एजेंट केवल वर्तमान Gateway प्रक्रिया द्वारा पंजीकृत Plugin टूल देखता है:

```bash
openclaw plugins list | grep google-meet
openclaw googlemeet setup
```

गैर-macOS Gateway होस्ट पर, `google_meet` दिखाई देता रहता है, लेकिन स्थानीय Chrome टॉक-बैक क्रियाएँ ऑडियो ब्रिज तक पहुँचने से पहले अवरुद्ध कर दी जाती हैं। डिफ़ॉल्ट स्थानीय Chrome एजेंट पथ के बजाय `mode: "transcribe"`, Twilio डायल-इन, या macOS `chrome-node` होस्ट का उपयोग करें।

### Google Meet-सक्षम कोई नोड कनेक्टेड नहीं है

नोड होस्ट पर:

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw plugins enable browser
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

Gateway होस्ट पर:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

नोड कनेक्टेड होना चाहिए और उसमें `googlemeet.chrome` के साथ `browser.proxy` सूचीबद्ध होना चाहिए; Gateway कॉन्फ़िगरेशन को दोनों की अनुमति देनी चाहिए:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["browser.proxy", "googlemeet.chrome"] },
    },
  },
}
```

यदि `googlemeet setup`, `chrome-node-connected` में विफल होता है, या Gateway लॉग `gateway token mismatch` रिपोर्ट करता है, तो वर्तमान Gateway टोकन के साथ नोड को पुनः इंस्टॉल या रीस्टार्ट करें:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install \
  --host <gateway-lan-ip> \
  --port 18789 \
  --display-name parallels-macos \
  --force
```

फिर नोड सेवा को पुनः लोड करें और इन्हें दोबारा चलाएँ:

```bash
openclaw googlemeet setup
openclaw nodes status --connected
```

### ब्राउज़र खुलता है लेकिन एजेंट जुड़ नहीं पाता

केवल-अवलोकन वाले जुड़ावों के लिए `googlemeet test-listen` या रियलटाइम जुड़ावों के लिए `googlemeet test-speech` चलाएँ, फिर लौटाई गई Chrome स्थिति की जाँच करें। यदि इनमें से कोई भी `manualActionRequired: true` रिपोर्ट करता है, तो ऑपरेटर को `manualActionMessage` दिखाएँ और ब्राउज़र क्रिया पूरी होने तक पुनः प्रयास करना बंद करें।

सामान्य मैन्युअल क्रियाएँ: Chrome प्रोफ़ाइल में साइन इन करें; Meet होस्ट खाते से अतिथि को प्रवेश दें; नेटिव प्रॉम्प्ट दिखाई देने पर Chrome को माइक्रोफ़ोन/कैमरा अनुमतियाँ दें; अटके हुए Meet अनुमति डायलॉग को बंद करें या ठीक करें।

केवल इसलिए "साइन इन नहीं है" रिपोर्ट न करें क्योंकि Meet पूछता है "क्या आप चाहते हैं कि मीटिंग में लोग आपकी आवाज़ सुनें?"; यह Meet का ऑडियो-विकल्प मध्यवर्ती पृष्ठ है। उपलब्ध होने पर OpenClaw ब्राउज़र ऑटोमेशन के माध्यम से **माइक्रोफ़ोन का उपयोग करें** पर क्लिक करता है और वास्तविक मीटिंग स्थिति की प्रतीक्षा करता रहता है; केवल-निर्माण ब्राउज़र फ़ॉलबैक के लिए यह इसके बजाय **माइक्रोफ़ोन के बिना जारी रखें** पर क्लिक कर सकता है, क्योंकि URL बनाने के लिए रियलटाइम ऑडियो पथ की आवश्यकता नहीं होती।

### मीटिंग निर्माण विफल होता है

OAuth कॉन्फ़िगर होने पर `googlemeet create`, Meet API `spaces.create` का उपयोग करता है, अन्यथा पिन किए गए Chrome नोड ब्राउज़र का। पुष्टि करें:

- **API निर्माण**: `oauth.clientId` और `oauth.refreshToken` (या संबंधित `OPENCLAW_GOOGLE_MEET_*` पर्यावरण चर) मौजूद हैं, और रीफ़्रेश टोकन निर्माण समर्थन जोड़े जाने के बाद बनाया गया था; पुराने टोकन में `meetings.space.created` अनुपस्थित हो सकता है, इसलिए `openclaw googlemeet auth login --json` दोबारा चलाएँ।
- **ब्राउज़र फ़ॉलबैक**: `defaultTransport: "chrome-node"` और `chromeNode.node` ऐसे कनेक्टेड नोड की ओर संकेत करते हैं जिसमें `browser.proxy` और `googlemeet.chrome` हैं; उस नोड पर OpenClaw Chrome प्रोफ़ाइल साइन इन है और `https://meet.google.com/new` खोल सकती है।
- **ब्राउज़र फ़ॉलबैक पुनः प्रयास**: नया टैब खोलने से पहले मौजूदा `.../new` या Google खाता प्रॉम्प्ट टैब का पुनः उपयोग करें; मैन्युअल रूप से दूसरा टैब खोलने के बजाय टूल कॉल को दोबारा चलाएँ।
- **मैन्युअल क्रिया**: यदि टूल `manualActionRequired: true` लौटाता है, तो ऑपरेटर का मार्गदर्शन करने के लिए `browser.nodeId`, `browser.targetId`, `browserUrl`, और `manualActionMessage` का उपयोग करें; लूप में पुनः प्रयास न करें।
- **ऑडियो-विकल्प मध्यवर्ती पृष्ठ**: यदि Meet "क्या आप चाहते हैं कि मीटिंग में लोग आपकी आवाज़ सुनें?" दिखाता है, तो टैब खुला छोड़ दें। OpenClaw को **माइक्रोफ़ोन का उपयोग करें** या (केवल निर्माण के लिए) **माइक्रोफ़ोन के बिना जारी रखें** पर क्लिक करके जनरेट किए गए URL की प्रतीक्षा करते रहना चाहिए; यदि वह ऐसा नहीं कर सकता, तो त्रुटि में `meet-audio-choice-required` का उल्लेख होना चाहिए, `google-login-required` का नहीं।

### एजेंट जुड़ता है लेकिन बोलता नहीं है

```bash
openclaw googlemeet setup
openclaw googlemeet doctor
```

STT -> OpenClaw एजेंट -> TTS पथ के लिए `mode: "agent"`, और सीधे रियलटाइम वॉइस फ़ॉलबैक के लिए `mode: "bidi"` का उपयोग करें। `mode: "transcribe"` जानबूझकर कोई टॉक-बैक ब्रिज शुरू नहीं करता। केवल-अवलोकन डीबगिंग के लिए, प्रतिभागियों के बोलने के बाद `openclaw googlemeet status --json <session-id>` चलाएँ और `captioning`, `transcriptLines`, `lastCaptionText` जाँचें। यदि `inCall` सही है लेकिन `transcriptLines`, `0` ही रहता है, तो संभव है कि Meet कैप्शन अक्षम हों, ऑब्ज़र्वर इंस्टॉल होने के बाद किसी ने बात न की हो, Meet UI बदल गया हो, या मीटिंग की भाषा/खाते के लिए लाइव कैप्शन उपलब्ध न हों।

`googlemeet test-speech` हमेशा रियलटाइम पथ की जाँच करता है और रिपोर्ट करता है कि उस आह्वान के लिए ब्रिज आउटपुट बाइट देखे गए या नहीं। यदि `speechOutputVerified` गलत है और `speechOutputTimedOut` सही है, तो हो सकता है रियलटाइम प्रदाता ने कथन स्वीकार कर लिया हो, लेकिन OpenClaw ने Chrome ऑडियो ब्रिज तक नए आउटपुट बाइट पहुँचते हुए न देखे हों।

यह भी सत्यापित करें: Gateway होस्ट पर रियलटाइम प्रदाता कुंजी (`OPENAI_API_KEY` या `GEMINI_API_KEY`) उपलब्ध है; Chrome होस्ट पर `BlackHole 2ch` दिखाई देता है; वहाँ `sox` मौजूद है; Meet का माइक/स्पीकर वर्चुअल ऑडियो पथ के माध्यम से रूट किया गया है (स्थानीय Chrome रियलटाइम जुड़ावों के लिए `doctor` को `meet output routed: yes` दिखाना चाहिए)।

`googlemeet doctor [session-id]` सेशन, नोड, कॉल-अंतर्गत स्थिति, मैन्युअल क्रिया का कारण, रियलटाइम प्रदाता कनेक्शन, `realtimeReady`, ऑडियो इनपुट/आउटपुट गतिविधि, अंतिम ऑडियो टाइमस्टैम्प, बाइट काउंटर, और ब्राउज़र URL प्रिंट करता है। कच्चे JSON के लिए `googlemeet status [session-id] --json`, और टोकन उजागर किए बिना OAuth रीफ़्रेश सत्यापित करने के लिए `googlemeet doctor --oauth` (`--meeting` या `--create-space` जोड़ें) का उपयोग करें।

यदि किसी एजेंट का समय समाप्त हो गया है और Meet टैब पहले से खुला है, तो दूसरा टैब खोले बिना उसकी जाँच करें:

```bash
openclaw googlemeet recover-tab
openclaw googlemeet recover-tab https://meet.google.com/abc-defg-hij
```

समतुल्य टूल क्रिया `recover_current_tab` है: यह नया टैब या सेशन खोले बिना चयनित ट्रांसपोर्ट ( `chrome` के लिए स्थानीय ब्राउज़र नियंत्रण, `chrome-node` के लिए कॉन्फ़िगर किया गया नोड) हेतु मौजूदा Meet टैब पर फ़ोकस करके उसकी जाँच करती है, और वर्तमान अवरोधक (लॉगिन, प्रवेश, अनुमतियाँ, ऑडियो-विकल्प स्थिति) रिपोर्ट करती है। CLI कमांड कॉन्फ़िगर किए गए Gateway से संवाद करता है, जिसका चलना आवश्यक है; `chrome-node` के लिए नोड का कनेक्टेड होना भी आवश्यक है।

### Twilio सेटअप जाँचें विफल होती हैं

जब `voice-call` अनुमत या सक्षम नहीं होता, तब `twilio-voice-call-plugin` विफल होता है: इसे `plugins.allow` में जोड़ें, `plugins.entries.voice-call` सक्षम करें, और Gateway को पुनः लोड करें।

जब Twilio बैकएंड में खाता SID, प्रमाणीकरण टोकन, या कॉलर नंबर नहीं होता, तब `twilio-voice-call-credentials` विफल होता है:

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
```

जब `voice-call` के पास सार्वजनिक Webhook एक्सपोज़र नहीं होता, या `publicUrl` लूपबैक/निजी नेटवर्क स्थान की ओर संकेत करता है, तब `twilio-voice-call-webhook` विफल होता है। `localhost`, `127.0.0.1`, `0.0.0.0`, `10.x`, `172.16.x`-`172.31.x`, `192.168.x`, `169.254.x`, `fc00::/7`, या `fd00::/8` को `publicUrl` के रूप में उपयोग न करें; कैरियर कॉलबैक उन तक नहीं पहुँच सकते। `plugins.entries.voice-call.config.publicUrl` को किसी सार्वजनिक URL पर सेट करें, या टनल/Tailscale एक्सपोज़र कॉन्फ़िगर करें:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          fromNumber: "+15550001234",
          publicUrl: "https://voice.example.com/voice/webhook",
        },
      },
    },
  },
}
```

स्थानीय विकास के लिए, निजी होस्ट URL के बजाय टनल या Tailscale एक्सपोज़र का उपयोग करें:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tunnel: { provider: "ngrok" },
          // या
          tailscale: { mode: "funnel", path: "/voice/webhook" },
        },
      },
    },
  },
}
```

Gateway को रीस्टार्ट या पुनः लोड करें, फिर:

```bash
openclaw googlemeet setup --transport twilio
openclaw voicecall setup
openclaw voicecall smoke
```

डिफ़ॉल्ट रूप से `voicecall smoke` केवल तत्परता की जाँच करता है। किसी विशिष्ट नंबर पर ड्राई-रन करें:

```bash
openclaw voicecall smoke --to "+15555550123"
```

लाइव आउटबाउंड कॉल जानबूझकर करने के लिए ही `--yes` जोड़ें:

```bash
openclaw voicecall smoke --to "+15555550123" --yes
```

### Twilio कॉल शुरू होती है लेकिन मीटिंग में कभी प्रवेश नहीं करती

पुष्टि करें कि Meet ईवेंट फ़ोन डायल-इन विवरण उपलब्ध कराता है, और सटीक डायल-इन नंबर के साथ PIN या कस्टम DTMF अनुक्रम दें:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

PIN से पहले विराम के लिए `--dtmf-sequence` में आरंभिक `w` या अल्पविराम का उपयोग करें।

यदि कॉल बन जाती है लेकिन Meet प्रतिभागी सूची में डायल-इन प्रतिभागी कभी दिखाई नहीं देता:

- `openclaw googlemeet doctor <session-id>`: प्रत्यायोजित Twilio कॉल ID, DTMF को कतार में रखा गया था या नहीं, और परिचयात्मक अभिवादन का अनुरोध किया गया था या नहीं, इसकी पुष्टि करें।
- `openclaw voicecall status --call-id <id>`: पुष्टि करें कि कॉल अभी भी सक्रिय है।
- `openclaw voicecall tail`: पुष्टि करें कि Twilio Webhook Gateway तक पहुँच रहे हैं।
- `openclaw logs --follow`: Twilio Meet अनुक्रम खोजें: Google Meet जुड़ाव प्रत्यायोजित करता है, Voice Call कनेक्शन-पूर्व DTMF TwiML को संग्रहीत और सर्व करता है, Voice Call Twilio कॉल के लिए रियलटाइम TwiML सर्व करता है, फिर Google Meet `voicecall.speak` के साथ परिचयात्मक वाणी का अनुरोध करता है।
- `openclaw googlemeet setup --transport twilio` दोबारा चलाएँ; हरी सेटअप जाँच आवश्यक है, लेकिन यह प्रमाणित नहीं करती कि मीटिंग PIN अनुक्रम सही है।
- पुष्टि करें कि डायल-इन नंबर उसी Meet आमंत्रण और क्षेत्र से संबंधित है जिससे PIN संबंधित है।
- यदि Meet धीरे उत्तर देता है या कनेक्शन-पूर्व DTMF भेजे जाने के बाद भी कॉल ट्रांसक्रिप्ट PIN प्रॉम्प्ट दिखाती है, तो `voiceCall.dtmfDelayMs` को 12-सेकंड के डिफ़ॉल्ट से बढ़ाएँ।
- यदि प्रतिभागी जुड़ जाता है लेकिन अभिवादन सुनाई नहीं देता, तो DTMF के बाद वाले `voicecall.speak` अनुरोध और मीडिया-स्ट्रीम TTS प्लेबैक या Twilio `<Say>` फ़ॉलबैक के लिए `openclaw logs --follow` जाँचें। यदि ट्रांसक्रिप्ट अब भी "मीटिंग PIN दर्ज करें" दिखाती है, तो फ़ोन लेग अभी तक Meet कक्ष में शामिल नहीं हुआ है, इसलिए प्रतिभागियों को वाणी सुनाई नहीं देगी।

यदि webhooks नहीं पहुँचते हैं, तो पहले Voice Call Plugin को डीबग करें: प्रदाता को `plugins.entries.voice-call.config.publicUrl` या कॉन्फ़िगर की गई टनल तक पहुँचने में सक्षम होना चाहिए। [वॉइस कॉल समस्या निवारण](/hi/plugins/voice-call#troubleshooting) देखें।

## टिप्पणियाँ

Google Meet का आधिकारिक मीडिया API प्राप्ति-उन्मुख है, इसलिए कॉल में बोलने के लिए अब भी प्रतिभागी पथ आवश्यक है। यह Plugin उस सीमा को स्पष्ट रखता है: Chrome ब्राउज़र सहभागिता और स्थानीय ऑडियो रूटिंग संभालता है; Twilio फ़ोन डायल-इन सहभागिता संभालता है।

Chrome टॉक-बैक मोड के लिए `BlackHole 2ch` के साथ निम्न में से कोई एक भी आवश्यक है:

- `chrome.audioInputCommand` के साथ `chrome.audioOutputCommand`: OpenClaw ब्रिज का स्वामी होता है और उन कमांड तथा चयनित प्रदाता के बीच `chrome.audioFormat` में ऑडियो पाइप करता है। `agent` मोड रीयल-टाइम ट्रांसक्रिप्शन और नियमित TTS का उपयोग करता है; `bidi` मोड रीयल-टाइम वॉइस प्रदाता का उपयोग करता है। डिफ़ॉल्ट पथ `chrome.audioBufferBytes: 4096` के साथ 24 kHz PCM16 है; लीगेसी कमांड युग्मों के लिए 8 kHz G.711 mu-law उपलब्ध रहता है।
- `chrome.audioBridgeCommand`: कोई बाहरी ब्रिज कमांड संपूर्ण स्थानीय ऑडियो पथ का स्वामी होता है और अपने डेमन को शुरू या सत्यापित करने के बाद उसे बंद हो जाना चाहिए। यह केवल `bidi` के लिए मान्य है, क्योंकि `agent` मोड को TTS के लिए कमांड-युग्म तक सीधी पहुँच चाहिए।

कमांड-युग्म Chrome ब्रिज के साथ, `chrome.bargeInInputCommand` एक अलग स्थानीय माइक्रोफ़ोन सुन सकता है और किसी मानव के बोलना शुरू करने पर सहायक का प्लेबैक साफ़ कर सकता है, जिससे सहायक के प्लेबैक के दौरान साझा BlackHole लूपबैक इनपुट अस्थायी रूप से दबा होने पर भी मानव की वाणी सहायक के आउटपुट से आगे रहती है। `chrome.audioInputCommand`/`chrome.audioOutputCommand` की तरह, यह ऑपरेटर द्वारा कॉन्फ़िगर किया गया स्थानीय कमांड है: किसी स्पष्ट विश्वसनीय कमांड पथ या आर्ग्युमेंट सूची का उपयोग करें, किसी अविश्वसनीय स्थान की स्क्रिप्ट का कभी उपयोग न करें।

स्वच्छ डुप्लेक्स ऑडियो के लिए, Meet आउटपुट और Meet माइक्रोफ़ोन को अलग-अलग वर्चुअल डिवाइसों या Loopback-शैली के वर्चुअल डिवाइस ग्राफ़ से रूट करें; एकल साझा BlackHole डिवाइस अन्य प्रतिभागियों की आवाज़ को वापस कॉल में प्रतिध्वनित कर सकता है।

`googlemeet speak` किसी Chrome सत्र के लिए सक्रिय टॉक-बैक ऑडियो ब्रिज को ट्रिगर करता है; `googlemeet leave` उसे रोकता है (और Voice Call के माध्यम से प्रत्यायोजित Twilio सत्रों के लिए, अंतर्निहित कॉल को काट देता है)। API-प्रबंधित स्पेस के लिए सक्रिय Google Meet कॉन्फ़्रेंस भी बंद करने हेतु `googlemeet end-active-conference` का उपयोग करें।

## संबंधित

- [मीटिंग Plugin का अवलोकन](/hi/plugins/meeting-plugins)
- [Voice Call Plugin](/hi/plugins/voice-call)
- [टॉक मोड](/hi/nodes/talk)
- [Plugin बनाना](/hi/plugins/building-plugins)
