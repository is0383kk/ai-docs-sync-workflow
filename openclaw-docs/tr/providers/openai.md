---
read_when:
    - OpenClaw'da OpenAI modellerini kullanmak istiyorsunuz
    - API anahtarları yerine Codex abonelik kimlik doğrulamasını kullanmak istiyorsunuz
    - Daha katı GPT-5 ajan yürütme davranışına ihtiyacınız var
summary: OpenClaw'da OpenAI'ı API anahtarları veya Codex aboneliği aracılığıyla kullanın
title: OpenAI
x-i18n:
    generated_at: "2026-07-26T23:58:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 612a36760899e01126364ddca523f0a6340036253cf349ae2755ba15c6451ba6
    source_path: providers/openai.md
    workflow: 16
---

OpenClaw, hem doğrudan API anahtarı kimlik doğrulaması hem de
ChatGPT/Codex abonelik kimlik doğrulaması için `openai` adlı tek bir sağlayıcı kimliği kullanır. `openai/*`, standart model rotasıdır.
Çalışma zamanı ilkesi ayarlanmamış veya `auto` olan gömülü ajan turlarında OpenAI'ın rota
bilgileri, OpenClaw'ın paketle birlikte gelen Codex uygulama sunucusu çalışma zamanını örtük
olarak seçip seçemeyeceğini belirler. Tek başına `openai/*` ön eki bir çalışma zamanı seçmez.

- **Ajan modelleri** - açık `agentRuntime` yapılandırması veya OpenAI'ın örtük rota ilkesiyle seçilen çalışma zamanı üzerinden `openai/*`.
  ChatGPT/Codex aboneliği kullanımı için Codex kimlik doğrulamasıyla oturum açın veya
  anahtar tabanlı faturalandırma istediğinizde bir API anahtarı kimlik doğrulama
  profili yapılandırın.
- **Ajan dışı OpenAI API'leri** - `OPENAI_API_KEY` veya bir `openai` API anahtarı kimlik doğrulama profili
  üzerinden, kullanım başına faturalandırılan doğrudan OpenAI Platform erişimi.
- **Eski yapılandırma** - `codex/*` ve `openai-codex/*` başvuruları,
  `openclaw doctor --fix` tarafından `openai/*` ile model kapsamlı
  `agentRuntime.id: "codex"` biçimine onarılır.

OpenAI, OpenClaw gibi harici araçlarda ve iş akışlarında abonelik OAuth
kullanımını açıkça destekler.

## Kullanım ve maliyet takibi

OpenClaw, abonelik kotası ile Platform API faturalandırmasını ayrı tutar:

- ChatGPT/Codex OAuth; abonelik planını, kota aralıklarını ve kredi bakiyesini gösterir.
- `OPENAI_ADMIN_KEY`, Control UI **Kullanım** bölümünde günlük harcama, istek/token toplamları, en çok kullanılan modeller ve maliyet kategorileri dâhil olmak üzere sağlayıcının bildirdiği 30 günlük kuruluş maliyetini ve tamamlama kullanımını gösterir.
- `OPENAI_PROJECT_ID`, isteğe bağlı olarak Admin API geçmişini tek bir projeyle sınırlar.
- OpenClaw, `OPENAI_API_KEY` veya bir `openai` çıkarım profilini kuruluş API'lerine asla göndermez; bu kimlik bilgileri özel, Azure veya ajan yerel uç noktalarına ait olabilir.

Açıkça belirtilen bir Admin anahtarı OAuth'a göre önceliklidir. Sağlayıcının bildirdiği geçmiş, OpenClaw'ın oturumdan türetilen tahmini maliyetiyle birleştirilmez; diğer istemcilerden gelen API etkinliklerini ve sağlayıcı tarafındaki faturalandırma düzeltmelerini içerebilir.

OpenAI'ın [API Kullanım Panosu](https://help.openai.com/en/articles/10478918) belgeleri, kullanım verileri için kuruluş sahibi olma ve açık Kullanım Panosu izni gereksinimlerini açıklar.

Sağlayıcı, model, çalışma zamanı ve kanal ayrı katmanlardır. Bu etiketler
birbirine karışıyorsa yapılandırmayı değiştirmeden önce [Ajan çalışma zamanları](/tr/concepts/agent-runtimes) sayfasını
okuyun.

## Hızlı seçim

| Hedef                                             | Kullanım                                                           | Notlar                                                              |
| ------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------- |
| ChatGPT/Codex aboneliği, yerel Codex çalışma zamanı | `openai/gpt-5.6-sol`                                               | Yeni abonelik kurulumu; Codex kimlik doğrulamasıyla oturum açın.    |
| Ajan turları için doğrudan API anahtarıyla faturalandırma | `openai/gpt-5.6` ve sıralı bir API anahtarı kimlik doğrulama profili | Yeni API anahtarı kurulumu; yalın doğrudan API kimliği Sol'a çözümlenir. |
| Tam bir GPT-5.6 katmanı seçme                     | `openai/gpt-5.6-sol`, `-terra` veya `-luna`                         | Bu hesapta kullanılabilen katmanlar için `models list` öğesini kontrol edin. |
| GPT-5.6 erişimi olmayan hesap                     | `openai/gpt-5.5`                                                   | Açık kurtarma seçeneği; OpenClaw sessizce alt sürüme geçmez.        |
| Doğrudan API anahtarıyla faturalandırma, açık OpenClaw çalışma zamanı | `openai/gpt-5.6` ve sağlayıcı/model `agentRuntime.id: "openclaw"` | Normal bir `openai` API anahtarı profili seçin.                    |
| En yeni ChatGPT Instant model diğer adı           | `openai/chat-latest`                                               | Yalnızca doğrudan API anahtarı; kararlı varsayılan değil, değişken bir diğer addır. |
| Görsel oluşturma veya düzenleme                    | `openai/gpt-image-2`                                               | `OPENAI_API_KEY` veya Codex OAuth ile çalışır.                     |
| Şeffaf arka planlı görseller                      | `openai/gpt-image-1.5`                                             | `outputFormat` değerini `png` veya `webp` olarak ve `background=transparent` değerini ayarlayın. |

## Adlandırma eşlemesi

| Gördüğünüz ad                            | Katman            | Anlamı                                                                                  |
| ---------------------------------------- | ----------------- | --------------------------------------------------------------------------------------- |
| `openai`                      | Sağlayıcı ön eki  | Standart OpenAI model rotası; örtük çalışma zamanını rota bilgileri belirler.            |
| `codex` Plugin               | Plugin            | Yerel Codex uygulama sunucusu çalışma zamanını ve `/codex` sohbet denetimlerini sağlayan paketle birlikte gelen Plugin. |
| sağlayıcı/model `agentRuntime.id: codex`       | Ajan çalışma zamanı | Eşleşen gömülü turlarda yerel Codex uygulama sunucusu çalıştırma düzeneğini zorunlu kılar. |
| `/codex ...`                      | Sohbet komut kümesi | Bir konuşmadan Codex uygulama sunucusu iş parçacıklarını bağlar/denetler.                |
| `runtime: "acp", agentId: "codex"`                      | ACP oturum rotası | Codex'i ACP/acpx üzerinden çalıştıran açık geri dönüş yolu.                             |

## Örtük ajan çalışma zamanı

Sağlayıcı/model `agentRuntime` ilkesi ayarlanmamış veya `auto` olduğunda OpenAI'ın
sağlayıcıya ait rota ilkesi, etkin uç nokta ve bağdaştırıcıya göre örtük
çalışma zamanını seçer:

| Etkin rota bilgileri                                                                                                                                                   | Örtük çalışma zamanı |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| `openai-responses` içeren tam resmî Platform HTTPS uç noktası veya `openai-chatgpt-responses` içeren tam resmî ChatGPT HTTPS uç noktası; yazılmış istek geçersiz kılması yok | Codex seçilebilir    |
| Yazılmış `openai-completions` bağdaştırıcısı                                                                                                                             | OpenClaw             |
| Özel uç nokta                                                                                                                                                          | OpenClaw             |
| HTTP kullanan açık ve tam resmî uç nokta                                                                                                                               | Reddedilir           |
| Yazılmış bir sağlayıcı/model isteği geçersiz kılması içeren rota                                                                                                       | OpenClaw             |

Açıkça belirtilen varsayılan dışı sağlayıcı/model `agentRuntime.id` ayarı belirleyici olmaya devam eder.
Örneğin `agentRuntime.id: "openclaw"`, normalde Codex'e uygun olan bir
rotayı OpenClaw'da tutarken `agentRuntime.id: "codex"`, Codex'i zorunlu kılar ve
etkin rota Codex uyumlu olarak bildirilmemişse güvenli biçimde başarısız olur.
Çalışma zamanı seçimi, kimlik bilgisi türünü veya faturalandırmayı değiştirmez: Platform API anahtarı
kimlik doğrulaması ile ChatGPT/Codex abonelik kimlik doğrulaması ayrı kalır.

`openclaw doctor --fix`; eski `codex/*` ve `openai-codex/*` model
başvurularını, eski Codex kimlik doğrulama profili kimliklerini ve eski Codex kimlik doğrulama sırası girdilerini
standart `openai` rotasına taşır. Taşınan model başvuruları model kapsamlı
`agentRuntime.id: "codex"` alır; yeni kimlik doğrulama sırası yapılandırması için `auth.order.openai` kullanın.

