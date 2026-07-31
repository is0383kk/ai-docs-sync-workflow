---
read_when:
    - आप स्थानीय इंस्टॉलेशन के बजाय कंटेनरयुक्त Gateway चाहते हैं
    - आप Docker प्रवाह को सत्यापित कर रहे हैं
summary: OpenClaw के लिए वैकल्पिक Docker-आधारित सेटअप और ऑनबोर्डिंग
title: Docker
x-i18n:
    generated_at: "2026-07-27T18:01:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1784bd49f6847db75633840a4d5a8e49205200728bd2e9d59b646a446e508d6
    source_path: install/docker.md
    workflow: 16
---

Docker **वैकल्पिक** है। इसका उपयोग किसी पृथक, अस्थायी Gateway परिवेश या स्थानीय इंस्टॉलेशन रहित होस्ट के लिए करें। यदि आप पहले से अपनी मशीन पर डेवलपमेंट करते हैं, तो इसके बजाय सामान्य इंस्टॉलेशन प्रवाह का उपयोग करें।

`agents.defaults.sandbox` सक्षम होने पर डिफ़ॉल्ट सैंडबॉक्स बैकएंड Docker का उपयोग करता है, लेकिन सैंडबॉक्सिंग डिफ़ॉल्ट रूप से बंद होती है और इसके लिए स्वयं Gateway को Docker में चलाने की आवश्यकता नहीं है। SSH और OpenShell सैंडबॉक्स बैकएंड भी उपलब्ध हैं; [सैंडबॉक्सिंग](/hi/gateway/sandboxing) देखें।

कई उपयोगकर्ताओं को होस्ट कर रहे हैं? प्रति टेनेंट एक सेल मॉडल के लिए [मल्टी-टेनेंट होस्टिंग](/hi/gateway/multi-tenant-hosting) देखें।

## पूर्वापेक्षाएँ

- Docker Desktop (या Docker Engine) + Docker Compose v2
- इमेज बिल्ड के लिए कम-से-कम 2 GB RAM (1 GB होस्ट पर `pnpm install` को निकास 137 के साथ OOM के कारण बंद किया जा सकता है)
- इमेज और लॉग के लिए पर्याप्त डिस्क स्थान
- VPS/सार्वजनिक होस्ट पर [नेटवर्क एक्सपोज़र के लिए सुरक्षा सुदृढ़ीकरण](/hi/gateway/security), विशेष रूप से Docker की `DOCKER-USER` फ़ायरवॉल शृंखला, की समीक्षा करें

## कंटेनरीकृत Gateway

<Steps>
  <Step title="इमेज बनाएँ">
    रेपो रूट से:

    ```bash
    ./scripts/docker/setup.sh
    ```

    यह Gateway इमेज को स्थानीय रूप से `openclaw:local` के रूप में बनाता है। इसके बजाय पहले से बनी इमेज का उपयोग करने के लिए:

    ```bash
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    पहले से बनी इमेज सबसे पहले [GitHub Container Registry](https://github.com/openclaw/openclaw/pkgs/container/openclaw) पर प्रकाशित की जाती हैं। GHCR रिलीज़ स्वचालन, पिन किए गए डिप्लॉयमेंट और प्रोवेनेंस जाँच के लिए प्राथमिक रजिस्ट्री है। यही रिलीज़ `openclaw/openclaw` पर Docker Hub मिरर भी प्रकाशित करती है:

    ```bash
    export OPENCLAW_IMAGE="openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    `ghcr.io/openclaw/openclaw` या `openclaw/openclaw` का उपयोग करें और अनाधिकारिक मिरर से बचें, क्योंकि उनकी रिलीज़ समय-सारणी या प्रतिधारण नीति OpenClaw के समान नहीं होती। संस्करण-विशिष्ट टैग में `2026.2.26` जैसे रिलीज़ और `2026.2.26-beta.1` जैसे प्रीरिलीज़ शामिल हैं। स्थिर रिलीज़ `latest` और `main` को आगे बढ़ाती हैं; अंतिम-माह वाली Gateway रिलीज़ केवल `extended-stable` को आगे बढ़ाती हैं। वेरिएंट में `slim`, `main-slim`, `extended-stable-slim`, `latest-browser`, `main-browser`, और `extended-stable-browser` शामिल हैं। डिफ़ॉल्ट इमेज में `codex` और `diagnostics-otel` Plugin बंडल होते हैं। `-browser` वेरिएंट Chromium को पहले से शामिल करके भी प्रदान किया जाता है, जो पहली बार Playwright इंस्टॉल किए बिना [सैंडबॉक्स किए गए ब्राउज़र](/hi/gateway/sandboxing#sandboxed-browser) टूल के लिए उपयोगी है।

  </Step>

  <Step title="एयर-गैप्ड पुनः संचालन">
    ऑफ़लाइन होस्ट पर पहले इमेज को स्थानांतरित और लोड करें:

    ```bash
    docker load -i openclaw-image.tar
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh --offline
    ```

    `--offline` सत्यापित करता है कि `OPENCLAW_IMAGE` स्थानीय रूप से पहले से मौजूद है, अंतर्निहित Compose पुल/बिल्ड अक्षम करता है, फिर सामान्य प्रवाह चलाता है: `.env` सिंक, अनुमति सुधार, ऑनबोर्डिंग, Gateway कॉन्फ़िगरेशन सिंक और Compose स्टार्टअप।

    यदि `OPENCLAW_SANDBOX=1`, तो ऑफ़लाइन सेटअप `OPENCLAW_DOCKER_SOCKET` के पीछे मौजूद डेमन पर कॉन्फ़िगर की गई डिफ़ॉल्ट और प्रति-एजेंट सैंडबॉक्स इमेज की भी जाँच करता है, जिसमें Docker-आधारित ब्राउज़र इमेज पर ब्राउज़र-अनुबंध लेबल शामिल है। यदि कोई आवश्यक इमेज अनुपस्थित या पुरानी है, तो सेटअप विफल सफलता की सूचना देने के बजाय सैंडबॉक्स कॉन्फ़िगरेशन बदले बिना समाप्त हो जाता है।

  </Step>

  <Step title="ऑनबोर्डिंग पूरी करें">
    सेटअप स्क्रिप्ट स्वचालित रूप से ऑनबोर्डिंग चलाती है:

    - प्रदाता API कुंजियों के लिए संकेत देती है
    - Gateway टोकन जनरेट करके उसे `.env` में लिखती है
    - प्रमाणीकरण-प्रोफ़ाइल गुप्त कुंजी डायरेक्टरी बनाती है
    - Docker Compose के माध्यम से Gateway शुरू करती है

    प्रारंभ-पूर्व ऑनबोर्डिंग और कॉन्फ़िगरेशन लेखन सीधे `openclaw-gateway` के माध्यम से (`--no-deps --entrypoint node` के साथ) चलते हैं, क्योंकि `openclaw-cli` Gateway का नेटवर्क नेमस्पेस साझा करता है और केवल Gateway कंटेनर के मौजूद होने के बाद काम करता है।

  </Step>

  <Step title="नियंत्रण UI खोलें">
    `http://127.0.0.1:18789/` खोलें और `.env` में लिखा टोकन सेटिंग्स में चिपकाएँ। यदि आपने कंटेनर को पासवर्ड प्रमाणीकरण पर स्विच किया है, तो इसके बजाय उस पासवर्ड का उपयोग करें।

    URL फिर से चाहिए?

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

  </Step>

  <Step title="चैनल कॉन्फ़िगर करें (वैकल्पिक)">
    ```bash
    # WhatsApp (QR)
    docker compose run --rm openclaw-cli channels login

    # Telegram
    docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

    # Discord
    docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
    ```

    दस्तावेज़: [WhatsApp](/hi/channels/whatsapp), [Telegram](/hi/channels/telegram), [Discord](/hi/channels/discord)

  </Step>
