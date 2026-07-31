---
read_when:
    - Bir OpenClaw aracısının görüntülü toplantıya katılmasını istiyorsunuz
    - Google Meet, Microsoft Teams toplantıları ve Zoom toplantıları pluginleri arasında seçim yapıyorsunuz
    - Paylaşılan Chrome, BlackHole, SoX veya toplantı modu kurulumuna ihtiyacınız var
summary: Google Meet, Microsoft Teams veya Zoom toplantılarına katılımı seçme ve yapılandırma
title: Toplantı Pluginleri
x-i18n:
    generated_at: "2026-07-26T22:53:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f41488de018402e3d5cfd01fa5351cdb6107412477d5d54e2d9e186e0fc8ee94
    source_path: plugins/meeting-plugins.md
    workflow: 16
---

OpenClaw'ın Google Meet, Microsoft Teams toplantıları ve Zoom için ayrı Plugin'leri vardır. Üçü de Chrome üzerinden katılabilir, aynı katılım modlarını kullanabilir ve Chrome'u Gateway ana bilgisayarında veya eşleştirilmiş bir node üzerinde çalıştırabilir. Platform URL'leri, kurulum modelleri ve ek yetenekleri farklıdır.

Bu Plugin'ler toplantılara katılır. [Microsoft Teams kanalı](/tr/channels/msteams) gibi mesajlaşma kanallarından ve [Sesli arama Plugin'inden](/tr/plugins/voice-call) ayrıdırlar.

## Bir Plugin seçme

| Platform        | Plugin                                      | Kabul edilen toplantı bağlantıları                                                                                      | Kurulum                                    | Katılım yolları                                      | Platforma özgü yetenekler                                                                                |
| --------------- | ------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ----------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Google Meet     | [`google-meet`](/tr/plugins/google-meet)       | `meet.google.com/...`                                                                                       | npm veya ClawHub'dan yüklenir; varsayılan olarak etkindir | Yerel Chrome, eşleştirilmiş bir node üzerindeki Chrome veya Twilio ile telefonla katılım | Meet API veya oturum açılmış bir tarayıcı üzerinden toplantı oluşturabilir; OAuth ile desteklenen Meet yapılarını okuyabilir |
| Microsoft Teams | [`teams-meetings`](/plugins/teams-meetings) | `teams.microsoft.com/l/meetup-join/...` altındaki iş bağlantıları ve `teams.live.com/meet/...` altındaki bireysel kullanıcı bağlantıları | Dahildir; varsayılan olarak etkindir                    | Yerel Chrome veya eşleştirilmiş bir node üzerindeki Chrome                  | İş ve bireysel kullanıcı toplantılarına misafir olarak katılım                                                                     |
| Zoom            | [`zoom-meetings`](/plugins/zoom-meetings)   | `zoom.us/j/...` ve `example.zoom.us/j/...` gibi hesap alt alan adları                                      | Dahildir; varsayılan olarak etkindir                    | Yerel Chrome veya eşleştirilmiş bir node üzerindeki Chrome                  | Zoom Web App üzerinden misafir olarak katılım                                                                           |

Toplantı oluşturma, Google API yapıları veya Twilio telefon yolu gerektiğinde Google Meet'i seçin. Bu platformlarda tarayıcı üzerinden doğrudan misafir katılımı için Teams veya Zoom'u seçin. Teams ve Zoom Plugin'leri toplantı oluşturmaz, telefonla bağlanmaz, sağlayıcı API'sini çağırmaz veya ses/video kaydı yapmaz.

## Bir mod seçme

Üç Plugin aynı modları paylaşır:

| Mod         | Davranış                                                                                              | Ses gereksinimleri                                      |
| ------------ | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `agent`      | Gerçek zamanlı transkripsiyon yapılandırılmış OpenClaw agent'ına gider; standart OpenClaw TTS yanıtı seslendirir.  | Chrome üzerinden sesli yanıt için BlackHole ve SoX köprüsü gerekir. |
| `bidi`       | Gerçek zamanlı bir ses modeli dinler ve doğrudan yanıt verir.                                                  | Chrome üzerinden sesli yanıt için BlackHole ve SoX köprüsü gerekir. |
| `transcribe` | Yalnızca gözlem amacıyla katılır ve platform altyazı sağladığında sınırlı bir canlı altyazı transkripti sunar. | BlackHole veya SoX sesli yanıt köprüsü gerekmez.                   |

Agent'ın yalnızca toplantı metnine ihtiyacı olduğunda `transcribe` kullanın. Standart OpenClaw muhakemesi ve araçları için `agent` kullanın. Her etkileşimi standart agent üzerinden yönlendirmekten çok düşük gecikmeli doğrudan ses önemliyse `bidi` kullanın.

Sınırlı canlı transkript yalnızca `transcribe` modunda kullanılabilir. Üç
modun tamamında tarayıcı üzerinden katılım, tamamlanmış altyazı satırlarını ve bunlardan türetilen bir
özeti paylaşılan durum veritabanında kalıcı olarak saklar. Toplantıdan ayrılmak görünür
altyazıları sonlandırır ve özeti yazar; listelemek, incelemek veya dışa aktarmak için
[`openclaw transcripts`](/tr/cli/transcripts) kullanın. Bu kalıcı not yolu, canlı
agent danışma transkriptini değiştirmez veya ses/video kaydı oluşturmaz.

Otomatik notlar varsayılan olarak açıktır. Kalıcı notları genel olarak devre dışı
bırakmak için `transcripts.enabled: false` ayarlayın. Açıkça seçilmiş bir `transcribe` oturumu,
kalıcı satırlar yazmadan sınırlı canlı altyazı kuyruğunu korur. Altyazı kullanılabilirliği
yine toplantı platformuna, hesaba, dile ve toplantı sahibi politikasına bağlıdır.

## Chrome ve sesi hazırlama

Chrome, Gateway ana bilgisayarında veya eşleştirilmiş bir node üzerinde çalışabilir. Uzak bir Chrome node'u `browser.proxy` ile birlikte platform komutuna izin vermelidir:

| Plugin          | Node komutu           |
| --------------- | ---------------------- |
| Google Meet     | `googlemeet.chrome`    |
| Microsoft Teams | `teamsmeetings.chrome` |
| Zoom            | `zoommeetings.chrome`  |

Chrome üzerinden `agent` veya `bidi` modu için Chrome'u macOS üzerinde çalıştırın ve paylaşılan ses bağımlılıklarını aynı ana bilgisayara yükleyin:

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Chrome eşleştirilmiş bir node üzerinde çalışırken OpenClaw agent'ının ve model kimlik bilgilerinin sahibi yine Gateway ana bilgisayarıdır. `agent` modu için gerçek zamanlı bir transkripsiyon sağlayıcısı ve OpenClaw TTS, `bidi` modu içinse gerçek zamanlı bir ses sağlayıcısı yapılandırın. Platform kılavuzları sağlayıcı ve ses komutu seçeneklerini içerir.

## Plugin'leri yükleme veya devre dışı bırakma

Google Meet'i ayrı olarak yükleyin; yüklendikten sonra varsayılan olarak etkindir. Teams toplantıları ve Zoom, OpenClaw'a dahildir ve varsayılan olarak etkindir:

```bash
# Yalnızca Google Meet
openclaw plugins install npm:@openclaw/google-meet
```

Kullanmadığınız toplantı Plugin'lerini devre dışı bırakın:

```bash
openclaw plugins disable google-meet
openclaw plugins disable teams-meetings
openclaw plugins disable zoom-meetings
```

Plugin yönetim yolunuz Gateway'i otomatik olarak yeniden başlatmıyorsa Gateway'i yeniden başlatın. Ardından katılmadan önce platform kurulum denetimini çalıştırın.

## Doğrulama ve katılma

