---
read_when:
    - Ses katmanı davranışını ayarlama
summary: Uyandırma sözcüğü ve bas-konuş çakıştığında ses katmanı yaşam döngüsü
title: Ses katmanı
x-i18n:
    generated_at: "2026-07-26T23:28:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eef571c3e8d41a97779537b1b373fab25b08f63575b50e5019f6c5fbcb782c52
    source_path: platforms/mac/voice-overlay.md
    workflow: 16
---

# Ses Katmanı Yaşam Döngüsü (macOS)

Hedef kitle: macOS uygulamasına katkıda bulunanlar. Amaç: uyandırma sözcüğü ile bas-konuş çakıştığında ses katmanının öngörülebilir davranmasını sağlamak.

## Davranış

- Katman uyandırma sözcüğü nedeniyle zaten görünür durumdaysa ve kullanıcı kısayol tuşuna basarsa, kısayol tuşu oturumu metni sıfırlamak yerine mevcut metni devralır. Kısayol tuşu basılı tutulduğu sürece katman açık kalır. Tuş bırakıldığında: kırpılmış metin varsa gönderilir, yoksa kapatılır.
- Yalnızca uyandırma sözcüğü kullanıldığında sessizlikte otomatik gönderim devam eder; bas-konuş, tuş bırakıldığında hemen gönderir.

## Uygulama

- `VoiceSessionCoordinator` (`apps/macos/Sources/OpenClaw/VoiceSessionCoordinator.swift`), etkin ses oturumunun tek sahibidir. Bir aktör değil, `@MainActor @Observable` tekil örneğidir. API: `startSession`, `updatePartial`, `finalize`, `sendNow`, `dismiss`, `updateLevel`, `snapshot`. Her oturum bir `UUID` belirteci taşır; eski veya eşleşmeyen bir belirteçle yapılan çağrılar yok sayılır.
- `VoiceWakeOverlayController` (`VoiceWakeOverlayController+Session.swift`), katmanı işler ve kullanıcı eylemlerini (`requestSend`, `dismiss`) oturum belirteci aracılığıyla koordinatöre geri iletir. Oturum durumunun sahibi hiçbir zaman kendisi değildir.
- Bas-konuş (`VoicePushToTalk.begin()`), görünür katmandaki tüm metni (`VoiceSessionCoordinator.shared.snapshot()` aracılığıyla) `adoptedPrefix` olarak devralır; böylece uyandırma katmanı açıkken kısayol tuşuna basılması metni korur ve yeni konuşmayı metne ekler. Tuş bırakıldığında, mevcut metni kullanmaya geçmeden önce son transkript için en fazla 1.5s bekler.
- `dismiss` gerçekleştiğinde katman, `VoiceSessionCoordinator.overlayDidDismiss` çağrısını yapar; bu da `VoiceWakeRuntime.refresh(state:)` işlemini tetikler. Böylece X ile elle kapatma, boş metin nedeniyle kapatma ve gönderim sonrası kapatma durumlarının tümünde uyandırma sözcüğünü dinleme yeniden başlatılır.
- Birleşik gönderim yolu: kırpılmış metin boşsa kapatılır; aksi hâlde `sendNow` gönderim sesini bir kez çalar, `VoiceWakeForwarder` aracılığıyla iletir ve ardından katmanı kapatır.

## Günlük Kaydı

Ses alt sistemi `ai.openclaw`; her bileşen kendi kategorisi altında günlük kaydı oluşturur:

| Kategori                | Bileşen                                       |
| ----------------------- | ----------------------------------------------- |
| `voicewake.coordinator` | `VoiceSessionCoordinator`                       |
| `voicewake.overlay`     | `VoiceWakeOverlayController`/`VoiceWakeOverlay` |
| `voicewake.ptt`         | Bas-konuş kısayol tuşu ve yakalama                 |
| `voicewake.runtime`     | Uyandırma sözcüğü çalışma zamanı                               |
| `voicewake.chime`       | Uyarı sesi çalma                                  |
| `voicewake.sync`        | Genel ayarları eşitleme                            |
| `voicewake.forward`     | Transkript iletme                           |
| `voicewake.meter`       | Mikrofon seviyesi izleyicisi                               |

## Hata ayıklama kontrol listesi

- Takılı kalan bir katmanı yeniden oluştururken günlükleri akış hâlinde görüntüleyin:

  ```bash
  sudo log stream --predicate 'subsystem == "ai.openclaw" AND category CONTAINS "voicewake"' --level info --style compact
  ```

- Yalnızca bir etkin oturum belirteci olduğunu doğrulayın; eski geri çağırmalar koordinatör tarafından yok sayılır.
- Bas-konuş tuşu bırakıldığında etkin belirteçle her zaman `end()` çağrısının yapıldığını doğrulayın; metin boşsa uyarı sesi çalınmadan ve gönderim yapılmadan katmanın kapatılması beklenir.

## İlgili

- [macOS uygulaması](/tr/platforms/macos)
- [Sesle uyandırma (macOS)](/tr/platforms/mac/voicewake)
- [Konuşma modu](/tr/nodes/talk)
