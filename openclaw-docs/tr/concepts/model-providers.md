---
read_when:
    - Sağlayıcı bazında bir model kurulum referansına ihtiyacınız var
    - Model sağlayıcıları için örnek yapılandırmalar veya CLI ilk katılım komutları istiyorsunuz
sidebarTitle: Model providers
summary: Örnek yapılandırmalar + CLI akışlarıyla model sağlayıcılarına genel bakış
title: Model sağlayıcıları
x-i18n:
    generated_at: "2026-07-26T23:15:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 51ce1ca5dde28821596ca667619cd860cebda4787993fadb6b0e20fc0e1624ac
    source_path: concepts/model-providers.md
    workflow: 16
---

**LLM/model sağlayıcıları** (WhatsApp/Telegram gibi sohbet kanalları değil) için başvuru kaynağı. Model seçimi kuralları için [Modeller](/tr/concepts/models) bölümüne bakın.

## Hızlı kurallar

<AccordionGroup>
  <Accordion title="Model referansları ve CLI yardımcıları">
    - Model referansları `provider/model` kullanır (örnek: `opencode/claude-opus-4-6`).
    - `agents.defaults.models` takma adları ve model başına ayarları saklar; `agents.defaults.modelPolicy.allow` isteğe bağlı açık geçersiz kılma izin listesidir.
    - CLI yardımcıları: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
    - `models.providers.*.contextWindow` / `contextTokens` / `maxTokens` sağlayıcı düzeyindeki varsayılanları belirler; `models.providers.*.models[].contextWindow` / `contextTokens` / `maxTokens` bunları model bazında geçersiz kılar.
    - Geri dönüş kuralları, bekleme süresi yoklamaları ve oturum geçersiz kılmalarının kalıcılığı: [Model yük devri](/tr/concepts/model-failover).

  </Accordion>
  <Accordion title="Sağlayıcı kimlik doğrulaması eklemek birincil modelinizi değiştirmez">
    `openclaw configure`, bir sağlayıcı eklediğinizde veya yeniden kimlik doğrulaması yaptığınızda mevcut `agents.defaults.model.primary` değerini korur. `openclaw models auth login` da `--set-default` iletmediğiniz sürece aynısını yapar. Sağlayıcı Plugin'leri, kimlik doğrulama yapılandırması yamalarında yine de önerilen bir varsayılan model döndürebilir; ancak birincil model zaten mevcutsa OpenClaw bunu "geçerli birincil modeli değiştir" olarak değil, "bu modeli kullanılabilir hâle getir" olarak değerlendirir.

    Varsayılan modeli bilinçli olarak değiştirmek için `openclaw models set <provider/model>` veya `openclaw models auth login --provider <id> --set-default` kullanın.

  </Accordion>
  <Accordion title="OpenAI sağlayıcı/çalışma zamanı ayrımı">
    OpenAI model referansları ve aracı çalışma zamanları birbirinden ayrıdır:

    - `openai/<model>`, standart OpenAI sağlayıcısını ve modelini seçer. Önek tek başına hiçbir zaman Codex'i seçmez.
    - Sağlayıcı/model çalışma zamanı ilkesi ayarlanmamışsa veya `auto` ise OpenAI, yalnızca yazılmış bir istek geçersiz kılması bulunmayan, tam olarak resmî HTTPS Platform Responses veya ChatGPT Responses rotası için Codex'i örtük olarak seçebilir.
    - Yazılmış Completions bağdaştırıcıları, özel uç noktalar ve yazılmış istek davranışına sahip rotalar OpenClaw üzerinde kalır. Düz metin kullanan resmî HTTP uç noktaları reddedilir.
    - Eski Codex model referansları, doctor'ın `openai/<model>` olarak yeniden yazdığı eski yapılandırmadır.
    - Sağlayıcı/model `agentRuntime.id: "openclaw"`, aksi hâlde uygun olan bir rotayı açıkça OpenClaw üzerinde tutar. `agentRuntime.id: "codex"` Codex gerektirir ve etkin rota Codex ile uyumlu değilse kapalı biçimde başarısız olur.

    [OpenAI örtük aracı çalışma zamanı](/tr/providers/openai#implicit-agent-runtime) ve [Codex koşum takımı](/tr/plugins/codex-harness) bölümlerine bakın. Sağlayıcı/çalışma zamanı ayrımı kafa karıştırıcıysa önce [Aracı çalışma zamanları](/tr/concepts/agent-runtimes) bölümünü okuyun.

    Plugin otomatik etkinleştirme de aynı sınırı izler: Örtük olarak Codex ile uyumlu etkin bir rota Codex Plugin'ini etkinleştirebilirken açık sağlayıcı/model `agentRuntime.id: "codex"` veya eski `codex/<model>` referansları bunu gerektirir. `openai/*` öneki tek başına bunu gerektirmez.

    Yeni OpenAI kurulumu rotaya özgü bir GPT-5.6 referansı kullanır: API anahtarıyla kurulum
    `openai/gpt-5.6` seçer (çıplak doğrudan API kimliği Sol olarak çözümlenir), buna karşılık
    ChatGPT/Codex OAuth, yerel Codex kataloğu için tam olarak `openai/gpt-5.6-sol` seçer.
    `openai/gpt-5.5` dâhil mevcut açık birincil modeller, OpenAI kimlik doğrulaması
    eklendiğinde veya yenilendiğinde korunur. GPT-5.5, GPT-5.6 erişimi olmayan hesaplar için
    her iki çalışma zamanı üzerinden de açık bir kurtarma seçeneği olarak kullanılabilir.

  </Accordion>
  <Accordion title="CLI çalışma zamanları">
    CLI çalışma zamanları aynı ayrımı kullanır: `anthropic/claude-*` veya `google/gemini-*` gibi standart model referanslarını seçin, ardından yerel bir CLI arka ucu istediğinizde sağlayıcı/model çalışma zamanı ilkesini `claude-cli` veya `google-gemini-cli` olarak ayarlayın.

    Eski `claude-cli/*` ve `google-gemini-cli/*` referansları, çalışma zamanı ayrı olarak kaydedilerek standart sağlayıcı referanslarına geri taşınır. Eski `codex-cli/*` referansları `openai/*` biçimine taşınır ve Codex uygulama sunucusu rotasını kullanır; OpenClaw artık paketlenmiş bir Codex CLI arka ucunu barındırmaz.

  </Accordion>
</AccordionGroup>

## Control UI'da sağlayıcıları yapılandırma

`models.providers.<id>.apiKey` içinde saklanan sağlayıcı API anahtarlarını eklemek, değiştirmek veya kaldırmak için Control UI'da **Settings → Model Providers** bölümünü açın. Sayfa, kimlik bilgilerini göstermeden her API anahtarının OpenClaw yapılandırmasından mı yoksa bir ortam değişkeninden mi geldiğini belirtir. Ortam tarafından sağlanan anahtarlar Gateway işleminin ortamı tarafından yönetilmeye devam eder.

Canlı bir sağlayıcı yoklaması çalıştırmak ve gecikmeyi ya da kategorize edilmiş bir kimlik doğrulama, hız sınırı, faturalandırma, zaman aşımı veya yanıt hatasını görmek için **Test connection** seçeneğini kullanın. Yoklama gerçek bir sağlayıcı isteği gönderir ve az sayıda token tüketebilir. OAuth ve token profillerindeki oturumlar da sağlayıcı kartından kapatılabilir.

**Default models** kartı, yapılandırılmış model kataloğundaki birincil modeli, sıralı geri dönüş modellerini ve yardımcı modeli yönetir. Modelleri seçin, ardından mevcut `agents.defaults.model` ve `agents.defaults.utilityModel` ayarlarına birlikte kaydedin. Yardımcı model için **Automatic** ayarı belirlenmemiş bırakır; **Disabled** ise yardımcı yönlendirmeyi kapatmak üzere boş bir dize saklar.

## Plugin'e ait sağlayıcı davranışı

Sağlayıcıya özgü mantığın çoğu sağlayıcı Plugin'lerinde (`registerProvider(...)`) bulunurken OpenClaw genel çıkarım döngüsünü korur. Plugin'ler ilk kurulumu, model kataloglarını, kimlik doğrulama ortam değişkeni eşlemesini, aktarım/yapılandırma normalleştirmesini, araç şeması temizliğini, yük devri sınıflandırmasını, OAuth yenilemeyi, kullanım raporlamasını, düşünme/akıl yürütme profillerini ve daha fazlasını yönetir.

Sağlayıcı SDK kancalarının ve paketlenmiş Plugin örneklerinin tam listesi [Sağlayıcı Plugin'leri](/tr/plugins/sdk-provider-plugins) bölümünde bulunur. Tamamen özel bir istek yürütücüsüne ihtiyaç duyan bir sağlayıcı, ayrı ve daha derin bir genişletme yüzeyidir.

<Note>
Sağlayıcıya ait çalıştırıcı davranışı; yeniden oynatma ilkesi, araç şeması normalleştirmesi, akış sarmalama ve aktarım/istek yardımcıları gibi açık sağlayıcı kancalarında bulunur. Eski `ProviderPlugin.capabilities` statik paketi yalnızca uyumluluk içindir ve artık paylaşılan çalıştırıcı mantığı tarafından okunmaz.
</Note>

## API anahtarı rotasyonu

<AccordionGroup>
  <Accordion title="Anahtar kaynakları ve öncelik">
    Birden çok anahtarı şunlar aracılığıyla yapılandırın:

    - `OPENCLAW_LIVE_<PROVIDER>_KEY` (tek canlı geçersiz kılma, en yüksek öncelik)
    - `<PROVIDER>_API_KEYS` (virgülle veya noktalı virgülle ayrılmış liste)
    - `<PROVIDER>_API_KEY` (birincil anahtar)
    - `<PROVIDER>_API_KEY_*` (numaralandırılmış liste, ör. `<PROVIDER>_API_KEY_1`)

    Google sağlayıcıları için `GOOGLE_API_KEY` da geri dönüş olarak dâhil edilir. Anahtar seçimi sırası önceliği korur ve yinelenen değerleri kaldırır.

  </Accordion>
  <Accordion title="Rotasyon ne zaman devreye girer">
    - İstekler yalnızca hız sınırı yanıtlarında (örneğin `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many concurrent requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai ... quota limit exceeded` veya dönemsel kullanım sınırı iletilerinde) sonraki anahtarla yeniden denenir.
    - Hız sınırı dışındaki hatalar anında başarısız olur; anahtar rotasyonu denenmez.
    - Tüm aday anahtarlar başarısız olduğunda son denemenin nihai hatası döndürülür.

  </Accordion>
</AccordionGroup>

## Resmî sağlayıcı Plugin'leri

Resmî sağlayıcı Plugin'leri kendi model kataloğu satırlarını yayımlar. Bu sağlayıcılar `models.providers` model girdisi gerektirmez; sağlayıcı Plugin'ini etkinleştirin, kimlik doğrulamayı ayarlayın ve bir model seçin. `models.providers` yalnızca açık özel sağlayıcılar veya zaman aşımları gibi dar kapsamlı istek ayarları için kullanılmalıdır.

### OpenAI

- Sağlayıcı: `openai`
- Kimlik doğrulama: `OPENAI_API_KEY`
- İsteğe bağlı rotasyon: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2` ve `OPENCLAW_LIVE_OPENAI_KEY` (tek geçersiz kılma)
- Yeni kurulum varsayılanı: `openai/gpt-5.6`; doğrudan API'de çıplak kimlik Sol olarak çözümlenir.
- Örnek modeller: `openai/gpt-5.6`, `openai/gpt-5.6-terra`, `openai/gpt-5.6-luna`, `openai/gpt-5.5`
- Belirli bir kurulum veya API anahtarı farklı davranıyorsa hesap/model kullanılabilirliğini `openclaw models list --provider openai` ile doğrulayın.
- CLI: `openclaw onboard --auth-choice openai-api-key`
- Varsayılan aktarım `auto` değeridir; OpenClaw aktarım seçimini paylaşılan model çalışma zamanına iletir.
- Model bazında `agents.defaults.models["openai/<model>"].params.transport` aracılığıyla geçersiz kılın (`"sse"`, `"websocket"` veya `"auto"`)
- OpenAI öncelikli işleme `agents.defaults.models["openai/<model>"].params.serviceTier` aracılığıyla etkinleştirilebilir
- `/fast` ve `params.fastMode`, `api.openai.com` üzerindeki doğrudan `openai/*` Responses isteklerini `service_tier=priority` olarak eşler
- Paylaşılan `/fast` anahtarı yerine açık bir katman istediğinizde `params.serviceTier` kullanın
- Gizli OpenClaw atıf başlıkları (`originator`, `version`, `User-Agent`) genel OpenAI uyumlu proxy'lere değil, yalnızca `api.openai.com` adresine giden yerel OpenAI trafiğine uygulanır
- Yerel OpenAI rotaları ayrıca Responses `store`, istem önbelleği ipuçları ve OpenAI akıl yürütme uyumluluğu yük biçimlendirmesini korur; proxy rotaları korumaz
- `openai/gpt-5.3-codex-spark` yalnızca ChatGPT/Codex OAuth üzerinden kullanılabilir; doğrudan OpenAI API anahtarı ve Azure API anahtarı rotaları bunu reddeder

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.6" } } },
}
```

API kuruluşu GPT-5.6'yı sunmuyorsa
`openai/gpt-5.5` değerini açıkça ayarlayın. Normal ilk kurulum ve yeniden kimlik doğrulama,
mevcut açık birincil modeli korur; `models auth login --set-default` ve
`models set` bilinçli değiştirme yollarıdır.

### Anthropic

- Sağlayıcı: `anthropic`
- Kimlik doğrulama: `ANTHROPIC_API_KEY`
- İsteğe bağlı rotasyon: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2` ve `OPENCLAW_LIVE_ANTHROPIC_KEY` (tek geçersiz kılma)
- Örnek model: `anthropic/claude-opus-5`
- CLI: `openclaw onboard --auth-choice apiKey`
- Doğrudan genel Anthropic istekleri, `api.anthropic.com` adresine gönderilen API anahtarı ve OAuth ile kimliği doğrulanmış trafik dâhil olmak üzere paylaşılan `/fast` anahtarını ve `params.fastMode` değerini destekler; OpenClaw bunu Anthropic `service_tier` (`auto` ile `standard_only`) biçiminde eşler
- Tercih edilen Claude CLI yapılandırması, model referansını standart biçimde tutar ve CLI
  arka ucunu ayrı olarak seçer: model kapsamlı `agentRuntime.id: "claude-cli"` ile
  `anthropic/claude-opus-5`. Eski
  `claude-cli/claude-opus-4-7` referansları uyumluluk için çalışmaya devam eder.

<Note>
Claude CLI yeniden kullanımı (`claude -p`) onaylanmış bir OpenClaw entegrasyon yoludur. Anthropic kurulum token'ıyla kimlik doğrulama desteklenmeye devam eder, ancak mevcut olduğunda OpenClaw Claude CLI yeniden kullanımını tercih eder.
</Note>

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-5" } } },
}
```

### OpenAI ChatGPT/Codex OAuth

- Sağlayıcı: `openai`
- Kimlik doğrulama: OAuth (ChatGPT)
- Yeni yerel Codex app-server test düzeneği referansı: `openai/gpt-5.6-sol`
- Yerel Codex app-server test düzeneği belgeleri: [Codex test düzeneği](/tr/plugins/codex-harness)
- Eski model referansları: `codex/gpt-*`, `openai-codex/gpt-*`
- Plugin sınırı: `openai/*`, OpenAI Plugin'ini yükler; yerel Codex app-server Plugin'inin seçilip seçilmeyeceğini açık çalışma zamanı politikası veya sağlayıcının sahip olduğu etkin rota belirler.
- CLI: `openclaw onboard --auth-choice openai` veya `openclaw models auth login --provider openai`
- OpenClaw'ın yerleşik ChatGPT Responses aktarımı varsayılan olarak `auto` kullanır (önce WebSocket, yedek olarak SSE).
- `agents.defaults.models["openai/<model>"].params.transport`, `params.serviceTier` ve `params.fastMode`, tanımlanmış yerleşik istek ayarlarıdır. Örtük çalışma zamanı seçimini OpenClaw'da tutarlar; yerel Codex, kendi app-server aktarımının ve hizmet katmanının sahibidir.
- Gizli OpenClaw ilişkilendirme başlıkları (`originator`, `version`, `User-Agent`) genel OpenAI uyumlu proxy'lere değil, yalnızca `chatgpt.com/backend-api` hedefine giden yerel Codex trafiğine eklenir
- Paylaşılan `/fast` geçişi, çalışma zamanı denetimi olarak kullanılabilir kalır; tanımlanmış model parametrelerinden farklıdır.
- Yerel Codex kataloğu, hesap erişimine göre tam `openai/gpt-5.6-sol`, `openai/gpt-5.6-terra` ve `openai/gpt-5.6-luna` referanslarını sunabilir. Doğrudan API'nin yalın `gpt-5.6` diğer adını istemci tarafında uygulamaz.
- `openai/gpt-5.5`, Codex kataloğunun yerel `contextWindow = 400000` değerini ve varsayılan çalışma zamanı `contextTokens = 272000` değerini kullanır; çalışma zamanı sınırını `models.providers.openai.models[].contextTokens` ile geçersiz kılın
- `openai` kimlik doğrulamasıyla oturum açın ve abonelik destekli yeni bir kurulum için `openai/gpt-5.6-sol` kullanın. Bu Codex çalışma alanı GPT-5.6'yı sunmuyorsa açıkça `openai/gpt-5.5` seçin.
- Normalde uygun olan bir rotayı yerleşik çalışma zamanında tutmak için `agentRuntime.id: "openclaw"` sağlayıcı/modelini kullanın. Çalışma zamanı ayarlanmamışsa veya `auto` ise yalnızca tanımlanmış istek geçersiz kılması bulunmayan, tam ve resmî bir HTTPS Responses/ChatGPT uyumlu rota Codex'i örtük olarak seçebilir.
- Eski Codex GPT referansları canlı bir sağlayıcı rotası değil, eski durumdur. Yeni aracı yapılandırması için standart `openai/*` referanslarını kullanın ve yerel Codex semantiklerini model kapsamlı `agentRuntime.id: "codex"` ile koruyarak `codex/*` ve `openai-codex/*` referanslarını taşımak için `openclaw doctor --fix` komutunu çalıştırın. Mevcut açık standart `openai/gpt-5.5` seçimleri yükseltilmez.

```json5
{
  plugins: { entries: { codex: { enabled: true } } },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.6-sol" },
    },
  },
}
```

```json5
{
  models: {
    providers: {
      openai: {
        models: [{ id: "gpt-5.5", contextTokens: 160000 }],
      },
    },
  },
}
```

### Abonelik tarzındaki diğer barındırılan seçenekler

<CardGroup cols={3}>
  <Card title="MiniMax" href="/tr/providers/minimax">
    MiniMax Coding Plan OAuth veya API anahtarı erişimi.
  </Card>
  <Card title="Qwen Cloud" href="/tr/providers/qwen">
    Qwen Cloud sağlayıcı yüzeyi ile Alibaba DashScope ve Coding Plan uç noktası eşlemesi.
  </Card>
  <Card title="Z.AI (GLM)" href="/tr/providers/zai">
    Z.AI Coding Plan veya genel API uç noktaları.
  </Card>
</CardGroup>

### OpenCode

- Kimlik doğrulama: `OPENCODE_API_KEY` (veya `OPENCODE_ZEN_API_KEY`)
- Zen çalışma zamanı sağlayıcısı: `opencode`
- Go çalışma zamanı sağlayıcısı: `opencode-go`
- Örnek modeller: `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.6`
- CLI: `openclaw onboard --auth-choice opencode-zen` veya `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (API anahtarı)

- Sağlayıcı: `google`
- Kimlik doğrulama: `GEMINI_API_KEY`
- İsteğe bağlı dönüşüm: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, yedek olarak `GOOGLE_API_KEY` ve `OPENCLAW_LIVE_GEMINI_KEY` (tek geçersiz kılma)
- Örnek modeller: `google/gemini-3.1-pro-preview`, `google/gemini-3.5-flash`
- Uyumluluk: `google/gemini-3.1-flash-preview` kullanan eski OpenClaw yapılandırması `google/gemini-3-flash-preview` olarak normalleştirilir
- Diğer ad: `google/gemini-3.1-pro` kabul edilir ve Google'ın canlı Gemini API kimliği olan `google/gemini-3.1-pro-preview` değerine normalleştirilir
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- Düşünme: `/think adaptive`, Google dinamik düşünmeyi kullanır. Gemini 3/3.1, sabit bir `thinkingLevel` değerini atlar; Gemini 2.5, `thinkingBudget: -1` gönderir.
- Doğrudan Gemini çalıştırmaları, sağlayıcıya özgü bir `cachedContents/...` tanıtıcısını iletmek için `agents.defaults.models["google/<model>"].params.cachedContent` (veya eski `cached_content`) değerini de kabul eder; Gemini önbellek isabetleri OpenClaw `cacheRead` olarak gösterilir

### Google Vertex ve Gemini CLI

- Sağlayıcılar: `google-vertex`, `google-gemini-cli`
- Kimlik doğrulama: Vertex, gcloud ADC kullanır; Gemini CLI kendi OAuth akışını kullanır

<Warning>
OpenClaw'daki Gemini CLI OAuth, resmî olmayan bir entegrasyondur. Bazı kullanıcılar, üçüncü taraf istemcileri kullandıktan sonra Google hesabı kısıtlamalarıyla karşılaştıklarını bildirmiştir. Devam etmeyi seçerseniz Google şartlarını inceleyin ve kritik olmayan bir hesap kullanın.
</Warning>

Gemini CLI OAuth, paketlenmiş `google` Plugin'inin bir parçası olarak sunulur.

<Steps>
  <Step title="Gemini CLI'ı yükleyin">
    <Tabs>
      <Tab title="brew">
        ```bash
        brew install gemini-cli
        ```
      </Tab>
      <Tab title="npm">
        ```bash
        npm install -g @google/gemini-cli
        ```
      </Tab>
    </Tabs>
  </Step>
  <Step title="Plugin'i etkinleştirin">
    ```bash
    openclaw plugins enable google
    ```
  </Step>
  <Step title="Oturum açın">
    ```bash
    openclaw models auth login --provider google-gemini-cli --set-default
    ```

    Varsayılan model: `google-gemini-cli/gemini-3-flash-preview`. `openclaw.json` içine istemci kimliği veya gizli anahtar **yapıştırmayın**. CLI oturum açma akışı, belirteçleri Gateway ana makinesindeki kimlik doğrulama profillerinde depolar.

  </Step>
  <Step title="Projeyi ayarlayın (gerekirse)">
    Oturum açtıktan sonra istekler başarısız olursa Gateway ana makinesinde `GOOGLE_CLOUD_PROJECT` veya `GOOGLE_CLOUD_PROJECT_ID` değerini ayarlayın.
  </Step>
</Steps>

Gemini CLI varsayılan olarak `stream-json` kullanır. OpenClaw, asistan akış
iletilerini okur ve `stats.cached` değerini `cacheRead` olarak normalleştirir; eski
`--output-format json` geçersiz kılmaları yanıt metnini hâlâ `response` üzerinden okur.

### Z.AI (GLM)

- Sağlayıcı: `zai`
- Kimlik doğrulama: `ZAI_API_KEY`
- Örnek model: `zai/glm-5.2`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - Model referansları standart `zai/*` sağlayıcı kimliğini kullanır.
  - `zai-api-key`, eşleşen Z.AI uç noktasını otomatik olarak algılar; `zai-coding-global`, `zai-coding-cn`, `zai-global` ve `zai-cn` belirli bir yüzeyi zorunlu kılar

### Vercel AI Gateway

- Sağlayıcı: `vercel-ai-gateway`
- Kimlik doğrulama: `AI_GATEWAY_API_KEY`
- Örnek modeller: `vercel-ai-gateway/anthropic/claude-opus-4.6`, `vercel-ai-gateway/moonshotai/kimi-k2.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### Paketlenmiş diğer sağlayıcı Plugin'leri

| Sağlayıcı                               | Kimlik                           | Kimlik doğrulama ortam değişkeni                       | Örnek model                                             |
| --------------------------------------- | -------------------------------- | ------------------------------------------------------ | ------------------------------------------------------ |
| Arcee                                   | `arcee`                          | `ARCEEAI_API_KEY` veya `OPENROUTER_API_KEY`            | `arcee/trinity-large-thinking`                         |
| BytePlus                                | `byteplus` / `byteplus-plan`     | `BYTEPLUS_API_KEY`                                   | `byteplus-plan/ark-code-latest`                        |
| Cerebras                                | `cerebras`                       | `CEREBRAS_API_KEY`                                   | `cerebras/zai-glm-4.7`                                 |
| Chutes                                  | `chutes`                         | `CHUTES_API_KEY` veya `CHUTES_OAUTH_TOKEN`             | `chutes/zai-org/GLM-5-TEE`                             |
| ClawRouter                              | `clawrouter`                     | `CLAWROUTER_API_KEY`                                 | `clawrouter/anthropic/claude-sonnet-4-6`               |
| Cohere                                  | `cohere`                         | `COHERE_API_KEY`                                     | `cohere/command-a-plus-05-2026`                        |
| DeepInfra                               | `deepinfra`                      | `DEEPINFRA_API_KEY`                                  | `deepinfra/deepseek-ai/DeepSeek-V4-Flash`              |
| DeepSeek                                | `deepseek`                       | `DEEPSEEK_API_KEY`                                   | `deepseek/deepseek-v4-flash`                           |
| Featherless AI                          | `featherless`                    | `FEATHERLESS_API_KEY`                                | `featherless/Qwen/Qwen3-32B`                           |
| GitHub Copilot                          | `github-copilot`                 | `COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN` | -                                                      |
| GMI Cloud                               | `gmi`                            | `GMI_API_KEY`                                        | `gmi/google/gemini-3.1-flash-lite`                     |
| Groq                                    | `groq`                           | `GROQ_API_KEY`                                       | `groq/llama-3.3-70b-versatile`                         |
| Hugging Face Inference                  | `huggingface`                    | `HUGGINGFACE_HUB_TOKEN` veya `HF_TOKEN`                | `huggingface/deepseek-ai/DeepSeek-R1`                  |
| MiniMax                                 | `minimax` / `minimax-portal`     | `MINIMAX_API_KEY` / `MINIMAX_OAUTH_TOKEN`            | `minimax/MiniMax-M3`                                   |
| Mistral                                 | `mistral`                        | `MISTRAL_API_KEY`                                    | `mistral/mistral-large-latest`                         |
| Moonshot                                | `moonshot`                       | `MOONSHOT_API_KEY`                                   | `moonshot/kimi-k2.6`                                   |
| NVIDIA                                  | `nvidia`                         | `NVIDIA_API_KEY`                                     | `nvidia/nvidia/nemotron-3-ultra-550b-a55b`             |
| NovitaAI                                | `novita`                         | `NOVITA_API_KEY`                                     | `novita/deepseek/deepseek-v3-0324`                     |
| [Ollama Cloud](/tr/providers/ollama-cloud) | `ollama-cloud`                   | `OLLAMA_API_KEY`                                     | `ollama-cloud/kimi-k2.6`                               |
| OpenRouter                              | `openrouter`                     | OpenRouter OAuth veya `OPENROUTER_API_KEY`             | `openrouter/auto`                                      |
| Qianfan                                 | `qianfan`                        | `QIANFAN_API_KEY`                                    | `qianfan/deepseek-v3.2`                                |
| Tencent TokenHub                        | `tencent-tokenhub`               | `TOKENHUB_API_KEY`                                   | `tencent-tokenhub/hy3-preview`                         |
| Together                                | `together`                       | `TOGETHER_API_KEY`                                   | `together/meta-llama/Llama-3.3-70B-Instruct-Turbo`     |
| Venice                                  | `venice`                         | `VENICE_API_KEY`                                     | -                                                      |
| Vercel AI Gateway                       | `vercel-ai-gateway`              | `AI_GATEWAY_API_KEY`                                 | `vercel-ai-gateway/anthropic/claude-opus-4.6`          |
| Volcano Engine (Doubao)                 | `volcengine` / `volcengine-plan` | `VOLCANO_ENGINE_API_KEY`                             | `volcengine-plan/ark-code-latest`                      |
| xAI                                     | `xai`                            | SuperGrok/X Premium OAuth veya `XAI_API_KEY`           | `xai/grok-4.3`                                         |
| Xiaomi                                  | `xiaomi` / `xiaomi-token-plan`   | `XIAOMI_API_KEY` / `XIAOMI_TOKEN_PLAN_API_KEY`       | `xiaomi/mimo-v2.5` / `xiaomi-token-plan/mimo-v2.5-pro` |

#### Bilinmesi yararlı özel durumlar

<AccordionGroup>
  <Accordion title="OpenRouter">
    Uygulama ilişkilendirme üstbilgilerini ve Anthropic `cache_control` işaretçilerini yalnızca doğrulanmış `openrouter.ai` rotalarında uygular. DeepSeek, Moonshot ve ZAI referansları, OpenRouter tarafından yönetilen istem önbelleğe alma için önbellek TTL'sine uygundur ancak Anthropic önbellek işaretçilerini almaz. Proxy tarzı OpenAI uyumlu bir yol olduğundan yalnızca yerel OpenAI'ye özgü biçimlendirmeyi (`serviceTier`, Responses `store`, istem önbelleği ipuçları, OpenAI akıl yürütme uyumluluğu) atlar. Gemini destekli referanslar yalnızca proxy-Gemini düşünce imzası temizliğini korur.
  </Accordion>
  <Accordion title="Kilo Gateway">
    Gemini destekli referanslar aynı proxy-Gemini temizleme yolunu izler; `kilocode/kilo-auto/balanced` ve proxy akıl yürütmeyi desteklemeyen diğer referanslar, proxy akıl yürütme eklemesini atlar.
  </Accordion>
  <Accordion title="MiniMax">
    API anahtarıyla ilk kurulum, açık M3 ve M2.7 sohbet modeli tanımlarını yazar; görüntü anlama, Plugin'in sahip olduğu `MiniMax-VL-01` medya sağlayıcısında kalır.
  </Accordion>
  <Accordion title="NVIDIA">
    Model kimlikleri bir `nvidia/<vendor>/<model>` ad alanı kullanır (örneğin `nvidia/nvidia/nemotron-...`); seçiciler değişmez `<provider>/<model-id>` bileşimini korurken API'ye gönderilen kurallı anahtar tek önekli kalır.
  </Accordion>
  <Accordion title="xAI">
    xAI Responses yolunu kullanır. Önerilen yol SuperGrok/X Premium OAuth'tır; API anahtarları `XAI_API_KEY` veya Plugin yapılandırması üzerinden çalışmaya devam eder ve Grok `web_search`, API anahtarı geri dönüşünden önce aynı kimlik doğrulama profilini yeniden kullanır. Grok 4.5, kullanılabildiği yerlerde sohbet, kodlama ve etmen tabanlı çalışmalar için seçilebilir; `grok-4.3` bölgesel olarak güvenli paketlenmiş varsayılan olmaya devam eder. Eski `/fast` ve `params.fastMode: true` yapılandırmaları xAI'ın Grok 4.3 uyumluluk yönlendirmeleri üzerinden çözümlenmeye devam eder ancak yeni yapılandırmalarda doğrudan güncel bir model seçilmelidir. `tool_stream` varsayılan olarak açıktır; `agents.defaults.models["xai/<model>"].params.tool_stream=false` üzerinden devre dışı bırakın.
  </Accordion>
</AccordionGroup>

## `models.providers` üzerinden sağlayıcılar (özel/temel URL)

**Özel** sağlayıcılar veya OpenAI/Anthropic uyumlu proxy'ler eklemek için `models.providers` (veya `models.json`) kullanın.

Aşağıdaki paketlenmiş sağlayıcı Plugin'lerinin çoğu zaten varsayılan bir katalog yayımlar. Açık `models.providers.<id>` girdilerini yalnızca varsayılan temel URL'yi, üstbilgileri veya model listesini geçersiz kılmak istediğinizde kullanın.

Paketlenmiş ve katalogca bilinen rotalar, `compat` yeteneklerini sahip olan sağlayıcı Plugin'inden alır. Bir yapılandırma `compat` bloğu, sözleşmesini doğruladığınız özel bir sağlayıcı/model veya farklı bir `api`/`baseUrl` rotası içindir; [özel sağlayıcı yetenek kılavuzuna](/tr/gateway/config-tools#custom-provider-capability-declarations) bakın. Doctor, yalnızca kataloğu yineleyen eski değerleri kaldırır ve farklı değerleri operatör incelemesi için görünür bırakır.

Gateway model yetenek kontrolleri açık `models.providers.<id>.models[]` meta verilerini de okur. Özel veya proxy bir model görüntüleri kabul ediyorsa WebChat ve Node kaynaklı ek yollarının görüntüleri yalnızca metin medya referansları yerine yerel model girdileri olarak iletmesi için bu modelde `input: ["text", "image"]` ayarını yapın.

`agents.defaults.models["provider/model"]`, etmenlerin takma adlarını ve model başına meta verilerini denetler. Tek başına ne geçersiz kılmaları sınırlar ne de yeni bir çalışma zamanı modeli kaydeder. Özel sağlayıcı modelleri için en azından eşleşen `id` ile birlikte `models.providers.<provider>.models[]` de ekleyin; geçersiz kılma kısıtlaması istediğinizde `agents.defaults.modelPolicy.allow` öğesini ayrıca kullanın.

### Moonshot AI (Kimi)

İlk kurulumdan önce `@openclaw/moonshot-provider` yükleyin. Açık bir `models.providers.moonshot` girdisini yalnızca temel URL'yi veya model meta verilerini geçersiz kılmanız gerektiğinde ekleyin:

- Sağlayıcı: `moonshot`
- Kimlik doğrulama: `MOONSHOT_API_KEY`
- Örnek model: `moonshot/kimi-k3`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` veya `openclaw onboard --auth-choice moonshot-api-key-cn`

Kimi model kimlikleri:

[//]: # "moonshot-kimi-k2-model-refs:start"

- `moonshot/kimi-k2.6`
- `moonshot/kimi-k3`
- `moonshot/kimi-k2.7-code`
- `moonshot/kimi-k2.7-code-highspeed`
- `moonshot/kimi-k2.5`

[//]: # "moonshot-kimi-k2-model-refs:end"

```json5
{
  agents: {
    defaults: { model: { primary: "moonshot/kimi-k2.6" } },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.6", name: "Kimi K2.6" }],
      },
    },
  },
}
```

Eksiksiz kurulum kılavuzu için [Moonshot AI (Kimi + Kimi Coding)](/tr/providers/moonshot) sayfasına bakın.

### Kimi Coding

Kimi Coding, Moonshot AI'ın Anthropic uyumlu uç noktasını kullanır:

- Sağlayıcı: `kimi`
- Kimlik doğrulama: `KIMI_API_KEY`
- Kimi K3: `kimi/k3` (256K) veya `kimi/k3[1m]` (1M plan)
- Kimi Code: `kimi/kimi-for-coding`
- Kimi Code HighSpeed: `kimi/kimi-for-coding-highspeed`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-for-coding" } },
  },
}
```

Eski `kimi/kimi-code` ve `kimi/k2p5`, uyumluluk modeli kimlikleri olarak kabul edilmeye devam eder ve Kimi'nin kararlı API modeli kimliğine normalleştirilir.

### Volcano Engine (Doubao)

Volcano Engine (火山引擎), Çin'de Doubao ve diğer modellere erişim sağlar.

- Sağlayıcı: `volcengine` (kodlama: `volcengine-plan`)
- Kimlik doğrulama: `VOLCANO_ENGINE_API_KEY`
- Örnek model: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

İlk kurulum varsayılan olarak kodlama yüzeyini kullanır ancak genel `volcengine/*` kataloğu da aynı anda kaydedilir.

İlk kurulum/yapılandırma model seçicilerinde Volcengine kimlik doğrulama seçeneği hem `volcengine/*` hem de `volcengine-plan/*` satırlarını tercih eder. Bu modeller henüz yüklenmemişse OpenClaw, boş bir sağlayıcı kapsamlı seçici göstermek yerine filtrelenmemiş kataloğa geri döner.

<Tabs>
  <Tab title="Standart modeller">
    - `volcengine/doubao-seed-1-8-251228` (Doubao Seed 1.8)
    - `volcengine/doubao-seed-code-preview-251028`
    - `volcengine/kimi-k2-5-260127` (Kimi K2.5)
    - `volcengine/glm-4-7-251222` (GLM 4.7)
    - `volcengine/deepseek-v3-2-251201` (DeepSeek V3.2)

  </Tab>
  <Tab title="Kodlama modelleri (volcengine-plan)">
    - `volcengine-plan/ark-code-latest`
    - `volcengine-plan/doubao-seed-code`

  </Tab>
</Tabs>

### BytePlus (Uluslararası)

BytePlus ARK, uluslararası kullanıcıların Volcano Engine ile aynı modellere erişmesini sağlar.

- Sağlayıcı: `byteplus` (kodlama: `byteplus-plan`)
- Kimlik doğrulama: `BYTEPLUS_API_KEY`
- Örnek model: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

İlk katılım varsayılan olarak kodlama yüzeyini kullanır, ancak genel `byteplus/*` kataloğu da aynı anda kaydedilir.

İlk katılım/yapılandırma model seçicilerinde BytePlus kimlik doğrulama seçeneği hem `byteplus/*` hem de `byteplus-plan/*` satırlarını tercih eder. Bu modeller henüz yüklenmemişse OpenClaw, sağlayıcı kapsamlı boş bir seçici göstermek yerine filtrelenmemiş kataloğa geri döner.

<Tabs>
  <Tab title="Standart modeller">
    - `byteplus/seed-1-8-251228` (Seed 1.8)
    - `byteplus/kimi-k2-5-260127` (Kimi K2.5)
    - `byteplus/glm-4-7-251222` (GLM 4.7)

  </Tab>
  <Tab title="Kodlama modelleri (byteplus-plan)">
    - `byteplus-plan/ark-code-latest`
    - `byteplus-plan/kimi-k2.5`
    - `byteplus-plan/glm-4.7`

  </Tab>
</Tabs>

### Synthetic

Synthetic, `synthetic` sağlayıcısı üzerinden Anthropic uyumlu modeller sunar:

- Sağlayıcı: `synthetic`
- Kimlik doğrulama: `SYNTHETIC_API_KEY`
- Örnek model: `synthetic/hf:MiniMaxAI/MiniMax-M3`
- CLI: `openclaw onboard --auth-choice synthetic-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" } },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "hf:MiniMaxAI/MiniMax-M3", name: "MiniMax M3" }],
      },
    },
  },
}
```

### MiniMax

MiniMax, özel uç noktalar kullandığından `models.providers` üzerinden yapılandırılır:

- MiniMax OAuth (Küresel): `--auth-choice minimax-global-oauth`
- MiniMax OAuth (Çin): `--auth-choice minimax-cn-oauth`
- MiniMax API anahtarı (Küresel): `--auth-choice minimax-global-api`
- MiniMax API anahtarı (Çin): `--auth-choice minimax-cn-api`
- Kimlik doğrulama: `minimax` için `MINIMAX_API_KEY`; `minimax-portal` için `MINIMAX_OAUTH_TOKEN` veya `MINIMAX_API_KEY`

Kurulum ayrıntıları, model seçenekleri ve yapılandırma parçacıkları için [/providers/minimax](/tr/providers/minimax) sayfasına bakın.

<Note>
MiniMax'in Anthropic uyumlu akış yolunda OpenClaw, açıkça ayarlamadığınız sürece M2.x ailesi için düşünmeyi varsayılan olarak devre dışı bırakır; MiniMax-M3 (ve M3.x) ise varsayılan olarak sağlayıcının belirtilmemiş/uyarlanabilir düşünme yolunda kalır. `/fast on`, `MiniMax-M2.7` değerini `MiniMax-M2.7-highspeed` olarak yeniden yazar.
</Note>

Plugin tarafından yönetilen yetenek ayrımı:

- Metin/sohbet varsayılanları `minimax/MiniMax-M3` üzerinde kalır
- Görsel oluşturma `minimax/image-01` veya `minimax-portal/image-01` kullanır
- Görsel anlama, her iki MiniMax kimlik doğrulama yolunda da Plugin tarafından yönetilen `MiniMax-VL-01` kullanır
- Web araması `minimax` sağlayıcı kimliğinde kalır

### LM Studio

LM Studio, yerel API'yi kullanan paketlenmiş bir sağlayıcı Plugin'i olarak sunulur:

- Sağlayıcı: `lmstudio`
- Kimlik doğrulama: `LM_API_TOKEN`
- Varsayılan çıkarım temel URL'si: `http://localhost:1234/v1`

Ardından bir model ayarlayın (`http://localhost:1234/api/v1/models` tarafından döndürülen kimliklerden biriyle değiştirin):

```json5
{
  agents: {
    defaults: { model: { primary: "lmstudio/openai/gpt-oss-20b" } },
  },
}
```

OpenClaw, keşif ve otomatik yükleme için LM Studio'nun yerel `/api/v1/models` ve `/api/v1/models/load` özelliklerini, varsayılan olarak çıkarım içinse `/v1/chat/completions` özelliğini kullanır. LM Studio'nun JIT yükleme, TTL ve otomatik çıkarma işlevlerinin model yaşam döngüsünü yönetmesini istiyorsanız `models.providers.lmstudio.params.preload: false` ayarını yapın. Kurulum ve sorun giderme için [/providers/lmstudio](/tr/providers/lmstudio) sayfasına bakın.

### Ollama

Ollama, paketlenmiş bir sağlayıcı Plugin'i olarak sunulur ve Ollama'nın yerel API'sini kullanır:

- Sağlayıcı: `ollama`
- Kimlik doğrulama: Gerekli değil (yerel sunucu)
- Örnek model: `ollama/llama3.3`
- Kurulum: [https://ollama.com/download](https://ollama.com/download)

```bash
# Ollama'yı kurun, ardından bir model çekin:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

`OLLAMA_API_KEY` ile etkinleştirdiğinizde Ollama, yerel olarak `http://127.0.0.1:11434` adresinde algılanır ve paketlenmiş sağlayıcı Plugin'i Ollama'yı doğrudan `openclaw onboard` ile model seçiciye ekler. İlk katılım, bulut/yerel mod ve özel yapılandırma için [/providers/ollama](/tr/providers/ollama) sayfasına bakın.

### vLLM

vLLM, yerel/kendi barındırdığınız OpenAI uyumlu sunucular için paketlenmiş bir sağlayıcı Plugin'i olarak sunulur:

- Sağlayıcı: `vllm`
- Kimlik doğrulama: İsteğe bağlı (sunucunuza bağlıdır)
- Varsayılan temel URL: `http://127.0.0.1:8000/v1`

Yerel otomatik keşfi etkinleştirmek için (sunucunuz kimlik doğrulamasını zorunlu kılmıyorsa herhangi bir değer kullanılabilir):

```bash
export VLLM_API_KEY="vllm-local"
```

Ardından bir model ayarlayın (`/v1/models` tarafından döndürülen kimliklerden biriyle değiştirin):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

Ayrıntılar için [/providers/vllm](/tr/providers/vllm) sayfasına bakın.

### SGLang

SGLang, kendi barındırdığınız hızlı OpenAI uyumlu sunucular için paketlenmiş bir sağlayıcı Plugin'i olarak sunulur:

- Sağlayıcı: `sglang`
- Kimlik doğrulama: İsteğe bağlı (sunucunuza bağlıdır)
- Varsayılan temel URL: `http://127.0.0.1:30000/v1`

Yerel otomatik keşfi etkinleştirmek için (sunucunuz kimlik doğrulamasını zorunlu kılmıyorsa herhangi bir değer kullanılabilir):

```bash
export SGLANG_API_KEY="sglang-local"
```

Ardından bir model ayarlayın (`/v1/models` tarafından döndürülen kimliklerden biriyle değiştirin):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

Ayrıntılar için [/providers/sglang](/tr/providers/sglang) sayfasına bakın.

### Yerel proxy'ler (LM Studio, vLLM, LiteLLM vb.)

Örnek (OpenAI uyumlu):

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "Local" } },
    },
  },
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "${LM_API_TOKEN}",
        api: "openai-completions",
        timeoutSeconds: 300,
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Varsayılan isteğe bağlı alanlar">
    Özel sağlayıcılar için `reasoning`, `input`, `cost`, `contextWindow` ve `maxTokens` isteğe bağlıdır. Bunlar belirtilmediğinde OpenClaw şu varsayılanları kullanır:

    - `reasoning: false`
    - `input: ["text"]`
    - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
    - `contextWindow: 200000`
    - `maxTokens: 8192`

    Önerilen: proxy/model sınırlarınızla eşleşen değerleri açıkça ayarlayın.

  </Accordion>
  <Accordion title="Proxy yolu biçimlendirme kuralları">
    - Yerel olmayan uç noktalardaki `api: "openai-completions"` için (ana bilgisayarı `api.openai.com` olmayan, boş olmayan herhangi bir `baseUrl`) OpenClaw, desteklenmeyen `developer` rolleri nedeniyle oluşan sağlayıcı 400 hatalarını önlemek amacıyla `compat.supportsDeveloperRole: false` değerini zorunlu kılar.
    - Proxy tarzı OpenAI uyumlu yollar, yalnızca yerel OpenAI'ye özgü istek biçimlendirmesini de atlar: `service_tier`, Responses `store`, Completions `store`, istem önbelleği ipuçları, OpenAI düşünme uyumluluğu yük biçimlendirmesi ve gizli OpenClaw atıf üstbilgileri kullanılmaz.
    - Satıcıya özgü alanlara ihtiyaç duyan OpenAI uyumlu Completions proxy'leri için, ek JSON'u giden istek gövdesiyle birleştirmek üzere `agents.defaults.models["provider/model"].params.extra_body` (veya `extraBody`) ayarını yapın.
    - vLLM sohbet şablonu denetimleri için `agents.defaults.models["provider/model"].params.chat_template_kwargs` ayarını yapın. Paketlenmiş vLLM Plugin'i, oturum düşünme düzeyi kapalıyken `vllm/nemotron-3-*` için `enable_thinking: false` ve `force_nonempty_content: true` değerlerini otomatik olarak gönderir.
    - Yavaş yerel modeller veya uzak LAN/tailnet ana bilgisayarları için `models.providers.<id>.timeoutSeconds` ayarını yapın. Bu, tüm ajan çalışma zamanı zaman aşımını artırmadan bağlantı, üstbilgiler, gövde akışı ve korumalı getirme işleminin toplam iptali dâhil olmak üzere sağlayıcı modeli HTTP isteği işlemeyi uzatır. `agents.defaults.timeoutSeconds` veya çalıştırmaya özgü bir zaman aşımı daha düşükse bu üst sınırı da yükseltin; sağlayıcı zaman aşımları tüm çalıştırmayı uzatamaz.
    - Model sağlayıcısı HTTP çağrıları, yalnızca yapılandırılmış sağlayıcı `baseUrl` ana bilgisayar adı için `198.18.0.0/15` ve `fc00::/7` içindeki Surge, Clash ve sing-box sahte IP DNS yanıtlarına izin verir. Özel/yerel sağlayıcı uç noktaları ayrıca geri döngü, LAN ve tailnet ana bilgisayarları dâhil olmak üzere korumalı model istekleri için tam olarak yapılandırılmış `scheme://host:port` kaynağına güvenir. Bu yeni bir yapılandırma seçeneği değildir; yapılandırdığınız `baseUrl`, istek politikasını yalnızca bu kaynak için genişletir. Sahte IP ana bilgisayar adı izni ve tam kaynak güveni bağımsız mekanizmalardır. Diğer özel, geri döngü, bağlantı-yerel, meta veri hedefleri ve farklı bağlantı noktaları yine açık bir `models.providers.<id>.request.allowPrivateNetwork: true` etkinleştirmesi gerektirir. Tam kaynak güvenini devre dışı bırakmak için `models.providers.<id>.request.allowPrivateNetwork: false` ayarını yapın.
    - `baseUrl` boşsa veya belirtilmemişse OpenClaw, varsayılan OpenAI davranışını korur (bu, `api.openai.com` değerine çözümlenir).
    - Güvenlik amacıyla, açıkça belirtilen bir `compat.supportsDeveloperRole: true` değeri yerel olmayan `openai-completions` uç noktalarında yine geçersiz kılınır.
    - Doğrudan olmayan uç noktalardaki `api: "anthropic-messages"` için (standart `anthropic` dışındaki herhangi bir sağlayıcı veya ana bilgisayarı genel bir `api.anthropic.com` uç noktası olmayan özel bir `models.providers.anthropic.baseUrl`) OpenClaw, özel Anthropic uyumlu proxy'lerin desteklenmeyen beta bayraklarını reddetmemesi için `claude-code-20250219`, `interleaved-thinking-2025-05-14` ve OAuth işaretçileri gibi örtük Anthropic beta üstbilgilerini bastırır. Proxy'niz belirli beta özelliklerine ihtiyaç duyuyorsa `models.providers.<id>.headers["anthropic-beta"]` değerini açıkça ayarlayın.

  </Accordion>
</AccordionGroup>

## CLI örnekleri

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

Ayrıca tam yapılandırma örnekleri için [Yapılandırma](/tr/gateway/configuration) sayfasına bakın.

## İlgili

- [Yapılandırma referansı](/tr/gateway/config-agents#agent-defaults) - model yapılandırma anahtarları
- [Model yük devri](/tr/concepts/model-failover) - geri dönüş zincirleri ve yeniden deneme davranışı
- [Modeller](/tr/concepts/models) - model yapılandırması ve takma adlar
- [Sağlayıcılar](/tr/providers) - sağlayıcıya özgü kurulum kılavuzları
