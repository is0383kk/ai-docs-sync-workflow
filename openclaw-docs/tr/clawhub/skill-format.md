---
read_when:
    - Skills yayımlama
    - Yayımlama hatalarında hata ayıklama
summary: Skill klasörü biçimi, gerekli dosyalar, destekleyici yapıtlar, sınırlar.
x-i18n:
    generated_at: "2026-07-26T23:12:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fdf16a589b8961ccd9181a53a9fa92a358952b9147d22eaf977f23e0b4b4d653
    source_path: clawhub/skill-format.md
    workflow: 16
---

# Skill biçimi

## Diskte

Bir skill, bir klasördür.

Gerekli:

- `SKILL.md` (veya `skill.md`; eski `skills.md` de kabul edilir)

İsteğe bağlı:

- destekleyici herhangi bir normal dosya (bkz. “Skill dosyaları”)
- `.clawhubignore` (yayımlama için yoksayma kalıpları, eski `.clawdhubignore`)
- `.gitignore` (bu da dikkate alınır)

## GitHub içe aktarma

Web GitHub içe aktarıcısı, yerel yayımlama/eşitlemeden daha katıdır. Yalnızca
oturum açmış GitHub hesabının sahip olduğu herkese açık, fork olmayan depolardaki
`SKILL.md` veya eski `skills.md` dosyalarını keşfeder. Özel depoları, fork'ları,
arşivlenmiş/devre dışı bırakılmış depoları veya üçüncü taraf herkese açık depoları içe aktarmaz.

Yerel kurulum meta verileri (CLI tarafından yazılır):

- `<skill>/.clawhub/origin.json` (eski `.clawdhub`)

Çalışma dizini kurulum durumu (CLI tarafından yazılır):

- `<workdir>/.clawhub/lock.json` (eski `.clawdhub`)

## `SKILL.md`

- İsteğe bağlı YAML frontmatter içeren Markdown.
- Sunucu, yayımlama sırasında meta verileri frontmatter'dan çıkarır.
- `description`, kullanıcı arayüzünde/aramada skill özeti olarak kullanılır.

Taşınabilir Agent Skills için `name`, üst dizinle eşleşmeli ve
1–64 küçük harf, rakam veya kısa çizgiden oluşmalıdır. ClawHub, yönlendirilebilir slug'ı ve
katalog görüntüleme adını ayrı tuttuğundan diğer istemcilerdeki mevcut adlar
yayımlanabilir durumda kalır ve sessizce yeniden yazılmaz. Katalog listeleri, saklanan adı
değiştirmeden uzun adları görsel olarak kısaltabilir.

## Frontmatter meta verileri

Skill meta verileri, `SKILL.md` dosyanızın üst kısmındaki YAML frontmatter içinde bildirilir. Bu, kayıt sistemine (ve güvenlik analizine) skill'inizin çalışmak için nelere ihtiyaç duyduğunu bildirir.

### Temel frontmatter

```yaml
---
name: my-skill
description: Bu skill'in ne yaptığının kısa özeti.
version: 1.0.0
---
```

### Çalışma zamanı meta verileri (`metadata.openclaw`)

Skill'inizin çalışma zamanı gereksinimlerini `metadata.openclaw` altında bildirin (diğer adlar: `metadata.clawdbot`, `metadata.clawdis`).

```yaml
---
name: my-skill
description: Todoist API aracılığıyla görevleri yönetin.
metadata:
  openclaw:
    requires:
      env:
        - TODOIST_API_KEY
      bins:
        - curl
    primaryEnv: TODOIST_API_KEY
---
```

Skill çalışmadan önce mevcut olması gereken ortam değişkenleri için `requires.env` kullanın. `required: false` ile isteğe bağlı değişkenler dâhil, değişken başına meta verilere ihtiyaç duyduğunuzda `envVars` kullanın.

### Tam alan referansı

