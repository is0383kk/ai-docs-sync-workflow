---
read_when:
    - BOOT.md kontrol listesi ekleme
summary: BOOT.md için çalışma alanı şablonu
title: BOOT.md şablonu
x-i18n:
    generated_at: "2026-07-26T23:40:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1adfb4d71f1f03716a1ddc4774a4cb6ead4b8be65bd9bb34066a9e1929a36b21
    source_path: reference/templates/BOOT.md
    workflow: 16
---

# BOOT.md

Buraya kısa ve açık başlangıç talimatları ekleyin. Birlikte sunulan `boot-md` hook'u, dosya mevcutsa ve boşluk dışı içerik barındırıyorsa Gateway her başlatıldığında bu dosyayı her agent çalışma alanı için bir kez çalıştırır. Bir çalışma alanını paylaşan birden fazla agent yalnızca tek bir çalıştırmayı tetikler.

Hook devre dışı olarak sunulur. Önce etkinleştirin:

```bash
openclaw hooks enable boot-md
```

Bir kontrol listesi öğesi mesaj gönderiyorsa mesaj aracını kullanın, ardından tam sessiz token `NO_REPLY` ile yanıt verin (büyük/küçük harfe duyarlı değildir).

## İlgili

- [Agent çalışma alanı](/tr/concepts/agent-workspace)
- [Hook'lar](/tr/automation/hooks#boot-md)
