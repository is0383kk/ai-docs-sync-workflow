---
read_when:
    - Skills yükleme, kurulum veya erişim denetimi davranışını yapılandırma
    - Aracı başına Skills görünürlüğünü ayarlama
    - Skills Atölyesi sınırlarını veya onay politikasını ayarlama
sidebarTitle: Skills config
summary: skills.* yapılandırma şeması, ajan izin listeleri, workshop ayarları ve korumalı alan ortam değişkeni işleme yöntemine ilişkin eksiksiz referans.
title: Skills yapılandırması
x-i18n:
    generated_at: "2026-07-26T23:07:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bc154bdf8a8537095a4d39bc6e86ebfd716e6beacd45def9c8a1c15fcdc93698
    source_path: tools/skills-config.md
    workflow: 16
---

OpenClaw’da çoğu Skills yapılandırması `~/.openclaw/openclaw.json` içindeki
`skills` altında bulunur. Agent’a özgü görünürlük
`agents.defaults.skills` ve `agents.entries.*.skills` altında bulunur.

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
      watch: true,
    },
    install: {
      preferBrew: true,
      nodeManager: "npm",
      allowUploadedArchives: false,
    },
    workshop: {
      autonomous: { enabled: false },
      allowSymlinkTargetWrites: false,
      approvalPolicy: "auto",
      maxPending: 50,
      maxSkillBytes: 40000,
    },
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<Note>
  Yerleşik görüntü üretimi için `skills.entries` yerine `agents.defaults.mediaModels.image`
  ile temel `image_generate` aracını kullanın. Skills
  girdileri yalnızca özel veya üçüncü taraf Skills iş akışları içindir.
</Note>

## Yükleme (`skills.load`)

<ParamField path="skills.load.extraDirs" type="string[]">
  En düşük öncelikle (paketle gelen ve Plugin Skills öğelerinin altında)
  taranacak ek Skills dizinleri. Yollar, `~` desteğiyle genişletilir.
</ParamField>

<ParamField path="skills.load.allowSymlinkTargets" type="string[]">
  Sembolik bağlantılı Skills klasörlerinin, sembolik bağlantı yapılandırılmış
  kökün dışında olsa bile çözümleyebileceği güvenilir gerçek hedef dizinler.
  Bunu `<workspace>/skills/manager -> ~/Projects/manager/skills` gibi bilinçli kardeş depo düzenleri için
  kullanın. Bu listeyi dar tutun — `~` veya `~/Projects` gibi
  geniş kökleri göstermeyin.
</ParamField>

<ParamField path="skills.load.watch" type="boolean" default="true">
  Skills klasörlerini izler ve `SKILL.md` dosyaları değiştiğinde
  Skills anlık görüntüsünü yeniler. Gruplandırılmış Skills köklerinin altındaki
  iç içe dosyaları kapsar.
</ParamField>

## Kurulum (`skills.install`)

<ParamField path="skills.install.preferBrew" type="boolean" default="true">
  `brew` kullanılabiliyorsa Homebrew yükleyicilerini tercih eder.
</ParamField>

<ParamField path="skills.install.nodeManager" type='"npm" | "pnpm" | "yarn" | "bun"' default='"npm"'>
  Skills kurulumları için Node paket yöneticisi tercihi. Bu yalnızca Skills
  kurulumlarını etkiler; standart durum deposu `node:sqlite` kullandığından
  OpenClaw CLI ve Gateway çalışma zamanı Node gerektirir. `openclaw setup --node-manager` ve
  `openclaw onboard --node-manager`; `npm`, `pnpm` veya `bun` değerlerini
  kabul eder. Yarn destekli Skills kurulumları için yapılandırmada
  doğrudan `"yarn"` değerini ayarlayın.
</ParamField>

<ParamField path="skills.install.allowUploadedArchives" type="boolean" default="false">
  Güvenilir `operator.admin` Gateway istemcilerinin `skills.upload.*` üzerinden
  hazırlanan özel zip arşivlerini kurmasına izin verir. Normal ClawHub
  kurulumları bu ayara ihtiyaç duymaz.
</ParamField>

## Operatör Kurulum İlkesi (`security.installPolicy`)

