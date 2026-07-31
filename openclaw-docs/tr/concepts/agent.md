---
read_when:
    - Aracı çalışma zamanı, çalışma alanı önyüklemesi veya oturum davranışını değiştirme
summary: Ajan çalışma zamanı, çalışma alanı sözleşmesi ve oturum önyüklemesi
title: Ajan çalışma zamanı
x-i18n:
    generated_at: "2026-07-26T23:55:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4d3dd9c0c65e4ccd791a2a6131f1b7457c8cfee6da71502d93c355280e094390
    source_path: concepts/agent.md
    workflow: 16
---

OpenClaw tek bir **gömülü ajan çalışma zamanı** sunar: harici bir
yürütme sürecine turları devretmekten farklı olarak yerleşik bir ajan döngüsü, araç
bağlantıları ve istem derlemesi. Yapılandırılmış her ajanın (birden fazla ajanı çalıştırmak
için [Çoklu ajan yönlendirmesi](/tr/concepts/multi-agent) bölümüne bakın) kendi çalışma alanı,
önyükleme dosyaları ve oturum deposu vardır. Bu sayfa söz konusu çalışma zamanı sözleşmesini
kapsar: çalışma alanının neleri içermesi gerektiğini, hangi dosyaların eklendiğini ve
oturumların buna göre nasıl önyüklendiğini.

## Çalışma alanı (zorunlu)

Her ajan, araçlar ve bağlam için **tek** çalışma dizini (`cwd`)
olarak tek bir çalışma alanı dizini (`agents.defaults.workspace` veya ajan başına
`agents.entries.*.workspace`) kullanır.

Önerilen: Eksikse `~/.openclaw/openclaw.json` oluşturmak ve çalışma alanı dosyalarını başlatmak için `openclaw setup` kullanın.

Tam çalışma alanı düzeni ve yedekleme kılavuzu: [Ajan çalışma alanı](/tr/concepts/agent-workspace)

`agents.defaults.sandbox` etkinse ana olmayan oturumlar bunu
`agents.defaults.sandbox.workspaceRoot` altındaki oturum başına çalışma alanlarıyla geçersiz kılabilir (bkz.
[Gateway yapılandırması](/tr/gateway/configuration)).

## Önyükleme dosyaları (eklenen)

OpenClaw, çalışma alanının içinde kullanıcı tarafından düzenlenebilen şu dosyaları bekler:

| Dosya           | Amaç                                              |
| -------------- | ---------------------------------------------------- |
| `AGENTS.md`    | Çalışma talimatları + "bellek"                    |
| `SOUL.md`      | Kişilik, sınırlar, üslup                            |
| `TOOLS.md`     | Kullanıcının yönettiği araç notları ve kuralları           |
| `IDENTITY.md`  | Ajan adı/tarzı/emojisi                                |
| `USER.md`      | Kullanıcı profili + tercih edilen hitap şekli                     |
| `HEARTBEAT.md` | Heartbeat'e özgü talimatlar                      |
| `BOOTSTRAP.md` | Tek seferlik ilk çalıştırma ritüeli (tamamlandıktan sonra silinir) |
| `MEMORY.md`    | Varsa kök uzun süreli bellek dosyası               |

Yeni bir oturumun ilk turunda OpenClaw, bu dosyaların içeriklerini sistem isteminin Proje Bağlamına ekler. `MEMORY.md` yalnızca çalışma alanı kökünde mevcutsa eklenir.

Boş dosyalar atlanır. İstemlerin yalın kalması için büyük dosyalar kısaltılır ve bir işaretleyiciyle kesilir (tam içerik için dosyayı okuyun). Eksik bir dosya (`MEMORY.md` dışında) bunun yerine tek bir "eksik dosya" işaretleyici satırı ekler; `openclaw setup` bunun için güvenli bir varsayılan şablon oluşturur.

