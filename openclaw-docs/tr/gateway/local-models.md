---
read_when:
    - Modelleri kendi GPU sunucunuzdan sunmak istiyorsunuz
    - LM Studio veya OpenAI uyumlu bir proxy bağlıyorsunuz
    - En güvenli yerel model yönlendirmesine ihtiyacınız var
summary: OpenClaw'u yerel LLM'lerde çalıştırın (LM Studio, vLLM, LiteLLM, özel OpenAI uç noktaları)
title: Yerel modeller
x-i18n:
    generated_at: "2026-07-26T22:46:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: af76c9e97bd1d3c9665c347944511b4f466f0b620bb8af7b5f95b1e9145aadec
    source_path: gateway/local-models.md
    workflow: 16
---

Yerel modeller çalışır, ancak donanım, bağlam boyutu ve istem enjeksiyonuna karşı savunma gereksinimlerini artırırlar: küçük veya agresif biçimde nicemlenmiş modeller bağlamı keser ve sağlayıcı tarafındaki güvenlik filtrelerini atlar. Bu sayfa, üst düzey yerel yığınları ve özel OpenAI uyumlu sunucuları ele alır. En sorunsuz yol için [LM Studio](/tr/providers/lmstudio) veya [Ollama](/tr/providers/ollama) ile başlayın ve `openclaw onboard`.

Yalnızca seçilen bir model ihtiyaç duyduğunda başlatılması gereken yerel sunucular için [Yerel model hizmetleri](/tr/gateway/local-model-services) bölümüne bakın.

## Asgari donanım

Rahat bir ajan döngüsü için **2 veya daha fazla tam donanımlı Mac Studio ya da eşdeğer bir GPU sistemi (~$30k+)** hedefleyin. Tek bir **24 GB** GPU, yalnızca daha hafif istemleri daha yüksek gecikmeyle işleyebilir. Her zaman barındırabileceğiniz **en büyük / tam boyutlu varyantı** çalıştırın - küçük veya yoğun biçimde nicemlenmiş denetim noktaları istem enjeksiyonu riskini artırır (bkz. [Güvenlik](/tr/gateway/security)).

## Arka uç seçme

| Arka uç                                             | Kullanım durumu                                                                      |
| --------------------------------------------------- | ------------------------------------------------------------------------------------ |
| [ds4](/tr/providers/ds4)                               | OpenAI uyumlu araç çağrılarıyla macOS Metal üzerinde yerel DeepSeek V4 Flash         |
| [LM Studio](/tr/providers/lmstudio)                    | İlk yerel kurulum, GUI yükleyici, yerel Responses API                                |
| LiteLLM / OAI-proxy / özel OpenAI uyumlu proxy      | Başka bir model API'sinin önüne proxy koyduğunuzda ve OpenClaw'un bunu OpenAI olarak değerlendirmesi gerektiğinde |
| MLX / vLLM / SGLang                                 | OpenAI uyumlu HTTP uç noktasıyla yüksek verimli, kendi barındırdığınız sunum          |
| [Ollama](/tr/providers/ollama)                         | CLI iş akışı, model kitaplığı, müdahale gerektirmeyen systemd hizmeti                 |

Arka uç desteklediğinde `api: "openai-responses"` kullanın (LM Studio destekler). Aksi takdirde `api: "openai-completions"` kullanın. `baseUrl` içeren özel bir sağlayıcıda `api` belirtilmezse OpenClaw varsayılan olarak `openai-completions` kullanır.

