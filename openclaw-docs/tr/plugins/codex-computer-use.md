---
read_when:
    - Codex modundaki OpenClaw ajanlarının Codex Computer Use'ı kullanmasını istiyorsunuz
    - Codex Computer Use, PeekabooBridge ve doğrudan cua-driver MCP arasında seçim yapıyorsunuz
    - Paketle birlikte gelen Codex plugini için computerUse'ı yapılandırıyorsunuz
    - /codex bilgisayar kullanımı durumuyla veya kurulumuyla ilgili sorunları gideriyorsunuz
summary: Codex modundaki OpenClaw ajanları için Codex Computer Use'ı kurma
title: Codex Bilgisayar Kullanımı
x-i18n:
    generated_at: "2026-07-27T00:05:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b11d00c74bc2990a4e33b6ffe23209ed76a1e10180ce5950dbb5073ea57ad05
    source_path: plugins/codex-computer-use.md
    workflow: 16
---

Computer Use, yerel masaüstü denetimi için Codex'e özgü bir MCP Plugin'idir. OpenClaw
masaüstü uygulamasını paketine dahil etmez, masaüstü eylemlerini kendisi yürütmez veya
Codex izinlarını atlamaz. Birlikte gelen `codex` Plugin'i yalnızca Codex app-server'ı hazırlar:
Codex Plugin desteğini etkinleştirir, yapılandırılmış Computer Use
Plugin'ini bulur veya yükler, `computer-use` MCP sunucusunun kullanılabilir olduğunu denetler ve ardından
Codex modundaki turlar sırasında yerel MCP araç çağrılarının denetimini
Codex'e bırakır.

Bu sayfayı OpenClaw zaten yerel Codex harness'ını kullanıyorsa kullanın. Çalışma zamanı
kurulumunun kendisi için [Codex harness](/tr/plugins/codex-harness) sayfasına bakın.

Bu, OpenClaw'ın yerleşik [Node destekli bilgisayar aracından](/tr/nodes/computer-use) farklıdır. Aynı ajan sözleşmesinin, ajan ister Gateway'de ister başka bir Node'da çalışsın, eşleştirilmiş bir Mac'i denetlemesi gerekiyorsa yerleşik aracı kullanın. Codex app-server'ın yerel MCP kurulumunu, izinleri ve yerel araç çağrılarını yönetmesi gerektiğinde Codex Computer Use'ı kullanın.

## OpenClaw.app ve Peekaboo

OpenClaw.app'in Peekaboo entegrasyonu, Codex Computer Use'dan ayrıdır.
macOS uygulaması, `peekaboo` CLI'ın Peekaboo'nun kendi
otomasyon araçları için uygulamanın yerel Erişilebilirlik ve Ekran Kaydı izinlerini yeniden kullanabilmesi amacıyla bir PeekabooBridge yuvası barındırabilir.
Bu köprü Codex Computer Use'ı yüklemez veya ona proxy görevi görmez ve
Codex Computer Use, PeekabooBridge yuvası üzerinden çağrı yapmaz.

OpenClaw.app'in Peekaboo CLI otomasyonu için
izinleri dikkate alan bir ana makine olmasını istediğinizde [Peekaboo köprüsünü](/tr/platforms/mac/peekaboo) kullanın. Codex
modundaki bir OpenClaw ajanının, tur başlamadan önce Codex'in yerel `computer-use` MCP Plugin'ini
kullanabilmesi gerektiğinde bu sayfayı kullanın.

## iOS uygulaması

iOS uygulaması, Codex Computer Use'dan ayrıdır. Codex
`computer-use` MCP sunucusunu yüklemez veya ona proxy görevi görmez ve bir masaüstü denetimi arka ucu değildir.
Bunun yerine iOS uygulaması bir OpenClaw Node'u olarak bağlanır ve `canvas.*`, `camera.*`, `screen.*`,
`location.*` ve `talk.*` gibi Node komutları üzerinden mobil
yetenekler sunar.

