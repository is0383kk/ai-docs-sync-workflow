---
read_when:
    - Hermes veya başka bir ajan sisteminden OpenClaw'a geçiş yapmak istiyorsunuz
    - Plugin tarafından sahip olunan bir geçiş sağlayıcısı ekliyorsunuz
summary: '`openclaw migrate` için CLI referansı (başka bir ajan sisteminden durumu içe aktarma)'
title: Geçiş Yap
x-i18n:
    generated_at: "2026-07-26T23:52:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f492535019f8a69706ff918462ba74cf5d26e733d2e4e9493b3c76bd77f2584d
    source_path: cli/migrate.md
    workflow: 16
---

# `openclaw migrate`

Başka bir ajan sisteminden durumu, Plugin'in sahip olduğu bir geçiş sağlayıcısı aracılığıyla içe aktarın. Birlikte sunulan sağlayıcılar Claude, Codex CLI ve [Hermes](/tr/install/migrating-hermes) desteğini kapsar; Plugin'ler ek sağlayıcılar kaydedebilir.

<Tip>
Kullanıcıya yönelik adım adım kılavuzlar için [Claude'dan geçiş](/tr/install/migrating-claude) ve [Hermes'ten geçiş](/tr/install/migrating-hermes) sayfalarına bakın. [Geçiş merkezi](/tr/install/migrating) tüm yolları listeler.
</Tip>

## Komutlar

```bash
openclaw migrate list
openclaw migrate claude --dry-run
openclaw migrate codex --dry-run
openclaw migrate codex --skill gog-vault77-google-workspace
openclaw migrate codex --plugin google-calendar --dry-run
openclaw migrate codex --plugin google-calendar --verify-plugin-apps --dry-run
openclaw migrate hermes --dry-run
openclaw migrate hermes
openclaw migrate apply codex --yes --skill gog-vault77-google-workspace
openclaw migrate apply codex --yes --plugin google-calendar
openclaw migrate apply codex --yes
openclaw migrate apply claude --yes
openclaw migrate apply hermes --yes
openclaw migrate apply hermes --include-secrets --yes
openclaw onboard --flow import
openclaw onboard --import-from claude --import-source ~/.claude
openclaw onboard --import-from hermes --import-source ~/.hermes
```

