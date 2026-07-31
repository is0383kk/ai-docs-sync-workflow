---
read_when:
    - Agent aracılığıyla arka planda veya paralel çalışma istiyorsunuz
    - sessions_spawn veya alt ajan araç politikasını değiştiriyorsunuz
    - İleti dizisine bağlı alt ajan oturumlarını uyguluyor veya sorunlarını gideriyorsunuz
sidebarTitle: Sub-agents
summary: Sonuçları istekte bulunan sohbete bildiren yalıtılmış arka plan agent çalıştırmaları başlatın
title: Alt ajanlar
x-i18n:
    generated_at: "2026-07-27T00:22:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e45b32fdb177c52ed785287712b9b6c2c30bbe392f0ce975970910ff91ed30ed
    source_path: tools/subagents.md
    workflow: 16
---

Alt ajanlar, mevcut bir ajan çalıştırmasından başlatılan arka plan ajan çalıştırmalarıdır.
Her biri kendi oturumunda (`agent:<agentId>:subagent:<uuid>`) çalışır ve
tamamlandığında sonucunu istekte bulunan sohbet kanalına **duyurur**.
Her alt ajan çalıştırması bir [arka plan görevi](/tr/automation/tasks) olarak izlenir.

Hedefler:

- Araştırmaları, uzun görevleri ve yavaş araç çalışmalarını ana çalıştırmayı engellemeden paralelleştirmek.
- Alt ajanları varsayılan olarak yalıtılmış tutmak (oturum ayrımı, isteğe bağlı korumalı alan kullanımı).
- Araç yüzeyinin kötüye kullanımını zorlaştırmak: alt ajanlar varsayılan olarak oturum veya mesaj araçlarına erişemez.
- Orkestratör kalıpları için yapılandırılabilir iç içe geçme derinliğini desteklemek.

<Note>
**Maliyet notu:** varsayılan olarak her alt ajanın kendi bağlamı ve token kullanımı
vardır. Yoğun veya tekrarlanan görevlerde alt ajanlar için daha ucuz bir model
ayarlayın ve `agents.defaults.subagents.model` ya da ajan başına geçersiz kılmalar aracılığıyla
ana ajanınızda daha yüksek kaliteli bir model kullanın. Bir alt öğenin
istekte bulunanın geçerli konuşma dökümüne gerçekten ihtiyacı varsa onu
`context: "fork"` ile başlatın. İş parçacığına bağlı alt ajan oturumları,
geçerli konuşmayı bir takip iş parçacığına dallandırdıkları için varsayılan olarak
`context: "fork"` kullanır.
</Note>

## Eğik çizgi komutu

`/subagents`, **geçerli oturumdaki** alt ajan çalıştırmalarını inceler:

```text
/subagents list
/subagents log <id|#> [limit] [tools]
/subagents info <id|#>
```

`/subagents info`, çalıştırma meta verilerini (durum, zaman damgaları, oturum kimliği,
konuşma dökümü yolu, temizlik) gösterir. `/subagents log`, bir çalıştırmanın son
sohbet turlarını yazdırır; araç çağrısı/sonuç mesajlarını (varsayılan olarak
atlanır) dahil etmek için `tools` token'ını ekleyin. Bir ajan turunun içinden
sınırlı ve güvenlik filtresinden geçirilmiş bir geri çağırma görünümü için
`sessions_history` kullanın veya ham ve eksiksiz konuşma dökümü için diskteki
konuşma dökümü yolunu inceleyin.

Control UI'da, yakın zamanda alt çalıştırmaları bulunan üst oturumların genişletilebilir
bir kenar çubuğu satırı vardır. İç içe satırlar alt öğenin durumunu ve çalışma süresini
gösterir; bunlardan biri seçildiğinde üst öğe hiyerarşisi korunarak o alt öğenin
sohbeti açılır.

### İş parçacığı bağlama denetimleri

