---
read_when:
    - before_tool_call, before_agent_reply, ileti kancaları veya yaşam döngüsü kancaları gerektiren bir plugin oluşturuyorsunuz
    - Bir pluginden gelen araç çağrılarını engellemeniz, yeniden yazmanız veya onay gerektirecek şekilde yapılandırmanız gerekiyor
    - Dahili hook'lar ile plugin hook'ları arasında karar veriyorsunuz
    - OpenClaw Cron uyanmalarını harici bir ana makine zamanlayıcısına yansıtıyorsunuz
summary: 'Plugin kancaları: ajan, araç, mesaj, oturum ve Gateway yaşam döngüsü olaylarına müdahale edin'
title: Plugin kancaları
x-i18n:
    generated_at: "2026-07-26T23:29:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 95d7ea2f7bfe26b5904ea3cd8f8db85ffd8163af58e03ec56d11eee992bc13d2
    source_path: plugins/hooks.md
    workflow: 16
---

Plugin kancaları, OpenClaw pluginleri için işlem içi genişletme noktalarıdır: agent çalıştırmalarını, araç çağrılarını, mesaj akışını, oturum yaşam döngüsünü, alt agent yönlendirmesini, kurulumları veya Gateway başlangıcını inceleyebilir ya da değiştirebilirler.

Komut ve Gateway olaylarına tepki veren, operatör tarafından kurulmuş küçük bir
`HOOK.md` betiği için bunun yerine [dahili kancaları](/tr/automation/hooks) kullanın; örneğin `/new`,
`/reset`, `/stop`, `agent:bootstrap` veya `gateway:startup`.

## Hızlı başlangıç

