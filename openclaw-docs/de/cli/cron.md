---
read_when:
    - Sie möchten geplante Aufgaben und Aktivierungen.
    - Sie debuggen die Cron-Ausführung und die Protokolle.
summary: CLI-Referenz für `openclaw cron` (Hintergrundaufträge planen und ausführen)
title: Cron
x-i18n:
    generated_at: "2026-07-26T17:42:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5989a7558f4ae2f046480b6a52e3fa296c95d47b14b11c5bad709fea4af6af3e
    source_path: cli/cron.md
    workflow: 16
---

# `openclaw cron`

Verwalten Sie Cron-Jobs für den Gateway-Scheduler.

<Tip>
Führen Sie `openclaw cron --help` aus, um den vollständigen Befehlsumfang anzuzeigen. Eine konzeptionelle Einführung finden Sie unter [Cron-Jobs](/de/automation/cron-jobs).
</Tip>

<Note>
Alle Cron-Änderungen (`add`/`create`, `update`/`edit`, `remove`, `run`) erfordern `operator.admin`. Läufe mit Befehls-Payload werden direkt im Gateway-Prozess ausgeführt, nicht als `tools.exec`-Tool-Aufruf eines Agenten; `tools.exec.*` und Ausführungsgenehmigungen gelten weiterhin für modellseitig sichtbare Ausführungs-Tools.
</Note>

## Jobs schnell erstellen

`openclaw cron create` ist ein Alias für `openclaw cron add`. Geben Sie bei neuen Jobs zuerst den Zeitplan und danach den Prompt an:

```bash
openclaw cron create "0 7 * * *" \
  "Fasse die Aktualisierungen über Nacht zusammen." \
  --name "Morgenübersicht" \
  --agent ops
```

Verwenden Sie `--webhook <url>`, wenn der Job die fertige Payload per POST senden soll, statt sie an ein Chat-Ziel zuzustellen:

```bash
openclaw cron create "0 18 * * 1-5" \
  "Fasse die heutigen Deployments als JSON zusammen." \
  --name "Deployment-Zusammenfassung" \
  --webhook "https://example.invalid/openclaw/cron"
```

Verwenden Sie `--command` für deterministische Shell-Jobs, die innerhalb von OpenClaw Cron ausgeführt werden, ohne einen isolierten Agenten-/Modelllauf zu starten:

```bash
openclaw cron create "*/15 * * * *" \
  --name "Prüfung der Warteschlangentiefe" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>` speichert `argv: ["sh", "-lc", <shell>]`. Verwenden Sie `--command-argv '["node","scripts/report.mjs"]'` für eine exakte argv-Ausführung. Befehls-Jobs erfassen stdout/stderr, zeichnen den normalen Cron-Verlauf auf und leiten die Ausgabe über dieselben Zustellmodi `announce`, `webhook` oder `none` wie isolierte Jobs weiter. Ein Befehl, der ausschließlich `NO_REPLY` ausgibt, wird unterdrückt.

## Sitzungen

`--session` akzeptiert `main`, `isolated`, `current` oder `session:<id>`.

<AccordionGroup>
  <Accordion title="Sitzungsschlüssel">
    - `main` wird an die Hauptsitzung des Agenten gebunden.
    - `isolated` erstellt für jeden Lauf ein neues Transkript und eine neue Sitzungs-ID.
    - `current` wird zum Erstellungszeitpunkt an die aktive Sitzung gebunden.
    - `session:<id>` wird an einen expliziten persistenten Sitzungsschlüssel gebunden.

  </Accordion>
  <Accordion title="Semantik isolierter Sitzungen">
    Isolierte Läufe setzen den umgebenden Konversationskontext zurück. Kanal- und Gruppen-Routing, Sende-/Warteschlangenrichtlinie, Rechteerweiterung, Ursprung und ACP-Laufzeitbindung werden für den neuen Lauf zurückgesetzt. Sichere Einstellungen und explizit vom Benutzer ausgewählte Modell- oder Authentifizierungsüberschreibungen können über mehrere Läufe hinweg übernommen werden.
  </Accordion>
