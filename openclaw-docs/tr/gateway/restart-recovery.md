---
read_when:
    - Gateway'i yeniden başlatmanın devam eden ajan çalışmalarını kaybettirip kaybettirmediğini öğrenmek istiyorsunuz
    - Bir ajan çalışması yeniden başlatma, çökme veya yapılandırmanın yeniden yüklenmesi nedeniyle kesintiye uğradı
    - Gateway yeniden çalışmaya başladıktan sonra otomatik oturum kurtarmada hata ayıklıyorsunuz
summary: 'Gateway yeniden başlatıldığında veya çöktüğünde neler korunur: kesintiye uğrayan ajan dönüşleri otomatik olarak devam eder, alt ajanlar ve arka plan görevleri kurtarılır, kuyruktaki iletiler gönderilir'
title: Yeniden başlatma sonrası kurtarma
x-i18n:
    generated_at: "2026-07-26T22:46:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bdea30f3a90697951f4f63a06897d2c1d936e5145138b47fed7d8ebd8b7187ad
    source_path: gateway/restart-recovery.md
    workflow: 16
---

Gateway'in yeniden başlatılması agent durumunun kaybolmasına neden olmaz. Konuşmalar, transkriptler,
zamanlanmış işler, arka plan görev kayıtları ve kuyruğa alınmış giden mesajların tümü
diskte tutulur; turun ortasında kesintiye uğrayan işler algılanır ve Gateway yeniden
çalışmaya başladıktan sonra otomatik olarak sürdürülür. Kurtarma her zaman etkindir ve
normalde manuel müdahale gerektirmez. Sürekli başarısız olan kurtarma işlemleri
sınırlandırılır ve inceleyene veya değiştirene kadar bir oturum karantinaya alınabilir.

Bu sayfa, yeniden başlatma sonrasında nelerin korunduğunu, kesintiye uğrayan işlerin nasıl
algılandığını ve otomatik sürdürmenin nasıl gerçekleştiğini açıklar.

## Yeniden başlatma sonrasında korunanlar

| Durum                         | Depolama                                     | Yeniden başlatmalar arasındaki davranış                              |
| ----------------------------- | ------------------------------------------- | ----------------------------------------------------------------------- |
| Konuşma geçmişi          | Agent başına SQLite veritabanı                   | Değişmez; oturumlar depolanan transkriptten devam eder                 |
| Kesintiye uğrayan ana oturum turu | Agent başına SQLite oturum satırı ve transkript | Başlatmadan birkaç saniye sonra otomatik olarak sürdürülür veya uzlaştırılır         |
| Alt agent çalıştırmaları                 | SQLite (paylaşılan durum veritabanı)              | Kayıt defteri açılışta geri yüklenir; kesintiye uğrayan çalıştırmalar sürdürülür                     |
| Arka plan görevleri              | SQLite (paylaşılan durum veritabanı)              | Açılışta uzlaştırılır; sahipsiz çalıştırmalar kurtarılır veya kayıp olarak işaretlenir              |
| Kuyruğa alınmış giden teslimatlar    | SQLite teslimat kuyruğu                       | Yeniden başlatmadan sonra boşaltılır; teslim edilmemiş yanıtlar yeniden denenir                  |
| Zamanlanmış (cron) işler         | SQLite cron deposu                           | Zamanlamalar korunur; zamanlayıcı açılışta yeniden kurulur                        |
| Yeniden başlatma devamı          | SQLite yeniden başlatma nöbetçisi                     | Yeniden başlatmayı isteyen oturuma tek seferlik takip gönderilir |

## Zarif yeniden başlatmalar önce mevcut işleri tamamlar

İstenen bir yeniden başlatma (`openclaw gateway restart`, yeniden başlatma gerektiren bir
yapılandırma değişikliği veya Gateway güncellemesi), devam eden işleri hemen sonlandırmaz. Gateway
yeni iş kabul etmeyi durdurur, ardından etkin agent turlarının ve arka plan
görevlerinin tamamlanmasını bir boşaltma bütçesi dâhilinde (varsayılan olarak 5 dakika) bekler. Bu nedenle
yeniden başlatmaların çoğu hiçbir şeyi kesintiye uğratmaz.

Yalnızca boşaltma bütçesi içinde tamamlanamayan işler (veya zorunlu yeniden başlatma
ya da çökme nedeniyle kesintiye uğrayan çalıştırmalar) iptal edilir ve bu gerçekleşmeden önce
etkilenen her oturum kurtarma için işaretlenir.

