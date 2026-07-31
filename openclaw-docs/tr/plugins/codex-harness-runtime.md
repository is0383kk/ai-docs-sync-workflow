---
read_when:
    - Codex harness çalışma zamanı destek sözleşmesine ihtiyacınız var
    - Yerel Codex araçlarında, kancalarda, Compaction’da veya geri bildirim yüklemesinde hata ayıklıyorsunuz
    - OpenClaw ve Codex çalıştırma düzeneği turları genelinde plugin davranışını değiştiriyorsunuz
summary: Codex donanımı için çalışma zamanı sınırları, kancalar, araçlar, izinler ve tanılama
title: Codex çalıştırma altyapısı
x-i18n:
    generated_at: "2026-07-26T22:52:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d18d42683df0d827b776547f7b45f60f572cf39410d00533f53f8fdcdccb0d2
    source_path: plugins/codex-harness-runtime.md
    workflow: 16
---

Codex düzenek turları için çalışma zamanı sözleşmesi. Kurulum ve yönlendirme için
[Codex düzeneği](/tr/plugins/codex-harness) bölümüne bakın. Yapılandırma alanları için
[Codex düzeneği referansı](/tr/plugins/codex-harness-reference) bölümüne bakın.

## Genel Bakış

Codex; yerel model döngüsünü, yerel iş parçacığını sürdürmeyi, yerel araç
devamını ve yerel Compaction işlemini yönetir. OpenClaw ise kanal yönlendirmesini, oturum
dosyalarını, görünür ileti teslimini, OpenClaw dinamik araçlarını, onayları, medya
teslimini ve bu sınırın çevresindeki transkript yansısını yönetir.

İstem yönlendirmesi yalnızca sağlayıcı dizesini değil, seçilen çalışma zamanını izler.
Yerel bir Codex turu, Codex uygulama sunucusu geliştirici talimatlarını alır; açıkça belirtilmiş
bir OpenClaw uyumluluk rotası ise Codex tarzı OpenAI kimlik doğrulamasını veya aktarımını
kullansa bile normal OpenClaw sistem istemini korur.

OpenClaw, yerel Codex iş parçacıklarını Codex'in yerleşik kişiliği
devre dışı bırakılmış şekilde (`personality: "none"`) başlatır ve sürdürür; böylece çalışma alanı kişilik dosyaları
ile OpenClaw aracısı kimliği belirleyici olmaya devam eder. Bunun dışında yerel Codex,
Codex tarafından yönetilen temel/model talimatlarını ve proje belgesi yüklemeyi korur. Hafif
OpenClaw çalıştırmaları (örneğin cron) proje belgesi yüklemeyi yine engeller.

OpenClaw geliştirici talimatları, OpenClaw çalışma zamanı konularını kapsar: kaynak kanal
teslimi, OpenClaw dinamik araçları, ACP devri, bağdaştırıcı bağlamı ve
etkin aracı çalışma alanı profil dosyaları. Skills katalogları ve araçla yönlendirilen
`MEMORY.md` işaretçileri, tur kapsamlı iş birliği geliştirici
talimatları olarak yansıtılır. Bellek araçları kullanılamadığında etkin `BOOTSTRAP.md` içeriği
ve tam `MEMORY.md` bunun yerine düz tur giriş bağlamına geri döner.

Çoğu OpenClaw dinamik aracı, aranabilir `openclaw` ad alanını kullanır.
`catalogMode: "direct-only"` olarak işaretlenen araçlar `openclaw_direct` kullanır; Codex bunu,
iç içe Code Mode yürütmesine sunmak yerine `DirectModelOnly` olarak doğrudan
modele görünür tutar.

## İş parçacığı bağlamaları ve model değişiklikleri

Bir OpenClaw oturumu mevcut bir Codex iş parçacığına eklendiğinde sonraki
tur; seçili modeli, onay politikasını, korumalı alanı,
onay inceleyicisini ve hizmet katmanını uygulama sunucusuna yeniden gönderir.
`openai/gpt-5.5` modelinden `openai/gpt-5.2` modeline geçiş, iş parçacığı bağlamasını korur ancak Codex'ten
yeni seçilen modelle devam etmesini ister.

