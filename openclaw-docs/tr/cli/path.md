---
read_when:
    - Terminalden bir çalışma alanı dosyasındaki bir yaprak öğeyi okumak veya yazmak istiyorsunuz
    - Çalışma alanı durumuna yönelik betikler yazıyorsunuz ve türden bağımsız, kararlı bir adresleme şeması istiyorsunuz
    - Bir `oc://` yolunda hata ayıklıyorsunuz (sözdizimini doğrulayın, neye çözümlendiğine bakın)
summary: '`oc://` adresleme şeması aracılığıyla çalışma alanı dosyalarını incelemek ve düzenlemek için `openclaw path` CLI referansı'
title: Yol
x-i18n:
    generated_at: "2026-07-26T23:52:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7afe5bd1c3a5fca8dd22c7d807e390e751ae7e895c54bf0e10e2734f3889436c
    source_path: cli/path.md
    workflow: 16
---

# `openclaw path`

`oc://` adresleme şemasına kabuk erişimi: adreslenebilir çalışma alanı dosyalarını (markdown, jsonc,
jsonl, yaml/yml/lobster) incelemek ve düzenlemek için türe göre yönlendirilen tek bir yol söz dizimi.
Kendi barındırmasını yapanlar, plugin yazarları ve düzenleyici uzantıları; her dosya için ayrı bir ayrıştırıcıyı
elle oluşturmak zorunda kalmadan dar kapsamlı bir konumu okumak, bulmak veya güncellemek için bunu kullanır.

`path`, paketle birlikte gelen isteğe bağlı `oc-path` plugin'i tarafından sağlanır. İlk
kullanımdan önce etkinleştirin:

```bash
openclaw plugins enable oc-path
```

CLI fiilleri adresleme modelini yansıtır:

- `resolve` somuttur ve tek eşleşme döndürür.
- `find`; joker karakterler, birleşimler, koşullar ve
  konumsal genişletme için çoklu eşleşme fiilidir.
- `set` yalnızca somut yolları veya ekleme işaretçilerini kabul eder; joker karakter kalıpları
  yazma işleminden önce reddedilir.
- `validate`, dosya sistemine erişmeden bir yolu ayrıştırır.
- `emit`, bir dosyayı ayrıştırma + çıktı oluşturma işleminden geçirir (bayt doğruluğu tanılaması).

## Neden kullanılmalı?

OpenClaw durumu; insanlar tarafından düzenlenen markdown, yorumlu JSONC
yapılandırması, yalnızca sonuna ekleme yapılan JSONL günlükleri ve YAML iş akışı/belirtim dosyaları arasında dağılmıştır. Betikler, kancalar
ve aracılar genellikle bu dosyalardan yalnızca küçük bir değere ihtiyaç duyar: bir frontmatter anahtarı,
bir plugin ayarı, günlük kaydı alanı, YAML adımı veya adlandırılmış bir bölümün altındaki
madde işaretli bir öğe.

`openclaw path`, bu çağıranlara her dosya türü için tek seferlik bir
grep, düzenli ifade veya ayrıştırıcı yerine kararlı bir adres sağlar. Aynı `oc://` yolu terminalden doğrulanabilir,
çözümlenebilir, aranabilir, deneme amaçlı çalıştırılabilir ve yazılabilir; bu da dar kapsamlı
otomasyonun incelenebilir ve yeniden yürütülebilir kalmasını sağlar. Dosyanın geri kalanını koruduğundan
tek bir yaprağın yazılması yorumları, satır sonlarını veya yakındaki
biçimlendirmeyi bozmaz.

İstenen şey mantıksal bir adrese sahipken dosya biçimi
değişiyorsa bunu kullanın:

- Bir kanca, yorumlu JSONC'den tek bir ayarı okur ve
  değeri geri yazarken yorumları kaybetmez.
- Bir bakım betiği, tüm günlüğü özel bir ayrıştırıcıya
  yüklemeden JSONL günlüğündeki eşleşen tüm olay alanlarını bulur.
- Bir düzenleyici, markdown bölümüne veya madde işaretli öğeye slug ile atlar ve ardından
  çözümlediği kesin satırı işler.
