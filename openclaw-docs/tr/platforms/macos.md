---
read_when:
    - macOS uygulamasını yükleme
    - macOS'te yerel ve uzak Gateway modu arasında karar verme
    - macOS uygulaması sürüm indirmeleri aranıyor
summary: OpenClaw macOS menü çubuğu uygulamasını yükleyin ve kullanın
title: macOS uygulaması
x-i18n:
    generated_at: "2026-07-27T00:06:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b319d72bcbffcf91b6bc012d352c2cf647abd66e08ab0146cf98f5edfae3bca1
    source_path: platforms/macos.md
    workflow: 16
---

macOS uygulaması, OpenClaw **menü çubuğu yardımcısıdır**: yerel tepsi kullanıcı arayüzü, macOS
izin istemleri, bildirimler, WebChat, sesli giriş, Canvas ve
`system.run` gibi Mac'te barındırılan Node araçları.

Tam bir pencere açmadan Spotlight tarzı bir ana oturum düzenleyicisi için **Hızlı Sohbet**'i kullanın. Varsayılan olarak Option-Space (⌥Space) tuşlarına basın, menü çubuğu menüsünden seçin veya **Ayarlar → Genel** bölümünde başka bir kısayol kaydedin.

Yalnızca CLI ve Gateway mi gerekiyor? [Başlarken](/tr/start/getting-started) ile başlayın.

## İndirme

