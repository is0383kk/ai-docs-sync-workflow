---
read_when:
    - Oturum panosu (boards) özelliğini uygulama veya inceleme
    - Widget barındırmayı, widget köprüsünü veya pano depolamasını değiştirme
summary: 'Oturum panoları: mimari ve uygulama planı (teknik tasarım, GA öncesi)'
title: Pano Mimarisi
x-i18n:
    generated_at: "2026-07-27T00:22:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a7c5da94ec19add55c6b7b530f0c17509a027e97fb301469ce48f520b325c169
    source_path: web/dashboard-architecture.md
    workflow: 16
---

<Note>
Oturum panosu özelliği için uygulama öncesinde ve uygulama sırasında yazılmış
teknik tasarım belgesi. Geliştirme için temel referans kaynağıdır. Özellik
yayınlandığında, `/web/dashboard` kullanıcıya yönelik sayfa hâline gelir ve bu
sayfa mimari referans olarak kalır.
</Note>

## Vizyon

Günümüzde bir agent ile çalışmak, bir metin akışından ibarettir. Pano bunu bir
çalışma tezgâhına dönüştürür: agent canlı, etkileşimli widget'lar oluşturur;
kullanıcı bunları kalıcı bir yüzeye sabitler; sohbet yana yerleşir (veya gizlenir)
ve ana içerik pano olur. Oturumdan hiç ayrılmadan "agent ile konuşmaktan",
"agent'ın sizin için oluşturduğu bir kontrol panelini işletmeye" geçersiniz.

İlkeler:

- **Pano yeni bir nesne değil, oturumun bir yüzüdür.** Her oturumun (iş parçacığının)
  iki yüzü vardır: döküm ve pano. Sabitlenmiş widget'ı olmayan bir oturum,
  sade sohbettir. Bir widget sabitlendiğinde pano var olur. Panolar oturumun
  kimliğini, agent sahipliğini, adlandırmasını, sabitlenmesini ve yaşam
  döngüsünü devralır. `dashboard_create`, pano kayıt defteri veya ayrı bir ACL
  modeli yoktur.
- **Agent ile eşdeğerlik.** Kullanıcının panoda yapabildiği her şeyi agent da
  araçlarla yapabilir: widget ekleme/güncelleme/kaldırma, düzenleme, sekmeleri
  yönetme, görünür sekmeyi değiştirme, sohbeti yerleştirme veya gizleme.
- **Gömülü değil, yerel.** Pano, Control UI kabuğundaki Lit bileşenlerinden
  oluşur (uygulamanın geri kalanıyla aynı tasarım sistemi). Yalnızca widget
  _içeriği_ iframe'lerde korumalı alana alınır. URL çubuğu veya tarayıcı arayüzü
  yoktur.
- **Küçük agent yüzeyi.** Widget'lara kararlı adlarıyla erişilir ve oldukları
  yerde güncellenirler. Yerleşim, kendini otomatik sıkıştıran akışkan bir
  ızgaradır; agent boyutları ve sabitleme noktalarını belirtir, piksel veya
  koordinatları asla belirtmez.
- **Güven yerine yetenekler.** Widget kodu, sıkı bir korumalı alanda çalışan,
  agent tarafından yazılmış rastgele HTML/JS kodudur. Erişim (gateway verileri,
  eylemler, ağ) yalnızca beyan edilmiş ve operatör tarafından verilmiş bir
  yetenek manifesti üzerinden mümkündür.

## Kavramlar

