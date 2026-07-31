---
read_when:
    - Kontrol Arayüzü Cihazlar sayfasındaki canlı durum sorunlarını ayıklama
    - Yinelenen veya güncelliğini yitirmiş örnek satırlarını araştırma
    - Gateway WS bağlantısını veya sistem olayı işaretlerini değiştirme
summary: OpenClaw iletişim durumu girdilerinin nasıl oluşturulduğu, birleştirildiği ve görüntülendiği
title: Durum
x-i18n:
    generated_at: "2026-07-26T23:16:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac5800eebddb82e69a7d0c06733e6a19addbc57be7776e7361411866af0c60f5
    source_path: concepts/presence.md
    workflow: 16
---

OpenClaw "presence", aşağıdakilerin hafif ve azami gayret esaslı bir görünümüdür:

- **Gateway**'in kendisi ve
- **Gateway'e bağlı, kullanıcı tarafından görülebilen istemciler** (Mac uygulaması, WebChat, Node'lar vb.)

Presence, canlı bağlantı meta verilerini Control UI **Cihazlar** sayfasında
(**Ayarlar → Cihazlar** altında) ve macOS uygulamasının **Örnekler** sekmesinde gösterir.

Bu sayfa, Gateway istemci listesini ele alır. En son kullandığınız Mac'i algılamak
ve Node uyarılarını oraya yönlendirmek için
[Etkin bilgisayar presence'ı](/tr/nodes/presence) bölümüne bakın.

## Presence alanları (görüntülenenler)

Presence girdileri, aşağıdakilere benzer alanlara sahip yapılandırılmış nesnelerdir:

- `instanceId` (isteğe bağlıdır ancak önemle önerilir): kararlı istemci kimliği (genellikle `connect.client.instanceId`)
- `host`: kolay anlaşılır ana makine adı
- `ip`: azami gayretle belirlenen IP adresi
- `version`: istemci sürümü dizesi
- `deviceFamily` / `modelIdentifier`: donanım ipuçları
- `mode`: `ui`, `webchat`, `cli`, `backend`, `node`, `probe`, `test`
- `lastInputSeconds`: biliniyorsa son kullanıcı girdisinden bu yana geçen saniye
- `reason`: istemcinin sağladığı serbest biçimli dize; Gateway'in kendisi yalnızca `self`, `connect` ve `disconnect` değerlerini yayınlar
- `deviceId`, `roles`, `scopes`: bağlantı el sıkışmasından alınan cihaz kimliği ve rol/kapsam ipuçları
- `ts`: son güncelleme zaman damgası (epoktan bu yana ms)

## Üreticiler (presence'ın kaynakları)

Presence girdileri birden fazla kaynak tarafından üretilir ve **birleştirilir**.

### 1) Gateway'in kendi girdisi

Gateway, henüz hiçbir istemci bağlanmadan önce bile kullanıcı arayüzlerinde Gateway ana makinesinin
gösterilmesi için başlangıçta her zaman bir "kendi" girdisi oluşturur.

### 2) WebSocket bağlantısı

Her WS istemcisi bir `connect` isteğiyle başlar. El sıkışma başarılı olduğunda
Gateway, bu bağlantının presence girdisini ekler veya günceller.

#### Geçici kontrol düzlemi bağlantıları neden görünmez?

CLI komutları, arka uç RPC istemcileri ve yoklamalar genellikle kısa süreliğine bağlanır. Bu
değişkenliği presence TTL süresinin tamamı boyunca tutmamak için `cli`, `backend`
veya `probe` modundaki istemciler presence girdilerine **dönüştürülmez**. Test modu istemcileri,
test paketleri onları gerçek istemcilerin yerine kullandığı için izlenmeye devam eder.

### 3) `system-event` sinyalleri

İstemciler, `system-event` yöntemiyle daha zengin periyodik sinyaller gönderebilir. Mac
uygulaması bunu ana makine adını, IP'yi, sürümü ve canlılık meta verilerini bildirmek için kullanır. Fiziksel
girdi etkinliği bu genel sinyalin parçası değildir; bunun sorumluluğu [Etkin bilgisayar presence'ı](/tr/nodes/presence)
bölümünde açıklanan amaca özel yerel Node olayındadır. Mac, bu sinyalleri
`system-presence-clear-last-input` ile etiketler; güncel Gateway'ler, eski bir uygulamadan
kalan girdi güncelliği verilerini kaldırmak için bu geriye dönük uyumlu işareti kullanır. Sinyal ayrıca
30 günlük sabit bir değer taşır; böylece etiketi yok sayan eski Gateway'ler tam güncellik verisini
korumak yerine üzerine yazar. Bu uyumluluk değeri için yeni etkinlik örneklenmez.

### 4) Node bağlantıları (rol: Node)

Bir Node, `role: node` ile Gateway WebSocket üzerinden bağlandığında Gateway,
o Node için bir presence girdisi ekler veya günceller (diğer WS istemcileriyle aynı akış).

## Birleştirme ve yinelenenleri kaldırma kuralları (`instanceId` neden önemlidir?)

Presence girdileri, sırasıyla ilk kullanılabilir olan eşleştirilmiş cihaz kimliği,
`connect.client.instanceId` veya son çare olarak bağlantı başına kimlik temelinde, büyük/küçük harfe
duyarsız anahtarlarla tek bir bellek içi haritada saklanır.

Geçici kontrol düzlemi istemcileri izleme dışında bırakılır (yukarıya bakın);
bu nedenle bağlantı kimlikleri hiçbir zaman anahtar hâline gelmez. Diğer tüm istemcilerde
bağlantı kimliği yedeği, kararlı bir `instanceId` olmadan yeniden bağlanan istemcinin
**yinelenen** bir satır olarak görünmesine neden olur.

## TTL ve sınırlı boyut

Presence kasıtlı olarak geçicidir:

- **TTL:** 5 dakikadan eski girdiler kaldırılır
- **Azami girdi:** 200 (önce en eskiler kaldırılır)

Bu, listeyi güncel tutar ve sınırsız bellek büyümesini önler.

## Uzak bağlantı/tünel uyarısı (geri döngü IP'leri)

Bir istemci SSH tüneli/yerel bağlantı noktası yönlendirmesi üzerinden bağlandığında Gateway,
uzak adresi `127.0.0.1` olarak görebilir. Bu tünel adresinin istemcinin IP'si olarak
kaydedilmesini önlemek için bağlantı işleme, geri döngü adresini girdiye yazmak yerine
yerel olduğu algılanan (geri döngü) istemcilerde `ip` alanını tamamen atlar.

## Tüketiciler

### Control UI Cihazlar sayfası

**Cihazlar** sayfası, `system-presence` verilerini kalıcı eşleştirme ve Node
kayıtlarıyla birleştirir. Gateway'in kendi sinyalini ilk sıraya sabitler ve canlı platform,
sürüm, model ve girdi güncelliği meta verileri için eşleşen cihaz veya örnek kimliklerini kullanır.

### macOS Örnekler sekmesi

macOS uygulaması, `system-presence` çıktısını gösterir ve son güncellemenin yaşına göre
küçük bir durum göstergesi (Etkin/Boşta/Eski) uygular.

## Hata ayıklama ipuçları

- Ham listeyi görmek için Gateway'de `system-presence` çağrısı yapın.
- Yinelenen girdiler görürseniz:
  - istemcilerin el sıkışmada kararlı bir `client.instanceId` gönderdiğini doğrulayın
  - periyodik sinyallerin aynı `instanceId` değerini kullandığını doğrulayın
  - bağlantıdan türetilen girdide `instanceId` alanının eksik olup olmadığını kontrol edin (yinelenen girdiler beklenir)

## İlgili

<CardGroup cols={2}>
  <Card title="Etkin bilgisayar presence'ı" href="/tr/nodes/presence" icon="computer-mouse">
    Fiziksel Mac girdisinin etkin bir Node'u nasıl seçtiği ve bağlantı uyarılarını nasıl yönlendirdiği.
  </Card>
  <Card title="Yazma göstergeleri" href="/tr/concepts/typing-indicators" icon="ellipsis">
    Yazma göstergelerinin ne zaman gönderildiği ve nasıl ayarlanacağı.
  </Card>
  <Card title="Akış ve parçalara ayırma" href="/tr/concepts/streaming" icon="bars-staggered">
    Giden akış, parçalara ayırma ve kanal başına biçimlendirme.
  </Card>
  <Card title="Gateway mimarisi" href="/tr/concepts/architecture" icon="diagram-project">
    Gateway bileşenleri ve presence güncellemelerini yöneten WebSocket protokolü.
  </Card>
  <Card title="Gateway protokolü" href="/tr/gateway/protocol" icon="plug">
    `connect`, `system-event` ve `system-presence` için kablo protokolü.
  </Card>
</CardGroup>
