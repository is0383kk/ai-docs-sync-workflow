---
read_when:
    - Codex modundaki OpenClaw ajanlarının yerel Codex pluginlerini kullanmasını istiyorsunuz
    - Kaynak koddan yüklenmiş, OpenAI tarafından seçilmiş Codex pluginlerini taşıyorsunuz
    - Mevcut bir çalışma alanı dizini Codex pluginini yapılandırıyorsunuz
    - codexPlugins, uygulama envanteri, yıkıcı işlemler veya Plugin uygulaması tanılamalarıyla ilgili sorunları gideriyorsunuz
summary: Codex modundaki OpenClaw aracıları için yerel Codex pluginlerini yapılandırın
title: Yerel Codex pluginleri
x-i18n:
    generated_at: "2026-07-26T23:29:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0b1cfa39838d4dbd1f33a1e5b7f52faec4b033f9fa98ef5c029003177c2e27e5
    source_path: plugins/codex-native-plugins.md
    workflow: 16
---

Yerel Codex plugin desteği, Codex modundaki bir OpenClaw aracısının OpenClaw dönüşünü işleyen aynı Codex iş parçacığı içinde Codex app-server'ın kendi uygulama ve plugin yeteneklerini kullanmasına olanak tanır. Plugin çağrıları yerel Codex transkriptinde kalır; uygulama destekli MCP yürütmesinin sahibi Codex app-server'dır. OpenClaw, Codex pluginlerini sentetik `codex_plugin_*` OpenClaw dinamik araçlarına dönüştürmez.

Bu sayfayı temel [Codex çalışma düzeneği](/tr/plugins/codex-harness) çalıştıktan sonra kullanın.

## Gereksinimler

- Aracı çalışma zamanı, yerel Codex çalışma düzeneği olmalıdır.
- `plugins.entries.codex.enabled`, `true` olmalıdır.
- `plugins.entries.codex.config.codexPlugins.enabled`, `true` olmalıdır.
- Hedef Codex app-server, beklenen pazar yeri, plugin ve uygulama envanterini görebilmelidir.
- Geçiş yalnızca kaynak Codex ana dizininde kaynaktan yüklenmiş olduğu gözlemlenen `openai-curated` pluginlerini destekler.
- Elle yapılandırılan `workspace-directory` pluginleri, `plugin/list` öğesi `marketplaceKinds` kabul eden ve yolsuz çalışma alanı özetleri `remotePluginId` içeren bir Codex app-server gerektirir. Plugin önceden yüklenmiş ve etkinleştirilmiş olmalı, ayrıca sahip olduğu uygulamalara `app/list` içinde erişilebilmelidir.

`codexPlugins`; OpenClaw sağlayıcısı çalıştırmalarını, ACP konuşma bağlamalarını veya diğer çalışma düzeneklerini etkilemez; çünkü bu yollar yerel `apps` yapılandırmasına sahip Codex app-server iş parçacıkları oluşturmaz.

OpenAI tarafındaki Codex hesabı, uygulama kullanılabilirliği ve çalışma alanı uygulama/plugin denetimleri, oturum açılmış Codex hesabından gelir. OpenAI hesabı ve yönetici modeli için [ChatGPT planınızla Codex'i kullanma](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan) sayfasına bakın.

## Hızlı başlangıç

Kaynak Codex ana dizininden geçişi önizleyin:

```bash
openclaw migrate codex --dry-run
```

Geçişin kaynak `app/list` çağrısını yapması ve yerel etkinleştirmeyi planlamadan önce sahip olunan her uygulamanın mevcut, etkin ve erişilebilir olmasını zorunlu tutması için `--verify-plugin-apps` ekleyin:

```bash
openclaw migrate codex --dry-run --verify-plugin-apps
```

Plan doğru görünüyorsa geçişi uygulayın:

```bash
openclaw migrate apply codex --yes
```

Geçiş, uygun pluginler için açık `codexPlugins` girdileri yazar ve seçilen pluginler için Codex app-server `plugin/install` çağrısını yapar. Geçirilmiş bir yapılandırma şu şekilde görünür:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
              },
            },
          },
        },
      },
    },
  },
}
```

Geçiş, `openai-curated` ile sınırlı kalır. Mevcut bir `workspace-directory` plugini kullanmak için onu `plugin/list` tarafından döndürülen, pazar yeri niteleyicisini içeren tam `summary.id` değeriyle elle ekleyin. Örneğin Codex `example-plugin@workspace-directory` döndürürse görünen adı yerine bu tam değeri yapılandırın:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            plugins: {
              "example-plugin": {
                enabled: true,
                marketplaceName: "workspace-directory",
                pluginName: "example-plugin@workspace-directory",
              },
            },
          },
        },
      },
    },
  },
}
```

