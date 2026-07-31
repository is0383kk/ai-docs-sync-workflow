---
read_when:
    - Ändern der Ausführung oder Parallelität automatischer Antworten
    - Erläuterung der `/queue`-Modi oder des Verhaltens bei der Nachrichtensteuerung
summary: Modi für die Warteschlange automatischer Antworten, Standardwerte und sitzungsbezogene Überschreibungen
title: Befehlswarteschlange
x-i18n:
    generated_at: "2026-07-26T18:56:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 69b40f67146226b0315492b27fc9d2218cace8bbd1eaff6514f7efb33b69d763
    source_path: concepts/queue.md
    workflow: 16
---

OpenClaw serialisiert eingehende Auto-Reply-Ausführungen (alle Kanäle) über eine kleine prozessinterne Warteschlange, um Kollisionen zwischen mehreren Agent-Ausführungen zu verhindern und gleichzeitig sichere Parallelität über mehrere Sitzungen hinweg zu ermöglichen.

## Warum

- Auto-Reply-Ausführungen können aufwendig sein (LLM-Aufrufe) und kollidieren, wenn mehrere eingehende Nachrichten kurz nacheinander eintreffen.
- Die Serialisierung verhindert Konkurrenz um gemeinsam genutzte Ressourcen (Sitzungsdateien, Protokolle, CLI-Standardeingabe) und verringert die Wahrscheinlichkeit, vorgelagerte Ratenbegrenzungen zu erreichen.

## Funktionsweise

- Eine Lane-spezifische FIFO-Warteschlange arbeitet jede Lane mit einer konfigurierbaren Parallelitätsobergrenze ab (standardmäßig 1 für nicht konfigurierte Lanes; `main` standardmäßig 4, `subagent` standardmäßig 8).
- `runEmbeddedAgent` reiht anhand des **Sitzungsschlüssels** (Lane `session:<key>`) ein, um sicherzustellen, dass pro Sitzung nur eine Ausführung aktiv ist.
- Jede Sitzungsausführung wird anschließend in eine **globale Lane** (standardmäßig `main`) eingereiht, sodass die Gesamtparallelität durch `agents.defaults.maxConcurrent` begrenzt wird.
- Wenn die ausführliche Protokollierung aktiviert ist, geben eingereihte Ausführungen einen kurzen Hinweis aus, falls sie vor dem Start länger als ~2s gewartet haben.
- Eingabeindikatoren werden weiterhin sofort beim Einreihen ausgelöst (sofern vom Kanal unterstützt), sodass die Benutzererfahrung unverändert bleibt, während die Ausführung wartet.

## Standardwerte

Wenn nichts festgelegt ist, verwenden alle Oberflächen für eingehende Kanäle:

- `mode: "steer"`
- `debounceMs: 500`
- `cap: 20`
- `drop: "summarize"`

Steuerung innerhalb desselben Turns ist der Standard. Ein Prompt, der während einer Ausführung eintrifft, wird in die aktive Runtime eingefügt, sofern die Ausführung Steuerung akzeptieren kann; daher wird keine zweite Sitzungsausführung gestartet. Kann die aktive Ausführung keine Steuerung akzeptieren, wartet OpenClaw vor dem Start des Prompts, bis die aktive Ausführung beendet ist.

## Warteschlangenmodi

`/queue` steuert, wie sich normale eingehende Nachrichten verhalten, während für eine Sitzung bereits eine Ausführung aktiv ist:

- `steer`: Nachrichten in die aktive Runtime einfügen. OpenClaw übermittelt alle ausstehenden Steuerungsnachrichten **nachdem der aktuelle Assistenten-Turn die Ausführung seiner Tool-Aufrufe beendet hat**, aber vor dem nächsten LLM-Aufruf; der Codex-App-Server empfängt ein gebündeltes `turn/steer`. Wenn die Ausführung nicht aktiv streamt oder keine Steuerung verfügbar ist, wartet OpenClaw vor dem Start des Prompts, bis die aktive Ausführung beendet ist.
- `followup`: keine Steuerung. Jede Nachricht für einen späteren Agent-Turn nach dem Ende der aktuellen Ausführung einreihen.
- `collect`: keine Steuerung. Eingereihte Nachrichten nach dem Ruhefenster zu einem **einzigen** nachfolgenden Turn zusammenfassen. Wenn Nachrichten an unterschiedliche Kanäle/Threads gerichtet sind, werden sie einzeln abgearbeitet, um das Routing beizubehalten.
- `interrupt`: die aktive Ausführung für diese Sitzung abbrechen und anschließend die neueste Nachricht ausführen.

Informationen zum Runtime-spezifischen Timing und Abhängigkeitsverhalten finden Sie unter [Steuerungswarteschlange](/de/concepts/queue-steering). Informationen zum expliziten Befehl `/steer <message>` finden Sie unter [Steuern](/de/tools/steer).

Die globale oder kanalspezifische Konfiguration erfolgt über `messages.queue`:

```json5
{
  messages: {
    queue: {
      mode: "steer",
      debounceMs: 500,
      cap: 20,
      drop: "summarize",
      byChannel: { discord: "collect" },
    },
  },
}
```

## Warteschlangenoptionen

Die Optionen gelten für die eingereihte Zustellung. `debounceMs` legt im Modus `steer` außerdem das Ruhefenster für die Codex-Steuerung fest:

- `debounceMs`: Ruhefenster vor der Abarbeitung eingereihter Folge-Turns oder zusammengefasster Stapel; im Codex-Modus `steer` das Ruhefenster vor dem Senden gebündelter `turn/steer`. Zahlen ohne Einheit werden als Millisekunden interpretiert; die Einheiten `ms`, `s`, `m`, `h` und `d` werden von den Optionen `/queue` akzeptiert.
- `cap`: maximale Anzahl eingereihter Nachrichten pro Sitzung. Werte unter `1` werden ignoriert.
- `drop: "summarize"` (Standard): die ältesten Warteschlangeneinträge nach Bedarf verwerfen, kompakte Zusammenfassungen beibehalten und diese als synthetischen Folge-Prompt einfügen.
- `drop: "old"`: die ältesten Warteschlangeneinträge nach Bedarf verwerfen, ohne Zusammenfassungen beizubehalten.
- `drop: "new"`: die neueste Nachricht ablehnen, wenn die Warteschlange bereits voll ist.

Standardwerte: `debounceMs: 500`, `cap: 20`, `drop: summarize`.

## Steuerung und Streaming

Wenn das Kanal-Streaming auf `partial` oder `block` gesetzt ist, kann die Steuerung wie mehrere kurze sichtbare Antworten aussehen, während die aktive Ausführung Runtime-Grenzen erreicht:

- `partial`: Die Vorschau kann vorzeitig abgeschlossen werden; anschließend startet eine neue Vorschau, nachdem die Steuerung akzeptiert wurde.
- `block`: Blöcke in Entwurfsgröße können denselben sequenziellen Eindruck erzeugen.
- Ohne Streaming wird die Steuerung nach der aktiven Ausführung als Folge-Turn ausgeführt, wenn die Runtime keine Steuerung innerhalb desselben Turns akzeptieren kann.

`steer` bricht laufende Tools nicht ab. Verwenden Sie `/queue interrupt`, wenn die neueste Nachricht die aktuelle Ausführung abbrechen soll.

## Rangfolge

Zur Auswahl des Modus wertet OpenClaw Folgendes aus:

1. Inline oder gespeichert: sitzungsspezifische Überschreibung durch `/queue`.
2. `messages.queue.byChannel.<channel>`.
3. `messages.queue.mode`.
4. Standard `steer`.

