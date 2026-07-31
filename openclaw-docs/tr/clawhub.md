---
read_when:
    - ClawHub'ın ne olduğunu açıklama
    - Skills veya plugin arama, yükleme ya da güncelleme
    - Kayıt defterinde beceri veya plugin yayımlama
    - openclaw ve clawhub CLI akışları arasında seçim yapma
sidebarTitle: ClawHub
summary: Keşif, kurulum, yayımlama, güvenlik ve clawhub CLI için genel ClawHub tanıtımı.
title: ClawHub
x-i18n:
    generated_at: "2026-07-26T22:39:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fde96ccb410b84dc4d3a48d42bbdbc0a80ac11dfb053afac2ee9e7e9d1605a5b
    source_path: clawhub/index.md
    workflow: 16
---

# ClawHub

ClawHub, OpenClaw Skills ve Plugin'leri için herkese açık kayıt defteridir.

- Skills aramak, yüklemek ve güncellemek ve ClawHub'dan Plugin'ler yüklemek için yerel `openclaw` komutlarını kullanın.
- Kayıt defteri kimlik doğrulaması, yayımlama ve silme/silmeyi geri alma iş akışları için ayrı `clawhub` CLI'ını kullanın.

Site: [clawhub.ai](https://clawhub.ai)

## Hızlı başlangıç

OpenClaw ile Skills arayın ve yükleyin:

```bash
openclaw skills search "calendar"
openclaw skills install @openclaw/demo
openclaw skills update --all
```

OpenClaw ile Plugin'leri arayın ve yükleyin:

```bash
openclaw plugins search "calendar"
openclaw plugins install clawhub:<package>
openclaw plugins update --all
```

Yayımlama veya silme/silmeyi geri alma gibi kayıt defteri kimliği doğrulanmış iş akışlarını
kullanmak istediğinizde ClawHub CLI'ını yükleyin:

```bash
npm i -g clawhub
# veya
pnpm add -g clawhub
```

## ClawHub'ın barındırdıkları

| Yüzey          | Sakladıkları                                                  | Tipik komut                                  |
| -------------- | ------------------------------------------------------------- | -------------------------------------------- |
| Skills         | `SKILL.md` ve destekleyici dosyalardan oluşan sürümlendirilmiş metin paketleri | `openclaw skills install @openclaw/demo`     |
| Kod Plugin'leri | Uyumluluk meta verilerine sahip OpenClaw Plugin paketleri     | `openclaw plugins install clawhub:<package>` |
| Paket Plugin'leri | OpenClaw dağıtımı için paketlenmiş Plugin paketleri          | `clawhub package publish <source>`           |

ClawHub; semver sürümlerini, `latest` gibi etiketleri, değişiklik günlüklerini, dosyaları,
indirmeleri, yıldızları ve güvenlik taraması özetlerini izler. Herkese açık sayfalar, kullanıcıların
yüklemeden önce bir Skill veya Plugin'i inceleyebilmesi için güncel kayıt defteri durumunu gösterir.

## Yerel OpenClaw akışları

Yerel OpenClaw komutları etkin OpenClaw çalışma alanına yükleme yapar ve sonraki
güncelleme komutlarının ClawHub'da kalabilmesi için kaynak meta verilerini kalıcı hâle getirir.

Bir Plugin yüklemesinin ClawHub üzerinden çözümlenmesi gerektiğinde `clawhub:<package>` kullanın.
Yalın, npm açısından güvenli Plugin belirtimleri, sürüm geçişleri sırasında npm üzerinden çözümlenebilir;
bir kaynağın açıkça belirtilmesi gerektiğinde `npm:<package>` yalnızca npm ile çalışmaya devam eder.

Plugin yüklemeleri, arşiv yüklemesi çalıştırılmadan önce duyurulan `pluginApi` ve `minGatewayVersion`
uyumluluğunu doğrular. Bir paket sürümü ClawPack yapıtı yayımladığında OpenClaw, tam olarak yüklenen npm-pack
`.tgz` dosyasını tercih eder, ClawHub özet üstbilgisini ve indirilen baytları doğrular ve sonraki
güncellemeler için yapıt meta verilerini kaydeder.

## ClawHub CLI

ClawHub CLI, kayıt defteri kimliği doğrulanmış çalışmalar içindir:

```bash
clawhub login
clawhub whoami
clawhub search "postgres backups"
clawhub skill publish ./my-skill --slug my-skill --name "My Skill" --version 1.0.0
clawhub package explore --family code-plugin
clawhub package inspect episodic-claw
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

CLI ayrıca doğrudan kayıt defteri iş akışları için Skill yükleme/güncelleme komutlarına sahiptir:

```bash
clawhub install @openclaw/demo
clawhub update @openclaw/demo
clawhub update --all
clawhub list
```

Bu komutlar Skills'i geçerli çalışma dizini altındaki `./skills` konumuna yükler
ve yüklenen sürümleri `.clawhub/lock.json` içinde kaydeder.

## Yayımlama

Skills'i `SKILL.md` içeren yerel bir klasörden yayımlayın:

```bash
clawhub skill publish <path>
```

Yaygın yayımlama seçenekleri:

- `--slug <slug>`: yayımlanan Skill'in URL adı.
- `--name <name>`: görünen ad.
- `--version <version>`: semver sürümü.
- `--changelog <text>`: değişiklik günlüğü metni.
- `--tags <tags>`: varsayılanı `latest` olan, virgülle ayrılmış etiketler.

Plugin'leri yerel bir klasörden, `owner/repo`, `owner/repo@ref` veya bir GitHub
URL'sinden yayımlayın:

```bash
clawhub package publish <source>
```

Yükleme yapmadan tam yayımlama planını oluşturmak için `--dry-run`, CI uyumlu
çıktı için `--json` kullanın.

Kod Plugin'leri, `openclaw.compat.pluginApi` ve
`openclaw.build.openclawVersion` dahil olmak üzere `package.json` içinde gerekli OpenClaw uyumluluk meta verilerini
içermelidir. Tam komut başvurusu için [CLI](/tr/clawhub/cli), Skill meta verileri için
[Skill biçimi](/clawhub/skill-format) bölümüne bakın.

## Güvenlik ve moderasyon

ClawHub varsayılan olarak açıktır: herkes yükleme yapabilir, ancak yayımlama işlemi, yükleme
eşiğini geçecek kadar eski bir GitHub hesabı gerektirir. Herkese açık ayrıntı sayfaları, yükleme
veya indirme öncesinde en son tarama durumunu özetler.

ClawHub, yayımlanan Skills ve Plugin sürümlerinde otomatik kontroller çalıştırır. Tarama nedeniyle
bekletilen veya engellenen sürümler, `/dashboard` içinde sahiplerine görünür kalırken herkese açık
katalogdan ve yükleme yüzeylerinden kaybolabilir.

Oturum açmış kullanıcılar Skills'i ve paketleri bildirebilir. Moderatörler bildirimleri inceleyebilir,
içeriği gizleyebilir veya geri yükleyebilir ve kötüye kullanımda bulunan hesapları yasaklayabilir.
Politika ve yaptırım ayrıntıları için
[Güvenlik](/tr/clawhub/security),
[Güvenlik Denetimleri](/clawhub/security-audits),
[Moderasyon ve Hesap Güvenliği](/clawhub/moderation) ve
[Kabul edilebilir kullanım](/tr/clawhub/acceptable-usage) bölümlerine bakın.

## Telemetri ve ortam

Oturum açmışken `clawhub install` çalıştırdığınızda CLI, ClawHub'ın toplam yükleme sayılarını
hesaplayabilmesi için en iyi çaba ilkesiyle bir yükleme olayı gönderebilir. Bunu şu şekilde devre dışı bırakın:

```bash
export CLAWHUB_DISABLE_TELEMETRY=1
```

Yararlı ortam geçersiz kılmaları:

| Değişken                      | Etki                                              |
| ----------------------------- | ------------------------------------------------- |
| `CLAWHUB_SITE`                | Tarayıcı oturum açma işleminde kullanılan site URL'sini geçersiz kılar. |
| `CLAWHUB_REGISTRY`            | Kayıt defteri API URL'sini geçersiz kılar.        |
| `CLAWHUB_CONFIG_PATH`         | CLI'ın belirteç/yapılandırma durumunu sakladığı konumu geçersiz kılar. |
| `CLAWHUB_WORKDIR`             | Varsayılan çalışma dizinini geçersiz kılar.       |
| `CLAWHUB_DISABLE_TELEMETRY=1` | Yükleme telemetrisini devre dışı bırakır.         |

Daha ayrıntılı başvuru materyali için [Telemetri](/clawhub/telemetry), [HTTP API](/clawhub/http-api) ve
[Sorun giderme](/clawhub/troubleshooting) bölümlerine bakın.
