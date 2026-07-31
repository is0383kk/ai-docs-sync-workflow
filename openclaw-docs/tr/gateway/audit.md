---
read_when:
    - İçeriği depolamadan Gateway'in yaptıklarına ilişkin kalıcı bir kayda ihtiyacınız var
    - Mesaj yaşam döngüsü denetimini etkinleştirip etkinleştirmemeye karar veriyorsunuz
    - Denetim kayıtlarının neyi kanıtladığını ve neyi kanıtlamadığını açıklamanız gerekir
summary: Ajan çalıştırmaları, araç eylemleri ve isteğe bağlı ileti yaşam döngüleri için yalnızca meta verilerden oluşan denetim geçmişi
title: Denetim geçmişi
x-i18n:
    generated_at: "2026-07-26T23:40:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1005b214a674f0f888d759837bd627be458cefcf9ed61bda722499333361dc45
    source_path: gateway/audit.md
    workflow: 16
---

# Denetim geçmişi

Gateway, paylaşılan OpenClaw durum veritabanında sınırlı ve yalnızca meta veri içeren bir denetim günlüğü tutar. Bu günlük; "hangi agent çalıştı, ne zaman çalıştı ve nasıl sonlandı", "bir çalıştırma hangi araç eylemlerini yürüttü" ve mesaj denetimi etkinleştirildiğinde "kabul edilen bir gelen mesaj yönlendirmeye ulaştı mı" ile "bir giden mesaj nihai teslimat durumuna ulaştı mı" gibi operasyonel soruları yanıtlar.

Günlük; kimlik, sıralama, kaynak, eylem, durum ve normalleştirilmiş sonuç kodlarını saklar. İstemleri, mesaj gövdelerini, araç bağımsız değişkenlerini, araç sonuçlarını, ekleri, dosya adlarını, URL'leri, komut çıktılarını veya ham hata metinlerini hiçbir zaman saklamaz.

## Kayıt aileleri

Denetim etkinleştirildiğinde (varsayılan) çalıştırma ve araç olayları kaydedilir. Mesaj yaşam döngüsü olayları isteğe bağlıdır ve varsayılan olarak devre dışıdır.

| Aile           | Eylemler                                                 | Varsayılan |
| -------------- | -------------------------------------------------------- | ---------- |
| Agent çalıştırmaları | `agent.run.started`, `agent.run.finished`                | açık       |
| Araç eylemleri | `tool.action.started`, `tool.action.finished`            | açık       |
| Mesajlar       | `message.inbound.processed`, `message.outbound.finished` | kapalı     |

Her kayıt; kararlı bir olay kimliği, monotonik bir günlük sırası, yaşam döngüsü zaman damgası, aktör, eylem, durum, `schemaVersion: 1` ve `redaction: "metadata_only"` içerir. Alanların tam başvurusu ve sorgu filtreleri için [Denetim kayıtları](/tr/cli/audit) bölümüne bakın.

## Mesaj yaşam döngüsü olayları