Gözetimli bağlamalar istisnadır. OpenClaw model seçici kilitli kalır
ve sürdürme işlemleri model ile sağlayıcı geçersiz kılmalarını atlar; böylece Codex, kurallı
iş parçacığının kalıcı modelini ve sağlayıcısını geri yükler. Ayrı bir yerel Codex denetimi,
kalıcı hale getirilmiş bu çifti değiştirebilir ve ilk anlık görüntü Codex'in normal
model farklılığı uyarısını oluşturabilir; dıştaki OpenClaw modeli ve geri dönüş zinciri
bunların hiçbirinin yerine geçmez.

## Gözetim ve güvenli devam

Codex gözetimi, aynı `codex` Plugin'inin isteğe bağlı bir özelliğidir. Yerel
iş parçacıklarını ayrı bir bağlantı üzerinden keşfeder ve yalnızca arşivlenmemiş
oturumları Gateway kataloğuna yansıtır. Açık `appServer` bağlantı
ayarları yoksa sıradan düzenek aracı kapsamında kalırken bu bağlantı, yönetilen kullanıcı ana dizini stdio'sunu
kullanır. Listeleme ve meta veri okumaları pasiftir: bir iş parçacığını
sürdürmez, OpenClaw'u canlı olaylarına abone etmez veya
onaylarını yanıtlamaz.

Gateway bilgisayarındaki depolanmış veya boşta olan bir oturum için **Dal olarak devam et**,
normal, modeli kilitli bir Sohbet oluşturur ve sınırlı kullanıcı ile asistan
geçmişini kaynağın kalıcı hale getirilmiş son sonlandırıcı turuna kadar yansıtır. İlk normal
Sohbet turu gerçek onay işleyicilerini kurar ve anlık görüntüyü bir model ya da sağlayıcı geçersiz kılması
olmadan sabitlemek için geçici bir yerel çatal kullanır. Codex App Server, mevcut
yerel yapılandırmasını kullanıp seçili çifti döndürür; bu model kaynağın kaydedilen son modelinden
farklıysa normal uyarısını verir.
Aynı gözetim bağlantısında OpenClaw, kurallı
`appServer` kaynaklı Codex düzenek iş parçacığını kendi çalışma dizini ve çalışma zamanı politikası altında,
ilk başlatma için döndürülen model ve sağlayıcıyı tam olarak kullanarak başlatır; sınırlı
görünür geçmişi ekler ve geçici çatalı arşivler. Kaynak hiçbir zaman
sürdürülmez. Kurallı iş parçacığı tam OpenClaw düzenek araç yüzeyine sahiptir;
kaynaktaki akıl yürütme, araç çağrıları ve araç sonuçları bu iş parçacığına kopyalanmaz.
Özel bağlantı kapsamı, bekleyen ve işlenmiş bağlama durumlarında varlığını korur; böylece
sonraki her tur, yerel kimlik doğrulaması ve sağlayıcı yapılandırmasıyla bu bağlantıda kalır.
Devre dışı bırakılmış gözetim veya bağlama/bağlantı sapması, sıradan aracı ana dizini düzeneğine
geçmek yerine kapalı biçimde başarısız olur.

Özgün CLI, VS Code, Atlas veya ChatGPT kaynağı her iki
katalog için de uygun olmaya devam eder. Kurallı dal yerel bir Codex iş parçacığıdır, ancak kaynak türü
`appServer` şeklindedir; yerel istemciler bu kaynak türünü filtreleyebileceğinden
Codex Desktop'ta görünmesi garanti edilmez.

