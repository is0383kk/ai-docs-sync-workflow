---
read_when:
    - memory-lancedb Plugin'ini yapılandırıyorsunuz
    - Otomatik hatırlama veya otomatik yakalama özellikli, LanceDB destekli uzun süreli bellek istiyorsunuz
    - Ollama gibi yerel OpenAI uyumlu gömmeleri kullanıyorsunuz
sidebarTitle: Memory LanceDB
summary: Yerel Ollama uyumlu gömmeler dahil olmak üzere resmî harici LanceDB bellek pluginini yapılandırın
title: Bellek LanceDB
x-i18n:
    generated_at: "2026-07-26T23:30:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bdb7208925ac6c76430ee36dfcd9733041530e0f2ee175950b3cdb8010d67b24
    source_path: plugins/memory-lancedb.md
    workflow: 16
---

`memory-lancedb`, uzun süreli belleği vektör aramasıyla LanceDB'de depolayan
resmî bir harici plugindir. Bir model sırasından önce ilgili anıları otomatik
olarak hatırlayabilir ve bir yanıttan sonra önemli olguları otomatik olarak yakalayabilir.

Yerel bir vektör veritabanı, OpenAI uyumlu bir gömme uç noktası veya varsayılan
yerleşik bellek arka ucunun dışında bir bellek deposu için kullanın.

## Kurulum

```bash
openclaw plugins install @openclaw/memory-lancedb
```

Plugin npm'de yayımlanır; OpenClaw çalışma zamanı görüntüsüne dahil değildir.
Yüklenmesi plugin girdisini yazar, etkinleştirir ve `plugins.slots.memory` değerini
`memory-lancedb` olarak değiştirir. Bellek yuvası şu anda başka bir plugine
aitse bu plugin bir uyarıyla devre dışı bırakılır.

<Note>
`memory-wiki` gibi eşlikçi pluginler `memory-lancedb` ile birlikte
çalışabilir, ancak etkin bellek yuvasına aynı anda yalnızca bir plugin sahip olur.
</Note>

<Note>
LanceDB'nin `memory_recall` özelliği, `memory.search.rememberAcrossConversations` tarafından kullanılan
korumalı özel döküm yetkilendirmesini almaz. [Gelişmiş Active Memory](/tr/concepts/active-memory#lancedb-memory)
aracılığıyla LanceDB'nin `autoRecall` özelliğini veya `memory_recall`
aracını kullanın. `openclaw doctor`, geçerli bellek sağlayıcısıyla Konuşmalar
arasında hatırlama kullanılamadığında bunu bildirir.
</Note>

## Hızlı başlangıç

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb",
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "openai",
            model: "text-embedding-3-small",
          },
          autoRecall: true,
          autoCapture: false,
        },
      },
    },
  },
}
```

Plugin yapılandırmasını değiştirdikten sonra Gateway'i yeniden başlatın ve
yüklendiğini doğrulayın:

```bash
openclaw gateway restart
openclaw plugins list
```

## Gömme yapılandırması

`embedding` zorunludur ve en az bir alan içermelidir. `provider`
varsayılan olarak `openai`, `model` ise varsayılan olarak
`text-embedding-3-small` değerini kullanır.

| Alan                   | Tür             | Notlar                                                                    |
| ---------------------- | --------------- | ------------------------------------------------------------------------- |
| `embedding.provider`   | dize            | Bağdaştırıcı kimliği; ör. `openai`, `github-copilot`, `ollama`. Varsayılan: `openai`. |
| `embedding.model`      | dize            | Varsayılan: `text-embedding-3-small`.                                     |
| `embedding.apiKey`     | dize            | İsteğe bağlıdır; `${ENV_VAR}` genişletmesini destekler.              |
| `embedding.baseUrl`    | dize            | İsteğe bağlıdır; `${ENV_VAR}` genişletmesini destekler.              |
| `embedding.dimensions` | tam sayı (>=1) | Yerleşik tabloda bulunmayan modeller için zorunludur (aşağıya bakın).      |

İki istek yolu vardır:

- **Sağlayıcı bağdaştırıcısı yolu** (varsayılan): `embedding.provider`
değerini ayarlayın ve `embedding.apiKey`/`embedding.baseUrl` değerlerini belirtmeyin.
Plugin; sağlayıcının yapılandırılmış kimlik doğrulama profilini, ortam değişkenini
veya `models.providers.<provider>.apiKey` değerini, `memory-core` tarafından kullanılan aynı
bellek gömme bağdaştırıcıları üzerinden çözümler. Bu yol `github-copilot`,
`ollama` ve gömme desteğine sahip diğer tüm paketlenmiş sağlayıcılar içindir.
- **Doğrudan OpenAI uyumlu istemci yolu**: `embedding.provider`
değerini ayarlamadan bırakın (veya `"openai"`) ve `embedding.apiKey` ile
`embedding.baseUrl` değerlerini ayarlayın. Paketlenmiş bir sağlayıcı bağdaştırıcısı
olmayan ham bir OpenAI uyumlu gömme uç noktası için bunu kullanın.

OpenAI Codex / ChatGPT OAuth, OpenAI Platform gömme kimlik bilgisi değildir.
OpenAI gömmeleri için bir OpenAI API anahtarı kimlik doğrulama profili,
`OPENAI_API_KEY` veya `models.providers.openai.apiKey` kullanın. Yalnızca OAuth kullanan
kullanıcılar `github-copilot` veya `ollama` gibi gömme özellikli
başka bir sağlayıcı seçmelidir.

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "github-copilot",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

Bazı OpenAI uyumlu gömme uç noktaları `encoding_format` parametresini reddeder;
diğerleri bunu yok sayar ve her zaman `number[]` döndürür. `memory-lancedb`,
isteklerde `encoding_format` değerini kullanmaz ve hem kayan noktalı sayı dizisi
hem de base64 ile kodlanmış float32 yanıtlarını kabul eder; dolayısıyla iki yanıt
biçimi de yapılandırma gerektirmeden çalışır.

### Boyutlar

OpenClaw yalnızca `text-embedding-3-small` (1536) ve `text-embedding-3-large` (3072) için
yerleşik boyuta sahiptir. LanceDB'nin vektör sütununu oluşturabilmesi için diğer
tüm modellerde açık bir `embedding.dimensions` değeri gerekir; örneğin 2048 boyutlu
ZhiPu `embedding-3`:

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            apiKey: "${ZHIPU_API_KEY}",
            baseUrl: "https://open.bigmodel.cn/api/paas/v4",
            model: "embedding-3",
            dimensions: 2048,
          },
        },
      },
    },
  },
}
```