Bu komutlar, kalıcı iş parçacığı bağlamalarını destekleyen kanallarda çalışır. Aşağıdaki
[İş parçacığını destekleyen kanallar](#thread-supporting-channels) bölümüne bakın.

```text
/focus <subagent-label|session-key|session-id|session-label>
/unfocus
/agents
/session idle <duration|off>
/session max-age <duration|off>
```

### Başlatma davranışı

Ajanlar, `sessions_spawn` aracıyla arka plan alt ajanlarını başlatır.
Tamamlanmalar, dahili üst oturum olayları olarak döner; üst/istekte bulunan
ajan, kullanıcıya yönelik bir güncellemenin gerekip gerekmediğine karar verir.

<AccordionGroup>
  <Accordion title="Engellemesiz, gönderim tabanlı tamamlanma">
    - `sessions_spawn` engellemesizdir; hemen bir çalıştırma kimliği döndürür.
    - Tamamlandığında alt ajan, üst/istekte bulunan oturuma geri bildirimde bulunur.
    - Alt öğe sonuçlarına ihtiyaç duyan ajan turları, gerekli işleri başlattıktan sonra `sessions_yield` çağrısı yapmalıdır. Bu, geçerli turu sonlandırır ve tamamlanma olayının modelin görebileceği bir sonraki mesaj olarak gelmesini sağlar.
    - Tamamlanma gönderim tabanlıdır. Başlatıldıktan sonra, yalnızca bitmesini beklemek amacıyla `/subagents list`, `sessions_list` veya `sessions_history` öğelerini döngü içinde yoklamayın; durumu yalnızca hata ayıklarken gerektiğinde kontrol edin.
    - Alt öğe çıktısı, istekte bulunan ajanın sentezlemesi için bir rapor/kanıttır. Kullanıcı tarafından yazılmış talimat metni değildir ve sistem, geliştirici veya kullanıcı politikasını geçersiz kılamaz.
    - Tamamlanma sırasında OpenClaw, duyuru temizleme akışı devam etmeden önce söz konusu alt ajan oturumu tarafından açılan izlenen tarayıcı sekmelerini/süreçlerini elinden geldiğince kapatır.

  </Accordion>
  <Accordion title="Tamamlanmanın teslimi">
    - OpenClaw, tamamlanmaları kararlı bir eşgüçlülük anahtarıyla bir `agent` turu üzerinden istekte bulunan oturuma geri iletir.
    - İstekte bulunan çalıştırma hâlâ etkinse OpenClaw, ikinci bir görünür yanıt yolu başlatmak yerine önce bu çalıştırmayı uyandırmayı/yönlendirmeyi dener.
    - Etkin bir istekte bulunan uyandırılamazsa OpenClaw, duyuruyu bırakmak yerine aynı tamamlanma bağlamıyla istekte bulunan ajana devretmeye geri döner.
    - Üst öğenin görünür bir kullanıcı güncellemesinin gerekmediğine karar verdiği durumlarda bile başarılı bir üst öğe devri, alt ajan teslimini tamamlar.
    - Yerel alt ajanlar mesaj aracına erişemez. Üst/istekte bulunan ajana düz asistan metni döndürürler; insanların görebildiği yanıtların sorumluluğu, üst/istekte bulunan ajanın normal teslim politikasında kalır.
    - Doğrudan devir kullanılamazsa teslimat önce kuyruk yönlendirmesine, ardından tamamen vazgeçilmeden önce duyurunun kısa bir üstel geri çekilmeli yeniden denemesine geri döner.
    - Teslimat, çözümlenen istekte bulunan rotasını korur: mevcut olduğunda iş parçacığına veya konuşmaya bağlı tamamlanma rotaları önceliklidir. Tamamlanmanın kaynağı yalnızca bir kanal sağlıyorsa OpenClaw, doğrudan teslimatın çalışmaya devam etmesi için eksik hedefi/hesabı istekte bulunan oturumun çözümlenen rotasından (`lastChannel` / `lastTo` / `lastAccountId`) doldurur.

  </Accordion>
  <Accordion title="Tamamlanma devri meta verileri">
    İstekte bulunan oturuma yapılan tamamlanma devri, çalışma zamanı tarafından oluşturulan
    dahili bağlamdır (kullanıcı tarafından yazılmış metin değildir) ve şunları içerir:

    - `Result` — alt öğenin görünür en son `assistant` yanıt metni. Araç/toolResult çıktısı alt öğe sonuçlarına yükseltilmez. Sonlandırıcı biçimde başarısız olan çalıştırmalar, yakalanmış yanıt metnini yeniden kullanmaz.
    - `Status` — `completed; ready for parent review` / `failed` / `timed out` / `unknown`.
    - Kompakt çalışma zamanı/token istatistikleri.
    - İstekte bulunan ajana, özgün görevin tamamlanıp tamamlanmadığına karar vermeden önce sonucu doğrulamasını söyleyen bir inceleme talimatı.
    - Alt öğe sonucu daha fazla işlem gerektiriyorsa istekte bulunan ajana göreve devam etmesini veya bir takip kaydı oluşturmasını söyleyen takip rehberliği.
    - Başka işlem gerekmeyen yol için ham dahili meta verileri iletmeden normal asistan diliyle yazılmış bir son güncelleme talimatı.

  </Accordion>
  <Accordion title="Modlar ve ACP çalışma zamanı">
    - `--model` ve `--thinking`, söz konusu çalıştırma için varsayılanları geçersiz kılar.
    - Tamamlanmadan sonra ayrıntıları ve çıktıyı incelemek için `info`/`log` kullanın.
    - Kalıcı iş parçacığına bağlı oturumlarda `thread: true` ve `mode: "session"` ile `sessions_spawn` kullanın.
    - İstekte bulunan kanal iş parçacığı bağlamalarını desteklemiyorsa olanaksız bir iş parçacığına bağlı birleşimi yeniden denemek yerine `mode: "run"` kullanın.
    - ACP donanım oturumu oturumlarında (Claude Code, Gemini CLI, OpenCode veya açık Codex ACP/acpx), araç bu çalışma zamanını sunduğunda `runtime: "acp"` ile `sessions_spawn` kullanın. Tamamlanmalarda veya ajanlar arası döngülerde hata ayıklarken [ACP teslim modeli](/tr/tools/acp-agents#delivery-model) bölümüne bakın. `codex` Plugin'i etkinleştirildiğinde, kullanıcı açıkça ACP/acpx istemediği sürece Codex sohbet/iş parçacığı denetimi ACP yerine `/codex ...` tercih etmelidir.
    - OpenClaw; ACP etkinleştirilene, istekte bulunan korumalı alanda olmayana ve `acpx` gibi bir arka uç Plugin'i yüklenene kadar `runtime: "acp"` öğesini gizler. `runtime: "acp"`, harici bir ACP donanım oturumu kimliği veya `runtime.type="acp"` içeren bir `agents.entries.*` girdisi bekler; `agents_list` kaynaklı normal OpenClaw yapılandırma ajanları için varsayılan alt ajan çalışma zamanını kullanın.

  </Accordion>
</AccordionGroup>

## Bağlam modları

Yerel alt ajanlar, çağıran taraf geçerli konuşma dökümünün dallandırılmasını
açıkça istemediği sürece yalıtılmış olarak başlar.

| Mod       | Ne zaman kullanılmalı                                                                                                                         | Davranış                                                                          |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `isolated` | Yeni araştırma, bağımsız uygulama, yavaş araç çalışması veya görev metninde açıklanabilecek herhangi bir iş                           | Temiz bir alt konuşma dökümü oluşturur. Bu varsayılandır ve token kullanımını daha düşük tutar.  |
| `fork`     | Geçerli konuşmaya, önceki araç sonuçlarına veya istekte bulunanın konuşma dökümünde zaten bulunan incelikli talimatlara bağlı işler | Alt öğe başlamadan önce istekte bulunanın konuşma dökümünü alt oturuma dallandırır. |

`fork` öğesini ölçülü kullanın. Bu, bağlama duyarlı yetkilendirme içindir;
açık bir görev istemi yazmanın yerine geçmez.

## Araç: `sessions_spawn`

Global `subagent` şeridinde `deliver: false` ile bir alt ajan çalıştırması
başlatır, ardından bir duyuru adımı çalıştırır ve duyuru yanıtını istekte bulunanın
sohbet kanalına gönderir.

Kullanılabilirlik, çağıran tarafın geçerli araç politikasına bağlıdır. Yerleşik
`coding` ve `messaging` profilleri `sessions_spawn`,
`sessions_yield` ve `subagents` öğelerini içerir; `minimal` içermez. `full` tüm
araçlara izin verir. Çalışmaları yetkilendirebilmesi gereken, özel ve daha dar
bir profildeki ajan için bu araçları `tools.alsoAllow` ile ekleyin veya yukarıdaki
profillerden birini kullanın.
Kanal/grup, sağlayıcı, korumalı alan ve ajan başına izin verme/reddetme politikaları,
profil aşamasından sonra da aracı kaldırabilir. Geçerli araç listesini doğrulamak
için aynı oturumdan `/tools` kullanın.

**Varsayılanlar:**

- **Model:** `agents.defaults.subagents.model` (veya ajan başına `agents.entries.*.subagents.model`) ayarlamadığınız sürece yerel alt ajanlar çağıran tarafın modelini devralır. ACP çalışma zamanı başlatmaları, yapılandırılmış alt ajan modeli mevcut olduğunda aynı modeli kullanır; aksi takdirde ACP donanım oturumu kendi varsayılanını korur. Açık bir `sessions_spawn.model` yine önceliklidir.
- **Düşünme:** `agents.defaults.subagents.thinking` (veya ajan başına `agents.entries.*.subagents.thinking`) ayarlamadığınız sürece yerel alt ajanlar çağıran tarafın ayarını devralır. ACP çalışma zamanı başlatmaları da seçilen model için `agents.defaults.models["provider/model"].params.thinking` uygular. Açık bir `sessions_spawn.thinking` yine önceliklidir.
- **Çalıştırma zaman aşımı:** OpenClaw, ayarlandığında `agents.defaults.subagents.runTimeoutSeconds` kullanır; aksi takdirde `0` değerine (zaman aşımı yok) geri döner. `sessions_spawn`, çağrı başına zaman aşımı geçersiz kılmalarını kabul etmez.
- **Süreç ömrü:** ayrılmış bir OpenClaw alt ajanının kendi çalıştırma yaşam döngüsü vardır. Harici bir CLI arka ucunda oluşturulan arka plan görevi farklıdır: üst CLI alt sürecini paylaşır ve bu üst öğe `agents.defaults.timeoutSeconds` durumuna ulaşırsa durur.
- **Görev teslimi:** yerel alt ajanlar, yetkilendirilen görevi ilk görünür `[Subagent Task]` mesajlarında alır. Alt ajan sistem istemi, görevin gizli bir kopyasını değil çalışma zamanı kurallarını ve yönlendirme bağlamını taşır.

Kabul edilen yerel alt ajan başlatmaları, çözümlenen alt model meta verilerini
araç sonucuna dahil eder: `resolvedModel` uygulanan model başvurusunu,
başvuruda varsa `resolvedProvider` ise sağlayıcı önekini içerir.

### Yetkilendirme istemi modu

`agents.defaults.subagents.delegationMode` yalnızca istem rehberliğini denetler; araç politikasını değiştirmez veya yetkilendirmeyi zorunlu kılmaz.

- `suggest` (varsayılan): daha büyük veya daha yavaş işler için alt ajan kullanmaya yönelik standart istem yönlendirmesini korur.
- `prefer`: ana ajana yanıt vermeye hazır kalmasını ve doğrudan yanıt vermekten daha kapsamlı olan her şeyi `sessions_spawn` aracılığıyla yetkilendirmesini söyler.

Ajan başına geçersiz kılma: `agents.entries.*.subagents.delegationMode`.

```json5
{
  agents: {
    defaults: {
      subagents: {
        delegationMode: "prefer",
        maxConcurrent: 4,
      },
    },
    list: [
      {
        id: "coordinator",
        subagents: { delegationMode: "prefer" },
      },
    ],
  },
}
```

### Araç parametreleri

<ParamField path="task" type="string" required>
  Alt ajanın görev açıklaması.
</ParamField>
<ParamField path="taskName" type="string">
  Daha sonraki durum çıktısında belirli bir alt öğeyi tanımlamak için isteğe bağlı kararlı tanıtıcı. `[a-z][a-z0-9_-]{0,63}` ile eşleşmelidir ve `last` veya `all` gibi ayrılmış bir hedef olamaz.
</ParamField>
<ParamField path="label" type="string">
  İsteğe bağlı, insanlar tarafından okunabilir etiket.
</ParamField>
<ParamField path="agentId" type="string">
  `subagents.allowAgents` tarafından izin verildiğinde yapılandırılmış başka bir ajan kimliği altında oluşturur.
</ParamField>
<ParamField path="cwd" type="string">
  Alt çalıştırma için isteğe bağlı görev çalışma dizini. Yerel alt ajanlar önyükleme dosyalarını yine hedef ajanın çalışma alanından yükler; `cwd` yalnızca çalışma zamanı araçlarının ve CLI koşumlarının devredilen işi nerede yaptığını değiştirir.
</ParamField>
<ParamField path="runtime" type='"subagent" | "acp"' default="subagent">
  `acp` yalnızca harici ACP koşumları (`claude`, `droid`, `gemini`, `opencode` veya açıkça istenen Codex ACP/acpx) ve `runtime.type` değeri `acp` olan `agents.entries.*` girdileri içindir.
</ParamField>
<ParamField path="resumeSessionId" type="string">
  Yalnızca ACP. `runtime: "acp"` durumunda mevcut bir ACP koşum oturumunu sürdürür; yerel alt ajan oluşturma işlemlerinde yok sayılır.
</ParamField>
<ParamField path="streamTo" type='"parent"'>
  Yalnızca ACP. `runtime: "acp"` durumunda ACP çalıştırma çıktısını üst oturuma aktarır; yerel alt ajan oluşturma işlemlerinde atlayın.
</ParamField>
<ParamField path="model" type="string">
  Alt ajan modelini geçersiz kılar. Geçersiz değerler atlanır ve alt ajan, araç sonucunda bir uyarıyla varsayılan modelde çalışır.
</ParamField>
<ParamField path="thinking" type="string">
  Alt ajan çalıştırması için düşünme düzeyini geçersiz kılar. `visible: true` ile kullanılamaz.
</ParamField>
<ParamField path="thread" type="boolean" default="false">
  `true` olduğunda bu alt ajan oturumu için kanal ileti dizisi bağlaması ister.
</ParamField>
<ParamField path="mode" type='"run" | "session"' default="run">
  `thread: true` ise ve `mode` atlanmışsa varsayılan değer `session` olur. `mode: "session"`, `thread: true` gerektirir.
  İstekte bulunan kanal için ileti dizisi bağlaması kullanılamıyorsa bunun yerine `mode: "run"` kullanın.
  `visible: true` ile `mode` değerini atlayın; görünür oturumlar kalıcıdır ve `mode: "run"` desteklemez.
</ParamField>
<ParamField path="cleanup" type='"delete" | "keep"' default="keep">
  `"delete"`, duyurudan hemen sonra oturumu arşivler (transkripti yeniden adlandırarak yine saklar).
</ParamField>
<ParamField path="sandbox" type='"inherit" | "require"' default="inherit">
  `require`, hedef alt çalışma zamanı korumalı alan içinde değilse oluşturma işlemini reddeder.
</ParamField>
<ParamField path="context" type='"isolated" | "fork"' default="isolated">
  `fork`, istekte bulunanın mevcut transkriptini alt oturuma dallandırır. Yalnızca yerel alt ajanlar. İletişim dizisine bağlı oluşturma işlemleri varsayılan olarak `fork`; iletişim dizisine bağlı olmayan oluşturma işlemleri ise `isolated` kullanır. Görünür bir dal, istekte bulunanla aynı ajanı hedeflemelidir.
</ParamField>
<ParamField path="visible" type="boolean" default="false">
  Kullanıcının Control UI içinde açabileceği kalıcı bir pano oturumu oluşturur. Görünür oluşturma işlemleri yalnızca `runtime: "subagent"` destekler ve oluşturulan oturumu her zaman saklar.
</ParamField>
<ParamField path="worktree" type="boolean" default="false">
  Yeni pano oturumu için yönetilen bir git çalışma ağacı sağlar. `visible: true` gerektirir.
</ParamField>
<ParamField path="worktreeName" type="string">
  İsteğe bağlı yönetilen çalışma ağacı adı. `visible: true` ve `worktree: true` gerektirir.
</ParamField>
<ParamField path="worktreeBaseRef" type="string">
  Yönetilen çalışma ağacı için isteğe bağlı git temel referansı. `visible: true` ve `worktree: true` gerektirir.
</ParamField>

<Warning>
`sessions_spawn`, kanal teslimi parametrelerini (`target`,
`channel`, `to`, `threadId`, `replyTo`, `transport`) **kabul etmez**. Yerel alt ajanlar
en son asistan dönüşlerini istekte bulunana geri bildirir; harici teslimat
üst/istekte bulunan ajanın sorumluluğunda kalır.
</Warning>

`visible: true`, `model`, `cwd` ve aynı ajana ait bir `context: "fork"` desteklenir. Korumalı alan içindeki bir hedef, `cwd` değerini o ajanın çalışma alanıyla sınırlar. Görünür oturumlar `sessions.create` aracılığıyla oluşturulan kalıcı pano oturumları olduğundan ileti dizisi bağlaması, `mode`, düşünme geçersiz kılmaları, `lightContext`, `attachments` ve `attachAs` bu yolda kullanılamaz. İstekte bulunanın kendisi devralınmış bir araç izin listesi veya engelleme listesiyle oluşturulmuşsa görünür oluşturma reddedilir; bu kısıtlama oluşturma anında sabitlenir ve yapılandırma ile geçersiz kılınamaz. Oturum listeleme ve adresleme `tools.sessions.visibility` kurallarına uyar; varsayılan `tree` kapsamı mevcut oturumu ve onun kendi oluşturma alt ağacını kapsar. Kullanıma alma adlandırması, kurulum, temizleme ve geri yükleme davranışı için [Yönetilen çalışma ağaçları](/tr/concepts/managed-worktrees) bölümüne bakın.

### Görev adları ve hedefleme

`taskName`, oturum anahtarı değil, orkestrasyon için modele yönelik bir tanıtıcıdır.
Bir koordinatörün daha sonra ilgili alt öğeyi incelemesi gerekebilecek durumlarda
`review_subagents`, `linux_validation` veya `docs_update` gibi kararlı alt öğe adları için kullanın.

Hedef çözümleme, tam `taskName` eşleşmelerini ve belirsiz olmayan
önekleri kabul eder. Eşleştirme, numaralı `/subagents` hedeflerinin kullandığı
aynı etkin/yakın tarihli hedef penceresiyle sınırlandırılır; dolayısıyla eski ve tamamlanmış
bir alt öğe, yeniden kullanılan bir tanıtıcıyı belirsiz hâle getirmez. Etkin veya yakın tarihli
iki alt öğe aynı `taskName` değerini paylaşıyorsa hedef belirsizdir; bunun yerine
liste dizinini, oturum anahtarını veya çalıştırma kimliğini kullanın.

`last` ve `all` ayrılmış hedefleri zaten denetim anlamlarına
sahip olduklarından geçerli `taskName` değerleri değildir.

## Araç: `sessions_yield`

Mevcut model dönüşünü sonlandırır ve başta alt ajan tamamlanma olayları olmak üzere
çalışma zamanı olaylarının bir sonraki ileti olarak gelmesini bekler. İstekte bulunan,
gerekli alt iş oluşturulduktan sonra bu işler tamamlanmadan nihai yanıt
üretemiyorsa kullanın.

`sessions_yield` bekleme ilkelidir. Yalnızca alt öğenin tamamlandığını algılamak için
bunu `subagents`, `sessions_list`, `sessions_history`, kabuk
`sleep` veya süreç yoklama döngüleriyle değiştirmeyin.

`sessions_yield` aracını yalnızca oturumun etkin araç listesi bu aracı içerdiğinde
kullanın. Bazı asgari veya özel araç profilleri `sessions_spawn` ve
`subagents` araçlarını sunarken `sessions_yield` aracını sunmayabilir; bu durumda
yalnızca tamamlanmayı beklemek için bir yoklama döngüsü uydurmayın.

Etkin alt öğeler mevcut olduğunda OpenClaw, istekte bulunanın yoklama yapmadan
mevcut alt oturumları, çalıştırma kimliklerini, durumları, etiketleri, görevleri ve
`taskName` diğer adlarını görebilmesi için normal dönüşlere çalışma zamanı tarafından oluşturulan
küçük bir `Active Subagents` istem bloğu ekler. Bu bloktaki görev ve etiket alanları,
kullanıcı/model tarafından sağlanan oluşturma bağımsız değişkenlerinden gelebilecekleri için
talimat olarak değil, veri olarak tırnak içine alınır.

## Araç: `subagents`

İstekte bulunan oturum ağacının sahip olduğu oluşturulmuş alt ajan çalışmalarını
ve arka plan görevi kayıtlarını listeler. Görev satırları yerel alt ajanları, ACP çalışmalarını,
Gateway CLI/medya işlerini ve cron yürütmelerini kapsar. Kapsamı mevcut
istekte bulunanla sınırlıdır; bir alt öğe yalnızca kendi denetimindeki alt öğeleri görebilir.

İsteğe bağlı durum ve hata ayıklama için `subagents` kullanın. Tamamlanma olaylarını
beklemek için `sessions_yield` kullanın.

Bir görevi durdurmak için `action: "list"` tarafından döndürülen bir `taskId` ile
`action: "cancel"` kullanın. İptal işlemi denetlenen oturum ağacıyla sınırlıdır; bir yaprak
alt ajan, başka bir oturumun sahip olduğu işi iptal edemez.

## İletişim dizisine bağlı oturumlar

Bir kanal için ileti dizisi bağlamaları etkinleştirildiğinde, bir alt ajan ileti dizisine
bağlı kalabilir; böylece bu ileti dizisindeki sonraki kullanıcı iletileri aynı
alt ajan oturumuna yönlendirilmeye devam eder.

### İletişim dizisini destekleyen kanallar

Bir kanal, konuşma bağlama bağdaştırıcısı kaydettiğinde kalıcı ileti dizisine bağlı
alt ajan oturumlarını (`sessions_spawn` ile `thread: true`) destekler.
Bu desteğe sahip paketlenmiş kanallar: **Discord**, **iMessage**, **Matrix** ve
**Telegram**. Discord ve Matrix varsayılan olarak bir alt ileti dizisi oluşturur;
Telegram ve iMessage varsayılan olarak mevcut konuşmayı bağlar. Etkinleştirme,
zaman aşımları ve `spawnSessions` için kanal başına `threadBindings`
yapılandırma anahtarlarını kullanın.

### Hızlı akış

<Steps>
  <Step title="Oluştur">
    `thread: true` (ve isteğe bağlı olarak `mode: "session"`) ile `sessions_spawn`.
  </Step>
  <Step title="Bağla">
    OpenClaw, etkin kanalda o oturum hedefine bir ileti dizisi oluşturur veya bağlar.
  </Step>
  <Step title="Sonraki iletileri yönlendir">
    Bu ileti dizisindeki yanıtlar ve sonraki iletiler bağlı oturuma yönlendirilir.
  </Step>
  <Step title="Zaman aşımlarını incele">
    Etkin olmama durumunda odağın otomatik kaldırılmasını incelemek/güncellemek için `/session idle`,
    kesin üst sınırı denetlemek için `/session max-age` kullanın.
  </Step>
  <Step title="Ayır">
    El ile ayırmak için `/unfocus` kullanın.
  </Step>
</Steps>

### El ile denetimler

| Komut            | Etki                                                                                    |
| ------------------ | ----------------------------------------------------------------------------------------- |
| `/focus <target>`  | Mevcut ileti dizisini bir alt ajan/oturum hedefine bağlar (veya bir tane oluşturur)                     |
| `/unfocus`         | Bağlı mevcut ileti dizisinin bağlamasını kaldırır                                           |
| `/agents`          | Etkin çalışmaları ve bağlama durumunu listeler (`binding:<id>`, `unbound` veya `bindings unavailable`) |
| `/session idle`    | Boştayken odağın otomatik kaldırılmasını inceler/günceller (yalnızca odaklanılmış bağlı ileti dizileri)                             |
| `/session max-age` | Kesin üst sınırı inceler/günceller (yalnızca odaklanılmış bağlı ileti dizileri)                                      |

### Yapılandırma anahtarları

- **Genel varsayılan:** `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
- **Kanal geçersiz kılması ve oluşturma sırasında otomatik bağlama anahtarları** bağdaştırıcıya özeldir. Yukarıdaki [İletişim dizisini destekleyen kanallar](#thread-supporting-channels) bölümüne bakın.

Güncel bağdaştırıcı ayrıntıları için [Yapılandırma referansı](/tr/gateway/configuration-reference) ve
[Eğik çizgi komutları](/tr/tools/slash-commands) bölümlerine bakın.

### İzin listesi

<ParamField path="agents.entries.*.subagents.allowAgents" type="string[]">
  Açık `agentId` aracılığıyla hedeflenebilecek yapılandırılmış ajan kimliklerinin listesi (`["*"]`, yapılandırılmış tüm hedeflere izin verir). Varsayılan: yalnızca istekte bulunan ajan. Bir liste ayarlayıp istekte bulunanın `agentId` ile kendisini oluşturabilmesini de istiyorsanız istekte bulunanın kimliğini listeye ekleyin.
</ParamField>
<ParamField path="agents.defaults.subagents.allowAgents" type="string[]">
  İstekte bulunan ajan kendi `subagents.allowAgents` değerini ayarlamadığında kullanılan, yapılandırılmış varsayılan hedef ajan izin listesi.
</ParamField>
<ParamField path="agents.defaults.subagents.requireAgentId" type="boolean" default="false">
  `agentId` değerini atlayan `sessions_spawn` çağrılarını engeller (açık profil seçimini zorunlu kılar). Ajan başına geçersiz kılma: `agents.entries.*.subagents.requireAgentId`.
</ParamField>
<ParamField path="agents.defaults.subagents.announceTimeoutMs" type="number" default="120000">
  Gateway `agent` duyuru teslimatı denemeleri için çağrı başına zaman aşımı. Değerler pozitif tam sayı milisaniyelerdir ve platformun güvenli zamanlayıcı üst sınırına sıkıştırılır. Geçici yeniden denemeler, toplam duyuru bekleme süresini yapılandırılmış tek bir zaman aşımından daha uzun hâle getirebilir.
</ParamField>

İstekte bulunan oturum korumalı alan içindeyse `sessions_spawn`,
korumalı alan dışında çalışacak hedefleri reddeder.

### Keşif

Geçerli olarak hangi ajan kimliklerine izin verildiğini görmek için `agents_list` kullanın:
`sessions_spawn`. Yanıt, çağıranların OpenClaw, Codex
app-server ve yapılandırılmış diğer yerel çalışma zamanlarını ayırt edebilmesi için listelenen her ajanın etkin
modelini ve gömülü çalışma zamanı meta verilerini içerir.

`allowAgents` girdileri, `agents.entries.*` içinde yapılandırılmış ajan kimliklerini göstermelidir.
`["*"]`, yapılandırılmış herhangi bir hedef ajan ile istekte bulunan anlamına gelir. Bir ajan yapılandırması
silinir ancak kimliği `allowAgents` içinde kalırsa, `sessions_spawn` bu kimliği reddeder
ve `agents_list` bunu dahil etmez. Eski izin listesi girdilerini temizlemek için
`openclaw doctor --fix` çalıştırın veya hedefin varsayılanları devralırken
oluşturulabilir kalması gerekiyorsa asgari bir `agents.entries.*` girdisi ekleyin.

### Otomatik arşivleme

- Alt ajan oturumları, `agents.defaults.subagents.archiveAfterMinutes` sonrasında (varsayılan `60`) otomatik olarak arşivlenir.
- Arşivleme, `sessions.delete` kullanır ve dökümün adını `*.deleted.<timestamp>` olarak değiştirir (aynı klasörde).
- `cleanup: "delete"`, duyurudan hemen sonra arşivler (dökümü yeniden adlandırarak korumaya devam eder).
- Otomatik arşivleme azami gayret esasına dayanır; Gateway yeniden başlatılırsa bekleyen zamanlayıcılar kaybolur.
- Yapılandırılmış çalışma zaman aşımı süreleri otomatik arşivleme **yapmaz**; yalnızca çalışmayı durdurur. Oturum, otomatik arşivlemeye kadar kalır.
- Otomatik arşivleme, derinlik 1 ve derinlik 2 oturumlarına eşit biçimde uygulanır.
- Tarayıcı temizliği, arşiv temizliğinden ayrıdır: döküm/oturum kaydı korunsa bile çalışma tamamlandığında izlenen tarayıcı sekmeleri/süreçleri azami gayretle kapatılır.

## İç içe alt ajanlar

Varsayılan olarak alt ajanlar kendi alt ajanlarını oluşturamaz
(`maxSpawnDepth: 1`). Bir iç içe geçme düzeyini etkinleştirmek için `maxSpawnDepth: 2` ayarını kullanın
— **orkestratör kalıbı**: ana → orkestratör alt ajan →
çalışan alt-alt ajanlar.

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxSpawnDepth: 2, // allow sub-agents to spawn children (default: 1, range 1-5)
        maxChildrenPerAgent: 5, // max active children per agent session (default: 5, range 1-20)
        maxConcurrent: 8, // global concurrency lane cap (default: 8)
        runTimeoutSeconds: 900, // default timeout for sessions_spawn (0 = no timeout)
        announceTimeoutMs: 120000, // per-call gateway announce timeout
      },
    },
  },
}
```

### Derinlik düzeyleri

| Derinlik | Oturum anahtarı biçimi                            | Rol                                          | Oluşturabilir mi?                   |
| ----- | -------------------------------------------- | --------------------------------------------- | ---------------------------- |
| 0     | `agent:<id>:main`                            | Ana ajan                                    | Her zaman                       |
| 1     | `agent:<id>:subagent:<uuid>`                 | Alt ajan (derinlik 2'ye izin verildiğinde orkestratör) | Yalnızca `maxSpawnDepth >= 2` ise |
| 2     | `agent:<id>:subagent:<uuid>:subagent:<uuid>` | Alt-alt ajan (uç çalışan)                   | Asla                        |

### Duyuru zinciri

Sonuçlar zincirde geriye doğru akar:

1. Derinlik 2 çalışanı tamamlanır → üst öğesine (derinlik 1 orkestratörü) duyurur.
2. Derinlik 1 orkestratörü duyuruyu alır, sonuçları sentezler, tamamlanır → ana ajana duyurur.
3. Ana ajan duyuruyu alır ve kullanıcıya iletir.

Her düzey yalnızca doğrudan alt öğelerinden gelen duyuruları görür.

<Note>
**Operasyonel rehber:** `sessions_list`,
`sessions_history`, `/subagents list` veya `exec` uyku komutlarının
çevresinde yoklama döngüleri oluşturmak yerine alt öğe çalışmasını bir kez başlatın ve tamamlanma
olaylarını bekleyin.
`sessions_list` ve `/subagents list`, alt oturum ilişkilerini
canlı çalışmaya odaklı tutar — canlı alt öğeler bağlı kalır, sona ermiş alt öğeler
yakın geçmiş penceresinde kısa süre görünür kalır ve yalnızca depoda bulunan eski alt öğe bağlantıları
güncellik pencerelerinden sonra yok sayılır. Bu, eski `spawnedBy` /
`parentSessionKey` meta verilerinin yeniden başlatma sonrasında hayalet alt öğeleri
yeniden ortaya çıkarmasını önler. Son yanıtı zaten gönderdikten sonra bir alt öğe tamamlanma olayı
gelirse doğru takip yanıtı, tam olarak şu sessiz belirteçtir:
`NO_REPLY` / `no_reply`.
</Note>

### Derinliğe göre araç politikası

- Bir alt öğe oluşturulduğunda istekte bulunanın etkin gönderici politikasını yakalar. Göndericisiz alt öğe çalışmaları ve kimliği doğrulanmış operatör devam ettirmeleri, `toolsBySender` daha sonra değişse bile bu anlık görüntüyü korur; geçerli genel, ajan, sağlayıcı, korumalı alan ve alt ajan kısıtlamaları uygulanmaya devam eder. Alt öğeyi hedefleyen yeni bir harici kanal etkileşimi ise geçerli gönderici politikasını yeniden çözümler.
- Rol ve denetim kapsamı, oluşturma sırasında oturum meta verilerine yazılır. Bu, düzleştirilmiş veya geri yüklenmiş oturum anahtarlarının yanlışlıkla orkestratör ayrıcalıklarını yeniden kazanmasını önler.
- **Derinlik 1 (orkestratör, `maxSpawnDepth >= 2` olduğunda):** alt öğeler oluşturabilmesi ve durumlarını inceleyebilmesi için `sessions_spawn`, `subagents`, `sessions_list`, `sessions_history` araçlarını alır. Diğer oturum/sistem araçları reddedilmeye devam eder.
- **Derinlik 1 (uç, `maxSpawnDepth == 1` olduğunda):** oturum aracı yoktur (geçerli varsayılan davranış).
- **Derinlik 2 (uç çalışan):** oturum aracı yoktur — `sessions_spawn` derinlik 2'de her zaman reddedilir. Başka alt öğeler oluşturamaz.

### Ajan başına oluşturma sınırı

Her ajan oturumu (herhangi bir derinlikte) aynı anda en fazla `maxChildrenPerAgent`
(varsayılan `5`) etkin alt öğeye sahip olabilir. Bu, tek bir orkestratörden
denetimsiz dallanmayı önler.

### Basamaklı durdurma

Derinlik 1 orkestratörünün durdurulması, tüm derinlik 2
alt öğelerini otomatik olarak durdurur:

- Ana sohbetteki `/stop`, tüm derinlik 1 ajanlarını durdurur ve durdurmayı bunların derinlik 2 alt öğelerine yayar.

## Kimlik doğrulama

Alt ajan kimlik doğrulaması oturum türüne göre değil, **ajan kimliğine** göre çözümlenir:

- Alt ajan oturum anahtarı `agent:<agentId>:subagent:<uuid>` şeklindedir.
- Kimlik doğrulama deposu, ilgili ajanın `agentDir` konumundan yüklenir.
- Ana ajanın kimlik doğrulama profilleri bir **geri dönüş seçeneği** olarak birleştirilir; çakışmalarda ajan profilleri ana profilleri geçersiz kılar.

Birleştirme eklemelidir; bu nedenle ana profiller her zaman
geri dönüş seçenekleri olarak kullanılabilir. Ajan başına tamamen yalıtılmış kimlik doğrulama henüz desteklenmemektedir.

## Duyuru

Alt ajanlar bir duyuru adımı aracılığıyla geri bildirimde bulunur:

- Duyuru adımı, istekte bulunanın oturumunda değil alt ajan oturumunda çalışır.
- Alt ajan tam olarak `ANNOUNCE_SKIP` yanıtını verirse hiçbir şey gönderilmez.
- En son asistan metni tam olarak `NO_REPLY` / `no_reply` sessiz belirteciyse, daha önce görünür ilerleme olsa bile duyuru çıktısı engellenir.

Teslimat, istekte bulunanın derinliğine bağlıdır:

- Üst düzey istekte bulunan oturumları, harici teslimatla (`deliver=true`) bir takip `agent` çağrısı kullanır.
- İç içe istekte bulunan alt ajan oturumları, orkestratörün alt öğe sonuçlarını oturum içinde sentezleyebilmesi için dahili bir takip eklemesi (`deliver=false`) alır.
- İç içe istekte bulunan alt ajan oturumu artık yoksa OpenClaw, mevcut olduğunda ilgili oturumun istekte bulunanına geri döner.

Üst düzey istekte bulunan oturumlarında, tamamlanma modundaki doğrudan teslimat önce
bağlı konuşma/ileti dizisi rotasını ve kanca geçersiz kılmasını çözümler, ardından
eksik kanal-hedef alanlarını istekte bulunan oturumunun kayıtlı rotasından doldurur.
Bu, tamamlanmanın kaynağı yalnızca kanalı tanımlasa bile tamamlanmaların doğru sohbet/konu üzerinde
kalmasını sağlar.

İç içe tamamlanma bulguları oluşturulurken alt öğe tamamlanmalarının birleştirilmesi
geçerli istekte bulunan çalışmasıyla sınırlandırılır; böylece önceki çalışmalardan kalan eski alt öğe
çıktılarının geçerli duyuruya sızması önlenir. Duyuru yanıtları, kanal bağdaştırıcılarında
mevcut olduğunda ileti dizisi/konu yönlendirmesini korur.

### Duyuru bağlamı

Duyuru bağlamı, kararlı bir dahili olay bloğuna normalleştirilir:

| Alan          | Kaynak                                                                                                   |
| -------------- | -------------------------------------------------------------------------------------------------------- |
| Kaynak         | `subagent` veya `cron`                                                                                     |
| Oturum kimlikleri    | Alt öğe oturum anahtarı/kimliği                                                                                     |
| Tür           | Duyuru türü + görev etiketi                                                                               |
| Durum         | Çalışma zamanı sonucundan türetilir (`ok`, `error`, `timeout` veya `unknown`) — model metninden çıkarım **yapılmaz** |
| Sonuç içeriği | Alt öğenin en son görünür asistan metni                                                             |
| Takip      | Ne zaman yanıt verileceğini veya sessiz kalınacağını açıklayan talimat                                                      |

Başarısız sonlandırılmış çalışmalar, yakalanan yanıt metnini yeniden oynatmadan
başarısızlık durumunu bildirir. Araç/araçSonucu çıktısı, alt öğe sonuç metnine yükseltilmez.

### İstatistik satırı

Duyuru yükleri, sonunda bir istatistik satırı içerir (sarmalanmış olsalar bile):

- Çalışma zamanı (ör. `runtime 5m12s`).
- Belirteç kullanımı (girdi/çıktı/toplam).
- Model fiyatlandırması yapılandırıldığında tahmini maliyet (`models.providers.*.models[].cost`).
- Ana ajanın `sessions_history` aracılığıyla geçmişi getirebilmesi veya diskteki dosyayı inceleyebilmesi için `sessionKey`, `sessionId` ve döküm yolu.

Dahili meta veriler yalnızca orkestrasyon içindir; kullanıcıya yönelik yanıtlar
normal asistan üslubuyla yeniden yazılmalıdır.

### Neden `sessions_history` tercih edilmeli?

`sessions_history`, bir ajan etkileşimi içinden alt öğenin
dökümünü okumak için daha güvenli orkestrasyon yoludur:

- Genel amaçlı günlük redaksiyonu devre dışı olsa bile kimlik bilgisi/belirteç benzeri metni redakte eder.
- Uzun metin bloklarını kısaltır (blok başına 4000 karakter) ve düşünme imzalarını, akıl yürütme yeniden oynatma yüklerini ve satır içi görüntü verilerini çıkarır.
- 80 KB yanıt sınırı uygular; aşırı büyük satırlar `[sessions_history omitted: message too large]` ile değiştirilir.
- Mevcut olduğunda eski döküm pencerelerinde geriye doğru sayfalama yapmak için `nextOffset` kullanın.
- `sessions_history`, ileti metninden akıl yürütme etiketlerini, `<relevant-memories>` iskeletini veya araç çağrısı XML'ini **çıkarmaz** — yalnızca redakte edilmiş ve boyutu sınırlandırılmış şekilde, ham döküm biçimine yakın yapılandırılmış içerik blokları döndürür. `/subagents log`, yapılandırılmış bloklar yerine düz sohbet satırları oluşturduğu için daha kapsamlı düzyazı temizleyicisini uygular (akıl yürütme etiketlerini, bellek iskeletini ve araç çağrısı XML'ini çıkarır).
- Bayt bayt tam döküme ihtiyacınız olduğunda, diskteki ham dökümün incelenmesi geri dönüş seçeneğidir.

## Araç politikası

Alt ajanlar önce üst veya hedef ajanla aynı profili ve araç politikası
işlem hattını kullanır. Ardından OpenClaw, alt ajan kısıtlama
katmanını uygular.

Alt ajanlar, derinlik veya rolden bağımsız olarak `gateway`, `agents_list`, `session_status` ve
`cron` araçlarını her zaman kaybeder (sistem düzeyindeki/etkileşimli araçlar veya
ana ajanın koordine etmesi gereken araçlar). Uç alt ajanlar (varsayılan derinlik 1
davranışı ve her zaman derinlik 2'de) ayrıca `subagents`,
`sessions_list`, `sessions_history` ve `sessions_spawn` araçlarını kaybeder. Alt ajanlar hiçbir zaman
`message` aracını almaz — bu araç, bu ret listesiyle filtrelenmek yerine oluşturma sırasında
devre dışı bırakılır — ve alt ajanların yalnızca duyuru zinciri üzerinden
iletişim kurması için `sessions_send` reddedilmeye devam eder.

`sessions_history` burada da sınırlandırılmış, temizlenmiş bir anımsama görünümü olarak kalır —
ham bir döküm çıktısı değildir.

`maxSpawnDepth >= 2` olduğunda, derinlik 1 orkestratör alt ajanları alt öğelerini
yönetebilmeleri için ayrıca `sessions_spawn`, `subagents`, `sessions_list` ve
`sessions_history` araçlarını alır.

### Yapılandırma aracılığıyla geçersiz kılma

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxConcurrent: 1,
      },
    },
  },
  tools: {
    subagents: {
      tools: {
        // deny önceliklidir
        deny: ["gateway", "cron"],
        // allow ayarlanırsa yalnızca izin verilenlere dönüşür (deny yine önceliklidir)
        // allow: ["read", "exec", "process"]
      },
    },
  },
}
```

`tools.subagents.tools.allow` son bir yalnızca izin verilenler filtresidir. Önceden çözümlenmiş
araç kümesini daraltabilir ancak `tools.profile` tarafından kaldırılan bir aracı
**geri ekleyemez**. Örneğin, `tools.profile: "coding"`
`web_search`/`web_fetch` öğelerini içerir ancak `browser` aracını içermez.
Kodlama profilli alt aracıların tarayıcı otomasyonu kullanabilmesi için tarayıcıyı
profil aşamasında ekleyin:

```json5
{
  tools: {
    profile: "coding",
    alsoAllow: ["browser"],
  },
}
```

Yalnızca tek bir aracının tarayıcı otomasyonuna erişmesi gerekiyorsa aracı başına
`agents.entries.*.tools.alsoAllow: ["browser"]` kullanın.

## Eşzamanlılık

Alt aracılar, özel bir işlem içi kuyruk hattı kullanır:

- **Hat adı:** `subagent`
- **Eşzamanlılık:** `agents.defaults.subagents.maxConcurrent` (varsayılan `8`)

## Canlılık ve kurtarma

OpenClaw, `endedAt` bulunmamasını bir
alt aracının hâlâ çalıştığına dair kalıcı kanıt olarak değerlendirmez. Eski çalıştırma penceresinden
(2 saat veya yapılandırılmış çalıştırma zaman aşımı ile kısa bir ek sürenin toplamından
hangisi daha uzunsa) daha eski, sonlandırılmamış çalıştırmalar; `/subagents list`,
durum özetleri, alt öğe tamamlanma denetimi ve oturum başına
eşzamanlılık kontrollerinde etkin/beklemede olarak sayılmaz.

Gateway yeniden başlatıldıktan sonra, eski ve sonlandırılmamış geri yüklenmiş çalıştırmalar,
bunların alt oturumu `abortedLastRun: true` olarak işaretlenmemişse temizlenir. Yeniden başlatma nedeniyle
iptal edilen çalıştırmalar, alt aracı yetim kurtarma akışına kayıtlı kalır: eski
çalıştırmalar devam ettirilmeden sonlandırılırken yeni alt oturumlar, iptal işareti
temizlenmeden önce yapay bir devam ettirme mesajı alır.

Otomatik yeniden başlatma kurtarması, alt oturum başına sınırlıdır. Aynı
alt aracı alt öğesi, hızlı yeniden kilitlenme penceresi içinde yetim kurtarma için
tekrar tekrar kabul edilirse OpenClaw bu oturumda bir kurtarma mezar taşı kalıcılaştırır
ve sonraki yeniden başlatmalarda oturumu otomatik olarak devam ettirmeyi durdurur. Görev
kaydını uzlaştırmak için `openclaw tasks maintenance --apply` komutunu veya mezar taşı bulunan oturumlardaki
eski iptal edilmiş kurtarma işaretlerini temizlemek için
`openclaw doctor --fix` komutunu çalıştırın.

<Note>
Bir alt aracı başlatma işlemi Gateway `PAIRING_REQUIRED` /
`scope-upgrade` hatasıyla başarısız olursa eşleştirme durumunu düzenlemeden önce RPC çağıranını
kontrol edin. Çağıran zaten Gateway istek bağlamında çalışıyorsa dahili
`sessions_spawn` koordinasyon gönderimleri işlem içinde gerçekleşir; bu nedenle
geri döngü WebSocket'i açılmaz ve CLI'ın eşleştirilmiş cihaz kapsamı temel değerine
bağlı değildir. Gateway işlemi dışındaki çağıranlar, doğrudan geri döngü
paylaşılan belirteç/parola kimlik doğrulaması üzerinden `client.mode: "backend"` ile
`client.id: "gateway-client"` olarak WebSocket yedeğini kullanmaya devam eder. Uzak çağıranlar, açık
`deviceIdentity`, açık cihaz belirteci yolları ve tarayıcı/Node istemcileri,
kapsam yükseltmeleri için yine normal cihaz onayı gerektirir.
</Note>

## Durdurma

- İstekte bulunan sohbette `/stop` göndermek, istekte bulunan oturumu iptal eder ve buradan başlatılan tüm etkin alt aracı çalıştırmalarını durdurarak bu işlemi iç içe alt öğelere yayar.

## Sınırlamalar

- Alt aracı duyurusu **mümkün olan en iyi çabayla** gerçekleştirilir. Gateway yeniden başlatılırsa bekleyen "geri duyurma" işi kaybolur.
- Alt aracılar yine aynı Gateway işlemi kaynaklarını paylaşır; `maxConcurrent` öğesini bir güvenlik valfi olarak değerlendirin.
- `sessions_spawn` her zaman engellemesizdir: `{ status: "accepted", runId, childSessionKey }` değerini hemen döndürür.
- Alt aracı bağlamı yalnızca `AGENTS.md` ve `TOOLS.md` öğelerini ekler (`SOUL.md`, `IDENTITY.md`, `USER.md`, `MEMORY.md`, `HEARTBEAT.md` veya `BOOTSTRAP.md` eklenmez). Codex'e özgü alt aracılar aynı sınırı izler: `TOOLS.md` devralınan Codex iş parçacığı talimatlarında kalırken yalnızca üst öğeye ait persona, kimlik ve kullanıcı dosyaları, alt öğelerin bunları kopyalamaması için tur kapsamlı iş birliği talimatları olarak eklenir.
- En fazla iç içe geçme derinliği 5'tir (`maxSpawnDepth` aralığı: 1-5). Çoğu kullanım durumu için 2 derinliği önerilir.
- `maxChildrenPerAgent`, oturum başına etkin alt öğeleri sınırlar (varsayılan `5`, aralık `1-20`).

## İlgili

- [Oturum araçları ve durum değişiklikleri](/tr/concepts/session-tool)
- [ACP aracıları](/tr/tools/acp-agents)
- [Aracı gönderimi](/tr/tools/agent-send)
- [Arka plan görevleri](/tr/automation/tasks)
- [Çok aracılı korumalı alan araçları](/tr/tools/multi-agent-sandbox-tools)
