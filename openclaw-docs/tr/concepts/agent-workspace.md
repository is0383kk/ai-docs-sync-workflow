---
read_when:
    - Aracı çalışma alanını veya dosya düzenini açıklamanız gerekir
    - Bir agent çalışma alanını yedeklemek veya taşımak istiyorsunuz
sidebarTitle: Agent workspace
summary: 'Agent çalışma alanı: konum, düzen ve yedekleme stratejisi'
title: Ajan çalışma alanı
x-i18n:
    generated_at: "2026-07-26T23:53:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b58ead9079c3dda4bcaec3253f8d55e67e7e554d5c5b87ccfec6b08ec4ba038f
    source_path: concepts/agent-workspace.md
    workflow: 16
---

Çalışma alanı aracının evidir: dosya araçları ve çalışma alanı bağlamı için kullanılan çalışma dizinidir. Bunu gizli tutun ve bellek olarak değerlendirin.

Bu, yapılandırmayı, kimlik bilgilerini ve oturumları depolayan `~/.openclaw/` konumundan ayrıdır.

<Warning>
Çalışma alanı **varsayılan cwd**'dir, katı bir korumalı alan değildir. Araçlar göreli yolları çalışma alanına göre çözümler, ancak korumalı alan etkinleştirilmediği sürece mutlak yollar ana makinenin başka yerlerine erişebilir. Yalıtım gerekiyorsa [`agents.defaults.sandbox`](/tr/gateway/sandboxing) (ve/veya aracı başına korumalı alan yapılandırması) kullanın.

Korumalı alan etkinleştirildiğinde ve `workspaceAccess`, `"rw"` olmadığında araçlar, ana makine çalışma alanınızda değil, `~/.openclaw/sandboxes` altındaki bir korumalı alan çalışma alanında çalışır.
</Warning>

## Varsayılan konum

- Varsayılan: `~/.openclaw/workspace`
- `OPENCLAW_PROFILE` ayarlanmışsa ve `"default"` değilse varsayılan, `~/.openclaw/workspace-<profile>` olur.
- `OPENCLAW_WORKSPACE_DIR` ayarlandığında yukarıdakilerin ikisini de geçersiz kılar.
- Açıkça belirtilmiş bir çalışma alanı olmayan varsayılan dışı aracılar (`agents.entries.*`), paylaşılan varsayılan çalışma alanına değil, `<state-dir>/workspace-<agentId>` konumuna çözümlenir.

