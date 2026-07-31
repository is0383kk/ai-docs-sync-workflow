---
read_when:
    - Kurulum veya onarım için OpenClaw ile sohbet etmek istiyorsunuz
    - İlk çalıştırma kurulumunu başlangıç sihirbazıyla yapıyorsunuz
    - Varsayılan çalışma alanı yolunu ayarlamak istiyorsunuz
    - Betikler için yalnızca temel yapılandırma bayrağına ihtiyacınız var
summary: '`openclaw setup` için CLI referansı (onboarding yedekli sistem aracısı sohbeti)'
title: Kurulum
x-i18n:
    generated_at: "2026-07-26T23:14:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3b4f70f2631683fcb03007a80fe43a06387be3d7e4d533381e5e536333af051
    source_path: cli/setup.md
    workflow: 16
---

# `openclaw setup`

`openclaw setup`, sistem aracısının giriş noktasıdır. Yapılandırılmış bir sistemde, yalın
`openclaw setup` etkileşimli bir OpenClaw sohbeti açar. Yeni bir sistemde ise
yönlendirmeli ilk kuruluma geçer. Tek bir istek için `-m`/`--message`, sihirbaz olmadan
yapılandırma/çalışma alanı klasörlerini başlatmak için `--baseline` kullanın.

Yönlendirme sırası:

1. Herhangi bir ilk kurulum seçeneği (`--wizard`, `--baseline`, çalışma alanı, sıfırlama,
   etkileşimsiz, akış, mod, Gateway, daemon, atlama, içe aktarma, uzak bağlantı veya kimlik doğrulama
   seçenekleri), ilk kurulumu tam olarak `openclaw onboard` gibi çalıştırır.
2. `-m`/`--message` veya `--yes`, sistem aracısını çalıştırır.
3. Yönlendirme seçeneği olmadığında, yapılandırılmış etkileşimli bir sistem OpenClaw'ı açar. Yeni bir
   sistem ilk kurulumu çalıştırır. Yapılandırılmış bir sistemde `--json`, TTY olmasa bile
   sistem genel görünümünü yazdırır; bir ilk kurulum seçeneği, ilk kurulumun
   JSON özetini korur.

Yönlendirmeli modda `--workspace <dir>`, OpenClaw'a önerilen çalışma alanıdır;
yalnızca bu öneriyi onaylamanızdan sonra kalıcı hâle getirilir. Temel, klasik ve
etkileşimsiz kurulum, yeni bir yüklemede sağlanan çalışma alanını normal akışları
üzerinden kalıcı hâle getirir. Mevcut bir aracı listesi yeniden eşlenecekse
klasik sihirbaz açık onay gerektirir; etkileşimsiz kurulum mevcut
filo çalışma alanını korur ve bir uyarı yazdırır.

Yönlendirmeli çıkarım algılaması, macOS veya Linux'ta Gateway ana makinesinde çalışır. CLI
ve macOS uygulaması; yapılandırılmış modelleri, desteklenen CLI oturum açmalarını, API anahtarı
ortam değişkenlerini ve önceden yüklenmiş Ollama ya da LM Studio modellerini denetleyen,
Gateway'e ait aynı algılayıcıyı çağırır. Bu otomatik geçiş sırasında yerel modeller asla
indirilmez. Algılanan yerel çalışma zamanları, CLI ve API anahtarı adaylarından sonra otomatik
olarak test edilir; birden fazla yerel model kullanılabiliyorsa OpenClaw, araç çağırmada en güçlü
instruct ailesini tercih eder. Seçilen adayın sağlayıcı ve model yapılandırması kaydedilmeden
önce gerçek bir tamamlama isteğine yanıt vermesi gerekir.
Yüklü Gemini, Antigravity, Pi ve OpenCode CLI'ları, yönlendirmeli kurulum için yeniden
kullanılabilir çıkarım rotası görevi göremediklerinde de bildirilir.

`setup`; kimlik doğrulama (`--auth-choice`, `--token`, sağlayıcı anahtarı bayrakları), Gateway
(`--gateway-port`, `--gateway-bind`, `--gateway-auth`, `--install-daemon`),
Tailscale (`--tailscale`), sıfırlama (`--reset`, `--reset-scope`), akış
(`--flow quickstart|advanced|manual|import`) ve atlama bayrakları
(`--skip-channels`, `--skip-skills`, `--skip-bootstrap`, `--skip-search`,
`--skip-health`, `--skip-ui`, `--skip-hooks`) dâhil olmak üzere `openclaw onboard` ile aynı ilk kurulum bayraklarını kabul eder. `openclaw onboard --tui` ile aynı
terminal çıkışını kullanmak için `--tui` iletin. Bayrakların tam başvurusu ve
etkileşimsiz örnekler için [İlk Kurulum](/tr/cli/onboard) ve
[CLI otomasyonu](/tr/start/wizard-cli-automation) bölümlerine bakın. `openclaw onboard --modern`, aynı çıkarım
geçitli OpenClaw yardımcısı için uyumluluk
girişi olarak kalır.

