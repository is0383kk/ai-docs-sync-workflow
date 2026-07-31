---
read_when:
    - Bir skill'i, plugin'i veya paketi bildirme
    - Bekletilen, gizlenen veya engellenen bir listelemeyi kurtarma
    - ClawHub moderasyonu, yasaklamaları veya hesap durumunu anlama
sidebarTitle: Moderation and Account Safety
summary: ClawHub raporlarının, moderasyon bekletmelerinin, gizli listelemelerin, yasaklamaların ve hesap durumunun işleyişi.
title: Moderasyon ve Hesap Güvenliği
x-i18n:
    generated_at: "2026-07-26T23:11:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54c1e0860411e6599923ef4d7db65d5cd5406ec63bf67c52968b4f99d893ffef
    source_path: clawhub/moderation.md
    workflow: 16
---

# Moderasyon ve Hesap Güvenliği

ClawHub yayımlamaya açıktır, ancak herkese açık keşif ve yükleme yüzeyleri yine de
koruyucu önlemlere ihtiyaç duyar. Bildirimler, moderasyon bekletmeleri, gizli listelemeler ve hesap işlemleri;
bir sürüm veya hesap güvensiz, yanıltıcı ya da politika dışı göründüğünde
kullanıcıların korunmasına yardımcı olur.

Bu sayfa moderasyonu ve hesap durumunu kapsar. `Pass`,
`Review`, `Warn`, `Malicious` gibi denetim etiketleri ve risk seviyesi için
[Güvenlik Denetimleri](/clawhub/security-audits) bölümüne bakın.

Ayrıca [Güvenlik](/clawhub/security) ve
[Kabul edilebilir kullanım](/clawhub/acceptable-usage) bölümlerine bakın. Telif hakkı veya diğer içerik
haklarıyla ilgili endişeler için [İçerik Hakları Talepleri](/clawhub/content-rights) sayfasını kullanın.

## Bildirimler

Oturum açmış kullanıcılar Skills, Plugin ve paketleri bildirebilir.

ClawHub bildirimlerini yalnızca aşağıdakiler gibi güvenli olmayan pazar yeri içerikleri için kullanın:

- kötü amaçlı listelemeler
- yanıltıcı meta veriler
- beyan edilmemiş kimlik bilgileri veya izin gereksinimleri
- şüpheli yükleme talimatları
- kimliğe bürünme
- kötü niyetli kayıtlar veya ticari markanın kötüye kullanılması
- [Kabul edilebilir kullanım](/clawhub/acceptable-usage) kurallarını ihlal eden içerikler

Bir Skills sayfasındaki **Skill'i bildir** düğmesini veya paketler için paket bildirme
komutunu/API'sini kullanın.

ClawHub bildirimlerini üçüncü taraf bir Skills veya
Plugin'in kendi kaynak kodundaki güvenlik açıkları için kullanmayın. Bunları doğrudan yayımcıya veya listelemede
bağlantısı verilen kaynak deposuna bildirin. ClawHub, üçüncü taraf
Skills veya Plugin kodunun bakımını ya da yamalanmasını gerçekleştirmez.

`openclaw/clawhub` için GitHub Security Advisories, doğrudan
ClawHub'daki güvenlik açıklarına yöneliktir. Web sitesi, API, CLI, kayıt defteri, kimlik doğrulama,
tarama, moderasyon veya indirme/yükleme güven sınırlarındaki hatalar buna örnektir. ClawHub
güvenlik bildirimlerini üçüncü taraf Skills veya Plugin'lerdeki güvenlik açıkları için kullanmayın.

İyi bildirimler belirli ve eyleme dönüştürülebilirdir. Bildirim sisteminin kötüye kullanılması da
hesap işlemine yol açabilir.

## Kuruluş ve ad alanı talepleri

Kuruluş, marka, paket kapsamı, sahip kullanıcı adı veya ad alanı sahipliğiyle ilgili anlaşmazlıklar;
ürün içi bildirim akışı ya da hesap itiraz formu yerine
[Kuruluş ve Ad Alanı Talepleri](/clawhub/namespace-claims) sürecini kullanmalıdır.

