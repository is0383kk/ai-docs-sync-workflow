---
read_when:
    - Barnacle veya ClawSweeper geri bildiriminin ardından takip etme
    - ClawSweeper'dan inceleme isteme
    - Barnacle, ClawSweeper, güncelliğini yitirmiş etiketler veya otomatik kapatmalarla ilgili hata ayıklama
sidebarTitle: PR review flow
summary: Barnacle ve ClawSweeper geri bildirimlerinin OpenClaw pull request'lerinin inceleme sürecinde ilerlemesine nasıl yardımcı olduğu.
title: Pull request inceleme akışı
x-i18n:
    generated_at: "2026-07-26T23:00:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e9bec4578d55d2279450e991480467946db7da5ca956f85c35b4221190b2babe
    source_path: reference/pull-request-review-flow.md
    workflow: 16
---

Bu sayfa, bir OpenClaw pull request'i açtıktan veya güncelledikten sonraki
inceleme akışını açıklar: Barnacle ve ClawSweeper'ın ne yaptığı, geri
bildirimlerinden yararlanarak PR'ın nasıl iyileştirileceği ve otomasyon sessiz
kaldığında nelerin kontrol edileceği.

Barnacle ve ClawSweeper, bakım sorumlularının inceleme kuyruğunu kullanılabilir
durumda tutmasına yardımcı olur. Bakım sorumlularının değerlendirmesinin yerini
almazlar.

## Barnacle

Barnacle, deterministik GitHub triyajıdır. Bilinen kuyruk yönetimi durumlarını
arar ve etiketler, yorumlar veya kapatma işlemleriyle yanıt verir.

Barnacle şu durumlarda işlem yapabilir:

- bir PR açıklaması çoğunlukla boşsa veya sorun bağlamı eksikse;
- bir PR'da yararlı kanıt yoksa;
- yalnızca dokümantasyon, yalnızca test, yalnızca yeniden düzenleme, yalnızca CI
  veya altyapı değişikliğinde bağlantılı bakım sorumlusu bağlamı yoksa;
- bir değişiklik çekirdek yerine ClawHub'a veya bir plugine ait görünüyorsa;
- dal ilgisiz çalışmalar içeriyorsa;
- bir yazarın 20'den fazla açık PR'ı varsa.

Barnacle, güvenilir depo iş akışı kodundan çalışır. Katkıda bulunanların kodunu
kullanıma almaz veya çalıştırmaz.

Yönlendirme etiketlerinin çoğu bakım sorumlusu veya otomasyon sinyalleridir; bu
nedenle katkıda bulunanların etiketleri kendilerinin eklemesi gerekmez.

## ClawSweeper

ClawSweeper, OpenClaw depoları için yapay zekâ destekli inceleme ve bakım
botudur. PR'ları inceleyebilir, kanıtları değerlendirebilir, kalıcı inceleme
yorumları bırakabilir ve korumalı onarım veya otomatik birleştirme akışlarında
bakım sorumlularına yardımcı olabilir.

Olumlu bir ClawSweeper sonucu destekleyici kanıttır, bakım sorumlusu onayı
değildir. Bir PR'ın birleştirilmeye hazır olup olmadığına ve ne zaman hazır
olduğuna yine bakım sorumluları karar verir.

ClawSweeper kuyruk tabanlıdır. Bir PR açtıktan, commit gönderdikten veya inceleme
isteği ekledikten hemen sonra yanıt beklemeyin. Bir ClawSweeper çalışmasından
sonraki etiket güncellemeleri de zaman alabilir.

Yeni PR'lar ClawSweeper inceleme kuyruğuna girer. Bakım sorumluları ayrıca
etiketler veya komutlarla inceleme, onarım ya da otomatik birleştirme akışlarını
kuyruğa alabilir. Katkıda bulunanların olağan güncellemelerinde, ClawSweeper'dan
yalnızca dalı, PR açıklamasını, kanıtı veya kodu güncelledikten sonra başka bir
inceleme isteyin. Ardından yeni bir PR yorumuyla yeni bir inceleme talep edin:

```text
@clawsweeper re-review
```

PR yazarları ayrıca `@clawsweeper re-run` kullanabilir; depoda yazma
erişimi olan kullanıcılar açık herhangi bir öğede her iki komutu da
kullanabilir. Sade `@clawsweeper review` komutu yalnızca bakım
sorumluları içindir. Sabırlı olun: İstenen değişiklikler mevcut olmadan yeniden
istemek yalnızca kuyruk gürültüsü oluşturur.

ClawSweeper inceleme konuşmaları bıraktığında bunları normal inceleme geri
bildirimi gibi ele alın ve aşağıdaki takip kontrol listesini kullanın.

Bir insan katkıda bulunan veya bakım sorumlusu PR'ı devralmış ve üzerinde etkin
olarak çalışıyorsa ClawSweeper'ı çağırmayın veya aynı anda PR üzerinde başka bir
çalışma yapmayın. Önce insan incelemesinin veya onarımının tamamlanmasını
bekleyin. Etkinlik durursa yazardan kanıt sağlamasının ya da başka güncellemeler
yapmasının istenip istenmediğini kontrol edin.

## İnceleme sırasında PR'ı iyileştirme

Barnacle, ClawSweeper veya bir bakım sorumlusu yanıt verdiğinde bu geri bildirimi
PR için sonraki adımların kontrol listesi olarak kullanın.

1. ClawSweeper'ın `Rank-up moves:` ve `Proof guidance:` bölümlerini ilgili PR'ın eylem
   listesi olarak okuyun. Derecelendirmeler ve etiketler sabit birleştirme
   hedefleri değil, inceleme sinyalleridir.
2. İstenen kod veya dokümantasyon değişikliğini gönderin; sorun, çözüm, kullanıcı
   etkisi ya da kanıt değiştiğinde PR açıklamasını güncelleyin.
3. Değişiklikle uyumlu kanıtlar kullanarak istenen kanıtı ekleyin.
4. Ele alınmış inceleme konuşmalarını kendiniz çözümleyin. Yalnızca bakım
   sorumlusunun veya inceleyicinin değerlendirmesine ihtiyaç duyduğunuzda yanıt
   verip konuşmayı açık bırakın.
5. Yalnızca dal, PR açıklaması, kanıt ve ilgili CI sonuçları güncel olduğunda
   yeniden inceleme isteyin. Yazar, bakım sorumlusu ve ClawSweeper arasında
   birden fazla güncelleme ve inceleme döngüsü olması normaldir.
6. Mümkün olduğunda tartışmayı PR üzerinde sürdürün. Yalnızca PR bakım sorumlusu
   koordinasyonu gerektirdiğinde, otomasyon engellenmiş göründüğünde veya sonraki
   kararı GitHub yorumlarında netleştirmek zor olduğunda Discord'daki
   `#clawtributors` bölümüne geçin. PR bağlantısını, mevcut durumu ve belirli
   soruyu ya da kalan kanıtı ekleyin.

PR açıklamasını güncel tutun. Yorumlar tartışmaya yardımcı olur, ancak PR
açıklaması bakım sorumlularının ve otomasyonun tekrar başvurduğu kalıcı özettir.

`status: ⏳ waiting on author`, sonraki eylemin PR yazarında olduğu anlamına gelir:
başka bir inceleme istemeden önce dalı, PR açıklamasını veya kanıtı güncelleyin
ya da eksik bağlamı içeren bir yanıt verin.

