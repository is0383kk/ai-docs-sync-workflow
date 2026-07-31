---
read_when:
    - Otomatik Compaction'ı ve /compact komutunu anlamak istiyorsunuz
    - Bağlam sınırlarına ulaşan uzun oturumlarda hata ayıklıyorsunuz
summary: OpenClaw uzun konuşmaları model sınırları içinde kalmak için nasıl özetler?
title: Compaction
x-i18n:
    generated_at: "2026-07-26T23:17:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eb1f794fa60affd602378bcff8b07786bfeca55ab3fa09d5fa7214a05fa48806
    source_path: concepts/compaction.md
    workflow: 16
---

Her modelin bir bağlam penceresi vardır: işleyebileceği en fazla token sayısı. Bir konuşma bu sınıra yaklaştığında OpenClaw, sohbetin devam edebilmesi için eski mesajları bir özet hâlinde **sıkıştırır**.

## Nasıl çalışır?

1. Konuşmanın eski turları, kısa bir girdi hâlinde özetlenir.
2. Özet, oturum transkriptine kaydedilir.
3. Son mesajlar olduğu gibi korunur.

OpenClaw, bir sıkıştırma bölme noktası seçerken asistan araç çağrılarını bunlarla eşleşen `toolResult` girdileriyle birlikte tutar. Nokta bir araç bloğunun içine denk gelirse OpenClaw, çiftin bir arada kalması ve henüz özetlenmemiş mevcut son kısmın korunması için sınırı kaydırır.

Konuşma geçmişinin tamamı diskte kalır. Sıkıştırma yalnızca modelin sonraki turda gördüklerini değiştirir.

<Note>
Yeni yapılandırmalarda `agents.defaults.compaction.mode` varsayılan olarak `"safeguard"` değerine ayarlanır (daha katı koruma önlemleri ve özet kalitesi denetimleri). Devre dışı bırakmak için `mode: "default"` değerini açıkça ayarlayın.
</Note>

## Otomatik sıkıştırma

Otomatik sıkıştırma varsayılan olarak açıktır. Oturum bağlam sınırına yaklaştığında veya model bir bağlam taşması hatası döndürdüğünde çalışır (bu durumda OpenClaw sıkıştırma yapıp yeniden dener).

Şunları görürsünüz:

- Normal Gateway günlüklerinde `embedded run auto-compaction start` / `complete`.
- Ayrıntılı modda `🧹 Auto-compaction complete`.
- `🧹 Compactions: <count>` değerini gösteren `/status`.

<Info>
OpenClaw, sıkıştırmadan önce aracıya önemli notları [bellek](/tr/concepts/memory) dosyalarına kaydetmesini otomatik olarak hatırlatır. Bu, bağlam kaybını önler.
</Info>

<AccordionGroup>
  <Accordion title="OpenClaw'ın tanıdığı taşma hatası kalıpları">
    OpenClaw, sağlayıcılara özgü düzinelerce taşma hatası dizesini (Anthropic, OpenAI, Bedrock, Gemini, Ollama, OpenRouter ve diğerleri) eşleştirir. Yaygın örnekler:

    - `request_too_large`
    - `context length exceeded`
    - `input exceeds the maximum number of tokens`
    - `input token count exceeds the maximum number of input tokens` (Bedrock)
    - `input is too long for the model`
    - `ollama error: context length exceeded`

  </Accordion>
</AccordionGroup>

## Manuel sıkıştırma

Sıkıştırmayı zorunlu olarak çalıştırmak için herhangi bir sohbete `/compact` yazın. Özeti yönlendirmek için talimat ekleyin:

```text
/compact API tasarımı kararlarına odaklan
```

`agents.defaults.compaction.keepRecentTokens` ayarlandığında (varsayılan: 20,000), manuel sıkıştırma bu kesme noktasına uyar ve yeniden oluşturulan bağlamda son kısmı korur. Açık bir koruma bütçesi olmadan manuel sıkıştırma kesin bir denetim noktası gibi davranır ve yalnızca yeni özetten devam eder.

## Yapılandırma

`openclaw.json` dosyanızda `agents.defaults.compaction` altındaki sıkıştırma ayarlarını yapılandırın. En yaygın ayarlar aşağıda listelenmiştir; tam başvuru için [Oturum yönetimine derinlemesine bakış](/tr/reference/session-management-compaction) sayfasına bakın.

