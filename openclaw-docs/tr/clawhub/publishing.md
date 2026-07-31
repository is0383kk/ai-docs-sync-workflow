---
read_when:
    - Bir skill veya plugin yayımlama
    - Sahip veya paket kapsamı hatalarında hata ayıklama
    - Yayımlama kullanıcı arayüzü, CLI veya arka uç davranışı ekleme
summary: ClawHub yayımlama sürecinin Skills, Pluginler, sahipler, kapsamlar, sürümler ve inceleme için işleyişi.
x-i18n:
    generated_at: "2026-07-26T23:51:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 582dffaf4429e9f24d7c38f2809cc7dc05f8471e4ae2f9c6be60153cc8604e3f
    source_path: clawhub/publishing.md
    workflow: 16
---

# Yayımlama

Yayımlama, bir skill klasörünü veya plugin paketini seçtiğiniz sahip adına ClawHub'a gönderir. ClawHub, token'ınızın bu sahip adına yayımlama yapabildiğini denetler; meta verileri, adı, sürümü, dosyaları ve kaynak bilgilerini doğrular; ardından sürümü depolayıp otomatik güvenlik denetimlerini başlatır.

Doğrulama başarısız olursa hiçbir şey yayımlanmaz. Yeni sürümler, inceleme tamamlanana kadar normal kurulum ve indirme yüzeylerinde de yer almayabilir.

## Skills

En basit yayımlama yolu CLI'dır. Oturum açtıktan sonra yerel bir skill klasörünü yayımlayın:

```bash
clawhub login
clawhub skill publish ./my-skill \
  --slug my-skill \
  --name "My Skill" \
  --owner <owner>
```

Bir kuruluş sahibi adına yayımlarken `--owner <handle>` kullanın. Kimliği doğrulanmış kullanıcı olarak yayımlamak için bunu atlayın. Yayımlama, değişmemiş içeriği atlar. Yeni bir skill `1.0.0` sürümünden başlar ve sonraki değişiklikler bir sonraki yama sürümünü otomatik olarak yayımlar. Yalnızca açık bir sürüm belirtmeniz gerektiğinde `--version` iletin.

Katalog depoları için ClawHub'ın yeniden kullanılabilir [`skill-publish.yml` iş akışını](https://github.com/openclaw/clawhub/blob/main/.github/workflows/skill-publish.yml) kullanın. Bu iş akışı, `root` altındaki (varsayılan: `skills`) her bir doğrudan skill klasörü veya yalnızca `skill_path` olarak sağlanan klasör için `skill publish` çağırır.

```yaml
jobs:
  publish:
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@main
    with:
      owner: <owner>
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

Yayımlamadan yeni ve değiştirilmiş skill'leri önizlemek için `dry_run: true` kullanın.

## Pluginler

Pluginler npm tarzı paket adları kullanır. Kapsamlı paket adlarında sahip, adın ilk bölümünde yer alır:

```text
@owner/package-name
```

Kapsam, seçilen yayımlama sahibiyle eşleşmelidir. Paketinizin adı `@openclaw/dronzer` ise yalnızca `@openclaw` olarak yayımlanabilir. `@vintageayu` olarak yayımlıyorsanız paketi `@vintageayu/dronzer` olarak yeniden adlandırın.

Bu, bir paketin yayımlayıcının denetlemediği bir kuruluş ad alanı üzerinde hak iddia etmesini önler.

ClawHub'da zaten talep edilmiş veya ayrılmış bir kuruluşun, markanın, paket kapsamının, sahip kullanıcı adının ya da ad alanının hak sahibiyseniz kamuya açık, hassas olmayan kanıtlarla bir [Kuruluş / Ad Alanı Talebi sorunu](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml) açın. Nelerin eklenmesi ve nelerin herkese açık sorunların dışında tutulması gerektiği için [Kuruluş ve Ad Alanı Talepleri](/clawhub/namespace-claims) sayfasına bakın.

### Plugin Yayımlamadan Önce

- Paket kapsamıyla eşleşen bir sahip seçin.
- `openclaw.plugin.json` ekleyin. Kod pluginleri ayrıca `openclaw.compat.pluginApi` ve `openclaw.build.openclawVersion` içeren `package.json` gerektirir.
- Ana sayfada ve plugin listesi sayfalarında özel bir plugin katalog simgesi göstermek için `openclaw.plugin.json` alanına herhangi bir HTTPS görsel URL'siyle `icon` ekleyin.
- Kaynak depo ve tam commit meta verilerini ekleyin veya bunların algılanabilmesi için CLI'ı GitHub destekli bir çalışma kopyasından kullanın.
- Yayımlamadan önce `clawhub package validate <source>` çalıştırın. Paket, manifest, SDK içe aktarımı veya yapıt bulguları için [Plugin doğrulama düzeltmeleri](/clawhub/plugin-validation-fixes) sayfasına bakın.
- Bir sürüm oluşturmadan önce `clawhub package publish <source> --dry-run` çalıştırın.
- Otomatik güvenlik denetimleri ve doğrulama tamamlanana kadar yeni sürümlerin herkese açık kurulum yüzeylerinde yer almamasını bekleyin.

### Paketler İçin Güvenilir Yayımlama

Paketler için güvenilir yayımlama iki adımda ayarlanır:

1. Paketi normal, manuel veya token ile kimlik doğrulanan `clawhub package publish` üzerinden bir kez yayımlayın. Bu işlem paket kaydını oluşturur ve güvenilir yayımlayıcı yapılandırmasını değiştirebilecek paket yöneticilerini belirler.
2. Bir paket yöneticisi GitHub Actions güvenilir yayımlayıcı yapılandırmasını ayarlar:

```bash
clawhub package trusted-publisher set @owner/package-name \
  --repository owner/repo \
  --workflow-filename package-publish.yml