Operatörlerin, Skills ve Plugin kurulumlarını ana makineye özgü bir ilkeyle
onaylamak veya engellemek için güvenilir bir yerel komuta ihtiyaç duyduğu
durumlarda `security.installPolicy` kullanın. İlke, OpenClaw kaynak materyali
hazırladıktan sonra ve kurulum ya da güncelleme devam etmeden önce çalışır.
ClawHub Skills öğeleri, yüklenen Skills öğeleri, Git/yerel Skills öğeleri,
Skills bağımlılık yükleyicileri ve Plugin kurulum/güncelleme kaynakları için
geçerlidir.

```json5
{
  security: {
    installPolicy: {
      enabled: true,
      // Desteklenen her hedefi kapsamak için targets alanını atlayın.
      targets: ["skill", "plugin"],
      exec: {
        source: "exec",
        command: "/usr/local/bin/openclaw-install-policy",
        args: ["--json"],
        timeoutMs: 10000,
        noOutputTimeoutMs: 10000,
        maxOutputBytes: 1048576,
        passEnv: ["OPENCLAW_STATE_DIR", "PATH"],
        env: { POLICY_MODE: "strict" },
        trustedDirs: ["/usr/local/bin"],
      },
    },
  },
}
```

<ParamField path="security.installPolicy.enabled" type="boolean" default="false">
  Operatöre ait kurulum ilkesini etkinleştirir. Geçerli bir `exec`
  komutu olmadan etkinleştirildiğinde kurulumlar güvenli biçimde başarısız olur.
</ParamField>

<ParamField path="security.installPolicy.targets" type='("skill" | "plugin")[]'>
  İsteğe bağlı hedef filtresi. Atlandığında ilke, yeni kurulumların beklenmedik
  şekilde açık kalmaması için desteklenen her hedefe uygulanır.
</ParamField>

<ParamField path="security.installPolicy.exec.command" type="string">
  Güvenilir ilke yürütülebilir dosyasının mutlak yolu. OpenClaw bunu kabuk
  kullanmadan çalıştırır ve kullanmadan önce yolu doğrular.
</ParamField>

<ParamField path="security.installPolicy.exec.args" type="string[]">
  `command` sonrasında aktarılan statik bağımsız değişkenler.
</ParamField>

<ParamField path="security.installPolicy.exec.timeoutMs" type="number" default="10000">
  Tek bir ilke kararı için azami gerçek zaman çalışma süresi.
</ParamField>

<ParamField path="security.installPolicy.exec.noOutputTimeoutMs" type="number" default="timeoutMs">
  İlkenin güvenli biçimde başarısız olmasından önce stdout veya stderr çıktısı
  olmadan geçebilecek azami süre.
</ParamField>

<ParamField path="security.installPolicy.exec.maxOutputBytes" type="number" default="1048576">
  İlke işleminden kabul edilen stdout ve stderr çıktılarının toplam azami bayt
  sayısı.
</ParamField>

<ParamField path="security.installPolicy.exec.env" type="Record<string, string>">
  İlke işlemine sağlanan sabit ortam değişkenleri.
</ParamField>

<ParamField path="security.installPolicy.exec.passEnv" type="string[]">
  OpenClaw işleminden ilke işlemine kopyalanan ortam değişkeni adları. Yalnızca
  adı belirtilen değişkenler aktarılır.
</ParamField>

<ParamField path="security.installPolicy.exec.trustedDirs" type="string[]">
  İlke yürütülebilir dosyasını içerebilecek dizinlerin isteğe bağlı izin listesi.
</ParamField>

<ParamField path="security.installPolicy.exec.allowInsecurePath" type="boolean" default="false">
  Komut yolu sahipliği ve izin denetimlerini atlar. Yalnızca yol başka bir
  mekanizmayla korunuyorsa kullanın.
</ParamField>

<ParamField path="security.installPolicy.exec.allowSymlinkCommand" type="boolean" default="false">
  Yapılandırılmış komut yolunun sembolik bağlantı olmasına izin verir.
  Çözümlenen hedef yine de diğer yol denetimlerini karşılamalıdır. Yorumlayıcı
  betik bağımsız değişkenleri sembolik bağlantı değil, doğrudan normal dosyalar
  olmalıdır.
</ParamField>

İlke stdin üzerinden `protocolVersion: 1`, `openclawVersion`, `targetType`,
`targetName`, `sourcePath`, `sourcePathKind`, isteğe bağlı yapılandırılmış
`source`, yapılandırılmış `origin` ve `request` içeren tek
bir JSON nesnesi alır. stdout üzerine tek bir JSON nesnesi yazmalıdır:
`{ "protocolVersion": 1, "decision": "allow" }` veya `{ "protocolVersion": 1, "decision": "block", "reason": "..." }`. Sıfır olmayan çıkış, zaman aşımı,
hatalı biçimlendirilmiş JSON, eksik alanlar veya desteklenmeyen protokol
sürümleri güvenli biçimde başarısız olur.

