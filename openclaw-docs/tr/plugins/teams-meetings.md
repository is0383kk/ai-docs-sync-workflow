---
read_when:
    - Bir OpenClaw agentının Microsoft Teams toplantısına katılmasını istiyorsunuz
    - Teams toplantısında geri konuşma için Chrome, BlackHole veya SoX'u yapılandırıyorsunuz
summary: 'Microsoft Teams toplantıları Plugin''i: iş veya tüketici toplantılarına Chrome tarayıcı konuğu olarak katılın'
title: Microsoft Teams toplantıları Plugin'i
x-i18n:
    generated_at: "2026-07-26T22:57:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f84e58d478185d026dd79a02a8500af48f51689ef6865d56badb0e27c6d2814
    source_path: plugins/teams-meetings.md
    workflow: 16
---

`teams-meetings` Plugin, OpenClaw Chrome profilinde Microsoft Teams bağlantılarına konuk olarak katılır. `teams.microsoft.com/l/meetup-join/...` altındaki iş bağlantılarını ve `teams.live.com/meet/...` altındaki tüketici bağlantılarını kabul eder. Toplantı oluşturmaz, telefonla bağlanmaz, Microsoft Graph'i çağırmaz veya ses/video kaydı almaz.

## Kurulum

Sesli yanıt, [Google Meet Plugin](/tr/plugins/google-meet) ile aynı yerel ses ön koşullarını kullanır: macOS, `BlackHole 2ch` sanal ses aygıtı ve SoX.

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Plugin varsayılan olarak dâhildir ve etkindir. Yalnızca özelleştirmek için bir girdi ekleyin, ardından kurulumu denetleyin:

```json5
{
  plugins: {
    entries: {
      "teams-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

Plugin'in etkin olmasını istemiyorsanız `openclaw plugins disable teams-meetings` komutunu çalıştırın.

```bash
openclaw teamsmeetings setup
openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'
```

Chrome, BlackHole ve SoX'u eşleştirilmiş bir macOS Node üzerinde çalıştırmak için `chromeNode.node` kullanın. Node, `teamsmeetings.chrome` ve `browser.proxy` işlemlerine izin vermelidir.

## Modlar

| Mod          | Davranış                                                                            |
| ------------ | ----------------------------------------------------------------------------------- |
| `agent`      | Gerçek zamanlı transkripsiyon, yapılandırılmış OpenClaw aracısına danışır; TTS yanıt verir. |
| `bidi`       | Gerçek zamanlı bir ses modeli doğrudan dinler ve yanıt verir.                        |
| `transcribe` | Canlı altyazı transkript anlık görüntüleriyle yalnızca gözlem amaçlı katılım.        |

OpenClaw'ın konuşmacılara göre ilişkilendirilmiş notları kalıcı hâle getirebilmesi için her modda toplantıya kabul edildikten sonra Teams canlı altyazıları etkinleştirilir. `transcript` eylemi, yalnızca `transcribe` oturumları için sınırlı canlı arabelleği döndürmeye devam eder. Ayrılma sırasında OpenClaw, kalıcı transkripti ve türetilmiş özeti paylaşılan durum veritabanında depolar; bunları [`openclaw transcripts`](/tr/cli/transcripts) ile listeleyin veya dışa aktarın.

Otomatik notlar varsayılan olarak etkindir. Kalıcı notları genel olarak devre dışı bırakmak için `transcripts.enabled: false` olarak ayarlayın; açıkça belirtilen `transcribe` modu yine yalnızca sınırlı canlı son bölümünü sunar.

## Konuk katılımı sınırları

Tarayıcı bağdaştırıcısı uygulama ara ekranını kapatır, konuk adını doldurur, kamerayı kapatır, mikrofonu seçilen moda göre yapılandırır ve katıl düğmesine tıklar. Görüşme içi durumda görüşmeyi sonlandırma denetimi kullanılır; lobi, kiracı oturum açma ve aygıt izni durumları, açıkça belirtilmiş manuel işlem nedenleri döndürür. Tüketici toplantısı başlatıcısının yönlendirmeleri ve Chrome tarafından gösterilen `BlackHole 2ch (Virtual)` etiketleri desteklenir.

Teams kiracı ilkesi oturum açmayı, e-posta doğrulamasını veya düzenleyicinin kabulünü zorunlu kılabilir. Bu adımı OpenClaw Chrome profilinde tamamlayın, ardından durumu veya konuşmayı yeniden deneyin. Plugin, kiracı ilkesini atlamaz.

Tüketici Teams web istemcisi; uygulama ara ekranı, konuk adı girişi, katılım öncesi mikrofon/kamera geçişleri, katılım, lobi kabulü, medya izinleri, görüşme içi durum algılama, canlı altyazılar, BlackHole giriş/çıkış yönlendirmesi, ayrılma ve görüşme sonrası algılama için canlı ortamda doğrulanmıştır. İş kiracıları farklı oturum açma, e-posta doğrulama, kabul ve ayrılma onayı ilkeleri uygulayabilir; bildirilen tüm manuel işlemleri OpenClaw Chrome profilinde tamamlayın.

## Araç ve Gateway yüzeyi

`teams_meetings` aracı; `join`, `leave`, `status`, `transcript` ve `speak` işlemlerini destekler. Gateway yöntemleri `teamsmeetings.*` ön ekini kullanır. Node komutu `teamsmeetings.chrome` şeklindedir.

## İlgili

- [Toplantı Plugin'lerine genel bakış](/plugins/meeting-plugins)
- [Microsoft Teams kanalı](/tr/channels/msteams)
