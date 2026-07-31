---
read_when:
    - Oturumlar ve kanallar arasında çalışan kalıcı bellek istiyorsunuz
    - Yapay zekâ destekli hatırlama ve kullanıcı modelleme istiyorsunuz
summary: Honcho Plugin'i aracılığıyla yapay zekâ tabanlı oturumlar arası bellek
title: Honcho belleği
x-i18n:
    generated_at: "2026-07-26T23:15:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fadcf6d8e2505ab4fe6a81340695b7c8fee49c3cb4889665af13389941619117
    source_path: concepts/memory-honcho.md
    workflow: 16
---

[Honcho](https://honcho.dev), harici bir plugin aracılığıyla OpenClaw'a yapay zekâya özgü bellek ekler. Konuşmaları özel bir hizmette kalıcı hâle getirir ve zaman içinde kullanıcı ile ajan modelleri oluşturarak ajanınıza çalışma alanındaki Markdown dosyalarının ötesine geçen oturumlar arası bağlam sağlar.

## Sağladıkları

- **Oturumlar arası bellek** - konuşmalar her etkileşimden sonra kalıcı hâle gelir; böylece bağlam, oturum sıfırlamaları, Compaction ve kanal geçişleri boyunca korunur.
- **Kullanıcı modelleme** - Honcho, her kullanıcı (tercihler, olgular, iletişim tarzı) ve ajan (kişilik, öğrenilmiş davranışlar) için bir profil tutar.
- **Anlamsal arama** - yalnızca geçerli oturumda değil, geçmiş konuşmalardan elde edilen gözlemlerde de arama yapar.
- **Çok ajanlı farkındalık** - üst ajanlar, oluşturulan alt ajanları otomatik olarak izler ve alt oturumlara gözlemci olarak eklenir.

## Kullanılabilir araçlar

Honcho, ajanın konuşma sırasında kullanabileceği araçları kaydeder:

**Veri alma (hızlı, LLM çağrısı yok):**

| Araç                        | İşlevi                                           |
| --------------------------- | ------------------------------------------------------ |
| `honcho_context`            | Oturumlar genelindeki eksiksiz kullanıcı temsili               |
| `honcho_search_conclusions` | Saklanan sonuçlar üzerinde anlamsal arama                |
| `honcho_search_messages`    | Oturumlar genelinde mesajları bulur (gönderene ve tarihe göre filtreleme) |
| `honcho_session`            | Geçerli oturumun geçmişi ve özeti                    |

**Soru-cevap (LLM destekli):**

| Araç         | İşlevi                                                              |
| ------------ | ------------------------------------------------------------------------- |
| `honcho_ask` | Kullanıcı hakkında soru sorar. Olgular için `depth='quick'`, sentez için `'thorough'` |

## Başlarken

Plugin'i yükleyin ve kurulumu çalıştırın:

```bash
openclaw plugins install @honcho-ai/openclaw-honcho
openclaw honcho setup
openclaw gateway --force
```

Kurulum komutu API kimlik bilgilerinizi ister, yapılandırmayı yazar ve isteğe bağlı olarak mevcut çalışma alanı bellek dosyalarını taşır.

<Info>
Honcho tamamen yerel olarak (kendi sunucunuzda) veya `api.honcho.dev` adresindeki yönetilen API üzerinden çalışabilir. Kendi sunucunuzda çalıştırma seçeneği için harici bağımlılık gerekmez.
</Info>

## Yapılandırma

Ayarlar `plugins.entries["openclaw-honcho"].config` altında bulunur:

```json5
{
  plugins: {
    entries: {
      "openclaw-honcho": {
        config: {
          apiKey: "your-api-key", // kendi sunucunuzda çalıştırıyorsanız atlayın
          workspaceId: "openclaw", // bellek yalıtımı
          baseUrl: "https://api.honcho.dev",
        },
      },
    },
  },
}
```

Kendi sunucunuzdaki örneklerde `baseUrl` değerini yerel sunucunuza (örneğin `http://localhost:8000`) yönlendirin ve API anahtarını atlayın.

## Mevcut belleği taşıma

Mevcut çalışma alanı bellek dosyalarınız (`USER.md`, `MEMORY.md`, `IDENTITY.md`, `memory/`, `canvas/`) varsa `openclaw honcho setup` bunları algılar ve taşıma seçeneği sunar.

<Info>
Taşıma işlemi tahribatsızdır; dosyalar Honcho'ya yüklenir. Orijinal dosyalar hiçbir zaman silinmez veya taşınmaz.
</Info>

## Çalışma şekli

Her yapay zekâ etkileşiminden sonra konuşma Honcho'da kalıcı hâle getirilir. Hem kullanıcı hem de ajan mesajları gözlemlenir; böylece Honcho zaman içinde modellerini oluşturabilir ve iyileştirebilir.

Honcho araçları, konuşma sırasında OpenClaw'ın `before_prompt_build` plugin kancasında hizmeti sorgulayarak model istemi görmeden önce ilgili bağlamı ekler.

## Honcho ve yerleşik bellek karşılaştırması

|                   | Yerleşik / QMD                | Honcho                              |
| ----------------- | ---------------------------- | ----------------------------------- |
| **Depolama**       | Çalışma alanı Markdown dosyaları     | Özel hizmet (yerel veya barındırılan) |
| **Oturumlar arası** | Bellek dosyaları aracılığıyla             | Otomatik, yerleşik                 |
| **Kullanıcı modelleme** | Manuel (MEMORY.md dosyasına yazma)  | Otomatik profiller                  |
| **Arama**        | Vektör + anahtar kelime (karma)    | Gözlemler üzerinde anlamsal          |
| **Çok ajanlı**   | İzlenmez                  | Üst/alt ajan farkındalığı              |
| **Bağımlılıklar**  | Yok (yerleşik) veya QMD ikili dosyası | Plugin kurulumu                      |

Honcho ve yerleşik bellek sistemi birlikte çalışabilir. QMD yapılandırıldığında, Honcho'nun oturumlar arası belleğinin yanı sıra yerel Markdown dosyalarında arama yapmak için ek araçlar kullanılabilir hâle gelir.

## CLI komutları

```bash
openclaw honcho setup                        # API anahtarını yapılandırın ve dosyaları taşıyın
openclaw honcho status                       # Bağlantı durumunu kontrol edin
openclaw honcho ask <question>               # Honcho'ya kullanıcı hakkında sorgu gönderin
openclaw honcho search <query> [-k N] [-d D] # Bellekte anlamsal arama yapın
```

## Ek okumalar

- [Plugin kaynak kodu](https://github.com/plastic-labs/openclaw-honcho)
- [Honcho belgeleri](https://docs.honcho.dev)
- [Honcho OpenClaw entegrasyon kılavuzu](https://docs.honcho.dev/v3/guides/integrations/openclaw)

## İlgili konular

- [Belleğe genel bakış](/tr/concepts/memory)
- [Yerleşik bellek motoru](/tr/concepts/memory-builtin)
- [QMD bellek motoru](/tr/concepts/memory-qmd)
- [Bağlam Motorları](/tr/concepts/context-engine)
