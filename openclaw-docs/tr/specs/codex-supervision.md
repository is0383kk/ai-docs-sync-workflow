---
read_when:
    - Codex oturum keşfi, devam ettirme veya arşiv davranışını tasarlama
    - Yerel oturum kataloğu kullanıcı arayüzünü veya Gateway RPC'lerini değiştirme
    - Eşleştirilmiş Node'lar genelinde Codex denetimini genişletme
summary: OpenClaw üzerinden yerel Codex oturumlarını denetlemeye yönelik mimari ve ürün sınırı.
title: Codex denetimi
x-i18n:
    generated_at: "2026-07-27T00:01:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5e259badc8f7fdec6fa093785a1dd04394e12287ae61f00474bcd45e7b95352d
    source_path: specs/codex-supervision.md
    workflow: 16
---

# Codex gözetimi

## Amaç

Codex gözetimi, bir OpenClaw operatörünün yerel Codex oturumlarını keşfetmesini ve,
güvenli olduğunda, normal OpenClaw Chat yüzeyi üzerinden yerel bir dal oluşturmasını sağlar.
Codex App Server, iş parçacığının ve model döngüsünün sahibi olmaya devam eder. OpenClaw;
filo kataloğunu, kimliği doğrulanmış operatör kullanıcı arayüzünü, oturum bağlamasını ve kanal teslimini sağlar.

Bu özellik, resmî `codex` pluginine aittir. Ayrı bir
Supervisor plugini veya ikinci bir Codex protokolü uygulaması yoktur.

## Ürün sınırı

Yerel oturum keşfi aşağıdaki ayarla açıkça devre dışı bırakılmadığı sürece, Codex plugini
etkin olduğunda katalog kaydedilir:

```text
plugins.entries.codex.config.sessionCatalog.enabled = false
```

Aracıya yönelik gözetim araçlarını şu ayarla etkinleştirin:

```text
plugins.entries.codex.config.supervision.enabled = true
```

Etkin ilk ürün, uzun vadeli filo planından bilinçli olarak daha sınırlıdır:

- Yalnızca arşivlenmemiş Codex iş parçacıklarını listeleyin.
- Yerel ve katılımı onaylanmış eşleştirilmiş Node satırlarını kararlı ana makine kimliğine göre gruplandırın.
- Depolanmış veya boşta olan Gateway yerel
  iş parçacığından normal, modeli kilitlenmiş bir Chat dalı oluşturun, ilk turda tam Codex donanımı iş parçacığını başlatın ya da önceki bir dal için
  oluşturulan Chat'i açın.
- Depolanmış veya boşta olan Gateway yerel iş parçacığını yalnızca
  başka çalıştırıcı olmadığı açıkça onaylandıktan sonra arşivleyin.
- Mevcut bir gözetimli Chat'in açılmasına izin vermeye devam ederken etkin yerel kaynakları yeni dal veya arşiv denetimleri olmadan
  gösterin.
- Ana kenar çubuğunda ana makine başına en yeni satırları gösterin, tam kataloğu
  oturumlar sayfasında tutun ve yerel ve eşleştirilmiş Node satırları için sınırlı, imleçle sayfalandırılmış döküm okumaları sağlayın.
- Katalog hatalarını ana makineye göre yalıtın.

Katalog, arşivlenmemiş koleksiyondur. Katalogdaki bir satırın tur durumu yine de
boşta, etkin, `notLoaded` veya hata olabilir.

Aracıya yönelik gözetim isteğe bağlı olmaya devam eder. Yönlendirmeli ilk katılım, yerel Codex kurulumu
algılandıktan ve seçilen çıkarım arka ucu canlı denetimini geçtikten sonra, kullanıcının
hangi birincil arka ucu seçtiğinden bağımsız olarak bunu kurmayı ve etkinleştirmeyi dener.
Gözetim yalnızca bu fırsatçı plugin kurulumu başarılı olduğunda
etkinleşir. Açıkça devre dışı bırakılmış bir plugin, politika engeli veya
`supervision.enabled: false`, gözetim araçları için belirleyici olmaya devam eder ancak
operatör oturum kataloğunu devre dışı bırakmaz. `sessionCatalog.enabled: false`,
operatör keşfini ve eşleştirilmiş Node katalog komutlarını devre dışı bırakır; Codex
sağlayıcısı ve donanımı etkin kalır.

## Sahiplik

