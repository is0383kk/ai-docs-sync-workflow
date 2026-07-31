---
read_when:
    - Hermes'ten geçiş yapıyorsunuz ve model yapılandırmanızı, istemlerinizi, belleğinizi ve becerilerinizi korumak istiyorsunuz
    - OpenClaw'un neleri otomatik olarak içe aktardığını ve nelerin yalnızca arşivde kaldığını öğrenmek istiyorsunuz
    - Temiz, betiklerle yönetilen bir geçiş yoluna ihtiyacınız var (CI, yeni dizüstü bilgisayar, otomasyon)
summary: Önizlenebilir, geri alınabilir bir içe aktarmayla Hermes'ten OpenClaw'a geçin
title: Hermes'ten Geçiş Yapma
x-i18n:
    generated_at: "2026-07-26T23:44:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8cdb7a77cfb8ecb0504ccc322b5600c6ed671a8bf9ac866d964fdf4b3494000
    source_path: install/migrating-hermes.md
    workflow: 16
---

Paketle birlikte gelen Hermes geçiş sağlayıcısı, `HERMES_HOME` ve etkin Hermes profilini izler; macOS/Linux'ta `~/.hermes`, Windows'ta ise `%LOCALAPPDATA%\hermes` seçeneğine geri döner. Uygulamadan önce her değişikliğin önizlemesini gösterir ve planlar ile raporlardaki gizli bilgileri maskeler. Bağımsız `openclaw migrate` doğrulanmış bir yedek yazar; yeni ilk kurulum yolu yapılandırmayı, kimlik bilgilerini ve dosyaları hazırlar ve bunları yalnızca içe aktarılan çıkarım doğrulandıktan sonra yayımlar. Açıkça belirtilen bir `--from` yolu her zaman önceliklidir.

<Note>
İçe aktarma işlemleri yeni bir OpenClaw kurulumu gerektirir. Zaten yerel OpenClaw durumunuz varsa önce yapılandırmayı, kimlik bilgilerini, oturumları ve çalışma alanını sıfırlayın ya da planı inceledikten sonra `--overwrite` ile doğrudan `openclaw migrate apply hermes` kullanın.
</Note>

## İçe aktarmanın iki yolu

<Tabs>
  <Tab title="İlk kurulum sihirbazı">
    Etkin Hermes ana dizinini/profilini algılar ve uygulamadan önce bir önizleme gösterir.

    ```bash
    openclaw onboard --flow import
    ```

    Veya belirli bir kaynağı gösterin:

    ```bash
    openclaw onboard --import-from hermes --import-source ~/.hermes
    ```

  </Tab>
  <Tab title="CLI">
    Betikli veya tekrarlanabilir çalıştırmalar için `openclaw migrate` kullanın. Tam başvuru için [`openclaw migrate`](/tr/cli/migrate) sayfasına bakın.

    ```bash
    openclaw migrate hermes --dry-run    # yalnızca önizleme
    openclaw migrate apply hermes --yes  # onay atlanarak uygula
    ```

    Hermes ana dizini/profili keşfini geçersiz kılmak için `--from <path>` ekleyin.

  </Tab>
</Tabs>

## İçe aktarılanlar

