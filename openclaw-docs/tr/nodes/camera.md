---
read_when:
    - Node platformlarında kamera yakalama özelliği ekleme veya değiştirme
    - Ajanların erişebildiği MEDIA geçici dosya iş akışlarını genişletme
summary: Fotoğraflar ve kısa video klipleri için iOS, Android, macOS ve Linux Node'larında kamera çekimi
title: Kamera çekimi
x-i18n:
    generated_at: "2026-07-26T22:50:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b819f7ff3fc9b51757ae998d27f540975bf6c1194ed32fd36b1fbe909e79400c
    source_path: nodes/camera.md
    workflow: 16
---

OpenClaw, eşleştirilmiş **iOS**, **Android**, **macOS** ve **Linux** Node'larında ajan iş akışları için kamera çekimini destekler: Gateway `node.invoke` aracılığıyla fotoğraf (`jpg`) veya isteğe bağlı ses içeren kısa bir video klip (`mp4`) çekilebilir.

Tüm kamera erişimi, her platformda kullanıcı tarafından denetlenen bir ayarla sınırlandırılır.

## iOS Node'u

### iOS kullanıcı ayarı

- iOS Settings sekmesi → **Camera** → **Allow Camera** (`camera.enabled`).
  - Varsayılan: **açık** (anahtarın bulunmaması etkin olarak değerlendirilir).
  - Kapalıyken: `camera.*` komutları `CAMERA_DISABLED` döndürür.

### iOS komutları (Gateway `node.invoke` aracılığıyla)

- `camera.list`
  - Yanıt yükü: `devices` — `{ id, name, position, deviceType }` dizisi.

- `camera.snap`
  - Parametreler:
    - `facing`: `front|back` (varsayılan: `front`)
    - `maxWidth`: sayı (isteğe bağlı; varsayılan `1600`)
    - `quality`: `0..1` (isteğe bağlı; varsayılan `0.9`, `[0.05, 1.0]` ile sınırlandırılır)
    - `format`: şu anda `jpg`
    - `delayMs`: sayı (isteğe bağlı; varsayılan `0`, dahili olarak `10000` ile sınırlandırılır)
    - `deviceId`: dize (isteğe bağlı; `camera.list` kaynağından)
  - Yanıt yükü: `format: "jpg"`, `base64`, `width`, `height`.
  - Yük koruması: base64 ile kodlanmış yükü 5MB altında tutmak için fotoğraflar yeniden sıkıştırılır.

- `camera.clip`
  - Parametreler:
    - `facing`: `front|back` (varsayılan: `front`)
    - `durationMs`: sayı (varsayılan `3000`, `[250, 60000]` ile sınırlandırılır)
    - `includeAudio`: mantıksal değer (varsayılan `true`)
    - `format`: şu anda `mp4`
    - `deviceId`: dize (isteğe bağlı; `camera.list` kaynağından)
  - Yanıt yükü: `format: "mp4"`, `base64`, `durationMs`, `hasAudio`.

### iOS ön plan gereksinimi

`canvas.*` gibi, iOS Node'u da `camera.*` komutlarına yalnızca **ön planda** izin verir. Arka plan çağrıları `NODE_BACKGROUND_UNAVAILABLE` döndürür.

### CLI yardımcısı

Medya dosyalarını edinmenin en kolay yolu, kodu çözülmüş medyayı geçici bir dosyaya yazan ve kaydedilen yolu yazdıran CLI yardımcısını kullanmaktır.

```bash
openclaw nodes camera snap --node <id>                 # varsayılan: ön + arka birlikte (2 MEDIA satırı)
openclaw nodes camera snap --node <id> --facing front
openclaw nodes camera clip --node <id> --duration 3000
openclaw nodes camera clip --node <id> --no-audio
```

`nodes camera snap` varsayılan olarak `--facing both` değerini kullanır ve ajana her iki görünümü de sağlamak için hem ön hem arka kamerayla çekim yapar; tek bir açık yön değeriyle `--device-id` iletin (`--device-id` ayarlandığında `both` reddedilir). Kendi sarmalayıcınızı oluşturmadığınız sürece çıktı dosyaları geçicidir (işletim sisteminin geçici dizininde).

