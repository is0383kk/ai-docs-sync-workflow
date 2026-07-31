---
read_when:
    - Varsayılan bellek arka ucunu anlamak istiyorsunuz
    - Gömme sağlayıcılarını veya hibrit aramayı yapılandırmak istiyorsunuz
summary: Anahtar kelime, vektör ve hibrit arama özelliklerine sahip varsayılan SQLite tabanlı bellek arka ucu
title: Yerleşik bellek motoru
x-i18n:
    generated_at: "2026-07-26T22:44:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3efb6f1449d9b55717b3c117444ba7d4519d0111b842b48790ad85551511433
    source_path: concepts/memory-builtin.md
    workflow: 16
---

Yerleşik motor, varsayılan bellek arka ucudur. Bellek dizininizi
ajan başına bir SQLite veritabanında depolar ve başlamak için
ek bağımlılık gerektirmez.

## Sağladıkları

- **Anahtar kelime araması**, FTS5 tam metin dizinleme (BM25 puanlaması) aracılığıyla.
- **Vektör araması**, desteklenen herhangi bir sağlayıcının gömmeleri aracılığıyla.
- **Hibrit arama**, en iyi sonuçlar için ikisini birleştirir.
- **CJK desteği**, Çince, Japonca ve Korece için trigram tokenleştirme aracılığıyla.
- **sqlite-vec hızlandırması**, veritabanı içi vektör sorguları için (isteğe bağlı).

## Başlarken

Yerleşik motor varsayılan olarak OpenAI gömmelerini kullanır. `OPENAI_API_KEY` veya
`models.providers.openai.apiKey` zaten yapılandırılmışsa vektör araması,
ek bellek yapılandırması olmadan çalışır.

Bir sağlayıcıyı açıkça ayarlamak için:

```json5
{
  memory: {
    search: {
      provider: "openai",
    },
  },
}
```

Bir gömme sağlayıcısı olmadan yalnızca anahtar kelime araması kullanılabilir.

Yerel GGUF gömmelerini zorunlu kılmak için resmi llama.cpp sağlayıcı
pluginini yükleyin, ardından `local.modelPath` öğesini bir GGUF dosyasına yönlendirin:

```bash
openclaw plugins install @openclaw/llama-cpp-provider
```

```json5
{
  memory: {
    search: {
      provider: "local",
      fallback: "none",
      local: {
        modelPath: "~/.node-llama-cpp/models/embeddinggemma-300m-qat-Q8_0.gguf",
      },
    },
  },
}
```

## Desteklenen gömme sağlayıcıları

| Sağlayıcı         | Kimlik              | Notlar                                      |
| ----------------- | ------------------- | ------------------------------------------- |
| Bedrock           | `bedrock`           | AWS kimlik bilgisi zincirini kullanır       |
| DeepInfra         | `deepinfra`         | Varsayılan: `BAAI/bge-m3`                   |
| Gemini            | `gemini`            | Çok modluyu destekler (görüntü + ses)       |
| GitHub Copilot    | `github-copilot`    | Copilot aboneliğinizi kullanır              |
| LM Studio         | `lmstudio`          | Yerel/kendi sunucunuzda                     |
| Yerel             | `local`             | `@openclaw/llama-cpp-provider`      |
| Mistral           | `mistral`           |                                             |
| Ollama            | `ollama`            | Yerel/kendi sunucunuzda                     |
| OpenAI            | `openai`            | Varsayılan: `text-embedding-3-small`   |
| OpenAI uyumlu     | `openai-compatible` | Genel `/v1/embeddings` uç noktası            |
| Voyage            | `voyage`            |                                             |

OpenAI'dan geçiş yapmak için `memory.search.provider` öğesini ayarlayın.

## Dizinleme nasıl çalışır?

OpenClaw, `MEMORY.md` ve `memory/*.md` öğelerini parçalara (varsayılan olarak
80 token çakışmalı 400 token) ayırarak dizinler ve bunları ajan başına bir SQLite veritabanında depolar.

