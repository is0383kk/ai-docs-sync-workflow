---
read_when:
    - Gateway aracısının eşleştirilmiş bir masaüstünü görmesine ve kontrol etmesine izin verme
    - Bilgisayar kullanımı için etkinleştirme, izinler veya güvenlik
    - computer.act Node komutunu veya karşılayıcılarını genişletme
summary: Bilgisayar aracı ve computer.act Node komutu aracılığıyla yetenek tabanlı masaüstü denetimi
title: Bilgisayar kullanımı
x-i18n:
    generated_at: "2026-07-26T23:26:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: df8ce87e607ce1b22d91e4ed8702d500bccd4d4f59dab7b0eafac565e730d48a
    source_path: nodes/computer-use.md
    workflow: 16
---

Bilgisayar kullanımı, gateway aracısının yetenekli ve eşleştirilmiş bir masaüstünü görmesini ve denetlemesini sağlar. Uygunluk, yeteneklere dayanır: bağlı Node hem `computer.act` hem de `screen.snapshot` yeteneklerini bildirmeli ve ikincisinin sonucu bir `displayFrameId` içermelidir. Araç, referans karesi olarak bir ekran görüntüsü yakalar, ardından tehlikeli `computer.act` komutu üzerinden işaretçiyi ve klavyeyi yönlendirir. Eylem kümesi, temel Anthropic bilgisayar kullanımı eylemlerini izler; isteğe bağlı `computer_20251124` yakınlaştırma özelliği sunulmaz. Görüntü işleme yeteneğine sahip bir model, yerleşik `computer` aracı aracı üzerinden bunu yönlendirir.

Aracı tek tip bir komut olan `computer.act` komutunu gönderir; bir Node'un bunu nasıl yerine getirdiğini bilemez. Birlikte sunulan macOS uygulaması, gömülü Peekaboo hizmetleri ve dar kapsamlı CoreGraphics temel işlevleriyle komutu süreç içinde işler (doğru TCC izinleri, ek süreç yoktur). Windows ve Linux, ayrı olarak yüklenen bir `cua-driver` ikili dosyasıyla isteğe bağlı, deneysel `cua-computer` Plugin'ini kullanabilir. Her iki yerine getirici de aynı eşleştirme ve etkinleştirme politikasını kullanır.

## Gereksinimler

- Hem `computer.act` hem de `screen.snapshot` yeteneklerini bildiren, `screen.snapshot` çağrısı `displayFrameId` döndüren eşleştirilmiş ve bağlı bir Node.
- **macOS yerine getiricisi:** **Allow Computer Control** uygulama ayarı etkinleştirilmiş olmalıdır (varsayılan: kapalı).
- **macOS yerine getiricisi:** OpenClaw'a **Accessibility** izni (işaretçi/klavye girdisi için) ve **Screen Recording** izni (`screen.snapshot` için) verilmiş olmalıdır.
- **Windows/Linux yerine getiricisi:** birlikte sunulan `cua-computer` Plugin'i etkinleştirilmiş ve uyumlu bir `cua-driver` 0.10.x yürütülebilir dosyası yüklenmiş olmalıdır.
- `computer.act` komutu gateway'de etkinleştirilmiş olmalıdır (tehlikelidir ve varsayılan olarak devre dışıdır).
- Görüntü işleme yeteneğine sahip bir aracı modeli.
- `computer` aracını sunan bir araç politikası. Varsayılan `coding` profili bunu sunmaz. `computer` değerini `tools.alsoAllow` öğesine ekleyin; korumalı alan aracılarının ayrıca bunu `tools.sandbox.tools.alsoAllow` içinde bulundurması gerekir.

## `computer` aracı aracı

Yerleşik `computer` aracı, çağrı başına bir eylem alır. Koordinatlar, en son ekran görüntüsündeki negatif olmayan tam sayı piksellerdir; Node bunları ekran noktalarına eşler. Koordinat eylemleri, ekran görüntüsü sonucundaki `frameId` değerini aynen göndermeli ve açıkça belirtilen bir `screenIndex` bu kareyle eşleşmelidir. OpenClaw ayrıca Node tarafından verilen ekran kimliğini ekran görüntüsünden eyleme taşır; böylece ekranın yeniden bağlanması veya geometrisinin değişmesi, aynı dizini sessizce yeniden hedeflemek yerine güvenli biçimde başarısız olur. Bu denetimler tahmin edilmiş belirteçleri ve başka bir teslim edilmiş kareye veya ekrana ait belirteçleri reddeder. Belirteç güncellik garantisi değildir: uygulamalar, yakalamadan sonra aynı ekrandaki pikselleri değiştirebilir; bu nedenle sahne değişmiş olabilecekse yeni bir ekran görüntüsü alın.

