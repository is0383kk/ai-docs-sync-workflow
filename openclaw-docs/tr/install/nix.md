---
read_when:
    - Yeniden üretilebilir, geri alınabilir kurulumlar istiyorsunuz
    - Zaten Nix/NixOS/Home Manager kullanıyorsunuz
    - Her şeyin sabitlenmesini ve bildirimsel olarak yönetilmesini istiyorsunuz
summary: OpenClaw'u Nix ile bildirimsel olarak yükleyin
title: Nix
x-i18n:
    generated_at: "2026-07-27T00:03:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f74e259ec3d909c73d9184db24d236135db04c29c2e7fab9be9e6fa7f98ba91
    source_path: install/nix.md
    workflow: 16
---

OpenClaw'u, birinci taraf, kullanıma hazır Home Manager modülü **[nix-openclaw](https://github.com/openclaw/nix-openclaw)** ile bildirimsel olarak kurun.

<Info>
[nix-openclaw](https://github.com/openclaw/nix-openclaw) deposu, Nix kurulumu için temel referans kaynağıdır. Bu sayfa kısa bir genel bakış sunar.
</Info>

## Neler sunulur?

- Gateway + macOS uygulaması + araçlar (whisper, spotify, kameralar), tümü sabitlenmiş sürümlerde
- Yeniden başlatmalardan etkilenmeyen Launchd hizmeti
- Bildirimsel yapılandırmaya sahip Plugin sistemi
- Anında geri alma: `home-manager switch --rollback`

## Hızlı başlangıç

<Steps>
  <Step title="Determinate Nix'i yükleyin">
    Nix henüz yüklü değilse [Determinate Nix yükleyicisi](https://github.com/DeterminateSystems/nix-installer) talimatlarını izleyin.
  </Step>
  <Step title="Yerel bir flake oluşturun">
    nix-openclaw deposundaki öncelikle aracı kullanan şablonu kullanın:
    ```bash
    mkdir -p ~/code/openclaw-local
    # nix-openclaw deposundan templates/agent-first/flake.nix dosyasını kopyalayın
    ```
  </Step>
  <Step title="Gizli bilgileri yapılandırın">
    Mesajlaşma botu belirtecinizi ve model sağlayıcısının API anahtarını ayarlayın. `~/.secrets/` konumundaki düz dosyalar sorunsuz çalışır.
  </Step>
  <Step title="Şablon yer tutucularını doldurun ve geçiş yapın">
    ```bash
    home-manager switch
    ```
  </Step>
  <Step title="Doğrulayın">
    launchd hizmetinin çalıştığını ve botunuzun mesajlara yanıt verdiğini doğrulayın.
  </Step>
</Steps>

Tüm modül seçenekleri ve örnekler için [nix-openclaw README dosyasına](https://github.com/openclaw/nix-openclaw) bakın.

## Nix modu çalışma zamanı davranışı

`OPENCLAW_NIX_MODE=1` ayarlandığında (nix-openclaw ile otomatik olarak), OpenClaw Nix tarafından yönetilen kurulumlar için belirlenimci bir moda girer. Diğer Nix paketleri de aynı modu ayarlayabilir; nix-openclaw birinci taraf referansıdır.

Bunu elle de ayarlayabilirsiniz:

```bash
export OPENCLAW_NIX_MODE=1
```

macOS'te GUI uygulaması, kabuk ortam değişkenlerini devralmaz. Bunun yerine `defaults` aracılığıyla Nix modunu etkinleştirin:

```bash
defaults write ai.openclaw.mac openclaw.nixMode -bool true
```

### Nix modunda neler değişir?

- Otomatik yükleme ve kendi kendini değiştirme akışları devre dışı bırakılır.
- `openclaw.json` değişmez kabul edilir. Başlangıçta türetilen varsayılanlar yalnızca çalışma zamanında kalır ve yapılandırma yazıcıları (kurulum, ilk kullanım, değişiklik yapan `openclaw update`, Plugin yükleme/güncelleme/kaldırma/etkinleştirme, `doctor --fix`, `doctor --generate-gateway-token`, `openclaw config set`) dosyayı düzenlemeyi reddeder.
- Bunun yerine Nix kaynağını düzenleyin. nix-openclaw için öncelikle aracı kullanan [Hızlı Başlangıç](https://github.com/openclaw/nix-openclaw#quick-start) kılavuzunu kullanın ve yapılandırmayı `programs.openclaw.config` veya `instances.<name>.config` altında ayarlayın.
- Eksik bağımlılıklar için Nix'e özgü düzeltme mesajları gösterilir.
- Kullanıcı arayüzünde salt okunur bir Nix modu başlığı gösterilir.

### Yapılandırma ve durum yolları

OpenClaw, JSON5 yapılandırmasını `OPENCLAW_CONFIG_PATH` konumundan okur ve değiştirilebilir verileri `OPENCLAW_STATE_DIR` konumunda saklar. Çalışma zamanı durumu ve yapılandırmanın değişmez deponun dışında kalması için Nix altında bunları açıkça Nix tarafından yönetilen konumlara ayarlayın.

| Değişken               | Varsayılan                                 |
| ---------------------- | --------------------------------------- |
| `OPENCLAW_HOME`        | `HOME` / `USERPROFILE` / `os.homedir()` |
| `OPENCLAW_STATE_DIR`   | `~/.openclaw`                           |
| `OPENCLAW_CONFIG_PATH` | `$OPENCLAW_STATE_DIR/openclaw.json`     |

### Hizmet PATH keşfi

launchd/systemd Gateway hizmeti, `nix` tarafından yüklenen yürütülebilir dosyaları çalıştıran Plugin'lerin ve araçların elle PATH ayarı olmadan çalışması için Nix profili ikili dosyalarını otomatik olarak keşfeder:

- `NIX_PROFILES` ayarlandığında, her giriş sağdan sola öncelik sırasıyla hizmet PATH'ine eklenir (Nix kabuk önceliğiyle eşleşir: en sağdaki kazanır).
- `NIX_PROFILES` ayarlanmadığında, `~/.nix-profile/bin` yedek olarak eklenir.

Bu, hem macOS launchd hem de Linux systemd hizmet ortamları için geçerlidir.

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="nix-openclaw" href="https://github.com/openclaw/nix-openclaw" icon="arrow-up-right-from-square">
    Temel referans kaynağı olan Home Manager modülü ve eksiksiz kurulum kılavuzu.
  </Card>
  <Card title="Kurulum sihirbazı" href="/tr/start/wizard" icon="wand-magic-sparkles">
    Nix kullanmayan CLI kurulumu için adım adım açıklama.
  </Card>
  <Card title="Docker" href="/tr/install/docker" icon="docker">
    Nix kullanmayan bir alternatif olarak kapsayıcılı kurulum.
  </Card>
  <Card title="Güncelleme" href="/tr/install/updating" icon="arrow-up-right-from-square">
    Home Manager tarafından yönetilen kurulumları paketle birlikte güncelleme.
  </Card>
</CardGroup>
