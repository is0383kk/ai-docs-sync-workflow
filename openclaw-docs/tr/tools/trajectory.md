---
read_when:
    - Bir ajanın neden belirli bir şekilde yanıt verdiğini, başarısız olduğunu veya araçları çağırdığını hata ayıklama
    - Bir OpenClaw oturumu için destek paketi dışa aktarma
    - İstem bağlamını, araç çağrılarını, çalışma zamanı hatalarını veya kullanım meta verilerini inceleme
    - Yörünge yakalamayı devre dışı bırakma
summary: Bir OpenClaw ajan oturumunda hata ayıklamak için hassas bilgileri çıkarılmış yörünge paketlerini dışa aktarın
title: Yörünge paketleri
x-i18n:
    generated_at: "2026-07-27T00:10:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7fc494732b6239ad4ea58dca3920a47cb7433c680e7566855dd265c986b55e74
    source_path: tools/trajectory.md
    workflow: 16
---

Yörünge yakalama, OpenClaw'ın oturum başına çalışan uçuş kaydedicisidir. Her
ajan çalıştırması için yapılandırılmış bir zaman çizelgesi kaydeder, ardından `/export-trajectory` mevcut
oturumu şu öğeleri kapsayan, hassas bilgileri ayıklanmış bir destek paketi hâline getirir:

- Modele gönderilen istem, sistem istemi ve araçlar
- Bir yanıta hangi transkript mesajlarının ve araç çağrılarının yol açtığı
- Çalıştırmanın zaman aşımına uğrayıp uğramadığı, iptal edilip edilmediği, sıkıştırılıp sıkıştırılmadığı veya sağlayıcı hatasıyla karşılaşıp karşılaşmadığı
- Hangi modelin, pluginlerin, skills'lerin ve çalışma zamanı ayarlarının etkin olduğu
- Sağlayıcının döndürdüğü kullanım ve istem önbelleği meta verileri