Nelerin kaydedileceğini seçmek için [`audit.messages`](/tr/gateway/configuration-reference#audit) değerini ayarlayın, ardından Gateway'i yeniden başlatın:

- `off` (varsayılan): mesaj kaydı yok.
- `direct`: yalnızca doğrudan görüşmelerdeki mesajlar.
- `all`: doğrudan, grup ve kanal mesajları.

İki yetkili sınır mesaj kayıtlarını oluşturur:

- **Gelen** satırlar, kabul edilen bir mesaj çekirdek yönlendirmeye ulaştığında yazılır; yinelenen ve nihai işleme sonuçları buna dahildir.
- **Giden** satırlar, paylaşılan dayanıklı teslimat nihai bir sonuca ulaştığında yazılır: gönderildi, engellendi, başarısız oldu veya çökme nedeniyle belirsiz gönderimler için açık bir `unknown`. Kuyruk kurtarma ve teslim edilemeyen ileti sonuçları dahildir. Her özgün mantıksal yanıt yükü için tek bir nihai satır oluşturulur; parçalara ayırma ve adaptör dağıtımı `resultCount` içinde birleştirilir.

### Görüşme türü sınıflandırması

`direct` modu bir gizlilik sınırıdır; bu nedenle bir mesaj, yalnızca hedef bilgileri bunu kanıtladığında doğrudan görüşme olarak sınıflandırılır: gönderme yolu hedef görüşme türünü bildirmiştir veya teslimat oturumu rotası, teslimatın yapıldığı kanal ve eşin adını tam olarak belirtir. İlke durumu veya kaynak görüşme gibi daha zayıf sinyaller, bir mesajı `group` olarak sınıflandırabilir (böylece `direct` toplamının dışında bırakır), ancak hiçbir zaman `direct` olduğunu ileri süremez. Doğrudan olduğu kanıtlanamayan mesajlar `unknown` olarak sınıflandırılır ve `direct` modunda kaydedilmez. Bu nedenle sohbet türlerini bildirmeyen kanallar, `direct` modunda `all` moduna kıyasla daha az satır kaydedebilir.

## Gizlilik modeli

Mesaj satırları hiçbir zaman ham platform tanımlayıcılarını saklamaz. İlişkilendirme mümkün olduğunda hesap, görüşme, mesaj ve hedef tanımlayıcıları yalnızca kuruluma özel anahtarlı takma adlar (`hmac-sha256:v1:<keyId>:<digest>`) olarak dışa aktarılır:

- HMAC anahtarı ilk kullanımda oluşturulur, tanımlayıcı türüne göre etki alanları ayrılır ve günlükle aynı durum veritabanında bulunur.
- Takma adlar tek bir kurulum içinde kararlıdır; böylece aynı görüşmeyle ilgili satırlar, platform tanımlayıcısını açığa çıkarmadan ilişkilendirilebilir.
- Bu, **anonimleştirme değil, ilişkilendirmedir**: durum veritabanına okuma erişimi olan herkes anahtara da sahiptir ve olası ham tanımlayıcıları takma adlarla karşılaştırabilir. RPC ve CLI dışa aktarımları anahtarı hiçbir zaman içermez.
- Mesaj satırları korunurken anahtar malzemesi eksik veya bozuksa Gateway, ilişkilendirmeyi bölecek yeni bir anahtara sessizce geçmek yerine güvenli biçimde kapanır ve yeni mesaj kayıtlarını bırakır.

Çalıştırma ve araç kayıtları, ilişkilendirme için `sessionKey` ve `sessionId` değerlerini korur; kanonik oturum anahtarlarının kendileri platform hesabı veya eş kimliklerini içerebilir. Mesaj kayıtları her ikisini de bilinçli olarak dışarıda bırakır.

Denetim dışa aktarımları, içerik olmasa bile hassas operasyonel meta veriler olarak kalır: zamanlama, kanallar, sonuçlar ve kararlı takma adlar etkinlikleri ilişkilendirebilir. Dışa aktarımları diğer operatör kayıtlarıyla aynı erişim denetimleri ve saklama uygulamalarıyla koruyun.

## Kapsam ve kanıt sınırları

Günlük, en iyi çaba esasına göre çalışır ve bilinçli olarak sınırlıdır. Onu gerçekleşenlerin kanıtı olarak değil, kaydedilenlerin kanıtı olarak değerlendirin:

- **Bir satırın bulunmaması hiçbir şeyi kanıtlamaz.** Kabul öncesinde bırakılan gelen mesajlar, çalışan bir Gateway kaydedicisi bulunmayan CLI süreçlerinden yapılan gönderimler ve paylaşılan dayanıklı teslimatı atlayan Plugin'e özgü veya doğrudan gönderim yolları hiçbir kayıt bırakmaz.
- Yazma işlemleri sınırlı bir arka plan çalışanı üzerinden gerçekleştirilir; çalışan arızası veya kuyruk doygunluğu kayıtların bırakılmasına neden olur ve tek bir operasyonel uyarı günlüğe kaydedilir.
- Çökme nedeniyle belirsiz giden gönderimler, uydurma sonuçlar yerine `unknown` olarak kaydedilir.

Bu günlük hata ayıklamayı ve operasyonel incelemeyi destekler. Kayıpsız bir uyumluluk arşivi değildir; böyle bir arşive ihtiyacınız varsa [OpenTelemetry](/tr/gateway/opentelemetry) veya kanal düzeyindeki araçlar tarafından beslenen harici bir sistem kullanın.

## Depolama, saklama ve geçiş

Kayıtlar paylaşılan durum veritabanında (`state/openclaw.sqlite`) bulunur ve teslimatın yoğun yürütme yolunun dışında yazılır. Sorgular hiçbir zaman 30 günden eski kayıtları döndürmez ve günlük 100.000 satırla sınırlıdır; süresi dolan satırlar başlangıçta, saatlik bakım sırasında ve sonraki yazma işlemlerinde temizlenir. Saklama bakımı, toplama devre dışı bırakıldığında bile çalışmaya devam eder.

Daha önceki yalnızca çalıştırma/araç günlüğüne sahip bir Gateway'den yükseltme yapıldığında şema, başlangıçta (veya `openclaw doctor --fix` aracılığıyla) otomatik olarak geçirilir; mevcut satırlar ve günlük sıraları korunur.

## Sorgulama

- CLI: agent, oturum, çalıştırma, tür, durum, yön, kanal, zaman sınırları ve imleçli sayfalama filtreleriyle [`openclaw audit`](/tr/cli/audit).
- Gateway RPC: `audit.activity.list` (`operator.read` gerektirir), sürümlendirilmiş V1 etkinlik olayı birleşimini döndürür; yayımlanan `audit.list` RPC, eski çalıştırma/araç istemcileri için değişmeden kalır. [Gateway protokolü](/tr/gateway/protocol#audit-ledger-rpc) bölümüne bakın.

## İlgili

- [Denetim kayıtları CLI'si](/tr/cli/audit)
- [Yapılandırma başvurusu](/tr/gateway/configuration-reference#audit)
- [Gateway protokolü](/tr/gateway/protocol#audit-ledger-rpc)
- [OpenTelemetry](/tr/gateway/opentelemetry)