`codex` plugini, tüm Codex App Server davranışlarının sahibidir:

- uç nokta keşfi ve bağlantı yaşam döngüsü
- protokol başlatma ve sürüm denetimleri
- iş parçacığı listeleme, okuma, sürdürme, arşivleme ve olay işleme
- onay ve kullanıcı girdisi köprüleri
- yerel iş parçacıklarının OpenClaw oturumlarına bağlanması
- devam ettirmeden sonra yalnızca Codex modelinin ve donanımının zorunlu kılınması

Control UI ve Gateway, pluginin sahip olduğu bu hizmeti kullanır. Codex yürütüm dosyalarını
doğrudan okumaz ve başka bir App Server istemcisi uygulamazlar.

Varsayılan yerel topoloji şöyledir:

```text
Codex Desktop -> özel stdio App Server -> kullanıcı Codex ana dizini
                                             ^
OpenClaw Codex plugini -> gözetim App Server bağlantısı
  (varsayılan olarak yönetilen kullanıcı ana dizini stdio'su kullanılır; açık appServer ayarlarına uyulur)
  -> pasif kaynak kataloğu ve okuma
  -> anlık görüntüyü sabitleme -> standart appServer kaynak dalı
  -> görünür geçmiş ekleme ve sonraki tüm gözetimli Chat turları

Sıradan OpenClaw Codex oturumları -> varsayılan olarak yönetilen aracı ana dizini stdio'su
  -> sıradan tam donanım iş parçacıkları -> OpenClaw Chat ve kanal teslimi
```

Gözetimin etkinleştirilmesi sıradan Codex donanımını değiştirmez: varsayılan olarak
aracı kapsamlı kalır. Ayrı gözetim bağlantısı varsayılan olarak
yönetilen kullanıcı ana dizini stdio'sunu kullanır; dolayısıyla kataloğu ve anlık görüntü işlemleri yerel
depolanmış iş parçacıklarını görür. Açık `appServer` bağlantı ayarlarına uyulur.
`homeScope` ayarlanmadığında gözetim bağlantısı, stdio
veya Unix için bunu `"user"`, WebSocket için `"agent"` olarak çözümler. `appServer.homeScope: "user"`
değerini yalnızca sıradan donanımın da yerel Codex ana dizinini paylaşması gerektiğinde
açıkça ayarlayın. Codex kenar çubuğu grubundan benimsenen bir Chat istisnadır: özel
gözetim bağlaması; kaynak okumalarını, standart dal oluşturmayı ve sonraki
turları gözetim bağlantısında tutar. Canlı durum ve sahiplik
işleme göre yereldir; OpenClaw'ın gözetim işleminin tanımadığı bir iş parçacığı, Codex Desktop onu etkin biçimde çalıştırıyor olsa bile `notLoaded`
olarak kabul edilir.

Codex'in, ayrı bir yükleyici tarafından yönetilen önyükleme sözleşmesine sahip deneysel, standart bir yerel daemon'ı vardır.
Bu özellik söz konusu daemon'ı örtük olarak önyüklememeli, sahiplenmemeli
veya varsaymamalıdır.

## Katalog akışı

Genel Gateway yöntemi `sessions.catalog.list`, her zaman `archived: false` isteyen ve App Server'ın
etkileşimli kaynak varsayılanını uygulamasına izin veren `codex`
katalog sağlayıcısına yönlendirir: `cli`, `vscode`, Atlas ve ChatGPT. Şunları
birleştirir:

1. Varsayılan olarak yönetilen kullanıcı ana dizini stdio'sunu kullanan gözetim App Server'dan gelen Gateway yerel `thread/list` sonuçları.
2. Bağlı ve katılımı onaylanmış her Node'dan gelen `codex.appServer.threads.list.v1` sonuçları.

Döküm seçimi, yerelde `itemsView: "full"` ile `thread/turns/list` veya seçilen
Node'da sürümlendirilmiş `codex.appServer.thread.turns.list.v1` komutunu kullanır.
Her yanıt en fazla 20 kalıcı tur ile opak ileri/geri imleçler içerir. Control UI,
sayfaları en yeniden başlayarak ister, her sayfayı kronolojik sırada işler ve eski sayfaları başa ekler.
Hiçbir zaman sınırsız bir `thread/read` kullanımına geri dönmez. OpenClaw ayrıca
20 MiB üzerindeki serileştirilmiş öğe sayfalarını Node veya Gateway aktarımını geçmeden önce reddeder.