Bei Optionen haben Inline- oder gespeicherte Optionen von `/queue` Vorrang vor der Konfiguration. Anschließend werden in dieser Reihenfolge die kanalspezifische Entprellung (`messages.queue.debounceMsByChannel`), die Standardwerte für die Plugin-Entprellung, die globalen Optionen von `messages.queue` und die integrierten Standardwerte angewendet. `cap` und `drop` sind globale bzw. Sitzungsoptionen und keine kanalspezifischen Konfigurationsschlüssel.

## Sitzungsspezifische Überschreibungen

- Senden Sie `/queue <steer|followup|collect|interrupt>` als eigenständigen Befehl, um den Warteschlangenmodus für die aktuelle Sitzung zu speichern.
- Optionen können kombiniert werden: `/queue collect debounce:0.5s cap:25 drop:summarize`
- `/queue default` oder `/queue reset` entfernt die Sitzungsüberschreibung.

## Abbruch eingereihter Turns

Während ein Prompt in der Folge-/Sammelwarteschlange liegt (beispielsweise wenn ein TUI- oder
Webchat-`chat.send` eintrifft, während ein anderer Turn aktiv ist), verwaltet der Gateway eine
**Gateway-eigene Abbruchidentität** für das `runId` dieses Clients, bis der eingereihte
Inhalt ausgeführt oder verworfen wird. Die Identität folgt Inhalten, die in eine
Überlaufzusammenfassung aufgenommen werden.

- `chat.abort` mit einem bestimmten `runId` bricht diesen Turn ab, solange er noch
  eingereiht ist, sofern der Anfragende autorisiert ist (dieselben Eigentumsregeln wie bei aktiven Ausführungen).
- `chat.abort` für eine Sitzung ohne `runId` bricht **zuerst autorisierte eingereihte Turns
  ab** und anschließend autorisierte aktive Ausführungen. Diese Reihenfolge verhindert, dass beim Abarbeiten der Warteschlange
  Arbeit in eine nur teilweise gestoppte Sitzung übernommen wird.
- Das Leeren der gesamten Sitzungswarteschlange ohne anfragerspezifische Prüfungen ist nicht der
  Stopp-Pfad für Sitzungen mit mehreren Eigentümern.
- Wartephasen in der Warteschlange werden für `sessions.list` nicht als aktive Agent-Ausführungen dargestellt und
  unterliegen nicht der Timeout-Semantik aktiver Ausführungen; dies gilt nur für die aktive Phase.

Gateway-gestützte Clients (einschließlich `openclaw tui`) leiten während einer Ausführung eintreffende Prompts weiter und
überlassen dem Gateway die Anwendung des Warteschlangenmodus. Esc/`/stop` verwendet einen sitzungsbezogenen Abbruch,
damit verlorene lokale Handles nicht dazu führen können, dass ein weiterhin eingereihter Prompt ausgeführt wird.

`openclaw chat` und `openclaw tui --local` wenden dieselben vier Modi in der
eingebetteten Runtime an. Lokales `steer` fügt Inhalte in eine aktive eingebettete Ausführung ein, wenn diese
Runtime Steuerung akzeptiert, und wird andernfalls zu einem Folge-Turn; `followup` und
`collect` bleiben lokal ausstehende Arbeit; `interrupt` bricht die aktive lokale Ausführung
ab, bevor die neueste Nachricht gestartet wird. Der explizite Befehl `/steer <message>` ist
kein Befehl für den lokalen Modus.

## Umfang und Garantien

