---
read_when:
    - Bir kullanıcı, ajanların araç çağrılarını tekrarlayıp takılı kaldığını bildiriyor
    - Yinelenen çağrı korumasını denetlemeniz gerekir
    - Agent araç/çalışma zamanı politikalarını düzenliyorsunuz
    - Bağlam taşması yeniden denemesinden sonra `compaction_loop_persisted` iptalleriyle karşılaşıyorsunuz
summary: Tekrarlanan araç çağrısı döngülerini algılayan koruma önlemleri nasıl etkinleştirilir?
title: Araç döngüsü algılama
x-i18n:
    generated_at: "2026-07-27T00:21:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 79b5aa1d85e02b8cf46a95b3bcebb255178b91456517cab804cce77b8f3b818e
    source_path: tools/loop-detection.md
    workflow: 16
---

OpenClaw, tekrarlayan araç çağrısı örüntülerine karşı birlikte çalışan iki korumaya sahiptir;
her ikisi de `tools.loopDetection` altında yapılandırılır:

1. **Döngü algılama** (`enabled`) - varsayılan olarak devre dışıdır. Tekrarlanan örüntüler ve bilinmeyen araç yeniden denemeleri için kayan
   araç çağrısı geçmişini izler.
2. **Compaction sonrası koruma** -
   `enabled` açıkça `false` olarak ayarlanmadığı sürece etkindir. Her Compaction yeniden denemesinden sonra devreye girer ve
   agent pencere içinde aynı `(tool, args, result)` üçlüsünü
   yinelerse çalıştırmayı iptal eder.

Her iki korumayı da susturmak için `tools.loopDetection.enabled: false` olarak ayarlayın.

## Bunun var olma nedeni

- İlerleme sağlamayan tekrarlayan dizileri algılamak.
- Yüksek frekanslı, sonuç üretmeyen döngüleri (aynı araç, aynı girdiler, tekrarlanan
  hatalar) algılamak.
- Bilinen yoklama araçlarına özgü tekrarlanan çağrı örüntülerini algılamak.
- Bağlam taşması -> Compaction -> aynı döngü çevrimlerini süresiz
  çalışmaya bırakmak yerine kesmek.

## Yapılandırma bloğu

Genel ayar:

```json5
{
  tools: {
    loopDetection: {
      enabled: false, // kayan geçmiş algılayıcıları için ana anahtar
    },
  },
}
```

Agent başına geçersiz kılma (isteğe bağlı, `agents.entries.*.tools.loopDetection` konumunda):

```json5
{
  agents: {
    list: [
      {
        id: "safe-runner",
        tools: {
          loopDetection: {
            enabled: true,
          },
        },
      },
    ],
  },
}
```

Agent başına ayar, genel ayarı geçersiz kılar.

### Alan davranışı

| Alan      | Varsayılan | Etki                                                                                              |
| --------- | ----------- | ------------------------------------------------------------------------------------------------- |
| `enabled` | `false` | Kayan geçmiş algılayıcıları için ana anahtar. `false` ayrıca Compaction sonrası korumayı devre dışı bırakır. |

`exec` için ilerleme yokluğu karması; kararlı komut sonuçlarını (durum,
çıkış kodu, zaman aşımı bayrağı, çıktı) karşılaştırır ve süre, PID, oturum kimliği
ve çalışma dizini gibi değişken çalışma zamanı meta verilerini yok sayar. Giden mesaj gönderme
sonuçlarının karması alınırken çağrı başına değişken kimlikler (mesaj kimliği, dosya kimliği, zaman damgası)
çıkarılır; böylece bir "gönderildi" sonucu, farklı bir "gönderildi"
sonucuyla aynı görünmez. Bir çalıştırma kimliği mevcut olduğunda geçmiş yalnızca o çalıştırma içinde
değerlendirilir; dolayısıyla zamanlanmış Heartbeat çevrimleri ve yeni çalıştırmalar, önceki
çalıştırmalardan kalan eski döngü sayılarını devralmaz.

## Önerilen kurulum