- Bir aracı, küçük bir çalışma alanı düzenlemesini uygulamadan önce deneme amaçlı çalıştırır ve
  değişen baytlar incelemede görünür olur.

Sıradan tam dosya düzenlemeleri, kapsamlı yapılandırma geçişleri veya
belleğe özgü yazma işlemleri için `openclaw path` kullanmayın; bunlar sahip komutunu veya plugin'i kullanmalıdır. `path`,
başka bir özel ayrıştırıcı yerine yinelenebilir bir terminal komutunun daha uygun olduğu küçük,
adreslenebilir dosya işlemleri içindir.

## Nasıl kullanılır?

İnsanlar tarafından düzenlenen bir yapılandırma dosyasından tek bir değer okuyun:

```bash
openclaw path resolve 'oc://config.jsonc/plugins/github/enabled'
```

Diske dokunmadan bir yazma işlemini önizleyin:

```bash
openclaw path set 'oc://config.jsonc/plugins/github/enabled' 'true' --dry-run
```

Yalnızca sonuna ekleme yapılan JSONL günlüğündeki eşleşen kayıtları bulun:

```bash
openclaw path find 'oc://session.jsonl/[event=tool_call]/name'
```

Markdown'daki bir talimatı satır numarası yerine bölüm ve öğe ile
adresleyin:

```bash
openclaw path resolve 'oc://AGENTS.md/runtime-safety/openclaw-gateway'
```

Bir betik okuma veya yazma işlemi gerçekleştirmeden önce CI'da ya da bir ön kontrol betiğinde yolu
doğrulayın:

```bash
openclaw path validate 'oc://AGENTS.md/tools/$last/risk'
```

Bu komutlar kabuk betiklerine kopyalanabilecek şekilde tasarlanmıştır. Bir çağıranın
yapılandırılmış çıktıya ihtiyacı olduğunda `--json`, bir kişi sonucu incelerken
`--human` kullanın.

## Nasıl çalışır?

1. `oc://` adresini şu yuvalara ayrıştırır: dosya, bölüm, öğe, alan ve
   isteğe bağlı bir oturum sorgusu.
2. Hedef uzantıdan dosya türü bağdaştırıcısını seçer (`.md`, `.jsonc`,
   `.json`, `.jsonl`, `.ndjson`, `.yaml`, `.yml`, `.lobster`).
3. Yuvaları ilgili dosya türünün yapısına göre çözümler: markdown
   başlıkları/öğeleri, JSONC nesne anahtarları/dizi indeksleri, JSONL satır kayıtları veya
   YAML eşleme/dizi düğümleri.
4. `set` için, düzenlenmiş baytları aynı bağdaştırıcı üzerinden oluşturur; böylece dosyanın
   dokunulmayan bölümleri, dosya türünün desteklediği ölçüde yorumlarını, satır sonlarını ve yakındaki
   biçimlendirmeyi korur.

`resolve` ve `set` tek bir somut hedef gerektirir. `find` keşif
fiilidir: joker karakterleri, birleşimleri, koşulları ve sıra belirteçlerini, yazılacak bir hedef seçmeden önce
inceleyebileceğiniz somut eşleşmelere genişletir.

## Alt komutlar

| Alt komut               | Amaç                                                                        |
| ----------------------- | --------------------------------------------------------------------------- |
| `resolve <oc-path>`     | Yoldaki somut eşleşmeyi (veya "bulunamadı") yazdırır.                       |
| `find <pattern>`        | Joker karakter / birleşim / koşul yolunun eşleşmelerini listeler.           |
| `set <oc-path> <value>` | Somut bir yoldaki yaprağı veya ekleme hedefini yazar. `--dry-run` destekler. |
| `validate <oc-path>`    | Yalnızca ayrıştırır; yapısal dökümü (dosya / bölüm / öğe / alan) yazdırır.   |
| `emit <file>`           | Bir dosyayı ayrıştırma + çıktı oluşturma işleminden geçirir (bayt doğruluğu tanılaması). |

## Genel bayraklar