</AccordionGroup>

## Zustellung

`openclaw cron list` und `openclaw cron show <job-id>` zeigen eine Vorschau der aufgelösten Zustellroute an. Für `channel: "last"` zeigt die Vorschau, ob die Route aus der Hauptsitzung oder der aktuellen Sitzung aufgelöst wurde oder sicher fehlschlagen wird.

Mit einem Provider-Präfix versehene Ziele können nicht aufgelöste Ankündigungskanäle eindeutig zuordnen. Beispielsweise wählt `to: "telegram:123"` Telegram aus, wenn `delivery.channel` nicht angegeben oder `last` ist. Nur Präfixe, die vom geladenen Plugin bekannt gegeben werden, dienen als Provider-Selektoren. Wenn `delivery.channel` explizit angegeben ist, muss das Präfix mit diesem Kanal übereinstimmen; `channel: "whatsapp"` mit `to: "telegram:123"` wird abgelehnt. Dienstpräfixe wie `imessage:` und `sms:` bleiben kanaleigene Zielsyntax.

<Note>
Isolierte `cron add`-Jobs verwenden standardmäßig die Zustellung `--announce`. Verwenden Sie `--no-deliver`, um die Ausgabe intern zu halten. `--deliver` bleibt als veralteter Alias für `--announce` bestehen.
</Note>

### Zustellungsverantwortung

Die Zustellung isolierter Cron-Chats wird zwischen dem Agenten und dem Runner aufgeteilt:

- Der Agent kann mit dem Tool `message` direkt senden, wenn eine Chat-Route verfügbar ist.
- `announce` stellt die endgültige Antwort nur ersatzweise zu, wenn der Agent nicht direkt an das aufgelöste Ziel gesendet hat.
- `webhook` sendet die fertige Payload per POST an eine URL.
- `none` deaktiviert die ersatzweise Zustellung durch den Runner.

Verwenden Sie `cron add|create --webhook <url>` oder `cron edit <job-id> --webhook <url>`, um die Webhook-Zustellung festzulegen. Kombinieren Sie `--webhook` nicht mit Chat-Zustellungsflags wie `--announce`, `--no-deliver`, `--channel`, `--to`, `--thread-id` oder `--account`.

`cron edit <job-id>` kann einzelne Routing-Felder der Zustellung mit `--clear-channel`, `--clear-to`, `--clear-thread-id` und `--clear-account` zurücksetzen (jedes davon wird abgelehnt, wenn es mit dem zugehörigen Setz-Flag kombiniert wird). Im Gegensatz zu `--no-deliver`, das nur die ersatzweise Zustellung durch den Runner deaktiviert, entfernen diese das gespeicherte Feld, sodass der Job diesen Teil seiner Route wieder anhand der Standardwerte auflöst.

`--announce` ist die ersatzweise Zustellung der endgültigen Antwort durch den Runner. `--no-deliver` deaktiviert diese Ersatzfunktion, entfernt aber nicht das Tool `message` des Agenten, wenn eine Chat-Route verfügbar ist.

Aus einem aktiven Chat erstellte Erinnerungen behalten das aktuelle Chat-Ziel für die ersatzweise Ankündigungszustellung bei. Interne Sitzungsschlüssel können kleingeschrieben sein; verwenden Sie sie nicht als maßgebliche Quelle für Provider-IDs mit Groß-/Kleinschreibung wie Matrix-Raum-IDs.

### Fehlerzustellung

Fehlerbenachrichtigungen werden in dieser Reihenfolge aufgelöst:

1. `delivery.failureDestination` im Job.
2. Globales `cron.failureDestination`.
3. Das primäre Ankündigungsziel des Jobs (wenn keines der beiden vorherigen zu einem konkreten Ziel aufgelöst wird).

