---
read_when:
    - Sie aktualisieren eine Konfiguration, die abgeleitete Zusicherungen verwendet hat
    - Sie möchten zuvor gespeicherte Nachverfolgungseinträge prüfen oder verwerfen
sidebarTitle: Commitments
summary: Status- und Bereinigungshinweise für eingestellte abgeleitete Folgevereinbarungen
title: Abgeleitete Verpflichtungen
x-i18n:
    generated_at: "2026-07-26T17:44:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cfaa8c44be4ffb8db48279dba5347d4f598a193bfc4e244aeaed7a93e00ffb79
    source_path: concepts/commitments.md
    workflow: 16
---

Das Experiment zu abgeleiteten Verpflichtungen wurde eingestellt. OpenClaw extrahiert keine neuen
Folgeaufgaben mehr aus Konversationen und stellt sie nicht mehr über Heartbeat zu; der frühere
`commitments`-Konfigurationsblock wird durch `openclaw doctor --fix` entfernt.

Exakte Erinnerungen und geplante Arbeiten verwenden weiterhin
[geplante Aufgaben](/de/automation/cron-jobs). Dauerhafte Fakten aus Konversationen gehören in den
[Speicher](/de/concepts/memory).

## Vorhandene Einträge

Zuvor gespeicherte Verpflichtungen verbleiben in der gemeinsamen SQLite-Zustandsdatenbank, damit ein
Upgrade den für Betreiber sichtbaren Verlauf nicht löscht. Verwenden Sie die Legacy-Wartungs-CLI, um diese Zeilen zu prüfen oder zu verwerfen:

```bash
openclaw commitments --all
openclaw commitments dismiss cm_abc123
```

Die Referenz zum Wartungsbefehl finden Sie unter [`openclaw commitments`](/de/cli/commitments).

## Verwandte Themen

- [Geplante Aufgaben](/de/automation/cron-jobs)
- [Speicherübersicht](/de/concepts/memory)
- [Heartbeat](/de/gateway/heartbeat)
