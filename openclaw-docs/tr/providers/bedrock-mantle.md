---
read_when:
    - OpenClaw ile Bedrock Mantle üzerinde barındırılan açık kaynaklı modelleri kullanmak istiyorsunuz
    - GPT-OSS, Qwen, Kimi veya GLM için Mantle'ın OpenAI uyumlu uç noktasına ihtiyacınız var
    - Claude Opus 5, Sonnet 5 veya Mythos 5'i Amazon Bedrock Mantle üzerinden kullanmak istiyorsunuz
summary: Amazon Bedrock Mantle'ın OpenAI uyumlu ve Claude Messages modellerini OpenClaw ile kullanın
title: Amazon Bedrock Mantle
x-i18n:
    generated_at: "2026-07-26T22:58:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3d2b49120560c4466aff217c3101fab057dd87c1c501f1b8eb94d74f62bd1037
    source_path: providers/bedrock-mantle.md
    workflow: 16
---

OpenClaw, Mantle'ın OpenAI uyumlu uç noktasına bağlanan yerleşik bir **Amazon Bedrock Mantle** sağlayıcısı içerir. Mantle, Bedrock altyapısıyla desteklenen standart bir
`/v1/chat/completions` yüzeyi üzerinden açık kaynak ve
üçüncü taraf modelleri (GPT-OSS, Qwen, Kimi, GLM ve benzerleri) barındırır. Mantle ayrıca
Anthropic Claude modellerini bir Anthropic Messages rotası üzerinden sunar.

| Özellik        | Değer                                                                                  |
| -------------- | -------------------------------------------------------------------------------------- |
| Sağlayıcı kimliği | `amazon-bedrock-mantle`                                                                |
| API            | Keşfedilen OSS modelleri için `openai-completions`, Claude modelleri için `anthropic-messages` |
| Kimlik doğrulama | Açıkça belirtilen `AWS_BEARER_TOKEN_BEDROCK` veya IAM kimlik bilgisi zinciriyle bearer token oluşturma |
| Varsayılan bölge | `us-east-1` (`AWS_REGION` veya `AWS_DEFAULT_REGION` ile geçersiz kılınabilir) |

## Başlarken

Tercih ettiğiniz kimlik doğrulama yöntemini seçin ve kurulum adımlarını izleyin.

<Tabs>
  <Tab title="Açıkça belirtilen bearer token">
    **En uygun olduğu durum:** Zaten bir Mantle bearer token'ına sahip olduğunuz ortamlar.

    <Steps>
      <Step title="Bearer token'ı Gateway ana makinesinde ayarlayın">
        ```bash
        export AWS_BEARER_TOKEN_BEDROCK="..."
        ```

        İsteğe bağlı olarak bir bölge ayarlayın (varsayılan: `us-east-1`):

        ```bash
        export AWS_REGION="us-west-2"
        ```
      </Step>
      <Step title="Modellerin keşfedildiğini doğrulayın">
        ```bash
        openclaw models list
        ```

        Keşfedilen modeller `amazon-bedrock-mantle` sağlayıcısı altında görünür. Varsayılanları
        geçersiz kılmak istemediğiniz sürece ek yapılandırma gerekmez.
      </Step>
    </Steps>

  </Tab>

  <Tab title="IAM kimlik bilgileri">
    **En uygun olduğu durum:** AWS SDK uyumlu kimlik bilgilerinin kullanılması (paylaşılan yapılandırma, SSO, web kimliği, bulut sunucusu veya görev rolleri).

    <Steps>
      <Step title="AWS kimlik bilgilerini Gateway ana makinesinde yapılandırın">
        AWS SDK uyumlu herhangi bir kimlik doğrulama kaynağı kullanılabilir:

        ```bash
        export AWS_PROFILE="default"
        export AWS_REGION="us-west-2"
        ```
      </Step>
      <Step title="Modellerin keşfedildiğini doğrulayın">
        ```bash
        openclaw models list
        ```

        OpenClaw, kimlik bilgisi zincirinden otomatik olarak bir Mantle bearer token'ı oluşturur.
      </Step>
    </Steps>

    <Tip>
    `AWS_BEARER_TOKEN_BEDROCK` ayarlanmadığında OpenClaw, paylaşılan kimlik bilgileri/yapılandırma profilleri, SSO, web kimliği ve bulut sunucusu veya görev rolleri dâhil olmak üzere AWS varsayılan kimlik bilgisi zincirinden bearer token'ı sizin için oluşturur.
    </Tip>

  </Tab>
