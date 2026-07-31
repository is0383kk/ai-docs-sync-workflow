---
read_when:
    - Bir OpenClaw aracısının Google Meet görüşmesine katılmasını istiyorsunuz
    - Bir OpenClaw aracısının yeni bir Google Meet görüşmesi oluşturmasını istiyorsunuz
    - Google Meet aktarımı olarak Chrome, Chrome node veya Twilio'yu yapılandırıyorsunuz
summary: 'Google Meet Plugin’i: açık Meet URL’lerine Chrome veya Twilio üzerinden, varsayılan agent geri konuşma ayarlarıyla katılma'
title: Google Meet Plugin'i
x-i18n:
    generated_at: "2026-07-26T22:52:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8a611e283fe900984a29b563969936a641c7af430b05933eb03b98dc93b5d0c8
    source_path: plugins/google-meet.md
    workflow: 16
---

`google-meet` Plugin, bir OpenClaw aracısı adına açıkça belirtilen Meet URL'lerine katılır. Kapsamı bilinçli olarak dardır:

- Yalnızca `https://meet.google.com/...` URL'lerine katılır; kendi bulduğu bir telefon numarasından toplantıyı asla aramaz.
- `googlemeet create`, Google Meet API'si (veya tarayıcı yedek yolu) aracılığıyla yeni bir Meet URL'si oluşturabilir ve varsayılan olarak bu URL'ye katılabilir.
- Chrome katılımı, isteğe bağlı olarak eşleştirilmiş bir Node üzerinde, oturum açılmış bir Chrome profili kullanır. Twilio katılımı, [Sesli arama Plugin'i](/tr/plugins/voice-call) aracılığıyla bir telefon numarasını ve PIN/DTMF'yi tuşlar; doğrudan bir Meet URL'sini arayamaz.
- `mode: "agent"` (varsayılan), katılımcı konuşmalarını gerçek zamanlı bir sağlayıcıyla yazıya döker, yapılandırılmış OpenClaw aracısına yönlendirir ve yanıtı normal OpenClaw TTS ile seslendirir. `mode: "bidi"`, gerçek zamanlı bir ses modelinin doğrudan yanıt vermesini sağlar. `mode: "transcribe"`, geri konuşma olmadan yalnızca gözlem için katılır.
- Plugin bir aramaya katıldığında otomatik bir onay duyurusu yapılmaz.
- CLI komutu `googlemeet`; `meet` ise daha geniş aracı telekonferans iş akışları için ayrılmıştır.

## Hızlı başlangıç

Plugin'i ve yerel ses bağımlılıklarını yükleyin, ardından gerçek zamanlı bir sağlayıcı anahtarı ayarlayın. OpenAI, `agent` modu için varsayılan yazıya dökme sağlayıcısıdır; Google Gemini Live, `bidi` modu için ses sağlayıcısı olarak kullanılabilir:

```bash
openclaw plugins install npm:@openclaw/google-meet
brew install blackhole-2ch sox
export OPENAI_API_KEY=sk-...
# yalnızca bidi modu için realtime.voiceProvider "google" olduğunda gereklidir
export GEMINI_API_KEY=...
```

`blackhole-2ch`, Chrome'un sesi yönlendirdiği `BlackHole 2ch` sanal ses aygıtını yükler. Homebrew yükleyicisi, macOS aygıtı kullanıma sunmadan önce yeniden başlatma gerektirir:

```bash
sudo reboot
```

Yeniden başlatmanın ardından her iki bileşeni de doğrulayın:

```bash
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Plugin, yüklemenin ardından varsayılan olarak etkinleştirilir. Yalnızca özelleştirmek için bir giriş ekleyin:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        config: {},
      },
    },
  },
}
```

Plugin'in etkin olmasını istemiyorsanız `openclaw plugins disable google-meet` komutunu çalıştırın.

Kurulumu denetleyin, ardından katılın:

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

`setup` çıktısı aracı tarafından okunabilir ve mod/taşıma yöntemine duyarlıdır: Chrome profilini, Node sabitlemesini ve gerçek zamanlı Chrome katılımları için BlackHole/SoX ses köprüsünü ve gecikmeli giriş kontrolünü bildirir. Yalnızca gözlem amaçlı katılımlar gerçek zamanlı ön koşulları atlar:

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
```

Twilio yetkilendirmesi yapılandırıldığında `setup`, ayrıca `voice-call`, Twilio kimlik bilgilerinin ve genel Webhook erişiminin hazır olup olmadığını bildirir. Bir aracı katılmadan önce tüm `ok: false` kontrollerini ilgili taşıma yöntemi/mod için engelleyici kabul edin. Makine tarafından okunabilir çıktı için `--json`, belirli bir taşıma yöntemini önceden ön kontrolden geçirmek için `--transport chrome|chrome-node|twilio` kullanın:

```bash
openclaw googlemeet setup --transport twilio
```

Alternatif olarak bir aracının `google_meet` aracı üzerinden katılmasını sağlayın:

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

macOS dışındaki Gateway ana makinelerinde `google_meet`; yapıt, takvim, kurulum, yazıya dökme, Twilio ve `chrome-node` eylemleri için görünür kalır ancak yerel Chrome geri konuşması (`mode: "agent"` veya `"bidi"` ile `transport: "chrome"`) ses köprüsüne ulaşmadan engellenir; çünkü bu yol şu anda macOS `BlackHole 2ch` bileşenine bağlıdır. Bunun yerine `mode: "transcribe"`, Twilio çevirmeli katılımı veya bir macOS `chrome-node` ana makinesi kullanın.

### Toplantı oluşturma

```bash
openclaw googlemeet create --transport chrome-node --mode agent
openclaw googlemeet create --no-join
```

`create`, sonuçtaki `source` alanında bildirilen iki yola sahiptir:

- **`api`**: Google Meet OAuth kimlik bilgileri yapılandırıldığında kullanılır. Belirlenimlidir; tarayıcı arayüzünün durumuna bağlı değildir.
- **`browser`**: OAuth kimlik bilgileri olmadan kullanılır. OpenClaw, sabitlenmiş Chrome Node'unda `https://meet.google.com/new` öğesini açar ve Google'ın gerçek bir toplantı kodu URL'sine yönlendirmesini bekler; söz konusu Node'daki OpenClaw Chrome profilinde Google oturumu önceden açılmış olmalıdır. Hem katılma hem de oluşturma işlemi, yeni bir sekme açmadan önce mevcut bir Meet sekmesini (veya devam eden bir `.../new` / Google hesabı istemi sekmesini) yeniden kullanır; sekme eşleştirmesi `authuser` gibi zararsız sorgu dizelerini yok sayar.

`create` varsayılan olarak katılır ve `joined: true` ile katılım oturumunu döndürür. Yalnızca URL'yi oluşturmak için `--no-join` (CLI) veya `"join": false` (araç) iletin.

API ile oluşturulan odalarda Google hesabının varsayılanını devralmak yerine açık bir erişim politikası belirleyin:

```bash
openclaw googlemeet create --access-type OPEN --transport chrome-node --mode agent
```

| `--access-type` | Kapıyı çalmadan kimler katılabilir                                  |
| --------------- | ------------------------------------------------------------------- |
| `OPEN`          | Meet URL'sine sahip herkes                                          |
| `TRUSTED`       | Ana makine kuruluşunun güvenilir kullanıcıları, davet edilen harici kullanıcılar ve çevirmeli katılım kullanıcıları |
| `RESTRICTED`    | Yalnızca davetliler                                                  |

Bu yalnızca API ile oluşturulan odalara uygulanır; dolayısıyla OAuth yapılandırılmalıdır. Bu seçenek mevcut olmadan önce kimlik doğrulaması yaptıysanız OAuth onay ekranınıza `meetings.space.settings` kapsamını ekledikten sonra `openclaw googlemeet auth login --json` komutunu yeniden çalıştırın.

Tarayıcı yedek yolu bir Google oturum açma veya Meet izin engelleyicisiyle karşılaşırsa araç; `manualActionReason`, `manualActionMessage` ve `browser.nodeId`/`browser.targetId`/`browserUrl` ile birlikte `manualActionRequired: true` döndürür. Bu mesajı bildirin ve operatör tarayıcı adımını tamamlayana kadar yeni Meet sekmeleri açmayı durdurun.

### Yalnızca gözlem amaçlı katılım

Çift yönlü gerçek zamanlı köprüyü atlamak için `"mode": "transcribe"` ayarını belirleyin (BlackHole/SoX gereksinimi ve geri konuşma yoktur). Yazıya dökme modundaki Chrome katılımları da OpenClaw'ın mikrofon/kamera izin verme adımını ve Meet'teki **Use microphone** yolunu atlar; Meet ses seçimi ara ekranını gösterirse otomasyon önce **Continue without microphone** seçeneğini dener. Yönetilen Chrome taşıma yöntemleri, canlı aracıya danışma yolunu değiştirmeden kalıcı notların kullanılabilmesi için her modda en iyi çaba esaslı bir Meet altyazı gözlemcisi kurar. `googlemeet status --json` ve `googlemeet doctor`; `captioning`, `captionsEnabledAttempted`, `transcriptLines`, `lastCaptionAt`, `lastCaptionSpeaker`, `lastCaptionText` ve bir `recentTranscript` kuyruğu bildirir.

Sınırlandırılmış oturum dökümü için izlenen Meet sekmesinin tam kaydını okuyun:

```bash
openclaw googlemeet transcript <session-id>
openclaw googlemeet transcript <session-id> --since <next-index> --json
```

Gözlemci, Meet sayfasında en fazla 2,000 tamamlanmış altyazı satırı tutar. Görünür aşamalı metin, altyazı satırı tamamlanana kadar durum sağlık kuyruğunda kalır; dolayısıyla `nextIndex` kaydedildiğinde daha sonraki bir metin genişlemesi atlanamaz. Ayrılma işlemi, anlık görüntüden önce görünür satırları sonlandırır. Sınır aşıldığında `droppedLines`, baştan kaybolan satırları bildirir. Sınırlandırılmış `googlemeet transcript` kuyruğu hâlâ yalnızca en son sona eren dört oturumu tutar ve Gateway ile birlikte sıfırlanır. OpenClaw ayrıca toplantı boyunca tamamlanan altyazı satırlarını paylaşılan durum veritabanına ekler ve ayrılma sırasında türetilmiş bir özet yazar. Bu kalıcı notları incelemek veya dışa aktarmak için [`openclaw transcripts`](/tr/cli/transcripts) kullanın.

Otomatik notlar varsayılan olarak etkindir. Kalıcı notları genel olarak
devre dışı bırakmak için `transcripts.enabled: false` ayarını belirleyin; açık `transcribe` modu yine de yalnızca
sınırlandırılmış canlı kuyruğunu gösterir. Twilio katılımlarında tarayıcı altyazı akışı yoktur ve
bu yol tarafından kaydedilmez.

Evet/hayır dinleme yoklaması için:

```bash
openclaw googlemeet test-listen <meet-url> --transport chrome-node
```

Yazıya dökme modunda katılır, yeni altyazı/döküm hareketini bekler ve `listenVerified`, `listenTimedOut`, elle yapılacak işlem alanları ve mevcut altyazı sağlığını döndürür.

### Gerçek zamanlı oturum sağlığı

Geri konuşmalı oturumlar sırasında `google_meet` durumu Chrome/ses köprüsü sağlığını bildirir: `inCall`, `manualActionRequired`, `providerConnected`, `realtimeReady`, `audioInputActive`, `audioOutputActive`, son giriş/çıkış zaman damgaları, bayt sayaçları ve köprünün kapalı olma durumu. Yönetilen Chrome oturumları, giriş/test ifadesini yalnızca sağlık durumu `inCall: true` bildirdikten sonra seslendirir; aksi takdirde `speechReady: false` ve konuşma girişimi sessizce hiçbir şey yapmamak yerine engellenir.

Yerel Chrome katılımları, oturum açılmış OpenClaw tarayıcı profili üzerinden gerçekleşir ve mikrofon/hoparlör yolu için `BlackHole 2ch` gerektirir. İlk duman testi için tek bir BlackHole aygıtı yeterlidir ancak yankı yapabilir; temiz çift yönlü ses için ayrı sanal aygıtlar veya Loopback tarzı bir grafik kullanın.

## Yerel Gateway + Parallels Chrome

Yalnızca Chrome sağlamak için macOS sanal makinesinin içinde tam bir Gateway veya model API anahtarı gerekmez. Gateway'i ve aracıyı yerel olarak, Node ana makinesini ise sanal makinede çalıştırın.

| Nerede çalışır       | Ne çalışır                                                                                       |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| Gateway ana makinesi | OpenClaw Gateway, aracı çalışma alanı, model/API anahtarları, gerçek zamanlı sağlayıcı, Google Meet Plugin yapılandırması |
| Parallels macOS VM   | OpenClaw CLI/Node ana makinesi, Chrome, SoX, BlackHole 2ch, Google oturumu açılmış bir Chrome profili |
| VM'de gerekmeyenler  | Gateway hizmeti, aracı yapılandırması, model sağlayıcısı kurulumu                               |

Sanal makine bağımlılıklarını yükleyin, yeniden başlatın ve doğrulayın:

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Plugin'i varsayılan olarak etkinleştirildiği sanal makineye yükleyin ve Node ana makinesini başlatın:

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw node run --host <gateway-host> --port 18789 --display-name parallels-macos
```

`<gateway-host>`, TLS içermeyen bir LAN IP'siyse bu güvenilir özel ağ için açıkça izin verin:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

LaunchAgent olarak yüklerken de aynı bayrağı kullanın (bu, işlem ortamıdır; yükleme komutunda bulunduğunda LaunchAgent ortamında saklanır, bir `openclaw.json` ayarı değildir):

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install --host <gateway-lan-ip> --port 18789 --display-name parallels-macos --force
openclaw node restart
```

Gateway ana makinesinden Node'u onaylayın, ardından hem `googlemeet.chrome` hem de tarayıcı yeteneğini/`browser.proxy` sunduğunu doğrulayın:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Meet'i bu Node üzerinden yönlendirin:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["googlemeet.chrome", "browser.proxy"] },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          chrome: {
            guestName: "OpenClaw Agent",
            autoJoin: true,
            reuseExistingTab: true,
          },
          chromeNode: {
            node: "parallels-macos",
          },
        },
      },
    },
  },
}
```

Şimdi Gateway ana makinesinden normal biçimde katılın:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

Bir oturum oluşturan veya mevcut oturumu yeniden kullanan, bilinen bir ifadeyi seslendiren ve oturum sağlığını yazdıran tek komutluk duman testi için:

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij
```

