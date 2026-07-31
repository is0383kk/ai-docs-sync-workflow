---
read_when:
    - Sesle uyandırma veya PTT yolları üzerinde çalışma
summary: Mac uygulamasında sesle uyandırma ve bas-konuş modlarının yanı sıra yönlendirme ayrıntıları
title: Sesle uyandırma (macOS)
x-i18n:
    generated_at: "2026-07-26T23:25:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d3b2a01ee997b4158bf88b9ef54b1e523503722620f943d594323516619e7502
    source_path: platforms/mac/voicewake.md
    workflow: 16
---

# Sesle Uyandırma ve Bas-Konuş

## Gereksinimler

Sesle Uyandırma ve bas-konuş için macOS 26 veya daha yeni bir sürüm gerekir. Daha eski macOS sürümlerinde denetimler Ses ayarları sayfasında gizlenir ve bunların yerine macOS 26 gereksinimi gösterilir.

Sesle Uyandırma, seçilen dilde aygıt üzerinde tanıma desteği için Apple Speech'ü gerektirir. Yalnızca yerel çalışma sözleşmesi kullanılamadığında uygulama pasif uyandırma sözcüğü dinlemesini başlatmayı reddeder; hiçbir zaman ağ üzerinden tanımaya geri dönmez. Bas-konuş, Konuşma Modu ve Hızlı Sohbet diktesi açık kullanıcı eylemleridir ve daha geniş dil kapsamı için Apple Speech ağ hizmetlerini kullanabilir.

## Modlar

- **Uyandırma sözcüğü modu** (varsayılan): Her zaman açık, aygıt üzerinde çalışan Speech tanıyıcısı tetikleyici belirteçleri bekler (`swabbleTriggerWords`). Eşleşme olduğunda kaydı başlatır, kısmi metinle kaplamayı gösterir ve sessizlikten sonra otomatik olarak gönderir.
- **Bas-konuş (sağ Option tuşunu basılı tutun)**: Tetikleyici gerekmeksizin hemen kaydetmek için sağ Option tuşunu basılı tutun. Tuş basılı tutulurken kaplama görünür; tuşu bırakmak, metni düzenleyebilmeniz için kısa bir gecikmenin ardından kaydı sonlandırıp iletir.

## Çalışma zamanı davranışı (uyandırma sözcüğü)

- Tanıyıcı `VoiceWakeRuntime` içinde bulunur.
- Tetikleyici yalnızca uyandırma sözcüğü ile sonraki sözcük arasında anlamlı bir duraklama olduğunda devreye girer (`triggerPauseWindow` = 0.55s). Kaplama/zil sesi, komut başlamadan önce bile duraklama sırasında başlayabilir.
- Sessizlik aralıkları: konuşma devam ederken 2.0s (`silenceWindow`), yalnızca tetikleyici duyulduysa 5.0s (`triggerOnlySilenceWindow`).
- Kesin durdurma: Denetimsiz oturumları önlemek için 120s (`captureHardStop`).
- Oturumlar arası bekleme: Gönderimden sonra 350ms (`debounceAfterSend`).
- Kaplama, kalıcı/geçici metin renklendirmesiyle `VoiceWakeOverlayController` üzerinden yönetilir.
- Gönderimden sonra tanıyıcı, sonraki tetikleyiciyi dinlemek üzere temiz bir şekilde yeniden başlatılır.

## Yaşam döngüsü değişmezleri

- Sesle Uyandırma etkinleştirilmiş ve izinler verilmişse uyandırma sözcüğü tanıyıcısı, etkin bir bas-konuş kaydı dışında dinlemeye devam eder.
- X düğmesiyle elle kapatma dâhil olmak üzere kaplamanın kapatılması, tanıyıcıyı her zaman devam ettirir: `VoiceSessionCoordinator.overlayDidDismiss`, her kapatma yolunda `VoiceWakeRuntime.refresh(state:)` çağrısını yapar. Oturum/belirteç modeli için [Ses kaplaması](/tr/platforms/mac/voice-overlay) bölümüne bakın.