</Tabs>

## Otomatik model keşfi

`AWS_BEARER_TOKEN_BEDROCK` ayarlandığında OpenClaw bunu doğrudan kullanır. Aksi takdirde
OpenClaw, AWS varsayılan kimlik bilgisi zincirinden bir Mantle bearer token'ı
oluşturmaya çalışır. Ardından bölgenin `/v1/models` uç noktasını sorgulayarak
kullanılabilir Mantle modellerini keşfeder.

| Davranış          | Ayrıntı                                                                               |
| ----------------- | ------------------------------------------------------------------------------------ |
| Keşif önbelleği   | Sonuçlar bölge başına 1 saat önbelleğe alınır; getirme hatasında önbelleğe alınmış son sonuç döndürülür |
| IAM token yenileme | Bölge başına önbelleğe alınarak her 2 saatte bir                                      |

Mantle Plugin'ini etkin tutarken otomatik keşfi ve IAM
bearer token oluşturmayı engellemek için Plugin'e ait keşif geçişini devre dışı bırakın:

```bash
openclaw config set plugins.entries.amazon-bedrock-mantle.config.discovery.enabled false
```

<Note>
Bearer token, standart [Amazon Bedrock](/tr/providers/bedrock) sağlayıcısı tarafından kullanılan `AWS_BEARER_TOKEN_BEDROCK` ile aynıdır.
</Note>

### Desteklenen bölgeler

`us-east-1`, `us-east-2`, `us-west-2`, `ap-northeast-1`,
`ap-south-1`, `ap-southeast-3`, `eu-central-1`, `eu-west-1`, `eu-west-2`,
`eu-south-1`, `eu-north-1`, `sa-east-1`.

## Manuel yapılandırma

Otomatik keşif yerine açık yapılandırmayı tercih ediyorsanız:

```json5
{
  models: {
    providers: {
      "amazon-bedrock-mantle": {
        baseUrl: "https://bedrock-mantle.us-east-1.api.aws/v1",
        api: "openai-completions",
        auth: "api-key",
        apiKey: "env:AWS_BEARER_TOKEN_BEDROCK",
        models: [
          {
            id: "gpt-oss-120b",
            name: "GPT-OSS 120B",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32000,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

Açıkça belirtilen ve boş olmayan bir `models` listesi belirleyicidir ve aşağıdaki
Claude satırları dâhil keşfedilen tüm satırların yerini alır. Otomatik Mantle
kataloğunu korumak için `models` değerini atlayın veya kullanmak istediğiniz eksiksiz Claude
model girdilerini ekleyin.

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Akıl yürütme desteği">
    Akıl yürütme desteği, model kimliklerinde
    `thinking`, `reasoner`, `reasoning`, `deepseek.r`, `gpt-oss-120b` veya
    `gpt-oss-safeguard-120b` gibi kalıpların bulunmasından çıkarılır. OpenClaw, keşif sırasında
    eşleşen modeller için `reasoning: true` değerini otomatik olarak ayarlar.
  </Accordion>

  <Accordion title="Uç noktanın kullanılamaması">
    Mantle uç noktası kullanılamıyorsa, model döndürmüyorsa veya bearer token
    çözümlemesi başarısız olursa keşif boş bir sonuç döndürür ve örtük
    sağlayıcı atlanır. OpenClaw hata vermez; yapılandırılmış diğer sağlayıcılar
    normal şekilde çalışmaya devam eder.
  </Accordion>

  <Accordion title="Anthropic Messages rotası üzerinden Claude">
    Model listesini otomatik keşif yönetirken OpenClaw, `/v1/models` ne döndürürse döndürsün
    başarılı bir aramanın ardından beş Claude
    modeli ekler:
    `amazon-bedrock-mantle/anthropic.claude-opus-5` (Claude Opus 5),
    `amazon-bedrock-mantle/anthropic.claude-sonnet-5` (Claude Sonnet 5),
    `amazon-bedrock-mantle/anthropic.claude-opus-4-7` (Claude Opus 4.7) ve
    `amazon-bedrock-mantle/anthropic.claude-mythos-5` (Claude Mythos 5), ayrıca
    `amazon-bedrock-mantle/anthropic.claude-mythos-preview` (Claude Mythos
    Preview). Bunlar `anthropic-messages` API yüzeyini kullanır ve bearer ile kimliği doğrulanan aynı Anthropic uyumlu uç nokta
    (`<mantle-base>/anthropic`) üzerinden akış gerçekleştirir; dolayısıyla AWS bearer token'ı bir
    Anthropic API anahtarı olarak değerlendirilmez.

    Claude Opus 5; 1,000,000 token'lık bağlam penceresi, 128,000 token'lık
    çıktı sınırı, görüntü girdisi ve `$5/$25` girdi/çıktı fiyatlandırması sunar. Uyarlanabilir
    düşünmenin varsayılanı `high` değeridir; `/think off` düşünmeyi devre dışı bırakır ve
    `/think xhigh|max` modelin yerel çaba düzeylerini kullanır. OpenClaw,
    çağıranın seçtiği örnekleme parametrelerini atlar.

    Claude Sonnet 5 her zaman uyarlanabilir düşünmeyi kullanır ve varsayılan çaba düzeyi `high`
    değeridir. Mantle rotası düşünmeyi devre dışı bırakamadığından `/think off` ve `/think minimal`, `low` ile eşlenir.
    OpenClaw ayrıca Sonnet 5 istekleri için özel sıcaklık değerini atlar.

    Claude Mythos 5 sınırlı erişime sahiptir. 1,000,000 token'lık bağlam
    penceresi ve 128,000 token'lık çıktı sınırı sunar, her zaman uyarlanabilir düşünmeyi kullanır,
    `/think off` ve `/think minimal` değerlerini `low` ile eşler ve çağıranın seçtiği
    örnekleme parametrelerini atlar.

    Claude Mythos Preview her zaman akıl yürütme ister; hiçbir `/think` düzeyi ayarlanmadığında varsayılan
    çaba düzeyi `high` olur (`xhigh`/`max` değerlerinden `high` değerine düşürülür,
    `minimal` değerinden `low` değerine yükseltilir). Mantle üzerindeki Opus 4.7, model tarafından sağlanan
    akıl yürütme olmadan akış gerçekleştirir ve Opus 4.7 bu rotada örnekleme geçersiz kılmalarını kabul etmediğinden OpenClaw
    `temperature` parametresini atlar; Mythos
    Preview ise bir `temperature` geçersiz kılmasını normal şekilde kabul eder.

    Boş olmayan ve açıkça belirtilmiş bir `models.providers["amazon-bedrock-mantle"].models`
    listesi, keşfedilen kataloğun tamamının yerini alır. Bu yerleşik Claude
    satırlarını istediğinizde bu listeyi atlayın.

  </Accordion>

  <Accordion title="Amazon Bedrock sağlayıcısıyla ilişkisi">
    Bedrock Mantle, standart
    [Amazon Bedrock](/tr/providers/bedrock) sağlayıcısından ayrı bir sağlayıcıdır. Mantle, OSS kataloğu için
    OpenAI uyumlu bir `/v1` yüzeyi kullanırken standart
    Bedrock sağlayıcısı yerel Bedrock Converse API'sini kullanır.

    Her iki sağlayıcı da mevcut olduğunda aynı `AWS_BEARER_TOKEN_BEDROCK` kimlik bilgisini paylaşır.

  </Accordion>
</AccordionGroup>

## İlgili kaynaklar

<CardGroup cols={2}>
  <Card title="Amazon Bedrock" href="/tr/providers/bedrock" icon="cloud">
    Anthropic Claude, Titan ve diğer modeller için yerel Bedrock sağlayıcısı.
  </Card>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıların, model referanslarının ve yük devretme davranışının seçilmesi.
  </Card>
  <Card title="OAuth ve kimlik doğrulama" href="/tr/gateway/authentication" icon="key">
    Kimlik doğrulama ayrıntıları ve kimlik bilgilerini yeniden kullanma kuralları.
  </Card>
  <Card title="Sorun giderme" href="/tr/help/troubleshooting" icon="wrench">
    Yaygın sorunlar ve bunların nasıl çözüleceği.
  </Card>
</CardGroup>
