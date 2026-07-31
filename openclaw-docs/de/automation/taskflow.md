---
read_when:
    - Sie möchten verstehen, wie Task Flow mit Hintergrundaufgaben zusammenhängt
    - Sie stoßen in Versionshinweisen oder der Dokumentation auf Task Flow oder openclaw tasks flow
    - Sie möchten den dauerhaften Ablaufstatus prüfen oder verwalten
summary: TaskFlow-Orchestrierungsebene oberhalb von Hintergrundaufgaben
title: Aufgabenablauf
x-i18n:
    generated_at: "2026-07-26T18:47:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5ccc6acf58b4b44c2989e3061bff08dabce8ef385706102360c756a1286ddd1b
    source_path: automation/taskflow.md
    workflow: 16
---

Task Flow ist die Orchestrierungsschicht über [Hintergrundaufgaben](/de/automation/tasks). Ein Flow ist ein dauerhafter Datensatz für mehrstufige Arbeit mit eigenem Status, JSON-Zustand, Revisionszähler und verknüpften Aufgabendatensätzen. Flows überstehen Gateway-Neustarts; einzelne Aufgaben bleiben die Einheit für entkoppelte Arbeit.

## Wann Task Flow verwendet werden sollte

| Szenario                                        | Verwendung                                      |
| ----------------------------------------------- | ----------------------------------------------- |
| Einzelner Hintergrundauftrag                    | Einfache Aufgabe                                |
| Durch Plugin-Code gesteuerte mehrstufige Pipeline | Task Flow (verwaltet)                           |
| Entkoppelter ACP- oder Subagent-Start           | Task Flow (gespiegelt, automatisch erstellt)    |
| Einmalige Erinnerung                            | Cron-Auftrag                                    |

## Synchronisierungsmodi

### Verwalteter Modus

Ein verwalteter Flow verfügt über einen Controller: Plugin-Code, der den Flow über die Task-Flow-API der Plugin-Laufzeit mit einem Ziel und einer erforderlichen Controller-ID erstellt und ihn anschließend explizit steuert.

- Jeder Schritt wird als Hintergrundaufgabe ausgeführt, die unter dem Flow erstellt wird; der Eigentümerschlüssel und der Ursprung des Anfordernden werden an untergeordnete Aufgaben weitergegeben.
- Der Controller überführt den Flow zwischen `running`, `waiting` und Endzuständen und speichert einen beliebigen JSON-Schrittzustand im Flow-Datensatz.
- Bei jeder Änderung wird die erwartete Revision des Flows übergeben. Ein Schreibvorgang mit veralteter Revision wird als Revisionskonflikt abgelehnt, statt einen neueren Zustand zu überschreiben.
- Sobald eine Abbrechung angefordert wurde, werden neue untergeordnete Aufgaben abgelehnt, und der Flow wird als `cancelled` abgeschlossen, wenn keine untergeordnete Aufgabe mehr aktiv ist.

Beispiel: ein wöchentlicher Berichts-Flow, der (1) Daten erfasst, (2) den Bericht erstellt und (3) ihn zustellt, mit einer Hintergrundaufgabe pro Schritt:

```
Flow: weekly-report
  Schritt 1: gather-data     → Aufgabe erstellt → erfolgreich
  Schritt 2: generate-report → Aufgabe erstellt → erfolgreich
  Schritt 3: deliver         → Aufgabe erstellt → wird ausgeführt
```

### Gespiegelter Modus

OpenClaw erstellt automatisch einen gespiegelten Flow mit einer Aufgabe, wenn ein entkoppelter ACP- oder Subagent-Lauf beginnt (sitzungsbezogene Aufgaben mit zustellbarem Abschluss). Der Flow-Datensatz spiegelt seine einzige zugrunde liegende Aufgabe – Status, Ziel und Zeitangaben –, sodass entkoppelte Starts ohne Controller eine stabile Flow-Kennung für Status- und Wiederholungsoberflächen erhalten. Gespiegelte Flows zeigen in der CLI den Synchronisierungsmodus `task_mirrored` an.

## Flow-Status

