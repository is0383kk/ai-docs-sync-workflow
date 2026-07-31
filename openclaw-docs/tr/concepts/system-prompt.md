---
read_when:
    - Sistem istemi metnini, araç listesini veya zaman/heartbeat bölümlerini düzenleme
    - Çalışma alanı bootstrap veya Skills ekleme davranışını değiştirme
summary: OpenClaw sistem isteminin içeriği ve nasıl oluşturulduğu
title: Sistem istemi
x-i18n:
    generated_at: "2026-07-26T22:44:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 669fbc6f21a82a2c3c067d2ff3a6365acb3316460a85f2db165b7ad49ce79f70
    source_path: concepts/system-prompt.md
    workflow: 16
---

OpenClaw, her agent çalıştırması için kendi sistem istemini oluşturur; çalışma zamanında varsayılan bir istem yoktur.

Birleştirme üç katmandan oluşur:

- `buildAgentSystemPrompt`, istemi açık girdilerden oluşturur. Saf bir oluşturucu olarak kalır ve genel yapılandırmayı doğrudan okumaz.
- `resolveAgentSystemPromptConfig`, belirli bir agent için yapılandırma destekli istem ayarlarını (sahip görünümü, TTS ipuçları, model takma adları, bellek alıntı modu, alt agent yetkilendirme modu) çözümler.
- Çalışma zamanı bağdaştırıcıları (gömülü, CLI, komut/dışa aktarma önizlemeleri, compaction) canlı olguları (araçlar, sandbox durumu, kanal yetenekleri, bağlam dosyaları, sağlayıcı istem katkıları) toplar ve yapılandırılmış istem cephesini çağırır.

Bu, her çalışma zamanı ayrıntısını tek ve yekpare bir oluşturucuya dönüştürmeden dışa aktarılan/hata ayıklama istem yüzeylerinin canlı çalıştırmalarla uyumlu kalmasını sağlar.

Sağlayıcı plugin'leri, OpenClaw'a ait istemi değiştirmeden önbellek farkındalığına sahip yönergeler sağlayabilir. Bir sağlayıcı çalışma zamanı şunları yapabilir:

- adlandırılmış üç temel bölümden birini değiştirebilir: `interaction_style`, `tool_call_style`, `execution_bias`
- istem önbelleği sınırının üzerine **kararlı bir önek** ekleyebilir
- istem önbelleği sınırının altına **dinamik bir sonek** ekleyebilir

Model ailesine özgü ayarlamalar için sağlayıcıya ait katkıları kullanın. Eski `before_prompt_build` kancasını uyumluluk veya gerçekten genel istem değişiklikleri için ayırın.

Paketlenmiş OpenAI/Codex GPT-5 ailesi katmanı (`resolveGpt5SystemPromptContribution`) bu mekanizmayı kullanır: bir `stablePrefix` davranış sözleşmesi (yürütme politikası, araç disiplini, çıktı sözleşmesi, tamamlama sözleşmesi) ve daha samimi bir ton için isteğe bağlı bir `interaction_style` geçersiz kılması. OpenAI veya Codex plugin'leri üzerinden yönlendirilen tüm `gpt-5*` model kimliklerine uygulanır ve `agents.defaults.promptOverlays.gpt5.personality` (`"friendly"`/`"on"` veya `"off"`) tarafından denetlenir.

## Yapı

İstem, sabit bölümlerden oluşan kompakt bir yapıdadır:

- **Araçlar**: yapılandırılmış araçların temel doğruluk kaynağı olduğuna ilişkin hatırlatma ve çalışma zamanı araç kullanım yönergeleri. Deneysel `update_plan` aracı etkinleştirildiğinde (`tools.experimental.planTool`), kendi araç açıklaması şunları ekler: yalnızca basit olmayan çok adımlı işler için kullanın, en fazla bir adımı `in_progress` durumunda tutun ve tek adımlı basit işler için kullanmayın.
- **Yürütme Eğilimi**: işlem yapılabilir isteklere aynı turda yanıt verin, tamamlanana veya engellenene kadar devam edin, yetersiz araç sonuçlarından sonra toparlanın, değişebilir durumu canlı olarak kontrol edin ve sonlandırmadan önce doğrulayın.
- **Güvenlik**: güç arayışındaki davranışlara veya gözetimi atlatmaya karşı kısa güvenlik sınırı hatırlatması.
- **Skills** (mevcut olduğunda): modele Skills talimatlarını gerektiğinde nasıl yükleyeceğini bildirir.
- **OpenClaw Denetimi**: yapılandırma/yeniden başlatma işleri için `gateway` aracını tercih edin; CLI komutları uydurmayın.
- **OpenClaw Kendi Kendini Güncelleme**: yapılandırmayı `config.schema.lookup` ile güvenle inceleyin, `config.patch` ile yamalayın, yapılandırmanın tamamını `config.apply` ile değiştirin ve `update.run` komutunu yalnızca kullanıcının açık isteği üzerine çalıştırın. Agent'a yönelik `gateway` aracı, `tools.exec.mode` öğesini yeniden yazmayı reddeder.
- **Çalışma Alanı**: çalışma dizini (`agents.defaults.workspace`).
- **Belgelendirme**: yerel belge/kaynak yolu ve bunların ne zaman okunacağı.
- **Çalışma Alanı Dosyaları (eklenmiş)**: önyükleme dosyalarının aşağıya eklendiğini belirtir.
- **Sandbox** (etkin olduğunda): sandbox içindeki çalışma zamanı, sandbox yolları, yükseltilmiş yürütme kullanılabilirliği.
- **Geçerli Tarih ve Saat**: yalnızca saat dilimi (önbellek açısından kararlıdır; canlı saat `session_status` kaynağından gelir).
- **Asistan Çıktı Yönergeleri**: kompakt ek, sesli not ve yanıt etiketi söz dizimi.
- **Heartbeat'ler**: varsayılan agent için heartbeat'ler etkinleştirildiğinde heartbeat istemi ve onay davranışı.
- **Çalışma Zamanı**: ana makine, işletim sistemi, node, model, depo kökü (algılandığında), düşünme düzeyi (tek satır).
- **Akıl Yürütme**: geçerli görünürlük düzeyi ve `/reasoning` geçiş ipucu.

