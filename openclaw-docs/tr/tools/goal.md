---
doc-schema-version: 1
read_when:
    - OpenClaw'ın uzun bir oturum boyunca tek bir hedefi görünür tutmasını istiyorsunuz
    - Bir oturum hedefini duraklatmanız, sürdürmeniz, engellemeniz, tamamlamanız veya temizlemeniz gerekiyor
    - get_goal, create_goal ve update_goal araçlarını anlamak istiyorsunuz
    - Hedeflerin TUI'de nasıl göründüğünü görmek istiyorsunuz
summary: 'Oturum hedefleri: oturum başına kalıcı amaçlar, /goal denetimleri, model hedef araçları, token bütçeleri ve TUI durumu'
title: Amaç
x-i18n:
    generated_at: "2026-07-27T00:20:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8bfe25eb9901394b32b61729fbcb6a7bd711ed859d284fa39b637000ed7f0a18
    source_path: tools/goal.md
    workflow: 16
---

# Hedef

Bir **hedef**, mevcut OpenClaw oturumuna bağlı kalıcı bir amaçtır.
Uzun süren çalışmalar için arka plan görevi, hatırlatıcı, cron işi veya
daimi emir oluşturmadan temsilci ile operatöre ortak bir amaç sunar.

Hedefler oturum durumudur: oturum anahtarıyla birlikte taşınır, süreç yeniden
başlatmalarından etkilenmez ve `/goal`, modele yönelik hedef araçları ile TUI
alt bilgisinde görünür.

Ayrılmış komut tamamlamaları, kullanıcıya yönelik kaynak ileti dizisine döner; böylece
komut yürütülürken ayrı bir korumalı alan ilkesi oturumu kullanılmış olsa bile sonraki
tur aynı hedefi görmeye devam eder.

## Hızlı başlangıç

```text
/goal start 87469 numaralı PR için CI'ı başarılı duruma getir ve düzeltmeyi gönder
/goal
/goal edit 87469 numaralı PR için CI'ı başarılı duruma getir, düzeltmeyi gönder ve belgeleri güncelle
/goal pause CI bekleniyor
/goal resume
/goal complete gönderildi ve doğrulandı
/goal clear
```

`start` isteğe bağlıdır: `/goal get CI green for PR 87469` da bir hedef oluşturur;
çünkü `/goal` sonrasındaki bilinen bir eylem sözcüğü olmayan her metin
yeni bir amaç olarak değerlendirilir.

## Hedefler ne için kullanılır?

Bir oturumda birçok tur boyunca görünür kalması gereken somut bir sonuç
olduğunda hedef kullanın:

- PR kapatma süreci: düzeltin, doğrulayın, otomatik inceleme yapın, gönderin ve PR'ı açın veya güncelleyin.
- Hata ayıklama çalışması: hatayı yeniden üretin, sorumlu yüzeyi belirleyin, yamalayın ve
  düzeltmeyi kanıtlayın.
- Belgelendirme geçişi: ilgili belgeleri okuyun, yeni sayfayı yazın, çapraz bağlantılar ekleyin ve
  belge derlemesini doğrulayın.
- Bakım görevi: mevcut durumu inceleyin, sınırlı değişiklikler yapın, doğru
  kontrolleri çalıştırın ve nelerin değiştiğini bildirin.

Hedef, görev kuyruğu değildir. Çalışmanın ayrılmış biçimde yürütülmesi,
bir programa göre tekrarlanması, yönetilen alt çalışmalara dağıtılması veya
ilke olarak kalıcı olması gerektiğinde [Task Flow](/tr/automation/taskflow),
[görevleri](/tr/automation/tasks), [cron işlerini](/tr/automation/cron-jobs) ya da
[daimi emirleri](/tr/automation/standing-orders) kullanın.

## Komut başvurusu

Bağımsız değişken olmadan `/goal`, mevcut hedef özetini yazdırır:

```text
Hedef
Durum: etkin
Amaç: 87469 numaralı PR için CI'ı başarılı duruma getir ve düzeltmeyi gönder
Kullanılan token: 12k
Token bütçesi: 12k/50k

Komutlar: /goal edit <objective>, /goal pause, /goal complete, /goal clear
```

