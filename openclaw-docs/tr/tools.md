---
doc-schema-version: 1
read_when:
    - OpenClaw'ın hangi araçları sağladığını anlamak istiyorsunuz
    - Yerleşik araçlar, Skills ve pluginler arasında seçim yapıyorsunuz
    - Araç politikası, otomasyon veya ajan koordinasyonu için doğru dokümantasyon giriş noktasına ihtiyacınız var
summary: 'OpenClaw araçlarına, Skills ve pluginlere genel bakış: ajanların neleri çağırabileceği ve bunların nasıl genişletileceği'
title: Genel Bakış
x-i18n:
    generated_at: "2026-07-26T23:05:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 45745bd5f2008a84cb6c4c1c9840073bfa8a9c40a0ff65bfefc682c5d99af09b
    source_path: tools/index.md
    workflow: 16
---

Doğru Capabilities yüzeyini seçmek için bu sayfayı kullanın. **Araçlar**
çağrılabilir eylemlerdir, **Skills** agent'lara nasıl çalışacaklarını öğretir ve **plugin'ler**;
araçlar, sağlayıcılar, kanallar, hook'lar ve paketlenmiş Skills gibi
çalışma zamanı yetenekleri ekler.

Bu bir genel bakış ve yönlendirme sayfasıdır. Kapsamlı araç politikası, varsayılanlar,
grup üyeliği, sağlayıcı kısıtlamaları ve yapılandırma alanları için
[Araçlar ve özel sağlayıcılar](/tr/gateway/config-tools) sayfasını kullanın.

## Buradan başlayın

Çoğu agent için yerleşik araç kategorileriyle başlayın, ardından politikayı
yalnızca agent'ın daha az araç görmesi veya açık ana makine erişimine ihtiyaç duyması durumunda ayarlayın.

| Şunları yapmanız gerekiyorsa...                            | Önce bunu kullanın                                 | Ardından okuyun                                                                                                                                              |
| -------------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Bir agent'ın mevcut yeteneklerle işlem yapmasını sağlamak  | [Yerleşik araçlar](#built-in-tool-categories)    | [Araç kategorileri](#built-in-tool-categories)                                                                                                           |
| Bir agent'ın neleri çağırabileceğini denetlemek               | [Araç politikası](#configure-access-and-approvals) | [Araçlar ve özel sağlayıcılar](/tr/gateway/config-tools)                                                                                                    |
| Bir agent'a iş akışı öğretmek                    | [Skills](#choose-tools-skills-or-plugins)      | [Skills](/tr/tools/skills), [Skills oluşturma](/tr/tools/creating-skills), [Skill Atölyesi](/tr/tools/skill-workshop) ve [Kendi kendine öğrenme](/tr/tools/self-learning) |
| Yeni bir entegrasyon veya çalışma zamanı yüzeyi eklemek     | [Plugin'ler](#extend-capabilities)                | [Plugin'ler](/tr/tools/plugin) ve [Plugin geliştirme](/tr/plugins/building-plugins)                                                                                |
| Çalışmayı daha sonra veya arka planda yürütmek          | [Otomasyon](/tr/automation)                      | [Otomasyona genel bakış](/tr/automation)                                                                                                                     |
| Birden fazla agent'ı veya harness'ı koordine etmek      | [Alt agent'lar](/tr/tools/subagents)                 | [ACP agent'ları](/tr/tools/acp-agents) ve [Agent gönderimi](/tr/tools/agent-send)                                                                                    |
| Eşzamanlı agent'ları koddan yönetmek      | [Swarm](/tools/swarm)                          | [Kod Modu](/tr/tools/code-mode) ve [Alt agent'lar](/tr/tools/subagents)                                                                                       |
| Büyük bir OpenClaw araç kataloğunda arama yapmak         | [Araç Arama](/tr/tools/tool-search)              | [Araç Arama](/tr/tools/tool-search)                                                                                                                      |
| Birkaç aracı tek bir kompakt programda birleştirmek | [Kod Modu](/tr/tools/code-mode)                  | [Kod Modu](/tr/tools/code-mode)                                                                                                                          |

## Araçları, Skills'i veya plugin'leri seçin

