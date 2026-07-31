---
read_when:
    - Amazon Bedrock modellerini OpenClaw ile kullanmak istiyorsunuz
    - Model çağrıları için AWS kimlik bilgileri/bölge yapılandırması gerekir
summary: Amazon Bedrock (Converse API) modellerini OpenClaw ile kullanma
title: Amazon Bedrock
x-i18n:
    generated_at: "2026-07-26T22:57:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9cbc9534c0d06e0d5642b8d167c633c16880908812b97adbbf9c6bd6c5511603
    source_path: providers/bedrock.md
    workflow: 16
---

OpenClaw, **Bedrock Converse** akış sağlayıcısı aracılığıyla **Amazon Bedrock** modellerini kullanabilir. Bedrock kimlik doğrulaması API anahtarı yerine **AWS SDK varsayılan kimlik bilgisi zincirini** kullanır.

| Özellik  | Değer                                                               |
| -------- | ------------------------------------------------------------------- |
| Sağlayıcı | `amazon-bedrock`                                                  |
| API      | `bedrock-converse-stream`                                                  |
| Kimlik doğrulama | AWS kimlik bilgileri (ortam değişkenleri, paylaşılan yapılandırma veya örnek rolü) |
| Bölge    | `AWS_REGION` veya `AWS_DEFAULT_REGION` (varsayılan: `us-east-1`) |

## Başlarken

Tercih ettiğiniz kimlik doğrulama yöntemini seçin ve kurulum adımlarını izleyin.

<Tabs>
  <Tab title="Erişim anahtarları / ortam değişkenleri">
    **En uygun olduğu durumlar:** AWS kimlik bilgilerini doğrudan yönettiğiniz geliştirici makineleri, CI veya ana makineler.

    <Steps>
      <Step title="Gateway ana makinesinde AWS kimlik bilgilerini ayarlayın">
        ```bash
        export AWS_ACCESS_KEY_ID="EXAMPLE_AWS_ACCESS_KEY_ID"
        export AWS_SECRET_ACCESS_KEY="..."
        export AWS_REGION="us-east-1"
        # İsteğe bağlı:
        export AWS_SESSION_TOKEN="..."
        export AWS_PROFILE="your-profile"
        # İsteğe bağlı (Bedrock API anahtarı/taşıyıcı belirteci):
        export AWS_BEARER_TOKEN_BEDROCK="..."
        ```
      </Step>
      <Step title="Yapılandırmanıza bir Bedrock sağlayıcısı ve modeli ekleyin">
        `apiKey` gerekli değildir. Sağlayıcıyı `auth: "aws-sdk"` ile yapılandırın:

        ```json5
        {
          models: {
            providers: {
              "amazon-bedrock": {
                baseUrl: "https://bedrock-runtime.us-east-1.amazonaws.com",
                api: "bedrock-converse-stream",
                auth: "aws-sdk",
                models: [
                  {
                    id: "us.anthropic.claude-opus-4-6-v1",
                    name: "Claude Opus 4.6 (Bedrock)",
                    reasoning: true,
                    input: ["text", "image"],
                    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                    contextWindow: 200000,
                    maxTokens: 8192,
                  },
                ],
              },
            },
          },
          agents: {
            defaults: {
              model: { primary: "amazon-bedrock/us.anthropic.claude-opus-4-6-v1" },
            },
          },
        }
        ```
      </Step>
      <Step title="Modellerin kullanılabilir olduğunu doğrulayın">
        ```bash
        openclaw models list
        ```
      </Step>
    </Steps>

    <Tip>
    Ortam işaretçisiyle kimlik doğrulamada (`AWS_ACCESS_KEY_ID`, `AWS_PROFILE` veya `AWS_BEARER_TOKEN_BEDROCK`), OpenClaw ek yapılandırma olmadan model keşfi için örtük Bedrock sağlayıcısını otomatik olarak etkinleştirir.
    </Tip>

  </Tab>

  <Tab title="EC2 örnek rolleri (IMDS)">
    **En uygun olduğu durumlar:** Kimlik doğrulama için örnek meta veri hizmetini kullanan ve IAM rolü iliştirilmiş EC2 örnekleri.

    <Steps>
      <Step title="Keşfi açıkça etkinleştirin">
        IMDS kullanılırken OpenClaw, AWS kimlik doğrulamasını yalnızca ortam işaretçilerinden algılayamaz; bu nedenle açıkça etkinleştirmeniz gerekir:

        ```bash
        openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
        openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1
        ```
      </Step>
      <Step title="İsteğe bağlı olarak otomatik mod için ortam işaretçisi ekleyin">
        Ortam işaretçisiyle otomatik algılama yolunun da çalışmasını istiyorsanız (örneğin `openclaw status` yüzeyleri için):

        ```bash
        export AWS_PROFILE=default
        export AWS_REGION=us-east-1
        ```

        Sahte bir API anahtarına **ihtiyacınız yoktur**.
      </Step>
      <Step title="Modellerin keşfedildiğini doğrulayın">
        ```bash
        openclaw models list
        ```
      </Step>
    </Steps>

    <Warning>
    EC2 örneğinize iliştirilmiş IAM rolü şu izinlere sahip olmalıdır:

    - `bedrock:InvokeModel`
    - `bedrock:InvokeModelWithResponseStream`
    - `bedrock:ListFoundationModels` (otomatik keşif için)
    - `bedrock:ListInferenceProfiles` (çıkarım profili keşfi için)

    Alternatif olarak `AmazonBedrockFullAccess` yönetilen politikasını iliştirin.
    </Warning>

    <Note>
    `AWS_PROFILE=default` yalnızca otomatik mod veya durum yüzeyleri için özellikle bir ortam işaretçisi istiyorsanız gereklidir. Asıl Bedrock çalışma zamanı kimlik doğrulama yolu AWS SDK varsayılan zincirini kullandığından, IMDS örnek rolüyle kimlik doğrulama ortam işaretçileri olmadan da çalışır.
    </Note>

  </Tab>