| Komut                                               | Etki                                                                     |
| --------------------------------------------------- | ------------------------------------------------------------------------ |
| `/goal` veya `/goal status`                           | Mevcut hedefi gösterir.                                                  |
| `/goal start <objective>`                           | Mevcut oturum için yeni bir hedef oluşturur.                             |
| `/goal set <objective>`, `/goal create <objective>` | `start` için diğer adlardır.                                              |
| `/goal <objective>`                                 | Ayrıca yeni bir hedef oluşturur (tanınan bir eylem sözcüğü olmayan her metin). |
| `/goal edit <objective>`                            | Mevcut amacı yeniden ifade eder; durum ve token muhasebesi değişmez.     |
| `/goal pause [note]`                                | Etkin bir hedefi duraklatır.                                             |
| `/goal resume [note]`                               | Duraklatılmış, engellenmiş, kullanım veya bütçe sınırına ulaşmış bir hedefi sürdürür. |
| `/goal complete [note]`                             | Hedefi başarılmış olarak işaretler.                                      |
| `/goal done [note]`                                 | `complete` için diğer addır.                                                 |
| `/goal block [note]`                                | Hedefi engellenmiş olarak işaretler.                                     |
| `/goal blocked [note]`                              | `block` için diğer addır.                                                 |
| `/goal clear`                                       | Hedefi oturumdan kaldırır.                                               |

Bir oturumda aynı anda yalnızca bir hedef bulunabilir. Geçerli hedef temizlenene
kadar ikinci bir hedef başlatma işlemi `Goal error: goal already exists` ile başarısız olur.

`/goal start`, token bütçesi bayrağı kabul etmez; bütçe yalnızca modele yönelik
`create_goal` aracı üzerinden ayarlanabilir.

## Durumlar

- `active`: oturum hedefi gerçekleştirmeye çalışmaktadır.
- `paused`: operatör hedefi duraklatmıştır; `/goal resume` hedefi yeniden
  etkinleştirir.
- `blocked`: temsilci veya operatör gerçek bir engel bildirmiştir; yeni bilgi
  veya durum mevcut olduğunda `/goal resume` hedefi yeniden etkinleştirir.
- `budget_limited`: yapılandırılan token bütçesine ulaşılmıştır; `/goal resume`,
  aynı amaç doğrultusundaki çalışmayı yeni bir bütçe penceresiyle yeniden başlatır.
- `usage_limited`: gelecekteki bir kullanım sınırı durdurma durumu için ayrılmıştır; `/goal
resume` çalışmayı aynı şekilde yeniden başlatır.
- `complete`: hedef başarılmıştır. Tamamlanan hedefler son durumdadır; başka bir hedef başlatmadan önce `/goal
clear` kullanın.

`/new` ve `/reset`, kasıtlı olarak yeni bir oturum bağlamı
başlattıkları için mevcut oturum hedefini temizler.

## Token bütçeleri

Hedeflerin, `create_goal` aracının `token_budget` parametresi üzerinden
ayarlanan isteğe bağlı pozitif bir token bütçesi olabilir. Bütçe, hedefin
oluşturulduğu sıradaki güncel oturum token sayısından itibaren ölçülür. Hedef
başladığında oturumda yalnızca eski veya bilinmeyen bir token anlık görüntüsü
varsa OpenClaw sonraki güncel anlık görüntüyü bekler ve bunu başlangıç noktası
olarak kullanır; böylece hedef oluşturulmadan önce harcanan token'lar hedefe
yansıtılmaz.

Kullanım bütçeye ulaştığında hedef `budget_limited` durumuna geçer. Bu işlem
hedefi silmez veya amacı ortadan kaldırmaz; operatöre ve temsilciye hedef
sürdürülene ya da temizlenene kadar artık etkin biçimde takip edilmediğini
bildirir. Sürdürme işlemi, mevcut güncel token sayısında yeni bir bütçe
penceresi başlatır.

Token bütçeleri bir faturalandırma üst sınırı değil, oturum hedefi için bir
koruyucu sınırdır. Sağlayıcı kotası, maliyet raporlaması ve bağlam penceresi
davranışı normal OpenClaw kullanım ve model denetimlerini kullanmaya devam
eder.

## Model araçları

OpenClaw, temsilci çalıştırma düzenlerine üç hedef aracı sunar:

| Araç          | Amaç                                                                                                                     |
| ------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `get_goal`    | Mevcut oturum hedefini okur: durum, amaç, token kullanımı ve token bütçesi.                                               |
| `create_goal` | Yalnızca kullanıcı veya sistem talimatları açıkça istediğinde hedef oluşturur. Oturumda zaten hedef varsa başarısız olur. |
| `update_goal` | Hedefi `complete` veya `blocked` olarak işaretler.                                                                    |

Model bir hedefi sessizce duraklatamaz, sürdüremez, temizleyemez veya
değiştiremez. Bunlar `/goal` ve sıfırlama komutları aracılığıyla
operatör/oturum denetimleri olarak kalır; böylece temsilci hedefi sessizce
değiştirmeden başarıyı veya gerçek bir engeli bildirebilir.