## Kesintiye uğrayan işler nasıl algılanır?

Birbirini tamamlayan üç mekanizma, turu tamamlanmayan oturumları işaretler:

- **Tur kabulünde:** mevcut bir ana oturumdaki sıradan metin turunda
  Gateway; model veya `before_agent_reply` kancası yürütülmeden önce kullanıcı mesajını ekler,
  oturumu çalışıyor olarak işaretler ve kurtarma teslimatı talebini tek bir SQLite
  işlemi içinde kaydeder. Control UI bunu `started` onayını döndürmeden önce
  yapar; kanal gönderimi ise hazırlanmış tur agent çalıştırmasını devraldığında
  yapar.
  Komutlar, ekler, tur başına geçersiz kılmalar, bekleyen teslimatlar, önceki iptal
  ipuçları, plugin tarafından yönetilen oturumlar ve yürütme kancalarına sahip turlar
  özelleştirilmiş kabul yollarını korur.
  Bir `before_agent_reply` kancası kuruluysa kabul işlemi, kancanın aşamasını da kaydeder.
  Kurtarma, çağrı sırasında kesintiye uğramış bir kancayı hiçbir zaman yeniden yürütmez.
  İşlenmemiş bir kanca tamamlandığında denetim noktası sonucu kaydeder, ancak bu kanca
  etkin kaldığı sürece kurtarma yine güvenli biçimde başarısız olur: bir denetim noktası,
  yeniden başlatma sonrasında aynı plugin kodunun ve yapılandırmasının yüklendiğini
  kanıtlayamaz. İşlenmiş metin ve sessiz sonuçlar, deterministik sonuçlandırma için
  ayrı ayrı denetim noktalarına kaydedilir.
  Eski sürümlerin yazdığı kalıcı kurtarma taleplerinde kaynak sahipliği işaretçisi
  bulunmadığından yükseltme sırasında aynı güvenli biçimde başarısız olan kanca
  denetimine tabi tutulurlar.
- **Kapatma sırasında:** yeniden başlatma boşaltması sırasında etkin bir çalıştırması
  bulunan her oturum, çalıştırma iptal edilmeden önce oturum deposunda bir kurtarma
  işaretçisiyle damgalanır.
- **Başlatma sırasında:** Gateway, hâlâ çalıştığını iddia eden ancak yeni süreçte
  canlı sahibi bulunmayan oturumları saptamak için oturum depolarını tarar. Bu işlem,
  hiçbir kapatma kodunun çalışmadığı sert çökmeleri ve sonlandırmaları yakalar. Eski
  transkript kilit dosyaları da aynı anda temizlenir.

## Otomatik sürdürme

Başlatmadan birkaç saniye sonra Gateway, işaretlenen her oturumu agent'a önceki turunun
yeniden başlatma nedeniyle kesintiye uğradığını ve mevcut transkriptten devam etmesi
gerektiğini bildiren yapay bir sistem mesajıyla yeniden gönderir. Nihai yanıt daha önce
üretilmiş ancak teslim edilmemişse metni eklenir; böylece agent işi yeniden yapmak yerine
yanıtı teslim edebilir.

Başlatma uzlaştırması, geçici hataları üstel geri çekilmeyle üç defaya kadar yeniden dener.
Ayrıca kesintiye uğrayan her ana oturum döngüsünün, Gateway yeniden başlatmaları boyunca
korunan üç ücretlendirilmiş otomatik gönderim denemesinden oluşan kalıcı bir bütçesi vardır.
OpenClaw, gönderimden önce bir denemeyi ücretlendirir; Gateway isteği kabul etmeden önce
açıkça reddettiğinde ücreti iade eder ve işin yeniden yürütülmesini önlemek için gönderim
sonrası sonucun belirsiz olduğu durumlarda ücreti korur. Oturumun hâlihazırda sahibi olan
ön plan işi sonuçlanana kadar otomatik kurtarma devre dışı kalır.

Kalıcı bütçe tükendikten sonra oturum sonsuza kadar döngüye girmek yerine mezar taşıyla
işaretlenir. Başarısız oturumu inceleyin ve yerine yenisini başlatmak için `/new`
veya `/reset` kullanın. `openclaw doctor --fix`, mezar taşıyla çakışan eski bir iptal
bayrağını onarabilir ancak bu kurtarma döngüsünü yeniden etkinleştirmez.

