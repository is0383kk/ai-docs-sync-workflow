---
read_when: You hit 'sandbox jail' or see a tool/elevated refusal and want the exact config key to change.
status: active
summary: 'Bir aracın engellenme nedenleri: korumalı alan çalışma zamanı, araç izin/ret politikası ve yükseltilmiş yürütme geçitleri'
title: Sandbox ile araç ilkesi ile yükseltilmiş yetkiler arasındaki farklar
x-i18n:
    generated_at: "2026-07-26T23:21:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4da521215fe55bf2774008a53d896d5c00b8babcbca2005dc4593ebfebc5343
    source_path: gateway/sandbox-vs-tool-policy-vs-elevated.md
    workflow: 16
---

OpenClaw birbiriyle ilişkili ancak farklı üç denetime sahiptir:

1. **Korumalı alan** (`agents.defaults.sandbox.*` / `agents.entries.*.sandbox.*`), **araçların nerede çalışacağını** (korumalı alan arka ucu veya ana makine) belirler.
2. **Araç politikası** (`tools.*`, `tools.sandbox.tools.*`, `agents.entries.*.tools.*`), **hangi araçların kullanılabilir/izinli olduğunu** belirler.
3. **Yükseltilmiş** (`tools.elevated.*`, `agents.entries.*.tools.elevated.*`), korumalı alandayken korumalı alan dışında çalıştırmaya yönelik **yalnızca exec için bir kaçış mekanizmasıdır** (varsayılan olarak `gateway` veya exec hedefi `node` olarak yapılandırıldığında `node`).

## Hızlı hata ayıklama

OpenClaw'ın _gerçekte_ ne yaptığını görmek için inceleyiciyi kullanın:

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

Şunları yazdırır:

- etkin korumalı alan modu/kapsamı/çalışma alanı erişimi
- oturumun şu anda korumalı alanda olup olmadığı (ana ve ana olmayan)
- etkin korumalı alan araç izin/ret kuralları (ve bunların ajan/genel/varsayılan kaynağından hangisinden geldiği)
- yükseltilmiş erişim kapıları ve düzeltme anahtarı yolları

## Korumalı alan: araçların çalıştığı yer

Korumalı alan kullanımı `agents.defaults.sandbox.mode` tarafından denetlenir:

- `"off"`: her şey ana makinede çalışır.
- `"non-main"`: yalnızca ana olmayan oturumlar korumalı alanda çalışır (gruplar/kanallar için sık karşılaşılan bir "sürpriz").
- `"all"`: her şey korumalı alanda çalışır.

`agents.defaults.sandbox.workspaceAccess`, korumalı alanın neleri görebileceğini denetler: `"none"`, `"ro"` veya `"rw"`.

Tam matris (kapsam, çalışma alanı bağlamaları, imajlar) için [Korumalı Alan Kullanımı](/tr/gateway/sandboxing) bölümüne bakın.

### Bağlama bağlamaları (hızlı güvenlik denetimi)

- `docker.binds`, korumalı alan dosya sistemini _deler_: bağladığınız her şey, ayarladığınız modla (`:ro` veya `:rw`) kapsayıcının içinde görünür.
- Modu belirtmezseniz varsayılan okuma-yazmadır; kaynak kod/gizli bilgiler için `:ro` tercih edin.
- `scope: "shared"`, ajan başına bağlamaları yok sayar (yalnızca genel bağlamalar uygulanır).
- OpenClaw bağlama kaynaklarını iki kez doğrular: önce normalleştirilmiş kaynak yolunda, ardından mevcut en derin üst dizin üzerinden çözümledikten sonra tekrar. Sembolik bağlantılı üst dizinlerden kaçış, engellenen yol veya izin verilen kök denetimlerini atlatmaz.
- Mevcut olmayan uç yollar da güvenli biçimde denetlenir. `/workspace/alias-out/new-file`, sembolik bağlantılı bir üst dizin üzerinden engellenen bir yola veya yapılandırılmış izin verilen köklerin dışına çözümlenirse bağlama reddedilir.
- `/var/run/docker.sock` bağlamak, ana makinenin denetimini fiilen korumalı alana verir; bunu yalnızca bilinçli olarak yapın.
- Çalışma alanı erişimi (`workspaceAccess`), bağlama modlarından bağımsızdır.