Plugin girişinden `api.on(...)` ile türü belirlenmiş kancaları kaydedin:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "tool-preflight",
  name: "Araç Ön Kontrolü",
  register(api) {
    api.on(
      "before_tool_call",
      async (event) => {
        if (event.toolName !== "web_search") {
          return;
        }

        return {
          requireApproval: {
            title: "Web araması çalıştır",
            description: `Arama sorgusuna izin ver: ${String(event.params.query ?? "")}`,
            severity: "info",
            timeoutMs: 60_000,
          },
        };
      },
      { priority: 50 },
    );
  },
});
```

Karar veya değişiklik döndürebilen işleyiciler, azalan
`priority` sırasıyla ardışık olarak çalışır; aynı önceliğe sahip işleyiciler kayıt sırasını korur.
Yalnızca gözlem yapan işleyiciler paralel çalışır ve beklemeden başlatılan gözlem
dağıtımları sonraki olaylarla çakışabilir. Gözlemin yan etkilerini sıralamak için
önceliği kullanmayın.

`api.on(name, handler, opts?)` şunları kabul eder:

| Seçenek      | Etki                                                                                                                                                                                            |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `priority`  | Sıralama; daha yüksek olan önce çalışır.                                                                                                                                                                      |
| `timeoutMs` | Kanca başına bekleme bütçesi. Süre dolduğunda OpenClaw bu işleyiciyi beklemeyi bırakır ve devam eder. İşleyiciyi veya yan etkilerini iptal etmez. Çalıştırıcının varsayılan kanca başına zaman aşımını kullanmak için atlayın. |

Operatörler, plugin koduna yama uygulamadan kanca bütçelerini ayarlayabilir:

```json
{
  "plugins": {
    "entries": {
      "my-plugin": {
        "hooks": {
          "timeoutMs": 30000,
          "timeouts": {
            "before_prompt_build": 90000,
            "agent_end": 60000
          }
        }
      }
    }
  }
}
```

`hooks.timeouts.<hookName>`, `hooks.timeoutMs` değerini geçersiz kılar; o da
plugin tarafından oluşturulan `api.on(..., { timeoutMs })` değerini geçersiz kılar. Her değer,
600000 ms'ye kadar pozitif bir tam sayı olmalıdır. Bir pluginin her yerde daha uzun
bir bütçe almaması için yavaş olduğu bilinen kancalarda kanca başına geçersiz kılmaları tercih edin.

Zaman aşımına uğrayan bir işleyici vaadi çalışmayı sürdürür çünkü kanca geri çağrıları
iptal sinyali almaz. Plugin işi hâlâ devam ederken kanca dağıtımı Gateway
kabulünü serbest bırakabilir. Uzun süren işlerin sahibi olan pluginler kendi
iptal ve kapatma yaşam döngülerini sağlamalıdır.

Giden iletileri değiştiren `message_sending` ve `reply_payload_sending` kancaları,
işleyici başına varsayılan olarak 15 saniye kullanır. Bunlardan biri zaman aşımına uğrarsa OpenClaw plugin hatasını
günlüğe kaydeder ve serileştirilmiş teslimat hattının sonuçlanabilmesi için en son yükle
devam eder. Teslimattan önce kasıtlı olarak daha yavaş çalışan pluginler için daha büyük
bir kanca başına bütçe ayarlayın.

`createReplyDispatcher` kullanan kanal pluginleri de benzer biçimde
`beforeDeliverOptions: { timeoutMs }` ile veya `dispatcher.appendBeforeDeliver(handler, { timeoutMs })` kullanarak
iş eklerken aşama başına daha büyük bir pozitif bütçe bildirebilir.
Sahibi tarafından bildirilmiş bir bütçe olmadan bu geri çağrılar, takılı kalan bir geri çağrının
serileştirilmiş teslimat hattını elinde tutamaması için aynı 15 saniyelik varsayılanı kullanır.

Her kanca, bu işleyiciyi kaydeden plugin için çözümlenen yapılandırma olan
`event.context.pluginConfig` değerini alır. OpenClaw, diğer pluginlerin gördüğü paylaşılan
olay nesnesini değiştirmeden bunu her işleyiciye ayrı ayrı ekler.

## Kanca kataloğu

Kancalar, genişlettikleri yüzeye göre gruplandırılır. **Kalın** adlar bir karar
sonucu kabul eder (engelleme, iptal, geçersiz kılma veya onay gerektirme); diğerleri
yalnızca gözlem amaçlıdır.

**Agent dönüşü**

| Kanca                            | Amaç                                                                                  |
| ------------------------------- | ---------------------------------------------------------------------------------------- |
| `before_model_resolve`          | Oturum mesajları yüklenmeden önce sağlayıcıyı veya modeli geçersiz kılma                                  |
| `agent_turn_prepare`            | Kuyruğa alınmış plugin dönüş eklemelerini tüketme ve istem kancalarından önce aynı dönüşe bağlam ekleme      |
| `before_prompt_build`           | Model çağrısından önce dinamik bağlam veya sistem istemi metni ekleme                          |
| **`before_agent_run`**          | Modele göndermeden önce son istemi ve oturum mesajlarını inceleme; çalıştırmayı engelleyebilir |
| **`before_agent_reply`**        | Yapay bir yanıt veya sessizlik ile model dönüşünü kısa devre etme                           |
| **`before_agent_finalize`**     | Doğal son yanıtı inceleme ve modelden bir geçiş daha isteme                         |
| `agent_end`                     | Son mesajları, başarı durumunu ve çalıştırma süresini gözlemleme                                  |
| `heartbeat_prompt_contribution` | Arka plan izleme ve yaşam döngüsü pluginleri için yalnızca Heartbeat bağlamı ekleme                  |

**Konuşma gözlemi**

| Kanca                                      | Amaç                                                                                                            |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `model_call_started` / `model_call_ended` | Temizlenmiş sağlayıcı/model çağrısı meta verileri: zamanlama, sonuç, sınırlandırılmış istek kimliği karmaları. İstem veya yanıt içeriği yoktur. |
| `llm_input`                               | Sağlayıcı girdisi: sistem istemi, istem, geçmiş                                                                     |
| `llm_output`                              | Sağlayıcı çıktısı, kullanım ve mevcut olduğunda çözümlenen `contextTokenBudget`                                       |

**Araçlar**

| Kanca                       | Amaç                                                   |
| -------------------------- | --------------------------------------------------------- |
| **`before_tool_call`**     | Araç parametrelerini yeniden yazma, yürütmeyi engelleme veya onay gerektirme |
| `after_tool_call`          | Araç sonuçlarını, hataları ve süreyi gözlemleme                |
| `resolve_exec_env`         | `exec` için plugine ait ortam değişkenleri sağlama   |
| **`tool_result_persist`**  | Bir araç sonucundan üretilen asistan mesajını yeniden yazma |
| **`before_message_write`** | Devam eden bir mesaj yazımını inceleme veya engelleme (nadiren)      |

**Mesajlar ve teslimat**

| Kanca                            | Amaç                                                           |
| ------------------------------- | ----------------------------------------------------------------- |
| **`inbound_claim`**             | Agent yönlendirmesinden önce gelen bir mesajı üstlenme (yapay yanıtlar) |
| **`channel_pairing_requested`** | Yeni oluşturulan doğrudan mesaj eşleştirme isteklerini gözlemleme                         |
| `message_received`              | Gelen içeriği, göndericiyi, ileti dizisini ve meta verileri gözlemleme             |
| **`message_sending`**           | Giden içeriği yeniden yazma veya teslimatı iptal etme                       |
| **`reply_payload_sending`**     | Teslimattan önce normalleştirilmiş yanıt yüklerini değiştirme veya iptal etme        |
| `message_sent`                  | Giden teslimatın başarısını veya başarısızlığını gözlemleme                      |
| **`before_dispatch`**           | Kanal devrine geçmeden önce giden bir dağıtımı inceleme veya yeniden yazma    |
| **`reply_dispatch`**            | Son yanıt dağıtım işlem hattına katılma                  |

**Oturumlar ve Compaction**

| Kanca                                     | Amaç                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `session_start` / `session_end`          | Oturum yaşam döngüsü sınırlarını izleme. `reason`; `new`, `reset`, `idle`, `daily`, `compaction`, `deleted`, `shutdown`, `restart` veya `unknown` değerlerinden biridir. `shutdown`/`restart`, işlem etkin oturumlarla durduğunda veya yeniden başlatıldığında Gateway kapatma sonlandırıcısından tetiklenir; böylece pluginler (bellek, transkript depoları), hayalet satırları yeniden başlatmalar boyunca açık bırakmak yerine sonlandırabilir. Yavaş bir pluginin SIGTERM/SIGINT sinyallerini engelleyememesi için sonlandırıcı sınırlandırılmıştır. |
| `before_compaction` / `after_compaction` | Compaction döngülerini gözlemleme veya açıklama ekleme                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `before_reset`                           | Oturum sıfırlama olaylarını gözlemleme (`/reset`, programatik sıfırlamalar)                                                                                                                                                                                                                                                                                                                                                                                                     |

`parentSessionKey` ve `emitCommandHooks: true` içeren `sessions.create` çağrılarında, ayrı bir alt öğe her zaman `session_start` alır. Çağıranlar, üst öğenin son `session_end` değerini de alıp almayacağını `succeedsParent` ile bildirir: `true` ardıl, `false` paralel alt öğe anlamına gelir. Atlanması, eski üst öğe devretme davranışını korur. `command:new` ve `before_reset` kancaları her iki durumda da istenen `/new` eylemini açıklamaya devam eder.

**Alt agentlar**

- `subagent_spawned` / `subagent_ended` - alt aracının başlatılmasını ve tamamlanmasını gözlemleyin.
- `subagent_delivery_target` - hiçbir çekirdek oturum bağlaması bir rota yansıtamadığında tamamlanma teslimatı için uyumluluk kancası.
- `subagent_spawning` - kullanımdan kaldırılmış uyumluluk kancası. Çekirdek artık `subagent_spawned` tetiklenmeden önce kanal oturum bağlama bağdaştırıcıları aracılığıyla `thread: true` alt araç bağlamalarını hazırlar.
- `subagent_spawned`, OpenClaw başlatmadan önce alt oturumun yerel modelini çözümlediğinde `resolvedModel` ve `resolvedProvider` değerlerini içerir.
- `subagent_ended`; `targetSessionKey` (kimlik - `subagent_spawned.childSessionKey` ile eşleşir), `targetKind` (`"subagent"` veya `"acp"`), `reason`, isteğe bağlı `outcome` (`"ok"`, `"error"`, `"timeout"`, `"killed"`, `"reset"` veya `"deleted"`), isteğe bağlı `error`, `runId`, `endedAt`, `accountId` ve `sendFarewell` değerlerini taşır. `agentId` veya `childSessionKey` değerlerini **içermez**; eşleşen `subagent_spawned` olayıyla ilişkilendirmek için `targetSessionKey` kullanın.

**Yaşam Döngüsü**

| Kanca                             | Amaç                                                                                              |
| -------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `gateway_start` / `gateway_stop` | Plugin'e ait hizmetleri Gateway ile birlikte başlatır veya durdurur                                                 |
| `deactivate`                     | `gateway_stop` için kullanımdan kaldırılmış uyumluluk diğer adı; yeni pluginlerde `gateway_stop` kullanın                 |
| `cron_reconciled`                | Başlatma veya yeniden yükleme sonrasında eksiksiz Gateway Cron durumuyla mutabakat sağlar                            |
| `cron_changed`                   | Gateway'in yönettiği Cron yaşam döngüsü değişikliklerini gözlemler (eklendi, güncellendi, kaldırıldı, başlatıldı, tamamlandı, zamanlandı) |
| **`before_install`**             | Yüklenmiş bir plugin çalışma zamanından aşamalandırılmış skill veya plugin kurulum malzemesini inceler                         |

### Kanal eşleştirme istekleri

Eşleştirilmemiş bir DM göndericisi bekleyen bir eşleştirme isteği oluşturduktan
sonra bir plugin'in operatörü bilgilendirmesi veya denetim kaydı yazması
gerektiğinde `channel_pairing_requested` kullanın. Kanca, istek oluşturulduğunda tetiklenir;
eşleştirme yanıtının kanal üzerinden teslimi, yavaş veya başarısız kanca işleyicileri
nedeniyle geciktirilmez.

```typescript
api.on("channel_pairing_requested", async (event) => {
  await notifyOperator({
    text: `${event.senderId} kaynağından yeni ${event.channel} eşleştirme isteği: ${event.code}`,
  });
});
```

Kanca yalnızca gözlem amaçlıdır. Eşleştirme yanıtını onaylamaz, reddetmez,
bastırmaz veya yeniden yazmaz. Yük; kanalı, isteğe bağlı `accountId`,
kanal kapsamlı `senderId`, eşleştirme `code` değerini ve kanal meta verilerini içerir.
Eşleştirme kodunu etkin, tek kullanımlık bir onay kimlik bilgisi olarak değerlendirin
ve yalnızca güvenilir bir operatör hedefine teslim edin. `metadata` değerini,
gönderici tarafından sağlanan güvenilmeyen kimlik metni olarak değerlendirin.
Kanca, gelen mesajın gövdesini veya medyasını içermez.

## Hata ayıklama çalışma zamanı kancaları

Bir araç dönüşü için sağlayıcıyı veya modeli değiştirmek üzere `before_model_resolve`
kullanın; model çözümlemesinden önce çalışır. `llm_output` yalnızca bir model
denemesi yardımcı çıktısı ürettikten sonra çalışır.

Geçerli oturum modelini doğrulamak için çalışma zamanı kayıtlarını inceleyin,
ardından `openclaw sessions` veya Gateway oturum/durum yüzeylerini kullanın. Sağlayıcı
yüklerinde hata ayıklamak için ham model akış olaylarını bir jsonl dosyasına
yazmak üzere Gateway'i `--raw-stream` ve `--raw-stream-path <path>` ile başlatın.

## Araç çağrısı politikası

`before_tool_call` şunları alır:

- `event.toolName`
- `event.params`
- isteğe bağlı `event.toolKind` ve `event.toolInputKind`; bilinçli olarak aynı adları
  paylaşan araçlar için sunucunun yetkili olduğu ayırt ediciler; örneğin, dış
  kod modu `exec` çağrıları `toolKind: "code_mode_exec"` kullanır ve giriş dili
  bilindiğinde `toolInputKind: "javascript" | "typescript"` değerini içerir
- isteğe bağlı `event.derivedPaths`; `apply_patch` gibi iyi bilinen araç zarfları
  için sunucu tarafından en iyi çabayla türetilen hedef yol ipuçları; bu yollar
  eksik olabilir veya aracın gerçekte dokunacağı yerleri olduğundan geniş
  gösterebilir (örneğin, hatalı biçimlendirilmiş veya kısmi girdilerde)
- isteğe bağlı `event.runId`
- isteğe bağlı `event.toolCallId`
- `ctx.agentId`, `ctx.sessionKey`, `ctx.sessionId`,
  `ctx.runId`, `ctx.toolKind`, `ctx.toolInputKind` ve tanılama amaçlı `ctx.trace`
  gibi bağlam alanları
- isteğe bağlı `ctx.requester`; mevcut mesaj çalışmasını başlatan ve sunucu
  tarafından türetilen istekte bulunan taraf. `channel`, `accountId`,
  `senderId`, `senderIsOwner` ve sağlayıcıya özgü `roleIds`
  değerlerini içerebilir. Eksik alanlar kanıtlanmamıştır; yanlış olmadığına dair
  güvence değildir. Politika gerektiriyorsa kapalı durumda başarısız olun.

Şunları döndürebilir:

```typescript
type BeforeToolCallResult = {
  params?: Record<string, unknown>;
  block?: boolean;
  blockReason?: string;
  requireApproval?: {
    title: string;
    description: string;
    severity?: "info" | "warning" | "critical";
    timeoutMs?: number;
    /** @deprecated Çözümlenmemiş onaylar her zaman reddedilir. */
    timeoutBehavior?: "allow" | "deny";
    allowedDecisions?: Array<"allow-once" | "allow-always" | "deny">;
    pluginId?: string;
    onResolution?: (
      decision: "allow-once" | "allow-always" | "deny" | "timeout" | "cancelled",
    ) => Promise<void> | void;
  };
};
```

Türü belirlenmiş yaşam döngüsü kancalarının koruma davranışı:

- `block: true` sonlandırıcıdır ve daha düşük öncelikli işleyicileri atlar.
- `block: false` karar verilmemiş olarak değerlendirilir.
- `params` yürütme için araç parametrelerini yeniden yazar.
- `requireApproval` araç çalışmasını duraklatır ve plugin onayları aracılığıyla
  kullanıcıya sorar. `/approve`, hem exec hem de plugin onaylarını
  onaylayabilir. Codex app-server rapor modundaki yerel `PreToolUse`
  aktarımlarında bu işlem, eşleşen app-server onay isteğine bırakılır; bkz.
  [Codex harness çalışma zamanı](/tr/plugins/codex-harness-runtime#hook-boundaries).
- Daha düşük öncelikli bir `block: true`, daha yüksek öncelikli bir kanca
  onay istemiş olsa bile engelleme uygulayabilir.
- `onResolution` çözümlenen kararı alır: `allow-once`, `allow-always`,
  `deny`, `timeout` veya `cancelled`.

### Tek dosyada göndericiye duyarlı politika

Bağımsız bir plugin dosyası, başka bir yapılandırma şeması eklemek yerine
dağıtıma özgü politikayı kodda tutabilir. Bu örnek, sahiplere tüm araçları
verir, yapılandırılmış bakım sorumlularının kısıtlayıcı bir araç ve mesaj eylemi
kümesi kullanmasına izin verir ve `/fix` değerini kanal yapılandırması
tarafından zaten yetkilendirilmiş göndericilere açar:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const AGENT_ID = "maintenance-agent";
const MAINTAINER_SCOPES = [
  {
    channel: "discord",
    accountId: "operations",
    senderIds: new Set(["maintainer-user-id"]),
    roleIds: new Set(["maintainer-role-id"]),
  },
];
const MAINTAINER_TOOLS = new Set(["read", "web_fetch", "web_search", "session_status", "message"]);
const MAINTAINER_MESSAGE_ACTIONS = new Set(["react", "reply", "thread-create", "thread-reply"]);

export default definePluginEntry({
  id: "maintenance-access",
  name: "Bakım erişimi",
  description: "Bakım aracına göndericiye duyarlı araç politikası uygula.",
  register(api) {
    api.on("before_tool_call", (event, ctx) => {
      if (ctx.agentId !== AGENT_ID) {
        return;
      }

      const requester = ctx.requester;
      if (requester?.senderIsOwner === true) {
        return;
      }

      const maintainerScope = requester
        ? MAINTAINER_SCOPES.find(
            (scope) =>
              scope.channel === requester.channel && scope.accountId === requester.accountId,
          )
        : undefined;
      const isMaintainer =
        maintainerScope !== undefined &&
        ((requester?.senderId !== undefined && maintainerScope.senderIds.has(requester.senderId)) ||
          requester?.roleIds?.some((roleId) => maintainerScope.roleIds.has(roleId)) === true);
      if (!isMaintainer) {
        return { block: true, blockReason: "Bakım sorumlusu erişimi gerekli." };
      }

      if (event.toolName === "message") {
        const action = typeof event.params.action === "string" ? event.params.action : "";
        if (MAINTAINER_MESSAGE_ACTIONS.has(action)) {
          return;
        }
        return { block: true, blockReason: `message.${action || "unknown"} için sahip gerekli.` };
      }

      if (MAINTAINER_TOOLS.has(event.toolName)) {
        return;
      }
      return { block: true, blockReason: `${event.toolName} için sahip gerekli.` };
    });

    api.registerCommand({
      name: "fix",
      description: "Bakım aracından bir sorunu araştırmasını ve düzeltmesini iste.",
      acceptsArgs: true,
      requireAuth: true,
      handler: async (ctx) =>
        ctx.agentId === AGENT_ID
          ? { continueAgent: true }
          : { text: "Bu komut yalnızca bakım görüşmesinde kullanılabilir." },
    });
  },
});
```