Gerçek zamanlı katılım sırasında tarayıcı otomasyonu konuk adını doldurur, Join/Ask to join düğmesine tıklar ve göründüğünde Meet'in ilk çalıştırmadaki "Use microphone" istemini (veya yalnızca gözlem amaçlı katılım ve yalnızca tarayıcı üzerinden toplantı oluşturma sırasında "Continue without microphone" seçeneğini) kabul eder. Profilin oturumu kapalıysa, Meet toplantı sahibinin kabulünü bekliyorsa, Chrome mikrofon/kamera iznine ihtiyaç duyuyorsa veya Meet çözümlenmemiş bir istemde takılı kaldıysa sonuç, `manualActionReason` ve `manualActionMessage` ile birlikte `manualActionRequired: true` bildirir. Yeniden denemeyi durdurun, bu mesajı `browserUrl`/`browserTitle` ile birlikte bildirin ve yalnızca manuel işlem tamamlandıktan sonra yeniden deneyin.

`chromeNode.node` belirtilmezse OpenClaw, yalnızca tam olarak bir bağlı Node hem `googlemeet.chrome` hem de tarayıcı denetimi sunduğunda otomatik seçim yapar; birden fazla uygun Node bağlı olduğunda `chromeNode.node` değerini (Node kimliği, görünen ad veya uzak IP) sabitleyin.

### Yaygın hata kontrolleri

| Belirti                                                  | Düzeltme                                                                                                                                                                                                                                                                                   |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Configured Google Meet node ... is not usable: offline` | Sabitlenen Node biliniyor ancak kullanılamıyor. Kurulum engelini bildirin; istenmedikçe sessizce başka bir aktarıma geri dönmeyin.                                                                                                                                                      |
| `No connected Google Meet-capable node`                  | VM'ye `npm:@openclaw/google-meet` yükleyin, `openclaw plugins enable browser` çalıştırın, `openclaw node run` başlatın ve eşleştirmeyi onaylayın. Google Meet açıkça devre dışı bırakıldıysa onu da etkinleştirin. `gateway.nodes.commands.allow` çıktısının `googlemeet.chrome` ve `browser.proxy` içerdiğini doğrulayın. |
| `BlackHole 2ch audio device not found`                   | Denetlenen ana sisteme `blackhole-2ch` yükleyin ve yeniden başlatın.                                                                                                                                                                                                                         |
| `BlackHole 2ch audio device not found on the node`       | VM'ye `blackhole-2ch` yükleyin ve VM'yi yeniden başlatın.                                                                                                                                                                                                                                  |
| Chrome açılıyor ancak katılamıyor                             | VM'deki tarayıcı profilinde oturum açın veya `chrome.guestName` ayarını koruyun. Otomatik konuk katılımı, Node tarayıcı proxy'si üzerinden OpenClaw tarayıcı otomasyonunu kullanır; Node'un `browser.defaultProfile` değerini (veya adlandırılmış bir mevcut oturum profilini) istediğiniz profile yönlendirin.                   |
| Yinelenen Meet sekmeleri                                      | `chrome.reuseExistingTab: true` ayarını koruyun. OpenClaw, başka bir sekme açmadan önce aynı URL'ye sahip mevcut bir sekmeyi etkinleştirir ve oluşturma işlemi devam eden bir `.../new` veya Google hesabı istemi sekmesini yeniden kullanır.                                                                                        |
| Ses yok                                                 | Meet mikrofon/hoparlörünü OpenClaw tarafından kullanılan sanal ses yolu üzerinden yönlendirin; temiz çift yönlü ses için ayrı sanal aygıtlar veya Loopback tarzı yönlendirme kullanın.                                                                                                                                |

## Yükleme notları

Chrome geri konuşma varsayılanı, OpenClaw'ın paketlemediği veya yeniden dağıtmadığı iki harici araç kullanır; bunları Homebrew aracılığıyla ana sistem bağımlılıkları olarak yükleyin:

- `sox`: komut satırı ses yardımcı programı. Plugin, varsayılan 24 kHz PCM16 ses köprüsü için açık CoreAudio aygıt komutları gönderir.
- `blackhole-2ch`: Chrome/Meet'in yönlendirildiği `BlackHole 2ch` aygıtını sağlayan macOS sanal ses sürücüsü.

SoX, `LGPL-2.0-only AND GPL-2.0-only` kapsamında lisanslanmıştır; BlackHole GPL-3.0 kapsamındadır. BlackHole'u OpenClaw ile paketleyen bir yükleyici veya cihaz oluşturursanız BlackHole'un üst kaynak lisanslamasını inceleyin ya da Existential Audio'dan ayrı bir lisans alın.

## Aktarımlar

| Aktarım     | Kullanım durumu                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------- |
| `chrome`      | Chrome/ses Gateway ana sisteminde çalışıyorsa                                                        |
| `chrome-node` | Chrome/ses eşleştirilmiş bir Node'da çalışıyorsa (örneğin bir Parallels macOS VM)                        |
| `twilio`      | Chrome katılımı kullanılamadığında Voice Call Plugin aracılığıyla telefonla arama yedeği |

### Chrome

Meet URL'sini OpenClaw tarayıcı denetimi aracılığıyla açar ve oturum açılmış OpenClaw tarayıcı profili olarak katılır. macOS'ta Plugin, başlatmadan önce `BlackHole 2ch` denetimi yapar ve yapılandırılmışsa Chrome'u açmadan önce bir ses köprüsü sağlık/başlatma komutu çalıştırır. Yerel Chrome için profili `browser.defaultProfile` ile seçin; bunun yerine `chrome.browserProfile`, `chrome-node` ana sistemlerine iletilir.

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome-node
```