</Steps>

### मैन्युअल प्रवाह

```bash
BUILD_GIT_COMMIT="$(git rev-parse HEAD)"
BUILD_TIMESTAMP="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
docker build \
  --build-arg "GIT_COMMIT=${BUILD_GIT_COMMIT}" \
  --build-arg "OPENCLAW_BUILD_TIMESTAMP=${BUILD_TIMESTAMP}" \
  -t openclaw:local -f Dockerfile .
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js onboard --mode local --no-install-daemon
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"},{"path":"gateway.controlUi.allowedOrigins","value":["http://localhost:18789","http://127.0.0.1:18789"]}]'
docker compose up -d openclaw-gateway
```

Docker संदर्भ `.git` को बाहर रखता है। स्रोत पहचान को ऊपर दिखाए अनुसार बिल्ड आर्ग्युमेंट के रूप में पास करें, ताकि इमेज की परिचय स्क्रीन चेकआउट किया गया कमिट और एक बिल्ड टाइमस्टैम्प दिखाए। `scripts/docker/setup.sh` दोनों मानों को स्वचालित रूप से हल करके पास करता है।

<Note>
`docker compose` को रेपो रूट से चलाएँ। यदि आपने `OPENCLAW_EXTRA_MOUNTS` या `OPENCLAW_HOME_VOLUME` सक्षम किया है, तो सेटअप स्क्रिप्ट `docker-compose.extra.yml` लिखती है; इसे स्वयं बनाए गए किसी भी `docker-compose.override.yml` के बाद शामिल करें, उदाहरण के लिए `-f docker-compose.yml -f docker-compose.override.yml -f docker-compose.extra.yml`।
</Note>

### कंटेनर इमेज अपग्रेड करना

जब आप OpenClaw इमेज बदलते हैं, लेकिन वही माउंट की गई स्थिति/कॉन्फ़िगरेशन रखते हैं, तो नया Gateway तैयार होने से पहले स्टार्टअप-सुरक्षित अपग्रेड माइग्रेशन और Plugin अभिसरण चलाता है। नियमित इमेज अपग्रेड के लिए अलग से `openclaw doctor --fix` चलाने की आवश्यकता नहीं होनी चाहिए।

यदि स्टार्टअप उन सुधारों को सुरक्षित रूप से पूरा नहीं कर सकता, तो Gateway स्वस्थ स्थिति की सूचना देने के बजाय बंद हो जाता है। पुनः आरंभ नीति के साथ Docker, Podman या Kubernetes Gateway कंटेनर को बार-बार पुनः आरंभ होते दिखा सकते हैं। माउंट किया गया स्थिति वॉल्यूम बनाए रखें, फिर उसी इमेज को एक बार कंटेनर कमांड के रूप में `openclaw doctor --fix` के साथ चलाएँ और वही स्थिति/कॉन्फ़िगरेशन माउंट उपयोग करें जिनका Gateway उपयोग करता है:

```bash
docker run --rm -v <openclaw-state>:/home/node/.openclaw <image> openclaw doctor --fix
podman run --rm -v <openclaw-state>:/home/node/.openclaw <image> openclaw doctor --fix
```

doctor पूरा होने के बाद Gateway कंटेनर को उसके डिफ़ॉल्ट कमांड के साथ पुनः आरंभ करें। Kubernetes में वही कमांड उसी PVC से माउंट किए गए एकबारगी Job या डिबग पॉड में चलाएँ, फिर Deployment या StatefulSet पुनः आरंभ करें।

### पर्यावरण चर

`scripts/docker/setup.sh` द्वारा (और Gateway कंटेनर के लिए सीधे `docker-compose.yml` द्वारा) स्वीकार किए जाने वाले वैकल्पिक चर:

