---
read_when:
    - OpenClaw'ın yerel model sunucusunu yalnızca model veya gömme sağlayıcısı seçildiğinde başlatmasını istiyorsunuz
    - ds4, inferrs, vLLM, llama.cpp, MLX veya OpenAI ile uyumlu başka bir yerel sunucu çalıştırıyorsunuz
    - Yerel sağlayıcılar için soğuk başlatmayı, hazır olma durumunu ve boşta kapanmayı denetlemeniz gerekir
summary: OpenClaw model ve gömme isteklerinden önce yerel model sunucularını talep üzerine başlatın
title: Yerel model hizmetleri
x-i18n:
    generated_at: "2026-07-26T23:59:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a761113dd591fed0394379b2bad173165efc5e284565c652493e73d1e724529d
    source_path: gateway/local-model-services.md
    workflow: 16
---

`models.providers.<id>.localService`, gerektiğinde sağlayıcının sahip olduğu yerel bir model sunucusunu başlatır. Bir model veya gömme isteği bu sağlayıcıyı seçtiğinde OpenClaw sağlık uç noktasını yoklar, çalışmıyorsa işlemi başlatır, hazır olmasını bekler ve ardından isteği gönderir. Maliyetli yerel sunucuları gün boyu çalışır durumda tutmaktan kaçınmak için bunu kullanın.

## Nasıl çalışır?

1. Bir model veya gömme isteği, yapılandırılmış bir sağlayıcıya çözümlenir.
2. Bu sağlayıcıda `localService` varsa OpenClaw, `healthUrl` değerini yoklar.
3. Yoklama başarılı olursa OpenClaw, zaten çalışan sunucuyu kullanır.
4. Yoklama başarısız olursa OpenClaw, `command` işlemini `args` ile başlatır.
5. OpenClaw, `readyTimeoutMs` süresi dolana kadar sağlık uç noktasını düzenli olarak yoklar.
6. İstek, normal model veya gömme aktarımı üzerinden geçer.
7. OpenClaw işlemi başlattıysa ve `idleStopMs` ayarlanmışsa son devam eden istek bu süre boyunca boşta kaldıktan sonra işlemi durdurur.

OpenClaw bunun için launchd, systemd, Docker veya herhangi bir daemon yüklemez. Sunucu, ona ilk ihtiyaç duyan OpenClaw işleminin sıradan bir alt işlemidir.

Başlatma, yapılandırılmış her sağlayıcı ve komut/bağımsız değişken/ortam kümesi için seri hâle getirilir; böylece aynı hizmete yönelik eşzamanlı sohbet ve gömme istekleri yinelenen sunucular başlatmaz. Her istek, yanıt işleme tamamlanana kadar kendi kiralamasını tutar; dolayısıyla boşta kapatma, devam eden tüm model ve gömme isteklerini bekler. Yapılandırılmış sağlayıcı takma adları ayrı kalır: iki takma ad, aynı Ollama, LM Studio veya OpenAI uyumlu bağdaştırıcı kimliğinde birleştirilmeden farklı GPU ana makinelerine işaret edebilir.

Başka bir OpenClaw işleminin aynı `healthUrl` konumunda zaten sağlıklı bir sunucusu varsa bu işlem, sunucuyu sahiplenmeden yeniden kullanır (her işlem yalnızca bizzat başlattığı alt işlemi yönetir). Başlatma ve çıkış günlükleri; zamanlama ve çıkış ayrıntılarının yanı sıra sınırlı, hassas verileri çıkarılmış alt işlem çıktı kuyruklarını içerir; yapılandırılmış ortam değerleri hiçbir zaman yayımlanmaz.

## Yapılandırma biçimi