macOS uygulama derlemelerini [OpenClaw GitHub sürümlerinden](https://github.com/openclaw/openclaw/releases) edinin.
Bir sürüm macOS uygulama varlıkları içerdiğinde şunları arayın:

- `OpenClaw-<version>.dmg` (tercih edilen)
- `OpenClaw-<version>.zip`

Bazı sürümler yalnızca CLI, kanıt veya Windows varlıkları içerir. En yeni sürümde
macOS uygulama varlığı yoksa bunu içeren en yeni sürümü kullanın veya
[macOS geliştirme kurulumu](/tr/platforms/mac/dev-setup) ile kaynaktan derleyin.

## İlk çalıştırma

1. **OpenClaw.app** uygulamasını yükleyip başlatın.
2. Yerel bir Gateway için **Bu Mac**'i seçin veya uzak bir Gateway'e bağlanın.
3. Uygulama eşleşen CLI çalışma zamanını yüklerken bekleyin. Yerel modda ayrıca
   Gateway'i yükleyip başlatır.
4. Canlı model denetimiyle çıkarım bağlantısını kurun. Denetim başarılı olduktan sonra kalan kurulumu
   OpenClaw tamamlar.
5. macOS izin denetim listesini tamamlayın ve ilk katılım test mesajını gönderin.

Uygulama, varsayılan aracısında yapılandırılmış bir model bulunan mevcut bir
Gateway'e ulaşırsa bu Gateway'i önceden kurulmuş olarak değerlendirir, sağlayıcı ilk katılımını ve
OpenClaw'ı atlar ve panoyu açar. Gateway'e bağlanılamazsa veya
varsayılan aracısında model yoksa kurtarma amacıyla çıkarım ilk katılımı
kullanılabilir durumda kalır.

CLI/Gateway kurulum yolu için [Başlarken](/tr/start/getting-started) sayfasını kullanın.
İzin kurtarma için [macOS izinleri](/tr/platforms/mac/permissions) sayfasını kullanın.

## Güncellemeler

Pano güncelleme kartı, uygulamanın neyi güncelleyeceğini belirtir:

- **Mac uygulamasını + Gateway'i güncelle**, imzalı uygulamanın yerel launchd
  Gateway'ini yönettiği anlamına gelir. Sparkle önce uygulamayı günceller; uygulama yeniden başlatıldıktan sonra
  Gateway'ini eşleşen sürüme otomatik olarak güncelleyip yeniden başlatır ve ardından
  bağlantıyı doğrular.
- **Gateway'i güncelle**, uygulamanın uzak bir Gateway'e, elle
  yönetilen yerel bir Gateway'e veya uygulamanın yönetmediği başka bir kuruluma bağlı olduğu anlamına gelir. Düğme,
  Mac uygulamasını değiştirmek yerine ilgili Gateway'in normal güncelleme akışını çalıştırır.

Başarısız bir eşgüdümlü güncelleme; yeniden deneme,
[güncelleme kılavuzu](/tr/install/updating) ve Discord eylemleriyle birlikte kurulum tarzı penceresinde kalır. Otomatik onarım,
daha yeni bir Gateway'in sürümünü asla düşürmez veya bir `extended-stable` kanal sabitlemesini geçersiz kılmaz.

Başarılı bir güncellemeden sonra uygulama, insanlar tarafından en son kullanılmış
üst düzey doğrudan oturumu bulur ve ilgili aracıya tek seferlik bir güncelleme olayı verir. Heartbeat
ve Cron etkinliği bu seçimi etkilemez. Ardından aracı, büyük olasılıkla kullandığınız
görüşmeden sizi yeniden karşılayabilir. Uzak modda uygulama
yalnızca yerel Mac Node çalışma zamanını günceller ve uzak Gateway uygulamadan
eskiyse bildirimi atlar.

Sparkle, Gateway'in `update.channel` ayarını izler. `beta` ve `dev`
beta uygulama derlemelerine katılım sağlar; `stable`, `extended-stable` ve eksik ya da bilinmeyen değerler
kararlı uygulama derlemelerinde kalır.

## Pano bağlantılarını açma

macOS uygulamasının gömülü panosunda harici bir web bağlantısına tıklandığında bağlantı, pano gezinmesi görünür kalacak şekilde pencere genişliğinin yarısını kaplayan ve yeniden boyutlandırılabilen bir tarayıcı kenar çubuğunda açılır. Başka bir genişlik seçmek için ayırıcıyı sürükleyin; uygulama seçiminizi hatırlar. Her bağlantı kendi sekmesinde açılır, birden çok sayfa açıkken sekme şeridi görünür ve aynı bağlantıya yeniden tıklamak mevcut sekmesini tekrar kullanır. Sekmeleri yeniden sıralamak için sürükleyin; sekme kapatma düğmesiyle veya orta tıklamayla kapatın ve **Open in Default Browser**, **Copy Link**, **Reload**, **Close Tab** ve **Close Other Tabs** seçenekleri için bir sekmeye sağ tıklayın. Pencerenin başlık çubuğundaki geri/ileri denetimleri ve izleme dörtgeni kaydırmaları pano geçmişinde gezinir; kenar çubuğunun kendi geri/ileri denetimleri etkin sekmenin geçmişinde gezinir. Kenar çubuğunda ayrıca yeniden yükleme, varsayılan tarayıcıda açma ve kapatma denetimleri bulunur.

Başlık çubuğu denetimleri uygulama kenar çubuğuna göre konumlanır: kenar çubuğu genişletilmişken geri/ileri denetimleri, kenar çubuğu düğmesinin yanında sağ kenarda bulunur; daraltılmışken ise yerlerini bir arama düğmesine (komut paletini açar) ve yeni oturum düğmesine bırakırlar.

**Open in Sidebar**, **Open in Default Browser** veya **Copy Link** seçeneklerinden birini belirlemek için harici bir bağlantıya sağ tıklayın. Panodan yapılan değiştirici tuşlu tıklamalar ve kullanıcı tarafından etkinleştirilen yeni pencere bağlantıları varsayılan tarayıcıda açılmaya devam eder; kenar çubuğundaki yeni pencere bağlantıları yeni kenar çubuğu sekmeleri olarak açılır. Tarayıcıda barındırılan normal Control UI sayfaları, tarayıcının olağan bağlantı ve bağlam menüsü davranışını korur.

## Tarayıcı oturumlarını içe aktarma

Uygulama yerel bir Gateway ile çalışırken tarayıcı kenar çubuğu ilk kez açıldığında, Mac'te çerezleri bulunan Chrome ailesinden bir profil varsa pano kapatılabilir bir banner gösterir. Banner, bu çerezleri aracıların gezinmek için kullandığı yalıtılmış ve yönetilen bir profile kopyalamayı önerir. **Import** denetiminden bir profil seçin (Touch ID gerekebilir); ilerleme ve içe aktarılan çerez sayısı satır içinde görünür ve yalnızca çerezler kopyalanır — parolalar kaynak tarayıcıdan asla ayrılmaz. Banner'ı kapatmak tercihi kaydeder; **Ayarlar → Genel → Tarayıcı oturumu → İçe aktar…** seçeneği bunu istediğiniz zaman yeniden sunar. Temel içe aktarma akışı ve `browser.allowSystemProfileImport` geçidi için [Tarayıcı](/tr/cli/browser) sayfasına bakın.

## Gateway modu seçme

| Mod    | Kullanım koşulu                                                                  | Ayrıntı sayfası                                     |
| ------ | -------------------------------------------------------------------------------- | -------------------------------------------------- |
| Yerel  | Gateway'i bu Mac çalıştırmalı ve launchd ile çalışır durumda tutmalıdır.          | [macOS'te Gateway](/tr/platforms/mac/bundled-gateway) |
| Uzak   | Gateway'i başka bir ana makine çalıştırır; bu Mac onu SSH, LAN veya Tailnet üzerinden denetler. | [Uzaktan denetim](/tr/platforms/mac/remote)            |