| Status      | Bedeutung                                                                  |
| ----------- | -------------------------------------------------------------------------- |
| `queued`    | Erstellt, Fortschritt hat noch nicht begonnen                              |
| `running`   | Flow macht aktiv Fortschritte                                               |
| `waiting`   | Verwalteter Flow wartet gemäß Wartemetadaten (Timer, externes Ereignis)     |
| `blocked`   | Ein Schritt wurde ohne verwertbares Ergebnis beendet; `blockedTaskId`/Zusammenfassung geben den betreffenden Schritt an |
| `succeeded` | Erfolgreich abgeschlossen                                                   |
| `failed`    | Mit einem Fehler abgeschlossen                                              |
| `cancelled` | Abbrechung angefordert und alle untergeordneten Aufgaben beendet            |
| `lost`      | Der Flow hat seinen maßgeblichen zugrunde liegenden Zustand verloren        |

## Dauerhafter Zustand und Revisionsverfolgung

Flow-Datensätze werden gemeinsam mit Aufgabendatensätzen in der gemeinsam genutzten SQLite-Zustandsdatenbank (`~/.openclaw/state/openclaw.sqlite`, Tabelle `flow_runs`) gespeichert, sodass der Fortschritt Gateway-Neustarts übersteht. Jeder Schreibvorgang erhöht `revision` des Flows; gleichzeitige Schreibende, die eine veraltete erwartete Revision übergeben, erhalten einen Konflikt und müssen die Daten erneut lesen. Das WAL-Wachstum wird durch automatische SQLite-Checkpoints sowie regelmäßige passive Checkpoints begrenzt; beim Herunterfahren werden Truncate-Checkpoints ausgeführt. Die veraltete `flows/registry.sqlite`-Begleitdatei aus älteren Installationen wird durch `openclaw doctor` importiert.

## Abbruchverhalten

`openclaw tasks flow cancel` setzt eine dauerhafte Abbruchabsicht für den Flow, bricht seine aktiven untergeordneten Aufgaben ab und lehnt neue verwaltete untergeordnete Aufgaben ab. Sobald keine untergeordnete Aufgabe mehr aktiv ist, wird der Flow als `cancelled` abgeschlossen – entweder sofort oder durch den Wartungsdurchlauf, falls die Beendigung der untergeordneten Aufgaben länger dauert. Die Absicht wird gespeichert, sodass ein abgebrochener Flow auch dann abgebrochen bleibt, wenn das Gateway neu startet, bevor alle untergeordneten Aufgaben beendet wurden.

## CLI-Befehle

```bash
# Aktive und kürzlich ausgeführte Flows auflisten
openclaw tasks flow list [--status <status>] [--json]

# Details zu einem bestimmten Flow anzeigen
openclaw tasks flow show <lookup> [--json]

# Einen laufenden Flow und seine aktiven Aufgaben abbrechen
openclaw tasks flow cancel <lookup>
```

| Befehl                            | Beschreibung                                                               |
| --------------------------------- | -------------------------------------------------------------------------- |
| `openclaw tasks flow list`        | Verfolgte Flows mit Synchronisierungsmodus, Status, Revision, Controller und Aufgabenanzahl |
| `openclaw tasks flow show <id>`   | Einen Flow anhand der Flow-ID oder des Eigentümerschlüssels einschließlich verknüpfter Aufgaben prüfen |
| `openclaw tasks flow cancel <id>` | Einen laufenden Flow und seine aktiven Aufgaben abbrechen                   |

Flows werden außerdem von `openclaw tasks audit` (Feststellungen zu veralteten oder beschädigten Flows) und `openclaw tasks maintenance` (schließt festhängende Abbrüche ab und entfernt Endzustands-Flows nach 7 Tagen) berücksichtigt.

## Muster für zuverlässige geplante Workflows

Behandeln Sie bei wiederkehrenden Workflows wie Marktanalyse-Briefings die Planung, Orchestrierung und Zuverlässigkeitsprüfungen als getrennte Schichten:

1. Verwenden Sie [Geplante Aufgaben](/de/automation/cron-jobs) für die zeitliche Planung.
2. Verwenden Sie eine persistente Cron-Sitzung, wenn der Workflow auf vorherigem Kontext aufbauen soll.
3. Verwenden Sie [Lobster](/de/tools/lobster) für deterministische Schritte, Genehmigungsschranken und Fortsetzungstoken.
4. Verwenden Sie Task Flow, um den mehrstufigen Lauf über untergeordnete Aufgaben, Wartezeiten, Wiederholungen und Gateway-Neustarts hinweg zu verfolgen.