`update_goal`, bir hedefi yalnızca amaç gerçekten gerçekleştirildiğinde
`complete` olarak işaretlemelidir. Bir hedefi sıradan zorluklar veya eksik
iyileştirmeler nedeniyle değil, yalnızca aynı engelleyici durum art arda en az
üç hedef turunda tekrarlandıktan sonra `blocked` olarak işaretlemelidir.

## Her turdaki hedef bağlamı

Etkin hedef içeren her kullanıcı/sohbet turunda şu kullanıcı rolü bağlam satırı bulunur:

```text
Etkin hedef: <objective> — ilerletin veya durumunu güncelleyin (get_goal/update_goal).
```

OpenClaw, uzun amaçları kısaltarak satırı kompakt tutar. Duraklatılmış,
engellenmiş, bütçe veya kullanım sınırına ulaşmış ve tamamlanmış hedefler
eklenmez; böylece operatörün durdurma kararı hedef sürdürülene kadar geçerli
kalır.

## Control UI

Web Control UI, hedefi sohbet oluşturucusunun üzerinde kompakt bir rozet olarak
gösterir: bir durum simgesi, durum etiketi (örneğin `Pursuing goal`), kısaltılmış
amaç ve canlı geçen süre sayacı.

Rozet satır içi denetimler içerir:

- **Kalem**, amacın yeniden ifade edilip gönderilebilmesi için oluşturucuyu
  `/goal edit <objective>` ile önceden doldurur.
- **Duraklat / sürdür**, mevcut duruma göre `/goal pause` ile `/goal resume`
  arasında geçiş yapar.
- **Çöp kutusu**, `/goal clear` gönderir.
- **Şevron**, tam amacı, en son durum notunu, token kullanımını ve geçen süreyi
  göstermek için rozeti genişletir.

Oluşturucu gönderim yapamadığında (örneğin Gateway bağlantısı kesildiğinde)
eylem düğmeleri gizlenir; genişletme şevronu çalışmaya devam eder.

## TUI

TUI alt bilgisi, etkin oturumun hedefini token/mod göstergelerinden önce
temsilci, oturum ve model alanlarının yanında görünür tutar.

Alt bilgi örnekleri:

- Token bütçesi olan etkin bir hedef için `Pursuing goal (12k/50k)`.
- Duraklatılmış bir hedef için `Goal paused (/goal resume)`.
- Engellenmiş bir hedef için `Goal blocked (/goal resume)`.
- Kullanım sınırına ulaşmış bir hedef için `Goal hit usage limits (/goal resume)`.
- Bütçe sınırına ulaşmış bir hedef için `Goal unmet (50k/50k)`.
- Tamamlanmış bir hedef için `Goal achieved (42k)`.

Alt bilgi kasıtlı olarak kompakt tutulur. Tam amaç, not, token bütçesi ve
kullanılabilir komutlar için `/goal` kullanın.

## Kanal davranışı

`/goal`, TUI ve metin komutlarına izin veren sohbet yüzeyleri dahil
olmak üzere komut destekli OpenClaw oturumlarında çalışır. Hedef durumu
aktarıma değil oturum anahtarına bağlıdır; dolayısıyla bir oturum anahtarını
paylaşan iki yüzey aynı hedefi görür.

Hedef durumu bir teslim yönergesi değildir: yanıtları bir kanal üzerinden
gönderilmeye zorlamaz, kuyruk davranışını değiştirmez, araçları onaylamaz veya
çalışma planlamaz.

## Sorun giderme

| İleti                                  | Anlam                                                                                                                                        |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `Goal error: goal already exists`      | Oturumda zaten bir hedef var. İncelemek için `/goal`, tamamlandıysa `/goal complete` veya farklı bir amaç başlatmadan önce `/goal clear` kullanın. |
| `Goal error: goal not found`           | Oturumda henüz hedef yok. `/goal start <objective>` ile bir hedef başlatın.                                                                  |
| `Goal error: goal is already complete` | Hedef son durumdadır. Başka bir amacı başlatmadan veya sürdürmeden önce hedefi temizleyin.                                                    |

Token kullanımı `0` gösteriyorsa veya eski görünüyorsa etkin
oturumda henüz güncel bir token anlık görüntüsü bulunmayabilir. OpenClaw oturum
kullanımını ve dökümden türetilen toplamları kaydettikçe kullanım yenilenir.

## İlgili

- [Eğik çizgi komutları](/tr/tools/slash-commands)
- [TUI](/tr/web/tui)
- [Oturum aracı](/tr/concepts/session-tool)
- [Compaction](/tr/concepts/compaction)
- [Task Flow](/tr/automation/taskflow)
- [Daimi emirler](/tr/automation/standing-orders)
