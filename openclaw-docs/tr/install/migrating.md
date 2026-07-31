---
read_when:
    - OpenClaw'u yeni bir dizüstü bilgisayara veya sunucuya taşıyorsunuz
    - Başka bir aracı sisteminden geçiş yapıyorsunuz ve durumu korumak istiyorsunuz
    - Yerinde kurulu bir plugin'i yükseltiyorsunuz
summary: 'Geçiş merkezi: sistemler arası içe aktarmalar, makineler arası taşımalar ve plugin yükseltmeleri'
title: Geçiş kılavuzu
x-i18n:
    generated_at: "2026-07-27T00:03:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e9ceb80045ab082c9cfc9e1aca59e079b6bf28b1d047265a0be40c03ebe5dac6
    source_path: install/migrating.md
    workflow: 16
---

OpenClaw üç geçiş yolunu destekler: başka bir aracı sisteminden içe aktarma, mevcut bir kurulumu yeni bir makineye taşıma ve bir Plugin'i yerinde yükseltme.

## Başka bir aracı sisteminden içe aktarma

Paketle birlikte sunulan geçiş sağlayıcıları; talimatları, MCP sunucularını, Skills'leri, model yapılandırmasını ve (isteğe bağlı) API anahtarlarını OpenClaw'a aktarır. Herhangi bir değişiklik yapılmadan önce planlar önizlenir ve raporlarda gizli bilgiler maskelenir. Bağımsız `openclaw migrate` doğrulanmış bir yedekle desteklenir; yeni ilk katılım içe aktarmaları ise geri döndürülemez herhangi bir harici etkinleştirmeden önce yapılandırmayı kaydederek, yerel yapıtları yayımlamadan önce hazırlar ve doğrular.

<CardGroup cols={2}>
  <Card title="Claude'dan geçiş" href="/tr/install/migrating-claude" icon="brain">
    `CLAUDE.md`, MCP sunucuları, Skills'ler ve proje komutları dahil olmak üzere Claude Code ve Claude Desktop durumunu içe aktarın.
  </Card>
  <Card title="Hermes'ten geçiş" href="/tr/install/migrating-hermes" icon="feather">
    Hermes yapılandırmasını, sağlayıcılarını, MCP sunucularını, belleğini, Skills'lerini ve desteklenen `.env` anahtarlarını içe aktarın.
  </Card>
</CardGroup>

CLI giriş noktası [`openclaw migrate`](/tr/cli/migrate) şeklindedir. İlk katılım, bilinen bir kaynak algıladığında da geçiş önerebilir (`openclaw onboard --flow import`).

## OpenClaw'ı yeni bir makineye taşıma

Şunları korumak için **durum dizinini** (varsayılan olarak `~/.openclaw/`) ve **çalışma alanınızı** kopyalayın:

- **Yapılandırma** — `openclaw.json` ve tüm Gateway ayarları.
- **Kimlik doğrulama** — aracı başına `auth-profiles.json` (API anahtarları ve OAuth) ile `credentials/` altındaki kanal veya sağlayıcı durumları.
- **Oturumlar** — konuşma geçmişi ve aracı durumu.
- **Kanal durumu** — WhatsApp oturum açma bilgileri, Telegram oturumu ve benzerleri.
- **Çalışma alanı dosyaları** — `MEMORY.md`, `USER.md`, Skills'ler ve istemler.

<Tip>
Durum dizini yolunuzu doğrulamak için eski makinede `openclaw status` komutunu çalıştırın. Özel profiller `~/.openclaw-<profile>/` veya `OPENCLAW_STATE_DIR` aracılığıyla ayarlanan bir yolu kullanır.
</Tip>

### Geçiş adımları

<Steps>
  <Step title="Gateway'i durdurun ve yedekleyin">
    **Eski** makinede, dosyaların kopyalama sırasında değişmemesi için Gateway'i durdurun ve ardından arşivleyin:

    ```bash
    openclaw gateway stop
    cd ~
    tar -czf openclaw-state.tgz .openclaw
    ```

    Birden fazla profil kullanıyorsanız (örneğin `~/.openclaw-work`), her birini ayrı ayrı arşivleyin.

  </Step>

  <Step title="OpenClaw'ı yeni makineye kurun">
    Yeni makineye CLI'yi (ve gerekirse Node'u) [kurun](/tr/install). İlk katılımın yeni bir `~/.openclaw/` oluşturmasında sakınca yoktur; sonraki adımda bunun üzerine yazacaksınız.
  </Step>

  <Step title="Durum dizinini ve çalışma alanını kopyalayın">
    Arşivi `scp`, `rsync -a` veya harici bir sürücü aracılığıyla aktarın ve ardından çıkarın:

    ```bash
    cd ~
    tar -xzf openclaw-state.tgz
    ```

    Gizli dizinlerin dahil edildiğini ve dosya sahipliğinin Gateway'i çalıştıracak kullanıcıyla eşleştiğini doğrulayın.

  </Step>

  <Step title="Doctor'ı çalıştırın ve doğrulayın">
    Yapılandırma geçişlerini uygulamak ve hizmetleri onarmak için yeni makinede [Doctor](/tr/gateway/doctor) aracını çalıştırın:

    ```bash
    openclaw doctor
    openclaw gateway restart
    openclaw status
    ```

  </Step>
