---
read_when:
    - Medya anlamlandırmayı tasarlama veya yeniden düzenleme
    - Gelen ses/video/görüntü ön işlemesini ayarlama
sidebarTitle: Media understanding
summary: Sağlayıcı + CLI geri dönüşleriyle gelen görüntü/ses/video anlama (isteğe bağlı)
title: Medya anlama
x-i18n:
    generated_at: "2026-07-27T00:04:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 38e9a0f89607bb9c4af85689ef0fbd3df9234b36e06d86c129e0d823d6e05143
    source_path: nodes/media-understanding.md
    workflow: 16
---

OpenClaw, yanıt işlem hattı çalışmadan önce gelen medyayı (görüntü/ses/video) özetleyebilir; böylece komut ayrıştırma ve yönlendirme, ham baytlar yerine kısa metin üzerinden çalışır. Anlama özelliği yerel araçları veya sağlayıcı anahtarlarını otomatik olarak algılar; alternatif olarak açık modeller yapılandırılabilir. Orijinal medya her zaman her zamanki gibi modele iletilir; anlama başarısız olduğunda veya devre dışı bırakıldığında yanıt akışı değişmeden devam eder.

Sağlayıcı Plugin'leri yetenek meta verilerini (hangi sağlayıcının hangi medya türünü desteklediği, varsayılan model, öncelik) kaydeder. OpenClaw çekirdeği, paylaşılan `tools.media` yapılandırmasının, geri dönüş sırasının ve yanıt işlem hattı entegrasyonunun sahibidir.

## Nasıl çalışır?

<Steps>
  <Step title="Ekleri toplayın">
    Sıralı gelen medya bilgilerini (`path`, `url`, `contentType` ve `kind`) toplayın.
  </Step>
  <Step title="Yetenek başına seçim yapın">
    Etkinleştirilen her yetenek (görüntü/ses/video) için ekleri `attachments` politikasına göre seçin (varsayılan: yalnızca ilk ek).
  </Step>
  <Step title="Bir model seçin">
    Uygun ilk model girdisini (boyut + yetenek + kimlik doğrulamanın kullanılabilirliği) seçin.
  </Step>
  <Step title="Başarısızlıkta geri dönüş yapın">
    Bir model hata verirse, zaman aşımına uğrarsa veya medya `maxBytes` sınırını aşarsa sonraki girdiyi deneyin.
  </Step>
  <Step title="Başarı durumunda uygulayın">
    `Body`, bir `[Image]`, `[Audio]` veya `[Video]` bloğuna dönüşür. Ses ayrıca `{{Transcript}}` değerini ayarlar; komut ayrıştırma, mevcutsa altyazı metnini, aksi takdirde transkripti kullanır. Altyazılar blok içinde `User text:` olarak korunur.
  </Step>
</Steps>

## Yapılandırma

`tools.media`, yetenek etiketli tek bir model listesinin yanı sıra yetenek başına küçük denetimleri içerir:

```json5
{
  tools: {
    media: {
      concurrency: 2, // eşzamanlı maksimum yetenek çalıştırma sayısı (varsayılan)
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["image", "video"] },
      ],
      image: { preferredModel: "google/gemini-3-flash-preview" },
      audio: { enabled: true },
      video: { enabled: true },
    },
  },
}
```

Yetenek başına (`image`/`audio`/`video`) anahtarlar:

| Anahtar              | Tür      | Varsayılan                                | Notlar                                                                |
| ---------------- | --------- | -------------------------------------- | -------------------------------------------------------------------- |
| `enabled`        | `boolean` | otomatik (`false` devre dışı bırakır)                | Bu yetenek için otomatik algılamayı kapatmak üzere `false` olarak ayarlayın              |
| `preferredModel` | `string`  | ilk uyumlu girdi                 | `provider/model`, model kimliği, `provider:<id>` veya `cli:command` tercih edilir |
| `prompt`         | `string`  | yetenek varsayılanı                     | Bir girdi geçersiz kılmadığında kullanılacak varsayılan istem                    |
| `maxChars`       | `number`  | görüntü/video için `500`, ses için ayarlanmamış         | Varsayılan çıktı sınırı                                                 |
| `maxBytes`       | `number`  | görüntü için 10MB, ses için 20MB, video için 50MB     | Varsayılan girdi sınırı                                                  |
| `timeoutSeconds` | `number`  | görüntü/ses için `60`, video için `120`          | Varsayılan istek zaman aşımı                                              |
| `language`       | `string`  | ayarlanmamış                                  | Ses transkripsiyonu ipucu                                             |
| `scope`          | nesne    | ayarlanmamış                                  | Kanal/sohbet türü/kaynak anahtarına göre geçit uygular                                 |
| `attachments`    | nesne    | `{ mode: "first", maxAttachments: 1 }` | Eşleşen eklerden hangilerinin işleneceğini seçer                      |
| `echoTranscript` | `boolean` | `false`                                | Yalnızca ses: aracı işlemeden önce transkripti yankılar              |
| `echoFormat`     | `string`  | `'📝 "{transcript}"'`                  | Yalnızca ses: yankılanan transkriptin biçimi                         |

İstemler, sınırlar, dil ipuçları, istek geçersiz kılmaları ve sağlayıcı seçenekleri yetenek varsayılanları olarak ayarlanabilir veya ayrı `tools.media.models[]` girdilerinde geçersiz kılınabilir. Açıkça bir model yapılandırılmadığında yetenek varsayılanları otomatik algılanan sağlayıcıları da kapsar.

### Model girdileri

Her `models[]` girdisi bir **sağlayıcı** girdisi (varsayılan) veya bir **CLI** girdisidir:

<Tabs>
  <Tab title="Sağlayıcı girdisi">
    ```json5
    {
      type: "provider", // belirtilmezse varsayılan
      provider: "openai",
      model: "gpt-5.6-sol",
      prompt: "Görüntüyü <= 500 karakterle açıklayın.",
      maxChars: 500,
      maxBytes: 10485760,
      timeoutSeconds: 60,
      capabilities: ["image"],
      profile: "vision-profile",
      preferredProfile: "vision-fallback",
    }
    ```
  </Tab>
  <Tab title="CLI girdisi">
    ```json5
    {
      type: "cli",
      command: "gemini",
      args: [
        "-m",
        "gemini-3-flash",
        "--allowed-tools",
        "read_file",
        "{{AttachmentPath}} konumundaki medyayı okuyun ve <= {{MaxChars}} karakterle açıklayın.",
      ],
      maxChars: 500,
      maxBytes: 52428800,
      timeoutSeconds: 120,
      capabilities: ["video", "image"],
    }
    ```

    CLI şablonları ayrıca `{{AttachmentUrl}}`, `{{AttachmentContentType}}`, `{{AttachmentDir}}`, `{{AttachmentIndex}}`, `{{OutputDir}}` (bu çalıştırma için oluşturulan geçici dizin) ve `{{OutputBase}}` (uzantısız geçici dosya temel yolu) değerlerini kullanabilir. Eski `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` ve `{{MediaDir}}` adları, kullanımdan kaldırılmış uyumluluk takma adları olarak kalır.

  </Tab>
</Tabs>

### Sağlayıcı kimlik bilgileri

Sağlayıcı medya anlama özelliği, normal model çağrılarıyla aynı kimlik doğrulama çözümlemesini kullanır: kimlik doğrulama profilleri, ortam değişkenleri ve ardından `models.providers.<providerId>.apiKey`. `tools.media.models[]` girdileri satır içi `apiKey` alanını kabul etmez.

```json5
{
  models: {
    providers: {
      openai: { apiKey: "<OPENAI_API_KEY>" },
      moonshot: { apiKey: "<MOONSHOT_API_KEY>" },
    },
  },
}
```

Profiller, ortam değişkenleri ve özel temel URL'ler için [Araçlar ve özel sağlayıcılar](/tr/gateway/config-tools) bölümüne bakın.

## Kurallar ve davranış