### Farklı bir model kullanma

Sıkıştırma varsayılan olarak aracının birincil modelini kullanır. Özetlemeyi daha yetenekli veya özel amaçlı bir modele devretmek için `agents.defaults.compaction.model` değerini ayarlayın. Geçersiz kılma, bir `provider/model-id` dizesini veya `agents.defaults.models` altında yapılandırılmış yalın bir takma adı kabul eder:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "openrouter/anthropic/claude-sonnet-4-6"
      }
    }
  }
}
```

Yapılandırılmış yalın takma adlar, sıkıştırma başlamadan önce standart sağlayıcı ve model değerlerine çözümlenir. Yalın bir değer hem bir takma adla hem de yapılandırılmış bir değişmez model kimliğiyle eşleşirse değişmez model kimliği öncelikli olur. Eşleşmeyen yalın bir değer, etkin sağlayıcıdaki model kimliği olarak kalır.

Bu, özetlemeye ayrılmış ikinci bir Ollama modeli gibi yerel modellerle de çalışır:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "ollama/llama3.1:8b"
      }
    }
  }
}
```

Ayar belirtilmediğinde sıkıştırma, etkin oturum modeliyle başlar. Özetleme, model geri dönüşüne uygun bir sağlayıcı hatasıyla başarısız olursa OpenClaw bu sıkıştırma denemesini oturumun mevcut model geri dönüş zinciri üzerinden yeniden dener. Geri dönüş seçimi geçicidir ve oturum durumuna geri yazılmaz. Açık bir `agents.defaults.compaction.model` geçersiz kılması kesin olarak uygulanır ve oturum geri dönüş zincirini devralmaz.

### Tanımlayıcıları koruma

Sıkıştırma özetlemesi, opak tanımlayıcıları varsayılan olarak korur (`identifierPolicy: "strict"`). Devre dışı bırakmak için `identifierPolicy: "off"` ile geçersiz kılın. Özel yönlendirme, sıkıştırma sağlayıcısının `summarize()` uygulamasında yer almalıdır.

### Etkin transkript bayt koruması

`agents.defaults.compaction.maxActiveTranscriptBytes` ayarlandığında, transkript geçmişi
bu boyuta ulaşmışsa OpenClaw bir çalıştırmadan önce normal yerel sıkıştırmayı
tetikler. Bu, sağlayıcı tarafındaki bağlam yönetimi model bağlamını sağlıklı
tutarken kalıcı transkript geçmişinin büyümeye devam edebildiği uzun süreli
oturumlar için kullanışlıdır. Ham baytları bölmez; normal sıkıştırma işlem
hattından anlamsal bir özet oluşturmasını ister.

<Warning>
Bayt koruması, etkin SQLite transkript geçmişine uygulanır. Eski JSONL
denetim noktası yapıtları etkin sıkıştırma hedefi değildir.
</Warning>

### Ardıl transkriptler

`agents.defaults.compaction.truncateAfterCompaction` etkinleştirildiğinde OpenClaw, mevcut transkripti yerinde yeniden yazmaz. Sıkıştırma özetinden, korunan durumdan ve özetlenmemiş son kısımdan yeni bir etkin ardıl transkript oluşturur; ardından dal/geri yükleme akışlarını bu sıkıştırılmış ardıla yönlendiren denetim noktası meta verilerini kaydeder.
Ardıl transkriptler ayrıca kısa bir yeniden deneme aralığında gelen, tamamen
aynı uzun kullanıcı turlarını eler; böylece kanal yeniden deneme fırtınaları
sıkıştırmadan sonraki etkin transkripte taşınmaz.

OpenClaw artık yeni sıkıştırmalar için ayrı `.checkpoint.*.jsonl` kopyaları
yazmaz. Mevcut eski denetim noktası dosyaları, bunlara başvurulmaya devam
edildiği sürece kullanılabilir ve normal oturum temizliği sırasında budanır.

### Sıkıştırma bildirimleri

