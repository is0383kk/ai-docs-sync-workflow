---
read_when:
    - Kodlama düzeneklerini ACP üzerinden çalıştırma
    - Mesajlaşma kanallarında konuşmaya bağlı ACP oturumlarını ayarlama
    - Bir mesaj kanalı görüşmesini kalıcı bir ACP oturumuna bağlama
    - ACP arka ucu, Plugin bağlantıları veya tamamlama teslimatıyla ilgili sorunları giderme
    - Sohbetten /acp komutlarını çalıştırma
sidebarTitle: ACP agents
summary: Harici kodlama düzeneklerini (Claude Code, Cursor, Gemini CLI, açık Codex ACP, OpenClaw ACP, OpenCode) ACP arka ucu üzerinden çalıştırın
title: ACP aracıları
x-i18n:
    generated_at: "2026-07-26T23:02:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fc7f32ff927c7e949be1595f6aa00ed034a51185c6a6b1e0df01a242954667d1
    source_path: tools/acp-agents.md
    workflow: 16
---

[Agent Client Protocol (ACP)](https://agentclientprotocol.com/) oturumları,
OpenClaw'ın bir ACP arka uç plugin'i üzerinden harici kodlama yürütme ortamlarını (Claude Code, Cursor, Copilot, Droid,
OpenClaw ACP, OpenCode, Gemini CLI ve desteklenen diğer ACPX yürütme ortamları)
çalıştırmasına olanak tanır. Her başlatma bir
[arka plan görevi](/tr/automation/tasks) olarak izlenir.

<Note>
**ACP, varsayılan Codex yolu değil, harici yürütme ortamı yoludur.** Yerel
Codex uygulama sunucusu plugin'i, agent turları için `/codex ...` denetimlerinin ve varsayılan
`openai/gpt-*` gömülü çalışma zamanının sahibidir; ACP ise `/acp ...` denetimlerinin
ve `sessions_spawn({ runtime: "acp" })` oturumlarının sahibidir.

Codex veya Claude Code'un harici bir MCP istemcisi olarak doğrudan mevcut
OpenClaw kanal konuşmalarına bağlanmasına izin vermek için ACP yerine
[`openclaw mcp serve`](/tr/cli/mcp) kullanın.
</Note>

## Hangi sayfayı kullanmalıyım?

| Şunu yapmak istiyorsanız...                                                                     | Bunu kullanın                          | Notlar                                                                                                                                                                       |
| ----------------------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Mevcut konuşmada Codex'i bağlamak veya denetlemek                                                | `/codex bind`, `/codex threads`       | `codex` plugin'i etkinleştirildiğinde yerel Codex uygulama sunucusu yolu: bağlı sohbet yanıtları, görüntü iletme, model/hızlı/izinler, durdurma ve yönlendirme. ACP açıkça seçilen bir geri dönüş yoludur |
| Claude Code, Gemini CLI, açıkça seçilen Codex ACP veya başka bir harici yürütme ortamını OpenClaw _üzerinden_ çalıştırmak | Bu sayfa                             | Sohbete bağlı oturumlar, `/acp spawn`, `sessions_spawn({ runtime: "acp" })`, arka plan görevleri, çalışma zamanı denetimleri                                                                 |
| Bir OpenClaw Gateway oturumunu bir düzenleyici veya istemci için ACP sunucusu _olarak_ sunmak    | [`openclaw acp`](/tr/cli/acp)            | Köprü modu: bir IDE/istemci, stdio/WebSocket üzerinden OpenClaw ile ACP kullanarak iletişim kurar                                                                                                      |
| Yerel bir AI CLI'ını yalnızca metin destekleyen geri dönüş modeli olarak yeniden kullanmak       | [CLI Arka Uçları](/tr/gateway/cli-backends) | ACP değildir: OpenClaw araçları, ACP denetimleri veya yürütme ortamı çalışma zamanı yoktur                                                                                                             |

## Bu, doğrudan kullanılabilir mi?

Evet, resmî ACP çalışma zamanı plugin'ini yükledikten sonra:

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

Kaynak kod çalışma kopyaları, `pnpm install` sonrasında yerel
`extensions/acpx` çalışma alanı plugin'ini kullanabilir. Hazır olma denetimi için `/acp doctor` çalıştırın.

OpenClaw, agent'lara ACP ile başlatma hakkında yalnızca ACP **gerçekten kullanılabilir** olduğunda
bilgi verir: ACP etkinleştirilmiş olmalı, sevk etme devre dışı bırakılmamış olmalı, mevcut oturum
korumalı alan tarafından engellenmemiş olmalı ve bir çalışma zamanı arka ucu yüklenmiş ve sağlıklı olmalıdır. Herhangi
bir koşul karşılanmazsa agent'ın kullanılamayan bir arka uç önermemesi için ACP Skills ve
`sessions_spawn` ACP rehberliği gizli kalır.

<AccordionGroup>
  <Accordion title="İlk çalıştırmada dikkat edilmesi gerekenler">
    - `plugins.allow` ayarlanmışsa bu, kısıtlayıcı bir plugin envanteridir ve `acpx` öğesini **içermelidir**; aksi takdirde yüklü ACP arka ucu kasıtlı olarak engellenir (`/acp doctor` eksik izin listesi girdisini bildirir).
    - Codex ACP bağdaştırıcısı `acpx` plugin'iyle birlikte gelir ve mümkün olduğunda yerel olarak başlatılır.
    - Codex ACP, yalıtılmış bir `CODEX_HOME` ile çalışır. OpenClaw, güvenilen proje güven girdilerini ve güvenli model/sağlayıcı yönlendirme yapılandırmasını (`model`, `model_provider`, `model_reasoning_effort`, `sandbox_mode` ve güvenli `model_providers.<name>` alanları) ana makinedeki Codex yapılandırmasından kopyalar; kimlik doğrulama, bildirimler ve kancalar yalnızca ana makine yapılandırmasında kalır.
    - Diğer hedef yürütme ortamı bağdaştırıcıları ilk kullanımda `npx` ile talep üzerine getirilebilir.
    - Bu yürütme ortamı için sağlayıcı kimlik doğrulaması ana makinede önceden mevcut olmalıdır.
    - Ana makinede npm veya ağ erişimi yoksa önbellekler önceden doldurulana ya da bağdaştırıcı başka bir yöntemle yüklenene kadar ilk çalıştırmadaki bağdaştırıcı getirme işlemleri başarısız olur.

  </Accordion>
  <Accordion title="Çalışma zamanı ön koşulları">
    ACP gerçek bir harici yürütme ortamı süreci başlatır. OpenClaw yönlendirmenin,
    arka plan görevi durumunun, teslimatın, bağlamaların ve politikanın sahibidir; yürütme ortamı ise
    sağlayıcı oturum açma işleminin, model kataloğunun, dosya sistemi davranışının ve yerel araçların sahibidir.

    OpenClaw'ı sorumlu tutmadan önce şunları doğrulayın:

    - `/acp doctor` etkin ve sağlıklı bir arka uç bildiriyor.
    - Bu izin listesi ayarlandığında hedef kimliğine `acp.allowedAgents` tarafından izin veriliyor.
    - Yürütme ortamı komutu Gateway ana makinesinde başlatılabiliyor.
    - Bu yürütme ortamı için sağlayıcı kimlik doğrulaması mevcut (`claude`, `codex`, `gemini`, `opencode`, `droid` vb.).
    - Seçilen model bu yürütme ortamında mevcut; model kimlikleri yürütme ortamları arasında taşınabilir değildir.
    - İstenen `cwd` mevcut ve erişilebilir; aksi takdirde `cwd` öğesini belirtmeyin ve arka ucun varsayılanını kullanmasına izin verin.
    - İzin modu çalışmayla eşleşiyor. Etkileşimsiz oturumlar yerel izin istemlerine tıklayamaz; bu nedenle yoğun yazma/çalıştırma gerektiren kodlama çalışmaları genellikle kullanıcı etkileşimi olmadan ilerleyebilen bir ACPX izin profiline ihtiyaç duyar.

  </Accordion>
</AccordionGroup>

OpenClaw plugin araçları ve yerleşik OpenClaw araçları varsayılan olarak ACP
yürütme ortamlarına **sunulmaz**. Yalnızca yürütme ortamının bu araçları
doğrudan çağırması gerektiğinde [ACP agent'ları - kurulum](/tr/tools/acp-agents-setup)
bölümündeki açık MCP köprülerini etkinleştirin.

## Desteklenen yürütme ortamı hedefleri

`acpx` arka ucuyla bu kimlikleri `/acp spawn <id>` veya
`sessions_spawn({ runtime: "acp", agentId: "<id>" })` hedefleri olarak kullanın:

| Yürütme ortamı kimliği | Tipik arka uç                                  | Notlar                                                                               |
| ------------ | ---------------------------------------------- | ----------------------------------------------------------------------------------- |
| `claude`     | Claude Code ACP bağdaştırıcısı                 | Ana makinede Claude Code kimlik doğrulaması gerektirir.                              |
| `codex`      | Codex ACP bağdaştırıcısı                       | Yalnızca yerel `/codex` kullanılamadığında veya ACP istendiğinde açıkça seçilen ACP geri dönüş yoludur. |
| `copilot`    | GitHub Copilot ACP bağdaştırıcısı              | Copilot CLI/çalışma zamanı kimlik doğrulaması gerektirir.                            |
| `cursor`     | Cursor CLI ACP (`cursor-agent acp`)            | Yerel yükleme farklı bir ACP giriş noktası sunuyorsa acpx komutunu geçersiz kılın.   |
| `droid`      | Factory Droid CLI                              | Yürütme ortamı ortamında Factory/Droid kimlik doğrulaması veya `FACTORY_API_KEY` gerektirir. |
| `fast-agent` | fast-agent-mcp ACP bağdaştırıcısı              | `uvx` ile talep üzerine getirilir.                                      |
| `gemini`     | Gemini CLI ACP bağdaştırıcısı                  | Gemini CLI kimlik doğrulaması veya API anahtarı kurulumu gerektirir.                 |
| `iflow`      | iFlow CLI                                      | Bağdaştırıcı kullanılabilirliği ve model denetimi, yüklü CLI'a bağlıdır.              |
| `kilocode`   | Kilo Code CLI                                  | Bağdaştırıcı kullanılabilirliği ve model denetimi, yüklü CLI'a bağlıdır.              |
| `kimi`       | Kimi/Moonshot CLI                              | Ana makinede Kimi/Moonshot kimlik doğrulaması gerektirir.                            |
| `kiro`       | Kiro CLI                                       | Bağdaştırıcı kullanılabilirliği ve model denetimi, yüklü CLI'a bağlıdır.              |
| `mux`        | Mux CLI ACP bağdaştırıcısı                     | `npx` ile talep üzerine getirilir.                                      |
| `opencode`   | OpenCode ACP bağdaştırıcısı                    | OpenCode CLI/sağlayıcı kimlik doğrulaması gerektirir.                                |
| `openclaw`   | `openclaw acp` üzerinden OpenClaw Gateway köprüsü | ACP uyumlu bir yürütme ortamının bir OpenClaw Gateway oturumuyla iletişim kurmasını sağlar. |
| `qoder`      | Qoder CLI                                      | Bağdaştırıcı kullanılabilirliği ve model denetimi, yüklü CLI'a bağlıdır.              |
| `qwen`       | Qwen Code / Qwen CLI                           | Ana makinede Qwen uyumlu kimlik doğrulaması gerektirir.                              |
| `trae`       | Trae CLI ACP bağdaştırıcısı                    | Bağdaştırıcı kullanılabilirliği ve model denetimi, yüklü CLI'a bağlıdır.              |

`pi` (pi-acp) de acpx arka ucuna kayıtlıdır ancak yukarıdakilerle
aynı anlamda bir kodlama yürütme ortamı değildir.

Özel acpx agent takma adları acpx'in kendisinde yapılandırılabilir ancak OpenClaw
politikası sevk etmeden önce yine de `acp.allowedAgents` öğesini ve varsa
`agents.entries.*.runtime.acp.agent` eşlemesini denetler.

## Operatör çalıştırma kılavuzu

Sohbetten hızlı `/acp` akışı:

<Steps>
  <Step title="Başlatma">
    `/acp spawn claude --bind here`,
    `/acp spawn gemini --mode persistent --thread auto` veya açıkça belirtilen
    `/acp spawn codex --bind here`.
  </Step>
  <Step title="Çalışma">
    Bağlı konuşmada veya ileti dizisinde devam edin (ya da oturum anahtarını
    açıkça hedefleyin).
  </Step>
  <Step title="Durumu denetleme">
    `/acp status`
  </Step>
  <Step title="Ayarlama">
    `/acp model <provider/model>`, `/acp permissions <profile>`,
    `/acp timeout <seconds>`.
  </Step>
  <Step title="Yönlendirme">
    Bağlamı değiştirmeden: `/acp steer tighten logging and continue`.
  </Step>
  <Step title="Durdurma">
    `/acp cancel` (mevcut tur) veya `/acp close` (oturum + bağlamalar).
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Yaşam döngüsü ayrıntıları">
    - Başlatma, bir ACP çalışma zamanı oturumu oluşturur veya sürdürür, ACP meta verilerini OpenClaw oturum deposuna kaydeder ve çalıştırma üst öğe tarafından yönetiliyorsa bir arka plan görevi oluşturabilir.
    - Üst öğe tarafından yönetilen ACP oturumları, çalışma zamanı oturumu kalıcı olsa bile arka plan işi olarak değerlendirilir; tamamlanma ve yüzeyler arası teslimat, normal bir kullanıcıya yönelik sohbet oturumu gibi davranmak yerine üst görev bildiricisi üzerinden gerçekleştirilir.
    - Görev bakımı, sonlandırılmış veya sahipsiz, üst öğe tarafından yönetilen tek seferlik ACP oturumlarını kapatır. Etkin bir konuşma bağlaması bulunduğu sürece kalıcı ACP oturumları korunur; etkin bağlaması olmayan eski kalıcı oturumlar, sahip görev tamamlandıktan veya görev kaydı silindikten sonra sessizce sürdürülememeleri için kapatılır.
    - Bağlanmış takip mesajları, bağlama kapatılana, odaktan çıkarılana, sıfırlanana veya süresi dolana kadar doğrudan ACP oturumuna gider.
    - Gateway komutları yerel kalır. `/acp ...`, `/status` ve `/unfocus`, bağlı bir ACP yürütücüsüne hiçbir zaman normal istem metni olarak gönderilmez.
    - `cancel`, arka uç iptali desteklediğinde etkin turu durdurur; bağlamayı veya oturum meta verilerini silmez.
    - `close`, OpenClaw açısından ACP oturumunu sonlandırır ve bağlamayı kaldırır. Bir yürütücü, sürdürmeyi destekliyorsa kendi yukarı akış geçmişini tutmaya devam edebilir.
    - acpx plugin'i, `close` sonrasında OpenClaw tarafından yönetilen sarmalayıcı ve bağdaştırıcı işlem ağaçlarını temizler ve Gateway başlatılırken sahipsiz kalmış OpenClaw tarafından yönetilen ACPX süreçlerini sonlandırır.
    - Boştaki çalışma zamanı işçileri, yerleşik boşta kalma süresinden sonra temizlenebilir; depolanan oturum meta verileri `/acp sessions` için kullanılabilir durumda kalır.

  </Accordion>
  <Accordion title="Yerel Codex yönlendirme kuralları">
    Etkinleştirildiğinde **yerel Codex plugin'ine** yönlendirilmesi gereken
    doğal dil tetikleyicileri:

    - "Bu Discord kanalını Codex'e bağla."
    - "Bu sohbeti `<id>` Codex iş parçacığına bağla."
    - "Codex iş parçacıklarını göster, ardından bunu bağla."

    Yerel Codex konuşma bağlaması, varsayılan sohbet denetimi yoludur.
    OpenClaw dinamik araçları OpenClaw üzerinden yürütülmeye devam ederken kabuk/apply-patch
    gibi Codex'e özgü araçlar Codex içinde yürütülür. Codex'e özgü
    araç olayları için OpenClaw, plugin kancalarının
    `before_tool_call` öğesini engelleyebilmesi, `after_tool_call` öğesini gözlemleyebilmesi ve Codex
    `PermissionRequest` olaylarını OpenClaw onayları üzerinden yönlendirebilmesi amacıyla her tur için yerel bir kanca aktarımı ekler. Codex `Stop` kancaları,
    plugin'lerin Codex yanıtını tamamlamadan önce
    bir model geçişi daha isteyebildiği OpenClaw `before_agent_finalize` öğesine aktarılır. Aktarım bilinçli olarak
    ihtiyatlı kalır: Codex'e özgü araç bağımsız değişkenlerini değiştirmez
    veya Codex iş parçacığı kayıtlarını yeniden yazmaz. ACP çalışma zamanı/oturum
    modelini istediğinizde yalnızca açık ACP kullanın. Gömülü Codex destek sınırı
    [Codex yürütücüsü v1 destek sözleşmesinde](/tr/plugins/codex-harness-runtime#v1-support-contract)
    belgelenmiştir.

  </Accordion>
  <Accordion title="Model / sağlayıcı / çalışma zamanı seçimi kısa başvuru tablosu">
    - eski Codex model başvuruları - doctor tarafından onarılan eski Codex OAuth/abonelik model rotası.
    - `openai/*` - OpenAI ajan turları için yerel Codex app-server gömülü çalışma zamanı.
    - `/codex ...` - yerel Codex konuşma denetimi.
    - `/acp ...` veya `runtime: "acp"` - açık ACP/acpx denetimi.

  </Accordion>
  <Accordion title="ACP yönlendirmesi için doğal dil tetikleyicileri">
    ACP çalışma zamanına yönlendirilmesi gereken tetikleyiciler:

    - "Bunu tek seferlik bir Claude Code ACP oturumu olarak çalıştır ve sonucu özetle."
    - "Bu görev için bir iş parçacığında Gemini CLI kullan, ardından takipleri aynı iş parçacığında sürdür."
    - "Codex'i ACP üzerinden bir arka plan iş parçacığında çalıştır."

    OpenClaw, `runtime: "acp"` öğesini seçer, `agentId` yürütücüsünü çözümler, desteklendiğinde
    mevcut konuşmaya veya iş parçacığına bağlanır ve kapatılana/süresi dolana kadar takipleri
    bu oturuma yönlendirir. Codex bu yolu yalnızca
    ACP/acpx açıkça belirtildiğinde veya istenen işlem için yerel Codex plugin'i
    kullanılamadığında izler.

    `sessions_spawn` için `runtime: "acp"`, yalnızca ACP
    etkinleştirildiğinde, istekte bulunan korumalı alanda olmadığında ve bir ACP çalışma zamanı arka ucu
    yüklendiğinde sunulur. `acp.dispatch.enabled=false`, otomatik ACP iş parçacığı gönderimini duraklatır
    ancak açık `sessions_spawn({ runtime: "acp" })` çağrılarını gizlemez veya
    engellemez. `codex`, `claude`, `droid`,
    `gemini` veya `opencode` gibi ACP yürütücü kimliklerini hedefler. Bu girdi açıkça
    `agents.entries.*.runtime.type="acp"` ile yapılandırılmadığı sürece `agents_list` içinden normal bir OpenClaw yapılandırma ajanı kimliği
    geçirmeyin; bunun yerine varsayılan alt ajan
    çalışma zamanını kullanın. Bir OpenClaw ajanı
    `runtime.type="acp"` ile yapılandırıldığında OpenClaw, temel
    yürütücü kimliği olarak `runtime.acp.agent` kullanır.

  </Accordion>
</AccordionGroup>

## ACP ile alt ajanların karşılaştırması

Harici bir yürütücü çalışma zamanı istediğinizde ACP kullanın. `codex` plugin'i
etkinleştirildiğinde Codex konuşma bağlaması/denetimi için **yerel Codex
app-server** kullanın. OpenClaw'a özgü devredilmiş çalıştırmalar istediğinizde **alt ajanları** kullanın.

| Alan          | ACP oturumu                           | Alt ajan çalıştırması                      |
| ------------- | ------------------------------------- | ---------------------------------- |
| Çalışma zamanı       | ACP arka uç plugin'i (örneğin acpx) | OpenClaw yerel alt ajan çalışma zamanı  |
| Oturum anahtarı   | `agent:<agentId>:acp:<uuid>`          | `agent:<agentId>:subagent:<uuid>`  |
| Ana komutlar | `/acp ...`                            | `/subagents ...`                   |
| Başlatma aracı    | `runtime:"acp"` ile `sessions_spawn` | `sessions_spawn` (varsayılan çalışma zamanı) |

Ayrıca bkz. [Alt ajanlar](/tr/tools/subagents).

## ACP, Claude Code'u nasıl çalıştırır?

ACP üzerinden Claude Code için yığın şöyledir:

1. OpenClaw ACP oturum denetim düzlemi.
2. Resmî `@openclaw/acpx` çalışma zamanı plugin'i.
3. Claude ACP bağdaştırıcısı.
4. Claude tarafındaki çalışma zamanı/oturum mekanizması.

ACP Claude; ACP denetimleri, oturum sürdürme,
arka plan görevi izleme ve isteğe bağlı konuşma/iş parçacığı bağlaması içeren bir **yürütücü oturumudur**.

CLI arka uçları, ayrı ve yalnızca metin kullanan yerel geri dönüş çalışma zamanlarıdır -
bkz. [CLI Arka Uçları](/tr/gateway/cli-backends).

Operatörler için pratik kural şöyledir:

- **`/acp spawn`, bağlanabilir oturumlar, çalışma zamanı denetimleri veya kalıcı yürütücü çalışması mı istiyorsunuz?** ACP kullanın.
- **Ham CLI üzerinden basit yerel metin geri dönüşü mü istiyorsunuz?** CLI arka uçlarını kullanın.

## Bağlı oturumlar

### Zihinsel model

- **Sohbet yüzeyi** - kişilerin konuşmayı sürdürdüğü yer (Discord kanalı, Telegram konusu, iMessage sohbeti).
- **ACP oturumu** - OpenClaw'ın yönlendirdiği kalıcı Codex/Claude/Gemini çalışma zamanı durumu.
- **Alt iş parçacığı/konu** - yalnızca `--thread ...` tarafından oluşturulan isteğe bağlı ek mesajlaşma yüzeyi.
- **Çalışma zamanı çalışma alanı** - yürütücünün çalıştığı dosya sistemi konumu (`cwd`, depo kullanıma alma dizini, arka uç çalışma alanı). Sohbet yüzeyinden bağımsızdır.

### Mevcut konuşma bağlamaları

`/acp spawn <harness> --bind here`, mevcut konuşmayı
başlatılan ACP oturumuna sabitler; alt iş parçacığı yoktur, aynı sohbet yüzeyi kullanılır. OpenClaw;
taşıma, kimlik doğrulama, güvenlik ve teslimatı yönetmeye devam eder. Bu
konuşmadaki takip mesajları aynı oturuma yönlendirilir; `/new` ve `/reset` oturumu
yerinde sıfırlar; `/acp close` bağlamayı kaldırır.

Örnekler:

```text
/codex bind                                              # yerel Codex bağlaması, gelecekteki mesajları buraya yönlendir
/codex model gpt-5.4                                     # bağlı yerel Codex iş parçacığını ayarla
/codex stop                                              # etkin yerel Codex turunu denetle
/acp spawn codex --bind here                             # Codex için açık ACP geri dönüşü
/acp spawn codex --thread auto                           # bir alt iş parçacığı/konu oluşturabilir ve oraya bağlanabilir
/acp spawn codex --bind here --cwd /workspace/repo       # aynı sohbet bağlaması, Codex /workspace/repo içinde çalışır
```

<AccordionGroup>
  <Accordion title="Bağlama kuralları ve karşılıklı dışlama">
    - `--bind here` ve `--thread ...` birbirini dışlar.
    - `--bind here` yalnızca mevcut konuşma bağlama desteğini bildiren kanallarda çalışır; aksi durumda OpenClaw açık bir desteklenmiyor mesajı döndürür. Bağlamalar, gateway yeniden başlatmaları boyunca kalıcıdır.
    - Discord'da `spawnSessions`, `--bind here` için değil, `--thread auto|here` için alt iş parçacığı oluşturulmasını denetler.
    - `--cwd` olmadan farklı bir ACP ajanı başlatırsanız OpenClaw varsayılan olarak **hedef ajanın** çalışma alanını devralır. Eksik devralınan yollar (`ENOENT`/`ENOTDIR`) arka uç varsayılanına geri döner; diğer erişim hataları (ör. `EACCES`) başlatma hataları olarak gösterilir.
    - Gateway yönetim komutları bağlı konuşmalarda yerel kalır; normal takip metni bağlı ACP oturumuna yönlendirilse bile `/acp ...` komutları OpenClaw tarafından işlenir; komut işleme söz konusu yüzey için etkinleştirildiğinde `/status` ve `/unfocus` da her zaman yerel kalır.

  </Accordion>
  <Accordion title="İş parçacığına bağlı oturumlar">
    Bir kanal bağdaştırıcısı için iş parçacığı bağlamaları etkinleştirildiğinde:

    - OpenClaw, bir iş parçacığını hedef ACP oturumuna bağlar.
    - Bu iş parçacığındaki takip mesajları bağlı ACP oturumuna yönlendirilir.
    - ACP çıktısı aynı iş parçacığına geri teslim edilir.
    - Odaktan çıkarma/kapatma/arşivleme/boşta kalma zaman aşımı veya azami yaş süresinin dolması bağlamayı kaldırır.
    - `/acp close`, `/acp cancel`, `/acp status`, `/status` ve `/unfocus`, ACP yürütücüsüne gönderilen istemler değil, Gateway komutlarıdır.

    İş parçacığına bağlı ACP için gerekli özellik bayrakları:

    - `acp.enabled=true`
    - `acp.dispatch.enabled` varsayılan olarak açıktır (otomatik ACP iş parçacığı gönderimini duraklatmak için `false` ayarlayın; açık `sessions_spawn({ runtime: "acp" })` çağrıları çalışmaya devam eder).
    - Kanal bağdaştırıcısı iş parçacığı oturum başlatmaları etkin (varsayılan: `true`):
      - Discord/Telegram: `session.threadBindings.spawnSessions=true`

    İş parçacığı bağlama desteği bağdaştırıcıya özeldir. Etkin kanal bağdaştırıcısı
    iş parçacığı bağlamalarını desteklemiyorsa OpenClaw açık bir
    desteklenmiyor/kullanılamıyor mesajı döndürür.

  </Accordion>
  <Accordion title="İş parçacığını destekleyen kanallar">
    - Oturum/iş parçacığı bağlama yeteneği sunan tüm kanal bağdaştırıcıları.
    - Mevcut yerleşik destek: **Discord** iş parçacıkları/kanalları, **Telegram** konuları (gruplarda/süper gruplarda forum konuları ve DM konuları).
    - Plugin kanalları aynı bağlama arayüzü üzerinden destek ekleyebilir.

  </Accordion>
</AccordionGroup>

## Kalıcı kanal bağlamaları

Geçici olmayan iş akışları için üst düzey
`bindings[]` girdilerinde kalıcı ACP bağlamalarını yapılandırın.

### Bağlama modeli

<ParamField path="bindings[].type" type='"acp"'>
  Kalıcı bir ACP konuşma bağlamasını işaretler.
</ParamField>
<ParamField path="bindings[].match" type="object">
  Hedef konuşmayı tanımlar. Kanal başına biçimler:

- **Discord kanalı/iş parçacığı:** `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
- **Slack kanalı/DM:** `match.channel="slack"` + `match.peer.id="<channelId|channel:<channelId>|#<channelId>|userId|user:<userId>|slack:<userId>|<@userId>>"`. Kararlı Slack kimliklerini tercih edin; kanal bağlamaları, o kanalın iş parçacıkları içindeki yanıtlarla da eşleşir.
- **Telegram forum konusu:** `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
- **WhatsApp DM/grubu:** `match.channel="whatsapp"` + `match.peer.id="<E.164|group JID>"`. Doğrudan sohbetler için `+15555550123` gibi E.164 numaralarını, gruplar için `120363424282127706@g.us` gibi WhatsApp grup JID'lerini kullanın.
- **iMessage DM/grubu:** `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`. Kararlı grup bağlamaları için `chat_id:*` tercih edin.

</ParamField>
<ParamField path="bindings[].agentId" type="string">
  Sahip olan OpenClaw aracısının kimliği.
</ParamField>
<ParamField path="bindings[].acp.mode" type='"persistent" | "oneshot"'>
  İsteğe bağlı ACP geçersiz kılması.
</ParamField>
<ParamField path="bindings[].acp.label" type="string">
  İsteğe bağlı, operatöre yönelik etiket.
</ParamField>
<ParamField path="bindings[].acp.cwd" type="string">
  İsteğe bağlı çalışma zamanı çalışma dizini.
</ParamField>
<ParamField path="bindings[].acp.backend" type="string">
  İsteğe bağlı arka uç geçersiz kılması.
</ParamField>

### Aracı başına çalışma zamanı varsayılanları

ACP varsayılanlarını aracı başına bir kez tanımlamak için `agents.entries.*.runtime` kullanın:

- `agents.entries.*.runtime.type="acp"`
- `agents.entries.*.runtime.acp.agent` (çalıştırma sistemi kimliği, ör. `codex` veya `claude`)
- `agents.entries.*.runtime.acp.backend`
- `agents.entries.*.runtime.acp.mode`
- `agents.entries.*.runtime.acp.cwd`

**ACP'ye bağlı oturumlar için geçersiz kılma önceliği:**

1. `bindings[].acp.*`
2. `agents.entries.*.runtime.acp.*`
3. Genel ACP varsayılanları (ör. `acp.backend`)

### Örnek

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
      {
        id: "claude",
        runtime: {
          type: "acp",
          acp: { agent: "claude", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
    {
      type: "acp",
      agentId: "claude",
      match: {
        channel: "telegram",
        accountId: "default",
        peer: { kind: "group", id: "-1001234567890:topic:42" },
      },
      acp: { cwd: "/workspace/repo-b" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "discord", accountId: "default" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "telegram", accountId: "default" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": { requireMention: false },
          },
        },
      },
    },
    telegram: {
      groups: {
        "-1001234567890": {
          topics: { "42": { requireMention: false } },
        },
      },
    },
  },
}
```

### Davranış

- OpenClaw, yapılandırılmış ACP oturumunun kanala özgü kabul işleminden sonra ve kullanılmadan önce mevcut olmasını sağlar.
- Bu kanal, konu veya sohbetteki mesajlar yapılandırılmış ACP oturumuna yönlendirilir.
- Yapılandırılmış ACP bağlamaları kendi oturum rotalarının sahibidir. Kanal yayını dallanması, eşleşen bir bağlama için yapılandırılmış ACP oturumunun yerini almaz.
- Bağlı konuşmalarda `/new` ve `/reset`, aynı ACP oturum anahtarını yerinde sıfırlar.
- Geçici çalışma zamanı bağlamaları (örneğin iş parçacığına odaklanma akışlarının oluşturdukları) mevcut oldukları yerde uygulanmaya devam eder.
- Açık bir `cwd` olmadan aracılar arası ACP başlatmalarında OpenClaw, hedef aracı çalışma alanını aracı yapılandırmasından devralır.
- Eksik devralınan çalışma alanı yolları, arka ucun varsayılan cwd değerine geri döner; mevcut yollardaki erişim hataları ise başlatma hataları olarak gösterilir.

## ACP oturumlarını başlatma

Bir ACP oturumunu başlatmanın iki yolu vardır:

<Tabs>
  <Tab title="sessions_spawn üzerinden">
    Bir aracı turundan veya araç çağrısından ACP oturumu başlatmak için
    `runtime: "acp"` kullanın.

    ```json
    {
      "task": "Depoyu aç ve başarısız olan testleri özetle",
      "runtime": "acp",
      "agentId": "codex",
      "thread": true,
      "mode": "session"
    }
    ```

    <Note>
    `runtime` varsayılan olarak `subagent` değerini kullanır; bu nedenle
    ACP oturumları için `runtime: "acp"` değerini açıkça ayarlayın. `agentId` belirtilmezse
    OpenClaw, yapılandırılmış olduğunda `acp.defaultAgent` kullanır. Kalıcı bir
    bağlı konuşmanın sürdürülmesi için `mode: "session"`, `thread: true` gerektirir.
    </Note>

  </Tab>
  <Tab title="/acp komutu üzerinden">
    Sohbetten açık operatör denetimi için `/acp spawn` kullanın.

    ```text
    /acp spawn codex --mode persistent --thread auto
    /acp spawn codex --mode oneshot --thread off
    /acp spawn codex --bind here
    /acp spawn codex --thread here
    ```

    Temel bayraklar:

    - `--mode persistent|oneshot`
    - `--bind here|off`
    - `--thread auto|here|off`
    - `--cwd <absolute-path>`
    - `--label <name>`

    Bkz. [Eğik çizgi komutları](/tr/tools/slash-commands).

  </Tab>
</Tabs>

### `sessions_spawn` parametreleri

<ParamField path="task" type="string" required>
  ACP oturumuna gönderilen ilk istem.
</ParamField>
<ParamField path="runtime" type='"acp"' required>
  ACP oturumları için `"acp"` olmalıdır.
</ParamField>
<ParamField path="agentId" type="string">
  ACP hedef çalıştırma sistemi kimliği. Ayarlanmışsa `acp.defaultAgent` değerine geri döner.
</ParamField>
<ParamField path="thread" type="boolean" default="false">
  Desteklendiği yerlerde iş parçacığı bağlama akışını talep eder.
</ParamField>
<ParamField path="mode" type='"run" | "session"' default="run">
  `"run"` tek seferliktir; `"session"` kalıcıdır. `thread: true` ayarlanmışsa ve
  `mode` belirtilmemişse OpenClaw, çalışma zamanı yoluna göre varsayılan olarak
  kalıcı davranışı kullanabilir. `mode: "session"`, `thread: true` gerektirir.
</ParamField>
<ParamField path="cwd" type="string">
  İstenen çalışma zamanı çalışma dizini (arka uç/çalışma zamanı politikası tarafından doğrulanır).
  Belirtilmezse ACP başlatma işlemi, yapılandırılmış olduğunda hedef aracı çalışma alanını devralır;
  eksik devralınan yollar arka uç varsayılanlarına geri dönerken gerçek erişim
  hataları döndürülür.
</ParamField>
<ParamField path="label" type="string">
  Oturum/başlık metninde kullanılan, operatöre yönelik etiket.
</ParamField>
<ParamField path="resumeSessionId" type="string">
  Yeni bir ACP oturumu oluşturmak yerine mevcut bir ACP oturumunu sürdürür. Aracı,
  konuşma geçmişini `session/load` üzerinden yeniden oynatır.
  `runtime: "acp"` gerektirir.
</ParamField>
<ParamField path="streamTo" type='"parent"'>
  `"parent"`, ilk ACP çalıştırma ilerleme özetlerini sistem olayları olarak istekte bulunan
  oturuma aktarır. OpenClaw, tam aktarma geçmişini alt aracının SQLite durumuna
  kaydeder ve alt oturumla birlikte kaldırır. Ana oturum ilerleme akışları, aksi
  `streaming.progress.commentary=false` ile belirtilmedikçe varsayılan olarak asistan açıklamalarını ve ACP durum
  ilerlemesini gösterir. Ayrıca hiçbir akış modu yapılandırılmadığında Discord,
  ana oturum önizlemelerinde varsayılan olarak ilerleme modunu kullanır. Durum
  ilerlemesi yine de `acp.stream.tagVisibility` ayarına uyar; dolayısıyla `plan` gibi etiketler
  açıkça etkinleştirilmedikçe gizli kalır.
</ParamField>

ACP `sessions_spawn` çalıştırmaları, varsayılan alt tur sınırı için
`agents.defaults.subagents.runTimeoutSeconds` kullanır. Araç, çağrı başına zaman aşımı geçersiz
kılmalarını kabul etmez (`runTimeoutSeconds`/`timeoutSeconds`, varsayılanı
yapılandırma hatasıyla reddedilir).

<ParamField path="model" type="string">
  ACP alt oturumu için açık model geçersiz kılması. Codex ACP başlatmaları,
  `openai/gpt-5.4` gibi OpenAI referanslarını `session/new` öncesinde Codex ACP başlangıç
  yapılandırmasına normalleştirir; `openai/gpt-5.4/high` gibi eğik çizgili biçimler de
  Codex ACP muhakeme eforunu ayarlar. Belirtilmediğinde `sessions_spawn({ runtime: "acp" })`,
  yapılandırılmışsa mevcut alt aracı model varsayılanlarını (`agents.defaults.subagents.model` veya
  `agents.entries.*.subagents.model`) kullanır; aksi takdirde ACP çalıştırma sisteminin kendi varsayılan
  modelini kullanmasına izin verir. Diğer çalıştırma sistemleri ACP
  `models` özelliğini ilan etmeli ve `session/set_model` desteği sunmalıdır; aksi takdirde OpenClaw/acpx,
  sessizce hedef aracının varsayılanına geri dönmek yerine açıkça başarısız olur.
</ParamField>
<ParamField path="thinking" type="string">
  Açık düşünme/muhakeme eforu. Codex ACP için `minimal` düşük
  efora eşlenir; `low`/`medium`/`high`/`xhigh` doğrudan eşlenir ve `off`,
  muhakeme eforu başlangıç geçersiz kılmasını atlar. Belirtilmediğinde ACP başlatmaları,
  seçili model için mevcut alt aracı düşünme varsayılanlarını ve model başına
  `agents.defaults.models["provider/model"].params.thinking` değerini kullanır.
</ParamField>

## Başlatma bağlama ve iş parçacığı modları

<Tabs>
  <Tab title="--bind here|off">
    | Mod   | Davranış                                                               |
    | ------ | ----------------------------------------------------------------------- |
    | `here` | Etkin konuşmayı yerinde bağlar; etkin konuşma yoksa başarısız olur. |
    | `off`  | Etkin konuşma bağlaması oluşturmaz.                          |

    Notlar:

    - `--bind here`, "bu kanalı veya sohbeti Codex destekli yap" için en basit operatör yoludur.
    - `--bind here`, alt iş parçacığı oluşturmaz.
    - `--bind here`, yalnızca etkin konuşma bağlama desteği sunan kanallarda kullanılabilir.
    - `--bind` ve `--thread`, aynı `/acp spawn` çağrısında birlikte kullanılamaz.

  </Tab>
  <Tab title="--thread auto|here|off">
    | Mod   | Davranış                                                                                            |
    | ------ | ------------------------------------------------------------------------------------------------- |
    | `auto` | Etkin bir iş parçacığındayken bu iş parçacığını bağlar. İş parçacığı dışında, destekleniyorsa alt iş parçacığı oluşturur/bağlar. |
    | `here` | Etkin bir iş parçacığı gerektirir; iş parçacığında değilse başarısız olur.                                                  |
    | `off`  | Bağlama yoktur. Oturum bağlantısız başlar.                                                                 |

    Notlar:

    - İş parçacığı bağlamayı desteklemeyen yüzeylerde varsayılan davranış fiilen `off` şeklindedir.
    - İş parçacığına bağlı başlatma, kanal politikası desteği gerektirir:
      - Discord/Telegram: `session.threadBindings.spawnSessions=true`
    - Alt iş parçacığı oluşturmadan etkin konuşmayı sabitlemek istediğinizde `--bind here` kullanın.

  </Tab>
</Tabs>

## Teslim modeli

ACP oturumları etkileşimli çalışma alanları veya ana oturumun sahip olduğu arka plan
çalışmaları olabilir. Teslim yolu bu yapıya bağlıdır.

<AccordionGroup>
  <Accordion title="Etkileşimli ACP oturumları">
    Etkileşimli oturumlar, görünür bir sohbet yüzeyinde konuşmayı sürdürmek için tasarlanmıştır:

    - `/acp spawn ... --bind here`, etkin konuşmayı ACP oturumuna bağlar.
    - `/acp spawn ... --thread ...`, bir kanal iş parçacığını/konusunu ACP oturumuna bağlar.
    - Kalıcı yapılandırılmış `bindings[].type="acp"`, eşleşen konuşmaları aynı ACP oturumuna yönlendirir.

    Bağlı konuşmadaki takip mesajları doğrudan ACP oturumuna yönlendirilir
    ve ACP çıktısı aynı kanala/iş parçacığına/konuya geri
    teslim edilir.

    OpenClaw'ın çalıştırma sistemine gönderdikleri:

    - Normal bağlı takipler, istem metni olarak ve yalnızca harness/backend bunları desteklediğinde eklerle birlikte gönderilir.
    - `/acp` yönetim komutları ve yerel Gateway komutları, ACP gönderiminden önce yakalanır.
    - Çalışma zamanı tarafından oluşturulan tamamlanma olayları hedef başına somutlaştırılır. OpenClaw ajanları, OpenClaw'ın dahili çalışma zamanı bağlam zarfını alır; harici ACP harness'leri ise alt öğenin sonucu ve talimatı içeren düz bir istem alır. Ham `<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>` zarfı hiçbir zaman harici harness'lere gönderilmemeli veya ACP kullanıcı transkript metni olarak kalıcılaştırılmamalıdır.
    - ACP transkript girdileri, kullanıcıya görünür tetikleme metnini veya düz tamamlanma istemini kullanır. Dahili olay meta verileri mümkün olduğunda OpenClaw içinde yapılandırılmış olarak kalır ve kullanıcı tarafından yazılmış sohbet içeriği olarak değerlendirilmez.

  </Accordion>
  <Accordion title="Üst öğenin sahip olduğu tek seferlik ACP oturumları">
    Başka bir ajan çalıştırması tarafından oluşturulan tek seferlik ACP oturumları,
    alt ajanlara benzer arka plan alt öğeleridir:

    - Üst öğe, `sessions_spawn({ runtime: "acp", mode: "run" })` ile iş yapılmasını ister.
    - Alt öğe kendi ACP harness oturumunda çalışır.
    - Alt öğe dönüşleri, yerel alt ajan oluşturmalarında kullanılan aynı arka plan şeridinde çalışır; böylece yavaş bir ACP harness'i, ilgisiz ana oturum çalışmalarını engellemez.
    - Tamamlanma, görev tamamlama duyuru yolu üzerinden geri bildirilir. OpenClaw, harici bir harness'e göndermeden önce dahili tamamlanma meta verilerini düz bir ACP istemine dönüştürür; böylece harness'ler yalnızca OpenClaw'a özgü çalışma zamanı bağlam işaretçilerini görmez.
    - Kullanıcıya yönelik bir yanıt yararlı olduğunda üst öğe, alt öğenin sonucunu normal asistan üslubuyla yeniden yazar.

    Bu yolu, üst öğe ile alt öğe arasında eşler arası sohbet olarak
    **değerlendirmeyin**. Alt öğenin üst öğeye geri dönen bir tamamlanma kanalı zaten vardır.

  </Accordion>
  <Accordion title="sessions_send ve A2A teslimatı">
    `sessions_send`, oluşturma işleminden sonra başka bir oturumu hedefleyebilir. Normal eş
    oturumları için OpenClaw, iletiyi ekledikten sonra ajandan ajana (A2A)
    takip yolunu kullanır:

    - Hedef oturumun yanıtını bekleyin.
    - İsteğe bağlı olarak, istekte bulunan ile hedefin sınırlı sayıda takip dönüşü alışverişi yapmasına izin verin.
    - Hedeften bir duyuru iletisi oluşturmasını isteyin.
    - Bu duyuruyu görünür kanala veya ileti dizisine teslim edin.

    Bu A2A yolu, gönderenin görünür bir takibe ihtiyaç duyduğu eş gönderimleri
    için bir geri dönüş yoludur. İlgisiz bir oturumun, örneğin geniş
    `tools.sessions.visibility` ayarları altında bir ACP hedefini görebildiği ve hedefe
    ileti gönderebildiği durumlarda etkin kalır.

    OpenClaw, A2A takibini yalnızca istekte bulunan kendi üst öğesinin sahip
    olduğu tek seferlik ACP alt öğesinin üst öğesiyse atlar. Bu durumda, görev
    tamamlamanın üzerine A2A çalıştırmak üst öğeyi alt öğenin sonucuyla uyandırabilir,
    üst öğenin yanıtını tekrar alt öğeye iletebilir ve bir üst öğe/alt öğe yankı
    döngüsü oluşturabilir. `sessions_send` sonucu, tamamlanma yolu sonuçtan zaten
    sorumlu olduğu için bu sahip olunan alt öğe durumunda `delivery.status="skipped"`
    bildirir.

  </Accordion>
  <Accordion title="Mevcut bir oturumu sürdürme">
    Yeni bir oturum başlatmak yerine önceki bir ACP oturumunu sürdürmek için
    `resumeSessionId` kullanın. Ajan, konuşma geçmişini `session/load`
    üzerinden yeniden oynatır; böylece önceki tüm bağlamla kaldığı yerden devam eder.

    ```json
    {
      "task": "Kaldığımız yerden devam et - kalan test hatalarını düzelt",
      "runtime": "acp",
      "agentId": "codex",
      "resumeSessionId": "<previous-session-id>"
    }
    ```

    Yaygın kullanım durumları:

    - Bir Codex oturumunu dizüstü bilgisayarınızdan telefonunuza devredin; ajanınıza kaldığınız yerden devam etmesini söyleyin.
    - CLI'da etkileşimli olarak başlattığınız bir kodlama oturumunu artık ajanınız üzerinden gözetimsiz olarak sürdürün.
    - Gateway yeniden başlatması veya boşta kalma zaman aşımı nedeniyle kesintiye uğrayan çalışmayı sürdürün.

    Notlar:

    - `resumeSessionId` yalnızca `runtime: "acp"` olduğunda geçerlidir; varsayılan alt ajan çalışma zamanı, yalnızca ACP'ye özgü bu alanı yok sayar.
    - `streamTo` yalnızca `runtime: "acp"` olduğunda geçerlidir; varsayılan alt ajan çalışma zamanı, yalnızca ACP'ye özgü bu alanı yok sayar.
    - `resumeSessionId`, bir OpenClaw kanal oturumu anahtarı değil, ana bilgisayara yerel bir ACP/harness sürdürme kimliğidir; OpenClaw gönderimden önce ACP oluşturma politikasını ve hedef ajan politikasını yine denetlerken üst akış kimliğini yüklemeye ilişkin yetkilendirme ACP backend'ine veya harness'e aittir.
    - `resumeSessionId`, üst akış ACP konuşma geçmişini geri yükler; `thread` ve `mode`, oluşturduğunuz yeni OpenClaw oturumuna normal şekilde uygulanmaya devam eder, dolayısıyla `mode: "session"` yine `thread: true` gerektirir.
    - Hedef ajan `session/load` desteğine sahip olmalıdır (Codex ve Claude Code destekler).
    - Oturum kimliği bulunamazsa oluşturma işlemi açık bir hatayla başarısız olur; yeni bir oturuma sessizce geri dönüş yapılmaz.

  </Accordion>
  <Accordion title="Dağıtım sonrası hızlı doğrulama testi">
    Bir Gateway dağıtımından sonra birim testlerine güvenmek yerine
    canlı bir uçtan uca denetim çalıştırın:

    1. Dağıtılan Gateway sürümünü ve hedef ana bilgisayardaki commit'i doğrulayın.
    2. Canlı bir ajana geçici bir ACPX köprü oturumu açın.
    3. Bu ajandan `runtime: "acp"`, `agentId: "codex"`, `mode: "run"` ve `Reply with exactly LIVE-ACP-SPAWN-OK` göreviyle `sessions_spawn` çağrısı yapmasını isteyin.
    4. `accepted=yes`, gerçek bir `childSessionKey` ve doğrulayıcı hatası bulunmadığını doğrulayın.
    5. Geçici köprü oturumunu temizleyin.

    Geçidi `mode: "run"` üzerinde tutun ve `streamTo: "parent"` öğesini atlayın;
    ileti dizisine bağlı `mode: "session"` ve akış aktarma yolları, ayrı ve daha kapsamlı
    entegrasyon geçişleridir.

  </Accordion>
</AccordionGroup>

## Sandbox uyumluluğu

ACP oturumları şu anda OpenClaw sandbox'ının içinde **değil**, ana bilgisayar
çalışma zamanında çalışır.

<Warning>
**Güvenlik sınırı:**

- Harici harness, kendi CLI izinlerine ve seçilen `cwd` öğesine göre okuyabilir/yazabilir.
- OpenClaw'ın sandbox politikası, ACP harness yürütmesini **kapsamaz**.
- OpenClaw; ACP özellik geçitlerini, izin verilen ajanları, oturum sahipliğini, kanal bağlamalarını ve Gateway teslimat politikasını uygulamaya devam eder.
- Sandbox tarafından uygulanan OpenClaw'a özgü çalışmalar için `runtime: "subagent"` kullanın.

</Warning>

Mevcut sınırlamalar:

- İstekte bulunan oturum sandbox içindeyse ACP oluşturmaları hem `sessions_spawn({ runtime: "acp" })` hem de `/acp spawn` için engellenir.
- `runtime: "acp"` ile `sessions_spawn`, `sandbox: "require"` desteği sunmaz.

## Oturum hedefi çözümleme

Çoğu `/acp` eylemi isteğe bağlı bir oturum hedefini (`session-key`,
`session-id` veya `session-label`) kabul eder.

**Çözümleme sırası:**

1. Açık hedef bağımsız değişkeni (veya `/acp steer` için `--session`)
   - önce anahtarı dener
   - ardından UUID biçimli oturum kimliğini
   - ardından etiketi
2. Geçerli ileti dizisi bağlaması (bu konuşma/ileti dizisi bir ACP oturumuna bağlıysa).
3. Geçerli istekte bulunan oturumuna geri dönüş.

Hem geçerli konuşma bağlamaları hem de ileti dizisi bağlamaları 2. adıma katılır.

Hiçbir hedef çözümlenemezse OpenClaw açık bir hata döndürür
(`Unable to resolve session target: ...`).

## ACP denetimleri

| Komut                | İşlevi                                                    | Örnek                                                         |
| -------------------- | --------------------------------------------------------- | ------------------------------------------------------------- |
| `/acp spawn`         | ACP oturumu oluşturur; isteğe bağlı geçerli bağlama veya ileti dizisi bağlaması. | `/acp spawn codex --bind here --cwd /repo`                    |
| `/acp cancel`        | Hedef oturumun devam eden dönüşünü iptal eder.             | `/acp cancel agent:codex:acp:<uuid>`                          |
| `/acp steer`         | Çalışan oturuma yönlendirme talimatı gönderir.             | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close`         | Oturumu kapatır ve ileti dizisi hedeflerinin bağını kaldırır. | `/acp close`                                                  |
| `/acp status`        | Backend'i, modu, durumu, çalışma zamanı seçeneklerini ve yetenekleri gösterir. | `/acp status`                                                 |
| `/acp set-mode`      | Hedef oturumun çalışma zamanı modunu ayarlar.              | `/acp set-mode plan`                                          |
| `/acp set`           | Genel çalışma zamanı yapılandırma seçeneği yazma işlemi.   | `/acp set model openai/gpt-5.4`                               |
| `/acp cwd`           | Çalışma zamanı çalışma dizini geçersiz kılmasını ayarlar.  | `/acp cwd /Users/user/Projects/repo`                          |
| `/acp permissions`   | Onay politikası profilini ayarlar.                         | `/acp permissions strict`                                     |
| `/acp timeout`       | Çalışma zamanı zaman aşımını (saniye) ayarlar.             | `/acp timeout 120`                                            |
| `/acp model`         | Çalışma zamanı model geçersiz kılmasını ayarlar.           | `/acp model anthropic/claude-opus-4-6`                        |
| `/acp reset-options` | Oturumun çalışma zamanı seçeneği geçersiz kılmalarını kaldırır. | `/acp reset-options`                                          |
| `/acp sessions`      | Depodaki son ACP oturumlarını listeler.                    | `/acp sessions`                                               |
| `/acp doctor`        | Backend sağlığı, yetenekler ve uygulanabilir düzeltmeler.  | `/acp doctor`                                                 |
| `/acp install`       | Belirlenimci kurulum ve etkinleştirme adımlarını yazdırır. | `/acp install`                                                |

Çalışma zamanı denetimleri (`spawn`, `cancel`, `steer`, `close`, `status`, `set-mode`,
`set`, `cwd`, `permissions`, `timeout`, `model` ve `reset-options`), harici
kanallardan sahip kimliği ve dahili Gateway istemcilerinden `operator.admin`
gerektirir. Yetkilendirilmiş ancak sahip olmayan gönderenler `sessions`,
`doctor`, `install` ve `help` komutlarını kullanmaya devam edebilir. Sahip olmayan gönderenler için `/acp sessions`
yalnızca geçerli bağlı oturumu veya istekte bulunan oturumunu listeler; sahip kimliği ve
`operator.admin` istemcileri tüm son oturumları görür.

`/acp status`, etkin çalışma zamanı seçeneklerinin yanı sıra çalışma zamanı düzeyindeki ve
backend düzeyindeki oturum tanımlayıcılarını gösterir. Bir backend bir yeteneğe sahip olmadığında,
desteklenmeyen denetim hataları açıkça gösterilir. Hedef belirteçlerini kabul eden komutlar
(`session-key`, `session-id` veya `session-label`), özel ajan başına `session.store`
kökleri dâhil olmak üzere bunları Gateway oturum keşfi üzerinden çözümler. `/acp sessions`
bir hedef belirteci kabul etmez.

### Çalışma zamanı seçenekleri eşlemesi

`/acp`, kolaylık komutlarına ve genel bir ayarlayıcıya sahiptir. Eşdeğer işlemler:

| Komut                      | Eşlendiği öğe                              | Notlar                                                                                                                                                                                                      |
| ---------------------------- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/acp model <id>`            | çalışma zamanı yapılandırma anahtarı `model`           | Codex ACP için OpenClaw, `openai/<model>` değerini bağdaştırıcı model kimliğine dönüştürür ve `openai/gpt-5.4/high` gibi eğik çizgili akıl yürütme soneklerini `reasoning_effort` değerine eşler.                                         |
| `/acp set thinking <level>`  | standart seçenek `thinking`          | OpenClaw, mevcut olduğunda arka uç tarafından bildirilen eşdeğeri gönderir; sırasıyla `thinking`, ardından `effort`, `reasoning_effort` veya `thought_level` tercih edilir. Codex ACP için bağdaştırıcı, değerleri `reasoning_effort` değerine eşler. |
| `/acp permissions <profile>` | standart seçenek `permissionProfile` | OpenClaw, mevcut olduğunda `approval_policy`, `permission_profile`, `permissions` veya `permission_mode` gibi arka uç tarafından bildirilen eşdeğeri gönderir.                                                       |
| `/acp timeout <seconds>`     | standart seçenek `timeoutSeconds`    | OpenClaw, mevcut olduğunda `timeout` veya `timeout_seconds` gibi arka uç tarafından bildirilen eşdeğeri gönderir.                                                                                                     |
| `/acp cwd <path>`            | çalışma zamanı cwd geçersiz kılması                 | Doğrudan güncelleme.                                                                                                                                                                                             |
| `/acp set <key> <value>`     | genel                              | `key=cwd`, cwd geçersiz kılma yolunu kullanır.                                                                                                                                                                      |
| `/acp reset-options`         | tüm çalışma zamanı geçersiz kılmalarını temizler         | -                                                                                                                                                                                                          |

## acpx test düzeneği, plugin kurulumu ve izinler

acpx test düzeneği yapılandırması (Claude Code / Codex / Gemini CLI takma adları),
plugin araçları ile OpenClaw araçlarının MCP köprüleri ve ACP izin modları için
bkz. [ACP aracıları - kurulum](/tr/tools/acp-agents-setup).

## Sorun giderme

| Belirti                                                                                   | Olası neden                                                                                                           | Çözüm                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ACP runtime backend is not configured`                                                   | Arka uç plugini eksik, devre dışı veya `plugins.allow` tarafından engellenmiş.                                                       | Arka uç pluginini kurup etkinleştirin, bu izin verilenler listesi ayarlanmışsa `acpx` değerini `plugins.allow` içine ekleyin, ardından `/acp doctor` komutunu çalıştırın.                                                 |
| `ACP is disabled by policy (acp.enabled=false)`                                           | ACP genel olarak devre dışı.                                                                                                 | `acp.enabled=true` değerini ayarlayın.                                                                                                                                                  |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`                         | Normal iş parçacığı mesajlarından otomatik yönlendirme devre dışı.                                                               | Otomatik iş parçacığı yönlendirmesini sürdürmek için `acp.dispatch.enabled=true` değerini ayarlayın; açık `sessions_spawn({ runtime: "acp" })` çağrıları çalışmaya devam eder.                                      |
| `ACP agent "<id>" is not allowed by policy`                                               | Aracı izin verilenler listesinde değil.                                                                                                | İzin verilen bir `agentId` kullanın veya `acp.allowedAgents` değerini güncelleyin.                                                                                                                     |
| `/acp doctor`, başlatmanın hemen ardından arka ucun hazır olmadığını bildiriyor                               | Arka uç plugini eksik, devre dışı, izin/verme engelleme ilkesi tarafından engellenmiş veya yapılandırılmış yürütülebilir dosyası kullanılamıyor.        | Arka uç pluginini kurun/etkinleştirin, `/acp doctor` komutunu yeniden çalıştırın ve sağlıksız kalırsa arka uç kurulum veya ilke hatasını inceleyin.                                           |
| Test düzeneği komutu bulunamadı                                                                 | Bağdaştırıcı CLI'ı kurulu değil, harici plugin eksik veya Codex dışı bir bağdaştırıcının ilk çalıştırmadaki `npx` getirme işlemi başarısız oldu. | `/acp doctor` komutunu çalıştırın, bağdaştırıcıyı Gateway ana makinesine kurun/önceden hazırlayın veya acpx aracı komutunu açıkça yapılandırın.                                                      |
| Test düzeneğinden model bulunamadı hatası                                                          | Model kimliği başka bir sağlayıcı/test düzeneği için geçerli, ancak bu ACP hedefi için geçerli değil.                                                | Bu test düzeneğinin listelediği bir modeli kullanın, modeli test düzeneğinde yapılandırın veya geçersiz kılmayı kaldırın.                                                                            |
| Test düzeneğinden sağlayıcı kimlik doğrulama hatası                                                        | OpenClaw sağlıklı, ancak hedef CLI/sağlayıcı oturum açmamış.                                                     | Gateway ana makinesi ortamında oturum açın veya gerekli sağlayıcı anahtarını sağlayın.                                                                                             |
| `Unable to resolve session target: ...`                                                   | Hatalı anahtar/kimlik/etiket belirteci.                                                                                                | `/acp sessions` komutunu çalıştırın, anahtarı/etiketi tam olarak kopyalayın ve yeniden deneyin.                                                                                                                        |
| `--bind here requires running /acp spawn inside an active ... conversation`               | `--bind here`, etkin ve bağlanabilir bir konuşma olmadan kullanılmış.                                                            | Hedef sohbete/kanala geçip yeniden deneyin veya bağlanmamış oluşturmayı kullanın.                                                                                                         |
| `Conversation bindings are unavailable for <channel>.`                                    | Bağdaştırıcı, mevcut konuşmaya ACP bağlama özelliğine sahip değil.                                                             | Desteklendiği yerlerde `/acp spawn ... --thread ...` kullanın, üst düzey `bindings[]` değerini yapılandırın veya desteklenen bir kanala geçin.                                                     |
| `--thread here requires running /acp spawn inside an active ... thread`                   | `--thread here`, iş parçacığı bağlamı dışında kullanılmış.                                                                         | Hedef iş parçacığına geçin veya `--thread auto`/`off` kullanın.                                                                                                                      |
| `Only <user-id> can rebind this channel/conversation/thread.`                             | Etkin bağlama hedefinin sahibi başka bir kullanıcı.                                                                           | Sahip olarak yeniden bağlayın veya farklı bir konuşma ya da iş parçacığı kullanın.                                                                                                               |
| `Thread bindings are unavailable for <channel>.`                                          | Bağdaştırıcı, iş parçacığı bağlama özelliğine sahip değil.                                                                               | `--thread off` kullanın veya desteklenen bir bağdaştırıcıya/kanala geçin.                                                                                                                 |
| `Sandboxed sessions cannot spawn ACP sessions ...`                                        | ACP çalışma zamanı ana makine tarafındadır; istekte bulunan oturum korumalı alandadır.                                                              | Korumalı alan oturumlarında `runtime="subagent"` kullanın veya ACP oluşturma işlemini korumalı alanda olmayan bir oturumdan çalıştırın.                                                                         |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...`                   | ACP çalışma zamanı için `sandbox="require"` istendi.                                                                         | Gerekli korumalı alan kullanımı için `runtime="subagent"` kullanın veya korumalı alanda olmayan bir oturumdan `sandbox="inherit"` ile ACP kullanın.                                                      |
| `Cannot apply --model ... did not advertise model support`                                | Hedef test düzeneği genel ACP model değiştirme özelliğini sunmuyor.                                                        | ACP `models`/`session/set_model` özelliğini bildiren bir test düzeneği kullanın, Codex ACP model başvurularını kullanın veya kendi başlatma bayrağı varsa modeli doğrudan test düzeneğinde yapılandırın. |
| Bağlı oturum için ACP meta verileri eksik                                                    | Eski/silinmiş ACP oturum meta verileri.                                                                                    | `/acp spawn` ile yeniden oluşturun, ardından iş parçacığını yeniden bağlayın/odaklayın.                                                                                                                    |
| `PermissionPromptUnavailableError: Permission prompt unavailable in non-interactive mode` | `permissionMode`, etkileşimli olmayan ACP oturumunda yazma/yürütme işlemlerini engelliyor.                                                    | `plugins.entries.acpx.config.permissionMode` değerini `approve-all` olarak ayarlayın ve Gateway'i yeniden başlatın. Bkz. [İzin yapılandırması](/tr/tools/acp-agents-setup#permission-configuration). |
| ACP oturumu az çıktıyla erken başarısız oluyor                                                | İzin istemleri `permissionMode`/`nonInteractivePermissions` tarafından engelleniyor.                                        | Gateway günlüklerinde `AcpRuntimeError` değerini kontrol edin. Tam izinler için `permissionMode=approve-all`; işlevselliğin kontrollü azalması için `nonInteractivePermissions=deny` değerini ayarlayın.        |
| ACP oturumu işi tamamladıktan sonra süresiz olarak takılıyor                                     | Test düzeneği işlemi tamamlandı ancak ACP oturumu tamamlandığını bildirmedi.                                                    | OpenClaw'u güncelleyin; güncel acpx temizleme işlemi, kapatma ve Gateway başlatma sırasında OpenClaw'a ait eski sarmalayıcı ve bağdaştırıcı işlemlerini sonlandırır.                                             |
| Test düzeneği `<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>` görüyor                                      | Dahili olay zarfı ACP sınırının dışına sızdı.                                                                | OpenClaw'u güncelleyin ve tamamlama akışını yeniden çalıştırın; harici test düzenekleri yalnızca düz tamamlama istemleri almalıdır.                                                          |

<Note>
`Command blocked by PreToolUse hook: Native hook relay unavailable`, ACP/acpx'e değil,
yerel Codex hook aktarıcısına aittir. Bağlı bir Codex sohbetinde
`/new` veya `/reset` ile yeni bir oturum başlatın; bir kez çalışıp
sonraki yerel araç çağrısında tekrar ortaya çıkarsa `/new` işlemini tekrarlamak yerine
Codex uygulama sunucusunu veya OpenClaw Gateway'i yeniden başlatın. Bkz.
[Codex test düzeneği sorun giderme](/tr/plugins/codex-harness#troubleshooting).
</Note>

## İlgili

- [ACP ajanları - kurulum](/tr/tools/acp-agents-setup)
- [Ajan gönderimi](/tr/tools/agent-send)
- [CLI Arka Uçları](/tr/gateway/cli-backends)
- [Codex test düzeneği](/tr/plugins/codex-harness)
- [Codex test düzeneği çalışma zamanı](/tr/plugins/codex-harness-runtime)
- [Çok ajanlı korumalı alan araçları](/tr/tools/multi-agent-sandbox-tools)
- [`openclaw acp` (köprü modu)](/tr/cli/acp)
- [Alt ajanlar](/tr/tools/subagents)
