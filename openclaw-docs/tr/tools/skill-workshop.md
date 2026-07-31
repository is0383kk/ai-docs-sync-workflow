---
read_when:
    - Agentın sohbet üzerinden bir skill oluşturmasını veya güncellemesini istiyorsunuz
    - Oluşturulan bir beceri taslağını incelemeniz, uygulamanız, reddetmeniz veya karantinaya almanız gerekiyor
    - Skill Workshop onayını, özerkliğini, depolamasını veya sınırlarını yapılandırıyorsunuz
    - Kendi kendine öğrenme önerilerinin nerede incelendiğini anlamak istiyorsunuz
sidebarTitle: Skill Workshop
summary: Skill Workshop incelemesi aracılığıyla çalışma alanı Skills oluşturun ve güncelleyin
title: Skill Atölyesi
x-i18n:
    generated_at: "2026-07-26T23:06:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c2590f2a1bcad3b22ef8504eac7b3a44611c3fedc0df3832660f8926ce04252
    source_path: tools/skill-workshop.md
    workflow: 16
---

Skill Workshop, çalışma alanı becerilerini oluşturmak ve güncellemek için OpenClaw'ın yönetişimli yoludur. Agent'lar ve operatörler bu yol üzerinden hiçbir zaman doğrudan `SKILL.md` yazmaz — bunun yerine, yalnızca uygulandığında canlı bir beceriye dönüşen bir **öneri** (içerik, hedef bağlama, tarayıcı durumu, karmalar ve geri alma meta verileri içeren bekleyen taslak) oluştururlar.

Skill Workshop yalnızca çalışma alanı becerilerini yazar. Paketlenmiş, plugin, ClawHub, ek kök, yönetilen, kişisel agent veya sistem becerilerine hiçbir zaman dokunmaz.

## Nasıl çalışır?

- **Önce öneri:** oluşturulan içerik `SKILL.md` olarak değil, `PROPOSAL.md` olarak depolanır.
- **Tek canlı yazma işlemi uygulamadır:** oluşturma, güncelleme ve düzeltme etkin becerileri hiçbir zaman değiştirmez.
- **Çalışma alanıyla sınırlı:** oluşturma işlemleri çalışma alanının `skills/` kökünü hedefler; güncellemelere yalnızca yazılabilir çalışma alanı becerileri için izin verilir.
- **Üzerine yazma yok:** hedef beceri zaten mevcutsa oluşturma başarısız olur.
- **Karmaya bağlı:** güncelleme önerileri mevcut hedef karmasına bağlanır ve canlı beceri uygulamadan önce değişirse `stale` durumuna geçer.
- **Tarayıcı denetimli:** uygulama, yazmadan önce güvenlik tarayıcısını yeniden çalıştırır.
- **Kurtarılabilir:** uygulama, canlı dosyalara dokunmadan önce geri alma meta verilerini yazar.
- **Tutarlı yüzeyler:** sohbet, CLI ve Gateway aynı hizmeti çağırır.

## Yaşam döngüsü

```text
oluşturma/güncelleme -> beklemede
düzeltme             -> beklemede
uygulama              -> uygulandı
reddetme              -> reddedildi
karantinaya alma      -> karantinaya alındı
hedef değişikliği     -> güncelliğini yitirdi
```

Yalnızca `pending` durumundaki bir öneri düzeltilebilir, uygulanabilir, reddedilebilir veya karantinaya alınabilir.

## Yaşam döngüsü düzenlemesi

Gateway, paylaşılan durum veritabanında toplu beceri kullanımını izler. Günde bir kez, Skill Workshop tarafından oluşturulup uygulanmış becerileri inceler. 30 günden uzun süredir kullanılmayan beceriler `stale`; 90 gün sonra ise `archived` durumuna geçer ve yeni agent beceri anlık görüntülerine dahil edilmez. Arşivlenmiş beceri dosyaları diskte değiştirilmeden kalır. Elle yazılmış beceriler hiçbir zaman düzenlemeye tabi tutulmaz; yaşam döngüsü düzenlemesine yalnızca Skill Workshop önerileriyle oluşturulan beceriler girer.

Sabitlenmiş beceriler yaşam döngüsü geçişlerinden etkilenmez. Güncelliğini yitirmiş bir beceri kullanıldıktan ve sonraki tarama çalıştıktan sonra `active` durumuna döner. Arşivlenmiş beceriler yalnızca açık bir geri yükleme işlemiyle döner:

Yaşam döngüsü geçişleri ve geri yüklemeler yeni oturumlara uygulanır; çalışan oturumlar mevcut beceri anlık görüntülerini korur.

```bash
openclaw skills curator status
openclaw skills curator pin <skill>
openclaw skills curator unpin <skill>
openclaw skills curator restore <skill>
```

Tüm düzenleyici komutları `--json` kabul eder. Durum ayrıca deterministik örtüşme adaylarını yalnızca öneri olarak bildirir; hiçbir zaman becerileri birleştirmez veya bir model çağırmaz.

## Sohbet

Agent'dan istediğiniz beceriyi talep edin; agent `skill_workshop` çağırır ve bir öneri kimliği döndürür.

### Son çalışmalardan öğrenme

Mevcut konuşmayı veya adlandırılmış kaynakları standartların yönlendirdiği tek bir beceri önerisine dönüştürmek için `/learn` kullanın:

```text
/learn
/learn docs/runbook.md ve https://example.com/guide; kurtarmaya odaklan
```

İstek olmadığında `/learn`, agent'dan mevcut konuşmadaki yeniden kullanılabilir iş akışını özümsemesini ister. İstek olduğunda agent; odak, kapsam ve adlandırma gereksinimlerine uyarken yolları, URL'leri, yapıştırılmış notları ve konuşma referanslarını kaynak olarak değerlendirir. Kaynakları mevcut araçlarıyla toplar, ardından `action: "create"` ile `skill_workshop` çağırır.

Ortaya çıkan öneri `pending` durumunda kalır; `/learn` bunu hiçbir zaman uygulamaz. Öneriyi normal onay akışı üzerinden veya `openclaw skills workshop` ile inceleyip uygulayın.

Oluşturma:

```text
Pazartesi gelen kutusu rutinimi çalıştıran morning-catchup adlı bir beceri oluştur.
```

Mevcut bir çalışma alanı becerisini güncelleme:

```text
trip-planning becerisini rezervasyondan önce koltuk haritalarını da kontrol edecek şekilde güncelle.
```

Bekleyen bir öneriyi yineleme:

```text
morning-catchup önerisini göster.
Acil olarak işaretlenen her şeyi de belirtecek şekilde düzelt.
morning-catchup önerisini uygula.
```

Agent tarafından başlatılan `apply`, `reject` ve `quarantine`, varsayılan olarak ek bir onay istemi olmadan çalışır. Bu eylemlerden önce operatör onayı gerektirmek için `skills.workshop.approvalPolicy` değerini `"pending"` olarak ayarlayın.

Onay gerektiğinde istem, öneri kimliğini ve hedef beceriyi belirtir; ayrıca öneri açıklamasını, destek dosyası sayısını ve gövde boyutunu gösterir. Onay istekleri, agent aracı zaman aşımı gözcüsünden önce tamamlanacak şekilde sınırlandırılır. İstemin süresi dolmadan karar verilmezse yaşam döngüsü eylemi çalışmaz: öneri beklemede ve değiştirilmeden kalır. Daha sonra Skill Workshop kullanıcı arayüzünde karar verin veya `openclaw skills workshop apply|reject|quarantine <proposal-id>` çalıştırın. Agent'lar süresi dolmuş bir yaşam döngüsü eylemini döngü içinde yeniden denememelidir.

## CLI

```bash
# Oluştur
openclaw skills workshop propose-create \
  --name morning-catchup \
  --description "Günlük gelen kutusu takibi: sınıflandır, arşivle, öne çıkar, taslak hazırla, planla" \
  --proposal ./PROPOSAL.md

# Mevcut bir çalışma alanı becerisini güncelle
openclaw skills workshop propose-update trip-planning --proposal ./PROPOSAL.md

# Listele ve incele
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>

# Onaydan önce düzelt
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md

# Sonuçlandır
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "Yinelenen"
openclaw skills workshop quarantine <proposal-id> --reason "Güvenlik incelemesi gerekiyor"
```

