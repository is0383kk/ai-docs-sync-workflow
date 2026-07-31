---
read_when:
    - Planen von Hintergrundaufgaben oder Aufweckvorgängen
    - Externe Auslöser (Webhooks, Gmail) mit OpenClaw verbinden
    - Entscheidung zwischen Heartbeat und Cron für geplante Aufgaben
sidebarTitle: Scheduled tasks
summary: Geplante Aufträge, Webhooks und Gmail-PubSub-Trigger für den Gateway-Scheduler
title: Geplante Aufgaben
x-i18n:
    generated_at: "2026-07-26T18:47:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dd889cf8e45196eda3ec7c2af930abcb2cc2bae8bad2dbdcaf3cd521a9e884b2
    source_path: automation/cron-jobs.md
    workflow: 16
---

Cron ist der integrierte Scheduler des Gateways. Er speichert Jobs dauerhaft, weckt den Agenten zum richtigen Zeitpunkt und kann Ausgaben an einen Chatkanal, einen Webhook oder gar nicht zustellen.

## Schnellstart

<Steps>
  <Step title="Einmalige Erinnerung hinzufügen">
    ```bash
    openclaw cron create "2027-02-01T16:00:00Z" \
      --name "Erinnerung" \
      --session main \
      --system-event "Erinnerung: Entwurf der Cron-Dokumentation prüfen" \
      --wake now \
      --delete-after-run
    ```
  </Step>
  <Step title="Ihre Jobs prüfen">
    ```bash
    openclaw cron list
    openclaw cron get <job-id>
    openclaw cron show <job-id>
    ```
  </Step>
  <Step title="Ausführungsverlauf anzeigen">
    ```bash
    openclaw cron runs --id <job-id>
    ```
  </Step>
</Steps>

## Funktionsweise von Cron

- Cron wird **innerhalb des Gateway-Prozesses** ausgeführt, nicht innerhalb des Modells. Das Gateway muss laufen, damit Zeitpläne ausgelöst werden.
- Jobdefinitionen, Laufzeitstatus und Ausführungsverlauf werden dauerhaft in der gemeinsam genutzten SQLite-Statusdatenbank von OpenClaw gespeichert, sodass Zeitpläne bei Neustarts nicht verloren gehen.
- Jede Cron-Ausführung erstellt einen Datensatz für eine [Hintergrundaufgabe](/de/automation/tasks).
- Einmalige Jobs (`--at`) werden nach erfolgreicher Ausführung standardmäßig automatisch gelöscht; übergeben Sie `--keep-after-run`, um sie beizubehalten.
- Wandzeitbudget pro Ausführung: `--timeout-seconds`, sofern festgelegt. Andernfalls werden isolierte/abgekoppelte Agent-Turn-Jobs durch den Cron-eigenen 60-Minuten-Watchdog begrenzt, bevor das zugrunde liegende Agent-Turn-Zeitlimit (`agents.defaults.timeoutSeconds`, standardmäßig 48 Stunden) überhaupt greifen würde; Befehlsjobs haben standardmäßig ein Zeitlimit von 10 Minuten und Skript-Payloads von 5 Minuten.
- Beim Start des Gateways werden überfällige isolierte Agent-Turn-Jobs neu geplant, statt sofort erneut ausgeführt zu werden. Dadurch bleibt die Bootstrap-Arbeit für Modell und Tools außerhalb des Zeitfensters für die Kanalverbindung.
- Wenn Sie `openclaw agent` über System-Cron oder einen anderen externen Scheduler steuern, umschließen Sie den Aufruf mit einer Eskalation zum erzwungenen Beenden, obwohl die CLI `SIGTERM`/`SIGINT` bereits verarbeitet. Gateway-gestützte Ausführungen fordern das Gateway auf, angenommene Ausführungen abzubrechen; `--local`-Ausführungen erhalten dasselbe Abbruchsignal. Bevorzugen Sie bei GNU `timeout` `timeout -k 60 600 openclaw agent ...` gegenüber dem einfachen `timeout 600 ...` — der Wert `-k` dient als letzte Absicherung, falls der Prozess nicht rechtzeitig beendet werden kann. Verwenden Sie für systemd-Units ein `SIGTERM`-Stoppsignal mit einem Kulanzzeitfenster (`TimeoutStopSec`) vor dem endgültigen Beenden. Wird ein `--run-id` erneut verwendet, während die ursprüngliche Gateway-Ausführung noch aktiv ist, wird das Duplikat als laufend gemeldet, statt eine zweite Ausführung zu starten.

<AccordionGroup>
  <Accordion title="Absicherung isolierter Ausführungen">
    - Isolierte Ausführungen versuchen nach bestem Ermessen, nach Abschluss die nachverfolgten Browser-Tabs/-Prozesse für ihre `cron:<jobId>`-Sitzung zu schließen, und geben alle für den Job erstellten gebündelten MCP-Laufzeitinstanzen über denselben gemeinsam genutzten Bereinigungspfad frei, der auch für Ausführungen der Hauptsitzung und benutzerdefinierter Sitzungen verwendet wird. Bereinigungsfehler werden ignoriert, damit das Cron-Ergebnis weiterhin maßgeblich bleibt.
    - Isolierte Ausführungen mit der eng begrenzten Berechtigung zur Cron-Selbstbereinigung können den Scheduler-Status, eine auf den eigenen Job beschränkte Liste und den Ausführungsverlauf dieses Jobs lesen und dürfen ausschließlich ihren eigenen Job entfernen.
    - Isolierte Ausführungen schützen vor veralteten Bestätigungsantworten: Wenn das erste Ergebnis lediglich eine vorläufige Statusaktualisierung ist (`on it`, `pulling everything together` und ähnliche Hinweise) und kein untergeordneter Subagent mehr für die endgültige Antwort zuständig ist, fordert OpenClaw einmal erneut das tatsächliche Ergebnis an, bevor es zugestellt wird.
    - Strukturierte Metadaten zur Ausführungsverweigerung (einschließlich Node-Host-`UNAVAILABLE`-Wrappern, deren verschachtelter Fehler mit `SYSTEM_RUN_DENIED` oder `INVALID_REQUEST` beginnt) werden erkannt, damit ein blockierter Befehl nicht als erfolgreiche Ausführung gemeldet wird, während gewöhnliche Prosa des Assistenten nicht fälschlich als Verweigerung interpretiert wird.
    - Fehler des Agenten auf Ausführungsebene gelten auch ohne Antwort-Payload als Jobfehler, sodass Modell-/Provider-Fehler die Fehlerzähler erhöhen und Fehlermeldungen auslösen, statt den Job als erfolgreich abzuschließen.
    - Wenn ein Job `timeoutSeconds` erreicht, bricht Cron die Ausführung ab und gewährt ihr ein kurzes Bereinigungszeitfenster. Wird sie nicht rechtzeitig beendet, hebt die Gateway-eigene Bereinigung die Sitzungszuordnung dieser Ausführung zwangsweise auf, bevor Cron das Zeitlimit protokolliert, damit in der Warteschlange befindliche Chat-Aufgaben nicht hinter einer veralteten Verarbeitungssitzung hängen bleiben.
    - Verzögerungen bei Einrichtung und Start erhalten ein phasenspezifisches Zeitlimit (beispielsweise `cron: isolated agent setup timed out before runner start` oder `cron: isolated agent run stalled before execution start (last phase: context-engine)`). Diese Watchdogs decken eingebettete und CLI-gestützte Provider bereits ab, bevor deren externer CLI-Prozess startet, und werden unabhängig von langen `timeoutSeconds`-Werten begrenzt, damit Fehler beim Kaltstart, bei der Authentifizierung oder beim Kontext schnell sichtbar werden.

  </Accordion>
  <Accordion title="Aufgabenabgleich">
    Beim Abgleich von Cron-Aufgaben hat zunächst die Laufzeit und erst danach der dauerhaft gespeicherte Verlauf Vorrang: Eine aktive Cron-Aufgabe bleibt aktiv, solange die Cron-Laufzeit den Job weiterhin als laufend nachverfolgt, selbst wenn noch eine alte Zeile einer untergeordneten Sitzung vorhanden ist. Sobald die Laufzeit nicht mehr für den Job zuständig ist und ein Kulanzzeitfenster von 5 Minuten abgelaufen ist, prüfen Wartungsroutinen die dauerhaft gespeicherten Ausführungsprotokolle und den Jobstatus für die passende `cron:<jobId>:<startedAt>`-Ausführung. Ein dort vorhandenes Endergebnis schließt das Aufgabenbuch ab; andernfalls kann die Gateway-eigene Wartung die Aufgabe als `lost` markieren. Eine Offline-Prüfung über die CLI kann den Status aus dem dauerhaft gespeicherten Verlauf wiederherstellen, aber ihre eigene leere Menge prozessinterner aktiver Jobs beweist nicht, dass eine Gateway-eigene Ausführung beendet ist.
  </Accordion>