<Note>
Jobs der Hauptsitzung dürfen `delivery.failureDestination` nur verwenden, wenn der primäre Zustellmodus `webhook` ist. Isolierte Jobs akzeptieren es in allen Modi.
</Note>

Isolierte Cron-Läufe behandeln Fehler des Agenten auf Laufebene auch dann als Jobfehler, wenn keine Antwort-Payload erzeugt wird. Daher erhöhen Modell-/Provider-Fehler weiterhin die Fehlerzähler und lösen Fehlerbenachrichtigungen aus.

Cron-Befehls-Jobs starten keinen isolierten Agentendurchlauf. Ein Exit-Code von null zeichnet `ok` auf; ein Exit-Code ungleich null, ein Signal, eine Zeitüberschreitung oder eine Zeitüberschreitung ohne Ausgabe zeichnet `error` auf und kann denselben Fehlerbenachrichtigungspfad auslösen.

Wenn bei einem isolierten Lauf vor der ersten Modellanfrage eine Zeitüberschreitung auftritt, enthalten `openclaw cron show` und `openclaw cron runs` einen phasenspezifischen Fehler wie `setup timed out before runner start` oder eine Stillstandsmeldung, die die zuletzt bekannte Startphase nennt (beispielsweise `context-engine`). Bei CLI-basierten Providern bleibt die Überwachung vor dem Modell aktiv, bis der externe CLI-Durchlauf beginnt. Dadurch werden Stillstände bei Sitzungssuche, Hook, Authentifizierung, Prompt, und CLI-Einrichtung als Cron-Fehler vor dem Modell gemeldet.

## Zeitplanung

### Einmalige Jobs

`--at <datetime>` plant einen einmaligen Lauf. Datums-/Zeitangaben ohne Offset werden als UTC behandelt, sofern Sie nicht zusätzlich `--tz <iana>` übergeben; dadurch wird die lokale Uhrzeit in der angegebenen Zeitzone interpretiert.

<Note>
Einmalige Jobs werden nach erfolgreicher Ausführung standardmäßig gelöscht. Verwenden Sie `--keep-after-run`, um sie beizubehalten.
</Note>

### Wiederkehrende Jobs

Wiederkehrende Jobs verwenden nach aufeinanderfolgenden Fehlern einen exponentiellen Wiederholungs-Backoff: 30s, 1m, 5m, 15m, 60m. Nach dem nächsten erfolgreichen Lauf kehrt der Zeitplan zum Normalbetrieb zurück.

Übersprungene Läufe werden getrennt von Ausführungsfehlern erfasst. Sie wirken sich nicht auf den Wiederholungs-Backoff aus, aber mit `openclaw cron edit <job-id> --failure-alert-include-skipped` können Fehlerwarnungen auch wiederholte Benachrichtigungen über übersprungene Läufe einschließen.

Bei isolierten Jobs, die einen lokal konfigurierten Modell-Provider verwenden (Basis-URL auf der Loopback-Schnittstelle, in einem privaten Netzwerk oder `.local`), führt Cron vor dem Start des Agentendurchlaufs eine einfache Provider-Vorabprüfung aus: `api: "ollama"`-Provider werden unter `/api/tags` geprüft; andere lokale OpenAI-kompatible Provider (`api: "openai-completions"`, z. B. vLLM, SGLang, LM Studio) werden unter `/models` geprüft. Wenn der Endpunkt nicht erreichbar ist, wird der Lauf als `skipped` aufgezeichnet und bei einem späteren Termin erneut versucht; das Erreichbarkeitsergebnis wird pro Endpunkt 5 Minuten lang zwischengespeichert, damit viele Jobs für denselben lokalen Server diesen nicht mit wiederholten Prüfungen überlasten.

Cron-Jobs, ausstehender Laufzeitstatus und Laufverlauf befinden sich in der gemeinsam genutzten SQLite-Statusdatenbank. Ältere Dateien vom Typ `jobs.json`, `<name>-state.json` und `runs/*.jsonl` werden einmalig importiert und mit dem Suffix `.migrated` umbenannt. Bearbeiten Sie Zeitpläne nach dem Import mit `openclaw cron add|edit|remove`, statt JSON-Dateien zu bearbeiten.

