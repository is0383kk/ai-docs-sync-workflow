---
doc-schema-version: 1
read_when:
    - Pluginleri yükleme veya yapılandırma
    - Plugin keşfini ve yükleme kurallarını anlama
    - Codex/Claude uyumlu plugin paketleriyle çalışma
sidebarTitle: Getting Started
summary: OpenClaw pluginlerini yükleme, yapılandırma ve yönetme
title: Pluginler
x-i18n:
    generated_at: "2026-07-26T23:06:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f210dccab059527192eeb0aa2e780dcea243959273938ffaacc867ec96f5085e
    source_path: tools/plugin.md
    workflow: 16
---

Pluginler OpenClaw'ı kanallar, model sağlayıcıları, ajan çalıştırma çerçeveleri, araçlar,
beceriler, konuşma, gerçek zamanlı transkripsiyon, ses, medya anlama, üretim,
web getirme, web arama ve diğer çalışma zamanı yetenekleriyle genişletir.

Bir Plugin yüklemek, Gateway'i yeniden başlatmak, çalışma zamanının
onu yüklediğini doğrulamak ve yaygın kurulum hatalarını yönlendirmek için bu sayfayı kullanın. Yalnızca komut örnekleri için
[Pluginleri yönetin](/tr/plugins/manage-plugins) sayfasına bakın. Paketlenmiş, resmî harici ve
yalnızca kaynak biçimindeki Pluginlerin oluşturulan envanteri için
[Plugin envanteri](/tr/plugins/plugin-inventory) sayfasına bakın.

## Gereksinimler

- kullanılabilir `openclaw` CLI ile bir OpenClaw kaynak kopyası veya kurulumu
- seçilen kaynağa (ClawHub, npm veya bir git barındırıcısı) ağ erişimi
- ilgili Pluginin kurulum belgelerinde belirtilen Plugine özgü kimlik bilgileri, yapılandırma anahtarları veya
  işletim sistemi araçları
- kanallarınıza hizmet veren Gateway'in yeniden yüklenmesi veya yeniden başlatılması için izin

## Hızlı başlangıç