<Steps>
  <Step title="Agent'ın işlem yapması gerektiğinde araç kullanın">
    Araç, agent'ın çağırabileceği `exec`, `browser`,
    `web_search`, `message` veya `image_generate` gibi türü belirlenmiş bir işlevdir. Agent'ın
    veri okuması, dosyaları değiştirmesi, mesaj göndermesi, bir sağlayıcıyı çağırması veya
    başka bir sistemi işletmesi gerektiğinde araçları kullanın. Görünür araçlar modele yapılandırılmış
    işlev tanımları olarak gönderilir.

    Model yalnızca etkin profil, izin/verme
    politikası, sağlayıcı kısıtlamaları, sandbox durumu, kanal izinleri ve
    plugin kullanılabilirliği süzgecinden geçen araçları görür.

  </Step>

  <Step title="Agent'ın talimatlara ihtiyacı olduğunda Skills kullanın">
    Skill, agent istemine yüklenen bir `SKILL.md` talimat paketidir. Agent
    ihtiyaç duyduğu araçlara zaten sahipse ancak tekrarlanabilir bir iş akışına, inceleme ölçütlerine,
    komut dizisine veya çalışma kısıtlamasına ihtiyaç duyuyorsa
    Skill kullanın.

    Skills bir çalışma alanında, paylaşılan Skill dizininde, yönetilen OpenClaw
    Skill kökünde veya plugin paketinde bulunabilir.

    [Skills](/tr/tools/skills) | [Skill Atölyesi](/tr/tools/skill-workshop) | [Kendi kendine öğrenme](/tr/tools/self-learning) | [Skills oluşturma](/tr/tools/creating-skills) | [Skills yapılandırması](/tr/tools/skills-config)

  </Step>

  <Step title="OpenClaw yeni bir yeteneğe ihtiyaç duyduğunda plugin kullanın">
    Bir plugin; araçlar, Skills, kanallar, model sağlayıcıları, konuşma,
    gerçek zamanlı ses, medya oluşturma, web araması, web'den getirme, hook'lar ve diğer
    çalışma zamanı yeteneklerini ekleyebilir. Yetenek kod,
    kimlik bilgileri, yaşam döngüsü hook'ları, manifest meta verileri veya kurulabilir
    paketleme içeriyorsa plugin kullanın. Mevcut plugin'ler ClawHub, npm, git,
    yerel dizinler veya arşivlerden kurulabilir.

    [Plugin'leri kurma ve yapılandırma](/tr/tools/plugin) | [Plugin geliştirme](/tr/plugins/building-plugins) | [Plugin SDK](/tr/plugins/sdk-overview)

  </Step>
</Steps>

## Yerleşik araç kategorileri

Tablo, yüzeyi tanıyabilmeniz için temsili araçları listeler. Bu,
politika başvurusunun tamamı değildir. Tam gruplar, varsayılanlar ve izin/verme
anlamları için [Araçlar ve özel sağlayıcılar](/tr/gateway/config-tools) sayfasını kullanın.

| Kategori                | Agent'ın şunları yapması gerektiğinde kullanın...                                                               | Temsili araçlar                                                                                                | Sonraki okuma                                                                                                              |
| ----------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Çalışma zamanı                 | Komut çalıştırmak, süreçleri yönetmek veya sağlayıcı destekli Python analizi kullanmak                       | `exec`, `process`, `terminal`, `code_execution`                                                                     | [Exec](/tr/tools/exec), [Control UI terminali](/tr/web/control-ui#operator-terminal), [Kod yürütme](/tr/tools/code-execution) |
| Dosyalar                   | Çalışma alanı dosyalarını okumak ve değiştirmek                                                              | `read`, `write`, `edit`, `apply_patch`                                                                              | [Yama uygula](/tr/tools/apply-patch)                                                                                      |
| İnsan girdisi             | Kullanıcının sorumluluğundaki yapılandırılmış bir karar için duraklamak                                            | `ask_user`                                                                                                          | [Kullanıcıya sor](/tools/ask-user)                                                                                            |
| Web                     | Web'de veya X gönderilerinde arama yapmak ya da okunabilir sayfa içeriğini getirmek                               | `web_search`, `x_search`, `web_fetch`                                                                               | [Web araçları](/tr/tools/web), [Web'den getirme](/tr/tools/web-fetch)                                                                 |
| Tarayıcı                 | Bir tarayıcı oturumunu işletmek                                                                    | `browser`                                                                                                           | [Tarayıcı](/tr/tools/browser)                                                                                              |
| Operatör kullanıcı arayüzü             | Bağlı Control UI bölmelerini, panellerini ve gezinmeyi düzenlemek                                   | `screen`                                                                                                            | [Ekran](/tools/screen)                                                                                                |
| Mesajlaşma ve kanallar  | Yanıt veya kanal eylemleri göndermek                                                              | `message`                                                                                                           | [Agent gönderimi](/tr/tools/agent-send)                                                                                        |
| Oturumlar ve agent'lar     | Oturumları incelemek, iş devretmek, toplayıcıları yönetmek, başka bir çalışmayı yönlendirmek veya durum bildirmek | `sessions_*`, `agents_wait`, `subagents`, `agents_list`, `session_status`, `get_goal`, `create_goal`, `update_goal` | [Hedef](/tr/tools/goal), [Swarm](/tools/swarm), [Alt agent'lar](/tr/tools/subagents), [Oturum aracı](/tr/concepts/session-tool)     |
| Otomasyon              | Çalışmayı zamanlamak veya arka plan olaylarına yanıt vermek                                                | `cron`, `heartbeat_respond`                                                                                         | [Otomasyon](/tr/automation)                                                                                              |
| Gateway ve Node'lar       | Gateway durumunu veya eşleştirilmiş hedef cihazları incelemek                                               | `gateway`, `nodes`                                                                                                  | [Gateway yapılandırması](/tr/gateway/configuration), [Node'lar](/tr/nodes)                                                       |
| Medya                   | Medyayı analiz etmek, oluşturmak veya seslendirmek                                                            | `image`, `image_generate`, `music_generate`, `video_generate`, `tts`                                                | [Medyaya genel bakış](/tr/tools/media-overview)                                                                                |
| Büyük OpenClaw katalogları | Her şemayı modele göndermeden uygun birçok aracı aramak, çağırmak ve birleştirmek      | `exec`, `wait`, `tool_search_code`, `tool_search`, `tool_describe`                                                  | [Kod Modu](/tr/tools/code-mode), [Araç Arama](/tr/tools/tool-search)                                                       |

