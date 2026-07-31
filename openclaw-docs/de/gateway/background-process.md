---
read_when:
    - Hinzufügen oder Ändern des Verhaltens bei der Hintergrundausführung
    - Debugging lang laufender Exec-Aufgaben
summary: Hintergrundausführung und Prozessverwaltung
title: Hintergrundausführung und Prozesstool
x-i18n:
    generated_at: "2026-07-26T18:57:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 37cb65ddf67227e32be972e77d16b9835d592120ecd12e041d05c48536fd2204
    source_path: gateway/background-process.md
    workflow: 16
---

OpenClaw führt Shell-Befehle über das Tool `exec` aus und hält lang laufende Aufgaben im Speicher. Das Tool `process` verwaltet diese Hintergrundsitzungen.

## exec-Tool

Parameter:

| Parameter    | Beschreibung                                                                                                                                                |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`    | Erforderlich. Auszuführender Shell-Befehl.                                                                                                                            |
| `workdir`    | Arbeitsverzeichnis; weglassen, um das standardmäßige aktuelle Arbeitsverzeichnis zu verwenden.                                                                                                            |
| `env`        | Zusätzliche Umgebungsvariablen für den Befehl.                                                                                                               |
| `yieldMs`    | Wartezeit in Millisekunden vor der Ausführung im Hintergrund (Standardwert: 10000).                                                                                                 |
| `background` | Sofort im Hintergrund ausführen.                                                                                                                             |
| `timeout`    | Zeitüberschreitung in Sekunden (Standardwert: `tools.exec.timeoutSeconds`); beendet den Prozess nach Ablauf. Legen Sie `timeout: 0` fest, um die Zeitüberschreitung des exec-Prozesses für diesen Aufruf zu deaktivieren. |
| `pty`        | Wenn verfügbar, in einem Pseudoterminal ausführen (TTY-erfordernde CLIs, Coding-Agenten).                                                                                |
| `elevated`   | Außerhalb der Sandbox ausführen, wenn der erweiterte Modus aktiviert/zulässig ist (standardmäßig `gateway` oder `node`, wenn das exec-Ziel `node` ist).                              |
| `host`       | exec-Ziel: `auto`, `sandbox`, `gateway` oder `node`.                                                                                                      |
| `node`       | Node-ID/-Name, verwendet mit `host: "node"`.                                                                                                                    |

Verhalten:

- Ausführungen im Vordergrund geben die Ausgabe direkt zurück.
- Bei Ausführung im Hintergrund (explizit oder durch Zeitüberschreitung von `yieldMs`) gibt das Tool `status: "running"` + `sessionId` und einen kurzen Ausschnitt vom Ende der Ausgabe zurück.
- Ausführungen im Hintergrund und mit `yieldMs` übernehmen `tools.exec.timeoutSeconds`, sofern der Aufruf nicht ausdrücklich `timeout` übergibt.
- Die Ausgabe bleibt im Speicher, bis die Sitzung abgefragt oder gelöscht wird.
- Wenn das Tool `process` nicht zulässig ist, werden `exec`-Ausführungen synchron ausgeführt und `yieldMs`/`background` ignoriert.
- Gestartete exec-Befehle erhalten `OPENCLAW_SHELL=exec` für kontextabhängige Shell-/Profilregeln.
- Für lang laufende Arbeiten, die jetzt beginnen: Starten Sie sie einmal und verlassen Sie sich, sofern aktiviert, auf die automatische Benachrichtigung bei Abschluss, sobald der Befehl eine Ausgabe erzeugt oder fehlschlägt.
- Wenn die automatische Benachrichtigung bei Abschluss nicht verfügbar ist oder Sie den stillen Erfolg eines Befehls bestätigen müssen, der ohne Ausgabe ordnungsgemäß beendet wird, fragen Sie mit `process` ab.
- Simulieren Sie Erinnerungen oder verzögerte Folgeaktionen nicht mit `sleep`-Schleifen oder wiederholten Abfragen – verwenden Sie Cron für zukünftige Arbeiten.

### Umgebungsüberschreibungen

| Variable                                 | Wirkung                                                                                                           |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_BASH_YIELD_MS`                 | Standardwartezeit vor der Ausführung im Hintergrund (ms). Standardwert: 10000, begrenzt auf 10–120000.                                       |
| `OPENCLAW_BASH_MAX_OUTPUT_CHARS`         | Obergrenze für die Ausgabe im Speicher (Zeichen).                                                                                    |
| `OPENCLAW_BASH_PENDING_MAX_OUTPUT_CHARS` | Obergrenze für ausstehende stdout-/stderr-Daten pro Stream (Zeichen).                                                                    |
| `OPENCLAW_BASH_JOB_TTL_MS`               | TTL für abgeschlossene Sitzungen (ms), begrenzt auf 1m–3h.                                                                |
| `OPENCLAW_PROCESS_INPUT_WAIT_IDLE_MS`    | Schwellenwert für ausbleibende Ausgabe, nach dem beschreibbare Hintergrundsitzungen als wahrscheinlich auf Eingabe wartend markiert werden. Standardwert: 15000. |