Her alt komut `--agent <id>` (hedef çalışma alanı; varsayılan olarak önce cwd'den çıkarılan, ardından varsayılan agent) ve `--json` (yapılandırılmış çıktı) kabul eder. `propose-create`, `propose-update` ve `revise` ayrıca öneri bağlamını `--proposal` ile birlikte kaydetmek için `--goal <text>` ve `--evidence <text>` kabul eder.

## Öneri içeriği

Öneri beklemedeyken, yalnızca öneriye özgü frontmatter ile `PROPOSAL.md` olarak depolanır:

```markdown
---
name: "morning-catchup"
description: "Günlük gelen kutusu takibi: sınıflandır, arşivle, öne çıkar, taslak hazırla, planla"
status: proposal
version: "v1"
date: "2026-05-30T00:00:00.000Z"
---
```

Uygulama sırasında Skill Workshop etkin `SKILL.md` dosyasını yazar ve yalnızca öneriye özgü alanları kaldırır: `status`, öneri `version` ve öneri `date`.

## Destek dosyaları

Önerilen beceri `PROPOSAL.md` yanında dosyalara ihtiyaç duyduğunda `--proposal-dir` kullanın:

```bash
openclaw skills workshop propose-create \
  --name weekly-update \
  --description "Cuma kapanışı: istatistikler, öne çıkanlar, gelecek haftanın en önemli üç maddesi" \
  --proposal-dir ./weekly-update-proposal
```

Dizin `PROPOSAL.md` içermelidir. Destek dosyaları `assets/`, `examples/`, `references/`, `scripts/` veya `templates/` altında bulunmalıdır. Skill Workshop bunları tarar, karmalarını hesaplar ve öneriyle birlikte depolar; ardından yalnızca uygulama sırasında canlı `SKILL.md` yanına yazar.

Reddedilen destek dosyası yolları: mutlak yollar, gizli yol segmentleri, yol geçişi, örtüşen yollar, yürütülebilir dosyalar, UTF-8 olmayan metinler, null baytları ve standart destek klasörlerinin dışındaki yollar.

## Agent aracı

Model, gerekli tek bir `action` ile `skill_workshop` kullanır:
`create | update | revise | list | inspect | apply | reject | quarantine`.
Diğer parametreler eyleme bağlı olarak uygulanır:

| Parametre                  | Kullananlar                                           | Notlar                                                               |
| -------------------------- | ---------------------------------------------------- | -------------------------------------------------------------------- |
| `name`                     | `create`, `inspect`, `revise`                        | `create` için gereklidir; aksi takdirde bekleyen bir öneriyi ada göre çözümler |
| `description`              | `create`, `update`, `revise`                         | En fazla 160 bayt                                                     |
| `skill_name`               | `update`                                             | Mevcut beceri adı veya anahtarı                                      |
| `proposal_content`         | `create`, `update`, `revise`                         | `PROPOSAL.md` olarak depolanır; `skills.workshop.maxSkillBytes` ile sınırlandırılır |
| `support_files`            | `create`, `update`, `revise`                         | `{ path, content }` dizisi                                            |
| `goal`, `evidence`         | `create`, `update`, `revise`                         | Serbest metin bağlamı                                                 |
| `proposal_id`              | `inspect`, `revise`, `apply`, `reject`, `quarantine` | Hedef öneri                                                           |
| `reason`                   | `apply`, `reject`, `quarantine`                      | İsteğe bağlı                                                         |
| `query`, `status`, `limit` | `list`                                               | Filtreleme/sayfalama; `limit` en fazla 50, varsayılan 20          |

Agent'lar oluşturulan beceri çalışmaları için `skill_workshop` kullanmalıdır. Öneri dosyalarını `write`, `edit`, `exec`, kabuk komutları veya doğrudan dosya sistemi işlemleri aracılığıyla oluşturmamalı veya değiştirmemelidir.

<Note>
`skill_workshop` yerleşik bir agent aracıdır ve `tools.profile: "coding"` kapsamına dahildir. Daha katı bir politika bunu gizliyorsa etkin `tools.allow` listesine `skill_workshop` ekleyin veya kapsam açık bir `tools.allow` içermeyen bir profil kullanıyorsa `tools.alsoAllow: ["skill_workshop"]` kullanın. Korumalı alan çalıştırmaları ana makine tarafındaki Skill Workshop aracını oluşturmaz; bu nedenle öneri inceleme eylemlerini normal bir ana makine tarafı agent oturumundan veya CLI'dan çalıştırın.
</Note>

## Önerilen beceriler

OpenClaw, etkileşimli bir tur sona erdiğinde başarısız turlar dahil olmak üzere “bir dahaki sefere”, “şunu hatırla” gibi kalıcı talimatları ve tepkisel düzeltmeleri algılar. Sonraki turda agent, algılanan en son iş akışını `skill_workshop` aracılığıyla kaydetmeyi önerir; öneri oluşturulup oluşturulmayacağına kullanıcı karar verir. Bu yerleşik öneri kendi başına bir beceri oluşturmaz veya değiştirmez. Bunun yerine bekleyen önerileri doğrudan oluşturmak için `skills.workshop.autonomous.enabled` etkinleştirin. Control UI'da Workshop sekmesi aynı ayarı sayfa başlığında bir **Kendi kendine öğrenme** anahtarı ve boş öneri panosunda bir etkinleştirme düğmesi olarak sunar.

### Geçmiş oturumları tarama

Control UI, otonom kendi kendine öğrenmeyi etkinleştirmeden eski çalışmaları inceleyebilir. **Plugins → Workshop** bölümünü açın ve **Beceri fikirleri bul** seçeneğini belirleyin. Tarama, uygun en yeni oturumlarla başlar ve kapsamı sınırlandırılmış önemli çalışmalar penceresini inceler. Cron, Heartbeat, kanca, alt agent, ACP, plugin'e ait ve dahili inceleme oturumlarının yanı sıra altıdan az model turu içeren konuşmaları atlar.

İnceleyici, seçilen agent'ın yapılandırılmış modelini kullanır ve gizli bilgileri ayıklanmış, boyutu sınırlandırılmış bir konuşma dökümü paketi alır. Deneyim incelemesiyle aynı ihtiyatlı ölçütü uygular: somut bir kurtarma kalıbı veya gelecekteki en az iki model ya da araç çağrısını ortadan kaldıracak kararlı bir prosedür. Rutin çalışmalar ve tek seferlik olgular hiçbir öneri üretmemelidir.

Tek bir tarama en fazla üç bekleyen öneri oluşturabilir veya düzeltebilir. Canlı bir beceriyi uygulayamaz, reddedemez, karantinaya alamaz veya düzenleyemez. Workshop, örneğin **20 oturum incelendi · 18 Haz–bugün · 2 fikir bulundu** biçiminde birikimli kapsamı gösterir. Kalıcı en eski oturum imlecinden devam etmek için **Daha eski çalışmaları tara** seçeneğini belirleyin. Kullanılabilir geçmiş tükendikten sonra eylem **Yeni çalışmaları tara** olur.

Geçmiş inceleme, `skills.workshop.autonomous.enabled` değeri `false` olduğunda bile
manuel olarak yapılır. Her tıklama bir model çalıştırması başlatır;
bu nedenle sağlayıcı fiyatlandırması ve veri işleme koşulları geçerlidir. İmleç ve kapsam sayıları
paylaşılan OpenClaw durum veritabanında saklanır; transkript içeriği
tarama durumuna kopyalanmaz.

Otonom yakalama etkinleştirildiğinde OpenClaw, başarılı ve
kapsamlı çalışmaların ardından ve tüm ajan sistemi boşa geçtiğinde ihtiyatlı bir inceleme de gerçekleştirebilir. Bu yalıtılmış inceleme en fazla
bir bekleyen teklif oluşturabilir veya revize edebilir. `approvalPolicy` değeri `"auto"` olsa bile canlı bir skill'i güncelleyemez veya bir
teklifi uygulayamaz, reddedemez ya da karantinaya alamaz.

Etkinleştirme, uygunluk, gizlilik ve maliyet ayrıntıları,
teklif eşiği ve sorun giderme için [Kendi kendine öğrenme](/tr/tools/self-learning) bölümüne bakın.

## Onay ve özerklik

```json5
{
  skills: {
    workshop: {
      autonomous: {
        enabled: false,
      },
      allowSymlinkTargetWrites: false,
      approvalPolicy: "auto",
      maxPending: 50,
      maxSkillBytes: 40000,
    },
  },
}
```

| Ayar                       | Varsayılan | Etki                                                                                                                                                               |
| -------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `autonomous.enabled`         | `false` | Açık düzeltmelerden ve bir boşta kalma gecikmesinin ardından, yeniden kullanılabilir kurtarma veya anlamlı gidiş-dönüş tasarrufları sağlayan tamamlanmış kapsamlı çalışmalardan bekleyen teklifler oluşturur. |
| `allowSymlinkTargetWrites`         | `false` | Gerçek hedefi `skills.load.allowSymlinkTargets` içinde listelenen çalışma alanı skill sembolik bağlantıları üzerinden uygulama işleminin yazmasına izin verir. |
| `approvalPolicy`         | `"auto"` | `"auto"`, ajan tarafından başlatılan `apply`, `reject` veya `quarantine` için ek istemi atlar (ajan yine de eylemi çağırmalıdır). `"pending"` onay gerektirir. |
| `maxPending`         | `50` | Çalışma alanı başına bekleyen ve karantinaya alınmış teklifleri sınırlar (1-200). |
| `maxSkillBytes`         | `40000` | Teklif gövdesi boyutunu bayt cinsinden sınırlar (1024-200000). |

Otonom yakalama, ileriye dönük kuralları (örneğin, “bundan sonra”) ve tepkisel
düzeltmeleri (örneğin, “istediğim bu değildi”) tanır. Yeni talimatları konuya göre tur başına en
fazla üç teklif hâlinde gruplar, sözcük dağarcığı eşleşmelerini mevcut yazılabilir çalışma alanı skill'lerine yönlendirir ve
başka bir düzeltme aynı skill'i hedeflediğinde kendi bekleyen teklifini revize eder.

Açık bir düzeltme olmadan tamamlanan başarılı ve kapsamlı çalışmalar için, seçilen
modelin yalıtılmış bir çalışması, tamamlanan gidişatın ihtiyatlı teklif eşiğini aşıp aşmadığına karar verir. Ön plan
modelinden yanıt vermeden önce öğrenmesi istenmez. Arka plan inceleyicisi,
ön plan çalışmasını teklifin kaynağı olarak korur, genel ajan araçlarına erişemez ve yaşam döngüsü
kararları veremez. İnceleme yalnızca ön plan çalışma zamanı hem tam olarak çözümlenmiş modelini hem de
`skill_workshop` aracının gerçekten kullanılabilir olduğunu bildirdiğinde başlar. Bu nedenle kısıtlayıcı veya bilinmeyen
araç politikası güvenli biçimde başarısız olur ve teklif oluşturmaz.

Eksiksiz otonom inceleme davranışı ve güvenlik
modeli için [Kendi kendine öğrenme](/tr/tools/self-learning) bölümüne bakın.

Teklif açıklamaları, `maxSkillBytes` değerinden bağımsız olarak her zaman 160 baytla
sınırlıdır.

## Gateway yöntemleri

| Yöntem                             | Kapsam            |
| ---------------------------------- | ----------------- |
| `skills.proposals.list`                 | `operator.read` |
| `skills.proposals.inspect`                 | `operator.read` |
| `skills.proposals.historyStatus`                 | `operator.read` |
| `skills.proposals.historyScan`                 | `operator.admin` |
| `skills.proposals.create`                 | `operator.admin` |
| `skills.proposals.update`                 | `operator.admin` |
| `skills.proposals.revise`                 | `operator.admin` |
| `skills.proposals.requestRevision`                 | `operator.admin` |
| `skills.proposals.apply`                 | `operator.admin` |
| `skills.proposals.reject`                 | `operator.admin` |
| `skills.proposals.quarantine`                 | `operator.admin` |
| `skills.curator.status`                 | `operator.read` |
| `skills.curator.pin`                 | `operator.admin` |
| `skills.curator.unpin`                 | `operator.admin` |
| `skills.curator.restore`                 | `operator.admin` |

`requestRevision` yalnızca Gateway'e özeldir (CLI veya ajan aracı eşdeğeri yoktur):
ajanın değişmez yeni içerik göndermek yerine revizyon yapmasının istendiği kullanıcı arayüzleri için
`PROPOSAL.md` değerini doğrudan değiştirmek yerine serbest metinli revizyon talimatlarını
sahip ajanın sohbet oturumuna iletir.

`historyStatus` ve `historyScan`, Control UI destek yöntemleridir. `historyScan`,
`direction: "older" | "newer"` kabul eder; sonuçları her zaman bekleyen
teklifler olarak bırakır.

## Depolama

```text
<OPENCLAW_STATE_DIR>/skill-workshop/
  proposals.json
  proposals/<proposal-id>/
    proposal.json
    PROPOSAL.md
    rollback.json
    assets/
    examples/
    references/
    scripts/
    templates/
```

Varsayılan durum dizini: `~/.openclaw`.

- `proposal.json`: standart teklif kaydı.
- `proposals.json`: teklif klasörlerinden yeniden oluşturulabilen hızlı listeleme dizini.
- `PROPOSAL.md`: bekleyen skill teklifi.
- `rollback.json`: uygulama işlemi canlı dosyaları değiştirmeden önce yazılan kurtarma meta verileri.

## Sınırlar

| Sınır                           | Değer                                                                |
| ------------------------------- | -------------------------------------------------------------------- |
| Açıklama                        | 160 bayt                                                             |
| Teklif gövdesi                  | `skills.workshop.maxSkillBytes` (varsayılan 40.000; kesin üst sınır 1 MiB) |
| Destek dosyaları                | Teklif başına 64                                                     |
| Destek dosyası boyutu           | Her biri 256 KiB, toplam 2 MiB                                       |
| Bekleyen + karantinaya alınmış teklifler | Çalışma alanı başına `skills.workshop.maxPending` (varsayılan 50)   |

## Sorun giderme

| Sorun                                          | Çözüm                                                                                                                                                                                                       |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Skill proposal description is too large`                             | `description` değerini 160 bayta veya daha aza kısaltın.                                                                                                                                                |
| `Skill proposal content is too large`                             | Teklif gövdesini kısaltın veya `skills.workshop.maxSkillBytes` değerini artırın.                                                                                                                                          |
| `Target skill changed after proposal creation`                             | Teklifi geçerli hedefe göre revize edin veya yeni bir teklif oluşturun.                                                                                                                                     |
| `Proposal scan failed`                             | Tarayıcı bulgularını inceleyin, ardından teklifi revize edin veya karantinaya alın.                                                                                                                         |
| `untrusted symlink target`                             | `skills.load.allowSymlinkTargets` yapılandırmasını yapın ve `skills.workshop.allowSymlinkTargetWrites` seçeneğini yalnızca bilinçli olarak paylaşılan skill kökleri için etkinleştirin.                                                             |
| `Support file paths must be under one of...`                             | Destek dosyalarını `assets/`, `examples/`, `references/`, `scripts/` veya `templates/` altına taşıyın.                                                                    |
| Teklif listede görünmüyor                      | Seçili `--agent` çalışma alanını ve `OPENCLAW_STATE_DIR` değerini kontrol edin.                                                                                                                       |
| Ajan `skill_workshop` çağrısını yapamıyor    | Etkin araç politikasını ve çalışma modunu kontrol edin. `coding` aracı içerir; kısıtlayıcı `tools.allow` politikaları aracı açıkça listelemeli ve korumalı alan çalışmaları normal bir ana makine tarafı ajan oturumunu veya CLI'yi kullanmalıdır. |

### Araç politikası tanılaması

Otonom yakalama etkinleştirildiğinde `openclaw doctor`, varsayılan ajan için
`core/doctor/skill-workshop-tool-policy` denetimini çalıştırır. Politika
`skill_workshop` aracını gizliyorsa uyarı, dışlayan ilk yapılandırma katmanını ve
yapılması gereken tam `allow` veya `alsoAllow` değişikliğini belirtir. Eski çalışma kitapları hâlâ
`openclaw plugins inspect skill-workshop` kullanıyor olabilir; bu komut artık Skill
Workshop'un yerleşik olduğunu açıklar ve uygulanabilir olduğunda aynı politika ipucunu yazdırır.

## İlgili

- Yükleme sırası, öncelik ve görünürlük için [Skills](/tr/tools/skills)
- Çalışma sonrası ihtiyatlı skill teklifleri için [Kendi kendine öğrenme](/tr/tools/self-learning)
- Elle yazılan `SKILL.md`
  temelleri için [Skill oluşturma](/tr/tools/creating-skills)
- Tam `skills.workshop` şeması için [Skills yapılandırması](/tr/tools/skills-config)
- `openclaw skills` komutları için [Skills CLI](/tr/cli/skills)
