---
read_when:
    - Aynı makinede birden fazla Gateway çalıştırma
    - Gateway başına yalıtılmış yapılandırma/durum/bağlantı noktaları gerekir
summary: Tek bir ana makinede birden fazla OpenClaw Gateway çalıştırma (yalıtım, bağlantı noktaları ve profiller)
title: Birden çok Gateway
x-i18n:
    generated_at: "2026-07-26T23:20:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 655fa865a98064d7c017a7c2eb08ea9a9683002d96a3dbe45a8c16cbd3c86ba1
    source_path: gateway/multiple-gateways.md
    workflow: 16
---

Çoğu kurulum için tek bir Gateway gerekir; tek bir Gateway, birden fazla mesajlaşma bağlantısını ve ajanı yönetir. Yalnızca daha güçlü yalıtım veya yedeklilik gerektiğinde (ör. bir kurtarma botu) yalıtılmış profillere/portlara sahip ayrı Gateway'ler çalıştırın.

## Kurtarma botu hızlı başlangıcı

En basit kurtarma botu kurulumu:

- Ana botu varsayılan profilde tutun.
- Kurtarma botunu kendi Telegram bot token'ıyla `--profile rescue` üzerinde çalıştırın.
- Kurtarma botunu farklı bir temel porta, ör. `19789` portuna yerleştirin.

Bu, birincil bot çalışmıyorsa kurtarma botunun hata ayıklayabilmesini veya yapılandırma değişiklikleri uygulayabilmesini sağlar. Türetilen tarayıcı/CDP portlarının hiçbir zaman çakışmaması için temel portlar arasında en az 20 port bırakın.

```bash
# Kurtarma botu (ayrı Telegram botu, ayrı profil, port 19789)
openclaw --profile rescue onboard
openclaw --profile rescue gateway install --port 19789
```

Ana botunuz zaten çalışıyorsa genellikle gerekenlerin tümü budur. İlk katılım kurtarma hizmetini zaten yüklediyse son `gateway install` adımını atlayın.

`openclaw --profile rescue onboard` sırasında:

- Kurtarma hesabına ayrılmış ayrı bir Telegram bot token'ı kullanın (yalnızca operatörlere açık tutması kolaydır, ana botun kanal/uygulama kurulumundan bağımsızdır ve DM tabanlı basit bir kurtarma yolu sağlar).
- `rescue` profil adını koruyun.
- Ana botunkinden en az 20 daha yüksek bir temel port kullanın.
- Zaten kendiniz yönettiğiniz bir çalışma alanı yoksa varsayılan kurtarma çalışma alanını kabul edin.

### `--profile rescue onboard` neleri değiştirir?

`--profile rescue onboard`, normal ilk katılım akışını çalıştırır ancak her şeyi ayrı bir profile yazar; böylece kurtarma botu kendine ait şunlara sahip olur:

- Profil/yapılandırma dosyası
- Durum dizini
- Çalışma alanı (varsayılan: `~/.openclaw/workspace-rescue`)
- Yönetilen hizmet adı
- Temel port (ve türetilen portlar)
- Telegram bot token'ı

Bunun dışındaki istemler normal ilk katılımla aynıdır.

## Genel çoklu Gateway kurulumu

Aynı yalıtım düzeni, tek bir ana makinedeki herhangi bir Gateway çifti veya grubu için geçerlidir; her ek Gateway'e kendi adlandırılmış profilini ve temel portunu verin:

```bash
# ana (varsayılan profil)
openclaw setup
openclaw gateway --port 18789

# ek gateway
openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

Her iki tarafta adlandırılmış profiller de kullanılabilir:

```bash
openclaw --profile main setup
openclaw --profile main gateway --port 18789

openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

Hizmetler de aynı düzeni izler:

```bash
openclaw gateway install
openclaw --profile ops gateway install --port 19789
```