- Okuma: `screenshot`.
- İşaretçi: `left_click`, `right_click`, `middle_click`, `double_click`, `triple_click`, `mouse_move`, `left_click_drag` (`startCoordinate` ile), `left_mouse_down`, `left_mouse_up`.
- Kaydırma: `scrollDirection` (`up|down|left|right`) ve `scrollAmount` (fare tekerleği adımları) ile `scroll`.
- Klavye: `type` (metin), `key` (`cmd+shift+t` veya `Return` gibi bir tuş birleşimi), `hold_key` (`text` tuş birleşimi `duration` saniye basılı tutulur).
- Hız ayarı: `wait` (`duration` saniye).

Değiştirici tuşlar, tıklama ve kaydırma eylemlerindeki `text` alanıyla gönderilir (`shift`, `ctrl`, `alt`, `cmd`). Bir giriş eyleminden sonra araç, modelin sonucu gözlemleyebilmesi için yeni bir ekran görüntüsü döndürür. Bilgisayar kullanma yeteneğine sahip birden fazla Node bağlıysa `node` değerini açıkça iletin.

Ekran görüntüleri **yalnızca modele özel** tutulur: hiçbir zaman sohbet kanalına otomatik olarak teslim edilmez. Ekrandaki tüm içeriği güvenilmeyen girdi olarak kabul edin; araç, modeli kullanıcının isteğiyle çelişen ekran talimatlarını izlememesi konusunda uyarır.

## Windows ve Linux (deneysel, cua-driver üzerinden)

Birlikte sunulan `cua-computer` Plugin'i, Windows ve Linux Node ana makineleri için deneysel bir yerine getirici sağlar. Varsayılan olarak devre dışıdır ve ön sürüm 0.10.x sürücü sözleşmesini gerektirir:

1. [Üst kaynak sürümlerinden](https://github.com/trycua/cua/releases) bir `cua-driver` 0.10.x ikili dosyası yükleyin ve bunu `PATH` üzerinde kullanılabilir hâle getirin. Başka bir yürütülebilir dosya konumu kullanmak için `plugins.entries.cua-computer.config.driverPath` değerini ayarlayın.
2. Plugin'i etkinleştirin:

   ```bash
   openclaw plugins enable cua-computer
   ```

3. `openclaw node run` öğesini etkileşimli masaüstü oturumundan başlatın. Plugin, ilk yakalama veya eylem geldiğinde yerel sürücü arka plan programını gecikmeli olarak başlatır.

Bu yerine getirici şu anda yalnızca birincil ekranı denetler. X11/XWayland, Linux için ilk tercih edilen yoldur. Yerel Wayland hâlâ üst kaynakta isteğe bağlıdır: Node'u başlatmadan önce `CUA_DRIVER_RS_ENABLE_WAYLAND` değerini kendiniz ayarlayın; OpenClaw bunu hiçbir zaman otomatik olarak ayarlamaz. KDE/KWin, üst kaynağın yerel Wayland giriş yolu tarafından desteklenmez. cua-driver 0.10.x, platformlar arası masaüstü kapsamlı basılı tutma sözleşmesine sahip olmadığından `hold_key`, `left_mouse_down` ve `left_mouse_up` kullanılamaz. Değiştirici tuş basılıyken kaydırma ve sürükleme her iki platformda da, değiştirici tuş basılıyken tıklama ise Linux'ta kullanılamaz. `key` eylemi adlandırılmış tuşları, harfleri ve değiştirici tuş birleşimlerini (örneğin `cmd+c` veya `Return`) kabul eder; sürücü bunların klavye düzenine bağlı Shift durumunu düşürdüğü için rakam ve noktalama tuşları reddedilir, dolayısıyla bu metni bunun yerine `type` eylemi üzerinden gönderin. Metin yazma, bir `type_text` sürücü çağrısının ortasında iptal edilemez.

cua-driver kararlı bir ekran kimliği bildirmediğinden kare yetkilendirmesi, sürücü bağlantısına ve birincil ekranın canlı geometrisine bağlanır. Arka plan programının veya oturumun yeniden bağlanması bekleyen kareleri geçersiz kılar; ancak bağlantıyı açık tutan ve aynı geometriye sahip bir birincil ekran değişimi algılanamaz. Bu yerine getirici için kararlı, tek ekranlı bir oturumu tercih edin.

OpenClaw, yönettiği `mcp` ve `serve` süreçleri için cua-driver telemetrisini ve güncelleme denetimlerini devre dışı bırakır. Sürücü ikili dosyasını indirmez veya güncellemez.

### Sorun giderme

`cua-computer` yerine getiricisi, araç sonucunda ve Node günlüklerinde türü belirtilmiş hata kodları sunar. Yaygın olanlar:

| Kod                                                 | Neden                                                                                                                                                           | Çözüm                                                                                                                                                                                                                                  |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `COMPUTER_DRIVER_UNAVAILABLE`                        | `cua-driver` ikili dosyası `PATH` üzerinde değildir (veya `driverPath` yanlıştır), arka plan programı zamanında hazır olmamıştır ya da Node Windows/Linux değildir.                 | `cua-driver` 0.10.x sürümünü `PATH` üzerine yükleyin veya `driverPath` değerini ayarlayın. `openclaw node run` öğesini etkileşimli masaüstü oturumunda çalıştırın; Linux'ta bir X11 `DISPLAY` öğesinin (veya `CUA_DRIVER_RS_ENABLE_WAYLAND` içeren bir `WAYLAND_DISPLAY` öğesinin) bulunduğundan emin olun. |
| `COMPUTER_DRIVER_UNSUPPORTED`                        | Bağlı sürücü `cua-driver` 0.10.x değildir veya yetenek/şema sürümü farklıdır.                                                                      | Desteklenen bir 0.10.x derlemesi yükleyin. Plugin, sorun giderildikten yaklaşık 30 saniye sonra yeniden yoklama yapar; bu nedenle Node'u yeniden başlatmak gerekmez.                                                                                                          |
| `COMPUTER_REFUSED_<code>`                            | Sürücü, `background_unavailable`, `background_occluded` veya `foreground_unavailable` (KDE/KWin Wayland) gibi yapılandırılmış bir kodla eylemi reddetmiştir.   | Hedef pencereyi öne getirin, X11'e geçin veya desteklenen bir birleştirici kullanın. Yukarıdaki uyumluluk notlarına bakın.                                                                                                                    |
| `COMPUTER_STALE_FRAME`                               | Koordinatlar artık güncel olmayan bir ekran görüntüsüne başvurmuştur (bağlam sıkıştırması, ekran geometrisi değişikliği veya referans genişliği değişikliği).                 | Koordinat eyleminden önce yeni bir `screenshot` alın.                                                                                                                                                                              |
| `COMPUTER_UNSUPPORTED_ACTION`                        | Bu yerine getiricinin aslına uygun biçimde teslim edemeyeceği bir eylem: `hold_key`, `left_mouse_down`, `left_mouse_up`, değiştirici tuş basılıyken sürükleme/kaydırma veya Linux'ta değiştirici tuş basılıyken tıklama. | Desteklenen bir eylem kullanın. cua-driver 0.10.x, masaüstü kapsamlı basılı tutulan giriş sözleşmesine sahip değildir.                                                                                                                                                  |
| `COMPUTER_UNSUPPORTED_DISPLAY`                       | Birincil olmayan bir `screenIndex`, yakalama/ekran geometrisi uyuşmazlığı veya birincil ekranın dışında kalan bir imleç.                                                       | Yalnızca birincil ekranı yönlendirin.                                                                                                                                                                                                      |
| `COMPUTER_UNSUPPORTED_KEY`                           | Sürücünün güvenilir biçimde yeniden üretemeyeceği bir `key` değeri: Shift durumu klavye düzenine bağlı bir rakam veya noktalama tuşu ya da bilinmeyen bir tuş.                        | Bu metni bunun yerine `type` eylemi üzerinden gönderin.                                                                                                                                                                                    |
| `COMPUTER_DRIVER_ERROR` / `COMPUTER_INVALID_REQUEST` | Sürücü yapılandırılmış bir kod olmadan başarısız olmuştur veya eylem bağımsız değişkenleri hatalı biçimlendirilmiştir.                                                                            | Sürücü durumunu denetleyip yeniden ekran görüntüsü alın; eylem bağımsız değişkenlerini düzeltin.                                                                                                                                                        |

## `computer.act` Node komutu

`computer.act`, aracın girdiyi yönlendirdiği tek Node komutudur (`command: "computer.act"` ile `node.invoke`). Özellikleri:

- **Varsayılan olarak tehlikelidir**: yerleşik tehlikeli Node komutları arasında listelenir ve açıkça etkinleştirilene kadar çalışma zamanı izin listesinin dışında tutulur. macOS, Windows ve Linux masaüstü Node'ları, yüzeyin bir kez onaylanması için eşleştirme sırasında bunu yine de bildirebilir.
- **Yetenek tabanlıdır**: araç, bağlı bir Node'un hem `computer.act` hem de `screen.snapshot` yeteneklerini bildirmesini gerektirir. Birlikte sunulan macOS uygulaması ve isteğe bağlı deneysel `cua-computer` Plugin'i aynı komut çiftini yerine getirir.

Okumalar `screen.snapshot` öğesini yeniden kullanır; ikinci bir yakalama yolu yoktur. Paylaşılan yakalama komutu için [Kamera ve ekran Node'ları](/tr/nodes/camera) bölümüne bakın.

## Etkinleştirme ve devreye alma

1. Platform yürütücüsünü etkinleştirin: macOS'te **Settings → Allow Computer Control** seçeneğini etkinleştirin, ardından **Settings → Permissions** altında **Accessibility** ve **Screen Recording** izinlerini verin; Windows/Linux'ta yukarıdaki deneysel `cua-computer` kurulumunu izleyin.
2. Gateway üzerinde eşleştirme güncellemesini onaylayın (yeni bir komut yeniden eşleştirmeyi zorunlu kılar).
3. Aracı görsel algılama özellikli ajanın kullanımına açın. Varsayılan `coding` profili için:

   ```json5
   {
     tools: {
       alsoAllow: ["computer"],
       // Korumalı alandaki ajanlar için bu ikinci geçit de gereklidir:
       sandbox: { tools: { alsoAllow: ["computer"] } },
     },
   }
   ```

4. Sınırlı bir süre için `computer.act` özelliğini etkinleştirin. `phone-control` plugini bir `computer` grubu sunar:

   ```text
   /phone arm computer 30m
   /phone status
   /phone disarm
   ```

   Etkinleştirme için `operator.admin` (veya sahip) gerekir ve yetki süresi otomatik olarak dolar. Eski `/phone arm all` grubu masaüstü denetimini kasıtlı olarak kapsam dışı bırakır; açıkça belirtilen `computer` grubunu kullanın. Etkinleştirme yalnızca Gateway'in neleri çağırabileceğini değiştirir; Node uygulaması, macOS'teki **Allow Computer Control**, Accessibility ve Screen Recording dahil olmak üzere platforma özgü ayarlarını ve işletim sistemi izinlerini uygulamaya devam eder.

Kalıcı yetkilendirme için `computer.act` değerini `gateway.nodes.commands.allow` listesine ekleyin **ve `gateway.nodes.commands.deny` listesinden kaldırın**; engelleme listesi önceliklidir. Kalıcı yetkilendirmenin süresi otomatik olarak dolmaz. `/phone arm` öncesinde zaten mevcut olan girdiler `/phone disarm` sonrasında da kalır; geçici bir izin etkinken bu izni kalıcı izne dönüştürmeyin.

Yetkilendirme, etkinleştirme ile kullanım arasında kasıtlı olarak bölünmüştür. `computer.act` özelliğini etkinleştirmek veya
kalıcı olarak yapılandırmak için yönetici yetkisi gerekir.
Etkinleştirildikten sonra `operator.write` yetkisine sahip kimliği doğrulanmış bir operatör,
iznin süresi dolana veya izin devre dışı bırakılana kadar `computer.act` öğesini `node.invoke` üzerinden
çağırabilir; her eylem için ayrı bir yönetici denetimi yoktur. `computer.act` bildiren bir Node'un onaylanması,
yalnızca daha sonra etkinleştirilebilmesi için yüzeyi kaydeder ve
tek başına çağrıyı etkinleştirmez.

## Güvenlik

- Yetkilendirmeden önce her katmanın (araç politikası, Gateway komut politikası, Node uygulaması ayarı ve platform izinleri) mutabık olması gerekir. Mevcut macOS yürütücüsü için buna **Allow Computer Control**, Accessibility ve Screen Recording dahildir. Etkinleştirildikten sonra eylemler, süre dolana veya `/phone disarm` gerçekleşene kadar her eylem için ayrı bir onay olmadan yürütülür.
- macOS yürütücüsü metni her seferinde bir grafem olacak şekilde gönderir; bu nedenle iptal, bağlantı kesilmesi, duraklatma, devre dışı bırakma veya uç nokta değişimi işlemi bir sonraki grafemden önce durdurur. Deneysel cua-driver yürütücüsü, yazma işlemi sırasında bir `type_text` çağrısını iptal edemez.
- Ekran görüntüleri yalnızca modele yöneliktir ve hiçbir zaman sohbete otomatik olarak gönderilmez (sorun [#44759](https://github.com/openclaw/openclaw/issues/44759)).
- Ekran içeriğini güvenilmeyen içerik olarak değerlendirin; istem enjeksiyonu içerebilir.

## Diğer masaüstü denetimi yollarıyla ilişkisi

Bu, ajan tarafından yönlendirilen yoldur. PeekabooBridge ana makinesi, Codex Computer Use ve doğrudan `cua-driver` MCP ile ilişkisi için [Peekaboo köprüsü](/tr/platforms/mac/peekaboo) bölümüne bakın.
