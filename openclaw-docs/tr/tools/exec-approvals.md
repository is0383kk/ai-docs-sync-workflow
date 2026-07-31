---
read_when:
    - exec onaylarını veya izin listelerini yapılandırma
    - macOS uygulamasında exec onayı kullanıcı deneyiminin uygulanması
    - Sandbox'tan kaçış istemlerini ve bunların etkilerini inceleme
sidebarTitle: Exec approvals
summary: 'Ana makinede komut yürütme onayları: ilke ayarları, izin listeleri ve YOLO/katı iş akışı'
title: Çalıştırma onayları
x-i18n:
    generated_at: "2026-07-26T23:04:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2bd09746375061232e9094b8803d33859cac4c13c7bde14a059b7d52e48b5de8
    source_path: tools/exec-approvals.md
    workflow: 16
---

Exec onayları, korumalı alandaki bir aracının gerçek bir ana makinede (`gateway` veya `node`) komut çalıştırmasına izin veren **yardımcı uygulama / Node ana makinesi koruma mekanizmasıdır**. Komutlar
yalnızca politika + izin listesi + (isteğe bağlı) kullanıcı onayının tümü uyumlu olduğunda
çalıştırılır. Onaylar, araç politikasının ve yükseltilmiş erişim denetiminin **üzerine eklenir** (yükseltilmiş
`full` bunları atlar).

`deny`, `allowlist`, `ask`, `auto`, `full`,
Codex Guardian eşlemesi ve ACPX çalıştırma ortamı izinlarına ilişkin mod odaklı genel bakış için
[İzin modları](/tr/tools/permission-modes) bölümüne bakın.

<Note>
Geçerli politika, `tools.exec.*` ile onay
varsayılanlarından **daha katı** olanıdır: onaylar yalnızca yapılandırmadan türetilen güvenliği/onay istemini
sıkılaştırabilir, asla gevşetemez. Bir onay alanı belirtilmezse
`tools.exec` değeri kullanılır. Ana makinede Exec ayrıca o makinedeki yerel onay durumunu kullanır;
oturum veya yapılandırma varsayılanları `ask: "on-miss"` istese bile yürütme ana makinesinin onay dosyasındaki
ana makineye özgü `ask: "always"` onay istemeye devam eder.
</Note>

## Uygulandığı yerler

Exec onayları yürütme ana makinesinde yerel olarak uygulanır:

- **Gateway ana makinesi** -> Gateway makinesindeki `openclaw` işlemi.
- **Node ana makinesi** -> Node çalıştırıcısı (macOS yardımcı uygulaması veya başsız Node ana makinesi).

### Güven modeli

- Gateway kimliği doğrulanmış çağrıcılar, söz konusu Gateway için güvenilir operatörlerdir.
- Eşleştirilmiş Node'lar, bu güvenilir operatör yeteneğini Node ana makinesine genişletir.
- Onaylar yanlışlıkla yürütme riskini azaltır ancak kullanıcı başına bir kimlik doğrulama sınırı veya salt okunur dosya sistemi politikası **değildir**.
- Onaylanan bir komut, seçilen ana makine veya korumalı alan dosya sistemi izinleri kapsamında dosyaları değiştirebilir.
- Onaylanan Node ana makinesi çalıştırmaları standart yürütme bağlamını bağlar: cwd, tam argv, mevcut olduğunda env bağlaması ve geçerli olduğunda sabitlenmiş yürütülebilir dosya yolu.
- Kabuk betikleri ve doğrudan yorumlayıcı/çalışma zamanı dosyası çağrıları için OpenClaw ayrıca tek bir somut yerel dosya işlenenini bağlamaya çalışır. Bu dosya onaydan sonra ancak yürütmeden önce değişirse değiştirilmiş içerik çalıştırılmak yerine çalıştırma reddedilir.
- Dosya bağlama azami gayret esasına dayanır ve her yorumlayıcı/çalışma zamanı yükleyici yolunu kapsayan eksiksiz bir model değildir. Tam olarak bir somut yerel dosya belirlenemezse OpenClaw tam kapsam sağlıyormuş gibi davranmak yerine onay destekli bir çalıştırma oluşturmayı reddeder.

### macOS ayrımı

- **Node ana makinesi hizmeti**, `system.run` öğesini yerel IPC üzerinden **macOS uygulamasına** iletir.
- **macOS uygulaması** onayları uygular ve komutu kullanıcı arayüzü bağlamında yürütür.