Dosyayı doğrudan yükleyin ve Gateway'i yeniden başlatın:

```json5
{
  agents: {
    list: [
      {
        id: "maintenance-agent",
        workspace: "~/.openclaw/workspace-maintenance",
      },
    ],
  },
  bindings: [
    {
      agentId: "maintenance-agent",
      match: {
        channel: "discord",
        accountId: "operations",
        peer: { kind: "channel", id: "maintenance-channel-id" },
      },
    },
  ],
  plugins: {
    load: { paths: ["~/.openclaw/policies/maintenance-access.ts"] },
  },
}
```

`AGENT_ID`, bakım görüşmesine bağlanan aracın adını belirtmelidir.
Bağlama, normal mesajlar ve `/fix` için bu aracı seçer; bağımsız dosya,
sahip ile bakım sorumlusu arasındaki araç politikasının tek sahibi olarak kalır.

`requireAuth: true`, her kanalın mevcut gönderici kabul mekanizmasını yeniden kullanır.
Discord için bir sunucu veya kanal `users`/`roles` izin listesi
bakım kitlesini yetkilendirebilir. Diğer kanallar kararlı gönderici kimliklerini
kullanabilir. Kanca daha sonra, Codex'in yerel `PreToolUse` çağrıları dahil
olmak üzere çalışmadaki her araç çağrısına daha ayrıntılı araç bazlı kararı uygular.
Modelin gördüğü bir aracı veto edebilir ancak sunucu tarafından atlanan bir aracı
ekleyemez. Mevcut sandbox, exec onayı, yalnızca sahibe açık çekirdek araç ve kanal
politikaları geçerliliğini korur; kanca bunları aşan izin veremez.

