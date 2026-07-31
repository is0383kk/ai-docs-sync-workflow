---
read_when:
    - Yeni bir çekirdek yeteneği ve Plugin kayıt yüzeyi ekleme
    - Kodun çekirdeğe mi, bir sağlayıcı pluginine mi yoksa bir özellik pluginine mi ait olduğuna karar verme
    - Kanallar veya araçlar için yeni bir çalışma zamanı yardımcısını bağlama
sidebarTitle: Adding capabilities
summary: OpenClaw Plugin sistemine yeni bir ortak yetenek eklemeye yönelik katkıda bulunanlar kılavuzu
title: Yetenek ekleme (katkıda bulunanlar kılavuzu)
x-i18n:
    generated_at: "2026-07-26T23:26:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 14f86c98eb10c6e92970d1b65009ac7bb103afcb6bc57bad2c39e59bc038c961
    source_path: plugins/adding-capabilities.md
    workflow: 16
---

<Info>
  Bu, OpenClaw çekirdek geliştiricileri için bir **katkıda bulunanlar kılavuzudur**. Harici bir plugin
  geliştiriyorsanız bunun yerine [Plugin geliştirme](/tr/plugins/building-plugins)
  bölümüne bakın. Ayrıntılı mimari başvurusu (yetenek modeli, sahiplik,
  yükleme işlem hattı, çalışma zamanı yardımcıları) için [Plugin iç yapısı](/tr/plugins/architecture) bölümüne bakın.
</Info>

OpenClaw; gömme, görüntü oluşturma, video oluşturma veya gelecekte
tedarikçi destekli başka bir özellik alanı gibi yeni bir paylaşılan etki alanına ihtiyaç duyduğunda bunu kullanın.

Kural:

- **plugin** = sahiplik sınırı
- **yetenek** = paylaşılan çekirdek sözleşmesi

Bir tedarikçiyi doğrudan bir kanala veya araca bağlamayın. Önce yeteneği tanımlayın.

## Ne zaman yetenek oluşturulmalı?

Yalnızca aşağıdakilerin **tümü** geçerliyse yeni bir yetenek oluşturun:

1. Birden fazla tedarikçinin bunu uygulaması makul ölçüde mümkün olmalıdır.
2. Kanallar, araçlar veya özellik pluginleri, tedarikçiyi önemsemeden bunu kullanabilmelidir.
3. Çekirdeğin geri dönüş, politika, yapılandırma veya teslim davranışını sahiplenmesi gerekir.

Çalışma yalnızca tedarikçiye özgüyse ve henüz paylaşılan bir sözleşme yoksa önce sözleşmeyi tanımlayın.

## Standart sıra

1. Türü belirlenmiş çekirdek sözleşmesini tanımlayın.
2. Bu sözleşme için plugin kaydını ekleyin.
3. Paylaşılan bir çalışma zamanı yardımcısı ekleyin.
4. Kanıt olarak gerçek bir tedarikçi pluginini bağlayın.
5. Özellik/kanal tüketicilerini çalışma zamanı yardımcısına taşıyın.
6. Sözleşme testleri ekleyin.
7. Operatöre yönelik yapılandırmayı ve sahiplik modelini belgelendirin.

## Ne nereye yerleştirilir?

| Katman                     | Sahip oldukları                                                                                                                                                                                                                         |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Çekirdek**               | İstek/yanıt türleri; sağlayıcı kayıt defteri ve çözümlemesi; geri dönüş davranışı; iç içe nesne, joker karakter, dizi öğesi ve bileşim düğümlerinde yayılan `title`/`description` belge meta verilerini içeren yapılandırma şeması; çalışma zamanı yardımcı yüzeyi. |
| **Tedarikçi plugini**      | Tedarikçi API çağrıları, tedarikçi kimlik doğrulama işlemleri, tedarikçiye özgü istek normalleştirmesi ve yetenek uygulamasının kaydı.                                                                                                    |
| **Özellik/kanal plugini**  | `api.runtime.*` veya eşleşen `plugin-sdk/*-runtime` yardımcısını çağırır. Bir tedarikçi uygulamasını hiçbir zaman doğrudan çağırmaz.                                                                                                      |

## Sağlayıcı ve çalıştırma düzeneği bağlantı noktaları

Davranış genel ajan döngüsünden ziyade model sağlayıcı sözleşmesine ait olduğunda **sağlayıcı kancalarını** kullanın. Örnekler arasında aktarım seçiminden sonraki sağlayıcıya özgü istek parametreleri, kimlik doğrulama profili tercihi, istem katmanları ve model/profil yük devretmesinden sonraki takip geri dönüşü yönlendirmesi bulunur.