</Tabs>

## Otomatik model keşfi

OpenClaw, **akış** ve **metin çıktısını** destekleyen Bedrock modellerini otomatik olarak keşfedebilir. Keşif, `bedrock:ListFoundationModels` ve `bedrock:ListInferenceProfiles` kullanır ve sonuçlar önbelleğe alınır (varsayılan: 1 saat).

Örtük sağlayıcının etkinleştirilme biçimi:

- Eğer `plugins.entries.amazon-bedrock.config.discovery.enabled`, `true` ise,
  OpenClaw hiçbir AWS ortam işaretçisi bulunmadığında bile keşfi dener.
- Eğer `plugins.entries.amazon-bedrock.config.discovery.enabled` ayarlanmamışsa,
  OpenClaw örtük Bedrock sağlayıcısını yalnızca şu AWS kimlik doğrulama
  işaretçilerinden birini gördüğünde otomatik olarak ekler:
  `AWS_BEARER_TOKEN_BEDROCK`, `AWS_ACCESS_KEY_ID` +
  `AWS_SECRET_ACCESS_KEY` veya `AWS_PROFILE`.
- Asıl Bedrock çalışma zamanı kimlik doğrulama yolu yine AWS SDK varsayılan zincirini kullanır; dolayısıyla
  keşfin etkinleştirilmesi için `enabled: true` gerekmiş olsa bile paylaşılan yapılandırma,
  SSO ve IMDS örnek rolüyle kimlik doğrulama çalışabilir.

<Note>
Açık `models.providers["amazon-bedrock"]` girdileri için OpenClaw, tam çalışma zamanı kimlik doğrulamasını yüklemeye zorlamadan `AWS_BEARER_TOKEN_BEDROCK` gibi AWS ortam işaretçilerinden Bedrock ortam işaretçisi kimlik doğrulamasını erkenden çözümleyebilir. Asıl model çağrısı kimlik doğrulama yolu yine AWS SDK varsayılan zincirini kullanır.
</Note>

