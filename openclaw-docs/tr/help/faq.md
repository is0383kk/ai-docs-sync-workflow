---
read_when:
    - Yaygın kurulum, yükleme, ilk yapılandırma veya çalışma zamanı destek sorularını yanıtlama
    - Daha ayrıntılı hata ayıklamadan önce kullanıcılar tarafından bildirilen sorunları önceliklendirme
summary: OpenClaw kurulumu, yapılandırması ve kullanımı hakkında sıkça sorulan sorular
title: SSS
x-i18n:
    generated_at: "2026-07-27T00:02:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7bddbf851a0e25323aa7e7cfc3882b33cc0d33a2aa223cccf00328af477ab4c4
    source_path: help/faq.md
    workflow: 16
---

Gerçek dünya kurulumları (yerel geliştirme, VPS, çoklu ajan, OAuth/API anahtarları, model yük devri) için hızlı yanıtlar ve ayrıntılı sorun giderme. Çalışma zamanı tanılamaları için [Sorun Giderme](/tr/gateway/troubleshooting) bölümüne bakın. Tam yapılandırma başvurusu için [Yapılandırma](/tr/gateway/configuration) bölümüne bakın.

## Bir şey bozulduğunda ilk 60 saniye

<Steps>
  <Step title="Hızlı durum">
    ```bash
    openclaw status
    ```
    Hızlı yerel özet: işletim sistemi + güncelleme, gateway/hizmet erişilebilirliği, ajanlar/oturumlar, sağlayıcı yapılandırması + çalışma zamanı sorunları (gateway erişilebilir olduğunda).
  </Step>
  <Step title="Yapıştırılabilir rapor (paylaşılması güvenli)">
    ```bash
    openclaw status --all
    ```
    Günlük sonuyla salt okunur tanılama (token'lar gizlenir).
  </Step>
  <Step title="Daemon + bağlantı noktası durumu">
    ```bash
    openclaw gateway status
    ```
    Gözetmen çalışma zamanı ile RPC erişilebilirliğini, yoklama hedefi URL'sini ve hizmetin muhtemelen hangi yapılandırmayı kullandığını gösterir.
  </Step>
  <Step title="Derin yoklamalar">
    ```bash
    openclaw status --deep
    ```
    Desteklendiğinde kanal yoklamaları da dahil olmak üzere canlı gateway sistem durumu yoklaması (erişilebilir bir gateway gerektirir). [Sistem Durumu](/tr/gateway/health) bölümüne bakın.
  </Step>
  <Step title="En son günlüğü takip edin">
    ```bash
    openclaw logs --follow
    ```
    RPC çalışmıyorsa şuna geri dönün:
    ```bash
    tail -f "/tmp/openclaw/openclaw-$(date +%F).log"
    # Adlandırılmış profil örneği:
    tail -f "/tmp/openclaw/openclaw-dev-$(date +%F).log"
    ```
    Dosya günlükleri hizmet günlüklerinden ayrıdır; [Günlük Kaydı](/tr/logging) ve [Sorun Giderme](/tr/gateway/troubleshooting) bölümlerine bakın.
  </Step>
  <Step title="Doctor'ı çalıştırın (onarımlar)">
    ```bash
    openclaw doctor
    ```
    Yapılandırmayı ve durumu onarır/taşır, ardından sistem durumu denetimlerini çalıştırır. [Doctor](/tr/gateway/doctor) bölümüne bakın.
  </Step>
  <Step title="Gateway anlık görüntüsü (yalnızca WS)">
    ```bash
    openclaw health --json
    openclaw health --verbose   # hatalarda hedef URL'yi + yapılandırma yolunu gösterir
    ```
    Çalışan gateway'den tam bir anlık görüntü ister. [Sistem Durumu](/tr/gateway/health) bölümüne bakın.
  </Step>
</Steps>

## Hızlı başlangıç ve ilk çalıştırma kurulumu

İlk çalıştırma soru-cevapları — kurulum, ilk yapılandırma, kimlik doğrulama yolları, abonelikler, ilk hatalar — [İlk Çalıştırma SSS](/tr/help/faq-first-run) sayfasındadır.

## OpenClaw nedir?

<AccordionGroup>
  <Accordion title="Tek paragrafta OpenClaw nedir?">
    OpenClaw, kendi cihazlarınızda çalıştırdığınız kişisel bir yapay zekâ asistanıdır. Hâlihazırda kullandığınız mesajlaşma ortamlarında (Discord, Google Chat, iMessage, Mattermost, Signal, Slack, Telegram, WebChat, WhatsApp ve QQ Bot gibi paketle birlikte gelen kanal plugin'leri) yanıt verir ve desteklenen platformlarda ses ile canlı Canvas özelliklerini de kullanabilir. **Gateway**, sürekli çalışan denetim düzlemidir; ürün ise asistandır.
  </Accordion>

  <Accordion title="Değer önerisi">
    OpenClaw, "yalnızca bir Claude sarmalayıcısı" değildir. **Kendi donanımınızda** yetenekli bir asistan çalıştıran, hâlihazırda kullandığınız sohbet uygulamalarından erişilebilen; durum bilgili oturumlar, bellek ve araçlar sunan, iş akışlarınızı barındırılan bir SaaS'a teslim etmenizi gerektirmeyen **önce yerel denetim düzlemidir**.

    - **Cihazlarınız, verileriniz**: Gateway'i istediğiniz yerde (Mac, Linux, VPS) çalıştırın; çalışma alanını ve oturum geçmişini yerel tutun.
    - **Web korumalı alanı değil, gerçek kanallar**: Discord/iMessage/Signal/Slack/Telegram/WhatsApp/vb. ile desteklenen platformlarda mobil ses ve Canvas.
    - **Modelden bağımsız**: ajan başına yönlendirme ve yük devriyle Anthropic, MiniMax, OpenAI, OpenRouter vb. kullanın.
    - **Yalnızca yerel seçeneği**: tüm verilerin cihazınızda kalabilmesi için yerel modeller çalıştırın.
    - **Çoklu ajan yönlendirmesi**: kanal, hesap veya görev başına ayrı ajanlar; her birinin kendi çalışma alanı ve varsayılanları bulunur.
    - **Açık kaynaklı ve özelleştirilebilir**: sağlayıcı bağımlılığı olmadan inceleyin, genişletin ve kendi sunucunuzda barındırın.

    Belgeler: [Gateway](/tr/gateway), [Kanallar](/tr/channels), [Çoklu ajan](/tr/concepts/multi-agent), [Bellek](/tr/concepts/memory).

  </Accordion>

  <Accordion title="Kurulumu yeni tamamladım — önce ne yapmalıyım?">
    İyi ilk projeler: bir web sitesi oluşturmak (WordPress, Shopify veya statik bir site); bir mobil uygulamanın prototipini hazırlamak (taslak, ekranlar, API planı); dosya ve klasörleri düzenlemek; Gmail'e bağlanıp özetleri veya takip işlemlerini otomatikleştirmek.

    Büyük görevleri yerine getirebilir, ancak görevler aşamalara ayrıldığında ve paralel çalışma için alt ajanlar kullanıldığında en iyi sonucu verir.

  </Accordion>

  <Accordion title="OpenClaw için günlük kullanımdaki en önemli beş kullanım senaryosu nedir?">
    - **Kişisel bilgilendirmeler**: gelen kutusu, takvim ve ilgilendiğiniz haberlerin özetleri.
    - **Araştırma ve taslak hazırlama**: e-postalar veya belgeler için hızlı araştırma, özetler ve ilk taslaklar.
    - **Hatırlatıcılar ve takip işlemleri**: cron veya heartbeat tarafından tetiklenen hatırlatmalar ve kontrol listeleri.
    - **Tarayıcı otomasyonu**: form doldurma, veri toplama ve web görevlerini yineleme.
    - **Cihazlar arası koordinasyon**: telefonunuzdan bir görev gönderin, Gateway'in görevi bir sunucuda çalıştırmasını sağlayın ve sonucu sohbette alın.

  </Accordion>

  <Accordion title="OpenClaw bir SaaS için potansiyel müşteri oluşturma, erişim, reklamlar ve bloglar konusunda yardımcı olabilir mi?">
    Evet, **araştırma, nitelendirme ve taslak hazırlama** için: siteleri tarama, kısa listeler oluşturma, potansiyel müşterileri özetleme, erişim veya reklam metni taslakları yazma.

    **Erişim veya reklam çalışmaları** için sürece bir insanı dahil edin. Spam'den kaçının, yerel yasalara ve platform politikalarına uyun ve gönderilmeden önce her şeyi inceleyin. OpenClaw taslağı hazırlasın; siz onaylayın.

    Belgeler: [Güvenlik](/tr/gateway/security).

  </Accordion>

  <Accordion title="Web geliştirme için Claude Code'a kıyasla avantajları nelerdir?">
    OpenClaw bir **kişisel asistan** ve koordinasyon katmanıdır; IDE'nin yerine geçmez. Bir repo içindeki en hızlı doğrudan kodlama döngüsü için Claude Code veya Codex kullanın. Kalıcı bellek, cihazlar arası erişim ve araç orkestrasyonu için OpenClaw kullanın.

    - Oturumlar arasında kalıcı bellek ve çalışma alanı.
    - Çok platformlu erişim (Telegram, WhatsApp, TUI, WebChat).
    - Araç orkestrasyonu (tarayıcı, dosyalar, zamanlama, hook'lar).
    - Sürekli çalışan Gateway (bir VPS'te çalıştırın, her yerden etkileşim kurun).
    - Yerel tarayıcı/ekran/kamera/yürütme için Node'lar.

    Vitrin: [https://openclaw.ai/showcase](https://openclaw.ai/showcase).

  </Accordion>
</AccordionGroup>

## Skills ve otomasyon

<AccordionGroup>
  <Accordion title="Repo'yu kirli tutmadan skill'leri nasıl özelleştirebilirim?">
    Repo kopyasını düzenlemek yerine yönetilen geçersiz kılmaları kullanın. Değişiklikleri `~/.openclaw/skills/<name>/SKILL.md` içine yerleştirin (veya `~/.openclaw/openclaw.json` içindeki `skills.load.extraDirs` aracılığıyla bir klasör ekleyin). Öncelik: `<workspace>/skills` -> `<workspace>/.agents/skills` -> `~/.agents/skills` -> `~/.openclaw/skills` -> paketle birlikte gelenler -> `skills.load.extraDirs`; böylece yönetilen geçersiz kılmalar git'e dokunmadan paketle birlikte gelen skill'lere göre öncelik kazanır. Genel olarak kurmak ancak görünürlüğü bazı ajanlarla sınırlamak için paylaşılan kopyayı `~/.openclaw/skills` içinde tutun ve görünürlüğü `agents.defaults.skills` / `agents.entries.*.skills` ile denetleyin. Yalnızca üst projeye katkı sağlamaya değer düzenlemeler repo kopyasına yönelik PR'lar olarak gönderilmelidir.
  </Accordion>

  <Accordion title="Skill'leri özel bir klasörden yükleyebilir miyim?">
    Evet: `~/.openclaw/openclaw.json` içindeki `skills.load.extraDirs` aracılığıyla dizinler ekleyin (yukarıdaki sıralamada en düşük öncelik). `clawhub` varsayılan olarak `./skills` içine kurulur; OpenClaw bunu bir sonraki oturumda `<workspace>/skills` olarak değerlendirir. Görünürlüğü belirli ajanlarla sınırlamak için `agents.defaults.skills` veya `agents.entries.*.skills` ile eşleştirin.
  </Accordion>

  <Accordion title="Farklı görevler için farklı modelleri veya ayarları nasıl kullanabilirim?">
    Desteklenen kalıplar:

    - **Cron işleri**: yalıtılmış işler, iş başına bir `model` geçersiz kılması ayarlayabilir.
    - **Ajanlar**: görevleri farklı varsayılan modellere, düşünme düzeylerine ve akış parametrelerine sahip ayrı ajanlara yönlendirin.
    - **İsteğe bağlı geçiş**: `/model` mevcut oturum modelini herhangi bir zamanda değiştirir.

    Örnek — aynı model, ajan başına farklı ayarlar:

    ```json5
    {
      agents: {
        list: [
          {
            id: "coder",
            model: "xiaomi/mimo-v2.5-pro",
            thinkingDefault: "high",
            params: { temperature: 0.1 },
          },
          {
            id: "chat",
            model: "xiaomi/mimo-v2.5-pro",
            thinkingDefault: "off",
            params: { temperature: 0.8 },
          },
        ],
      },
    }
    ```

    Paylaşılan model başına varsayılanları `agents.defaults.models["provider/model"].params` içine, ardından ajana özgü geçersiz kılmaları düz `agents.entries.*.params` içine yerleştirin. Aynı modeli iç içe `agents.entries.*.models["provider/model"].params` altında yinelemeyin; bu yol ajan başına model kataloğu ve çalışma zamanı geçersiz kılmaları içindir.

    [Cron işleri](/tr/automation/cron-jobs), [Çoklu Ajan Yönlendirmesi](/tr/concepts/multi-agent), [Yapılandırma](/tr/gateway/config-agents), [Eğik çizgi komutları](/tr/tools/slash-commands) bölümlerine bakın.

  </Accordion>

  <Accordion title="Bot ağır iş yaparken donuyor. Bu işi nasıl devredebilirim?">
    Uzun veya paralel görevler için **alt ajanları** kullanın: kendi oturumlarında çalışır, bir özet döndürür ve ana sohbetinizin yanıt vermeye devam etmesini sağlarlar. Bottan "bu görev için bir alt ajan başlatmasını" isteyin veya `/subagents` kullanın. Gateway'in o anda meşgul olup olmadığını görmek için `/status` kullanın.

    Uzun görevler ve alt ajanların her ikisi de token tüketir; maliyet önemliyse `agents.defaults.subagents.model` aracılığıyla alt ajanlar için daha ucuz bir model ayarlayın.

    Belgeler: [Alt ajanlar](/tr/tools/subagents), [Arka Plan Görevleri](/tr/automation/tasks).

  </Accordion>

  <Accordion title="Discord'da ileti dizisine bağlı alt ajan oturumları nasıl çalışır?">
    Bir Discord ileti dizisini bir alt ajana veya oturum hedefine bağlayın; böylece buradaki takip mesajları bağlı oturumda kalır.

    - `thread: true` kullanarak `sessions_spawn` ile başlatın (kalıcı takip için isteğe bağlı olarak `mode: "session"`).
    - Veya `/focus <target>` ile manuel olarak bağlayın.
    - `/agents` bağlama durumunu inceler.
    - `/session idle <duration|off>` ve `/session max-age <duration|off>` otomatik odaktan çıkarma davranışını denetler.
    - `/unfocus` ileti dizisinin bağlantısını kaldırır.

    Yapılandırma: `session.threadBindings.enabled` (genel anahtar), `session.threadBindings.idleHours` (varsayılan `24`, `0` devre dışı bırakır), `session.threadBindings.maxAgeHours` (varsayılan `0` = kesin üst sınır yok) ve başlatma sırasında otomatik bağlama için `session.threadBindings.spawnSessions` (varsayılan `true`).

    Belgeler: [Alt ajanlar](/tr/tools/subagents), [Discord](/tr/channels/discord), [Yapılandırma Başvurusu](/tr/gateway/configuration-reference), [Eğik çizgi komutları](/tr/tools/slash-commands).

  </Accordion>

  <Accordion title="Bir alt ajan tamamlandı ancak tamamlanma güncellemesi yanlış yere gitti veya hiç gönderilmedi. Neleri denetlemeliyim?">
    Çözümlenen istekte bulunan rotasını denetleyin:

    - Tamamlanma modundaki alt ajan teslimatı, mevcutsa bağlı bir ileti dizisini veya konuşma rotasını tercih eder.
    - Tamamlanma kaynağı yalnızca bir kanal taşıyorsa OpenClaw, doğrudan teslimatın yine de başarılı olabilmesi için istekte bulunan oturumunun depolanmış rotasına (`lastChannel` / `lastTo` / `lastAccountId`) geri döner.
    - Bağlı rota ve kullanılabilir depolanmış rota yoksa doğrudan teslimat başarısız olabilir ve sonuç hemen gönderilmek yerine kuyruklu oturum teslimatına geri döner.
    - Geçersiz veya eski hedefler de kuyruğa geri dönüşü ya da nihai teslimat hatasını zorunlu kılabilir.
    - Alt öğenin son görünür asistan yanıtı tam olarak `NO_REPLY` / `no_reply` veya `ANNOUNCE_SKIP` ise OpenClaw, eski önceki ilerlemeyi göndermek yerine duyuruyu kasıtlı olarak engeller.

    Hata ayıklama: `<lookup>` bir görev kimliği, çalıştırma kimliği veya oturum anahtarı olmak üzere `openclaw tasks show <lookup>`.

    Belgeler: [Alt ajanlar](/tr/tools/subagents), [Arka Plan Görevleri](/tr/automation/tasks), [Oturum Araçları](/tr/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron veya hatırlatıcılar tetiklenmiyor. Neleri denetlemeliyim?">
    Cron, Gateway işlemi içinde çalışır; Gateway sürekli çalışmıyorsa tetiklenmez.

    - Cron'un etkin olduğunu (`cron.enabled`) ve `OPENCLAW_SKIP_CRON` ayarının yapılmadığını doğrulayın.
    - Gateway'in 24/7 çalıştığını doğrulayın (uyku/yeniden başlatma yok).
    - İşin saat dilimini doğrulayın (`--tz` ile ana makinenin saat dilimi).

    Hata ayıklama:
    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    Belgeler: [Cron işleri](/tr/automation/cron-jobs), [Otomasyon](/tr/automation).

  </Accordion>

  <Accordion title="Cron tetiklendi, ancak kanala hiçbir şey gönderilmedi. Neden?">
    Teslim modunu kontrol edin:

    - `--no-deliver` / `delivery.mode: "none"`: çalıştırıcının yedek gönderim yapması beklenmez.
    - Duyuru hedefi eksik veya geçersiz (`channel` / `to`): çalıştırıcı giden teslimatı atladı.
    - Kanal kimlik doğrulama hataları (`unauthorized`, `Forbidden`): çalıştırıcı teslim etmeyi denedi, ancak kimlik bilgileri bunu engelledi.
    - Sessiz ve yalıtılmış bir sonuç (yalnızca `NO_REPLY` / `no_reply`) kasıtlı olarak teslim edilemez kabul edilir; bu nedenle sıraya alınmış yedek teslimat da engellenir.

    Yalıtılmış Cron işlerinde, bir sohbet rotası mevcut olduğunda agent yine de `message` aracını kullanarak doğrudan gönderim yapabilir. `--announce`, yalnızca agent'ın kendisinin zaten göndermediği son metin için çalıştırıcının yedek teslimatını denetler.

    Hata ayıklama:
    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <lookup>
    ```

    Belgeler: [Cron işleri](/tr/automation/cron-jobs), [Arka Plan Görevleri](/tr/automation/tasks).

  </Accordion>

  <Accordion title="Yalıtılmış bir Cron çalıştırması neden model değiştirdi veya bir kez yeniden denedi?">
    Bu, yinelenen zamanlama değil, canlı model değiştirme yoludur. Yalıtılmış Cron, çalışma zamanı model devrini kalıcı hâle getirir ve etkin çalıştırma `LiveSessionModelSwitchError` hatasını oluşturduğunda yeniden denemeden önce değiştirilen sağlayıcıyı/modeli (ve değiştirilmiş herhangi bir kimlik doğrulama profili geçersiz kılmasını) koruyarak yeniden dener.

    Model seçimi önceliği: önce Gmail kancası model geçersiz kılması (`hooks.gmail.model`), ardından iş başına `model`, sonra saklanan herhangi bir Cron oturumu model geçersiz kılması ve son olarak normal agent/varsayılan model seçimi.

    Yeniden deneme döngüsü ilk deneme ve 2 değiştirme yeniden denemesiyle sınırlıdır; ardından Cron sonsuza kadar döngüye girmek yerine işlemi iptal eder.

    Hata ayıklama:
    ```bash
    openclaw cron runs --id <jobId> --limit 50
    ```

    Belgeler: [Cron işleri](/tr/automation/cron-jobs), [Cron CLI](/tr/cli/cron).

  </Accordion>

  <Accordion title="Linux'ta Skills nasıl kurulur?">
    Yerel `openclaw skills` komutlarını kullanın veya Skills öğelerini çalışma alanınıza yerleştirin; macOS Skills kullanıcı arayüzü Linux'ta kullanılamaz. Skills öğelerine [https://clawhub.ai](https://clawhub.ai) adresinden göz atın.

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install @owner/<skill-slug>
    openclaw skills install @owner/<skill-slug> --version <version>
    openclaw skills install @owner/<skill-slug> --force
    openclaw skills install @owner/<skill-slug> --global
    openclaw skills update --all
    openclaw skills update --all --global
    openclaw skills list --eligible
    openclaw skills check
    ```

    Yerel `openclaw skills install`, varsayılan olarak etkin çalışma alanının `skills/` dizinine yazar. Tüm yerel agent'lar için paylaşılan yönetilen Skills dizinine kurmak üzere `--global` ekleyin. Ayrı `clawhub` CLI'ını yalnızca kendi Skills öğelerinizi yayımlamak veya eşitlemek için kurun. Hangi agent'ların paylaşılan Skills öğelerini görebileceğini daraltmak için `agents.defaults.skills` veya `agents.entries.*.skills` kullanın.

  </Accordion>

  <Accordion title="OpenClaw görevleri zamanlanmış olarak veya arka planda sürekli çalıştırabilir mi?">
    Evet, Gateway zamanlayıcısı aracılığıyla:

    - Zamanlanmış veya yinelenen görevler için **Cron işleri** (yeniden başlatmalarda kalıcıdır).
    - Ana oturumdaki periyodik kontroller için **Heartbeat**.
    - Özet yayımlayan veya sohbetlere teslimat yapan özerk agent'lar için **yalıtılmış işler**.

    Belgeler: [Cron işleri](/tr/automation/cron-jobs), [Otomasyon](/tr/automation), [Heartbeat](/tr/gateway/heartbeat).

  </Accordion>

  <Accordion title="Yalnızca Apple macOS'ta çalışan Skills öğelerini Linux'tan çalıştırabilir miyim?">
    Doğrudan çalıştıramazsınız. macOS Skills öğeleri, `metadata.openclaw.os` ve gerekli ikili dosyalarla sınırlandırılır ve yalnızca **Gateway ana makinesinde** uygun olduklarında yüklenir. Linux'ta yalnızca `darwin` için olan Skills öğeleri (`apple-notes`, `apple-reminders`, `things-mac`), sınırlandırmayı geçersiz kılmadığınız sürece yüklenmez.

    Desteklenen üç yöntem:

    **Seçenek A - Gateway'i bir Mac'te çalıştırın (en basit)**. Gateway'i macOS ikili dosyalarının bulunduğu yerde çalıştırın, ardından Linux'tan [uzak modda](#gateway-ports-already-running-and-remote-mode) veya Tailscale üzerinden bağlanın. Gateway ana makinesi macOS olduğundan Skills öğeleri normal şekilde yüklenir.

    **Seçenek B - bir macOS Node'u kullanın (SSH olmadan)**. Gateway'i Linux'ta çalıştırın, bir macOS Node'unu (menü çubuğu uygulaması) eşleştirin ve Mac'te **Node Run Commands** ayarını "Always Ask" veya "Always Allow" olarak belirleyin. OpenClaw, gerekli ikili dosyalar Node'da bulunduğunda yalnızca macOS'ta çalışan Skills öğelerini uygun kabul eder; agent bunları `nodes` aracıyla çalıştırır. "Always Ask" kullanıldığında istemde "Always Allow" seçeneğini onaylamak, ilgili komutu izin listesine ekler.

    **Seçenek C - macOS ikili dosyalarını SSH üzerinden proxy'leyin (ileri düzey)**. Gateway'i Linux'ta tutun, ancak gerekli CLI ikili dosyalarının bir Mac'te çalışan SSH sarmalayıcılarına çözümlenmesini sağlayın; ardından uygun kalması için Skill'i Linux'a izin verecek şekilde geçersiz kılın.

    1. İkili dosya için bir SSH sarmalayıcısı oluşturun (örnek: Apple Notes için `memo`):
       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```
    2. Sarmalayıcıyı Linux ana makinesindeki `PATH` konumuna yerleştirin (örneğin `~/bin/memo`).
    3. Linux'a izin vermek için Skill meta verilerini (çalışma alanında veya `~/.openclaw/skills` içinde) geçersiz kılın:
       ```markdown
       ---
       name: apple-notes
       description: Apple Notes'u macOS'taki memo CLI aracılığıyla yönetin.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```
    4. Skills anlık görüntüsünün yenilenmesi için yeni bir oturum başlatın.

  </Accordion>

  <Accordion title="Notion veya HeyGen entegrasyonunuz var mı?">
    Şu anda yerleşik olarak sunulmuyor. Seçenekler:

    - **Özel Skill / Plugin**: güvenilir API erişimi için en iyi seçenektir (ikisinin de API'leri vardır).
    - **Tarayıcı otomasyonu**: kod olmadan çalışır, ancak daha yavaş ve daha kırılgandır.

    Ajans tarzı müşteri başına bağlam için her müşteri adına bir Notion sayfası tutun (bağlam + tercihler + etkin çalışma) ve agent'dan oturumun başında bu sayfayı getirmesini isteyin.

    Yerel bir entegrasyon için özellik isteği açın veya bu API'lere yönelik bir Skill oluşturun.

    ```bash
    openclaw skills install @owner/<skill-slug>
    openclaw skills update --all
    ```

    Yerel kurulumlar etkin çalışma alanının `skills/` dizinine yapılır; tüm yerel agent'lar için `--global` kullanın veya görünürlüğü sınırlamak üzere `agents.defaults.skills` / `agents.entries.*.skills` yapılandırın. Bazı Skills öğeleri Homebrew ile kurulmuş ikili dosyalar bekler; Linux'ta bu, Linuxbrew anlamına gelir.

    Bkz. [Skills](/tr/tools/skills), [Skills yapılandırması](/tr/tools/skills-config), [ClawHub](/tools/clawhub).

  </Accordion>

  <Accordion title="Mevcut, oturum açılmış Chrome'umu OpenClaw ile nasıl kullanırım?">
    Chrome DevTools MCP üzerinden bağlanan yerleşik `user` tarayıcı profilini kullanın:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    Özel bir ad için açık bir MCP profili oluşturun:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    Bu, yerel ana makine tarayıcısını veya bağlı bir tarayıcı Node'unu kullanabilir. Gateway başka bir yerde çalışıyorsa tarayıcı makinesinde bir Node ana makinesi çalıştırın veya bunun yerine uzak CDP kullanın.

    Yönetilen `openclaw` profiline kıyasla `existing-session` / `user` profillerindeki mevcut sınırlamalar:

    - `click`, `type`, `hover`, `scrollIntoView`, `drag` ve `select`, CSS seçicileri değil anlık görüntü referanslarını gerektirir.
    - Yükleme kancaları, her seferinde bir dosya olmak üzere `ref` veya `inputRef` gerektirir; CSS `element` desteklenmez.
    - `responsebody`, PDF dışa aktarma, indirme yakalama ve toplu eylemler hâlâ yönetilen tarayıcı yolunu gerektirir.

    Tam karşılaştırma için [Tarayıcı](/tr/tools/browser#existing-session-via-chrome-devtools-mcp) bölümüne bakın.

  </Accordion>
</AccordionGroup>

## Korumalı alan ve bellek

<AccordionGroup>
  <Accordion title="Korumalı alan için özel bir belge var mı?">
    Evet: [Korumalı alan](/tr/gateway/sandboxing). Docker'a özgü kurulum için (Docker'da tam Gateway veya korumalı alan görüntüleri) [Docker](/tr/install/docker) bölümüne bakın.
  </Accordion>

  <Accordion title="Docker sınırlı geliyor; tüm özellikleri nasıl etkinleştirebilirim?">
    Varsayılan görüntü güvenlik önceliklidir ve `node` kullanıcısı olarak çalışır; bu nedenle sistem paketlerini, Homebrew'i ve paketle birlikte gelen tarayıcıları içermez. Daha kapsamlı bir kurulum için:

    - Önbelleklerin kalıcı olması için `/home/node` öğesini `OPENCLAW_HOME_VOLUME` ile kalıcılaştırın.
    - Sistem bağımlılıklarını `OPENCLAW_IMAGE_APT_PACKAGES` ile görüntüye dahil edin.
    - Playwright tarayıcılarını paketle birlikte gelen CLI aracılığıyla kurun: `node /app/node_modules/playwright-core/cli.js install chromium`.
    - `PLAYWRIGHT_BROWSERS_PATH` ayarını yapın ve bu yolu kalıcılaştırın.

    Belgeler: [Docker](/tr/install/docker), [Tarayıcı](/tr/tools/browser).

  </Accordion>

  <Accordion title="Tek bir agent ile doğrudan mesajları kişisel tutup grupları herkese açık/korumalı alanlı yapabilir miyim?">
    Evet; özel trafik **doğrudan mesajlar**, herkese açık trafik ise **gruplar** ise mümkündür. Ana doğrudan mesaj oturumu ana makinede kalırken grup/kanal oturumlarının (ana olmayan anahtarlar) yapılandırılan korumalı alan arka ucunda çalışması için `agents.defaults.sandbox.mode: "non-main"` ayarını yapın. Korumalı alan etkinleştirildiğinde varsayılan arka uç Docker'dır. Korumalı alanlı oturumlarda kullanılabilen araçları `tools.sandbox.tools` aracılığıyla sınırlayın.

    Kurulum kılavuzu: [Gruplar: kişisel doğrudan mesajlar + herkese açık gruplar](/tr/channels/groups#pattern-personal-dms-public-groups-single-agent). Temel başvuru: [Gateway yapılandırması](/tr/gateway/config-agents#agentsdefaultssandbox).

  </Accordion>

  <Accordion title="Bir ana makine klasörünü korumalı alana nasıl bağlarım?">
    `agents.defaults.sandbox.docker.binds` ayarını `["host:container:mode"]` olarak belirleyin (örneğin `"/home/user/src:/src:ro"`). Genel ve agent başına bağlamalar birleştirilir; `scope: "shared"` olduğunda agent başına bağlamalar yok sayılır. Hassas öğeler için `:ro` kullanın; bağlamalar korumalı alan dosya sistemi duvarlarını aşar.

    OpenClaw, bağlama kaynaklarını hem normalleştirilmiş yola hem de mevcut en derin üst öğe üzerinden çözümlenen kurallı yola göre doğrular; böylece son yol bölümü henüz mevcut olmasa bile sembolik bağlantı üst öğesi üzerinden kaçışlar güvenli biçimde başarısız olur.

    Bkz. [Korumalı alan](/tr/gateway/sandboxing#custom-bind-mounts) ve [Korumalı Alan ile Araç İlkesi ile Yükseltilmiş Yetki Karşılaştırması](/tr/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check).

  </Accordion>

  <Accordion title="Bellek nasıl çalışır?">
    OpenClaw belleği, agent çalışma alanındaki Markdown dosyalarından oluşur: `memory/YYYY-MM-DD.md` içinde günlük notlar, `MEMORY.md` içinde düzenlenmiş uzun vadeli notlar (yalnızca ana/özel oturumlar).

    OpenClaw ayrıca Compaction konuşmayı özetlemeden önce sessiz bir **Compaction öncesi bellek boşaltma** işlemi çalıştırarak modele önce kalıcı notları yazmasını hatırlatır. Yalnızca çalışma alanı yazılabilir olduğunda çalışır (salt okunur korumalı alanlar bunu atlar); `agents.defaults.compaction.memoryFlush.enabled: false` ile devre dışı bırakın. Bkz. [Bellek](/tr/concepts/memory).

  </Accordion>

  <Accordion title="Bellek sürekli bir şeyleri unutuyor. Kalıcı olmasını nasıl sağlarım?">
    Bottan **bilgiyi belleğe yazmasını** isteyin: uzun vadeli notlar `MEMORY.md` içine, kısa vadeli bağlam ise `memory/YYYY-MM-DD.md` içine yazılır. Modele anıları saklamasını hatırlatmak genellikle sorunu çözer. Unutmaya devam ederse Gateway'in her çalıştırmada aynı çalışma alanını kullandığını doğrulayın.

    Belgeler: [Bellek](/tr/concepts/memory), [Agent çalışma alanı](/tr/concepts/agent-workspace).

  </Accordion>

  <Accordion title="Bellek sonsuza kadar kalıcı mı? Sınırlar nelerdir?">
    Bellek dosyaları diskte bulunur ve silinene kadar kalıcıdır; sınır model değil, depolama alanınızdır. **Oturum bağlamı** ise modelin bağlam penceresiyle sınırlıdır; bu nedenle uzun konuşmalar sıkıştırılabilir veya kesilebilir. Bellek aramasının var olma nedeni budur: yalnızca ilgili kısımları yeniden bağlama getirir.

    Belgeler: [Bellek](/tr/concepts/memory), [Bağlam](/tr/concepts/context).

  </Accordion>

  <Accordion title="Anlamsal bellek araması için OpenAI API anahtarı gerekli mi?">
    Yalnızca varsayılan sağlayıcı olan **OpenAI gömmelerini** kullanıyorsanız gereklidir. Codex OAuth, sohbet/tamamlama işlemlerini kapsar ancak gömmelere erişim **sağlamaz**; dolayısıyla Codex ile oturum açmak (OAuth veya Codex CLI oturum açma işlemiyle) anlamsal bellek aramasını etkinleştirmez. OpenAI gömmeleri için yine de gerçek bir API anahtarı (`OPENAI_API_KEY` veya `models.providers.openai.apiKey`) gerekir.

    Yerel kalmak için `memory.search.provider: "local"` (GGUF/llama.cpp) ayarını kullanın. Desteklenen diğer sağlayıcılar: Bedrock, DeepInfra, Gemini (`GEMINI_API_KEY` veya `memory.search.remote.apiKey`), GitHub Copilot, LM Studio, Mistral, Ollama, OpenAI uyumlu sağlayıcılar ve Voyage. Kurulum ayrıntıları için [Bellek](/tr/concepts/memory) ve [Bellek araması](/tr/concepts/memory-search) bölümlerine bakın.

  </Accordion>
</AccordionGroup>

## Öğelerin diskte bulunduğu yerler

<AccordionGroup>
  <Accordion title="OpenClaw ile kullanılan tüm veriler yerel olarak kaydedilir mi?">
    Hayır: **OpenClaw'ın kendi durumu yereldir**, ancak **harici hizmetler onlara gönderdiklerinizi görmeye devam eder**.

    - **Varsayılan olarak yerel**: oturumlar, bellek dosyaları, yapılandırma ve çalışma alanı Gateway ana makinesinde bulunur (`~/.openclaw` ve çalışma alanı dizininiz).
    - **Zorunlu olarak uzak**: model sağlayıcılarına (Anthropic/OpenAI/vb.) gönderilen mesajlar onların API'lerine gider ve sohbet platformları (Slack/Telegram/WhatsApp/vb.) mesaj verilerini kendi sunucularında depolar.
    - **Kapsamı siz kontrol edersiniz**: yerel modeller istemleri makinenizde tutar, ancak kanal trafiği yine de kanalın sunucularından geçer.

    İlgili bölümler: [Ajan çalışma alanı](/tr/concepts/agent-workspace), [Bellek](/tr/concepts/memory).

  </Accordion>

  <Accordion title="OpenClaw verilerini nerede depolar?">
    Her şey `$OPENCLAW_STATE_DIR` altında bulunur (varsayılan: `~/.openclaw`):

    | Yol                                                               | Amaç                                                            |
    | ------------------------------------------------------------------ | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                                 | Ana yapılandırma (JSON5)                                                 |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                        | Eski OAuth içe aktarımı (ilk kullanımda kimlik doğrulama profillerine kopyalanır)        |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json`     | Kimlik doğrulama profilleri (OAuth, API anahtarları, isteğe bağlı `keyRef`/`tokenRef`)        |
    | `$OPENCLAW_STATE_DIR/secrets.json`                                  | `file` SecretRef sağlayıcıları için isteğe bağlı, dosya tabanlı gizli veri yükü   |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`              | Eski uyumluluk dosyası (statik `api_key` girdileri temizlenmiştir)        |
    | `$OPENCLAW_STATE_DIR/credentials/`                                  | Sağlayıcı durumu (örneğin `whatsapp/<accountId>/creds.json`)      |
    | `$OPENCLAW_STATE_DIR/agents/`                                       | Ajan başına durum (agentDir + eski/arşivlenmiş oturum yapıtları)        |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/openclaw-agent.sqlite`  | Oturum satırları ve dökümler dâhil, ajan başına SQLite durumu      |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                    | Eski oturum geçişi kaynakları ve arşiv/destek yapıtları      |

    Eski tek ajanlı `~/.openclaw/agent/*` yolu, `openclaw doctor` tarafından taşınır.

    **Çalışma alanınız** (AGENTS.md, bellek dosyaları, skills vb.) ayrıdır ve `agents.defaults.workspace` aracılığıyla yapılandırılır (varsayılan: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="AGENTS.md / SOUL.md / USER.md / MEMORY.md nerede bulunmalı?">
    Bunlar `~/.openclaw` içinde değil, **ajan çalışma alanında** bulunur.

    - **Çalışma alanı (ajan başına)**: `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`, `MEMORY.md`, `memory/YYYY-MM-DD.md`, isteğe bağlı `HEARTBEAT.md`. Küçük harfli kök `memory.md`, yalnızca eski onarım girdisidir; her ikisi de mevcut olduğunda `openclaw doctor --fix`, bunu `MEMORY.md` ile birleştirebilir.
    - **Durum dizini (`~/.openclaw`)**: yapılandırma, kanal/sağlayıcı durumu, kimlik doğrulama profilleri, oturumlar, günlükler, paylaşılan skills (`~/.openclaw/skills`).

    Varsayılan çalışma alanı `~/.openclaw/workspace` olup yapılandırılabilir:

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    Bot yeniden başlatıldıktan sonra "unutuyorsa", Gateway'in her başlatmada aynı çalışma alanını kullandığını doğrulayın (uzak mod, yerel dizüstü bilgisayarınızın değil, **gateway ana makinesinin** çalışma alanını kullanır).

    İpucu: kalıcı davranış veya tercih için sohbet geçmişine güvenmek yerine bottan **bunu AGENTS.md veya MEMORY.md dosyasına yazmasını** isteyin.

    [Ajan çalışma alanı](/tr/concepts/agent-workspace) ve [Bellek](/tr/concepts/memory) bölümlerine bakın.

  </Accordion>

  <Accordion title="SOUL.md dosyasını büyütebilir miyim?">
    Evet. `SOUL.md`, ajan bağlamına eklenen çalışma alanı önyükleme dosyalarından biridir. Dosya başına varsayılan ekleme sınırı `20000` karakterdir; dosyalar arasındaki toplam önyükleme bütçesi `60000` karakterdir.

    Paylaşılan varsayılanları değiştirin:

    ```json5
    {
      agents: {
        defaults: {
          bootstrapMaxChars: 50000,
          bootstrapTotalMaxChars: 300000,
        },
      },
    }
    ```

    Alternatif olarak `agents.entries.*.bootstrapMaxChars` / `bootstrapTotalMaxChars` altındaki tek bir ajan için geçersiz kılın.

    Ham ve eklenen boyutları ve kesme gerçekleşip gerçekleşmediğini kontrol etmek için `/context` kullanın. `SOUL.md` dosyasını ses, duruş ve kişiliğe odaklı tutun; çalışma kurallarını `AGENTS.md` içine, kalıcı bilgileri ise belleğe koyun.

    [Bağlam](/tr/concepts/context) ve [Ajan yapılandırması](/tr/gateway/config-agents) bölümlerine bakın.

  </Accordion>

  <Accordion title="Önerilen yedekleme stratejisi">
    **Ajan çalışma alanınızı** **özel** bir git deposuna koyun ve özel bir yerde (örneğin özel GitHub deposunda) yedekleyin. Bu işlem belleği ve AGENTS/SOUL/USER dosyalarını kapsar ve yardımcının "zihnini" daha sonra geri yüklemenizi sağlar.

    `~/.openclaw` altındaki hiçbir şeyi (kimlik bilgileri, oturumlar, token'lar, şifrelenmiş gizli veri yükleri) kaydetmeyin. Tam geri yükleme için çalışma alanını ve durum dizinini ayrı ayrı yedekleyin.

    Belgeler: [Ajan çalışma alanı](/tr/concepts/agent-workspace).

  </Accordion>

  <Accordion title="OpenClaw'ı tamamen nasıl kaldırabilirim?">
    [Kaldırma](/tr/install/uninstall) bölümüne bakın.
  </Accordion>

  <Accordion title="Ajanlar çalışma alanının dışında çalışabilir mi?">
    Evet. Çalışma alanı katı bir sandbox değil, **varsayılan cwd** ve bellek bağlantı noktasıdır. Göreli yollar çalışma alanı içinde çözümlenir; sandbox etkinleştirilmediği sürece mutlak yollar ana makinedeki diğer konumlara erişebilir. Yalıtım için [`agents.defaults.sandbox`](/tr/gateway/sandboxing) veya ajan başına sandbox ayarlarını kullanın. Bir depoyu varsayılan çalışma dizini yapmak için söz konusu ajanın `workspace` ayarını depo köküne yönlendirin. OpenClaw deposunun kendisi yalnızca kaynak koddur; bu nedenle ajanın kasıtlı olarak depo içinde çalışmasını istemediğiniz sürece çalışma alanını ayrı tutun.

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Uzak mod: oturum deposu nerede?">
    Oturum durumu **gateway ana makinesine** aittir. Uzak modda önem verdiğiniz oturum deposu yerel dizüstü bilgisayarınızda değil, uzak makinededir. [Oturum yönetimi](/tr/concepts/session) bölümüne bakın.
  </Accordion>
</AccordionGroup>

## Yapılandırma temelleri

<AccordionGroup>
  <Accordion title="Yapılandırmanın biçimi nedir? Nerede bulunur?">
    OpenClaw, isteğe bağlı **JSON5** yapılandırmasını `$OPENCLAW_CONFIG_PATH` konumundan okur (varsayılan: `~/.openclaw/openclaw.json`). Dosya yoksa `~/.openclaw/workspace` varsayılan çalışma alanı dâhil, nispeten güvenli varsayılanları kullanır.
  </Accordion>

  <Accordion title='gateway.bind: "lan" (veya "tailnet") ayarını yaptım; artık hiçbir şey dinlemiyor / kullanıcı arayüzü yetkisiz olduğumu söylüyor'>
    Geri döngü dışı bağlamalar **geçerli bir gateway kimlik doğrulama yolu gerektirir**: paylaşılan gizli anahtar kimlik doğrulaması (token veya parola) ya da doğru yapılandırılmış kimlik farkındalığına sahip bir ters proxy arkasında `gateway.auth.mode: "trusted-proxy"`.

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    - `gateway.remote.token` / `.password`, yerel gateway kimlik doğrulamasını kendi başlarına **etkinleştirmez**; yerel çağrı yolları yalnızca `gateway.auth.*` ayarlanmamışsa `gateway.remote.*` değerini yedek olarak kullanabilir.
    - Parola kimlik doğrulaması için `gateway.auth.mode: "password"` ile birlikte `gateway.auth.password` (veya `OPENCLAW_GATEWAY_PASSWORD`) ayarlayın.
    - `gateway.auth.token` / `.password`, SecretRef aracılığıyla açıkça yapılandırılmış ancak çözümlenmemişse çözümleme güvenli biçimde başarısız olur (uzak yedek mekanizması hatayı maskelemez).
    - Paylaşılan gizli anahtarlı Control UI kurulumları, `connect.params.auth.token` veya `connect.params.auth.password` (uygulama/kullanıcı arayüzü ayarlarında saklanır) aracılığıyla kimlik doğrular. Tailscale Serve veya `trusted-proxy` gibi kimlik taşıyan modlar bunun yerine istek üstbilgilerini kullanır; paylaşılan gizli anahtarları URL'lere koymaktan kaçının.
    - `gateway.auth.mode: "trusted-proxy"` kullanıldığında, aynı ana makinedeki geri döngü ters proxy'leri açık bir `gateway.auth.trustedProxy.allowLoopback = true` ayarı ve `gateway.trustedProxies` içinde bir geri döngü girdisi gerektirir.

  </Accordion>

  <Accordion title="Artık localhost üzerinde neden token gerekiyor?">
    OpenClaw, geri döngü dâhil olmak üzere gateway kimlik doğrulamasını varsayılan olarak zorunlu kılar. Açık bir kimlik doğrulama yolu yapılandırılmamışsa başlangıç, token modunu seçer ve yalnızca o başlatma süresince geçerli bir token üretir; dolayısıyla yerel WS istemcilerinin kimlik doğrulaması gerekir. Bu, diğer yerel işlemlerin Gateway'i çağırmasını engeller.

    İstemcilerin yeniden başlatmalar arasında kararlı bir gizli anahtara ihtiyacı olduğunda `gateway.auth.token`, `gateway.auth.password`, `OPENCLAW_GATEWAY_TOKEN` veya `OPENCLAW_GATEWAY_PASSWORD` ayarını açıkça yapılandırın. Ayrıca parola modunu veya kimlik farkındalığına sahip ters proxy'ler için `trusted-proxy` seçeneğini belirleyebilirsiniz. Açık geri döngü için `gateway.auth.mode: "none"` ayarını açıkça belirleyin. `openclaw doctor --generate-gateway-token` her zaman bir token üretir.

  </Accordion>

  <Accordion title="Yapılandırmayı değiştirdikten sonra yeniden başlatmam gerekir mi?">
    Gateway yapılandırmayı izler ve çalışırken yeniden yüklemeyi destekler: `gateway.reload.mode: "hybrid"` (varsayılan), güvenli değişiklikleri çalışırken uygular ve kritik değişiklikler için yeniden başlatır. `hot`, `restart` ve `off` de desteklenir. Çoğu `tools.*`, `agents.*` ilkesi, `session.*` ve `messages.*` değişikliği herhangi bir yeniden yükleme işlemi olmadan hemen uygulanır; `gateway.*` bağlama/port değişiklikleri yeniden başlatma gerektirir.
  </Accordion>

  <Accordion title="Web aramasını (ve web içeriği getirmeyi) nasıl etkinleştiririm?">
    `web_fetch` API anahtarı olmadan çalışır. `web_search` seçtiğiniz sağlayıcıya bağlıdır:

    | Sağlayıcı | Anahtarsız | Ortam değişkenleri |
    | --- | --- | --- |
    | Brave | Hayır | `BRAVE_API_KEY` |
    | DuckDuckGo | Evet (resmî olmayan, HTML tabanlı) | - |
    | Exa | Hayır | `EXA_API_KEY` |
    | Firecrawl | Hayır | `FIRECRAWL_API_KEY` |
    | Gemini | Hayır | `GEMINI_API_KEY` |
    | Grok | Hayır (xAI OAuth veya anahtar) | `XAI_API_KEY` |
    | Kimi | Hayır | `KIMI_API_KEY` veya `MOONSHOT_API_KEY` |
    | MiniMax Search | Hayır | `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY` veya `MINIMAX_API_KEY` |
    | Ollama Web Search | Evet (`ollama signin` gerekir) | - |
    | Perplexity | Hayır | `PERPLEXITY_API_KEY` veya `OPENROUTER_API_KEY` |
    | SearXNG | Evet (kendi sunucunuzda barındırılır) | `SEARXNG_BASE_URL` |
    | Tavily | Hayır | `TAVILY_API_KEY` |

    Grok ayrıca model kimlik doğrulamasındaki xAI OAuth'u (`openclaw onboard --auth-choice xai-oauth`) yeniden kullanabilir.

    **Önerilen**: `openclaw configure --section web` ve bir sağlayıcı seçin.

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
      },
      tools: {
        web: {
          search: {
            enabled: true,
            provider: "brave",
            maxResults: 5,
          },
          fetch: {
            enabled: true,
            provider: "firecrawl", // isteğe bağlı; otomatik algılama için çıkarın
          },
        },
      },
    }
    ```

    Sağlayıcıya özgü web araması yapılandırması `plugins.entries.<plugin>.config.webSearch.*` altında bulunur. Eski `tools.web.search.*` sağlayıcı yolları uyumluluk için hâlâ yüklenir ancak yeni yapılandırmalarda kullanılmamalıdır. Firecrawl web getirme yedek yapılandırması `plugins.entries.firecrawl.config.webFetch.*` altında bulunur.

    - İzin listeleri: `web_search`/`web_fetch`/`x_search` veya üçü birden için `group:web` ekleyin.
    - `web_fetch` varsayılan olarak etkindir.
    - `tools.web.fetch.provider` belirtilmezse OpenClaw, kullanılabilir kimlik bilgilerinden hazır durumdaki ilk getirme yedek sağlayıcısını otomatik olarak algılar; resmi Firecrawl Plugin'i bu yedeği sağlar.
    - Arka plan hizmetleri, ortam değişkenlerini `~/.openclaw/.env` dosyasından (veya hizmet ortamından) okur.

    Belgeler: [Web araçları](/tr/tools/web).

  </Accordion>

  <Accordion title="config.apply yapılandırmamı sildi. Nasıl kurtarabilir ve bunu önleyebilirim?">
    `config.apply`, **yapılandırmanın tamamını** değiştirir; kısmi bir nesne diğer her şeyi kaldırır.

    Güncel OpenClaw, kazara gerçekleşen üzerine yazmaların çoğunu önler:

    - OpenClaw'a ait yapılandırma yazma işlemleri, yazmadan önce değişiklik sonrası yapılandırmanın tamamını doğrular.
    - Geçersiz veya yıkıcı OpenClaw yazma işlemleri reddedilir ve `openclaw.json.rejected.*` olarak kaydedilir.
    - Başlatmayı veya çalışırken yeniden yüklemeyi bozan doğrudan bir düzenleme, Gateway'in güvenli biçimde başarısız olmasına veya yeniden yüklemeyi atlamasına neden olur; `openclaw.json` dosyasını yeniden yazmaz.
    - Onarımın sahibi `openclaw doctor --fix`'dir; bilinen son iyi durumu geri yükleyebilir ve reddedilen dosyayı `openclaw.json.clobbered.*` olarak kaydeder.

    Kurtarma:

    - `openclaw logs --follow` içinde `Invalid config at`, `Config write rejected:` veya `config reload skipped (invalid config)` olup olmadığını kontrol edin.
    - Etkin yapılandırmanın yanındaki en yeni `openclaw.json.clobbered.*` veya `openclaw.json.rejected.*` dosyasını inceleyin.
    - `openclaw config validate` ve `openclaw doctor --fix` komutlarını çalıştırın.
    - Yalnızca amaçlanan anahtarları `openclaw config set` veya `config.patch` ile geri kopyalayın.
    - Bilinen son iyi durum veya reddedilmiş veri yoksa yedekten geri yükleyin ya da `openclaw doctor` komutunu yeniden çalıştırıp kanalları/modelleri yeniden yapılandırın.
    - Beklenmeyen kayıp durumunda, bilinen son yapılandırmanız veya bir yedeğinizle hata bildirimi oluşturun. Yerel bir kodlama aracısı, günlüklerden veya geçmişten çoğu zaman çalışan bir yapılandırmayı yeniden oluşturabilir.

    Önlemek için: küçük değişikliklerde `openclaw config set`, etkileşimli düzenlemelerde `openclaw configure`, bilinmeyen bir yolu incelemek için `config.schema.lookup` (yüzeysel bir şema düğümü ve doğrudan alt öğe özetlerini döndürür), kısmi RPC düzenlemelerinde ise `config.patch` kullanın; `config.apply` seçeneğini yapılandırmanın tamamını değiştirmek için ayırın. Aracıya yönelik `gateway` çalışma zamanı aracı, eski `tools.bash.*` takma adları üzerinden bile `tools.exec.ask` / `tools.exec.security` yollarını yeniden yazmayı reddeder.

    Belgeler: [Yapılandırma](/tr/cli/config), [Yapılandırma işlemi](/tr/cli/configure), [Gateway sorunlarını giderme](/tr/gateway/troubleshooting#gateway-rejected-invalid-config), [Doctor](/tr/gateway/doctor).

  </Accordion>

  <Accordion title="Cihazlar arasında uzmanlaşmış çalışanlarla merkezi bir Gateway'i nasıl çalıştırırım?">
    Yaygın düzen: **bir Gateway** (örneğin Raspberry Pi) ile **Node'lar** ve **aracılar**.

    - **Gateway (merkezi)**: kanalların (Signal/WhatsApp), yönlendirmenin ve oturumların sahibidir.
    - **Node'lar (cihazlar)**: Mac/iOS/Android cihazlar çevre birimi olarak bağlanır ve yerel araçları (`system.run`, `canvas`, `camera`) kullanıma sunar.
    - **Aracılar (çalışanlar)**: özel roller için (örneğin operasyonlar ve kişisel veriler) ayrı beyinler/çalışma alanlarıdır.
    - **Alt aracılar**: paralellik sağlamak için ana aracıdan arka plan işleri başlatır.
    - **TUI**: Gateway'e bağlanır ve aracılar/oturumlar arasında geçiş yapar.

    Belgeler: [Node'lar](/tr/nodes), [Uzaktan erişim](/tr/gateway/remote), [Çok Aracılı Yönlendirme](/tr/concepts/multi-agent), [Alt aracılar](/tr/tools/subagents), [TUI](/tr/web/tui).

  </Accordion>

  <Accordion title="OpenClaw tarayıcısı başsız çalışabilir mi?">
    Evet:

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    Varsayılan değer `false`'dir (arayüzlü). Başsız modun bazı sitelerde bot karşıtı denetimleri tetikleme olasılığı daha yüksektir (X/Twitter başsız oturumları sık sık engeller). Aynı Chromium motorunu kullanır ve çoğu otomasyon için çalışır; temel fark, görünür bir tarayıcı penceresinin olmamasıdır (görseller için ekran görüntülerini kullanın). Bkz. [Tarayıcı](/tr/tools/browser).

  </Accordion>

  <Accordion title="Tarayıcı kontrolü için Brave'i nasıl kullanırım?">
    `browser.executablePath` değerini Brave ikili dosyanızın (veya Chromium tabanlı herhangi bir tarayıcının) yolu olarak ayarlayın ve Gateway'i yeniden başlatın. Bkz. [Tarayıcı](/tr/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Uzak Gateway'ler ve Node'lar

<AccordionGroup>
  <Accordion title="Komutlar Telegram, Gateway ve Node'lar arasında nasıl iletilir?">
    Telegram mesajları, aracıyı çalıştıran **Gateway** tarafından işlenir ve yalnızca bir Node aracına ihtiyaç duyulduğunda **Gateway WebSocket** üzerinden Node'lar çağrılır:

    Telegram -> Gateway -> Aracı -> `node.*` -> Node -> Gateway -> Telegram

    Node'lar gelen sağlayıcı trafiğini görmez; yalnızca Node RPC çağrılarını alırlar.

  </Accordion>

  <Accordion title="Gateway uzakta barındırılıyorsa aracım bilgisayarıma nasıl erişebilir?">
    Bilgisayarınızı bir **Node** olarak eşleştirin. Gateway başka bir yerde çalışır ancak Gateway WebSocket üzerinden yerel makinenizdeki `node.*` araçlarını (ekran, kamera, sistem) çağırabilir.

    1. Gateway'i sürekli açık olan ana makinede (VPS/ev sunucusu) çalıştırın.
    2. Gateway ana makinesini ve bilgisayarınızı aynı tailnet'e yerleştirin.
    3. Gateway WS'ye erişilebildiğinden emin olun (tailnet bağlaması veya SSH tüneli).
    4. macOS uygulamasını yerel olarak açın ve Node olarak kaydolması için **Remote over SSH** modunda (veya doğrudan tailnet üzerinden) bağlanın.
    5. Node'u onaylayın:
       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Ayrı bir TCP köprüsü gerekmez; Node'lar Gateway WebSocket üzerinden bağlanır.

    Güvenlik hatırlatması: bir macOS Node'unu eşleştirmek, bu makinede `system.run` kullanımına izin verir. Yalnızca güvendiğiniz cihazları eşleştirin; [Güvenlik](/tr/gateway/security) bölümünü inceleyin.

    Belgeler: [Node'lar](/tr/nodes), [Gateway protokolü](/tr/gateway/protocol), [macOS uzak modu](/tr/platforms/mac/remote), [Güvenlik](/tr/gateway/security).

  </Accordion>

  <Accordion title="Tailscale bağlı ancak yanıt alamıyorum. Şimdi ne yapmalıyım?">
    Temel noktaları kontrol edin:

    ```bash
    openclaw gateway status
    openclaw status
    openclaw channels status
    ```

    Ardından kimlik doğrulamayı ve yönlendirmeyi doğrulayın: Tailscale Serve kullanıyorsanız `gateway.auth.allowTailscale` değerinin doğru ayarlandığını onaylayın; SSH tüneli üzerinden bağlanıyorsanız tünelin etkin olduğunu ve doğru bağlantı noktasını hedeflediğini doğrulayın; doğrudan mesaj/grup izin listelerinizin hesabınızı içerdiğini onaylayın.

    Belgeler: [Tailscale](/tr/gateway/tailscale), [Uzaktan erişim](/tr/gateway/remote), [Kanallar](/tr/channels).

  </Accordion>

  <Accordion title="İki OpenClaw örneği birbiriyle iletişim kurabilir mi (yerel + VPS)?">
    Evet, ancak yerleşik bir bottan bota köprü yoktur.

    **En basit yöntem**: iki botun da erişebildiği normal bir sohbet kanalı (Slack/Telegram/WhatsApp) kullanın. Bot A'nın Bot B'ye mesaj göndermesini sağlayın, ardından Bot B'nin her zamanki gibi yanıt vermesine izin verin.

    **CLI köprüsü (genel)**: diğer botun dinlediği bir sohbeti hedefleyerek `openclaw agent --message ... --deliver` ile diğer Gateway'i çağıran bir betik çalıştırın. Botlardan biri uzak bir VPS'deyse CLI'nızı SSH/Tailscale üzerinden bu uzak Gateway'e yönlendirin (bkz. [Uzaktan erişim](/tr/gateway/remote)):

    ```bash
    openclaw agent --message "Yerel bottan merhaba" --deliver --channel telegram --reply-to <chat-id>
    ```

    İki botun sonsuz bir döngüye girmemesi için bir koruma ekleyin (yalnızca bahsetme, kanal izin listeleri veya "bot mesajlarına yanıt verme" kuralı).

    Belgeler: [Uzaktan erişim](/tr/gateway/remote), [Aracı CLI'sı](/tr/cli/agent), [Aracı gönderimi](/tr/tools/agent-send).

  </Accordion>

  <Accordion title="Birden fazla aracı için ayrı VPS'lere ihtiyacım var mı?">
    Hayır. Tek bir Gateway, her biri kendi çalışma alanına, model varsayılanlarına ve yönlendirmesine sahip birden fazla aracıyı barındırır; bu normal kurulumdur ve aracı başına bir VPS kullanmaktan çok daha ucuz/basittir. Ayrı VPS'leri yalnızca katı yalıtım (güvenlik sınırları) veya paylaşmak istemediğiniz çok farklı yapılandırmalar için kullanın.
  </Accordion>

  <Accordion title="VPS'den SSH kullanmak yerine kişisel dizüstü bilgisayarımda Node kullanmanın bir avantajı var mı?">
    Evet: Node'lar, uzak bir Gateway'den dizüstü bilgisayarınıza ulaşmanın birinci sınıf yoludur ve kabuk erişiminden daha fazlasını kullanıma açar. Gateway macOS/Linux üzerinde (Windows'ta WSL2 üzerinden) çalışır ve hafiftir (küçük bir VPS veya Raspberry Pi sınıfı cihaz yeterlidir; 4 GB RAM fazlasıyla yeterli olur), bu nedenle yaygın kurulum sürekli açık bir ana makine ile Node olarak kullanılan dizüstü bilgisayarınızdır.

    - **Gelen SSH gerekmez** - Node'lar cihaz eşleştirme yoluyla Gateway WebSocket'e dışarı doğru bağlanır.
    - **Daha güvenli yürütme denetimleri** - `system.run`, söz konusu dizüstü bilgisayardaki Node izin listeleri/onaylarıyla sınırlandırılır.
    - **Daha fazla cihaz aracı** - Node'lar `system.run`'ye ek olarak `canvas`, `camera` ve `screen` araçlarını kullanıma sunar.
    - **Yerel tarayıcı otomasyonu** - Gateway'i bir VPS'de tutup Chrome'u bir Node ana makinesi üzerinden yerel olarak çalıştırın veya Chrome MCP üzerinden yerel Chrome'a bağlanın.

    SSH, geçici kabuk erişimi için uygundur; devam eden aracı iş akışları ve cihaz otomasyonu için Node'lar daha basittir.

    Belgeler: [Node'lar](/tr/nodes), [Node CLI'sı](/tr/cli/nodes), [Tarayıcı](/tr/tools/browser).

  </Accordion>

  <Accordion title="Node'lar bir Gateway hizmeti çalıştırır mı?">
    Hayır. Bilerek yalıtılmış profiller çalıştırmadığınız sürece ana makine başına yalnızca **bir Gateway** çalışmalıdır (bkz. [Birden fazla Gateway](/tr/gateway/multiple-gateways)). Node'lar Gateway'e bağlanan çevre birimleridir (iOS/Android Node'ları veya menü çubuğu uygulamasındaki macOS "node mode"). Başsız Node ana makineleri ve CLI denetimi için bkz. [Node ana makinesi CLI'sı](/tr/cli/node).

    `gateway`, `discovery` ve barındırılan Plugin yüzeyi değişiklikleri için tam yeniden başlatma gerekir.

  </Accordion>

  <Accordion title="Yapılandırmayı uygulamanın bir API / RPC yolu var mı?">
    Evet:

    - `config.schema.lookup`: yazmadan önce yüzeysel şema düğümü, eşleşen kullanıcı arayüzü ipucu ve doğrudan alt öğe özetleriyle tek bir yapılandırma alt ağacını inceleyin.
    - `config.get`: mevcut anlık görüntüyü ve karmayı alın.
    - `config.patch`: güvenli kısmi güncelleme (çoğu RPC düzenlemesi için tercih edilir); mümkün olduğunda çalışırken yeniden yükler, gerektiğinde yeniden başlatır.
    - `config.apply`: yapılandırmanın tamamını doğrular ve değiştirir; mümkün olduğunda çalışırken yeniden yükler, gerektiğinde yeniden başlatır.
    - Aracıya yönelik `gateway` çalışma zamanı aracı, `tools.exec.ask` / `tools.exec.security` yollarını yeniden yazmayı hâlâ reddeder; eski `tools.bash.*` takma adları aynı korumalı yollara normalleştirilir.

  </Accordion>

  <Accordion title="İlk kurulum için makul en küçük yapılandırma">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    Çalışma alanınızı ayarlar ve botu kimlerin tetikleyebileceğini sınırlar.

  </Accordion>

  <Accordion title="Bir VPS'te Tailscale'i nasıl kurar ve Mac'imden nasıl bağlanırım?">
    1. **VPS'te kurun ve oturum açın**:
       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```
    2. Tailscale uygulamasını kullanarak **Mac'inizde kurun ve oturum açın**; aynı tailnet'i kullanın.
    3. VPS'in kararlı bir ada sahip olması için Tailscale yönetici konsolunda **MagicDNS'i etkinleştirin**.
    4. **Tailnet ana bilgisayar adını kullanın**: SSH `ssh user@your-vps.tailnet-xxxx.ts.net`; Gateway WS `ws://your-vps.tailnet-xxxx.ts.net:18789`.

    SSH olmadan Denetim Arayüzü'nü kullanmak için VPS'te Tailscale Serve'ü kullanın:

    ```bash
    openclaw gateway --tailscale serve
    ```

    Bu, Gateway'i geri döngüye bağlı tutar ve HTTPS'i Tailscale üzerinden kullanıma açar. Bkz. [Tailscale](/tr/gateway/tailscale).

  </Accordion>

  <Accordion title="Bir Mac node'unu uzak bir Gateway'e nasıl bağlarım (Tailscale Serve)?">
    Serve, **Gateway Denetim Arayüzü + WS**'yi kullanıma açar; node'lar aynı Gateway WS uç noktası üzerinden bağlanır.

    1. VPS ile Mac'in aynı tailnet'te olduğundan emin olun.
    2. macOS uygulamasını Uzak modda kullanın (SSH hedefi tailnet ana bilgisayar adı olabilir) — uygulama Gateway portuna tünel açar ve node olarak bağlanır.
    3. Node'u onaylayın:
       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Belgeler: [Gateway protokolü](/tr/gateway/protocol), [Keşif](/tr/gateway/discovery), [macOS uzak modu](/tr/platforms/mac/remote).

  </Accordion>

  <Accordion title="İkinci bir dizüstü bilgisayara kurulum yapmalı mıyım, yoksa yalnızca bir node mu eklemeliyim?">
    İkinci dizüstü bilgisayarda **yalnızca yerel araçları** (ekran/kamera/exec) kullanmak için cihazı **node** olarak ekleyin; tek Gateway kullanılır ve yapılandırma çoğaltılmaz. Yerel node araçları şu anda yalnızca macOS'te kullanılabilir. İkinci bir Gateway'i yalnızca **katı yalıtım** veya tamamen ayrı iki bot için kurun.

    Belgeler: [Node'lar](/tr/nodes), [Node CLI'si](/tr/cli/nodes), [Birden çok Gateway](/tr/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## Ortam değişkenleri ve .env yükleme

<AccordionGroup>
  <Accordion title="OpenClaw ortam değişkenlerini nasıl yükler?">
    OpenClaw, ortam değişkenlerini üst süreçten (kabuk, launchd/systemd, CI vb.) okur ve ayrıca şunları yükler:

    - `.env`, geçerli çalışma dizininden.
    - `~/.openclaw/.env` (`$OPENCLAW_STATE_DIR/.env`) konumundaki genel yedek `.env`.

    İki `.env` dosyası da mevcut ortam değişkenlerini geçersiz kılmaz. Sağlayıcı kimlik bilgileri ve uç nokta yönlendirme anahtarları, çalışma alanı `.env` için istisnadır: `GEMINI_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY` gibi anahtarlar veya `_ENDPOINT` ile biten herhangi bir anahtar (ve diğer paketlenmiş sağlayıcı kimlik doğrulama ya da uç nokta ortam değişkenleri), çalışma alanı `.env` içinden yok sayılır ve süreç ortamında, `~/.openclaw/.env` içinde veya `env` yapılandırmasında bulunmalıdır.

    Yapılandırmadaki satır içi ortam değişkenleri yalnızca süreç ortamında eksik olmaları durumunda uygulanır:

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    Tam öncelik sırası ve kaynaklar için bkz. [/environment](/tr/help/environment).

  </Accordion>

  <Accordion title="Gateway'i hizmet üzerinden başlattım ve ortam değişkenlerim kayboldu. Şimdi ne yapmalıyım?">
    İki çözüm vardır:

    1. Eksik anahtarları `~/.openclaw/.env` içine koyun; böylece hizmet, kabuk ortamınızı devralmasa bile yüklenirler.
    2. Kabuktan içe aktarmayı etkinleştirin (isteğe bağlı kolaylık):
       ```json5
       {
         env: {
           shellEnv: {
             enabled: true,
             timeoutMs: 15000,
           },
         },
       }
       ```
       Bu, oturum açma kabuğunuzu çalıştırır ve yalnızca eksik olan beklenen anahtarları içe aktarır (mevcut olanları hiçbir zaman geçersiz kılmaz). Ortam değişkeni eşdeğerleri: `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='COPILOT_GITHUB_TOKEN ayarladım, ancak model durumu "Shell env: off." gösteriyor. Neden?'>
    `openclaw models status`, **kabuk ortamını içe aktarmanın** etkin olup olmadığını bildirir. "Shell env: off", ortam değişkenlerinizin eksik olduğu anlamına **gelmez**; yalnızca OpenClaw'ın oturum açma kabuğunuzu otomatik olarak yüklemeyeceği anlamına gelir.

    Gateway bir hizmet olarak (launchd/systemd) çalışıyorsa kabuk ortamınızı devralmaz. Token'ı `~/.openclaw/.env` içine koyarak, `env.shellEnv.enabled: true` özelliğini etkinleştirerek veya `env` yapılandırmasına ekleyerek (yalnızca eksikse uygulanır) sorunu giderin; ardından Gateway'i yeniden başlatıp tekrar kontrol edin:

    ```bash
    openclaw models status
    ```

    Copilot token'ları şu sırayla çözümlenir: `OPENCLAW_GITHUB_TOKEN`, ardından `COPILOT_GITHUB_TOKEN`, sonra `GH_TOKEN` ve son olarak `GITHUB_TOKEN`.

    Bkz. [/concepts/model-providers](/tr/concepts/model-providers) ve [/environment](/tr/help/environment).

  </Accordion>
</AccordionGroup>

## Oturumlar ve birden çok sohbet

<AccordionGroup>
  <Accordion title="Yeni bir konuşmayı nasıl başlatırım?">
    `/new` veya `/reset` değerini tek başına bir mesaj olarak gönderin. Bkz. [Oturum yönetimi](/tr/concepts/session).
  </Accordion>

  <Accordion title="Hiç /new göndermezsem oturumlar otomatik olarak sıfırlanır mı?">
    Hayır, varsayılan olarak sıfırlanmaz. Oturumlar aynı `sessionId` değerini korur ve konuşmalar büyüdükçe Compaction, etkin model bağlamını sınırlar. `/new` ve `/reset` kullanılabilir durumda kalır; ayrıca `mode: "daily"` veya `mode: "idle"` ile otomatik sıfırlamaları etkinleştirebilirsiniz. Günlük mod, Gateway ana bilgisayarında `session.reset.atHour` saatinde (varsayılan `4`, 0-23) yeni güne geçer; boşta modu ise Heartbeat/Cron/exec sistem olaylarını değil, son gerçek etkileşimden bu yana geçen `session.reset.idleMinutes` süresini kullanır.

    ```json5
    {
      session: {
        reset: { mode: "daily", atHour: 4 },
        resetByType: {
          group: { mode: "idle", idleMinutes: 120 },
          thread: { mode: "daily", atHour: 6 },
        },
        resetByChannel: {
          discord: { mode: "idle", idleMinutes: 10080 },
        },
      },
    }
    ```

    `resetByType`; `direct`, `group` ve `thread` değerlerini destekler. Doctor, eski `dm` girdilerini `direct` biçimine taşır; şema `dm` değerini reddeder. Eski üst düzey `session.idleMinutes`, hiçbir `session.reset`/`resetByType` bloğu ayarlanmamışsa boşta modu varsayılanı için uyumluluk takma adı olarak çalışmaya devam eder. Yaşam döngüsünün tamamı için bkz. [Oturum yönetimi](/tr/concepts/session).

  </Accordion>

  <Accordion title="OpenClaw örneklerinden oluşan bir ekip (bir CEO ve çok sayıda agent) oluşturmanın bir yolu var mı?">
    Evet, **çoklu agent yönlendirmesi** ve **alt agent'lar** aracılığıyla: bir koordinatör agent ile kendi çalışma alanlarına ve modellerine sahip birkaç çalışan agent.

    Bunu eğlenceli bir deney olarak görmek en iyisidir; çok fazla token tüketir ve çoğu zaman ayrı oturumlara sahip tek bir bottan daha az verimlidir. Tipik model, konuştuğunuz tek bir botun paralel işler için farklı oturumlar kullanması ve gerektiğinde alt agent'lar oluşturmasıdır.

    Belgeler: [Çoklu agent yönlendirmesi](/tr/concepts/multi-agent), [Alt agent'lar](/tr/tools/subagents), [Agent CLI'si](/tr/cli/agents).

  </Accordion>

  <Accordion title="Bağlam görevin ortasında neden kesildi? Bunu nasıl önleyebilirim?">
    Oturum bağlamı, model penceresiyle sınırlıdır. Uzun sohbetler, büyük araç çıktıları veya çok sayıda dosya Compaction ya da kesilmeyi tetikleyebilir.

    - Bottan mevcut durumu özetlemesini ve bir dosyaya yazmasını isteyin.
    - Uzun görevlerden önce `/compact`, konu değiştirirken `/new` kullanın.
    - Önemli bağlamı çalışma alanında tutun ve bottan yeniden okumasını isteyin.
    - Ana sohbetin daha küçük kalması için uzun veya paralel işlerde alt agent'ları kullanın.
    - Bu durum sık yaşanıyorsa daha büyük bağlam penceresine sahip bir model seçin.

  </Accordion>

  <Accordion title="OpenClaw'ı kurulu tutarak tamamen nasıl sıfırlarım?">
    ```bash
    openclaw reset
    ```

    Etkileşimsiz tam sıfırlama:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    Ardından kurulumu yeniden çalıştırın:

    ```bash
    openclaw onboard --install-daemon
    ```

    Mevcut bir yapılandırma algılarsa ilk katılım işlemi de **Sıfırla** seçeneğini sunar; bkz. [İlk katılım (CLI)](/tr/start/wizard). Profiller (`--profile` / `OPENCLAW_PROFILE`) kullandıysanız her durum dizinini sıfırlayın (varsayılan `~/.openclaw-<profile>`). Yalnızca geliştirmeye yönelik sıfırlama: `openclaw gateway --dev --reset`, geliştirme yapılandırmasını, kimlik bilgilerini, oturumları ve çalışma alanını siler.

  </Accordion>

  <Accordion title='"context too large" hataları alıyorum; nasıl sıfırlar veya Compaction uygularım?'>
    - **Compaction uygulayın** (konuşmayı korur, eski sıraları özetler): özeti yönlendirmek için `/compact` veya `/compact <instructions>`.
    - **Sıfırlayın** (aynı sohbet anahtarı için yeni oturum kimliği): `/new` veya `/reset`.

    Sorun devam ederse eski araç çıktılarını kırpmak için **oturum budamayı** (`agents.defaults.contextPruning`) ayarlayın veya daha büyük bağlam penceresine sahip bir model kullanın.

    Belgeler: [Compaction](/tr/concepts/compaction), [Oturum budama](/tr/concepts/session-pruning), [Oturum yönetimi](/tr/concepts/session).

  </Accordion>

  <Accordion title='"LLM request rejected: messages.content.tool_use.input field required" iletisini neden görüyorum?'>
    Sağlayıcı doğrulama hatası: model, gerekli `input` alanı olmadan bir `tool_use` bloğu üretti. Bu genellikle oturum geçmişinin eski veya bozuk olduğu anlamına gelir (çoğunlukla uzun ileti dizilerinden ya da bir araç/şema değişikliğinden sonra).

    Çözüm: `/new` ile yeni bir oturum başlatın (tek başına mesaj).

  </Accordion>

  <Accordion title="Neden her 30 dakikada bir Heartbeat mesajı alıyorum?">
    Heartbeat'ler varsayılan olarak her **30m** aralıkla; çözümlenen kimlik doğrulama modu Anthropic OAuth/token kimlik doğrulamasıysa (Claude CLI yeniden kullanımı dâhil) ve `heartbeat.every` ayarlanmamışsa her **1h** aralıkla çalışır. Ayarlamak veya devre dışı bırakmak için:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // or "0m" to disable
          },
        },
      },
    }
    ```

    `HEARTBEAT.md` mevcut ancak işlevsel olarak boşsa (yalnızca boş satırlar, Markdown/HTML yorumları, ATX başlıkları, çit işaretçileri veya boş liste öğesi taslakları içeriyorsa), OpenClaw API çağrılarını azaltmak için Heartbeat çalıştırmasını atlar. Dosya eksikse Heartbeat yine çalışır ve ne yapılacağına model karar verir.

    Agent başına geçersiz kılmalar `agents.entries.*.heartbeat` kullanır. Belgeler: [Heartbeat](/tr/gateway/heartbeat).

  </Accordion>

  <Accordion title='Bir WhatsApp grubuna "bot hesabı" eklemem gerekir mi?'>
    Hayır. OpenClaw **kendi hesabınızda** çalışır; gruptaysanız OpenClaw grubu görebilir. Varsayılan olarak, gönderenlere izin verene kadar grup yanıtları engellenir (`groupPolicy: "allowlist"`).

    Grup yanıtlarını yalnızca kendinizle sınırlandırmak için:

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Bir WhatsApp grubunun JID'sini nasıl bulurum?">
    En hızlı yöntem: günlükleri canlı izleyin ve gruba bir test mesajı gönderin.

    ```bash
    openclaw logs --follow --json
    ```

    `1234567890-1234567890@g.us` gibi, `@g.us` ile biten `chatId` (veya `from`) değerini arayın.

    Zaten yapılandırılmış/izin verilenler listesine eklenmişse grupları yapılandırmadan listeleyin:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    Belgeler: [WhatsApp](/tr/channels/whatsapp), [Dizin](/tr/cli/directory), [Günlükler](/tr/cli/logs).

  </Accordion>

  <Accordion title="OpenClaw bir grupta neden yanıt vermiyor?">
    İki yaygın neden vardır: bahsetme kapısı varsayılan olarak açıktır (bottan @bahsetmeniz veya `mentionPatterns` ile eşleşmeniz gerekir) ya da `channels.whatsapp.groups` değerini `"*"` olmadan yapılandırmışsınızdır ve grup izin verilenler listesinde değildir.

    Bkz. [Gruplar](/tr/channels/groups) ve [Grup mesajları](/tr/channels/group-messages).

  </Accordion>

  <Accordion title="Gruplar/ileti dizileri, doğrudan mesajlarla bağlam paylaşır mı?">
    Doğrudan sohbetler varsayılan olarak ana oturumda birleştirilir. Grupların/kanalların kendi oturum anahtarları vardır; Telegram konuları ve Discord ileti dizileri ayrı oturumlardır. Bkz. [Gruplar](/tr/channels/groups) ve [Grup mesajları](/tr/channels/group-messages).
  </Accordion>

  <Accordion title="Kaç çalışma alanı ve agent oluşturabilirim?">
    Kesin bir sınır yoktur; onlarca, hatta yüzlerce oluşturabilirsiniz, ancak şunlara dikkat edin:

    - **Disk büyümesi**: etkin oturumlar ve transkriptler ajan başına SQLite veritabanında bulunur; eski/arşiv yapıtları yine de `~/.openclaw/agents/<agentId>/sessions/` altında birikebilir.
    - **Token maliyeti**: daha fazla ajan, daha fazla eşzamanlı model kullanımı demektir.
    - **Operasyon yükü**: ajan başına kimlik doğrulama profilleri, çalışma alanları ve kanal yönlendirmesi.

    Her ajan için bir **etkin** çalışma alanı (`agents.defaults.workspace`) tutun, disk büyürse eski oturumları `openclaw sessions cleanup` ile temizleyin (etkin SQLite durumunu elle düzenlemeyin) ve sahipsiz çalışma alanlarıyla profil uyuşmazlıklarını tespit etmek için `openclaw doctor` kullanın.

  </Accordion>

  <Accordion title="Aynı anda birden fazla bot veya sohbet (Slack) çalıştırabilir miyim ve bunu nasıl kurmalıyım?">
    Evet, **Çok Ajanlı Yönlendirme** aracılığıyla: birden fazla yalıtılmış ajan çalıştırın ve gelen mesajları kanal/hesap/eş düzey varlığa göre yönlendirin. Slack bir kanal olarak desteklenir ve belirli ajanlara bağlanabilir.

    Tarayıcı erişimi güçlüdür ancak "bir insanın yapabildiği her şeyi yapamaz"; bot önleme mekanizmaları, CAPTCHA'lar ve MFA yine de otomasyonu engelleyebilir. En güvenilir denetim için ana makinedeki yerel Chrome MCP'yi veya tarayıcıyı gerçekten çalıştıran makinedeki CDP'yi kullanın.

    En iyi uygulama kurulumu: sürekli açık bir Gateway ana makinesi (VPS/Mac mini), rol başına bir ajan (bağlamalar), bu ajanlara bağlanmış Slack kanalları ve gerektiğinde Chrome MCP ya da bir Node aracılığıyla yerel tarayıcı.

    Belgeler: [Çok Ajanlı Yönlendirme](/tr/concepts/multi-agent), [Slack](/tr/channels/slack), [Tarayıcı](/tr/tools/browser), [Node'lar](/tr/nodes).

  </Accordion>
</AccordionGroup>

## Modeller, yük devretme ve kimlik doğrulama profilleri

Model SSS'si (varsayılanlar, seçim, takma adlar, geçiş, yük devretme ve kimlik doğrulama profilleri) [Modeller SSS](/tr/help/faq-models) sayfasındadır.

## Gateway: bağlantı noktaları, "zaten çalışıyor" ve uzak mod

<AccordionGroup>
  <Accordion title="Gateway hangi bağlantı noktasını kullanır?">
    `gateway.port`, WebSocket + HTTP (Control UI, hook'lar vb.) için tek çoklanmış bağlantı noktasını denetler. Öncelik sırası:

    ```text
    --port > OPENCLAW_GATEWAY_PORT > gateway.port > varsayılan 18789
    ```

  </Accordion>

  <Accordion title='openclaw gateway status neden "Runtime: running" ancak "Connectivity probe: failed" diyor?'>
    "Running", **gözetmenin** görünümüdür (launchd/systemd/schtasks); bağlantı yoklaması ise CLI'ın gerçekten Gateway WebSocket'ine bağlanmasıdır. `openclaw gateway status` çıktısındaki şu satırlara güvenin: `Probe target:` (yoklamanın kullandığı URL), `Listening:` (bağlantı noktasına gerçekte neyin bağlandığı), `Last gateway error:` (işlem çalıştığı hâlde bağlantı noktası dinlemiyorsa yaygın temel neden).
  </Accordion>

  <Accordion title='openclaw gateway status neden "Config (cli)" ile "Config (service)" değerlerini farklı gösteriyor?'>
    Hizmet başka bir yapılandırma dosyasını çalıştırırken siz bir yapılandırma dosyasını düzenliyorsunuz (genellikle `--profile` / `OPENCLAW_STATE_DIR` uyuşmazlığı).

    Düzeltmek için hizmetin kullanmasını istediğiniz `--profile` / ortamdan şunu çalıştırın:

    ```bash
    openclaw gateway install --force
    ```

  </Accordion>

  <Accordion title='"another gateway instance is already listening" ne anlama geliyor?'>
    OpenClaw, başlangıçta WebSocket dinleyicisini hemen bağlayarak bir çalışma zamanı kilidi uygular (varsayılan `ws://127.0.0.1:18789`). Bağlama `EADDRINUSE` ile başarısız olursa `GatewayLockError` ("another gateway instance is already listening") hatasını verir.

    Düzeltme: diğer örneği durdurun, bağlantı noktasını boşaltın veya `openclaw gateway --port <port>` ile çalıştırın.

  </Accordion>

  <Accordion title="OpenClaw'u uzak modda (istemci başka bir yerdeki Gateway'e bağlanır) nasıl çalıştırırım?">
    `gateway.mode: "remote"` değerini ayarlayıp uzak bir WebSocket URL'sine yönlendirin; isteğe bağlı olarak paylaşılan gizli anahtar tabanlı uzak kimlik bilgilerini kullanabilirsiniz:

    ```json5
    {
      gateway: {
        mode: "remote",
        remote: {
          url: "ws://gateway.tailnet:18789",
          token: "your-token",
          password: "your-password",
        },
      },
    }
    ```

    - `openclaw gateway` yalnızca `gateway.mode` değeri `local` olduğunda (veya bir geçersiz kılma bayrağı ilettiğinizde) başlar.
    - macOS uygulaması yapılandırma dosyasını izler ve bu değerler değiştiğinde modları canlı olarak değiştirir.
    - `gateway.remote.token` / `.password` yalnızca istemci tarafındaki uzak kimlik bilgileridir; tek başlarına yerel Gateway kimlik doğrulamasını etkinleştirmezler.

  </Accordion>

  <Accordion title='Control UI "unauthorized" diyor (veya sürekli yeniden bağlanıyor). Şimdi ne yapmalıyım?'>
    Gateway kimlik doğrulama yolunuz ile UI'ın kimlik doğrulama yöntemi eşleşmiyor.

    Gerçekler (koddan):

    - Control UI, token'ı `sessionStorage` içinde tutar ve geçerli tarayıcı sekmesiyle seçili Gateway URL'siyle sınırlar; böylece aynı sekmedeki yenilemeler, token'ın localStorage'da uzun süreli kalıcılığı olmadan çalışmaya devam eder.
    - `AUTH_TOKEN_MISMATCH` durumunda güvenilen istemciler, Gateway yeniden deneme ipuçları (`canRetryWithDeviceToken=true`, `recommendedNextStep=retry_with_device_token`) döndürdüğünde önbelleğe alınmış bir cihaz token'ıyla sınırlandırılmış tek bir yeniden deneme yapabilir.
    - Bu önbelleğe alınmış token yeniden denemesi, cihaz token'ıyla saklanan önbelleğe alınmış onaylı kapsamları yeniden kullanır; açık `deviceToken` / açık `scopes` çağıranlar, önbelleğe alınmış kapsamları devralmak yerine istedikleri kapsam kümesini korur.
    - Bu yeniden deneme yolunun dışında bağlantı kimlik doğrulama önceliği şöyledir: önce açıkça belirtilmiş paylaşılan token/parola, ardından açık `deviceToken`, sonra saklanan cihaz token'ı ve son olarak önyükleme token'ı.
    - Yerleşik kurulum kodu önyüklemesi, güvenilen mobil ilk katılım için `scopes: []` içeren bir Node cihaz token'ı ve sınırlandırılmış bir operatör devir token'ı döndürür. Operatör devri, kurulum zamanındaki yerel yapılandırmayı okuyabilir ancak eşleştirme değişikliği kapsamlarını veya `operator.admin` yetkisini vermez.

    Düzeltme:

    - En hızlı yol: `openclaw dashboard` (pano URL'sini yazdırır ve kopyalar, açmayı dener; grafik arabirimsizse bir SSH ipucu gösterir).
    - Henüz token yoksa: `openclaw doctor --generate-gateway-token`.
    - Uzak bağlantıda: önce `ssh -N -L 18789:127.0.0.1:18789 user@host` ile tünel oluşturun, ardından `http://127.0.0.1:18789/` adresini açın.
    - Paylaşılan gizli anahtar modunda: `gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` veya `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` değerlerini ayarlayın, ardından eşleşen gizli anahtarı Control UI ayarlarına yapıştırın.
    - Tailscale Serve modunda: `gateway.auth.allowTailscale` özelliğinin etkin olduğunu ve Tailscale kimlik üstbilgilerini atlayan ham bir geri döngü/tailnet URL'si yerine Serve URL'sini açtığınızı doğrulayın.
    - Güvenilen proxy modunda: yapılandırılmış kimlik farkındalığına sahip proxy üzerinden geldiğinizi doğrulayın. Aynı ana makinedeki geri döngü proxy'leri de `gateway.auth.trustedProxy.allowLoopback = true` gerektirir.
    - Tek yeniden denemeden sonra uyuşmazlık sürerse eşleştirilmiş cihaz token'ını döndürün/yeniden onaylayın:
      ```bash
      openclaw devices list
      openclaw devices rotate --device <id> --role operator
      ```
    - Döndürme reddedilirse: eşleştirilmiş cihaz oturumları, `operator.admin` yetkisine de sahip olmadıkça yalnızca **kendi** cihazlarını döndürebilir ve açık `--scope` değerleri çağıranın mevcut operatör kapsamlarını aşamaz.
    - Hâlâ çözülemediyse: `openclaw status --all` ve [Sorun Giderme](/tr/gateway/troubleshooting). Kimlik doğrulama ayrıntıları için [Pano](/tr/web/dashboard) sayfasına bakın.

  </Accordion>

  <Accordion title="gateway.bind tailnet ayarladım ancak yalnızca geri döngüde dinliyor">
    `tailnet` bağlaması, ağ arabirimlerinizden bir Tailscale IP'si seçer (100.64.0.0/10). Makine Tailscale üzerinde değilse (veya arabirim kapalıysa) Gateway, başka bir ağ arabirimini dışarı açmak yerine geri döngüye döner.

    Düzeltme: bu ana makinede Tailscale'i başlatıp Gateway'i yeniden başlatın veya açıkça `gateway.bind: "loopback"` / `"lan"` seçeneğine geçin.

    `tailnet` açıktır; `auto` geri döngüyü tercih eder. Gerekli aynı ana makine `127.0.0.1` dinleyicisini korurken geri döngü dışı erişimi Tailnet ile sınırlamak için `gateway.bind: "tailnet"` kullanın.

  </Accordion>

  <Accordion title="Aynı ana makinede birden fazla Gateway çalıştırabilir miyim?">
    Genellikle hayır; tek bir Gateway birden fazla mesajlaşma kanalını ve ajanı çalıştırabilir. Birden fazla Gateway'i yalnızca yedeklilik (örneğin bir kurtarma botu) veya katı yalıtım için kullanın ve her birini kendi `OPENCLAW_CONFIG_PATH`, `OPENCLAW_STATE_DIR`, `agents.defaults.workspace` ve benzersiz `gateway.port` değeriyle yalıtın.

    Önerilen: örnek başına `openclaw --profile <name> ...` (otomatik olarak `~/.openclaw-<name>` oluşturur), profil yapılandırması başına benzersiz bir `gateway.port` (veya elle çalıştırmalar için `--port`) ve `openclaw --profile <name> gateway install` ile profil başına bir hizmet.

    Profiller ayrıca hizmet adlarına son ek ekler: launchd `ai.openclaw.<profile>`, systemd `openclaw-gateway-<profile>.service`, Windows `OpenClaw Gateway (<profile>)`. Niteliksiz `openclaw-gateway` systemd birimi yalnızca varsayılan profil için bulunur; yeniden adlandırma öncesindeki eski systemd birim adı `clawdbot-gateway` otomatik olarak taşınır.

    Tam kılavuz: [Birden fazla Gateway](/tr/gateway/multiple-gateways).

  </Accordion>

  <Accordion title='"invalid handshake" / kod 1008 ne anlama geliyor?'>
    Gateway bir **WebSocket sunucusudur** ve ilk mesajın bir `connect` çerçevesi olmasını bekler. Bunun dışındaki her şey bağlantıyı **kod 1008** (ilke ihlali) ile kapatır.

    Yaygın nedenler: bir WS istemcisi yerine tarayıcıda **HTTP** URL'sini açmanız, yanlış bağlantı noktasını/yolu kullanmanız veya bir proxy/tünelin kimlik doğrulama üstbilgilerini kaldırması ya da Gateway dışı bir istek göndermesi.

    Düzeltme: WS URL'sini (`ws://<host>:18789` veya HTTPS üzerinden `wss://...`) kullanın, WS bağlantı noktasını normal bir tarayıcı sekmesinde açmayın ve kimlik doğrulama açıkken token'ı/parolayı `connect` çerçevesine ekleyin. CLI/TUI örneği:

    ```bash
    openclaw tui --url ws://<host>:18789 --token <token>
    ```

    Protokol ayrıntıları: [Gateway protokolü](/tr/gateway/protocol).

  </Accordion>
</AccordionGroup>

## Günlük kaydı ve hata ayıklama

<AccordionGroup>
  <Accordion title="Günlükler nerede?">
    Dosya günlükleri (yapılandırılmış): varsayılan profil için `/tmp/openclaw/openclaw-YYYY-MM-DD.log` veya adlandırılmış bir profil için `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`. `logging.file` ile kararlı bir yol; `logging.level` ile dosya günlük düzeyi; `--verbose` ve `logging.consoleLevel` ile konsol ayrıntı düzeyi ayarlayın.

    En hızlı canlı izleme:

    ```bash
    openclaw logs --follow
    ```

    Hizmet/gözetmen günlükleri (Gateway launchd/systemd aracılığıyla çalıştığında):

    - macOS launchd standart çıktısı: `~/Library/Logs/openclaw/gateway.log` (profiller `gateway-<profile>.log` kullanır; standart hata çıktısı bastırılır).
    - Linux: `journalctl --user -u openclaw-gateway[-<profile>].service -n 200 --no-pager`.
    - Windows: `schtasks /Query /TN "OpenClaw Gateway (<profile>)" /V /FO LIST`.

    Daha fazla bilgi için [Sorun Giderme](/tr/gateway/troubleshooting) sayfasına bakın.

  </Accordion>

  <Accordion title="Gateway hizmetini nasıl başlatır/durdurur/yeniden başlatırım?">
    ```bash
    openclaw gateway status
    openclaw gateway restart
    ```

    Gateway'i elle çalıştırıyorsanız `openclaw gateway --force` bağlantı noktasını geri alabilir. Bkz. [Gateway](/tr/gateway).

  </Accordion>

  <Accordion title="Windows'ta terminalimi kapattım; OpenClaw'u nasıl yeniden başlatırım?">
    Üç Windows kurulum modu:

    **1) Windows Hub yerel kurulumu**: yerel uygulama, uygulamanın sahip olduğu yerel bir WSL Gateway'i yönetir. Başlat menüsünden veya sistem tepsisinden **OpenClaw Companion** uygulamasını açın, ardından **Gateway Setup** seçeneğini veya Connections sekmesini kullanın.

    **2) Elle kurulan WSL2 Gateway**: Gateway Linux içinde çalışır.
    ```powershell
    wsl
    openclaw gateway status
    openclaw gateway restart
    ```
    Hizmeti hiç kurmadıysanız ön planda başlatın: `openclaw gateway run`.

    **3) Yerel Windows CLI/Gateway**: doğrudan Windows'ta çalışır.
    ```powershell
    openclaw gateway status
    openclaw gateway restart
    ```
    Elle çalıştırıyorsanız (hizmet yoksa): `openclaw gateway run`.

    Belgeler: [Windows](/tr/platforms/windows), [Gateway hizmeti operasyon kılavuzu](/tr/gateway).

  </Accordion>

  <Accordion title="Gateway çalışıyor ancak yanıtlar hiç ulaşmıyor. Neleri kontrol etmeliyim?">
    Hızlı sistem durumu taraması:

    ```bash
    openclaw status
    openclaw models status
    openclaw channels status
    openclaw logs --follow
    ```

    Yaygın nedenler: model kimlik doğrulamasının **Gateway ana makinesinde** yüklenmemesi (`models status` değerini kontrol edin), kanal eşleştirmesinin/izin listesinin yanıtları engellemesi (kanal yapılandırmasını ve günlükleri kontrol edin) veya WebChat/Pano'nun doğru token olmadan açık olması. Uzak bağlantı kullanıyorsanız tünel/Tailscale bağlantısının etkin olduğunu ve Gateway WebSocket'ine erişilebildiğini doğrulayın.

    Docs: [Kanallar](/tr/channels), [Sorun giderme](/tr/gateway/troubleshooting), [Uzaktan erişim](/tr/gateway/remote).

  </Accordion>

  <Accordion title='"Gateway bağlantısı kesildi: neden yok" - şimdi ne yapılmalı?'>
    Genellikle kullanıcı arayüzünün WebSocket bağlantısını kaybettiği anlamına gelir. Şunları kontrol edin: Gateway çalışıyor mu (`openclaw gateway status`)? Sağlıklı mı (`openclaw status`)? Kullanıcı arayüzünde doğru token var mı (`openclaw dashboard`)? Uzaktaysa tünel/Tailscale bağlantısı etkin mi?

    Ardından günlükleri takip edin:

    ```bash
    openclaw logs --follow
    ```

    Docs: [Pano](/tr/web/dashboard), [Uzaktan erişim](/tr/gateway/remote), [Sorun giderme](/tr/gateway/troubleshooting).

  </Accordion>

  <Accordion title="Telegram setMyCommands başarısız oluyor. Neleri kontrol etmeliyim?">
    ```bash
    openclaw channels status
    openclaw channels logs --channel telegram
    ```

    Ardından hatayı eşleştirin:

    - `BOT_COMMANDS_TOO_MUCH`: Telegram menüsünde çok fazla giriş var. OpenClaw zaten Telegram sınırına göre kırpar ve daha az komutla yeniden dener, ancak bazı menü girişleri yine de çıkarılabilir. Plugin/skill/özel komutları azaltın veya menüye ihtiyacınız yoksa `channels.telegram.commands.native` seçeneğini devre dışı bırakın.
    - `TypeError: fetch failed`, `Network request for 'setMyCommands' failed!` veya benzer ağ hataları: Bir VPS'de veya proxy arkasındaysanız giden HTTPS bağlantılarına izin verildiğini ve DNS'in `api.telegram.org` için çalıştığını doğrulayın.

    Gateway uzaktaysa Gateway ana makinesindeki günlükleri kontrol edin.

    Docs: [Telegram](/tr/channels/telegram), [Kanal sorunlarını giderme](/tr/channels/troubleshooting).

  </Accordion>

  <Accordion title="TUI çıktı göstermiyor. Neleri kontrol etmeliyim?">
    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    Geçerli durumu görmek için TUI'da `/status` kullanın. Bir sohbet kanalında yanıt bekliyorsanız iletimin etkinleştirildiğini doğrulayın (`/deliver on`).

    Docs: [TUI](/tr/web/tui), [Eğik çizgi komutları](/tr/tools/slash-commands).

  </Accordion>

  <Accordion title="Gateway'i tamamen durdurup ardından nasıl başlatırım?">
    Hizmeti yüklediyseniz (macOS'ta launchd, Linux'ta systemd):

    ```bash
    openclaw gateway stop
    openclaw gateway start
    ```

    Ön planda Ctrl-C ile durdurun, ardından `openclaw gateway run`.

    Docs: [Gateway hizmeti çalışma kılavuzu](/tr/gateway).

  </Accordion>

  <Accordion title="En basit anlatımla: openclaw gateway restart ile openclaw gateway arasındaki fark">
    `openclaw gateway restart`, **arka plan hizmetini** (launchd/systemd) yeniden başlatır. `openclaw gateway`, bu terminal oturumu için gateway'i **ön planda** çalıştırır. Hizmeti yüklediyseniz gateway alt komutlarını, tek seferlik kullanım için doğrudan ön plan çalıştırmasını kullanın.
  </Accordion>

  <Accordion title="Bir şey başarısız olduğunda daha fazla ayrıntı edinmenin en hızlı yolu">
    Daha fazla konsol ayrıntısı için Gateway'i `--verbose` ile başlatın, ardından kanal kimlik doğrulaması, model yönlendirmesi ve RPC hataları için günlük dosyasını inceleyin.
  </Accordion>
</AccordionGroup>

## Medya ve ekler

<AccordionGroup>
  <Accordion title="Skill'im bir görüntü/PDF oluşturdu ancak hiçbir şey gönderilmedi">
    Agent'tan giden ekler `media`, `mediaUrl`, `path` veya `filePath` gibi yapılandırılmış medya alanlarını kullanmalıdır. Bkz. [OpenClaw asistan kurulumu](/tr/start/openclaw) ve [Agent gönderimi](/tr/tools/agent-send).

    ```bash
    openclaw message send --target +15555550123 --message "Buyurun" --media /path/to/file.png
    ```

    Ayrıca şunları kontrol edin: Hedef kanal giden medyayı destekliyor ve izin listeleri tarafından engellenmiyor; dosya, sağlayıcının boyut sınırları içinde (görüntüler en fazla 2048px kenar uzunluğuna yeniden boyutlandırılır); `tools.fs.workspaceOnly=true`, yerel yol üzerinden gönderimleri çalışma alanı, geçici/medya deposu ve sandbox tarafından doğrulanmış dosyalarla sınırlar; `tools.fs.workspaceOnly=false` (varsayılan), yapılandırılmış yerel medya gönderimlerinin agent'ın zaten okuyabildiği ana makineye ait yerel dosyaları medya ve güvenli belge türleri (görüntüler, ses, video, PDF, Office belgeleri ve Markdown/MD, TXT, JSON, YAML/YML gibi doğrulanmış metin belgeleri) için kullanmasına izin verir. Bu bir gizli bilgi tarayıcısı değildir; uzantı ve içerik doğrulaması eşleştiğinde agent tarafından okunabilen bir `secret.txt` veya `config.json` eklenebilir. Hassas dosyaları agent tarafından okunabilen yolların dışında tutun veya daha katı yerel yol gönderimleri için `tools.fs.workspaceOnly=true` ayarını koruyun.

    Bkz. [Görüntüler](/tr/nodes/images).

  </Accordion>
</AccordionGroup>

## Güvenlik ve erişim denetimi

<AccordionGroup>
  <Accordion title="OpenClaw'u gelen DM'lere açmak güvenli mi?">
    Gelen DM'leri güvenilmeyen girdi olarak değerlendirin. Varsayılanlar riski azaltır:

    - DM destekleyen kanallardaki varsayılan davranış **eşleştirme**dir: bilinmeyen gönderenler bir eşleştirme kodu alır ve mesajları işlenmez. `openclaw pairing approve --channel <channel> [--account <id>] <code>` ile onaylayın. Bekleyen istekler **kanal başına 3** ile sınırlıdır; kod ulaşmadıysa `openclaw pairing list --channel <channel> [--account <id>]` değerini kontrol edin.
    - DM'leri herkese açmak açıkça etkinleştirme (`dmPolicy: "open"` ve izin listesi `"*"`) gerektirir.

    Riskli DM politikalarını ortaya çıkarmak için `openclaw doctor` çalıştırın.

  </Accordion>

  <Accordion title="Prompt enjeksiyonu yalnızca herkese açık botlar için mi endişe kaynağıdır?">
    Hayır. Prompt enjeksiyonu, yalnızca bota kimin DM gönderebildiğiyle değil, **güvenilmeyen içerikle** ilgilidir. Asistanınız harici içerik (web araması/getirme, tarayıcı sayfaları, e-postalar, belgeler, ekler, yapıştırılmış günlükler) okuyorsa bu içerik, tek gönderen siz olsanız bile modeli ele geçirmeye çalışan talimatlar taşıyabilir.

    En büyük risk, araçlar etkinleştirildiğinde ortaya çıkar: model, bağlamı dışarı sızdırması veya sizin adınıza araçları çağırması için kandırılabilir. Etki alanını daraltın:

    - güvenilmeyen içeriği özetlemek için salt okunur veya araçları devre dışı bırakılmış bir "okuyucu" agent kullanın
    - araçların etkin olduğu agent'lar için `web_search` / `web_fetch` / `browser` özelliklerini kapalı tutun
    - kodu çözülmüş dosya/belge metnini de güvenilmeyen olarak değerlendirin: OpenResponses `input_file` ve medya eki çıkarma işlemi, ham dosya metnini iletmek yerine çıkarılan metni açık harici içerik sınırı işaretçileriyle sarmalar
    - sandbox kullanın ve katı araç izin listeleri uygulayın

    Ayrıntılar: [Güvenlik](/tr/gateway/security).

  </Accordion>

  <Accordion title="OpenClaw, Rust/WASM yerine TypeScript/Node kullandığı için daha mı az güvenli?">
    Dil ve çalışma zamanı önemlidir ancak kişisel bir agent için ana risk değildir. Pratik riskler; gateway'in dışarıya açılması, bota kimlerin mesaj gönderebildiği, prompt enjeksiyonu, araç kapsamı, kimlik bilgilerinin işlenmesi, tarayıcı erişimi, yürütme erişimi ve üçüncü taraf skill/Plugin güvenidir.

    Rust ve WASM, bazı kod sınıfları için daha güçlü yalıtım sağlayabilir ancak prompt enjeksiyonunu, hatalı izin listelerini, herkese açık gateway erişimini, aşırı geniş kapsamlı araçları veya hassas hesaplarda zaten oturum açmış bir tarayıcı profilini çözmez. Bunları birincil denetimler olarak değerlendirin: Gateway'i özel veya kimlik doğrulamalı tutun, DM'ler/gruplar için eşleştirme ve izin listeleri kullanın, güvenilmeyen girdiler için riskli araçları reddedin veya sandbox'a alın, yalnızca güvenilir Plugin'leri ve skill'leri yükleyin ve yapılandırma değişikliklerinden sonra `openclaw security audit --deep` çalıştırın.

    Ayrıntılar: [Güvenlik](/tr/gateway/security), [Sandbox kullanımı](/tr/gateway/sandboxing).

  </Accordion>

  <Accordion title="Dışarıya açık OpenClaw örnekleri hakkında haberler gördüm. Neleri kontrol etmeliyim?">
    ```bash
    openclaw security audit --deep
    openclaw gateway status
    ```

    Daha güvenli bir temel: `loopback` adresine bağlanmış veya yalnızca kimliği doğrulanmış özel erişim (tailnet, SSH tüneli, token/parola kimlik doğrulaması ya da doğru yapılandırılmış güvenilir bir proxy) üzerinden dışarıya açılmış Gateway; `pairing` veya `allowlist` modundaki DM'ler; her üye güvenilir olmadığı sürece izin listesine alınmış ve bahsetme koşuluna bağlanmış gruplar; güvenilmeyen içerik okuyan agent'lar için reddedilmiş veya sıkı kapsamlandırılmış yüksek riskli araçlar (`exec`, `browser`, `gateway`, `cron`); araç yürütme işleminin daha dar bir etki alanına ihtiyaç duyduğu yerlerde etkinleştirilmiş sandbox kullanımı.

    Kimlik doğrulaması olmayan herkese açık bağlantılar, araçlarla birlikte açık DM'ler/gruplar ve dışarıya açık tarayıcı denetimi önce düzeltilmesi gereken bulgulardır. Ayrıntılar: [openclaw security audit](/tr/gateway/security#openclaw-security-audit).

  </Accordion>

  <Accordion title="ClawHub skill'lerini ve üçüncü taraf Plugin'lerini yüklemek güvenli mi?">
    Üçüncü taraf skill'leri ve Plugin'leri, güvenmeyi seçtiğiniz kodlar olarak değerlendirin. ClawHub skill sayfaları yüklemeden önce tarama durumunu gösterir ancak taramalar eksiksiz bir güvenlik sınırı değildir. OpenClaw, Plugin/skill yükleme veya güncelleme sırasında yerleşik yerel tehlikeli kod engellemesi çalıştırmaz; yerel izin verme/engelleme kararları için operatörün yönettiği `security.installPolicy` kullanın.

    Daha güvenli yaklaşım: güvenilir yazarları ve sabitlenmiş sürümleri tercih edin, etkinleştirmeden önce skill'i/Plugin'i okuyun, Plugin/skill izin listelerini dar tutun, güvenilmeyen girdi iş akışlarını en az sayıda araçla sandbox içinde çalıştırın ve üçüncü taraf kodlarına geniş dosya sistemi, yürütme, tarayıcı veya gizli bilgi erişimi vermekten kaçının.

    Ayrıntılar: [Skills](/tr/tools/skills), [Plugin'ler](/tr/tools/plugin), [Güvenlik](/tr/gateway/security).

  </Accordion>

  <Accordion title="Botumun kendi e-posta adresi, GitHub hesabı veya telefon numarası olmalı mı?">
    Çoğu kurulum için evet. Botu ayrı hesaplar ve telefon numaralarıyla yalıtmak, bir sorun çıkması durumunda etki alanını daraltır ve kişisel hesaplarınızı etkilemeden kimlik bilgilerini yenilemeyi veya erişimi iptal etmeyi kolaylaştırır.

    Küçük başlayın: yalnızca gerçekten ihtiyaç duyduğunuz araçlara ve hesaplara erişim verin, gerekirse daha sonra genişletin.

    Docs: [Güvenlik](/tr/gateway/security), [Eşleştirme](/tr/channels/pairing).

  </Accordion>

  <Accordion title="Kısa mesajlarım üzerinde özerklik verebilir miyim ve bu güvenli mi?">
    Kişisel mesajlarınız üzerinde tam özerklik vermenizi **önermiyoruz**. En güvenli yaklaşım: DM'leri **eşleştirme modunda** veya dar bir izin listesinde tutun, sizin adınıza mesaj göndermesi gerekiyorsa **ayrı bir numara veya hesap** kullanın ve göndermeden önce sizin **onaylayacağınız** taslaklar hazırlamasına izin verin.

    Denemek için bunu ayrılmış, yalıtılmış bir hesapta yapın. Bkz. [Güvenlik](/tr/gateway/security).

  </Accordion>

  <Accordion title="Kişisel asistan görevleri için daha ucuz modeller kullanabilir miyim?">
    Evet, agent **yalnızca sohbet amaçlıysa** ve girdi güvenilir ise. Daha küçük katmanlar talimatlarla ele geçirilmeye daha yatkındır; bu nedenle araçların etkin olduğu agent'larda veya güvenilmeyen içerik okunurken bunlardan kaçının. Daha küçük bir model kullanmanız gerekiyorsa araçları sıkı biçimde sınırlandırın ve sandbox içinde çalıştırın. Bkz. [Güvenlik](/tr/gateway/security).
  </Accordion>

  <Accordion title="Telegram'da /start çalıştırdım ancak eşleştirme kodu almadım">
    Eşleştirme kodları **yalnızca** bilinmeyen bir gönderen bota mesaj gönderdiğinde ve `dmPolicy: "pairing"` etkin olduğunda gönderilir; `/start` tek başına kod oluşturmaz.

    Bekleyen istekleri kontrol edin:

    ```bash
    openclaw pairing list telegram
    ```

    Anında erişim için gönderen kimliğinizi izin listesine alın veya bu hesap için `dmPolicy: "open"` ayarlayın.

  </Accordion>

  <Accordion title="WhatsApp: kişilerime mesaj gönderir mi? Eşleştirme nasıl çalışır?">
    Hayır. Varsayılan WhatsApp DM politikası **eşleştirme**dir. Bilinmeyen gönderenler yalnızca bir eşleştirme kodu alır; mesajları **işlenmez**. OpenClaw yalnızca aldığı sohbetlere veya açıkça tetiklediğiniz gönderimlere yanıt verir.

    ```bash
    openclaw pairing approve whatsapp <code>
    openclaw pairing list whatsapp
    ```

    Sihirbazın telefon numarası istemi, kendi DM'lerinize izin verilmesi için **izin listenizi/sahibinizi** ayarlar; otomatik gönderim için kullanılmaz. Kişisel WhatsApp numaranızda bu numarayı kullanın ve `channels.whatsapp.selfChatMode` seçeneğini etkinleştirin.

  </Accordion>
</AccordionGroup>

## Sohbet komutları, görevleri iptal etme ve "durmuyor"

<AccordionGroup>
  <Accordion title="Dahili sistem mesajlarının sohbette görünmesini nasıl engellerim?">
    Çoğu dahili/araç mesajı yalnızca o oturum için **ayrıntılı**, **izleme** veya **akıl yürütme** etkinleştirildiğinde görünür.

    Gördüğünüz sohbette düzeltin:

    ```text
    /verbose off
    /trace off
    /reasoning off
    ```

    Hâlâ çok fazla mesaj varsa: Control UI'daki oturum ayarlarını kontrol edin ve ayrıntılı seçeneğini **devral** olarak ayarlayın; yapılandırmada `verboseDefault: "on"` bulunan bir bot profili kullanmadığınızı doğrulayın.

    Docs: [Düşünme ve ayrıntılı çıktı](/tr/tools/thinking), [Güvenlik](/tr/gateway/security/index#reasoning-and-verbose-output-in-groups).

  </Accordion>

  <Accordion title="Çalışan bir görevi nasıl durdurur/iptal ederim?">
    İptali tetiklemek için bunlardan herhangi birini **tek başına bir mesaj olarak** (eğik çizgi olmadan) gönderin: `stop`, `stop action`, `stop current action`, `stop run`, `stop current run`, `stop agent`, `stop the agent`, `stop openclaw`, `openclaw stop`, `stop don't do anything`, `stop do not do anything`, `stop doing anything`, `do not do that`, `please stop`, `stop please`, `abort`, `esc`, `exit`, `interrupt`, `halt`. Yaygın İngilizce dışı tetikleyiciler (Fransızca, Almanca, İspanyolca, Çince, Japonca, Hintçe, Arapça, Rusça) de çalışır.

    exec aracı tarafından başlatılan arka plan işlemleri için agent'tan şunu çalıştırmasını isteyin:

    ```text
    process action:kill sessionId:XXX
    ```

    Çoğu eğik çizgi komutu, `/` ile başlayan **tek başına** bir mesaj olarak gönderilmelidir; ancak bazı kısayollar (`/status` gibi), izin listesindeki gönderenler için satır içinde de çalışır. Bkz. [Eğik çizgi komutları](/tr/tools/slash-commands).

  </Accordion>

  <Accordion title='Telegram üzerinden nasıl Discord mesajı gönderirim? ("Bağlamlar arası mesajlaşma reddedildi")'>
    OpenClaw, **sağlayıcılar arası** mesajlaşmayı varsayılan olarak engeller. Bir araç çağrısı Telegram'a bağlıysa, açıkça izin vermediğiniz sürece Discord'a gönderim yapmaz. Bu değişiklik hemen yürürlüğe girer ve Gateway'in yeniden başlatılması gerekmez:

    ```json5
    {
      tools: {
        message: {
          crossContext: {
            allowAcrossProviders: true,
            marker: { enabled: true, prefix: "[from {channel}] " },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title='Bot neden art arda hızla gönderilen mesajları "yok sayıyor" gibi görünüyor?'>
    Çalışma sırasında gönderilen istemler varsayılan olarak etkin çalışmaya yönlendirilir. Etkin çalışma davranışını seçmek için `/queue` kullanın:

    - `steer` (varsayılan) - etkin çalışmayı bir sonraki model sınırında yönlendirir.
    - `followup` - mesajları kuyruğa alır ve mevcut çalışma sona erdikten sonra bunları teker teker çalıştırır.
    - `collect` - uyumlu mesajları kuyruğa alır ve mevcut çalışma sona erdikten sonra tek seferde yanıt verir.
    - `interrupt` - mevcut çalışmayı iptal eder ve baştan başlar.

    Kuyruklu modlara `debounce:0.5s cap:25 drop:summarize` gibi seçenekler ekleyin. Bkz. [Komut kuyruğu](/tr/concepts/queue) ve [Yönlendirme kuyruğu](/tr/concepts/queue-steering).

  </Accordion>
</AccordionGroup>

## Çeşitli

<AccordionGroup>
  <Accordion title='API anahtarıyla Anthropic için varsayılan model nedir?'>
    Kimlik bilgileri ve model seçimi birbirinden ayrıdır. `ANTHROPIC_API_KEY` ayarını yapmak (veya kimlik doğrulama profillerinde bir Anthropic API anahtarı saklamak) kimlik doğrulamayı etkinleştirir; ancak gerçek varsayılan model, `agents.defaults.model.primary` içinde yapılandırdığınız modeldir (örneğin `anthropic/claude-sonnet-4-6` veya `anthropic/claude-opus-4-6`). `No credentials found for profile "anthropic:default"`, Gateway'in çalışan agent için beklenen `auth-profiles.json` içinde Anthropic kimlik bilgilerini bulamadığı anlamına gelir.
  </Accordion>
</AccordionGroup>

---

Hâlâ çözemediniz mi? [Discord](https://discord.com/invite/clawd) içinde sorun veya bir [GitHub tartışması](https://github.com/openclaw/openclaw/discussions) açın.

## İlgili

- [İlk çalıştırma SSS](/tr/help/faq-first-run) - kurulum, ilk yapılandırma, kimlik doğrulama, abonelikler, ilk hatalar
- [Modeller SSS](/tr/help/faq-models) - model seçimi, yük devretme, kimlik doğrulama profilleri
- [Sorun giderme](/tr/help/troubleshooting) - önce belirtiye göre triyaj