- **Dizin konumu:** sahibi olan ajan veritabanında:
  `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- **Depolama bakımı:** SQLite WAL yan dosyaları, periyodik ve
  kapatma sırasındaki kontrol noktalarıyla sınırlandırılır.
- **Dosya izleme:** bellek dosyalarındaki değişiklikler, gecikmeli bir yeniden dizinlemeyi
  tetikler (varsayılan 1.5s).
- **Otomatik yeniden dizinleme:** gömme sağlayıcısı, model, parçalama yapılandırması,
  yapılandırılmış kaynaklar veya kapsam değiştiğinde dizin otomatik olarak yeniden oluşturulur.
- **İstek üzerine yeniden dizinleme:** `openclaw memory index --force`

<Info>
Çalışma alanı dışındaki Markdown dosyalarını da
`memory.search.extraPaths` ile dizinleyebilirsiniz. Bkz.
[yapılandırma referansı](/tr/reference/memory-config#additional-memory-paths).
</Info>

## Ne zaman kullanılmalı?

Yerleşik motor çoğu kullanıcı için doğru seçimdir:

- Ek bağımlılık gerektirmeden kullanıma hazır çalışır.
- Anahtar kelime ve vektör aramasını iyi yönetir.
- Tüm gömme sağlayıcılarını destekler.
- Hibrit arama, her iki getirme yaklaşımının en iyi yönlerini birleştirir.

Yeniden sıralamaya, sorgu genişletmeye ihtiyacınız varsa veya çalışma alanı dışındaki
dizinleri dizinlemek istiyorsanız [QMD](/tr/concepts/memory-qmd) kullanmayı düşünün.

Otomatik kullanıcı modellemeyle oturumlar arası bellek istiyorsanız
[Honcho](/tr/concepts/memory-honcho) kullanmayı düşünün.

## Sorun giderme

**Bellek araması devre dışı mı?** `openclaw memory status` öğesini kontrol edin. Sağlayıcı
algılanmazsa açıkça bir tane ayarlayın veya bir API anahtarı ekleyin.

**Yerel sağlayıcı algılanmıyor mu?** Yerel yolun mevcut olduğunu doğrulayın ve şunları çalıştırın:

```bash
openclaw memory status --deep --agent main
openclaw memory index --force --agent main
```

Hem bağımsız CLI komutları hem de Gateway aynı `local` sağlayıcı kimliğini kullanır.
Yerel gömmeler istediğinizde `memory.search.provider: "local"` öğesini ayarlayın.

**Sonuçlar güncel değil mi?** Yeniden oluşturmak için `openclaw memory index --force` komutunu çalıştırın. İzleyici,
nadir uç durumlarda değişiklikleri kaçırabilir.

**sqlite-vec yüklenmiyor mu?** OpenClaw otomatik olarak işlem içi kosinüs
benzerliğine geri döner. `openclaw memory status --deep`, yerel
vektör deposunu gömme sağlayıcısından ayrı olarak bildirir; bu nedenle `Vector store:
unavailable`, sqlite-vec yüklemesini, `Embeddings: unavailable` ise
sağlayıcı/kimlik doğrulama veya model hazırlığını gösterir. Belirli yükleme
hatası için günlükleri kontrol edin.

## Yapılandırma

Gömme sağlayıcısı kurulumu, hibrit arama ayarlaması (ağırlıklar, MMR, zamansal
azalma), toplu dizinleme, çok modlu bellek, sqlite-vec, ek yollar ve diğer tüm
yapılandırma seçenekleri için
[Bellek yapılandırma referansına](/tr/reference/memory-config) bakın.

## İlgili

- [Belleğe genel bakış](/tr/concepts/memory)
- [Bellek araması](/tr/concepts/memory-search)
- [Active Memory](/tr/concepts/active-memory)