| Bayrak          | Uygulandığı yer                  | Amaç                                                                     |
| --------------- | -------------------------------- | ------------------------------------------------------------------------ |
| `--cwd <dir>`   | `resolve`, `find`, `set`, `emit` | Dosya yuvasını bu dizine göre çözümler (varsayılan: `process.cwd()`).  |
| `--file <path>` | `resolve`, `find`, `set`, `emit` | Dosya yuvasının çözümlenen yolunu geçersiz kılar (mutlak erişim).         |
| `--json`        | tümü                             | JSON çıktısını zorunlu kılar (stdout bir TTY değilken varsayılandır).     |
| `--human`       | tümü                             | İnsan tarafından okunabilir çıktıyı zorunlu kılar (stdout bir TTY iken varsayılandır). |
| `--value-json`  | `set`                            | JSON/JSONC/JSONL yaprak değişimi için `<value>` değerini JSON olarak ayrıştırır. |
| `--dry-run`     | `set`                            | Yazma işlemi yapmadan yazılacak baytları yazdırır.                        |
| `--diff`        | `set` (`--dry-run` gerektirir) | Tam baytlar yerine birleşik fark çıktısı yazdırır.                        |

`validate` yalnızca `--json` / `--human` alır; dosya sistemine erişmez, bu nedenle
`--cwd` ve `--file` geçerli değildir.

## `oc://` söz dizimi

```text
oc://FILE/SECTION/ITEM/FIELD?session=SCOPE
```

Yuva kuralları: `field`, `item` gerektirir; `item` ise `section` gerektirir. Dört
yuvanın tamamında:

- **Tırnak içindeki segmentler** — `"a/b.c"`, `/` ve `.` ayırıcılarından etkilenmez. İçerik
  bayt düzeyinde değişmezdir; tırnakların içinde `"` ve `\` kullanılamaz. Dosya yuvası da
  tırnakları dikkate alır: `oc://"skills/email-drafter"/Tools/$last`,
  `skills/email-drafter` değerini tek bir dosya yolu olarak ele alır.
- **Koşullar** — `[k=v]`, `[k!=v]`, `[k<v]`, `[k<=v]`, `[k>v]`, `[k>=v]`.
  Sayısal operatörler, her iki tarafın da sonlu sayılara dönüştürülebilmesini gerektirir.
- **Birleşimler** — `{a,b,c}`, alternatiflerden herhangi biriyle eşleşir.
- **Joker karakterler** — `*` (tek alt segment) ve `**` (sıfır veya daha fazla,
  özyinelemeli). `find` bunları kabul eder; `resolve` ve `set` belirsiz oldukları için
  reddeder.
- **Konumsal** — `$first` / `$last`, ilk / son indekse veya
  bildirilmiş anahtara çözümlenir.
- **Sıra belirteci** — Belge sırasındaki N'inci eşleşme için `#N`.
- **Ekleme işaretçileri** — Anahtarlı / indeksli ekleme için `+`, `+key`, `+nnn`
  (`set` ile kullanın).
- **Oturum kapsamı** — `?session=cron-daily` vb. Yuva iç içeliğinden bağımsızdır.
  Oturum değerleri hamdır, yüzde kodlaması çözülmez; denetim
  karakterlerini veya ayrılmış sorgu sınırlayıcılarını (`?`, `&`, `%`) içeremez.

Tırnak içindeki, koşul veya birleşim segmentleri dışındaki ayrılmış karakterler
(`?`, `&`, `%`) reddedilir. Denetim karakterleri (U+0000-U+001F, U+007F),
`session` sorgu değeri dâhil her yerde reddedilir.

Standart yollar için `formatOcPath(parseOcPath(path)) === path` garanti edilir.
Standart olmayan sorgu parametreleri, ilk boş olmayan `session=`
değeri dışında yok sayılır.

Kesin sınırlar: bir yol en fazla 4096 bayt, en fazla 4 yuva (dosya/bölüm/öğe/
alan), yuva başına en fazla 64 noktayla ayrılmış alt segment ve derin JSON yolları için en fazla 256 iç içe
gezinme düzeyi içerebilir. Ayrıca, 16 MiB üzerindeki tüm JSONC/JSON dosya girdileri,
bu dosyayı yükleyen tüm fiillerde ayrıştırılmak yerine bir ayrıştırma tanılamasıyla
reddedilir.