Gönderici ve rol kimliklerini gösterildiği gibi tam bir kanal/hesap çiftiyle
sınırlandırın; ikisi de sağlayıcıya özgü ad alanlarıdır. İzin listelerini
kısıtlayıcı tutun. Yazma veya yürütme araçlarını yalnızca dağıtımın sandbox ve
onay politikası bunu güvenli kılıyorsa ekleyin. Otomatik veya sistem çalışmaları
için eksik bir `ctx.requester` değerinin geçip geçmemesi gerektiğine açıkça
karar verin; örnek, kapsamlı araç için bunu reddeder.

Onay yönlendirmesi, karar davranışı ve isteğe bağlı araçlar veya exec onayları
yerine ne zaman `requireApproval` kullanılacağı hakkında bilgi için
[Plugin izin istekleri](/tr/plugins/plugin-permission-requests) bölümüne bakın.

Sunucu düzeyinde politikaya ihtiyaç duyan pluginler, güvenilir araç politikalarını
`api.registerTrustedToolPolicy(...)` ile kaydedebilir. Bunlar sıradan
`before_tool_call` kancalarından ve normal kanca kararlarından önce çalışır. Paketle
birlikte gelen güvenilir politikalar önce, kurulu pluginlerin güvenilir politikaları
ise plugin yükleme sırasına göre daha sonra çalışır; sıradan `before_tool_call`
kancaları bunların ardından çalışır. Paketle birlikte gelen pluginler mevcut
güvenilir politika yolunu korur. Kurulu pluginlerin açıkça etkinleştirilmesi ve
her politika kimliğini `contracts.trustedToolPolicies` içinde bildirmesi gerekir; bildirilmemiş
kimlikler kayıttan önce reddedilir. Politika kimlikleri kaydeden plugin kapsamında
olduğundan farklı pluginler aynı yerel kimliği yeniden kullanabilir. Bu katmanı
yalnızca çalışma alanı politikası, bütçe uygulaması veya ayrılmış iş akışı güvenliği
gibi sunucunun güvendiği korumalar için kullanın.

### Exec ortamı hook'u

`resolve_exec_env`, komut çalıştırılmadan önce Plugin'lerin `exec`
araç çağrılarına ortam değişkenleri eklemesine olanak tanır. Şunları alır:

- `event.sessionKey`
- `event.toolName`, şu anda her zaman `"exec"`
- `event.host`, `"gateway"`, `"sandbox"` veya `"node"` değerlerinden biri
- `ctx.agentId`, `ctx.sessionKey`,
  `ctx.messageProvider` ve `ctx.channelId` gibi bağlam alanları

Exec ortamıyla birleştirilmek üzere bir `Record<string, string>` döndürün. İşleyiciler
öncelik sırasına göre çalışır; daha sonraki sonuçlar aynı anahtar için önceki
sonuçları geçersiz kılar.

Hook çıktısı, birleştirmeden önce ana makinenin exec ortamı anahtar politikasından
geçirilerek filtrelenir. `PATH` her zaman çıkarılır (komut çözümleme ve güvenli ikili dosya denetimleri
buna bağlıdır). Geçersiz anahtarlar ile `LD_*`,
`DYLD_*`, `NODE_OPTIONS` gibi tehlikeli ana makine geçersiz kılma anahtarları, proxy değişkenleri (`HTTP_PROXY`, `HTTPS_PROXY`,
`ALL_PROXY`, `NO_PROXY`) ve TLS geçersiz kılma değişkenleri (`NODE_TLS_REJECT_UNAUTHORIZED`,
`SSL_CERT_FILE` ve benzerleri) çıkarılır. Filtrelenmiş Plugin ortamı,
Gateway onay/denetim meta verilerine eklenir ve node ana makinesi yürütme
isteklerine iletilir.

### Araç sonucu kalıcılığı

Araç sonuçları; kullanıcı arayüzünde işleme, tanılama, medya yönlendirme
veya Plugin'e ait meta veriler için yapılandırılmış `details` içerebilir. `details` değerini istem içeriği
olarak değil, çalışma zamanı meta verisi olarak değerlendirin:

- OpenClaw, meta verilerin model bağlamına dönüşmemesi için sağlayıcı yeniden oynatımından ve Compaction
  girdisinden önce `toolResult.details` değerini çıkarır.
- Kalıcı oturum girdileri yalnızca sınırlandırılmış `details` değerini tutar. Aşırı büyük ayrıntılar,
  kısa bir özet ve `persistedDetailsTruncated: true` ile değiştirilir.
- `tool_result_persist` ve `before_message_write`, son
  kalıcılık sınırından önce çalışır. Döndürülen `details` değerini küçük tutun ve
  istemle ilgili metni yalnızca `details` içine yerleştirmekten kaçının; modelin görebileceği araç çıktısını
  `content` içine koyun.

## İstem ve model hook'ları

Yeni Plugin'ler için aşamaya özgü hook'ları kullanın:

- `before_model_resolve`: yalnızca geçerli istemi ve ek
  meta verilerini alır. `providerOverride` veya `modelOverride` döndürün.
- `agent_turn_prepare`: geçerli istemi, hazırlanmış oturum
  mesajlarını ve bu oturum için boşaltılan, tam olarak bir kez uygulanacak sıraya alınmış eklemeleri alır.
  `prependContext` veya `appendContext` döndürün.
- `before_prompt_build`: geçerli istemi ve oturum mesajlarını alır.
  `prependContext`, `appendContext`, `systemPrompt`,
  `prependSystemContext` veya `appendSystemContext` döndürün.
- `heartbeat_prompt_contribution`: yalnızca Heartbeat turlarında çalışır ve
  `prependContext` veya `appendContext` döndürür. Kullanıcı tarafından başlatılan turları
  değiştirmeden mevcut durumu özetlemesi gereken arka plan izleyicileri için tasarlanmıştır.

