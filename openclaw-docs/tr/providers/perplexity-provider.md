---
read_when:
    - Perplexity'yi bir web arama sağlayıcısı olarak yapılandırmak istiyorsunuz
    - Perplexity API anahtarına veya OpenRouter proxy yapılandırmasına ihtiyacınız var
summary: Perplexity web arama sağlayıcısı kurulumu (API anahtarı, arama modları, filtreleme)
title: Perplexity
x-i18n:
    generated_at: "2026-07-26T22:59:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea76a5cb7befce95756e9bcc8f9c1637fac87711d02d8a486ec2a1b9f51b73dc
    source_path: providers/perplexity-provider.md
    workflow: 16
---

Perplexity Plugin'i, iki aktarım mekanizmasına sahip bir `web_search` sağlayıcısı kaydeder:
yerel Perplexity Search API'si (filtreli yapılandırılmış sonuçlar) ve doğrudan ya da
OpenRouter üzerinden Perplexity Sonar sohbet tamamlamaları (alıntılar içeren,
yapay zekâ tarafından sentezlenmiş yanıtlar).

<Note>
Bu sayfa Perplexity **sağlayıcı** kurulumunu ele alır. Perplexity **aracı** (agent'ın bunu nasıl kullandığı) için [Perplexity araması](/tr/tools/perplexity-search) bölümüne bakın.
</Note>

| Özellik     | Değer                                                                  |
| ----------- | ---------------------------------------------------------------------- |
| Tür         | Web araması sağlayıcısı (model sağlayıcısı değildir)                   |
| Kimlik doğrulama | `PERPLEXITY_API_KEY` (yerel) veya `OPENROUTER_API_KEY` (OpenRouter üzerinden) |
| Yapılandırma yolu | `plugins.entries.perplexity.config.webSearch.apiKey`                   |
| Geçersiz kılmalar | `plugins.entries.perplexity.config.webSearch.baseUrl` / `.model`       |
| Anahtar edinme | [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api)   |

## Plugin'i yükleme

```bash
openclaw plugins install @openclaw/perplexity-plugin
openclaw gateway restart
```

## Başlarken

<Steps>
  <Step title="API anahtarını ayarlayın">
    ```bash
    openclaw configure --section web
    ```

    Alternatif olarak anahtarı doğrudan ayarlayın:

    ```bash
    openclaw config set plugins.entries.perplexity.config.webSearch.apiKey "pplx-xxxxxxxxxxxx"
    ```

    Gateway ortamında `PERPLEXITY_API_KEY` veya `OPENROUTER_API_KEY` olarak dışa
    aktarılan bir anahtar da çalışır.

  </Step>
  <Step title="Aramaya başlayın">
    `web_search`, anahtarı kullanılabilir arama kimlik bilgisi olduğunda
    Perplexity'yi otomatik olarak algılar; başka bir kurulum gerekmez. Sağlayıcıyı açıkça sabitlemek için:

    ```bash
    openclaw config set tools.web.search.provider perplexity
    ```

  </Step>
</Steps>

## Arama modları

Plugin, aktarım mekanizmasını şu sırayla çözümler:

1. `webSearch.baseUrl` veya `webSearch.model` ayarlanmışsa: anahtar türünden bağımsız olarak her zaman söz konusu uç noktaya yönelik Sonar sohbet tamamlamaları üzerinden yönlendirir.
2. Aksi takdirde uç noktayı anahtarın kaynağı belirler: yapılandırılmış bir anahtarın ön eki aktarım mekanizmasını seçer (yapılandırma, ortam değişkenlerine göre önceliklidir); ortam anahtarı ise doğrudan eşleşen uç noktasını kullanır.

| Anahtar ön eki | Aktarım mekanizması                                      | Özellikler                                        |
| -------------- | ------------------------------------------------------- | ------------------------------------------------- |
| `pplx-`    | Yerel Perplexity Search API'si (`https://api.perplexity.ai`) | Yapılandırılmış sonuçlar, alan adı/dil/tarih filtreleri |
| `sk-or-`   | OpenRouter (`https://openrouter.ai/api/v1`), Sonar modeli       | Alıntılar içeren, yapay zekâ tarafından sentezlenmiş yanıtlar |

Başka herhangi bir ön eke sahip yapılandırılmış anahtar da yerel Search API'sini
kullanır. Sohbet tamamlamaları yolu varsayılan olarak `perplexity/sonar-pro`
modelini kullanır; bunu `plugins.entries.perplexity.config.webSearch.model` ile geçersiz kılın.

## Yerel API filtreleme

| Filtre                               | Açıklama                                                        | Aktarım mekanizması |
| ------------------------------------ | --------------------------------------------------------------- | ------------------- |
| `count`                              | Arama başına sonuç sayısı, 1-10 (varsayılan 5)                  | Yalnızca yerel      |
| `freshness`                          | Güncellik aralığı: `day`, `week`, `month`, `year`                  | Her ikisi           |
| `country`                            | 2 harfli ülke kodu (`us`, `de`, `jp`)                        | Yalnızca yerel      |
| `language`                           | ISO 639-1 dil kodu (`en`, `fr`, `zh`)                      | Yalnızca yerel      |
| `date_after` / `date_before`         | `YYYY-MM-DD` biçiminde yayımlanma tarihi aralığı             | Yalnızca yerel      |
| `domain_filter`                      | En fazla 20 alan adı; izin verilenler listesi veya `-` ön ekli engellenenler listesi, birlikte kullanılamaz | Yalnızca yerel      |
| `max_tokens` / `max_tokens_per_page` | Tüm sonuçlar / sayfa başına içerik bütçesi                       | Yalnızca yerel      |

Yalnızca yerel aktarıma özgü filtreler, sohbet tamamlamaları yolunda açıklayıcı
bir hata döndürür. `freshness`, `date_after`/`date_before`
ile birlikte kullanılamaz.

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Daemon işlemleri için ortam değişkeni">
    <Warning>
    Yalnızca etkileşimli bir kabukta dışa aktarılan anahtar, söz konusu ortam
    açıkça içe aktarılmadığı sürece launchd/systemd Gateway daemon'ı tarafından
    görülemez. Gateway işleminin anahtarı okuyabilmesi için anahtarı
    `~/.openclaw/.env` içinde veya `env.shellEnv` aracılığıyla ayarlayın. Tam
    öncelik sırası için [Ortam değişkenleri](/tr/help/environment) bölümüne bakın.
    </Warning>
  </Accordion>

  <Accordion title="OpenRouter proxy kurulumu">
    Perplexity aramalarını OpenRouter üzerinden yönlendirmek için yerel Perplexity
    anahtarı yerine bir `OPENROUTER_API_KEY` (`sk-or-` ön ekli) ayarlayın.
    OpenClaw anahtarı algılar ve otomatik olarak Sonar aktarım mekanizmasına geçer.
    OpenRouter faturalandırmasını zaten ayarladıysanız ve sağlayıcıları burada
    birleştirmek istiyorsanız kullanışlıdır.
  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Perplexity arama aracı" href="/tr/tools/perplexity-search" icon="magnifying-glass">
    Agent'ın Perplexity aramalarını nasıl çağırdığı ve sonuçları nasıl yorumladığı.
  </Card>
  <Card title="Yapılandırma başvurusu" href="/tr/gateway/configuration-reference" icon="gear">
    Plugin girdileri dâhil tam yapılandırma başvurusu.
  </Card>
</CardGroup>
