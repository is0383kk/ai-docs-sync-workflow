---
read_when:
    - OpenClaw'u Ollama aracılığıyla bulut veya yerel modellerle çalıştırmak istiyorsunuz
    - Ollama kurulumu ve yapılandırması için rehberliğe ihtiyacınız var
    - Görüntüleri anlamak için Ollama görsel modellerini kullanmak istiyorsunuz
summary: OpenClaw'u Ollama ile çalıştırma (bulut ve yerel modeller)
title: Ollama
x-i18n:
    generated_at: "2026-07-26T23:33:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80ae833d006ce307406fac11fe3457809165035a38b7e0a970777baf126cc9cb
    source_path: providers/ollama.md
    workflow: 16
---

OpenClaw, OpenAI uyumlu
`/v1` uç noktasıyla değil, Ollama'nın yerel API'siyle (`/api/chat`) iletişim kurar. Üç mod desteklenir:

| Mod           | Kullandığı kaynak                                                                  |
| ------------- | ---------------------------------------------------------------------------------- |
| Bulut + Yerel | Yerel modelleri ve (oturum açılmışsa) `:cloud` modellerini sunan, erişilebilir bir Ollama ana makinesi |
| Yalnızca bulut | Doğrudan `https://ollama.com`; yerel daemon yok                                    |
| Yalnızca yerel | Erişilebilir bir Ollama ana makinesi; yalnızca yerel modeller                    |

Özel `ollama-cloud` sağlayıcı kimliğiyle yalnızca bulut kurulumu için
[Ollama Cloud](/tr/providers/ollama-cloud) sayfasına bakın. Bulut yönlendirmesini
yerel bir `ollama` sağlayıcısından ayrı tutmak istediğinizde
`ollama-cloud/<model>` referanslarını kullanın.

<Warning>
`/v1` OpenAI uyumlu URL'sini (`http://host:11434/v1`) kullanmayın. Bu, araç çağrısını bozar ve modeller ham araç çağrısı JSON'unu düz metin olarak üretebilir. Yerel URL'yi kullanın: `baseUrl: "http://host:11434"` (`/v1` olmadan).
</Warning>

Standart yapılandırma anahtarı `baseUrl` şeklindedir. OpenAI SDK tarzı
örnekler için `baseURL` da kabul edilir ancak yeni yapılandırmalarda
`baseUrl` kullanılmalıdır.

## Kimlik doğrulama kuralları

<AccordionGroup>
  <Accordion title="Yerel ve LAN ana makineleri">
    Geri döngü, özel ağ, `.local` ve yalın ana makine adlı Ollama URL'leri gerçek bir bearer token gerektirmez. OpenClaw bunlar için `ollama-local` işaretçisini kullanır.
  </Accordion>
  <Accordion title="Uzak ve Ollama Cloud ana makineleri">
    Herkese açık uzak ana makineler ve `https://ollama.com` gerçek bir kimlik bilgisi gerektirir: `OLLAMA_API_KEY`, bir kimlik doğrulama profili veya sağlayıcının `apiKey` değeri. Doğrudan barındırılan kullanım için `ollama-cloud` sağlayıcısını tercih edin.
  </Accordion>
  <Accordion title="Özel sağlayıcı kimlikleri">
    `api: "ollama"` kullanan özel bir sağlayıcı aynı kurallara uyar. Örneğin, özel bir LAN ana makinesine yönlendirilen `ollama-remote` sağlayıcısı `apiKey: "ollama-local"` kullanabilir; alt ajanlar bu işaretçiyi eksik bir kimlik bilgisi olarak değerlendirmek yerine Ollama sağlayıcı kancası üzerinden çözümler. `memory.search.provider` ayrıca özel bir sağlayıcı kimliğine yönlendirilebilir; böylece gömmeler söz konusu Ollama uç noktasını kullanır.
  </Accordion>
  <Accordion title="Kimlik doğrulama profilleri">
    `auth-profiles.json`, bir sağlayıcı kimliğinin kimlik bilgisini saklar; uç nokta ayarlarını (`baseUrl`, `api`, modeller, üstbilgiler, zaman aşımları) `models.providers.<id>` içine koyun. `{ "ollama-windows": { "apiKey": "ollama-local" } }` gibi eski düz dosyalar çalışma zamanı biçimi değildir; `openclaw doctor --fix` bunları bir yedekle birlikte standart bir `ollama-windows:default` API anahtarı profiline dönüştürür. Bu eski dosyadaki bir `baseUrl` değeri gereksizdir ve sağlayıcı yapılandırmasına taşınmalıdır.
  </Accordion>
  <Accordion title="Bellek gömme kapsamı">
    Ollama bellek gömmeleri için bearer kimlik doğrulamasının kapsamı, bildirildiği ana makineyle sınırlıdır:

    - Sağlayıcı düzeyindeki bir anahtar yalnızca söz konusu sağlayıcının ana makinesine gönderilir.
    - `memory.search.remote.apiKey` ve ajan başına geçersiz kılmalar yalnızca kendi uzak gömme ana makinelerine gönderilir.
    - Yalnızca `OLLAMA_API_KEY` ortam değişkeni değeri, Ollama Cloud kuralı olarak değerlendirilir ve varsayılan olarak yerel/kendi kendine barındırılan ana makinelere gönderilmez.

  </Accordion>
</AccordionGroup>

## Başlarken

