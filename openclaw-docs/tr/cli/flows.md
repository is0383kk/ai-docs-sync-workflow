---
read_when:
    - Eski belgelerde veya sürüm notlarında `openclaw flows` ile karşılaşabilirsiniz
    - Hızlı bir TaskFlow inceleme başvurusu istiyorsunuz
summary: 'Yönlendirme: akış komutları `openclaw tasks flow` altında bulunur'
title: Akışlar (yönlendirme)
x-i18n:
    generated_at: "2026-07-26T23:13:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 05d27154190d6087649612d81ce15f0cbc9459aa89ab22211582c18f4fc2943c
    source_path: cli/flows.md
    workflow: 16
---

# `openclaw tasks flow`

Üst düzey bir `openclaw flows` komutu yoktur. Kalıcı TaskFlow incelemesi `openclaw tasks flow` altında yer alır.

## Alt komutlar

```bash
openclaw tasks flow list   [--json] [--status <name>]
openclaw tasks flow show   <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

| Alt komut | Açıklama                | Argümanlar / seçenekler                                                                   |
| ---------- | -------------------------- | ------------------------------------------------------------------------------------- |
| `list`     | İzlenen TaskFlow'ları listeler.    | Makine tarafından okunabilir çıktı için `--json`; filtre için `--status <name>` (aşağıdaki durum değerlerine bakın). |
| `show`     | Bir TaskFlow'u gösterir.         | Akış kimliği veya sahip anahtarı için `<lookup>`; makine tarafından okunabilir çıktı için `--json`.                    |
| `cancel`   | Çalışan bir TaskFlow'u iptal eder. | Akış kimliği veya sahip anahtarı için `<lookup>`.                                                      |

`<lookup>`, bir akış kimliğini (`list` / `show` tarafından döndürülür) veya akışın sahip anahtarını (akışın sahibi olan alt sistemin akışı izlemek için kullandığı kararlı tanımlayıcı) kabul eder.

### Durum filtresi değerleri

`list` üzerindeki `--status` şunlardan birini kabul eder: `queued`, `running`, `waiting`, `blocked`, `succeeded`, `failed`, `cancelled`, `lost`.

## Örnekler

```bash
openclaw tasks flow list
openclaw tasks flow list --status running
openclaw tasks flow list --json
openclaw tasks flow show flow_abc123
openclaw tasks flow show flow_abc123 --json
openclaw tasks flow cancel flow_abc123
```

TaskFlow kavramları ve yazımı için [TaskFlow](/tr/automation/taskflow) sayfasına bakın. Üst `tasks` komutu için [tasks CLI referansı](/tr/cli/tasks) sayfasına bakın.

## İlgili

- [CLI referansı](/tr/cli)
- [Otomasyon](/tr/automation)
- [TaskFlow](/tr/automation/taskflow)
