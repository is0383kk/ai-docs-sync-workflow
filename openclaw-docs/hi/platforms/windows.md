---
read_when:
    - Windows पर OpenClaw इंस्टॉल करना
    - Windows Hub, नेटिव Windows और WSL2 में से चुनना
    - Windows सहायक ऐप या Windows Node मोड सेट अप करना
summary: 'Windows समर्थन: Windows Hub, नेटिव CLI और Gateway, WSL2 Gateway सेटअप, Node मोड और समस्या निवारण'
title: Windows
x-i18n:
    generated_at: "2026-07-27T18:36:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c231b81971e1df9f3ee4de1b102c25328c242109331c6465dc802ec003af722b
    source_path: platforms/windows.md
    workflow: 16
---

OpenClaw एक मूल **Windows Hub** सहायक ऐप के साथ Windows CLI समर्थन प्रदान करता है।
सेटअप, ट्रे स्थिति, चैट, Command Center निदान और Windows Node क्षमताओं वाले डेस्कटॉप ऐप के लिए Windows Hub का उपयोग करें। सीधे CLI/Gateway के लिए PowerShell
इंस्टॉलर का उपयोग करें। सर्वाधिक
Linux-संगत Gateway रनटाइम के लिए WSL2 का उपयोग करें।

## अनुशंसित: Windows Hub

Windows Hub, Windows 10 20H2+ और
Windows 11 के लिए मूल WinUI सहायक ऐप है। यह व्यवस्थापक विशेषाधिकारों के बिना इंस्टॉल होता है और अपने रिलीज़ पृष्ठ से हस्ताक्षरित x64
और ARM64 इंस्टॉलर प्रदान करता है।

