---
read_when:
    - Web araması için Perplexity Search'ü kullanmak istiyorsunuz
    - PERPLEXITY_API_KEY veya OPENROUTER_API_KEY yapılandırması gereklidir
summary: web_search için Perplexity Search API ve Sonar/OpenRouter uyumluluğu
title: Perplexity araması
x-i18n:
    generated_at: "2026-07-26T23:05:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a7ca97355110e70a05f1d57acab475dda8dec89393804df40c6e9be5e30780e8
    source_path: tools/perplexity-search.md
    workflow: 16
---

OpenClaw, Perplexity Search API'yi bir `web_search` sağlayıcısı olarak destekler. `title`, `url` ve `snippet` alanlarıyla yapılandırılmış sonuçlar döndürür.

OpenClaw, uyumluluk amacıyla eski Perplexity Sonar/OpenRouter kurulumlarını da destekler. `OPENROUTER_API_KEY` kullanırsanız, `plugins.entries.perplexity.config.webSearch.apiKey` içinde bir `sk-or-...` anahtarı kullanırsanız veya `plugins.entries.perplexity.config.webSearch.baseUrl` / `model` ayarlarsanız sağlayıcı, sohbet tamamlama yoluna geçer ve yapılandırılmış Search API sonuçları yerine alıntılar içeren, yapay zekâ tarafından sentezlenmiş yanıtlar döndürür.

## Plugin'i yükleme

Resmî Plugin'i yükleyin, ardından Gateway'i yeniden başlatın:

```bash
openclaw plugins install @openclaw/perplexity-plugin
openclaw gateway restart
```

## Perplexity API anahtarı alma

1. [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api) adresinde bir Perplexity hesabı oluşturun.
2. Kontrol panelinde bir API anahtarı oluşturun.
3. Anahtarı yapılandırmada saklayın veya Gateway ortamında `PERPLEXITY_API_KEY` ayarlayın.

## OpenRouter uyumluluğu

Perplexity Sonar için zaten OpenRouter kullanıyorsanız `provider: "perplexity"` değerini koruyun ve Gateway ortamında `OPENROUTER_API_KEY` ayarlayın veya `plugins.entries.perplexity.config.webSearch.apiKey` içinde bir `sk-or-...` anahtarı saklayın.

İsteğe bağlı uyumluluk denetimleri:

- `plugins.entries.perplexity.config.webSearch.baseUrl`
- `plugins.entries.perplexity.config.webSearch.model`

## Yapılandırma örnekleri

### Yerel Perplexity Search API

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "pplx-...",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

### OpenRouter / Sonar uyumluluğu

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "<openrouter-api-key>",
            baseUrl: "https://openrouter.ai/api/v1",
            model: "perplexity/sonar-pro",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

## Anahtarın ayarlanacağı yer

**Yapılandırma aracılığıyla:** `openclaw configure --section web` komutunu çalıştırın. Bu komut, anahtarı `plugins.entries.perplexity.config.webSearch.apiKey` altında `~/.openclaw/openclaw.json` içinde saklar. Bu alan SecretRef nesnelerini de kabul eder.