Chrome mikrofon/hoparlör sesi, yerel OpenClaw ses köprüsü üzerinden yönlendirilir. `BlackHole 2ch` yüklü değilse katılım, ses yolu olmadan katılmak yerine bir kurulum hatasıyla başarısız olur.

### Twilio

[Voice Call Pluginine](/tr/plugins/voice-call) devredilen katı bir arama planıdır. Telefon numaraları için Meet sayfalarını ayrıştırmaz; Google Meet, toplantı için telefonla katılım numarası ve PIN sunmalıdır.

Voice Call'u Chrome Node'da değil, Gateway ana sisteminde etkinleştirin:

```json5
{
  plugins: {
    allow: ["google-meet", "voice-call", "google"],
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          // veya Twilio varsayılan olacaksa "twilio" olarak ayarlayın
        },
      },
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          inboundPolicy: "allowlist",
          realtime: {
            enabled: true,
            provider: "google",
            instructions: "Bu Google Meet'e bir OpenClaw aracısı olarak katıl. Kısa konuş.",
            toolPolicy: "safe-read-only",
            providers: {
              google: {
                silenceDurationMs: 500,
                startSensitivity: "high",
              },
            },
          },
        },
      },
      google: {
        enabled: true,
      },
    },
  },
}
```

Gizli bilgileri `openclaw.json` dışında tutmak için Twilio kimlik bilgilerini ortam üzerinden sağlayın:

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
export GEMINI_API_KEY=...
```

Gerçek zamanlı ses sağlayıcısı OpenAI ise bunun yerine `realtime.provider: "openai"` ile `OPENAI_API_KEY` kullanın.

`voice-call` etkinleştirildikten sonra Gateway'i yeniden başlatın veya yeniden yükleyin; Plugin yapılandırması değişiklikleri yeniden yüklemeye kadar etkili olmaz. Doğrulayın:

```bash
openclaw config validate
openclaw plugins list | grep -E 'google-meet|voice-call'
openclaw googlemeet setup
```

Twilio devri bağlandığında `googlemeet setup`, `twilio-voice-call-plugin`, `twilio-voice-call-credentials` ve `twilio-voice-call-webhook` kontrollerini içerir.

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

Özel bir dizi için `--dtmf-sequence` kullanın; PIN'den önce duraklama için başına `w` veya virgül ekleyin:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

## OAuth ve ön kontrol

Meet bağlantısı oluşturmak için OAuth isteğe bağlıdır çünkü `googlemeet create` tarayıcı otomasyonuna geri dönebilir. Resmî API ile oluşturma, alan çözümleme veya Meet Media API ön kontrolü için OAuth'u yapılandırın. Chrome/Chrome-node katılımları hiçbir zaman OAuth'a bağlı değildir; her iki durumda da oturum açılmış bir Chrome profili, BlackHole/SoX ve (`chrome-node` için) bağlı bir Node kullanırlar.

### Google kimlik bilgileri oluşturma

Google Cloud Console'da:

<Steps>
<Step title="Bir proje oluşturun veya seçin">
</Step>
<Step title="Google Meet REST API'yi etkinleştirin">
</Step>
<Step title="OAuth izin ekranını yapılandırın">
Bir Google Workspace kuruluşu için Internal en basit seçenektir. External, kişisel/test kurulumları için çalışır; uygulama Testing durumundayken yetkilendirme yapacak her Google hesabını test kullanıcısı olarak ekleyin.
</Step>
<Step title="İstenen kapsamları ekleyin">
- `https://www.googleapis.com/auth/meetings.space.created`
- `https://www.googleapis.com/auth/meetings.space.readonly`
- `https://www.googleapis.com/auth/meetings.space.settings`
- `https://www.googleapis.com/auth/meetings.conference.media.readonly`
- `https://www.googleapis.com/auth/calendar.events.readonly` (Takvim araması)
- `https://www.googleapis.com/auth/drive.meet.readonly` (transkript/akıllı not belge gövdesi dışa aktarımı)

</Step>
<Step title="Bir OAuth istemci kimliği oluşturun">
Uygulama türü **Web application**. Yetkilendirilmiş yönlendirme URI'si:

```text
http://localhost:8085/oauth2callback
```

</Step>
<Step title="İstemci kimliğini ve istemci gizli anahtarını kopyalayın">
</Step>
</Steps>

`meetings.space.created`, `spaces.create` için gereklidir. `meetings.space.readonly`, Meet URL'lerini/kodlarını alanlara çözümler. `meetings.space.settings`, OpenClaw'ın API ile oda oluşturma sırasında `accessType` gibi `SpaceConfig` ayarlarını iletmesini sağlar. `meetings.conference.media.readonly`, Meet Media API ön kontrolü ve medya çalışmaları içindir; Google, Media API'nin fiilî kullanımı için Developer Preview kaydı gerektirebilir. `calendar.events.readonly` yalnızca `--today`/`--event` takvim araması için gereklidir. `drive.meet.readonly` yalnızca `--include-doc-bodies` dışa aktarımı için gereklidir. Yalnızca tarayıcı tabanlı Chrome katılımlarına ihtiyacınız varsa OAuth'u tamamen atlayın.

### Yenileme belirtecini oluşturma

`oauth.clientId` ve isteğe bağlı olarak `oauth.clientSecret` yapılandırın (veya bunları ortam değişkenleri olarak iletin), ardından şunu çalıştırın:

```bash
openclaw googlemeet auth login --json
```

Bu, `http://localhost:8085/oauth2callback` üzerinde localhost geri çağrısıyla bir PKCE akışı çalıştırır ve yenileme belirteci içeren bir `oauth` yapılandırma bloğu yazdırır. Tarayıcı yerel geri çağrıya erişemediğinde kopyala/yapıştır akışı için `--manual` ekleyin:

```bash
OPENCLAW_GOOGLE_MEET_CLIENT_ID="your-client-id" \
OPENCLAW_GOOGLE_MEET_CLIENT_SECRET="your-client-secret" \
openclaw googlemeet auth login --json --manual
```

JSON çıktısı:

```json
{
  "oauth": {
    "clientId": "your-client-id",
    "clientSecret": "your-client-secret",
    "refreshToken": "refresh-token",
    "accessToken": "access-token",
    "expiresAt": 1770000000000
  },
  "scope": "..."
}
```