<Steps>
  <Step title="Plugini bulun">
    Herkese açık Plugin paketleri için [ClawHub](/tr/clawhub) üzerinde arama yapın:

    ```bash
    openclaw plugins search "calendar"
    ```

    ClawHub, topluluk Pluginleri için birincil keşif yüzeyidir. Lansman
    geçişi sırasında, sıradan yalın paket belirtimleri resmî bir Plugin kimliğiyle
    eşleşmedikleri sürece npm'den yüklenmeye devam eder. Paketlenmiş bir Pluginle
    eşleşen ham `@openclaw/*` belirtimleri, bu paketlenmiş kopyaya çözümlenir.
    Belirli bir kaynağa ihtiyacınız olduğunda açık bir kaynak öneki kullanın.

  </Step>

  <Step title="Plugini yükleyin">
    ```bash
    # ClawHub'dan.
    openclaw plugins install clawhub:<package>

    # npm'den.
    openclaw plugins install npm:<package>

    # git'ten.
    openclaw plugins install git:github.com/<owner>/<repo>@<ref>

    # Yerel bir geliştirme kaynak kopyasından.
    openclaw plugins install ./my-plugin
    openclaw plugins install --link ./my-plugin
    ```

    Plugin yüklemelerini kod çalıştırmak gibi değerlendirin. Yeniden üretilebilir
    üretim kurulumları için sabitlenmiş sürümleri tercih edin. ClawHub paketleri ve OpenClaw'ın
    paketlenmiş/resmî kataloğu güvenilir kaynaklardır. Yeni ve rastgele npm, git,
    yerel yol/arşiv, `npm-pack:` veya pazar yeri kaynakları, kaynağı
    inceleyip güvendikten sonra etkileşimsiz kurulumlarda
    `--force` gerektirir.

  </Step>

  <Step title="Yapılandırın ve etkinleştirin">
    Plugine özgü ayarları `plugins.entries.<id>.config` altında yapılandırın.
    Henüz etkin değilse Plugini etkinleştirin:

    ```bash
    openclaw plugins enable <plugin-id>
    ```

    `plugins.allow` ayarlanmışsa Pluginin yüklenebilmesi için kurulan Plugin kimliği
    bu listede bulunmalıdır. `openclaw plugins install`, kurulan kimliği mevcut
    `plugins.allow` listesine ekler ve aynı kimliği `plugins.deny`
    listesinden kaldırır; böylece açıkça yüklenen Plugin yeniden başlatmadan sonra yüklenebilir.

  </Step>

  <Step title="Gateway'in yeniden yüklenmesine izin verin">
    Plugin kodunu yüklemek, güncellemek veya kaldırmak Gateway'in yeniden
    başlatılmasını gerektirir. Yapılandırma yeniden yüklemesi etkin olan yönetilen bir Gateway,
    değişen Plugin yükleme kaydını algılar ve otomatik olarak yeniden başlatılır. Aksi takdirde
    kendiniz yeniden başlatın:

    ```bash
    openclaw gateway restart
    ```

    Etkinleştirme/devre dışı bırakma işlemleri yapılandırmayı ve soğuk kayıt defterini günceller. Çalışma zamanı incelemesi,
    canlı çalışma zamanı yüzeylerinin hâlâ en açık kanıtıdır.

  </Step>

  <Step title="Çalışma zamanı kaydını doğrulayın">
    ```bash
    openclaw plugins inspect <plugin-id> --runtime --json
    ```

    Kayıtlı araçları, kancaları, hizmetleri, Gateway yöntemlerini veya Plugine ait
    CLI komutlarını kanıtlamak için `--runtime` kullanın. Düz `inspect`
    yalnızca soğuk bildirim ve kayıt defteri denetimidir.

  </Step>
</Steps>

## Yapılandırma

### Bir yükleme kaynağı seçin

| Kaynak      | Kullanılacağı durum                                                             | Örnek                                                          |
| ----------- | ------------------------------------------------------------------------------ | -------------------------------------------------------------- |
| ClawHub     | OpenClaw'a özgü keşif, taramalar, sürüm meta verileri ve yükleme ipuçları istediğinizde | `openclaw plugins install clawhub:<package>`                   |
| npm         | Doğrudan npm kayıt defteri veya dist-tag iş akışlarına ihtiyacınız olduğunda   | `openclaw plugins install npm:<package>`                       |
| git         | Bir depodan dal, etiket veya kayda ihtiyacınız olduğunda                       | `openclaw plugins install git:github.com/<owner>/<repo>@<ref>` |
| yerel yol   | Aynı makinede bir Plugin geliştirirken veya test ederken                       | `openclaw plugins install --link ./my-plugin`                  |
| pazar yeri  | Claude uyumlu bir pazar yeri Plugini yüklerken                                 | `openclaw plugins install <plugin> --marketplace <source>`     |