`~/.openclaw/openclaw.json` içinde geçersiz kılın:

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
    },
  },
}
```

Aracı başına geçersiz kılma: `agents.entries.*.workspace`.

`openclaw onboard`, `openclaw configure` veya `openclaw setup`, çalışma alanını oluşturur ve eksiklerse başlangıç dosyalarını yerleştirir.

<Note>
Korumalı alan başlangıç kopyaları yalnızca çalışma alanı içindeki normal dosyaları kabul eder; kaynak çalışma alanının dışına çözümlenen sembolik bağlantı/sabit bağlantı diğer adları yok sayılır.
</Note>

Çalışma alanı dosyalarını zaten kendiniz yönetiyorsanız başlangıç dosyası oluşturmayı devre dışı bırakın:

```json5
{ agents: { defaults: { skipBootstrap: true } } }
```

## Ek çalışma alanı klasörleri

Eski kurulumlar `~/openclaw` oluşturmuş olabilir. Aynı anda yalnızca bir çalışma alanı etkin olduğundan, birden fazla çalışma alanı dizinini tutmak kafa karıştırıcı kimlik doğrulama veya durum sapmalarına neden olabilir.

<Note>
**Öneri:** tek bir etkin çalışma alanı tutun. Ek klasörleri artık kullanmıyorsanız arşivleyin veya Çöp Kutusu'na taşıyın (örneğin `trash ~/openclaw`). Bilerek birden fazla çalışma alanı tutuyorsanız `agents.defaults.workspace` (veya aracı başına `workspace` anahtarı) etkin olanı göstermelidir.
</Note>

## Çalışma alanı dosya haritası

OpenClaw'ın çalışma alanında bulunmasını beklediği standart dosyalar:

<AccordionGroup>
  <Accordion title="AGENTS.md - çalışma talimatları">
    Aracıya ve belleği nasıl kullanması gerektiğine ilişkin çalışma talimatları. Her oturumun başında yüklenir. Kurallar, öncelikler ve "nasıl davranılacağına" ilişkin ayrıntılar için uygun bir yerdir.
  </Accordion>
  <Accordion title="SOUL.md - kişilik ve üslup">
    Kişilik, üslup ve sınırlar. Her oturumda yüklenir. Kılavuz: [SOUL.md kişilik kılavuzu](/tr/concepts/soul).
  </Accordion>
  <Accordion title="USER.md - kullanıcının kim olduğu">
    Kullanıcının kim olduğu ve kendisine nasıl hitap edileceği. Her oturumda yüklenir.
  </Accordion>
  <Accordion title="IDENTITY.md - ad, hava, emoji">
    Aracının adı, havası ve emojisi. Başlangıç ritüeli sırasında oluşturulur/güncellenir.
  </Accordion>
  <Accordion title="TOOLS.md - yerel araç kuralları">
    Yerel araçlarınız ve kurallarınız hakkındaki notlar. Araçların kullanılabilirliğini kontrol etmez; yalnızca yol gösterir.
  </Accordion>
  <Accordion title="HEARTBEAT.md - heartbeat kontrol listesi">
    Heartbeat çalıştırmaları için isteğe bağlı küçük kontrol listesi. Token tüketimini önlemek için kısa tutun.
  </Accordion>
  <Accordion title="BOOT.md - başlatma kontrol listesi">
    Gateway yeniden başlatıldığında ([dahili hook'lar](/tr/automation/hooks) etkinse) otomatik olarak çalıştırılan isteğe bağlı başlatma kontrol listesi. Kısa tutun; dışarıya gönderimler için mesaj aracını kullanın.
  </Accordion>
  <Accordion title="BOOTSTRAP.md - ilk çalıştırma ritüeli">
    Bir defalık ilk çalıştırma ritüeli. Yalnızca yepyeni bir çalışma alanı için oluşturulur. Ritüel tamamlandıktan sonra silin.
  </Accordion>
  <Accordion title="memory/YYYY-MM-DD.md - günlük bellek kaydı">
    Günlük bellek kaydı (günde bir dosya). Oturum başlangıcında bugünün ve dünün kayıtlarının okunması önerilir.
  </Accordion>
  <Accordion title="MEMORY.md - düzenlenmiş uzun vadeli bellek (isteğe bağlı)">
    Düzenlenmiş uzun vadeli bellek: kalıcı olgular, tercihler, kararlar ve kısa özetler. Ayrıntılı kayıtları `memory/YYYY-MM-DD.md` içinde tutun; böylece bellek araçları, bunları her isteme eklemeden gerektiğinde getirebilir. `MEMORY.md` dosyasını yalnızca ana, özel oturumda yükleyin (paylaşılan/grup bağlamlarında değil). İş akışı ve otomatik bellek boşaltımı için [Bellek](/tr/concepts/memory) bölümüne bakın.
  </Accordion>
  <Accordion title="skills/ - çalışma alanı Skills'ları (isteğe bağlı)">
    Çalışma alanına özgü Skills. Adlar çakıştığında proje aracı Skills'ları, kişisel aracı Skills'ları, yönetilen Skills, paketlenmiş Skills ve `skills.load.extraDirs` konumundan önce gelen, o çalışma alanı için en yüksek öncelikli Skill konumudur.
  </Accordion>
  <Accordion title="canvas/ - Canvas kullanıcı arayüzü dosyaları (isteğe bağlı)">
    Node gösterimleri için Canvas kullanıcı arayüzü dosyaları (örneğin `canvas/index.html`).
  </Accordion>
</AccordionGroup>

<Note>
Bir başlangıç dosyası eksikse OpenClaw, oturuma bir "eksik dosya" işareti ekler ve devam eder. Büyük başlangıç dosyaları eklenirken kesilir; sınırları `agents.defaults.bootstrapMaxChars` (varsayılan: `20000`) ve `agents.defaults.bootstrapTotalMaxChars` (varsayılan: `60000`) ile ayarlayın. `openclaw setup`, mevcut dosyaların üzerine yazmadan eksik varsayılanları yeniden oluşturabilir.
</Note>

## Çalışma alanında OLMAYANLAR

Bunlar `~/.openclaw/` altında bulunur ve çalışma alanı deposuna kaydedilmemelidir:

- `~/.openclaw/openclaw.json` (yapılandırma)
- `~/.openclaw/state/openclaw.sqlite` (paylaşılan çalışma alanı kurulum durumu ve tasdikler)
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (model kimlik doğrulama profilleri: OAuth + API anahtarları)
- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` (oturum satırları, transkriptler ve aracı başına çalışma zamanı durumu)
- `~/.openclaw/agents/<agentId>/agent/codex-home/` (aracı başına Codex çalışma zamanı hesabı, yapılandırması, Skills'ları, Plugin'leri ve yerel iş parçacığı durumu)
- `~/.openclaw/credentials/` (kanal/sağlayıcı durumu ve eski OAuth içe aktarma verileri)
- `~/.openclaw/agents/<agentId>/sessions/` (eski taşıma kaynakları ve arşiv/destek yapıtları)
- `~/.openclaw/skills/` (yönetilen Skills)

Oturumları veya yapılandırmayı taşımanız gerekiyorsa bunları ayrı olarak kopyalayın ve sürüm denetiminin dışında tutun.

Eski OpenClaw sürümleri, çalışma alanı yan dosyaları olan `openclaw-workspace-state.json`,
`.openclaw/workspace-state.json` ve `.attested` dosyalarını yazıyordu. Geçerli
çalışma zamanı bu durum için yalnızca paylaşılan SQLite veritabanını kullanır. Doctor bu
dosyalardan birini bildirirse `openclaw doctor --fix` komutunu çalıştırın; Doctor, geçerli eski
durumu içe aktarır ve bir kaynağı yalnızca veritabanı satırlarını doğruladıktan sonra siler.

