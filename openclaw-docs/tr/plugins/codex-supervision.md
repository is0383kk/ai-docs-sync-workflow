---
read_when:
    - Codex Desktop veya CLI oturumlarının OpenClaw'da görünmesini istiyorsunuz
    - Depolanmış veya boşta olan yerel bir Codex oturumundan dal oluşturmanız ya da oturumu arşivlemeniz gerekiyor
    - Eşleştirilmiş node'lardan Codex oturumlarını ve transkript geçmişini kullanıma açıyorsunuz
sidebarTitle: Codex supervision
summary: OpenClaw Node'ları genelinde arşivlenmemiş yerel Codex oturumlarına ve sayfalandırılmış transkriptlere göz atın
title: Codex oturumlarını denetleyin
x-i18n:
    generated_at: "2026-07-26T23:26:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f365e3207dff092c3dfd8f7588d60d70a16f0cce484991eb4ab3fc0bd15f8051
    source_path: plugins/codex-supervision.md
    workflow: 16
---

Codex denetimi, resmî `codex` plugininin isteğe bağlı bir özelliğidir. Gateway bilgisayarındaki ve katılımı etkinleştirilmiş eşleştirilmiş bilgisayarlardaki arşivlenmemiş Codex CLI, VS Code, Atlas ve ChatGPT kaynak oturumlarını normal oturumlar kenar çubuğunda ve Sohbet bölmesinde gösterir.

İlk sürüm, sorumluluk kapsamını bilinçli olarak dar tutar:

- Depolanmış veya boşta olan yerel bir oturum, sınırlandırılmış kalıcı kullanıcı ve asistan geçmişinden modele kilitli bir OpenClaw Sohbeti oluşturabilir. İlk mesaj yerel bir anlık görüntü dalı oluşturur, ardından Codex App Server'ın bu dal için seçtiği model ve sağlayıcıyla tam Codex çalışma sistemi iş parçacığını başlatır. Sonraki dönüşler, denetlenen bağlama OpenClaw'ın başka bir çalışma zamanı, model veya geri dönüş seçeneği kullanmasını engellerken standart yerel iş parçacığının kalıcı çiftini geri yükler. Ayrı bir yerel Codex denetimi bu kalıcı çifti yine de değiştirebilir. Önceden oluşturulmuş bir dal, mevcut Sohbetini açar.
- Başka bir Codex işleminden keşfedilen depolanmış bir oturumun canlı etkinliği bilinmez. Oturumdan dal oluşturulabilir veya yalnızca operatör başka hiçbir Codex istemcisinin onu kullanmadığını onayladıktan sonra arşivlenebilir.
- Etkin bir kaynak görünür kalır ancak geçerli dönüşü tamamlanana kadar kaynakta dal oluşturulamaz veya kaynak arşivlenemez. Zaten denetlenen bir Sohbeti varsa **Sohbeti Aç** seçeneği kullanılabilir durumda kalır.
- Eşleştirilmiş bir Node üzerindeki oturum, kalıcı transkriptini sınırlandırılmış ve imleçle sayfalandırılan App Server okumaları aracılığıyla sunar. Uzaktan devam ettirme, gelecekte sunulacak akış özellikli bir Node köprüsü gerektirir; uzaktan arşivleme ise ayrıca bir çalıştırıcı sahipliği kirası veya eşdeğer bir çitleme mekanizması gerektirir.
- Arşivlenmiş oturumlar listelenmez. Depolanmış veya boşta olan yerel bir oturum yalnızca operatör başka hiçbir Codex istemcisinin onu kullanmadığını onayladıktan sonra arşivlenebilir.

## Başlamadan önce

- Resmî `@openclaw/codex` pluginini Gateway'e yükleyin. OpenClaw macOS uygulaması, Codex özelliklerini etkinleştirdiğinizde bunu yükleyebilir; CLI kurulumları `openclaw plugins install @openclaw/codex` komutunu çalıştırabilir.
- Oturumlarını listelemek istediğiniz her bilgisayara Codex Desktop veya Codex CLI'ı yükleyip oturum açın.
- Uzak bilgisayarları OpenClaw Node'ları olarak eşleştirin. Her bilgisayar katılımı yerel olarak etkinleştirmelidir; denetimin yalnızca Gateway'de etkinleştirilmesi başka bir Node'a yetki vermez.
- Sahibinin denetimindeki bir Gateway kullanın. Oturum başlıkları, çalışma dizinleri ve Git dalları hassas proje bilgilerini açığa çıkarabilir.