`before_agent_run`, istem oluşturulduktan sonra ve isteme yerel görüntü yükleme ile
`llm_input` gözlemi dâhil olmak üzere herhangi bir model girdisinden önce çalışır. Geçerli
kullanıcı girdisini `prompt` olarak, yüklenmiş oturum geçmişini `messages`
içinde ve etkin sistem istemini alır. Model istemi okumadan önce çalışmayı durdurmak için
`{ outcome: "block", reason, message? }` döndürün. `reason` dâhilîdir;
`message` kullanıcıya gösterilen ikamedir. Yalnızca `pass` ve `block` sonuçları
desteklenir; desteklenmeyen karar biçimleri güvenli biçimde başarısız olur.

Bir çalışma engellendiğinde OpenClaw, `message.content` içinde yalnızca ikame metni
ve engelleyen Plugin kimliği ile zaman damgası gibi hassas olmayan engelleme meta verilerini
saklar. Özgün kullanıcı metni transkriptte veya gelecekteki bağlamda tutulmaz.
Dâhilî engelleme nedenleri hassas kabul edilir ve transkript, geçmiş, yayın,
günlük ve tanılama yüklerinden hariç tutulur. Gözlemlenebilirlik; engelleyici kimliği,
sonuç, zaman damgası veya güvenli bir kategori gibi temizlenmiş alanları kullanmalıdır.

`agent_end` dâhil olmak üzere ajan turu hook'ları, OpenClaw etkin çalışmayı
tanımlayabildiğinde `event.runId` içerir; aynı değer `ctx.runId` üzerinde de bulunur. Cron tarafından yürütülen
çalışmalar ayrıca ajan turu bağlamında `ctx.jobId` (kaynak Cron işi kimliği) sunar;
böylece hook'lar metrikleri, yan etkileri veya durumu belirli bir
zamanlanmış işle sınırlandırabilir. `ctx.jobId`, `before_tool_call` araç bağlamının parçası değildir.

Kanal kaynaklı çalışmalar için `ctx.channel` ve `ctx.messageProvider`,
`discord` veya `telegram` gibi sağlayıcı yüzeyini tanımlar; `ctx.channelId` ise
OpenClaw bunu oturum anahtarından veya teslimat meta verilerinden türetebildiğinde
konuşma hedefi tanımlayıcısıdır.

Gönderen kimliği kullanılabilir olduğunda ajan hook bağlamları ayrıca şunları içerir:

- `ctx.senderId` - kanal kapsamındaki gönderen kimliği (ör. Feishu `open_id`, Discord
  kullanıcı kimliği). Çalışma, gönderen meta verileri bilinen bir kullanıcı mesajından
  kaynaklandığında doldurulur.
- `ctx.chatId` - aktarıma özgü konuşma tanımlayıcısı (ör. Feishu
  `chat_id`, Telegram `chat_id`). Kaynak kanal
  yerel bir konuşma kimliği sağladığında doldurulur.
- `ctx.channelContext.sender.id` - Plugin'lerin kanala özgü alanlarla genişletebileceği,
  kanala ait bir nesne altında `ctx.senderId` ile aynı gönderen kimliği.
- `ctx.channelContext.chat.id` - Plugin'lerin kanala özgü
  alanlarla genişletebileceği, kanala ait bir nesne altında `ctx.chatId` ile aynı konuşma kimliği.

Çekirdek yalnızca iç içe `id` alanlarını tanımlar. Gelen yardımcı üzerinden
daha zengin gönderen veya sohbet meta verileri geçiren kanal Plugin'leri,
`openclaw/plugin-sdk/channel-inbound` içindeki `PluginHookChannelSenderContext` veya `PluginHookChannelChatContext`
alanlarını genişletebilir:

```ts
declare module "openclaw/plugin-sdk/channel-inbound" {
  interface PluginHookChannelSenderContext {
    unionId?: string;
    userId?: string;
  }
}
```

Kanal Plugin'leri bu alanları gelen SDK yardımcısı üzerinden geçirir:

```ts
buildChannelInboundEventContext({
  // ...
  channelContext: {
    sender: { id: senderOpenId, unionId, userId },
    chat: { id: chatId },
  },
});
```

Bu alanlar isteğe bağlıdır ve sistem kaynaklı çalışmalarda (Heartbeat,
Cron, exec olayı) bulunmaz.

`ctx.senderExternalId`, eski Plugin'ler için kullanımdan kaldırılmış bir kaynak uyumluluğu alanı
olarak kalır. Çekirdek bunu doldurmaz; yeni kanala özgü gönderen
kimlikleri, modül genişletme yoluyla `ctx.channelContext.sender` altında
bulunmalıdır.

`agent_end` bir gözlem hook'udur. Gateway ve kalıcı çalıştırma düzeneği yolları,
turdan sonra bunu beklemeden çalıştırırken kısa ömürlü tek seferlik CLI yolları,
güvenilen Plugin'lerin terminal gözlemlenebilirliğini boşaltabilmesi veya durumu
yakalayabilmesi için süreç temizliğinden önce hook promise'ini bekler. Hook çalıştırıcısı,
takılı kalan bir Plugin'in veya gömme uç noktasının hook promise'ini sonsuza kadar
beklemede bırakamaması için 30 saniyelik zaman aşımı uygular. Zaman aşımı günlüğe kaydedilir
ve OpenClaw devam eder; Plugin kendi iptal sinyalini de kullanmadığı sürece
Plugin'e ait ağ çalışmasını iptal etmez.

Ham istemleri, geçmişi, yanıtları, başlıkları, istek gövdelerini veya sağlayıcı istek
kimliklerini almaması gereken sağlayıcı çağrısı telemetrisi için `model_call_started` ve
`model_call_ended` kullanın. Bu hook'lar `runId`,
`callId`, `provider`, `model`, isteğe bağlı `api`/`transport`, son
`durationMs`/`outcome` ve OpenClaw sınırlandırılmış bir sağlayıcı istek kimliği özeti
türetebildiğinde `upstreamRequestIdHash` gibi kararlı meta verileri içerir. Çalışma zamanı
bağlam penceresi meta verilerini çözümlediğinde hook olayı ve bağlamı ayrıca
model/yapılandırma/ajan sınırlarından sonraki etkin token bütçesi olan
`contextTokenBudget` ile daha düşük bir sınır uygulandığında `contextWindowSource` ve
`contextWindowReferenceTokens` değerlerini içerir.

`before_agent_finalize` yalnızca bir çalıştırma düzeneği doğal bir son asistan yanıtını
kabul etmek üzereyken çalışır. `/stop` iptal yolu değildir ve kullanıcı
bir turu iptal ettiğinde çalışmaz. Sonlandırmadan önce çalıştırma düzeneğinden
bir model geçişi daha istemek için `{ action: "revise", reason }`, sonlandırmayı zorlamak için
`{ action:
"finalize", reason? }` döndürün veya devam etmek için bir sonuç döndürmeyin.
İşleyicilerin varsayılan bütçesi 15 saniyedir; zaman aşımında OpenClaw hatayı günlüğe kaydeder ve
özgün son yanıtla devam eder.
Codex yerel `Stop` hook'ları, bu hook'a OpenClaw
`before_agent_finalize` kararları olarak aktarılır.

Plugin'ler `action: "revise"` döndürürken ek model geçişini sınırlandırılmış ve yeniden oynatmaya
güvenli hâle getirmek için `retry` meta verilerini ekleyebilir:

```typescript
type BeforeAgentFinalizeRetry = {
  instruction: string;
  idempotencyKey?: string;
  maxAttempts?: number;
};
```

