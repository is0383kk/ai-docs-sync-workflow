---
read_when:
    - Bir yan etki çalıştırılmadan önce sormak için bir plugin kancasına veya aracına ihtiyacınız var
    - Plugin onay istemlerinin nereye iletileceğini yapılandırmanız gerekir
    - İsteğe bağlı araçlar, yürütme onayları ve plugin onayları arasında karar veriyorsunuz
sidebarTitle: Permission requests
summary: Kullanıcılardan Plugin araç çağrılarını ve Plugin'e ait izin istemlerini onaylamalarını isteyin
title: Plugin izin istekleri
x-i18n:
    generated_at: "2026-07-26T22:53:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 675534212e70cc7b2e7bdc801955929c6a8156b08d620483edf0133afc3bfdaa
    source_path: plugins/plugin-permission-requests.md
    workflow: 16
---

Plugin izin istekleri, kullanıcı onaylayana veya reddedene kadar Plugin kodunun bir araç çağrısını ya da Plugin'e ait bir işlemi duraklatmasına olanak tanır. Bunlar Gateway
`plugin.approval.*` akışını ve sohbet onay düğmeleriyle `/approve` komutlarını işleyen aynı onay kullanıcı arabirimi yüzeylerini kullanır.

Plugin/uygulama izinleri için Plugin izin isteklerini kullanın. Bunlar ana makine yürütme onaylarının, isteğe bağlı araç izin listelerinin veya Codex'in yerel izin incelemesinin yerini almaz.

## Doğru geçidi seçme

İhtiyaç duyduğunuz karar noktasına uygun geçidi seçin:

| Geçit                            | Kullanılacağı durum                                                       | Denetlediği öğe                                                                                                      |
| -------------------------------- | ------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| İsteğe bağlı araçlar             | Kullanıcı kabul edene kadar bir araç modele görünmemelidir.                | `tools.allow` üzerinden araç erişimi.                                                                           |
| Plugin izin istekleri            | Bir Plugin kancası veya Plugin'e ait işlem, bir eylem çalışmadan önce sormalıdır. | `plugin.approval.*` üzerinden çalışma zamanı onayı.                                                              |
| Yürütme onayları                 | Bir ana makine komutu veya kabuk benzeri araç için operatör onayı gerekir. | Ana makine yürütme politikası ve kalıcı yürütme izin listeleri.                                                       |
| Codex yerel izin istekleri       | Codex, yerel kabuk, dosya, MCP veya uygulama sunucusu eylemlerinden önce sorar. | Codex uygulama sunucusu veya yerel kanca onaylarının işlenmesi; istem OpenClaw'a ait olduğunda Plugin onayları üzerinden yönlendirilir. |
| MCP onay talepleri               | Bir Codex MCP sunucusu, araç çağrısı için onay ister.                      | OpenClaw Plugin onayları üzerinden köprülenen MCP onay yanıtları.                                                     |

İsteğe bağlı araçlar, keşif zamanında uygulanan bir geçittir. Plugin izin istekleri ise çağrı başına uygulanan bir geçittir. Hassas bir aracın model tarafından görülebilmesi için açıkça kabul edilmesi ve eylem çalışmadan önce onaylanması gerekiyorsa ikisini birlikte kullanın.

## Araç çağrısından önce onay isteme

