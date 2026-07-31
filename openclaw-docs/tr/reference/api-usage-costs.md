---
read_when:
    - Ücretli API'leri hangi özelliklerin çağırabileceğini anlamak istiyorsunuz
    - Anahtarları, maliyetleri ve kullanım görünürlüğünü denetlemeniz gerekir
    - /status veya /usage maliyet raporlamasını açıklıyorsunuz
summary: Nelerin para harcayabileceğini, hangi anahtarların kullanıldığını ve kullanımın nasıl görüntüleneceğini denetleyin
title: API kullanımı ve maliyetleri
x-i18n:
    generated_at: "2026-07-27T00:16:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 22caad8b8fa168739563223b3663a04adceeef7e83576a53dc9cdf885a35750d
    source_path: reference/api-usage-costs.md
    workflow: 16
---

Ücretli sağlayıcı API'lerini çağırabilen OpenClaw özelliklerinin, her birinin kimlik bilgilerini nereden okuduğunun ve ortaya çıkan maliyetin nerede gösterildiğinin haritası.

## Maliyetlerin gösterildiği yerler

**`/status`** (oturum başına anlık görüntü)

- Geçerli oturum modelini, bağlam kullanımını ve son yanıtın token'larını gösterir.
- OpenClaw etkin model için kullanım meta verilerine ve yerel fiyatlandırmaya sahip olduğunda, açıkça fiyatlandırılmış Bedrock `aws-sdk` modelleri gibi API anahtarı kullanmayan sağlayıcılar da dahil olmak üzere son yanıt için **tahmini maliyet** ekler.
- Canlı oturum anlık görüntüsündeki veriler yetersizse `/status`, en son transkript kullanım girdisinden token/önbellek sayaçlarını ve etkin model etiketini kurtarır. Sıfırdan farklı mevcut canlı değerler transkript verilerine göre önceliklidir; depolanan toplam eksik veya daha küçükse istem boyutundaki bir transkript toplamı yine de öncelikli olabilir.

**`/usage`** (ileti başına alt bilgi)

- `/usage full`, yerel fiyatlandırma yapılandırılmış ve kullanım meta verileri mevcut olduğunda **tahmini maliyet** de dahil olmak üzere her yanıta bir kullanım alt bilgisi ekler.
- `/usage tokens` yalnızca token'ları gösterir. Abonelik türü OAuth/token ve CLI çalışma zamanları, uyumlu kullanım meta verileriyle birlikte açık bir yerel fiyat sağlamadıkları sürece yalnızca token'ları gösterir.
- `/usage cost` yerel bir maliyet özeti yazdırır; `/usage off` alt bilgiyi devre dışı bırakır.
- Gemini CLI notu: Hem `stream-json` hem de eski `json` çıktısı, kullanım verilerini `stats` altında taşır. OpenClaw, `stats.cached` değerini `cacheRead` biçimine normalleştirir ve gerektiğinde girdi token'larını `stats.input_tokens - stats.cached` değerinden türetir.

**Control UI → Usage** (oturumlar arası analiz)

- Seçilen tarih aralığı için transkriptlerden türetilen token ve tahmini maliyet toplamlarını; sağlayıcı, model, aracı, kanal ve token türüne göre dökümlerle gösterir.
- Seçilen aralığın bitiş tarihinde sona eren daha kısa takvim aralıklarını karşılaştırır. Eksik tarihler, sıfır kullanımlı takvim günleri olarak sayılır; daha yoğun bir aralık oluşturmak amacıyla atlanmaz.
- Günlük grafik ölçeğini doğrudan etiketler. `√` rozeti, karekök sıkıştırmasının düşük kullanımlı günleri görünür tuttuğu anlamına gelir.
- Bu toplamlar, bir sağlayıcı faturasını veya ömür boyu faturalandırma kaydını değil, mevcut yerel oturum geçmişini açıklar. Bazı girdiler için fiyatlandırma eksik olduğunda kullanıcı arayüzü uyarı verir.

**CLI kullanım aralıkları** (ileti başına maliyet değil, sağlayıcı kotaları)

- `openclaw status --usage` ve `openclaw channels list`, sağlayıcı **kullanım aralıklarını** `X% left` olarak gösterir.
- Mevcut kullanım aralığı sağlayıcıları: Anthropic, ClawRouter, DeepSeek, GitHub Copilot, Gemini CLI, MiniMax, OpenAI (ChatGPT/Codex OAuth/token kimlik doğrulamasını kapsar), Xiaomi ve z.ai. Sağlayıcıların ve bayrakların tam listesi için [Modeller CLI'sine](/tr/cli/models) ve [Kanallar CLI'sine](/tr/cli/channels) bakın.
- MiniMax'in ham `usage_percent` / `usagePercent` alanları kalan kotayı bildirir; bu nedenle OpenClaw bunları tersine çevirir. Mevcut olduklarında sayı tabanlı alanlar önceliklidir. Yanıt bir `model_remains` dizisi içeriyorsa OpenClaw sohbet modeli girdisini seçer, gerektiğinde zaman damgalarından aralık etiketini türetir ve plan etiketine model adını ekler.
- Kullanım kimlik doğrulaması, mevcut olduğunda sağlayıcıya özgü kancalardan gelir; aksi takdirde OpenClaw, kimlik doğrulama profillerindeki, ortamdaki veya yapılandırmadaki eşleşen OAuth/API anahtarı kimlik bilgilerini kullanır.

