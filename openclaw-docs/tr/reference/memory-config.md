---
read_when:
    - Bellek arama sağlayıcılarını veya gömme modellerini yapılandırmak istiyorsunuz
    - QMD arka ucunu kurmak istiyorsunuz
    - Hibrit aramayı, MMR'yi veya zamansal azalmayı etkinleştirmek istiyorsunuz
    - Çok modlu bellek indekslemeyi etkinleştirmek istiyorsunuz
sidebarTitle: Memory config
summary: Bellek arama sağlayıcıları, getirme modları, QMD ve çok modlu indeksleme
title: Bellek yapılandırma başvurusu
x-i18n:
    generated_at: "2026-07-26T23:35:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91f843b1516093c49e18b3d659ab24ea9cb7be32aaaac722205eca8bc3f2ca5b
    source_path: reference/memory-config.md
    workflow: 16
---

Bu sayfa, OpenClaw bellek araması için tüm yapılandırma seçeneklerini listeler. Kavramsal genel bakışlar için bkz.:

<CardGroup cols={2}>
  <Card title="Belleğe genel bakış" href="/tr/concepts/memory">
    Belleğin çalışma şekli.
  </Card>
  <Card title="Yerleşik motor" href="/tr/concepts/memory-builtin">
    Varsayılan SQLite arka ucu.
  </Card>
  <Card title="QMD motoru" href="/tr/concepts/memory-qmd">
    Önce yerel yaklaşımını kullanan yardımcı süreç.
  </Card>
  <Card title="Bellek araması" href="/tr/concepts/memory-search">
    Arama işlem hattı ve ayarlama.
  </Card>
  <Card title="Active Memory" href="/tr/concepts/active-memory">
    Etkileşimli oturumlar için bellek alt aracısı.
  </Card>
</CardGroup>

Tüm paylaşılan bellek ayarları, `openclaw.json` içindeki üst düzey `memory` altında bulunur. Arama varsayılanları `memory.search`; aracı başına arama geçersiz kılmaları ise `agents.entries.*.memory.search` kullanır.

<Note>
Önerilen kişisel aracı iş akışı için
`memory.search.rememberAcrossConversations` kullanın. Gelişmiş Active Memory hedefleme,
model, istem ve gecikme denetimleri `plugins.entries.active-memory` altında bulunur.

Her iki etkinleştirme yolu, transkript kalıcılığı ve güvenli kullanıma alma
rehberi için [Active Memory](/tr/concepts/active-memory) bölümüne bakın.
</Note>

---

## Konuşmalar arasında hatırlama

| Anahtar                       | Tür       | Varsayılan                                                 | Açıklama                                                                       |
| ----------------------------- | --------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `rememberAcrossConversations` | `boolean` | Kişisel kurulumlarda açık; yapılandırılmış DM yalıtımında kapalı | Bu aracının tanınan diğer özel konuşmalarındaki ilgili bağlamı kullanın. |

Yalnızca güvenilir bir kişisel aracının konuşmalar arası transkript
hatırlamasını kullanması gerektiğinde bunu aracı başına yapılandırın:

```json5
{
  agents: {
    entries: {
      personal: {
        memory: {
          search: {
            rememberAcrossConversations: true,
          },
        },
      },
    },
  },
}
```

Değer, aracı başına geçersiz kılmayla normal `memory.search`
devralmasını izler. Ayarlanmadığında, yalnızca genel
`session.dmScope` ayarlanmamışsa veya `"main"` ise ve hiçbir bağlamada `session.dmScope`
geçersiz kılması yoksa varsayılan olarak açılır. Yapılandırılmış herhangi bir DM yalıtımı, varsayılan olarak bunu kapatır. Açıkça belirtilen `true` veya
`false` her zaman önceliklidir. Etkinleştirilmesi, oturum transkripti dizinlemeyi gerektirir ve
aracının çözümlenmiş bellek kaynaklarına `sessions` ekler. QMD ile ayrıca
o aracının oturum dışa aktarımını etkinleştirir; bu mod için ayrı bir
`memory.qmd.sessions.enabled` ayarı gerekmez.

OpenClaw'ın yerleşik bellek sağlayıcısı, bu korumalı yolu hem yerleşik
hem de QMD arka uçlarıyla destekler. Alternatif bellek sağlayıcıları kendi
hatırlama kancalarını ve gelişmiş Active Memory araçlarını kullanmaya devam edebilir, ancak geçerli sağlayıcı
korumalı özel transkript hatırlamayı desteklemediği sürece bu ayar atlanır.
`openclaw doctor`, desteklenmeyen bir sağlayıcıyı veya `memory_search` öğesini içermeyen açık bir Active Memory
`toolsAllow` listesini bildirir.

Getirme sınırı, genel oturum aramasından daha dardır:

- yalnızca aynı aracının tanınan özel konuşmaları uygundur
- yanıtlanan konuşma hariç tutulur
- gruplar ve kanallar kaynak ve hedef olarak hariç tutulur
- bilinmeyen konuşma türleri güvenli biçimde reddedilir
- korumalı alandaki hatırlama, özel konuşmalar arası yetkilendirmeyi kullanamaz

Bu ayar; `tools.sessions.visibility`, oturum anahtarları,
transkript depolama, teslim yönlendirme veya `sessions_list`,
`sessions_history` ve `sessions_send` izinlerini değiştirmez. Active Memory, sınırlı ve
salt okunur bir getirme geçişi gerçekleştirir; kullanılamayan veya zaman aşımına uğrayan getirme,
yanıtı engellemez.

---

## Sağlayıcı seçimi

| Anahtar    | Tür       | Varsayılan       | Açıklama                                                                                                                                                                                                                                                                                    |
| ---------- | --------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`  | `boolean` | `true`           | Bellek aramasını etkinleştirin veya devre dışı bırakın                                                                                                                                                                                                                                      |
| `provider` | `string`  | `"openai"`       | `bedrock`, `deepinfra`, `gemini`, `github-copilot`, `local`, `mistral`, `ollama`, `openai`, `openai-compatible` veya `voyage` gibi gömme bağdaştırıcısı kimliği; ayrıca `api` değeri bir bellek gömme bağdaştırıcısını veya OpenAI uyumlu model API'sini gösteren yapılandırılmış bir `models.providers.<id>` olabilir |
| `model`    | `string`  | sağlayıcı varsayılanı | Gömme modeli adı                                                                                                                                                                                                                                                                            |
| `fallback` | `string`  | `"none"`         | Birincil bağdaştırıcı başarısız olduğunda kullanılacak yedek bağdaştırıcı kimliği                                                                                                                                                                                                            |

`provider` ayarlanmadığında OpenClaw, OpenAI gömmelerini kullanır. Bedrock, DeepInfra, Gemini, GitHub Copilot, Mistral, Ollama,
Voyage, yerel bir GGUF modeli veya OpenAI uyumlu bir `/v1/embeddings` uç noktası kullanmak için `provider`
değerini açıkça ayarlayın.
Hâlâ `provider: "auto"` belirten eski yapılandırmalar `openai` olarak çözümlenir.

<Warning>
Gömme sağlayıcısının, modelin, sağlayıcı ayarlarının, kaynakların, kapsamın,
parçalamanın veya belirteç oluşturucunun değiştirilmesi mevcut SQLite vektör dizinini uyumsuz hâle getirebilir.
OpenClaw, her şeyi otomatik olarak yeniden gömmek yerine vektör aramasını duraklatır
ve bir dizin kimliği uyarısı bildirir. Hazır olduğunuzda
`openclaw memory status --index --agent <id>` veya
`openclaw memory index --force --agent <id>` ile yeniden oluşturun.
</Warning>

`provider` ayarlanmamışsa, eski `provider: "auto"` mevcutsa veya
`provider: "none"` bilinçli olarak yalnızca FTS modunu seçiyorsa, gömmeler kullanılamadığında
bellekten hatırlama yine de sözcüksel FTS sıralamasını kullanabilir.

Açıkça belirtilen yerel olmayan sağlayıcılar güvenli biçimde reddedilir. `memory.search.provider` değerini
Bedrock, DeepInfra, Gemini, GitHub
Copilot, LM Studio, Mistral, Ollama, OpenAI, Voyage veya OpenAI uyumlu
özel sağlayıcı gibi uzak destekli somut bir sağlayıcıya ayarlarsanız ve bu sağlayıcı çalışma zamanında kullanılamazsa, `memory_search`
sessizce yalnızca FTS hatırlamayı kullanmak yerine kullanılamıyor sonucu
döndürür. Sağlayıcı/kimlik doğrulama yapılandırmasını düzeltin, erişilebilir bir sağlayıcıya geçin veya
bilinçli olarak yalnızca FTS hatırlama istiyorsanız `provider: "none"` değerini ayarlayın.

### Özel sağlayıcı kimlikleri

`memory.search.provider`; `ollama` gibi belleğe özgü sağlayıcı bağdaştırıcıları veya `openai-responses` / `openai-completions` gibi OpenAI uyumlu model API'leri için özel bir `models.providers.<id>` girdisini gösterebilir. OpenClaw, uç nokta, kimlik doğrulama ve model öneki işlemleri için özel sağlayıcı kimliğini korurken gömme bağdaştırıcısına ait sağlayıcının `api` sahibini çözümler. Bu, çoklu GPU veya çoklu ana makine kurulumlarının bellek gömmelerini belirli bir yerel uç noktaya ayırmasına olanak tanır:

```json5
{
  models: {
    providers: {
      "ollama-5080": {
        api: "ollama",
        baseUrl: "http://gpu-box.local:11435",
        apiKey: "ollama-local",
        models: [{ id: "qwen3-embedding:0.6b", name: "Qwen3 Embedding 0.6B" }],
      },
    },
  },
  memory: {
    search: {
      provider: "ollama-5080",
      model: "qwen3-embedding:0.6b",
    },
  },
}
```

### API anahtarı çözümleme

Uzak gömmeler bir API anahtarı gerektirir. Bedrock bunun yerine AWS SDK varsayılan kimlik bilgisi zincirini kullanır (örnek rolleri, SSO, erişim anahtarları veya Bedrock API anahtarı).

| Sağlayıcı      | Ortam değişkeni                                     | Yapılandırma anahtarı               |
| -------------- | --------------------------------------------------- | ----------------------------------- |
| Bedrock        | AWS kimlik bilgisi zinciri veya `AWS_BEARER_TOKEN_BEDROCK` | API anahtarı gerekmez               |
| DeepInfra      | `DEEPINFRA_API_KEY`                                 | `models.providers.deepinfra.apiKey` |
| Gemini         | `GEMINI_API_KEY`                                    | `models.providers.google.apiKey`    |
| GitHub Copilot | `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, `GITHUB_TOKEN`  | Cihaz oturum açma yoluyla kimlik doğrulama profili |
| Mistral        | `MISTRAL_API_KEY`                                   | `models.providers.mistral.apiKey`   |
| Ollama         | `OLLAMA_API_KEY` (yer tutucu)                      | --                                  |
| OpenAI         | `OPENAI_API_KEY`                                    | `models.providers.openai.apiKey`    |
| Voyage         | `VOYAGE_API_KEY`                                    | `models.providers.voyage.apiKey`    |