<Note>
`openclaw setup`, değiştirilebilir yapılandırma yüklemeleri içindir. Nix modunda (`OPENCLAW_NIX_MODE=1`) yapılandırma dosyası Nix tarafından yönetildiği için OpenClaw kurulum yazma işlemlerini reddeder. Birinci taraf [nix-openclaw Hızlı Başlangıç](https://github.com/openclaw/nix-openclaw#quick-start) kılavuzunu veya başka bir Nix paketi için eşdeğer kaynak yapılandırmasını kullanın.
</Note>

## Seçenekler

| Bayrak                       | Açıklama                                                                                          |
| -------------------------- | ---------------------------------------------------------------------------------------------------- |
| `-m, --message <text>`     | Tek bir OpenClaw isteği çalıştırın.                                                                            |
| `--yes`                    | Tek bir `--message` isteği için kalıcı yapılandırma yazma işlemlerini onaylayın.                                        |
| `--workspace <dir>`        | Çalışma alanı önerisi; mevcut filolar klasik onay gerektirir ve etkileşimsiz olarak korunur. |
| `--baseline`               | İlk kurulum olmadan temel yapılandırma/çalışma alanı/oturum klasörlerini oluşturun.                                 |
| `--wizard`                 | Etkileşimli ilk kurulumu zorlayın.                                                                        |
| `--tui`                    | Tarayıcıya devretmek yerine terminal çıkışını kullanın.                                               |
| `--non-interactive`        | İlk kurulumu istemler olmadan çalıştırın.                                                                      |
| `--accept-risk`            | Tam sistem aracı erişimi riskini kabul edin; `--non-interactive` ile kullanılması zorunludur.                        |
| `--mode <mode>`            | İlk kurulum modu: `local` veya `remote`.                                                                |
| `--flow <flow>`            | İlk kurulum akışı: `quickstart`, `advanced`, `manual` veya `import`.                                       |
| `--reset`                  | İlk kurulumdan önce yapılandırmayı + kimlik bilgilerini + oturumları sıfırlayın (çalışma alanı yalnızca `--reset-scope full` ile).  |
| `--reset-scope <scope>`    | Sıfırlama kapsamı: `config`, `config+creds+sessions` veya `full`.                                           |
| `--import-from <provider>` | İlk kurulum sırasında çalıştırılacak geçiş sağlayıcısı.                                                         |
| `--import-source <path>`   | `--import-from` için kaynak aracı ana dizini.                                                               |
| `--import-secrets`         | İlk kurulum geçişi sırasında desteklenen gizli bilgileri içe aktarın.                                                |
| `--remote-url <url>`       | Uzak Gateway WebSocket URL'si.                                                                        |
| `--remote-token <token>`   | Uzak Gateway token'ı (isteğe bağlı).                                                                     |
| `--json`                   | Yapılandırılmış sistem: OpenClaw genel görünümü. İlk kurulum rotası: ilk kurulum özeti.                          |

`--classic` ve `--non-interactive` birbirini dışlar: klasik, istemli
sihirbazı açarken etkileşimsiz kurulum otomasyon yolunu kullanır.
Etkileşimli ilk kurulumda `--remote-url` ve `--remote-token`, uzak
Gateway adımını önceden doldurur ve o çalıştırma için depolanan uzak bağlantı değerlerine göre önceliklidir.
URL'nin değiştirilmesi, ayrıca bir token iletmediğiniz sürece depolanan kimlik bilgilerini yeniden kullanmaz.
Token maskeli kalır ve sihirbazda seçilen düz metin veya SecretRef
depolama modunu kullanır.

### Temel mod

`openclaw setup --baseline`, eski yalnızca temel davranışı korur:
yapılandırma, çalışma alanı ve oturum dizinlerini oluşturur, ardından
ilk kurulumu çalıştırmadan çıkar. `--workspace` ve zararsız çıktı denetimlerini kabul eder ancak
açık ilk kurulum, Gateway, kimlik doğrulama, sıfırlama veya daemon seçeneklerini sessizce
yok saymak yerine reddeder. Mevcut bir yapılandırma geçersizse temel kurulum bunu korur
ve yeniden denemeden önce `openclaw doctor` çalıştırmanızı ister.

## Örnekler

```bash
openclaw setup
openclaw setup -m "status"
openclaw setup -m "restart gateway" --yes
openclaw setup --json
openclaw setup --wizard
openclaw setup --baseline
openclaw setup --workspace ~/.openclaw/workspace
openclaw setup --import-from hermes --import-source ~/.hermes
openclaw setup --non-interactive --accept-risk --mode remote --remote-url wss://gateway-host:18789 --remote-token <token>
```

## Notlar

- Temel kurulumdan sonra tam yönlendirmeli süreç için `openclaw onboard`, hedefli değişiklikler için `openclaw configure` veya kanal hesapları eklemek için `openclaw channels add` çalıştırın.
- Hermes durumu algılanırsa etkileşimli ilk kurulum, geçişi otomatik olarak önerebilir. İçe aktarmalı ilk kurulum yeni bir kurulum gerektirir; ilk kurulum dışındaki prova planları, yedeklemeler ve üzerine yazma modu için [Geçiş](/tr/cli/migrate) bölümünü kullanın.

## İlgili

- [CLI başvurusu](/tr/cli)
- [İlk Kurulum](/tr/cli/onboard)
- [İlk kurulum (CLI)](/tr/start/wizard)
- [Başlarken](/tr/start/getting-started)
- [Yüklemeye genel bakış](/tr/install)
