---
read_when:
    - CLI ilk katılımını çalıştırma veya yapılandırma
    - Yeni bir makine kurma
sidebarTitle: 'Onboarding: CLI'
summary: 'CLI ilk katılımı: çıkarımı doğrulayın, ardından kalan kurulumu OpenClaw’a devredin'
title: İlk Kurulum (CLI)
x-i18n:
    generated_at: "2026-07-27T00:18:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 150adfac1424b42d66fa3035339082574cc631ce0dc3db09ad32376ef139bf1c
    source_path: start/wizard.md
    workflow: 16
---

```bash
openclaw onboard
```

CLI ilk katılımı, macOS, Linux ve Windows'ta (yerel veya WSL2) önerilen terminal kurulum yoludur. Varsayılan olarak makinede zaten mevcut olan yapay zekâ erişimini algılar, gerçek bir tamamlama ile doğrular ve çalışma alanını, Gateway'i ve isteğe bağlı özellikleri yapılandırmak için OpenClaw'ı başlatır. `openclaw setup` aynı akışı çalıştırır ([Kurulum](/tr/cli/setup), yalnızca yapılandırma amaçlı `--baseline` varyantını ele alır). Windows masaüstü kullanıcıları [Windows Hub](/tr/platforms/windows) üzerinden de başlayabilir.

Yönlendirmeli ilk katılım önce çıkarımı kurar. Mevcut yapay zekâ erişimini algılar, gerçek bir tamamlama gerektirir ve ancak bundan sonra OpenClaw'ın geri kalanını yapılandırmak için [OpenClaw](/tr/cli/openclaw) başlatır. **Şimdilik atla** seçildiğinde OpenClaw başlatılmadan ilk katılımdan çıkılır.

Klasik sihirbaz; özel sağlayıcılar, uzak Gateway kurulumu, kanal eşleştirme, daemon denetimleri, Skills ve içe aktarma işlemleri için kullanılabilir olmaya devam eder. `openclaw onboard --classic` ile açıkça çalıştırılmalıdır; yönlendirmeli çıkarım seçici bunu klasik sihirbaza devretmez. Çıkarım başarılı olduktan sonra OpenClaw, sır gerektiren kanal kurulumunu maskeli bir terminal sihirbazına devretmek için `open channel wizard for
<channel>` kullanabilir. Model sağlayıcısını veya kimlik doğrulamasını değiştirmek için OpenClaw'dan çıkıp `openclaw onboard` çalıştırılmalıdır; OpenClaw yönlendirmeli veya klasik sağlayıcı akışlarını açmaz.

<Info>
En hızlı ilk sohbet: yönlendirmeli kurulumu tamamlayın, `openclaw dashboard` çalıştırın ve Control UI üzerinden tarayıcıda sohbet edin. Belgeler: [Pano](/tr/web/dashboard).
</Info>

## Yerel ayar

Sihirbaz, sabit ilk katılım metinlerini yerelleştirir. Sırasıyla `OPENCLAW_LOCALE`, `LC_ALL`, `LC_MESSAGES` ve `LANG` arasındaki ilk boş olmayan değeri kullanır, ardından İngilizceye geri döner. Desteklenen yerel ayarlar: `en`, `zh-CN`, `zh-TW`.

```bash
OPENCLAW_LOCALE=zh-CN openclaw onboard
OPENCLAW_LOCALE=en openclaw onboard # Açık İngilizce geçersiz kılma
```

Ürün adları, komutlar, yapılandırma anahtarları, URL'ler, sağlayıcı kimlikleri, model kimlikleri ve plugin/kanal etiketleri yerel ayardan bağımsız olarak İngilizce kalır.

Çıkarım dışı ayarları daha sonra yeniden yapılandırmak için:

```bash
openclaw configure
openclaw agents add <name>
```

<Note>
`--json`, etkileşimsiz mod anlamına gelmez. Betikler için `--non-interactive` kullanın (bkz. [CLI otomasyonu](/tr/start/wizard-cli-automation)).
</Note>