| चर                                             | उद्देश्य                                                                                                               |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_IMAGE`                                | स्थानीय रूप से बिल्ड करने के बजाय रिमोट इमेज का उपयोग करें                                                                    |
| `OPENCLAW_IMAGE_APT_PACKAGES`                   | बिल्ड के दौरान अतिरिक्त apt पैकेज इंस्टॉल करें (स्पेस से अलग किए गए)। लेगेसी उपनाम: `OPENCLAW_DOCKER_APT_PACKAGES`           |
| `OPENCLAW_IMAGE_PIP_PACKAGES`                   | बिल्ड के दौरान अतिरिक्त Python पैकेज इंस्टॉल करें (स्पेस से अलग किए गए)                                                      |
| `OPENCLAW_EXTENSIONS`                           | समर्थित चयनित Plugin को कंपाइल/पैकेज करें और उनकी रनटाइम निर्भरताएँ इंस्टॉल करें (कॉमा या स्पेस से अलग की गई ids) |
| `OPENCLAW_DOCKER_BUILD_NODE_OPTIONS`            | स्थानीय स्रोत-बिल्ड Node विकल्पों को ओवरराइड करें (डिफ़ॉल्ट `--max-old-space-size=8192`)                                |
| `OPENCLAW_DOCKER_BUILD_TSDOWN_MAX_OLD_SPACE_MB` | स्थानीय स्रोत-बिल्ड tsdown हीप को MB में ओवरराइड करें                                                                 |
| `OPENCLAW_DOCKER_BUILD_SKIP_DTS`                | केवल-रनटाइम स्थानीय इमेज बिल्ड के दौरान घोषणा आउटपुट छोड़ें (डिफ़ॉल्ट `1`)                                      |
| `OPENCLAW_INSTALL_BROWSER`                      | बिल्ड के समय Chromium + Xvfb को इमेज में शामिल करें                                                                 |
| `OPENCLAW_EXTRA_MOUNTS`                         | अतिरिक्त होस्ट बाइंड माउंट (कॉमा से अलग किए गए `source:target[:opts]`)                                                   |
| `OPENCLAW_HOME_VOLUME`                          | `/home/node` को नामित Docker वॉल्यूम में स्थायी रखें                                                                     |
| `OPENCLAW_SANDBOX`                              | सैंडबॉक्स बूटस्ट्रैप के लिए ऑप्ट इन करें (`1`, `true`, `yes`, `on`)                                                            |
| `OPENCLAW_SKIP_ONBOARDING`                      | इंटरैक्टिव ऑनबोर्डिंग चरण छोड़ें (`1`, `true`, `yes`, `on`)                                                   |
| `OPENCLAW_DOCKER_SOCKET`                        | Docker सॉकेट पथ को ओवरराइड करें                                                                                   |
| `OPENCLAW_DISABLE_BONJOUR`                      | Bonjour/mDNS विज्ञापन को चालू (`0`) या बंद (`1`) करने के लिए बाध्य करें; [Bonjour / mDNS](#bonjour--mdns) देखें                        |
| `OPENCLAW_DISABLE_BUNDLED_SOURCE_OVERLAYS`      | बंडल किए गए Plugin स्रोत बाइंड-माउंट ओवरले अक्षम करें                                                                 |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                   | OpenTelemetry निर्यात के लिए साझा OTLP/HTTP कलेक्टर एंडपॉइंट                                                      |
| `OTEL_EXPORTER_OTLP_*_ENDPOINT`                 | ट्रेस, मेट्रिक्स या लॉग के लिए सिग्नल-विशिष्ट OTLP एंडपॉइंट                                                       |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                   | OTLP प्रोटोकॉल ओवरराइड। आज केवल `http/protobuf` समर्थित है                                                   |
| `OTEL_SERVICE_NAME`                             | OpenTelemetry संसाधनों के लिए प्रयुक्त सेवा नाम                                                                     |
| `OTEL_SEMCONV_STABILITY_OPT_IN`                 | नवीनतम प्रायोगिक GenAI सिमेंटिक विशेषताओं के लिए ऑप्ट इन करें                                                           |
| `OPENCLAW_OTEL_PRELOADED`                       | जब कोई OpenTelemetry SDK पहले से लोड हो, तब दूसरा SDK शुरू करना छोड़ें                                                    |

आधिकारिक इमेज में Homebrew शामिल नहीं है। ऑनबोर्डिंग के दौरान OpenClaw, `brew` रहित Linux कंटेनर में केवल-brew Skills निर्भरता इंस्टॉलर छिपाता है; उन निर्भरताओं को कस्टम इमेज के माध्यम से उपलब्ध कराएँ या मैन्युअल रूप से इंस्टॉल करें। Debian-पैकेज्ड निर्भरताओं के लिए `OPENCLAW_IMAGE_APT_PACKAGES` और Python निर्भरताओं के लिए `OPENCLAW_IMAGE_PIP_PACKAGES` का उपयोग करें (यह बिल्ड समय पर `python3 -m pip install --break-system-packages` चलाता है, इसलिए संस्करण पिन करें और केवल विश्वसनीय इंडेक्स का उपयोग करें)।

यदि Docker `ResourceExhausted`, `cannot allocate memory` की सूचना देता है या `tsdown` के दौरान रुक जाता है, तो Docker बिल्डर की मेमोरी सीमा बढ़ाएँ या छोटे स्पष्ट हीप के साथ पुनः प्रयास करें:

```bash
OPENCLAW_DOCKER_BUILD_NODE_OPTIONS=--max-old-space-size=4096 OPENCLAW_DOCKER_BUILD_TSDOWN_MAX_OLD_SPACE_MB=4096
```

### चयनित Plugin वाली स्रोत से बनी इमेज

`OPENCLAW_EXTENSIONS` स्रोत चेकआउट से Plugin मैनिफ़ेस्ट आईडी चुनता है;
अलग होने पर मौजूदा स्रोत-डायरेक्टरी नाम भी स्वीकार किए जाते हैं। Docker
बिल्ड चयन को एक बार स्रोत डायरेक्टरियों में रिज़ॉल्व करता है, प्रोडक्शन
डिपेंडेंसी इंस्टॉल करता है और, जब कोई चयनित Plugin अलग से
`openclaw.build.bundledDist: false` के साथ प्रकाशित होता है, तो उसके रनटाइम को रूट बंडल किए गए
dist में कंपाइल करता है। केवल Docker की यह पैकेजिंग Plugin के npm या ClawHub
आर्टिफ़ैक्ट अनुबंध को नहीं बदलती। अज्ञात, अमान्य या अस्पष्ट आईडी के कारण इमेज बिल्ड विफल हो जाता है।
ज्ञात डिपेंडेंसी/केवल-स्रोत आईडी, कंपाइल की गई रूट dist प्रविष्टि पाए बिना
अपनी मौजूदा स्रोत और डिपेंडेंसी स्टेजिंग बनाए रखते हैं। एकीकृत बिल्ड प्रविष्टियों वाला
चयनित Plugin सफलतापूर्वक कंपाइल होना आवश्यक है; अचयनित बाहरी Plugin
स्रोत और रनटाइम आउटपुट हटा दिए जाते हैं।

उदाहरण के लिए, ये कमांड ClickClack, Slack और Microsoft Teams के लिए अलग-अलग,
मल्टी-आर्किटेक्चर वाले स्टैंडअलोन FakeCo Gateway इमेज बनाते हैं। ClawRouter
पहले से ही रूट OpenClaw रनटाइम का हिस्सा है, इसलिए ClickClack इमेज केवल
`clickclack` चुनती है। स्पष्ट रूप से खाली ब्राउज़र आर्ग्युमेंट डिफ़ॉल्ट इमेज को
Chromium से मुक्त रखता है:

```bash
SOURCE_SHA="$(git rev-parse HEAD)"
BUILD_TIMESTAMP="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
REGISTRY="registry.example.com/fakeco"