Plugin tarafından oluşturulan istemlerin çoğu bir `before_tool_call` kancasında başlamalıdır. Kanca, model bir araç seçtikten sonra ve OpenClaw aracı çalıştırmadan önce yürütülür:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "deploy-policy",
  name: "Dağıtım Politikası",
  register(api) {
    api.on("before_tool_call", async (event) => {
      if (event.toolName !== "deploy_service") {
        return;
      }

      const environment =
        typeof event.params.environment === "string" ? event.params.environment : "unknown";

      return {
        requireApproval: {
          title: "Hizmeti dağıt",
          description: `Hizmeti ${environment} ortamına dağıt.`,
          severity: environment === "production" ? "critical" : "warning",
          allowedDecisions:
            environment === "production"
              ? ["allow-once", "deny"]
              : ["allow-once", "allow-always", "deny"],
          timeoutMs: 120_000,
          onResolution(decision) {
            console.log(`dağıtım onayı sonuçlandı: ${decision}`);
          },
        },
      };
    });
  },
});
```

İstem metnini, eylemi onaylayacak kişi için yazın:

- `title` değerini kısa ve eylem odaklı tutun; Gateway bunu 80 karakterle sınırlar.
- `description` değerini belirli ve sınırlı tutun; Gateway bunu 512
  karakterle sınırlar.
- Eylemi, hedefi ve riski belirtin. Sohbet onay yüzeylerinde görünmemesi gereken gizli bilgileri, belirteçleri veya özel yükleri eklemeyin.
- `severity` belirtilmediğinde varsayılan olarak `"warning"` kullanılır. `"critical"` değerini yalnızca yanlış kararın üretim ortamında hasara veya veri kaybına yol açabileceği eylemler için kullanın.
- `allowedDecisions` belirtilmediğinde varsayılan olarak `["allow-once", "allow-always", "deny"]` kullanılır. Söz konusu eylem için kalıcı güven uygun değilse `["allow-once", "deny"]` değerini geçirin.
- `timeoutMs` varsayılan olarak 120000 (2 dakika) değerindedir ve istenen değerden bağımsız olarak en fazla 600000 (10 dakika) olabilir.

## Karar davranışı

OpenClaw, bir `plugin:` kimliğiyle bekleyen bir onay oluşturur, bunu kullanılabilir onay yüzeylerine iletir ve karar verilmesini bekler.

| Karar             | Sonuç                                                                     |
| ----------------- | ------------------------------------------------------------------------- |
| `allow-once`      | Geçerli çağrı devam eder.                                                 |
| `allow-always`    | Geçerli çağrı devam eder ve karar Plugin'e aktarılır.                     |
| `deny`            | Çağrı, reddedilmiş bir araç sonucuyla engellenir.                         |
| Zaman aşımı       | Çağrı engellenir.                                                         |
| İptal             | Çalıştırma iptal edildiğinde çağrı engellenir.                            |
| Onay yolu yok     | Bağlı hiçbir onay yüzeyi isteği sonuçlandıramadığı için çağrı engellenir. |

Yalnızca isteğin izin verdiği tam `allow-once` ve `allow-always` kararları yürütmeye izin verir. Bilinmeyen, hatalı biçimlendirilmiş, eşleşmeyen, eksik ve zaman aşımına uğramış kararlar güvenli biçimde reddedilir. Eski `timeoutBehavior` alanı Plugin uyumluluğu için kabul edilmeye devam eder ancak kullanımdan kaldırılmıştır ve yok sayılır; yeni kancalarda bunu ayarlamayın.

`allow-always` yalnızca istekte bulunan Plugin veya çalışma zamanı bu kalıcılığı uyguladığında kalıcıdır. Sıradan `before_tool_call.requireApproval` kancalarında OpenClaw, `allow-once` ve `allow-always` değerlerini geçerli çağrıya yönelik onay kararları olarak değerlendirir ve çözümlenen değeri `onResolution` öğesine aktarır. Plugin'iniz `allow-always` seçeneğini sunuyorsa gelecekteki hangi çağrılara güvendiğini tam olarak belgeleyip uygulayın.

Kanca ayrıca `params` döndürürse OpenClaw bu parametre değişikliklerini yalnızca onay başarılı olduktan sonra uygular. Daha düşük öncelikli bir kanca, daha yüksek öncelikli bir kanca onay istemiş olsa bile çağrıyı engelleyebilir.

`allowedDecisions`, kullanıcıya gösterilen düğmeleri ve komutları sınırlar. Gateway, isteğin sunmadığı herhangi bir karara yönelik çözümleme girişimini reddeder.

## Onay istemlerini yönlendirme

Onay istemleri yerel kullanıcı arabirimi yüzeylerinde veya onay işlemeyi destekleyen sohbet kanallarında çözümlenebilir. Plugin onay istemlerini açık sohbet hedeflerine iletmek için `approvals.plugin` öğesini yapılandırın:

```json5
{
  approvals: {
    plugin: {
      enabled: true,
      mode: "targets",
      agentFilter: ["main"],
      targets: [{ channel: "slack", to: "U12345678" }],
    },
  },
}
```

`approvals.plugin`, `approvals.exec` öğesinden bağımsızdır. Yürütme onayı iletimini etkinleştirmek Plugin onay istemlerini yönlendirmez; Plugin onayı iletimini etkinleştirmek de ana makine yürütme politikasını değiştirmez.

Bir istem manuel onay metni içerdiğinde bunu sunulan kararlardan biriyle çözümleyin:

```text
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