```json5
{
  models: {
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "local-model",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "/absolute/path/to/server",
          args: ["--host", "127.0.0.1", "--port", "8000"],
          cwd: "/absolute/path/to/working-dir",
          env: { LOCAL_MODEL_CACHE: "/absolute/path/to/cache" },
          healthUrl: "http://127.0.0.1:8000/v1/models",
          readyTimeoutMs: 180000,
          idleStopMs: 0,
        },
        models: [
          {
            id: "my-local-model",
            name: "My Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Yavaş soğuk başlatmaların ve uzun üretimlerin varsayılan model isteği zaman aşımına uğramaması için sağlayıcı girdisinde (`localService` üzerinde değil) `timeoutSeconds` değerini ayarlayın. Sunucunuz hazır olma durumunu temel URL'deki `/models` dışında bir konumda sunduğunda açık bir `healthUrl` değeri ayarlayın.

## Alanlar

| Alan             | Gerekli  | Açıklama                                                                                                                             |
| ---------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `command`        | evet     | Mutlak yürütülebilir dosya yolu. Kabuk PATH araması yapılmaz.                                                                         |
| `args`           | hayır    | İşlem bağımsız değişkenleri. Kabuk genişletmesi, kanallar, glob eşleştirmesi veya tırnak işleme yapılmaz.                              |
| `cwd`            | hayır    | İşlemin çalışma dizini.                                                                                                               |
| `env`            | hayır    | OpenClaw işlem ortamının üzerine birleştirilen ortam değişkenleri.                                                                    |
| `healthUrl`      | hayır    | Hazır olma URL'si. Varsayılan olarak sonuna `/models` eklenmiş `baseUrl` kullanılır (`http://127.0.0.1:8000/v1`, `http://127.0.0.1:8000/v1/models` olur). |
| `readyTimeoutMs` | hayır    | Başlatma hazır olma son süresi. Varsayılan: `120000`.                                                                       |
| `idleStopMs`     | hayır    | OpenClaw tarafından başlatılan bir işlemin boşta kapatma gecikmesi. `0` değeri veya belirtilmemesi, OpenClaw kapanana kadar işlemi çalışır durumda tutar. |

## Inferrs örneği

Inferrs, özel bir OpenAI uyumlu `/v1` arka ucudur; dolayısıyla aynı `localService` API'si bir `inferrs` sağlayıcı girdisiyle çalışır:

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/google/gemma-4-E2B-it" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "/opt/homebrew/bin/inferrs",
          args: [
            "serve",
            "google/gemma-4-E2B-it",
            "--host",
            "127.0.0.1",
            "--port",
            "8080",
            "--device",
            "metal",
          ],
          healthUrl: "http://127.0.0.1:8080/v1/models",
          readyTimeoutMs: 180000,
          idleStopMs: 0,
        },
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: { requiresStringContent: true },
          },
        ],
      },
    },
  },
}
```

`command` değerini, OpenClaw'u çalıştıran makinedeki `which inferrs` sonucuyla değiştirin. Eksiksiz Inferrs kurulumu: [Inferrs](/tr/providers/inferrs).

## ds4 örneği

```json5
{
  models: {
    providers: {
      ds4: {
        baseUrl: "http://127.0.0.1:18000/v1",
        apiKey: "ds4-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "<DS4_DIR>/ds4-server",
          args: [
            "--model",
            "<DS4_DIR>/ds4flash.gguf",
            "--host",
            "127.0.0.1",
            "--port",
            "18000",
            "--ctx",
            "32768",
            "--tokens",
            "128",
          ],
          cwd: "<DS4_DIR>",
          healthUrl: "http://127.0.0.1:18000/v1/models",
          readyTimeoutMs: 300000,
          idleStopMs: 0,
        },
        models: [],
      },
    },
  },
}
```

Eksiksiz kurulum, bağlam boyutlandırması ve doğrulama komutları: [ds4](/tr/providers/ds4).

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="Yerel modeller" href="/tr/gateway/local-models" icon="server">
    Yerel model kurulumu, sağlayıcı seçenekleri ve güvenlik rehberliği.
  </Card>
  <Card title="Inferrs" href="/tr/providers/inferrs" icon="cpu">
    OpenClaw'u Inferrs OpenAI uyumlu yerel sunucusu üzerinden çalıştırın.
  </Card>
</CardGroup>
