---
read_when:
    - आप Gateway को किसी Linux सर्वर या क्लाउड VPS पर चलाना चाहते हैं
    - आपको होस्टिंग गाइडों की एक संक्षिप्त रूपरेखा चाहिए
    - आप OpenClaw के लिए सामान्य Linux सर्वर ट्यूनिंग चाहते हैं
sidebarTitle: Linux Server
summary: किसी Linux सर्वर या क्लाउड VPS पर OpenClaw चलाएँ — प्रदाता चयन, आर्किटेक्चर और ट्यूनिंग
title: Linux सर्वर
x-i18n:
    generated_at: "2026-07-27T20:13:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 634a246850ab8b854c2c799688fd368ebed3a02124baa85bf38d5ff6ef8cec64
    source_path: vps.md
    workflow: 16
---

किसी भी Linux सर्वर या क्लाउड VPS पर OpenClaw Gateway चलाएँ। यह पृष्ठ आपको
प्रदाता चुनने में सहायता करता है, बताता है कि क्लाउड परिनियोजन कैसे काम करते हैं, और सामान्य Linux
ट्यूनिंग को शामिल करता है जो हर जगह लागू होती है।

## प्रदाता चुनें

<CardGroup cols={2}>
  <Card title="Azure" href="/hi/install/azure">Linux VM</Card>
  <Card title="DigitalOcean" href="/hi/install/digitalocean">सरल सशुल्क VPS</Card>
  <Card title="exe.dev" href="/hi/install/exe-dev">HTTPS प्रॉक्सी वाला VM</Card>
  <Card title="Fly.io" href="/hi/install/fly">Fly Machines</Card>
  <Card title="GCP" href="/hi/install/gcp">Compute Engine</Card>
  <Card title="Hetzner" href="/hi/install/hetzner">Hetzner VPS पर Docker</Card>
  <Card title="Hostinger" href="/hi/install/hostinger">एक-क्लिक सेटअप वाला VPS</Card>
  <Card title="Northflank" href="/hi/install/northflank">एक-क्लिक, ब्राउज़र सेटअप</Card>
  <Card title="Oracle Cloud" href="/hi/install/oracle">हमेशा निःशुल्क ARM टियर</Card>
  <Card title="Railway" href="/hi/install/railway">एक-क्लिक, ब्राउज़र सेटअप</Card>
  <Card title="Raspberry Pi" href="/hi/install/raspberry-pi">ARM स्व-होस्टेड</Card>
</CardGroup>

