---
read_when:
    - OpenClaw zengin yanıtlarını işleyen Matrix istemcileri oluşturma
    - com.openclaw.presentation olay içeriğinde hata ayıklama
summary: OpenClaw uyumlu istemciler için Matrix MessagePresentation meta verileri
title: Matrix sunum meta verileri
x-i18n:
    generated_at: "2026-07-26T23:10:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0de4d13c6cefc6f91dcc7a4b0edeea6bf001f3bd71f52c9f0498ad422783d8a
    source_path: channels/matrix-presentation.md
    workflow: 16
---

OpenClaw, giden Matrix `m.room.message` olaylarına `com.openclaw.presentation` içerik anahtarı altında normalleştirilmiş `MessagePresentation` meta verileri ekler.

Standart Matrix istemcileri düz metin `body` içeriğini işlemeye devam eder. OpenClaw destekli istemciler yapılandırılmış meta verileri okuyabilir ve düğmeler, seçimler, bağlam satırları ve ayırıcılar gibi yerel kullanıcı arayüzlerini işleyebilir.

## Olay içeriği

```json
{
  "msgtype": "m.text",
  "body": "Model seçin\n\nModel seçin:\n- DeepSeek",
  "com.openclaw.presentation": {
    "version": 1,
    "type": "message.presentation",
    "title": "Model seçin",
    "tone": "info",
    "blocks": [
      {
        "type": "select",
        "placeholder": "Model seçin",
        "options": [
          {
            "label": "DeepSeek",
            "value": "/model deepseek/deepseek-chat"
          }
        ]
      }
    ]
  }
}
```

- `version` meta veri şeması sürümüdür; mevcut sürüm `1` değeridir. `type` kararlı bir ayırt edicidir ve her zaman `"message.presentation"` değerindedir. Matrix bağdaştırıcısı yalnızca tam olarak bu sürüme ve türe sahip yükleri gönderir; istemciler de güvenli biçimde yorumlayamadıkları bilinmeyen sürümleri, bilinmeyen `type` değerlerini ve bilinmeyen blok türlerini yok saymalıdır.
- `title` ve `tone` (`info`, `success`, `warning`, `danger`, `neutral`) isteğe bağlı ipuçlarıdır.
- Düğmeler ve seçim seçenekleri, eski dize `value` ile birlikte türü belirlenmiş bir `action` (`{ "type": "command", "command": "/..." }` veya `{ "type": "callback", "value": "..." }`) taşıyabilir. Her ikisi de mevcutsa `action` tercih edilmelidir.

## Geri dönüş davranışı

OpenClaw, her zaman okunabilir bir düz metin geri dönüşünü `body` içine işler. Yapılandırılmış meta veriler ek niteliktedir ve temel Matrix birlikte çalışabilirliği için zorunlu olmamalıdır.

Geri dönüş işleme kuralları:

- `title`, `text` ve `context` içerikleri düz satırlar olarak işlenir.
- `command` eylemine sahip düğmeler, komutun kopyalanabilir kalması için ``label: `/command` `` olarak işlenir. `callback` eylemine sahip veya yalnızca eski bir `value` değeri bulunan düğmeler, opak geri çağırma değerlerinin gizli kalması için yalnızca etiketle işlenir; devre dışı bırakılmış düğmeler her zaman yalnızca etiketle işlenir. URL ve web uygulaması düğmeleri `label: URL` olarak işlenir.
- Seçim blokları, yer tutucuyu (veya `Options:`) başlık ve ardından yalnızca etiket içeren seçenek satırları olarak işler.
- Hiçbir şey işlenmezse, örneğin yalnızca ayırıcı içeren bir sunumda, gövde `---` değerine geri döner.

Desteklenmeyen istemciler geri dönüş metnini göstermeye devam eder. OpenClaw destekli istemciler, kopyalama, arama, bildirimler ve erişilebilirlik için geri dönüşü korurken görüntüleme amacıyla yapılandırılmış meta verileri tercih edebilir.

## Desteklenen bloklar

Matrix giden bağdaştırıcısı aşağıdakiler için yerel destek sunduğunu bildirir:

- `buttons`
- `select`
- `context`
- `divider`

`text` blokları her zaman geri dönüş gövdesi aracılığıyla desteklenir. Tüm blokları mümkün olduğunda kullanılan sunum ipuçları olarak değerlendirin; mesajın tamamını başarısız kılmak yerine bilinmeyen alanları ve blok türlerini yok sayın.

## Etkileşimler

Bu meta veriler Matrix geri çağırma anlam bilimi eklemez. Düğme ve seçim değerleri, genellikle eğik çizgi komutları veya metin komutları olan geri dönüş etkileşim yükleridir. Etkileşimi desteklemek isteyen bir Matrix istemcisi, denetim değerini (`action.command`, ardından `action.value`, ardından `value`) çözümler ve normal bir mesaj olarak odaya geri gönderir.

Örneğin, `/model deepseek/deepseek-chat` değerine sahip bir düğme, bu değer aynı odada şifrelenmiş bir Matrix metin mesajı olarak gönderilerek işlenebilir.

## Onay meta verileriyle ilişkisi

`com.openclaw.presentation` genel zengin mesaj sunumu içindir.

Onay istemleri, güvenlik açısından hassas durum, kararlar ve yürütme/plugin ayrıntıları taşıdığından özel `com.openclaw.approval` meta verilerini kullanır. Aynı olayda her iki meta veri anahtarı da mevcutsa istemciler özel onay işleyicisini tercih etmelidir.

## Medya mesajları

Bir yanıt birden çok medya URL'si içerdiğinde OpenClaw, her medya URL'si için bir Matrix olayı gönderir. İstemcilerin yinelenen işleyiciler olmadan tek bir kararlı yapılandırılmış yük alması için açıklama metni ve sunum meta verileri yalnızca ilk olaya eklenir. Uzun metin olaylara bölündüğünde de aynı kural geçerlidir: meta veriler yalnızca ilk olayda taşınır.

Sunum meta verilerini küçük tutun. Kullanıcı tarafından görülebilen büyük metinler `body` içinde kalmalı ve normal Matrix metin parçalama yolunu kullanmalıdır.
