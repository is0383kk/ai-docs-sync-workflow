---
read_when:
    - Claude Code veya Claude Desktop'tan geçiş yapıyorsunuz ve talimatları, MCP sunucularını ve becerileri korumak istiyorsunuz
    - OpenClaw'un neleri otomatik olarak içe aktardığını ve nelerin yalnızca arşivde kaldığını anlamanız gerekir
summary: Ön izlemesi gösterilen bir içe aktarma işlemiyle Claude Code ve Claude Desktop yerel durumunu OpenClaw'a taşıyın
title: Claude'dan Geçiş
x-i18n:
    generated_at: "2026-07-26T23:23:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0d5a5e63727e1583fc3fa27ac45215c72df9074b21d7c5f6b33800bec916769b
    source_path: install/migrating-claude.md
    workflow: 16
---

OpenClaw, paketle birlikte sunulan Claude geçiş sağlayıcısı aracılığıyla yerel Claude durumunu içe aktarır. Sağlayıcı, durumu değiştirmeden önce her öğenin önizlemesini gösterir ve planlar ile raporlardaki gizli bilgileri maskeler. Bağımsız `openclaw migrate`, doğrulanmış bir yedek oluşturur; yeni ilk katılım yolu içe aktarımı hazırlar ve yalnızca doğrulama başarılı olduktan sonra yayımlar.

<Note>
İlk katılım içe aktarımları yeni bir OpenClaw kurulumu gerektirir. Zaten yerel OpenClaw durumunuz varsa önce yapılandırmayı, kimlik bilgilerini, oturumları ve çalışma alanını sıfırlayın veya planı inceledikten sonra `--overwrite` ile birlikte doğrudan `openclaw migrate` kullanın.
</Note>

## İçe aktarmanın iki yolu

<Tabs>
  <Tab title="İlk katılım sihirbazı">
    Sihirbaz, yerel Claude durumu algıladığında Claude seçeneğini sunar.

    ```bash
    openclaw onboard --flow import
    ```

    Ya da belirli bir kaynağı gösterin:

    ```bash
    openclaw onboard --import-from claude --import-source ~/.claude
    ```

  </Tab>
  <Tab title="CLI">
    Betikli veya tekrarlanabilir çalıştırmalar için `openclaw migrate` kullanın. Tam başvuru için [`openclaw migrate`](/tr/cli/migrate) sayfasına bakın.

    ```bash
    openclaw migrate claude --dry-run
    openclaw migrate apply claude --yes
    ```

    Belirli bir Claude Code ana dizinini veya proje kökünü içe aktarmak için `--from <path>` ekleyin.

  </Tab>
</Tabs>

## İçe aktarılanlar

<AccordionGroup>
  <Accordion title="Talimatlar ve bellek">
    - Proje `CLAUDE.md` ve `.claude/CLAUDE.md` içeriği, OpenClaw aracısının `AGENTS.md` çalışma alanına kopyalanır veya eklenir.
    - Kullanıcı `~/.claude/CLAUDE.md` içeriği, çalışma alanındaki `USER.md` öğesine eklenir.

  </Accordion>
  <Accordion title="MCP sunucuları">
    MCP sunucusu tanımları, mevcut olduklarında proje `.mcp.json`, Claude Code `~/.claude.json` ve Claude Desktop `claude_desktop_config.json` öğelerinden içe aktarılır.
  </Accordion>
  <Accordion title="Skills ve komutlar">
    - `SKILL.md` dosyası bulunan Claude Skills öğeleri, OpenClaw çalışma alanının Skills dizinine kopyalanır.
    - `.claude/commands/` veya `~/.claude/commands/` altındaki Claude komut Markdown dosyaları, `disable-model-invocation: true` ile OpenClaw Skills öğelerine dönüştürülür.

  </Accordion>
</AccordionGroup>

## Yalnızca arşivde kalanlar

Sağlayıcı, elle incelenmeleri için bunları geçiş raporuna kopyalar ancak canlı OpenClaw yapılandırmasına **yüklemez**:

- Claude kancaları
- Claude izinleri ve geniş araç izin listeleri
- Claude ortam varsayılanları
- `CLAUDE.local.md`
- `.claude/rules/`
- `.claude/agents/` veya `~/.claude/agents/` altındaki Claude alt aracıları
- Claude Code önbellekleri, planları ve proje geçmişi dizinleri
- Claude Desktop uzantıları ve işletim sisteminde saklanan kimlik bilgileri

OpenClaw; kancaları yürütmeyi, izin listelerine güvenmeyi veya şeffaf olmayan OAuth ve Desktop kimlik bilgisi durumunu otomatik olarak çözmeyi reddeder. Arşivi inceledikten sonra ihtiyaç duyduklarınızı elle taşıyın.

## Kaynak seçimi

`--from` olmadan OpenClaw, `~/.claude` konumundaki varsayılan Claude Code ana dizinini, örneklenmiş Claude Code `~/.claude.json` durum dosyasını ve macOS'taki Claude Desktop MCP yapılandırmasını inceler.

