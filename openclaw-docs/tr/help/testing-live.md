---
read_when:
    - Canlı model matrisi / CLI arka ucu / ACP / medya sağlayıcısı duman testlerini çalıştırma
    - Canlı test kimlik bilgisi çözümlemesinde hata ayıklama
    - Sağlayıcıya özel yeni bir canlı test ekleme
sidebarTitle: Live tests
summary: 'Canlı (ağa erişen) testler: model matrisi, CLI arka uçları, ACP, medya sağlayıcıları, kimlik bilgileri'
title: 'Test: canlı paketler'
x-i18n:
    generated_at: "2026-07-26T23:23:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea8279e734e3aa09dd1fa184806c925e0404edfa9acf0f682f73a4955ed90b8b
    source_path: help/testing-live.md
    workflow: 16
---

Hızlı başlangıç, QA çalıştırıcıları, birim/entegrasyon paketleri ve Docker akışları için
[Testler](/tr/help/testing) sayfasına bakın. Bu sayfa **canlı** (ağa erişen) testleri kapsar:
model matrisi, CLI arka uçları, ACP, medya sağlayıcıları ve kimlik bilgisi yönetimi.

## Canlı testler ile gerçek gateway'iniz arasındaki fark

Canlı paketler ve geçici duman testleri, hâlihazırda gerçek trafiğe hizmet veren
bir gateway'i (size veya başka bir operatöre ait) asla etkilememelidir:

- Kendi gateway'inizi kullanın: işlem içi gateway'i (aşağıdaki Katman 2) kullanın veya
  yalıtılmış bir durum dizini (`OPENCLAW_STATE_DIR=<scratch>`) ve
  boş bir bağlantı noktasıyla bir geliştirme örneği başlatın. Gerçek bir gateway
  varsayılan gateway bağlantı noktasında (18789) çalışırken bu bağlantı noktasına bağlanmayın.
- Bu oturumda başlatmadığınız bir hizmete `openclaw gateway stop`/`restart` (veya `launchctl`/`systemctl`/tmux
  eşdeğerlerini) uygulamayın; bu, operatörün canlı örneğidir. Önce açık onay alın.
- Gerçekçi verilere mi ihtiyacınız var? Canlı durumu/DB'yi geliştirme durum dizininize kopyalayın ve
  kopyaya karşı test edin. Canlı bir gateway'in durumunda yerinde geçişler yapmak da
  açık onay gerektirir.

## Canlı: yerel duman testi komutları

Geçici canlı kontrollerden önce gerekli sağlayıcı anahtarını işlem ortamına
aktarın.

Güvenli medya duman testi:

```bash
pnpm openclaw infer tts convert --local --json \
  --text "OpenClaw canlı duman testi." \
  --output /tmp/openclaw-live-smoke.mp3
```

Güvenli sesli arama hazırlık duman testi:

```bash
pnpm openclaw voicecall setup --json
pnpm openclaw voicecall smoke --to "+15555550123"
```

`--yes` da mevcut olmadığı sürece `voicecall smoke` bir deneme çalıştırmasıdır; `--yes` öğesini yalnızca
gerçek bir arama yapmak istediğinizde kullanın. Twilio, Telnyx ve Plivo için
başarılı bir hazırlık kontrolü herkese açık bir webhook URL'si gerektirir; bu
sağlayıcılar erişemediği için yerel/özel geri döngü URL'leri reddedilir.

## Canlı: Android Node yetenek taraması

- Test: `src/gateway/android-node.capabilities.live.test.ts`
- Betik: `pnpm android:test:integration`
- Amaç: bağlı bir Android Node tarafından **şu anda duyurulan her komutu** çağırmak ve komut sözleşmesi davranışını doğrulamak.
- Kapsam:
  - Ön koşullu/manuel kurulum (paket uygulamayı kurmaz/çalıştırmaz/eşleştirmez).
  - Seçilen Android Node için komut bazında gateway `node.invoke` doğrulaması.
- Gerekli ön kurulum:
  - Android uygulaması gateway'e önceden bağlı ve eşleştirilmiş olmalıdır.
  - Uygulama ön planda tutulmalıdır.
  - Başarılı olmasını beklediğiniz yetenekler için izinler/yakalama onayı verilmelidir.
- İsteğe bağlı hedef geçersiz kılmaları:
  - `OPENCLAW_ANDROID_NODE_ID` veya `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Eksiksiz Android kurulum ayrıntıları: [Android Uygulaması](/tr/platforms/android)

## Canlı: model duman testi (profil anahtarları)

Canlı model testleri, hataları yalıtmak için iki katmana ayrılır:

- "Doğrudan model", sağlayıcının/modelin verilen anahtarla yanıt verip veremediğini gösterir.
- "Gateway duman testi", tam gateway+ajan işlem hattının söz konusu model için çalışıp çalışmadığını gösterir (oturumlar, geçmiş, araçlar, korumalı alan politikası vb.).

Aşağıdaki seçilmiş model listeleri `src/agents/live-model-filter.ts` içinde bulunur ve
zamanla değişir; doğruluk kaynağı olarak bu sayfayı değil, oradaki dizileri
kabul edin.

MiniMax M3, varsayılan sağlayıcı/model referansı olarak `minimax/MiniMax-M3` kullanır.

### Katman 1: Doğrudan model tamamlama (gateway yok)

- Test: `src/agents/models.profiles.live.test.ts`
- Amaç:
  - Keşfedilen modelleri listelemek
  - Kimlik bilgilerine sahip olduğunuz modelleri seçmek için `getApiKeyForModel` kullanmak
  - Her model için küçük bir tamamlama çalıştırmak (ve gerektiğinde hedefli regresyonlar)
- Etkinleştirme:
  - `pnpm test:live` (veya Vitest doğrudan çağrılıyorsa `OPENCLAW_LIVE_TEST=1`)
  - Bu paketi gerçekten çalıştırmak için `OPENCLAW_LIVE_MODELS=modern`, `small` veya `all` (`modern` için diğer ad) ayarlayın; aksi takdirde atlanır, böylece `pnpm test:live` tek başına gateway duman testine odaklanmayı sürdürür.
- Model seçimi:
  - `OPENCLAW_LIVE_MODELS=modern`, seçilmiş yüksek sinyalli öncelik listesini çalıştırır (bkz. [Canlı: model matrisi](#live-model-matrix-what-we-cover))
  - `OPENCLAW_LIVE_MODELS=small`, seçilmiş küçük model öncelik listesini çalıştırır
  - `OPENCLAW_LIVE_MODELS=all`, `modern` için diğer addır
  - veya `OPENCLAW_LIVE_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,..."` (virgülle ayrılmış izin listesi)
  - Yerel Ollama küçük model çalıştırmaları varsayılan olarak `http://127.0.0.1:11434` kullanır; `OPENCLAW_LIVE_OLLAMA_BASE_URL` öğesini yalnızca LAN, özel veya Ollama Cloud uç noktaları için ayarlayın.
  - Modern/tüm ve küçük taramalar, varsayılan üst sınır olarak kendi seçilmiş liste uzunluklarını kullanır; kapsamlı bir seçili profil taraması için `OPENCLAW_LIVE_MAX_MODELS=0`, daha küçük bir üst sınır için pozitif bir sayı ayarlayın.
  - Kapsamlı taramalar, doğrudan model testinin tamamının zaman aşımı için `OPENCLAW_LIVE_TEST_TIMEOUT_MS` kullanır. Varsayılan: 60 dakika.
  - Doğrudan model yoklamaları varsayılan olarak 20 yönlü paralellikle çalışır; geçersiz kılmak için `OPENCLAW_LIVE_MODEL_CONCURRENCY` ayarlayın.
