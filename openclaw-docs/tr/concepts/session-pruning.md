---
read_when:
    - Araç çıktılarından kaynaklanan bağlam büyümesini azaltmak istiyorsunuz
    - Anthropic istem önbelleği optimizasyonunu anlamak istiyorsunuz
summary: Bağlamı sade ve önbelleğe almayı verimli tutmak için eski araç sonuçlarını kırpma
title: Oturum budama
x-i18n:
    generated_at: "2026-07-26T22:44:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dd5cb4582cb8d9d7265213abe1f5b5893634882b9f8b3ce1deef746293dd07db
    source_path: concepts/session-pruning.md
    workflow: 16
---

Oturum budama, her LLM çağrısından önce **eski araç sonuçlarını** bağlamdan kırpar. Normal konuşma metnini yeniden yazmadan birikmiş araç çıktılarının (yürütme sonuçları, dosya okumaları, arama sonuçları) yol açtığı bağlam şişmesini azaltır.

<Info>
Budama yalnızca bellekte gerçekleşir; diskteki oturum dökümünü değiştirmez. Geçmişinizin tamamı her zaman korunur.
</Info>

## Neden önemlidir?

Uzun oturumlarda biriken araç çıktıları bağlam penceresini şişirir. Bu, maliyeti artırır ve [Compaction](/tr/concepts/compaction) işlemini gerekenden daha erken zorunlu kılabilir.

Budama, özellikle **Anthropic istem önbelleğe alma** için değerlidir. Önbellek TTL süresi dolduktan sonra bir sonraki istek, istemin tamamını yeniden önbelleğe alır. Budama, önbelleğe yazma boyutunu azaltarak maliyeti doğrudan düşürür.

## Nasıl çalışır?

Budama, hem zaman hem de bağlam boyutu denetimine bağlı olarak `cache-ttl` modunda çalışır:

1. Önbellek TTL süresinin dolmasını bekleyin (elle ayarlandığında varsayılan 5 dakikadır; Anthropic otomatik varsayılanı için [Akıllı varsayılanlar](#smart-defaults) bölümüne bakın). TTL dolmadan önce, yakın zamanlı dönüşlerde istem önbelleğinin yeniden kullanılmasını korumak için budama tamamen atlanır.
2. TTL dolduktan sonra toplam bağlam boyutunu modelin bağlam penceresine göre tahmin edin. Oran `softTrimRatio` değerinin (varsayılan 0.3) altındaysa budamayı atlayın ve TTL saatini çalışır durumda tutun.
3. Oranın üzerindeki büyük boyutlu araç sonuçlarını **yumuşak kırpın**: başlangıcı ve sonu koruyun (varsayılan olarak her biri 1500 karakter, toplamda en fazla 4000 karakter) ve araya `...` ekleyin.
4. Oran hâlâ `hardClearRatio` değerinde veya üzerindeyse (varsayılan 0.5) ve budanabilir araç içeriğinden en az `minPrunableToolChars` (varsayılan 50.000) kaldıysa bu sonuçları **tamamen temizleyin**: içeriklerini bir yer tutucuyla değiştirin (varsayılan `[Old tool result content cleared]`).
5. TTL saatini yalnızca budama bağlamı gerçekten değiştirdiğinde sıfırlayın; böylece sonraki istekler yeni önbelleği yeniden kullanır.

Eşiklerden bağımsız olarak iki güvenlik kuralı uygulanır: en son `keepLastAssistants` asistan dönüşü (varsayılan 3) hiçbir zaman budanmaz ve oturumun ilk kullanıcı mesajından önceki hiçbir şey budanmaz (`SOUL.md`/`USER.md` gibi önyükleme okumalarını korur).

Yalnızca `toolResult` mesajları uygundur; normal konuşma metnine dokunulmaz. Hangi araç adlarının budanabilir olduğunu sınırlamak için `agents.defaults.contextPruning.tools.{allow,deny}` kullanın.

## Eski görüntüleri temizleme

OpenClaw ayrıca geçmişte ham görüntü bloklarını veya istem hazırlama medya işaretçilerini kalıcı olarak tutan oturumlar için ayrı, birden çok kez güvenle uygulanabilen bir yeniden oynatma görünümü oluşturur.

- Yakın zamanlı takip isteklerine ait istem önbelleği öneklerinin kararlı kalması için **tamamlanan en son 3 dönüşü** bayt bayt korur. Bu sayı yalnızca görüntü içerenleri değil, tamamlanan tüm dönüşleri kapsar; bu nedenle yalnızca metin içeren dönüşler de pencereyi tüketir.
- Yeniden oynatma görünümünde, `user` veya `toolResult` geçmişindeki daha eski ve önceden işlenmiş görüntü blokları `[image data removed - already processed by model]` ile değiştirilir.
- `[media attached: ...]`, `[Image: source: ...]` ve `media://inbound/...` gibi daha eski metinsel medya başvuruları `[media reference removed - already processed by model]` ile değiştirilir. Mevcut dönüşün ek işaretçileri olduğu gibi kalır; böylece görsel modeller yeni görüntüleri hazırlamaya devam edebilir.
- Ham oturum dökümü yeniden yazılmaz; böylece geçmiş görüntüleyicileri özgün mesaj girdilerini ve görüntülerini göstermeye devam edebilir.
- Bu, yukarıdaki normal önbellek TTL budamasından ayrıdır. Yinelenen görüntü yüklerinin veya eski medya başvurularının sonraki dönüşlerde istem önbelleklerini bozmasını önlemek için kullanılır.

## Akıllı varsayılanlar

Paketle birlikte gelen Anthropic Plugin, bir Anthropic (veya Claude CLI) kimlik doğrulama profilini ilk kez çözümlerken budama ve Heartbeat sıklığını otomatik olarak yapılandırır; ancak bunu yalnızca daha önce açıkça ayarlamadığınız alanlar için yapar:

| Kimlik doğrulama modu                    | `contextPruning.mode` | `contextPruning.ttl` | `heartbeat.every` |
| ---------------------------------------- | --------------------- | -------------------- | ----------------- |
| OAuth/token (Claude CLI yeniden kullanımı dâhil) | `cache-ttl`           | `1h`                 | `1h`              |
| API anahtarı                             | `cache-ttl`           | `1h`                 | `30m`             |

`agents.defaults.contextPruning.mode` veya `agents.defaults.heartbeat.every` değerini kendiniz ayarlarsanız OpenClaw bunları geçersiz kılmaz. Bu otomatik varsayılan yalnızca Anthropic ailesi kimlik doğrulaması için devreye girer; yapılandırmadığınız sürece diğer sağlayıcılarda budama `off` olur.

## Etkinleştirme veya devre dışı bırakma

Budama, Anthropic dışındaki sağlayıcılarda varsayılan olarak kapalıdır. Etkinleştirmek için:

```json5
{
  agents: {
    defaults: {
      contextPruning: { mode: "cache-ttl", ttl: "5m" },
    },
  },
}
```

Devre dışı bırakmak için `mode: "off"` değerini ayarlayın.

## Budama ve Compaction karşılaştırması

|            | Budama             | Compaction              |
| ---------- | ------------------ | ----------------------- |
| **Ne yapar?**   | Araç sonuçlarını kırpar | Konuşmayı özetler |
| **Kaydedilir mi?** | Hayır (istek başına) | Evet (dökümde)     |
| **Kapsam**  | Yalnızca araç sonuçları | Tüm konuşma     |

Birbirlerini tamamlarlar; budama, Compaction döngüleri arasında araç çıktısını sade tutar.

## Ek okumalar

- [Compaction](/tr/concepts/compaction): özetlemeye dayalı bağlam azaltma
- [Gateway Yapılandırması](/tr/gateway/configuration): tüm budama yapılandırma seçenekleri (`contextPruning.*`)

## İlgili konular

- [Oturum yönetimi](/tr/concepts/session)
- [Oturum araçları](/tr/concepts/session-tool)
- [Bağlam motoru](/tr/concepts/context-engine)