<Tabs>
  <Tab title="İlk kurulum (önerilen)">
    <Steps>
      <Step title="İlk kurulumu çalıştırın">
        ```bash
        openclaw onboard
        ```

        **Ollama** seçeneğini, ardından bir mod seçin: **Bulut + Yerel**, **Yalnızca bulut** veya **Yalnızca yerel**.

        Yeni bir yönlendirmeli kurulumda OpenClaw önce varsayılan veya yapılandırılmış
        Ollama ana makinesini denetler. Yüklü bir model yalnızca
        `/api/show` araç desteğini ve en az 16K bağlam penceresini
        doğruladığında otomatik olarak sunulur; eksik veya daha küçük bağlam
        meta verileri, manuel kurulum yolunda kalır. Paylaşılan CLI/macOS kurulum
        sıralaması, seçilen rotayı kaydetmeden önce gerçek bir tamamlama ile
        doğrulamaya devam eder. Bu otomatik denetim hiçbir zaman model indirmez;
        uygun bir yüklü model yoksa ilk kurulum normal Ollama seçicisiyle devam eder.
      </Step>
      <Step title="Bir model seçin">
        `Cloud only`, `OLLAMA_API_KEY` için istem gösterir ve barındırılan bulut varsayılanlarını önerir. `Cloud + Local` ve `Local only`, bir Ollama temel URL'si için istem gösterir, kullanılabilir modelleri keşfeder ve seçilen yerel model eksikse otomatik olarak indirir. `gemma4:latest` gibi yüklü bir `:latest` etiketi, `gemma4` değerini yinelemek yerine bir kez gösterilir. `Cloud + Local` ayrıca ana makinede bulut erişimi için oturum açılıp açılmadığını denetler.
      </Step>
      <Step title="Doğrulayın">
        ```bash
        openclaw models list --provider ollama
        ```
      </Step>
    </Steps>

    Etkileşimsiz:

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --custom-base-url "http://ollama-host:11434" \
      --custom-model-id "qwen3.5:27b" \
      --accept-risk
    ```

    `--custom-base-url` ve `--custom-model-id` isteğe bağlıdır; bunların belirtilmemesi yerel varsayılan ana makineyi ve önerilen `gemma4` modelini kullanır.

  </Tab>

  <Tab title="Manuel kurulum">
    <Steps>
      <Step title="Ollama'yı yükleyin ve başlatın">
        [ollama.com/download](https://ollama.com/download) adresinden edinin, ardından bir model indirin:

        ```bash
        ollama pull gemma4
        ```

        Hibrit bulut erişimi için aynı ana makinede `ollama signin` çalıştırın.
      </Step>
      <Step title="Bir kimlik bilgisi ayarlayın">
        ```bash
        export OLLAMA_API_KEY="ollama-local"    # yerel/LAN ana makinesi, herhangi bir değer çalışır
        export OLLAMA_API_KEY="your-real-key"   # yalnızca https://ollama.com
        ```

        Ya da yapılandırmada: `openclaw config set models.providers.ollama.apiKey "OLLAMA_API_KEY"`.
      </Step>
      <Step title="Modeli seçin">
        ```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        Ya da yapılandırmada:

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "ollama/gemma4" },
            },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Yerel bir ana makine üzerinden bulut modelleri

`Cloud + Local`, hem yerel hem de `:cloud` modellerini erişilebilir tek
bir Ollama ana makinesi üzerinden yönlendirir. Bu, Ollama'nın hibrit akışıdır ve
her ikisini de istediğinizde kurulum sırasında seçmeniz gereken moddur.

OpenClaw temel URL için istem gösterir, yerel modelleri keşfeder ve
`ollama signin` durumunu denetler. Oturum açılmışsa barındırılan varsayılanları
(`kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`, `glm-5.2:cloud`)
önerir. Oturum açılmamışsa `ollama signin` çalıştırılana kadar kurulum yalnızca
yerel modda kalır.

Yerel bir daemon olmadan yalnızca bulut erişimi için `openclaw onboard --auth-choice ollama-cloud` kullanın
ve [Ollama Cloud](/tr/providers/ollama-cloud) sayfasına bakın. Bu yol
`ollama signin` veya çalışan bir sunucu gerektirmez:

```bash
openclaw onboard --auth-choice ollama-cloud
openclaw models set ollama-cloud/kimi-k2.5:cloud
```

`openclaw onboard` sırasında gösterilen bulut modeli listesi,
`https://ollama.com/api/tags` kaynağından canlı olarak doldurulur ve 500 girdiyle
sınırlandırılır; böylece seçici güncel barındırılan kataloğu yansıtır.
`ollama.com` erişilemez durumdaysa veya kurulum sırasında hiç model
döndürmezse OpenClaw, ilk kurulumun tamamlanabilmesi için sabit kodlanmış
öneri listesine geri döner.

## Model keşfi (örtük sağlayıcı)

`OLLAMA_API_KEY` (veya bir kimlik doğrulama profili) ayarlandığında ve ne
`models.providers.ollama` ne de `api: "ollama"` kullanan başka bir özel sağlayıcı
tanımlandığında OpenClaw, modelleri `http://127.0.0.1:11434` üzerinden keşfeder:

| Davranış             | Ayrıntı                                                                                                                                                                                                                                                                                       |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Katalog sorgusu      | `/api/tags`                                                                                                                                                                                                                                                                            |
| Yetenek algılama     | En iyi çabayla yapılan `/api/show`; `contextWindow`, `num_ctx` Modelfile parametrelerini ve yetenekleri (görsel/araçlar/düşünme) okur                                                                                                                                       |
| Görsel modeller      | `/api/show` kaynağındaki bir `vision` yeteneği, modeli görüntü destekli (`input: ["text", "image"]`) olarak işaretler                                                                                                                                                                  |
| Akıl yürütme algılama | Varsa `/api/show` kaynağındaki `thinking` yeteneğini kullanır; Ollama yetenekleri belirtmediğinde ad sezgisine (`r1`, `reason`, `reasoning`, `think`) geri döner. `glm-5.2:cloud` ve `deepseek-v4-flash\|pro:cloud`, bildirilen yeteneklerden bağımsız olarak her zaman akıl yürütme modeli sayılır. |
| Token sınırları      | `maxTokens`, varsayılan olarak OpenClaw'ın Ollama azami token sınırını kullanır                                                                                                                                                                                                         |
| Maliyetler           | Tüm maliyetler `0` değerindedir                                                                                                                                                                                                                                                |

```bash
ollama list
openclaw models list
```

