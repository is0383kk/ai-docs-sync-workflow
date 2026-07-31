---
read_when:
    - Sie möchten Aufzeichnungen zu Hintergrundaufgaben überprüfen, auditieren oder abbrechen
    - Sie dokumentieren TaskFlow-Befehle unter `openclaw tasks flow`
summary: CLI-Referenz für `openclaw tasks` (Hintergrundaufgaben-Ledger und TaskFlow-Status)
title: '`openclaw tasks`'
x-i18n:
    generated_at: "2026-07-26T18:53:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b03a4aa9fab12b6e5773259a76a1e89fd6e6398c73e5b0533a31e5e3a3894f9c
    source_path: cli/tasks.md
    workflow: 16
---

Untersuchen Sie dauerhafte Hintergrundaufgaben und den TaskFlow-Status. Ohne Unterbefehl
entspricht `openclaw tasks` dem Befehl `openclaw tasks list`.

Unter [Hintergrundaufgaben](/de/automation/tasks) finden Sie das Lebenszyklus- und Zustellungsmodell
sowie im Abschnitt `tasks audit` vollständige Beschreibungen der Befunde.

## Verwendung

```bash
openclaw tasks
openclaw tasks list
openclaw tasks list --runtime acp
openclaw tasks list --status running
openclaw tasks show <lookup>
openclaw tasks notify <lookup> state_changes
openclaw tasks cancel <lookup>
openclaw tasks audit
openclaw tasks maintenance
openclaw tasks maintenance --apply
openclaw tasks flow list
openclaw tasks flow show <lookup>
openclaw tasks flow cancel <lookup>
```

## Stammoptionen

| Flag               | Beschreibung                                                                                       |
| ------------------ | -------------------------------------------------------------------------------------------------- |
| `--json`           | Gibt JSON aus.                                                                                     |
| `--runtime <name>` | Filtert nach Art: `subagent`, `acp`, `cron` oder `cli`.                                               |
| `--status <name>`  | Filtert nach Status: `queued`, `running`, `succeeded`, `failed`, `timed_out`, `cancelled` oder `lost`. |

## Unterbefehle

### `list`

```bash
openclaw tasks list [--runtime <name>] [--status <name>] [--json]
```

Listet verfolgte Hintergrundaufgaben absteigend nach Aktualität auf.

### `show`

```bash
openclaw tasks show <lookup> [--json]
```

Zeigt eine Aufgabe anhand der Aufgaben-ID, Ausführungs-ID oder des Sitzungsschlüssels an.

### `notify`

```bash
openclaw tasks notify <lookup> <done_only|state_changes|silent>
```

Ändert die Benachrichtigungsrichtlinie für eine laufende Aufgabe.

### `cancel`

```bash
openclaw tasks cancel <lookup>
```

Bricht eine laufende Hintergrundaufgabe ab.

### `audit`

```bash
openclaw tasks audit [--severity <warn|error>] [--code <name>] [--limit <n>] [--json]
```

Zeigt veraltete, verlorene, bei der Zustellung fehlgeschlagene oder anderweitig inkonsistente Aufgaben- und
TaskFlow-Datensätze an. Bis `cleanupAfter` aufbewahrte verlorene Aufgaben sind Warnungen;
abgelaufene oder nicht mit einem Zeitstempel versehene verlorene Aufgaben sind Fehler.

`--code` akzeptiert Aufgabencodes (`stale_queued`, `stale_running`, `lost`,
`delivery_failed`, `missing_cleanup`, `inconsistent_timestamps`) und TaskFlow-Codes
(`restore_failed`, `stale_waiting`, `stale_blocked`,
`cancel_stuck`, `missing_linked_tasks`, `blocked_task_missing`). Unter
[Hintergrundaufgaben](/de/automation/tasks) finden Sie Einzelheiten zum Schweregrad und Auslöser jedes
Codes.

### `maintenance`

```bash
openclaw tasks maintenance [--apply] [--json]
```

Zeigt eine Vorschau der Abstimmung, Kennzeichnung für die Bereinigung,
Bereinigung und Entfernung veralteter Registrierungseinträge für Cron-Ausführungssitzungen von Aufgaben und TaskFlow an oder wendet diese Vorgänge an.

Bei Cron-Aufgaben verwendet die Abstimmung persistierte Ausführungsprotokolle und den Auftragsstatus, bevor
eine alte aktive Aufgabe als `lost` markiert wird, damit abgeschlossene Cron-Ausführungen nicht
fälschlicherweise als Überprüfungsfehler gelten, nur weil der speicherinterne Laufzeitstatus des Gateways nicht mehr vorhanden ist.
Eine Offline-CLI-Überprüfung ist für die prozesslokale Menge aktiver Cron-Aufträge des Gateways
nicht maßgeblich. CLI-Aufgaben mit einer Ausführungs-ID/Quell-ID werden als `lost` markiert, wenn
ihr aktiver Gateway-Ausführungskontext nicht mehr vorhanden ist, selbst wenn noch ein alter Datensatz der untergeordneten Sitzung
vorhanden ist.

Bei der Anwendung entfernt die Wartung außerdem `cron:<jobId>:run:<uuid>`-Sitzungsregistrierungsdatensätze,
die älter als 7 Tage sind, wobei derzeit laufende Cron-Aufträge beibehalten und
Sitzungsdatensätze, die nicht zu Cron gehören, unverändert gelassen werden.

### `flow`

```bash
openclaw tasks flow list [--status <name>] [--json]
openclaw tasks flow show <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

Untersucht dauerhaften TaskFlow-Status im Aufgabenjournal oder bricht ihn ab.
`flow list --status` akzeptiert `queued`, `running`, `waiting`, `blocked`,
`succeeded`, `failed`, `cancelled` oder `lost`.

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Hintergrundaufgaben](/de/automation/tasks)