`instruction`, çalıştırma düzeneğine gönderilen düzeltme nedenine eklenir.
`idempotencyKey`, ana makinenin eşdeğer sonlandırma kararları arasında aynı Plugin isteği
için yeniden denemeleri saymasını sağlar; `maxAttempts` ise ana makinenin doğal son yanıtla
devam etmeden önce izin vereceği ek geçiş sayısını sınırlar.

Ham konuşma hook'larına (`before_model_resolve`,
`before_agent_reply`, `llm_input`, `llm_output`, `before_agent_finalize`,
`agent_end` veya `before_agent_run`) ihtiyaç duyan paketle birlikte gelmeyen Plugin'ler şunu ayarlamalıdır:

```json
{
  "plugins": {
    "entries": {
      "my-plugin": {
        "hooks": {
          "allowConversationAccess": true
        }
      }
    }
  }
}
```

İstemi değiştiren hook'lar ve kalıcı sonraki tur eklemeleri, Plugin başına
`plugins.entries.<id>.hooks.allowPromptInjection=false` ile devre dışı bırakılabilir.

### Oturum uzantıları ve sonraki tur eklemeleri

İş akışı Plugin'leri, `api.session.state.registerSessionExtension(...)` ile JSON uyumlu küçük oturum durumlarını kalıcılaştırabilir ve
Gateway `sessions.pluginPatch` yöntemi üzerinden güncelleyebilir. Oturum satırları, kayıtlı
uzantı durumunu `pluginExtensions` üzerinden yansıtarak Control UI ve diğer
istemcilerin Plugin iç işleyişini öğrenmeden Plugin'e ait durumu işlemesine olanak tanır.
`api.registerSessionExtension(...)` çalışmaya devam eder ancak
`api.session.state` ad alanı lehine kullanımdan kaldırılmıştır.

Bir Plugin'in kalıcı bağlamı tam olarak bir kez sonraki model turuna ulaştırması
gerektiğinde `api.session.workflow.enqueueNextTurnInjection(...)` kullanın (üst düzey
`api.enqueueNextTurnInjection(...)`, aynı davranışa sahip kullanımdan kaldırılmış bir diğer addır).
OpenClaw, sıraya alınmış eklemeleri istem hook'larından önce boşaltır, süresi dolmuş
eklemeleri çıkarır ve Plugin başına `idempotencyKey` değerine göre yinelenenleri ayıklar. Bu,
onay devamları, politika özetleri, arka plan izleyici farkları ve sonraki turda
modele görünmesi gereken ancak kalıcı sistem istemi metnine dönüşmemesi gereken
komut devamları için doğru bağlantı noktasıdır.

Temizleme semantiği sözleşmenin bir parçasıdır. Oturum uzantısı temizleme ve
çalışma zamanı yaşam döngüsü temizleme geri çağrıları `reset`, `delete`, `disable` veya
`restart` alır. Ana makine sıfırlama/silme/devre dışı bırakma işlemlerinde sahip Plugin'in
kalıcı oturum uzantısı durumunu ve bekleyen sonraki tur eklemelerini kaldırır;
yeniden başlatma kalıcı oturum durumunu korurken temizleme geri çağrıları Plugin'lerin
eski çalışma zamanı nesline ait zamanlayıcı işlerini, çalışma bağlamını ve diğer
bant dışı kaynakları serbest bırakmasına olanak tanır.

## Mesaj hook'ları

Kanal düzeyinde yönlendirme ve teslimat politikası için mesaj hook'larını kullanın:

- `message_received`: gelen içeriği, göndereni, `threadId`,
  `messageId`, `senderId`, isteğe bağlı çalışma/oturum korelasyonunu, sıralı `media`
  ve meta verileri gözlemler.
- `message_sending`: `content` değerini yeniden yazar veya `{ cancel: true }` döndürür.
- `reply_payload_sending`: normalleştirilmiş `ReplyPayload` nesnelerini
  (`presentation`, `delivery`, medya referansları ve metin dâhil) yeniden yazar veya
  `{ cancel: true }` döndürür.
- `message_sent`: nihai başarıyı veya başarısızlığı gözlemler.

Yalnızca ses içeren TTS yanıtlarında, kanal yükünde görünür metin/açıklama
olmasa bile `content` gizli sözlü transkripti içerebilir.
Bu `content` değerinin yeniden yazılması yalnızca hook'un görebildiği transkripti günceller;
medya açıklaması olarak işlenmez.

`reply_payload_sending` olayları, en iyi çabayla sağlanan canlı
tur başına model/kullanım/bağlam anlık görüntüsü olan `usageState` değerini içerebilir. Kalıcı teslimat, kurtarılmış yeniden oynatma ve
tam çalışma korelasyonu olmayan yanıtlar bunu içermez.

İleti hook bağlamları, mevcut olduğunda kararlı korelasyon alanlarını kullanıma sunar:
`ctx.sessionKey`, `ctx.runId`, `ctx.messageId`, `ctx.senderId`, `ctx.trace`,
`ctx.traceId`, `ctx.spanId`, `ctx.parentSpanId` ve `ctx.callDepth`. Gelen
ve `before_dispatch` bağlamları ayrıca, kanal görünürlüğe göre filtrelenmiş
alıntılanan ileti verilerine sahip olduğunda yanıt meta verilerini kullanıma sunar:
`replyToId`, `replyToIdFull`, `replyToBody`, `replyToSender` ve
`replyToIsQuote`. Eski meta verileri okumadan önce bu birinci sınıf alanları
tercih edin.

Kanala özgü meta verileri kullanmadan önce türü belirlenmiş `threadId` ve
`replyToId` alanlarını tercih edin.

Gelen talep ve ileti alındı olayları, standart ek API'si olarak
`media?:
PluginHookMediaFact[]` alanını kullanıma sunar. Her olgu `path`,
`url`, `contentType`, `kind`, `transcribed`,
`messageId` ve `workspaceDir` taşıyabilir; dizi konumu ekin
kimliğidir. Uzak bir ek henüz yerel olarak hazırlanmadığında
`media` atlanır, `mediaStagingPending: true` ve `originalMedia`
sağlayıcı tarafındaki olguları içerir. Daha sonraki bir hazırlanmış olay
`media` sağlayana kadar `originalMedia.path` alanını yerel olarak
okunabilir kabul etmeyin.

Tekil/çoğul `mediaPath`, `mediaUrl`, `mediaType`,
`mediaPaths`, `mediaUrls`, `mediaTypes` ve bunlarla eşleşen
`originalMedia*` meta veri özellikleri, kullanım dışı bırakılmış uyumluluk
takma adlarıdır. Yeni hook'lar, türü belirlenmiş üst düzey dizileri kullanmalıdır.

Karar kuralları:

- `message_sending` ile `cancel: true` sonlandırıcıdır.
- `message_sending` ile `cancel: false` karar verilmemiş olarak değerlendirilir.
- Yeniden yazılan `content`, daha sonraki bir hook
  teslimatı iptal etmediği sürece daha düşük öncelikli hook'lara devam eder.
- `reply_payload_sending`, yük normalleştirmesinden sonra ve
  kaynak kanala geri yönlendirilen yanıtlar dâhil olmak üzere kanal
  teslimatından önce çalışır. İşleyiciler sırayla çalışır ve her işleyici,
  daha yüksek öncelikli işleyicilerin ürettiği en son yükü görür.
- `reply_payload_sending` yükleri, `trustedLocalMedia` gibi çalışma
  zamanı güven işaretlerini kullanıma sunmaz; plugin'ler yükün yapısını
  düzenleyebilir ancak yerel medya güveni veremez.