Uygulama Node ana makine çalışma zamanını yeniden kullandığı için her iki modda da kurulu bir `openclaw`
CLI gerekir. Yeni bir Mac'te uygulama eşleşen CLI'yi otomatik olarak yükler; ardından yerel
mod Gateway sihirbazını başlatırken uzak mod ikinci bir yerel Gateway
başlatmadan seçilen Gateway'e bağlanır.
Elle kurtarma için [macOS'te Gateway](/tr/platforms/mac/bundled-gateway) sayfasına bakın.

## Uygulamanın yönettiği öğeler

- Menü çubuğu durumu, bildirimler, sistem sağlığı, WebChat ve yüzen Hızlı Sohbet çubuğu.
- Ekran, mikrofon, konuşma, otomasyon ve erişilebilirlik için macOS izin istemleri.
- Yerel Canvas, kamera/ekran yakalama, bildirimler,
  konum ve bilgisayar denetimini; CLI Node ana makinesinin sistem, tarayıcı,
  Plugin, Skills ve MCP komutlarıyla birleştiren tek bir Mac Node'u.
- Mac'te barındırılan komutlar için yürütme onayı istemleri.
- Paylaşılan Node politikasını CLI çalışma zamanı yönetirken uygulamanın macOS
  izin ilişkilendirmesini koruyarak onaylı kabuk komutlarını uygulama bağlamında yürütme.
- Uzak mod SSH tünelleri veya doğrudan Gateway bağlantıları.

Gömülü Control UI'da **Ayarlar → Bildirimler**, uygulama bildirimleri yerel olarak
ilettiği için tarayıcı anlık bildirim izni yerine uygulamanın yerel bildirim iznini gösterir.

Uygulama, Gateway veya genel CLI belgelerinin yerini **almaz**. Gateway
yapılandırması, sağlayıcılar, Plugin'ler, kanallar, araçlar ve güvenlik kendi
belgelerinde yer alır.

## macOS ayrıntı sayfaları

| Görev                                    | Okunacak kaynak                                                                            |
| ---------------------------------------- | ------------------------------------------------------------------------------------------- |
| CLI/Gateway hizmetini yükleme veya hata ayıklama | [macOS'te Gateway](/tr/platforms/mac/bundled-gateway)                                    |
| Durumu bulutla eşitlenen klasörlerin dışında tutma | [macOS'te Gateway](/tr/platforms/mac/bundled-gateway#state-directory-on-macos)           |
| Uygulama keşfi ve bağlantı sorunlarını ayıklama | [macOS'te Gateway](/tr/platforms/mac/bundled-gateway#debug-app-connectivity)              |
| launchd davranışını anlama               | [Gateway yaşam döngüsü](/tr/platforms/mac/child-process)                                       |
| İzin veya imzalama/TCC sorunlarını düzeltme | [macOS izinleri](/tr/platforms/mac/permissions)                                             |
| En son kullandığınız Mac'i algılama      | [Etkin bilgisayar varlığı](/tr/nodes/presence)                                                  |
| Uzak bir Gateway'e bağlanma              | [Uzaktan denetim](/tr/platforms/mac/remote)                                                     |
| Menü çubuğu durumunu ve sistem sağlığı denetimlerini okuma | [Menü çubuğu](/tr/platforms/mac/menu-bar), [Sistem sağlığı denetimleri](/tr/platforms/mac/health) |
| Gömülü sohbet kullanıcı arayüzünü kullanma | [WebChat](/tr/platforms/mac/webchat)                                                          |
| Sesle uyandırmayı veya bas-konuş özelliğini kullanma | [Sesle uyandırma](/tr/platforms/mac/voicewake)                                         |
| Canvas'ı ve Canvas derin bağlantılarını kullanma | [Canvas](/tr/platforms/mac/canvas)                                                       |
| Kullanıcı arayüzü otomasyonu için PeekabooBridge barındırma | [Peekaboo köprüsü](/tr/platforms/mac/peekaboo)                             |
| Komut onaylarını yapılandırma            | [Yürütme onayları](/tr/tools/exec-approvals), [gelişmiş ayrıntılar](/tr/tools/exec-approvals-advanced) |
| Mac Node komutlarını ve uygulama IPC'sini inceleme | [macOS IPC](/tr/platforms/mac/xpc)                                                     |
| Günlükleri yakalama                      | [macOS günlük kaydı](/tr/platforms/mac/logging)                                                 |
| Kaynaktan derleme                        | [macOS geliştirme kurulumu](/tr/platforms/mac/dev-setup)                                       |

## İlgili

- [Platformlar](/tr/platforms)
- [Başlarken](/tr/start/getting-started)
- [Gateway](/tr/gateway)
- [Yürütme onayları](/tr/tools/exec-approvals)