`oauth` nesnesini Plugin yapılandırması altında saklayın:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          oauth: {
            clientId: "your-client-id",
            clientSecret: "your-client-secret",
            refreshToken: "refresh-token",
          },
        },
      },
    },
  },
}
```

Yenileme belirtecini yapılandırmada tutmak istemediğinizde ortam değişkenlerini tercih edin; önce yapılandırma çözümlenir, ardından yedek olarak ortam kullanılır. Toplantı oluşturma, takvim araması veya belge gövdesi dışa aktarma desteği mevcut olmadan önce kimlik doğrulaması yaptıysanız yenileme belirtecinin geçerli kapsam kümesini kapsaması için `openclaw googlemeet auth login --json` komutunu yeniden çalıştırın.

### OAuth'u doctor ile doğrulama

```bash
openclaw googlemeet doctor --oauth --json
```

Bu, Chrome çalışma zamanını yüklemeden veya bağlı bir Node gerektirmeden OAuth yapılandırmasının mevcut olduğunu ve yenileme token'ının bir erişim token'ı oluşturabildiğini denetler. Rapor yalnızca durum alanlarını (`ok`, `configured`, `tokenSource`, `expiresAt`, denetim mesajları) içerir ve erişim token'ını, yenileme token'ını veya istemci sırrını asla yazdırmaz.

| Denetim                | Anlamı                                                                          |
| -------------------- | -------------------------------------------------------------------------------- |
| `oauth-config`       | `oauth.clientId` ile `oauth.refreshToken` veya önbelleğe alınmış bir erişim token'ı mevcut |
| `oauth-token`        | Önbelleğe alınmış erişim token'ı hâlâ geçerli veya yenileme token'ı yeni bir tane oluşturdu    |
| `meet-spaces-get`    | İsteğe bağlı `--meeting` denetimi mevcut bir Meet alanını çözümledi                       |
| `meet-spaces-create` | İsteğe bağlı `--create-space` denetimi yeni bir Meet alanı oluşturdu                         |

Yan etkili oluşturma denetimiyle Meet API'nin etkinleştirildiğini ve `spaces.create` kapsamını doğrulayın:

```bash
openclaw googlemeet doctor --oauth --create-space --json
```

Mevcut bir alana okuma erişimini doğrulayın:

```bash
openclaw googlemeet doctor --oauth --meeting https://meet.google.com/abc-defg-hij --json
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
```

Bu denetimlerden gelen bir `403` genellikle Meet REST API'nin devre dışı olduğu, yenileme token'ında gerekli kapsamın bulunmadığı veya Google hesabının bu alana erişemediği anlamına gelir. Yenileme token'ı hatası, `openclaw googlemeet auth login --json` komutunu yeniden çalıştırmanız ve yeni `oauth` bloğunu saklamanız gerektiği anlamına gelir.

Tarayıcı geri dönüşü için OAuth gerekmez; buradaki Google kimlik doğrulaması OpenClaw yapılandırmasından değil, seçilen Node üzerindeki oturum açılmış Chrome profilinden gelir.

Bu ortam değişkenleri geri dönüş olarak kabul edilir:

- `OPENCLAW_GOOGLE_MEET_CLIENT_ID` veya `GOOGLE_MEET_CLIENT_ID`
- `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET` veya `GOOGLE_MEET_CLIENT_SECRET`
- `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` veya `GOOGLE_MEET_REFRESH_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN` veya `GOOGLE_MEET_ACCESS_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` veya `GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT`
- `OPENCLAW_GOOGLE_MEET_DEFAULT_MEETING` veya `GOOGLE_MEET_DEFAULT_MEETING`
- `OPENCLAW_GOOGLE_MEET_PREVIEW_ACK` veya `GOOGLE_MEET_PREVIEW_ACK`

### Çözümleme, ön denetim ve artefaktları okuma

```bash
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet preflight --meeting https://meet.google.com/abc-defg-hij
```

Meet konferans kayıtlarını oluşturduktan sonra:

```bash
openclaw googlemeet artifacts --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet attendance --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet export --meeting https://meet.google.com/abc-defg-hij --output ./meet-export
```

`--meeting` ile `artifacts` ve `attendance` varsayılan olarak en son konferans kaydını kullanır; tutulan her kayıt için `--all-conference-records` parametresini iletin.

Takvim araması, artefaktları okumadan önce toplantı URL'sini Google Calendar'dan çözümler (Calendar etkinlikleri salt okunur kapsamını içeren bir yenileme token'ı gerektirir):

```bash
openclaw googlemeet latest --today
openclaw googlemeet calendar-events --today --json
openclaw googlemeet artifacts --event "Weekly sync"
openclaw googlemeet attendance --today --format csv --output attendance.csv
```

`--today`, Meet bağlantısı içeren bir etkinlik için bugünün `primary` takvimini arar; `--event <query>` eşleşen etkinlik metnini arar; `--calendar <id>` birincil olmayan bir takvimi hedefler. `calendar-events` eşleşen etkinliklerin önizlemesini gösterir ve `latest`/`artifacts`/`attendance`/`export` seçeneklerinin hangisini seçeceğini işaretler.

Konferans kaydı kimliğini zaten biliyorsanız doğrudan belirtin:

```bash
openclaw googlemeet latest --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 --json
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 --json
```

API tarafından oluşturulan bir alanın odasını kapatın:

```bash
openclaw googlemeet end-active-conference https://meet.google.com/abc-defg-hij
```

`spaces.endActiveConference` çağrısını yapar ve yetkilendirilmiş hesabın yönetebildiği bir alan için `meetings.space.created` kapsamına sahip OAuth gerektirir. Bir Meet URL'sini, toplantı kodunu veya `spaces/{id}` değerini kabul eder ve önce bunu API alan kaynağına çözümler. Bu, `googlemeet leave` işleminden ayrıdır: `leave`, OpenClaw'ın yerel/oturum katılımını durdurur; `end-active-conference`, Google Meet'ten alanın etkin konferansını sonlandırmasını ister.

Okunabilir bir rapor yazın:

```bash
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 \
  --format markdown --output meet-artifacts.md
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 \
  --format csv --output meet-attendance.csv
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --zip --output meet-export
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --dry-run
```

`artifacts`, Google tarafından sunulduğunda konferans kaydı meta verileriyle birlikte katılımcı, kayıt, transkript, yapılandırılmış transkript girdisi ve akıllı not kaynağı meta verilerini döndürür. `--no-transcript-entries`, büyük toplantılarda girdi aramasını atlar. `attendance`, katılımcıları ilk/son görülme zamanlarını, toplam oturum süresini, geç kalma/erken ayrılma bayraklarını içeren katılımcı oturumu satırlarına genişletir ve yinelenen katılımcı kaynaklarını oturum açmış kullanıcıya veya görünen ada göre birleştirir; `--no-merge-duplicates` ham kaynakları ayrı tutar, `--late-after-minutes`/`--early-before-minutes` eşikleri ayarlar.

`export`; `summary.md`, `attendance.csv`, `transcript.md`, `artifacts.json`, `attendance.json` ve `manifest.json` içeren bir klasör yazar. `manifest.json`, seçilen girdiyi, dışa aktarma seçeneklerini, konferans kayıtlarını, çıktı dosyalarını, sayıları, token kaynağını, kullanılan Calendar etkinliğini ve kısmi alma uyarılarını kaydeder. `--zip`, klasörün yanına taşınabilir bir arşiv de yazar. `--include-doc-bodies`, bağlantılı transkript/akıllı not Google Docs metnini Drive `files.export` aracılığıyla dışa aktarır (Drive Meet salt okunur kapsamını gerektirir); bu olmadan dışa aktarımlar yalnızca Meet meta verilerini ve yapılandırılmış transkript girdilerini içerir. Kısmi bir artefakt hatası (akıllı not listeleme, transkript girdisi veya belge gövdesi hatası), tüm dışa aktarımın başarısız olmasına neden olmak yerine uyarıyı özette/manifestte tutar. `--dry-run`, aynı verileri getirir ve klasörü veya ZIP'i oluşturmadan manifest JSON'unu yazdırır.

Aracılar aynı eylemleri `google_meet` aracı üzerinden kullanır (`export`, `accessType` ile `create`, `end_active_conference`, `test_listen`); bkz. [Araç](#tool).

### Canlı duman testi

```bash
OPENCLAW_LIVE_TEST=1 \
OPENCLAW_GOOGLE_MEET_LIVE_MEETING=https://meet.google.com/abc-defg-hij \
pnpm test:live -- extensions/google-meet/google-meet.live.test.ts
```

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
openclaw googlemeet test-listen https://meet.google.com/abc-defg-hij --transport chrome-node --timeout-ms 30000
```

| Değişken                                                                                                                  | Amaç                                                                |
| ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `OPENCLAW_LIVE_TEST=1`                                                                                                    | Korumalı canlı testleri etkinleştirir                                             |
| `OPENCLAW_GOOGLE_MEET_LIVE_MEETING`                                                                                       | Tutulan Meet URL'si, kodu veya `spaces/{id}`                              |
| `OPENCLAW_GOOGLE_MEET_CLIENT_ID` / `GOOGLE_MEET_CLIENT_ID`                                                                | OAuth istemci kimliği                                                        |
| `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` / `GOOGLE_MEET_REFRESH_TOKEN`                                                        | Yenileme token'ı                                                          |
| `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET`, `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN`, `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` | İsteğe bağlı; `OPENCLAW_` ön eki olmayan aynı geri dönüş adları da çalışır |

Temel artefakt/katılım duman testi `meetings.space.readonly` ve `meetings.conference.media.readonly` gerektirir. Calendar araması `calendar.events.readonly` gerektirir. Drive belge gövdesi dışa aktarımı `drive.meet.readonly` gerektirir.

### Oluşturma örnekleri

```bash
openclaw googlemeet create
```

Yeni toplantı URI'sini, kaynağını ve katılma oturumunu yazdırır. OAuth ile Meet API'yi; OAuth olmadan sabitlenmiş Chrome Node'unun oturum açılmış profilini kullanır. Tarayıcı geri dönüşü JSON'u:

```json
{
  "source": "browser",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

Tarayıcı geri dönüşü önce Google oturum açma sayfasına veya bir Meet izin engeline ulaşırsa `google_meet`, düz bir dize yerine yapılandırılmış ayrıntılar döndürür:

```json
{
  "source": "browser",
  "error": "google-login-required: OpenClaw tarayıcı profilinde Google'da oturum açın, ardından toplantı oluşturmayı yeniden deneyin.",
  "manualActionRequired": true,
  "manualActionReason": "google-login-required",
  "manualActionMessage": "OpenClaw tarayıcı profilinde Google'da oturum açın, ardından toplantı oluşturmayı yeniden deneyin.",
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1",
    "browserUrl": "https://accounts.google.com/signin",
    "browserTitle": "Sign in - Google Accounts"
  }
}
```

API oluşturma JSON'u:

```json
{
  "source": "api",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "space": {
    "name": "spaces/abc-defg-hij",
    "meetingCode": "abc-defg-hij",
    "meetingUri": "https://meet.google.com/abc-defg-hij"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

Oluşturma işlemi varsayılan olarak toplantıya katılır ancak Chrome/Chrome-node'un tarayıcı üzerinden katılması için yine de oturum açılmış bir Google profili gerekir; oturum kapalıysa OpenClaw, `manualActionRequired: true` veya bir tarayıcı geri dönüşü hatası bildirir ve yeniden denemeden önce operatörden Google oturum açma işlemini tamamlamasını ister.

`preview.enrollmentAcknowledged: true` değerini yalnızca Cloud projenizin, OAuth sorumlusunun ve toplantı katılımcılarının Meet medya API'leri için Google Workspace Developer Preview Program'a kayıtlı olduğunu doğruladıktan sonra ayarlayın.

## Yapılandırma

Yaygın Chrome aracı yolu yalnızca Plugin'in etkinleştirilmesini, BlackHole'u, SoX'u, gerçek zamanlı bir sağlayıcı anahtarını ve yapılandırılmış bir OpenClaw TTS sağlayıcısını gerektirir:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {},
      },
    },
  },
}
```

### Varsayılanlar

| Anahtar                           | Varsayılan                               | Notlar                                                                                                                                                                                                            |
| --------------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `defaultTransport`                | `"chrome"`                       |                                                                                                                                                                                                                   |
| `defaultMode`                | `"agent"`                       | `"realtime"`, `"agent"` için eski bir diğer ad olarak kabul edilir; yeni çağıranlar `"agent"` kullanmalıdır                                                                                |
| `chromeNode.node`                | ayarlanmamış                             | `chrome-node` için Node kimliği/adı/IP'si; birden fazla uygun Node bağlı olabiliyorsa gereklidir                                                                                                              |
| `chrome.launch`                | `true`                       | Katılma işlemi için Chrome'u başlatır; `false` yalnızca zaten açık olan bir oturum yeniden kullanılırken ayarlanmalıdır                                                                                 |
| `chrome.audioBackend`                | `"blackhole-2ch"`                       |                                                                                                                                                                                                                   |
| `chrome.guestName`                | `"OpenClaw Agent"`                       | Oturum kapalıyken Meet konuk ekranında gösterilir                                                                                                                                                                 |
| `chrome.autoJoin`                | `true`                       | `chrome-node` üzerinde konuk adını mümkün olduğunca doldurur ve Join Now düğmesine tıklar                                                                                                                     |
| `chrome.reuseExistingTab`                | `true`                       | Yinelenen sekmeler açmak yerine mevcut bir Meet sekmesini etkinleştirir                                                                                                                                           |
| `chrome.waitForInCallMs`                | `20000`                       | Geri konuşma girişi başlatılmadan önce Meet sekmesinin görüşmede olduğunu bildirmesini bekler                                                                                                                     |
| `chrome.audioFormat`                | `"pcm16-24khz"`                       | Komut çifti ses biçimi; `"g711-ulaw-8khz"` yalnızca telefon sesi üreten eski/özel komut çiftleri içindir                                                                                                           |
| `chrome.audioBufferBytes`                | `4096`                       | Oluşturulan komut çifti ses komutları için SoX işleme arabelleği (SoX'un varsayılan 8192 baytlık arabelleğinin yarısıdır ve kanal gecikmesini azaltır); değerler en az 17 bayt olacak şekilde sınırlandırılır      |
| `chrome.audioInputCommand`                | oluşturulan SoX komutu                   | CoreAudio `BlackHole 2ch` üzerinden okur, sesi `chrome.audioFormat` biçiminde yazar                                                                                                                              |
| `chrome.audioOutputCommand`                | oluşturulan SoX komutu                   | Sesi `chrome.audioFormat` biçiminde okur, CoreAudio `BlackHole 2ch` üzerine yazar                                                                                                                                |
| `chrome.bargeInInputCommand`                | ayarlanmamış                             | Asistan oynatımı sırasında insanın araya girdiğini algılamak için işaretli 16 bit küçük son haneli mono PCM yazan isteğe bağlı yerel mikrofon komutu; Gateway üzerinde barındırılan komut çifti köprüsüne uygulanır |
| `chrome.bargeInRmsThreshold`                | `650`                       | İnsan müdahalesi olarak sayılan RMS düzeyi                                                                                                                                                                        |
| `chrome.bargeInPeakThreshold`                | `2500`                       | İnsan müdahalesi olarak sayılan tepe düzeyi                                                                                                                                                                       |
| `chrome.bargeInCooldownMs`                | `900`                       | Yinelenen müdahale temizlemeleri arasındaki en kısa gecikme                                                                                                                                                       |
| `mode` (istek başına) | `"agent"`                       | Geri konuşma modu; [Ajan ve bidi modları](#agent-and-bidi-modes) tablosuna bakın                                                                                                                                  |
| `realtime.provider`                | `"openai"`                       | Aşağıdaki kapsamlı alanlar ayarlanmamışken kullanılan uyumluluk geri dönüşü                                                                                                                                        |
| `realtime.transcriptionProvider`                | `"openai"`                       | Gerçek zamanlı transkripsiyon için `agent` modunun kullandığı sağlayıcı kimliği                                                                                                                        |
| `realtime.voiceProvider`                | ayarlanmamış                             | Doğrudan gerçek zamanlı ses için `bidi` modunun kullandığı sağlayıcı kimliği; ajan modu transkripsiyonunu OpenAI üzerinde tutarken Gemini Live kullanmak için `"google"` olarak ayarlayın. Belirli Gemini Live modelini seçmek için `realtime.model` ile eşleştirin. |
| `realtime.toolPolicy`                | `"safe-read-only"`                       | [Ajan ve bidi modları](#agent-and-bidi-modes) bölümüne bakın                                                                                                                                                       |
| `realtime.instructions`                | kısa sözlü yanıt talimatları             | Modele kısa konuşmasını ve daha ayrıntılı yanıtlar için `openclaw_agent_consult` kullanmasını söyler                                                                                                                    |
| `realtime.introMessage`                | `"Say exactly: I'm here and listening."`                       | Gerçek zamanlı köprü bağlandığında bir kez söylenir; sessizce katılmak için `""` olarak ayarlayın                                                                                                   |
| `realtime.agentId`                | `"main"`                       | `openclaw_agent_consult` için kullanılan OpenClaw ajan kimliği                                                                                                                                                          |
| `voiceCall.enabled`                | `true`                       | Twilio PSTN aramasını, DTMF'yi ve giriş selamlamasını Voice Call pluginine devreder                                                                                                                               |
| `voiceCall.dtmfDelayMs`                | `12000`                       | PIN'den türetilen bir DTMF dizisi Twilio üzerinden oynatılmadan önceki başlangıç beklemesi                                                                                                                        |
| `voiceCall.postDtmfSpeechDelayMs`                | `5000`                       | Voice Call, Twilio ayağını başlattıktan sonra gerçek zamanlı giriş selamlamasını istemeden önceki gecikme                                                                                                         |

`chrome.audioBridgeCommand` ve `chrome.audioBridgeHealthCommand`, `chrome.audioInputCommand`/`chrome.audioOutputCommand` yerine harici bir köprünün yerel ses yolunun tamamını yönetmesine olanak tanır; bunları hangi modun kullanabileceğine ilişkin kısıtlama için [Notlar](#notes) bölümüne bakın.

Eski `realtime.provider: "google"` biçimi için bir `openclaw doctor --fix` geçişi vardır: bu alanlar henüz ayarlanmamışsa söz konusu amacı `realtime.voiceProvider: "google"` ile `realtime.transcriptionProvider: "openai"` alanlarına taşır.

### İsteğe bağlı geçersiz kılmalar

```json5
{
  defaults: {
    meeting: "https://meet.google.com/abc-defg-hij",
  },
  browser: {
    defaultProfile: "openclaw",
  },
  chrome: {
    guestName: "OpenClaw Agent",
    waitForInCallMs: 30000,
    bargeInInputCommand: [
      "sox",
      "-q",
      "-t",
      "coreaudio",
      "External Microphone",
      "-r",
      "24000",
      "-c",
      "1",
      "-b",
      "16",
      "-e",
      "signed-integer",
      "-t",
      "raw",
      "-",
    ],
  },
  chromeNode: {
    node: "parallels-macos",
  },
  defaultMode: "agent",
  realtime: {
    provider: "openai",
    transcriptionProvider: "openai",
    voiceProvider: "google",
    model: "gemini-3.1-flash-live-preview",
    agentId: "jay",
    toolPolicy: "owner",
    introMessage: "Tam olarak şunu söyle: Buradayım.",
    providers: {
      google: {
        speakerVoice: "Kore",
      },
    },
  },
}
```

Hem ajan modunda dinleme hem de konuşma için ElevenLabs:

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        modelId: "eleven_v3",
        speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
      },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        config: {
          realtime: {
            transcriptionProvider: "elevenlabs",
            providers: {
              elevenlabs: {
                modelId: "scribe_v2_realtime",
                audioFormat: "ulaw_8000",
                sampleRate: 8000,
                commitStrategy: "vad",
              },
            },
          },
        },
      },
    },
  },
}
```

Kalıcı Meet sesi `tts.providers.elevenlabs.speakerVoiceId` kaynağından gelir. TTS modeli geçersiz kılmaları etkinleştirildiğinde ajan yanıtları, yanıt başına `[[tts:speakerVoiceId=... model=eleven_v3]]` yönergelerini de kullanabilir; ancak toplantılar için yapılandırma belirlenimci varsayılandır. Katılım sırasında günlüklerde `transcriptionProvider=elevenlabs` gösterilir ve söylenen her yanıt `provider=elevenlabs model=eleven_v3 speakerVoiceId=<voiceId>` olarak günlüğe kaydedilir.

Yalnızca Twilio yapılandırması:

```json5
{
  defaultTransport: "twilio",
  twilio: {
    defaultDialInNumber: "+15551234567",
    defaultPin: "123456",
  },
  voiceCall: {
    gatewayUrl: "ws://127.0.0.1:18789",
  },
}
```

`voiceCall.enabled: true` (varsayılan) ve Twilio aktarımı kullanıldığında Voice Call, gerçek zamanlı medya akışını açmadan önce DTMF dizisini gönderir, ardından kaydedilmiş giriş metnini ilk gerçek zamanlı selamlama olarak kullanır. `voice-call` etkin değilse Google Meet, arama planını yine de doğrulayıp kaydedebilir ancak Twilio aramasını gerçekleştiremez.

Yerel güvenilir Gateway çalışma zamanını kullanmak için `voiceCall.gatewayUrl` ayarını yapmayın; bu çalışma zamanı, çağrıyı
başlatan agent'ı çağrının tamamı boyunca korur. Yapılandırılmış bir Gateway URL'si açık bir WebSocket hedefi olarak kalır ve
plugin kaynağının kimliğini doğrulayamaz; varsayılan olmayan agent katılımları, sessizce
başka bir agent kullanmak yerine güvenli biçimde başarısız olur. Agent başına
yönlendirme gerektiğinde Google Meet ve Voice Call'u aynı Gateway işleminde çalıştırın.

## Araç

Agent'lar `google_meet` aracını kullanır:

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