- `maxBytes` sınırını aşan medya, söz konusu modeli atlar ve sonrakini dener.
- 1024 bayttan küçük ses dosyaları boş/bozuk kabul edilir ve transkripsiyondan önce atlanır; bunun yerine aracı deterministik bir yer tutucu transkript alır.
- Etkin birincil görüntü modeli görme yeteneğini zaten yerel olarak destekliyorsa OpenClaw, `[Image]` özet bloğunu atlar ve orijinal görüntüyü doğrudan modele iletir. MiniMax bir istisnadır: eski MiniMax M2.x sohbet meta verileri görüntü girdisini desteklediğini iddia etse bile `minimax`, `minimax-cn`, `minimax-portal` ve `minimax-portal-cn` görüntü anlamayı her zaman Plugin'e ait `MiniMax-VL-01` medya sağlayıcısı üzerinden yönlendirir (yalnızca `MiniMax-M3` ve sonraki sürümler yerel görme yeteneğine sahip kabul edilir).
- Bir Gateway/WebChat birincil modeli yalnızca metin destekliyorsa görüntü ekleri, görüntü/PDF araçlarının veya yapılandırılmış bir görüntü modelinin eki kaybetmek yerine yine de inceleyebilmesi için dışa aktarılmış `media://inbound/*` referansları olarak korunur.
- Açık `openclaw infer image describe --file <path> --model <provider/model>` (takma ad: `openclaw capability image describe`), `models.providers.ollama.models[]` altında eşleşen, görüntü destekli bir model yapılandırıldığında `ollama/qwen2.5vl:7b` gibi Ollama referansları dâhil olmak üzere görüntü destekli sağlayıcıyı/modeli doğrudan çalıştırır.
- `<capability>.enabled`, `false` değilse ancak hiçbir model yapılandırılmamışsa OpenClaw, sağlayıcısı söz konusu yeteneği desteklediğinde etkin yanıt modelini dener.

### Otomatik algılama (varsayılan)

`tools.media.<capability>.enabled`, `false` değilse ve hiçbir model yapılandırılmamışsa OpenClaw aşağıdakileri sırasıyla dener ve çalışan ilk seçenekte durur:

<Steps>
  <Step title="Yapılandırılmış görüntü modeli (yalnızca görüntü)">
    Etkin yanıt modeli görme yeteneğini zaten yerel olarak desteklemiyorsa `agents.defaults.imageModel` birincil/geri dönüş referansları kullanılır. `provider/model` referanslarını tercih edin; yalın referanslar yalnızca eşleşme benzersiz olduğunda yapılandırılmış, görüntü destekli sağlayıcı model girdilerinden nitelenir.
  </Step>
  <Step title="Etkin yanıt modeli">
    Sağlayıcısı söz konusu yeteneği desteklediğinde etkin yanıt modeli.
  </Step>
  <Step title="Sağlayıcı kimlik doğrulaması (yalnızca ses, yerel CLI'lerden önce)">
    Sesi destekleyen yapılandırılmış `models.providers.*` girdileri, yerel CLI'lerden önce denenir. Birlikte gelen sağlayıcı öncelik sırası (eşitlikler sağlayıcı kimliğine göre alfabetik olarak çözülür): Groq/OpenAI &rarr; xAI &rarr; Deepgram &rarr; OpenRouter &rarr; Google/SenseAudio &rarr; Deepinfra/ElevenLabs &rarr; Mistral.
  </Step>
  <Step title="Yerel CLI'ler (yalnızca ses)">
    Hazır yerel ikili dosyalar, sıralı bir geri dönüş listesine dönüşür:
    - `whisper-cli`, yalnızca geçerli işlemdeki daha önceki bir model çağrısı Metal veya CUDA gözlemledikten sonra ilk sırada
    - CPU varsayılanlı `sherpa-onnx-offline` (`tokens.txt`/`encoder.onnx`/`decoder.onnx`/`joiner.onnx` ile `SHERPA_ONNX_MODEL_DIR` gerektirir)
    - Hızlandırma yalnızca derleme tarafından destekleniyorsa veya gözlemlenmemişse `whisper-cli`
    - Apple Silicon'da `parakeet-mlx` (MLX destekli, cihaz kullanımı gözlemlenmemiş)
    - `whisper` (Python CLI; varsayılan olarak `turbo` modelini kullanır, otomatik olarak indirir)

    Arka uç yetenek incelemesi önbelleğe alınır ve bir model yüklemez. Derleme yeteneği, istenen arka uç bayrakları ve gerçek bir çağrıdan gözlemlenen arka uç ayrı tutulur. Otomatik algılanan whisper.cpp, yukarı akış tarafından seçilen arka uç satırının kaydedilebilmesi için model çalıştırma günlüklerini etkin bırakır. Açık CLI girdileri yapılandırılmış sıralarını, arka uç bayraklarını ve çıktı bayraklarını korur.

  </Step>
  <Step title="Sağlayıcı kimlik doğrulaması (görüntü/video)">
    Yeteneği destekleyen yapılandırılmış `models.providers.*` girdileri, birlikte gelen geri dönüş sırasından önce denenir. Görüntü destekli bir modele sahip yalnızca görüntü yapılandırma sağlayıcıları, birlikte gelen bir sağlayıcı Plugin'i olmasalar bile medya anlama için otomatik olarak kaydedilir.

    Birlikte gelen sağlayıcı öncelik sırası (eşitlikler sağlayıcı kimliğine göre alfabetik olarak çözülür):
    - Görüntü: Anthropic/OpenAI &rarr; Google &rarr; MiniMax &rarr; Deepinfra &rarr; MiniMax Portal &rarr; Z.AI
    - Video: Google &rarr; Qwen &rarr; Moonshot

  </Step>
  <Step title="Antigravity CLI (yalnızca görüntü/video)">
    Yüklenmiş ilk `agy` veya `antigravity` ikili dosyası (`OPENCLAW_ANTIGRAVITY_CLI` ile geçersiz kılınabilir), medyanın dizinine karşı korumalı alanda çalıştırılır.
  </Step>
</Steps>