</AccordionGroup>

## Zeitplantypen

| Art       | CLI-Flag           | Beschreibung                                                                                             |
| --------- | ------------------ | -------------------------------------------------------------------------------------------------------- |
| `at`      | `--at`             | Einmaliger Zeitstempel (ISO 8601 oder relativ wie `20m`)                                                 |
| `every`   | `--every`          | Festes Intervall (`10m`, `1h`, `1d`)                                                                    |
| `cron`    | `--cron`           | Cron-Ausdruck mit 5 oder 6 Feldern und optionalem `--tz`                                               |
| `on-exit` | `--on-exit`        | Einmal auslösen, wenn ein überwachter Befehl beendet wird (Ereignisauslöser; übersteht den Abbau des Turns; optionales `--on-exit-cwd`) |
| `stream`  | `--stream-command` | Aus gebündelten Zeilen auslösen, die von einem überwachten langlebigen Befehl erzeugt werden              |

Zeitstempel ohne Zeitzone werden als UTC behandelt. Fügen Sie `--tz America/New_York` hinzu, um eine `--at`-Datums-/Zeitangabe ohne Offset in dieser IANA-Zeitzone zu interpretieren oder einen Cron-Ausdruck darin auszuwerten. Cron-Ausdrücke ohne `--tz` verwenden die Zeitzone des Gateway-Hosts. `--tz` ist mit `--every` oder `--on-exit` nicht gültig.

Wiederkehrende Ausdrücke zur vollen Stunde (Minute `0` mit einem Platzhalter im Stundenfeld) werden automatisch um bis zu 5 Minuten gestaffelt, um Lastspitzen zu reduzieren. Verwenden Sie `--exact`, um eine präzise Zeitsteuerung zu erzwingen, oder `--stagger 30s` für ein explizites Zeitfenster (nur Cron-Zeitpläne).

### Migration von Heartbeat-Aufgaben

Ältere Heartbeat-Zwischenspeicher unterstützten einen strukturierten `tasks:`-Block. Führen Sie nach dem Upgrade `openclaw doctor --fix` aus, um jeden Eintrag in einen gewöhnlichen, bearbeitbaren Cron-Job der Hauptsitzung umzuwandeln. Doctor behält das Intervall und den vorherigen Zeitpunkt der letzten Ausführung bei, erstellt die Jobs vor dem Entfernen des Blocks und führt dieselben Deklarationsschlüssel bei erneuter Ausführung sicher zusammen.

Diese migrierten Jobs enthalten öffentliche `systemEvent`-Payloads, sodass `openclaw cron list`, `get`, `edit` und `remove` sowie das Cron-Tool sie wie andere Jobs verwalten. Ihre Ausführung verwendet den abgesicherten Heartbeat-Weckmechanismus für Aufgaben: Aktive Zeiten, Mindestabstände, Überlastungsschutz und Wiederholungsversuche bei Auslastung gelten weiterhin, während Cron den unabhängigen Rhythmus jeder Aufgabe verwaltet. Jobs, die im selben Zusammenfassungszeitfenster fällig werden, können sich einen Heartbeat-Turn teilen. Ein geplanter Termin außerhalb der aktiven Heartbeat-Zeiten wird übersprungen und beim nächsten Termin des Jobs erneut versucht.

Der Heartbeat-Zwischenspeicher enthält jetzt ausschließlich Überwachungsprosa. Laufzeit-Heartbeats interpretieren `tasks:`-Text nicht als Zeitpläne; erstellen Sie neue wiederkehrende Aufgaben mit Cron.

### Stream-Quellen

Ein Stream-Zeitplan hält einen vom Betreiber erstellten argv-Befehl unter dem Gateway am Laufen und löst den Job anhand seiner stdout- und stderr-Zeilen aus. Stream-Zeitpläne sind ereignisgesteuert, niemals zeitgebunden und erfordern `cron.triggers.enabled: true`, da der langlebige Befehl derselben unbeaufsichtigten Vertrauensklasse wie Trigger-Skripte angehört. Durch Deaktivieren oder Entfernen des Jobs wird der Prozess beendet; beim Herunterfahren des Gateways wird auf den Abbau des Prozessbaums gewartet. Schnelle Fehler führen zu einem Neustart mit dem integrierten Fehler-Backoff von Cron. Fünf aufeinanderfolgende Ausführungen mit einer Dauer von weniger als 60 Sekunden versetzen den Job in einen Fehlerstatus und verwenden den normalen Pfad für Fehlerwarnungen; aktivieren Sie den Job manuell erneut, um die Neustartbegrenzung zurückzusetzen.

```bash
openclaw cron add \
  --name "Build-Ereignisstream" \
  --stream-command '["node","scripts/build-events.mjs"]' \
  --stream-mode match \
  --stream-match '^(failed|recovered):' \
  --stream-batch-ms 250 \
  --session isolated \
  --message "Diese Build-Ereignisse untersuchen."
```

`mode: "line"` (die Standardeinstellung) akzeptiert jede Zeile. `mode: "match"` akzeptiert nur Zeilen, die dem kompilierten regulären Ausdruck `match` entsprechen. Ein Batch wird nach `batchMs` ohne Aktivität (standardmäßig 250 ms, begrenzt auf 50–5000) oder bei `maxBatchBytes` (standardmäßig 16384, begrenzt auf 1024–65536) geschlossen. Beim Erreichen der Byte-Begrenzung endet der Batch mit `[truncated]`. Im Übereinstimmungsmodus werden vollständige Zeilen stets anhand ihres gesamten Texts ausgewertet, auch über `maxBatchBytes` hinaus (nur der zugestellte Batch wird gekürzt); eine an der begrenzten Rohdatenaufnahme abgeschnittene Zeile ist lediglich ein Präfix und wird daher als nicht übereinstimmend behandelt, statt ein am Ende verankertes Muster auf den abgeschnittenen Text reagieren zu lassen. Der Batch wird an den Systemereignistext oder die Agent-Turn-Nachricht angehängt. Befehls-Payloads werden für Stream-Zeitpläne abgelehnt, da sonst die Prozesszuständigkeit zwischen Quellbefehl und Payload-Befehl nicht eindeutig wäre.

Pro Job werden nur eine Payload-Auslösung und ein begrenzter ausstehender Batch aufbewahrt. Zeilen, die während der Ausführung einer Payload oder vor Ablauf des integrierten Triggerintervalls von 30 Sekunden eintreffen, werden in diesem ausstehenden Batch zusammengeführt, statt eine unbegrenzte Warteschlange aufzubauen. Eine einzige serialisierte Instanz zeichnet Gate-Verwerfungen, Payload-Fehler und Auslösungen bei nicht laufendem Zustand in `streamDroppedBatches` auf; begrenzte Zusammenführungen erhöhen `streamCoalescedBatches`. Fehlgeschlagene Payloads werden nicht erneut versucht, da sie möglicherweise nicht idempotent sind. Eine logische Quellidentität bleibt über Neustarts überwachter Kindprozesse hinweg stabil, wechselt jedoch, wenn die Quelle deaktiviert, entfernt oder ersetzt wird. Dadurch können Batches aus der ausgemusterten Quelle selbst nach einer Bearbeitung von A zu B und zurück zu A nicht ausgelöst werden. Nach Abschluss eines Stopps haben verspätete Callbacks eines alten Kindprozesses keine Wirkung. V1 enthält keine native WebSocket-Quelle; überbrücken Sie eine solche Quelle mit einem argv-Befehl wie `websocat wss://example.invalid/events`.

Wenn ein Stream-Job zusätzlich `trigger.script` besitzt, wird das Gate einmal pro geschlossenem Batch ausgeführt. Der aktuelle Batch ist neben `trigger.state` als tief eingefrorener `trigger.streamBatch`-String verfügbar. `fire: false` verwirft diesen Batch nach der dauerhaften Speicherung des Gate-Status. `fire: true` behält die bestehende Semantik der Trigger-Nachricht bei und hängt anschließend den Batch an die resultierende Payload an. Ein Stream-Job kann stattdessen eine Skript-Payload ohne Bedingungs-Gate verwenden; dieses Skript erhält den Batch über denselben `trigger.streamBatch`-Wert. Die Kombination einer Skript-Payload mit einem Bedingungs-Gate wird abgelehnt, da beide für den dauerhaft gespeicherten `trigger.state`-Slot zuständig wären.

