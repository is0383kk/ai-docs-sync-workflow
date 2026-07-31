---
read_when:
    - Çıkarımlanan takip taahhütlerini incelemek istiyorsunuz
    - Bekleyen yoklamaları kapatmak istiyorsunuz
    - Heartbeat'in neler iletebileceğini denetliyorsunuz
summary: '`openclaw commitments` için CLI başvurusu (çıkarılan takip işlemlerini inceleme ve kapatma)'
title: '`openclaw commitments`'
x-i18n:
    generated_at: "2026-07-26T23:12:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4a7c573daad6a9bc6ce4532514c8cc22b3c510b4fc0cf9d1a79048413f08c1a2
    source_path: cli/commitments.md
    workflow: 16
---

Kullanımdan kaldırılan çıkarımsal taahhütler denemesinden kalan kayıtları inceleyin ve kapatın.
OpenClaw artık yeni taahhütler oluşturmaz veya teslim etmez, ancak yükseltmelerin mevcut
SQLite satırlarını denetleyip temizleyebilmesi için bakım komutunu korur.

Alt komut verilmediğinde, `openclaw commitments` bekleyen taahhütleri listeler.

## Kullanım

```bash
openclaw commitments [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments list [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments dismiss <id...> [--json]
```

## Seçenekler

- `--all`: yalnızca bekleyen taahhütler yerine tüm durumları gösterir.
- `--agent <id>`: tek bir aracı kimliğine göre filtreler.
- `--status <status>`: duruma göre filtreler. Değerler: `pending`, `sent`,
  `dismissed`, `snoozed` veya `expired`. Bilinmeyen değerlerde hata verilerek çıkılır.
- `--json`: makine tarafından okunabilir JSON çıktısı üretir.

`dismiss`, belirtilen taahhüt kimliklerini `dismissed` olarak işaretler.

## Örnekler

Bekleyen taahhütleri listeleyin:

```bash
openclaw commitments
```

Saklanan tüm taahhütleri listeleyin:

```bash
openclaw commitments --all
```

Tek bir aracıya göre filtreleyin:

```bash
openclaw commitments --agent main
```

Ertelenen taahhütleri bulun:

```bash
openclaw commitments --status snoozed
```

Bir veya daha fazla taahhüdü kapatın:

```bash
openclaw commitments dismiss cm_abc123 cm_def456
```

JSON olarak dışa aktarın:

```bash
openclaw commitments --all --json
```

## Çıktı

Metin çıktısı; taahhüt sayısını, paylaşılan SQLite veritabanı yolunu, etkin filtreleri
ve her taahhüt için bir satırı yazdırır:

- taahhüt kimliği
- durum
- tür (`event_check_in`, `deadline_check`, `care_check_in` veya `open_loop`)
- en erken son tarih
- kapsam (aracı/kanal/hedef)
- önerilen durum kontrolü metni

JSON çıktısı; sayıyı, etkin durum ve aracı filtrelerini, paylaşılan
SQLite veritabanı yolunu ve saklanan kayıtların tamamını içerir.

## İlgili

- [Çıkarımsal taahhütler](/tr/concepts/commitments)
- [Belleğe genel bakış](/tr/concepts/memory)
- [Heartbeat](/tr/gateway/heartbeat)
- [Zamanlanmış görevler](/tr/automation/cron-jobs)