<Note>
Codex OAuth yalnızca sohbet/tamamlama işlemlerini kapsar ve gömme isteklerini karşılamaz.
</Note>

---

## Uzak uç nokta yapılandırması

Genel OpenAI sohbet kimlik bilgilerini devralmaması gereken genel, OpenAI uyumlu bir
`/v1/embeddings` sunucusu için `provider: "openai-compatible"` kullanın.

<ParamField path="remote.baseUrl" type="string">
  Özel API temel URL'si.
</ParamField>
<ParamField path="remote.apiKey" type="string">
  API anahtarını geçersiz kılar.
</ParamField>
<ParamField path="remote.headers" type="object">
  Ek HTTP üstbilgileri (sağlayıcı varsayılanlarıyla birleştirilir).
</ParamField>

```json5
{
  memory: {
    search: {
      provider: "openai-compatible",
      model: "text-embedding-3-small",
      remote: {
        baseUrl: "https://api.example.com/v1/",
        apiKey: "YOUR_KEY",
      },
    },
  },
}
```

---

## Sağlayıcıya özgü yapılandırma

<AccordionGroup>
  <Accordion title="Gemini">
    | Anahtar                | Tür      | Varsayılan             | Açıklama                                    |
    | ---------------------- | -------- | ---------------------- | ------------------------------------------- |
    | `model`                | `string` | `gemini-embedding-001` | `gemini-embedding-2-preview` desteği de vardır |
    | `outputDimensionality` | `number` | `3072`                 | Embedding 2 için: 768, 1536 veya 3072       |

    <Warning>
    Modelin veya `outputDimensionality` değerinin değiştirilmesi dizin kimliğini değiştirir. OpenClaw,
    bellek dizinini açıkça yeniden oluşturana kadar vektör aramasını duraklatır.
    </Warning>

  </Accordion>
  <Accordion title="OpenAI uyumlu girdi türleri">
    OpenAI uyumlu gömme uç noktaları, sağlayıcıya özgü `input_type` istek alanlarını etkinleştirebilir. Bu, sorgu ve belge gömmeleri için farklı etiketler gerektiren asimetrik gömme modellerinde kullanışlıdır.

    | Anahtar             | Tür      | Varsayılan | Açıklama                                                    |
    | ------------------- | -------- | ---------- | ----------------------------------------------------------- |
    | `inputType`         | `string` | ayarlanmamış | Sorgu ve belge gömmeleri için paylaşılan `input_type` |
    | `queryInputType`    | `string` | ayarlanmamış | Sorgu zamanı `input_type`; `inputType` değerini geçersiz kılar |
    | `documentInputType` | `string` | ayarlanmamış | Dizin/belge `input_type`; `inputType` değerini geçersiz kılar |

    ```json5
    {
      memory: {
        search: {
          provider: "openai-compatible",
          remote: {
            baseUrl: "https://embeddings.example/v1",
            apiKey: "${EMBEDDINGS_API_KEY}",
          },
          model: "asymmetric-embedder",
          queryInputType: "query",
          documentInputType: "passage",
        },
      },
    }
    ```

    Bu değerlerin değiştirilmesi, sağlayıcının toplu indekslemesi için gömme önbelleği kimliğini etkiler ve üst model etiketleri farklı şekilde ele alıyorsa ardından belleğin yeniden indekslenmesi gerekir.

  </Accordion>
  <Accordion title="Bedrock">
    ### Bedrock gömme yapılandırması

    Bedrock, AWS SDK varsayılan kimlik bilgisi zincirini ve OpenClaw tarafından denetlenen bir taşıyıcı belirteci kullanır; bu nedenle yapılandırmada API anahtarı saklanmaz. OpenClaw, Bedrock'ın etkinleştirildiği bir bulut sunucusu rolüne sahip EC2 üzerinde çalışıyorsa yalnızca sağlayıcıyı ve modeli ayarlayın:

    ```json5
    {
      memory: {
        search: {
          provider: "bedrock",
          model: "amazon.titan-embed-text-v2:0",
        },
      },
    }
    ```

    | Anahtar                | Tür      | Varsayılan                     | Açıklama                         |
    | ---------------------- | -------- | ------------------------------ | -------------------------------- |
    | `model`     | `string` | `amazon.titan-embed-text-v2:0` | Herhangi bir Bedrock gömme modeli kimliği |
    | `outputDimensionality`     | `number` | model varsayılanı              | Titan V2 için: 256, 512 veya 1024 |

    **Desteklenen modeller** (aile algılama ve varsayılan boyutlarla):

    | Model kimliği                               | Sağlayıcı  | Varsayılan boyutlar | Yapılandırılabilir boyutlar |
    | ------------------------------------------- | ---------- | ------------------- | --------------------------- |
    | `amazon.titan-embed-text-v2:0`                          | Amazon     | 1024                | 256, 512, 1024              |
    | `amazon.titan-embed-text-v1`                          | Amazon     | 1536                | --                          |
    | `amazon.titan-embed-g1-text-02`                          | Amazon     | 1536                | --                          |
    | `amazon.titan-embed-image-v1`                          | Amazon     | 1024                | --                          |
    | `amazon.nova-2-multimodal-embeddings-v1:0`                          | Amazon     | 1024                | 256, 384, 1024, 3072        |
    | `cohere.embed-english-v3`                          | Cohere     | 1024                | --                          |
    | `cohere.embed-multilingual-v3`                          | Cohere     | 1024                | --                          |
    | `cohere.embed-v4:0`                          | Cohere     | 1536                | 256, 384, 512, 768, 1024, 1536 |
    | `twelvelabs.marengo-embed-3-0-v1:0`                          | TwelveLabs | 512                 | --                          |
    | `twelvelabs.marengo-embed-2-7-v1:0`                          | TwelveLabs | 1024                | --                          |

    Aktarım hızı son ekli değişkenler (ör. `amazon.titan-embed-text-v1:2:8k`) ve bölge ön ekli çıkarım profili kimlikleri (ör. `us.amazon.titan-embed-text-v2:0`), temel modelin yapılandırmasını devralır.

    **Bölge:** şu sırayla çözümlenir: `memory.search.remote.baseUrl` geçersiz kılması, `models.providers.amazon-bedrock.baseUrl` yapılandırması, `AWS_REGION`, `AWS_DEFAULT_REGION` ve ardından `us-east-1` varsayılanı.

    **Kimlik doğrulama:** OpenClaw önce `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` veya `AWS_BEARER_TOKEN_BEDROCK` değerlerini denetler, ardından standart AWS SDK varsayılan kimlik bilgisi sağlayıcı zincirine geçer:

    1. Ortam değişkenleri (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`), `AWS_PROFILE` da ayarlanmamışsa
    2. SSO (yalnızca SSO alanları yapılandırıldığında)
    3. Paylaşılan kimlik bilgileri ve yapılandırma dosyaları (`fromIni`, `AWS_PROFILE` dahil)
    4. Kimlik bilgisi işlemi (AWS yapılandırma dosyasındaki `credential_process`)
    5. Web kimlik belirteci kimlik bilgileri
    6. ECS veya EC2 bulut sunucusu meta verisi kimlik bilgileri

    **IAM izinleri:** IAM rolü veya kullanıcısı şunlara ihtiyaç duyar:

    ```json
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "*"
    }
    ```

    En az ayrıcalık için `InvokeModel` kapsamını belirli modelle sınırlandırın:

    ```text
    arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
    ```

  </Accordion>
  <Accordion title="Yerel (GGUF + llama.cpp)">
    | Anahtar               | Tür                | Varsayılan              | Açıklama                                                                                                                                                                                                                                                                                                                                 |
    | --------------------- | ------------------ | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `local.modelPath`    | `string` | otomatik indirilir       | GGUF model dosyasının yolu                                                                                                                                                                                                                                                                                                               |
    | `local.modelCacheDir`    | `string` | node-llama-cpp varsayılanı | İndirilen modellerin önbellek dizini                                                                                                                                                                                                                                                                                                   |
    | `local.contextSize`    | `number \| "auto"` | `4096`       | Gömme bağlamının bağlam penceresi boyutu. 4096, ağırlık dışı VRAM kullanımını sınırlarken tipik parçaları (128-512 belirteç) kapsar. Kaynakları kısıtlı ana makinelerde 1024-2048'e düşürün. `"auto"`, modelin eğitildiği azami değeri kullanır -- 8B+ modeller için önerilmez (Qwen3-Embedding-8B: 40 960 belirtece kadar çıkılması VRAM kullanımını ~32 GB'a yükseltebilir). |

    Önce resmî llama.cpp sağlayıcısını yükleyin: `openclaw plugins install @openclaw/llama-cpp-provider`.
    Varsayılan model: `embeddinggemma-300m-qat-Q8_0.gguf` (~0.6 GB, otomatik indirilir). Kaynak kod kopyaları yine de yerel derleme onayı gerektirir: `pnpm approve-builds` ve ardından `pnpm rebuild node-llama-cpp`.

    Gateway'in kullandığı sağlayıcı yolunu doğrulamak için bağımsız CLI'yi kullanın:

    ```bash
    openclaw memory status --deep --agent main
    openclaw memory index --force --agent main
    ```

    Sayısal `local.contextSize` değerleri, model ağırlıklarıyla istenen gömme bağlamının birlikte sığdırılması için node-llama-cpp'nin otomatik GPU katmanı yerleşimini de belirler. `openclaw memory status --deep`, çalışma zamanı yüklendikten sonra bilinen son llama.cpp arka ucunu, cihazı, aktarımı, istenen bağlamı ve zaman damgalı bellek bilgilerini bildirir; pasif durum denetimi bir model yüklemez.

    Yerel GGUF gömmeleri için `provider: "local"` değerini açıkça ayarlayın. `hf:` ve HTTP(S) model başvuruları, açık yerel yapılandırmalar için (node-llama-cpp'nin model çözümlemesi aracılığıyla) desteklenir ancak varsayılan sağlayıcıyı değiştirmez.

  </Accordion>
</AccordionGroup>

## İndeksleme davranışı

Bellek motorları eşitleme, toplu işleme, izleme ve Compaction sonrası
indeksleme buluşsal yöntemlerini yönetir. OpenClaw, kurulum başına zamanlama
anahtarları sunmak yerine bu davranışları bakımı yapılan varsayılanlarla etkin tutar.

## Karma arama yapılandırması

Tümü `memory.search.query` altında:

| Anahtar      | Tür      | Varsayılan | Açıklama                                      |
| ------------ | -------- | ---------- | --------------------------------------------- |
| `maxResults` | `number` | `6` | Eklemeden önce döndürülen azami bellek eşleşmesi |
| `minScore` | `number` | `0.35` | Bir eşleşmeyi dahil etmek için asgari alaka puanı |

Karma getirme etkin kalır; MMR ve zamansal azalma, yerleşik motor politikası
tarafından devre dışı bırakılmış olarak kalır.

### Tam örnek

```json5
{
  memory: {
    search: {
      query: {
        maxResults: 6,
        minScore: 0.35,
      },
    },
  },
}
```

---

## Ek bellek yolları

| Anahtar      | Tür        | Açıklama                                |
| ------------ | ---------- | --------------------------------------- |
| `extraPaths` | `string[]` | İndekslenecek ek dizinler veya dosyalar |

```json5
{
  memory: {
    search: {
      extraPaths: ["../team-docs", "/srv/shared-notes"],
    },
  },
}
```

Yollar mutlak veya çalışma alanına göre olabilir. Dizinler, `.md` dosyaları için özyinelemeli olarak taranır. Sembolik bağlantıların işlenmesi etkin arka uca bağlıdır: yerleşik motor sembolik bağlantıları atlarken QMD, temel QMD tarayıcısının davranışını izler.

Aracı kapsamındaki aracılar arası transkript araması için `memory.qmd.paths` yerine `agents.entries.*.memory.search.qmd.extraCollections` kullanın. Bu ek koleksiyonlar aynı `{ path, name, pattern? }` biçimini izler ancak aracı başına birleştirilir ve yol geçerli çalışma alanının dışını gösterdiğinde açık paylaşılan adları koruyabilir. Aynı çözümlenmiş yol hem `memory.qmd.paths` hem de `memory.search.qmd.extraCollections` içinde görünürse QMD ilk girdiyi tutar ve yineleneni atlar.

---

## Çok modlu bellek (Gemini)

Gemini Embedding 2 kullanarak Markdown'ın yanı sıra görüntüleri ve sesleri de indeksleyin:

| Anahtar                   | Tür        | Varsayılan | Açıklama                                 |
| ------------------------- | ---------- | ---------- | ---------------------------------------- |
| `multimodal.enabled`        | `boolean` | `false` | Çok modlu indekslemeyi etkinleştirir |
| `multimodal.modalities`        | `string[]` | --         | `["image"]`, `["audio"]` veya `["all"]` |
| `multimodal.maxFileBytes`        | `number` | `10485760` | İndeksleme için azami dosya boyutu (10 MiB) |

<Note>
Yalnızca `extraPaths` içindeki dosyalara uygulanır. Varsayılan bellek kökleri yalnızca Markdown olarak kalır. `gemini-embedding-2-preview` gerektirir. `fallback`, `"none"` olmalıdır.
</Note>

Desteklenen biçimler: `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif` (görüntüler); `.mp3`, `.wav`, `.ogg`, `.opus`, `.m4a`, `.aac`, `.flac` (ses).

---

## Gömme önbelleği

| Anahtar         | Tür       | Varsayılan | Açıklama                              |
| --------------- | --------- | ---------- | ------------------------------------- |
| `cache.enabled` | `boolean` | `true` | Parça gömmelerini SQLite'ta önbelleğe alır |

Yeniden indeksleme veya transkript güncellemeleri sırasında değişmemiş metnin yeniden gömülmesini önler.

---

## Toplu indeksleme

| Anahtar                      | Tür       | Varsayılan | Açıklama                   |
| ---------------------------- | --------- | ---------- | -------------------------- |
| `remote.nonBatchConcurrency`           | `number` | `4` | Paralel satır içi gömmeler |
| `remote.batch.enabled`           | `boolean` | `false` | Toplu gömme API'sini etkinleştirir |

`gemini`, `openai` ve `voyage` için kullanılabilir. OpenAI toplu işleme, büyük geriye dönük doldurma işlemleri için genellikle en hızlı ve en ucuz seçenektir.

Eşzamanlılık, yoklama ve zaman aşımı davranışı sağlayıcı tarafından yönetilir.

---

## Oturum belleği araması

Oturum transkriptlerini indeksleyin ve `memory_search` aracılığıyla kullanıma sunun:

| Anahtar                       | Tür        | Varsayılan | Açıklama                                      |
| ----------------------------- | ---------- | ---------- | --------------------------------------------- |
| `rememberAcrossConversations`            | `boolean` | `false` | Konuşmalar arasında özel hatırlamaya izin verir |
| `sources`            | `string[]` | `["memory"]` | Transkriptleri dahil etmek için `"sessions"` ekler |

<Warning>
Oturum indeksleme isteğe bağlıdır ve eşzamansız çalışır. Sonuçlar biraz güncelliğini yitirmiş olabilir. Oturum günlükleri diskte bulunur; bu nedenle dosya sistemi erişimini güven sınırı olarak kabul edin.
</Warning>

Model tarafından çağrılan sıradan oturum transkripti araması
[`tools.sessions.visibility`](/tr/gateway/config-tools#toolssessions) ayarına uyar. Varsayılan
`tree` görünürlüğü geçerli oturumu, onun başlattığı oturumları ve
ortam grup farkındalığı aracılığıyla izlenen aynı ajana ait grup oturumlarını açığa çıkarır. İlgisiz diğer
oturumlar `agent` görünürlüğünü gerektirir (veya yalnızca ajanlar arası
hatırlama da gerektiğinde ve ajanlar arası politika buna izin verdiğinde `all`).

`rememberAcrossConversations` bu ayarın kapsamını genişletmez. Sınırlı Active Memory
geçişi sırasında aynı ajana ait özel transkriptlerle sınırlı, yalnızca çalışma zamanında
geçerli ayrı bir yetkilendirme sağlar.

Aşağıdaki örneklerde bu ayarlar üst düzey `memory.search` altında yer alır. Yalnızca bir
ajanın oturum transkriptlerini indekslemesi ve araması gerekiyorsa eşdeğer ayarları ajan başına
`memory.search` geçersiz kılmasında da uygulayabilirsiniz.

Aynı ajan için gateway'den DM'e hatırlama:

<Tabs>
  <Tab title="Yerleşik arka uç">
    ```json5
    {
      memory: {
        search: {
          experimental: { sessionMemory: true },
          sources: ["memory", "sessions"],
        },
      },
      tools: {
        sessions: { visibility: "agent" },
      },
    }
    ```
  </Tab>
  <Tab title="QMD arka ucu">
    ```json5
    {
      memory: {
        backend: "qmd",
        search: {
          experimental: { sessionMemory: true },
          sources: ["memory", "sessions"],
        },
        qmd: {
          sessions: { enabled: true },
        },
      },
      tools: {
        sessions: { visibility: "agent" },
      },
    }
    ```
  </Tab>
</Tabs>

QMD kullanılırken `sources: ["sessions"]` tek başına transkriptleri QMD'ye dışa aktarmaz.
`memory.qmd.sessions.enabled: true` ayarını da yapın. Üst düzey
`rememberAcrossConversations: true` ayarı istisnadır: ilgili ajan için
gerekli QMD oturum dışa aktarımını dolaylı olarak etkinleştirir. Dolaylı dışa aktarımlar özel kalır:
her zaman varsayılan dahili dışa aktarma konumunu kullanırlar (yapılandırılmış
`sessions.exportDir` yalnızca açık dışa aktarımlar için geçerlidir), yalnızca
ilgili ajanın konuşmalar arası hatırlaması sırasında aranırlar ve sıradan `memory_get`
bunları okuyamaz. Açık
`memory.qmd.sessions.enabled: true` mevcut davranışını korur ve
dışa aktarılan transkriptleri sıradan bellek bütüncesinin parçası hâline getirir.

---

## SQLite vektör hızlandırması (sqlite-vec)

| Anahtar                      | Tür       | Varsayılan | Açıklama                              |
| ---------------------------- | --------- | ---------- | ------------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`  | Vektör sorguları için sqlite-vec kullan |
| `store.vector.extensionPath` | `string`  | bundled | sqlite-vec yolunu geçersiz kıl        |