### Dynamischer Rhythmus (Taktung)

Wiederkehrende Jobs können `pacing.min` und/oder `pacing.max` auf Dauerangaben wie `15m` oder `4h` setzen; mindestens eine Grenze ist erforderlich. Verwenden Sie `--pacing-min` und `--pacing-max` zusammen mit `cron add|edit` (`--clear-pacing` entfernt beide Grenzen).

Während eines isolierten Laufs kann ein getakteter Job das Tool `cron` mit `action: "next_check"` und `in: "30m"` aufrufen. Der Vorschlag gilt nur für diesen derzeit laufenden Job und wird ab dem erfolgreichen Abschluss des Laufs bemessen. OpenClaw begrenzt ihn stillschweigend auf die konfigurierten Grenzen.

Eine Taktung ohne Vorschlag lässt den normalen Zeitplan unverändert. Fehlgeschlagene, wegen Zeitüberschreitung abgebrochene und übersprungene Läufe verwerfen den Vorschlag, sodass das bestehende Wiederholungs- und Fehler-Backoff-Verhalten Vorrang hat. Das manuelle Erzwingen eines wiederkehrenden Jobs erfolgt außerhalb des regulären Ablaufs und behält dessen anstehenden natürlichen oder getakteten Zeitpunkt bei. Bei bedingungsgesteuerten Jobs bleibt das integrierte Mindestintervall eine Untergrenze, selbst wenn ein Vorschlag eine frühere Prüfung anfordert.

### Tag des Monats und Wochentag verwenden ODER-Logik

