---
read_when:
    - Bulut worker sağlama, worker modu veya oturum devri tasarlama ya da uygulama
    - Ortamları değiştirme.*, çalışan protokolü, transkript alımı veya çıkarım proxy'si RPC'leri
    - Uzak agent yürütmesinin güvenlik duruşunu inceleme
summary: Gateway üzerinden proxy'lenen çıkarım ve canlı kenar çubuğu akışıyla, geçici ve SSH üzerinden erişilebilen makinelerde ajan oturumları çalıştırın.
title: Bulut çalışanları planı
x-i18n:
    generated_at: "2026-07-27T00:04:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 134c3f6e486837607225d95d12a3153525b14237b362b9f9957313d9bc379dc4
    source_path: plan/cloud-workers.md
    workflow: 16
---

## Durum

Teklif, revizyon 3. Uygulanmadı. Yön 2026-07 tarihinde kararlaştırıldı; revizyon 2, karşıt inceleme bulgularını (özel worker protokolü, yerleşim/ortam durum makineleri, git-duyarlı gelen eşitleme, tek yönlü v1 devri, denetimli çıkış güvenliği ifadeleri) içeriyordu. Revizyon 3, eşitleme sahipliği modelini kesinleştirir (commit'leri worker oluşturur, gateway benimser ve yayımlar), git içermeyen düz bir eşitleme modu ekler, worker exec'i kutu içinde tam yetkili olacak şekilde düzeltir, internet politikasını hazırlama aşamasına taşır ve agent gönderimini 3. kilometre taşına geri getirir.

## Sorun

OpenClaw agent oturumları döngülerini, araçlarını ve çıkarımlarını tek bir makinedeki gateway işlemi içinde çalıştırır. İşlem kapasitesi bu makineyle sınırlıdır, uzun görevler makineyi meşgul eder ve paralel işler aynı kapasite için rekabet eder. Barındırılan ürünler (Cursor cloud agents, web üzerinde Claude Code, Codex cloud) bunu görev başına geçici bulut sandbox'larıyla çözer, ancak sağlayıcı altyapısına ve sağlayıcıya güvenilmesini gerektirir.

Halihazırda yedek makinelere sahip olan (veya bunları düşük maliyetle kiralayabilen) operatörlerin şunu söylemesinin bir yolu yoktur: bu oturumu şurada çalıştır, diğer tüm oturumlar gibi kenar çubuğumda göster ve sonrasında makineyi ortadan kaldır.

## Hedefler

- Tam bir agent oturumunu (döngü + araçlar) geçici bir uzak makinede ("cloud worker") çalıştırırken oturumun Control UI'da yerel bir oturumla tamamen aynı şekilde görünmesini ve akış sağlamasını mümkün kılmak.
- Worker üzerinde kalıcı kimlik bilgileri bulundurmamak (sağlayıcı kimlik doğrulaması ve forge token'ları yok) ve doğrudan ağ çıkışına izin vermemek; kutunun yalnızca erişilebilir bir sshd'ye ihtiyacı vardır.
- Hazırla, eşitle, çalıştır, topla, yok et — tamamen otomatik ve sağlayıcı takılabilir yapıda (ilk sağlayıcı: Crabbox tarzı kiralama CLI'ları).
- Çalışan işi bir tur sınırında gateway'den bir worker'a; transkripti, oturum kimliğini veya (istek baytları eşdeğer kaldığında) sağlayıcı önbelleği yakınlığını kaybetmeden göndermek ve sonuçları güvenli biçimde geri çekmek.
- Hem insanların (UI) hem de agent'ların (araç) işi bir cloud worker'a gönderebilmesini sağlamak.
- Günlerce süren oturumları desteklemek; kullanım ömrü sabit kodlanmış bir sınır değil, politikadır.

## Hedef dışı konular (v1)

- Worker'larda harici kodlama donanımları (Claude Code, Codex CLI) yoktur. Worker oturumları yalnızca OpenClaw'ın gömülü çalıştırıcısını çalıştırır. Donanımlar kendi kimlik bilgileriyle kendi çıkarımlarını gerçekleştirdiği için donanım desteği v2'de isteğe bağlıdır.
- En iyi N / paralel deneme dallandırması yoktur.
- VPN/tailnet bağımlılığı yoktur. Aktarım yalnızca SSH üzerinden yapılır.
- Yeni bir sandbox çalışma zamanı yoktur. Yalıtım sınırı worker makinesidir; kutu içi işletim sistemi sandbox'ı daha sonra katman olarak eklenebilir.
- v1'de simetrik canlı geçiş yoktur: gönderim yerel → worker yönündedir; worker → yerel geçişi, durdurulmuş bir oturum ve tamamlanmış çalışma alanı uzlaştırması gerektirir. Canlı çift yönlü devir daha sonra aynı bariyer mekanizması üzerine kurulacaktır.
- Gateway üzerinde JSON yan durumu yoktur; ortam, yerleşim, imleç ve yetki durumu SQLite içinde tutulur.

## Önceki çalışmalar (neyi kopyalıyoruz, neyi tersine çeviriyoruz)

- Cursor cloud agents: agent döngüsü kendi bulutlarında çalışır; VM bir araç yürütme hedefidir; yalnızca eklemeli konuşma deposu tüm istemcilere aktarılır; kurulum sonrası anlık görüntüyle sıcak başlangıç yapılır; kendi kendine barındırılan worker'lar yalnızca dışarıya bağlanan worker işlemleridir. "Konuşmanın doğruluk kaynağı orkestratörde kalır" yaklaşımını ve akış modelini kopyalıyoruz; döngünün yerleşimini tersine çeviriyoruz (aşağıdaki karara bakın).
- Codex cloud: iki aşamalı çalışma zamanı — ağ bağlantılı kurulum aşaması, ardından gizli bilgilerin kaldırıldığı çevrimdışı agent aşaması; hızlı takip çalışmaları için kapsayıcı durumu önbelleği. Çıkış yaklaşımımız olarak aşama ayrımını, v2 sıcak imajları içinse önbellek fikrini kopyalıyoruz.
- Web üzerinde Claude Code: oturum başına VM; kimlik bilgilerini yalıtan git proxy'si (gerçek token'lar sandbox'a hiçbir zaman girmez, push oturum dalıyla sınırlandırılır); kurulumdan sonra dosya sistemi anlık görüntüsü; teleport devri = gönderilmiş dal + yeniden oynatılmış geçmiş. Kimlik bilgisi yalıtımını ve devir çerçevesini kopyalıyoruz, ancak dışarı giden eşitleme gateway'den rsync ile yapıldığından kirli çalışma ağaçları çalışır ve kutunun yakınında hiçbir yerde forge token'ı bulunmaz.
- Copilot coding agent: paket kayıt sistemi izin listesiyle varsayılan olarak reddedilen çıkış. Kararlı durum varsayılanımız daha güçlüdür (hiç doğrudan çıkış yoktur), çünkü çıkarım ve web araması SSH tüneli üzerinden gelir; ancak bunun neden "sıfır çıkış" değil, "denetimli çıkış" olduğu için Güvenlik bölümüne bakın.

## Mimari karar: döngü worker üzerinde, çıkarım gateway üzerinden

Üç yerleşim değerlendirildi:

1. Döngü gateway üzerinde kalır, worker araçları yürütür (Cursor modeli). En güvenli hata alanıdır (transkript, çıkarım, onaylar ve yeniden başlatma kurtarması yerelde kalır) ve incelemeyi yapanların tercih ettiği ilk kilometre taşıdır. Ürün mimarisi olarak reddedildi: OpenClaw'ın exec dışı araçları işlem içi dosya sistemi işlemleridir; bu nedenle her dosya okuma/düzenleme/grep işlemi bir ağ gidiş dönüşüne veya kaba çalışma alanı RPC'lerine yönelik büyük bir araç yüzeyi yeniden düzenlemesine dönüşür; çalışma zamanı davranışı yoğun iletişimlidir ve gecikmeyle sınırlıdır. Halihazırda oluşturulmuş olduğu yerde bu yaklaşımın ruhunu yeniden kullanıyoruz (exec işlemlerinin node'lara aktarılması), ancak araçları uzaklaştırma katmanını oluşturmuyoruz.
2. Döngü ve çıkarımın ikisi de worker üzerinde çalışır. En basit hata alanıdır, ancak model kimlik bilgilerinin (OAuth profilleri dahil) geçici makinelere gönderilmesi gerekir, gateway politika/yönlendirme/denetim kontrolünü kaybeder ve geçiş, sağlayıcıyı çağıran kimliği değiştirerek sağlayıcı önbelleklerini geçersiz kılar.
3. Döngü + araçlar worker üzerinde çalışır, model çağrıları gateway üzerinden proxy'lenir. Seçilen yaklaşım budur. Araç çağrısı başına değil, model turu başına bir gidiş dönüş; araçlar kodun yanında çalışır; gateway, kimlik doğrulama profillerinin, sağlayıcı yönlendirmesinin ve politikanın tek sahibi olmaya devam eder; worker hiçbir gizli bilgi tutmaz.

3\. seçeneğin maliyeti, her model turu sırasında gateway'e eşzamanlı bağımlılıktır; bu nedenle dayanıklılık kuralları sonradan düşünülen ayrıntılar değil, kararın bir parçasıdır:

- Bir turun ortasında gateway'in kaybedilmesi etkin sağlayıcı çağrısının başarısız olmasına neden olur. Tur başarısız olarak işaretlenir ve yeniden bağlandıktan sonra yeni bir tur olarak yeniden denenir; işlemdeki sağlayıcı akışının şeffaf biçimde yeniden oynatılması yoktur (çift ücretlendirme/çift araç çağrısı riski).
- Her worker↔gateway işlemi kalıcı kimlik taşır (Worker protokolüne bakın); böylece yeniden bağlantılar askıda kalmak yerine sürdürülür veya önbelleğe alınmış nihai sonuçları getirir.
- Gateway kapasitesi yönetilen bir bileşendir: eşzamanlı worker sınırları, akış denetimi ve yük azaltma v1 kapsamındadır (Kapasite bölümüne bakın).

Gateway hem transkripti depoladığı hem de tüm sağlayıcı trafiğini başlattığı için oturum konumdan bağımsızdır: döngünün gateway ile worker arasında taşınması, sağlayıcı tarafında veya UI veri yolunda hiçbir şeyi değiştirmez. Gönderimi ve geri çekmeyi ucuz kılan budur.

## Bileşenler

### 1. Ortam durum makinesi + sağlayıcı sözleşmesi

`environments.*` gateway protokolünde şu anda yalnızca durum projeksiyonudur. Kalıcı çekirdek, RPC biçimlerinden önce tasarlanan, SQLite sahipliğindeki bir ortam kaydı ve durum makinesidir:

`requested → provisioning → bootstrapping → ready → (attached|idle) → draining → destroying → destroyed | failed | orphaned`

- Hazırlama çökmeye karşı güvenlidir: niyet satırı, sağlayıcı çağrısından önce belirlenimci bir işlem kimliğiyle kalıcılaştırılır; böylece gateway yeniden başlatıldığında iki kez hazırlama yapmak veya ücretli bir makineyi sahipsiz bırakmak yerine devam eden bir kiralamayı benimseyebilir.
- Yeniden başlatma uzlaştırması ve sahipsiz kaynak temizleyicisi (sağlayıcı `inspect` ile yerel kayıtların karşılaştırılması) sağlamlaştırma değil, v1 gereksinimleridir.

Sağlayıcı sözleşmesi (Plugin tarafından uygulanır; çekirdekte sağlayıcı adı veya politika bulunmaz):

```ts
type WorkerProvider = {
  id: string;
  provision(profile: WorkerProfile, opId: string): Promise<WorkerLease>; // → ssh ana bilgisayarı/bağlantı noktası/kullanıcısı/anahtar malzemesi
  inspect(lease: { leaseId: string; profile: WorkerProfile }): Promise<LeaseStatus>; // benimseme/sağlık/sahipsiz kaynak taraması
  renew?(leaseId: string): Promise<void>; // uzun ömürlü oturumlar ile sağlayıcı TTL'leri
  destroy(lease: { leaseId: string; profile: WorkerProfile }): Promise<void>; // eşgüçlüdür, yalnızca kaldırma kanıtlandığında döner
};
```

RPC'ler: `environments.create`, `environments.destroy`, genişletilmiş `environments.list/status` (sağlayıcı, kiralama kimliği, durum, yaş, boşta kalma süresi, bağlı oturumlar). İlk sağlayıcılar: Crabbox biçimli bir kiralama CLI sarmalayıcısı (ürün yolu) ve yalnızca geliştirme amaçlı olarak işaretlenmiş statik SSH ana bilgisayarı sağlayıcısıdır — paylaşılan bir ana bilgisayardaki worker, ana bilgisayardaki ilgisiz verileri okuyabilir; bu nedenle statik ana bilgisayarlar varsayılan yaklaşım için değil, özellik geliştirme içindir.

### 2. Worker önyüklemesi: kutuya OpenClaw kurulması

Özel bir worker yapıtı ve npm kullanılabilirliğine bağımlılık yoktur:

- Tüm modlar için standart kurulum: gateway'in kendi derleme çıktısının tarball olarak paketlendiği, gateway tarafından üretilmiş ve içerik karması alınmış bir worker paketi SSH üzerinden gönderilip kutuya kurulur. Bu, tasarım gereği geliştirme derlemelerini ve yayımlanmamış commit'leri kapsar.
- `npm i -g openclaw@<exact gateway version>`, gateway yayımlanmış bir sürüm çalıştırdığında bir optimizasyondur; hiçbir zaman `latest` değildir.
- Önyükleme eşgüçlüdür; paket karması eşleşen sıcak bir kiralamada kurulum atlanır. Ham makineler ağ bağlantılı bir araç zinciri aşamasına (Node çalışma zamanı) ihtiyaç duyabilir; bu, kurulum aşamasının bir parçasıdır ve sonrasında kapatılır.
- El sıkışma, worker derleme karmasını, protokol özellik kümesini ve çalışma zamanı uyumluluğunu doğrular. Mevcut gateway sürüm/protokol denetimleri bunun için yetersizdir (SSH tünelli node'lar tam sürüm reddinden muaftır); bu nedenle worker kabulü kendi tam derleme denetimini gerçekleştirir.

Worker modu (`openclaw worker`) bir çatallanma değil, bir giriş noktasıdır: bağlantı işleme ile gömülü agent çalıştırıcısından oluşur; oturum kalıcılığı ve model çağrıları gateway RPC'leriyle desteklenir. Gateway yüzeylerini başlatmamalıdır: kanal yoktur, oturum araç kümesi dışındaki Plugin'ler otomatik olarak başlatılmaz, geçici bir durum dizini kullanılır ve yerel kimlik doğrulama profilleri bulunmaz.

### 3. Aktarım: her şey SSH üzerinden

Bağlantının sahibi gateway'dir; worker'ın sshd dışında hiçbir şeye ihtiyacı yoktur:

- Gateway worker'a SSH bağlantısı açar (kimlik bilgileri sağlayıcı kiralamasından gelir, ana bilgisayar anahtarı hazırlama çıktısından sabitlenir — `StrictHostKeyChecking=no` yoktur) ve worker'a yerel bir soketi gateway'in WS uç noktasına yönlendiren ters tünel kurar.
- Denetim/model trafiği ile çalışma alanı aktarımı, aynı sabitlenmiş güven malzemesiyle ayrı SSH bağlantıları kullanır; böylece rsync, token akışlarında sıra başı engellemesine neden olamaz.
- Tünel yaşam döngüsünün (keepalive, geri çekilmeli yeniden bağlantı) sahibi gateway üzerindeki ortam çalışma zamanıdır. Tüneldeki kısa bir kesinti oturum düzeyinde görünmez: aşağıdaki kalıcı protokol durumu worker'ın yeniden bağlanmasını ve devam etmesini sağlar.

### 4. Worker protokolü (özel; node protokolü değildir)

Mevcut node bağlantılarına karşı yapılan karşıt inceleme, bunların doğrudan yeniden kullanımını eledi: bekleyen node çağrıları bağlantıyla birlikte yok olan işlem içi promise'lerdir, node eşgüçlülük anahtarları ayrıştırılır ancak yinelenenler ayıklanmaz ve — en önemlisi — bağlı bir node sıradan node olayları (agent çalıştırma istekleri dahil) yayabilir; dolayısıyla "node türü + yetenek tavanı" bir giriş güvenliği sınırı değildir. Bu nedenle worker'lar kapalı, sürümlendirilmiş bir RPC/olay izin listesine sahip, kimliği doğrulanmış bir `worker` rolü alır; worker bağlantıları hiçbir eski node olay işleyicisine erişemez.

Kimlik ve kimlik bilgileri: hazırlama işlemi, ortam kimliğine, worker anahtarına, paket karmasına, izin verilen tek oturuma, izin verilen RPC kümesine ve sona erme zamanına bağlı, kısa ömürlü bir worker kimlik bilgisi üretir. SSH ile doğrulanan eşleştirme geçerliliğini korur (kutuyu biz hazırladık ve anahtarı tutuyoruz), ancak yetkilendirme beyan edilen node yüzeyinden değil, üretilen kimlik bilgisinden gelir.

Kalıcı işlem semantiği (biçim, mevcut ACP çalışma zamanından ve onun olay defterinden alınmıştır — kararlı tanıtıcılar, oturum başına serileştirme, kalıcı `(session, seq)` yeniden oynatma):

- Her işlem `(sessionId, lifecycleRevision, runId, ownerEpoch, streamKind, seq)` kapsamında yürütülür.
- Sahiplik dönemleri eski çalışanları sınırlar: yedek çalışan dönemi ilerletir; eski döneme ait geç sonuçlar deterministik biçimde reddedilir.
- SQLite'ta kalıcılaştırılmış ACK imleçleri ve önbelleğe alınmış terminal sonuçlarıyla en az bir kez teslim; yinelenenleri ayıklama deterministiktir. Tam olarak bir kez teslim garantisi verilmez.
- İptal, kapatma, sürdürme ve terminal sonuçları için açık çerçeveler; akışlarda kredi/pencere tabanlı akış denetimi.
- Protokol özelliği uzlaşması, genel Node protokol sürümünden bağımsızdır.

### 5. Oturum arka uç RPC'leri

İki ayrı sözleşme — mevcut kod tabanı, kalıcı transkript mutasyonlarını (oturum yöneticisinin sahip olduğu, üst/son durumuna sahip JSONL ağacı) işlem yerelindeki canlı olaylardan (akış deltaları, araç yaşam döngüsü, onaylar) ayırır ve çalışan protokolü bu ayrımı korumalıdır:

- Kalıcı transkript işlemeleri: çalışan, `runEpoch` + temel son öğe karşılaştır-ve-değiştir işlemiyle anlamsal ekleme toplu işlemleri gönderir; Gateway oturum yöneticisi girdi kimliklerini ve üst öğe kimliklerini oluşturur. Çalışan hiçbir zaman güvenilir transkript satırları, girdi kimlikleri, üst öğe kimlikleri veya yabancı oturum kimlikleri sağlayamaz.
- Yeniden oynatılabilir canlı olaylar: çalışan sıra numaraları, Gateway ACK'leri, sınırlı saklama ve geç olay sınırlaması içeren türü belirlenmiş bir olay birleşimi; sohbet görünümünün, araç satırlarının ve okunmamış/durum mantığının yerel oturumlarla aynı şekilde davranması için mevcut ajan olayı dağıtımını besler.

Çıkarım proxy'si: mevcut çalışma zamanı proxy akış istemcisinin (`src/agents/runtime/proxy.ts`) olay sözlüğünü yeniden kullanın, ancak güven sınırını taşıyın. Çalışan yalnızca oturum/çalıştırma kimliğini, onaylanmış bir model referansını, bağlamı ve kısıtlanmış üretim seçeneklerini gönderir; Gateway sağlayıcıyı, uç noktayı, kimlik doğrulamayı, başlıkları, yönlendirmeyi ve maliyet politikasını kendi kataloğundan çözümler. Çalışan tarafından sağlanan bir model nesnesi (ör. saldırgan denetimindeki `baseUrl`) reddedilir. İstek boyutu sınırları, iptal, denetim ve terminal sonucu yeniden oynatma uygulanır. Gateway'de bulunan araçlar (websearch) Gateway üzerinde çalıştırılır ve sonuçları aynı kanal üzerinden döndürür.

### 6. Çalışma alanı eşitleme

Eşitleme dayanağı, özel yerleştirme sahipliğine sahip Gateway yerelindeki bir çalışma alanıdır: git çalışma alanları için özel bir yönetilen çalışma ağacı (mevcut yönetilen çalışma ağacı meta verileri — dal, temel, anlık görüntü sahipliği — temel oluşturur); git dışı çalışma alanları için Gateway'in sahip olduğu bir hedef dizin. Asla kullanıcının canlı çalışma kopyası değildir. Oturum uzaktan yerleştirilmişken özel sahiplik, gelen eşitlemeyi tasarım gereği çakışmasız kılar.

Sahiplik ayrımı — işleme ve yayımlama:

- Çalışan tarafındaki ajan, kendi kopyasında normal şekilde işlemeler oluşturur (`git commit` yerel ve kimlik bilgisi gerektirmeyen bir işlemdir; yazar kimliği Gateway yapılandırmasından yansıtılır). Gateway bunları benimseyene kadar bu işlemeler etkisiz nesnelerdir.
- Güven gerektiren her şeyi Gateway yapar: gelen işlemelerin kaydedilmiş temel üzerine kurulduğunu doğrulama, yerel çalışma ağacını hızlı ileri alma, gönderme, PR oluşturma ve isteğe bağlı imzalama/yeniden imzalama — tümü Gateway yerelindeki kimlik bilgileriyle yapılır. Çalışan hiçbir zaman git veya kod barındırma platformu kimlik bilgilerini tutmaz ve hiçbir uzak depoya erişmez.

Çalışma alanının bir git deposu olup olmamasına göre seçilen iki eşitleme modu:

- Git modu. Giden: çalışma ağacını (işlenmemiş ve uygun izlenmeyen dosyalar dâhil; crabbox tarzı dâhil etme/hariç tutma, `.worktreeinclude` gözetilerek) tünelin SSH kimliği üzerinden rsync ile eşitleyin ve değişmez bir temel bildirim olarak kaydedin (içerik karmaları + temel işleme). Gelen: yeni işlemeler, kaydedilmiş temele karşı bir git paketi veya geçici referans olarak döner; izlenmeyen yapıtlar, boyut/tür/sembolik bağlantı kapsamı denetimleri içeren açık bir bildirim aracılığıyla döner. Benimseme, temel soyunu doğrular ve ayrışma durumunda durur — hiçbir şey iki tarafın da üzerine sessizce yazmaz. Silmeler, yeniden adlandırmalar, alt modüller ve sembolik bağlantı kaçışları rsync buluşsal yöntemleriyle değil, bildirim kurallarıyla ele alınır.
- Düz mod (git yok — ör. kutuda sıfırdan proje oluşturma). Giden, aynı rsync + temel bildirimidir. Gelen, silme yayılımıyla birlikte bildirim farkı alınmış bir yansıyı Gateway'in sahip olduğu hedef dizine geri aktarır. Git moduyla aynı nedenle güvenlidir: özel sahiplik, çakışabilecek eşzamanlı yerel düzenlemelerin bulunmadığı anlamına gelir; temel bildirim yine de beklenmedik yerel sapmayı algılar ve üzerine yazmak yerine durur.

Kontrol noktaları, günler süren oturumları kira kaybından korur: düzenli gelen kontrol noktaları (git modunda oturum dalı işlemeleri, düz modda bildirim anlık görüntüleri); sıklık profil politikasıdır (varsayılan olarak dönüş tabanlı).

### 7. Yerleştirme durum makinesi, oturumlar ve kullanıcı arayüzü

Çalışma zamanı yerleştirmesi, bir çift bağımsız satır alanı değil, oturuma anahtarlanmış ve SQLite'ın sahip olduğu bir durum makinesidir:

`local → requested → provisioning → syncing → starting → active(worker) → draining → reconciling → local | reclaimed | failed`

Ortam kimliğini, geçiş neslini, etkin sahip dönemini, çalışma alanı temel bildirimini, çalışan paketi karmasını ve son ACK imleçlerini kalıcılaştırır. Dönüş kabulü, herhangi bir döngü dönüş başlatmadan önce yerleştirmeyi atomik olarak talep eder; böylece eski bir anlık görüntüye göre kabul edilen yerel mesaj hiçbir zaman bir çalışan dönüşüyle yarışamaz — herhangi bir anda oturumun sahibi tam olarak tek bir döngüdür.

Kullanıcı arayüzü:

- Çalışan oturumu, yerleştirme meta verilerine sahip sıradan bir oturum satırıdır. Normal depoda bulunur, `sessions.list` aracılığıyla listelenir ve mevcut abonelikler aracılığıyla akar — kenar çubuğu ve sohbet için yeni bir veri yolu gerekmez; yalnızca sunum gerekir: çalışan rozeti ve yerleştirme/ortam durumu (`provisioning / syncing / running / idle / reconciling / reclaimed`).
- Oluşturma kullanıcı deneyimi: oturum hedef çubuğuna (oturumlar kenar çubuğu yeniden tasarımı), Gateway ve Node'un yanında bir bulut çalışanı hedefi eklenir. Yapılandırılmış bir sağlayıcı profili gerektirir; özellik yapılandırılana kadar görünmez.
- Ajan gönderimi: bir oturum aracı, ajanın işi bir insanın yaptığı gibi bir bulut çalışanına devretmesini sağlar (çalışan destekli alt oturum, alt ajan tarzında). İnsan gönderimiyle aynı dönüm noktasında sunulur ve aynı isteğe bağlı sağlayıcı yapılandırmasıyla sınırlandırılır. Özyineleme yapısal olarak sınırlandırılır (çalışan oturumları v1'de kendileri çalışan gönderemez); harcama denetimi kota mekanizması değil, ortam başına muhasebe/denetimdir.

## Gönderim ve devretme

v1 bilinçli olarak asimetriktir:

- Yerel → çalışan (gönderim): aşağıdaki geçiş engelini aşın, bir çalışan sağlayın veya yeniden kullanın, eşitleyin, yerleştirmeyi değiştirin; sonraki dönüş uzaktan yürütülür.
- Çalışan → yerel (geri çekme): oturumu durdurun (çalışanı aynı engele göre boşaltın), gelen uzlaştırmayı tamamlayın, yerleştirmeyi yerele değiştirin. Canlı geçiş değildir.
- Simetrik canlı devretme (etkin olarak çalışan bir oturumu durdurmadan iki yönde taşıma), aynı engel ve uzlaştırma mekanizmasını yeniden kullanır ve hata ekleme testleri engeli doğruladıktan sonra sunulur.

Geçiş engeli ("dönüş sınırı" tek başına yetersizdir — onaylar, arka plan işlemleri ve kilit bırakıldıktan sonraki transkript birleştirmeleri bu sınırın ötesine geçebilir):

1. Yeni dönüş kabulünü durdurun (yerleştirme talebi).
2. Etkin çalıştırmaları iptal edin veya boşaltın.
3. Bekleyen exec onaylarını ve yürütme izinlerini iptal edin.
4. Transkript yan yazmalarını ve canlı olay ACK'lerini boşaltın.
5. Çalışan alt işlemlerini sonlandırın.
6. Sahip dönemini ilerleterek eski sahibi sınırlayın.
7. Çalışma alanını uzlaştırın (gelen, çakışma farkındalıklı).
8. Yeni sahibi etkinleştirin.

Önbellek yakınlığı: sağlayıcı istekleri her iki yerleştirmede de Gateway'den kaynaklandığı için serileştirilmiş sağlayıcı isteği eşdeğer kaldığında önbellek yakınlığı korunur — aynı araç sırası, sistem talimatları, sağlayıcı sarmalayıcıları ve önbellek meta verileri (bunlar Gateway tarafında kalır). Bu bir varsayım değil, test edilebilir bir özelliktir: desteklenen her sağlayıcı aktarımı için yerel/çalışan yerleştirmesi arasındaki bayt eşdeğerliği testleri, çalışan döngüsünü kullanıma sunan dönüm noktasının parçasıdır.

## Güvenlik modeli

Kesin ifade etmek gerekirse: çalışanın doğrudan ağ çıkışı ve kalıcı sağlayıcı/kod barındırma platformu kimlik bilgileri yoktur. Bu, "sıfır çıkış" değildir — çıkarım ve Gateway tarafından çalıştırılan araçlar denetimli çıkış kanallarıdır (istem enjeksiyonuna uğramış bir çalışan yine de çalışma alanı baytlarını model bağlamına veya websearch sorgularına yerleştirebilir). Buna göre:

- Denetimli çıkış muhasebesi: çıkarım proxy'sinde ve Gateway araçlarında ortam başına denetim ve operatör tarafından görülebilen muhasebe. Hız/bayt sınırları harcama kotası mekanizması olarak değil, protokol akış denetimi (kapasite) olarak bulunur.
- Çalışandan Gateway'e giriş, kapalı çalışan protokolü izin listesidir; transkript yazmaları yapısal olarak kısıtlanır (Gateway tarafından oluşturulan kimlikler, tek bağlı oturum).
- Çalışan exec'i kutu içinde tam izinlidir. Kutu tek kullanımlık ve kimlik bilgisi içermediğinden komut başına onay hiçbir şeyi korumadan sürtünme ekler; korunan sınır, gelen uzlaştırma ve denetimdir. Exec hiçbir zaman Gateway Node onay yolundan geçmez.
- İnternet politikası, sağlama zamanında verilen bir sağlayıcı kararıdır: ortam profili kutu oluşturulurken karar verir (güvenlik duvarı/güvenlik grubu/çıkışsız ağ); isteğe bağlı olarak sağlayıcının ajan aşamasından önce kapattığı ağ bağlantılı bir kurulum aşaması bulunabilir. Çekirdek, çalışma zamanı ağ anahtarı uygulamaz.
- Sağlama zamanında kutu temizliği: bulut meta veri uç noktası engellenir veya bulunmadığı doğrulanır, örnek profili yoktur, devralınmış SSH ajanı yoktur, Docker soketi yoktur, ortam/ana dizin temizdir. SSH ana makine anahtarları sağlama çıktısından sabitlenir.
- Gateway tarafındaki her şey (gönderme, PR, sağlayıcı çağrıları) için onaylar ve politika Gateway üzerinde çalışmaya devam eder.

Ele geçirilmiş bir çalışan oturumunun etki alanı: eşitlenmiş çalışma alanı kopyası ve denetlenen proxy kanallarının izin verdiği kapsam — kimlik bilgisi yok, doğrudan ağ yok, izin listesinin ötesinde Gateway yüzeyi yok.

## Kapasite

Gateway, N çalışan için her istemi ve token akışını aktarır; bu nedenle v1, kapasiteyi üretimde keşfetmek yerine bir kapasite modeli tanımlar: Gateway başına eşzamanlı çalışan sınırları, akış başına kredi pencereleri (mevcut olay akışı kuyruğu sınırsızdır ve Node soket arabelleği tavanı yavaş tüketicileri zorla kapatır — ikisi de değiştirilmeden kullanıma uygun değildir), ani yükler için sınırlı disk biriktirme ve kullanıcı arayüzünde görünür geri basınç durumlarıyla yük azaltma. Çalışma alanı aktarımı kendi SSH kanalında kalır.

## Yaşam döngüsü

- Boştayken otomatik durdurma ve TTL, sabit sabitler değil sağlayıcı profili politikasıdır. Varsayılanlar, açık bir canlı tutma seçeneğiyle cömerttir; günler süren çalışma birinci sınıftır (kira tabanlı arka uçlar için sağlayıcı `renew` bulunur); devam eden bir dönüşü veya yakın zamanda etkinliği olan oturum hiçbir zaman geri alınmaz.
- Çalışan öldüğünde veya geri alındığında: yerleştirme `reclaimed` durumuna geçer, oturum satırı kalır, sonraki mesaj yeni bir çalışan sağlar ve son kontrol noktasından yeniden eşitler. Konuşma hiçbir zaman kaybolmaz (Gateway tarafındaki depo); son kontrol noktasından sonraki çalışma alanı değişiklikleri kaybolur ve kullanıcı arayüzü bunu belirtir.
- İlk günden sıcak kira yeniden kullanımı (bunu destekleyen sağlayıcılar); önyüklemeden sonraki görüntü anlık görüntüsü, v2 hızlı başlatma yoludur.

## Yapılandırma yüzeyi

Asgari ve isteğe bağlıdır: bir sağlayıcı profili bloğu (sağlayıcı kimliği, kimlik bilgileri/CLI referansı, eşitleme kuralları, yaşam süresi politikası, bütçeler, isteğe bağlı kurulum aşaması) ve oturum başına yerleştirme seçimi. Yeni ortam değişkeni yoktur. Yapılandırılmamış kurulumlarda hiçbir şey görünmez.

## Dönüm noktaları

Uygulama, küçük ve bağımsız olarak birleştirilebilir PR'lar hâlinde sunulur; aşağıdaki her dönüm noktası tek bir değişiklik değil, bir PR serisidir.

1. Temeller: ortam durum makinesi + sağlayıcı sözleşmesi + crabbox biçimli sağlayıcı (geliştirme düzeneği olarak statik SSH), çalışan paketi önyüklemesi + kabul el sıkışması, SSH tüneli + ana makine anahtarı sabitleme, yönetilen çalışma ağacı anlık görüntüsü + dışa giden eşitleme (git + düz modlar). Sahipsiz öğeleri temizleme + yeniden başlatmada benimseme.
2. Çalışan protokolü + çalışan döngüsü: kimliği doğrulanmış çalışan rolü, kalıcı işlemler/epoch'lar/ACK imleçleri, transkript kaydı + canlı olay sözleşmeleri, Gateway tarafından çözümlenen modellerle çıkarım proxy'si, akış denetimi. Tek sağlayıcı, yalnızca yeni oturumların insan tarafından gönderilmesi, devir yok. Hata ekleme testleri (tünel bölünmesi, Gateway'in yeniden başlatılması, çalışanın durması) çıkışı engeller.
3. Gönderme + geri çekme + ajan gönderimi: geçiş engeli, kullanıcı arayüzü hedef çubuğuna bağlanmış yerleşim durum makinesi, gelen verileri uzlaştırma + denetim noktaları, ortam başına denetim, kapasite sınırları, ajan gönderim aracı (çalışan oturumları yinelemeli gönderim yapamaz). İstem önbelleği bayt eşdeğerliği testleri.
4. 3\. kilometre taşı hata ekleme kanıtından sonra simetrik canlı devir.

Daha sonra: ortam başına kimlik bilgisi yükleme tercihi olarak çalışanlarda ACP düzenekleri; anlık görüntü/sıcak imgeyle hızlı başlatma; yelpaze biçiminde dağıtım (N kiralama, aynı istem); kutu içi işletim sistemi korumalı alanı; artifacts şeması aracılığıyla daha zengin yapıt yakalama.

## Açık sorular

- Çalışanlarda Plugin/skill kullanılabilirliği: depo kapsamında taşınan skill'ler çalışma alanıyla ücretsiz olarak eşitlenir; Gateway tarafından yapılandırılan ajan skill'leri/plugin'leri için açık bir eşitleme veya hariç tutma kararı gerekir (her iki durumda da araç/plugin manifesti kabul el sıkışmasının bir parçasıdır).
- Varsayılan denetim noktası sıklığı: çok yoğun mesajlaşmalı oturumlar için tur tabanlı mı, zaman tabanlı mı?
- Ortam profillerinin çok ajanlı yönlendirmeyle etkileşimi (ajan başına varsayılan profiller mi, yoksa yalnızca oturum başına seçim mi?).
