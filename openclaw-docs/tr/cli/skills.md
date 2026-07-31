---
read_when:
    - Hangi Skills öğelerinin kullanılabilir ve çalıştırılmaya hazır olduğunu görmek istiyorsunuz
    - ClawHub'da arama yapmak veya ClawHub, Git ya da yerel dizinlerden Skills yüklemek istiyorsunuz
    - ClawHub ile bir ClawHub skill'ini doğrulamak istiyorsunuz
    - Skills için eksik ikili dosyaların/ortam değişkenlerinin/yapılandırmanın hatalarını ayıklamak istiyorsunuz
summary: '`openclaw skills` için CLI referansı (arama/yükleme/güncelleme/doğrulama/listeleme/bilgi/denetleme/atölye)'
title: Skills
x-i18n:
    generated_at: "2026-07-26T23:16:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3eafd40704b666e6be185aa8148b60613c861a2899fb9b0cc3353212e8e4d678
    source_path: cli/skills.md
    workflow: 16
---

# `openclaw skills`

Yerel Skills öğelerini inceleyin, ClawHub'da arama yapın, ClawHub/Git/yerel
dizinlerden Skills yükleyin, ClawHub Skills öğelerini doğrulayın ve ClawHub tarafından izlenen yüklemeleri güncelleyin.

İlgili:

- Skills sistemi: [Skills](/tr/tools/skills)
- Skill Atölyesi: [Skill Atölyesi](/tr/tools/skill-workshop)
- Skills yapılandırması: [Skills yapılandırması](/tr/tools/skills-config)
- ClawHub yüklemeleri: [ClawHub](/tr/clawhub/cli)

## Komutlar

```bash
openclaw skills search "calendar"
openclaw skills search --limit 20 --json
openclaw skills install @owner/<slug>
openclaw skills install @owner/<slug> --version <version>
openclaw skills install git:owner/repo
openclaw skills install git:owner/repo@main
openclaw skills install ./path/to/skill --as custom-name
openclaw skills install @owner/<slug> --force
openclaw skills install @owner/<slug> --force-install
openclaw skills install @owner/<slug> --acknowledge-clawhub-risk
openclaw skills install @owner/<slug> --agent <id>
openclaw skills install @owner/<slug> --global
openclaw skills update @owner/<slug>
openclaw skills update @owner/<slug> --force-install
openclaw skills update @owner/<slug> --acknowledge-clawhub-risk
openclaw skills update @owner/<slug> --global
openclaw skills update --all
openclaw skills update --all --agent <id>
openclaw skills update --all --global
openclaw skills verify @owner/<slug>
openclaw skills verify @owner/<slug> --version <version>
openclaw skills verify @owner/<slug> --tag <tag>
openclaw skills verify @owner/<slug> --card
openclaw skills verify @owner/<slug> --global
openclaw skills list
openclaw skills list --eligible
openclaw skills list --json
openclaw skills list --verbose
openclaw skills list --agent <id>
openclaw skills info <name>
openclaw skills info <name> --json
openclaw skills info <name> --agent <id>
openclaw skills check
openclaw skills check --agent <id>
openclaw skills check --json
openclaw skills workshop propose-create --name "qa-check" --description "QA checklist" --proposal ./PROPOSAL.md
openclaw skills workshop propose-update qa-check --proposal ./PROPOSAL.md
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "Not reusable"
openclaw skills workshop quarantine <proposal-id> --reason "Needs security review"
```

`search`, `update` ve `verify` doğrudan ClawHub'ı kullanır. `install @owner/<slug>`
bir ClawHub Skill öğesini yükler, `install git:owner/repo[@ref]` bir Git Skill öğesini klonlar
ve `install ./path` yerel bir Skill dizinini kopyalar. Varsayılan olarak `install`,
`update` ve `verify`, etkin çalışma alanının `skills/` dizinini hedefler; 
`--global` ile paylaşılan yönetilen Skills dizinini hedefler. `list`/`info`/`check`
yine de geçerli çalışma alanı ve yapılandırma tarafından görülebilen yerel Skills öğelerini inceler.
Çalışma alanı destekli komutlar hedef çalışma alanını önce `--agent <id>` kaynağından,
ardından geçerli çalışma dizini yapılandırılmış bir ajan çalışma alanının içindeyse bu dizinden,
son olarak da varsayılan ajandan çözümler.