build_gateway_image() {
  gateway="$1"
  selected_plugin="$2"
  docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --build-arg "GIT_COMMIT=${SOURCE_SHA}" \
    --build-arg "OPENCLAW_BUILD_TIMESTAMP=${BUILD_TIMESTAMP}" \
    --build-arg "OPENCLAW_EXTENSIONS=${selected_plugin}" \
    --build-arg OPENCLAW_INSTALL_BROWSER= \
    --provenance=mode=max \
    --sbom=true \
    --tag "${REGISTRY}/openclaw-${gateway}:${SOURCE_SHA}" \
    --push \
    .
}

build_gateway_image clickclack clickclack
build_gateway_image slack slack
build_gateway_image teams msteams
```

एकल नेटिव स्थानीय बिल्ड के लिए `--platform linux/arm64 --load` या `--platform linux/amd64 --load` का उपयोग करें।
मल्टी-प्लेटफ़ॉर्म आउटपुट और संलग्न SBOM/प्रोवेनेंस के लिए ऐसी रजिस्ट्री या अन्य Buildx
आउटपुट आवश्यक है जो सत्यापन संलग्नक सुरक्षित रखे। पुश करने के बाद, मैनिफ़ेस्ट का निरीक्षण करें और
परिवर्तनीय स्रोत-SHA टैग के बजाय अपरिवर्तनीय डाइजेस्ट डिप्लॉय करें:

```bash
docker buildx imagetools inspect \
  "${REGISTRY}/openclaw-clickclack:${SOURCE_SHA}"
# डिप्लॉय करें: registry.example.com/fakeco/openclaw-clickclack@sha256:<manifest-digest>
```

ये इमेज स्टैंडअलोन OCI-आधारित Gateway और सामान्य Docker उपयोगकर्ताओं के लिए हैं।
Crabhelm-प्रबंधित Gateway इनका उपयोग नहीं करते: वह वितरण पथ एक अलग
x86_64 उपकरण आर्काइव बनाता है, जिसमें OpenClaw npm टारबॉल होता है और
Node, आर्काइव तथा मैनिफ़ेस्ट डाइजेस्ट पिन किए जाते हैं। उस उपकरण को
उसी लैंड किए गए OpenClaw स्रोत से अलग से बिल्ड करें।

पैकेज की गई इमेज के विरुद्ध बंडल किए गए Plugin स्रोत का परीक्षण करने के लिए, एक Plugin स्रोत डायरेक्टरी को उसके पैकेज किए गए स्रोत पथ पर माउंट करें, जैसे `OPENCLAW_EXTRA_MOUNTS=/path/to/fork/extensions/synology-chat:/app/extensions/synology-chat:ro`। यह उसी Plugin आईडी के मेल खाते कंपाइल किए गए `/app/dist/extensions/synology-chat` बंडल को ओवरराइड करता है।

### प्रेक्षणीयता

OpenTelemetry एक्सपोर्ट Gateway कंटेनर से आपके OTLP कलेक्टर की ओर आउटबाउंड होता है; इसके लिए किसी Docker पोर्ट को प्रकाशित करने की आवश्यकता नहीं है। स्थानीय रूप से बिल्ड की गई इमेज में बंडल किया गया एक्सपोर्टर शामिल करने के लिए:

```bash
export OPENCLAW_EXTENSIONS="diagnostics-otel"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4318"
export OTEL_SERVICE_NAME="openclaw-gateway"
./scripts/docker/setup.sh
```

आधिकारिक पूर्वनिर्मित इमेज में `diagnostics-otel` पहले से बंडल होता है; यदि आपने इसे हटा दिया है, तभी `clawhub:@openclaw/diagnostics-otel` स्वयं इंस्टॉल करें। एक्सपोर्ट सक्षम करने के लिए, कॉन्फ़िगरेशन में `diagnostics-otel` Plugin को अनुमति देकर सक्षम करें, फिर `diagnostics.otel.enabled=true` सेट करें ([OpenTelemetry एक्सपोर्ट](/hi/gateway/opentelemetry) में पूरा उदाहरण देखें)। कलेक्टर प्रमाणीकरण हेडर `diagnostics.otel.headers` के माध्यम से जाते हैं, Docker पर्यावरण वेरिएबल के माध्यम से नहीं।

Prometheus मेट्रिक्स पहले से प्रकाशित Gateway पोर्ट का पुनः उपयोग करते हैं। `clawhub:@openclaw/diagnostics-prometheus` इंस्टॉल करें, `diagnostics-prometheus` Plugin सक्षम करें, फिर स्क्रेप करें:

```text
http://<gateway-host>:18789/api/diagnostics/prometheus
```

यह रूट Gateway प्रमाणीकरण द्वारा सुरक्षित है; अलग सार्वजनिक `/metrics` पोर्ट या अप्रमाणित रिवर्स-प्रॉक्सी पथ उजागर न करें। [Prometheus मेट्रिक्स](/hi/gateway/prometheus) देखें।

### स्वास्थ्य जाँच

कंटेनर प्रोब एंडपॉइंट (प्रमाणीकरण आवश्यक नहीं):

```bash
curl -fsS http://127.0.0.1:18789/healthz   # सक्रियता
curl -fsS http://127.0.0.1:18789/readyz     # तत्परता
```

इमेज का अंतर्निहित `HEALTHCHECK`, `/healthz` को पिंग करता है; बार-बार विफलता होने पर कंटेनर को `unhealthy` चिह्नित किया जाता है, ताकि ऑर्केस्ट्रेटर उसे पुनः आरंभ या प्रतिस्थापित कर सकें।

प्रमाणित विस्तृत स्वास्थ्य स्नैपशॉट:

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### LAN बनाम लूपबैक

`scripts/docker/setup.sh` का डिफ़ॉल्ट `OPENCLAW_GATEWAY_BIND=lan` है, ताकि होस्ट पर `http://127.0.0.1:18789` Docker पोर्ट प्रकाशन के साथ काम करे।

