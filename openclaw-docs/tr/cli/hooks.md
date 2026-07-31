---
read_when:
    - Ajan kancalarını yönetmek istiyorsunuz
    - Hook kullanılabilirliğini incelemek veya çalışma alanı hook'larını etkinleştirmek istiyorsunuz
summary: '`openclaw hooks` (ajan hook''ları) için CLI referansı'
title: Kancalar
x-i18n:
    generated_at: "2026-07-26T23:52:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d4d58ea2270cf5122018f7be2943401229929f48f448b15fdd126d1cc99e1e56
    source_path: cli/hooks.md
    workflow: 16
---

# `openclaw hooks`

Ajan hook'larını (`/new`, `/reset` ve gateway başlatma gibi komutlara yönelik olay güdümlü otomasyonları) yönetin. Yalın `openclaw hooks`, `openclaw hooks list` ile eşdeğerdir.

İlgili: [Hook'lar](/tr/automation/hooks) - [Plugin hook'ları](/tr/plugins/hooks)

## Hook'ları listeleme

```bash
openclaw hooks list [--eligible] [--json] [-v|--verbose]
```

Çalışma alanı, yönetilen, ek ve paketlenmiş dizinlerde keşfedilen hook'ları listeler.

- `--eligible`: yalnızca gereksinimleri karşılanan hook'lar.
- `--json`: yapılandırılmış çıktı.
- `-v, --verbose`: karşılanmamış gereksinimleri içeren bir Missing sütunu ekler.

```
Hook'lar (4/5 hazır)

Hazır:
  🚀 boot-md ✓ - Gateway başlatılırken BOOT.md dosyasını çalıştır
  📎 bootstrap-extra-files ✓ - Ajan önyüklemesi sırasında ek çalışma alanı önyükleme dosyalarını ekle
  📝 command-logger ✓ - Tüm komut olaylarını merkezi bir denetim dosyasına kaydet
  💾 session-memory ✓ - /new veya /reset komutu verildiğinde oturum bağlamını belleğe kaydet
```

## Hook bilgilerini alma

```bash
openclaw hooks info <name> [--json]
```

`<name>`, hook adı veya hook anahtarıdır (örneğin `session-memory`). Kaynağı, dosya/işleyici yollarını, ana sayfayı, olayları ve gereksinim başına durumu (ikili dosyalar, ortam, yapılandırma, işletim sistemi) gösterir.

## Uygunluğu denetleme

```bash
openclaw hooks check [--json]
```

Hazır/hazır değil sayı özetini yazdırır; hazır olmayan hook'lar varsa her birini engelleyen nedenle birlikte listeler.

## Bir hook'u etkinleştirme

```bash
openclaw hooks enable <name>
```

Yapılandırmadaki `hooks.internal.entries.<name>.enabled = true` öğesini ekler/günceller ve ayrıca `hooks.internal.enabled` ana anahtarını açar (en az bir hook yapılandırılana kadar gateway hiçbir dahili hook işleyicisini yüklemez). Hook mevcut değilse, Plugin tarafından yönetiliyorsa veya uygun değilse (gereksinimleri eksikse) başarısız olur.

Plugin tarafından yönetilen hook'lar `hooks list` içinde `plugin:<id>` gösterir ve buradan etkinleştirilemez/devre dışı bırakılamaz; bunun yerine sahibi olan Plugin'i etkinleştirin veya devre dışı bırakın.

Hook'ları yeniden yüklemesi için etkinleştirdikten sonra gateway'i yeniden başlatın (macOS menü çubuğu uygulamasını veya geliştirme ortamındaki gateway işleminizi yeniden başlatın).

## Bir hook'u devre dışı bırakma

```bash
openclaw hooks disable <name>
```

`hooks.internal.entries.<name>.enabled = false` olarak ayarlar. Ardından gateway'i yeniden başlatın.

## Hook paketlerini yükleme ve güncelleme