Kapsamlı bir Gateway destek raporu için bunun yerine
[`/diagnostics`](/tr/gateway/diagnostics#chat-command) ile başlayın; bu komut,
temizlenmiş Gateway paketini toplar ve OpenAI Codex çalıştırma ortamı oturumlarında onay alındıktan
sonra Codex geri bildirimini OpenAI'a gönderebilir. Ayrıntılı oturum başına
istem, araç ve transkript zaman çizelgesine ihtiyaç duyduğunuzda `/export-trajectory` kullanın.

## Hızlı başlangıç

Etkin oturumda gönderin (diğer adıyla `/trajectory`):

```text
/export-trajectory
```

OpenClaw, paketi çalışma alanı altında yazar:

```text
.openclaw/trajectory-exports/openclaw-trajectory-<session>-<timestamp>/
```

Geçersiz kılmak için göreli bir çıktı dizini adı iletin:

```text
/export-trajectory bug-1234
```

Ad, `.openclaw/trajectory-exports/` içinde çözümlenir. Mutlak yollar ve
`~` yolları reddedilir.

Yörünge paketleri istemler, model mesajları, araç şemaları, araç
sonuçları, çalışma zamanı olayları ve yerel yollar içerebildiğinden sohbet komutu her zaman
exec onayından geçer. Paketi oluşturmak istediğinizde dışa aktarmayı bir kez
onaylayın; tümüne izin ver seçeneğini kullanmayın. Grup sohbetlerinde OpenClaw, yörünge
ayrıntılarını paylaşılan odaya göndermek yerine onay istemini ve dışa aktarma
sonucunu sahibine özel olarak gönderir.

Yerel inceleme veya destek iş akışları için temel CLI komutunu
doğrudan çalıştırın:

```bash
openclaw sessions export-trajectory --session-key "agent:main:telegram:direct:123" --workspace .
```

Diğer bayraklar: `--output <path>` (`.openclaw/trajectory-exports` içindeki
dizin adı), `--store <path>` (oturum deposunu geçersiz kılma),
`--agent <id>` (depo çözümlemesi için ajan kimliği), `--json` (yapılandırılmış çıktı).

## Erişim

Yörünge dışa aktarma bir sahip komutudur. Gönderen, kanalın sahip denetimine
ek olarak normal komut yetkilendirme denetimlerinden geçmelidir.

## Kaydedilenler

Yörünge yakalama, OpenClaw ajan çalıştırmaları için varsayılan olarak açıktır.

Çalışma zamanı olayları şunları içerir:

- `session.started`
- `trace.metadata`
- `context.compiled`
- `prompt.submitted`
- `model.fallback_step`; kaynak model, sonraki model, hata nedeni/ayrıntısı, zincir konumu ve zincirin ilerleyip ilerlemediği, başarılı olup olmadığı veya tükenip tükenmediği dâhil
- `model.completed`
- `trace.artifacts`
- `session.ended`

Transkript olayları etkin oturum dalından yeniden oluşturulur: kullanıcı
mesajları, asistan mesajları, araç çağrıları, araç sonuçları, sıkıştırmalar, model
değişiklikleri, etiketler ve özel oturum girdileri.

Olaylar, şu şema işaretçisiyle JSON Lines biçiminde yazılır:

```json
{
  "traceSchema": "openclaw-trajectory",
  "schemaVersion": 1
}
```

## Paket dosyaları

| Dosya                  | İçerik                                                                                       |
| --------------------- | ---------------------------------------------------------------------------------------------- |
| `manifest.json`       | Paket şeması, kaynak dosyalar, olay sayıları ve oluşturulan dosya listesi                             |
| `events.jsonl`        | Sıralı çalışma zamanı ve transkript zaman çizelgesi                                                        |
| `session-branch.json` | Hassas bilgileri ayıklanmış etkin transkript dalı ve oturum üstbilgisi                                           |
| `metadata.json`       | OpenClaw sürümü, işletim sistemi/çalışma zamanı, model, yapılandırma anlık görüntüsü, pluginler, skills'ler ve istem meta verileri     |
| `artifacts.json`      | Son durum, hatalar, kullanım, istem önbelleği, sıkıştırma sayısı, asistan metni ve araç meta verileri |
| `prompts.json`        | Gönderilen istemler ve seçili istem oluşturma ayrıntıları                                         |
| `system-prompt.txt`   | Yakalandığında en son derlenmiş sistem istemi                                                   |
| `tools.json`          | Yakalandığında modele gönderilen araç tanımları                                              |

`manifest.json`, belirli bir pakette bulunan dosyaları listeler; oturum
ilgili çalışma zamanı verilerini yakalamadıysa bazı dosyalar dahil edilmez.

## Yakalama depolaması

Çalışma zamanı yörünge olayları, oturumla birlikte ajan başına SQLite
veritabanında depolanır. Bir yörüngenin dışa aktarılması, hassas bilgileri ayıklanmış bir JSONL destek paketi oluşturur;
canlı çalışma zamanı yakalaması, oturuma bitişik bir JSONL yan dosyası değildir.

Eski sürümlerden veya açıkça yapılan eski dosya dışa aktarımlarından kalan
`.trajectory.jsonl` ve `.trajectory-path.json` dosyaları hâlâ görünebilir. Oturum bakımı bu
dosyaları temizleme hedefi olarak ele alır; etkin yakalama veritabanı satırları yazar.

## Yakalamayı devre dışı bırakma

```bash
export OPENCLAW_TRAJECTORY=0
```

Bu, OpenClaw başlatılmadan önce çalışma zamanı yörünge yakalamasını devre dışı bırakır.
`/export-trajectory` yine de transkript dalını dışa aktarabilir ancak derlenmiş
bağlam, sağlayıcı yapıtları ve istem meta verileri gibi yalnızca çalışma zamanına ait
veriler eksik olabilir.

## Boşaltma zaman aşımını ayarlama

OpenClaw, ajan temizliği sırasında çalışma zamanı yörünge satırlarını boşaltır. Varsayılan
temizleme zaman aşımı 10,000 ms'dir. Yavaş disklerde veya büyük depolarda OpenClaw'ı
başlatmadan önce `OPENCLAW_TRAJECTORY_FLUSH_TIMEOUT_MS` değerini ayarlayın:

```bash
export OPENCLAW_TRAJECTORY_FLUSH_TIMEOUT_MS=30000
```

Bu, OpenClaw'ın ne zaman `openclaw-trajectory-flush` zaman aşımını günlüğe kaydedip
devam edeceğini kontrol eder; yörünge boyutu üst sınırlarını değiştirmez. Açık bir zaman aşımı
iletmeyen tüm ajan temizleme adımlarını ayarlamak için
`OPENCLAW_AGENT_CLEANUP_TIMEOUT_MS` değerini belirleyin.

## Gizlilik ve sınırlar

Yörünge paketleri destek ve hata ayıklama içindir, herkese açık olarak paylaşılmamalıdır. OpenClaw,
dışa aktarma dosyalarını yazmadan önce hassas değerleri ayıklar:

- kimlik bilgileri ve gizli değer benzeri olduğu bilinen yük alanları
- görüntü verileri
- yerel durum yolları
- `$WORKSPACE_DIR` ile değiştirilen çalışma alanı yolları
- algılandığında ana dizin yolları

Dışa aktarıcı ayrıca girdi boyutunu sınırlar:

- çalışma zamanı yakalaması: canlı yakalama, 10 MiB ile sınırlandırılmış döngüsel bir penceredir ve yeni olaylara yer açmak için en eski olayları düşürür; dışa aktarma, 50 MiB'ye kadar mevcut eski çalışma zamanı yan dosyalarını kabul eder
- oturum dosyaları: 50 MiB
- dışa aktarma başına çalışma zamanı olayları: 200,000
- dışa aktarılan toplam olaylar: 250,000
- tekil çalışma zamanı olay satırları 256 KiB'nin üzerinde kesilir

Paketleri ekibinizin dışında paylaşmadan önce inceleyin. Hassas bilgileri ayıklama en iyi çabayı
gösterir ve uygulamaya özgü her gizli değeri bilemez.

## Sorun giderme

Dışa aktarmada çalışma zamanı olayı yoksa:

- OpenClaw'ın `OPENCLAW_TRAJECTORY=0` olmadan başlatıldığını doğrulayın
- oturumda başka bir mesaj çalıştırın, ardından yeniden dışa aktarın
- `runtimeEventCount` için `manifest.json` öğesini inceleyin

Komut çıktı yolunu reddederse:

- `bug-1234` gibi göreli bir ad kullanın
- `/tmp/...` veya `~/...` iletmeyin
- dışa aktarmayı `.openclaw/trajectory-exports/` içinde tutun

Dışa aktarma bir boyut hatasıyla başarısız olursa oturum veya yan dosya, yukarıdaki
dışa aktarma güvenlik sınırlarını aşmıştır. Yeni bir oturum başlatın veya daha küçük bir
yeniden üretim örneğini dışa aktarın.

## İlgili

- [Farklar](/tr/tools/diffs)
- [Oturum yönetimi](/tr/concepts/session)
- [Exec aracı](/tr/tools/exec)
