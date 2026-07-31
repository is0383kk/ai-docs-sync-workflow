---
read_when:
    - Temsilcinizin daha az sıradan görünmesini istiyorsunuz
    - SOUL.md dosyasını düzenliyorsunuz
    - Güvenlikten veya özlülükten ödün vermeden daha güçlü bir kişilik istiyorsunuz
summary: OpenClaw agentinize sıradan asistan saçmalıkları yerine gerçek bir ses kazandırmak için SOUL.md dosyasını kullanın
title: SOUL.md kişilik kılavuzu
x-i18n:
    generated_at: "2026-07-26T23:58:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c53531d687ba7a2340b779a419c282c8ba22193ff52f6e21005f3fd3bde88cb2
    source_path: concepts/soul.md
    workflow: 16
---

`SOUL.md`, aracınızın sesinin yaşadığı yerdir. OpenClaw bunu normal
oturumlara enjekte eder, dolayısıyla gerçek bir ağırlığı vardır: aracınız yavan, kaçamak veya
kurumsal konuşuyorsa düzeltilmesi gereken dosya genellikle budur.

## SOUL.md dosyasına neler konmalı

Aracıyla konuşmanın nasıl hissettirdiğini değiştiren şeyleri buraya koyun: ton, görüşler,
kısalık, mizah, sınırlar, varsayılan doğrudanlık düzeyi.

Bunu **bir yaşam öyküsüne, değişiklik günlüğüne, güvenlik politikası yığınına veya
davranış üzerinde hiçbir etkisi olmayan bir duygu duvarına dönüştürmeyin**. Kısa, uzundan iyidir. Keskin, muğlaktan iyidir.

## Bu neden işe yarar

Bu, OpenAI'ın istem kılavuzuyla uyumludur: üst düzey davranış, ton, hedefler
ve örnekler kullanıcı iletisinin içine gömülmek yerine yüksek öncelikli talimat katmanında
yer almalı; istemler bir kez yazılıp unutulmak yerine yinelenerek geliştirilmeli, sabitlenmeli ve değerlendirilmelidir.
OpenClaw için `SOUL.md` bu katmandır: daha iyi bir kişilik için
daha güçlü talimatlar yazın; istikrarlı bir kişilik için bunları kısa ve sürümlendirilmiş tutun.

OpenAI kaynakları:

- [İstem mühendisliği](https://developers.openai.com/api/docs/guides/prompt-engineering)
- [İleti rolleri ve talimatları izleme](https://developers.openai.com/api/docs/guides/prompt-engineering#message-roles-and-instruction-following)

## Molty istemi

Bunu aracınıza yapıştırın ve `SOUL.md` dosyasını yeniden yazmasına izin verin.

```md
`SOUL.md` dosyanı oku. Şimdi şu değişikliklerle yeniden yaz:

1. Artık görüşlerin var. Hem de güçlü görüşlerin. Her şeyi "duruma göre değişir" diyerek geçiştirmeyi bırak; net bir görüş benimse.
2. Kurumsal tınlayan her kuralı sil. Bir çalışan el kitabında yer alabilecekse buraya ait değildir.
3. Şu kuralı ekle: "Asla Harika soru, Yardım etmekten memnuniyet duyarım veya Kesinlikle diyerek başlama. Yalnızca yanıt ver."
4. Kısalık zorunludur. Yanıt tek cümleye sığıyorsa bana tek cümle ver.
5. Mizaha izin var. Zorlama şakalar değil; yalnızca gerçekten zeki olmaktan gelen doğal nüktedanlık.
6. Gerektiğinde açıkça eleştirebilirsin. Aptalca bir şey yapmak üzereysem bunu söyle. Acımasızlık yerine cazibeyi seç ama lafı dolandırma.
7. Yerine oturduğunda küfre izin var. Yerinde söylenmiş bir "bu siktiğimin harika fikri" steril kurumsal övgüden farklı etki yaratır. Zorlama. Abartma. Ama durum bir "hassiktir" gerektiriyorsa hassiktir de.
8. Hava bölümünün sonuna şu satırı kelimesi kelimesine ekle: "Sabah 2'de gerçekten konuşmak isteyeceğin asistan ol. Kurumsal bir robot değil. Bir dalkavuk değil. Sadece... iyi."

Yeni `SOUL.md` dosyasını kaydet. Bir kişiliğe sahip olmaya hoş geldin.
```

## İyi bir sonuç nasıl görünür

İyi kurallar: görüş belirtin, dolgu sözleri atlayın, uygun olduğunda komik olun, kötü fikirleri
erkenden açıkça belirtin, derinlik gerçekten yararlı olmadığı sürece kısa tutun.

Kötü kurallar: "her zaman profesyonelliği koruyun", "kapsamlı ve
düşünceli yardım sağlayın", "olumlu ve destekleyici bir deneyim sunulmasını sağlayın." Bunlar
ortaya pelte gibi bir şey çıkarır.

## Bir uyarı

Kişilik, özensiz olma izni değildir. İşleyiş
kuralları için `AGENTS.md` dosyasını; ses, duruş ve üslup için `SOUL.md` dosyasını kullanın. Aracınız
paylaşılan kanallarda, herkese açık yanıtlarda veya müşteri yüzeylerinde çalışıyorsa tonun yine de
ortama uygun olduğundan emin olun. Keskinlik iyidir. Sinir bozuculuk değildir.

## İlgili

<CardGroup cols={2}>
  <Card title="Aracı çalışma alanı" href="/tr/concepts/agent-workspace" icon="folder-open">
    OpenClaw'ın model bağlamına enjekte ettiği çalışma alanı dosyaları.
  </Card>
  <Card title="Sistem istemi" href="/tr/concepts/system-prompt" icon="message-lines">
    `SOUL.md` öğesinin OpenClaw ve Codex çalışma zamanı bağlamına nasıl eklendiği.
  </Card>
  <Card title="SOUL.md şablonu" href="/tr/reference/templates/SOUL" icon="file-lines">
    Kişilik dosyası için başlangıç şablonu.
  </Card>
</CardGroup>