- `message_sending`, bir iptalle birlikte `cancelReason`
  ve sınırlandırılmış `metadata` döndürebilir. Yeni ileti yaşam
  döngüsü API'leri bunu `cancelled_by_message_sending_hook` nedenine sahip engellenmiş bir
  teslimat sonucu olarak kullanıma sunar; eski doğrudan teslimat, uyumluluk
  için boş bir sonuç dizisi döndürmeye devam eder.
- `message_sent` yalnızca gözlem amaçlıdır. İşleyici
  hataları günlüğe kaydedilir ve teslimat sonucunu değiştirmez.

## Hook'ları yükleme

Operatöre ait izin verme/engelleme kararları için `security.installPolicy`
kullanın. Bu politika OpenClaw yapılandırmasından çalışır, CLI yükleme ve
güncelleme yollarını kapsar ve etkinleştirilmiş ancak kullanılamaz olduğunda
kapalı durumda başarısız olur.

`before_install`, plugin çalışma zamanı yaşam döngüsü hook'udur. Yalnızca
Gateway destekli yükleme akışları gibi plugin hook'larının önceden yüklenmiş
olduğu OpenClaw işleminde `security.installPolicy` sonrasında çalışır. Plugin'e ait
gözlemler, uyarılar ve uyumluluk denetimleri için kullanışlıdır ancak
yüklemelerdeki birincil kurumsal veya ana makine güvenlik sınırı değildir.
`builtinScan` alanı uyumluluk amacıyla olay yükünde kalır ancak OpenClaw
artık yükleme zamanında yerleşik tehlikeli kod engellemesi çalıştırmadığından
boş bir `ok` sonucudur. Bu işlemde yüklemeyi durdurmak için ek
bulgular veya `{ block: true, blockReason }` döndürün.

`block: true` sonlandırıcıdır. `block: false` karar verilmemiş olarak
değerlendirilir. İşleyici hataları, kapalı durumda başarısız olarak yüklemeyi
engeller.

## Gateway yaşam döngüsü

Genel plugin hizmetlerini başlatmak için `gateway_start`, uzun süre çalışan
kaynakları temizlemek için `gateway_stop` kullanın. `gateway_start`
çalışırken cron zamanlayıcısı hâlâ yükleniyor olabilir; bu nedenle bunu harici
bir cron izdüşümü için temel sinyal olarak kullanmayın.

Plugin'e ait çalışma zamanı hizmetleri için dahili `gateway:startup`
hook'una güvenmeyin.

`cron_reconciled`, Gateway cron zamanlayıcısı ve çıkış sırasında çalışan
izleyicileri kalıcı durumlarını uzlaştırdıktan sonra tetiklenir. Hem ilk
başlatmada hem de yapılandırma yeniden yüklenirken zamanlayıcı değiştirildiğinde
tetiklenir. Olay, `reason` (`startup` veya
`reload`) ve etkin `enabled` durumunu bildirir. Devre dışı
cron bile `enabled: false` ile olay yayarak harici bir izdüşümün eski
uyandırmaları temizlemesine olanak tanır. Uzlaştırmayı tamamlayan tam zamanlayıcı
örneği için `ctx.getCron?.()` kullanın; daha sonraki bir yeniden yükleme bu
geri çağrıyı yeniden hedeflemez. `ctx.abortSignal` aynı zamanlayıcı anlık
görüntüsüne sahiptir. Gateway, daha yeni bir zamanlayıcı hazırlanır hazırlanmaz
veya kapatma başladığında bunu iptal eder. Bunu her kalıcı yan etkiye iletin ve
iptal edildikten sonra anlık görüntüyü kabul etmeyin. Bu, bir plugin
etkinleştirme sinyali değil, zamanlayıcı yaşam döngüsü sinyalidir: yalnızca
plugin'e yönelik çalışırken yeniden yükleme bunu yeniden yürütmez. Yeni
etkinleştirilen bir tüketici, ilk temelini bir sonraki zamanlayıcı değişiminde
veya Gateway başlangıcında alır.

Diğer gözlem hook'larında olduğu gibi `gateway_start` ve
`cron_reconciled` geri çağrıları çakışabilir. Her iki işleyici de plugin
başlatma işlemini paylaşıyorsa geri çağrı sırasına bağlı kalmak yerine bunları
plugin'e yerel bir hazır olma promise'ı ile koordine edin.

`cron_changed`, `added`, `updated`,
`removed`, `started`, `finished` ve
`scheduled` nedenlerini kapsayan türü belirlenmiş bir olay yüküyle
Gateway'e ait cron yaşam döngüsü olayları için tetiklenir. Olay, bir
`not-requested` | `delivered` | `not-delivered` |
`unknown` türündeki `PluginHookGatewayCronDeliveryStatus` ile birlikte bir
`PluginHookGatewayCronJob` anlık görüntüsü (mevcut olduğunda `state.nextRunAtMs`,
`state.lastRunStatus` ve `state.lastError` dâhil) taşır. Kaldırılan olaylar
işlem sonrasıdır: yalnızca kalıcı silme başarıyla tamamlandıktan sonra
tetiklenir ve harici zamanlayıcıların durumu uzlaştırabilmesi için silinen iş
anlık görüntüsünü taşımaya devam eder.

Bir `scheduled` olayı işlem sonrası gerçekleşir: yalnızca başarılı bir
kalıcı yazma işlemi mevcut bir işin etkin `nextRunAtMs` değerini
değiştirdikten sonra, o işin açık `added`, `updated` veya
`removed` yaşam döngüsü olayı hariç tutularak tetiklenir. Üst düzey
`event.nextRunAtMs`, kaydedilmiş sonraki uyandırmadır; mevcut değilse işin
sonraki uyandırması yoktur. Bu olayları sıralı bir fark günlüğü değil,
uzlaştırma ipuçları olarak değerlendirin. Bunları, `cron_reconciled`
tarafından en son yakalanan zamanlayıcıyı yeniden okumaya yönelik
birleştirilebilir ipuçları olarak kullanın; zamanlayıcıyı bir
`cron_changed` bağlamından devralmayın. Zamanı gelen denetimler ve yürütme
için doğruluk kaynağı olarak OpenClaw'ı koruyun.

### Güvenli harici cron izdüşümü

Cron olay farklarını iletmek yerine tam bir uyandırma anlık görüntüsünün
izdüşümünü oluşturun. Harici bağdaştırıcının `replaceAll` işlemi atomik
ve eşgüçlü olmalı, ayrıca yalnızca ana makine anlık görüntüyü kalıcı olarak
kabul ettikten sonra çözümlenmelidir. Sağlanan iptal sinyaline de uymalıdır:
sinyal kalıcı kabulden önce iptal edilirse bağdaştırıcı bu anlık görüntüyü
kabul etmemelidir.

Bu desen, en son durum için tek bir çalışan işlemi yürütme hâlinde tutar.
Yalnızca `cron_reconciled` bir zamanlayıcı örneğini devralır;
`cron_changed` ise yalnızca bu çalışan işlemden yetkili örneği yeniden
okumasını ister; böylece geç gelen bir ipucu eski bir zamanlayıcıyı geri
yükleyemez. Daha yeni bir revizyon, eski bir anlık görüntüyü kabul edemeden
önce etkin ana makine girişimini iptal eder.