sqlite-vec kullanılamadığında OpenClaw otomatik olarak işlem içi kosinüs benzerliğine geri döner.

---

## İndeks depolama

Yerleşik bellek indeksleri her ajanın OpenClaw SQLite veritabanında
`agents/<agentId>/agent/openclaw-agent.sqlite` konumunda bulunur.

| Anahtar               | Tür      | Varsayılan | Açıklama                                  |
| --------------------- | -------- | ---------- | ----------------------------------------- |
| `store.fts.tokenizer` | `string` | `unicode61` | FTS5 belirteçleştiricisi (`unicode61` veya `trigram`) |

---

## QMD arka uç yapılandırması

Etkinleştirmek için `memory.backend = "qmd"` ayarını yapın. Tüm QMD ayarları `memory.qmd` altında bulunur:

| Anahtar                  | Tür       | Varsayılan | Açıklama                                                                                     |
| ------------------------ | --------- | ---------- | -------------------------------------------------------------------------------------------- |
| `command`                | `string`  | `qmd`    | QMD yürütülebilir dosya yolu; hizmet `PATH` değeriniz kabuğunuzdan farklıysa mutlak yol ayarlayın |
| `searchMode`             | `string`  | `search` | Arama komutu: `search`, `vsearch`, `query`                                          |
| `rerank`                 | `boolean` | --       | QMD yeniden sıralamasını atlamak için `searchMode: "query"` ve QMD 2.1+ ile `false` olarak ayarlayın          |
| `includeDefaultMemory`   | `boolean` | `true`   | `MEMORY.md` + `memory/**/*.md` öğelerini otomatik indeksle                                             |
| `paths[]`                | `array`   | --       | Ek yollar: `{ name, path, pattern? }`                                               |
| `sessions.enabled`       | `boolean` | `false`  | Oturum transkriptlerini QMD'ye dışa aktar                                                   |
| `sessions.retentionDays` | `number`  | --       | Transkript saklama süresi                                                                  |
| `sessions.exportDir`     | `string`  | --       | Dışa aktarma dizini                                                                        |

