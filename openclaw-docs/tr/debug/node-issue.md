---
read_when:
    - Eksik bir __name yardımcısından söz eden tsx/esbuild yükleyici çökmesini araştırma
summary: Geçmişteki Node + tsx "__name is not a function" çökmesi ve nedeni
title: Node + tsx çökmesi
x-i18n:
    generated_at: "2026-07-26T22:44:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97d2f62d24860cee65753027ba84c14c8d4ffb910ee17bb0032cf0409c427589
    source_path: debug/node-issue.md
    workflow: 16
---

# Node + tsx "\_\_name is not a function" çökmesi

## Durum

Çözüldü. Bu çökme, `package.json` içinde sabitlenen mevcut `tsx`
sürümünde (`4.22.3`) veya güncel Node sürümlerinde yeniden oluşmuyor. Gelecekteki bir
`tsx`/esbuild yükseltmesinin sorunu yeniden ortaya çıkarması ihtimaline karşı burada tutulmuştur.

## İlk belirti

OpenClaw geliştirme betiklerini `tsx` üzerinden çalıştırmak, başlangıçta şu hatayla başarısız oluyordu:

```text
[openclaw] CLI başlatılamadı: TypeError: __name is not a function
    at createSubsystemLogger (src/logging/subsystem.ts)
    at <caller> (src/agents/auth-profiles/constants.ts)
```

Satır numaraları verilmemiştir; ilk çökmeden bu yana her iki dosya da değiştiğinden
belirtilen satırlar artık eşleşmemektedir.

Bu sorun, Bun'ı isteğe bağlı hâle getirmek amacıyla geliştirme betiklerinin Bun yerine
`tsx` kullanmaya geçmesinden sonra (`2871657e`, 2026-01-06) ortaya çıktı.
Bun tabanlı eşdeğer yol çökmüyordu. Sorun ilk olarak macOS üzerinde Node v25.3.0 ile
gözlemlendi; Node 25 çalıştıran diğer platformların da etkilenmesi olası kabul edildi.

## Neden

`tsx`, dönüşüm seçeneklerinde sabit kodlanmış `keepNames: true` ile TS/ESM'yi
esbuild üzerinden dönüştürür. Bu ayar, `fn.name` küçültme ve paketleme sırasında
korunsun diye esbuild'in adlandırılmış işlev/sınıf bildirimlerini bir `__name`
yardımcısı çağrısıyla sarmalamasına neden olur. Çökme, etkilenen `tsx`/Node
birleşiminde söz konusu modülün çağrı noktasında yardımcının eksik olduğu veya gölgelendiği
anlamına gelir; bu nedenle `__name(...)`, sarılmış değeri döndürmek yerine hata fırlattı.

## Güncel yeniden oluşturma kontrolü

```bash
node --version
pnpm install
node --import tsx src/entry.ts status
```

Asgari yalıtılmış yeniden oluşturma (yalnızca ilk yığın izlemesindeki modülü yükler):

```bash
node --import tsx scripts/repro/tsx-name-repro.ts
```

Her iki komut da şu anda hatasız sonlanıyor. Bunlardan biri yeniden `__name is not a
function`
hatası fırlatırsa üst projeye bildirmeden önce tam Node sürümünü, `tsx`
sürümünü (`node_modules/tsx/package.json`) ve eksiksiz yığın izlemesini kaydedin.

## Geçici çözümler (çökme geri dönerse)

- Geliştirme betiklerini `node --import tsx` yerine Bun ile çalıştırın.
- Tür denetimi için `pnpm tsgo` komutunu çalıştırın, ardından
  kaynağı `tsx` üzerinden çalıştırmak yerine derlenmiş çıktıyı çalıştırın:

  ```bash
  pnpm tsgo
  node openclaw.mjs status
  ```

- Paketlediği esbuild sürümünün hatayı yeniden ortaya çıkarıp çıkarmadığını
  ikiye bölerek belirlemek için farklı bir `tsx` sürümü deneyin
  (`pnpm add -D tsx@<version>` bir bağımlılık değişikliğidir ve depo politikası gereği onay gerektirir).
- Hatanın sürüme özgü olup olmadığını görmek için farklı bir Node
  ana/alt sürümünde test edin.

## Kaynaklar

- [https://esbuild.github.io/api/#keep-names](https://esbuild.github.io/api/#keep-names)
- [https://github.com/evanw/esbuild/issues/1031](https://github.com/evanw/esbuild/issues/1031)

## İlgili

- [Node.js kurulumu](/tr/install/node)
- [Gateway sorun giderme](/tr/gateway/troubleshooting)
