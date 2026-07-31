---
read_when:
    - Grup sohbetlerini özel aracılara yönlendirirsiniz
    - Uzun süren tek bir görevin tüm sohbetleri engellemediği paralel çalışma istiyorsunuz
    - Çok aracılı bir operasyon düzeni tasarlıyorsunuz
sidebarTitle: Specialist lanes
status: active
summary: Paylaşılan model ve araç kapasitesini tıkamadan uzman ajanları paralel olarak çalıştırın
title: Paralel uzmanlık kanalları
x-i18n:
    generated_at: "2026-07-26T22:44:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09852b6cf5a790e98fb5e0805b0df57b2f3719b1387ecfacfb4973bb6841abb4
    source_path: concepts/parallel-specialist-lanes.md
    workflow: 16
---

Paralel uzman hatları, tek bir Gateway'in farklı sohbetleri veya odaları farklı
agent'lara yönlendirirken kullanıcı deneyimini hızlı tutmasını sağlar. Paralelliği
yalnızca "daha fazla agent" olarak değil, kıt kaynaklara yönelik bir tasarım problemi olarak ele alın.

## Temel ilkeler

Bir uzman hattı, yalnızca gerçek darboğazlardaki çekişmeyi azalttığında iş hacmini
iyileştirir:

- **Oturum kilitleri**: belirli bir oturumu aynı anda yalnızca bir çalıştırma değiştirmelidir.
- **Genel model kapasitesi**: görünür tüm sohbet çalıştırmaları yine sağlayıcı sınırlarını paylaşır.
- **Araç kapasitesi**: kabuk, tarayıcı, ağ ve depo çalışmaları model turunun
  kendisinden daha yavaş olabilir.
- **Bağlam bütçesi**: uzun dökümler gelecekteki her turu daha yavaş ve daha az
  odaklı hâle getirir.
- **Sahiplik belirsizliği**: aynı işi yapan yinelenen agent'lar kapasiteyi boşa harcar.

OpenClaw, çalıştırmaları zaten oturum başına seri hâle getirir ve genel paralelliği
[komut kuyruğu](/tr/concepts/queue) üzerinden sınırlar. Uzman hatları bunun üzerine
politika ekler: hangi işin hangi agent'a ait olduğu, nelerin sohbette kaldığı ve nelerin
arka plan çalışmasına dönüştüğü.

## Önerilen kullanıma alma planı

### Aşama 1: hat sözleşmeleri + arka planda ağır işler

Her hatta, çalışma alanında ve sistem isteminde yazılı bir sözleşme verin:

- **Amaç**: bu hattın sahip olduğu çalışma.
- **Hedef dışı işler**: girişimde bulunmak yerine devretmesi gereken çalışmalar.
- **Sohbet bütçesi**: hızlı yanıtlar sohbette kalır; uzun görevler kısaca kabul edilir,
  ardından bir arka plan alt agent'ında veya görevinde çalıştırılır.
- **Devir kuralı**: çalışma başka bir hatta ait olduğunda, nereye gitmesi gerektiğini belirtin ve
  kısa bir devir özeti sağlayın.
- **Araç riski kuralı**: işi yapabilecek en küçük araç yüzeyini tercih edin.

Bu, en düşük maliyetli aşamadır ve tıkanmaların çoğunu giderir: tek bir kodlama işi artık
araştırma hattını aşırı yavaşlatmaz ve her sohbet kendi bağlamını
temiz tutar.

### Aşama 2: öncelik ve eşzamanlılık denetimleri

Kuyruk ve model kapasitesini her hattın iş değerine göre ayarlayın:

```json5
{
  agents: {
    defaults: {
      maxConcurrent: 4,
      subagents: { maxConcurrent: 8, delegationMode: "prefer" },
    },
  },
  messages: {
    queue: {
      mode: "collect",
      debounceMs: 1000,
      cap: 20,
      drop: "summarize",
    },
  },
}
```

Doğrudan/kişisel sohbetleri ve üretim operasyonları agent'larını yüksek öncelikli işler için kullanın. Sistem
meşgul olduğunda araştırma, taslak hazırlama ve toplu kodlama çalışmalarının arka plan görevlerine
taşınmasını sağlayın.

### Aşama 3: koordinatör / trafik denetleyicisi

Birden fazla hat etkin olduğunda küçük bir koordinatör düzeni ekleyin:

- Etkin hat görevlerini ve sahiplerini takip edin.
- Gruplar arasındaki yinelenen istekleri tespit edin.
- Devir özetlerini hatlar arasında yönlendirin.
- Yalnızca engelleri, tamamlanan sonuçları ve insanın vermesi gereken kararları gösterin.

Buradan başlamayın. Hat sözleşmeleri olmayan bir koordinatör yalnızca kaosu koordine eder.

## Asgari hat sözleşmesi şablonu

```md
# Hat sözleşmesi

## Sahip olduğu işler

- <job this lane is responsible for>

## Sahip olmadığı işler

- <work to hand off>

## Sohbet bütçesi

- Hızlı soruları doğrudan yanıtlayın.
- Çok adımlı, yavaş veya yoğun araç kullanımı gerektiren çalışmalar için: kısaca kabul edin, çalışmayı
  bir alt agent'ta/arka planda başlatın ve tamamlandığında sonucu döndürün.

## Devir

İstek başka bir hatta aitse şunlarla yanıt verin:

- hedef hat
- amaç
- ilgili bağlam
- bir sonraki kesin eylem

## Araç yaklaşımı

Görevi tamamlayabilecek en küçük araç yüzeyini kullanın. Bu hat açıkça sahiplenmediği sürece
kapsamlı kabuk veya ağ çalışmalarından kaçının.
```

## İlgili konular

- [Çoklu agent yönlendirmesi](/tr/concepts/multi-agent)
- [Komut kuyruğu](/tr/concepts/queue)
- [Alt agent'lar](/tr/tools/subagents)