Etkin kaynaklar yeni bir dal başlatamaz veya arşivlenemez; mevcut gözetimli
Sohbet yine de açılabilir. `notLoaded`, boşta olmayı değil etkinliğin bilinmediğini belirtir;
OpenClaw, yerel bir `idle` veya `notLoaded` satırını yalnızca başka çalıştırıcı olmadığına dair
açık onayın ve işlem düzeyinde yeni bir durum okumasının ardından arşivlemeye izin verir. Codex,
iş parçacığı değişikliklerini tek bir App Server işlemi içinde seri hale getirir ancak
işlemler arası özel bir çalıştırıcı veya onay sahibi kiralaması sağlamaz; dolayısıyla bu okuma,
başka bir işlemin iş parçacığını kullanmadığını kanıtlayamaz. OpenClaw, tam hedef ya da
Codex'in sayfalandırılmış alt öğe sorgusunun döndürdüğü arşivlenmemiş herhangi bir oluşturulmuş alt öğe
için bilinen etkin bir bağlama sahibini engeller. Numaralandırma hataları, döngüler ve
güvenlik sınırının tükenmesi kapalı biçimde başarısız olur. Yerel arşivleme, başka bir işlemdeki
yeni bir turla yine de yarışabilir; bu nedenle onay, bilinmeyen istemcileri ve
durum okumasıyla arşivleme arasındaki boşluğu kapsar. Gözetimli, modeli kilitli bir Sohbet,
yerel bağlamayı korurken silinemez.

Eşleştirilmiş Node katalogları ilk sürümde yalnızca meta veri olarak kalır. Mevcut
Node çağırma sınırı istek/yanıt biçimindedir ve gerçek bir Codex düzenek
bağlamasının gerektirdiği uzun ömürlü tur olaylarını, onay isteklerini veya akışlı çıktıyı
taşıyamaz. Bu nedenle satır boşta olsa bile uzaktan **Devam Et** ve **Arşivle**
kullanılamaz.

Operatör kurulumu ve görünür Control UI davranışı için
[Codex gözetimi](/plugins/codex-supervision) bölümüne bakın.

## Görünür yanıtlar ve Heartbeat'ler

Codex düzeneği üzerinden yapılan doğrudan/kaynak sohbet turları, dahili WebChat
yüzeyleri için varsayılan olarak otomatik son asistan teslimini kullanır ve Pi düzeneği
sözleşmesiyle eşleşir: aracı normal şekilde yanıt verir ve OpenClaw son metni
kaynak konuşmaya gönderir. Aracı `message(action="send")` çağrısı yapmadığı sürece
son asistan metnini özel tutmak için `messages.visibleReplies: "message_tool"` ayarını belirleyin.

Codex Heartbeat turları, uyandırmanın sessiz kalması mı yoksa bildirim göndermesi mi gerektiğini
aracının kaydedebilmesi için varsayılan olarak aranabilir OpenClaw araç
kataloğunda `heartbeat_respond` alır. Heartbeat girişim rehberliği, Heartbeat turuyla
sınırlı bir Codex iş birliği modu geliştirici talimatı olarak gönderilir; sıradan sohbet turları
Codex Default modunda kalır. `HEARTBEAT.md` boş olmadığında Heartbeat
talimatları, içeriğini satır içinde vermek yerine Codex'i dosyaya yönlendirir.

## Kanca sınırları

| Katman                                | Sahip                    | Amaç                                                                |
| ------------------------------------- | ------------------------ | ------------------------------------------------------------------- |
| OpenClaw Plugin kancaları             | OpenClaw                 | OpenClaw ve Codex düzenekleri genelinde ürün/Plugin uyumluluğu.      |
| Codex app-server uzantı ara yazılımı  | OpenClaw paketli Plugin'leri | OpenClaw dinamik araçlarının çevresindeki tur başına bağdaştırıcı davranışı. |
| Codex yerel kancaları                 | Codex                    | Codex yapılandırmasından düşük düzeyli Codex yaşam döngüsü ve yerel araç politikası. |