```

Yapılandırma ayarlandıktan sonra gelecekteki desteklenen GitHub Actions yayımlamaları, depoda uzun ömürlü bir ClawHub token'ı saklamadan OIDC/güvenilir yayımlamayı kullanabilir. Yapılandırılan depo ve iş akışı dosya adı, GitHub Actions OIDC iddiasıyla eşleşmelidir. Ayrıca `--environment <name>` iletirseniz GitHub Actions ortam iddiası bu adla tam olarak eşleşmelidir.

ClawHub, güvenilir yayımlayıcı yapılandırması ayarlanırken yapılandırılan GitHub deposunu doğrular. Herkese açık depolar, herkese açık GitHub meta verileri üzerinden doğrulanabilir. Özel depolar, örneğin gelecekteki bir ClawHub GitHub App kurulumu veya başka bir yetkili GitHub entegrasyonu aracılığıyla ClawHub'ın söz konusu depoya GitHub erişimine sahip olmasını gerektirir.

Mevcut yeniden kullanılabilir paket yayımlama iş akışı, `id-token: write` mevcut olduğunda `workflow_dispatch` yayımlamaları için secretsız güvenilir yayımlamayı destekler. Etiket gönderimiyle yapılan gerçek yayımlamalar hâlâ `clawhub_token` gerektirir; bu nedenle etiket sürümleri, ilk yayımlamalar, güvenilmeyen paketler veya acil durum yayımlamaları için `CLAWHUB_TOKEN` kullanılabilir durumda tutun.

Yapılandırmayı incelemek veya kaldırmak için:

```bash
clawhub package trusted-publisher get @owner/package-name
clawhub package trusted-publisher delete @owner/package-name
```

Güvenilir yayımlayıcı yapılandırmasını silmek geri alma yoludur. Bir paket yöneticisi yapılandırmayı yeniden ayarlayana kadar gelecekte güvenilir yayımlama token'larının oluşturulmasını devre dışı bırakır.

## SSS

### Paket kapsamı seçilen sahiple eşleşmelidir

Paket kapsamı ile seçilen sahip eşleşmezse ClawHub yayımlamayı reddeder:

```text
"@openclaw" paket kapsamı, seçilen "@vintageayu" sahibiyle eşleşmelidir.
"@openclaw" olarak yayımlayın veya bu paketi "@vintageayu/dronzer" olarak yeniden adlandırın.
```

Düzeltmek için paket kapsamının adlandırdığı sahibi seçin veya kapsamı yayımlama yapabildiğiniz sahiple eşleşecek şekilde paketi yeniden adlandırın.

Paket adı zaten doğru kapsama sahipse ancak paketin sahibi yanlış yayımlayıcıysa bunun yerine sahipliği aktarın:

```sh
clawhub package transfer @opik/opik-openclaw --to opik
```

Paket veya skill aktarımını yalnızca hem mevcut sahibe hem de hedef yayımlayıcıya yönetici erişiminiz olduğunda kullanın. Paket aktarımı, yönetemediğiniz bir kapsamda yayımlama yapmanıza izin vermez.

Mevcut sahibe erişiminiz yoksa ancak kuruluşunuzun, projenizin veya markanızın ad alanının hak sahibi olduğuna inanıyorsanız personel incelemesi için kamuya açık, hassas olmayan kanıtlarla bir [Kuruluş / Ad Alanı Talebi sorunu](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml) açın. Başvuru yapmadan önce [Kuruluş ve Ad Alanı Talepleri](/clawhub/namespace-claims) sayfasına bakın.

Bu, kuruluş ad alanlarını korur. `@openclaw/dronzer` adlı bir paket `@openclaw` ad alanı üzerinde hak iddia eder; dolayısıyla bunu yalnızca `@openclaw` sahibine erişimi olan yayımlayıcılar yayımlayabilir.