Yalın paket belirtimleri özel uyumluluk davranışına sahiptir: paketlenmiş bir
Plugin kimliğiyle eşleşen yalın ad, bu paketlenmiş kaynağı kullanır; resmî harici
bir Plugin kimliğiyle eşleşen yalın ad, resmî paket kataloğunu kullanır; diğer
tüm yalın belirtimler lansman geçişi sırasında npm üzerinden yüklenir. Paketlenmiş
Pluginlerle eşleşen ham `@openclaw/*` belirtimleri de npm geri dönüşünden önce
paketlenmiş kopyaya çözümlenir. Paketlenmiş kopya yerine harici npm paketini
bilerek yüklemek için `npm:@openclaw/<plugin>@<version>` kullanın. Belirlenimci kaynak seçimi için
`clawhub:`, `npm:`, `git:` veya
`npm-pack:` kullanın. Tam komut sözleşmesi için
[`openclaw plugins`](/tr/cli/plugins#install) sayfasına bakın.

npm yüklemelerinde sabitlenmemiş belirtimler ve `@latest`, bu OpenClaw
derlemesiyle uyumluluğunu bildiren en yeni kararlı paketi seçer. npm'in mevcut
en son sürümü, bu derlemenin desteklediğinden daha yeni bir `openclaw.compat.pluginApi`
veya `openclaw.install.minHostVersion` bildiriyorsa OpenClaw daha eski kararlı sürümleri
tarar ve uygun olan en yenisini yükler. Kesin sürümler ve `@beta`
gibi açık kanal etiketleri seçilen pakete sabitlenmiş olarak kalır ve uyumsuzsa
başarısız olur.

### Operatör yükleme politikası

Bir Plugin yükleme veya güncelleme işlemi devam etmeden önce güvenilir bir yerel
politika komutu çalıştırmak için `security.installPolicy` yapılandırın. Politika, meta
verileri ve hazırlanmış kaynak yolunu alır; yüklemeye izin verebilir veya yüklemeyi
engelleyebilir. Hem CLI hem de Gateway destekli yükleme/güncelleme yollarını
kapsar. Plugin `before_install` kancaları daha sonra ve yalnızca Plugin
kancalarının yüklendiği OpenClaw süreçlerinde çalışır; bu nedenle operatöre ait
yükleme kararları için bunun yerine `security.installPolicy` kullanın. Kullanımdan
kaldırılmış `--dangerously-force-unsafe-install` bayrağı uyumluluk amacıyla kabul edilir ancak
işlem yapmaz: yükleme politikasını veya OpenClaw'ın yerleşik Plugin bağımlılığı
engelleme listesini atlamaz.

Hem beceriler hem de Pluginler tarafından kullanılan ortak `security.installPolicy`
exec şeması için [Skills yapılandırması](/tr/tools/skills-config#operator-install-policy-securityinstallpolicy)
sayfasına bakın.

### Plugin politikasını yapılandırın

Yaygın Plugin yapılandırma biçimi şöyledir:

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: { paths: ["~/Projects/oss/voice-call-plugin"] },
    slots: { memory: "memory-core" },
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } },
    },
  },
}
```

Temel politika kuralları:

- `plugins.enabled: false` tüm Pluginleri devre dışı bırakır ve keşif/yükleme
  çalışmalarını atlar. Bu etkinken eski Plugin başvuruları eylemsiz kalır; eski
  kimliklerin kaldırılmasını istiyorsanız doctor temizliğini çalıştırmadan önce
  Pluginleri yeniden etkinleştirin.
- `plugins.deny`, izin listesini ve Plugin bazındaki etkinleştirmeyi geçersiz kılar.
- `plugins.allow` özel bir izin listesidir. `tools.allow`,
  `"*"` öğesini içerse bile izin listesinin dışındaki Plugine ait
  araçlar kullanılamaz.
- `plugins.entries.<id>.enabled: false`, yapılandırmasını korurken tek bir Plugini devre dışı bırakır.
- `plugins.load.paths`, açık yerel Plugin dosyaları veya dizinleri ekler.
  Yönetilen `plugins install` yerel yolları Plugin dizinleri veya arşivleri
  olmalıdır; bağımsız Plugin dosyaları için `plugins.load.paths` kullanın.
- Çalışma alanı kökenli Pluginler varsayılan olarak devre dışıdır; yerel çalışma
  alanı kodunu kullanmadan önce bunları açıkça etkinleştirin veya izin listesine ekleyin.
- Paketlenmiş Pluginler, yapılandırma açıkça geçersiz kılmadığı sürece yerleşik
  varsayılan-açık/varsayılan-kapalı meta verilerini izler.
- `plugins.slots.<slot>` (`memory` veya `contextEngine`), özel bir
  kategori için bir Plugin seçer. Yuva seçimi açık etkinleştirme sayılır ve
  normalde isteğe bağlı olsa bile seçilen Plugini bu yuva için zorla etkinleştirir.
  `plugins.deny` ve `plugins.entries.<id>.enabled: false` bunu yine de engeller.
- Paketlenmiş isteğe bağlı Pluginler; yapılandırma, sahip oldukları sağlayıcı/model
  başvurusu, kanal yapılandırması, CLI arka ucu veya ajan çalıştırma çerçevesi çalışma
  zamanı gibi yüzeylerden birini adlandırdığında otomatik olarak etkinleşebilir.
- OpenAI ailesi Codex yönlendirmesi, sağlayıcı ve çalışma zamanı Plugin sınırlarını
  ayrı tutar: eski Codex model başvuruları doctor'ın onardığı eski yapılandırmadır;
  paketlenmiş `codex` Plugini ise kurallı `openai/*` ajan
  başvuruları, açık `agentRuntime.id: "codex"` ve eski `codex/*` başvuruları için
  Codex uygulama sunucusu çalışma zamanına sahiptir.

`plugins.allow` ayarlanmamışken paketlenmemiş Pluginler çalışma alanından veya
genel Plugin köklerinden otomatik keşfedildiğinde başlangıç günlükleri, keşfedilen
Plugin kimlikleriyle ve kısa listelerde asgari bir `plugins.allow` parçacığıyla
`plugins.allow is empty; discovered non-bundled plugins may auto-load: ...` kaydını gösterir. Güvenilir Pluginleri `openclaw.json`
içine kopyalamadan önce listelenen Plugin kimliği üzerinde
[`openclaw plugins list --enabled --verbose`](/tr/cli/plugins#list) veya
[`openclaw plugins inspect <id>`](/tr/cli/plugins#inspect) çalıştırın. Tanılama, bir Pluginin
`without install/load-path provenance` yüklediğini belirttiğinde de aynı güven sabitlemesi geçerlidir:
bu Plugin kimliğini inceleyin, ardından `plugins.allow` içinde sabitleyin veya
OpenClaw'ın yükleme kaynağını kaydetmesi için güvenilir bir kaynaktan yeniden yükleyin.

Yapılandırma doğrulaması eski Plugin kimlikleri, izin listesi/araç uyuşmazlıkları
veya eski paketlenmiş Plugin yolları bildirdiğinde `openclaw doctor` ya da
`openclaw doctor --fix` çalıştırın.

## Plugin biçimlerini anlayın

OpenClaw iki Plugin biçimini tanır:

| Biçim                  | Nasıl yüklenir                                                               | Kullanılacağı durum                                                     |
| ---------------------- | ---------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Yerel OpenClaw Plugini | `openclaw.plugin.json` ve süreç içinde yüklenen bir çalışma zamanı modülü        | OpenClaw'a özgü çalışma zamanı yetenekleri yüklerken veya oluştururken |
| Uyumlu paket           | OpenClaw Plugin envanterine eşlenen Codex, Claude veya Cursor Plugin düzeni   | Uyumlu becerileri, komutları, kancaları veya paket meta verilerini yeniden kullanırken |

Her iki biçim de `openclaw plugins list`, `openclaw plugins inspect`,
`openclaw plugins enable` ve `openclaw plugins disable` içinde görünür. Paket uyumluluğu sınırı
için [Plugin paketleri](/tr/plugins/bundles), yerel Plugin yazımı için
[Plugin oluşturma](/tr/plugins/building-plugins) sayfasına bakın.

## Plugin kancaları

Pluginler çalışma zamanında iki farklı API üzerinden kanca kaydedebilir:

- `api.on(...)`, çalışma zamanı yaşam döngüsü olayları için türü belirtilmiş
  kancalardır. Ara yazılım, politika, ileti yeniden yazma, istem şekillendirme ve
  araç denetimi için tercih edilen yüzey budur.
- `api.registerHook(...)`, [Kancalar](/tr/automation/hooks) bölümünde açıklanan iç
  kanca sistemi içindir. Bu, çoğunlukla kaba komut/yaşam döngüsü yan etkileri ve
  mevcut HOOK tarzı otomasyonla uyumluluk içindir.

Kısa kural: işleyicinin önceliğe, birleştirme semantiğine veya engelleme/iptal
davranışına ihtiyacı varsa türü belirtilmiş kancaları kullanın. Yalnızca
`command:new`, `command:reset`, `message:sent` veya benzeri kaba
olaylara tepki veriyorsa `api.registerHook` uygundur.

Plugin tarafından yönetilen iç kancalar, `openclaw hooks list` içinde
`plugin:<id>` ile görünür. Bunları `openclaw hooks` üzerinden etkinleştiremez
veya devre dışı bırakamazsınız; bunun yerine Plugini etkinleştirin veya devre dışı bırakın.

## Etkin Gateway'i doğrulayın

`openclaw plugins list` ve düz `openclaw plugins inspect`, soğuk yapılandırma,
manifest ve kayıt defteri durumunu okur. Bunlar, hâlihazırda çalışan bir
Gateway'in aynı Plugin kodunu içe aktardığını kanıtlamaz.

Bir Plugin kurulu görünmesine rağmen canlı sohbet trafiği onu kullanmıyorsa:

```bash
openclaw gateway status --deep --require-rpc
openclaw plugins inspect <plugin-id> --runtime --json
openclaw gateway restart
```

Yönetilen Gateway'ler, Plugin kaynağını değiştiren Plugin kurma, güncelleme ve
kaldırma işlemlerinden sonra otomatik olarak yeniden başlatılır. VPS veya konteyner kurulumlarında,
elle yeniden başlatma işleminin yalnızca bir sarmalayıcıyı ya da denetleyiciyi değil,
kanallarınıza hizmet veren gerçek `openclaw gateway run` alt sürecini hedeflediğinden emin olun.

## Sorun giderme

| Belirti                                                        | Kontrol                                                                                                                                      | Düzeltme                                                                                                     |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| Plugin, `plugins list` içinde görünüyor ancak çalışma zamanı kancaları çalışmıyor  | `openclaw plugins inspect <id> --runtime --json` kullanın ve `gateway status --deep --require-rpc` ile etkin Gateway'i doğrulayın             | Kurma, güncelleme, yapılandırma veya kaynak değişikliklerinden sonra canlı Gateway'i yeniden başlatın                               |
| Yinelenen kanal veya araç sahipliği tanılamaları görünüyor         | `openclaw plugins list --enabled --verbose` komutunu çalıştırın, şüphelenilen her Plugin'i `--runtime --json` ile inceleyin ve kanal/araç sahipliğini karşılaştırın | Sahiplerden birini devre dışı bırakın, eski kurulumları kaldırın veya kasıtlı değiştirme için manifest `preferOver` kullanın      |
| Yapılandırma bir Plugin'in eksik olduğunu söylüyor                                | Yerleşik, resmî haricî veya yalnızca kaynak olarak sunulup sunulmadığını öğrenmek için [Plugin envanterini](/tr/plugins/plugin-inventory) kontrol edin                           | Haricî paketi kurun, yerleşik Plugin'i etkinleştirin veya eski yapılandırmayı kaldırın                         |
| Kurulum sırasında yapılandırma geçersiz                                | Doğrulama mesajını okuyun ve eski Plugin durumunu işaret ediyorsa `openclaw doctor --fix` komutunu çalıştırın                                             | Doctor, girdiyi devre dışı bırakıp geçersiz yükü kaldırarak geçersiz Plugin yapılandırmasını karantinaya alabilir     |
| Şüpheli sahiplik veya izinler nedeniyle Plugin yolu engelleniyor | Yapılandırma hatasından önceki tanılamayı inceleyin                                                                                             | Dosya sistemi sahipliğini/izinlerini düzeltin, ardından `openclaw plugins registry --refresh` komutunu çalıştırın                    |
| `OPENCLAW_NIX_MODE=1`, yaşam döngüsü komutlarını engelliyor                | Kurulumun Nix tarafından yönetildiğini doğrulayın                                                                                                      | Plugin değiştirici komutlarını kullanmak yerine Nix kaynağındaki Plugin seçimini değiştirin                      |
| Çalışma zamanında bağımlılık içe aktarma işlemi başarısız oluyor                             | Plugin'in npm/git/ClawHub üzerinden mi kurulduğunu yoksa yerel bir yoldan mı yüklendiğini kontrol edin                                                 | `openclaw plugins update <id>` komutunu çalıştırın, kaynağı yeniden kurun veya yerel Plugin bağımlılıklarını kendiniz kurun |

Etkinleştirilmiş yönetilen bir Plugin, Gateway başlatılırken yük doğrulamasında
başarısız olduğunda OpenClaw, ilgili kurulu Plugin kökünü o önyükleme için karantinaya
alır ve diğer Plugin'lere hizmet vermeyi sürdürür. `openclaw status --all`, `openclaw health`
ve `openclaw doctor`, bunu `configured-unavailable` olarak bildirir. Plugin'i düzeltin
veya yeniden kurun, ardından Gateway'i yeniden başlatın. Aynı Plugin kimliğine sahip
sağlıklı ve açık bir `plugins.load.paths` geçersiz kılması, eski ve bozuk bir kurulum
nedeniyle karantinaya alınmaz.

Eski Plugin yapılandırması artık keşfedilemeyen bir kanal Plugin'ini hâlâ adlandırıyorsa,
yapılandırma doğrulaması bu kanal anahtarını kesin hata yerine uyarıya indirger;
böylece Gateway başlatma işlemi diğer tüm kanallara hizmet vermeyi sürdürebilir.
Eski Plugin ve kanal girdilerini kaldırmak için `openclaw doctor --fix` komutunu çalıştırın.
Eski Plugin kanıtı bulunmayan bilinmeyen kanal anahtarları doğrulamada yine başarısız
olur; böylece yazım hataları görünür kalır.

Kasıtlı kanal değiştirme için tercih edilen Plugin, eski veya daha düşük öncelikli
Plugin kimliğini içeren `channelConfigs.<channel-id>.preferOver` bildirimini yapmalıdır.
Her iki Plugin de açıkça etkinleştirilmişse OpenClaw bu isteği korur ve sessizce
bir sahip seçmek yerine yinelenen kanal/araç tanılamalarını bildirir.

Kurulu bir paket `requires compiled runtime output for
TypeScript entry ...` bildiriminde bulunuyorsa paket,
OpenClaw'ın çalışma zamanında ihtiyaç duyduğu JavaScript dosyaları olmadan yayımlanmıştır.
Yayımcı derlenmiş JavaScript'i sunduktan sonra güncelleyin veya yeniden kurun;
o zamana kadar Plugin'i devre dışı bırakın ya da kaldırın.

### Engellenen Plugin yolu sahipliği

Tanılamalar
`blocked plugin candidate: suspicious ownership (... uid=1000, expected uid=0 or root)`
diyorsa ve doğrulama bunu `plugin present but blocked` ile izliyorsa OpenClaw,
Plugin dosyalarının onları yükleyen süreçten farklı bir Unix kullanıcısına ait olduğunu
tespit etmiştir. Plugin yapılandırmasını yerinde tutun; dosya sistemi sahipliğini düzeltin
veya OpenClaw'ı durum dizininin sahibi olan kullanıcıyla çalıştırın.

Docker kurulumlarında resmî imaj `node` (uid `1000`) olarak çalışır;
bu nedenle ana makineye bağlama yoluyla bağlanan OpenClaw yapılandırma ve çalışma alanı
dizinleri normalde uid `1000` kullanıcısına ait olmalıdır:

```bash
sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
```

OpenClaw'ı kasıtlı olarak root kullanıcısı şeklinde çalıştırıyorsanız bunun yerine
yönetilen Plugin kökünün sahipliğini root olarak düzeltin:

```bash
sudo chown -R root:root /path/to/openclaw-config/npm
```

Sahipliği düzelttikten sonra kalıcı Plugin kayıt defterinin onarılan dosyalarla
eşleşmesi için `openclaw doctor --fix` veya `openclaw plugins registry --refresh`
komutunu yeniden çalıştırın.

### Yavaş Plugin aracı kurulumu

Araçlar hazırlanırken agent turları takılıyor gibi görünüyorsa izleme günlük kaydını
etkinleştirin ve Plugin aracı fabrikası zamanlama satırlarını kontrol edin:

```bash
openclaw config set logging.level trace
openclaw logs --follow
```

Şunu arayın:

```text
[trace:plugin-tools] factory timings ...
```

Özet, Plugin kimliği, bildirilen araç adları, sonuç biçimi ve aracın isteğe bağlı olup
olmadığı dâhil olmak üzere toplam fabrika süresini ve en yavaş Plugin aracı fabrikalarını
listeler. Tek bir fabrika en az 1s sürdüğünde veya toplam Plugin aracı fabrikası hazırlığı
en az 5s sürdüğünde yavaş satırlar uyarı düzeyine yükseltilir.

OpenClaw, aynı etkin istek bağlamıyla yinelenen çözümlemeler için başarılı Plugin aracı
fabrikası sonuçlarını önbelleğe alır. Önbellek anahtarı; etkin çalışma zamanı yapılandırmasını,
çalışma alanı ve agent kimliğini, korumalı alan politikasını, tarayıcı ayarlarını, teslimat
bağlamını, istekte bulunanın kimliğini ve sahiplik durumunu içerir; böylece bu güvenilir
alanlara bağlı fabrikalar bağlam değiştiğinde yeniden çalışır. Zamanlamalar yüksek kalırsa
Plugin, araç tanımlarını döndürmeden önce maliyetli işlemler yapıyor olabilir.

Zamanlamada tek bir Plugin baskınsa çalışma zamanı kayıtlarını inceleyin:

```bash
openclaw plugins inspect <plugin-id> --runtime --json
```

Ardından bu Plugin'i güncelleyin, yeniden kurun veya devre dışı bırakın. Plugin yazarları,
maliyetli bağımlılık yükleme işlemini araç fabrikasının içinde yapmak yerine araç yürütme
yolunun arkasına taşımalıdır.

Bağımlılık kökleri, paket meta verisi doğrulaması, kayıt defteri kayıtları, başlangıçta
yeniden yükleme davranışı ve eski öğelerin temizlenmesi hakkında bilgi için
[Plugin bağımlılığı çözümlemesine](/tr/plugins/dependency-resolution) bakın.

## İlgili

- [Plugin'leri yönetme](/tr/plugins/manage-plugins) - listeleme, kurma, güncelleme, kaldırma ve yayımlama için komut örnekleri
- [`openclaw plugins`](/tr/cli/plugins) - eksiksiz CLI başvurusu
- [Plugin envanteri](/tr/plugins/plugin-inventory) - oluşturulan yerleşik ve haricî Plugin listesi
- [Plugin başvurusu](/tr/plugins/reference) - Plugin başına oluşturulan başvuru sayfaları
- [Topluluk Plugin'leri](/tr/plugins/community) - ClawHub keşfi ve dokümantasyon PR politikası
- [Plugin bağımlılığı çözümlemesi](/tr/plugins/dependency-resolution) - kurulum kökleri, kayıt defteri kayıtları ve çalışma zamanı sınırları
- [Plugin oluşturma](/tr/plugins/building-plugins) - yerel Plugin yazma kılavuzu
- [Plugin SDK'ya genel bakış](/tr/plugins/sdk-overview) - çalışma zamanı kaydı, kancalar ve API alanları
- [Plugin manifesti](/tr/plugins/manifest) - manifest ve paket meta verileri