- `lan` (डिफ़ॉल्ट): होस्ट ब्राउज़र और होस्ट CLI प्रकाशित Gateway पोर्ट तक पहुँच सकते हैं।
- `loopback`: केवल कंटेनर नेटवर्क नेमस्पेस के भीतर की प्रक्रियाएँ Gateway तक सीधे पहुँच सकती हैं।

<Note>
`gateway.bind` में बाइंड मोड मानों (`lan` / `loopback` / `custom` / `tailnet` / `auto`) का उपयोग करें, `0.0.0.0` या `127.0.0.1` जैसे होस्ट उपनामों का नहीं।
</Note>

### होस्ट के स्थानीय प्रदाता

कंटेनर के भीतर, `127.0.0.1` स्वयं कंटेनर है, होस्ट नहीं। होस्ट पर चल रहे प्रदाताओं के लिए `host.docker.internal` का उपयोग करें:

| प्रदाता  | होस्ट का डिफ़ॉल्ट URL         | Docker सेटअप URL                    |
| --------- | ------------------------ | ----------------------------------- |
| LM Studio | `http://127.0.0.1:1234`  | `http://host.docker.internal:1234`  |
| Ollama    | `http://127.0.0.1:11434` | `http://host.docker.internal:11434` |

बंडल किया गया सेटअप उन URL का उपयोग LM Studio/Ollama ऑनबोर्डिंग डिफ़ॉल्ट के रूप में करता है और `docker-compose.yml`, Linux Docker Engine पर `host.docker.internal` को होस्ट Gateway से मैप करता है (Docker Desktop macOS/Windows पर यही उपनाम प्रदान करता है)। होस्ट सेवाओं को ऐसे पते पर सुनना आवश्यक है जिस तक Docker पहुँच सके:

```bash
lms server start --port 1234 --bind 0.0.0.0
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

अपनी Compose फ़ाइल या `docker run` का उपयोग कर रहे हैं? वही मैपिंग स्वयं जोड़ें, जैसे `--add-host=host.docker.internal:host-gateway`।

### Docker में Claude CLI बैकएंड

आधिकारिक इमेज Claude Code को पहले से इंस्टॉल नहीं करती। कंटेनर के `node` उपयोगकर्ता के भीतर इंस्टॉल और लॉग इन करें, फिर उस कंटेनर होम को स्थायी बनाएँ ताकि इमेज अपग्रेड बाइनरी या प्रमाणीकरण स्थिति न मिटाएँ।

नई स्थापना के लिए, सेटअप चलाने से पहले स्थायी `/home/node` वॉल्यूम सक्षम करें:

```bash
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
export OPENCLAW_HOME_VOLUME="openclaw_home"
./scripts/docker/setup.sh
```

मौजूदा स्थापना के लिए, पहले स्टैक रोकें और वर्तमान `.env` मान पुनः लोड करें—सेटअप स्क्रिप्ट हमेशा वर्तमान शेल और डिफ़ॉल्ट से `.env` को दोबारा लिखती है, वह फ़ाइल को स्वयं नहीं पढ़ती:

```bash
set -a
. ./.env
set +a
export OPENCLAW_HOME_VOLUME="${OPENCLAW_HOME_VOLUME:-openclaw_home}"
./scripts/docker/setup.sh
```

यदि `.env` में ऐसे मान हैं जिन्हें आपका शेल स्रोत नहीं कर सकता, तो जिन पर आप निर्भर हैं उन्हें पहले मैन्युअल रूप से पुनः एक्सपोर्ट करें (`OPENCLAW_IMAGE`, पोर्ट, बाइंड मोड, कस्टम पथ, `OPENCLAW_EXTRA_MOUNTS`, सैंडबॉक्स, ऑनबोर्डिंग छोड़ना)। जनरेट किया गया ओवरले `openclaw-gateway` और `openclaw-cli` दोनों के लिए होम वॉल्यूम माउंट करता है; शेष कमांड उसी ओवरले के साथ चलाएँ (और यदि आप किसी का उपयोग करते हैं, तो पहले `docker-compose.override.yml`):

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint sh openclaw-cli -lc \
  'curl -fsSL https://claude.ai/install.sh | bash'
```

नेटिव इंस्टॉलर `claude` को `/home/node/.local/bin/claude` में लिखता है।
OpenClaw इमेज में `PATH` पर `/home/node/.local/bin` शामिल है, इसलिए बंडल किया गया
Anthropic Plugin इसे अडैप्टर कॉन्फ़िगरेशन ओवरराइड के बिना रिज़ॉल्व करता है।

उसी स्थायी होम से लॉग इन करके सत्यापित करें:

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint /home/node/.local/bin/claude openclaw-cli auth login
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint /home/node/.local/bin/claude openclaw-cli auth status --text
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli models auth login \
  --provider anthropic --method cli --set-default
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli models list --provider anthropic
```

फिर बंडल किए गए `claude-cli` बैकएंड का उपयोग करें:

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli agent \
  --agent main \
  --model claude-cli/claude-sonnet-4-6 \
  --message "Docker Claude CLI से नमस्ते कहें"
```

`OPENCLAW_HOME_VOLUME`, नेटिव स्थापना को `/home/node/.local/bin` और `/home/node/.local/share/claude` के अंतर्गत तथा Claude Code सेटिंग्स/प्रमाणीकरण को `/home/node/.claude` और `/home/node/.claude.json` के अंतर्गत स्थायी रखता है। केवल `/home/node/.openclaw` को स्थायी रखना पर्याप्त नहीं है; यदि आप होम वॉल्यूम के बजाय `OPENCLAW_EXTRA_MOUNTS` का उपयोग करते हैं, तो उन सभी Claude पथों को दोनों सेवाओं में माउंट करें।

<Note>
साझा प्रोडक्शन ऑटोमेशन या पूर्वानुमेय Anthropic बिलिंग के लिए, Anthropic API-कुंजी पथ को प्राथमिकता दें। Claude CLI का पुनः उपयोग Claude Code के इंस्टॉल किए गए संस्करण, अकाउंट लॉगिन, बिलिंग और अपडेट व्यवहार का अनुसरण करता है।
</Note>

### Bonjour / mDNS

