---
doc-schema-version: 1
read_when:
    - Sie möchten, dass OpenClaw während einer langen Sitzung ein Ziel sichtbar hält
    - Sie müssen ein Sitzungsziel pausieren, fortsetzen, blockieren, abschließen oder löschen.
    - Sie möchten die Tools get_goal, create_goal und update_goal verstehen
    - Sie möchten sehen, wie Ziele in der TUI angezeigt werden
summary: 'Sitzungsziele: dauerhafte sitzungsspezifische Zielsetzungen, /goal-Steuerung, Modell-Zielwerkzeuge, Token-Budgets und TUI-Status'
title: Ziel
x-i18n:
    generated_at: "2026-07-26T18:49:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8bfe25eb9901394b32b61729fbcb6a7bd711ed859d284fa39b637000ed7f0a18
    source_path: tools/goal.md
    workflow: 16
---

# Ziel

Ein **Ziel** ist eine dauerhafte Zielsetzung, die der aktuellen OpenClaw-Sitzung zugeordnet ist.
Es gibt dem Agenten und dem Operator ein gemeinsames Ziel für länger laufende Arbeiten,
ohne dieses Ziel in eine Hintergrundaufgabe, Erinnerung, einen Cron-Job oder
Dauerauftrag umzuwandeln.

Ziele sind Sitzungsstatus: Sie werden mit dem Sitzungsschlüssel übertragen, bleiben über
Prozessneustarts hinweg erhalten und erscheinen in `/goal`, den modellseitigen Ziel-Tools und in der
Fußzeile der TUI.

Abgeschlossene entkoppelte Befehle kehren zum ursprünglichen benutzerseitigen Thread zurück, sodass
der nächste Turn weiterhin dasselbe Ziel sieht, selbst wenn für die Befehlsausführung
eine separate Sitzung mit eigenen Sandbox-Richtlinien verwendet wurde.

## Schnellstart

```text
/goal start get CI green for PR 87469 and push the fix
/goal
/goal edit get CI green for PR 87469, push the fix, and update docs
/goal pause waiting for CI
/goal resume
/goal complete pushed and verified
/goal clear
```

`start` ist optional: `/goal get CI green for PR 87469` erstellt ebenfalls ein Ziel,
da jeder Text nach `/goal`, der kein bekanntes Aktionswort ist, als
neue Zielsetzung behandelt wird.

## Wofür Ziele vorgesehen sind

Verwenden Sie ein Ziel, wenn eine Sitzung ein konkretes Ergebnis hat, das über
viele Turns hinweg sichtbar bleiben soll:

- Abschluss eines PRs: korrigieren, verifizieren, automatisch prüfen, pushen und den PR öffnen oder aktualisieren.
- Ein Debugging-Durchlauf: den Fehler reproduzieren, die zuständige Oberfläche identifizieren, korrigieren und
  die Korrektur nachweisen.
- Eine Dokumentationsüberarbeitung: die relevanten Dokumente lesen, die neue Seite verfassen, Querverweise hinzufügen und
  den Dokumentations-Build verifizieren.
- Eine Wartungsaufgabe: den aktuellen Zustand prüfen, begrenzte Änderungen vornehmen, die
  richtigen Prüfungen ausführen und die Änderungen melden.

Ein Ziel ist keine Aufgabenwarteschlange. Verwenden Sie [Task Flow](/de/automation/taskflow),
[Aufgaben](/de/automation/tasks), [Cron-Jobs](/de/automation/cron-jobs) oder
[Daueraufträge](/de/automation/standing-orders), wenn Arbeiten entkoppelt ausgeführt,
nach einem Zeitplan wiederholt, in verwaltete Teilaufgaben aufgegliedert oder als Richtlinie beibehalten werden sollen.

## Befehlsreferenz

`/goal` ohne Argumente gibt die aktuelle Zielzusammenfassung aus:

```text
Ziel
Status: aktiv
Zielsetzung: CI für PR 87469 erfolgreich abschließen und die Korrektur pushen
Verwendete Token: 12k
Token-Budget: 12k/50k

Befehle: /goal edit <objective>, /goal pause, /goal complete, /goal clear
```

| Befehl                                             | Wirkung                                                                   |
| --------------------------------------------------- | ------------------------------------------------------------------------ |
| `/goal` oder `/goal status`                           | Zeigt das aktuelle Ziel an.                                                   |
| `/goal start <objective>`                           | Erstellt ein neues Ziel für die aktuelle Sitzung.                               |
| `/goal set <objective>`, `/goal create <objective>` | Aliasse für `start`.                                                     |
| `/goal <objective>`                                 | Erstellt ebenfalls ein neues Ziel (jeder Text, der kein erkanntes Aktionswort ist). |
| `/goal edit <objective>`                            | Formuliert die aktuelle Zielsetzung neu; Status und Token-Erfassung bleiben unverändert.      |
| `/goal pause [note]`                                | Pausiert ein aktives Ziel.                                                    |
| `/goal resume [note]`                               | Setzt ein pausiertes, blockiertes, nutzungsbegrenztes oder budgetbegrenztes Ziel fort.         |
| `/goal complete [note]`                             | Markiert das Ziel als erreicht.                                                  |
| `/goal done [note]`                                 | Alias für `complete`.                                                    |
| `/goal block [note]`                                | Markiert das Ziel als blockiert.                                                   |
| `/goal blocked [note]`                              | Alias für `block`.                                                       |
| `/goal clear`                                       | Entfernt das Ziel aus der Sitzung.                                        |