## Android Node'u

### Android kullanıcı ayarı

- Android Settings sayfası → **Camera** → **Allow Camera** (`camera.enabled`).
  - **Yeni kurulumlarda varsayılan olarak kapalıdır.** Bu ayardan daha eski mevcut kurulumlar, yükseltmelerin daha önce çalışan kamera erişimini sessizce kaybetmemesi için **açık** durumuna geçirilir.
  - Kapalıyken: `camera.*` komutları `CAMERA_DISABLED: enable Camera in Settings` döndürür.

### İzinler

- `CAMERA`, hem `camera.snap` hem de `camera.clip` için gereklidir; eksik veya reddedilmiş izin `CAMERA_PERMISSION_REQUIRED` döndürür.
- `includeAudio` değeri `true` olduğunda `camera.clip` için `RECORD_AUDIO` gereklidir; eksik veya reddedilmiş izin `MIC_PERMISSION_REQUIRED` döndürür.

Uygulama, mümkün olduğunda çalışma zamanı izinlerini ister.

### Android ön plan gereksinimi

`canvas.*` gibi, Android Node'u da `camera.*` komutlarına yalnızca **ön planda** izin verir. Arka plan çağrıları `NODE_BACKGROUND_UNAVAILABLE: command requires foreground` döndürür.

### Android komutları (Gateway `node.invoke` aracılığıyla)

- `camera.list`
  - Yanıt yükü: `devices` — `{ id, name, position, deviceType }` dizisi.

- `camera.snap`
  - Parametreler: `facing` (`front|back`, varsayılan `front`), `quality` (varsayılan `0.95`, `[0.1, 1.0]` ile sınırlandırılır), `maxWidth` (varsayılan `1600`), `deviceId` (isteğe bağlı; bilinmeyen kimlik `INVALID_REQUEST` hatasıyla sonuçlanır).
  - Yanıt yükü: `format: "jpg"`, `base64`, `width`, `height`.
  - Yük koruması: base64 verisini 5MB altında tutmak için yeniden sıkıştırılır (iOS ile aynı bütçe).

- `camera.clip`
  - Parametreler: `facing` (varsayılan `front`), `durationMs` (varsayılan `3000`, `[200, 60000]` ile sınırlandırılır), `includeAudio` (varsayılan `true`), `deviceId` (isteğe bağlı).
  - Yanıt yükü: `format: "mp4"`, `base64`, `durationMs`, `hasAudio`.
  - Yük koruması: ham MP4, base64 kodlamasından önce 18MB ile sınırlandırılır; sınırı aşan klipler `PAYLOAD_TOO_LARGE` hatasıyla sonuçlanır (`durationMs` değerini azaltıp yeniden deneyin).

## macOS uygulaması

### macOS kullanıcı ayarı

macOS yardımcı uygulaması bir onay kutusu sunar:

- **Settings → General → Allow Camera** (`openclaw.cameraEnabled`).
  - Varsayılan: **kapalı**.
  - Kapalıyken: kamera istekleri `CAMERA_DISABLED: enable Camera in Settings` döndürür.

### CLI yardımcısı (Node çağrısı)

macOS Node'unda kamera komutlarını çağırmak için ana `openclaw` CLI'ını kullanın.

```bash
openclaw nodes camera list --node <id>                     # kamera kimliklerini listeler
openclaw nodes camera snap --node <id>                     # kaydedilen yolu yazdırır
openclaw nodes camera snap --node <id> --max-width 1280
openclaw nodes camera snap --node <id> --delay-ms 2000
openclaw nodes camera snap --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --duration 10s       # kaydedilen yolu yazdırır
openclaw nodes camera clip --node <id> --duration-ms 3000   # kaydedilen yolu yazdırır (eski bayrak)
openclaw nodes camera clip --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --no-audio
```