## Ollama gömmeleri

Paketlenmiş Ollama sağlayıcı bağdaştırıcısı yolunu (`embedding.provider: "ollama"`) kullanın.
Bu yol, Ollama'nın yerel `/api/embed` uç noktasını çağırır ve
[Ollama](/tr/providers/ollama) sağlayıcısıyla aynı kimlik doğrulama/temel URL
kurallarını izler.

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb",
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "ollama",
            baseUrl: "http://127.0.0.1:11434",
            model: "mxbai-embed-large",
            dimensions: 1024,
          },
          recallMaxChars: 400,
          autoRecall: true,
          autoCapture: false,
        },
      },
    },
  },
}
```

`mxbai-embed-large` yerleşik boyut tablosunda bulunmadığından `dimensions`
zorunludur. Küçük yerel gömme modellerinde yerel sunucu bağlam uzunluğu hataları
döndürüyorsa `recallMaxChars` değerini düşürün.

## Hatırlama ve yakalama sınırları

| Ayar              | Varsayılan | Aralık                       | Uygulandığı alan                                             |
| ----------------- | ---------- | ---------------------------- | ------------------------------------------------------------ |
| `recallMaxChars`  | `1000`  | 100-10000                    | Hatırlama için gömme API'sine gönderilen metin.              |
| `captureMaxChars` | `500`   | 100-10000                    | Otomatik yakalama için uygun mesaj uzunluğu.                 |
| `customTriggers`  | `[]`    | 0-50 öğe, her biri <=100 karakter | Otomatik yakalamanın bir mesajı değerlendirmesini sağlayan değişmez ifadeler. |

`recallMaxChars`; `before_prompt_build` otomatik hatırlama sorgusunu,
`memory_recall` aracını, `memory_forget` sorgu yolunu ve
`openclaw ltm
search` değerini sınırlar. Otomatik hatırlama, sıradaki en son kullanıcı
mesajını gömer ve yalnızca kullanıcı mesajı bulunmadığında tam isteme geri döner;
böylece kanal meta verileri ve büyük istem blokları gömme isteğinin dışında tutulur.

`captureMaxChars`, sıranın `agent_end` olayındaki bir kullanıcı mesajının
otomatik yakalama için değerlendirilebilecek kadar kısa olup olmadığını belirler;
hatırlama sorgularını etkilemez.

`customTriggers`, regex kullanmadan değişmez otomatik yakalama ifadeleri ekler.
Yerleşik tetikleyiciler yaygın İngilizce, Çekçe, Çince, Japonca ve Korece bellek
ifadelerini (`remember`, `prefer`, `记住`,
`覚えて`, `기억해` ve benzerleri) kapsar.

Otomatik yakalama ayrıca zarf/taşıma meta verisine, istem enjeksiyonu yüklerine
veya önceden eklenmiş `<relevant-memories>` bağlamına benzeyen metinleri reddeder
ve her aracı sırası için en fazla 3 bellek yakalar.

Her bellek tek bir aracıya aittir. Hatırlama, yinelenen öğe algılama, yakalama,
listeleme, ham sorgular ve silme işlemlerinin tümü, satırları döndürmeden veya
değiştirmeden önce bu sahipliği zorunlu kılar. `agents.entries.*` girdisinde
`memory.search.enabled: false` bulunan veya devre dışı bırakılmış üst düzey aramayı devralan
bir aracı da `memory_recall`, `memory_store` ya da `memory_forget`
araçlarının hiçbirini almaz ve plugin düzeyindeki `autoRecall`/
`autoCapture` bayrakları açık olsa bile otomatik hatırlama veya yakalamaya
katılmaz.

## Komutlar

`memory-lancedb`, yüklendiği her durumda (yalnızca etkin bellek yuvasına sahip
olduğunda değil) `ltm` CLI ad alanını kaydeder:

```bash
openclaw ltm list [--agent <id>] [--limit <n>] [--order-by-created-at]
openclaw ltm search <query> [--agent <id>] [--limit <n>]
openclaw ltm stats [--agent <id>]
```

`ltm query`, doğrudan LanceDB tablosunda vektörsüz bir sorgu çalıştırır:

```bash
openclaw ltm query --agent research --cols id,text,createdAt --limit 20
openclaw ltm query --filter "category = 'preference'" --order-by createdAt:desc
```

| Bayrak                            | Varsayılan                              | Notlar                                                                                                                                    |
| --------------------------------- | --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `--agent <id>`                    | yapılandırılmış varsayılan aracı         | Özel aracı ad alanını seçer. `list`, `search`, `query` ve `stats` üzerinde kullanılabilir.                                         |
| `--cols <columns>`                | `id,text,importance,category,createdAt` | Virgülle ayrılmış sütun izin listesi.                                                                                                     |
| `--filter <condition>`            | yok                                     | Bir çıktı sütunu üzerinde `category = 'preference'` veya `importance >= 0.8` gibi tek bir karşılaştırma. Dize değerleri tırnak içine alınmalıdır. |
| `--limit <n>`                     | `10`                                    | Pozitif tam sayı.                                                                                                                         |
| `--order-by <column>:<asc\|desc>` | yok                                     | Filtre çalıştıktan sonra bellekte sıralanır; sıralama sütunu projeksiyona otomatik eklenir ve istenmemişse çıktıdan çıkarılır.             |

Aracılar etkin bellek plugininden üç araç alır:

- `memory_recall`: depolanan belleklerde vektör araması.
- `memory_store`: bir olguyu, tercihi, kararı veya varlığı
kaydetme (istem enjeksiyonu yüküne benzeyen metni reddeder; neredeyse yinelenen
kayıtları atlar).
- `memory_forget`: `memoryId` ile veya
`query` ile silme (%90 puanın üzerindeki tek bir eşleşmeyi otomatik
olarak siler; aksi hâlde belirsizliği gidermek için aday kimlikleri listeler).

## Depolama

LanceDB verileri varsayılan olarak `~/.openclaw/memory/lancedb` konumunda saklanır.
`dbPath` ile geçersiz kılın:

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          dbPath: "~/.openclaw/memory/lancedb",
          embedding: {
            apiKey: "${OPENAI_API_KEY}",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

Plugin tek bir LanceDB tablosu tutar ve her satırda normalleştirilmiş bir aracı
sahibi depolar. Bu, arama sonrası uygulanan bir filtre değil, bir depolama
sınırıdır: aracı sahipliği vektör sıralamasından önce uygulanır ve listeleme,
sorgulama, sayma ve silme koşullarına dahil edilir. `ltm query --filter`, genel
çıktı sütunları üzerinde doğrulanmış tek bir karşılaştırmayı kabul eder. Depo,
bu karşılaştırmayı zorunlu sahip koşulundan ayrı oluşturur; dolayısıyla bir
filtre sorguyu başka bir aracıya genişletemez.

Aracı başına sahiplikten önce oluşturulan veritabanlarında güvenilir satır
kökeni bulunmaz. Yükseltme sırasında `openclaw doctor --fix`, bu eski satırları
bir kez yapılandırılmış varsayılan aracıya atar. Çalışma zamanı erişimi bu geçiş
tamamlanana kadar kapalı kalır; diğer aracılar eski paylaşılan satırları hiçbir
zaman devralmaz.

`storageOptions`, LanceDB depolama arka uçları (ör. S3 uyumlu nesne depolama) için dize anahtar/değer çiftlerini kabul eder ve `${ENV_VAR}` genişletmesini destekler:

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          dbPath: "s3://memory-bucket/openclaw",
          storageOptions: {
            access_key: "${AWS_ACCESS_KEY_ID}",
            secret_key: "${AWS_SECRET_ACCESS_KEY}",
            endpoint: "${AWS_ENDPOINT_URL}",
          },
          embedding: {
            apiKey: "${OPENAI_API_KEY}",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

## Çalışma zamanı bağımlılıkları ve platform desteği

`memory-lancedb`, plugin paketinin sahip olduğu yerel `@lancedb/lancedb` paketine bağlıdır (OpenClaw çekirdek dağıtımına değil). Gateway başlatılırken plugin bağımlılıkları onarılmaz; yerel bağımlılık eksikse veya yüklenemezse plugin paketini yeniden kurun ya da güncelleyin ve Gateway'i yeniden başlatın.

`@lancedb/lancedb`, `darwin-x64` (Intel Mac) için yerel bir derleme yayımlamaz. Bu platformda plugin, yükleme sırasında LanceDB'nin kullanılamadığını günlüğe kaydeder; varsayılan bellek arka ucunu kullanın, Gateway'i desteklenen bir platformda/mimaride çalıştırın veya `memory-lancedb` seçeneğini devre dışı bırakın.

## Sorun giderme

### Girdi uzunluğu bağlam uzunluğunu aşıyor

Gömme modeli geri çağırma sorgusunu reddetti:

```text
memory-lancedb: geri çağırma başarısız oldu: Hata: 400 girdi uzunluğu bağlam uzunluğunu aşıyor
```

`recallMaxChars` değerini düşürün, ardından Gateway'i yeniden başlatın:

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        config: {
          recallMaxChars: 400,
        },
      },
    },
  },
}
```