OpenClaw, Plugin davranışını yönlendirmek için proje veya genel Codex
`hooks.json` dosyalarını kullanmaz. Yerel araç ve izin köprüsü için OpenClaw;
`PreToolUse`, `PostToolUse`, `PermissionRequest` ve
`Stop` için iş parçacığı başına Codex yapılandırması ekler.

Codex app-server onayları etkinleştirildiğinde (`approvalPolicy`,
`"never"` değilse), varsayılan olarak eklenen yerel kanca yapılandırması
`PermissionRequest` öğesini atlar; böylece Codex'in app-server inceleyicisi ile OpenClaw'un
onay köprüsü, incelemenin ardından gerçek yetki yükseltmelerini işler. Uyumluluk aktarıcısını
yine de zorlamak için `nativeHookRelay.events` öğesine `permission_request` ekleyin.
`SessionStart` ve `UserPromptSubmit` gibi diğer Codex kancaları Codex düzeyinde
denetimler olarak kalır; v1 sözleşmesinde OpenClaw Plugin kancaları olarak sunulmaz.

OpenClaw dinamik araçları için OpenClaw, Codex çağrıyı istedikten sonra
aracı yürütür; dolayısıyla Plugin ve ara yazılım davranışı düzenek bağdaştırıcısında çalışır. Codex
Code Mode, genel dinamik sonuçları metin olarak alır ve iç içe
dinamik çağrıları seri hale getirir; çağıranlar JSON görünümündeki sonuçları ayrıştırmalı ve
eşzamanlı gönderim için `Promise.all` öğesine güvenmemelidir. Codex'e özgü araçlarda Codex,
kurallı araç kaydını yönetir; OpenClaw seçili olayları yansıtabilir ancak Codex bunu
app-server veya yerel kanca geri çağırmaları üzerinden sunmadıkça yerel iş parçacığını
yeniden yazamaz.

Codex app-server rapor modu `PreToolUse` olayları, Plugin onayını eşleşen
app-server onayına erteler. Yerel yük `openclaw_approval_mode:
"report"` değerini ayarlarken bir OpenClaw
`before_tool_call` kancası `requireApproval` döndürürse yerel kanca aktarıcısı, Plugin onayı
gereksinimini kaydeder ve hiçbir yerel karar döndürmez. Codex daha sonra aynı araç kullanımı
için app-server onay isteğini gönderdiğinde OpenClaw, Plugin onayı istemini açar ve
kararı Codex'e eşler. Codex `PermissionRequest` olayları,
ayrı bir onay yoludur ve bu köprü için yapılandırıldığında yine de OpenClaw onayları
üzerinden yönlendirilebilir.

Codex app-server öğe bildirimleri ayrıca, yerel
`PostToolUse` aktarıcısının zaten kapsamadığı yerel araç tamamlanmaları için eşzamansız
`after_tool_call` gözlemleri sağlar. Bunlar yalnızca telemetri/uyumluluk içindir;
yerel araç çağrısını engelleyemez, geciktiremez veya değiştiremez.

Compaction ve LLM yaşam döngüsü yansıtmaları, yerel Codex kanca komutlarından değil
Codex app-server bildirimlerinden ve OpenClaw bağdaştırıcı durumundan gelir.
`before_compaction`, `after_compaction`, `llm_input` ve `llm_output`,
Codex'in dahili istek veya Compaction yüklerinin bayt bayt yakalamaları değil,
bağdaştırıcı düzeyindeki gözlemlerdir.

Codex yerel `hook/started` ve `hook/completed` app-server bildirimleri,
yörünge ve hata ayıklama için `codex_app_server.hook` aracı olayları olarak
yansıtılır. Bunlar OpenClaw Plugin kancalarını çağırmaz.

## V1 destek sözleşmesi

Codex çalışma zamanı v1'de desteklenenler:

| Yüzey                                         | Destek                                                                            | Neden                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| --------------------------------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Codex üzerinden OpenAI model döngüsü          | Desteklenir                                                                       | Codex app-server; OpenAI turunu, yerel iş parçacığını sürdürmeyi ve yerel araç devamlılığını yönetir.                                                                                                                                                                                                                                                                                                                                                                                  |
| OpenClaw kanal yönlendirmesi ve teslimatı     | Desteklenir                                                                       | Telegram, Discord, Slack, WhatsApp, iMessage ve diğer kanallar model çalışma zamanının dışında kalır.                                                                                                                                                                                                                                                                                                                                                                                  |
| OpenClaw dinamik araçları                     | Desteklenir                                                                       | Codex bu araçları yürütmesini OpenClaw'dan ister; böylece OpenClaw yürütme yolunda kalır.                                                                                                                                                                                                                                                                                                                                                                                              |
| İstem ve bağlam Plugin'leri                   | Desteklenir                                                                       | OpenClaw, OpenClaw'a özgü istemi/bağlamı Codex turuna yansıtırken Codex tarafından yönetilen temel, model ve yapılandırılmış proje belgesi istemlerini yerel Codex hattında bırakır. OpenClaw, ajan çalışma alanı kişilik dosyalarının yetkili kalması için yerel iş parçacıklarında Codex'in yerleşik kişiliğini devre dışı bırakır. Yerel Codex geliştirici talimatları yalnızca açıkça `codex_app_server` kapsamına alınmış komut yönlendirmesini kabul eder; eski genel komut ipuçları Codex dışı istem yüzeyleri için korunur. |
| Bağlam motoru yaşam döngüsü                   | Desteklenir                                                                       | Birleştirme, içeri alma ve tur sonrası bakım, Codex turlarının çevresinde çalışır. Bağlam motorları yerel Codex Compaction işleminin yerini almaz.                                                                                                                                                                                                                                                                                                                                      |
| Dinamik araç kancaları                        | Desteklenir                                                                       | `before_tool_call`, `after_tool_call` ve araç sonucu ara katman yazılımı, OpenClaw tarafından yönetilen dinamik araçların çevresinde çalışır.                                                                                                                                                                                                                                                                                                                                          |
| Yaşam döngüsü kancaları                       | Bağdaştırıcı gözlemleri olarak desteklenir                                        | `llm_input`, `llm_output`, `agent_end`, `before_compaction` ve `after_compaction`, gerçeğe uygun Codex modu yükleriyle tetiklenir.                                                                                                                                                                                                                                                                                                                                    |
| Son yanıt revizyon kapısı                     | Yerel kanca aktarımı üzerinden desteklenir                                        | Codex `Stop`, `before_agent_finalize` öğesine aktarılır; `revise`, sonlandırmadan önce Codex'ten bir model geçişi daha ister.                                                                                                                                                                                                                                                                                                                                           |
| Yerel kabuğu, yamayı ve MCP'yi engelleme veya gözlemleme | Yerel kanca aktarımı üzerinden desteklenir                              | Codex `PreToolUse` ve `PostToolUse`, Codex app-server `0.142.0` veya daha yeni sürümlerdeki MCP yükleri dâhil olmak üzere, işlenmiş yerel araç yüzeyleri için aktarılır. Engelleme desteklenir; bağımsız değişkenlerin yeniden yazılması desteklenmez.                                                                                                                                                                                                                   |
| Yerel izin politikası                         | Codex app-server onayları ve uyumluluk amaçlı yerel kanca aktarımı üzerinden desteklenir | Codex app-server onay istekleri, Codex incelemesinden sonra OpenClaw üzerinden yönlendirilir. Codex bunu koruyucu incelemesinden önce yaydığı için `PermissionRequest` yerel kanca aktarımı, yerel onay modlarında isteğe bağlıdır.                                                                                                                                                                                                                                                           |
| App-server seyir kaydı                        | Desteklenir                                                                       | OpenClaw, app-server'a gönderdiği isteği ve app-server'dan aldığı bildirimleri kaydeder.                                                                                                                                                                                                                                                                                                                                                                                               |

Codex çalışma zamanı v1'de desteklenmeyenler:

| Yüzey                                               | V1 sınırı                                                                                                                                       | Gelecekteki yol                                                                           |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Yerel araç bağımsız değişkenlerini değiştirme       | Codex yerel araç öncesi kancaları engelleyebilir ancak OpenClaw, Codex'e özgü yerel araç bağımsız değişkenlerini yeniden yazmaz.                 | İkame araç girdisi için Codex kanca/şema desteği gerektirir.                              |
| Düzenlenebilir Codex yerel transkript geçmişi       | Standart yerel iş parçacığı geçmişini Codex yönetir. OpenClaw bir yansıyı yönetir ve gelecekteki bağlamı yansıtabilir ancak desteklenmeyen iç bileşenleri değiştirmemelidir. | Yerel iş parçacığı üzerinde müdahale gerekiyorsa açık Codex app-server API'leri ekleyin. |
| Codex yerel araç kayıtları için `tool_result_persist`  | Bu kanca, Codex yerel araç kayıtlarını değil, OpenClaw tarafından yönetilen transkript yazımlarını dönüştürür.                                  | Dönüştürülmüş kayıtlar yansıtılabilir ancak standart yeniden yazma için Codex desteği gerekir. |
| Zengin yerel Compaction meta verileri               | OpenClaw yerel Compaction talep edebilir ancak kararlı bir tutulan/çıkarılan listesi, belirteç farkı, tamamlama özeti veya özet yükü almaz.      | Daha zengin Codex Compaction olayları gerekir.                                            |
| Compaction müdahalesi                               | OpenClaw, Plugin'lerin veya bağlam motorlarının yerel Codex Compaction işlemini veto etmesine, yeniden yazmasına ya da değiştirmesine izin vermez. | Plugin'lerin yerel Compaction işlemini veto etmesi veya yeniden yazması gerekiyorsa Codex Compaction öncesi/sonrası kancaları ekleyin. |
| Bayt bazında model API isteği yakalama               | OpenClaw, app-server isteklerini ve bildirimlerini yakalayabilir ancak nihai OpenAI API isteğini Codex çekirdeği kendi içinde oluşturur.         | Codex model isteği izleme olayı veya hata ayıklama API'si gerekir.                        |

## Yerel izinler ve MCP bilgi istemleri

`PermissionRequest` için OpenClaw, yalnızca politika karar verdiğinde açık izin
veya ret kararları döndürür. Karar verilmemesi izin anlamına gelmez: Codex
bunu kanca kararı yok şeklinde değerlendirir ve kendi koruyucu veya kullanıcı
onayı yoluna geçer.

Codex app-server onay modları varsayılan olarak bu yerel kancayı içermez. Bu,
`permission_request` açıkça `nativeHookRelay.events` içine eklenmedikçe veya bir
uyumluluk çalışma zamanı bunu yüklemedikçe geçerlidir.

Bir operatör Codex yerel izin isteği için `allow-always` seçtiğinde,
OpenClaw bu tam sağlayıcı/oturum/araç girdisi/cwd parmak izini sınırlı bir
oturum aralığı boyunca hatırlar. Hatırlanan karar kasıtlı olarak yalnızca tam
eşleşme için geçerlidir: değişen bir komut, bağımsız değişkenler, araç yükü veya
cwd yeni bir onay oluşturur.

Codex MCP araç onayı bilgi istemleri, Codex `_meta.codex_approval_kind` değerini
`"mcp_tool_call"` olarak işaretlediğinde OpenClaw'ın Plugin onay akışı
üzerinden yönlendirilir. Codex `request_user_input`, kaynak oturum için
sağlayıcıdan bağımsız bir Gateway sorusu kaydeder. Control UI, Gateway soru
kartını görüntüler ve kanal destekliyorsa gizli olmayan tek bir seçim için
türlü kanal düğmeleri kullanılır. Düğme dokunuşları, Control UI yanıtları ve
sıradaki kuyruğa alınmış düz metin yanıtı; OpenClaw app-server yanıtını
döndürmeden önce aynı Gateway kaydını çözümler. Codex otomatik çözümleme ve
deneme iptalleri, bekleme süresini sınırlar ve kaydı iptal eder. Gizli sorular
tamamen uyarılı metin yanıtı yolunda kalır. Diğer MCP bilgi istemi istekleri
güvenli biçimde reddedilir.