Windows Hub, OpenClaw CLI और Gateway से स्वतंत्र रूप से प्रकाशित होता है। नवीनतम स्थिर Hub इंस्टॉलर
[Windows Hub रिलीज़ पृष्ठ](https://github.com/openclaw/openclaw-windows-node/releases/latest)
से या सीधे `releases/latest/download` के माध्यम से डाउनलोड करें:

- [OpenClawCompanion-Setup-x64.exe](https://github.com/openclaw/openclaw-windows-node/releases/latest/download/OpenClawCompanion-Setup-x64.exe)
- [OpenClawCompanion-Setup-arm64.exe](https://github.com/openclaw/openclaw-windows-node/releases/latest/download/OpenClawCompanion-Setup-arm64.exe)

यदि ऊपर दी गई कोई लिंक 404 त्रुटि देती है, तो [Windows Hub रिलीज़ पृष्ठ](https://github.com/openclaw/openclaw-windows-node/releases)
पर जाएँ और नवीनतम स्थिर Windows Hub रिलीज़ खोलें। OpenClaw की नियमित स्थिर रिलीज़
एक पिन किया हुआ, रिलीज़-सत्यापित Windows Hub बिल्ड भी मिरर करती हैं; वह मिरर
नई स्वतंत्र Hub रिलीज़ से पीछे रह सकता है।

इंस्टॉल करने के बाद, Start मेनू या सिस्टम
ट्रे से **OpenClaw Companion** लॉन्च करें। इंस्टॉलर Gateway Setup, Chat, Settings,
Check for Updates और अनइंस्टॉल के शॉर्टकट भी जोड़ता है।

### Windows Hub में क्या शामिल है

- सिस्टम ट्रे स्थिति और लॉगिन पर लॉन्च।
- स्थानीय, ऐप-स्वामित्व वाले WSL Gateway के लिए प्रथम-रन सेटअप।
- स्थानीय, रिमोट और SSH-टनल वाले Gateways के लिए कनेक्शन सेटिंग।
- मूल चैट विंडो और ब्राउज़र Control UI तक पहुँच।
- सत्रों, उपयोग, चैनलों, Nodes, पेयरिंग और सुधार कमांडों के लिए Command Center निदान।
- एजेंट-नियंत्रित कैनवास, स्क्रीन, कैमरा,
  सूचनाओं, डिवाइस स्थिति, बातचीत और नियंत्रित `system.run` के लिए Windows Node मोड।
- Claude Desktop, Claude Code और Cursor जैसे MCP क्लाइंट के लिए स्थानीय MCP सर्वर मोड।

### पहला लॉन्च

पहले लॉन्च पर, यदि उपयोग करने योग्य सहेजा गया
Gateway नहीं है, तो Windows Hub सेटअप खोलता है। सबसे तेज़ तरीका **Set up locally** है, जो
ऐप-स्वामित्व वाली `OpenClawGateway` WSL डिस्ट्रो का प्रावधान करता है, उसके अंदर Gateway इंस्टॉल करता है और
ऐप को पेयर करता है। यह आपकी मौजूदा Ubuntu डिस्ट्रो को एक्सपोर्ट या संशोधित नहीं करता।

यदि आपके पास पहले से Gateway है, तो **Advanced setup** चुनें या Connections टैब खोलें।
आप इनसे कनेक्ट कर सकते हैं:

- इस PC पर स्थानीय Gateway
- इस PC पर WSL Gateway
- URL और टोकन या सेटअप कोड द्वारा रिमोट Gateway
- SSH टनल के माध्यम से पहुँचा गया Gateway

सेटअप पूरा होने पर ट्रे आइकन हरा हो जाता है। कनेक्शन, पेयरिंग, Node स्थिति और चैनल की स्थिति की पुष्टि करने के लिए
ट्रे से **Command Center** खोलें।

## Windows Node मोड

Windows Hub स्वयं को OpenClaw Node के रूप में पंजीकृत कर सकता है, ताकि एजेंट Gateway के माध्यम से घोषित
Windows-मूल क्षमताओं का उपयोग कर सके। Node कमांड चलने से पहले
उन्हें Node द्वारा घोषित और Gateway नीति द्वारा अनुमत होना आवश्यक है; पूर्ण अनुमति/अस्वीकृति मॉडल के लिए
[Nodes](/hi/nodes#command-policy) देखें।

सामान्य कमांड:

| परिवार | कमांड                                                                               |
| ------ | ------------------------------------------------------------------------------------ |
| कैनवास | `canvas.present`, `canvas.hide`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot` |
| स्क्रीन | `screen.snapshot`; `screen.record` के लिए स्पष्ट ऑप्ट-इन आवश्यक है                          |
| कैमरा | `camera.list`; `camera.snap`, `camera.clip` के लिए स्पष्ट ऑप्ट-इन आवश्यक है                  |
| सिस्टम | `system.notify`, `system.run`, `system.run.prepare`, `system.which`                  |
| डिवाइस | `location.get`, `device.info`, `device.status`                                       |
| बातचीत   | `talk.ptt.start`, `talk.ptt.stop`, `talk.ptt.cancel`, `talk.ptt.once`, `talk.speak`  |

Node मोड के लिए Gateway पेयरिंग आवश्यक है। यदि ऐप पेयरिंग अनुरोध दिखाता है,
तो Gateway होस्ट से उसे स्वीकृत करें:

```powershell
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Gateway केवल उन्हीं कमांड को अग्रेषित करता है जिन्हें Node घोषित करता है और सर्वर नीति
अनुमति देती है। `screen.record`, `camera.snap`
और `camera.clip` जैसे गोपनीयता-संवेदनशील कमांडों के लिए स्पष्ट `gateway.nodes.commands.allow` ऑप्ट-इन आवश्यक है।

## स्थानीय MCP मोड

Windows Hub उसी Windows-मूल क्षमता रजिस्ट्री को लूपबैक पर स्थानीय
MCP सर्वर के रूप में उपलब्ध करा सकता है, ताकि स्थानीय MCP क्लाइंट किसी चालू OpenClaw Gateway के बिना
Windows क्षमताओं को नियंत्रित कर सकें।

इसे Windows Hub Settings के डेवलपर/उन्नत अनुभाग में सक्षम करें। सर्वर सक्षम होने पर
ऐप लूपबैक एंडपॉइंट और बेयरर टोकन दिखाता है।

मोड मैट्रिक्स:

| Node मोड | MCP सर्वर | व्यवहार                           |
| --------- | ---------- | ---------------------------------- |
| बंद       | बंद        | केवल ऑपरेटर वाला डेस्कटॉप ऐप          |
| चालू        | बंद        | Gateway से जुड़ा Windows Node     |
| बंद       | चालू         | केवल स्थानीय MCP सर्वर              |
| चालू        | चालू         | Gateway Node के साथ स्थानीय MCP सर्वर |

## मूल Windows CLI और Gateway

टर्मिनल-प्रथम उपयोग के लिए PowerShell से OpenClaw इंस्टॉल करें:

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

सत्यापित करें:

```powershell
openclaw --version
openclaw doctor
openclaw gateway status --json
```

उपलब्ध होने पर प्रबंधित स्टार्टअप Windows Scheduled Tasks का उपयोग करता है। टास्क
पठनीय `gateway.cmd` स्क्रिप्ट को OpenClaw स्टेट डायरेक्टरी में रखता है, लेकिन इसे
जनरेट किए गए `gateway.vbs` WScript रैपर के माध्यम से लॉन्च करता है, ताकि पृष्ठभूमि Gateway
कोई दृश्यमान कंसोल विंडो न खोले। यदि टास्क बनाना अस्वीकृत हो जाता है, तो OpenClaw
प्रति-उपयोगकर्ता Startup-folder लॉगिन आइटम पर वापस चला जाता है।

Gateway सेवा इंस्टॉल करें:

```powershell
openclaw gateway install
openclaw gateway status --json
```

प्रबंधित Gateway सेवा के बिना केवल CLI उपयोग के लिए:

```powershell
openclaw onboard --non-interactive --skip-health
openclaw gateway run
```

## WSL2 Gateway

WSL2, Windows पर सर्वाधिक Linux-संगत Gateway रनटाइम बना हुआ है। Windows
Hub आपके लिए ऐप-स्वामित्व वाला WSL Gateway सेटअप कर सकता है, या आप अपनी डिस्ट्रो के अंदर
इसे मैन्युअल रूप से इंस्टॉल कर सकते हैं।

मैन्युअल सेटअप:

```powershell
wsl --install
# या स्पष्ट रूप से कोई डिस्ट्रो चुनें:
wsl --list --online
wsl --install -d Ubuntu-24.04
```

WSL के अंदर systemd सक्षम करें:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

PowerShell से WSL पुनः प्रारंभ करें:

```powershell
wsl --shutdown
```

फिर Linux क्विकस्टार्ट का उपयोग करके WSL के अंदर OpenClaw इंस्टॉल करें:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw gateway status
```

## Windows लॉगिन से पहले Gateway का स्वतः प्रारंभ

हेडलेस WSL सेटअप के लिए सुनिश्चित करें कि कोई Windows में लॉगिन न करे, तब भी पूरी बूट शृंखला चलती रहे।

WSL के अंदर:

```bash
sudo apt-get install -y dbus-x11
sudo loginctl enable-linger "$(whoami)"
openclaw gateway install
```

व्यवस्थापक के रूप में PowerShell में:

```powershell
schtasks /create /tn "WSL Boot" /tr "wsl.exe -d Ubuntu --exec dbus-launch true" /sc onstart /ru "$env:USERNAME"
```

`Ubuntu` को यहाँ से प्राप्त अपनी डिस्ट्रो के नाम से बदलें:

```powershell
wsl --list --verbose
```

<Note>
पुरानी विधियों से दो बदलाव:

- **`dbus-launch true`, `/bin/true` के बजाय**: WSL >= 2.6.1.0 पर एक
  रिग्रेशन ([microsoft/WSL #13416](https://github.com/microsoft/WSL/issues/13416))
  अंतिम क्लाइंट के बाहर निकलने के 15-20 सेकंड बाद डिस्ट्रो को निष्क्रियता के कारण बंद कर देता है, भले ही
  लिंगर सक्षम हो। समाधान के रूप में `dbus-launch true` एक child-of-init प्रक्रिया को सक्रिय रखता है
  (समुदाय चर्चा, [microsoft/WSL #9245](https://github.com/microsoft/WSL/discussions/9245))।
- **`/ru "$env:USERNAME"`, `/ru SYSTEM` के बजाय**: प्रति-उपयोगकर्ता WSL डिस्ट्रो (डिफ़ॉल्ट
  सेटअप) SYSTEM खाते को दिखाई नहीं देतीं, इसलिए टास्क चलता हुआ दिखता है,
  लेकिन डिस्ट्रो कभी शुरू नहीं होती। इसे अपने खाते से चलाने पर यह समस्या नहीं होती;
  टास्क बनाते समय Windows आपका पासवर्ड माँगता है।

</Note>

रीबूट के बाद WSL से सत्यापित करें:

```bash
systemctl --user is-enabled openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

## LAN पर WSL सेवाएँ उपलब्ध कराएँ

WSL का अपना वर्चुअल नेटवर्क है। यदि किसी अन्य मशीन को WSL के अंदर किसी सेवा तक पहुँचना हो,
तो Windows पोर्ट को वर्तमान WSL IP पर अग्रेषित करें। पुनः प्रारंभ होने के बाद WSL IP
बदल सकता है, इसलिए आवश्यकता होने पर अग्रेषण नियम रीफ़्रेश करें।

व्यवस्थापक के रूप में PowerShell में उदाहरण:

```powershell
$Distro = "Ubuntu-24.04"
$ListenPort = 2222
$TargetPort = 22

$WslIp = (wsl -d $Distro -- hostname -I).Trim().Split(" ")[0]
if (-not $WslIp) { throw "WSL IP नहीं मिला।" }

netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=$ListenPort `
  connectaddress=$WslIp connectport=$TargetPort

New-NetFirewallRule -DisplayName "WSL SSH $ListenPort" -Direction Inbound `
  -Protocol TCP -LocalPort $ListenPort -Action Allow
```

नोट्स:

- किसी अन्य मशीन से SSH, Windows होस्ट IP को लक्षित करता है, जैसे `ssh user@windows-host -p 2222`।
- रिमोट Nodes को पहुँच योग्य Gateway URL की ओर संकेत करना चाहिए, `127.0.0.1` की ओर नहीं।
- LAN पहुँच के लिए `listenaddress=0.0.0.0`, केवल स्थानीय पहुँच के लिए `127.0.0.1` का उपयोग करें।

## समस्या निवारण

### ट्रे आइकन दिखाई नहीं देता

Task Manager में `OpenClaw.Tray.WinUI.exe` की जाँच करें। यदि यह चल रहा है, तो
छिपे हुए ट्रे-आइकन क्षेत्र को खोलकर इसे पिन करें। यदि नहीं, तो Start मेनू से **OpenClaw Companion** लॉन्च करें।

### स्थानीय सेटअप विफल होता है

Windows Hub से सेटअप लॉग खोलें या इसका निरीक्षण करें:

```powershell
notepad "$env:LOCALAPPDATA\OpenClawTray\Logs\Setup\easy-setup-latest.txt"
```

सामान्य कारण: अक्षम WSL, अवरुद्ध वर्चुअलाइज़ेशन, पुरानी ऐप-स्वामित्व वाली WSL
स्थिति या Gateway पैकेज इंस्टॉल करते समय नेटवर्क विफलता।

### ऐप कहता है कि पेयरिंग आवश्यक है

Gateway से ऑपरेटर या Node अनुरोध स्वीकृत करें:

```powershell
openclaw devices list
openclaw devices approve <requestId>
```

यदि डिवाइस के पास पहले से टोकन था, तो स्वीकृति के बाद Connections टैब से
फिर से कनेक्ट करें।

### वेब चैट रिमोट Gateway तक नहीं पहुँच सकती

रिमोट वेब चैट के लिए HTTPS या localhost आवश्यक है। स्व-हस्ताक्षरित प्रमाणपत्रों के लिए,
Windows में प्रमाणपत्र पर विश्वास करें या localhost URL के लिए SSH टनल का उपयोग करें।

### `screen.snapshot`, कैमरा या ऑडियो कमांड विफल होते हैं

कैमरा, माइक्रोफ़ोन, स्क्रीन कैप्चर और सूचनाओं के लिए Windows अनुमतियों की पुष्टि करें।
पैकेज्ड इंस्टॉल संरक्षित क्षमताएँ घोषित करते हैं, लेकिन किसी कमांड द्वारा पहली बार
उनका उपयोग करने पर Windows फिर भी संकेत दे सकता है।

### Git या GitHub कनेक्टिविटी विफल होती है

कुछ नेटवर्क GitHub के HTTPS को ब्लॉक या सीमित करते हैं। यदि `git clone` या
`gh auth login` विफल होता है, तो दूसरा नेटवर्क, VPN या HTTP/HTTPS प्रॉक्सी आज़माएँ।

वर्तमान सत्र में टोकन-आधारित `gh` प्रमाणीकरण के लिए:

```powershell
$env:GH_TOKEN="<your-token>"
gh auth status
gh auth setup-git
```

टोकन कभी कमिट न करें या उन्हें इश्यू अथवा पुल रिक्वेस्ट में पेस्ट न करें।

## संबंधित

- [इंस्टॉल अवलोकन](/hi/install)
- [Node.js सेटअप](/hi/install/node)
- [Nodes](/hi/nodes)
- [Control UI](/hi/web/control-ui)
- [Gateway कॉन्फ़िगरेशन](/hi/gateway/configuration)
