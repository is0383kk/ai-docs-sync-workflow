---
read_when:
    - OpenClaw'un etkin Mac'i tanımlamasını istiyorsunuz
    - Son giriş etkinliğinde veya etkin Node seçiminde hata ayıklıyorsunuz
    - Node bağlantı bildirimi yönlendirmesini anlamak istiyorsunuz
summary: En son kullandığınız Mac’i algılayın ve Node uyarılarını oraya yönlendirin
title: Etkin bilgisayar varlığı
x-i18n:
    generated_at: "2026-07-26T22:50:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3f1d1d0e98b1f3b7478cf80696dc693677b57897b07260cce30938e9187c314
    source_path: nodes/presence.md
    workflow: 16
---

Etkin bilgisayar varlığı, Gateway'e bağlı macOS node'larından hangisinin en son
fiziksel fare veya klavye girdisini aldığını bildirir. OpenClaw bu sinyali kullanarak
bir Mac'i `active` olarak işaretler, agente kararlı bir etkin node ipucu verir ve
node bağlantı uyarılarını bulunma olasılığınızın en yüksek olduğu bilgisayara yönlendirir.

Bu, Gateway istemcilerinin canlı listesi olan [sistem varlığından](/tr/concepts/presence)
ve bir mobil node'u bağlı olarak değerlendirmeden en son ne zaman uyandığını
kaydeden kalıcı `node.presence.alive` işaretlerinden ayrıdır.

## Gereksinimler

- OpenClaw macOS uygulaması eşleştirilmiş ve node modunda bağlıdır.
- **Settings -> Permissions -> Active computer detection** etkindir. Varsayılan olarak kapalıdır.
- İmzalı OpenClaw uygulamasına **Accessibility** izni verilmiştir.
- Bağlantı uyarıları için **Notifications** izni de verilmiştir ve
  Mac node'u `system.notify` özelliğini sunar.

Etkinlik bildirimi şu anda yerel macOS node'u tarafından uygulanmaktadır. iOS,
Android, watchOS ve başsız node ana makineleri bağlantı veya arka planda
son görülme durumunu bildirebilir, ancak etkin bilgisayar olarak belirlenmek için yarışmazlar.

## Etkin bilgisayarı kontrol etme

1. macOS uygulamasında **Settings -> Permissions** bölümünü açın,
   **Active computer detection** seçeneğini etkinleştirin ve macOS System Settings içinde **Accessibility** izni verin.
2. Mac node'unun bağlı olduğunu doğrulayın:

   ```bash
   openclaw nodes status --connected
   ```

3. Bu Mac'te fareyi hareket ettirin veya bir tuşa basın, ardından şunları çalıştırın:

   ```bash
   openclaw nodes status
   openclaw nodes describe --node <node-id-or-name>
   ```

En güncel uygun Mac `active` olarak işaretlenir. Durum çıktısı son girdiden
beri geçen süreyi gösterir; `describe`, `active`, `lastActiveAtMs` ve `presenceUpdatedAtMs` değerlerini sunar.
Etkinlik kasıtlı olarak birleştirildiğinden, yakın zamanda yapılan bir bildirimden
sonraki başka bir girdinin ekrana yansıması yaklaşık 15 saniye sürebilir.

## Etkinliğin varlığa dönüşme biçimi

macOS bildiricisi, HID sisteminin boşta kalma saatini iki saniyede bir örnekler.
Bir node bağlantısı hazır olduğunda bir kez bildirir, ardından daha yeni fiziksel
etkinliği en fazla 15 saniyede bir bildirir. Boştayken her üç dakikada bir
keepalive gönderir. Çok eski bir örneğin ileri kayarak yanlışlıkla en yeni
bilgisayar hâline gelmemesi için boşta kalma süresi 30 günle sınırlandırılmıştır.

**Active computer detection** seçeneğinin devre dışı bırakılması örneklemeyi durdurur
ve mevcut node bağlantısı üzerinden kimliği doğrulanmış bir temizleme olayı gönderir.
Gateway, bu Mac'in saklanan etkinlik zaman damgalarını hemen kaldırır ve etkin
bilgisayarı yeniden hesaplar; diğer node yetenekleri ve devam eden işler bağlı kalır.
Bağlı Gateway bu temizleme eyleminden önceki bir sürümse, bağlantı kesme temizliğinin
saklanan etkinliği kaldırabilmesi için Mac node'u bir kez yeniden bağlanır.

Gateway etkinliği yalnızca aşağıdakilerin tümü doğru olduğunda kabul eder:

- olay, söz konusu node kimliğinin geçerli kimliği doğrulanmış bağlantısına aittir;
- node etkin `accessibility: true` iznine sahiptir;
- yük, sınırlandırılmış bir tam sayı `idleSeconds` değeri içerir.

Gateway, `lastActiveAtMs` değerini türetmek için kendi gözlem zamanından
`idleSeconds` değerini çıkarır. Node tarafından sağlanan duvar saati zaman damgasına
asla güvenmez. Bağlı ve uygun Mac'ler arasında en yeni `lastActiveAtMs` kazanır;
eşitlik durumunda en son varlık güncellemesi kullanılır.