Yerel macOS eşleştirilmiş Node uygulaması, yalnızca ayarlanmamış/varsayılan ya da açık
`appServer.transport: "stdio"` ile ayarlanmamış/varsayılan gözetim kapsamını veya
açık `appServer.homeScope: "user"` değerini destekler. Yapılandırılmış `command`, `args`
ve normalleştirilmiş `clearEnv` değerlerini alt işleme aktarır. `"unix"`, `"websocket"`
veya açık `homeScope: "agent"` ile ne katalog yeteneğini ne de komutu duyurur;
doğrudan çağrı da güvenli biçimde başarısız olur. Aracı kapsamlı bir yapılandırma için kullanıcı
Codex ana dizinini hiçbir zaman açığa çıkarmamalı veya açık bir uç nokta yerine yerel stdio
kullanmamalıdır.

Katalog izdüşümü; tanımlayıcıları, başlığı, cwd'yi, durumu, etkin bekleme
bayraklarını, zaman damgalarını, kaynağı, model sağlayıcısını, Codex sürümünü ve Git dalını normalleştirir.
Döküm önizlemelerini, turları, yürütüm yollarını, Codex ana dizini yollarını,
Git uzak depolarını, commit SHA'larını, ham uç noktaları veya ham App Server hatalarını döndürmez.
Döküm yanıtları yalnızca açıkça istenen App Server öğe sayfasını ve opak imleçlerini
içerir.

Ana makine hataları her ana makine sonucuyla sınırlı kalır. Çevrimdışı bir Node veya kullanılamayan
yerel App Server, sağlıklı ana makineleri sayfadan silmez. Bağlantı, iş parçacığı durumu değil
ana makine özelliğidir: başarısız bir ana makine sonucu yeni oturum satırları içermez
ve yerel iş parçacıklarına `offline` yansıtmaz.

Control UI, aşamalı katalog güncellemeleri ister. Her yerel veya eşleştirilmiş ana makine,
kendi App Server listelemesi tamamlandığında görünür; toplu yanıt, uyumluluk ve kurtarma
anlık görüntüsü olmaya devam eder. Görünür sayfa; bağlantı değişikliklerinden sonra, odaklanıldığında
ve en fazla her 30 saniyede bir, değişikliklerden sonra daha hızlı bir geçişle uzlaştırılır.
Bu nedenle başka bir istemcide oluşturulan yerel Codex oturumları, OpenClaw depolama alanına
aktarılmadan zaman içinde keşfedilir.

Katalog keşfi pasiftir. Meta verileri listelemek veya okumak,
`thread/resume` çağrısı yapmamalı, OpenClaw istemcisini canlı iş parçacığı isteklerine abone etmemeli
veya bir onayı yanıtlamamalıdır.

Arama yalnızca başlıkta yapılır ve büyük/küçük harfe duyarsızdır. Döndürülen her katalog sayfası için
Gateway ve eşleştirilmiş Mac, sorguyu App Server'a iletmeden sınırlı sayıda yerel sayfa tarar;
çünkü yerel arama döküm önizlemeleriyle de eşleşebilir. Döndürülen yerel imleç, çağıranların taramayı
sürdürmesini sağlar.

## Operatör CLI sınırı

Plugin, Gateway destekli üç kabuk komutu kaydeder:

```text
openclaw codex sessions [--search <text>] [--host <id>] [--limit <count>] [--cursor <cursor>] [--json] [gateway-options]
openclaw codex continue <thread-id> [--json] [gateway-options]
openclaw codex archive <thread-id> --confirm-no-other-runner [--json] [gateway-options]
```

`[gateway-options]`; `--url <url>`, `--token <token>`, `--timeout <ms>` ve
devralınan `--expect-final` anahtarıdır. Oturum listeleme varsayılan olarak 75,000 ms;
devam ettirme ve arşivleme ise varsayılan olarak 30,000 ms kullanır;
`--expect-final` bu tekli RPC'ler için ek bir etki oluşturmaz. Oturum araması
yalnızca başlıkta yapılır ve büyük/küçük harfe duyarsızdır; her yanıt sınırlı bir yerel sayfa zincirini
tarar ve `--cursor` daha eski sonuçlarla devam eder. Sınır ana makine başına varsayılan olarak 50'dir
ve 1 ile 100 arasındaki değerleri kabul eder; bir imleç, tek bir kararlı `--host`
hedefi gerektirir. Hiçbir komut
arşivlenmiş/arşivlenmişleri-dahil-et seçeneğini kabul etmez. Yalnızca `sessions` eşleştirilmiş ana makineleri hedefleyebilir;
`continue` ve `archive` her zaman `hostId: "gateway:local"` gönderir ve arşivleme
açık onay bayrağını gerektirir.

