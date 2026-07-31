---
read_when:
    - Skills ekleme veya değiştirme
    - Skill geçiş koşullarını, izin listelerini veya yükleme kurallarını değiştirme
    - Skills önceliğini ve anlık görüntü davranışını anlama
sidebarTitle: Skills
summary: Skills, aracınıza araçları nasıl kullanacağını öğretir. Nasıl yüklendiklerini, önceliğin nasıl çalıştığını ve geçit denetimi, izin listeleri ile ortam eklemenin nasıl yapılandırılacağını öğrenin.
title: Skills
x-i18n:
    generated_at: "2026-07-26T23:44:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6925add85652023e3dd2f51f607412fd0bf00581923f76ab2aafd2ca5b8d72be
    source_path: tools/skills.md
    workflow: 16
---

Skills, ajana araçları nasıl ve ne zaman kullanacağını öğreten markdown talimat dosyalarıdır. Her Skill, YAML
frontmatter ve markdown gövdesi içeren bir `SKILL.md` dosyasının bulunduğu bir dizinde
yer alır. OpenClaw, paketle birlikte gelen Skills'leri ve tüm yerel
geçersiz kılmaları yükler; yükleme sırasında ortam, yapılandırma ve
ikili dosyaların varlığına göre filtreler.

<CardGroup cols={2}>
  <Card title="Skills oluşturma" href="/tr/tools/creating-skills" icon="hammer">
    Sıfırdan özel bir Skill oluşturun ve test edin.
  </Card>
  <Card title="Skill Atölyesi" href="/tr/tools/skill-workshop" icon="flask">
    Ajan tarafından taslak hâline getirilen Skill önerilerini inceleyin ve onaylayın.
  </Card>
  <Card title="Skills yapılandırması" href="/tr/tools/skills-config" icon="gear">
    Tam `skills.*` yapılandırma şeması ve ajan izin listeleri.
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="cloud">
    Topluluk Skills'lerine göz atın ve bunları yükleyin.
  </Card>
</CardGroup>

## Yükleme sırası

OpenClaw şu kaynaklardan, **en yüksek öncelik ilk sırada** olacak şekilde yükleme yapar. Aynı
Skill adı birden fazla yerde görünürse en yüksek öncelikli kaynak geçerli olur.

| Öncelik       | Kaynak                  | Yol                                     |
| ------------- | ----------------------- | --------------------------------------- |
| 1 — en yüksek | Çalışma alanı Skills'leri | `<workspace>/skills`                    |
| 2             | Proje ajanı Skills'leri | `<workspace>/.agents/skills`            |
| 3             | Kişisel ajan Skills'leri | `~/.agents/skills`                      |
| 4             | Yönetilen / yerel Skills | `~/.openclaw/skills`                    |
| 5             | Paketle gelen Skills    | kurulumla birlikte gelir                |
| 6 — en düşük  | Ek dizinler             | `skills.load.extraDirs` + Plugin Skills'leri |

Skill kökleri gruplandırılmış düzenleri destekler. OpenClaw, yapılandırılmış bir kökün herhangi bir
yerinde `SKILL.md` bulunduğunda (en fazla 6 düzey derinlikte) bir Skill keşfeder:

```text
<workspace>/skills/research/SKILL.md          ✓ "research" olarak bulundu
<workspace>/skills/personal/research/SKILL.md ✓ ayrıca "research" olarak bulundu
```

Klasör yolu yalnızca düzenleme amaçlıdır. Skill'in adı ve eğik çizgi komutu
`name` frontmatter alanından gelir (`name` eksikse dizin adından
gelir). Ajan izin listeleri (aşağıda) de bu `name` ile eşleşir.

<Note>
  Codex CLI'ın yerel `$CODEX_HOME/skills` dizini bir OpenClaw
  Skill kökü **değildir**. Bu Skills'lerin envanterini çıkarmak için `openclaw migrate plan codex`, ardından
  bunları OpenClaw çalışma alanınıza kopyalamak için `openclaw migrate codex` kullanın.
</Note>

## Node üzerinde barındırılan Skills