In einer Sitzung kann jeweils nur ein Ziel vorhanden sein. Das Starten eines zweiten Ziels schlägt
mit `Goal error: goal already exists` fehl, bis das aktuelle Ziel gelöscht wurde.

`/goal start` akzeptiert kein Token-Budget-Flag; ein Budget kann nur
über das modellseitige Tool `create_goal` festgelegt werden.

## Status

- `active`: Die Sitzung verfolgt das Ziel.
- `paused`: Der Operator hat das Ziel pausiert; `/goal resume` aktiviert es
  wieder.
- `blocked`: Der Agent oder Operator hat einen tatsächlichen Blocker gemeldet; `/goal resume`
  aktiviert es wieder, sobald neue Informationen oder ein neuer Zustand verfügbar sind.
- `budget_limited`: Das konfigurierte Token-Budget wurde erreicht; `/goal resume`
  setzt die Verfolgung derselben Zielsetzung mit einem neuen Budgetfenster fort.
- `usage_limited`: Für einen zukünftigen Stoppzustand aufgrund eines Nutzungslimits reserviert; `/goal
resume` setzt die Verfolgung auf dieselbe Weise fort.
- `complete`: Das Ziel wurde erreicht. Abgeschlossene Ziele sind endgültig; verwenden Sie `/goal
clear`, bevor Sie ein weiteres Ziel starten.

`/new` und `/reset` löschen das aktuelle Sitzungsziel, da sie absichtlich
einen neuen Sitzungskontext beginnen.

## Token-Budgets

Ziele können ein optionales positives Token-Budget haben, das über den Parameter
`token_budget` des Tools `create_goal` festgelegt wird. Das Budget wird ab dem
aktuellen Token-Zählerstand der Sitzung zum Zeitpunkt der Zielerstellung gemessen. Wenn die Sitzung beim Start des Ziels nur über einen
veralteten oder unbekannten Token-Snapshot verfügt, wartet OpenClaw auf den
nächsten aktuellen Snapshot und verwendet diesen als Ausgangswert, sodass Token, die vor der
Existenz des Ziels verbraucht wurden, diesem nicht angerechnet werden.

Wenn die Nutzung das Budget erreicht, wechselt das Ziel zu `budget_limited`. Dadurch wird
das Ziel weder gelöscht noch die Zielsetzung entfernt; der Operator und der
Agent werden darüber informiert, dass das Ziel nicht mehr aktiv verfolgt wird, bis es fortgesetzt oder
gelöscht wird. Beim Fortsetzen beginnt ein neues Budgetfenster beim aktuellen
Token-Zählerstand.

Token-Budgets dienen als Leitplanke für Sitzungsziele, nicht als Abrechnungslimit. Provider-
Kontingente, Kostenberichte und das Verhalten des Kontextfensters verwenden weiterhin die normalen
Nutzungs- und Modellsteuerungen von OpenClaw.

## Modell-Tools

OpenClaw stellt Agent-Harnesses drei Ziel-Tools bereit:

| Tool          | Zweck                                                                                                                  |
| ------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `get_goal`    | Liest das aktuelle Sitzungsziel: Status, Zielsetzung, Token-Nutzung und Token-Budget.                                         |
| `create_goal` | Erstellt nur dann ein Ziel, wenn der Benutzer oder die Systemanweisungen dies ausdrücklich verlangen. Schlägt fehl, wenn die Sitzung bereits ein Ziel hat. |
| `update_goal` | Markiert das Ziel als `complete` oder `blocked`.                                                                                   |

Das Modell kann ein Ziel nicht unbemerkt pausieren, fortsetzen, löschen oder ersetzen. Dies bleiben
Operator- und Sitzungssteuerungen über `/goal` und Zurücksetzungsbefehle, sodass der Agent
das Erreichen des Ziels oder einen tatsächlichen Blocker melden kann, ohne das
Ziel unbemerkt zu verschieben.

`update_goal` sollte ein Ziel nur dann als `complete` markieren, wenn die Zielsetzung
tatsächlich erreicht wurde. Ein Ziel sollte erst dann als `blocked` markiert werden, wenn dieselbe
blockierende Bedingung in mindestens drei aufeinanderfolgenden Ziel-Turns erneut auftritt, nicht bei
gewöhnlichen Schwierigkeiten oder fehlendem Feinschliff.

