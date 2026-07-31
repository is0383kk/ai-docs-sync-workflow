---
read_when:
    - Belleğin nasıl çalıştığını anlamak istiyorsunuz
    - Hangi bellek dosyalarının yazılacağını öğrenmek istiyorsunuz
summary: OpenClaw'un oturumlar arasında bilgileri nasıl hatırladığı
title: Belleğe genel bakış
x-i18n:
    generated_at: "2026-07-26T23:54:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cdfd5276d6289a4ee38b5203eb5443312c4b040d4ea67abe4a9c579703136339
    source_path: concepts/memory.md
    workflow: 16
---

OpenClaw, aracınızın çalışma alanına düz Markdown dosyaları yazarak bilgileri
hatırlar (varsayılan `~/.openclaw/workspace`). Model yalnızca diske kaydedilenleri
hatırlar; gizli bir durum yoktur.

## Nasıl çalışır

Aracınızın bellekle ilgili üç dosyası vardır:

- **`MEMORY.md`** — uzun süreli bellek. Kalıcı olgular, tercihler ve
  kararlar. Bir oturumun başlangıcında yüklenir.
- **`memory/YYYY-MM-DD.md`** (veya `memory/YYYY-MM-DD-<slug>.md`) — günlük notlar.
  Süregelen bağlam ve gözlemler. Bugüne ve düne ait tarihli notlar, yalın bir
  `/new` veya `/reset` kullanıldığında otomatik olarak yüklenir;
  paketle birlikte gelen oturum belleği kancası tarafından yazılanlar gibi kısa
  ad eklenmiş çeşitler de yalnızca tarihi içeren dosyayla birlikte alınır.
- **`DREAMS.md`** (isteğe bağlı) — insanların incelemesi için Dream Diary
  ve Dreaming taraması özetleri; dayanaklı geçmişe dönük doldurma girdilerini de içerir.

<Tip>
Aracınızın bir şeyi hatırlamasını istiyorsanız yalnızca şunu söyleyin:
"TypeScript'i tercih ettiğimi hatırla." Notu uygun dosyaya yazar.
</Tip>

## Nereye ne yazılır

`MEMORY.md`, küçük ve özenle düzenlenmiş katmandır: bir oturumun başlangıcında
kullanılabilir olması gereken kalıcı olgular, tercihler, geçerli kararlar ve kısa
özetler. Ham bir döküm, günlük kayıt veya kapsamlı arşiv değildir.

`memory/YYYY-MM-DD.md` dosyaları çalışma katmanıdır: daha sonra hâlâ yararlı
olabilecek ayrıntılı günlük notlar, gözlemler, oturum özetleri ve ham bağlam.
Bunlar `memory_search` ve `memory_get` için dizine eklenir ancak her turda
önyükleme istemine eklenmez.

Araç zamanla günlük notlardaki yararlı materyali ayıklayıp
`MEMORY.md` içine aktarır ve eskimiş uzun süreli girdileri kaldırır.
Oluşturulan çalışma alanı talimatları ve Heartbeat akışı bunu düzenli aralıklarla
yapar; her ayrıntı için `MEMORY.md` dosyasını elle düzenlemeniz gerekmez.

`MEMORY.md` önyükleme dosyası bütçesini aşarsa OpenClaw, diskteki dosyayı
değiştirmeden tutar ancak bağlama eklenen kopyayı kısaltır. Bunu, ayrıntılı
materyali `memory/*.md` içine taşımanız, `MEMORY.md` içinde yalnızca
kalıcı bir özet tutmanız veya daha fazla istem bütçesi harcamak istiyorsanız
önyükleme sınırlarını yükseltmeniz gerektiğine dair bir işaret olarak değerlendirin.
Ham ve eklenen boyutları ve kısaltma durumunu görmek için `/context list`,
`/context detail` veya `openclaw doctor` kullanın.

## Kodlama yardımcılarından içe aktarma

Control UI, Codex ve Claude Code'daki mevcut yerel belleği içe aktarabilir.
**Settings** → **Import Memory** bölümünü açın, hedef aracı seçin, algılanan
dosyaları inceleyin ve içe aktarmayı onaylayın. OpenClaw yalnızca Markdown
belleğini kopyalar:

- Codex: `~/.codex/memories` (veya `CODEX_HOME/memories`) altındaki
  birleştirilmiş `MEMORY.md` ve `memory_summary.md` dosyaları. Ham rollout ve
  döküm dosyaları içe aktarılmaz.