<Note>
Yeni OpenAI kurulumu, yalnızca birincil model yapılandırılmamışsa GPT-5.6'yı
birincil olarak uygular. OpenAI kimlik doğrulaması eklemek veya yenilemek,
`openai/gpt-5.5` dâhil olmak üzere mevcut açık seçimi korur; ancak
`models auth login --set-default` veya `models set` açıkça kullanılırsa bu geçerli değildir. Bir ajan modeli için yalnızca
API anahtarı kimlik doğrulaması istediğinizde API anahtarı kimlik doğrulama profili kullanın.
</Note>

## GPT-5.6 sınırlı önizlemesi

OpenClaw, tam `openai/gpt-5.6-sol`,
`openai/gpt-5.6-terra` ve `openai/gpt-5.6-luna` model kimliklerini tanır. Mevcut katalogda üçü de
`xhigh` ve `max` akıl yürütme sunar. OpenAI; Sol'u
amiral gemisi katmanı, Terra'yı dengeli katman, Luna'yı ise hızlı ve
daha düşük maliyetli katman olarak tanımlar. Bkz.
[GPT-5.6 lansman duyurusu](https://openai.com/index/previewing-gpt-5-6-sol/)
ve [erişim kılavuzu](https://help.openai.com/en/articles/20001325-a-preview-of-gpt-5-6-sol-terra-and-luna).

Doğrudan OpenAI API anahtarı kimlik doğrulamasıyla yalın `openai/gpt-5.6` kimliği, Sol'un diğer adıdır
ve yeni kurulumun varsayılanıdır. Yerel Codex kataloğu bu doğrudan API diğer adını
istemci tarafında uygulamaz; çalışma alanı erişimine bağlı olarak tam Sol, Terra ve Luna
kimliklerini gösterebilir. Bu nedenle yeni ChatGPT/Codex OAuth kurulumu
`openai/gpt-5.6-sol` kullanır. Mevcut hesabı şununla kontrol edin:

```bash
openclaw models list --provider openai
```

API kuruluşu ile Codex çalışma alanı erişimi farklı olabilir. GPT-5.6
kullanılamıyorsa GPT-5.5'i açıkça seçin:

```bash
openclaw models set openai/gpt-5.5
```

OpenClaw, üst sistemden gelen erişim hatasını gösterir ve bir
GPT-5.6 seçimini sessizce GPT-5.5 ile değiştirmez.

<Note>
Uygun tam resmî HTTPS rotaları, çalışma zamanı ilkesi ayarlanmamışsa veya `auto` ise
paketle birlikte gelen Codex uygulama sunucusu Plugin'ini seçebilir; yazılmış Completions rotaları,
özel uç noktalar ve istek aktarımı geçersiz kılmaları OpenClaw'da kalır. Düz metin
resmî HTTP uç noktaları reddedilir. Açık sağlayıcı/model çalışma zamanı yapılandırması
belirleyici olmaya devam eder. Açık çalışma zamanı yapılandırmasıyla ayarlanmamış eski Codex model
başvurularını, `codex-cli/*` başvurularını veya eski çalışma zamanı oturum sabitlemelerini onarmak için `openclaw doctor --fix` çalıştırın.
</Note>

## OpenClaw özellik kapsamı

| OpenAI yeteneği         | OpenClaw yüzeyi                                                                              | Durum                                                          |
| ------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| Sohbet / Yanıtlar          | `openai/<model>` model sağlayıcısı                                                               | Evet                                                             |
| Codex abonelik modelleri | OpenAI OAuth ile `openai/<model>`                                                            | Evet                                                             |
| Eski Codex model referansları   | eski Codex model referansları, `codex-cli/<model>`                                                     | doctor tarafından `openai/<model>` olarak onarılır                          |
| Codex app-server çalışma düzeneği  | Çalışma zamanı ayarlanmamış/`auto` olan Codex uyumlu HTTPS rotası veya açıkça belirtilmiş `agentRuntime.id: codex`  | Evet                                                             |
| Sunucu taraflı web araması    | Yerel OpenAI Responses aracı                                                                  | Evet, web araması etkinse ve başka bir sağlayıcı sabitlenmemişse |
| Görüntüler                    | `image_generate`                                                                              | Evet                                                             |
| Videolar                    | `video_generate`                                                                              | Evet                                                             |
| Metinden konuşmaya            | `tts.provider: "openai"` / `tts`                                                              | Evet                                                             |
| Toplu konuşmadan metne      | `tools.media.audio` / medya anlama                                                     | Evet                                                             |
| Akışlı konuşmadan metne  | Sesli Arama `streaming.provider: "openai"`                                                     | Evet                                                             |
| Gerçek zamanlı ses            | Sesli Arama `realtime.provider: "openai"` / Kontrol Arayüzü Konuşma `talk.realtime.provider: "openai"` | Evet (OpenAI Platform API anahtarı)                                   |
| Gömme vektörleri                | bellek gömme vektörü sağlayıcısı                                                                     | Evet                                                             |

<Note>
OpenAI gerçek zamanlı ses, herkese açık **OpenAI Platform Realtime
API** üzerinden çalışır ve bir Platform API anahtarı gerektirir. Codex OAuth token'ları ise
ChatGPT Codex arka ucunda kimlik doğrular; herkese açık Realtime uç noktaları için
Platform API anahtarlarıyla birbirlerinin yerine kullanılamazlar.

API anahtarıyla kimlik doğrulama faturalandırmanın eksik olduğunu bildirirse, API anahtarıyla
kimlik doğrulamayı kullanırken gerçek zamanlı kimlik bilgilerinizi destekleyen kuruluş için
[platform.openai.com/account/billing](https://platform.openai.com/account/billing)
adresinden Platform kredisi yükleyin. Gerçek zamanlı ses; `openclaw onboard --auth-choice openai-api-key` tarafından
oluşturulan `openai` API anahtarı kimlik doğrulama profilini, Kontrol Arayüzü Konuşma için
`talk.realtime.providers.openai.apiKey` aracılığıyla ayarlanan bir Platform API anahtarını,
Sesli Arama için `plugins.entries.voice-call.config.realtime.providers.openai.apiKey` değerini
veya `OPENAI_API_KEY` ortam değişkenini kabul eder.

Kontrol Arayüzü Görüntülü Konuşma'da OpenAI WebRTC, kamera bağlamını isteğe bağlı olarak alır:
model `describe_view` çağrısı yaptığında tarayıcı, gerçek zamanlı veri kanalı üzerinden
sınırlandırılmış tek bir JPEG gönderir. OpenClaw, OpenAI oturumuna kesintisiz bir kamera izi
eklemez.
</Note>

## Bellek gömme vektörleri

OpenClaw, `memory_search` indeksleme ve sorgu gömme vektörleri için
OpenAI'ı veya OpenAI uyumlu bir gömme vektörü uç noktasını kullanabilir:

```json5
{
  memory: {
    search: {
      provider: "openai",
      model: "text-embedding-3-small",
    },
  },
}
```

Asimetrik gömme vektörü etiketleri gerektiren OpenAI uyumlu uç noktalar için
`memory.search` altında `queryInputType` ve `documentInputType` değerlerini ayarlayın. OpenClaw
bunları sağlayıcıya özgü `input_type` istek alanları olarak iletir: sorgu
gömme vektörleri `queryInputType` kullanır; indekslenen bellek parçaları ve toplu indeksleme
`documentInputType` kullanır. Tam örnek için
[Bellek yapılandırması referansına](/tr/reference/memory-config#provider-specific-config)
bakın.

## Başlarken

<Tabs>
  <Tab title="API anahtarı (OpenAI Platform)">
    **En uygun kullanım:** doğrudan API erişimi ve kullanıma dayalı faturalandırma.

    <Steps>
      <Step title="API anahtarınızı alın">
        [OpenAI Platform panosundan](https://platform.openai.com/api-keys) bir API anahtarı oluşturun veya kopyalayın.
      </Step>
      <Step title="İlk kurulumu çalıştırın">
        ```bash
        openclaw onboard --auth-choice openai-api-key
        ```

        Ya da anahtarı doğrudan iletin:

        ```bash
        openclaw onboard --openai-api-key "$OPENAI_API_KEY"
        ```
      </Step>
      <Step title="Modelin kullanılabilir olduğunu doğrulayın">
        ```bash
        openclaw models list --provider openai
        ```
      </Step>
    </Steps>

    ### Rota özeti

    | Model referansı        | Çalışma zamanı ilkesi veya rota bilgileri                                 | Rota                     | Kimlik doğrulama                              |
    | ---------------- | ------------------------------------------------------------- | ------------------------- | --------------------------------- |
    | `openai/gpt-5.6` | ayarlanmamış/`auto`, tam resmî yerel HTTPS rotası, istek geçersiz kılması yok | Codex seçilebilir     | Sıralanmış API anahtarı kimlik doğrulama profili      |
    | `openai/gpt-5.6` | sağlayıcı/model `agentRuntime.id: "openclaw"`                  | OpenClaw gömülü çalışma zamanı | Seçilen `openai` API anahtarı profili |
    | `openai/gpt-5.5` | açıkça belirtilmiş sağlayıcı/model `agentRuntime.id`                     | Seçilen ajan çalışma zamanı    | Seçilen OpenAI API anahtarı profili   |
    | `openai/*`       | yazılmış Completions, özel veya istek geçersiz kılması | OpenClaw gömülü çalışma zamanı | Kimlik bilgisi türü değişmeden kalır |
    | `openai/*`       | düz metin resmî HTTP uç noktası                  | Reddedildi                 | Kimlik bilgisi gönderilmez             |

    <Note>
    Çalışma zamanı ayarlanmamışken veya `auto` olduğunda, yalnızca uygun ve tam bir resmî yerel HTTPS
    rotası Codex app-server çalışma düzeneğini örtük olarak seçebilir. Bir ajan modelinde API anahtarıyla
    kimlik doğrulama için bir `openai` API anahtarı kimlik doğrulama profili oluşturup bunu
    `auth.order.openai` ile sıralayın; `OPENAI_API_KEY`, ajan dışı OpenAI API yüzeyleri için
    doğrudan geri dönüş olarak kalır. Eski Codex kimlik doğrulama sırası girdilerini taşımak için
    `openclaw doctor --fix` komutunu çalıştırın.
    </Note>

    ### Yapılandırma örneği

    ```json5
    {
      env: { OPENAI_API_KEY: "example-openai-key-not-real" },
      agents: { defaults: { model: { primary: "openai/gpt-5.6" } } },
    }
    ```

    Sade doğrudan API `gpt-5.6` kimliği Sol katmanına çözümlenir. Bu API
    kuruluşu GPT-5.6'yı sunmuyorsa birincil modeli açıkça
    `openai/gpt-5.5` olarak ayarlayın.

    OpenAI API üzerinden ChatGPT'nin mevcut Instant modelini denemek için modeli
    `openai/chat-latest` olarak ayarlayın:

    ```json5
    {
      env: { OPENAI_API_KEY: "example-openai-key-not-real" },
      agents: { defaults: { model: { primary: "openai/chat-latest" } } },
    }
    ```

    `chat-latest` değişken bir takma addır. Yeni OpenAI API anahtarı kurulumu bunun yerine,
    sade doğrudan API kimliği Sol'a çözümlenen `openai/gpt-5.6` değerini kullanır. `openai/gpt-5.5`
    dâhil mevcut açıkça belirtilmiş birincil modeller değişmeden kalır.
    `chat-latest` takma adı yalnızca `medium` metin ayrıntı düzeyini kabul eder; OpenClaw
    bu model için istenen diğer tüm ayrıntı düzeylerini `medium` olarak zorunlu kılar.

    <Warning>
    OpenClaw, doğrudan OpenAI API anahtarı rotasında `gpt-5.3-codex-spark` değerini
    **sunmaz**. Bu değer yalnızca oturum açmış hesabınız bunu sunuyorsa Codex abonelik kataloğu
    girdileri üzerinden kullanılabilir.
    </Warning>

  </Tab>

  <Tab title="Codex aboneliği">
    **En uygun kullanım:** ayrı bir API anahtarı yerine yerel Codex
    app-server yürütmesiyle ChatGPT/Codex aboneliğinizi kullanmak. Codex bulutu,
    ChatGPT oturumu açılmasını gerektirir.

    <Steps>
      <Step title="Codex OAuth'u çalıştırın">
        ```bash
        openclaw onboard --auth-choice openai
        ```

        Ya da OAuth'u doğrudan çalıştırın:

        ```bash
        openclaw models auth login --provider openai
        ```

        Başsız veya geri çağrıya elverişsiz kurulumlarda, localhost tarayıcı
        geri çağrısı yerine ChatGPT cihaz kodu akışıyla oturum açmak için
        `--device-code` ekleyin:

        ```bash
        openclaw models auth login --provider openai --device-code
        ```
      </Step>
      <Step title="Standart OpenAI model rotasını kullanın">
        ```bash
        openclaw config set agents.defaults.model.primary openai/gpt-5.6-sol
        ```

        Bu tam resmî yerel HTTPS rotası için çalışma zamanı yapılandırması
        gerekmez. Codex app-server çalışma zamanını otomatik olarak seçebilir ve
        bu çalışma zamanı seçildiğinde OpenClaw paketlenmiş Codex Plugin'ini
        yükler veya onarır.
      </Step>
      <Step title="Codex kimlik doğrulamasının kullanılabilir olduğunu doğrulayın">
        ```bash
        openclaw models list --provider openai
        ```

        Gateway çalışmaya başladıktan sonra yerel app-server çalışma zamanını doğrulamak için
        sohbette `/codex status` veya `/codex models` gönderin.
      </Step>
    </Steps>

    ### Rota özeti

    | Model referansı                | Çalışma zamanı ilkesi veya rota bilgileri                                 | Rota                                                    | Kimlik doğrulama                                               |
    | ------------------------ | ------------------------------------------------------------- | -------------------------------------------------------- | -------------------------------------------------- |
    | `openai/gpt-5.6-sol`     | ayarlanmamış/`auto`, tam resmî yerel HTTPS rotası, istek geçersiz kılması yok | Codex seçilebilir                                    | Codex oturum açma veya sıralanmış bir `openai` kimlik doğrulama profili |
    | `openai/gpt-5.6-terra`   | ayarlanmamış/`auto`, tam resmî yerel HTTPS rotası, istek geçersiz kılması yok | Codex seçilebilir                                    | Katalog Terra'yı sunuyorsa Codex oturum açma       |
    | `openai/gpt-5.6-luna`    | ayarlanmamış/`auto`, tam resmî yerel HTTPS rotası, istek geçersiz kılması yok | Codex seçilebilir                                    | Katalog Luna'yı sunuyorsa Codex oturum açma        |
    | `openai/gpt-5.6-sol`     | sağlayıcı/model `agentRuntime.id: "openclaw"`                  | OpenClaw gömülü çalışma zamanı, dâhilî Codex kimlik doğrulama aktarımı | Seçilen `openai` OAuth profili                    |
    | `openai/gpt-5.5`         | açıkça belirtilmiş sağlayıcı/model `agentRuntime.id`                     | Seçilen ajan çalışma zamanı                                   | Seçilen OpenAI kimlik doğrulama profili                       |
    | `openai/*`               | yazılmış Completions, özel veya istek geçersiz kılması | OpenClaw gömülü çalışma zamanı                                | Kimlik bilgisi gereksinimi rotaya özgü kalır      |
    | `openai/*`               | düz metin resmî HTTP uç noktası                  | Reddedildi                                                 | Kimlik bilgisi gönderilmez                              |
    | Eski Codex GPT-5.5 referansı | doctor tarafından onarılır                                            | `openai/gpt-5.5` olarak yeniden yazılır                            | Taşınmış OpenAI OAuth profili                      |
    | `codex-cli/gpt-5.5`      | doctor tarafından onarılır                                            | `openai/gpt-5.5` olarak yeniden yazılır                            | Codex app-server kimlik doğrulaması                              |

    <Warning>
    Abonelik destekli yeni kurulum tam olarak `openai/gpt-5.6-sol` kullanır;
    yerel Codex kataloğu tam Terra veya Luna referanslarını da sunabilir. Hesap
    GPT-5.6'yı sunmuyorsa açıkça `openai/gpt-5.5` seçin. Eski
    Codex GPT referansları, yerel Codex çalışma zamanı yolu değil, eski OpenClaw
    rotalarıdır; mevcut açık GPT-5.5 seçimini yükseltmeden bunları taşımak için
    `openclaw doctor --fix` çalıştırın. `gpt-5.3-codex-spark`, yalnızca Codex abonelik
    kataloğunda sunulduğu belirtilen hesaplarla sınırlı kalır; buna yönelik doğrudan
    OpenAI API anahtarı ve Azure referansları gizli kalır.
    </Warning>

    <Note>
    Yeni yapılandırma, OpenAI ajan kimlik doğrulama sırasını `auth.order.openai`
    altında tutmalıdır; doctor eski Codex kimlik doğrulama sırası girdilerini taşır.
    </Note>

    ### Yapılandırma örneği

    ```json5
    {
      plugins: { entries: { codex: { enabled: true } } },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-sol" },
        },
      },
    }
    ```

    API anahtarı yedeğiyle, seçilen modeli `openai/*` altında ve kimlik
    doğrulama sırasını `openai` altında tutun. OpenClaw, Codex
    altyapısında kalırken önce aboneliği, ardından API anahtarını dener:

    ```json5
    {
      plugins: { entries: { codex: { enabled: true } } },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-sol" },
        },
      },
      auth: {
        order: {
          openai: [
            "openai:user@example.com",
            "openai:api-key-backup",
          ],
        },
      },
    }
    ```

    <Note>
    İlk katılım artık OAuth materyalini `~/.codex` konumundan içe aktarmaz.
    Tarayıcı OAuth'u (varsayılan) veya yukarıdaki cihaz kodu akışıyla oturum açın;
    OpenClaw, elde edilen kimlik bilgilerini kendi ajan kimlik doğrulama deposunda yönetir.
    </Note>

    ### Codex OAuth yönlendirmesini denetleme ve kurtarma

    ```bash
    openclaw models status
    openclaw models auth list --provider openai
    openclaw config get agents.defaults.model --json
    openclaw config get models.providers.openai.agentRuntime --json
    ```

    Belirli bir ajan için `--agent <id>` ekleyin:

    ```bash
    openclaw models status --agent <id>
    openclaw models auth list --agent <id> --provider openai
    ```

    Eski bir yapılandırmada hâlâ eski Codex GPT referansları veya açık çalışma
    zamanı yapılandırması olmadan geçerliliğini yitirmiş bir OpenAI çalışma zamanı
    oturum sabitlemesi varsa bunu onarın:

    ```bash
    openclaw doctor --fix
    openclaw config validate
    ```

    `models auth list --provider openai` kullanılabilir bir profil göstermiyorsa yeniden
    oturum açın:

    ```bash
    openclaw models auth login --provider openai
    openclaw models status --probe --probe-provider openai
    ```

    Aynı ajanda birden çok Codex OAuth oturum açma işlemi için `--profile-id`
    kullanın, ardından bunları kimlik doğrulama sıralaması veya `/model ...@<profileId>`
    aracılığıyla denetleyin:

    ```bash
    openclaw models auth login --provider openai --profile-id openai:ritsuko
    openclaw models auth login --provider openai --profile-id openai:lain
    ```

    Profil sıralamasına güvenmeden önce eski OpenAI Codex ön ekli profil
    kimliklerini ve sıra girdilerini taşımak için `openclaw doctor --fix` çalıştırın.

    ### Durum göstergesi

    Sohbette `/status`, geçerli oturumda hangi model çalışma zamanının
    etkin olduğunu gösterir. Paketle birlikte gelen Codex app-server altyapısı,
    uygun bir örtük rota veya açık sağlayıcı/model çalışma zamanı politikası
    tarafından seçildiğinde `Runtime: OpenAI Codex` olarak görünür.

    ### Doctor uyarısı

    Yapılandırmada veya oturum durumunda eski Codex model referansları ya da
    geçerliliğini yitirmiş OpenAI çalışma zamanı sabitlemeleri kalırsa,
    OpenClaw açıkça yapılandırılmadığı sürece `openclaw doctor --fix` bunları Codex
    çalışma zamanı ile `openai/*` olarak yeniden yazar.

    ### Bağlam penceresi varsayılanları ve uzun bağlamı etkinleştirme

    OpenClaw, yerel model kapasitesini ve etkin çalışma zamanı bütçesini ayrı
    değerler olarak ele alır:

    - `contextWindow`, sağlayıcının toplam model penceresini bildirir.
    - `contextTokens`, OpenClaw'un etkin girdi için bu pencerenin ne kadarını kullanacağını sınırlar.

    ChatGPT/Codex OAuth, canlı Codex hesap kataloğunu izler. Geçerli katalog,
    GPT-5.6 için yaygın olarak `272000` tokenlık etkin pencere sunar.
    Doğrudan API anahtarlı GPT-5.5 ve GPT-5.6 modelleri de Platform API daha
    büyük bir yerel pencere sunsa bile varsayılan olarak `272000`
    `contextTokens` değerlerini kullanır. Bu, normal gecikme, kalite ve maliyet
    profilini kimlik doğrulama modları arasında tutarlı kılar. Yapılandırılmış bir
    `agents.defaults.contextTokens` değeri bu bütçeyi daha da düşürebilir ancak bir modeli,
    yapılandırılmış `contextTokens` üst sınırının üzerine çıkaramaz.

    Doğrudan API anahtarlı GPT-5.5 ve GPT-5.6 için OpenAI, `1050000`
    tokenlık sağlayıcı penceresi ve `128000` azami çıktı tokenı belgeler.
    Tam çıktı payını ayırmak, girdi için `922000` token bırakır. Bu,
    sağlayıcı tarafından yayımlanmış ayrı bir girdi sınırı değil, türetilmiş bir
    işletim bütçesidir. Resmî [model karşılaştırmasına](https://developers.openai.com/api/docs/models/compare)
    ve [GPT-5.5 model sayfasına](https://developers.openai.com/api/docs/models/gpt-5.5)
    bakın. Aşağıdaki örnek, bir Terra modelini bu paya dahil eder ve OpenAI'dan
    `700000` etkin tokenda sıkıştırma yapmasını ister:

    ```json5
    {
      models: {
        providers: {
          openai: {
            models: [
              {
                id: "gpt-5.6-terra",
                name: "GPT-5.6 Terra",
                contextWindow: 1050000,
                contextTokens: 922000,
                maxTokens: 128000,
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-terra" },
          models: {
            "openai/gpt-5.6-terra": {
              agentRuntime: { id: "openclaw" },
              params: {
                responsesServerCompaction: true,
                responsesCompactThreshold: 700000,
              },
            },
          },
        },
      },
    }
    ```

    Bu örnekte `agentRuntime.id: "openclaw"` bilinçli olarak kullanılmıştır. Bu, gömülü
    OpenClaw Responses yolunun yukarıdaki model meta verilerini ve sunucu tarafı
    sıkıştırma ayarlarını kullandığını kanıtlar. Yerel bir Codex altyapısı iş
    parçacığı ise bağlam bütçesini Codex yapılandırmasında yönetir; bkz.
    [Codex altyapısında uzun bağlam](/tr/plugins/codex-harness#direct-api-long-context).

    <Warning>
    Bir GPT-5.5 veya GPT-5.6 isteği `272000` girdi tokenını aştığında
    OpenAI daha yüksek uzun bağlam fiyatlandırması uygular: koşulları karşılayan
    isteğin tamamı 2× girdi ve 1.5× çıktı tarifesiyle ücretlendirilir. Büyük
    istemler turlar arasında yeniden gönderilir veya sıkıştırılır; bu nedenle,
    görünür yanıt kısa olsa bile etkinleştirilmiş bir oturumun maliyeti
    varsayılandan önemli ölçüde daha yüksek olabilir. Bkz.
    [OpenAI API fiyatlandırması](https://developers.openai.com/api/docs/pricing).
    Hesap erişimi, gerçek sınırlar ve faturalandırma konusunda API yetkili kaynak
    olmaya devam eder.
    </Warning>

    ### Katalog kurtarma

    OpenClaw, mevcut olduğunda `gpt-5.5` için üst kaynak Codex katalog
    meta verilerini kullanır. Hesabın kimliği doğrulanmışken canlı Codex keşfi
    `gpt-5.5` satırını atlıyorsa OpenClaw, cron, alt ajan ve
    yapılandırılmış varsayılan model çalıştırmalarının `Unknown model` hatasıyla
    başarısız olmaması için bu OAuth model satırını oluşturur.

  </Tab>
</Tabs>

## Yerel Codex app-server kimlik doğrulaması

Yerel Codex app-server altyapısı, uygun bir tam resmî HTTPS rotası tarafından
örtük olarak seçildiğinde veya sağlayıcı/model `agentRuntime.id: "codex"` tarafından açıkça
seçildiğinde `openai/*` model referanslarını kullanır. Kimlik doğrulaması
yine hesap tabanlıdır. OpenClaw kimlik doğrulamayı şu sırayla seçer:

1. Ajan için tercihen `auth.order.openai` altında sıralanmış OpenAI kimlik
   doğrulama profilleri. Eski Codex kimlik doğrulama profili kimliklerini ve
   kimlik doğrulama sırasını taşımak için `openclaw doctor --fix` çalıştırın.
2. Yerel Codex CLI ChatGPT oturum açması gibi app-server'ın mevcut hesabı.
   Varsayılan yalıtılmış ajan ana dizini için OpenClaw, bu yerel CLI hesabını oturum
   açma RPC'si üzerinden app-server'a bağlar; CLI'ın yapılandırmasını, pluginlerini
   veya iş parçacığı deposunu paylaşmaz.
3. Yalnızca yerel stdio app-server başlatmaları için ve yalnızca app-server
   hesap olmadığını bildirdiğinde: `CODEX_API_KEY`, ardından `OPENAI_API_KEY`.

Yerel ChatGPT/Codex abonelik oturum açması, Gateway işlemi doğrudan OpenAI
modelleri veya gömmeleri için `OPENAI_API_KEY` değerine de sahip diye
değiştirilmez. Ortam API anahtarı geri dönüşü yalnızca yerel stdio hesapsız
yoluna uygulanır; WebSocket app-server bağlantıları üzerinden asla gönderilmez.
Abonelik tarzı bir Codex profili seçildiğinde OpenClaw ayrıca `CODEX_API_KEY`
ve `OPENAI_API_KEY` değerlerini başlatılan stdio app-server alt işleminden uzak
tutar ve seçilen kimlik bilgilerini bunun yerine app-server oturum açma RPC'si
üzerinden gönderir.

Bu abonelik profili bir Codex kullanım sınırı nedeniyle engellendiğinde OpenClaw,
profili Codex'in bildirdiği sıfırlama zamanına kadar engellenmiş olarak işaretler
ve seçilen modeli değiştirmeden ya da Codex altyapısından çıkmadan kimlik doğrulama
sıralamasının bir sonraki `openai:*` profiline geçmesine izin verir.
Sıfırlama zamanı geçtikten sonra abonelik profili yeniden uygun hâle gelir.

## Görsel oluşturma

Paketle gelen `openai` plugini, `image_generate` aracı üzerinden görsel
oluşturmayı kaydeder. Aynı `openai/gpt-image-2` model referansı üzerinden hem OpenAI
API anahtarıyla hem de Codex OAuth ile görsel oluşturmayı destekler.

| Yetenek                    | OpenAI API anahtarı                  | Codex OAuth                               |
| ------------------------- | ---------------------------------- | ------------------------------------ |
| Model referansı           | `openai/gpt-image-2`               | `openai/gpt-image-2`                 |
| Kimlik doğrulama          | `OPENAI_API_KEY`                   | OpenAI Codex OAuth oturum açma            |
| Aktarım                   | OpenAI Images API                  | Codex Responses arka ucu                   |
| İstek başına azami görsel | 4                                  | 4                                         |
| Düzenleme modu            | Etkin (en fazla 5 referans görsel) | Etkin (en fazla 5 referans görsel)        |
| Boyut geçersiz kılmaları  | 2K/4K boyutlar dâhil desteklenir   | 2K/4K boyutlar dâhil desteklenir          |
| En-boy oranı / çözünürlük | OpenAI Images API'ye iletilmez     | Güvenli olduğunda desteklenen boyuta eşlenir |

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "openai/gpt-image-2" },
    },
  },
}
```

<Note>
Paylaşılan araç parametreleri, sağlayıcı seçimi ve yük devretme davranışı için
bkz. [Görsel Oluşturma](/tr/tools/image-generation).
</Note>

`gpt-image-2`, OpenAI metinden görsele oluşturma ve görsel düzenleme için
varsayılandır. `gpt-image-1.5`, `gpt-image-1` ve `gpt-image-1-mini` açık
model geçersiz kılmaları olarak kullanılabilir durumda kalır. Şeffaf arka planlı
PNG/WebP çıktısı için `openai/gpt-image-1.5` kullanın; geçerli `gpt-image-2` API,
`background: "transparent"` değerini reddeder.

Şeffaf arka plan isteği için `image_generate` öğesini `model: "openai/gpt-image-1.5"`,
`outputFormat: "png"` veya `"webp"` ve `background: "transparent"` ile çağırın;
eski `openai.background` sağlayıcı seçeneği hâlâ kabul edilir. OpenClaw ayrıca
varsayılan `openai/gpt-image-2` şeffaf isteklerini `gpt-image-1.5` olarak yeniden
yazarak herkese açık OpenAI ve OpenAI Codex OAuth rotalarını korur; Azure ve özel
OpenAI uyumlu uç noktalar, yapılandırılmış dağıtım/model adlarını korur.

Aynı ayar, başsız CLI çalıştırmaları için de sunulur:

```bash
openclaw infer image generate \
  --model openai/gpt-image-1.5 \
  --output-format png \
  --background transparent \
  --prompt "Şeffaf bir arka plan üzerinde basit bir kırmızı daire çıkartması" \
  --json
```

Bir girdi dosyasından başlarken `openclaw infer image edit` ile aynı `--output-format`
ve `--background` bayraklarını kullanın. `--openai-background`, OpenAI'ya özgü
bir diğer ad olarak kullanılabilir durumda kalır. OpenAI Images kalitesini ve
maliyetini denetlemek için `--quality low|medium|high|auto` kullanın.
OpenAI'ın moderasyon ipucunu `image generate` veya `image edit` üzerinden
iletmek için `--openai-moderation low|auto` kullanın.

ChatGPT/Codex OAuth kurulumları için aynı `openai/gpt-image-2` referansını koruyun. Bir
`openai` OAuth profili yapılandırıldığında OpenClaw, depolanan OAuth
erişim belirtecini çözümler ve görüntü isteklerini Codex Responses arka ucu üzerinden gönderir;
önce `OPENAI_API_KEY` yöntemini denemez veya sessizce bir API anahtarına geri dönmez.
Bunun yerine doğrudan OpenAI Images API yolunu kullanmak istediğinizde
`models.providers.openai` değerini bir API anahtarı, özel temel
URL veya Azure uç noktasıyla açıkça yapılandırın. Bu özel görüntü uç noktası güvenilen bir LAN/özel adresteyse
`browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: true` değerini de ayarlayın; OpenClaw,
bu açık katılım olmadığı sürece özel/dahili OpenAI uyumlu görüntü uç noktalarını engellemeyi
sürdürür.

Oluşturma:

```
/tool image_generate model=openai/gpt-image-2 prompt="macOS'te OpenClaw için özenle hazırlanmış bir lansman posteri" size=3840x2160 count=1
```

Şeffaf bir PNG oluşturma:

```
/tool image_generate model=openai/gpt-image-1.5 prompt="Şeffaf bir arka plan üzerinde basit bir kırmızı daire çıkartması" outputFormat=png background=transparent
```

Düzenleme:

```
/tool image_generate model=openai/gpt-image-2 prompt="Nesnenin şeklini koru, malzemeyi yarı saydam cama dönüştür" image=/path/to/reference.png size=1024x1536
```

## Video oluşturma

Paketle gelen `openai` Plugin'i, video oluşturmayı
`video_generate` aracı üzerinden kaydeder.

| Yetenek          | Değer                                                                              |
| ---------------- | ---------------------------------------------------------------------------------- |
| Varsayılan model | `openai/sora-2`                                                                    |
| Modlar            | Metinden videoya, görüntüden videoya, tek video düzenleme                          |
| Referans girdileri | 1 görüntü veya 1 video                                                            |
| Boyut geçersiz kılmaları | Metinden videoya ve görüntüden videoya için desteklenir                    |
| En-boy oranı      | Ham olarak iletilmez, desteklenen en yakın boyuta dönüştürülür                     |
| Diğer geçersiz kılmalar | `resolution`, `audio`, `watermark` desteklenmez ve bir araç uyarısıyla kaldırılır |

OpenAI görüntüden videoya istekleri, bir görüntü
`input_reference` ile `POST /v1/videos` kullanır. Tek video düzenlemeleri, yüklenen video
`video` alanında olacak şekilde `POST /v1/videos/edits` kullanır.

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "openai/sora-2" },
    },
  },
}
```

<Note>
Paylaşılan araç parametreleri, sağlayıcı seçimi ve yük devretme davranışı için
[Video Oluşturma](/tr/tools/video-generation) bölümüne bakın.

OpenAI sağlayıcısı `supportsSize` bildirir ancak `supportsAspectRatio` veya
`supportsResolution` bildirmez. OpenClaw'ın paylaşılan normalleştirme katmanı, istenen
bir `aspectRatio` değerini, istek sağlayıcıya ulaşmadan önce en yakın eşleşen OpenAI
`size` değerine dönüştürür; bu nedenle en-boy oranı istekleri genellikle çalışmaya devam eder.
`resolution` için boyut geri dönüşü yoktur ve bu değer kaldırılarak çağırana
`Ignored unsupported overrides for openai/<model>: resolution=<value>` olarak gösterilir.
</Note>

## GPT-5 istem katkısı

OpenClaw, `openai` sağlayıcısındaki GPT-5 ailesi modelleri için
paylaşılan bir GPT-5 istem katkısı ekler (normalleştirilerek
`openai/*` olan onarım öncesi eski Codex referansları dâhil). OpenRouter veya opencode yolları gibi
GPT-5 ailesi model kimliklerini de sunan diğer sağlayıcılar bu katmanı almaz; bu katman
yalnızca model kimliğine göre değil, `openai` sağlayıcı kimliğine göre sınırlandırılır. Eski GPT-4.x modelleri
bunu hiçbir zaman almaz.

Yerel Codex uygulama sunucusu donanımı, kişilik/araç
disiplini davranış sözleşmesini veya samimi etkileşim tarzı katmanını
geliştirici talimatları üzerinden almaz; yerel Codex, Codex'e ait temel, model ve
proje belgesi davranışını korur ve OpenClaw, ajan çalışma alanı kişilik dosyalarının yetkili kalması için
yerel iş parçacıklarında Codex'in yerleşik kişiliğini devre dışı bırakır.
OpenClaw, yerel Codex iş parçacıklarına yalnızca çalışma zamanı bağlamı sağlar: kanal
teslimatı, OpenClaw dinamik araçları, ACP devri, çalışma alanı bağlamı ve
OpenClaw Skills. Aynı katkıdaki Heartbeat rehberlik metni
tek istisnadır: yerel Codex Heartbeat dönüşleri bunu alır; metin, paylaşılan istem katkısı
kancası üzerinden değil, özel iş birliği talimatları olarak eklenir.

GPT-5 katkısı, eşleşen OpenClaw tarafından birleştirilmiş istemlere kişilik
kalıcılığı, yürütme güvenliği, araç disiplini, çıktı biçimi, tamamlanma
kontrolleri ve doğrulama için etiketli bir davranış sözleşmesi ekler. Kanala
özgü yanıt ve sessiz mesaj davranışı, paylaşılan OpenClaw sistem
isteminde ve giden teslimat politikasında kalır. Samimi etkileşim tarzı katmanı
ayrıdır ve yapılandırılabilir.

| Değer                  | Etki                                      |
| ---------------------- | ------------------------------------------- |
| `"friendly"` (varsayılan) | Samimi etkileşim tarzı katmanını etkinleştir |
| `"on"`                 | `"friendly"` için diğer ad                      |
| `"off"`                | Yalnızca samimi tarz katmanını devre dışı bırak       |

<Tabs>
  <Tab title="Yapılandırma">
    ```json5
    {
      agents: {
        defaults: {
          promptOverlays: {
            gpt5: { personality: "friendly" },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="CLI">
    ```bash
    openclaw config set agents.defaults.promptOverlays.gpt5.personality off
    ```
  </Tab>
</Tabs>

<Tip>
Değerler çalışma zamanında büyük/küçük harfe duyarlı değildir; dolayısıyla `"Off"` ve `"off"`
değerlerinin ikisi de samimi tarz katmanını devre dışı bırakır.
</Tip>

<Note>
Paylaşılan `agents.defaults.promptOverlays.gpt5.personality` ayarı belirlenmemişse eski
`plugins.entries.openai.config.personality` değeri uyumluluk geri dönüşü olarak
okunmaya devam eder.
</Note>

## Ses ve konuşma

<AccordionGroup>
  <Accordion title="Konuşma sentezi (TTS)">
    Paketle gelen `openai` Plugin'i,
    `tts` yüzeyi için konuşma sentezini kaydeder.

    | Ayar          | Yapılandırma yolu                                      | Varsayılan                          |
    | ------------- | --------------------------------------------------------- | ----------------------------------- |
    | Model         | `tts.providers.openai.model`                  | `gpt-4o-mini-tts`                |
    | Ses           | `tts.providers.openai.speakerVoice`           | `coral`                          |
    | Hız           | `tts.providers.openai.speed`                  | (ayarlanmamış)                          |
    | Talimatlar    | `tts.providers.openai.instructions`           | (ayarlanmamış, yalnızca `gpt-4o-mini-tts`)  |
    | Biçim         | `tts.providers.openai.responseFormat`         | sesli notlar için `opus`, dosyalar için `mp3` |
    | API anahtarı  | `tts.providers.openai.apiKey`                 | `OPENAI_API_KEY` değerine geri döner   |
    | Temel URL     | `tts.providers.openai.baseUrl`                | `https://api.openai.com/v1`      |
    | Ek gövde      | `tts.providers.openai.extraBody` / `extra_body` | (ayarlanmamış)                        |

    Kullanılabilir modeller: `gpt-4o-mini-tts`, `tts-1`, `tts-1-hd`. Kullanılabilir sesler:
    `alloy`, `ash`, `ballad`, `cedar`, `coral`, `echo`, `fable`, `juniper`,
    `marin`, `onyx`, `nova`, `sage`, `shimmer`, `verse`.

    `extraBody`, OpenClaw tarafından oluşturulan alanlardan sonra
    `/audio/speech` istek JSON'uyla birleştirilir; dolayısıyla `lang` gibi
    ek anahtarlar gerektiren OpenAI uyumlu uç noktalar için bunu kullanın. Prototip anahtarları yok sayılır.

    ```json5
    {
      tts: {
        providers: {
          openai: { model: "gpt-4o-mini-tts", speakerVoice: "coral" },
        },
      },
    }
    ```

    <Note>
    Sohbet API uç noktasını etkilemeden TTS temel URL'sini geçersiz kılmak için
    `OPENAI_TTS_BASE_URL` değerini ayarlayın. OpenAI TTS ve Realtime sesin ikisi de
    bir OpenAI Platform API anahtarı üzerinden yapılandırılır; yalnızca OAuth kullanan kurulumlar
    Codex destekli sohbet modellerini kullanmaya devam edebilir ancak OpenAI canlı sesli yanıt özelliğini kullanamaz.
    </Note>

  </Accordion>

  <Accordion title="Konuşmayı metne dönüştürme">
    Paketle gelen `openai` Plugin'i, OpenClaw'ın medya anlama
    transkripsiyon yüzeyi üzerinden toplu konuşmayı metne dönüştürme özelliğini kaydeder.

    - Varsayılan model: `gpt-4o-transcribe`
    - Uç nokta: OpenAI REST `/v1/audio/transcriptions`
    - Girdi yolu: çok parçalı ses dosyası yükleme
    - Discord ses kanalı bölümleri ve kanal ses ekleri dâhil olmak üzere,
      gelen ses transkripsiyonunun `tools.media.audio` okuduğu her yerde kullanılır

    Gelen ses transkripsiyonu için OpenAI kullanımını zorunlu kılmak üzere:

    ```json5
    {
      tools: {
        media: {
          audio: {
            models: [
              {
                type: "provider",
                provider: "openai",
                model: "gpt-4o-transcribe",
              },
            ],
          },
        },
      },
    }
    ```

    Paylaşılan ses medyası yapılandırması veya çağrı başına transkripsiyon isteği tarafından sağlandığında,
    dil ve istem ipuçları OpenAI'a iletilir.

  </Accordion>

  <Accordion title="Gerçek zamanlı transkripsiyon">
    Paketle gelen `openai` Plugin'i, Voice Call Plugin'i için
    gerçek zamanlı transkripsiyonu kaydeder.

    | Ayar             | Yapılandırma yolu                                                       | Varsayılan |
    | ----------------- | ----------------------------------------------------------------------- | --------- |
    | Model             | `plugins.entries.voice-call.config.streaming.providers.openai.model` | `gpt-4o-transcribe` |
    | Dil               | `...openai.language`                                                 | (ayarlanmamış) |
    | İstem             | `...openai.prompt`                                                   | (ayarlanmamış) |
    | Sessizlik süresi  | `...openai.silenceDurationMs`                                        | `800`   |
    | VAD eşiği         | `...openai.vadThreshold`                                             | `0.5`   |
    | Kimlik doğrulama  | `...openai.apiKey`, `OPENAI_API_KEY` veya `openai` API anahtarı profili    | Platform API anahtarı gereklidir |

    <Note>
    G.711 u-law (`g711_ulaw` / `audio/pcmu`) ses biçimiyle
    `wss://api.openai.com/v1/realtime` adresine bir WebSocket bağlantısı kullanır. Bir `openai`
    API anahtarı profili için Gateway, WebSocket'i açmadan önce geçici bir Realtime transkripsiyon istemci
    gizli anahtarı oluşturur. Bu akış sağlayıcısı, Voice Call'un gerçek zamanlı transkripsiyon yolu içindir;
    Discord ses özelliği şu anda kısa bölümler kaydeder ve bunun yerine toplu
    `tools.media.audio` transkripsiyon yolunu kullanır.
    </Note>

  </Accordion>

  <Accordion title="Gerçek zamanlı ses">
    Paketle gelen `openai` Plugin'i, Voice Call
    Plugin'i için gerçek zamanlı sesi kaydeder.

    | Ayar                                   | Yapılandırma yolu                                                          | Varsayılan                    |
    | --------------------------------------- | ---------------------------------------------------------------------------- | ------------------------------ |
    | Model                                   | `plugins.entries.voice-call.config.realtime.providers.openai.model`     | `gpt-realtime-2.1`  |
    | Ses                                     | `...openai.voice`                                                       | `alloy`             |
    | Sıcaklık (Azure dağıtım köprüsü)        | `...openai.temperature`                                                 | `0.8`               |
    | VAD eşiği                               | `...openai.vadThreshold`                                                | `0.5`                |
    | Sessizlik süresi                        | `...openai.silenceDurationMs`                                           | `500`                |
    | Önek dolgusu                            | `...openai.prefixPaddingMs`                                             | `300`                |
    | Akıl yürütme eforu                      | `...openai.reasoningEffort`                                             | (ayarlanmamış)              |
    | Kimlik doğrulama                        | `openai` API anahtarı profili, `...openai.apiKey` veya `OPENAI_API_KEY` | OpenAI Platform API anahtarı gerekli |

    `gpt-realtime-2.1` için kullanılabilen yerleşik Realtime sesleri: `alloy`, `ash`,
    `ballad`, `coral`, `echo`, `sage`, `shimmer`, `verse`, `marin`, `cedar`.
    OpenAI, en iyi Realtime kalitesi için `marin` ve `cedar` seslerini önerir. Bu,
    yukarıdaki metinden konuşmaya seslerinden ayrı bir kümedir; `fable`,
    `nova` veya `onyx` gibi yalnızca TTS'ye yönelik bir ses, Realtime oturumları için geçerli değildir.
    Daha küçük ve daha düşük maliyetli Realtime 2.1 varyantını tercih ettiğinizde
    modeli açıkça `gpt-realtime-2.1-mini` olarak ayarlayın.

    <Note>
    **GPT-Live (yakında).** OpenAI'ın tam çift yönlü `gpt-live-1` ve
    `gpt-live-1-mini` modelleri, Temmuz 2026'da ChatGPT sesli modunun yerini aldı;
    geliştirici API'si erken erişim kuruluşlarına aşamalı olarak sunuluyor. OpenClaw
    model ailesini tanır ancak henüz çalıştırmaz: GPT-Live oturumları yalnızca
    WebRTC kullanır, konuşma sıralarını kendileri yönetir (VAD yoktur) ve ajan çalışmalarını
    OpenClaw'ın gerçek zamanlı taşıma katmanlarının henüz uygulamadığı
    bir devir olayı protokolü üzerinden aktarır. Bir `gpt-live-*` modeli yapılandırıldığında,
    ajan erişimi olmadan sesi sessizce bağlamak yerine hem WebSocket köprüsü
    hem de Talk tarayıcı oturumları hakkında yönlendirmeyle kapalı biçimde başarısız olur.
    Erken erişim sırasında API erişimi de OpenAI kuruluşu başına
    kısıtlanır. GPT-Live desteği gelene kadar `gpt-realtime-2.1` seçeneğini
    (varsayılan) kullanmaya devam edin.
    </Note>

    <Note>
    Arka uç OpenAI gerçek zamanlı köprüleri, `session.temperature` kabul etmeyen
    genel kullanıma sunulmuş Realtime WebSocket oturum biçimini kullanır. Azure OpenAI
    dağıtımları `azureEndpoint` ve `azureDeployment` üzerinden kullanılabilir olmaya devam eder ve
    dağıtımla uyumlu oturum biçimini (`temperature` dâhil) korur.
    Çift yönlü araç çağrısını ve G.711 u-law sesini destekler.
    </Note>

    <Note>
    Realtime sesi, oturum oluşturulurken seçilir. OpenAI, çoğu oturum
    alanının daha sonra değiştirilmesine izin verir ancak model o oturumda ses
    ürettikten sonra ses değiştirilemez. OpenClaw şu anda yerleşik
    Realtime ses kimliklerini dize olarak sunar.
    </Note>

    <Note>
    Control UI Talk, Gateway tarafından oluşturulan geçici istemci sırrı ve
    OpenAI Realtime API ile doğrudan tarayıcı WebRTC SDP alışverişi aracılığıyla
    OpenAI tarayıcı gerçek zamanlı oturumlarını kullanır. Gateway bu istemci sırrını
    seçili `openai` kimlik bilgisiyle oluşturur. Yapılandırılmış anahtarlar, API anahtarı profilleri ve
    `OPENAI_API_KEY` önceliklidir; `openai` OAuth profili veya harici
    Codex oturum açma işlemi yedek seçenektir. Gateway aktarıcısı ve Voice Call arka uç gerçek zamanlı
    WebSocket köprüleri, yerel OpenAI uç noktaları için aynı kimlik bilgisi sırasını kullanır.
    Bakım sorumlularının canlı doğrulaması
    `OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts` ile kullanılabilir;
    OpenAI aşamaları, sırları günlüğe kaydetmeden hem arka uç WebSocket köprüsünü
    hem de tarayıcı WebRTC SDP alışverişini doğrular.
    Bu iki aşamayı Google kimlik bilgileri olmadan çalıştırmak için `--openai-only` iletin.
    </Note>

  </Accordion>
</AccordionGroup>

## Azure OpenAI uç noktaları

Paketle gelen `openai` sağlayıcısı, temel URL'yi geçersiz kılarak görüntü
oluşturma için bir Azure OpenAI kaynağını hedefleyebilir. Görüntü oluşturma yolunda OpenClaw,
`models.providers.openai.baseUrl` üzerindeki Azure ana bilgisayar adlarını algılar ve
otomatik olarak Azure'un istek biçimine geçer.

<Note>
Realtime sesi ayrı bir yapılandırma yolu
(`plugins.entries.voice-call.config.realtime.providers.openai.azureEndpoint`)
kullanır ve `models.providers.openai.baseUrl` bundan etkilenmez. Azure
ayarları için [Ses ve konuşma](#voice-and-speech) altındaki **Realtime
sesi** akordeonuna bakın.
</Note>

Şu durumlarda Azure OpenAI kullanın:

- Zaten bir Azure OpenAI aboneliğiniz, kotanız veya kurumsal
  sözleşmeniz varsa
- Azure'un sağladığı bölgesel veri yerleşimi veya uyumluluk denetimlerine ihtiyacınız varsa
- Trafiği mevcut bir Azure kiracısı içinde tutmak istiyorsanız

### Yapılandırma

Paketle gelen `openai` sağlayıcısı üzerinden Azure görüntü oluşturmak için
`models.providers.openai.baseUrl` değerini Azure kaynağınıza yönlendirin ve `apiKey` değerini
Azure OpenAI anahtarı (OpenAI Platform anahtarı değil) olarak ayarlayın:

```json5
{
  models: {
    providers: {
      openai: {
        baseUrl: "https://<your-resource>.openai.azure.com",
        apiKey: "<azure-openai-api-key>",
      },
    },
  },
}
```

OpenClaw, Azure görüntü oluşturma rotası için şu Azure ana bilgisayar son eklerini
tanır:

- `*.openai.azure.com`
- `*.services.ai.azure.com`
- `*.cognitiveservices.azure.com`

Tanınan bir Azure ana bilgisayarındaki görüntü oluşturma istekleri için OpenClaw:

- `Authorization: Bearer` yerine `api-key` üst bilgisini gönderir
- Dağıtım kapsamlı yolları (`/openai/deployments/{deployment}/...`) kullanır
- Her isteğe `?api-version=...` ekler
- Azure görüntü oluşturma çağrıları için varsayılan 600s istek zaman aşımını kullanır.
  Çağrı başına `timeoutMs` değerleri yine de bu varsayılanı geçersiz kılar.

Diğer temel URL'ler (genel OpenAI, OpenAI uyumlu proxy'ler) standart
OpenAI görüntü isteği biçimini korur.

<Note>
`openai` sağlayıcısının görüntü oluşturma yolu için Azure yönlendirmesi
OpenClaw 2026.4.22 veya sonraki bir sürümü gerektirir. Önceki sürümler, herhangi bir özel
`openai.baseUrl` değerini genel OpenAI uç noktası gibi işler ve Azure görüntü
dağıtımlarında başarısız olur.
</Note>

### API sürümü

Azure görüntü oluşturma yolu için belirli bir Azure önizleme veya genel kullanıma sunulmuş sürümü
sabitlemek üzere `AZURE_OPENAI_API_VERSION` ayarlayın:

```bash
export AZURE_OPENAI_API_VERSION="2024-12-01-preview"
```

Değişken ayarlanmadığında varsayılan değer `2024-12-01-preview` olur.

### Model adları dağıtım adlarıdır

Azure OpenAI, modelleri dağıtımlara bağlar. Paketle gelen `openai` sağlayıcısı
üzerinden yönlendirilen Azure görüntü oluşturma isteklerinde, OpenClaw'daki `model` alanı
genel OpenAI model kimliği değil, Azure portalında yapılandırdığınız **Azure dağıtım adı**
olmalıdır.

`gpt-image-2` sunan `gpt-image-2-prod` adlı bir dağıtım oluşturursanız:

```
/tool image_generate model=openai/gpt-image-2-prod prompt="Temiz bir poster" size=1024x1024 count=1
```

Aynı dağıtım adı kuralı, paketle gelen `openai` sağlayıcısı üzerinden yönlendirilen
tüm görüntü oluşturma çağrıları için geçerlidir.

### Bölgesel kullanılabilirlik

Azure görüntü oluşturma şu anda yalnızca belirli bölgelerin bir alt kümesinde
kullanılabilir (örneğin `eastus2`, `swedencentral`, `polandcentral`, `westus3`,
`uaenorth`). Bir dağıtım oluşturmadan önce Microsoft'un güncel bölge listesini
kontrol edin ve ilgili modelin bölgenizde sunulduğunu doğrulayın.

### Parametre farklılıkları

Azure OpenAI ve genel OpenAI her zaman aynı görüntü parametrelerini kabul etmez.
Azure, genel OpenAI'ın izin verdiği seçenekleri (örneğin `gpt-image-2` üzerindeki belirli
`background` değerlerini) reddedebilir veya bunları yalnızca belirli model
sürümlerinde sunabilir. Bu farklılıklar OpenClaw'dan değil, Azure'dan ve temel
modelden kaynaklanır. Bir Azure isteği doğrulama hatasıyla başarısız olursa,
belirli dağıtımınız ve API sürümünüz tarafından desteklenen parametre kümesini
Azure portalında kontrol edin.

<Note>
Azure OpenAI, yerel taşıma ve uyumluluk davranışını kullanır ancak
OpenClaw'ın gizli ilişkilendirme üst bilgilerini almaz — [Gelişmiş yapılandırma](#advanced-configuration)
altındaki **Yerel ve OpenAI uyumlu rotalar** akordeonuna bakın.

Azure'da sohbet veya Responses trafiği (görüntü oluşturmanın ötesinde) için
ilk katılım akışını ya da özel bir Azure sağlayıcı yapılandırmasını kullanın; yalnızca `openai.baseUrl`
Azure API/kimlik doğrulama biçimini devralmaz. Ayrı bir
`azure-openai-responses/*` sağlayıcısı vardır; aşağıdaki Sunucu tarafı Compaction
akordeonuna bakın.
</Note>

## Gelişmiş yapılandırma

Aşağıdaki model başına `params` örnekleri, OpenClaw'ın gömülü sağlayıcı
isteğini biçimlendirir. Bunları yapılandırmak, açıkça tanımlanmış istek davranışıdır; bu nedenle normalde uygun olan
bir `auto` rotası, Codex'i örtük olarak seçmek yerine OpenClaw'da kalır. Yerel
Codex uygulama sunucusu test sistemi kendi taşımasını ve istek ayarlarını yönetir; etkin rota
Codex uyumlu olarak bildirilmemişse açık `agentRuntime.id: "codex"` kapalı biçimde başarısız olur.

<AccordionGroup>
  <Accordion title="Taşıma (WebSocket veya SSE)">
    OpenClaw, `openai/*` için SSE yedeğiyle (`"auto"`) önce WebSocket'i kullanır.

    `"auto"` modunda OpenClaw:
    - SSE'ye geçmeden önce erken bir WebSocket hatasını bir kez yeniden dener
    - Bir hatadan sonra WebSocket'i 60 saniye boyunca bozulmuş olarak işaretler ve
      bekleme süresince SSE kullanır
    - Yeniden denemeler ve yeniden bağlantılar için kararlı oturum ve konuşma sırası
      kimliği üst bilgileri ekler
    - Taşıma varyantları genelinde kullanım sayaçlarını (`input_tokens` / `prompt_tokens`)
      normalleştirir

    | Değer                          | Davranış                                |
    | ------------------------------ | --------------------------------------- |
    | `"auto"` (varsayılan) | Önce WebSocket, SSE yedeği              |
    | `"sse"`              | Yalnızca SSE'yi zorla                   |
    | `"websocket"`              | Yalnızca WebSocket'i zorla              |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: { transport: "auto" },
            },
          },
        },
      },
    }
    ```

    İlgili OpenAI belgeleri:
    - [WebSocket ile Realtime API](https://platform.openai.com/docs/guides/realtime-websocket)
    - [API yanıtlarını akışla iletme (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

  </Accordion>

  <Accordion title="Hızlı mod">
    OpenClaw, `openai/*` için ortak bir hızlı mod anahtarı sunar:

    - **Sohbet/UI:** `/fast status|auto|on|off`
    - **Yapılandırma:** `agents.defaults.models["<provider>/<model>"].params.fastMode`

    Etkinleştirildiğinde OpenClaw, hızlı modu OpenAI öncelikli işlemeye
    (`service_tier = "priority"`) eşler. Mevcut `service_tier` değerleri
    korunur ve hızlı mod `reasoning` veya
    `text.verbosity` değerlerini yeniden yazmaz. `fastMode: "auto"`, otomatik
    kesme noktasına kadar yeni model çağrılarını hızlı başlatır; ardından sonraki yeniden deneme,
    yedek, araç sonucu veya devam çağrılarını hızlı mod olmadan başlatır.
    Kesme noktası varsayılan olarak 60 saniyedir; değiştirmek için etkin modelde
    `params.fastAutoOnSeconds` ayarlayın.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": { params: { fastMode: "auto", fastAutoOnSeconds: 30 } },
          },
        },
      },
    }
    ```

    <Note>
    Oturum geçersiz kılmaları yapılandırmaya üstün gelir. Sessions UI'da oturum
    geçersiz kılmasını temizlemek, oturumu yapılandırılmış varsayılana döndürür.
    </Note>

  </Accordion>

  <Accordion title="Öncelikli işleme (service_tier)">
    OpenAI API'si, `service_tier` aracılığıyla öncelikli işleme sunar. Bunu OpenClaw'da
    her model için ayarlayın:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": { params: { serviceTier: "priority" } },
          },
        },
      },
    }
    ```

    Desteklenen değerler: `auto`, `default`, `flex`, `priority`.

    <Warning>
    `serviceTier` yalnızca yerel OpenAI uç noktalarına
    (`api.openai.com`) ve yerel Codex uç noktalarına (`chatgpt.com/backend-api`) iletilir.
    Herhangi bir sağlayıcıyı bir proxy üzerinden yönlendirirseniz OpenClaw,
    `service_tier` değerini değiştirmeden bırakır.
    </Warning>

  </Accordion>

  <Accordion title="Sunucu tarafı Compaction (Responses API)">
    Doğrudan OpenAI Responses modelleri için (`api.openai.com` üzerinde `openai/*`),
    OpenAI Plugin'inin OpenClaw akış sarmalayıcısı sunucu tarafı Compaction'ı
    otomatik olarak etkinleştirir:

    - `store: true` değerini zorunlu kılar (model uyumluluğu `supportsStore: false` ayarlamadığı sürece)
    - `context_management: [{ type: "compaction", compact_threshold: ... }]` ekler
    - Varsayılan `compact_threshold`: `contextWindow` değerinin %70'i (veya
      kullanılamadığında `80000`)

    Bu, yerleşik OpenClaw çalışma zamanı yoluna ve gömülü çalıştırmaların
    kullandığı OpenAI sağlayıcı kancalarına uygulanır. Yerel Codex uygulama
    sunucusu çalıştırma sistemi, kendi bağlamını Codex üzerinden yönetir ve bu
    ayardan etkilenmez.

    <Tabs>
      <Tab title="Açıkça etkinleştirme">
        Azure OpenAI Responses gibi uyumlu uç noktalar için kullanışlıdır:

        ```json5
        {
          agents: {
            defaults: {
              models: {
                "azure-openai-responses/gpt-5.5": {
                  params: { responsesServerCompaction: true },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="Özel eşik">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.5": {
                  params: {
                    responsesServerCompaction: true,
                    responsesCompactThreshold: 120000,
                  },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="Devre dışı bırakma">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.5": {
                  params: { responsesServerCompaction: false },
                },
              },
            },
          },
        }
        ```
      </Tab>
    </Tabs>

    <Note>
    `responsesServerCompaction` yalnızca `context_management` eklenmesini denetler.
    Doğrudan OpenAI Responses modelleri, uyumluluk `supportsStore: false` ayarlamadığı
    sürece yine de `store: true` değerini zorunlu kılar.
    </Note>

  </Accordion>

  <Accordion title="Katı aracılı GPT modu">
    OpenClaw'ın gömülü çalışma zamanı üzerinden çalıştırılan `openai` sağlayıcısının
    GPT-5 ailesi modelleri için OpenClaw, `strict-agentic` adlı daha katı bir
    yürütme sözleşmesini zaten varsayılan olarak kullanır. Yapılandırma açıkça
    devre dışı bırakmadığı sürece çözümlenen sağlayıcı `openai` olduğunda
    ve model kimliği GPT-5 ailesiyle eşleştiğinde otomatik olarak etkinleşir:

    ```json5
    {
      agents: {
        defaults: {
          embeddedAgent: { executionContract: "default" },
        },
      },
    }
    ```

    `"strict-agentic"` değerini açıkça ayarlamak, desteklenen bir kanalda etkisizdir
    (zaten varsayılandır) ve desteklenmeyen sağlayıcı/model çiftlerinde işlevsizdir.

    `strict-agentic` etkinken OpenClaw:
    - Kapsamlı işler için `update_plan` özelliğini otomatik olarak etkinleştirir
    - Yapısal olarak boş veya yalnızca akıl yürütme içeren turları, görünür yanıt
      devamıyla yeniden dener
    - Seçilen çalıştırma sistemi sağladığında açık çalıştırma sistemi plan
      olaylarını kullanır

    OpenClaw, bir turun plan, ilerleme güncellemesi veya nihai yanıt olup
    olmadığına karar vermek için asistan metnini sınıflandırmaz.

    <Note>
    Bu sözleşme tamamen OpenClaw'ın gömülü ajan çalıştırıcısında yer alır. Kendi
    tur ve plan davranışını yöneten yerel Codex uygulama sunucusu çalıştırma
    sistemine uygulanmaz; yerel Codex çalıştırmalarında çalıştırma sistemi
    seçimi, yürütme sözleşmesi ayarından daha önemlidir.
    </Note>

  </Accordion>

  <Accordion title="Yerel ve OpenAI uyumlu yollar">
    OpenClaw, doğrudan OpenAI, Codex ve Azure OpenAI uç noktalarını genel
    OpenAI uyumlu `/v1` proxy'lerinden farklı biçimde ele alır:

    **Yerel yollar** (`openai/*`, Azure OpenAI):
    - `reasoning: { effort: "none" }` değerini yalnızca OpenAI `none`
      çabasını destekleyen modeller için korur
    - `reasoning.effort: "none"` değerini reddeden modeller veya proxy'ler için
      devre dışı akıl yürütmeyi dahil etmez
    - Araç şemalarını varsayılan olarak katı moda ayarlar
    - Gizli ilişkilendirme başlıklarını yalnızca doğrulanmış yerel ana makinelere ekler
      (Azure OpenAI yerel bir yol olmasına rağmen bu başlıkları almaz)
    - Yalnızca OpenAI'a özgü istek biçimlendirmesini korur (`service_tier`, `store`,
      akıl yürütme uyumluluğu, istem önbelleği ipuçları)

    **Proxy/uyumlu yollar:**
    - Daha esnek uyumluluk davranışı kullanır
    - Yerel olmayan `openai-completions` yüklerinden Completions `store` alanını çıkarır
    - OpenAI uyumlu Completions proxy'leri için gelişmiş `params.extra_body`/`params.extraBody`
      aktarım JSON'unu kabul eder
    - vLLM gibi OpenAI uyumlu Completions proxy'leri için
      `params.chat_template_kwargs` değerini kabul eder
    - Katı araç şemalarını veya yalnızca yerel yollara özgü başlıkları zorunlu kılmaz

  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Görüntü oluşturma" href="/tr/tools/image-generation" icon="image">
    Paylaşılan görüntü aracı parametreleri ve sağlayıcı seçimi.
  </Card>
  <Card title="Video oluşturma" href="/tr/tools/video-generation" icon="video">
    Paylaşılan video aracı parametreleri ve sağlayıcı seçimi.
  </Card>
  <Card title="OAuth ve kimlik doğrulama" href="/tr/gateway/authentication" icon="key">
    Kimlik doğrulama ayrıntıları ve kimlik bilgilerini yeniden kullanma kuralları.
  </Card>
</CardGroup>
