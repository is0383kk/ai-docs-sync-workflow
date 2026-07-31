---
read_when:
    - Yüklemeleri kötüye kullanım veya politika ihlalleri açısından inceleme
    - Moderasyon belgeleri veya inceleyici çalışma kılavuzları yazma
    - Bir skill'in gizlenip gizlenmemesine veya bir kullanıcının yasaklanıp yasaklanmamasına karar verme
sidebarTitle: Acceptable Usage
summary: 'Pazar yeri politikası: ClawHub''ın nelere izin verdiği ve neleri barındırmayacağı.'
title: Kabul Edilebilir Kullanım
x-i18n:
    generated_at: "2026-07-26T22:39:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ace357e7a3e9f4d242f113ad791b254e94ae8a841dd9a864a77c5bac15713132
    source_path: clawhub/acceptable-usage.md
    workflow: 16
---

# Kabul Edilebilir Kullanım

ClawHub, OpenClaw için becerileri, Plugin’leri, paketleri ve pazar yeri meta verilerini barındırır.
İçeriğin veya yayımlama davranışının ClawHub’a ait olup olmadığına karar vermek için
bu sayfayı kullanın.

Bu kurallar; bir listelemenin ne yaptığı, kullanıcılardan ne çalıştırmalarını istediği, kendisini
nasıl tanıttığı ve yayıncıların ClawHub’ın keşif, yükleme ve güven yüzeylerini nasıl kullandığı
için geçerlidir. Moderasyon durumları ve hesap itibarı için
[Moderasyon ve Hesap Güvenliği](/clawhub/moderation) bölümüne bakın. Telif hakkı veya diğer hak
talepleri için [İçerik Hakları Talepleri](/clawhub/content-rights) bölümüne bakın.

## İzin verilen içerik

ClawHub; yararlı, anlaşılır ve iyi niyetle yayımlanmış içerikleri memnuniyetle karşılar.

| Kategori                                         | Şu durumda izin verilir                                                                                                                      |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| Geliştirici üretkenliği                           | Listeleme, kullanıcıların yazılım geliştirmesine, test etmesine, taşımasına, hata ayıklamasına, belgelemesine veya işletmesine yardımcı olur.                                               |
| Kullanıcı arayüzü, veri ve otomasyon iş akışları               | Kapsam nettir, gerekli kimlik bilgileri açıkça belirtilmiştir ve riskli eylemler inceleme, deneme çalıştırması, önizleme veya onay yolları içerir. |
| Savunma amaçlı güvenlik, moderasyon ve kötüye kullanım incelemesi | Araç, yetkili inceleme için tasarlanmıştır, kanıtları korur ve insan onayı sınırlarını açık tutar.                          |
| Kişisel veya ekip iş akışları                       | İş akışı; rızaya dayalı hesaplar, şeffaf kurulum ve açık izinler kullanır.                                            |
| Bakımı yapılan kataloglar                              | Her listeleme farklı, yararlı, doğru şekilde açıklanmış ve makul ölçüde bakımı yapılmış durumdadır.                                                |

Bağlam önemlidir. Aynı konu, dar kapsamlı savunma amaçlı veya
rızaya dayalı bir ortamda kabul edilebilirken kötüye kullanım iş akışı olarak paketlendiğinde kabul edilemez olabilir.

## İzin verilmeyen içerik

ClawHub; temel amacı kötüye kullanım, aldatma, güvenli olmayan
yürütme veya hak ihlali olan içerikleri barındırmaz.

| Kategori                                                    | İzin verilmez                                                                                                                                                                                                                                                                                                   |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Yetkisiz erişim veya güvenlik önlemlerini aşma                      | Kimlik doğrulamayı aşma, hesap ele geçirme, hız sınırını kötüye kullanma, canlı çağrı veya ajan ele geçirme, yeniden kullanılabilir oturum hırsızlığı ya da onaylanmamış kullanıcılar için eşleştirme akışlarını otomatik olarak onaylama.                                                                                                                                                   |
| Platformu kötüye kullanma ve yasaktan kaçınma                              | Yasaklamalardan sonra gizli hesaplar, hesap ısıtma veya yetiştirme, sahte etkileşim, çok hesaplı otomasyon, toplu gönderim, spam botları ya da tespit edilmekten kaçınmak üzere geliştirilmiş otomasyon.                                                                                                                                          |
| Dolandırıcılık, sahtekârlık ve aldatıcı finansal iş akışları             | Sahte sertifikalar veya faturalar, aldatıcı ödeme akışları, dolandırıcılık amaçlı iletişim, sahte sosyal kanıt, dolandırıcılık için sentetik kimlik iş akışları ya da açık insan onayı olmadan harcama veya ücretlendirme yapan araçlar.                                                                                                                    |
| Gizliliği ihlal eden veri zenginleştirme veya gözetim                 | Spam için iletişim bilgisi kazıma, kişisel bilgileri ifşa etme, takipçilik, istenmeyen iletişimle birlikte potansiyel müşteri verisi çıkarma, gizli izleme, rıza dışı biyometrik eşleştirme ya da sızdırılmış verilerin veya veri ihlali dökümlerinin kullanılması.                                                                                                                  |
| Rıza dışı taklit veya kimlik manipülasyonu       | Yüz değiştirme, dijital ikizler, klonlanmış fenomenler, sahte kişilikler ya da taklit etmek veya yanıltmak için kullanılan diğer araçlar.                                                                                                                                                                                                 |
| Açık cinsel içerik veya güvenliği devre dışı bırakılmış yetişkin içerik üretimi | NSFW görsel, video veya içerik üretimi; üçüncü taraf API’leri çevreleyen yetişkin içerik sarmalayıcıları ya da temel amacı açık cinsel içerik olan listelemeler.                                                                                                                                                       |
| Gizli, güvenli olmayan veya yanıltıcı yürütme gereksinimleri        | Gizlenmiş yükleme komutları; açıkça incelenebilir olmadan indirilen içeriğin `sh` veya `bash` ile çalıştırılması gibi kabuğa yönlendirmeli yükleyiciler; beyan edilmemiş gizli bilgi veya özel anahtar gereksinimleri; açıkça incelenebilir olmayan uzaktan `npx @latest` yürütme ya da listelemenin çalışmak için gerçekte neye ihtiyaç duyduğunu gizleyen meta veriler. |
| Telif hakkını veya diğer hakları ihlal eden materyaller           | Başkasının becerisini, Plugin’ini, belgelerini, marka varlıklarını veya özel mülkiyetli kodunu izinsiz yeniden yayımlama; lisans koşullarını ihlal etme ya da özgün yazarın veya yayıncının kimliğine bürünme.                                                                                                                            |