Varlık, işlem yerelinde ve bağlantıya bağlıdır. Geçerli oturumun bağlantısının
kesilmesi, aynı node kimliğini kullanan başka bir oturumla değiştirilmesi veya
Accessibility izninin iptal edilmesi, ilgili node'un etkinlik durumunu temizler
ve etkin Mac'i yeniden hesaplar.

## Gizlilik ve model bağlamı

Etkinlik paylaşımı varsayılan olarak kapalıdır ve kullanıcı arayüzü otomasyonunda
kullanılan Accessibility izninden ayrıdır. OpenClaw girdi içeriğini değil, boşta
kalma süresini gönderir. Tuş değerlerini, fare koordinatlarını, uygulama adlarını,
pencere başlıklarını veya ham girdi olaylarını göndermez. macOS bildiricisi
donanım HID durumunu okuduğundan, sentetik bilgisayar kontrolü olayları otomatik
bir Mac'in fiziksel olarak kullandığınız bilgisayar gibi görünmesine neden olmaz.

Sürekli etkinlik, modele yönelik sistem olayları oluşturmaz. Dinamik çalışma
zamanı satırı yalnızca kimliği doğrulanmış node kimliğini içerir:

```text
active_node=<node-id>
```

Prompt enjeksiyonunu ve önbellek dalgalanmasını önlemek için kesin zaman damgaları
ve node tarafından denetlenen görünen adlar prompt dışında tutulur. Agent güncel
ayrıntılara ihtiyaç duyduğunda bunun yerine `nodes` aracı
`node.list` veya `node.describe` değerini okuyabilir.

## Bağlantı uyarılarının yönlendirilme biçimi

Bir node, onaydan sonraki ilk başarılı Gateway el sıkışmasını tamamladığında
OpenClaw, bağlanan Mac'in ilk etkinlik örneğini gönderebilmesi için 750 milisaniye
bekler. Ardından en güncel etkinliğe sahip, bağlı ve bildirim özelliği bulunan
Mac'i dener.

- Birincil teslimat başarılı olursa uyarıyı başka hiçbir Mac almaz.
- Etkin Mac yoksa veya birincil teslimat başarısız olursa OpenClaw beş saniye
  bekler ve `system.notify` özelliğini sunan diğer tüm bağlı Mac'leri dener.
- Daha sonraki yeniden bağlantılar sessizdir. Gateway başarılı bağlantıyı
  eşleştirme meta verilerine kaydeder; böylece Gateway'in yeniden başlatılması,
  önceden bağlanmış her node için uyarıları yeniden oynatmaz.

Uyarılar, kimliği doğrulanmış node kimliğine bağlıdır. Aynı node'un yerine geçen
bir oturum, bekleyen ilk bağlantı uyarısını devralır; teslimat çalıştırıldığında
bu node artık bağlı değilse uyarı iptal edilir.

## Sorun giderme

| Belirti                                   | Kontrol                                                                                                                                                                |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Hiçbir satır `active` olarak işaretlenmiyor                 | Etkin bilgisayar algılamanın açık, yerel bir macOS node'unun bağlı ve `openclaw nodes describe --node <id>` çıktısında `permissions.accessibility: true` değerinin gösterildiğini doğrulayın.   |
| Yanlış Mac etkin kalıyor              | Bu Mac'i fiziksel olarak kullanın, birleştirme aralığını bekleyin ve ardından `openclaw nodes status` komutunu yeniden çalıştırın. Sentetik bilgisayar kontrolü eylemleri sayılmaz.                        |
| Son girdi verileri kayboluyor                | Mac bağlantısının kesilip kesilmediğini, node oturumunun değiştirilip değiştirilmediğini veya Accessibility izninin iptal edilip edilmediğini kontrol edin. Her koşul etkinliği kasıtlı olarak temizler.                       |
| Uyarı birden fazla Mac'te görünüyor         | Birincil teslimat kullanılamadığı veya başarısız olduğu için gecikmeli geri dönüş çalıştı. Etkin Mac'in bağlı olduğunu, bildirimlere izin verdiğini ve `system.notify` özelliğini sunduğunu doğrulayın. |
| Agent etkin Mac'ten söz etmiyor | Etkinlik değiştikten sonra yeni bir tur başlatın. Çalışma zamanı ipucu kararlı ve kompakttır; geçerli meta verilerin kesin değerleri için `nodes` aracını kullanın.                                    |

TCC kurtarma işlemi için [macOS izinleri](/tr/platforms/mac/permissions) bölümüne bakın.
Node bağlantısı ve komut hataları için [Node sorun giderme](/tr/nodes/troubleshooting)
bölümüne bakın.

## İlgili

- [Node'lar](/tr/nodes)
- [Node CLI](/tr/cli/nodes)
- [Sistem varlığı](/tr/concepts/presence)
- [Gateway protokolü](/tr/gateway/protocol#presence)
- [macOS uygulaması](/tr/platforms/macos)