Bir yetenek için otomatik algılamayı devre dışı bırakmak üzere:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: false,
      },
    },
  },
}
```

<Note>
İkili dosya algılama macOS/Linux/Windows genelinde en iyi çaba esasına dayanır; CLI'nin `PATH` üzerinde olduğundan (`~` genişletilir) emin olun veya tam komut yoluyla açık bir CLI model girdisi ayarlayın.
</Note>

### Proxy desteği (ses/video sağlayıcı çağrıları)

Sağlayıcı tabanlı **ses** ve **video** anlama, `NO_PROXY`/`no_proxy` atlama kuralları dâhil standart giden proxy ortam değişkenlerine uyar: `HTTPS_PROXY`, `HTTP_PROXY`, `ALL_PROXY`, `https_proxy`, `http_proxy`, `all_proxy`. Küçük harfli değişkenler, büyük harfli olanlara göre önceliklidir. Hiçbiri ayarlanmamışsa medya anlama doğrudan çıkış kullanır; proxy değeri hatalı biçimlendirilmişse OpenClaw bir uyarı günlüğe kaydeder ve doğrudan getirmeye geri döner. Görüntü anlama bu proxy yolundan geçmez.

## Yetenekler

Belirli medya türleriyle sınırlamak için bir `models[]` girdisinde `capabilities` değerini ayarlayın. Paylaşılan listeler için OpenClaw, birlikte gelen sağlayıcı başına varsayılanları çıkarır:

| Sağlayıcı                                                                 | Yetenekler             |
| ------------------------------------------------------------------------ | ---------------------- |
| `openai`, `anthropic`, `minimax`                                         | görüntü                |
| `minimax-portal`                                                         | görüntü                |
| `moonshot`                                                               | görüntü + video        |
| `openrouter`                                                             | görüntü + ses          |
| `google` (Gemini API)                                                    | görüntü + ses + video  |
| `qwen`                                                                   | görüntü + video        |
| `deepinfra`                                                              | görüntü + ses          |
| `mistral`                                                                | ses                    |
| `zai`                                                                    | görüntü                |
| `groq`, `xai`, `deepgram`, `senseaudio`                                  | ses                    |
| Görüntü destekli bir modele sahip herhangi bir `models.providers.<id>.models[]` kataloğu | görüntü                |

CLI girdilerinde beklenmedik eşleşmeleri önlemek için `capabilities` değerini açıkça ayarlayın; belirtilmezse girdi, göründüğü her yetenek listesi için uygun kabul edilir.

## Sağlayıcı destek matrisi

| Yetenek | Sağlayıcılar                                                                                                                                               | Notlar                                                                                                                                                                                   |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Görüntü      | Anthropic, Codex app-server, Deepinfra, Google, MiniMax, MiniMax Portal, Moonshot, OpenAI, OpenAI Codex OAuth, OpenRouter, Qwen, Z.AI, yapılandırma sağlayıcıları | Sağlayıcı pluginleri görüntü desteğini kaydeder; `openai/*`, API anahtarı veya Codex OAuth yönlendirmesini kullanabilir; `codex/*`, sınırlı bir Codex app-server turu kullanır; görüntü destekli yapılandırma sağlayıcıları otomatik olarak kaydedilir. |
| Ses      | Deepgram, Deepinfra, ElevenLabs, Google, Groq, Mistral, OpenAI, OpenRouter, SenseAudio, xAI                                                             | Sağlayıcı transkripsiyonu (Whisper/Groq/xAI/Deepgram/OpenRouter STT/Gemini/SenseAudio/Scribe/Voxtral).                                                                                     |
| Video      | Google, Moonshot, Qwen                                                                                                                                  | Sağlayıcı pluginleri aracılığıyla video anlama; Qwen video anlama, standart DashScope uç noktalarını kullanır.                                                                        |

<Note>
**MiniMax notu**: Eski MiniMax M2.x sohbet meta verileri görüntü girdisi desteklediğini belirtse bile `minimax`, `minimax-cn`, `minimax-portal` ve `minimax-portal-cn` için görüntü anlama her zaman pluginin sahip olduğu `MiniMax-VL-01` medya sağlayıcısından gelir.
</Note>

## Model seçimi rehberi

- Kalite ve güvenlik önemli olduğunda her medya yeteneği için mevcut neslin en güçlü modelini tercih edin.
- Güvenilmeyen girdileri işleyen araç destekli aracılarda eski veya daha zayıf medya modellerinden kaçının.
- Kullanılabilirlik için her yetenek başına en az bir yedek bulundurun (kaliteli model + daha hızlı/ucuz model).
- CLI yedekleri (`whisper-cli`, `whisper`, `gemini`), sağlayıcı API'leri kullanılamadığında yardımcı olur.
- Bilinen dosya çıktı modları belirleyicidir: çıkarılan transkript dosyası boşsa veya yoksa CLI ilerleme çıktısına geri dönülmez ve transkript üretilmez.
- `parakeet-mlx`: `--output-dir` ve varsayılan `{filename}` çıktı şablonuyla `--output-format txt` (veya `all`) kullanın. Üst kaynaklı `PARAKEET_OUTPUT_FORMAT` ve `PARAKEET_OUTPUT_TEMPLATE` ortam değişkenleri de dikkate alınır. OpenClaw, `<output-dir>/<media-basename>.txt` değerini okur; varsayılan `srt` biçimi, diğer biçimler ve özel çıktı şablonları stdout'u kullanmaya devam eder.

## Ek politikası

Yetenek başına `attachments`, hangi eklerin işlendiğini denetler:

<ParamField path="mode" type='"first" | "all"' default="first">
  Yalnızca seçilen ilk eki veya tümünü işleyin.
</ParamField>
<ParamField path="maxAttachments" type="number" default="1">
  İşlenen eklerin sayısını sınırlayın.
</ParamField>
<ParamField path="prefer" type='"first" | "last" | "path" | "url"'>
  Aday ekler arasındaki seçim tercihi.
</ParamField>

`mode: "all"` olduğunda çıktılar `[Image 1/2]`, `[Audio 2/2]` vb. olarak etiketlenir.

### Dosya eki çıkarımı

- Çıkarılan dosya metni, medya istemine eklenmeden önce güvenilmeyen harici içerik olarak sarmalanır; bunun için `<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>` gibi sınır işaretçileri ve bir `Source: External` meta veri satırı kullanılır.
- Bu yol, medya istemini kısa tutmak için uzun `SECURITY NOTICE:` başlığını kasıtlı olarak atlar; sınır işaretçileri ve meta veriler yine uygulanır.
- Çıkarılabilir metni olmayan bir dosya `[No extractable text]` alır.
- Bir PDF, oluşturulmuş sayfa görüntülerine geri dönerse OpenClaw bu görüntüleri görsel destekli yanıt modellerine iletir ve dosya bloğundaki `[PDF content rendered to images]` yer tutucusunu korur.

## Yapılandırma örnekleri

<Tabs>
  <Tab title="Paylaşılan modeller + geçersiz kılmalar">
    ```json5
    {
      tools: {
        media: {
          models: [
            { provider: "openai", model: "gpt-5.6-sol", capabilities: ["image"] },
            {
              provider: "google",
              model: "gemini-3-flash-preview",
              capabilities: ["image", "audio", "video"],
            },
            {
              type: "cli",
              command: "gemini",
              args: [
                "-m",
                "gemini-3-flash",
                "--allowed-tools",
                "read_file",
                "{{AttachmentPath}} konumundaki medyayı okuyun ve <= {{MaxChars}} karakterle açıklayın.",
              ],
              capabilities: ["image", "video"],
            },
          ],
          audio: {
            attachments: { mode: "all", maxAttachments: 2 },
          },
          video: {
            maxChars: 500,
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Yalnızca ses + video">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [
              { provider: "openai", model: "gpt-4o-mini-transcribe" },
              {
                type: "cli",
                command: "whisper",
                args: ["--model", "base", "{{AttachmentPath}}"],
              },
            ],
          },
          video: {
            enabled: true,
            maxChars: 500,
            models: [
              { provider: "google", model: "gemini-3-flash-preview" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "{{AttachmentPath}} konumundaki medyayı okuyun ve <= {{MaxChars}} karakterle açıklayın.",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Yalnızca görüntü">
    ```json5
    {
      tools: {
        media: {
          image: {
            enabled: true,
            maxBytes: 10485760,
            maxChars: 500,
            models: [
              { provider: "openai", model: "gpt-5.6-sol" },
              { provider: "anthropic", model: "claude-opus-5" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "{{AttachmentPath}} konumundaki medyayı okuyun ve <= {{MaxChars}} karakterle açıklayın.",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Çok modlu tek girdi">
    ```json5
    {
      tools: {
        media: {
          image: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          audio: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          video: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## Durum çıktısı

Medya anlama çalıştığında `/status`, yetenek başına bir özet satırı içerir:

```
📎 Medya: görüntü başarılı (openai/gpt-5.6-sol) · ses başarılı (whisper-cli gözlemlenen=metal)
```

Ön kontrol envanteri için `openclaw capability audio providers` çalıştırın. Yerel satırlar, yerel yedek kazananı genel sağlayıcı seçiminden, hazırlık durumundan ve ayrı yeterli/istenen/gözlemlenen arka uç alanlarından ayrı olarak gösterir. Aynı yerel seçim, bilgilendirici bir doctor bulgusu olarak da kullanılabilir:

```bash
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

## Notlar

- Anlama, mümkün olan en iyi çabayla gerçekleştirilir. Hatalar yanıtları engellemez.
- Anlama devre dışı olsa bile ekler modellere iletilmeye devam eder.
- Anlamanın nerede çalışacağını sınırlamak için `scope` kullanın (örneğin yalnızca DM'lerde).

## İlgili

- [Yapılandırma](/tr/gateway/configuration)
- [Görüntü ve medya desteği](/tr/nodes/images)