### Manuelle Läufe

`openclaw cron run <job-id>` erzwingt standardmäßig einen Lauf und kehrt zurück, sobald der manuelle Lauf in die Warteschlange gestellt wurde. Erfolgreiche Antworten enthalten `{ ok: true, enqueued: true, runId }`. Verwenden Sie die zurückgegebene `runId`, um das spätere Ergebnis zu prüfen:

```bash
openclaw cron run <job-id>
openclaw cron runs --id <job-id> --run-id <run-id>
```

Fügen Sie `--wait` hinzu, wenn ein Skript blockieren soll, bis genau dieser in die Warteschlange gestellte Lauf einen Endstatus aufzeichnet:

```bash
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
```

Mit `--wait` ruft die CLI weiterhin zuerst `cron.run` auf und fragt danach `cron.runs` nach der zurückgegebenen `runId` ab. Der Befehl wird nur mit `0` beendet, wenn der Lauf mit dem Status `ok` abgeschlossen wird. Er wird mit einem Exit-Code ungleich null beendet, wenn der Lauf mit `error` oder `skipped` abgeschlossen wird, wenn die Gateway-Antwort keine `runId` enthält oder wenn `--wait-timeout` abläuft (standardmäßig `10m`, wobei standardmäßig alle `2s` abgefragt wird). `--poll-interval` muss größer als null sein.

<Note>
Verwenden Sie `--due`, wenn der manuelle Befehl nur ausgeführt werden soll, falls der Job aktuell fällig ist. Wenn `--due --wait` keinen Lauf in die Warteschlange stellt, gibt der Befehl die normale Antwort für einen nicht ausgeführten Lauf zurück, statt eine Abfrage durchzuführen.
</Note>

## Modelle

`cron add|edit --model <ref>` wählt ein zulässiges Modell für den Job aus. `cron add|edit --fallbacks <list>` legt Ersatzmodelle für einzelne Jobs fest, beispielsweise `--fallbacks openrouter/gpt-4.1-mini,openai/gpt-5`; übergeben Sie `--fallbacks ""` für einen strikten Lauf ohne Ersatzmodelle. `cron edit <job-id> --clear-fallbacks` entfernt die jobbezogene Ersatzmodellüberschreibung. `cron edit <job-id> --clear-model` entfernt die jobbezogene Modellüberschreibung, sodass der Job der normalen Rangfolge der Cron-Modellauswahl folgt (eine gespeicherte Cron-Sitzungsüberschreibung, sofern vorhanden, andernfalls das Agenten-/Standardmodell); es kann nicht mit `--model` kombiniert werden. `cron add|edit --thinking <level>` legt eine jobbezogene Denküberschreibung fest; `cron edit <job-id> --clear-thinking` entfernt sie, sodass der Job der normalen Rangfolge für das Cron-Denken folgt, und kann nicht mit `--thinking` kombiniert werden.

<Warning>
Wenn das Modell nicht zulässig ist oder nicht aufgelöst werden kann, lässt Cron den Lauf mit einem expliziten Validierungsfehler fehlschlagen, statt auf die Agenten- oder Standardmodellauswahl des Jobs zurückzugreifen.
</Warning>

Cron-`--model` ist ein **primäres Jobmodell**, keine `/model`-Überschreibung einer Chat-Sitzung. Das bedeutet:

- Konfigurierte Modell-Fallbacks gelten weiterhin, wenn das ausgewählte Jobmodell fehlschlägt.
- Die jobbezogene Payload `fallbacks` ersetzt die konfigurierte Fallback-Liste, wenn sie vorhanden ist.
- Eine leere jobbezogene Fallback-Liste (`--fallbacks ""` oder `fallbacks: []` in der Job-Payload/API) macht den Cron-Lauf strikt.
- Wenn ein Job `--model` besitzt, aber keine Fallback-Liste konfiguriert ist, übergibt OpenClaw eine explizit leere Fallback-Überschreibung, sodass das primäre Agentenmodell nicht als verborgenes Wiederholungsziel angehängt wird.
- Vorabprüfungen lokaler Provider durchlaufen die konfigurierten Fallbacks, bevor ein Cron-Lauf als `skipped` markiert wird.

`openclaw doctor` meldet Jobs, für die `payload.model` bereits festgelegt ist, einschließlich der Anzahl pro Provider-Namespace und Abweichungen von `agents.defaults.model`. Verwenden Sie diese Prüfung, wenn sich Authentifizierungs-, Provider- oder Abrechnungsverhalten zwischen Live-Chat und geplanten Jobs unterscheiden.

### Modellrangfolge für isolierte Cron-Läufe

Isolierte Cron-Läufe bestimmen das aktive Modell in dieser Reihenfolge:

1. Gmail-Hook-Überschreibung.
2. Jobbezogenes `--model`.
3. Gespeicherte Modellüberschreibung der Cron-Sitzung (wenn der Benutzer eine ausgewählt hat).
4. Agenten- oder Standardmodellauswahl.

### Schneller Modus

Der isolierte Cron-Schnellmodus folgt der aufgelösten Live-Modellauswahl. Die Modellkonfiguration `params.fastMode` wird standardmäßig angewendet, aber eine gespeicherte Sitzungsüberschreibung `fastMode` hat weiterhin Vorrang vor der Konfiguration. Wenn der aufgelöste Modus `auto` ist, verwendet der Grenzwert den Wert `params.fastAutoOnSeconds` des ausgewählten Modells, standardmäßig 60 Sekunden.

### Wiederholungsversuche bei Live-Modellwechseln

Wenn ein isolierter Lauf `LiveSessionModelSwitchError` auslöst, speichert Cron vor dem erneuten Versuch den gewechselten Provider und das gewechselte Modell (sowie, sofern vorhanden, die Überschreibung des gewechselten Authentifizierungsprofils) für den aktiven Lauf. Die äußere Wiederholungsschleife ist nach dem ersten Versuch auf zwei Wechselwiederholungen begrenzt und bricht anschließend ab, statt endlos weiterzulaufen.

## Laufausgabe und Ablehnungen

### Unterdrückung veralteter Bestätigungen

Isolierte Cron-Durchläufe unterdrücken veraltete Antworten, die ausschließlich aus einer Bestätigung bestehen. Wenn das erste Ergebnis nur eine vorläufige Statusaktualisierung ist und kein nachgeordneter Subagent-Lauf für die letztendliche Antwort verantwortlich ist, fordert Cron das tatsächliche Ergebnis vor der Zustellung einmal erneut an.

### Unterdrückung stiller Tokens

Wenn ein isolierter Cron-Lauf nur das stille Token (`NO_REPLY` oder `no_reply`) zurückgibt, unterdrückt Cron sowohl die direkte ausgehende Zustellung als auch den ersatzweisen Pfad für die Zusammenfassung in der Warteschlange, sodass nichts an den Chat zurückgesendet wird.

### Strukturierte Ablehnungen

Isolierte Cron-Läufe verwenden strukturierte Metadaten zur Ausführungsablehnung aus dem eingebetteten Lauf (schwerwiegende Fehler des Ausführungs-Tools mit dem Code `SYSTEM_RUN_DENIED` oder `INVALID_REQUEST`) als maßgebliches Ablehnungssignal. Sie berücksichtigen außerdem Node-Host-Wrapper `UNAVAILABLE` um einen verschachtelten strukturierten Fehler, der einen dieser Codes enthält.

Cron klassifiziert weder Prosa in der endgültigen Ausgabe noch wie Genehmigungsablehnungen formulierte Sätze als Ablehnungen, sofern der eingebettete Lauf nicht ebenfalls strukturierte Ablehnungsmetadaten bereitstellt. Dadurch wird gewöhnlicher Assistententext nicht als blockierter Befehl behandelt.