OpenClaw, bir `workspace-directory` plugini için `plugin/install` çağrısı yapmaz veya kimlik doğrulamayı başlatmaz. OpenClaw ilkesini eklemeden veya etkinleştirmeden önce plugini Codex içinde yükleyin, etkinleştirin ve kimliğini doğrulayın. Yanıt tam pazar yerini, plugin kimliğini, ayrıntı kimliğini veya uygulamanın hazır olduğuna dair kanıtı içermediğinde OpenClaw uygulamaları gizli tutar. Codex açık çalışma alanı `plugin/list` isteğini reddederse OpenClaw, etkinleştirilmiş her çalışma alanı plugini için `marketplace_missing` bildirir ve bağımsız olarak keşfedilen seçkili pluginleri kullanılabilir tutar.

Bir `codexPlugins` değişikliğinden sonra yeni Codex konuşmaları güncellenmiş uygulama kümesini otomatik olarak alır. Geçerli konuşmayı yenilemek için `/new` veya `/reset` çalıştırın. Plugin etkinleştirme/devre dışı bırakma değişiklikleri için gateway'in yeniden başlatılması gerekmez.

## Pluginleri sohbetten yönetin

`/codex plugins`, Codex çalışma düzeneğini kullandığınız aynı sohbetten yapılandırılmış yerel Codex pluginlerini inceler veya değiştirir:

```text
/codex plugins
/codex plugins list
/codex plugins disable google-calendar
/codex plugins enable google-calendar
```

`/codex plugins`, `/codex plugins list` için bir diğer addır. Liste, `plugins.entries.codex.config.codexPlugins.plugins` içindeki her yapılandırılmış pluginin anahtarını, açık/kapalı durumunu, Codex plugin adını ve pazar yerini gösterir.

`enable`/`disable` yalnızca `~/.openclaw/openclaw.json` içine yazar; hiçbir zaman `~/.codex/config.toml` öğesini düzenlemez veya yeni Codex pluginleri yüklemez. Bunları yalnızca sahibi veya `operator.admin` kapsamına sahip bir gateway istemcisi çalıştırabilir.

Yapılandırılmış bir plugini etkinleştirmek, genel `codexPlugins.enabled` anahtarını da açar. Geçiş `auth_required` döndürdüğü için seçkili bir plugin devre dışı olarak yazıldıysa OpenClaw içinde etkinleştirmeden önce uygulamayı Codex'te yeniden yetkilendirin. Bir `workspace-directory` girdisi için burada etkinleştirmek yalnızca OpenClaw ilkesini değiştirir; plugin ve uygulama Codex'te önceden etkin olmalıdır.

## Yerel plugin kurulumu nasıl çalışır?

Entegrasyon üç durumu izler:

| Durum       | Anlamı                                                                                                                             |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Yüklü       | Codex, hedef app-server çalışma zamanında plugin paketine sahiptir.                                                                |
| Etkin       | Codex, pluginin etkin olduğunu bildirir ve OpenClaw yapılandırması Codex çalışma düzeneği dönüşlerinde buna izin verir.             |
| Erişilebilir | Codex app-server, pluginin uygulama girdilerinin etkin hesap için kullanılabildiğini ve yapılandırılmış plugin kimliğiyle eşleştiğini doğrular. |

`openai-curated` pluginleri için geçiş, kalıcı yükleme/uygunluk adımıdır:

- Planlama sırasında OpenClaw, kaynak Codex `plugin/read` ayrıntılarını okur ve kaynak Codex app-server hesabının bir ChatGPT abonelik hesabı olduğunu denetler. ChatGPT olmayan veya eksik bir hesap yanıtı, uygulama destekli pluginleri `codex_subscription_required` ile atlar.
- Geçiş varsayılan olarak kaynak `app/list` çağrısını atlar: hesap geçidini geçen uygulama destekli kaynak pluginleri, kaynak uygulama erişilebilirliği doğrulanmadan planlanır ve hesap araması aktarım hatalarında `codex_account_unavailable` ile atlanır.
- `--verify-plugin-apps` kullanıldığında geçiş yeni bir kaynak `app/list` anlık görüntüsü alır ve yerel etkinleştirmeyi planlamadan önce sahip olunan her uygulamanın mevcut, etkin ve erişilebilir olmasını zorunlu tutar. Bu durumda hesap araması aktarım hataları doğrudan atlanmak yerine kaynak uygulama envanteri geçidine aktarılır.

