---
read_when:
    - Ses transkripsiyonunu veya medya işlemeyi değiştirme
summary: Gelen sesli mesajların nasıl indirildiği, yazıya döküldüğü ve yanıtlara eklendiği
title: Ses ve sesli notlar
x-i18n:
    generated_at: "2026-07-26T22:50:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4076e3e55eb5c6dcc94cfdd842619697c8d756b924956d7b266d18446b4dd9be
    source_path: nodes/audio.md
    workflow: 16
---

## Ne yapar

Ses anlama etkinleştirildiğinde (veya otomatik olarak algılandığında), OpenClaw:

1. İlk ses ekini (yerel yol veya URL) bulur ve gerekirse indirir.
2. Her model girdisine göndermeden önce `maxBytes` sınırını uygular.
3. Sırayla uygun olan ilk model girdisini (sağlayıcı veya CLI) çalıştırır; bir girdi başarısız olursa veya atlanırsa (boyut/zaman aşımı), sonraki girdi denenir.
4. Başarılı olduğunda `Body` değerini bir `[Audio]` bloğuyla değiştirir ve `{{Transcript}}` değerini ayarlar.

Transkripsiyon başarılı olduğunda, eğik çizgi komutlarının çalışmaya devam etmesi için `CommandBody`/`RawBody` değerleri de transkript olarak ayarlanır. `--verbose` kullanıldığında günlükler, transkripsiyonun ne zaman çalıştığını ve gövdenin ne zaman değiştirildiğini gösterir.

## Otomatik algılama (varsayılan)

Modelleri yapılandırmadıysanız ve `tools.media.audio.enabled`, `false` değilse OpenClaw aşağıdaki sırayla otomatik algılama yapar ve çalışan ilk seçenekte durur:

1. Sağlayıcısı ses anlamayı destekliyorsa **etkin yanıt modeli**.
2. **Yapılandırılmış sağlayıcı kimlik doğrulaması** — ses transkripsiyonunu destekleyen bir sağlayıcı için kimlik doğrulaması kullanılabilir olan herhangi bir `models.providers.*` girdisi. Bu, yerel CLI'lerden önce denetlenir; dolayısıyla yapılandırılmış bir API anahtarı, `PATH` üzerindeki yerel ikili dosyaya her zaman üstün gelir.
   Birden fazla sağlayıcı yapılandırıldığındaki öncelik: Groq, OpenAI, xAI, Deepgram, Google, SenseAudio, ElevenLabs, Mistral.
3. **Yerel CLI'ler** (yalnızca sağlayıcı kimlik doğrulaması çözümlenmediyse). OpenClaw sıralı bir geri dönüş listesi oluşturur:
   - Yalnızca geçerli süreçteki daha önceki bir model çağrısı Metal veya CUDA gözlemlediyse CPU varsayılanlarından önce `whisper-cli`
   - Varsayılan CPU sağlayıcısında `sherpa-onnx-offline` (`tokens.txt`, `encoder.onnx`, `decoder.onnx` ve `joiner.onnx` ile `SHERPA_ONNX_MODEL_DIR` gerektirir)
   - Metal/CUDA yalnızca derleme özelliğine sahip olduğunda veya seçili arka uç başka şekilde gözlemlenmediğinde `whisper-cli`
   - Apple Silicon üzerinde `parakeet-mlx` (MLX özellikli; cihaz kullanımı gözlemlenmemiş olarak kalır)
   - `whisper` (Python CLI; modelleri otomatik olarak indirir)

Kurulum/bağlantı kaynağı, yürütme kanıtı değil yetenek kanıtıdır. Tek başına hiçbir zaman bir adayı CPU sherpa'nın önüne taşımaz. OpenClaw, yalnızca bir arka ucu yoklamak için kurulum veya durum denetimleri sırasında model yüklemez.
Otomatik algılanan whisper.cpp, OpenClaw'ın yukarı akış `using … backend` satırını kaydedebilmesi için normal model çalıştırma günlüklerini etkin tutar. Açık CLI girdileri, yapılandırılmış çıktı bayraklarını korur.

Medya anlama için Gemini CLI otomatik algılamasının yerini, görüntü/video için korumalı alanda çalışan Antigravity CLI (`agy`) geri dönüşü almıştır; ses, yukarıdaki yerel ikili dosyaların ötesinde bir CLI geri dönüşü kullanmaz.

Otomatik algılamayı devre dışı bırakmak için `tools.media.audio.enabled: false` değerini ayarlayın. Özelleştirmek için `tools.media.models` alanına yetenek etiketli girdiler ekleyin.

<Note>
İkili dosya algılama macOS/Linux/Windows genelinde en iyi çabayla gerçekleştirilir. CLI'nin `PATH` üzerinde olduğundan (`~` genişletilir) emin olun veya tam komut yoluyla açık bir CLI modeli ayarlayın.
</Note>

Ses transkripsiyonu yapmadan yerel seçimi inceleyin:

```bash
openclaw capability audio providers
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

Sağlayıcı envanteri, yerel geri dönüş kazananını genel sağlayıcı seçiminden ayrı olarak ve bunun yanında kullanılabilir, istenen ve gözlemlenen arka uç alanlarını bildirir. Transkripsiyon çalıştıktan sonra `/status`, medya satırında istenen veya gözlemlenen arka ucu bildirir. Açıkça ses özelliğine sahip `tools.media.models` CLI girdileri yine de otomatik seçimi atlar; sherpa `--provider=cuda` veya whisper.cpp `--no-gpu`/`--device` gibi arka uca özgü bayraklarını kullanın.

## Yapılandırma örnekleri

### Sağlayıcı + CLI geri dönüşü (OpenAI + Whisper CLI)

```json5
{
  tools: {
    media: {
      models: [
        { provider: "openai", model: "gpt-4o-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          timeoutSeconds: 45,
          capabilities: ["audio"],
        },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-transcribe" },
    },
  },
}
```

### Yalnızca sağlayıcı (Deepgram)

```json5
{
  tools: {
    media: {
      models: [{ provider: "deepgram", model: "nova-3", capabilities: ["audio"] }],
      audio: { enabled: true },
    },
  },
}
```

### Yalnızca sağlayıcı (Mistral Voxtral)

```json5
{
  tools: {
    media: {
      models: [{ provider: "mistral", model: "voxtral-mini-latest", capabilities: ["audio"] }],
      audio: { enabled: true },
    },
  },
}
```

### Yalnızca sağlayıcı (SenseAudio)

```json5
{
  tools: {
    media: {
      models: [
        {
          provider: "senseaudio",
          model: "senseaudio-asr-pro-1.5-260319",
          capabilities: ["audio"],
        },
      ],
      audio: { enabled: true },
    },
  },
}
```

### Transkripti sohbete yansıtma (isteğe bağlı)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        echoTranscript: true,
        echoFormat: '📝 "{transcript}"',
      },
    },
  },
}
```

## Notlar ve sınırlar

- Sağlayıcı kimlik doğrulaması standart model kimlik doğrulama sırasını izler (kimlik doğrulama profilleri, ortam değişkenleri, `models.providers.*.apiKey`).
- Groq kurulum ayrıntıları: [Groq](/tr/providers/groq).
- `provider: "deepgram"` kullanıldığında Deepgram, `DEEPGRAM_API_KEY` değerini alır. Kurulum ayrıntıları: [Deepgram](/tr/providers/deepgram).
- Mistral kurulum ayrıntıları: [Mistral](/tr/providers/mistral).
- `provider: "senseaudio"` kullanıldığında SenseAudio, `SENSEAUDIO_API_KEY` değerini alır. Kurulum ayrıntıları: [SenseAudio](/tr/providers/senseaudio).
- Ses sağlayıcıları `tools.media.audio` altındaki varsayılanları kullanabilir veya `tools.media.models[]` girdilerindeki `baseUrl`, `headers`, `providerOptions` ve sınırları geçersiz kılabilir.
- Yerleşik ses boyutu sınırı 20MB'tır. Girdi düzeyindeki `maxBytes` geçersiz kılması bunu değiştirebilir; sınırı aşan ses o model için atlanır ve sonraki girdi denenir.
- 1024 bayttan küçük ses dosyaları, sağlayıcı/CLI transkripsiyonundan önce atlanır.
- Ses için varsayılan `maxChars` **ayarlanmamıştır** (tam transkript). Çıktıyı kısaltmak için `tools.media.audio.maxChars` veya girdi başına `maxChars` ayarlayın.
- OpenAI otomatik algılama varsayılanı `gpt-4o-transcribe`; daha ucuz/hızlı bir seçenek için `model: "gpt-4o-mini-transcribe"` ayarlayın.
- Transkript, şablonlarda `{{Transcript}}` olarak kullanılabilir.
- `tools.media.audio.echoTranscript` varsayılan olarak kapalıdır; `echoFormat`, bir `{transcript}` yer tutucusunu kabul eder.
- CLI standart çıktısı 5MB ile sınırlıdır; CLI çıktısını kısa tutun.
- CLI `args`, yerel ses dosyası yolu için `{{AttachmentPath}}` kullanmalıdır. Eski `audio.transcription.command` yapılandırmalarındaki kullanımdan kaldırılmış `{input}` yer tutucularını taşımak için `openclaw doctor --fix` çalıştırın (kullanımdan kaldırılan anahtar: `audio.transcription`, yerine `tools.media.models` getirildi). `{{MediaPath}}`, kullanımdan kaldırılmış bir uyumluluk diğer adı olarak kalır.
- `tools.media.concurrency`, medya görevlerini sınırlar; bir GPU zamanlayıcısı değildir.

### Yerleşik yerel STT

Otomatik algılanan yerel STT, istek başına süreç olarak çalışmaya devam eder. OpenClaw şu anda yerleşik bir whisper.cpp sunucusunu yönetmez; çünkü standart Homebrew `whisper-cpp` paketi bu sunucuyu devre dışı bırakırken yukarı akış örneğinde yapılandırılmış sınırlı bir kabul kuyruğu yoktur. Plugin tarafından yönetilen yerleşik bir yaşam döngüsünün güvenle etkinleştirilebilmesi için durum/başlangıç denetimine, modelin bellekte tutulmasına, sınırlı kuyruğa alma işlemine, iptal/zaman aşımına, yalnızca geri döngü üzerinden kimlik doğrulamasız çalışmaya ve bulut geri dönüşünün bulunmamasına sahip, bakımı yapılan paketlenmiş bir çalışan gerekir.

### Proxy ortamı desteği

Sağlayıcı tabanlı ses transkripsiyonu, undici'nin `EnvHttpProxyAgent` semantiğiyle eşleşen standart giden proxy ortam değişkenlerini dikkate alır:

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `ALL_PROXY` / `all_proxy`

Küçük harfli değişkenler büyük harfli değişkenlere göre önceliklidir; `NO_PROXY`/`no_proxy` girdileri (ana bilgisayar adları, `*.suffix` veya `host:port`) proxy'yi atlar. Hiçbir proxy ortam değişkeni ayarlanmamışsa doğrudan dış bağlantı kullanılır. Proxy kurulumu başarısız olursa (hatalı biçimlendirilmiş URL), OpenClaw bir uyarıyı günlüğe kaydeder ve doğrudan getirmeye geri döner.

## Gruplarda bahsetme algılama

Ses ön kontrolünü destekleyen kanallarda OpenClaw, bir grup sohbeti için `requireMention: true` ayarlandığında bahsetmeleri denetlemeden **önce** ses transkripsiyonu yapar. Bu, başlıksız bir sesli notun transkripti yapılandırılmış bir bahsetme kalıbı içerdiğinde bahsetme geçidinden geçmesini sağlar. Kanala özgü belgeler, yazılı bir bahsetme gerektiren aktarımları açıklar.

**Nasıl çalışır:**

1. Bir sesli mesajın metin gövdesi yoksa ve grup bahsetme gerektiriyorsa OpenClaw, ilk ses ekinin ön kontrol transkripsiyonunu gerçekleştirir.
2. Transkript, bahsetme kalıpları (örneğin `@BotName`, emoji tetikleyicileri) bakımından denetlenir.
3. Bir bahsetme bulunursa mesaj, tam yanıt işlem hattından geçer.

**Geri dönüş davranışı:** Ön kontrol transkripsiyonu başarısız olursa (zaman aşımı, API hatası vb.), karma mesajların (metin + ses) hiçbir zaman bırakılmaması için mesaj yalnızca metne dayalı bahsetme algılamasına geri döner.

**Telegram grubu/konusu başına devre dışı bırakma:**

- Bu grup için ön kontrol transkriptindeki bahsetme denetimlerini atlamak üzere `channels.telegram.groups.<chatId>.disableAudioPreflight: true` ayarlayın.
- Konu başına geçersiz kılmak için `channels.telegram.groups.<chatId>.topics.<threadId>.disableAudioPreflight` ayarlayın (atlamak için `true`, etkinleştirmeye zorlamak için `false`).
- Varsayılan değer `false` şeklindedir (bahsetme geçidi koşulları eşleştiğinde ön kontrol etkindir).

**Örnek:** Bir kullanıcı, `requireMention: true` bulunan bir Telegram grubunda "Hey @Claude, hava nasıl?" diyen bir sesli not gönderir. Sesli not yazıya dökülür, bahsetme algılanır ve ajan yanıt verir.

## Dikkat edilmesi gerekenler

- Kapsam kurallarında ilk eşleşme kazanır; `chatType`, `direct`, `group` veya `channel` olarak normalleştirilir.
- CLI'nizin 0 koduyla çıktığından ve düz metin yazdırdığından emin olun; JSON çıktısının `jq -r .text` aracılığıyla işlenmesi gerekir.
- Bilinen dosya çıktısı modları belirleyicidir: çıkarımlanan transkript dosyasının boş veya eksik olması, CLI ilerleme çıktısına geri dönmek yerine transkript üretilmemesine neden olur.
- `parakeet-mlx` için, `--output-dir` ve varsayılan `{filename}` çıktı şablonuyla birlikte `--output-format txt` (veya `all`) kullanın. Yukarı akış `PARAKEET_OUTPUT_FORMAT` ve `PARAKEET_OUTPUT_TEMPLATE` ortam değişkenleri de dikkate alınır. OpenClaw, `<output-dir>/<media-basename>.txt` değerini okur; varsayılan `srt` biçimi, diğer biçimler ve özel çıktı şablonları standart çıktıyı kullanmaya devam eder.
- Yanıt kuyruğunun engellenmesini önlemek için zaman aşımlarını makul tutun (`timeoutSeconds`, varsayılan 60s).
- Ön kontrol transkripsiyonu, bahsetme algılaması için yalnızca **ilk** ses ekini işler. Ek ses dosyaları ana medya anlama aşamasında işlenir.

## İlgili

- [Medya anlama](/tr/nodes/media-understanding)
- [Konuşma modu](/tr/nodes/talk)
- [Sesle uyandırma](/tr/nodes/voicewake)