`BOOTSTRAP.md` yalnızca **tamamen yeni bir çalışma alanı** için oluşturulur (başka önyükleme dosyası yoksa). Beklemede olduğu sürece OpenClaw bunu Proje Bağlamında tutar ve kullanıcı mesajına kopyalamak yerine ilk ritüel için sistem istemine önyükleme rehberliği ekler. Ritüeli tamamladıktan sonra silerseniz sonraki yeniden başlatmalarda yeniden oluşturulmaz.

Bir çalışma alanı gözlemlendikten sonra OpenClaw, kurulum durumunu ve
doğrulamasını `~/.openclaw/state/openclaw.sqlite` konumundaki paylaşılan SQLite
veritabanında saklar. Yakın zamanda doğrulanmış bir çalışma alanı
kaybolursa veya silinirse başlangıç, `BOOTSTRAP.md` dosyasını sessizce yeniden oluşturmayı reddeder;
çalışma alanını geri yükleyin veya çalışma alanı ile veritabanı durumunun
birlikte temizlenmesi için tam bir ilk kurulum sıfırlaması kullanın.

Eski sürümler çalışma alanı JSON dosyalarını ve `.attested` yardımcı dosyalarını kullanıyordu. Çalışma zamanı
bu dosyaları okumaz. Bunları doğrulamak, durumlarını SQLite'a aktarmak ve
aktarılan satırlar doğrulandıktan sonra her kaynak dosyayı kaldırmak için `openclaw doctor --fix` çalıştırın.

Önceden hazırlanmış çalışma alanlarında önyükleme dosyası oluşturmayı tamamen devre dışı bırakmak için şunu ayarlayın:

```json5
{ agents: { defaults: { skipBootstrap: true } } }
```

## Yerleşik araçlar

Temel araçlar (okuma/yürütme/düzenleme/yazma ve ilgili sistem araçları), araç
politikasına tabi olarak her zaman kullanılabilir.
`apply_patch`, OpenAI modellerinde varsayılan olarak açıktır ve
`tools.exec.applyPatch` (`enabled`, `workspaceOnly`, `allowModels`) tarafından denetlenir. `TOOLS.md` hangi araçların mevcut olduğunu **denetlemez**;
bu, araçların nasıl kullanılmasını _istediğinize_ ilişkin rehberliktir.

## Skills

OpenClaw, becerileri şu konumlardan yükler (öncelik sırası en yüksekten en düşüğe):

- Çalışma alanı: `<workspace>/skills`
- Proje ajanı becerileri: `<workspace>/.agents/skills`
- Kişisel ajan becerileri: `~/.agents/skills`
- Yönetilen/yerel: `~/.openclaw/skills`
- Paketle birlikte gelenler (kurulumla sunulur)
- Ek beceri klasörleri: `skills.load.extraDirs`

Beceri kökleri `<workspace>/skills/personal/foo/SKILL.md` gibi gruplanmış klasörler içerebilir;
beceri yine de düz frontmatter adıyla sunulur; örneğin `foo`.

Beceriler yapılandırma/ortam tarafından kısıtlanabilir ([Gateway yapılandırması](/tr/gateway/configuration) içindeki `skills` bölümüne bakın).

## Çalışma zamanı sınırları

Gömülü ajan çalışma zamanı OpenClaw tarafından yönetilir: model keşfi, araç bağlantıları,
istem derlemesi, oturum yönetimi ve kanal teslimi tek bir bütünleşik
çalışma zamanı yüzeyini paylaşır.

## Oturumlar

Oturum satırları ajan başına SQLite veritabanında saklanır:

- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`

Transkript JSONL dosyaları; eski geçiş girdileri, silinmiş veya
sıfırlanmış arşivler, içe aktarmalar, dışa aktarmalar ve destek eserleri olarak
`~/.openclaw/agents/<agentId>/sessions/` altında bulunmaya devam edebilir. Etkin ajan geçmişi,
oturum satırlarıyla birlikte SQLite'ta saklanır. Oturum kimliği kararlıdır ve
OpenClaw tarafından seçilir. OpenClaw diğer araçların oturum klasörlerini okumaz.

## Akış sırasında yönlendirme

Çalışmanın ortasında gelen istemler varsayılan olarak mevcut çalışmaya yönlendirilir.
Yönlendirme, **mevcut asistan turu araç çağrılarını yürütmeyi bitirdikten sonra**,
bir sonraki LLM çağrısından önce teslim edilir ve artık mevcut asistan mesajındaki
kalan araç çağrılarını atlamaz.

`/queue steer` varsayılan etkin çalışma davranışıdır. `/queue followup` ve
`/queue collect`, mesajların yönlendirilmek yerine sonraki bir turu beklemesini sağlar.
`/queue interrupt` ise etkin çalışmayı iptal eder. Kuyruk ve sınır davranışı için
[Kuyruk](/tr/concepts/queue) ve [Yönlendirme kuyruğu](/tr/concepts/queue-steering) bölümlerine bakın.

Blok akışı, tamamlanan asistan bloklarını biter bitmez gönderir; bu özellik
**varsayılan olarak kapalıdır** (`agents.defaults.blockStreamingDefault: "off"`).
Sınırı `agents.defaults.blockStreamingBreak` üzerinden ayarlayın (`text_end` veya `message_end`; varsayılan `text_end`).
Esnek blok parçalamayı `agents.defaults.blockStreamingChunk` ile denetleyin (varsayılan
800-1200 karakterdir; önce paragraf sonlarını, ardından yeni satırları, son olarak cümleleri tercih eder).
Tek satırlık mesaj yığınını azaltmak için akış parçalarını `agents.defaults.blockStreamingCoalesce` ile
birleştirin (göndermeden önce boşta kalma süresine dayalı birleştirme). Telegram dışındaki kanallarda
blok yanıtlarını etkinleştirmek için açıkça `*.streaming.block.enabled: true` gerekir (QQ Bot ise
`channels.qqbot.streaming.mode` değeri `"off"` olmadığı sürece blok yanıtlarını akışla gönderir).
Ayrıntılı araç özetleri araç başlangıcında yayımlanır (geciktirme yoktur); Control UI
kullanılabilir olduğunda araç çıktısını ajan olayları üzerinden akışla gönderir.
Daha fazla ayrıntı: [Akış + parçalama](/tr/concepts/streaming).

## Model referansları

Yapılandırmadaki model referansları (örneğin `agents.defaults.model` ve `agents.defaults.models`), **ilk** `/` karakterinden bölünerek ayrıştırılır.

- Modelleri yapılandırırken `provider/model` kullanın.
- Model kimliği `/` içeriyorsa (OpenRouter tarzı), sağlayıcı önekini ekleyin (örnek: `openrouter/moonshotai/kimi-k2`).
- Sağlayıcıyı belirtmezseniz OpenClaw önce bir diğer adı, ardından tam olarak bu model
  kimliği için yapılandırılmış sağlayıcılar arasında benzersiz bir eşleşmeyi dener ve ancak
  bundan sonra yapılandırılmış varsayılan sağlayıcıya geri döner. Bu sağlayıcı artık
  yapılandırılmış varsayılan modeli sunmuyorsa OpenClaw, kaldırılmış bir sağlayıcıya ait eski
  varsayılanı göstermek yerine yapılandırılmış ilk sağlayıcıya/modele geri döner.

## Yapılandırma (asgari)

En azından şunları ayarlayın:

- `agents.defaults.workspace`
- `channels.whatsapp.allowFrom` (önemle önerilir)

## İlgili

- [Ajan çalışma alanı](/tr/concepts/agent-workspace)
- [Çoklu ajan yönlendirmesi](/tr/concepts/multi-agent)
- [Oturum yönetimi](/tr/concepts/session)
- [Grup sohbetleri](/tr/channels/group-messages)
