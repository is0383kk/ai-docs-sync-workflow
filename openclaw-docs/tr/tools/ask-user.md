---
read_when:
    - Bir aracının kullanıcıya yapılandırılmış bir soru sormasını istiyorsunuz
    - Bir ask_user istemini yanıtlıyor veya hata ayıklıyorsunuz
    - ask_user şemasına, zaman aşımına veya kanal davranışına ihtiyacınız var
summary: ask_user yapılandırılmış bir insan kararı için bir agent turunu nasıl duraklatır?
title: Kullanıcıya sor
x-i18n:
    generated_at: "2026-07-26T23:37:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 32556314a34c26054c3aabfdd8ecc474cf85196e5cc71adb833face596edbd24
    source_path: tools/ask-user.md
    workflow: 16
---

`ask_user`, ajanın insana bir ila üç yapılandırılmış soru sormasını ve
yanıtları beklemesini sağlar. Rutin onaylar veya ajanın istekten, koddan ya da
makul bir varsayılandan çıkarabileceği bilgiler için değil, gerçekten kullanıcıya
ait kararlar içindir.

Araç yalnızca ana oturumda kullanılabilir. Alt ajanlar ve birincil olmayan diğer
çalıştırmalar bu araca erişemez.

## Bir soruyu yanıtlama

Desteklenen herhangi bir konuşma yüzeyinden yanıt verebilirsiniz:

- Web Control UI, soru panelini doğrudan oluşturucunun üzerine sabitler. Birden
  çok sorulu istemlerde panel, soruları kısa bir adım göstergesiyle ilerleyerek
  teker teker gösterir. Çözümlendikten sonra panel kapanır ve sohbette yalnızca
  kısa bir yanıt özeti kalır.
- Telegram, Discord ve Slack; tek seçimli, tek sorulu bir istem için yerel
  düğmeler oluşturur.
- Düz metin yanıtı her kanalda çalışır. Bir sayı, seçenek etiketi veya kendi
  yanıtınızla cevap verin.

OpenClaw, serbest metinli **Diğer** yanıtını her zaman etkinleştirir. Ajan,
hazırlanan seçenek listesine `Other` seçeneğini eklememelidir.

## Platform davranışı

Yanıtlar, desteklenen her konuşma yüzeyinde çalışır. Web Control UI, genişletildiğinde
oluşturucunun yerini alan sabitlenmiş bir adım göstergesi kullanır; daraltıldığında
ince bir soru çubuğunun altında tam oluşturucu geri gelir. iOS, macOS ve Android
satır içi kartlar gösterir; birden çok soru, dokunmatik kullanıma elverişli ve
kasıtlı bir düzen olarak üst üste kalır. Her platform, soru-yanıt özetini etkin
sohbet zaman çizelgesinde süreli olarak kaldırmadan tutar ve **Atla** her yerde
kullanılabilir.

Birden çok sorulu ve çoklu seçimli istemler dâhil olmak üzere yerel düğmeleri
kullanamayan istemler, kanallarda okunabilir metne indirgenir. Control UI,
yapılandırılmış adım göstergesinin tamamını korur.

## Zaman aşımı ve yanıt verilmemesi

Varsayılan zaman aşımı 900 saniyedir. `timeoutSeconds`, 30 ile 3600 saniye
aralığıyla sınırlandırılır.

Soru, yanıt gelmeden önce zaman aşımına uğrar veya iptal edilirse araç
`status: "no_answer"` döndürür. Ardından ajan kendi en iyi muhakemesiyle devam eder.
Yarıda kesilen bir ajan çalıştırması, bekleyen Gateway sorusunu iptal eder.

## Araç şeması

```ts
{
  questions: Array<{
    id: string; // benzersiz snake_case yanıt anahtarı
    header: string; // kısa etiket; 12 karakterle sınırlandırılır
    question: string; // tek cümle
    options: Array<{
      label: string;
      description?: string;
    }>; // 2-4 seçenek
    multiSelect?: boolean;
  }>; // 1-3 soru
  timeoutSeconds?: number; // tam sayı; varsayılan 900, 30-3600 aralığıyla sınırlandırılır
}
```

`multiSelect: true` ile kullanıcı birden fazla seçenek belirleyebilir. Yanıt
değerleri her soru için bir dizi olarak döndürülür.

Yanıtlanmış sonuç örneği:

```json
{
  "status": "answered",
  "answers": {
    "answers": {
      "deploy_target": ["Staging (Recommended)"]
    }
  }
}
```

## Model yönlendirmesi

Modele yönelik sözleşme, ajana şunları söyler:

- yalnızca gerçekten kullanıcıya ait bir karar nedeniyle engellendiğinde sormak;
- tek soruyu tercih etmek ve en fazla üç soru kullanmak;
- önerilen seçeneği ilk sıraya koymak ve etiketinin sonuna `(Recommended)` eklemek;
- serbest metin otomatik olarak eklendiğinden hazırlanmış bir `Other` seçeneğini dâhil etmemek;
- `no_answer` sonrasında en iyi muhakemeyle devam etmek.

Ajan, devam edip edemeyeceğini sormak veya kendi planını onaylatmak için
`ask_user` kullanmamalıdır.