- `openclaw nodes camera snap`, geçersiz kılınmadığı sürece varsayılan olarak `maxWidth=1600` değerini kullanır.
- `camera.snap`, ısınma/pozlama sabitlendikten sonra çekim yapmadan önce `delayMs` bekler (varsayılan 2000ms, `[0, 10000]` ile sınırlandırılır).
- Fotoğraf yükleri, base64 verisini 5MB altında tutmak için yeniden sıkıştırılır.

## Linux Node ana makinesi

Paketle gelen Linux Node Plugin'i, CLI `openclaw node` hizmetine kamera çekimi ekler. Ekransız bir ana makinede çalışır ve Linux masaüstü uygulamasını gerektirmez.

Kamera erişimi varsayılan olarak kapalıdır. Plugin girdisi altında etkinleştirin, ardından Gateway duyurusunun yeniden oluşturulması için Node hizmetini yeniden başlatın:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          camera: { enabled: true },
        },
      },
    },
  },
}
```

Gereksinimler:

- V4L2 girdisi, `libx264` ve AAC desteği bulunan FFmpeg
- Node hizmeti kullanıcısının okuyabildiği bir `/dev/video*` aygıtı; yaygın dağıtımlarda bu kullanıcıyı `video` grubuna ekleyin
- varsayılan `includeAudio: true` ile çekilen klipler için çalışan bir PulseAudio sunucusu veya varsayılan kaynağa sahip PipeWire PulseAudio uyumluluk katmanı

Linux, `camera.list` üzerinden çekim yapabilen ve okunabilir V4L2 aygıt yollarını döndürür; FFmpeg her `/dev/video*` adayını yoklar ve meta veri ya da yalnızca çıktı Node'larını dışarıda bırakır. Aygıt `position`, `unknown` olduğundan `deviceId` olmadan yapılan yön istekleri, ön veya arka kamera olduğunu iddia etmek yerine `unknown` konumunda tek bir fotoğraf ya da klip üretir. Bir ana makinede birden fazla kamera varsa `deviceId` kullanın. `camera.snap`, `delayMs` için FFmpeg girdi ısınmasını kullanır ve genişliği sınırlarken en-boy oranını korur. `camera.clip`, mikrofon sesini MP4 ses parçası olarak kaydeder; OpenClaw bilinçli olarak bağımsız bir mikrofon komutu sunmaz.

Plugin, MP4 video için `libx264` kullanır ve codec'leri sessizce değiştirmez. Gerekli girdi veya kodlayıcılara sahip olmayan bir FFmpeg derlemesi `CAMERA_UNAVAILABLE` döndürür. 25MB base64 yük bütçesini aşacak fotoğraflar ve klipler `PAYLOAD_TOO_LARGE` hatasıyla sonuçlanır.

`camera.snap` ve `camera.clip` tehlikeli komutlar olmaya devam eder. Bunları `gateway.nodes.commands.allow` listesine yalnızca çekimi etkinleştirmeyi amaçladığınızda ekleyin; yalnızca Plugin'i etkinleştirmek Gateway politikasını atlamaz.

## Güvenlik ve pratik sınırlar

- Kamera ve mikrofon erişimi, olağan işletim sistemi izin istemlerini tetikler (ve `Info.plist` içinde kullanım dizeleri gerektirir).
- Aşırı büyük Node yüklerini önlemek için video klipleri 60s ile sınırlandırılır (base64 ek yükü ve mesaj sınırları).

## macOS ekran videosu (işletim sistemi düzeyinde)

Kamera değil, _ekran_ videosu için macOS yardımcı uygulamasını kullanın:

```bash
openclaw nodes screen record --node <id> --duration 10s --fps 15   # kaydedilen yolu yazdırır
```

macOS **Screen Recording** izni (TCC) gerektirir.

## İlgili

- [Görüntü ve medya desteği](/tr/nodes/images)
- [Medya anlama](/tr/nodes/media-understanding)
- [Konum komutu](/tr/nodes/location-command)