Yedek operatör hattı için kurtarma botu hızlı başlangıcını; farklı kanallar, kiracılar, çalışma alanları veya operasyonel roller genelinde uzun süre çalışan birden fazla Gateway için genel profil düzenini kullanın.

## Yalıtım kontrol listesi

Her Gateway örneği için şunları benzersiz tutun:

| Ayar                         | Amaç                                        |
| ---------------------------- | ------------------------------------------- |
| `OPENCLAW_CONFIG_PATH`       | Örneğe özgü yapılandırma dosyası            |
| `OPENCLAW_STATE_DIR`         | Örneğe özgü oturumlar, kimlik bilgileri, önbellekler |
| `agents.defaults.workspace`  | Örneğe özgü çalışma alanı kökü              |
| `gateway.port` (veya `--port`) | Her örnek için benzersiz                    |
| Türetilen tarayıcı/CDP portları | Aşağıya bakın                           |

Bunlardan herhangi birinin paylaşılması yapılandırma, durum veya port çakışmalarına neden olur. Gateway başlangıcı,
`OPENCLAW_ALLOW_MULTI_GATEWAY=1` yapılandırma başına tek örnek kuralını atlasa bile
durum dizini sahipliğinin benzersiz olmasını zorunlu kılar.

## Port eşlemesi (türetilen)

Temel port = `gateway.port` (veya `OPENCLAW_GATEWAY_PORT` / `--port`).

- Tarayıcı denetim hizmeti portu = temel + 2 (yalnızca geri döngü).
- Canvas ana makinesi doğrudan Gateway HTTP sunucusunda sunulur (`gateway.port` ile aynı port).
- Tarayıcı profili CDP portları, `browser control port + 9` ile `+ 108` arasından otomatik olarak ayrılır.

Bunlardan herhangi birini yapılandırmada veya ortam değişkenlerinde geçersiz kılarsanız her örnek için benzersiz tutmanız gerekir.

## Tarayıcı/CDP notları (yaygın tuzak)

- `browser.cdpUrl` değerini birden fazla örnekte aynı değere **sabitlemeyin**.
- Her örnek için kendi tarayıcı denetim portu ve CDP aralığı (Gateway portundan türetilir) gerekir.
- Açık CDP portları için her örnekte `browser.profiles.<name>.cdpPort` değerini ayarlayın.
- Uzak Chrome için `browser.profiles.<name>.cdpUrl` kullanın (profil ve örnek başına).

## Elle ortam değişkeni örneği

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/main.json \
OPENCLAW_STATE_DIR=~/.openclaw \
openclaw gateway --port 18789

OPENCLAW_CONFIG_PATH=~/.openclaw/rescue.json \
OPENCLAW_STATE_DIR=~/.openclaw-rescue \
openclaw gateway --port 19789
```

## Hızlı kontroller

```bash
openclaw gateway status --deep
openclaw --profile rescue gateway status --deep
openclaw --profile rescue gateway probe
openclaw status
openclaw --profile rescue status
openclaw --profile rescue browser status
```

- `gateway status --deep`, eski kurulumlardan kalan güncelliğini yitirmiş launchd/systemd/schtasks hizmetlerini yakalar.
- `gateway probe` uyarı metninin, örneğin `multiple reachable gateway identities detected`, yalnızca birden fazla yalıtılmış Gateway'i kasıtlı olarak çalıştırdığınızda veya OpenClaw erişilebilir yoklama hedeflerinin aynı Gateway olduğunu doğrulayamadığında görülmesi beklenir. Aynı Gateway'e yönelik bir SSH tüneli, proxy URL'si veya yapılandırılmış uzak URL, aktarım portları farklı olsa bile birden fazla aktarıma sahip tek bir Gateway'dir.

## İlgili

- [Gateway çalışma kılavuzu](/tr/gateway)
- [Gateway kilidi](/tr/gateway/gateway-lock)
- [Yapılandırma](/tr/gateway/configuration)