<Note>
Kod Modu ve Araç Arama, deneysel OpenClaw agent yüzeyleridir. Codex
harness çalışmaları, `tools.codeMode` veya `tools.toolSearch` yerine Codex'e özgü kod modunu,
yerel araç aramasını, ertelenmiş dinamik araçları ve iç içe araç çağrılarını kullanır.
</Note>

## Plugin tarafından sağlanan araçlar

Plugin'ler ek araçlar kaydedebilir. Plugin yazarları araçları
`api.registerTool(...)` ve manifestin `contracts.tools` alanı üzerinden bağlar; sözleşme ayrıntıları için
[Plugin SDK](/tr/plugins/sdk-overview) ve [Plugin manifesti](/tr/plugins/manifest)
sayfalarını kullanın.

Plugin tarafından yaygın olarak sağlanan araçlar şunlardır:

- Dosya ve Markdown farklarını işlemek için [Farklar](/tr/tools/diffs)
- Desteklenen sohbet istemcilerinde bağımsız satır içi SVG ve HTML için [Widget göster](/tr/tools/show-widget)
- Bağlı bir Control UI'ı düzenlemek için [Ekran](/tools/screen)
- Yalnızca JSON içeren iş akışı adımları için [LLM Görevi](/tr/tools/llm-task)
- Sürdürülebilir onaylara sahip türü belirlenmiş iş akışları için [Lobster](/tr/tools/lobster)
- Gürültülü `exec` ve `bash` aracı
  çıktısını sıkıştırmak için [Tokenjuice](/tr/tools/tokenjuice)
- Her şemayı isteme koymadan büyük araç
  kataloglarını keşfetmek ve çağırmak için [Araç Arama](/tr/tools/tool-search)
- Node Canvas denetimi ve A2UI
  işleme için [Canvas](/tr/plugins/reference/canvas)

## Erişimi ve onayları yapılandırma

Araç politikası, model çağrısından önce uygulanır. Politika bir aracı kaldırırsa
model, o tur için aracın şemasını almaz. Bir çalıştırma; genel yapılandırma,
ajan başına yapılandırma, kanal politikası, sağlayıcı kısıtlamaları, sandbox
kuralları, kanal/çalışma zamanı politikası veya plugin kullanılabilirliği
nedeniyle araçları kaybedebilir.

- [Araçlar ve özel sağlayıcılar](/tr/gateway/config-tools); araç profillerini,
  izin/verme listelerini, sağlayıcıya özgü kısıtlamaları, döngü algılamayı ve
  sağlayıcı destekli araç ayarlarını belgeler.