**Ortam aracılığıyla:** Gateway işlem ortamında `PERPLEXITY_API_KEY` veya `OPENROUTER_API_KEY` ayarlayın. Bir Gateway kurulumu için bunu `~/.openclaw/.env` dosyasına (veya hizmet ortamınıza) ekleyin. Bkz. [Ortam değişkenleri](/tr/help/faq#env-vars-and-env-loading).

`provider: "perplexity"` yapılandırılmışsa ve Perplexity anahtarının SecretRef'i çözümlenemiyorsa ve ortam geri dönüşü yoksa başlatma/yeniden yükleme hızlıca başarısız olur.

## Araç parametreleri

Bu parametreler yerel Perplexity Search API yolu için geçerlidir.

<ParamField path="query" type="string" required>
Arama sorgusu.
</ParamField>

<ParamField path="count" type="number" default="5">
Döndürülecek sonuç sayısı (1-10).
</ParamField>

<ParamField path="country" type="string">
2 harfli ISO ülke kodu (ör. `US`, `DE`).
</ParamField>

<ParamField path="language" type="string">
ISO 639-1 dil kodu (ör. `en`, `de`, `fr`).
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
Zaman filtresi - `day` 24 saattir.
</ParamField>

<ParamField path="date_after" type="string">
Yalnızca bu tarihten sonra yayımlanan sonuçlar (`YYYY-MM-DD`).
</ParamField>

<ParamField path="date_before" type="string">
Yalnızca bu tarihten önce yayımlanan sonuçlar (`YYYY-MM-DD`).
</ParamField>

<ParamField path="domain_filter" type="string[]">
Etki alanı izin listesi/engelleme listesi dizisi (en fazla 20).
</ParamField>

<ParamField path="max_tokens" type="number" default="25000">
Toplam içerik bütçesi (en fazla 1000000).
</ParamField>

<ParamField path="max_tokens_per_page" type="number" default="2048">
Sayfa başına token sınırı.
</ParamField>

Eski Sonar/OpenRouter uyumluluk yolu için:

- `query`, `count` ve `freshness` kabul edilir.
- `count` burada yalnızca uyumluluk içindir; yanıt, N sonuçlu bir liste yerine yine alıntılar içeren tek bir sentezlenmiş yanıttır.
- Yalnızca Search API'ye özgü filtreler (`country`, `language`, `date_after`, `date_before`, `domain_filter`, `max_tokens`, `max_tokens_per_page`) açık hatalar döndürür.

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

// Etki alanı filtreleme (izin listesi)
await web_search({
  query: "iklim araştırması",
  domain_filter: ["nature.com", "science.org", ".edu"],
});

// Etki alanı filtreleme (engelleme listesi - başına - ekleyin)
await web_search({
  query: "ürün incelemeleri",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});

// Daha fazla içerik çıkarma
await web_search({
  query: "ayrıntılı yapay zekâ araştırması",
  max_tokens: 50000,
  max_tokens_per_page: 4096,
});
```

### Etki alanı filtresi kuralları

- Filtre başına en fazla 20 etki alanı.
- Aynı istekte izin listesi ve engelleme listesi girdileri birlikte kullanılamaz.
- Engelleme listesi girdileri için `-` önekini kullanın (ör. `["-reddit.com"]`).

## Notlar

- Perplexity Search API, yapılandırılmış web araması sonuçları döndürür (`title`, `url`, `snippet`).
- OpenRouter veya açıkça belirtilmiş bir `plugins.entries.perplexity.config.webSearch.baseUrl` / `model`, uyumluluk amacıyla Perplexity'yi yeniden Sonar sohbet tamamlamalarına geçirir.
- Sonar/OpenRouter uyumluluğu, yapılandırılmış sonuç satırları yerine alıntılar içeren tek bir sentezlenmiş yanıt döndürür.
- Sonuçlar varsayılan olarak 15 dakika önbelleğe alınır (`cacheTtlMinutes` aracılığıyla yapılandırılabilir).

## İlgili

<CardGroup cols={2}>
  <Card title="Web aramasına genel bakış" href="/tr/tools/web" icon="globe">
    Tüm sağlayıcılar ve otomatik algılama kuralları.
  </Card>
  <Card title="Brave araması" href="/tr/tools/brave-search" icon="shield">
    Ülke ve dil filtreleriyle yapılandırılmış sonuçlar.
  </Card>
  <Card title="Exa araması" href="/tr/tools/exa-search" icon="magnifying-glass">
    İçerik çıkarma özellikli sinir ağı tabanlı arama.
  </Card>
  <Card title="Perplexity Search API belgeleri" href="https://docs.perplexity.ai/docs/search/quickstart" icon="arrow-up-right-from-square">
    Resmî Perplexity Search API hızlı başlangıç kılavuzu ve başvuru belgeleri.
  </Card>
</CardGroup>
