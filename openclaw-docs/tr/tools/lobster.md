---
read_when:
    - Açık onaylara sahip deterministik çok adımlı iş akışları istiyorsunuz
    - Önceki adımları yeniden çalıştırmadan bir iş akışını sürdürmeniz gerekiyor
summary: Devam ettirilebilir onay geçitlerine sahip OpenClaw için tür belirtilmiş iş akışı çalışma zamanı.
title: Lobster
x-i18n:
    generated_at: "2026-07-27T00:08:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85b7900f86bfedc9d73fcc91c3d0dac37b81f7413b1e68c54dd8a797b70f79fc
    source_path: tools/lobster.md
    workflow: 16
---

Lobster, çok adımlı araç işlem hatlarını açık onay denetim noktaları ve
sürdürme belirteçleriyle tek bir deterministik araç çağrısı olarak çalıştırır. Ayrılmış
arka plan işlerinin bir katman üzerinde yer alır: birçok ayrılmış görev arasındaki akışları
düzenlemek için bkz. [Task Flow](/tr/automation/taskflow) (`openclaw tasks flow`); görev
etkinliği defteri için bkz. [Arka Plan Görevleri](/tr/automation/tasks).

## Neden

Lobster olmadan çok adımlı bir iş, modelin her adımı düzenlediği çok sayıda
gidiş-dönüş araç çağrısı anlamına gelir. Lobster bu düzenlemeyi türü belirlenmiş bir
çalışma zamanına taşır:

- **Birçok çağrı yerine tek çağrı**: tek bir Lobster araç çağrısı, tüm işlem
  hattı için yapılandırılmış bir sonuç döndürür.
- **Yerleşik onaylar**: yan etkiler (gönderme, yayımlama, silme), açıkça
  onaylanana kadar iş akışını durdurur.
- **Sürdürülebilir**: durdurulan bir iş akışı bir belirteç döndürür; önceki
  adımları yeniden çalıştırmadan onaylayıp sürdürün.

Lobster, genel amaçlı bir betik dili yerine küçük ve kısıtlı bir DSL'dir:
onaylama/sürdürme kalıcı, yerleşik bir temel öğedir; işlem hatları veridir (günlüklemek,
farklarını almak, yeniden oynatmak ve incelemek kolaydır); küçük dil bilgisi, doğrulamanın
gerçekçi kalması için "yaratıcı" kod yollarını sınırlar; zaman aşımları, çıktı sınırları,
korumalı alan denetimleri ve izin listeleri her betik tarafından değil, çalışma zamanı
tarafından uygulanır. Her adım yine de herhangi bir CLI'ı veya betiği çağırabilir; daha
zengin bir yazım dili istiyorsanız diğer araçlardan `.lobster` dosyaları oluşturun.

Lobster olmadan yinelenen bir e-posta önceliklendirmesi şöyle görünür:

```text
Kullanıcı: "E-postamı kontrol et ve yanıt taslakları hazırla"
→ openclaw, gmail.list'i çağırır
→ LLM özetler
→ Kullanıcı: "#2 ve #5 için yanıt taslakları hazırla"
→ LLM taslakları hazırlar
→ Kullanıcı: "#2'yi gönder"
→ openclaw, gmail.send'i çağırır
(her gün tekrarlanır, önceliklendirilenlerin kaydı tutulmaz)
```

Lobster ile aynı iş, onay için duran ve ardından sürdürülen tek bir çağrıdır:

```json
{ "action": "run", "pipeline": "email.triage --limit 20", "timeoutMs": 30000 }
```

```json
{
  "ok": true,
  "status": "needs_approval",
  "output": [{ "summary": "5 need replies, 2 need action" }],
  "requiresApproval": {
    "type": "approval_request",
    "prompt": "Send 2 draft replies?",
    "items": [],
    "resumeToken": "..."
  }
}
```

## Nasıl çalışır?

OpenClaw, Lobster iş akışlarını gömülü bir çalıştırıcı olarak paketlenmiş
`@clawdbot/lobster` paketini kullanarak **işlem içinde** çalıştırır. Harici bir `lobster`
alt işlemi başlatılmaz; araç çağrısı doğrudan bir JSON zarfı döndürür. İşlem hattı
onay için durursa zarf, daha sonra devam edebilmeniz için bir sürdürme belirteci
(veya kısa bir onay kimliği) taşır.

## Etkinleştirme

Lobster, varsayılan olarak etkinleştirilmeyen **isteğe bağlı** bir Plugin aracıdır. Paketlenmiş
olarak gönderildiğinden ayrı bir kurulum adımı gerekmez; yalnızca araca izin verin:

```json
{
  "tools": {
    "alsoAllow": ["lobster"]
  }
}
```

Veya aracı bazında:

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": {
          "alsoAllow": ["lobster"]
        }
      }
    ]
  }
}
```

<Note>
`alsoAllow`, diğer çekirdek araçları kısıtlamadan etkin araç profilinin üzerine
`lobster` ekler. Bunun yerine yalnızca kısıtlayıcı bir izin listesi modu
istiyorsanız `tools.allow` kullanın.
</Note>

Araç, korumalı alan içindeki araç bağlamlarında tamamen devre dışıdır.

Geliştirme veya harici işlem hatları için (gömülü gateway çalıştırıcısının dışında)
bağımsız Lobster CLI'a ihtiyacınız varsa [Lobster deposundan](https://github.com/openclaw/lobster)
kurun ve `lobster` öğesini `PATH` üzerine koyun.

## Kalıp: küçük CLI + JSON kanalları + onaylar

JSON ile iletişim kuran küçük komutlar oluşturun, ardından bunları tek bir Lobster
çağrısında zincirleyin. (Aşağıdaki örnek komut adlarını kendi komutlarınızla değiştirin.)

```bash
inbox list --json
inbox categorize --json
inbox apply --json
```

```json
{
  "action": "run",
  "pipeline": "exec --json --shell 'inbox list --json' | exec --stdin json --shell 'inbox categorize --json' | exec --stdin json --shell 'inbox apply --json' | approve --preview-from-stdin --limit 5 --prompt 'Apply changes?'",
  "timeoutMs": 30000
}
```

İşlem hattı onay isterse belirteçle sürdürün:

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

Örnek: girdi öğelerini araç çağrılarına eşleyin:

```bash
gog.gmail.search --query 'newer_than:1d' \
  | openclaw.invoke --tool message --action send --each --item-key message --args-json '{"provider":"telegram","to":"..."}'
```

## Yalnızca JSON kullanan LLM adımları (llm-task)

Bir iş akışında **yapılandırılmış bir LLM adımı** için isteğe bağlı
`llm-task` Plugin aracını etkinleştirin ve Lobster'dan çağırın:

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": { "alsoAllow": ["llm-task"] }
      }
    ]
  }
}
```

### Önemli sınırlama: gömülü Lobster ile `openclaw.invoke`

Paketlenmiş Lobster Plugin'i, iş akışlarını gateway içinde **işlem içinde** çalıştırır.
Bu gömülü modda `openclaw.invoke`, iç içe OpenClaw CLI araç çağrıları için bir
gateway URL'sini/kimlik doğrulama bağlamını otomatik olarak devralmaz.

Bu, aşağıdaki kalıbın **şu anda gömülü çalıştırıcıda güvenilir olmadığı** anlamına gelir:

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{ ... }'
```

Aşağıdaki örneği yalnızca `openclaw.invoke` öğesinin doğru gateway/kimlik doğrulama
bağlamıyla zaten yapılandırıldığı bir ortamda **bağımsız Lobster CLI** çalıştırırken kullanın.

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "Verilen girdi e-postasına göre amacı ve taslağı döndür.",
  "thinking": "low",
  "input": { "subject": "Merhaba", "body": "Yardım edebilir misiniz?" },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

Bugün gömülü Lobster Plugin'ini kullanıyorsanız şunlardan birini tercih edin:

- Lobster dışında doğrudan bir `llm-task` araç çağrısı veya
- desteklenen bir gömülü köprü eklenene kadar Lobster işlem hattında
  `openclaw.invoke` olmayan adımlar.

Ayrıntılar ve yapılandırma seçenekleri için bkz. [LLM Görevi](/tr/tools/llm-task).

## İş akışı dosyaları (.lobster)

Lobster; `name`, `args`, `steps`, `env`,
`condition` ve `approval` alanlarına sahip YAML/JSON iş akışı dosyalarını çalıştırabilir.
Araç çağrısında `pipeline` değerini dosya yolu olarak ayarlayın.

```yaml
name: inbox-triage
args:
  tag:
    default: "family"
steps:
  - id: collect
    command: inbox list --json
  - id: categorize
    command: inbox categorize --json
    stdin: $collect.stdout
  - id: approve
    command: inbox apply --approve
    stdin: $categorize.stdout
    approval: required
  - id: execute
    command: inbox apply --execute
    stdin: $categorize.stdout
    condition: $approve.approved