Açık bir `models` dizisiyle `models.providers.ollama` ayarlamak veya
`api: "ollama"` kullanan ve geri döngü olmayan bir `baseUrl`
değerine sahip özel sağlayıcı tanımlamak otomatik keşfi devre dışı bırakır;
bu durumda modeller manuel olarak tanımlanmalıdır
([Yapılandırma](#configuration) bölümüne bakın). Barındırılan
`https://ollama.com` adresine yönlendirilen bir `models.providers.ollama` girdisi de
keşfi atlar çünkü Ollama Cloud modelleri sağlayıcı tarafından yönetilir.
`http://127.0.0.2:11434` gibi geri döngü kullanan özel sağlayıcılar yerel sayılmaya
devam eder ve otomatik keşfi korur.

Elle yazılmış bir `models.json` girdisi olmadan
`ollama/<pulled-model>:latest` gibi tam bir referans kullanabilirsiniz; OpenClaw bunu canlı
olarak çözümler. Oturum açılmış ana makinelerde, listelenmeyen bir
`ollama/<model>:cloud` referansının seçilmesi, söz konusu modeli
`/api/show` ile doğrular ve yalnızca Ollama meta verileri doğrularsa
çalışma zamanı kataloğuna ekler; yazım hataları yine bilinmeyen model hatası verir.

### Duman testleri

Tam ajan araç yüzeyini atlayan dar kapsamlı bir metin denemesi için:

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/llama3.2:latest \
    --prompt "Tam olarak şu yanıtı ver: pong" \
    --json
```

Hafif bir görsel modeli denemesi için bir görüntüyle birlikte
`--file` ekleyin (PNG/JPEG/WebP kabul edilir; görüntü olmayan dosyalar
Ollama çağrılmadan önce reddedilir; ses için `openclaw infer audio transcribe` kullanın):

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/qwen2.5vl:7b \
    --prompt "Bu görüntüyü tek cümleyle açıklayın." \
    --file ./photo.jpg \
    --json
```

Her iki yol da sohbet araçlarını, belleği veya oturum bağlamını yüklemez. Bunlar
başarılı olurken normal ajan yanıtları başarısız oluyorsa sorun büyük olasılıkla
uç noktada değil, modelin araç/ajan kapasitesindedir.

`/model ollama/<model>` ile bir model seçmek kesin bir kullanıcı tercihidir: yapılandırılmış
`baseUrl` erişilemez durumdaysa sonraki yanıt, yapılandırılmış başka bir modele
sessizce geri dönmek yerine sağlayıcı hatasıyla başarısız olur.

Yalıtılmış cron işleri, agent turunu başlatmadan önce bir yerel güvenlik denetimi ekler:
seçilen model yerel/özel ağ/`.local` Ollama
sağlayıcısına çözümleniyorsa ve `/api/tags` erişilemez durumdaysa OpenClaw, hata
metninde modelle birlikte bu çalıştırmayı `skipped` olarak kaydeder. Bu uç nokta denetimi
ana bilgisayar başına 5 dakika önbelleğe alınır; böylece durdurulmuş bir arka plan programına yönelik tekrarlanan cron işleri
başarısız olacak isteklerin tümünü başlatmaz.

Canlı doğrulama:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 \
  pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

Ollama Cloud için aynı canlı testi barındırılan uç noktaya yöneltin (varsayılan olarak
gömmeleri atlar; bir bulut anahtarı `/api/embed` için yetki vermeyebileceğinden
`OPENCLAW_LIVE_OLLAMA_EMBEDDINGS=1` ile zorlayın):

```bash
export OLLAMA_API_KEY='<your-ollama-cloud-api-key>'
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 \
OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com \
OPENCLAW_LIVE_OLLAMA_MODEL=glm-5.1:cloud \
OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=1 \
pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

Bir model eklemek için modeli çekin; otomatik olarak keşfedilir:

```bash
ollama pull mistral
```

## Node üzerinde yerel çıkarım

Agent'lar kısa bir görevi eşleştirilmiş bir masaüstü veya
sunucu Node'u üzerindeki Ollama modeline devredebilir. İstem ve yanıt, mevcut kimliği doğrulanmış
Gateway/Node bağlantısından geçer; istek Node'un kendi geri döngü Ollama
uç noktasında (`http://127.0.0.1:11434`) çalışır.

<Steps>
  <Step title="Node üzerinde Ollama'yı başlatın">
    ```bash
    ollama pull qwen3:0.6b
    ollama list
    ```
  </Step>
  <Step title="Node ana bilgisayarını bağlayın">
    ```bash
    openclaw node run \
      --host <gateway-host> \
      --port 18789 \
      --display-name "Local inference"
    ```

    Gateway ana bilgisayarında cihazı ve Node komutlarını onaylayın, ardından doğrulayın:

    ```bash
    openclaw devices list
    openclaw devices approve <deviceRequestId>
    openclaw nodes pending
    openclaw nodes approve <nodeRequestId>
    openclaw nodes status --connected
    ```

    İlk bağlantı veya Ollama komutları ekleyen bir yükseltme,
    Node komutu onayını tetikleyebilir. Node, `ollama.models` ve
    `ollama.chat` özelliklerini duyurmadan bağlanırsa `openclaw nodes pending` değerini yeniden denetleyin.

  </Step>
  <Step title="Bir agent üzerinden kullanın">
    Birlikte gelen Ollama Plugin'i `node_inference` aracını sunar. Agent'lar
    önce `action: "discover"`, ardından bu sonuçtaki bir Node ve modelle
    `action: "run"` çağrısını yapar (tam olarak bir yetenekli Node
    bağlıysa `run` Node'u atlayabilir). Örneğin: "Node'larımdaki Ollama modellerini keşfet,
    ardından bu metni özetlemek için yüklenmiş en hızlı modeli kullan."
  </Step>
</Steps>

Keşif, `/api/tags` değerini okur, `/api/show` yeteneklerini denetler ve
mevcut olduğunda zaten yüklenmiş modelleri önce sıralamak için `/api/ps` kullanır. Yalnızca
Ollama'nın sohbet yeteneğine sahip (`completion` yeteneği) olarak bildirdiği yerel modelleri döndürür —
Ollama Cloud satırları ve yalnızca gömme modelleri hariç tutulur. Her çalıştırma
model düşünmesini devre dışı bırakır ve araç çağrısı farklı bir `maxTokens`
istemediği sürece çıktıyı varsayılan olarak 512 token ile sınırlar (kesin üst sınır 8192); bazı modeller
(örneğin GPT-OSS) düşünmeyi devre dışı bırakmayı desteklemez ve yine de akıl yürütme token'ları üretebilir.

Ollama'yı agent'lara açmadan bir Node üzerinde çalışır durumda tutmak için:

```bash
openclaw config set plugins.entries.ollama.config.nodeInference.enabled false
```

Node'u yeniden başlatın (`openclaw node restart` veya ön plandaki bir oturum için
`openclaw node run` işlemini durdurup yeniden çalıştırın). Node, `ollama.models` ve
`ollama.chat` özelliklerini duyurmayı durdurur; Ollama'nın kendisi ve Gateway'in Ollama sağlayıcısı bundan etkilenmez.
Değeri yeniden `true` olarak ayarlayıp etkinleştirmek için yeniden başlatın; değişmiş bir komut
yüzeyi, yeniden bağlandıktan sonra `openclaw nodes pending` onayını tekrar gerektirebilir.

Node komutlarını bir agent turu olmadan doğrudan doğrulayın:

```bash
openclaw nodes invoke \
  --node "Local inference" \
  --command ollama.models \
  --params '{}' \
  --invoke-timeout 90000 \
  --timeout 100000

openclaw nodes invoke \
  --node "Local inference" \
  --command ollama.chat \
  --params '{"model":"qwen3:0.6b","prompt":"Reply with exactly: pong","maxTokens":32,"timeoutMs":120000}' \
  --invoke-timeout 130000 \
  --timeout 140000
```

`--invoke-timeout`, Node'un komutu çalıştırmak için sahip olduğu süreyi sınırlar;
`--timeout`, genel Gateway çağrısını sınırlar ve daha büyük olmalıdır.

Node üzerinde yerel çıkarım her zaman Node'un kendi geri döngü uç noktasını kullanır —
yapılandırılmış uzak/bulut `models.providers.ollama.baseUrl` değerini yeniden kullanmaz. Node
komutları macOS, Linux ve Windows Node ana bilgisayarlarında varsayılan olarak kullanılabilir
ve normal Node eşleştirme/komut politikasına tabi olmaya devam eder.

## Görüntü ve resim açıklaması

Birlikte gelen Ollama Plugin'i, Ollama'yı görüntü yetenekli
bir medya anlama sağlayıcısı olarak kaydeder; böylece OpenClaw açık resim açıklama
isteklerini ve yapılandırılmış görüntü modeli varsayılanlarını yerel veya barındırılan Ollama
görüntü modelleri üzerinden yönlendirebilir.

```bash
ollama pull qwen2.5vl:7b
export OLLAMA_API_KEY="ollama-local"
openclaw infer image describe --file ./photo.jpg --model ollama/qwen2.5vl:7b --json
```

`--model` tam bir `<provider/model>` başvurusu olmalıdır; ayarlandığında `infer image
describe`, zaten yerel görüntü desteğine sahip modeller için
açıklamayı atlamak yerine önce bu modeli dener. Çağrı başarısız olursa OpenClaw,
`agents.defaults.imageModel.fallbacks` üzerinden devam edebilir; dosya/URL hazırlama hataları
geri dönüş denenmeden önce başarısız olur. OpenClaw'ın görüntü anlama akışı ve
yapılandırılmış `imageModel` için `infer image describe`; özel bir istemle ham çok modlu
bir yoklama için `infer model run
--file` kullanın.

Ollama'yı gelen medyalar için varsayılan görüntü anlama sağlayıcısı yapmak üzere:

```json5
{
  agents: {
    defaults: {
      imageModel: {
        primary: "ollama/qwen2.5vl:7b",
      },
    },
  },
}
```

Tam `ollama/<model>` başvurusunu tercih edin. `qwen2.5vl:7b` gibi yalın bir
`imageModel` başvurusu, yalnızca tam olarak bu model `models.providers.ollama.models` altında
`input: ["text", "image"]` ile listelenmişse ve yapılandırılmış başka hiçbir görüntü sağlayıcısı
aynı yalın kimliği sunmuyorsa `ollama/qwen2.5vl:7b` olarak normalleştirilir; aksi takdirde sağlayıcı ön ekini açıkça kullanın.

Yavaş yerel görüntü modelleri, bulut modellerinden daha uzun bir görüntü anlama zaman aşımına
ihtiyaç duyabilir ve Ollama modelin duyurulan tam görüntü bağlamını
ayırmaya çalışırsa kısıtlı donanımlarda çökebilir. Bir yetenek
zaman aşımı ayarlayın ve `num_ctx` değerini sınırlayın:

```json5
{
  models: {
    providers: {
      ollama: {
        models: [
          {
            id: "qwen2.5vl:7b",
            name: "qwen2.5vl:7b",
            input: ["text", "image"],
            params: { num_ctx: 2048, keep_alive: "1m" },
          },
        ],
      },
    },
  },
  tools: {
    media: {
      image: {
        timeoutSeconds: 180,
        models: [{ provider: "ollama", model: "qwen2.5vl:7b", timeoutSeconds: 300 }],
      },
    },
  },
}
```

Bu zaman aşımı, gelen görüntülerin anlaşılmasına ve açık
`image` aracına uygulanır. `models.providers.ollama.timeoutSeconds`, normal model çağrıları için
temeldeki Ollama HTTP istek korumasını denetlemeye devam eder.

Canlı doğrulama:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA_IMAGE=1 \
  pnpm test:live -- src/agents/tools/image-tool.ollama.live.test.ts
```

`models.providers.ollama.models` değerini elle tanımlarsanız görüntü modellerini
açıkça işaretleyin:

```json5
{
  id: "qwen2.5vl:7b",
  name: "qwen2.5vl:7b",
  input: ["text", "image"],
  contextWindow: 128000,
  maxTokens: 8192,
}
```

OpenClaw, görüntü yetenekli olarak işaretlenmemiş modeller için resim açıklama
isteklerini reddeder. Örtük keşifte bu, `/api/show` görüntü
yeteneğinden gelir.

## Yapılandırma

<Tabs>
  <Tab title="Temel (örtük keşif)">
    ```bash
    export OLLAMA_API_KEY="ollama-local"
    ```

    <Tip>
    `OLLAMA_API_KEY` ayarlanmışsa sağlayıcı girdisinde `apiKey` değerini atlayabilirsiniz; OpenClaw kullanılabilirlik denetimleri için bunu doldurur.
    </Tip>

  </Tab>

  <Tab title="Açık (elle tanımlanan modeller)">
    Barındırılan bulut kurulumu, varsayılan olmayan bir ana bilgisayar/port, zorunlu
    bağlam pencereleri veya tamamen elle tanımlanan model listeleri için açık yapılandırma kullanın:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "https://ollama.com",
            apiKey: "OLLAMA_API_KEY",
            api: "ollama",
            models: [
              {
                id: "kimi-k2.5:cloud",
                name: "kimi-k2.5:cloud",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 128000,
                maxTokens: 8192
              }
            ]
          }
        }
      }
    }
    ```

  </Tab>

  <Tab title="Özel temel URL">
    Açık yapılandırma otomatik keşfi devre dışı bırakır; bu nedenle modeller listelenmelidir:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            apiKey: "ollama-local",
            baseUrl: "http://ollama-host:11434", // /v1 yok - yerel Ollama API URL'si
            api: "ollama", // Açık: yerel araç çağırma davranışını garanti eder
            timeoutSeconds: 300, // İsteğe bağlı: soğuk yerel modeller için daha uzun bağlantı/akış bütçesi
            models: [
              {
                id: "qwen3:32b",
                name: "qwen3:32b",
                params: {
                  keep_alive: "15m", // İsteğe bağlı: modeli turlar arasında yüklü tutar
                },
              },
            ],
          },
        },
      },
    }
    ```

    <Warning>
    `/v1` eklemeyin. Bu yol, araç çağırmanın güvenilir olmadığı OpenAI uyumlu modu seçer.
    </Warning>

  </Tab>
</Tabs>

## Yaygın tarifler

Model kimliklerini `ollama list` veya
`openclaw models list --provider ollama` kaynaklarındaki tam adlarla değiştirin.

<AccordionGroup>
  <Accordion title="Otomatik keşifli yerel model">
    Gateway ile aynı makinedeki Ollama otomatik olarak keşfedilir:

    ```bash
    ollama serve
    ollama pull gemma4
    export OLLAMA_API_KEY="ollama-local"
    openclaw models list --provider ollama
    openclaw models set ollama/gemma4
    ```

    Elle tanımlanan modellere ihtiyacınız yoksa bir `models.providers.ollama` bloğu eklemeyin.

  </Accordion>

  <Accordion title="Elle tanımlanan modellere sahip LAN Ollama ana bilgisayarı">
    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 300,
            contextWindow: 32768,
            maxTokens: 8192,
            models: [
              {
                id: "qwen3.5:9b",
                name: "qwen3.5:9b",
                reasoning: true,
                input: ["text"],
                params: {
                  num_ctx: 32768,
                  thinking: false,
                  keep_alive: "15m",
                },
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "ollama/qwen3.5:9b" },
        },
      },
    }
    ```

    `contextWindow`, OpenClaw'ın bağlam bütçesidir; `params.num_ctx`,
    Ollama'ya gönderilir. Donanım modelin duyurulan tam bağlamını çalıştıramadığında
    bunları uyumlu tutun.

  </Accordion>

  <Accordion title="Yalnızca Ollama Cloud">
    Yerel arka plan programı olmadan doğrudan barındırılan modeller:

    ```bash
    export OLLAMA_API_KEY="your-ollama-api-key"
    ```

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "https://ollama.com",
            apiKey: "OLLAMA_API_KEY",
            api: "ollama",
            models: [
              {
                id: "kimi-k2.5:cloud",
                name: "kimi-k2.5:cloud",
                reasoning: false,
                input: ["text", "image"],
                contextWindow: 128000,
                maxTokens: 8192,
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "ollama/kimi-k2.5:cloud" },
        },
      },
    }
    ```

    Bu yapı yerine özel `ollama-cloud` sağlayıcı kimliği için
    [Ollama Cloud](/tr/providers/ollama-cloud) bölümüne bakın.

  </Accordion>

  <Accordion title="Oturum açılmış bir daemon üzerinden bulut ve yerel">
    ```bash
    ollama signin
    ollama pull gemma4
    ```

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://127.0.0.1:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 300,
            models: [
              { id: "gemma4", name: "gemma4", input: ["text"] },
              { id: "kimi-k2.5:cloud", name: "kimi-k2.5:cloud", input: ["text", "image"] },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "ollama/gemma4",
            fallbacks: ["ollama/kimi-k2.5:cloud"],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Birden fazla Ollama sunucusu">
    Birden fazla Ollama sunucusu çalıştırırken özel sağlayıcı kimlikleri kullanılır; her biri
    kendi sunucusuna, modellerine, kimlik doğrulamasına ve zaman aşımına sahip olur.

    ```json5
    {
      models: {
        providers: {
          "ollama-fast": {
            baseUrl: "http://mini.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            contextWindow: 32768,
            models: [{ id: "gemma4", name: "gemma4", input: ["text"] }],
          },
          "ollama-large": {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 420,
            contextWindow: 131072,
            maxTokens: 16384,
            models: [{ id: "qwen3.5:27b", name: "qwen3.5:27b", input: ["text"] }],
          },
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "ollama-fast/gemma4",
            fallbacks: ["ollama-large/qwen3.5:27b"],
          },
        },
      },
    }
    ```

    OpenClaw, Ollama'yı çağırmadan önce etkin sağlayıcı önekini kaldırır (yalın bir
    `ollama/` önekine geri döner); böylece `ollama-large/qwen3.5:27b`,
    Ollama'ya `qwen3.5:27b` olarak ulaşır.

  </Accordion>

  <Accordion title="Hafif yerel model profili">
    Bazı yerel modeller basit istemleri işleyebilir ancak eksiksiz ajan
    araç yüzeyinde zorlanır. Genel çalışma zamanı ayarlarına dokunmadan önce
    araçları ve bağlamı sınırlayın:

    ```json5
    {
      agents: {
        list: [
          {
            id: "local",
            experimental: {
              localModelLean: true,
            },
            model: { primary: "ollama/gemma4" },
          },
        ],
      },
      models: {
        providers: {
          ollama: {
            baseUrl: "http://127.0.0.1:11434",
            apiKey: "ollama-local",
            api: "ollama",
            contextWindow: 32768,
            models: [
              {
                id: "gemma4",
                name: "gemma4",
                input: ["text"],
                params: { num_ctx: 32768 },
                compat: { supportsTools: false },
              },
            ],
          },
        },
      },
    }
    ```

    `compat.supportsTools: false` seçeneğini yalnızca model veya sunucu araç şemalarında
    güvenilir biçimde başarısız olduğunda kullanın; bu seçenek kararlılık karşılığında ajan yeteneğini azaltır.
    `localModelLean`, açıkça gerekmedikçe ağır tarayıcı, cron, mesaj, medya oluşturma,
    ses ve PDF araçlarını doğrudan ajan yüzeyinden kaldırır ve daha büyük katalogları
    Araç Arama'nın arkasına yerleştirir. Ollama'nın çalışma zamanı bağlamını veya düşünme modunu
    değiştirmez. Döngüye giren ya da bütçesini gizli akıl yürütmeye
    harcayan küçük Qwen tarzı düşünme modelleri için bunu `params.num_ctx` ve
    `params.thinking: false` ile birlikte kullanın.

  </Accordion>
</AccordionGroup>

### Model seçimi

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

Özel sağlayıcı kimlikleri de aynı şekilde çalışır: `ollama-spark/qwen3:32b` gibi etkin sağlayıcı
önekini kullanan bir referans için OpenClaw, Ollama'yı çağırmadan önce bu öneki kaldırır
ve `qwen3:32b` gönderir.

Yavaş yerel modellerde tüm ajan çalışma zamanı zaman aşımını artırmadan önce
sağlayıcı kapsamlı ayarlamayı tercih edin:

```json5
{
  models: {
    providers: {
      ollama: {
        timeoutSeconds: 300,
        models: [
          {
            id: "gemma4:26b",
            name: "gemma4:26b",
            params: { keep_alive: "15m" },
          },
        ],
      },
    },
  },
}
```

`timeoutSeconds`, model HTTP isteğinin bağlantı kurulumu, üstbilgiler,
gövde akışı ve korumalı getirmenin toplam iptali dâhil tamamını kapsar. `params.keep_alive`,
yerel `/api/chat` isteklerinde üst düzey `keep_alive` olarak iletilir; ilk turdaki
yükleme süresi darboğaz olduğunda bunu model başına ayarlayın.

### Hızlı doğrulama

```bash
# Ollama daemon'u bu makine tarafından görülebiliyor
curl http://127.0.0.1:11434/api/tags

