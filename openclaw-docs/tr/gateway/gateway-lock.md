---
read_when:
    - Gateway sürecini çalıştırma veya hata ayıklama
    - Tek örnek zorunluluğunun araştırılması
summary: 'Gateway tekil örnek koruması: dosya kilidi ve WebSocket/HTTP bağlama'
title: Gateway kilidi
x-i18n:
    generated_at: "2026-07-26T23:19:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f5ac6d42c437b481c68a23a0aa4c00aeac9131acd76f3516ce3e949f325e265b
    source_path: gateway/gateway-lock.md
    workflow: 16
---

## Neden

- Bir durum dizininin sahibi yalnızca bir gateway işlemi olmalıdır; ek gateway'leri yalıtılmış profiller, durum dizinleri, yapılandırmalar ve bağlantı noktalarıyla çalıştırın.
- Eski kilit dosyaları bırakmadan çökmelerden/SIGKILL'den sonra çalışmaya devam edin.
- Başka bir gateway bağlantı noktasının zaten sahibiyse anlaşılır bir hatayla hızla başarısız olun.

## Üç katman

Başlatma, sahipliği sırasıyla üç adımda zorunlu kılar:

1. **Durum sahipliği kilidi**, standart durum dizinine göre anahtarlanmış bir kilit alır. `OPENCLAW_ALLOW_MULTI_GATEWAY=1` ile başlatılan Gateway'ler dâhil tüm Gateway'ler buna katılır; böylece yıkıcı SQLite bakımı çalışan bir sahiple yarışamaz.
2. **Yapılandırma kilidi**, geçmişten gelen yapılandırma başına kilidi alır ve çalışma zamanı bağlantı noktasını kaydeder. Çoklu Gateway modu bu tekil yapılandırma kısıtlamasını atlar ancak durum sahipliği kilidini korur.
3. **Soket bağlama**, HTTP/WebSocket dinleyicisini (varsayılan `ws://127.0.0.1:18789`) özel bir TCP dinleyicisi olarak bağlar.

Her katman bağımsız olarak başarısız olabilir ve kendi `GatewayLockError` hatasını oluşturur.

### Durum ve yapılandırma kilitleri

- Kilidin geçerliliği; kaydedilen PID'den, mevcut olduğunda platform işlem başlangıç kimliğinden ve Gateway işlem kimliğinden belirlenir. Doğrulanmış bir sahip, bağlantı noktası dinlemeye başlamadan önceki başlatma sürecinde yetkili olmaya devam eder.
- Özel bir SQLite koordinatörü; meta veri incelemesini, eski sahiplerin geri alınmasını ve kilit değiştirmeyi seri hâle getirir. Sahip işlem çökerse özel işlemi otomatik olarak serbest bırakılır.
- Bir kilit dosyası eksikse veya kaydedilen sahip işlem sona ermişse başlatma kilidi geri alır ve devam eder.
- Kilitlerden biri etkin olarak tutuluyorsa başlatma, vazgeçmeden önce en fazla 5 saniye (varsayılan) yeniden dener:

  ```text
  GatewayLockError("gateway zaten çalışıyor (pid <pid>); <ms>ms sonra kilit zaman aşımına uğradı")
  ```

### Soket bağlama

- `EADDRINUSE` durumunda başlatma, yakın zamanda sonlanan bir işlemden sonraki `TIME_WAIT` aralığını aşmak için bağlamayı 500ms aralıklarla en fazla 20 kez (toplam yaklaşık 10 saniye) yeniden dener.
- Yeniden denemelerden sonra bağlantı noktası hâlâ kullanımdaysa:

  ```text
  GatewayLockError("başka bir gateway örneği zaten ws://127.0.0.1:<port> üzerinde dinliyor")
  ```

- Diğer bağlama hataları:

  ```text
  GatewayLockError("gateway soketi ws://127.0.0.1:<port> üzerinde bağlanamadı: <cause>")
  ```

Kapatma sırasında gateway, HTTP/WebSocket sunucusunu kapatır ve durum
ile yapılandırma kilit dosyalarını kaldırır.

## İşletim notları

- Bağlantı noktası gateway olmayan farklı bir işlem tarafından kullanılıyorsa hata aynıdır; bağlantı noktasını boşaltın veya `openclaw gateway --port <port>` ile başka bir bağlantı noktası seçin.
- `OPENCLAW_ALLOW_MULTI_GATEWAY=1`, paylaşılan değiştirilebilir duruma değil, birden fazla yapılandırma/çalışma zamanı örneğine izin verir. Her örnek yine benzersiz bir `OPENCLAW_STATE_DIR` gerektirir.
- Bir hizmet denetleyicisi altında, yukarıdaki hatalardan biriyle karşılaşan yeni gateway işlemi önce mevcut işlemde `/healthz` yoklaması yapar. Bu işlem sağlıklıysa yeni işlem başarısız olmak yerine denetimi ona bırakır. systemd üzerinde `78` koduyla çıkar; birimin `RestartPreventExitStatus=78` ayarı, `Restart=always` öğesinin bir kilit veya `EADDRINUSE` çakışması nedeniyle döngüye girmesini önler. Mevcut işlem hiçbir zaman sağlıklı duruma gelmezse sistem durumu yoklamasının yeniden deneme süresi sınırlıdır ve başlatma, sonsuza dek döngüye girmek yerine yukarıdaki kilit hatasıyla başarısız olur.
- macOS uygulaması, gateway'i başlatmadan önce kendi hafif PID korumasını sürdürür; asıl çalışma zamanı zorlaması yukarıdaki dosya kilidi ve soket bağlamadır.

## İlgili

- [Birden Fazla Gateway](/tr/gateway/multiple-gateways) - benzersiz bağlantı noktalarıyla birden fazla örnek çalıştırma
- [Sorun Giderme](/tr/gateway/troubleshooting) - `EADDRINUSE` ve bağlantı noktası çakışmalarını tanılama