**AWS (EC2 / Lightsail / निःशुल्क टियर)** भी अच्छी तरह काम करता है।
समुदाय का एक वीडियो मार्गदर्शन यहाँ उपलब्ध है:
[x.com/techfrenAJ/status/2014934471095812547](https://x.com/techfrenAJ/status/2014934471095812547)
(सामुदायिक संसाधन -- अनुपलब्ध हो सकता है)।

## क्लाउड सेटअप कैसे काम करते हैं

- **Gateway VPS पर चलता है** और स्टेट + वर्कस्पेस का स्वामी होता है।
- आप अपने लैपटॉप या फ़ोन से **कंट्रोल UI** या **Tailscale/SSH** के माध्यम से कनेक्ट करते हैं।
- VPS को सत्य का स्रोत मानें और स्टेट + वर्कस्पेस का नियमित रूप से **बैकअप लें**।
- सुरक्षित डिफ़ॉल्ट: Gateway को लूपबैक पर रखें और SSH टनल या Tailscale Serve के माध्यम से उस तक पहुँचें।
  यदि आप `lan` या `tailnet` से बाइंड करते हैं, तो Gateway को साझा सीक्रेट
  (`gateway.auth.token` या `gateway.auth.password`) की आवश्यकता होती है, जब तक कि प्रमाणीकरण किसी
  विश्वसनीय प्रॉक्सी को प्रत्यायोजित न किया गया हो।

संबंधित पृष्ठ: [Gateway की दूरस्थ पहुँच](/hi/gateway/remote), [प्लेटफ़ॉर्म केंद्र](/hi/platforms)।

## पहले एडमिन पहुँच को सुदृढ़ करें

किसी सार्वजनिक VPS पर OpenClaw इंस्टॉल करने से पहले, तय करें कि आप स्वयं
उस मशीन का प्रशासन कैसे करना चाहते हैं।

- केवल Tailnet से एडमिन पहुँच के लिए: पहले Tailscale इंस्टॉल करें, VPS को अपने
  टेलनेट से जोड़ें, Tailscale IP या MagicDNS नाम के माध्यम से दूसरे SSH सत्र की पुष्टि करें,
  फिर सार्वजनिक SSH को प्रतिबंधित करें।
- Tailscale के बिना: अधिक सेवाएँ एक्सपोज़ करने से पहले
  अपने SSH पथ पर समतुल्य सुरक्षा सुदृढ़ीकरण लागू करें।
- यह Gateway पहुँच से अलग है। आप फिर भी OpenClaw को
  लूपबैक से बाइंड रख सकते हैं और डैशबोर्ड के लिए SSH टनल या Tailscale Serve का उपयोग कर सकते हैं।

Tailscale-विशिष्ट Gateway विकल्प [Tailscale](/hi/gateway/tailscale) में दिए गए हैं।

## VPS पर साझा कंपनी एजेंट

जब प्रत्येक उपयोगकर्ता समान विश्वास सीमा में हो और एजेंट का उपयोग केवल व्यवसाय के लिए किया जाए,
तब टीम के लिए एकल एजेंट चलाना एक वैध सेटअप है।

- इसे समर्पित रनटाइम (VPS/VM/कंटेनर + समर्पित OS उपयोगकर्ता/अकाउंट) पर रखें।
- उस रनटाइम को व्यक्तिगत Apple/Google अकाउंट या व्यक्तिगत ब्राउज़र/पासवर्ड-मैनेजर प्रोफ़ाइल में साइन इन न करें।
- यदि उपयोगकर्ता एक-दूसरे के प्रति शत्रुतापूर्ण हैं, तो Gateway/होस्ट/OS उपयोगकर्ता के आधार पर विभाजित करें।

सुरक्षा मॉडल का विवरण: [सुरक्षा](/hi/gateway/security)।

## VPS के साथ Node का उपयोग

आप Gateway को क्लाउड में रख सकते हैं और अपने स्थानीय डिवाइस
(Mac/iOS/Android/हेडलेस) पर **Node** पेयर कर सकते हैं। Gateway के क्लाउड में बने रहने के दौरान Node स्थानीय स्क्रीन/कैमरा/कैनवास और `system.run`
क्षमताएँ प्रदान करते हैं।

दस्तावेज़: [Node](/hi/nodes), [Node CLI](/hi/cli/nodes)।

## छोटे VM और ARM होस्ट के लिए स्टार्टअप ट्यूनिंग

यदि कम-शक्ति वाले VM (या ARM होस्ट) पर CLI कमांड धीमी लगती हैं, तो Node का मॉड्यूल कंपाइल कैश सक्षम करें:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

- `NODE_COMPILE_CACHE` बार-बार चलाए जाने वाले कमांड के स्टार्टअप समय को बेहतर बनाता है; पहली बार चलने पर कैश तैयार होता है।
- `OPENCLAW_NO_RESPAWN=1` नियमित Gateway रीस्टार्ट को उसी प्रोसेस में रखता है, जिससे अतिरिक्त प्रोसेस हस्तांतरण से बचा जाता है और छोटे होस्ट पर PID ट्रैकिंग सरल रहती है।
- Raspberry Pi से संबंधित विवरण के लिए, [Raspberry Pi](/hi/install/raspberry-pi) देखें।

### systemd ट्यूनिंग चेकलिस्ट (वैकल्पिक)

`systemd` का उपयोग करने वाले VM होस्ट के लिए, इन पर विचार करें:

- स्थिर स्टार्टअप पथ के लिए सेवा परिवेश: `OPENCLAW_NO_RESPAWN=1` और
  `NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache`
- स्पष्ट रीस्टार्ट व्यवहार: `Restart=always`, `RestartSec=2`, `TimeoutStartSec=90`
- रैंडम-I/O कोल्ड-स्टार्ट दंड कम करने के लिए स्टेट/कैश पथों हेतु SSD-समर्थित डिस्क।

मानक `openclaw onboard --install-daemon` पथ एक systemd उपयोगकर्ता
यूनिट इंस्टॉल करता है; इसे इससे संपादित करें:

```bash
systemctl --user edit openclaw-gateway.service
```

```ini
[Service]
Environment=OPENCLAW_NO_RESPAWN=1
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Restart=always
RestartSec=2
TimeoutStartSec=90
```

यदि आपने जानबूझकर इसके बजाय सिस्टम यूनिट इंस्टॉल की है, तो इसे
`sudo systemctl edit openclaw-gateway.service` के माध्यम से संपादित करें।

`Restart=` नीतियाँ स्वचालित पुनर्प्राप्ति में कैसे सहायता करती हैं:
[systemd सेवा पुनर्प्राप्ति को स्वचालित कर सकता है](https://www.redhat.com/en/blog/systemd-automate-recovery)।

Linux OOM व्यवहार, चाइल्ड प्रोसेस विक्टिम चयन और `exit 137`
निदान के लिए, [Linux मेमोरी दबाव और OOM किल](/hi/platforms/linux#memory-pressure-and-oom-kills) देखें।

## संबंधित

- [इंस्टॉलेशन का अवलोकन](/hi/install)
- [DigitalOcean](/hi/install/digitalocean)
- [Fly.io](/hi/install/fly)
- [Hetzner](/hi/install/hetzner)