## Denetimi etkinleştirme

Yönlendirmeli `openclaw onboard` ve macOS ilk çalıştırma kurulumu, yerel bir Codex kurulumu algılandıktan ve seçilen çıkarım arka ucu başarıyla etkinleştirildikten sonra Codex denetimini yükleyip etkinleştirmeyi dener. Codex'in birincil arka uç olması gerekmez. Bu fırsatçı plugin etkinleştirmesi başarılı olduğunda denetim kullanılabilir hâle gelir. App Server kullanılabilirliği, denetim ilk kez bağlandığında kontrol edilir. Codex plugininin açıkça devre dışı bırakılması veya bir politika engeli, fırsatçı etkinleştirmeyi önler; mevcut açık bir `supervision.enabled: false`, ajanlara yönelik denetim araçlarını devre dışı bırakır. `sessionCatalog.enabled: false` devre dışı bırakmadığı sürece operatör kataloğu, Codex plugini etkin olduğunda kayıtlı kalır. Bu ayrı anahtar, Codex sağlayıcısını, çalışma sistemini ve ajanlara yönelik denetim politikasını değiştirmeden bırakırken eşleştirilmiş Node katalog listeleme/okuma komutlarını da bu ana makineden kaldırır. Mevcut kurulumlar aynı özelliği elle etkinleştirebilir:

`codex` pluginini ve denetim özelliğini `openclaw.json` içinde etkinleştirin:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          supervision: {
            enabled: true,
          },
        },
      },
    },
  },
}
```

`plugins.allow` mevcutsa `codex` değerini ekleyin. Plugin etkinleştirmesini değiştirdikten sonra Gateway'i yeniden başlatın.

Açık bir `appServer` bağlantı ayarı olmadığında denetim, yerel kullanıcı Codex ana dizinine karşı ayrı bir yönetilen stdio denetim bağlantısı kullanır. Normal Codex çalışma sistemi varsayılan olarak ajan kapsamlı kalır. Böylece normal OpenClaw dönüşlerinin yerel Codex durumunu paylaşmasına neden olmadan yerel oturumlar her iki uygulamada da görünür olur. Çalışma sisteminin de bu durumu paylaşması gerekiyorsa `appServer.homeScope: "user"` değerini açıkça ayarlayın. Denetim, açık `appServer` bağlantı ayarlarını kendi yerel kullanıcı ana dizini varsayılanıyla değiştirmek yerine bunlara uyar.

**Codex** kenar çubuğu grubundan benimsenen bir Sohbet, normal bir çalışma sistemi oturumu değildir. Özel denetim bağlaması; kaynak okumaları, standart dal oluşturma, geçmiş ekleme ve sonraki her dönüş için denetim bağlantısını kullanır. Varsayılan yerel bağlantıyla bu, diğer oturumların varsayılanını değiştirmeden yerel kullanıcı Codex ana dizinini, kimlik doğrulamasını ve sağlayıcı yapılandırmasını korur. İzlenen benimsenmiş Sohbetler de [oturum durumu farkındalığına](/tr/concepts/session-state) katılır.

Varsayılan yerel denetim bağlantısında depo, yerel Codex istemcileriyle paylaşılır. OpenClaw, başka bir istemcinin aynı canlı App Server işlemini paylaştığını varsaymaz ve yerel durum sahipliği işlem kapsamındadır. Bu nedenle, kendi denetim App Server'ının `notLoaded` olarak bildirdiği bir iş parçacığını boşta olarak değil, **Depolanmış / etkinlik bilinmiyor** olarak değerlendirir.

Oturumlarının görünmesi gereken her başsız Node ana makinesinde aynı katılımı etkinleştirin. Yerel OpenClaw macOS uygulaması, Codex kataloğunu eşleştirilmiş Gateway'e duyururken aynı yerel ayarı okur. Bu eşleştirilmiş yerel Mac kataloğu yalnızca varsayılan veya açık `appServer.transport: "stdio"` ile ayarlanmamış ya da açık `appServer.homeScope: "user"` değerini destekler. Bu stdio işlemi için `command`, `args` ve `clearEnv` dikkate alınır. Mac yapılandırması `"unix"`, `"websocket"` veya `homeScope: "agent"` seçerse uygulama katalog özelliğini veya komutunu duyurmaz; eski bir doğrudan çağrı, kullanıcı Codex ana dizinini açığa çıkarmak veya farklı bir yerel stdio App Server başlatmak yerine başarısız olur.

Yeni duyurulan bir Node komutu, Node'un onaylı komut yüzeyini değiştirir. Güncellemeyi Gateway ana makinesinden onaylayın:

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

Arşivlenmemiş Codex oturumları, ana Denetim Arayüzü kenar çubuğunda da ana makineye göre gruplandırılmış olarak görünür. Kalıcı transkriptini okumak için birini seçin. Görüntüleyici, en yeni Codex `thread/turns/list` API'sini `itemsView: "full"` ile kullanır ve istek başına en fazla 20 dönüş yükler; **Daha eski transkript öğelerini yükle**, en yeni sayfadaki opak App Server imlecini izler. Yüklenen sayfalar kronolojik sırada işlenir. Görüntüleyici hiçbir zaman sınırsız bir `thread/read` geçmişi yüklemez. 20 MiB aktarım güvenliği sınırını aşan bir sayfa, Node veya Gateway bağlantısını riske atmak yerine kapalı biçimde başarısız olur.

Normal oturumlar kenar çubuğunda **Codex** grubunu açın. Aynı oturumları ana makineye göre gruplandırılmış olarak listeler. **Daha fazla oturum yükle**, daha eski satırları bulunan her ana makinenin sonraki sayfasını ekler ve eklenen bu satırlar kenar çubuğunun düzenli yenilemeleri sırasında korunur. Her ana makine, kendi yerel listelemesi tamamlanır tamamlanmaz görünür. Görünür sayfa; Node bağlantısı değişikliklerinden sonra, yeniden odağı aldığında ve en fazla 30 saniyede bir uzlaştırılır; değişen bir sonuç daha hızlı bir takip geçişi başlatır. Böylece Codex Desktop, CLI veya başka bir yerel istemcide oluşturulan oturumlar sayfanın tamamen yeniden yüklenmesine gerek kalmadan görünür. İlk sayfa Codex'in en son güncellenene göre kendi sıralamasını izlediğinden yeni oluşturulan yerel bir oturum hemen uygun hâle gelir.
Her döndürülen arama sayfası, sorguyu App Server'a göndermek yerine ana makine başına sınırlı sayıda yerel sayfayı tarar; çünkü yerel arama transkript önizlemeleriyle de eşleşebilir.

Ana makine kullanılabilirliği ile iş parçacığı durumu birbirinden ayrıdır. **Çevrimdışı** veya **Kullanılamıyor**, bir ana makine yenilemesini tanımlar; kullanılamayan bir ana makine yeni oturum satırları döndürmez ve bir iş parçacığının yerel durumunu `offline` olarak değiştirmez. Oturum satırları `idle`, `active`, `notLoaded` veya hata gibi Codex durumlarını kullanır. Başarısız olan bir ana makine, sağlıklı ana makinelerin sonuçlarını gizlemez.

Kenar çubuğu uyarısı, katalog hata kodunu ve altta yatan güvenli Gateway hatasını içerir. Codex'i devre dışı bırakmadan keşfi devre dışı bırakmak için **Ayarlar > Otomasyon > Pluginler > Codex > Yerel Oturum Keşfi** bölümünü açın. `NODE_LIST_FAILED` için `openclaw nodes list` ile **Ayarlar > Cihazlar** bölümünü karşılaştırın; ayrıntılı neden, onarılması gereken eşleştirme deposu, Node kayıt defteri, izin veya Gateway yaşam döngüsü hatasını belirtir.

## Operatör CLI'ını kullanma

Terminal CLI, aynı arşivlenmemiş kataloğu ve Gateway'e yerel dal oluşturma ve arşivleme eylemlerini sunar:

```bash
openclaw codex sessions [--search <text>] [--host <id>] [--limit <count>] [--cursor <cursor>] [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
openclaw codex continue <thread-id> [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
openclaw codex archive <thread-id> --confirm-no-other-runner [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
```

`openclaw codex sessions` seçenekleri:

- `--search <text>`, oturum başlıklarında büyük/küçük harfe duyarlı olmadan arama yapar.
- `--host <id>`, yanıtı `gateway:local` veya `node:<node-id>` gibi tek bir kararlı katalog ana makinesiyle sınırlar.
- `--limit <count>`, ana makine başına 1 ile 100 arasında satır ayarlar; varsayılan değer 50'dir.
- `--cursor <cursor>`, tek bir ana makine sayfasını devam ettirir ve bu nedenle `--host` gerektirir.
- `--json`, yapılandırılmış Gateway yanıtını yazdırır.

Üç komut da Gateway istemcisinden `--url`, `--token` ve `--timeout <ms>` değerlerini devralır. Soğuk eşleştirilmiş Node kataloglarının tamamlanabilmesi için oturum listeleme varsayılan olarak 75,000 ms kullanır; devam ettirme ve arşivleme için varsayılan değer 30,000 ms'dir. Ayrıca bu tekli denetim RPC'lerini değiştirmeyen ortak `--expect-final` anahtarını sunarlar. Her komut `operator.write` Gateway kapsamını gerektirir.
Standart `-h, --help` çıktısı her alt komutta kullanılabilir.
Arşivlenmiş veya arşivlenmişleri dahil et seçeneği yoktur. `sessions` eşleştirilmiş ana makineleri listeleyebilir ancak `continue` ve `archive` her zaman `gateway:local` hedefini kullanır; eşleştirilmiş satırlar yalnızca listelenebilir. Arşivleme her zaman `--confirm-no-other-runner` gerektirir.

Bu kabuk komutları, sohbet içindeki `/codex` çalışma zamanı komutlarından farklıdır.
`/codex threads [filter]`, geçerli konuşma bağlantısının erişebildiği App Server iş parçacıklarını listeler. `/codex sessions --host <node>`, denetim filosu kataloğunu değil, tek bir Node üzerindeki sürdürülebilir Codex CLI oturum dosyalarını listeler. `/codex
resume` ve `/codex bind`, güvenli ve denetlenen bir dal oluşturmak yerine geçerli konuşmayı bağlar; modele kilitli denetlenen bir Sohbet bu bağlama değişikliklerini reddeder. `/codex continue` veya `/codex archive` çalışma zamanı komutu yoktur.

## Yerel bir oturumdan dal oluşturma

Gateway bilgisayarındaki depolanmış veya boşta olan bir satırda **Dal olarak devam et** seçeneğini belirleyin. OpenClaw normal bir Sohbet girdisi oluşturur, sınırlı kullanıcı ve asistan geçmişini kaynağın son kalıcı terminal dönüşüne (tamamlanmış, kesintiye uğramış veya başarısız) kadar yansıtır, bekleyen bir çalışma sistemi dalını kaydeder ve Sohbeti açar. Genel model seçici kilitlenir ancak henüz belirli bir model veya sağlayıcı seçilmemiştir. Kaynak sürdürülmez ve standart çalışma sistemi iş parçacığı henüz başlatılmaz. Eylemin yinelenmesi başka bir dal oluşturmak yerine mevcut Sohbeti açar.

Yansıtma, üç sınırın tamamına uyan en yeni görünür kuyruğu korur: en fazla 200 kullanıcı veya asistan mesajı, toplam 512 KiB UTF-8 metni ve mesaj başına 64 KiB. Büyük boyutlu mesajlar bir işaretleyiciyle kısaltılır ve sınıra ulaşıldığında daha eski mesajlar atlanır. Bir görüntü veya yerel görüntü girdisi, değişmez `[Image attachment]` yer tutucusuna dönüşür; görüntü verileri ve yerel yollar kopyalanmaz.

Çalışmaya başlamak için ilk normal Chat mesajını gönderin. Codex donanımı gerçek
onay, bilgi isteme, olay ve teslim işleyicilerini kurar. Model veya sağlayıcı
geçersiz kılması sağlamadan kaynak anlık görüntüsünü sabitlemek için denetim
bağlantısında geçici bir yerel çatallanma kullanır. Codex App Server, her ikisini
de mevcut yerel yapılandırmasından seçer ve gerçek seçimi döndürür. Aynı
bağlantıda OpenClaw, tam donanımlı standart `appServer` kaynak iş parçacığını,
tam olarak döndürülen bu çiftle, kendi cwd'si ve çalışma zamanı politikası
altında başlatır; sınırlandırılmış görünür geçmişi ekler ve geçici çatallanmayı
arşivler. Standart iş parçacığı, OpenClaw donanımının tüm araç yüzeyine sahiptir.
Bu, tam bir yerel yürütme klonu değil, görünür geçmiş dalıdır: kaynak muhakemesi,
araç çağrıları ve araç sonuçları dahil edilmez. Bu tur ve sonraki tüm turlar,
başka bir OpenClaw model çalışma zamanı veya olağan agent-home donanımı yerine
denetlenen Codex bağlantısında kalır.

Döndürülen seçim, kaynağın geçmişteki modelinin kanıtı değildir. Mevcut yerel
yapılandırma, kaynağın son turu için kaydedilen modelden farklıysa Codex normal
model farkı uyarısını yayınlar. OpenClaw, standart iş parçacığını başlatmak için
döndürülen çifti kullanır. Codex, bu standart iş parçacığının yerel modelini ve
sağlayıcısını kalıcılaştırır; OpenClaw model ve sağlayıcı geçersiz kılmalarını
atladığı için sonraki sürdürmeler bunları korur. Standart iş parçacığı ayrı bir
yerel Codex denetimi üzerinden değiştirilirse OpenClaw, Codex'in kalıcılaştırdığı
seçimi kabul eder. OpenClaw hiçbir zaman kendi dış modelini veya geri dönüş
zincirini ikame etmez.

Denetlenen, modele kilitli Chat silinemez, model değiştiremez, `/new`
veya `/reset` kullanamaz, Gateway oturum sıfırlama eylemini çağıramaz
ya da genel **Oturumu çatalla** eylemini kullanamaz. `/codex model <model>`,
`/codex
bind`, `/codex resume` (`--bind here` içeren bir node oturumu
dahil) ile `/codex detach` veya `/codex unbind` üzerinde değişiklik yapmak
da kilitli yerel bağlamayı değiştireceği veya temizleyeceği için reddedilir.
`/codex model` sorgusu ile `/codex fast`, `/codex permissions` ve
`/codex threads` kullanılabilir durumda kalır. Farklı bir model veya yeni bir
iş parçacığı istediğinizde başka bir olağan oturum başlatın.

Bu Chat için denetimi etkin tutun. Denetim devre dışı bırakılırsa veya saklanan
bağlantı bağlaması kullanılamaz ya da tutarsız hâle gelirse tur, olağan bir
agent-home oturumuna geçmek yerine güvenli biçimde başarısız olur.

`codex` Plugin'ini devre dışı bırakmak veya kaldırmak bu sahipliği
serbest bırakmaz ya da Chat'i başka bir model için uygun hâle getirmez. Kilitli
Chat korunur ancak kullanılamaz durumda kalır; sürdürmek için aynı Plugin'i
yeniden kurun veya yeniden etkinleştirin ve Gateway'i yeniden başlatın. Bu kasıtlı
güvenli başarısızlık davranışı, saklama temizliğinin veya geçici bir Plugin
kesintisinin yerel bağlamayı sessizce sahipsiz bırakmasını önler.

`codex_threads` aracı da aynı sınıra uyar. Farklı bir çatallanmayı bağlayamaz
veya Chat'e bağlı yerel iş parçacığını arşivleyemez. Listeleme ve yalnızca meta
veri okuma kullanılabilir durumda kalır. Ham transkript okumaları
`allowRawTranscripts` gerektirir. Ham erişim devre dışıyken `codex_threads`,
yerel arama transkript önizlemeleri içerdiği için liste aramasını da reddeder;
Control UI ve operatör CLI'si sınırlandırılmış, yalnızca başlık içeren arama
sağlamaya devam eder. Yeniden adlandırma, arşivden çıkarma, bağlantısız çatallama
ve sahip olunmayan ilgisiz bir iş parçacığını arşivleme
`allowWriteControls` gerektirir. İki seçenek de kilitli bağlamayı atlamaz.

OpenClaw yalnızca kaynak iş parçacığını listelerken veya bekleyen Chat'i
görüntülerken onay isteklerine abone olmaz ya da yanıt vermez. İlk turda ayrı
bir standart donanım iş parçacığı başlatmak, rakip yürütme yazıcıları
oluşturmadan başka bir Codex sürecinin kaynağa sahip olmaya devam etmesini
sağlar.

Özgün CLI, VS Code, Atlas veya ChatGPT kaynağı yerel istemciler ve OpenClaw
kataloğu tarafından görünür kalır. Standart dal, yerel bir Codex iş parçacığı
olarak saklanır ancak kaynak türü `appServer` şeklindedir; Codex Desktop
veya başka bir yerel istemci bu kaynak türünü filtreleyebilir, dolayısıyla dalın
kendisinin her yerel geçmiş görünümünde görünmesi garanti edilmez.

OpenClaw'ın App Server'ı tarafından etkin olarak bildirilen bir satır yeni bir
dal başlatamaz. Mevcut turun bitmesini bekleyin ve kataloğu yenileyin. Codex App
Server, değişiklikleri tek bir süreç içinde seri hâle getirir ancak süreçler
arası özel bir çalıştırıcı veya onay sahibi kiralaması sağlamaz.

**Saklandı / etkinlik bilinmiyor** satırı için Chat yansısı ve ilk tur anlık
görüntü sabitlemesi, Codex'in son kalıcılaştırılmış terminal turuna kadarki
durumunu kullanır. Kaynak iş parçacığı sürdürülmez, kesintiye uğratılmaz veya
arşivlenmez. Başka bir süreçte devam eden bir tur varsa bu turun en son işlemdeki
çalışması dalda bulunmayabilir.

## Yerel bir oturumu arşivleme

Saklanan veya boşta olan Gateway-yerel bir satırda **Arşivle** seçeneğini
belirleyin, ardından başka hiçbir Codex istemcisinin veya OpenClaw çalıştırıcısının
bu iş parçacığını ya da onun oluşturduğu alt öğeleri kullanmadığını doğrulayın.
OpenClaw süreç-yerel durumu yeniden okur, yalnızca `idle` veya
`notLoaded` için devam eder, yerel Codex arşivleme işlemini çağırır ve
oturumu arşivlenmemiş listesinden kaldırır. Yerel Codex ayrıca iş parçacığının
oluşturduğu alt öğeleri arşivlemeyi dener.

Yeniden okuma oturumu etkin veya hata durumunda bildiriyorsa, oturum eşleştirilmiş
bir node'a aitse ya da yeni oluşturulan denetimli Chat'in bu kaynaktan hâlâ
bekleyen bir dalı varsa arşivleme kullanılamaz. Kaynağı arşivlemeden önce standart
dalını somutlaştırmak için Chat'in ilk mesajını gönderin. OpenClaw etkin bir
bağlamanın tam hedef iş parçacığına veya arşivlenmemiş herhangi bir oluşturulmuş
alt öğeye sahip olduğunu biliyorsa arşivleme de engellenir. OpenClaw, deneysel
Codex alt öğe sorgusunu her sayfada izler; geçersiz yanıt, istek hatası,
yinelenen imleç veya iş parçacığı ya da güvenlik sınırının tükenmesi arşivlemeyi
reddettirir.

Okuma, alt öğeleri numaralandırma ve arşivleme istekleri tek bir koşullu işlem
değildir; dolayısıyla bunların arasında yine de bir tur başlayabilir. App Server
durumu da bağımsız süreçler arasında paylaşılmaz. Bu nedenle onay, bilinmeyen
istemciler ve bu yarış durumu için güvenlik sınırıdır: onaylamadan önce diğer
tüm istemcilerden çıkın veya bunları başka bir şekilde doğrulayın. Arşivlenmiş
bir iş parçacığını Codex Desktop, Codex CLI veya sahip tarafından yetkilendirilmiş
yerel bir iş parçacığı yönetim akışıyla geri yükleyin; arşivden çıkarıldıktan
sonra yeniden görünür.

```bash
codex unarchive <thread-id>
```

## Eşleştirilmiş node sınırlarını anlama

Eşleştirilmiş node'lar sürümlendirilmiş, salt okunur
`codex.appServer.threads.list.v1` ve
`codex.appServer.thread.turns.list.v1` komutlarını sunar. Codex CLI'nin
kullanılabildiği yerel node ana makineleri ayrıca izin verilenler listesindeki
`codex.terminal.resume.v1` komutunu sunar. Gateway, ham App Server
uç noktalarını hiçbir zaman değil, normalleştirilmiş meta verileri ve açıkça
istenen sınırlandırılmış transkript sayfalarını alır. Operatör terminalinde bir
satırı açmak, sahip olan ana makinede `codex resume <thread-id>` komutunu çalıştırır ve
bu komutun PTY'sini aktarır; genel bir kabuk veya Gateway tarafından sağlanan
argv sunmaz.

Terminal aktarımı, donanım sürdürme veya arşiv sahipliği sözleşmelerini sağlamaz.
Bu nedenle uzak satırlar görünür kalır ancak uzak iş parçacığı boşta olsa bile
**Devam Et** veya **Arşivle** seçeneklerini sunmaz. Bu bilgisayarda Codex'i
**Terminalde aç** üzerinden kullanın veya güvenli bir çalıştırıcı sahipliği
sınırına sahip gelecekteki bir sürdürme akışını kullanın.

## Meta veriler ve izinler

Katalog satırları şunları içerebilir:

- iş parçacığı ve oturum tanımlayıcıları
- başlık ve çalışma dizini
- mevcut durum ve etkin bekleme bayrakları
- oluşturulma, güncellenme ve etkinlik zaman damgaları
- kaynak, model sağlayıcısı, Codex CLI sürümü ve Git dalı

Katalog projeksiyonu; transkript önizlemelerini, turları, yürütme yollarını,
Codex ana dizin yolunu, Git uzak depolarını, commit SHA'larını ve ham App Server
hatalarını hariç tutar. Katalog erişimi ve Control UI transkript okumaları
`operator.write` Gateway kapsamını gerektirir çünkü her iki node komutu da salt
okunur olmasına rağmen filo toplama standart `node.invoke` yolunu kullanır.

`supervision.allowRawTranscripts` ve `supervision.allowWriteControls`, otonom agent ve bağımsız MCP araçlarını
yönetir. Her ikisi de varsayılan olarak `false` değerindedir. Denetim
etkinken `codex_threads`, ham transkriptlere izin verilmediği sürece liste ve
yalnızca meta veri okuma sonuçlarından transkript önizlemelerini ve turları
kaldırır; tur içeren bir okuma güvenli biçimde başarısız olur. Her çatallama,
yeniden adlandırma, arşivleme ve arşivden çıkarma yazma denetimleri gerektirir.
Bu seçenekler, kimliği doğrulanmış Control UI transkript görüntülemeyi kısıtlamaz
ve bağlama, ana makine, durum veya onay kontrollerini atlamaz.

### Uyumluluk araçları

Resmî `codex` Plugin'i, mevcut agent ve bağımsız MCP istemcileri için
yayınlanmış beş Supervisor araç adını korur:

- `codex_endpoint_probe`
- `codex_sessions_list`
- `codex_session_read`
- `codex_session_send`
- `codex_session_interrupt`

`codex_sessions_list` varsayılan olarak yalnızca yüklenenleri kapsar;
`loaded_only` parametresi yoktur. Codex'in durum veritabanından arşivlenmemiş
saklanan satırları da okumak için `include_stored: true` ayarını yapın. İsteğe bağlı
`max_stored_sessions` üst sınırı varsayılan olarak 200'dür ve uç nokta başına 1 ile
1.000 arasında satır kabul eder. Yüklenen satırları sınırlamaz. Ham transkript
izni olmadan liste sonuçları, transkriptlerden türetilen adları, önizlemeleri ve
ayrıntılı uç nokta hatalarını içermez.
`codex_session_read`, `allowRawTranscripts` gerektirir; `include_turns: true` ayrıca
Codex'ten turları ister.

`codex_session_send` ve `codex_session_interrupt`,
`allowWriteControls` gerektirir. Gönderme işlemi `mode: "auto" | "start" | "steer"` kabul eder ancak
`"start"` her zaman reddedilir ve hem `"auto"` hem de
`"steer"` yalnızca okunabilir etkin bir turu yönlendirebilir. Boştaki
bir iş parçacığı, tam donanımın sürdürmeden önce onay ve araç işleyicilerini
kurduğu **Codex Oturumları** seçeneğini kullanma yönlendirmesiyle reddedilir.
Kesme işlemi de aynı şekilde etkin ve okunabilir bir tur gerektirir. Bu araçlar,
boştaki bir kaynak iş parçacığını sürdürmez veya başlatmaz.

`openclaw doctor --fix`, kullanımdan kaldırılmış bir `codex-supervisor` girdisini,
onun uç nokta ve izin alanlarını ve Plugin izin/verme-reddetme politikası
başvurularını, açık standart ayarların üzerine yazmadan resmî
`codex` Plugin'ine taşır. Bağımsız uyumluluk MCP bağdaştırıcısı aynı
beş aracı bu Plugin'den yüklemeye devam eder; eski politika ortam değişkenleri
yalnızca bu güvenilir bağdaştırıcının içinde uygulanır.

Her denetim yapılandırma alanı için
[Codex donanım başvurusuna](/tr/plugins/codex-harness-reference#supervision) bakın.

## Sorun giderme

**Hiçbir oturum görünmüyor:** `@openclaw/codex` öğesinin kurulu olduğunu, hem
Plugin'in hem de `supervision.enabled` değerinin true olduğunu, mevcut Plugin izin
verilenler listesinin `codex` öğesine izin verdiğini ve oturumların
arşivlenmemiş olduğunu doğrulayın. Etkinleştirmeyi değiştirdikten sonra Gateway'i
veya node'u yeniden başlatın.

**Devam Et devre dışı:** eşlenmemiş bir satır etkin, eşleştirilmiş bir node'a ait,
ana makinesi çevrimdışı veya başka bir eylem bekliyor. Gateway-yerel saklanan ve
boştaki satırlar, güvenli olmayan tam iş parçacığı devralma yerine **Dal olarak
devam et** seçeneğini sunar. Zaten denetimli bir Chat'e sahip satır **Chat'i aç**
seçeneğini sunar.

**Arşivle devre dışı:** arşivleme, başka çalıştırıcı olmadığının onaylanmasının
ardından saklanan/etkinliği bilinmeyen ve boştaki Gateway-yerel satırlar için
kullanılabilir. Etkin, hatalı, çevrimdışı, eşleştirilmiş node'a ait, bekleyen
dallı ve bilinen tam bağlama sahibine ait satırlar arşivleme açısından salt
okunur kalır.

**Arşivlenmiş bir oturum kayboldu:** bu beklenen bir durumdur. Denetim sayfasında
arşivlenmiş görünümü yoktur. Yeniden göstermek için `codex unarchive <thread-id>` komutunu
çalıştırın veya Codex Desktop'ı kullanın.

**Eski `codex-supervisor` yapılandırması duruyor:** `openclaw doctor --fix` komutunu
çalıştırın. Doctor, kullanımdan kaldırılmış Plugin girdisini ve ilgili Plugin
politikası başvurularını açık Codex ayarlarının üzerine yazmadan
`plugins.entries.codex.config.supervision` içine taşır.

## İlgili

- [Codex donanımı](/tr/plugins/codex-harness)
- [Codex donanım başvurusu](/tr/plugins/codex-harness-reference)
- [Codex donanım çalışma zamanı](/tr/plugins/codex-harness-runtime)
- [Codex denetim mimarisi](/specs/codex-supervision)
- [Node'lar](/tr/nodes)
- [Gateway güvenliği](/tr/gateway/security)