Davranış, bir dönüşü yürüten çalışma zamanına ait olduğunda **ajan çalıştırma düzeneği kancalarını** kullanın. Çalıştırma düzenekleri; boş çıktı, görünür çıktı olmadan akıl yürütme veya nihai yanıtı olmayan yapılandırılmış bir plan gibi açık protokol sonuçlarını sınıflandırabilir; böylece dış modelin geri dönüş politikası yeniden deneme kararını verebilir.

Her iki bağlantı noktasını da dar kapsamlı tutun:

- Yeniden deneme/geri dönüş politikasının sahibi çekirdektir.
- Sağlayıcı pluginleri, sağlayıcıya özgü istek/kimlik doğrulama/yönlendirme ipuçlarını sahiplenir.
- Çalıştırma düzeneği pluginleri, çalışma zamanına özgü deneme sınıflandırmasını sahiplenir.
- Üçüncü taraf pluginleri, çekirdek durumunu doğrudan değiştirmek yerine ipuçları döndürür.

## Dosya kontrol listesi

Yeni bir yetenek için şu alanlara dokunmanız beklenir:

- `src/<capability>/types.ts`
- `src/<capability>/...registry/runtime.ts`
- `src/plugins/types.ts`
- `src/plugins/registry.ts`
- `src/plugins/captured-registration.ts`
- `src/plugins/contracts/registry.ts`
- `src/plugins/runtime/types-core.ts`
- `src/plugins/runtime/index.ts`
- `src/plugin-sdk/<capability>.ts`
- `src/plugin-sdk/<capability>-runtime.ts`
- Bir veya daha fazla paketlenmiş plugin paketi.
- Yapılandırma, belgeler, testler.

## Uygulamalı örnek: görüntü oluşturma

Görüntü oluşturma standart yapıyı izler:

1. Çekirdek, `ImageGenerationProvider` öğesini tanımlar.
2. Çekirdek, `registerImageGenerationProvider(...)` öğesini kullanıma sunar.
3. Çekirdek, `api.runtime.imageGeneration.generate(...)` ve `.listProviders(...)` öğelerini kullanıma sunar.
4. Tedarikçi pluginleri (`comfy`, `deepinfra`, `fal`, `google`, `litellm`, `microsoft-foundry`, `minimax`, `openai`, `openrouter`, `vydra`, `xai`) tedarikçi destekli uygulamaları kaydeder.
5. Gelecekteki tedarikçiler, kanalları/araçları değiştirmeden aynı sözleşmeyi kaydeder.

Yapılandırma anahtarı, görsel analiz yönlendirmesinden bilinçli olarak ayrıdır:

- `agents.defaults.imageModel` görüntüleri analiz eder.
- `agents.defaults.mediaModels.image` görüntüler oluşturur.

Geri dönüşün ve politikanın açık kalması için bunları ayrı tutun.

## Gömme sağlayıcıları

Yeniden kullanılabilir vektör gömme sağlayıcıları için `registerEmbeddingProvider(...)` / `embeddingProviders`
sözleşmesini kullanın. Bu sözleşme, bilinçli olarak bellekten daha geniş kapsamlıdır:
araçlar, arama, erişim, içe aktarıcılar veya gelecekteki özellik pluginleri
bellek motoruna bağımlı olmadan gömmeleri kullanabilir. Bellek araması
da genel `embeddingProviders` öğesini kullanır.

Eski belleğe özgü kayıt API'si ve `memoryEmbeddingProviders`
sözleşmesi kullanımdan kaldırılmıştır. Tüm yeni gömme sağlayıcıları için
`registerEmbeddingProvider` ve `embeddingProviders` kullanın.

## İnceleme kontrol listesi

Yeni bir yeteneği yayımlamadan önce şunları doğrulayın:

- Hiçbir kanal/araç, tedarikçi kodunu doğrudan içe aktarmaz.
- Paylaşılan yol çalışma zamanı yardımcısıdır.
- En az bir sözleşme testi, paketlenmiş sahipliği doğrular.
- Yapılandırma belgeleri yeni model/yapılandırma anahtarını adlandırır.
- Plugin belgeleri sahiplik sınırını açıklar.

Bir PR yetenek katmanını atlayıp tedarikçi davranışını bir kanala/araca sabit kodluyorsa geri gönderin ve önce sözleşmeyi tanımlayın.

## İlgili

- [Plugin iç yapısı](/tr/plugins/architecture) — yetenek modeli, sahiplik, yükleme işlem hattı, çalışma zamanı yardımcıları.
- [Plugin geliştirme](/tr/plugins/building-plugins) — ilk plugin öğreticisi.
- [SDK'ya genel bakış](/tr/plugins/sdk-overview) — içe aktarma eşlemesi ve kayıt API'si başvurusu.
- [Skills oluşturma](/tr/tools/creating-skills) — tamamlayıcı katkıda bulunan yüzeyi.
