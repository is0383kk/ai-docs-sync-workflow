---
read_when:
    - ClawHub güvenlik denetimi sonuçlarını anlama
    - Bir skill'in mi yoksa plugin'in mi kurulacağına karar verme
    - ClawHub denetim durumunu, risk düzeyini veya bulgularını açıklama
sidebarTitle: Security Audits
summary: Bir skill veya plugin yüklemeden önce ClawHub güvenlik denetimi sonuçları nasıl anlaşılır?
title: Güvenlik Denetimleri
x-i18n:
    generated_at: "2026-07-26T22:37:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4178a568c9b8e202da666ed95d2200ad73f931a22c7e473aeaba84545e8bb25
    source_path: clawhub/security-audits.md
    workflow: 16
---

# Güvenlik Denetimleri

ClawHub güvenlik denetimleri, bir skill veya Plugin'in kurulacak kadar güvenli
olup olmadığına karar vermenize yardımcı olur. Bir sürümün ne yaptığını, hangi yetkileri istediğini ve
dosyalara, hesaplara, kimlik bilgilerine, koda veya harici hizmetlere erişebilmesi için
öncesinde özellikle incelenmesi gereken bir unsur bulunup bulunmadığını gösterir.

Denetimler güçlü güvenlik göstergeleridir ancak bir sürümün
risksiz olduğunu garanti etmez. Hassas erişim vermeden önce her zaman sağduyulu davranın.

Ayrıca bkz. [Güvenlik](/clawhub/security), [Kabul edilebilir kullanım](/clawhub/acceptable-usage)
ve [Moderasyon ve Hesap Güvenliği](/clawhub/moderation).

## Kurulumdan önce kontrol edilmesi gerekenler

Kurulumdan önce şunları inceleyin:

- genel denetim durumu
- risk düzeyi
- listelenen tüm bulgular
- gerekli kimlik bilgileri, izinler veya ortam değişkenleri
- sahip, kaynak, sürüm, değişiklik günlüğü, indirme sayısı, yıldızlar ve diğer güven göstergeleri

Yalnızca anladığınız ve güvendiğiniz içerikleri kurun.

## Denetim durumu

Denetim durumu, denetim sonucuna nasıl tepki vermeniz gerektiğini belirtir:

| Durum      | Anlamı                                                                   |
| ----------- | ------------------------------------------------------------------------- |
| `Pass`      | Düşük riskin üzerinde görünür bir sorun bulunmadı.                                |
| `Review`    | Kurulumdan önce bulguları okuyun. Sürüm yine de meşru olabilir. |
| `Warn`      | Özellikle dikkatli olun. ClawHub, yüksek etkili bir endişe veya uyarı işareti buldu. |
| `Malicious` | Kurmayın.                                                           |
| `Pending`   | Denetimler henüz tamamlanmadı.                                             |
| `Error`     | Denetim tamamlanamadı.                                         |

Bir `Pass` güven vericidir ancak kendi değerlendirmenizin yerini tutmaz. Bu durum
özellikle içerik yayımlayabilen, verileri düzenleyebilen, komut çalıştırabilen, dosyaları okuyabilen veya
üretim sistemlerine erişebilen araçlar için önemlidir.

## Risk düzeyi

Risk düzeyi, etki alanını, yani sürümü amaçlandığı şekilde
kullandığınızda ne kadar yetkiye sahip göründüğünü açıklar.

| Risk düzeyi | Anlamı                                                                       |
| ---------- | ----------------------------------------------------------------------------- |
| `Low`      | Çok az hassas yetki veya kullanıcı etkisi bulundu.                          |
| `Medium`   | Sürüm; hesap erişimi veya veri değişiklikleri gibi kayda değer yetkilere sahiptir. |
| `High`     | Sürüm; yüksek etkili yetkilere, ciddi bulgulara veya kötü amaçlı sinyallere sahiptir. |

Risk düzeyi ve denetim durumu farklı soruları yanıtlar:

- Risk düzeyi şunu sorar: "Burada ne kadar yetki var?"
- Denetim durumu şunu sorar: "Bu sonuç karşısında ne yapmalıyım?"

Örneğin, yayımlama yapan bir skill, `Medium` riskle birlikte `Review` gösterebilir. Bu,
kötü amaçlı olduğu anlamına gelmez. Skill'in amacıyla uyumlu göründüğü ancak
kayda değer hesap yetkileriyle işlem yapabildiği anlamına gelir.

## Bulgular

Bulgular, bir denetim sonucunun neden gösterildiğini açıklar. Her bulgu genellikle şunları içerir:

- ne anlama geldiği
- neden işaretlendiği
- ilgili skill veya Plugin içeriği
- bir öneri

Bulgular `Info`, `Low`, `Medium`, `High` veya `Critical` olarak etiketlenebilir. Daha yüksek
önem derecesine sahip bulgular, risk düzeyine ve denetim durumuna daha güçlü katkıda bulunur.

Düşük güvenilirlikli bulgular, sayfanın
yararlı kanıtlara odaklanmasını sağlamak için herkese açık denetim özetinde gizlenir.

## ClawHub'ın kontrol ettikleri

ClawHub, gönderilen sürüm yapıtlarını denetler. Bunlar şunları içerir:

- skill talimatları veya Plugin meta verileri
- beyan edilen ortam değişkenleri ve izinler
- kurulum talimatları ve paket meta verileri
- dahil edilen dosyalar ve dosya manifestleri
- uyumluluk ve yetenek meta verileri

Temel soru tutarlılıktır: ad, özet, meta veriler, istenen
yetkiler ve gerçek içerik, kullanıcıların makul olarak bekleyeceği şeylerle uyumlu mu?

Güçlü davranışlar otomatik olarak kötü değildir. Birçok yararlı araç; kimlik bilgilerine,
yerel komutlara, sağlayıcı API'lerine veya paket kurulumlarına ihtiyaç duyar. Denetim, bu
yetkinin beklenen, açıklanmış ve orantılı olup olmadığını kontrol eder.

Yapıt sayfaları, tam denetime şu adresten bağlantı verir:

```text
/<owner>/skills/<slug>/security-audit
```

Denetim sayfası şunları bir araya getirir:

1. SkillSpector
2. VirusTotal
3. Risk analizi

## VirusTotal

ClawHub, denetim yığınında kötü amaçlı yazılım telemetrisi olarak VirusTotal'ı kullanır. VirusTotal,
dosya itibarı ve kötü amaçlı yazılım taraması için güvenilir bir endüstri standardıdır ve
ortaklığımız ClawHub'ın skill ve Plugin
incelemelerine daha kapsamlı güvenlik bilgileri eklemesini sağlar.

VirusTotal özellikle bilinen kötü amaçlı yapıtlar, motor eşleşmeleri ve
ClawHub'ın ajan odaklı incelemesini tamamlayan itibar sinyalleri açısından yararlıdır. Sağlayıcı
motor sayıları mevcut olduğunda denetim bunları sade bir dille şöyle özetler:

```text
62/62 sağlayıcı bu skill'i temiz olarak işaretledi.
```

veya:

```text
2/64 sağlayıcı bu skill'i kötü amaçlı, 1/64 sağlayıcı şüpheli ve 61/64 sağlayıcı temiz olarak işaretledi.
```

ClawHub'ın özetleyebileceği bir sağlayıcı sayısı telemetrisi olmadığında denetimde şu ifade yer alır:

```text
VirusTotal bulgusu yok
```

VirusTotal bir telemetri kaynağı olmaya devam eder. ClawHub'ın yapıt odaklı
kendi risk analizinin yerini tutmaz.

## Risk analizi

Risk analizi, ClawHub'ın kendi güvenlik denetimi
sistemi olan ClawScan tarafından dahili olarak desteklenir. Her sürümü ajanlara yönelik bir yapıt olarak inceler: talimatlar,
meta veriler, beyan edilen izinler, dosyalar, yetenek sinyalleri, statik tarama sinyalleri,
SkillSpector bulguları, VirusTotal telemetrisi ve yayımcı tarafından sağlanan bağlam.
Statik tarama sinyalleri, bu incelemenin dahili bağlamıdır; bağımsız,
herkese açık bir denetim bölümü veya kurulumu engelleyen bir hüküm değildir.

Risk analizi, istem enjeksiyonu, araçların kötüye kullanımı, kimlik bilgilerinin açığa çıkması,
güvenli olmayan yürütme, bellek veya bağlam zehirlenmesi ve aşırı özerklik gibi riskleri değerlendirmek için
[OWASP Agentic Skills Top 10](https://owasp.org/www-project-agentic-skills-top-10/)
çerçevesini kullanır.

ClawScan, ürkütücü görünen bir yeteneği otomatik olarak kötü amaçlı kabul etmez.
Yeteneğin açıklanıp açıklanmadığını, amaçla uyumlu olup olmadığını ve
sürümün belirtilen kullanım senaryosuyla desteklenip desteklenmediğini değerlendirir.
