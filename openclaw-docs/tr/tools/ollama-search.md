---
read_when:
    - web_search için Ollama kullanmak istiyorsunuz
    - Anahtar gerektirmeyen bir web_search sağlayıcısı istiyorsunuz
    - OLLAMA_API_KEY ile barındırılan Ollama Web Search'ü kullanmak istiyorsunuz
    - Ollama Web Search kurulum rehberine ihtiyacınız var
summary: Yerel bir Ollama ana bilgisayarı veya barındırılan Ollama API'si üzerinden Ollama Web Araması
title: Ollama web araması
x-i18n:
    generated_at: "2026-07-26T23:39:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: edbbd887841339ab4c0c62ab7682a22fe99434a788957a91989fce6942187e9a
    source_path: tools/ollama-search.md
    workflow: 16
---

OpenClaw, paketle birlikte sunulan bir `web_search` sağlayıcısı olarak **Ollama Web Search**'ü destekler ve Ollama'nın web arama API'sinden başlıkları, URL'leri ve parçacıkları döndürür.

Yerel/kendi barındırdığınız Ollama için varsayılan olarak API anahtarı gerekmez; erişilebilir bir Ollama ana bilgisayarı ve `ollama signin` gerekir. Doğrudan barındırılan arama (yerel Ollama olmadan) için `baseUrl: "https://ollama.com"` ve gerçek bir `OLLAMA_API_KEY` gerekir.

## Kurulum

<Steps>
  <Step title="Ollama'yı başlatın">
    Ollama'nın yüklü ve çalışır durumda olduğundan emin olun.
  </Step>
  <Step title="Oturum açın">
    ```bash
    ollama signin
    ```
  </Step>
  <Step title="Ollama Web Search'ü seçin">
    ```bash
    openclaw configure --section web
    ```

    Sağlayıcı olarak **Ollama Web Search**'ü seçin.

  </Step>
</Steps>

Ollama'yı modeller için zaten kullanıyorsanız Ollama Web Search, yapılandırılmış aynı ana bilgisayarı yeniden kullanır.

<Note>
  OpenClaw, Ollama Web Search'ü daha yüksek öncelikli, kimlik bilgileriyle yapılandırılmış bir sağlayıcıya tercih ederek hiçbir zaman otomatik olarak seçmez; `tools.web.search.provider: "ollama"` ile açıkça seçmeniz gerekir.
</Note>

## Yapılandırma

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

Yalnızca web aramasıyla sınırlı isteğe bağlı ana bilgisayar geçersiz kılma ayarı:

```json5
{
  plugins: {
    entries: {
      ollama: {
        config: {
          webSearch: {
            baseUrl: "http://ollama-host:11434",
          },
        },
      },
    },
  },
}
```

Ya da Ollama model sağlayıcısı için zaten yapılandırılmış ana bilgisayarı yeniden kullanın:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
      },
    },
  },
}
```

`models.providers.ollama.baseUrl` kurallı anahtardır; web arama sağlayıcısı, OpenAI SDK tarzı yapılandırma örnekleriyle uyumluluk için burada `baseURL` değerini de kabul eder. Hiçbir şey ayarlanmazsa OpenClaw varsayılan olarak `http://127.0.0.1:11434` değerini kullanır.

Doğrudan barındırılan Ollama Web Search (yerel Ollama olmadan):

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "https://ollama.com",
        apiKey: "OLLAMA_API_KEY",
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

## Kimlik doğrulama ve istek yönlendirme

- Web aramasına özel bir API anahtarı alanı yoktur; yapılandırılmış ana bilgisayar kimlik doğrulama ile korunuyorsa sağlayıcı, `models.providers.ollama.apiKey` değerini (veya ortam değişkeni destekli eşleşen sağlayıcı kimlik doğrulamasını) yeniden kullanır.
- Ana bilgisayar çözümleme sırası: `plugins.entries.ollama.config.webSearch.baseUrl` →
  `models.providers.ollama.baseUrl` (veya `baseURL`) → `http://127.0.0.1:11434`.
- Çözümlenen ana bilgisayar `https://ollama.com` ise OpenClaw, API anahtarını taşıyıcı kimlik doğrulaması olarak kullanarak doğrudan `https://ollama.com/api/web_search` çağrısı yapar.
- Aksi takdirde OpenClaw önce yerel proxy uç noktası `/api/experimental/web_search` değerini çağırır (bu uç nokta isteği imzalayıp Ollama Cloud'a iletir), ardından aynı ana bilgisayardaki `/api/web_search` değerine geri döner. Her ikisi de başarısız olursa ve `OLLAMA_API_KEY` ayarlanmışsa bu anahtarı yerel ana bilgisayara göndermeden `https://ollama.com/api/web_search` üzerinde bir kez daha dener.
- Ollama'ya erişilemiyorsa veya oturum açılmamışsa OpenClaw kurulum sırasında uyarır ancak sağlayıcının seçilmesini engellemez.

## İlgili

- [Web Aramasına genel bakış](/tr/tools/web) -- tüm sağlayıcılar ve otomatik algılama
- [Ollama](/tr/providers/ollama) -- Ollama model kurulumu ve bulut/yerel modlar