- Claude Code: Her projenin `~/.claude/projects/*/memory` altındaki otomatik bellek
  dizininde bulunan Markdown dosyaları ve mevcutsa kullanıcı tarafından
  yapılandırılmış `autoMemoryDirectory`. Proje talimatları, oturumlar, ayarlar
  ve kimlik bilgileri yalnızca belleğe yönelik bu işlemin parçası değildir.

İçe aktarılan dosyalar, seçilen araç çalışma alanında `memory/imports/codex/` ve
`memory/imports/claude-code/` altında ayrı tutulur. `memory_search` için dizine
eklenir ve `memory_get` aracılığıyla kullanılabilirler; aracın önyükleme
`MEMORY.md` dosyasıyla birleştirilmezler. Kaynak dosyalar değiştirilmez.

Önizleme, hedefteki çakışmaları işaretler. Bu dosyaları değiştirmek için
**Replace existing imports** seçeneğini etkinleştirin; uygulama işlemi,
içe aktarma öncesi doğrulanmış bir yedek oluşturur ve üzerine yazılan dosyaların
öğe düzeyindeki kopyalarını geçiş raporunda korur.

## Eyleme duyarlı anılar

Çoğu anı sıradan Markdown notlarıdır. Bazıları aracın daha sonra ne yapması
gerektiğini etkiler; bunlar için yalnızca olguyu değil, nota göre ne zaman
güvenle harekete geçilebileceğini de kaydedin.

Bir not şunları içerdiğinde bu eylem sınırını kaydedin:

- onay veya izin gereksinimleri,
- geçici kısıtlamalar,
- başka bir oturuma, ileti dizisine veya kişiye devirler,
- geçerlilik sonu koşulları,
- harekete geçmek için güvenli zamanlama,
- kaynak veya sahip yetkisi,
- cazip bir eylemden kaçınma talimatları.

Yararlı bir eyleme duyarlı anı şunları açıkça belirtir:

- gelecekteki davranışı neyin değiştirdiği,
- ne zaman veya hangi koşul altında geçerli olduğu,
- ne zaman sona erdiği veya eylemin kilidini neyin açtığı,
- aracın ne yapmaktan kaçınması gerektiği,
- güveni veya yetkiyi etkiliyorsa kaynağın ya da sahibin kim olduğu.

Bellek, onay bağlamını koruyabilir ancak politikayı uygulamaz. Katı operasyonel
denetimler için OpenClaw onay ayarlarını, korumalı alan kullanımını ve
zamanlanmış görevleri kullanın.

Örnek:

```md
API geçişi başka bir oturumda tasarlanıyor. Gelecekteki turlar bu ileti
dizisinden API uygulamasını düzenlememeli; geçiş planı tamamlanana kadar
buradaki bulguları yalnızca tasarım girdisi olarak kullanın.
```

Başka bir örnek:

```md
Güvenilmeyen bir kaynaktan gelen raporun öne çıkarılmadan önce incelenmesi
gerekir. Gelecekteki turlar bunu yalnızca kanıt olarak değerlendirmeli;
güvenilir bir inceleyen içeriği doğrulayana kadar kalıcı bellek olarak
saklamayın.
```

Bu, her anı için zorunlu bir şema değildir; basit olgular kısa tutulabilir.
Zamanlama, yetki, geçerlilik sonu veya harekete geçme güvenliği bağlamının
kaybolması, aracın daha sonra yanlış bir şey yapmasına neden olabilecekse
eyleme duyarlı sınırlar kullanın.

Kesin anımsatıcılar, zamanlanmış denetimler ve yinelenen işler için
[zamanlanmış görevleri](/tr/automation/cron-jobs) kullanın. Bellek, bu işlerin
çevresindeki kalıcı bağlamı yine de özetleyebilir.

## Kullanımdan kaldırılan çıkarımsal taahhütler

Geleceğe yönelik bazı takip işlemleri kalıcı olgular değildir. Yarın yapılacak
bir görüşmeden söz ederseniz yararlı anı, "bunu sonsuza kadar
`MEMORY.md` içinde sakla" değil, "görüşmeden sonra durumu sor" olabilir.

Çıkarımsal taahhütler deneyi kullanımdan kaldırılmıştır. OpenClaw artık bu
takip işlemlerini ayıklamaz veya iletmez. Gelecekteki eylemler için
[zamanlanmış görevleri](/tr/automation/cron-jobs) kullanın; eski
`openclaw commitments` komutu, mevcut saklanmış satırları incelemek veya kapatmak
için kullanılabilir durumda kalır.

## Bellek araçları