| Platform        | Kurulum denetimi                    | Katılma komutu                                                                  |
| --------------- | ------------------------------ | ----------------------------------------------------------------------------- |
| Google Meet     | `openclaw googlemeet setup`    | `openclaw googlemeet join 'https://meet.google.com/abc-defg-hij'`             |
| Microsoft Teams | `openclaw teamsmeetings setup` | `openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'` |
| Zoom            | `openclaw zoommeetings setup`  | `openclaw zoommeetings join 'https://zoom.us/j/1234567890'`                   |

Başarısız olan her kurulum denetimini ilgili aktarım ve mod için engelleyici olarak değerlendirin. Yalnızca gözlem amaçlı bir duman testi için `transcribe` modunu seçin ve altyazı metni beklemeden önce durumun devam eden bir görüşme oturumu bildirdiğini doğrulayın.

Sesli yanıt duman testlerinde, konuşmanın doğrulanması için oynatma komutunun baytları kabul etmesi yeterli değildir. Paylaşılan komut çifti köprüsü, mevcut çıktı üretiminden alınan sınırlı bir dalga biçimi parmak izini BlackHole mikrofon yakalama yolundan dönen sesle ilişkilendirir; yalnızca çıktı bayt sayacı ilerlediğinde veya ilgisiz katılımcı sesi bulunduğunda Google Meet, Teams ve Zoom `speechOutputVerified: true` bildirmez.

## Platform politikası istemlerini yönetme

Tarayıcı otomasyonu; standart misafir adı, katılım öncesi kamera ve mikrofon, katılma, görüşme içi ve ayrılma denetimlerini yönetir. Platform veya düzenleyici politikasını atlatmaz.

- Google Meet, Google oturumu açılmasını, toplantı sahibinin kabulünü veya bir tarayıcı izni kararını gerektirebilir.
- Microsoft Teams, kiracı oturumu açılmasını, e-posta doğrulamasını veya düzenleyici kabulünü gerektirebilir.
- Zoom, kimlik doğrulama, e-posta doğrulaması, parola, CAPTCHA tamamlama veya toplantı sahibinin kabulünü gerektirebilir; ayrıca bir hesap tarayıcıdan katılımı devre dışı bırakabilir.

Bir katılma veya durum sonucu `manualActionRequired` bildirdiğinde, yeniden denemeden önce bildirilen adımı aynı OpenClaw Chrome profilinde tamamlayın. Sürekli yeni sekmeler açmak hesap, kiracı, lobi veya CAPTCHA engelini çözmez.

Yalnızca operatörün agent ekleme yetkisine sahip olduğu toplantılara katılın. Yerel politika veya onay kuralları otomatik katılımın, transkripsiyonun ya da sentezlenmiş konuşmanın açıklanmasını gerektiriyorsa katılımcıları bilgilendirin.

## Discord sesli sohbeti

[Discord ses kanalları](/tr/channels/discord#voice-channels), tarayıcı toplantı otomasyonu olmadan yerel, yalnızca sesli ve gerçek zamanlı görüşme sağlar. OpenClaw bir ses kanalına katılabilir, dinleyebilir, etkileşimleri bir OpenClaw agent'ı veya gerçek zamanlı ses modeli üzerinden yönlendirebilir ve yanıtları seslendirebilir. İnsanlar aynı Discord kanalında video kullansa bile kamera videosu veya ekran paylaşımı göndermez ya da almaz; bu nedenle Discord sesi, dördüncü bir tarayıcı toplantı Plugin'i değil, ilişkili bir canlı görüşme yüzeyidir.

## Platform kılavuzları

- [Google Meet Plugin'i](/tr/plugins/google-meet)
- [Microsoft Teams toplantıları Plugin'i](/plugins/teams-meetings)
- [Zoom toplantıları Plugin'i](/plugins/zoom-meetings)
- [Plugin'leri yönetme](/tr/plugins/manage-plugins)
- [Tarayıcı denetimi](/tr/tools/browser)