<AccordionGroup>
  <Accordion title="Keşif yapılandırma seçenekleri">
    Yapılandırma seçenekleri `plugins.entries.amazon-bedrock.config.discovery` altında bulunur:

    ```json5
    {
      plugins: {
        entries: {
          "amazon-bedrock": {
            config: {
              discovery: {
                enabled: true,
                region: "us-east-1",
                providerFilter: ["anthropic", "amazon"],
                refreshInterval: 3600,
                defaultContextWindow: 32000,
                defaultMaxTokens: 4096,
              },
            },
          },
        },
      },
    }
    ```

    | Seçenek | Varsayılan | Açıklama |
    | ------- | ---------- | -------- |
    | `enabled` | otomatik | Otomatik modda OpenClaw, örtük Bedrock sağlayıcısını yalnızca desteklenen bir AWS ortam işaretçisi gördüğünde etkinleştirir. Keşfi zorlamak için `true` olarak ayarlayın. |
    | `region` | `AWS_REGION` / `AWS_DEFAULT_REGION` / `us-east-1` | Keşif API çağrılarında kullanılan AWS bölgesi. |
    | `providerFilter` | (tümü) | Bedrock sağlayıcı adlarıyla eşleşir (örneğin `anthropic`, `amazon`). |
    | `refreshInterval` | `3600` | Saniye cinsinden önbellek süresi. Önbelleğe almayı devre dışı bırakmak için `0` olarak ayarlayın. |
    | `defaultContextWindow` | `32000` | Bilinen belirteç sınırları olmayan keşfedilmiş modeller için kullanılan bağlam penceresi (model sınırlarınızı biliyorsanız geçersiz kılın). |
    | `defaultMaxTokens` | `4096` | Bilinen belirteç sınırları olmayan keşfedilmiş modeller için kullanılan en fazla çıktı belirteci sayısı (model sınırlarınızı biliyorsanız geçersiz kılın). |

  </Accordion>

  <Accordion title="Bağlam penceresi ve en fazla belirteç sınırları">
    Bedrock `ListFoundationModels` ve `GetFoundationModel` API'leri
    belirteç sınırı meta verisi döndürmez; yalnızca model kimliğini, adını, kiplerini ve yaşam döngüsü
    durumunu döndürür. OpenClaw, oturum yönetiminin, Compaction eşiklerinin ve
    bağlam taşması algılamasının bu modellerde doğru çalışması için popüler Bedrock
    modellerinin (Claude, Nova, Llama, Mistral, DeepSeek ve diğerleri) bilinen bağlam
    pencereleri ve çıktı sınırlarını içeren bir arama tablosuyla birlikte gelir.

    Tabloda bulunmayan keşfedilmiş modeller `defaultContextWindow`
    ve `defaultMaxTokens` değerlerine geri döner. Kullandığınız bir modelin
    doğru sınırları eksikse açık bir
    `models.providers["amazon-bedrock"].models` girdisiyle bu değerleri geçersiz kılın.

  </Accordion>
</AccordionGroup>

## Hızlı kurulum (AWS yolu)

Bu adım adım kılavuz bir IAM rolü oluşturur, Bedrock izinlerini iliştirir,
örnek profilini bağlar ve EC2 ana makinesinde OpenClaw keşfini etkinleştirir.