Her yeniden deneme aynı kalıcı gönderim tanımlayıcısını yeniden kullanır; bu nedenle belirsiz
bir bağlantı hatası aynı kurtarmayı iki kez başlatamaz. Tamamlanmış ve sürdürülemez Control UI
turları da sınırlı kalıcı eşgüçlülük mezar taşlarını korur; böylece yeniden bağlanan bir giden
kutusu, isteği yeniden yürütmeden bunları sonlandırabilir.

Yalnızca mesaj aracı kullanan yanıtlar ikinci bir kalıcı korelasyon kullanır. Aynı konuşmaya
yönelik sonlandırıcı gönderim kanala ulaşmadan önce Gateway, tam oturum ve kaynak tur üzerinde
çözümlenmemiş bir teslimat niyeti kaydeder. Doğrulanan sağlayıcı başarısı bunu kalıcı bir teslim
edildi makbuzuna dönüştürür; doğrulanan başarısızlık ise temizler. Kurtarma, araçları yeniden
çalıştırmadan teslim edilmiş bir makbuzu tamamlar. Bir çökme sağlayıcı sonucunu bilinmez
durumda bırakırsa kurtarma, harici bir etkiyi yeniden yürütmek yerine güvenli biçimde başarısız
olur.

Teslim edilen yanıt, kaynak mesaj kimliğiyle birlikte transkripte de yansıtılır. Sonlandırıcı
yansımalar farklı bir makbuz anahtarı kullanır; bu nedenle aynı sağlayıcı eşgüçlülük anahtarına
sahip bir ilerleme gönderimi sonlandırıcı işaretçiyi maskeleyemez. Önceki turlardan gelen
ilerleme gönderimleri ve makbuzlar mevcut turu tamamlayamaz. Mesaj eylemi yetkisini yalnızca
kalıcı kanal girişi talepleri geri yükleyebilir. Sürdürülen bir çalıştırma, istekte bulunanın
kimliği ve aynı kanal/ileti dizisi kısıtlamaları dâhil olmak üzere özgün kaynak teslimat modunu
ve kaynak korelasyonunu korur; böylece kurtarma sırasında başka bir yeniden başlatma
gerçekleşse bile aynı makbuz yetkili kalır. Yeniden oluşturulabilir kanal yetkisi bulunmayan,
yalnızca mesaj aracı kullanan bir tur güvenli biçimde başarısız olur ve tek seferlik yeniden
gönderme bildirimini alır.

Gateway, sürdürmeden önce transkript sonunun devam etmek için güvenli olduğunu denetler.
Güvenli değilse (örneğin tur, eskimiş bir bekleyen onayla sona erdiyse) oturum körlemesine
yeniden çalıştırılmaz; bunun yerine agent, kullanıcıdan son isteği yeniden göndermesini isteyen
kısa bir bildirim gönderir. WebChat için bu bildirim doğrudan oturum geçmişine yazılır; böylece
yeniden bağlandıktan sonra görünür kalır.

OpenClaw, kesintiye uğrayan salt okunur [Code Mode](/tr/tools/code-mode) çalışmalarını da yeniden
oluşturabilir. Code Mode, bu çalıştırmaları yeniden başlatmaya dayanıklı olarak işaretler ve
yan etkiye sahip katalog araçlarını veya plugin ad alanlarını yürütülmeden önce reddeder.
Yeniden başlatma `wait` denetiminde gerçekleşirse yeni Gateway, turu transkriptinden
yeniden oluşturur ve model bu bayrağı atlasa veya temizlese bile yeniden oluşturulan yürütmenin
yeniden başlatmaya dayanıklı kalmasını zorunlu kılar. Ana makine, yeniden başlatmadan sonra
Code Mode devre dışı bırakılsa bile yeniden oluşturulan turun tamamını denetlenmiş salt okunur
çekirdek araçları ve yeniden yürütülmesi açıkça güvenli plugin araçlarıyla sınırlar. Yan etkiye
sahip işler, yinelenen yazma riski yerine yeniden gönderme bildirimiyle korunmaya devam eder.

### Alt agent'lar

Alt agent çalıştırmaları paylaşılan SQLite durum veritabanında kalıcılaştırılır; bu nedenle
alt agent kayıt defteri süreç boyunca korunur. Açılışta kayıt defteri geri yüklenir ve kesintiye
uğrayan alt agent oturumları özgün görev bağlamlarıyla sürdürülür. İki güvenlik mekanizması
uygulanır:

- 2 saatten daha uzun süre önce kesintiye uğrayan çalıştırmalar sürdürülmek
  yerine sonlandırılır; böylece gece boyunca kapalı kalan bir Gateway eski işleri yeniden
  canlandırmaz.
