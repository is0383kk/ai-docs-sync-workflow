---
read_when:
    - Bir agentın web sohbetinde, yerel bir uygulamada veya Discord'da etkileşimli bir sonuç oluşturmasını istiyorsunuz
    - Widget düğmelerinin sohbete takip istemleri göndermesini istiyorsunuz
    - Widget'ları paylaşılan tasarım belirteçleriyle temalandırmak istiyorsunuz
    - show_widget girdisi, güvenlik veya saklama sözleşmesi gereklidir
sidebarTitle: Show widget
summary: Desteklenen sohbet yüzeylerinde bağımsız HTML araç takımlarını gösterin
title: Widget’ı göster
x-i18n:
    generated_at: "2026-07-27T00:20:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 903adff1fadeb9d224d3e2d839c86082b5244e1e319255c8d3f6619344b749a3
    source_path: tools/show-widget.md
    workflow: 16
---

`show_widget`, kullanıcının mevcut yüzeyinde bağımsız bir HTML widget'ı gösteren temel bir araçtır. OpenClaw bunu Control UI'da ve iOS, Android, macOS ve Linux Hızlı Sohbet dökümlerinde satır içinde işler; Linux panosu tarayıcı Control UI'ını kullanır. [Activities](/channels/discord-activities) etkinleştirilmiş bir Discord oturumunda Discord plugin'i, bunu bir Activity olarak başlatan **Widget'ı aç** düğmesi gönderir.

## Widget'lar nasıl çalışır?

Aracı `show_widget` çağırdığında OpenClaw çekirdeği, `widget_code` öğesini minimal bir HTML belgesine sarar, Canvas belgesi olarak depolar ve bir önizleme tanıtıcısı döndürür. Control UI bu tanıtıcıyı korumalı bir iframe içinde işlerken iOS, Android, macOS ve Linux Hızlı Sohbet yalıtılmış web görünümleri kullanır. Tam sohbet istemcileri, geçmiş yeniden yüklendikten sonra widget'ı geri yükler; Hızlı Sohbet ise widget'ı etkin yanıtı boyunca korur.

Control UI oturumlarında bir Canvas widget'ı oturum panosuna da sabitlenebilir. Araç çağrısında `pin: true` ayarını yapın veya mevcut bir döküm widget'ında **Panoya sabitle** seçeneğini kullanın. Sabitlenmiş HTML, MCP Apps tarafından kullanılan aynı ayrılmış kaynaklı, çift iframe'li korumalı alan ana makinesinin arkasında çalışır; tarayıcı, güvenilmeyen çerçevenin içinde bir widget veri bağlamasını asla çözümlemez.

Tarayıcıya gömme amacıyla sarmalayıcı belge, widget kodunun çevresine dört küçük ana makine köprüsü enjekte eder:

- Bir boyut bildiricisi, işlenen içeriğin yüksekliğini gömen sohbete gönderir; sohbet bu yüksekliği sınırlar ve iframe'i buna uyarlar (160 ile 1200 piksel).
- Bir ana makine köprüsü, eski `sendPrompt(text)` yardımcısının yanı sıra yapılandırılmış `openclaw.prompt`, `openclaw.state`, `openclaw.data` ve `openclaw.cron` API'lerini tanımlar. Satır içi sohbet istemleri özel ileti kanallarını korur; pano API'leri görsel bilete bağlı bir istek kanalı kullanır. Bkz. [Etkileşimli widget'lar](#interactive-widgets) ve [Pano yetenekleri](#dashboard-capabilities).
- Bir tema köprüsü, Control UI'ın mevcut tasarım belirteçlerini dinler ve bunları yükleme sırasında ve her tema değişikliğinde yeniden CSS değişkenleri olarak uygular.
- Bir anlık görüntü köprüsü, gömen sohbet dışa aktarma istediğinde mevcut widget belgesini PNG olarak işler.

Diğer her şey çerçevenin içinde kalır: belge, katı bir İçerik Güvenliği Politikası ile opak bir kaynakta çalışır; dolayısıyla widget betikleri Control UI'a, Gateway'e veya ağa erişemez.

Çekirdek uygulaması yalnızca kaynak Gateway istemcisi `inline-widgets` yeteneğini bildirdiğinde kullanılabilir. Control UI ve desteklenen yerel uygulamalar bu yeteneği otomatik olarak bildirir. Linux Hızlı Sohbet, platform WebView'ı bu sabitlemeyi bağlayamadığı için özel bir TLS yaprak sertifikası sabitlemesi gerektiren Gateway bağlantılarında yalnızca metin modunda kalır. Discord uygulaması yalnızca Activities yapılandırılmış Discord oturumlarında kullanılabilir. Diğer kanal çalıştırmaları `show_widget` öğesini almaz.

Yetenek aktarımı; gömülü, Codex uygulama sunucusu ve CLI destekli model arka uçlarını kapsar. Yetkiyle kimliği doğrulanmış MCP çağıranları ve doğrudan HTTP araç çağırma istemcileri, istemci yeteneklerini bildirmedikleri için kapalı kalır.

## Tasarım sistemi

Her Canvas widget'ı sınıfsız bir temel stil sayfası ve küçük bir belirteç kümesi içerir:

| Belirteç                                                                              | Amaç                                  |
| ------------------------------------------------------------------------------------- | ------------------------------------- |
| `--surface`                                                                           | Sayfa düzeyi yüzey rengi              |
| `--card`                                                                              | Kart, düğme ve kod arka planı         |
| `--elevated`                                                                          | Yükseltilmiş form denetimi arka planı |
| `--text`                                                                              | Varsayılan gövde ve denetim metni     |
| `--text-strong`                                                                       | Başlıklar ve öne çıkan değerler       |
| `--muted`                                                                             | İkincil metin ve hafif kenarlıklar    |
| `--border`                                                                            | Standart ayırıcılar ve kart kenarlıkları |
| `--border-strong`                                                                     | Güçlü denetim kenarlıkları            |
| `--accent`                                                                            | Bağlantılar ve odak halkaları         |
| `--accent-fill`                                                                       | Birincil eylem dolgusu                |
| `--accent-fg`                                                                         | Birincil eylem üzerindeki metin       |
| `--ok`                                                                                | Başarı durumu                         |
| `--warn`                                                                              | Uyarı durumu                          |
| `--danger`                                                                            | Hata veya yıkıcı durum                |
| `--info`                                                                              | Bilgilendirme durumu                  |
| `--radius`                                                                            | Paylaşılan denetim ve kart köşe yarıçapı |
| `--font-body`                                                                         | Ana makine gövde yazı tipi yığını     |
| `--font-mono`                                                                         | Ana makine eş aralıklı yazı tipi yığını |
| `--accent-subtle`, `--ok-subtle`, `--warn-subtle`, `--danger-subtle`, `--info-subtle` | Türetilmiş yarı saydam durum arka planları |

Yalın başlıklar, paragraflar, bağlantılar, düğmeler, girişler, seçimler, metin alanları, tablolar ve kod blokları temel stilleri alır. Yardımcı sınıflar yaygın kalıplar sağlar:

- Kenarlıklı içerik yüzeyi için `.card`
- Kompakt durum etiketleri için `.badge`; ayrıca `.ok`, `.warn`, `.danger` veya `.info`
- Öne çıkan sayısal değer için `.metric`
- İkincil metin için `.muted`
- Satır kaydıran yatay düzen için `.row`
- Birincil eylem için `button.primary`

Control UI, bir widget yüklendiğinde ve tema her değiştiğinde etkin tema değerlerini içeren bir `openclaw:widget-theme` iletisi gönderir. Bu nedenle widget'lar, yeniden yüklenmeden Claw, Knot, Dash ve özel temalar dâhil her tema ailesini izler. Yerel uygulamalar ve doğrudan açmalar dâhil Control UI dışında widget'lar, `prefers-color-scheme` tarafından seçilen gömülü açık veya koyu paleti kullanır.

Widget'ları üç kurala göre hazırlayın:

1. Her renk ve arka plan için tasarım değişkenlerini kullanın. Renk değerlerini doğrudan kodlamayın.
2. Widget'ın ana makine yüzeyine ait görünmesi için sayfa arka planını şeffaf tutun.
3. En fazla bir birincil eylem için `--accent-fill` ayırın.

**Dışa aktarma:** Web sohbetinde, işlenen widget'ı panoya kopyalamak veya PNG olarak indirmek için widget kartı menüsünü açın. Anlık görüntü köprüsü bulunmayan eski widget belgeleri, alternatif olarak HTML dosyası indirmeye geçer.

## Aracı kullanma

Her iki uygulama da aynı zorunlu alanları kullanır:

<ParamField path="title" type="string" required>
  Satır içi önizlemeyle ve barındırılan belge başlığında gösterilen kısa başlık.
</ParamField>

<ParamField path="widget_code" type="string" required>
  Bağımsız HTML veya SVG. Satır içi widget istemcilerinde, kırpma sonrasında `<svg` ile başlayan girdi SVG modunda işlenir; azami uzunluk 262.144 karakterdir. Discord, 48 KiB'a kadar eksiksiz bir HTML belgesini veya gövde parçasını kabul eder.
</ParamField>

Discord ayrıca Activity başlatma düğmesi için isteğe bağlı `button_label` metnini kabul eder. Canvas şeması, yalnızca Discord'a özgü bu alanı kasıtlı olarak içermez.

Temel Canvas aracı şu isteğe bağlı pano yerleştirme alanlarını kabul eder:

- `pin`: widget'ı oturum panosuna da yerleştirir.
- `name`: kararlı widget adı; varsayılan olarak `title` öğesinin slug biçimini kullanır.
- `tab`: hedef sekme slug'ı.
- `size`: `sm`, `md`, `lg`, `xl` veya `full` değerlerinden biri.
- `after`: widget'ın arkasına yerleştirileceği kardeş widget adı.
- `capabilities`: sabitlenmiş bir widget'ın istediği erişim. `netOrigins` tam HTTPS kaynaklarını; `tools` ise `prompt`, izin listesindeki bir okuma bağlamasını veya tam bir `cron.trigger:<jobId>` eylemini içerir.

Çekirdek sonuç bir Canvas önizleme tanıtıcısı içerir; bu sayede Control UI ve desteklenen yerel uygulamalar widget'ı doğrudan araç çağrısından işler ve geçmiş yeniden yüklendikten sonra geri yükler. Sabitlenmiş sonuçlar pano widget'ının adını da korur; böylece Control UI, döküm yeniden yüklendikten sonra yinelenen bir sabitleme seçeneği sunmaz. Discord, depolanan widget ve gönderilen ileti tanımlayıcılarını döndürür.

`discord_widget`, bir sürüm boyunca kullanımdan kaldırılmış bir diğer ad olarak kayıtlı kalır. Yeni aracı çağrıları `show_widget` kullanmalıdır.

## Etkileşimli widget'lar

Control UI'da widget betikleri konuşmayı yönlendirebilir. Sarmalayıcı belge, genel bir `sendPrompt(text)` işlevi tanımlar; bu işlev çağrıldığında `text`, kullanıcı iletiyi yazıp göndermiş gibi sohbete iletilir. Seçiciler, testler veya ayrıntıya inme panoları gibi etkileşimli akışlar oluşturmak için bunu düğmelere ya da diğer denetimlere bağlayın. Yerel uygulamalar etkileşimli widget kodunu işler ancak bu sohbet istemi köprüsünü sunmaz.

```html
<button onclick="sendPrompt('Başarısız testleri ayrıntılı olarak göster')">Başarısız testler</button>
```

Her istem, çerçeve sınırının her iki tarafında da doğrulanır:

- `sendPrompt`, widget içinde [geçici kullanıcı etkinleştirmesi](https://developer.mozilla.org/en-US/docs/Web/Security/User_activation) gerektirir: yalnızca kullanıcı widget içinde tıkladıktan veya bir tuşa bastıktan sonraki birkaç saniye boyunca çalışır; bu nedenle bunu düğmelere ve diğer tıklama hedeflerine bağlayın — yükleme sırasında otomatik olarak çağrılması hiçbir şey yapmaz. Köprü, gönderen uç noktayı yalnızca kendisine özel tutar ve kullanıcı etkinleştirmesini sunmayan tarayıcılarda kapalı kalır; böylece widget kodu denetimi atlayamaz.
- İstem yetkisi yalnızca özgün widget belgesine aittir. Güvenilir köprü, widget kodu çalışmadan veya çerçevede gezinmeden önce kanal uç noktasını sohbete sunar; sohbet yalnızca bu ilk sunumu benimser ve gezinme sırasında kanal belgeyle birlikte sonlanır. Harici olarak izin verilen gömme URL'leri asla benimsenmez.
- Widget çerçevesi sohbet dökümünde görünür olmalı ve odağı elinde tutmalıdır; bu, kullanıcının gerçekten bu widget ile etkileşim kurduğunu ana makinenin gözlemlediği ek bir sinyaldir.
- Metin, kırpıldıktan sonra boş olmamalı ve en fazla 4.000 karakter içermelidir.
- `/` ile başlayan istemler reddedilir; böylece widget kodu `/approve` veya `/stop` gibi sohbet komutlarını tetikleyemez.
- Her widget belgesi, kayan bir dakikalık süre içinde en fazla 10 istem gönderebilir; fazla istemler sessizce bırakılır.

Kabul edilen istemler dökümde normal kullanıcı iletileri olarak görünür ve widget'ın sahibi olan oturumda normal bir aracı turu başlatır. Widget'a geri bildirim kanalı yoktur: bırakılan bir istem sessizce başarısız olur ve widget aracının yanıtını okuyamaz.

## Pano yetenekleri

Sabitlenmiş widget'lar, operatör bekleyen kartta gösterilen bildirimi inceledikten sonra bilete bağlı bir ana makine API'sini kullanabilir:

- `openclaw.prompt.send(text)` geçici kullanıcı etkinleştirmesi gerektirir ve görünür bir oluşturucu mesajı gönderir. `prompt` araç izninin bildirilmesi ve alınması, her tıklamada ek onayı atlar; doğrulama, odak kontrolleri ve hız sınırları uygulanmaya devam eder.
- `openclaw.state.emit(payload)` bir oturum bildirimi ekler. Yükler 8 KiB ile sınırlandırılır ve beş saniye içinde istemciden gönderilen özdeş iletiler birleştirilir.
- `openclaw.data.read(bindingId, params?)` yalnızca Gateway'de çözümlenir. İzin verilebilir bağlamalar `sessions.list`, `usage.status`, `usage.cost`, `cron.list`, `cron.status`, `agents.list` ve `health` şeklindedir.
- `openclaw.cron.trigger(jobId)`, mevcut bir işi yalnızca tam olarak `cron.trigger:<jobId>` yeteneği verilmişse hemen çalıştırır.

Ağ erişimi, ana makine araçlarından ayrıdır. Tam HTTPS kaynaklarını `capabilities.netOrigins` içine yerleştirin; onaydan sonra yalnızca bu kaynaklar aracın `connect-src` öğesine eklenir. Joker karakterler, kimlik bilgileri, yollar, sorgu dizeleri ve bildirilmemiş kaynaklar engellenmeye devam eder. Değişmez bir bağlantı noktasına yalnızca bildirilen kaynağın bir parçası olduğunda izin verilir.

## Güvenlik ve depolama

Araç belgeleri kısıtlayıcı İçerik Güvenliği İlkeleri kullanır. Satır içi stile ve betiğe izin verilirken harici kaynak yüklemeleri engellenmeye devam eder. Satır içi transkript araçları ağa erişemez. Sabitlenmiş bir pano aracı yalnızca aracının bildirdiği ve operatörün izin verdiği tam HTTPS kaynaklarına erişebilir.

Genel gömme modu `trusted` olsa bile Control UI iframe'i her zaman `allow-same-origin` öğesini hariç tutar; böylece araç betikleri üst uygulamanın kaynağını okuyamaz. Yerel istemciler yalıtılmış, kalıcı olmayan web görünümleri kullanır ve barındırılan araçtan başka bir yere gezinmeyi engeller. Çekirdek belge ana makinesi ayrıca araçları bir `Content-Security-Policy: sandbox allow-scripts` yanıt başlığıyla sunar; böylece doğrudan oluşturma işleminde bile araç, uygulama kaynağı yerine opak bir kaynakta çalışır. Yalnızca bu yalıtılmış çerçevede çalıştırmaya razı olduğunuz araç kodunu oluşturun.

Iframe ayrıca [`gateway.controlUi.embedSandbox`](/tr/web/control-ui#hosted-embeds) yönergelerine uyar. Varsayılan `scripts` katmanı, kaynak yalıtımını korurken etkileşimli araçları destekler.

Kabul edilen WebRTC veri kanalı çıkışına ilişkin kalan risk, [Pano Mimarisi](/web/dashboard-architecture#modeled-residual-webrtc-data-channels) bölümünde belgelenmiştir.

Canvas, oturum başına (veya oturum kullanılamadığında aracı başına) en fazla 32 aracı saklar. Başka bir araç oluşturulduğunda, ilgili kapsamdaki en eski belge kaldırılır.

## İlgili

- [Control UI barındırılan gömmeleri](/tr/web/control-ui#hosted-embeds)
- [Discord Etkinlikleri](/channels/discord-activities)
- [Canvas Node denetimleri](/tr/plugins/reference/canvas)
- [Gateway protokolü istemci yetenekleri](/tr/gateway/protocol#client-capabilities)