`workspace-directory` pluginleri için kurulum OpenClaw dışında gerçekleşir. OpenClaw bu pazar yerini yalnızca en az bir etkin çalışma alanı girdisi yapılandırıldığında sorgular, her plugini tam `summary.id` değerine göre çözümler ve mevcut `plugin/read` sahiplik ve `app/list` hazırlık denetimlerini yeniden kullanır. Yüklenmemiş, devre dışı, erişilemez veya kimliği doğrulanmamış bir plugin hiçbir uygulamayı göstermez; OpenClaw yükleme veya kimlik doğrulama girişiminde bulunmaz.

Çalışma zamanı uygulama envanteri, hem geçirilmiş seçkili pluginler hem de elle yapılandırılmış çalışma alanı pluginleri için hedef oturum erişilebilirlik denetimidir. Codex çalışma düzeneği oturum kurulumu, etkin ve erişilebilir plugin uygulamalarından kısıtlayıcı bir iş parçacığı uygulama yapılandırması hesaplar; bu yapılandırma her dönüşte yeniden hesaplanmadığından `/codex plugins enable`/`disable` yalnızca yeni Codex konuşmalarını etkiler. Geçerli konuşmada değişikliği almak için `/new` veya `/reset` kullanın.

## V1 destek sınırı

- Yalnızca kaynak Codex app-server envanterinde önceden yüklü olan `openai-curated` pluginleri geçiş için uygundur.
- Çalışma zamanı ayrıca `plugin/list` öğesi `marketplaceKinds` uygulayan ve yolsuz çalışma alanı özetleri için `remotePluginId` döndüren app-server derlemelerinde açık `workspace-directory` girdilerini destekler. Bu girdiler, pazar yeri niteleyicisini içeren tam `summary.id` değerlerini kullanmalı ve önceden yüklenmiş, etkinleştirilmiş ve uygulama açısından erişilebilir olmalıdır. Reddedilen bir çalışma alanı listeleme isteği, mevcut plugin başına `marketplace_missing` tanılamasını üretir; eksik pazar yeri, plugin, ayrıntı veya uygulama kanıtı hiçbir çalışma alanı uygulamasını göstermez. Varsayılan liste isteğinden gelen seçkili envanter kullanılabilir durumda kalır.
- Uygulama destekli kaynak pluginleri, geçiş sırasındaki abonelik geçidini geçmelidir. `--verify-plugin-apps`, kaynak uygulama envanteri geçidini ekler. Abonelik geçidine takılan hesaplar ve doğrulama modunda erişilemeyen/devre dışı/eksik kaynak uygulamalar ya da uygulama envanteri yenileme hataları, etkin yapılandırma girdileri yerine atlanmış elle işlenecek öğeler olarak bildirilir. Okunamayan plugin ayrıntıları, uygulama envanteri geçidinden önce atlanır.
- Geçiş, açık plugin kimliklerini (`marketplaceName` ve `pluginName`) yazar; yerel `marketplacePath` önbellek yollarını yazmaz.
- `codexPlugins.enabled` tek genel etkinleştirme anahtarıdır; rastgele yükleme yetkisi veren bir `plugins["*"]` joker karakteri veya yapılandırma anahtarı yoktur.
- Seçkili olmayan pazar yerleri, önbelleğe alınmış plugin paketleri, kancalar ve Codex yapılandırma dosyaları otomatik olarak etkinleştirilmez; elle incelenmek üzere geçiş raporunda korunur. Çalışma zamanı, elle yapılandırılan `workspace-directory` girdilerini kabul eder; diğer pazar yerleri desteklenmez.

## Uygulama envanteri ve sahiplik

OpenClaw, Codex uygulama envanterini app-server `app/list` üzerinden okur, bellekte bir saat boyunca önbelleğe alır ve eski ya da eksik girdileri eşzamansız olarak yeniler. Önbellek işleme özeldir; CLI veya gateway yeniden başlatıldığında silinir ve OpenClaw sonraki `app/list` okumasından yeniden oluşturur.