- Sağlayıcı seçimi:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (virgülle ayrılmış izin listesi)
- Anahtarların kaynağı:
  - Varsayılan olarak: profil deposu ve ortam geri dönüşleri
  - Yalnızca **profil deposunu** zorunlu kılmak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` ayarlayın
- Bunun varlık nedeni:
  - "Sağlayıcı API'si bozuk / anahtar geçersiz" durumunu "gateway ajan işlem hattı bozuk" durumundan ayırır
  - Küçük, yalıtılmış regresyonlar içerir (örnek: OpenAI Responses/Codex Responses akıl yürütme yeniden oynatma + araç çağrısı akışları)

### Katman 2: Gateway + geliştirme ajanı duman testi ("@openclaw" gerçekte ne yapar)

- Test: `src/gateway/gateway-models.profiles.live.test.ts`
- Amaç:
  - İşlem içi bir gateway başlatmak
  - Bir `agent:dev:*` oturumu oluşturmak/yamamak (çalıştırma başına model geçersiz kılması)
  - Anahtarı bulunan modeller üzerinde yineleme yapmak ve şunları doğrulamak:
    - "anlamlı" yanıt (araç yok)
    - gerçek bir araç çağrısı çalışır (okuma yoklaması)
    - isteğe bağlı ek araç yoklamaları (çalıştırma+okuma yoklaması)
    - OpenAI regresyon yolları (yalnızca araç çağrısı -> takip) çalışmayı sürdürür
- Yoklama ayrıntıları (hataları hızla açıklayabilmeniz için):
  - `read` yoklaması: test, çalışma alanına nonce içeren bir dosya yazar ve ajandan bunu `read` ile okuyup nonce değerini geri yankılamasını ister.
  - `exec+read` yoklaması: test, ajandan geçici bir dosyaya nonce değerini `exec` ile yazmasını, ardından bunu `read` ile geri okumasını ister.
  - görüntü yoklaması: test, oluşturulmuş bir PNG (kedi + rastgele kod) ekler ve modelin `cat <CODE>` döndürmesini bekler.
  - Uygulama referansı: `src/gateway/gateway-models.profiles.live.test.ts` ve `test/helpers/live-image-probe.ts`.
- Etkinleştirme:
  - `pnpm test:live` (veya Vitest doğrudan çağrılıyorsa `OPENCLAW_LIVE_TEST=1`)
- Model seçimi:
  - Varsayılan: seçilmiş yüksek sinyalli (`modern`) öncelik listesi
  - `OPENCLAW_LIVE_GATEWAY_MODELS=small`, seçilmiş küçük model listesini tam gateway+ajan işlem hattından geçirir
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all`, `modern` için diğer addır
  - Ya da kapsamı daraltmak için `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (veya virgülle ayrılmış liste) ayarlayın
  - Modern/tüm ve küçük gateway taramaları, varsayılan üst sınır olarak kendi seçilmiş liste uzunluklarını kullanır; kapsamlı bir seçili tarama için `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0`, daha küçük bir üst sınır için pozitif bir sayı ayarlayın.
- Sağlayıcı seçimi ("her şey OpenRouter" yaklaşımından kaçının):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (virgülle ayrılmış izin listesi)
- Bu canlı testte araç + görüntü yoklamaları her zaman açıktır:
  - `read` yoklaması + `exec+read` yoklaması (araç stres testi)
  - model görüntü girişi desteği bildirdiğinde görüntü yoklaması çalışır
  - Akış (üst düzey):
    - Test, "CAT" + rastgele kod içeren küçük bir PNG oluşturur (`test/helpers/live-image-probe.ts`)
    - Bunu `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]` üzerinden gönderir
    - Gateway, ekleri `images[]` biçimine ayrıştırır (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Gömülü ajan, çok modlu bir kullanıcı mesajını modele iletir
    - Doğrulama: yanıt `cat` + kodu içerir (OCR toleransı: küçük hatalara izin verilir)

<Tip>
Makinenizde neleri test edebileceğinizi (ve tam `provider/model` kimliklerini) görmek için şunu çalıştırın:

```bash
openclaw models list
openclaw models list --json
```

</Tip>

## Canlı: CLI arka ucu duman testi (Claude, Gemini veya diğer yerel CLI'lar)

- Test: `src/gateway/gateway-cli-backend.live.test.ts`
- Amaç: varsayılan yapılandırmanıza dokunmadan yerel bir CLI arka ucu kullanarak Gateway + ajan işlem hattını doğrulamak.
- Arka uca özgü duman testi varsayılanları, sahibi olan Plugin'in `cli-backend.ts` tanımıyla birlikte bulunur.
- Etkinleştirme:
  - `pnpm test:live` (veya Vitest doğrudan çağrılıyorsa `OPENCLAW_LIVE_TEST=1`)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Varsayılanlar:
  - Varsayılan sağlayıcı/model: `claude-cli/claude-sonnet-4-6`
  - Komut/bağımsız değişken/görüntü davranışı, sahibi olan CLI arka uç Plugin meta verilerinden gelir.
- Geçersiz kılmalar (isteğe bağlı):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/claude"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["-p","--output-format","json"]'`
  - Gerçek bir görüntü eki göndermek için `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` (yollar isteme eklenir). Docker tariflerinde varsayılan olarak kapalıdır.
  - İsteme eklemek yerine görüntü dosyası yollarını CLI bağımsız değişkenleri olarak iletmek için `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`.
  - `IMAGE_ARG` ayarlandığında görüntü bağımsız değişkenlerinin nasıl iletileceğini denetlemek için `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (veya `"list"`).
  - İkinci bir tur göndermek ve devam ettirme akışını doğrulamak için `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`.
  - Seçilen model bir geçiş hedefini desteklediğinde Claude Sonnet -> Opus aynı oturum sürekliliği yoklamasını etkinleştirmek için `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=1`. Docker tarifleri dâhil varsayılan olarak kapalıdır.
  - MCP/araç geri döngü yoklamasını etkinleştirmek için `OPENCLAW_LIVE_CLI_BACKEND_MCP_PROBE=1`. Docker tariflerinde varsayılan olarak kapalıdır.

Örnek:

```bash
  OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