| Kavram              | Tanım                                                                                                                                                            |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Oturum (iş parçacığı) | Kararlı `sessionKey` ile anahtarlanan mevcut gateway oturumu. Bir agent'a aittir.                                                                          |
| Pano                | Bir oturumun widget yüzü. Yalnızca oturumda widget'lar/sekmeler varsa mevcuttur. `/new`/`/reset` sonrasında kalır (`sessionKey` öğesine bağlıdır, döküme değil). |
| Sekme               | Panonun bir sunum sayfası: hangi widget'ların bulunduğu, düzenleri ve sohbet yerleşiminin durumu (`left`/`right`/`bottom`/`hidden`). Panolar örtük tek bir sekmeyle başlar. |
| Widget              | Oturuma ait, adlandırılmış ve korumalı alana alınmış HTML/JS programı. `sessionKey` + `name` olarak adreslenir. Ada göre olduğu yerde güncellenir. |
| Yetenek manifesti   | Widget başına erişim beyanı: `data` (okuma bağlamaları), `actions` (izin verilen fiiller), `prompt` (oturuma gönderme), `net` (izin verilen kaynaklar). |
| Sabitleme (widget)  | Bir döküm widget'ını oturumun panosuna taşıma (kullanıcı özelliği veya agent araç argümanı). Sabitlemeyi kaldırmak, onu panodan kaldırır.                              |
| Sabitleme (oturum)  | Oturumların kenar çubuğuna mevcut sabitlenmesi. Panosu olan sabitlenmiş bir oturum, pano yüzünde açılır.                                                           |

## Kullanıcı deneyimi akışları

- **Panoya geçiş:** agent herhangi bir sohbette `show_widget` çağrısı yapar → widget,
  bugün olduğu gibi döküm içinde satır içi oluşturulur → üzerine gelindiğinde
  **Panoya sabitle** gösterilir → widget oturumun panosunda görünür. Agent aynı
  işlemi yapmak için `pin: true` iletebilir.
- **Pano görünümü:** panosu olan bir oturumda yüz değiştirme anahtarı bulunur
  (Sohbet / Pano). Pano görünümü = sekme şeridi (yalnızca >1 sekme olduğunda) +
  akışkan ızgara + yerleştirilmiş sohbet bölmesi. Sohbet yerleşimi, tam olarak
  kenar çubuğu gibi yeniden boyutlandırılabilir, taşınabilir (sol/sağ/alt) ve
  daraltılabilir. Her sekmenin yerleşim durumu hatırlanır.