Aracın bellekle çalışmak için iki aracı vardır:

- **`memory_search`** — ifade biçimi özgün metinden farklı olsa bile
  anlamsal aramayla ilgili notları bulur.
- **`memory_get`** — belirli bir bellek dosyasını veya satır aralığını okur.

Her iki araç da etkin bellek Plugin'i tarafından sağlanır (varsayılan:
`memory-core`).

## Bellek araması

Bir gömme sağlayıcısı yapılandırıldığında `memory_search` karma arama
kullanır: anahtar sözcük eşleştirmesiyle (kimlikler ve kod sembolleri gibi tam
terimler) birleştirilmiş vektör benzerliği (anlamsal anlam). Bu, desteklenen
herhangi bir sağlayıcı için API anahtarıyla kullanıma hazır olarak çalışır.

<Info>
OpenClaw varsayılan olarak OpenAI gömmelerini kullanır. Gemini, Voyage,
Mistral, Bedrock, DeepInfra, yerel GGUF, Ollama, LM Studio, GitHub Copilot veya
genel bir OpenAI uyumlu uç nokta kullanmak için `memory.search.provider` değerini
açıkça ayarlayın.
</Info>

Aramanın nasıl çalıştığı, ayarlama seçenekleri ve sağlayıcı kurulumu için
[Bellek araması](/tr/concepts/memory-search) bölümüne bakın.

## Bellek arka uçları

<CardGroup cols={3}>
<Card title="Yerleşik (varsayılan)" icon="database" href="/tr/concepts/memory-builtin">
SQLite tabanlıdır. Anahtar sözcük araması, vektör benzerliği ve karma aramayla
kullanıma hazır olarak çalışır. Ek bağımlılık gerektirmez.
</Card>
<Card title="QMD" icon="search" href="/tr/concepts/memory-qmd">
Yeniden sıralama, sorgu genişletme ve çalışma alanı dışındaki dizinleri dizine
ekleme özelliğine sahip, önce yerel yaklaşımı benimseyen yardımcı süreç.
</Card>
<Card title="Honcho" icon="brain" href="/tr/concepts/memory-honcho">
Kullanıcı modelleme, anlamsal arama ve çoklu araç farkındalığı özelliklerine
sahip, yapay zekâya özgü oturumlar arası bellek. Plugin kurulumu.
</Card>
<Card title="LanceDB" icon="layers" href="/tr/plugins/memory-lancedb">
OpenAI uyumlu gömmeler, otomatik anımsama, otomatik yakalama ve yerel Ollama
gömme desteği sunan LanceDB destekli bellek. Plugin kurulumu.
</Card>
</CardGroup>

## Bilgi vikisi katmanı

Kalıcı belleğin ham notlardan çok bakımı yapılan bir bilgi tabanı gibi
davranmasını istiyorsanız paketle birlikte gelen `memory-wiki` Plugin'ini
kullanın. Kalıcı bilgiyi; belirlenimci sayfa yapısı, yapılandırılmış iddialar
ve kanıtlar, çelişki ve güncellik takibi, oluşturulan panolar, derlenmiş
özetler ve vikiye özgü araçlar (`wiki_status`, `wiki_search`,
`wiki_get`, `wiki_apply`, `wiki_lint`) içeren bir viki
kasasında derler.

`memory-wiki`, etkin bellek Plugin'inin yerini almaz; anımsama, öne
çıkarma ve Dreaming yine etkin bellek Plugin'inin sorumluluğundadır.
`memory-wiki`, bunun yanına köken bilgisi açısından zengin bir bilgi
katmanı ekler.

<CardGroup cols={1}>
<Card title="Bellek Vikisi" icon="book" href="/tr/plugins/memory-wiki">
Kalıcı belleği; iddialar, panolar, köprü modu ve Obsidian dostu iş akışları
içeren, köken bilgisi açısından zengin bir viki kasasında derler.
</Card>
</CardGroup>

## Otomatik bellek boşaltma

[Compaction](/tr/concepts/compaction) konuşmanızı özetlemeden önce OpenClaw,
araca önemli bağlamı bellek dosyalarına kaydetmesini hatırlatan sessiz bir tur
çalıştırır. Bu özellik varsayılan olarak açıktır; kapatmak için
`agents.defaults.compaction.memoryFlush.enabled: false` değerini ayarlayın.