| `action`                | Amaç                                                                                              |
| ----------------------- | ------------------------------------------------------------------------------------------------- |
| `join`                  | Açıkça belirtilen bir Meet URL'sine katılma                                                       |
| `create`                | Bir alan oluşturma (ve varsayılan olarak katılma); `accessType`/`entryPointAccess` desteği sunar |
| `status`                | Etkin oturumları listeleme veya birini `sessionId` ile inceleme                            |
| `setup_status`          | `googlemeet setup` ile aynı kontrolleri çalıştırma                                                |
| `resolve_space`         | `spaces.get` aracılığıyla bir URL'yi/kodu/`spaces/{id}` değerini çözümleme              |
| `preflight`             | OAuth ve toplantı çözümleme ön koşullarını doğrulama                                               |
| `latest`                | Bir toplantının en son konferans kaydını bulma                                                    |
| `calendar_events`       | Meet bağlantılı Calendar etkinliklerini önizleme                                                  |
| `artifacts`             | Konferans kayıtlarını ve katılımcı/kayıt/transkript/akıllı not meta verilerini listeleme           |
| `attendance`            | Katılımcıları ve katılımcı oturumlarını listeleme                                                  |
| `export`                | Yapıt/katılım/transkript/manifest paketini yazma; yalnızca manifest için `"dryRun": true` ayarlayın |
| `recover_current_tab`   | Yeni bir sekme açmadan mevcut bir Meet sekmesine odaklanma/sekmeyi inceleme                       |
| `transcript`            | Sınırlandırılmış altyazı transkriptini okuma; `sinceIndex`, önceki `nextIndex` konumundan devam eder |
| `leave`                 | Oturumu sonlandırma (Chrome Ayrıl düğmesine tıklar; yalnızca açtığı sekmeleri kapatır; Twilio aramayı kapatır) |
| `end_active_conference` | API tarafından yönetilen bir alanın etkin Google Meet konferansını sonlandırma                    |
| `speak`                 | `sessionId` ve `message` verildiğinde gerçek zamanlı agent'ın hemen konuşmasını sağlama |
| `test_speech`           | Oturum oluşturma/yeniden kullanma, bilinen bir ifadeyi tetikleme ve Chrome durumunu döndürme       |
| `test_listen`           | Yalnızca gözlem oturumu oluşturma/yeniden kullanma ve altyazı/transkript hareketini bekleme        |

`test_speech` her zaman `mode: "agent"` veya `"bidi"` kullanımını zorunlu kılar ve yalnızca gözlem oturumları konuşma üretemediğinden `mode: "transcribe"` içinde çalıştırılması istendiğinde başarısız olur. `speechOutputVerified`, hem yeni gerçek zamanlı çıkış baytlarının hem de bu çıkış sırasında köprünün mikrofon yakalama yoluna dönen yeni ve sessiz olmayan sesin bulunmasını gerektirir. Yeniden kullanılan bir oturumun eski çıkışı veya geri döngü sinyali hesaba katılmaz ve yalnızca alıcı baytlarındaki artış artık konuşmanın doğrulandığını bildirmez.

Chrome aktarımlarında `leave`, Meet'in Ayrıl düğmesine tıklandıktan sonra yeniden kullanılan ve kullanıcıya ait olan sekmeyi açık tutar. OpenClaw tarafından açılan sekmeler ayrılmanın ardından kapatılır.

Chrome Gateway ana makinesinde çalıştığında `transport: "chrome"`, eşleştirilmiş bir Node üzerinde çalıştığında `transport: "chrome-node"` kullanın. Her iki durumda da model sağlayıcıları ve `openclaw_agent_consult` Gateway ana makinesinde çalışır; böylece model kimlik bilgileri orada kalır. Agent modu günlükleri, köprü başlangıcında çözümlenen transkripsiyon sağlayıcısını/modelini ve sentezlenen her yanıttan sonra TTS sağlayıcısını/modelini/sesini/çıkış biçimini/örnekleme hızını içerir. Ham `mode: "realtime"`, `mode: "agent"` için eski bir uyumluluk diğer adı olarak hâlâ kabul edilir, ancak artık aracın `mode` enum'unda gösterilmez.

API destekli bir oda ve açık erişim politikasıyla `create`:

```json
{
  "action": "create",
  "transport": "chrome-node",
  "mode": "agent",
  "accessType": "OPEN"
}
```

Bilinen bir odanın etkin konferansını sonlandırma:

```json
{
  "action": "end_active_conference",
  "meeting": "https://meet.google.com/abc-defg-hij"
}
```

Bir toplantının yararlı olduğunu öne sürmeden önce dinleme öncelikli doğrulama:

```json
{
  "action": "test_listen",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "timeoutMs": 30000
}
```

İstek üzerine konuşma:

```json
{
  "action": "speak",
  "sessionId": "meet_...",
  "message": "Tam olarak şunu söyle: Buradayım ve dinliyorum."
}
```

`status`, kullanılabildiğinde Chrome durumunu içerir:

| Alan                                                                  | Anlamı                                                                                                                 |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `inCall`                                                              | Chrome, Meet aramasının içinde görünüyor                                                                               |
| `micMuted`                                                            | En iyi çabayla belirlenen Meet mikrofon durumu                                                                         |
| `manualActionRequired` / `manualActionReason` / `manualActionMessage` | Konuşmanın çalışabilmesi için tarayıcı profilinde manuel oturum açılması, Meet ana makinesi kabulü, izinler veya tarayıcı denetimi onarımı gerekiyor |
| `speechReady` / `speechBlockedReason` / `speechBlockedMessage`        | Yönetilen Chrome konuşmasına şu anda izin verilip verilmediği; `speechReady: false`, OpenClaw'ın giriş/test ifadesini göndermediği anlamına gelir |
| `providerConnected` / `realtimeReady`                                 | Gerçek zamanlı ses köprüsü durumu                                                                                      |
| `lastInputAt` / `lastOutputAt`                                        | Köprüden alınan/köprüye gönderilen son ses                                                                             |
| `audioOutputRouted` / `audioOutputDeviceLabel`                        | Meet sekmesinin medya çıkışının köprünün BlackHole aygıtına etkin biçimde yönlendirilip yönlendirilmediği              |
| `lastOutputLoopbackAt` / `outputLoopbackSignalBytes`                  | Dalga biçimi parmak izi BlackHole mikrofon yakalama yolunda ilişkilendirilen yeni çıkış                                 |
| `lastOutputLoopbackCorrelation`                                       | Yakalanan sinyali geçerli asistan çıkışı üretimine bağlayan korelasyon puanı                                            |
| `outputGeneration` / `verifiedOutputGeneration`                       | Monotonik kimlikler; eşitlik, eski bir ifade yerine geçerli çıkışın geri döngü kanıtından geçtiği anlamına gelir       |
| `lastOutputLoopbackRms` / `lastOutputLoopbackPeak`                    | En son doğrulanmış geri döngü yakalama parçasına ait ses enerjisi tanılamaları                                          |
| `lastSuppressedInputAt` / `suppressedInputBytes`                      | Asistan oynatımı etkinken geri döngü girişinin yok sayılması                                                           |

## Agent ve çift yönlü modlar

| Mod     | Yanıta kim karar verir          | Konuşma çıkış yolu                         | Kullanım amacı                                          |
| ------- | ------------------------------- | ------------------------------------------ | ------------------------------------------------------- |
| `agent` | Yapılandırılmış OpenClaw agent'ı | Normal OpenClaw TTS çalışma zamanı          | "Agent'ım toplantıda" davranışını istediğinizde         |
| `bidi`  | Gerçek zamanlı ses modeli       | Gerçek zamanlı ses sağlayıcısının ses yanıtı | En düşük gecikmeli konuşma ses döngüsünü istediğinizde  |

`agent` modu: Gerçek zamanlı transkripsiyon sağlayıcısı toplantı sesini dinler, nihai katılımcı transkriptleri yapılandırılmış OpenClaw agent'ı üzerinden yönlendirilir ve yanıt normal OpenClaw TTS aracılığıyla seslendirilir. Yakın nihai transkript parçaları danışma öncesinde birleştirilir; böylece tek bir konuşma sırası, birkaç eski kısmi yanıt üretmez. Kuyruktaki asistan sesi oynatılmaya devam ederken gerçek zamanlı giriş engellenir ve BlackHole geri döngüsünün agent'ın kendi konuşmasına yanıt vermesine yol açmaması için yakın tarihli, asistana benzer transkript yankıları danışma öncesinde yok sayılır.

`bidi` modu: Gerçek zamanlı ses modeli doğrudan yanıt verir ve daha derin akıl yürütme, güncel bilgiler veya normal OpenClaw araçları için `openclaw_agent_consult` çağrısı yapabilir. Danışma aracı, arka planda normal OpenClaw agent'ını yakın tarihli toplantı transkripti bağlamıyla çalıştırır ve kısa, seslendirilebilir bir yanıt döndürür; `agent` modunda OpenClaw bu yanıtı doğrudan TTS'ye gönderir, `bidi` modunda ise gerçek zamanlı ses modeli yanıtı seslendirebilir. Voice Call ile aynı paylaşılan danışma mekanizmasını kullanır.

Danışmalar varsayılan olarak `main` agent'ı üzerinde çalışır; bir Meet hattını özel bir agent çalışma alanına, model varsayılanlarına, araç politikasına, belleğe ve oturum geçmişine yönlendirmek için `realtime.agentId` ayarlayın. Agent modu danışmaları, takip sorularının normal agent politikasını devralırken toplantı bağlamını koruması için toplantı başına bir `agent:<id>:subagent:google-meet:<session>` oturum anahtarı kullanır. Bir agent, agent modunda `google_meet` çağrısı yaptığında danışman oturumu, katılımcı konuşmasını yanıtlamadan önce çağıranın geçerli transkriptinden çatallanır; Meet oturumu ayrı kalır, böylece toplantıdaki takip soruları çağıranın transkriptini doğrudan değiştirmez.