OpenClaw, normal Gateway başlatma sırasında kurulum ilkesini yürütmez.
İlke etkin ancak kullanılamaz olduğunda kurulumlar ve güncellemeler güvenli
biçimde başarısız olur. `openclaw doctor` statik doğrulama gerçekleştirir;
`openclaw doctor --deep` ise yapılandırılmış komuta karşı yapay bir kurulum sınaması
yürütür.

Toplu güncellemeler ilkeyi hedef başına uygular: engellenmiş bir Skills veya
Plugin güncellemesi, ilkeyi devre dışı bırakmadan ya da gruptaki sonraki
hedefleri atlamadan ilgili hedef için başarısız olur.

Örnek stdin:

```json
{
  "protocolVersion": 1,
  "openclawVersion": "2026.6.1",
  "targetType": "skill",
  "targetName": "weather",
  "sourcePath": "/var/folders/.../openclaw-skill-clawhub/root",
  "sourcePathKind": "directory",
  "source": {
    "kind": "clawhub",
    "authority": "openclaw",
    "mutable": false,
    "network": true
  },
  "origin": {
    "type": "clawhub",
    "registry": "https://clawhub.openclaw.ai",
    "slug": "weather",
    "version": "1.0.0"
  },
  "request": {
    "kind": "skill-install",
    "mode": "install",
    "requestedSpecifier": "clawhub:weather@1.0.0"
  },
  "skill": {
    "installId": "clawhub"
  }
}
```

Asgari ilke komutu:

```js
#!/usr/bin/env node

let input = "";
process.stdin.setEncoding("utf8");
process.stdin.on("data", (chunk) => {
  input += chunk;
});
process.stdin.on("end", () => {
  const request = JSON.parse(input);
  if (request.targetType === "plugin" && request.source?.kind === "local-path") {
    process.stdout.write(
      JSON.stringify({
        protocolVersion: 1,
        decision: "block",
        reason: "yerel Plugin yolları bu ana makinede onaylanmamıştır",
      }),
    );
    return;
  }
  process.stdout.write(JSON.stringify({ protocolVersion: 1, decision: "allow" }));
});
```

## Paketle gelen Skills izin listesi

<ParamField path="skills.allowBundled" type="string[]">
  Yalnızca **paketle gelen** Skills öğeleri için isteğe bağlı izin listesi.
  Ayarlandığında yalnızca listedeki paketle gelen Skills öğeleri uygun olur.
  Yönetilen, Agent düzeyindeki ve çalışma alanındaki Skills öğeleri etkilenmez.
</ParamField>

## Skills başına girdiler (`skills.entries`)

`entries` altındaki anahtarlar varsayılan olarak Skills
`name` değeriyle eşleşir. Bir Skills öğesi `metadata.openclaw.skillKey`
tanımlıyorsa bunun yerine o anahtarı kullanın. Kısa çizgili adları tırnak içine
alın (JSON5, tırnaklı anahtarlara izin verir).

<ParamField path="skills.entries.<key>.enabled" type="boolean">
  `false`, paketle gelmiş veya kurulmuş olsa bile Skills öğesini devre
  dışı bırakır. Paketle gelen `coding-agent` Skills öğesi isteğe bağlıdır —
  bunu `true` olarak ayarlayın ve `claude`,
  `codex`, `opencode` veya desteklenen başka bir CLI'ın kurulu
  ve kimliği doğrulanmış olduğundan emin olun.
</ParamField>

<ParamField path="skills.entries.<key>.apiKey" type='string | { source, provider, id }'>
  `metadata.openclaw.primaryEnv` bildiren Skills öğeleri için kolaylık alanı.
  Düz metin dizesini veya SecretRef değerini destekler: `{ source: "env", provider: "default", id: "VAR_NAME" }`.
</ParamField>

<ParamField path="skills.entries.<key>.env" type="Record<string, string>">
  Agent çalıştırması için eklenen ortam değişkenleri. Yalnızca değişken işlemde
  zaten ayarlanmamışsa eklenir.
</ParamField>

<ParamField path="skills.entries.<key>.config" type="object">
  Skills öğesine özgü özel yapılandırma alanları için isteğe bağlı bir alan.
</ParamField>