<Tip>
Klasik sihirbaz, sağlayıcı seçebileceğiniz bir web araması adımı içerir: Brave, DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Perplexity, SearXNG veya Tavily. Bazıları API anahtarı gerektirirken bazıları anahtarsızdır. Bunu daha sonra `openclaw configure --section web` ile yapılandırın. Belgeler: [Web araçları](/tr/tools/web).
</Tip>

## Yönlendirmeli varsayılan

Düz `openclaw onboard` şu yolu izler:

1. Güvenlik bildirimini kabul edin.
2. Yapılandırılmış modelleri, API anahtarı ortam değişkenlerini, desteklenen yerel yapay zekâ CLI'larını ve Gateway ana makinesinden erişilebilen Ollama veya LM Studio sunucularında önceden kurulmuş araç özellikli modelleri algılar. Bu salt okunur geçiş hiçbir zaman model indirmez. Gemini CLI, Antigravity, Pi ve OpenCode kurulumları, yönlendirmeli kurulum için yeniden kullanılabilir çıkarım yolu olarak hizmet veremediklerinde de bildirilir. Gemini ve Antigravity araçsız yoklamayı zorunlu kılamaz; Pi ve OpenCode ise kurulum çıkarım yolları değil, tam ajan çerçeveleridir.
3. Algılanan ilk adayı gerçek bir tamamlama ile test eder. Başarısızlık durumunda nedeni gösterir ve kullanılabilir sonraki adayla devam eder.
4. Algılama seçenekleri tükenirse OpenAI, Anthropic, xAI (Grok), Google veya OpenRouter'ı seçin ya da kalan sağlayıcılar için **Daha fazla…** seçeneğini belirleyin. Her sağlayıcının bölgeleri, planları ve desteklenen tarayıcı, cihaz, API anahtarı veya token yöntemleri ikinci bir menüde görünür ve aynı gerçek tamamlama ile test edilir. OpenClaw'ı başlatmadan çıkmak için **Şimdilik atla** seçeneğini belirleyin.
5. Yalnızca doğrulanmış model yolunu ve bunun gerektirdiği kimlik bilgisi/plugin durumunu kalıcı hâle getirir. Çalışma alanı ve Gateway ayarlarına dokunulmaz.
6. Çalışma alanını, Gateway'i, kanalları, ajanları, plugin'leri ve kalan isteğe bağlı kurulumu yapılandırabilmesi için OpenClaw'ı doğrulanmış modelle başlatır.

Komut yapılandırılmış bir kurulumda yeniden çalıştırıldığında önce geçerli varsayılan modeli test eder; böylece yönlendirmeli akış bir doğrulama ve onarım geçişi hâline gelir. Başarısız bir denetim, yapılandırılmış modeli hiçbir zaman otomatik olarak değiştirmez; ilk katılım durur ve nasıl devam edileceğini sorar. Daha sonraki çıkarım dışı eklemeler için `openclaw channels add` veya `openclaw configure`; sağlayıcı ya da kimlik doğrulama yolu değişiklikleri için `openclaw onboard` kullanın.

## Klasik sihirbaz: QuickStart ve Advanced

Tam sihirbazı açmak için `openclaw onboard --classic` çalıştırın. Sihirbaz, **QuickStart** (varsayılanlar) ile **Advanced** (tam denetim) arasında bir seçimle başlar. Klasik akışı seçmek ve bu istemi atlamak için `--flow quickstart` veya `--flow advanced` (`manual` takma adı) iletin.