Bir ajanın Gateway üzerinden bir iPhone Node'unu
yönetmesini istediğinizde [iOS](/tr/platforms/ios) sayfasını kullanın. Codex modundaki bir ajanın
Codex'in yerel Computer Use Plugin'i üzerinden yerel macOS masaüstünü denetlemesi gerektiğinde bu sayfayı kullanın.

## Doğrudan cua-driver MCP

Masaüstü denetimi sunmanın tek yolu Codex Computer Use değildir.
OpenClaw tarafından yönetilen çalışma zamanlarının TryCua sürücüsünü doğrudan çağırmasını istiyorsanız
Codex'e özgü pazar yeri akışı yerine OpenClaw'ın MCP kayıt defteri üzerinden yukarı akış
`cua-driver mcp` sunucusunu kullanın.

`cua-driver` yüklendikten sonra ya OpenClaw komutunu vermesini isteyin:

```bash
cua-driver mcp-config --client openclaw
```

ya da stdio sunucusunu doğrudan kaydedin:

```bash
openclaw mcp set cua-driver '{"command":"cua-driver","args":["mcp"]}'
```

Bu yol, sürücü şemaları ve yapılandırılmış MCP yanıtları dahil olmak üzere yukarı akış MCP araç yüzeyini
değiştirmeden korur. CUA sürücüsünün
normal bir OpenClaw MCP sunucusu olarak kullanılabilmesini istediğinizde bunu kullanın. Codex app-server'ın Codex modundaki turlar içinde
Plugin kurulumunu, MCP yeniden yüklemelerini ve
yerel araç çağrılarını yönetmesi gerektiğinde bu sayfadaki Codex Computer Use kurulumunu kullanın.

CUA sürücüsü; macOS, Windows (x64 ve ARM64) ve
Linux (x64 ve ARM64, önizleme katmanı) için ön sürüm derlemeleri sunar. Yine de uygulamasının
macOS'taki Erişilebilirlik ve Ekran Kaydı gibi istediği yerel işletim sistemi
izinlerini gerektirir. OpenClaw, `cua-driver`'ı yüklemez, bu izinleri vermez veya
yukarı akış sürücüsünün güvenlik modelini atlamaz.

## Hızlı kurulum

