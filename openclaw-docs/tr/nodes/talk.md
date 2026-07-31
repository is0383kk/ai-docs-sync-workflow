---
read_when:
    - macOS/iOS/Android'de Konuşma modunu uygulama
    - Ses/TTS/kesme davranışını değiştirme
summary: 'Konuşma modu: yerel STT/TTS ve gerçek zamanlı ses üzerinden kesintisiz sesli konuşmalar'
title: Konuşma modu
x-i18n:
    generated_at: "2026-07-26T22:51:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b21319eee169ba898331f87279a2b2a5170441131a1e9cdc85c15b268d165e21
    source_path: nodes/talk.md
    workflow: 16
---

Konuşma modu beş çalışma zamanı biçimini kapsar:

- **Yerel macOS/iOS/Android Konuşması**: yerel konuşma tanıma, Gateway sohbeti ve `talk.speak` TTS. macOS/iOS'taki Apple Speech tanıma ağ hizmetlerini kullanabilir; Android davranışı, yüklü konuşma hizmetine bağlıdır. Node'lar `talk` yeteneğini duyurur ve hangi `talk.*` komutlarını desteklediklerini bildirir.
- **iOS Konuşması (gerçek zamanlı)**: `webrtc` aktarımını seçen veya aktarımı belirtmeyen OpenAI gerçek zamanlı yapılandırmaları için istemciye ait WebRTC. Açıkça belirtilen `gateway-relay`, `provider-websocket` ve OpenAI dışı gerçek zamanlı yapılandırmalar Gateway'e ait aktarıcıda kalır; gerçek zamanlı olmayan yapılandırmalar yerel konuşma döngüsünü kullanır.
- **Tarayıcı Konuşması**: istemciye ait `webrtc`/`provider-websocket` oturumları için `talk.client.create` veya Gateway'e ait `gateway-relay` oturumları için `talk.session.create`. `managed-room`, Gateway devri ve bas-konuş odaları için ayrılmıştır.
- **Android Konuşması (gerçek zamanlı)**: `talk.realtime.mode: "realtime"` ve `talk.realtime.transport: "gateway-relay"` ile etkinleştirin. Aksi takdirde Android; yerel konuşma tanıma, Gateway sohbeti ve `talk.speak` kullanmaya devam eder.
- **Yalnızca transkripsiyon istemcileri**: asistanın sesli yanıtı olmadan altyazılar/dikte için `talk.session.create({ mode: "transcription", transport: "gateway-relay", brain: "none" })`, ardından `talk.session.appendAudio`, `talk.session.cancelTurn` ve `talk.session.close`. Tek seferlik yüklenen sesli notlar yine [medya anlama](/tr/nodes/media-understanding) ses yolunu kullanır.

Yerel Konuşma sürekli bir döngüdür: konuşmayı dinler, transkripti etkin oturum üzerinden modele gönderir, yanıtı bekler ve ardından yapılandırılmış Konuşma sağlayıcısı (`talk.speak`) aracılığıyla seslendirir.

İstemciye ait gerçek zamanlı Konuşma, sağlayıcı araç çağrılarını doğrudan `chat.send` çağırmak yerine `talk.client.toolCall` üzerinden iletir. Gerçek zamanlı bir danışma etkinken istemciler, sözlü girdiyi `status`, `steer`, `cancel` veya `followup` olarak sınıflandırmak için `talk.client.steer` ya da `talk.session.steer` çağırabilir. Kabul edilen yönlendirme, etkin gömülü çalıştırmanın kuyruğuna eklenir; reddedilen yönlendirme `no_active_run`, `not_streaming` veya `compacting` gibi bir neden döndürür.

Sonlandırılan gerçek zamanlı kullanıcı ve asistan ifadeleri her zaman etkin ajan oturumuna anlık olarak eklenir; böylece sonraki sohbet ve sesli konuşma sıraları aynı geçmişi paylaşır. İstemciye ait aktarımlar, sonlandırılmış transkriptlerini kararlı girdi kimlikleriyle bildirir; Gateway aktarıcı oturumları aynı olayları sunucu tarafında ekler. Sağlayıcı oturumları, Discord sesinin kullandığı sınırlı gerçek zamanlı profil bağlamını da alır.