- Kurtarma işlemi sürekli başarısız olan bir oturum, kurtarmanın sonsuza kadar
  döngüye girmemesi için sıkışmış olarak mezar taşıyla işaretlenir.

### Arka plan görevleri

[Arka plan görev kayıt defteri](/tr/automation/tasks) SQLite desteklidir ve açılışta ve düzenli
aralıklarla uzlaştırılır: tamamlanmış çalıştırmalar tarafından kaydedilen kalıcı sonuçlar
kurtarılır; sahibi olan süreç kaybolan çalıştırmalar ise sonsuza kadar askıda kalmak yerine
bir ek sürenin ardından kayıp olarak işaretlenir.

### Agent tarafından istenen yeniden başlatmalar

Agent'ın kendisi bir yeniden başlatmayı tetiklediğinde (bir yapılandırma değişikliğini uygulama,
Gateway'i güncelleme veya açık bir yeniden başlatma isteği), süreç sonlanmadan önce SQLite'a bir
yeniden başlatma nöbetçisi yazılır. Açılıştan sonra Gateway, sonucu kaynak sohbete geri gönderir
ve agent'ın tam olarak bıraktığı yerden, aynı kanal ve ileti dizisinde devam etmesi için tek
seferlik bir sürdürme turu gönderir.

Nöbetçinin türü belirlenmiş SQLite sütunları yeniden başlatma işlemleri için yetkili kaynaktır;
`payload_json` değeri yalnızca yeniden yürütme/hata ayıklama gölgesidir. Çalışma zamanı,
dosya geri dönüşü olmadan SQLite durumunu okur, yazar ve temizler. Depolama geçişi sırasında,
bir güncellemeden sonra eski süreç tarafından bırakılan doğrulanmış bir `restart-sentinel.json`
değerini korumak için başlatmada ve Doctor aracılığıyla sınırlı bir durum geçişi çalışır.
Geçiş, türü belirlenmiş satırı doğrular ve normal yeniden başlatma işlemleri devam etmeden önce
kaynak dosyayı kaldırır.

## Güvenlik mekanizmaları ve gözlemlenebilirlik

- **Çökme döngüsü kesicisi:** 5 dakika içinde 3 temiz olmayan açılış,
  bir sonraki açılışta otomatik başlatılan yan hizmetleri engelleyen bir kesiciyi tetikler;
  böylece çöken bir Gateway kendi etkisini büyütmez. Temiz olmayan açılış penceresi boşaldığında
  düzelir.
- **Ana oturum deneme bütçesi:** kesintiye uğrayan döngü başına üç
  ücretlendirilmiş otomatik gönderim denemesi; bütçenin tükenmesi, incelenip değiştirilene
  kadar bu oturumu mezar taşıyla işaretler.
- **Metrikler:** kurtarma etkinliği [Prometheus](/tr/gateway/prometheus)
  aracılığıyla `openclaw_session_recovery_total` ve `openclaw_session_recovery_age_seconds` olarak dışa aktarılır.
- **Günlükler:** kurtarma kararları `main-session-restart-recovery` ve
  `subagent-interrupted-resume` alt sistemleri altında günlüğe kaydedilir.

## Sürdürülmeyenler

- Başka bir sahip tarafından zaten yönetildikleri için ana oturum
  kurtarmasının dışında bırakılan oturumlar: alt agent oturumları (alt agent kurtarması),
  cron oturumları (zamanlayıcı, programa göre yeniden çalıştırır) ve ACP tarafından yönetilen
  oturumlar (sürdürmenin sahibi bağlı IDE veya istemcidir).
- Transkript sonundan güvenli bir şekilde devam edilemeyen oturumlar;
  bunlar sessizce yeniden çalıştırılmak yerine yukarıda açıklanan yeniden gönderme bildirimini
  alır.
- Hiç kabul edilmemiş işler: boşaltma penceresi sırasında gelen mesajlar,
  sonlanmakta olan bir sürecin kuyruğuna sessizce alınmak yerine açık bir yeniden başlatma
  hatasıyla reddedilir.
- Bağımsız gömülü turlar, Gateway'in yaşam döngüsü sahibini paylaşmadıkları
  için bekleyen yeniden başlatma kurtarması bulunan bir ana oturumu devralamaz. Turu Gateway
  üzerinden çalıştırın veya `/new` ya da `/reset` ile orada sıfırlayın.