Bu bakım turunu yerel bir modelde tutmak için yalnızca bellek boşaltma turuna
uygulanan kesin bir geçersiz kılma ayarlayın (etkin oturumun model geri dönüş
zincirini devralmaz):

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "model": "ollama/qwen3:8b"
        }
      }
    }
  }
}
```

<Tip>
Bellek boşaltma, Compaction sırasında bağlam kaybını önler. Aracınızın
konuşmada henüz bir dosyaya yazılmamış önemli olguları varsa özetleme
gerçekleşmeden önce otomatik olarak kaydedilirler.
</Tip>

## Dreaming

Dreaming, bellek için isteğe bağlı bir arka plan birleştirme geçişidir. Kısa
süreli anımsama sinyallerini toplar, adayları puanlar ve yalnızca yeterli
nitelikteki öğeleri uzun süreli belleğe (`MEMORY.md`) aktarır:

- **İsteğe bağlı**: varsayılan olarak devre dışıdır.
- **Zamanlanmış**: etkinleştirildiğinde `memory-core`, tam Dreaming
  taraması için yinelenen tek bir Cron işini otomatik olarak yönetir.
- **Eşikli**: aktarımlar puan, anımsama sıklığı ve sorgu çeşitliliği
  eşiklerini geçmelidir.
- **İncelenebilir**: aşama özetleri ve günlük girdileri, insanların
  incelemesi için `DREAMS.md` konumuna yazılır.

Aşama davranışı, puanlama sinyalleri ve Dream Diary ayrıntıları için
[Dreaming](/tr/concepts/dreaming) bölümüne bakın.

## Dayanaklı geçmişe dönük doldurma ve canlı aktarım

Dreaming sisteminin birbiriyle ilişkili iki inceleme hattı vardır:

- **Canlı Dreaming**, `memory/.dreams/` altındaki kısa süreli Dreaming
  deposundan çalışır ve normal derin aşamanın nelerin `MEMORY.md` içine
  aktarılacağına karar vermek için kullandığı yöntemdir.
- **Dayanaklı geçmişe dönük doldurma**, geçmiş `memory/YYYY-MM-DD.md`
  notlarını bağımsız gün dosyaları olarak okur ve yapılandırılmış inceleme
  çıktısını `DREAMS.md` içine yazar.

Dayanaklı geçmişe dönük doldurma, `MEMORY.md` dosyasını elle
düzenlemeden eski notları yeniden oynatmak ve sistemin neleri kalıcı olarak
değerlendirdiğini incelemek için yararlıdır.

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

`--stage-short-term` bayrağı, dayanaklı kalıcı adayları normal derin aşamanın
zaten kullandığı aynı kısa süreli Dreaming deposuna hazırlar; bunları doğrudan
aktarmaz. Dolayısıyla:

- `DREAMS.md`, insanların inceleme yüzeyi olarak kalır.
- Kısa süreli depo, makineye yönelik sıralama yüzeyi olarak kalır.
- `MEMORY.md` hâlâ yalnızca derin aktarım tarafından yazılır.

Sıradan günlük girdilerine veya normal anımsama durumuna dokunmadan yeniden
oynatmayı geri almak için:

```bash
openclaw memory rem-backfill --rollback
openclaw memory rem-backfill --rollback-short-term
```

## CLI

```bash
openclaw memory status          # Dizin durumunu ve sağlayıcıyı denetle
openclaw memory search "query"  # Komut satırından ara
openclaw memory index --force   # Dizini yeniden oluştur
```

## Ek okumalar

- [Bellek arama](/tr/concepts/memory-search): arama işlem hattı, sağlayıcılar ve ince ayar.
- [Yerleşik bellek motoru](/tr/concepts/memory-builtin): varsayılan SQLite arka ucu.
- [QMD bellek motoru](/tr/concepts/memory-qmd): gelişmiş, önce yerel yan süreç.
- [Honcho bellek](/tr/concepts/memory-honcho): yapay zekâya özgü, oturumlar arası bellek.
- [Memory LanceDB](/tr/plugins/memory-lancedb): OpenAI uyumlu gömmelere sahip, LanceDB destekli plugin.
- [Bellek Vikisi](/tr/plugins/memory-wiki): derlenmiş bilgi kasası ve vikiye özgü araçlar.
- [Dreaming](/tr/concepts/dreaming): kısa süreli hatırlamadan uzun süreli belleğe arka planda yükseltme.
- [Bellek yapılandırma başvurusu](/tr/reference/memory-config): tüm yapılandırma ayarları.
- [Compaction](/tr/concepts/compaction): Compaction'ın bellekle nasıl etkileşime girdiği.
- [Active Memory](/tr/concepts/active-memory): etkileşimli sohbet oturumları için alt ajan belleği.