Ses kaynaklı danışma çalıştırmaları; mesaj gönderme, Node'ları denetleme, tarayıcı/bilgisayar eylemleri, hizmet değişiklikleri, yıkıcı kabuk komutları veya yayımlama gibi yüksek etkili eylemlerden önce yeni ve birebir sözlü onay gerektirir. Onay yalnızca engellenen araç bağımsız değişkenlerinin tam olarak eşleşen hâli için geçerlidir ve bir kez kullanılır; ilgisiz eş zamanlı çalıştırmalar etkilenmez. Bir çağrı kapandığında OpenClaw, değişiklik yapan araçlar için kısa bir **Sesli çağrı değişiklikleri** özetini oturumun WebChat dışındaki son teslim hedefine gönderebilir.

Yalnızca transkripsiyon Konuşması, gerçek zamanlı ve STT/TTS oturumlarıyla aynı Konuşma olay zarfını yayar ancak `mode: "transcription"` ve `brain: "none"` kullanır. Tüm Konuşma oturumları olayları `talk.event` kanalında yayınlar; istemciler kısmi/nihai transkript güncellemeleri (`transcript.delta`/`transcript.done`) ve diğer oturum telemetrisi için bu kanala abone olur.

Tarayıcı Görüntülü Konuşma, OpenAI Realtime WebRTC ve Google Live
sağlayıcı-WebSocket oturumları için kullanılabilir. `describe_view`
görsel bağlam istediğinde OpenAI tek bir sınırlı JPEG alır; sürekli bir
kamera akışı almaz. Google Live, sınırlı JPEG karelerini doğrudan
tarayıcıdan saniyede en fazla bir kare hızında alırken `describe_view`
kamera akışı durumunu bildirir. Her iki durumda da kamera kareleri Gateway'i
atlayarak iletilir ve Konuşma'nın durdurulması kamera ve mikrofon akışlarını serbest bırakır.

## Davranış (macOS)

- Konuşma modu etkinken her zaman açık katman.
- **Dinleme &rarr; Düşünme &rarr; Konuşma** aşama geçişleri.
- Kısa bir duraklamada (sessizlik aralığı) mevcut transkript gönderilir.
- Yanıtlar WebChat'e yazılır (yazmayla aynı).
- **Konuşmayla kesme** (varsayılan olarak açık): asistan konuşurken kullanıcı konuşursa oynatma durur ve kesinti zaman damgası sonraki istem için kaydedilir.

## Yanıtlardaki ses yönergeleri

Asistan, sesi denetlemek için yanıtın başına tek bir JSON satırı ekleyebilir:

```json
{ "voice": "<voice-id>", "once": true }
```

Kurallar:

- Yalnızca boş olmayan ilk satır; JSON satırı TTS oynatımından önce kaldırılır.
- Bilinmeyen anahtarlar yok sayılır.
- `once: true` yalnızca mevcut yanıta uygulanır; bu olmadan ses, yeni Konuşma modu varsayılanı olur.

Desteklenen anahtarlar: `voice` / `voice_id` / `voiceId`, `model` / `model_id` / `modelId`, `speed`, `rate` (dakika başına kelime), `stability`, `similarity`, `style`, `speakerBoost`, `seed`, `normalize`, `lang`, `output_format`, `latency_tier`, `once`.