## Geçerli politikayı inceleme

| Komut                                                          | Gösterdikleri                                                                          |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `openclaw approvals get` / `--gateway` / `--node <id\|name\|ip>` | İstenen politika, ana makine politika kaynakları ve geçerli sonuç.                       |
| `openclaw exec-policy show`                                      | Yerel makinenin birleştirilmiş görünümü.                                                             |
| `openclaw exec-policy set` / `preset`                            | Yerel olarak istenen politikayı yerel ana makine onay dosyasıyla tek adımda eşitler. |

<Note>
Oturum başına `/exec` geçersiz kılmaları dahil değildir. Geçerli varsayılanlarını incelemek için ilgili oturumda `/exec` komutunu çalıştırın. [Oturum geçersiz kılmaları](/tr/tools/exec#session-overrides-exec) bölümüne bakın.
</Note>

Tam CLI referansı (bayraklar, JSON çıktısı, izin listesine ekleme/çıkarma): [Onaylar CLI'si](/tr/cli/approvals).

Yerel bir kapsam `host=node` istediğinde `exec-policy show`, yerel onay
dosyasını doğruluk kaynağı olarak kabul etmek yerine bu kapsamı çalışma zamanında Node tarafından yönetiliyor
olarak bildirir.

Yardımcı uygulama kullanıcı arayüzü **kullanılamıyorsa**, normalde
onay isteyecek her istek **onay istemi geri dönüşü** ile çözümlenir (varsayılan: `deny`).

<Tip>
Yerel sohbet onay istemcileri, bekleyen onay mesajına kanala özgü
kolaylıklar ekleyebilir. Matrix, mesajda geri dönüş olarak `/approve ...` öğesini
bırakmaya devam ederken tepki kısayolları sağlar (`✅` bir kez izin ver,
`♾️` her zaman izin ver, `❌` reddet).
</Tip>

## Ayarlar ve depolama

Onaylar, yürütme ana makinesindeki yerel bir JSON dosyasında tutulur.
`OPENCLAW_STATE_DIR` ayarlandığında dosya bu durum dizinini izler;
aksi takdirde varsayılan OpenClaw durum dizinini kullanır:

```text
$OPENCLAW_STATE_DIR/exec-approvals.json
# aksi takdirde
~/.openclaw/exec-approvals.json
```

Varsayılan onay soketi aynı kökü izler:
`$OPENCLAW_STATE_DIR/exec-approvals.sock`; değişken ayarlanmamışsa
`~/.openclaw/exec-approvals.sock`.

Durum dizinleri birbirinden bağımsız güven kapsamlarıdır. `OPENCLAW_STATE_DIR`
başka bir konumu gösterdiğinde OpenClaw, `~/.openclaw/exec-approvals.json` öğesini
asla içe aktarmaz veya arşivlemez; özel durum dizini için onayları ayrıca
yapılandırın. Doctor da eski `plugin-binding-approvals.json` öğesini yalnızca etkin durum
dizinine ait olduğunda içe aktarır.

Örnek şema:

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "argPattern": "sha256:argv:...",
          "source": "allow-always",
          "lastUsedAt": 1737150000000,
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        },
        {
          "pattern": "~/Projects/**/bin/git"
        }
      ]
    }
  }
}
```

## Politika ayarları

### `tools.exec.mode`

`tools.exec.mode`, ana makinede Exec için tercih edilen normalleştirilmiş politika yüzeyidir:

| Değer       | Davranış                                                                                                                                                                  |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `deny`      | Ana makinede Exec'i engeller.                                                                                                                                                          |
| `allowlist` | Yalnızca izin listesindeki komutları onay istemeden çalıştırır.                                                                                                                             |
| `ask`       | İzin listesi politikasını kullanır ve eşleşme olmadığında onay ister.                                                                                                                                   |
| `auto`      | İzin listesi politikasını kullanır, belirlenimci eşleşmeleri doğrudan çalıştırır ve onay eşleşmezliklerini insan onayı yoluna geri dönmeden önce OpenClaw'ın yerel otomatik inceleyicisine gönderir. |
| `full`      | Ana makinede Exec'i onay istemleri olmadan çalıştırır.                                                                                                                                   |

Doctor, kullanımdan kaldırılmış kalıcı `tools.exec.security` / `tools.exec.ask`
çiftini `tools.exec.mode` biçimine geçirir.

### `exec.security`

<ParamField path="security" type='"deny" | "allowlist" | "full"'>
  - `deny` - tüm ana makine Exec isteklerini engeller.
  - `allowlist` - yalnızca izin listesindeki komutlara izin verir.
  - `full` - her şeye izin verir (yükseltilmiş erişime eşdeğer).

Gateway/Node ana makineleri için varsayılan `full`; `sandbox` ana makinesi içinse
varsayılan `deny` değeridir.
</ParamField>

### `exec.ask`

<ParamField path="ask" type='"off" | "on-miss" | "always"'>
  Ana makinede Exec için yapılandırılmış onay isteme politikası. `tools.exec.ask` ve ana makine onay
  varsayılanlarından gelen temel onay istemi davranışını denetler.
  Varsayılan `off` değeridir. Çağrı başına `ask` araç parametresi (bkz.
  [Exec aracı](/tr/tools/exec#parameters)) yalnızca bu temeli sıkılaştırabilir ve
  kanal kaynaklı model çağrıları, geçerli ana makine onay istemi `off` olduğunda bunu yok sayar.

- `off` - hiçbir zaman onay istemez.
- `on-miss` - yalnızca izin listesi eşleşmediğinde onay ister.
- `always` - her komutta onay ister. Geçerli onay isteme modu `always` olduğunda `allow-always` kalıcı güven, istemleri **engellemez**.

</ParamField>

### `askFallback`

<ParamField path="askFallback" type='"deny" | "allowlist" | "full"'>
  Onay istemi gerektiği hâlde hiçbir kullanıcı arayüzüne erişilemediğinde (veya
  istem zaman aşımına uğradığında) uygulanacak çözüm. Belirtilmediğinde varsayılan `deny` değeridir.

- `deny` - engeller.
- `allowlist` - yalnızca izin listesi eşleşirse izin verir.
- `full` - izin verir.

</ParamField>

### `tools.exec.strictInlineEval`

<ParamField path="strictInlineEval" type="boolean">
  `true` olduğunda yorumlayıcı ikili dosyasının kendisi izin listesinde olsa bile
  satır içi kod değerlendirme biçimlerini yalnızca onayla çalıştırılabilir olarak ele alır. Tek bir kararlı dosya işleneniyle
  düzgün biçimde eşleşmeyen yorumlayıcı yükleyicileri için derinlemesine savunma sağlar.
</ParamField>

Katı modun yakaladığı örnekler: `python -c`, `node -e`/`--eval`/`-p`,
`ruby -e`, `perl -e`/`-E`, `php -r`, `lua -e`, `osascript -e` (ayrıca `awk`,
`sed`, `make`, `find -exec` ve `xargs` satır içi biçimleri).

Katı modda bu komutlar inceleyici veya açık onay gerektirir.
`tools.exec.mode: "auto"` ile komutun uygulanabilir bir planı varsa inceleyici düşük riskli tek bir yürütmeye izin verebilir;
aksi takdirde OpenClaw bir insandan onay ister.
İnceleyici geri dönüşüne ulaşan `Codex app-server` komut onayları, onay istekleri uygulanabilir ve çözümlenmiş
bir yürütülebilir dosya sunmadığından bir insandan onay ister.
`allow-always`, satır içi değerlendirme komutları için yeni izin listesi girdilerini kalıcı hâle getirmez.

### `tools.exec.commandHighlighting`

<ParamField path="commandHighlighting" type="boolean" default="false">
  Yalnızca sunum amaçlıdır: etkinleştirildiğinde OpenClaw, Web onay istemlerinin komut belirteçlerini
  vurgulayabilmesi için ayrıştırıcıdan türetilen komut aralıkları ekleyebilir. `security`, `ask`,
  izin listesi eşleştirmesi, katı satır içi değerlendirme davranışı, onay iletimi veya komut yürütmeyi
  **değiştirmez**.
</ParamField>

Genel olarak `tools.exec.commandHighlighting` altında veya aracı başına
`agents.entries.*.tools.exec.commandHighlighting` altında ayarlayın.

## YOLO modu (onaysız)

Ana makinede Exec'i onay istemleri olmadan çalıştırmak için **her iki** politika katmanını da açın:
OpenClaw yapılandırmasındaki istenen Exec politikası (`tools.exec.*`) **ve**
yürütme ana makinesi onay dosyasındaki ana makineye özgü onay politikası.

Belirtilmeyen `askFallback` için varsayılan `deny` değeridir. Kullanıcı arayüzü olmadığında bir onay isteminin
izin verme seçeneğine geri dönmesi gerekiyorsa ana makine `askFallback` değerini açıkça `full` olarak ayarlayın.

| Katman              | YOLO ayarı               |
| ------------------ | -------------------------- |
| `tools.exec.mode`  | `gateway`/`node` üzerinde `full` |
| Ana makine `askFallback` | `full`                     |

<Warning>
**Önemli ayrımlar:**

- `tools.exec.host=auto`, exec'in **nerede** çalışacağını seçer: kullanılabiliyorsa sandbox'ta, aksi takdirde gateway'de.
- YOLO, ana makinedeki exec'in **nasıl** onaylanacağını seçer: `security=full` ve `ask=off`.
- YOLO, yapılandırılmış ana makine exec politikasına ek olarak ayrı bir sezgisel komut gizleme onay geçidi veya betik ön kontrol reddetme katmanı eklemez.
- `auto`, gateway yönlendirmesini sandbox içindeki bir oturumdan serbestçe geçersiz kılınabilir hâle getirmez. Çağrı başına `host=node` isteğine `auto` üzerinden izin verilir; `host=gateway` seçeneğine yalnızca etkin bir sandbox çalışma zamanı olmadığında `auto` üzerinden izin verilir. Kararlı ve otomatik olmayan bir varsayılan için `tools.exec.host` değerini ayarlayın veya `/exec host=...` seçeneğini açıkça kullanın.

</Warning>

Kendi etkileşimsiz izin modlarını sunan CLI destekli sağlayıcılar
bu politikayı izleyebilir. OpenClaw'ın etkin exec
politikası YOLO olduğunda Claude CLI,
`--permission-mode bypassPermissions` ekler. OpenClaw tarafından yönetilen Claude canlı oturumlarında, OpenClaw'ın
etkin exec politikası Claude'un yerel izin modu üzerinde belirleyicidir:
YOLO, canlı başlatmaları `--permission-mode bypassPermissions` olarak normalleştirir ve
kısıtlayıcı etkin exec politikası, ham Claude arka uç argümanları başka bir
mod belirtse bile canlı başlatmaları
`--permission-mode default` olarak normalleştirir.

Daha ihtiyatlı bir kurulum istiyorsanız OpenClaw exec politikasını yeniden
`allowlist` / `on-miss` veya `deny` olarak sıkılaştırın.

### Kalıcı gateway ana makinesi "asla sorma" kurulumu

<Steps>
  <Step title="İstenen yapılandırma politikasını ayarlayın">
    ```bash
    openclaw config set tools.exec.host gateway
    openclaw config set tools.exec.mode full
    openclaw gateway restart
    ```
  </Step>
  <Step title="Ana makine onay dosyasını eşleştirin">
    ```bash
    openclaw approvals set --stdin <<'EOF'
    {
      version: 1,
      defaults: {
        security: "full",
        ask: "off",
        askFallback: "full"
      }
    }
    EOF
    ```
  </Step>
</Steps>

### Yerel kısayol

```bash
openclaw exec-policy preset yolo
```

Hem yerel `tools.exec.host/security/ask` değerini hem de yerel onay
dosyasının varsayılanlarını (`askFallback: "full"` dâhil) günceller. Bu, kasıtlı olarak
yalnızca yerelde çalışır. Gateway ana makinesi veya Node ana makinesi onaylarını uzaktan değiştirmek için
`openclaw approvals set --gateway` ya da `openclaw approvals set --node
<id|name|ip>` kullanın.

Diğer yerleşik ön ayarlar: `cautious` (`host=gateway`, `security=allowlist`,
`ask=on-miss`, `askFallback=deny`) ve `deny-all` (`host=gateway`,
`security=deny`, `ask=off`, `askFallback=deny`). Aynı şekilde uygulayın:
`openclaw exec-policy preset cautious`.

Tam bir ön ayar yerine alanları ayrı ayrı ayarlamak için bu bayrakların herhangi bir alt kümesiyle
`openclaw exec-policy set --host <auto|sandbox|gateway|node> --security
<deny|allowlist|full> --ask <off|on-miss|always> --ask-fallback
<deny|allowlist|full>` kullanın.

### Node ana makinesi

Bunun yerine aynı onay dosyasını Node üzerinde uygulayın:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

<Note>
**Yalnızca yerel kullanımın sınırlamaları:**

- `openclaw exec-policy`, Node onaylarını eşitlemez.
- `openclaw exec-policy set --host node` reddedilir.
- Node exec onayları çalışma zamanında Node üzerinden alınır; bu nedenle Node hedefli güncellemeler `openclaw approvals --node ...` kullanmalıdır.

</Note>

### Yalnızca oturuma yönelik kısayol

- `/exec security=full ask=off`, yalnızca mevcut oturumu değiştirir.
- `/elevated full`, yalnızca hem istenen politika hem de ana makine onay dosyası
`security: "full"` ve `ask: "off"` olarak çözümlendiğinde exec onaylarını atlayan
acil durum kısayoludur. `ask:
"always"` gibi daha katı bir ana makine dosyası yine onay ister.

Ana makine onay dosyası yapılandırmadan daha katı kalırsa daha katı olan ana makine
politikası yine geçerli olur.

## İzin verilenler listesi (agent başına)

İzin verilenler listeleri **agent başınadır**. Birden fazla agent varsa macOS uygulamasında
düzenlediğiniz agent'ı değiştirin. Desenler glob eşleşmeleridir.

Desenler, çözümlenmiş ikili dosya yolu glob'ları veya yalnızca komut adı içeren glob'lar olabilir.
Yalnızca ad içerenler sadece `PATH` aracılığıyla çağrılan komutlarla eşleşir; dolayısıyla komut `rg` olduğunda `rg`,
`/opt/homebrew/bin/rg` ile eşleşebilir ancak **`./rg` veya
`/tmp/rg` ile eşleşmez**. Belirli tek bir ikili dosya konumuna güvenmek için yol glob'u kullanın.

Eski `agents.default` girdileri yükleme sırasında `agents.main` biçimine taşınır.
`echo ok && pwd` gibi kabuk zincirlerinde yine her üst düzey bölümün
izin verilenler listesi kurallarını karşılaması gerekir.

Örnekler:

- `rg`
- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

### Argümanları argPattern ile kısıtlama

Bir izin verilenler listesi girdisinin bir ikili dosyayla ve belirli bir
argüman biçimiyle eşleşmesi gerektiğinde `argPattern` ekleyin. OpenClaw, her ana makinede
ECMAScript (JavaScript) düzenli ifade semantiğini kullanır ve ifadeyi
çalıştırılabilir dosya belirteci (`argv[0]`) hariç, ayrıştırılmış komut argümanlarına göre
değerlendirir. Elle yazılan girdilerde argümanlar tek bir boşlukla birleştirilir; bu nedenle
tam eşleşme gerektiğinde deseni sabitleyin.

```json
{
  "version": 1,
  "agents": {
    "main": {
      "allowlist": [
        {
          "pattern": "python3",
          "argPattern": "^safe\\.py$"
        }
      ]
    }
  }
}
```

Bu girdi `python3 safe.py` komutuna izin verir; `python3 other.py`, izin verilenler listesiyle
eşleşmez. Aynı ikili dosya için yalnızca yol içeren bir girdi de varsa eşleşmeyen
argümanlar yine bu yalnızca yol içeren girdiye geri dönebilir. Amaç ikili dosyayı
bildirilen argümanlarla kısıtlamaksa yalnızca yol içeren girdiyi dahil etmeyin.

Onay akışları tarafından kaydedilen girdiler, tam argv eşleşmesi için dahili bir
ayırıcı biçimi kullanır. Kodlanmış değeri elle düzenlemek yerine bu girdileri yeniden oluşturmak için
kullanıcı arayüzünü veya onay akışını tercih edin. OpenClaw bir komut bölümü için argv'yi
ayrıştıramazsa `argPattern` içeren girdiler eşleşmez.

Oluşturulan `allow-always` girdileri argv'ye bağlıdır. Yeni oluşturulan girdiler
`argPattern` içerir; daha eski, yalnızca yol içeren oluşturulmuş girdiler yok sayılır ve yeni bir
onay gerektirir. Elle oluşturulan yalnızca yol içeren bir kural için hem `source` hem de `argPattern` alanlarını dahil etmeyin.

Her izin verilenler listesi girdisi şunları destekler:

| Alan               | Anlamı                                                                   |
| ------------------ | ------------------------------------------------------------------------ |
| `pattern`          | Çözümlenmiş ikili dosya yolu glob'u veya yalnızca komut adı içeren glob   |
| `argPattern`       | ECMAScript argv regex'i veya oluşturulan tam argv karması; dahil edilmezse yalnızca yol |
| `id`               | Kararlı, opak kimlik; yoksa UUID olarak oluşturulur                       |
| `source`           | `allow-always` gibi oluşturulan girdi kaynağı; elle oluşturulan girdilerde dahil etmeyin |
| `commandText`      | Eski düz metin girdisi; yükleme sırasında atılır                          |
| `lastUsedAt`       | Son kullanım zaman damgası                                                |
| `lastUsedCommand`  | Eşleşen son komut; oluşturulan karmalı argv girdilerinde dahil edilmez    |
| `lastResolvedPath` | Son çözümlenen ikili dosya yolu                                           |

## Skill CLI'larına otomatik izin verme

**Skill CLI'larına otomatik izin verme** (`autoAllowSkills`) etkinleştirildiğinde, bilinen Skills tarafından
başvurulan çalıştırılabilir dosyalar Node'larda (macOS Node'u
veya başsız Node ana makinesi) izin verilenler listesinde kabul edilir. Skill ikili dosya listesini
almak için Gateway RPC üzerinden `skills.bins` kullanılır. Kesin ve elle yönetilen
izin verilenler listeleri istiyorsanız bunu devre dışı bırakın.

<Warning>
- Bu, elle oluşturulan yol izin verilenler listesi girdilerinden ayrı bir **örtük kolaylık izin verilenler listesidir**.
- Gateway ile Node'un aynı güven sınırı içinde olduğu güvenilir operatör ortamları için tasarlanmıştır.
- Kesin ve açık güven gerekiyorsa `autoAllowSkills: false` kullanmaya devam edin ve yalnızca elle oluşturulan yol izin verilenler listesi girdilerini kullanın.

</Warning>

## Güvenli ikili dosyalar ve onay iletme

Güvenli ikili dosyalar (yalnızca stdin kullanan hızlı yol), yorumlayıcı bağlama ayrıntıları ve
onay istemlerinin Slack/Discord/Telegram'a nasıl iletileceği (veya yerel onay istemcileri olarak nasıl
çalıştırılacağı) hakkında bilgi için
[Gelişmiş exec onayları](/tr/tools/exec-approvals-advanced) bölümüne bakın.

## Control UI ile düzenleme

Varsayılanları, agent başına geçersiz kılmaları ve izin verilenler listelerini düzenlemek için **Control UI -> Nodes -> Exec approvals** kartını kullanın.
Bir kapsam (Defaults veya bir agent) seçin,
politikayı ayarlayın, izin verilenler listesi desenleri ekleyin/kaldırın ve ardından **Save** seçeneğine tıklayın. Kullanıcı arayüzü,
listeyi düzenli tutabilmeniz için desen başına son kullanım meta verilerini
gösterir.

Hedef seçici **Gateway** (yerel onaylar) veya bir **Node** seçer.
Node'lar `system.execApprovals.get/set` özelliğini duyurmalıdır (macOS uygulaması veya başsız
Node ana makinesi). Bir Node henüz exec onaylarını duyurmuyorsa yerel
onay dosyasını doğrudan düzenleyin.

Windows yardımcı uygulaması dâhil bazı Node ana makineleri farklı bir onay
politikası biçimine sahiptir. Control UI, ana makineye özgü bu politikaları salt okunur gösterir. Bunları düzenlemek için
yardımcı uygulamayı veya yerel politika biçimiyle `openclaw approvals set --node <id|name|ip>` kullanın;
bkz. [Onaylar CLI'sı](/tr/cli/approvals).

CLI: `openclaw approvals`, gateway veya Node düzenlemeyi destekler; bkz.
[Onaylar CLI'sı](/tr/cli/approvals).

## Onay akışı

Onay istemi gerektiğinde gateway,
`exec.approval.requested` olayını operatör istemcilerine yayınlar. Control UI ve macOS
uygulaması bunu `exec.approval.resolve` aracılığıyla çözümler; ardından gateway,
onaylanan isteği Node ana makinesine iletir.

`host=node` için onay istekleri, standartlaştırılmış bir `systemRunPlan`
yükü içerir. Gateway, onaylanmış `system.run` isteklerini iletirken bu planı
belirleyici komut/cwd/oturum bağlamı olarak kullanır:

- Node exec yolu, baştan tek bir standartlaştırılmış plan hazırlar.
- Onay kaydı bu planı ve bağlama meta verilerini saklar.
- Onaylandıktan sonra iletilen son `system.run` çağrısı, çağıranın sonraki düzenlemelerine güvenmek yerine saklanan planı yeniden kullanır.
- Çağıran, onay isteği oluşturulduktan sonra `command`, `rawCommand`, `cwd`, `agentId` veya `sessionKey` değerlerini değiştirirse gateway, iletilen çalıştırmayı onay uyuşmazlığı nedeniyle reddeder.

## Sistem olayları ve retler

Exec yaşam döngüsü, Node tamamlanma bildirdikten sonra agent'ın
oturumuna bir `Exec finished` sistem iletisi gönderir. OpenClaw ayrıca
onay verildikten ve `tools.exec.approvalRunningNoticeMs` süresi geçtikten sonra
(varsayılan `10000`; `0` bunu devre dışı bırakır) devam eden bir işlem bildirimi yayımlayabilir.
Reddedilen exec onayları ana makine komutu açısından nihaidir: komut
çalıştırılmaz.

- Kaynak oturumu bulunan ana agent eşzamansız onaylarında OpenClaw,
  agent'ın eşzamansız komutu beklemeyi bırakabilmesi ve eksik sonuç
  onarımından kaçınabilmesi için reddi dahili bir takip olarak o oturuma geri gönderir.
- Oturum yoksa veya oturum devam ettirilemiyorsa OpenClaw yine de
  operatöre veya doğrudan sohbet rotasına kısa bir ret bildirebilir.
- Alt agent ve Cron oturumlarının retleri o
  oturuma geri gönderilmez.

Gateway ana makinesi exec onayları aynı tamamlanma yaşam döngüsü olayını yayımlar.
Onay geçitli exec işlemleri, bekleyen isteği tamamlanma/ret iletisiyle
(`Exec finished (gateway
id=...)` / `Exec denied (gateway id=...)`) ilişkilendirmek için onay kimliğini yeniden kullanır.

## Sonuçlar

- **`full`** güçlüdür; mümkün olduğunda izin verilenler listelerini tercih edin.
- **`ask`**, hızlı onaylara izin verirken sizi süreçten haberdar eder.
- Agent başına izin verilenler listeleri, bir agent'ın onaylarının diğerlerine sızmasını önler.
- Onaylar yalnızca **yetkili göndericilerden** gelen ana makine exec isteklerine uygulanır. Yetkisiz göndericiler `/exec` gönderemez.
- `/exec security=full`, yetkili operatörler için oturum düzeyinde bir kolaylıktır ve tasarım gereği onayları atlar. Ana makine exec işlemini kesin olarak engellemek için onay güvenliğini `deny` olarak ayarlayın veya araç politikası aracılığıyla `exec` aracını reddedin.

## İlgili

<CardGroup cols={2}>
  <Card title="Exec onayları - gelişmiş" href="/tr/tools/exec-approvals-advanced" icon="gear">
    Güvenli ikili dosyalar, yorumlayıcı bağlama ve onayların sohbete iletilmesi.
  </Card>
  <Card title="Exec aracı" href="/tr/tools/exec" icon="terminal">
    Kabuk komutlarını yürütme aracı.
  </Card>
  <Card title="Yükseltilmiş mod" href="/tr/tools/elevated" icon="shield-exclamation">
    Onayları da atlayan acil durum erişim yolu.
  </Card>
  <Card title="Korumalı alan kullanımı" href="/tr/gateway/sandboxing" icon="box">
    Korumalı alan modları ve çalışma alanı erişimi.
  </Card>
  <Card title="Güvenlik" href="/tr/gateway/security" icon="lock">
    Güvenlik modeli ve sağlamlaştırma.
  </Card>
  <Card title="Korumalı alan, araç politikası ve yükseltilmiş mod karşılaştırması" href="/tr/gateway/sandbox-vs-tool-policy-vs-elevated" icon="sliders">
    Her bir denetimin ne zaman kullanılacağı.
  </Card>
  <Card title="Skills" href="/tr/tools/skills" icon="sparkles">
    Skills destekli otomatik izin verme davranışı.
  </Card>
</CardGroup>