Hızlı Gemini MCP yapılandırma duman testi:

```bash
OPENCLAW_LIVE_TEST=1 \
  pnpm test:live src/agents/cli-runner/bundle-mcp.gemini.live.test.ts
```

Bu, Gemini'dan yanıt oluşturmasını istemez. OpenClaw'ın Gemini'a verdiği sistem
ayarlarının aynısını yazar, ardından kaydedilmiş bir `transport: "streamable-http"` sunucusunun
Gemini'ın HTTP MCP biçimine normalleştirildiğini ve yerel bir akış destekli HTTP MCP
sunucusuna bağlanabildiğini kanıtlamak için `gemini --debug mcp list` çalıştırır.

Docker tarifi:

```bash
pnpm test:docker:live-cli-backend
```

Tek sağlayıcılı Docker tarifleri:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:gemini
```

Notlar:

- Docker çalıştırıcısı `scripts/test-live-cli-backend-docker.sh` konumunda bulunur.
- Depo Docker imajı içindeki canlı CLI arka ucu smoke testini root olmayan `node` kullanıcısı olarak çalıştırır.
- CLI smoke testi meta verilerini sahibi olan pluginden çözümler, ardından eşleşen Linux CLI paketini (`@anthropic-ai/claude-code` veya `@google/gemini-cli`) `OPENCLAW_DOCKER_CLI_TOOLS_DIR` konumundaki önbelleğe alınmış, yazılabilir bir ön eke kurar (varsayılan: `~/.cache/openclaw/docker-cli-tools`).
- `codex-cli` artık paketle birlikte gelen bir CLI arka ucu değildir; bunun yerine Codex app-server çalışma zamanı ile `openai/*` kullanın (bkz. [Canlı: Codex app-server test düzeneği smoke testi](#live-codex-app-server-harness-smoke)).
- `pnpm test:docker:live-cli-backend:claude-subscription`, `claude setup-token` konumundaki `~/.claude/.credentials.json` ile `claudeAiOauth.subscriptionType` ya da `CLAUDE_CODE_OAUTH_TOKEN` üzerinden taşınabilir Claude Code abonelik OAuth'ı gerektirir. Önce Docker'da doğrudan `claude -p` doğrulaması yapar, ardından Anthropic API anahtarı ortam değişkenlerini korumadan iki Gateway CLI arka ucu turu çalıştırır. Bu abonelik hattı, oturum açılmış aboneliğin kullanım limitlerini tükettiği ve Anthropic, Claude Agent SDK / `claude -p` faturalandırma ve hız sınırı davranışını bir OpenClaw sürümü olmadan değiştirebildiği için Claude MCP/araç ve görüntü yoklamalarını varsayılan olarak devre dışı bırakır.
- Claude ve Gemini, yukarıdaki bayraklar üzerinden aynı yoklama kümesini (metin turu, görüntü sınıflandırması, MCP `cron` araç çağrısı, model değiştirme sürekliliği) destekler; ancak bu yoklamaların hiçbiri varsayılan olarak çalışmaz — gerektiğinde her birini ilgili bayrakla etkinleştirin.

## Canlı: APNs HTTP/2 proxy erişilebilirliği

- Test: `src/infra/push-apns-http2.live.test.ts`
- Amaç: yerel bir HTTP CONNECT proxy üzerinden Apple'ın korumalı alan APNs uç noktasına tünel açmak, APNs HTTP/2 doğrulama isteğini göndermek ve Apple'ın gerçek `403 InvalidProviderToken` yanıtının proxy yolu üzerinden geri geldiğini doğrulamak.
- Etkinleştirme:
  - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_APNS_REACHABILITY=1 pnpm test:live src/infra/push-apns-http2.live.test.ts`
- İsteğe bağlı zaman aşımı:
  - `OPENCLAW_LIVE_APNS_TIMEOUT_MS=30000`

## Canlı: ACP bağlama smoke testi (`/acp spawn ... --bind here`)

- Test: `src/gateway/gateway-acp-bind.live.test.ts`
- Amaç: canlı bir ACP aracısı ile gerçek ACP konuşma bağlama akışını doğrulamak:
  - `/acp spawn <agent> --bind here` gönderme
  - yapay bir mesaj kanalı konuşmasını yerinde bağlama
  - aynı konuşmada normal bir takip iletisi gönderme
  - takip iletisinin bağlı ACP oturumu dökümüne ulaştığını doğrulama
- Etkinleştirme:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Varsayılanlar:
  - Docker'daki ACP aracıları: `claude,codex,gemini`
  - Doğrudan `pnpm test:live ...` için ACP aracısı: `claude`
  - Yapay kanal: Slack DM tarzı konuşma bağlamı
  - ACP arka ucu: `acpx`
- Geçersiz kılmalar:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=droid`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=opencode`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
  - `OPENCLAW_LIVE_ACP_BIND_CODEX_MODEL=gpt-5.6-luna`
  - `OPENCLAW_LIVE_ACP_BIND_OPENCODE_MODEL=opencode/kimi-k2.6`
  - Görüntü yoklamasını zorla etkinleştirmek için `OPENCLAW_LIVE_ACP_BIND_IMAGE_PROBE=1` (veya `on`/`true`/`yes`); diğer tüm değerler yoklamayı zorla devre dışı bırakır. `opencode` dışındaki her aracı için varsayılan olarak çalışır.
  - `OPENCLAW_LIVE_ACP_BIND_REQUIRE_CRON=1`
  - `OPENCLAW_LIVE_ACP_BIND_PARENT_MODEL=openai/gpt-5.6-luna`
- Notlar:
  - Bu hat, testlerin harici teslimat yapıyormuş gibi davranmadan mesaj kanalı bağlamı ekleyebilmesi için yalnızca yöneticilere açık yapay kaynak rota alanlarıyla Gateway `chat.send` yüzeyini kullanır.
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` ayarlanmamışsa test, seçilen ACP test düzeneği aracısı için yerleşik `acpx` plugininin yerleşik aracı kayıt defterini kullanır.
  - Harici ACP test düzenekleri, bağlama/görüntü doğrulaması geçtikten sonra MCP çağrılarını iptal edebildiği için bağlı oturum Cron MCP oluşturma işlemi varsayılan olarak en iyi çaba esasına dayanır; bu bağlama sonrası Cron yoklamasını katı hale getirmek için `OPENCLAW_LIVE_ACP_BIND_REQUIRE_CRON=1` ayarlayın.

Örnek:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Docker tarifi:

```bash
pnpm test:docker:live-acp-bind
```

Tek aracılı Docker tarifleri:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:droid
pnpm test:docker:live-acp-bind:gemini
pnpm test:docker:live-acp-bind:opencode
```

Docker notları:

- Docker çalıştırıcısı `scripts/test-live-acp-bind-docker.sh` konumunda bulunur.
- Varsayılan olarak ACP bağlama smoke testini toplu canlı CLI aracıları üzerinde sırasıyla çalıştırır: `claude`, `codex`, ardından `gemini`.
- Matrisi daraltmak için `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=droid`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` veya `OPENCLAW_LIVE_ACP_BIND_AGENTS=opencode` kullanın.
- Eşleşen CLI kimlik doğrulama materyalini kapsayıcıya hazırlar, ardından eksikse istenen canlı CLI'ı (`@anthropic-ai/claude-code`, `@openai/codex`, `https://app.factory.ai/cli` üzerinden Factory Droid, `@google/gemini-cli` veya `opencode-ai`) kurar. ACP arka ucunun kendisi, resmî `acpx` pluginindeki yerleşik `acpx/runtime` paketidir.
- Droid Docker varyantı ayarlar için `~/.factory` dosyasını hazırlar, `FACTORY_API_KEY` değerini iletir ve yerel Factory OAuth/anahtarlık kimlik doğrulaması kapsayıcıya taşınabilir olmadığından bu API anahtarını zorunlu kılar. ACPX'in yerleşik `droid exec --output-format acp` kayıt defteri girdisini kullanır.
- OpenCode Docker varyantı, katı bir tek aracılı regresyon hattıdır. `OPENCLAW_LIVE_ACP_BIND_OPENCODE_MODEL` değerinden (varsayılan `opencode/kimi-k2.6`) geçici bir `OPENCODE_CONFIG_CONTENT` varsayılan modeli yazar.
- Doğrudan `acpx` CLI çağrıları yalnızca Gateway dışındaki davranışları karşılaştırmaya yönelik manuel/geçici çözüm yoludur. Docker ACP bağlama smoke testi, OpenClaw'ın yerleşik `acpx` çalışma zamanı arka ucunu çalıştırır.

## Canlı: Codex app-server test düzeneği smoke testi

- Amaç: pluginin sahip olduğu Codex test düzeneğini normal Gateway
  `agent` yöntemi üzerinden doğrulamak:
  - paketle birlikte gelen `codex` pluginini yükleme
  - `/model <ref> --runtime codex` üzerinden bir OpenAI modeli seçme
  - istenen düşünme düzeyiyle ilk Gateway aracı turunu gönderme
  - aynı OpenClaw oturumuna ikinci bir tur gönderme ve app-server
    iş parçacığının devam edebildiğini doğrulama
  - `/codex status` ve `/codex models` işlemlerini aynı Gateway komut
    yolu üzerinden çalıştırma
  - isteğe bağlı olarak Guardian tarafından incelenen, yükseltilmiş iki kabuk yoklaması çalıştırma: onaylanması gereken zararsız
    bir komut ve aracının geri sorması için reddedilmesi gereken sahte gizli bilgi
    yüklemesi
- Test: `src/gateway/gateway-codex-harness.live.test.ts`
- Etkinleştirme: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- Test düzeneği temel modeli: `openai/gpt-5.6-luna`
- Yeni OpenAI API anahtarı seçimi varsayılanı: `openai/gpt-5.6`
- Varsayılan düşünme: `low`
- Model geçersiz kılması: `OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/<model>`
- Düşünme geçersiz kılması: `OPENCLAW_LIVE_CODEX_HARNESS_THINKING=<level>`
- Varsayılan olmayan model eforu doğrulaması:
  `OPENCLAW_LIVE_CODEX_HARNESS_EXPECTED_EFFORT=<level>`
- Matris geçersiz kılması: `OPENCLAW_LIVE_CODEX_HARNESS_TARGETS=<model>=<thinking>,...`
- Kimlik doğrulama modu: `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=codex-auth` (varsayılan), kopyalanmış
  Codex oturumunu kullanır; `api-key`, Codex app-server üzerinden `OPENAI_API_KEY` kullanır.
- İsteğe bağlı görüntü yoklaması: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- İsteğe bağlı MCP/araç yoklaması: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- İsteğe bağlı Guardian yoklaması: `OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=1`
- İsteğe bağlı devam ettirme stres testi: `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1`,
  dört geçmiş turu ekler, ardından aynı yerel iş parçacığı kimliğini ve konuşma
  geçmişini zorunlu tutarak Gateway ile Codex app-server'ı üç kez kapatıp yeniden
  başlatır. Sınırlı sayıları `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS_HISTORY_TURNS` (1-20) ve
  `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS_RESTARTS` (1-10) ile geçersiz kılın.
- İsteğe bağlı dağıtma stres testi: `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1`
  ve `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_COUNT` (1-12) ayarlayın. Test düzeneği
  her alt aracıyı eşzamanlı başlatır, tüm sonlandırılmış çalışmaları bekler ve
  her benzersiz alt aracı yanıtını ve yerel iş parçacığı kimliğini doğrular.
- İsteğe bağlı Compaction stres testi: `OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS=1`,
  sınırlı yerel araç çıktısı oluşturur, otomatik Compaction olaylarını zorunlu kılar,
  kalıcı Compaction sayısını ve gizli işaretçi hatırlamasını doğrular, Gateway'i
  ve fiziksel Codex app-server'ı yeniden başlatır, ardından çıktı ve Compaction
  dalgasını tekrarlar. Sınırlı işi `OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS_TURNS` (1-8) ve
  `OPENCLAW_LIVE_CODEX_HARNESS_LARGE_OUTPUT_BYTES` (100000-800000) ile ayarlayın.
- Tam doğrudan API bağlamı: `OPENCLAW_LIVE_CODEX_HARNESS_FULL_CONTEXT=1`,
  `922000` bağlamını ve `700000` toplam Compaction sınırlarını uygular, yoğun ve sınırlı
  kullanıcı turları gönderir, her dalgada iki açık yerel Compaction kontrol noktası çalıştırır ve
  her kontrol noktasından sonra sonraki turlarla devam eder.
  `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key` ile mutlak bir
  `OPENCLAW_LIVE_CODEX_HARNESS_MODEL_CATALOG` yolu gerektirir. Codex'in geçersiz kılmayı
  normal katalog penceresine geri sınırlamaması için katalog, seçilen modeli
  `max_context_window: 922000` ile sunmalıdır. Yukarıdaki olağan düşürülmüş eşik
  stres testi, daha katı otomatik Compaction ve gizli işaretçi
  saklama doğrulamalarını korur.
- İsteğe bağlı döngü aktarma devre dışı bırakma yoklaması:
  `OPENCLAW_LIVE_CODEX_HARNESS_DISABLE_LOOP_RELAY=1`
- İstenen düşünme tercihi, Codex'in bu model için sunduğu en yakın eforla
  eşleştirilebilir. Örneğin Luna, `minimal` değerini `low` ile eşleştirir.
- Bilinen Codex katalog modelleri, tam olarak bu yerel eforu otomatik olarak türetir.
  Bilinmeyen model geçersiz kılmalarında beklenen eşlenmiş efor belirtilmelidir.
- Smoke testi, sağlayıcı/modeli `agentRuntime.id: "codex"` olarak zorlar; böylece bozuk bir Codex
  test düzeneği sessizce OpenClaw'a geri dönerek testi geçemez.
- Kimlik doğrulama: yerel Codex abonelik oturumundan Codex app-server kimlik doğrulaması veya
  `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key` olduğunda `OPENAI_API_KEY`. Docker, abonelik çalıştırmaları için
  `~/.codex/auth.json` ve `~/.codex/config.toml` dosyalarını kopyalayabilir.

Yerel tarif:

```bash
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/gpt-5.6-luna \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Docker tarifi:

```bash
pnpm test:docker:live-codex-harness
```

Yeniden başlatma ve geçmiş stres testi:

```bash
OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1 \
pnpm test:docker:live-codex-harness
```

Dağıtma, büyük çıktı, Compaction ve yeniden başlatma stres testi:

```bash
OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_COUNT=8 \
  OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS=1 \
  pnpm test:docker:live-codex-harness
```

Tam yerel Codex `922000` girdi bütçesi Compaction stres testi:

```bash
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_FULL_CONTEXT=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL_CATALOG=/absolute/path/to/models-api-1m.json \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/gpt-5.6-terra \
  OPENCLAW_LIVE_CODEX_HARNESS_THINKING=medium \
  OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS_TURNS=8 \
  OPENCLAW_LIVE_CODEX_HARNESS_LARGE_OUTPUT_BYTES=800000 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

GPT-5.6 yerel Codex matrisi:

```bash
OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_TARGETS='openai/gpt-5.6-sol=ultra,openai/gpt-5.6-terra=ultra,openai/gpt-5.6-luna=max' \
  pnpm test:docker:live-codex-harness
```

## Canlı: OpenAI yinelenen Compaction

- Amaç: gömülü OpenClaw `openai-responses` ajan döngüsünü en az
  iki gerçek otomatik Compaction işleminden geçirmek, ardından kalıcı bir işaretçinin korunduğunu doğrulamak.
- Test: `src/agents/sessions/agent-session.openai-compaction.live.test.ts`
- Etkinleştirme: `OPENCLAW_LIVE_OPENAI_COMPACTION=1`
- Varsayılan model: `gpt-5.6-luna`
- Model geçersiz kılma: `OPENCLAW_LIVE_OPENAI_COMPACTION_MODEL=<model>`
- Normal stres modu, aynı gerçek Compaction yoluna sınırlı API
  harcamasıyla ulaşmak için azaltılmış bir istemci bağlam bütçesi kullanır.
- Tam bağlam modu, istemci bütçesini `922000` ve Compaction rezervini
  `222000` olarak ayarlar; böylece otomatik Compaction `700000` değerinde başlar. Ayrıca
  `272000` uzun bağlam fiyatlandırma sınırının üzerinde gözlemlenmiş bir sağlayıcı girdi sayısı gerektirir.

Sınırlı canlı tarif:

```bash
OPENCLAW_LIVE_TEST=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION=1 \
  pnpm test:live -- src/agents/sessions/agent-session.openai-compaction.live.test.ts
```

Tam `922000` girdi bütçesi tarifi:

```bash
OPENCLAW_LIVE_TEST=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION_FULL=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION_MODEL=gpt-5.6-terra \
  pnpm test:live -- src/agents/sessions/agent-session.openai-compaction.live.test.ts
```

<Warning>
Tam mod, OpenAI'ın uzun bağlam fiyatlandırma sınırını bilinçli olarak aşar ve
birkaç büyük API çağrısı yapabilir. Yalnızca açık harcama onayıyla kullanın.
</Warning>

Yeni OpenAI API anahtarı varsayılanı:

```bash
OPENCLAW_LIVE_GATEWAY_OPENAI_API_DEFAULT=1 \
  OPENCLAW_LIVE_GATEWAY_PROVIDERS=openai \
  OPENCLAW_LIVE_GATEWAY_THINKING=off \
  pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
```

Bu kanıt `OPENCLAW_LIVE_GATEWAY_MODELS` ayarını yapılmamış durumda bırakır, modeli
yeni ilk katılım çıkarım seçimi bağlantı noktası üzerinden çözümler, `openai/gpt-5.6` değerini doğrular ve ardından
çözümlenen modelle gerçek bir Gateway turu çalıştırır.

GPT-5.6 gömülü OpenClaw matrisi:

```bash
OPENCLAW_LIVE_GATEWAY_THINKING=ultra \
  OPENCLAW_LIVE_GATEWAY_PROVIDERS=openai \
  OPENCLAW_LIVE_GATEWAY_MODELS='openai/gpt-5.6-sol,openai/gpt-5.6-terra,openai/gpt-5.6-luna' \
  pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
```

Docker notları:

- Docker çalıştırıcısı `scripts/test-live-codex-harness-docker.sh` konumundadır.
- `OPENAI_API_KEY` değerini aktarır, mevcut olduğunda Codex CLI kimlik doğrulama dosyalarını kopyalar,
  `@openai/codex` paketini yazılabilir, bağlanmış bir npm
  ön ekine kurar, kaynak ağacını hazırlar ve ardından yalnızca Codex test düzeneği canlı testini çalıştırır.
- Docker, görüntü, MCP/araç ve Guardian yoklamalarını varsayılan olarak etkinleştirir. Daha dar kapsamlı bir hata ayıklama
  çalıştırması gerektiğinde `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` veya
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` ya da
  `OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0` ayarını yapın.
- Docker aynı açık Codex çalışma zamanı yapılandırmasını kullanır; bu nedenle eski diğer adlar veya OpenClaw
  geri dönüşü, Codex test düzeneğindeki bir gerilemeyi gizleyemez.
- Matris hedefleri tek bir konteynerde sırayla çalışır. Docker betiği,
  varsayılan 35 dakikalık zaman aşımını hedef sayısına göre ölçeklendirir; dış kabuk veya CI zaman aşımı da
  aynı toplam süreye izin vermelidir. Standart CI, her GPT-5.6 hedefini ayrı bir parçada tutar.

### Önerilen canlı tarifler

Dar ve açık izin listeleri en hızlı ve en az kararsız olanlardır:

- Tek model, doğrudan (Gateway olmadan):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.6-luna" pnpm test:live src/agents/models.profiles.live.test.ts`

- Küçük model doğrudan profili:
  - `OPENCLAW_LIVE_MODELS=small pnpm test:live src/agents/models.profiles.live.test.ts`

- Küçük model Gateway profili:
  - `OPENCLAW_LIVE_GATEWAY_MODELS=small pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Ollama Cloud API duman testi:
  - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com OPENCLAW_LIVE_OLLAMA_MODEL=glm-5.1:cloud OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 pnpm test:live -- extensions/ollama/ollama.live.test.ts`

- Tek model, Gateway duman testi:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Birden çok sağlayıcıda araç çağırma:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,google/gemini-3.5-flash,deepseek/deepseek-v4-flash,zai/glm-5.1,minimax/MiniMax-M3" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Z.AI Coding Plan GLM-5.2 doğrudan duman testi:
  - `ZAI_CODING_LIVE_TEST=1 pnpm test:live src/agents/zai.live.test.ts`

- Google odağı (Gemini API anahtarı + Antigravity):
  - Gemini (API anahtarı): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3.5-flash" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google uyarlamalı düşünme duman testi (özel QA CLI'dan `qa manual` — `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1` ve bir kaynak çalışma kopyası gerektirir; bkz. [QA genel bakışı](/tr/concepts/qa-e2e-automation)):
  - Gemini 3 dinamik varsayılanı: `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa manual --provider-mode live-frontier --model google/gemini-3.1-pro-preview --alt-model google/gemini-3.1-pro-preview --message '/think adaptive Reply exactly: GEMINI_ADAPTIVE_OK' --timeout-ms 180000`
  - Gemini 2.5 dinamik bütçesi: `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa manual --provider-mode live-frontier --model google/gemini-2.5-flash --alt-model google/gemini-2.5-flash --message '/think adaptive Reply exactly: GEMINI25_ADAPTIVE_OK' --timeout-ms 180000`

Notlar:

- `google/...`, Gemini API'yi (API anahtarı) kullanır.
- `google-antigravity/...`, Antigravity OAuth köprüsünü (Cloud Code Assist tarzı ajan uç noktası) kullanır.
- `google-gemini-cli/...`, makinenizdeki yerel Gemini CLI'ı kullanır (ayrı kimlik doğrulama ve araç kullanımına özgü farklılıklar).
- Gemini API ile Gemini CLI karşılaştırması:
  - API: OpenClaw, Google'ın barındırılan Gemini API'sini HTTP üzerinden çağırır (API anahtarı / profil kimlik doğrulaması); çoğu kullanıcının "Gemini" ile kastettiği budur.
  - CLI: OpenClaw, yerel bir `gemini` ikili dosyasını kabuk üzerinden çalıştırır; bunun kendi kimlik doğrulaması vardır ve farklı davranabilir (akış/araç desteği/sürüm uyumsuzluğu).

## Canlı: model matrisi (kapsadıklarımız)

Canlı test isteğe bağlıdır, dolayısıyla sabit bir "CI model listesi" yoktur. `OPENCLAW_LIVE_MODELS=modern` / `OPENCLAW_LIVE_GATEWAY_MODELS=modern` (ve bunların `all` diğer adı), `src/agents/live-model-filter.ts` içindeki `HIGH_SIGNAL_LIVE_MODEL_PRIORITY` kaynağından derlenen öncelikli listeyi şu öncelik sırasıyla çalıştırır:

| Sağlayıcı/model                               | Notlar     |
| --------------------------------------------- | ---------- |
| `anthropic/claude-opus-5`                     |            |
| `anthropic/claude-opus-4-8`                   |            |
| `anthropic/claude-sonnet-5`                   |            |
| `anthropic/claude-sonnet-4-6`                 |            |
| `anthropic/claude-opus-4-7`                   |            |
| `google/gemini-3.1-pro-preview`               | Gemini API |
| `google/gemini-3.5-flash`                     | Gemini API |
| `cohere/command-a-plus-05-2026`               |            |
| `moonshot/kimi-k3`                            |            |
| `anthropic/claude-opus-4-6`                   |            |
| `deepseek/deepseek-v4-flash`                  |            |
| `deepseek/deepseek-v4-pro`                    |            |
| `minimax/MiniMax-M3`                          |            |
| `openai/gpt-5.5`                              |            |
| `openrouter/openai/gpt-5.2-chat`              |            |
| `openrouter/minimax/minimax-m2.7`             |            |
| `opencode-go/glm-5`                           |            |
| `openrouter/ai21/jamba-large-1.7`             |            |
| `xai/grok-4.5`                                |            |
| `xai/grok-4.20-0309-reasoning`                |            |
| `zai/glm-5.1`                                 |            |
| `fireworks/accounts/fireworks/models/glm-5p1` |            |
| `minimax-portal/minimax-m3`                   |            |

`SMALL_LIVE_MODEL_PRIORITY` kaynağındaki derlenmiş **küçük model** listesi (`OPENCLAW_LIVE_MODELS=small` / `OPENCLAW_LIVE_GATEWAY_MODELS=small`):

| Sağlayıcı/model              |
| ---------------------------- |
| `lmstudio/qwen/qwen3.5-9b`   |
| `vllm/qwen/qwen3-8b`         |
| `sglang/qwen/qwen3-8b`       |
| `ollama/gemma3:4b`           |
| `openrouter/qwen/qwen3.5-9b` |
| `openrouter/z-ai/glm-5.1`    |
| `openrouter/z-ai/glm-5`      |
| `zai/glm-5.1`                |

Modern listeye ilişkin notlar:

- `codex` ve `codex-cli` sağlayıcıları varsayılan modern taramanın dışında tutulur (bunlar yukarıda ayrı olarak test edilen CLI arka ucu/ACP davranışını kapsar). `openai/gpt-5.5` ise varsayılan olarak Codex uygulama sunucusu test düzeneği üzerinden yönlendirilir; bkz. [Canlı: Codex uygulama sunucusu test düzeneği duman testi](#live-codex-app-server-harness-smoke).
- `fireworks`, `google`, `openrouter` ve `xai`, modern taramada yalnızca açıkça derlenmiş model kimliklerini çalıştırır (otomatik "bu sağlayıcının her modeli" genişletmesi yoktur).
- Görüntü yoklamasını uygulamak için `OPENCLAW_LIVE_GATEWAY_MODELS` içine en az bir görüntü özellikli model (Claude/Gemini/OpenAI ailesi görüntü varyantları vb.) ekleyin.

Elle seçilmiş, sağlayıcılar arası bir kümede araçlar ve görüntüyle Gateway duman testini çalıştırın:

```bash
OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3.5-flash,google-antigravity/claude-opus-4-6-thinking,deepseek/deepseek-v4-flash,zai/glm-5.1,minimax/MiniMax-M3" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts
```

Derlenmiş listelerin dışında isteğe bağlı ek kapsam (olması faydalıdır; etkinleştirdiğiniz, "araçlar" özelliğine sahip bir model seçin):

- Mistral: `mistral/...`
- Cerebras: `cerebras/...` (erişiminiz varsa)
- LM Studio: `lmstudio/...` (yerel; araç çağırma API moduna bağlıdır)

### Toplayıcılar / alternatif Gateway'ler

Anahtarları etkinleştirdiyseniz şunlar üzerinden de test edebilirsiniz:

- OpenRouter: `openrouter/...` (yüzlerce model; araç ve görüntü özellikli adayları bulmak için `openclaw models scan` kullanın)
- OpenCode: Zen için `opencode/...`, Go için `opencode-go/...` (kimlik doğrulama: `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Canlı matrise ekleyebileceğiniz diğer sağlayıcılar (kimlik bilgileriniz/yapılandırmanız varsa):

- Yerleşik: `anthropic`, `cerebras`, `github-copilot`, `google`, `google-antigravity`, `google-gemini-cli`, `google-vertex`, `groq`, `mistral`, `openai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `zai`
- `models.providers` üzerinden (özel uç noktalar): `minimax` (bulut/API) ve OpenAI/Anthropic uyumlu tüm proxy'ler (LM Studio, vLLM, LiteLLM vb.)

<Tip>
Belgelerde "tüm modeller" ifadesini sabit kodlamayın. Yetkili liste, makinenizde `discoverModels(...)` komutunun döndürdüğü modeller ile kullanılabilir anahtarların belirlediği listedir.
</Tip>

## Kimlik bilgileri (asla kaydetmeyin)

Canlı testler kimlik bilgilerini CLI ile aynı şekilde keşfeder. Bunun pratik sonuçları:

- CLI çalışıyorsa canlı testler de aynı anahtarları bulmalıdır.
- Bir canlı test "kimlik bilgisi yok" diyorsa hatayı `openclaw models list` / model seçimi için uygulayacağınız yöntemle ayıklayın.

- Ajan başına kimlik doğrulama profilleri: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (canlı testlerde "profil anahtarları" ile kastedilen budur)
- Yapılandırma: `~/.openclaw/openclaw.json` (veya `OPENCLAW_CONFIG_PATH`)
- Eski OAuth dizini: `~/.openclaw/credentials/` (mevcut olduğunda hazırlanmış canlı ana dizine kopyalanır, ancak ana profil anahtarı deposu değildir)
- Yerel canlı çalıştırmalar, etkin yapılandırmayı (`agents.*.workspace` / `agentDir` geçersiz kılmaları çıkarılmış şekilde) ve her ajanın `auth-profiles.json` öğesini — ajanın dizininin geri kalanını değil; dolayısıyla `workspace/` ve `sandboxes/` verileri hazırlanmış ana dizine asla ulaşmaz — ayrıca eski `credentials/` dizinini ve desteklenen harici CLI kimlik doğrulama dosyalarını/dizinlerini (`.claude.json`, `.claude/.credentials.json`, `.claude/settings*.json`, `.claude/backups`, `.codex/auth.json`, `.codex/config.toml`, `.gemini`, `.minimax`) geçici bir test ana dizinine kopyalar.

Ortam anahtarlarını kullanmak istiyorsanız bunları yerel testlerden önce dışa aktarın veya aşağıdaki
Docker çalıştırıcılarını açık bir `OPENCLAW_PROFILE_FILE` ile kullanın.

## Deepgram canlı (ses yazıya dökümü)

- Test: `extensions/deepgram/audio.live.test.ts`
- Etkinleştirme: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live extensions/deepgram/audio.live.test.ts`

## BytePlus kodlama planı canlı testi

- Test: `extensions/byteplus/live.test.ts`
- Etkinleştirme: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live extensions/byteplus/live.test.ts`
- İsteğe bağlı model geçersiz kılma: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI iş akışı medya canlı testi

- Test: `extensions/comfy/comfy.live.test.ts`
- Etkinleştirme: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Kapsam:
  - Paketlenmiş comfy görüntü, video ve `music_generate` yollarını uygular
  - `plugins.entries.comfy.config.<capability>` yapılandırılmadıkça her özelliği atlar
  - Comfy iş akışı gönderimi, yoklama, indirmeler veya Plugin kaydı değiştirildikten sonra kullanışlıdır

## Canlı görüntü oluşturma

- Test: `test/image-generation.runtime.live.test.ts`
- Komut: `pnpm test:live test/image-generation.runtime.live.test.ts`
- Test düzeneği: `pnpm test:live:media image`
- Kapsam:
  - Kayıtlı tüm görüntü oluşturma sağlayıcısı pluginlerini listeler
  - Yoklama yapmadan önce önceden dışa aktarılmış sağlayıcı ortam değişkenlerini kullanır
  - Varsayılan olarak depolanan kimlik doğrulama profillerinden önce canlı/ortam API anahtarlarını kullanır; böylece `auth-profiles.json` içindeki eski test anahtarları gerçek kabuk kimlik bilgilerini maskelemez
  - Kullanılabilir kimlik doğrulaması/profili/modeli olmayan sağlayıcıları atlar
  - Yapılandırılmış her sağlayıcıyı paylaşılan görüntü oluşturma çalışma zamanı üzerinden çalıştırır:
    - `<provider>:generate`
    - Sağlayıcı düzenleme desteği bildirdiğinde `<provider>:edit`
- Kapsanan mevcut paketlenmiş sağlayıcılar:
  - `deepinfra`
  - `fal`
  - `google`
  - `minimax`
  - `openai`
  - `openrouter`
  - `vydra`
  - `xai`
- İsteğe bağlı daraltma:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google,openrouter,xai"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="deepinfra"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-2,google/gemini-3.1-flash-image,openrouter/google/gemini-3.1-flash-image-preview,xai/grok-imagine-image"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit,openrouter:generate,xai:default-generate,xai:default-edit"`
- İsteğe bağlı kimlik doğrulama davranışı:
  - Profil deposu kimlik doğrulamasını zorunlu kılmak ve yalnızca ortam değişkenlerinden gelen geçersiz kılmaları yok saymak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

Yayınlanan CLI yolu için sağlayıcı/çalışma zamanı canlı testi geçtikten sonra bir `infer` duman testi ekleyin:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_INFER_CLI_TEST=1 pnpm test:live -- test/image-generation.infer-cli.live.test.ts
openclaw infer image providers --json
openclaw infer image generate \
  --model google/gemini-3.1-flash-image \
  --prompt "Metinsiz, beyaz arka plan üzerinde tek bir mavi kareden oluşan minimal düz test görüntüsü." \
  --output ./openclaw-infer-image-smoke.png \
  --json
```

Bu; CLI bağımsız değişkenlerinin ayrıştırılmasını, yapılandırma/varsayılan ajan çözümlemesini, paketlenmiş plugin etkinleştirmesini, paylaşılan görüntü oluşturma çalışma zamanını ve canlı sağlayıcı isteğini kapsar. Plugin bağımlılıklarının çalışma zamanı yüklenmeden önce mevcut olması beklenir.

## Canlı müzik oluşturma

- Test: `extensions/music-generation-providers.live.test.ts`
- Etkinleştirme: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Test düzeneği: `pnpm test:live:media music`
- Kapsam:
  - Paylaşılan paketlenmiş müzik oluşturma sağlayıcısı yolunu çalıştırır
  - Şu anda `fal`, `google`, `minimax` ve `openrouter` kapsam dahilindedir
  - Yoklama yapmadan önce önceden dışa aktarılmış sağlayıcı ortam değişkenlerini kullanır
  - Varsayılan olarak depolanan kimlik doğrulama profillerinden önce canlı/ortam API anahtarlarını kullanır; böylece `auth-profiles.json` içindeki eski test anahtarları gerçek kabuk kimlik bilgilerini maskelemez
  - Kullanılabilir kimlik doğrulaması/profili/modeli olmayan sağlayıcıları atlar
  - Kullanılabilir olduğunda bildirilen her iki çalışma zamanı modunu da çalıştırır:
    - Yalnızca istem girdisiyle `generate`
    - Sağlayıcı `capabilities.edit.enabled` bildirdiğinde `edit`
  - `comfy`, bu paylaşılan taramadan ayrı kendi canlı dosyasına sahiptir
- İsteğe bağlı daraltma:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.6"`
- İsteğe bağlı kimlik doğrulama davranışı:
  - Profil deposu kimlik doğrulamasını zorunlu kılmak ve yalnızca ortam değişkenlerinden gelen geçersiz kılmaları yok saymak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Canlı video oluşturma

- Test: `extensions/video-generation-providers.live.test.ts`
- Etkinleştirme: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Test düzeneği: `pnpm test:live:media video`
- Kapsam:
  - `alibaba`, `byteplus`, `deepinfra`, `fal`, `google`, `minimax`, `openai`, `openrouter`, `pixverse`, `qwen`, `runway`, `together`, `vydra`, `xai` genelinde paylaşılan paketlenmiş video oluşturma sağlayıcısı yolunu çalıştırır
  - Varsayılan olarak yayın açısından güvenli duman testi yolunu kullanır: sağlayıcı başına bir metinden videoya isteği, bir saniyelik ıstakoz istemi ve `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` üzerinden sağlayıcı başına işlem sınırı (varsayılan olarak `180000`)
  - Sağlayıcı tarafındaki kuyruk gecikmesi yayın süresine baskın gelebileceğinden FAL varsayılan olarak atlanır; açıkça çalıştırmak için `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"` iletin (veya atlama listesini temizleyin)
  - Yoklama yapmadan önce önceden dışa aktarılmış sağlayıcı ortam değişkenlerini kullanır
  - Varsayılan olarak depolanan kimlik doğrulama profillerinden önce canlı/ortam API anahtarlarını kullanır; böylece `auth-profiles.json` içindeki eski test anahtarları gerçek kabuk kimlik bilgilerini maskelemez
  - Kullanılabilir kimlik doğrulaması/profili/modeli olmayan sağlayıcıları atlar
  - Varsayılan olarak yalnızca `generate` çalıştırır
  - Kullanılabilir olduğunda bildirilen dönüştürme modlarını da çalıştırmak için `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1` ayarlayın:
    - Sağlayıcı `capabilities.imageToVideo.enabled` bildirdiğinde ve seçilen sağlayıcı/model paylaşılan taramada arabellek destekli yerel görüntü girdisini kabul ettiğinde `imageToVideo`
    - Sağlayıcı `capabilities.videoToVideo.enabled` bildirdiğinde ve seçilen sağlayıcı/model paylaşılan taramada arabellek destekli yerel video girdisini kabul ettiğinde `videoToVideo`
  - Paylaşılan taramada bildirilen ancak şu anda atlanan `imageToVideo` sağlayıcısı:
    - `vydra` (bu hatta arabellek destekli yerel görüntü girdisi desteklenmez)
  - Sağlayıcıya özgü Vydra kapsamı:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - Bu dosya, `veo3` metinden videoya hattının yanı sıra varsayılan olarak uzak görüntü URL'si fikstürü kullanan bir `kling` görüntüden videoya hattını çalıştırır (geçersiz kılmak için `OPENCLAW_LIVE_VYDRA_KLING_IMAGE_URL`).
  - Sağlayıcıya özgü xAI kapsamı:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "classic Grok Imagine"`
    - Klasik durum önce kare biçiminde yerel bir PNG ilk karesi oluşturur, geometriyi belirtmez, bir saniyelik görüntüden videoya klibi ister, tamamlanana kadar yoklama yapar ve indirilen arabelleği doğrular.
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "Grok Imagine Video 1.5"`
    - 1.5 durumu yerel bir PNG ilk karesi oluşturur, bir saniyelik 1080P görüntüden videoya klibi ister, tamamlanana kadar yoklama yapar ve indirilen arabelleği doğrular.
  - Mevcut `videoToVideo` canlı kapsamı:
    - Yalnızca seçilen model `gen4_aleph` olarak çözümlendiğinde `runway`
  - Paylaşılan taramada bildirilen ancak şu anda atlanan `videoToVideo` sağlayıcıları:
    - `alibaba`, `google`, `openai`, `qwen`, `xai`; çünkü bu yollar şu anda arabellek destekli yerel girdi yerine uzak `http(s)` referans URL'leri gerektirir
- İsteğe bağlı daraltma:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="deepinfra,google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - FAL dahil tüm sağlayıcıları varsayılan taramaya dahil etmek için `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""`
  - Yoğun bir duman testi çalıştırmasında her sağlayıcının işlem sınırını azaltmak için `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000`
- İsteğe bağlı kimlik doğrulama davranışı:
  - Profil deposu kimlik doğrulamasını zorunlu kılmak ve yalnızca ortam değişkenlerinden gelen geçersiz kılmaları yok saymak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Canlı medya test düzeneği

- Komut: `pnpm test:live:media`
- Giriş noktası: Seçilen her paket için `pnpm test:live -- <suite-test-file>` çalıştıran `test/e2e/qa-lab/media/hosted-media-provider-live.ts`; böylece Heartbeat ve sessiz mod davranışı diğer `pnpm test:live` çalıştırmalarıyla tutarlı kalır.
- Amaç:
  - Paylaşılan canlı görüntü, müzik ve video paketlerini tek bir depoya özgü giriş noktası üzerinden çalıştırır
  - Eksik sağlayıcı ortam değişkenlerini `~/.profile` üzerinden otomatik olarak yükler
  - Varsayılan olarak her paketi şu anda kullanılabilir kimlik doğrulamasına sahip sağlayıcılarla otomatik olarak sınırlar
- Bayraklar:
  - `--providers <csv>` genel sağlayıcı filtresidir; `--image-providers` / `--music-providers` / `--video-providers` ise filtreyi tek bir paketle sınırlar
  - `--all-providers` kimlik doğrulamasına dayalı otomatik filtreyi atlar
  - Filtrelemeden sonra çalıştırılabilir sağlayıcı kalmadığında `--allow-empty`, `0` koduyla çıkar
  - `--quiet` / `--no-quiet`, `test:live` öğesine aktarılır
- Örnekler:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## İlgili

- [Test](/tr/help/testing) - birim, entegrasyon, QA ve Docker paketleri