```

Notlar:

- `stdin: $step.stdout` ve `stdin: $step.json`, önceki bir adımın çıktısını aktarır.
- `condition` (veya `when`), adımları `$step.approved` temelinde koşullandırabilir.

### Eklenen ortam değişkenleri

Her adımın kabuğu, üst ortamın yanı sıra Lobster tarafından eklenen şu değişkenleri
devralır; böylece komutlar, ham değerleri komut dizesine gömmeden çözümlenmiş iş akışı
bağımsız değişkenlerine başvurabilir:

- `LOBSTER_ARG_<NAME>` - iş akışı bağımsız değişkeni başına bir tane. Ad, alfasayısal
  olmayan ardışık karakterlerin her biri `_` biçimine daraltılarak büyük harfe
  dönüştürülür; böylece `user-id` bağımsız değişkeni `LOBSTER_ARG_USER_ID` olur.
- `LOBSTER_ARGS_JSON` - çözümlenmiş tüm bağımsız değişkenler tek bir JSON dizesi olarak.

Eklenen kümenin tamamı budur. `LOBSTER_STEP_<id>_STDOUT` veya `LOBSTER_STEP_<id>_JSON_<field>` gibi
adım başına çıktı değişkenleri **yoktur**; kabuklar bu adları ayarlanmamış olarak
değerlendirdiğinden parametre genişletme varsayılanları hatayı gizleyebilir.
Bunun yerine önceki bir adımın çıktısını `stdin:`, `env:` veya
`condition:` değerindeki `$step.stdout`, `$step.json` ya da
`$step.json.<field>` gibi adım başvuruları aracılığıyla okuyun. (`LOBSTER_STATE_DIR`,
durum dizini için ayrı bir çalışma zamanı ayarıdır; çalıştırma başına bağımsız değişken değildir.)

## Araç parametreleri

### `run`

```json
{
  "action": "run",
  "pipeline": "gog.gmail.search --query 'newer_than:1d' | email.triage",
  "cwd": "workspace",
  "timeoutMs": 30000,
  "maxStdoutBytes": 512000
}
```

Bir iş akışı dosyasını bağımsız değişkenlerle çalıştırın:

```json
{
  "action": "run",
  "pipeline": "/path/to/inbox-triage.lobster",
  "argsJson": "{\"tag\":\"family\"}"
}
```

| Alan             | Varsayılan  | Notlar                                                                                                       |
| ---------------- | ----------- | ------------------------------------------------------------------------------------------------------------ |
| `pipeline`       | gerekli     | Satır içi işlem hattı dizesi veya iş akışı dosyası için `.lobster`/`.yaml`/`.yml`/`.json` ile biten bir yol. |
| `cwd`            | gateway cwd | Göreli çalışma dizini; gateway çalışma dizini içinde çözümlenmelidir (mutlak yollar reddedilir).             |
| `timeoutMs`      | `20000`     | Aşılırsa çalıştırmayı iptal eder.                                                                             |
| `maxStdoutBytes` | `512000`    | Yakalanan stdout veya stderr bu boyutu aşarsa çalıştırmayı iptal eder.                                        |
| `argsJson`       | -           | Bir iş akışı dosyasının bağımsız değişkenlerini içeren JSON dizesi (satır içi işlem hatlarında yok sayılır).  |

### `resume`

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

`resume`, `token` (`requiresApproval` içindeki tam sürdürme belirteci)
veya `approvalId` (aynı nesnedeki kısa kimlik) değerlerinden birini kabul eder;
durdurulan çalıştırma hangisini döndürdüyse onu kullanın. `approve` gereklidir.

### Yönetilen Task Flow modu

`run` üzerinde `flowControllerId` ve `flowGoal` (veya
`resume` üzerinde `flowId` ve `flowExpectedRevision`) aktarılması,
çağrının yalın bir zarf döndürmesi yerine Plugin çalışma zamanının yönetilen
[Task Flow](/tr/automation/taskflow) API'si üzerinden yürütülmesini sağlar: OpenClaw,
kalıcı bir akış kaydı oluşturur veya sürdürür, Lobster zarfını bu kayda uygular
(onay sırasında `waiting`, tamamlanma sırasında `succeeded`/`failed`)
ve `{ ok, envelope, flow, mutation }` döndürür. Bu mod, bağlı bir Task Flow çalışma zamanı gerektirir
ve tipik geçici aracı kullanımı için değil, gateway yeniden başlatmaları boyunca
kalıcı akış durumuna ihtiyaç duyan Plugin/denetleyici kodu için tasarlanmıştır.

## Çıktı zarfı

Lobster, üç durumdan birini içeren bir JSON zarfı döndürür:

- `ok` - başarıyla tamamlandı
- `needs_approval` - duraklatıldı; `requiresApproval`, çalıştırmayı
  sürdürebilecek bir `resumeToken` ve kısa bir `approvalId` taşır
- `cancelled` - açıkça reddedildi veya iptal edildi

Araç, zarfı hem `content` (biçimlendirilmiş JSON) hem de `details`
(ham nesne) içinde sunar.

## Onaylar

`requiresApproval` varsa istemi inceleyip karar verin:

- `approve: true` - sürdür ve yan etkilere devam et
- `approve: false` - iş akışını iptal et ve sonlandır

Özel jq/heredoc bağlayıcıları olmadan onay isteklerine bir JSON önizlemesi eklemek için
`approve --preview-from-stdin --limit N` kullanın. Sürdürme durumu, Lobster durum dizininde (varsayılan olarak
`~/.lobster/state`; `LOBSTER_STATE_DIR` ile geçersiz kılınabilir) küçük JSON dosyaları
olarak saklanır; belirtecin kendisi tam işlem hattı durumunu değil, yalnızca bu duruma
işaret eden bir göstericiyi kodlar.

## OpenProse

OpenProse, Lobster ile iyi bir uyum sağlar: çok aracılı hazırlığı düzenlemek için
`/prose` kullanın, ardından deterministik onaylar için bir Lobster işlem
hattı çalıştırın. Bir Prose programı Lobster'a ihtiyaç duyuyorsa `tools.subagents.tools`
aracılığıyla alt aracılar için `lobster` aracına izin verin.
Bkz. [OpenProse](/tr/prose).

## Güvenlik

- **Yalnızca süreç içi yerel çalışma** - iş akışları Gateway süreci içinde yürütülür; Plugin'in kendisinden
  ağ çağrısı yapılmaz.
- **Gizli bilgi yok** - Lobster, OAuth'u yönetmez; bunu yapan OpenClaw araçlarını
  çağırır.
- **Korumalı alan duyarlı** - araç bağlamı korumalı alandayken devre dışı bırakılır.
- **Güçlendirilmiş** - zaman aşımları ve çıktı sınırları, gömülü çalıştırıcı tarafından uygulanır.

## Sorun giderme

| Hata                                                          | Neden / çözüm                                                                    |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `lobster runtime timed out`                                   | İşlem hattı `timeoutMs` değerini aştı. Bu değeri artırın veya işlem hattını bölün. |
| `lobster stdout exceeded maxStdoutBytes` (veya `stderr`)        | Yakalanan çıktı sınırı aştı. `maxStdoutBytes` değerini artırın veya çıktıyı azaltın. |
| `run --args-json must be valid JSON`                          | `argsJson` (iş akışı dosyası çalıştırmaları) ayrıştırılamadı. JSON dizesini düzeltin. |
| `lobster runtime failed` (veya başka bir `runtime_error` iletisi) | Gömülü çalışma zamanı bir hata zarfı döndürdü. Ayrıntılar için Gateway günlüklerini kontrol edin. |

## Daha fazla bilgi

- [Pluginler](/tr/tools/plugin)
- [Plugin aracı yazma](/tr/plugins/building-plugins#registering-agent-tools)

## Vaka çalışması: topluluk iş akışları

Herkese açık bir örnek: üç Markdown kasasını (kişisel, eşe ait, paylaşılan) yöneten bir "ikinci beyin" CLI'sı + Lobster işlem hatları. CLI; istatistikler,
gelen kutusu listeleri ve eskimiş öğe taramaları için JSON çıktısı üretir; Lobster bu komutları
`weekly-review`, `inbox-triage`, `memory-consolidation` ve
`shared-task-sync` gibi, her biri onay kapıları içeren iş akışlarında zincirler. Yapay zekâ, kullanılabilir olduğunda
değerlendirme (sınıflandırma) işlemini gerçekleştirir; kullanılamadığında ise deterministik kurallara
geri döner.

- Konu: [https://x.com/plattenschieber/status/2014508656335770033](https://x.com/plattenschieber/status/2014508656335770033)
- Depo: [https://github.com/bloomedai/brain-cli](https://github.com/bloomedai/brain-cli)

## İlgili

- [Otomasyon](/tr/automation) - tüm otomasyon mekanizmaları
- [Araçlara Genel Bakış](/tr/tools) - kullanılabilir tüm ajan araçları
