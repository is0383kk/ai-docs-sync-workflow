---
read_when:
    - Yerel olarak paketlenmiş bir Plugin ile ilk katılım veya kurulum akışlarını test etme
    - Bir plugin paketini yayımlamadan önce doğrulama
    - Otomatik Plugin kurulumunu bir test yapıtıyla değiştirme
sidebarTitle: Install overrides
summary: Paketlenmiş plugin geçersiz kılmalarını kurulum zamanı yükleme akışlarıyla test edin
title: Plugin yükleme geçersiz kılmaları
x-i18n:
    generated_at: "2026-07-26T23:26:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: adc823f49ea9f8fa86e6a89933e43fdc309d808ac24397770495dbe81cb4b0d7
    source_path: plugins/install-overrides.md
    workflow: 16
---

Plugin yükleme geçersiz kılmaları, bakım sorumlularının kurulum sırasındaki Plugin yüklemelerini katalog, paketle birlikte gelen veya varsayılan npm kaynağı yerine belirli bir npm paketine ya da yerel bir npm-pack tarball arşivine yönlendirmesine olanak tanır. Bunlar yalnızca E2E ve paket doğrulaması için mevcuttur; normal kullanıcılar Plugin'leri
[`openclaw plugins install`](/tr/cli/plugins) ile yükler.

<Warning>
Geçersiz kılmalar, sağladığınız kaynaktaki Plugin kodunu çalıştırır. Bunları yalnızca yalıtılmış bir durum dizininde veya tek kullanımlık bir test makinesinde kullanın.
</Warning>

## Ortam

Her iki değişken de ayarlanmadıkça geçersiz kılmalar devre dışıdır:

```bash
export OPENCLAW_ALLOW_PLUGIN_INSTALL_OVERRIDES=1
export OPENCLAW_PLUGIN_INSTALL_OVERRIDES='{
  "codex": "npm-pack:/tmp/openclaw-codex-2026.5.8.tgz",
  "openclaw-web-search": "npm:@openclaw/web-search@2026.5.8"
}'
```

Geçersiz kılma eşlemesi, anahtar olarak Plugin kimliklerini kullanan JSON biçimindedir. Değerler şunları destekler:

| Önek                  | Kaynak                                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------------ |
| `npm:<registry-spec>` | Kayıt defteri paketleri, tam sürümler veya etiketler                                              |
| `npm-pack:<path.tgz>` | `npm pack` tarafından üretilen yerel tarball arşivleri; göreli yollar geçerli çalışma dizininden çözümlenir |

## Davranış

Kurulum sırasındaki bir akış, kimliği eşlemede bulunan bir Plugin'i yüklediğinde OpenClaw katalog, paketle birlikte gelen veya varsayılan npm kaynağı yerine geçersiz kılma kaynağını kullanır. Bu, ilk katılım ve paylaşılan kurulum sırası Plugin yükleyicisini kullanan diğer tüm akışlar için geçerlidir.

- Geçersiz kılmalar beklenen Plugin kimliğini yine de zorunlu tutar: `codex` ile eşlenen bir tarball arşivi, manifest kimliği `codex` olan bir Plugin yüklemelidir.
- Geçersiz kılmalar, resmî güvenilir kaynak durumunu devralmaz. Katalog girdisi normalde OpenClaw'a ait bir paketi temsil etse bile geçersiz kılma, operatör tarafından sağlanan test girdisi olarak değerlendirilir.
- Çalışma alanı `.env` dosyaları yükleme geçersiz kılmalarını etkinleştiremez; her iki ortam değişkeni de engellenen çalışma alanı dotenv listesindedir. Bunları OpenClaw'ı başlatan güvenilir kabukta, CI işinde veya uzak test komutunda ayarlayın.

## Paket E2E

Paket yüklemelerinin ve yükleme kayıtlarının normal OpenClaw durumunuza dokunmaması için yalıtılmış bir durum dizini kullanın:

```bash
npm pack extensions/codex --pack-destination /tmp

OPENCLAW_STATE_DIR="$(mktemp -d)" \
OPENCLAW_ALLOW_PLUGIN_INSTALL_OVERRIDES=1 \
OPENCLAW_PLUGIN_INSTALL_OVERRIDES='{"codex":"npm-pack:/tmp/openclaw-codex-2026.5.8.tgz"}' \
pnpm openclaw onboard --mode local
```

Yüklenen paketi durum dizini altında doğrulayın:

```bash
find "$OPENCLAW_STATE_DIR/npm/projects" -path '*/node_modules/@openclaw/codex/package.json' -print
grep -R '"@openclaw/codex"' "$OPENCLAW_STATE_DIR/npm/projects"/*/package-lock.json
```

Canlı sağlayıcı E2E testi için test komutunu başlatmadan önce gerçek API anahtarını güvenilir bir kabuktan veya CI gizli bilgisinden yükleyin. Anahtarları yazdırmayın; yalnızca kaynağı ve anahtarın mevcut olup olmadığını bildirin.