Birden fazla ana makine klasörü, erişim modu ve harici kaynak güvenliği için açık onay içeren ajan başına yapılandırma örneği için [Bir ajan için birden fazla klasör](/tr/gateway/sandboxing#multiple-folders-for-one-agent) bölümüne bakın.

## Araç politikası: hangi araçların mevcut/çağrılabilir olduğu

İki katman önemlidir:

- **Araç profili**: `tools.profile` ve `agents.entries.*.tools.profile` (temel izin listesi)
- **Sağlayıcı araç profili**: `tools.byProvider[provider].profile` ve `agents.entries.*.tools.byProvider[provider].profile`
- **Genel/ajan başına araç politikası**: `tools.allow`/`tools.deny` ve `agents.entries.*.tools.allow`/`agents.entries.*.tools.deny`
- **Sağlayıcı araç politikası**: `tools.byProvider[provider].allow/deny` ve `agents.entries.*.tools.byProvider[provider].allow/deny`
- **Korumalı alan araç politikası** (yalnızca korumalı alandayken uygulanır): `tools.sandbox.tools.allow`/`tools.sandbox.tools.deny` ve `agents.entries.*.tools.sandbox.tools.*`

Genel kurallar:

- `deny` her zaman önceliklidir.
- `allow` boş değilse diğer her şey engellenmiş kabul edilir.
- Araç politikası kesin engeldir: `/exec`, reddedilmiş bir `exec` aracını geçersiz kılamaz.
- Araç politikası araç kullanılabilirliğini ada göre filtreler; `exec` içindeki yan etkileri incelemez. `exec` izinliyse `write`, `edit` veya `apply_patch` öğelerinin reddedilmesi kabuk komutlarını salt okunur hâle getirmez.
- `/exec`, yalnızca yetkili gönderenler için oturum varsayılanlarını değiştirir; araç erişimi vermez.
- Sağlayıcı araç anahtarları `provider` (ör. `google-antigravity`) veya `provider/model` (ör. `openai/gpt-5.4`) kabul eder.
- Bir araç politikası adımı araçları kaldırdığında veya korumalı alan araç politikası bir çağrıyı engellediğinde Gateway günlükleri `agents/tool-policy` denetim girdileri içerir. Kural etiketini, yapılandırma anahtarını ve etkilenen araç adlarını görmek için `openclaw logs` kullanın.

### Araç grupları (kısaltmalar)

Araç politikaları (genel, ajan, korumalı alan), birden fazla araca genişletilen `group:*` girdilerini destekler:

```json5
{
  tools: {
    sandbox: {
      tools: {
        allow: ["group:runtime", "group:fs", "group:sessions", "group:memory"],
      },
    },
  },
}
```

Kullanılabilir gruplar:

| Grup               | Araçlar                                                                                                                                                                                                                                                |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash`, `exec` için bir diğer ad olarak kabul edilir)                                                                                                                                                              |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `spawn_task`, `dismiss_task` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`, `screen`, `terminal`, `canvas`, `show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`, `cron`, `gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`, `computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`, `image_generate`, `music_generate`, `video_generate`, `tts`                                                                                                                                                                                   |
| `group:openclaw`   | yerleşik OpenClaw araçlarının çoğu (`read`/`write`/`edit`/`apply_patch`/`exec`/`process` dosya sistemi ve çalışma zamanı temel öğeleri, `canvas` ve sağlayıcı pluginleri hariç)                                                                                             |
| `group:plugins`    | `bundle-mcp` üzerinden sunulan yapılandırılmış MCP sunucuları dâhil, yüklenen ve pluginlerin sahip olduğu tüm araçlar                                                                                                                                                           |

Salt okunur ajanlar için, korumalı alan dosya sistemi politikası veya ayrı bir ana makine sınırı salt okunur kısıtlamasını uygulamıyorsa dosya sistemini değiştiren araçların yanı sıra `group:runtime` öğesini de reddedin.

Korumalı alandaki MCP sunucuları için korumalı alan araç politikası ikinci bir izin kapısıdır. `mcp.servers` yapılandırılmış olmasına rağmen korumalı alandaki turlarda yalnızca yerleşik araçlar görünüyorsa `bundle-mcp`, `group:plugins` veya `outlook__send_mail` ya da `outlook__*` gibi sunucu ön ekli bir MCP araç adı/glob ifadesini `tools.sandbox.tools.alsoAllow` öğesine ekleyin; ardından Gateway'i yeniden başlatın/yeniden yükleyin ve araç listesini yeniden yakalayın. Sunucu glob ifadeleri, sağlayıcı açısından güvenli MCP sunucu ön ekini kullanır: `[A-Za-z0-9_-]` olmayan karakterler `-` olur, harfle başlamayan adlara `mcp-` ön eki eklenir ve uzun veya yinelenen ön ekler kısaltılabilir ya da son ek alabilir.

`openclaw doctor`, şu anda `mcp.servers` içindeki OpenClaw tarafından yönetilen sunucular için bu yapıyı denetler. Paketlenmiş plugin manifestlerinden veya Claude `.mcp.json` üzerinden yüklenen MCP sunucuları aynı korumalı alan kapısını kullanır ancak bu tanılama henüz bu kaynakları listelemez; araçları korumalı alan turlarında kaybolursa aynı izin listesi girdilerini kullanın.

## Yükseltilmiş: yalnızca exec için "ana makinede çalıştır"

Yükseltilmiş erişim ek araçlar **vermez**; yalnızca `exec` öğesini etkiler.

- Korumalı alandaysanız `/elevated on` (veya `elevated: true` ile `exec`) korumalı alan dışında çalışır (onaylar yine de geçerli olabilir).
- Oturum için exec onaylarını atlamak üzere `/elevated full` kullanın.
- Zaten doğrudan çalışıyorsanız yükseltilmiş erişim fiilen etkisizdir (kapılara yine tabidir).
- Yükseltilmiş erişim **beceri kapsamlı değildir** ve araç izin/ret kurallarını **geçersiz kılmaz**.
- Yükseltilmiş erişim, `host=auto` üzerinden rastgele makineler arası geçersiz kılma yetkisi vermez; normal exec hedefi kurallarını izler ve `node` öğesini yalnızca yapılandırılmış/oturum hedefi zaten `node` olduğunda korur.
- `/exec`, yükseltilmiş erişimden ayrıdır. Yalnızca yetkili gönderenler için oturum başına exec varsayılanlarını ayarlar.

Kapılar:

- Etkinleştirme: `tools.elevated.enabled` (ve isteğe bağlı olarak `agents.entries.*.tools.elevated.enabled`)
- Gönderen izin listeleri: `tools.elevated.allowFrom.<provider>` (ve isteğe bağlı olarak `agents.entries.*.tools.elevated.allowFrom.<provider>`)

[Yükseltilmiş Mod](/tr/tools/elevated) bölümüne bakın.

## Yaygın "korumalı alan hapishanesi" düzeltmeleri

### "X aracı korumalı alan araç politikası tarafından engellendi"

Düzeltme anahtarları (birini seçin):

- Korumalı alanı devre dışı bırakın: `agents.defaults.sandbox.mode=off` (veya ajan başına `agents.entries.*.sandbox.mode=off`)
- Araca korumalı alan içinde izin verin:
  - aracı `tools.sandbox.tools.deny` listesinden kaldırın (veya ajan başına `agents.entries.*.tools.sandbox.tools.deny`)
  - ya da `tools.sandbox.tools.allow` listesine ekleyin (veya ajan başına izin verin)
- `agents/tool-policy` girdisi için `openclaw logs` öğesini kontrol edin. Bu öğe, korumalı alan modunu ve izin ya da ret kuralının aracı engelleyip engellemediğini kaydeder.

### "Bunun ana oturum olduğunu sanıyordum, neden korumalı alanda?"

`"non-main"` modunda grup/kanal anahtarları ana oturum _değildir_. Ana oturum anahtarını (`sandbox explain` tarafından gösterilir) kullanın veya modu `"off"` olarak değiştirin.

## İlgili

- [Korumalı Alan](/tr/gateway/sandboxing) -- korumalı alanın tam başvurusu (modlar, kapsamlar, arka uçlar, imajlar)
- [Çok Ajanlı Korumalı Alan ve Araçlar](/tr/tools/multi-agent-sandbox-tools) -- ajan başına geçersiz kılmalar ve öncelik
- [Yükseltilmiş Mod](/tr/tools/elevated)
