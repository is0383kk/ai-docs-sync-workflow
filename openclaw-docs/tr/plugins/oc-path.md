---
read_when:
    - Çalışma alanındaki bir dosyanın tek bir yaprak öğesini terminalden incelemek veya düzenlemek istiyorsunuz
    - Çalışma alanı durumuna yönelik betikler yazıyorsunuz ve türden bağımsız, kararlı bir adresleme düzenine ihtiyacınız var
    - Kendi barındırdığınız bir Gateway'de isteğe bağlı `oc-path` Plugin'ini etkinleştirip etkinleştirmemeye karar veriyorsunuz
summary: 'Paketlenmiş `oc-path` plugin''i: `oc://` çalışma alanı dosyası adresleme şeması için `openclaw path` CLI''ını içerir'
title: OC Path Plugin'i
x-i18n:
    generated_at: "2026-07-27T00:06:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eb7bb1aacd37e5cc9c391372b871dc519f4048232d93a0016138ae00a6985a59
    source_path: plugins/oc-path.md
    workflow: 16
---

Paketle birlikte gelen `oc-path` plugin'i, `oc://` çalışma alanı dosyası adresleme şeması için [`openclaw path`](/tr/cli/path) CLI'sini ekler. OpenClaw reposunda `extensions/oc-path/` altında sunulur ancak isteğe bağlıdır: kurulum/derleme, siz etkinleştirene kadar onu devre dışı bırakır.

`oc://` adresleri, bir çalışma alanı dosyasındaki tek bir yaprak düğümü (veya joker karakterle eşleşen bir yaprak düğümü kümesini) gösterir. Plugin dört dosya türünü tanır:

- **markdown** (`.md`): frontmatter, bölümler, öğeler, alanlar
- **jsonc** (`.jsonc`, `.json`): yorumlar ve biçimlendirme korunur
- **jsonl** (`.jsonl`, `.ndjson`): satır odaklı kayıtlar
- **yaml** (`.yaml`, `.yml`, `.lobster`): `yaml` paketinin `Document` API'si üzerinden eşleme/dizi/skaler düğümleri

Kendi sistemlerinde barındıranlar ve düzenleyici uzantıları, doğrudan SDK'ye yönelik betik yazmadan tek bir yaprak düğümü okumak veya yazmak için CLI'yi kullanır; agent'lar ve hook'lar ise bayt doğruluğundaki gidiş dönüşlerin ve redaksiyon sentinel korumasının tüm türlerde aynı biçimde uygulanması için bunu deterministik bir temel olarak kullanır. Tam dil bilgisi, her komuta özgü bayrak listesi ve dosya türü başına uygulamalı örnekler için [CLI referansına](/tr/cli/path) bakın; bu sayfa plugin'in neden ve nasıl etkinleştirileceğini açıklar.

## Neden etkinleştirilmeli?

Betiklerin, hook'ların veya yerel agent araçlarının her dosya biçimi için özel bir ayrıştırıcı kullanmadan çalışma alanı durumunun belirli bir parçasını göstermesi gerektiğinde `oc-path` özelliğini etkinleştirin. Tek bir `oc://` adresi; bir markdown frontmatter anahtarını, bölüm öğesini, JSONC yapılandırma yaprak düğümünü, JSONL olay alanını veya YAML iş akışı adımını adlandırabilir.

Bu, değişikliğin küçük, denetlenebilir ve tekrarlanabilir kalması gereken bakımcı iş akışları için önemlidir: tek bir değeri inceleyin, eşleşen kayıtları bulun, bir yazma işleminin deneme çalıştırmasını yapın, ardından yorumları, satır sonlarını ve yakındaki biçimlendirmeyi olduğu gibi bırakarak yalnızca ilgili yaprak düğümüne uygulayın.

Etkinleştirmek için yaygın nedenler:

- **Yerel otomasyon**: shell betikleri, ayrı markdown, JSONC, JSONL ve YAML ayrıştırma kodları taşımak yerine `openclaw path … --json` ile tek bir çalışma alanı değerini çözümler veya günceller.
- **Agent tarafından görülebilen düzenlemeler**: bir agent, yazmadan önce adreslenmiş tek bir yaprak düğümü için deneme çalıştırması farkını gösterir; bunu incelemek serbest biçimli bir dosya yeniden yazımından daha kolaydır.
- **Düzenleyici entegrasyonları**: bir düzenleyici, başlık metninden tahminde bulunmadan `oc://AGENTS.md/tools/gh` öğesini tam markdown düğümü ve satır numarasıyla eşler.
- **Tanılama**: `emit`, bir dosyayı ayrıştırıcıdan ve yayıcıdan geçirerek gidiş dönüş işlemi yapar; böylece otomatik düzenlemelere güvenmeden önce bir dosya türünün bayt düzeyinde kararlı olup olmadığını denetleyebilirsiniz.

```bash
# GitHub plugin'i bu yapılandırmada etkin mi?
openclaw path resolve 'oc://config.jsonc/plugins/github/enabled' --json

# Bu oturum günlüğünde hangi araç çağrısı adları görünüyor?
openclaw path find 'oc://session.jsonl/[event=tool_call]/name' --json

# Bu küçük yapılandırma düzenlemesi hangi baytları yazar?
openclaw path set 'oc://config.jsonc/plugins/github/enabled' 'true' --dry-run
```