Tam iletim modeli, aynı sohbette onay davranışı, yerel kanal iletimi ve kanala özgü onaylayan kuralları için [Gelişmiş yürütme onayları](/tr/tools/exec-approvals-advanced#plugin-approval-forwarding) bölümüne bakın.

## Codex yerel izinleri

Codex yerel izin istemleri de Plugin onayları üzerinden iletilebilir ancak bunların sahipliği, Plugin tarafından oluşturulan kancalardan farklıdır.

- Codex uygulama sunucusu onay istekleri, Codex incelemesinden sonra OpenClaw üzerinden yönlendirilir.
- Yerel `permission_request` kanca aktarıcısı etkinleştirildiğinde `plugin.approval.request` üzerinden istekte bulunabilir.
- Codex, `_meta.codex_approval_kind` öğesini `"mcp_tool_call"` olarak işaretlediğinde MCP araç onayı talepleri Plugin onayları üzerinden yönlendirilir.

Codex'e özgü davranış ve geri dönüş kuralları için [Codex çalıştırma altyapısı çalışma zamanı](/tr/plugins/codex-harness-runtime#native-permissions-and-mcp-elicitations) bölümüne bakın.

## Sorun giderme

**Araç, Plugin onaylarının kullanılamadığını söylüyor.** Hiçbir onay kullanıcı arabirimi veya yapılandırılmış onay yolu isteği kabul etmedi. Onay özelliğine sahip bir istemci bağlayın, aynı sohbette `/approve` desteği sunan bir kanal kullanın veya `approvals.plugin` öğesini yapılandırın.

**`allow-always` görünüyor ancak sonraki çağrı yeniden istem gösteriyor.** Genel Plugin onay akışı, rastgele kancalar için güveni otomatik olarak kalıcı hâle getirmez. `onResolution("allow-always")` sonrasında Plugin'e ait güveni Plugin'inizde kalıcılaştırın veya yalnızca `allow-once` ve `deny` seçeneklerini sunun.

**`/approve` kararı reddediyor.** İstek, `allowedDecisions` değerini sınırlandırmıştır. İstemde yazdırılan kararlardan birini kullanın.

**Bir Discord, Matrix, Slack veya Telegram istemi, yürütme onaylarından farklı şekilde yönlendiriliyor.** Plugin onayları ve yürütme onayları ayrı yapılandırmalar kullanır ve farklı yetkilendirme kontrolleri uygulayabilir. Yalnızca `approvals.exec` öğesini kontrol etmek yerine `approvals.plugin` öğesini ve kanalın Plugin onayı desteğini doğrulayın.

## İlgili konular

- [Plugin kancaları](/tr/plugins/hooks#tool-call-policy)
- [Plugin oluşturma](/tr/plugins/building-plugins#registering-tools)
- [Gelişmiş yürütme onayları](/tr/tools/exec-approvals-advanced#plugin-approval-forwarding)
- [Gateway protokolü](/tr/gateway/protocol)
- [Codex çalıştırma altyapısı çalışma zamanı](/tr/plugins/codex-harness-runtime#native-permissions-and-mcp-elicitations)