Codex modundaki turlarda bir iş parçacığı başlamadan önce
Computer Use'ın kullanılabilir olması gerekiyorsa `plugins.entries.codex.config.computerUse` değerini ayarlayın. `autoInstall: true`,
Computer Use'ı etkinleştirir ve OpenClaw'ın turdan önce onu yüklemesine veya yeniden etkinleştirmesine olanak tanır:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          computerUse: {
            autoInstall: true,
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

Bu yapılandırmayla OpenClaw, Codex modundaki her
turdan önce Codex app-server'ı denetler. Computer Use eksikse ancak Codex app-server yüklenebilir bir pazar yerini zaten keşfetmişse OpenClaw, Codex app-server'dan Plugin'i yüklemesini veya
yeniden etkinleştirmesini ve MCP sunucularını yeniden yüklemesini ister. macOS'ta yalıtılmış bir
Codex app-server'ı başlatmadan önce otomatik yükleme, yerel istemci eksik olduğunda seçilen masaüstü uygulama paketindeki resmî imzalı
Computer Use hizmet uygulamasını da ilgili Codex ana dizininin
`computer-use` dizinine kopyalar.
macOS'ta eşleşen bir
pazar yeri kayıtlı değilse ve standart bir masaüstü uygulama paketi mevcutsa OpenClaw,
`/Applications/Codex.app/Contents/Resources/plugins/openai-bundled`'ı eski bağımsız kurulumlar için
geri dönüş olarak koruyarak
`/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled` konumundaki paketlenmiş Codex pazar yerini de kaydetmeye çalışır.
Kurulum yine de MCP sunucusunu kullanılabilir hâle getiremezse tur, iş parçacığı başlamadan önce başarısız olur.
Katı hazırlık hataları harness ön kontrol hatalarıdır; bu nedenle model geri dönüşü,
her model adayı için aynı yerel hazırlık dizisini tekrarlamaz.

Computer Use yapılandırmasını değiştirdikten sonra mevcut bir Codex iş parçacığı zaten başladıysa testten önce etkilenen
sohbette `/new` veya `/reset` kullanın.

macOS'ta Computer Use için yönetilen başlatma, önce
`/Applications/ChatGPT.app/Contents/Resources/codex` konumundaki masaüstü uygulama ikilisini tercih eder, ardından eski
bağımsız kurulumlar için `/Applications/Codex.app/Contents/Resources/codex` konumuna
geri döner. Bu, kendi istemcisini başlatan tek seferlik Computer Use durum ve
yükleme komutları için de geçerlidir. Böylece masaüstü denetimi, yerel macOS izinlerinin sahibi olan
uygulama paketinin altında kalır. Masaüstü uygulaması yüklü değilse
OpenClaw, Plugin'in yanına yüklenmiş yönetilen Codex ikilisine geri döner.
Varsayılan yalıtılmış ajan ana dizinini kullanan sıradan yönetilen Codex turları, eski bir masaüstü uygulamasının güncel model
desteğini gölgelememesi için önce sabitlenmiş paketi tercih eder. Kullanıcı kapsamlı ana dizinler, yerel
Computer Use durumunu yükleyebildikleri için masaüstünü öncelemeye devam eder. Etkin Codex yapılandırması
Computer Use'ı etkinleştiren yalıtılmış bir ajan ana dizini de masaüstünü öncelemeye devam eder. Açık
`appServer.command` yapılandırması veya `OPENCLAW_CODEX_APP_SERVER_BIN` yine de
bu yönetilen seçimi geçersiz kılar.

OpenClaw, çalışan tek bir Gateway içinde yerel Codex yapılandırma okumalarını ve Computer Use kurulumunu
seri hâle getirir. Ayrı bir Codex işlemi veya başka bir Gateway bu korumanın
parçası değildir. Yerel Codex Plugin yapılandırmasını Gateway dışında değiştirdikten sonra yeni
seçime güvenmeden önce Gateway'i yeniden başlatın ve yeni bir sohbet başlatın.

## Komutlar

`codex` Plugin komut yüzeyinin kullanılabildiği herhangi bir sohbet yüzeyinden
`/codex computer-use` komutlarını kullanın. Bunlar `openclaw codex ...` CLI alt komutları değil,
OpenClaw sohbet/çalışma zamanı komutlarıdır:

```text
/codex computer-use status
/codex computer-use install
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
/codex computer-use install --marketplace <name>
```

`status` varsayılan eylemdir ve salt okunurdur: pazar yeri
kaynakları eklemez, Plugin yüklemez veya Codex Plugin desteğini etkinleştirmez. Hiçbir yapılandırma
Computer Use'ı etkinleştirmiyorsa `status`, tek seferlik bir yükleme
komutundan sonra bile devre dışı olduğunu bildirebilir.

`install`, Codex app-server Plugin desteğini etkinleştirir, isteğe bağlı olarak
yapılandırılmış bir pazar yeri kaynağı ekler, yapılandırılmış Plugin'i
Codex app-server üzerinden yükler veya yeniden etkinleştirir, MCP sunucularını yeniden yükler ve MCP
sunucusunun araçlar sunduğunu doğrular. Kurulum güvenilen ana makine kaynaklarını değiştirdiği için
yalnızca bir sahip veya `operator.admin` Gateway istemcisi `install` komutunu çalıştırabilir. Diğer
yetkili göndericiler, geçersiz kılmalarla birlikte de salt okunur `status` komutunu
kullanmaya devam edebilir.

Eski sürümler tek seferlik `--plugin`, `--server` ve `--mcp-server`
kimlik geçersiz kılmalarını kabul ediyordu. Bunun yerine `computerUse.pluginName` ve
`computerUse.mcpServerName` değerlerini kalıcı olarak yapılandırın. Eski bir kimlik bayrağı
kullanıldığında komut, kalıcılaştırılacak tam ayarı belirtir ve geçiş kılavuzunda
istenen eylemi ve desteklenen tüm pazar yeri bayraklarını tekrarlar.

## Pazar yeri seçenekleri

OpenClaw, Codex'in kendisinin sunduğu app-server API'sinin aynısını kullanır.
Pazar yeri alanları, Codex'in `computer-use` öğesini nerede bulacağını seçer.

| Alan                 | Kullanım durumu                                                  | Yükleme desteği                                          |
| -------------------- | --------------------------------------------------------------- | -------------------------------------------------------- |
| Pazar yeri alanı yok | Codex app-server'ın zaten bildiği pazar yerlerini kullanmasını istiyorsunuz. | Evet, app-server yerel bir pazar yeri döndürdüğünde.     |
| `marketplaceSource`  | App-server'ın ekleyebileceği bir Codex pazar yeri kaynağınız var. | Evet, açık `/codex computer-use install` için.         |
| `marketplacePath`    | Ana makinedeki yerel pazar yeri dosya yolunu zaten biliyorsunuz. | Evet, açık yükleme ve tur başlangıcında otomatik yükleme için. |
| `marketplaceName`    | Zaten kayıtlı bir pazar yerini ada göre seçmek istiyorsunuz.    | Yalnızca seçilen pazar yerinin yerel bir yolu olduğunda evet. |

Yeni Codex ana dizinlerinin resmî
pazar yerlerini hazırlaması kısa bir süre alabilir. Yükleme sırasında OpenClaw, `plugin/list` için
`marketplaceDiscoveryTimeoutMs` milisaniyeye kadar (varsayılan 60 saniye) yoklama yapar.

Bilinen birden fazla pazar yeri Computer Use içeriyorsa OpenClaw önce
`openai-bundled`, ardından `openai-curated`, sonra `local` değerini tercih eder. Bilinmeyen ve belirsiz
eşleşmeler güvenli biçimde başarısız olur ve `marketplaceName` veya
`marketplacePath` ayarlamanızı ister.

## Paketlenmiş macOS pazar yeri

Güncel ChatGPT masaüstü derlemeleri Computer Use'ı burada paketler; eski bağımsız
Codex masaüstü derlemeleri `Codex.app` altında aynı düzeni kullanır:

```text
/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
```

`computerUse.autoInstall` true olduğunda ve
`computer-use` içeren hiçbir pazar yeri kayıtlı olmadığında OpenClaw, mevcut olan ilk standart
paketlenmiş pazar yeri kökünü eklemeye çalışır:

```text
/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled
```

Bunu Codex ile bir kabuktan açıkça da kaydedebilirsiniz:

```bash
codex plugin marketplace add /Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
```

Standart olmayan bir Codex uygulama yolu kullanıyorsanız `/codex computer-use install
--source <marketplace-root>` komutunu bir kez çalıştırın veya `computerUse.marketplacePath` değerini
yerel bir pazar yeri dosya yoluna ayarlayın. `--marketplace-path` değerini yalnızca
paketlenmiş pazar yeri köküne değil, pazar yeri JSON dosyasının yoluna sahip olduğunuzda kullanın.

### Paylaşılan Plugin önbelleği

Varsayılan `pluginCacheMode: "independent"`, her Codex ana dizinini ve onun
Plugin önbelleğini yönetimsiz bırakır. App-server başlatılmadan önce paketlenmiş
Computer Use Plugin'ini etkin Codex ana dizininin keşfedilebilir Plugin önbelleğine kopyalamak için
`pluginCacheMode: "shared"` ayarını yapın. Paylaşılan mod, çalışan Codex istemcileri sürümlendirilmiş Plugin dizinlerine hâlâ başvurabildiği için
eski önbelleğe alınmış sürümleri korur; başarısız bir değiştirme kopyası da etkin önbelleği korur. Açık
`marketplaceName` veya `marketplacePath` yapılandırması, OpenClaw'ın bu seçimi geçersiz kılmaması için
bu uzlaştırmayı devre dışı bırakır.

## Uzak katalog sınırı

Codex app-server yalnızca uzakta bulunan katalog girdilerini listeleyebilir ve okuyabilir, ancak şu anda
uzak `plugin/install` desteği sunmaz. Bu, `marketplaceName` değerinin
durum denetimleri için yalnızca uzakta bulunan bir pazar yerini seçebileceği, ancak yüklemeler ve
yeniden etkinleştirmeler için yine de `marketplaceSource` veya
`marketplacePath` üzerinden yerel bir pazar yerine ihtiyaç duyulduğu anlamına gelir.

Durum, Plugin'in uzak bir Codex pazar yerinde kullanılabilir olduğunu ancak
uzak yüklemenin desteklenmediğini belirtiyorsa yüklemeyi yerel bir kaynak veya yolla çalıştırın:

```text
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
```

## Yapılandırma referansı

| Alan                            | Varsayılan     | Anlamı                                                                         |
| ------------------------------- | -------------- | ------------------------------------------------------------------------------ |
| `enabled`                       | çıkarılan       | Computer Use gerektirir. Başka bir Computer Use alanı ayarlandığında varsayılan olarak true olur. |
| `autoInstall`                   | false          | Yerel istemciyi hazırlar ve tur başlangıcında plugini yükler veya yeniden etkinleştirir. |
| `marketplaceDiscoveryTimeoutMs` | 60000          | Yüklemenin Codex app-server pazar yeri keşfini ne kadar bekleyeceği.            |
| `liveTestTimeoutMs`             | 60000          | Geçici hazır olma iş parçacığı ve temizleme istekleri için zaman aşımı.         |
| `toolCallTimeoutMs`             | 60000          | Computer Use `list_apps` hazır olma araç çağrısı için zaman aşımı.              |
| `healthCheckEnabled`            | false          | Sahibi olan app-server istemcisi etkinken düzenli hazır olma yoklamaları çalıştırır. |
| `healthCheckIntervalMinutes`    | 60             | Yoklama sıklığı; kabul edilen değerler 30, 60, 120 veya 240 dakikadır.          |
| `pluginCacheMode`               | `independent`  | Codex ana dizini önbelleğini paketlenmiş masaüstü plugininden yenilemek için `shared` kullanır. |
| `strictReadiness`               | false          | Canlı yoklama başarısız olduğunda uyarıyla devam etmek yerine başlatmayı durdurur. |
| `autoRepair`                    | false          | Eski kapsamlı Computer Use MCP alt süreçlerini sonlandırır ve başarısız bir yoklamayı bir kez yeniden dener. |
| `marketplaceSource`             | ayarlanmamış   | Codex app-server `marketplace/add` öğesine iletilen kaynak dizesi.             |
| `marketplacePath`               | ayarlanmamış   | Plugini içeren yerel Codex pazar yeri dosya yolu.                               |
| `marketplaceName`               | ayarlanmamış   | Seçilecek kayıtlı Codex pazar yeri adı.                                         |
| `pluginName`                    | `computer-use` | Codex pazar yeri plugin adı.                                                    |
| `mcpServerName`                 | `computer-use` | Yüklenen plugin tarafından sunulan MCP sunucusu adı.                            |

Tur başlangıcındaki otomatik yükleme, yapılandırılmış `marketplaceSource`
değerlerini bilinçli olarak reddeder. Yeni bir kaynak eklemek açık bir kurulum
işlemidir; bu nedenle `/codex computer-use install --source <marketplace-source>` öğesini bir kez kullanın, ardından keşfedilen
yerel pazar yerlerinden gelecekteki yeniden etkinleştirmeleri
`autoInstall` öğesinin yönetmesine izin verin.
Tur başlangıcındaki otomatik yükleme, yapılandırılmış bir `marketplacePath`
kullanabilir; çünkü bu zaten ana makinedeki yerel bir yoldur.

Her alan, eşleşen yapılandırma anahtarı ayarlanmamış olduğunda denetlenen bir
ortam değişkeni geçersiz kılmasını da kabul eder:

| Alan                            | Ortam değişkeni                                                 |
| ------------------------------- | -------------------------------------------------------------- |
| `enabled`                       | `OPENCLAW_CODEX_COMPUTER_USE`                                  |
| `autoInstall`                   | `OPENCLAW_CODEX_COMPUTER_USE_AUTO_INSTALL`                     |
| `marketplaceDiscoveryTimeoutMs` | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_DISCOVERY_TIMEOUT_MS` |
| `liveTestTimeoutMs`             | `OPENCLAW_CODEX_COMPUTER_USE_LIVE_TEST_TIMEOUT_MS`             |
| `toolCallTimeoutMs`             | `OPENCLAW_CODEX_COMPUTER_USE_TOOL_CALL_TIMEOUT_MS`             |
| `healthCheckEnabled`            | `OPENCLAW_CODEX_COMPUTER_USE_HEALTH_CHECK_ENABLED`             |
| `healthCheckIntervalMinutes`    | `OPENCLAW_CODEX_COMPUTER_USE_HEALTH_CHECK_INTERVAL_MINUTES`    |
| `pluginCacheMode`               | `OPENCLAW_CODEX_COMPUTER_USE_PLUGIN_CACHE_MODE`                |
| `strictReadiness`               | `OPENCLAW_CODEX_COMPUTER_USE_STRICT_READINESS`                 |
| `autoRepair`                    | `OPENCLAW_CODEX_COMPUTER_USE_AUTO_REPAIR`                      |
| `marketplaceSource`             | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_SOURCE`               |
| `marketplacePath`               | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_PATH`                 |
| `marketplaceName`               | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_NAME`                 |
| `pluginName`                    | `OPENCLAW_CODEX_COMPUTER_USE_PLUGIN_NAME`                      |
| `mcpServerName`                 | `OPENCLAW_CODEX_COMPUTER_USE_MCP_SERVER_NAME`                  |

## OpenClaw neyi denetler?

OpenClaw, kararlı bir kurulum nedenini dahili olarak bildirir ve kullanıcıya
gösterilen durumu sohbet için biçimlendirir:

| Neden                        | Anlamı                                                 | Sonraki adım                                  |
| ---------------------------- | ------------------------------------------------------ | --------------------------------------------- |
| `disabled`                   | `computerUse.enabled` false olarak çözümlendi.         | `enabled` veya başka bir Computer Use alanı ayarlayın. |
| `marketplace_missing`        | Eşleşen bir pazar yeri yoktu.                           | Kaynağı, yolu veya pazar yeri adını yapılandırın. |
| `plugin_not_installed`       | Pazar yeri mevcut ancak plugin yüklü değil.              | Yüklemeyi çalıştırın veya `autoInstall` öğesini etkinleştirin. |
| `plugin_disabled`            | Plugin yüklü ancak Codex yapılandırmasında devre dışı.  | Yeniden etkinleştirmek için yüklemeyi çalıştırın. |
| `remote_install_unsupported` | Seçili pazar yeri yalnızca uzaktır.                      | `marketplaceSource` veya `marketplacePath` kullanın. |
| `mcp_missing`                | Plugin etkin ancak MCP sunucusu kullanılamıyor.         | Codex Computer Use ve işletim sistemi izinlerini denetleyin. |
| `ready`                      | Plugin ve MCP araçları kullanılabilir.                  | Codex modu turunu başlatın.                   |
| `check_failed`               | Durum denetimi sırasında bir Codex app-server isteği başarısız oldu. | App-server bağlantısını ve günlükleri denetleyin. |
| `auto_install_blocked`       | Tur başlangıcı kurulumu yeni bir kaynak eklemeyi gerektirir. | Önce açık yüklemeyi çalıştırın.               |

Sohbet çıktısı; plugin durumunu, MCP sunucusu durumunu, pazar yerini,
kullanılabilir olduğunda araçları ve başarısız olan kurulum adımına özgü iletiyi içerir.

## macOS izinleri

Codex'in sahip olduğu bu Computer Use yolu macOS üzerinde çalışır; burada MCP
sunucusunun uygulamaları inceleyebilmesi veya denetleyebilmesi için önce yerel
işletim sistemi izinlerine ihtiyacı olabilir. (Windows ve Linux node ana
makinelerinde platformlar arası masaüstü denetimi için
[cua-computer yerine getiricisine](/tr/nodes/computer-use#windows-and-linux-experimental-via-cua-driver) bakın.)
OpenClaw, Computer Use'ın yüklü olduğunu ancak MCP sunucusunun kullanılamadığını
söylüyorsa önce Codex tarafındaki Computer Use kurulumunu doğrulayın:

- Codex app-server, masaüstü denetiminin gerçekleşmesi gereken aynı ana makinede çalışıyor.
- Computer Use plugini Codex yapılandırmasında etkin.
- `computer-use` MCP sunucusu Codex app-server MCP durumunda görünüyor.
- macOS, masaüstü denetimi uygulaması için gerekli izinleri verdi.
- Geçerli ana makine oturumu, denetlenen masaüstüne erişebiliyor.

OpenClaw, `computerUse.enabled` true olduğunda bilinçli olarak kapalı durumda
başarısız olur. Codex modu turu, yapılandırmanın gerektirdiği yerel masaüstü
araçları olmadan sessizce ilerlememelidir.

## Sorun giderme

**Durum, yüklü olmadığını söylüyor.** `/codex computer-use install` çalıştırın. Pazar
yeri keşfedilmezse `--source` veya `--marketplace-path` iletin.

**Durum, yüklü ancak devre dışı olduğunu söylüyor.** `/codex computer-use install`
öğesini yeniden çalıştırın. Codex app-server yüklemesi, plugin yapılandırmasını
etkin olarak geri yazar.

**Durum, uzaktan yüklemenin desteklenmediğini söylüyor.** Yerel bir pazar
yeri kaynağı veya yolu kullanın. Yalnızca uzak katalog girdileri incelenebilir,
ancak geçerli app-server API'si üzerinden yüklenemez.

**Durum, MCP sunucusunun kullanılamadığını söylüyor.** MCP sunucularının
yeniden yüklenmesi için yüklemeyi bir kez daha çalıştırın. Kullanılamamaya devam
ederse Codex Computer Use uygulamasını, Codex app-server MCP durumunu veya macOS
izinlerini düzeltin.

**Durum veya bir yoklama `computer-use.list_apps` üzerinde zaman aşımına uğruyor.**
Plugin ve MCP sunucusu mevcut ancak yerel Computer Use köprüsü yanıt vermedi.
Codex Computer Use'dan çıkın veya onu yeniden başlatın; gerekirse Codex Desktop'ı
yeniden açın, ardından yeni bir OpenClaw oturumunda tekrar deneyin. Ana makine
daha önce Computer Use'ı eski bir yönetilen Codex app-server üzerinden çalıştırdıysa
yüklenen plugini masaüstüyle paketlenmiş pazar yerinden yenileyin (bağımsız Codex
masaüstü yüklemeleri için `Codex.app` yolunu kullanın):

```text
/codex computer-use install --source /Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
```

**Bir Computer Use aracı `Native hook relay unavailable` diyor.** Codex'e özgü araç kancası,
yerel köprü veya Gateway yedeği üzerinden etkin bir OpenClaw aktarıcısına
ulaşamadı. `/new` veya `/reset` ile yeni bir OpenClaw
oturumu başlatın. Bir kez çalışıp sonraki bir araç çağrısında yeniden başarısız
olursa `/new` yalnızca geçerli denemeyi temizliyordur; eski iş
parçacıklarının ve kanca kayıtlarının bırakılması için Codex app-server'ı veya
OpenClaw Gateway'i yeniden başlatın, ardından yeni bir oturumda tekrar deneyin.

**Tur başlangıcındaki otomatik yükleme bir kaynağı reddediyor.** Bu kasıtlıdır.
Önce kaynağı açık `/codex computer-use install --source
<marketplace-source>` ile ekleyin; ardından gelecekteki tur
başlangıcı otomatik yüklemeleri keşfedilen yerel pazar yerini kullanabilir.

## İlgili

- [Codex çalıştırma düzeneği](/tr/plugins/codex-harness)
- [Peekaboo köprüsü](/tr/platforms/mac/peekaboo)
- [iOS uygulaması](/tr/platforms/ios)