Docker ब्रिज नेटवर्किंग सामान्यतः Bonjour/mDNS मल्टीकास्ट (`224.0.0.251:5353`) को विश्वसनीय रूप से अग्रेषित नहीं करती। जब `OPENCLAW_DISABLE_BONJOUR` सेट नहीं होता, तो बंडल किया गया Bonjour Plugin कंटेनर में चलने का पता लगाते ही LAN विज्ञापन स्वतः अक्षम कर देता है, ताकि ब्रिज द्वारा छोड़े गए मल्टीकास्ट को बार-बार पुनः प्रयास करते हुए वह क्रैश-लूप न करे। पहचान की परवाह किए बिना इसे बंद करने के लिए `OPENCLAW_DISABLE_BONJOUR=1` सेट करें या इसे चालू करने के लिए `0` सेट करें (केवल होस्ट नेटवर्किंग, macvlan या ऐसे किसी अन्य नेटवर्क पर जहाँ mDNS मल्टीकास्ट का काम करना ज्ञात हो)।

अन्यथा Docker होस्ट के लिए प्रकाशित Gateway URL, Tailscale या वाइड-एरिया DNS-SD का उपयोग करें। ध्यान देने योग्य बातों और समस्या निवारण के लिए [Bonjour खोज](/hi/gateway/bonjour) देखें।

### स्टोरेज और स्थायित्व

Docker Compose, `OPENCLAW_CONFIG_DIR` को `/home/node/.openclaw`, `OPENCLAW_WORKSPACE_DIR` को `/home/node/.openclaw/workspace` और `OPENCLAW_AUTH_PROFILE_SECRET_DIR` को `/home/node/.config/openclaw` पर बाइंड-माउंट करता है, इसलिए ये पथ कंटेनर प्रतिस्थापन के बाद भी बने रहते हैं। जब कोई वेरिएबल सेट नहीं होता, तो `docker-compose.yml`, `${HOME}` के अंतर्गत या यदि स्वयं `HOME` उपलब्ध नहीं है तो `/tmp` पर फ़ॉलबैक करता है, ताकि सामान्य परिवेशों में `docker compose up` कभी खाली-स्रोत वाला वॉल्यूम विनिर्देश न बनाए।

उस माउंट की गई कॉन्फ़िगरेशन डायरेक्टरी में ये होते हैं:

- व्यवहार कॉन्फ़िगरेशन के लिए `openclaw.json`
- संग्रहित प्रदाता OAuth/API-कुंजी प्रमाणीकरण के लिए `agents/<agentId>/agent/auth-profiles.json`
- `OPENCLAW_GATEWAY_TOKEN` जैसे पर्यावरण-समर्थित रनटाइम सीक्रेट के लिए `.env`

प्रमाणीकरण-प्रोफ़ाइल सीक्रेट डायरेक्टरी OAuth-समर्थित प्रमाणीकरण प्रोफ़ाइल टोकन सामग्री के लिए स्थानीय एन्क्रिप्शन कुंजी संग्रहित करती है। इसे अपने Docker होस्ट की स्थिति के साथ रखें, लेकिन `OPENCLAW_CONFIG_DIR` से अलग रखें।

इंस्टॉल किए गए डाउनलोड योग्य Plugin, पैकेज स्थिति को माउंट किए गए OpenClaw होम के अंतर्गत संग्रहित करते हैं, इसलिए इंस्टॉल रिकॉर्ड और पैकेज रूट कंटेनर प्रतिस्थापन के बाद भी बने रहते हैं; Gateway स्टार्टअप बंडल किए गए Plugin की डिपेंडेंसी ट्री दोबारा जनरेट नहीं करता।

