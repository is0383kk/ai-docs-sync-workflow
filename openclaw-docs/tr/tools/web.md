---
read_when:
    - web_search özelliğini etkinleştirmek veya yapılandırmak istiyorsunuz
    - x_search'ü etkinleştirmek veya yapılandırmak istiyorsunuz
    - Bir arama sağlayıcısı seçmeniz gerekiyor
    - Otomatik algılamayı ve sağlayıcı seçimini anlamak istiyorsunuz
sidebarTitle: Web Search
summary: web_search, x_search ve web_fetch -- web'de arama yapın, X gönderilerinde arama yapın veya sayfa içeriğini getirin
title: Web araması
x-i18n:
    generated_at: "2026-07-26T23:08:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 997e51064b0cd08d0f30987aa038e2f4a98da22f1094974b45f59c18491bd979
    source_path: tools/web.md
    workflow: 16
---

`web_search`, yapılandırdığınız sağlayıcıyla web'de arama yapar ve
sorguya göre 15 dakika boyunca önbelleğe alınan (yapılandırılabilir) normalleştirilmiş sonuçlar döndürür. OpenClaw
ayrıca X (eski adıyla Twitter) gönderileri için `x_search` ve
hafif URL getirme için `web_fetch` araçlarını da içerir. `web_fetch` her zaman yerel olarak çalışır; sağlayıcı Grok olduğunda `web_search`
xAI Responses üzerinden yönlendirilir ve `x_search` her zaman
xAI Responses kullanır.

<Info>
  `web_search`, tarayıcı otomasyonu değil, hafif bir HTTP aracıdır.
  Yoğun JS kullanan siteler veya oturum açma işlemleri için [Web Tarayıcısı](/tr/tools/browser) aracını kullanın.
  Belirli bir URL'yi getirmek için [Web Fetch](/tr/tools/web-fetch) aracını kullanın.
</Info>

## Hızlı başlangıç

<Steps>
  <Step title="Bir sağlayıcı seçin">
    Bir sağlayıcı seçin ve gerekli tüm kurulumları tamamlayın. Bazı sağlayıcılar
    anahtarsızdır, bazılarıysa API anahtarı gerektirir. Ayrıntılar için
    aşağıdaki sağlayıcı sayfalarına bakın.
  </Step>
  <Step title="Yapılandırın">
    ```bash
    openclaw configure --section web
    ```
    Bu işlem sağlayıcıyı ve gereken kimlik bilgilerini saklar. API tabanlı
    sağlayıcılarda bunun yerine sağlayıcının ortam değişkenini (örneğin
    `BRAVE_API_KEY`) ayarlayıp bu adımı atlayabilirsiniz.
  </Step>
  <Step title="Kullanın">
    ```javascript
    await web_search({ query: "OpenClaw plugin SDK" });
    ```

    X gönderileri için:

    ```javascript
    await x_search({ query: "akşam yemeği tarifleri" });
    ```

  </Step>
</Steps>

## Sağlayıcı seçimi

<CardGroup cols={2}>
  <Card title="Brave Search" icon="shield" href="/tr/tools/brave-search">
    Parçacıklar içeren yapılandırılmış sonuçlar. `llm-context` modunu ve ülke/dil filtrelerini destekler. Ücretsiz katman mevcuttur.
  </Card>
  <Card title="Codex Hosted Search" icon="search" href="/tr/plugins/codex-harness">
    Codex app-server hesabınız üzerinden yapay zekâ tarafından sentezlenen, kaynaklara dayalı yanıtlar.
  </Card>
  <Card title="DuckDuckGo" icon="bird" href="/tr/tools/duckduckgo-search">
    Anahtarsız sağlayıcı. API anahtarı gerekmez. Resmî olmayan HTML tabanlı entegrasyon.
  </Card>
  <Card title="Exa" icon="brain" href="/tr/tools/exa-search">
    İçerik çıkarma (öne çıkan bölümler, metin, özetler) özellikli sinirsel + anahtar kelime araması.
  </Card>
  <Card title="Firecrawl" icon="flame" href="/tr/tools/firecrawl">
    Yapılandırılmış sonuçlar. Derinlemesine çıkarma için en iyi `firecrawl_search` ve `firecrawl_scrape` ile birlikte çalışır.
  </Card>
  <Card title="Gemini" icon="sparkles" href="/tr/tools/gemini-search">
    Google Search temellendirmesi aracılığıyla alıntılar içeren, yapay zekâ tarafından sentezlenmiş yanıtlar.
  </Card>
  <Card title="Grok" icon="zap" href="/tr/tools/grok-search">
    xAI web temellendirmesi aracılığıyla alıntılar içeren, yapay zekâ tarafından sentezlenmiş yanıtlar.
  </Card>
  <Card title="Kimi" icon="moon" href="/tr/tools/kimi-search">
    Moonshot web araması aracılığıyla alıntılar içeren, yapay zekâ tarafından sentezlenmiş yanıtlar; temellendirilmemiş sohbet geri dönüşleri açıkça başarısız olur.
  </Card>
  <Card title="MiniMax Search" icon="globe" href="/tr/tools/minimax-search">
    MiniMax Token Plan arama API'si aracılığıyla yapılandırılmış sonuçlar.
  </Card>
  <Card title="Ollama Web Search" icon="globe" href="/tr/tools/ollama-search">
    Oturum açılmış yerel bir Ollama ana makinesi veya barındırılan Ollama API'si üzerinden arama.
  </Card>
  <Card title="Parallel" icon="layer-group" href="/tr/tools/parallel-search">
    Ücretli Parallel Search API'si (`PARALLEL_API_KEY`); daha yüksek hız sınırları ve hedef ayarlama.
  </Card>
  <Card title="Parallel Search (Ücretsiz)" icon="layer-group" href="/tr/tools/parallel-search">
    İsteğe bağlı anahtarsız kullanım. LLM için optimize edilmiş yoğun alıntılar sunan ve API anahtarı gerektirmeyen Parallel'ın ücretsiz Search MCP'si.
  </Card>
  <Card title="Perplexity" icon="search" href="/tr/tools/perplexity-search">
    İçerik çıkarma denetimleri ve alan adı filtreleme özellikli yapılandırılmış sonuçlar.
  </Card>
  <Card title="SearXNG" icon="server" href="/tr/tools/searxng-search">
    Kendi sunucunuzda barındırılan meta arama. API anahtarı gerekmez. Google, Bing, DuckDuckGo ve daha fazlasını birleştirir.
  </Card>
  <Card title="Tavily" icon="globe" href="/tr/tools/tavily">
    Arama derinliği, konu filtreleme ve URL çıkarma için `tavily_extract` özellikli yapılandırılmış sonuçlar.
  </Card>