</Steps>

Telegram veya Discord varsayılan ortam değişkeni geri dönüşünü (`TELEGRAM_BOT_TOKEN` veya `DISCORD_BOT_TOKEN`) kullanıyorsa, gizli değerleri yazdırmadan, taşınan durum dizinindeki `.env` dosyasının bu anahtarları içerdiğini doğrulayın:

```bash
awk -F= '/^(TELEGRAM_BOT_TOKEN|DISCORD_BOT_TOKEN)=/ { print $1 "=present" }' ~/.openclaw/.env
```

`openclaw doctor`, etkinleştirilmiş varsayılan bir Telegram veya Discord hesabında yapılandırılmış bir token bulunmadığında ve eşleşen ortam değişkeni Doctor işlemi tarafından kullanılamadığında da uyarır.

### Yaygın sorunlar

<AccordionGroup>
  <Accordion title="Profil veya durum dizini uyuşmazlığı">
    Eski Gateway `--profile` veya `OPENCLAW_STATE_DIR` kullanırken yenisi kullanmıyorsa, kanalların oturumu kapalı görünür ve oturumlar boş olur. Gateway'i taşıdığınız **aynı** profil veya durum diziniyle başlatın, ardından `openclaw doctor` komutunu yeniden çalıştırın.
  </Accordion>

  <Accordion title="Yalnızca openclaw.json dosyasını kopyalama">
    Yapılandırma dosyası tek başına yeterli değildir. Model kimlik doğrulama profilleri `agents/<agentId>/agent/auth-profiles.json` altında, kanal ve sağlayıcı durumu ise `credentials/` altında bulunur. Her zaman **tüm** durum dizinini taşıyın.
  </Accordion>

  <Accordion title="İzinler ve sahiplik">
    Kök kullanıcı olarak kopyaladıysanız veya kullanıcı değiştirdiyseniz Gateway kimlik bilgilerini okuyamayabilir. Durum dizininin ve çalışma alanının Gateway'i çalıştıran kullanıcıya ait olduğundan emin olun.
  </Accordion>

  <Accordion title="Uzak mod">
    Kullanıcı arayüzünüz **uzak** bir Gateway'e bağlanıyorsa oturumların ve çalışma alanının sahibi uzak ana makinedir. Yerel dizüstü bilgisayarınızı değil, Gateway ana makinesinin kendisini taşıyın. [SSS](/tr/help/faq#where-things-live-on-disk) sayfasına bakın.
  </Accordion>

  <Accordion title="Yedeklerdeki gizli bilgiler">
    Durum dizini; kimlik doğrulama profillerini, kanal kimlik bilgilerini ve diğer sağlayıcı durumlarını içerir. Yedekleri şifrelenmiş olarak saklayın, güvenli olmayan aktarım kanallarından kaçının ve açığa çıkmış olabileceğinden şüpheleniyorsanız anahtarları yenileyin.
  </Accordion>
</AccordionGroup>

### Doğrulama kontrol listesi

Yeni makinede şunları doğrulayın:

- [ ] `openclaw status`, Gateway'in çalıştığını gösteriyor.
- [ ] Kanallar hâlâ bağlı (yeniden eşleştirme gerekmiyor).
- [ ] Gösterge paneli açılıyor ve mevcut oturumları gösteriyor.
- [ ] Çalışma alanı dosyaları (bellek, yapılandırmalar) mevcut.

## Bir Plugin'i yerinde yükseltme

Yerinde Plugin yükseltmeleri aynı Plugin kimliğini ve yapılandırma anahtarlarını korur ancak disk üzerindeki durumu geçerli düzene taşıyabilir. Plugin'e özgü yükseltme kılavuzları ilgili kanalların yanında bulunur:

- [Matrix geçişi](/tr/channels/matrix-migration): şifrelenmiş durum kurtarma sınırları, otomatik anlık görüntü davranışı ve manuel kurtarma komutları.

## İlgili

- [`openclaw migrate`](/tr/cli/migrate): sistemler arası içe aktarmalar için CLI başvurusu.
- [Kuruluma genel bakış](/tr/install): tüm kurulum yöntemleri.
- [Doctor](/tr/gateway/doctor): geçiş sonrası sistem durumu denetimi.
- [Kaldırma](/tr/install/uninstall): OpenClaw'ı temiz bir şekilde kaldırma.