## Agent izin listeleri (`agents`)

Aynı makine/çalışma alanı Skills köklerini, ancak Agent başına farklı bir
görünür Skills kümesini istediğinizde Agent yapılandırmasını kullanın.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // paylaşılan temel
    },
    list: [
      { id: "writer" }, // github ve weather devralınır
      { id: "docs", skills: ["docs-search"] }, // varsayılanların tamamının yerini alır
      { id: "locked-down", skills: [] }, // Skills yok
    ],
  },
}
```

<ParamField path="agents.defaults.skills" type="string[]">
  `agents.entries.*.skills` alanını atlayan Agent’ların devraldığı paylaşılan temel
  izin listesi. Skills öğelerini varsayılan olarak kısıtlamamak için bu alanı
  tamamen atlayın.
</ParamField>

<ParamField path="agents.entries.*.skills" type="string[]">
  İlgili Agent için açıkça belirtilmiş nihai Skills kümesi. Açık listeler,
  devralınan varsayılanların **yerini alır** — bunlarla birleştirilmez.
  İlgili Agent’a hiçbir Skills öğesi sunmamak için `[]` olarak
  ayarlayın.
</ParamField>

<Warning>
  Agent Skills izin listeleri; OpenClaw Skills keşfi, istemler, eğik çizgi
  komutu keşfi, sandbox eşitlemesi ve Skills anlık görüntüleri için görünürlük
  ve yükleme filtresidir. Kabuk zamanı yetkilendirme sınırı değildir. Bir Agent,
  ana makine `exec` çalıştırabiliyorsa bu kabuk, `~/.openclaw/skills/config/mcporter.json`
  gibi MCP istemci kayıtları da dahil olmak üzere yürütme kullanıcısının
  görebildiği harici istemcileri çalıştırabilir veya ana makine dosyalarını
  okuyabilir. Agent başına MCP yalıtımı için Skills izin listelerini sandbox/OS
  kullanıcısı yalıtımıyla birleştirin, ana makine exec erişimini reddedin veya
  sıkı bir izin listesiyle sınırlandırın ve MCP sunucusunda Agent başına kimlik
  bilgilerini tercih edin.
</Warning>

## Workshop (`skills.workshop`)

<ParamField path="skills.workshop.autonomous.enabled" type="boolean" default="false">
  `true` olduğunda OpenClaw, kalıcı düzeltmelerden bekleyen öneriler oluşturabilir
  ve sistem boştayken başarıyla tamamlanmış önemli çalışmaları inceleyebilir.
  Bu, uygun dönüşlerden sonra arka planda bir model çalıştırması ekleyebilir. Kullanıcı tarafından
  istenen skill oluşturma ve `/learn`, ayar `false` olduğunda çalışmaya devam eder.
</ParamField>

Uygunluk, gizlilik, maliyet, yalnızca öneri izinleri ve sorun giderme için
[Kendi kendine öğrenme](/tr/tools/self-learning) bölümüne bakın.

<ParamField path="skills.workshop.approvalPolicy" type='"pending" | "auto"' default='"auto"'>
  `auto`, ek bir onay istemi olmadan agent tarafından başlatılan uygulama,
  reddetme veya karantinaya alma işlemlerine izin verir. `pending` operatör onayı gerektirir.
</ParamField>

<ParamField path="skills.workshop.allowSymlinkTargetWrites" type="boolean" default="false">
  Skill Workshop uygulamasının, gerçek hedefi `skills.load.allowSymlinkTargets` tarafından zaten
  güvenilir kabul edilen çalışma alanı skill sembolik bağlantıları üzerinden yazmasına izin verir.
  Oluşturulan önerilerin uygulanması bu paylaşılan skill kökünü değiştirmeyecekse
  bu ayarı devre dışı tutun.
</ParamField>

<ParamField path="skills.workshop.maxPending" type="number" default="50">
  Çalışma alanı başına saklanan bekleyen ve karantinaya alınmış önerilerin
  azami sayısı (izin verilen aralık: 1-200).
</ParamField>

<ParamField path="skills.workshop.maxSkillBytes" type="number" default="40000">
  Bayt cinsinden azami öneri gövdesi boyutu (izin verilen aralık: 1024-200000).
  Öneri açıklamaları, keşif ve listeleme çıktısında göründükleri için ayrıca
  kesin olarak 160 baytla sınırlandırılır.
</ParamField>

Bu yapılandırmanın denetlediği öneri yaşam döngüsü, CLI komutları, agent aracı
parametreleri ve Gateway yöntemleri için [Skill Workshop](/tr/tools/skill-workshop) bölümüne bakın.

## Sembolik bağlantılı skill kökleri

Varsayılan olarak çalışma alanı, proje agent'ı, ek dizin ve paketlenmiş skill
kökleri kapsama sınırlarıdır. `<workspace>/skills` altında bulunan ve kökün
dışına çözümlenen sembolik bağlantılı bir skill klasörü, bir günlük mesajıyla atlanır.

Kasıtlı bir sembolik bağlantı düzenine izin vermek için güvenilir hedefi bildirin:

```json5
{
  skills: {
    load: {
      extraDirs: ["~/Projects/manager/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
  },
}
```

Bu yapılandırmayla `<workspace>/skills/manager -> ~/Projects/manager/skills`, realpath çözümlemesinden sonra kabul edilir.
`extraDirs` kardeş depoyu doğrudan tarar; `allowSymlinkTargets` mevcut
düzenler için sembolik bağlantılı yolu korur.

Skill Workshop uygulaması varsayılan olarak bu sembolik bağlantılar üzerinden
yazmaz. Workshop uygulamasının, zaten güvenilir olan sembolik bağlantı
hedefleri altındaki skill'leri değiştirmesine izin vermek için ayrıca etkinleştirin:

```json5
{
  skills: {
    load: {
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    workshop: {
      allowSymlinkTargetWrites: true,
    },
  },
}
```

Yönetilen `~/.openclaw/skills` ve kişisel `~/.agents/skills` dizinleri, skill
dizini sembolik bağlantılarını zaten koşulsuz olarak kabul eder (skill başına
`SKILL.md` kapsam denetimi yine uygulanır) — `allowSymlinkTargets` yalnızca
çalışma alanı, ek dizin ve proje agent'ı (`<workspace>/.agents/skills`) kökleri için gereklidir.

## Korumalı alandaki skill'ler ve ortam değişkenleri

<Warning>
  `skills.entries.<skill>.env` ve `apiKey` yalnızca **ana makine** çalıştırmalarına
  uygulanır. Korumalı alan içinde hiçbir etkileri yoktur — `GEMINI_API_KEY`
  bağımlılığı olan bir skill, değişken korumalı alana ayrıca verilmedikçe
  `apiKey not configured` ile başarısız olur.
</Warning>

Gizli bilgileri bir Docker korumalı alanına şu şekilde aktarın:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          env: { GEMINI_API_KEY: "your-key-here" },
        },
      },
    },
  },
}
```

<Note>
  Docker daemon erişimi olan kullanıcılar, `sandbox.docker.env` değerlerini
  Docker meta verileri üzerinden inceleyebilir. Bu ifşa kabul edilebilir değilse
  bağlanmış bir gizli bilgi dosyası, özel bir imaj veya başka bir aktarım yolu kullanın.
</Note>

## Yükleme sırası hatırlatması

```text
workspace/skills      (en yüksek)
workspace/.agents/skills
~/.agents/skills
~/.openclaw/skills
paketlenmiş skill'ler
skills.load.extraDirs (en düşük)
```

İzleyici etkin olduğunda skill ve yapılandırma değişiklikleri bir sonraki yeni
oturumda veya izleyici bir değişiklik algıladığında bir sonraki agent dönüşünde
geçerli olur.

## İlgili konular

<CardGroup cols={2}>
  <Card title="Skills referansı" href="/tr/tools/skills" icon="puzzle-piece">
    Skill'lerin ne olduğu, yükleme sırası, geçit denetimi ve SKILL.md biçimi.
  </Card>
  <Card title="Skill oluşturma" href="/tr/tools/creating-skills" icon="hammer">
    Özel çalışma alanı skill'leri yazma.
  </Card>
  <Card title="Skill Workshop" href="/tr/tools/skill-workshop" icon="flask">
    Agent tarafından taslak hâline getirilen skill'ler için öneri kuyruğu.
  </Card>
  <Card title="Kendi kendine öğrenme" href="/tr/tools/self-learning" icon="brain">
    Tamamlanan çalışmalardan oluşturulan temkinli ve isteğe bağlı öneriler.
  </Card>
  <Card title="Eğik çizgi komutları" href="/tr/tools/slash-commands" icon="terminal">
    Yerel eğik çizgi komutları kataloğu ve sohbet yönergeleri.
  </Card>
</CardGroup>