## Dosya türüne göre adresleme

| Tür           | Dosya uzantıları             | Adresleme modeli                                                                                       |
| ------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------ |
| Markdown      | `.md`                       | Slug ile H2 bölümleri, slug veya `#N` ile madde işaretli öğeler, `[frontmatter]` üzerinden frontmatter. |
| JSONC/JSON    | `.jsonc`, `.json`           | Nesne anahtarları ve dizi indeksleri; tırnak içine alınmadıkça noktalar iç içe alt segmentleri ayırır. |
| JSONL         | `.jsonl`, `.ndjson`         | Üst düzey satır adresleri (`L1`, `L2`, `$first`, `$last`), ardından satır içinde JSONC tarzı iniş. |
| YAML/.lobster | `.yaml`, `.yml`, `.lobster` | Eşleme anahtarları ve dizi indeksleri; yorumlar ve akış stili YAML belge API'si tarafından işlenir. |

`resolve`, 1 tabanlı bir satır numarasıyla yapılandırılmış bir eşleşme döndürür:
`root`, `node`, `leaf` veya
`insertion-point`. Yaprak değerleri, plugin yazarlarının dosya türüne özgü AST biçimine
bağımlı olmadan önizlemeler oluşturabilmesi için metin ve bir `leafType` olarak sunulur.

## Değişiklik sözleşmesi

`set` tek bir somut hedef yazar:

- Markdown frontmatter değerleri ve `- key: value` öğe alanları dize
  yapraklarıdır. Markdown eklemeleri bölümler, frontmatter anahtarları veya bölüm
  öğeleri ekler ve değiştirilen dosya için standart bir markdown biçimi oluşturur. Bölüm
  gövdelerinin tamamı `set` aracılığıyla yazılamaz.
- JSONC yaprak yazmaları, dize değerini mevcut yaprak türüne dönüştürür
  (`string`, sonlu `number`, `true`/`false` veya `null`). Bir JSONC/JSON/JSONL yaprak değiştirme işleminin `<value>` değerini JSON olarak ayrıştırması ve
  bir dize secret-ref kısaltmasını bir nesneyle değiştirmek gibi durumlarda biçimi
  değiştirebilmesi gerektiğinde `--value-json` kullanın. JSONC nesne ve dizi eklemeleri `<value>` değerini JSON olarak ayrıştırır ve sıradan yaprak yazmaları için
  `jsonc-parser` düzenleme yolunu kullanarak yorumları
  ve yakındaki biçimlendirmeyi korur.
- JSONL yaprak yazmaları, bir satır içinde JSONC gibi dönüşüm yapar. Tam satır değiştirme
  ve sona ekleme işlemleri `<value>` değerini JSON olarak ayrıştırır. Oluşturulan JSONL, dosyanın
  baskın LF/CRLF satır sonu kuralını korur (dosyadaki
  satır sonları genelinde çoğunluk oylaması yapılır; böylece çoğunlukla CRLF kullanan bir dosya, birkaç aykırı LF içerse bile CRLF olarak kalır).
- YAML yaprak yazmaları, mevcut skaler türe dönüştürülür (`string`, sonlu
  `number`, `true`/`false` veya `null`). YAML eklemeleri, eşleme/dizi güncellemeleri için paketle gelen
  `yaml` paketinin belge API'sini kullanır. Ayrıştırıcı hataları içeren bozuk YAML
  belgelerinde, değişiklik yapılmadan önce
  `parse-error` ile işlem reddedilir.