<Warning>
**WSL2 + Ollama + NVIDIA/CUDA:** Resmî Ollama Linux yükleyicisi, `Restart=always` içeren bir systemd hizmetini etkinleştirir. WSL2 GPU kurulumlarında otomatik başlatma, önyükleme sırasında son modeli yeniden yükleyip ana makine belleğini sabitleyerek sanal makinenin tekrar tekrar yeniden başlamasına neden olabilir. [WSL2 çökme döngüsü](/tr/providers/ollama#troubleshooting) bölümüne bakın.
</Warning>

## LM Studio + büyük yerel model (Responses API)

Bu, şu anda en iyi yerel yığındır. LM Studio'da büyük bir model (tam boyutlu bir Qwen, DeepSeek veya Llama derlemesi) yükleyin, yerel sunucuyu etkinleştirin (varsayılan `http://127.0.0.1:1234`) ve akıl yürütmeyi nihai metinden ayrı tutmak için Responses API'yi kullanın.

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "lmstudio/my-local-model": { alias: "Local" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Kurulum kontrol listesi:

- LM Studio'yu yükleyin: [https://lmstudio.ai](https://lmstudio.ai)
- **Mevcut en büyük model derlemesini** indirin ("small"/yoğun biçimde nicemlenmiş varyantlardan kaçının), sunucuyu başlatın ve `http://127.0.0.1:1234/v1/models` komutunun modeli listelediğini doğrulayın.
- `my-local-model` değerini LM Studio'da gösterilen gerçek model kimliğiyle değiştirin.
- Modeli yüklü tutun; soğuk yükleme, başlangıç gecikmesi ekler.
- LM Studio derlemeniz farklıysa `contextWindow`/`maxTokens` değerlerini ayarlayın.
- WhatsApp için yalnızca nihai metnin gönderilmesi amacıyla Responses API'yi kullanmaya devam edin.
- Barındırılan modellerin yedek olarak kullanılabilir kalması için `models.mode: "merge"` değerini koruyun.

### Hibrit yapılandırma: birincil barındırılan model, yerel yedek

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["lmstudio/my-local-model", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "lmstudio/my-local-model": { alias: "Local" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Barındırılan bir güvenlik ağıyla yerel modeli öncelikli kullanmak için `primary`/`fallbacks` sırasını değiştirin ve aynı `providers` bloğunu ve `models.mode: "merge"` değerini koruyun.

### Bölgesel barındırma / veri yönlendirme

Barındırılan MiniMax/Kimi/GLM varyantları, bölgeye sabitlenmiş uç noktalarla (örneğin ABD'de barındırılan) OpenRouter üzerinde de bulunur. Anthropic/OpenAI yedekleri için `models.mode: "merge"` değerini korurken trafiği seçtiğiniz yargı alanında tutmak için bölgesel varyantı seçin. Yalnızca yerel kullanım hâlâ en güçlü gizlilik yoludur; sağlayıcı özelliklerine ihtiyaç duyduğunuz ancak veri akışı üzerinde denetim istediğiniz durumlarda barındırılan bölgesel yönlendirme orta yolu sunar.

## Diğer OpenAI uyumlu yerel proxy'ler

MLX (`mlx_lm.server`), vLLM, SGLang, LiteLLM, OAI-proxy veya herhangi bir özel Gateway, OpenAI tarzı bir `/v1/chat/completions` uç noktası sunduğu sürece çalışır. Arka uç `/v1/responses` desteğini açıkça belgelemediği sürece `openai-completions` kullanın.

```json5
{
  agents: {
    defaults: {
      model: { primary: "local/my-local-model" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "sk-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 120000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Özel/yerel sağlayıcı girdileri; geri döngü, LAN, tailnet ve özel DNS ana makineleri dâhil olmak üzere korumalı model isteklerinde tam olarak yapılandırılmış `baseUrl` kökenine güvenir. Meta veri/bağlantı-yerel kökenleri ne olursa olsun her zaman engellenir. Diğer özel kökenlere yapılan istekler yine `models.providers.<id>.request.allowPrivateNetwork: true` gerektirir; tam köken güvenini devre dışı bırakmak için güven bayrağını `false` olarak ayarlayın.

`models.providers.<id>.models[].id` sağlayıcıya özeldir - sağlayıcı önekini eklemeyin. `mlx_lm.server --model mlx-community/Qwen3-30B-A3B-6bit` ile başlatılan bir MLX sunucusu için:

- `models.providers.mlx.models[].id: "mlx-community/Qwen3-30B-A3B-6bit"`
- `agents.defaults.model.primary: "mlx/mlx-community/Qwen3-30B-A3B-6bit"`

Görüntü eklerinin ajan turlarına eklenmesi için yerel veya proxy üzerinden sunulan görüntü modellerinde `input: ["text", "image"]` ayarlayın. Etkileşimli özel sağlayıcı başlangıç kurulumu, yaygın görüntü modeli kimliklerini çıkarır ve yalnızca bilinmeyen adlar hakkında soru sorar; etkileşimsiz başlangıç kurulumu da aynı çıkarımı kullanır ve bunu geçersiz kılmak için `--custom-image-input` / `--custom-text-input` seçeneklerini sunar.

`agents.defaults.timeoutSeconds` değerini artırmadan önce yavaş yerel/uzak model sunucuları için `models.providers.<id>.timeoutSeconds` kullanın. Sağlayıcı zaman aşımı yalnızca model HTTP istekleri için bağlantıyı, üstbilgileri, gövde akışını ve korumalı getirmenin toplam iptal süresini kapsar - ajan/çalıştırma zaman aşımı daha düşükse onu da artırın; çünkü sağlayıcı zaman aşımı tüm çalıştırmanın süresini uzatamaz.

<Note>
Özel OpenAI uyumlu sağlayıcılarda, `baseUrl` geri döngüye, özel bir LAN'a, `.local` değerine veya yalın bir ana makine adına çözümlendiğinde `apiKey: "ollama-local"` gibi gizli olmayan bir yerel işaretçi kabul edilir - OpenClaw bunu eksik anahtar olarak bildirmek yerine geçerli bir yerel kimlik bilgisi olarak değerlendirir. Genel bir ana makine adını kabul eden tüm sağlayıcılarda gerçek bir değer kullanın.
</Note>

Yerel/proxy üzerinden sunulan `/v1` arka uçlarına ilişkin davranış notları:

- OpenClaw bunları yerel OpenAI uç noktaları olarak değil, proxy tarzı OpenAI uyumlu rotalar olarak değerlendirir.
- Yalnızca yerel OpenAI'ye özgü istek biçimlendirmesi uygulanmaz: `service_tier` yoktur, Responses `store` yoktur, OpenAI akıl yürütme uyumluluğu için yük biçimlendirmesi yoktur, istem önbelleği ipuçları yoktur.
- Gizli OpenClaw ilişkilendirme üstbilgileri (`originator`, `version`, `User-Agent`) özel proxy URL'lerine eklenmez.

Uyumluluk bildirimleri yalnızca bu sağlayıcı satırında açıklanan özel uç nokta içindir. Katalogda bilinen rotalar bunun yerine sağlayıcının sahip olduğu yetenekleri kullanır; [özel sağlayıcı yetenek kılavuzuna](/tr/gateway/config-tools#custom-provider-capability-declarations) bakın.

Daha katı OpenAI uyumlu arka uçlar için uyumluluk geçersiz kılmaları:

- **Yalnızca dize içeriği**: bazı sunucular yapılandırılmış içerik parçası dizilerini değil, yalnızca dize biçimindeki `messages[].content` değerini kabul eder. `models.providers.<provider>.models[].compat.requiresStringContent: true` ayarlayın.
- **Katı ileti anahtarları**: sunucu `role`/`content` dışında anahtarlar içeren ileti girdilerini reddediyorsa `compat.strictMessageKeys: true` ayarlayın.
- **Köşeli parantezli araç metni**: bazı yerel modeller, `[tool_name]` sonrasında JSON ve ardından `[END_TOOL_REQUEST]` gibi bağımsız köşeli parantezli araç isteklerini metin olarak üretir. OpenClaw bunları yalnızca ad, ilgili tur için kayıtlı bir araçla tam olarak eşleştiğinde gerçek araç çağrılarına dönüştürür; aksi takdirde gizli, desteklenmeyen metin olarak kalır.
- **Yapılandırılmamış, araç çağrısına benzeyen metin**: bir model araç çağrısına benzeyen ancak yapılandırılmış bir çağrı olmayan JSON/XML/ReAct tarzı metin üretirse OpenClaw bunu metin olarak bırakır ve çalıştırma kimliği, sağlayıcı/model, algılanan kalıp ve mevcut olduğunda araç adıyla birlikte bir uyarı kaydeder. Bu, tamamlanmış bir araç çalıştırması değil, sağlayıcı/model uyumsuzluğudur.
- **Araç kullanımını zorlama**: araçlar asistan metni olarak görünüyorsa (ham JSON/XML/ReAct veya boş bir `tool_calls` dizisi), önce sunucunun sohbet şablonunun/ayrıştırıcısının araç çağrılarını desteklediğini doğrulayın. Ayrıştırıcı yalnızca araç kullanımı zorlandığında çalışıyorsa `tool_choice: "auto"` varsayılan proxy değerini model bazında geçersiz kılın:

  ```json5
  {
    agents: {
      defaults: {
        models: {
          "local/my-local-model": {
            params: {
              extra_body: {
                tool_choice: "required",
              },
            },
          },
        },
      },
    },
  }
  ```

  Bunu yalnızca her normal turun bir araç çağırması gereken durumlarda kullanın. `local/my-local-model` değerini `openclaw models list` içindeki tam referansla değiştirin veya CLI üzerinden ayarlayın:

  ```bash
  openclaw config set agents.defaults.models '{"local/my-local-model":{"params":{"extra_body":{"tool_choice":"required"}}}}' --strict-json --merge
  ```

- **Ek akıl yürütme düzeyleri**: özel bir OpenAI uyumlu model, yerleşik profilin ötesindeki OpenAI akıl yürütme düzeylerini kabul ediyorsa bunları modelin uyumluluk bloğunda bildirin. `"xhigh"` eklemek, bu model referansı için bunu `/think xhigh`, oturum seçiciler, Gateway doğrulaması ve `llm-task` doğrulamasında kullanılabilir hâle getirir:

  ```json5
  {
    models: {
      providers: {
        local: {
          baseUrl: "http://127.0.0.1:8000/v1",
          apiKey: "sk-local",
          api: "openai-responses",
          models: [
            {
              id: "gpt-5.4",
              name: "Yerel proxy üzerinden GPT 5.4",
              reasoning: true,
              input: ["text"],
              cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
              contextWindow: 196608,
              maxTokens: 8192,
              compat: {
                supportedReasoningEfforts: ["low", "medium", "high", "xhigh"],
                reasoningEffortMap: { xhigh: "xhigh" },
              },
            },
          ],
        },
      },
    },
  }
  ```

## Daha küçük veya daha katı arka uçlar

Model sorunsuz yükleniyor ancak tam ajan turları hatalı davranıyorsa yukarıdan aşağıya ilerleyin: önce aktarımı doğrulayın, ardından yüzeyi daraltın.

1. **Yerel modelin yanıt verdiğini doğrulayın** - araç ve ajan bağlamı olmadan:

   ```bash
   openclaw infer model run --local --model <provider/model> --prompt "Tam olarak şununla yanıt ver: pong" --json
   ```

2. **Gateway yönlendirmesini doğrulayın** - yalnızca istemi gönderir; transkripti, AGENTS önyüklemesini, bağlam motoru derlemesini, araçları ve paketlenmiş MCP sunucularını atlar ancak yine de Gateway yönlendirmesini, kimlik doğrulamayı ve sağlayıcı seçimini çalıştırır:

   ```bash
   openclaw infer model run --gateway --model <provider/model> --prompt "Tam olarak şununla yanıt ver: pong" --json
   ```

3. Her iki yoklama da başarılı olduğu hâlde gerçek ajan turları hatalı biçimlendirilmiş araç çağrıları veya aşırı büyük istemler nedeniyle başarısız oluyorsa **yalın modu deneyin**: `agents.defaults.experimental.localModelLean: true` değerini ayarlayın. Açıkça gerekli olmadıkları sürece ağır tarayıcı, cron, mesaj, medya oluşturma, ses ve PDF araçlarını kaldırır; daha büyük araç kataloglarını varsayılan olarak yapılandırılmış Tool Search denetimlerinin arkasına alırken `exec` öğesini doğrudan görünür tutar. Ayrıntılar ve etkin olduğunu doğrulama yöntemi için [Deneysel Özellikler -> Yerel model yalın modu](/tr/concepts/experimental-features#local-model-lean-mode) bölümüne bakın.

4. Son çare olarak, ilgili model için `models.providers.<provider>.models[].compat.supportsTools: false` değerini ayarlayarak **araçları tamamen devre dışı bırakın** - ajan bundan sonra araç çağrıları olmadan çalışır.

5. **Bunun ötesinde darboğaz yukarı akıştadır.** Arka uç, yalın mod ve `supportsTools: false` sonrasında hâlâ yalnızca daha büyük OpenClaw çalıştırmalarında başarısız oluyorsa kalan sorun genellikle OpenClaw'ın aktarım katmanı değil; modelin veya sunucunun kendisidir: bağlam penceresi, GPU belleği, kv-cache tahliyesi veya bir arka uç hatası.

## Sorun giderme

- **Gateway proxy'ye ulaşamıyor mu?** `curl http://127.0.0.1:1234/v1/models`.
- **LM Studio modeli yüklenmemiş mi?** Yeniden yükleyin; soğuk başlatma, yaygın bir "takılma" nedenidir.
- **Yerel sunucu `terminated`, `ECONNRESET` diyor veya akışı turun ortasında kapatıyor mu?** OpenClaw, tanılamalara düşük kardinaliteli bir `model.call.error.failureKind` ile OpenClaw işleminin RSS/heap anlık görüntüsünü kaydeder. LM Studio/Ollama bellek baskısı için model sunucusunun sonlandırılıp sonlandırılmadığını doğrulamak üzere bu zaman damgasını sunucu günlüğüyle veya bir macOS çökme/jetsam günlüğüyle eşleştirin.
- **Bağlam hataları mı var?** OpenClaw, bağlam penceresi ön kontrol eşiklerini algılanan model penceresinden (veya `agents.defaults.contextTokens` bunu düşürüyorsa sınırlandırılmış pencereden) türetir; %20'nin altında **8k** alt sınırıyla uyarır ve %10'un altında **4k** alt sınırıyla kesin olarak engeller (aşırı büyük model meta verilerinin geçerli bir kullanıcı sınırını reddetmemesi için etkin bağlam penceresiyle sınırlandırılır). `contextWindow` değerini düşürün veya sunucu/model bağlam sınırını yükseltin.
- **`messages[].content ... expected a string`?** İlgili model girdisine `compat.requiresStringContent: true` ekleyin.
- **`validation.keys` veya "mesaj girdileri yalnızca `role` ve `content` değerlerine izin veriyor" mu?** İlgili model girdisine `compat.strictMessageKeys: true` ekleyin.
- **Doğrudan `/v1/chat/completions` çağrıları çalışıyor ancak `openclaw infer model run --local` Gemma veya başka bir yerel modelde başarısız mı oluyor?** Önce sağlayıcı URL'sini, model referansını, kimlik doğrulama işaretçisini ve sunucu günlüklerini kontrol edin; `model run` ajan araçlarını tamamen atlar. `model run` başarılı oluyor ancak daha büyük ajan turları başarısız oluyorsa araç yüzeyini `localModelLean` veya `compat.supportsTools: false` ile azaltın.
- **Araç çağrıları ham JSON/XML/ReAct metni olarak mı görünüyor veya sağlayıcı boş bir `tool_calls` dizisi mi döndürüyor?** Asistan metnini körlemesine araç yürütmeye dönüştüren bir proxy eklemeyin; önce sunucunun sohbet şablonunu/ayrıştırıcısını düzeltin. Model yalnızca araç kullanımı zorlandığında çalışıyorsa yukarıdaki `params.extra_body.tool_choice: "required"` geçersiz kılmasını ekleyin ve bu model girdisini yalnızca her turda bir araç çağrısının beklendiği oturumlar için kullanın.
- **Güvenlik**: Yerel modeller, sağlayıcı tarafındaki filtreleri atlar. İstem enjeksiyonunun etki alanını sınırlamak için ajanları dar kapsamlı ve Compaction'ı etkin tutun.

## İlgili

- [Yapılandırma referansı](/tr/gateway/configuration-reference)
- [Model yük devretmesi](/tr/concepts/model-failover)