### Konfiguration (gegenüber Umgebungsüberschreibungen bevorzugt)

| Schlüssel                                   | Standardwert | Wirkung                                                                          |
| ------------------------------------- | ------- | ------------------------------------------------------------------------------- |
| `tools.exec.backgroundMs`             | 10000   | Entspricht `OPENCLAW_BASH_YIELD_MS`.                                               |
| `tools.exec.timeoutSeconds`           | 1800    | Standardmäßige Zeitüberschreitung pro Aufruf.                                                       |
| `tools.exec.cleanupMs`                | 1800000 | Entspricht `OPENCLAW_BASH_JOB_TTL_MS`.                                             |
| `tools.exec.notifyOnExit`             | true    | Ein Systemereignis in die Warteschlange einreihen und einen Heartbeat anfordern, wenn eine exec-Ausführung im Hintergrund beendet wird.      |
| `tools.exec.notifyOnExitEmptySuccess` | false   | Abschlussereignisse auch für erfolgreiche Hintergrundausführungen ohne Ausgabe in die Warteschlange einreihen. |

## Überbrückung von Kindprozessen

Wenn lang laufende Kindprozesse außerhalb der exec-/process-Tools gestartet werden (CLI-Neustarts, Gateway-Hilfsprozesse), binden Sie den Hilfsmechanismus zur Überbrückung von Kindprozessen ein, damit Beendigungssignale weitergeleitet und Listener beim Beenden/bei Fehlern entfernt werden. Dies verhindert verwaiste Prozesse unter systemd und gewährleistet ein plattformübergreifend konsistentes Herunterfahren.

## process-Tool

Aktionen:

| Aktion      | Wirkung                                                                        |
| ----------- | ----------------------------------------------------------------------------- |
| `list`      | Laufende und abgeschlossene Sitzungen.                                                  |
| `poll`      | Neue Ausgabe einer Sitzung abrufen (meldet auch den Beendigungsstatus).                    |
| `log`       | Zusammengefasste Ausgabe und Hinweise zur Wiederaufnahme der Eingabe lesen. Unterstützt `offset` + `limit`. |
| `write`     | stdin senden (`data`, optional `eof`).                                          |
| `send-keys` | Explizite Tastentokens oder Bytes an eine PTY-gestützte Sitzung senden.                    |
| `submit`    | Eingabetaste/Wagenrücklauf an eine PTY-gestützte Sitzung senden.                           |
| `paste`     | Literaltext senden, optional im Modus für geklammertes Einfügen.                |
| `kill`      | Eine Hintergrundsitzung beenden.                                               |
| `clear`     | Eine abgeschlossene Sitzung aus dem Speicher entfernen.                                        |
| `remove`    | Bei laufender Sitzung beenden, andernfalls eine abgeschlossene Sitzung löschen.                                 |