Bu istemleri taşıyan genel Plugin onay akışı için
[Plugin izin istekleri](/tr/plugins/plugin-permission-requests) bölümüne bakın.

## Kuyruk yönlendirmesi

Etkin çalıştırma kuyruğu yönlendirmesi, Codex app-server `turn/steer` üzerine eşlenir. Varsayılan
`messages.queue.mode: "steer"` ile OpenClaw, yönlendirme modundaki sohbet
mesajlarını yapılandırılmış sessizlik aralığı boyunca toplu hâle getirir ve varış sırasına göre tek bir `turn/steer`
isteği olarak gönderir.

Codex inceleme ve manuel Compaction dönüşleri, aynı dönüşte yönlendirmeyi reddedebilir. Bu
durumda OpenClaw, istemi başlatmadan önce etkin çalıştırmanın tamamlanmasını
bekler. Mesajların varsayılan olarak yönlendirilmek yerine kuyruğa alınması gerektiğinde `/queue followup` veya `/queue collect` kullanın. Bkz. [Yönlendirme kuyruğu](/tr/concepts/queue-steering).

## Codex geri bildirimini yükleme

Yerel Codex çalıştırma altyapısındaki bir oturum için `/diagnostics [note]` onaylandığında
OpenClaw, listelenen her iş parçacığının günlükleri ve mevcut olduğunda oluşturulan Codex
alt iş parçacıkları dâhil olmak üzere ilgili Codex iş parçacıkları için Codex app-server
`feedback/upload` çağrısını da yapar.

Yükleme, Codex'in normal geri bildirim yolu üzerinden OpenAI sunucularına gider. Bu
app-server'da Codex geri bildirimi devre dışıysa komut,
app-server hatasını döndürür. Tamamlanan tanılama yanıtı; gönderilen iş parçacıklarına ait kanalları,
OpenClaw oturum kimliklerini, Codex iş parçacığı kimliklerini ve yerel `codex resume <thread-id>`
komutlarını listeler.

Onayı reddeder veya yok sayarsanız OpenClaw, bu Codex kimliklerini
yazdırmaz ve Codex geri bildirimi göndermez. Yükleme, yerel
Gateway tanılama dışa aktarımının yerini almaz. Onay, gizlilik, yerel paket ve grup sohbeti
davranışı için bkz. [Tanılama dışa aktarımı](/tr/gateway/diagnostics).

`/codex diagnostics [note]` seçeneğini yalnızca tam Gateway tanılama
paketi olmadan, o anda bağlı olan iş parçacığının Codex geri bildirimini yüklemek
istediğinizde kullanın.

## Compaction ve transkript yansısı

Seçilen model Codex çalıştırma altyapısını kullandığında, yerel iş parçacığı Compaction işlemi
Codex app-server'a aittir. OpenClaw; Codex dönüşleri için ön kontrol Compaction işlemi
çalıştırmaz, Codex Compaction işlemini bağlam motoru Compaction işlemiyle değiştirmez veya
yerel Compaction başlatılamadığında OpenClaw ya da herkese açık OpenAI özetlemesine
geri dönmez. OpenClaw; kanal geçmişi, arama,
`/new`, `/reset` ve gelecekteki model ya da çalıştırma altyapısı geçişleri için bir transkript yansısı tutar.