</CardGroup>

### Sağlayıcı karşılaştırması

| Sağlayıcı                                         | Sonuç biçimi                                                   | Filtreler                                          | API anahtarı                                                                                 |
| ------------------------------------------------ | -------------------------------------------------------------- | ------------------------------------------------ | --------------------------------------------------------------------------------------- |
| [Brave](/tr/tools/brave-search)                     | Yapılandırılmış parçacıklar                                            | Ülke, dil, zaman, `llm-context` modu      | `BRAVE_API_KEY`                                                                         |
| [Codex Hosted Search](/tr/plugins/codex-harness)    | Yapay zekâ sentezli + kaynak URL'leri                                   | Alan adları, bağlam boyutu, kullanıcı konumu             | Yok; Codex/OpenAI oturum açma bilgilerini kullanır                                                         |
| [DuckDuckGo](/tr/tools/duckduckgo-search)           | Yapılandırılmış parçacıklar                                            | --                                               | Yok (anahtarsız)                                                                         |
| [Exa](/tr/tools/exa-search)                         | Yapılandırılmış + çıkarılmış                                         | Sinirsel/anahtar kelime modu, tarih, içerik çıkarma    | `EXA_API_KEY`                                                                           |
| [Firecrawl](/tr/tools/firecrawl)                    | Yapılandırılmış parçacıklar                                            | `firecrawl_search` aracı üzerinden                      | `FIRECRAWL_API_KEY`                                                                     |
| [Gemini](/tr/tools/gemini-search)                   | Yapay zekâ sentezli + alıntılar                                     | --                                               | `GEMINI_API_KEY`                                                                        |
| [Grok](/tr/tools/grok-search)                       | Yapay zekâ sentezli + alıntılar                                     | --                                               | xAI OAuth, `XAI_API_KEY` veya `plugins.entries.xai.config.webSearch.apiKey`              |
| [Kimi](/tr/tools/kimi-search)                       | Yapay zekâ sentezli + alıntılar; temellendirilmemiş sohbet geri dönüşlerinde başarısız olur | --                                               | `KIMI_API_KEY` / `MOONSHOT_API_KEY`                                                     |
| [MiniMax Search](/tr/tools/minimax-search)          | Yapılandırılmış parçacıklar                                            | Bölge (`global` / `cn`)                         | `MINIMAX_CODE_PLAN_KEY` / `MINIMAX_CODING_API_KEY` / `MINIMAX_OAUTH_TOKEN`              |
| [Ollama Web Search](/tr/tools/ollama-search)        | Yapılandırılmış parçacıklar                                            | --                                               | Oturum açılmış yerel ana makineler için yok; doğrudan `https://ollama.com` araması için `OLLAMA_API_KEY` |
| [Parallel](/tr/tools/parallel-search)               | LLM bağlamına göre sıralanmış yoğun alıntılar                          | --                                               | `PARALLEL_API_KEY` (ücretli)                                                               |
| [Parallel Search (Ücretsiz)](/tr/tools/parallel-search) | LLM bağlamına göre sıralanmış yoğun alıntılar                          | --                                               | Yok (ücretsiz Search MCP)                                                                  |
| [Perplexity](/tr/tools/perplexity-search)           | Yapılandırılmış parçacıklar                                            | Ülke, dil, zaman, alan adları, içerik sınırları | `PERPLEXITY_API_KEY` / `OPENROUTER_API_KEY`                                             |
| [SearXNG](/tr/tools/searxng-search)                 | Yapılandırılmış parçacıklar                                            | Kategoriler, dil                             | Yok (kendi sunucunuzda barındırılır)                                                                      |
| [Tavily](/tr/tools/tavily)                          | Yapılandırılmış parçacıklar                                            | `tavily_search` aracı üzerinden                         | `TAVILY_API_KEY`                                                                        |