<Tabs>
  <Tab title="QuickStart (varsayılanlar)">
    - Yerel Gateway, geri döngü bağlaması
    - Varsayılan çalışma alanı (veya mevcut çalışma alanı)
    - Gateway portu **18789**
    - Gateway kimlik doğrulaması **Token** (geri döngüde bile otomatik oluşturulur)
    - Araç politikası: yeni kurulumlar için `tools.profile: "coding"` (mevcut açık profil korunur)
    - DM oturumları: İlk katılım, açıkça ayarlanmış bir `session.dmScope` değerini korur; aksi hâlde değeri ayarlanmamış bırakır. Böylece `"main"` varsayılanı, tüm kanallardaki doğrudan mesajları ajanın kayan ana oturumunda tutar; bu, kişisel ajan varsayılanıdır. Paylaşılan veya çok kullanıcılı gelen kutuları için `"per-channel-peer"` kullanın; `openclaw security audit`, çok kullanıcılı DM trafiği algıladığında yalıtım önerir. Ayrıntılar: [CLI kurulum başvurusu](/tr/start/wizard-cli-reference#outputs-and-internals)
    - Tailscale erişimi **Kapalı**
    - Telegram ve WhatsApp DM'leri varsayılan olarak **izin listesi** kullanır: Telegram sayısal bir Telegram kullanıcı kimliği, WhatsApp ise telefon numarası ister

  </Tab>
  <Tab title="Advanced (tam denetim)">
    - Tüm adımları sunar: mod, çalışma alanı, Gateway, kanallar, daemon, Skills

  </Tab>
</Tabs>

Uzak mod (`--mode remote`) her zaman gelişmiş akışı kullanır; yalnızca bu makineyi başka bir yerdeki Gateway'e bağlanacak şekilde yapılandırır ve uzak ana makinede hiçbir şey kurmaz veya değiştirmez.

## Klasik ilk katılımın yapılandırdıkları

Yerel mod (varsayılan) şu adımlardan geçer:

1. **Model/Kimlik Doğrulama** - Özel Sağlayıcı (OpenAI uyumlu, OpenAI Responses uyumlu, Anthropic uyumlu veya Bilinmiyor otomatik algılama) dâhil bir sağlayıcı kimlik doğrulama akışı (API anahtarı, OAuth veya sağlayıcıya özgü manuel kimlik doğrulama) seçin. Varsayılan bir model seçin. Yeni OpenAI API anahtarı kurulumu varsayılan olarak `openai/gpt-5.6` kullanır (çıplak doğrudan API kimliği Sol'a çözümlenir); yeni ChatGPT/Codex kurulumu varsayılan olarak `openai/gpt-5.6-sol` kullanır. Kurulumun yeniden çalıştırılması, `openai/gpt-5.5` dâhil mevcut açık modeli korur. Hesap GPT-5.6'yı sunmuyorsa `openai/gpt-5.5` seçeneğini açıkça belirleyin. Güvenlik notu: Bu ajan araç çalıştıracak veya webhook/kanca içeriğini işleyecekse mevcut en güçlü yeni nesil modeli tercih edin ve araç politikasını sıkı tutun; daha zayıf veya eski katmanlarda istem enjeksiyonu daha kolaydır. Etkileşimsiz çalıştırmalarda `--secret-input-mode ref`, düz metin API anahtarı değerleri yerine ortam destekli başvuruları saklar; başvurulan ortam değişkeni önceden ayarlanmış olmalıdır, aksi takdirde ilk katılım hemen başarısız olur. Etkileşimli sır başvurusu modu, kaydetmeden önce hızlı bir ön kontrolle bir ortam değişkenine veya yapılandırılmış sağlayıcı başvurusuna (`file` ya da `exec`) işaret edebilir. Model/kimlik doğrulama kurulumundan sonra sihirbaz isteğe bağlı bir canlı tamamlama testi sunar; başarısızlık durumunda model/kimlik doğrulama kurulumuna bir kez dönülebilir veya klasik sihirbazın geri kalanı engellenmeden başarısızlık yok sayılabilir. Başarısızlığın yok sayılması OpenClaw'ın kilidini açmaz; konuşmalı kurulum yine de başarılı bir çıkarım denetimi gerektirir.
2. **Çalışma alanı** - ajan dosyalarının dizini (varsayılan `~/.openclaw/workspace`). Başlangıç dosyalarını oluşturur.
3. **Gateway** - port, bağlama adresi, kimlik doğrulama modu, Tailscale erişimi. Etkileşimli token modunda düz metin token depolamayı (varsayılan) seçin veya SecretRef kullanmayı tercih edin. Etkileşimsiz SecretRef yolu: `--gateway-token-ref-env <ENV_VAR>`.
4. **Kanallar** - Discord, Feishu, Google Chat, iMessage, Mattermost, Microsoft Teams, QQ Bot, Signal, Slack, Telegram, WhatsApp ve daha fazlası dâhil yerleşik ve resmî plugin sohbet kanalları.
5. **Daemon** - bir LaunchAgent (macOS), systemd kullanıcı birimi (Linux/WSL2) veya kullanıcı başına Startup klasörü geri dönüşüne sahip yerel bir Windows Scheduled Task kurar. Token kimlik doğrulaması gerekliyse ve `gateway.auth.token` SecretRef ile yönetiliyorsa daemon kurulumu bunu doğrular ancak çözümlenmiş token'ı gözetmen hizmeti ortamı meta verilerinde kalıcı hâle getirmez; çözümlenmemiş bir SecretRef, yönlendirme sağlayarak kurulumu engeller. `gateway.auth.mode` ayarlanmamışken hem `gateway.auth.token` hem de `gateway.auth.password` ayarlanmışsa mod açıkça ayarlanana kadar kurulum engellenir.
6. **Sistem durumu denetimi** - Gateway'i başlatır ve erişilebilir olduğunu doğrular.
7. **Skills** - önerilen becerileri ve isteğe bağlı bağımlılıklarını kurar.

<Note>
İlk katılımın yeniden çalıştırılması, açıkça **Reset** seçeneğini belirlemediğiniz (veya `--reset` iletmediğiniz) sürece hiçbir şeyi **silmez**. CLI `--reset` varsayılan olarak yapılandırmayı, kimlik bilgilerini ve oturumları sıfırlar; çalışma alanını da kaldırmak için `--reset-scope full` kullanın. Yapılandırma geçersizse veya eski anahtarlar içeriyorsa ilk katılım önce `openclaw doctor` çalıştırmanızı ister.
</Note>

`--flow import`, yeni kurulum yerine klasik sihirbazda algılanan bir geçiş akışını (örneğin Hermes) çalıştırır; [Geçiş](/tr/cli/migrate) ve [Kurulum](/tr/install/migrating-hermes) altındaki geçiş kılavuzlarına bakın. `openclaw onboard --modern`, [OpenClaw](/tr/cli/openclaw) için bir uyumluluk takma adıdır. `openclaw setup` ile aynı çıkarım geçidini kullanır: doğrulanmış çıkarım asistanı başlatırken etkileşimli bir başarısızlık yönlendirmeli çıkarım kurulumuna döner.

## Başka bir ajan ekleme

Kendi çalışma alanına, oturumlarına ve kimlik doğrulama profillerine sahip ayrı bir ajan oluşturmak için `openclaw agents add <name>` kullanın. `--workspace` olmadan çalıştırıldığında ad, çalışma alanı, kimlik doğrulama, kanallar ve bağlamalar için etkileşimli bir akış başlatılır; bu, tam `openclaw onboard` sihirbazı değildir.

Ayarladıkları:

- `agents.entries.*.name`
- `agents.entries.*.workspace`
- `agents.entries.*.agentDir`

Notlar:

- Varsayılan çalışma alanı: `~/.openclaw/workspace-<agentId>` (veya ayarlanmışsa `agents.defaults.workspace` altında).
- Gelen mesajları bu ajana yönlendirmek için `bindings` ekleyin (ilk katılım bunu sizin için yapabilir).
- Etkileşimsiz bayraklar: `--model`, `--agent-dir`, `--bind`, `--non-interactive`.

## Tam başvuru

Ayrıntılı adım adım davranış ve yapılandırma çıktıları için [CLI kurulum başvurusuna](/tr/start/wizard-cli-reference) bakın.
Etkileşimsiz örnekler için [CLI otomasyonuna](/tr/start/wizard-cli-automation) bakın.
Tüm bayrakların başvurusu için [`openclaw onboard`](/tr/cli/onboard) sayfasına bakın.

## İlgili belgeler

- CLI komut başvurusu: [`openclaw onboard`](/tr/cli/onboard)
- İlk katılıma genel bakış: [İlk katılıma genel bakış](/tr/start/onboarding-overview)
- macOS uygulamasında ilk katılım: [İlk katılım](/tr/start/onboarding)
- Ajanın ilk çalıştırma ritüeli: [Ajan Başlatma](/tr/start/bootstrapping)
