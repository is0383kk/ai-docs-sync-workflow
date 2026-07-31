---
read_when:
    - Çıkarıma dayalı taahhütler kullanan bir yapılandırmayı yükseltiyorsunuz
    - Daha önce saklanan takip kayıtlarını incelemek veya kapatmak istiyorsunuz
sidebarTitle: Commitments
summary: Kullanımdan kaldırılan, çıkarıma dayalı takip taahhütleri için durum ve temizleme kılavuzu
title: Çıkarılan taahhütler
x-i18n:
    generated_at: "2026-07-26T23:14:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cfaa8c44be4ffb8db48279dba5347d4f598a193bfc4e244aeaed7a93e00ffb79
    source_path: concepts/commitments.md
    workflow: 16
---

Çıkarılan taahhütler deneyi kullanımdan kaldırılmıştır. OpenClaw artık yeni
konuşma takiplerini çıkarmaz veya bunları Heartbeat aracılığıyla iletmez ve önceki
`commitments` yapılandırma bloğu `openclaw doctor --fix` tarafından kaldırılır.

Kesin hatırlatıcılar ve zamanlanmış çalışmalar için
[zamanlanmış görevler](/tr/automation/cron-jobs) kullanılmaya devam eder. Kalıcı konuşma olguları
[bellekte](/tr/concepts/memory) tutulmalıdır.

## Mevcut kayıtlar

Önceden depolanan taahhütler, yükseltme sırasında operatörün görebildiği geçmişin
yok edilmemesi için paylaşılan SQLite durum veritabanında kalır. Bu satırları incelemek
veya kapatmak için eski bakım CLI'sini kullanın:

```bash
openclaw commitments --all
openclaw commitments dismiss cm_abc123
```

Bakım komutu başvurusu için [`openclaw commitments`](/tr/cli/commitments)
sayfasına bakın.

## İlgili

- [Zamanlanmış görevler](/tr/automation/cron-jobs)
- [Belleğe genel bakış](/tr/concepts/memory)
- [Heartbeat](/tr/gateway/heartbeat)
