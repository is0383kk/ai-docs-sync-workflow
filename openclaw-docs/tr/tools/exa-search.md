---
read_when:
    - web_search için Exa kullanmak istiyorsunuz
    - Bir EXA_API_KEY gereklidir
    - Sinirsel arama veya içerik çıkarma istiyorsunuz
summary: Exa AI araması -- içerik çıkarma özellikli sinirsel ve anahtar kelime araması
title: Exa araması
x-i18n:
    generated_at: "2026-07-27T00:20:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3ddfd6fb471f92e705facf5a2d02361c1a343b9032fa8e0a7b135af634df65b7
    source_path: tools/exa-search.md
    workflow: 16
---

[Exa AI](https://exa.ai/), sinirsel, anahtar kelime ve hibrit arama modlarının yanı sıra yerleşik içerik çıkarma (öne çıkan bölümler, metin, özetler) özelliklerine sahip bir `web_search` sağlayıcısıdır.

## Plugin'i yükleme

```bash
openclaw plugins install @openclaw/exa-plugin
openclaw gateway restart
```

## API anahtarı alma

<Steps>
  <Step title="Hesap oluşturma">
    [exa.ai](https://exa.ai/) adresinde kaydolun ve kontrol panelinizden bir API anahtarı oluşturun.
  </Step>
  <Step title="Anahtarı saklama">
    Gateway ortamında `EXA_API_KEY` değerini ayarlayın veya şununla yapılandırın:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## Yapılandırma

```json5
{
  plugins: {
    entries: {
      exa: {
        config: {
          webSearch: {
            apiKey: "exa-...", // EXA_API_KEY ayarlanmışsa isteğe bağlı
            baseUrl: "https://api.exa.ai", // isteğe bağlı; OpenClaw /search ekler
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "exa",
      },
    },
  },
}
```

**Ortam alternatifi:** Gateway ortamında `EXA_API_KEY` değerini ayarlayın. Bir gateway kurulumu için bunu `~/.openclaw/.env` içine yerleştirin. Bkz. [Ortam değişkenleri](/tr/help/faq#env-vars-and-env-loading).

## Temel URL'yi geçersiz kılma

Exa arama isteklerini uyumlu bir proxy veya alternatif uç nokta üzerinden yönlendirmek için `plugins.entries.exa.config.webSearch.baseUrl` değerini ayarlayın. OpenClaw, çıplak ana bilgisayar adlarının başına `https://` ekleyerek bunları normalleştirir ve yol zaten bununla bitmiyorsa `/search` ekler. Çözümlenen uç nokta, arama önbelleği anahtarının bir parçasıdır; dolayısıyla farklı uç noktalardan gelen sonuçlar hiçbir zaman paylaşılmaz.

## Araç parametreleri

<ParamField path="query" type="string" required>
Arama sorgusu.
</ParamField>

<ParamField path="count" type="number" default="5">
Döndürülecek sonuçlar (1-100, Exa arama türü sınırlarına tabidir).
</ParamField>

<ParamField path="type" type="'auto' | 'neural' | 'fast' | 'deep' | 'deep-reasoning' | 'instant'">
Arama modu.
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
Zaman filtresi. `date_after`/`date_before` ile birlikte kullanılamaz.
</ParamField>

<ParamField path="date_after" type="string">
Bu tarihten sonraki sonuçlar (`YYYY-MM-DD`).
</ParamField>

<ParamField path="date_before" type="string">
Bu tarihten önceki sonuçlar (`YYYY-MM-DD`).
</ParamField>

<ParamField path="contents" type="object">
İçerik çıkarma seçenekleri (aşağıya bakın).
</ParamField>

### İçerik çıkarma

Sonuçlarda çıkarılan içeriği denetlemek için bir `contents` nesnesi iletin:

```javascript
await web_search({
  query: "transformer mimarisi açıklaması",
  type: "neural",
  contents: {
    text: true, // tam sayfa metni
    highlights: { numSentences: 3 }, // temel cümleler
    summary: true, // AI özeti
  },
});
```

| İçerik seçeneği | Tür                                                                   | Açıklama                   |
| --------------- | --------------------------------------------------------------------- | -------------------------- |
| `text`          | `boolean \| { maxCharacters }`                                        | Tam sayfa metnini çıkarır  |
| `highlights`    | `boolean \| { maxCharacters, query, numSentences, highlightsPerUrl }` | Temel cümleleri çıkarır    |
| `summary`       | `boolean \| { query }`                                                | AI tarafından oluşturulan özet |

`contents` belirtilmezse Exa varsayılan olarak `{ highlights: true }` kullanır; böylece sonuçlar temel cümle alıntılarını içerir. Sonuç açıklamaları önce öne çıkan bölümlerden, ardından özetten, sonra da tam metinden olmak üzere kullanılabilir olan ilk kaynaktan çözümlenir. Sonuçlar ayrıca mevcut olduğunda Exa API yanıtındaki ham `highlightScores` ve `summary` alanlarını korur.

### Arama modları

| Mod              | Açıklama                                  |
| ---------------- | ----------------------------------------- |
| `auto`           | Exa en iyi modu seçer (varsayılan)        |
| `neural`         | Anlamsal/anlama dayalı arama              |
| `fast`           | Hızlı anahtar kelime araması              |
| `deep`           | Kapsamlı derin arama                      |
| `deep-reasoning` | Akıl yürütmeli derin arama                |
| `instant`        | En hızlı sonuçlar                         |

## Notlar

- `count`, Exa arama türü sınırlarına tabi olarak en fazla 100 değerini kabul eder.
- Sonuçlar varsayılan olarak 15 dakika önbelleğe alınır. Exa dahil tüm `web_search` sağlayıcılarının önbelleğe alma ve istek zaman aşımını değiştirmek için ortak `tools.web.search.cacheTtlMinutes` (dakika) ve `tools.web.search.timeoutSeconds` (varsayılan 30 sn.) değerlerini yapılandırın.

## İlgili

- [Web Aramasına genel bakış](/tr/tools/web) -- tüm sağlayıcılar ve otomatik algılama
- [Brave Search](/tr/tools/brave-search) -- ülke/dil filtreleriyle yapılandırılmış sonuçlar
- [Perplexity Search](/tr/tools/perplexity-search) -- alan adı filtrelemeli yapılandırılmış sonuçlar
