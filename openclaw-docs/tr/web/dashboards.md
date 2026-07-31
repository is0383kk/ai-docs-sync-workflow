---
read_when:
    - Control UI'daki oturum panolarını kullanma veya açıklama
    - Bir panoda ajanların neler yapabileceğine ve nelerin operatör izni gerektirdiğine karar verme
summary: 'Oturum panoları: aracının oluşturduğu widget''lar, panolar, sekmeler ve sabitlenmiş sohbet'
title: Oturum Panoları
x-i18n:
    generated_at: "2026-07-26T23:06:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3babbc859e261aa959740ea778b44fdc1a07bce8ce7628cbabcfbc5fa207a0ce
    source_path: web/dashboards.md
    workflow: 16
---

Control UI'daki her iş parçacığının iki yüzü vardır: bildiğiniz konuşma ve
agent'ınızın sizin için oluşturduğu canlı widget'lardan oluşan bir ızgara olan
**dashboard**. Hiç widget'ı olmayan bir iş parçacığı yalnızca sohbettir. Bir widget
sabitlendiği anda başlıkta **Sohbet | Dashboard** geçiş düğmesi görünür ve dashboard,
sohbetiniz yanına sabitlenmiş hâlde ana yüzey olur.

Kurulacak veya ayrı bir uygulamada yapılandırılacak hiçbir şey yoktur: dashboard'lar
iş parçacığına ait, agent ile birlikte saklanan temel bir özelliktir ve
`/new` ile `/reset` işlemlerinden sonra da korunur (konuşma bağlamı temizlenir; pano kalır).

## İsteyerek bir dashboard oluşturun

Agent'ınıza ne görmek istediğinizi söyleyin:

> revenue-graph adında bir widget oluştur: aylık geliri gösteren etkileşimli bir
> çubuk grafik. Görünümler arasında geçiş yapan "Çubuklar" ve "Eğilim" düğmeleri ekle.
> Bunu dashboard'uma sabitle.

Agent, herhangi bir yere eklemeden önce inceleyebilmeniz için widget'ı ilk olarak
sohbet içinde satır içi olarak işler. Ardından:

- **Siz sabitlersiniz**: satır içi bir widget'ın üzerine gelin ve **Dashboard'a sabitle** seçeneğini belirleyin.
- **Ya da agent doğrudan sabitler** ve daha sonra istediğinizde adıyla
  günceller — widget'ların adları sabittir; dolayısıyla "revenue-graph'i
  Haziran rakamlarıyla güncelle" komutu, pano yerinde kalırken içeriği bulunduğu
  yerde değiştirir.

Widget'lar, sıkı bir korumalı alanda çalışan, kendi kendine yeten küçük uygulamalardır
(HTML/JS/SVG). Bir widget içindeki düğmeler ve görünüm geçişleri anında çalışır —
grafik görünümünü değiştirmek için agent'a hiçbir zaman gerek yoktur.

## Pano

- **Akışkan ızgara.** Widget'ları tutamaçlarından sürükleyin; her şey otomatik
  olarak yeniden düzenlenip sıkıştırılır. Tutamaçtan yeniden boyutlandırın veya
  widget menüsünden bir boyut ön ayarı (küçük, orta, büyük, ekstra büyük) seçin.
  Pikselleri hiç kimse yerleştirmez — ne siz ne de agent.
- **Sekmeler.** Bir panonun birkaç sayfası olabilir; örneğin bir genel bakış
  sekmesi ve tek bir büyük widget içeren odaklanmış bir sekme. Her sekme, sohbet
  panelinin konumunu ayrı olarak hatırlar.
- **Sabitlenmiş sohbet.** Dashboard yüzünde konuşmanız sola, sağa veya alta
  sabitlenir, kenar çubuğu gibi yeniden boyutlandırılır ve tamamen gizlenebilir —
  geri getirdiğinizde agent sizi yine duyar.
- **Agent eşitliği.** Yapabildiğiniz her şeyi agent da
  `dashboard` aracıyla yapabilir: widget ekleme, güncelleme, taşıma, yeniden
  boyutlandırma ve kaldırma; sekmeleri yönetme, görünür sekmeyi değiştirme ve
  sohbet panelini taşıma veya gizleme. "Sohbeti sola al ve finans sekmesini
  göster" deyin ve gerçekleşmesini izleyin.

## Widget'ların yapmasına izin verilenler

Yalnızca görüntüleme yapan bir widget onay gerektirmez — tıpkı satır içi sohbet
widget'ları gibi anında görünür ve ağ erişimi tamamen devre dışıdır.

**Erişim** isteyen widget'lar bunu bildirmelidir; siz de widget başına bir kez,
tek dokunuşla izin verirsiniz:

- **Ağ** (`net`): bildirilen HTTPS kaynaklarına doğrudan korumalı alandan
  erişir — örneğin kendisini bir API'den yenileyen bir hava durumu kartı.
- **Gateway verileri** (`data`): oturumlar, kullanım veya cron durumu gibi
  salt okunur akışlar Gateway tarafından çözümlenir — widget hiçbir zaman token'ınızı
  tutmaz.
- **Otomasyon** (`actions`): belirli bir cron görevini tetikler; böylece bir
  düğme, ana konuşmanızı uyandırmadan gerçek bir görevi (daha küçük bir model
  kullanabilir) çalıştırabilir.
- **İstem** (`prompt`): onaylanmamış widget'ların gerektirdiği her tıklamada
  onay olmadan iş parçacığınıza mesaj gönderir.

Etkin Plugin'ler, bu yetenek listelerine kendi adlandırılmış salt okunur akışlarını ve eylemlerini ekleyebilir; Plugin'in devre dışı bırakılması bu entegrasyonları kaldırır.

İzinler, incelediğiniz tam widget baytlarına ve revizyona bağlıdır. Agent widget'ı
değiştirip onayladığınızdan _daha fazlasını_ isterse yeniden beklemeye alınır;
aynı izinler kapsamında içeriğin yenilenmesi izni korur. Agent'ın bilmesi gereken
widget etkileşimleri (tıkladığınız filtreler, değiştirdiğiniz görünümler) oturum
bildirimleri olarak sessizce ona ulaşır — kesintiye uğramadan bilgi sahibi olur.

## Panodaki MCP uygulamaları

Gateway'inizde MCP sunucuları yapılandırılmışsa, sohbette görünen etkileşimli MCP
uygulamaları herhangi bir widget gibi sabitlenebilir. Sabitlenmiş uygulamalar panoda
yeni oturumlarla yeniden canlanır; varsayılan olarak yalnızca görüntüleme yaparlar.
Widget'a bildirdiği sunucu araçları için izin verilmesi, diğer her şeyle aynı tek
dokunuşlu ve revizyona bağlı onayla onu tamamen etkileşimli hâle getirir.

## Bilinmesi gerekenler

- Panosu olan bir iş parçacığını sıfırlamak için onay istenir ve pano
  korunur.
- Bir iş parçacığını silmek, panosunu da siler.
- Panolar Gateway'inizde (sahip olan agent'ın veritabanında) bulunur ve
  bağlandığınız her cihazda görünür.
- Güvenlik modeli, depolama ayrıntıları ve tasarım gerekçeleri,
  belgelenmiş korumalı alan ödünleşimleriyle birlikte
  [Dashboard Mimarisi](/web/dashboard-architecture) bölümünde açıklanmaktadır.