## Bas-konuş ayrıntıları

- Kısayol tuşu algılama, sağ Option için genel bir `.flagsChanged` izleyicisi kullanır (`keyCode 61` + `.option`). Yalnızca olayları gözlemler, onları hiçbir zaman engellemez.
- Kayıt `VoicePushToTalk` içinde gerçekleşir: Speech'ü hemen başlatır, kısmi sonuçları kaplamaya aktarır ve tuş bırakıldığında `VoiceWakeForwarder` çağrısını yapar.
- Bas-konuşun başlatılması, çakışan ses bağlantılarını önlemek için uyandırma sözcüğü çalışma zamanını duraklatır; tuş bırakıldıktan sonra otomatik olarak yeniden başlatılır.
- İzinler: Mikrofon + Konuşma gerekir; tuş olaylarını almak için Erişilebilirlik/Giriş İzleme onayı gerekir.
- Harici klavyeler: Bazıları sağ Option tuşunu beklendiği gibi sunmaz. Kullanıcılar algılama sorunları bildirirse alternatif bir kısayol sunun.

## Kullanıcıya yönelik ayarlar

- **Sesle Uyandırma** anahtarı: Uyandırma sözcüğü çalışma zamanını etkinleştirir.
- **Konuşmak için sağ Option tuşunu basılı tutun**: Bas-konuş izleyicisini etkinleştirir.
- Seçilen dil bu Mac'te aygıt üzerinde tanımayı desteklemiyorsa Bas-Konuş ve Konuşma Modu kullanılabilir kalırken Sesle Uyandırma devre dışı kalır.
- Dil ve mikrofon seçicileri, canlı seviye ölçer, tetikleyici sözcük tablosu ve test aracı (yalnızca yerel çalışır, hiçbir zaman iletmez).
- Mikrofon seçici, bir aygıtın bağlantısı kesilirse son seçimi korur, bağlantının kesildiğine dair bir ipucu gösterir ve aygıt geri dönene kadar geçici olarak sistem varsayılanını kullanır.
- **Sesler**: Tetikleyici algılandığında ve gönderimde zil sesi çalar; varsayılan olarak macOS "Glass" sistem sesi kullanılır. Her olay için `NSSound` tarafından yüklenebilen herhangi bir dosyayı (ör. MP3/WAV/AIFF) seçin veya **Ses Yok** seçeneğini belirleyin.

## İletme davranışı

- İletme sırasında `VoiceWakeForwarder.selectedSessionOptions`, ayarlanmışsa etkin WebChat oturum anahtarını, aksi takdirde gateway'in ana oturum anahtarını seçer.
- Bu oturumu `sessions.list` aracılığıyla bulur ve teslim kanalını ve hedefi oturumun teslim bağlamından türetir (önce son kanalına/hedefine, ardından ayrıştırılmış oturum anahtarına geri döner); hiçbir şey çözümlenemezse varsayılan olarak WebChat kullanılır.
- Teslim başarısız olursa hata günlüğe kaydedilir (`voicewake.forward` kategorisi) ve çalıştırma WebChat/oturum günlükleri üzerinden yine de görülebilir.

## İletme yükü

- `VoiceWakeForwarder.prefixedTranscript(_:)`, uyandırma sözcüğü ve bas-konuş yolları arasında paylaşılan transkriptin önüne bir makine ipucu satırı (çözümlenen ana makine adı; bulunamazsa "bu Mac") ekler.

## Hızlı doğrulama

- Bas-konuşu açın, sağ Option tuşunu basılı tutun, konuşun ve bırakın: Kaplama önce kısmi sonuçları göstermeli, ardından göndermelidir.
- Tuş basılı tutulurken menü çubuğu kulakları büyütülmüş durumda kalmalıdır (`triggerVoiceEars(ttl: nil)`); tuş bırakıldıktan sonra küçülürler.

## İlgili

- [Sesle uyandırma](/tr/nodes/voicewake)
- [Ses kaplaması](/tr/platforms/mac/voice-overlay)
- [macOS uygulaması](/tr/platforms/macos)