Hinweise:

- Nur Hintergrundsitzungen werden aufgelistet/gespeichert – ausschließlich im Speicher, nicht auf dem Datenträger. Sitzungen gehen bei einem Prozessneustart verloren.
- Eine aktive Hintergrundsitzung verhindert die kooperative Suspendierung des Hosts und einen sicheren Neustart des Gateways, bis der Prozesseigentümer das tatsächliche Ende bestätigt.
- `process remove` kann eine laufende Sitzung unmittelbar nach dem Anfordern der Beendigung ausblenden; Suspendierung und Neustart bleiben bis zur Bestätigung des Endes blockiert.
- Sitzungsprotokolle werden nur im Chatverlauf gespeichert, wenn Sie `process poll`/`log` ausführen und das Tool-Ergebnis aufgezeichnet wird.
- `process` ist auf den jeweiligen Agenten beschränkt; es sieht nur Sitzungen, die von diesem Agenten gestartet wurden.
- Verwenden Sie `poll`/`log` für Status, Protokolle oder die Bestätigung des Abschlusses, wenn die automatische Benachrichtigung bei Abschluss nicht verfügbar ist.
- Verwenden Sie `log`, bevor Sie eine interaktive CLI wiederaufnehmen, damit das aktuelle Transkript, der stdin-Status und der Hinweis zum Warten auf Eingabe gemeinsam sichtbar sind.
- Verwenden Sie `write`/`send-keys`/`submit`/`paste`/`kill`, wenn Eingaben oder ein Eingriff erforderlich sind.
- `process list` enthält für eine schnelle Übersicht eine abgeleitete Angabe `name` (Befehlsverb + Ziel).
- `process list`, `poll` und `log` melden `waitingForInput` nur, wenn die Sitzung weiterhin über beschreibbares stdin verfügt und länger als der Schwellenwert für das Warten auf Eingabe inaktiv war (Standardwert: 15000 ms, `OPENCLAW_PROCESS_INPUT_WAIT_IDLE_MS`).
- `process log` verwendet zeilenbasierte Angaben für `offset`/`limit`. Wenn beide weggelassen werden, werden die letzten 200 Zeilen mit einem Hinweis zur Seitennavigation zurückgegeben. Wenn `offset` festgelegt ist und `limit` nicht, wird der Bereich von `offset` bis zum Ende zurückgegeben (nicht auf 200 begrenzt).
- `poll`s `timeout` wartet vor der Rückgabe bis zu der angegebenen Anzahl von Millisekunden; Werte über 30000 werden auf 30000 begrenzt.
- Abfragen dienen dem bedarfsgesteuerten Statusabruf, nicht der Planung von Warteschleifen. Wenn die Arbeit später stattfinden soll, verwenden Sie Cron.

## Beispiele

Eine lang laufende Aufgabe ausführen und später abfragen:

```json
{ "tool": "exec", "command": "sleep 5 && echo done", "yieldMs": 1000 }
```

```json
{ "tool": "process", "action": "poll", "sessionId": "<id>" }
```

Eine interaktive Sitzung vor dem Senden einer Eingabe prüfen:

```json
{ "tool": "process", "action": "log", "sessionId": "<id>" }
```

Sofort im Hintergrund starten:

```json
{ "tool": "exec", "command": "npm run build", "background": true }
```

stdin senden:

```json
{ "tool": "process", "action": "write", "sessionId": "<id>", "data": "y\n" }
```

PTY-Tasten senden:

```json
{ "tool": "process", "action": "send-keys", "sessionId": "<id>", "keys": ["C-c"] }
```

Aktuelle Zeile absenden:

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Literaltext einfügen:

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## Verwandte Themen

- [Exec-Tool](/de/tools/exec)
- [Exec-Genehmigungen](/de/tools/exec-approvals)