```typescript
import { setTimeout as sleep } from "node:timers/promises";
import type { OpenClawPluginApi } from "openclaw/plugin-sdk/plugin-entry";

type ExternalWake = { jobId: string; runAtMs: number };

type ExternalWakeHost = {
  replaceAll(wakes: readonly ExternalWake[], options: { signal: AbortSignal }): Promise<void>;
  close(): Promise<void>;
};

type CronReader = {
  list(options: { includeDisabled: true }): Promise<
    Array<{
      id: string;
      enabled?: boolean;
      state?: { nextRunAtMs?: number };
    }>
  >;
};

export function registerCronProjection(api: OpenClawPluginApi, host: ExternalWakeHost) {
  const lifecycle = new AbortController();
  let cron: CronReader | undefined;
  let enabled = false;
  let hasBaseline = false;
  let reconciliationSignal: AbortSignal | undefined;
  let requestedRevision = 0;
  let appliedRevision = 0;
  let worker = Promise.resolve();
  let activeAttempt: AbortController | undefined;

  const projectLatest = async () => {
    let retryMs = 1_000;

    while (!lifecycle.signal.aborted && appliedRevision < requestedRevision) {
      const ownerSignal = reconciliationSignal;
      if (!ownerSignal || ownerSignal.aborted) {
        return;
      }
      const targetRevision = requestedRevision;
      const attempt = new AbortController();
      const signal = AbortSignal.any([lifecycle.signal, ownerSignal, attempt.signal]);
      activeAttempt = attempt;

      try {
        const jobs = enabled && cron ? await cron.list({ includeDisabled: true }) : [];
        if (signal.aborted || targetRevision !== requestedRevision) {
          continue;
        }
        const wakes = jobs
          .flatMap((job): ExternalWake[] => {
            const runAtMs = job.enabled === false ? undefined : job.state?.nextRunAtMs;
            return runAtMs === undefined ? [] : [{ jobId: job.id, runAtMs }];
          })
          .sort((a, b) => a.runAtMs - b.runAtMs || a.jobId.localeCompare(b.jobId));

        await host.replaceAll(wakes, { signal });
        if (signal.aborted || targetRevision !== requestedRevision) {
          continue;
        }
        appliedRevision = targetRevision;
        retryMs = 1_000;
      } catch {
        if (lifecycle.signal.aborted || ownerSignal.aborted) {
          return;
        }
        if (attempt.signal.aborted) {
          continue;
        }
        api.logger.warn(`harici cron izdüşümü başarısız oldu; ${retryMs}ms sonra yeniden deneniyor`);
        try {
          await sleep(retryMs, undefined, { signal });
        } catch {
          if (lifecycle.signal.aborted) {
            return;
          }
          if (attempt.signal.aborted) {
            continue;
          }
        }
        retryMs = Math.min(retryMs * 2, 30_000);
      } finally {
        if (activeAttempt === attempt) {
          activeAttempt = undefined;
        }
      }
    }
  };

  const requestProjection = () => {
    const targetRevision = ++requestedRevision;
    activeAttempt?.abort();
    worker = worker.then(async () => {
      if (!lifecycle.signal.aborted && appliedRevision < targetRevision) {
        await projectLatest();
      }
    });
    return worker;
  };

  api.on("cron_reconciled", (event, ctx) => {
    const reconciledCron = ctx.getCron?.();
    if (event.enabled && !reconciledCron) {
      api.logger.warn("cron uzlaştırması bir zamanlayıcıyı kullanıma sunmadı");
      return;
    }
    cron = reconciledCron;
    enabled = event.enabled;
    hasBaseline = true;
    reconciliationSignal = ctx.abortSignal;
    return requestProjection();
  });

  api.on("cron_changed", () => {
    if (hasBaseline) {
      return requestProjection();
    }
  });

  api.on("gateway_stop", async () => {
    lifecycle.abort();
    await worker;
    await host.close();
  });
}
```

`cron_reconciled`, `enabled: false` bildirdiğinde aynı yol
`replaceAll([])` çağrısını yapar ve eski harici uyandırmaları temizler. Bu
örnekteki yeniden deneme/geri çekilme işlem yerelidir ve çalışma zamanı
bağdaştırıcısı hatalarını geçici olarak değerlendirir; yeniden denenemeyen
yapılandırmayı kayıttan önce doğrulayın. OpenClaw, plugin hook etkileri için
bir giden kutusu sağlamaz. İşlem kalıcı kabulden önce sonlanırsa bir sonraki
Gateway başlangıcı yeni bir yetkili `cron_reconciled` anlık görüntüsü yayar.
`gateway_stop`, yürütülmekte olan ana makine çalışmasını iptal eder,
çalışanın durulmasını bekler ve ardından bağdaştırıcıyı kapatır.

## Yaklaşan kullanımdan kaldırmalar

Hook'larla bağlantılı birkaç yüzey kullanımdan kaldırılmıştır ancak hâlâ
desteklenmektedir. Bir sonraki ana sürümden önce geçiş yapın:

- `inbound_claim` ve `message_received`
  işleyicilerindeki **düz metin kanal zarfları**. Düz zarf metnini ayrıştırmak yerine
  `BodyForAgent` ve yapılandırılmış kullanıcı bağlamı bloklarını okuyun. Bkz.
  [Düz metin kanal zarfları → BodyForAgent](/tr/plugins/sdk-migration#active-deprecations).
- **`subagent_spawning`**, eski pluginlerle uyumluluk için korunmaktadır ancak
  yeni pluginler bundan iş parçacığı yönlendirmesi döndürmemelidir. Çekirdek,
  `subagent_spawned` tetiklenmeden önce kanal oturum bağlama adaptörleri aracılığıyla
  `thread: true` alt ajan bağlamalarını hazırlar.
- **`deactivate`**, 2026-08-16 sonrasına kadar kullanımdan kaldırılmış bir
  temizleme uyumluluk takma adı olarak korunmaktadır. Yeni pluginler `gateway_stop`
  kullanmalıdır.
- **`before_tool_call` içindeki `onResolution`**, artık serbest biçimli bir
  `string` yerine türü belirlenmiş `PluginApprovalResolution` birleşimini
  (`allow-once` / `allow-always` / `deny` /
  `timeout` / `cancelled`) kullanır.
- **`api.registerSessionExtension` / `api.enqueueNextTurnInjection`**,
  üst düzey uyumluluk takma adları olarak korunmaktadır. Yeni pluginler
  `api.session.state.registerSessionExtension(...)` ve
  `api.session.workflow.enqueueNextTurnInjection(...)` kullanmalıdır.

Bellek yeteneği kaydı, sağlayıcı düşünme profili, harici kimlik doğrulama
sağlayıcıları, sağlayıcı keşif türleri, görev çalışma zamanı erişimcileri ve
`command-auth` → `command-status` yeniden adlandırması dahil tam liste için
bkz. [Plugin SDK geçişi → Etkin kullanımdan kaldırmalar](/tr/plugins/sdk-migration#active-deprecations).

## İlgili

- [Plugin SDK geçişi](/tr/plugins/sdk-migration) - etkin kullanımdan kaldırmalar ve kaldırma zaman çizelgesi
- [Plugin oluşturma](/tr/plugins/building-plugins)
- [Plugin SDK genel bakışı](/tr/plugins/sdk-overview)
- [Plugin giriş noktaları](/tr/plugins/sdk-entrypoints)
- [Dahili kancalar](/tr/automation/hooks)
- [Plugin mimarisinin iç işleyişi](/tr/plugins/architecture-internals)