Tam baytlar önemli olduğunda kullanıcıya görünür yazma işlemlerinden önce `--dry-run` kullanın. JSONC
ve YAML düzenlemeleri mevcut belgeye yama uygular (`jsonc-parser` veya `yaml`
belge API'si aracılığıyla); bu nedenle dokunulmayan baytlar genellikle korunur. Markdown ise herhangi bir düzenlemede dosyayı
ayrıştırılmış yapısından yeniden oluşturur; bu da değiştirilen yaprağın dışındaki ikincil
biçimlendirmeyi normalleştirebilir. Önizlemeyi oluşturulan dosyanın tamamı yerine odaklanmış
bir önce/sonra yaması olarak görmek istediğinizde `--diff` ekleyin.

## Örnekler

```bash
# Bir yolu doğrula (dosya sistemi erişimi yok)
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk'

# Bir yaprağı oku
openclaw path resolve 'oc://gateway.jsonc/version'

# Joker karakterli arama
openclaw path find 'oc://session.jsonl/*/event' --file ./logs/session.jsonl

# Bir yazma işlemini deneme amaçlı çalıştır
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run

# Bir yazma işlemini birleşik diff olarak deneme amaçlı çalıştır
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff

# Yazma işlemini uygula
openclaw path set 'oc://gateway.jsonc/version' '2.0'

# Bayt doğruluğunda gidiş dönüş (tanılama)
openclaw path emit ./AGENTS.md
```

Diğer dil bilgisi örnekleri:

```bash
# / veya . içeren anahtarları tırnak içine al
openclaw path resolve 'oc://config.jsonc/agents.defaults.models/"anthropic/claude-opus-4-7"/alias'

# Derin JSON/JSONC yolları eğik çizgili segmentler kullanabilir; bunlar noktalı alt segmentlere normalleştirilir
openclaw path set 'oc://openclaw.json/agents/list/0/tools/exec/security' 'allowlist' --dry-run

# Bir JSONC yaprağını ayrıştırılmış bir nesneyle değiştir
openclaw path set 'oc://openclaw.json/gateway/auth/token' '{"source":"file","provider":"secrets","id":"/test"}' --value-json --dry-run

# JSONC alt öğeleri üzerinde koşul araması
openclaw path find 'oc://config.jsonc/plugins/[enabled=true]/id'

# Bir JSONC dizisine ekle
openclaw path set 'oc://config.jsonc/items/+1' '{"id":"new","enabled":true}' --dry-run

# Bir JSONC nesne anahtarı ekle
openclaw path set 'oc://config.jsonc/plugins/+github' '{"enabled":true}' --dry-run

# Bir JSONL olayını sona ekle
openclaw path set 'oc://session.jsonl/+' '{"event":"checkpoint","ok":true}' --file ./logs/session.jsonl

# Son JSONL değer satırını çözümle
openclaw path resolve 'oc://session.jsonl/$last/event' --file ./logs/session.jsonl

# Bir YAML iş akışı adımını çözümle
openclaw path resolve 'oc://workflow.yaml/steps/0/id'

# Bir YAML skalerini güncelle
openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --dry-run

# Markdown frontmatter'ını adresle
openclaw path resolve 'oc://AGENTS.md/[frontmatter]/name'

# Markdown frontmatter'ı ekle
openclaw path set 'oc://AGENTS.md/[frontmatter]/+description' 'Agent instructions' --dry-run

# Markdown öğe alanlarını bul
openclaw path find 'oc://SKILL.md/Tools/*/send_email'

# Oturum kapsamlı bir yolu doğrula
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk?session=cron-daily'
```

## Dosya türüne göre tarifler

Aynı beş fiil tüm türlerde çalışır; adresleme düzeni dosya uzantısına göre
yönlendirme yapar.

### Markdown

```text
<!-- frontmatter.md -->
---
name: taslakçı
description: e-posta taslağı hazırlayan agent
tier: çekirdek
---
## Araçlar
- gh: GitHub CLI
- curl: HTTP istemcisi
- send_email: etkin
```

```bash
$ openclaw path resolve 'oc://x.md/[frontmatter]/tier' --file frontmatter.md --human
yaprak @ L4: "core" (dize)

$ openclaw path resolve 'oc://x.md/tools/gh/gh' --file frontmatter.md --human
yaprak @ L9: "GitHub CLI" (dize)

$ openclaw path find 'oc://x.md/tools/*' --file frontmatter.md --human
oc://x.md/tools/* için 3 eşleşme:
  oc://x.md/tools/gh           →  düğüm @ L9 [md-item]
  oc://x.md/tools/curl         →  düğüm @ L10 [md-item]
  oc://x.md/tools/send-email   →  düğüm @ L11 [md-item]
```

`[frontmatter]` koşulu YAML frontmatter bloğunu adresler; `tools`
slug aracılığıyla `## Tools` başlığıyla eşleşir ve kaynakta alt çizgi kullanılsa bile öğe yaprakları slug biçimini
korur (`send_email`, `send-email` olur).

### JSONC

```text
// config.jsonc
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": false, "role": "chat"}
  }
}
```

```bash
$ openclaw path resolve 'oc://config.jsonc/plugins/github/enabled' --file config.jsonc --human
yaprak @ L4: "true" (boolean)

$ openclaw path set 'oc://config.jsonc/plugins/slack/enabled' 'true' --file config.jsonc --dry-run
--dry-run: /…/config.jsonc konumuna 142 bayt yazılacaktı
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": true, "role": "chat"}
  }
}
```

JSONC düzenlemeleri `jsonc-parser` üzerinden gerçekleştirilir; böylece yorumlar ve boşluklar bir
`set` işleminden sonra korunur. Değişiklikleri uygulamadan önce baytları incelemek için ilk olarak `--dry-run` ile çalıştırın.
`.json` dosyaları, `.jsonc` ile aynı adaptörü ve düzenleme yolunu kullanır.

### JSONL

```text
{"event":"start","userId":"u1","ts":1}
{"event":"action","userId":"u1","ts":2}
{"event":"end","userId":"u1","ts":3}
```

```bash
$ openclaw path find 'oc://session.jsonl/[event=action]/userId' --file session.jsonl --human
oc://session.jsonl/[event=action]/userId için 1 eşleşme:
  oc://session.jsonl/L2/userId  →  yaprak @ L2: "u1" (dize)

$ openclaw path resolve 'oc://session.jsonl/L2/ts' --file session.jsonl --human
yaprak @ L2: "2" (sayı)
```

Her satır bir kayıttır. Satır numarasını bilmiyorsanız koşula göre (`[event=action]`),
biliyorsanız standart `LN` segmentine göre adresleyin.
`.ndjson` dosyaları, `.jsonl` ile aynı adaptörü kullanır.

### YAML

```text
# workflow.yaml
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify
    command: openclaw.invoke
```

```bash
$ openclaw path resolve 'oc://workflow.yaml/steps/0/id' --file workflow.yaml --human
yaprak @ L3: "fetch" (dize)

$ openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --file workflow.yaml --dry-run
--dry-run: /…/workflow.yaml konumuna 99 bayt yazılacaktı
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify-renamed
    command: openclaw.invoke
```

YAML, elle yazılmış bir ayrıştırıcı yerine `yaml` paketinin `Document` API'sini
kullanır; bu nedenle sıradan ayrıştırma/oluşturma gidiş dönüşleri yorumları ve yazım
biçimini korurken çözümlenen yollar JSONC ile aynı eşleme anahtarı / dizi dizini modelini
kullanır. Aynı adaptör `.yaml`, `.yml` ve `.lobster` dosyalarını işler.

## Alt komut başvurusu

### `resolve <oc-path>`

Tek bir yaprağı veya düğümü okuyun. Joker karakterler reddedilir; bunlar için `find` kullanın.
Bir eşleşmede `0`, temiz bir eşleşmeme durumunda `1`, ayrıştırma hatasında veya reddedilen
desende `2` koduyla çıkar.

```bash
openclaw path resolve 'oc://AGENTS.md/tools/gh/risk' --human
openclaw path resolve 'oc://gateway.jsonc/server/port' --json
```

### `find <pattern>`

Joker karakter / koşul / birleşim deseni için her eşleşmeyi listeleyin. En az bir eşleşmede `0`,
sıfır eşleşmede `1` koduyla çıkar. Dosya yuvası joker karakterleri
`OC_PATH_FILE_WILDCARD_UNSUPPORTED` ile reddedilir; somut bir dosya belirtin (birden fazla dosyada
glob kullanımı sonraki bir özelliktir).

```bash
openclaw path find 'oc://AGENTS.md/tools/**/risk'
openclaw path find 'oc://session.jsonl/[event=action]/userId'
openclaw path find 'oc://config.jsonc/plugins/{github,slack}/enabled'
```

### `set <oc-path> <value>`

Bir yaprak yazın. Dosyaya dokunmadan yazılacak baytları önizlemek için `--dry-run` ile birlikte kullanın.
Birleşik diff önizlemesi için `--diff` ekleyin.
Başarılı bir yazmada `0`, altyapı reddederse (örneğin bir
sentinel koruması tetiklenirse) `1`, ayrıştırma hatalarında `2` koduyla çıkar.

```bash
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff
openclaw path set 'oc://gateway.jsonc/version' '2.0'
openclaw path set 'oc://AGENTS.md/Tools/+gh/risk' 'low'
```

`+key` ekleme işareti, adlandırılmış alt öğe zaten mevcut değilse onu oluşturur;
`+nnn` ve yalın `+` sırasıyla dizinli ekleme ve sona ekleme
için çalışır.

### `validate <oc-path>`

Yalnızca ayrıştırma denetimi. Dosya sistemi erişimi yoktur. Değişkenleri yerleştirmeden önce bir
şablon yolunun doğru biçimlendirildiğini doğrulamak veya hata ayıklama için
yapısal dökümü görmek istediğinizde kullanışlıdır:

```bash
$ openclaw path validate 'oc://AGENTS.md/tools/gh' --human
geçerli: oc://AGENTS.md/tools/gh
  dosya:   AGENTS.md
  bölüm:   tools
  öğe:     gh
```

Geçerliyse `0`, geçersizse (yapılandırılmış bir `code` ve
`message` ile) `1`, bağımsız değişken hatalarında `2` koduyla çıkar.

### `emit <file>`

Bir dosyayı türüne özgü ayrıştırıcı ve oluşturucudan geçirerek gidiş dönüş yapın. Sağlam bir dosyada çıktı
girdiyle bayt düzeyinde özdeş olmalıdır; farklılık, bir
ayrıştırıcı hatasına veya sentinel tetiklenmesine işaret eder. Gerçek dünya girdilerinde altyapı davranışını
hata ayıklamak için kullanışlıdır.

```bash
openclaw path emit ./AGENTS.md
openclaw path emit ./gateway.jsonc --json
```

## Çıkış kodları

| Kod | Anlam                                                                      |
| ---- | -------------------------------------------------------------------------- |
| `0`  | Başarılı. (`resolve` / `find`: en az bir eşleşme. `set`: yazma başarılı.) |
| `1`  | Eşleşme yok veya `set` altyapı tarafından reddedildi (sistem düzeyinde hata yok). |
| `2`  | Bağımsız değişken veya ayrıştırma hatası.                                   |

## Çıktı modu

`openclaw path` TTY'yi algılar: terminalde insan tarafından okunabilir çıktı, stdout bir kanala aktarılır veya
yönlendirilirse JSON üretir. `--json` ve `--human`
otomatik algılamayı geçersiz kılar.

## Notlar

- `set`, baytları altyapının emit yolu üzerinden yazar; bu yol
  redaksiyon belirteci korumasını otomatik olarak uygular. `__OPENCLAW_REDACTED__`
  (aynen veya bir alt dize olarak) taşıyan bir yaprak, yazma
  sırasında reddedilir.
- JSONC ayrıştırması ve yaprak düzenlemeleri, plugin'e özgü `jsonc-parser`
  bağımlılığını kullanır; böylece sıradan yaprak yazmalarında yorumlar ve
  biçimlendirme korunur ve elle yazılmış bir ayrıştırıcı/yeniden işleme yolu
  kullanılmaz.
- `path`, bilinen son iyi (LKG) yapılandırma izleme veya kurtarma işlemlerinden haberdar değildir;
  bu yaşam döngüsünün sahipliği başka yerdedir. `path` aracılığıyla düzenlediğiniz bir dosya
  aynı zamanda LKG ile izleniyorsa, sonraki yapılandırma okuması dosyanın yükseltilmesine mi
  yoksa kurtarılmasına mı karar verir; `path` düzenlemesini, o dosyaya yapılan
  diğer tüm doğrudan yazmalarla aynı şekilde ele alın.

## İlgili

- [CLI referansı](/tr/cli)