`realtime.toolPolicy`, danışma çalıştırmasını denetler:

| Politika           | Davranış                                                                                                                          |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | Danışma aracını kullanıma sunar; normal agent'ı `read`, `web_search`, `web_fetch`, `x_search`, `memory_search`, `memory_get` ile sınırlar |
| `owner` | Danışma aracını kullanıma sunar; normal agent'ın kendi normal araç politikasını kullanmasına izin verir                          |
| `none` | Danışma aracını gerçek zamanlı ses modeline sunmaz                                                                                |

Danışma oturum anahtarı Meet oturumu başına kapsamlandırılır; bu nedenle takip eden danışma çağrıları aynı toplantı sırasında önceki danışma bağlamını yeniden kullanır.

Chrome tamamen katıldıktan sonra sesli bir hazır olma kontrolünü zorunlu kılın:

```bash
openclaw googlemeet speak meet_... "Say exactly: I'm here and listening."
```

Tam katılma ve konuşma duman testi:

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "Say exactly: I'm here and listening."
```

## Canlı test kontrol listesi

Bir toplantıyı gözetimsiz bir agent'a devretmeden önce:

```bash
openclaw googlemeet setup
openclaw nodes status
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "Say exactly: Google Meet speech test complete."
```

Beklenen Chrome-node durumu:

- `googlemeet setup` tamamen yeşildir ve Chrome-node varsayılan aktarım olduğunda veya bir node sabitlendiğinde `chrome-node-connected` içerir.
- `nodes status`, seçili node'un bağlı olduğunu ve hem `googlemeet.chrome` hem de `browser.proxy` özelliklerini duyurduğunu gösterir.
- Meet sekmesi katılır ve `test-speech`, `inCall: true` ile Chrome durumunu döndürür.

Parallels macOS VM gibi uzak bir Chrome ana makinesi için Gateway veya VM güncellendikten sonraki en kısa güvenli denetim:

```bash
openclaw googlemeet setup
openclaw nodes status --connected
openclaw nodes invoke \
  --node parallels-macos \
  --command googlemeet.chrome \
  --params '{"action":"setup"}'
```

Bu, bir temsilci gerçek bir toplantı sekmesi açmadan önce Gateway Plugin'inin yüklendiğini, VM node'unun güncel token ile bağlı olduğunu ve Meet ses köprüsünün kullanılabilir olduğunu doğrular.

Twilio duman testi için telefonla katılım bilgilerini sunan bir toplantı kullanın:

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

Beklenen Twilio durumu:

- `googlemeet setup`; yeşil `twilio-voice-call-plugin`, `twilio-voice-call-credentials` ve `twilio-voice-call-webhook` denetimlerini içerir.
- `voicecall`, Gateway yeniden yüklendikten sonra CLI'da kullanılabilir.
- Döndürülen oturumda `transport: "twilio"` ve bir `twilio.voiceCallId` bulunur.
- `openclaw logs --follow`, gerçek zamanlı TwiML'den önce DTMF TwiML'in sunulduğunu, ardından ilk karşılamanın kuyruğa alındığı bir gerçek zamanlı köprüyü gösterir.
- `googlemeet leave <sessionId>`, devredilen sesli aramayı sonlandırır.

## Sorun giderme

### Temsilci Google Meet aracını göremiyor

Plugin'in etkin olduğunu doğrulayın ve Gateway'i yeniden yükleyin; çalışan temsilci yalnızca mevcut Gateway işlemi tarafından kaydedilmiş Plugin araçlarını görür:

```bash
openclaw plugins list | grep google-meet
openclaw googlemeet setup
```

macOS dışındaki Gateway ana makinelerinde `google_meet` görünür kalır, ancak yerel Chrome geri konuşma eylemleri ses köprüsüne ulaşmadan engellenir. Varsayılan yerel Chrome temsilcisi yolu yerine `mode: "transcribe"`, Twilio telefonla katılımı veya macOS `chrome-node` ana makinesi kullanın.

### Bağlı, Google Meet özellikli node yok

Node ana makinesinde:

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw plugins enable browser
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

Gateway ana makinesinde:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Node bağlı olmalı ve `googlemeet.chrome` ile `browser.proxy` öğelerini listelemelidir; Gateway yapılandırması her ikisine de izin vermelidir:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["browser.proxy", "googlemeet.chrome"] },
    },
  },
}
```

`googlemeet setup`, `chrome-node-connected` denetiminde başarısız olursa veya Gateway günlüğü `gateway token mismatch` bildirirse node'u güncel Gateway token'ı ile yeniden kurun ya da yeniden başlatın:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install \
  --host <gateway-lan-ip> \
  --port 18789 \
  --display-name parallels-macos \
  --force
```

Ardından node hizmetini yeniden yükleyip şunları tekrar çalıştırın:

```bash
openclaw googlemeet setup
openclaw nodes status --connected
```

### Tarayıcı açılıyor ancak temsilci katılamıyor

Yalnızca gözlem amaçlı katılımlar için `googlemeet test-listen`, gerçek zamanlı katılımlar için `googlemeet test-speech` çalıştırın; ardından döndürülen Chrome durumunu inceleyin. Bunlardan biri `manualActionRequired: true` bildirirse operatöre `manualActionMessage` gösterin ve tarayıcı eylemi tamamlanana kadar yeniden denemeyi bırakın.

Yaygın manuel eylemler: Chrome profiline giriş yapın; konuğu Meet ana makine hesabından kabul edin; yerel istem göründüğünde Chrome mikrofon/kamera izinlerini verin; takılı kalan Meet izin iletişim kutusunu kapatın veya düzeltin.

Meet yalnızca "Do you want people to hear you in the meeting?" diye sorduğu için "oturum açılmamış" bildiriminde bulunmayın; bu, Meet'in ses seçimi ara ekranıdır. OpenClaw, kullanılabilir olduğunda tarayıcı otomasyonu aracılığıyla **Use microphone** öğesine tıklar ve gerçek toplantı durumunu beklemeye devam eder; yalnızca oluşturma amaçlı tarayıcı geri dönüşünde bunun yerine **Continue without microphone** öğesine tıklayabilir çünkü URL oluşturmak için gerçek zamanlı ses yolu gerekmez.

### Toplantı oluşturma başarısız oluyor

`googlemeet create`, OAuth yapılandırılmışsa Meet API `spaces.create` öğesini, aksi takdirde sabitlenmiş Chrome node tarayıcısını kullanır. Şunları doğrulayın:

- **API ile oluşturma**: `oauth.clientId` ve `oauth.refreshToken` (veya eşleşen `OPENCLAW_GOOGLE_MEET_*` ortam değişkenleri) mevcuttur ve yenileme token'ı oluşturma desteği eklendikten sonra üretilmiştir; eski token'larda `meetings.space.created` bulunmayabilir, bu nedenle `openclaw googlemeet auth login --json` komutunu yeniden çalıştırın.
- **Tarayıcı geri dönüşü**: `defaultTransport: "chrome-node"` ve `chromeNode.node`, `browser.proxy` ile `googlemeet.chrome` özelliklerine sahip bağlı bir node'u gösterir; bu node'daki OpenClaw Chrome profilinde oturum açılmıştır ve profil `https://meet.google.com/new` öğesini açabilir.
- **Tarayıcı geri dönüşü yeniden denemeleri**: yeni bir sekme açmadan önce mevcut bir `.../new` veya Google hesabı istem sekmesini yeniden kullanın; manuel olarak başka bir sekme açmak yerine araç çağrısını yeniden deneyin.
- **Manuel eylem**: araç `manualActionRequired: true` döndürürse operatörü yönlendirmek için `browser.nodeId`, `browser.targetId`, `browserUrl` ve `manualActionMessage` kullanın; döngü içinde yeniden denemeyin.
- **Ses seçimi ara ekranı**: Meet "Do you want people to hear you in the meeting?" mesajını gösterirse sekmeyi açık bırakın. OpenClaw, **Use microphone** veya (yalnızca oluşturma için) **Continue without microphone** öğesine tıklayıp oluşturulan URL'yi beklemeye devam etmelidir; bunu yapamazsa hata `google-login-required` değil, `meet-audio-choice-required` belirtmelidir.

### Temsilci katılıyor ancak konuşmuyor

```bash
openclaw googlemeet setup
openclaw googlemeet doctor
```

STT -> OpenClaw temsilcisi -> TTS yolu için `mode: "agent"`, doğrudan gerçek zamanlı ses geri dönüşü için `mode: "bidi"` kullanın. `mode: "transcribe"` kasıtlı olarak geri konuşma köprüsü başlatmaz. Yalnızca gözlem amaçlı hata ayıklamada katılımcılar konuştuktan sonra `openclaw googlemeet status --json <session-id>` çalıştırın ve `captioning`, `transcriptLines`, `lastCaptionText` öğelerini denetleyin. `inCall` doğruysa ancak `transcriptLines`, `0` olarak kalıyorsa Meet altyazıları devre dışı olabilir, gözlemci kurulduğundan beri kimse konuşmamış olabilir, Meet kullanıcı arayüzü değişmiş olabilir veya toplantı dili/hesabı için canlı altyazılar kullanılamıyor olabilir.

`googlemeet test-speech` her zaman gerçek zamanlı yolu denetler ve bu çağrı için köprü çıkış baytlarının gözlemlenip gözlemlenmediğini bildirir. `speechOutputVerified` yanlış ve `speechOutputTimedOut` doğruysa gerçek zamanlı sağlayıcı ifadeyi kabul etmiş olabilir ancak OpenClaw, Chrome ses köprüsüne ulaşan yeni çıkış baytlarını görmemiştir.