## Sonuç biçimi

`web_search`, tüm yerleşik ve harici plugin sağlayıcılarını çekirdek
araç sınırında normalleştirir. Çağıranlar tam olarak şu kapalı biçimlerden birini alır:

```typescript
type WebSearchOutput =
  | {
      kind: "error";
      provider: string;
      error: "provider_error";
      message: string;
      docs?: string;
    }
  | {
      kind: "results";
      provider: string;
      query: string;
      count: number;
      tookMs?: number;
      results: Array<{
        title: string;
        url: string;
        snippet?: string;
        published?: string;
        siteName?: string;
      }>;
      externalContent: {
        untrusted: true;
        source: "web_search";
        wrapped: true;
        provider: string;
      };
      cached?: true;
    }
  | {
      kind: "answer";
      provider: string;
      query: string;
      tookMs?: number;
      content: string;
      citations?: Array<{ url: string; title?: string }>;
      externalContent: {
        untrusted: true;
        source: "web_search";
        wrapped: true;
        provider: string;
      };
      cached?: true;
    }
  | {
      kind: "raw";
      provider: string;
      data: unknown;
    };
```

Yapılandırılmış sağlayıcılar `kind: "results"`, sentezlenmiş sağlayıcılarsa
`kind: "answer"` kullanır. Yükleri bu iki biçimden hiçbirine uymayan harici plugin
sağlayıcıları, uyumluluk için `kind: "raw"` olarak değiştirilmeden geçirilir. Ham puanlar, alıntılar, ilgili aramalar, satır içi alıntı
konumları, model kimlikleri veya oturum meta verileri gibi sağlayıcıya özgü
alanlar normalleştirilmiş dallarda aktarılmaz. Daha zengin yanıtı
iş akışınızın bir parçasıysa sağlayıcının özel aracını kullanın.

`externalContent.wrapped: true`, sınırın kendisinin doğruladığı bir güven işaretidir:
sağlayıcı metinlerindeki (`title`, `snippet`, `siteName`, `content`, alıntı
başlıkları, hata `message`) önceden var olan tüm zarf satırları kaldırılır ve
çekirdek sınırında tam olarak bir kez yeniden sarılır; böylece hiçbir sağlayıcı meta verisi
işareti taklit edemez. `query` her zaman istenen sorgudur, alıntı ve sonuç URL'leri
http(s) olarak ayrıştırılabilmelidir, `published` ISO tarih biçiminde olmalıdır, URL'ler standartlaştırılmış olarak yayımlanır ve
`error` anahtarı taşıyan bir yük her zaman `kind: "error"` olarak bildirilirken
ham sağlayıcı kodu sarılmış iletinin içinde korunur. Ham olarak doğrudan geçirilen
yükler, sağlayıcının ayarladığı işaretleri korur.

## Otomatik algılama

Belgelerdeki ve kurulum akışlarındaki sağlayıcı listeleri alfabetik sıradadır. Otomatik algılama
ayrı, sabit bir öncelik sırası kullanır ve kimlik bilgisi gerektiren bir sağlayıcıyı
(`requiresCredential !== false`) yalnızca yapılandırılmış olduğunu tespit ederse seçer.
Hiçbir `provider` ayarlanmamışsa OpenClaw sağlayıcıları şu sırayla denetler ve
hazır olan ilkini kullanır:

Önce API tabanlı sağlayıcılar:

1. **Brave** -- `BRAVE_API_KEY` veya `plugins.entries.brave.config.webSearch.apiKey` (sıra 10)
2. **MiniMax Search** -- `MINIMAX_CODE_PLAN_KEY` / `MINIMAX_CODING_API_KEY` / `MINIMAX_OAUTH_TOKEN` / `MINIMAX_API_KEY` veya `plugins.entries.minimax.config.webSearch.apiKey` (sıra 15)
3. **Gemini** -- `plugins.entries.google.config.webSearch.apiKey`, `GEMINI_API_KEY` veya `models.providers.google.apiKey` (sıra 20)
4. **Grok** -- xAI OAuth, `XAI_API_KEY` veya `plugins.entries.xai.config.webSearch.apiKey` (sıra 30)
5. **Kimi** -- `KIMI_API_KEY` / `MOONSHOT_API_KEY` veya `plugins.entries.moonshot.config.webSearch.apiKey` (sıra 40)
6. **Perplexity** -- `PERPLEXITY_API_KEY` / `OPENROUTER_API_KEY` veya `plugins.entries.perplexity.config.webSearch.apiKey` (sıra 50)
7. **Firecrawl** -- `FIRECRAWL_API_KEY` veya `plugins.entries.firecrawl.config.webSearch.apiKey` (sıra 60)
8. **Exa** -- `EXA_API_KEY` veya `plugins.entries.exa.config.webSearch.apiKey`; isteğe bağlı `plugins.entries.exa.config.webSearch.baseUrl`, Exa uç noktasını geçersiz kılar (sıra 65)
9. **Tavily** -- `TAVILY_API_KEY` veya `plugins.entries.tavily.config.webSearch.apiKey` (sıra 70)
10. **Parallel** -- `PARALLEL_API_KEY` veya `plugins.entries.parallel.config.webSearch.apiKey` üzerinden ücretli Parallel Search API; isteğe bağlı `plugins.entries.parallel.config.webSearch.baseUrl`, uç noktayı geçersiz kılar (sıra 75)

Bunlardan sonra yapılandırılmış uç nokta sağlayıcıları:

11. **SearXNG** -- `SEARXNG_BASE_URL` veya `plugins.entries.searxng.config.webSearch.baseUrl` (sıra 200)

**Parallel Search (Free)**, **DuckDuckGo**,
**Ollama Web Search** ve **Codex Hosted Search** gibi anahtar gerektirmeyen sağlayıcılar,
dahili bir sıra değerine sahip olsalar bile otomatik algılamada hiçbir zaman
seçilmez. Yalnızca `tools.web.search.provider` ile veya
`openclaw configure --section web` üzerinden açıkça seçildiklerinde kullanılırlar.
API destekli hiçbir sağlayıcı yapılandırılmadığı için OpenClaw, yönetilen
`web_search` sorgularını anahtar gerektirmeyen bir sağlayıcıya göndermez.

OpenAI Responses modelleri bir istisnadır: `tools.web.search.provider`
ayarlanmamışken yukarıdaki yönetilen sağlayıcılar yerine OpenAI'ın yerel web
aramasını kullanırlar (aşağıya bakın). Bunları bunun yerine yönetilen yol
üzerinden yönlendirmek için `tools.web.search.provider` değerini
`parallel-free` (veya başka bir sağlayıcı) olarak ayarlayın.

<Note>
  Tüm sağlayıcı anahtar alanları SecretRef nesnelerini destekler.
  `plugins.entries.<plugin>.config.webSearch.apiKey` altındaki Plugin kapsamlı SecretRef'ler;
  sağlayıcı `tools.web.search.provider` aracılığıyla açıkça seçilmiş veya otomatik
  algılama yoluyla belirlenmiş olsa da Brave, Exa, Firecrawl, Gemini, Grok,
  Kimi, MiniMax, Parallel, Perplexity ve Tavily dâhil olmak üzere yüklü API
  destekli web arama sağlayıcıları için çözümlenir. Otomatik algılama modunda
  OpenClaw yalnızca seçilen sağlayıcının anahtarını çözümler; seçilmeyen
  SecretRef'ler etkinlik dışı kalır. Böylece kullanmadığınız sağlayıcılar için
  çözümleme maliyetine katlanmadan birden fazla sağlayıcıyı yapılandırabilirsiniz.
</Note>

## Yerel OpenAI web araması