VM के पूर्ण स्थायित्व विवरण के लिए, [Docker VM रनटाइम—क्या कहाँ स्थायी रहता है](/hi/install/docker-vm-runtime#what-persists-where) देखें।

**डिस्क वृद्धि के प्रमुख स्थान:** `media/`, प्रति-एजेंट SQLite डेटाबेस, पुराने सत्र JSONL ट्रांसक्रिप्ट, साझा SQLite स्थिति डेटाबेस, इंस्टॉल किए गए Plugin पैकेज रूट और `/tmp/openclaw/` के अंतर्गत रोलिंग फ़ाइल लॉग।

### शेल सहायक (वैकल्पिक)

रोज़मर्रा के छोटे कमांड के लिए, [ClawDock](/hi/install/clawdock) इंस्टॉल करें:

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

यदि आपने पुराने `scripts/shell-helpers/clawdock-helpers.sh` पथ से इंस्टॉल किया था, तो ऊपर दिया गया कमांड दोबारा चलाएँ ताकि आपका स्थानीय सहायक वर्तमान स्थान का अनुसरण करे। फिर `clawdock-start`, `clawdock-stop`, `clawdock-dashboard` आदि का उपयोग करें (पूरी सूची के लिए `clawdock-help` चलाएँ)।

<AccordionGroup>
  <Accordion title="Docker gateway के लिए एजेंट सैंडबॉक्स सक्षम करें">
    ```bash
    export OPENCLAW_SANDBOX=1
    ./scripts/docker/setup.sh
    ```

    कस्टम सॉकेट पथ (उदा. रूटलेस Docker):

    ```bash
    export OPENCLAW_SANDBOX=1
    export OPENCLAW_DOCKER_SOCKET=/run/user/1000/docker.sock
    ./scripts/docker/setup.sh
    ```

    स्क्रिप्ट सैंडबॉक्स की पूर्वापेक्षाएँ पूरी होने के बाद ही `docker.sock` को माउंट करती है। यदि सैंडबॉक्स सेटअप पूरा नहीं हो पाता, तो यह `agents.defaults.sandbox.mode` को `off` पर रीसेट कर देती है। जिन टर्न में OpenClaw सैंडबॉक्स सक्रिय होता है, उनमें Codex कोड मोड अक्षम रहता है ([सैंडबॉक्सिंग § Docker बैकएंड](/hi/gateway/sandboxing#docker-backend) देखें); होस्ट Docker सॉकेट को एजेंट सैंडबॉक्स कंटेनरों में कभी माउंट न करें।

  </Accordion>

  <Accordion title="ऑटोमेशन / CI (गैर-इंटरैक्टिव)">
    `-T` से Compose छद्म-TTY आवंटन अक्षम करें:

    ```bash
    docker compose run -T --rm openclaw-cli gateway probe
    docker compose run -T --rm openclaw-cli devices list --json
    ```

  </Accordion>

  <Accordion title="साझा-नेटवर्क सुरक्षा नोट">
    `openclaw-cli`, `network_mode: "service:openclaw-gateway"` का उपयोग करता है, ताकि CLI कमांड `127.0.0.1` के माध्यम से gateway तक पहुँच सकें। इसे साझा विश्वास सीमा मानें। Compose कॉन्फ़िगरेशन `openclaw-gateway` और `openclaw-cli` दोनों पर `NET_RAW`/`NET_ADMIN` को हटाता है और `no-new-privileges` को सक्षम करता है।
  </Accordion>

  <Accordion title="openclaw-cli में Docker Desktop DNS विफलताएँ">
    कुछ Docker Desktop सेटअप में `NET_RAW` हटाए जाने के बाद साझा-नेटवर्क `openclaw-cli` साइडकार से DNS लुकअप विफल हो जाते हैं, जो `openclaw plugins install` जैसे npm-समर्थित कमांड के दौरान `EAI_AGAIN` के रूप में दिखाई देता है। सामान्य संचालन के लिए डिफ़ॉल्ट सुदृढ़ Compose फ़ाइल बनाए रखें। नीचे दिया गया ओवरराइड केवल `openclaw-cli` कंटेनर के लिए डिफ़ॉल्ट क्षमताएँ पुनर्स्थापित करता है — इसका उपयोग केवल उस एकबारगी कमांड के लिए करें जिसे रजिस्ट्री एक्सेस चाहिए, अपने डिफ़ॉल्ट आह्वान के रूप में नहीं:

    ```bash
    printf '%s\n' \
      'services:' \
      '  openclaw-cli:' \
      '    cap_drop: !reset []' \
      > docker-compose.cli-no-dropped-caps.local.yml

    docker compose -f docker-compose.yml -f docker-compose.cli-no-dropped-caps.local.yml run --rm openclaw-cli plugins install <package>
    ```

    यदि आपने पहले ही लंबे समय तक चलने वाला `openclaw-cli` कंटेनर बनाया है, तो उसे उसी ओवरराइड के साथ दोबारा बनाएँ — `docker compose exec`/`docker exec` पहले से बने कंटेनर की Linux क्षमताएँ नहीं बदल सकते।

  </Accordion>

  <Accordion title="अनुमतियाँ और EACCES">
    इमेज `node` (uid 1000) के रूप में चलती है। यदि आपको `/home/node/.openclaw` पर अनुमति त्रुटियाँ दिखाई दें, तो सुनिश्चित करें कि आपके होस्ट बाइंड माउंट का स्वामी uid 1000 है:

    ```bash
    sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
    ```

    यही बेमेल स्थिति `blocked plugin candidate: suspicious ownership (... uid=1000, expected uid=0 or root)` और उसके बाद `plugin present but blocked` के रूप में दिखाई दे सकती है — प्रक्रिया uid और माउंट की गई Plugin डायरेक्टरी के स्वामी में असहमति है। डिफ़ॉल्ट uid 1000 के रूप में चलाना और बाइंड माउंट का स्वामित्व ठीक करना बेहतर है। यदि आप जानबूझकर OpenClaw को लंबे समय तक रूट के रूप में चलाते हैं, तभी `/path/to/openclaw-config/npm` का स्वामी `root:root` करें।

  </Accordion>

  <Accordion title="अधिक तेज़ पुनर्निर्माण">
    अपनी Dockerfile को इस तरह क्रमबद्ध करें कि डिपेंडेंसी लेयर कैश हो जाएँ और लॉकफ़ाइल बदलने के अलावा `pnpm install` को दोबारा चलाने से बचा जा सके:

    ```dockerfile
    FROM node:24-bookworm
    RUN curl -fsSL https://bun.sh/install | bash
    ENV PATH="/root/.bun/bin:${PATH}"
    RUN corepack enable
    WORKDIR /app
    COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
    COPY ui/package.json ./ui/package.json
    COPY scripts ./scripts
    RUN pnpm install --frozen-lockfile
    COPY . .
    RUN pnpm build
    RUN pnpm ui:install
    RUN pnpm ui:build
    ENV NODE_ENV=production
    CMD ["node","dist/index.js"]
    ```

  </Accordion>

  <Accordion title="उन्नत उपयोगकर्ता कंटेनर विकल्प">
    डिफ़ॉल्ट इमेज सुरक्षा को प्राथमिकता देती है और गैर-रूट `node` के रूप में चलती है। अधिक सुविधायुक्त कंटेनर के लिए:

    1. **`/home/node` को स्थायी रखें**: `export OPENCLAW_HOME_VOLUME="openclaw_home"`
    2. **सिस्टम डिपेंडेंसी इमेज में शामिल करें**: `export OPENCLAW_IMAGE_APT_PACKAGES="git curl jq"`
    3. **Python डिपेंडेंसी इमेज में शामिल करें**: `export OPENCLAW_IMAGE_PIP_PACKAGES="requests==2.32.5 humanize==4.14.0"`
    4. **Playwright Chromium इमेज में शामिल करें**: `export OPENCLAW_INSTALL_BROWSER=1`, या आधिकारिक `-browser` इमेज टैग का उपयोग करें
    5. **या Playwright ब्राउज़र को स्थायी वॉल्यूम में इंस्टॉल करें**:
       ```bash
       docker compose run --rm openclaw-cli \
         node /app/node_modules/playwright-core/cli.js install chromium
       ```
    6. **ब्राउज़र डाउनलोड स्थायी रखें**: `OPENCLAW_HOME_VOLUME` या `OPENCLAW_EXTRA_MOUNTS` का उपयोग करें। OpenClaw, Linux पर इमेज के Playwright-प्रबंधित Chromium का स्वतः पता लगा लेता है।

  </Accordion>

  <Accordion title="OpenAI Codex OAuth (हेडलेस Docker)">
    यदि आप विज़ार्ड में OpenAI Codex OAuth चुनते हैं, तो यह ब्राउज़र URL खोलता है। Docker या हेडलेस सेटअप में, उस पूर्ण रीडायरेक्ट URL को कॉपी करें जिस पर आप पहुँचते हैं और प्रमाणीकरण पूरा करने के लिए उसे वापस विज़ार्ड में पेस्ट करें।
  </Accordion>

  <Accordion title="बेस इमेज मेटाडेटा">
    रनटाइम इमेज `node:24-bookworm-slim` का उपयोग करती है और `tini` को PID 1 के रूप में चलाती है, ताकि लंबे समय तक चलने वाले कंटेनरों में ज़ॉम्बी प्रक्रियाएँ हटाई जाएँ और सिग्नल सही ढंग से संभाले जाएँ। यह `org.opencontainers.image.base.name` और `org.opencontainers.image.source` सहित OCI बेस-इमेज एनोटेशन प्रकाशित करती है। Dependabot पिन किए गए Node बेस डाइजेस्ट को रीफ़्रेश करता है; रिलीज़ बिल्ड अलग डिस्ट्रो अपग्रेड लेयर नहीं चलाते। [OCI इमेज एनोटेशन](https://github.com/opencontainers/image-spec/blob/main/annotations.md) देखें।
  </Accordion>
</AccordionGroup>

### VPS पर चला रहे हैं?

बाइनरी को इमेज में शामिल करने, स्थायित्व और अपडेट सहित साझा VM परिनियोजन चरणों के लिए [Hetzner (Docker VPS)](/hi/install/hetzner) और [Docker VM रनटाइम](/hi/install/docker-vm-runtime) देखें।

## एजेंट सैंडबॉक्स

जब Docker बैकएंड के साथ `agents.defaults.sandbox` सक्षम होता है, तब gateway एजेंट टूल निष्पादन (शेल, फ़ाइल पढ़ना/लिखना आदि) को अलग-अलग Docker कंटेनरों के भीतर चलाता है, जबकि gateway स्वयं होस्ट पर रहता है — इससे पूरे gateway को कंटेनर में डाले बिना अविश्वसनीय या बहु-किरायेदार एजेंट सत्रों के चारों ओर एक कठोर सुरक्षा दीवार बनती है।

सैंडबॉक्स का दायरा प्रति-एजेंट (डिफ़ॉल्ट), प्रति-सत्र या साझा हो सकता है; प्रत्येक दायरे को `/workspace` पर माउंट किया गया अपना कार्यक्षेत्र मिलता है। आप टूल की अनुमति/अस्वीकृति नीतियाँ, नेटवर्क अलगाव, संसाधन सीमाएँ और ब्राउज़र कंटेनर भी कॉन्फ़िगर कर सकते हैं।

पूर्ण कॉन्फ़िगरेशन, इमेज, सुरक्षा नोट और बहु-एजेंट प्रोफ़ाइल के लिए:

- [सैंडबॉक्सिंग](/hi/gateway/sandboxing) -- संपूर्ण सैंडबॉक्स संदर्भ
- [OpenShell](/hi/gateway/openshell) -- सैंडबॉक्स कंटेनरों के लिए इंटरैक्टिव शेल एक्सेस
- [बहु-एजेंट सैंडबॉक्स और टूल](/hi/tools/multi-agent-sandbox-tools) -- प्रति-एजेंट ओवरराइड

### शीघ्र सक्षम करें

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // बंद | गैर-मुख्य | सभी
        scope: "agent", // सत्र | एजेंट | साझा
      },
    },
  },
}
```

डिफ़ॉल्ट सैंडबॉक्स इमेज बनाएँ (स्रोत चेकआउट से):

```bash
scripts/sandbox-setup.sh
```

स्रोत चेकआउट के बिना npm इंस्टॉल के लिए, इनलाइन `docker build` कमांड हेतु [सैंडबॉक्सिंग § इमेज और सेटअप](/hi/gateway/sandboxing#images-and-setup) देखें।

## समस्या निवारण

<AccordionGroup>
  <Accordion title="इमेज अनुपलब्ध है या सैंडबॉक्स कंटेनर शुरू नहीं हो रहा">
    सैंडबॉक्स इमेज को [`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh) (स्रोत चेकआउट) या [सैंडबॉक्सिंग § इमेज और सेटअप](/hi/gateway/sandboxing#images-and-setup) के इनलाइन `docker build` कमांड (npm इंस्टॉल) से बनाएँ, या `agents.defaults.sandbox.docker.image` को अपनी कस्टम इमेज पर सेट करें। कंटेनर माँग पर प्रत्येक सत्र के लिए स्वतः बनाए जाते हैं।
  </Accordion>

  <Accordion title="सैंडबॉक्स में अनुमति त्रुटियाँ">
    `docker.user` को ऐसे UID:GID पर सेट करें जो आपके माउंट किए गए कार्यक्षेत्र के स्वामित्व से मेल खाता हो, या कार्यक्षेत्र फ़ोल्डर का स्वामी बदलें।
  </Accordion>

  <Accordion title="सैंडबॉक्स में कस्टम टूल नहीं मिले">
    OpenClaw, `sh -lc` (लॉगिन शेल) के साथ कमांड चलाता है, जो `/etc/profile` को स्रोत करता है और PATH को रीसेट कर सकता है। अपने कस्टम टूल पथों को आगे जोड़ने के लिए `docker.env.PATH` सेट करें, या अपनी Dockerfile में `/etc/profile.d/` के अंतर्गत एक स्क्रिप्ट जोड़ें।
  </Accordion>

  <Accordion title="इमेज बिल्ड के दौरान OOM के कारण प्रक्रिया बंद हुई (निकास 137)">
    VM को कम-से-कम 2 GB RAM चाहिए। बड़ी मशीन श्रेणी का उपयोग करें और फिर प्रयास करें।
  </Accordion>

  <Accordion title="Control UI में अनधिकृत या पेयरिंग आवश्यक">
    नया डैशबोर्ड लिंक प्राप्त करें और ब्राउज़र डिवाइस को स्वीकृत करें:

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    अधिक जानकारी: [डैशबोर्ड](/hi/web/dashboard), [डिवाइस](/hi/cli/devices)।

  </Accordion>

  <Accordion title="Gateway लक्ष्य ws://172.x.x.x दिखाता है या Docker CLI से पेयरिंग त्रुटियाँ आती हैं">
    Gateway मोड और बाइंड रीसेट करें:

    ```bash
    docker compose run --rm openclaw-cli config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"}]'
    docker compose run --rm openclaw-cli devices list --url ws://127.0.0.1:18789
    ```

  </Accordion>
</AccordionGroup>

## संबंधित

- [इंस्टॉलेशन अवलोकन](/hi/install) — इंस्टॉलेशन के सभी तरीके
- [Podman](/hi/install/podman) — Docker का Podman विकल्प
- [ClawDock](/hi/install/clawdock) — Docker Compose सामुदायिक सेटअप
- [अपडेट करना](/hi/install/updating) — OpenClaw को अद्यतित रखना
- [कॉन्फ़िगरेशन](/hi/gateway/configuration) — इंस्टॉलेशन के बाद gateway कॉन्फ़िगरेशन