## İzin verilmeyen pazar yeri davranışları

ClawHub, yayıncıların pazar yerini nasıl kullandığını da inceler. ClawHub’ı
keşfi, metrikleri, güven sinyallerini, moderasyon sistemlerini veya kullanıcı
ilgisini manipüle etmek için kullanmayın.

İzin verilmeyen pazar yeri davranışları şunları içerir:

- gerçek kullanıcı değeri taşıdığı görülmeyen, düşük çabalı, yinelenen, yer tutucu veya
  makine tarafından oluşturulmuş çok sayıda listelemeyi toplu olarak yayımlamak
- arama veya kategori yüzeylerini neredeyse aynı becerilerle veya Plugin’lerle doldurmak
- çok az kullanımı, bakımı, kaynak açıklığı veya anlamlı farklılığı olan ya da bunların hiçbiri bulunmayan
  yüzlerce listeleme yayımlamak
- otomasyon, kendi kendine yükleme döngüleri, sahte hesaplar, koordineli
  etkinlik, ücretli etkileşim veya diğer organik olmayan davranışlar aracılığıyla yüklemeleri, indirmeleri, yıldızları veya diğer etkileşim
  metriklerini yapay olarak artırmak
- moderasyondan, yasaklardan, yayıncı sınırlarından veya
  pazar yeri incelemesinden kaçınmak için hesaplar oluşturmak veya hesapları dönüşümlü kullanmak
- mülkiyet, kaynak, yetenekler, güvenlik duruşu,
  yükleme gereksinimleri ya da başka bir proje veya yayıncıyla bağlantı hakkında kullanıcıları yanıltmak
- temelindeki sorunu düzeltmeden daha önce gizlenmiş, kaldırılmış veya engellenmiş
  içeriği tekrar tekrar yüklemek

Yüksek hacimli yayımlama otomatik olarak kötüye kullanım sayılmaz. Listelemeler
anlamlı ölçüde farklı, doğru şekilde açıklanmış, bakımı yapılmış ve gerçek kullanıcılar
tarafından kullanılıyor olduğunda büyük kataloglar kabul edilebilir. Hacim; yüzeysel,
yinelenen, yanıltıcı, bakımsız veya yapay olarak öne çıkarılan listelemelerle
birleştiğinde büyük kataloglar bir güvenlik ve güven sorunu hâline gelir.

## İçerik hakları

ClawHub’daki içeriğin telif hakkınızı veya diğer haklarınızı ihlal ettiğini düşünüyorsanız
[İçerik Hakları Talepleri](/clawhub/content-rights) sayfasını kullanın. Listeleme aynı zamanda güvenli değil,
kötü amaçlı veya yanıltıcı olmadığı sürece telif hakkı ya da hak talepleri için normal pazar yeri
raporlarını kullanmayın.

## İnceleme ve yaptırım

ClawHub, güvenli olmayan içerikleri veya kötüye kullanım niteliğindeki yayımlama davranışlarını belirlemek için otomatik kontrolleri, istatistiksel kötüye kullanım sinyallerini, kullanıcı raporlarını ve
personel incelemesini kullanabilir. Bir sinyal
tek başına kötüye kullanımı kanıtlamaz; ClawHub’ın nelerin incelenmesi gerektiğine karar vermesine yardımcı olur.

Şunları yapabiliriz:

- ihlal niteliğindeki listelemeleri gizlemek, beklemeye almak, kaldırmak, geçici olarak silmek veya kaynak türü için desteklendiğinde
  kalıcı olarak silmek
- güvenli olmayan sürümler için indirmeleri veya yüklemeleri engellemek
- API belirteçlerini iptal etmek
- ilişkili içeriği geçici olarak silmek
- yayımlama erişimini kısıtlamak
- tekrarlayan veya ağır ihlallerde bulunanları yasaklamak

Açıkça kötüye kullanım niteliğindeki durumlarda yaptırımdan önce uyarı verileceğini garanti etmeyiz. Raporlar, moderasyon bekletmeleri,
gizli listelemeler, yasaklar ve hesap itibarı için
[Moderasyon ve Hesap Güvenliği](/clawhub/moderation) bölümüne bakın.
