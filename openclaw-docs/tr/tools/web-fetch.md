---
read_when:
    - Bir URL'yi getirmek ve okunabilir içeriği ayıklamak istiyorsunuz
    - web_fetch veya Firecrawl geri dönüşünü yapılandırmanız gerekir
    - web_fetch sınırlarını ve önbelleğe almayı anlamak istiyorsunuz
sidebarTitle: Web Fetch
summary: web_fetch aracı -- okunabilir içerik ayıklamalı HTTP getirme işlemi
title: Web'den getirme
x-i18n:
    generated_at: "2026-07-26T23:06:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf312245064672dcf489e8714740fa3e034827e16b33be8fb6a87db04f19ef8
    source_path: tools/web-fetch.md
    workflow: 16
---

`web_fetch` düz bir HTTP GET isteği gerçekleştirir ve okunabilir içeriği (HTML'den
markdown'a veya metne) çıkarır. JavaScript'i **çalıştırmaz**. JS ağırlıklı siteler veya
oturum açma korumalı sayfalar için bunun yerine [Web Tarayıcısı](/tr/tools/browser) kullanın.

## Hızlı başlangıç

Varsayılan olarak etkindir, yapılandırma gerekmez:

```javascript
await web_fetch({ url: "https://example.com/article" });
```

## Araç parametreleri

<ParamField path="url" type="string" required>
Getirilecek URL. Yalnızca `http(s)`.
</ParamField>

<ParamField path="extractMode" type="'markdown' | 'text'" default="markdown">
Ana içerik çıkarıldıktan sonraki çıktı biçimi.
</ParamField>

<ParamField path="maxChars" type="number">
Çıktıyı bu karakter sayısına kadar kırpın. `tools.web.fetch.maxCharsCap` ile sınırlandırılır.
</ParamField>

## Sonuç

`web_fetch` şu alanları içeren kapalı, yapılandırılmış bir sonuç döndürür:

- İstek meta verileri: `url`, `finalUrl`, `status`, `extractMode` ve `extractor`
- İsteğe bağlı yanıt meta verileri: `contentType`, `title` ve `warning` (mevcut olmadığında dahil edilmez)
- Sarmalanmış içerik meta verileri: `externalContent`, `truncated`, `length`, `rawLength`,
  `fetchedAt`, `tookMs` ve `text`
- Önbellek isabetinde isteğe bağlı `cached: true`
- Kırpılmış içerik özel bir geçici dosyaya yazıldığında isteğe bağlı
  `spill: { path, chars, truncated? }`; `truncated` yalnızca bu dosya kısmi kaynak içeriği
  barındırdığında mevcuttur

`length`, sarmalanmış `text` uzunluğudur. `rawLength`, harici içerik
sarmalamasından önceki çıkarılmış içerik uzunluğudur.

## Nasıl çalışır?

<Steps>
  <Step title="Getirme">
    Chrome benzeri bir User-Agent ve `Accept-Language` üst bilgisiyle bir HTTP GET
    isteği gönderir. Özel/dahili ana bilgisayar adlarını engeller ve yönlendirmeleri yeniden denetler.
  </Step>
  <Step title="Çıkarma">
    HTML yanıtında Readability'yi (ana içerik çıkarma) çalıştırır.
  </Step>
  <Step title="Geri dönüş (isteğe bağlı)">
    Readability başarısız olursa ve bir getirme sağlayıcısı mevcutsa istek
    bu sağlayıcı üzerinden yeniden denenir (örneğin Firecrawl'ın bot engellerini aşma modu).
  </Step>
  <Step title="Önbellek">
    Aynı URL'nin tekrar tekrar getirilmesini azaltmak için sonuçlar 15 dakika
    boyunca önbelleğe alınır (yapılandırılabilir).
  </Step>
</Steps>

## İlerleme güncellemeleri

`web_fetch`, yalnızca getirme işlemi beş saniye sonra hâlâ beklemedeyse
herkese açık bir ilerleme satırı yayınlar:

```text
Sayfa içeriği getiriliyor...
```

Hızlı önbellek isabetleri ve hızlı ağ yanıtları zamanlayıcı tetiklenmeden önce tamamlandığından
hiçbir zaman ilerleme satırı göstermez. Çağrının iptal edilmesi zamanlayıcıyı temizler.
İlerleme satırı yalnızca kanal kullanıcı arayüzü durumudur ve getirilen sayfa içeriğini asla barındırmaz.

## Yapılandırma

```json5
{
  tools: {
    web: {
      fetch: {
        enabled: true, // varsayılan: true
        provider: "firecrawl", // isteğe bağlı; otomatik algılama için dahil etmeyin
        maxChars: 20000, // varsayılan çıktı karakterleri; maxCharsCap ile sınırlandırılır
        maxCharsCap: 20000, // maxChars parametresi için kesin üst sınır
        maxResponseBytes: 750000, // kırpmadan önceki en yüksek indirme boyutu (32000-10000000)
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        useTrustedEnvProxy: false, // güvenilir bir HTTP(S) ortam proxy'sinin DNS'i çözmesine izin ver
        readability: true, // Readability çıkarmasını kullan
        userAgent: "Mozilla/5.0 ...", // User-Agent'ı geçersiz kıl
        ssrfPolicy: {
          allowRfc2544BenchmarkRange: true, // 198.18.0.0/15 kullanan güvenilir sahte IP proxy'leri için açık katılım
          allowIpv6UniqueLocalRange: true, // fc00::/7 kullanan güvenilir sahte IP proxy'leri için açık katılım
        },
      },
    },
  },
}
```

## Firecrawl geri dönüşü

Readability çıkarma işlemi başarısız olursa `web_fetch`, bot engellerini aşmak ve
daha iyi çıkarma sağlamak için [Firecrawl](/tr/tools/firecrawl) hizmetine geri dönebilir:

```json5
{
  tools: {
    web: {
      fetch: {
        provider: "firecrawl", // isteğe bağlı; mevcut kimlik bilgilerinden otomatik algılama için dahil etmeyin
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            // apiKey: "fc-...", // isteğe bağlı; anahtarsız başlangıç erişimi için dahil etmeyin
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 172800000, // önbellek süresi (2 gün)
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

`plugins.entries.firecrawl.config.webFetch.apiKey` isteğe bağlıdır ve SecretRef nesnelerini destekler.
Eski `tools.web.fetch.firecrawl.*` yapılandırması, `openclaw doctor --fix` aracılığıyla
otomatik olarak `plugins.entries.firecrawl.config.webFetch` biçimine geçirilir.

<Note>
  Bir Firecrawl API anahtarı SecretRef'i yapılandırırsanız ve bu değer,
  `FIRECRAWL_API_KEY` ortam geri dönüşü olmadan çözümlenemezse Gateway başlatma işlemi hızlıca başarısız olur.
</Note>

<Note>
  Firecrawl `baseUrl` geçersiz kılmaları sıkı biçimde kısıtlanmıştır: barındırılan trafik
  `https://api.firecrawl.dev` kullanır; kendi barındırdığınız geçersiz kılmalar özel veya
  dahili uç noktaları hedeflemelidir ve `http://` yalnızca bu özel hedefler için kabul edilir.
</Note>

Geçerli çalışma zamanı davranışı:

- `tools.web.fetch.provider`, getirme geri dönüşü sağlayıcısını açıkça seçer.
- `provider` dahil edilmezse OpenClaw, yapılandırılmış kimlik bilgilerinden hazır durumdaki ilk web getirme
  sağlayıcısını otomatik olarak algılar. Korumalı alanda çalışmayan `web_fetch`, `contracts.webFetchProviders` bildiren ve çalışma
  zamanında eşleşen bir sağlayıcı kaydeden yüklü Plugin'leri kullanabilir. Resmî Firecrawl Plugin'i
  günümüzde bu geri dönüşü sağlar.
- Korumalı alandaki `web_fetch` çağrıları, paketlenmiş sağlayıcıların yanı sıra resmî npm veya ClawHub
  kaynağı doğrulanmış yüklü sağlayıcılara izin verir. Günümüzde bu, resmî Firecrawl Plugin'ine
  izin verir; üçüncü taraf harici getirme Plugin'leri kapsam dışında kalır.
- Readability devre dışıysa `web_fetch` doğrudan seçilen sağlayıcı
  geri dönüşüne geçer. Kullanılabilir sağlayıcı yoksa güvenli biçimde başarısız olur.

## Güvenilir ortam proxy'si

Dağıtımınızda `web_fetch` isteğinin güvenilir bir giden
HTTP(S) proxy'si üzerinden geçmesi gerekiyorsa `tools.web.fetch.useTrustedEnvProxy: true` değerini ayarlayın.

Bu modda OpenClaw, isteği göndermeden önce ana bilgisayar adına dayalı SSRF denetimlerini
uygulamaya devam eder ancak yerel DNS sabitlemesi yapmak yerine proxy'nin DNS'i çözmesine
izin verir. Bunu yalnızca proxy operatör denetimindeyse ve DNS çözümlemesinden sonra
giden trafik politikasını uyguluyorsa etkinleştirin.

<Note>
  Hiçbir HTTP(S) proxy ortam değişkeni yapılandırılmamışsa veya hedef ana bilgisayar
  `NO_PROXY` tarafından hariç tutulmuşsa `web_fetch`, yerel DNS
  sabitlemesi kullanan normal katı yola geri döner.
</Note>

## Sınırlar ve güvenlik

- `maxChars`, `tools.web.fetch.maxCharsCap` ile sınırlandırılır (varsayılan `20000`)
- Yanıt gövdesi ayrıştırılmadan önce `maxResponseBytes` ile sınırlandırılır (varsayılan `750000`,
  32000-10000000 aralığıyla sınırlandırılır); aşırı büyük yanıtlar bir uyarıyla kırpılır
- Özel/dahili ana bilgisayar adları engellenir
- `tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange` ve
  `tools.web.fetch.ssrfPolicy.allowIpv6UniqueLocalRange`, güvenilir sahte IP proxy yığınları için dar kapsamlı açık katılımlardır;
  proxy'niz bu sentetik aralıkların sahibi değilse ve kendi hedef politikasını uygulamıyorsa
  bunları ayarlamayın
- Yönlendirmeler denetlenir ve `maxRedirects` ile sınırlandırılır (varsayılan `3`)
- `useTrustedEnvProxy` açık bir katılımdır ve yalnızca DNS çözümlemesinden sonra da
  giden trafik politikasını uygulayan, operatör denetimindeki proxy'ler için etkinleştirilmelidir
- `web_fetch` en iyi çaba esasına dayanır -- bazı siteler [Web Tarayıcısı](/tr/tools/browser) gerektirir

## Araç profilleri

Araç profilleri veya izin listeleri kullanıyorsanız `web_fetch` ya da `group:web` ekleyin:

```json5
{
  tools: {
    allow: ["web_fetch"],
    // veya: allow: ["group:web"]  (web_fetch, web_search ve x_search içerir)
  },
}
```

## İlgili

- [Web Araması](/tr/tools/web) -- birden fazla sağlayıcıyla web'de arama yapın
- [Web Tarayıcısı](/tr/tools/browser) -- JS ağırlıklı siteler için tam tarayıcı otomasyonu
- [Firecrawl](/tr/tools/firecrawl) -- Firecrawl arama ve veri kazıma araçları