<AccordionGroup>
  <Accordion title="Model yapılandırması">
    - Hermes `config.yaml` kaynağındaki varsayılan model seçimi.
    - Mevcut Hermes Chat Completions, Codex Responses ve Anthropic Messages aktarımları dâhil olmak üzere `model`, `providers` ve `custom_providers` kaynaklarındaki yapılandırılmış model sağlayıcıları ve özel uç noktalar.

  </Accordion>
  <Accordion title="MCP sunucuları">
    Devre dışı durumu, zaman aşımları, paralel araç desteği, OAuth kapsamı, uyumlu TLS alanları ve yerel/kaynak/istem araç politikası dâhil olmak üzere `mcp_servers` veya `mcp.servers` kaynağındaki MCP sunucu tanımları. Değişmez ortam değişkenleri ve üstbilgiler, kimlik bilgilerini içe aktarma izni gerektirir. Yalnızca Hermes'e özgü yaşam döngüsü, örnekleme, bilgi isteme, ön kontrol, bağlantıyı canlı tutma, CA paketi, parola korumalı istemci anahtarı ve önceden kaydedilmiş OAuth istemcisi ayarları, geçersiz OpenClaw yapılandırması yerine elle incelenecek öğelere dönüşür.
  </Accordion>
  <Accordion title="Çalışma alanı dosyaları">
    - `SOUL.md` ve `AGENTS.md`, OpenClaw ajan çalışma alanına kopyalanır.
    - `memories/MEMORY.md` ve `memories/USER.md`, üzerlerine yazılmak yerine eşleşen OpenClaw bellek dosyalarına **eklenir**.
    - Yalnızca belleğe yönelik yüzeyler farklı davranır: İlk kurulum bellek sayfası ve Control UI Bellek içe aktarma sayfası, dizinlenmiş hatırlama için bu iki dosyayı `memory/imports/hermes/` altına kopyalar ve mevcut çalışma alanı belleğine dokunmaz.

  </Accordion>
  <Accordion title="Bellek yapılandırması">
    OpenClaw dosya belleği için bellek yapılandırması varsayılanları. Honcho gibi harici bellek sağlayıcıları, bunları bilinçli olarak taşıyabilmeniz için arşiv veya elle incelenecek öğeler olarak kaydedilir.
  </Accordion>
  <Accordion title="Skills">
    `skills/` altında herhangi bir yerde `SKILL.md` dosyası bulunan Skills özyinelemeli olarak keşfedilir, OpenClaw çalışma alanı skill dizininde düzleştirilir ve destek dosyalarıyla birlikte kopyalanır. `skills.config` kaynağındaki skill başına yapılandırma değerleri korunur.
  </Accordion>
  <Accordion title="Kimlik doğrulama bilgileri">
    Etkileşimli `openclaw migrate`, varsayılan olarak evet seçiliyken kimlik doğrulama bilgilerini içe aktarmadan önce sorar. Kabul edilen içe aktarımlar arasında mevcut Hermes OpenAI Codex OAuth girdileri, OpenCode OpenAI OAuth ve GitHub Copilot girdileri ile [desteklenen Hermes `.env` anahtarları](/tr/cli/migrate#supported-env-keys) bulunur. Etkileşimsiz içe aktarma için `--include-secrets`, kimlik bilgilerini atlamak için `--no-auth-credentials` veya ilk kurulumun `--import-secrets` bayrağını kullanın. Hermes OAuth'u içe aktardıktan sonra Hermes ile OpenClaw'ın aynı yenileme yetkisini kullanmasına izin vermeyin; ikisini birlikte çalıştırmadan önce taraflardan birinde yeniden kimlik doğrulayın.
  </Accordion>
</AccordionGroup>

## Yalnızca arşivde kalanlar

Sağlayıcı, elle incelenmeleri için bunları geçiş raporu dizinine kopyalar ancak canlı OpenClaw yapılandırmasına veya kimlik bilgilerine **yüklemez**:

- `plugins/`
- `sessions/`
- `logs/`
- `cron/`
- `mcp-tokens/`
- `plans/`, `workspace/`, `skins/` ve `kanban/`
- `pairing/` ve `platforms/` depolarının yanı sıra gateway yönlendirme/işlem durumu
- `state.db`, `hermes_state.db`, `projects.db`, `response_store.db`, `memory_store.db`, `verification_evidence.db`, `kanban.db` ve `retaindb_queue.db`

Biçimler ve güven varsayımları sistemler arasında farklılaşabileceğinden OpenClaw bu durumu otomatik olarak yürütmeyi veya güvenilir kabul etmeyi reddeder. Arşivi inceledikten sonra ihtiyaç duyduklarınızı elle taşıyın.

## Önerilen akış

<Steps>
  <Step title="Planın önizlemesini gösterin">
    ```bash
    openclaw migrate hermes --dry-run
    ```

    Plan; çakışmalar, atlanan öğeler ve hassas öğeler dâhil olmak üzere değişecek her şeyi listeler. İç içe geçmiş, gizli bilgiye benzeyen anahtarlar çıktıda maskelenir.

  </Step>
  <Step title="Yedekle birlikte uygulayın">
    ```bash
    openclaw migrate apply hermes --yes
    ```

    OpenClaw uygulamadan önce bir yedek oluşturur ve doğrular. Bu etkileşimsiz örnek yalnızca gizli olmayan durumu içe aktarır. Kimlik bilgileri istemini etkileşimli olarak yanıtlamak için `--yes` olmadan çalıştırın veya gözetimsiz bir çalıştırmada desteklenen kimlik bilgilerini dâhil etmek için `--include-secrets` ekleyin.

  </Step>
  <Step title="Doctor'ı çalıştırın">
    ```bash
    openclaw doctor
    ```

    [Doctor](/tr/gateway/doctor), bekleyen yapılandırma geçişlerini yeniden uygular ve içe aktarma sırasında ortaya çıkan sorunları denetler.

  </Step>
  <Step title="Yeniden başlatın ve doğrulayın">
    ```bash
    openclaw gateway restart
    openclaw status
    ```

    Gateway'in sağlıklı olduğunu ve içe aktarılan modelinizin, belleğinizin ve Skills'in yüklendiğini doğrulayın.

  </Step>
</Steps>

## Çakışma yönetimi

Plan çakışma bildirdiğinde (hedefte zaten bir dosya veya yapılandırma değeri bulunduğunda) uygulama devam etmeyi reddeder.

<Warning>
Yalnızca mevcut hedefin değiştirilmesi amaçlanıyorsa `--overwrite` ile yeniden çalıştırın. Sağlayıcılar, üzerine yazılan dosyalar için geçiş raporu dizinine yine de öğe düzeyinde yedekler yazabilir.
</Warning>

Yeni bir kurulumda çakışmalar olağan dışıdır. Genellikle içe aktarmayı kullanıcı düzenlemeleri bulunan bir kurulumda yeniden çalıştırdığınızda ortaya çıkarlar.

Uygulamanın ortasında bir çakışma ortaya çıkarsa (örneğin, bir yapılandırma dosyasında beklenmeyen bir yarış durumu), bağımsız dosyalar, Skills, kimlik bilgileri, arşivler ve yapılandırma girdileri devam ederken bu öğe çakışma olarak bildirilir. Çakışan öğeyi çözün ve içe aktarmayı yeniden çalıştırın; aynı bellek içe aktarımları eş etkisizdir.

## Gizli bilgiler

Etkileşimli `openclaw migrate`, algılanan kimlik doğrulama bilgilerini içe aktarıp aktarmayacağınızı varsayılan olarak evet seçiliyken sorar.

- Kabul edildiğinde mevcut Hermes OpenAI Codex OAuth girdileri, OpenCode OpenAI OAuth ve GitHub Copilot girdileri ile [desteklenen `.env` anahtarları](/tr/cli/migrate#supported-env-keys) içe aktarılır.
- Yalnızca gizli olmayan durumu içe aktarmak için `--no-auth-credentials` kullanın veya istemde hayır yanıtını verin.
- Gözetimsiz bir `--yes` çalıştırmasında kimlik bilgilerini içe aktarmak için `--include-secrets` kullanın.
- Kimlik bilgilerini sihirbazdan içe aktarmak için ilk kurulum sihirbazının `--import-secrets` bayrağını kullanın.

## Otomasyon için JSON çıktısı

```bash
openclaw migrate hermes --dry-run --json
openclaw migrate apply hermes --json --yes
```

`--json` kullanıldığında ve `--yes` kullanılmadığında uygulama planı yazdırır ve durumu değiştirmez; bu, CI ve paylaşılan betikler için en güvenli moddur.

## Sorun giderme

<AccordionGroup>
  <Accordion title="Uygulama çakışmalar nedeniyle reddediliyor">
    Plan çıktısını inceleyin. Her çakışma, kaynak yolunu ve mevcut hedefi tanımlar. Her öğe için atlama, hedefi düzenleme veya `--overwrite` ile yeniden çalıştırma seçeneklerinden birini belirleyin.
  </Accordion>
  <Accordion title="Hermes ~/.hermes dışında bulunuyor">
    `--from /actual/path` (CLI) veya `--import-source /actual/path` (ilk kurulum) iletin.
  </Accordion>
  <Accordion title="İlk kurulum mevcut bir kuruluma aktarmayı reddediyor">
    İlk kurulumdan içe aktarma yeni bir kurulum gerektirir. Durumu sıfırlayıp ilk kurulumu yeniden yapın veya `--overwrite` ve açık yedekleme denetimini destekleyen `openclaw migrate apply hermes` komutunu doğrudan kullanın.
  </Accordion>
  <Accordion title="API anahtarları içe aktarılmadı">
    Etkileşimli `openclaw migrate`, API anahtarlarını yalnızca kimlik bilgileri istemini kabul ettiğinizde içe aktarır. Etkileşimsiz `--yes` çalıştırmaları `--include-secrets`; ilk kurulum içe aktarmaları ise `--import-secrets` gerektirir. Yalnızca [desteklenen `.env` anahtarları](/tr/cli/migrate#supported-env-keys) tanınır; diğer `.env` değişkenleri yok sayılır.
  </Accordion>
</AccordionGroup>

## İlgili kaynaklar

- [`openclaw migrate`](/tr/cli/migrate): tam CLI başvurusu, plugin sözleşmesi ve JSON şekilleri.
- [İlk kurulum](/tr/cli/onboard): sihirbaz akışı ve etkileşimsiz bayraklar.
- [Geçiş](/tr/install/migrating): bir OpenClaw kurulumunu makineler arasında taşıma.
- [Doctor](/tr/gateway/doctor): geçiş sonrası sistem durumu denetimi.
- [Ajan çalışma alanı](/tr/concepts/agent-workspace): `SOUL.md`, `AGENTS.md` ve bellek dosyalarının bulunduğu yer.
