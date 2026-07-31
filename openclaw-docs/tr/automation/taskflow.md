---
read_when:
    - Task Flow'un arka plan görevleriyle nasıl ilişkili olduğunu anlamak istiyorsunuz
    - Sürüm notlarında veya belgelerde TaskFlow ya da openclaw tasks flow ifadeleriyle karşılaşırsınız
    - Kalıcı akış durumunu incelemek veya yönetmek istiyorsunuz
summary: Arka plan görevlerinin üzerindeki Task Flow orkestrasyon katmanı
title: Görev akışı
x-i18n:
    generated_at: "2026-07-26T23:49:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5ccc6acf58b4b44c2989e3061bff08dabce8ef385706102360c756a1286ddd1b
    source_path: automation/taskflow.md
    workflow: 16
---

Task Flow, [arka plan görevlerinin](/tr/automation/tasks) üzerindeki orkestrasyon katmanıdır. Bir akış; kendi durumu, JSON hali, revizyon sayacı ve bağlantılı görev kayıtları bulunan, çok adımlı çalışmanın kalıcı bir kaydıdır. Akışlar Gateway yeniden başlatmalarından etkilenmez; bağımsız çalışma birimi tekil görevler olmaya devam eder.

## Task Flow ne zaman kullanılmalı?

| Senaryo                                      | Kullanım                                     |
| -------------------------------------------- | -------------------------------------------- |
| Tek arka plan işi                            | Düz görev                                    |
| Plugin koduyla yürütülen çok adımlı işlem hattı | Task Flow (yönetilen)                     |
| Bağımsız ACP veya alt aracı başlatma          | Task Flow (yansıtılan, otomatik oluşturulur) |
| Tek seferlik hatırlatıcı                      | Cron işi                                     |

## Eşitleme modları

### Yönetilen mod

Yönetilen akışın bir denetleyicisi vardır: akışı Plugin çalışma zamanı Task Flow API'si aracılığıyla bir hedef ve zorunlu denetleyici kimliğiyle oluşturan, ardından açıkça yürüten Plugin kodu.

- Her adım, akış altında oluşturulan bir arka plan görevi olarak çalışır; akışın sahip anahtarı ve istekte bulunanın kaynağı alt görevlere aktarılır.
- Denetleyici, akışı `running`, `waiting` ve sonlandırıcı durumlar arasında ilerletir ve akış kaydında isteğe bağlı JSON adım hali saklar.
- Her değişiklikte akışın beklenen revizyonu iletilir. Eski bir yazma işlemi, daha yeni halin üzerine yazmak yerine revizyon çakışması olarak reddedilir.
- İptal istendikten sonra yeni alt görevler reddedilir ve etkin alt görev kalmadığında akış `cancelled` olarak sonlandırılır.

Örnek: (1) veri toplayan, (2) raporu oluşturan ve (3) teslim eden, her adım için bir arka plan görevi kullanan haftalık rapor akışı:

```
Akış: weekly-report
  Adım 1: gather-data     → görev oluşturuldu → başarılı
  Adım 2: generate-report → görev oluşturuldu → başarılı
  Adım 3: deliver         → görev oluşturuldu → çalışıyor
```

### Yansıtılan mod

Bağımsız bir ACP veya alt aracı çalıştırması başladığında (teslim edilebilir tamamlanmaya sahip oturum kapsamlı görevler), OpenClaw otomatik olarak yansıtılan tek görevli bir akış oluşturur. Akış kaydı, tek dayanak görevinin durumunu, hedefini ve zamanlamasını yansıtır; böylece bağımsız başlatmalar, denetleyici olmadan durum ve yeniden deneme yüzeyleri için kararlı bir akış tanıtıcısı edinir. Yansıtılan akışlar CLI'da `task_mirrored` eşitleme modunu gösterir.

## Akış durumları