| Alan               | Tür        | Açıklama                                                                                                                                     |
| ------------------ | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `requires.env`     | `string[]` | Skill'inizin beklediği gerekli ortam değişkenleri.                                                                                            |
| `requires.bins`    | `string[]` | Tümünün kurulu olması gereken CLI ikili dosyaları.                                                                                            |
| `requires.anyBins` | `string[]` | En az birinin mevcut olması gereken CLI ikili dosyaları.                                                                                      |
| `requires.config`  | `string[]` | Skill'inizin okuduğu yapılandırma dosyası yolları.                                                                                            |
| `primaryEnv`       | `string`   | Skill'inizin ana kimlik bilgisi ortam değişkeni.                                                                                              |
| `envVars`          | `array`    | `name`, isteğe bağlı `required` ve isteğe bağlı `description` içeren ortam değişkeni bildirimleri. İsteğe bağlı ortam değişkenleri için `required: false` ayarlayın. |
| `always`           | `boolean`  | `true` ise skill her zaman etkindir (açık bir kurulum gerekmez).                                                                             |
| `skillKey`         | `string`   | Skill'in çağırma anahtarını geçersiz kılar.                                                                                                   |
| `emoji`            | `string`   | Skill için görüntülenen emoji.                                                                                                                |
| `homepage`         | `string`   | Skill'in ana sayfasının veya belgelerinin URL'si.                                                                                             |
| `os`               | `string[]` | İşletim sistemi kısıtlamaları (ör. `["macos"]`, `["linux"]`).                                                                                 |
| `install`          | `array`    | Bağımlılıklar için kurulum belirtimleri (aşağıya bakın).                                                                                       |
| `nix`              | `object`   | Nix Plugin belirtimi (README'ye bakın).                                                                                                       |
| `config`           | `object`   | Clawdbot yapılandırma belirtimi (README'ye bakın).                                                                                             |

### Kurulum belirtimleri

Skill'inizin bağımlılıkların kurulmasına ihtiyacı varsa bunları `install` dizisinde bildirin:

```yaml
metadata:
  openclaw:
    install:
      - kind: brew
        formula: jq
        bins: [jq]
      - kind: node
        package: typescript
        bins: [tsc]
```

Desteklenen kurulum türleri: `brew`, `node`, `go`, `uv`.

### İsteğe bağlı ortam değişkenleri

İsteğe bağlı ortam değişkenlerini `metadata.openclaw.envVars` altında bildirin ve `required: false` ayarlayın. `requires.env`, skill'in bunlar olmadan çalışamayacağı anlamına geldiğinden `requires.env` içine isteğe bağlı girdiler eklemeyin.

```yaml
metadata:
  openclaw:
    primaryEnv: TODOIST_API_KEY
    envVars:
      - name: TODOIST_API_KEY
        required: true
        description: Kimliği doğrulanmış istekler için kullanılan Todoist API token'ı.
      - name: TODOIST_PROJECT_ID
        required: false
        description: Kullanıcı bir proje belirtmediğinde kullanılacak isteğe bağlı varsayılan proje kimliği.
```

### Bunun önemi

ClawHub'ın güvenlik analizi, skill'inizin bildirdikleriyle gerçekte yaptıklarının eşleşip eşleşmediğini denetler. Kodunuz `TODOIST_API_KEY` öğesine başvuruyor ancak frontmatter'ınız bunu `requires.env`, `primaryEnv` veya `envVars` altında bildirmiyorsa analiz bir meta veri uyuşmazlığını işaretler. Bildirimleri doğru tutmak, skill'inizin incelemeden geçmesine ve kullanıcıların ne kurduklarını anlamasına yardımcı olur.

### Örnek: eksiksiz frontmatter

```yaml
---
name: todoist-cli
description: Todoist görevlerini, projelerini ve etiketlerini komut satırından yönetin.
version: 1.2.0
metadata:
  openclaw:
    requires:
      env:
        - TODOIST_API_KEY
      bins:
        - curl
    primaryEnv: TODOIST_API_KEY
    envVars:
      - name: TODOIST_API_KEY
        required: true
        description: Todoist API token'ı.
      - name: TODOIST_PROJECT_ID
        required: false
        description: İsteğe bağlı varsayılan proje kimliği.
    emoji: "\u2705"
    homepage: https://github.com/example/todoist-cli
---
```

## Skill dosyaları

Yayımlama, uzantıdan bağımsız olarak skill klasöründeki tüm normal dosyaları kabul eder. Yoksayılan dosyalar,
gizli yollar, sembolik bağlantılar, macOS meta verileri ve sunucu tarafı boyut sınırları uygulanmaya devam eder.

- Geçerli UTF-8 içeren sınırlı boyuttaki dosyalar, kaçış karakterli düz metin olarak önizlenebilir ve
  sınırlı metin analizine dâhil edilir.
- Diğer dosyalar tam baytlarını korur ve indirilebilir.
- Güvenlik tarayıcıları saklanan yapıtın tamamını alır; metin algılama, yükleme izin listesi değil,
  görüntüleme ve analiz konusudur.

Sınırlar (sunucu tarafı):

- Toplam paket boyutu: 50MB.
- Gömme metni, `SKILL.md` + yaklaşık 40'a kadar sınırlı boyutta UTF-8 dosyası içerir (mümkün olan en iyi sınır).

## Slug'lar

- Varsayılan olarak klasör adından türetilir.
- Paket kapsamları, ClawHub yayımcı tanıtıcısıyla tam olarak eşleşmelidir. Yayımcı tanıtıcıları küçük harfler, rakamlar, kısa çizgiler, noktalar ve alt çizgiler kullanabilir; küçük harf veya rakamla başlayıp bitmelidir.
- Paket slug'ları küçük harfli ve npm açısından güvenli olmalıdır; örneğin `@example.tools/demo-plugin` veya `demo-plugin`.

## Sürüm oluşturma + etiketler

- Her yayımlama yeni bir sürüm oluşturur (semver).
- Etiketler bir sürüme işaret eden dizelerdir; `latest` yaygın olarak kullanılır.

## Lisans

- ClawHub'da yayımlanan tüm skill'ler `MIT-0` kapsamında lisanslanır.
- Herkes yayımlanmış skill'leri ticari amaçlar da dâhil olmak üzere kullanabilir, değiştirebilir ve yeniden dağıtabilir.
- Atıf gerekli değildir.
- `SKILL.md` içine çelişen lisans koşulları eklemeyin; ClawHub skill başına lisans geçersiz kılmalarını desteklemez.

## Ücretli skill'ler

- ClawHub ücretli skill'leri, skill başına fiyatlandırmayı, ödeme duvarlarını veya gelir paylaşımını desteklemez.
- `SKILL.md` içine fiyatlandırma meta verileri eklemeyin; bu, skill biçiminin bir parçası değildir ve yayımlanan bir skill'i ücretli hâle getirmez.
- Skill'iniz ücretli bir üçüncü taraf hizmetiyle entegre oluyorsa harici maliyeti ve gerekli hesabı skill talimatlarında ve ortam bildirimlerinde açıkça belgeleyin (gerekli değişkenler için `requires.env` veya isteğe bağlı değişkenler için `required: false` ile `envVars`).
