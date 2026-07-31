---
read_when:
    - API anahtarı gerektirmeyen bir web arama sağlayıcısı istiyorsunuz
    - web_search için DuckDuckGo kullanmak istiyorsunuz
    - Açıkça seçilmiş, anahtar gerektirmeyen bir arama sağlayıcısı istiyorsunuz
summary: DuckDuckGo web araması -- anahtar gerektirmeyen sağlayıcı (deneysel, HTML tabanlı)
title: DuckDuckGo araması
x-i18n:
    generated_at: "2026-07-27T00:07:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 84e90532de276dcb3f73c67015dffe5f5a62be673e44a19053b2b1dfcb0986ac
    source_path: tools/duckduckgo-search.md
    workflow: 16
---

OpenClaw, DuckDuckGo'yu **anahtar gerektirmeyen** bir `web_search` sağlayıcısı olarak destekler. API anahtarı veya hesap gerekmez.

<Warning>
  DuckDuckGo, resmi bir API değil, DuckDuckGo'nun JavaScript kullanmayan HTML arama sayfalarından veri ayıklayan **deneysel ve resmi olmayan** bir entegrasyondur. Bot doğrulama sayfaları veya HTML değişiklikleri nedeniyle zaman zaman bozulması beklenebilir.
</Warning>

## Kurulum

Otomatik algılama yalnızca kullanılabilir kimlik bilgilerine sahip sağlayıcıları dikkate aldığından DuckDuckGo hiçbir zaman otomatik olarak seçilmez. Açıkça ayarlayın:

<Steps>
  <Step title="Yapılandırma">
    ```bash
    openclaw configure --section web
    # Sağlayıcı olarak "duckduckgo"yu seçin
    ```
  </Step>
</Steps>

## Yapılandırma

Sağlayıcıyı doğrudan yapılandırmada ayarlayın:

```json5
{
  tools: {
    web: {
      search: {
        provider: "duckduckgo",
      },
    },
  },
}
```

Bölge ve SafeSearch için isteğe bağlı Plugin düzeyi ayarları:

```json5
{
  plugins: {
    entries: {
      duckduckgo: {
        config: {
          webSearch: {
            region: "us-en", // DuckDuckGo bölge kodu
            safeSearch: "moderate", // "strict", "moderate" veya "off"
          },
        },
      },
    },
  },
}
```

## Araç parametreleri

<ParamField path="query" type="string" required>
Arama sorgusu.
</ParamField>

<ParamField path="count" type="number" default="5">
Döndürülecek sonuçlar (1-10).
</ParamField>

<ParamField path="region" type="string">
DuckDuckGo bölge kodu (ör. `us-en`, `uk-en`, `de-de`).
</ParamField>

<ParamField path="safeSearch" type="'strict' | 'moderate' | 'off'" default="moderate">
SafeSearch düzeyi.
</ParamField>

`region` ve `safeSearch` araç parametreleri, yukarıdaki Plugin yapılandırma değerlerini sorgu bazında geçersiz kılar.

## Notlar

- **API anahtarı yok** -- DuckDuckGo, `web_search` sağlayıcısı olarak seçildiğinde çalışır.
- **Deneysel** -- resmi bir API veya SDK kullanmak yerine DuckDuckGo'nun JavaScript kullanmayan HTML arama sayfalarından veri ayıklar. Sonuçlar, bildirimde bulunulmadan değişebilecek sayfa yapısına bağlıdır.
- **Bot doğrulama riski** -- DuckDuckGo, yoğun veya otomatik kullanımda CAPTCHA sunabilir ya da istekleri engelleyebilir.
- **Yalnızca açık seçim** -- OpenClaw'ın otomatik algılaması yalnızca kullanılabilir kimlik bilgilerine sahip sağlayıcıları dikkate alır; bu nedenle DuckDuckGo gibi anahtar gerektirmeyen bir sağlayıcı hiçbir zaman otomatik olarak seçilmez. `provider: "duckduckgo"` değerini ayarlamanız gerekir.
- Yapılandırılmadığında **SafeSearch varsayılan olarak `moderate` değerini kullanır**.

<Tip>
  Üretim kullanımı için [Brave Search](/tr/tools/brave-search) (ücretsiz katman mevcuttur) veya API destekli başka bir sağlayıcı kullanmayı değerlendirin.
</Tip>

## İlgili

- [Web Aramasına genel bakış](/tr/tools/web) -- tüm sağlayıcılar ve otomatik algılama
- [Brave Search](/tr/tools/brave-search) -- ücretsiz katmanla yapılandırılmış sonuçlar
- [Exa Search](/tr/tools/exa-search) -- içerik çıkarma özellikli sinir ağı tabanlı arama
