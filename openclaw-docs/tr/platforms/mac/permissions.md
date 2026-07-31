---
read_when:
    - Eksik veya takılı kalan macOS izin istemlerinde hata ayıklama
    - Node veya bir CLI çalışma zamanına Erişilebilirlik izni verilip verilmeyeceğine karar verme
    - macOS uygulamasını paketleme veya imzalama
    - Paket kimliklerini veya uygulama yükleme yollarını değiştirme
summary: macOS izin kalıcılığı (TCC) ve imzalama gereksinimleri
title: macOS izinleri
x-i18n:
    generated_at: "2026-07-27T00:04:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e561aa641e44fc1e1b95a3db244f31124e4e51d13ae709bee188d86054301e34
    source_path: platforms/mac/permissions.md
    workflow: 16
---

macOS izin yetkilendirmeleri kırılgandır. TCC, bir izin yetkilendirmesini uygulamanın kod imzası, paket tanımlayıcısı ve diskteki yoluyla ilişkilendirir. Bunlardan herhangi biri değişirse macOS uygulamayı yeni olarak değerlendirir ve istemleri kaldırabilir veya gizleyebilir.

## Kararlı izinler için gereksinimler

- Aynı yol: uygulamayı sabit bir konumdan çalıştırın (OpenClaw için `dist/OpenClaw.app`).
- Aynı paket tanımlayıcısı: OpenClaw'ın paket kimliği `ai.openclaw.mac`; bunun değiştirilmesi yeni bir izin kimliği oluşturur.
- İmzalı uygulama: imzasız veya geçici olarak imzalanmış derlemelerde izinler kalıcı olmaz.
- Tutarlı imza: yeniden derlemeler arasında imzanın kararlı kalması için gerçek bir Apple Development veya Developer ID sertifikası kullanın.

Geçici imzalar her derlemede yeni bir kimlik oluşturur. macOS önceki yetkilendirmeleri unutur ve eski girdiler temizlenene kadar istemler tamamen kaybolabilir.

## Node ve CLI çalışma zamanları için Erişilebilirlik yetkilendirmeleri

Genel bir `node` ikilisi yerine Erişilebilirlik iznini OpenClaw.app, Peekaboo.app veya kendi paket tanımlayıcısına sahip başka bir imzalı yardımcı uygulamaya vermeyi tercih edin.

macOS TCC, Erişilebilirlik iznini gördüğü işlemin kod kimliğine verir. Bir Homebrew, nvm, pnpm veya npm iş akışı, paylaşılan bir `node` yürütülebilir dosyasının Erişilebilirlik izni almasına neden olursa aynı yürütülebilir dosya üzerinden başlatılan herhangi bir JavaScript paketi GUI otomasyonu ayrıcalıklarını devralabilir.

Sistem Ayarları'ndaki bir `node` girdisini tek bir npm paketinin izni olarak değil, söz konusu Node çalışma zamanı için geniş kapsamlı bir izin olarak değerlendirin. Bu Node kurulumuyla başlatılan her betiğe ve pakete güvenmiyorsanız `node` için Erişilebilirlik izni vermekten kaçının.

Erişilebilirlik onayı etkinlik paylaşımını etkinleştirmez. **Settings -> Permissions -> Active computer detection**, sınırlı boşta kalma süresinin Gateway'inizle paylaşılmasını sağlayan, varsayılan olarak kapalı ayrı bir denetimdir. Bu denetimi kapatmak, Erişilebilirlik iznini iptal etmeden veya node bağlantısını kesmeden saklanan etkinliği temizler.

Yanlışlıkla `node` için Erişilebilirlik izni verdiyseniz bu girdiyi System Settings -> Privacy & Security -> Accessibility konumundan kaldırın. Ardından Erişilebilirlik iznini, kullanıcı arayüzü otomasyonuna sahip olması gereken imzalı uygulamaya veya yardımcı uygulamaya verin.

## İstemler kaybolduğunda kurtarma kontrol listesi

1. Uygulamadan çıkın.
2. System Settings -> Privacy & Security içindeki uygulama girdisini kaldırın.
3. Uygulamayı aynı yoldan yeniden başlatın ve izinleri yeniden verin.
4. İstem hâlâ görünmüyorsa TCC girdilerini `tccutil` ile sıfırlayıp yeniden deneyin.
5. Bazı izinler yalnızca macOS tamamen yeniden başlatıldıktan sonra tekrar görünür.

Örnek sıfırlamalar (OpenClaw'ın paket kimliği `ai.openclaw.mac` kullanılarak):

```bash
sudo tccutil reset Accessibility ai.openclaw.mac
sudo tccutil reset ScreenCapture ai.openclaw.mac
sudo tccutil reset AppleEvents
```

## Dosya ve klasör izinleri (Masaüstü/Belgeler/İndirilenler)

macOS, terminal ve arka plan işlemlerinin Masaüstü, Belgeler ve İndirilenler klasörlerine erişimini de kısıtlayabilir. Dosya okumaları veya dizin listelemeleri takılı kalırsa dosya işlemlerini gerçekleştiren aynı işlem bağlamına (örneğin Terminal/iTerm, LaunchAgent tarafından başlatılan uygulama veya SSH işlemi) erişim izni verin.

Geçici çözüm: klasör başına izin vermekten kaçınmak istiyorsanız dosyaları OpenClaw çalışma alanına (`~/.openclaw/workspace`) taşıyın.

İzinleri test ediyorsanız her zaman gerçek bir sertifikayla imzalayın. Geçici derlemeler yalnızca izinlerin önemli olmadığı hızlı yerel çalıştırmalar için kabul edilebilir.

## İlgili

- [macOS uygulaması](/tr/platforms/macos)
- [macOS imzalama](/tr/platforms/mac/signing)