## Zielkontext bei jedem Turn

Jeder Benutzer-/Chat-Turn mit einem aktiven Ziel enthält diese Kontextzeile mit der Benutzerrolle:

```text
Aktives Ziel: <objective> — bringen Sie es voran oder aktualisieren Sie seinen Status (get_goal/update_goal).
```

OpenClaw hält die Zeile kompakt, indem lange Zielsetzungen gekürzt werden. Pausierte,
blockierte, budgetbegrenzte, nutzungsbegrenzte und abgeschlossene Ziele werden nicht eingefügt,
sodass ein Stopp durch den Operator wirksam bleibt, bis das Ziel fortgesetzt wird.

## Control UI

Die webbasierte Control UI zeigt das Ziel als kompakte Plakette über dem Chat-Eingabefeld:
ein Statussymbol, die Statusbezeichnung (zum Beispiel `Pursuing goal`), die gekürzte
Zielsetzung und einen live aktualisierten Zeitgeber für die verstrichene Zeit.

Die Plakette enthält Inline-Steuerelemente:

- **Stift** füllt das Eingabefeld mit `/goal edit <objective>`, sodass die
  Zielsetzung neu formuliert und abgesendet werden kann.
- **Pausieren/Fortsetzen** wechselt abhängig vom aktuellen Status zwischen `/goal pause` und `/goal resume`.
- **Papierkorb** sendet `/goal clear`.
- **Chevron** erweitert die Plakette, um die vollständige Zielsetzung, den neuesten Statushinweis,
  die Token-Nutzung und die verstrichene Zeit anzuzeigen.

Die Aktionsschaltflächen sind ausgeblendet, solange das Eingabefeld nicht senden kann (zum Beispiel
wenn die Gateway-Verbindung unterbrochen ist); das Chevron zum Erweitern funktioniert weiterhin.

## TUI

Die Fußzeile der TUI hält das Ziel der aktiven Sitzung neben den Feldern für Agent,
Sitzung und Modell sowie vor den Token-/Modusindikatoren sichtbar.

Beispiele für die Fußzeile:

- `Pursuing goal (12k/50k)` für ein aktives Ziel mit Token-Budget.
- `Goal paused (/goal resume)` für ein pausiertes Ziel.
- `Goal blocked (/goal resume)` für ein blockiertes Ziel.
- `Goal hit usage limits (/goal resume)` für ein nutzungsbegrenztes Ziel.
- `Goal unmet (50k/50k)` für ein budgetbegrenztes Ziel.
- `Goal achieved (42k)` für ein abgeschlossenes Ziel.

Die Fußzeile ist absichtlich kompakt. Verwenden Sie `/goal` für die vollständige Zielsetzung,
den Hinweis, das Token-Budget und die verfügbaren Befehle.

## Kanalverhalten

`/goal` funktioniert in befehlsfähigen OpenClaw-Sitzungen, einschließlich der TUI und
Chat-Oberflächen, die Textbefehle erlauben. Der Zielstatus ist dem
Sitzungsschlüssel zugeordnet, nicht dem Transport, sodass zwei Oberflächen mit demselben Sitzungsschlüssel
dasselbe Ziel sehen.

Der Zielstatus ist keine Zustellanweisung: Er erzwingt keine Antworten über einen
Kanal, ändert nicht das Warteschlangenverhalten, genehmigt keine Tools und plant keine Arbeiten.

## Fehlerbehebung

| Meldung                                | Bedeutung                                                                                                                                      |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `Goal error: goal already exists`      | Die Sitzung hat bereits ein Ziel. Verwenden Sie `/goal`, um es zu prüfen, `/goal complete`, wenn es abgeschlossen ist, oder `/goal clear`, bevor Sie eine andere Zielsetzung starten. |
| `Goal error: goal not found`           | Die Sitzung hat noch kein Ziel. Starten Sie eines mit `/goal start <objective>`.                                                                       |
| `Goal error: goal is already complete` | Das Ziel ist endgültig. Löschen Sie es, bevor Sie eine andere Zielsetzung starten oder fortsetzen.                                                                |

Wenn die Token-Nutzung `0` anzeigt oder veraltet erscheint, verfügt die aktive Sitzung möglicherweise noch nicht über einen
aktuellen Token-Snapshot. Die Nutzung wird aktualisiert, während OpenClaw die Sitzungsnutzung
und aus dem Transkript abgeleitete Gesamtwerte erfasst.

## Verwandte Themen

- [Slash-Befehle](/de/tools/slash-commands)
- [TUI](/de/web/tui)
- [Sitzungs-Tool](/de/concepts/session-tool)
- [Compaction](/de/concepts/compaction)
- [Task Flow](/de/automation/taskflow)
- [Daueraufträge](/de/automation/standing-orders)