Git ve yerel dizin yüklemelerinde kaynak kökünde `SKILL.md` bulunması beklenir. Yükleme
kısa adı, geçerliyse `SKILL.md` frontmatter içindeki `name` değerinden, ardından
kaynak dizin veya depo adından alınır; bunu geçersiz kılmak için `--as <slug>` kullanın.
`--version` yalnızca ClawHub içindir. Skill yüklemeleri npm paket belirtimlerini
veya zip/arşiv yollarını desteklemez; `openclaw skills update` ise yalnızca ClawHub tarafından izlenen
yüklemeleri günceller.

İlk katılım veya Skills ayarlarından tetiklenen Gateway destekli Skill bağımlılığı yüklemeleri
bunun yerine ayrı `skills.install` istek yolunu kullanır.

Notlar:

| Bayrak/davranış                    | Açıklama                                                                                                                                                                                                                                                                       |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `search [query...]`              | İsteğe bağlı sorgu; varsayılan ClawHub arama akışına göz atmak için sorguyu belirtmeyin.                                                                                                                                                                                        |
| `search --limit <n>`             | Döndürülen sonuçların sayısını sınırlar.                                                                                                                                                                                                                                        |
| `install git:owner/repo[@ref]`   | Bir Git Skill öğesi yükler. Dal referansları `git:owner/repo@feature/foo` gibi eğik çizgiler içerebilir.                                                                                                                                                                                   |
| `install ./path/to/skill`        | Kökünde `SKILL.md` bulunan yerel bir dizini yükler.                                                                                                                                                                                                                             |
| `install --as <slug>`            | Git ve yerel dizin yüklemeleri için çıkarılan kısa adı geçersiz kılar.                                                                                                                                                                                                           |
| `install --version <version>`    | Yalnızca ClawHub Skill referanslarına uygulanır.                                                                                                                                                                                                                                 |
| `install --force`                | Aynı kısa ada sahip mevcut çalışma alanı Skill klasörünün üzerine yazar.                                                                                                                                                                                                         |
| `install/update --force-install` | ClawHub taraması tamamlanmadan önce beklemedeki GitHub destekli bir ClawHub Skill öğesini yükler.                                                                                                                                                                                |
| `--global`                       | Paylaşılan yönetilen Skills dizinini hedefler; `--agent <id>` ile birlikte kullanılamaz.                                                                                                                                                                                      |
| `--agent <id>`                   | Yapılandırılmış bir ajan çalışma alanını hedefler; geçerli çalışma dizini çıkarımını geçersiz kılar.                                                                                                                                                                             |
| `update @owner/<slug>`           | İzlenen tek bir Skill öğesini günceller. Çalışma alanı yerine paylaşılan yönetilen Skills dizinini hedeflemek için `--global` ekleyin.                                                                                                                                     |
| `update --all`                   | Seçilen çalışma alanındaki veya `--global` ile paylaşılan yönetilen Skills dizinindeki izlenen ClawHub yüklemelerini günceller.                                                                                                                                           |
| `verify @owner/<slug>`           | Varsayılan olarak ClawHub'ın `clawhub.skill.verify.v1` JSON zarfını yazdırır. JSON zaten varsayılan olduğundan `--json` bayrağı yoktur. Skill zaten yüklüyse veya belirsiz değilse uyumluluk amacıyla yalın kısa adlar kabul edilir; sahip nitelemeli referanslar yayıncı belirsizliğini önler. |
| `verify` kaynağı              | ClawHub sunucu tarafından çözümlenmiş kaynak kökeni döndürdüğünde, doğrulama JSON'u sabitlenmiş bir commit içeren `openclaw.verifiedSourceUrl` değerini de içerir. Kullanılamayan veya kaynağın kendisi tarafından beyan edilen kaynak URL'leri yalnızca ham köken zarfında kalır ve üst düzeye çıkarılmaz. |
| `verify` sürüm seçicisi        | `verify`, yüklü ClawHub Skills öğeleri için `.clawhub/origin.json` kullanır; böylece yüklü sürümü geldiği kayıt defterine göre doğrular. `--version` ve `--tag` sürüm seçicisini geçersiz kılar ancak kaynak meta verileri mevcut olduğunda yüklü kayıt defterini korur. |
| `verify --card`                  | JSON yerine oluşturulan Skill Kartı Markdown'ını yazdırır. ClawHub `ok: false` veya `decision: "fail"` döndürdüğünde sıfırdan farklı bir kodla çıkar; ClawHub politikası değişmediği sürece imzasız imzalar yalnızca bilgilendirme amaçlıdır. |
| Skill Kartı parmak izi           | Yüklü ClawHub paketleri oluşturulmuş bir `skill-card.md` içerebilir. OpenClaw, doğrulamayı bir ClawHub sunucu kararı olarak ele alır ve yalnızca oluşturulan kart paket parmak izini değiştirdiği için yüklü bir Skill öğesini reddetmez. |
| `check --agent <id>`             | Seçilen ajanın çalışma alanını denetler ve hangi hazır Skills öğelerinin o ajanın isteminde veya komut yüzeyinde gerçekten görünür olduğunu bildirir. |
| `list`                           | Alt komut sağlanmadığında varsayılan eylemdir. |
| `list`/`info`/`check` çıktısı     | İşlenmiş çıktı stdout'a gider. `--json` ile makine tarafından okunabilir yük, kanallar ve betikler için stdout'ta kalır. |