Yararlı kanıtlar arasında odaklanmış test çıktıları, CI sonuçları, ekran
görüntüleri, kayıtlar, terminal çıktıları, canlı gözlemler, hassas bilgileri
ayıklanmış günlükler veya yapı bağlantıları bulunur. Görsel değişiklikler için
uygulanabilir olduğunda öncesi ve sonrası ekran görüntülerini ekleyin. Kanıt
dosyaları için CI yapılarının, GitHub'a yüklenen ekran görüntülerinin veya
kayıtların bağlantılarını ya da hassas bilgileri ayıklanmış kısa bir günlük
alıntısını tercih edin. Üretilen kanıt dosyalarını gerçek dokümantasyon, test
veya ürün değişikliğinin parçası olmadıkları sürece commit etmeyin.

Hassas verilerin ayıklanması katkıda bulunanın sorumluluğundadır. Kanıt
göndermeden önce gizli bilgileri, tokenleri, özel URL'leri, kullanıcı verilerini
ve ilgisiz günlükleri kaldırın.

OpenClaw ayrıca ayrı bir bayatlık otomasyonu kullanır. Atanmamış sorunlar ve
PR'lar 14 günlük hareketsizlikten sonra bayat olarak işaretlenebilir, ardından 7
gün daha etkinlik olmazsa kapatılabilir. Atanmış PR'lar daha sonraki
güncellemelerden bağımsız olarak açıldıktan 27 gün sonra bayat olarak işaretlenir
ve bayat kaldıkları 7 gün boyunca etkinlik olmazsa kapatılır. Atanmış bir PR
hâlâ etkinse üzerinde çalışan bakım sorumlusuyla koordinasyon kurun.

## Otomasyon sessiz kaldığında

Bir bakım sorumlusu öğeyi zaten ele alıyorsa, inceleme veya onarım isteği hâlâ
kuyruktaysa, olay rutinse ya da ClawSweeper hattı istenen eylem için
yapılandırılmamışsa otomasyon sessiz kalabilir.

Güvenilir bir iş akışının güvenilmeyen katkıda bulunan kodunu çalıştırması
gerektiğinde de işlem yapmaktan kaçınabilir. Bu durumda bakım sorumluları bunun
yerine normal incelemeyi veya daha güvenli bir iş akışını kullanır.

## Sorun giderme

ClawSweeper hemen yanıt vermezse yeniden denemeden önce bekleyin. Hizmet kuyruk
tabanlıdır ve tekrarlanan yorumlar veya etiket değişiklikleri kuyruğu
hızlandırmadan iş parçacığının incelenmesini zorlaştırabilir.

Yardım istemeden önce şunları kontrol edin:

- PR açıklaması güncel;
- en son commit istenen değişikliği içeriyor;
- CI tamamlandı veya PR açıklaması kalan herhangi bir hatanın neden PR ile
  ilgisiz olduğunu açıklıyor;
- en son inceleme isteği bir PR yorumu olarak yapıldı:
  `@clawsweeper re-review`;
- bir bakım sorumlusu veya katkıda bulunan hâlihazırda PR üzerinde etkin olarak
  çalışmıyor;
- en son istek hâlâ normal ClawSweeper kuyruk gecikmesi içinde değil.

PR güncel olduktan birkaç saat sonra hâlâ ClawSweeper yanıtı yoksa veya PR
otomasyon tarafından engellenmiş görünüyorsa Discord'daki `#clawtributors`
bölümünden yardım isteyin. PR bağlantısını, ne beklediğinizi, ne zaman
istediğinizi ve son bot yorumundan bu yana nelerin değiştiğini ekleyin.

## Otomasyonu çatallama

Benzer inceleme otomasyonu isteyen projeler ClawSweeper'ı inceleyebilir veya
çatallayabilir:

- [openclaw/clawsweeper](https://github.com/openclaw/clawsweeper)
- [ClawSweeper dokümantasyonu](https://clawsweeper.bot/)

## İlgili

- [Katkıda bulunma](https://github.com/openclaw/openclaw/blob/main/CONTRIBUTING.md)
- [CI işlem hattı](/tr/ci)
