---
read_when:
    - Yazıyor göstergesi davranışını veya varsayılanlarını değiştirme
summary: OpenClaw'un yazıyor göstergelerini ne zaman gösterdiği ve bunların nasıl ayarlanacağı
title: Yazıyor göstergeleri
x-i18n:
    generated_at: "2026-07-26T23:39:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3c66d61ea7e3e809b8e88ae2eabb9794f0886b629094753716ed02912843ffc
    source_path: concepts/typing-indicators.md
    workflow: 16
---

Yazma göstergeleri, bir çalıştırma etkin olduğu sürece sohbet kanalına gönderilir. Yazmanın **ne zaman** başlayacağını denetlemek için `agents.defaults.typingMode`, yenilemenin **ne sıklıkta** yapılacağını denetlemek için `typingIntervalSeconds` kullanın (canlı tutma sıklığı, varsayılan 6 saniye).

## Varsayılanlar

`agents.defaults.typingMode` **ayarlanmamışsa**:

- **Doğrudan sohbetler**: Model döngüsü başladığı anda yazma göstergesi başlar.
- **Bahsetme içeren grup sohbetleri**: Yazma göstergesi hemen başlar.
- **Bahsetme içermeyen grup sohbetleri**: Yazma göstergesi, kabul edilen çalıştırmada yürütme altyapısı etkinliği veya mesaj metni gibi kullanıcı tarafından görülebilen bir etkinlik olduğunda başlar.
- **Heartbeat çalıştırmaları**: Çözümlenen Heartbeat hedefi yazma göstergesini destekleyen bir sohbetse ve yazma göstergesi devre dışı bırakılmamışsa, Heartbeat çalıştırması başladığında yazma göstergesi başlar.

## Modlar

`agents.defaults.typingMode` değerini şunlardan birine ayarlayın:

- `never` - hiçbir zaman yazma göstergesi gösterilmez.
- `instant` - çalıştırma daha sonra yalnızca sessiz yanıt belirtecini döndürse bile yazma göstergesi **model döngüsü başlar başlamaz** başlatılır.
- `thinking` - yazma göstergesi **ilk akıl yürütme deltasıyla** veya tur kabul edildikten sonraki etkin yürütme altyapısı çalışmasıyla başlatılır.
- `message` - yazma göstergesi, etkin yürütme altyapısı çalışması veya sessiz olmayan bir metin deltası gibi **kullanıcı tarafından görülebilen ilk yanıt etkinliğiyle** başlatılır. `NO_REPLY` gibi sessiz yanıt belirteçleri metin etkinliği sayılmaz.

“Ne kadar erken tetiklendiği” sırası: `never` -> `message`/`thinking` -> `instant`.

## Yapılandırma

Aracı düzeyindeki varsayılanı ayarlayın:

```json5
{
  agents: {
    defaults: {
      typingMode: "thinking",
      typingIntervalSeconds: 6,
    },
  },
}
```

Tek bir aracı için ilkeyi geçersiz kılın:

```json5
{
  agents: {
    entries: {
      support: {
        typingMode: "message",
      },
    },
  },
}
```

## Notlar

- `message` modu sessiz yanıt belirteçleriyle başlamaz, ancak etkin yürütme herhangi bir asistan metni kullanılabilir olmadan önce de yazma göstergesini gösterebilir.
- `thinking`, akışla iletilen akıl yürütmeye (`reasoningLevel: "stream"`) yine tepki verir ve akıl yürütme deltaları gelmeden önce etkin yürütmeyle de başlayabilir.
- Heartbeat yazma göstergesi, çözümlenen teslimat hedefi için bir canlılık sinyalidir. `message` veya `thinking` akış zamanlamasını izlemek yerine Heartbeat çalıştırması başladığında başlar. Devre dışı bırakmak için `typingMode: "never"` ayarını kullanın.
- Heartbeat hedefi `"none"` olduğunda, hedef çözümlenemediğinde, Heartbeat için sohbet teslimatı devre dışı bırakıldığında veya kanal yazma göstergesini desteklemediğinde Heartbeat'ler yazma göstergesi göstermez.
- `agents.defaults.typingIntervalSeconds`, başlangıç zamanını değil, her aracı için **yenileme sıklığını** denetler. Varsayılan: 6 saniye.

## İlgili

<CardGroup cols={2}>
  <Card title="Varlık" href="/tr/concepts/presence" icon="signal">
    Gateway'in, Kontrol Arayüzü Cihazlar sayfası ve macOS Örnekleri sekmesi için bağlı istemcileri nasıl izlediği.
  </Card>
  <Card title="Akış ve parçalara ayırma" href="/tr/concepts/streaming" icon="bars-staggered">
    Giden akış davranışı, parça sınırları ve kanala özgü teslimat.
  </Card>
</CardGroup>