# OpenClaw kataloğu ve seçilen model
openclaw models list --provider ollama
openclaw models status

# Doğrudan model duman testi
openclaw infer model run \
  --model ollama/gemma4 \
  --prompt "Tam olarak şununla yanıt ver: ok"
```

Uzak sunucular için `127.0.0.1` değerini `baseUrl` sunucusuyla değiştirin. `curl`
çalışıyor ancak OpenClaw çalışmıyorsa Gateway'in farklı bir makine, konteyner veya
hizmet hesabında çalışıp çalışmadığını kontrol edin.

## Ollama Web Arama

OpenClaw, **Ollama Web Arama** özelliğini bir `web_search` sağlayıcısı olarak paketler.

| Özellik     | Ayrıntı                                                                                                                                                    |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Sunucu      | Ayarlandığında `models.providers.ollama.baseUrl`, aksi takdirde `http://127.0.0.1:11434`; `https://ollama.com` barındırılan API'yi doğrudan kullanır                               |
| Kimlik doğrulama | Oturum açılmış yerel bir sunucu için anahtarsız; doğrudan `https://ollama.com` araması veya kimlik doğrulama korumalı sunucular için `OLLAMA_API_KEY` ya da yapılandırılmış sağlayıcı kimlik doğrulaması |
| Gereksinim  | Yerel/kendi barındırılan sunucular çalışıyor ve `ollama signin` ile oturum açılmış olmalıdır; doğrudan barındırılan arama için `baseUrl: "https://ollama.com"` ile gerçek bir API anahtarı gerekir |

