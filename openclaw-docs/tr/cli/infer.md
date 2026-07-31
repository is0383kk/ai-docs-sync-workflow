---
read_when:
    - '`openclaw infer` komutlarını ekleme veya değiştirme'
    - Kararlı başsız yetenek otomasyonu tasarlama
summary: Sağlayıcı destekli model, görüntü, ses, TTS, video, web ve gömme iş akışları için önce çıkarım yapan CLI
title: Çıkarım CLI'si
x-i18n:
    generated_at: "2026-07-26T22:40:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3147bb516a08e12c4eacd6bd527af62049ecae25b5fde9439da6a4431c147b07
    source_path: cli/infer.md
    workflow: 16
---

`openclaw infer`, sağlayıcı destekli çıkarım için standart başsız yüzeydir. Ham Gateway RPC adlarını veya ajan araç kimliklerini değil, yetenek ailelerini (`model`, `image`, `audio`, `tts`, `video`, `web`, `embedding`) sunar. `openclaw capability ...`, aynı komut ağacının diğer adıdır.

Tek seferlik bir sağlayıcı sarmalayıcısı yerine bunu tercih etme nedenleri:

- OpenClaw'da önceden yapılandırılmış sağlayıcıları ve modelleri yeniden kullanır.
- Betikler ve ajan güdümlü otomasyon için kararlı `--json` zarfı sunar (bkz. [JSON çıktısı](#json-output)).
- Çoğu alt komut için Gateway olmadan normal yerel yolu çalıştırır.
- Uçtan uca sağlayıcı denetimlerinde, sağlayıcı isteği gönderilmeden önce yayımlanan CLI'ı, yapılandırma yüklemeyi, varsayılan ajan çözümlemesini, paketlenmiş Plugin etkinleştirmeyi ve paylaşılan yetenek çalışma zamanını sınar.

## Infer'ı bir skill'e dönüştürme

Bunu kopyalayıp bir ajana yapıştırın:

```text
https://docs.openclaw.ai/cli/infer sayfasını okuyun, ardından yaygın iş akışlarımı `openclaw infer` komutuna yönlendiren bir skill oluşturun.
Model çalıştırmalarına, görüntü oluşturmaya, video oluşturmaya, ses transkripsiyonuna, TTS'ye, web aramasına ve gömmelere odaklanın.
```

Infer tabanlı iyi bir skill, yaygın kullanıcı amaçlarını doğru alt komutla eşleştirir, her iş akışı için birkaç standart örnek içerir, daha düşük seviyeli alternatifler yerine `openclaw infer ...` tercih eder ve skill gövdesinde infer yüzeyinin tamamını yeniden belgelemez.

## Komut ağacı

```text
 openclaw infer
  list
  inspect

  model
    run
    list
    inspect
    providers
    auth login
    auth logout
    auth status

  image
    generate
    edit
    describe
    describe-many
    providers

  audio
    transcribe
    providers

  tts
    convert
    voices
    providers
    personas
    status
    enable
    disable
    set-provider
    set-persona

  video
    generate
    describe
    providers

  web
    search
    fetch
    providers

  embedding
    create
    providers
```

`infer list` / `infer inspect --name <capability>`, bu ağacı veri olarak gösterir (yetenek kimliği, aktarımlar, açıklama).

## Yaygın görevler

| Görev                              | Komut                                                                                         | Notlar                                                             |
| ---------------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Metin/model istemi çalıştırma      | `openclaw infer model run --prompt "..." --json`                                              | Varsayılan olarak yerel                                            |
| Görüntüler üzerinde model istemi çalıştırma | `openclaw infer model run --prompt "Describe this" --file ./image.png --model provider/model` | Birden fazla görüntü için `--file` seçeneğini yineleyin  |
| Görüntü oluşturma                  | `openclaw infer image generate --prompt "..." --json`                                         | Mevcut bir dosyadan başlarken `image edit` kullanın          |
| Görüntü dosyasını veya URL'yi açıklama | `openclaw infer image describe --file ./image.png --prompt "..." --json`                      | `--model`, görüntü destekli bir `<provider/model>` olmalıdır |
| Ses transkripsiyonu                | `openclaw infer audio transcribe --file ./memo.m4a --json`                                    | `--model`, `<provider/model>` olmalıdır                   |
| Konuşma sentezleme                 | `openclaw infer tts convert --text "..." --output ./speech.mp3 --json`                        | `tts status` yalnızca Gateway üzerinden çalışır              |
| Video oluşturma                    | `openclaw infer video generate --prompt "..." --json`                                         | `--resolution` gibi sağlayıcı ipuçlarını destekler             |
| Video dosyasını açıklama           | `openclaw infer video describe --file ./clip.mp4 --json`                                      | `--model`, `<provider/model>` olmalıdır                   |
| Web'de arama yapma                 | `openclaw infer web search --query "..." --json`                                              |                                                                    |
| Web sayfasını getirme              | `openclaw infer web fetch --url https://example.com --json`                                   |                                                                    |
| Gömme oluşturma                    | `openclaw infer embedding create --text "..." --json`                                         |                                                                    |

## Davranış

- Çıktı başka bir komuta veya betiğe aktarılacaksa `--json`, aksi takdirde metin çıktısı kullanın.
- Belirli bir arka ucu sabitlemek için `--provider` veya `--model provider/model` kullanın.
- Tek seferlik düşünme/akıl yürütme geçersiz kılması için `model run --thinking <level>` kullanın: `off`, `minimal`, `low`, `medium`, `high`, `adaptive`, `xhigh` veya `max`.
- `image describe`, `audio transcribe` ve `video describe` için `--model`, `<provider/model>` biçimini kullanmalıdır.
- `image describe` için `--file`, yerel yolları ve HTTP(S) URL'lerini kabul eder; uzak URL'ler normal medya getirme SSRF politikasından geçer.
- Durumsuz yürütme komutları (`model run`, `image *`, `audio *`, `video *`, `web *`, `embedding *`) varsayılan olarak yereldir. Gateway tarafından yönetilen durum komutları (`tts status`) varsayılan olarak Gateway'i kullanır.
- Yerel yol, Gateway'in çalışıyor olmasını hiçbir zaman gerektirmez.
- Yerel `model run`, yalın ve tek seferlik bir sağlayıcı tamamlamasıdır: yapılandırılmış ajan modelini ve kimlik doğrulamayı çözümler ancak bir sohbet ajanı turu başlatmaz, araçları yüklemez veya paketlenmiş MCP sunucularını açmaz.
- `model run --file`, görüntü dosyalarını (otomatik algılanan MIME türüyle) isteme ekler; birden fazla görüntü için `--file` seçeneğini yineleyin. Görüntü olmayan dosyalar reddedilir — bunun yerine `infer audio transcribe` veya `infer video describe` kullanın.
- `model run --gateway`; Gateway yönlendirmesini, kaydedilmiş kimlik doğrulamayı, sağlayıcı seçimini ve gömülü çalışma zamanını sınar ancak ham model yoklaması olarak kalır: önceki oturum transkripti, önyükleme/AGENTS bağlamı, araçlar veya paketlenmiş MCP sunucuları yoktur.
- `model run --gateway --model <provider/model>`, Gateway'den tek seferlik bir sağlayıcı/model geçersiz kılması çalıştırmasını istediği için güvenilir operatör Gateway kimlik bilgisi gerektirir.

## Model

Metin çıkarımı ve model/sağlayıcı incelemesi.

```bash
openclaw infer model run --prompt "Reply with exactly: smoke-ok" --json
openclaw infer model run --prompt "Summarize this changelog entry" --model openai/gpt-5.4 --json
openclaw infer model run --prompt "Describe this image in one sentence" --file ./photo.jpg --model google/gemini-2.5-flash --json
openclaw infer model run --prompt "Use more reasoning here" --thinking high --json
openclaw infer model providers --json
openclaw infer model inspect --model gpt-5.6-sol --json
```

Gateway'i başlatmadan veya ajan araç yüzeyini yüklemeden tek bir sağlayıcının hızlı çalışırlık testini yapmak için `--local` ile tam `<provider/model>` başvurularını kullanın:

```bash
openclaw infer model run --local --model anthropic/claude-sonnet-4-6 --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model cerebras/zai-glm-4.7 --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model google/gemini-2.5-flash --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model groq/llama-3.1-8b-instant --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model mistral/mistral-medium-3-5 --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model mistral/mistral-small-latest --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model openai/gpt-5.6-luna --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model ollama/qwen2.5vl:7b --prompt "Describe this image." --file ./photo.jpg --json
```

Notlar:

- Yerel `model run`, sağlayıcı/model/kimlik doğrulama durumuna yönelik en dar kapsamlı CLI çalışırlık testidir: ChatGPT-Codex dışındaki sağlayıcılara yalnızca verilen istemi gönderir.
- Yerel `model run --model <provider/model>`, sağlayıcı yapılandırmaya yazılmadan önce paketlenmiş statik katalogdaki tam satırları (`openclaw models list --all` tarafından gösterilen satırların aynısını) çözümleyebilir. Sağlayıcı kimlik doğrulaması yine de gereklidir; eksik kimlik bilgileri `Unknown model` olarak değil, kimlik doğrulama hataları olarak başarısız olur.
- Mistral Medium 3.5 akıl yürütme yoklamalarında sıcaklığı ayarlanmamış/varsayılan bırakın. Mistral, `reasoning_effort="high"` değerini `temperature: 0` ile reddeder; varsayılan sıcaklığı veya `0.7` gibi sıfır olmayan bir değeri kullanın.
- OpenAI ChatGPT/Codex OAuth (`openai-chatgpt-responses` API) yerel yoklamaları, aktarımın gerekli `instructions` alanını doldurabilmesi için asgari bir sistem talimatı ekler — tam ajan bağlamı, araçlar, bellek veya oturum transkripti eklemez.
- `model run --file`, görüntü içeriğini doğrudan tek kullanıcı iletisine ekler. MIME türü `image/*` olarak algılandığında yaygın biçimler (PNG, JPEG, WebP) çalışır; desteklenmeyen veya tanınmayan dosyalar sağlayıcı çağrılmadan önce başarısız olur. Doğrudan çok modlu model yoklaması yerine OpenClaw'ın görüntü modeli yönlendirmesini ve geri dönüşlerini kullanmak istediğinizde `infer image describe` kullanın.
- Seçilen model görüntü girdisini desteklemelidir; yalnızca metin destekleyen modeller isteği sağlayıcı katmanında reddedebilir.
- `model run --prompt`, boşluk olmayan metin içermelidir; boş istemler herhangi bir sağlayıcı veya Gateway çağrısından önce reddedilir.
- Sağlayıcı metin çıktısı döndürmediğinde yerel `model run` sıfır olmayan bir kodla çıkar; böylece erişilemeyen sağlayıcılar ve boş tamamlamalar başarılı yoklamalar gibi görünmez.
- Model girdisini ham tutarken Gateway yönlendirmesini veya ajan çalışma zamanı kurulumunu test etmek için `model run --gateway` kullanın. Tam ajan bağlamı, araçlar, bellek ve oturum transkripti için `openclaw agent` veya bir sohbet yüzeyi kullanın.
- `--thinking adaptive`, tamamlama çalışma zamanı düzeyindeki `medium` değerine eşlenir; `--thinking max`, yerel azami eforu destekleyen OpenAI modellerinde `max` değerine, aksi takdirde `xhigh` değerine eşlenir.
- `model auth login`, `model auth logout` ve `model auth status`, kaydedilmiş sağlayıcı kimlik doğrulama durumunu yönetir.

## Görüntü

Oluşturma, düzenleme ve açıklama.

```bash
openclaw infer image generate --prompt "friendly lobster illustration" --json
openclaw infer image generate --prompt "cinematic product photo of headphones" --json
openclaw infer image generate --model openai/gpt-image-1.5 --output-format png --background transparent --prompt "simple red circle sticker on a transparent background" --json
openclaw infer image generate --model openai/gpt-image-2 --quality low --openai-moderation low --prompt "low-cost draft poster" --json
openclaw infer image generate --prompt "slow image backend" --timeout-ms 180000 --json
openclaw infer image edit --file ./logo.png --model openai/gpt-image-1.5 --output-format png --background transparent --prompt "keep the logo, remove the background" --json
openclaw infer image edit --file ./poster.png --prompt "make this a vertical story ad" --size 2160x3840 --aspect-ratio 9:16 --resolution 4K --json
openclaw infer image describe --file ./photo.jpg --json
openclaw infer image describe --file https://example.com/photo.png --json
openclaw infer image describe --file ./receipt.jpg --prompt "Extract the merchant, date, and total" --json
openclaw infer image describe-many --file ./before.png --file ./after.png --prompt "Compare the screenshots and list visible UI changes" --json
openclaw infer image describe --file ./ui-screenshot.png --model openai/gpt-5.4-mini --json
openclaw infer image describe --file ./photo.jpg --model ollama/qwen2.5vl:7b --prompt "Describe the image in one sentence" --timeout-ms 300000 --json
```

Notlar:

- Mevcut girdi dosyalarından başlarken `image edit` kullanın; `--size`, `--aspect-ratio` veya `--resolution`, bunları destekleyen sağlayıcılara/modellere geometri ipuçları ekler.
- `--output-format png --background transparent` ile `--model openai/gpt-image-1.5`, şeffaf arka planlı OpenAI PNG çıktısı sağlar; `--openai-background`, aynı ipucu için OpenAI'ye özgü bir diğer addır. Arka plan desteği bildirmeyen sağlayıcılar bunu yok sayılan bir geçersiz kılma olarak raporlar ([JSON zarfındaki](#json-output) `ignoredOverrides` bölümüne bakın).
- `--quality low|medium|high|auto`, OpenAI dahil olmak üzere görüntü kalitesi ipuçlarını destekleyen sağlayıcılarda çalışır. OpenAI ayrıca `--openai-moderation low|auto` değerini de kabul eder.
- `image providers --json`, paketle birlikte gelen görüntü sağlayıcılarından hangilerinin keşfedilebilir, yapılandırılmış ve seçili olduğunu ve her birinin hangi üretme/düzenleme yeteneklerini sunduğunu listeler.
- `image generate --model <provider/model> --json`, görüntü üretme değişiklikleri için en dar kapsamlı canlı duman testidir:

  ```bash
  openclaw infer image providers --json
  openclaw infer image generate \
    --model google/gemini-3.1-flash-image \
    --prompt "Beyaz arka plan üzerinde tek bir mavi kare bulunan, metinsiz, minimal düz test görüntüsü." \
    --output ./openclaw-infer-image-smoke.png \
    --json
  ```

  Yanıt; `ok`, `provider`, `model`, `attempts` ve yazılan çıktı yollarını bildirir. `--output` ayarlandığında son uzantı, sağlayıcının döndürdüğü MIME türüne göre belirlenebilir.

- `image describe` ve `image describe-many` için göreve özgü bir talimat (OCR, karşılaştırma, kullanıcı arayüzü incelemesi, kısa açıklama) vermek üzere `--prompt` kullanın.
- Yavaş yerel görsel modeller veya soğuk Ollama başlangıçları için `--timeout-ms` kullanın.
- `image describe` için önce açıkça belirtilen bir `--model` (görüntü özellikli bir `<provider/model>` olmalıdır) çalıştırılır, ardından bu çağrı başarısız olursa yapılandırılmış `agents.defaults.imageModel.fallbacks` denenir. Girdi hazırlama hataları (eksik dosya, desteklenmeyen URL) herhangi bir geri dönüş denemesinden önce başarısız olur ve model, model kataloğunda veya sağlayıcı yapılandırmasında görüntü özellikli olmalıdır.
- Yerel Ollama görsel modelleri için önce modeli çekin ve `OLLAMA_API_KEY` değerini herhangi bir yer tutucu değere, örneğin `ollama-local` değerine ayarlayın. [Ollama](/tr/providers/ollama#vision-and-image-description) bölümüne bakın.

## Ses

Dosya transkripsiyonu (gerçek zamanlı oturum yönetimi değil).

```bash
openclaw infer audio transcribe --file ./memo.m4a --json
openclaw infer audio transcribe --file ./team-sync.m4a --language en --prompt "Adlara ve eylem öğelerine odaklan" --json
openclaw infer audio transcribe --file ./memo.m4a --model openai/whisper-1 --json
```

`--model`, `<provider/model>` olmalıdır.

## TTS

Konuşma sentezi ve TTS sağlayıcı/persona durumu.

```bash
openclaw infer tts convert --text "openclaw'dan merhaba" --output ./hello.mp3 --json
openclaw infer tts convert --text "Derlemeniz tamamlandı" --output ./build-complete.mp3 --json
openclaw infer tts providers --json
openclaw infer tts personas --json
openclaw infer tts status --json
```

Notlar:

- `tts status` yalnızca `--gateway` değerini destekler (Gateway tarafından yönetilen TTS durumunu yansıtır).
- TTS davranışını incelemek ve yapılandırmak için `tts providers`, `tts voices`, `tts personas`, `tts set-provider` ve `tts set-persona` kullanın.

## Video

Üretme ve açıklama.

```bash
openclaw infer video generate --prompt "okyanus üzerinde sinematik gün batımı" --json
openclaw infer video generate --prompt "bir orman gölü üzerinde yavaş drone çekimi" --resolution 768P --duration 6 --json
openclaw infer video describe --file ./clip.mp4 --json
openclaw infer video describe --file ./clip.mp4 --model openai/gpt-5.4-mini --json
```

Notlar:

- `video generate`; video üretme çalışma zamanına iletilen `--size`, `--aspect-ratio`, `--resolution`, `--duration`, `--audio`, `--watermark` ve `--timeout-ms` değerlerini kabul eder.
- `video describe` için `--model`, `<provider/model>` olmalıdır.

## Web

Arama ve getirme.

```bash
openclaw infer web search --query "OpenClaw belgeleri" --json
openclaw infer web search --query "OpenClaw infer web sağlayıcıları" --json
openclaw infer web fetch --url https://docs.openclaw.ai/cli/infer --json
openclaw infer web providers --json
```

`web providers`, arama ve getirme için kullanılabilir, yapılandırılmış ve seçili sağlayıcıları listeler.

## Gömme

Vektör oluşturma ve gömme sağlayıcısını inceleme.

```bash
openclaw infer embedding create --text "arkadaş canlısı ıstakoz" --json
openclaw infer embedding create --text "müşteri destek kaydı: geciken gönderi" --model openai/text-embedding-3-large --json
openclaw infer embedding providers --json
```

## JSON çıktısı

Infer komutları, JSON çıktısını paylaşılan bir zarf altında normalleştirir:

```json
{
  "ok": true,
  "capability": "image.generate",
  "transport": "local",
  "provider": "openai",
  "model": "gpt-image-2",
  "attempts": [],
  "outputs": []
}
```

Kararlı üst düzey alanlar:

- `ok`
- `capability`
- `transport`
- `provider`
- `model`
- `attempts`
- `inputs` (uygun olduğunda istekle birlikte gönderilen görüntü ekleri)
- `outputs`
- `ignoredOverrides` (uygun olduğunda sağlayıcının desteklemediği ipucu anahtarları)
- `error`

Üretilen medya komutları için `outputs`, OpenClaw tarafından yazılan dosyaları içerir. Otomasyon için insan tarafından okunabilir stdout çıktısını ayrıştırmak yerine bu dizideki `path`, `mimeType`, `size` ve medyaya özgü boyutları kullanın.

## Yaygın sorunlar

```bash
# Kötü
openclaw infer media image generate --prompt "arkadaş canlısı ıstakoz"

# İyi
openclaw infer image generate --prompt "arkadaş canlısı ıstakoz"
```

```bash
# Kötü
openclaw infer audio transcribe --file ./memo.m4a --model whisper-1 --json

# İyi
openclaw infer audio transcribe --file ./memo.m4a --model openai/whisper-1 --json
```

## İlgili

- [CLI başvurusu](/tr/cli)
- [Modeller](/tr/concepts/models)