Geçiş ve çalışma zamanı ayrı önbellek anahtarları kullanır:

- Kaynak geçiş doğrulaması, kaynak Codex ana dizinini ve başlatma seçeneklerini kullanır. Yalnızca `--verify-plugin-apps` ile çalışır ve bu planlama çalıştırması için yeni bir kaynak `app/list` taramasını zorunlu kılar.
- Hedef çalışma zamanı kurulumu, iş parçacığı uygulama yapılandırmasını oluştururken hedef aracının Codex app-server kimliğini kullanır. Seçkili plugin etkinleştirmesi bu hedef önbellek anahtarını geçersiz kılar, ardından `plugin/install` sonrasında zorla yeniler. `workspace-directory` kurulumu bu etkinleştirme yolunu hiçbir zaman çalıştırmaz.

Bir plugin uygulaması yalnızca OpenClaw onu kararlı sahiplik üzerinden yapılandırılmış pluginle eşleyebildiğinde gösterilir: plugin ayrıntısındaki tam uygulama kimliği, bilinen bir MCP sunucu adı veya benzersiz kararlı meta veriler. Yalnızca görünen ada dayalı veya belirsiz sahiplik, sonraki envanter yenilemesi sahipliği kanıtlayana kadar hariç tutulur.

## Bağlı hesap uygulamaları