`openclaw migrate <provider>` başka hiçbir bayrak olmadan çalıştırıldığında uygulamadan önce planlama ve önizleme yapar ve (bir TTY'de) onay ister. `openclaw migrate plan <provider>` ve `openclaw migrate apply <provider>`, aynı bayrakları kullanarak önizleme ve uygulama işlemlerini ayrı alt komutlara böler.

<ParamField path="<provider>" type="string">
  Kayıtlı bir geçiş sağlayıcısının adı; örneğin `hermes`. Yüklü sağlayıcıları görmek için `openclaw migrate list` komutunu çalıştırın.
</ParamField>
<ParamField path="--dry-run" type="boolean">
  Planı oluşturun ve durumu değiştirmeden çıkın.
</ParamField>
<ParamField path="--from <path>" type="string">
  Kaynak durum dizinini geçersiz kılın. Hermes, `$HERMES_HOME` ve etkin profili izler, ardından platform varsayılanını (`~/.hermes` veya `%LOCALAPPDATA%\hermes`) kullanır. Codex varsayılan olarak `~/.codex` (veya `$CODEX_HOME`), Claude ise `~/.claude` kullanır.
</ParamField>
<ParamField path="--include-secrets" type="boolean">
  Desteklenen kimlik bilgilerini onay istemeden içe aktarın. Etkileşimli uygulama, algılanan kimlik doğrulama bilgilerini içe aktarmadan önce varsayılan olarak evet seçili biçimde onay ister; etkileşimsiz `--yes`, bunları içe aktarmak için `--include-secrets` gerektirir.
</ParamField>
<ParamField path="--no-auth-credentials" type="boolean">
  Etkileşimli istem dâhil olmak üzere kimlik doğrulama bilgilerinin içe aktarımını atlayın.
</ParamField>
<ParamField path="--overwrite" type="boolean">
  Plan çakışmalar bildirdiğinde uygulamanın mevcut hedefleri değiştirmesine izin verin.
</ParamField>
<ParamField path="--yes" type="boolean">
  Onay istemini atlayın. Etkileşimsiz modda gereklidir.
</ParamField>
<ParamField path="--skill <name>" type="string">
  Skill adına veya öğe kimliğine göre bir Skill kopyalama öğesi seçin. Birden fazla Skill'i geçirmek için bayrağı yineleyin. Belirtilmediğinde etkileşimli Codex geçişleri bir onay kutusu seçicisi gösterir; etkileşimsiz geçişler ise planlanan tüm Skill'leri korur.
</ParamField>
<ParamField path="--plugin <name>" type="string">
  Plugin adına veya öğe kimliğine göre bir Codex Plugin yükleme öğesi seçin. Birden fazla Codex Plugin'ini geçirmek için bayrağı yineleyin. Belirtilmediğinde etkileşimli Codex geçişleri yerel bir Codex Plugin onay kutusu seçicisi gösterir; etkileşimsiz geçişler ise planlanan tüm Plugin'leri korur. Yalnızca Codex uygulama sunucusu envanterinin keşfettiği, kaynaktan yüklenmiş `openai-curated` Codex Plugin'leri için geçerlidir.
</ParamField>
<ParamField path="--verify-plugin-apps" type="boolean">
  Yalnızca Codex. Yerel Plugin etkinleştirmesini planlamadan önce kaynak Codex uygulama sunucusunda yeni bir `app/list` dolaşmasını zorunlu kılar. Geçiş planlamasını hızlı tutmak için varsayılan olarak kapalıdır.
</ParamField>
<ParamField path="--backup-output <path>" type="string">
  Geçiş öncesi yedekleme arşivinin yolu veya dizini. `openclaw backup create` öğesine aynen aktarılır.
</ParamField>
<ParamField path="--no-backup" type="boolean">
  Uygulama öncesi yedeklemeyi atlayın. Yerel OpenClaw durumu mevcutsa `--force` gerektirir.
</ParamField>
<ParamField path="--force" type="boolean">
  Uygulama işlemi aksi hâlde yedeklemeyi atlamayı reddedecekse `--no-backup` ile birlikte gereklidir.
</ParamField>
<ParamField path="--json" type="boolean">
  Planı veya uygulama sonucunu JSON olarak yazdırın. `--json` kullanıldığında ve `--yes` kullanılmadığında uygulama, planı yazdırır ve durumu değiştirmez.
</ParamField>

## Güvenlik modeli

`openclaw migrate` öncelikle önizleme yapar.

<AccordionGroup>
  <Accordion title="Uygulamadan önce önizleme">
    Sağlayıcı, herhangi bir şey değişmeden önce çakışmaları, atlanan öğeleri ve hassas öğeleri içeren ayrıntılı bir plan döndürür. JSON planları, uygulama çıktısı ve geçiş raporları; API anahtarları, token'lar, yetkilendirme üst bilgileri, çerezler ve parolalar gibi iç içe geçmiş, gizli bilgi izlenimi veren anahtarları gizler.

    `openclaw migrate apply <provider>`, planı önizler ve `--yes` ayarlanmadıkça durumu değiştirmeden önce onay ister. Etkileşimsiz modda uygulama için `--yes` gerekir.

  </Accordion>
  <Accordion title="Yedeklemeler">
    Uygulama, geçişi uygulamadan önce bir OpenClaw yedeği oluşturur ve doğrular. Henüz yerel OpenClaw durumu yoksa yedekleme adımı atlanır ve geçiş devam eder. Durum mevcutken yedeklemeyi atlamak için hem `--no-backup` hem de `--force` öğelerini geçirin.
  </Accordion>
  <Accordion title="Çakışmalar">
    Planda çakışmalar olduğunda uygulama devam etmeyi reddeder. Planı inceleyin; mevcut hedeflerin değiştirilmesi amaçlanıyorsa işlemi `--overwrite` ile yeniden çalıştırın. Sağlayıcılar, üzerine yazılan dosyalar için geçiş raporu dizinine yine de öğe düzeyinde yedekler yazabilir.
  </Accordion>
  <Accordion title="Gizli bilgiler">
    Etkileşimli uygulama, algılanan kimlik doğrulama bilgilerinin içe aktarılıp aktarılmayacağını varsayılan olarak evet seçili biçimde sorar. Bunları atlamak için `--no-auth-credentials`, `--yes` ile gözetimsiz kimlik bilgisi içe aktarımı içinse `--include-secrets` kullanın.
  </Accordion>
</AccordionGroup>

## Claude sağlayıcısı

Birlikte sunulan Claude sağlayıcısı, varsayılan olarak `~/.claude` konumundaki Claude Code durumunu algılar. Belirli bir Claude Code ana dizinini veya proje kökünü içe aktarmak için `--from <path>` kullanın.

<Tip>
Kullanıcıya yönelik adım adım kılavuz için [Claude'dan geçiş](/tr/install/migrating-claude) sayfasına bakın.
</Tip>

### Claude'un içe aktardıkları

- `~/.claude/projects/*/memory` konumundaki Claude Code otomatik bellek Markdown'ı ve kullanıcı tarafından yapılandırılan bir
  `autoMemoryDirectory`; dizinlenmiş geri çağırma için
  `memory/imports/claude-code/` altına kopyalanır.
- Proje `CLAUDE.md` ve `.claude/CLAUDE.md` dosyaları, OpenClaw ajan çalışma alanına (`AGENTS.md`) aktarılır.
- Kullanıcı `~/.claude/CLAUDE.md`, çalışma alanındaki `USER.md` dosyasına eklenir.
- Proje `.mcp.json`, Claude Code `~/.claude.json` (proje başına girdileri dâhil) ve Claude Desktop `claude_desktop_config.json` kaynaklarındaki MCP sunucusu tanımları.
- `SKILL.md` içeren Claude Skill dizinleri (kullanıcı `~/.claude/skills` ve proje `.claude/skills`).
- Claude komut Markdown dosyaları (kullanıcı `~/.claude/commands` ve proje `.claude/commands`), yalnızca elle çağrılabilen OpenClaw Skill'lerine dönüştürülür.

### Arşiv ve elle inceleme durumu

Claude kancaları, izinleri, ortam varsayılanları, proje `CLAUDE.local.md`, `.claude/rules`, kullanıcı ve proje `agents/` dizinleri ile proje geçmişi (`~/.claude` altındaki `projects`, `cache`, `plans`) geçiş raporunda korunur veya elle incelenecek öğeler olarak bildirilir. OpenClaw; kancaları çalıştırmaz, geniş izin listelerini kopyalamaz veya OAuth/Desktop kimlik bilgisi durumunu otomatik olarak içe aktarmaz.

## Codex sağlayıcısı

Birlikte sunulan Codex sağlayıcısı, varsayılan olarak `~/.codex` konumundaki Codex CLI durumunu veya ilgili ortam değişkeni ayarlandığında `CODEX_HOME` konumundaki durumu algılar. Belirli bir Codex ana dizininin envanterini çıkarmak için `--from <path>` kullanın.

OpenClaw Codex altyapısına geçerken ve yararlı kişisel Codex CLI varlıklarını bilinçli olarak aktarmak istediğinizde bu sağlayıcıyı kullanın. Yerel Codex uygulama sunucusu başlatmaları ajan başına bir `CODEX_HOME` kullanır; bu nedenle varsayılan olarak kişisel `~/.codex` öğenizi okumaz. Normal süreç `HOME` yine devralındığından Codex, paylaşılan `$HOME/.agents/*` Skill/Plugin pazar yeri girdilerini görebilir ve alt süreçler kullanıcı ana dizinindeki yapılandırma ile token'ları bulabilir.

`openclaw migrate codex` etkileşimli bir terminalde çalıştırıldığında tam planı önizler, ardından son uygulama onayından önce onay kutusu seçicilerini açar. Önce Skill kopyalama öğeleri sorulur. Toplu seçim için `Toggle all on` veya `Toggle all off` kullanın. Satırların seçimini değiştirmek için Space tuşuna, vurgulanan satırı etkinleştirip devam etmek için Enter tuşuna basın. Planlanan Skill'ler seçili, çakışan Skill'ler seçilmemiş olarak başlar; `Skip for now`, Plugin seçimine devam ederken bu çalıştırmadaki Skill kopyalamalarını atlar. Kaynaktan yüklenmiş, seçilmiş Codex Plugin'leri geçirilebiliyorsa ve `--plugin` belirtilmemişse geçiş daha sonra Plugin adına göre yerel Codex Plugin etkinleştirmesi ister. Hedef OpenClaw Codex Plugin yapılandırmasında ilgili Plugin zaten bulunmadığı sürece Plugin öğeleri seçili olarak başlar. Mevcut hedef Plugin'ler seçilmemiş olarak başlar ve `conflict: plugin exists` gibi bir çakışma ipucu gösterir; bu çalıştırmada hiçbir yerel Codex Plugin'ini geçirmemek için `Toggle all off`, uygulamadan önce durmak içinse `Skip for now` seçeneğini belirleyin.

Betikli veya kesin çalıştırmalar için bir ya da daha fazla Skill veya Plugin'i açıkça seçin:

```bash
openclaw migrate codex --dry-run --skill gog-vault77-google-workspace
openclaw migrate apply codex --yes --skill gog-vault77-google-workspace
openclaw migrate codex --dry-run --plugin google-calendar
openclaw migrate apply codex --yes --plugin google-calendar
```

### Codex'in içe aktardıkları

- `$CODEX_HOME/memories` konumundaki birleştirilmiş Codex `MEMORY.md` ve `memory_summary.md`,
  dizinlenmiş geri çağırma için `memory/imports/codex/` altına
  kopyalanır. Ham rollout belleği içe aktarılmaz.
- `$CODEX_HOME/skills` altındaki Codex CLI Skill dizinleri; Codex'in `.system` önbelleği hariç tutulur.
- `$HOME/.agents/skills` altındaki kişisel AgentSkills, ajan başına sahiplik için geçerli OpenClaw ajan çalışma alanına kopyalanır.
- Codex uygulama sunucusu `plugin/list` aracılığıyla keşfedilen, kaynaktan yüklenmiş `openai-curated` Codex Plugin'leri. Planlama, etkinleştirilmiş ve yüklenmiş her Plugin için `plugin/read` öğesini okur.

Uygulama destekli Plugin geçişinde ek denetimler bulunur:

- Uygulama destekli Plugin'ler, kaynak Codex uygulama sunucusu hesabının bir ChatGPT abonelik hesabı olmasını gerektirir. ChatGPT dışındaki veya eksik hesap yanıtları `codex_subscription_required` ile atlanır.
- Geçiş, varsayılan olarak kaynak `app/list` çağrısını yapmaz; bu nedenle hesap denetimini geçen uygulama destekli Plugin'ler kaynak uygulama erişilebilirliği doğrulanmadan planlanır ve hesap arama aktarım hataları `codex_account_unavailable` ile atlanır.
- Yeni bir kaynak `app/list` anlık görüntüsünü zorunlu kılmak ve yerel etkinleştirmeyi planlamadan önce sahip olunan her uygulamanın mevcut, etkin ve erişilebilir olmasını gerektirmek için `--verify-plugin-apps` geçirin. Bu modda hesap arama aktarım hataları, kaynak uygulama envanteri doğrulamasına geçer. Anlık görüntü yalnızca geçerli işlem boyunca bellekte tutulur; hiçbir zaman geçiş çıktısına veya hedef yapılandırmasına yazılmaz.

Devre dışı Plugin'ler, okunamayan Plugin ayrıntıları, abonelik kısıtlamalı kaynak hesaplar ve (`--verify-plugin-apps` ayarlandığında) eksik, devre dışı veya erişilemeyen uygulamalar; hedef yapılandırma girdileri yerine türü belirtilmiş nedenlerle elle atlanan öğelere dönüşür. Uygulama, hedef uygulama sunucusu ilgili Plugin'in zaten yüklenmiş ve etkin olduğunu bildirse bile seçilen her uygun Plugin için uygulama sunucusu `plugin/install` çağrısını yapar. Geçirilen Codex Plugin'leri yalnızca yerel Codex altyapısını seçen oturumlarda kullanılabilir; OpenClaw sağlayıcı çalıştırmalarına, ACP konuşma bağlamalarına veya diğer altyapılara sunulmaz.

### Elle incelenecek Codex durumu

Codex `config.toml`, yerel `hooks/hooks.json`, seçkili olmayan pazaryerleri, kaynak üzerinden kurulmuş seçkili plugin olmayan önbelleğe alınmış plugin paketleri ve kaynak aboneliği geçidini geçemeyen kaynak üzerinden kurulmuş plugin otomatik olarak etkinleştirilmez. `--verify-plugin-apps` ayarlandığında, kaynak uygulama envanteri geçidini geçemeyen plugin de atlanır. Bunların tümü, elle incelenmek üzere taşıma raporuna kopyalanır veya raporda bildirilir.

Taşınan, kaynak üzerinden kurulmuş seçkili plugin için şu yazma işlemleri uygulanır:

- `plugins.entries.codex.enabled: true`
- `plugins.entries.codex.config.codexPlugins.enabled: true`
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions: true`
- seçilen her plugin için `marketplaceName: "openai-curated"` ve `pluginName` içeren açık bir plugin girdisi

Taşıma hiçbir zaman `plugins["*"]` yazmaz ve yerel pazaryeri önbellek yollarını hiçbir zaman depolamaz.

Atlanan plugin hedef yapılandırmaya yazılmaz. Kaynak tarafındaki abonelik hataları, elle incelenecek öğelerde şu türlendirilmiş nedenlerle bildirilir: `codex_subscription_required`, `codex_account_unavailable`, `plugin_disabled` veya `plugin_read_unavailable`. `--verify-plugin-apps` ile kaynak uygulama envanteri hataları ayrıca `app_inaccessible`, `app_disabled`, `app_missing` veya `app_inventory_unavailable` olarak görünebilir. Hedef tarafında kimlik doğrulaması gerektiren kurulumlar, etkilenen plugin öğesinde `status: "skipped"`, `reason: "auth_required"` ve arındırılmış uygulama tanımlayıcılarıyla bildirilir; bunların açık yapılandırma girdileri, yeniden yetkilendirip etkinleştirene kadar devre dışı olarak yazılır. Diğer kurulum hataları, öğe kapsamlı `error` sonuçlarıdır.

Planlama sırasında Codex uygulama sunucusu plugin envanterine erişilemiyorsa taşıma, tüm taşıma işlemini başarısız kılmak yerine önbelleğe alınmış paket danışma öğelerine geri döner.

## Hermes sağlayıcısı

Paketle gelen Hermes sağlayıcısı `$HERMES_HOME` ve etkin profili izler, ardından platform varsayılanını (`~/.hermes` veya `%LOCALAPPDATA%\hermes`) kullanır. Keşfi geçersiz kılmak için `--from <path>` kullanın.

### Hermes'in içe aktardıkları

- `config.yaml` kaynağındaki varsayılan model yapılandırması.
- `model`, `providers` ve `custom_providers` kaynaklarındaki yapılandırılmış model sağlayıcıları ve özel OpenAI uyumlu uç noktalar.
- `mcp_servers` veya `mcp.servers` kaynağındaki MCP sunucusu tanımları. Tam OpenClaw eşlemeleri; varsayılan Streamable HTTP yönlendirmesini, OAuth kapsamını, mantıksal TLS doğrulamasını, ayrı istemci sertifikası/anahtar yollarını ve Hermes yerel/kaynak/istem aracı politikasını kapsar. Desteklenmeyen, yalnızca Hermes'e özgü çalışma zamanı veya kimlik bilgisi alanları elle incelenmek üzere bildirilir.
- `SOUL.md` ve `AGENTS.md`, OpenClaw aracısı çalışma alanına.
- `memories/MEMORY.md` ve `memories/USER.md`, çalışma alanı bellek dosyalarına eklenir.
  Yalnızca belleğe yönelik yüzeyler (ilk katılım bellek sayfası ve Control UI Bellek
  içe aktarma sayfası) ise mevcut çalışma alanı belleğine dokunmadan dizinlenmiş
  geri çağırma için bu dosyaları `memory/imports/hermes/` altına kopyalar.
- OpenClaw dosya belleği için bellek yapılandırması varsayılanları ve Honcho gibi harici bellek sağlayıcıları için arşiv veya elle inceleme öğeleri.
- `skills/` altında herhangi bir yerde `SKILL.md` dosyası içeren Skills; iç içe Skills, çalışma alanı Skills dizininde düzleştirilir.
- `skills.config` kaynağındaki Skill başına yapılandırma değerleri.
- Etkileşimli kimlik bilgisi taşıması kabul edildiğinde veya `--include-secrets` ayarlandığında geçerli Hermes OpenAI Codex OAuth kimlik bilgileri ve OpenCode OpenAI OAuth kimlik bilgileri. Hermes ile OpenClaw'un aynı içe aktarılmış yenileme yetkisini kullanmaya devam etmesine izin vermeyin.
- Etkileşimli kimlik bilgisi taşıması kabul edildiğinde veya `--include-secrets` ayarlandığında Hermes `.env` ve OpenCode `auth.json` kaynaklarındaki desteklenen API anahtarları ve belirteçler.

### Desteklenen `.env` anahtarları

`AI_GATEWAY_API_KEY`, `ALIBABA_API_KEY`, `ANTHROPIC_API_KEY`, `ARCEEAI_API_KEY`, `CEREBRAS_API_KEY`, `CHUTES_API_KEY`, `CLOUDFLARE_AI_GATEWAY_API_KEY`, `COPILOT_GITHUB_TOKEN`, `DASHSCOPE_API_KEY`, `DEEPINFRA_API_KEY`, `DEEPSEEK_API_KEY`, `FIREWORKS_API_KEY`, `GEMINI_API_KEY`, `GH_TOKEN`, `GITHUB_TOKEN`, `GLM_API_KEY`, `GOOGLE_API_KEY`, `GROQ_API_KEY`, `HF_TOKEN`, `HUGGINGFACE_HUB_TOKEN`, `KILOCODE_API_KEY`, `KIMICODE_API_KEY`, `KIMI_API_KEY`, `KIMI_CODING_API_KEY`, `MINIMAX_API_KEY`, `MINIMAX_CODING_API_KEY`, `MISTRAL_API_KEY`, `MODELSTUDIO_API_KEY`, `MOONSHOT_API_KEY`, `NVIDIA_API_KEY`, `OPENAI_API_KEY`, `OPENCODE_API_KEY`, `OPENCODE_GO_API_KEY`, `OPENCODE_ZEN_API_KEY`, `OPENROUTER_API_KEY`, `QIANFAN_API_KEY`, `QWEN_API_KEY`, `TOGETHER_API_KEY`, `VENICE_API_KEY`, `XAI_API_KEY`, `XIAOMI_API_KEY`, `ZAI_API_KEY`, `Z_AI_API_KEY`.

### Yalnızca arşivlenen durum

OpenClaw'un güvenli biçimde yorumlayamadığı Hermes durumu, elle incelenmek üzere taşıma raporuna kopyalanır ancak canlı OpenClaw yapılandırmasına veya kimlik bilgilerine yüklenmez. Buna `plugins/`, `sessions/`, `logs/`, `cron/`, `mcp-tokens/`, `plans/`, `workspace/`, `skins/`, `kanban/`, eşleştirme/platform durumu, Gateway yönlendirme/işlem durumu ve algılanan Hermes SQLite veritabanları dahildir.

### Uyguladıktan sonra

```bash
openclaw doctor
```

## Plugin sözleşmesi

Taşıma kaynakları pluginlerdir. Bir plugin, sağlayıcı kimliklerini `openclaw.plugin.json` içinde bildirir:

```json
{
  "contracts": {
    "migrationProviders": ["hermes"]
  }
}
```

Çalışma zamanında plugin, `api.registerMigrationProvider(...)` çağrısını yapar. Sağlayıcı; `detect`, `plan` ve `apply` uygular. CLI düzenlemesi, yedekleme politikası, istemler, JSON çıktısı ve çakışma ön denetimi çekirdeğin sorumluluğundadır. Çekirdek, incelenmiş planı `apply(ctx, plan)` içine geçirir ve sağlayıcılar yalnızca uyumluluk amacıyla bu bağımsız değişken bulunmadığında planı yeniden oluşturabilir. Taşıma öğeleri, ilk katılımın aşamalı yerel veriler kalıcı olarak yayımlanana kadar ertelemesi gereken harici etkinleştirme etkileri için `applyPhase: "after-promotion"` ayarlayabilir. Bu sağlayıcılar `deferredApply: { retrySafe: true }` bildirmeli ve kesintiye uğramış bir işlemden sonra ertelenmiş her etkinin güvenle yeniden yürütülebilmesini sağlamalıdır; ilk katılım, bildirilmemiş ertelenmiş etkileri reddeder. Eş etkili bir işlem yapmama durumu, kurtarmanın bunu tamamlanmış olarak kaydedebilmesi için `deferredCompletion: true` içeren, değişiklik yapmayan bir öğe döndürmelidir. Bağımsız `openclaw migrate`, eksiksiz planı normal, yedekleme destekli akışı üzerinden uygulamaya devam eder.

Sağlayıcı pluginler, öğe oluşturma ve özet sayımları için `openclaw/plugin-sdk/migration`; çakışmaya duyarlı dosya kopyaları, yalnızca arşivlenen rapor kopyaları, önbelleğe alınmış yapılandırma çalışma zamanı sarmalayıcıları ve taşıma raporları için de `openclaw/plugin-sdk/migration-runtime` kullanabilir.

## İlk katılım entegrasyonu

Bir sağlayıcı bilinen bir kaynak algıladığında ilk katılım taşıma seçeneği sunabilir. Hem `openclaw onboard --flow import` hem de `openclaw setup --wizard --import-from hermes` aynı plugin taşıma sağlayıcısını kullanır ve uygulamadan önce yine de bir önizleme gösterir. Bağımsız taşımadan farklı olarak yeni hedefe yönelik ilk katılım yolu; yerel yapıtları ve içe aktarılmış kimlik bilgilerini aşamalar, içe aktarılmış çıkarımı aşama ortamında doğrular veya onarır, ardından yapılandırmayı işlemeden önce çalışma alanını ve aracı durumunu terfi ettirir. Modu `0600` olan bir terfi günlüğü, sonraki çalıştırmanın kesintiye uğramış bir yayını, ertelenmiş harici etkinleştirmeler dahil olmak üzere, içe aktarılmış yerel verileri yeniden yürütmeden tamamlamasına veya geri almasına olanak tanır.

<Note>
İlk katılım içe aktarmaları yeni bir OpenClaw kurulumu gerektirir. Zaten yerel durumunuz varsa önce yapılandırmayı, kimlik bilgilerini, oturumları ve çalışma alanını sıfırlayın. Yedekleyip üzerine yazma veya birleştirme yoluyla içe aktarmalar, mevcut kurulumlar için özellik geçidine tabidir.
</Note>

## İlgili

- [Hermes'ten taşıma](/tr/install/migrating-hermes): kullanıcıya yönelik adım adım kılavuz.
- [Claude'dan taşıma](/tr/install/migrating-claude): kullanıcıya yönelik adım adım kılavuz.
- [Taşıma](/tr/install/migrating): OpenClaw'u yeni bir makineye taşıma.
- [Doctor](/tr/gateway/doctor): taşıma uygulandıktan sonraki sistem durumu denetimi.
- [Pluginler](/tr/tools/plugin): plugin kurulumu ve kaydı.