Kabuk ad alanı, Chat içi `/codex` çalışma zamanı ad alanı değildir. Özellikle
`/codex sessions --host <node>`, tek bir Node'daki Codex CLI oturum dosyalarını listeler;
`/codex threads`, mevcut konuşma bağlantısının App Server iş parçacıklarını listeler ve
`/codex resume` veya `/codex bind`, bu konuşmanın bağlamasını değiştirir.
Bu komutlar `sessions.catalog.continue` yerine geçmez ve
`/codex continue` ya da `/codex archive` çalışma zamanı komutu yoktur.

## Yerel devam ettirme

Depolanmış veya boşta olan Gateway yerel satırı için kullanıcı arayüzü,
ana makine ve iş parçacığı kimlikleriyle birlikte `catalogId: "codex"` kullanarak
`sessions.catalog.continue` çağrısı yapar. Plugin:

1. Kaynağın zaten bir gözetimli Chat'i varsa mevcut Chat'i yeniden kullanır.
2. Aksi takdirde sınırlı kullanıcı ve asistan geçmişini, kaynağın
   son kalıcı terminal turuna (tamamlandı, kesintiye uğradı veya başarısız oldu) kadar yeni bir
   OpenClaw Chat'e yansıtır ve bekleyen bir donanım dalını kaydeder.
3. Somut bir model veya sağlayıcı seçimi yerine bekleyen yalnızca Codex
   model kilidi politikasını ve özel gözetim bağlantısı kapsamını depolar ve
   OpenClaw `sessionKey` değerini döndürür.

Geçmiş izdüşümü, görünür kullanıcı ve asistan iletilerinin en yeni son bölümünü seçer;
200 ileti, toplam 512 KiB UTF-8 metin ve ileti başına 64 KiB kesin sınırları vardır.
Görüntü ve yerel görüntü girdilerini `[Image attachment]` ile değiştirir,
görüntü yüklerini veya yollarını hiçbir zaman kopyalamaz ve akıl yürütmeyi,
araç çağrılarını ve araç sonuçlarını dahil etmez.

Kullanıcı arayüzü bu oturum anahtarıyla normal Chat'e gider. Henüz standart bir donanım
iş parçacığı yoktur. İlk normal Chat turunda donanım, gerçek
Codex onay, bilgi isteme, olay ve teslim işleyicilerini kurar; ardından:

1. Bir model veya sağlayıcı geçersiz kılması olmadan yerel `thread/fork` çağrısı yapmak ve kalıcı kaynak anlık görüntüsünü sabitlemek için gözetim bağlantısını kullanır. Codex'in geçerli
   `ConfigManager` durumu modeli ve sağlayıcıyı seçer; çatallama yanıtı
   gerçek çifti bildirir. Model, kaynakta kaydedilen son modelden farklıysa
   Codex normal model farkı uyarısını yayınlar.
2. Aynı bağlantıda; `threadSource: "appServer"`, OpenClaw'ın cwd'si, politikası, yapılandırması, ortamı,
   tam OpenClaw donanım araç yüzeyi ve bu ilk başlatma için
   çatallama tarafından döndürülen model ile sağlayıcının tam olarak aynısını kullanarak standart tam Codex donanım iş parçacığını başlatır.
3. Sınırlı görünür kullanıcı ve asistan geçmişini bu bağlantı üzerinden ekler,
   standart bağlamayı gözetim kapsamını kaldırmadan commit eder,
   turu çalıştırır ve geçici çatallamayı arşivler.