Cron-Ausdrücke werden von [croner](https://github.com/Hexagon/croner) geparst. Wenn sowohl das Feld für den Tag des Monats als auch das Feld für den Wochentag keine Platzhalter sind, gilt bei croner eine Übereinstimmung, wenn **eines der beiden** Felder übereinstimmt, nicht beide. Dies entspricht dem standardmäßigen Verhalten von Vixie cron.

```bash
# Beabsichtigt: „9 Uhr am 15., aber nur wenn es ein Montag ist“
# Tatsächlich:  „9 Uhr an jedem 15. UND 9 Uhr an jedem Montag“
0 9 15 * 1
```

Dies wird ungefähr 5–6-mal pro Monat statt 0–1-mal pro Monat ausgelöst. Um beide Bedingungen vorauszusetzen, verwenden Sie den Wochentagsmodifikator `+` von croner (`0 9 15 * +1`) oder planen Sie nach einem Feld und prüfen Sie das andere im Prompt oder Befehl Ihres Jobs.

## Ereignisauslöser (Bedingungsüberwachung)

Ein Ereignisauslöser ergänzt einen `every`-, `cron`- oder `stream`-Zeitplan um ein unbeaufsichtigt ausgeführtes Bedingungsskript. Zeitpläne werten es zum fälligen Zeitpunkt aus; Stream-Zeitpläne werten es für jeden abgeschlossenen Batch aus. Cron führt die normale Nutzlast nur aus, wenn das Skript `fire: true` zurückgibt:

```json5
{
  schedule: { kind: "every", everyMs: 30000 },
  trigger: {
    // Wird nur ausgelöst, wenn sich der beobachtete Status von der letzten Auswertung unterscheidet.
    script: "const res = await tools.call('exec', { command: 'gh pr checks 123 --json state -q \\'.[].state\\' | sort -u' }); const status = String(res?.result?.details?.aggregated ?? '').trim(); json({ fire: status !== trigger.state?.status, message: `PR 123 CI: ${trigger.state?.status ?? 'unknown'} -> ${status}`, state: { status } });",
    once: false,
  },
  payload: { kind: "agentTurn", message: "Untersuchen Sie die Änderung des CI-Status." },
}
```

Das Skript muss `{ fire, message?, state? }` zurückgeben. Der vorherige JSON-Zustand ist als tiefgehend eingefrorenes `trigger.state` verfügbar; Stream-Gates erhalten außerdem den aktuellen Batch als `trigger.streamBatch`. Geben Sie einen neuen `state`-Wert zurück, um ihn dauerhaft zu speichern. Der Zustand ist auf 16 KB begrenzt. Wenn ein auslösendes Ergebnis `message` enthält, hängt cron es vor der Ausführung an den Systemereignistext oder die Agent-Turn-Nachricht an. `once: true` deaktiviert den Job nach seiner ersten erfolgreich ausgelösten Nutzlast.

`fire: false` speichert den Auswertungszustand und die Zähler dauerhaft und plant anschließend neu, ohne einen Laufverlauf zu erstellen. Wenn ein ausgelöster Nutzlastlauf fehlschlägt, wird der zurückgegebene `state` **nicht** gespeichert – die nächste Auswertung sieht den vorherigen Zustand und kann erneut auslösen. Schreiben Sie Skripte daher als schreibgeschützte Prüfungen und belassen Sie Aktionen in der Nutzlast. Auslöserzeitpläne haben ein integriertes Mindestintervall von 30 Sekunden. Jede Auswertung hat ein Zeitbudget von 30 Sekunden Echtzeit und bis zu 5 Tool-Aufrufe.

Richten Sie Überwachungen auf **handlungsrelevante Zustände** aus, nicht nur auf Erfolge: Eine Überwachung, die verstummt, wenn ihre Prüfung fehlschlägt oder eine Zeitüberschreitung auftritt, erscheint gesund, obwohl sie defekt ist. Vergleichen Sie die Beobachtung mit `trigger.state` und geben Sie einen neuen Zustand zurück, um Duplikate zu vermeiden; verlassen Sie sich nicht auf Modell- oder Prozessspeicher. Gestalten Sie `message` beim Auslösen eigenständig, da es zum vollständigen Ereigniskontext des ausgelösten Laufs wird.

<Warning>
Die Aktivierung von `cron.triggers.enabled` gestattet sowohl bedingungsgesteuerten Skripten als auch `script`-Nutzlasten die unbeaufsichtigte Ausführung mit der **vollständigen Tool-Richtlinie des besitzenden Agenten, einschließlich `exec`**. Behandeln Sie dies als unbeaufsichtigte Codeausführung mit den Berechtigungen dieses Agenten; lassen Sie die Funktion deaktiviert, sofern nicht jeder Agent, der Cron-Jobs erstellen darf, entsprechend vertrauenswürdig ist.
</Warning>

Erstellen Sie eine Überwachung aus einer lokalen Skriptdatei (`-` liest das Skript von stdin):

```bash
openclaw cron add \
  --name "PR-CI-Überwachung" \
  --every 30s \
  --trigger-script ./watch-pr-ci.js \
  --message "Reagieren Sie auf die Änderung des CI-Status" \
  --session isolated
```

## Nutzlasten

Jeder Job enthält genau eine durch ein Flag ausgewählte Nutzlastart:

| Nutzlast      | Flag                                           | Ausführung                                                 |
| ------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Systemereignis | `--system-event <text>`                        | Wird in die Hauptsitzung eingereiht, löst selbst keinen Modellaufruf aus |
| Agentennachricht | `--message <text>`                             | Ein modellgestützter Agent-Turn                            |
| Befehl        | `--command <shell>` oder `--command-argv <json>` | Eine Shell/ein Prozess auf dem Gateway-Host, kein Modellaufruf |
| Skript        | `--script <file\|->`                           | Ein unbeaufsichtigtes Code-Modus-Skript, das die Tools des besitzenden Agenten verwendet |

Eine zusätzliche Nutzlastart, `heartbeat`, ist systemeigen: Das Gateway gleicht für jeden Agenten mit aktiviertem Heartbeat genau einen Heartbeat-Überwachungsjob ab (siehe [Heartbeat](/de/gateway/heartbeat)). Er erscheint in `cron list --all`, kann jedoch nicht über die CLI oder API erstellt oder bearbeitet werden. Die Heartbeat-Konfiguration wird beim Start, beim erneuten Laden der Konfiguration oder durch `openclaw doctor --fix` in den dauerhaft gespeicherten Überwachungszeitplan übernommen. Wenn Cron deaktiviert ist, wird die Überwachung nicht ausgeführt und es läuft kein ersatzweiser Heartbeat-Timer.

### Optionen für Agent-Turns

<ParamField path="--message" type="string" required>
  Prompt-Text (für isolierte Jobs sowie Jobs der aktuellen oder einer benutzerdefinierten Sitzung erforderlich).
</ParamField>
<ParamField path="--model" type="string">
  Modellüberschreibung; muss zu einem zulässigen Modell aufgelöst werden, andernfalls schlägt der Lauf mit einem Validierungsfehler fehl.
</ParamField>
<ParamField path="--fallbacks" type="string">
  Liste der Ausweichmodelle pro Job, zum Beispiel `--fallbacks openai/gpt-5.6-sol,openrouter/meta-llama/llama-3.3-70b-instruct:free`. Übergeben Sie `--fallbacks ""` für einen strikten Lauf ohne Ausweichmodelle.
</ParamField>
<ParamField path="--clear-fallbacks" type="boolean">
  Entfernt bei `cron edit` die Ausweichmodellüberschreibung pro Job, sodass der Job der konfigurierten Ausweichpriorität folgt. Kann nicht mit `--fallbacks` kombiniert werden.
</ParamField>
<ParamField path="--clear-model" type="boolean">
  Entfernt bei `cron edit` die Modellüberschreibung pro Job, sodass der Job der normalen Cron-Modellpriorität folgt (gespeicherte Überschreibung der Cron-Sitzung, andernfalls Agenten-/Standardmodell). Kann nicht mit `--model` kombiniert werden.
</ParamField>
<ParamField path="--thinking" type="string">
  Überschreibung der Denkstufe (`off|minimal|low|medium|high|xhigh|adaptive|max|ultra`). Die verfügbaren Stufen hängen weiterhin vom ausgewählten Modell und der Agentenlaufzeit ab.
</ParamField>
<ParamField path="--clear-thinking" type="boolean">
  Entfernt bei `cron edit` die Denkstufenüberschreibung pro Job. Kann nicht mit `--thinking` kombiniert werden.
</ParamField>
<ParamField path="--light-context" type="boolean">
  Überspringt das Einfügen der Workspace-Bootstrap-Dateien.
</ParamField>
<ParamField path="--tools" type="string">
  Beschränkt, welche Tools der Job verwenden darf, zum Beispiel `--tools exec,read`.
</ParamField>

Neue Jobs, die Tools ausführen können, speichern stets eine ausdrückliche Tool-Richtlinie. Von einem Agenten erstellte Jobs
sind auf die Tools begrenzt, die dem erstellenden Turn zur Verfügung stehen, und der Agent kann die
gespeicherte Liste nicht erweitern. Jobs, die von einem authentifizierten Operator ohne `--tools` erstellt werden, speichern eine
uneingeschränkte `*`-Richtlinie; `cron edit --clear-tools` stellt diese ausdrücklich uneingeschränkte
Richtlinie wieder her. Bestehende Jobs, die vor der Einführung ausdrücklicher Tool-Richtlinien erstellt wurden, behalten ihr aktuelles Verhalten bei,
bis ihre Tool-Richtlinie ausdrücklich bearbeitet oder der Job neu erstellt wird.

`--model` legt das primäre Modell des Jobs fest; es ersetzt keine `/model`-Überschreibung einer Sitzung, sodass konfigurierte Ausweichketten weiterhin zusätzlich angewendet werden. Ein nicht auflösbares oder unzulässiges Modell lässt den Lauf mit einem ausdrücklichen Validierungsfehler fehlschlagen, statt stillschweigend auf den Standard zurückzufallen. Wenn ein Job `--model`, aber keine ausdrückliche oder konfigurierte Ausweichliste hat, übergibt OpenClaw eine leere Ausweichüberschreibung, statt das primäre Agentenmodell stillschweigend als verborgenes Wiederholungsziel anzuhängen.

Priorität der Modellauswahl für isolierte Jobs, beginnend mit der höchsten:

1. Nutzlast `model` pro Job (ausdrückliche Konfiguration; ein unzulässiges Modell lässt den Lauf fehlschlagen)
2. Modellüberschreibung des Gmail-Hooks (nur wenn der Lauf von Gmail stammt und diese Überschreibung zulässig ist)
3. Vom Benutzer ausgewählte, gespeicherte Modellüberschreibung der Cron-Sitzung
4. Agenten-/Standardmodellauswahl

Der schnelle Modus folgt der aufgelösten Live-Auswahl. Wenn die ausgewählte Modellkonfiguration `params.fastMode` enthält, verwendet isoliertes Cron dies standardmäßig; eine gespeicherte `fastMode`-Überschreibung der Sitzung (und anschließend ein `fastModeDefault` des Agenten) hat weiterhin in beide Richtungen Vorrang vor der Modellkonfiguration. Der automatische Modus verwendet den `params.fastAutoOnSeconds`-Grenzwert des Modells, standardmäßig 60 Sekunden.

Wenn ein Lauf auf eine Live-Übergabe beim Modellwechsel trifft, wiederholt Cron den Versuch mit dem gewechselten Provider/Modell und speichert diese Auswahl (sowie jedes neue Authentifizierungsprofil) für den aktiven Lauf dauerhaft. Die Wiederholungsversuche sind begrenzt: Nach dem ersten Versuch und 2 Wechselwiederholungen bricht Cron ab, statt eine Endlosschleife zu bilden.

Vor dem Start eines isolierten Laufs prüft OpenClaw erreichbare lokale Endpunkte für konfigurierte `api: "ollama"`- und `api: "openai-completions"`-Provider, deren `baseUrl` Loopback, privates Netzwerk oder `.local` ist. Diese Vorabprüfung durchläuft die konfigurierte Ausweichkette des Jobs und markiert den Lauf erst dann als `skipped`, wenn jeder Kandidat nicht erreichbar ist; `--fallbacks ""` beschränkt diese Prüfung strikt auf das primäre Modell. Bei einem ausgefallenen Endpunkt wird der Lauf mit einem eindeutigen Fehler als `skipped` aufgezeichnet, statt einen Modellaufruf zu starten. Das Ergebnis wird pro Endpunkt (nicht pro Job oder Modell) 5 Minuten lang zwischengespeichert, sodass viele fällige Jobs, die denselben ausgefallenen lokalen Ollama-/vLLM-/SGLang-/LM-Studio-Server verwenden, nur eine Prüfung statt eines Anfragesturms verursachen. Durch die Vorabprüfung übersprungene Läufe erhöhen den Backoff bei Ausführungsfehlern nicht; setzen Sie `failureAlert.includeSkipped`, um wiederholte Überspringungswarnungen zu aktivieren.

### Befehlsnutzlasten

Befehlsnutzlasten führen deterministische Skripte innerhalb des Gateway-Schedulers aus, ohne einen modellgestützten Turn zu starten. Sie werden auf dem Gateway-Host ausgeführt, erfassen stdout/stderr, zeichnen den Lauf im Cron-Verlauf auf und verwenden dieselben Zustellmodi `announce`, `webhook` und `none` wie Agent-Turn-Jobs.

<Note>
Befehls-Cron ist eine Gateway-Automatisierungsoberfläche für Operator-Administratoren, kein `tools.exec`-Aufruf eines Agenten. Das Erstellen, Aktualisieren, Entfernen oder manuelle Ausführen von Cron-Jobs erfordert `operator.admin`; geplante Befehlsläufe werden später innerhalb des Gateway-Prozesses als diese vom Administrator erstellte Automatisierung ausgeführt. Die Exec-Richtlinie des Agenten (`tools.exec.mode`, Genehmigungsaufforderungen, Tool-Zulassungslisten pro Agent) gilt für modellseitig sichtbare Exec-Tools, nicht für Befehls-Cron-Nutzlasten.
</Note>

```bash
openclaw cron create "*/15 * * * *" \
  --name "Warteschlangentiefenprüfung" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>` speichert `argv: ["sh", "-lc", <shell>]`. Verwenden Sie `--command-argv '["node","scripts/report.mjs"]'` für die exakte argv-Ausführung ohne Shell-Parsing. Optionale `--command-env KEY=VALUE` (wiederholbar), `--command-input`, `--timeout-seconds` (standardmäßig 10 Minuten), `--no-output-timeout-seconds` und `--output-max-bytes` steuern die Prozessumgebung, stdin und Ausgabegrenzen.

Der zugestellte Text wird aus der Prozessausgabe abgeleitet: Nicht leeres stdout hat Vorrang; wenn stdout leer und stderr nicht leer ist, wird stderr zugestellt; wenn beide vorhanden sind, sendet Cron einen kleinen `stdout:`-/`stderr:`-Block. Der Exit-Code `0` zeichnet den Lauf als `ok` auf; ein Exit-Code ungleich null, ein Signal, eine Zeitüberschreitung oder eine Zeitüberschreitung wegen fehlender Ausgabe zeichnet `error` auf und kann Fehlerwarnungen auslösen. Ein Befehl, der ausschließlich `NO_REPLY` ausgibt, verwendet die normale Unterdrückung stiller Cron-Token und sendet nichts an den Chat zurück.

### Skriptnutzlasten

Skript-Payloads werden ohne Benutzeroberfläche im selben Code-Mode-Executor wie Trigger-Skripte ausgeführt, ohne einen Turn eines dialogorientierten Agenten zu starten. Aktivieren Sie `cron.triggers.enabled`, bevor Sie sie erstellen oder ausführen; diese Schutzschranke für gefährliche Automatisierung gilt sowohl für Trigger-Skripte als auch für Skript-Payloads. Skript-Jobs unterstützen nur die Sitzungsziele `main` und `isolated`.

```bash
openclaw cron create "0 * * * *" \
  --name "Hourly queue check" \
  --script ./automation/check-queue.js \
  --script-timeout-seconds 300 \
  --script-tool-budget 50 \
  --session isolated \
  --announce
```

Verwenden Sie `--script <file|->`, um JavaScript aus einer Datei oder von stdin zu lesen. Das Zeitlimit beträgt standardmäßig 300 Sekunden und ist auf 900 begrenzt; das Tool-Budget beträgt standardmäßig 50 Aufrufe und ist auf 200 begrenzt. Diese Payload-Budgets sind von den kleineren Auswertungsbudgets der Trigger-Schutzschranke getrennt.

Das Skript kann ein Objekt mit diesen optionalen Feldern zurückgeben:

- `notify`: Text, der über den Übermittlungsmodus `announce`, `webhook` oder `none` des Jobs übermittelt wird. Wenn das Feld fehlt, wird nichts übermittelt. Bei einem `main`-Job wird der Text zu einem Systemereignis.
- `wake`: `"now"` fordert unmittelbar nach dem Einreihen von `notify` (oder eines kompakten Abschlussereignisses) einen Heartbeat an; `"next-heartbeat"` reiht das Ereignis für den nächsten Heartbeat ein.
- `state`: JSON-Zustand, auf 16 KB begrenzt und nur nach einer erfolgreichen Ausführung persistiert. Die nächste Ausführung erhält wie bei Trigger-Skripten eine eingefrorene Kopie als `trigger.state`. Da dieser Namespace nur einen persistierten Eigentümer hat, kann ein Skript-Payload nicht mit einem Bedingungstrigger im selben Job kombiniert werden.
- `nextCheck`: Eine Dauer wie `"15m"`. Sie ist nur für Jobs mit aktivierter Taktung gültig und verwendet dieselbe Taktungsbegrenzung wie Vorschläge von Agenten-Turns.

Ausnahmen, Zeitüberschreitungen, ausgeschöpfte Tool-Budgets, ungültige Ergebnisse und `nextCheck` ohne Taktung sind normale Cron-Ausführungsfehler: Sie werden im Ausführungsverlauf, bei der Rückstufung und bei der Behandlung von Fehlerwarnungen berücksichtigt, ohne den zurückgegebenen Zustand zu persistieren.

## Ausführungsarten

| Art             | Wert von `--session` | Wird ausgeführt in        | Am besten geeignet für           |
| --------------- | ------------------- | ------------------------ | ------------------------------- |
| Hauptsitzung    | `main`              | Dedizierte Cron-Aufweck-Lane | Erinnerungen, Systemereignisse |
| Isoliert        | `isolated`          | Dedizierte `cron:<jobId>` | Berichte, Hintergrundaufgaben |
| Aktuelle Sitzung | `current`           | Bei Erstellung gebunden   | Kontextbezogene wiederkehrende Aufgaben |
| Benutzerdefinierte Sitzung | `session:custom-id` | Persistente benannte Sitzung | Workflows, die auf dem Verlauf aufbauen |

<AccordionGroup>
  <Accordion title="Hauptsitzung im Vergleich zu isoliert und benutzerdefiniert">
    Jobs der **Hauptsitzung** reihen ein Systemereignis in eine Cron-eigene Ausführungs-Lane ein und wecken optional den Heartbeat (`--wake now` oder `--wake next-heartbeat`). Sie können für Antworten den letzten Übermittlungskontext der Ziel-Hauptsitzung verwenden, hängen routinemäßige Cron-Turns jedoch nicht an die menschliche Chat-Lane an und verlängern nicht die Aktualität für tägliche oder inaktivitätsbedingte Zurücksetzungen der Zielsitzung. **Isolierte** Jobs führen einen dedizierten Agenten-Turn mit einer neuen Sitzung aus. **Benutzerdefinierte Sitzungen** (`session:xxx`) bewahren den Kontext über mehrere Ausführungen hinweg und ermöglichen so Workflows wie tägliche Stand-ups, die auf vorherigen Zusammenfassungen aufbauen.

    Cron-Ereignisse der Hauptsitzung sind eigenständige Systemereignis-Erinnerungen. Sie enthalten nicht automatisch den standardmäßigen Heartbeat-Prompt oder den Arbeitsbereich des Heartbeat-Monitors; geben Sie dies ausdrücklich im Text des Cron-Ereignisses an, wenn eine Erinnerung diesen Kontext berücksichtigen soll.

  </Accordion>
  <Accordion title="Was „neue Sitzung“ bei isolierten Jobs bedeutet">
    Eine neue Transkript-/Sitzungs-ID pro Ausführung. OpenClaw übernimmt sichere Einstellungen (Einstellungen für Denken/schnell/ausführlich, Labels, explizit vom Benutzer ausgewählte Modell-/Authentifizierungsüberschreibungen), erbt jedoch keinen umgebenden Gesprächskontext aus einer älteren Cron-Zeile: Kanal-/Gruppen-Routing, Sende- oder Warteschlangenrichtlinie, Rechteerweiterung, Ursprung oder ACP-Laufzeitbindung. Verwenden Sie `current` oder `session:<id>`, wenn ein wiederkehrender Job bewusst auf demselben Gesprächskontext aufbauen soll.
  </Accordion>
  <Accordion title="Vertrag für unbeaufsichtigte Ausführungen">
    Isolierte Cron- und Hook-Agenten-Turns sind ausdrücklich unbeaufsichtigt: Es ist niemand anwesend, um Rückfragen zu beantworten oder etwas zu genehmigen. Die endgültige Antwort muss das Ergebnis sein und darf kein Plan, keine Bestätigung und keine Bitte um Eingabe sein. Der Agent gibt `HEARTBEAT_OK` zurück, wenn nichts zu tun ist, und benennt Fehler eindeutig; Cron verwaltet die Richtlinien für Wiederholungsversuche und Fehlerwarnungen.

    Bei vertrauenswürdigen geplanten Jobs haben die eigenen Anweisungen des Jobs Vorrang, wenn sie absichtlich um eine Frage oder einen Plan bitten, und der Agent darf einen nicht mehr benötigten Job entfernen. Externe Hook-Turns erhalten nur den allgemeinen Vertrag für unbeaufsichtigte Ausführungen; über die Grenze für externe Inhalte hinweg erhalten sie weder diese Außerkraftsetzung noch Hinweise zur Selbstentfernung.

  </Accordion>
  <Accordion title="Übermittlung durch Subagenten und Discord">
    Wenn isolierte Cron-Ausführungen Subagenten orchestrieren, wird für die Übermittlung die endgültige Ausgabe des letzten Nachkommen gegenüber veraltetem vorläufigem Text des übergeordneten Agenten bevorzugt. Falls Nachkommen noch ausgeführt werden, unterdrückt OpenClaw diese teilweise Aktualisierung des übergeordneten Agenten, anstatt sie anzukündigen.

    Bei reinen Text-Ankündigungszielen in Discord sendet OpenClaw den kanonischen endgültigen Assistententext einmal, anstatt sowohl gestreamten/vorläufigen Text als auch die endgültige Antwort erneut wiederzugeben. Medien und strukturierte Discord-Payloads werden weiterhin separat übermittelt, damit Anhänge und Komponenten nicht verloren gehen.

  </Accordion>
</AccordionGroup>

## Übermittlung und Ausgabe

| Modus      | Vorgang                                                             |
| ---------- | ------------------------------------------------------------------- |
| `announce` | Endgültigen Text ersatzweise an das Ziel übermitteln, falls der Agent ihn nicht gesendet hat |
| `webhook`  | Payload des Abschlussereignisses per POST an eine URL senden       |
| `none`     | Keine ersatzweise Übermittlung durch den Runner                   |

Verwenden Sie `--announce --channel telegram --to "-1001234567890"` für die Kanalübermittlung. Verwenden Sie für Telegram-Forumsthemen `-1001234567890:topic:123`; OpenClaw akzeptiert auch die Telegram-eigene Kurzform `-1001234567890:123`. Direkte RPC-/Konfigurationsaufrufer können `delivery.threadId` als Zeichenfolge oder Zahl übergeben. Ziele für Slack/Discord/Mattermost verwenden explizite Präfixe (`channel:<id>`, `user:<id>`). Bei Matrix-Raum-IDs wird zwischen Groß- und Kleinschreibung unterschieden; verwenden Sie die exakte Raum-ID oder die Form `room:!room:server` von Matrix.

Wenn die Ankündigungsübermittlung `channel: "last"` verwendet oder `channel` fehlt, kann ein Ziel mit Provider-Präfix wie `telegram:123` den Kanal auswählen, bevor Cron auf den Sitzungsverlauf oder einen einzelnen konfigurierten Kanal zurückgreift. Nur Präfixe, die vom geladenen Plugin bekannt gegeben werden, sind Provider-Selektoren. Wenn `delivery.channel` explizit angegeben ist, muss das Zielpräfix denselben Provider benennen; `channel: "whatsapp"` mit `to: "telegram:123"` wird abgelehnt, anstatt WhatsApp die Telegram-ID als Telefonnummer interpretieren zu lassen. Präfixe für Zielart und Dienst (`channel:<id>`, `user:<id>`, `imessage:<handle>`, `sms:<number>`) bleiben kanaleigene Zielsyntax und sind keine Provider-Selektoren.

Bei isolierten Jobs wird die Chat-Übermittlung gemeinsam genutzt: Wenn eine Chat-Route verfügbar ist, kann der Agent das Tool `message` auch mit `--no-deliver` verwenden. Wenn der Agent an das konfigurierte/aktuelle Ziel sendet, überspringt OpenClaw die ersatzweise Ankündigung. Andernfalls steuern `announce`, `webhook` und `none` nur, was der Runner nach dem Agenten-Turn mit der endgültigen Antwort tut.

Wenn ein Agent aus einem aktiven Chat eine isolierte Erinnerung erstellt, speichert OpenClaw das beibehaltene aktive Übermittlungsziel für die ersatzweise Ankündigungsroute. Interne Sitzungsschlüssel können kleingeschrieben sein; Provider-Übermittlungsziele werden nicht aus diesen Schlüsseln rekonstruiert, wenn der aktuelle Chat-Kontext verfügbar ist.

Die implizite Ankündigungsübermittlung verwendet konfigurierte Kanal-Zulassungslisten, um veraltete Ziele zu validieren und umzuleiten. Genehmigungen im DM-Kopplungsspeicher sind keine Empfänger für ersatzweise Automatisierung; legen Sie `delivery.to` fest oder konfigurieren Sie den Kanaleintrag `allowFrom`, wenn ein geplanter Job proaktiv an eine DM senden soll.

### Fehlerbenachrichtigungen

Fehlerbenachrichtigungen verwenden einen separaten Zielpfad:

- `cron.failureDestination` legt einen globalen Standard für Fehlerbenachrichtigungen fest.
- `job.delivery.failureDestination` überschreibt diesen pro Job.
- Wenn keines von beiden festgelegt ist und der Job bereits über `announce` übermittelt, greifen Fehlerbenachrichtigungen auf dieses primäre Ankündigungsziel zurück.
- `delivery.failureDestination` wird nur bei `sessionTarget="isolated"`-Jobs unterstützt, sofern der primäre Übermittlungsmodus nicht `webhook` ist.
- `failureAlert.includeSkipped: true` aktiviert für einen Job oder eine globale Cron-Warnrichtlinie wiederholte Warnungen über übersprungene Ausführungen. Übersprungene Ausführungen verwenden einen separaten Zähler für aufeinanderfolgende Überspringungen und beeinflussen daher nicht die Rückstufung bei Ausführungsfehlern.
- `openclaw cron edit` stellt die jobspezifische Warnungsanpassung bereit: `--failure-alert`/`--no-failure-alert`, `--failure-alert-after <n>`, `--failure-alert-channel`, `--failure-alert-to`, `--failure-alert-cooldown`, `--failure-alert-include-skipped`/`--failure-alert-exclude-skipped`, `--failure-alert-mode` und `--failure-alert-account-id`.

### Ausgabesprache

Cron-Jobs leiten die Antwortsprache nicht aus Kanal, Gebietsschema oder vorherigen Nachrichten ab. Fügen Sie die Sprachregel in die geplante Nachricht oder Vorlage ein:

```bash
openclaw cron edit <jobId> \
  --message "Fassen Sie die Aktualisierungen zusammen. Antworten Sie auf Chinesisch; lassen Sie URLs, Code und Produktnamen unverändert."
```

Behalten Sie bei Vorlagendateien die Sprachanweisung im gerenderten Prompt bei und stellen Sie sicher, dass Platzhalter wie `{{language}}` ausgefüllt sind, bevor der Job ausgeführt wird. Wenn die Ausgabe Sprachen mischt, formulieren Sie die Regel ausdrücklich, zum Beispiel: „Verwenden Sie Chinesisch für Fließtext und behalten Sie technische Begriffe auf Englisch bei.“

## CLI-Beispiele

<Tabs>
  <Tab title="Einmalige Erinnerung">
    ```bash
    openclaw cron add \
      --name "Calendar check" \
      --at "20m" \
      --session main \
      --system-event "Next heartbeat: check calendar." \
      --wake now
    ```
  </Tab>
  <Tab title="Wiederkehrender isolierter Job">
    ```bash
    openclaw cron create "0 7 * * *" \
      "Summarize overnight updates." \
      --name "Morning brief" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --announce \
      --channel slack \
      --to "channel:C1234567890"
    ```
  </Tab>
  <Tab title="Überschreibung von Modell und Denken">
    ```bash
    openclaw cron add \
      --name "Deep analysis" \
      --cron "0 6 * * 1" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --message "Weekly deep analysis of project progress." \
      --model "opus" \
      --thinking high \
      --announce
    ```
  </Tab>
  <Tab title="Webhook-Ausgabe">
    ```bash
    openclaw cron create "0 18 * * 1-5" \
      "Summarize today's deploys as JSON." \
      --name "Deploy digest" \
      --webhook "https://example.invalid/openclaw/cron"
    ```
  </Tab>
  <Tab title="Befehlsausgabe">
    ```bash
    openclaw cron create "*/15 * * * *" \
      --name "Queue depth probe" \
      --command "scripts/check-queue.sh" \
      --command-cwd "/srv/app" \
      --announce \
      --channel telegram \
      --to "-1001234567890"
    ```
  </Tab>
</Tabs>

## Jobs verwalten

```bash
# Aktivierte Jobs auflisten
openclaw cron list

# Deaktivierte Jobs einschließen
openclaw cron list --all

# Einen gespeicherten Job als JSON abrufen
openclaw cron get <jobId>

# Einen Job einschließlich des aufgelösten Zustellungswegs anzeigen
openclaw cron show <jobId>

# Aktivieren/deaktivieren, ohne zu löschen
openclaw cron enable <jobId>
openclaw cron disable <jobId>

# Einen Job bearbeiten
openclaw cron edit <jobId> --message "Aktualisierter Prompt" --model "opus"

# Einen Job jetzt zwangsweise ausführen
openclaw cron run <jobId>

# Einen Job jetzt zwangsweise ausführen und auf seinen Endstatus warten
openclaw cron run <jobId> --wait --wait-timeout 10m --poll-interval 2s

# Nur ausführen, wenn fällig
openclaw cron run <jobId> --due

# Ausführungsverlauf anzeigen
openclaw cron runs --id <jobId> --limit 50

# Eine bestimmte Ausführung anzeigen
openclaw cron runs --id <jobId> --run-id <runId>

# Einen Job löschen
openclaw cron remove <jobId>

# Agent-Auswahl (Multi-Agent-Konfigurationen)
openclaw cron create "0 6 * * *" "Betriebswarteschlange prüfen" --name "Betriebsprüfung" --session isolated --agent ops
openclaw cron edit <jobId> --clear-agent
```

Beim Archivieren einer Sitzung (Control UI oder `sessions.patch { archived: true }` durch einen Aufrufer mit Operator-Adminrechten) wird jeder aktivierte Cron-Job deaktiviert, der an diese Sitzung gebunden ist: ihre isolierte `cron:<jobId>`-Sitzung, ein `session:<key>`-Ziel oder eine `sessionKey`-Lane für Zustellung/Aktivierung. Durch das Wiederherstellen der Sitzung werden diese Jobs nicht erneut aktiviert; verwenden Sie `openclaw cron enable <jobId>`. Sitzungen mit einem aktivierten gebundenen Job zeigen in der Seitenleiste der Control UI ein Uhrsymbol an.

`openclaw cron run <jobId>` kehrt zurück, nachdem die manuelle Ausführung in die Warteschlange gestellt wurde. Verwenden Sie `--wait` für Hooks beim Herunterfahren, Wartungsskripte oder andere Automatisierungen, die blockieren müssen, bis die eingereihte Ausführung abgeschlossen ist; der zurückgegebene `runId` wird abgefragt (Standard-Zeitüberschreitung `10m`, Abfrageintervall `2s`), und der Prozess wird für den Status `ok` mit `0` beendet, für `error`, `skipped` oder eine Wartezeitüberschreitung dagegen mit einem Wert ungleich null.

Das Agent-Tool `cron` gibt kompakte Job-Zusammenfassungen (`id`, `name`, `enabled`, `nextRunAtMs`, `scheduleKind`, `lastRunStatus`) aus `cron(action: "list")` zurück; verwenden Sie `cron(action: "get", jobId: "...")` für eine vollständige Job-Definition. Direkte Gateway-Aufrufer können `compact: true` an `cron.list` übergeben; wenn dies weggelassen wird, bleibt die vollständige Antwort mit Zustellungsvorschauen erhalten.

`openclaw cron create` ist ein Alias für `openclaw cron add`. Neue Jobs können einen positionellen Zeitplan (`"0 9 * * 1"`, `"every 1h"`, `"20m"` oder einen ISO-Zeitstempel) verwenden, gefolgt von einem positionellen Agent-Prompt. Verwenden Sie `--webhook <url>` bei `cron add|create` oder `cron edit`, um die Nutzlast der abgeschlossenen Ausführung per POST an einen HTTP-Endpunkt zu senden; die Webhook-Zustellung kann nicht mit Flags für die Chat-Zustellung (`--announce`, `--channel`, `--to`, `--thread-id`, `--account`) kombiniert werden. Bei `cron edit`, `--clear-channel`, `--clear-to`, `--clear-thread-id` und `--clear-account` werden diese Routing-Felder einzeln aufgehoben (jedes wird zusammen mit dem zugehörigen Setz-Flag abgelehnt) – anders als `--no-deliver`, das nur die Fallback-Zustellung des Runners deaktiviert.

<Note>
Hinweis zur Modellüberschreibung:

- `openclaw cron add|edit --model ...` ändert das ausgewählte Modell des Jobs.
- Wenn das Modell zulässig ist, erreicht genau dieser Provider/dieses Modell die isolierte Agent-Ausführung.
- Wenn es nicht zulässig ist oder nicht aufgelöst werden kann, lässt Cron die Ausführung mit einem expliziten Validierungsfehler fehlschlagen.
- Patches der API-Nutzlast `cron.update` können `model: null` festlegen, um eine gespeicherte Job-Modellüberschreibung zu löschen.
- `openclaw cron edit <job-id> --clear-model` löscht diese Überschreibung über die CLI (gleiche Wirkung wie der Patch `model: null`) und kann nicht mit `--model` kombiniert werden.
- Konfigurierte Fallback-Ketten gelten weiterhin, da Cron-`--model` ein primäres Job-Modell und keine `/model`-Überschreibung der Sitzung ist.
- `openclaw cron add|edit --fallbacks ...` setzt die Nutzlast `fallbacks` und ersetzt damit die konfigurierten Fallbacks für diesen Job; `--fallbacks ""` deaktiviert Fallbacks und macht die Ausführung strikt. `openclaw cron edit <job-id> --clear-fallbacks` löscht die jobspezifische Überschreibung.
- Ein einfaches `--model` ohne explizite oder konfigurierte Fallback-Liste weicht nicht stillschweigend auf das primäre Agent-Modell als zusätzliches Wiederholungsziel aus.

</Note>

## Webhooks

Das Gateway kann HTTP-Webhook-Endpunkte für externe Auslöser bereitstellen. Aktivieren Sie sie in der Konfiguration:

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
  },
}
```

### Authentifizierung

Jede Anfrage muss das Hook-Token über einen Header enthalten:

- `Authorization: Bearer <token>` (empfohlen)
- `x-openclaw-token: <token>`

Token in Abfragezeichenfolgen werden abgelehnt.

<AccordionGroup>
  <Accordion title="POST /hooks/wake">
    Ein Systemereignis für die Hauptsitzung in die Warteschlange stellen:

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/wake \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"text":"Neue E-Mail empfangen","mode":"now"}'
    ```

    <ParamField path="text" type="string" required>
      Ereignisbeschreibung.
    </ParamField>
    <ParamField path="mode" type="string" default="now">
      `now` oder `next-heartbeat`.
    </ParamField>

  </Accordion>
  <Accordion title="POST /hooks/agent">
    Einen isolierten Agent-Durchlauf ausführen:

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/agent \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"message":"Posteingang zusammenfassen","name":"E-Mail","model":"openai/gpt-5.6-sol"}'
    ```

    Felder: `message` (erforderlich), `name`, `agentId`, `sessionKey` (erfordert `hooks.allowRequestSessionKey=true`), `idempotencyKey`, `wakeMode`, `deliver`, `channel`, `to`, `model`, `thinking`, `timeoutSeconds`.

  </Accordion>
  <Accordion title="Zugeordnete Hooks (POST /hooks/<name>)">
    Benutzerdefinierte Hook-Namen werden über `hooks.mappings` in der Konfiguration aufgelöst. Zuordnungen können beliebige Nutzlasten mithilfe von Vorlagen oder Codetransformationen in `wake`- oder `agent`-Aktionen umwandeln.
  </Accordion>
</AccordionGroup>

<Warning>
Beschränken Sie den Zugriff auf Hook-Endpunkte auf Loopback, Tailnet oder einen vertrauenswürdigen Reverse-Proxy.

- Verwenden Sie ein dediziertes Hook-Token; verwenden Sie Gateway-Authentifizierungstoken nicht erneut.
- Belassen Sie `hooks.path` auf einem dedizierten Unterpfad; `/` wird abgelehnt.
- Legen Sie `hooks.allowedAgentIds` fest, um einzuschränken, welchen effektiven Agent ein Hook ansprechen kann, einschließlich des Standard-Agent, wenn `agentId` weggelassen wird.
- Behalten Sie `hooks.allowRequestSessionKey=false` bei, sofern Sie keine vom Aufrufer ausgewählten Sitzungen benötigen.
- Wenn Sie `hooks.allowRequestSessionKey` aktivieren, legen Sie auch `hooks.allowedSessionKeyPrefixes` fest, um die zulässigen Formen von Sitzungsschlüsseln einzuschränken.
- Hook-Nutzlasten werden standardmäßig mit Sicherheitsgrenzen umschlossen.

</Warning>

## Gmail-PubSub-Integration

Verbinden Sie Auslöser des Gmail-Posteingangs über Google PubSub mit OpenClaw.

<Note>
**Voraussetzungen:** `gcloud`-CLI, `gog` (gogcli), aktivierte OpenClaw-Hooks, Tailscale für den öffentlichen HTTPS-Endpunkt.
</Note>

### Einrichtung per Assistent (empfohlen)

```bash
openclaw webhooks gmail setup --account openclaw@gmail.com
```

Dadurch wird die Konfiguration `hooks.gmail` geschrieben, die Gmail-Voreinstellung aktiviert und Tailscale Funnel als Standard für den Push-Endpunkt (`--tailscale funnel|serve|off`) festgelegt.

<Warning>
Die sitzungsweise Trennung pro Nachricht der Gmail-Voreinstellung trennt den Konversationskontext; sie beschränkt nicht die Tools oder den Arbeitsbereich des Ziel-Agent. Ohne eine benutzerdefinierte Zuordnung, die `agentId` festlegt, werden Gmail-Hooks als Standard-Agent ausgeführt.

Leiten Sie den Hook für nicht vertrauenswürdige Posteingänge an einen dedizierten Lese-Agent weiter, gewähren Sie diesem Agent ausschließlich Lesezugriff oder keinen Zugriff auf den Arbeitsbereich und verweigern Sie Schreibzugriff auf das Dateisystem, Shell, Browser sowie andere unnötige Tools. Wenn er den Haupt-Agent benachrichtigen muss, erlauben Sie nur die erforderliche Übergabe zwischen Agents. Siehe [Prompt-Injection](/de/gateway/security#prompt-injection), [Multi-Agent-Sandbox und -Tools](/de/tools/multi-agent-sandbox-tools) und [`tools.agentToAgent`](/de/gateway/config-tools#toolsagenttoagent).
</Warning>

### Automatischer Gateway-Start

Wenn `hooks.enabled=true` aktiviert und `hooks.gmail.account` festgelegt ist, startet das Gateway beim Hochfahren `gog gmail watch serve` und erneuert die Überwachung automatisch. Legen Sie `OPENCLAW_SKIP_GMAIL_WATCHER=1` fest, um dies zu deaktivieren.

### Einmalige manuelle Einrichtung

<Steps>
  <Step title="GCP-Projekt auswählen">
    Wählen Sie das GCP-Projekt aus, zu dem der von `gog` verwendete OAuth-Client gehört:

    ```bash
    gcloud auth login
    gcloud config set project <project-id>
    gcloud services enable gmail.googleapis.com pubsub.googleapis.com
    ```

  </Step>
  <Step title="Thema erstellen und Gmail-Push-Zugriff gewähren">
    ```bash
    gcloud pubsub topics create gog-gmail-watch
    gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
      --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
      --role=roles/pubsub.publisher
    ```
  </Step>
  <Step title="Überwachung starten">
    ```bash
    gog gmail watch start \
      --account openclaw@gmail.com \
      --label INBOX \
      --topic projects/<project-id>/topics/gog-gmail-watch
    ```
  </Step>
</Steps>

### Gmail-Modellüberschreibung

```json5
{
  hooks: {
    gmail: {
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

Verwenden Sie für nicht vertrauenswürdige Posteingänge das verfügbare Modell der neuesten Generation und höchsten Qualitätsstufe Ihres Providers. Der obige Wert ist ein Beispiel; das Modell muss in Ihrem konfigurierten Katalog und Ihrer Zulassungsliste vorhanden sein.

## Konfiguration

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    triggers: {
      enabled: false,
    },
    webhookToken: "replace-with-dedicated-webhook-token",
    sessionRetention: "24h",
  },
}
```

`webhookToken` wird bei Cron-Webhook-POSTs als `Authorization: Bearer <token>` gesendet.

`cron.store` ist ein logischer Speicherschlüssel und ein Doctor-Migrationspfad, keine aktive JSON-Datei zur manuellen Bearbeitung. Job-Daten befinden sich in SQLite; verwenden Sie für Änderungen die CLI oder die Gateway-API.

Cron deaktivieren: `cron.enabled: false` oder `OPENCLAW_SKIP_CRON=1`.

<AccordionGroup>
  <Accordion title="Wiederholungsverhalten">
    **Einmalige Wiederholung**: Bei vorübergehenden Fehlern (Ratenbegrenzung, Überlastung, Netzwerk, Zeitüberschreitung, Serverfehler) wird ein integrierter Wiederholungszeitplan verwendet. Dauerhafte Fehler deaktivieren den Job sofort.

    **Wiederkehrende Wiederholung**: Bei aufeinanderfolgenden Ausführungsfehlern werden die Abstände nach einem erweiterten Zeitplan verlängert (30s, 60s, 5m, 15m, 60m). Nach der nächsten erfolgreichen Ausführung wird der Backoff zurückgesetzt.

  </Accordion>
  <Accordion title="Wartung">
    `cron.sessionRetention` (Standard `24h`, `false` deaktiviert) bereinigt Einträge isolierter Ausführungssitzungen. Der Ausführungsverlauf behält die neuesten 2000 Endstatuszeilen pro Job bei; verlorene Zeilen behalten ihr 24-Stunden-Bereinigungsfenster.
  </Accordion>
  <Accordion title="Migration des Legacy-Speichers">
    Führen Sie beim Upgrade `openclaw doctor --fix` aus, um ältere `~/.openclaw/cron/jobs.json`-, `jobs-state.json`- und `runs/*.jsonl`-Dateien in SQLite zu importieren und sie mit dem Suffix `.migrated` umzubenennen. Fehlerhafte Job-Zeilen werden von der Laufzeit übersprungen und zur späteren Reparatur oder Prüfung nach `jobs-quarantine.json` kopiert.
  </Accordion>
</AccordionGroup>

## Fehlerbehebung

### Befehlsabfolge

```bash
openclaw status
openclaw gateway status
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
openclaw doctor
```

<AccordionGroup>
  <Accordion title="Cron wird nicht ausgelöst">
    - Prüfen Sie `cron.enabled` und die Umgebungsvariable `OPENCLAW_SKIP_CRON`.
    - Vergewissern Sie sich, dass das Gateway kontinuierlich ausgeführt wird.
    - Überprüfen Sie bei `cron`-Zeitplänen die Zeitzone (`--tz`) im Vergleich zur Zeitzone des Hosts.
    - `reason: not-due` in der Ausführungsausgabe bedeutet, dass die manuelle Ausführung mit `openclaw cron run <jobId> --due` geprüft wurde und der Job noch nicht fällig war.

  </Accordion>
  <Accordion title="Cron wurde ausgelöst, aber keine Zustellung erfolgte">
    - Der Zustellungsmodus `none` bedeutet, dass kein Fallback-Versand durch den Runner erwartet wird. Der Agent kann weiterhin direkt mit dem Tool `message` senden, wenn eine Chat-Route verfügbar ist.
    - Ein fehlendes/ungültiges Zustellungsziel (`channel`/`to`) bedeutet, dass der ausgehende Versand übersprungen wurde.
    - Bei Matrix können kopierte oder ältere Jobs mit kleingeschriebenen `delivery.to`-Raum-IDs fehlschlagen, da bei Matrix-Raum-IDs die Groß-/Kleinschreibung beachtet wird. Bearbeiten Sie den Job und verwenden Sie exakt den Wert `!room:server` oder `room:!room:server` aus Matrix.
    - Kanalauthentifizierungsfehler (`unauthorized`, `Forbidden`) bedeuten, dass die Zustellung aufgrund der Anmeldedaten blockiert wurde.
    - Wenn der isolierte Lauf nur das Stille-Token (`NO_REPLY` / `no_reply`) zurückgibt, unterdrückt OpenClaw die direkte ausgehende Zustellung und den Fallback-Pfad für die Zusammenfassung in der Warteschlange, sodass nichts an den Chat zurückgesendet wird.
    - Wenn der Agent dem Benutzer selbst eine Nachricht senden soll, prüfen Sie, ob der Job über eine nutzbare Route verfügt (`channel: "last"` mit einem vorherigen Chat oder einen expliziten Kanal/ein explizites Ziel).

  </Accordion>
  <Accordion title="Cron oder Heartbeat scheint einen Rollover im /new-Stil zu verhindern">
    - Die Aktualität für tägliche und inaktivitätsbedingte Zurücksetzungen basiert nicht auf `updatedAt`; siehe [Sitzungsverwaltung](/de/concepts/session#session-lifecycle).
    - Cron-Aktivierungen, Heartbeat-Läufe, Ausführungsbenachrichtigungen und Gateway-Verwaltung können die Sitzungszeile für Routing/Status aktualisieren, verlängern jedoch weder `sessionStartedAt` noch `lastInteractionAt`.
    - Bei älteren Zeilen, die vor der Einführung dieser Felder erstellt wurden, kann OpenClaw `sessionStartedAt` aus dem JSONL-Sitzungsheader des Transkripts wiederherstellen, sofern die Datei noch verfügbar ist. Ältere inaktive Zeilen ohne `lastInteractionAt` verwenden diese wiederhergestellte Startzeit als Ausgangspunkt für die Inaktivität.

  </Accordion>
  <Accordion title="Fallstricke bei Zeitzonen">
    - Cron ohne `--tz` verwendet die Zeitzone des Gateway-Hosts.
    - `at`-Zeitpläne ohne Zeitzone werden als UTC behandelt.
    - Heartbeat `activeHours` verwendet die konfigurierte Zeitzonenauflösung.

  </Accordion>
</AccordionGroup>

## Verwandte Themen

- [Automatisierung](/de/automation) — alle Automatisierungsmechanismen auf einen Blick
- [Hintergrundaufgaben](/de/automation/tasks) — Aufgabenprotokoll für Cron-Ausführungen
- [Heartbeat](/de/gateway/heartbeat) — regelmäßige Durchläufe der Hauptsitzung
- [Zeitzone](/de/concepts/timezone) — Zeitzonenkonfiguration