Ollama için ayrıca yerel gömme uç noktasını kullanarak gömme sunucusuna Gateway ana makinesinden erişilebildiğini doğrulayın:

```bash
curl http://127.0.0.1:11434/api/embed \
  -H "Content-Type: application/json" \
  -d '{"model":"mxbai-embed-large","input":"hello"}'
```

### Desteklenmeyen gömme modeli

`embedding.dimensions` olmadan yalnızca yerleşik OpenAI gömme boyutları bilinir (`text-embedding-3-small`, `text-embedding-3-large`). Diğer tüm modeller için `embedding.dimensions` değerini modelin bildirdiği vektör boyutuna ayarlayın.

### Plugin yükleniyor ancak hiçbir anı görünmüyor

`plugins.slots.memory` değerinin `memory-lancedb` hedefine işaret ettiğini doğrulayın, ardından şunları çalıştırın:

```bash
openclaw ltm stats
openclaw ltm search "recent preference"
```

`autoCapture` devre dışıysa plugin mevcut anıları geri çağırmaya devam eder ancak yenilerini otomatik olarak depolamaz. `memory_store` aracını kullanın veya `autoCapture` seçeneğini etkinleştirin.

## İlgili

- [Belleğe genel bakış](/tr/concepts/memory)
- [Active Memory](/tr/concepts/active-memory)
- [Bellek araması](/tr/concepts/memory-search)
- [Bellek Wiki'si](/tr/plugins/memory-wiki)
- [Ollama](/tr/providers/ollama)
