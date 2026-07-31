---
read_when:
    - Yeni kurulum, takılı kalan ilk katılım veya ilk çalıştırma hataları
    - Kimlik doğrulama ve sağlayıcı aboneliklerini seçme
    - docs.openclaw.ai adresine erişilemiyor, kontrol paneli açılamıyor, kurulum takıldı
sidebarTitle: First-run FAQ
summary: 'SSS: hızlı başlangıç ve ilk çalıştırma kurulumu — yükleme, ilk katılım, kimlik doğrulama, abonelikler, başlangıç hataları'
title: 'SSS: ilk çalıştırma kurulumu'
x-i18n:
    generated_at: "2026-07-26T23:43:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e1c93b89da625ae5f092db854c9b74adc005be75dd913af4bf89ed1a4f35396a
    source_path: help/faq-first-run.md
    workflow: 16
---

Hızlı başlangıç ve ilk çalıştırma soru-cevapları. Günlük işlemler, modeller, kimlik doğrulama, oturumlar
ve sorun giderme için ana [SSS](/tr/help/faq) sayfasına bakın.

## Hızlı başlangıç ve ilk çalıştırma kurulumu

<AccordionGroup>
  <Accordion title="Takıldım, sorunu çözmenin en hızlı yolu">
    **Makinenizi görebilen** yerel bir yapay zekâ aracısı kullanın. "Takıldım" durumlarının çoğu,
    uzaktaki bir yardımcının inceleyemeyeceği **yerel yapılandırma veya ortam sorunlarından** kaynaklanır;
    bu nedenle bu yöntem Discord'da sormaktan daha etkilidir.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    Kodu ve belgeleri okuyabilmesi ve çalıştırdığınız tam sürüm hakkında akıl yürütebilmesi için
    aracıya özelleştirilebilir (git) kurulum aracılığıyla kaynak kodun tamamını verin:

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Aracıdan düzeltmeyi adım adım planlayıp denetlemesini, ardından yalnızca gerekli
    komutları yürütmesini isteyin; daha küçük farkların denetlenmesi daha kolaydır.

    Yardım isterken (Discord'da veya bir GitHub kaydında) şu çıktıları paylaşın:

    | Komut | Gösterdikleri |
    | --- | --- |
    | `openclaw status` | Gateway/aracı durumu + temel yapılandırma anlık görüntüsü |
    | `openclaw status --all` | Yapıştırılabilir, tam salt okunur tanılama |
    | `openclaw models status` | Sağlayıcı kimlik doğrulaması + model kullanılabilirliği |
    | `openclaw doctor` | Yaygın yapılandırma/durum sorunlarını doğrular ve onarır |
    | `openclaw logs --follow` | Canlı günlük takibi |
    | `openclaw gateway status --deep` | Ayrıntılı Gateway/yapılandırma/plugin durum denetimi |
    | `openclaw health --verbose` | Ayrıntılı durum raporu |

    Gerçek bir hata veya düzeltme mi buldunuz? Bir kayıt açın veya PR gönderin:
    [Kayıtlar](https://github.com/openclaw/openclaw/issues) /
    [Pull request'ler](https://github.com/openclaw/openclaw/pulls).

    Hızlı hata ayıklama döngüsü: [Bir şey bozulduğunda ilk 60 saniye](/tr/help/faq#first-60-seconds-if-something-is-broken).
    Kurulum belgeleri: [Kurulum](/tr/install), [Kurucu bayrakları](/tr/install/installer), [Güncelleme](/tr/install/updating).

  </Accordion>

  <Accordion title="Heartbeat sürekli atlanıyor. Atlama nedenleri ne anlama geliyor?">
    | Atlama nedeni | Anlamı |
    | --- | --- |
    | `quiet-hours` | Yapılandırılmış etkin saatler aralığının dışında |
    | `empty-heartbeat-file` | Heartbeat izleyicisinin taslağı mevcut ancak yalnızca boşluk, yorum, başlık, çit veya boş kontrol listesi iskeleti içeriyor |
    | `alerts-disabled` | Tüm Heartbeat görünürlüğü kapalı (`showOk`, `showAlerts` ve `useIndicator` seçeneklerinin tümü devre dışı) |

    Eski Heartbeat `tasks:` blokları, `openclaw doctor --fix` ile bağımsız olarak zamanlanan cron işlerine taşınır.

    Belgeler: [Heartbeat](/tr/gateway/heartbeat), [Otomasyon](/tr/automation).

  </Accordion>

  <Accordion title="OpenClaw'u kurmak ve ayarlamak için önerilen yöntem">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    Kaynaktan (katkıda bulunanlar/geliştiriciler):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build
    openclaw onboard
    ```

    Henüz genel kurulum yok mu? Bunun yerine `pnpm openclaw onboard` çalıştırın. Control UI varlıkları
    eksikse katılım süreci bunları kendisi oluşturmaya çalışır ve başarısız olursa `pnpm ui:build` kullanır.

  </Accordion>

  <Accordion title="Katılım sürecinden sonra gösterge panelini nasıl açarım?">
    Katılım süreci, kurulumdan hemen sonra tarayıcınızı temiz (token içermeyen) bir gösterge paneli
    URL'sinde açar ve bağlantıyı özette yazdırır. Bu sekmeyi açık tutun; açılmadıysa yazdırılan
    URL'yi aynı makinede kopyalayıp yapıştırın.
  </Accordion>

  <Accordion title="Gösterge panelinde localhost ve uzak bağlantı için nasıl kimlik doğrularım?">
    **Localhost (aynı makine):**

    - `http://127.0.0.1:18789/` adresini açın.
    - Paylaşılan gizli bilgiyle kimlik doğrulaması isterse yapılandırılmış token'ı veya parolayı Control UI ayarlarına yapıştırın.
    - Token kaynağı: `gateway.auth.token` (veya `OPENCLAW_GATEWAY_TOKEN`).
    - Parola kaynağı: `gateway.auth.password` (veya `OPENCLAW_GATEWAY_PASSWORD`).
    - Henüz paylaşılan gizli bilgi yapılandırılmadı mı? `openclaw doctor --generate-gateway-token` (veya `openclaw doctor --fix --generate-gateway-token`) çalıştırın.

    **Localhost üzerinde değilse:**

    - **Tailscale Serve** (önerilen): bağlamayı geri döngüde tutun, `openclaw gateway --tailscale serve` çalıştırın ve `https://<magicdns>/` adresini açın. `gateway.auth.allowTailscale: true` ile kimlik üstbilgileri Control UI/WebSocket kimlik doğrulamasını karşılar (paylaşılan gizli bilgi yapıştırılmaz, güvenilir bir Gateway ana makinesi varsayılır); özel giriş `none` veya güvenilir proxy HTTP kimlik doğrulamasını bilinçli olarak kullanmadığınız sürece HTTP API'leri yine paylaşılan gizli bilgiyle kimlik doğrulaması gerektirir.
      Aynı istemciden gelen eşzamanlı hatalı kimlik doğrulamalı Serve girişimleri, başarısız kimlik doğrulama sınırlayıcısı bunları kaydetmeden önce sıralı hâle getirilir; bu nedenle ikinci bir hatalı yeniden deneme zaten `retry later` gösterebilir.
    - **Tailnet bağlaması**: `openclaw gateway --bind tailnet --token "<token>"` çalıştırın (veya parola kimlik doğrulamasını yapılandırın), `http://<tailscale-ip>:18789/` adresini açın ve eşleşen paylaşılan gizli bilgiyi gösterge paneli ayarlarına yapıştırın.
    - **Kimlik farkındalıklı ters proxy**: Gateway'i güvenilir bir proxy'nin arkasında tutun, `gateway.auth.mode: "trusted-proxy"` ayarlayın ve proxy URL'sini açın. Aynı ana makinedeki geri döngü proxy'leri açıkça `gateway.auth.trustedProxy.allowLoopback: true` gerektirir.
    - **SSH tüneli**: `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host`, ardından `http://127.0.0.1:18789/` adresini açın. Paylaşılan gizli bilgiyle kimlik doğrulaması tünel üzerinden de geçerlidir; istenirse yapılandırılmış token'ı veya parolayı yapıştırın.

    Bağlama modları ve kimlik doğrulama ayrıntıları için [Gösterge paneli](/tr/web/dashboard) ve [Web yüzeyleri](/tr/web) sayfalarına bakın.

  </Accordion>

  <Accordion title="Sohbet onayları için neden iki exec onay yapılandırması var?">
    Farklı katmanları denetlerler:

    - `approvals.exec` - onay istemlerini sohbet hedeflerine iletir.
    - `channels.<channel>.execApprovals` - ilgili kanalı exec onayları için yerel bir onay istemcisi yapar.

    Ana makinenin exec politikası gerçek onay kapısı olmaya devam eder; sohbet yapılandırması yalnızca
    istemlerin nerede görüneceğini ve kişilerin bunları nasıl yanıtlayacağını denetler.

    Her ikisine birden nadiren ihtiyaç duyulur:

    - Sohbet zaten komutları ve yanıtları destekliyorsa aynı sohbetteki `/approve` paylaşılan yol üzerinden çalışır.
    - Desteklenen yerel bir kanal onaylayıcıları güvenle çıkarabiliyorsa OpenClaw, `channels.<channel>.execApprovals.enabled` ayarlanmamış veya `"auto"` olduğunda önce DM kullanan yerel onayları otomatik olarak etkinleştirir.
    - Yerel onay kartları/düğmeleri kullanılabiliyorsa birincil arayüz budur; yalnızca araç sonucu sohbet onaylarının kullanılamadığını söylüyorsa elle kullanılan `/approve` komutundan söz edin.
    - Yalnızca istemlerin diğer sohbetlere veya açıkça belirtilmiş operasyon odalarına da ulaşması gerekiyorsa `approvals.exec` kullanın.
    - Yalnızca onay istemlerinin kaynak odaya/konuya geri gönderilmesini istiyorsanız `channels.<channel>.execApprovals.target: "channel"` veya `"both"` kullanın.
    - Plugin onayları ayrıdır: varsayılan olarak aynı sohbette `/approve`, isteğe bağlı `approvals.plugin` iletimi kullanılır ve yalnızca bazı yerel kanallar bunlar için de yerel işlemeyi korur.

    Kısaca: iletme yönlendirme içindir, yerel istemci yapılandırması ise kanala özgü daha zengin bir kullanıcı deneyimi içindir.
    [Exec Onayları](/tr/tools/exec-approvals) sayfasına bakın.

  </Accordion>

  <Accordion title="Hangi çalışma zamanına ihtiyacım var?">
    Node **22.22.3+**, **24.15+** veya **25.9+** gereklidir (Node 24 önerilir). `pnpm`, deponun paket yöneticisidir.
    Bun bağımlılıkları kurabilir ve paket betiklerini çalıştırabilir ancak `node:sqlite` içermediğinden OpenClaw CLI veya Gateway'i çalıştıramaz.
  </Accordion>

  <Accordion title="Raspberry Pi üzerinde çalışır mı?">
    Evet, ancak önce RAM'i kontrol edin: Pi 5 ve Pi 4 (2 GB+) en uygun seçeneklerdir; Pi 3B+ (1 GB) çalışır ancak yavaştır; Pi Zero 2 W (512 MB) önerilmez.

    | Model | RAM | Uygunluk |
    | --- | --- | --- |
    | Pi 5 | 4/8 GB | En iyi |
    | Pi 4 | 4 GB | İyi |
    | Pi 4 | 2 GB | Uygun, takas alanı ekleyin |
    | Pi 4 | 1 GB | Kısıtlı |
    | Pi 3B+ | 1 GB | Yavaş |
    | Pi Zero 2 W | 512 MB | Önerilmez |

    Mutlak minimum: 1 GB RAM, 1 çekirdek, 500 MB boş disk, 64 bit işletim sistemi. Pi yalnızca
    Gateway'i çalıştırdığından (modeller bulut API'lerini çağırır), mütevazı bir Pi bile yükü kaldırabilir.

    Küçük bir Pi/VPS yalnızca Gateway'i de barındırabilir; yerel ekran/kamera/tuval veya komut yürütme için
    dizüstü bilgisayarınızda/telefonunuzda **Node'ları** eşleştirebilirsiniz. [Node'lar](/tr/nodes) sayfasına bakın.

    Tam kurulum kılavuzu: [Raspberry Pi](/tr/install/raspberry-pi).

  </Accordion>

  <Accordion title="Raspberry Pi kurulumları için öneriler var mı?">
    - **64 bit** işletim sistemi kullanın; 32 bit Raspberry Pi OS kullanmayın.
    - 2 GB veya daha küçük kartlara takas alanı ekleyin.
    - Performans ve kullanım ömrü için SD kart yerine **USB SSD** tercih edin.
    - Günlükleri görebilmek ve hızlı güncelleme yapabilmek için özelleştirilebilir (git) kurulumu tercih edin.
    - Kanallar/Skills olmadan başlayın ve bunları tek tek ekleyin.
    - Tuhaf ikili dosya hataları ("exec format error") genellikle isteğe bağlı bir beceri aracının ARM64 derlemesinin eksik olmasından kaynaklanır.

    Tam kılavuz: [Raspberry Pi](/tr/install/raspberry-pi). Ayrıca [Linux](/tr/platforms/linux) sayfasına bakın.

  </Accordion>

  <Accordion title="Wake up my friend ekranında takılıyor / katılım süreci tamamlanmıyor. Ne yapmalıyım?">
    Bu ekran, Gateway'in erişilebilir ve kimliği doğrulanmış olmasına bağlıdır. Bir model sağlayıcısı
    yapılandırıldığında TUI, ilk açılışta otomatik olarak "Wake up, my friend!" mesajını da gönderir.
    Model/kimlik doğrulama kurulumunu atladıysanız katılım süreci "Model auth missing" notunu gösterir ve
    hiçbir şey göndermeden TUI'yi açar; `openclaw configure --section model` ile bir sağlayıcı ekleyin.
    Uyandırma satırını görüyor ancak **yanıt alamıyorsanız** ve token sayısı 0'da kalıyorsa aracı hiç çalışmamıştır.

    1. Gateway'i yeniden başlatın:

    ```bash
    openclaw gateway restart
    ```

    2. Durumu ve kimlik doğrulamasını kontrol edin:

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. Hâlâ takılıyor mu? Şunu çalıştırın:

    ```bash
    openclaw doctor
    ```

    Gateway uzaktaysa tünel/Tailscale bağlantısının etkin olduğunu ve kullanıcı arayüzünün
    doğru Gateway'i gösterdiğini doğrulayın. [Uzaktan erişim](/tr/gateway/remote) sayfasına bakın.

  </Accordion>

  <Accordion title="Katılım sürecini yeniden yapmadan kurulumumu yeni bir makineye taşıyabilir miyim?">
    Evet. **Durum dizinini** ve **çalışma alanını** kopyalayın, ardından Doctor'ı bir kez çalıştırın:

    1. OpenClaw'u yeni makineye kurun.
    2. `$OPENCLAW_STATE_DIR` dizinini (varsayılan: `~/.openclaw`) eski makineden kopyalayın.
    3. Çalışma alanınızı (varsayılan: `~/.openclaw/workspace`) kopyalayın.
    4. `openclaw doctor` çalıştırın ve Gateway hizmetini yeniden başlatın.

    Bu işlem yapılandırmayı, kimlik doğrulama profillerini, WhatsApp kimlik bilgilerini, oturumları ve belleği korur;
    **her iki** konumu da kopyaladığınız sürece botunuz tamamen aynı kalır. Uzak modda oturum deposunun ve
    çalışma alanının sahibi Gateway ana makinesidir.

    **Önemli:** yalnızca çalışma alanınızı GitHub'a kaydedip gönderirseniz
    **bellek + önyükleme dosyalarını** yedeklersiniz ancak oturum geçmişini veya kimlik doğrulama bilgilerini yedeklemezsiniz.
    Bunlar `~/.openclaw/` altında bulunur (örneğin `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`).

    İlgili konular: [Taşıma](/tr/install/migrating), [Dosyaların diskte bulunduğu yerler](/tr/help/faq#where-things-live-on-disk),
    [Aracı çalışma alanı](/tr/concepts/agent-workspace), [Doctor](/tr/gateway/doctor),
    [Uzak mod](/tr/gateway/remote).

  </Accordion>

  <Accordion title="En son sürümdeki yenilikleri nerede görebilirim?">
    GitHub değişiklik günlüğüne bakın:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    En yeni girdiler en üsttedir. En üstteki bölüm **Yayımlanmadı** ise sonraki tarihli
    bölüm yayımlanmış en son sürümdür. Girdiler **Öne Çıkanlar**, **Değişiklikler**
    ve **Düzeltmeler** altında gruplandırılır (gerektiğinde belgeler/diğer bölümler de bulunur).

  </Accordion>

  <Accordion title="docs.openclaw.ai adresine erişilemiyor (SSL hatası)">
    Bazı Comcast/Xfinity bağlantıları, Xfinity Advanced Security aracılığıyla `docs.openclaw.ai` adresini
    yanlışlıkla engeller. Bu özelliği devre dışı bırakın veya `docs.openclaw.ai` adresini izin verilenler
    listesine ekleyip yeniden deneyin. Engelin kaldırılmasına yardımcı olun:
    [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    Hâlâ engellendiniz mi? Belgeler GitHub'da yansıtılıyor:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="Kararlı sürüm ile beta arasındaki fark">
    **Kararlı sürüm** ve **beta**, ayrı kod hatları değil, **npm dist-tag'leridir**:

    - `latest` = kararlı sürüm
    - `beta` = test için erken derleme (beta yoksa veya mevcut kararlı sürümden eskiyse `latest` sürümüne geri döner)

    Kararlı bir sürüm genellikle önce **beta** kanalına gelir, ardından açık bir yükseltme adımı
    sürüm numarasını değiştirmeden aynı sürümü `latest` kanalına taşır. Bakım sorumluları
    doğrudan `latest` kanalında da yayımlayabilir. Bu nedenle yükseltme sonrasında beta ve kararlı sürüm
    **aynı sürümü** gösterebilir.

    Nelerin değiştiğine bakın: [CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md).

    Tek satırlık kurulum komutları ve beta ile dev arasındaki fark için sonraki akordeona bakın.

  </Accordion>

  <Accordion title="Beta sürümünü nasıl kurarım ve beta ile dev arasındaki fark nedir?">
    **Beta**, `beta` npm dist-tag'idir (yükseltme sonrasında `latest` ile eşleşebilir).
    **Dev**, `main` dalının hareketli en güncel durumudur (git); npm'de yayımlandığında `dev` dist-tag'ini kullanır.

    Tek satırlık komutlar (macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Windows yükleyicisi (PowerShell): `iwr -useb https://openclaw.ai/install.ps1 | iex`

    Ayrıntılı bilgi: [Geliştirme kanalları](/tr/install/development-channels) ve [Yükleyici bayrakları](/tr/install/installer).

  </Accordion>

  <Accordion title="En son bileşenleri nasıl deneyebilirim?">
    İki seçenek vardır:

    1. **Dev kanalı (mevcut kurulum):**

    ```bash
    openclaw update --channel dev
    ```

    Bu işlem `main` dalının bir git çalışma kopyasına geçer, upstream üzerine rebase eder, derler ve
    CLI'yi bu çalışma kopyasından kurar.

    2. **Değiştirilebilir (git) kurulum (yeni makine):**

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Manuel klonlamayı tercih edin:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    Belgeler: [Güncelleme](/tr/cli/update), [Geliştirme kanalları](/tr/install/development-channels), [Kurulum](/tr/install).

  </Accordion>

  <Accordion title="Kurulum ve ilk yapılandırma genellikle ne kadar sürer?">
    Yaklaşık süreler:

    - **Kurulum:** 2-5 dakika.
    - **QuickStart ilk yapılandırması:** birkaç dakika (geri döngü Gateway'i, otomatik token, varsayılan çalışma alanı).
    - **Gelişmiş/tam ilk yapılandırma:** sağlayıcı oturumu açma, kanal eşleştirme, daemon kurulumu, ağ indirmeleri veya Skills ek kurulum gerektirdiğinde daha uzun sürer.

    Sihirbaz bu zaman çizelgesini baştan gösterir. İsteğe bağlı adımları atlayıp daha sonra
    `openclaw configure` ile geri dönün.

    Takıldı mı? Yukarıdaki [Takıldım](#quick-start-and-first-run-setup) bölümüne bakın.

  </Accordion>

  <Accordion title="Yükleyici takıldı mı? Nasıl daha fazla geri bildirim alabilirim?">
    `--verbose` ile yeniden çalıştırın:

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --verbose
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    `install.ps1` için özel bir ayrıntılı çıktı anahtarı yoktur; bunun yerine onu `Set-PSDebug -Trace 1` /
    `-Trace 0` içine alın. Bayrakların tam listesi: [Yükleyici bayrakları](/tr/install/installer).

  </Accordion>

  <Accordion title="Windows kurulumu git bulunamadı veya openclaw tanınmıyor diyor">
    Windows'ta sık karşılaşılan iki sorun:

    **1) npm hatası: spawn git / git bulunamadı**

    - **Git for Windows** uygulamasını kurun ve `git` öğesinin PATH üzerinde olduğundan emin olun.
    - PowerShell'i kapatıp yeniden açın, ardından yükleyiciyi yeniden çalıştırın.

    **2) Kurulumdan sonra openclaw tanınmıyor**

    - npm genel ikili dosya klasörünüz PATH üzerinde değil.
    - Kontrol edin: `npm config get prefix`.
    - Bu dizini kullanıcı PATH'inize ekleyin (`\bin` son eki gerekmez; çoğu sistemde `%AppData%\npm` konumundadır).
    - PowerShell'i kapatıp yeniden açın.

    Masaüstü uygulamasını mı tercih ediyorsunuz? **Windows Hub** kullanın. Yalnızca terminal kurulumu için hem PowerShell
    yükleyicisi hem de WSL2 Gateway yolları desteklenir. Belgeler: [Windows](/tr/platforms/windows).

  </Accordion>

  <Accordion title="Windows exec çıktısında bozuk Çince metin görünüyor; ne yapmalıyım?">
    Bu durum genellikle yerel Windows kabuklarındaki konsol kod sayfası uyuşmazlığından kaynaklanır.

    Belirtiler: `system.run`/`exec` çıktısında Çince karakterler bozuk görünür; aynı komut
    başka bir terminal profilinde düzgün görünür.

    PowerShell'de geçici çözüm:

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    Ardından Gateway'i yeniden başlatıp tekrar deneyin:

    ```powershell
    openclaw gateway restart
    ```

    Bu sorun en son OpenClaw sürümünde hâlâ oluşuyor mu? Takip edin/bildirin: [Sorun #30640](https://github.com/openclaw/openclaw/issues/30640).

  </Accordion>

  <Accordion title="Belgeler sorumu yanıtlamadı; nasıl daha iyi bir yanıt alabilirim?">
    Tüm kaynak koduna ve belgelere yerel olarak sahip olmak için değiştirilebilir (git) kurulumu kullanın, ardından
    botunuza (veya Claude/Codex'e) depoyu okuyup kesin yanıt verebilmesi için **o klasörden** sorun.

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Ayrıntılı bilgi: [Kurulum](/tr/install) ve [Yükleyici bayrakları](/tr/install/installer).

  </Accordion>

  <Accordion title="OpenClaw'u Linux'a nasıl kurarım?">
    - Linux hızlı yolu + hizmet kurulumu: [Linux](/tr/platforms/linux).
    - Tam kılavuz: [Başlarken](/tr/start/getting-started).
    - Yükleyici + güncellemeler: [Kurulum ve güncellemeler](/tr/install/updating).

  </Accordion>

  <Accordion title="OpenClaw'u bir VPS'ye nasıl kurarım?">
    Herhangi bir Linux VPS kullanılabilir. Sunucuya kurun, ardından Gateway'e SSH/Tailscale üzerinden erişin.

    Kılavuzlar: [exe.dev](/tr/install/exe-dev), [Hetzner](/tr/install/hetzner), [Fly.io](/tr/install/fly).
    Uzaktan erişim: [Uzak Gateway](/tr/gateway/remote).

  </Accordion>

  <Accordion title="Bulut/VPS kurulum kılavuzları nerede?">
    Yaygın sağlayıcıların yer aldığı barındırma merkezi:

    - [VPS barındırma](/tr/vps) (tüm sağlayıcılar tek yerde)
    - [Fly.io](/tr/install/fly)
    - [Hetzner](/tr/install/hetzner)
    - [exe.dev](/tr/install/exe-dev)

    Bulutta **Gateway sunucuda çalışır** ve dizüstü bilgisayarınızdan/telefonunuzdan ona
    Control UI (veya Tailscale/SSH) üzerinden erişirsiniz. Durumunuz ve çalışma alanınız sunucuda bulunur; bu nedenle
    ana makineyi doğruluk kaynağı olarak kabul edin ve yedekleyin.

    Gateway bulutta kalırken dizüstü bilgisayarınızda yerel
    ekran/kamera/canvas veya komut yürütme için **Node'ları** (Mac/iOS/Android/headless) bu bulut Gateway'iyle eşleştirin.

    Merkez: [Platformlar](/tr/platforms). Uzaktan erişim: [Uzak Gateway](/tr/gateway/remote).
    Node'lar: [Node'lar](/tr/nodes), [Node CLI](/tr/cli/nodes).

  </Accordion>

  <Accordion title="OpenClaw'dan kendisini güncellemesini isteyebilir miyim?">
    Mümkündür ancak önerilmez. Güncelleme akışı Gateway'i yeniden başlatabilir (etkin
    oturumu sonlandırır), temiz bir git çalışma kopyası gerektirebilir ve onay isteyebilir.
    Güncellemeleri operatör olarak bir kabuktan çalıştırmak daha güvenlidir.

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|extended-stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    Bir agent üzerinden otomatikleştirme:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    Belgeler: [Güncelleme](/tr/cli/update), [Güncelleme](/tr/install/updating).

  </Accordion>

  <Accordion title="İlk yapılandırma gerçekte ne yapar?">
    `openclaw onboard` önerilen kurulum yoludur. **Yerel modda** şu adımları uygular:

    1. **Model/Kimlik doğrulama** - sağlayıcı OAuth'ı, API anahtarları veya manuel kimlik doğrulama (LM Studio gibi yerel seçenekler dâhil); varsayılan modeli seçer.
    2. **Çalışma alanı** - konum + başlangıç dosyaları.
    3. **Gateway** - bağlantı noktası, bağlama adresi, kimlik doğrulama modu, Tailscale üzerinden erişim.
    4. **Kanallar** - yerleşik ve resmî Plugin sohbet kanalları: iMessage, Discord, Feishu, Google Chat, Mattermost, Microsoft Teams, QQ Bot, Signal, Slack, Telegram, WhatsApp ve diğerleri.
    5. **Daemon** - LaunchAgent (macOS), systemd kullanıcı birimi (Linux/WSL2) veya yerel Windows Scheduled Task.
    6. **Sistem durumu denetimi** - Gateway'i başlatır ve çalıştığını doğrular.
    7. **Skills** - önerilen becerileri ve isteğe bağlı bağımlılıkları kurar.

    Süre beklentilerini baştan belirler ve yapılandırılmış modeliniz bilinmiyorsa
    veya kimlik doğrulaması eksikse uyarır. Tam açıklama: [İlk yapılandırma (CLI)](/tr/start/wizard).

  </Accordion>

  <Accordion title="Bunu çalıştırmak için Claude veya OpenAI aboneliğine ihtiyacım var mı?">
    Hayır. Verilerinizin cihazınızda kalması için OpenClaw'u **API anahtarlarıyla**
    (Anthropic/OpenAI/diğerleri) veya **yalnızca yerel modellerle** çalıştırın. Abonelikler
    (Claude Pro/Max, ChatGPT/Codex), bu sağlayıcılarda kimlik doğrulaması için isteğe bağlı yöntemlerdir.

    Anthropic için: Bir **API anahtarı**, standart kullandıkça öde faturalandırması sağlar; **Claude CLI**
    aynı ana makinedeki mevcut Claude Code oturumunu yeniden kullanır. Anthropic şu anda
    Claude CLI'nin etkileşimsiz `claude -p` yolunu, aboneliğinizin plan sınırlarını
    kullanmaya devam eden Agent SDK/programatik kullanım olarak değerlendirir; abonelik davranışına
    güvenmeden önce güncel Anthropic faturalandırma belgelerini kontrol edin. Uzun süreli Gateway ana makineleri ve paylaşılan
    otomasyon için Anthropic API anahtarı daha öngörülebilir bir seçimdir.

    OpenAI Codex OAuth (ChatGPT/Codex aboneliği), agent modelleri için tamamen desteklenir.
    OpenClaw ayrıca **Qwen Cloud Coding Plan**, **MiniMax Coding Plan**
    ve **Z.AI / GLM Coding Plan** dâhil barındırılan abonelik tarzı seçenekleri destekler.

    Belgeler: [Anthropic](/tr/providers/anthropic), [OpenAI](/tr/providers/openai),
    [Qwen Cloud](/tr/providers/qwen), [MiniMax](/tr/providers/minimax), [Z.AI (GLM)](/tr/providers/zai),
    [Yerel modeller](/tr/gateway/local-models), [Modeller](/tr/concepts/models).

  </Accordion>

  <Accordion title="API anahtarı olmadan Claude Max aboneliğini kullanabilir miyim?">
    Evet. OpenClaw, Pro/Max/Team/Enterprise planlarında Claude CLI'nin yeniden kullanılmasını destekler. Anthropic
    şu anda OpenClaw'un kullandığı `claude -p` yolunu, ayrı bir ücretsiz kullanım hakkı olarak değil,
    planınızın sınırlarına tabi abonelik planı kullanımı olarak değerlendirir; güncel faturalandırma ayrıntıları ve
    Anthropic'in kendi destek makalelerine bağlantılar için [Anthropic](/tr/providers/anthropic) sayfasına bakın.
    En öngörülebilir sunucu tarafı kurulumu için bunun yerine Anthropic API anahtarı kullanın.
  </Accordion>

  <Accordion title="Claude aboneliğiyle kimlik doğrulamayı (Claude Pro veya Max) destekliyor musunuz?">
    Evet, Claude CLI'nin yeniden kullanılması yoluyla. Anthropic'in `claude -p`/Agent SDK kullanımına yönelik faturalandırma yaklaşımı
    zaman içinde değişmiştir; belirli bir faturalandırma davranışına güvenmeden önce güncel durum ve
    Anthropic'in destek makalelerine giden tarihli bağlantılar için [Anthropic](/tr/providers/anthropic) sayfasına
    bakın.

    Anthropic setup-token kimlik doğrulaması da hâlâ desteklenen bir token yoludur, ancak OpenClaw kullanılabilir olduğunda
    Claude CLI'ın yeniden kullanılmasını ve `claude -p` tercih eder. Üretim veya çok kullanıcılı
    iş yükleri için Anthropic API anahtarı daha güvenli ve daha öngörülebilir seçenek olmaya devam eder. Diğer
    abonelik tarzı barındırılan seçenekler: [OpenAI](/tr/providers/openai), [Qwen Cloud](/tr/providers/qwen),
    [MiniMax](/tr/providers/minimax), [Z.AI (GLM)](/tr/providers/zai).

  </Accordion>

</AccordionGroup>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>

<AccordionGroup>
  <Accordion title="Anthropic'ten neden HTTP 429 rate_limit_error görüyorum?">
    Geçerli dönem için **Anthropic kotanız/hız sınırınız** tükenmiştir. **Claude
    CLI** kullanıyorsanız dönemin sıfırlanmasını bekleyin veya planınızı yükseltin. **Anthropic API anahtarı** kullanıyorsanız
    Anthropic Console'da kullanımı/faturalandırmayı kontrol edin ve gerektiğinde sınırları yükseltin.

    İleti özellikle `Extra usage is required for long context requests` ise
    istek Anthropic'in 1M bağlam penceresini (genel kullanıma sunulabilen 1M Claude 4.x
    modeli veya eski `params.context1m: true` yapılandırması) kullanmaya çalışıyordur ve mevcut kimlik bilginiz
    uzun bağlam faturalandırmasına uygun değildir.

    Bir sağlayıcının hız sınırına ulaşıldığında OpenClaw'ın yanıt vermeye devam etmesi için bir **yedek model** ayarlayın.
    Bkz. [Modeller](/tr/cli/models), [OAuth](/tr/concepts/oauth) ve
    [Uzun bağlam için Anthropic 429 ek kullanım gereksinimi](/tr/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="AWS Bedrock destekleniyor mu?">
    Evet. OpenClaw, paketle birlikte gelen bir **Amazon Bedrock (Converse)** sağlayıcısına sahiptir. AWS ortam
    işaretçileri mevcutsa (`AWS_ACCESS_KEY_ID`, `AWS_PROFILE`, `AWS_BEARER_TOKEN_BEDROCK`),
    OpenClaw model keşfi için örtük Bedrock sağlayıcısını otomatik olarak etkinleştirir; aksi takdirde
    `plugins.entries.amazon-bedrock.config.discovery.enabled: true` ayarlayın veya elle bir
    sağlayıcı girdisi ekleyin. Bkz. [Amazon Bedrock](/tr/providers/bedrock) ve [Model sağlayıcıları](/tr/providers/models).
    Yönetilen bir anahtar akışını tercih ediyorsanız Bedrock'ın önündeki OpenAI uyumlu bir proxy de geçerli bir seçenektir.
  </Accordion>

  <Accordion title="Codex kimlik doğrulaması nasıl çalışır?">
    OpenClaw, OAuth (ChatGPT oturum açma) üzerinden **OpenAI Codex** desteği sunar. Birincil
    modeli olmayan yeni bir kurulum, ChatGPT/Codex abonelik kimlik doğrulaması ve yerel Codex
    app-server yürütmesi için tam olarak `openai/gpt-5.6-sol` kullanır.
    Yeniden kimlik doğrulama, `openai/gpt-5.5` dâhil olmak üzere açıkça belirtilmiş mevcut modeli korur.
    Codex çalışma alanı GPT-5.6'yı sunmuyorsa
    açıkça `openai/gpt-5.5` seçin; OpenClaw sessizce daha düşük bir sürüme geçmez. Eski
    Codex ön ekli model başvuruları, `openclaw doctor
    --fix` tarafından onarılan eski yapılandırmadır. Doğrudan OpenAI API anahtarıyla erişim, ajan dışı OpenAI
    API yüzeyleri için ve sıralı bir `openai` API anahtarı profili aracılığıyla ajan
    modelleri için de kullanılabilir. Bkz. [Model sağlayıcıları](/tr/concepts/model-providers) ve
    [İlk kurulum (CLI)](/tr/start/wizard).
  </Accordion>

  <Accordion title="OpenClaw neden hâlâ eski OpenAI Codex ön ekinden söz ediyor?">
    `openai`, hem OpenAI API anahtarları hem de
    ChatGPT/Codex OAuth için geçerli sağlayıcı ve kimlik doğrulama profili kimliğidir; OpenAI Codex bununla birleştirilmiştir. Eski yapılandırmalarda ve geçiş uyarılarında hâlâ eski bir
    `openai-codex` ön eki görebilirsiniz:

    - `openai/gpt-5.6-sol` = ajan turları için yerel Codex çalışma zamanı kullanan yeni ChatGPT/Codex abonelik kurulumu.
    - `openai/gpt-5.5` = mevcut yapılandırma veya GPT-5.6 erişimi olmayan hesaplar için açıkça desteklenen seçim.
    - Eski `openai-codex/*` model başvuruları = `openclaw doctor --fix` tarafından onarılan eski rota.
    - `openai/gpt-5.5` ve sıralı bir `openai` API anahtarı profili = bir OpenAI ajan modeli için API anahtarı kimlik doğrulaması.
    - Eski `openai-codex` kimlik doğrulama profili kimlikleri = `openclaw doctor --fix` tarafından taşınan eski kimlikler.

    Doğrudan OpenAI Platform faturalandırması mı istiyorsunuz? `OPENAI_API_KEY` ayarlayın. ChatGPT/Codex
    abonelik kimlik doğrulaması mı istiyorsunuz? `openclaw models auth login --provider openai` çalıştırın. Model
    başvurularını standart `openai/*` sağlayıcısı altında tutun. Yeni abonelik
    kurulumu tam olarak `openai/gpt-5.6-sol` kullanır; doctor, açıkça belirtilmiş bir `openai/gpt-5.5` seçimini
    yükseltmeden eski Codex ön ekli başvuruları onarır.

  </Accordion>

  <Accordion title="Codex OAuth sınırları neden ChatGPT web'den farklı olabilir?">
    Codex OAuth, aynı hesapta bile ChatGPT web sitesi/uygulaması deneyiminden farklı olabilen,
    OpenAI tarafından yönetilen ve plana bağlı kota dönemlerini kullanır.

    `openclaw models status`, o anda görülebilen sağlayıcı kullanım/kota dönemlerini gösterir ancak
    ChatGPT web haklarını doğrudan API erişimine dönüştürmez veya normalleştirmez. Doğrudan
    OpenAI Platform faturalandırma/sınır yolu için API anahtarıyla `openai/*` kullanın.

  </Accordion>

  <Accordion title="OpenAI abonelik kimlik doğrulamasını (Codex OAuth) destekliyor musunuz?">
    Evet, tamamen desteklenir. OpenAI, OpenClaw gibi harici
    araçlarda/iş akışlarında abonelik OAuth kullanımına açıkça izin verir. İlk kurulum, OAuth akışını sizin için çalıştırabilir.

    Bkz. [OAuth](/tr/concepts/oauth), [Model sağlayıcıları](/tr/concepts/model-providers) ve [İlk kurulum (CLI)](/tr/start/wizard).

  </Accordion>

  <Accordion title="Gemini CLI OAuth'ı nasıl ayarlarım?">
    Gemini CLI, `openclaw.json` içindeki bir istemci kimliği veya gizli anahtar yerine **Plugin kimlik doğrulama akışı** kullanır.

    1. `gemini` öğesinin `PATH` üzerinde olması için Gemini CLI'ı yerel olarak kurun:
       - Homebrew: `brew install gemini-cli`
       - npm: `npm install -g @google/gemini-cli`
    2. Plugin'i etkinleştirin: `openclaw plugins enable google`
    3. Oturum açın: `openclaw models auth login --provider google-gemini-cli --set-default`
    4. Oturum açtıktan sonraki varsayılan model: `google/gemini-3.1-pro-preview` (çalışma zamanı `google-gemini-cli`)
    5. Oturum açtıktan sonra istekler başarısız mı oluyor? Gateway ana makinesinde `GOOGLE_CLOUD_PROJECT` veya `GOOGLE_CLOUD_PROJECT_ID` ayarlayıp yeniden deneyin.

    OAuth token'ları Gateway ana makinesindeki kimlik doğrulama profillerinde saklanır. Ayrıntılar: [Google](/tr/providers/google), [Model sağlayıcıları](/tr/concepts/model-providers).

  </Accordion>

  <Accordion title="Yerel bir model gündelik sohbetler için uygun mu?">
    Genellikle hayır. OpenClaw geniş bağlam ve güçlü güvenlik gerektirir; küçük kartlar bağlamı
    keser ve sağlayıcı tarafındaki güvenlik filtrelerini atlar. Kullanmanız gerekiyorsa yerel olarak çalıştırabileceğiniz **en büyük**
    model derlemesini (LM Studio) çalıştırın; bkz. [Yerel modeller](/tr/gateway/local-models). Daha küçük/nicelenmiş
    modeller istem enjeksiyonu riskini artırır; bkz. [Güvenlik](/tr/gateway/security).
  </Accordion>

  <Accordion title="Barındırılan model trafiğini belirli bir bölgede nasıl tutarım?">
    Bölgeye sabitlenmiş uç noktaları seçin. OpenRouter, MiniMax, Kimi
    ve GLM için ABD'de barındırılan seçenekler sunar; verileri bölge içinde tutmak için ABD'de barındırılan çeşidi seçin. Seçtiğiniz bölgesel sağlayıcıya
    uymaya devam ederken yedeklerin kullanılabilir kalması için bunların yanında `models.mode: "merge"` ile
    Anthropic/OpenAI'ı listelemeye devam edebilirsiniz.
  </Accordion>

  <Accordion title="Bunu kurmak için Mac Mini satın almam gerekiyor mu?">
    Hayır. OpenClaw macOS veya Linux'ta çalışır (Windows'ta WSL2 üzerinden). Mac mini popüler bir
    sürekli açık ana makine seçeneğidir ancak küçük bir VPS, ev sunucusu veya Raspberry Pi sınıfı cihaz da kullanılabilir.

    Yalnızca **macOS'e özgü araçlar için** bir Mac gerekir. iMessage için Messages oturumu açık herhangi bir Mac'te
    `imsg` ile [iMessage](/tr/channels/imessage) kullanın; Gateway Linux'ta veya başka bir yerde çalışıyorsa
    `channels.imessage.cliPath` değerini, o Mac'te `imsg` çalıştıran bir SSH sarmalayıcısına ayarlayın. Diğer
    macOS'e özgü araçlar için Gateway'i bir Mac'te çalıştırın veya bir macOS Node'u eşleştirin.

    Belgeler: [iMessage](/tr/channels/imessage), [Node'lar](/tr/nodes), [Mac uzak modu](/tr/platforms/mac/remote).

  </Accordion>

  <Accordion title="iMessage desteği için Mac mini gerekiyor mu?">
    Messages oturumu açık **herhangi bir macOS cihazı** gerekir; bunun Mac mini olması gerekmez, herhangi bir
    Mac kullanılabilir. `imsg` ile [iMessage](/tr/channels/imessage) kullanın; Gateway bu
    Mac'te veya başka bir yerde `cliPath` SSH sarmalayıcısıyla çalışabilir.

    Yaygın kurulumlar:

    - Gateway Linux/VPS üzerinde; `channels.imessage.cliPath`, Messages oturumu açık bir Mac'te `imsg` çalıştıran SSH sarmalayıcısına ayarlanır.
    - En basit tek makineli kurulum için her şey tek bir Mac'te çalışır.

    Belgeler: [iMessage](/tr/channels/imessage), [Node'lar](/tr/nodes), [Mac uzak modu](/tr/platforms/mac/remote).

  </Accordion>

  <Accordion title="OpenClaw'ı çalıştırmak için Mac mini satın alırsam onu MacBook Pro'ma bağlayabilir miyim?">
    Evet. **Mac mini Gateway'i çalıştırabilir**, MacBook Pro'nuz ise bir **Node**
    (yardımcı cihaz) olarak bağlanır. Node'lar Gateway'i çalıştırmaz; bu cihazda
    ekran/kamera/canvas ve `system.run` gibi yetenekler ekler.

    Yaygın düzen: Gateway sürekli açık Mac mini'de çalışır; MacBook Pro macOS uygulamasını veya bir
    Node ana makinesini çalıştırır ve Gateway ile eşleşir. `openclaw nodes status` / `openclaw nodes list` ile kontrol edin.

    Belgeler: [Node'lar](/tr/nodes), [Node CLI](/tr/cli/nodes).

  </Accordion>

  <Accordion title="Bun kullanabilir miyim?">
    Bağımlılıkları kurmak veya paket betiklerini çalıştırmak için Bun kullanabilirsiniz. Standart durum deposu
    `node:sqlite` kullandığı ve Bun bu API'yi sağlamadığı için OpenClaw CLI ve
    Gateway, **Node** gerektirir.
  </Accordion>

  <Accordion title="Telegram: allowFrom içine ne yazılır?">
    `channels.telegram.allowFrom`, bot kullanıcı adı değil, **insan göndericinin sayısal Telegram kullanıcı kimliğidir**.
    Kurulum yalnızca sayısal kullanıcı kimliklerini ister; `openclaw doctor --fix`,
    eski `@username` girdilerini çözümlemeyi deneyebilir.

    Daha güvenli (üçüncü taraf bot yok): Botunuza DM gönderin, `openclaw logs --follow` çalıştırın, `from.id` değerini okuyun.

    Resmî Bot API: Botunuza DM gönderin, `https://api.telegram.org/bot<bot_token>/getUpdates` çağrısı yapın, `message.from.id` değerini okuyun.

    Üçüncü taraf (daha az gizli): `@userinfobot` veya `@getidsbot` hesabına DM gönderin.

    Bkz. [Telegram erişim denetimi](/tr/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="Birden fazla kişi, farklı OpenClaw örnekleriyle tek bir WhatsApp numarasını kullanabilir mi?">
    Evet, **çoklu ajan yönlendirmesi** aracılığıyla. Her göndericinin WhatsApp DM'sini (`peer: { kind: "direct", id: "+15551234567" }`) farklı bir `agentId` ile ilişkilendirerek her kişiye kendi çalışma alanını ve oturum deposunu verin. Yanıtlar yine **aynı WhatsApp hesabından** gelir; DM erişim denetimi (`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) hesap başına geneldir. Bkz. [Çoklu Ajan Yönlendirmesi](/tr/concepts/multi-agent) ve [WhatsApp](/tr/channels/whatsapp).
  </Accordion>

  <Accordion title='Bir "hızlı sohbet" ajanı ve bir "kodlama için Opus" ajanı çalıştırabilir miyim?'>
    Evet. Çoklu ajan yönlendirmesini kullanın: her ajana kendi varsayılan modelini verin, ardından gelen
    rotaları (sağlayıcı hesabı veya belirli eşler) ilgili ajanlara bağlayın. Örnek yapılandırma:
    [Çoklu Ajan Yönlendirmesi](/tr/concepts/multi-agent). Ayrıca bkz. [Modeller](/tr/concepts/models) ve
    [Yapılandırma](/tr/gateway/configuration).
  </Accordion>

  <Accordion title="Homebrew Linux'ta çalışır mı?">
    Evet, Linuxbrew üzerinden:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    OpenClaw'ı systemd üzerinden çalıştırırken hizmet PATH'inin
    `/home/linuxbrew/.linuxbrew/bin` (veya brew ön ekinizi) içerdiğinden emin olun; böylece `brew` ile kurulan araçlar
    oturum açılmayan kabuklarda çözümlenebilir. Son derlemeler ayrıca Linux
    systemd hizmetlerinde yaygın kullanıcı bin dizinlerini (örneğin `~/.local/bin`, `~/.npm-global/bin`,
    `~/.local/share/pnpm`, `~/.bun/bin`) başa ekler ve ayarlandıklarında `PNPM_HOME`, `NPM_CONFIG_PREFIX`,
    `BUN_INSTALL`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `NVM_DIR` ve `FNM_DIR` değerlerini dikkate alır.

  </Accordion>

  <Accordion title="Değiştirilebilir git kurulumu ile npm kurulumu arasındaki fark">
    - **Değiştirilebilir (git) kurulum:** tam kaynak kodu çalışma kopyasıdır, düzenlenebilir ve katkıda bulunanlar için en uygun seçenektir. Yerel olarak derleyebilir ve kodu/belgeleri yamalayabilirsiniz.
    - **npm kurulumu:** depo içermeyen genel CLI kurulumudur ve "yalnızca çalıştırmak" için en uygun seçenektir. Güncellemeler npm dist-tag'lerinden gelir.

    Belgeler: [Başlarken](/tr/start/getting-started), [Güncelleme](/tr/install/updating).

  </Accordion>

  <Accordion title="Daha sonra npm ve git kurulumları arasında geçiş yapabilir miyim?">
    Evet, mevcut bir kurulumda `openclaw update --channel ...` ile geçiş yapabilirsiniz. Bu işlem **verilerinizi
    silmez**; yalnızca OpenClaw kod kurulumu değişir. Durum (`~/.openclaw`) ve
    çalışma alanı (`~/.openclaw/workspace`) olduğu gibi kalır.

    npm'den git'e:

    ```bash
    openclaw update --channel dev
    ```

    git'ten npm'ye:

    ```bash
    openclaw update --channel stable
    ```

    Planlanan mod geçişini önce önizlemek için `--dry-run` ekleyin. Güncelleyici, Doctor
    takip işlemlerini çalıştırır, hedef kanalın plugin kaynaklarını yeniler ve
    `--no-restart` seçeneğini iletmediğiniz sürece Gateway'i yeniden başlatır.

    Kurucu da her iki modu zorunlu kılabilir:

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method npm
    ```

    Yedekleme ipuçları: [Öğelerin diskte bulunduğu yerler](/tr/help/faq#where-things-live-on-disk).

  </Accordion>

  <Accordion title="Gateway'i dizüstü bilgisayarımda mı yoksa bir VPS'de mi çalıştırmalıyım?">
    24/7 güvenilirlik mi istiyorsunuz? Bir **VPS** kullanın. En düşük kurulum zahmetini istiyor
    ve uyku/yeniden başlatma durumlarını sorun etmiyor musunuz? Yerel olarak çalıştırın.

    **Dizüstü bilgisayar (yerel Gateway)**

    - **Artıları:** sunucu maliyeti yoktur, yerel dosyalara doğrudan erişilir, canlı bir tarayıcı penceresi vardır.
    - **Eksileri:** uyku/ağ kesintileri bağlantıyı koparır, işletim sistemi güncellemeleri/yeniden başlatmaları çalışmayı kesintiye uğratır, bilgisayarın uyanık kalması gerekir.

    **VPS / bulut**

    - **Artıları:** sürekli açıktır, ağ kararlıdır, dizüstü bilgisayarın uyku sorunları yoktur, çalışır durumda tutmak daha kolaydır.
    - **Eksileri:** genellikle başsızdır (ekran görüntülerini kullanın), yalnızca uzaktan dosya erişimi vardır, güncellemeler için SSH gerekir.

    WhatsApp/Telegram/Slack/Mattermost/Discord hizmetlerinin tümü bir VPS'den sorunsuz çalışır;
    asıl ödünleşim, başsız tarayıcı ile görünür pencere arasındadır. Bkz. [Tarayıcı](/tr/tools/browser).

    Varsayılan öneri: Daha önce Gateway bağlantı kesintileri yaşadıysanız VPS kullanın; Mac'i
    etkin olarak kullanıyorsanız ve yerel dosya erişimi ya da görünür tarayıcı kullanıcı arayüzü
    otomasyonu istiyorsanız yerel kurulum idealdir.

  </Accordion>

  <Accordion title="OpenClaw'ı özel olarak ayrılmış bir makinede çalıştırmak ne kadar önemlidir?">
    Zorunlu değildir ancak güvenilirlik ve yalıtım için önerilir.

    - **Özel olarak ayrılmış ana makine (VPS/Mac mini/Raspberry Pi):** sürekli açıktır, uyku/yeniden başlatma kaynaklı kesintiler daha azdır, izinler daha düzenlidir, çalışır durumda tutmak daha kolaydır.
    - **Paylaşılan dizüstü/masaüstü bilgisayar:** test ve etkin kullanım için uygundur ancak makine uykuya geçtiğinde veya güncellendiğinde duraklamalar beklenmelidir.

    Her iki yaklaşımın avantajlarını birleştirmek için Gateway'i özel olarak ayrılmış bir ana makinede tutun ve
    yerel ekran/kamera/çalıştırma araçları için dizüstü bilgisayarınızı bir **Node** olarak eşleştirin.
    Bkz. [Node'lar](/tr/nodes) ve [Güvenlik](/tr/gateway/security).

  </Accordion>

  <Accordion title="Asgari VPS gereksinimleri ve önerilen işletim sistemi nedir?">
    - **Mutlak asgari:** 1 vCPU, 1 GB RAM, ~500 MB disk.
    - **Önerilen:** Ek kapasite (günlükler, medya, birden fazla kanal) için 1-2 vCPU, 2 GB+ RAM. Node araçları ve tarayıcı otomasyonu yoğun kaynak kullanabilir.

    İşletim sistemi: **Ubuntu LTS** (veya herhangi bir modern Debian/Ubuntu); Linux için en kapsamlı şekilde test edilmiş kurulum yoludur.

    Belgeler: [Linux](/tr/platforms/linux), [VPS barındırma](/tr/vps).

  </Accordion>

  <Accordion title="OpenClaw'ı bir sanal makinede çalıştırabilir miyim ve gereksinimleri nelerdir?">
    Evet. Bir sanal makineyi VPS gibi değerlendirin: sürekli açık ve erişilebilir olmalı, ayrıca
    Gateway ve etkinleştirdiğiniz tüm kanallar için yeterli RAM'e sahip olmalıdır.

    - **Mutlak asgari:** 1 vCPU, 1 GB RAM.
    - **Önerilen:** Birden fazla kanal, tarayıcı otomasyonu veya medya araçları için 2 GB+ RAM.
    - **İşletim sistemi:** Ubuntu LTS veya başka bir modern Debian/Ubuntu.

    Windows'ta masaüstü kurulumu için **Windows Hub**'ı, geniş araç uyumluluğuna sahip Linux tarzı
    bir Gateway sanal makinesi içinse WSL2'yi kullanın. Bkz. [Windows](/tr/platforms/windows), [VPS barındırma](/tr/vps).
    macOS'i bir sanal makinede çalıştırmak için bkz. [macOS sanal makinesi](/tr/install/macos-vm).

  </Accordion>
</AccordionGroup>

## İlgili

- [SSS](/tr/help/faq) - ana SSS (modeller, oturumlar, Gateway, güvenlik ve daha fazlası)
- [Kuruluma genel bakış](/tr/install)
- [Başlarken](/tr/start/getting-started)
- [Sorun giderme](/tr/help/troubleshooting)