**Proje Bağlamı** dâhil olmak üzere büyük ve kararlı içerik, dahili istem önbelleği sınırının üzerinde kalır. Değişken, tur başına bölümler (Control UI gömme yönergeleri, **Mesajlaşma**, **Ses**, **Grup Sohbeti Bağlamı**, **Tepkiler**, **Heartbeat'ler**, **Çalışma Zamanı**) bu sınırın altına eklenir; böylece önek önbelleklerine sahip yerel arka uçlar, kararlı çalışma alanı önekini kanal turları arasında yeniden kullanabilir. Kabul edilen şema bu çalışma zamanı ayrıntısını zaten taşıyorsa araç açıklamalarına geçerli kanal adları gömülmemelidir.

Araçlar ayrıca uzun süreli çalışma yönergelerini de içerir:

- gelecekteki takip işlemleri (`check back later`, hatırlatıcılar, yinelenen işler) için `exec` uyku döngüleri, `yieldMs` geciktirme hileleri veya yinelenen `process` yoklamaları yerine cron kullanın
- yalnızca şimdi başlayan ve arka planda devam eden komutlar için `exec` / `process` kullanın
- otomatik tamamlanma uyandırması etkin olduğunda komutu bir kez başlatın ve gönderim tabanlı uyandırma yoluna güvenin
- çalışan bir komutun günlükleri, durumu, girdisi veya müdahalesi için `process` kullanın
- daha büyük görevler için `sessions_spawn` tercih edin; alt agent tamamlanması gönderim tabanlıdır ve istekte bulunana otomatik olarak bildirilir
- yalnızca tamamlanmayı beklemek için `subagents list` / `sessions_list` öğelerini döngü içinde yoklamayın

`agents.defaults.subagents.delegationMode` (varsayılan `"suggest"`) bunu güçlendirebilir. `"prefer"`, ana agent'a hızlı yanıt veren bir koordinatör olarak hareket etmesini ve doğrudan yanıttan daha kapsamlı olan her şeyi `sessions_spawn` üzerinden yönlendirmesini bildiren özel bir **Alt Agent Yetkilendirmesi** bölümü ekler. Bu yalnızca istem düzeyindedir; `sessions_spawn` aracının kullanılabilir olup olmadığını yine araç politikası denetler.

Sistem istemindeki güvenlik sınırları tavsiye niteliğindedir, yaptırım değildir. Kesin yaptırım için araç politikasını, yürütme onaylarını, sandbox kullanımını ve kanal izin listelerini kullanın; operatörler istem güvenlik sınırlarını tasarım gereği devre dışı bırakabilir.

Yerel onay kartları/düğmeleri bulunan kanallarda istem, agent'a önce bu kullanıcı arayüzüne güvenmesini ve yalnızca araç sonucu sohbet onaylarının kullanılamadığını veya tek yolun manuel onay olduğunu belirttiğinde manuel bir `/approve` komutu eklemesini söyler.

## İstem modları

OpenClaw, alt agent'lar için daha küçük sistem istemleri oluşturur. Çalışma zamanı, her çalıştırma için bir `promptMode` belirler (kullanıcıya yönelik bir yapılandırma değildir):

- `full` (varsayılan): yukarıdaki tüm bölümler.
- `minimal`: alt agent'lar için kullanılır; bellek istemi bölümünü (**Belleği Hatırlama** olarak paketlenir), **OpenClaw Kendi Kendini Güncelleme**, **Model Takma Adları**, **Kullanıcı Kimliği**, **Asistan Çıktı Yönergeleri**, **Mesajlaşma**, **Sessiz Yanıtlar** ve **Heartbeat'ler** bölümlerini atlar. Araçlar, **Güvenlik**, **Skills** (sağlandığında), Çalışma Alanı, Sandbox, Geçerli Tarih ve Saat (bilindiğinde), Çalışma Zamanı ve eklenen bağlam kullanılabilir durumda kalır.
- `none`: yalnızca temel kimlik satırını döndürür.

`promptMode=minimal` altında, eklenen ilave istemler **Grup Sohbeti Bağlamı** yerine **Alt Agent Bağlamı** olarak etiketlenir.

Kanal otomatik yanıt çalıştırmalarında OpenClaw, doğrudan, grup veya yalnızca mesaj aracı bağlamı görünür yanıt sözleşmesini zaten üstlendiğinde genel **Sessiz Yanıtlar** bölümünü atlar. Yalnızca eski otomatik grup/kanal modu `NO_REPLY` öğesini gösterir; doğrudan sohbetler ve yalnızca mesaj aracı yanıtları sessiz belirteç yönergelerini atlar.

## İstem anlık görüntüleri

OpenClaw, Codex çalışma zamanının başarılı yolu için kaydedilmiş istem anlık görüntülerini `test/fixtures/agents/prompt-snapshots/codex-runtime-happy-path/` altında tutar. Bunlar, seçili uygulama sunucusu iş parçacığı/tur parametrelerini ve Telegram doğrudan, Discord grup ve heartbeat turları için yeniden oluşturulmuş model bağlantılı istem katmanı yığınını oluşturur: sabitlenmiş bir Codex `gpt-5.5` model istemi fikstürü, Codex başarılı yol izin geliştirici metni, OpenClaw geliştirici talimatları, OpenClaw tarafından sağlandığında tur kapsamlı iş birliği modu talimatları, kullanıcı tur girdisi ve dinamik araç özelliklerine başvurular.

Sabitlenmiş Codex model istemi fikstürünü `pnpm prompt:snapshots:sync-codex-model` ile yenileyin. Varsayılan olarak önce `$CODEX_HOME/models_cache.json`, ardından `~/.codex/models_cache.json`, son olarak bakımcı çalışma kopyası kuralı olan `~/code/codex/codex-rs/models-manager/models.json` aranır; hiçbiri yoksa kaydedilmiş fikstür değiştirilmeden çıkılır. Belirli bir `models_cache.json` veya `models.json` dosyasından yenilemek için `--catalog <path>` geçin.

Bu anlık görüntüler, ham OpenAI isteğinin bayt bayt yakalanmış hâli değildir. Codex, OpenClaw iş parçacığı ve tur parametrelerini gönderdikten sonra çalışma zamanına ait çalışma alanı bağlamını (`AGENTS.md`, ortam bağlamı, bellekler, uygulama/plugin talimatları, yerleşik Varsayılan iş birliği modu talimatları) ekleyebilir.

`pnpm prompt:snapshots:gen` ile yeniden oluşturun; sapmayı `pnpm prompt:snapshots:check` ile doğrulayın. CI, sapma denetimini ek sınır parçalarıyla birlikte çalıştırır; böylece istem değişiklikleri ve anlık görüntü güncellemeleri aynı PR'da yer alır.

## Çalışma alanı önyükleme ekleme işlemi

Önyükleme dosyaları etkin çalışma alanından çözümlenir ve yaşam sürelerine uyan istem yüzeyine yönlendirilir:

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (yalnızca yepyeni çalışma alanlarında)
- `MEMORY.md` mevcut olduğunda

Yerel Codex çalıştırma çerçevesinde OpenClaw, kararlı çalışma alanı dosyalarını her kullanıcı turunda yinelemekten kaçınır. Codex, `AGENTS.md` öğesini kendi proje belgesi keşfi aracılığıyla yükler. `TOOLS.md`, devralınan Codex geliştirici talimatları olarak iletilir. `SOUL.md`, `IDENTITY.md` ve `USER.md`, yerel Codex alt agent'larının bunları devralmaması için tur kapsamlı iş birliği geliştirici talimatları olarak iletilir. `HEARTBEAT.md` içeriği doğrudan eklenmez; heartbeat turları, dosya mevcut ve boş değilse dosyaya işaret eden bir iş birliği modu notu alır. `MEMORY.md` içeriği de her yerel Codex turuna yapıştırılmaz: çalışma alanı için bellek araçları kullanılabilir olduğunda Codex turları, modeli `memory_search` veya `memory_get` öğesine yönlendiren küçük bir çalışma alanı belleği notu alır. Araçlar devre dışıysa, bellek araması kullanılamıyorsa veya etkin çalışma alanı agent bellek çalışma alanından farklıysa `MEMORY.md`, normal sınırlı tur bağlamı yoluna geri döner. `BOOTSTRAP.md`, normal tur bağlamı rolünü korur.

Codex dışı çalıştırma çerçevelerinde önyükleme dosyaları, mevcut geçitlerine göre OpenClaw isteminde birleştirilir. Varsayılan agent için heartbeat'ler devre dışı olduğunda veya `agents.defaults.heartbeat.includeSystemPromptSection` false olduğunda normal çalıştırmalarda `HEARTBEAT.md` atlanır. Eklenen dosyaları, özellikle Codex dışı `MEMORY.md` dosyasını kısa tutun: ayrıntılı günlük notlar `memory/*.md` içinde bulunup `memory_search` / `memory_get` aracılığıyla gerektiğinde alınabilirken bu dosya özenle seçilmiş uzun vadeli bir özet olarak kalmalıdır. Aşırı büyük Codex dışı `MEMORY.md` dosyaları istem kullanımını artırır ve aşağıdaki önyükleme dosyası sınırları kapsamında kısmen eklenebilir.

<Note>
`memory/*.md` günlük dosyaları, normal önyükleme Proje Bağlamının parçası **değildir**. Olağan turlarda bunlara `memory_search` / `memory_get` aracılığıyla gerektiğinde erişilir; dolayısıyla model bunları açıkça okumadığı sürece bağlam penceresine dâhil edilmezler. Yalın `/new` ve `/reset` turları istisnadır: çalışma zamanı, ilk tur için yakın tarihli günlük belleği tek seferlik bir başlangıç bağlamı bloğu olarak başa ekleyebilir.
</Note>

Büyük dosyalar bir işaretçiyle kısaltılır:

| Sınır                                        | Yapılandırma anahtarı                                         | Varsayılan  |
| -------------------------------------------- | -------------------------------------------------- | -------- |
| Dosya başına azami karakter                      | `agents.defaults.bootstrapMaxChars`                | 20000    |
| Tüm dosyalardaki toplam                       | `agents.defaults.bootstrapTotalMaxChars`           | 60000    |
| Kısaltma uyarısı (`off`\|`once`\|`always`) | `agents.defaults.bootstrapPromptTruncationWarning` | `always` |

Eksik dosyalar kısa bir eksik-dosya işaretçisi ekler. Ayrıntılı ham/eklenen sayımlar `/context`, `/status`, doctor ve günlükler gibi tanılama kaynaklarında kalır.

Bellek dosyalarında kısaltma veri kaybı değildir: dosya diskte olduğu gibi kalır. Yerel Codex'te `MEMORY.md`, kullanılabilir olduğunda bellek araçları aracılığıyla isteğe bağlı olarak okunur; aksi durumda sınırlı bir istem geri dönüşü kullanılır. Diğer çalışma düzeneklerinde model, belleği doğrudan okuyana veya arayana kadar yalnızca kısaltılmış eklenen kopyayı görür. `MEMORY.md` sürekli kısaltılıyorsa onu daha kısa ve kalıcı bir özete dönüştürün, ayrıntılı geçmişi `memory/*.md` içine taşıyın veya bootstrap sınırlarını bilinçli olarak yükseltin.

Alt ajan oturumları yalnızca `AGENTS.md` ve `TOOLS.md` öğelerini ekler (alt ajan bağlamını küçük tutmak için diğer bootstrap dosyaları filtrelenir).

Dahili kancalar, eklenen bootstrap dosyalarını değiştirmek veya yenileriyle değiştirmek için `agent:bootstrap` olayı aracılığıyla bu adıma müdahale edebilir (örneğin `SOUL.md` öğesini alternatif bir kişilikle değiştirmek).

Daha az genel bir üslup için [SOUL.md Kişilik Kılavuzu](/tr/concepts/soul) ile başlayın.

Eklenen her dosyanın ne kadar katkıda bulunduğunu (ham ve eklenen, kısaltma, araç şeması ek yükü) incelemek için `/context list` veya `/context detail` kullanın. Bkz. [Bağlam](/tr/concepts/context).

## Zaman yönetimi

**Geçerli Tarih ve Saat** bölümü yalnızca kullanıcının saat dilimi bilindiğinde görünür ve istem önbelleğini kararlı tutmak için yalnızca **saat dilimini** içerir (dinamik saat veya saat biçimi içermez).

Ajanın geçerli saate ihtiyacı olduğunda `session_status` kullanın; durum kartında bir zaman damgası satırı bulunur. Aynı araç isteğe bağlı olarak oturum başına bir model geçersiz kılması ayarlayabilir (`model=default` bunu temizler).

Şunlarla yapılandırın:

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat` (`auto` | `12` | `24`)

Tam davranış ayrıntıları için [Saat Dilimleri](/tr/concepts/timezone) ve [Tarih ve Saat](/tr/date-time) bölümlerine bakın.

## Skills

Uygun Skills mevcut olduğunda OpenClaw, her skill için **dosya yolu** ve içerikten türetilmiş bir `<version>sha256:...</version>` işaretçisi içeren kompakt bir `<available_skills>` listesi (`formatSkillsForPrompt`) ekler. İstem, modele listelenen konumdaki (çalışma alanı, yönetilen veya paketlenmiş) SKILL.md dosyasını yüklemek için `read` kullanmasını ve bir skill'in `<version>` değeri önceki turdakinden farklıysa skill'i yeniden okumasını söyler. Uygun Skills yoksa Skills bölümü atlanır.

Yerel Codex turları, tam zamanlanmış istemi koruyan hafif cron turları dışında, bu listeyi tur başına kullanıcı girdisi yerine tur kapsamlı iş birliği geliştirici talimatları olarak alır. Diğer çalışma düzenekleri normal istem bölümünü korur.

Konum, `skills/personal/foo/SKILL.md` gibi iç içe yerleştirilmiş bir skill'i gösterebilir. İç içe yerleştirme yalnızca düzenleme amaçlıdır; istem, `SKILL.md` frontmatter bölümündeki düz skill adını kullanır.

Uygunluk; skill meta verisi geçitlerini, çalışma zamanı ortamı/yapılandırma denetimlerini ve `agents.defaults.skills` veya `agents.entries.*.skills` yapılandırıldığında etkin ajan skill izin listesini kapsar. Plugin ile paketlenmiş Skills yalnızca sahibi olan Plugin etkinleştirildiğinde uygundur; böylece araç Plugin'leri, tüm bu yönlendirmeleri her araç açıklamasına gömmeden daha ayrıntılı kullanım kılavuzları sunabilir.

```xml
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
    <version>sha256:...</version>
  </skill>
</available_skills>
```

Bu, hedefli skill kullanımını mümkün kılarken temel istemi küçük tutar. Boyutlandırma, genel çalışma zamanı okuma/ekleme boyutlandırmasından ayrı olarak Skills alt sistemi tarafından yönetilir:

| Kapsam    | Skills istem bütçesi                                 | Çalışma zamanı alıntı bütçesi       |
| --------- | ---------------------------------------------------- | ---------------------------------- |
| Genel     | `skills.limits.maxSkillsPromptChars`                 | `agents.defaults.contextLimits.*`  |
| Ajan başına | `agents.entries.*.skillsLimits.maxSkillsPromptChars` | `agents.entries.*.contextLimits.*` |

Çalışma zamanı alıntı bütçesi; `memory_get`, canlı araç sonuçları ve Compaction sonrası `AGENTS.md` yenilemelerini kapsar.

## Dokümantasyon

**Dokümantasyon** bölümü, kullanılabilir olduğunda yerel dokümanlara (Git çalışma kopyasında `docs/` veya paketlenmiş npm paketi dokümanları) yönlendirir; aksi durumda [https://docs.openclaw.ai](https://docs.openclaw.ai) adresine geri döner. Ayrıca OpenClaw kaynak konumunu listeler: Git çalışma kopyaları yerel kaynak kökünü gösterirken paket kurulumları, dokümanlar eksik veya güncelliğini yitirmiş olduğunda kaynağın orada incelenmesi gerektiğine ilişkin talimatlarla birlikte GitHub kaynak URL'sini gösterir.

İstem, model OpenClaw'ın nasıl çalıştığını (bellek/günlük notlar, oturumlar, araçlar, Gateway, yapılandırma, komutlar, proje bağlamı) anlamadan önce dokümanları OpenClaw öz bilgisi için yetkili kaynak olarak tanımlar ve modele `AGENTS.md`, proje bağlamı, çalışma alanı/profil/bellek notları ve `memory_search` öğelerini OpenClaw tasarım/uygulama bilgisi yerine talimat bağlamı veya kullanıcı belleği olarak ele almasını söyler. Dokümanlar bu konuda bilgi içermiyorsa veya güncelliğini yitirmişse model bunu belirtmeli ve kaynağı incelemelidir. Ayrıca modele mümkün olduğunda `openclaw status` komutunu kendisinin çalıştırmasını, yalnızca erişimi olmadığında kullanıcıya sormasını söyler.

Özellikle yapılandırma için ajanları, alan düzeyindeki kesin dokümanlar ve kısıtlamalar amacıyla `gateway` aracının `config.schema.lookup` eylemine, ardından daha kapsamlı yönlendirme için `docs/gateway/configuration.md` ve `docs/gateway/configuration-reference.md` öğelerine yönlendirir.

## İlgili

- [Ajan çalışma zamanı](/tr/concepts/agent)
- [Ajan çalışma alanı](/tr/concepts/agent-workspace)
- [Bağlam motoru](/tr/concepts/context-engine)
