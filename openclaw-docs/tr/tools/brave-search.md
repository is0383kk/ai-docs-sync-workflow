---
read_when:
    - web_search için Brave Search'ü kullanmak istiyorsunuz
    - Bir BRAVE_API_KEY veya plan ayrıntıları gereklidir
summary: web_search için Brave Search API kurulumu
title: Brave araması
x-i18n:
    generated_at: "2026-07-27T00:06:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52168db93abb564eda5868584261e0530ce3cff57c3463a2fc1eded351df30f2
    source_path: tools/brave-search.md
    workflow: 16
---

OpenClaw, bir `web_search` sağlayıcısı olarak Brave Search API'yi destekler.

## API anahtarı edinme

1. [https://brave.com/search/api/](https://brave.com/search/api/) adresinde bir Brave Search API hesabı oluşturun.
2. Kontrol panelinde **Search** planını seçin ve bir API anahtarı oluşturun.
3. Anahtarı yapılandırmada saklayın veya Gateway ortamında `BRAVE_API_KEY` değerini ayarlayın.

## Yapılandırma örneği

```json5
{
  plugins: {
    entries: {
      brave: {
        config: {
          webSearch: {
            apiKey: "BRAVE_API_KEY_HERE",
            mode: "web", // veya "llm-context"
            baseUrl: "https://api.search.brave.com", // isteğe bağlı proxy/temel URL geçersiz kılması
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "brave",
        maxResults: 5,
        timeoutSeconds: 30,
      },
    },
  },
}
```

Sağlayıcıya özgü Brave arama ayarları `plugins.entries.brave.config.webSearch.*` altında bulunur; standart yapılandırma yolu budur.

`webSearch.mode`, Brave aktarımını denetler:

- `web` (varsayılan): başlıklar, URL'ler ve özet parçacıklarıyla normal Brave web araması
- `llm-context`: temellendirme için önceden ayıklanmış metin parçaları ve kaynaklar sunan Brave LLM Context API

`webSearch.baseUrl`, Brave isteklerini güvenilir bir Brave uyumlu proxy'ye
veya gateway'e yönlendirebilir. OpenClaw, yapılandırılmış temel URL'ye
`/res/v1/web/search` ya da `/res/v1/llm/context` ekler ve temel URL'yi önbellek anahtarında tutar. Herkese açık
uç noktalar `https://` kullanmalıdır; `http://` yalnızca güvenilir geri döngü
veya özel ağ proxy sunucuları için kabul edilir.

## Araç parametreleri

<ParamField path="query" type="string" required>
Arama sorgusu.
</ParamField>

<ParamField path="count" type="number" default="5">
Döndürülecek sonuç sayısı (1–10).
</ParamField>

<ParamField path="country" type="string">
2 harfli ISO ülke kodu (ör. `US`, `DE`).
</ParamField>

<ParamField path="language" type="string">
Arama sonuçları için ISO 639-1 dil kodu (ör. `en`, `de`, `fr`).
</ParamField>

<ParamField path="search_lang" type="string">
Brave arama dili kodu (ör. `en`, `en-gb`, `zh-hans`).
</ParamField>

<ParamField path="ui_lang" type="string">
Kullanıcı arayüzü öğeleri için ISO dil kodu.
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
Zaman filtresi — `day` 24 saattir.
</ParamField>

<ParamField path="date_after" type="string">
Yalnızca bu tarihten sonra yayımlanan sonuçlar (`YYYY-MM-DD`).
</ParamField>

<ParamField path="date_before" type="string">
Yalnızca bu tarihten önce yayımlanan sonuçlar (`YYYY-MM-DD`).
</ParamField>

**Örnekler:**

```javascript
// Ülkeye ve dile özgü arama
await web_search({
  query: "yenilenebilir enerji",
  country: "DE",
  language: "de",
});

// Yakın tarihli sonuçlar (geçen hafta)
await web_search({
  query: "yapay zekâ haberleri",
  freshness: "week",
});

// Tarih aralığı araması
await web_search({
  query: "yapay zekâ gelişmeleri",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});
```

## Notlar

- OpenClaw, Brave **Search** planını kullanır. Eski bir aboneliğiniz varsa (ör. ayda 2.000 sorgu sunan özgün Free planı), aboneliğiniz geçerliliğini korur ancak LLM Context veya daha yüksek hız sınırları gibi yeni özellikleri içermez.
- Her Brave planı, her ay yenilenen **aylık \$5 ücretsiz kredi** içerir. Search planının maliyeti 1.000 istek başına \$5 olduğundan kredi, ayda 1.000 sorguyu karşılar. Beklenmedik ücretlerden kaçınmak için Brave kontrol panelinde kullanım sınırınızı ayarlayın. Güncel planlar için [Brave API portalına](https://brave.com/search/api/) bakın.
- Search planı, LLM Context uç noktasını ve yapay zekâ çıkarımı haklarını içerir. Modelleri eğitmek veya ince ayarlamak amacıyla sonuçları depolamak, açık depolama haklarına sahip bir plan gerektirir. Brave [Hizmet Şartları](https://api-dashboard.search.brave.com/terms-of-service) belgesine bakın.
- `llm-context` modu, normal web araması özet parçacığı biçimi yerine temellendirilmiş kaynak girdileri döndürür.
- `llm-context` modu, `freshness` ve sınırlandırılmış `date_after` + `date_before` aralıklarını destekler. `ui_lang` değerini desteklemez; Brave, özel güncellik aralıklarının hem başlangıç hem de bitiş tarihini içermesini gerektirdiğinden `date_after` olmadan `date_before` reddedilir.
- `ui_lang`, `en-US` gibi bir bölge alt etiketi içermelidir.
- Sonuçlar varsayılan olarak 15 dakika önbelleğe alınır (`cacheTtlMinutes` aracılığıyla yapılandırılabilir).
- Özel `webSearch.baseUrl` değerleri Brave önbellek kimliğine dahil edilir; böylece
  proxy'ye özgü yanıtlar çakışmaz.
- Sorun giderirken Brave istek URL'lerini/sorgu parametrelerini, yanıt durumunu/zamanlamasını ve arama önbelleği isabet/ıskalama/yazma olaylarını günlüğe kaydetmek için `brave.http` tanılama bayrağını etkinleştirin. Bayrak, API anahtarını veya yanıt gövdelerini hiçbir zaman günlüğe kaydetmez; ancak arama sorguları hassas olabilir.

## İlgili içerik

- [Web Aramasına genel bakış](/tr/tools/web) -- tüm sağlayıcılar ve otomatik algılama
- [Perplexity Search](/tr/tools/perplexity-search) -- alan adı filtrelemeli yapılandırılmış sonuçlar
- [Exa Search](/tr/tools/exa-search) -- içerik ayıklamalı sinir ağı tabanlı arama