`searchMode: "search"` yalnızca sözcüksel/BM25'tir. OpenClaw, `memory status --deep` sırasında da dâhil olmak üzere bu mod için anlamsal vektör hazır olma yoklamaları veya QMD gömme bakımı çalıştırmaz; `vsearch` ve `query` QMD vektör hazırlığını ve gömmeleri gerektirmeye devam eder.

`rerank: false` yalnızca QMD `query` modunu değiştirir ve QMD 2.1 veya daha yenisini gerektirir. Doğrudan CLI modunda OpenClaw `--no-rerank` değerini; mcporter destekli MCP modunda ise QMD'nin birleşik sorgu aracına `rerank: false` değerini iletir. QMD'nin varsayılan sorgu yeniden sıralama davranışını kullanmak için ayarlamadan bırakın.

OpenClaw güncel QMD koleksiyon ve MCP sorgu biçimlerini tercih eder ancak gerektiğinde uyumlu koleksiyon kalıbı bayraklarını ve eski MCP araç adlarını deneyerek eski QMD sürümlerinin çalışmasını sağlar. QMD birden fazla koleksiyon filtresini desteklediğini bildirdiğinde aynı kaynağa ait koleksiyonlar tek bir QMD işlemiyle aranır; eski QMD derlemeleri koleksiyon başına uyumluluk yolunu korur. Aynı kaynak, kalıcı bellek koleksiyonlarının (varsayılan bellek dosyaları ve özel yollar) birlikte gruplandırılması anlamına gelir; oturum transkripti koleksiyonları ise kaynak çeşitlendirmesinin her iki girdiyi de koruması için ayrı bir grup olarak kalır.