- Daha küçük modeller için `enabled: true` olarak ayarlayın. Amiral gemisi modeller kayan geçmiş algılamasına nadiren ihtiyaç duyar ve
  Compaction sonrası korumadan yararlanmaya devam ederken ana anahtarı
  `false` olarak bırakabilir.
- Compaction sonrası koruma dâhil her şeyi devre dışı bırakmak için
  açıkça `tools.loopDetection.enabled: false` olarak ayarlayın.

## Compaction sonrası koruma

Bağlam taşmasını izleyen bir Compaction yeniden denemesinden sonra çalıştırıcı,
sonraki birkaç araç çağrısında kısa pencereli bir korumayı devreye alır. Agent bu pencere içinde
aynı `(toolName, argsHash, resultHash)` üçlüsünü yeterince yinelerse koruma, Compaction işleminin
döngüyü kıramadığı sonucuna varır ve çalıştırmayı bir `compaction_loop_persisted` hatasıyla iptal eder.

Koruma, tek bir farkla ana `tools.loopDetection.enabled` bayrağı tarafından denetlenir:
bayrak ayarlanmamışken veya `true` iken **etkin kalır** ve yalnızca
bayrak açıkça `false` olarak ayarlandığında kapanır. Bu bilinçli bir tercihtir; koruma,
aksi takdirde sınırsız miktarda token tüketecek Compaction döngülerinden çıkmak için vardır,
bu nedenle yapılandırma yapmamış bir kullanıcı da korumadan yararlanır.

```json5
{
  tools: {
    loopDetection: {
      // ana anahtar; korumayı kayan algılayıcılarla birlikte devre dışı bırakmak için false olarak ayarlayın
      enabled: true,
    },
  },
}
```

- Sonuçlar değişirken koruma hiçbir zaman iptal etmez; yalnızca pencere boyunca
  bayt düzeyinde aynı olan sonuçlar korumayı tetikler.
- Yalnızca bir Compaction yeniden denemesinin hemen ardından devreye girer, çalıştırmanın
  diğer noktalarında değil.

<Note>
  Compaction sonrası koruma, hiç `tools.loopDetection` bloğu yazmamış olsanız bile ana bayrak açıkça `false` olmadığı sürece çalışır. Doğrulamak için bir Compaction olayından hemen sonra Gateway günlüğünde `post-compaction guard armed for N attempts` ifadesini arayın.
</Note>

## Günlükler ve beklenen davranış

Bir döngü algılandığında OpenClaw bir döngü olayı kaydeder ve önem derecesine bağlı olarak
sonraki araç çevrimini uyarır veya engeller; normal araç erişimini korurken kontrolsüz token
harcamasına ve kilitlenmelere karşı koruma sağlar.

- Önce uyarılar gelir.
- Bir örüntü uyarı eşiğini aşacak kadar sürdüğünde engelleme uygulanır.
- Kritik eşikler sonraki araç çevrimini engeller ve çalıştırma kaydında açık bir
  döngü algılama nedeni gösterir.
- Compaction sonrası koruma, soruna neden olan aracı ve aynı çağrı sayısını belirten
  `compaction_loop_persisted` hataları üretir.

## İlgili

<CardGroup cols={2}>
  <Card title="Exec onayları" href="/tr/tools/exec-approvals" icon="shield">
    Kabuk yürütmesi için izin verme/reddetme ilkesi.
  </Card>
  <Card title="Düşünme düzeyleri" href="/tr/tools/thinking" icon="brain">
    Akıl yürütme çabası düzeyleri ve sağlayıcı ilkesi etkileşimi.
  </Card>
  <Card title="Alt agent'lar" href="/tr/tools/subagents" icon="users">
    Kontrolden çıkan davranışı sınırlamak için yalıtılmış agent'lar oluşturma.
  </Card>
  <Card title="Yapılandırma başvurusu" href="/tr/gateway/config-tools#toolsloopdetection" icon="gear">
    Tam `tools.loopDetection` şeması ve birleştirme semantiği.
  </Card>
</CardGroup>