Bir ad alanının ayrılması, devredilmesi, yeniden adlandırılması, gizlenmesi, karantinaya alınması, diğer ad verilmesi
veya başka şekilde incelenmesi gerektiğine ilişkin hassas olmayan kanıtların ClawHub personeli tarafından incelenmesi gerektiğinde
bu süreci kullanın. Herkese açık bir soruna gizli bilgiler, özel belgeler, özel hukuki
dosyalar, kişisel kimlik belgeleri, API belirteçleri veya DNS doğrulama belirteçleri
eklemeyin.

## Moderasyon bekletmeleri

Bazı ciddi bulgular veya politika sorunları, bir yayımcıyı ya da listelemeyi
moderasyon bekletmesine alabilir. Bu durumda etkilenen içerik herkese açık
keşiften gizlenebilir veya sorun incelenene kadar gelecekteki yayımlar gizli olarak başlayabilir.

Moderasyon bekletmeleri, ClawHub yüksek riskli
vakaları çözüme kavuştururken kullanıcıları korumayı amaçlar. Yanlış pozitif doğrulandığında da kaldırılabilirler.

## Gizli veya engellenmiş listelemeler

Bir listeleme bekletilmiş, gizlenmiş, karantinaya alınmış, iptal edilmiş veya başka bir nedenle
herkese açık yükleme yüzeylerinde kullanılamaz durumda olabilir.

Bu durumlardan birini görürseniz, sahibi sorunu çözmedikçe veya moderasyon listelemeyi geri yüklemedikçe
sürümü yüklemeyin.

Sahipler, bekletilmiş veya gizlenmiş kendi listelemelerine ilişkin tanılamaları görmeye devam edebilir. Bu
tanılamalar, ne olduğunu ve listelemenin herkese açık yüzeylere dönebilmesi için nelerin
değişmesi gerektiğini açıklamaya yardımcı olur.

## Yasaklamalar ve hesap durumu

ClawHub politikasını ihlal eden hesaplar yayımlama erişimini kaybedebilir. Ciddi kötüye kullanım;
hesabın yasaklanmasına, belirteçlerin iptaline, içeriğin gizlenmesine veya listelemelerin kaldırılmasına
yol açabilir. Yayımcı kötüye kullanım baskısı sinyalleri günlük olarak kontrol edilir. ClawHub'ın
potansiyel yasaklama eşiğine ulaşan sinyaller otomatik bir uyarıyı tetikleyebilir. Uyarı son tarihinden sonraki
ilk uygun tarama yayımcıyı hâlâ potansiyel yasaklama
eşiğine yerleştiriyorsa ClawHub hesap işlemini otomatik olarak uygulayabilir.
Daha düşük güvenilirlikteki ve zaman açısından sınırlandırılmış inceleme sinyalleri otomatik
yaptırımın dışında tutulur.

Silinmiş, yasaklanmış veya devre dışı bırakılmış hesaplar ClawHub API belirteçlerini kullanamaz. Bir hesap
işleminden sonra CLI kimlik doğrulaması başarısız olmaya başlarsa hesap
durumunu incelemek için web kullanıcı arayüzünde oturum açın. Oturum açma veya normal CLI erişimi, yasaklama ya da devre dışı bırakılmış hesap nedeniyle engellenmişse
kurtarma incelemesi için [ClawHub itiraz formunu](https://appeals.openclaw.ai/) kullanın.

Tarayıcı tarafından tetiklenen bir e-postada bir Skills veya Plugin sürümü kötü amaçlı olarak belirtiliyorsa,
engellenen gönderilmiş sürüm için saklanan tarama sonuçlarını indirin:
`clawhub scan download <slug> --version <version>`. Plugin'ler için
`--kind plugin` ekleyin. Tarama çıktısını inceleyin, listelemeyi düzeltin, sürüm
numarasını artırın ve düzeltilmiş sürümü yükleyin.

## Yayımcılar için rehberlik

Yanlış pozitifleri azaltmak ve kullanıcı güvenini artırmak için:

- adları, özetleri, etiketleri ve değişiklik günlüklerini doğru tutun
- gerekli ortam değişkenlerini ve izinleri beyan edin
- gizlenmiş yükleme komutlarından kaçının
- mümkün olduğunda kaynak bağlantısı verin
- Plugin'leri yayımlamadan önce deneme çalıştırmaları kullanın
- kullanıcılar veya moderatörler sürüm davranışı hakkında soru sorarsa açıkça yanıt verin
