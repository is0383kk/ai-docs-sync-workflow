---
read_when:
    - Bir agenti veya aracı kimin çalıştırdığını, ne zaman çalıştığını ve nasıl sonuçlandığını yanıtlayabilmeniz gerekir
    - İçerik içermeyen gelen veya giden mesaj yaşam döngüsü meta verilerine ihtiyacınız var
    - Sınırları belirlenmiş, redaksiyon açısından güvenli bir etkinlik dışa aktarımına ihtiyacınız var
summary: Yalnızca meta veri içeren çalıştırma, araç ve ileti yaşam döngüsü denetim kayıtları için CLI referansı
title: Denetim kayıtları
x-i18n:
    generated_at: "2026-07-26T23:14:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: da9df6f388b0a24c3b79d755fa59d047cce99262bc6d9c890be7a83da75693a8
    source_path: cli/audit.md
    workflow: 16
---

# `openclaw audit`

Ajan çalıştırmaları, araç eylemleri ve isteğe bağlı mesaj yaşam döngüsü kayıtları için Gateway'in yalnızca meta veri içeren denetim defterini sorgulayın.

Defter, çalıştırma ve araç olayları için varsayılan olarak etkindir. Tüm yeni olay kayıtlarını durdurmak için
[`audit.enabled: false`](/tr/gateway/configuration-reference#audit) ayarını yapın ve
Gateway'i yeniden başlatın. Mesaj kayıtları ise varsayılan olarak ayrıca devre dışıdır;
bunları kaydetmek için `audit.messages` değerini `direct` veya `all` olarak ayarlayın ve Gateway'i yeniden başlatın. Mevcut kayıtlar, süreleri dolana kadar (30 gün) sorgulanabilir durumda kalır.

Defter, konuşma dökümlerinden ayrıdır: kimlik, sıralama, kaynak, eylem, durum ve normalleştirilmiş sonuç kodlarını kaydeder ancak hiçbir zaman içerik depolamaz; mesaj tanımlayıcıları yalnızca kuruluma özgü, anahtarlı takma kimlikler olarak görünür. [Denetim geçmişi](/tr/gateway/audit), tam veri modelini, gizlilik semantiğini, depolama/saklama sınırlarını ve kapsam kısıtlarını tanımlar; bu sayfa komut yüzeyini ele alır.

```bash
openclaw audit
openclaw audit --agent main --status failed
openclaw audit --session "agent:main:main" --after 2026-07-01T00:00:00Z
openclaw audit --run 8c69f72e-8b11-4c54-98d5-1a3dd67450c3
openclaw audit --kind tool_action --limit 50 --json
openclaw audit --kind message --direction outbound --channel telegram --json
```

## Filtreler

- `--agent <id>`: tam ajan kimliği
- `--session <key>`: tam oturum anahtarı
- `--run <id>`: tam çalıştırma kimliği
- `--kind <kind>`: `agent_run`, `tool_action` veya `message`
- `--status <status>`: `started`, `succeeded`, `failed`, `cancelled`,
  `timed_out`, `blocked` veya `unknown`
- `--direction <direction>`: mesaj yönü, `inbound` veya `outbound`
- `--channel <channel>`: tam mesaj kanalı
- `--after <timestamp>` / `--before <timestamp>`: sınırlar dâhil ISO zaman damgası veya
  Unix milisaniyesi
- `--limit <count>`: 1 ile 500 arasında sayfa boyutu; varsayılan `100`
- `--cursor <sequence>`: önceki en yeniden en eskiye sorguyu sürdürür
- `--json`: sınırlı sayfayı JSON olarak yazdırır

CLI, tek bir komutun yapılandırılmış defterin tamamını göstermesi için sürümlendirilmiş etkinlik RPC'sini sorgular. Metin çıktısı zamanı, türü, yönü, kanalı, durumu, ajanı, çalıştırmayı ve eylemi gösterir. Eksik mesaj kaynak bilgisi `-` olarak görüntülenir; OpenClaw ajan veya çalıştırma kimlikleri uydurmaz. Araç eylemleri ayrıca araç adını gösterir. Başka bir sayfa varsa JSON çıktısı `nextCursor` içerir. Sayfalama sırasında gelen kayıtları yeniden sıralamadan devam etmek için bu değeri `--cursor` seçeneğine iletin.

Mesaj gövdeleri ve ham mesaj kimliği alanları bulunmasa da bu dışa aktarımlar hassas operasyonel meta veri niteliğini korur. Ajan, oturum ve çalıştırma kimlikleri, zamanlama, kanallar, sonuçlar ve kararlı HMAC referansları etkinlikleri ilişkilendirebilir. Bunları diğer operatör kayıtlarıyla aynı erişim denetimleri ve saklama uygulamalarıyla koruyun.

## Kaydedilen olaylar

Gateway, güvenilir yaşam döngüsü akışlarını altı eyleme yansıtır:

- `agent.run.started`
- `agent.run.finished`
- `tool.action.started`
- `tool.action.finished`
- `message.inbound.processed`
- `message.outbound.finished`

Döndürülen her kayıt; kararlı bir olay kimliği, monoton olarak artan bir defter sıra numarası, yaşam döngüsü zaman damgası, aktör, eylem, durum, bir
`schemaVersion: 1` işareti, kaynak sıra numarası ve `redaction: "metadata_only"` içerir.
Ajan/oturum/çalıştırma kaynak bilgileri ve olaya özgü alanlar yalnızca güvenilir kaynak bunları sağladığında bulunur. Mesaj kayıtları `sessionKey` ve `sessionId` alanlarını kasıtlı olarak içermez; bu nedenle `--session` filtreleri yalnızca çalıştırma ve araç kayıtlarında çalışır.

Sonlandırılmış çalıştırma ve araç kayıtları; başarıyı, başarısızlığı, iptali, zaman aşımını ve politika engellemelerini kapalı durum ve hata kodlarıyla birbirinden ayırır. Bir üst çalışma zamanı yetkili bir sonlandırma sonucu sunmadığında `unknown`, açıkça başarısız bir sonuçtur. Araç çağrısı kimlikleri yalnızca kararlı parmak izleri olarak dışa aktarılır. Araç adları, modele yönelik kısa ad sözleşmesiyle eşleşmelidir; diğer değerler `unknown` olur.

Mesaj kayıtları; yön, kanal, konuşma türü ve sonucun yanı sıra isteğe bağlı teslimat türü, hata aşaması, süre, sonuç sayısı, normalleştirilmiş neden kodu ve anahtarlı hesap/konuşma/mesaj/hedef takma kimliklerini ekler. Geçerli gelen ileti sınırı, çekirdek yineleme ve sonlandırılmış işleme sonuçları dâhil olmak üzere çekirdek dağıtıma ulaşan kabul edilmiş mesajları kapsar. Giden ileti sınırı, paylaşılan kalıcı teslimata ulaşan her özgün mantıksal yanıt yükü için bir sonlandırılmış satır yazar; parçalara ayırma ve adaptör yayılımı `resultCount` içinde birleştirilir. Kuyruğa alınmış, yeniden denenebilir veya belirsiz gönderimler yalnızca bir onay, ölü mektup ya da uzlaştırma sonucu sonlandırılmış hâle geldikten sonra kaydedilir. Bu paylaşılan sınırları atlayan Plugin'e özgü ve doğrudan gönderim yolları henüz kapsanmamaktadır; bir satırın bulunmaması, hiçbir mesajın var olmadığını kanıtlamaz.

Denetim defteri; dökümlerin, görev geçmişinin, Cron çalıştırma geçmişinin veya günlüklerin yerini almaz. Konuşma içeriğini başka bir depoya kopyalamadan operatör soruları için çalıştırmalar arası küçük bir dizin sağlar.

Gelen satırlarda `durationMs`, çekirdek dağıtımını ölçer ve `resultCount`, sonlandırılmış kuyruklu araç, engelleme ve yanıt yüklerini sayar. Giden satırlarda `durationMs`, sonlandırılmasına kadar teslimat sahipliğini (ve dolayısıyla kuyrukta bekleme süresini) içerirken `resultCount`, tanımlanmış fiziksel platform gönderimlerini sayar. `deliveryKind`, mevcut olduğunda kanca sonrası, oluşturma sonrası etkili yükü açıklar; engellenmiş ve kilitlenme nedeniyle belirsiz satırlarda bu alan bulunmaz.

## Gateway RPC

`audit.activity.list`, `operator.read` gerektirir ve aynı filtreleri kabul eder. Çalıştırma, araç, gelen mesaj ve giden mesaj kayıtları dâhil olmak üzere adlandırılmış V1 etkinlik olayı birleşimini döndürür.

```bash
openclaw gateway call audit.activity.list --params '{"channel":"telegram","limit":50}'
```

Sonuç `{ "events": AuditActivityEventV1[], "nextCursor"?: string }` olur.
Sonuçlar en yeniden en eskiye sıralanır ve istek başına 500 kayıtla sınırlıdır.

Yayımlanan `audit.list` RPC, eski çalıştırma/araç istemcileri için değişmeden kalır. `audit.activity.list` eski bir Gateway'de kullanılamadığında CLI, yalnızca istenen her filtre bu eski yöntem tarafından destekleniyorsa `audit.list` çağrısını yeniden dener. `--kind message`,
`--direction` ve `--channel`, sessizce yok sayılmak yerine eski bir Gateway'de yükseltme mesajıyla başarısız olur.

## İlgili

- [Denetim geçmişi](/tr/gateway/audit)
- [Gateway protokolü](/tr/gateway/protocol#audit-ledger-rpc)
- [Oturumlar](/tr/cli/sessions)
- [Görevler](/tr/cli/tasks)
- [Cron işleri](/tr/automation/cron-jobs)
