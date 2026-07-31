---
read_when:
    - Bir ajan araçları kullanırken steer'ın nasıl davrandığını açıklama
    - Etkin çalıştırma kuyruğu davranışını veya çalışma zamanı yönlendirme entegrasyonunu değiştirme
    - Yönlendirmeyi followup, collect ve interrupt kuyruk modlarıyla karşılaştırma
summary: Etkin çalıştırma yönlendirmesinin çalışma zamanı sınırlarında iletileri nasıl kuyruğa aldığı
title: Yönlendirme kuyruğu
x-i18n:
    generated_at: "2026-07-26T23:55:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 131f04f19934b9b1f6dd8ffb2cf2428950c319483abdc2ccdecec741809cda2a
    source_path: concepts/queue-steering.md
    workflow: 16
---

Bir oturum çalışması zaten akış hâlindeyken normal bir istem gelirse ve kuyruk modu `steer` ise (varsayılan; yapılandırma gerekmez), OpenClaw bu istemi etkin çalışma zamanına göndermeye çalışır. OpenClaw ve yerel Codex app-server yürütme düzeneği, teslim ayrıntılarını farklı biçimlerde uygular.

Bu sayfa, `steer` modundaki normal gelen iletiler için kuyruk modu yönlendirmesini kapsar. `followup` veya `collect` modunda normal iletiler bu yolu atlar ve etkin çalışma bitene kadar bekler. Açık `/steer <message>` komutu için [Yönlendir](/tr/tools/steer) bölümüne bakın.

## Çalışma zamanı sınırı

Yönlendirme, zaten çalışmakta olan bir araç çağrısını kesintiye uğratmaz. OpenClaw, model sınırlarında kuyruğa alınmış yönlendirme iletilerini denetler:

1. Asistan araç çağrıları ister.
2. OpenClaw, mevcut asistan iletisinin araç çağrısı grubunu yürütür.
3. OpenClaw, tur sonu olayını yayınlar.
4. OpenClaw, kuyruğa alınmış yönlendirme iletilerini kuyruktan çıkarır.
5. OpenClaw, bu iletileri bir sonraki LLM çağrısından önce kullanıcı iletileri olarak ekler.

Bu işlem, araç sonuçlarını onları isteyen asistan iletisiyle eşleştirilmiş hâlde tutar ve ardından bir sonraki model çağrısının en güncel kullanıcı girdisini görmesini sağlar.

Yerel Codex app-server yürütme düzeneği, OpenClaw çalışma zamanının dahili yönlendirme kuyruğu yerine `turn/steer` sunar. OpenClaw, kuyruğa alınmış istemleri yapılandırılmış sessiz pencere boyunca gruplar ve ardından toplanan tüm kullanıcı girdilerini geliş sırasına göre içeren tek bir `turn/steer` isteği gönderir.

Codex inceleme ve manuel Compaction turları, aynı turda yönlendirmeyi reddeder. Bir çalışma zamanı `steer` modunda yönlendirmeyi kabul edemediğinde OpenClaw, istemi başlatmadan önce etkin çalışmanın bitmesini bekler.

## Modlar

| Mod         | Etkin çalışma sırasındaki davranış                             | Sonraki davranış                                                                                 |
| ----------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `steer`     | Mümkün olduğunda istemi etkin çalışma zamanına yönlendirir.   | Yönlendirme kullanılamıyorsa etkin çalışmanın bitmesini bekler.                                  |
| `followup`  | Yönlendirme yapmaz.                                           | Etkin çalışma sona erdikten sonra kuyruğa alınmış iletileri yürütür.                              |
| `collect`   | Yönlendirme yapmaz.                                           | Uyumlu kuyruk iletilerini bekleme penceresinden sonra tek bir sonraki turda birleştirir.          |
| `interrupt` | Etkin çalışmayı yönlendirmek yerine iptal eder.                | İptal işleminden sonra en yeni iletiyi başlatır.                                                  |

## Yoğun ileti örneği

Temsilci bir araç çağrısını yürütürken dört kullanıcı ileti gönderirse:

- Varsayılan davranışta etkin çalışma zamanı, bir sonraki model kararından önce dört iletinin tümünü geliş sırasına göre alır. OpenClaw bunları bir sonraki model sınırında kuyruktan çıkarır; Codex ise bunları gruplandırılmış tek bir `turn/steer` olarak alır.
- `/queue collect` ile OpenClaw yönlendirme yapmaz. Etkin çalışma sona erene kadar bekler, ardından bekleme penceresinden sonra uyumlu kuyruk iletileriyle bir takip turu oluşturur.
- `/queue interrupt` ile OpenClaw etkin çalışmayı iptal eder ve yönlendirme yapmak yerine en yeni iletiyi başlatır.

## Kapsam

Yönlendirme her zaman mevcut etkin oturum çalışmasını hedefler. Yeni bir oturum oluşturmaz, etkin çalışmanın araç politikasını değiştirmez veya iletileri gönderene göre bölmez. Çok kullanıcılı kanallarda gelen istemler zaten gönderen ve rota bağlamını içerir; böylece bir sonraki model çağrısı her iletiyi kimin gönderdiğini görebilir.

İletilerin etkin çalışmaya yönlendirilmek yerine varsayılan olarak kuyruğa alınmasını istediğinizde `followup` veya `collect` kullanın. En yeni istemin etkin çalışmanın yerini alması gerektiğinde `interrupt` kullanın.

## Bekleme süresi

Yerleşik kuyruk bekleme süresi, kuyruğa alınmış `followup` ve `collect` teslimine uygulanır. Yerel Codex yürütme düzeneğiyle `steer` modunda, gruplandırılmış `turn/steer` gönderilmeden önceki sessiz pencereyi de ayarlar. OpenClaw için etkin yönlendirmenin kendisi bekleme zamanlayıcısını kullanmaz; çünkü OpenClaw iletileri bir sonraki model sınırına kadar doğal olarak gruplar.

## İlgili

- [Komut kuyruğu](/tr/concepts/queue)
- [Yönlendir](/tr/tools/steer)
- [İletiler](/tr/concepts/messages)
- [Temsilci döngüsü](/tr/concepts/agent-loop)