`oc-path`, üst düzey semantiğin sahibi olacak şekilde tasarlanmamıştır. Bellek plugin'leri bellek yazma işlemlerinin, yapılandırma komutları tam yapılandırma yönetiminin ve son bilinen iyi (LKG) yapılandırma kurtarma mekanizması da geri yükleme/yükseltme işlemlerinin sahibi olmaya devam eder. `oc-path`, bu üst düzey araçların çevresinde geliştirilebileceği dar kapsamlı adresleme ve bayt korumalı dosya işlemi katmanıdır.

## Nerede çalışır?

Plugin, komutu çağırdığınız ana makinede **`openclaw` CLI içinde, aynı süreçte** çalışır. Çalışan bir Gateway gerektirmez ve herhangi bir ağ soketi açmaz; her komut, belirttiğiniz dosya üzerinde saf bir dönüşümdür.

Plugin meta verileri `extensions/oc-path/openclaw.plugin.json` içinde bulunur:

```json
{
  "id": "oc-path",
  "name": "OC Path",
  "activation": {
    "onStartup": false,
    "onCommands": ["path"]
  },
  "commandAliases": [{ "name": "path", "kind": "cli" }]
}
```

`onStartup: false`, plugin'i Gateway başlangıç yolunun dışında tutar. `commandAliases` ve `activation.onCommands`, `openclaw path …` komutunu ilk kez çalıştırdığınızda CLI'nin plugin'i tembel olarak yüklemesini sağlar; böylece bu komutu hiç kullanmayan kurulumlar herhangi bir maliyete katlanmaz.

## Etkinleştirme

```bash
openclaw plugins enable oc-path
```

Manifest anlık görüntüsünün yeni durumu alması için Gateway'i (çalıştırıyorsanız) yeniden başlatın. Yalın `openclaw path` çağrıları aynı ana makinede hemen çalışır; CLI, plugin'i isteğe bağlı olarak yükler.

Devre dışı bırakmak için:

```bash
openclaw plugins disable oc-path
```

## Bağımlılıklar

Tüm ayrıştırıcı bağımlılıkları plugin'e özeldir; `oc-path` özelliğini etkinleştirmek, çekirdek çalışma zamanına yeni paketler eklemez:

| Bağımlılık     | Amaç                                                                |
| -------------- | ---------------------------------------------------------------------- |
| `commander`    | `resolve`, `find`, `set`, `validate`, `emit` için alt komut bağlantıları.    |
| `jsonc-parser` | Yorumları ve sondaki virgülleri koruyarak JSONC ayrıştırma ve yaprak düğümü düzenlemeleri.     |
| `markdown-it`  | Bölüm / öğe / alan modeli için Markdown tokenizasyonu.            |
| `yaml`         | Yorumları ve akış biçemini koruyarak YAML `Document` ayrıştırma / yayma / düzenleme. |

JSONL elle uygulanmaya devam eder: satır odaklı ayrıştırma herhangi bir bağımlılıktan daha basittir ve her satırın ayrıştırılması zaten `jsonc-parser` üzerinden geçer.

## Neler sağlar?

| Yüzey                        | Sağlayan                                             |
| ------------------------------ | ------------------------------------------------------- |
| `openclaw path` CLI            | `extensions/oc-path/cli-registration.ts`                |
| `oc://` ayrıştırıcı / biçimlendirici     | `extensions/oc-path/src/oc-path/oc-path.ts`             |
| Türe göre ayrıştırma / yayma / düzenleme   | `extensions/oc-path/src/oc-path/{md,jsonc,jsonl,yaml}`  |
| Evrensel çözümleme / bulma / ayarlama | `extensions/oc-path/src/oc-path/{resolve,find,edit}.ts` |
| Redaksiyon sentinel koruması       | `extensions/oc-path/src/oc-path/sentinel.ts`            |

CLI, günümüzdeki tek genel kullanıma açık yüzeydir. Temel komutlar plugin'e özeldir; tüketiciler CLI'yi kullanır (veya SDK'ye yönelik kendi plugin'lerini geliştirir).

## Diğer plugin'lerle ilişkisi

- **`memory-*`**: bellek yazma işlemleri `oc-path` üzerinden değil, bellek plugin'leri üzerinden gerçekleştirilir. `oc-path` genel amaçlı bir dosya temelidir; bellek plugin'leri kendi semantiklerini bunun üzerine katmanlar.
- **LKG**: `path`, son bilinen iyi yapılandırmayı geri yükleme mekanizması hakkında bilgi sahibi değildir. `path` üzerinden düzenlediğiniz bir dosya LKG tarafından da izleniyorsa sonraki yapılandırma gözlem döngüsü, dosyanın yükseltilmesine veya kurtarılmasına karar verir; bir `path` düzenlemesine, söz konusu dosyaya yapılan diğer doğrudan yazma işlemleriyle aynı şekilde yaklaşın.

## Güvenlik

`set`, temel katmanın redaksiyon sentinel korumasını otomatik olarak uygulayan yayma yolu üzerinden ham baytlar yazar. `__OPENCLAW_REDACTED__` değerini (birebir veya alt dize olarak) taşıyan bir yaprak düğümünün yazılması, `OC_EMIT_SENTINEL` ile reddedilir. CLI ayrıca yazdırdığı tüm insan tarafından okunabilir veya JSON çıktılarından değişmez sentinel değerini temizleyerek `[REDACTED]` ile değiştirir; böylece terminal kayıtları ve işlem hatları işaretçiyi hiçbir zaman sızdırmaz.

## İlgili

- [`openclaw path` CLI referansı](/tr/cli/path)
- [Plugin'leri yönetme](/tr/plugins/manage-plugins)
- [Plugin geliştirme](/tr/plugins/building-plugins)