- **Sürükleme:** kullanıcı widget'ları sürükler; ızgara kendini otomatik olarak
  sıkıştırır (widget'lar yukarı çıkar, komşular yeniden akar). Tutamaçla yeniden
  boyutlandırma, boyut adımlarına oturur. Hiç kimse için piksel tabanlı
  yerleştirme yoktur.
- **Sıfırlama uyarısı:** panosu olan bir oturumda `/new` /
  `/reset`, web kullanıcı arayüzünde onay ister ("bağlam sıfırlanır,
  pano kalır") ve panoyu korur.
- **Kenar çubuğu:** sabitlenmiş oturumların panosu varsa pano yüzleri
  oluşturulur. Ana Sayfa oturumunun panosu, varsayılan "agent panosu"dur.
- **Etkileşimler** (üç katman, aşağıya bakın): sessiz durum olayları, görünür
  istem gönderimleri ve otomasyon tetikleyicileri.

## Etkileşim katmanları

1. **Durum olayları (varsayılan).** Modelin bilmesi ancak yanıtlamaması gereken
   widget kullanıcı arayüzü etkileşimleri. `bridge.emitState({...})`, yapılandırılmış
   bir oturum bildirimi ekler (grup etkinliği bildirimleriyle aynı mekanizma).
   Agent sırası başlatılmaz; model, bir sonraki çalıştırmada birikmiş bildirimleri
   görür.
2. **İstemler (açıkça konuşma).** `bridge.sendPrompt(text)` — kullanıcı
   etkinleştirmesi gerektirir; oturuma görünür bir kullanıcı mesajı gönderir
   (yerleştirilmiş sohbet bunu gösterir). Hız sınırı uygulanır; widget
   `prompt` yetenek iznine sahip değilse her gönderim kullanıcı
   tarafından onaylanır.
3. **Otomasyon.** `bridge.runAction(name, args)` — manifestte beyan edilmiş
   bir eylemi çalıştırır. İlk fiil kümesi: `cron.trigger` (mevcut bir Cron
   işini hemen çalıştırma) ve `binding.refresh`. Cron işleri zaten görünür,
   yalıtılmış çalıştırma oturumlarında çalışır ve daha ucuz bir model
   kullanabilir: "küçük model widget'ı çalıştırır" yolu budur. Hiçbir yerde
   gizli oturum yoktur.

## Widget modeli ve barındırma

Widget HTML/JS kodu agent tarafından yazılır (genellikle `show_widget`
aracılığıyla), standart belge kabuğuna (CSP meta verisi, boyut bildiricisi,
köprü önyüklemesi) sarılır ve `<iframe sandbox="allow-scripts">` içinde oluşturulur
(asla `allow-same-origin` içinde değil).

- **Satır içi (döküm) widget'lar** mevcut tuval-belge işlem hattını korur:
  durum dizini altına yazılır, gateway tarafından sunulur, kapsam başına
  budanır ve onay gerektirmez (yapıları gereği yeteneksizdirler — istem
  gönderimleri kullanıcı tarafından onaylanır).
- **Pano widget'ları** oturum durumudur: baytlar, sahibi olan agent'ın SQLite
  veritabanında (`board_widgets`) bulunur ve veritabanını okuyan bir çekirdek
  gateway rotası (`/__openclaw__/board/<agentId>/<sessionKey>/<name>/`) tarafından sunulur.
  Bir döküm widget'ını sabitlemek baytları kopyalar. Sınırlar: widget başına
  256 KB, pano başına 48 widget.
- **Olduğu yerde güncelleme:** aynı `name` ile bir widget'ın yeniden
  yayımlanması baytları değiştirir, `revision` değerini artırır,
  `board.changed` yayınını yapar ve canlı görünümler yalnızca ilgili iframe'i
  yeniden yükler.
- **Bayt sabitleme:** verilen yetenekler widget baytlarının sha256 değerine
  bağlanır. Baytların değiştirilmesi, yeni revizyon verilen manifestin bir alt
  kümesini beyan ediyorsa `data`/`net`/`actions`
  izinlerini korur; kapsamı genişletilmiş bir manifest operatörden yeniden onay
  ister.

### Widget'lar içerik barındırır; MCP uygulamaları bir içerik türüdür

**Widget, OpenClaw'ın temel öğesidir**: adlandırılmış, sabitlenmiş,
boyutlandırılmış, oturuma ait ve izin kaydı bulunan pano hücresi. İçinde
oluşturulan şey bir içerik türüdür:

- `html` — `show_widget` aracılığıyla agent tarafından
  yazılır; baytlar pano depolamasındadır.
- `mcp-app` — yapılandırılmış bir sunucudan gelen üçüncü taraf
  MCP uygulama görünümü (`ui://` kaynağı), widget hücresinde
  barındırılır.

MCP uygulamaları widget modelini tanımlamaz; widget'lar onları barındırma
yeteneği kazanmıştır. Kimlik, yerleştirme, sabitleme, izinler ve yazara yönelik
API OpenClaw'a ait kalır; böylece `show_widget` kodu bugün olduğu kadar kısa
kalır ve MCP Apps belirtiminin varlığını hiçbir zaman bilmesi gerekmez.

Altta paylaşılan altyapı (sadeleştirmenin gerçekleştiği yer):

- **Tek korumalı alan barındırıcısı.** `html` widget'ları,
  ikinci bir özel iframe barındırıcısı yerine MCP uygulamalarıyla birlikte
  sunulan aynı güçlendirilmiş işlem hattı üzerinden oluşturulur (ayrılmış
  korumalı alan kaynağında çift iframe, widget başına beyan edilen ve kapalı
  kalacak şekilde çözümlenen CSP). Proxy, HTML'yi değer olarak alır; dolayısıyla
  yerel içerik doğal kullanım durumudur.
- **Tek yetkilendirme modeli.** Türü ne olursa olsun bir widget'ın erişimi,
  izin verilmiş bir izin listesidir: `html` widget'ları için
  barındırıcı araçları; `mcp-app` widget'ları için sunucunun uygulamaya
  görünür araçları (mevcut `allowedAppToolNames` mekanizması üzerinden, oluşturma
  çalıştırması başına değil widget başına kalıcı hâle getirilmiş).
- **`html` widget'ları için barındırıcı araçları** (widget köprüsü
  üzerinden sunulur ve izne göre denetlenir):
  - `openclaw.prompt.send` — katman 2; görünür düzenleyici üzerinden
    yönlendirilir, izin verilmedikçe kullanıcı tarafından onaylanır
  - `openclaw.state.emit` — katman 1 oturum bildirimleri (birleştirilmiş,
    boyutu sınırlandırılmış)
  - `openclaw.data.read` — parametreli salt okunur bağlamalar (mevcut
    izin verilen okuma RPC kümesi), gateway tarafında çözümlenir
  - `openclaw.cron.trigger` — katman 3 otomasyon
- **`net` = CSP.** Ağ erişimi, daha önce sunulmuş widget başına
  CSP beyanını (`connect-src` kaynakları) kullanır — kendini güncelleyen
  hava durumu widget'ı API'sini doğrudan korumalı alandan getirir, gateway
  sürece dahil olmaz.
- **İzinler.** Hiçbir şey beyan etmeyen bir widget hemen oluşturulur (korumalı
  alanda, `default-src 'none'`, istem gönderimleri ayrı ayrı onaylanır) — bugünkü
  satır içi sohbet widget'larıyla aynı güven düzeyi. Beyan edilen
  araçlar/kaynaklar, widget'ı panoda `pending` durumuna geçirir: bir yer
  tutucu kart bunları insanların okuyabileceği biçimde listeler ve tek
  dokunuşla **İzin Ver**/**Reddet** seçeneklerini sunar. İzinler widget adı
  başınadır; `html` widget'ları için bayta sabitlenmiştir (sha256) ve
  değiştirilen baytlar, yalnızca beyanın kapsamı daraldıysa izni korur.
- **Yazma uyumluluk katmanı.** Belge sarmalayıcısı kararlı yazar API'si olarak
  `window.openclaw.prompt`, `window.openclaw.state`, `window.openclaw.data` ve
  `window.openclaw.cron` öğelerini ekler. Pano çağrıları, görünüm biletine bağlı tek
  bir istek kanalını paylaşır; boyut bildirimi ve tema belirteçleri ayrı
  barındırıcı bildirimleri olarak kalır.

### Plugin yetenek beyanları

Etkin Plugin'ler, `openclaw.plugin.json` içindeki `dashboard.dataBindings` ve
`dashboard.actionVerbs` aracılığıyla widget barındırıcısını genişletebilir.
Plugin'e yerel kimlikler, `workboard.cards.list` ve `workboard.dispatch` gibi Plugin
kimliği önekli izin adlarına dönüşür; farklı bir Plugin/yerel kimlik ayrımının
aynı kalıcı izni devralamaması için Plugin kimliği bölümündeki
`%` ve `.` kaçışlanır. OpenClaw, Plugin kaydı
sırasında her bağlamanın aynı Plugin tarafından `operator.read` ile
kaydedilmiş bir RPC'yi ve her eylemin `operator.write` ile kaydedilmiş bir
RPC'yi hedeflediğini doğrular; geçersiz beyanlar Plugin yüklemesini başarısız
kılar. Doğrulanmış kayıt defteri yalnızca Plugin yaşam döngüsü değişikliklerinde
yeniden oluşturulurken widget izinleri widget başına ve bayt ile revizyona bağlı
kalmaya devam eder.

### Modellenmiş artık risk: WebRTC veri kanalları

Korumalı alan CSP'si önerilen `webrtc 'block'` yönergesini yayımlar ancak
[Chromium'un mevcut CSP yönerge kümesi](https://chromium.googlesource.com/chromium/src/+/main/services/network/public/mojom/content_security_policy.mojom#95)
bunu uygulamaz. Bu nedenle betik çalıştırabilen widget'lar, mevcut Chromium'da
dışarı veri aktarmak için WebRTC veri kanallarını kullanabilir. Aynı artık risk,
`main` üzerindeki satır içi sohbet widget'ları ve MCP Apps
barındırıcısı için zaten sunulmaktadır.

**Kabul edilen ödünleşim:** OpenClaw, betik çalıştırılabilen widget'ları bu
artık riske göre kısıtlamaz. Widget içeriği, hassas OpenClaw verilerine yalnızca
operatör tarafından verilmiş, baytları sabitlenmiş bir `data:read` yeteneği üzerinden erişir ve sandbox
Permissions Policy kamera ile mikrofon erişimini engeller. DOM API koruması
güvenlik sınırı değil, çaba ölçüsünde uygulanan derinlemesine savunmadır ve
sonraki sağlamlaştırma çalışmalarına aittir.

### Transkript görünümü: tek widget kartı

Satır içi görünüm, widget temel öğesi üzerinde birleştirilir. Bir araç sonucu kullanıcı arayüzü —
`show_widget` çıktısı veya uygulama kaynağı içeren bir MCP araç sonucu — taşıdığında sistem
**geçici, otomatik adlandırılan bir widget** (oturum kapsamlı, budanan) oluşturur ve
transkript, içerik türüne göre yönlendirme yapan tek bir widget kartı işler.
MCP uygulamasının otomatik gösterimi, spesifikasyonun beklediği şekilde aynen kalır (sıfır ek model çalışması);
altta yalnızca bir widget _olarak bulunur_. Bu, sohbet işlemedeki paralel `mcpApp`
özel durumlarını (yüzey kısıtlaması, ayrı tekilleştirme) kaldırır, her
satır içi kullanıcı arayüzüne aynı sabitleme olanağını verir ve widget kayıt defterini birincil
yeniden açma yolu hâline getirir (transkript taramasıyla yeniden oluşturma, hiç sabitlenmemiş
geçmiş için yedek olarak kalır). Salt okunur, biletli bağımsız ana makine,
kalıcı yeniden açma yüzeyi olarak panolarla örtüşür — T6'da değerlendirilecek bir
birleştirme adayıdır, varsayım değildir.

Birleştirme: v1, ızgara bitişikliğidir (tek sekmede bir uygulama widget'ının yanında
ajan çerçevesi widget'ı). v2, **ana makine tarafından yönetilen uygulama yuvaları** ekler — ajan widget HTML'i bir
yuva bölgesi bildirir ve ana makine, gerçek uygulama görünümünü kardeş sandbox olarak birleştirir.
Uygulama hiçbir zaman ajanın iframe'i içinde işlenmez: iç içe yerleştirme, köprü
kimliğini bozar ve izin verilen uygulama kullanıcı arayüzünün üzerinin kapatılmasına/tıklama kaçırmasına olanak tanır; bu nedenle yuva,
gömme değil bir yerleşim sözleşmesidir.

### Sunucudan sağlanan widget'lar (sabitlenmiş MCP uygulamaları)

Birleşik ana makineyle, üçüncü taraf bir MCP uygulamasını sabitlemek yalnızca içeriği
depolanmak yerine sunucudan getirilen bir widget'tır: `board_widgets`, HTML baytları yerine
tanımlayıcıyı (`serverName`, `toolName`, `uiResourceUri`, kaynak
`toolCallId` + `sessionKey`) tutar ve pano, sohbet turunun 10 dakikalık TTL süresini aşınca
görünüm kirasını yeniden oluşturur (eskidiğinde `ui://` kaynağını yeniden
getirir). Sohbet içindeki satır içi MCP uygulama görünümleri, ajan widget'larıyla aynı **Panoya sabitle**
olanağını edinir. Yeniden açılan görünümler bugün tasarım gereği salt okunurdur;
etkileşimli kalması gereken sabitlenmiş uygulamalar, sunucunun
uygulamaya görünür araçları üzerinde kalıcı bir izin alır (sabitleme sırasında operatöre açık izin listesi gösterilir) ve bu izin,
oluşturma çalışmasından ayrıdır. İzin verilmemiş sabitlemeler salt okunur kalır — görüntüleme
panoları için yine de kullanışlıdır. v1, kaynak oturumun panosuna sabitlenir; oturumlar arası sabitleme
bir kira aracısı gerektirir ve bekler. Açık PR #109807 (`ui/message`
oluşturucu yönlendirmesi, tema/boyut yayılımı) ile koordine edin.

### WorkBoard entegrasyonu

WorkBoard entegrasyon programı, kartları ve panoları plugin mülkiyetinde tutarken gönderilen kartları mevcut `sessionKey` ve `runId` üzerinden yeniden oturum panolarına bağlar, WorkBoard akışlarını ve gönderimi plugin tarafından bildirilen bağlamalar ve eylemler aracılığıyla sunar ve WorkBoard'a özgü bir widget türü eklemek yerine bu sonuçları mevcut `html` ve `mcp-app` widget türleriyle birleştirir.

## Yerleşim: akışkan ızgara

12 sütun, sabit satır yüksekliği, **otomatik sıkıştırılan** (yukarı çekim, sürüklemede
yana itme — gridstack semantiği, yerel olarak uygulanır; ızgara matematiği saf ve
DOM'dan bağımsız kalır). Sekme başına widget yerleşim durumu: `{ name, w (1-12), h (rows) }` artı
sıra. Ajan sözlüğü:

- `size`: `sm` (3×3) · `md` (6×4) · `lg` (8×6) · `xl` (12×8) · `full`
  (tek widget'lı sekme)
- `after: <widgetName>` isteğe bağlı sıralama çapası; belirtilmezse = sona ekle
- Kullanıcı serbestçe sürükleyip yeniden boyutlandırır; aynı sıra+boyut modeli gidiş dönüşte korunur.

## Veri modeli (ajan başına DB)

`agents/<agentId>/agent/openclaw-agent.sqlite` içinde yeni tablolar
(**ajan DB şema sürümünün artırılmasını gerektirir — bu değişiklik eklenmeden
önce operatör onayı gereklidir**):

```sql
CREATE TABLE board_tabs (
  session_key TEXT NOT NULL,
  tab_id      TEXT NOT NULL,           -- slug
  title       TEXT NOT NULL,
  position    INTEGER NOT NULL,
  chat_dock   TEXT NOT NULL DEFAULT 'right',  -- left|right|bottom|hidden
  created_by  TEXT NOT NULL,           -- 'user' | 'agent'
  PRIMARY KEY (session_key, tab_id)
) STRICT;

CREATE TABLE board_widgets (
  session_key  TEXT NOT NULL,
  name         TEXT NOT NULL,          -- stable widget name
  tab_id       TEXT NOT NULL,
  title        TEXT,
  html         BLOB NOT NULL,          -- wrapped document source
  sha256       TEXT NOT NULL,
  revision     INTEGER NOT NULL,
  size_w       INTEGER NOT NULL,
  size_h       INTEGER NOT NULL,
  position     INTEGER NOT NULL,       -- order within tab (auto-compact input)
  manifest     TEXT NOT NULL DEFAULT '{}',  -- capability manifest JSON
  grant_state  TEXT NOT NULL DEFAULT 'none', -- none|pending|granted|rejected
  granted_sha  TEXT,                   -- byte-frozen grant
  created_by   TEXT NOT NULL,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL,
  PRIMARY KEY (session_key, name)
) STRICT;
```

Pano varlığı = `sessionKey` için herhangi bir satırın bulunması. Bir oturumun silinmesi, onun
pano satırlarını siler. `/new`/`/reset` bunlara dokunmaz.

## Protokol yüzeyi

RPC'ler (çekirdek yöntem tablosu, `gateway-protocol` içindeki typebox şemaları):

- `board.get { sessionKey }` → sekmeler + widget meta verileri (bayt yok) — `operator.read`
- `board.update { sessionKey, ops[] }` — sekme CRUD/yeniden sıralama, widget taşıma/yeniden boyutlandırma/
  kaldırma/sabitlemeyi kaldırma, yuva durumu, sekmeye odaklanma — `operator.write`
- `board.widget.put { sessionKey, name, html, manifest, placement }` —
  `operator.write` (ajan aracı yolu ve sabitleme yolu)
- `board.widget.grant { sessionKey, name, decision }` — `operator.approvals`
- `board.event { ticket, payload }` — bilete bağlı katman-1 durum olayı alımı;
  eski güvenilir ana makine `{ sessionKey, widget, payload }` biçimi korunur —
  `operator.write`
- `board.prompt.authorize { ticket }` — görünür bir istem gönderiminin
  hâlâ tıklama başına onay gerektirip gerektirmediğini döndürür — `operator.read`
- `board.data.read { ticket, bindingId, params? }` — Gateway tarafında izin listesine alınmış
  çekirdek veya etkin plugin okuma bağlaması çözümlemesi — `operator.read`
- `board.action { ticket, action, ... }` — mevcut cron hemen çalıştır yolu veya etkin bir plugin'in doğrulanmış eylem
  fiili üzerinden tam izinli otomasyon gönderimi — `operator.write`

Olaylar (`EVENT_SCOPE_GUARDS` içinde, okuma kapsamı):

- `board.changed { sessionKey, revision, widget? }` — kalıcı durum değişti;
  kullanıcı arayüzü yeniden getirir (ve `widget` mevcut olduğunda bir iframe'i yeniden yükler).
- `board.command { sessionKey, command }` — geçici kullanıcı arayüzü yönlendirmesi (ajan görünür
  sekmeyi değiştirir, sohbet yuvasını açıp kapatır) — `ui.command` kalıbı.

Widget baytları, soket üzerinden değil kimliği doğrulanmış HTTP yüzeyi üzerinden sunulur.

## Ajan araçları

Toplam üç araç (çekirdek, her zaman kayıtlı; işleme bugün olduğu gibi
`inline-widgets` istemci yeteneğine göre kısıtlanır):

- `show_widget { title, widget_code, name?, pin?, size?, tab?, after?,
capabilities? }` — ada göre oluştur/güncelle; `pin` bunu panoya yerleştirir.
  `name`/`pin` olmadan tamamen bugünkü gibi davranır (satır içi, geçici).
- `dashboard { action, ... }` — pano yönetimi fiilleri: `read`, `tab_create`,
  `tab_update`, `tab_delete`, `tabs_reorder`, `widget_move`, `widget_remove`,
  `unpin`, `focus_tab`, `set_chat_dock`.
- Mevcut `cron` araçları otomasyon katmanını kapsar; yeni araç gerekmez.

Araç açıklamaları boyut/çapa sözlüğünü ve katman modelini öğretir. Ajan,
kullanıcı katman-1 olaylarından oturum bildirimleri aracılığıyla haberdar edilir, ör.
`[dashboard] user clicked "Refresh" on widget weather (tab main)`.

## Bunun yerine geçtiği bileşenler

- **`extensions/workspaces` silinir.** Deneyseldir, `enabledByDefault:
false`, hiçbir zaman kararlı bir sürümde yer almamıştır (ilk kez 2026.7.2 beta sürümlerinde ortaya çıktı).
  Geçiş yoktur; bir doctor kuralı varsa eski `<stateDir>/workspaces/` öğesini kaldırır.
  Alınan fikirler: saf ızgara matematiği, köprü güvenlik modeli (port önyüklemesi,
  bağlama kısıtlaması, hız sınırları), baytları sabitlenmiş onay.
- **Widget barındırma `extensions/canvas` içinden çekirdeğe taşınır.** Canvas belge
  deposu, belge sarmalayıcısı, HTTP sunumu ve `show_widget` aracı çekirdeğe
  (`src/canvas/`) taşınır; plugin, node-canvas denetim aracını (`canvas`) ve
  A2UI'ı tutar. `pluginSurfaceUrls["canvas"]` duyurusu ve
  `/__openclaw__/canvas` yolları yayımlanmış yerel istemci sözleşmeleridir ve
  kararlı kalır. Discord oturumları, Discord'a ait `show_widget` çeşidini korur.

## Hedef dışı kapsam (bu program)

- Çok kullanıcılı pano paylaşımı/ACL'ler (gelecekte; oturum paylaşımıyla gelecek).
- Yerel macOS/iOS pano işleme (Control UI'ı gömdükleri her yerde bunu edinirler;
  satır içi widget yolu değişmez).
- Yerleşik veri widget'ları (oturumlar/kullanım/cron kartları) — yetenek köprüsü ve
  ajan tarafından yazılan widget'lar v1'i kapsar; yerleşik tür kayıt defteri daha sonra gelebilir.

## Uygulama planı

Bağımsız çalışma ağaçları, Codex ile oluşturulur, sırayla incelenip eklenir. Ekle, ardından düzelt.

| #   | Dal                                  | Kapsam                                                                                                                                                                             | Bağımlılıklar                     |
| --- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| T1  | `claude/dashboard-remove-workspaces` | Workspaces plugin'i + kullanıcı arayüzü + belgeler + i18n anahtarlarını silme; doctor temizleme kuralı                                                                             | —                                 |
| T2  | `claude/dashboard-canvas-core`       | Widget barındırmayı + `show_widget` öğesini çekirdeğe yükseltme; canvas plugin'i node aracını tutar; sıfır davranış değişikliği                                                     | —                                 |
| T3  | `claude/dashboard-domain`            | Ajan DB tabloları (şema artışı), `board.*` RPC'leri + olaylar, `dashboard` aracı, `show_widget` sabitleme/ad/manifest bağımsız değişkenleri, katman-1 bildirimleri, sıfırlamada panoyu koruma | T2                                |
| T4  | `claude/dashboard-ui`                | Pano yüzü + sekme şeridi + akışkan otomatik sıkıştırılan ızgara + sohbet yuvası (sol/sağ/alt/gizli) + transkript sabitleme olanağı + kenar çubuğu pano yüzü + sıfırlama onayı       | T3 (geliştirme fikstürleriyle önce sahte) |
| T5  | `claude/dashboard-capabilities`      | İzin deposu/kullanıcı arayüzü + bayt sabitleme; `html` widget'larını paylaşılan sandbox ana makinesine taşıma; ana makine araçları (`openclaw.prompt.send/state.emit/data.read/cron.trigger`); `net` CSP; yazarlık uyumluluk katmanı | T3, T4                            |
| T7  | `claude/dashboard-mcp-apps`          | `mcp-app` içerik türü: satır içi uygulama görünümlerinde sabitleme olanağı, tanımlayıcı depolama, kirayı yeniden oluşturma/yenileme, kalıcı sunucu aracı izinleri (yayımlanmış MCP Apps ana makinesini yeniden kullanır) | T3, T4                            |
| T6  | iyileştirme                          | Geçici bir Gateway üzerinde canlı E2E (gerçek anahtarlar), ekran görüntüleri, düzeltmeler, kullanıcı odaklı `/web/dashboard` yeniden yazımı, varsayılan olarak etkinleştirme incelemesi | tümü                              |

Depo kurallarına göre doğrulama: yerel olarak odaklı vitest, tam kontroller
Crabbox/Testbox üzerinde, her eklemeden önce `$autoreview`, T6 için canlı kanıt.