- Gilt für Auto-Reply-Agent-Ausführungen über alle eingehenden Kanäle hinweg, die die Gateway-Antwortpipeline verwenden (WhatsApp Web, Telegram, Slack, Discord, Signal, iMessage, Webchat usw.).
- Die Standard-Lane (`main`) gilt prozessweit für eingehende Nachrichten und Haupt-Heartbeats; setzen Sie `agents.defaults.maxConcurrent`, um mehrere Sitzungen parallel zuzulassen.
- Es können zusätzliche Lanes vorhanden sein (z. B. `cron`, `cron-nested`, `nested`, `subagent`), damit Hintergrundaufträge parallel ausgeführt werden können, ohne eingehende Antworten zu blockieren. Isolierte Cron-Agent-Turns belegen einen `cron`-Slot, während ihre interne Agent-Ausführung `cron-nested` verwendet. Gemeinsam genutzte Nicht-Cron-`nested`-Abläufe behalten ihr eigenes Lane-Verhalten bei. Diese entkoppelten Ausführungen werden als [Hintergrundaufgaben](/de/automation/tasks) erfasst.
- Sitzungsspezifische Lanes stellen sicher, dass jeweils nur eine Agent-Ausführung auf eine bestimmte Sitzung zugreift.
- Keine externen Abhängigkeiten oder Hintergrund-Worker-Threads; ausschließlich TypeScript und Promises.

## Fehlerbehebung

- Wenn Befehle hängen zu bleiben scheinen, aktivieren Sie die ausführliche Protokollierung und suchen Sie nach Zeilen mit „queued for ...ms“, um zu bestätigen, dass die Warteschlange abgearbeitet wird.
- Ausführungen des Codex-App-Servers, die einen Turn akzeptieren und anschließend keine Fortschrittsmeldungen mehr ausgeben, werden vom Codex-Adapter unterbrochen, damit die aktive Sitzungs-Lane freigegeben werden kann, anstatt auf das Timeout der äußeren Ausführung zu warten.
- Wenn die Diagnose aktiviert ist, werden Sitzungen, die über den integrierten Warnschwellenwert hinaus in `processing` verbleiben, ohne dass eine Antwort oder ein Fortschritt bei Tools, Status, Blöcken oder ACP beobachtet wurde, anhand ihrer aktuellen Aktivität klassifiziert:
  - Aktive Arbeit mit aktuellem Fortschritt wird als `session.long_running` protokolliert. Zugeordnete stille Modellaufrufe verbleiben ebenfalls bis zum integrierten Abbruchschwellenwert in `session.long_running`, damit langsame oder nicht streamende Provider nicht zu früh als blockiert gemeldet werden.
  - Aktive Arbeit ohne aktuellen Fortschritt wird als `session.stalled` protokolliert; zugeordnete Modellaufrufe, blockierte Tool-Aufrufe und blockierte eingebettete Ausführungen wechseln beim oder nach dem Abbruchschwellenwert zu `session.stalled`. Veraltete Modell-/Tool-Aktivität ohne Eigentümer wird nicht als lang laufend verborgen.
  - `session.stuck` ist für wiederherstellbare veraltete Sitzungsbuchführung reserviert, einschließlich inaktiver eingereihter Sitzungen mit veralteter Modell-/Tool-Aktivität ohne Eigentümer.
  - `session.stuck` löst immer eine Wiederherstellung aus, durch die die betroffene Sitzungs-Lane freigegeben werden kann. Eine Klassifizierung als `session.stalled` nach dem Abbruchschwellenwert (blockierter Tool-Aufruf, blockierter Modellaufruf oder blockierte eingebettete Ausführung) kann ebenfalls eine Wiederherstellung durch aktiven Abbruch auslösen, sodass beide Klassifizierungen eine Warteschlange wieder freigeben können, nicht nur `session.stuck`.
  - Wiederholte Warnprotokollzeilen für `session.stuck` und `session.long_running` werden exponentiell seltener ausgegeben, solange die Sitzung unverändert bleibt; Wiederherstellungsversuche werden unabhängig davon weiterhin bei jedem Heartbeat-Tick ausgeführt.

## Verwandte Themen

- [Sitzungsverwaltung](/de/concepts/session)
- [Steuerungswarteschlange](/de/concepts/queue-steering)
- [Steuern](/de/tools/steer)
- [Wiederholungsrichtlinie](/de/concepts/retry)
