---
read_when:
    - Belirli bir ilk katılım adımı veya bayrağı arama
    - Etkileşimsiz modla ilk katılımı otomatikleştirme
    - Onboarding davranışında hata ayıklama
sidebarTitle: Onboarding Reference
summary: 'CLI ilk kurulumu için eksiksiz başvuru: her adım, bayrak ve yapılandırma alanı'
title: İlk katılım referansı
x-i18n:
    generated_at: "2026-07-26T23:40:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e5e7e42fa3fc1a6d85ad422d0d28dfeda233c89a4d7e97eee4fb974831816372
    source_path: reference/wizard.md
    workflow: 16
---

Bu, `openclaw onboard` için eksiksiz başvuru belgesidir.
Genel bir bakış için [İlk kurulum (CLI)](/tr/start/wizard) bölümüne bakın. Adım adım
davranış ve çıktılar için [CLI kurulum başvurusu](/tr/start/wizard-cli-reference) bölümüne bakın.

## Akış ayrıntıları (yerel mod)

<Steps>
  <Step title="Sıfırlama (isteğe bağlı)">
    - `--reset`, kurulum çalışmadan önce durumu sıfırlar; bu seçenek olmadan ilk kurulumu yeniden çalıştırmak
      mevcut yapılandırmayı korur ve varsayılan değerler olarak yeniden kullanır.
    - `--reset-scope`, `--reset` seçeneğinin neleri kaldıracağını belirler: `config` (yalnızca yapılandırma
      dosyası), `config+creds+sessions` (varsayılan) veya `full` (çalışma alanını da
      kaldırır).
    - Yapılandırma dosyası geçersizse ilk kurulum durur ve önce
      `openclaw doctor` komutunu çalıştırmanızı, ardından kurulumu yeniden çalıştırmanızı söyler.
    - Sıfırlama, durumu Çöp Kutusu'na taşır (hiçbir zaman doğrudan silmez).

  </Step>
  <Step title="Risk onayı">
    - İlk çalıştırma (veya `wizard.securityAcknowledgedAt` ayarlanmadan önceki herhangi bir çalıştırma),
      ajanların güçlü olduğunu ve tam sistem erişiminin riskli olduğunu
      anladığınızı onaylamanızı ister.
    - `--non-interactive`, `--accept-risk` seçeneğinin açıkça belirtilmesini gerektirir; bu seçenek olmadan
      ilk kurulum istem göstermek yerine hatayla sonlanır.
    - Etkileşimli çalıştırmalarda bayrak yerine bir onay istemi gösterilir; reddedilmesi
      kurulumu iptal eder.

  </Step>
  <Step title="Model/Kimlik doğrulama">
    - **Anthropic API anahtarı**: varsa `ANTHROPIC_API_KEY` değerini kullanır veya bir anahtar ister, ardından daemon tarafından kullanılmak üzere kaydeder.
    - **Anthropic Claude CLI**: Claude CLI oturumu zaten açılmışsa tercih edilen yerel yoldur; OpenClaw alternatif olarak Anthropic kurulum belirteciyle kimlik doğrulamayı da destekler.
    - **OpenAI Code (Codex) aboneliği (OAuth)**: tarayıcı akışı; `code#state` değerini yapıştırın.
      - Birincil modeli olmayan yeni bir kurulumda, Codex çalışma zamanı üzerinden `agents.defaults.model` değerini `openai/gpt-5.6-sol` olarak ayarlar.
    - **OpenAI Code (Codex) aboneliği (cihaz eşleştirme)**: kısa ömürlü bir cihaz koduyla tarayıcı eşleştirme akışı.
      - Birincil modeli olmayan yeni bir kurulumda, Codex çalışma zamanı üzerinden `agents.defaults.model` değerini `openai/gpt-5.6-sol` olarak ayarlar.
    - **OpenAI API anahtarı**: varsa `OPENAI_API_KEY` değerini kullanır veya bir anahtar ister, ardından kimlik doğrulama profillerinde saklar.
      - Birincil modeli olmayan yeni bir kurulumda, `agents.defaults.model` değerini `openai/gpt-5.6` olarak ayarlar; yalın doğrudan API model kimliği Sol katmanına çözümlenir.
    - OpenAI eklemek veya yeniden kimlik doğrulamak, `openai/gpt-5.5` dahil olmak üzere mevcut ve açıkça belirtilmiş birincil modeli korur. Hesap GPT-5.6'yı sunmuyorsa `openai/gpt-5.5` değerini açıkça seçin; OpenClaw modeli sessizce daha alt bir sürüme düşürmez.
    - **xAI OAuth**: localhost geri çağrısı gerektirmeyen cihaz kodlu tarayıcı oturumu açma yöntemi olduğundan SSH/Docker/VPS üzerinden de çalışır (`--auth-choice xai-oauth`).
    - **xAI API anahtarı**: `XAI_API_KEY` değerini ister (`--auth-choice xai-api-key`).
    - `--auth-choice xai-device-code`, aynı xAI OAuth cihaz kodu akışı için yalnızca elle kullanılan bir uyumluluk takma adı olarak çalışmaya devam eder; yeni betikler için `xai-oauth` kullanın.
    - **OpenCode**: `OPENCODE_API_KEY` (veya `OPENCODE_ZEN_API_KEY`; https://opencode.ai/auth adresinden edinin) değerini ister ve Zen ya da Go kataloğunu seçmenizi sağlar.
    - **Ollama**: önce **Bulut + Yerel**, **Yalnızca bulut** veya **Yalnızca yerel** seçeneklerini sunar. `Cloud only`, `OLLAMA_API_KEY` değerini ister ve `https://ollama.com` kullanır; ana makine destekli modlar Ollama temel URL'sini (varsayılan `http://127.0.0.1:11434`) ister, kullanılabilir modelleri keşfeder ve gerektiğinde seçilen yerel modeli otomatik olarak indirir; `Cloud + Local` ayrıca söz konusu Ollama ana makinesinde bulut erişimi için oturum açılıp açılmadığını denetler.
    - Daha fazla ayrıntı: [Ollama](/tr/providers/ollama)
    - **API anahtarı**: anahtarı sizin için saklar.
    - **Vercel AI Gateway (çok modelli vekil sunucu)**: `AI_GATEWAY_API_KEY` değerini ister.
    - Daha fazla ayrıntı: [Vercel AI Gateway](/tr/providers/vercel-ai-gateway)
    - **Cloudflare AI Gateway**: Hesap Kimliği, Gateway Kimliği ve `CLOUDFLARE_AI_GATEWAY_API_KEY` değerini ister.
    - Daha fazla ayrıntı: [Cloudflare AI Gateway](/tr/providers/cloudflare-ai-gateway)
    - **MiniMax**: yapılandırma otomatik olarak yazılır; barındırılan varsayılan değer `MiniMax-M3` şeklindedir.
      API anahtarı kurulumu `minimax/...`, OAuth kurulumu ise
      `minimax-portal/...` kullanır.
    - Daha fazla ayrıntı: [MiniMax](/tr/providers/minimax)
    - **StepFun**: yapılandırma, Çin veya küresel uç noktalardaki StepFun standard ya da Step Plan için otomatik olarak yazılır.
    - Standard seçeneğinin mevcut varsayılanı `step-3.5-flash`; Step Plan ayrıca `step-3.5-flash-2603` seçeneğini içerir.
    - Daha fazla ayrıntı: [StepFun](/tr/providers/stepfun)
    - **Synthetic (Anthropic uyumlu)**: `SYNTHETIC_API_KEY` değerini ister.
    - Daha fazla ayrıntı: [Synthetic](/tr/providers/synthetic)
    - **Moonshot (Kimi K2)**: yapılandırma otomatik olarak yazılır.
    - **Kimi Coding**: yapılandırma otomatik olarak yazılır.
    - Daha fazla ayrıntı: [Moonshot AI (Kimi + Kimi Coding)](/tr/providers/moonshot)
    - **Özel Sağlayıcı**: OpenAI uyumlu, OpenAI Responses uyumlu veya Anthropic uyumlu uç noktalarla çalışır. Etkileşimsiz bayraklar: `--auth-choice custom-api-key`, `--custom-base-url`, `--custom-model-id`, `--custom-api-key` (isteğe bağlı; `CUSTOM_API_KEY` değerine geri döner), `--custom-provider-id` (isteğe bağlı; temel URL'den otomatik türetilir), `--custom-compatibility openai|openai-responses|anthropic` (varsayılan `openai`), `--custom-image-input` / `--custom-text-input` (çıkarımla belirlenen görsel model algılamasını geçersiz kılar).
    - **Atla**: henüz kimlik doğrulama yapılandırılmaz.
    - Algılanan seçeneklerden varsayılan bir model seçin (veya sağlayıcı/modeli elle girin). En iyi kalite ve daha düşük istem enjeksiyonu riski için sağlayıcı yığınınızda bulunan en güçlü, en yeni nesil modeli seçin.
    - İlk kurulum bir model denetimi çalıştırır ve yapılandırılan model bilinmiyorsa veya kimlik doğrulaması eksikse uyarır.
    - API anahtarı depolama modu varsayılan olarak düz metin kimlik doğrulama profili değerlerini kullanır. Bunun yerine ortam değişkeni destekli başvuruları saklamak için `--secret-input-mode ref` kullanın (örneğin `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`); başvurulan ortam değişkeni önceden ayarlanmış olmalıdır, aksi takdirde ilk kurulum hemen başarısız olur.
    - Kimlik doğrulama profilleri `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` içinde bulunur (API anahtarları + OAuth). `~/.openclaw/credentials/oauth.json` yalnızca eski içe aktarma içindir.
    - Daha fazla ayrıntı: [OAuth](/tr/concepts/oauth)
    <Note>
    Başsız/sunucu ipucu: OAuth işlemini tarayıcısı olan bir makinede tamamlayın, ardından
    ilgili ajanın `auth-profiles.json` dosyasını (örneğin
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` veya buna karşılık gelen
    `$OPENCLAW_STATE_DIR/...` yolunu) Gateway ana makinesine kopyalayın. `credentials/oauth.json`
    yalnızca eski bir içe aktarma kaynağıdır.
    </Note>
  </Step>
  <Step title="Çalışma alanı">
    - Varsayılan `~/.openclaw/workspace` (yapılandırılabilir).
    - Ajan önyükleme ritüeli için gereken çalışma alanı dosyalarını başlangıç verileriyle oluşturur.
    - Tam çalışma alanı düzeni + yedekleme kılavuzu: [Ajan çalışma alanı](/tr/concepts/agent-workspace)

  </Step>
  <Step title="Gateway">
    - Bağlantı noktası (varsayılan **18789**), bağlama, kimlik doğrulama modu, Tailscale üzerinden erişim.
    - Kimlik doğrulama önerisi: yerel WS istemcilerinin kimlik doğrulaması gerekmesi için geri döngüde bile **Belirteç** seçeneğini koruyun.
    - Belirteç modunda etkileşimli kurulum şunları sunar:
      - **Düz metin belirteci oluştur/sakla** (varsayılan)
      - **SecretRef kullan** (isteğe bağlı)
      - Hızlı başlangıç, ilk kurulum yoklaması/pano önyüklemesi için `env`, `file` ve `exec` sağlayıcılarında mevcut `gateway.auth.token` SecretRef değerlerini yeniden kullanır.
      - Söz konusu SecretRef yapılandırılmış ancak çözümlenemiyorsa ilk kurulum, çalışma zamanı kimlik doğrulamasını sessizce zayıflatmak yerine anlaşılır bir düzeltme mesajıyla erkenden başarısız olur.
    - Parola modunda etkileşimli kurulum, düz metin veya SecretRef depolamayı da destekler.
    - Etkileşimsiz belirteç SecretRef yolu: `--gateway-token-ref-env <ENV_VAR>`.
      - İlk kurulum işleminin ortamında boş olmayan bir ortam değişkeni gerektirir.
      - `--gateway-token` ile birlikte kullanılamaz.
    - Kimlik doğrulamayı yalnızca her yerel işleme tamamen güveniyorsanız devre dışı bırakın.
    - Geri döngü dışı bağlamalar yine de kimlik doğrulama gerektirir.

  </Step>
  <Step title="Kanallar">
    - [WhatsApp](/tr/channels/whatsapp): isteğe bağlı QR ile oturum açma.
    - [Telegram](/tr/channels/telegram): bot belirteci.
    - [Discord](/tr/channels/discord): bot belirteci.
    - [Google Chat](/tr/channels/googlechat): hizmet hesabı JSON'u + Webhook hedef kitlesi.
    - [Mattermost](/tr/channels/mattermost) (plugin): bot belirteci + temel URL.
    - [Signal](/tr/channels/signal) (plugin): isteğe bağlı `signal-cli` kurulumu + hesap yapılandırması.
    - [iMessage](/tr/channels/imessage): `imsg` CLI yolu + Messages DB erişimi; Gateway Mac dışında çalışıyorsa SSH sarmalayıcısı kullanın.
    - Discord, Feishu, Microsoft Teams, QQ Bot, Slack ve diğer kanallar,
      ilk kurulumun sizin için yükleyebileceği pluginler olarak sunulur. Tam katalog: [Kanallar](/tr/channels).
    - DM güvenliği: varsayılan yöntem eşleştirmedir. İlk DM bir kod gönderir; `openclaw pairing approve <channel> <code>` aracılığıyla onaylayın veya izin listelerini kullanın.

  </Step>
  <Step title="Web araması">
    - Brave, Codex (Barındırılan Arama), DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Parallel, Perplexity, SearXNG veya Tavily gibi desteklenen bir sağlayıcı seçin (ya da atlayın).
    - API destekli sağlayıcılar hızlı kurulum için ortam değişkenlerini veya mevcut yapılandırmayı kullanabilir; anahtar gerektirmeyen sağlayıcılar ise kendilerine özgü ön koşulları kullanır.
    - `--skip-search` ile atlayın.
    - Daha sonra yapılandırın: `openclaw configure --section web`.

  </Step>
  <Step title="Daemon kurulumu">
    - macOS: LaunchAgent
      - Oturum açmış bir kullanıcı oturumu gerektirir; başsız kullanım için özel bir LaunchDaemon kullanın (birlikte sunulmaz).
    - Linux (ve WSL2 üzerinden Windows): systemd kullanıcı birimi
      - İlk kurulum, oturum kapatıldıktan sonra Gateway'in çalışmaya devam etmesi için `loginctl enable-linger <user>` aracılığıyla kalıcı oturumu etkinleştirmeye çalışır.
      - Sudo isteyebilir (`/var/lib/systemd/linger` dosyasına yazar); önce sudo olmadan dener.
    - Yerel Windows: önce Zamanlanmış Görev; görev oluşturma reddedilirse OpenClaw, kullanıcı başına Başlangıç klasöründeki bir oturum açma öğesine geri döner ve Gateway'i hemen başlatır.
    - **Çalışma zamanı seçimi:** Standart çalışma zamanı durum deposu `node:sqlite` kullandığından Node gereklidir. Eski Bun hizmetleri onarım sırasında Node'a geçirilir.
    - Belirteç kimlik doğrulaması bir belirteç gerektiriyorsa ve `gateway.auth.token` SecretRef tarafından yönetiliyorsa daemon kurulumu bunu doğrular ancak çözümlenen düz metin belirteç değerlerini denetleyici hizmetin ortam meta verilerinde kalıcı olarak saklamaz.
    - Belirteç kimlik doğrulaması bir belirteç gerektiriyor ve yapılandırılan belirteç SecretRef'i çözümlenemiyorsa daemon kurulumu, uygulanabilir yönlendirmelerle engellenir.
    - Hem `gateway.auth.token` hem de `gateway.auth.password` yapılandırılmış ve `gateway.auth.mode` ayarlanmamışsa mod açıkça ayarlanana kadar daemon kurulumu engellenir.

  </Step>
  <Step title="Sağlık denetimi">
    - Gateway'i başlatır (gerekirse) ve `openclaw health` komutunu çalıştırır.
    - İpucu: `openclaw status --deep`, desteklendiğinde kanal yoklamaları dahil olmak üzere canlı Gateway sağlık yoklamasını durum çıktısına ekler (erişilebilir bir Gateway gerektirir).

  </Step>
  <Step title="Skills (önerilir)">
    - Kullanılabilir becerileri okur ve gereksinimleri denetler.
    - Bir Node yöneticisi seçmenizi sağlar: **npm / pnpm / bun**.
    - Güvenilir, paketle birlikte sunulan beceriler için isteğe bağlı bağımlılıkları otomatik olarak yükler (bazıları macOS'te Homebrew kullanır).
    - Homebrew, uv veya Go yükleyicisi ön koşulu kullanılamayan becerileri atlar, bunları elle kurulum yönlendirmeleriyle gruplandırır ve ön koşul yüklendikten sonra sizi `openclaw doctor` konumuna yönlendirir.

  </Step>
  <Step title="Tamamlama">
    - Terminal, Tarayıcı veya daha sonrası seçeneklerini sunan **Ajanınızı nasıl başlatmak istiyorsunuz?** istemi dahil olmak üzere özet + sonraki adımlar.

  </Step>
</Steps>

<Note>
GUI algılanmazsa ilk katılım, bir tarayıcı açmak yerine Control UI için SSH bağlantı noktası yönlendirme talimatlarını yazdırır.
Control UI varlıkları eksikse ilk katılım bunları derlemeyi dener; geri dönüş seçeneği `pnpm ui:build` şeklindedir (UI bağımlılıklarını otomatik olarak yükler).
</Note>

## Etkileşimsiz mod

İlk katılımı otomatikleştirmek veya betiklemek için `--non-interactive --accept-risk` kullanın (bu
bayrak, gerekli risk kabulüdür; onsuz ilk katılım bir hatayla
sonlanır):

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

Makine tarafından okunabilir bir özet için `--json` ekleyin.

Etkileşimsiz modda Gateway token'ı SecretRef'i:

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN
```

`--gateway-token` ve `--gateway-token-ref-env` birbirini dışlar.

<Note>
`--json`, etkileşimsiz modu **ifade etmez**. Betikler için `--non-interactive --accept-risk` (ve `--workspace`) kullanın.
</Note>

Sağlayıcıya özgü komut örnekleri [CLI Otomasyonu](/tr/start/wizard-cli-automation#provider-specific-examples) sayfasında yer alır.
Bayrak anlamları ve adım sıralaması için bu başvuru sayfasını kullanın.

### Aracı ekleme (etkileşimsiz)

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.6-sol \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

`main` ayrılmış bir aracı kimliğidir ve `openclaw agents add` için kullanılamaz.

## Gateway sihirbazı RPC'si

Gateway, ilk katılım akışını RPC üzerinden sunar (`wizard.start`, `wizard.next`, `wizard.cancel`, `wizard.status`).
İstemciler (macOS uygulaması, Control UI), ilk katılım mantığını yeniden uygulamadan adımları işleyebilir.

## Signal kurulumu (signal-cli)

İlk katılım, `signal-cli` öğesinin `PATH` üzerinde bulunup bulunmadığını algılar ve eksikse yüklemeyi önerir:

- Linux x86-64: Resmî yerel GraalVM derlemesini `signal-cli` GitHub sürümlerinden indirir ve `~/.openclaw/tools/signal-cli/<version>/` altında depolar.
- macOS ve diğer mimariler: Bunun yerine Homebrew aracılığıyla yükler.
- Yerel Windows: Henüz desteklenmemektedir; Linux yükleme yolunu kullanmak için ilk katılımı WSL2 içinde çalıştırın.
- Her iki durumda da `kind: "managed-native"` ile `channels.signal.transport.cliPath` dosyasını yazar.

## Sihirbazın yazdıkları

`~/.openclaw/openclaw.json` içindeki tipik alanlar:

- `agents.defaults.workspace`
- `--skip-bootstrap` geçirildiğinde `agents.defaults.skipBootstrap`
- `agents.defaults.model` / `models.providers` (Minimax seçilirse)
- `tools.profile` (yerel ilk katılım, ayarlanmamışsa varsayılan olarak `"coding"` kullanır; mevcut açık değerler korunur)
- `gateway.*` (mod, bağlama, kimlik doğrulama, tailscale)
- `session.dmScope` (ilk katılım açık değerleri korur, aksi takdirde ayarlanmamış bırakır; böylece `"main"` varsayılanı, kanallar arasındaki tüm doğrudan mesajları aracının sürekli ana oturumunda tutar—kişisel aracı varsayılanı. Paylaşılan veya çok kullanıcılı gelen kutuları için `"per-channel-peer"` kullanın; `openclaw security audit`, çok kullanıcılı DM trafiği algıladığında yalıtım önerir. Ayrıntılar: [CLI Kurulum Başvurusu](/tr/start/wizard-cli-reference#outputs-and-internals))
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- Kanal istemleri sırasında katılmayı seçtiğinizde kanal DM izin listeleri. Discord, Matrix, Microsoft Teams ve Slack mümkün olduğunda adları kimliklere çözümler; diğer kanallar kimlikleri doğrudan alır (örneğin sayısal Telegram gönderici kimlikleri veya WhatsApp telefon numaraları).
- `skills.install.nodeManager`
  - `setup --node-manager`; `npm`, `pnpm` veya `bun` kabul eder.
  - Elle yapılandırma, `skills.install.nodeManager` doğrudan ayarlanarak `yarn` kullanmaya devam edebilir.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`
- `wizard.securityAcknowledgedAt`

`openclaw agents add`, `agents.entries.*` ve isteğe bağlı `bindings` yazar.

WhatsApp kimlik bilgileri `~/.openclaw/credentials/whatsapp/<accountId>/` altında bulunur.
Etkin oturumlar ve dökümler
`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` içinde depolanır.
`~/.openclaw/agents/<agentId>/sessions/` dizini, eski geçiş
girdileri ve arşiv/destek yapıtları için kullanılır.

Bazı kanallar Plugin olarak sunulur. Kurulum sırasında bunlardan birini seçtiğinizde ilk katılım,
yapılandırılabilmesi için önce onu yüklemenizi (npm veya yerel bir yol üzerinden) ister.

## İlgili belgeler

- İlk katılıma genel bakış: [İlk Katılım (CLI)](/tr/start/wizard)
- CLI kurulum başvurusu: [CLI kurulum başvurusu](/tr/start/wizard-cli-reference)
- macOS uygulamasında ilk katılım: [İlk Katılım](/tr/start/onboarding)
- Yapılandırma başvurusu: [Gateway yapılandırması](/tr/gateway/configuration)
- Sağlayıcılar: [WhatsApp](/tr/channels/whatsapp), [Telegram](/tr/channels/telegram), [Discord](/tr/channels/discord), [Google Chat](/tr/channels/googlechat), [Signal](/tr/channels/signal), [iMessage](/tr/channels/imessage)
- Skills: [Skills](/tr/tools/skills), [Skills yapılandırması](/tr/tools/skills-config)
