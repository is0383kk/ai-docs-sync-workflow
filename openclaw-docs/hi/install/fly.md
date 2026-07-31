---
read_when:
    - Fly.io पर OpenClaw परिनियोजित करना
    - Fly वॉल्यूम, सीक्रेट और पहली बार चलाने की कॉन्फ़िगरेशन सेट अप करना
summary: स्थायी स्टोरेज और HTTPS के साथ OpenClaw को Fly.io पर डिप्लॉय करने की चरण-दर-चरण प्रक्रिया
title: Fly.io
x-i18n:
    generated_at: "2026-07-27T18:25:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d2b5119c1df8ee077f4db4f44fa92c6ae0e2bf3c355c2117e0fd39146bb49875
    source_path: install/fly.md
    workflow: 16
---

**लक्ष्य:** स्थायी स्टोरेज, स्वचालित HTTPS और Discord/चैनल एक्सेस के साथ [Fly.io](https://fly.io) मशीन पर चलने वाला OpenClaw Gateway।

## आपको क्या चाहिए

- [flyctl CLI](https://fly.io/docs/hands-on/install-flyctl/) इंस्टॉल किया हुआ
- Fly.io खाता (निःशुल्क टियर काम करता है)
- मॉडल प्रमाणीकरण: आपके चुने हुए मॉडल प्रदाता की API कुंजी
- चैनल क्रेडेंशियल: Discord बॉट टोकन, Telegram टोकन आदि।

## शुरुआती लोगों के लिए त्वरित तरीका

1. रेपो क्लोन करें, `fly.toml` को अनुकूलित करें
2. ऐप और वॉल्यूम बनाएँ, सीक्रेट सेट करें
3. `fly deploy` के साथ डिप्लॉय करें
4. कॉन्फ़िग बनाने के लिए SSH से कनेक्ट करें या Control UI का उपयोग करें

<Steps>
  <Step title="Fly ऐप बनाएँ">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw

    # अपना नाम चुनें
    fly apps create my-openclaw

    # सामान्यतः 1GB पर्याप्त है
    fly volumes create openclaw_data --size 1 --region iad
    ```

    अपने निकट का क्षेत्र चुनें। सामान्य विकल्प: `lhr` (लंदन), `iad` (वर्जीनिया), `sjc` (सैन होज़े)।

  </Step>

  <Step title="fly.toml कॉन्फ़िगर करें">
    अपने ऐप के नाम और आवश्यकताओं से मिलाने के लिए `fly.toml` संपादित करें। रेपो में ट्रैक किया गया `fly.toml` नीचे दिखाया गया सार्वजनिक टेम्पलेट है; `deploy/fly.private.toml` अधिक सुरक्षित, सार्वजनिक-IP-रहित संस्करण है ([निजी डिप्लॉयमेंट](#private-deployment-hardened) देखें)।

    ```toml
    app = "my-openclaw"  # आपके ऐप का नाम
    primary_region = "iad"

    [build]
      dockerfile = "Dockerfile"

    [env]
      NODE_ENV = "production"
      OPENCLAW_PREFER_PNPM = "1"
      OPENCLAW_STATE_DIR = "/data"
      NODE_OPTIONS = "--max-old-space-size=1536"

    [processes]
      app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"

    [http_service]
      internal_port = 3000
      force_https = true
      auto_stop_machines = false
      auto_start_machines = true
      min_machines_running = 1
      processes = ["app"]

    [[vm]]
      size = "shared-cpu-2x"
      memory = "2048mb"

    [mounts]
      source = "openclaw_data"
      destination = "/data"
    ```

    OpenClaw Docker इमेज का एंट्रीपॉइंट `tini` है, जो डिफ़ॉल्ट रूप से `node openclaw.mjs gateway` चलाता है। Fly `[processes]`, Docker के `CMD` को बदल देता है (यहाँ यह `node dist/index.js gateway ...`, उसी कंपाइल किए गए एंट्रीपॉइंट, को सीधे चलाता है), लेकिन `ENTRYPOINT` को नहीं बदलता, इसलिए प्रक्रिया अब भी `tini` के अंतर्गत चलती है।

    **मुख्य सेटिंग्स:**

    | सेटिंग                        | कारण                                                                         |
    | ------------------------------ | --------------------------------------------------------------------------- |
    | `--bind lan`                   | `0.0.0.0` से बाइंड करता है, ताकि Fly का प्रॉक्सी Gateway तक पहुँच सके                     |
    | `--allow-unconfigured`         | कॉन्फ़िग फ़ाइल के बिना शुरू करता है (आप इसे बाद में बनाते हैं)                        |
    | `internal_port = 3000`         | Fly स्वास्थ्य जाँचों के लिए `--port 3000` (या `OPENCLAW_GATEWAY_PORT`) से मेल खाना आवश्यक है |
    | `memory = "2048mb"`            | 512MB बहुत कम है; 2GB अनुशंसित है                                         |
    | `OPENCLAW_STATE_DIR = "/data"` | स्थिति को वॉल्यूम पर स्थायी रखता है                                                |

  </Step>

  <Step title="सीक्रेट सेट करें">
    ```bash
    # आवश्यक: गैर-लूपबैक बाइंडिंग के लिए Gateway प्रमाणीकरण टोकन
    fly secrets set OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)

    # मॉडल प्रदाता की API कुंजियाँ
    fly secrets set ANTHROPIC_API_KEY=example-anthropic-key-not-real

    # वैकल्पिक: अन्य प्रदाता
    fly secrets set OPENAI_API_KEY=example-openai-key-not-real
    fly secrets set GOOGLE_API_KEY=...

    # चैनल टोकन
    fly secrets set DISCORD_BOT_TOKEN=example-discord-bot-token
    ```

    गैर-लूपबैक बाइंड (`--bind lan`) के लिए एक मान्य Gateway प्रमाणीकरण पथ आवश्यक है। यह उदाहरण `OPENCLAW_GATEWAY_TOKEN` का उपयोग करता है, लेकिन `gateway.auth.password` या सही ढंग से कॉन्फ़िगर किया गया गैर-लूपबैक विश्वसनीय-प्रॉक्सी डिप्लॉयमेंट भी आवश्यकता पूरी करता है। SecretRef अनुबंध के लिए [सीक्रेट प्रबंधन](/hi/gateway/secrets) देखें।

    इन टोकनों को पासवर्ड की तरह सुरक्षित रखें। API कुंजियों और टोकनों के लिए कॉन्फ़िग फ़ाइल के बजाय env vars/`fly secrets` को प्राथमिकता दें, ताकि सीक्रेट `openclaw.json` से बाहर रहें।

  </Step>

  <Step title="डिप्लॉय करें">
    ```bash
    fly deploy
    ```

    पहला डिप्लॉय Docker इमेज बनाता है। डिप्लॉयमेंट के बाद सत्यापित करें:

    ```bash
    fly status
    fly logs
    ```

    HTTP/WebSocket लिसनर शुरू होने पर Gateway स्टार्टअप लॉग एक बार `gateway ready` दर्ज करते हैं। Fly की अपनी स्वास्थ्य जाँच `fly.toml` के अनुसार `internal_port = 3000` पर नज़र रखती है; इमेज का Docker `HEALTHCHECK` निर्देश अतिरिक्त रूप से उसके डिफ़ॉल्ट पोर्ट 18789 पर `/healthz` को पोल करता है, जिसका यहाँ उपयोग नहीं होता क्योंकि यह डिप्लॉयमेंट Gateway को `--port 3000` पर ओवरराइड करता है।

  </Step>

  <Step title="कॉन्फ़िग फ़ाइल बनाएँ">
    उचित कॉन्फ़िग बनाने के लिए SSH से मशीन में कनेक्ट करें:

    ```bash
    fly ssh console
    ```

    ```bash
    mkdir -p /data
    cat > /data/openclaw.json << 'EOF'
    {
      "agents": {
        "defaults": {
          "model": {
            "primary": "anthropic/claude-opus-4-6",
            "fallbacks": ["anthropic/claude-sonnet-4-6", "openai/gpt-5.4"]
          },
          "maxConcurrent": 4
        },
        "list": [
          {
            "id": "main",
            "default": true
          }
        ]
      },
      "auth": {
        "profiles": {
          "anthropic:default": { "mode": "token", "provider": "anthropic" },
          "openai:default": { "mode": "token", "provider": "openai" }
        }
      },
      "bindings": [
        {
          "agentId": "main",
          "match": { "channel": "discord" }
        }
      ],
      "channels": {
        "discord": {
          "enabled": true,
          "groupPolicy": "allowlist",
          "guilds": {
            "YOUR_GUILD_ID": {
              "channels": { "general": { "allow": true } },
              "requireMention": false
            }
          }
        }
      },
      "gateway": {
        "mode": "local",
        "bind": "auto",
        "controlUi": {
          "allowedOrigins": [
            "https://my-openclaw.fly.dev",
            "http://localhost:3000",
            "http://127.0.0.1:3000"
          ]
        }
      },
      "meta": {}
    }
    EOF
    ```

    `OPENCLAW_STATE_DIR=/data` के साथ, कॉन्फ़िग पथ `/data/openclaw.json` है।

    `https://my-openclaw.fly.dev` को अपने वास्तविक Fly ऐप ओरिजिन से बदलें। Gateway स्टार्टअप रनटाइम के `--bind` और `--port` मानों से स्थानीय Control UI ओरिजिन आरंभिक रूप से भरता है, ताकि कॉन्फ़िग मौजूद होने से पहले पहला बूट आगे बढ़ सके, लेकिन Fly के माध्यम से ब्राउज़र एक्सेस के लिए अब भी `gateway.controlUi.allowedOrigins` में सटीक HTTPS ओरिजिन सूचीबद्ध होना आवश्यक है।

    Discord टोकन इनमें से किसी भी स्रोत से आ सकता है:

    - पर्यावरण चर `DISCORD_BOT_TOKEN` (सीक्रेट के लिए अनुशंसित); इसे कॉन्फ़िग में जोड़ने की आवश्यकता नहीं है, Gateway इसे स्वचालित रूप से पढ़ता है
    - कॉन्फ़िग फ़ाइल `channels.discord.token`

    लागू करने के लिए पुनः आरंभ करें:

    ```bash
    exit
    fly machine restart <machine-id>
    ```

  </Step>

  <Step title="Gateway एक्सेस करें">
    ### Control UI

    ```bash
    fly open
    ```

    या `https://my-openclaw.fly.dev/` पर जाएँ।

    कॉन्फ़िगर किए गए साझा सीक्रेट से प्रमाणीकरण करें: `OPENCLAW_GATEWAY_TOKEN` का Gateway टोकन, या यदि आपने पासवर्ड प्रमाणीकरण अपनाया है तो आपका पासवर्ड।

    ### लॉग

    ```bash
    fly logs              # लाइव लॉग
    fly logs --no-tail    # हाल के लॉग
    ```

    ### SSH कंसोल

    ```bash
    fly ssh console
    ```

  </Step>
</Steps>

## समस्या निवारण

### "ऐप अपेक्षित पते पर नहीं सुन रहा है"

Gateway `0.0.0.0` के बजाय `127.0.0.1` से बाइंड हो रहा है।

**समाधान:** `fly.toml` में अपने प्रक्रिया कमांड में `--bind lan` जोड़ें।

### स्वास्थ्य जाँच विफल / कनेक्शन अस्वीकृत

Fly कॉन्फ़िगर किए गए पोर्ट पर Gateway तक नहीं पहुँच सकता।

**समाधान:** सुनिश्चित करें कि `internal_port`, Gateway पोर्ट (`--port 3000` या `OPENCLAW_GATEWAY_PORT=3000`) से मेल खाता है।

### OOM / मेमोरी समस्याएँ

कंटेनर बार-बार पुनः आरंभ हो रहा है या बंद किया जा रहा है। संकेत: `SIGABRT`, `v8::internal::Runtime_AllocateInYoungGeneration`, या बिना किसी संदेश के पुनः आरंभ होना।

**समाधान:** `fly.toml` में मेमोरी बढ़ाएँ:

```toml
[[vm]]
  memory = "2048mb"
```

या किसी मौजूदा मशीन को अपडेट करें:

```bash
fly machine update <machine-id> --vm-memory 2048 -y
```

512MB बहुत कम है। 1GB काम कर सकता है, लेकिन लोड या विस्तृत लॉगिंग के दौरान OOM हो सकता है। 2GB अनुशंसित है।

### Gateway लॉक समस्याएँ

कंटेनर पुनः आरंभ होने के बाद Gateway "पहले से चल रहा है" त्रुटियों के कारण शुरू होने से मना करता है।

रनटाइम लॉक फ़ाइलें `<tmpdir>/openclaw-<uid>/gateway.<hash>.lock`
और `gateway.state.<hash>.lock` (Linux:
`/tmp/openclaw-<uid>/gateway.*.lock`) पर रहती हैं, स्थायी `/data` वॉल्यूम पर नहीं, इसलिए
कंटेनर को पूरी तरह पुनः आरंभ करने पर वे सामान्यतः शेष
कंटेनर फ़ाइल सिस्टम के साथ साफ़ हो जाती हैं। यदि कोई लॉक बना रहता है (उदाहरण के लिए ऐसा `fly machine restart`
जो कंटेनर फ़ाइल सिस्टम को सुरक्षित रखता है) और स्टार्टअप को अवरुद्ध करता है, तो उसे
मैन्युअल रूप से हटाएँ:

```bash
fly ssh console --command "rm -f /tmp/openclaw-*/gateway.*.lock"
fly machine restart <machine-id>
```

### कॉन्फ़िग पढ़ा नहीं जा रहा

`--allow-unconfigured` केवल स्टार्टअप सुरक्षा जाँच को बायपास करता है। यह `/data/openclaw.json` को बनाता या सुधारता नहीं है, इसलिए सुनिश्चित करें कि आपका वास्तविक कॉन्फ़िग मौजूद है और सामान्य स्थानीय Gateway स्टार्ट के लिए उसमें `"gateway": { "mode": "local" }` शामिल है।

सत्यापित करें कि कॉन्फ़िग मौजूद है:

```bash
fly ssh console --command "cat /data/openclaw.json"
```

### SSH के माध्यम से कॉन्फ़िग लिखना

`fly ssh console -C` शेल रीडायरेक्शन का समर्थन नहीं करता। कॉन्फ़िग फ़ाइल लिखने के लिए:

```bash
# echo + tee (स्थानीय से रिमोट पर पाइप करें)
echo '{"your":"config"}' | fly ssh console -C "tee /data/openclaw.json"

# या sftp
fly sftp shell
> put /local/path/config.json /data/openclaw.json
```

यदि फ़ाइल पहले से मौजूद है, तो `fly sftp` विफल हो सकता है; पहले उसे हटाएँ:

```bash
fly ssh console --command "rm /data/openclaw.json"
```

### स्थिति स्थायी नहीं रह रही

यदि पुनः आरंभ करने के बाद आपके प्रमाणीकरण प्रोफ़ाइल, चैनल/प्रदाता स्थिति या सत्र खो जाते हैं, तो स्टेट डायरेक्टरी वॉल्यूम के बजाय कंटेनर फ़ाइल सिस्टम में लिख रही है।

**समाधान:** सुनिश्चित करें कि `OPENCLAW_STATE_DIR=/data`, `fly.toml` में सेट है और फिर से डिप्लॉय करें।

## अपडेट करना

```bash
git pull
fly deploy
fly status
fly logs
```

यहाँ `git pull` + `fly deploy` पर्यवेक्षित तरीका है: यह Dockerfile से इमेज को फिर से बनाता है, इसलिए CLI/Gateway संस्करण, आधार OS इमेज और Dockerfile में हुए सभी बदलाव एक साथ अपडेट होते हैं। चल रहे कंटेनर के अंदर `openclaw update` वही प्रक्रिया नहीं है, क्योंकि इमेज Docker द्वारा निर्मित `dist/` ट्री के रूप में आती है, जिसमें उसके द्वारा पहचानने के लिए न तो `.git` चेकआउट होता है, न ही npm-प्रबंधित वैश्विक इंस्टॉलेशन; VM-जैसे इंस्टॉलेशन पर उस प्रक्रिया के लिए [अपडेट करना](/hi/install/updating) देखें।

### मशीन कमांड अपडेट करना

पूर्ण पुनः डिप्लॉय किए बिना स्टार्टअप कमांड बदलने के लिए:

```bash
fly machines list
fly machine update <machine-id> --command "node dist/index.js gateway --port 3000 --bind lan" -y

# या मेमोरी बढ़ाने के साथ
fly machine update <machine-id> --vm-memory 2048 --command "node dist/index.js gateway --port 3000 --bind lan" -y
```

बाद का `fly deploy` मशीन कमांड को `fly.toml` में मौजूद मान पर रीसेट कर देता है; पुनः डिप्लॉय करने के बाद मैन्युअल बदलाव फिर से लागू करें।

## निजी डिप्लॉयमेंट (अधिक सुरक्षित)

डिफ़ॉल्ट रूप से Fly सार्वजनिक IP आवंटित करता है, इसलिए आपका Gateway `https://your-app.fly.dev` पर उपलब्ध होता है और इंटरनेट स्कैनर (Shodan, Censys आदि) उसे खोज सकते हैं।

**बिना सार्वजनिक IP** वाले अधिक सुरक्षित डिप्लॉयमेंट के लिए `deploy/fly.private.toml` का उपयोग करें: इसमें `[http_service]` शामिल नहीं है, इसलिए कोई सार्वजनिक इनग्रेस आवंटित नहीं होता।

### निजी डिप्लॉयमेंट का उपयोग कब करें

- केवल आउटबाउंड कॉल/संदेश (कोई इनबाउंड Webhook नहीं)
- ngrok या Tailscale टनल किसी भी Webhook कॉलबैक को संभालती हैं
- Gateway एक्सेस ब्राउज़र के बजाय SSH, प्रॉक्सी या WireGuard के माध्यम से होता है
- डिप्लॉयमेंट को इंटरनेट स्कैनरों से छिपाया जाना चाहिए

### सेटअप

```bash
fly deploy -c deploy/fly.private.toml
```

या किसी मौजूदा डिप्लॉयमेंट को रूपांतरित करें:

```bash
# वर्तमान IP सूचीबद्ध करें
fly ips list -a my-openclaw

# सार्वजनिक IP जारी करें
fly ips release <public-ipv4> -a my-openclaw
fly ips release <public-ipv6> -a my-openclaw

# निजी कॉन्फ़िगरेशन पर स्विच करें, ताकि भविष्य के डिप्लॉयमेंट सार्वजनिक IP फिर से आवंटित न करें
fly deploy -c deploy/fly.private.toml

# केवल-निजी IPv6 आवंटित करें
fly ips allocate-v6 --private -a my-openclaw
```

इसके बाद, `fly ips list` में केवल `private` प्रकार का IP दिखाई देना चाहिए:

```text
संस्करण  IP                   प्रकार           क्षेत्र
v6       fdaa:x:x:x:x::x      निजी             वैश्विक
```

### निजी डिप्लॉयमेंट तक पहुँचना

**विकल्प 1: स्थानीय प्रॉक्सी (सबसे सरल)**

```bash
fly proxy 3000:3000 -a my-openclaw
# ब्राउज़र में http://localhost:3000 खोलें
```

**विकल्प 2: WireGuard VPN**

```bash
fly wireguard create
# इसे WireGuard क्लाइंट में इम्पोर्ट करें, फिर आंतरिक IPv6 के माध्यम से पहुँचें
# उदाहरण: http://[fdaa:x:x:x:x::x]:3000
```

**विकल्प 3: केवल SSH**

```bash
fly ssh console -a my-openclaw
```

### निजी डिप्लॉयमेंट के साथ Webhook

सार्वजनिक रूप से उजागर किए बिना Webhook कॉलबैक (Twilio, Telnyx आदि) के लिए:

1. **ngrok टनल**: ngrok को कंटेनर के अंदर या साइडकार के रूप में चलाएँ
2. **Tailscale Funnel**: Tailscale के माध्यम से विशिष्ट पाथ उजागर करें
3. **केवल आउटबाउंड**: कुछ प्रदाता (Twilio) Webhook के बिना आउटबाउंड कॉल के लिए काम करते हैं

`plugins.entries.voice-call.config` के अंतर्गत ngrok के साथ वॉइस-कॉल कॉन्फ़िगरेशन का उदाहरण:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          tunnel: { provider: "ngrok" },
          webhookSecurity: {
            allowedHosts: ["example.ngrok.app"],
          },
        },
      },
    },
  },
}
```

ngrok टनल कंटेनर के अंदर चलती है और Fly ऐप को स्वयं उजागर किए बिना एक सार्वजनिक Webhook URL प्रदान करती है। फ़ॉरवर्ड किए गए होस्ट हेडर स्वीकार करने के लिए `webhookSecurity.allowedHosts` को टनल होस्टनेम पर सेट करें।

### सुरक्षा संबंधी समझौते

| पहलू              | सार्वजनिक       | निजी          |
| ----------------- | ------------ | ---------- |
| इंटरनेट स्कैनर    | खोजने योग्य     | छिपा हुआ      |
| प्रत्यक्ष हमले     | संभव            | अवरुद्ध        |
| कंट्रोल UI पहुँच  | ब्राउज़र         | प्रॉक्सी/VPN   |
| Webhook डिलीवरी   | प्रत्यक्ष        | टनल के माध्यम से |

## टिप्पणियाँ

- Fly.io x86 आर्किटेक्चर का उपयोग करता है; Dockerfile x86 और ARM दोनों के साथ संगत है।
- WhatsApp/Telegram ऑनबोर्डिंग के लिए, `fly ssh console` का उपयोग करें।
- स्थायी डेटा `/data` पर स्थित वॉल्यूम में रहता है।
- Signal के लिए इमेज पर signal-cli (Java-आधारित CLI) आवश्यक है; कस्टम इमेज का उपयोग करें और मेमोरी 2GB+ रखें।

## लागत

अनुशंसित कॉन्फ़िगरेशन (`shared-cpu-2x`, 2GB RAM) के साथ, उपयोग के आधार पर लगभग $10-15/माह की अपेक्षा करें; मुफ़्त टियर कुछ आधारभूत भत्ते को कवर करता है। वर्तमान दरों के लिए [Fly.io की मूल्य-निर्धारण जानकारी](https://fly.io/docs/about/pricing/) देखें।

## अगले चरण

- मैसेजिंग चैनल सेट अप करें: [चैनल](/hi/channels)
- Gateway कॉन्फ़िगर करें: [Gateway कॉन्फ़िगरेशन](/hi/gateway/configuration)
- OpenClaw को अद्यतित रखें: [अपडेट करना](/hi/install/updating)

## संबंधित

- [इंस्टॉलेशन का अवलोकन](/hi/install)
- [Hetzner](/hi/install/hetzner)
- [Docker](/hi/install/docker)
- [VPS होस्टिंग](/hi/vps)