- [Exec onayları](/tr/tools/exec-approvals), ana makine komutu onay
  politikasını belgeler.
- [Yükseltilmiş exec](/tr/tools/elevated), sandbox dışında denetimli yürütmeyi
  belgeler.
- [Sandbox, araç politikası ve yükseltilmiş erişim karşılaştırması](/tr/gateway/sandbox-vs-tool-policy-vs-elevated),
  dosya ve süreç erişimini hangi katmanın denetlediğini açıklar.
- [Ajan başına sandbox ve araç kısıtlamaları](/tr/tools/multi-agent-sandbox-tools),
  devredilen çalıştırmalar için ajana özgü kısıtlamaları belgeler.

## Yetenekleri genişletme

OpenClaw'un yapmasını istediğiniz işe göre genişletme yolunu seçin:

- Mevcut bir plugini [Pluginler](/tr/tools/plugin) ile kurun veya yönetin.
- [Plugin oluşturma](/tr/plugins/building-plugins) ile yeni bir entegrasyon,
  sağlayıcı, kanal, araç veya kanca oluşturun.
- [Skills](/tr/tools/skills) ve [Skills oluşturma](/tr/tools/creating-skills) ile
  yeniden kullanılabilir ajan talimatları ekleyin veya ayarlayın.
- Uygulama sözleşmelerine ihtiyaç duyduğunuzda
  [Plugin SDK](/tr/plugins/sdk-overview) ve [Plugin manifesti](/tr/plugins/manifest)
  kullanın.

## Eksik araçlarda sorun giderme

Model bir aracı göremiyor veya çağıramıyorsa mevcut tur için geçerli
politikayla başlayın:

1. [Araçlar ve özel sağlayıcılar](/tr/gateway/config-tools) bölümünde etkin profili,
   `tools.allow` ve `tools.deny` öğelerini denetleyin.
2. [Araçlar ve özel sağlayıcılar](/tr/gateway/config-tools) bölümündeki
   sağlayıcıya özgü kısıtlamaları denetleyin ve seçilen
   [model sağlayıcısının](/tr/concepts/model-providers) araç biçimini
   desteklediğini doğrulayın.
3. [Sandbox, araç politikası ve yükseltilmiş erişim karşılaştırması](/tr/gateway/sandbox-vs-tool-policy-vs-elevated)
   ve [Yükseltilmiş exec](/tr/tools/elevated) ile kanal izinlerini, sandbox
   durumunu ve yükseltilmiş erişimi denetleyin.
4. [Pluginler](/tr/tools/plugin) bölümünde sahibi olan pluginin kurulu ve etkin
   olup olmadığını denetleyin.
5. Devredilen çalıştırmalar için [Ajan başına sandbox ve araç kısıtlamaları](/tr/tools/multi-agent-sandbox-tools)
   bölümündeki ajan başına kısıtlamaları denetleyin.
6. Büyük OpenClaw katalogları için çalıştırmanın doğrudan araç
   sunumunu, [Kod Modu](/tr/tools/code-mode) veya [Araç Arama](/tr/tools/tool-search)
   kullanıp kullanmadığını doğrulayın.

## İlgili

- Cron, görevler, Heartbeat, kancalar, kalıcı talimatlar ve Task Flow
  için [Otomasyon](/tr/automation)
- Ajan modeli, oturumlar, bellek ve çok ajanlı koordinasyon
  için [Ajanlar](/tr/concepts/agent)
- Standart araç politikası başvurusu için
  [Araçlar ve özel sağlayıcılar](/tr/gateway/config-tools)
- Plugin kurulumu ve yönetimi için [Pluginler](/tr/tools/plugin)
- Plugin yazarı başvurusu için [Plugin SDK](/tr/plugins/sdk-overview)
- Skill yükleme sırası, geçit oluşturma ve yapılandırma için [Skills](/tr/tools/skills)
- Oluşturulan ve incelenen Skill'ların hazırlanması için
  [Skill Atölyesi](/tr/tools/skill-workshop)
- Kompakt OpenClaw araç kataloğu keşfi için
  [Araç Arama](/tr/tools/tool-search)
- Gizli bir OpenClaw araç kataloğu üzerinde kompakt JavaScript veya TypeScript
  iş akışları için [Kod Modu](/tr/tools/code-mode)
- Kod Modu'ndan yapılandırılmış dağıtım ve toplama için [Swarm](/tools/swarm)