## Yapılandırma (`~/.openclaw/openclaw.json`)

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "elevenlabs_voice_id",
        modelId: "eleven_v3",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
    realtime: {
      provider: "openai",
      providers: {
        openai: {
          apiKey: "openai_api_key",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
      instructions: "Sıcak bir üslupla konuş ve yanıtları kısa tut.",
      mode: "realtime",
      transport: "webrtc",
      brain: "agent-consult",
    },
  },
}
```

| Anahtar                                  | Varsayılan                                 | Notlar                                                                                                                                                                                                                                                                     |
| ---------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`                               | -                                          | Active Talk TTS sağlayıcısı. macOS yerel oynatma yolları için `elevenlabs`, `mlx` veya `system` kullanın.                                                                                                                                                                             |
| `providers.<id>.voiceId`                 | -                                          | ElevenLabs, `ELEVENLABS_VOICE_ID` / `SAG_VOICE_ID` değerlerine veya API anahtarı bulunan ilk kullanılabilir sese geri döner.                                                                                                                                                             |
| `speechLocale`                           | cihaz varsayılanı                          | Android, iOS ve macOS yerel konuşma tanıma için BCP 47 yerel ayarı. Apple Speech ağ hizmetlerini kullanabilir; Android ayrıca dil bileşenini gerçek zamanlı giriş transkripsiyonuna iletir.                                                                                  |
| `providers.elevenlabs.modelId`           | `eleven_v3`                                |                                                                                                                                                                                                                                                                            |
| `providers.mlx.modelId`                  | `mlx-community/Soprano-80M-bf16`           |                                                                                                                                                                                                                                                                            |
| `providers.elevenlabs.apiKey`            | -                                          | `ELEVENLABS_API_KEY` değerine (veya varsa Gateway kabuk profiline) geri döner.                                                                                                                                                                                                |
| `silenceTimeoutMs`                       | `700` ms macOS/Android, `900` ms iOS       | Talk transkripsiyonu göndermeden önceki duraklama aralığı.                                                                                                                                                                                                                             |
| `interruptOnSpeech`                      | `true`                                     |                                                                                                                                                                                                                                                                            |
| `outputFormat`                           | `pcm_44100` macOS/iOS, `pcm_24000` Android | MP3 akışını zorlamak için `mp3_*` değerini ayarlayın.                                                                                                                                                                                                                                        |
| `consultThinkingLevel`                   | ayarlanmamış                              | Gerçek zamanlı `openclaw_agent_consult` çağrılarının arkasındaki ajan çalıştırması için düşünme düzeyi geçersiz kılma değeri.                                                                                                                                                                                  |
| `consultFastMode`                        | ayarlanmamış                              | Gerçek zamanlı `openclaw_agent_consult` çağrıları için hızlı mod geçersiz kılma değeri.                                                                                                                                                                                                            |
| `realtime.provider`                      | -                                          | WebRTC için `openai`, sağlayıcı WebSocket'i için `google` veya Gateway aktarımı üzerinden yalnızca köprü sağlayıcısı.                                                                                                                                                                     |
| `realtime.providers.<id>`                | -                                          | Sağlayıcının yönettiği gerçek zamanlı yapılandırma. Tarayıcılar yalnızca geçici/kısıtlı oturum kimlik bilgilerini alır; standart bir API anahtarını asla almaz.                                                                                                                                                 |
| `realtime.providers.openai.speakerVoice` | `alloy`                                    | Yerleşik OpenAI Realtime ses kimliği (eski `voice` anahtarı hâlâ çalışır ancak kullanımdan kaldırılmıştır). Güncel `gpt-realtime-2.1` sesleri: `alloy`, `ash`, `ballad`, `cedar`, `coral`, `echo`, `marin`, `sage`, `shimmer`, `verse`; en iyi kalite için `marin` ve `cedar` önerilir. |
| `realtime.transport`                     | -                                          | `webrtc`: iOS'ta ve tarayıcıda istemcinin yönettiği OpenAI WebRTC. `provider-websocket`: tarayıcının yönettiği, iOS'ta Gateway aktarımında kalır. `gateway-relay`: sağlayıcı sesini Gateway üzerinde tutar; Android gerçek zamanlı özelliği yalnızca bu aktarımla kullanır.                                  |
| `realtime.brain`                         | -                                          | `agent-consult`, gerçek zamanlı araç çağrılarını Gateway politikası üzerinden yönlendirir; `direct-tools`, eski doğrudan araç uyumluluğudur; `none`, transkripsiyon/harici orkestrasyon içindir.                                                                                                 |
| `realtime.consultRouting`                | -                                          | `provider-direct`, sağlayıcı `openclaw_agent_consult` adımını atladığında doğrudan yanıtını korur; `force-agent-consult` ise sonlandırılmış kullanıcı transkripsiyonlarını OpenClaw üzerinden yönlendirir.                                                                                          |
| `realtime.instructions`                  | -                                          | OpenClaw'ın yerleşik gerçek zamanlı istemine sağlayıcıya yönelik sistem talimatları ekler (ses stili/tonu); varsayılan `openclaw_agent_consult` yönlendirmesi korunur.                                                                                                                |