## Git yedeklemesi (önerilir, özel)

Çalışma alanını özel bellek olarak değerlendirin. Yedeklenebilmesi ve kurtarılabilmesi için **özel** bir git deposuna koyun.

Bu adımları Gateway'in çalıştığı makinede çalıştırın (çalışma alanı burada bulunur).

<Steps>
  <Step title="Depoyu başlatın">
    Git yüklüyse yepyeni çalışma alanları otomatik olarak başlatılır. Bu çalışma alanı henüz bir depo değilse şunu çalıştırın:

    ```bash
    cd ~/.openclaw/workspace
    git init
    git add AGENTS.md SOUL.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md memory/
    git commit -m "Add agent workspace"
    ```

  </Step>
  <Step title="Özel bir uzak depo ekleyin">
    <Tabs>
      <Tab title="GitHub web UI">
        1. GitHub'da yeni bir **özel** depo oluşturun.
        2. Bir README ile başlatmayın (birleştirme çakışmalarını önler).
        3. HTTPS uzak depo URL'sini kopyalayın.
        4. Uzak depoyu ekleyip gönderin:

        ```bash
        git branch -M main
        git remote add origin <https-url>
        git push -u origin main
        ```
      </Tab>
      <Tab title="GitHub CLI (gh)">
        ```bash
        gh auth login
        gh repo create openclaw-workspace --private --source . --remote origin --push
        ```
      </Tab>
      <Tab title="GitLab web UI">
        1. GitLab'da yeni bir **özel** depo oluşturun.
        2. Bir README ile başlatmayın (birleştirme çakışmalarını önler).
        3. HTTPS uzak depo URL'sini kopyalayın.
        4. Uzak depoyu ekleyip gönderin:

        ```bash
        git branch -M main
        git remote add origin <https-url>
        git push -u origin main
        ```
      </Tab>
    </Tabs>

  </Step>
  <Step title="Süregelen güncellemeler">
    ```bash
    git status
    git add .
    git commit -m "Update memory"
    git push
    ```
  </Step>
</Steps>

## Gizli bilgileri kaydetmeyin

<Warning>
Özel bir depoda bile gizli bilgileri çalışma alanında depolamaktan kaçının:

- API anahtarları, OAuth token'ları, parolalar veya özel kimlik bilgileri.
- `~/.openclaw/` altındaki herhangi bir şey.
- Sohbetlerin ham dökümleri veya hassas ekler.

Hassas referansları depolamanız gerekiyorsa yer tutucular kullanın ve gerçek gizli bilgiyi başka bir yerde tutun (parola yöneticisi, ortam değişkenleri veya `~/.openclaw/`).
</Warning>

Önerilen `.gitignore` başlangıç içeriği:

```gitignore
.DS_Store
.env
**/*.key
**/*.pem
**/secrets*
```

## Çalışma alanını yeni bir makineye taşıma

<Steps>
  <Step title="Depoyu klonlayın">
    Depoyu istediğiniz yola klonlayın (varsayılan `~/.openclaw/workspace`).
  </Step>
  <Step title="Yapılandırmayı güncelleyin">
    `~/.openclaw/openclaw.json` içinde `agents.defaults.workspace` değerini bu yol olarak ayarlayın.
  </Step>
  <Step title="Eksik dosyaları yerleştirin">
    Eksik dosyaları yerleştirmek için `openclaw setup --workspace <path>` komutunu çalıştırın.
  </Step>
  <Step title="Oturumları kopyalayın (isteğe bağlı)">
    Oturumlara ihtiyacınız varsa `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
    konumunu eski makineden ayrı olarak kopyalayın. `~/.openclaw/agents/<agentId>/sessions/`
    konumunu yalnızca eski taşıma girdilerine veya arşiv/destek yapıtlarına da ihtiyacınız olduğunda kopyalayın.
  </Step>
</Steps>

## Gelişmiş notlar

- Çok aracılı yönlendirme, `agents.entries.*.workspace` aracılığıyla aracı başına farklı çalışma alanları kullanabilir. Yönlendirme yapılandırması için [Kanal yönlendirme](/tr/channels/channel-routing) bölümüne bakın.
- `agents.defaults.sandbox` etkinse ana olmayan oturumlar, `agents.defaults.sandbox.workspaceRoot` altındaki oturum başına korumalı alan çalışma alanlarını kullanabilir.

## İlgili konular

- [Heartbeat](/tr/gateway/heartbeat) - HEARTBEAT.md çalışma alanı dosyası
- [Korumalı alan](/tr/gateway/sandboxing) - korumalı alan ortamlarında çalışma alanı erişimi
- [Oturum](/tr/concepts/session) - oturum depolama yolları
- [Kalıcı talimatlar](/tr/automation/standing-orders) - çalışma alanı dosyalarındaki kalıcı talimatlar