Beispiel für eine Cron-Struktur:

```bash
openclaw cron add \
  --name "Marktanalyse-Briefing" \
  --cron "0 7 * * 1-5" \
  --tz "America/New_York" \
  --session session:market-intel \
  --message "Den Lobster-Workflow für Marktanalysen ausführen. Vor der Zusammenfassung die Aktualität der Quellen überprüfen." \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

Verwenden Sie `--session session:<id>` anstelle von `isolated`, wenn der wiederkehrende Workflow einen gezielt aufgebauten Verlauf, Zusammenfassungen vorheriger Läufe oder dauerhaften Kontext benötigt. Verwenden Sie `isolated`, wenn jeder Lauf ohne vorherigen Kontext beginnen soll und der gesamte erforderliche Zustand explizit im Workflow angegeben ist.

Platzieren Sie Zuverlässigkeitsprüfungen innerhalb des Workflows vor dem LLM-Zusammenfassungsschritt:

```yaml
name: market-intel-brief
steps:
  - id: preflight
    command: market-intel check --json
  - id: collect
    command: market-intel collect --json
    stdin: $preflight.json
  - id: summarize
    command: market-intel summarize --json
    stdin: $collect.json
  - id: approve
    command: market-intel deliver --preview
    stdin: $summarize.json
    approval: required
  - id: deliver
    command: market-intel deliver --execute
    stdin: $summarize.json
    condition: $approve.approved
```

Empfohlene Vorabprüfungen:

- Browser-Verfügbarkeit und Profilauswahl, beispielsweise `openclaw` für einen verwalteten Zustand oder `user`, wenn eine angemeldete Chrome-Sitzung erforderlich ist. Siehe [Browser](/de/tools/browser).
- API-Anmeldedaten und Kontingent für jede Quelle.
- Netzwerkerreichbarkeit der erforderlichen Endpunkte.
- Für den Agenten aktivierte erforderliche Werkzeuge, etwa `lobster`, `browser` und `llm-task`.
- Für Cron konfiguriertes Fehlerziel, damit Fehler bei Vorabprüfungen sichtbar sind. Siehe [Geplante Aufgaben](/de/automation/cron-jobs#delivery-and-output).

Empfohlene Felder zur Datenherkunft für jedes erfasste Element:

```json
{
  "sourceUrl": "https://example.com/report",
  "retrievedAt": "2026-04-24T12:00:00Z",
  "asOf": "2026-04-24",
  "title": "Beispielbericht",
  "content": "..."
}
```

Lassen Sie den Workflow veraltete Elemente vor der Zusammenfassung ablehnen oder als veraltet kennzeichnen. Der LLM-Schritt sollte ausschließlich strukturiertes JSON erhalten und angewiesen werden, `sourceUrl`, `retrievedAt` und `asOf` in seiner Ausgabe beizubehalten. Verwenden Sie [LLM-Aufgabe](/de/tools/llm-task), wenn Sie innerhalb des Workflows einen schemavalidierten Modellschritt benötigen.

Verpacken Sie bei wiederverwendbaren Team- oder Community-Workflows die CLI, die `.lobster`-Dateien und sämtliche Einrichtungshinweise als Skill oder Plugin und veröffentlichen Sie das Paket über [ClawHub](/de/clawhub). Behalten Sie Workflow-spezifische Schutzmechanismen in diesem Paket, sofern der Plugin-API keine benötigte generische Funktion fehlt.

## Beziehung zwischen Flows und Aufgaben

Flows koordinieren Aufgaben, sie ersetzen sie nicht. Ein einzelner Flow kann während seiner Lebensdauer mehrere Hintergrundaufgaben steuern. Verwenden Sie `openclaw tasks`, um einzelne Aufgabendatensätze zu prüfen, und `openclaw tasks flow`, um den orchestrierenden Flow zu prüfen.

## Verwandte Themen

- [Hintergrundaufgaben](/de/automation/tasks) – das Verzeichnis entkoppelter Arbeiten, die von Flows koordiniert werden
- [CLI: Aufgaben](/de/cli/tasks) – CLI-Befehlsreferenz für `openclaw tasks flow`
- [Automatisierungsübersicht](/de/automation) – alle Automatisierungsmechanismen auf einen Blick
- [Cron-Aufträge](/de/automation/cron-jobs) – geplante Aufträge, die Flows speisen können