`cron list` und der Laufverlauf zeigen den Ablehnungsgrund an, statt einen blockierten Befehl als `ok` zu melden.

## Aufbewahrung

Aufbewahrungsverhalten:

- `cron.sessionRetention` (standardmäßig `24h`; zum Deaktivieren `false`) entfernt abgeschlossene isolierte Laufsitzungen.
- Der Laufverlauf behält pro Cron-Auftrag die neuesten 2000 Abschlusszeilen. Verlorene Zeilen behalten das standardmäßige 24-Stunden-Bereinigungsfenster für verlorene Aufgaben bei.

## Ältere Aufträge migrieren

<Note>
Wenn Sie Cron-Aufträge aus der Zeit vor dem aktuellen Zustellungs- und Speicherformat haben, führen Sie `openclaw doctor --fix` aus. Doctor normalisiert veraltete Cron-Felder (`jobId`, `schedule.cron`, Zustellungsfelder der obersten Ebene einschließlich des veralteten Felds `threadId` sowie Zustellungsaliase in der Nutzlast `provider`) und migriert Webhook-Fallback-Aufträge mit `notify: true` vom eingestellten Rohwert `cron.webhook` zur expliziten Webhook-Zustellung, bevor dieser Konfigurationsschlüssel entfernt wird. Aufträge, die bereits Mitteilungen an einen Chat senden, behalten diese Zustellung bei und erhalten ein Webhook-Ziel für den Abschluss. Ohne einen veralteten Webhook wird die inaktive Markierung `notify` der obersten Ebene bei Aufträgen ohne Migrationsziel entfernt (die vorhandene Zustellung bleibt unverändert erhalten), sodass `doctor --fix` nicht mehr wiederholt vor ihnen warnt.
</Note>

## Häufige Änderungen

Zustellungseinstellungen aktualisieren, ohne die Nachricht zu ändern:

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "123456789"
```

Zustellung für einen isolierten Auftrag deaktivieren:

```bash
openclaw cron edit <job-id> --no-deliver
```

Schlanken Bootstrap-Kontext für einen isolierten Auftrag aktivieren:

```bash
openclaw cron edit <job-id> --light-context
```

Mitteilung an einen bestimmten Kanal senden:

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
```

Mitteilung an ein Telegram-Forumsthema senden:

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "-1001234567890" --thread-id 42
```

Einen isolierten Auftrag mit schlankem Bootstrap-Kontext erstellen:

```bash
openclaw cron create "0 7 * * *" \
  "Fassen Sie die Aktualisierungen über Nacht zusammen." \
  --name "Schlanker morgendlicher Überblick" \
  --session isolated \
  --light-context \
  --no-deliver
```

`--light-context` gilt nur für isolierte Agent-Durchlauf-Aufträge. Bei Cron-Läufen lässt der schlanke Modus den Bootstrap-Kontext leer, statt den vollständigen Bootstrap-Satz des Arbeitsbereichs einzufügen.

Einen Befehlsauftrag mit exakten Angaben für argv, cwd, env, stdin und Ausgabegrenzen erstellen:

```bash
openclaw cron create "*/30 * * * *" \
  --name "Positionsexport" \
  --command-argv '["node","scripts/export-position.mjs"]' \
  --command-cwd "/srv/app" \
  --command-env "NODE_ENV=production" \
  --command-input '{"mode":"summary"}' \
  --timeout-seconds 120 \
  --no-output-timeout-seconds 30 \
  --output-max-bytes 65536 \
  --webhook "https://example.invalid/openclaw/cron"