```bash
# 1. IAM rolünü ve örnek profilini oluşturun
aws iam create-role --role-name EC2-Bedrock-Access \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy --role-name EC2-Bedrock-Access \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess

aws iam create-instance-profile --instance-profile-name EC2-Bedrock-Access
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-Bedrock-Access \
  --role-name EC2-Bedrock-Access

# 2. EC2 örneğinize iliştirin
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxx \
  --iam-instance-profile Name=EC2-Bedrock-Access

# 3. EC2 örneğinde keşfi açıkça etkinleştirin
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# 4. İsteğe bağlı: açıkça etkinleştirmeden otomatik modu kullanmak istiyorsanız bir ortam işaretçisi ekleyin
echo 'export AWS_PROFILE=default' >> ~/.bashrc
echo 'export AWS_REGION=us-east-1' >> ~/.bashrc
source ~/.bashrc

# 5. Modellerin keşfedildiğini doğrulayın
openclaw models list
```

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Çıkarım profilleri">
    OpenClaw, temel modellerle birlikte **bölgesel ve küresel çıkarım profillerini**
    keşfeder. Bir profil bilinen bir temel modelle eşleştiğinde profil, o modelin
    yeteneklerini (bağlam penceresi, en fazla belirteç sayısı, akıl yürütme, görsel işleme)
    devralır ve doğru Bedrock istek bölgesi otomatik olarak eklenir. Böylece bölgeler arası
    Claude profilleri, sağlayıcıyı elle geçersiz kılmaya gerek kalmadan çalışır. Küresel
    bölgeler arası profiller (`global.*`), genellikle daha iyi kapasite
    ve otomatik yük devretme sunduklarından `openclaw models list` içinde ilk sırada listelenir.

    Çıkarım profili kimlikleri `us.anthropic.claude-opus-4-6-v1` (bölgesel)
    veya `anthropic.claude-opus-4-6-v1` (küresel) biçimindedir. Temel model keşif
    sonuçlarında zaten bulunuyorsa profil, modelin tüm yetenek kümesini devralır;
    aksi takdirde güvenli varsayılanlar uygulanır.

    Ek yapılandırma gerekmez. Keşif etkin olduğu ve IAM sorumlusu
    `bedrock:ListInferenceProfiles` iznine sahip olduğu sürece profiller,
    `openclaw models list` içinde temel modellerin yanında görünür.

  </Accordion>

  <Accordion title="Hizmet katmanı">
    Bazı Bedrock modelleri maliyeti veya gecikmeyi optimize etmek için
    `service_tier` parametresini destekler. Şu katmanlar kullanılabilir:

    | Katman | Açıklama |
    |--------|----------|
    | `default` | Standart Bedrock katmanı |
    | `flex` | Daha uzun gecikmeyi tolere edebilen iş yükleri için indirimli işleme |
    | `priority` | Gecikmeye duyarlı iş yükleri için öncelikli işleme |
    | `reserved` | Kararlı durumdaki iş yükleri için ayrılmış kapasite |

    Bedrock model istekleri için `serviceTier` (veya `service_tier`)
    değerini `agents.defaults.params` aracılığıyla ya da model bazında
    `agents.defaults.models["<model-key>"].params` içinde ayarlayın:

    ```json5
    {
      agents: {
        defaults: {
          params: {
            serviceTier: "flex", // tüm modellere uygulanır
          },
          models: {
            "amazon-bedrock/mistral.mistral-large-3-675b-instruct": {
              params: {
                serviceTier: "priority", // model bazında geçersiz kılma
              },
            },
          },
        },
      },
    }
    ```

    Geçerli değerler `default`, `flex`, `priority` ve `reserved` değerleridir. Claude
    Fable 5, Opus 5 ve Sonnet 5 yalnızca `default` katmanını destekler; bu modeller için
    `flex`, `priority` veya `reserved` istendiğinde OpenClaw uyarı verir ve
    isteği yok sayar. Diğer modellerde her model her katmanı desteklemez; desteklenmeyen bir katman
    Bedrock doğrulama hatası döndürür ve hata mesajı yanıltıcı olabilir
    (örneğin, sorunun katman olduğunu belirtmek yerine "The provided model identifier is invalid"
    mesajını verebilir). Bu hatayı görürseniz modelin istenen katmanı destekleyip
    desteklemediğini kontrol edin.

  </Accordion>

  <Accordion title="Claude Opus 5, 4.8 ve 4.7 sıcaklığı">
    Bedrock, Claude Opus 5, Opus 4.8 ve Opus 4.7 için `temperature`
    parametresini reddeder. OpenClaw; temel model kimlikleri, adlandırılmış çıkarım profilleri,
    temelindeki modeli `bedrock:GetInferenceProfile` aracılığıyla Opus 5/4.8/4.7 olarak çözümlenen uygulama
    çıkarım profilleri ve isteğe bağlı bölge ön eklerine sahip noktalı
    `opus-4.7`/`opus-4.8` varyantları (`us.`, `eu.`, `ap.`, `apac.`, `au.`, `jp.`,
    `global.`) dâhil olmak üzere eşleşen tüm Bedrock referanslarında `temperature`
    değerini otomatik olarak çıkarır. Herhangi bir yapılandırma ayarı gerekmez ve bu çıkarma hem
    istek seçenekleri nesnesine hem de `inferenceConfig` yük alanına uygulanır.
  </Accordion>

  <Accordion title="Claude Opus 5">
    Messages-API Bedrock uç noktasında `amazon-bedrock/anthropic.claude-opus-5` değerini
    veya Bedrock keşfinde göründüğünde `global.anthropic.claude-opus-5` gibi bölgesel/küresel
    bir çıkarım profilini kullanın. OpenClaw; 1.000.000 token bağlam penceresini,
    128.000 token çıktı sınırını, görüntü girdisini, istem önbelleğe almayı,
    ret durumuna dayanıklı akışı ve yerel `xhigh`/`max`
    çaba düzeylerini uygular.

    Uyarlanabilir düşünme varsayılan olarak `high` değerini kullanır.
    `/think off` düşünmeyi devre dışı bırakırken `/think xhigh|max` uyarlanabilir düşünmeyi
    etkin tutar. OpenClaw, özel örnekleme parametrelerini ve desteklenmeyen
    varsayılan dışı hizmet katmanlarını çıkarır.

  </Accordion>

  <Accordion title="Claude Fable 5">
    `us-east-1` içinde `amazon-bedrock/anthropic.claude-fable-5` değerini veya
    `us.anthropic.claude-fable-5` gibi bölgesel çıkarım kimliklerini kullanın.
    OpenClaw; Fable'ın 1M bağlam penceresini, 128K çıktı sınırını, her zaman açık
    uyarlanabilir düşünmeyi ve desteklenen çaba eşlemesini uygular. `/think off` ve
    `/think minimal`, `low` değerine eşlenir; sıcaklık ve zorunlu araç seçimi denetimleri
    Opus 4.7/4.8 rotasıyla uyumlu olarak çıkarılır. Akış çıktısı, akışın ortasındaki retlerin
    kısmi metni açığa çıkarmaması için Bedrock bir sonlandırma durumu döndürene kadar
    bekletilir.

    AWS, Fable kullanılabilir olmadan önce açık bir `provider_data_share` veri saklama
    kabulü gerektirir. İstemler ve tamamlamalar Anthropic ile paylaşılır ve
    güven ile emniyet amacıyla 30 güne kadar saklanır. Modeli etkinleştirmeden önce
    [Bedrock veri saklama](https://docs.aws.amazon.com/bedrock/latest/userguide/data-retention.html)
    ayarlarını inceleyip yapılandırın.

  </Accordion>

  <Accordion title="Claude Mythos 5">
    Claude Mythos 5, Bedrock üzerinden yalnızca gerekli sınırlı erişim
    onayına sahip hesaplar tarafından kullanılabilir. OpenClaw, `anthropic.claude-mythos-5`
    temel modelini ve `us.anthropic.claude-mythos-5` gibi bölgesel veya küresel çıkarım
    profillerini tanır.

    OpenClaw; 1.000.000 token bağlam penceresini, 128.000 token çıktı
    sınırını, görüntü girdisini, istem önbelleğe almayı, ret durumuna dayanıklı akışı ve yerel
    çaba düzeylerini uygular. Uyarlanabilir düşünme her zaman etkindir: `/think off` ve
    `/think minimal`, `low` değerine eşlenirken `xhigh` ve `max` kullanılabilir durumda kalır.
    Özel örnekleme ve zorunlu araç seçimi değerleri çıkarılır.

  </Accordion>

  <Accordion title="Claude Sonnet 5">
    AWS, Sonnet 5'i hem
    [`bedrock-runtime` hem de `bedrock-mantle` uç noktaları](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5.html)
    için belgeler. OpenClaw, `anthropic.claude-sonnet-5` Bedrock temel modelini
    ve `us.anthropic.claude-sonnet-5` gibi bölgesel veya küresel çıkarım profillerini
    tanır. 1.000.000 token bağlam penceresini, 128.000 token çıktı sınırını,
    görüntü girdisini, yerel çaba düzeylerini, istem önbelleğe almayı ve
    ret durumuna dayanıklı akışı uygular.

    Bedrock, Sonnet 5 için uyarlanabilir düşünmeyi etkin tutar. OpenClaw varsayılan olarak
    `high` değerini kullanır; bu rota düşünmeyi devre dışı bırakamadığı için
    `/think off` ve `/think minimal`, `low` değerine eşlenir.
    Uyarlanabilir düşünme etkinken özel sıcaklık ve zorunlu araç seçimi değerleri
    çıkarılır.

  </Accordion>

  <Accordion title="Koruma önlemleri">
    `amazon-bedrock` plugin yapılandırmasına bir `guardrail` nesnesi ekleyerek
    [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
    özelliğini tüm Bedrock model çağrılarına uygulayabilirsiniz. Koruma önlemleri; içerik filtreleme,
    konu reddetme, sözcük filtreleri, hassas bilgi filtreleri ve bağlamsal
    temellendirme denetimlerini zorunlu kılmanıza olanak tanır.

    ```json5
    {
      plugins: {
        entries: {
          "amazon-bedrock": {
            config: {
              guardrail: {
                guardrailIdentifier: "abc123", // koruma önlemi kimliği veya tam ARN
                guardrailVersion: "1", // sürüm numarası veya "DRAFT"
                streamProcessingMode: "sync", // isteğe bağlı: "sync" veya "async"
                trace: "enabled", // isteğe bağlı: "enabled", "disabled" veya "enabled_full"
              },
            },
          },
        },
      },
    }
    ```

    `guardrailIdentifier` ve `guardrailVersion` gereklidir.

    | Seçenek | Açıklama |
    | ------ | ----------- |
    | `guardrailIdentifier` | Koruma önlemi kimliği (ör. `abc123`) veya tam ARN (ör. `arn:aws:bedrock:us-east-1:123456789012:guardrail/abc123`). |
    | `guardrailVersion` | Yayımlanmış sürüm numarası veya çalışma taslağı için `"DRAFT"`. |
    | `streamProcessingMode` | Akış sırasında koruma önlemi değerlendirmesi için `"sync"` veya `"async"`. Belirtilmezse Bedrock varsayılanını kullanır. |
    | `trace` | Hata ayıklama için `"enabled"` veya `"enabled_full"`; üretimde belirtmeyin veya `"disabled"` olarak ayarlayın. |

    <Warning>
    Gateway tarafından kullanılan IAM sorumlusu, standart çağırma izinlerine ek olarak `bedrock:ApplyGuardrail` iznine sahip olmalıdır.
    </Warning>

  </Accordion>

  <Accordion title="Bellek araması için gömmeler">
    Bedrock ayrıca [bellek araması](/tr/concepts/memory-search) için gömme sağlayıcısı
    olarak kullanılabilir. Bu, çıkarım sağlayıcısından ayrı olarak yapılandırılır;
    `memory.search.provider` değerini `"bedrock"` olarak ayarlayın:

    ```json5
    {
      memory: {
        search: {
          provider: "bedrock",
          model: "amazon.titan-embed-text-v2:0", // varsayılan
        },
      },
    }
    ```

    Bedrock gömmeleri, çıkarımla aynı AWS SDK kimlik bilgisi zincirini kullanır (örnek
    rolleri, SSO, erişim anahtarları, paylaşılan yapılandırma ve web kimliği). API anahtarı
    gerekmez.

    Desteklenen gömme modelleri arasında Amazon Titan Embed (v1, v2), Amazon Nova
    Embed, Cohere Embed (v3, v4) ve TwelveLabs Marengo bulunur. Tam model listesi
    ve boyut seçenekleri için
    [Bellek yapılandırması referansı -- Bedrock](/tr/reference/memory-config#bedrock-embedding-config)
    bölümüne bakın.

  </Accordion>

  <Accordion title="Notlar ve dikkat edilmesi gerekenler">
    - Bedrock, AWS hesabınızda/bölgenizde **model erişiminin** etkinleştirilmesini gerektirir.
    - Otomatik keşif, `bedrock:ListFoundationModels` ve
      `bedrock:ListInferenceProfiles` izinlerini gerektirir.
    - Otomatik moda güveniyorsanız Gateway ana makinesinde desteklenen AWS kimlik doğrulama ortam
      işaretçilerinden birini ayarlayın. Ortam işaretçileri olmadan IMDS/paylaşılan yapılandırma
      kimlik doğrulamasını tercih ediyorsanız `plugins.entries.amazon-bedrock.config.discovery.enabled: true` değerini ayarlayın.
    - OpenClaw, kimlik bilgisi kaynağını şu sırayla gösterir: `AWS_BEARER_TOKEN_BEDROCK`,
      ardından `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`, ardından `AWS_PROFILE`, ardından
      varsayılan AWS SDK zinciri.
    - Akıl yürütme desteği modele bağlıdır; güncel yetenekler için Bedrock model kartını
      kontrol edin.
    - Yönetilen anahtar akışını tercih ediyorsanız Bedrock'ın önüne OpenAI uyumlu
      bir proxy yerleştirip bunun yerine OpenAI sağlayıcısı olarak yapılandırabilirsiniz.
  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Bellek araması" href="/tr/concepts/memory-search" icon="magnifying-glass">
    Bellek araması yapılandırması için Bedrock gömmeleri.
  </Card>
  <Card title="Bellek yapılandırması referansı" href="/tr/reference/memory-config#bedrock-embedding-config" icon="database">
    Tam Bedrock gömme modeli listesi ve boyut seçenekleri.
  </Card>
  <Card title="Sorun giderme" href="/tr/help/troubleshooting" icon="wrench">
    Genel sorun giderme ve sık sorulan sorular.
  </Card>
</CardGroup>
