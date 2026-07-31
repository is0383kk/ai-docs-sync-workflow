---
read_when:
    - Güvenlik bulgularına veya tehdit senaryolarına katkıda bulunmak istiyorsunuz
    - Tehdit modelini inceleme veya güncelleme
summary: OpenClaw tehdit modeline nasıl katkıda bulunulur
title: Tehdit modeline katkıda bulunma
x-i18n:
    generated_at: "2026-07-27T00:18:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4e2e5cd95e8a2bf5ee4bd167afedfadf9aa876e4260e2d0bfb5f414cd4255410
    source_path: security/CONTRIBUTING-THREAT-MODEL.md
    workflow: 16
---

[tehdit modeli](/tr/security/THREAT-MODEL-ATLAS) yaşayan bir belgedir. Herkesin katkısı memnuniyetle karşılanır; güvenlik veya MITRE ATLAS konusunda deneyimli olmanız gerekmez.

<Note>
Bu bölüm tehdit modeline ekleme yapmak içindir, mevcut güvenlik açıklarını bildirmek için değildir. İstismar edilebilir bir güvenlik açığı bulduysanız bunun yerine [Trust sayfasındaki](https://trust.openclaw.ai) sorumlu açıklama talimatlarını izleyin.
</Note>

## Katkıda bulunma yolları

**Tehdit ekleyin.** Saldırı senaryosunu kendi sözcüklerinizle açıklayan bir issue'yu [openclaw/trust](https://github.com/openclaw/trust/issues) üzerinde açın. Aşağıdakiler yararlıdır ancak zorunlu değildir:

- Saldırı senaryosu ve bunun nasıl istismar edilebileceği.
- Hangi bileşenlerin etkilendiği (CLI, gateway, kanallar, ClawHub, MCP sunucuları vb.).
- Önem derecesine ilişkin tahmininiz (düşük / orta / yüksek / kritik).
- İlgili araştırmalara, CVE'lere veya gerçek dünyadan örneklere bağlantılar.

Bakım sorumluları inceleme sırasında ATLAS eşlemesini, tehdit kimliğini ve risk düzeyini belirler.

**Bir azaltma önlemi önerin.** Tehdide atıfta bulunan bir issue veya PR açın. Açık ve uygulanabilir olun: "gateway'de gönderici başına dakikada 10 mesajlık hız sınırlaması", "hız sınırlaması uygulayın" ifadesinden daha yararlıdır.

**Bir saldırı zinciri önerin.** Saldırı zincirleri, birden fazla tehdidin gerçekçi bir senaryoda nasıl birleştiğini gösterir. Adımları ve bir saldırganın bunları nasıl zincirleyeceğini açıklayın; kısa bir anlatım resmî bir şablondan daha etkilidir.

**Mevcut içeriği düzeltin veya iyileştirin.** Yazım hataları, açıklamalar, güncelliğini yitirmiş bilgiler, daha iyi örnekler: PR'lar memnuniyetle karşılanır, issue açılması gerekmez.

## Çerçeve referansı

Tehditler; istem enjeksiyonu, araçların kötüye kullanılması ve agent istismarı gibi AI/ML'ye özgü tehditlere yönelik bir çerçeve olan [MITRE ATLAS](https://atlas.mitre.org/) (AI Sistemleri için Düşmanca Tehdit Ortamı) ile eşleştirilir. Katkıda bulunmak için ATLAS'ı bilmeniz gerekmez; bakım sorumluları gönderimleri inceleme sırasında eşler.

**Tehdit kimlikleri.** Her tehdit, inceleme sırasında bakım sorumluları tarafından atanan `T-EXEC-003` gibi bir kimlik alır.

| Kod     | Kategori                                    |
| ------- | ------------------------------------------- |
| RECON   | Keşif - bilgi toplama                       |
| ACCESS  | İlk erişim - sisteme giriş sağlama          |
| EXEC    | Yürütme - kötü amaçlı eylemler gerçekleştirme |
| PERSIST | Kalıcılık - erişimi sürdürme                |
| EVADE   | Savunmadan kaçınma - tespit edilmekten kaçınma |
| DISC    | Ortam keşfi - ortam hakkında bilgi edinme   |
| EXFIL   | Veri sızdırma - veri çalma                  |
| IMPACT  | Etki - hasar veya kesinti                   |

**Risk düzeyleri.** Düzeyden emin değilseniz yalnızca etkiyi açıklayın; bakım sorumluları düzeyi değerlendirir.

| Düzey        | Anlamı                                                            |
| ------------ | ----------------------------------------------------------------- |
| **Kritik**   | Sistemin tamamen ele geçirilmesi veya yüksek olasılık + kritik etki |
| **Yüksek**   | Önemli hasar olasılığı veya orta olasılık + kritik etki            |
| **Orta**     | Orta düzey risk veya düşük olasılık + yüksek etki                  |
| **Düşük**    | Düşük olasılık ve sınırlı etki                                    |

## İnceleme süreci

1. **Ön değerlendirme** - yeni gönderimler 48 saat içinde incelenir.
2. **Değerlendirme** - bakım sorumluları uygulanabilirliği doğrular, ATLAS eşlemesini ve tehdit kimliğini atar, risk düzeyini doğrular.
3. **Belgelendirme** - biçimlendirme ve eksiksizlik kontrolü yapılır.
4. **Birleştirme** - tehdit modeline ve görselleştirmeye eklenir.

## Kaynaklar

- [ATLAS web sitesi](https://atlas.mitre.org/)
- [ATLAS teknikleri](https://atlas.mitre.org/techniques/)
- [ATLAS vaka çalışmaları](https://atlas.mitre.org/studies/)

## İletişim

- **Güvenlik açıkları:** bildirim talimatları için [Trust sayfası](https://trust.openclaw.ai) veya `security@openclaw.ai`.
- **Tehdit modeliyle ilgili sorular:** [openclaw/trust](https://github.com/openclaw/trust/issues) üzerinde bir issue açın.
- **Genel sohbet:** Discord `#security` kanalı.

## Takdir

Tehdit modeline katkıda bulunanlara, tehdit modelinin teşekkür bölümünde ve sürüm notlarında yer verilir; önemli katkıda bulunanlar ayrıca OpenClaw güvenlik onur listesine alınır.

## İlgili

- [Tehdit modeli](/tr/security/THREAT-MODEL-ATLAS)
- [Olay müdahalesi](/tr/security/incident-response)
- [Biçimsel doğrulama](/tr/security/formal-verification)