İlk turdan önce Chat, görünür bir geçmiş yansıtması bulunan kilitli ve bekleyen bir daldır; sonrasında her model turu, denetim bağlantısındaki standart Codex harness iş parçacığı üzerinden çalışır. Dal, tam bir yerel rollout klonu değildir: kaynak akıl yürütmesi, araç çağrıları ve araç sonuçları kasıtlı olarak dahil edilmez. Anlık görüntünün sabitlenmesi veya standart iş parçacığının oluşturulması başarısız olursa bekleyen dal yeniden denenebilir durumda kalır. Bağlama yarışı, devre dışı bırakılmış denetim ya da kullanılamayan veya eşleşmeyen bir denetim bağlantısı, sıradan agent-home harness'ına geri dönmek yerine tur çalışmadan önce kapalı biçimde başarısız olur.

Bu, kaynağın geçmiş modelinin korunmasını değil, Codex'in sahip olduğu seçimi garanti eder. Fork'un döndürdüğü çift, standart iş parçacığının başlatılması için kullanılır ve Codex bu iş parçacığının yerel modelini ve sağlayıcısını kalıcı hâle getirir. Sonraki sürdürmelerde OpenClaw model ve sağlayıcı geçersiz kılmaları kullanılmaz; böylece Codex kalıcı hâle getirilen çifti geri yükler. Ayrı bir yerel Codex denetimi standart iş parçacığını değiştirirse OpenClaw, yerel olarak kalıcı hâle getirilen bu seçimi kabul eder. Dıştaki OpenClaw modeli ve geri dönüş zinciri hiçbir zaman bunun yerine geçmez.

Denetlenen, modeli kilitli Chat için model değişiklikleri, oturum silme ve oturum sıfırlama/yeni oluşturma işlemleri kapalı biçimde başarısız olur. `/codex model <model>`, `/codex
bind`, `/codex resume` (node `--bind here` dahil) ve `/codex detach` veya `/codex unbind` üzerinde değişiklik yapmak da bağlamayı değiştirdiği veya temizlediği için kapalı biçimde başarısız olur. `/codex model` sorgusu ile `/codex fast`, `/codex permissions` ve `/codex
threads` kullanılabilir durumda kalır. `codex_threads` agent aracı yeni bir fork ekleyemez veya bağlı yerel iş parçacığını arşivleyemez. Listeleme ve yalnızca meta veri okuma kullanılabilir durumda kalır; transkript alanları `supervision.allowRawTranscripts` gerektirirken yeniden adlandırma, arşivden çıkarma, ayrık fork ve ilgisiz bir iş parçacığını arşivleme işlemleri `supervision.allowWriteControls` gerektirir. İki seçenek de kilitli bağlamayı değiştiremez.
OpenClaw girdisinin silinmesi veya sıfırlanması, aksi hâlde yerel bağlamayı atar ve Codex görünümündeki bir oturumun arkasında genel bir iş parçacığı oluşturur ya da buna izin verirdi. Bu nedenle saklama bakımı, sıradan yaş, sayı veya disk bütçesi sınırlarını aşsalar bile modeli kilitli girdileri korur. Sahip Plugin'in devre dışı bırakılması veya kaldırılması da kilidi ve Plugin sahipliği işaretçisini korur. Aynı Plugin yeniden etkinleştirilene kadar Chat kullanılamaz durumda kalır ve kapalı biçimde başarısız olur; temizleme işlemi onu hiçbir zaman sıradan bir model oturumuna dönüştürmez.

Bu işlem kaynağı hiçbir zaman sürdürmez veya değiştirmez. Geçici fork bir anlık görüntüyü sabitler; kalıcı sürdürme iş parçacığı değildir. İlk turda ayrı bir standart harness iş parçacığının başlatılması, süreç yerelindeki durumun Desktop'a ait bir turu görememesi nedeniyle OpenClaw'un rakip bir kaynak yazıcısına dönüşmesini engeller. Görünür geçmiş yansıtması ve sabitlenmiş anlık görüntü, etkin bir kaynakta henüz tamamlanmamış çalışmaları içermeyebilir. Özgün CLI, VS Code, Atlas veya ChatGPT kaynağı hem yerel hem de OpenClaw katalogları için uygun olmaya devam eder. Standart dal, denetim deposunda yerel bir Codex iş parçacığı olarak kalır ancak yerel istemciler bunun `appServer` kaynak türünü filtreleyebilir; dolayısıyla Codex Desktop'ta görünürlük bir sözleşme değildir.

## Arşiv davranışı