Doğrudan OpenAI Responses modelleri (`api: "openai-responses"`, sağlayıcı `openai`,
temel URL olmadan veya resmî bir OpenAI API temel URL'siyle), OpenClaw web
araması etkinleştirildiğinde ve hiçbir yönetilen sağlayıcı sabitlenmediğinde
OpenAI'ın barındırılan `web_search` aracını otomatik olarak kullanır.
Bu, paketlenmiş OpenAI Plugin'ine ait sağlayıcı davranışıdır ve OpenAI uyumlu
proxy temel URL'lerine veya Azure yollarına uygulanmaz. OpenAI modellerinde
yönetilen `web_search` aracını korumak için `tools.web.search.provider` değerini
`brave` gibi başka bir sağlayıcıya ayarlayın ya da hem yönetilen
aramayı hem de yerel OpenAI aramasını devre dışı bırakmak için
`tools.web.search.enabled: false` değerini ayarlayın.

## Yerel Codex web araması

Codex app-server çalışma zamanı, web araması etkinleştirildiğinde ve hiçbir
yönetilen sağlayıcı seçilmediğinde Codex'in barındırılan `web_search`
aracını otomatik olarak kullanır. Yerel barındırılan arama ile OpenClaw'ın
yönetilen dinamik `web_search` aracı birbirini dışlar; dolayısıyla
yönetilen arama, yerel alan adı kısıtlamalarını atlayamaz. Barındırılan arama
kullanılamadığında, açıkça devre dışı bırakıldığında veya seçili bir yönetilen
sağlayıcıyla değiştirildiğinde OpenClaw yönetilen aracı kullanır. Üretim
app-server trafiği kullanıcı tanımlı `web` ad alanını reddettiği
için OpenClaw, Codex'in bağımsız `web.run` uzantısını devre dışı
(`features.standalone_web_search: false`) tutar.

- Yerel aramayı `tools.web.search.openaiCodex` altında yapılandırın
- Herhangi bir üst model için Codex Hosted Search'ü yönetilen
  `web_search` sağlayıcısı olarak hazırlamak üzere `tools.web.search.provider: "codex"`
  değerini ayarlayın. Her çağrı, sınırlı ve geçici bir Codex app-server turu
  çalıştırır ve Codex barındırılan bir `webSearch` öğesi üretmezse
  başarısız olur.
- `mode: "cached"` varsayılan tercihtir ancak Codex bunu
  kısıtlanmamış app-server turları için canlı harici erişime çözümler; canlı
  erişimi açıkça istemek için `"live"` değerini ayarlayın
- Bunun yerine OpenClaw'ın yönetilen `web_search` aracını
  kullanmak için `tools.web.search.provider` değerini `brave` gibi yönetilen
  bir sağlayıcıya ayarlayın
- Codex tarafından barındırılan aramadan çıkmak için
  `tools.web.search.openaiCodex.enabled: false` değerini ayarlayın; diğer yönetilen sağlayıcılar
  kullanılabilir kalır
- Codex yerel araç yüzeyinin kısıtlanması, yönetilen
  `web_search` aracını da kullanılabilir tutar
- `allowedDomains` ayarlandığında barındırılan arama
  kullanılamıyorsa otomatik yönetilen geri dönüş kapalı biçimde başarısız olur;
  böylece yerel izin listesi atlanamaz
- Araçların devre dışı olduğu yalnızca LLM çalıştırmaları hem
  yerel hem de yönetilen aramayı devre dışı bırakır
- `tools.web.search.enabled: false` hem yönetilen hem de yerel aramayı devre
  dışı bırakır

Kalıcı ve etkin Codex arama politikası değişiklikleri, önceden yüklenmiş bir
app-server iş parçacığının eski barındırılan arama erişimini koruyamaması için
yeni bir bağlı iş parçacığı başlatır. Tur başına geçici kısıtlamalar, geçici ve
kısıtlı bir iş parçacığı kullanır; daha sonra devam etmek üzere mevcut bağı
korur.

Doğrudan OpenAI ChatGPT Responses trafiği de OpenAI'ın barındırılan
`web_search` aracını kullanabilir. Bu ayrı yol,
`tools.web.search.openaiCodex.enabled: true` aracılığıyla isteğe bağlı kalır ve yalnızca
`api: "openai-chatgpt-responses"` kullanan uygun `openai/*` modellerine uygulanır.

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        // İsteğe bağlı: Codex Hosted Search'ü Codex dışı üst modellerden de kullanın.
        provider: "codex",
        openaiCodex: {
          enabled: true,
          mode: "cached",
          allowedDomains: ["example.com"],
          contextSize: "high",
          userLocation: {
            country: "US",
            city: "New York",
            timezone: "America/New_York",
          },
        },
      },
    },
  },
}
```

Yerel Codex aramasını desteklemeyen çalışma zamanları ve sağlayıcılar için
Codex, OpenClaw'ın dinamik araç ad alanı üzerinden yönetilen
`web_search` geri dönüşünü kullanabilir. Codex tarafından barındırılan
arama yerine OpenClaw'ın sağlayıcıya özgü ağ denetimlerine ihtiyaç duyduğunuzda
açıkça yönetilen bir sağlayıcı kullanın.

`provider: "codex"` seçildiğinde paketlenmiş `codex` Plugin'i
etkinleştirilir ve yukarıda gösterilen `tools.web.search.openaiCodex` kısıtlamaları
kullanılır. Önce `openclaw models auth login --provider openai` ile Codex app-server kimlik doğrulamasını
yapın. Üst aracı herhangi bir model veya çalışma zamanı kullanabilir; yalnızca
sınırlı arama çalışanı Codex üzerinden çalışır.

## Ağ güvenliği

Yönetilen HTTP `web_search` sağlayıcı çağrıları, geçerli sağlayıcının
kendi ana makine adıyla sınırlandırılmış OpenClaw korumalı getirme yolunu
kullanır. OpenClaw yalnızca bu ana makine adı için `198.18.0.0/15` ve
`fc00::/7` içindeki Surge, Clash ve sing-box sahte IP DNS yanıtlarına
izin verir. Diğer özel, geri döngü, bağlantı yerel ve meta veri hedefleri
engellenmeye devam eder. Codex Hosted Search istisnadır: sınırlı çalışanı, ağ
erişimini Codex app-server'ın barındırılan `web_search` aracına devreder.

Bu otomatik izin, rastgele `web_fetch` URL'lerine uygulanmaz.
`web_fetch` için yalnızca güvenilir proxy'niz bu sentetik aralıkların
sahibiyse `tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange` ve `tools.web.fetch.ssrfPolicy.allowIpv6UniqueLocalRange` seçeneklerini açıkça
etkinleştirin.

## Yapılandırma

```json5
{
  tools: {
    web: {
      search: {
        enabled: true, // varsayılan: true
        provider: "brave", // veya otomatik algılama için atlayın
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
    },
  },
}
```

Sağlayıcıya özgü yapılandırma (API anahtarları, temel URL'ler, modlar)
`plugins.entries.<plugin>.config.webSearch.*` altında bulunur. Gemini ayrıca özel web arama yapılandırması
ve `GEMINI_API_KEY` sonrasında daha düşük öncelikli geri dönüşler olarak
`models.providers.google.apiKey` ve `models.providers.google.baseUrl` değerlerini yeniden kullanabilir.
Örnekler için sağlayıcı sayfalarına bakın.
Grok ayrıca `openclaw models auth login
--provider xai --method oauth` içindeki bir xAI OAuth kimlik doğrulama profilini
yeniden kullanabilir; API anahtarı yapılandırması geri dönüş seçeneği olmaya
devam eder.

`tools.web.search.provider`, paketlenmiş ve yüklü Plugin manifestleri tarafından
bildirilen web arama sağlayıcısı kimliklerine göre doğrulanır.
`"brvae"` gibi bir yazım hatası, sessizce otomatik algılamaya geri
dönmek yerine yapılandırma doğrulamasının başarısız olmasına neden olur.
Yapılandırılmış bir sağlayıcının yalnızca güncelliğini yitirmiş Plugin kanıtı
varsa (örneğin üçüncü taraf bir Plugin kaldırıldıktan sonra kalan bir
`plugins.entries.<plugin>` bloğu), OpenClaw başlangıcın dayanıklılığını korur ve
Plugin'i yeniden yükleyebilmeniz veya eski yapılandırmayı temizlemek için
`openclaw doctor --fix` komutunu çalıştırabilmeniz amacıyla bir uyarı bildirir.

`web_fetch` geri dönüş sağlayıcısı seçimi ayrıdır:

- bunu `tools.web.fetch.provider` ile seçin
- veya bu alanı atlayarak OpenClaw'ın yapılandırılmış kimlik
  bilgilerinden hazır durumdaki ilk web getirme sağlayıcısını otomatik olarak
  algılamasına izin verin
- korumalı alanda olmayan `web_fetch`, 
  `contracts.webFetchProviders` bildiren yüklü Plugin sağlayıcılarını kullanabilir;
  korumalı alan getirmeleri paketlenmiş sağlayıcılara ve doğrulanmış resmî
  Plugin yüklemelerine izin verir ancak harici üçüncü taraf Plugin'lerini
  hariç tutar
- resmî Firecrawl Plugin'i günümüzde paketlenmiş tek
  `webFetchProviders` katkı sağlayıcısıdır ve
  `plugins.entries.firecrawl.config.webFetch.*` altında yapılandırılır

`openclaw onboard` veya `openclaw configure --section web` sırasında **Kimi**'yi seçtiğinizde
OpenClaw şunları da sorabilir:

- Moonshot API bölgesi (`https://api.moonshot.ai/v1` veya `https://api.moonshot.cn/v1`)
- varsayılan Kimi web arama modeli (varsayılan:
  `kimi-k2.6`)

`x_search` için `plugins.entries.xai.config.xSearch.*` değerini yapılandırın. Sohbetle aynı
xAI kimlik doğrulama profilini veya Grok web araması tarafından kullanılan
`XAI_API_KEY` / Plugin web arama kimlik bilgisini kullanır.
Eski `tools.web.x_search.*` yapılandırması, `openclaw doctor --fix` tarafından otomatik
olarak taşınır.
`openclaw onboard` veya `openclaw configure --section web` sırasında Grok'u seçtiğinizde OpenClaw,
Grok kurulumu tamamlandıktan hemen sonra aynı kimlik bilgisiyle isteğe bağlı
`x_search` kurulumu da sunar. Bu, ayrı bir üst düzey web arama
sağlayıcısı seçeneği değil, Grok yolu içindeki ayrı bir takip adımıdır. Başka
bir sağlayıcı seçerseniz OpenClaw, `x_search` istemini göstermez.

### API anahtarlarını saklama

<Tabs>
  <Tab title="Yapılandırma dosyası">
    `openclaw configure --section web` komutunu çalıştırın veya anahtarı doğrudan ayarlayın:

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "YOUR_KEY", // pragma: allowlist secret
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="Ortam değişkeni">
    Sağlayıcının ortam değişkenini Gateway işleminin ortamında ayarlayın:

    ```bash
    export BRAVE_API_KEY="YOUR_KEY"
    ```

    Bir Gateway yüklemesi için bunu `~/.openclaw/.env` içine yerleştirin.
    [Ortam değişkenleri](/tr/help/faq#env-vars-and-env-loading) bölümüne bakın.

  </Tab>
</Tabs>

## Araç parametreleri

| Parametre             | Açıklama                                                        |
| --------------------- | ------------------------------------------------------------------ |
| `query`               | Arama sorgusu (zorunlu)                                            |
| `count`               | Döndürülecek sonuçlar (1-10, varsayılan: 5)                               |
| `country`             | 2 harfli ISO ülke kodu (ör. "US", "DE")                        |
| `language`            | ISO 639-1 dil kodu (ör. "en", "de")                          |
| `search_lang`         | Arama dili kodu (yalnızca Brave)                                  |
| `freshness`           | Zaman filtresi: `day`, `week`, `month` veya `year`                     |
| `date_after`          | Bu tarihten sonraki sonuçlar (YYYY-MM-DD)                               |
| `date_before`         | Bu tarihten önceki sonuçlar (YYYY-MM-DD)                              |
| `ui_lang`             | Kullanıcı arayüzü dil kodu (yalnızca Brave)                                      |
| `domain_filter`       | Etki alanı izin/ret listesi dizisi (yalnızca Perplexity)                  |
| `max_tokens`          | Toplam içerik token bütçesi, yalnızca yerel Perplexity Search API      |
| `max_tokens_per_page` | Sayfa başına ayıklama token sınırı, yalnızca yerel Perplexity Search API |

<Warning>
  Tüm parametreler tüm sağlayıcılarla çalışmaz. Brave `llm-context` modu
  `ui_lang` parametresini reddeder; Brave özel güncellik aralıkları
  hem başlangıç hem de bitiş tarihlerini gerektirdiğinden `date_before` için
  ayrıca `date_after` gerekir.
  Gemini, Grok ve Kimi, alıntılarla birlikte sentezlenmiş tek bir yanıt döndürür.
  Paylaşılan araç uyumluluğu için `count` parametresini kabul ederler,
  ancak bu parametre temellendirilmiş yanıtın biçimini değiştirmez. Gemini,
  `day` güncelliğini bir yakın geçmiş ipucu olarak değerlendirir;
  daha geniş güncellik değerleri ve açık tarihler, Google Search temellendirme
  zaman aralıklarını belirler.
  Sonar/OpenRouter uyumluluk yolunu (`plugins.entries.perplexity.config.webSearch.baseUrl` /
  `model` veya `OPENROUTER_API_KEY`) kullandığınızda Perplexity de aynı
  şekilde davranır; bu yol ayrıca `max_tokens` ve
  `max_tokens_per_page` desteğini kaldırır.
  SearXNG, `http://` değerini yalnızca güvenilir özel ağ veya geri döngü
  ana bilgisayarları için kabul eder; herkese açık SearXNG uç noktaları
  `https://` kullanmalıdır.
  Firecrawl ve Tavily, `query` ve `count` parametrelerini
  yalnızca `web_search` üzerinden destekler; gelişmiş seçenekler için
  bunlara özel araçları kullanın.
</Warning>

## x_search

`x_search`, xAI kullanarak X (eski adıyla Twitter) gönderilerini sorgular
ve alıntılarla birlikte yapay zekâ tarafından sentezlenmiş yanıtlar döndürür.
Doğal dil sorgularını ve isteğe bağlı yapılandırılmış filtreleri kabul eder.
OpenClaw, yerleşik xAI `x_search` aracını kalıcı olarak kayıtlı tutmak
yerine her istek için oluşturur; dolayısıyla araç yalnızca gerçekten çağrıldığı
etkileşim sırasında etkindir.

<Warning>
  `x_search`, xAI sunucularında çalışır. xAI, modelin giriş ve çıkış
  token ücretlerine ek olarak 1.000 araç çağrısı başına $5 ücret alır.
</Warning>

<Note>
  xAI, `x_search` özelliğinin anahtar kelime aramasını, anlamsal aramayı,
  kullanıcı aramasını ve ileti dizisi getirmeyi desteklediğini belirtir. Yeniden
  gönderimler, yanıtlar, yer imleri veya görüntülemeler gibi gönderi başına
  etkileşim istatistikleri için tam gönderi URL'sine veya durum kimliğine yönelik
  hedefli bir aramayı tercih edin. Geniş anahtar kelime aramaları doğru gönderiyi
  bulabilir ancak gönderi başına daha az eksiksiz meta veri döndürebilir. İyi bir
  yöntem şöyledir: önce gönderiyi bulun, ardından tam olarak bu gönderiye
  odaklanan ikinci bir `x_search` sorgusu çalıştırın.
</Note>

### x_search yapılandırması

`enabled` belirtilmediğinde `x_search`, yalnızca etkin modelin
sağlayıcısı `xai` olduğunda ve xAI kimlik bilgileri çözümlendiğinde
kullanıma sunulur. Sağlayıcısı xAI olmadığı bilinen etkin bir modelde sağlayıcılar
arası kullanımı etkinleştirmek için `plugins.entries.xai.config.xSearch.enabled` değerini
`true` olarak ayarlayın. Etkin model sağlayıcısı eksikse veya
çözümlenemiyorsa araç gizli kalır. Tüm sağlayıcılarda devre dışı bırakmak için
`enabled` değerini `false` olarak ayarlayın. xAI kimlik
bilgileri her zaman zorunludur.

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          xSearch: {
            enabled: true, // xAI olmadığı bilinen bir model sağlayıcısı için zorunlu
            model: "grok-4.3",
            baseUrl: "https://api.x.ai/v1", // isteğe bağlıdır, webSearch.baseUrl değerini geçersiz kılar
            inlineCitations: false,
            maxTurns: 2,
            timeoutSeconds: 30,
            cacheTtlMinutes: 15,
          },
          webSearch: {
            apiKey: "xai-...", // bir xAI kimlik doğrulama profili veya XAI_API_KEY ayarlanmışsa isteğe bağlıdır
            baseUrl: "https://api.x.ai/v1", // isteğe bağlı paylaşılan xAI Responses temel URL'si
          },
        },
      },
    },
  },
}
```

`plugins.entries.xai.config.xSearch.baseUrl` ayarlandığında `x_search`,
`<baseUrl>/responses` adresine gönderim yapar. Bu alan belirtilmezse önce
`plugins.entries.xai.config.webSearch.baseUrl`, ardından herkese açık xAI uç noktası
(`https://api.x.ai/v1`) kullanılır.

