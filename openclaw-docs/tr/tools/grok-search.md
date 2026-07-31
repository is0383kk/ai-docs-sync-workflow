---
read_when:
    - web_search için Grok kullanmak istiyorsunuz
    - Web araması için xAI OAuth veya bir XAI_API_KEY kullanmak istiyorsunuz
summary: xAI web tabanlı yanıtları aracılığıyla Grok web araması
title: Grok araması
x-i18n:
    generated_at: "2026-07-27T00:20:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6e39edd660d0ffe8be066ae81317810da691a7dbd8c59a74222a59145cff5c77
    source_path: tools/grok-search.md
    workflow: 16
---

OpenClaw, canlı arama sonuçlarıyla desteklenen ve atıflar içeren, yapay zekâ tarafından sentezlenmiş yanıtlar üretmek için xAI'ın web temellendirmeli
yanıtlarını kullanarak Grok'u bir `web_search` sağlayıcısı olarak destekler.

Grok web araması, mevcut olduğunda var olan bir xAI OAuth oturum açma profilini tercih eder.
OAuth profili yoksa aynı xAI API anahtarı, X (eski adıyla Twitter) gönderilerini aramaya yönelik yerleşik
`x_search` aracını ve `code_execution`
aracını da çalıştırır. Anahtarın `plugins.entries.xai.config.webSearch.apiKey` konumunda saklanması,
OpenClaw'un bunu paketle birlikte sunulan xAI model sağlayıcısı için yedek olarak yeniden kullanmasına da olanak tanır.

Gönderi düzeyindeki X ölçümleri (yeniden gönderiler, yanıtlar, yer imleri, görüntülemeler) için
geniş kapsamlı bir arama sorgusu yerine tam gönderi URL'si veya durum kimliğiyle
[`x_search`](/tr/tools/web#x_search) kullanın.

## İlk kurulum ve yapılandırma

`openclaw onboard` veya `openclaw configure --section
web` sırasında **Grok** seçildiğinde OpenClaw, ayrı bir web araması anahtarı istemeden mevcut bir xAI OAuth profilini yeniden kullanabilir. OAuth olmadan xAI API anahtarı kurulumuna geri döner.

Ardından OpenClaw, aynı xAI kimlik bilgisiyle `x_search` özelliğini etkinleştirmek için bir takip adımı sunar.
Bu takip adımı:

- yalnızca `web_search` için Grok'u seçtikten sonra görünür
- ayrı bir üst düzey web araması sağlayıcısı seçeneği değildir
- aynı akışta isteğe bağlı olarak `x_search` modelini ayarlayabilir

`x_search` özelliğini daha sonra yapılandırmada etkinleştirmek veya değiştirmek için bu adımı atlayın.

## Oturum açma veya API anahtarı edinme

<Steps>
  <Step title="xAI OAuth kullanma">
    İlk kurulum veya model kimlik doğrulaması sırasında xAI ile zaten oturum açtıysanız
    `web_search` sağlayıcısı olarak Grok'u seçin. Ayrı bir API anahtarı gerekmez:

    ```bash
    openclaw onboard --auth-choice xai-oauth
    openclaw config set tools.web.search.provider grok
    ```

  </Step>
  <Step title="API anahtarı yedeği kullanma">
    OAuth kullanılamadığında veya özellikle anahtarla desteklenen bir web araması yapılandırması
    istediğinizde [xAI](https://console.x.ai/) üzerinden bir API anahtarı edinin.
  </Step>
  <Step title="Anahtarı saklama">
    Gateway ortamında `XAI_API_KEY` değişkenini ayarlayın veya şununla yapılandırın:

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
      xai: {
        config: {
          webSearch: {
            apiKey: "xai-...", // xAI OAuth veya XAI_API_KEY kullanılabiliyorsa isteğe bağlıdır
            baseUrl: "https://api.x.ai/v1", // isteğe bağlı Responses API proxy/temel URL geçersiz kılma ayarı
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "grok",
      },
    },
  },
}
```

**Kimlik bilgisi alternatifleri:** Gateway ortamında `openclaw models auth login --provider xai
--method oauth`, `XAI_API_KEY` veya
`plugins.entries.xai.config.webSearch.apiKey`. Bir gateway kurulumu için ortam
değişkenlerini `~/.openclaw/.env` içine yerleştirin.

## Nasıl çalışır?

Grok, Gemini'ın Google Arama temellendirme yaklaşımına benzer şekilde satır içi
atıflar içeren yanıtlar sentezlemek için xAI'ın web temellendirmeli yanıtlarını kullanır.

## Desteklenen parametreler

Grok araması `query` parametresini destekler. `count`, paylaşılan `web_search`
uyumluluğu için kabul edilir ancak Grok, N sonuçtan oluşan bir liste yerine her zaman
atıflar içeren tek bir sentezlenmiş yanıt döndürür. Sağlayıcıya özgü filtreler desteklenmez.

xAI Responses web temellendirmeli aramaları, paylaşılan `web_search` varsayılanından daha uzun
sürebildiği için Grok varsayılan olarak 60 saniyelik zaman aşımı kullanır. Bunu
`tools.web.search.timeoutSeconds` ile geçersiz kılın.

## Temel URL geçersiz kılma ayarları

Grok web aramasını bir operatör proxy'si veya xAI uyumlu Responses uç noktası
üzerinden yönlendirmek için `plugins.entries.xai.config.webSearch.baseUrl` ayarını yapın. OpenClaw,
sondaki eğik çizgileri kaldırdıktan sonra `<baseUrl>/responses` adresine gönderi yapar. `x_search`,
`plugins.entries.xai.config.xSearch.baseUrl` ayarlanmadığı sürece aynı `webSearch.baseUrl` değerine
geri döner.

## İlgili

- [Web Aramasına genel bakış](/tr/tools/web) -- tüm sağlayıcılar ve otomatik algılama
- [Web Aramasında x_search](/tr/tools/web#x_search) -- xAI aracılığıyla birinci sınıf X araması
- [Gemini Araması](/tr/tools/gemini-search) -- Google temellendirmesi aracılığıyla yapay zekâ tarafından sentezlenmiş yanıtlar