Depolanmış veya boşta olan Gateway yerelindeki bir satır için `sessions.catalog.archive` ile
`catalogId: "codex"`, açıkça `confirmNoOtherRunner: true` verilmesini gerektirir, süreç yerelindeki mevcut durumu yeniden okur, yalnızca `idle` veya `notLoaded` için ilerler, yerel `thread/archive` çağrısını yapar ve yalnızca Codex işlemi kabul ettikten sonra başarı döndürür. Ardından satır, arşivlenmemiş katalogdan çıkar.

Yeniden okumadan gelen etkin veya hata durumu, arşivlemeyi reddeder. Kaynaktan gelen başlatılmakta olan veya bekleyen denetimli dal da reddedilir: kaynağın arşivlenebilmesi için ilk Chat turunun standart dalı somutlaştırması gerekir. Tam hedef için bilinen etkin bir OpenClaw bağlama sahibi veya arşivlenmemiş herhangi bir oluşturulmuş alt öğe de arşivlemeyi reddeder. OpenClaw, Codex'in deneysel `thread/list ancestorThreadId` ilişkisini sayfalar ve istek ya da yanıt hatalarında, imleç veya iş parçacığı döngülerinde ve güvenlik sınırının tükenmesinde kapalı biçimde başarısız olur. Yerel arşivleme, yüklenmiş üst öğe ve alt öğe çalışmalarını kapatabilir; bu nedenle arşivleme bir kesme kısayolu değildir. Okuma, alt öğeleri numaralandırma ve arşivleme çağrıları atomik değildir. Bağımsız bir istemci, yerel olarak boşta veya `notLoaded` görünen bir satır üzerinde hâlâ çalışma sahibi olabilir ya da çalışma başlatabilir. Başka çalıştırıcı bulunmadığına ilişkin onay, Codex koşullu arşivleme veya süreçler arası kiralama sağlayana kadar bilinmeyen istemcileri ve bu yarışı kapsar.
Eşleştirilmiş node üzerinde arşivleme yasaktır.

Codex kataloğunda arşivlenmiş görünüm yoktur. Sahibi tarafından yetkilendirilmiş başka bir Codex yüzeyinde `thread/unarchive` ile geri yüklenen bir iş parçacığı, yeniden arşivlenmemiş katalog için uygun hâle gelir.

## Etkin iş parçacığı güvenliği

Codex, tek bir App Server'ın istemcileri arasında bir iş parçacığına yönelik değişiklikleri seri hâle getirir ancak süreçler arası özel bir çalıştırıcı veya onay sahibi kiralaması sunmaz. Bağımsız stdio App Server'ları aynı rollout'a ekleme yapabilirken her biri yalnızca kendi bellek içi durumunu görür. Onay istekleri de tek bir sunucunun tüm abonelerine ulaşabilir ve ilk geçerli yanıt isteği tamamlar.

Bu nedenle:

- pasif katalog istemcileri onaylara abone olmaz veya onları otomatik olarak reddetmez
- şu anda etkin olarak bildirilen satırlar ne yeni bir dalı ne de Arşivle seçeneğini sunar
- eşlenmemiş bir kaynak, standart harness iş parçacığı kaynağı hiçbir zaman sürdürmeyen görünür geçmişli bir dala dönüşür
- `notLoaded`, etkinliği bilinmiyor olarak gösterilir ve yalnızca başka çalıştırıcı bulunmadığı bilinçli biçimde onaylandıktan sonra arşivlenebilir
- yerel arşivleme, bu onaya ek olarak yeniden `idle` veya `notLoaded` okunmasını gerektirirken okuma ile arşivleme arasındaki protokol yarışını kabul eder

Kesme ve çok istemcili devretme, gelecekteki ürün kararlarıdır. Etkin bir satırın gösterilmesi bunları ima etmez.

## Eşleştirilmiş node sınırı

Node çağrısı şu anda yalnızca istek/yanıt biçimindedir. Sınırlandırılmış katalog meta verilerini ve transkript tur sayfalarını güvenle döndürebilir ancak bir Codex harness çalıştırması için gereken uzun ömürlü olay akışını, onay isteklerini, araç çağrılarını, iptali ve asistan deltalarını taşıyamaz.

Bu nedenle node sözleşmesi, liste ve transkript tur sayfalarını destekler. Uzak satırlar okunabilir durumda kalır ancak boşta olma durumundan bağımsız olarak **Devam Et** ve **Arşivle** kullanılamaz. Gerçek bir uzaktan sürdürme, yerel harness ile aynı onay ve bağlama değişmezlerini koruyan node tarafında bir çalıştırıcı ve akış köprüsü gerektirir.