`talk.catalog`, standart sağlayıcı kimliklerini ve kayıt defteri diğer adlarını, her sağlayıcının geçerli modlarını/aktarımlarını/beyin stratejilerini/gerçek zamanlı ses biçimlerini/yetenek bayraklarını ve çalışma zamanı tarafından seçilen hazır olma sonucunu sunar. Birinci taraf Talk istemcileri, sağlayıcı diğer adlarını yerel olarak yönetmek yerine bu kataloğu okumalıdır; grup hazır olma durumunu içermeyen eski bir Gateway'i kesin biçimde yapılandırılmamış değil, doğrulanmamış olarak değerlendirin. Akışlı transkripsiyon sağlayıcıları `talk.catalog.transcription` üzerinden keşfedilir; mevcut Gateway aktarımı, özel bir Talk transkripsiyon yapılandırma yüzeyi sunulana kadar Voice Call akış sağlayıcısı yapılandırmasını kullanır.

## macOS kullanıcı arayüzü

- Menü çubuğu geçiş düğmesi: **Talk**
- Yapılandırma sekmesi: **Talk Mode** grubu (ses kimliği + kesme geçiş düğmesi)
- Katman: küre, evrensel konuşma dalga biçimini oluşturur (iOS, watchOS ve Android ile ortaktır). Dinleme, canlı mikrofon düzeyini izler; Konuşma, gerçek TTS oynatma zarfını izler; Düşünme, hafifçe nefes alır. Duraklatmak/sürdürmek için küreye tıklayın, konuşmayı durdurmak için çift tıklayın, Talk modundan çıkmak için X'e tıklayın.

## Android kullanıcı arayüzü

- Android'in ana gezinme öğeleri **Home**, **Chat** ve **Settings**'dır. Ses girişi,
  ayrı bir Voice sekmesi yerine Chat oluşturucusunda bulunur.
- Cihaz üzerinde dikte için oluşturucu mikrofonuna dokunun. Sesli not eki kaydetmek
  için uzun basın. Talk dalga biçiminden kesintisiz Talk'ı başlatın.
- Dikte, sesli not kaydı ve Talk birbirini dışlayan mikrofon
  yollarıdır; birini başlatmak diğerlerini durdurur veya engeller.
- Gerçek zamanlı Talk, bağlı bir Bluetooth Classic veya BLE kulaklık
  mikrofonunu tercih eder; bağlantı kesilirse uygulama başka bir kulaklık girişi ister veya
  varsayılan mikrofona geri döner ve yakalama durduğunda
  varsayılan tercihi geri yükler.
- Uygulama ön plandan ayrıldığında veya kullanıcı Chat'ten
  çıktığında dikte ve sesli not kaydı durur.
- Talk Mode, kapatılana veya Node bağlantısı kesilene kadar çalışmaya devam eder ve etkin olduğu sırada Android'in mikrofon ön plan hizmeti türünü kullanır.
- Android, düşük gecikmeli `AudioTrack` akışı için `pcm_16000`, `pcm_22050`, `pcm_24000` ve `pcm_44100` çıktı biçimlerini destekler.

## Notlar

- Konuşma + Mikrofon izinleri gerektirir.
- Yerel Talk, etkin Gateway oturumunu kullanır ve yalnızca yanıt olayları kullanılamadığında geçmiş yoklamasına geri döner.
- Gateway, etkin Talk sağlayıcısını kullanarak Talk oynatmasını `talk.speak` üzerinden çözümler. Android yalnızca bu RPC kullanılamadığında yerel sistem TTS'sine geri döner.
- macOS yerel MLX oynatması, varsa paketlenmiş `openclaw-mlx-tts` yardımcısını veya `PATH` üzerindeki bir yürütülebilir dosyayı kullanır. Geliştirme sırasında özel bir yardımcı ikili dosyasını göstermek için `OPENCLAW_MLX_TTS_BIN` değerini ayarlayın.
- Ses yönergesi değer aralıkları (ElevenLabs): `stability`, `similarity` ve `style`, `0..1` değerini kabul eder; `speed`, `0.5..2` değerini kabul eder; `latency_tier`, `0..4` değerini kabul eder.

## İlgili

- [Sesle uyandırma](/tr/nodes/voicewake)
- [Ses ve sesli notlar](/tr/nodes/audio)
- [Medya anlama](/tr/nodes/media-understanding)