| Durum       | Anlam                                                                      |
| ----------- | -------------------------------------------------------------------------- |
| `queued`    | Oluşturuldu, henüz ilerlemiyor                                             |
| `running`   | Akış etkin olarak ilerliyor                                                |
| `waiting`   | Yönetilen akış, bekleme meta verilerinde duraklatıldı (zamanlayıcı, harici olay) |
| `blocked`   | Bir adım kullanılabilir sonuç olmadan tamamlandı; hangisi olduğunu `blockedTaskId`/özet belirtir |
| `succeeded` | Başarıyla tamamlandı                                                       |
| `failed`    | Hatayla tamamlandı                                                         |
| `cancelled` | İptal istendi ve tüm alt görevler sonuçlandı                               |
| `lost`      | Akış, yetkili dayanak halini kaybetti                                      |

## Kalıcı hal ve revizyon takibi

Akış kayıtları, görev kayıtlarıyla birlikte paylaşılan SQLite hal veritabanında (`~/.openclaw/state/openclaw.sqlite`, `flow_runs` tablosu) kalıcı olarak saklanır; böylece ilerleme, Gateway yeniden başlatmalarından etkilenmez. Her yazma işlemi akışın `revision` değerini artırır; eski bir beklenen revizyon ileten eşzamanlı yazıcılar çakışma alır ve yeniden okumalıdır. WAL büyümesi, SQLite otomatik denetim noktası oluşturma ve düzenli pasif denetim noktalarıyla sınırlandırılır; kapatma sırasında truncate denetim noktaları kullanılır. Eski kurulumlardaki eski `flows/registry.sqlite` yan dosyası `openclaw doctor` tarafından içe aktarılır.

## İptal davranışı

`openclaw tasks flow cancel`, akışta kalıcı bir iptal niyeti belirler, etkin alt görevlerini iptal eder ve yeni yönetilen alt görevleri reddeder. Etkin alt görev kalmadığında akış `cancelled` olarak sonlandırılır: hemen veya alt görevlerin sonuçlanması daha uzun sürerse bakım taraması aracılığıyla. Niyet kalıcı olarak saklanır; dolayısıyla tüm alt görevler sonlanmadan önce Gateway yeniden başlatılsa bile iptal edilen bir akış iptal edilmiş olarak kalır.

## CLI komutları

```bash
# Etkin ve son akışları listele
openclaw tasks flow list [--status <status>] [--json]

# Belirli bir akışın ayrıntılarını göster
openclaw tasks flow show <lookup> [--json]

# Çalışan bir akışı ve etkin görevlerini iptal et
openclaw tasks flow cancel <lookup>
```

| Komut                             | Açıklama                                                               |
| --------------------------------- | ---------------------------------------------------------------------- |
| `openclaw tasks flow list`        | Eşitleme modu, durum, revizyon, denetleyici ve görev sayılarıyla izlenen akışlar |
| `openclaw tasks flow show <id>`   | Bağlantılı görevler dahil olmak üzere akış kimliği veya sahip anahtarıyla bir akışı incele |
| `openclaw tasks flow cancel <id>` | Çalışan bir akışı ve etkin görevlerini iptal et                         |

Akışlar ayrıca `openclaw tasks audit` (eski veya bozuk akış bulguları) ve `openclaw tasks maintenance` (takılı kalan iptalleri sonlandırır, sonlandırılmış akışları 7 gün sonra temizler) kapsamındadır.

## Güvenilir zamanlanmış iş akışı örüntüsü

Pazar istihbaratı bilgilendirmeleri gibi yinelenen iş akışlarında zamanlamayı, orkestrasyonu ve güvenilirlik kontrollerini ayrı katmanlar olarak ele alın:

1. Zamanlama için [Zamanlanmış Görevler](/tr/automation/cron-jobs) kullanın.
2. İş akışının önceki bağlamı temel alması gerektiğinde kalıcı bir Cron oturumu kullanın.
3. Belirlenimci adımlar, onay geçitleri ve sürdürme belirteçleri için [Lobster](/tr/tools/lobster) kullanın.
4. Alt görevler, beklemeler, yeniden denemeler ve Gateway yeniden başlatmaları boyunca çok adımlı çalıştırmayı izlemek için Task Flow kullanın.