Topluluk ClawHub Skill yüklemeleri ve güncellemeleri, indirmeden önce güveni
kontrol eder. Sürümlendirilmiş topluluk arşivi yayınları, tam sürüm güven meta
verilerini kullanır. Çözümleyici destekli GitHub Skills öğeleri, sabitlenmiş bir
commit döndürmeden önce tarama ve zorunlu yükleme politikasını uygulamak için
ClawHub'ın yükleme çözümleyicisine dayanır; tarama tamamlanmadan önce beklemedeki
GitHub destekli bir Skill öğesini yüklemek için `--force-install` kullanın.
Kötü amaçlı veya engellenmiş topluluk yayınları reddedilir. Riskli topluluk
yayınlarında inceleme gerekir; etkileşimsiz bir komutun bu incelemeden sonra
devam etmesi gerektiğinde `--acknowledge-clawhub-risk` gerekir. Resmî ClawHub Skill
yayıncıları ve paketlenmiş OpenClaw Skill kaynakları bu yayın güveni istemini
atlar.

## Skill Atölyesi

`openclaw skills workshop`, seçili çalışma alanındaki bekleyen skill önerilerini
yönetir. Öneriler uygulanana kadar etkin skill değildir. Öneri
depolama, destek dosyası güvenlik önlemleri, Gateway yöntemleri ve onay politikası için
[Skill Atölyesi](/tr/tools/skill-workshop) bölümüne bakın.

```bash
openclaw skills workshop propose-create \
  --name "qa-check" \
  --description "Tekrarlanabilir QA kontrol listesi" \
  --proposal ./PROPOSAL.md
openclaw skills workshop propose-create \
  --name "qa-check" \
  --description "Tekrarlanabilir QA kontrol listesi" \
  --proposal-dir ./qa-check-proposal
openclaw skills workshop propose-update qa-check --proposal ./PROPOSAL.md
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "Yinelenen"
openclaw skills workshop quarantine <proposal-id> --reason "Güvenlik incelemesi gerekiyor"
```

`propose-create`, `propose-update` ve `revise` ayrıca önerinin motivasyonunu ve destekleyici
notları `--proposal`/`--proposal-dir` içeriğiyle birlikte kaydetmek için `--goal <text>`
ve `--evidence <text>` seçeneklerini de kabul eder.

## İlgili

- [CLI referansı](/tr/cli)
- [Skills](/tr/tools/skills)
