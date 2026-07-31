---
read_when:
    - Önbelleği koruyarak istem token maliyetlerini azaltmak istiyorsunuz
    - Çok aracılı kurulumlarda aracı başına önbellek davranışına ihtiyacınız vardır
    - Heartbeat ve cache-ttl budamasını birlikte ayarlıyorsunuz
summary: İstem önbellekleme ayarları, birleştirme sırası, sağlayıcı davranışı ve ince ayar kalıpları
title: İstem önbelleğe alma
x-i18n:
    generated_at: "2026-07-27T00:16:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 99dfd3d226d37014110adf16818051236114dcb0277e9b4d13eaced0f1fc03aa
    source_path: reference/prompt-caching.md
    workflow: 16
---

Prompt önbelleğe alma, bir model sağlayıcısının değişmemiş bir istem önekini (sistem/geliştirici talimatları, araç tanımları, diğer kararlı bağlam) her istekte yeniden işlemek yerine dönüşler arasında yeniden kullanmasını sağlar. Bu, tekrarlanan bağlama sahip uzun süreli oturumlarda token maliyetini ve gecikmeyi azaltır.

OpenClaw, yukarı akış API'sinin bu sayaçları sunduğu her yerde sağlayıcı kullanımını `cacheRead` ve `cacheWrite` olarak normalleştirir. Kullanım özetleri (`/status` ve benzerleri), canlı oturum anlık görüntüsünde önbellek sayaçları bulunmadığında son transkript kullanım girdisine geri döner; sıfır olmayan canlı değer her zaman geri dönüş değerine üstün gelir.

Sağlayıcı referansları:

- [Anthropic istem önbelleğe alma](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [OpenAI istem önbelleğe alma](https://developers.openai.com/api/docs/guides/prompt-caching)

## Temel ayarlar

### `cacheRetention`

Değerler: `"none" | "short" | "long"`. Genel varsayılan olarak, model başına ve ajan başına yapılandırılabilir.
`"standard"` bir takma ad değildir; sağlayıcının varsayılan önbellek penceresi için `"short"` kullanın. Geçersiz değerler bir uyarıyla yok sayılır.

```yaml
agents:
  defaults:
    params:
      cacheRetention: "long" # none | short | long
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "short" # bu model için genel varsayılanı geçersiz kılar
  list:
    - id: "alerts"
      params:
        cacheRetention: "none" # bu ajan için her iki varsayılanı da geçersiz kılar
```

Birleştirme sırası (sonraki üstün gelir):

1. `agents.defaults.params` - tüm modeller için genel varsayılan
2. `agents.defaults.models["provider/model"].params` - model başına geçersiz kılma
3. `agents.entries.*.params` - ajan kimliğiyle eşleştirilen ajan başına geçersiz kılma

Kaynak: `src/agents/embedded-agent-runner/extra-params.ts` (`resolveExtraParams`).

### `contextPruning.mode: "cache-ttl"`

Önbellek TTL penceresi dolduktan sonra eski araç sonucu bağlamını budar; böylece boşta kalma sonrasındaki bir istek aşırı büyük geçmişi yeniden önbelleğe almaz.

```yaml
agents:
  defaults:
    contextPruning:
      mode: "cache-ttl"
      ttl: "1h"
```

Tüm davranış için [Oturum budama](/tr/concepts/session-pruning) bölümüne bakın.

### Heartbeat ile sıcak tutma

Heartbeat, önbellek pencerelerini sıcak tutabilir ve boşta kalma aralıklarından sonra yinelenen önbellek yazımlarını azaltabilir. Genel olarak (`agents.defaults.heartbeat`) veya ajan başına (`agents.entries.*.heartbeat`) yapılandırılabilir.

```yaml
agents:
  defaults:
    heartbeat:
      every: "55m"
```

## Sağlayıcı davranışı

### Anthropic (doğrudan API ve Vertex AI)

- `cacheRetention`, `anthropic` ve `anthropic-vertex` sağlayıcıları için ve `cacheRetention` açıkça ayarlandığında `amazon-bedrock` üzerindeki Claude modelleri ile özel `anthropic-messages` uyumlu uç noktalar için desteklenir.
- Ayarlanmadığında OpenClaw, doğrudan Anthropic için `cacheRetention: "short"` değerini başlangıç olarak belirler (yalnızca `anthropic` ve `anthropic-vertex` sağlayıcıları; diğer Anthropic ailesi rotaları açık bir değer gerektirir).
- Yerel Anthropic Messages yanıtları, `cacheRead` ve `cacheWrite` değerlerine eşlenen `cache_read_input_tokens` ve `cache_creation_input_tokens` alanlarını sunar.
- `cacheRetention: "short"`, varsayılan 5 dakikalık geçici önbelleğe eşlenir. `cacheRetention: "long"`, açıkça ayarlandığında 1 saatlik TTL'yi (`cache_control: { type: "ephemeral", ttl: "1h" }`) talep eder. Örtük/ortam değişkeniyle belirlenen uzun saklama (`OPENCLAW_CACHE_RETENTION=long` ve açık bir `cacheRetention` olmadan), yalnızca `api.anthropic.com` veya Vertex AI (`aiplatform.googleapis.com` / `*-aiplatform.googleapis.com`) ana bilgisayarlarında 1 saatlik TTL'ye yükseltilir; diğer ana bilgisayarlar 5 dakikalık önbelleği korur.

Kaynak: `packages/ai/src/transports/anthropic-payload-policy.ts` (`resolveAnthropicEphemeralCacheControl`, `isLongTtlEligibleEndpoint`).

### OpenAI (doğrudan API)

- İstem önbelleğe alma, desteklenen güncel modellerde otomatiktir; OpenClaw blok düzeyinde önbellek işaretçileri eklemez.
- OpenClaw, önbellek yönlendirmesini dönüşler arasında kararlı tutmak için `prompt_cache_key` gönderir. Doğrudan `api.openai.com` ana bilgisayarları bunu otomatik olarak alır. OpenAI uyumlu proxy'lerin (oMLX, llama.cpp, özel uç noktalar) katılım sağlaması için model yapılandırmasında `compat.supportsPromptCacheKey: true` gerekir; bu, bir proxy için hiçbir zaman otomatik olarak algılanmaz.
- `prompt_cache_retention: "24h"`, yalnızca `cacheRetention: "long"` seçildiğinde ve çözümlenen uç nokta hem önbellek anahtarını hem de uzun saklamayı desteklediğinde eklenir (`compat.supportsLongCacheRetention`, varsayılan olarak true; Together AI ve Cloudflare uyumluluk profilleri bunu devre dışı bırakır). `cacheRetention: "none"` her iki alanı da engeller.
- Önbellek isabetleri, `cacheRead` değerine eşlenen `usage.prompt_tokens_details.cached_tokens` (Chat Completions) veya `input_tokens_details.cached_tokens` (Responses API) üzerinden görünür.
- Responses API yükleri ayrıca `cacheWrite` değerine eşlenen ve modelin önbellek yazma tarifesiyle fiyatlandırılan `input_tokens_details.cache_write_tokens` alanını sunabilir; alanı atlayan Responses yükleri `cacheWrite` değerini `0` olarak tutar. OpenAI'ın Chat Completions API'si bir `cache_write_tokens` sayacını belgelemez veya üretmez; ancak OpenClaw, ayrı bir yazma sayısı bildiren OpenRouter uyumlu ve DeepSeek tarzı proxy'ler için burada yine de `prompt_tokens_details.cache_write_tokens` alanını okur.
- Uygulamada OpenAI, Anthropic'in hareketli tam geçmiş yeniden kullanımından çok başlangıç öneki önbelleği gibi davranır; aşağıdaki [OpenAI canlı beklentileri](#openai-live-expectations) bölümüne bakın.

### Amazon Bedrock

- Anthropic Claude model referansları (`amazon-bedrock/*anthropic.claude*` ile AWS sistem çıkarım profili önekleri `us.`/`eu.`/`global.anthropic.claude*`), açık `cacheRetention` aktarımını destekler.
- Anthropic dışındaki Bedrock modelleri (örneğin `amazon.nova-*`), yapılandırılmış herhangi bir `cacheRetention` değerinden bağımsız olarak çalışma zamanında önbellek saklama olmadan çözümlenir.
- Opak Bedrock uygulama çıkarım profili ARN'leri (`claude` içermeyen profil kimlikleri), model ailesi yalnızca ARN'den çıkarılamadığı için `cacheRetention` açıkça ayarlanmadıkça önbellek saklama olmadan çözümlenir.

### OpenRouter

`openrouter/anthropic/*` model referansları için OpenClaw, sistem/geliştirici istem bloklarına Anthropic `cache_control` işaretçilerini ekler; ancak bunu yalnızca istek doğrulanmış bir OpenRouter rotasını (`openrouter` varsayılan uç noktasında veya `openrouter.ai` olarak çözümlenen herhangi bir sağlayıcı/temel URL) hedeflemeye devam ettiğinde yapar. Modelin rastgele bir OpenAI uyumlu proxy URL'sine yönlendirilmesi bu eklemeyi durdurur.

`contextPruning.mode: "cache-ttl"`; `openrouter/anthropic/*`, `openrouter/deepseek/*`, `openrouter/moonshot/*`, `openrouter/moonshotai/*` ve `openrouter/zai/*` model referansları için kullanılabilir; çünkü bu rotalar, OpenClaw'ın eklediği işaretçilere ihtiyaç duymadan sağlayıcı tarafında istem önbelleğe almayı yönetir.

Kaynak: `extensions/openrouter/index.ts` (`OPENROUTER_CACHE_TTL_MODEL_PREFIXES`).

OpenRouter'da DeepSeek önbelleğinin oluşturulması en iyi çaba esasına dayanır ve birkaç saniye sürebilir; hemen gönderilen bir takip isteği hâlâ `cached_tokens: 0` gösterebilir. Kısa bir gecikmeden sonra aynı önekli isteği yineleyerek ve önbellek isabeti sinyali olarak `usage.prompt_tokens_details.cached_tokens` kullanarak doğrulayın.

### Google Gemini (doğrudan API)

- Doğrudan Gemini aktarımı (`api: "google-generative-ai"`), önbellek isabetlerini `cacheRead` değerine eşlenen yukarı akış `cachedContentTokenCount` üzerinden bildirir.
- Uygun model aileleri: `gemini-2.5*` ve `gemini-3*` (bu önek eşleşmesinin dışındaki Live/önizleme varyantları hariçtir; örneğin `gemini-live-2.5-flash-preview`).
- Uygun bir modelde `cacheRetention` ayarlandığında OpenClaw, sistem istemi için bir `cachedContents` kaynağını otomatik olarak oluşturur, yeniden kullanır ve yeniler; elle önbelleğe alınmış içerik tanıtıcısı gerekmez. TTL, `cacheRetention: "short"` için `300s` ve `"long"` için `3600s` değeridir.
- Önceden mevcut bir Gemini önbelleğe alınmış içerik tanıtıcısını `params.cachedContent` (veya eski `params.cached_content`) olarak aktarmaya devam edebilirsiniz; açık bir tanıtıcı, otomatik önbellek yönetimi yolunu tamamen atlar.
- Bu, Anthropic/OpenAI istem öneki önbelleğe almasından ayrıdır: OpenClaw, satır içi önbellek işaretçileri eklemek yerine Gemini için sağlayıcıya özgü bir `cachedContents` kaynağını yönetir.

Kaynak: `src/agents/embedded-agent-runner/google-prompt-cache.ts`.

### CLI çalıştırma düzeneği sağlayıcıları (Claude Code, Gemini CLI)

JSONL kullanım olayları (`jsonlDialect: "claude-stream-json"` veya `"gemini-stream-json"`) üreten CLI arka uçları, `cacheRead` değerine eşlenen düz bir `cached` sayacı da dahil olmak üzere çeşitli alan adı varyantlarını tanıyan ortak bir kullanım ayrıştırıcısından geçer. CLI'ın JSON yükü doğrudan bir girdi token alanını atladığında OpenClaw bunu `input_tokens - cached` olarak türetir. Bu yalnızca kullanım normalleştirmesidir; CLI aracılığıyla çalıştırılan bu modeller için Anthropic/OpenAI tarzı istem önbelleği işaretçileri oluşturmaz.

Kaynak: `src/agents/cli-output.ts` (`toCliUsage`).

### Diğer sağlayıcılar

Bir sağlayıcı yukarıdaki önbellek modlarından hiçbirini desteklemiyorsa `cacheRetention` etkisizdir.

## Sistem istemi önbellek sınırı

OpenClaw, sistem istemini dahili bir önbellek öneki sınırında **kararlı önek** ve **değişken sonek** olarak ayırır. Sınırın üzerindeki içerik (araç tanımları, Skills meta verileri, çalışma alanı dosyaları), dönüşler arasında bayt düzeyinde aynı kalacak şekilde sıralanır. Sınırın altındaki içerik (örneğin `HEARTBEAT.md`, çalışma zamanı zaman damgaları ve dönüş başına diğer meta veriler), önbelleğe alınmış öneki geçersiz kılmadan değişebilir.

Temel tasarım tercihleri:

- Kararlı çalışma alanı proje bağlamı dosyaları, Heartbeat değişimleri kararlı öneki bozmasın diye `HEARTBEAT.md` öncesinde sıralanır.
- Sınır; Anthropic ailesi, OpenAI ailesi, Google ve CLI aktarım biçimlendirmesinin tamamında geçerlidir; böylece desteklenen tüm sağlayıcılar aynı önek kararlılığından yararlanır.
- Codex Responses ve Anthropic Vertex istekleri, önbelleğin yeniden kullanımı sağlayıcıların gerçekte aldıklarıyla uyumlu kalsın diye sınır duyarlı önbellek biçimlendirmesi üzerinden yönlendirilir.
- Sistem istemi parmak izleri (boşluklar, satır sonları, kanca tarafından eklenen bağlam, çalışma zamanı yetenek sıralaması) normalleştirilir; böylece anlamsal olarak değişmemiş istemler dönüşler arasında önbelleği paylaşır.

Bir yapılandırma veya çalışma alanı değişikliğinden sonra beklenmedik `cacheWrite` artışları görürseniz değişikliğin önbellek sınırının üstünde mi altında mı yer aldığını kontrol edin. Değişken içeriği sınırın altına taşımak (veya kararlı hâle getirmek) genellikle sorunu çözer.

## OpenClaw önbellek kararlılığı korumaları

- Birlikte sunulan MCP araç katalogları, araç kaydından önce belirlenimci biçimde (önce sunucu adına, ardından araç adına göre) sıralanır; böylece `listTools()` sıra değişiklikleri araçlar bloğunda değişime yol açıp istem önbelleği öneklerini bozmaz.
- Kalıcı görüntü bloklarına sahip eski oturumlar, **en son tamamlanmış 3 dönüşü** olduğu gibi korur (yalnızca görüntü içerenleri değil, tamamlanmış tüm dönüşleri sayarak). Görüntü ağırlıklı takip isteklerinin büyük ve eski yükleri tekrar tekrar göndermemesi için daha eski, önceden işlenmiş görüntü blokları bir metin işaretçisiyle değiştirilir.

## Ayarlama kalıpları

### Karma trafik (önerilen varsayılan)

Ana ajanınızda uzun ömürlü bir temel çizgiyi koruyun, ani yoğunluk gösteren bildirim ajanlarında önbelleğe almayı devre dışı bırakın:

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m"
    - id: "alerts"
      params:
        cacheRetention: "none"
```

### Önce maliyet temel çizgisi

- Temel `cacheRetention: "short"` değerini ayarlayın.
- `contextPruning.mode: "cache-ttl"` özelliğini etkinleştirin.
- Heartbeat aralığını yalnızca sıcak önbelleklerden yararlanan ajanlar için TTL'nizin altında tutun.

## Canlı regresyon testleri

OpenClaw; yinelenen önekleri, araç dönüşlerini, görüntü dönüşlerini, MCP tarzı araç transkriptlerini ve önbelleksiz bir Anthropic denetimini kapsayan tek bir birleşik canlı önbellek regresyon kapısı çalıştırır.

- `src/agents/live-cache-regression.live.test.ts`
- `src/agents/live-cache-regression-runner.ts`
- `src/agents/live-cache-regression-baseline.ts`

Şununla çalıştırın:

```sh
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache
```

Temel dosya, en son gözlemlenen canlı değerlerin yanı sıra testin karşılaştırma için kullandığı sağlayıcıya özgü regresyon alt sınırlarını depolar. Her çalıştırma, önceki önbellek durumunun geçerli örneklemi etkilememesi için çalıştırmaya özgü yeni oturum kimlikleri ve istem ad alanları kullanır. Anthropic ve OpenAI farklı uygulama yöntemleri kullanır: Anthropic alt sınırının karşılanmaması kesin bir regresyondur (test başarısız olur), OpenAI alt sınırının karşılanmaması ise yalnızca izleme amaçlıdır (uyarı olarak kaydedilir, çalıştırmanın başarısız olmasına neden olmaz). Sağlayıcılar arasında ortak tek bir eşik kullanmazlar.

### Anthropic canlı ortam beklentileri

- `cacheWrite` aracılığıyla açık ısınma yazımları beklenir.
- Anthropic'in önbellek denetimi, önbellek kesme noktasını konuşma boyunca ilerlettiğinden, yinelenen turlarda geçmişin neredeyse tamamının yeniden kullanılması beklenir.
- Kararlı, araç, görüntü ve MCP tarzı hatların temel alt sınırları kesin regresyon kapılarıdır.

### OpenAI canlı ortam beklentileri

- Yalnızca `cacheRead` beklenir; Chat Completions üzerinde `cacheWrite`, `0` olarak kalır.
- Yinelenen turlardaki önbellek yeniden kullanımını, Anthropic tarzı ilerleyen tam geçmiş yeniden kullanımı olarak değil, sağlayıcıya özgü bir plato olarak değerlendirin.
- Alt sınırlar yalnızca izleme amaçlıdır (karşılanmaması test başarısızlığı olarak değil, uyarı olarak kaydedilir) ve `gpt-5.4-mini` üzerindeki gözlemlenen canlı davranıştan türetilmiştir:

| Senaryo              | `cacheRead` alt sınırı | İsabet oranı alt sınırı |
| -------------------- | ----------------------------: | ----------------------: |
| Kararlı ön ek        |                         4,608 |                    0.90 |
| Araç dökümü          |                         4,096 |                    0.85 |
| Görüntü dökümü       |                         3,840 |                    0.82 |
| MCP tarzı döküm      |                         4,096 |                    0.85 |

En son gözlemlenen temel değerler (`live-cache-regression-baseline.ts` kaynağından) şu düzeylere ulaştı: kararlı ön ek `cacheRead=4864`, isabet oranı `0.966`; araç dökümü `cacheRead=4608`, isabet oranı `0.896`; görüntü dökümü `cacheRead=4864`, isabet oranı `0.954`; MCP tarzı döküm `cacheRead=4608`, isabet oranı `0.891`.

Doğrulamaların farklı olmasının nedeni: Anthropic açık önbellek kesme noktaları ve ilerleyen konuşma geçmişi yeniden kullanımı sunarken, OpenAI'ın canlı trafikteki etkin yeniden kullanılabilir ön eki, tam istemden daha önce bir platoya ulaşabilir. İki sağlayıcıyı, sağlayıcılar arasında ortak tek bir yüzde eşiğiyle karşılaştırmak hatalı regresyonlar üretir.

## `diagnostics.cacheTrace` yapılandırması

```yaml
diagnostics:
  cacheTrace:
    enabled: true
    filePath: "~/.openclaw/logs/cache-trace.jsonl" # isteğe bağlı
    includeMessages: false # varsayılan true
    includePrompt: false # varsayılan true
    includeSystem: false # varsayılan true
```

Varsayılanlar:

| Anahtar           | Varsayılan                                  |
| ----------------- | -------------------------------------------- |
| `filePath`        | `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl` |
| `includeMessages` | `true`                                       |
| `includePrompt`   | `true`                                       |
| `includeSystem`   | `true`                                       |

### Ortam değişkeni anahtarları (tek seferlik hata ayıklama)

| Değişken                             | Etki                                        |
| ------------------------------------ | ------------------------------------------- |
| `OPENCLAW_CACHE_TRACE=1`             | Önbellek izlemeyi etkinleştirir             |
| `OPENCLAW_CACHE_TRACE_FILE=path`     | Çıktı yolunu geçersiz kılar                 |
| `OPENCLAW_CACHE_TRACE_MESSAGES=0\|1` | Tam ileti yükü yakalamayı açar veya kapatır |
| `OPENCLAW_CACHE_TRACE_PROMPT=0\|1`   | İstem metni yakalamayı açar veya kapatır    |
| `OPENCLAW_CACHE_TRACE_SYSTEM=0\|1`   | Sistem istemi yakalamayı açar veya kapatır  |

### İncelenecek noktalar

- Önbellek izleme olayları; `session:loaded`, `prompt:before`, `stream:context` ve `session:after` gibi aşamalı anlık görüntüler içeren JSONL biçimindedir.
- Tur başına önbellek token etkisi, normal kullanım yüzeylerinde görülebilir: `cacheRead` ve `cacheWrite`; `/usage tokens`, `/status`, oturum kullanım özetleri ve özel `messages.usageTemplate` düzenlerinde görünür.
- Anthropic için önbelleğe alma etkin olduğunda hem `cacheRead` hem de `cacheWrite` beklenir.
- OpenAI için önbellek isabetlerinde `cacheRead` beklenir; `cacheWrite` yalnızca bunu içeren Responses API yüklerinde doldurulur (yukarıdaki [OpenAI](#openai-direct-api) bölümüne bakın).
- OpenAI ayrıca `x-request-id`, `openai-processing-ms` ve `x-ratelimit-*` gibi izleme ve hız sınırı üst bilgileri döndürür; bunları istekleri izlemek için kullanın, ancak önbellek isabeti hesaplaması yine de üst bilgilerden değil, kullanım yükünden alınmalıdır.

## Hızlı sorun giderme

- **Çoğu turda yüksek `cacheWrite`**: değişken sistem istemi girdilerini kontrol edin; modelin/sağlayıcının önbellek ayarlarınızı desteklediğini doğrulayın.
- **Anthropic'te yüksek `cacheWrite`**: çoğunlukla önbellek kesme noktasının her istekte değişen içeriğe yerleştirildiği anlamına gelir.
- **Düşük OpenAI `cacheRead`**: kararlı ön ekin başta olduğunu, yinelenen ön ekin en az 1024 token olduğunu ve önbelleği paylaşması gereken turlar için aynı `prompt_cache_key` değerinin yeniden kullanıldığını doğrulayın.
- **`cacheRetention` etkisiz**: model anahtarının `agents.defaults.models["provider/model"]` ile eşleştiğini doğrulayın.
- **Önbellek ayarları içeren Bedrock Nova istekleri**: beklenen bir durumdur; bunlar çalışma zamanında önbellek saklama olmadan çözümlenir.

İlgili belgeler:

- [Anthropic](/tr/providers/anthropic)
- [Token kullanımı ve maliyetleri](/tr/reference/token-use)
- [Oturum budama](/tr/concepts/session-pruning)
- [Gateway yapılandırma başvurusu](/tr/gateway/configuration-reference)

## İlgili

- [Token kullanımı ve maliyetleri](/tr/reference/token-use)
- [API kullanımı ve maliyetleri](/tr/reference/api-usage-costs)