Bağlı bir başsız Node, etkin OpenClaw Skills dizininde yüklü Skills'leri
yayımlayabilir (varsayılan olarak `~/.openclaw/skills`; profil ortamı geçersiz kılmaları
uygulanır). Node bağlıyken normal ajan Skill listesinde görünür, bağlantısı
kesildiğinde kaybolurlar. Ad çakışmasında yerel veya Gateway Skill kendi adını
korur; Node Skill, belirlenimci ve Node önekli bir ad alır.
Node üzerinde barındırılan v1, dizin adının Skill'in `name`
frontmatter alanıyla eşleşmesini gerektirir.

Skill girdisi Node konumlandırıcısını içerir. Dosyaları, göreli başvuruları ve
ikili dosyaları Node üzerinde bulunduğundan, `exec host=node node=<node-id>` ile
yükleyip çalıştırın. Skill dosyalarını değiştirdikten sonra Node ana bilgisayarını
yeniden başlatın. Eşleştirme ve devre dışı bırakma seçenekleri için [Node'lar](/tr/nodes#node-hosted-skills) bölümüne bakın.

## Ajan başına ve paylaşılan Skills

Çok ajanlı kurulumlarda her ajanın kendi çalışma alanı vardır. İstediğiniz
görünürlüğe karşılık gelen yolu kullanın:

| Kapsam         | Yol                          | Görünürlük                       |
| -------------- | ---------------------------- | -------------------------------- |
| Ajan başına    | `<workspace>/skills`         | Yalnızca o ajan                  |
| Proje ajanı    | `<workspace>/.agents/skills` | Yalnızca o çalışma alanının ajanı |
| Kişisel ajan   | `~/.agents/skills`           | Bu makinedeki tüm ajanlar        |
| Paylaşılan yönetilen | `~/.openclaw/skills`    | Bu makinedeki tüm ajanlar        |
| Ek dizinler    | `skills.load.extraDirs`      | Bu makinedeki tüm ajanlar        |

## Ajan izin listeleri

Skill **konumu** (öncelik) ve Skill **görünürlüğü** (hangi ajanın
kullanabileceği) ayrı denetimlerdir. Nereden yüklendiklerine bakılmaksızın bir ajanın
hangi Skills'leri göreceğini kısıtlamak için izin listelerini kullanın.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // paylaşılan temel küme
    },
    list: [
      { id: "writer" }, // github ve weather'ı devralır
      { id: "docs", skills: ["docs-search"] }, // varsayılanların tamamının yerine geçer
      { id: "locked-down", skills: [] }, // Skill yok
    ],
  },
}
```

<AccordionGroup>
  <Accordion title="İzin listesi kuralları">
    - Tüm Skills'leri varsayılan olarak kısıtlamadan bırakmak için `agents.defaults.skills` öğesini atlayın.
    - `agents.defaults.skills` öğesini devralmak için `agents.entries.*.skills` öğesini atlayın.
    - O ajan için hiçbir Skill sunmamak üzere `agents.entries.*.skills: []` değerini ayarlayın.
    - Boş olmayan bir `agents.entries.*.skills` listesi **nihai** kümedir — varsayılanlarla
      birleştirilmez.
    - Etkin izin listesi; istem oluşturma, eğik çizgi komutu
      keşfi, korumalı alan eşitlemesi ve Skill anlık görüntüleri genelinde uygulanır.
    - Bu, ana bilgisayar kabuğu için bir yetkilendirme sınırı değildir. Aynı ajan
      `exec` kullanabiliyorsa bu kabuğu korumalı alan, işletim sistemi kullanıcısı
      yalıtımı, yürütme engelleme/izin listeleri ve kaynak başına kimlik bilgileriyle ayrıca kısıtlayın.
  </Accordion>
</AccordionGroup>

## Plugin'ler ve Skills

Plugin'ler, `openclaw.plugin.json` içinde `skills` dizinlerini
listeleyerek kendi Skills'lerini sunabilir (yollar Plugin köküne göredir). Plugin Skills'leri,
Plugin etkinleştirildiğinde yüklenir — örneğin tarayıcı Plugin'i, çok adımlı
tarayıcı denetimi için bir `browser-automation` Skill'i sunar.

Plugin Skill dizinleri, `skills.load.extraDirs` ile aynı düşük öncelik düzeyinde
birleştirilir; dolayısıyla aynı adlı paketle gelen, yönetilen, ajan veya çalışma alanı
Skill'i bunları geçersiz kılar. Bir Plugin Skill'inin kendi uygunluğunu, diğer tüm
Skills'lerde olduğu gibi frontmatter bölümündeki `metadata.openclaw.requires` aracılığıyla belirleyin.

Tam Plugin sistemi için [Plugin'ler](/tr/tools/plugin) ve [Araçlar](/tr/tools) bölümlerine bakın.

## Skill Atölyesi

[Skill Atölyesi](/tr/tools/skill-workshop), ajan ile etkin Skill dosyalarınız
arasında yer alan bir öneri kuyruğudur. Ajan yeniden kullanılabilir bir çalışma fark ettiğinde
doğrudan `SKILL.md` içine yazmak yerine bir öneri taslağı hazırlar. Herhangi bir
değişiklik yapılmadan önce bunu inceler ve onaylarsınız.

```bash
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

