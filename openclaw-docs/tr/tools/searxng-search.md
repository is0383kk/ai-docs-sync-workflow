---
read_when:
    - Kendi barındırdığınız bir web arama sağlayıcısı istiyorsunuz
    - Web araması için SearXNG kullanmak istiyorsunuz
    - Gizlilik odaklı veya dış ağlardan yalıtılmış bir arama seçeneğine ihtiyacınız var
summary: SearXNG web araması -- kendi barındırdığınız, anahtar gerektirmeyen meta arama sağlayıcısı
title: SearXNG araması
x-i18n:
    generated_at: "2026-07-27T00:09:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cae8de9f8e2c8dd9cec615adb48da5c1fd7654bffe96c7afc1acea3effbcf1fc
    source_path: tools/searxng-search.md
    workflow: 16
---

OpenClaw, Google, Bing, DuckDuckGo ve diğer kaynaklardan gelen sonuçları bir araya getiren açık kaynaklı bir meta arama motoru olan [SearXNG](https://docs.searxng.org/)'yi **kendi sunucunuzda barındırılan,
anahtar gerektirmeyen** `web_search` sağlayıcısı olarak destekler.

Avantajlar:

- **Ücretsiz ve sınırsız** -- API anahtarı veya ticari abonelik gerekmez
- **Gizlilik / ağ yalıtımı** -- sorgular hiçbir zaman ağınızdan çıkmaz
- **Her yerde çalışır** -- ticari arama API'lerindeki bölge kısıtlamaları yoktur

## Kurulum

<Steps>
  <Step title="Plugin'i yükleyin">
    ```bash
    openclaw plugins install @openclaw/searxng-plugin
    ```
  </Step>
  <Step title="Bir SearXNG örneği çalıştırın">
    ```bash
    docker run -d -p 8888:8080 searxng/searxng
    ```

    Alternatif olarak, erişiminiz olan mevcut herhangi bir SearXNG dağıtımını kullanın. Üretim kurulumu için
    [SearXNG belgelerine](https://docs.searxng.org/) bakın.

  </Step>
  <Step title="Yapılandırın">
    ```bash
    openclaw configure --section web
    # Sağlayıcı olarak "searxng" seçeneğini belirleyin
    ```

    Alternatif olarak, ortam değişkenini ayarlayın ve otomatik algılamanın bunu bulmasına izin verin:

    ```bash
    export SEARXNG_BASE_URL="http://localhost:8888"
    ```

  </Step>
</Steps>

## Yapılandırma

```json5
{
  tools: {
    web: {
      search: {
        provider: "searxng",
      },
    },
  },
}
```

SearXNG örneği için Plugin düzeyindeki ayarlar:

```json5
{
  plugins: {
    entries: {
      searxng: {
        config: {
          webSearch: {
            baseUrl: "http://localhost:8888",
            categories: "general,news", // isteğe bağlı
            language: "en", // isteğe bağlı
          },
        },
      },
    },
  },
}
```

`baseUrl`, bir SecretRef nesnesini de kabul eder (örneğin `{ source: "env", id: "SEARXNG_BASE_URL" }`).

## Ortam değişkeni

Yapılandırmaya alternatif olarak `SEARXNG_BASE_URL` değişkenini ayarlayın:

```bash
export SEARXNG_BASE_URL="http://localhost:8888"
```

Çözümleme sırası: yapılandırılmış `baseUrl` dizesi, ardından
`baseUrl` üzerindeki satır içi ortam SecretRef'i, ardından `SEARXNG_BASE_URL`. Yapılandırma yollarından hiçbiri ayarlanmamışsa,
`SEARXNG_BASE_URL` mevcutsa ve açıkça bir sağlayıcı seçilmemişse otomatik algılama
SearXNG'yi seçer.

## Plugin yapılandırma başvurusu

| Alan         | Açıklama                                                           |
| ------------ | ------------------------------------------------------------------ |
| `baseUrl`    | SearXNG örneğinizin temel URL'si (zorunlu)                          |
| `categories` | `general`, `news` veya `science` gibi virgülle ayrılmış kategoriler |
| `language`   | Sonuçlar için `en`, `de` veya `fr` gibi dil kodu              |

`web_search` araç çağrısı, çağrı başına geçersiz kılmalar olarak `count` (1-10 sonuç), `categories`
ve `language` değerlerini de kabul eder.

## Notlar

- **JSON API** -- HTML kazıma yerine SearXNG'nin yerel `format=json` uç noktasını kullanır
- **Görsel sonuç URL'leri** -- SearXNG doğrudan bir görsel URL'si döndürdüğünde, görsel kategorisindeki sonuçlar
  `img_src` içerir
- **API anahtarı yoktur** -- herhangi bir SearXNG örneğiyle kullanıma hazır olarak çalışır
- **Temel URL doğrulaması** -- `baseUrl`, geçerli bir `http://` veya `https://`
  URL'si olmalıdır
- **Ağ koruması** -- `http://` temel URL'leri güvenilen bir özel veya
  geri döngü ana bilgisayarını hedeflemelidir (genel ana bilgisayarlar `https://` kullanmalıdır); özel/dahili bir adrese
  çözümlenen `https://` temel URL'leri aynı kendi sunucusunda barındırma iznini alırken,
  genel bir adrese çözümlenen `https://` temel URL'leri katı SSRF korumasını sürdürür
- **Otomatik algılama sırası** -- SearXNG, yapılandırılmış bir `baseUrl` gerektirir (gerekli kimlik bilgilerine
  zaten sahip sağlayıcılar arasında sıra 200). DuckDuckGo veya Ollama Web Search gibi anahtar gerektirmeyen
  sağlayıcılar otomatik algılamada hiçbir zaman örtük olarak seçilmez;
  yalnızca açık bir `provider` seçimiyle etkinleşir
- **Kendi sunucunuzda barındırılır** -- örneği, sorguları ve yukarı akış arama motorlarını siz denetlersiniz
- **Kategoriler**, yapılandırılmadığında varsayılan olarak `general` değerini kullanır
- **Kategori yedeği** -- `general` dışındaki bir kategori isteği başarılı olur ancak
  sıfır sonuç döndürürse OpenClaw, boş bir sonuç kümesi döndürmeden önce aynı sorguyu `general` ile
  bir kez yeniden dener
- **Sonuç önbelleğe alma** -- aynı sorgular (aynı sorgu, sayı, kategoriler,
  dil ve temel URL), kısa bir TTL boyunca işlem içinde önbelleğe alınır
- **Sürüm gereksinimi** -- Plugin, `minHostVersion: >=2026.6.9` bildirir

<Tip>
  SearXNG JSON API'nin çalışması için SearXNG örneğinizde `json`
  biçiminin `settings.yml` dosyasındaki `search.formats` altında etkinleştirildiğinden emin olun.
</Tip>

## İlgili

- [Web Aramasına genel bakış](/tr/tools/web) -- tüm sağlayıcılar ve otomatik algılama
- [DuckDuckGo Araması](/tr/tools/duckduckgo-search) -- anahtar gerektirmeyen başka bir sağlayıcı
- [Brave Araması](/tr/tools/brave-search) -- ücretsiz katmanla yapılandırılmış sonuçlar