`--from` bir proje kökünü gösterdiğinde OpenClaw yalnızca söz konusu projenin `CLAUDE.md`, `.claude/settings.json`, `.claude/commands/`, `.claude/skills/` ve `.mcp.json` gibi Claude dosyalarını içe aktarır. Proje kökü içe aktarımı sırasında genel Claude ana dizininizi okumaz.

## Önerilen akış

<Steps>
  <Step title="Planın önizlemesini görüntüleyin">
    ```bash
    openclaw migrate claude --dry-run
    ```

    Plan; çakışmalar, atlanan öğeler ve iç içe MCP `env` veya `headers` alanlarında maskelenen hassas değerler dâhil olmak üzere değiştirilecek her şeyi listeler.

  </Step>
  <Step title="Yedek alarak uygulayın">
    ```bash
    openclaw migrate apply claude --yes
    ```

    OpenClaw, uygulamadan önce bir yedek oluşturur ve doğrular.

  </Step>
  <Step title="Doctor'ı çalıştırın">
    ```bash
    openclaw doctor
    ```

    [Doctor](/tr/gateway/doctor), içe aktarımdan sonra yapılandırma veya durum sorunlarını denetler.

  </Step>
  <Step title="Yeniden başlatın ve doğrulayın">
    ```bash
    openclaw gateway restart
    openclaw status
    ```

    Gateway'in sağlıklı olduğunu ve içe aktarılan talimatlarınızın, MCP sunucularınızın ve Skills öğelerinizin yüklendiğini doğrulayın.

  </Step>
</Steps>

## Çakışmaları işleme

Plan çakışma bildirdiğinde (hedefte zaten bir dosya veya yapılandırma değeri mevcutsa) uygulama devam etmeyi reddeder.

<Warning>
Yalnızca mevcut hedefi değiştirmek bilerek isteniyorsa `--overwrite` ile yeniden çalıştırın. Sağlayıcılar, üzerine yazılan dosyalar için geçiş raporu dizinine öğe düzeyinde yedekler yazmaya devam edebilir.
</Warning>

Yeni bir OpenClaw kurulumunda çakışmalar olağan değildir. Genellikle içe aktarımı zaten kullanıcı düzenlemeleri bulunan bir kurulumda yeniden çalıştırdığınızda ortaya çıkarlar.

## Otomasyon için JSON çıktısı

```bash
openclaw migrate claude --dry-run --json
openclaw migrate apply claude --json --yes
```

Etkileşimli bir terminal dışında `migrate apply` için `--yes` gereklidir; bu olmadan OpenClaw uygulamak yerine hata verir, dolayısıyla betikler ve CI açıkça `--yes` iletmelidir. Önce `--dry-run --json` ile önizleme yapın, ardından plan doğru göründüğünde `--json --yes` ile uygulayın.

## Sorun giderme

<AccordionGroup>
  <Accordion title="Claude durumu ~/.claude dışında bulunuyor">
    `--from /actual/path` (CLI) veya `--import-source /actual/path` (ilk katılım) iletin.
  </Accordion>
  <Accordion title="İlk katılım, mevcut bir kurulumda içe aktarmayı reddediyor">
    İlk katılım içe aktarımları yeni bir kurulum gerektirir. Durumu sıfırlayıp ilk katılımı yeniden gerçekleştirin veya `--overwrite` ve açık yedekleme denetimini destekleyen `openclaw migrate apply claude` öğesini doğrudan kullanın.
  </Accordion>
  <Accordion title="Claude Desktop'taki MCP sunucuları içe aktarılmadı">
    Claude Desktop, platforma özgü bir yoldan `claude_desktop_config.json` dosyasını okur. OpenClaw bunu otomatik olarak algılamadıysa `--from` öğesini bu dosyanın dizinine yönlendirin.
  </Accordion>
  <Accordion title="Claude komutları, model çağrısı devre dışı bırakılmış Skills öğelerine dönüştü">
    Bu, tasarım gereğidir. Claude komutları kullanıcı tarafından tetiklendiğinden OpenClaw bunları `disable-model-invocation: true` ile Skills öğeleri olarak içe aktarır. Aracının bunları otomatik olarak çağırmasını istiyorsanız her Skill öğesinin frontmatter bölümünü düzenleyin.
  </Accordion>
</AccordionGroup>

## İlgili

- [`openclaw migrate`](/tr/cli/migrate): tam CLI başvurusu, Plugin sözleşmesi ve JSON biçimleri.
- [Geçiş kılavuzu](/tr/install/migrating): tüm geçiş yolları.
- [Hermes'ten geçiş](/tr/install/migrating-hermes): diğer sistemler arası içe aktarma yolu.
- [İlk katılım](/tr/cli/onboard): sihirbaz akışı ve etkileşimsiz bayraklar.
- [Doctor](/tr/gateway/doctor): geçiş sonrası sistem durumu denetimi.
- [Aracı çalışma alanı](/tr/concepts/agent-workspace): `AGENTS.md`, `USER.md` ve Skills öğelerinin bulunduğu yer.