Bunu `openclaw onboard` veya `openclaw configure --section web` sırasında seçin ya da şunu ayarlayın:

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

Ollama Cloud üzerinden doğrudan barındırılan arama için:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "https://ollama.com",
        apiKey: "OLLAMA_API_KEY",
        api: "ollama",
        models: [{ id: "kimi-k2.5:cloud", name: "kimi-k2.5:cloud", input: ["text"] }],
      },
    },
  },
  tools: {
    web: {
      search: { provider: "ollama" },
    },
  },
}
```

Kendi barındırdığınız bir sunucuda OpenClaw önce yerel `/api/experimental/web_search`
proxy'sini dener, ardından aynı sunucudaki barındırılan `/api/web_search` yoluna geri döner;
oturum açılmış yerel bir daemon normalde yerel proxy üzerinden yanıt verir. Doğrudan
`https://ollama.com` çağrıları her zaman barındırılan `/api/web_search` uç noktasını kullanır.

<Note>
Eksiksiz kurulum ve davranış için [Ollama Web Arama](/tr/tools/ollama-search) bölümüne bakın.
</Note>

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Eski OpenAI uyumlu mod">
    <Warning>
    **Bu modda araç çağırma güvenilir değildir.** Yalnızca bir proxy OpenAI biçimi gerektirdiğinde ve yerel araç çağırmaya bağımlı olmadığınızda kullanın.
    </Warning>

    `/v1/chat/completions` arkasındaki bir proxy için `api: "openai-completions"` değerini
    açıkça ayarlayın:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: true, // varsayılan: true
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

    Bu mod, akış ve araç çağırmayı aynı anda desteklemeyebilir; modelde
    `params: { streaming: false }` kullanmanız gerekebilir.

    OpenClaw, Ollama'nın sessizce 4096 token'lık bir bağlama geri dönmemesi için
    bu modda varsayılan olarak `options.num_ctx` ekler. Proxy'niz bilinmeyen
    `options` alanlarını reddediyorsa bunu devre dışı bırakın:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: false,
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="Bağlam pencereleri">
    Otomatik keşfedilen modellerde OpenClaw, özel Modelfile'lardan gelen daha büyük
    `PARAMETER num_ctx` değerleri de dâhil olmak üzere `/api/show` tarafından bildirilen
    bağlam penceresini kullanır; aksi takdirde OpenClaw'ın varsayılan Ollama bağlam
    penceresine geri döner.

    Sağlayıcı düzeyindeki `contextWindow`, `contextTokens` ve `maxTokens`,
    bu sağlayıcı altındaki her model için varsayılanları belirler ve model başına geçersiz kılınabilir.
    `contextWindow`, OpenClaw'ın kendi istem/Compaction bütçesidir. Yerel
    `/api/chat` istekleri, `params.num_ctx` açıkça ayarlanmadıkça
    `options.num_ctx` değerini ayarlanmamış bırakır; böylece Ollama kendi model,
    `OLLAMA_CONTEXT_LENGTH` veya VRAM tabanlı varsayılanını uygular. Geçersiz, sıfır, negatif
    veya sonlu olmayan `params.num_ctx` değerleri yok sayılır. Daha eski bir yapılandırma,
    yerel istek bağlamını zorlamak için yalnızca `contextWindow`/`maxTokens`
    kullandıysa bunları `params.num_ctx` içine kopyalamak için `openclaw doctor --fix`
    komutunu çalıştırın. OpenAI uyumlu bağdaştırıcı, yapılandırılmış `params.num_ctx`
    veya `contextWindow` değerinden varsayılan olarak hâlâ `options.num_ctx`
    ekler; üst sistem `options` değerini reddediyorsa `injectNumCtxForOpenAICompat: false`
    ile devre dışı bırakın.

    Yerel model girdileri ayrıca `params` altında yaygın Ollama çalışma zamanı
    seçeneklerini kabul eder; bunlar yerel `/api/chat` `options` olarak iletilir: `num_keep`, `seed`,
    `num_predict`, `top_k`, `top_p`, `min_p`, `typical_p`, `repeat_last_n`,
    `temperature`, `repeat_penalty`, `presence_penalty`, `frequency_penalty`,
    `stop`, `num_batch`, `num_gpu`, `main_gpu`, `use_mmap` ve `num_thread`.
    Birkaç anahtar (`format`, `keep_alive`, `truncate`, `shift`), iç içe
    `options` yerine üst düzey istek alanları olarak iletilir. OpenClaw yalnızca
    bu Ollama istek anahtarlarını iletir; dolayısıyla `streaming` gibi yalnızca çalışma
    zamanına ait parametreler Ollama'ya hiçbir zaman gönderilmez. Üst düzey `think`
    değerini ayarlamak için `params.think` (veya `params.thinking`) kullanın;
    `false`, Qwen tarzı düşünme modellerinde API düzeyindeki düşünmeyi devre dışı bırakır.

    ```json5
    {
      models: {
        providers: {
          ollama: {
            contextWindow: 32768,
            models: [
              {
                id: "llama3.3",
                contextWindow: 131072,
                maxTokens: 65536,
                params: {
                  num_ctx: 32768,
                  temperature: 0.7,
                  top_p: 0.9,
                  thinking: false,
                },
              }
            ]
          }
        }
      }
    }
    ```

    Model başına `agents.defaults.models["ollama/<model>"].params.num_ctx` da
    çalışır; ikisi de ayarlanmışsa açık sağlayıcı model girdisi önceliklidir.

  </Accordion>

  <Accordion title="Düşünme denetimi">
    OpenClaw, düşünmeyi Ollama'nın beklediği şekilde iletir: üst düzey `think`,
    `options.think` değil. `/api/show` değeri bir
    `thinking` yeteneği bildiren otomatik keşfedilmiş modeller `/think low`, `/think medium`, `/think high`
    ve `/think max` seçeneklerini sunar; düşünmeyen modeller yalnızca `/think off` seçeneğini sunar.

    ```bash
    openclaw agent --model ollama/gemma4 --thinking off
    openclaw agent --model ollama/gemma4 --thinking low
    ```

    Alternatif olarak bir model varsayılanı ayarlayın:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "ollama/gemma4": {
              thinking: "low",
            },
          },
        },
      },
    }
    ```

    Model başına `params.think`/`params.thinking`, belirli bir model için API
    düşünmesini devre dışı bırakabilir veya zorunlu kılabilir. Etkin çalıştırmada yalnızca örtük
    `off` varsayılanı bulunduğunda OpenClaw bu açık yapılandırmayı
    korur; `/think medium` gibi kapalı olmayan bir çalışma zamanı komutu yine de bunu geçersiz kılar. Doğru değerli
    bir düşünme isteği, açıkça `reasoning: false` olarak işaretlenmiş
    bir modele hiçbir zaman gönderilmez; `think: false` isteği ise her durumda gönderilir.

  </Accordion>

  <Accordion title="Akıl yürütme modelleri">
    `deepseek-r1`, `reasoning`, `reason` veya `think` adlı modeller varsayılan olarak
    akıl yürütme yeteneğine sahip kabul edilir; ek yapılandırma gerekmez:

    ```bash
    ollama pull deepseek-r1:32b
    ```

  </Accordion>

  <Accordion title="Model maliyetleri">
    Ollama yerel olarak ve ücretsiz çalıştığından, hem otomatik keşfedilen hem de
    elle tanımlanan modellerin tüm model maliyetleri `0` değerindedir.
  </Accordion>

  <Accordion title="Bellek gömmeleri">
    Paketle birlikte gelen Ollama Plugin'i, [bellek araması](/tr/concepts/memory) için
    bir bellek gömme sağlayıcısı kaydeder. Yapılandırılmış Ollama temel URL'sini
    ve API anahtarını kullanır, `/api/embed` çağrısı yapar ve mümkün olduğunda
    birden fazla bellek parçasını tek bir `input` isteğinde toplar.

    `proxy.enabled=true` olduğunda, yapılandırılmış `baseUrl` değerinden türetilen
    tam ana makine yerel geri döngü kaynağına yönelik gömme istekleri, yönetilen iletme proxy'si
    yerine OpenClaw'ın korumalı doğrudan yolunu kullanır. Yapılandırılmış
    ana makine adı bizzat `localhost` veya bir geri döngü IP sabiti olmalıdır;
    yalnızca geri döngüye çözümlenen DNS adları yine yönetilen proxy yolunu
    kullanır. LAN, tailnet, özel ağ ve genel Ollama ana makineleri her zaman
    yönetilen proxy yolunda kalır ve başka bir ana makineye/porta yönlendirmeler
    güveni devralmaz. `proxy.loopbackMode: "proxy"`, geri döngü trafiğini yine de
    proxy üzerinden yönlendirir; `proxy.loopbackMode: "block"` ise bağlanmadan önce bunu reddeder;
    bkz. [Yönetilen proxy](/tr/security/network-proxy#gateway-loopback-mode).

    | Özellik | Değer |
    | --- | --- |
    | Varsayılan model | `nomic-embed-text` |
    | Otomatik çekme | Evet, yerel olarak mevcut değilse |
    | Varsayılan satır içi eşzamanlılık | 1 (diğer sağlayıcılarda varsayılan daha yüksektir; ana makine kaldırabiliyorsa `nonBatchConcurrency` ile artırın) |

    Sorgu zamanı gömmeleri, bunları gerektiren veya öneren modeller için
    getirme öneklerini kullanır: `nomic-embed-text`, `qwen3-embedding` ve
    `mxbai-embed-large`. Belge grupları ham hâlde kalır, dolayısıyla mevcut dizinler
    için biçim geçişi gerekmez.

    ```json5
    {
      memory: {
        search: {
          provider: "ollama",
          remote: {
            // Ollama için varsayılan. Yeniden dizinleme çok yavaşsa daha büyük ana makinelerde artırın.
            nonBatchConcurrency: 1,
          },
        },
      },
    }
    ```

    Uzak bir gömme ana makinesi için kimlik doğrulamayı o ana makineyle sınırlı tutun:

    ```json5
    {
      memory: {
        search: {
          provider: "ollama",
          model: "nomic-embed-text",
          remote: {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            nonBatchConcurrency: 2,
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Akış yapılandırması">
    Ollama varsayılan olarak, akış ve araç çağrısını birlikte destekleyen
    **yerel API'yi** (`/api/chat`) kullanır; özel yapılandırma gerekmez.

    Yerel isteklerde düşünme denetimi doğrudan iletilir: açık bir
    `params.think`/`params.thinking` yapılandırılmadığı sürece `/think off`
    ve `openclaw agent --thinking off`, üst düzey `think: false` gönderir; `/think
    low|medium|high` eşleşen efor dizesini gönderir; `/think max`, Ollama'nın en yüksek eforu olan
    `think: "high"` değerine eşlenir.

    <Tip>
    Bunun yerine OpenAI uyumlu uç nokta için yukarıdaki "Eski OpenAI uyumlu mod" bölümüne bakın; burada akış ve araç çağrısı birlikte çalışmayabilir.
    </Tip>

  </Accordion>
</AccordionGroup>

## Sorun giderme

<AccordionGroup>
  <Accordion title="WSL2 çökme döngüsü (yinelenen yeniden başlatmalar)">
    NVIDIA/CUDA kullanılan WSL2'de resmî Ollama Linux yükleyicisi,
    `Restart=always` içeren bir `ollama.service` systemd birimi oluşturur. Bu hizmet
    otomatik başlatılır ve WSL2 önyüklemesi sırasında GPU destekli bir model yüklerse Ollama,
    yükleme esnasında ana makine belleğini sabitleyebilir; Hyper-V bellek geri kazanımı bu
    sayfaları her zaman geri kazanamaz. Bunun sonucunda Windows WSL2 sanal makinesini
    sonlandırabilir, systemd Ollama'yı yeniden başlatır ve döngü yinelenir.

    Kanıtlar: yinelenen WSL2 yeniden başlatmaları/sonlandırmaları, WSL2 başlangıcından
    hemen sonra `app.slice` veya `ollama.service` içinde yüksek CPU kullanımı ve
    Linux OOM sonlandırıcısı yerine systemd'den gelen SIGTERM.

    OpenClaw; WSL2'yi, `Restart=always` ile etkinleştirilmiş `ollama.service`
    değerini ve görünür CUDA işaretlerini algıladığında bir başlangıç uyarısı kaydeder.

    Azaltma:

    ```bash
    sudo systemctl disable ollama
    ```

    Windows tarafında bunu `%USERPROFILE%\.wslconfig` dosyasına ekleyin, ardından
    `wsl --shutdown` komutunu çalıştırın:

    ```ini
    [experimental]
    autoMemoryReclaim=disabled
    ```

    Alternatif olarak canlı tutma süresini kısaltın / Ollama'yı yalnızca gerektiğinde elle başlatın:

    ```bash
    export OLLAMA_KEEP_ALIVE=5m
    ollama serve
    ```

    Bkz. [ollama/ollama#11317](https://github.com/ollama/ollama/issues/11317).

  </Accordion>

  <Accordion title="Ollama algılanmıyor">
    Ollama'nın çalıştığını, `OLLAMA_API_KEY` değerinin (veya bir kimlik doğrulama profilinin)
    ayarlandığını ve `models.providers.ollama` değerinin açıkça tanımlanmadığını doğrulayın:

    ```bash
    ollama serve
    curl http://localhost:11434/api/tags
    ```

  </Accordion>

  <Accordion title="Kullanılabilir model yok">
    Modeli yerel olarak çekin veya `models.providers.ollama` içinde
    açıkça tanımlayın:

    ```bash
    ollama list  # Nelerin yüklü olduğunu görün
    ollama pull gemma4
    ollama pull gpt-oss:20b
    ollama pull llama3.3     # Veya başka bir model
    ```

  </Accordion>

  <Accordion title="Bağlantı reddedildi">
    ```bash
    # Ollama'nın çalışıp çalışmadığını kontrol edin
    ps aux | grep ollama

    # Veya Ollama'yı yeniden başlatın
    ollama serve
    ```

  </Accordion>

  <Accordion title="Uzak ana makine curl ile çalışıyor ancak OpenClaw ile çalışmıyor">
    Gateway'i çalıştıran aynı makine ve çalışma zamanından doğrulayın:

    ```bash
    openclaw gateway status --deep
    curl http://ollama-host:11434/api/tags
    ```

    Yaygın nedenler:

    - `baseUrl`, `localhost` değerini gösteriyor ancak Gateway Docker'da veya başka bir ana makinede çalışıyor.
    - URL, yerel Ollama yerine OpenAI uyumlu davranışı seçen `/v1` değerini kullanıyor.
    - Uzak ana makinede güvenlik duvarı veya LAN bağlama değişiklikleri gerekiyor.
    - Model dizüstü bilgisayarınızdaki daemon'da bulunuyor ancak uzak daemon'da bulunmuyor.

  </Accordion>

  <Accordion title="Model, araç JSON'unu metin olarak çıktılıyor">
    Genellikle sağlayıcı OpenAI uyumlu moddadır veya model araç şemalarını
    işleyemiyordur. Yerel modu tercih edin:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434",
            api: "ollama",
          },
        },
      },
    }
    ```

    Küçük bir yerel model araç şemalarında yine başarısız olursa o model girdisinde
    `compat.supportsTools: false` değerini ayarlayıp yeniden test edin.

  </Accordion>

  <Accordion title="Kimi veya GLM bozuk semboller döndürüyor">
    Uzun, dilsel olmayan sembol dizilerinden oluşan barındırılan Kimi/GLM yanıtları,
    başarılı bir yanıt yerine başarısız bir sağlayıcı çağrısı olarak değerlendirilir.
    Böylece bozuk metni oturuma kalıcı olarak yazmak yerine normal yeniden
    deneme/yedek/ hata işleme devreye girer.

    Sorun yinelenirse model adını, geçerli oturum dosyasını ve çalıştırmanın
    `Cloud + Local` mı yoksa `Cloud only` mı kullandığını kaydedin; ardından yeni
    bir oturum ve yedek model deneyin:

    ```bash
    openclaw infer model run --model ollama/kimi-k2.5:cloud --prompt "Reply with exactly: ok" --json
    openclaw models set ollama/gemma4
    ```

  </Accordion>

  <Accordion title="Soğuk yerel model zaman aşımına uğruyor">
    Büyük yerel modellerin ilk yüklemesi uzun sürebilir. Zaman aşımını Ollama
    sağlayıcısıyla sınırlayın ve isteğe bağlı olarak modeli dönüşler arasında yüklü tutun:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            timeoutSeconds: 300,
            models: [
              {
                id: "gemma4:26b",
                name: "gemma4:26b",
                params: { keep_alive: "15m" },
              },
            ],
          },
        },
      },
    }
    ```

    Ana makinenin bağlantıları kabul etmesi de yavaşsa `timeoutSeconds`,
    bu sağlayıcı için korumalı bağlantı zaman aşımını da uzatır.

  </Accordion>

  <Accordion title="Geniş bağlamlı model çok yavaş veya belleği tükeniyor">
    Birçok model, donanımınızın rahatça çalıştırabileceğinden daha geniş bağlamlar
    bildirir. `params.num_ctx` ayarlanmadığı sürece yerel Ollama kendi çalışma zamanı
    varsayılanını kullanır. Öngörülebilir ilk token gecikmesi için hem OpenClaw'ın
    bütçesini hem de Ollama'nın istek bağlamını sınırlayın:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            contextWindow: 32768,
            maxTokens: 8192,
            models: [
              {
                id: "qwen3.5:9b",
                name: "qwen3.5:9b",
                params: { num_ctx: 32768, thinking: false },
              },
            ],
          },
        },
      },
    }
    ```

    OpenClaw çok fazla istem gönderiyorsa `contextWindow` değerini düşürün.
    Ollama'nın çalışma zamanı bağlamı makine için çok büyükse `params.num_ctx`
    değerini düşürün. Üretim çok uzun sürüyorsa `maxTokens` değerini düşürün.

  </Accordion>
</AccordionGroup>

<Note>
Daha fazla yardım: [Sorun giderme](/tr/help/troubleshooting) ve [SSS](/tr/help/faq).
</Note>

## İlgili

<CardGroup cols={2}>
  <Card title="Ollama Cloud" href="/tr/providers/ollama-cloud" icon="cloud">
    Ayrılmış `ollama-cloud` sağlayıcısıyla yalnızca buluta yönelik kurulum.
  </Card>
  <Card title="Model sağlayıcıları" href="/tr/concepts/model-providers" icon="layers">
    Tüm sağlayıcılara, model referanslarına ve yük devretme davranışına genel bakış.
  </Card>
  <Card title="Model seçimi" href="/tr/concepts/models" icon="brain">
    Modellerin nasıl seçileceği ve yapılandırılacağı.
  </Card>
  <Card title="Ollama Web Araması" href="/tr/tools/ollama-search" icon="magnifying-glass">
    Ollama destekli web aramasının tam kurulum ve davranış ayrıntıları.
  </Card>
  <Card title="Yapılandırma" href="/tr/gateway/configuration" icon="gear">
    Tam yapılandırma başvurusu.
  </Card>
</CardGroup>