### x_search parametreleri

| Parametre                    | Açıklama                                            |
| ---------------------------- | ------------------------------------------------------ |
| `query`                      | Arama sorgusu (zorunlu)                                |
| `allowed_x_handles`          | Sonuçları en fazla 20 X kullanıcı adıyla sınırlandırır               |
| `excluded_x_handles`         | En fazla 20 X kullanıcı adını hariç tutar                           |
| `from_date`                  | Yalnızca bu tarihte veya sonrasında yayımlanan gönderileri dahil eder (YYYY-MM-DD)  |
| `to_date`                    | Yalnızca bu tarihte veya öncesinde yayımlanan gönderileri dahil eder (YYYY-MM-DD) |
| `enable_image_understanding` | xAI'ın eşleşen gönderilere eklenmiş görselleri incelemesine izin verir      |
| `enable_video_understanding` | xAI'ın eşleşen gönderilere eklenmiş videoları incelemesine izin verir      |

`allowed_x_handles` ve `excluded_x_handles` birbirini dışlar.

### x_search örneği

```javascript
await x_search({
  query: "akşam yemeği tarifleri",
  allowed_x_handles: ["nytfood"],
  from_date: "2026-03-01",
});
```

```javascript
// Gönderi başına istatistikler: mümkün olduğunda tam durum URL'sini veya durum kimliğini kullanın
await x_search({
  query: "https://x.com/huntharo/status/1905678901234567890",
});
```

