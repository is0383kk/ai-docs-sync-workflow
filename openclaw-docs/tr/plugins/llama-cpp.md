---
read_when:
    - API anahtarı veya model sunucusu olmadan yerel metin çıkarımı istiyorsunuz
    - Yerel bir GGUF modelinden bellek arama gömmeleri istiyorsunuz
    - memory.search.provider = "local" yapılandırmasını yapıyorsunuz
    - node-llama-cpp çalışma zamanının sahibi olan OpenClaw pluginine ihtiyacınız var
sidebarTitle: llama.cpp Provider
summary: llama.cpp ile OpenClaw'da yerel GGUF metin çıkarımı ve bellek gömmeleri çalıştırın
title: llama.cpp Sağlayıcısı
x-i18n:
    generated_at: "2026-07-26T23:50:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 88e6d66943adcbc602421b8cc00359b3ed87357194c3ffaa845c1db7fbcd9c38
    source_path: plugins/llama-cpp.md
    workflow: 16
---

`llama-cpp`, işlem içi yerel GGUF metin çıkarımı ve gömmeleri için resmi harici sağlayıcı Plugin'idir. `llama-cpp` metin sağlayıcısını ve `local` gömme sağlayıcısını kaydeder; ayrıca `node-llama-cpp` yerel çalışma zamanının sahibidir.

Yerel çıkarımı veya yerel bellek gömmelerini kullanmadan önce bunu yükleyin:

```bash
openclaw plugins install @openclaw/llama-cpp-provider
```

Ana `openclaw` npm paketi `node-llama-cpp` içermez. Yerel bağımlılığın bu Plugin'de tutulması, normal OpenClaw npm güncellemelerinin OpenClaw paket dizinine elle yüklenmiş bir çalışma zamanını silmesini önler.

## Yerel metin çıkarımı

Etkileşimli ilk katılım sırasında **Yerel model (llama.cpp)** seçeneğini belirleyin. OpenClaw, varsayılan modeli indirmeden önce sorar:

`hf:bartowski/Qwen_Qwen3-4B-Instruct-2507-GGUF/Qwen_Qwen3-4B-Instruct-2507-Q4_K_M.gguf`

Qwen3 4B Instruct 2507 Q4_K_M dosyası yaklaşık 2,5 GB'tır. Model ağırlıkları için yaklaşık 3 GB RAM'in yanı sıra bağlam ve OpenClaw çalışma zamanı ek yükünü hesaba katın. Varsayılan bağlam, 8 GB belleğe sahip makinelerde kullanılabilir kalması için 8.192 token sınırıyla otomatik olarak boyutlandırılır. Daha büyük bir bağlamı yalnızca makinede yeterli bellek olduğunda yapılandırın.

İlk katılım keşif denetimi salt okunurdur. llama.cpp seçeneğini yalnızca varsayılan veya yapılandırılmış GGUF dosyası model önbelleğinde zaten bulunduğunda otomatik olarak sunar; keşif sırasında hiçbir zaman indirme yapmaz. Ollama ve LM Studio ayrı yerel hizmet seçenekleri olarak kalır ve kendi keşif akışlarını korur. Varsayılan modelin indirilmesi için istem gösteren yol, llama.cpp seçeneğini elle belirlemektir.

Sağlayıcı, GGUF modelinin gömülü sohbet şablonunu ve yerel node-llama-cpp işlev çağrısını kullanır. Metin, token token akışla iletilir. Araç çağrıları node-llama-cpp içinde çalıştırılmak yerine yürütülmek üzere OpenClaw'a döner.

### Başka bir GGUF modeli kullanma

`models.providers.llama-cpp` öğesine bir model ekleyin. `params.modelPath` içine yerel bir yol veya tam `hf:` dosya URI'si koyun:

```json5
{
  models: {
    mode: "merge",
    providers: {
      "llama-cpp": {
        baseUrl: "local://llama-cpp",
        api: "openai-completions",
        params: {
          modelCacheDir: "~/.node-llama-cpp/models",
        },
        models: [
          {
            id: "my-local-model",
            name: "My local GGUF",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 2048,
            params: {
              modelPath: "~/Models/my-model.Q4_K_M.gguf",
              contextSize: 8192,
            },
            compat: { supportsTools: true },
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "llama-cpp/my-local-model" },
    },
  },
}
```

Çıkarım, eksik bir modeli hiçbir zaman örtük olarak indirmez. Özel bir `hf:` URI'si için önce GGUF dosyasını `modelCacheDir` içine indirin. Keşif; depo, dal ve bölünmüş dosya adlandırması dahil olmak üzere node-llama-cpp'nin kendi salt okunur önbellek çözümleyicisini kullanır.

## Bellek gömme yapılandırması

`memory.search.provider` değerini `local` olarak ayarlayın:

```json5
{
  memory: {
    search: {
      provider: "local",
      local: {
        modelPath: "hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf",
      },
    },
  },
}
```

`local.modelPath` varsayılan olarak yukarıda gösterilen `hf:` URI'sini (`embeddinggemma-300m-qat-Q8_0.gguf`) kullanır. Başka bir model kullanmak için bunu farklı bir `hf:` URI'sine veya yerel bir `.gguf` dosyasına yönlendirin. `local.modelCacheDir`, indirilen modellerin önbelleğe alınacağı konumu geçersiz kılar (varsayılan: `~/.node-llama-cpp/models`); `local.contextSize` ise bir tam sayı veya `"auto"` kabul eder.

`local.contextSize` sayısal olduğunda sağlayıcı, bu gereksinimi node-llama-cpp'nin otomatik GPU katmanı yerleşimine de iletir. Bu, node-llama-cpp'nin bellek güvenliği denetimlerini korurken modeli ve gömme bağlamını birlikte sığdırmasına olanak tanır. `"auto"` kullanıldığında node-llama-cpp normal otomatik yerleşimini korur.

## Yerel çalışma zamanı

En sorunsuz yerel yükleme yolu için Node 24 kullanın. pnpm kullanan kaynak kullanıma almalarında yerel bağımlılığın onaylanması ve yeniden derlenmesi gerekebilir:

```bash
pnpm approve-builds
pnpm rebuild node-llama-cpp
```

## Bellek çalışma zamanı tanılamaları

Seçili arka ucu ve derlemeyi, cihaz adlarını, GPU'ya aktarılan katmanları, istenen bağlam boyutunu ve son gözlemlenen VRAM veya birleşik bellek anlık görüntüsünü incelemek için sağlayıcı yüklendikten sonra `openclaw memory status --deep` komutunu çalıştırın. Pasif durum okumaları modeli yeniden yüklemediği veya cihazı yoklamadığı için VRAM değerleri bir gözlem zaman damgası içerir.

Çalışan Gateway yerel sağlayıcıyı zaten kullandıysa aynı son bilinen bilgiler `openclaw doctor` içinde görünebilir. Normal bir durum veya doctor komutu yalnızca tanılama verilerini toplamak için model yüklemez.

## Sorun giderme

`node-llama-cpp` eksikse veya yüklenemezse OpenClaw, hatayı şu bilgilerle bildirir:

1. Plugin'i yükleyin: `openclaw plugins install @openclaw/llama-cpp-provider`.
2. Yerel yüklemeler/güncellemeler için Node 24 kullanın.
3. Bir pnpm kaynak kullanıma almasından: `pnpm approve-builds`, ardından `pnpm rebuild node-llama-cpp`.

İşlem içi yerel bağımlılık olmadan yerel çıkarım için bunun yerine Ollama veya LM Studio sağlayıcısını kullanın. Daha az zahmetli yerel gömmeler için bunun yerine `memory.search.provider` değerini `lmstudio`, `ollama`, `openai` veya `voyage` gibi bir uzak gömme sağlayıcısına ayarlayın.
