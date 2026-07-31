---
read_when:
    - Qwen'i OpenClaw ile kullanmak istiyorsunuz
    - Alibaba Cloud Token Plan aboneliğiniz var
summary: Qwen Cloud'u OpenClaw Plugin'i aracılığıyla kullanın
title: Qwen
x-i18n:
    generated_at: "2026-07-26T23:58:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 74f94a35631dcdf8c9afc12e86d7a9d6b51a359411ba36f8820f8b1e7c03a27a
    source_path: providers/qwen.md
    workflow: 16
---

Qwen Cloud, standart kimliği `qwen` olan resmî bir harici OpenClaw sağlayıcı Plugin'idir. Qwen Cloud / Alibaba DashScope Standard ve Coding Plan uç noktalarını hedefler, Token Plan'ı `qwen-token-plan` olarak sunar, `modelstudio` değerini uyumluluk takma adı olarak korur ve Alibaba'nın belgelenmiş `bailian-token-plan` özel sağlayıcı kimliğinin sahipliğini bağımsız olarak üstlenir.

| Özellik                   | Değer                                      |
| ------------------------- | ------------------------------------------ |
| Sağlayıcı                 | `qwen`                                     |
| Token Plan sağlayıcısı    | `qwen-token-plan`                          |
| Tercih edilen ortam değişkeni | `QWEN_API_KEY`                             |
| Token Plan ortam değişkeni | `QWEN_TOKEN_PLAN_API_KEY`                  |
| Ayrıca kabul edilen (uyumluluk) | `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY` |
| API biçimi                | OpenAI uyumlu                              |

<Tip>
`qwen3.7-plus` ve `qwen3.6-plus`, Coding Plan ve Standard uç noktalarıyla çalışır.
`qwen3.7-max` veya `qwen3.6-flash` için bir **Standard (kullandıkça öde)** uç noktası kullanın.
</Tip>

## Plugin'i yükleme

`qwen`, çekirdekle birlikte paketlenmeyen resmî bir harici Plugin olarak sunulur. Plugin'i yükleyin ve Gateway'i yeniden başlatın:

```bash
openclaw plugins install @openclaw/qwen-provider
openclaw gateway restart
```

## Başlarken

Plan türünüzü seçin ve kurulum adımlarını izleyin.