Sıkıştırma varsayılan olarak sessizce çalışır. Sıkıştırma başladığında ve tamamlandığında kısa durum mesajları göstermek ve sıkıştırma öncesi bellek boşaltma denemeleri tükendiğinde ancak yanıt devam ettiğinde işlev kaybı bildirimini göstermek için `notifyUser` değerini ayarlayın:

```json5
{
  agents: {
    defaults: {
      compaction: {
        notifyUser: true,
      },
    },
  },
}
```

### Bellek boşaltma

OpenClaw, kalıcı notları diske kaydetmek için sıkıştırmadan önce **sessiz bir bellek boşaltma** turu çalıştırabilir. Bu bakım turunun etkin konuşma modeli yerine yerel bir model kullanması gerektiğinde `agents.defaults.compaction.memoryFlush.model` değerini ayarlayın:

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

Bellek boşaltma modeli geçersiz kılması kesin olarak uygulanır ve etkin oturum geri dönüş zincirini devralmaz. Ayrıntılar ve yapılandırma için [Bellek](/tr/concepts/memory) sayfasına bakın.

## Takılıp çıkarılabilir sıkıştırma sağlayıcıları

Plugin'ler, Plugin API'sindeki `registerCompactionProvider()` aracılığıyla özel bir sıkıştırma sağlayıcısı kaydedebilir. Bir sağlayıcı kaydedilip yapılandırıldığında OpenClaw, yerleşik LLM işlem hattı yerine özetlemeyi bu sağlayıcıya devreder.

Kayıtlı bir sağlayıcıyı kullanmak için yapılandırmanızda sağlayıcının kimliğini ayarlayın:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "provider": "my-provider"
      }
    }
  }
}
```

Bir `provider` ayarlamak, `mode: "safeguard"` değerini otomatik olarak zorunlu kılar. Sağlayıcılar, yerleşik yolla aynı sıkıştırma talimatlarını ve tanımlayıcı koruma politikasını alır; OpenClaw ayrıca sağlayıcı çıktısından sonra yakın tarihli tur ve bölünmüş tur son ek bağlamını korumaya devam eder.

<Note>
Sağlayıcı başarısız olursa veya boş bir sonuç döndürürse OpenClaw, yerleşik LLM özetlemesine geri döner.
</Note>

## Sıkıştırma ve budama

|                  | Compaction                              | Budama                                    |
| ---------------- | --------------------------------------- | ----------------------------------------- |
| **Ne yapar?**    | Eski konuşmayı özetler                  | Eski araç sonuçlarını kırpar              |
| **Kaydedilir mi?** | Evet (oturum transkriptinde)          | Hayır (yalnızca bellekte, istek başına)   |
| **Kapsam**       | Konuşmanın tamamı                       | Yalnızca araç sonuçları                   |

[Oturum budama](/tr/concepts/session-pruning), özetleme yapmadan araç çıktısını kırpan daha hafif bir tamamlayıcıdır.

## Sorun giderme

**Çok sık mı sıkıştırılıyor?** Modelin bağlam penceresi küçük veya araç çıktıları büyük olabilir. [Oturum budamayı](/tr/concepts/session-pruning) etkinleştirmeyi deneyin.

**Sıkıştırmadan sonra bağlam güncelliğini yitirmiş gibi mi geliyor?** Özeti yönlendirmek için `/compact Focus on <topic>` kullanın veya notların korunması için [bellek boşaltmayı](/tr/concepts/memory) etkinleştirin.

**Temiz bir başlangıç mı gerekiyor?** `/new`, sıkıştırma yapmadan yeni bir oturum başlatır.

Gelişmiş yapılandırma (ayrılmış token'lar, tanımlayıcı koruma, özel bağlam motorları ve OpenAI sunucu tarafı sıkıştırma) için [Oturum yönetimine derinlemesine bakış](/tr/reference/session-management-compaction) sayfasına bakın.

## İlgili konular

- [Oturum](/tr/concepts/session): oturum yönetimi ve yaşam döngüsü.
- [Oturum budama](/tr/concepts/session-pruning): araç sonuçlarını kırpma.
- [Bağlam](/tr/concepts/context): aracı turları için bağlamın nasıl oluşturulduğu.
- [Kancalar](/tr/automation/hooks): sıkıştırma yaşam döngüsü kancaları (`before_compaction`, `after_compaction`).