## Örnekler

```javascript
// Temel arama
await web_search({ query: "OpenClaw plugin SDK" });

// Almancaya özgü arama
await web_search({ query: "çevrimiçi TV izleme", country: "DE", language: "de" });

// Güncel sonuçlar (geçen hafta)
await web_search({ query: "yapay zekâ gelişmeleri", freshness: "week" });

// Tarih aralığı
await web_search({
  query: "iklim araştırması",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});

// Etki alanı filtreleme (yalnızca Perplexity)
await web_search({
  query: "ürün incelemeleri",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});
```

## Araç profilleri

Araç profilleri veya izin listeleri kullanıyorsanız `web_search`,
`x_search` veya `group:web` ekleyin:

```json5
{
  tools: {
    allow: ["web_search", "x_search"],
    // veya: allow: ["group:web"]  (web_search, x_search ve web_fetch içerir)
  },
}
```

## İlgili

- [Web Fetch](/tr/tools/web-fetch) -- bir URL'yi getirir ve okunabilir içeriği ayıklar
- [Web Browser](/tr/tools/browser) -- yoğun JS kullanan siteler için tam tarayıcı otomasyonu
- [Grok Search](/tr/tools/grok-search) -- `web_search` sağlayıcısı olarak Grok
- [Ollama Web Search](/tr/tools/ollama-search) -- Ollama ana bilgisayarınız üzerinden anahtarsız web araması