## İzinler

Her bilgisayar yerel olarak katılmayı seçer. Gateway'in etkinleştirilmesi, başka bir node'a onun Codex meta verilerini okuma yetkisi vermez. Node yeteneği, normal eşleştirme ve komut politikası onayından geçmelidir.

Filo listeleme ve transkript görüntüleme, eşleştirilmiş node'ları çağırdıkları için `operator.write` Gateway kapsamını kullanır. Yerel sürdürme ve arşivleme, kimliği doğrulanmış operatör işlemleridir ve ana makine ile durum denetimlerine tabi olmaya devam eder.

Otonom agent ve bağımsız MCP erişimi ayrıdır. Dağıtılan
`codex_endpoint_probe`, `codex_sessions_list`, `codex_session_read`,
`codex_session_send` ve `codex_session_interrupt` araç sözleşmelerinin sahibi
`codex` Plugin'i olmaya devam eder. Denetim etkinleştirildiğinde, ham `codex_threads` transkript okumaları ve transkriptten türetilen liste alanları da
`supervision.allowRawTranscripts` gerektirir; her `codex_threads` fork, yeniden adlandırma, arşivleme veya arşivden çıkarma işlemi `supervision.allowWriteControls` gerektirir. Her iki politika da varsayılan olarak devre dışıdır.

## Uyumluluk

`openclaw doctor --fix`, uç noktalar ve transkript/yazma politikaları dahil dağıtılmış `plugins.entries.codex-supervisor` yapılandırmasını ve Plugin izin/ret başvurularını
`plugins.entries.codex.config.supervision` içine taşır. Açık standart hedef değerleri çakışmalarda önceliklidir. Çalışma zamanı kodu, taşıma sonrasında yalnızca standart `codex` Plugin şeklini kullanır.

Resmî Plugin tam olarak beş Supervisor uyumluluk aracını korur:
`codex_endpoint_probe`, `codex_sessions_list`, `codex_session_read`,
`codex_session_send` ve `codex_session_interrupt`. Oturum listesi varsayılan olarak yalnızca yüklenmiş öğeleri içerir; `loaded_only` parametresi yoktur. `include_stored: true`, uç nokta başına `max_stored_sessions` ile sınırlandırılan (varsayılan 200, kabul edilen aralık 1 ile 1.000) arşivlenmemiş durum veritabanı satırlarını ekler; yüklenmiş satırlar bu ayarla sınırlandırılmaz. Transkriptten türetilen alanlar ve okumalar `allowRawTranscripts` tarafından, gönderme ve kesme ise `allowWriteControls` tarafından denetlenmeye devam eder.

Uyumluluk gönderimi hiçbir zaman boşta olan bir iş parçacığını başlatmaz veya sürdürmez. `mode: "start"` her zaman reddedilir; `"auto"` ve `"steer"` yalnızca okunabilir etkin bir turu yönlendirir. Kesme de benzer şekilde etkin ve okunabilir bir tur gerektirir. Boşta sürdürme, tam harness'ın onaylara, araçlara ve bağlamaya sahip olması için yerel Codex kataloğuna yönlendirilir.
Bağımsız eski MCP bağdaştırıcısı, aynı araçları resmî Plugin'den çözümler ve korunan eski politika ortam değişkenlerini dikkate alan tek yoldur.

Temmuz katalog kullanıcı arayüzü, Gateway yöntemi, node yeteneği ve CLI kaydı eski Plugin kimliğiyle dağıtılmamıştı. Bunlar ikinci bir çalışma zamanı cephesi olmadan doğrudan `codex` sahipliğine taşınır.

## Gelecekteki çalışmalar

- uzaktan sürdürme için node tarafında akış çalıştırıcısı ve olay köprüsü
- eşzamanlı istemci devretme için açık çalıştırıcı ve onay sahibi kiralamaları
- çalıştırıcı sahipliği kiralaması veya eşdeğer bir sınırlama mekanizması sağlandıktan sonra uzaktan arşivleme
- kesme ve daha kapsamlı etkin oturum gözlemi
- Codex Desktop, CLI ve OpenClaw arasında denetlenmiş devretme

Arşivlenmiş öğelere göz atma, planlanan denetim kenar çubuğunun parçası değildir. Yerel Codex yüzeyleri, arşivlenmiş iş parçacıkları için kurtarma yolu olmaya devam eder.

