---
read_when:
    - Eksik operatör kapsamı hatalarında hata ayıklama
    - Cihaz veya Node eşleştirme onaylarını inceleme
    - Gateway RPC yöntemlerini ekleme veya sınıflandırma
summary: Gateway istemcileri için operatör rolleri, kapsamları ve onay zamanı denetimleri
title: Operatör kapsamları
x-i18n:
    generated_at: "2026-07-26T23:20:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 40053793bb5a80afab28fdfcdcac6565abde6bca988389b03a407272c70043e2
    source_path: gateway/operator-scopes.md
    workflow: 16
---

Gateway operatör kapsamları, bir Gateway istemcisinin kimlik doğrulamasından sonra neler yapabileceğini sınırlar.
Bunlar, güvenilen tek bir Gateway operatör alanı içindeki kontrol düzlemi güvenlik sınırıdır;
kötü niyetli çok kiracılı ortamlar için yalıtım sağlamaz. Kişiler, ekipler veya makineler arasında
güçlü ayrım sağlamak için farklı işletim sistemi kullanıcıları ya da ana makineler altında ayrı Gateway'ler çalıştırın.

İlgili: [Güvenlik](/tr/gateway/security), [Gateway protokolü](/tr/gateway/protocol),
[Gateway eşleştirme](/tr/gateway/pairing), [Cihazlar CLI'sı](/tr/cli/devices).

## Roller

Her Gateway WebSocket istemcisi tek bir rolle bağlanır:

- `operator`: CLI, Control UI, otomasyon ve
  güvenilen yardımcı işlemler gibi kontrol düzlemi istemcileri.
- `node`: komutları `node.invoke` üzerinden sunan
  yetenek ana makineleri (macOS, iOS, Android, başsız).

Operatör RPC yöntemleri `operator` rolünü; node kaynaklı yöntemler ise
`node` rolünü gerektirir.

## Kapsam düzeyleri

| Kapsam                  | Anlamı                                                                                                                                                        |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operator.read`         | Salt okunur durum, listeler, katalog, günlükler, oturum okumaları ve değişiklik yapmayan diğer çağrılar.                                                       |
| `operator.write`        | Değişiklik yapan operatör eylemleri: mesaj gönderme, araç çağırma, konuşma/ses ayarlarını güncelleme, node komutu aktarma. `operator.read` kapsamını da karşılar. |
| `operator.admin`        | Yönetici erişimi. Tüm `operator.*` kapsamlarını karşılar. Yapılandırma değişiklikleri, güncellemeler, yerel kancalar, ayrılmış ad alanları ve yüksek riskli onaylar için gereklidir. |
| `operator.pairing`      | Cihaz ve node eşleştirme yönetimi: listeleme, onaylama, reddetme, kaldırma, döndürme, iptal etme.                                                               |
| `operator.approvals`    | Çalıştırma ve Plugin onay API'leri.                                                                                                                           |
| `operator.questions`    | Etkileşimli soruları listeleme, okuma, yanıtlama ve çözümleme.                                                                                                |
| `operator.talk.secrets` | Gizli bilgiler dâhil olmak üzere Talk yapılandırmasını okuma.                                                                                                 |

Gelecekteki bilinmeyen `operator.*` kapsamları, çağıran taraf zaten
`operator.admin` kapsamına sahip değilse tam eşleşme gerektirir.

## Yöntem kapsamı yalnızca ilk denetimdir

Her Gateway RPC'si, bir isteğin işleyicisine ulaşıp ulaşmayacağını belirleyen
en az ayrıcalıklı bir yöntem kapsamına sahiptir. Parametrelere duyarlı yöntemler,
yetkilendirme hatalarının tek bir standart yapılandırılmış yanıtı olması için bu kapsamı
yönlendirmeden önce türetir:

- `agent`, sıradan etkileşimler için `operator.write`; `/new`
  veya `/reset` oturum yaşam döngüsü komutları için
  `operator.admin` gerektirir.
- `node.invoke`, sıradan aktarma komutları için `operator.write`;
  `browser.proxy`, `fs.listDir` ve `terminal.upload` için
  `operator.admin` gerektirir.
- `talk.config`, `operator.read` gerektirir; `includeSecrets: true`
  ayrıca `operator.talk.secrets` gerektirir.

Bazı işleyiciler daha sonra onaylanan veya değiştirilen somut öğeye göre daha
sıkı denetimler uygular:

- `device.pair.approve` yöntemine `operator.pairing` ile erişilebilir; ancak bir
  operatör cihazının onaylanması yalnızca çağıran tarafın zaten sahip olduğu kapsamları oluşturabilir veya koruyabilir.
- `node.pair.approve` yöntemine `operator.pairing` ile erişilebilir; ardından bekleyen
  node'un bildirdiği komut listesinden ek onay kapsamları türetilir.
- `chat.send`, yazma kapsamlı bir yöntemdir; ancak `/config set` ve
  `/config unset` sohbet komutları, çağıran tarafın sohbet gönderme kapsamından bağımsız olarak
  buna ek olarak `operator.admin` gerektirir.

Bu, daha düşük kapsamlı operatörlerin tüm eşleştirme onaylarını yalnızca
yöneticiye özel hâle getirmeden düşük riskli eşleştirme eylemleri gerçekleştirmesini sağlar.

Oturum değiştirme RPC'leri, bağlanan istemcinin `client.id` veya
`client.mode` değerinden bağımsız olarak, üzerinde anlaşılmış operatör kapsamlarıyla yetkilendirilir.
İstemci kimliği bağlantı ve cihaz kimlik doğrulama politikasını yine etkileyebilir;
ancak oturum değiştirme yetkisi vermez veya bu yetkiyi kaldırmaz.

## Cihaz eşleştirme onayları

Cihaz eşleştirme kayıtları, onaylanmış rollerin ve kapsamların kalıcı kaynağıdır.
Önceden eşleştirilmiş bir cihaz sessizce daha geniş erişim elde etmez: daha geniş bir rol
veya daha geniş kapsamlar isteyen yeniden bağlantı, bekleyen yeni bir yükseltme isteği oluşturur.

Bir cihaz isteğini onaylama:

- Operatör rolü olmayan bir istek, operatör kapsamı onayı gerektirmez.
- Operatör dışı bir cihaz rolü isteği (örneğin `node`),
  `device.pair.approve` yönteminin kendisi yalnızca `operator.pairing`
  gerektirse de `operator.admin` gerektirir.
- `operator.read`, `operator.write`, `operator.approvals`,
  `operator.questions`, `operator.pairing` veya `operator.talk.secrets` isteği,
  çağıran tarafın bu kapsama veya `operator.admin` kapsamına zaten sahip olmasını gerektirir.
- `operator.admin` isteği, `operator.admin` gerektirir.
- Açık kapsamları olmayan bir onarım isteği, mevcut operatör
  token'ının kapsamlarını devralabilir; bu token yönetici kapsamlıysa onay yine
  `operator.admin` gerektirir.

Yönetici olmayan paylaşılan gizli bilgi ve güvenilen proxy oturumları, operatör
cihazı isteklerini yalnızca kendi bildirdikleri operatör kapsamları dâhilinde onaylayabilir;
bu oturumlar aksi hâlde `operator.pairing` kullanabilse bile operatör dışı rollerin
onaylanması yalnızca yöneticiye açıktır.

Eşleştirilmiş cihaz token'ı oturumlarında, çağıran taraf `operator.admin` kapsamına
sahip değilse yönetim kendi cihazıyla sınırlıdır: yönetici olmayan çağıran taraf yalnızca
kendi eşleştirme girdilerini görür ve yalnızca kendi cihaz girdisini onaylayabilir, reddedebilir,
döndürebilir, iptal edebilir veya kaldırabilir.

## Node eşleştirme onayları

Eski `node.pair.*` yöntemleri, Gateway'in sahip olduğu ayrı bir node eşleştirme deposu kullanır.
WS node'ları bunun yerine cihaz eşleştirmeyi (`role: node`) kullanır; ancak aynı onay
terminolojisi geçerlidir. İki deponun ilişkisi için [Gateway eşleştirme](/tr/gateway/pairing) bölümüne bakın.

`node.pair.approve`, bekleyen isteğin komut listesinden gerekli ek kapsamları türetir:

| Bildirilen komutlar                                                                                                  | Gerekli kapsamlar                       |
| -------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| yok                                                                                                                  | `operator.pairing`                     |
| sıradan node komutları                                                                                               | `operator.pairing` + `operator.write` |
| `system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `fs.listDir` veya `system.execApprovals.get/set` | `operator.pairing` + `operator.admin` |

Bir node bildiriminin onaylanması, ayrı bir çalışma zamanı izin listesi denetimine
sahip komutları etkinleştirmez. Örneğin, `computer.act` bildiren bir node'un
onaylanması eşleştirme ve yazma kapsamı gerektirir, ancak yalnızca yüzeyi kaydeder.
Bir yönetici veya sahip yine de `computer.act` özelliğini etkinleştirmelidir. Etkin
kaldığı sürece, `node.invoke` üzerinden çağrılması yazma kapsamı gerektirir;
ancak her eylem için yönetici kapsamı gerektirmez.

Node eşleştirme, kimlik ve güven oluşturur; node'un kendi
`system.run` çalıştırma onayı politikasının yerini almaz.

## Paylaşılan gizli bilgiyle kimlik doğrulama

Paylaşılan Gateway token'ı/parolasıyla kimlik doğrulama, ilgili Gateway için
güvenilen operatör erişimi olarak değerlendirilir. OpenAI uyumlu HTTP yüzeyleri,
`/tools/invoke` ve HTTP oturum geçmişi uç noktaları, çağıran taraf daha dar
bildirilmiş kapsamlar gönderse bile paylaşılan gizli bilgi taşıyıcı kimlik doğrulaması
için varsayılan operatör kapsamlarının tamamını geri yükler.

Güvenilen proxy kimlik doğrulaması veya özel giriş `none` gibi kimlik
taşıyan kipler, açıkça bildirilmiş kapsamları yine de dikkate alabilir. Gerçek güven
sınırı ayrımı için ayrı Gateway'ler kullanın.