```

## Häufige Verwaltungsbefehle

Manueller Lauf und Überprüfung:

```bash
openclaw cron list
openclaw cron list --agent ops
openclaw cron get <job-id>
openclaw cron show <job-id>
openclaw cron run <job-id>
openclaw cron run <job-id> --due
openclaw cron run <job-id> --wait --wait-timeout 10m
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
openclaw cron runs --id <job-id> --limit 50
openclaw cron runs --id <job-id> --run-id <run-id>
```

`openclaw cron list` zeigt standardmäßig aktivierte Aufträge an. Übergeben Sie `--all`, um deaktivierte Aufträge einzuschließen, oder `--agent <id>`, um nur Aufträge anzuzeigen, deren effektive normalisierte Agent-ID übereinstimmt. Aufträge ohne gespeicherte Agent-ID werden dem konfigurierten Standard-Agent zugerechnet.

`openclaw cron get <job-id>` gibt das gespeicherte Auftrags-JSON direkt zurück. Verwenden Sie `cron show <job-id>`, wenn Sie die menschenlesbare Ansicht mit einer Vorschau der Zustellungsroute wünschen.

`cron list --json` und `cron show <job-id> --json` enthalten für jeden Auftrag ein Feld `status` der obersten Ebene, das aus `enabled`, `state.runningAtMs` und `state.lastRunStatus` berechnet wird. Werte: `disabled`, `running`, `ok`, `error`, `skipped` oder `idle`. Der JSON-Status bleibt kanonisch und unverändert, sodass externe Werkzeuge den Auftragsstatus lesen können, ohne ihn erneut ableiten zu müssen. Die menschenlesbare Ausgabe kann wiederholte Statuswerte `error` mit einer Fehleranzahl versehen.

Einträge in `cron runs` enthalten Zustellungsdiagnosen mit dem vorgesehenen Cron-Ziel, dem aufgelösten Ziel, Sendevorgängen des Nachrichten-Tools, der Nutzung des Fallbacks und dem Zustellungsstatus.

Privater Notizbereich pro Auftrag (Heartbeat-Prüflisten und ähnlicher Überwachungskontext):

```bash
openclaw cron scratch <job-id>                  # aktuellen Inhalt des Notizbereichs ausgeben
openclaw cron scratch <job-id> --json           # Notizbereich plus Revisionsmetadaten
openclaw cron scratch <job-id> --set "text"     # Notizbereich durch exakten Text ersetzen
openclaw cron scratch <job-id> --file notes.md  # Notizbereich aus einer Datei ersetzen (- für stdin)
openclaw cron scratch <job-id> --unset          # Zeile des Notizbereichs entfernen
```

Der Notizbereich wird in der gemeinsamen Statusdatenbank gespeichert, ist auf 256 KiB begrenzt und wird niemals in die Ausgabe von `cron list`/`cron get`/`cron runs` aufgenommen. Schreibvorgänge werden durch Vergleichen und Austauschen anhand der beim Befehlsstart gelesenen Revision abgesichert. Übergeben Sie stattdessen `--expected-revision <n>`, um eine bestimmte Revision festzulegen. Unter [Heartbeat](/de/gateway/heartbeat#monitor-scratch-optional) erfahren Sie, wie Heartbeat-Monitore den Notizbereich verwenden.

Neuzuweisung von Agent und Sitzung:

```bash
openclaw cron edit <job-id> --agent ops
openclaw cron edit <job-id> --clear-agent
openclaw cron edit <job-id> --session current
openclaw cron edit <job-id> --session "session:daily-brief"
```

`openclaw cron add` warnt, wenn `--agent` bei Agent-Durchlauf-Aufträgen ausgelassen wird, und greift auf den Standard-Agent (`main`) zurück. Übergeben Sie beim Erstellen `--agent <id>`, um einen bestimmten Agent festzulegen.

Anpassungen der Zustellung:

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
openclaw cron edit <job-id> --webhook "https://example.invalid/openclaw/cron"
openclaw cron edit <job-id> --best-effort-deliver
openclaw cron edit <job-id> --no-best-effort-deliver
openclaw cron edit <job-id> --no-deliver
```

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Geplante Aufgaben](/de/automation/cron-jobs)