<Tabs>
  <Tab title="Coding Plan (abonelik)">
    **En uygun olduğu kullanım:** Qwen Coding Plan üzerinden aboneliğe dayalı erişim.

    <Steps>
      <Step title="API anahtarınızı alma">
        [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) adresinden bir API anahtarı oluşturun veya kopyalayın.
      </Step>
      <Step title="İlk kurulumu çalıştırma">
        **Global** uç noktası için:

        ```bash
        openclaw onboard --auth-choice qwen-api-key
        ```

        **China** uç noktası için:

        ```bash
        openclaw onboard --auth-choice qwen-api-key-cn
        ```
      </Step>
      <Step title="Varsayılan model ayarlama">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "qwen/qwen3.5-plus" },
            },
          },
        }
        ```
      </Step>
      <Step title="Modelin kullanılabilir olduğunu doğrulama">
        ```bash
        openclaw models list --provider qwen
        ```
      </Step>
    </Steps>

    <Note>
    Eski `modelstudio-*` kimlik doğrulama seçeneği kimlikleri ve `modelstudio/...` model başvuruları
    uyumluluk takma adları olarak çalışmaya devam eder, ancak yeni kurulum akışları standart
    `qwen-*` kimlik doğrulama seçeneği kimliklerini ve `qwen/...` model başvurularını tercih etmelidir. Başka bir `api` değeriyle tam eşleşen
    özel bir `models.providers.modelstudio` girdisi tanımlarsanız, Qwen uyumluluk
    takma adı yerine `modelstudio/...` başvurularının sahipliğini bu özel sağlayıcı üstlenir.
    </Note>

  </Tab>

  <Tab title="Standard (kullandıkça öde)">
    **En uygun olduğu kullanım:** Coding Plan'da kullanılamayan `qwen3.7-max` ve `qwen3.6-flash` dâhil olmak üzere Standard Model Studio uç noktası üzerinden kullandıkça öde erişimi.

    <Steps>
      <Step title="API anahtarınızı alma">
        [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) adresinden bir API anahtarı oluşturun veya kopyalayın.
      </Step>
      <Step title="İlk kurulumu çalıştırma">
        **Global** uç noktası için:

        ```bash
        openclaw onboard --auth-choice qwen-standard-api-key
        ```

        **China** uç noktası için:

        ```bash
        openclaw onboard --auth-choice qwen-standard-api-key-cn
        ```
      </Step>
      <Step title="Varsayılan model ayarlama">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "qwen/qwen3.5-plus" },
            },
          },
        }
        ```
      </Step>
      <Step title="Modelin kullanılabilir olduğunu doğrulama">
        ```bash
        openclaw models list --provider qwen
        ```
      </Step>
    </Steps>

    <Note>
    Eski `modelstudio-*` kimlik doğrulama seçeneği kimlikleri ve `modelstudio/...` model başvuruları
    uyumluluk takma adları olarak çalışmaya devam eder, ancak yeni kurulum akışları standart
    `qwen-*` kimlik doğrulama seçeneği kimliklerini ve `qwen/...` model başvurularını tercih etmelidir. Başka bir `api` değeriyle tam eşleşen
    özel bir `models.providers.modelstudio` girdisi tanımlarsanız, Qwen uyumluluk
    takma adı yerine `modelstudio/...` başvurularının sahipliğini bu özel sağlayıcı üstlenir.
    </Note>

  </Tab>

  <Tab title="Token Plan (Ekip Sürümü)">
    **En uygun olduğu kullanım:** Alibaba Cloud Model Studio üzerinden Qwen'e ve desteklenen üçüncü taraf modellere kredi tabanlı ekip aboneliği erişimi.

    <Steps>
      <Step title="Size özel anahtarı alma">
        Bir Token Plan lisansı atayın ve bu lisansa özel `sk-sp-...` anahtarını oluşturun. Token Plan, Coding Plan ve kullandıkça öde anahtarları birbirinin yerine kullanılamaz. [Global Token Plan genel bakışına](https://www.alibabacloud.com/help/en/model-studio/token-plan-overview) veya [China Token Plan genel bakışına](https://help.aliyun.com/zh/model-studio/token-plan-overview) bakın.
      </Step>
      <Step title="İlk kurulumu çalıştırma">
        Singapur'daki **Global / International** uç noktası için:

        ```bash
        openclaw onboard --auth-choice qwen-token-plan
        ```

        Pekin'deki **China** uç noktası için:

        ```bash
        openclaw onboard --auth-choice qwen-token-plan-cn
        ```
      </Step>
      <Step title="Sağlayıcıyı doğrulama">
        ```bash
        openclaw models list --provider qwen-token-plan
        openclaw agent --model qwen-token-plan/qwen3.7-plus --message "Şununla yanıt ver: token planı hazır"
        ```
      </Step>
    </Steps>

    <Note>
    Alibaba'nın OpenClaw kılavuzu, manuel bir özel
    sağlayıcı için `bailian-token-plan` değerini kullanır. Plugin bu kimliği bir uyumluluk sahibi olarak kaydeder, ancak yeni
    yapılandırmalar `qwen-token-plan` değerini kullanmalıdır. Tam eşleşen özel bir
    `models.providers.bailian-token-plan` girdisi, yapılandırılmış
    aktarımının ve kataloğunun sahipliğini korur; hiçbir zaman standart OpenAI kataloğuyla birleştirilmez.
    </Note>

    <Warning>
    Token Plan'ı yalnızca etkileşimli OpenClaw oturumları için kullanın.
    Cron işleri, gözetimsiz betikler veya uygulama arka uçları için seçmeyin. Alibaba,
    etkileşimsiz kullanımın aboneliği askıya alabileceğini veya API anahtarını iptal edebileceğini belirtir.
    </Warning>

  </Tab>

</Tabs>

## Plan türleri ve uç noktalar

| Plan                       | Bölge  | Kimlik doğrulama seçeneği | Uç nokta                                                         |
| -------------------------- | ------ | -------------------------- | ---------------------------------------------------------------- |
| Coding Plan (abonelik)     | China  | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`                               |
| Coding Plan (abonelik)     | Global | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`                          |
| Standard (kullandıkça öde) | China  | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`                      |
| Standard (kullandıkça öde) | Global | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1`                 |
| Token Plan (Ekip Sürümü)   | China  | `qwen-token-plan-cn`       | `token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`     |
| Token Plan (Ekip Sürümü)   | Global | `qwen-token-plan`          | `token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1` |

Sağlayıcı, kimlik doğrulama seçiminize göre uç noktayı otomatik olarak seçer. Standart
seçenekler `qwen-*` ailesini kullanır; `modelstudio-*` yalnızca uyumluluk amacıyla kalır.
Yapılandırmada özel bir `baseUrl` kullanarak bunu geçersiz kılabilirsiniz.

<Tip>
**Anahtarları yönetin:** [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) |
**Belgeler:** [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)
</Tip>

## Yerleşik katalog

OpenClaw bu statik Qwen kataloğuyla birlikte gelir. Katalog uç noktaya duyarlıdır: Coding
Plan yapılandırmaları, yalnızca Standard uç noktasında çalışan modelleri içermez.

| Model başvurusu            | Girdi       | Bağlam    | Notlar                  |
| -------------------------- | ----------- | --------- | ----------------------- |
| `qwen/qwen3.5-plus`         | metin, görüntü | 1,000,000 | Varsayılan model        |
| `qwen/qwen3.6-flash`         | metin, görüntü | 1,000,000 | Yalnızca Standard uç noktaları |
| `qwen/qwen3.6-plus`         | metin, görüntü | 1,000,000 | Coding Plan + Standard  |
| `qwen/qwen3.7-max`         | metin       | 1,000,000 | Yalnızca Standard uç noktaları |
| `qwen/qwen3.7-plus`         | metin, görüntü | 1,000,000 | Coding Plan + Standard  |
| `qwen/qwen3-max-2026-01-23`         | metin       | 262,144   | Qwen Max serisi         |
| `qwen/qwen3-coder-next`         | metin       | 262,144   | Kodlama                 |
| `qwen/qwen3-coder-plus`         | metin       | 1,000,000 | Kodlama                 |
| `qwen/MiniMax-M2.5`         | metin       | 1,000,000 | Akıl yürütme etkin      |
| `qwen/glm-5`         | metin       | 202,752   | GLM                     |
| `qwen/glm-4.7`         | metin       | 202,752   | GLM                     |
| `qwen/kimi-k2.5`         | metin, görüntü | 262,144 | Alibaba üzerinden Moonshot AI |

<Note>
Bir model statik katalogda bulunsa bile kullanılabilirliği uç noktaya ve faturalandırma planına göre değişebilir.
</Note>

### Token Plan kataloğu

Token Plan, ayrı bir tam dize izin listesi kullanır. Yalnızca görüntü oluşturmaya yönelik plan
modelleri farklı API'ler kullandığından burada yer almaz.

| Model başvurusu                    | Girdi          | Bağlam    |
| ---------------------------------- | -------------- | --------- |
| `qwen-token-plan/qwen3.7-max`                 | metin          | 1,000,000 |
| `qwen-token-plan/qwen3.7-plus`                 | metin, görüntü | 1,000,000 |
| `qwen-token-plan/qwen3.6-plus`                 | metin, görüntü | 1,000,000 |
| `qwen-token-plan/qwen3.6-flash`                 | metin, görüntü | 1,000,000 |
| `qwen-token-plan/deepseek-v4-pro`                 | metin          | 1,000,000 |
| `qwen-token-plan/deepseek-v4-flash`                 | metin          | 1,000,000 |
| `qwen-token-plan/deepseek-v3.2`                 | metin          | 131,072   |
| `qwen-token-plan/kimi-k2.7-code`                 | metin, görüntü | 262,144   |
| `qwen-token-plan/kimi-k2.6`                 | metin, görüntü | 262,144   |
| `qwen-token-plan/kimi-k2.5`                 | metin, görüntü | 262,144   |
| `qwen-token-plan/glm-5.2`                 | metin          | 1,000,000 |
| `qwen-token-plan/glm-5.1`                 | metin          | 202,752   |
| `qwen-token-plan/glm-5`                 | metin          | 202,752   |
| `qwen-token-plan/MiniMax-M2.5`                 | metin          | 196,608   |

## Düşünme denetimleri

`qwen3.7-max`, `qwen3.7-plus`, `qwen3.6-flash` ve `qwen3.6-plus`,
yerleşik katalogda akıl yürütme özelliği etkin modellerdir. `qwen`
ailesindeki akıl yürütme modellerinde sağlayıcı, OpenClaw düşünme düzeylerini DashScope'un üst düzey
`enable_thinking` istek bayrağına eşler: düşünme devre dışı olduğunda `enable_thinking: false`,
diğer tüm düzeylerde `enable_thinking: true` gönderilir. Özel modeller, model girdisinde
`compat.thinkingFormat: "qwen-chat-template"` ayarlayarak alternatif bir sohbet şablonu düşünme yükünü
etkinleştirebilir.

Token Plan modelleri de akıl yürütme yeteneğine sahip olarak işaretlenir. `kimi-k2.7-code` ve
`MiniMax-M2.5` yalnızca düşünme modunda çalıştığından, oturum `/think off` talep etse bile
OpenClaw düşünmeyi etkin tutar. DeepSeek V4, `minimal` ile `high` arasındaki değerleri
hizmetin `high` çaba düzeyine, `xhigh` veya `max` değerlerini ise `max` değerine eşler. GLM 5.2,
`minimal` ile `max` arasındaki tüm aralığı kabul eder; GLM 5.1 ve GLM 5,
`xhigh` değerine kadar olan aralığı kabul eder ve üçü de varsayılan olarak `high` kullanır. Diğer hibrit modeller,
istenen açık/kapalı durumunu izler.

## Çok modlu eklentiler

`qwen` Plugin'i, çok modlu özellikleri yalnızca **Standard** DashScope
uç noktalarında sunar; Coding Plan uç noktalarında sunmaz:

- **Görüntü ve video anlama**: `qwen3.6-plus`
- **Wan video oluşturma**: `wan2.6-t2v` (varsayılan), `wan2.6-i2v`, `wan2.6-r2v`, `wan2.6-r2v-flash`, `wan2.7-r2v`

Medya anlama, yapılandırılmış Qwen kimlik doğrulamasından otomatik olarak çözümlenir; ek
yapılandırma gerekmez. Medya anlamanın çalışması için bir Standard (kullandıkça öde) uç noktasında
olduğunuzdan emin olun.

Qwen'i varsayılan video sağlayıcısı yapmak için:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

Video oluşturma sınırları: İstek başına 1 çıkış videosu, en fazla 1 giriş görüntüsü
(görüntüden videoya), en fazla 4 giriş videosu (videodan videoya), azami 10 saniye
süre. `size`, `aspectRatio`, `resolution`, `audio` ve
`watermark` desteklenir. Referans görüntü/video girişleri uzak http(s) URL'leri gerektirir; DashScope video uç noktası bu referanslar için yüklenen yerel arabellekleri
kabul etmediğinden yerel dosya yolları en başta reddedilir.

<Note>
Paylaşılan araç parametreleri, sağlayıcı seçimi ve yük devretme davranışı için [Video oluşturma](/tr/tools/video-generation) bölümüne bakın.
</Note>

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Qwen 3.6 ve 3.7 kullanılabilirliği">
    `qwen3.7-plus` ve `qwen3.6-plus`, Coding Plan ve Standard uç noktalarında kullanılabilir. `qwen3.7-max` ve `qwen3.6-flash` yalnızca Standard'da kullanılabilir. Standard (kullandıkça öde) uç noktaları şunlardır:

    - Çin: `dashscope.aliyuncs.com/compatible-mode/v1`
    - Global: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

    OpenClaw, `qwen3.7-max` ve `qwen3.6-flash` öğelerini Coding Plan kataloglarına dahil etmez.
    Bir Coding Plan uç noktası bunlardan biri için "unsupported model" hatası döndürürse
    eşleşen Standard uç noktasına ve anahtarına geçin.

  </Accordion>

  <Accordion title="Video oluşturma bölge yönlendirmesi">
    OpenClaw, bir video işi göndermeden önce yapılandırılmış Qwen bölgesini
    eşleşen DashScope AIGC ana makinesiyle eşler:

    - Global/Uluslararası: `https://dashscope-intl.aliyuncs.com`
    - Çin: `https://dashscope.aliyuncs.com`

    Coding Plan veya Standard Qwen ana makinelerinden birini gösteren normal bir
    `models.providers.qwen.baseUrl`, video oluşturmayı yine eşleşen bölgesel
    DashScope video uç noktasına yönlendirir.

  </Accordion>

  <Accordion title="Akış kullanımı uyumluluğu">
    Yerel Qwen uç noktaları, paylaşılan `openai-completions` aktarımında
    akış kullanımı uyumluluğunu bildirir; böylece aynı yerel ana makineleri hedefleyen
    DashScope uyumlu özel sağlayıcı kimlikleri, özellikle yerleşik
    `qwen` sağlayıcı kimliğini gerektirmeden aynı davranışı devralır. Bu, Coding Plan,
    Standard ve Token Plan uç noktaları için geçerlidir:

    - `https://coding.dashscope.aliyuncs.com/v1`
    - `https://coding-intl.dashscope.aliyuncs.com/v1`
    - `https://dashscope.aliyuncs.com/compatible-mode/v1`
    - `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
    - `https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1`
    - `https://token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`

  </Accordion>

  <Accordion title="Yetenek planı">
    `qwen` plugin'i yalnızca kodlama/metin modellerinin değil, Qwen
    Cloud yüzeyinin tamamının sağlayıcı yuvası olarak konumlandırılmaktadır.

    - **Metin/sohbet modelleri:** plugin aracılığıyla kullanılabilir
    - **Araç çağırma, yapılandırılmış çıkış, düşünme:** OpenAI uyumlu aktarımdan devralınır
    - **Görüntü oluşturma:** sağlayıcı-plugin katmanında planlanmaktadır
    - **Görüntü/video anlama:** Standard uç noktasında plugin aracılığıyla kullanılabilir
    - **Konuşma/ses:** sağlayıcı-plugin katmanında planlanmaktadır
    - **Bellek gömmeleri/yeniden sıralama:** gömme bağdaştırıcısı yüzeyi üzerinden planlanmaktadır
    - **Video oluşturma:** paylaşılan video oluşturma yeteneği üzerinden plugin aracılığıyla kullanılabilir

  </Accordion>

  <Accordion title="Ortam ve daemon kurulumu">
    Gateway bir daemon (launchd/systemd) olarak çalışıyorsa `QWEN_API_KEY`
    veya `QWEN_TOKEN_PLAN_API_KEY` öğesinin bu süreç tarafından erişilebilir olduğundan emin olun (örneğin
    `~/.openclaw/.env` içinde veya `env.shellEnv` aracılığıyla).
  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Video oluşturma" href="/tr/tools/video-generation" icon="video">
    Paylaşılan video aracı parametreleri ve sağlayıcı seçimi.
  </Card>
  <Card title="Alibaba Model Studio" href="/tr/providers/alibaba" icon="cloud">
    Aynı DashScope platformundaki paketlenmiş Wan video oluşturma sağlayıcısı.
  </Card>
  <Card title="Sorun giderme" href="/tr/help/troubleshooting" icon="wrench">
    Genel sorun giderme ve SSS.
  </Card>
</CardGroup>