Tam yaşam döngüsü, CLI başvurusu ve yapılandırma için
[Skill Atölyesi](/tr/tools/skill-workshop) bölümüne bakın.

## ClawHub'dan yükleme

[ClawHub](https://clawhub.ai), herkese açık Skills kayıt defteridir. Yükleme ve güncelleme
için `openclaw skills` komutlarını, yayımlama ve eşitleme için `clawhub`
CLI'ını kullanın.

| Eylem                                  | Komut                                                  |
| -------------------------------------- | ------------------------------------------------------ |
| Çalışma alanına bir Skill yükle        | `openclaw skills install @owner/<slug>`                |
| Git deposundan yükle                   | `openclaw skills install git:owner/repo@ref`           |
| Yerel bir Skill dizini yükle           | `openclaw skills install ./path/to/skill --as my-tool` |
| Tüm yerel ajanlar için yükle           | `openclaw skills install @owner/<slug> --global`       |
| Tüm çalışma alanı Skills'lerini güncelle | `openclaw skills update --all`                         |
| Paylaşılan yönetilen bir Skill'i güncelle | `openclaw skills update @owner/<slug> --global`        |
| Tüm paylaşılan yönetilen Skills'leri güncelle | `openclaw skills update --all --global`                |
| Bir Skill'in güven sınırlarını doğrula | `openclaw skills verify @owner/<slug>`                 |
| Oluşturulan Skill Kartını yazdır       | `openclaw skills verify @owner/<slug> --card`          |
| ClawHub CLI aracılığıyla yayımla / eşitle | `clawhub sync --all`                                   |

<AccordionGroup>
  <Accordion title="Yükleme ayrıntıları">
    `openclaw skills install`, varsayılan olarak etkin çalışma alanının `skills/`
    dizinine yükleme yapar. Ajan izin listeleri kapsamı daraltmadığı sürece tüm yerel
    ajanlar tarafından görülebilen paylaşılan `~/.openclaw/skills` dizinine yüklemek
    için `--global` ekleyin.

    Git ve yerel yüklemeler, kaynak kökünde `SKILL.md` bulunmasını bekler. Kısa ad,
    geçerliyse `SKILL.md` frontmatter `name` değerinden gelir; ardından
    dizin veya depo adına geri döner. Geçersiz kılmak için `--as <slug>` kullanın.
    `openclaw skills update` yalnızca ClawHub yüklemelerini izler — Git veya
    yerel kaynakları yenilemek için yeniden yükleyin.

  </Accordion>
  <Accordion title="Doğrulama ve güvenlik taraması">
    `openclaw skills verify @owner/<slug>`, ClawHub'dan Skill'in
    `clawhub.skill.verify.v1` güven sınırlarını ister. Yüklü ClawHub Skills'leri,
    `.clawhub/origin.json` içinde kaydedilen sürüme ve kayıt defterine göre doğrulanır.
    Yalın kısa adlar, mevcut yüklü veya belirsizlik taşımayan Skills için kabul edilmeye
    devam eder; ancak sahip nitelemeli başvurular yayımlayıcı belirsizliğini önler.

    ClawHub Skill sayfaları, yüklemeden önce en son güvenlik taraması durumunu;
    VirusTotal, ClawScan ve statik analiz ayrıntı sayfalarıyla birlikte gösterir.
    ClawHub doğrulamayı başarısız olarak işaretlediğinde komut sıfırdan farklı bir kodla çıkar.
    Yayımlayıcılar yanlış pozitifleri ClawHub panosu veya
    `clawhub skill rescan @owner/<slug>` aracılığıyla düzeltebilir.

  </Accordion>
  <Accordion title="Özel arşiv yüklemeleri">
    ClawHub dışı teslimata ihtiyaç duyan Gateway istemcileri, `skills.upload.begin`,
    `skills.upload.chunk` ve `skills.upload.commit` ile bir zip Skill arşivini hazırlayıp
    ardından `skills.install({ source: "upload", ... })` ile yükleyebilir. Bu yol
    varsayılan olarak kapalıdır ve `openclaw.json` içinde
    `skills.install.allowUploadedArchives: true` gerektirir. Normal ClawHub yüklemeleri bu ayara hiçbir zaman ihtiyaç duymaz.
  </Accordion>
</AccordionGroup>

## Güvenlik

<Warning>
  Üçüncü taraf Skills'lerini **güvenilmeyen kod** olarak değerlendirin. Etkinleştirmeden
  önce okuyun. Güvenilmeyen girdiler ve riskli araçlar için korumalı alan çalıştırmalarını
  tercih edin. Ajan tarafındaki denetimler için [Korumalı Alan](/tr/gateway/sandboxing) bölümüne bakın.
</Warning>

<AccordionGroup>
  <Accordion title="Yol sınırlaması">
    Çalışma alanı, proje ajanı ve ek dizin Skill keşfi; `skills.load.allowSymlinkTargets` bir hedef
    köke açıkça güvenmediği sürece yalnızca çözümlenen gerçek yolu yapılandırılmış kökün
    içinde kalan Skill köklerini kabul eder.
    Skill Atölyesi, yalnızca `skills.workshop.allowSymlinkTargetWrites` etkinleştirildiğinde bu güvenilen
    hedeflere yazar.
    Yönetilen `~/.openclaw/skills` ve kişisel `~/.agents/skills`, sembolik bağlantılı
    Skill klasörleri içerebilir; ancak her `SKILL.md` gerçek yolu yine
    çözümlenen Skill dizininin içinde kalmalıdır.
  </Accordion>
  <Accordion title="Operatör yükleme politikası">
    Skill yüklemeleri devam etmeden önce güvenilir bir yerel politika komutu
    çalıştırmak için `security.installPolicy` yapılandırın. Politika, meta verileri ve hazırlanan
    kaynak yolunu alır; ClawHub, yüklenen dosya, Git, yerel, güncelleme ve
    bağımlılık yükleyicisi yollarına uygulanır ve komut geçerli bir karar
    döndüremediğinde güvenli biçimde başarısız olur.
  </Accordion>
  <Accordion title="Gizli bilgi ekleme kapsamı">
    `skills.entries.*.env` ve `skills.entries.*.apiKey`, gizli bilgileri yalnızca o ajan turu
    için **ana bilgisayar** sürecine ekler — korumalı alana eklemez. Gizli bilgileri
    istemlerden ve günlüklerden uzak tutun.
  </Accordion>
</AccordionGroup>

Daha geniş tehdit modeli ve güvenlik kontrol listeleri için
[Güvenlik](/tr/gateway/security) bölümüne bakın.

## SKILL.md biçimi

Her Skill'in frontmatter bölümünde en az bir `name` ve `description` bulunmalıdır:

```markdown
---
name: image-lab
description: Sağlayıcı destekli bir görüntü iş akışı aracılığıyla görüntüler oluşturun veya düzenleyin
---

Kullanıcı bir görüntü oluşturmak istediğinde `image_generate` aracını kullanın...
```

<Note>
  OpenClaw, [AgentSkills](https://agentskills.io) belirtimini izler. Frontmatter
  önce YAML olarak ayrıştırılır; bu başarısız olursa yalnızca tek satırlı
  ayrıştırıcıya geri döner. İç içe `metadata` blokları (çok satırlı YAML eşlemeleri
  dâhil) bir JSON dizesine düzleştirilir ve JSON5 olarak yeniden ayrıştırılır; bu nedenle
  [Koşullama](#gating) altında gösterilen blok biçimi çalışır. Skill klasör yoluna başvurmak
  için gövdede `{baseDir}` kullanın.
</Note>

### İsteğe bağlı frontmatter anahtarları

<ParamField path="homepage" type="string">
  macOS Skills kullanıcı arayüzünde "Web Sitesi" olarak gösterilen URL. Ayrıca
  `metadata.openclaw.homepage` aracılığıyla da desteklenir.
</ParamField>

<ParamField path="user-invocable" type="boolean" default="true">
  `true` olduğunda Skills, kullanıcı tarafından çağrılabilen bir eğik çizgi komutu olarak sunulur.
</ParamField>

<ParamField path="disable-model-invocation" type="boolean" default="false">
  `true` olduğunda OpenClaw, Skills talimatlarını aracının normal
  istemine dahil etmez. Skills, `user-invocable`
  aynı zamanda `true` olduğunda eğik çizgi komutu olarak kullanılmaya devam eder.
</ParamField>

<ParamField path="command-dispatch" type='"tool"'>
  `tool` olarak ayarlandığında eğik çizgi komutu modeli atlar ve
  doğrudan kayıtlı bir araca yönlendirilir.
</ParamField>

<ParamField path="command-tool" type="string">
  `command-dispatch: tool` ayarlandığında çağrılacak araç adı.
</ParamField>

<ParamField path="command-arg-mode" type='"raw"' default="raw">
  Araç yönlendirmesinde, ham argüman dizesini çekirdek tarafından ayrıştırılmadan
  araca iletir. Araç,
  `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }` alır.
</ParamField>

## Geçitler

OpenClaw, yükleme sırasında Skills öğelerini `metadata.openclaw` kullanarak filtreler (frontmatter içine
yerleştirilmiş JSON5 nesnesi; yukarıdaki ayrıştırma notuna bakın). `metadata.openclaw`
bloğu olmayan bir Skills, açıkça devre dışı bırakılmadığı sürece her zaman uygundur.

```markdown
---
name: image-lab
description: Sağlayıcı destekli bir görüntü iş akışı aracılığıyla görüntüler oluşturun veya düzenleyin
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
---
```

<ParamField path="always" type="boolean">
  `true` olduğunda Skills öğesini her zaman dahil eder ve diğer tüm geçitleri atlar.
</ParamField>

<ParamField path="emoji" type="string">
  macOS Skills kullanıcı arayüzünde gösterilen isteğe bağlı emoji.
</ParamField>

<ParamField path="homepage" type="string">
  macOS Skills kullanıcı arayüzünde "Website" olarak gösterilen isteğe bağlı URL.
</ParamField>

<ParamField path="os" type='("darwin" | "linux" | "win32")[]'>
  Platform filtresi. Ayarlandığında Skills yalnızca listelenen bir işletim sisteminde uygundur.
</ParamField>

<ParamField path="requires.bins" type="string[]">
  Her ikili dosya `PATH` üzerinde bulunmalıdır.
</ParamField>

<ParamField path="requires.anyBins" type="string[]">
  En az bir ikili dosya `PATH` üzerinde bulunmalıdır.
</ParamField>

<ParamField path="requires.env" type="string[]">
  Her ortam değişkeni süreçte bulunmalı veya yapılandırma aracılığıyla sağlanmalıdır.
</ParamField>

<ParamField path="requires.config" type="string[]">
  Her `openclaw.json` yolu doğru değerli olmalıdır.
</ParamField>

<ParamField path="primaryEnv" type="string">
  `skills.entries.<name>.apiKey` ile ilişkilendirilmiş ortam değişkeni adı.
</ParamField>

<ParamField path="install" type="object[]">
  macOS Skills kullanıcı arayüzü tarafından kullanılan isteğe bağlı yükleyici tanımları (brew / node / go / uv / download).
</ParamField>

<Note>
  Eski `metadata.clawdbot` blokları, `metadata.openclaw`
  bulunmadığında hâlâ kabul edilir; böylece önceden yüklenmiş Skills öğeleri
  bağımlılık geçitlerini ve yükleyici ipuçlarını korur. Yeni Skills öğeleri
  `metadata.openclaw` kullanmalıdır.
</Note>

### Yükleyici tanımları

Yükleyici tanımları, macOS Skills kullanıcı arayüzüne bir bağımlılığın nasıl yükleneceğini bildirir:

```markdown
---
name: gemini
description: Kodlama desteği ve Google arama sorguları için Gemini CLI'ı kullanın.
metadata:
  {
    "openclaw":
      {
        "emoji": "♊️",
        "requires": { "bins": ["gemini"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gemini-cli",
              "bins": ["gemini"],
              "label": "Gemini CLI'ı yükle (brew)",
            },
          ],
      },
  }
---
```

<AccordionGroup>
  <Accordion title="Yükleyici seçim kuralları">
    - Birden fazla yükleyici listelendiğinde Gateway, tercih edilen bir
      seçeneği belirler (varsa brew, aksi takdirde node).
    - Tüm yükleyiciler `download` ise OpenClaw, kullanılabilir tüm
      yapıtları görebilmeniz için her girdiyi listeler.
    - Tanımlar, platforma göre filtreleme yapmak için `os: ["darwin"|"linux"|"win32"]` içerebilir.
    - Node yüklemeleri, `openclaw.json` içindeki `skills.install.nodeManager` ayarını
      dikkate alır (varsayılan: npm; seçenekler: npm / pnpm / yarn / bun). Bu yalnızca Skills
      yüklemelerini etkiler; Gateway çalışma zamanı yine Node olmalıdır.
    - Gateway yükleyici tercihi: Homebrew → uv → yapılandırılmış node yöneticisi →
      go → indirme.
  </Accordion>
  <Accordion title="Yükleyici bazında ayrıntılar">
    - **Homebrew:** OpenClaw, Homebrew'u otomatik olarak yüklemez veya brew
      formüllerini sistem paketi komutlarına dönüştürmez. `brew`
      bulunmayan Linux kapsayıcılarında yalnızca brew kullanan yükleyiciler gizlenir; özel bir
      imaj kullanın veya bağımlılığı manuel olarak yükleyin.
    - **Go:** OpenClaw, otomatik Skills yüklemeleri için Go 1.21 veya daha yeni bir sürüm gerektirir.
      `go` eksikse ve Homebrew kullanılabiliyorsa OpenClaw önce Go'yu
      Homebrew aracılığıyla yükler; Homebrew bulunmayan Linux'ta ise yenilenmiş `golang-go`
      adayı minimum sürümü karşılıyorsa bunun yerine root olarak veya parolasız `sudo`
      üzerinden `apt-get` kullanabilir. Bağımlılığa yönelik gerçek `go install`,
      yapılandırılmış `GOBIN` yerine her zaman OpenClaw tarafından yönetilen özel bir
      ikili dosya dizinini (yeni yüklemede Homebrew'un `bin` dizini, aksi takdirde
      `~/.local/bin`) hedefler — size ait `GOBIN`, `GOPATH` ve `GOTOOLCHAIN`
      ortam değişkenleri okunur ancak hiçbir zaman üzerine yazılmaz.
    - **İndirme:** `url` (gerekli), `archive` (`tar.gz` | `tar.bz2` | `zip`),
      `extract` (varsayılan: arşiv algılandığında otomatik), `stripComponents`,
      `targetDir` (varsayılan: `~/.openclaw/tools/<skillKey>`).
  </Accordion>
  <Accordion title="Korumalı alan notları">
    `requires.bins`, Skills yükleme sırasında **ana makinede** denetlenir. Bir aracı
    korumalı alanda çalışıyorsa ikili dosya **kapsayıcının içinde** de bulunmalıdır.
    Bunu `agents.defaults.sandbox.docker.setupCommand` veya özel bir
    imaj aracılığıyla yükleyin. `setupCommand`, kapsayıcı oluşturulduktan sonra bir kez çalışır ve
    ağ çıkışı, yazılabilir bir kök dosya sistemi ve korumalı alanda bir root kullanıcısı gerektirir.
  </Accordion>
</AccordionGroup>

## Yapılandırma geçersiz kılmaları

Paketlenmiş veya yönetilen Skills öğelerini
`~/.openclaw/openclaw.json` içindeki `skills.entries` altında etkinleştirin ve yapılandırın:

```json5
{
  skills: {
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
        config: {
          endpoint: "https://example.invalid",
          model: "nano-pro",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<ParamField path="enabled" type="boolean">
  `false`, paketlenmiş veya yüklenmiş olsa bile Skills öğesini devre dışı bırakır. `coding-agent`
  paketlenmiş Skills öğesi isteğe bağlıdır — `skills.entries.coding-agent.enabled: true` değerini ayarlayın
  ve `claude`, `codex`, `opencode` veya desteklenen başka bir CLI'ın
  yüklendiğinden ve kimliğinin doğrulandığından emin olun.
</ParamField>

<ParamField path="apiKey" type='string | { source, provider, id }'>
  `metadata.openclaw.primaryEnv` bildiren Skills öğeleri için kolaylık alanı.
  Düz metin dizesini veya SecretRef nesnesini destekler.
</ParamField>

<ParamField path="env" type="Record<string, string>">
  Aracı çalıştırması için enjekte edilen ortam değişkenleri. Yalnızca değişken
  süreçte zaten ayarlanmamışsa enjekte edilir.
</ParamField>

<ParamField path="config" type="object">
  Skills bazında özel yapılandırma alanları için isteğe bağlı alan grubu.
</ParamField>

<ParamField path="allowBundled" type="string[]">
  Yalnızca **paketlenmiş** Skills öğeleri için isteğe bağlı izin listesi. Ayarlandığında yalnızca
  listedeki paketlenmiş Skills öğeleri uygun olur. Yönetilen ve çalışma alanı Skills öğeleri etkilenmez.
</ParamField>

<Note>
  Yapılandırma anahtarları varsayılan olarak **Skills adıyla** eşleşir. Bir Skills
  `metadata.openclaw.skillKey` tanımlıyorsa bunun yerine `skills.entries` altında bu anahtarı kullanın.
  Kısa çizgili adları tırnak içine alın: JSON5, tırnak içine alınmış anahtarlara izin verir.
</Note>

## Ortam enjeksiyonu

Bir aracı çalıştırması başladığında OpenClaw:

<Steps>
  <Step title="Skills meta verilerini okur">
    OpenClaw; geçit kurallarını, izin listelerini ve yapılandırma geçersiz kılmalarını
    uygulayarak aracı için geçerli Skills listesini çözümler.
  </Step>
  <Step title="Ortam değişkenlerini ve API anahtarlarını enjekte eder">
    `skills.entries.<key>.env` ve `skills.entries.<key>.apiKey`,
    çalıştırma süresince `process.env` üzerine uygulanır.
  </Step>
  <Step title="Sistem istemini oluşturur">
    Uygun Skills öğeleri derlenerek kompakt bir XML bloğuna dönüştürülür ve
    sistem istemine enjekte edilir.
  </Step>
  <Step title="Ortamı geri yükler">
    Çalıştırma sona erdikten sonra özgün ortam geri yüklenir.
  </Step>
</Steps>

<Warning>
  Ortam değişkeni enjeksiyonu, korumalı alana değil **ana makinedeki** aracı çalıştırmasına özgüdür.
  Korumalı alan içinde `env` ve `apiKey` etkisizdir. Gizli bilgilerin
  korumalı alan çalıştırmalarına nasıl aktarılacağı için
  [Skills yapılandırması](/tr/tools/skills-config#sandboxed-skills-and-env-vars) bölümüne bakın.
</Warning>

Paketlenmiş `claude-cli` arka ucu için OpenClaw, aynı uygun Skills
anlık görüntüsünü geçici bir Claude Code Plugin'i olarak da somutlaştırır ve
`--plugin-dir` aracılığıyla iletir. Diğer CLI arka uçları yalnızca istem kataloğunu kullanır.

## Anlık görüntüler ve yenileme

OpenClaw, uygun Skills öğelerinin anlık görüntüsünü **oturum başladığında** alır ve
bu listeyi oturumdaki sonraki tüm dönüşlerde yeniden kullanır. Skills veya yapılandırma
değişiklikleri bir sonraki yeni oturumda yürürlüğe girer.

Skills, oturumun ortasında iki durumda yenilenir:

- Skills izleyicisi bir `SKILL.md` değişikliği algılar.
- Yeni bir uygun uzak Node bağlanır.

Yenilenen liste, bir sonraki aracı dönüşünde alınır. Geçerli aracı
izin listesi değişirse OpenClaw, görünür Skills öğelerini uyumlu tutmak için
anlık görüntüyü yeniler.

<AccordionGroup>
  <Accordion title="Skills izleyicisi">
    OpenClaw varsayılan olarak Skills klasörlerini izler ve `SKILL.md`
    dosyaları değiştiğinde anlık görüntüyü ilerletir. `skills.load` altında yapılandırın:

    ```json5
    {
      skills: {
        load: {
          extraDirs: ["~/Projects/agent-scripts/skills"],
          allowSymlinkTargets: ["~/Projects/manager/skills"],
          watch: true, // varsayılan
        },
      },
    }
    ```

    İzleyici olayları yerleşik 250 ms bekletme kullanır. Bir Skills
    kök sembolik bağlantısının yapılandırılmış kökün dışını gösterdiği, örneğin
    `<workspace>/skills/manager -> ~/Projects/manager/skills` gibi bilinçli sembolik bağlantı düzenleri için `allowSymlinkTargets`
    kullanın.
    `skills.workshop.allowSymlinkTargetWrites` seçeneğini yalnızca Skills Workshop'un
    önerileri bu güvenilir sembolik bağlantılı yollar üzerinden de uygulaması gerektiğinde etkinleştirin.

  </Accordion>
  <Accordion title="Uzak macOS Node'ları (Linux Gateway)">
    Gateway Linux'ta çalışıyor ancak `system.run` izni verilmiş bir **macOS Node'u**
    bağlıysa OpenClaw, gerekli ikili dosyalar bu Node'da bulunduğunda yalnızca macOS'a özgü
    Skills öğelerini uygun kabul edebilir. Aracı, bu Skills öğelerini `host=node` ile
    `exec` aracını kullanarak çalıştırmalıdır.

    Çevrimdışı Node'lar, yalnızca uzaktan kullanılabilen Skills öğelerini görünür **kılmaz**.
    Bir Node ikili dosya sorgularına yanıt vermeyi durdurursa OpenClaw, önbelleğe alınmış
    ikili dosya eşleşmelerini temizler.

  </Accordion>
</AccordionGroup>

## Token etkisi

Skills öğeleri uygun olduğunda OpenClaw, sistem istemine kompakt bir XML bloğu
enjekte eder. Maliyet belirlenebilirdir ve Skills başına doğrusal olarak ölçeklenir:

- **Temel ek yük** (yalnızca 1+ Skills uygun olduğunda): sabit bir giriş
  metni bloğu ve `<available_skills>` sarmalayıcısı.
- **Skills başına:** ~97 karakter + `name`, `description` ve `location`
  alan uzunluklarınız.
- XML kaçış işlemi, `& < > " '` öğesini varlıklara genişleterek her
  oluşum başına birkaç karakter ekler.
- ~4 karakter/token oranında, alan uzunluklarından önce Skills başına 97 karakter ≈ 24 token eder.

İşlenen blok yapılandırılmış istem bütçesini
(`skills.limits.maxSkillsPromptChars`) aşacaksa OpenClaw önce açıklamasız kompakt biçime
sığabilecek kadar çok Skills kimliğini (ad, konum ve sürüm) korur. Ardından kalan
bütçeyi kısaltılmış açıklamalar için kullanır. Açıklama bütçesi kalmazsa açıklamalar
çıkarılır. Kompakt biçimlendirme veya liste kısaltma gerektiğinde istemde
`openclaw skills check` öğesine yönlendiren bir not bulunur.

İstem ek yükünü en aza indirmek için açıklamaları kısa ve açıklayıcı tutun.

## İlgili

<CardGroup cols={2}>
  <Card title="Skills oluşturma" href="/tr/tools/creating-skills" icon="hammer">
    Özel bir skill yazmaya yönelik adım adım kılavuz.
  </Card>
  <Card title="Skill Atölyesi" href="/tr/tools/skill-workshop" icon="flask">
    Agent tarafından taslak hâline getirilen skill'ler için öneri kuyruğu.
  </Card>
  <Card title="Skills yapılandırması" href="/tr/tools/skills-config" icon="gear">
    Tam `skills.*` yapılandırma şeması ve agent izin listeleri.
  </Card>
  <Card title="Eğik çizgi komutları" href="/tr/tools/slash-commands" icon="terminal">
    Skill eğik çizgi komutlarının nasıl kaydedildiği ve yönlendirildiği.
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="cloud">
    Herkese açık kayıt defterindeki skill'lere göz atın ve skill yayımlayın.
  </Card>
  <Card title="Plugin'ler" href="/tr/tools/plugin" icon="plug">
    Plugin'ler, belgeledikleri araçlarla birlikte skill'ler sunabilir.
  </Card>
</CardGroup>
