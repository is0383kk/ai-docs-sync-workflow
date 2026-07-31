---
read_when:
    - Yerel kişisel ajan güvenilirlik kontrollerini çalıştırma
    - Depo destekli QA senaryo kataloğunu genişletme
    - Hatırlatmayı, yanıtı, belleği, karartmayı, güvenli araç takibini, görev durumunu, güvenle paylaşılabilir tanılamayı, kanıtlarla desteklenen tamamlanma iddialarını ve hata kurtarmayı doğrulama
summary: Gizliliği koruyan kişisel asistan iş akışı kontrolleri için yerel qa-channel senaryoları.
title: Kişisel ajan kıyaslama paketi
x-i18n:
    generated_at: "2026-07-26T22:44:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 35da45e4b22b1044a777fa8d6bce87f9ace377950dd0af3f2419b40cfe4d9be6
    source_path: concepts/personal-agent-benchmark-pack.md
    workflow: 16
---

Kişisel Agent Karşılaştırma Paketi, yerel kişisel asistan iş akışları için
depo destekli küçük bir QA senaryo paketidir. Genel amaçlı bir model karşılaştırması değildir ve
yeni bir çalıştırıcı gerektirmez: özel QA yığınını ([QA genel bakışı](/tr/concepts/qa-e2e-automation)),
sentetik [QA kanalını](/tr/channels/qa-channel) ve mevcut
`qa/scenarios` YAML kataloğunu yeniden kullanır.

## Senaryolar

`qa/scenarios/personal/*.yaml` içinde tanımlanan on senaryo:

| Senaryo kimliği                            | Kontroller                                                                                   |
| ------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `personal-reminder-roundtrip`              | Yerel cron teslimi üzerinden sahte kişisel hatırlatıcılar                                    |
| `personal-channel-thread-reply`            | `qa-channel` üzerinden sahte DM ve ileti dizisi yanıtı yönlendirmesi                          |
| `personal-memory-preference-recall`        | Geçici QA çalışma alanı bellek dosyalarından sahte tercihlerin hatırlanması                   |
| `personal-redaction-no-secret-leak`        | Sahte sırların yinelenmediğine ilişkin kontroller                                             |
| `personal-tool-safety-followthrough`       | Onay tarzı kısa bir etkileşimden sonra güvenli, okuma destekli araç takibi                    |
| `personal-approval-denial-stop`            | Hassas bir yerel okuma isteği için onay reddi sonrasında durma davranışı                      |
| `personal-task-followthrough-status`       | Bekleyen, engellenen ve tamamlanan durumları ayrı tutan, kanıta dayalı görev durumu raporlaması |
| `personal-share-safe-diagnostics-artifact` | Ham kişisel içeriği dışarıda bırakırken yararlı durumu koruyan, güvenle paylaşılabilir tanılama yapıtları |
| `personal-no-fake-progress`                | Yerel kanıt bulunmadan önce sahte ilerleme bildiriminden kaçınan, kanıta dayalı tamamlanma beyanları |
| `personal-failure-recovery`                | Kısmi durumu bildiren ve yeniden deneme sınırlarını net tutan hata kurtarma                   |

Makine tarafından okunabilir paket meta verileri (kimlik listesi, başlık, açıklama)
`extensions/qa-lab/src/scenario-packs.ts` içinde `QA_PERSONAL_AGENT_SCENARIO_IDS` olarak bulunur.
Paketi `--pack personal-agent` ile çalıştırın:

```bash
OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa suite \
  --provider-mode mock-openai \
  --pack personal-agent \
  --concurrency 1
```

`--pack`, yinelenen `--scenario` bayraklarıyla eklemeli çalışır. Açıkça belirtilen senaryolar
önce çalışır; ardından yinelenenler kaldırılarak paket senaryoları
`QA_PERSONAL_AGENT_SCENARIO_IDS` sırasına göre çalışır.

Paket, `mock-openai` veya başka bir yerel QA sağlayıcı
hattıyla `qa-channel` hedefler. Paketi canlı sohbet hizmetlerine veya gerçek kişisel hesaplara yönlendirmeyin.

## Gizlilik Modeli

Senaryolar yalnızca sahte kullanıcıları, sahte tercihleri, sahte sırları ve paket tarafından
oluşturulan geçici QA Gateway çalışma alanını kullanır. Gerçek OpenClaw kullanıcı belleğini,
oturumlarını, kimlik bilgilerini, başlatma agent'larını, genel yapılandırmaları veya canlı
Gateway durumunu okumamalı ya da bunlara yazmamalıdır.

Yapıtlar mevcut QA paketi yapıt dizini altında kalır ve test çıktısı
olarak değerlendirilir. Redaksiyon kontrolleri sahte işaretleyiciler kullandığından hataları
incelemek ve sorunlara kaydetmek güvenlidir.

## Paketi genişletme

`qa/scenarios/personal/` altına yeni `.yaml` durumları ekleyin, ardından senaryo kimliğini
`QA_PERSONAL_AGENT_SCENARIO_IDS` öğesine ekleyin. Her durumu küçük, yerel,
`mock-openai` içinde deterministik ve tek bir kişisel asistan davranışına odaklı tutun.

İyi takip adayları: redakte edilmiş yörünge dışa aktarma kontrolleri, yalnızca yerel
Plugin iş akışı kontrolleri.

Senaryo kataloğunda bu yüzeyi gerekçelendirecek kadar kararlı durum bulunana kadar
yeni bir çalıştırıcı, Plugin, bağımlılık, canlı aktarım veya model değerlendiricisi eklemekten kaçının.