Sahibi tarafından işletilen aracılar, eşleşen bir plugin paketi gerektirmeden Codex hesaplarına önceden bağlı tüm uygulamaları kullanmayı seçebilir:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
          },
        },
      },
    },
  },
}
```

`allow_all_plugins: true`, yeni bir yerel Codex iş parçacığı oluşturulduğunda eksiksiz bir `app/list` anlık görüntüsü alır ve yalnızca o hesap için erişilebilir olarak işaretlenen uygulamalara izin verir. Uygulamaları genel olarak yüklemez, kimliklerini doğrulamaz veya etkinleştirmez. Mevcut iş parçacıkları kalıcı uygulama kümelerini korur; yeni bağlanan veya erişimi kaldırılan uygulamaları almak için `/new`, `/reset` kullanın ya da gateway'i yeniden başlatın.

Hesap uygulamaları, `true`, `false`, `"auto"` veya `"ask"` değerlerini kabul eden genel `codexPlugins.allow_destructive_actions` değerini devralır.
Uygulama kimliklerinin çakıştığı durumlarda açıkça belirtilen Plugin başına politika,
genel politikayı geçersiz kılar. Envanter hatalarında, sınırsız bir varsayılana
geri dönmek yerine erişim kapalı tutulur.

## İş parçacığı uygulama yapılandırması

OpenClaw, Codex iş parçacığı için kısıtlayıcı bir `config.apps` yaması ekler:
`_default` devre dışı bırakılır ve yalnızca etkinleştirilmiş, yapılandırılmış Plugin'lerin sahip olduğu uygulamalar veya
`allow_all_plugins` tarafından kabul edilen erişilebilir hesap uygulamaları etkinleştirilir.

Her uygulamadaki `destructive_enabled`, geçerli genel veya
Plugin başına `allow_destructive_actions` politikasından gelir; `true`, `"auto"` ve `"ask"`
değerlerinin tümü `destructive_enabled: true` değerini ayarlar, `false` ise bunu `false` olarak ayarlar. Codex,
kendi yerel uygulama aracı ek açıklamalarındaki yıkıcı araç meta verilerini uygulamaya devam eder.
`_default`, `open_world_enabled: false` ile devre dışı bırakılır; etkinleştirilmiş Plugin uygulamaları
`open_world_enabled: true` değerini alır. OpenClaw, Plugin düzeyinde ayrı bir
açık dünya politikası ayarı sunmaz ve Plugin başına
yıkıcı araç adı engelleme listeleri tutmaz.

Araç onay modu, kabul edilen uygulamalar için varsayılan olarak otomatiktir; böylece yıkıcı olmayan
okuma araçları aynı iş parçacığında onay istemi olmadan çalışır. Yıkıcı araçlar,
her uygulamanın `destructive_enabled` politikası tarafından denetlenmeye devam eder.

## Yıkıcı eylem politikası

Yıkıcı Plugin bilgi istemlerine, yapılandırılmış Codex
Plugin'leri için varsayılan olarak izin verilir; güvenli olmayan şemalarda ve belirsiz sahiplikte ise erişim kapalı tutulur:

- Genel `allow_destructive_actions` varsayılan olarak `true` değerini kullanır.
- Plugin başına `allow_destructive_actions`, söz konusu Plugin için genel politikayı
  geçersiz kılar.
- `false`: OpenClaw belirlenimsel bir ret yanıtı döndürür.
- `true`: OpenClaw yalnızca bir boole onay alanı gibi, bir onay
  yanıtına eşleyebildiği güvenli şemaları otomatik olarak kabul eder.
- `"auto"`: OpenClaw, yıkıcı Plugin eylemlerini Codex'e sunar ve ardından
  sahipliği kanıtlanmış MCP onay bilgi istemlerini, Codex onay yanıtını
  döndürmeden önce OpenClaw Plugin onaylarına dönüştürür.
- `"ask"`: OpenClaw, `"auto"` ile aynı Codex yazma/yıkıcı erişim
  denetimini kullanır, iş parçacığı başlamadan önce uygulama için kalıcı Codex araç başına onay
  geçersiz kılmalarını temizler ve kalıcı onayların sonraki yazma eylemi istemlerini
  bastıramaması için yalnızca tek seferlik onay veya ret sunar.
  `"ask"` kullanan kabul edilmiş her uygulama için OpenClaw, o uygulamada Codex'in insan onayları
  inceleyicisini seçer; böylece Codex, onay bilgi istemlerini
  OpenClaw'a gönderir. Diğer uygulamalar ve uygulama dışı iş parçacığı onayları, yapılandırılmış
  inceleyici ve politikalarını korur.
- Plugin kimliğinin eksik olması, sahipliğin belirsiz olması, dönüş kimliğinin eksik veya
  eşleşmiyor olması ya da güvenli olmayan bir bilgi istemi şeması, istem göstermek yerine reddedilir.

## Sorun giderme

| Kod                                               | Anlamı                                                                                                                              | Düzeltme                                                                                                               |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `auth_required`                                   | Geçiş Plugin'i yükledi ancak uygulamalarından biri hâlâ kimlik doğrulaması gerektiriyor. Yeniden yetkilendirene kadar girdi devre dışı olarak yazılır. | Uygulamayı Codex'te yeniden yetkilendirin, ardından Plugin'i OpenClaw'da etkinleştirin.                                 |
| `app_inaccessible`, `app_disabled`, `app_missing` | `--verify-plugin-apps` kullanıldığında, kaynak Codex uygulama envanteri sahip olunan tüm uygulamaları mevcut, etkin ve erişilebilir olarak göstermedi. | Uygulamayı Codex'te yeniden yetkilendirin veya etkinleştirin, ardından geçişi `--verify-plugin-apps` ile yeniden çalıştırın. |
| `app_inventory_unavailable`                       | Katı kaynak uygulama doğrulaması istendi ancak kaynak Codex uygulama envanteri yenilenemedi.                                          | Kaynak Codex uygulama sunucusu erişimini düzeltin veya daha hızlı, hesap denetimli planı kabul etmek için `--verify-plugin-apps` olmadan yeniden deneyin. |
| `codex_subscription_required`                     | Kaynak Codex uygulama sunucusu hesabı bir ChatGPT abonelik hesabı değildi.                                                            | Codex uygulamasında abonelik kimlik doğrulamasıyla oturum açın, ardından geçişi yeniden çalıştırın.                    |
| `codex_account_unavailable`                       | Kaynak Codex uygulama sunucusu hesabı okunamadı.                                                                                      | Kaynak Codex uygulama sunucusu kimlik doğrulamasını düzeltin veya uygunluğa kaynak uygulama envanterinin karar vermesi için `--verify-plugin-apps` ile yeniden çalıştırın. |
| `marketplace_missing`, `plugin_missing`           | Pazar yeri veya tam Plugin kullanılamıyor; açık çalışma alanı katalog isteği reddedilmiş olabilir; çalışma alanı uygulamalarında erişim kapalı tutulur. | Aşağıda açıklanan uyumlu uygulama sunucusu sözleşmesini ve tam kimliği doğrulayın.                                      |
| `plugin_detail_unavailable`                       | OpenClaw, Plugin sahipliği ayrıntılarını okuyamadı.                                                                                   | Hedef uygulama sunucusunun `plugin/list` ve `plugin/read` yanıtlarını inceleyin.                             |
| `plugin_disabled`                                 | Codex, Plugin'in yüklü ancak devre dışı olduğunu bildiriyor.                                                                          | Seçkili etkinleştirme bunu düzeltebilir; yeniden denemeden önce Codex'te bir çalışma alanı Plugin'ini etkinleştirin.    |
| `plugin_activation_failed`                        | Plugin etkinleştirmesi tamamlanmadı.                                                                                                  | Pazar yeri, kimlik doğrulama, yenileme veya çalışma alanı hazır olma hatalarını ayırt etmek için ekli tanılamayı kullanın. |
| `app_inventory_missing`, `app_inventory_stale`    | Uygulamanın hazır olma durumu boş veya eski bir önbellekten geldi.                                                                    | OpenClaw otomatik olarak eşzamansız yenileme zamanlar; sahiplik ve hazır olma durumu bilinene kadar Plugin uygulamaları hariç tutulur. |
| `app_ownership_ambiguous`                         | Uygulama envanteri yalnızca görünen ada göre eşleşti.                                                                                 | Daha sonraki bir yenileme sahipliği kanıtlayana kadar uygulama Codex iş parçacığından gizli kalır.                     |

**Çalışma alanı Plugin'i yüklü ancak görünmüyor:** çalışma alanı
`plugin/list` sonucunun, yapılandırılan tam kimliği yüklü ve etkin olarak bildirdiğini doğrulayın;
ardından `app/list` sonucunun, aynı Codex hesabı için sahip olunan her uygulamayı erişilebilir
olarak bildirdiğini doğrulayın. Hesap envanteri şu anda uygulamayı devre dışı olarak bildirse bile OpenClaw,
erişilebilir bir uygulamayı iş parçacığı için etkinleştirebilir. Gateway uygulama
envanterini önbelleğe aldıktan sonra bu durumu değiştirdiyseniz, bir saatlik önbellek yenilemesini bekleyin veya Gateway'i yeniden başlatın; ardından
`/new` ya da `/reset` kullanın. OpenClaw, çalışma alanı Plugin'lerini onarmaz veya bunların kimliğini doğrulamaz.
Açık çalışma alanı liste isteği reddedilirse, etkinleştirilmiş her çalışma alanı
girdisi `marketplace_missing` bildirir; ilgisiz seçkili girdiler varsayılan liste
yanıtından ilerlemeye devam eder.

`plugin_detail_unavailable` için, yolu olmayan bir çalışma alanı özeti
`remotePluginId` içermelidir; bu seçici veya sonraki
`plugin/read` sonucu kullanılamadığında OpenClaw, sahip olunan uygulamaları gizli tutar.
`plugin_activation_failed` için seçkili Plugin'ler bir pazar yeri, kimlik doğrulama veya
yükleme sonrası yenileme hatası bildirebilir. Bir çalışma alanı Plugin'i henüz
etkin değilse bu kodu bildirir; Plugin'i OpenClaw dışında yükleyin, etkinleştirin ve kimliğini doğrulayın.

**Yapılandırma değişti ancak aracı Plugin'i göremiyor:** yapılandırılan durumu doğrulamak için `/codex plugins
list` çalıştırın, ardından `/new` veya `/reset` çalıştırın. Mevcut
Codex iş parçacığı bağlamaları, OpenClaw yeni bir çalıştırma sistemi oturumu
oluşturana veya eski bir bağlamayı değiştirene kadar başlangıçtaki uygulama yapılandırmasını korur.

**Yıkıcı eylem reddediliyor:** genel ve Plugin başına
`allow_destructive_actions` değerlerini kontrol edin. `true`, `"auto"` veya `"ask"`
kullanılsa bile güvenli olmayan bilgi istemi şemalarında ve belirsiz Plugin kimliğinde erişim kapalı tutulur.

## İlgili

- [Codex çalıştırma sistemi](/tr/plugins/codex-harness)
- [Codex çalıştırma sistemi başvurusu](/tr/plugins/codex-harness-reference)
- [Codex çalıştırma sistemi çalışma zamanı](/tr/plugins/codex-harness-runtime)
- [Yapılandırma başvurusu](/tr/gateway/configuration-reference#codex-harness-plugin-config)
- [Geçiş CLI'si](/tr/cli/migrate)