Örnek Cron biçimi:

```bash
openclaw cron add \
  --name "Pazar istihbaratı özeti" \
  --cron "0 7 * * 1-5" \
  --tz "America/New_York" \
  --session session:market-intel \
  --message "market-intel Lobster iş akışını çalıştır. Özetlemeden önce kaynakların güncelliğini doğrula." \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

Yinelenen iş akışı bilinçli olarak geçmişe, önceki çalıştırma özetlerine veya kalıcı bağlama ihtiyaç duyduğunda `isolated` yerine `--session session:<id>` kullanın. Her çalıştırmanın sıfırdan başlaması ve gerekli tüm halin iş akışında açıkça belirtilmesi gerektiğinde `isolated` kullanın.

İş akışında güvenilirlik kontrollerini LLM özetleme adımından önce yerleştirin:

```yaml
name: market-intel-brief
steps:
  - id: preflight
    command: market-intel check --json
  - id: collect
    command: market-intel collect --json
    stdin: $preflight.json
  - id: summarize
    command: market-intel summarize --json
    stdin: $collect.json
  - id: approve
    command: market-intel deliver --preview
    stdin: $summarize.json
    approval: required
  - id: deliver
    command: market-intel deliver --execute
    stdin: $summarize.json
    condition: $approve.approved
```

Önerilen ön kontroller:

- Tarayıcı kullanılabilirliği ve profil seçimi; örneğin yönetilen hal için `openclaw` veya oturum açılmış bir Chrome oturumu gerektiğinde `user`. Bkz. [Tarayıcı](/tr/tools/browser).
- Her kaynak için API kimlik bilgileri ve kota.
- Gerekli uç noktalar için ağ erişilebilirliği.
- Aracı için etkinleştirilmiş gerekli araçlar; örneğin `lobster`, `browser` ve `llm-task`.
- Ön kontrol hatalarının görünür olması için Cron hata hedefinin yapılandırılması. Bkz. [Zamanlanmış Görevler](/tr/automation/cron-jobs#delivery-and-output).

Toplanan her öğe için önerilen veri kökeni alanları:

```json
{
  "sourceUrl": "https://example.com/report",
  "retrievedAt": "2026-04-24T12:00:00Z",
  "asOf": "2026-04-24",
  "title": "Örnek rapor",
  "content": "..."
}
```

İş akışının özetlemeden önce eski öğeleri reddetmesini veya eski olarak işaretlemesini sağlayın. LLM adımı yalnızca yapılandırılmış JSON almalı ve çıktısında `sourceUrl`, `retrievedAt` ve `asOf` değerlerini koruması istenmelidir. İş akışı içinde şemayla doğrulanan bir model adımına ihtiyaç duyduğunuzda [LLM Görevi](/tr/tools/llm-task) kullanın.

Yeniden kullanılabilir ekip veya topluluk iş akışlarında CLI'ı, `.lobster` dosyalarını ve tüm kurulum notlarını bir skill veya Plugin olarak paketleyip [ClawHub](/clawhub) üzerinden yayımlayın. Plugin API'sinde gerekli genel bir yetenek eksik olmadığı sürece iş akışına özgü korumaları bu pakette tutun.

## Akışların görevlerle ilişkisi

Akışlar görevlerin yerine geçmez, onları koordine eder. Tek bir akış, kullanım ömrü boyunca birden fazla arka plan görevini yürütebilir. Tekil görev kayıtlarını incelemek için `openclaw tasks`, orkestrasyonu gerçekleştiren akışı incelemek için `openclaw tasks flow` kullanın.

## İlgili

- [Arka Plan Görevleri](/tr/automation/tasks) - akışların koordine ettiği bağımsız çalışma günlüğü
- [CLI: görevler](/tr/cli/tasks) - `openclaw tasks flow` için CLI komut başvurusu
- [Otomasyona Genel Bakış](/tr/automation) - tüm otomasyon mekanizmalarına genel bakış
- [Cron İşleri](/tr/automation/cron-jobs) - akışları besleyebilen zamanlanmış işler