```bash
openclaw plugins install <package>        # varsayılan olarak npm
openclaw plugins install npm:<package>    # yalnızca npm
openclaw plugins install <package> --pin  # çözümlenen sürümü sabitle
openclaw plugins install <path>           # yerel dizin veya arşiv
openclaw plugins install -l <path>        # yerel dizini kopyalamak yerine bağla

openclaw plugins update <id>
openclaw plugins update --all
openclaw plugins update --dry-run
```

Hook paketleri birleşik Plugin yükleyicisi/güncelleyicisi aracılığıyla yüklenir; `openclaw hooks install` / `openclaw hooks update`, uyarı yazdırıp `plugins` komutlarına yönlendiren, kullanımdan kaldırılmış takma adlar olarak çalışmaya devam eder.

- Npm belirtimleri yalnızca kayıt defteriyle sınırlıdır: paket adı ve isteğe bağlı tam sürüm veya dist-tag. Git/URL/dosya belirtimleri ve semver aralıkları reddedilir. Bağımlılık yüklemeleri `--ignore-scripts` ile projeye yerel olarak çalışır.
- Yalın belirtimler ve `@latest` kararlı kanalda kalır; npm bir ön sürüm çözümlerse OpenClaw durur ve açıkça katılmanızı ister (`@beta`, `@rc` veya tam bir ön sürüm numarası).
- Desteklenen arşivler: `.zip`, `.tgz`, `.tar.gz`, `.tar`.
- `-l, --link`, yerel bir dizini kopyalamak yerine bağlar (`hooks.internal.load.extraDirs` öğesine ekler); bağlı hook paketleri çalışma alanı hook'ları değil, operatör tarafından yapılandırılmış bir dizindeki yönetilen hook'lardır.
- `--pin`, npm yüklemelerini paylaşılan SQLite durumunda tam çözümlenmiş bir `name@version` olarak kaydeder.
- Yükleme, paketi `~/.openclaw/hooks/<id>` konumuna kopyalar, hook'larını `hooks.internal.entries.*` altında etkinleştirir ve yükleme kaynağını paylaşılan SQLite durumuna kaydeder.
- Saklanan bütünlük karması artık getirilen yapıtla eşleşmiyorsa OpenClaw uyarır ve devam etmeden önce onay ister; istemi atlamak için genel `--yes` seçeneğini kullanın (örneğin CI'da).

## Paketlenmiş hook'lar

| Hook                  | Olaylar                                            | İşlevi                                                                                       |
| --------------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| boot-md               | `gateway:startup`                                 | Yapılandırılmış her ajan kapsamı için gateway başlatılırken `BOOT.md` dosyasını çalıştırır                                  |
| bootstrap-extra-files | `agent:bootstrap`                                 | Ajan önyüklemesi sırasında ek önyükleme dosyaları (örneğin monorepo `AGENTS.md`/`TOOLS.md`) ekler |
| command-logger        | `command`                                         | Komut olaylarını `~/.openclaw/logs/commands.log` konumuna kaydeder                                             |
| compaction-notifier   | `session:compact:before`, `session:compact:after` | Oturum sıkıştırması başlayıp tamamlandığında görünür sohbet bildirimleri gönderir                             |
| session-memory        | `command:new`, `command:reset`                    | `/new` veya `/reset` sırasında oturum bağlamını belleğe kaydeder                                              |

Paketlenmiş herhangi bir hook'u `openclaw hooks enable <hook-name>` ile etkinleştirin. Tüm ayrıntılar, yapılandırma anahtarları ve varsayılanlar: [Paketlenmiş hook'lar](/tr/automation/hooks#bundled-hooks).

### command-logger günlük dosyası

```bash
tail -n 20 ~/.openclaw/logs/commands.log        # son komutlar
cat ~/.openclaw/logs/commands.log | jq .          # okunaklı biçimde yazdır
grep '"action":"new"' ~/.openclaw/logs/commands.log | jq .   # eyleme göre filtrele
```

## Notlar

- `hooks list --json`, `info --json` ve `check --json` yapılandırılmış JSON'u doğrudan standart çıktıya yazar.

## İlgili

- [CLI başvurusu](/tr/cli)
- [Otomasyon hook'ları](/tr/automation/hooks)