Ayrıntılı örnekler için [Token kullanımı ve maliyetler](/tr/reference/token-use) bölümüne bakın.

<Note>
Anthropic, yeni bir politika yayımlamadığı sürece Claude CLI'ın yeniden kullanılmasının (`claude -p` dahil) onaylanmış bir entegrasyon modeli olduğunu doğrulamıştır. Anthropic ileti başına dolar cinsinden tahmin sunmadığından `/usage full`, Claude CLI kullanımı için maliyet gösteremez.
</Note>

## Anahtarlar nasıl keşfedilir?

- **Kimlik doğrulama profilleri**: Aracı başına `auth-profiles.json` içinde depolanır.
- **Ortam değişkenleri**: Örneğin `OPENAI_API_KEY`, `BRAVE_API_KEY`, `FIRECRAWL_API_KEY`.
- **Yapılandırma**: `models.providers.*.apiKey`, `plugins.entries.*.config.webSearch.apiKey`, `plugins.entries.firecrawl.config.webFetch.apiKey`, `memory.search.*`, `talk.providers.*.apiKey`.
- **Skills**: Anahtarı skill işleminin ortamına aktarabilen `skills.entries.<name>.apiKey`.

## Anahtarları kullanarak harcama yapabilen özellikler

### Temel model yanıtları (sohbet + araçlar)

Her yanıt veya araç çağrısı, geçerli model sağlayıcısında çalışır. Bu, OpenClaw'ın yerel kullanıcı arayüzü dışında faturalandırılan abonelik türü barındırılan planlar da dahil olmak üzere kullanımın ve maliyetin birincil kaynağıdır: OpenAI Codex, Alibaba Cloud Model Studio Coding Plan, MiniMax Coding Plan, Z.AI/GLM Coding Plan ve Extra Usage etkinleştirilmiş Anthropic Claude oturum açma yolu.

Fiyatlandırma yapılandırması için [Modeller](/tr/providers/models), gösterim için [Token kullanımı ve maliyetler](/tr/reference/token-use) bölümlerine bakın.

### Medya anlama (ses/görüntü/video)

Gelen medya, yanıt işlem hattı çalışmadan önce bir sağlayıcı API'si aracılığıyla özetlenebilir veya metne dönüştürülebilir. Sağlayıcı desteği Plugin başına kaydedilir ve Plugin'ler eklendikçe değişir; güncel liste ve yapılandırma için [Medya anlama](/tr/nodes/media-understanding) bölümüne bakın.

### Görüntü ve video oluşturma

`image_generate` ve `video_generate`, kimliği doğrulanmış mevcut sağlayıcılardan uygun olana yönlendirilir. Her ikisi de `agents.defaults.mediaModels` girdileri ayarlanmamış olduğunda kimlik doğrulamasına dayalı varsayılan bir sağlayıcı çıkarımı yapabilir.

Güncel sağlayıcı listesi için [Görüntü oluşturma](/tr/tools/image-generation) ve [Video oluşturma](/tr/tools/video-generation) bölümlerine bakın.

### Bellek gömmeleri ve anlamsal arama

Anlamsal bellek araması, `memory.search.provider` uzak bir bağdaştırıcıyı adlandırdığında (örneğin `openai`, `gemini`, `voyage`, `mistral`, `deepinfra`, `github-copilot`, `amazon-bedrock`) gömme API'lerini kullanır. `memory.search.provider = "lmstudio"` veya `"ollama"`, yerel/kendi kendine barındırılan bir sunucuda çalışır ve genellikle barındırma ücreti oluşturmaz. `memory.search.provider = "local"`, API kullanmadan her şeyi cihaz üzerinde tutar. İsteğe bağlı bir `memory.search.fallback` sağlayıcısı, yerel gömme hatalarını karşılayabilir.

[Bellek](/tr/concepts/memory) bölümüne bakın.

### Web arama aracı

`web_search`, seçilen sağlayıcıya bağlı olarak kullanım ücreti oluşturabilir. Her sağlayıcı anahtarını önce bir ortam değişkeninden, ardından `plugins.entries.<id>.config.webSearch.apiKey` içinden okur:

| Sağlayıcı               | Ortam değişkenleri                                                                                                                                                     |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Brave Search           | `BRAVE_API_KEY`                                                                                                                                                        |
| DuckDuckGo             | anahtar gerektirmez; resmî değildir, HTML tabanlıdır, faturalandırma yoktur                                                                                           |
| Exa                    | `EXA_API_KEY`                                                                                                                                                          |
| Firecrawl              | `FIRECRAWL_API_KEY`                                                                                                                                                    |
| Gemini (Google Search) | `GEMINI_API_KEY`                                                                                                                                                       |
| Grok (xAI)             | xAI OAuth profili veya `XAI_API_KEY`                                                                                                                                |
| Kimi (Moonshot)        | `KIMI_API_KEY` veya `MOONSHOT_API_KEY`                                                                                                                               |
| MiniMax Search         | `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN` veya `MINIMAX_API_KEY`                                                                         |
| Ollama Web Search      | erişilebilir ve oturum açılmış bir yerel ana makine için anahtar gerektirmez; doğrudan `https://ollama.com` araması `OLLAMA_API_KEY` kullanır; kimlik doğrulamasıyla korunan ana makineler normal Ollama sağlayıcısının bearer kimlik doğrulamasını yeniden kullanır |
| Parallel               | `PARALLEL_API_KEY`                                                                                                                                                     |
| Perplexity Search API  | `PERPLEXITY_API_KEY` veya `OPENROUTER_API_KEY`                                                                                                                           |
| SearXNG                | `SEARXNG_BASE_URL`; anahtar gerektirmez/kendi kendine barındırılır, barındırma ücreti yoktur                                                                          |
| Tavily                 | `TAVILY_API_KEY`                                                                                                                                                       |

Eski `tools.web.search.*` yapılandırma yolları bir uyumluluk katmanı üzerinden yüklenmeye devam eder ancak artık önerilen yüzey değildir.

**Brave Search ücretsiz kredisi**: Her plan, aylık yenilenen $5 ücretsiz kredi içerir. Search planının maliyeti 1,000 istek başına $5 olduğundan kredi, aylık 1,000 isteği ücretsiz olarak karşılar. Beklenmeyen ücretleri önlemek için Brave panosunda bir kullanım sınırı ayarlayın.

[Web araçları](/tr/tools/web) bölümüne bakın.

### Web getirme aracı (Firecrawl)

`web_fetch`, anahtarsız başlangıç erişimiyle Firecrawl'ı çağırabilir; daha yüksek sınırlar için `FIRECRAWL_API_KEY` (veya `plugins.entries.firecrawl.config.webFetch.apiKey`) ekleyin. Firecrawl yapılandırılmamışsa araç, doğrudan getirmeye ve paketle birlikte gelen `web-readability` Plugin'ine geri döner (ücretli API yoktur). Yerel Readability ayıklamasını atlamak için `plugins.entries.web-readability.enabled` özelliğini devre dışı bırakın.

[Web araçları](/tr/tools/web) bölümüne bakın.

### Sağlayıcı kullanım anlık görüntüleri (durum/sağlık)

`openclaw status --usage` ve `openclaw models status --json`, kota aralıklarını veya kimlik doğrulama durumunu göstermek için sağlayıcı kullanım uç noktalarını çağırır. Çağrı hacmi düşüktür ancak yine de sağlayıcı API'lerine ulaşır.

[Modeller CLI'sine](/tr/cli/models) bakın.

### Compaction koruması özetlemesi

Compaction koruması, oturum geçmişini geçerli modeli kullanarak özetleyebilir; bu işlem çalıştığında sağlayıcı API'lerini çağırır.

[Oturum yönetimi ve Compaction](/tr/reference/session-management-compaction) bölümüne bakın.

### Model tarama / yoklama

`openclaw models scan`, OpenRouter modellerini yoklayabilir ve yoklama etkinleştirildiğinde `OPENROUTER_API_KEY` kullanır.

[Modeller CLI'sine](/tr/cli/models) bakın.

### Konuşma (ses)

Konuşma modu yapılandırıldığında ElevenLabs'i çağırabilir: `ELEVENLABS_API_KEY` veya `talk.providers.elevenlabs.apiKey`.

[Konuşma modu](/tr/nodes/talk) bölümüne bakın.

### Skills (üçüncü taraf API'leri)

Skills, `apiKey` değerini `skills.entries.<name>.apiKey` içinde depolayabilir. Bir skill bu anahtarı harici bir API ile kullanırsa maliyet, skill'in sağlayıcısına bağlıdır.

[Skills](/tr/tools/skills) bölümüne bakın.

## İlgili

- [Token kullanımı ve maliyetler](/tr/reference/token-use)
- [İstem önbelleğe alma](/tr/reference/prompt-caching)
- [Kullanım takibi](/tr/concepts/usage-tracking)