Şunları da doğrulayın: Gateway ana makinesinde bir gerçek zamanlı sağlayıcı anahtarı (`OPENAI_API_KEY` veya `GEMINI_API_KEY`) kullanılabilir; Chrome ana makinesinde `BlackHole 2ch` görünür; orada `sox` bulunur; Meet mikrofonu/hoparlörü sanal ses yolu üzerinden yönlendirilir (`doctor`, yerel Chrome gerçek zamanlı katılımları için `meet output routed: yes` göstermelidir).

`googlemeet doctor [session-id]`; oturumu, node'u, aramadaki durumu, manuel eylem nedenini, gerçek zamanlı sağlayıcı bağlantısını, `realtimeReady`, ses giriş/çıkış etkinliğini, son ses zaman damgalarını, bayt sayaçlarını ve tarayıcı URL'sini yazdırır. Ham JSON için `googlemeet status [session-id] --json`, token'ları açığa çıkarmadan OAuth yenilemesini doğrulamak için `googlemeet doctor --oauth` (`--meeting` veya `--create-space` ekleyin) kullanın.

Bir temsilci zaman aşımına uğradıysa ve bir Meet sekmesi zaten açıksa başka bir sekme açmadan inceleyin:

```bash
openclaw googlemeet recover-tab
openclaw googlemeet recover-tab https://meet.google.com/abc-defg-hij
```

Eşdeğer araç eylemi `recover_current_tab`'dir: seçili aktarım için mevcut bir Meet sekmesine odaklanır ve sekmeyi (`chrome` için yerel tarayıcı denetimi, `chrome-node` için yapılandırılmış node) yeni bir sekme veya oturum açmadan inceler ve mevcut engeli (giriş, kabul, izinler, ses seçimi durumu) bildirir. CLI komutu, çalışıyor olması gereken yapılandırılmış Gateway ile iletişim kurar; `chrome-node` ayrıca node'un bağlı olmasını gerektirir.

### Twilio kurulum denetimleri başarısız oluyor

`voice-call` izinli veya etkin olmadığında `twilio-voice-call-plugin` başarısız olur: bunu `plugins.allow` öğesine ekleyin, `plugins.entries.voice-call` öğesini etkinleştirin ve Gateway'i yeniden yükleyin.

Twilio arka ucunda hesap SID'si, kimlik doğrulama token'ı veya arayan numarası eksik olduğunda `twilio-voice-call-credentials` başarısız olur:

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
```

`voice-call` herkese açık Webhook erişimine sahip olmadığında veya `publicUrl` geri döngü/özel ağ alanını gösterdiğinde `twilio-voice-call-webhook` başarısız olur. `publicUrl` olarak `localhost`, `127.0.0.1`, `0.0.0.0`, `10.x`, `172.16.x`-`172.31.x`, `192.168.x`, `169.254.x`, `fc00::/7` veya `fd00::/8` kullanmayın; operatör geri çağrıları bunlara ulaşamaz. `plugins.entries.voice-call.config.publicUrl` değerini herkese açık bir URL olarak ayarlayın veya bir tünel/Tailscale erişimi yapılandırın:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          fromNumber: "+15550001234",
          publicUrl: "https://voice.example.com/voice/webhook",
        },
      },
    },
  },
}
```

Yerel geliştirmede özel bir ana makine URL'si yerine tünel veya Tailscale erişimi kullanın:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tunnel: { provider: "ngrok" },
          // veya
          tailscale: { mode: "funnel", path: "/voice/webhook" },
        },
      },
    },
  },
}
```

Gateway'i yeniden başlatın veya yeniden yükleyin, ardından:

```bash
openclaw googlemeet setup --transport twilio
openclaw voicecall setup
openclaw voicecall smoke
```

`voicecall smoke` varsayılan olarak yalnızca hazır olma durumunu denetler. Belirli bir numara için deneme çalıştırması yapın:

```bash
openclaw voicecall smoke --to "+15555550123"
```

Yalnızca kasıtlı olarak gerçek bir giden arama yapmak için `--yes` ekleyin:

```bash
openclaw voicecall smoke --to "+15555550123" --yes
```

### Twilio araması başlıyor ancak toplantıya hiç girmiyor

Meet etkinliğinin telefonla katılım bilgilerini sunduğunu doğrulayın ve tam katılım numarasıyla birlikte PIN'i veya özel bir DTMF dizisini iletin:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

PIN'den önce duraklatmak için `--dtmf-sequence` içinde başta `w` veya virgüller kullanın.

Arama oluşturulduğu hâlde Meet katılımcı listesinde telefonla katılan kişi hiç görünmüyorsa:

- `openclaw googlemeet doctor <session-id>`: devredilen Twilio arama kimliğini, DTMF'nin kuyruğa alınıp alınmadığını ve giriş karşılamasının istenip istenmediğini doğrulayın.
- `openclaw voicecall status --call-id <id>`: aramanın hâlâ etkin olduğunu doğrulayın.
- `openclaw voicecall tail`: Twilio Webhook'larının Gateway'e ulaştığını doğrulayın.
- `openclaw logs --follow`: Twilio Meet sırasını arayın: Google Meet katılımı devreder, Voice Call bağlantı öncesi DTMF TwiML'i depolayıp sunar, Voice Call Twilio araması için gerçek zamanlı TwiML sunar, ardından Google Meet `voicecall.speak` ile giriş konuşmasını ister.
- `openclaw googlemeet setup --transport twilio` komutunu yeniden çalıştırın; yeşil kurulum denetimi gereklidir ancak toplantı PIN dizisinin doğru olduğunu kanıtlamaz.
- Telefonla katılım numarasının PIN ile aynı Meet davetine ve bölgesine ait olduğunu doğrulayın.
- Meet yavaş yanıt veriyorsa veya bağlantı öncesi DTMF gönderildikten sonra arama dökümü hâlâ PIN istemini gösteriyorsa `voiceCall.dtmfDelayMs` değerini 12 saniyelik varsayılandan artırın.
- Katılımcı katıldığı hâlde karşılamayı duymuyorsanız DTMF sonrası `voicecall.speak` isteği ve medya akışı TTS oynatımı ya da Twilio `<Say>` geri dönüşü için `openclaw logs --follow` öğesini denetleyin. Döküm hâlâ "enter the meeting PIN" gösteriyorsa telefon bağlantısı henüz Meet odasına katılmamıştır; bu nedenle katılımcılar konuşmayı duymaz.

Webhook'lar ulaşmıyorsa önce Voice Call plugininde hata ayıklayın: sağlayıcının `plugins.entries.voice-call.config.publicUrl` adresine veya yapılandırılmış tünele erişebilmesi gerekir. Bkz. [Sesli arama sorunlarını giderme](/tr/plugins/voice-call#troubleshooting).

## Notlar

Google Meet'in resmi medya API'si alım odaklıdır; bu nedenle bir aramada konuşmak için yine de bir katılımcı yolu gerekir. Bu plugin, söz konusu sınırı görünür tutar: Chrome, tarayıcı katılımını ve yerel ses yönlendirmesini; Twilio ise telefonla katılımı yönetir.

Chrome geri konuşma modları için `BlackHole 2ch` ile birlikte aşağıdakilerden biri gerekir:

- `chrome.audioInputCommand` ve `chrome.audioOutputCommand`: Köprünün sahibi OpenClaw'dur ve sesi bu komutlarla seçilen sağlayıcı arasında `chrome.audioFormat` biçiminde aktarır. `agent` modu, gerçek zamanlı transkripsiyon ile normal TTS'yi; `bidi` modu ise gerçek zamanlı ses sağlayıcısını kullanır. Varsayılan yol, `chrome.audioBufferBytes: 4096` ile 24 kHz PCM16'dır; eski komut çiftleri için 8 kHz G.711 mu-law seçeneği kullanılabilir durumda kalır.
- `chrome.audioBridgeCommand`: Harici bir köprü komutu, yerel ses yolunun tamamının sahibidir ve arka plan programını başlattıktan veya doğruladıktan sonra sonlanmalıdır. Yalnızca `bidi` için geçerlidir; çünkü `agent` modu, TTS için komut çiftine doğrudan erişim gerektirir.

Komut çiftli Chrome köprüsünde `chrome.bargeInInputCommand`, ayrı bir yerel mikrofonu dinleyebilir ve bir kişi konuşmaya başladığında asistan oynatımını temizleyebilir. Böylece paylaşılan BlackHole geri döngü girişi, asistan oynatımı sırasında geçici olarak bastırılsa bile insan konuşmasına asistan çıktısından önce öncelik verilir. `chrome.audioInputCommand`/`chrome.audioOutputCommand` gibi bu da operatör tarafından yapılandırılan yerel bir komuttur: açıkça belirtilmiş, güvenilir bir komut yolu veya bağımsız değişken listesi kullanın; güvenilmeyen bir konumdaki betiği asla kullanmayın.

Temiz çift yönlü ses için Meet çıkışını ve Meet mikrofonunu ayrı sanal cihazlar ya da Loopback tarzı bir sanal cihaz grafiği üzerinden yönlendirin; tek bir paylaşılan BlackHole cihazı, diğer katılımcıların sesini aramaya geri yansıtabilir.

`googlemeet speak`, bir Chrome oturumu için etkin geri konuşma ses köprüsünü tetikler; `googlemeet leave` bunu durdurur (ve Voice Call üzerinden devredilen Twilio oturumlarında temel aramayı sonlandırır). API tarafından yönetilen bir alanın etkin Google Meet konferansını da kapatmak için `googlemeet end-active-conference` kullanın.

## İlgili

- [Toplantı pluginlerine genel bakış](/tr/plugins/meeting-plugins)
- [Sesli arama plugini](/tr/plugins/voice-call)
- [Konuşma modu](/tr/nodes/talk)
- [Plugin oluşturma](/tr/plugins/building-plugins)