<Note>
QMD model geçersiz kılmaları OpenClaw yapılandırmasında değil, QMD tarafında kalır. QMD modellerini genel olarak geçersiz kılmanız gerekiyorsa gateway çalışma zamanı ortamında `QMD_EMBED_MODEL`, `QMD_RERANK_MODEL` ve `QMD_GENERATE_MODEL` gibi ortam değişkenlerini ayarlayın.
</Note>

<AccordionGroup>
  <Accordion title="Sınırlar">
    | Anahtar                     | Tür      | Varsayılan | Açıklama                         |
    | --------------------------- | -------- | ---------- | -------------------------------- |
    | `limits.maxResults`       | `number` | `4`     | En fazla arama sonucu            |
    | `limits.maxSnippetChars`  | `number` | `450`   | Parçacık uzunluğunu sınırla      |
    | `limits.maxInjectedChars` | `number` | `2200`  | Eklenen toplam karakteri sınırla |
    | `limits.timeoutMs`        | `number` | `4000`  | `memory_search` dâhil, QMD destekli arama sırasında QMD komutu zaman aşımı; kurulum, eşitleme, yerleşik geri dönüş ve tamamlayıcı çalışmalar varsayılan araç son tarihini korur |
  </Accordion>
  <Accordion title="Kapsam">
    Hangi oturumların QMD arama sonuçlarını alabileceğini denetler. [`session.sendPolicy`](/tr/gateway/config-agents#session) ile aynı şema:

    ```json5
    {
      memory: {
        qmd: {
          scope: {
            default: "deny",
            rules: [{ action: "allow", match: { chatType: "direct" } }],
          },
        },
      },
    }
    ```

    Sunulan varsayılan yalnızca DM/doğrudan oturumlara izin verir; grupları ve diğer kanal türlerini reddeder. `match.keyPrefix` normalleştirilmiş oturum anahtarıyla; `match.rawKeyPrefix` ise `agent:<id>:` dâhil ham anahtarla eşleşir.

  </Accordion>
  <Accordion title="Atıflar">
    `memory.citations` tüm arka uçlar için geçerlidir:

    | Değer             | Davranış                                                 |
    | ----------------- | -------------------------------------------------------- |
    | `auto` (varsayılan) | Parçacıklara `Source: <path#line>` alt bilgisini ekle |
    | `on`             | Alt bilgiyi her zaman ekle                         |
    | `off`            | Alt bilgiyi çıkar (yol yine de ajana dahili olarak iletilir) |

  </Accordion>
</AccordionGroup>

QMD, bellek ilk kez kullanıldığında tembel olarak başlatılır; yenileme ve gömme zamanlamalarını bağdaştırıcısı yönetir.

### Tam QMD örneği

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 4, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming

Dreaming, `memory.search` altında değil, `plugins.entries.memory-core.config.dreaming` altında yapılandırılır.

Dreaming, zamanlanmış tek bir tarama olarak çalışır ve dahili hafif/derin/REM aşamalarını uygulama ayrıntısı olarak kullanır.

Kavramsal davranış ve eğik çizgi komutları için [Dreaming](/tr/concepts/dreaming) sayfasına bakın.

### Kullanıcı ayarları

| Anahtar                                | Tür       | Varsayılan    | Açıklama                                                                                                                        |
| -------------------------------------- | --------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                              | `boolean` | `false`       | Dreaming'i tamamen etkinleştirin veya devre dışı bırakın                                                                        |
| `frequency`                            | `string`  | `0 3 * * *`   | Tam Dreaming taraması için isteğe bağlı cron sıklığı                                                                             |
| `model`                                | `string`  | varsayılan model | İsteğe bağlı Dream Diary alt ajan model geçersiz kılması                                                                         |
| `phases.deep.maxPromotedSnippetTokens` | `number`  | `160`         | `MEMORY.md` içine yükseltilen her kısa süreli hatırlama parçacığından tutulan tahmini en fazla belirteç sayısı; kaynak metadata'sı görünür kalır |

### Örnek

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        subagent: {
          allowModelOverride: true,
          allowedModels: ["anthropic/claude-sonnet-4-6"],
        },
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *",
            model: "anthropic/claude-sonnet-4-6",
          },
        },
      },
    },
  },
}
```

<Note>
- Dreaming, makine durumunu `memory/.dreams/` konumuna yazar.
- Dreaming, insan tarafından okunabilen anlatı çıktısını `DREAMS.md` (veya mevcut `dreams.md`) konumuna yazar.
- `dreaming.model` mevcut Plugin alt ajan güvenlik kapısını kullanır; etkinleştirmeden önce `plugins.entries.memory-core.subagent.allowModelOverride: true` ayarını yapın.
- Yapılandırılan model kullanılamadığında Dream Diary, oturumun varsayılan modeliyle bir kez daha dener. Güven veya izin listesi hataları günlüğe kaydedilir ve sessizce yeniden denenmez.
- Hafif/derin/REM aşama politikası ve eşikleri, kullanıcıya yönelik yapılandırma değil, dahili davranıştır.

</Note>

## İlgili

- [Yapılandırma başvurusu](/tr/gateway/configuration-reference)
- [Belleğe genel bakış](/tr/concepts/memory)
- [Bellek araması](/tr/concepts/memory-search)