## Kabul testleri

- Denetimin etkinleştirilmesi, arşivlenmemiş yerel oturumları listeler.
- Arşivlenmiş oturumlar katalog yanıtında veya kullanıcı arayüzünde hiçbir zaman görünmez.
- Başka bir ana makine başarısız olduğunda sağlıklı ana makineler görünür kalır; kullanılamayan bir ana makine,
  çevrimdışı oturum durumu uydurmak yerine yeni satır döndürmez.
- Depolanan veya boşta olan bir yerel satır, yalnızca Codex'e özgü bir
  model/çalışma zamanı kilidine sahip Chat yansısı oluşturur; ilk tur geçici bir anlık görüntüyü sabitler ve
  standart tam harness iş parçacığını başlatır; Continue işleminin tekrarlanması mevcut Chat'i açar.
- İlk tur, anlık görüntü çatalında model/sağlayıcı geçersiz kılmalarını kullanmaz ve
  Codex mevcut modelinin kaynağın son kaydedilen modelinden farklı olduğu konusunda uyarıda bulunsa bile
  standart başlangıcı Codex tarafından döndürülen tam çifte sabitler.
- Bekleyen ve kaydedilmiş denetimli bağlamalar; kaynak erişimi,
  standart dal oluşturma ve sonraki tüm turlar için denetim bağlantısını kullanır; sıradan
  Codex oturumları aracı kapsamında kalır.
- Sonraki sürdürme işlemleri OpenClaw model/sağlayıcı geçersiz kılmalarını kullanmaz, Codex'in
  standart kalıcı seçimini korur, bu iş parçacığına ayrı yerel değişiklikleri kabul eder
  ve hiçbir zaman dış OpenClaw modelini veya geri dönüş zincirini ikame etmez.
- Denetimin devre dışı bırakılması veya bağlama/bağlantı yaşam döngüsünün kaybedilmesi, Chat'i
  sıradan aracı ana dizini harness'ına taşımak yerine güvenli biçimde başarısız olur.
- Denetimli ve model kilitli bir Chat, yerel bağlamayı koruduğu sürece silinemez.
- Chat en fazla 200 kullanıcı ve asistan mesajını, toplam 512 KiB'ı ve
  mesaj başına 64 KiB'ı yansıtır. Görseller yer tutuculara dönüşür; kaynak akıl yürütme, araç çağrıları,
  araç sonuçları, görsel yükleri ve yerel yollar klonlanmaz.
- Dal akışı kaynak iş parçacığını hiçbir zaman sürdürmez.
- Özgün kaynak her iki katalog için de uygun kalır. Standart yerel
  dal, `appServer` kaynak türünü kullanır ve Codex Desktop'ta görünmesi
  garanti edilmez.
- Etkin yerel kaynaklar dal oluşturamaz veya arşivlenemez; mevcut
  denetimli Chat yine de açılabilir.
- Etkinliği bilinmeyen satırlar onay olmadan dallanabilir; arşivleme için
  başka çalıştırıcı bulunmadığının açıkça onaylanması gerekir.
- Başlatılmakta veya beklemede olan denetimli dala sahip bir kaynak,
  ilk Chat turu standart dalı somutlaştırana kadar arşivlenemez.
- Tam hedefin veya arşivlenmemiş, oluşturulmuş herhangi bir alt öğenin bilinen etkin bağlama
  sahibi arşivlemeyi engeller; alt öğe numaralandırma hataları güvenli biçimde başarısız olur ve
  açık onay, bilinmeyen istemcilerden ve durum ile arşivleme arasındaki
  yarış koşulundan sorumlu olmaya devam eder.
- Onaylanmış, depolanan veya boşta olan yerel arşivleme, yerel işlem başarıyla tamamlandıktan sonra satırı kaldırır.
- Eşleştirilmiş Node satırları Continue veya Archive olmadan görünür kalır.
- Pasif listeleme, iş parçacığı onaylarına hiçbir zaman abone olmaz veya yanıt vermez.
- Eski Supervisor yapılandırması, standart Codex yapılandırma biçimine geçirilir.
- Eski liste varsayılan olarak yalnızca yüklenir, depolanan numaralandırma uç nokta başına
  sınırına uyar ve uyumluluk gönderimi boşta olan bir iş parçacığını hiçbir zaman başlatmaz veya sürdürmez.