`/compact` veya bir Plugin tarafından istenen manuel
Compaction işlemi gibi açık Compaction istekleri, `thread/compact/start` ile yerel Codex Compaction işlemini başlatır.
OpenClaw, Codex eşleşen `contextCompaction` tamamlanma öğesini yayımlayana kadar isteği
ve paylaşılan istemci kirasını açık tutar, ardından Compaction
dönüşünü tamamlandı olarak bildirir. Bu sonlandırıcı dönüş, yapılandırılmış Compaction
zaman aşımını geçerse OpenClaw yerel dönüş kesintisi ister. Kira ve iş parçacığı başına
Compaction engeli, Codex sonlandırıcı durumu bildirene veya
kesinti RPC'sini onaylayana kadar tutulmaya devam eder. Codex, kesinti tolerans
süresi içinde onay vermezse OpenClaw engeli serbest bırakmadan önce bağlantıyı kullanımdan kaldırır. Uzak
bağlantılar ayrıca eşleşen iş parçacığı bağlamasını ayırır; böylece sonraki çalışmalar,
onaylanmamış bir uzak dönüşle çakışamaz. Kullanımdan kaldırılmış bir bağlantıdaki diğer dönüşler başarısız olur
ve yeni bir istemcide yeniden denenebilir. İstemcinin kapanması, isteğin iptal edilmesi veya
başarısız bir Compaction dönüşü, başarısız bir işlem döndürür. Bağlam baskısına bağlı otomatik
Compaction, Codex'in görevidir; OpenClaw yerel Compaction işlemini yalnızca manuel
olarak istenen tetikleyiciler için başlatır.

Bir bağlam motoru Codex iş parçacığı önyükleme projeksiyonu istediğinde OpenClaw,
araç çağrısı adlarını ve kimliklerini, girdi şekillerini ve sansürlenmiş araç sonucu
içeriğini yeni Codex iş parçacığına yansıtır. Ham araç çağrısı bağımsız değişkeni
değerlerini bu projeksiyona kopyalamaz.

Yansı; kullanıcı istemini, nihai asistan metnini ve app-server bunları yayımladığında hafif
Codex akıl yürütme veya plan kayıtlarını içerir. OpenClaw,
yerel Compaction başlangıcını ve sonlandırıcı durumunu kaydeder ancak
insanlar tarafından okunabilir bir Compaction özeti ya da Codex'in Compaction sonrasında hangi
girdileri tuttuğunu gösteren denetlenebilir bir liste sunmaz.

Codex, kurallı yerel iş parçacığının sahibi olduğundan `tool_result_persist`,
Codex'e özgü araç sonucu kayıtlarını yeniden yazmaz. Yalnızca OpenClaw,
OpenClaw'a ait bir oturum transkripti araç sonucu yazdığında uygulanır.

## Medya ve teslimat

OpenClaw, medya teslimatının ve medya sağlayıcısı seçiminin sahibi olmaya devam eder. Görsel,
video, müzik, PDF, TTS ve medya anlama; `agents.defaults.mediaModels.image`,
`agents.defaults.mediaModels.video`, `pdfModel` ve `tts` gibi eşleşen sağlayıcı/model
ayarlarını kullanır.

Metin, görseller, video, müzik, TTS, onaylar ve mesajlaşma aracı çıktıları
normal OpenClaw teslimat yolu üzerinden devam eder; medya üretimi,
eski çalışma zamanını gerektirmez. Codex bir `savedPath` içeren yerel bir görsel üretme öğesi yayımladığında
OpenClaw, Codex dönüşünde asistan metni olmasa bile tam olarak bu dosyayı normal yanıt medyası
yolu üzerinden iletir.

## İlgili

- [Codex çalıştırma altyapısı](/tr/plugins/codex-harness)
- [Codex çalıştırma altyapısı başvurusu](/tr/plugins/codex-harness-reference)
- [Codex gözetimi](/plugins/codex-supervision)
- [Yerel Codex Plugin'leri](/tr/plugins/codex-native-plugins)
- [Plugin kancaları](/tr/plugins/hooks)
- [Ajan çalıştırma altyapısı Plugin'leri](/tr/plugins/sdk-agent-harness)
- [Tanılama dışa aktarımı](/tr/gateway/diagnostics)
- [Yörünge dışa aktarımı](/tr/tools/trajectory)
